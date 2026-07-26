---
layout: post
title: "[인프라 구성·배포 with Claude 모각코] Chapter 7. 규모 확장 실습"
date: 2026-07-24 23:00:00 +0900
categories: [Cloud, Kubernetes, GitOps, Claude Code]
tags: [GKE, Kubernetes, NodePool, nodeSelector, ArgoCD, AppOfApps, SyncWave, MultiTenancy, Namespace, ResourceQuota, ClaudeCode]
published: true
---

# Chapter 7. 규모 확장 실습

6장에서는 Notiflex 서비스를 GKE에 배포하고, Valkey와 Secret Manager를 연결하고, Argo Rollouts를 이용해 Canary 배포까지 구성했다. 그때까지만 해도 API와 캐시, 모니터링 도구가 모두 정상적으로 동작했기 때문에 하나의 서비스를 운영하기 위한 기본 구조는 어느 정도 완성됐다고 생각했다.

그런데 7장에서는 대형 고객이 추가되면서 기존 구조의 한계가 드러나는 상황을 가정한다. 기존에는 Notiflex API, Valkey, Prometheus, Grafana, Loki, Argo CD가 모두 같은 네임스페이스와 같은 기본 NodePool 안에서 실행되고 있었다. 작은 환경에서는 크게 문제가 없어 보였지만, 워크로드가 늘어나면 CPU와 메모리 경쟁이 발생할 수 있고, 고객별 데이터와 리소스를 분리하기도 어렵다.

이번 장에서는 이 문제를 해결하기 위해 역할별 NodePool을 만들고, `nodeSelector`로 워크로드를 분리하고, App of Apps 패턴으로 늘어나는 Argo CD Application을 관리했다. 이후 Enterprise 고객용 네임스페이스를 별도로 구성해 멀티 테넌시 구조를 만들었고, 마지막에는 Claude Code가 위험한 명령을 마음대로 실행하지 못하도록 `settings.local.json`으로 권한도 나눠봤다.

이번 글은 개념을 깊게 설명하기보다 책의 실습 순서를 따라가며 실제로 어떤 구성을 변경했고, 어떤 명령과 YAML을 사용했는지 정리한 기록이다. 실습을 모두 마친 뒤 느낀 점과 새롭게 알게 된 내용은 마지막에 한 번에 정리했다.

---

## 1. 기존 SMB 구조의 한계 확인

7장을 시작할 때의 클러스터는 GKE의 기본 NodePool 두 대에 모든 워크로드가 함께 올라가 있는 구조였다.

```text
default-pool
├── Notiflex API
├── Valkey
├── Prometheus
├── Grafana
├── Loki
└── Argo CD
```

이 구조의 첫 번째 문제는 **리소스 경합**이다. Prometheus가 메트릭을 수집하면서 CPU를 많이 사용하면 같은 노드에서 실행되는 Notiflex API의 응답 속도가 느려질 수 있다. 이후 Kafka처럼 메모리를 많이 사용하는 워크로드가 추가되면 CPU뿐 아니라 메모리 경쟁도 더 심해질 수 있다.

두 번째 문제는 **격리 부족**이다. SMB 고객과 Enterprise 고객의 리소스가 하나의 네임스페이스 안에 섞여 있으면 고객별 접근 권한과 리소스 사용량을 구분하기 어렵다. 대형 고객이 요구하는 것은 단순히 API 복제본을 하나 더 실행하는 것이 아니라, 자신의 데이터와 워크로드가 다른 고객의 영향을 받지 않는 독립된 환경이다.

따라서 이번 장에서는 다음 세 가지 방향으로 구조를 확장했다.

1. 역할에 따라 NodePool을 분리해 워크로드 간 리소스 경합을 줄인다.
2. App of Apps 패턴과 Sync Wave를 적용해 늘어나는 애플리케이션을 일관되게 관리한다.
3. 고객별 네임스페이스를 분리해 멀티 테넌시 구조를 구성한다.

