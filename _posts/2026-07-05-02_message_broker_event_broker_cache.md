---
layout: post
title: "02. Message Queue, Event Broker, Cache 개념 정리"
date: 2026-07-05 11:50:00 +0900
categories: [Infra, Middleware, Messaging]
tags: [MessageQueue, EventBroker, Cache, PubSub, RabbitMQ, Kafka, Redis]
published: true
---

> RabbitMQ, Kafka, Redis를 제대로 비교하려면 먼저 Queue, Broker, Producer, Consumer, Topic, Offset, Cache 같은 공통 개념을 이해해야 한다.

## 1. Producer, Broker, Consumer

메시징 시스템의 가장 기본적인 구성 요소는 Producer, Broker, Consumer다.

```text
Producer
  ↓
Broker
  ↓
Consumer
```

Producer는 메시지나 이벤트를 생성해서 브로커에 보내는 주체다. 일반적으로 API 서버, 백엔드 서비스, 배치 애플리케이션, 로그 에이전트가 Producer가 된다.

Broker는 Producer와 Consumer 사이에서 메시지를 받아 저장하고 전달하는 중간 서버다. RabbitMQ, Kafka, Redis가 이 위치에 들어갈 수 있다. 다만 세 제품의 저장 방식과 전달 방식은 서로 다르다.

Consumer는 Broker로부터 메시지나 이벤트를 읽어 실제 작업을 처리하는 주체다. 이메일 발송 워커, 이미지 리사이징 워커, 로그 처리 애플리케이션, 분석 시스템, 검색 색인 시스템 등이 Consumer가 될 수 있다.

이 구조의 핵심은 송신자와 수신자를 직접 연결하지 않는 것이다. Producer는 Consumer가 살아 있는지, 몇 개인지, 지금 바쁜지 몰라도 된다. Consumer도 Producer와 직접 통신하지 않고 브로커에서 필요한 메시지를 가져가면 된다. 이 특성을 느슨한 결합이라고 한다.

## 2. Queue란 무엇인가

Queue는 먼저 들어온 데이터가 먼저 나가는 FIFO(First In, First Out) 구조다.

```text
Queue
[1번 작업] → [2번 작업] → [3번 작업]
```

작업 큐에서는 Producer가 작업을 큐에 넣고, Worker가 큐에서 하나씩 꺼내 처리한다.

```text
API Server
  ↓
Queue
  ↓
Worker 1
Worker 2
Worker 3
```

Worker를 여러 개 두면 같은 Queue에서 작업을 나눠 가져가므로 수평 확장이 가능하다. 예를 들어 이미지 처리 작업이 10만 개 쌓여 있다면 Worker Pod 수를 늘려 처리량을 높일 수 있다.

Queue는 다음 상황에 적합하다.

- 이메일, SMS, 푸시 알림 발송
- 이미지 리사이징
- 영상 인코딩
- 주문 후처리
- 결제 후처리
- 외부 API 호출
- 배치 작업 분산 처리

Queue를 사용할 때 중요한 점은 메시지가 중복 처리될 수 있다는 점이다. Worker가 메시지를 처리하다가 죽으면 같은 메시지가 다시 전달될 수 있다. 따라서 결제, 포인트 지급, 주문 상태 변경 같은 작업은 멱등성(idempotency)을 고려해야 한다.

## 3. ACK와 메시지 신뢰성

ACK는 Acknowledgement의 약자로, Consumer가 브로커에게 "이 메시지를 정상적으로 처리했다"고 알려주는 신호다.

```text
Broker → Consumer : 메시지 전달
Consumer : 작업 처리
Consumer → Broker : ACK 전송
Broker : 메시지 삭제 또는 처리 완료 처리
```

ACK가 중요한 이유는 Worker 장애 때문이다. Worker가 메시지를 가져간 직후 죽으면 어떻게 해야 할까? 브로커가 메시지를 이미 삭제했다면 작업은 유실된다. 반대로 ACK를 받을 때까지 메시지를 보관하면 Worker가 죽어도 다른 Worker에게 다시 전달할 수 있다.

