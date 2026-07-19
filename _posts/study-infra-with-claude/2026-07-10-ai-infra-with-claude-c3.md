---
layout: post
title: "[AI 인프라 모각코] 3장: 첫 번째 배포 파이프라인"
date: 2026-07-10 20:50:00 +0900
categories: [AI, Claude Code, GitOps]
tags: [Claude Code, GitHub Actions, ArgoCD, GitOps, Kubernetes, GCP, Artifact Registry]
published: true
---

# Claude Code로 GitHub Actions와 ArgoCD GitOps 파이프라인 구축해보기

최근 Claude Code를 사용해서 GitHub Actions와 ArgoCD를 연결한 GitOps 파이프라인을 직접 구축해보았다.

처음에는 단순히 Claude Code에게 명령을 내려서 배포 자동화를 만들어보는 정도로 생각했다.

하지만 실제로 따라가다 보니 단순히 코드를 생성하는 과정이 아니라 GitOps가 왜 필요한지, CI와 CD가 어떻게 나뉘는지, GitHub Actions와 ArgoCD가 어떤 역할을 담당하는지까지 다시 정리할 수 있었다.

특히 평소에는 `kubectl apply`로 바로 배포하는 방식에 익숙했는데, 이번 실습에서는 Git을 기준 상태로 두고 ArgoCD가 클러스터를 동기화하도록 구성했다.

그래서 단순히 "배포가 자동으로 된다"는 것보다, 배포 이력과 변경 이유를 Git에 남기고 클러스터 상태를 Git 기준으로 유지하는 것이 GitOps의 핵심이라는 점을 이해하는 데 도움이 되었다.

이 글에서는 Claude Code를 이용해 GitOps 파이프라인을 구축한 전체 과정과, 실습하면서 이해한 개념, 중간에 막혔던 부분, 직접 확인해야 했던 설정, Claude Code를 사용하면서 느낀 점을 함께 정리한다.

---

## 1. 이번 실습에서 구축하려고 한 구조

이번 실습의 목표는 애플리케이션 코드를 수정한 뒤 Git에 Push하는 것만으로 이미지 빌드부터 Kubernetes 배포까지 자동으로 이어지는 구조를 만드는 것이었다.

최종 구조는 다음과 같다.

```text
Developer
    |
    | git push
    v
GitHub Repository
    |
    | Workflow Trigger
    v
GitHub Actions
    |
    | Docker Build
    v
Container Image
    |
    | Push
    v
Google Artifact Registry
    |
    | deployment.yaml Image Tag 수정
    v
Git Commit & Push
    |
    | Git 변경 감지
    v
ArgoCD
    |
    | Auto Sync
    v
Kubernetes Deployment
    |
    | Rolling Update
    v
새 버전 Pod 배포
```

흐름만 보면 단순해 보이지만, 실제로는 다음과 같은 요소가 모두 연결되어야 한다.

- GitHub Actions Workflow
- GCP 인증
- Artifact Registry 권한
- Docker Build와 Push
- Image Tag 관리
- Kubernetes Manifest 수정
- Git Commit과 Push
- ArgoCD Application
- Auto Sync
- Self Heal
- Rolling Update

처음에는 Claude Code가 이 전체 구조를 한 번에 완성해줄 수 있을지 궁금했다.

결론적으로 기본 구조는 상당히 빠르게 만들어주었지만, 권한과 인증, 환경별 변수, GitOps 운영 방식은 사용자가 반드시 이해하고 확인해야 했다.

---

## 2. 기존 배포 방식에서 불편했던 점

기존에는 애플리케이션 코드를 수정한 뒤 다음과 같은 방식으로 배포할 수 있었다.

```bash
docker build -t app:v1.0.1 .
docker push registry.example.com/app:v1.0.1

kubectl set image deployment/app \
  app=registry.example.com/app:v1.0.1 \
  -n app
```

또는 Manifest 파일을 직접 수정하고 다음 명령을 실행할 수도 있다.

```bash
kubectl apply -f deployment.yaml
```

이 방식은 빠르고 간단하다.

하지만 운영 환경에서 반복적으로 사용하면 여러 문제가 생길 수 있다.

첫 번째 문제는 누가 언제 어떤 버전을 배포했는지 확인하기 어렵다는 점이다.

터미널에서 직접 명령을 실행하면 명령 기록이 남을 수는 있지만, 팀 전체가 확인할 수 있는 변경 이력으로 관리되지는 않는다.

두 번째 문제는 Git에 저장된 Manifest와 실제 클러스터 상태가 달라질 수 있다는 점이다.

예를 들어 Git에는 다음과 같이 저장되어 있다고 가정한다.

```yaml
image: registry.example.com/app:v1.0.0
```

그런데 누군가 다음 명령으로 이미지를 변경했다.

```bash
kubectl set image deployment/app \
  app=registry.example.com/app:v1.0.1
```

그러면 Git에는 `v1.0.0`이 남아 있지만 클러스터에는 `v1.0.1`이 실행된다.

이처럼 Git에 정의된 상태와 실제 클러스터 상태가 달라지는 현상을 Configuration Drift라고 한다.

