---
layout: post
title: "04. Kubernetes에서 RabbitMQ, Kafka, Redis를 배포하고 운영할 때 보는 것"
date: 2026-07-05 13:30:00 +0900
categories: [Kubernetes, Middleware, PlatformEngineering]
tags: [Kubernetes, StatefulSet, PVC, RabbitMQ, Kafka, Redis, HA, Operator, Helm]
published: true
---

> RabbitMQ, Kafka, Redis는 모두 Kubernetes 위에 올릴 수 있다. 그러나 단순히 Pod로 실행한다고 끝나는 애플리케이션이 아니다. 상태를 가진 Stateful Middleware이기 때문에 StatefulSet, PVC, Headless Service, HA, 장애 복구, 모니터링을 함께 이해해야 한다.

## 1. Kubernetes에서 미들웨어를 운영한다는 의미

Kubernetes는 컨테이너를 배포하고 관리하는 플랫폼이다. Stateless 웹 애플리케이션은 Deployment로 쉽게 확장하고 교체할 수 있다. 하지만 RabbitMQ, Kafka, Redis 같은 미들웨어는 상태를 가진다.

상태를 가진다는 것은 다음을 의미한다.

- Pod가 재시작되어도 데이터가 유지되어야 한다.
- Pod 이름과 네트워크 식별자가 일정하게 유지되어야 할 수 있다.
- 노드 장애 시 데이터 복구나 리더 재선출이 필요하다.
- 디스크 성능, 메모리, 네트워크 지연 시간이 서비스 품질에 직접 영향을 준다.
- 단순 Rolling Update가 데이터 정합성 문제를 만들 수 있다.

따라서 Kubernetes에서 RabbitMQ, Kafka, Redis를 운영할 때는 Deployment 하나로 끝내지 않는다. 보통 StatefulSet, Headless Service, PVC, PodDisruptionBudget, Anti-Affinity, Readiness Probe, Monitoring 구성을 함께 설계한다.

## 2. Deployment와 StatefulSet 차이

Deployment는 Stateless 애플리케이션에 적합하다. Pod 이름이 바뀌어도 되고, 어느 Pod가 어떤 데이터를 가지고 있는지 중요하지 않은 경우에 사용한다.

```text
api-deployment-abc123
api-deployment-def456
api-deployment-ghi789
```

반면 StatefulSet은 Pod에 안정적인 이름과 순서를 부여한다.

```text
rabbitmq-0
rabbitmq-1
rabbitmq-2

kafka-0
kafka-1
kafka-2

redis-0
redis-1
redis-2
```

StatefulSet은 각 Pod에 고정된 PVC를 연결할 수 있다.

```text
rabbitmq-0 → data-rabbitmq-0
rabbitmq-1 → data-rabbitmq-1
rabbitmq-2 → data-rabbitmq-2
```

RabbitMQ, Kafka, Redis Cluster는 노드 ID, broker ID, replica 관계, cluster membership이 중요하기 때문에 StatefulSet이 자주 사용된다.

## 3. Headless Service가 필요한 이유

일반 ClusterIP Service는 여러 Pod 뒤에 하나의 가상 IP를 제공한다.

```text
Service IP
  ├─ Pod A
  ├─ Pod B
  └─ Pod C
```

하지만 Kafka나 RabbitMQ, Redis Cluster처럼 각 노드가 서로를 식별해야 하는 시스템에서는 개별 Pod DNS가 필요하다. 이때 Headless Service를 사용한다.

```text
rabbitmq-0.rabbitmq-headless.namespace.svc.cluster.local
rabbitmq-1.rabbitmq-headless.namespace.svc.cluster.local
rabbitmq-2.rabbitmq-headless.namespace.svc.cluster.local
```

Kafka Broker도 각 Broker가 독립적인 네트워크 주소를 가져야 한다. Consumer와 Producer가 특정 Broker로 연결해야 하고, Broker metadata에 각 Broker의 advertised address가 들어가기 때문이다.

## 4. PVC와 StorageClass

RabbitMQ, Kafka, Redis를 운영할 때 저장소는 매우 중요하다.