RabbitMQ는 이 ACK 기반 메시지 신뢰성을 강하게 지원한다. Consumer가 ACK를 보내기 전까지 메시지를 처리 완료로 보지 않을 수 있고, Consumer가 죽으면 메시지를 다시 큐에 넣어 다른 Consumer에게 전달할 수 있다.

Redis를 단순 List 기반 Queue로 사용할 경우에는 Worker가 메시지를 꺼내는 순간 List에서 사라질 수 있다. 이 경우 처리 중 Worker가 죽으면 메시지 유실 위험이 있다. Redis Streams나 Celery 설정을 강화하면 일부 보완할 수 있지만, RabbitMQ처럼 메시지 브로커로 설계된 시스템과는 기본 철학이 다르다.

## 4. Retry와 Dead Letter Queue

비동기 작업은 항상 성공하지 않는다. 외부 API가 일시적으로 장애일 수 있고, 네트워크 타임아웃이 발생할 수 있으며, Consumer 코드에 버그가 있을 수도 있다.

실무에서는 실패한 작업을 바로 버리지 않고 재시도한다.

```text
1차 실패
  ↓
1초 후 재시도
  ↓
2초 후 재시도
  ↓
4초 후 재시도
  ↓
계속 실패하면 DLQ로 이동
```

재시도 간격을 점점 늘리는 방식을 exponential backoff라고 한다. 외부 시스템이 회복될 시간을 주기 위한 방식이다.

Dead Letter Queue(DLQ)는 계속 실패한 메시지를 따로 보관하는 큐다. DLQ에 들어간 메시지는 자동 처리 대상에서 제외하고, 운영자나 개발자가 원인을 확인한 뒤 수정하고 재처리한다.

DLQ는 운영 관점에서 매우 중요하다. 실패한 메시지를 그냥 버리면 데이터 정합성이 깨질 수 있고, 무한 재시도하면 시스템 리소스를 계속 낭비한다. DLQ는 실패 메시지를 격리하고, 장애 분석과 재처리를 가능하게 한다.

## 5. Event와 Event Streaming

Queue가 "작업을 처리하기 위한 줄"이라면, Event Streaming은 "시스템에서 발생한 이벤트를 시간 순서대로 저장한 로그"에 가깝다.

이벤트는 이미 발생한 사실이다.

```text
UserRegistered
OrderCreated
PaymentCompleted
ProductViewed
LoginFailed
```

Kafka에서는 이런 이벤트를 Topic에 저장한다.

```text
Topic: order-events
  offset 0: OrderCreated(order_id=1)
  offset 1: PaymentCompleted(order_id=1)
  offset 2: OrderCancelled(order_id=2)
```

Kafka의 중요한 특징은 Consumer가 메시지를 읽어도 즉시 삭제되지 않는다는 점이다. 이벤트는 설정된 retention 기간이나 용량까지 유지된다. Consumer는 자신이 어디까지 읽었는지를 offset으로 관리한다.

이 구조 덕분에 Kafka에서는 다음이 가능하다.

- 장애가 발생한 지점부터 다시 읽기
- 새로운 Consumer가 과거 이벤트부터 읽기
- 여러 시스템이 같은 이벤트를 독립적으로 소비하기
- 로그 기반 데이터 파이프라인 구성하기
- 실시간 분석과 데이터 레이크 적재를 동시에 처리하기

## 6. Topic, Partition, Offset

Kafka를 이해하려면 Topic, Partition, Offset을 반드시 알아야 한다.

### 6-1) Topic

Topic은 이벤트를 분류하는 논리적 단위다.

```text
user-events
order-events
payment-events
click-events
```

Producer는 특정 Topic에 이벤트를 기록하고, Consumer는 관심 있는 Topic을 구독해서 읽는다.

### 6-2) Partition

Partition은 Topic을 물리적으로 나눈 단위다.