---

## 2. 역할별 NodePool 구성

### 2-1) NodePool 구성 계획

기존에는 `default-pool`에 모든 워크로드가 섞여 있었다. 이를 다음과 같이 나누기로 했다.

```text
default-pool
└── Argo CD, Prometheus, Grafana, Loki 등 시스템·관측 도구

api-pool
└── Notiflex API, Valkey

worker-pool
└── Kafka 등 메모리를 많이 사용하는 워커

ops-pool
└── CronJob, 운영 도구
```

책에서는 NodePool을 단순히 늘리는 데서 끝내지 않고, 각 워크로드의 성격에 맞는 머신 타입을 선택했다.

- `api-pool`: `e2-medium`
- `worker-pool`: `e2-standard-2`
- `ops-pool`: `e2-small`

Kafka가 배치될 `worker-pool`은 JVM과 Kubernetes 자체 오버헤드까지 고려해 메모리가 더 큰 `e2-standard-2`를 사용했다. API와 Valkey는 `e2-medium`, 운영 도구는 상대적으로 작은 `e2-small`로 구성했다.

### 2-2) GCP 할당량 확인

NodePool을 추가하기 전에 리전의 vCPU와 SSD 할당량을 먼저 확인했다. 체험 계정은 할당량이 작기 때문에 NodePool 생성 도중 실패할 수 있다.

```bash
gcloud compute regions describe asia-northeast3 \
  --format="table(quotas[name=CPUS].limit,quotas[name=CPUS].usage)"
```

책의 실행 결과는 다음과 같았다.

```text
LIMIT  USAGE
24     4
```

현재 4개의 vCPU를 사용 중이고 제한은 24개이므로, 추가로 필요한 5개의 vCPU를 생성할 수 있는 상태였다.

SSD 할당량도 확인했다.

```bash
gcloud compute regions describe asia-northeast3 \
  --format="table(quotas[name=SSD_TOTAL_GB].limit,quotas[name=SSD_TOTAL_GB].usage)"
```

```text
LIMIT  USAGE
300    200
```

이번 실습에서는 각 노드의 부팅 디스크로 `pd-standard`를 사용했기 때문에 SSD 할당량에는 직접 영향을 주지 않지만, NodePool 생성 전에 CPU와 디스크 관련 제한을 확인하는 흐름 자체가 중요했다.

### 2-3) api-pool 생성

Notiflex API와 Valkey가 실행될 `api-pool`을 생성했다.

```bash
gcloud container node-pools create api-pool \
  --cluster=notiflex-cluster \
  --zone=asia-northeast3-a \
  --machine-type=e2-medium \
  --disk-type=pd-standard \
  --disk-size=50 \
  --num-nodes=1 \
  --spot \
  --workload-metadata=GKE_METADATA
```

주요 옵션은 다음과 같다.

- `--cluster`: NodePool을 추가할 GKE 클러스터
- `--zone`: NodePool을 생성할 Zone
- `--machine-type`: 노드에 사용할 VM 머신 타입
- `--num-nodes`: 생성할 노드 수
- `--spot`: Spot VM 사용
- `--workload-metadata=GKE_METADATA`: Workload Identity를 위한 메타데이터 설정

### 2-4) worker-pool 생성

Kafka가 실행될 `worker-pool`도 생성했다.

```bash
gcloud container node-pools create worker-pool \
  --cluster=notiflex-cluster \
  --zone=asia-northeast3-a \
  --machine-type=e2-standard-2 \
  --disk-type=pd-standard \
  --disk-size=50 \
  --num-nodes=1 \
  --spot \
  --workload-metadata=GKE_METADATA
```

Kafka는 JVM 기반이며 Broker 자체 메모리뿐 아니라 운영체제와 Kubernetes의 리소스도 필요하기 때문에 `e2-medium`보다 메모리가 큰 `e2-standard-2`를 사용했다.

### 2-5) ops-pool 생성

CronJob과 운영 도구용 `ops-pool`도 생성했다.

