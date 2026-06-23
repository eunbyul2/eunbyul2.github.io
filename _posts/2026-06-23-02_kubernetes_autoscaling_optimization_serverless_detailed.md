---
layout: post
title: "Kubernetes Autoscaling과 Optimization 상세 정리: HPA, VPA, Goldilocks, In-place Pod Resize, Knative"
date: 2026-06-23 18:00:00 +0900
categories: [Kubernetes, Infrastructure]
tags: [Kubernetes, HPA, VPA, Goldilocks, InPlacePodResize, Patch, Knative, Lambda, Serverless, RightSizing, Autoscaling]
published: true
---

## 1. 글의 목적

대규모 Kubernetes 환경에서 리소스 문제를 해결할 때 단순히 Worker Node를 추가하는 방식만으로는 한계가 있다. Pending Pod가 발생했을 때 Node를 늘리면 당장은 Pod가 스케줄링될 수 있지만, 문제의 원인이 `requests` 과다 설정, 유휴 워크로드 방치, 잘못된 확장 정책, 멀티테넌시 구조 문제라면 같은 문제가 반복된다.

이 글은 Kubernetes Autoscaling과 Optimization을 중심으로 다음 개념을 상세히 정리한다.

```text
HPA
VPA
Goldilocks
Right Sizing
In-place Pod Resize
kubectl edit
kubectl patch
Resize Policy
Patch 기반 리소스 축소 자동화
Serverless Computing
AWS Lambda
Knative
Scale to Zero
Cold Start
Event Driven Architecture
```

핵심 목표는 “어떤 기술이 무엇을 자동화하는지”를 구분하는 것이다. HPA는 Pod 개수를 조정하고, VPA는 Pod의 리소스 크기를 조정하며, Goldilocks는 VPA 추천값을 시각화한다. In-place Pod Resize는 실행 중인 Pod의 CPU/Memory request/limit을 Pod 재생성 없이 바꾸는 기능이고, Knative는 Kubernetes 위에서 서버리스 스타일의 scale-to-zero를 제공한다.

공식 문서:

- Kubernetes Horizontal Pod Autoscaling: <https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/>
- Kubernetes HPA Walkthrough: <https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/>
- Kubernetes Autoscaling Workloads: <https://kubernetes.io/docs/concepts/workloads/autoscaling/>
- Kubernetes Vertical Pod Autoscaler: <https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/>
- Kubernetes VPA GitHub: <https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler>
- Kubernetes In-place Pod Resize: <https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/>
- Kubernetes v1.33 In-place Pod Resize Beta: <https://kubernetes.io/blog/2025/05/16/kubernetes-v1-33-in-place-pod-resize-beta/>
- Kubernetes v1.35 In-place Pod Resize Stable: <https://kubernetes.io/blog/2025/12/19/kubernetes-v1-35-in-place-pod-resize-ga/>
- kubectl edit 공식 문서: <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/>
- kubectl patch 공식 문서: <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_patch/>
- kubectl patch Task 문서: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/>
- Fairwinds Goldilocks Documentation: <https://goldilocks.docs.fairwinds.com/>
- Fairwinds Goldilocks GitHub: <https://github.com/FairwindsOps/goldilocks>
- Knative Serving Autoscaling: <https://knative.dev/docs/serving/autoscaling/>
- Knative Scale to Zero: <https://knative.dev/docs/serving/autoscaling/scale-to-zero/>
- AWS Lambda: <https://aws.amazon.com/lambda/>
- AWS Lambda Developer Guide: <https://docs.aws.amazon.com/lambda/latest/dg/welcome.html>

## 2. Kubernetes Autoscaling의 큰 분류

### 2-1) Autoscaling이 필요한 이유

Kubernetes에서 Autoscaling은 워크로드의 부하 변화에 맞춰 리소스를 자동으로 조정하는 방식이다. 사용량이 증가할 때는 처리량을 확보해야 하고, 사용량이 감소할 때는 불필요한 리소스 예약을 줄여야 한다.

Autoscaling을 하지 않으면 다음과 같은 문제가 생긴다.

```text
트래픽 증가
↓
Pod 수나 리소스가 그대로임
↓
응답 지연, 쿼리 지연, 장애 발생

트래픽 감소
↓
Pod 수나 request가 그대로임
↓
유휴 리소스 점유
↓
클러스터 효율 저하
```

대규모 데이터 플랫폼에서는 트래픽뿐 아니라 쿼리 수, 쿼리 복잡도, 배치 작업, 데이터 적재량, 사용자 접속 패턴에 따라 리소스 사용량이 크게 달라진다. 따라서 단순 평균 사용량만 보고 리소스를 고정하면 과다 할당 또는 부족 할당이 발생하기 쉽다.

### 2-2) 수평 확장과 수직 확장

Autoscaling은 크게 수평 확장과 수직 확장으로 나눌 수 있다.

```text
Horizontal Scaling = Pod 개수를 늘리거나 줄임
Vertical Scaling   = Pod 하나의 CPU/Memory 크기를 늘리거나 줄임
```

HPA는 Horizontal Pod Autoscaler이므로 Pod 개수를 조정한다.

```text
Pod 3개
↓
Pod 10개
```

VPA는 Vertical Pod Autoscaler이므로 Pod 하나의 CPU/Memory request/limit을 조정한다.

```text
CPU request 4core
Memory request 8Gi
↓
CPU request 1core
Memory request 2Gi
```

수평 확장은 stateless 웹 서비스처럼 Pod를 여러 개 늘려도 문제가 없는 워크로드에 잘 맞는다. 수직 확장은 Pod 개수를 쉽게 늘리기 어렵거나, Pod 하나의 request가 실제 사용량보다 과도하게 잡힌 워크로드의 right-sizing에 유용하다.

### 2-3) Pod 확장과 Node 확장의 차이

Pod 확장과 Node 확장은 서로 다른 계층의 작업이다.

```text
HPA:
Pod 개수를 늘리거나 줄임

VPA:
Pod 하나의 request/limit을 조정함

Cluster Autoscaler:
Node 개수를 늘리거나 줄임

Karpenter:
Pending Pod 요구사항에 맞는 Node를 동적으로 프로비저닝함
```

중요한 점은 Node Autoscaling이 항상 근본 해결책은 아니라는 것이다. Pod request가 과도하게 잡혀 있으면 Scheduler는 클러스터에 자원이 부족하다고 판단하고, Cluster Autoscaler는 Node를 더 늘릴 수 있다. 그러나 실제 사용량이 낮다면 이는 리소스 낭비를 Node 증설로 덮는 구조가 된다.

따라서 일반적인 접근 순서는 다음과 같다.

