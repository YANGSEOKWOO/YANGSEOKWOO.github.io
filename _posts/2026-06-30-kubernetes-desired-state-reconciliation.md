---
layout: post
title: "K8s 핵심 동작 원리 — desired state 와 reconciliation loop 바닥부터"
description: "Deployment·Service 는 쓸 줄 알지만 '속이 어떻게 도는지' 가 비었다면 — Control Plane / Worker, desired state, control loop 를 직접 Pod 죽여가며 확인한 노트"
category: CS
tags: [Kubernetes, ControlLoop, DesiredState, Reconciliation, ReplicaSet, Service, Infra]
date: 2026-06-30 00:00:00 +0900
---

## 한 줄 정의 (Feynman)
> 쿠버네티스는 "내가 원하는 상태(desired state)" 만 적어두면, 두뇌(Control Plane)가 "지금 실제 상태(current state)" 와 끊임없이 비교해서 다르면 같아지게 맞춰주는 시스템이다. 이 비교-보정 무한루프 하나가 K8s 전부다.

---

이 글은 "Deployment → Service → Ingress 로 배포는 하는데, 정작 K8s 가 속으로 무슨 일을 하는지 모르겠다" 는 상태에서 출발해, minikube 로 클러스터를 직접 띄우고 **Pod 를 죽여가며** 동작 원리를 눈으로 확인한 학습 기록이다. 손에 익은 `kubectl apply` 가 사실은 무슨 행위였는지 다시 보게 된 과정.

## 1. 핵심 내용 / 구조

### Control Plane(두뇌) vs Worker(일꾼)

대규모 자동화를 하려면 역할이 자연히 둘로 갈린다 — **결정하는 쪽**과 **실행하는 쪽**.

```text
┌────────────── Control Plane (두뇌, 내 컨테이너는 여기서 안 돈다) ──────────────┐
│  "원하는 상태: 3개"  vs  "지금 상태: 2개"  → "1개 더 띄워!" 라고 결정/명령        │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
   Worker 1 🟦              Worker 2 🟦              Worker 3 (새로 여기) 🟦
        실제 Pod(컨테이너)가 도는 곳
```

- **Worker 노드**: 내 Pod 가 실제로 도는 머신.
- **Control Plane**: 내 앱은 안 돌리고 **관리만** 한다. 핵심 임무 단 하나 — *desired state* 와 *current state* 를 끊임없이 비교하고, 다르면 같아지게 만든다.
- minikube 는 노드가 1개라, 그 하나가 **control-plane 과 worker 를 겸직**한다(원래 control-plane 노드에 걸리는 빗장 taint 를 풀어둔 특수 세팅).

### Control Plane 안의 부품 = 그 루프를 쪼갠 것

| 부품 | 역할 | 비유 |
|---|---|---|
| **etcd** | 모든 상태를 저장하는 key-value DB (원함 + 현실 전부) | 두뇌의 장기기억 노트 |
| **API Server** | 모든 요청의 정문. **etcd 와 직접 대화하는 유일한 부품** | 금고 열쇠 가진 안내데스크 |
| **Scheduler** | 새 Pod 를 **어느 노드에 둘지**만 결정 | 택배 분류 담당 |
| **Controller Manager** | reconciliation loop 를 실제로 돌리는 엔진 (안에 컨트롤러 여러 종류) | "모자라네? 채워!" 닦달하는 실행 의지 |
| **kubelet** *(Worker 쪽)* | 명령받아 노드에서 **진짜 컨테이너를 실행** | 현장 작업반장 |

`kubectl -n kube-system get pod kube-controller-manager-minikube -o yaml` 로 까보면 `--controllers=*,...` 플래그가 보인다 — "컨트롤러가 여러 종류 들어있다" 의 직접 증거.

### 한 사이클 (Pod 가 자동 복구되는 흐름)

```text
1. kubectl apply (replicas:3)  → [API Server] : "이 상태 원해"
2. [API Server]                → [etcd]       : 원하는 상태 저장
3. [ReplicaSet 컨트롤러]        : (watch 로 감시) "3개 원하는데 2개네? 보충"
                                 → API Server 에 새 Pod 생성 등록
4. [Scheduler]                 : "이 Pod 어느 노드에? → 배정"
5. 해당 노드 [kubelet]          : 실제 컨테이너 실행 🟦
6. Controller Manager 는 영원히 감시 (이게 reconciliation loop)
```

## 2. 무엇을 배웠나

- **선언형(declarative) 의 정체**: `kubectl apply` 는 "이거 켜라/꺼라"(명령형) 가 아니라 **"최종 상태가 이랬으면 좋겠어"(상태 선언)** 였다. 행동이 아니라 *상태* 를 등록하는 행위.
- **모든 것은 `spec` + `status`**: Deployment 만의 기능이 아니다. Pod·Service·Node 등 K8s 의 **모든 object** 가 `spec`(내가 원하는 것) + `status`(지금 현실) 구조다. 나는 항상 `spec` 만 쓰고, 컨트롤러는 둘을 같게 만든다. desired state 는 K8s 의 세계관 그 자체.
- **Deployment → ReplicaSet → Pod 3층 구조**: Pod 이름 `web-68d995574f-xxxxx` 의 가운데 해시는 사실 **ReplicaSet 이름**이었다.
  - **ReplicaSet** = 개수 지킴이 (replicas N개 유지)
  - **Deployment** = 버전 지휘자 (ReplicaSet 을 여러 개 부려 롤아웃/롤백)
