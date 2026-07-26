---
layout: post
title: "[인프라 구성·배포 with Claude 모각코] Chapter 8: 고도화"
date: 2026-07-26 12:00:00 +0900
categories: [CloudNet, Kubernetes, GitOps, Observability]
tags: [Kubernetes, Kafka, Strimzi, Tempo, OpenTelemetry, CronJob, GitOps]
published: true
---

# Chapter 8: 고도화

이번 챕터에서는 기존 애플리케이션에 Kafka를 추가해 이벤트 기반 구조를 만들고, Tempo와 OpenTelemetry를 이용해 분산 추적 환경을 구성한 뒤, CronJob과 Guardrail을 통해 운영 자동화와 안전장치까지 적용했다.

앞선 챕터까지는 애플리케이션을 배포하고 GitOps 방식으로 변경 사항을 반영하는 흐름에 집중했다면, 이번 챕터부터는 단순히 애플리케이션을 실행하는 것을 넘어 실제 운영 환경에서 필요한 메시지 처리, 요청 추적, 주기 작업, 위험 명령 방지 같은 기능을 추가했다.

---

## 1. Kafka를 이용한 이벤트 기반 처리

기존 구조에서는 API가 요청을 받으면 필요한 작업을 같은 흐름 안에서 바로 처리했다. 이런 방식은 구조가 단순하지만 처리 시간이 긴 작업이나 대량의 요청이 들어오는 상황에서는 API 응답 속도와 안정성에 영향을 줄 수 있다.

이번 실습에서는 Kafka를 추가해 요청을 바로 처리하지 않고 이벤트를 Kafka에 전달한 뒤, 별도의 Consumer가 이를 읽어 처리하는 구조를 만들었다.

```text
Client
  ↓
API Server
  ↓
Kafka Topic
  ↓
Consumer
  ↓
실제 작업 처리
```

처음에는 서비스 규모가 크지 않은데 Kafka까지 사용하는 것이 다소 복잡하게 느껴졌다. 하지만 Producer와 Consumer를 분리하면 API는 이벤트를 전달한 뒤 빠르게 응답할 수 있고, Consumer 처리 속도가 느려지더라도 Kafka에 메시지가 남아 있기 때문에 전체 요청이 바로 유실되지 않는다는 장점이 있었다.

### 1-1) Kafka 구성 요소 간단히 이해하기

Kafka에서는 메시지를 보내는 쪽을 Producer, 메시지를 읽어 처리하는 쪽을 Consumer라고 한다. Producer가 보낸 메시지는 Topic에 저장되며, Topic은 다시 여러 Partition으로 나눌 수 있다.

```text
Producer
   ↓
Topic
 ├─ Partition 0
 ├─ Partition 1
 └─ Partition 2
   ↓
Consumer Group
```

Partition을 여러 개로 나누면 여러 Consumer가 메시지를 나누어 처리할 수 있다. 같은 Consumer Group 안에서는 하나의 Partition을 한 Consumer가 담당하므로 Consumer 수를 늘려 병렬 처리 성능을 높일 수 있다.

Offset은 Consumer가 Partition의 어느 위치까지 메시지를 처리했는지를 나타내는 값이다. Consumer가 재시작하더라도 마지막으로 처리한 Offset을 기준으로 다시 메시지를 읽을 수 있다.

이번 실습을 통해 Kafka는 단순히 메시지를 전달하는 도구가 아니라, 처리할 데이터를 일정 기간 보관하고 Consumer의 처리 위치까지 관리하는 이벤트 스트리밍 플랫폼에 가깝다는 점을 이해할 수 있었다.

### 1-2) Strimzi Operator 설치

Kubernetes 환경에서 Kafka를 직접 구성하려면 Broker, Listener, Storage, 사용자 권한 등 여러 설정을 관리해야 한다. 이번 실습에서는 Strimzi Operator를 사용해 Kafka Cluster를 Kubernetes Resource 형태로 배포했다.

```bash
kubectl create namespace kafka

helm repo add strimzi https://strimzi.io/charts/
helm repo update

helm install strimzi-kafka-operator   strimzi/strimzi-kafka-operator   --namespace kafka
```

Operator가 정상적으로 실행되는지 확인했다.

```bash
kubectl get pods -n kafka
```

Strimzi를 사용하면 Kafka Cluster와 Topic을 YAML로 선언할 수 있다. 애플리케이션처럼 Git에서 Kafka 설정을 관리할 수 있다는 점이 GitOps 환경과 잘 맞았다.