RabbitMQ는 durable queue와 persistent message를 사용할 경우 메시지를 디스크에 저장한다. Queue가 쌓이면 디스크 사용량이 증가하고, 디스크 latency가 메시지 처리 지연에 영향을 준다.

Kafka는 이벤트를 디스크 로그로 저장한다. Kafka 성능은 순차 I/O, 디스크 처리량, page cache, replication, flush 정책에 영향을 받는다. Kafka에서 PVC 성능이 낮으면 producer latency, consumer lag, broker under-replicated partition 문제가 발생할 수 있다.

Redis는 기본적으로 메모리 기반이지만 RDB/AOF를 사용하면 디스크에 데이터를 기록한다. Redis가 캐시 용도라면 PVC가 필수는 아닐 수 있지만, 세션이나 중요한 상태를 저장한다면 persistence와 PVC를 고려해야 한다.

Kubernetes 환경에서는 다음을 확인해야 한다.

- StorageClass가 어떤 스토리지를 사용하는가?
- ReadWriteOnce인지 ReadWriteMany인지?
- 디스크 IOPS와 latency가 충분한가?
- Pod가 다른 노드로 이동했을 때 PVC가 정상 attach되는가?
- 백업과 복구 절차가 있는가?
- 스토리지 장애가 미들웨어 장애로 어떻게 전파되는가?

## 5. RabbitMQ on Kubernetes

### 5-1) 기본 구성

RabbitMQ를 Kubernetes에서 운영할 때 일반적인 구성은 다음과 같다.

```text
StatefulSet
  ├─ rabbitmq-0
  ├─ rabbitmq-1
  └─ rabbitmq-2

Headless Service
ClusterIP Service
PVC per Pod
Management UI Service
```

Headless Service는 클러스터 노드 간 discovery에 사용하고, 일반 ClusterIP Service는 애플리케이션이 RabbitMQ에 접근할 때 사용한다. Management UI를 별도 Service로 노출하면 운영자가 Queue 상태, connection, channel, exchange, binding, message rate를 확인할 수 있다.

### 5-2) HA와 Queue 복제

RabbitMQ에서 HA를 생각할 때 핵심은 Queue의 가용성이다. 과거에는 mirrored queue를 많이 사용했지만, 최근에는 quorum queue가 권장되는 흐름이다. Quorum Queue는 Raft 기반으로 여러 노드에 메시지를 복제해 leader 장애 시 follower가 승격될 수 있게 한다.

운영 관점에서는 다음 질문을 해야 한다.

- 중요한 Queue는 quorum queue로 구성되어 있는가?
- 메시지 durability가 필요한가?
- ACK와 publisher confirm을 사용하고 있는가?
- Consumer가 죽었을 때 메시지가 재전달되는가?
- DLQ 정책이 있는가?
- Queue가 특정 노드에 쏠리지 않는가?

### 5-3) 장애 대응

RabbitMQ Pod 하나가 죽으면 Kubernetes는 Pod를 재생성한다. 하지만 이것만으로 메시징 고가용성이 보장되는 것은 아니다. Queue가 어떤 노드에 있었는지, 메시지가 디스크에 저장되었는지, quorum queue가 몇 개 replica를 가지고 있는지에 따라 복구 방식이 달라진다.

Consumer가 죽으면 RabbitMQ는 unacked message를 다시 Queue로 돌려 다른 Consumer에게 전달할 수 있다. 이 때문에 Consumer 로직은 멱등성을 가져야 한다. 같은 메시지가 두 번 처리될 수 있기 때문이다.

### 5-4) 모니터링 지표

RabbitMQ에서 중요하게 봐야 할 지표는 다음과 같다.

- Queue depth
- Ready messages
- Unacked messages
- Publish rate
- Deliver rate
- Ack rate
- Consumer count
- Connection count
- Channel count
- Disk free alarm
- Memory alarm
- Node partition 상태

Queue depth가 계속 증가하면 Consumer 처리량이 부족하다는 신호다. Unacked messages가 증가하면 Consumer가 메시지를 가져갔지만 ACK를 보내지 못하고 있다는 의미다. Disk free alarm이나 memory alarm은 RabbitMQ가 flow control을 걸 수 있는 위험 신호다.

## 6. Kafka on Kubernetes

### 6-1) 기본 구성