처음에는 단순히 `kubectl`로 수정하는 것이 더 빠른데 왜 굳이 Git을 한 번 더 거쳐야 하는지 의문이 있었다.

하지만 실습을 진행하면서 Git을 기준 상태로 두는 이유는 단순한 자동화가 아니라 변경 이력, 재현성, 복구 가능성 때문이라는 점을 이해하게 되었다.

---

## 3. GitOps에서 Git이 기준이 되는 이유

GitOps에서는 Git 저장소를 Single Source of Truth로 사용한다.

즉, 클러스터가 어떤 상태여야 하는지는 Git에 저장된 Manifest가 결정한다.

```text
Git의 Desired State
        |
        v
ArgoCD Reconciliation
        |
        v
Kubernetes Actual State
```

ArgoCD는 Git의 상태와 클러스터의 실제 상태를 주기적으로 비교한다.

두 상태가 다르면 이를 OutOfSync로 판단하고, 설정에 따라 자동으로 동기화한다.

이 과정이 반복되는 구조를 Reconciliation Loop라고 한다.

Kubernetes 자체도 Controller가 Desired State와 Actual State를 비교하면서 상태를 맞추는 방식으로 동작한다.

ArgoCD는 이 구조를 Git까지 확장한 Controller라고 볼 수 있다.

실습 전에는 ArgoCD를 단순히 Kubernetes 배포 도구라고 생각했다.

하지만 실제로는 Git을 기준으로 클러스터 상태를 계속 맞추는 Reconciliation Controller라는 설명이 더 정확하다고 느꼈다.

---

## 4. Claude Code에게 처음 요청한 내용

처음에는 Claude Code에게 복잡하게 설명하지 않았다.

다음과 같이 요청했다.

```text
GitHub Actions를 이용해서
Go 애플리케이션 Docker 이미지를 빌드하고
Google Artifact Registry에 Push하는 Workflow를 만들어줘.
```

Claude Code는 프로젝트 파일을 확인한 뒤 `.github/workflows/ci.yaml` 파일을 생성했다.

기본적인 Workflow 구조는 다음과 같았다.

```yaml
name: CI

on:
  push:
    branches:
      - main
    paths:
      - "app/**"

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2

      - name: Set up gcloud
        uses: google-github-actions/setup-gcloud@v2

      - name: Configure Docker
        run: |
          gcloud auth configure-docker asia-northeast3-docker.pkg.dev

      - name: Build and Push
        run: |
          docker build -t ${IMAGE}:${TAG} app/
          docker push ${IMAGE}:${TAG}
```

Claude Code가 단순히 Shell Script만 작성하지 않고 GitHub와 Google에서 제공하는 공식 Action을 사용한 점은 좋았다.

```yaml
uses: actions/checkout@v4
uses: google-github-actions/auth@v2
uses: google-github-actions/setup-gcloud@v2
```

공식 Action을 사용하면 인증과 설정 과정을 직접 스크립트로 구현하는 것보다 구조가 명확하고 유지보수하기도 쉽다.

이때 처음 느낀 점은 Claude Code가 단순히 명령어를 나열하는 것이 아니라 현재 사용되는 일반적인 구성 패턴을 어느 정도 알고 있다는 것이었다.

다만 생성된 코드가 현재 환경에서 바로 동작하는지는 별개의 문제였다.

---

## 5. GitHub Actions Workflow 구조 이해하기

Claude Code가 Workflow를 만들어주더라도 구조를 이해하지 못하면 문제가 발생했을 때 수정하기 어렵다.

GitHub Actions Workflow는 크게 다음 구조로 이루어진다.

```text
Workflow
    |
    +-- Event
    |
    +-- Job
          |
          +-- Step
          +-- Step
          +-- Step
```

### 5-1) `name`

```yaml
name: CI
```

GitHub Actions 화면에 표시되는 Workflow 이름이다.

### 5-2) `on`

```yaml
on:
  push:
    branches:
      - main
```

Workflow가 언제 실행될지를 지정한다.

위 설정은 `main` 브랜치에 Push가 발생하면 Workflow를 실행한다는 의미다.

### 5-3) `jobs`

```yaml
jobs:
  build-and-push:
```

Workflow 안에서 실행할 작업 단위를 정의한다.

Job은 서로 독립된 Runner에서 실행될 수 있다.

### 5-4) `runs-on`

```yaml
runs-on: ubuntu-latest
```

Job을 실행할 Runner 환경을 지정한다.

### 5-5) `steps`

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v4
```

Job 안에서 순서대로 실행되는 작업이다.

### 5-6) `uses`

```yaml
uses: actions/checkout@v4
```

다른 사람이 만들어 놓은 Action을 사용한다.

### 5-7) `run`

```yaml
run: |
  docker build .
  docker push ...