```text
1. 실제 사용량과 request 차이 확인
2. request/limit right-sizing
3. HPA/VPA/Goldilocks 적용 가능성 검토
4. In-place Pod Resize 또는 상위 리소스 spec 변경
5. 그래도 부족하면 Node 증설 또는 Node Autoscaling 검토
```

## 3. HPA: Horizontal Pod Autoscaler

### 3-1) HPA란 무엇인가

HPA는 Horizontal Pod Autoscaler이다. 워크로드의 부하에 따라 Pod 개수를 자동으로 늘리거나 줄인다. Kubernetes 공식 문서 기준 HPA는 Kubernetes API resource이면서 controller이며, Deployment 같은 scale subresource를 가진 리소스의 desired replica 수를 metrics에 따라 주기적으로 조정한다.

예를 들어 Deployment가 3개의 Pod로 실행 중이고 CPU 사용률이 목표치를 초과하면 HPA는 replicas를 늘릴 수 있다.

```text
replicas: 3
↓
replicas: 10
```

HPA는 다음과 같은 리소스를 대상으로 사용할 수 있다.

```text
Deployment
ReplicaSet
StatefulSet
ReplicationController
scale subresource를 지원하는 custom resource
```

### 3-2) HPA가 사용하는 Metrics

HPA는 CPU utilization뿐 아니라 memory utilization, custom metrics, external metrics를 사용할 수 있다.

대표적인 기준은 CPU utilization이다.

```text
CPU utilization = 실제 CPU 사용량 / CPU request
```

예를 들어 Pod의 CPU request가 1core이고 실제 CPU 사용량이 500m이면 CPU utilization은 50%이다. HPA target이 70%라면 아직 scale out하지 않을 수 있다. 반대로 실제 CPU 사용량이 900m이면 utilization은 90%이므로 scale out 대상이 될 수 있다.

이 구조 때문에 HPA를 CPU 기준으로 사용할 때는 request가 매우 중요하다. request가 너무 낮으면 utilization이 높게 계산되어 불필요하게 Pod가 늘 수 있고, request가 너무 높으면 utilization이 낮게 계산되어 scale out이 늦어질 수 있다.

### 3-3) HPA 동작 예시 YAML

아래 예시는 `web-api` Deployment를 CPU 평균 사용률 60% 기준으로 3개에서 10개 사이로 자동 조정하는 HPA이다.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-api-hpa
  namespace: app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

이 YAML의 의미는 다음과 같다.

```text
scaleTargetRef:
어떤 워크로드를 조정할지 지정

minReplicas:
최소 Pod 개수

maxReplicas:
최대 Pod 개수

metrics:
어떤 지표를 기준으로 확장할지 지정

averageUtilization:
Pod들의 평균 CPU utilization 목표값
```

### 3-4) HPA가 잘 맞는 워크로드

HPA는 stateless 서비스에 잘 맞는다.

```text
REST API
웹 서버
백엔드 애플리케이션
worker 서비스
stateless batch consumer
```

이런 서비스는 Pod 개수를 늘리면 처리량이 비교적 선형적으로 증가한다. 특정 Pod에 상태가 강하게 묶여 있지 않기 때문에 scale out과 scale in이 상대적으로 쉽다.

### 3-5) HPA가 조심스러운 워크로드

HPA가 모든 워크로드에 맞는 것은 아니다.

```text
데이터베이스
분산 저장소
분산 분석 엔진
상태가 강한 StatefulSet
Pod마다 고유 PVC를 가진 시스템
```

이런 워크로드는 Pod 개수를 늘리는 것이 단순하지 않다. 새 Pod가 클러스터에 조인해야 하고, 데이터 리밸런싱이 필요할 수 있으며, scale in 시 데이터 이동이나 쿼리 영향이 발생할 수 있다.

예를 들어 분석 DB나 분산 쿼리 엔진은 Pod 수를 늘리는 것 자체보다 쿼리 분산 방식, 데이터 위치, 메모리 사용량, 캐시, 클러스터 멤버십이 더 중요할 수 있다. 따라서 HPA를 적용하기 전에 해당 애플리케이션이 수평 확장을 지원하는지 확인해야 한다.

### 3-6) HPA와 VPA를 함께 쓸 때 주의점

HPA가 CPU utilization을 기준으로 동작하면 VPA가 CPU request를 바꾸는 순간 HPA 계산식도 바뀐다.

예를 들어 실제 CPU 사용량이 500m이고 CPU request가 1core라면 utilization은 50%이다.

```text
500m / 1000m = 50%
```

그런데 VPA가 request를 500m로 낮추면 같은 실제 사용량에서도 utilization은 100%가 된다.

```text
500m / 500m = 100%
```

이 경우 HPA가 불필요하게 scale out할 수 있다. 그래서 HPA와 VPA를 함께 쓸 때는 CPU utilization 기반 HPA 대신 custom metric 기반 HPA를 쓰거나, VPA를 recommendation mode로만 사용하는 방식이 더 안전할 수 있다.

## 4. VPA: Vertical Pod Autoscaler

### 4-1) VPA란 무엇인가

VPA는 Vertical Pod Autoscaler이다. Pod 개수를 늘리는 것이 아니라 Pod 하나의 CPU/Memory request/limit을 조정한다.

```text
CPU request 4core → 1core
Memory request 8Gi → 2Gi
```

Kubernetes 공식 문서 기준 VPA는 workload management resource, 예를 들어 Deployment나 StatefulSet을 자동 업데이트하여 실제 사용량에 맞게 infrastructure resource requests와 limits를 조정하는 기능이다. 다만 VPA는 Kubernetes 기본 설치에 항상 포함되는 기능이 아니며, 별도 add-on으로 배포해야 한다.

즉 HPA와 달리 VPA를 사용하려면 보통 다음이 필요하다.

```text
VPA CRD 설치
VPA Recommender 설치
VPA Updater 설치
VPA Admission Controller 설치
```

### 4-2) VPA가 필요한 이유

Kubernetes에서 request를 정확하게 설정하는 것은 어렵다. 개발 초기에는 장애를 피하기 위해 request를 크게 잡는 경우가 많다. 시간이 지나 실제 사용량을 보면 request 대비 usage가 매우 낮은 워크로드가 많을 수 있다.

예를 들어 다음과 같은 Pod가 있다고 하자.

```yaml
resources:
  requests:
    cpu: "4"
    memory: "8Gi"
  limits:
    cpu: "8"
    memory: "16Gi"
```

실제 사용량이 다음과 같다면 과다 할당 상태이다.

```text
CPU 평균 사용량: 200m
Memory 평균 사용량: 1Gi
```

이런 경우 VPA는 더 낮은 request를 추천할 수 있다.

반대로 request가 너무 낮은 경우도 있다. Memory request가 1Gi인데 실제 사용량이 3Gi 이상이고 OOMKilled가 발생한다면 VPA는 request를 올리도록 추천할 수 있다.

