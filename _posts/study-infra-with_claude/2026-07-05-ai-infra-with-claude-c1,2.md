---
layout: post
title: "[AI 인프라 모각코] 2장: Claude Code로 GKE 클러스터 구축하고 첫 배포하기"
date: 2026-07-05 22:45:00 +0900
categories: [스터디, AI인프라모각코, Kubernetes, GCP]
tags: [ClaudeCode, GKE, Kubernetes, Docker, GCP, Go, DevOps]
published: true
---

# AI 시대에 개발자가 알아야 하는 인프라 구성 · 배포 with Claude Code

![ai_infra_with_claude교재](../img/ai_infra_with_claude.jpg)

이번 달부터 4주 동안 진행되는 AI 인프라 모각코에 참여하게 되었습니다.

평소에도 Kubernetes나 인프라 관련 공부를 꾸준히 해왔지만, AI Agent를 활용한 개발 방식은 아직 제대로 경험해본 적이 없었습니다.

그래서 이번에는 단순히 책을 읽는 것이 아니라 **직접 실습하면서 모든 과정을 기록해보자**는 목표로 스터디에 참여하게 되었습니다.

이번 모각코는 **가시다 님**께서 운영해주시는 스터디입니다.
평소 다양한 커뮤니티를 통해 가시다 님의 활동을 자주 접했는데, 이번에는 직접 진행하시는 스터디에 참여하게 되어 개인적으로도 기대가 컸습니다.

온라인으로만 접하던 스터디를 직접 함께하게 되니 조금 신기한 기분도 들었습니다.

혼자 공부하면 어느 순간 흐름이 끊기거나 미루게 되는 경우가 많은데, 함께 공부하는 사람들이 있다는 것만으로도 꾸준히 학습할 수 있는 좋은 동기부여가 되는 것 같습니다.

특히 단순히 책을 따라가는 것이 아니라 각자 실습한 내용을 공유하고, 막히는 부분을 함께 이야기할 수 있다는 점이 이번 스터디의 가장 큰 장점이라고 생각했습니다.

이번에 함께 공부하는 책은 **『AI 시대에 개발자가 알아야 하는 인프라 구성 · 배포 with Claude Code』** 입니다.

책의 저자인 **조훈 님**은 Kubernetes와 Cloud Native 분야에서 꾸준히 활동하고 계시는 엔지니어입니다.

실제 운영 환경에서 사용하는 방식과 실습 중심의 내용을 많이 다루시는 것으로 알고 있었는데, 이번 책 역시 단순히 명령어를 따라 하는 수준이 아니라 왜 이런 구성을 선택하는지까지 설명해주는 점이 좋았습니다.

단순히 Kubernetes 사용법만 설명하는 것이 아니라,

- Claude Code를 어떻게 개발에 활용하는지
- AI Agent와 함께 인프라를 구축하는 방법
- GitAIOps 방식으로 운영하는 방법
- GKE 기반으로 실제 서비스를 배포하는 과정

까지 하나의 흐름으로 경험할 수 있도록 구성되어 있다는 점이 인상적이었습니다.

앞으로 4주 동안은 책 내용을 그대로 따라 하는 데서 끝내지 않고,

실제로 왜 이런 구성을 사용하는지,
실무에서는 어떻게 활용되는지,
그리고 제가 기존에 공부했던 Kubernetes와 어떤 차이가 있는지까지 같이 정리해보려고 합니다.

단순한 실습 기록이 아니라 나중에 다시 봐도 도움이 되는 학습 노트처럼 남겨보는 것이 이번 목표입니다.


## 이번 주 학습 범위

이번 주(스터디 1주차)는 책의 1,2장을 실습했습니다.
책의 첫 실습이라고 볼 수 있는 장이었는데, 앞으로 진행될 GitAIOps 실습을 위한 기본 환경을 구축하는 과정이었습니다.

Claude Code를 개발 환경에 연결하고, GCP에 GKE 클러스터를 구축한 뒤 Go 애플리케이션을 컨테이너화하여 Kubernetes에 처음 배포하는 과정까지 진행했습니다.

단순히 명령어를 따라 실행하는 것이 아니라 각 단계에서 왜 이런 구성을 사용하는지 함께 이해하려고 노력했습니다.

## 이번 주 목표

2장에서는 Claude Code를 활용하여 실제 운영 가능한 Kubernetes 환경을 처음부터 구축하는 것이 목표였습니다.

**완료한 작업:**
- Claude Code 설치 및 statusline 설정
- GCP 프로젝트 생성 및 gcloud CLI 설정
- GitHub 저장소 구성
- GKE 클러스터 생성
- Go 애플리케이션 개발 및 컨테이너화
- Artifact Registry에 이미지 빌드
- Kubernetes에 배포 및 테스트

