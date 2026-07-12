---
layout: post
title: "Kubernetes Resource Management: Request, Limit, Quota, Scheduler, Overcommit"
date: 2026-06-23 17:30:00 +0900
categories: [Kubernetes, Infrastructure]
tags: [Kubernetes, RequestLimit, ResourceQuota, LimitRange, Scheduler, QoS, Overcommit, CapacityPlanning]
published: true
---

## 1. 글의 목적

대규모 Kubernetes 클러스터에서 리소스 부족 문제가 발생했다고 할 때, 실제 원인이 항상 물리 CPU나 Memory 부족인 것은 아니다. Kubernetes는 Pod를 Node에 배치할 때 현재 CPU 사용률이나 현재 Memory 사용량만 보고 판단하지 않는다. Scheduler는 Pod에 선언된 `requests`를 기준으로 Node에 배치 가능한지 판단한다.

따라서 실제 사용량은 낮지만 `requests`가 과도하게 잡혀 있으면, 클러스터는 겉보기에는 여유가 있어 보여도 새 Pod가 Pending 상태에 머물 수 있다. 이 글은 Kubernetes 리소스 관리의 기본인 Request, Limit, ResourceQuota, LimitRange, Scheduler, QoS Class, Resource Overcommit, Capacity Planning을 연결해서 정리한다.

공식 문서:

- Kubernetes Resource Management for Pods and Containers: <https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/>
- Kubernetes Resource Quotas: <https://kubernetes.io/docs/concepts/policy/resource-quotas/>
- Kubernetes LimitRange: <https://kubernetes.io/docs/concepts/policy/limit-range/>
- Kubernetes kube-scheduler: <https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/>
- Kubernetes Pod QoS Classes: <https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/>
- Kubernetes Node-pressure Eviction: <https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/>
- Kubernetes Reserve Compute Resources for System Daemons: <https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/>

## 2. Request / Limit

### 2-1) Request란 무엇인가

Kubernetes에서 `request`는 컨테이너가 실행되기 위해 최소한 필요하다고 선언하는 CPU와 Memory 양이다. 이 값은 단순 참고값이 아니라 Scheduler가 Pod를 어느 Node에 배치할지 판단할 때 사용하는 핵심 기준이다.

예를 들어 다음과 같은 설정이 있다고 하자.

```yaml
resources:
  requests:
    cpu: "4"
    memory: "8Gi"
```

이 설정은 해당 컨테이너가 최소 CPU 4core와 Memory 8Gi를 필요로 한다고 Kubernetes에 선언하는 것이다. 실제로 컨테이너가 CPU를 0.1core만 사용하고 있더라도 Scheduler는 이 Pod가 CPU 4core를 예약하고 있다고 계산한다. 즉 `request`는 현재 사용량이 아니라 스케줄링 관점의 예약량에 가깝다.

대규모 클러스터에서 request를 과도하게 설정하면 실제 사용량은 낮아도 Scheduler 기준으로는 클러스터 자원이 이미 많이 예약된 것처럼 보일 수 있다. 이때 운영자는 `kubectl top node`나 모니터링 대시보드에서 실제 CPU 사용률이 낮은 것을 보고 “왜 Pod가 안 뜨지?”라고 생각할 수 있다. 하지만 Scheduler는 실사용량이 아니라 request를 기준으로 판단하기 때문에 이런 현상이 발생한다.

### 2-2) Limit란 무엇인가

`limit`은 컨테이너가 사용할 수 있는 최대 CPU/Memory 상한이다.

```yaml
resources:
  limits:
    cpu: "8"
    memory: "16Gi"
```

CPU limit과 Memory limit은 동작 방식이 다르다. CPU는 limit을 초과하려 하면 Linux cgroup에 의해 throttling이 발생할 수 있다. 프로세스가 CPU를 더 쓰고 싶어도 제한 때문에 대기하게 되는 것이다. 애플리케이션 입장에서는 응답 지연, 처리량 저하, 쿼리 지연으로 보일 수 있다.

