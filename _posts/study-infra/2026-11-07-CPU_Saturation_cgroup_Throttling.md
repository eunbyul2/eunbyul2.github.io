---
title: "Linux CPU 병목 분석 및 서비스 성능 최적화 - CPU Saturation과 cgroup Throttling 비교"
date: 2026-11-07 20:00:00 +0900
categories: [Infrastructure, Linux, Performance]
tags: [Linux, CPU, CPU-Saturation, CPU-Throttling, cgroup, Kubernetes, Performance-Tuning, Troubleshooting]
published: true
---

# Linux CPU 병목 분석 및 서비스 성능 최적화
### CPU Saturation과 cgroup Throttling 비교

## 1. CPU 사용률이 낮아도 서비스가 느려질 수 있는 이유

서비스의 응답 속도가 느려지면 가장 먼저 확인하는 지표 중 하나가 CPU 사용률이다.

하지만 CPU 사용률만으로 성능 병목 여부를 판단하기는 어렵다.

예를 들어 CPU 사용률이 95%인 서버에서 응답 지연이 발생한다면 CPU 자원 부족을 의심할 수 있다. 반면 CPU 사용률이 40%에 불과한 서버에서도 특정 프로세스가 CPU 실행 시간을 충분히 확보하지 못해 서비스가 느려질 수 있다.

특히 Kubernetes와 같은 컨테이너 환경에서는 Linux cgroup을 통해 CPU 사용량이 제한되므로, 노드에 CPU 자원이 남아 있어도 개별 컨테이너에서 CPU Throttling이 발생할 수 있다.

따라서 CPU 성능을 분석할 때는 **CPU Utilization(사용률), CPU Saturation(포화도), CPU Throttling(실행 제한)**을 구분해야 한다.

| 구분 | 의미 | 주요 원인 |
|---|---|---|
| CPU Utilization | CPU가 작업을 실행하는 데 사용된 시간의 비율 | 실제 CPU 연산 부하 |
| CPU Saturation | CPU를 사용하려는 작업이 처리 능력을 초과해 실행을 기다리는 상태 | CPU 자원 부족, 작업 경합 |
| CPU Throttling | cgroup 등의 CPU 대역폭 제한에 따라 실행이 일시적으로 제한되는 현상 | CPU Quota 소진 |
| CPU Steal Time | 가상머신이 하이퍼바이저로부터 CPU 실행 시간을 확보하지 못한 비율 | 호스트의 CPU 스케줄링 및 자원 경합 |

CPU 사용률이 높다고 반드시 병목이 발생한 것은 아니며, CPU 사용률이 낮다고 CPU 관련 문제가 없다고 판단할 수도 없다.

---

## 2. CPU Saturation이란?

CPU Saturation은 CPU에서 실행하려는 작업이 CPU 처리 능력을 초과하여 실행 대기가 발생하는 상태를 의미한다.

예를 들어 4개의 논리 CPU가 있는 서버에서 CPU를 지속적으로 사용하려는 작업이 10개 존재한다고 가정한다.

이 경우 한 시점에 4개의 작업이 CPU를 사용하고 있다면 나머지 작업은 CPU 실행 시간을 할당받기 위해 대기할 수 있다.

```text
4 vCPU Linux Server

CPU 0 ─── Process A 실행
CPU 1 ─── Process B 실행
CPU 2 ─── Process C 실행
CPU 3 ─── Process D 실행

Run Queue
  ├── Process E
  ├── Process F
  ├── Process G
  ├── Process H
  ├── Process I
  └── Process J
```

위 구조는 CPU 실행 경쟁을 설명하기 위해 단순화한 것이다. 실제 Linux 스케줄러는 작업을 시분할로 실행하고 CPU별 실행 큐를 관리한다.

CPU를 기다리는 시간이 증가하면 애플리케이션이 요청을 처리하는 시간에도 영향을 줄 수 있다.

### 2.1 CPU Saturation의 주요 지표

#### CPU Utilization

CPU가 얼마나 사용되고 있는지를 나타내는 지표이다.

```bash
mpstat -P ALL 1 5
```

주요 항목:

- `%usr`: 사용자 영역에서 CPU가 사용된 시간
- `%sys`: 커널 영역에서 CPU가 사용된 시간
- `%idle`: CPU 유휴 시간
- `%iowait`: CPU가 유휴 상태에서 I/O 완료를 기다린 시간
- `%steal`: 가상화 환경에서 하이퍼바이저에 의해 확보하지 못한 CPU 시간

