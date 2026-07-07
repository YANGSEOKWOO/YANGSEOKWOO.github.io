---
layout: post
title: "시스템 아키텍처 배우기 — Celery+Redis를 Kafka·outbox 없이 버티게 만든 DB 재조정 패턴"
description: "자바의 Kafka+outbox 결제 트랜잭션 방어와 사내 백엔드의 Celery+Redis 구조를 매핑하며 배운 것 — 브로커를 믿지 않고 DB를 진실원본으로 두는 재조정 루프, 그리고 그 루프의 심장인 beat가 SPOF라는 사실"
category: Project
tags: [Celery, Redis, Outbox, Reconciliation, Beat]
date: 2026-07-08 00:00:00 +0900
---

> 📊 **인터랙티브 Q&A 버전**: [비동기 인프라 아키텍처 — 예상 질문 14선 + 설계 회고](/interactive/celery-redis-reconciliation.html) — 브로커·워커·beat·재조정·HA를 카드형 Q&A로, 강점/단점/트레이드오프까지 정리한 페이지.

## 한 줄 회고 (Feynman)
> 이 일을 처음 듣는 사람에게 한 문장으로 설명한다면?

> 브로커(Redis)를 믿는 대신 DB를 진실원본으로 두고, beat가 주기적으로 현실을 원하는 상태로 맞춘다 — 메시지를 잃어도 "해야 할 일"은 DB에 남아 결국 따라잡는다.

---

## 1. 문제 상황

- **맥락**: 자바 결제 시스템에서는 서버가 죽을 때 트랜잭션 유실을 막으려고 Kafka(내구성 있는 로그) + outbox 패턴(비즈니스 write와 메시지 발행을 한 트랜잭션으로 묶기)을 쓴다. 그 사고방식을 **우리가 만든 백엔드 서버(Celery + Redis)에 매핑**하면 무엇이 대응되는지 궁금했다.
- **왜 문제였나**: 우리 서버는 Redis를 broker로 쓴다. Redis는 기본적으로 휘발성이다. 그러면 순진하게 보면 "Redis 죽으면 큐에 있던 task 다 날아가는 거 아냐?"라는 의문이 생긴다. 실제로 이걸 어떻게 방어하는지 코드로 확인해야 했다.
- **세션 중 직접 제기한 급소**: "beat가 주기적으로 DB를 읽어 따라잡는 방식이라면, **그 beat가 죽으면 누가 따라잡나?**" — 재조정 루프의 구동원 자체의 내결함성을 물은 것.

### 확인한 실제 구성

운영 중인 Celery/브로커 설정 기준:

| 항목 | 값 | 의미 |
|---|---|---|
| Broker | Redis (`CELERY_BROKER_URL`, `rediss://`면 TLS) | task 메시지 전달만 |
| Result backend | 기본 `database` → PostgreSQL, 게다가 `task_ignore_result=True` | 결과는 Redis에 안 담김 |
| `worker_prefetch_multiplier` | `1` | 워커가 한 번에 1개만 선점 |
| `task_acks_late` | **설정 없음 = False** | task 받는 순간 ACK → 실행 중 워커 죽으면 그 task 유실 |
| `task_reject_on_worker_lost` | **설정 없음 = False** | 워커 유실 시 재전달 안 함 |
| Sentinel | `CELERY_USE_SENTINEL` (기본 off) | 켜면 Redis 마스터-슬레이브 자동 승격(failover) |
| beat 스케줄러 | Celery 기본 `PersistentScheduler` (redbeat/리더선출 **없음**) | beat = 단일 프로세스 SPOF |

## 2. 가능했던 해결책들

| 옵션 | 장점 | 단점 |
|---|---|---|
| A. Kafka 도입 (자바식) | 디스크 로그 + replica로 메시지 내구성 강함, 오프셋 되감기로 재처리, 파티션 순서 보장 | 인프라 무게(운영·학습 비용) 큼, 파이썬/Celery 생태계와 결이 다름 |
| B. Redis durability 강화 (AOF/replica/Sentinel 상시) | 대기 메시지 휘발 자체를 줄임 | 여전히 브로커를 "믿는" 구조, `acks_late=False`면 워커 크래시 유실은 못 막음 |
| C. **DB를 진실원본으로 둔 재조정(reconciliation)** | 브로커를 안 믿음 → Redis 휘발/워커 크래시와 무관하게 DB 기준으로 따라잡음, 가벼움 | at-least-once → 멱등성 필수, 재조정 구동원(beat)이 SPOF |

