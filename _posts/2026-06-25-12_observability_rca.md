---
layout: post
title: "Observability와 RCA: Logging, Metrics, Tracing, Monitoring, Alerting으로 장애 원인 분석하기"
date: 2026-06-25 08:56:00 +0900
categories: [스터디, 인프라, Observability, Troubleshooting]
tags: [Observability, RCA, Logging, Metrics, Tracing, Monitoring, Alerting, Prometheus, Grafana, WhaTap, SLI, SLO]
published: true
---

## 1. 이 문서의 목적

이 문서는 장애가 발생했을 때 단순히 명령어를 실행하는 수준을 넘어서, **로그, 메트릭, 트레이스, 알림을 근거로 원인을 좁히고 Root Cause Analysis(RCA)로 정리하는 방법**을 다룹니다.

앞선 문서들이 네트워크 계층, Kubernetes 리소스, 명령어 기반 점검 방법을 다뤘다면, 이 문서는 장애를 관측하고 증거를 모아 원인과 재발 방지 대책까지 정리하는 단계입니다.

공식 참고자료:

- Prometheus Documentation: https://prometheus.io/docs/introduction/overview/
- PromQL Basics: https://prometheus.io/docs/prometheus/latest/querying/basics/
- Grafana Documentation: https://grafana.com/docs/grafana/latest/
- OpenTelemetry Documentation: https://opentelemetry.io/docs/
- Kubernetes Logging Architecture: https://kubernetes.io/docs/concepts/cluster-administration/logging/
- Kubernetes Metrics: https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/
- Google SRE Book - Monitoring Distributed Systems: https://sre.google/sre-book/monitoring-distributed-systems/

----

## 2. 11번 문서와 12번 문서의 차이

11번 문서는 Kubernetes 리소스 흐름에서 어디가 끊겼는지 찾는 문서입니다.

```text
Ingress
  ↓
Service
  ↓
EndpointSlice
  ↓
Pod
  ↓
Application
```

12번 문서는 장애를 관측하고 RCA로 정리하는 문서입니다.

```text
Alert 발생
  ↓
로그 / 메트릭 / 트레이스 확인
  ↓
영향 범위 파악
  ↓
원인 후보 수립
  ↓
증거로 검증
  ↓
Root Cause 확정
  ↓
임시 조치 / 근본 조치 / 재발 방지 정리
```

즉, 11번은 **장애 지점 탐색**, 12번은 **장애 원인 분석과 보고**가 중심입니다.

----

## 3. Observability란 무엇인가

Observability는 시스템 외부에서 수집한 신호를 통해 내부 상태를 이해하는 능력입니다.

대표적인 세 가지 신호는 다음과 같습니다.

```text
Logs
Metrics
Traces
```

여기에 운영 관점에서는 Monitoring과 Alerting이 함께 붙습니다.

```text
Telemetry 수집
  ↓
Monitoring
  ↓
Alerting
  ↓
Troubleshooting
  ↓
RCA
```

Observability가 중요한 이유는 장애가 발생했을 때 다음 질문에 답해야 하기 때문입니다.

```text
언제부터 문제가 발생했는가?
어떤 사용자가 영향을 받았는가?
어떤 서비스가 느려졌는가?
어떤 변경 이후 문제가 시작되었는가?
네트워크 문제인가, 애플리케이션 문제인가, 인프라 문제인가?
임시 조치로 복구되었는가?
근본 원인은 무엇인가?
재발 방지 대책은 무엇인가?
```

----

## 4. Logging, Metrics, Tracing의 차이

| 항목 | 설명 | 장점 | 한계 |
|---|---|---|---|
| Logging | 이벤트와 에러를 텍스트로 기록 | 상세 원인 파악에 유리 | 양이 많고 상관관계 분석이 어려움 |
| Metrics | 숫자 시계열 데이터 | 추세, 임계치, 알림에 유리 | 상세 요청 내용은 부족 |
| Tracing | 요청이 서비스들을 거친 경로 | 분산 시스템 병목 파악에 유리 | 계측이 필요하고 샘플링 한계 있음 |

예를 들어 API latency가 증가했을 때:

```text
Metrics
  → latency p95 증가 확인

Logs
  → 특정 에러 메시지 확인

Traces
  → DB query span에서 지연 확인
```

세 가지를 함께 봐야 원인을 좁힐 수 있습니다.

----

## 5. Monitoring과 Alerting

Monitoring은 시스템 상태를 지속적으로 관찰하는 것입니다.

Alerting은 특정 조건이 충족되었을 때 담당자에게 알림을 보내는 것입니다.