CPU 사용률이 높다면 CPU 자원 사용량이 많다는 사실을 알 수 있지만, CPU Saturation 여부는 실행 대기 관련 지표를 함께 확인해야 한다.

#### Run Queue

Linux의 `vmstat` 명령어로 CPU 실행 가능 작업 수를 확인할 수 있다.

```bash
vmstat 1 5
```

출력 예시:

```text
procs -----------cpu----------
 r  b   us sy id wa st
10  0   92  3  5  0  0
11  0   94  2  4  0  0
12  0   93  3  4  0  0
```

여기서 `r`은 실행 중이거나 실행 가능한 상태의 작업 수를 나타낸다.

4 vCPU 환경에서 `r`이 지속적으로 10~12 수준이라면 CPU 실행 경쟁이 발생하고 있을 가능성이 높다.

다만 `r`은 실행 중인 작업도 포함하며, 순간적인 수치만으로 병목을 확정할 수는 없다.

#### Load Average

```bash
uptime
```

출력 예시:

```text
load average: 6.50, 5.20, 3.10
```

각각 최근 1분, 5분, 15분의 평균 시스템 부하를 의미한다.

Linux의 Load Average에는 CPU 실행 가능 작업 외에도 중단 불가능한 대기 상태의 작업이 포함된다.

따라서 Load Average가 높다고 해서 무조건 CPU Saturation이라고 판단해서는 안 된다.

CPU 사용률, Run Queue, I/O 대기 상태를 함께 확인해야 한다.

---

## 3. CPU Throttling이란?

CPU Throttling은 프로세스 또는 컨테이너가 설정된 CPU 사용 시간의 한도에 도달했을 때 Linux 커널이 실행을 제한하는 현상이다.

Kubernetes에서는 컨테이너에 설정한 CPU Limit이 Linux cgroup의 CPU 대역폭 제어 기능을 통해 적용된다.

예를 들어 다음과 같이 설정했다고 가정한다.

```yaml
resources:
  requests:
    cpu: "250m"
  limits:
    cpu: "500m"
```

CPU Request는 Pod 스케줄링과 CPU 자원 배분에 사용되며, CPU Limit은 컨테이너의 CPU 실행 시간에 상한을 적용한다.

`500m`은 평균적으로 논리 CPU 0.5개에 해당하는 CPU 시간 예산을 의미한다.

### 3.1 Linux cgroup의 CPU 제한 원리

cgroup(Control Groups)은 Linux 커널에서 프로세스 그룹의 자원 사용을 관리하는 기능이다.

cgroup v2에서는 `cpu.max` 파일을 통해 CPU 대역폭 제한을 확인할 수 있다.

```bash
cat /sys/fs/cgroup/cpu.max
```

출력 예시:

```text
50000 100000
```

| 항목 | 값 | 의미 |
|---|---|---|
| CPU Quota | 50,000μs | 한 주기에 사용할 수 있는 총 CPU 실행 시간 |
| CPU Period | 100,000μs | CPU 실행 예산을 관리하는 주기 |

즉, 100ms 주기마다 총 50ms의 CPU 실행 시간이 주어지는 설정이다.

CPU 사용량이 Quota에 도달하면 다음 주기에 실행 예산이 보충될 때까지 실행이 제한될 수 있다.

```text
CPU Limit: 500m
CPU Period: 100ms

0ms                50ms              100ms
|-------------------|------------------|
       CPU 실행          Throttling
       Quota 소진        실행 제한

다음 Period에서 CPU 실행 예산 보충
```

위 예시는 단일 CPU에서 계속 실행되는 작업을 가정한 것이다.

멀티스레드 환경에서는 여러 CPU에서 동시에 사용한 CPU 시간이 합산되므로 실행 예산이 더 빠르게 소진될 수 있다.

### 3.2 CPU Saturation과 Throttling의 차이

| 비교 항목 | CPU Saturation | CPU Throttling |
|---|---|---|
| 주요 원인 | CPU 처리 능력 대비 높은 작업 수요 | cgroup CPU Quota 소진 |
| CPU 사용률 | 일반적으로 높게 관측될 수 있음 | 노드 사용률이 낮아도 발생 가능 |
| 작업 지연 | CPU 실행 기회를 기다림 | CPU 사용 제한으로 실행이 지연됨 |
| 주요 확인 지표 | Run Queue, CPU 사용률, Scheduler Latency | `nr_throttled`, `throttled_usec` |
| 주요 개선 방향 | CPU 부하 감소, 자원 확장, 부하 분산 | CPU Limit 재설정, CPU 사용량 최적화 |