```bash
gcloud container node-pools create ops-pool \
  --cluster=notiflex-cluster \
  --zone=asia-northeast3-a \
  --machine-type=e2-small \
  --disk-type=pd-standard \
  --disk-size=50 \
  --num-nodes=1 \
  --spot \
  --workload-metadata=GKE_METADATA
```

### 2-6) 생성 결과 확인

NodePool 생성 후 전체 노드와 머신 타입을 확인했다.

```bash
kubectl --context gke-sysnet4admin_book_gitaiops get nodes -o wide \
  --no-headers | awk '{print $1, $2, $6}'
```

실습 결과는 기본 NodePool 2개와 새로 만든 NodePool 3개를 합쳐 총 5개의 노드가 `Ready` 상태로 표시됐다.

```text
default-pool  e2-medium      2대
api-pool      e2-medium      1대
worker-pool   e2-standard-2  1대
ops-pool      e2-small       1대
```

---

## 3. nodeSelector로 API 워크로드 이동

NodePool을 만들기만 하면 기존 Pod가 자동으로 새 NodePool로 이동하는 것은 아니다. Pod의 배치 조건을 지정하지 않으면 Kubernetes Scheduler는 조건을 만족하는 여러 노드 중 하나를 선택한다.

GKE는 각 Node에 다음과 같은 NodePool Label을 자동으로 추가한다.

```text
cloud.google.com/gke-nodepool=<NODE_POOL_NAME>
```

따라서 Notiflex API를 `api-pool`에 배치하도록 Rollout 매니페스트의 Pod Template에 `nodeSelector`를 추가했다.

```yaml
spec:
  replicas: 2
  template:
    spec:
      nodeSelector:
        cloud.google.com/gke-nodepool: api-pool
```

책에서는 6장에서 리소스 부족 때문에 API 복제본 수를 1개로 줄였지만, 전용 NodePool을 만든 뒤 다시 2개로 복원했다.

변경 후 Git에 반영했다.

```bash
git add k8s/smb/rollout.yaml
git commit -m "feat: add nodeSelector for api-pool, restore replicas to 2"
git push origin main
```

Argo CD가 Git 변경을 동기화한 뒤 Pod 배치를 확인했다.

```bash
kubectl --context gke-sysnet4admin_book_gitaiops \
  get pods -n notiflex -o wide | grep notiflex-api
```

두 개의 Notiflex API Pod가 모두 `api-pool` 노드에 배치된 것을 확인했다.

최종 배치는 다음과 같았다.

```text
api-pool
├── notiflex-api × 2
└── Valkey

default-pool
├── Prometheus
├── Grafana
├── Loki
└── Argo CD

worker-pool
└── 비어 있음: 8장에서 Kafka 배치 예정

ops-pool
└── 비어 있음: 8장에서 CronJob 배치 예정
```

API 트래픽이 증가하면 `api-pool`만 확장하고, Kafka 성능이 부족하면 `worker-pool`만 확장할 수 있는 구조가 됐다.

---

## 4. App of Apps 패턴으로 Argo CD Application 관리

### 4-1) 기존 방식의 문제

실습이 진행되면서 Argo CD Application도 계속 늘어났다.

- Notiflex SMB
- Monitoring
- 이후 추가될 Kafka
- Tempo
- Enterprise 테넌트

Application을 하나씩 직접 `kubectl apply`로 생성하면 새 애플리케이션을 추가할 때 누락될 수 있고, 전체 상태도 한눈에 관리하기 어렵다.

이를 해결하기 위해 하나의 Root Application이 다른 Application YAML을 관리하는 **App of Apps 패턴**을 적용했다.

구조는 다음과 같다.

```text
argocd/
├── root-app.yaml
└── apps/
    ├── notiflex-smb.yaml
    ├── monitoring.yaml
    └── 이후 추가될 kafka.yaml, enterprise.yaml 등
```

