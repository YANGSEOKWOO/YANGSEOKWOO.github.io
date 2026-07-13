---
layout: post
title: "Git 서브모듈: 부모 레포엔 '포인터'만 산다"
description: "서브모듈은 파일이 아니라 URL+커밋 해시라는 지시서다 — 로컬에선 되는데 CI에선 안 되는 이유"
category: Project
tags: [git, submodule, ci-cd, gitlab, monorepo]
date: 2026-07-08
---

## 한 줄 회고 (Feynman)
> 이 일을 처음 듣는 사람에게 한 문장으로 설명한다면?

> TODO: (본인 언어로) "서브모듈은 ___ 이다" 한 문장으로.

---

## 1. 문제 상황
- Docker 빌드가 `ERROR: miso-skills submodule missing`(Dockerfile 가드)로 실패.
- 플랫폼 스킬이 별도 레포(`miso-skills`)로 분리되어 **submodule**로 주입되는 구조로 바뀌면서 발생.
- 증상의 혼란 포인트:
  - **로컬 폴더엔 파일이 다 있는데** CI에서는 "없다"고 실패.
  - CI 워크플로우(yml)에는 `miso-skills`도 레포 URL도 **한 줄도 없는데** 어떻게 가져오는지 불명확.

## 2. 서브모듈의 실제 구조 (핵심 개념)

부모 레포는 서브모듈의 **파일을 담지 않는다.** "어느 레포의 어느 커밋을 여기 갖다놔라"는 **지시서 2개**만 저장한다.

| 정보 | 저장 위치 | 예시 |
|---|---|---|
| 어느 레포인가 (URL) | `.gitmodules` (일반 파일, 커밋됨) | `url = .../miso-skills.git` |
| 어느 커밋인가 (해시) | 부모 tree의 **gitlink** | `160000 commit 59a9112…` |

- `git ls-tree HEAD <path>` → `160000 commit <sha>` 가 보이면 그 경로는 파일이 아니라 **포인터**.
- `.git`도 폴더가 아니라 **파일**(`gitdir: ../../.git/modules/…`)로, 실제 git 데이터는 부모의 `.git/modules/` 안에 있다.

→ 그래서 `git clone`만 하면 서브모듈 폴더는 **텅 빈 채**로 오고, `git submodule update --init --recursive`를 해야 URL+해시를 조합해 실제 파일을 받는다.

## 3. "로컬은 되는데 CI는 안 되는" 이유

| | 로컬 | CI |
|---|---|---|
| 시작 상태 | 예전에 `submodule update`로 이미 채워둠 | **매번 빈 상태에서 새 clone** |
| 파일 출처 | 부모 레포엔 없음 → 원본 레포에서 받아야 | 동일 |
| 인증 | 내 계정(원본 org 협력자) | 기본 토큰(현재 레포 전용) → **다른 org private 접근 불가** |

- 부모 레포엔 파일이 없으므로 "로컬에 있음"은 CI에 아무 도움이 안 됨.
- 원본이 **다른 org의 private 레포**라, CI 기본 토큰으론 `Repository not found`(private엔 권한 없음도 "not found"로 응답).

## 4. 방향(URL) 바꾸기 — 두 가지 방법

원본에 접근 못 하는 환경(사내망 등)에서는 **미러를 만들고 그쪽을 보게** 한다.

| 방법 | 내용 | 트레이드오프 |
|---|---|---|
| `.gitmodules` URL 직접 수정 | `git submodule set-url` 후 커밋 | 상위 레포에서 그 파일을 바꾸면 **머지 충돌** |
| CI에서 `insteadOf` 재작성 | `git config url."<mirror>".insteadOf "<origin>"` | tracked 파일 불변 → **충돌 0** |

- 미러는 **파일 복사가 아니라 git 히스토리 미러**여야 한다 (`git clone --mirror` + `git push --mirror`) — 그래야 gitlink가 가리키는 **커밋 해시가 보존**된다. 단순 복사는 새 해시 → 포인터 깨짐.
- `.gitmodules`는 **git 표준 파일**이라 GitHub/GitLab 어디서든 호환. 바꾸는 건 "이사"가 아니라 **가리키는 주소만 교체**하는 것.

## 5. CI마다 다른 "서브모듈 받기" 설정 (Docker는 fetch 안 함)

**중요**: Dockerfile은 `COPY . .`로 **디스크에 이미 있는 파일을 복사만** 한다 (git 명령 없음). 그래서 서브모듈을 **받아오는 책임은 CI 체크아웃 단계**에 있다.

| 환경 | 서브모듈 받는 설정 |
|---|---|
| GitHub Actions | `actions/checkout` + `submodules: recursive` + (크로스 레포면) `token:` |
| Jenkins | `checkout scmGit(extensions:[submodule(parentCredentials:true, trackingSubmodules:true)])` |

- Jenkins `parentCredentials: true` → 서브모듈도 **부모와 같은 자격증명**으로 클론 → URL이 그 자격증명으로 접근 가능한 곳이어야 함.
- `trackingSubmodules: true` → 고정 해시가 아니라 **브랜치 tip**을 추적(`--remote`). 미러 브랜치만 최신이면 됨.

## 6. 예상 vs 실제

| | 예상 | 실제 |
|---|---|---|
| 로컬에 파일 있음 | CI도 당연히 될 것 | 부모 레포엔 포인터만 → CI는 별도로 받아야 함 |
| `submodules: recursive`만 켜면 됨 | 통과할 것 | 인증 없어 `Repository not found` → 토큰 필요 |
| 빌드가 성공 중이니 문제 없음 | 안심 | **스킬 들어오기 전** 빌드였음 → 다음 빌드에서 터질 예정 |
| `.gitmodules` 로컬 수정 | UI/CI에 반영될 것 | **커밋·push** 안 하면 origin은 옛 URL 그대로 |

> **가장 빗나간 지점은?** ← 여기에 학습이 있음

> TODO: 위 4개 중 본인에게 가장 의외였던 것 + 왜 그렇게 오해했는지 한 줄.

## 7. 추후 개선 방향
- URL 직접수정 대신 **상대경로 서브모듈**(`../miso-skills.git`, 같은 GitLab 인스턴스 자동 해석) 또는 `insteadOf`로 머지 충돌 원천 차단 검토.
- 미러 최신화 자동화(예: GitLab Pull Mirroring)로 "해시가 미러에 없음" 실패 예방.
- 재발 신호: 상위(52g)에서 `.gitmodules`를 건드리는 커밋이 내려오면 충돌/포인터 점검.

## 8. 관련 CS 지식
> TODO: 아래 후보 중 승인된 것만 남기기
- [Kubernetes: 선언적 상태와 Reconciliation](/cs/kubernetes-desired-state-reconciliation/) — "원하는 상태(고정 커밋 포인터) vs 실제(체크아웃 내용)"의 분리라는 점에서 유사
- (후보) [DB Connection Pool을 밑바닥부터](/cs/db-connection-pool-from-scratch/) — 간접 참조/핸들 vs 실체 구분 관점
