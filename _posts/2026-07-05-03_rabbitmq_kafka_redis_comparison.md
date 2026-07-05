---
layout: post
title: "03. RabbitMQ, Kafka, Redis 내부 구조와 선택 기준 비교"
date: 2026-07-05 12:40:00 +0900
categories: [Infra, Middleware, Messaging]
tags: [RabbitMQ, Kafka, Redis, AMQP, EventStreaming, MessageQueue, RedisStreams]
published: true
---

> RabbitMQ, Kafka, Redis는 모두 메시징과 관련해 함께 비교되지만, 내부 구조와 설계 목적은 크게 다르다. 이 문서는 메시지 브로커, 이벤트 브로커, 인메모리 저장소 관점에서 세 기술의 차이를 자세히 정리한다.

## 1. 한 줄 요약

RabbitMQ는 메시지를 안전하게 전달하고 작업을 분산 처리하기 위한 메시지 브로커다. Kafka는 대량 이벤트를 디스크 로그에 저장하고 여러 소비자가 독립적으로 읽는 이벤트 스트리밍 플랫폼이다. Redis는 메모리 기반 데이터 구조 저장소이며, 캐시·세션·분산락·간단한 큐·Pub/Sub·Streams 등에 활용된다.

```text
RabbitMQ = 작업 큐 / 안정적 메시지 전달 / 복잡한 라우팅
Kafka   = 이벤트 로그 / 대량 스트리밍 / 재처리 / 데이터 파이프라인
Redis   = 인메모리 저장소 / 캐시 / 빠른 임시 데이터 / 간단한 메시징
```

이 세 제품은 서로 완전한 대체재가 아니다. 일부 기능이 겹쳐 보일 뿐, 주 사용 목적이 다르다.

## 2. RabbitMQ: Message Broker

### 2-1) RabbitMQ의 역할

RabbitMQ는 메시지 브로커다. Producer가 메시지를 보내면 RabbitMQ가 메시지를 받아 Queue에 보관하고, Consumer가 메시지를 가져가 처리한다. 메시지는 보통 처리 완료 후 Queue에서 제거된다.

```text
Producer
  ↓
Exchange
  ↓
Queue
  ↓
Consumer
```

RabbitMQ는 단순히 Queue 하나만 제공하는 도구가 아니다. Exchange, Binding, Routing Key를 통해 메시지를 어떤 Queue로 보낼지 유연하게 제어할 수 있다. 이 점이 Redis List 기반 Queue와 큰 차이다.

### 2-2) AMQP와 RabbitMQ

RabbitMQ는 대표적으로 AMQP(Advanced Message Queuing Protocol)를 사용한다. AMQP는 메시징을 위한 표준 프로토콜이며, Producer, Broker, Consumer 사이의 메시지 전달 방식을 정의한다.

구조적으로 보면 RabbitMQ에서는 Producer와 Broker 사이, Broker와 Consumer 사이가 AMQP 기반으로 통신한다.

```text
Producer ── AMQP ── RabbitMQ Broker ── AMQP ── Consumer
```

AMQP를 사용한다는 것은 단순 TCP 연결 위에서 임의로 데이터를 주고받는 것이 아니라, 메시지 전달, Queue, Exchange, Routing, ACK 등 메시징에 필요한 개념이 프로토콜 수준에서 체계화되어 있다는 의미다.

### 2-3) Exchange, Binding, Queue, Routing Key

RabbitMQ에서 Producer는 보통 Queue에 직접 메시지를 넣지 않는다. Producer는 Exchange에 메시지를 보낸다. Exchange는 메시지를 받아 Binding 규칙과 Routing Key를 보고 적절한 Queue로 전달한다.

```text
Producer
  ↓
Exchange
  ↓ Binding + Routing Key
Queue A / Queue B / Queue C
  ↓
Consumer
```

이 구조 덕분에 RabbitMQ는 복잡한 라우팅이 가능하다.

### 2-4) Exchange 타입

RabbitMQ의 핵심은 Exchange다. 대표적인 Exchange 타입은 Direct, Topic, Fanout, Headers다.

Direct Exchange는 Routing Key가 정확히 일치하는 Queue로 메시지를 보낸다.