```text
Metric 수집
  ↓
Rule 평가
  ↓
Alert 발생
  ↓
Notification 전송
  ↓
Operator 확인
```

좋은 Alert는 단순히 수치가 높다는 것을 알려주는 것이 아니라, **조치가 필요한 상황**을 알려야 합니다.

나쁜 Alert 예시:

```text
CPU 80% 초과
```

상황에 따라 정상일 수도 있습니다.

좋은 Alert 예시:

```text
API 5xx 비율이 5분 동안 5% 초과했고, 실제 사용자 요청 성공률이 SLO 기준 이하로 떨어짐
```

----

## 6. Symptoms, Cause, Impact 구분

RCA에서 가장 중요한 것은 증상과 원인을 구분하는 것입니다.

| 구분 | 의미 | 예시 |
|---|---|---|
| Symptom | 겉으로 드러난 현상 | Ingress 503 발생 |
| Impact | 사용자/서비스 영향 | Harbor 로그인 불가 |
| Cause Candidate | 원인 후보 | EndpointSlice empty, Readiness Probe 실패 |
| Root Cause | 검증된 근본 원인 | 잘못된 환경변수로 앱 health endpoint 실패 |
| Trigger | 장애를 유발한 계기 | 신규 이미지 배포 |
| Resolution | 복구 조치 | 이전 버전 롤백 |
| Prevention | 재발 방지 | 배포 전 health check 검증 추가 |

예를 들어 `Ingress 503`은 원인이 아닙니다. 증상입니다.

```text
Ingress 503
  ↓
EndpointSlice empty
  ↓
Pod Ready=False
  ↓
Readiness Probe 500
  ↓
신규 배포에서 DB_HOST 오타
```

여기서 Root Cause는 `DB_HOST 오타로 인해 Readiness Probe가 실패했고, EndpointSlice에서 Pod가 제외된 것`입니다.

----

## 7. RCA 기본 흐름

RCA는 다음 순서로 진행합니다.

```text
1. 증상 확인
2. 영향 범위 파악
3. 발생 시각 확인
4. 최근 변경 사항 확인
5. 로그 / 메트릭 / 트레이스 수집
6. 원인 후보 나열
7. 후보별 증거 검증
8. 임시 복구 조치
9. 근본 원인 확정
10. 재발 방지 대책 수립
11. 장애 보고서 작성
```

중요한 원칙:

```text
추측과 확정 원인을 구분한다.
증거가 없는 결론은 RCA에 쓰지 않는다.
증상만 보고 원인이라고 단정하지 않는다.
```

확실하지 않음: 장애 당시 데이터가 없으면 Root Cause를 확정할 수 없습니다. 이 경우 “확정 원인”이 아니라 “가장 가능성이 높은 원인 후보”로 기록해야 합니다.

----

## 8. Request Flow Tracing

Request Flow Tracing은 하나의 요청이 시스템을 어떻게 통과했는지 추적하는 방법입니다.

예시:

```text
Client
  ↓
Load Balancer
  ↓
Ingress Controller
  ↓
Service
  ↓
Pod
  ↓
Application
  ↓
Database
```

각 구간에서 남길 수 있는 증거:

| 구간 | 확인할 증거 |
|---|---|
| Client | 응답 코드, timeout, DNS 결과 |
| Load Balancer | access log, health check 상태 |
| Ingress Controller | access log, upstream status |
| Service | EndpointSlice 존재 여부 |
| Pod | application log, container restart |
| Database | connection error, slow query |

요청 흐름을 추적할 때는 Request ID 또는 Trace ID가 중요합니다.

----

## 9. X-Request-ID, Trace ID, Span ID

분산 시스템에서는 하나의 요청이 여러 서비스를 거칠 수 있습니다.

```text
Frontend
  ↓
API Gateway
  ↓
Backend API
  ↓
Auth Service
  ↓
Database
```

이때 각 서비스 로그를 연결하려면 공통 식별자가 필요합니다.

| 식별자 | 의미 |
|---|---|
| X-Request-ID | 요청 단위 식별자 |
| Trace ID | 분산 트레이스 전체 식별자 |
| Span ID | 트레이스 내 개별 작업 식별자 |
| Parent Span ID | 상위 작업과의 관계 |

좋은 로그 예시:

```json
{
  "timestamp": "2026-06-24T10:15:31+09:00",
  "level": "ERROR",
  "service": "backend-api",
  "request_id": "req-abc123",
  "trace_id": "trace-001",
  "method": "GET",
  "path": "/api/users",
  "status": 500,
  "latency_ms": 1240,
  "message": "database connection timeout"
}
```