### 1-3) KRaft 기반 Kafka Cluster 생성

이번 실습에서는 ZooKeeper를 별도로 구성하지 않고 KRaft 방식으로 Kafka Cluster를 구성했다.

KRaft는 Kafka가 Cluster Metadata를 자체적으로 관리하는 방식이다. 예전에는 ZooKeeper가 Broker 정보와 Controller 선출 등을 담당했지만, KRaft에서는 Kafka Controller가 이 역할을 직접 수행한다.

Strimzi에서는 KafkaNodePool과 Kafka Custom Resource를 이용해 Node 역할과 Cluster 설정을 정의했다.

```bash
kubectl apply -f kafka-node-pool.yaml
kubectl apply -f kafka-cluster.yaml
```

리소스를 적용한 뒤 Kafka Pod와 관련 Resource를 확인했다.

```bash
kubectl get pods -n kafka
kubectl get kafka -n kafka
kubectl get kafkanodepool -n kafka
```

직접 StatefulSet과 Service를 하나씩 작성하는 대신 Operator가 필요한 Resource를 자동으로 생성하고 상태를 관리해 주었다. Kubernetes에서 Kafka를 운영할 때 Operator를 사용하는 이유를 실습을 통해 확인할 수 있었다.

### 1-4) Kafka Topic 생성

애플리케이션이 사용할 Topic도 `KafkaTopic` Resource로 생성했다.

```bash
kubectl apply -f kafka-topic.yaml
```

생성된 Topic을 확인했다.

```bash
kubectl get kafkatopic -n kafka
```

Topic 설정에는 Partition 수와 Replication Factor가 포함된다. Partition 수는 동시에 처리할 수 있는 병렬성에 영향을 주고, Replication Factor는 Broker 장애 시 데이터 가용성에 영향을 준다.

실습 환경에서는 작은 값으로 구성했지만, 실제 운영 환경에서는 예상 메시지 처리량과 Broker 수를 기준으로 값을 결정해야 한다.

### 1-5) Producer와 Consumer 적용

Kafka Cluster와 Topic을 구성한 뒤 애플리케이션에 Producer와 Consumer 코드를 추가했다.

Producer는 API 요청을 받아 Kafka Topic에 이벤트를 전송하고, Consumer는 해당 Topic을 구독해 메시지를 처리했다.

```text
API 요청 수신
→ Producer가 Kafka에 메시지 전송
→ Kafka가 Topic에 메시지 저장
→ Consumer가 메시지 수신
→ 후속 작업 실행
```

애플리케이션을 빌드한 뒤 새로운 이미지 태그를 적용하고 Git 저장소의 Manifest를 수정했다.

```bash
git add .
git commit -m "Add Kafka event processing"
git push
```

GitOps 환경에서 변경 사항이 반영된 뒤 Pod 상태와 로그를 확인했다.

```bash
kubectl get pods
kubectl logs <producer-pod>
kubectl logs <consumer-pod>
```

Producer 로그에서 메시지가 전송되는지 확인하고, Consumer 로그에서 동일한 메시지가 처리되는지 확인했다.

이번 실습을 통해 API와 실제 처리 로직을 분리하면 각 구성 요소를 독립적으로 확장할 수 있다는 점을 알게 되었다. 요청이 많아질 경우 API Pod와 Consumer Pod의 Replica를 각각 다르게 조정할 수 있고, Consumer에 일시적인 문제가 생기더라도 Kafka에 메시지가 남아 있어 복구 후 다시 처리할 수 있다.

---

## 2. Tempo와 OpenTelemetry를 이용한 분산 추적

애플리케이션이 여러 서비스로 분리되면 하나의 요청이 어느 서비스를 거쳐 처리되었는지 확인하기 어려워진다.

일반 로그만으로도 각 서비스의 상태를 확인할 수 있지만, 서로 다른 Pod에서 생성된 로그가 하나의 요청과 관련된 것인지 직접 연결해야 한다. 분산 추적은 하나의 요청에 Trace ID를 부여하고 요청이 지나간 각 구간을 Span으로 기록해 전체 처리 흐름을 보여준다.

```text
하나의 Trace

API Span
  ↓
Kafka Producer Span
  ↓
Kafka Consumer Span
  ↓
Database Span
```

### 2-1) Trace와 Span

