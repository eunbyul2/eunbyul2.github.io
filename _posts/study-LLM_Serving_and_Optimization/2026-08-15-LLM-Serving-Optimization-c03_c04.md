---
layout: post
title: "[Hands-On LLM Serving and Optimization] Chapter 3·4"
date: 2026-08-15 19:20:00 +0900
categories: [LLMOps, LLM Serving]
tags: [LLM, Model Serving, vLLM, Multi-Model Serving, RAG, Kubernetes, Ray Serve]
published: true
---

# Chapter 3. Building a Model Serving System from Scratch

Chapter 2에서는 LLM이 실제로 어떻게 Token을 생성하는지, KV Cache가 왜 필요한지, Prefill/Decode가 어떻게 나뉘는지, Streaming과 Batching이 왜 필요한지를 살펴봤다.

Chapter 3에서는 이제 단순히 Python 코드에서 모델을 실행하는 것이 아니라 **여러 사용자의 요청을 받아 실제 Web API 형태로 모델을 제공하려면 어떤 시스템이 필요한가**를 다룬다.

즉 이번 장의 핵심 질문은 다음과 같다.

```text
모델을 GPU에 올려서 inference 하는 것
                ↓
        이것만으로 서비스가 될까?
                ↓
              아니다
                ↓
여러 사용자의 요청을 동시에 받고,
요청을 관리하고,
모델 실행을 스케줄링하고,
결과를 스트리밍하고,
장애와 부하를 처리할 수 있어야 한다.
```

## 1. Model Serving System은 무엇을 해결해야 하는가

LLM을 단순하게 실행하는 코드는 대략 다음과 같이 생각할 수 있다.

```text
Prompt
  ↓
Model
  ↓
Generated Text
```

하지만 실제 서비스에서는 요청이 하나만 들어오지 않는다.

```text
User A ─┐
User B ─┤
User C ─┼─→ Model Serving System ─→ GPU / Model
User D ─┤
User E ─┘
```

여기서 Serving System은 단순히 모델 호출을 전달하는 것이 아니라 다음 문제들을 처리해야 한다.

- 여러 사용자의 동시 요청
- 요청 대기열과 스케줄링
- Batch 구성
- Model Worker 관리
- Token Streaming
- Request 상태 추적
- 실패한 요청 처리
- 모델 실행과 API 계층 분리
- 확장성과 자원 활용률

따라서 Model Serving은 모델 자체보다도 **분산 시스템, API 서버, 스케줄링, 동시성, 자원 관리**의 성격이 강하다.

---

## 2. Single-Model Serving System의 기본 구조

교재의 단순화된 구현은 크게 다음 네 구성요소로 이해할 수 있다.

```text
Client Request
      ↓
┌────────────────────┐
│     API Server     │
└────────────────────┘
      ↓
┌────────────────────┐
│ Workload Manager   │
└────────────────────┘
      ↓
┌────────────────────┐
│  Model Executor    │
└────────────────────┘
      ↓
┌────────────────────┐
│   Model Worker     │
└────────────────────┘
      ↓
     LLM
```

각 컴포넌트가 역할을 분리하는 이유가 중요하다.

### API Server

API Server는 외부 사용자나 Application이 접속하는 **Frontend Interface**다.

주요 역할은 다음과 같다.

- HTTP 요청 수신
- 요청 내용을 내부 Request 형식으로 변환
- Workload Manager에 전달
- 생성 결과를 Client에 반환
- Streaming 응답 처리

즉 API Server 자체가 직접 GPU에서 모델을 실행하는 것이 아니라, **외부 요청을 내부 Serving Pipeline으로 연결하는 입구**에 가깝다.

이렇게 분리하면 API 처리 로직과 모델 실행 로직을 독립적으로 변경할 수 있다.

### Workload Manager

Workload Manager는 여러 요청을 실제 모델 실행 단계로 보내기 전에 관리한다.

```text
Request A ─┐
Request B ─┤
Request C ─┼─→ Workload Manager
Request D ─┘
```

주요 역할은 다음과 같이 이해할 수 있다.

- 현재 들어온 Request 추적
- 대기 중인 Request 관리
- 실행할 Request 선택
- Batch 구성
- Model Executor로 전달
- 생성된 Token과 Request를 다시 연결

여기서 중요한 점은 **동시 요청을 그냥 모두 GPU에 던질 수 없다는 것**이다.

GPU 메모리와 계산 자원에는 한계가 있기 때문에 Serving System은 어떤 요청을 언제 실행할지 결정해야 한다.

Chapter 2에서 배운 Batching이 실제 Serving System에서는 바로 이 계층의 스케줄링 문제와 연결된다.


### Model Executor

Model Executor는 Workload Manager가 선정한 작업을 실제 Model Worker에 전달하고 모델 실행을 제어하는 계층이다.

개념적으로는 다음과 같다.

```text
Workload Manager
      ↓
[실행할 Batch / Request]
      ↓
Model Executor
      ↓
Model Worker
```

이 계층을 별도로 두면 요청 관리와 실제 모델 실행을 분리할 수 있다.

Production Serving Framework에서는 이런 추상화가 훨씬 복잡해질 수 있으며, GPU 수가 여러 개라면 Executor가 여러 Worker를 관리하거나 분산 실행을 조정할 수도 있다.

### Model Worker

Model Worker는 최종적으로 **모델을 메모리에 올리고 inference를 수행하는 실행 단위**다.

```text
Model Worker
├── Model Load
├── Tokenizer
├── Inference
└── Output Generation
```

실제 GPU 자원과 가장 가까운 계층이라고 볼 수 있다.