이렇게 구조화되어 있으면 장애 분석이 훨씬 쉬워집니다.

----

## 10. Logging 설계 기준

로그는 장애가 발생한 뒤 가장 먼저 보는 증거 중 하나입니다.

좋은 로그는 다음 정보를 포함합니다.

| 필드 | 설명 |
|---|---|
| timestamp | 발생 시각 |
| level | INFO, WARN, ERROR |
| service | 서비스 이름 |
| namespace | Kubernetes namespace |
| pod | Pod 이름 |
| node | Node 이름 |
| request_id | 요청 식별자 |
| method | HTTP Method |
| path | 요청 경로 |
| status | HTTP Status Code |
| latency_ms | 처리 시간 |
| error | 에러 내용 |
| user_id | 사용자 식별자. 개인정보 주의 필요 |

주의:

- 비밀번호, 토큰, Secret 값을 로그에 남기면 안 됩니다.
- 개인정보는 마스킹해야 합니다.
- 로그 레벨을 구분해야 합니다.
- 에러 로그에는 원인과 context가 있어야 합니다.

나쁜 로그:

```text
error occurred
```

좋은 로그:

```text
DB connection timeout host=postgres.app.svc.cluster.local port=5432 timeout=5s request_id=req-abc123
```

----

## 11. Kubernetes 로그 확인

Pod 로그:

```bash
kubectl logs -n app deploy/backend --tail=100
kubectl logs -n app deploy/backend -f
kubectl logs -n app <pod-name> --previous
```

Ingress Controller 로그:

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=100
```

CoreDNS 로그:

```bash
kubectl -n kube-system logs deploy/coredns --tail=100
```

CNI 로그 예시:

```bash
kubectl -n kube-system logs ds/cilium --tail=100
kubectl -n kube-system logs ds/calico-node --tail=100
```

확실하지 않음: CNI DaemonSet 이름과 namespace는 설치 방식에 따라 다를 수 있습니다. 실제 환경에서 `kubectl get pod -A | grep -i cilium`처럼 먼저 확인해야 합니다.

----

## 12. Metrics란 무엇인가

Metrics는 시간에 따라 변하는 숫자 데이터입니다.

예시:

| Metric | 의미 |
|---|---|
| request_total | 요청 수 |
| request_duration_seconds | 요청 처리 시간 |
| http_requests_total{status="500"} | 500 응답 수 |
| container_cpu_usage_seconds_total | 컨테이너 CPU 사용량 |
| container_memory_working_set_bytes | 컨테이너 메모리 사용량 |
| node_network_receive_bytes_total | 노드 네트워크 수신량 |

Metrics는 추세와 상관관계 분석에 강합니다.

예시:

```text
14:00 배포
14:03 5xx 증가
14:04 latency p95 증가
14:05 Pod restart 증가
```

이 흐름을 보면 배포가 장애와 관련 있을 가능성이 높아집니다.

----

## 13. Prometheus 기본 개념

Prometheus는 Pull 방식으로 target의 metrics endpoint를 주기적으로 scrape합니다.

```text
Application / Exporter
  ↓ /metrics
Prometheus
  ↓ PromQL
Grafana / Alertmanager
```

주요 개념:

| 개념 | 설명 |
|---|---|
| Target | 메트릭 수집 대상 |
| Scrape | target에서 metrics를 가져오는 작업 |
| Job | target 그룹 |
| Time Series | label 조합별 시계열 |
| PromQL | Prometheus Query Language |
| Alert Rule | 알림 조건 |
| Alertmanager | 알림 라우팅/그룹화/중복 제거 |

Kubernetes에서는 ServiceMonitor, PodMonitor 같은 CRD를 사용하는 경우도 많습니다. 이는 Prometheus Operator 구성에 따라 다릅니다.

----

## 14. PromQL 기초

HTTP 요청 수:

```promql
http_requests_total
```

5xx 요청 비율 예시:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

p95 latency 예시:

```promql
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)
```

Pod restart 증가:

```promql
increase(kube_pod_container_status_restarts_total[10m])
```

CPU 사용률 예시:

```promql
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)
```

메모리 사용량 예시:

```promql
container_memory_working_set_bytes{container!=""}
```

주의: 메트릭 이름은 설치된 exporter, application instrumentation, Kubernetes 버전에 따라 다를 수 있습니다.

----

## 15. Grafana에서 보는 방법

Grafana는 Prometheus 같은 데이터 소스를 시각화하는 도구입니다.

장애 분석 시 볼 패널:

| 패널 | 확인 내용 |
|---|---|
| Request Rate | 요청량 급증/급감 |
| Error Rate | 4xx/5xx 증가 여부 |
| Latency p50/p95/p99 | 지연 증가 여부 |
| Pod CPU/Memory | 리소스 부족 여부 |
| Pod Restart | 재시작 증가 여부 |
| Network RX/TX | 네트워크 트래픽 변화 |
| Ingress Status | upstream status, response code |
| Node 상태 | CPU, Memory, Disk, Network |

Grafana에서 중요한 것은 특정 시점의 값 하나가 아니라 **장애 시작 시각 전후의 변화**입니다.

```text
정상 구간
  ↓