### 4-3) VPA가 분석하는 정보

VPA는 다음 정보를 기반으로 권장값을 계산한다.

```text
CPU 사용량
Memory 사용량
과거 사용 패턴
피크 사용량
OOM 이벤트
워크로드 안정성
```

CPU는 사용량 패턴을 기준으로 적정 request를 계산하기 쉽지만, Memory는 더 조심해야 한다. Memory는 순간적으로 부족하면 OOMKilled로 이어질 수 있기 때문에 평균 사용량만 보고 낮추면 위험하다. 특히 데이터베이스, 분석 엔진, 쿼리 엔진은 특정 쿼리에서 메모리가 급증할 수 있다.

### 4-4) VPA 구성요소

VPA는 보통 세 가지 구성요소로 설명한다.

```text
VPA Recommender
VPA Updater
VPA Admission Controller
```

각 구성요소의 역할은 다르다.

```text
Recommender:
Metrics를 분석해 적정 request 권장값을 계산

Updater:
기존 Pod가 권장값과 다를 때 업데이트를 유도

Admission Controller:
새 Pod가 생성될 때 추천값을 반영
```

운영 환경에서 가장 먼저 활용할 수 있는 것은 Recommender이다. Recommender는 바로 Pod를 바꾸지 않고 권장값을 계산하므로 위험이 낮다. Updater와 Admission Controller는 실제 워크로드 생성/변경 흐름에 영향을 주므로 더 신중하게 적용해야 한다.

### 4-5) VPA Recommender

VPA Recommender는 VPA의 핵심 컴포넌트이다. Pod와 컨테이너의 실제 리소스 사용량을 분석하고, 현재 request가 적정한지 판단한다.

Recommender가 생성하는 추천값은 보통 다음과 같은 의미를 가진다.

```text
lowerBound:
이보다 낮추면 위험할 수 있는 하한

target:
일반적으로 권장되는 request

upperBound:
이보다 높게 잡으면 과다할 수 있는 상한
```

실제 VPA 상태를 보면 recommendation이 다음처럼 나타난다.

```yaml
status:
  recommendation:
    containerRecommendations:
      - containerName: app
        lowerBound:
          cpu: 100m
          memory: 256Mi
        target:
          cpu: 500m
          memory: 1Gi
        uncappedTarget:
          cpu: 450m
          memory: 900Mi
        upperBound:
          cpu: "2"
          memory: 4Gi
```

이 값의 의미는 다음과 같이 해석할 수 있다.

```text
lowerBound:
너무 낮게 잡지 말아야 할 최소 수준

target:
현재 사용량 패턴 기준 추천값

uncappedTarget:
정책 제한을 적용하기 전 계산된 목표값

upperBound:
과도하게 높지 않은 상한 기준
```

운영자가 바로 target을 적용해도 되는 것은 아니다. 피크 시간대, 배치 작업, OOM 이력, 서비스 중요도, SLA를 함께 보고 조정해야 한다.

### 4-6) VPA Updater

VPA Updater는 기존 Pod의 리소스 설정이 추천값과 차이가 크면 업데이트를 유도한다. 과거 방식에서는 Pod resource 변경을 위해 Pod 재생성이 필요했기 때문에 Updater가 Pod를 evict하고 새 request로 다시 생성되게 할 수 있었다.

문제는 데이터베이스나 분석 엔진 같은 Stateful workload에서 Pod eviction이 서비스 영향으로 이어질 수 있다는 점이다.

```text
VPA Updater가 Pod evict
↓
Pod 재생성
↓
쿼리 중단 또는 캐시 손실
↓
클러스터 재조인 또는 복구 발생
```

따라서 StatefulSet, DB, 분석 엔진, 메시지 큐, 스토리지 시스템에는 VPA Updater 자동 적용을 바로 켜면 안 된다. 먼저 recommendation mode로 관측하고, 수동 또는 별도 자동화 정책으로 적용하는 것이 안전하다.

### 4-7) VPA Admission Controller

VPA Admission Controller는 새 Pod가 생성될 때 VPA 추천값을 반영할 수 있다. 예를 들어 Deployment가 새 Pod를 만들 때 Admission Controller가 Pod spec의 resources를 권장값으로 조정할 수 있다.

이 방식은 새로 생성되는 Pod에 적정 request를 적용하는 데 유용하지만, Admission Webhook은 Pod 생성 경로에 개입한다. Admission Controller에 문제가 생기면 Pod 생성이 지연되거나 실패할 수 있다.

따라서 운영 환경에서는 다음을 확인해야 한다.

```text
Admission Controller 고가용성
Webhook timeout 설정
실패 시 정책
적용 Namespace 범위
적용 대상 workload 범위
장애 시 Pod 생성 영향
```

### 4-8) VPA updateMode

VPA는 updateMode에 따라 동작 방식이 달라진다.

```text
Off:
추천값만 계산하고 실제 적용하지 않음

Initial:
Pod 생성 시점에만 추천값 적용

Auto:
VPA가 권장값을 자동 적용

Recreate:
필요 시 Pod를 재생성하여 권장값 적용
```

VPA 버전과 Kubernetes 기능 상태에 따라 InPlace 또는 InPlaceOrRecreate 같은 방식이 논의될 수 있다. 그러나 실제 사용 가능 여부는 Kubernetes 버전, VPA 버전, 기능 게이트, 설치 방식에 따라 달라진다. 운영 환경에서는 반드시 현재 클러스터의 VPA 버전과 공식 문서를 확인해야 한다.

### 4-9) VPA YAML 예시: Recommendation Only

운영 환경에서 가장 안전한 시작점은 VPA를 추천 모드로 사용하는 것이다. 아래 예시는 Deployment `web-api`에 대해 추천값만 계산하고 실제로 Pod를 변경하지 않는 VPA이다.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-api-vpa
  namespace: app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-api
  updatePolicy:
    updateMode: "Off"
```

이 설정의 의미는 다음과 같다.

```text
targetRef:
VPA가 관찰할 대상 workload

updateMode: "Off":
권장값만 계산하고 실제 Pod는 변경하지 않음
```

이 모드는 Goldilocks와 함께 사용하기 좋다. 실제 운영 리소스를 바꾸기 전에 추천값을 확인할 수 있기 때문이다.

### 4-10) VPA YAML 예시: Initial Mode

아래 예시는 Pod가 새로 생성될 때만 추천값을 반영하는 VPA이다.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: worker-vpa
  namespace: app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker
  updatePolicy:
    updateMode: "Initial"
```

`Initial` 모드는 기존 Pod를 건드리지 않고 새로 생성되는 Pod에만 권장 request를 적용한다. 기존 Pod 재시작이 부담스러운 환경에서 상대적으로 안전하게 시작할 수 있다.

