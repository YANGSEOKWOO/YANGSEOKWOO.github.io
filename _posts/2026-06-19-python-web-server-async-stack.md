---
layout: post
title: "WSGI·ASGI·async부터 gunicorn·uvicorn·gevent까지 — 파이썬 백엔드 스택 바닥부터 풀기"
description: "gunicorn·uvicorn·gevent·celery·async가 헷갈린다면, '같은 줄에 선 경쟁자'가 아니라 '서로 다른 층'으로 보면 한 번에 풀린다."
category: CS
tags: [Python, async, asyncio, WSGI, ASGI, gunicorn, uvicorn, gevent, GIL, coroutine, event-loop]
date: 2026-06-19
---

## 한 줄 정의 (Feynman)
> `gunicorn`, `uvicorn`, `gevent`, `django`, `Flask`, `FastAPI`, `async` — 이건 **백엔드 서버 종류 모음**이 아니라, **서로 다른 4개의 자리(프레임워크 / 서버 / 동시성 모델 / 태스크 큐)에 흩어져 있는 부품**이다. "뭐가 제일 좋아?"가 아니라 "**어느 자리에 있는 애야?**"를 먼저 물어야 한다.

---

## 1. 핵심 내용 / 구조

키워드를 한 덩어리로 보면 영원히 헷갈린다. 4개의 자리로 나누는 게 출발점이다.

| 질문 | 자리(역할) | 여기 있는 애들 |
|---|---|---|
| 1. 내 **앱 로직**을 뭘로 짜? | **프레임워크** | Django, Flask, FastAPI |
| 2. 누가 내 코드를 **실행하고 HTTP를 받아?** | **서버** | gunicorn, uvicorn |
| 3. 한 번에 **요청 여러 개**를 어떻게 처리해? | **동시성 모델** | async(asyncio), gevent |
| 4. 요청-응답 **바깥에서 도는 작업**은? | **태스크 큐** | celery |

그리고 서버(2)와 프레임워크(1)를 잇는 **표준 콘센트**가 WSGI / ASGI다.

```text
브라우저 ──HTTP──> Nginx ──HTTP──> gunicorn ──WSGI──> Flask 앱
                  (C, 리버스       (파이썬        (파이썬)
                   프록시)          WSGI 서버)
                                   └──────┬──────┘
                                    WSGI 경계 (둘 다 파이썬!)
```

- **WSGI**(PEP 3333) = 동기(sync) 규격. **ASGI** = async 가능한 규격.
- WSGI/ASGI는 "규격(글)"이고, gunicorn/uvicorn은 그 규격을 따르는 "서버 구현체(물건)"다.
- WSGI **app** = `def app(environ, start_response)` 모양의 callable 하나. 프레임워크는 네 라우트들을 이 모양으로 포장해주는 기계다. → **프레임워크 인스턴스 == 그 app.**
- ASGI **app** = `async def app(scope, receive, send)` 모양. async라서 시그니처가 다르다.

## 2. 무엇을 배웠나

바닥부터 쌓은 순서:

1. **요청 시간의 대부분은 "기다림"(I/O-bound)** 이다. DB·외부 API 응답을 기다리는 동안 CPU는 논다.
2. **sync 모델**: 워커 하나가 기다리는 동안 통째로 블록 → 동시성은 "워커(프로세스) 수 늘리기"로 해결 (`gunicorn --workers 4`).
3. 근데 프로세스는 무겁고(메모리), 대부분 멍때려서 낭비다. → **async**: 한 스레드가 기다리는 동안 다른 요청을 처리하는 **저글링**.
4. 저글링의 부품: **코루틴**(멈췄다 재개되는 함수, `yield`/제너레이터가 뿌리) + **이벤트 루프**(스레드 위에서 도는 `while` 스케줄러) + **epoll**(OS가 "준비된 소켓 알려줌").
5. async는 **협조형(cooperative)**: `await`에서만 양보. OS의 선점형(강제로 뺏음)과 정반대 → 그래서 **CPU-bound 작업 하나가 전체를 마비**시킨다.
6. **WSGI는 sync 한 방 계약이라 코루틴을 담을 수 없다** → 그래서 **ASGI**가 태어났고, **uvicorn**이 그 ASGI 서버다.
7. **gevent**: `async`/`await` 없이 **monkey-patch**로 양보를 *암묵적*으로 끼워넣어 같은 저글링을 한다.

## 3. 내가 오해하고 있던 것
> 오늘 가장 값진 부분. 실제로 헤맨 지점들.

- **"gunicorn == WSGI다"**
  - 정정: gunicorn은 WSGI라는 *규격*을 따르는 *서버 구현체 중 하나*다 (uWSGI, mod_wsgi, waitress 등도 있음). "삼성 멀티탭 == 220V 규격"이라고 말한 셈.
  - 왜 헷갈렸나: 규격(스펙)과 구현체(물건)를 같은 것으로 봄.

- **"웹서버(Nginx)가 C라서 WSGI가 필요하다"**
  - 정정: WSGI의 존재 이유는 "C라서"가 아니라 **프레임워크↔서버 조립성(N×M → N+M)**. 그리고 **Nginx는 WSGI를 안 쓴다** — Nginx↔gunicorn은 그냥 HTTP. WSGI 경계는 gunicorn↔앱(둘 다 파이썬)이다.
  - 단, mod_wsgi(Apache, C)처럼 C 서버가 WSGI를 직접 말하는 방식은 실존 → 직관이 완전 헛다리는 아니었음. ("Gateway Interface"는 옛 CGI 계보.)