Kafka를 Kubernetes에서 운영할 때 일반적인 구성은 다음과 같다.

```text
StatefulSet
  ├─ kafka-0
  ├─ kafka-1
  └─ kafka-2

Headless Service
Broker Service
PVC per Broker
Kafka Controller / KRaft
```

Kafka는 Broker ID와 advertised listener 설정이 중요하다. Producer와 Consumer는 bootstrap server를 통해 클러스터 metadata를 받고, 각 Broker로 직접 연결한다. Kubernetes 내부 DNS, 외부 접근, Ingress/LB, advertised address가 맞지 않으면 클라이언트 연결 문제가 발생한다.

### 6-2) ZooKeeper와 KRaft

기존 Kafka는 클러스터 metadata 관리를 위해 ZooKeeper를 사용했다. 최근 Kafka는 KRaft 모드로 ZooKeeper 없이 자체 controller quorum을 구성하는 방향으로 발전했다. 운영자는 현재 설치하려는 Kafka 배포판이 ZooKeeper 기반인지 KRaft 기반인지 확인해야 한다.

Kubernetes에서는 Kafka Operator를 사용하는 경우가 많다. 대표적으로 Strimzi Operator가 있다. Operator는 Kafka 클러스터, Topic, User, 인증서, rolling update 등을 Kubernetes CRD로 관리할 수 있게 도와준다.

### 6-3) Partition, Replication, ISR

Kafka HA의 핵심은 replication이다. 각 Partition은 leader와 follower replica를 가진다.

```text
Partition 0
  leader: kafka-0
  follower: kafka-1
  follower: kafka-2
```

ISR(In-Sync Replica)은 leader와 동기화 상태를 유지하고 있는 replica 목록이다. leader broker가 죽으면 ISR 안의 follower가 leader로 승격될 수 있다.

운영자는 다음을 확인해야 한다.

- replication factor가 충분한가?
- min.insync.replicas 설정은 적절한가?
- under-replicated partitions가 없는가?
- leader election이 정상적으로 되는가?
- broker 간 network latency가 높지 않은가?
- partition 수가 너무 많거나 적지 않은가?

### 6-4) Consumer Lag

Kafka 운영에서 가장 중요한 지표 중 하나는 Consumer Lag이다.

```text
Consumer Lag = 최신 Offset - Consumer가 처리한 Offset
```

Consumer Lag이 증가한다는 것은 Consumer가 Producer 속도를 따라가지 못한다는 의미다. 원인은 여러 가지다.

- Consumer Pod 수가 부족함
- Partition 수가 부족해 병렬 처리 한계에 도달함
- Consumer 처리 로직이 느림
- 외부 DB나 API가 병목임
- Broker 또는 Storage 성능이 부족함
- Rebalance가 자주 발생함

Kafka는 단순히 Pod 수를 늘린다고 항상 처리량이 늘어나지 않는다. 하나의 Consumer Group에서 동시에 처리할 수 있는 Consumer 수는 Partition 수에 의해 제한된다. Partition이 3개인데 Consumer를 10개 띄우면 7개는 할당받을 Partition이 없어 놀 수 있다.

### 6-5) 모니터링 지표

Kafka에서 중요하게 봐야 할 지표는 다음과 같다.

- Broker CPU, Memory, Disk, Network
- Disk usage by log
- Under replicated partitions
- Offline partitions
- Active controller count
- Request latency
- Produce/Fetch request rate
- Consumer lag
- ISR shrink/expand rate
- Rebalance count
- Partition count

Kafka 장애는 단순 Pod 장애뿐 아니라 consumer lag 증가, partition imbalance, disk full, ISR 축소, network latency로도 나타난다.

## 7. Redis on Kubernetes

### 7-1) 기본 구성

Redis를 Kubernetes에서 운영하는 방식은 용도에 따라 달라진다.

단순 캐시라면 Deployment 또는 StatefulSet 단일 인스턴스로 시작할 수 있다. 하지만 운영 환경에서는 보통 replication, Sentinel, Cluster mode를 고려한다.

```text
Redis Master
  ├─ Redis Replica 1
  └─ Redis Replica 2

Sentinel
  ├─ sentinel-0
  ├─ sentinel-1
  └─ sentinel-2
```

