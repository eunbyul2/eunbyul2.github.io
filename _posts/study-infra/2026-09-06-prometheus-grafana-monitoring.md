---
layout: post
title: "Prometheus와 Grafana 기반 모니터링 구조와 운영 방식"
date: 2026-09-06 18:30:00 +0900
categories: [Infrastructure, Monitoring, Kubernetes]
tags: [Prometheus, Grafana, Alertmanager, Monitoring, Kubernetes, Metrics, PromQL, kube-prometheus-stack]
published: true
---

# Prometheus와 Grafana 기반 모니터링 구조와 운영 방식

이번 인프라 스터디 주제는 Zabbix, Prometheus, Grafana와 같은 모니터링 도구의 설치 및 운영 방식과 알림·티켓 자동화, 애플리케이션 모니터링에 관한 내용이었다.

처음에는 Zabbix까지 함께 정리하려고 했지만 범위가 너무 넓어질 것 같아, 이번에는 Kubernetes와 Cloud Native 환경에서 자주 사용하는 **Prometheus와 Grafana**를 중심으로 정리해보았다.

Prometheus와 Grafana는 보통 한 세트처럼 이야기되지만 역할은 서로 다르다.

- **Prometheus**: Metric을 수집하고 저장하고 조회하는 시스템
- **Grafana**: Prometheus 등 여러 데이터 소스에 저장된 데이터를 조회하여 시각화하는 시스템
- **Alertmanager**: 발생한 Alert를 분류하고 묶고 적절한 수신처로 전달하는 시스템

가장 단순하게 표현하면 다음과 같다.

```text
Server / Kubernetes / Application
                |
                | Metrics
                v
           Prometheus
          /          \
         /            \
        v              v
    Grafana        Alertmanager
       |               |
       v               v
   Dashboard      Slack / Email
                  Webhook / Ticket
```

하지만 실제 운영 환경에서는 여기에서 `Exporter`, `Service Discovery`, `PromQL`, `Recording Rule`, `Alert Rule`, `ServiceMonitor`, `kube-state-metrics`, `node-exporter` 등 여러 구성 요소가 추가된다.

이 글에서는 단순히 설치 방법만 나열하기보다는 **왜 이런 구성 요소가 필요한지와 데이터가 실제로 어떤 흐름으로 이동하는지**를 중심으로 정리한다.

---

# 1. 먼저 Monitoring이 무엇인지

모니터링을 단순히 "Grafana에서 그래프를 보는 것"이라고 생각하면 Prometheus와 Grafana의 관계를 이해하기 어렵다.

운영 환경에서 모니터링은 보통 다음 과정을 포함한다.

```text
상태 데이터 생성
      |
      v
Metric 수집
      |
      v
저장
      |
      v
Query / 분석
      |
      +------------------+
      |                  |
      v                  v
Dashboard            Alert 조건 평가
                         |
                         v
                     Notification
                         |
                         v
                   장애 대응 / Ticket
```

예를 들어 Kubernetes Worker Node의 디스크가 차기 시작했다고 생각해보자.

단순히 현재 디스크 사용률만 보는 것이 아니라 다음과 같은 질문에 답할 수 있어야 한다.

- 현재 디스크 사용률은 몇 %인가?
- 1시간 전에는 몇 %였는가?
- 어느 시점부터 빠르게 증가했는가?
- 특정 Node에만 발생한 문제인가?
- `/var/lib/containerd`와 같은 특정 Filesystem에서 발생한 문제인가?
- 90%를 초과한 상태가 일정 시간 지속되고 있는가?
- 문제가 발생했을 때 운영자에게 알림이 전달되었는가?

이런 판단을 가능하게 하는 핵심 데이터가 **Metric**이다.

---

# 2. Metric이란?

Metric은 시스템이나 애플리케이션의 상태를 **숫자로 표현한 측정값**이다.

예를 들어 서버에서 다음과 같은 정보를 수집할 수 있다.

```text
CPU 사용량
Memory 사용량
Disk 사용량
Network 송수신량
Load Average
```

애플리케이션에서는 다음과 같은 Metric을 만들 수 있다.

```text
HTTP 요청 수
HTTP 5xx 오류 수
API 응답 시간
현재 DB Connection 수
Queue에 쌓여 있는 작업 수
```

Prometheus에서 Metric은 다음과 같은 형태로 볼 수 있다.

```text
node_memory_MemAvailable_bytes 8453754880
```

조금 더 복잡한 예는 다음과 같다.

```text
http_requests_total{method="GET",status="200"} 18372
```

이 값에는 크게 세 가지 요소가 있다.

```text
Metric Name
+
Label
+
Value
```

위 예시에서는:

```text
Metric Name
http_requests_total

Labels
method="GET"
status="200"

Value
18372
```

가 된다.

---

# 3. Time Series란?

Prometheus는 Metric의 현재 값 하나만 저장하는 것이 아니다.

예를 들어 CPU 사용률이 다음처럼 변했다고 하자.

```text
10:00  20%
10:01  32%
10:02  45%
10:03  68%
10:04  91%
```

이렇게 **시간에 따라 변화하는 값의 흐름**을 Time Series라고 한다.

Prometheus는 Metric을 Time Series 데이터로 저장한다.

따라서 운영자는 단순히

```text
현재 CPU 사용률 = 91%
```

만 보는 것이 아니라,

```text
CPU가 언제부터 증가했는가?
10분 전에는 몇 %였는가?
배포 직후부터 증가한 것인가?
```

같은 분석을 할 수 있다.

Prometheus에서는 일반적으로 하나의 Time Series가 다음 조합으로 식별된다.

```text
Metric Name + Label 조합
```

예를 들어:

```text
http_requests_total{method="GET",status="200"}
http_requests_total{method="GET",status="500"}
http_requests_total{method="POST",status="200"}
```

는 이름은 모두 `http_requests_total`이지만 Label 조합이 다르므로 서로 다른 Time Series다.

이 특징은 Prometheus의 강력한 장점이지만, 뒤에서 설명할 **Cardinality 문제**와도 직접 연결된다.

---

# 4. Prometheus란?

Prometheus는 오픈소스 시스템 모니터링 및 Alerting Toolkit이다.

Prometheus 공식 문서에서는 주요 특징을 다음과 같이 설명한다.

- Multi-dimensional Data Model
- Metric Name과 Key/Value Label 기반 Time Series
- PromQL이라는 Query Language
- HTTP 기반 Pull Model
- Static Configuration 및 Service Discovery
- 자체 Time Series Storage
- Alert Rule 지원

즉 Prometheus를 단순히 "데이터를 긁어오는 프로그램"이라고 보면 부족하다.

Prometheus Server가 담당하는 핵심 역할은 다음과 같다.

```text
1. Monitoring Target 발견
2. Target의 Metric Scrape
3. Time Series 저장
4. PromQL Query 처리
5. Recording Rule 평가
6. Alert Rule 평가
```

이를 구조로 표현하면 다음과 같다.

```text
                  Service Discovery
                         |
                         v
                   Target 목록 확인
                         |
                         v
Application -------> /metrics
Node Exporter -----> /metrics
kube-state-metrics -> /metrics
                         ^
                         |
                       Scrape
                         |
                    Prometheus
                  /      |      \
                 /       |       \
                v        v        v
              TSDB    PromQL    Rules
                         |        |
                         |        +------> Alertmanager
                         |
                         +---------------> Grafana
```