```

Runner에서 직접 Shell 명령을 실행한다.

처음 Workflow를 볼 때는 YAML이 길어서 복잡해 보였지만, Workflow, Job, Step이라는 세 단계로 나누어 보니 구조가 훨씬 단순하게 보였다.

Claude Code가 코드를 생성해주는 것은 빠르지만, 최소한 이 구조는 직접 이해하고 있어야 수정과 장애 분석이 가능하다고 느꼈다.

---

## 6. GCP 인증과 Service Account

GitHub Actions Runner가 Artifact Registry에 이미지를 Push하려면 GCP에 인증해야 한다.

사용자가 로컬에서 다음 명령으로 로그인하는 것과는 다르다.

```bash
gcloud auth login
```

GitHub Actions는 사람이 직접 로그인할 수 없으므로 Service Account를 사용한다.

Service Account는 사람이 아니라 애플리케이션이나 자동화 시스템이 GCP 리소스에 접근할 때 사용하는 계정이다.

예를 들어 GitHub Actions가 Artifact Registry에 이미지를 Push해야 한다면 다음 권한이 필요하다.

```text
roles/artifactregistry.writer
```

Service Account에 필요한 권한만 부여하는 것이 중요하다.

다음처럼 과도한 권한을 부여하면 편할 수는 있지만 보안상 위험하다.

```text
roles/owner
roles/editor
```

이번 실습에서는 학습 편의를 위해 Service Account Key를 생성하고 GitHub Secrets에 저장하는 방식으로 진행했다.

```bash
gcloud iam service-accounts keys create \
  github-ci-key.json \
  --iam-account=github-ci@PROJECT_ID.iam.gserviceaccount.com
```

그리고 GitHub Repository Secret에 등록했다.

```text
GCP_SA_KEY
```

Workflow에서는 다음과 같이 참조한다.

```yaml
- name: Authenticate to Google Cloud
  uses: google-github-actions/auth@v2
  with:
    credentials_json: ${{ secrets.GCP_SA_KEY }}
```

이 방식은 이해하기 쉽지만 Service Account Key가 장기간 유효하다는 문제가 있다.

키 파일은 삭제하기 전까지 계속 사용할 수 있기 때문에 유출되면 위험하다.

Claude Code도 Workload Identity Federation을 대안으로 설명했다.

Workload Identity Federation을 사용하면 GitHub Actions가 OIDC Token을 발급하고, GCP가 이를 검증한 뒤 단기 자격증명을 제공한다.

```text
GitHub Actions
    |
    | OIDC Token
    v
GCP Workload Identity Provider
    |
    | Short-lived Credential
    v
Artifact Registry
```

운영 환경에서는 장기 Service Account Key보다 Workload Identity Federation을 사용하는 것이 더 적절하다.

이번 실습에서 가장 주의 깊게 본 부분도 인증이었다.

Workflow 코드는 Claude Code가 빠르게 작성해주었지만, IAM 권한과 인증 방식은 잘못 설정하면 보안 문제로 이어질 수 있으므로 그대로 믿고 넘어가면 안 된다고 느꼈다.

---

## 7. GitHub Secrets

Service Account Key처럼 코드에 포함하면 안 되는 값은 GitHub Secrets에 저장한다.

예를 들어 다음 값을 Workflow 파일에 직접 작성하면 안 된다.

```yaml
credentials_json: |
  {
    "type": "service_account",
    "private_key": "..."
  }
```

이렇게 작성하면 Git 저장소에 민감정보가 남는다.

대신 GitHub Secrets에 저장하고 다음과 같이 참조한다.

```yaml
credentials_json: ${{ secrets.GCP_SA_KEY }}
```

GitHub Secrets는 크게 다음과 같이 나뉜다.

|종류|적용 범위|
|---|---|
|Repository Secret|특정 Repository|
|Environment Secret|특정 Environment|
|Organization Secret|Organization 내 여러 Repository|

이번 실습에서는 하나의 Repository에서만 사용하므로 Repository Secret을 사용했다.

처음에는 GitHub Secrets에 넣으면 완전히 안전하다고 생각하기 쉽지만, Workflow 권한이 너무 크거나 Secret을 출력하는 코드가 들어가면 노출될 수 있다.

Secret을 저장하는 것뿐 아니라 Workflow 자체를 검토하는 것도 중요하다.

---

## 8. 이미지 태그로 Git Commit SHA 사용하기

Claude Code는 Docker Image Tag로 Git Commit SHA 앞 7자리를 사용하도록 구성했다.

```bash
TAG=$(echo "$GITHUB_SHA" | cut -c1-7)
```

예를 들어 Commit SHA가 다음과 같다고 가정한다.

```text
f6g7h8i1234567890
```

이미지 태그는 다음처럼 만들어진다.

```text
api:f6g7h8i
```

처음에는 사람이 읽기 쉬운 `v1.0.1` 같은 버전 태그가 더 좋지 않은지 궁금했다.

하지만 SHA Tag는 다음과 같은 장점이 있다.

- 어떤 Commit으로 빌드된 이미지인지 바로 확인할 수 있다.
- 같은 태그를 덮어쓰는 실수를 줄일 수 있다.
- Rollback할 버전을 명확하게 지정할 수 있다.
- Git 이력과 Container Image를 연결할 수 있다.

예를 들어 다음 이미지가 실행 중이라면

```text
asia-northeast3-docker.pkg.dev/project/app/api:f6g7h8i
```

다음 명령으로 해당 코드 변경을 확인할 수 있다.

```bash
git show f6g7h8i
```

이 방식은 이미지와 소스 코드를 연결하는 데 매우 유용했다.

단순히 자동으로 빌드된 이미지를 사용하는 것이 아니라, 배포된 이미지가 어떤 코드에서 만들어졌는지 추적할 수 있다는 점이 GitOps와 잘 맞는다고 느꼈다.

---

## 9. CI와 CD의 역할 분리

이번 실습에서 가장 중요하게 이해한 부분은 CI와 CD의 역할 분리다.

GitHub Actions는 CI 역할을 담당한다.

```text
Source Code 변경
    |
    v