### 4-11) VPA YAML 예시: resourcePolicy

VPA가 아무 값이나 추천하도록 두면 위험할 수 있다. `resourcePolicy`를 사용하면 컨테이너별 최소/최대 request 범위를 제한할 수 있다.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
  namespace: app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: api
        minAllowed:
          cpu: 200m
          memory: 512Mi
        maxAllowed:
          cpu: "4"
          memory: 8Gi
        controlledResources:
          - cpu
          - memory
```

이 설정은 VPA 추천값이 너무 낮거나 너무 높아지는 것을 방지한다.

```text
minAllowed:
이보다 낮게 추천하지 않도록 제한

maxAllowed:
이보다 높게 추천하지 않도록 제한

controlledResources:
VPA가 제어할 resource 지정
```

데이터 플랫폼 워크로드에서는 `minAllowed`가 중요하다. 평균 사용량이 낮다고 해서 request를 지나치게 낮추면 피크 쿼리에서 장애가 날 수 있기 때문이다.

### 4-12) VPA YAML 예시: 특정 리소스만 제어

CPU만 VPA 대상으로 삼고 Memory는 제외할 수도 있다.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: cpu-only-vpa
  namespace: app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: api
        controlledResources:
          - cpu
```

Memory는 OOM 위험이 크기 때문에 처음에는 CPU만 관측하거나 조정하는 전략도 가능하다. 특히 데이터베이스나 분석 엔진은 Memory 조정을 더 신중하게 해야 한다.

### 4-13) VPA 추천값 확인

VPA를 생성한 후에는 다음 명령으로 추천값을 확인할 수 있다.

```bash
kubectl describe vpa web-api-vpa -n app
```

또는 YAML로 확인할 수 있다.

```bash
kubectl get vpa web-api-vpa -n app -o yaml
```

확인할 부분은 `status.recommendation.containerRecommendations`이다.

```yaml
status:
  recommendation:
    containerRecommendations:
      - containerName: web-api
        lowerBound:
          cpu: 100m
          memory: 256Mi
        target:
          cpu: 500m
          memory: 1Gi
        upperBound:
          cpu: "2"
          memory: 4Gi
```

이 추천값을 기준으로 실제 request/limit을 조정할지 결정한다.

### 4-14) VPA 운영 전략

대규모 운영 환경에서는 다음 순서가 안전하다.

```text
1. VPA 설치
2. updateMode: Off로 추천값만 수집
3. Goldilocks Dashboard로 워크로드별 추천값 확인
4. request 대비 usage 차이가 큰 대상 선별
5. stateless workload부터 적용
6. Stateful workload는 별도 테스트
7. 적용 후 OOMKilled, CPU throttling, latency 확인
8. 문제가 없으면 자동화 범위 확대
```

VPA는 “자동으로 다 해결해주는 기능”이 아니라 “실제 사용량 기반으로 request를 조정할 근거를 제공하는 도구”로 보는 것이 안전하다.

## 5. Goldilocks

### 5-1) Goldilocks란 무엇인가

Goldilocks는 Fairwinds에서 만든 Kubernetes 리소스 권장 도구이다. Goldilocks는 Namespace의 각 workload에 VPA를 생성하고, 해당 VPA의 recommendation 정보를 조회해서 Dashboard에 보여준다.

중요한 점은 Goldilocks 자체가 복잡한 리소스 계산 엔진이 아니라는 점이다. Goldilocks는 VPA의 recommendation 기능을 활용해서 request/limit 권장값을 보기 쉽게 제공하는 도구에 가깝다.

```text
Metrics 수집
↓
VPA Recommender
↓
Goldilocks Controller
↓
Goldilocks Dashboard
↓
운영자 확인
```

### 5-2) Goldilocks가 필요한 이유

대규모 클러스터에서는 수백 개 또는 수천 개의 workload가 존재할 수 있다. 운영자가 각 Pod의 request와 실제 사용량을 직접 비교하는 것은 비효율적이다.

Goldilocks는 다음 질문에 답하는 데 도움을 준다.

```text
이 Pod의 CPU request는 너무 큰가?
이 Pod의 Memory request는 너무 작은가?
이 Namespace에서 낭비되는 request가 많은가?
어떤 워크로드부터 조정해야 효과가 큰가?
VPA 추천값 기준으로 right-sizing 후보는 무엇인가?
```

### 5-3) Goldilocks 동작 방식

Goldilocks는 보통 Namespace에 opt-in label을 붙인 뒤 해당 Namespace의 workload에 대해 VPA를 생성한다. 그런 다음 VPA recommendation을 Dashboard에서 확인한다.

예를 들어 특정 Namespace를 Goldilocks 대상으로 지정할 수 있다.

```bash
kubectl label namespace app goldilocks.fairwinds.com/enabled=true
```

이후 Goldilocks Controller가 해당 Namespace의 workload를 보고 VPA를 생성한다. 실제 label 이름이나 동작 방식은 Goldilocks 버전과 설치 설정에 따라 달라질 수 있으므로 공식 문서를 확인해야 한다.

### 5-4) Goldilocks 설치 개념

Goldilocks는 Helm Chart로 설치하는 경우가 많다. 개념적인 설치 흐름은 다음과 같다.

```bash
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm repo update
helm install goldilocks fairwinds-stable/goldilocks \
  --namespace goldilocks \
  --create-namespace
```

운영 환경에서는 values.yaml로 다음을 명확히 정해야 한다.

```text
VPA 설치 여부
Dashboard 노출 방식
대상 Namespace 범위
VPA updateMode
RBAC 범위
Ingress 또는 port-forward 접근 방식
```

Goldilocks가 VPA를 사용하므로, VPA CRD와 Recommender가 없으면 추천값을 만들 수 없다.

### 5-5) Goldilocks는 직접 리소스를 바꾸는가

Goldilocks는 기본적으로 추천 도구이다. 자동으로 모든 Pod의 request/limit을 안전하게 변경하는 만능 도구가 아니다. 실제 적용은 운영자가 결정해야 한다.

```yaml
requests:
  cpu: "4"
  memory: "8Gi"
```

실제 사용량이 CPU 200m, Memory 1.2Gi 수준이라면 Goldilocks/VPA는 다음과 같은 권장값을 제시할 수 있다.

```yaml
requests:
  cpu: "500m"
  memory: "2Gi"
```

이 추천값을 바로 운영에 적용할지, 일부만 적용할지, 피크 시간대를 더 관찰할지는 운영자가 판단해야 한다.

### 5-6) Goldilocks와 Right Sizing

Right Sizing은 워크로드에 맞게 request/limit을 적정하게 맞추는 작업이다. Goldilocks는 Right Sizing을 위한 판단 근거를 제공한다.