```text
routing_key = order.created
  ↓
order.created에 바인딩된 Queue로 전달
```

Topic Exchange는 패턴 매칭을 사용한다. 예를 들어 `order.*`, `payment.#` 같은 패턴으로 여러 종류의 메시지를 분류할 수 있다.

```text
order.created
order.cancelled
order.shipped
```

Fanout Exchange는 연결된 모든 Queue로 메시지를 복사해 전달한다. 방송에 가깝다.

```text
OrderCreated
  ↓
Fanout Exchange
  ├─ payment.queue
  ├─ inventory.queue
  └─ notification.queue
```

Headers Exchange는 Routing Key 대신 메시지 헤더를 기준으로 라우팅한다. 일반적으로 Direct, Topic, Fanout에 비해 덜 자주 사용된다.

### 2-5) RabbitMQ가 강한 영역

RabbitMQ는 다음 상황에 적합하다.

첫째, 작업 큐가 필요한 경우다. 이메일 발송, 이미지 처리, 문서 변환, 결제 후처리처럼 하나의 작업을 하나의 Worker가 안전하게 처리해야 하는 경우 RabbitMQ가 적합하다.

둘째, 복잡한 라우팅이 필요한 경우다. 메시지 유형, 업무 도메인, 지역, 서비스별로 Queue를 나눠야 한다면 Exchange와 Routing Key 구조가 유리하다.

셋째, 처리 완료 확인이 중요한 경우다. Consumer가 ACK를 보내기 전까지 메시지를 보관하고, Consumer 장애 시 다른 Consumer에게 재전달할 수 있다.

넷째, 메시지 우선순위, TTL, DLQ 같은 엔터프라이즈 메시징 기능이 필요한 경우다.

### 2-6) RabbitMQ의 한계

RabbitMQ는 대량 이벤트를 장기간 저장하고 여러 Consumer Group이 독립적으로 재처리하는 용도로는 Kafka보다 적합하지 않다. RabbitMQ는 기본적으로 메시지를 전달하고 처리되면 제거하는 모델에 가깝다. 물론 설정에 따라 메시지를 보관할 수 있지만, Kafka처럼 이벤트 로그를 장기간 저장하고 replay하는 것이 중심 설계는 아니다.

또한 대규모 클러스터링과 파티션 기반 병렬 처리에서는 Kafka가 더 자연스럽다. RabbitMQ 클러스터는 Queue의 상태와 메시지 삭제/ACK 상태를 노드 간 맞춰야 하므로, 대량 스트리밍 시스템처럼 운영하는 데는 설계상 부담이 있다.

## 3. Kafka: Event Streaming Platform

### 3-1) Kafka의 역할

Kafka는 단순 메시지 큐라기보다 이벤트 스트리밍 플랫폼이다. Producer가 이벤트를 Topic에 기록하면 Kafka는 이를 디스크 로그에 저장한다. Consumer는 Topic의 Partition을 읽고, 자신이 어디까지 읽었는지를 Offset으로 관리한다.

```text
Producer
  ↓
Kafka Broker
  ↓
Topic
  ↓
Partition
  ↓
Consumer Group
```

Kafka의 핵심은 메시지를 소비했다고 즉시 삭제하지 않는다는 점이다. Kafka는 이벤트를 로그처럼 보관하고, Consumer는 Offset을 통해 읽기 위치를 관리한다.

### 3-2) Kafka의 통신 방식

Kafka는 TCP 기반 자체 프로토콜을 사용한다. RabbitMQ가 AMQP 같은 메시징 프로토콜 중심이라면, Kafka는 고성능 로그 저장과 스트리밍 처리를 위해 자체 프로토콜과 클라이언트 생태계를 발전시켰다.

```text
Producer ── TCP 기반 Kafka Protocol ── Kafka Broker ── TCP 기반 Kafka Protocol ── Consumer
```

중요한 것은 "TCP를 쓴다" 자체가 아니라, Kafka가 메시지를 Queue가 아니라 Log로 다룬다는 점이다.

### 3-3) Topic과 Partition

Kafka에서 Topic은 이벤트의 논리적 분류 단위다.