---

## 실습 내용

### 1. 환경 구성

#### Claude Code statusline 설정

터미널 하단에 실시간 정보를 표시하는 statusline을 설정했습니다.

**표시 정보:**
- 모델명 (Sonnet 4.6)
- 컨텍스트 사용량 (게이지바)
- API 사용률 (5시간 윈도우)
- Kubernetes 컨텍스트
- 현재 경로

```bash
# 설정 파일 위치
~/.claude/settings.json
~/.claude/statusline-command.sh
```

**느낀 점:**
회사에서도 Kubernetes를 사용하고 있지만, Claude Code를 중심으로 하나의 프로젝트를 처음부터 구축해본 경험은 이번이 처음이었습니다.

특히 AI가 단순히 코드를 대신 작성해주는 도구가 아니라, 개발과 운영 과정 전체를 함께하는 협업 도구가 될 수도 있겠다는 생각이 들었습니다.

물론 AI가 모든 것을 해결해주는 것은 아니었습니다.
권한 문제나 GCP 설정 오류처럼 결국 원인을 찾아 검증하는 과정은 여전히 개발자의 몫이었습니다.

그래서 앞으로 남은 4주도 AI에게 맡기는 공부가 아니라, AI를 활용해서 더 깊게 이해하는 공부를 해보고 싶습니다.

---

#### GCP 프로젝트 설정

```bash
# 프로젝트 생성
gcloud projects create notiflex-eunbyul-2026 --name="Notiflex GitAIOps"

# 기본 설정
gcloud config set project notiflex-eunbyul-2026
gcloud config set compute/region asia-northeast3
gcloud config set compute/zone asia-northeast3-a

# Artifact Registry 인증
gcloud auth configure-docker asia-northeast3-docker.pkg.dev
```

**프로젝트 정보:**
- **프로젝트 ID**: `notiflex-eunbyul-2026`
- **리전**: `asia-northeast3` (서울)
- **존**: `asia-northeast3-a`

---

### 2. GKE 클러스터 생성

#### 클러스터 사양

```bash
gcloud container clusters create notiflex-cluster \
  --zone=asia-northeast3-a \
  --machine-type=e2-medium \
  --num-nodes=2 \
  --spot \
  --gateway-api=standard \
  --disk-size=30
```

**선택 이유:**
- **e2-medium**: 2 vCPU, 4GB RAM - 실습에 충분
- **Spot VM**: 일반 VM 대비 60-90% 저렴
- **노드 2개**: 가용성 확보 + 비용 최소화
- **Gateway API**: 5장에서 트래픽 관리에 사용 예정

**비용 예상:** 약 $29/월 (Spot VM 기준)

**느낀 점:**
처음에는 왜 GKE Standard를 선택했는지 의문이었는데, Autopilot과 비교해보니 학습 목적으로는 Standard가 훨씬 적합했습니다. 노드를 직접 관리하면서 리소스 할당, 스케일링 등을 배울 수 있었고, Spot VM으로 비용도 크게 절감할 수 있었습니다.

---

### 3. Notiflex API 개발

#### Go 애플리케이션 (app/main.go)

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
	"sync"
)

var (
	counter int
	mu      sync.Mutex
	podName string
)

func main() {
	podName = os.Getenv("POD_NAME")
	if podName == "" {
		podName = "unknown"
	}

	http.HandleFunc("/health", healthHandler)
	http.HandleFunc("/id", idHandler)

	port := ":8080"
	log.Printf("Starting Notiflex API server on %s (Pod: %s)", port, podName)
	if err := http.ListenAndServe(port, nil); err != nil {
		log.Fatalf("Server failed to start: %v", err)
	}
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprintf(w, "OK")
}

func idHandler(w http.ResponseWriter, r *http.Request) {
	mu.Lock()
	counter++
	id := counter
	mu.Unlock()

	w.Header().Set("Content-Type", "application/json")
	fmt.Fprintf(w, `{"id":%d,"pod":"%s"}`+"\n", id, podName)
}
```

**엔드포인트:**
- `GET /health` - 헬스 체크
- `GET /id` - 순차 ID 생성 + Pod 이름 반환

**기술 선택:**
- **Go 1.25**: 이후 OTel SDK와 호환성 확보
- **표준 라이브러리만 사용**: 외부 프레임워크 없이 경량화

---

#### Dockerfile (멀티스테이지 빌드)

```dockerfile
# Build stage
FROM golang:1.25-alpine AS builder

WORKDIR /app
COPY go.mod ./
COPY main.go ./

# 정적 바이너리 빌드 (CGO 비활성화)
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

# Runtime stage
FROM scratch

WORKDIR /
COPY --from=builder /app/app /app

EXPOSE 8080

