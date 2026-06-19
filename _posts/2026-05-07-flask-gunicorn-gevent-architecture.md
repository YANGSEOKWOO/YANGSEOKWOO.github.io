---
layout: post
title: "Flask + Gunicorn + gevent 아키텍처를 뜯어보며 배운 것들"
description: "Python 백엔드 프레임워크부터 Nginx → Gunicorn → Flask 의 계층 구조, 그리고 gevent 의 함정과 대비책까지"
category: CS
tags: [Flask, Gunicorn, gevent, Python, LLM-Backend]
date: 2026-05-07
---

## 한 줄 정의 (Feynman)
> **프레임워크는 "요청 처리 로직" 만 담당하고, WSGI 서버(Gunicorn) 가 "프로세스·동시성" 을 관리하며, Nginx 가 "외부 세계의 거친 입력" 을 막아주는 3계층 구조다. gevent 는 그 중 Gunicorn 워커가 "동기 코드처럼 보이는데 실제로는 비동기로 도는" 마법을 걸어주는 라이브러리다.**

각 계층은 자기가 잘하는 일만 하고, 옆 계층의 일은 절대 안 한다. 그래서 한 명에게 몰빵했을 때보다 훨씬 견고하다.

---

## 1. 핵심 내용 / 구조

### 3계층 모델
```text
[Internet]
    ↓
┌──────────────────────────────┐
│  Nginx                        │  ← "문지기"
│  · SSL 종료                    │
│  · 정적 파일                    │
│  · 큰 업로드 버퍼링              │
│  · 느린 클라이언트 차단           │
│  · 로드밸런싱                   │
└──────────┬───────────────────┘
           ↓ proxy_pass
┌──────────────────────────────┐
│  Gunicorn (gevent worker)     │  ← "프로세스 매니저"
│  · 워커 프로세스 N개 관리         │
│  · 워커당 greenlet 1000+        │
│  · --max-requests 로 자동 재시작 │
│  · timeout 관리                │
└──────────┬───────────────────┘
           ↓ WSGI
┌──────────────────────────────┐
│  Flask app                    │  ← "비즈니스 로직"
│  · 라우팅, 컨트롤러              │
│  · DB 쿼리, LLM 호출            │
│  · 응답 생성                    │
└──────────────────────────────┘
```

### 왜 3계층인가
| 계층 | 잘하는 일 | 못하는 일 |
|---|---|---|
| **Nginx** | C 로 짠 이벤트 루프, 수만 동시 커넥션, SSL/압축/캐시 | 비즈니스 로직 |
| **Gunicorn** | 워커 풀 관리, 신호 처리, 재시작, 타임아웃 | HTTP 1.1 슬로우 클라이언트 방어 |
| **Flask** | 라우팅, DB 모델, 비즈니스 규칙 | 자기 자신을 관리 |

→ 한 명에게 다 시키면 **그 사람이 잘하는 일을 못 하게 된다**. 각자 잘하는 일에만 집중시키는 게 핵심.

### gevent 의 본질
```python
# 파일 최상단에서 단 한 번
from gevent import monkey
monkey.patch_all()

# 이후 모든 동기 코드가 자동으로 비동기처럼 동작
import requests
def fetch():
    r = requests.get("...")  # ← I/O 대기 시 다른 greenlet 으로 자동 양보
    return r.json()
```

`async/await` 한 글자도 안 쓰고 동시 처리량이 **워커당 4 → 4000** 으로 점프.

---

## 2. 무엇을 배웠나

### (1) "프레임워크 ≠ 서버"
Flask 는 **요청을 어떻게 처리할지** 만 안다. 포트 열기, 워커 띄우기, SSL 처리, 큰 업로드 받기 — 이건 다 다른 도구의 일.
이 분리를 모르면 `python app.py` 로 프로덕션을 돌리는 사고가 난다.

### (2) WSGI/ASGI 라는 표준의 의미
프레임워크와 서버가 **표준 인터페이스(WSGI/ASGI)** 로만 대화하기 때문에 자유롭게 갈아끼울 수 있다.
- WSGI(동기): Flask/Django + Gunicorn/uWSGI
- ASGI(비동기): FastAPI + Uvicorn/Hypercorn
- 하이브리드: Gunicorn(프로세스 관리) + UvicornWorker(ASGI 비동기) ← FastAPI 프로덕션 표준