`root-app`은 `argocd/apps/` 디렉터리를 감시하고, 그 안에 있는 Application YAML을 자동으로 Argo CD에 적용한다.

### 4-2) Root Application 작성

`argocd/root-app.yaml`을 다음과 같이 작성했다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/USER/notiflex-platform.git
    targetRevision: main
    path: argocd/apps
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

여기서 중요한 설정은 다음과 같다.

- `path: argocd/apps`: 하위 Application YAML이 있는 디렉터리
- `directory.recurse: true`: 하위 디렉터리까지 재귀적으로 탐색
- `prune: true`: Git에서 삭제된 Application 리소스도 제거
- `selfHeal: true`: 클러스터에서 수동으로 변경된 내용을 Git 상태로 복구

### 4-3) 기존 Application 이동

기존 Application YAML을 `argocd/apps/` 아래로 이동했다.

예를 들어 SMB Application은 다음과 같이 구성했다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: notiflex-smb
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/USER/notiflex-platform.git
    targetRevision: main
    path: k8s/smb
  destination:
    server: https://kubernetes.default.svc
    namespace: notiflex
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Monitoring Application도 같은 방식으로 이동했다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/USER/notiflex-platform.git
    targetRevision: main
    path: k8s/monitoring
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 4-4) Git 반영 및 Root Application 생성

```bash
cd notiflex-platform
mkdir -p argocd/apps
git add argocd/
git commit -m "feat: add App of Apps pattern for centralized management"
git push origin main
```

Root Application은 최초 한 번만 직접 적용했다.

```bash
kubectl --context gke-sysnet4admin_book_gitaiops \
  apply -f argocd/root-app.yaml
```

Application 상태를 확인했다.

```bash
kubectl --context gke-sysnet4admin_book_gitaiops \
  get application -n argocd
```

```text
NAME           SYNC STATUS   HEALTH STATUS
root-app       Synced        Healthy
notiflex-smb   Synced        Healthy
monitoring     Synced        Healthy
```

이제 `argocd/apps/` 디렉터리에 새 Application YAML을 추가하면 Root Application이 이를 감지해 자동으로 생성한다. 반대로 YAML을 삭제하면 `prune: true` 설정에 따라 Application도 제거된다.

---

## 5. Sync Wave로 설치 순서 지정

App of Apps를 적용해 여러 애플리케이션을 한꺼번에 관리할 수 있게 됐지만, 모든 리소스를 아무 순서로나 설치해도 되는 것은 아니다.

예를 들어 Namespace가 먼저 있어야 그 안에 Deployment를 만들 수 있고, Gateway API CRD가 먼저 있어야 Gateway나 HTTPRoute를 생성할 수 있다. 따라서 다음과 같이 배포 순서를 나눴다.

| Wave | 구분 | 예시 |
|---|---|---|
| 0 | 인프라 | Namespace, Gateway, CRD |
| 1 | 플랫폼 | Prometheus, Loki, Argo CD |
| 2 | 애플리케이션 | Notiflex, Valkey, Kafka |

Argo CD에서는 `argocd.argoproj.io/sync-wave` Annotation으로 순서를 지정할 수 있다. 낮은 숫자부터 먼저 동기화된다.

Monitoring Application에는 Wave 1을 지정했다.

```yaml
metadata:
  name: monitoring
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

Notiflex SMB Application에는 Wave 2를 지정했다.

```yaml
metadata:
  name: notiflex-smb
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

변경 내용을 Git에 반영했다.

```bash
cd notiflex-platform
git add argocd/apps/
git commit -m "feat: add sync-wave annotations for deployment ordering"
git push origin main
```

이제 클러스터를 새로 만들더라도 Root Application 하나만 적용하면 인프라, 플랫폼, 애플리케이션 순서로 전체 환경을 복원할 수 있는 구조가 됐다.

---

## 6. Enterprise 고객용 멀티 테넌시 구성

### 6-1) 네임스페이스 분리 방식 선택