```text
Topic: user-events
Topic: order-events
Topic: payment-events
```

Partition은 Topic을 나눈 물리적 저장 단위다.

```text
Topic: order-events
  Partition 0
  Partition 1
  Partition 2
```

Producer는 Key를 기준으로 특정 Partition에 이벤트를 기록할 수 있다. 같은 Key를 가진 이벤트는 같은 Partition에 들어가므로 순서 보장이 가능하다. 예를 들어 주문 ID를 Key로 사용하면 같은 주문에 대한 이벤트 순서를 유지할 수 있다.

### 3-4) Offset과 Replay

Kafka의 각 메시지는 Partition 안에서 Offset을 가진다.

```text
Partition 0
  offset 0: OrderCreated
  offset 1: PaymentCompleted
  offset 2: ShippingStarted
```

Consumer는 마지막으로 읽은 Offset을 저장한다. 장애가 발생하면 마지막 Offset부터 다시 읽을 수 있고, 필요하면 과거 Offset으로 되돌려 재처리할 수도 있다.

이것이 Kafka가 이벤트 브로커로서 강력한 이유다. RabbitMQ에서는 메시지가 처리되면 삭제되는 모델이 일반적이지만, Kafka에서는 일정 기간 이벤트가 보존되므로 같은 이벤트를 여러 번 읽거나 새로운 시스템이 과거 이벤트부터 처리할 수 있다.

### 3-5) Consumer Group

Consumer Group은 같은 목적을 가진 Consumer들의 묶음이다. 하나의 Consumer Group 안에서는 각 Partition이 하나의 Consumer에게 할당된다. 이를 통해 병렬 처리가 가능하다.

```text
Topic: order-events
  Partition 0 → Consumer A
  Partition 1 → Consumer B
  Partition 2 → Consumer C
```

다른 Consumer Group은 같은 Topic을 독립적으로 읽을 수 있다.

```text
order-events Topic
  ├─ analytics-consumer-group
  ├─ notification-consumer-group
  └─ fraud-detection-consumer-group
```

각 Consumer Group은 별도의 Offset을 가지므로 서로 영향을 주지 않는다. 이것이 Kafka가 여러 팀과 여러 시스템이 같은 이벤트를 독립적으로 소비하는 데 적합한 이유다.

### 3-6) Kafka가 강한 영역

Kafka는 다음 상황에 적합하다.

첫째, 대량 이벤트 스트리밍이 필요한 경우다. 클릭 로그, 결제 이벤트, 사용자 행동 이벤트, IoT 데이터, 애플리케이션 로그처럼 초당 많은 이벤트가 발생하는 시스템에 적합하다.

둘째, 여러 시스템이 같은 이벤트를 독립적으로 읽어야 하는 경우다. 주문 이벤트를 결제 시스템, 배송 시스템, 추천 시스템, 분석 시스템이 각각 읽어야 한다면 Kafka가 적합하다.

셋째, 이벤트 재처리가 필요한 경우다. 장애 발생 후 특정 Offset부터 다시 읽거나, 신규 분석 시스템이 과거 이벤트를 다시 읽어 데이터셋을 만들 수 있다.

넷째, 데이터 파이프라인이 필요한 경우다. Kafka는 실시간 ETL, 데이터 레이크 적재, 스트림 처리, 로그 수집과 잘 맞는다.

### 3-7) Kafka의 한계

Kafka는 설치와 운영이 RabbitMQ나 Redis보다 복잡하다. Broker, Controller, Partition, Replication, ISR, Offset, Consumer Lag, Rebalance, Retention, Compaction 등 운영자가 이해해야 할 개념이 많다.

또한 단순 작업 큐에는 과할 수 있다. 이메일 발송이나 간단한 백그라운드 작업 몇 개를 처리하려고 Kafka를 도입하면 운영 복잡도에 비해 이점이 작을 수 있다.

## 4. Redis: In-Memory Data Store

### 4-1) Redis의 역할

Redis는 인메모리 데이터 구조 저장소다. 데이터를 메모리에 저장하기 때문에 매우 빠른 읽기/쓰기 성능과 낮은 지연 시간을 제공한다.