ENTRYPOINT ["/app"]
```

**scratch 베이스를 선택한 이유:**
- **최소 크기**: 바이너리만 포함, 5MB 이하
- **보안**: 쉘, 패키지 매니저 등 불필요한 도구 제거
- **공격 표면 최소화**: CVE 대상 감소

**비교:**
| 베이스 이미지 | 크기 | 보안 | 디버깅 |
|-------------|------|------|--------|
| scratch | ~5MB | 최고 | 불가능 |
| alpine | ~15MB | 좋음 | 가능 |
| distroless | ~20MB | 좋음 | 제한적 |

**느낀 점:**
처음에는 scratch가 너무 극단적이라고 생각했는데, 실제로 빌드해보니 정적 바이너리만으로도 충분히 동작했습니다. 디버깅이 어렵다는 단점이 있지만, 로컬에서 테스트를 충분히 하고 배포하면 문제없었습니다.

---

### 4. 컨테이너 이미지 빌드 및 푸시

```bash
# Artifact Registry 리포지토리 생성
gcloud artifacts repositories create notiflex \
  --repository-format=docker \
  --location=asia-northeast3

# 빌드 및 푸시 (Cloud Build 사용)
gcloud builds submit app/ \
  --tag=asia-northeast3-docker.pkg.dev/notiflex-eunbyul-2026/notiflex/api:v0.1.0
```

**빌드 결과:**
- 빌드 시간: 1분 2초
- 이미지 크기: ~5MB
- 다이제스트: `sha256:48239e2d...`

---

### 5. Kubernetes 배포

#### 매니페스트 구조

```
k8s/smb/
├── namespace.yaml      # notiflex 네임스페이스
├── deployment.yaml     # 2개 레플리카, health probe
└── service.yaml        # ClusterIP (80 → 8080)
```

#### 배포 명령어

```bash
# 네임스페이스 먼저 생성
kubectl --context gke-sysnet4admin_book_gitaiops apply -f k8s/smb/namespace.yaml

# 전체 리소스 적용
kubectl --context gke-sysnet4admin_book_gitaiops apply -f k8s/smb/

# Pod 확인
kubectl --context gke-sysnet4admin_book_gitaiops get pods -n notiflex
```

**배포 결과:**
```
NAME                            READY   STATUS    RESTARTS   AGE
notiflex-api-646c5dffb5-2mbh2   1/1     Running   0          41s
notiflex-api-646c5dffb5-lncqx   1/1     Running   0          41s
```

---

#### API 테스트

```bash
# Port-forward
kubectl --context gke-sysnet4admin_book_gitaiops port-forward svc/notiflex-api -n notiflex 8080:80 &

# Health 체크
curl http://localhost:8080/health
# 출력: OK

# ID 생성 테스트
curl http://localhost:8080/id
# 출력: {"id":1,"pod":"notiflex-api-646c5dffb5-2mbh2"}

curl http://localhost:8080/id
# 출력: {"id":2,"pod":"notiflex-api-646c5dffb5-2mbh2"}

curl http://localhost:8080/id
# 출력: {"id":3,"pod":"notiflex-api-646c5dffb5-2mbh2"}
```

**성공!** API가 정상적으로 동작하고 순차 ID를 생성합니다.

---

### 6. /update-docs 커스텀 스킬 생성

각 장 완료 시 자동으로 문서를 갱신하는 스킬을 만들었습니다.

**파일 위치:** `.claude/commands/update-docs.md`

**기능:**
- JOURNEY.md 진행 현황 자동 업데이트
- 도구 선택 기록 추가
- 현재 버전 정보 클러스터에서 조회
- 변경사항 Git 커밋

**사용법:**
```bash
/update-docs
```

**느낀 점:**
수동으로 문서를 갱신하다 보면 빠뜨리기 쉬운데, 스킬로 자동화하니 실수를 방지할 수 있었습니다. 특히 이후 장에서 새 문서가 추가되어도 스킬 수정 없이 동작하도록 설계한 점이 인상적이었습니다.

---

## 트러블슈팅

### 문제 1: 터미널 명령어 줄바꿈 문제

**증상:**
복사한 명령어가 줄바꿈 때문에 잘못 실행됨
```bash
gcloud builds submit app/ --tag=asia-northeast3-docker.pkg.dev/
  notiflex-eunbyul-2026/notiflex/api:v0.1.0