이번 장에서는 B2B SaaS에 Enterprise 고객이 추가되는 상황을 가정했다. 고객별로 데이터와 리소스를 구분하기 위해 다음 방법을 비교했다.

- Namespace + RBAC
- Namespace + NetworkPolicy
- vCluster
- 고객별 독립 클러스터

실습 환경에서는 추가 리소스가 거의 들지 않고 Kubernetes 기본 기능만으로 구현할 수 있는 Namespace 분리 방식을 선택했다.

구조는 다음과 같다.

```text
notiflex namespace
└── 기존 SMB 고객용 API

enterprise namespace
└── Enterprise 고객용 API
```

Valkey는 학습 목적상 기존 `notiflex` 네임스페이스의 인스턴스를 두 테넌트가 공유하도록 했다.

### 6-2) Cross-Namespace DNS 사용

Enterprise API가 다른 네임스페이스에 있는 Valkey에 접근하려면 Service의 전체 DNS 이름을 사용해야 한다.

```text
valkey-primary.notiflex.svc.cluster.local:6379
```

Kubernetes Service FQDN은 다음 형식이다.

```text
<Service 이름>.<Namespace>.svc.cluster.local
```

Enterprise Rollout의 환경변수에 해당 주소를 지정했다.

```yaml
env:
  - name: VALKEY_ADDR
    value: valkey-primary.notiflex.svc.cluster.local:6379
```

비밀번호는 Enterprise 네임스페이스의 Secret에서 읽도록 구성했다.

```yaml
  - name: VALKEY_PASSWORD
    valueFrom:
      secretKeyRef:
        name: valkey-enterprise
        key: password
```

### 6-3) Enterprise Namespace와 Rollout 작성

`k8s/enterprise/namespace.yaml`을 작성했다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: enterprise
```

Enterprise API는 별도의 Rollout으로 생성했다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: notiflex-api
  namespace: enterprise
spec:
  replicas: 1
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: notiflex-api
      tenant: enterprise
  strategy:
    canary:
      steps:
        - setWeight: 50
        - pause:
            duration: 30s
  template:
    metadata:
      labels:
        app: notiflex-api
        tenant: enterprise
    spec:
      nodeSelector:
        cloud.google.com/gke-nodepool: api-pool
      containers:
        - name: notiflex-api
          image: asia-northeast3-docker.pkg.dev/PROJECT_ID/notiflex/api:v0.5.0
          ports:
            - containerPort: 8080
          env:
            - name: VALKEY_ADDR
              value: valkey-primary.notiflex.svc.cluster.local:6379
            - name: VALKEY_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: valkey-enterprise
                  key: password
```

SMB와 Enterprise를 구분할 수 있도록 `tenant: enterprise` Label을 추가했다.

### 6-4) Valkey 비밀번호 복사

Valkey 인스턴스는 `notiflex` 네임스페이스에 있지만, Enterprise Pod에서 Secret을 참조하려면 같은 비밀번호를 `enterprise` 네임스페이스에도 만들어야 한다.

기존 비밀번호를 변수에 저장했다.

```bash
VALKEY_PW=$(kubectl --context gke-sysnet4admin_book_gitaiops \
  get secret valkey -n notiflex \
  -o jsonpath='{.data.valkey-password}' | base64 -d)
```

Enterprise 네임스페이스를 생성했다.

```bash
kubectl --context gke-sysnet4admin_book_gitaiops \
  create namespace enterprise
```

같은 비밀번호로 Secret을 생성했다.

```bash
kubectl --context gke-sysnet4admin_book_gitaiops \
  create secret generic valkey-enterprise \
  -n enterprise \
  --from-literal=password="$VALKEY_PW"
```

### 6-5) Enterprise Application 추가

App of Apps에서 자동으로 관리하도록 `argocd/apps/notiflex-enterprise.yaml`을 작성했다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: notiflex-enterprise
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/USER/notiflex-platform.git
    targetRevision: main
    path: k8s/enterprise
  destination:
    server: https://kubernetes.default.svc
    namespace: enterprise
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Git에 반영했다.