## 3. 현실적 타협점

- 우리 서버는 **C(DB 재조정)**를 택했다. 정통 transactional outbox(별도 outbox 테이블 + relay)보다 가벼운 변형 — "DB에 상태를 남기고 주기 beat가 다시 스캔해 미처리분 재발행".
- 문서 임베딩(지식 인덱싱) 파이프라인이 교과서적 예시:
  - 진실원본 = 문서의 상태 컬럼들(인덱싱 상태 + 큐 등록/발행 시각)
  - "발행 대상" = 대기 상태 + 아직 큐에 안 넣음 + 재조정 대상으로 등록됨
  - 디스패치 트리거 **3중화**: ① 등록 직후 ② 파일 task 완료 직후 ③ **안전망 beat(60초)** — ①②가 유실돼도 ③이 DB 재스캔으로 복구
  - stale 회수 beat(15분 주기): 재조정 대상 문서가 오래 멈춰 있으면 재큐 또는 poison(영구 실패) 처리
- **동작 과정 요약**:
  ```
  [정상]   등록 → DB에 상태 기록 → Redis dispatch → 워커 실행 → DB 갱신
  [Redis죽음] 대기 메시지 휘발 → 60초 후 안전망 beat가 DB 재스캔 → 재발행
  [워커죽음]  진행 중 task 유실(acks_late off) → 15분 후 stale beat가 재큐/poison
  ```
- 폭주 방지: 복구 시점에 밀린 일이 한꺼번에 쏟아지지 않도록 beat에 `options={"expires": N}` (다음 주기와 안 겹치게 만료).

### 이 패턴은 임베딩만이 아니다 — 도메인 전반의 재조정

임베딩은 이 패턴의 **가장 최근 사례일 뿐**, 같은 골격이 서버 곳곳에 이미 있다. 성격이 둘로 나뉜다.

**A. 장애 복구형** — "죽은 걸 DB 상태로 감지해 되살림"

| 사례 | 상태원본 | 주기 | 특징 |
|---|---|---|---|
| 임베딩 회수 | 문서 상태·시각 컬럼 | 15분 | 기준 사례 |
| 워크플로우 stale | 실행 상태 + 시각 | 5분 | in-stream `finally` 유실분의 2차 방어 |
| 장시간 Job stale | 하트비트 시각 | 5분 | **하트비트** 방식 |

**B. 시간 만료형** — "때가 됐으면 상태 전이"

| 사례 | 상태원본 | 주기 | 특징 |
|---|---|---|---|
| 일시정지 타임아웃 | 일시정지 상태 + Redis TTL | 5분 | **Redis 휘발을 신호로 활용** |
| API 키풀 복구 | 리셋 예정 시각 | 1분 | 시각 지나면 active 복귀 |
| 앱 차단 자동해제 | 차단 만료 시각 | 1분 | 멱등 처리 |

임베딩엔 없던 **새로 배운 변형 2가지**:

1. **하트비트(장시간 Job)** — 단일 시작시각 하나로는 "정상적으로 오래 걸리는 일"과 "죽어서 멈춘 일"을 구분 못 한다. 실행 중 하트비트 시각을 주기적으로 갱신하면 "살아있으면 계속 신호를 보낸다"라서 그 둘을 정확히 구분한다.
2. **Redis TTL + DB 이중상태(일시정지)** — 일시정지를 Redis 키(TTL 있음) + DB 상태 두 군데에 둔다. Redis 키가 TTL로 만료·소실되면 → beat가 "Redis 없음 = 타임아웃"으로 판정해 DB를 실패 상태로 마감. **처음 걱정했던 "Redis 휘발"을 오히려 신호로 뒤집어 쓰는** 사례. DB가 최종 진실원본이라 Redis가 사라져도 상태가 미아가 안 된다.

그리고 가장 근본은 앱 스케줄러(매분): 스케줄 설정 테이블을 source of truth로 삼는다. "DB를 진실원본으로, beat가 주기적으로 읽어 행동"하는 이 패턴의 원형이고, 임베딩 재조정은 그 사상을 인덱싱 도메인에 적용한 것.