---

# 5. Prometheus의 Pull 방식

Prometheus의 중요한 특징 중 하나가 **Pull Model**이다.

많이 헷갈리는 부분인데, 기본적으로 Monitoring Target이 Prometheus에게 계속 데이터를 보내는 것이 아니다.

Prometheus가 Target에게 직접 접근한다.

```text
Prometheus
    |
    | HTTP GET
    v
http://target:port/metrics
```

이 동작을 **Scrape**라고 한다.

예를 들어 Linux 서버에 node-exporter가 동작하고 있다면 Prometheus는 다음 Endpoint를 일정 주기로 조회한다.

```text
http://10.0.0.10:9100/metrics
```

그러면 node-exporter는 다음과 같은 Metric을 반환한다.

```text
node_cpu_seconds_total
node_memory_MemAvailable_bytes
node_filesystem_size_bytes
node_network_receive_bytes_total
...
```

Prometheus는 이 값을 받아 TSDB에 저장한다.

따라서 기본적인 데이터 흐름은 다음과 같다.

```text
[Linux Server]
     |
     v
[node-exporter]
     ^
     |
     | HTTP Scrape
     |
[Prometheus]
```

화살표 방향을 이해하는 것이 중요하다.

Metric을 "가져오는 주체"는 Prometheus다.

---

# 6. Scrape Interval

Prometheus는 Target을 한 번만 조회하는 것이 아니라 일정 주기로 반복해서 Scrape한다.

예를 들어 Scrape Interval이 15초라면:

```text
10:00:00 scrape
10:00:15 scrape
10:00:30 scrape
10:00:45 scrape
10:01:00 scrape
```

와 같은 식으로 Metric Sample을 수집한다.

Scrape Interval을 짧게 하면 더 세밀한 데이터를 얻을 수 있지만 그만큼:

- 수집 요청 수 증가
- Prometheus CPU 사용량 증가
- Network 사용량 증가
- 저장되는 Sample 수 증가
- Disk 사용량 증가

등의 비용이 발생한다.

따라서 모든 Metric을 무조건 1초마다 수집하는 것이 좋은 것은 아니다.

---

# 7. Push 방식은 사용할 수 없는가?

Prometheus의 기본 철학은 Pull이지만 Push가 완전히 불가능한 것은 아니다.

짧게 실행되고 종료되는 Batch Job처럼 Prometheus가 적절한 시점에 Scrape하기 어려운 작업을 위해 **Pushgateway**를 사용할 수 있다.

개념적으로는 다음과 같다.

```text
Short-lived Job
      |
      | push
      v
Pushgateway
      ^
      |
      | scrape
      |
Prometheus
```

중요한 점은 Prometheus가 Pushgateway에서도 최종적으로는 **Scrape**한다는 것이다.

또한 Pushgateway를 일반 서버 Metric 전송용으로 무분별하게 사용하는 것은 권장되지 않는다. Job Lifecycle과 Metric Lifecycle이 달라 오래된 Metric이 남는 문제 등이 발생할 수 있기 때문이다.

---

# 8. Exporter란?

그렇다면 Linux 서버나 MySQL이 모두 Prometheus 형식의 `/metrics` Endpoint를 기본적으로 제공할까?

그렇지 않은 경우가 많다.

이때 사용하는 것이 **Exporter**다.

Exporter는 특정 시스템에서 정보를 가져와 Prometheus가 이해할 수 있는 Metric 형식으로 변환해서 제공한다.

```text
Monitoring Target
      |
      | 시스템 고유 방식
      v
    Exporter
      |
      | Prometheus Metric
      v
   /metrics
      ^
      |
   Prometheus
```

대표적인 예는 다음과 같다.

| 대상 | 대표 Exporter |
|---|---|
| Linux | node_exporter |
| MySQL | mysqld_exporter |
| PostgreSQL | postgres_exporter |
| Redis | redis_exporter |
| HTTP/TCP/DNS Probe | blackbox_exporter |

모든 애플리케이션에서 Exporter가 필요한 것은 아니다.

애플리케이션 자체에 Prometheus Client Library나 관련 Framework를 적용해서 직접 Metric Endpoint를 제공할 수도 있다.

예를 들어 Spring Boot에서는 Micrometer를 이용하여 Prometheus용 Metric을 노출할 수 있다.

```text
Application
    |
    v
/actuator/prometheus
    ^
    |
    | Scrape
    |
Prometheus
```

---

# 9. node-exporter는 무엇을 하는가?

Kubernetes 또는 Linux 서버를 모니터링할 때 가장 많이 보게 되는 Exporter가 `node-exporter`다.

node-exporter는 Linux OS 수준의 여러 정보를 Metric으로 제공한다.

대표적인 영역은 다음과 같다.

```text
CPU
Memory
Filesystem
Disk I/O
Network
Load
Kernel 관련 정보
```

예를 들어 다음과 같은 Metric을 볼 수 있다.

```text
node_cpu_seconds_total
node_memory_MemAvailable_bytes
node_filesystem_avail_bytes
node_network_receive_bytes_total
```

중요한 점은 `node-exporter`가 Kubernetes의 Pod 상태를 알려주는 프로그램은 아니라는 것이다.

node-exporter는 기본적으로 **Node의 OS/Hardware 관점 Metric**을 제공한다.

따라서 다음 질문:

```text
이 Node의 Memory가 얼마나 남았는가?
Disk 공간이 얼마나 남았는가?
Network Traffic이 얼마나 발생하는가?
```

에는 node-exporter가 적합하다.

반면:

```text
Deployment Replica가 몇 개인가?
Pod가 Pending 상태인가?
PVC가 Bound 상태인가?
```

는 Kubernetes Object 상태이므로 다른 구성 요소가 필요하다.

---

# 10. kube-state-metrics는 무엇인가?

Kubernetes에서는 `kube-state-metrics`라는 구성 요소를 자주 볼 수 있다.

이름 때문에 node-exporter와 헷갈릴 수 있지만 역할이 다르다.

kube-state-metrics는 Kubernetes API를 확인하고 Kubernetes Object의 상태를 Metric 형태로 노출한다.

```text
Kubernetes API Server
          |
          v
 kube-state-metrics
          ^
          |
          | Scrape
          |
      Prometheus
```

예를 들어 다음과 같은 정보를 Metric으로 만들 수 있다.

```text
Pod 상태
Deployment Replica
StatefulSet 상태
Node Condition
PVC 상태
Job 상태
```

대표적인 Metric 예시는 다음과 같다.

```text
kube_pod_status_phase
kube_deployment_status_replicas_available
kube_node_status_condition
kube_persistentvolumeclaim_status_phase
```

따라서 차이를 정리하면:

| 구성 요소 | 주로 보는 것 |
|---|---|
| node-exporter | Node OS의 CPU, Memory, Disk, Network 등 |
| kube-state-metrics | Kubernetes Object의 상태 |
| kubelet/cAdvisor 계열 Metric | Container Resource 사용량 |
| Application `/metrics` | 실제 서비스의 Request, Error, Latency 등 |

이 구분은 Kubernetes Monitoring을 이해할 때 상당히 중요하다.

