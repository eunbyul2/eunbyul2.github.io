---
layout: post
title: "05. RabbitMQ, Kafka, Redis를 함께 쓰는 실제 아키텍처"
date: 2026-07-05 15:00:00 +0900
categories: [스터디, 인프라, Architecture, Middleware, PlatformEngineering]
tags: [RabbitMQ, Kafka, Redis, Architecture, QnA, EventDrivenArchitecture, MSA]
published: true
---

> RabbitMQ, Kafka, Redis는 서로 경쟁 제품처럼 보이지만 실제 서비스에서는 함께 사용되는 경우가 많다. 이 문서는 실제 아키텍처 예시와 발표 질의응답 대비용 비교 질문을 정리한다.

## 1. 세 기술을 함께 쓰는 이유

RabbitMQ, Kafka, Redis는 각각 해결하는 문제가 다르다. 그래서 실무에서는 하나만 선택하는 것이 아니라, 역할을 나눠 함께 사용하는 경우가 많다.

```text
Redis   → 빠른 조회, 캐시, 세션, 임시 상태
RabbitMQ → 작업 큐, 비동기 후처리, 안정적 메시지 전달
Kafka   → 이벤트 로그, 데이터 파이프라인, 실시간 분석
```

예를 들어 이커머스 서비스에서 주문이 발생하면 다음과 같은 흐름을 만들 수 있다.

```text
User
  ↓
Ingress / API Gateway
  ↓
Backend API
  ├─ Redis 조회
  ├─ Database 저장
  ├─ RabbitMQ에 후처리 작업 등록
  └─ Kafka에 OrderCreated 이벤트 발행
```

Redis는 API 응답 속도를 높이기 위해 DB 앞에 위치한다. RabbitMQ는 이메일 발송, 재고 후처리, 결제 후처리 같은 작업을 Worker에게 안정적으로 전달한다. Kafka는 주문 이벤트를 분석, 추천, 로그, 데이터레이크 적재 시스템이 독립적으로 읽을 수 있게 한다.

## 2. 예시 아키텍처: 회원가입

회원가입 기능을 예로 들면 다음과 같다.

```text
Client
  ↓
API Server
  ├─ 사용자 정보 DB 저장
  ├─ Redis에 인증/세션 정보 저장
  ├─ RabbitMQ에 Welcome Email 작업 등록
  └─ Kafka에 UserRegistered 이벤트 발행
```

Redis는 로그인 세션이나 인증 토큰을 저장한다. TTL을 설정해 일정 시간이 지나면 자동 만료되게 할 수 있다.

RabbitMQ는 환영 이메일 발송 작업을 큐에 넣는다. 이메일 서버가 느리더라도 회원가입 API 응답은 빠르게 반환된다. Worker가 죽으면 메시지는 재전달될 수 있고, 계속 실패하면 DLQ에 보관할 수 있다.

Kafka는 UserRegistered 이벤트를 저장한다. 추천 시스템, CRM, 분석 시스템, 데이터 웨어하우스 적재 시스템이 이 이벤트를 각자 독립적으로 읽을 수 있다. 나중에 새로운 분석 시스템이 추가되더라도 Kafka에 저장된 이벤트를 읽어 초기 데이터를 만들 수 있다.

## 3. 예시 아키텍처: 주문 처리

주문 처리에서는 동기와 비동기를 구분해야 한다.

결제 승인, 주문 생성, 재고 선점처럼 반드시 즉시 성공 여부를 알아야 하는 작업은 동기로 처리해야 한다. 반면 이메일 발송, 알림, 로그 저장, 추천 이벤트 전달은 비동기로 처리할 수 있다.

```text
Order API
  ↓
동기 처리
  ├─ 결제 승인
  ├─ 주문 DB 저장
  └─ 재고 선점

비동기 처리
  ├─ RabbitMQ: 이메일/알림/송장 생성 작업
  └─ Kafka: OrderCreated 이벤트 스트리밍
```

여기서 RabbitMQ와 Kafka를 둘 다 쓰는 이유가 중요하다.

RabbitMQ는 특정 작업을 워커가 반드시 처리해야 하는 경우에 사용한다. 예를 들어 송장 생성, 알림 발송, 정산 작업은 실패하면 재시도하고, 계속 실패하면 DLQ에서 확인해야 한다.