변경 시점
  ↓
이상 징후 시작
  ↓
장애 발생
  ↓
복구 조치
  ↓
정상화
```

----

## 16. WhaTap에서 확인할 항목

WhaTap은 APM, 서버 모니터링, Kubernetes 모니터링 등을 제공하는 관측성 도구입니다.

WhaTap에서 장애 분석 시 볼 항목은 다음과 같습니다.

| 영역 | 확인 내용 |
|---|---|
| Application | transaction, error, response time |
| Server | CPU, memory, disk, network |
| Kubernetes | Pod 상태, container restart, node 상태 |
| DB | connection, slow query, error |
| Alert | 알림 발생 시각, 지속 시간 |

WhaTap을 볼 때도 핵심은 같습니다.

```text
장애 시각 기준으로
  ↓
어떤 지표가 먼저 변했는가?
  ↓
그 변화가 증상인가 원인 후보인가?
  ↓
로그와 이벤트로 검증 가능한가?
```

확실하지 않음: WhaTap의 메뉴명과 제공 지표는 사용 중인 상품, 에이전트 버전, 설정에 따라 다를 수 있습니다. 실제 환경의 대시보드 구조에 맞춰 확인해야 합니다.

----

## 17. RED Method

RED Method는 주로 서비스 관측에 사용하는 방법입니다.

```text
Rate
Errors
Duration
```

| 항목 | 의미 | 예시 |
|---|---|---|
| Rate | 초당 요청 수 | requests/sec |
| Errors | 실패 요청 수/비율 | 5xx rate |
| Duration | 요청 처리 시간 | latency p95 |

웹 API나 Ingress, Gateway, Backend 서비스는 RED 기준으로 보는 것이 좋습니다.

예시:

```text
Rate 정상
Errors 증가
Duration 증가
  ↓
애플리케이션 내부 에러 또는 downstream 지연 가능성
```

----

## 18. USE Method

USE Method는 주로 인프라 리소스 분석에 사용합니다.

```text
Utilization
Saturation
Errors
```

| 항목 | 의미 | 예시 |
|---|---|---|
| Utilization | 사용률 | CPU 80%, Disk 70% |
| Saturation | 포화/대기 | run queue, disk queue |
| Errors | 오류 | NIC errors, disk errors |

Node, CPU, Memory, Disk, Network 같은 리소스는 USE 기준으로 봅니다.

예시:

```text
Node network RX drops 증가
  ↓
Pod 통신 timeout 증가
  ↓
CNI 또는 NIC 레벨 문제 후보
```

----

## 19. Golden Signals

Google SRE에서 자주 언급되는 대표 신호입니다.

| 신호 | 설명 |
|---|---|
| Latency | 요청 처리 시간 |
| Traffic | 요청량 |
| Errors | 실패 비율 |
| Saturation | 시스템 포화도 |

장애 분석에서는 이 네 가지를 함께 봅니다.

```text
Traffic 증가
  ↓
Saturation 증가
  ↓
Latency 증가
  ↓
Errors 증가
```

이 흐름이라면 트래픽 증가로 인한 용량 문제가 원인 후보가 될 수 있습니다.

반대로:

```text
Traffic 정상
  ↓
Errors만 갑자기 증가
  ↓
