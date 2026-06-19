---
layout: post
title: "K8s 볼륨 모델 — emptyDir, PV, PVC, AccessMode 의 진짜 의미"
description: "Persistent 는 '영구 보존' 이 아니고, RWX 는 Linux rwx 와 다른 약자다 — K8s 볼륨 추상화의 의도를 다시 정리한 노트"
category: CS
tags: [Kubernetes, Volume, PVC, Storage, Infra]
date: 2026-04-30
---

## 한 줄 정의 (Feynman)
> K8s 볼륨은 "어떤 컨테이너가, 어떤 디스크를, 얼마나 오래, 몇 명이 동시에 쓸지" 를 분리해서 다룰 수 있게 만든 추상화 계층이다.

---

## 1. 핵심 내용 / 구조

K8s 가 컨테이너에 디스크를 붙여줄 때, 실제 스토리지(EFS, NFS, 노드 디스크 등) 와 Pod 사이에는 여러 계층이 끼어든다. 처음엔 그냥 "PVC = 디스크" 라고 외우면 될 것 같았는데, 각 계층이 다른 역할을 한다는 걸 정리하지 않으면 트러블슈팅이 안 된다.

```text
실제 스토리지 (EFS, NFS, 노드 디스크 등)
   ↓ 드라이버가 인식
CSI Driver (예: efs.csi.aws.com)
   ↓
StorageClass — "어떤 스토리지를 어떻게 빌려줄지" 규칙
   ↓ PVC 요청 들어오면 자동 생성
PV (PersistentVolume) — 실제 K8s 볼륨 객체
   ↓ PVC 가 PV 를 바인딩
PVC (PersistentVolumeClaim) — "RWX 50Gi 주세요" 라는 요청서
   ↓ Pod spec.volumes 에서 참조
Pod 의 컨테이너 → /data 에 마운트
```

핵심 추상화는 **세 가지**:

| 분리 축 | 결정 주체 | 예시 |
|---|---|---|
| 어떤 스토리지인가 | StorageClass / CSI Driver | EFS, EBS, NFS, Longhorn |
| 얼마나 오래 사는가 | reclaimPolicy + 볼륨 종류 | `emptyDir`, `Retain`, `Delete` |
| 몇 명이 동시에 쓰는가 | AccessMode | RWO, ROX, RWX, RWOP |

이 세 축이 따로 결정된다는 게 핵심이다. "EFS = RWX = 영구" 처럼 묶어서 외우면 다른 조합이 들어왔을 때 헷갈린다.

## 2. 무엇을 배웠나

- **추상화 계층의 의도**: PV/PVC 분리는 "스토리지 운영자(인프라) ↔ 사용자(개발자)" 의 책임 분리다. 인프라가 StorageClass 를 미리 깔아두면, 앱 개발자는 PVC 만 선언하면 된다.
- **emptyDir 의 라이프사이클**: 컨테이너 재시작은 견디지만 Pod 재스케줄은 못 견딘다. "Pod" 와 "컨테이너" 의 라이프사이클이 다르다는 걸 처음으로 명확히 구분하게 됐다.
- **AccessMode 는 "용량" 이 아니라 "동시성" 정책**: RWX 가 비싼 이유는 용량이 커서가 아니라, 여러 노드에서 동시에 R/W 가능한 백엔드(파일시스템 기반)를 요구하기 때문이다.
- **블록 vs 파일 vs 객체** 가 AccessMode 지원에 직결: EBS(블록) → RWO 만, EFS(파일) → RWX 가능, S3(객체) → 애초에 PVC 모델이 안 맞음.

## 3. 내가 오해하고 있던 것

> 학습 효과가 가장 큰 부분.

### 오해 1. "Persistent" = "영구 저장"

- **잘못 알고 있던 점**: PV 의 "Persistent" 를 "데이터를 영원히 보존" 으로 읽었다. 그래서 "분석 끝나면 S3 에 올리고 지울 임시 파일에 왜 PV 를 쓰지?" 라는 의문이 생겼다.
- **정정된 이해**: Persistent 는 **"Pod 라이프사이클과 분리"** 라는 뜻이다. PV 자체는 Pod 가 죽어도 살아있지만, `reclaimPolicy: Delete` 면 PVC 삭제 시 사라진다. 영구성은 `reclaimPolicy` 가 결정한다.
- **왜 헷갈렸나**: 영어 단어의 일상적 의미("영원한") 가 K8s 의 기술적 의미("Pod 보다 오래 사는") 보다 훨씬 강하게 들어와 있었다. 그리고 emptyDir 라는 "비-Persistent" 옵션을 모르고 있었기 때문에, 비교 대상이 없어서 Persistent 의 의미가 좁혀지지 않았다.

### 오해 2. RWX = Linux 의 `rwx`