Trace는 하나의 요청 전체 흐름을 의미하고, Span은 그 안에서 수행된 개별 작업 단위를 의미한다.

예를 들어 API 요청이 들어와 Kafka에 메시지를 보내고 Consumer가 이를 처리한다면 각 단계가 하나의 Span이 된다. 여러 Span은 동일한 Trace ID로 연결된다.

```text
Trace ID: abc123

Span 1: API 요청 처리
Span 2: Kafka 메시지 전송
Span 3: Consumer 메시지 수신
Span 4: 데이터 처리
```

실습 전에는 분산 추적이 단순히 로그를 보기 좋게 연결하는 기능이라고 생각했지만, 실제로는 각 구간의 처리 시간과 부모·자식 관계를 확인해 병목 지점을 찾는 데 더 큰 의미가 있었다.

### 2-2) Tempo 설치

이번 실습에서는 Trace 저장소로 Grafana Tempo를 사용했다.

Tempo는 Trace 데이터를 저장하고 조회하는 역할을 하며, Grafana와 연동해 Trace를 시각화할 수 있다.

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install tempo grafana/tempo   --namespace monitoring   --create-namespace
```

설치 상태를 확인했다.

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Tempo는 Jaeger나 Zipkin과 비슷한 분산 추적 Backend이지만, Grafana 환경과 자연스럽게 연결할 수 있다는 점이 편리했다.

### 2-3) OpenTelemetry 적용

애플리케이션에서 Trace를 생성하고 Tempo로 전달하기 위해 OpenTelemetry를 적용했다.

OpenTelemetry는 Trace, Metric, Log 같은 Telemetry 데이터를 수집하고 전달하기 위한 표준이다. 특정 Vendor에 종속되지 않고 OTLP를 통해 여러 Backend로 데이터를 전송할 수 있다.

```text
Application
  ↓ OpenTelemetry SDK
OTLP
  ↓
Tempo
  ↓
Grafana
```

애플리케이션 설정에 Tempo 또는 OpenTelemetry Collector의 Endpoint를 추가하고, 새 이미지를 배포했다.

```bash
git add .
git commit -m "Add OpenTelemetry tracing"
git push
```

배포 후 실제 요청을 발생시키고 Grafana에서 Trace를 조회했다.

하나의 요청이 여러 Span으로 나뉘어 표시되었고, 각 Span의 소요 시간과 호출 관계를 확인할 수 있었다. 로그만 봤을 때는 어느 구간에서 시간이 지연되었는지 알기 어려웠지만, Trace 화면에서는 지연이 발생한 구간을 바로 확인할 수 있었다.

---

## 3. CronJob을 이용한 주기적인 상태 확인

이번에는 Kubernetes CronJob을 이용해 애플리케이션 상태를 주기적으로 확인하는 작업을 구성했다.

CronJob은 지정된 시간마다 Job을 생성하고, Job은 실제 작업을 수행할 Pod를 실행한다.

```text
CronJob
  ↓ 정해진 시간
Job 생성
  ↓
Pod 실행
  ↓
Health Check 수행
```

### 3-1) Health Check CronJob 생성

CronJob Manifest에는 실행 주기와 사용할 Container Image, 실행 Command 등을 정의했다.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: application-healthcheck
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: healthcheck
              image: curlimages/curl
              args:
                - -f
                - http://application-service/health
```

위 설정은 5분마다 애플리케이션의 Health Endpoint를 호출한다.

```bash
kubectl apply -f healthcheck-cronjob.yaml
```

생성된 CronJob과 Job을 확인했다.

```bash
kubectl get cronjob
kubectl get job
kubectl get pods
```

실행된 Pod의 로그도 확인했다.

```bash
kubectl logs <healthcheck-pod>
```

CronJob은 단순한 스케줄러처럼 보이지만, 실제로는 실행할 때마다 별도의 Job과 Pod를 생성한다. 따라서 이전 실행 이력을 Job 단위로 확인할 수 있고, 실패한 작업도 Pod 로그를 통해 추적할 수 있었다.

### 3-2) CronJob 운영 시 확인할 설정

CronJob에서는 실행 주기 외에도 중복 실행과 실행 이력 관리가 중요하다.

`concurrencyPolicy`를 사용하면 이전 Job이 끝나지 않았을 때 새로운 Job을 실행할지 결정할 수 있다.

```yaml
concurrencyPolicy: Forbid
```