Memory는 더 직접적인 장애로 이어질 수 있다. 컨테이너가 Memory limit을 초과하면 OOMKilled가 발생할 수 있다. CPU는 제한되면 느려지는 방향으로 나타나는 경우가 많지만, Memory는 프로세스 종료로 이어질 수 있으므로 더 보수적으로 설계해야 한다.

### 2-3) CPU Request / Limit

Kubernetes CPU는 core 또는 millicore 단위로 표현한다. `cpu: "1"`은 CPU 1core를 의미하고, `cpu: "500m"`은 CPU 0.5core를 의미한다. 여기서 `m`은 millicore이며 1000m는 1core와 같다.

CPU request는 Scheduler가 Node 배치 가능 여부를 계산할 때 사용하는 값이다. CPU limit은 컨테이너가 런타임에 사용할 수 있는 상한이다. CPU limit을 너무 낮게 잡으면 CPU throttling이 발생할 수 있고, 분석 DB, 쿼리 엔진, 데이터 처리 엔진처럼 CPU를 많이 쓰는 워크로드에서는 쿼리 지연이나 처리량 저하로 직접 나타날 수 있다.

### 2-4) Memory Request / Limit

Memory request도 스케줄링 기준이다. Scheduler는 Node의 allocatable memory에서 이미 예약된 request 합계를 빼고, 새 Pod의 memory request를 만족할 수 있는지 판단한다.

Memory limit은 컨테이너가 사용할 수 있는 최대 메모리 상한이다. Memory limit을 초과하면 컨테이너는 OOMKilled될 수 있다. 데이터베이스나 분석 엔진은 쿼리 실행 중 일시적으로 메모리를 많이 사용할 수 있으므로 Memory limit을 무작정 낮추면 운영 장애로 이어질 수 있다.

분석 워크로드는 평상시에는 메모리를 적게 쓰다가 특정 쿼리, join, aggregation, batch 작업에서 메모리 사용량이 급증할 수 있다. 이런 워크로드에서는 평균 사용량만 보고 request/limit을 낮추면 위험하다. 반드시 피크 사용량, OOM 이력, 쿼리 패턴, 캐시 사용량을 함께 봐야 한다.

### 2-5) Request와 Limit의 차이

Request와 Limit은 목적이 다르다.

```text
Request = Scheduler가 Pod를 배치할 때 보는 예약 기준
Limit   = 컨테이너가 런타임에 사용할 수 있는 최대 상한
```

다음 예시를 보자.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "2Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

이 경우 Scheduler는 이 Pod를 CPU 0.5core, Memory 2Gi를 예약하는 Pod로 본다. 하지만 실제 컨테이너는 부하가 올라가면 CPU 4core, Memory 8Gi까지 사용할 수 있다. 이 방식은 클러스터의 예약량을 줄이는 데 유리하지만, 여러 Pod가 동시에 limit 가까이 사용하면 Node 전체 자원이 부족해질 수 있다.

## 3. ResourceQuota

### 3-1) ResourceQuota란 무엇인가

ResourceQuota는 Kubernetes Namespace 단위로 리소스 사용량을 제한하는 정책 객체이다. 특정 Namespace 안에서 사용할 수 있는 CPU, Memory, Storage, Pod 개수, Service 개수 등을 제한할 수 있다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: user-quota
  namespace: user-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "50"