배포/설정/의존성 장애 가능성
```

----

## 20. SLI, SLO, SLA

| 용어 | 의미 |
|---|---|
| SLI | Service Level Indicator. 측정 지표 |
| SLO | Service Level Objective. 목표 수준 |
| SLA | Service Level Agreement. 계약상 보장 수준 |

예시:

```text
SLI: HTTP 성공률
SLO: 30일 기준 99.9% 이상
SLA: 99.5% 미만이면 보상
```

Kubernetes 서비스에서 SLI 예시:

- HTTP 2xx/3xx 성공률
- p95 latency
- API availability
- error rate
- request success rate

SLO가 있어야 장애의 심각도를 객관적으로 판단할 수 있습니다.

----

## 21. Alert Severity

Alert는 심각도에 따라 나누는 것이 좋습니다.

| Severity | 의미 | 예시 |
|---|---|---|
| Critical | 즉시 대응 필요 | 전체 서비스 다운, API 5xx 급증 |
| Warning | 주의 필요 | CPU 높음, latency 증가 |
| Info | 참고 | 배포 완료, 일시적 restart |

나쁜 Alert 설계:

```text
모든 Pod restart에 Critical
```

좋은 Alert 설계:

```text
사용자 요청 실패율이 SLO를 위반할 때 Critical
Pod restart가 증가하지만 사용자 영향이 없으면 Warning
```

Alert Fatigue를 줄이려면 조치 가능한 알림만 강하게 보내야 합니다.

----

## 22. MTTD와 MTTR

| 지표 | 의미 |
|---|---|
| MTTD | Mean Time To Detect. 장애를 감지하기까지 걸린 평균 시간 |
| MTTR | Mean Time To Resolve/Recover. 복구까지 걸린 평균 시간 |

RCA에서는 다음 시각을 기록하는 것이 좋습니다.

```text
장애 시작 시각
감지 시각
대응 시작 시각
임시 복구 시각
완전 복구 시각
```

이를 통해 감지 체계와 대응 프로세스를 개선할 수 있습니다.

----

## 23. 장애 타임라인 작성

장애 분석에서 타임라인은 매우 중요합니다.

예시:

| 시각 | 이벤트 |
|---|---|
| 14:00 | backend v1.2.3 배포 시작 |
| 14:02 | 새 Pod Ready 실패 |
| 14:03 | EndpointSlice backend 감소 |
| 14:04 | Ingress 503 증가 |
| 14:06 | Alert 발생 |
| 14:08 | 담당자 확인 시작 |
| 14:12 | 이전 버전 롤백 |
| 14:15 | 5xx 정상화 |
| 14:20 | 원인 후보 확인 |

타임라인은 추측이 아니라 로그, 이벤트, 메트릭, 배포 기록을 근거로 작성해야 합니다.

----

## 24. 최근 변경 사항 확인

장애는 변경 직후 발생하는 경우가 많습니다.

확인할 변경 사항:

- 애플리케이션 배포
- Helm values 변경
- ConfigMap 변경
- Secret 변경
- Ingress 변경
- Service 변경
- NetworkPolicy 변경
- 인증서 갱신
- CNI 업그레이드
- Node 재부팅
- 방화벽 정책 변경
- 외부 DNS 변경

Kubernetes에서 확인:

```bash
kubectl rollout history deploy/backend -n app
kubectl describe deploy/backend -n app
kubectl get events -n app --sort-by=.lastTimestamp
```

GitOps 환경이라면 ArgoCD sync history와 Git commit을 확인해야 합니다.

----

## 25. 장애 재현(Reproduce)

재현은 원인 후보를 검증하는 과정입니다.

예시:

```text
사용자 제보: Harbor 로그인 실패
  ↓
curl로 로그인 API 호출
  ↓
브라우저와 동일한 Host/Header/TLS 조건 재현
  ↓
Ingress log와 application log 비교
```

재현 시 주의:

- 운영 데이터에 영향을 주지 않는 방식으로 수행
- 동일한 Host Header 사용
- 동일한 TLS/SNI 조건 사용
- 동일한 namespace 또는 Pod 위치에서 테스트
- 임의로 설정을 바꾸지 않기

네트워크 재현 예시:

```bash
curl -vk https://harbor.example.com/api/v2.0/health
curl -v http://<ingress-ip>/api/v2.0/health -H 'Host: harbor.example.com'
kubectl run netshoot -n harbor --rm -it --image=nicolaka/netshoot -- bash
curl -v http://harbor-core.harbor.svc.cluster.local:80/api/v2.0/health
```

----

## 26. 패킷 캡처와 RCA

패킷 캡처는 네트워크 계층에서 실제 패킷이 오가는지 확인하는 강력한 증거입니다.

예시:

```bash
sudo tcpdump -i any -nn host <pod-ip> and port 8080
```

Kubernetes에서는 다음 위치에서 캡처할 수 있습니다.

| 위치 | 목적 |
|---|---|
| Client | 요청이 나가는지 확인 |
| Ingress Node | 외부 요청 도달 여부 |
| Ingress Controller Pod | backend로 전달 여부 |
| Backend Node | Pod로 패킷 도달 여부 |
| Backend Pod | 애플리케이션까지 도달 여부 |

RCA에서 패킷 캡처를 사용할 때는 다음을 구분합니다.

| 패턴 | 의미 |
|---|---|
| SYN만 나감, SYN/ACK 없음 | 대상 도달 실패, 방화벽, NetworkPolicy, 라우팅 문제 |
| SYN/SYN-ACK/ACK 성공 후 RST | 포트 또는 애플리케이션 연결 거부 가능성 |
| TCP 연결 성공, HTTP 응답 500 | 애플리케이션 문제 가능성 |
| TLS Alert | 인증서, 프로토콜, SNI 문제 가능성 |

----

## 27. Health Check 검증

Health Check는 장애 분석에서 매우 중요합니다.

하지만 Health Check가 정확하지 않으면 오히려 장애를 숨길 수 있습니다.

나쁜 Health Check:

```text
/healthz가 항상 200 반환
DB 연결 장애가 있어도 Ready
```

좋은 Health Check:

```text
/readiness
  - 앱 초기화 완료 여부
  - 필수 의존성 연결 가능 여부