두 현상은 모두 CPU 실행 지연을 유발할 수 있지만 발생 원인과 해결 방법이 다르다.

또한 CPU Saturation과 Throttling은 동시에 발생할 수도 있다.

---

## 4. 서비스 성능 저하 시나리오

동일하게 API 응답 시간이 증가하는 상황에서 두 가지 CPU 병목을 비교한다.

아래 수치는 모두 가상의 측정값이다.

### 4.1 사례 A: CPU Saturation

4 vCPU Linux 서버에서 API 요청이 증가하면서 CPU 부하가 지속적으로 높아졌다고 가정한다.

| 성능 지표 | 정상 상태 | 장애 상태 |
|---|---:|---:|
| CPU 사용률 | 35% | 95% |
| `vmstat r` | 1~2 | 10~12 |
| Load Average (1분) | 1.5 | 9.2 |
| API P95 응답 시간 | 180ms | 1,500ms |

CPU 사용률이 높아지고 실행 가능한 작업 수도 증가했다.

이 경우 CPU 작업 수요가 처리 능력을 초과하여 실행 대기가 누적되고 있을 가능성이 높다.

**원인 가설:** API 요청 증가로 CPU 연산 작업이 집중되어 CPU Saturation이 발생했고, CPU 실행 대기가 서비스 응답 시간을 증가시켰다.

단, 실제 원인으로 확정하려면 프로세스별 CPU 사용량 및 서비스 처리 시간을 추가로 분석해야 한다.

### 4.2 사례 B: CPU Throttling

이번에는 8 vCPU Kubernetes Worker Node에서 API 서비스를 운영한다고 가정한다.

노드 전체 CPU 사용률은 낮지만 API 컨테이너에는 `500m`의 CPU Limit이 설정되어 있다.

| 성능 지표 | 정상 상태 | 장애 상태 |
|---|---:|---:|
| Node CPU 사용률 | 25% | 40% |
| Container CPU 사용량 | 180m | 480m |
| Container CPU Limit | 500m | 500m |
| Throttling 발생 주기 비율 | 2% | 35% |
| API P95 응답 시간 | 150ms | 1,200ms |

노드 CPU 사용률이 40%인데도 특정 API의 응답 시간이 증가했다.

해당 컨테이너의 CPU 사용량이 Limit에 근접하고, Throttling 발생 비율도 상승한 상황이다.

**원인 가설:** API 컨테이너의 CPU Limit이 실제 처리 수요에 비해 낮아 CPU Throttling이 발생했고, 이로 인해 서비스 응답이 지연되었다.

하지만 실제 원인 여부는 동일 부하에서 CPU Limit을 조정한 후 응답 시간과 Throttling 지표가 개선되는지 확인해야 한다.

---

## 5. CPU 병목 분석 방법

### 5.1 CPU 전체 사용률 및 실행 대기 확인

```bash
# CPU 개수
nproc

# CPU별 사용률
mpstat -P ALL 1 5

# CPU 실행 가능 작업 수
vmstat 1 5

# 시스템 부하
uptime

# 프로세스별 CPU 사용량
pidstat -u 1 5
```

`vmstat`의 `r` 값이 지속적으로 높고 CPU 사용률이 포화 상태에 가깝다면 CPU Saturation을 의심할 수 있다.

반면 노드 CPU에 여유가 있다면 개별 프로세스나 컨테이너에 적용된 자원 제한을 추가로 확인해야 한다.

### 5.2 Linux PSI를 통한 CPU Pressure 확인

Linux에서는 PSI(Pressure Stall Information)를 통해 자원 부족으로 작업이 정체되는 정도를 확인할 수 있다.

```bash
cat /proc/pressure/cpu
```

출력 예시:

```text
some avg10=12.50 avg60=8.20 avg300=4.10 total=123456789
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

`some`은 관측 범위에서 하나 이상의 작업이 CPU 자원 때문에 정체된 시간의 비율을 의미한다.

`avg10`, `avg60`, `avg300`은 각각 10초, 60초, 300초 평균을 나타낸다.

시스템 전체 CPU PSI와 특정 컨테이너의 PSI는 관측 범위가 다르므로, 컨테이너 문제를 분석할 때는 해당 cgroup의 `cpu.pressure`를 확인하는 것이 적합하다.

PSI는 실제 작업이 자원 부족으로 대기하는 정도를 파악하는 데 유용하지만 CPU Throttling 여부를 단독으로 확정하는 지표는 아니다.

### 5.3 Kubernetes CPU Limit 확인

```bash
kubectl describe pod <POD_NAME> \
  -n <NAMESPACE>