```

이 설정은 `user-a` Namespace 안에서 CPU request 총합은 20core, Memory request 총합은 40Gi, CPU limit 총합은 40core, Memory limit 총합은 80Gi, Pod 개수는 50개까지로 제한한다.

### 3-2) ResourceQuota가 필요한 이유

대규모 멀티테넌트 Kubernetes 환경에서는 여러 사용자, 팀, 조직이 하나의 클러스터를 공유한다. 특정 사용자가 무제한으로 Pod를 생성하거나 CPU/Memory를 과도하게 요청하면 다른 사용자의 워크로드가 영향을 받을 수 있다. ResourceQuota는 이런 상황을 막기 위한 기본 안전장치이다.

### 3-3) ResourceQuota와 Request/Limit의 관계

ResourceQuota는 실제 사용량을 제한하는 것이 아니라 Kubernetes 객체에 선언된 request/limit의 총합을 기준으로 제한한다. 예를 들어 Namespace의 `requests.cpu` quota가 20core이고, Pod 하나가 CPU request 4core를 요구한다면 이 Namespace에는 이론상 5개까지만 해당 Pod를 만들 수 있다.

```text
20core / 4core = 5개
```

실제 CPU 사용량이 낮아도 ResourceQuota는 request 합계를 기준으로 제한한다. 이 점은 Scheduler의 동작 방식과 비슷하다.

### 3-4) ResourceQuota의 한계

ResourceQuota는 사용량 상한을 제한하는 기능이지, 실제 사용하지 않는 리소스를 자동으로 회수하는 기능은 아니다. 어떤 Namespace에 무거운 데이터 애플리케이션이 떠 있고 request가 이미 잡혀 있다면, 사용자가 실제로 아무 쿼리를 실행하지 않아도 ResourceQuota는 그 request를 사용 중인 것으로 본다.

ResourceQuota는 유휴 Pod가 request를 계속 점유하는 문제, 실제 사용량과 request 사이의 괴리, 사용하지 않는 사용자 환경을 자동 축소하는 문제를 직접 해결하지 못한다. 따라서 ResourceQuota는 격리에는 필요하지만, 리소스 최적화에는 VPA, Goldilocks, In-place Resize, 운영 자동화가 추가로 필요하다.

## 4. LimitRange

### 4-1) LimitRange란 무엇인가

LimitRange는 Namespace 안에서 개별 Pod 또는 Container가 가질 수 있는 request/limit의 기본값, 최소값, 최대값을 제한하는 정책 객체이다. ResourceQuota가 Namespace 전체 총량을 제한한다면, LimitRange는 개별 컨테이너 또는 Pod 단위의 기본값과 범위를 제한한다.

```text
ResourceQuota = Namespace 전체 총량 제한
LimitRange    = 개별 Container/Pod의 기본값, 최소값, 최대값 제한
```

### 4-2) LimitRange 예시

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-container-limits
  namespace: user-a
spec:
  limits:
  - type: Container
    default:
      cpu: "1"
      memory: "2Gi"
    defaultRequest:
      cpu: "500m"
      memory: "1Gi"
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "100m"
      memory: "256Mi"
```

이 설정은 Namespace 안에서 생성되는 컨테이너에 기본 request/limit을 부여하고, 너무 작거나 너무 큰 값을 막는다.

### 4-3) ResourceQuota와 LimitRange를 같이 써야 하는 이유

ResourceQuota만 있으면 사용자가 개별 Pod에 지나치게 큰 request를 넣는 것을 세밀하게 막기 어렵다. LimitRange만 있으면 Namespace 전체 사용량이 너무 커지는 것을 막기 어렵다. 사용자별 Namespace 운영에서는 ResourceQuota와 LimitRange를 함께 설계해야 한다.

## 5. Scheduler 동작 원리

### 5-1) Scheduler의 역할

Kubernetes Scheduler는 아직 Node에 배치되지 않은 Pending 상태의 Pod를 어느 Node에 실행할지 결정하는 컴포넌트이다. Pod가 생성되면 처음에는 특정 Node가 정해져 있지 않다. Scheduler는 Node들의 상태, Pod의 request, taint/toleration, nodeSelector, affinity, topology spread constraint 등을 종합해서 적절한 Node를 선택한다.

### 5-2) Filtering과 Scoring

