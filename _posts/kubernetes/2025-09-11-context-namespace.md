---
layout: post
title: "쿠버네티스 Context & Namespace"
date: 2025-09-11 11:11:00 +0900
categories: [Kubernetes, Network, Infra]
tags: [kubernetes, context, namespace, default, guide]
published: true
---

> **핵심 목표**
>
> 1. **Context**가 정확히 무엇인지, 왜 필요한지
> 2. **Namespace**를 만들지 않으면 어떻게 되는지(= `default` 동작)
> 3. 내 실제 환경(**context: `aila` / namespace: `study`**)을 예시로 **실전 워크플로우** 정리

## 0. 내 프롬프트가 의미하는 것

```
ubuntu  k8s-aila-m01  ⎈ aila  study  ~  $
```

- `ubuntu` : 리눅스 서버 로그인 계정
- `k8s-aila-m01` : 접속 중인 노드(hostname)

## 1. Context(콘텍스트) — “클러스터 접속 프로필”

**정의**  
Context는 kubeconfig(`~/.kube/config`)에 저장되는 **접속 프로필**로, 아래 3가지를 **묶음**으로 관리한다.

- **Cluster**: 어느 API 서버(주소/CA 인증서)에 붙을지
- **User(AuthInfo)**: 어떤 자격(토큰/인증서)으로 접근할지

**빠른 점검**

```bash
# 컨텍스트 목록
kubectl config get-contexts

# 현재 컨텍스트 전환
kubectl config use-context aila

# 현재 컨텍스트의 기본 네임스페이스 확인
kubectl config view --minify --output 'jsonpath={..namespace}'; echo
```

**kubeconfig 구조 예시(요약)**

```yaml
apiVersion: v1
clusters:
  - name: aila-cluster
    cluster:
      certificate-authority-data: <BASE64>
users:
  - name: aila-user
    user:
contexts:
  - name: aila
    context:
      user: aila-user
      namespace: qks-cicd
current-context: aila
```

- `current-context: aila` → 프롬프트에 `⎈ aila`로 보임
- `namespace: qks-cicd` → 기본 네임스페이스(수시로 변경 가능)

## 2. Namespace(네임스페이스) — “클러스터 내부의 작업 공간”

**정의**  
클러스터 내부 리소스를 **논리적으로 분리**하는 단위.  
팀/서비스/환경별로 구획을 나누면 이름 충돌 방지, 권한/자원 한도 분리, 장애 격리, 모니터링/정리 용이성이 생긴다.

**내 클러스터의 실제 네임스페이스 (요약)**

```
cert-manager, cilium-secrets, default, istio-system,
kube-system, qks, qks-ceph, qks-cicd, qks-harbor, qks-system,
study  ← 내가 직접 생성
```

**핵심 포인트**

- **네임스페이스를 만들지 않아도 된다.** 만들지 않으면 **자동으로 `default`** 에 모든 것이 생성된다.
- **실무/운영**에서는 거의 항상 **분리**한다(팀/서비스/환경 단위).

```bash
# 네임스페이스 생성
kubectl create namespace study

# 현재 컨텍스트의 기본 네임스페이스를 study로 변경
kubectl config set-context --current --namespace=study

# 확인
kubectl get ns
kubectl get pods          # -n 없이도 현재 기본 ns에 조회됨
```

**관계 ASCII**

```
[Context: aila]
   ├─ Cluster: aila-cluster
   ├─ User: aila-user
   └─ Namespace (기본):
        ├─ default   ← 생성 안 하면 여기에 쌓임
        ├─ qks-cicd
        ├─ study     ← 지금 사용 중
        └─ ...
```

## 3. default만 vs 분리 — 실무 비교표

| 항목                   | default만 사용      | 분리 사용(팀/서비스/환경)   |
| ---------------------- | ------------------- | --------------------------- |
| 이름 충돌              | 높음                | 낮음                        |
| RBAC(권한)             | 거칠게 넓어짐       | 세밀 제어(최소 권한)        |
| 자원 한도(Quota/Limit) | 전역 공통 적용 난해 | ns별 CPU/메모리/객체수 제어 |
| 장애 확산              | 전체로 번지기 쉬움  | ns 경계로 격리              |
| 가시성/정리            | 뒤섞여 혼란         | `-n` 단위로 명확            |
| CI/CD 타깃팅           | 충돌 위험           | 안전한 매핑(dev/stg/prod)   |
| 비용/사용량            | 팀별 파악 어려움    | ns별 리포팅 용이            |