```

CPU Request와 Limit 설정을 확인한다.

```bash
kubectl top nodes
kubectl top pods -n <NAMESPACE>
```

노드와 Pod의 CPU 사용량을 비교한다.

### 5.4 cgroup CPU Throttling 통계 확인

cgroup v2에서는 대상 컨테이너의 `cpu.stat` 파일에서 CPU 실행 제한 통계를 확인할 수 있다.

```bash
cat /sys/fs/cgroup/cpu.stat
```

출력 예시:

```text
usage_usec 125000000
nr_periods 10000
nr_throttled 3500
throttled_usec 72000000
```

| 항목 | 의미 |
|---|---|
| `usage_usec` | 누적 CPU 사용 시간 |
| `nr_periods` | CPU 대역폭 제어 주기 수 |
| `nr_throttled` | Throttling이 발생한 주기 수 |
| `throttled_usec` | 누적 Throttling 시간 |

이 예시에서는 10,000개 주기 중 3,500개 주기에서 Throttling이 발생했으므로 발생 주기 비율은 35%다.

단, 이 값은 전체 API 요청의 35%가 지연되었다는 의미가 아니다.

또한 카운터는 누적되므로 특정 장애 구간의 Throttling 비율을 확인하려면 해당 구간의 증가량을 계산해야 한다.

`cpu.stat`이 실제 대상 컨테이너의 cgroup 통계인지 확인해야 하며, 컨테이너 런타임과 cgroup namespace 구성에 따라 경로가 달라질 수 있다.

### 5.5 Prometheus를 활용한 Throttling 분석

Prometheus에서 관련 cAdvisor 지표가 제공되는 경우 다음 PromQL로 최근 5분간 Throttling 발생 주기 비율을 계산할 수 있다.

```promql
sum by (namespace, pod, container) (
  rate(container_cpu_cfs_throttled_periods_total{
    container!="", container!="POD"
  }[5m])
)
/
clamp_min(
  sum by (namespace, pod, container) (
    rate(container_cpu_cfs_periods_total{
      container!="", container!="POD"
    }[5m])
  ),
  1e-9
)
```

컨테이너 CPU 사용량은 다음과 같이 확인한다.

```promql
sum by (namespace, pod, container) (
  rate(container_cpu_usage_seconds_total{
    container!="", container!="POD"
  }[5m])
)
```

실제 지표 노출 여부와 라벨 구성은 클러스터의 모니터링 환경에 따라 다를 수 있다.

성능 병목 분석에서는 CPU Throttling 지표만 확인하지 않고 CPU 사용량과 API P95/P99 응답 시간, 오류율 및 처리량을 함께 비교해야 한다.

---

## 6. 병목 원인별 단기 개선책

### 6.1 CPU Saturation이 발생한 경우

CPU Saturation이 실제 병목으로 확인되었다면 다음 방안을 검토할 수 있다.

**① CPU 사용량이 높은 프로세스 분석**

```bash
ps -eo pid,comm,%cpu --sort=-%cpu | head
```

특정 프로세스에서 비효율적인 연산이 발생하는지 확인한다.

**② 불필요한 CPU 부하 감소**

비필수 배치 작업이나 백그라운드 작업이 서비스 피크 시간대에 집중되어 있다면 실행 시점을 조정한다.

**③ CPU 자원 확장 및 부하 분산**

실제 CPU 용량이 부족하다면 VM vCPU 증설, 워크로드 재배치, 애플리케이션 인스턴스 확장 등을 검토한다.

단, 애플리케이션의 병렬 처리 특성과 실제 CPU 사용량을 고려해야 한다.

### 6.2 CPU Throttling이 발생한 경우

CPU Throttling이 서비스 성능 저하의 주요 원인으로 확인되었다면 CPU Limit을 조정할 수 있다.

기존 설정:

```yaml
resources:
  requests:
    cpu: "250m"
  limits:
    cpu: "500m"
```

변경 검토안:

```yaml
resources:
  requests:
    cpu: "250m"
  limits:
    cpu: "1000m"