/liveness
  - 프로세스가 죽었는지 여부
```

Kubernetes 확인:

```bash
kubectl describe pod -n app <pod-name>
kubectl logs -n app <pod-name> --previous
```

직접 확인:

```bash
kubectl exec -n app <pod-name> -- curl -v http://127.0.0.1:8080/healthz
kubectl exec -n app <pod-name> -- curl -v http://127.0.0.1:8080/readyz
```

RCA에서는 Health Check가 실제 장애를 감지했는지 기록해야 합니다.

----

## 28. End-to-End Flow 검증

장애 복구 후에는 전체 요청 흐름을 검증해야 합니다.

```text
Client
  ↓
DNS
  ↓
Load Balancer / Ingress
  ↓
Service
  ↓
Pod
  ↓
Application
  ↓
Dependency
```

검증 예시:

```bash
# 외부에서 확인
curl -vk https://app.example.com/health

# Ingress 우회
kubectl port-forward -n app svc/backend 8080:80
curl -v http://127.0.0.1:8080/health

# Pod 내부에서 의존성 확인
kubectl exec -n app <pod> -- curl -v http://dependency.app.svc.cluster.local/health

# DNS 확인
kubectl exec -n app <pod> -- nslookup dependency.app.svc.cluster.local
```

E2E 검증은 단일 endpoint만 보는 것이 아니라 사용자 주요 흐름을 확인해야 합니다.

예시:

```text
로그인
  ↓
목록 조회
  ↓
상세 조회
  ↓
쓰기 작업
  ↓
파일 업로드
```

----

## 29. 원인 후보 검증표

장애 분석에서는 원인 후보를 나열하고 증거로 검증합니다.

예시:

| 원인 후보 | 확인 방법 | 결과 | 판단 |
|---|---|---|---|
| DNS 문제 | dig/nslookup | 정상 해석 | 제외 |
| Ingress rule 문제 | describe ingress | Host 정상 | 제외 |
| Endpoint 없음 | get endpointslice | empty | 유력 |
| Pod Ready 실패 | describe pod | readiness 500 | 유력 |
| DB 연결 실패 | app logs | timeout 발생 | Root Cause 후보 |
| NetworkPolicy 차단 | netshoot 테스트 | 허용됨 | 제외 |

이렇게 정리하면 추측이 줄어듭니다.

----

## 30. 5 Whys

5 Whys는 왜를 반복해서 근본 원인에 가까워지는 방법입니다.

예시:

```text
문제: Ingress 503 발생

왜 1: Service backend가 없었다.
왜 2: EndpointSlice가 비어 있었다.
왜 3: Pod가 Ready=False였다.
왜 4: Readiness Probe가 500을 반환했다.
왜 5: 신규 배포에서 DB_HOST 환경변수가 잘못 설정되었다.
```

주의:

- 억지로 5번을 채울 필요는 없습니다.
- 사람 탓으로 끝내면 안 됩니다.
- 시스템적으로 막을 방법을 찾아야 합니다.

나쁜 결론:

```text
담당자가 환경변수를 잘못 넣었다.
```

좋은 결론:

```text
배포 전 환경변수 검증과 readiness endpoint 검증이 CI/CD에 포함되어 있지 않았다.
```

----

## 31. 임시 조치와 근본 조치 구분

| 구분 | 의미 | 예시 |
|---|---|---|
| Mitigation | 사용자 영향 줄이기 위한 임시 조치 | 이전 버전 롤백 |
| Resolution | 장애 복구 조치 | 환경변수 수정 후 재배포 |
| Prevention | 재발 방지 | CI validation 추가 |

예시:

```text
임시 조치: backend v1.2.2로 롤백
근본 조치: v1.2.3 환경변수 수정
재발 방지: Helm values schema 검증 추가
```

RCA에는 이 세 가지를 분리해서 적어야 합니다.

----

## 32. 장애 보고서 템플릿

```markdown
# 장애 보고서