- **잘못 알고 있던 점**: K8s 매니페스트에서 `accessModes: [ReadWriteMany]` 를 보고도, 약자 RWX 를 따로 보면 자연스럽게 Linux 파일 권한 `rwx` (read/write/execute) 로 읽었다.
- **정정된 이해**: K8s RWX = **ReadWriteMany**, 여러 노드에서 동시에 R/W 가능하다는 **동시성 모드**다. 권한 비트(read/write/execute) 와는 무관하다.
  - RWO = ReadWriteOnce (단일 노드)
  - ROX = ReadOnlyMany (여러 노드 읽기만)
  - RWX = ReadWriteMany (여러 노드 R/W)
  - RWOP = ReadWriteOncePod (단일 Pod)
- **왜 헷갈렸나**: 약자가 우연히 같다. Linux 파일 권한은 워낙 친숙해서, 같은 글자를 보면 자동으로 그쪽으로 매핑된다. **풀네임(ReadWriteMany)을 외우는 게 약자(RWX)를 외우는 것보다 훨씬 안전**하다는 교훈.

## 4. 언제 쓰고, 언제 쓰지 않는가 (Trade-offs)

### 볼륨 종류 선택

| 상황 | 적합도 | 이유 |
|---|---|---|
| 같은 Pod 안 컨테이너끼리만 임시 공유 (캐시, scratch) | ✅ `emptyDir` | 가장 단순, 공짜, CSI 불필요 |
| 분석 끝나면 결과를 S3 에 업로드하고 작업 영역은 버려도 됨 | ✅ `emptyDir` | "영구" 가 필요 없음 — Pod = task 단위면 충분 |
| 여러 Pod·노드가 같은 파일을 공유 (HPA, 분산 처리) | ✅ RWX PVC | emptyDir 로는 Pod 간 공유 불가 |
| Pod 재기동 시 진행 중 데이터 보존 | ✅ PV (RWO 도 가능) | emptyDir 는 Pod 재스케줄 시 유실 |
| 단일 Pod 모델인데 RWX PVC 사용 | ⚠️ over-spec | 비용·복잡도 오버킬, emptyDir 로 충분 |
| 객체 스토리지 모델이 더 자연스러운 데이터 흐름 | ❌ PVC 자체 부적합 | S3 SDK 직접 사용이 깔끔 |

### AccessMode × 백엔드

| 백엔드 | RWO | RWX |
|---|---|---|
| AWS EBS | ✅ | ❌ |
| AWS EFS | ✅ | ✅ |
| GCP PD | ✅ | ❌ |
| GCP Filestore | ✅ | ✅ |
| Azure Disk | ✅ | ❌ |
| Azure Files | ✅ | ✅ |
| NFS | ✅ | ✅ |
| Longhorn | ✅ | ✅ (NFS provisioner 경유) |
| Ceph RBD | ✅ | ❌ |
| CephFS | ✅ | ✅ |

블록 스토리지는 본질적으로 RWX 불가, 파일시스템 기반만 RWX 지원. **"EFS 를 쓰고 싶다" 가 아니라 "RWX 가 필요하다 → 그래서 파일시스템 백엔드 → 그래서 EFS"** 순서로 결정 흐름이 가야 한다.

## 5. 직접 실험

> TODO: 다음 실험을 직접 돌려서 결과 추가
>
> 1. 같은 Pod 의 두 컨테이너가 `emptyDir` 로 `/data` 공유 — A 가 쓰고 B 가 읽기
> 2. Pod 삭제 후 재생성 → `emptyDir` 데이터 사라지는지 확인
> 3. 컨테이너만 재시작 (`kubectl exec` 로 PID 1 kill) → `emptyDir` 데이터 유지되는지 확인
> 4. RWO PVC 를 Deployment replica=2 로 띄우면 어떤 에러가 나는지 (서로 다른 노드에 스케줄됐을 때)

실험 후 채울 것: "예상대로 동작했나? 의외였던 점은?"

## 6. 다른 개념과의 연결

- **상위 개념**: K8s 의 "스펙 ↔ 실제 리소스" 추상화 — Pod/Deployment 도 같은 패턴 (Selector → ReplicaSet → Pod). PVC ↔ PV 도 같은 추상화.
- **하위/연관 개념**:
  - CSI (Container Storage Interface) — 백엔드 플러그인 표준
  - StorageClass — 동적 프로비저닝 정책
  - reclaimPolicy — 볼륨 회수 시점
- **비교 대상**:
  - **Linux bind mount**: K8s 볼륨이 아닌, 컨테이너 런타임(dockerd 등) 이 직접 거는 마운트. K8s 가 모르는 영역. DinD 패턴에서 sandbox 컨테이너에 데이터를 넘기는 건 PVC 가 아니라 이 bind mount 다.
  - **객체 스토리지(S3, MinIO)**: PVC 모델과는 다른 패러다임. 파일시스템 인터페이스가 아니라 SDK 호출. RWX 같은 "동시성" 개념 자체가 다르게 풀린다 (eventual consistency 등).

## 7. 관련 Project

> TODO: sheet_analysis 이관 의사결정 글 작성 후 링크 추가