Chapter 2에서 직접 모델을 실행했던 코드가 Chapter 3에서는 Model Worker 내부의 역할로 들어간다고 생각하면 이해하기 쉽다.

---

## 3. Request가 처리되는 전체 흐름

전체 흐름을 연결하면 다음과 같다.

```text
1. Client가 Prompt 전송
          ↓
2. API Server가 HTTP Request 수신
          ↓
3. Request를 내부 Workload로 등록
          ↓
4. Workload Manager가 실행 대상 결정
          ↓
5. 여러 Request를 Batch로 구성할 수 있음
          ↓
6. Model Executor가 Worker에 실행 요청
          ↓
7. Model Worker가 LLM inference 수행
          ↓
8. Token 생성
          ↓
9. 생성 결과를 원래 Request와 연결
          ↓
10. API Server가 Client에 Streaming 또는 최종 응답 반환
```

Chapter 2에서 보았던 LLM 내부 추론 과정이 **Serving System 내부의 한 부분**으로 들어간다는 점이 중요하다.

```text
Chapter 2
Prompt → Tokenizer → LLM → Token Generation

Chapter 3
Client
  ↓
API Server
  ↓
Scheduler / Workload Manager
  ↓
Executor
  ↓
[Chapter 2에서 본 LLM inference]
  ↓
Streaming Response
```

---

## 4. 왜 Concurrent Request 처리가 어려운가

사용자가 한 명이라면 구조는 단순하다.

```text
Request 1 → Model → Response 1
```

하지만 사용자가 여러 명이면 문제가 달라진다.

```text
Request 1 ─┐
Request 2 ─┤
Request 3 ─┼─→ 하나의 GPU
Request 4 ─┘
```

각 Request의 Prompt 길이와 생성할 Token 수가 다르기 때문에 처리 시간도 다르다.

예를 들어 다음 두 요청을 생각할 수 있다.

```text
Request A
Prompt: 20 tokens
Output: 20 tokens

Request B
Prompt: 4,000 tokens
Output: 1,000 tokens
```

두 요청의 계산량은 동일하지 않다.

따라서 단순 FIFO나 단순 Round-Robin으로만 요청을 처리하면 GPU 활용률이나 Latency가 나빠질 수 있다.

LLM Serving에서 Scheduling이 중요한 이유가 여기 있다.

---

## 5. Batch Processing

Batching은 여러 Request를 하나의 실행 단위로 묶어서 GPU에서 병렬로 처리하는 방식이다.

```text
Request 1 ┐
Request 2 ├─→ Batch → GPU
Request 3 ┤
Request 4 ┘
```

GPU는 Matrix 연산을 병렬로 처리하는 데 강하기 때문에 하나씩 순차 처리하는 것보다 여러 Sequence를 함께 처리하면 Throughput을 높일 수 있다.

하지만 단순한 Static Batching에는 문제가 있다.

```text
Batch Size = 4

Request 1 도착
Request 2 도착
Request 3 도착

→ Request 4가 올 때까지 기다린다면?
→ 앞의 요청 Latency 증가
```

따라서 실제 LLM Serving에서는 Batch Size뿐 아니라 **얼마나 기다렸다가 Batch를 만들지**, **실행 중인 Batch에 새로운 Request를 어떻게 넣을지** 같은 Scheduling 전략이 중요하다.

후속 Chapter에서 Continuous Batching을 더 자세히 다루지만, Chapter 3에서는 Serving System을 직접 구성해보면서 이런 기능이 왜 Serving Framework 내부에 필요한지를 보여준다.

---

## 6. Streaming

LLM은 Token을 하나씩 생성한다.

따라서 모든 Token 생성이 완료된 뒤 한 번에 응답할 수도 있지만, Chat Service에서는 사용자 경험이 좋지 않다.

```text
비 Streaming

Request
  ↓
Token 1 생성
Token 2 생성
Token 3 생성
...
Token 500 생성
  ↓
한 번에 Response
```

사용자는 전체 생성이 끝날 때까지 아무것도 볼 수 없다.

Streaming은 생성된 결과를 중간중간 바로 전달한다.

```text
Request
  ↓
Token 1 → Client
Token 2 → Client
Token 3 → Client
...
```

Streaming에서는 Serving System이 각 Request의 상태와 생성된 Token을 계속 추적해야 한다.

즉 단순히 `model.generate()`를 호출하는 것보다 다음 관리가 추가된다.

- Request ID 관리
- 현재 생성 위치 추적
- Token Buffer 관리
- Client 연결 상태 확인
- Request Cancellation 처리
- 완료 여부 확인

이 때문에 Streaming 역시 Production Serving Framework가 제공해야 하는 핵심 기능이다.

---

## 7. Prompt Tracking과 Request 상태 관리

동시에 여러 Request가 실행될 때는 생성된 Token이 어느 사용자 요청에 속하는지를 알아야 한다.

```text
Request A → Token A1, A2, A3 ...
Request B → Token B1, B2, B3 ...
Request C → Token C1, C2, C3 ...
```

Batch로 같이 실행하더라도 응답은 다시 각 Client로 분리해서 보내야 한다.

따라서 Serving System은 Request마다 다음과 같은 상태를 관리해야 한다.

```text
Request ID
Prompt
Generated Tokens
Status
Streaming Queue
Start Time
Completion State
```

이런 관리가 있어야 동시 요청과 Streaming을 안정적으로 지원할 수 있다.

---

## 8. 직접 구현과 vLLM의 차이

교재가 처음부터 vLLM만 사용하는 대신 Serving System을 직접 구성해보는 이유는 **vLLM이 어떤 문제를 대신 해결하는지를 이해하기 위해서**다.