Build
    |
    v
Test
    |
    v
Container Image 생성
    |
    v
Registry Push
```

ArgoCD는 CD 역할을 담당한다.

```text
Git Manifest 변경
    |
    v
Git과 Cluster 상태 비교
    |
    v
Sync
    |
    v
Kubernetes 배포
```

GitHub Actions가 Kubernetes에 직접 배포하도록 만들 수도 있다.

예를 들어 다음 명령을 Workflow에서 실행할 수 있다.

```bash
kubectl apply -f deployment.yaml
```

하지만 이렇게 하면 배포가 Git을 거치지 않고 직접 클러스터에 적용된다.

그러면 Git에 저장된 상태와 실제 클러스터 상태가 달라질 수 있다.

이번 실습에서는 GitHub Actions가 다음 역할까지만 수행하도록 했다.

```text
Image Build
    |
    v
Image Push
    |
    v
Manifest 수정
    |
    v
Git Commit & Push
```

클러스터에 적용하는 역할은 ArgoCD가 담당한다.

이렇게 역할을 나누면 문제가 생겼을 때 어느 단계에서 실패했는지 구분하기 쉽다.

- 이미지 빌드 실패는 GitHub Actions
- Registry Push 실패는 GCP 인증
- Manifest 수정 실패는 Workflow Script
- Sync 실패는 ArgoCD
- Pod 실행 실패는 Kubernetes

처음에는 한 Workflow에서 빌드와 배포를 모두 처리하는 것이 더 간단해 보였다.

하지만 역할을 나누면 구조는 조금 길어져도 운영과 문제 해결은 훨씬 명확해진다는 점을 이해했다.

---

## 10. 이미지 태그를 Manifest에 반영하기

CI가 이미지를 빌드하고 Registry에 Push하는 것만으로는 Kubernetes에 새 버전이 배포되지 않는다.

Kubernetes Deployment가 참조하는 Image Tag도 변경되어야 한다.

예를 들어 기존 Manifest가 다음과 같다고 가정한다.

```yaml
containers:
  - name: notiflex-api
    image: asia-northeast3-docker.pkg.dev/project/notiflex/api:old-tag
```

CI가 새로운 이미지를 만들었다.

```text
api:f6g7h8i
```

그러면 Manifest를 다음처럼 수정해야 한다.

```yaml
containers:
  - name: notiflex-api
    image: asia-northeast3-docker.pkg.dev/project/notiflex/api:f6g7h8i
```

이번 실습에서는 `sed`를 사용했다.

```bash
sed -i \
  "s|image: .*notiflex/api:.*|image: ${IMAGE}:${TAG}|" \
  k8s/smb/deployment.yaml
```

이후 Git Commit과 Push를 수행한다.

```bash
git add k8s/smb/deployment.yaml
git commit -m "ci: update image tag to ${TAG}"
git push
```

이 Commit을 ArgoCD가 감지하고 새 버전을 배포한다.

`sed` 방식은 구조가 단순해서 실습하기 좋았다.

하지만 문자열 치환 방식이므로 YAML 구조가 바뀌거나 비슷한 문자열이 여러 개 있으면 잘못 수정할 가능성이 있다.

운영 환경에서는 Kustomize나 ArgoCD Image Updater 같은 방법을 검토할 수 있다.

### Kustomize

```bash
kustomize edit set image \
  notiflex-api=${IMAGE}:${TAG}
```

Kustomize는 YAML의 Image 설정을 구조적으로 관리할 수 있다.

### ArgoCD Image Updater

ArgoCD Image Updater는 Registry를 감시하다가 새 이미지가 올라오면 Manifest 또는 ArgoCD Application Parameter를 자동으로 갱신한다.

이번 실습에서는 전체 흐름을 직접 이해하기 위해 `sed` 방식을 사용했다.

처음부터 Image Updater를 사용했으면 자동화는 더 간단했겠지만, CI가 Manifest를 수정하고 Git에 Commit하는 과정을 직접 보지 못했을 것이다.

학습 목적에서는 다소 단순한 방식을 먼저 사용한 것이 오히려 도움이 되었다고 느꼈다.

---

## 11. `GITHUB_OUTPUT`으로 Step 간 값 전달하기

이미지 빌드 Step에서 만든 Image Tag를 다음 Step에서 사용하려면 Output으로 전달할 수 있다.

```yaml
- name: Build and Push
  id: build
  run: |
    IMAGE=asia-northeast3-docker.pkg.dev/project/notiflex/api
    TAG=$(echo "$GITHUB_SHA" | cut -c1-7)

    docker build -t ${IMAGE}:${TAG} app/
    docker push ${IMAGE}:${TAG}

    echo "image=${IMAGE}:${TAG}" >> "$GITHUB_OUTPUT"