Scheduler는 크게 Filtering과 Scoring 단계를 거친다. Filtering 단계에서는 Pod가 배치될 수 없는 Node를 제외한다. Node에 충분한 CPU/Memory request 여유가 없거나, taint/toleration 조건이 맞지 않거나, nodeSelector/nodeAffinity 조건이 맞지 않으면 후보에서 제외된다.

Scoring 단계에서는 Filtering을 통과한 Node들 중에서 어떤 Node가 더 적합한지 점수를 매긴다. 가장 점수가 높은 Node에 Pod를 배치한다. 대규모 클러스터에서는 request가 과도하게 잡힌 Pod가 많으면 Filtering 단계에서 대부분의 Node가 탈락할 수 있다.

### 5-3) Scheduler는 실사용량이 아니라 Request를 본다

Scheduler는 Pod를 배치할 때 Node의 실제 CPU 사용률이나 실제 Memory 사용량을 기준으로 계산하지 않는다. 기존 Pod들의 request 합계와 새 Pod의 request를 보고 Node의 allocatable resource 안에 들어가는지 판단한다.

```text
Node allocatable CPU: 64core
이미 배치된 Pod들의 CPU request 합계: 60core
새로 배치할 Pod의 CPU request: 8core
```

실제 CPU 사용량이 10core뿐이어도 Scheduler는 새 Pod를 이 Node에 배치하지 않는다.

```text
60core + 8core = 68core > 64core
```

즉 “실제로 남아 있는 것 같은데 왜 Pod가 안 뜨지?”라는 상황은 request 기준 계산 때문에 발생하는 경우가 많다.

### 5-4) Pending 상태에서 확인할 것

Pod가 Pending 상태라면 먼저 Events를 확인해야 한다.

```bash
kubectl describe pod <pod-name> -n <namespace>
```

다음과 비슷한 메시지가 나올 수 있다.

```text
0/300 nodes are available: insufficient cpu, insufficient memory
```

이 메시지는 Node 전체에 실제 CPU가 없다는 뜻이 아니다. Scheduler 관점에서 새 Pod의 request를 만족할 수 있는 Node가 없다는 뜻이다.

Node의 예약 상태는 다음 명령으로 확인할 수 있다.

```bash
kubectl describe node <node-name>
```

출력의 `Allocated resources` 부분을 보면 해당 Node에 이미 배치된 Pod들의 request와 limit 합계를 확인할 수 있다.

## 6. Resource Reservation, Utilization, Overcommit

### 6-1) Resource Reservation

Resource Reservation은 Kubernetes 관점에서 Pod가 선언한 request만큼 Node 자원을 예약한 것으로 계산하는 개념이다. 이 예약은 실제 사용량과 다르다.

```text
Reservation = request 기준 예약량
Utilization = 실제 사용량
```

Pod가 CPU request 4core를 선언했지만 실제 사용량이 0.2core라면 reservation은 4core이고 utilization은 0.2core이다. Scheduler는 reservation을 기준으로 배치 판단을 하고, 운영자는 utilization을 보고 실제 사용 효율을 판단한다.

### 6-2) Resource Utilization

Resource Utilization은 실제로 사용 중인 CPU/Memory 양이다. `kubectl top`, Metrics Server, Prometheus, Grafana에서 보는 값이 여기에 해당한다. 운영에서 중요한 것은 reservation과 utilization의 차이다.

```text
request 대비 실제 사용량이 낮다 = over-provisioning 가능성 있음
request 대비 실제 사용량이 높다 = under-provisioning 또는 throttling/OOM 위험 가능성 있음
```

### 6-3) Resource Overcommit

Overcommit은 실제 물리 자원보다 더 많은 논리 리소스를 할당하는 전략이다. Kubernetes에서는 request를 낮게 잡고 limit을 높게 잡으면 Scheduler 기준 예약량을 줄이면서, 실제 부하가 있을 때는 더 많은 자원을 사용할 수 있게 만들 수 있다.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "2Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