- **Service 의 느슨한 연결**: Service 는 Pod IP 를 적어두지 않는다. `selector` 로 *"`app=web` 라벨 가진 Pod"* 라는 **조건**만 걸고, **Endpoints** 가 그 조건에 맞는 현재 살아있는 Pod IP 목록을 실시간 관리하며, **kube-proxy** 가 그걸 노드의 iptables 규칙으로 구현한다. 그래서 Pod 가 죽고 IP 가 바뀌어도 새 Pod 가 같은 라벨로 나오면 자동 편입된다.
- 기존 지식과의 연결: 평소 쓰던 `deployment → service → ingress` 에서 **service 가 속으로 어떻게 도는지**(label/selector → endpoints → kube-proxy)가 채워졌다.

## 3. 내가 오해하고 있던 것
> 학습 효과가 가장 큰 부분 — 실제로 헤맨 지점 그대로.

**① reconciliation 은 "몇 초마다 체크하는 타이머(폴링)" 다?**
- 잘못 알던 점: 컨트롤러가 주기적으로 돌면서 상태를 *센다* 고 생각했다 (사실 학습 초반의 `while: 세어봄` 의사코드가 이 오해를 부추겼다).
- 정정된 이해: **폴링이 아니라 이벤트 기반(watch)**. 컨트롤러는 API Server 에 "변화 생기면 알려줘" 라고 구독해두고, Pod 가 사라지는 *순간* 알림받아 거의 실시간으로 반응한다. 주기적인 resync 는 "혹시 알림 놓쳤을까 봐 가끔 전체 재점검" 하는 **백업**일 뿐 주력 메커니즘이 아니다. (그 resync 주기 숫자가 K8s 버전 타는 것.)
- 왜 헷갈렸나: "감시" 라는 말에서 "주기적으로 들여다본다" 를 자동 연상해서. 실제론 "알림 구독" 에 가깝다.

**② Pod 를 자동 복구하는 건 "api controller" 가 etcd 를 직접 보고 한다?**
- 잘못 알던 점: 'API + 컨트롤러' 를 하나로 뭉뚱그려 "api controller 가 etcd 정보 보고 kubelet 시킨다" 고 말했다.
- 정정된 이해: 결정하는 건 **ReplicaSet 컨트롤러**(Controller Manager 안). 그리고 컨트롤러는 **etcd 를 직접 안 본다** — etcd 와 직접 대화하는 건 **오직 API Server**. 컨트롤러는 API Server 에 watch 를 걸어 정보를 받는다. 또 kubelet 직전에 **Scheduler 가 노드 배정** 단계가 있다.
- 왜 헷갈렸나: 부품 이름표를 안 붙인 채 "두뇌가 알아서" 로 통째로 이해해서.

**③ ReplicaSet 의 존재 이유가 버전 관리다?**
- 잘못 알던 점: "ReplicaSet 을 둔 이유 = 롤아웃 버전관리" 라고 역할을 거꾸로 알았다.
- 정정된 이해: 버전관리는 **Deployment** 의 일. ReplicaSet 은 그냥 **개수 유지**만 한다. `set image` 하면 ReplicaSet 이 2개로 갈리는데, 각 RS 는 자기 개수만 멍청하게 지키고, 그 둘의 개수를 밀고 당겨 버전을 갈아끼우는 **지휘자가 Deployment**다.

**④ namespace 로 나누면 물리적·네트워크 격리가 된다?**
- 정정된 이해: namespace 는 **논리적** 구분(이름 충돌 방지 + 권한/할당량). 물리 머신과 무관하고(같은 ns Pod 가 다른 노드에, 다른 ns Pod 가 같은 노드에 공존 가능), **네트워크도 기본은 안 막힌다**. 진짜 막으려면 NetworkPolicy 를 따로 건다.

## 4. 언제 쓰고, 언제 쓰지 않는가 (Trade-offs)

| 상황 | 적합도 | 이유 |
|---|---|---|
| 항상 N개 떠 있어야 / 죽으면 자동 복구 / 무중단 배포 | ✅ | desired state + reconciliation 이 정확히 이걸 위해 만들어짐 |
| 한 Pod 에 여러 컨테이너 (사이드카: 로그수집·프록시) | ✅ | 같은 IP·디스크 공유하며 한 몸으로 살고 죽어야 할 때 |
| 학습/로컬 검증 (minikube) | ✅ | 1노드지만 "진짜 K8s" 라 동작 원리가 운영과 동일 |
| 분산 스케줄링·노드 장애 이전(failover) 체감 | ⚠️ | minikube 는 노드 1개라 "다른 노드로 옮김" 은 안 보임 |
| namespace 로 보안 격리 기대 | ❌ | 논리적 구분일 뿐, 격리는 NetworkPolicy/RBAC 따로 |