직접 구현할 경우 개발자가 다음을 직접 고려해야 한다.

```text
API Server
Request Queue
Scheduling
Batching
Worker Management
Token Streaming
Request Cancellation
KV Cache Management
GPU Memory Management
Concurrency
Failure Handling
```

반면 vLLM과 같은 Production-grade Serving Framework를 사용하면 상당수 기능을 Framework가 추상화한다.

```text
Client
  ↓
vLLM API / Serving Layer
  ↓
Scheduler
  ↓
Batching / KV Cache Management
  ↓
Model Execution
  ↓
GPU
```

그래서 Framework를 사용하면 구현 코드는 단순해진다.

하지만 내부 원리를 모르면 설정값을 제대로 튜닝하기 어렵다.

예를 들어 다음 설정들을 조정할 때도 각각이 무엇에 영향을 주는지 이해해야 한다.

- Batch 관련 설정
- Concurrent Sequence 수
- GPU Memory Utilization
- KV Cache 관련 설정
- Parallelism 관련 설정

즉 Chapter 3의 메시지는 단순히 "직접 Serving Server를 구현하자"가 아니라 다음에 가깝다.

> **Serving Framework를 블랙박스로 사용하지 않기 위해 내부에서 어떤 문제가 해결되고 있는지 이해하자.**

---

# 9. Single-Model Serving

Single-Model Serving은 하나의 Serving Service가 하나의 Model을 전담하는 방식이다.

```text
Service A
└── Model A

Service B
└── Model B

Service C
└── Model C
```

LLM은 모델 하나가 GPU Memory를 많이 사용하기 때문에 Single-Model Serving이 특히 일반적이다.

장점은 다음과 같다.

- 모델 간 자원 경합이 적음
- 모델별 독립 Scaling 가능
- 장애 격리가 쉬움
- Debugging과 Monitoring이 단순함
- 모델에 맞는 GPU 설정 가능

하지만 모델 수가 매우 많으면 Service도 함께 증가한다.

```text
100 Customers
× 10 Models
= 1,000 Model Services
```

이 경우 운영 비용과 Resource 낭비가 커질 수 있다.

이 문제 때문에 Multi-Model Serving이 등장한다.

---

# 10. Multi-Model Serving

Multi-Model Serving은 여러 모델을 하나의 Serving Infrastructure에서 공유하는 방식이다.

```text
Model Serving Container
├── Model A
├── Model B
├── Model C
└── Model D
```

핵심 목표는 **Compute와 Memory Resource를 여러 모델이 공유해 Cost Efficiency를 높이는 것**이다.

모든 모델을 항상 GPU Memory에 올릴 필요가 없다.

```text
Model Storage
   ↓
필요한 Model만 Load
   ↓
Inference
   ↓
오랫동안 사용하지 않은 Model은 Unload
```

이 방식은 많은 모델 중 실제로 일부만 자주 호출되는 환경에서 유리하다.

---

## 11. Multi-Model Serving에서 Model Cache가 필요한 이유

모델을 요청마다 Storage에서 다시 다운로드하고 Memory에 Load한다면 Latency가 매우 커진다.

따라서 자주 사용하는 모델은 Memory에 유지한다.

```text
Model Cache
├── Model A
├── Model C
└── Model F
```

새로운 Model B 요청이 왔다고 가정한다.

### Model B가 이미 Cache에 있는 경우

```text
Request B
   ↓
Cache Hit
   ↓
바로 Inference
```

### Model B가 Cache에 없는 경우

```text
Request B
   ↓
Cache Miss
   ↓
Model Storage에서 Load
   ↓
Memory에 적재
   ↓
Inference
```

두 번째 경우에는 **Cold Start**가 발생한다.

---

## 12. LRU Cache와 Model Unload

GPU Memory는 무한하지 않다.

따라서 새로운 모델을 Load하기 위한 공간이 부족하면 기존 모델 중 일부를 내려야 한다.

교재에서는 이런 Resource 관리에 LRU(Least Recently Used) 방식의 Cache 개념을 설명한다.

```text
현재 Cache

Model A : 최근 사용
Model B : 최근 사용
Model C : 오래전에 사용

새로운 Model D Load 필요
Memory 부족

→ Model C Unload
→ Model D Load
```

LRU는 **가장 오랫동안 사용되지 않은 모델을 먼저 제거**하는 방식이다.

이 구조는 Cost Efficiency는 높지만 Model Load/Unload가 자주 발생하면 Latency가 증가할 수 있다.

---

## 13. Multi-Model Routing

Multi-Model Serving에서는 Request를 아무 Container에나 보낼 수 없다.

예를 들어 다음 상태라고 하자.

```text
Container 1
Cache: Model A, B

Container 2
Cache: Model C, D

Container 3
Cache: Model A, D
```

Model A 요청을 Container 2로 보내면 Model A가 없기 때문에 새로 Load해야 한다.

반면 Container 1이나 3으로 보내면 즉시 실행할 수 있다.

```text
Model A Request
      ↓
Router
      ↓
Model A가 이미 Load된 Container 선택
      ↓
Container 1 또는 3
```

따라서 Multi-Model Serving에서는 **Model-aware Routing**이 중요하다.

Router는 어떤 Model이 어떤 Host 또는 Container에 올라가 있는지 Map을 관리할 수 있다.

```text
Model A → Container 1, 3
Model B → Container 1
Model C → Container 2
Model D → Container 2, 3
```

---

## 14. Model Replica와 Per-Model Scaling

모든 모델이 동일한 Traffic을 받는 것은 아니다.