## 4. 왜 이게 최선이었나

- **판단 기준**: "진짜 안 잃어야 할 상태는 이미 PostgreSQL에 있다. 그러면 브로커의 내구성에 돈·복잡도를 쓸 게 아니라, DB를 주기적으로 재조정하면 된다."
- **A(Kafka)를 버린 이유**: 결제처럼 강한 순서·내구성이 요구되는 도메인이 아니고, 이미 멱등 task + DB 상태가 깔려 있어 Kafka의 이득 대비 인프라 무게가 과했다.
- **B를 버린(만으로 부족한) 이유**: durability를 높여도 `acks_late=False`인 한 워커 크래시 유실은 남는다. C는 그 케이스까지 DB 기준으로 흡수한다.
- **`acks_late=True`를 굳이 안 켠 이유**: 켜면 워커 크래시 시 재전달되지만 중복 실행 위험이 늘어난다. 이미 재조정 beat가 있어 굳이 중복 리스크를 감수할 이유가 약하다.

### 그런데 — beat가 죽으면? (이 글의 핵심 발견)

- 재조정 루프를 **전부 beat가 굴린다**(안전망 60초, stale 15분, 앱 스케줄러 매분 등).
- 그런데 **beat 자신은 이중화가 없다**: redbeat/리더선출 미사용 → beat를 2개 띄우면 모든 주기 task가 2번씩 발화(중복) → 구조적으로 단일 인스턴스 = **SPOF**.
- 즉 **워커·Redis 장애는 beat가 자가치유하지만, beat 장애를 자가치유하는 상위 안전망은 코드에 없다.**
- beat를 살리는 건 애플리케이션이 아니라 **오케스트레이터**: Docker restart policy / K8s liveness probe가 죽은 beat를 재시작. 내결함성이 "이중화"가 아니라 **"단일 프로세스 빨리 되살리기"**에 위임돼 있다.
- 재시작 후: crontab 태스크는 다운타임 중 놓친 실행을 **backfill 안 함**(다음 매칭 시각 발화). 하지만 "해야 할 일"은 DB에 남아 있어, 복구 직후 첫 스캔에서 밀린 dispatchable/stale을 한꺼번에 따라잡는다.
- **자바/Kafka 대비**: Kafka 컨슈머는 죽으면 리밸런싱으로 다른 컨슈머가 파티션을 이어받는다(자동 failover). 우리 beat엔 그런 리밸런싱이 없다 — beat는 Kafka 컨슈머보다 "단일 크론 트리거"에 가깝고, 죽으면 오케스트레이터 재시작에만 의존한다.

### 그럼 잘못된 설계인가? 파이썬 한계인가?

이 질문을 스스로 던졌고, 결론은 **둘 다 아니다.**

- **파이썬 한계 아님** — Java로 짜도 같은 구조가 나온다. `@Scheduled`/Quartz = beat, **ShedLock**(분산락으로 스케줄러 하나만 활성) = redbeat, Spring Batch = 재조정 잡. 특히 "스케줄러를 여러 대 띄우면 중복 실행되니 분산락으로 하나만" — 이건 ShedLock이 존재하는 이유와 **글자 그대로 동일한 문제·해법**이다. beat SPOF는 언어가 아니라 "단일 리더 스케줄러" 개념 자체의 성질.
- **잘못된 설계 아님 — 오히려 정석** — 핵심 오해를 풀어야 한다: "브로커를 믿고 재조정 없이 가는 것"이 더 좋은 설계가 아니다. Kafka+outbox를 쓰는 성숙한 결제 시스템도 거의 항상 reconciliation 배치를 backstop으로 둔다. 메시지 시스템은 언제든 미묘하게 어긋나기 때문. 즉 우리 서버는 **"outbox의 앞부분(브로커 내구성)을 생략하고, 진짜 안전망인 뒷부분(DB 재조정)만 남긴"** 것 — 안전망을 뺀 게 아니라 가장 튼튼한 층을 택한 것. K8s가 전 세계 프로덕션을 돌리는 원리가 이것이다.
- **그래도 트레이드오프는 분명하다** — 여기가 직감이 옳았던 부분: (1) 폴링이라 복구가 즉시가 아니라 다음 주기(수 분), (2) at-least-once라 멱등성 강제, (3) 재조정의 심장 beat가 SPOF라 운영(restart/liveness)이 반드시 받쳐야, (4) exactly-once 불가.
- **판단**: 이 대상들(임베딩·인덱싱·정리·토큰갱신)은 "몇 분 늦게 복구돼도 되고 멱등하게 다시 해도 되는" 백그라운드 작업이라 현 구조가 맞다. 여기에 Kafka+outbox+exactly-once를 얹으면 **과설계**. 반대로 이 구조로 **결제**를 처리한다면 그땐 정말 잘못된 선택이 된다.