---

# 11. Container Metric은 어디에서 오는가?

Kubernetes에서 Pod/Container CPU와 Memory를 모니터링하려면 Container Resource Metric이 필요하다.

예를 들어 다음과 같은 Metric을 볼 수 있다.

```text
container_cpu_usage_seconds_total
container_memory_working_set_bytes
```

Kubernetes에서는 kubelet이 Container Runtime과 Node 상태를 관리하며, Container Resource Metric은 kubelet이 노출하는 Metric 계열을 통해 수집되는 구조를 자주 사용한다.

개념적으로 보면 다음과 같다.

```text
Container
   |
   v
Container Runtime / cAdvisor 계열 정보
   |
   v
Kubelet Metrics
   ^
   |
   | Scrape
   |
Prometheus
```

따라서 Kubernetes에서 "CPU를 본다"고 해도 무엇을 보는지 정확히 구분해야 한다.

```text
Node 전체 CPU
→ node-exporter

Container CPU
→ kubelet/cAdvisor 계열 Metric

Deployment 상태
→ kube-state-metrics
```

---

# 12. Kubernetes Control Plane Metric

Prometheus는 Worker Node나 Pod만 모니터링하는 것이 아니다.

Kubernetes 자체 Control Plane 구성 요소들도 중요한 모니터링 대상이다.

예를 들어:

```text
kube-apiserver
etcd
kube-scheduler
kube-controller-manager
CoreDNS
```

등의 Metric을 수집할 수 있다.

운영 관점에서는 단순히 Node CPU만 정상이라고 클러스터가 정상이라고 판단할 수 없다.

예를 들어 API Server 요청 지연이 심해지거나 etcd 요청이 느려지면 Kubernetes 전체 제어 동작이 영향을 받을 수 있다.

따라서 실제 Kubernetes Monitoring 구조는 대략 다음과 같이 생각할 수 있다.

```text
Kubernetes Cluster

├── Node OS
│    └── node-exporter
│
├── Kubernetes Objects
│    └── kube-state-metrics
│
├── Containers
│    └── kubelet / cAdvisor metrics
│
├── Control Plane
│    ├── kube-apiserver metrics
│    ├── scheduler metrics
│    ├── controller-manager metrics
│    └── etcd metrics
│
└── Application
     └── application /metrics
             |
             v
         Prometheus
```

---

# 13. Application Metric이 중요한 이유

Infrastructure Metric이 정상이라고 서비스까지 정상인 것은 아니다.

예를 들어 다음 상태라고 하자.

```text
CPU       30%
Memory    40%
Disk      50%
Node      Ready
Pod       Running
```

Infrastructure 입장에서는 모두 정상처럼 보인다.

하지만 사용자는 다음 문제를 경험할 수 있다.

```text
로그인 API 응답에 8초가 걸림
결제 API에서 HTTP 500 발생
특정 API 요청이 Timeout
DB Connection Pool 고갈
```

이 문제들은 단순 Node CPU나 Pod Running 상태만으로는 알 수 없다.

그래서 실제 애플리케이션에서 다음과 같은 Metric을 노출한다.

```text
HTTP Request Count
HTTP Error Count
Request Duration
Active Request
DB Connection
Queue Length
Cache Hit Ratio
```

이 부분이 Infrastructure Monitoring과 Application Monitoring의 차이다.

---

# 14. RED Method

서비스나 API를 모니터링할 때 많이 사용하는 관점 중 하나가 RED Method다.

RED는 다음 세 가지를 의미한다.

```text
R = Rate
E = Errors
D = Duration
```

## Rate

얼마나 많은 요청을 처리하고 있는가?

예:

```text
초당 HTTP Request 수
초당 RPC Request 수
```

## Errors

요청 중 얼마나 많은 요청이 실패했는가?

예:

```text
HTTP 5xx 비율
RPC Error 비율
```

## Duration

요청 처리 시간이 얼마나 걸리는가?

예:

```text
평균 응답 시간
P50
P95
P99 Latency
```

예를 들어 서비스 장애를 분석할 때 다음처럼 볼 수 있다.

```text
Request Rate 증가
        |
        v
Latency 증가
        |
        v
HTTP 5xx 증가
```

이렇게 Metric 간의 관계를 함께 보면 단순히 "CPU가 높다"보다 서비스 장애 원인을 더 빠르게 좁힐 수 있다.

---

# 15. USE Method

Infrastructure Resource를 볼 때는 USE Method도 자주 사용한다.

```text
U = Utilization
S = Saturation
E = Errors
```

## Utilization

Resource가 얼마나 사용되고 있는가?

예:

```text
CPU Utilization
Memory Usage
Disk Utilization
```

## Saturation

Resource가 처리 가능한 양보다 일이 많이 들어와 대기하고 있는가?

예:

```text
CPU Run Queue
Disk I/O Queue
```

## Errors

Resource 관련 오류가 발생하고 있는가?

예:

```text
Disk I/O Error
Network Error
Hardware Error
```

따라서 모니터링에서 CPU 사용률 하나만 보는 것보다 Resource의 사용률, 대기, 오류를 함께 보는 것이 더 정확하다.

---

# 16. Service Discovery가 필요한 이유

Prometheus가 Target을 수집하려면 "어디를 Scrape해야 하는지" 알아야 한다.

서버가 몇 대뿐이고 IP가 고정되어 있다면 다음과 같이 Static Target을 직접 적을 수 있다.

```yaml
static_configs:
  - targets:
      - "10.0.0.11:9100"
      - "10.0.0.12:9100"
      - "10.0.0.13:9100"
```

하지만 Kubernetes에서는 이 방식이 현실적이지 않다.

Pod는 계속 생성되고 삭제된다.

```text
Pod A  10.244.1.10
Pod B  10.244.1.11
Pod C  10.244.2.20

Pod A 삭제
Pod D 생성 10.244.3.15
```

Prometheus 설정 파일에 Pod IP를 매번 직접 수정할 수 없다.

그래서 Prometheus는 **Service Discovery** 기능을 사용한다.

```text
Kubernetes API
      |
      | 현재 Pod / Service / Endpoint 정보
      v
Prometheus Service Discovery
      |
      v
Scrape Target 자동 생성
```

Prometheus 공식 문서에서도 Target은 Static Configuration뿐 아니라 Service Discovery를 통해 발견할 수 있다고 설명한다.

Kubernetes처럼 Resource가 계속 변하는 Dynamic 환경에서는 이 기능이 매우 중요하다.

---

# 17. Prometheus Operator는 왜 필요한가?

Kubernetes에서 Prometheus를 사용하다 보면 `Prometheus Operator`라는 이름이 등장한다.

Prometheus 자체와 Prometheus Operator는 다른 구성 요소다.

Prometheus를 Kubernetes에서 직접 운영한다면 다음과 같은 작업이 필요하다.

```text
Prometheus Pod 관리
Prometheus Configuration 관리
Scrape Target 관리
Alert Rule 관리
Alertmanager 연동
```

이런 구성을 Kubernetes Resource 방식으로 관리하기 쉽게 만들어주는 것이 Prometheus Operator다.

Operator를 이용하면 다음과 같은 Custom Resource를 사용할 수 있다.

