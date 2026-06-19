---
layout: post
title: "MISO sheet_analysis 를 EKS 로 이관하며 배운 것들"
description: "Docker-compose 시절 코드를 EKS + EFS + ArgoCD 환경에 끼워맞추며 마주친 K8s/AWS/Celery 패턴들"
category: Project
tags: [EKS, CDK, DinD, EFS, ArgoCD]
date: 2026-05-07
---

## 한 줄 회고 (Feynman)
> TODO: 이 일을 처음 듣는 동료에게 한 문장으로 어떻게 설명할까? (직접 채워보기)

---

## 1. 문제 상황

`miso_dev` (MISO 개발 환경, EKS) 에 sheet_analysis 도구를 동작시켜야 했음.
sheet_analysis 는 사용자가 업로드한 Excel/CSV 를 Claude Agent SDK 로 분석하는 기능으로,
원래 docker-compose 환경 (host dockerd 에 직접 `docker run`) 가정으로 만들어진 코드.

EKS 환경의 제약:
- 컨테이너 런타임이 **containerd** → host dockerd 자체가 없음. host docker.sock 마운트 불가.
- worker Pod 는 자기 컨테이너 안에서 직접 `docker run` 명령이 안 먹힘.
- 업로드 파일은 **S3** (`STORAGE_TYPE=s3`) 에 저장되는데, sheet_analysis 코드는 로컬
  파일시스템만 가정.
- 팀 원칙: **모든 인프라 변경은 IaC(CDK)** 로만.

이걸 EKS 위에서 돌게 만드는 게 목표였음.

## 2. 가능했던 해결책들

### Docker spawn 방식
| 옵션 | 장점 | 단점 |
|---|---|---|
| A. DinD 사이드카 | docker-compose 시절 코드 그대로 동작 | privileged 필요, 보안 ↓ |
| B. K8s Job-per-task | "K8s native", privileged 불필요 | sheet_analysis 코드 대거 수정 |
| C. host docker.sock hostPath | 단순 | EKS 는 containerd 라 docker.sock 자체가 없음 (불가) |

### 파일 공유 방식
| 옵션 | 장점 | 단점 |
|---|---|---|
| A. EFS PVC (RWX) | 다중 Pod 동시 mount, 코드 그대로 | EFS 비용·지연 |
| B. EBS PVC (RWO) | 빠름 | 단일 Pod 만 mount → api/worker 공유 불가 |
| C. S3 + storage abstraction | 클라우드 native | 코드 패치 필요 |

### ECR repo 명명 컨벤션
| 옵션 | 장점 | 단점 |
|---|---|---|
| A. 처음 직관 (`miso/sheet-analysis-main` 슬래시) | "miso 그룹" 으로 묶임 | self-hosted api 코드(`miso-sheet-analysis-main` 대시) 와 불일치 |
| B. self-hosted 코드 컨벤션 (대시) | 코드와 명확히 일치, env 1개(`ECR_REPO`)로 path 계산 | replace 필요 |

## 3. 현실적 타협점

- **나는 EKS/AWS 학습 단계** — 이론(개념 → 한 줄 박스) → 실습(CDK 작성 → deploy)으로 단계별 진행이 필요했음.
- **dev 환경**이라 IMMUTABLE 의 안전성보다 `:latest` 갱신 편의가 더 가치 있었음. prd 는 분기로 별도.
- **self-hosted 는 별도 레포** (`gsn-miso-self-hosted`) — `_download_input_files` 의 storage abstraction gap 같은 *코드 버그* 는 본 작업 범위 밖, 별도 PR.
- **권한 제약** — `siwoo-api` 가 ECR 이미지 삭제 권한 없음 → CDK `empty_on_delete=True` 옵션으로 우회.
- **GitOps** — kubectl apply 직접 실행은 ArgoCD `selfHeal` 이 revert. 모든 변경은 git commit + push.

## 4. 왜 이게 최선이었나