```text
Model A : 10,000 RPS
Model B : 10 RPS
Model C : 100 RPS
```

Model A처럼 많이 사용되는 모델은 Replica를 더 많이 유지해야 한다.

```text
Model A replicas = 3
Model B replicas = 1
Model C replicas = 1
```

따라서 Multi-Model Serving에서는 Container 전체를 Scaling하는 것뿐 아니라 **모델별 Replica 수를 조정하는 문제**도 중요해진다.

이 때문에 Multi-Model Architecture는 Single-Model보다 Resource 효율은 좋아질 수 있지만 운영 복잡도는 크게 증가한다.

---

## 15. Cost 최적화형과 Latency 최적화형 Multi-Model Design

Multi-Model Serving은 무엇을 우선하느냐에 따라 설계가 달라진다.

### Cost Efficiency를 우선하는 경우

가능한 한 많은 모델이 적은 Resource를 공유한다.

```text
많은 Models
    ↓
Shared GPU / Memory Pool
    ↓
필요할 때 Load / Unload
```

장점:

- GPU Resource 활용률 향상
- 유휴 Model에 대한 비용 감소

단점:

- Cold Start
- Model Swap
- Latency 증가 가능

### Latency를 우선하는 경우

자주 쓰는 Model은 미리 충분한 Replica를 유지한다.

```text
Hot Model A → Replica 1, 2, 3
Hot Model B → Replica 1, 2
```

장점:

- Cold Start 감소
- 낮은 Latency

단점:

- 실제 Traffic보다 많은 Resource를 미리 할당할 수 있음
- 비용 증가

즉 Model Serving Architecture에는 하나의 정답이 있는 것이 아니라 **Latency, Throughput, Cost, Operational Complexity 사이의 Trade-off**가 있다.

---

## 16. Multi-Model Serving은 LLM에도 적용될 수 있다

LLM은 GPU Memory 요구량이 커서 일반적으로 Single-Model Serving이 많이 사용된다.

하지만 교재는 Multi-Model Serving 개념이 LLM에서도 여전히 중요하다고 설명한다.

대표적인 사례는 다음과 같다.

### Prefix Cache Routing

공통 Prompt Prefix를 사용하는 Request를 동일 Replica로 보내 이미 계산된 KV Cache를 재사용할 수 있다.

```text
공통 Prefix Request
       ↓
KV Cache가 존재하는 Replica로 Routing
       ↓
중복 계산 감소
```

### Multi-LoRA Serving

하나의 Base Model 위에 여러 LoRA Adapter를 동적으로 Load할 수 있다.

```text
Base LLM
├── LoRA A
├── LoRA B
├── LoRA C
└── LoRA D
```

Base Model 전체를 고객마다 복제하는 것보다 Memory를 효율적으로 사용할 수 있다.

---

# 17. Chapter 3 전체 연결

Chapter 3의 흐름을 하나로 연결하면 다음과 같다.

```text
Chapter 2
LLM inference 내부 이해
  ↓
Token Generation / KV Cache / Batching / Streaming
  ↓

Chapter 3
이 inference를 실제 Web Service로 만들기
  ↓
API Server
  ↓
Workload Manager
  ↓
Model Executor
  ↓
Model Worker
  ↓
Concurrent Request
  ↓
Batch / Streaming / Request Tracking
  ↓
직접 구현의 복잡성 확인
  ↓
vLLM 같은 Production Serving Framework 필요성 이해
  ↓
Single-Model Serving
  ↓
Multi-Model Serving
  ↓
Routing / Cache / LRU / Replica / Cost-Latency Trade-off
```

---

## Chapter 3을 읽고

Chapter 2까지는 vLLM, Batching, Streaming 같은 기능을 주로 **LLM inference 최적화 기능**이라고 생각하기 쉬웠다. Chapter 3에서는 이런 기능들이 실제로는 API Server와 Worker 사이에서 여러 사용자의 Request를 관리하기 위한 **Serving System의 구성요소**라는 점이 더 명확하게 연결된다.

특히 Model Serving은 단순히 "모델을 GPU에 올리고 API 하나를 열어주는 작업"이 아니었다. 실제 환경에서는 Request Queue, Scheduling, Batching, Streaming, Worker 관리처럼 일반적인 Backend/Distributed System 문제와 GPU Resource 관리 문제가 함께 존재한다.

또 vLLM 같은 Serving Framework가 편리한 이유도 단순히 모델 실행 코드가 짧아져서가 아니라, 직접 구현하면 상당히 복잡한 Request Scheduling과 Batch 구성, KV Cache 관리 등을 내부에서 처리하기 때문이라는 점을 이해할 수 있었다.

Single-Model과 Multi-Model도 단순히 모델 개수 차이가 아니라 운영 목표가 달랐다. Single-Model은 성능과 격리, 운영 단순성이 강점이고, Multi-Model은 Resource 공유와 비용 효율성이 강점이다. 대신 Multi-Model에서는 Model Cache, LRU, Cold Start, Model-aware Routing처럼 새로운 문제가 생긴다.

결국 Serving Architecture는 기술 하나를 선택하는 문제가 아니라 **Latency, Throughput, Cost, Resource Utilization, Operational Complexity 중 무엇을 우선할 것인지 결정하는 문제**라는 점이 가장 중요하게 느껴졌다.


<br/><br/>

---

# Chapter 4. LLM Serving Best Practices

Chapter 3에서는 Single-Model과 Multi-Model Serving System을 직접 설계하면서 하나의 Serving Service 내부가 어떻게 동작하는지를 살펴봤다.