```text
Application
  ↓
Redis
  ↓ Cache Miss
Database
```

Redis는 캐시, 세션 저장소, 인증 토큰 저장, 인증번호 TTL 관리, 분산락, 랭킹, 카운터, Rate Limiting 등에 많이 사용된다.

### 4-2) Redis 자료구조

Redis는 단순 Key-Value 저장소가 아니라 다양한 자료구조를 제공한다.

- String: 일반 값, 카운터, 토큰
- Hash: 객체 형태 데이터
- List: Queue처럼 사용 가능
- Set: 중복 없는 집합
- Sorted Set: 랭킹, 점수 기반 정렬
- Bitmap: 대량 boolean 상태
- HyperLogLog: 대략적인 고유 개수 추정
- Stream: 로그형 메시지 처리
- Geo: 위치 기반 데이터

이 자료구조 때문에 Redis는 단순 캐시를 넘어 다양한 실시간 기능 구현에 사용된다.

### 4-3) Redis를 Queue로 사용하는 방식

Redis List를 사용하면 간단한 Queue를 만들 수 있다. 예를 들어 왼쪽에 넣고 오른쪽에서 꺼내면 FIFO Queue처럼 동작한다.

```text
LPUSH queue task1
BRPOP queue
```

이 방식은 단순하고 빠르다. 이미 Redis를 캐시로 사용 중이라면 별도 브로커를 도입하지 않고 간단한 비동기 작업을 처리할 수 있다.

하지만 단순 List Queue는 메시지 안정성 측면에서 주의가 필요하다. Worker가 메시지를 꺼낸 뒤 처리 중 죽으면 메시지가 이미 List에서 제거되어 유실될 수 있다. Redis Streams를 사용하거나 Celery 설정을 강화하면 일부 보완할 수 있지만, RabbitMQ처럼 처음부터 안정적인 메시지 전달을 위해 설계된 도구와는 차이가 있다.

### 4-4) Redis Pub/Sub

Redis Pub/Sub는 Publisher가 Channel에 메시지를 발행하고, Subscriber가 해당 Channel을 구독해 메시지를 받는 구조다.

```text
Publisher
  ↓
Redis Channel
  ├─ Subscriber A
  └─ Subscriber B
```

Redis Pub/Sub는 매우 빠르고 간단하다. 채팅, 실시간 알림, 라이브 업데이트 같은 기능에서 사용할 수 있다. 하지만 기본 Pub/Sub는 메시지를 저장하지 않는다. Subscriber가 연결되어 있지 않은 동안 발행된 메시지는 놓칠 수 있다.

그래서 Redis Pub/Sub는 "반드시 저장되고 재처리되어야 하는 이벤트"보다는 "현재 연결된 대상에게 빠르게 전달하면 되는 실시간 알림"에 적합하다.

### 4-5) Redis Streams

Redis Streams는 Redis에서 로그형 메시지 스트림을 다루기 위한 자료구조다. Pub/Sub와 달리 메시지를 저장하고, Consumer Group과 Pending Entries List를 통해 어느 Consumer가 어떤 메시지를 처리 중인지 관리할 수 있다.

Redis Streams는 Kafka와 일부 유사해 보일 수 있다. 하지만 Kafka처럼 대규모 분산 로그, 장기 보존, Partition 기반 병렬 처리, 데이터 파이프라인 생태계를 목표로 설계된 시스템은 아니다. Redis Streams는 Redis 환경 안에서 비교적 가볍게 스트림 처리를 하고 싶을 때 유용하다.

### 4-6) Redis가 강한 영역

Redis는 다음 상황에 적합하다.

첫째, 낮은 지연 시간이 중요한 경우다. DB보다 빠르게 데이터를 읽어야 하는 캐시, 세션, 인증 토큰에 적합하다.

둘째, TTL 기반 임시 데이터가 필요한 경우다. 인증번호, 임시 토큰, 로그인 세션처럼 일정 시간이 지나면 사라져야 하는 데이터에 적합하다.

셋째, 실시간 랭킹이나 카운터가 필요한 경우다. Sorted Set과 atomic increment 기능을 활용할 수 있다.