```

다음 Step에서는 다음과 같이 참조한다.

```yaml
${{ steps.build.outputs.image }}
```

예를 들어 Manifest 수정 Step은 다음과 같이 작성할 수 있다.

```yaml
- name: Update Manifest
  run: |
    sed -i \
      "s|image: .*notiflex/api:.*|image: ${{ steps.build.outputs.image }}|" \
      k8s/smb/deployment.yaml
```

처음에는 환경 변수만 사용하면 되는 것 아닌지 궁금했다.

하지만 Step마다 실행 환경이 분리되어 있기 때문에 다른 Step에서 값을 사용하려면 Output이나 환경 파일을 사용해야 한다.

Claude Code가 코드를 생성한 뒤 이 부분을 질문하면서 GitHub Actions에서 Step Output이 어떻게 동작하는지 이해할 수 있었다.

---

## 12. Git Push 권한과 `permissions`

GitHub Actions에서 Manifest를 수정한 뒤 Commit과 Push를 수행하려면 Workflow에 쓰기 권한이 필요하다.

```yaml
permissions:
  contents: write
```

이 설정이 없으면 다음과 같은 오류가 발생할 수 있다.

```text
remote: Permission to repository denied
fatal: unable to access repository
```

GitHub Actions는 기본적으로 최소 권한을 사용하도록 설정되어 있다.

따라서 필요한 권한을 명시해야 한다.

이번 Workflow에서는 Repository 내용을 수정하고 Push해야 하므로 `contents: write`가 필요했다.

```yaml
permissions:
  contents: write
```

GitHub Actions 권한에는 다음과 같은 항목도 있다.

```text
contents
packages
pull-requests
checks
actions
id-token
```

Workload Identity Federation을 사용할 경우에는 다음 권한도 필요하다.

```yaml
permissions:
  contents: read
  id-token: write
```

Claude Code가 `contents: write`를 추가해주었지만, 왜 필요한지 이해하지 못하면 권한을 무작정 늘릴 수 있다.

보안에서는 필요한 최소 권한만 주는 것이 중요하므로 Workflow 권한도 직접 확인해야 한다.

---

## 13. `fetch-depth: 0`을 사용하는 이유

`actions/checkout`은 기본적으로 최근 Commit 일부만 가져올 수 있다.

Git Commit과 Push를 안정적으로 수행하려면 전체 Git 이력이 필요한 경우가 있다.

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
```

`fetch-depth: 0`은 전체 Git 이력을 가져온다.

이번 실습에서는 CI가 Manifest를 수정하고 다시 Push해야 했기 때문에 추가했다.

단순 빌드만 수행한다면 반드시 필요한 설정은 아니지만, Git 작업이 포함될 경우에는 고려해야 한다.

---

## 14. CI가 만든 Commit이 다시 CI를 실행하는 문제

CI가 Manifest를 수정하고 Git Push를 수행하면 다시 Workflow가 실행될 수 있다.

구조만 보면 다음과 같은 무한 루프가 생길 수 있다.

```text
Developer Push
    |
    v
CI 실행
    |
    v
Manifest Commit
    |
    v
CI 재실행
    |
    v
Manifest Commit
    |
    v
반복
```

이번 실습에서는 두 가지 방식으로 이를 방지했다.

첫 번째는 `paths` 필터다.

```yaml
on:
  push:
    branches:
      - main
    paths:
      - "app/**"
```

Workflow는 `app/` 디렉터리가 변경된 경우에만 실행된다.

CI가 수정하는 파일은 다음 위치에 있다.

```text
k8s/smb/deployment.yaml
```

따라서 CI가 만든 Manifest Commit은 Workflow Trigger 조건에 해당하지 않는다.

두 번째는 기본 `GITHUB_TOKEN`으로 생성한 Push가 다시 Workflow를 트리거하지 않는 GitHub의 동작을 활용하는 것이다.

다만 Personal Access Token이나 GitHub App Token을 사용하면 동작 방식이 달라질 수 있으므로 Trigger 조건을 명확히 설정하는 것이 더 안전하다.

이 부분은 Claude Code와 대화하면서 가장 실무적인 문제라고 느꼈다.

파이프라인은 동작하는 것만 확인할 것이 아니라, 자동화가 자기 자신을 다시 호출하지 않는지도 반드시 확인해야 한다.

---

## 15. ArgoCD Application 연결

ArgoCD는 Git 저장소와 배포 대상 클러스터를 Application 리소스로 연결한다.

예시는 다음과 같다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: notiflex-smb
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/example/notiflex-platform.git
    targetRevision: main
    path: k8s/smb

  destination:
    server: https://kubernetes.default.svc
    namespace: notiflex

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

각 설정의 의미는 다음과 같다.

### `repoURL`

ArgoCD가 감시할 Git Repository다.

### `targetRevision`

감시할 Branch, Tag 또는 Commit이다.

```yaml
targetRevision: main
```

### `path`