Kafka는 주문이 생성되었다는 사실을 여러 시스템이 독립적으로 소비해야 할 때 사용한다. 통계 시스템은 주문 이벤트를 집계하고, 추천 시스템은 사용자 행동 모델에 반영하고, 데이터레이크는 원천 이벤트를 저장한다.

## 4. 예시 아키텍처: 로그와 모니터링

로그 수집 구조에서는 Kafka가 자주 사용된다.

```text
Application Logs
  ↓
Fluent Bit / Log Agent
  ↓
Kafka Topic
  ├─ Elasticsearch / OpenSearch
  ├─ Data Lake
  ├─ Real-time Alerting
  └─ Analytics Pipeline
```

로그는 대량으로 발생하고, 여러 시스템이 동시에 필요로 한다. 또한 장애가 발생했을 때 특정 지점부터 다시 처리해야 할 수 있다. 이런 경우 Kafka의 Topic, Partition, Retention, Offset 구조가 유리하다.

RabbitMQ도 로그 전달에 사용할 수는 있지만, 로그를 장기간 보존하고 여러 시스템이 재처리하는 구조에는 Kafka가 더 자연스럽다.

Redis는 이 구조에서 실시간 대시보드의 임시 집계값, Rate Limit 카운터, 최근 이벤트 캐시 등에 사용될 수 있다.

## 5. 예시 아키텍처: AI/ML 플랫폼

AI 플랫폼에서도 RabbitMQ, Kafka, Redis는 모두 등장할 수 있다.

```text
사용자
  ↓
Model Serving API
  ├─ Redis: 모델 메타데이터 캐시, 세션, Rate Limit
  ├─ RabbitMQ: 비동기 추론 요청, 배치 작업 큐
  └─ Kafka: 추론 로그, 피드백 이벤트, 모니터링 이벤트
```

Redis는 모델 메타데이터, 사용자 세션, API Rate Limit, feature cache에 사용할 수 있다.

RabbitMQ는 시간이 오래 걸리는 배치 추론, 이미지/영상 전처리, 리포트 생성 작업을 워커에게 분산하는 데 사용할 수 있다.

Kafka는 추론 요청/응답 로그, 사용자 피드백 이벤트, 모델 성능 모니터링 이벤트, 데이터 드리프트 분석 이벤트를 수집하는 데 사용할 수 있다.

이 구조는 AI Platform Engineer 관점에서도 중요하다. 모델을 배포하는 것뿐 아니라, 추론 요청을 어떻게 안정적으로 처리하고, 로그를 어떻게 수집하며, 장애와 지연을 어떻게 관측할지 설계해야 하기 때문이다.

## 6. 선택 기준 시나리오

### 6-1) 이메일 발송

이메일 발송은 RabbitMQ가 적합하다. 이메일 발송은 작업 큐 성격이 강하다. API 서버는 이메일 발송 요청을 큐에 넣고, Worker가 가져가서 처리하면 된다. 실패하면 재시도하고, 계속 실패하면 DLQ로 보낸다.

Redis로도 간단히 처리할 수 있지만, 메시지 유실이 치명적이거나 라우팅과 재시도 정책이 필요하면 RabbitMQ가 더 안정적이다.

Kafka는 이메일 발송 작업 자체에는 과할 수 있다. 다만 이메일 발송 이벤트를 분석하거나 로그로 남기기 위해 Kafka에 이벤트를 발행할 수는 있다.

### 6-2) 클릭 로그 수집

클릭 로그 수집은 Kafka가 적합하다. 클릭 이벤트는 대량으로 발생하고, 분석 시스템, 추천 시스템, 광고 시스템, 데이터레이크가 모두 같은 데이터를 필요로 할 수 있다.

Kafka는 이벤트를 저장하고 여러 Consumer Group이 독립적으로 읽을 수 있으므로 이 구조에 잘 맞는다.

RabbitMQ는 클릭 로그처럼 초고속 대량 스트리밍과 장기 재처리가 필요한 구조에는 상대적으로 덜 적합하다.

Redis는 최근 클릭 이벤트를 임시로 저장하거나 실시간 카운터를 계산하는 데 사용할 수 있지만, 원천 로그 저장소로는 적합하지 않다.

### 6-3) 세션 저장

세션 저장은 Redis가 적합하다. 세션은 빠른 조회와 TTL이 중요하다. Redis는 메모리 기반이라 지연 시간이 낮고, TTL로 세션 만료를 자연스럽게 처리할 수 있다.