`Forbid`로 설정하면 이전 작업이 실행 중일 때 다음 실행을 건너뛴다. 작업 시간이 길어질 수 있는 경우 중복 실행을 막기 위해 필요한 설정이다.

성공하거나 실패한 Job의 보관 개수도 지정할 수 있다.

```yaml
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 3
```

실습에서는 단순한 Health Check를 수행했지만, 같은 방식으로 정기 백업, 리포트 생성, 임시 데이터 정리 같은 작업도 구성할 수 있다.

---

## 4. Command Guardrail 적용

운영 환경에서는 잘못된 명령 한 번이 서비스 장애로 이어질 수 있다. 특히 Kubernetes Resource 삭제나 Namespace 삭제 같은 작업은 실행 즉시 영향이 발생한다.

이번 실습에서는 Command Guardrail을 적용해 위험한 명령을 실행하기 전에 추가 확인 절차를 거치도록 구성했다.

```text
사용자 명령 입력
  ↓
위험 명령 여부 확인
  ↓
승인 또는 차단
  ↓
명령 실행
```

Guardrail은 사용자의 모든 명령을 막는 것이 아니라, 삭제나 강제 변경처럼 위험도가 높은 작업을 구분해 실수를 줄이는 안전장치다.

처음에는 명령 실행 전에 확인 단계가 추가되는 것이 번거롭게 느껴질 수 있다고 생각했다. 하지만 운영 환경에서는 편의성보다 잘못된 명령을 예방하는 것이 더 중요하며, 특히 여러 사용자가 같은 Cluster를 관리할 때 유용할 수 있다는 생각이 들었다.

GitOps에서도 비슷한 안전장치가 존재한다. 운영 Cluster에 직접 명령을 실행하는 대신 Git Pull Request와 Review를 통해 변경 내용을 확인한 뒤 반영하면, 누가 무엇을 변경했는지 기록을 남길 수 있다.

```text
직접 kubectl 수정
→ 즉시 반영되지만 검토와 이력 관리가 어려움

GitOps 변경
→ Git Commit과 Review를 거쳐 반영
→ 변경 이력과 롤백 기준이 명확함
```

이번 Guardrail 실습은 단순히 특정 명령을 차단하는 기능을 넘어, 운영 자동화에는 반드시 통제 절차가 함께 필요하다는 점을 보여주었다.

---

## 5. Chapter 8 실습을 마치며

이번 챕터에서는 애플리케이션에 새로운 기능을 추가하는 것보다, 실제 운영 환경에서 서비스를 안정적으로 유지하기 위한 구조를 만드는 과정이 중심이었다.

Kafka 실습을 통해 API 요청 처리와 실제 작업을 분리하는 이벤트 기반 구조를 확인했다. Producer와 Consumer를 분리하면 처리 속도가 다른 구성 요소를 독립적으로 확장할 수 있고, Consumer에 문제가 생겨도 Kafka에 남아 있는 메시지를 다시 처리할 수 있었다.

Tempo와 OpenTelemetry를 적용하면서 로그만으로는 확인하기 어려운 서비스 간 요청 흐름을 Trace와 Span으로 연결해 볼 수 있었다. 특히 여러 서비스와 Kafka가 연결된 구조에서는 단순히 Pod 로그를 각각 확인하는 것보다 하나의 Trace ID를 기준으로 전체 흐름을 확인하는 것이 훨씬 효율적이었다.

CronJob 실습에서는 Kubernetes가 일반적인 애플리케이션뿐 아니라 주기적으로 실행되는 운영 작업도 Resource로 관리할 수 있다는 점을 확인했다. 실행 주기, 재시작 정책, 중복 실행 정책, 작업 이력까지 YAML로 관리할 수 있어 운영 작업도 GitOps 방식으로 관리할 수 있었다.

마지막 Guardrail 실습을 통해 자동화가 많아질수록 실행 권한과 안전장치도 중요해진다는 점을 알게 되었다. 명령을 빠르게 실행하는 것만이 좋은 운영 방식은 아니며, 위험한 작업은 검토와 확인 절차를 거치도록 만드는 것이 안정적인 운영에 더 가깝다.

이번 챕터를 진행하면서 Kubernetes 환경에서의 운영은 단순히 Pod를 실행하고 상태를 확인하는 수준에서 끝나지 않는다는 점을 다시 느꼈다. 메시지 처리, 요청 추적, 주기 작업, 변경 통제까지 함께 구성해야 실제 서비스 운영에 가까운 환경이 만들어진다.