```text
현재 request
실제 usage
VPA recommendation
운영자가 정한 최소 request
SLA 기준 request
```

이 값들을 비교해서 최종 request/limit을 결정해야 한다.

Right Sizing 절차는 다음과 같이 진행하는 것이 안전하다.

```text
1. 현재 request/limit 수집
2. 실제 usage 수집
3. VPA/Goldilocks 추천값 확인
4. 평균이 아니라 p95/p99/peak 확인
5. OOMKilled와 throttling 이력 확인
6. 조정 후보 선정
7. 낮은 위험도의 workload부터 적용
8. 적용 후 지표 재확인
```

### 5-7) Goldilocks 적용 시 주의점

Goldilocks 추천값만 보고 바로 request를 낮추면 안 된다. 추천값은 과거 관측된 metrics를 기반으로 하므로 다음 상황을 놓칠 수 있다.

```text
월말 배치 작업
주간 리포트 쿼리
이벤트성 트래픽
장시간 실행 쿼리
운영자가 알고 있는 중요 피크
업무 시간 외 batch
```

따라서 Goldilocks는 최종 결정자가 아니라 판단 보조 도구로 사용해야 한다.

## 6. In-place Pod Resize

### 6-1) In-place Pod Resize란 무엇인가

In-place Pod Resize는 실행 중인 Pod의 CPU/Memory request/limit을 Pod 재생성 없이 변경하는 Kubernetes 기능이다. 과거에는 Pod의 container resources 필드가 사실상 immutable에 가까웠기 때문에 CPU/Memory를 바꾸려면 Pod를 삭제하고 다시 생성하는 방식이 필요했다.

Kubernetes 공식 문서 기준으로 In-place Pod Resize는 Kubernetes v1.35에서 stable이며, CPU/Memory request/limit을 Pod를 재생성하지 않고 변경할 수 있다. v1.33에서는 beta로 승격되었고, v1.35에서 stable로 승격되었다.

### 6-2) 왜 중요한가

대규모 클러스터에서 유휴 Pod가 높은 request를 계속 점유하는 경우, Pod를 삭제하지 않고 request를 줄일 수 있다면 Scheduler 기준 예약량을 줄일 수 있다.

```text
Idle Pod
CPU request 4core
Memory request 8Gi
↓
In-place Resize
↓
CPU request 500m
Memory request 2Gi
↓
Pod는 계속 Running
↓
Scheduler 기준 예약량 감소
```

이 방식은 특히 Stateful workload에 중요하다. 데이터베이스나 분석 엔진은 Pod를 재시작하면 캐시 손실, 쿼리 중단, 재조인, 리밸런싱 같은 영향이 생길 수 있기 때문이다.

### 6-3) resize subresource

최신 Kubernetes에서는 Pod의 `resize` subresource를 통해 CPU/Memory request/limit 변경을 요청할 수 있다.

```bash
kubectl patch pod example-pod \
  -n example-namespace \
  --subresource resize \
  --type merge \
  -p '{
    "spec": {
      "containers": [
        {
          "name": "app",
          "resources": {
            "requests": {
              "cpu": "500m",
              "memory": "2Gi"
            },
            "limits": {
              "cpu": "2",
              "memory": "4Gi"
            }
          }
        }
      ]
    }
  }'
```

실제 명령은 Kubernetes 버전, kubectl 버전, 권한, Pod 생성 방식, 컨테이너 이름, feature gate, 배포 도구에 따라 달라질 수 있다. 운영 환경에서는 반드시 테스트 클러스터에서 검증해야 한다.

### 6-4) resize 가능한 Pod 예시 YAML

In-place Resize를 고려한다면 Pod spec에 `resizePolicy`를 명시하는 것이 좋다. 아래 예시는 CPU는 재시작 없이 변경하고, Memory는 필요 시 컨테이너 재시작을 허용하는 설정이다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resize-demo
  namespace: app
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: "1"
          memory: "1Gi"
        limits:
          cpu: "2"
          memory: "2Gi"
      resizePolicy:
        - resourceName: cpu
          restartPolicy: NotRequired
        - resourceName: memory
          restartPolicy: RestartContainer
```

이 YAML에서 중요한 부분은 `resizePolicy`이다.

```text
cpu + NotRequired:
CPU 변경 시 컨테이너 재시작을 요구하지 않음

memory + RestartContainer:
Memory 변경 시 필요하면 컨테이너 재시작을 허용
```

Memory 축소는 CPU보다 위험하다. 프로세스가 이미 많은 메모리를 사용 중이라면 memory limit 축소가 즉시 적용되지 않거나 재시작이 필요할 수 있다.

### 6-5) desired resources와 actual resources

In-place Pod Resize에서는 원하는 값과 실제 적용된 값을 구분해서 봐야 한다. 사용자가 Pod spec에 새로운 resources를 요청하면 Kubernetes API에는 desired resources가 반영된다. 이후 kubelet이 해당 Node에서 실제 적용 가능 여부를 판단하고, 적용 결과가 status에 반영된다.

명령을 실행했다고 해서 즉시 모든 리소스가 적용되었다고 단정하면 안 된다. Pod 상태, resize condition, actual resources 반영 여부를 확인해야 한다.

```bash
kubectl get pod resize-demo -n app -o yaml
```

확인해야 할 항목은 다음과 같다.

```text
spec.containers[].resources:
원하는 리소스 값

status.containerStatuses[].resources:
실제 적용된 리소스 값

Pod conditions:
ResizePending, ResizeInProgress 등 resize 상태
```

정확한 status 필드 구조는 Kubernetes 버전과 구현 상태에 따라 다를 수 있으므로 현재 클러스터 버전의 공식 문서를 확인해야 한다.

### 6-6) In-place Resize 제약사항

In-place Resize는 만능이 아니다. 공식 문서 기준 CPU와 Memory resource 변경에 초점이 맞춰져 있고, QoS Class 변경 등은 제한된다. 또한 Windows Pod, 일부 container 유형, init container, ephemeral container 등에 제약이 있을 수 있다.

운영 전에 확인해야 할 점은 다음과 같다.

```text
Kubernetes 서버 버전
kubectl 버전
kubelet 지원 여부
feature gate 상태
Pod QoS Class 변경 여부
resizePolicy 설정
Memory 축소 시 재시작 가능성
StatefulSet/Helm/Operator의 reconcile 여부
RBAC 권한
```

특히 StatefulSet, Helm, Operator로 관리되는 Pod는 직접 patch를 해도 상위 리소스가 다시 원래 상태로 되돌릴 수 있다. 예를 들어 Operator가 Pod spec을 계속 reconcile한다면 Pod에 직접 patch한 값이 유지되지 않을 수 있다.

### 6-7) In-place Resize와 VPA의 관계

VPA는 request/limit 추천값을 계산하거나 자동 적용하는 역할을 한다. In-place Resize는 실행 중인 Pod의 resource를 바꾸는 Kubernetes 기능이다. 둘은 같은 것이 아니다.

```text
VPA:
얼마로 바꾸면 좋을지 추천하거나 적용