### (3) gevent 의 monkey patch 는 "은밀한 마법"
보이지 않게 표준 라이브러리를 비동기 버전으로 통째로 교체한다.
- 장점: 기존 동기 코드를 한 글자도 안 고쳐도 됨
- 단점: 패치 순서 틀리거나 C extension 만나면 조용히 실패

### (4) 멀티테넌트에서 ContextVar 의 중요성
gevent 환경에서 `threading.local()` 은 의미가 변한다. **`contextvars.ContextVar`** 만이 greenlet/asyncio/threading 셋 모두에서 안전하다.
이걸 안 쓰면 가끔 사용자 A 가 사용자 B 의 데이터를 보는 **세션 누출 보안 사고** 가 난다.

---

## 3. 내가 오해하고 있던 것
> 학습 효과가 가장 큰 부분 — 솔직하게 적기

- **잘못 알고 있던 점**: "Flask 가 알아서 HTTP 서버까지 해주는 줄 알았다." `python app.py` 만으로 프로덕션이 되는 줄 착각.
- **정정된 이해**: Flask 는 WSGI 인터페이스만 노출하는 **순수 로직 라이브러리**. 외부 세계와의 모든 접점은 Gunicorn 같은 별도 서버가 담당. Nginx 는 그 위에서 또 다른 계층의 일을 한다.
- **왜 헷갈렸나**: 개발 모드에서 `flask run` 이 그냥 돌기 때문에 — 이게 **개발용 단일 스레드 서버** 라는 사실이 잘 안 보임. 프로덕션 배포를 직접 해보기 전엔 차이를 체감하기 어려움.

> TODO: gevent 와 asyncio 의 차이를 처음 알았을 때의 인상도 추가하기

---

## 4. 언제 쓰고, 언제 쓰지 않는가 (Trade-offs)

### Flask + Gunicorn + gevent 가 적합한 상황

| 상황 | 적합도 | 이유 |
|---|---|---|
| 기존 동기 Flask 코드베이스를 비동기처럼 돌리고 싶다 | ✅ | 코드 수정 0, gevent 만 켜면 끝 |
| LLM API 호출처럼 I/O bound 작업이 대부분 | ✅ | 워커가 응답 대기로 묶이지 않음 |
| 검증된 ecosystem (flask-sqlalchemy, flask-migrate 등) 활용 | ✅ | 10년 이상 된 표준 도구들 |
| 자가호스팅 / 온프레미스 배포 | ✅ | docker-compose 로 단일 호스트 OK |
| 신규 프로젝트, async 네이티브로 시작 가능 | ❌ | FastAPI 가 더 깔끔, 학습 곡선 낮음 |
| CPU bound 작업이 많음 (파싱, 임베딩 인메모리) | ❌ | gevent 는 CPU 작업 시 워커 통째 멈춤 |
| 99.99% SLA · 글로벌 SaaS 규모 | ⚠️ | 모놀리식 한계, k8s 로 가야 함 |
| 디버깅 편의성 최우선 | ⚠️ | greenlet 스택 트레이스가 꼬임 |

---

## 5. 직접 실험

### gevent 워커가 정말 더 많은 동시 요청을 처리하는가?

```python
# app.py
from gevent import monkey
monkey.patch_all()  # ← 반드시 최상단

import time
from flask import Flask

app = Flask(__name__)

@app.route("/slow")
def slow():
    time.sleep(2)  # ← LLM 응답 대기 흉내
    return "ok"
```

```bash
# 실험 1: sync worker (동기)
gunicorn app:app -w 4 -k sync

# 실험 2: gevent worker
gunicorn app:app -w 4 -k gevent --worker-connections 1000

# 부하 도구
ab -n 100 -c 50 http://localhost:8000/slow
```

**예상 결과**:
- sync: 동시 4개씩만 처리 → 100개에 약 50초
- gevent: 동시 50개 한 번에 처리 → 100개에 약 4초 (2초 × 2 round)