- **DinD 선택**: sheet_analysis 코드를 안 건드리고 동작. Eugene 팀(self-hosted 운영팀) 도 동일 패턴이라는 확인을 슬랙으로 받음.
- **EFS 선택**: api Pod 가 `inputs/outputs` 디렉터리 만들고 worker Pod 가 같은 절대 경로로 보는 흐름 → 두 Pod 동시 mount 가 필수. EBS RWO 로는 불가능.
- **ECR 대시 컨벤션 (B)**: api 코드의 `SHEET_ANALYSIS_MAIN_IMAGE` property 가
  `{ECR_REPO}/miso-sheet-analysis-main:{tag}` 형식으로 path 를 *계산* 함. 코드 안 건드리려면 그 컨벤션 따라가는 게 비용 가장 낮음.
- **MUTABLE + `:latest` only (dev)**: 빠른 iteration > IMMUTABLE 의 drift 방지. config 분기로 prd 는 IMMUTABLE 유지.

## 5. 예상 vs 실제

| | 예상 | 실제 |
|---|---|---|
| 결과 | 매뉴얼 따라하면 한 번에 동작 | 4단계 버그 cascade |
| 소요 시간 | 1~2일 | 누적 학습 + 디버깅 며칠 |
| 부작용 | (없음) | self-hosted 의 `_download_input_files` 가 storage abstraction 안 쓰는 버그 발견 |

> **가장 빗나간 지점은?**
>
> "ECR 에 이미지 push 하고 `worker.yaml` env 에 `SHEET_ANALYSIS_MAIN_IMAGE` 박으면 끝"
> 이라고 가정했는데, 코드는 그 env 를 **안 읽고** `ECR_REPO + IS_QA` property 로 path 를
> *계산*하고 있었음. **env var 의 단순 주입이 아니라, 코드의 명명 컨벤션과 정확히 맞춰야**
> 한다는 게 가장 큰 학습.
>
> 그리고 매 단계마다 "다음엔 작동하겠지" 했는데, **에러가 한 단계씩 더 안쪽으로 밀려나는**
> 양파 까는 디버깅이었음. queue → image → directory → file 순으로 숨어있던 4단계 버그.

## 6. 학습 포인트 정리

### K8s 기본 개념
- **Pod 와 Container 차이**: Pod 안 컨테이너끼리는 **Network/IPC/UTS namespace 만 공유**, Mount/PID/Cgroup 은 분리. 그래서 sidecar 끼리 `localhost` 로 통신은 되지만 파일시스템은 PVC 등으로 따로 mount 해야 함 (이게 Phase 5 에서 sheet-analysis-shared PVC 를 양쪽에 같은 경로로 mount 한 이유).
- **PSS (Pod Security Standards)**: privileged 컨테이너를 차단하는 namespace label. `miso` namespace 는 미설정이라 DinD `privileged: true` 가 막히지 않음 (덕분에 동작).
- **IRSA**: K8s ServiceAccount 가 IAM Role 을 assume. `miso-sa` 가 ECR pull 권한 갖는 방식.
- **EFS CSI vs EBS CSI**: AccessMode `RWX` (Multi-AZ 동시 mount) vs `RWO` (단일 노드만). 우리 케이스는 RWX 필수라 EFS.

### CDK / CloudFormation 동작
- **Replacement vs In-place update**: `repository_name` 변경은 replacement (delete + create), `image_tag_mutability` 만 변경하면 in-place. `cdk diff` 가 `replace` 로 표시되면 기존 리소스 삭제 동반.
- **`empty_on_delete=True`**: ECR Repository 삭제 시 이미지를 CDK 가 자동으로 비움 (custom resource Lambda). 단, *이번 deploy 로 새로 등록되는* hook 이라 **첫 등록 시점부터** 효과 발생. 우리 케이스에선 구 repo 가 이전 등록 시 옵션 없었어서 cleanup 자동 비움이 안 먹혔음.
- **CFn cleanup phase**: 새 리소스 생성 → 구 리소스 삭제 순서. 삭제 실패는 stack 을 `UPDATE_COMPLETE_CLEANUP_FAILED` 로 만들지만 **새 리소스는 살아있음** (다음 작업 진행 가능).