In-place Pod Resize:
실행 중인 Pod의 CPU/Memory 변경을 가능하게 하는 메커니즘
```

따라서 현실적인 조합은 다음과 같다.

```text
VPA Recommender / Goldilocks
↓
적정 request 추천
↓
운영 정책 검토
↓
In-place Pod Resize 또는 patch로 적용
```

## 7. kubectl edit과 kubectl patch

### 7-1) kubectl edit

`kubectl edit`은 YAML을 편집기에서 열어 사람이 직접 수정하는 방식이다. 공식 문서 기준 edit은 command-line tools로 가져올 수 있는 API resource를 기본 편집기에서 직접 수정하게 해준다.

```bash
kubectl edit pod example-pod -n example-namespace
```

이 명령을 실행하면 로컬 편집기가 열리고, 사용자는 리소스 YAML을 직접 수정한다. 저장하면 Kubernetes API Server에 변경 요청이 전달된다.

`edit`은 다음 상황에 적합하다.

```text
단건 테스트
현재 리소스 구조 확인
운영자가 직접 수동 수정
자동화 전 사전 검증
```

그러나 대규모 운영에는 적합하지 않다.

```text
사람이 직접 수정하므로 실수 가능
변경 이력 관리가 어려움
수천 개 리소스에 반복 적용하기 어려움
GitOps와 충돌 가능
```

### 7-2) kubectl patch

`kubectl patch`는 특정 필드만 명령어로 수정하는 방식이다. 공식 문서 기준 kubectl patch는 strategic merge patch, JSON merge patch, JSON patch를 사용해 리소스 필드를 업데이트할 수 있다.

```bash
kubectl patch pod example-pod -n example-namespace --type merge -p '...'
```

patch는 스크립트, CronJob, 운영 자동화에 적합하다. 대규모 환경에서는 유휴 Pod의 request를 정책에 따라 일괄 축소하는 자동화를 만들 때 patch가 더 적합하다.

### 7-3) patch type

`kubectl patch`에는 여러 patch type이 있다.

```text
strategic merge patch:
Kubernetes built-in type에 대해 구조를 이해하고 merge

JSON merge patch:
JSON object를 merge하는 방식

JSON patch:
add, remove, replace 같은 operation 목록으로 변경
```

Custom Resource에는 strategic merge patch가 지원되지 않을 수 있다. 따라서 Operator CR 같은 Custom Resource를 patch할 때는 merge patch나 JSON patch를 써야 할 수 있다.

예를 들어 Deployment의 replicas를 merge patch로 바꾸는 명령은 다음과 같다.

```bash
kubectl patch deployment web-api \
  -n app \
  --type merge \
  -p '{"spec":{"replicas":5}}'
```

JSON patch 예시는 다음과 같다.

```bash
kubectl patch deployment web-api \
  -n app \
  --type json \
  -p='[
    {
      "op": "replace",
      "path": "/spec/replicas",
      "value": 5
    }
  ]'
```

### 7-4) Patch 기반 리소스 축소 자동화

대규모 환경에서는 patch를 수동으로 실행하면 안 된다. 다음과 같은 자동화 구조가 필요하다.

```text
Prometheus / Metrics Server에서 사용량 수집
↓
최근 사용량과 요청 발생 여부 분석
↓
Idle Workload 목록 산정
↓
Goldilocks/VPA 추천값 참고
↓
Patch로 request 축소
↓
적용 상태 확인
↓
결과 보고
```

Patch 자동화는 반드시 보수적으로 설계해야 한다. 잘못된 patch는 대량 장애로 이어질 수 있다.

### 7-5) Patch 적용 전 체크리스트

Patch 적용 전에는 다음을 확인해야 한다.

```text
대상 Namespace가 맞는가
대상 Pod/Workload가 맞는가
컨테이너 이름이 맞는가
현재 request/limit 값은 무엇인가
추천값은 어디서 왔는가
최소 request 정책을 만족하는가
Memory 축소 시 OOM 위험은 없는가
상위 리소스가 원복하지 않는가
롤백 값은 준비되어 있는가
```

### 7-6) Patch 적용 후 체크리스트

Patch를 적용한 뒤에는 다음을 확인해야 한다.

```bash
kubectl get pod <pod-name> -n <namespace> -o yaml
kubectl describe pod <pod-name> -n <namespace>
kubectl describe node <node-name>
kubectl top pod -n <namespace>
```

확인해야 할 항목은 다음과 같다.

```text
Pod가 Running 상태인지
resize 요청이 실제 적용되었는지
OOMKilled가 발생하지 않았는지
CPU throttling이 증가하지 않았는지
Node의 allocated resources가 줄었는지
Scheduler Pending 문제가 완화되었는지
상위 리소스가 값을 원복하지 않았는지
```

### 7-7) 자동화 정책 예시

대규모 환경에서는 사용량 기준으로 workload를 분류할 수 있다.

```text
Active:
최근 사용량 높음
request 유지 또는 증가

Idle:
최근 사용량 낮음
request 축소

Dormant:
장기간 사용 없음
scale down 또는 중지 후보
```

각 상태마다 resource profile을 정의한다.

```text
active-profile:
cpu request 4core
memory request 8Gi

idle-profile:
cpu request 500m
memory request 2Gi