> TODO: 실제 실험 결과 측정 후 숫자 채우기

### monkey patch 순서 실험
```python
# 잘못된 순서
import requests
from gevent import monkey
monkey.patch_all()
# requests 는 이미 동기 socket 으로 import 됨 → 효과 없음

# 올바른 순서
from gevent import monkey
monkey.patch_all()
import requests
# requests 가 비동기 socket 으로 import 됨 → 정상 작동
```

> TODO: 두 케이스의 처리량 차이 실측

---

## 6. gevent 의 12가지 함정과 대비책

> 이 섹션이 이 글의 가치 — 단순히 "쓰자/말자" 가 아니라 **"쓸 때 무엇을 조심해야 하나"**

| # | 함정 | 대비책 |
|---|---|---|
| 1 | monkey patch 순서 틀림 | `app.py` 최상단에서 가장 먼저 호출 |
| 2 | psycopg2(C extension) 블로킹 | `psycogreen.gevent.patch_psycopg()` 추가, 또는 풀 튜닝 |
| 3 | CPU bound 작업이 워커 통째 멈춤 | Celery 워커로 큐 분리 |
| 4 | native 코드 내 sleep 블로킹 | 무거운 ML 연산은 별도 프로세스 |
| 5 | `threading.local()` 의미 변경 | **`contextvars.ContextVar`** 사용 |
| 6 | DB 풀 고갈 | `SQLALCHEMY_POOL_SIZE` × 워커 수 ≤ Postgres `max_connections` 보장 |
| 7 | 로그 인터리빙 | 로그 필터로 `request_id` 자동 주입 |
| 8 | subprocess/signal 미묘한 동작 | 외부 프로세스 호출은 timeout 명시 |
| 9 | asyncio 충돌 | `nest-asyncio` 트릭 (단, 부작용 주의) |
| 10 | long-running greenlet 메모리 누수 | `--max-requests N` 로 워커 자동 재시작 + tracemalloc 모니터링 |
| 11 | 헬스체크가 거짓말 | `/health` 에 DB ping, Redis ping 포함 |
| 12 | 개발/프로덕션 동작 차이 | DEBUG 모드에선 monkey patch 끄고, 스테이징에서 gevent 로 검증 |

### 가장 중요한 3가지
1. **`monkey.patch_all()` 을 무조건 최상단에서 호출** — 위반 시 트래픽 늘면 조용히 망함.
2. **`ContextVar` 일관 사용** — 위반 시 멀티테넌트 보안 사고.
3. **`--max-requests` + 메모리 모니터링** — 누수가 누적되어도 주기적 자가 치유.

---

## 7. 다른 개념과의 연결

### 상위 개념
- **WSGI / ASGI 표준** — 프레임워크와 서버를 분리한 PEP 3333
- **리버스 프록시 패턴** — 외부 트래픽과 내부 서비스 분리

### 하위/연관 개념
- **Greenlet** — gevent 의 기반, OS 스레드보다 가벼운 코루틴
- **이벤트 루프** — gevent(libev), asyncio, Node.js 가 공유하는 모델
- **Monkey patching** — 런타임에 라이브러리 동작을 바꾸는 메타프로그래밍 기법

### 비교 대상
- **FastAPI + Uvicorn** — async 네이티브, 학습 곡선 ↑, 신규 프로젝트에 유리
- **Gunicorn + UvicornWorker** — Gunicorn 의 안정성 + Uvicorn 의 비동기 성능 (FastAPI 프로덕션 표준)
- **Java Project Loom (가상 스레드)** — gevent 와 동일한 철학을 JVM 에서 구현

---

## 8. 관련 Project
> TODO: gevent 패턴이 적용된 프로젝트 글 작성 시 여기에 링크

---

## 마무리: 한 줄 정리
> **Flask + Gunicorn + gevent 는 "동기 코드의 단순함" 과 "비동기의 효율" 을 동시에 얻으려는 실용적 타협이다.** 마법처럼 보이지만 그 마법은 12개의 함정 위에 서 있고, 그 함정들 각각에 명시적인 방어를 깔아둘 때만 안전해진다.