또는 Redis Cluster를 구성할 수 있다.

```text
redis-0 master
redis-1 master
redis-2 master
redis-3 replica
redis-4 replica
redis-5 replica
```

### 7-2) Redis Sentinel

Sentinel은 Redis master를 감시하고 장애 발생 시 replica를 master로 승격시키는 역할을 한다. 애플리케이션은 Sentinel을 통해 현재 master 주소를 확인하거나, 클라이언트 라이브러리의 Sentinel 지원 기능을 사용한다.

Sentinel 구성에서는 다음을 확인해야 한다.

- Sentinel quorum이 충분한가?
- master 장애 감지 시간이 적절한가?
- failover 후 애플리케이션이 새 master로 연결되는가?
- split-brain 가능성은 없는가?
- replica 동기화 지연은 어느 정도인가?

### 7-3) Redis Cluster

Redis Cluster는 데이터를 slot 단위로 나눠 여러 master에 분산 저장한다. 고가용성을 위해 각 master에 replica를 붙인다.

Redis Cluster는 수평 확장에 유리하지만, 클라이언트가 cluster-aware해야 한다. 즉, MOVED/ASK redirect를 처리할 수 있는 클라이언트 라이브러리를 사용해야 한다.

Kubernetes에서 Redis Cluster를 운영할 때는 Pod IP 변경, DNS, persistent identity, slot migration, resharding을 고려해야 한다.

### 7-4) Persistence와 메모리 운영

Redis는 메모리 기반이므로 가장 중요한 운영 지표는 메모리다.

- used_memory
- maxmemory
- eviction count
- keyspace hits/misses
- expired keys
- rejected connections
- blocked clients
- replication lag
- AOF rewrite 상태
- RDB save 상태

Redis를 캐시로 쓴다면 eviction이 어느 정도 발생해도 서비스가 동작할 수 있다. 하지만 세션이나 중요한 상태를 저장하는 경우 eviction은 장애로 이어질 수 있다. 따라서 Redis에 어떤 종류의 데이터를 저장하는지 먼저 분류해야 한다.

## 8. 공통 운영 설계 포인트

### 8-1) Resource Request/Limit

RabbitMQ, Kafka, Redis는 CPU보다 메모리와 디스크, 네트워크 영향을 크게 받는 경우가 많다. Kubernetes에서 request/limit을 너무 낮게 잡으면 OOMKilled, throttling, latency 증가가 발생할 수 있다.

Redis는 메모리 limit과 maxmemory 설정을 일치시켜야 한다. 컨테이너 메모리 limit보다 Redis maxmemory가 높으면 OOMKilled가 발생할 수 있다.

Kafka는 page cache를 많이 활용하므로 메모리 limit을 지나치게 낮게 잡으면 디스크 I/O가 증가하고 성능이 떨어질 수 있다.

RabbitMQ는 queue 적체와 unacked message 증가가 메모리 압박으로 이어질 수 있다.

### 8-2) Anti-Affinity

클러스터를 구성했더라도 모든 Pod가 같은 Worker Node에 올라가면 노드 장애 시 전체 미들웨어가 같이 죽을 수 있다. Pod Anti-Affinity 또는 Topology Spread Constraints를 사용해 Pod를 여러 노드에 분산해야 한다.

```text
worker-1: kafka-0
worker-2: kafka-1
worker-3: kafka-2
```

이런 분산은 Kafka, RabbitMQ, Redis 모두에서 중요하다.

### 8-3) PodDisruptionBudget

PDB는 자발적 중단 시 동시에 내려갈 수 있는 Pod 수를 제한한다. 노드 drain, upgrade, maintenance 중에 quorum이 깨지지 않도록 하는 데 필요하다.

예를 들어 Kafka 3 Broker 중 2개가 동시에 내려가면 replication과 leader election에 문제가 생길 수 있다. RabbitMQ quorum queue도 quorum을 잃으면 쓰기가 불가능할 수 있다. Redis Sentinel도 quorum이 깨지면 failover 판단이 어려워진다.

### 8-4) Probe

Readiness Probe와 Liveness Probe를 신중히 설정해야 한다.