dormant-profile:
scale down 또는 최소 리소스 유지
```

자동화는 다음 순서로 동작할 수 있다.

```text
1. 최근 24시간 usage 조회
2. 최근 요청/쿼리 여부 조회
3. Idle 후보 선정
4. VPA/Goldilocks 추천값과 비교
5. idle-profile보다 낮추지 않도록 제한
6. patch 적용
7. 적용 결과 확인
8. 변경 이력 기록
9. 문제 발생 시 이전 profile로 복구
```

## 8. Serverless Computing

### 8-1) Serverless란 무엇인가

Serverless는 서버가 실제로 없다는 뜻이 아니다. 서버는 존재하지만 사용자가 서버를 직접 프로비저닝하거나 관리하지 않는다는 뜻이다. 인프라 관리 책임을 플랫폼이 가져가고, 사용자는 코드나 컨테이너 실행 단위에 집중한다.

Serverless의 핵심 특징은 다음과 같다.

```text
이벤트 기반 실행
자동 확장
사용하지 않을 때 비용 또는 리소스 절감
서버 직접 관리 최소화
```

### 8-2) Event Driven Architecture

Event Driven Architecture는 이벤트가 발생하면 그 이벤트를 트리거로 특정 작업이 실행되는 구조이다.

```text
파일 업로드 이벤트
↓
처리 함수 실행
↓
결과 저장
```

서버리스는 이벤트 기반 구조와 잘 맞는다. 요청이나 이벤트가 없으면 실행 환경을 줄이고, 이벤트가 발생하면 필요한 실행 단위를 올린다.

### 8-3) Serverless가 잘 맞는 작업

Serverless는 다음 작업에 잘 맞는다.

```text
짧은 API 처리
파일 업로드 후처리
이미지 리사이징
이벤트 기반 알림
비동기 작업
주기적 작업
Webhook 처리
```

이런 작업은 항상 서버를 띄워둘 필요가 없고, 이벤트가 있을 때만 실행해도 된다.

### 8-4) Serverless가 애매한 작업

Serverless가 잘 맞지 않는 작업도 있다.

```text
상태가 있는 데이터베이스
장시간 실행되는 분석 엔진
큰 메모리를 지속적으로 요구하는 서비스
낮은 지연시간이 절대적으로 중요한 서비스
클러스터 멤버십이 필요한 분산 시스템
```

이런 워크로드는 scale-to-zero나 cold start가 장애 또는 성능 문제로 이어질 수 있다.

## 9. AWS Lambda

### 9-1) Lambda란 무엇인가

AWS Lambda는 AWS의 대표적인 서버리스 컴퓨팅 서비스이다. AWS 공식 문서 기준 Lambda는 서버를 프로비저닝하거나 관리하지 않고 코드를 실행할 수 있는 서버리스 컴퓨팅 서비스이다. Lambda는 필요할 때 코드를 실행하고 자동으로 확장되며, 사용한 compute time에 대해 비용을 지불한다.

예를 들어 S3에 파일이 업로드되면 Lambda가 실행되어 썸네일을 만들고, 작업이 끝나면 실행 환경이 줄어든다.

```text
S3 파일 업로드 이벤트
↓
Lambda 실행
↓
이미지 리사이징
↓
결과 저장
↓
실행 종료
```

또는 API Gateway 요청을 받아 Lambda가 실행되고 응답을 반환할 수 있다.

```text
API Gateway 요청
↓
Lambda 실행
↓
응답 반환
```

### 9-2) Lambda의 장점

Lambda의 장점은 다음과 같다.

```text
서버 관리 불필요
이벤트 기반 실행
자동 확장
사용량 기반 과금
AWS 서비스와 쉬운 연동
짧은 작업 처리에 적합
```

운영자는 EC2 인스턴스 크기나 서버 패치보다 함수 코드, IAM 권한, 이벤트 소스, timeout, memory 설정, 로그를 중심으로 관리한다.

### 9-3) Lambda의 한계

Lambda는 모든 워크로드에 적합하지 않다.

```text
실행 시간 제한
Cold Start
상태 유지 어려움
로컬 디스크 제약
네트워크/VPC 설정 복잡성
장시간 연결 유지에 부적합
대규모 상태성 시스템에 부적합
```

특히 데이터베이스나 분산 분석 엔진 같은 시스템은 Lambda로 대체하는 대상이 아니다. Lambda는 이벤트 처리 단위에 가깝고, 데이터베이스는 지속적으로 상태와 데이터를 관리하는 시스템이기 때문이다.

### 9-4) Lambda와 Kubernetes의 차이

Lambda는 AWS가 인프라를 관리하는 서버리스 서비스이다. Kubernetes는 사용자가 직접 클러스터와 워크로드를 관리하는 컨테이너 오케스트레이션 플랫폼이다.

```text
AWS Lambda:
함수 단위 실행
AWS가 서버 관리
이벤트 기반
사용량 기반 과금

Kubernetes:
컨테이너 단위 실행
클러스터 운영 필요
Deployment/StatefulSet/Service 관리
Pod/Node 리소스 직접 설계
```

Knative는 Kubernetes 위에서 Lambda와 유사한 서버리스 운영 경험을 제공하려는 프레임워크로 볼 수 있다.

## 10. Knative

### 10-1) Knative란 무엇인가

Knative는 Kubernetes 위에서 서버리스 스타일로 애플리케이션을 배포하고 운영하기 위한 프레임워크이다. Knative Serving은 컨테이너 기반 애플리케이션을 배포하고, 트래픽에 따라 자동으로 scale out/scale in 할 수 있게 해준다.

```text
AWS Lambda:
AWS가 제공하는 관리형 서버리스 컴퓨팅 서비스

Knative:
내 Kubernetes 클러스터 위에 설치해서 사용하는 서버리스 프레임워크
```

Knative는 Kubernetes의 Deployment, Service, Ingress 같은 일반 리소스와는 다른 Serving 모델을 제공한다. Revision, Route, Configuration 같은 개념을 통해 트래픽 분산, 버전 관리, autoscaling을 지원한다.

### 10-2) Knative Service YAML 예시

아래는 간단한 Knative Service 예시이다.

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
  namespace: app
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"
        autoscaling.knative.dev/max-scale: "10"
    spec:
      containers:
        - image: gcr.io/knative-samples/helloworld-go
          env:
            - name: TARGET
              value: "Kubernetes"
```

이 설정의 의미는 다음과 같다.

```text
min-scale: "0":
트래픽이 없을 때 0개까지 축소 가능

max-scale: "10":
최대 10개 replica까지 확장 가능

containers.image:
실행할 컨테이너 이미지
```

실제 annotation 이름과 동작은 Knative 버전 및 클러스터 설정에 따라 달라질 수 있으므로 공식 문서를 확인해야 한다.

### 10-3) Scale to Zero

Scale to Zero는 요청이 없을 때 replica를 0까지 줄이는 기능이다. Knative 공식 문서 기준 scale-to-zero 값이 true이면 replica를 0까지 줄일 수 있고, false이면 최소 1개 replica에서 멈춘다.

```text
요청 없음
↓
Pod 0개
↓
요청 발생
↓
Activator가 요청 수신
↓
Pod 생성
↓
요청 처리
```

이 구조는 유휴 시간에 리소스를 거의 사용하지 않게 만들 수 있다. 다만 요청이 다시 들어왔을 때 Pod를 새로 띄워야 하므로 초기 응답 지연이 발생할 수 있다.

### 10-4) Knative Autoscaling의 주요 개념

Knative Autoscaling은 단순 CPU 사용량만 보는 것이 아니라 concurrency 기반으로 동작하는 경우가 많다. 즉 동시에 처리 중인 요청 수를 기준으로 replica를 늘리거나 줄일 수 있다.

주요 개념은 다음과 같다.

```text
Revision:
특정 버전의 배포 단위

Route:
트래픽을 어떤 Revision으로 보낼지 결정

Activator:
scale-to-zero 상태에서 최초 요청을 받아 Pod가 뜰 때까지 중계

Autoscaler:
트래픽과 concurrency를 보고 replica 수 조정

min-scale / max-scale:
최소/최대 replica 경계
```