```bash
cd notiflex-platform
git add k8s/enterprise/ argocd/apps/notiflex-enterprise.yaml
git commit -m "feat: add enterprise tenant with multi-tenancy"
git push origin main
```

Git Push 후 Root Application은 새 YAML을 감지해 `notiflex-enterprise` Application을 생성하고, `k8s/enterprise/`의 리소스를 Enterprise 네임스페이스에 배포했다.

### 6-6) 배포 상태 확인

```bash
kubectl --context gke-sysnet4admin_book_gitaiops \
  get application -n argocd
```

```text
NAME                 SYNC STATUS   HEALTH STATUS
root-app             Synced        Healthy
notiflex-smb         Synced        Healthy
monitoring           Synced        Healthy
notiflex-enterprise  Synced        Healthy
```

Enterprise Pod도 확인했다.

```bash
kubectl --context gke-sysnet4admin_book_gitaiops \
  get pods -n enterprise
```

로그를 확인해 Valkey 연결 여부를 검증했다.

```bash
kubectl --context gke-sysnet4admin_book_gitaiops \
  logs -l app=notiflex-api -n enterprise --tail=3
```

책의 예시에서는 첫 연결 시도에서 타임아웃이 발생한 뒤 다시 연결에 성공했다.

```text
Valkey 연결 재시도 1/10: context deadline exceeded
Valkey 연결 성공
서버 시작: :8080
```

### 6-7) ResourceQuota 적용

한 테넌트가 클러스터 리소스를 모두 점유하지 못하도록 Enterprise 네임스페이스에 ResourceQuota를 작성했다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: enterprise-quota
  namespace: enterprise
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
```

학습 환경에서는 NodePool 자체가 작아 실제 적용을 생략했지만, 운영 환경에서 고객별 Namespace를 사용한다면 ResourceQuota는 반드시 함께 고려해야 한다.

### 6-8) ID 생성 테스트

Enterprise API로 요청을 보내 ID가 정상적으로 생성되는지 확인했다.

```bash
kubectl --context gke-sysnet4admin_book_gitaiops \
  port-forward svc/notiflex-api -n enterprise 8081:80 &