## 1. 개요
- 장애명:
- 발생 일시:
- 감지 일시:
- 복구 일시:
- 작성자:

## 2. 영향 범위
- 영향 서비스:
- 영향 사용자:
- 영향 기능:
- 오류 유형:

## 3. 증상
- 사용자가 본 증상:
- 시스템에서 관측된 증상:
- 주요 에러 메시지:

## 4. 타임라인
| 시각 | 내용 |
|---|---|
|  |  |

## 5. 원인 분석
- 원인 후보 1:
- 원인 후보 2:
- 제외한 원인:
- 확정 원인:

## 6. 근거
- 로그:
- 메트릭:
- 트레이스:
- Kubernetes Event:
- 배포 이력:

## 7. 조치 내용
- 임시 조치:
- 복구 조치:
- 검증 방법:

## 8. 재발 방지 대책
- 단기:
- 중기:
- 장기:

## 9. 후속 작업
| 작업 | 담당자 | 기한 | 상태 |
|---|---|---|---|
|  |  |  |  |
```

----

## 33. 장애 분석 예시: Ingress 503

증상:

```text
사용자가 https://app.example.com 접속 시 503 응답
```

타임라인:

```text
14:00 backend 신규 배포
14:02 backend Pod Ready=False
14:03 EndpointSlice empty
14:04 Ingress 503 증가
14:06 Alert 발생
14:10 롤백
14:13 정상화
```

증거:

```bash
kubectl get endpointslice -n app -l kubernetes.io/service-name=backend
kubectl describe pod -n app <pod>
kubectl logs -n app deploy/backend --previous
```

원인 후보:

| 후보 | 결과 |
|---|---|
| Ingress host 오류 | 정상 |
| Service selector 오류 | 정상 |
| Endpoint 없음 | 확인됨 |
| Pod readiness 실패 | 확인됨 |
| DB 연결 오류 | 로그에서 확인됨 |

Root Cause:

```text
신규 배포에서 DB_HOST 환경변수가 잘못 설정되어 backend readiness endpoint가 500을 반환했다. 이로 인해 Pod가 EndpointSlice에서 제외되었고, Ingress가 Service backend를 찾지 못해 503을 반환했다.
```

재발 방지:

- Helm values schema 검증
- 배포 전 readiness endpoint smoke test
- ArgoCD sync 후 자동 health 검증
- DB_HOST secret/config 변경 리뷰 강화

----

## 34. 장애 분석 예시: TLS 인증서 오류

증상:

```text
브라우저에서 인증서 경고 발생
curl에서 certificate verify failed 발생
```

확인:

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com
kubectl get secret -n app app-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -subject -issuer -dates
kubectl describe ingress -n app app-ingress
```

볼 것:

- 인증서 만료 여부
- SAN에 도메인이 포함되어 있는지
- Secret namespace가 Ingress와 같은지
- Ingress TLS secretName이 맞는지
- cert-manager Certificate 상태

Root Cause 예시:

```text
cert-manager HTTP-01 challenge가 IngressClass 불일치로 실패했고 인증서가 갱신되지 않았다. 만료된 Secret이 계속 사용되어 TLS 오류가 발생했다.
```

----

## 35. 장애 분석 예시: DNS 실패

증상:

```text
Pod 내부에서 backend.app.svc.cluster.local 해석 실패
```

확인:

```bash
kubectl exec -n app <pod> -- cat /etc/resolv.conf
kubectl exec -n app <pod> -- nslookup backend.app.svc.cluster.local
kubectl get svc -n app backend
kubectl -n kube-system get pod -l k8s-app=kube-dns
kubectl -n kube-system logs deploy/coredns --tail=100
```

원인 후보:

- Service 이름 오류
- namespace 오류
- CoreDNS Pod 장애
- CoreDNS upstream DNS 장애
- NetworkPolicy가 DNS egress 차단
- CNI 문제

Root Cause 예시:

```text
새로 적용한 NetworkPolicy가 app namespace Pod의 kube-dns egress UDP/53을 허용하지 않아 Service DNS 조회가 timeout되었다.
```