## 5. 예상 vs 실제

| | 예상 | 실제 |
|---|---|---|
| 결과 | Redis가 죽으면 큐에 있던 task가 다 날아가 작업이 손실될 줄 알았다 | DB가 진실원본이라 beat 재조정이 흡수 — 대기 메시지는 휘발돼도 "작업 손실"은 아니었다 |
| 이해 범위 | 브로커/Celery 설정만 보면 그림이 나올 줄 알았다 | 유실 방지의 축이 브로커가 아니라 DB 재조정이라, 여러 도메인의 stale/만료 태스크까지 봐야 전체 그림이 잡혔다 |
| 부작용 | 재조정이 있으면 공짜로 튼튼해지는 줄 | at-least-once라 멱등성·fencing token이 필수 부담으로 따라오고, beat SPOF라는 "운영이 책임지는 영역"이 새로 드러났다 |

> **가장 빗나간 지점은?** ← 여기에 학습이 있음
> beat가 단일 프로세스(SPOF)라는 걸 보고 "이거 설계가 틀린 거 아냐? 파이썬 한계 아냐?"라고 예단한 것. 실제로는 분산 시스템의 정석(K8s reconciliation loop, outbox의 backstop)이었고, Java도 `@Scheduled` + ShedLock으로 **같은 문제를 같은 방식으로** 푼다. "잘못된 설계"라는 내 직감이 가장 크게 빗나갔고, 거기서 "약점처럼 보이는 게 실은 의도된 트레이드오프일 수 있다"를 배웠다.

## 6. 추후 개선 방향

- **beat 이중화가 필요해지면**: `redbeat`(Redis 분산 락으로 여러 beat 중 하나만 활성) 도입, 또는 K8s Deployment(replicas=1) + liveness로 "빠른 재생성" 보장 강화.
- **유실 창을 더 줄이려면**: `task_acks_late=True` + `task_reject_on_worker_lost=True` — 단 멱등성 필수 + `visibility_timeout`(현재 기본 3600s) 튜닝 필요. 지금은 재조정 beat가 있어 중복 리스크 대비 이득이 작다.
- **재설계 시그널**: 강한 순서 보장/정확히 한 번(exactly-once)이 요구되는 도메인(예: 결제)이 생기면 그때 Kafka+outbox 정통 구현을 검토.
- **모니터링 관점**: beat 프로세스 liveness와 "재조정 지연(dispatchable이 오래 쌓임)" 알람이 있으면 beat 다운을 빨리 감지 가능. (현재 실제 알람 유무는 배포 매니페스트 접근이 막혀 미확인 — 코드가 아니라 helm/k8s 설정을 봐야 한다. 배포에서 beat의 restart/liveness와 재조정 지연 알람이 실제로 걸려 있는지가 이 아키텍처의 실질적 안전 여부를 가른다.)

## 7. 관련 CS 지식

- [K8s 핵심 동작 원리 — desired state 와 reconciliation loop](/cs/kubernetes-desired-state-reconciliation/) — beat+DB 재조정은 K8s의 "current state를 desired state로 끊임없이 맞추는 control loop"와 **완전히 같은 사상**이다. beat = control loop, DB의 상태 컬럼 = desired/current state 비교 대상.
- [Flask + Gunicorn + gevent 아키텍처](/cs/flask-gunicorn-gevent-architecture/) — Celery 워커의 프로세스/동시성 모델(`prefetch_multiplier=1`, 워커 크래시 시 동작)을 이해하는 배경.