```text
Prometheus
Alertmanager
ServiceMonitor
PodMonitor
PrometheusRule
```

즉 긴 `prometheus.yml`을 수동으로 계속 수정하는 대신 Kubernetes Resource를 선언하여 Monitoring 설정을 관리하는 방식이다.

---

# 18. ServiceMonitor란?

Prometheus Operator 환경에서 가장 자주 보게 되는 Resource 중 하나가 `ServiceMonitor`다.

예를 들어 애플리케이션이 다음 Endpoint를 제공한다고 하자.

```text
my-app Service
      |
      v
:8080/metrics
```

Prometheus가 이 Service를 자동으로 Scrape하도록 만들고 싶다.

이때 ServiceMonitor를 정의할 수 있다.

개념적으로 다음과 같다.

```text
Application Pod
      |
      v
Kubernetes Service
      ^
      |
ServiceMonitor
      |
      v
Prometheus Operator
      |
      v
Prometheus Scrape Configuration
```

간단한 예시는 다음과 비슷한 형태다.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

중요한 것은 `ServiceMonitor` 자체가 Metric을 수집하는 프로그램은 아니라는 점이다.

```text
ServiceMonitor
= 어떤 Service를 어떤 방식으로 Prometheus가 Scrape할지 선언하는 Kubernetes Resource
```

라고 이해하는 것이 정확하다.

---

# 19. PodMonitor

`PodMonitor`도 목적은 비슷하지만 Service가 아니라 Pod를 직접 대상으로 설정할 때 사용한다.

```text
ServiceMonitor
→ Service를 기반으로 Target 선택

PodMonitor
→ Pod를 기반으로 Target 선택
```

실제 환경에서는 애플리케이션 배포 구조에 따라 선택하여 사용한다.

---

# 20. Prometheus Data Model과 Label

Prometheus의 핵심 특징 중 하나가 Label 기반의 Multi-dimensional Data Model이다.

예:

```text
http_requests_total{
  method="GET",
  status="200",
  service="api",
  namespace="production"
}
```

Label을 이용하면 동일한 Metric을 여러 기준으로 나누어 조회할 수 있다.

예를 들어:

```text
전체 HTTP 요청 수
서비스별 HTTP 요청 수
Status Code별 HTTP 요청 수
Namespace별 HTTP 요청 수
```

를 같은 Metric에서 분석할 수 있다.

이것이 Prometheus의 강력한 점이다.

하지만 Label 설계를 잘못하면 큰 운영 문제가 발생한다.

---

# 21. Cardinality란?

Prometheus를 실제 운영할 때 반드시 알아야 하는 개념 중 하나가 **Cardinality**다.

예를 들어 다음 Metric이 있다고 하자.

```text
http_requests_total{
  method="GET",
  status="200"
}
```

`method` 값이 5개, `status` 값이 10개 정도라면 Time Series 수가 크게 늘어나지 않는다.

하지만 다음처럼 작성하면 문제가 된다.

```text
http_requests_total{
  user_id="123456789"
}
```

사용자가 100만 명이라면 매우 많은 Label 값이 만들어질 수 있다.

또 다음 Label은 특히 주의해야 한다.

```text
user_id
request_id
trace_id
session_id
random_uuid
```

값의 종류가 거의 무한하게 늘어날 수 있기 때문이다.

Prometheus에서는:

```text
Metric Name + Label Set
```

조합마다 별도의 Time Series가 만들어지므로 Label 조합이 폭발하면:

```text
Time Series 증가
      |
      +--> Memory 사용량 증가
      |
      +--> Disk 사용량 증가
      |
      +--> Query 비용 증가
      |
      +--> Prometheus 성능 저하
```

로 이어질 수 있다.

따라서 Prometheus 운영에서는 단순히 Metric 개수뿐 아니라 **Label Cardinality 관리**가 매우 중요하다.

---

# 22. Prometheus의 Metric Type

Prometheus Client Library 관점에서 자주 사용하는 Metric Type에는 다음이 있다.

```text
Counter
Gauge
Histogram
Summary
```

이 차이를 이해하면 PromQL을 이해하기 쉬워진다.

---

## 22.1 Counter

Counter는 일반적으로 **계속 증가하는 누적 값**이다.

예:

```text
http_requests_total
process_cpu_seconds_total
```

Request가 발생할 때마다:

```text
100
101
102
103
104
```

처럼 증가한다.

프로세스가 재시작되면 0으로 Reset될 수 있다.

Counter에서 단순 현재 값보다 "얼마나 빠르게 증가하는지"가 중요하기 때문에 `rate()`를 자주 사용한다.

예:

```promql
rate(http_requests_total[5m])
```

의 의미는 대략:

```text
최근 5분 동안 HTTP Request Counter가
초당 어느 정도 속도로 증가했는가?
```

라고 이해할 수 있다.

---

## 22.2 Gauge

Gauge는 값이 증가하거나 감소할 수 있다.

예:

```text
현재 Memory 사용량
현재 Queue Length
현재 Temperature
현재 Connection 수
```

값이:

```text
30
50
40
70
20
```

처럼 자유롭게 변할 수 있다.

---

## 22.3 Histogram

Histogram은 요청 시간이나 응답 크기와 같은 값을 여러 Bucket으로 나누어 관찰할 때 사용한다.

예를 들어 HTTP 요청 시간:

```text
0.1초 이하
0.5초 이하
1초 이하
5초 이하
```

처럼 구간을 나누어 누적할 수 있다.

이를 활용해 P95, P99 같은 Percentile Latency를 계산할 수 있다.

Prometheus에서는 `histogram_quantile()` 같은 함수를 이용해 Histogram 기반 Quantile을 계산하는 경우가 많다.

---

## 22.4 Summary

Summary도 요청 시간 등 관찰값을 다루는 Metric Type이다.

Histogram과 사용 목적이 비슷해 보이지만 Quantile 계산 위치와 Aggregation 특성이 다르다.

이번 정리에서는 내부 차이까지 깊게 들어가기보다 다음 정도로 기억하면 충분하다.

```text
Counter
→ 누적 증가 값

Gauge
→ 현재 상태처럼 증가/감소 가능한 값

Histogram
→ Bucket 기반 분포 측정

Summary
→ Observation과 Quantile 중심 측정
```

---

# 23. PromQL이란?

Prometheus에 Metric이 쌓여 있어도 원하는 데이터를 조회할 수 없다면 활용하기 어렵다.

Prometheus에서는 **PromQL(Prometheus Query Language)**을 사용한다.

PromQL은 Time Series를 선택하고 필터링하고 계산하고 집계할 수 있는 Query Language다.

---

# 24. 가장 기본적인 PromQL

가장 단순한 예:

```promql
up
```

Prometheus가 Target Scrape에 성공했는지 확인할 때 많이 사용하는 Metric이다.

보통:

```text
up = 1
→ Scrape 성공

up = 0
→ Scrape 실패
```

로 이해한다.

특정 Job만 선택하고 싶다면 Label Selector를 사용할 수 있다.

```promql
up{job="node-exporter"}
```

특정 Instance:

```promql
up{instance="10.0.0.10:9100"}
```

---

# 25. rate()

Counter에서 매우 자주 사용하는 함수다.

```promql
rate(http_requests_total[5m])
```