```

CPU Limit을 상향하면 컨테이너가 사용할 수 있는 CPU 실행 예산이 증가한다.

다만 CPU Request도 함께 조정해야 하는지는 실제 CPU 사용량과 노드 가용 자원을 기준으로 판단해야 한다.

노드 자체가 CPU 포화 상태라면 Limit을 높이는 것만으로 문제가 해결되지 않을 수 있다.

또한 CPU Limit을 제거하는 방법도 검토할 수 있지만, 다른 워크로드와의 CPU 경합 및 자원 격리 요구사항을 고려해야 한다.

---

## 7. 장기적인 성능 최적화 방안

### 7.1 워크로드별 CPU 리소스 정책 수립

서비스별 CPU 사용 특성과 응답 시간 요구사항을 분석하여 CPU Request/Limit을 설정한다.

특히 순간적인 CPU 부하가 많은 API 서비스와 지속적으로 CPU를 사용하는 Batch 작업은 서로 다른 리소스 정책이 필요할 수 있다.

### 7.2 CPU 성능 모니터링

Prometheus와 Grafana를 활용하여 다음 지표를 수집한다.

- 노드 CPU 사용률
- CPU 실행 대기 및 Pressure
- 컨테이너 CPU 사용량
- CPU Throttling 발생 비율
- API P95/P99 응답 시간
- API 오류율 및 처리량

지표를 개별적으로 확인하는 것이 아니라 동일한 시간대에 비교하여 자원 상태와 서비스 성능 사이의 연관성을 분석한다.

### 7.3 부하 테스트와 용량 계획

예상 트래픽에서 CPU 사용량과 응답 시간이 어떻게 변하는지 측정한다.

이를 통해 워크로드별 적절한 CPU 자원량을 산정하고, VM 증설이나 Kubernetes Pod 확장 여부를 결정한다.

### 7.4 리소스 설정 자동화

Kubernetes 환경에서는 Helm Chart의 `values.yaml`을 통해 CPU Request/Limit을 관리할 수 있다.

GitOps 기반으로 설정을 관리하면 리소스 변경 이력을 추적하고 환경별 설정 차이를 줄일 수 있다.

---

## 8. 성능 개선 효과 검증

CPU 병목 원인에 따라 개선책을 적용한 뒤 동일한 부하 조건에서 개선 전후의 성능을 비교해야 한다.

다음 표는 CPU Throttling 사례의 가상 개선 목표이다.

| 지표 | 개선 전 | 개선 목표 |
|---|---:|---:|
| CPU Limit | 500m | 1000m |
| Throttling 발생 주기 비율 | 35% | 5% 이하 |
| API P95 응답 시간 | 1,200ms | 300ms 이하 |
| API 오류율 | 1.5% | 0.1% 이하 |
| API 처리량 | 300 RPS | 300 RPS 이상 유지 |

※ 개선 목표는 실제 성능 측정 결과가 아니다.

CPU Limit 변경 후 Throttling이 감소하고, 동일한 처리량에서 API 응답 시간과 오류율이 개선된다면 CPU 제한이 서비스 병목에 기여했다는 근거가 된다.

반면 응답 시간이 개선되지 않았다면 CPU Limit 이외의 병목 원인을 추가로 분석해야 한다.

또한 개선 과정에서 다른 Pod의 CPU 사용량이나 응답 시간이 악화되지 않았는지도 확인해야 한다.

---

## 9. 결론

Linux 서버의 CPU 성능을 분석할 때 CPU 사용률만으로 병목 여부를 판단해서는 안 된다.

CPU Saturation은 실제 CPU 처리 능력에 비해 실행하려는 작업이 많아 대기가 발생하는 현상이며, CPU Throttling은 cgroup의 CPU 사용 제한으로 인해 실행이 일시적으로 제한되는 현상이다.

두 현상은 모두 서비스 응답 지연을 유발할 수 있지만 원인과 해결 방법이 다르다.

CPU Saturation이 문제라면 CPU 부하 감소, 자원 확장, 워크로드 분산을 검토해야 한다.

반면 CPU Throttling이 문제라면 CPU Limit과 실제 CPU 수요를 비교하고 적절한 자원 제한 정책을 적용해야 한다.

장기적으로는 CPU 사용률뿐 아니라 실행 대기, CPU Pressure, Throttling, 서비스 응답 시간을 함께 모니터링하고 부하 테스트를 통해 적정 자원량을 산정해야 한다.

**서버 성능 최적화의 핵심은 단순히 CPU 사용률을 낮추는 것이 아니라, 실제 병목 원인을 확인하고 서비스의 응답 시간과 처리량이 개선되었는지 검증하는 것이다.**

---

## 10. 참고자료

- [Brendan Gregg — USE Method](https://www.brendangregg.com/usemethod.html)
- [Linux Kernel — Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)
- [Linux Kernel — Pressure Stall Information](https://docs.kernel.org/accounting/psi.html)
- [Kubernetes — Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [cAdvisor — Prometheus Metrics](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md)