넷째, 간단한 메시징이나 작업 큐를 빠르게 구성해야 하는 경우다. 단, 메시지 유실이 치명적이면 RabbitMQ나 Kafka를 검토해야 한다.

### 4-7) Redis의 한계

Redis는 메모리 기반이므로 대규모 데이터를 장기간 저장하는 용도에는 비용이 커질 수 있다. 또한 캐시로 쓰는 경우 데이터가 사라져도 원본 DB에서 다시 만들 수 있지만, 중요한 상태를 Redis에만 저장하면 장애 시 데이터 유실 문제가 발생할 수 있다.

Redis를 메시지 브로커로 사용할 수는 있지만, RabbitMQ처럼 복잡한 메시지 라우팅과 강한 전달 보장을 전문적으로 제공하는 도구는 아니다. Kafka처럼 대규모 이벤트 로그와 재처리를 중심으로 설계된 플랫폼도 아니다.

## 5. Queue 기반과 Log 기반의 차이

RabbitMQ와 Kafka의 가장 중요한 차이는 메시지를 다루는 방식이다.

RabbitMQ는 메시지를 Queue 형태로 저장하고, Consumer가 처리하면 제거하는 방식이다.

```text
Queue
  [msg1] [msg2] [msg3]
Consumer가 msg1 처리
  ↓
Queue
  [msg2] [msg3]
```

Kafka는 메시지를 Log 형태로 저장하고, Consumer가 읽어도 삭제하지 않는다.

```text
Log
  offset 0: msg1
  offset 1: msg2
  offset 2: msg3

Consumer A offset = 1
Consumer B offset = 0
```

RabbitMQ는 "작업을 전달하고 처리 완료 여부를 관리"하는 데 강하다. Kafka는 "이벤트를 저장하고 여러 소비자가 독립적으로 읽고 재처리"하는 데 강하다.

## 6. Smart Broker / Dumb Consumer vs Dumb Broker / Smart Consumer

RabbitMQ와 Kafka를 비교할 때 자주 나오는 표현이 있다.

RabbitMQ는 Smart Broker / Dumb Consumer에 가깝다. Broker가 Exchange, Routing, Queue, ACK, 재전달 같은 많은 일을 담당한다. Consumer는 상대적으로 단순하게 메시지를 받아 처리하고 ACK를 보내면 된다.

Kafka는 Dumb Broker / Smart Consumer에 가깝다. Broker는 이벤트를 로그에 저장하고 순차적으로 제공하는 역할에 집중한다. Consumer는 Offset 관리, 재처리, 처리 위치 제어, Consumer Group 동작을 이해해야 한다. 실제 운영에서는 Kafka Consumer 설계가 매우 중요하다.

이 차이는 운영 방식에도 영향을 준다. RabbitMQ는 브로커의 Queue 상태와 메시지 처리 상태가 중요하고, Kafka는 Consumer Lag과 Offset 관리가 중요하다.

## 7. Redis Pub/Sub와 Kafka의 차이

Redis Pub/Sub와 Kafka는 둘 다 Publish/Subscribe처럼 보이지만 목적이 다르다.

Redis Pub/Sub는 현재 연결된 Subscriber에게 빠르게 메시지를 전달한다. 메시지는 기본적으로 저장되지 않는다. Subscriber가 연결되어 있지 않으면 메시지를 놓친다. 따라서 실시간 알림, 채팅, live update처럼 유실이 어느 정도 허용되거나 별도 상태 저장이 있는 경우에 적합하다.

Kafka는 메시지를 Topic에 저장한다. Consumer가 늦게 읽어도 retention 안에 있으면 읽을 수 있다. Consumer Group별로 Offset을 독립적으로 관리하므로 여러 시스템이 같은 이벤트를 각자 처리할 수 있다. 따라서 로그 수집, 분석, 데이터 파이프라인, 이벤트 기반 마이크로서비스에 적합하다.

```text
Redis Pub/Sub = 빠른 실시간 전달, 저장 없음, 재처리 약함
Kafka = 저장된 이벤트 로그, 재처리 가능, 대량 스트리밍
```

## 8. Redis Streams와 Kafka의 차이