Repository 안에서 Kubernetes Manifest가 위치한 경로다.

```yaml
path: k8s/smb
```

### `destination.server`

배포 대상 Kubernetes API Server다.

```yaml
server: https://kubernetes.default.svc
```

ArgoCD가 설치된 동일 클러스터를 의미한다.

### `destination.namespace`

리소스를 배포할 Namespace다.

```yaml
namespace: notiflex
```

### `automated`

Git 변경을 감지했을 때 자동으로 Sync할지 설정한다.

### `prune`

Git에서 삭제된 리소스를 클러스터에서도 삭제한다.

### `selfHeal`

클러스터에서 직접 수정된 리소스를 Git 상태로 복원한다.

처음에는 `selfHeal`을 장애를 자동으로 해결해주는 기능처럼 이해하기 쉬웠다.

하지만 정확히는 Git의 Desired State와 다른 Kubernetes Spec을 원래 상태로 되돌리는 기능이다.

애플리케이션 내부 오류, 외부 서비스 장애, 데이터 손상까지 해결하는 기능은 아니다.

---

## 16. Synced와 Healthy는 다른 상태다

ArgoCD에서는 Sync Status와 Health Status를 구분한다.

### Sync Status

Git의 Manifest와 클러스터의 리소스 정의가 같은지 나타낸다.

```text
Synced
OutOfSync
Unknown
```

### Health Status

실제 리소스가 정상 동작 중인지 나타낸다.

```text
Healthy
Progressing
Degraded
Suspended
Missing
```

다음과 같은 상태가 가능하다.

```text
Synced / Healthy
Synced / Degraded
OutOfSync / Healthy
OutOfSync / Missing
```

예를 들어 Git과 클러스터 Manifest는 동일하지만 Pod가 CrashLoopBackOff 상태라면 다음과 같이 보일 수 있다.

```text
Synced / Degraded
```

반대로 클러스터에서 Image Tag를 직접 수정했지만 Pod는 정상 동작 중이라면 다음과 같이 보일 수 있다.

```text
OutOfSync / Healthy
```

실습 전에는 Synced 상태면 애플리케이션도 정상이라고 생각하기 쉬웠다.

하지만 Git과 상태가 동일한 것과 서비스가 실제로 정상 동작하는 것은 별개의 문제라는 점을 이해하게 되었다.

---

## 17. ArgoCD가 Git 변경을 감지하는 방식

ArgoCD는 기본적으로 일정 주기로 Git 저장소를 확인한다.

즉, 다음 과정을 반복한다.

```text
Git Repository 확인
    |
    v
Desired State 생성
    |
    v
Cluster Live State 조회
    |
    v
Diff 계산
    |
    v
Sync 여부 판단
```

기본 Polling만 사용하면 Git Push 후 반영까지 약간의 시간이 걸릴 수 있다.

GitHub Webhook을 ArgoCD에 연결하면 Push 이벤트가 발생한 직후 Reconciliation을 시작할 수 있다.

다만 Webhook을 사용하려면 ArgoCD Server가 외부에서 접근 가능한 URL을 가져야 한다.

이번 실습 환경에서는 Port Forwarding으로 접근하고 있었기 때문에 Webhook을 바로 연결하지 않았다.

처음에는 ArgoCD가 Push 이벤트를 자동으로 받는다고 생각했지만, 실제로는 Polling 또는 Webhook 구성이 필요하다는 점도 확인했다.

---

## 18. Rolling Update 과정 확인하기

Deployment의 Image Tag가 변경되면 Kubernetes는 기본적으로 Rolling Update를 수행한다.

예를 들어 Replicas가 2개라고 가정한다.

```yaml
replicas: 2
```

Rolling Update 과정은 다음과 같다.

```text
기존 Pod 2개
    |
    v
새 Pod 1개 생성
    |
    v
새 Pod Ready
    |
    v
기존 Pod 1개 종료
    |
    v
새 Pod 1개 추가 생성
    |
    v
기존 Pod 1개 종료
```

다음 명령으로 과정을 확인할 수 있다.

```bash
kubectl get pods -n notiflex -w
```

또는 Rollout 상태를 확인할 수 있다.

```bash
kubectl rollout status \
  deployment/notiflex-api \
  -n notiflex
```

Rolling Update 전략은 다음 설정으로 조정할 수 있다.