```text
Topic: order-events
  Partition 0
  Partition 1
  Partition 2
```

Partition을 여러 개로 나누면 여러 Broker에 분산 저장할 수 있고, Consumer Group 안의 Consumer들이 병렬로 읽을 수 있다. Kafka의 높은 처리량은 Partition 기반 병렬 처리에서 나온다.

단, 순서 보장은 Topic 전체가 아니라 Partition 내부에서 보장된다. 주문 이벤트의 순서를 지켜야 한다면 같은 주문 ID가 같은 Partition으로 들어가도록 Key 설계를 해야 한다.

### 6-3) Offset

Offset은 Partition 안에서 각 이벤트가 가진 순번이다.

```text
Partition 0
  offset 0
  offset 1
  offset 2
```

Consumer는 자신이 어디까지 읽었는지를 offset으로 저장한다. 장애가 발생하면 마지막 commit된 offset부터 다시 읽을 수 있다. 이 구조가 Kafka의 재처리와 replay를 가능하게 한다.

## 7. Pub/Sub란 무엇인가

Pub/Sub는 Publish/Subscribe의 약자다. 발행자는 특정 채널이나 Topic에 메시지를 발행하고, 구독자는 해당 채널이나 Topic을 구독해서 메시지를 받는다.

```text
Publisher
  ↓
Channel / Topic
  ├─ Subscriber A
  ├─ Subscriber B
  └─ Subscriber C
```

Redis Pub/Sub, RabbitMQ Fanout Exchange, Kafka Consumer Group은 모두 Pub/Sub처럼 보일 수 있다. 하지만 내부 동작은 다르다.

Redis Pub/Sub는 매우 빠르고 단순하지만, 구독자가 연결되어 있지 않으면 메시지를 놓칠 수 있다. 기본 Pub/Sub 메시지는 저장되지 않는다.

RabbitMQ의 Fanout Exchange는 연결된 여러 Queue로 메시지를 복사해 전달할 수 있다. 각 Queue가 메시지를 보관하므로 Consumer가 잠시 느려도 Queue에 메시지가 쌓일 수 있다.

Kafka는 Topic에 이벤트를 저장하고, 여러 Consumer Group이 독립적으로 offset을 관리한다. 메시지를 소비해도 바로 삭제되지 않으므로 재처리와 다중 소비에 강하다.

## 8. Cache란 무엇인가

Cache는 자주 사용하는 데이터를 빠른 저장소에 임시로 저장하는 구조다. Redis가 대표적인 캐시 저장소다.

```text
API Server
  ↓
Redis Cache
  ↓ Cache Miss
Database
```

캐시의 목적은 DB 부하를 줄이고 응답 속도를 높이는 것이다. 예를 들어 상품 상세 정보, 사용자 세션, 인증 토큰, 인기 게시글 목록, API 응답 결과처럼 자주 조회되는 데이터를 Redis에 저장할 수 있다.

캐시를 사용할 때는 Cache Hit와 Cache Miss를 이해해야 한다.

Cache Hit는 요청한 데이터가 Redis에 있는 경우다. API 서버는 DB에 가지 않고 Redis에서 바로 응답한다. Cache Miss는 Redis에 데이터가 없는 경우다. 이때 API 서버는 DB에서 데이터를 조회하고, 결과를 Redis에 저장한 뒤 응답한다.

## 9. TTL과 Eviction

TTL(Time To Live)은 데이터가 살아 있는 시간을 의미한다. Redis에서는 키마다 TTL을 설정할 수 있다.

```text
session:user:1001 → 30분 후 만료
email:code:abc123 → 5분 후 만료
product:detail:88 → 10분 후 만료
```

TTL은 캐시 데이터가 무한정 쌓이는 것을 방지하고, 오래된 데이터가 계속 사용되는 것을 줄인다.