RabbitMQ와 Kafka는 세션 저장소가 아니다. 메시지 전달이나 이벤트 스트리밍 도구를 세션 저장에 사용하는 것은 목적에 맞지 않는다.

### 6-4) 주문 이벤트 기반 마이크로서비스

주문 이벤트를 여러 마이크로서비스가 독립적으로 처리해야 한다면 Kafka가 적합하다.

```text
OrderCreated
  ├─ Payment Service
  ├─ Delivery Service
  ├─ Recommendation Service
  ├─ Analytics Service
  └─ Data Lake Sink
```

다만 특정 후처리 작업이 반드시 실행되어야 하고 실패 시 DLQ 관리가 중요하다면 RabbitMQ를 함께 사용할 수 있다.

## 7. RabbitMQ 대신 Kafka를 쓰면 안 되나요?

쓸 수는 있지만 목적이 다르다. Kafka도 Consumer Group을 사용하면 작업 큐처럼 사용할 수 있다. 하지만 Kafka는 이벤트를 로그로 저장하고 여러 소비자가 독립적으로 읽는 데 강점이 있다. 단순 작업 큐, 복잡한 라우팅, ACK 기반 처리 완료 관리, DLQ 중심 운영에는 RabbitMQ가 더 직관적일 수 있다.

정리하면 다음과 같다.

```text
작업을 처리하고 끝낼 목적이면 RabbitMQ,
이벤트를 저장하고 여러 시스템이 읽게 할 목적이면 Kafka.
```

## 8. Kafka 대신 RabbitMQ를 쓰면 안 되나요?

작은 규모에서는 가능할 수 있다. 하지만 대량 이벤트 스트리밍, 장기 보존, replay, Consumer Group별 독립 소비가 필요하면 Kafka가 더 적합하다.

RabbitMQ는 메시지가 처리되면 제거되는 Queue 모델에 가깝다. Kafka는 메시지를 로그에 저장하고 Consumer별 Offset을 관리한다. 따라서 새로운 Consumer가 과거 이벤트를 읽어야 하거나, 장애 후 특정 지점부터 재처리해야 한다면 Kafka가 유리하다.

## 9. Redis Pub/Sub와 Kafka는 어떻게 다른가?

Redis Pub/Sub는 현재 연결된 구독자에게 빠르게 메시지를 전달하는 기능이다. 메시지를 저장하지 않기 때문에 구독자가 없거나 연결이 끊겨 있으면 메시지를 놓칠 수 있다.

Kafka는 이벤트를 Topic에 저장한다. Consumer가 늦게 읽어도 retention 안에 있으면 읽을 수 있고, Consumer Group별로 Offset을 따로 관리한다.

```text
Redis Pub/Sub = 빠른 실시간 전달, 저장 없음, 재처리 약함
Kafka = 저장되는 이벤트 로그, 재처리 가능, 다중 소비 가능
```

## 10. Redis Streams와 Kafka는 어떻게 다른가?

Redis Streams는 메시지를 저장하고 Consumer Group을 제공하므로 Pub/Sub보다 Kafka에 가깝다. 하지만 Kafka는 대규모 분산 로그, Partition 기반 병렬 처리, 장기 보존, 데이터 파이프라인 생태계를 위해 설계된 플랫폼이다.

Redis Streams는 Redis 안에서 가볍게 스트림 처리를 구현할 때 적합하다. 이미 Redis를 운영 중이고 규모가 크지 않으며 빠른 구현이 중요하다면 Redis Streams가 유용할 수 있다. 하지만 회사 전체 데이터 파이프라인, 대규모 로그 수집, 여러 팀의 독립 소비 구조라면 Kafka가 더 적합하다.

## 11. RabbitMQ는 Push고 Kafka는 Pull인가?

일반적으로 RabbitMQ는 Consumer에게 메시지를 전달하는 push 모델에 가깝고, Kafka는 Consumer가 Broker에서 데이터를 가져가는 pull 모델에 가깝다.

RabbitMQ에서는 Broker가 Queue와 Consumer 상태를 보고 메시지를 전달한다. Consumer는 prefetch 설정을 통해 한 번에 받을 메시지 수를 조절할 수 있다.