```yaml
strategy:
  type: RollingUpdate

  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

### `maxSurge`

원하는 Replicas보다 추가로 생성할 수 있는 Pod 수다.

### `maxUnavailable`

업데이트 중 사용할 수 없어도 되는 Pod 수다.

`maxUnavailable: 0`이면 업데이트 중에도 기존 Replica 수를 유지하려고 한다.

다만 Rolling Update라고 해서 항상 무중단이 보장되는 것은 아니다.

다음 설정이 함께 고려되어야 한다.

- Readiness Probe
- Graceful Shutdown
- `preStop`
- `terminationGracePeriodSeconds`
- PodDisruptionBudget
- Connection Draining

이번 실습에서 Rolling Update 중 구버전과 신버전 Pod가 동시에 응답하는 것을 확인했다.

이 과정을 보면서 Rolling Update는 중단을 줄이는 방식이지, 모든 애플리케이션에서 완전한 무중단을 보장하는 방식은 아니라는 점을 이해했다.

API 호환성이 깨지거나 DB Schema 변경이 필요한 경우에는 Blue/Green이나 Canary 배포 전략이 더 적절할 수 있다.

---

## 19. Git Revert를 이용한 Rollback

새 버전에 문제가 생기면 Git Commit을 되돌릴 수 있다.

```bash
git revert HEAD
git push
```

`git revert`는 기존 Commit을 삭제하지 않는다.

대신 이전 변경을 되돌리는 새로운 Commit을 생성한다.

예를 들어 Git 이력은 다음처럼 남는다.

```text
a1b2c3d feat: deploy v1.0.1
d4e5f6g Revert "feat: deploy v1.0.1"
```

ArgoCD는 Revert Commit을 감지하고 이전 Manifest 상태로 다시 Sync한다.

이 방식의 장점은 Rollback 자체도 Git 이력에 남는다는 것이다.

누가 언제 어떤 이유로 이전 버전으로 되돌렸는지 확인할 수 있다.

처음에는 `kubectl rollout undo`가 더 빠르다고 생각했다.

하지만 GitOps 환경에서는 클러스터를 직접 되돌리는 것보다 Git 상태를 되돌려야 Desired State와 Actual State가 다시 일치한다.

이번 실습을 통해 Rollback도 배포와 마찬가지로 Git을 통해 수행해야 한다는 점을 이해했다.

---

## 20. Claude Code에 GitOps 규칙 추가하기

이번 실습에서는 Claude Code가 실수로 `kubectl apply`나 `kubectl delete`를 직접 실행하지 않도록 `CLAUDE.md`에 규칙을 추가했다.

예시는 다음과 같다.

```markdown
## GitOps 운영 규칙

- 이 클러스터에서 관리 중인 애플리케이션 리소스는 직접 `kubectl apply`하지 않는다.
- 배포 변경은 Git Manifest를 수정하고 Commit 및 Push한 뒤 ArgoCD로 반영한다.
- `kubectl delete` 실행 전 반드시 삭제 대상과 영향 범위를 확인한다.
- 변경 전 현재 상태와 Git Diff를 먼저 보여준다.
```

이후 Claude Code에게 다음과 같이 요청했다.

```text
notiflex namespace의 notiflex-api Deployment를 지워줘.
```

Claude Code는 바로 `kubectl delete`를 실행하지 않고 Git에서 Manifest를 제거한 뒤 ArgoCD Prune으로 삭제하는 방식을 제안했다.

이 부분은 꽤 흥미로웠다.

Claude Code가 단순히 명령을 수행하는 도구가 아니라 프로젝트 문서에 작성한 운영 규칙을 참고해 행동을 조정할 수 있다는 점을 확인했기 때문이다.

다만 자연어 규칙은 강제 정책이 아니다.

대화가 길어지거나 규칙이 명확하지 않으면 다시 직접 명령을 제안할 가능성도 있다.

따라서 운영 환경에서는 자연어 규칙만 믿기보다 다음과 같은 기술적 통제가 필요하다.

- Kubernetes RBAC
- Admission Policy
- OPA Gatekeeper
- Kyverno
- Claude Code Permission 설정
- 명령어 차단 Hook

실습을 통해 `CLAUDE.md`는 가이드라인으로는 유용하지만 보안 통제 수단으로 보기는 어렵다는 점도 알게 되었다.

---

## 21. 전체 파이프라인 테스트

최종적으로 애플리케이션 코드를 수정하고 Push했다.

```bash
git add .
git commit -m "feat: add version endpoint"
git push origin main
```

GitHub Actions가 실행되었다.

```text
Checkout
    |
    v
GCP Authentication
    |
    v
Docker Build
    |
    v
Artifact Registry Push
    |
    v
Manifest Update
    |
    v
Git Commit & Push
```

CI가 만든 Commit도 확인할 수 있었다.

```text
ci: update image tag to f6g7h8i
```

ArgoCD가 Manifest 변경을 감지한 뒤 새 이미지를 배포했다.

```bash
kubectl get application -n argocd
```

```text
NAME           SYNC STATUS   HEALTH STATUS
notiflex-smb   Synced        Healthy
```

배포된 이미지도 확인했다.

```bash
kubectl get pods -n notiflex \
  -o jsonpath='{.items[*].spec.containers[*].image}'
```

```text
asia-northeast3-docker.pkg.dev/project/notiflex/api:f6g7h8i
asia-northeast3-docker.pkg.dev/project/notiflex/api:f6g7h8i
```

개발자가 수행한 작업은 코드 수정과 Git Push뿐이었다.

나머지는 자동으로 처리되었다.

```text
Code Push
    |
    v
GitHub Actions CI
    |
    v
Artifact Registry
    |
    v
Manifest Commit
    |
    v
ArgoCD Auto Sync
    |
    v