Eviction은 Redis 메모리가 가득 찼을 때 어떤 데이터를 제거할지 결정하는 정책이다. Redis는 메모리 기반이기 때문에 용량 관리가 중요하다. 메모리 사용량이 한계를 넘으면 쓰기 실패가 발생하거나, 설정된 정책에 따라 일부 키가 제거될 수 있다.

인프라 운영자는 Redis를 단순히 빠른 저장소로만 보면 안 된다. 메모리 사용량, maxmemory, eviction policy, key TTL, persistence 설정을 함께 봐야 한다.

## 10. Persistence와 Durability

Persistence는 데이터를 디스크에 저장해 재시작 후에도 복구할 수 있게 하는 기능이다. Durability는 장애가 발생해도 데이터가 안전하게 유지되는 성질이다.

RabbitMQ는 메시지를 durable queue와 persistent message로 설정하면 디스크에 저장할 수 있다. 다만 모든 메시지를 디스크에 강하게 저장하면 성능과 지연 시간에 영향을 줄 수 있다.

Kafka는 기본적으로 이벤트를 디스크 로그에 저장한다. Replication을 통해 여러 Broker에 복제하고, retention 정책에 따라 일정 기간 보관한다. Kafka는 로그 기반 저장 구조와 순차 I/O를 활용해 높은 처리량을 낸다.

Redis는 기본적으로 메모리 기반이지만 RDB snapshot과 AOF(Append Only File)로 영속화를 제공한다. RDB는 특정 시점 스냅샷이고, AOF는 쓰기 명령을 로그로 기록하는 방식이다. Redis를 캐시로만 사용한다면 데이터가 사라져도 DB에서 다시 만들 수 있지만, 세션이나 중요한 상태를 저장한다면 persistence와 replication을 반드시 고려해야 한다.

## 11. Backpressure와 Consumer Lag

Backpressure는 생산 속도가 소비 속도보다 빠를 때 시스템이 압력을 받는 현상이다.

예를 들어 API 서버가 초당 10,000개의 메시지를 넣는데 Worker가 초당 1,000개만 처리하면 Queue가 계속 쌓인다. 이 상태가 오래 지속되면 브로커 메모리나 디스크가 가득 차고, 지연 시간이 증가하며, 결국 장애로 이어질 수 있다.

Kafka에서는 Consumer가 Producer보다 느리면 consumer lag이 증가한다. Consumer lag은 Consumer가 현재 최신 offset보다 얼마나 뒤처져 있는지를 나타내는 지표다. 실시간 처리 시스템에서는 consumer lag이 핵심 모니터링 지표다.

RabbitMQ에서는 queue depth, unacked messages, ready messages, publish rate, deliver rate를 본다. Redis에서는 memory usage, blocked clients, stream pending entries, keyspace hits/misses 등을 본다.

## 12. 정리

RabbitMQ, Kafka, Redis를 비교하기 전에 반드시 알아야 하는 공통 개념은 다음과 같다.

- Producer: 메시지나 이벤트를 생성하는 주체
- Broker: 메시지를 받아 저장하고 전달하는 중간 서버
- Consumer: 메시지나 이벤트를 읽어 처리하는 주체
- Queue: 작업을 줄 세워 처리하는 구조
- ACK: Consumer가 처리 완료를 알리는 신호
- Retry: 실패한 작업을 다시 시도하는 방식
- DLQ: 계속 실패한 메시지를 격리하는 큐
- Topic: Kafka에서 이벤트를 분류하는 논리 단위
- Partition: Topic을 나누는 물리적 병렬 처리 단위
- Offset: Consumer가 어디까지 읽었는지 나타내는 위치
- Cache: 자주 쓰는 데이터를 빠른 저장소에 임시 저장하는 구조
- TTL: 데이터 만료 시간
- Persistence: 장애 후 복구를 위해 디스크에 저장하는 방식

이 개념을 이해하면 RabbitMQ는 Queue와 ACK 중심, Kafka는 Topic/Partition/Offset 중심, Redis는 Memory/TTL/Data Structure 중심으로 자연스럽게 구분된다.