재발 방지:

- NetworkPolicy 변경 전 DNS egress 허용 검증
- netshoot 기반 smoke test 추가
- DNS 실패율 alert 추가

----

## 36. 장애 분석 예시: Timeout

증상:

```text
API 요청이 간헐적으로 timeout
```

확인할 지표:

- request latency p95/p99
- 5xx rate
- Pod CPU/Memory
- Pod restart
- DB connection pool
- Node network errors/drops
- Ingress upstream response time

확인 명령:

```bash
kubectl top pod -n app
kubectl get events -n app --sort-by=.lastTimestamp
kubectl logs -n app deploy/backend --tail=100
```

가능한 원인:

- DB slow query
- connection pool 고갈
- CPU throttling
- Memory pressure
- NetworkPolicy 또는 CNI packet drop
- 외부 API 지연

RCA에서는 timeout이라는 증상만으로 네트워크 문제라고 단정하면 안 됩니다.

----

## 37. Dashboard 설계 기준

좋은 대시보드는 장애 대응 질문에 바로 답해야 합니다.

서비스 대시보드:

- Request Rate
- Error Rate
- Latency p50/p95/p99
- Saturation
- Top error path
- 최근 배포 버전

Kubernetes 대시보드:

- Pod Ready 수
- Pod Restart 수
- CPU/Memory 사용량
- OOMKilled
- Node Ready
- Network RX/TX

Ingress 대시보드:

- status code 분포
- upstream status
- upstream response time
- host/path별 요청량

Node 대시보드:

- CPU
- Memory
- Disk
- Network drops/errors
- Load average

----

## 38. Alert 설계 기준

Alert는 다음 형태가 좋습니다.

```text
사용자 영향 + 지속 시간 + 조치 가능성
```

예시:

```text
High5xxRate
조건: 5분 동안 5xx 비율이 5% 초과
영향: 사용자 요청 실패 증가
확인: Ingress log, backend log, recent deploy
```

```text
EndpointEmpty
조건: 중요 Service의 EndpointSlice가 3분 이상 empty
영향: Service backend 없음
확인: Pod readiness, selector, rollout
```

```text
PodRestartLoop
조건: 10분 동안 restart 증가
영향: 앱 불안정
확인: logs --previous, describe pod, OOMKilled
```

----

## 39. RCA에서 피해야 할 실수

- 증상을 원인으로 적기
- 추측을 확정처럼 쓰기
- 장애 시각을 대략적으로만 쓰기
- 로그/메트릭 증거 없이 결론 내기
- 사람 탓으로 끝내기
- 임시 조치와 재발 방지를 구분하지 않기
- 영향 범위를 적지 않기
- 복구 검증 방법을 적지 않기
- 후속 작업 담당자와 기한을 정하지 않기

나쁜 RCA:

```text
원인: 서버 오류
조치: 재시작
```

좋은 RCA:

```text
원인: 14:00 배포된 backend v1.2.3에서 DB_HOST 환경변수가 잘못 설정되어 readiness probe가 실패했다. 이로 인해 EndpointSlice가 empty가 되었고 Ingress에서 503이 발생했다.
조치: v1.2.2로 롤백 후 EndpointSlice 복구와 5xx 정상화를 확인했다.
재발 방지: Helm values schema 검증과 배포 후 smoke test를 추가한다.
```

----

## 40. 최종 체크리스트

장애 발생 시:

```text
1. 사용자가 보는 증상 확인
2. 장애 시작 시각 확인
3. 영향 범위 확인
4. 최근 변경 사항 확인
5. 로그 확인
6. 메트릭 확인
7. 트레이스 확인
8. Kubernetes Event 확인
9. 원인 후보 나열
10. 증거로 후보 제거
11. 임시 복구
12. E2E 검증
13. Root Cause 정리
14. 재발 방지 대책 수립
```

관측성 신호별 질문:

| 신호 | 질문 |
|---|---|
| Logs | 어떤 에러가 발생했는가? 요청이 앱까지 도달했는가? |
| Metrics | 언제부터 수치가 변했는가? 어떤 지표가 먼저 변했는가? |
| Traces | 어느 구간에서 지연되었는가? |
| Events | Kubernetes 리소스에 어떤 변화가 있었는가? |
| Deploy history | 장애 직전에 무엇이 바뀌었는가? |
| Packet capture | 패킷이 실제로 오갔는가? |

핵심 문장:

```text
장애 분석은 명령어 실행이 아니라 증거를 모아 원인 후보를 제거하는 과정이다.
```