### ECR 인증 in DinD
- `apk add aws-cli` → Alpine musl libc + python ABI 충돌 (`pyexpat` symbol). **Alpine 에서 aws-cli 쓰지 말 것**.
- `amazon-ecr-credential-helper` Go binary (의존성 0, GitHub release) 가 정답. `~/.docker/config.json` 의 `credHelpers` 등록만 하면 `docker pull/push` 시점마다 자동 인증, 12h 토큰 만료 신경 X.

### Cross-platform Docker build
- Mac arm64 에서 linux/amd64 EKS 노드용 이미지 만들려면 `docker buildx build --platform linux/amd64 --load` (QEMU emulation 자동). `--load` 빠지면 buildx cache 에만 남고 `docker images` 에 안 보임.

### Celery 워크로드의 데이터 분리 패턴
- broker (Redis/RabbitMQ) 메시지에는 **file_id (UUID) 만** 전달. 파일 byte 는 broker 로 안 보냄 (사이즈 제한, retry 비용).
- worker 는 file_id 로 storage 에서 fetch. **storage 가 어디 있느냐** (local/S3/EFS) 는 storage abstraction 이 추상화해야 함.
- 코드가 storage abstraction 안 쓰고 로컬 파일시스템에 직접 접근하면 → S3 환경에선 깨짐. sheet_analysis 의 `_download_input_files` 가 정확히 그 케이스.

### ConfigMap 갱신과 Pod
- `envFrom: configMapRef` 는 **Pod 시작 시점에 1회 주입**. ConfigMap 을 변경해도 기존 Pod 의 env 는 그대로.
- 반영하려면 `kubectl rollout restart deployment/...`. 단 **ArgoCD sync 가 아직 안 된 시점에 restart 하면** 옛 ConfigMap 으로 새 Pod 가 다시 뜸 → 한 번 더 restart 필요.

### GitOps + ArgoCD 운영
- `kubectl apply` 로 직접 매니페스트 수정하면 ArgoCD `selfHeal` 이 다음 sync 때 revert. **모든 변경은 git commit + push** 경유.
- 자동 sync 폴링은 보통 3분. 급하면 ArgoCD UI 에서 manual Refresh + Sync.

### 디버깅 양파 까기 — 5단계 cascade

1. **queue mismatch**: `sheet_analysis_task` 가 `code_interpreter` queue 로 enqueue 되는데 worker 의 `CELERY_QUEUES` 에 없어 task pickup 자체가 안 됨 → ConfigMap 에 `code_interpreter` 추가.
2. **image name mismatch**: env `SHEET_ANALYSIS_MAIN_IMAGE` 가 무시되고 코드가 `ECR_REPO` 분기로 path 를 *계산* → ECR repo 이름을 코드 컨벤션(대시 + `:latest`) 에 맞춤.
3. **directory not found**: api Pod 가 sheet-analysis-shared PVC 를 마운트 안 해서 api 가 만든 `inputs/outputs` 디렉터리를 worker 가 못 봄 → `api.yaml` 에도 PVC mount + ConfigMap `STORAGE_LOCAL_PATH` 통일.
4. **file not found**: `_download_input_files` 가 storage abstraction 안 쓰고 로컬 경로만 봄 → `STORAGE_TYPE=s3` 환경에서 S3 파일을 못 가져옴. ✅ self-hosted PR (`GSN-FORK-PATCH(SHEET-S3)` commit `e6c13c37d`, `e0567c701`) 으로 `storage.load()` 패턴 적용 — dev cluster 검증 완료.
5. **SSE streaming 깨짐**: 분석 결과는 정상이지만 사용자 화면에 progress chunk 가 안 뜨고 결과만 한 번에 표시됨. main 컨테이너의 ClaudeApp 가 `REDIS_HOST` 를 `host.docker.internal` (K8s 비호환) 로 가정 → cluster redis 못 찾아 stream publish 실패. ✅ self-hosted PR (`GSN-FORK-PATCH(SHEET-REDIS-HOST)` commit `c0524d960`) 로 docker_cmd 에 `-e REDIS_HOST=...` 명시 주입 — dev cluster 에서 SSE 정상 작동 검증.