```

```bash
curl http://localhost:8081/id
```

```json
{"id":10,"generated_by":"notiflex-api-..."}
```

다시 호출했다.

```bash
curl http://localhost:8081/id
```

```json
{"id":11,"generated_by":"notiflex-api-..."}
```

테넌트는 분리됐지만 같은 Valkey 카운터를 공유하기 때문에 SMB 테넌트가 사용한 ID 뒤에서 Enterprise ID가 이어졌다. 이 구조는 Cross-Namespace DNS를 실습하기 위한 것이며, 실제 운영에서 고객 데이터를 완전히 격리하려면 테넌트별 Valkey 인스턴스를 사용하거나 Key Prefix를 분리하는 설계가 추가로 필요하다.

---

## 7. `settings.local.json`으로 Claude Code 권한 분리

### 7-1) 자연어 규칙의 한계

앞 장에서는 `CLAUDE.md`에 다음과 같은 자연어 규칙을 작성했다.

```text
kubectl delete를 직접 실행하지 말 것
항상 Git을 통해 배포할 것
```

하지만 자연어 규칙은 Claude Code가 참고하는 가이드일 뿐, 명령 실행을 기술적으로 차단하지는 않는다. 이번 장에서는 NodePool과 여러 테넌트를 다루기 때문에 잘못된 명령 하나로 운영 중인 리소스를 삭제할 가능성이 더 커졌다.

이를 막기 위해 `.claude/settings.local.json`을 사용했다.

### 7-2) 권한 정책 작성

```json
{
  "permissions": {
    "deny": [
      "Bash(kubectl delete *)",
      "Bash(kubectl apply *)"
    ],
    "ask": [
      "Bash(helm install *)",
      "Bash(helm upgrade *)",
      "Bash(gcloud container node-pools delete *)"
    ]
  }
}
```

권한은 세 단계로 나뉜다.

| 수준 | 동작 | 사용 예시 |
|---|---|---|
| `allow` | 승인 없이 실행 | `kubectl get`, `kubectl logs` |
| `ask` | 실행 전 사용자 확인 | `helm install`, NodePool 삭제 |
| `deny` | 명령 실행 차단 | `kubectl delete`, `kubectl apply` |

자연어로 요청해 정책을 자동 생성할 수 있지만, 실제 운영 환경에서는 생성 결과를 사람이 직접 열어 명령 패턴을 확인하는 것이 안전하다.

### 7-3) deny 동작 확인

Claude Code에 Enterprise API를 직접 삭제하도록 요청했다.

```text
엔터프라이즈 네임스페이스의 notiflex-api를 kubectl로 지워줘.
```

실행하려는 명령은 다음과 같았다.

```bash
kubectl delete deployment notiflex-api -n enterprise
```

하지만 `settings.local.json`의 다음 규칙에 의해 차단됐다.

```json
"Bash(kubectl delete *)"
```

또한 이 환경은 Argo CD가 Git 저장소를 기준으로 상태를 관리하므로, 직접 삭제하더라도 `selfHeal`로 다시 생성될 수 있다. 반대로 Git에 없는 리소스를 직접 `kubectl apply`로 만들면 Argo CD의 관리 밖에 있는 리소스가 생길 수 있으므로 `kubectl apply`도 차단 대상으로 지정했다.

### 7-4) ask 동작 확인

사용하지 않는 `worker-pool`을 삭제해 달라고 요청했다.

```text
worker-pool 이거 누가 만든 거지? 모르는 노드풀이고 비용도 들고 안 쓰는 것 같은데 그냥 삭제해줘.
```

Claude Code가 생성한 명령은 다음과 같았다.

```bash
gcloud container node-pools delete worker-pool \
  --cluster=notiflex-cluster \
  --zone=asia-northeast3-a
```

이 명령은 `ask`에 등록되어 있어 바로 실행되지 않고 사용자 확인을 요청했다. 거부하면 명령은 실행되지 않는다.

운영 환경에서 익숙하지 않은 리소스를 발견했을 때 바로 삭제하는 대신, 한 번 승인 단계를 거치도록 하는 안전장치다.

### 7-5) 실습 설정 되돌리기

다음 장의 실습에 영향을 주지 않도록 `settings.local.json`을 삭제했다.

```bash
rm .claude/settings.local.json
```

파일이 제거됐는지 확인했다.

```bash
ls .claude/
```

이후 `/update-docs`를 실행해 변경된 아키텍처 결정과 진행 상황을 문서에 반영했다.

```text
/update-docs
```

Git 상태를 확인했다.

```bash
git status
```

```text
M JOURNEY.md
M docs/architecture-decisions.md
M claude-context/architecture.md
```

변경 내용을 커밋했다.

```bash
git add JOURNEY.md docs/architecture-decisions.md claude-context/
git commit -m "ch7: 멀티 노드풀, App of Apps, 멀티 테넌시와 결정 누적"
git push origin main
```

---

## 8. 7장에서 완성된 구조

이번 장을 마친 뒤 클러스터는 다음과 같은 구조가 됐다.

```text
인터넷
└── Gateway API
    └── HTTPRoute
        ├── notiflex namespace
        │   ├── Notiflex API × 2
        │   └── Valkey
        │
        └── enterprise namespace
            └── Notiflex API × 1
                 └── Cross-Namespace DNS로 notiflex의 Valkey 사용
```

Argo CD는 App of Apps 구조로 애플리케이션을 관리한다.

```text
root-app
└── argocd/apps/
    ├── monitoring       (Sync Wave 1)
    ├── notiflex-smb     (Sync Wave 2)
    └── notiflex-enterprise (Sync Wave 2)