Kubernetes Rolling Update
```

이 과정을 실제로 끝까지 확인했을 때 GitOps가 단순히 유행하는 운영 방식이 아니라, 변경 이력과 배포 상태를 연결하는 구조라는 점이 가장 명확하게 느껴졌다.

---

## 22. Claude Code를 사용하면서 좋았던 점

Claude Code를 사용하면서 가장 좋았던 점은 반복적인 설정 파일 작성 속도가 빨라졌다는 것이다.

GitHub Actions Workflow, ArgoCD Application, GCP 인증 설정처럼 문법이 긴 파일을 직접 처음부터 작성하지 않아도 되었다.

또한 생성된 코드에 대해 계속 질문할 수 있었다.

예를 들어 다음과 같은 질문을 했다.

```text
왜 Image Tag로 SHA를 사용해?
```

```text
왜 GitHub Actions가 kubectl apply를 직접 하면 안 돼?
```

```text
왜 contents: write 권한이 필요해?
```

```text
왜 Service Account Key보다 WIF가 안전해?
```

단순히 코드를 생성하는 것보다, 생성된 코드의 이유를 바로 질문하면서 학습할 수 있다는 점이 좋았다.

공식 문서를 처음부터 모두 읽고 시작하는 것보다 전체 구조를 빠르게 파악하는 데 도움이 되었다.

---

## 23. Claude Code를 그대로 믿으면 안 되는 이유

Claude Code가 생성한 설정이 항상 현재 환경에 맞는 것은 아니다.

다음 값은 환경마다 달라진다.

- GCP Project ID
- Region
- Artifact Registry Repository
- Service Account
- IAM Role
- Git Repository URL
- Branch
- Namespace
- ArgoCD Application 이름
- Manifest 경로
- Container 이름

권한 설정도 반드시 직접 확인해야 한다.

예를 들어 Artifact Registry Push만 필요한데 과도한 권한을 제안할 수도 있다.

```text
roles/editor
```

이런 권한을 그대로 적용하면 편할 수는 있지만 최소 권한 원칙에 맞지 않는다.

또한 Claude Code가 작성한 Shell Script도 예외 처리와 멱등성이 부족할 수 있다.

```bash
sed -i ...
git commit ...
git push
```

Manifest가 변경되지 않은 경우 `git commit`이 실패할 수 있으므로 다음과 같은 방어 로직이 필요하다.

```bash
git diff --cached --quiet && exit 0
```

결국 Claude Code는 초안을 빠르게 만들고 구조를 설명하는 데 매우 유용하지만, 최종 검토와 운영 판단은 사람이 해야 한다.

이번 실습에서 가장 크게 느낀 부분도 이것이었다.

Claude Code가 많은 작업을 대신해주기는 하지만, 무엇을 만들고 있는지 이해하지 못하면 잘못된 설정도 빠르게 자동화될 수 있다.

---

## 24. 최종 정리

이번 실습에서는 Claude Code를 활용해 GitHub Actions와 ArgoCD를 연결한 GitOps 파이프라인을 구축했다.

전체 과정은 다음과 같다.

```text
1. 개발자가 애플리케이션 코드를 수정한다.
2. GitHub Repository에 Push한다.
3. GitHub Actions가 Workflow를 실행한다.
4. Docker Image를 빌드한다.
5. Git Commit SHA를 Image Tag로 사용한다.
6. Artifact Registry에 이미지를 Push한다.
7. Kubernetes Manifest의 Image Tag를 수정한다.
8. CI가 Manifest 변경을 Git에 Commit하고 Push한다.
9. ArgoCD가 Git 변경을 감지한다.
10. Kubernetes Deployment를 자동으로 Sync한다.
11. Rolling Update로 새 버전 Pod가 배포된다.
```

이번 실습을 통해 다음 내용을 정리할 수 있었다.

- GitOps에서 Git이 Single Source of Truth가 되는 이유
- Configuration Drift가 발생하는 이유
- GitHub Actions와 ArgoCD의 역할 차이
- Service Account와 GitHub Secrets의 역할
- SHA 기반 Image Tag의 장점
- CI가 Manifest를 수정하는 방식
- ArgoCD Auto Sync와 Self Heal
- Synced와 Healthy 상태의 차이
- Rolling Update의 동작과 한계
- Git Revert를 이용한 GitOps Rollback
- CLAUDE.md 규칙의 역할과 한계

Claude Code를 사용하면서 자동화 코드를 빠르게 만들 수 있었고, 코드가 왜 그렇게 작성되었는지도 바로 질문할 수 있었다.

다만 AI가 생성한 설정을 그대로 사용하는 것이 아니라, 권한과 인증, 배포 전략, 운영 영향은 직접 확인해야 한다.

이번 실습에서 Claude Code는 모든 작업을 대신 수행하는 도구라기보다 GitOps 환경을 함께 설계하고 구현하는 Pair Engineer에 가까웠다.

반복적인 파일 작성은 줄어들었지만, 최종 구조를 이해하고 판단하는 역할은 여전히 사용자에게 남아 있었다.

다음에는 이번에 구축한 GitOps 파이프라인을 기반으로 Prometheus, Grafana, Loki를 연결해 Kubernetes Observability 환경을 구성하고, 실제 장애 상황에서 어떤 지표와 로그를 확인해야 하는지 정리해보려고 한다.