매 단계마다 "끝났다" 싶었지만 한 단계씩 더 안쪽으로 밀려난 패턴. **로그를 신뢰하고, 한 번에 한 가지만 고치는** 자세가 옳았음. 그리고 *분석 결과 정상* 과 *UX 완성* 은 다른 차원의 검증임을 5단계에서 배웠음 (1~4 까지는 결과 정상이면 OK 였지만, 5단계는 사용자 경험 영역).

### 인프라 정의 vs 데이터 정리
"IaC only" 원칙이 적용되는 건 *인프라 리소스 정의* (Repository/Bucket/Role 의 존재·속성). *그 안의 데이터* (ECR 이미지, S3 객체) 의 일회성 정리는 **콘솔/CLI 가 합리적**. 데이터 정리를 위해 IAM 정책 추가 PR 만드는 식의 over-engineering 은 비용만 늘림 — 이건 작업 중 사용자가 직접 지적해준 부분.

### TASK_REDIS_HOST vs REDIS_HOST — 두 코드베이스 합쳐서 생긴 design 불일치

self-hosted 는 사실 **두 코드베이스의 합체** — `api/` (Flask + Celery) 와 `tools-external/sheet_analysis/src/` (분석 launcher, 별도 docker 이미지). 두 코드베이스가 *역사적으로 따로 자라서* 같은 의미의 변수에 다른 이름을 씀:
- `api/configs/extra/docker_task_config.py` → `TASK_REDIS_HOST` (Celery task-level)
- `tools-external/sheet_analysis/src/` → `REDIS_HOST` (main 컨테이너 안 ClaudeApp)

같은 cluster redis 를 가리키지만 코드 path 별로 *다른 env 이름* 을 읽음. ConfigMap 의 envFrom 만으론 main 컨테이너가 자동으로 받지 못해서, docker_cmd 에 `-e REDIS_HOST=...` 를 *명시 주입* 해야 함.

**일반화된 교훈**: 합쳐진 시스템에서는 "같은 의미인데 다른 변수 이름" 의 *configuration sprawl* 이 흔함. 새 환경에 이식할 때 *어느 코드가 어느 변수를 보는지* 매핑부터 해야 함.

### "main 컨테이너" 의 두 가지 의미 — 용어 충돌

이번 작업의 가장 큰 *의사소통* 함정은 **"main container" 가 두 곳에서 다른 의미** 였다는 것:

| 컨텍스트 | "main" 의 의미 | 위치 |
|---|---|---|
| K8s/Pod 레벨 | Pod 의 주 워크로드 컨테이너 = **Celery worker** | Pod 안, dind sidecar 와 형제 |
| sheet_analysis 내부 | 분석 launcher (`claude-code-app` image) | **dind 안에 nested** spawn |

두 main 은 *계층* 이 다름. K8s 레벨 main 이 sheet_analysis main 의 부모 (worker 가 dockerd 명령으로 spawn). 같은 단어가 헷갈리게 함. 용어 정리하지 않으면 디버깅이 미궁.

### dind + K8s service DNS 호환성 — 검증된 사실

이번에 의외로 *우연히* 검증된 것: **dind 안의 default bridge network 안에서도 K8s service DNS (`miso-redis-svc.miso.svc.cluster.local`) 가 resolve 됨**. 이전에는 `--network host` 추가 패치가 필요할 수도 있다고 걱정했지만, REDIS_HOST env 만 주입했더니 정상 동작.

추정: dind 가 worker Pod 의 `/etc/resolv.conf` 를 상속해서 nested 컨테이너에도 그 DNS resolver 를 전달함. 결과적으로 nested 컨테이너의 정의되지 않은 도메인이 K8s coredns 로 forwarding 됨.

→ K8s + dind 환경에서 K8s service 와 nested 컨테이너 간 통신은 *공식 문서엔 명시 안 됐지만* 실제로는 잘 작동. 단 안정성 측면에서 변할 수 있는 가정이라 *production* 가서는 명시적 `--network host` 또는 `--add-host` 로 hardening 권장.

### Bedrock fallback — self-hosted 자동 동작

Anthropic API 의 rate limit (`30,000 input tokens/min`) 에 부딪힌 케이스에서, 우려와 달리 self-hosted 가 **자동으로 Bedrock backend 로 fallback** 함:

```
✅ Anthropic API key injected → 429 rate_limit
   (이후 재시도)
✅ Bedrock credentials injected → 정상 처리
```

`bedrock_api_key` + `bedrock_region` 이 task argument 로 전달되면 self-hosted launcher 가 자동 우선 사용. 이건 plan 문서에 명시되지 않았던 *발견된 fallback 메커니즘* 으로, dev 환경에서 안정성 측면 큰 이점.

### GSN-FORK-PATCH 마커 컨벤션 — upstream fork patch 추적

self-hosted 는 *upstream(52g) 을 주기적으로 merge* 받는 fork 구조. 우리가 한 환경별 패치 (S3 호환, K8s 호환) 가 머지 시 충돌하지 않게 *추적 가능한 마커* 가 필요함.

채택한 컨벤션:
```python
# 한 줄 패치: prefix 단일 라인
# GSN-FORK-PATCH(SHEET-REDIS-HOST): K8s 환경 호환
docker_cmd.extend(["-e", f"REDIS_HOST={miso_config.REDIS_HOST}"])

# 다중 라인 패치: 시작/끝 블록 마커
# >>> GSN-FORK-PATCH(SHEET-S3): storage abstraction 사용 ─ STORAGE_TYPE=s3 환경 호환 <<<
from extensions.ext_storage import storage
...
# <<< /GSN-FORK-PATCH(SHEET-S3) >>>
```

장점:
- `git grep "GSN-FORK-PATCH"` 로 모든 우리 fork divergence 한 번에 발견
- 패치별 ID (예: `SHEET-REDIS-HOST`) 로 분류
- 머지 conflict 시 블록 통째로 옮기거나 upstream 새 코드에 재적용 쉬움
- upstream PR 머지되면 마커만 제거 (코드는 upstream 의 것 그대로)

초안엔 사유·회귀·후속 3섹션을 다 박았다가 사용자 피드백 ("주석이 고봉밥") 으로 단일 라인으로 축소. **코드 가독성 > 사유 보존** 의 trade-off 가 마커 컨벤션 결정.

## 7. 추후 개선 방향

- ✅ **`_download_input_files` 의 storage abstraction 추가** (self-hosted 레포 PR): commit `e6c13c37d`, `e0567c701` 로 patch 완료. `storage.load()` 패턴 적용, dev cluster 검증 끝.
- ✅ **SSE streaming K8s 호환** — commit `c0524d960` 로 `REDIS_HOST` docker_cmd 주입 패치. dev cluster 에서 progressive 메시지 정상 동작 검증.
- **EFS → S3 완전 전환** (별도 plan 문서로 분리): `sheet-analysis-efs-to-s3-migration-plan.md` 에 4 Phase 정리. base_path 흐름을 emptyDir 기반으로 refactor 하면 EFS 자체 불필요해짐. 현재 dev 가 동작 중이라 백로그.
- **52g upstream 에 `GSN-FORK-PATCH` 들 PR**: storage abstraction + REDIS_HOST 둘 다 *알려진 design gap* 이므로 upstream 도 환영할 가능성. merge 되면 fork patch 자연 제거.
- **prd 환경 분기 검증**: 현재 dev 는 MUTABLE + DESTROY + `empty_on_delete=True` 인데, prd 는 IMMUTABLE + RETAIN. 첫 prd 배포 시 마이그레이션 plan 필요.
- **DinD 보안 강화**: privileged 가 여전히 필요. multi-tenancy 환경 (PSS=restricted) 으로 가면 차단됨 → 그때는 K8s Job-per-task 로 재설계.

> TODO: 시간이 더 있다면 어떻게 다시 풀 것인가? (직접 채워보기)
>
> TODO: 어떤 시그널이 보이면 재설계가 필요한가? (직접 채워보기)

## 8. 관련 CS 지식

- [Kubernetes Volume Model — PV/PVC, AccessMode](/cs/kubernetes-volume-model-pv-pvc-accessmode/) — `sheet-analysis-shared` PVC 에 `RWX` (ReadWriteMany) 를 골라야 했던 이유. EFS 라 다중 Pod (api + worker) 동시 mount 가능했고, EBS 였으면 `RWO` 라 불가능했을 것.