Chapter 4에서는 다시 시야를 넓혀 **실제 Production 환경에서 LLM Serving Platform을 어떻게 구성하고 운영할 것인지**를 다룬다.

이번 장의 핵심 영역은 크게 다음 네 가지다.

```text
1. Agentic Application에서 Model Serving이 어떻게 사용되는가
2. Enterprise급 LLM Serving Architecture는 어떻게 구성되는가
3. 직접 구축과 Cloud Vendor 중 무엇을 선택할 것인가
4. Serving 성능을 어떻게 측정하고 비교할 것인가
```

즉 Chapter 3이 Serving Engine과 Service 내부 구조에 가까웠다면 Chapter 4는 **Platform Architecture와 운영 전략**에 가깝다.

---

## 1. Model Serving in an Agentic World

기존의 일반적인 Model Serving은 다음과 같이 생각할 수 있다.

```text
User Request
    ↓
Model
    ↓
Prediction
    ↓
Response
```

하지만 Agent Application에서는 LLM이 한 번만 호출되지 않는다.

```text
User Request
    ↓
LLM Reasoning
    ↓
Information Retrieval
    ↓
LLM Reasoning
    ↓
Tool Execution
    ↓
Result
    ↓
LLM Reasoning
    ↓
Final Answer
```

즉 하나의 사용자 요청이 여러 번의 Model Serving Request를 발생시킬 수 있다.

이 때문에 Agent 시대에는 Model Serving의 Latency, Throughput, Reliability, Cost가 더욱 중요해진다.

---

## 2. Agent는 여러 종류의 Model과 Tool을 사용한다

Agent는 보통 하나의 LLM만 사용하는 시스템이 아니다.

교재에서 설명하는 전형적인 구성요소는 다음과 같다.

- LLM: Reasoning, Planning, Dialogue Generation
- Embedding Model: Retrieval, Similarity Search, Semantic Matching
- Vision/Speech Model: Multimodal Input 처리
- Task-specific Model: Classification, Summarization 등
- External API
- Database
- Search Service
- Business Application

즉 실제 Agent는 다음처럼 움직일 수 있다.

```text
User
  ↓
Agent
  ├── LLM Service
  ├── Embedding Service
  ├── Vector DB
  ├── Search API
  ├── Business API
  └── Other Model Services
```

각 모델과 도구가 독립적인 HTTP/gRPC Service로 제공될 수 있으며 Agent가 필요할 때 호출한다.

따라서 Agent Platform의 성능은 LLM 하나의 성능뿐 아니라 **여러 Serving Service를 연결한 전체 Pipeline의 성능**에 영향을 받는다.

---

## 3. Tool Calling

Modern Agent의 핵심 기능 중 하나는 Tool Calling이다.

교재에서 설명하는 흐름은 네 단계로 정리할 수 있다.

```text
1. LLM이 User Request와 사용 가능한 Tool을 분석
        ↓
2. 호출할 Tool과 Parameter를 구조화된 형태(JSON 등)로 생성
        ↓
3. Agent가 실제 Tool 실행
        ↓
4. Tool Result를 다시 LLM에 전달
        ↓
필요하면 반복
```

예를 들어 다음과 같은 요청이 있다고 하자.

```text
"이번 달 매출을 확인하고 지난달과 비교해줘."
```

Agent는 단순히 언어적으로 답을 생성하는 것이 아니라 다음과 같이 움직일 수 있다.

```text
LLM
 ↓
Sales Database 조회 Tool 선택
 ↓
DB Query 실행
 ↓
결과 반환
 ↓
LLM이 비교/분석
 ↓
최종 답변
```

이처럼 Agent는 **Reasoning과 실제 시스템 실행을 결합**한다.

---

## 4. RAG — Retrieval-Augmented Generation

RAG는 LLM이 답변하기 전에 외부 Knowledge Source에서 관련 정보를 검색하여 Prompt에 추가하는 방식이다.

```text
User Question
     ↓
Embedding
     ↓
Vector Search
     ↓
Relevant Documents
     ↓
Prompt에 추가
     ↓
LLM
     ↓
Answer
```

RAG가 필요한 이유는 LLM의 학습 데이터가 항상 최신이 아니며, 회사 내부 문서나 Private Data를 기본적으로 알지 못하기 때문이다.

RAG를 사용하면 외부 Knowledge를 검색해서 현재 Prompt Context에 넣을 수 있다.

장점:

- 최신 정보 사용 가능
- 사내 문서 등 Private Knowledge 활용
- 답변 Grounding 강화

대신 Retrieval 단계가 추가되므로 전체 Pipeline은 길어진다.

```text
Retrieval Latency
+
Prompt 증가
+
LLM Inference Latency
```

즉 RAG는 단순한 Application Pattern이 아니라 Serving 성능에도 직접 영향을 준다.

---

## 5. CAG — Cache-Augmented Generation

교재에서는 RAG와 함께 CAG(Cache-Augmented Generation)를 소개한다.

CAG는 자주 사용하는 Knowledge를 매번 Retrieval하는 대신 Context 또는 KV Cache에 미리 준비해두고 재사용하는 방향으로 이해할 수 있다.

```text
Knowledge
   ↓
Context / KV Cache에 준비
   ↓
반복 Request에서 재사용
```

교재의 핵심 구분은 다음과 같다.

### RAG

목표:

- Knowledge Grounding
- Freshness
- 필요한 정보를 동적으로 검색

### CAG

목표:

- Latency 감소
- Throughput 향상
- 중복 KV Cache 계산 감소
- Cost Efficiency 향상

따라서 둘은 서로 경쟁하는 기술이 아니다.