Counter의 현재 값이 100만이라고 해도 그것만으로 현재 Traffic이 높은지 알기 어렵다.

`rate()`를 사용하면 최근 일정 구간에서 Counter가 얼마나 빠르게 증가했는지 계산할 수 있다.

예:

```text
http_requests_total

10:00  1000
10:01  1060
10:02  1120
```

라면 일정 시간 동안 요청 수가 어떤 속도로 증가하는지 계산하는 데 사용할 수 있다.

---

# 26. Aggregation

PromQL에서는 여러 Time Series를 집계할 수 있다.

예:

```promql
sum(rate(http_requests_total[5m]))
```

서비스별로 묶고 싶다면:

```promql
sum by (service) (
  rate(http_requests_total[5m])
)
```

Status Code별:

```promql
sum by (status) (
  rate(http_requests_total[5m])
)
```

Prometheus가 Kubernetes와 Microservice 환경에서 강한 이유 중 하나가 바로 이런 Label 기반 조회와 집계가 유연하기 때문이다.

---

# 27. Recording Rule

PromQL Query가 매우 복잡하고 자주 실행된다면 매번 같은 계산을 반복하는 것이 비효율적일 수 있다.

이럴 때 Recording Rule을 사용한다.

개념적으로:

```text
복잡한 PromQL
     |
     | 미리 계산
     v
새로운 Time Series로 저장
```

하는 방식이다.

예를 들어 여러 Metric을 조합한 복잡한 Query를 Dashboard에서 수십 번 반복한다면 Recording Rule을 이용해 계산 결과를 미리 만들어 둘 수 있다.

---

# 28. Alert Rule

Prometheus는 Metric을 수집하는 것뿐 아니라 Query 결과를 기반으로 Alert 조건도 평가할 수 있다.

예를 들어:

```text
Node Disk 사용률이 90% 이상
그리고
그 상태가 10분 이상 지속
```

이라는 조건을 Alert Rule로 정의할 수 있다.

개념적인 구조는 다음과 같다.

```text
Prometheus Metric
      |
      v
PromQL Expression
      |
      v
Alert Rule
      |
      | 조건 만족
      v
Pending
      |
      | 일정 시간 지속
      v
Firing
```

단순 순간 Spike에 바로 Alert를 발생시키지 않기 위해 일정 시간 지속 조건을 사용하는 것이 중요하다.

---

# 29. Alertmanager는 왜 필요한가?

Prometheus가 "문제가 발생했다"고 판단하는 것과 운영자에게 알림을 보내는 것은 다른 문제다.

Prometheus:

```text
CPU가 임계치를 넘었다.
Disk가 90%를 넘었다.
Pod가 Down 상태다.
```

를 판단한다.

Alertmanager:

```text
이 Alert를 어느 팀에 보낼 것인가?
같은 Alert 여러 개를 묶을 것인가?
점검 시간에는 알림을 중단할 것인가?
상위 장애가 있으면 하위 Alert를 억제할 것인가?
```

를 담당한다.

전체 구조:

```text
Prometheus
    |
    | Alert
    v
Alertmanager
    |
    +--> Slack
    +--> Email
    +--> PagerDuty
    +--> Webhook
    +--> Incident / Ticket System
```

Prometheus 공식 문서에서 Alertmanager는 Alert의 Deduplication, Grouping, Routing, Silence, Inhibition 등의 기능을 담당한다.

---

# 30. Grouping

하나의 장애가 발생하면 관련 Alert가 여러 개 발생할 수 있다.

예를 들어 Node 하나가 Down되었다.

그러면:

```text
NodeDown
node-exporter Down
PodDown
ApplicationDown
```

같은 Alert가 동시에 발생할 수 있다.

이 Alert를 각각 따로 20개 보내면 운영자는 중요한 장애를 파악하기 어려워진다.

Alertmanager는 관련 Alert를 묶어 Notification 수를 줄일 수 있다.

이것을 **Grouping**이라고 한다.

---

# 31. Routing

모든 Alert를 같은 곳에 보낼 필요는 없다.

예:

```text
severity=critical
→ PagerDuty / 긴급 채널

severity=warning
→ Slack

team=database
→ DB 운영팀

team=kubernetes
→ Platform 운영팀
```

Alert Label을 기준으로 알림 대상과 전달 방식을 결정할 수 있다.

이것이 Routing이다.

---

# 32. Silence

정기 점검 시간에 서버를 재부팅하면 의도된 장애인데도 Alert가 발생할 수 있다.

예:

```text
Kubernetes Upgrade
Network 작업
Node 재부팅
Storage 점검
```

이런 Maintenance Window에 특정 Alert Notification을 일시적으로 중단하는 것이 Silence다.

중요한 점은 Alert Rule 평가 자체를 삭제하는 개념과는 다르다는 것이다.

---

# 33. Inhibition

상위 장애가 이미 발생한 경우 그 결과로 발생한 하위 Alert를 억제할 수 있다.

예:

```text
Node Down
  |
  +--> Pod Down
  +--> Exporter Down
  +--> Application Down
```

Node 자체가 Down된 것이 Root Cause라면 관련된 하위 Alert를 모두 운영자에게 보낼 필요가 없을 수 있다.

이런 의존 관계에 따라 특정 Alert Notification을 억제하는 것을 Inhibition이라고 한다.

---

# 34. Grafana는 무엇인가?

Prometheus를 공부하면 자연스럽게 Grafana가 같이 나온다.

하지만 둘은 역할이 다르다.

```text
Prometheus
= Metric 수집 + 저장 + Query + Rule 평가

Grafana
= Data Source Query + Dashboard + Visualization + Alerting 기능
```

Grafana를 단순하게 말하면 여러 Data Source에 저장된 데이터를 조회해서 사람이 보기 좋은 Dashboard로 보여주는 도구다.

```text
Prometheus
    ^
    |
    | PromQL Query
    |
Grafana
    |
    v
Dashboard
```

Grafana가 Prometheus Metric 원본을 자기 내부에 복사하여 저장한 뒤 보여주는 구조로 이해하면 안 된다.

Grafana 자체는 사용자, Dashboard 설정 등의 메타데이터를 저장할 수 있지만, **Prometheus Time Series 원본의 주 저장소 역할은 Prometheus**다.

---

# 35. Grafana Data Source

Grafana가 어느 시스템에서 데이터를 조회할지 정의한 것이 Data Source다.

Prometheus를 Data Source로 등록하면 Grafana가 Prometheus API를 이용해 PromQL Query를 실행할 수 있다.

Grafana는 Prometheus만 지원하는 것이 아니다.

예를 들어 다음과 같은 여러 Data Source를 사용할 수 있다.

```text
Prometheus
Loki
Elasticsearch
MySQL
PostgreSQL
InfluxDB
Cloud Monitoring 계열
```

따라서 하나의 Grafana에서:

```text
Metric
Log
Database Data
```

등을 다양한 Data Source에서 가져와 시각화할 수 있다.

---

# 36. Dashboard와 Panel

Grafana Dashboard는 여러 Panel의 집합이다.

예를 들어 Kubernetes Cluster Dashboard를 만든다면 다음과 같이 구성할 수 있다.