## 5. 직접 실험

minikube(`--driver=docker`)로 클러스터를 띄우고, 실제로 Pod 를 죽여 reconciliation 을 눈으로 확인했다.

```bash
kubectl create deployment web --image=nginx --replicas=3
kubectl get pods -w        # 실시간 감시 켜두고
kubectl delete pod web-68d995574f-4xhnr   # 다른 창에서 하나 죽임
```

`-w` 출력 타임라인:

```text
32s  web-...-4xhnr  Terminating    ← 죽인 순간
0s   web-...-tzvkt  Pending        ← ⚡거의 동시에 새 Pod 등장 (이름/해시 다름)
0s   web-...-tzvkt  ContainerCreating
2s   web-...-tzvkt  Running        ← 보충 완료, 다시 3개
```

- **관찰 1 (watch 증명)**: `Terminating` 과 새 Pod `Pending` 이 *사실상 같은 순간* 에 찍혔다. 5초/30초 타이머를 안 기다렸다 → 폴링이 아니라 이벤트 기반(watch)이라는 증거.
- **관찰 2 (집합을 본다)**: 죽은 `4xhnr` 가 아니라 이름 다른 `tzvkt` 가 떴다. 두뇌는 *그 Pod* 가 아니라 *"3개라는 숫자"* 를 맞춘다. Pod 는 일회용품.

롤아웃도 ReplicaSet 갈리는 걸로 확인:

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl get replicaset
```

```text
NAME             DESIRED   CURRENT   READY
web-669bb98f9f   1→3       ...       ...   ← 신버전 RS, 서서히 늘림
web-68d995574f   3→0       ...       ...   ← 구버전 RS, 서서히 줄임 (롤백 위해 0으로 남겨둠)
```

- **관찰 3 (무중단의 정체)**: 신버전이 `READY` 될 때까지 구버전이 `3/3/3` 으로 버텼다 → 서빙 가능한 Pod 가 한순간도 0이 안 됐다.
- **관찰 4 (잠깐 4개)**: 롤아웃 중 `신1 + 구3 = 4개` 순간 존재 → "잠깐 넘쳐도 됨"(maxSurge)으로 안전하게 갈아끼움.
- **관찰 5 (DESIRED = spec)**: `DESIRED` 컬럼이 곧 각 RS 의 `spec.replicas`. 롤아웃이란 결국 **두 RS 의 desired 숫자를 반대로 미는 것**이었다. `kubectl rollout undo` 하면 그대로 역방향(구 RS 0→3, 신 RS 3→0).

Service 의 느슨한 연결도 확인:

```bash
kubectl expose deployment web --port=80
kubectl get service web      # CLUSTER-IP 10.96.x.x (고정)
kubectl get endpoints web    # 뒤의 Pod IP 10.244.x.x 목록
kubectl delete pod <하나>     # 죽이면 → endpoints 에서 그 IP 빠지고 새 IP 편입, Service IP 는 불변
```

## 6. 다른 개념과의 연결

- **상위 개념**: 선언형 인프라(declarative infrastructure), 제어 이론의 control loop(목표값-측정값-보정).
- **하위/연관 개념**: Pod(같은 IP·볼륨·운명을 공유하는 컨테이너 묶음) → 볼륨을 어떻게 붙이는지는 [K8s 볼륨 모델 — emptyDir/PV/PVC/AccessMode](/cs/kubernetes-volume-model-pv-pvc-accessmode/) 에서 이어진다. label/selector, Endpoints, kube-proxy.
- **비교 대상**: watch(이벤트 기반) 모델은 파이썬 [WSGI·ASGI·async 스택](/cs/python-web-server-async-stack/) 의 event loop 와 같은 "폴링 말고 이벤트로 반응" 철학을 공유한다. 또 그 글의 *"경쟁자가 아니라 서로 다른 층으로 보라"* 는 사고방식은 여기 Deployment/ReplicaSet/Pod 3층을 이해하는 데 그대로 쓰였다.

## 7. 관련 Project
- [MISO sheet_analysis 를 EKS 로 이관하며 배운 것들](/project/miso-sheet-analysis-eks-migration/) — 이 글의 desired state·Deployment·Service 원리가 실제 EKS + ArgoCD 운영 환경에서 어떻게 쓰였는지.

<!-- 앞으로 쓸 글 후보 (그래프 엣지 보강용):
     - Ingress 가 속으로 어떻게 도는가 (Ingress Controller, L7 라우팅)
     - ConfigMap/Secret 이 Pod 에 주입되는 원리 (env, volume mount)
     - Pod 헬스체크 (liveness/readiness probe) 와 graceful shutdown
     - taint/toleration, NodeSelector, affinity (스케줄링 심화) -->