```text
RAG
→ 무엇을 Prompt에 넣을 것인가?

CAG
→ 반복되는 Context 계산을 어떻게 효율적으로 재사용할 것인가?
```

Production Agent에서는 둘을 함께 사용할 수도 있다.

```text
RAG로 필요한 Knowledge 검색
        ↓
반복되는 Prefix / Context는 CAG 형태로 Cache 재사용
```

다만 큰 Context Window와 Cache 자체도 GPU Memory와 Compute Resource를 요구하므로 CAG 역시 무조건 공짜인 최적화는 아니다.

---

# 6. Enterprise LLM Serving은 단순한 Model Endpoint가 아니다

작은 PoC에서는 다음 정도만 있어도 동작할 수 있다.

```text
Application
    ↓
LLM API
```

하지만 Enterprise 환경에서는 이것만으로 부족하다.

실제 Production에서는 다음 문제들을 함께 해결해야 한다.

- Authentication
- Authorization
- API Management
- Networking
- Resource Management
- GPU Scheduling
- Model Deployment
- Model Versioning
- Observability
- Monitoring
- Experimentation
- Cost Management
- Reliability
- On-call Operation
- Security
- Governance

즉 Model Serving은 하나의 Model Server가 아니라 **Platform Engineering 문제**가 된다.

---

## 7. Layered Enterprise Serving Architecture

교재는 Enterprise Serving System을 Layered Architecture로 바라본다.

정확한 구현은 조직마다 달라질 수 있지만 개념적으로 다음처럼 이해할 수 있다.

```text
┌──────────────────────────────┐
│ Application / Agent Layer    │
├──────────────────────────────┤
│ API / Gateway / Routing      │
├──────────────────────────────┤
│ Serving / Orchestration      │
├──────────────────────────────┤
│ Model Runtime / Framework    │
├──────────────────────────────┤
│ Resource Management          │
├──────────────────────────────┤
│ Kubernetes / Compute         │
├──────────────────────────────┤
│ GPU / CPU / Network / Storage│
└──────────────────────────────┘
```

Layer를 나누는 핵심 이유는 **각 영역을 독립적으로 발전시키기 위해서**다.

예를 들어 Model Serving Framework를 vLLM에서 다른 Engine으로 바꾸더라도 Application Layer 전체를 수정하지 않는 구조가 좋다.

반대로 GPU Infrastructure가 바뀌더라도 Agent Application은 동일 API를 사용할 수 있어야 한다.

이런 Decoupling이 Enterprise Architecture에서 중요하다.

---

## 8. Open Source Kubernetes Stack

교재는 Production Serving을 구현할 수 있는 Open Source 기반 Stack의 중요한 기반으로 Kubernetes를 다룬다.

Kubernetes를 사용하는 이유는 Model Serving에서도 일반 Application과 마찬가지로 다음 기능들이 필요하기 때문이다.

- Container Scheduling
- Replica 관리
- Service Discovery
- Rolling Update
- Health Check
- Resource Request / Limit
- Autoscaling
- 장애 복구

LLM Serving에서는 여기에 GPU Resource가 추가된다.

```text
Kubernetes Cluster

Node A
├── GPU
└── Model Serving Pod

Node B
├── GPU
└── Model Serving Pod

Node C
└── API / Routing Pod
```

즉 Kubernetes는 모델을 직접 추론하는 Framework가 아니라 **Serving Components와 GPU Workload를 배치하고 운영하는 Orchestration Layer**다.

이 부분은 이후 과제의 KubeRay/RayService와 직접 연결해서 볼 수 있다.

```text
Kubernetes
   ↓
KubeRay Operator
   ↓
Ray Cluster
   ↓
Ray Serve
   ↓
LLM Serving Application
```

---

# 9. Build vs Cloud는 이분법이 아니다

LLM Serving Platform을 구축할 때 흔히 다음처럼 생각하기 쉽다.

```text
직접 구축(Self-hosting)
        VS
Cloud Managed Service
```

하지만 교재에서는 이것을 **Binary Choice가 아니라 Spectrum**으로 본다.

```text
완전 Managed
    │
    │  일부 Customization
    │
    │  Managed Infrastructure + Self-managed Serving
    │
    │  Kubernetes + Open Source Serving Framework
    │
    │
완전 Self-hosted
```

실제 기업은 보통 중간에 위치한다.

예를 들어 Infrastructure는 EKS를 사용하지만 Serving Engine은 vLLM을 직접 운영할 수 있다.

```text
AWS Managed Infrastructure
        ↓
EKS
        ↓
직접 관리하는 KubeRay / Ray Serve / vLLM
```

이런 구조는 Cloud와 Self-hosting을 혼합한 형태다.

---

## 10. 선택 기준

어떤 방식을 사용할지는 다음 요소에 따라 달라진다.

### 개발 속도

초기 PoC에서는 Managed Service가 빠르다.

```text
빠른 개발
→ Vendor API / Managed Endpoint
```

### Traffic 규모

Traffic이 증가할수록 API 사용 비용이 커질 수 있고, 직접 최적화한 Serving Infrastructure가 유리해질 수 있다.

### Cost

단순 Instance 비용만 볼 것이 아니라 다음을 함께 봐야 한다.

```text
GPU Cost
+ Engineering Cost
+ Operation Cost
+ Idle Resource Cost
+ Managed Service Premium
```

### Security / Data Governance

민감 데이터를 외부 Vendor로 전송할 수 없는 조직에서는 Self-hosted 또는 Private Cloud 방식이 필요할 수 있다.

### Customization