```text
Kubernetes Cluster Dashboard

+---------------------------------------+
| Node CPU Usage                        |
+---------------------------------------+

+---------------------------------------+
| Node Memory Usage                     |
+---------------------------------------+

+-------------------+-------------------+
| Running Pods      | Pending Pods      |
+-------------------+-------------------+

+---------------------------------------+
| API Server Request Latency            |
+---------------------------------------+

+---------------------------------------+
| Network Receive / Transmit            |
+---------------------------------------+
```

각 Panel에는 Prometheus Data Source를 대상으로 하는 PromQL Query가 들어갈 수 있다.

즉:

```text
Grafana Panel
      |
      v
PromQL Query
      |
      v
Prometheus
      |
      v
Query Result
      |
      v
Graph / Table / Gauge
```

와 같은 구조다.

---

# 37. Variable

운영 Dashboard를 사용하다 보면 Cluster, Namespace, Node, Pod를 매번 다른 Dashboard로 만들 수는 없다.

Grafana의 Variable을 사용하면 하나의 Dashboard에서 선택 값을 바꾸어 여러 대상을 볼 수 있다.

예:

```text
Cluster:   prod-cluster
Namespace: monitoring
Node:      worker-01
```

Dropdown을 변경하면 Panel Query에서 해당 값을 참조하도록 구성할 수 있다.

이 기능을 이용하면 운영 Dashboard를 훨씬 재사용하기 쉽다.

---

# 38. Grafana Alerting

Grafana 자체에도 Alerting 기능이 있다.

따라서 현재 환경에서는 크게 두 가지 Alerting 흐름을 볼 수 있다.

### Prometheus 중심

```text
Prometheus Alert Rule
        |
        v
Alertmanager
        |
        v
Notification
```

### Grafana Managed Alerting

```text
Grafana
   |
   | Data Source Query
   v
Prometheus / Loki / 기타 Data Source
   |
   v
Grafana Alert Rule 평가
   |
   v
Grafana Alertmanager
   |
   v
Contact Point / Notification Policy
```

Grafana 공식 문서 기준으로 Grafana Alerting은 여러 Data Source에 Query를 실행하여 Alert Rule을 평가할 수 있으며, Notification Policy와 Contact Point를 통해 알림을 Routing할 수 있다.

---

# 39. Grafana Contact Point

Contact Point는 Alert Notification을 실제로 어디로 보낼지를 정의한다.

예:

```text
Slack
Email
Webhook
PagerDuty
Jira
Microsoft Teams
```

등을 사용할 수 있다.

따라서 Alert 처리 흐름은 다음과 같이 만들 수 있다.

```text
Alert Rule
    |
    v
Notification Policy
    |
    v
Contact Point
    |
    +--> Slack
    +--> Email
    +--> Jira
    +--> Webhook
```

---

# 40. 티켓 자동화는 어떻게 연결되는가?

이번 스터디 주제에 티켓 자동화 이야기도 있었기 때문에 이 부분도 함께 보면 좋다.

예를 들어 Kubernetes Node의 Disk 사용률이 90% 이상으로 10분간 유지되었다고 하자.

```text
Disk Metric
    |
    v
Prometheus
    |
    v
Alert Rule
    |
    v
Alertmanager / Grafana Alerting
    |
    v
Webhook / Jira Integration
    |
    v
Incident Ticket 생성
```

생성되는 Ticket에는 다음 정보를 포함할 수 있다.

```text
Title
[Critical] worker-01 Disk Usage High

Cluster
production

Node
worker-01

Filesystem
/var/lib/containerd

Current Usage
94%

Severity
critical

Started At
2026-09-06 14:30

Runbook
https://example.com/runbook/disk-full
```

여기에서 중요한 것은 **Metric을 많이 모으는 것 자체가 목적이 아니라 실제 장애 대응 과정과 연결하는 것**이다.

---

# 41. Alert Fatigue

모니터링 시스템을 운영하면 "Alert가 많으면 안전하다"고 생각하기 쉽다.

실제로는 지나치게 많은 Alert가 더 큰 문제를 만들 수 있다.

예를 들어 하나의 Node 장애로:

```text
NodeDown
KubeletDown
ExporterDown
10개의 PodDown
5개의 ApplicationDown
```

등 수십 개의 Alert가 동시에 발생하면 운영자는 무엇이 Root Cause인지 찾기 어려워진다.

이런 상태가 반복되면 Alert 자체를 무시하게 될 수 있다.

이를 **Alert Fatigue**라고 한다.

따라서 운영에서는 다음이 중요하다.

```text
중요한 Alert만 정의
Grouping 사용
Routing 구분
Inhibition 구성
Maintenance Silence
적절한 Threshold 및 지속 시간 설정
```

---

# 42. False Positive와 Flapping

## False Positive

실제로 장애가 아닌데 Alert가 발생하는 경우다.

예를 들어 다음 조건:

```text
CPU > 90%
```

만 설정하면 배치 작업 때문에 CPU가 10초간 95%가 된 상황에도 Alert가 발생할 수 있다.

그래서 상황에 따라:

```text
CPU > 90%가 5분 이상 지속
```

처럼 지속 시간 조건을 함께 둔다.

## Flapping

상태가 임계값 근처에서 계속 움직이면:

```text
Normal
Critical
Normal
Critical
Normal
Critical
```

상태가 반복될 수 있다.

그 결과 Notification이 지나치게 많이 발생한다.

Threshold, 지속 시간, Alert 설계를 적절하게 조정해야 하는 이유다.

---

# 43. `up=0`이 항상 서비스 장애를 의미하는가?

Prometheus에서 자주 보는 Metric이 `up`이다.

하지만:

```text
up == 0
```

의 의미를 정확히 이해해야 한다.

이 값은 기본적으로 Prometheus가 해당 Target을 **성공적으로 Scrape하지 못했다**는 의미다.

원인은 여러 가지일 수 있다.

```text
Target Process Down
Network 장애
DNS 문제
Firewall
TLS 인증 문제
Authentication 문제
/metrics Endpoint 오류
Timeout
```

따라서 `up=0`을 바로 "애플리케이션 자체가 죽었다"고 단정하면 안 된다.

Scrape Path 전체를 확인해야 한다.

---

# 44. Monitoring the Monitoring

모니터링 시스템 자체가 장애가 나는 상황도 고려해야 한다.

예를 들어 Prometheus가 Down되면:

```text
Application 장애 발생
        +
Prometheus 장애
```

상태일 수 있는데 Metric 수집 자체가 중단되므로 Application 장애를 감지하지 못할 수 있다.

그래서 운영 환경에서는 다음과 같은 것도 고려해야 한다.

```text
Prometheus 자체 상태
Alertmanager 상태
Grafana 상태
Scrape 실패율
Disk 사용량
TSDB 상태
Rule Evaluation 실패
Notification 전달 실패
```

모니터링 시스템도 결국 하나의 운영 시스템이기 때문에 자체 모니터링이 필요하다.

---

# 45. Retention

Metric은 계속 쌓인다.

예를 들어:

```text
15초마다 수집
Target 수 증가
Metric 수 증가
Label 조합 증가
```

가 계속되면 저장량도 증가한다.

따라서 어느 기간까지 Local Metric을 보관할지 결정해야 한다.

이를 Retention이라고 한다.

```text
최근 15일
최근 30일
최근 90일
```

