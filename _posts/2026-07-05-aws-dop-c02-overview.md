---
layout: post
title: "AWS DOP-C02 총괄 — 시험 구조와 6개 도메인 지도"
description: "DevOps Engineer Professional 학습 시리즈의 허브 글. 공식 가이드로 검증한 시험 구조와 6개 도메인 지도, 그리고 도메인별 글 인덱스."
category: Certificate
tags: [AWS, DOP-C02, DevOps, 자격증]
date: 2026-07-05 00:00:00 +0900
---

## TL;DR
- **DOP-C02 = AWS Certified DevOps Engineer – Professional.** "코드를 실제 서버까지 자동으로, 안전하게, 죽지 않게 굴리는" 능력을 검증하는 프로 등급 시험이에요.
- 이 글은 **시리즈의 총괄(허브)** 글이에요. 시험 구조 + 6개 도메인 지도만 잡아두고, 각 도메인은 별도 글에서 강의처럼 하나씩 파고듭니다.
- 아래 수치는 전부 **공식 Exam Guide(v1.6)를 직접 대조해서 확인**한 값이에요. (검증 얘기는 맨 아래에 따로 적어뒀어요.)

---

## 1. 시험 한눈에

| 항목 | 값 |
|---|---|
| 정식 명칭 | AWS Certified DevOps Engineer – Professional (DOP-C02) |
| 문항 수 | **75문항** (채점 65 + 비채점 10) |
| 문제 유형 | 객관식(정답 1 / 오답 3), 복수응답(정답 2+ / 보기 5+) |
| 시간 | **180분** |
| 합격선 | **750 / 1000** (100–1000 스케일) |
| 채점 방식 | 보상형(compensatory) — 섹션별 합격 필요 없이 **전체 합산**으로만 판정 |
| 응시료 | $300 USD |
| 권장 경력 | AWS 프로비저닝·운영·관리 **2년 이상** + SDLC·스크립팅 경험 |

> 💡 "비채점 10문항"이 포인트예요. 75문제를 다 풀지만 그중 10개는 AWS가 미래 출제용으로 테스트하는 거라 점수에 안 들어가요. 근데 **어느 게 비채점인지 안 알려주니** 결국 75개 다 성실히 풀어야 해요. 그리고 **찍기 감점이 없으니** 모르는 것도 무조건 답을 골라야 하고요.

---

## 2. 6개 도메인 지도

시험은 6개 도메인으로 나뉘고, 비중이 곧 **공부 우선순위**예요. Domain 1(CI/CD)이 22%로 제일 크고, 보안이 17%로 그다음이에요.

| # | 도메인 | 비중 | 핵심 서비스 | 이 시리즈에서 다룰 것 |
|---|---|---|---|---|
| 1 | **SDLC Automation** (CI/CD) | 22% | CodePipeline · CodeBuild · CodeDeploy · CodeCommit · CodeArtifact · ECR | 파이프라인 구조, 배포 전략(blue/green·canary·rolling), 테스트 자동화, 아티팩트 |
| 2 | **Config Management & IaC** | 17% | CloudFormation · CDK · SAM · StackSets · Systems Manager · Config · Organizations · Control Tower | IaC 사고방식, 멀티계정 자동화, 거버넌스 |
| 3 | **Resilient Cloud Solutions** | 15% | Auto Scaling · ELB · Route 53 · RDS/Aurora · DynamoDB Global Tables · 백업/DR | 고가용성 설계, DR 전략(RTO/RPO), 자가치유 |
| 4 | **Monitoring & Logging** | 15% | CloudWatch(Logs·Metrics·Alarms·EMF) · X-Ray · CloudTrail · EventBridge | 지표·로그·추적 수집과 대시보드/알람 |
| 5 | **Incident & Event Response** | 14% | EventBridge · CloudWatch Alarms · SNS · Lambda · SSM Automation | 이벤트 기반 자동 대응·자동 복구 |
| 6 | **Security & Compliance** | 17% | IAM · Secrets Manager · KMS · Config · Security Hub · GuardDuty · Inspector | 최소권한, 시크릿 관리, 규정 준수 자동화 |

합계 = 22 + 17 + 15 + 15 + 14 + 17 = **100%**. ✅

> 각 도메인 안은 다시 **Task Statement**(예: "1.1 Implement CI/CD pipelines")로 쪼개져요. 실제 공부는 이 Task Statement 단위로 하는 게 시험 가이드 구조와 딱 맞아요.

---

## 3. 도메인별 글 인덱스

강의·질의응답을 하나씩 진행하면서 글로 정리하고, 여기 링크를 채워 나갈게요. (완성되면 `/graph`로 이 총괄 글과 엣지를 이어줍니다.)

- [ ] **Domain 1 — SDLC Automation** 🚧 작성 예정
- [ ] **Domain 2 — Config Management & IaC** 🚧 작성 예정
- [ ] **Domain 3 — Resilient Cloud Solutions** 🚧 작성 예정
- [ ] **Domain 4 — Monitoring & Logging** 🚧 작성 예정
- [ ] **Domain 5 — Incident & Event Response** 🚧 작성 예정
- [ ] **Domain 6 — Security & Compliance** 🚧 작성 예정

---

## 4. 관련 글 (이미 이 블로그에 있는 것)

DOP 도메인은 이미 정리해 둔 글들과 자연스럽게 이어져요:

- [MISO sheet_analysis 를 EKS 로 이관하며 배운 것들](/project/miso-sheet-analysis-eks-migration/) — **Domain 3(복원력)·Domain 2(컨테이너)**의 실전 사례
- [K8s 핵심 동작 원리 — desired state 와 reconciliation loop](/cs/kubernetes-desired-state-reconciliation/) — **Domain 2**의 "선언적 IaC" 사고방식의 뿌리
- [K8s 볼륨 모델 — PV, PVC, AccessMode](/cs/kubernetes-volume-model-pv-pvc-accessmode/) — **Domain 3**의 상태 저장/스토리지
- [AWS Credential Chain 바닥부터](/cs/aws-ec2-credential-chain/) — **Domain 6(보안)**의 IAM 역할·자격증명
- [파이썬 백엔드 스택 바닥부터 (WSGI·ASGI·async)](/cs/python-web-server-async-stack/) · [Flask + Gunicorn + gevent 아키텍처](/cs/flask-gunicorn-gevent-architecture/) — CI/CD가 실제로 **배포하는 대상**

---

## 5. 사실 검증 노트 (왜 이 글의 수치를 믿어도 되나)

이 글을 쓰면서 시험 수치를 자동 요약 도구로 한 번 뽑아봤는데, **문항 수를 80개로, 도메인을 7개로 잘못 알려줬어요.** 그래서 [공식 Exam Guide PDF](https://d1.awsstatic.com/training-and-certification/docs-devops-pro/AWS-Certified-DevOps-Engineer-Professional_Exam-Guide.pdf)(Version 1.6, DOP-C02) 원문을 직접 읽어 대조했고, 위의 모든 수치는 그 원문 기준이에요.

> 교훈: 자격증 수치·비중은 **반드시 해당 시험의 공식 Exam Guide 최신 버전**으로 확인하기. AI 요약이든 블로그든 2차 자료는 버전이 섞이거나 다른 시험과 혼동되기 쉬워요.

---

## 진행 상황
> 공부하면서 직접 업데이트하는 칸.

- 시작일: 2026-07-05
- 목표 응시일: `> TODO`
- 현재 집중 도메인: Domain 1 (SDLC Automation)