Kafka에서는 Consumer가 자신이 읽을 Partition과 Offset을 기준으로 데이터를 가져온다. Consumer가 처리 속도를 조절하고, Offset을 관리한다. 이 때문에 Kafka에서는 Consumer 설계와 Consumer Lag 관리가 중요하다.

## 12. 메시지는 한 번만 처리되나요?

분산 시스템에서 "정확히 한 번만 처리"는 매우 어렵다. 대부분의 메시징 시스템은 at-most-once, at-least-once, exactly-once 같은 전달 의미론을 제공하거나 구성할 수 있다.

At-most-once는 최대 한 번 전달이다. 중복은 없지만 유실될 수 있다. At-least-once는 최소 한 번 전달이다. 유실 가능성을 줄이지만 중복 처리될 수 있다. Exactly-once는 정확히 한 번 처리처럼 보이게 하는 모델이지만, 실제로는 브로커, Consumer, DB 트랜잭션, idempotency 설계가 함께 맞아야 한다.

실무에서는 at-least-once + idempotency 조합이 많이 사용된다. 즉, 메시지가 중복 전달될 수 있다고 가정하고, 같은 작업이 여러 번 실행되어도 결과가 달라지지 않게 설계한다.

## 13. Kubernetes에서는 왜 StatefulSet을 쓰나요?

RabbitMQ, Kafka, Redis는 상태를 가진 미들웨어다. Pod가 재시작되어도 데이터와 식별자가 유지되어야 한다.

StatefulSet은 다음을 제공한다.

- 안정적인 Pod 이름
- 순서 있는 생성/삭제
- Pod별 고정 PVC
- Headless Service와 함께 안정적인 DNS 제공

Kafka Broker ID, RabbitMQ cluster node name, Redis Cluster node identity처럼 안정적인 식별자가 필요한 시스템에서는 StatefulSet이 적합하다.

## 14. PVC는 왜 필요한가요?

PVC는 Pod가 재시작되어도 데이터를 유지하기 위해 필요하다.

RabbitMQ는 persistent message와 durable queue를 사용할 때 메시지를 디스크에 저장한다. Kafka는 이벤트 로그를 디스크에 저장한다. Redis는 RDB/AOF를 사용할 때 디스크에 저장한다.

PVC가 없으면 Pod가 재생성될 때 데이터가 사라질 수 있다. 물론 Redis를 순수 캐시로만 사용한다면 데이터가 사라져도 원본 DB에서 다시 만들 수 있으므로 PVC 없이 운영할 수도 있다. 하지만 Kafka와 RabbitMQ처럼 메시지/이벤트 유실이 문제가 되는 시스템에서는 저장소 설계가 중요하다.

## 15. 어떤 지표를 모니터링해야 하나요?

RabbitMQ는 Queue depth, unacked messages, publish/deliver/ack rate, consumer count, memory alarm, disk alarm을 본다.

Kafka는 consumer lag, under-replicated partitions, offline partitions, broker disk usage, request latency, ISR 상태, rebalance 빈도를 본다.

Redis는 memory usage, eviction count, keyspace hit/miss, blocked clients, replication lag, expired keys, connected clients를 본다.

이 지표들은 단순 상태 확인이 아니라 장애 원인 분석의 출발점이다. 예를 들어 Kafka consumer lag이 증가하면 Consumer 처리량, Partition 수, DB 병목, Broker 성능, Rebalance 여부를 차례로 확인해야 한다.

## 16. 정리

```text
RabbitMQ는 작업을 안전하게 전달하기 위한 메시지 브로커다.
Kafka는 이벤트를 로그처럼 저장하고 여러 시스템이 독립적으로 읽게 하는 스트리밍 플랫폼이다.
Redis는 메모리 기반의 빠른 데이터 저장소로 캐시, 세션, 락, 간단한 메시징에 사용된다.
```

그리고 선택 기준은 다음과 같다.

```text
비동기 작업 처리와 신뢰성 있는 메시지 전달 → RabbitMQ
대량 이벤트 수집, 재처리, 실시간 데이터 파이프라인 → Kafka
초고속 캐시, 세션, TTL, 실시간 상태 저장 → Redis
```

인프라 및 플랫폼 엔지니어 관점에서는 제품 기능만 보는 것이 아니라, Kubernetes 배포 방식, 저장소, HA, 장애 복구, 모니터링, 백업, 운영 자동화까지 함께 봐야 한다.