등 운영 요구사항에 따라 정책을 결정할 수 있다.

장기간 Metric 보존이 필요하거나 여러 Prometheus 데이터를 중앙에서 조회해야 하는 환경에서는 별도 장기 저장/확장 솔루션을 함께 사용하는 경우가 있다.

대표적으로 다음 프로젝트들이 있다.

```text
Thanos
Grafana Mimir
VictoriaMetrics
```

이번 글에서는 각각의 내부 구조까지 다루지는 않는다.

---

# 46. Prometheus HA를 생각할 때

Prometheus는 단일 서버만으로도 독립적으로 동작하는 것을 중요하게 설계한 시스템이다.

하지만 운영 환경에서 Prometheus 한 대만 두면 Prometheus 자체 장애가 Monitoring Blind Spot으로 이어질 수 있다.

그래서 HA 구성을 고려할 수 있다.

다만 Prometheus HA는 단순히 두 Pod를 띄우면 모든 문제가 해결되는 개념은 아니다.

다음과 같은 부분을 함께 고려해야 한다.

```text
Prometheus Replica
Alert 중복 처리
Alertmanager HA
Storage
Remote Write
Query 통합
```

환경 규모에 따라 Thanos나 Mimir와 같은 별도 구성까지 고려할 수 있다.

이번 단계에서는 **Prometheus 자체도 SPOF가 될 수 있다**는 점을 이해하는 것이 우선이다.

---

# 47. kube-prometheus-stack

Kubernetes에서 Prometheus와 Grafana를 설치할 때 `kube-prometheus-stack` Helm Chart를 자주 보게 된다.

Prometheus Community의 공식 Chart 설명에 따르면 kube-prometheus-stack은 Kubernetes에서 Prometheus Operator를 기반으로 End-to-End Monitoring을 구성할 수 있도록 Kubernetes Manifest, Grafana Dashboard, Prometheus Rule 등을 함께 제공한다.

현재 Chart에는 기본적으로 다음과 같은 주요 구성 요소가 포함된다.

```text
Prometheus Operator
Prometheus
Alertmanager
Grafana
kube-state-metrics
prometheus-node-exporter
Prometheus Rules
Grafana Dashboards
```

단순하게 관계를 표현하면:

```text
                kube-prometheus-stack

    +---------------------------------------+
    | Prometheus Operator                   |
    |                                       |
    | Prometheus                            |
    | Alertmanager                          |
    | Grafana                               |
    | kube-state-metrics                    |
    | node-exporter                         |
    | ServiceMonitor / Rules / Dashboard    |
    +---------------------------------------+
```

따라서 Helm Chart 하나를 설치했다고 해서 하나의 프로그램만 설치되는 것이 아니다.

---

# 48. kube-prometheus-stack 구성 요소를 각각 보면

## Prometheus Operator

```text
Monitoring 관련 Kubernetes CR 관리
Prometheus/Alertmanager 구성 자동화
ServiceMonitor/PodMonitor 처리
```

## Prometheus

```text
Metric Scrape
TSDB 저장
PromQL
Rule 평가
```

## Alertmanager

```text
Alert Grouping
Routing
Silence
Inhibition
Notification
```

## Grafana

```text
Dashboard
Visualization
Data Source Query
Grafana Alerting
```

## node-exporter

```text
Node OS Metric
```

## kube-state-metrics

```text
Kubernetes Object State Metric
```

이 역할을 서로 섞지 않고 설명할 수 있어야 kube-prometheus-stack 구조를 이해했다고 볼 수 있다.

---

# 49. 전체 Kubernetes Monitoring 흐름

지금까지 내용을 하나로 합치면 다음과 같다.

```text
                         Kubernetes Cluster

      +-----------------------+------------------------+
      |                       |                        |
      v                       v                        v
 node-exporter        kube-state-metrics         Application
      |                       |                    /metrics
      |                       |                        |
      +-----------------------+------------------------+
                              |
                              | Scrape
                              v
                         Prometheus
                        /     |      \
                       /      |       \
                      v       v        v
                    TSDB    PromQL   Alert Rule
                              |         |
                              |         v
                              |    Alertmanager
                              |         |
                              v         v
                           Grafana   Notification
                              |         |
                              v         +--> Slack
                          Dashboard     +--> Email
                                        +--> Webhook
                                        +--> Ticket
```

Control Plane, kubelet 등의 Metric을 포함하면 실제 구조는 더 복잡하지만 기본 흐름은 이와 같다.

---

# 50. 설치해서 직접 확인한다면

단순 이론만 보는 것보다 실제 Cluster에서 kube-prometheus-stack을 설치한 뒤 구성 요소를 확인하면 이해가 빨라진다.

공식 Prometheus Community Helm Repository를 사용하는 방법의 예는 다음과 같다.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Chart 검색:

```bash
helm search repo prometheus-community/kube-prometheus-stack
```

설치 예:

```bash
helm install monitoring \
  prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace
```

> 실제 운영 환경에서는 바로 기본값으로 설치하기보다 Chart Version을 고정하고 `values.yaml`을 검토한 뒤 배포하는 것이 좋다.

설치 후:

```bash
kubectl get pods -n monitoring
```

을 보면 환경과 Chart 설정에 따라 다음과 비슷한 구성 요소를 볼 수 있다.

```text
alertmanager
grafana
kube-state-metrics
prometheus
prometheus-node-exporter
prometheus-operator
```

여기서 중요한 것은 Pod가 Running인지 보는 것으로 끝내지 않고:

```text
이 Pod가 어떤 역할인가?
어떤 Metric을 제공하는가?
누가 누구를 Scrape하는가?
```

를 연결해서 확인하는 것이다.

---

# 51. 설치 후 공부해볼 항목

## 1. Prometheus Target 확인

Prometheus UI의 Targets에서 어떤 Endpoint를 Scrape하고 있는지 확인한다.

확인할 부분:

```text
Target
State UP/DOWN
Labels
Last Scrape
Scrape Duration
Error
```

Target이 Down이면:

```text
DNS
Service
Endpoint
NetworkPolicy
Port
Path
TLS
Authentication
```

등을 확인한다.

---

## 2. `up` Metric 확인

```promql
up
```

어떤 Target이 정상적으로 Scrape되고 있는지 확인한다.

---

## 3. node-exporter Metric 확인

예:

```promql
node_memory_MemAvailable_bytes
```

Node별 Metric이 실제로 존재하는지 확인한다.

---

## 4. kube-state-metrics 확인

예:

```promql
kube_pod_status_phase
```

Pod의 Kubernetes 상태 정보가 Metric으로 만들어지고 있는지 확인한다.

---

## 5. Container Metric 확인

예:

```promql
container_cpu_usage_seconds_total
```

Container 관련 Metric이 어디에서 수집되고 있는지 확인한다.

---

## 6. Grafana Data Source 확인

Grafana에서 Prometheus가 Data Source로 등록되어 있는지 확인한다.

그다음 Dashboard의 Panel이 어떤 PromQL Query를 사용하는지 직접 열어보는 것이 좋다.

완성된 Dashboard만 보는 것보다 Query를 확인하는 것이 훨씬 공부가 된다.

---

# 52. 운영 장애 시 확인 순서

Prometheus/Grafana Dashboard에서 데이터가 안 보이는 상황을 생각해보자.