CPU overcommit은 throttling 위험을 키우고, Memory overcommit은 OOMKilled, Node pressure, eviction 위험을 키운다. CPU는 시간 공유가 가능하기 때문에 어느 정도 overcommit이 가능하지만, Memory는 초과 사용 시 프로세스 종료나 eviction으로 이어질 수 있으므로 훨씬 조심해야 한다.

## 7. QoS Class

### 7-1) QoS Class란 무엇인가

Kubernetes는 Pod의 request/limit 설정에 따라 QoS Class를 부여한다. 대표적인 QoS Class는 Guaranteed, Burstable, BestEffort이다. QoS Class는 Node 자원이 부족할 때 어떤 Pod를 먼저 eviction할지 판단하는 데 영향을 준다.

### 7-2) Guaranteed

Guaranteed는 모든 컨테이너에 CPU/Memory request와 limit이 설정되어 있고, 각 resource의 request와 limit이 동일한 경우이다.

```yaml
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "2"
    memory: "4Gi"
```

Guaranteed Pod는 자원 부족 상황에서 상대적으로 보호받는다. 다만 request와 limit이 같기 때문에 overcommit 여지가 줄어들고 클러스터 효율은 낮아질 수 있다.

### 7-3) Burstable

Burstable은 request와 limit이 일부 설정되어 있지만 Guaranteed 조건을 만족하지 않는 경우이다.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "2Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

Burstable은 일반 애플리케이션에서 많이 사용된다. 기본 예약량은 낮추면서 필요할 때 더 많은 자원을 사용할 수 있다. 다만 Node 자원이 부족할 때 Guaranteed보다 eviction 위험이 높다.

### 7-4) BestEffort

BestEffort는 request와 limit이 없는 경우이다. 스케줄러 입장에서는 예약량이 거의 없으므로 배치가 쉬워 보일 수 있지만, Node 자원이 부족할 때 가장 먼저 eviction될 가능성이 높다. 운영 중요도가 있는 데이터 플랫폼 워크로드에는 일반적으로 적합하지 않다.

## 8. Node Capacity, Allocatable, Eviction

### 8-1) Node Capacity와 Allocatable

Node의 전체 CPU/Memory가 모두 Pod에 사용 가능한 것은 아니다. OS, kubelet, container runtime, system daemon이 사용하는 자원을 제외해야 한다. Kubernetes에서는 Pod에 할당 가능한 자원을 `Allocatable`로 표현한다.

```bash
kubectl describe node <node-name>
```

출력에서 `Capacity`와 `Allocatable`을 비교할 수 있다.

```text
Capacity:
  cpu: 64
  memory: 256Gi

Allocatable:
  cpu: 60
  memory: 240Gi
```

Scheduler는 Capacity가 아니라 Allocatable을 기준으로 request를 계산한다.

### 8-2) Eviction

Node에 MemoryPressure, DiskPressure, PIDPressure가 발생하면 kubelet은 일부 Pod를 eviction할 수 있다. request를 너무 낮게 잡고 limit만 크게 두는 overcommit 전략은 eviction 위험을 높일 수 있다. 데이터 플랫폼에서는 eviction이 쿼리 실패, 재처리, 캐시 손실, 서비스 지연으로 이어질 수 있다.

## 9. Capacity Planning

### 9-1) Capacity Planning이란 무엇인가

Capacity Planning은 현재와 미래의 워크로드를 기준으로 필요한 CPU, Memory, Storage, Network, Node 수를 계획하는 작업이다. Kubernetes에서는 단순 평균 사용률보다 request 기반 예약량을 함께 봐야 한다.

### 9-2) Kubernetes Capacity Planning에서 봐야 할 지표

```text
Node별 CPU/Memory 실제 사용률
Node별 CPU/Memory request 합계
Node별 CPU/Memory limit 합계
Namespace별 request 합계
Pod별 request 대비 실제 사용량
Pending Pod 수
Scheduling Failure 이벤트
Node allocatable 대비 request 비율
OOMKilled / eviction / throttling 이력
```