### 10-5) Knative가 잘 맞는 워크로드

Knative는 stateless하고 요청 기반으로 동작하는 서비스에 잘 맞는다.

```text
REST API
Webhook
이미지 변환 서비스
이벤트 처리 함수
짧은 백엔드 작업
비동기 이벤트 처리
```

이런 서비스는 요청이 없을 때 내려가 있어도 큰 문제가 없고, 요청이 들어오면 다시 떠서 처리할 수 있다.

### 10-6) Knative가 데이터베이스 계열 워크로드에 애매한 이유

분산 데이터베이스나 분석 엔진은 Knative와 잘 맞지 않을 수 있다. 이런 시스템은 상태, 메타데이터, 캐시, 세션, 스토리지 연결, 클러스터 멤버십을 가진다.

데이터베이스 계열 워크로드를 scale-to-zero로 내리면 다음 문제가 생길 수 있다.

```text
Cold Start 시간이 길 수 있음
쿼리 세션이 끊길 수 있음
캐시가 사라짐
컴포넌트 재기동 순서가 중요할 수 있음
StatefulSet/PVC/Operator와 충돌 가능
데이터 일관성 및 복구 시간 검토 필요
```

따라서 데이터 플랫폼 리소스 최적화에서는 Knative가 아이디어 검토 대상은 될 수 있지만, 가장 먼저 적용할 현실적 해법은 VPA/Goldilocks 기반 right-sizing과 In-place Pod Resize인 경우가 많다.

## 11. 유휴 리소스 회수 전략

### 11-1) 유휴 리소스 회수란 무엇인가

유휴 리소스 회수는 실제로 사용하지 않는 워크로드가 계속 request를 점유하는 문제를 해결하는 작업이다. Kubernetes Scheduler는 request를 기준으로 Pod 배치 가능 여부를 판단하므로, 유휴 Pod가 높은 request를 계속 가지고 있으면 클러스터 전체 효율이 낮아진다.

유휴 리소스 회수의 목표는 다음과 같다.

```text
실제 사용하지 않는 workload 식별
↓
request/limit을 낮춤
↓
필요 시 scale down
↓
사용자가 다시 활성화되면 복구
```

### 11-2) Idle / Dormant 분류

워크로드를 다음과 같이 나눌 수 있다.

```text
Active:
최근 사용량이 높고 요청이 있음

Idle:
최근 사용량은 낮지만 언제든 다시 사용될 수 있음

Dormant:
장기간 사용 이력이 거의 없음
```

각 상태별로 다른 정책을 적용한다.

```text
Active:
기존 request 유지 또는 상향

Idle:
request 축소

Dormant:
scale down 또는 중지 후보
```

### 11-3) 유휴 판단 기준

단순 CPU 사용량만으로 Idle을 판단하면 안 된다. 데이터 플랫폼에서는 CPU는 낮지만 중요한 쿼리 대기 상태일 수 있고, 메모리 캐시가 의미 있을 수 있다.

함께 봐야 할 기준은 다음과 같다.

```text
최근 CPU 사용량
최근 Memory working set
최근 요청 수
최근 쿼리 수
마지막 접속 시각
마지막 쿼리 시각
업무 중요도
SLA
OOMKilled 이력
CPU throttling 이력
```

### 11-4) 회수 방법

유휴 리소스 회수 방법은 여러 단계가 있다.

```text
1단계:
request만 축소

2단계:
limit도 일부 축소

3단계:
replica 수 축소

4단계:
scale-to-zero 또는 중지
```

데이터베이스나 분석 엔진은 바로 scale-to-zero로 내리기보다 request 축소부터 시작하는 것이 안전하다.

### 11-5) 회수 후 복구

사용자가 다시 활성화되면 리소스를 복구해야 한다.

```text
사용자 요청 발생
↓
Idle profile 감지
↓
Active profile로 request 상향
↓
필요 시 replica 증가
↓
서비스 정상 처리
```

자동화에는 반드시 원복 정책이 있어야 한다. 줄이는 것만 자동화하고 복구를 수동으로 두면 운영 장애로 이어질 수 있다.

## 12. 전체 적용 전략

### 12-1) 안전한 적용 순서

대규모 운영 환경에서 Autoscaling과 Optimization은 다음 순서로 적용하는 것이 안전하다.

```text
1. 현재 request/limit 수집
2. 실제 usage 수집
3. Pending, OOMKilled, CPU throttling 확인
4. VPA를 updateMode Off로 적용
5. Goldilocks Dashboard로 추천값 확인
6. 낮은 위험도의 stateless workload부터 right-sizing
7. In-place Resize를 테스트 환경에서 검증
8. 운영 환경 일부 Namespace에 제한 적용
9. 적용 결과 모니터링
10. 자동화 범위 확대
```

### 12-2) 데이터 플랫폼 워크로드에 대한 원칙

데이터 플랫폼 워크로드에는 다음 원칙이 필요하다.

```text
평균 사용량만 보고 낮추지 않는다.
Memory는 CPU보다 보수적으로 조정한다.
Stateful workload는 자동 eviction을 피한다.
VPA는 처음에 recommendation mode로 사용한다.
Goldilocks는 추천 도구로 사용한다.
In-place Resize는 테스트 후 적용한다.
Knative scale-to-zero는 stateless 서비스부터 검토한다.
상위 리소스 Helm/Operator spec과의 일관성을 확인한다.
```

## 13. 최종 정리

Kubernetes Autoscaling과 Optimization은 하나의 기능으로 해결되는 문제가 아니다. HPA, VPA, Goldilocks, In-place Pod Resize, patch 자동화, Knative는 각각 해결하는 문제가 다르다.

```text
HPA:
Pod 개수 조정

VPA:
Pod request/limit 권장값 계산 또는 조정

Goldilocks:
VPA recommendation을 Dashboard로 제공

In-place Pod Resize:
Pod 재시작 없이 CPU/Memory request/limit 변경

kubectl patch:
대량 자동화에 적합한 필드 변경 방식

Knative:
요청 기반 scale-to-zero를 제공하는 Kubernetes 서버리스 프레임워크

AWS Lambda:
AWS 관리형 서버리스 컴퓨팅 서비스
```

대규모 데이터 플랫폼에서는 무작정 scale-to-zero를 적용하기보다 먼저 실제 사용량을 관측하고 VPA/Goldilocks로 적정 request를 계산한 뒤, In-place Pod Resize와 patch 자동화로 유휴 리소스를 줄이는 방식이 현실적이다. 이후 장기적으로는 workload 상태별 profile, Shared Cluster, Multi-Tenancy 구조 개선과 연결해야 한다.