무조건 Grafana 문제라고 판단하면 안 된다.

데이터 흐름을 역방향으로 확인하면 된다.

```text
Application / Exporter
        |
        v
Prometheus Target
        |
        v
Prometheus TSDB
        |
        v
PromQL
        |
        v
Grafana Data Source
        |
        v
Panel
```

예를 들어 Grafana Panel에 `No Data`가 나온다면 다음 순서로 볼 수 있다.

### 1. 원본 Metric Endpoint 확인

```bash
curl http://target:port/metrics
```

Metric 자체가 노출되는지 확인한다.

### 2. Prometheus Target 확인

Target이 `UP`인지 확인한다.

### 3. Prometheus에서 Metric 직접 Query

Grafana를 거치지 않고 Prometheus UI에서 해당 Metric을 직접 조회한다.

### 4. Grafana Data Source 확인

Grafana에서 Prometheus 연결이 정상인지 확인한다.

### 5. Panel Query 확인

PromQL과 Label Filter가 잘못되지 않았는지 확인한다.

이런 식으로 **Metric 발생 지점부터 시각화 지점까지 데이터 경로를 따라가며 확인하는 것**이 중요하다.

---

# 53. 실제 운영에서 자주 생각해야 하는 문제

Prometheus/Grafana 설치 자체보다 운영 과정이 더 중요하다.

최소한 다음 항목은 고려해야 한다.

```text
1. 무엇을 Metric으로 수집할 것인가?
2. Scrape Interval은 어떻게 정할 것인가?
3. Retention을 얼마로 할 것인가?
4. Label Cardinality를 어떻게 관리할 것인가?
5. Alert Threshold를 어떻게 정할 것인가?
6. Alert Fatigue를 어떻게 줄일 것인가?
7. Prometheus 자체 장애는 어떻게 감지할 것인가?
8. Metric 장기 보관이 필요한가?
9. Dashboard를 팀/서비스별로 어떻게 관리할 것인가?
10. Ticket/Incident 시스템과 어떻게 연동할 것인가?
```

이런 부분이 실제 운영에서 단순 설치보다 더 많은 고민이 필요한 지점이다.

---

# 54. Prometheus와 Grafana의 차이를 다시 정리

가장 헷갈리지 않게 정리하면 다음과 같다.

| 구분 | Prometheus | Grafana |
|---|---|---|
| 핵심 역할 | Metric 수집/저장/조회 | 데이터 시각화 |
| 데이터 저장 | 자체 TSDB에 Metric 저장 | 원본 Metric 저장소 역할이 아님 |
| Query | PromQL 제공 | Data Source의 Query 사용 |
| 수집 | Target을 Scrape | 직접 Metric Scrape가 핵심 역할은 아님 |
| Dashboard | 기본 UI는 있으나 제한적 | Dashboard 기능이 강점 |
| Alert | Alert Rule 평가 가능 | Grafana Alerting 제공 |
| Kubernetes 사용 | Monitoring Backend로 많이 사용 | Visualization/Alert UI로 많이 사용 |

따라서 둘을 경쟁 제품으로 보면 안 된다.

오히려:

```text
Prometheus가 모은 데이터를
Grafana가 보기 좋게 보여준다.
```

라고 이해하는 것이 가장 쉽다.

---

# 55. 전체 데이터 흐름을 말로 설명해보기

이번 내용을 공부한 뒤 아래 문장을 이해하고 설명할 수 있으면 기본 구조는 잡힌 것이다.

> Kubernetes Node의 OS Metric은 node-exporter가 Prometheus 형식으로 노출하고, Kubernetes Object의 상태는 kube-state-metrics가 Kubernetes API에서 정보를 가져와 Metric으로 노출한다. Container와 Control Plane 구성 요소도 각각 Metric Endpoint를 제공한다. Prometheus는 Static Configuration이나 Service Discovery, Prometheus Operator 환경에서는 ServiceMonitor/PodMonitor 등의 설정을 바탕으로 Target을 발견하고 일정 주기로 HTTP Scrape를 수행한다. 수집한 Sample은 Time Series 형태로 TSDB에 저장되고 PromQL을 이용해 조회할 수 있다. Grafana는 Prometheus를 Data Source로 등록하고 PromQL Query 결과를 Panel과 Dashboard로 시각화한다. 장애 조건은 Prometheus Alert Rule이나 Grafana Alerting으로 평가할 수 있으며, Prometheus 기반 Alert는 Alertmanager에서 Grouping, Routing, Silence, Inhibition 등을 처리한 뒤 Slack, Email, Webhook, Ticket 시스템 등으로 전달할 수 있다.

---

# 56. 정리

이번에 Prometheus와 Grafana를 정리하면서 가장 먼저 구분해야 하는 것은 각 도구의 역할이다.

```text
Metric을 만든다
      |
      v
Exporter / Application
      |
      v
Prometheus가 Scrape
      |
      v
Prometheus TSDB 저장
      |
      +--------------------+
      |                    |
      v                    v
   Grafana              Alert Rule
      |                    |
      v                    v
 Dashboard             Alertmanager
                           |
                           v
                 Slack / Email / Ticket
```

특히 Kubernetes에서는 Metric 출처를 구분하는 것이 중요하다.

```text
Node OS
→ node-exporter

Kubernetes Object
→ kube-state-metrics

Container
→ kubelet/cAdvisor 계열 Metrics

Application
→ Application 자체 Metrics

수집/저장
→ Prometheus

시각화
→ Grafana

Alert 전달 관리
→ Alertmanager
```

그리고 실제 운영에서는 단순히 모든 Metric을 최대한 많이 모으는 것보다:

```text
어떤 Metric이 실제 장애 판단에 필요한가?
어떤 Label이 필요한가?
Cardinality가 너무 높지는 않은가?
Alert가 실제 대응 가능한 수준으로 설계되어 있는가?
Metric이 장기적으로 얼마나 저장되어야 하는가?
모니터링 시스템 자체 장애를 어떻게 감지할 것인가?
```

를 함께 고려해야 한다.

결국 Prometheus와 Grafana는 단순히 예쁜 Dashboard를 만들기 위한 도구가 아니라, **시스템 상태를 수치화하고 장애를 빠르게 발견하고 원인을 좁히며 운영 대응까지 연결하기 위한 Monitoring Infrastructure**라고 이해할 수 있다.

---

# 참고 자료

- Prometheus 공식 Overview  
  <https://prometheus.io/docs/introduction/overview/>

- Prometheus PromQL 기본 문서  
  <https://prometheus.io/docs/prometheus/latest/querying/basics/>

- Prometheus Alertmanager 문서  
  <https://prometheus.io/docs/alerting/latest/alertmanager/>

- Prometheus Alertmanager Configuration  
  <https://prometheus.io/docs/alerting/latest/configuration/>

- Grafana Alerting 공식 문서  
  <https://grafana.com/docs/grafana/latest/alerting/>

- Grafana Notification / Contact Point 공식 문서  
  <https://grafana.com/docs/grafana/latest/alerting/fundamentals/notifications/>

- kube-prometheus-stack Helm Chart  
  <https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack>

- Prometheus Operator / kube-prometheus  
  <https://github.com/prometheus-operator/kube-prometheus>