> **결론**
>
> - 개인 실습/임시 테스트 → `default`만으로도 충분
> - 운영/협업/보안 요구 → **네임스페이스 분리**가 사실상 표준

## 4. 내 환경 기준 실전 워크플로우

### 4.1 현재 상태 확인

```bash
kubectl config get-contexts
kubectl config use-context aila
kubectl config view --minify --output 'jsonpath={..namespace}'; echo
kubectl get ns
```

### 4.2 작업 공간 준비

```bash
# (이미 생성했다면 생략)
kubectl create ns study

# 기본 네임스페이스를 study로 고정
kubectl config set-context --current --namespace=study
```

### 4.3 배포/점검

```bash
# 샘플 웹 배포(Deployment + Service)
kubectl create deployment web --image=nginx:1.25 --replicas=2
kubectl expose deployment web --port=80 --target-port=80

# 상태 확인
kubectl get deploy,rs,pods,svc -o wide
kubectl describe deploy web
kubectl logs -l app=web --tail=50
```

### 4.4 접근(로컬 포워딩)

```bash
kubectl port-forward svc/web 8080:80
# 다른 터미널
curl -I http://127.0.0.1:8080
```

### 4.5 운영 루틴

```bash
# 스케일
kubectl scale deploy web --replicas=4

# 롤링 업데이트
kubectl set image deploy/web nginx=nginx:1.27
kubectl rollout status deploy/web
kubectl rollout history deploy/web

# 롤백
kubectl rollout undo deploy/web
```

## 5. 최소 표준 템플릿(바로 사용 가능)

### 5.1 ResourceQuota + LimitRange

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-standard
  namespace: study
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "50"
---
apiVersion: v1
metadata:
  name: lr-defaults
  namespace: study
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "200m"
        memory: "256Mi"
```

### 5.2 RBAC(네임스페이스 관리자)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ns-admin
  namespace: study
rules:
  - apiGroups: ["", "apps", "batch"]
    resources:
      [
        "pods",
        "services",
        "deployments",
        "replicasets",
        "jobs",
        "cronjobs",
        "configmaps",
        "secrets",
      ]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ns-admin-binding
  namespace: study
subjects:
  - kind: User
    name: study-user # 실제 사용자/ServiceAccount로 교체
roleRef:
  kind: Role
  name: ns-admin
  apiGroup: rbac.authorization.k8s.io
```

### 5.3 샘플 애플리케이션(YAML)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: study
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
---
apiVersion: v1
metadata:
  name: web
  namespace: study
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

## 6. 운영에서 자주 생기는 실수와 예방

- **네임스페이스 누락**: `kubectl apply/get/describe`에서 `-n` 깜빡  
  → 해결: 기본 ns를 아예 `study`로 고정
- **라벨/셀렉터 불일치**: Service가 Pod를 못 찾음  
  → 해결: `spec.selector` ↔ `template.metadata.labels` 일관성 유지
- **Quota/Limit 미설정**: 특정 배포가 전체 자원 고갈  
  → 해결: 각 ns에 최소한의 `ResourceQuota`/`LimitRange` 적용
- **과다 권한**: 실수로 전역 리소스 삭제  
  → 해결: 최소 권한 원칙(RBAC 세분화)

## 7. 치트시트

```bash
# 컨텍스트
kubectl config get-contexts
kubectl config use-context aila

# 네임스페이스
kubectl create ns study
kubectl config set-context --current --namespace=study
kubectl get ns

# 배포/서비스
kubectl create deployment web --image=nginx:1.25 --replicas=2
kubectl expose deployment web --port=80 --target-port=80
kubectl get deploy,rs,pods,svc -o wide

# 접근/운영
kubectl port-forward svc/web 8080:80
kubectl scale deploy web --replicas=4
kubectl set image deploy/web nginx=nginx:1.27
kubectl rollout status deploy/web
```

## 8. 결론

- **Context** = 클러스터 접속 프로필 (cluster + user + namespace)

---

kubectl config use-context aila