특수한 Quantization, Serving Engine, Model Routing, Custom Scheduler 등이 필요하다면 직접 관리 영역이 증가한다.

### 운영 역량

직접 구축은 자유도가 높지만 다음 역량이 필요하다.

- Kubernetes
- GPU Infrastructure
- Distributed System
- Monitoring
- On-call
- Serving Framework

따라서 "직접 구축이 무조건 싸다" 또는 "Managed가 무조건 좋다"고 단정할 수 없다.

---

# 11. AWS Serving Options

교재에서는 AWS 환경을 예로 들어 여러 Serving 선택지를 비교한다.

핵심은 상품명을 외우는 것보다 **관리 범위와 Customization 수준이 서로 다르다**는 점이다.

개념적으로 다음 스펙트럼으로 이해하면 된다.

```text
높은 Managed 수준
   ↓
Foundation Model API / Managed AI Service
   ↓
Managed Model Endpoint
   ↓
Managed Kubernetes + Custom Serving
   ↓
VM / GPU Instance + Custom Serving
   ↓
완전 Self-managed

높은 Customization 수준
```

Managed 영역이 커질수록 개발과 운영 부담은 줄어드는 반면, 세밀한 최적화와 Cost Control 범위는 줄어들 수 있다.

반대로 Self-managed 영역이 커질수록 자유도는 높아지지만 운영 복잡성이 증가한다.

이 관점이 이후 AWS EKS + Ray Serve 과제와 직접 연결된다.

EKS 과제는 완전 Managed LLM API를 호출하는 것이 아니라 다음 구조에 가깝다.

```text
AWS가 Kubernetes Control Plane 제공
        ↓
EKS
        ↓
사용자가 KubeRay/Ray Serve 구성
        ↓
사용자가 LLM Serving Stack 운영
```

즉 Cloud Infrastructure는 활용하지만 Serving Layer는 상당 부분 직접 구성하는 방식이다.

---

# 12. Performance Measurement가 중요한 이유

Serving System을 최적화하려면 먼저 **무엇을 측정할지 정확히 정의**해야 한다.

단순히 "빠르다"고 말하는 것은 의미가 없다.

예를 들어 다음 두 시스템을 비교한다고 하자.

```text
System A
첫 Token은 빠름
전체 생성은 느림

System B
첫 Token은 조금 느림
전체 Token 생성 속도는 빠름
```

어떤 것이 더 좋은지는 Use Case에 따라 다르다.

Chat에서는 First Token이 빠른 것이 중요할 수 있고, Batch Inference에서는 전체 Throughput이 더 중요할 수 있다.

---

## 13. Latency Metrics

Latency는 단순히 하나의 숫자로만 보면 부족하다.

LLM은 Prefill과 Decode 특성이 다르고 Token을 Streaming하기 때문이다.

대표적으로 다음 개념을 구분해 이해해야 한다.

### End-to-End Latency

Request를 보낸 시점부터 전체 Response가 완료될 때까지 걸린 시간이다.

```text
Request Start
     ↓
Prefill
     ↓
Decode
     ↓
Last Token
     ↓
Response Complete
```

### Time to First Token (TTFT)

Request를 보낸 뒤 **첫 번째 Token을 받을 때까지 걸린 시간**이다.

```text
Request
   ↓
[Waiting + Prefill + Scheduling]
   ↓
First Token
```

Chatbot User Experience에서 매우 중요하다.

### Time per Output Token / Inter-Token 관련 지표

첫 Token 이후 Token이 얼마나 빠르게 계속 생성되는지도 중요하다.

첫 Token은 빨라도 이후 Token 생성이 느리면 답변이 끊겨 보일 수 있다.

따라서 LLM Serving에서는 "전체 요청 Latency" 하나만 측정하기보다 **Prefill과 Decode 특성을 반영한 세부 지표**를 함께 보는 것이 중요하다.

---

# 14. Throughput Metrics

Throughput은 일정 시간 동안 얼마나 많은 작업을 처리할 수 있는지를 나타낸다.

일반적인 Request 기반 시스템에서는 다음과 같이 표현할 수 있다.

```text
Requests / Second (RPS)
```

LLM에서는 Token 생성량이 중요하기 때문에 Token 기준 Throughput도 많이 사용한다.

```text
Tokens / Second (TPS)
```

하지만 TPS도 Context를 봐야 한다.

예를 들어 Batch Size를 매우 크게 하면 전체 TPS는 증가할 수 있지만 개별 사용자의 Latency가 증가할 수 있다.

```text
Batch Size ↑
      ↓
GPU Utilization ↑
      ↓
Throughput ↑

하지만

Queue / Waiting Time ↑ 가능
      ↓
Latency ↑ 가능
```

따라서 Throughput과 Latency를 따로 최적화하는 것이 아니라 **Trade-off를 함께 측정**해야 한다.

---

# 15. Benchmark는 실제 Traffic과 비슷해야 한다

Serving Benchmark를 할 때 단순히 동일 Prompt 하나를 반복하는 것만으로 Production 성능을 판단하면 안 된다.

실제 Traffic은 다음처럼 다양할 수 있다.

- Prompt 길이 다름
- Output Token 길이 다름
- Request Arrival Pattern 다름
- 동시 사용자 수 다름
- Peak Traffic 존재
- Streaming / Non-streaming 혼재

따라서 실제 User Workload와 비슷한 Traffic Pattern을 만들어야 의미 있는 결과를 얻을 수 있다.

---

## 16. 한 번에 하나의 변수만 변경하기

Serving Optimization은 설정 간 상호작용이 많다.

예를 들어 동시에 다음을 바꿨다고 하자.