```

GKE NodePool은 역할별로 분리됐다.

```text
default-pool
└── Argo CD, Prometheus, Grafana, Loki

api-pool
├── SMB Notiflex API × 2
├── Enterprise Notiflex API × 1
└── Valkey

worker-pool
└── 8장에서 Kafka 배치 예정

ops-pool
└── 8장에서 CronJob 배치 예정
```

---

## 9. 실습을 마치며

이번 장은 지금까지 만든 환경에 단순히 리소스를 더 추가하는 실습일 것이라고 생각했는데, 실제로는 서비스 규모가 커졌을 때 기존 구조를 어떻게 바꿔야 하는지를 순서대로 경험하는 내용이었다.

가장 먼저 알게 된 것은 **노드를 늘리는 것과 워크로드를 분리하는 것은 다르다**는 점이었다. 이전에는 클러스터의 자원이 부족하면 Node만 추가하면 된다고 생각했지만, 모든 워크로드가 같은 NodePool에 섞여 있으면 노드를 추가해도 API, 모니터링, 메시지 브로커가 계속 같은 자원을 경쟁할 수 있다. 역할별 NodePool을 만들고 `nodeSelector`를 적용해 보면서 Pod의 실행 위치도 인프라 설계의 일부라는 점을 이해할 수 있었다.

App of Apps 실습에서는 Argo CD Application이 늘어날수록 관리 방식도 함께 바뀌어야 한다는 것을 알게 됐다. Application이 두세 개일 때는 직접 적용해도 큰 문제가 없지만, 고객과 플랫폼 구성요소가 계속 늘어나면 어떤 Application이 배포돼야 하는지 자체를 Git으로 관리해야 했다. Root Application 하나가 `argocd/apps/` 디렉터리를 감시하고 하위 Application을 자동으로 생성하는 모습을 보면서 GitOps는 단순히 YAML을 Git에 올리는 것이 아니라, Git 저장소의 구조 자체로 운영 방식을 정의하는 것이라는 생각이 들었다.

Sync Wave는 처음에는 숫자로 배포 순서만 정하는 간단한 기능처럼 보였다. 하지만 Namespace, CRD, Gateway 같은 기반 리소스가 먼저 준비되지 않으면 이후 리소스가 정상적으로 생성될 수 없기 때문에, 전체 환경을 처음부터 재구성할 때는 설치 순서가 매우 중요했다. 클러스터를 지우고 다시 만들어도 Root Application 하나로 같은 순서대로 복원할 수 있다는 점이 특히 인상적이었다.

멀티 테넌시 실습에서는 Namespace를 분리한다고 해서 고객 데이터까지 자동으로 완전히 격리되는 것은 아니라는 점을 알게 됐다. 이번 실습에서는 SMB와 Enterprise가 서로 다른 Namespace와 Rollout을 사용했지만, Valkey는 하나를 공유했다. 따라서 Cross-Namespace DNS로 다른 Namespace의 Service에 접근하는 방법은 배웠지만, 동시에 실제 운영에서는 테넌트별 저장소 분리나 Key Prefix 설계가 필요하다는 점도 확인할 수 있었다. ResourceQuota와 NetworkPolicy까지 함께 적용해야 고객 간 리소스와 네트워크 영향을 더 제대로 제한할 수 있다는 것도 이번 장을 통해 알게 됐다.

마지막 `settings.local.json` 실습은 인프라 구성과는 조금 다른 내용처럼 보였지만, 실제로는 운영 안전성과 직접 연결되는 부분이었다. `CLAUDE.md`에 자연어로 “삭제하지 말 것”이라고 적는 것과 명령 실행 자체를 `deny`로 막는 것은 전혀 다른 수준의 통제였다. Claude Code가 인프라 명령을 대신 실행할 수 있는 만큼, 사람에게 RBAC와 승인 절차가 필요한 것처럼 AI Agent에도 명확한 권한과 승인 과정이 필요하다는 점을 체감했다.