# 에러: 경로가 잘림
```

**해결:**
스크립트 파일로 명령어를 저장하여 실행
```bash
# build.sh 생성
./build.sh
```

**배운 점:**
긴 명령어는 heredoc이나 스크립트 파일로 관리하는 것이 안전합니다.

---

### 문제 2: 결제 계정 미설정

**증상:**
```
ERROR: FAILED_PRECONDITION: Billing account for project is not found.
```

**해결:**
GCP 콘솔에서 결제 계정 연결 (무료 크레딧 $300 사용)

**배운 점:**
GCP에서 리소스를 만들려면 반드시 결제 계정이 필요합니다. 무료 평가판을 활용하면 실습 비용을 절감할 수 있습니다.

---

### 문제 3: Cloud Build API 권한 부족

**증상:**
```
ERROR: The caller does not have permission.
```

**해결:**
GCP 콘솔 → IAM → 본인 계정에 "Cloud Build 편집자" 역할 추가

**배운 점:**
GCP는 최소 권한 원칙을 따르므로, 필요한 API마다 권한을 명시적으로 부여해야 합니다.

---

## 배운 점과 느낀 점

### 1. Claude Code의 강력함

터미널에서 AI를 활용하니 다음과 같은 장점이 있었습니다:
- **컨텍스트 유지**: 이전 대화를 기억하고 연속적인 작업 가능
- **실시간 피드백**: 에러가 나면 즉시 분석하고 해결 방안 제시
- **문서 자동화**: 반복적인 문서 작업을 스킬로 자동화

### 2. Infrastructure as Code의 중요성

모든 설정을 코드(매니페스트, Dockerfile, 스크립트)로 관리하니:
- **재현 가능**: 언제든 동일한 환경 재구성
- **버전 관리**: Git으로 변경 이력 추적
- **협업 용이**: 팀원과 설정 공유 쉬움

### 3. 최소 비용으로 실습하기

- **Spot VM**: 일반 VM 대비 60-90% 저렴
- **scratch 베이스**: 이미지 크기 최소화 → 네트워크 비용 절감
- **적절한 리소스**: e2-medium 2개로 충분

**예상 월 비용:** ~$29 (Spot VM 기준)

### 4. 문서화의 중요성

실습하면서 동시에 문서를 작성하니:
- **나중에 참고하기 쉬움**: 몇 주 후에도 무엇을 했는지 명확히 기억
- **블로그 작성 용이**: 실습 내용이 이미 정리되어 있음
- **트러블슈팅 재활용**: 같은 문제를 다시 겪지 않음

## 5. 느낀점

이번 장은 Kubernetes를 설치하는 내용이라기보다,

"AI Agent와 함께 인프라를 구축하는 개발 방식"을 처음 경험해본 시간에 가까웠습니다.

예전에는 공식 문서와 검색을 오가며 하나씩 설정했다면, 이제는 Claude Code와 함께 작업하면서 반복적인 작업은 자동화하고, 중요한 설계와 검증에 더 많은 시간을 사용할 수 있다는 점이 가장 인상 깊었습니다.

물론 AI가 항상 정답을 주는 것은 아니었습니다.

중간중간 권한 문제나 GCP 설정 오류도 있었고, 결국 원인을 확인하고 해결하는 과정은 여전히 개발자의 역할이라는 것도 다시 느낄 수 있었습니다.

앞으로 남은 4주 동안도 단순히 책을 따라가는 것에서 끝내지 않고,

왜 이렇게 설계했는지,
실무에서는 어떻게 활용할 수 있는지,
그리고 기존 Kubernetes 운영 방식과는 어떤 차이가 있는지까지 함께 정리해보려고 합니다.

다음 장에서는 GitAIOps 방식이 조금 더 본격적으로 등장하는 만큼 더욱 기대됩니다.

---

## 최종 아키텍처

```
┌─────────────────────────────────────────┐
│         Google Cloud Platform           │
│  ┌───────────────────────────────────┐  │
│  │  GKE Cluster (notiflex-cluster)   │  │
│  │                                   │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │  Namespace: notiflex        │  │  │
│  │  │                             │  │  │
│  │  │  ┌──────────────────────┐   │  │  │
│  │  │  │  Service (ClusterIP) │   │  │  │
│  │  │  │  notiflex-api:80     │   │  │  │
│  │  │  └──────────┬───────────┘   │  │  │
│  │  │             │               │  │  │
│  │  │  ┌──────────▼───────────┐   │  │  │
│  │  │  │  Deployment          │   │  │  │
│  │  │  │  replicas: 2         │   │  │  │
│  │  │  │                      │   │  │  │
│  │  │  │  ┌────┐    ┌────┐   │   │  │  │
│  │  │  │  │Pod1│    │Pod2│   │   │  │  │
│  │  │  │  └────┘    └────┘   │   │  │  │
│  │  │  └──────────────────────┘   │  │  │
│  │  └─────────────────────────────┘  │  │
│  │                                   │  │
│  │  Nodes: 2 x e2-medium (Spot VM)  │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  Artifact Registry                │  │
│  │  notiflex/api:v0.1.0              │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```