- **"이벤트 루프 = 스레드 하나"**
  - 정정: 이벤트 루프는 *스레드가 아니라* **스레드가 실행하는 `while` 스케줄러 코드**다. 기본은 "스레드 하나 = 그 자체가 루프"(전용 스레드가 따로 생기는 게 아님). 루프 돌리는 스레드의 공식 명칭은 없고 보통 "메인 스레드".

- **"한 스레드 저글링 = OS 멀티프로세스 같은 거"**
  - 정정: 결과는 비슷해도 메커니즘이 정반대. OS는 **선점형**(아무 때나 강제로 뺏음), async는 **협조형**(`await`서만 스스로 양보). 그래서 async는 무거운 계산이 끼면 전체가 멈춘다.

- **"긴 CPU 작업은 워커 스레드 여러 개로"**
  - 정정: 순수 CPU엔 스레드가 답이 아니다. **GIL** 때문에 스레드는 병렬이 안 됨 → **프로세스**(`ProcessPoolExecutor`)가 답. (블로킹 I/O엔 스레드가 맞음 — I/O 중 GIL 놔주니까.) 이게 gunicorn이 프로세스를 여러 개 띄우는 이유와 한 바퀴로 연결된다.

- **"monkey-patch면 gevent가 모든 I/O를 커버"**
  - 정정: 패치된 **표준 라이브러리를 거치는 I/O만** 커버. C 확장 드라이버(psycopg2 등)가 자기 소켓을 직접 열면 hub가 통째로 막힌다. → 순수 파이썬 드라이버 교체 / psycogreen 같은 shim / threadpool 탈출구가 필요.

## 4. 언제 쓰고, 언제 쓰지 않는가 (Trade-offs)

**sync(WSGI) vs async(ASGI)**

| 상황 | 적합도 | 이유 |
|---|---|---|
| I/O-bound (DB·API 호출 많음) | async ✅ | 기다리는 동안 다른 요청 처리 — 적은 자원으로 높은 동시성 |
| CPU-bound (무거운 계산) | async ❌ | 양보할 I/O가 없어 이벤트 루프 전체가 막힘 → 프로세스로 |
| 레거시 sync 코드에 동시성 얹기 | gevent ✅ | monkey-patch로 코드 거의 안 고치고 I/O 동시성 |
| race 추론을 명확히 하고 싶음 | asyncio ✅ | `await`로 양보 지점이 눈에 보임 (gevent는 암묵적) |

**동시성 모델 비교: asyncio vs gevent (둘 다 단일 스레드 + 이벤트 루프 + 가벼운 양보 단위)**

| | asyncio | gevent |
|---|---|---|
| 양보 방식 | 명시적 (`await`) | 암묵적 (monkey-patch) |
| 코드 모양 | async로 다시 써야 함 | 동기 코드 그대로 |
| 생태계 | ASGI (uvicorn, FastAPI) | WSGI (`gunicorn -k gevent`, Flask) |
| C 블로킹 드라이버 | async 네이티브 필수 (asyncpg 등) | shim/순수파이썬/threadpool 필요 |

**핵심 교훈:** 협조형 동시성은 *모든* 블로킹 지점이 협조해야 성립한다. 단 하나라도 협조 안 하는 블로킹 호출이 단일 스레드 배 전체를 가라앉힌다 — asyncio든 gevent든 동일.

## 5. 직접 실험
> ⚠️ 이번 세션은 로컬에 파이썬이 없어 **코드로 직접 검증하지 못했다.** 설명은 PEP 3333 / ASGI 스펙 / asyncio·gevent·gunicorn 공식 문서 기반이다. 파이썬 설치 후 아래를 직접 돌려 확인할 것.

```python
# (1) 코루틴이 진짜 "멈췄다 이어진다"를 제너레이터로 눈으로 보기
def gen():
    print("시작")
    x = yield 1          # 여기서 얼고 1을 내줌
    print("재개됨, x =", x)
    yield 2
g = gen()
print(next(g))   # "시작", 1  → yield에서 정지
print(next(g))   # "재개됨, x = None", 2

# (2) CPU-bound: 스레드는 안 빨라지고(GIL) 프로세스는 빨라진다
#   ThreadPoolExecutor vs ProcessPoolExecutor 로 무거운 루프 시간 재보기

# (3) gevent: monkey.patch_all() 후 time.sleep을 여러 greenlet에서 동시에
#   직렬이면 N초 걸릴 게 ~1초에 끝나면 "양보가 일어났다"는 증거
```
- 결과/관찰: _(파이썬 설치 후 채울 것)_

## 6. 다른 개념과의 연결
- 상위 개념: 동시성(concurrency) vs 병렬성(parallelism), I/O 멀티플렉싱(epoll/select)
- 하위/연관 개념: 코루틴, 이벤트 루프, greenlet, **GIL**, 제너레이터/`yield`
- 비교 대상: 선점형 vs 협조형 멀티태스킹, 프로세스 vs 스레드

## 7. 관련 글
<!-- 이 블로그의 첫 글이라 아직 링크할 다른 글이 없다. 앞으로 쓸 글 후보(= 그래프 엣지 예약): -->
- _(예정)_ **GIL 바닥부터** — 왜 한 스레드만 파이썬 바이트코드를 실행하는가, free-threaded(3.13+)는 뭘 바꾸나 → 이 글의 "스레드 vs 프로세스" 절과 연결
- _(예정)_ **celery — 요청-응답 바깥의 작업** — 왜 웹서버로 다 못 하고 태스크 큐가 필요한가 (오늘 안 다룬 마지막 조각)
- _(예정)_ **asyncio 직접 실험** — 위 5번 실험을 실제로 돌려 결과 채우기, `async`/`await`·이벤트 루프 손으로 만들어보기
- _(예정)_ **제너레이터와 코루틴** — `yield`에서 코루틴이 어떻게 자라났나 (PEP 342 → 492)