실제 사용률만 보면 아직 여유가 있다고 오해할 수 있다. 그러나 Scheduler 기준 request가 이미 가득 차 있으면 새 Pod는 뜨지 않는다.

### 9-3) Node 증설이 근본 해결이 아닌 경우

Pending Pod 문제가 생겼을 때 Node를 추가하면 단기적으로는 해결될 수 있다. 하지만 문제의 원인이 request 과다 설정이라면 Node 증설은 근본 해결책이 아니다.

```text
request 과다
↓
Pending 발생
↓
Node 증설
↓
일시적 해결
↓
사용자/Pod 증가
↓
다시 Pending 발생
```

이 구조를 끊으려면 request/limit right-sizing, 유휴 리소스 회수, Multi-Tenancy 구조 개선이 필요하다.

### 9-4) 근본적인 해결 방향

Node 증설은 단기적으로 Pending 문제를 완화할 수 있지만, 장기적으로는 다음 세 가지 관점의 개선이 필요하다.

#### 1) Request/Limit Right-Sizing

현재 설정된 Request와 실제 사용량 간의 차이를 분석하여 적정 수준으로 조정해야 한다.

주요 방법은 다음과 같다.

- Namespace 및 Pod 단위 Request 사용 현황 분석
- Goldilocks 및 VPA 기반 권장값 산정
- p95, p99, Peak 사용량 및 OOM 이력 기반 조정
- In-place Pod Resize를 통한 무중단 리소스 변경

Request/Limit 최적화 방법과 실제 운영 전략은 「02. Kubernetes Autoscaling & Optimization」 문서에서 자세히 다룬다.

#### 2) 유휴 리소스 회수

실제 사용하지 않는 워크로드가 예약 리소스를 계속 점유하는 문제를 해결해야 한다.

주요 방법은 다음과 같다.

- 최근 사용량 기반 Idle / Dormant 분류
- Goldilocks 및 VPA 기반 리소스 축소
- kubectl patch 및 In-place Resize 활용
- 필요 시 Scale Down 또는 Scale To Zero 적용

유휴 리소스 회수 전략은 「02. Kubernetes Autoscaling & Optimization」 문서에서 자세히 설명한다.

#### 3) Multi-Tenancy 구조 개선

장기적으로는 사용자별 애플리케이션을 개별 배포하는 구조보다 Shared Cluster 기반 구조가 효율적이다.

주요 방법은 다음과 같다.

- Shared Cluster 구축
- Tenant 기반 논리 분리
- Resource Group 기반 자원 제어
- Physical Isolation → Logical Isolation 전환

Multi-Tenancy 아키텍처와 Shared Cluster 설계는 「03. Multi-Tenancy & Data Platform Architecture」 및 「04. StarRocks / Trino / Milvus」 문서에서 자세히 다룬다.

## 10. 최종 정리

Kubernetes 리소스 관리는 단순히 CPU와 Memory 값을 적는 작업이 아니다. Request는 Scheduler의 예약 기준이고, Limit은 런타임 상한이다. ResourceQuota는 Namespace 전체 총량을 제한하고, LimitRange는 개별 Pod/Container의 기본값과 범위를 제한한다. Scheduler는 실제 사용량이 아니라 request를 기준으로 Node 배치 가능 여부를 판단한다.

대규모 환경에서 리소스 문제가 발생하면 다음 순서로 봐야 한다.

```text
1. Pod별 request/limit 확인
2. Namespace별 ResourceQuota/LimitRange 확인
3. Node별 Allocated resources 확인
4. 실제 usage와 request 차이 확인
5. Pending Pod의 Scheduler 이벤트 확인
6. QoS Class와 eviction 위험 확인
7. Overcommit 수준 확인
8. Capacity Planning 재검토
```