Readiness Probe는 Pod가 트래픽을 받을 준비가 되었는지 판단한다. Kafka Broker가 클러스터에 합류하기 전, RabbitMQ가 cluster formation을 마치기 전, Redis가 replica sync 중일 때는 ready로 보면 안 된다.

Liveness Probe는 너무 공격적으로 설정하면 복구 중인 Pod를 계속 재시작시켜 오히려 장애를 키울 수 있다. Stateful 미들웨어에서는 Liveness Probe를 보수적으로 설정해야 한다.

## 9. Helm과 Operator

RabbitMQ, Kafka, Redis는 Helm Chart로 설치할 수 있다. 하지만 운영 수준에서는 Operator를 사용하는 경우도 많다.

Helm은 설치와 values 관리에 유리하다. 단순 환경이나 테스트 환경에서는 Helm Chart로 충분할 수 있다.

Operator는 Kubernetes CRD를 통해 클러스터 운영 로직을 자동화한다. Kafka의 경우 Strimzi Operator, RabbitMQ의 경우 RabbitMQ Cluster Operator, Redis의 경우 여러 커뮤니티/벤더 Operator가 있다.

Operator를 사용하면 cluster scaling, rolling update, certificate management, user/topic 관리, failover 등을 더 Kubernetes-native하게 처리할 수 있다. 다만 Operator 자체의 동작 방식과 CRD 버전 관리도 운영 부담이 될 수 있다.

## 10. 장애 시나리오별 확인 포인트

### 10-1) RabbitMQ 장애

RabbitMQ에서 메시지가 쌓이면 먼저 Queue depth와 Consumer 상태를 본다. Consumer가 없거나 ACK를 보내지 못하면 unacked messages가 증가한다. Memory alarm이나 Disk alarm이 발생하면 RabbitMQ가 publish를 제한할 수 있다.

확인 순서:

1. Pod 상태 확인
2. RabbitMQ Management UI 확인
3. Queue depth, unacked 확인
4. Consumer 연결 수 확인
5. Disk/memory alarm 확인
6. DLQ 적재 여부 확인
7. 애플리케이션 Consumer 로그 확인

### 10-2) Kafka 장애

Kafka에서 가장 먼저 볼 것은 broker 상태, under-replicated partitions, consumer lag이다. Producer 오류가 있다면 advertised listener, broker 연결, ACL, disk full을 확인한다.

확인 순서:

1. Broker Pod 상태 확인
2. Topic/Partition 상태 확인
3. Under replicated partitions 확인
4. Consumer lag 확인
5. Broker disk usage 확인
6. Controller 상태 확인
7. Rebalance 빈도 확인
8. Producer/Consumer 로그 확인

### 10-3) Redis 장애

Redis 장애는 메모리, eviction, failover, connection 문제로 자주 나타난다. 캐시 미스가 급증하면 DB 부하가 함께 증가할 수 있다.

확인 순서:

1. Redis Pod 상태 확인
2. master/replica 역할 확인
3. Sentinel 또는 Cluster 상태 확인
4. memory usage와 eviction 확인
5. keyspace hit/miss 확인
6. blocked clients 확인
7. persistence 상태 확인
8. 애플리케이션 connection pool 확인

## 11. 정리

Kubernetes에서 RabbitMQ, Kafka, Redis를 운영한다는 것은 단순히 Helm install을 실행하는 것이 아니다. 이들은 상태를 가진 핵심 인프라 미들웨어이므로 Pod, Service, PVC, Storage, HA, Failover, Monitoring, Backup을 함께 설계해야 한다.

RabbitMQ는 Queue와 ACK, DLQ, Quorum Queue 중심으로 봐야 한다. Kafka는 Topic, Partition, Replication, ISR, Consumer Lag 중심으로 봐야 한다. Redis는 Memory, TTL, Eviction, Persistence, Sentinel/Cluster 중심으로 봐야 한다.

인프라 및 플랫폼 엔지니어의 역할은 애플리케이션이 이 미들웨어를 사용할 수 있게 설치하는 데서 끝나지 않는다. 장애가 났을 때 어떤 지표를 보고, 어떤 레이어에서 병목이 발생했는지 찾고, 데이터 유실과 서비스 중단을 최소화할 수 있도록 운영 구조를 설계하는 것이다.