Redis Streams는 Pub/Sub보다 Kafka에 더 가까워 보인다. 메시지를 저장하고 Consumer Group도 지원한다. 하지만 다음 차이가 있다.

Kafka는 처음부터 분산 로그와 대규모 스트리밍 처리를 위해 설계되었다. Topic, Partition, Replication, Retention, Offset, Consumer Group, Connect, Streams 같은 생태계가 잘 갖춰져 있다.

Redis Streams는 Redis 안에서 제공되는 스트림 자료구조다. 가볍고 빠르며 Redis를 이미 운영 중인 환경에서는 도입이 쉽다. 하지만 대규모 데이터 파이프라인, 장기 보존, 수많은 Consumer Group, 파티션 기반 처리량 확장에서는 Kafka가 더 적합하다.

## 9. 비교표

| 구분 | RabbitMQ | Kafka | Redis |
|---|---|---|---|
| 기본 분류 | Message Broker | Event Streaming Platform | In-memory Data Store |
| 주 목적 | 안정적인 메시지 전달, 작업 큐 | 대량 이벤트 저장, 스트리밍, 재처리 | 캐시, 세션, 빠른 데이터 접근 |
| 데이터 모델 | Exchange → Queue → Consumer | Topic → Partition → Offset | Key-Value + 자료구조 |
| 저장 방식 | Queue 중심, 메모리/디스크 | 디스크 로그 중심 | 메모리 중심, RDB/AOF 가능 |
| 메시지 소비 후 | 보통 삭제 | retention까지 유지 | Pub/Sub는 저장 없음, Streams는 저장 |
| 재처리 | 제한적 | 강함 | Pub/Sub 약함, Streams 일부 가능 |
| 라우팅 | 매우 강함 | Topic/Partition 중심 | 단순함 |
| 처리량 | 중간~높음 | 매우 높음 | 매우 빠른 저지연 |
| 운영 복잡도 | 중간 | 높음 | 낮음~중간 |
| 대표 용도 | 이메일, 작업 큐, 주문 후처리 | 로그, 이벤트 파이프라인, 분석 | 캐시, 세션, 락, 랭킹 |

## 10. 선택 기준

RabbitMQ를 선택해야 하는 경우는 다음과 같다.

- 작업 큐가 필요하다.
- Consumer가 처리 완료한 뒤 메시지를 삭제하면 된다.
- ACK, Retry, DLQ가 중요하다.
- 복잡한 메시지 라우팅이 필요하다.
- 주문, 결제, 알림처럼 안정적 전달이 중요하다.

Kafka를 선택해야 하는 경우는 다음과 같다.

- 대량 이벤트를 지속적으로 수집해야 한다.
- 여러 시스템이 같은 이벤트를 독립적으로 읽어야 한다.
- 과거 이벤트를 재처리해야 한다.
- 데이터 파이프라인이나 실시간 분석이 필요하다.
- Consumer Lag과 Offset 기반 운영을 감당할 수 있다.

Redis를 선택해야 하는 경우는 다음과 같다.

- 매우 빠른 캐시가 필요하다.
- 세션, 인증번호, TTL 데이터가 필요하다.
- 간단한 메시징이나 실시간 알림이 필요하다.
- 이미 Redis를 운영 중이고 단순 작업 큐 정도만 필요하다.
- 데이터 유실이 치명적이지 않거나 별도 보완 설계가 있다.

## 11. 정리

RabbitMQ, Kafka, Redis는 이름만 나란히 비교하면 비슷해 보이지만 설계 목적이 다르다.

RabbitMQ는 "작업을 안전하게 전달하고 처리하는 시스템"이다. Kafka는 "이벤트를 로그처럼 저장하고 여러 시스템이 독립적으로 읽는 플랫폼"이다. Redis는 "메모리를 활용해 매우 빠르게 데이터를 저장하고 조회하는 자료구조 서버"다.

따라서 선택 기준은 다음 한 문장으로 정리할 수 있다.

```text
작업 큐와 안정적 전달이면 RabbitMQ,
대량 이벤트 스트리밍과 재처리면 Kafka,
초고속 캐시와 임시 상태 저장이면 Redis.
```