```text
Batch Size 변경
GPU Memory Utilization 변경
Quantization 변경
Model Version 변경
```

성능이 좋아졌더라도 어떤 변경이 효과를 냈는지 알 수 없다.

따라서 실험에서는 가능한 한 **한 번에 하나의 Knob만 변경**해야 한다.

```text
Baseline
 ↓
Batch Size만 변경
 ↓
측정
 ↓
다음 변수 변경
```

이 방식이 있어야 Optimization의 인과관계를 파악할 수 있다.

---

## 17. Hardware Utilization을 함께 봐야 한다

Latency와 Throughput만 보면 병목 원인을 알기 어렵다.

따라서 다음 Hardware Metrics를 함께 확인해야 한다.

- GPU Utilization
- GPU Memory Usage
- CPU Usage
- System Memory
- Network

예를 들어 Throughput이 낮다고 해서 무조건 GPU 성능이 부족한 것은 아니다.

```text
GPU Utilization 30%
Throughput 낮음
```

이라면 GPU 자체가 느린 게 아니라 Request Scheduling이나 Batch 구성이 비효율적일 가능성이 있다.

반대로 GPU Utilization이 계속 100%라면 Compute Bottleneck일 가능성을 생각할 수 있다.

---

## 18. Performance Metric을 인위적으로 부풀리면 안 된다

Benchmark 조건을 유리하게 만들면 숫자는 좋아 보일 수 있다.

예를 들어 실제 Production에서는 긴 Prompt가 많은데 짧은 Prompt만 가지고 Benchmark하면 TPS가 높게 나올 수 있다.

하지만 실제 서비스에서는 그 성능이 나오지 않는다.

따라서 Benchmark는 좋은 숫자를 만들기 위한 작업이 아니라 **실제 시스템의 Capacity와 Bottleneck을 정확하게 이해하기 위한 작업**이어야 한다.

---

## 19. Production Monitoring과 Regression Test

성능 측정은 배포 전에 한 번 하고 끝나는 작업이 아니다.

Traffic Pattern과 User Behavior는 계속 변할 수 있다.

```text
배포
 ↓
Monitoring
 ↓
Regression Detection
 ↓
Optimization
 ↓
재배포
 ↓
Monitoring
```

교재는 Production에서도 지속적으로 성능을 관찰하고 정기적으로 Test Suite를 실행해야 한다고 설명한다.

대표적인 테스트는 다음과 같다.

### Regression Test

새로운 Version이 이전보다 Latency나 Throughput을 악화시키지 않는지 확인한다.

### Scalability Test

Peak Traffic에서 시스템이 얼마나 확장되는지 확인한다.

### A/B Test

새로운 Optimization 기법을 기존 설정과 공정하게 비교한다.

---

# 20. Chapter 4 전체 연결

이번 장을 하나의 흐름으로 정리하면 다음과 같다.

```text
Chapter 3
Model Serving Service 내부 구조 이해
        ↓

Chapter 4
실제 Production Platform으로 확장
        ↓
Agentic Application
        ↓
LLM + Embedding + Tool + RAG
        ↓
하나의 요청이 여러 Model Serving Call 발생
        ↓
Latency / Throughput / Cost의 중요성 증가
        ↓
Enterprise Layered Architecture
        ↓
Kubernetes / Resource Management / Observability
        ↓
Build vs Cloud 선택
        ↓
Managed와 Self-hosted의 Spectrum
        ↓
성능 측정
        ↓
Latency / TTFT / Throughput / Hardware Utilization
        ↓
Benchmark / Regression / Scalability Test
```

---

## Chapter 4를 읽고

Chapter 3까지는 Model Serving을 하나의 Server 또는 Framework 관점에서 많이 봤다면, Chapter 4에서는 실제 기업 환경에서는 **모델 하나를 잘 실행하는 것만으로는 Production Serving이 완성되지 않는다**는 점이 더 분명해졌다.

특히 Agent Application에서는 하나의 User Request가 LLM 한 번 호출로 끝나는 것이 아니라 Retrieval, Embedding, Tool Calling, 다시 LLM 호출 같은 여러 단계를 거칠 수 있다. 따라서 각 Model Endpoint가 조금씩 느리면 전체 Agent Workflow에서는 그 Latency가 계속 누적된다. Agent 시대에 Serving 성능이 더 중요해지는 이유를 이해할 수 있었다.

RAG와 CAG의 차이도 단순히 두 종류의 LLM Application 기법이 아니라 목표가 다르다는 점이 중요했다. RAG는 Knowledge Grounding과 Freshness 문제를 해결하고, CAG는 반복 Context와 KV Cache 계산을 줄여 Serving Efficiency를 높이는 방향이다. Production에서는 둘을 같이 사용할 수도 있다는 점이 기억에 남았다.

또 Cloud와 Self-hosting을 둘 중 하나로만 결정하는 것이 아니라 Spectrum으로 보는 관점이 현실적이었다. 실제로 EKS 같은 Managed Kubernetes를 사용하면서 그 위에 KubeRay, Ray Serve, vLLM을 직접 운영하는 형태도 Cloud와 Self-hosting이 섞인 구조다.

마지막으로 성능 측정에서 단순 TPS 하나만 높다고 좋은 Serving System이라고 볼 수 없다는 점도 중요했다. Batch Size를 높여 Throughput을 올리면 Latency가 증가할 수 있고, TTFT가 빠르더라도 이후 Token 생성 속도가 느릴 수 있다. 결국 실제 User Traffic과 비슷한 조건에서 Latency, Throughput, GPU Utilization을 같이 봐야 한다.