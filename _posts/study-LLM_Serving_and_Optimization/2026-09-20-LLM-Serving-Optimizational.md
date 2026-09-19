---
layout: post
title: "[LLM Serving] Kubernetes Load Balancing은 왜 LLM Serving에 부족할까? - llm-d Intelligent Routing"
date: 2026-09-19 23:40:00 +0900
categories: [LLM, LLMOps, vLLM, Kubernetes]
tags: [LLM Serving, llm-d, vLLM, Kubernetes, Load Balancing, Inference Routing, KV Cache, EPP, InferencePool]
published: true
---

일반적인 웹 애플리케이션에서는 동일한 역할을 수행하는 여러 서버나 Pod 앞에 Load Balancer를 두고 요청을 분산한다.

Kubernetes에서도 Service를 통해 여러 Pod를 하나의 Endpoint 집합으로 묶고 트래픽을 분산할 수 있다.

하지만 LLM Serving은 일반적인 웹 API와 요청 특성이 다르다.

같은 API 요청이라도 Prompt 길이, 생성할 Output Token 수, 현재 KV Cache 상태, 실행 중인 요청 수에 따라 GPU가 실제로 처리해야 하는 작업량이 크게 달라진다.

따라서 단순히 "Pod가 살아 있는가"만 확인해서 요청을 분산하는 것보다 **각 추론 서버의 현재 상태를 고려해서 요청을 보내는 방식**이 중요해진다.

이번 글에서는 Kubernetes의 일반적인 Load Balancing이 LLM Serving에서 어떤 한계를 가지는지 살펴보고, llm-d가 이를 어떻게 해결하려고 하는지 정리한다.

## 1. 일반적인 Kubernetes Load Balancing

Kubernetes에서 여러 Pod가 동일한 애플리케이션을 실행하고 있다면 보통 Service를 통해 접근한다.

```text
Client
   |
Service
   |
   +---- Pod A
   |
   +---- Pod B
   |
   +---- Pod C
```

Service는 여러 Backend Pod를 하나의 논리적인 Endpoint로 제공한다.

이 구조는 일반적인 웹 서버나 API 서버에서는 충분히 잘 동작한다.

예를 들어 동일한 REST API 서버가 여러 개 실행되고 있다면 요청을 여러 Backend로 분산함으로써 처리량과 가용성을 높일 수 있다.

하지만 여기서 Load Balancer가 알고 있는 정보는 제한적이다.

대체로 다음과 같은 정보는 알 수 있다.

- 어떤 Endpoint가 존재하는가
- Endpoint가 Ready 상태인가
- 연결 가능한가

반면 해당 Backend에서 실행 중인 LLM의 내부 상태까지는 알지 못한다.

예를 들어 다음은 일반적인 Service 레벨에서 직접 알기 어렵다.

- 현재 몇 개의 추론 요청을 처리하고 있는가
- 대기 중인 요청이 몇 개인가
- KV Cache가 얼마나 사용되고 있는가
- 동일한 Prefix가 어느 서버의 KV Cache에 존재하는가
- 특정 요청을 처리했을 때 예상되는 TTFT나 TPOT는 어느 정도인가

즉 **서버가 살아 있다는 것과 지금 이 요청을 보내기에 좋은 서버라는 것은 다른 문제**다.

## 2. LLM Serving은 왜 일반 Web API와 다른가

LLM은 요청 하나의 처리 비용이 일정하지 않다.

예를 들어 다음 두 요청을 비교해볼 수 있다.

```text
요청 A
Prompt: 30 tokens
Output: 20 tokens

요청 B
Prompt: 8,000 tokens
Output: 1,000 tokens
```

둘 다 HTTP 요청 하나이지만 GPU가 수행해야 하는 작업량은 크게 다르다.

LLM은 크게 Prefill과 Decode 단계로 동작한다.

### Prefill

입력 Prompt 전체를 처리하는 단계다.

Prompt가 길수록 처리해야 할 Token이 많아지고 연산량도 증가한다.

### Decode

다음 Token을 하나씩 생성하는 단계다.

출력 Token이 많아질수록 Decode 반복 횟수가 늘어나며 요청이 GPU에 오래 남아 있게 된다.

따라서 단순한 요청 개수만으로 Backend의 부하를 판단하기 어렵다.

```text
Pod A
- Request 2개
- 각각 매우 긴 Context
- GPU 사용량 높음

Pod B
- Request 5개
- 모두 짧은 요청
- 상대적으로 여유 있음
```

이 경우 단순히 "활성 요청 수가 적다"는 이유만으로 Pod A를 선택하는 것도 반드시 좋은 판단은 아니다.

LLM Serving에서는 요청의 특성과 추론 서버 내부 상태를 함께 고려해야 한다.

## 3. KV Cache도 라우팅에 영향을 준다

LLM은 이전 Token에 대한 Attention 계산 결과를 반복 계산하지 않기 위해 KV Cache를 사용한다.

특히 Prefix Caching을 사용하는 경우 여러 요청이 동일한 Prefix를 공유하면 기존 계산 결과를 재사용할 수 있다.

예를 들어 다음과 같은 요청이 있다고 가정한다.

```text
요청 1:
"You are a Kubernetes expert. 다음 질문에 답해줘: ..."

요청 2:
"You are a Kubernetes expert. 다음 질문에 답해줘: ..."
```

두 요청이 긴 System Prompt를 공유한다면 해당 Prefix 계산 결과가 이미 특정 vLLM 서버의 KV Cache에 존재할 수 있다.

```text
Pod A
└─ 해당 Prefix KV Cache 존재

Pod B
└─ 해당 Prefix 없음
```

이 경우 요청 2를 Pod A로 보내면 일부 계산을 재사용할 수 있다.

하지만 요청 2를 Pod B로 보내면 같은 Prefix를 다시 계산해야 한다.

즉 LLM Serving에서는 다음과 같은 질문이 중요해진다.

> "어느 서버가 가장 한가한가?"

뿐만 아니라

> "이 요청을 가장 효율적으로 처리할 수 있는 서버는 어디인가?"

까지 판단해야 한다.

## 4. llm-d

llm-d는 Kubernetes 환경에서 LLM Inference Serving을 확장하기 위한 오픈소스 스택이다.

vLLM이나 SGLang 같은 실제 추론 엔진을 대체하는 것이 아니라, 여러 Model Server를 Kubernetes 환경에서 효율적으로 연결하고 라우팅하는 계층을 제공한다.

전체 구조를 단순화하면 다음과 같다.

```text
Client
   |
   v
Proxy / Gateway
   |
   v
EPP
   |
   v
InferencePool
   |
   +---- vLLM Pod A
   |
   +---- vLLM Pod B
   |
   +---- vLLM Pod C
```

여기서 중요한 구성요소는 Proxy, EPP, InferencePool이다.

## 5. Proxy

Proxy는 실제 요청이 들어오는 데이터 플레인 역할을 한다.

Envoy나 Istio와 같은 L7 Proxy를 사용할 수 있다.

Client의 요청이 Proxy에 도착하면 바로 임의의 Backend로 보내는 것이 아니라, EPP에게 어느 Endpoint를 선택할지 물어볼 수 있다.

개념적으로는 다음과 같다.

```text
Client Request
      |
      v
    Proxy
      |
      | 어느 Backend로 보낼까?
      v
     EPP
      |
      | Pod B 선택
      v
    Proxy
      |
      v
 vLLM Pod B
```

Proxy는 실제 트래픽 전달을 담당하고, Endpoint 선택 로직은 EPP가 담당한다.

## 6. EPP: Endpoint Picker

EPP는 llm-d Router에서 실제 Endpoint 선택을 담당하는 핵심 컴포넌트다.

일반적인 Load Balancer가 Round Robin이나 Least Connection 같은 알고리즘을 사용한다면, EPP는 LLM Serving에 필요한 상태까지 판단 근거로 사용할 수 있다.

예를 들면 다음과 같다.

```text
Pod A
Queue: 8
KV Cache Usage: 90%
Running Requests: 12

Pod B
Queue: 1
KV Cache Usage: 45%
Running Requests: 3

Pod C
Queue: 4
KV Cache Usage: 60%
Running Requests: 6
```

EPP는 이런 정보를 이용해 현재 요청을 어느 Endpoint로 보내는 것이 적절한지 결정한다.

## 7. Filter → Score → Pick

llm-d의 Request Scheduler는 Endpoint를 선택할 때 크게 다음 흐름을 사용한다.

```text
Candidate Endpoints
       |
       v
     Filter
       |
       v
      Score
       |
       v
      Pick
       |
       v
Selected Endpoint
```

### Filter

먼저 요청을 보낼 수 없는 Endpoint를 후보에서 제외한다.

예를 들어 다음과 같은 조건을 사용할 수 있다.

- 특정 Label 조건과 맞지 않는 Endpoint
- Prefill 또는 Decode 역할이 맞지 않는 Endpoint
- SLO 조건을 만족하기 어려운 Endpoint

즉 **"누구에게 보내면 안 되는가"**를 먼저 판단한다.

### Score

남은 Endpoint마다 점수를 계산한다.

llm-d는 다양한 Scorer를 사용할 수 있다.

대표적으로 다음과 같은 정보를 활용할 수 있다.

- KV Cache Utilization
- Queue Depth
- Running Requests
- Prefix Cache 일치 정도
- Session Affinity
- 예상 Latency

예를 들어 Queue가 짧고 KV Cache 여유가 있는 Endpoint는 높은 점수를 받을 수 있다.

또 동일한 Prefix Cache를 가지고 있는 Endpoint를 우선할 수도 있다.

### Pick

최종 점수를 기반으로 실제 Endpoint 하나를 선택한다.

기본적으로 가장 높은 점수를 받은 Endpoint를 선택하거나, 점수에 따라 Weighted Random 방식으로 선택할 수도 있다.

즉 기존의 단순한 Load Balancing보다 **LLM의 실행 상태를 반영한 Scheduling에 가까운 Routing**이 이루어진다.

## 8. InferencePool

InferencePool은 동일한 추론 서비스를 제공하는 Model Server들을 하나의 논리적인 Pool로 묶는다.

예를 들어 같은 Qwen 모델을 실행하는 vLLM Pod가 세 개 있다고 가정한다.

```text
InferencePool: qwen

- vLLM Pod A
- vLLM Pod B
- vLLM Pod C
```

InferencePool은 "누가 Backend 후보인가"를 정의하고, EPP는 그 후보들 중 실제로 어느 Endpoint를 사용할지 결정한다.

즉 역할을 나누면 다음과 같다.

```text
InferencePool
= 후보 Endpoint 집합 관리

EPP
= 후보 중 실제 Endpoint 선택
```

Kubernetes 환경에서는 Label Selector를 사용하여 특정 Model Server Pod들을 하나의 InferencePool에 포함시킬 수 있다.

## 9. 일반 Load Balancing과 llm-d 비교

| 구분 | 일반 Kubernetes Load Balancing | llm-d |
| --- | --- | --- |
| 주요 대상 | 일반 애플리케이션 | LLM Inference |
| Backend 판단 | Endpoint 상태 중심 | Inference 상태까지 고려 |
| Queue 인식 | 제한적 | 가능 |
| KV Cache 인식 | 없음 | 가능 |
| Prefix Cache 활용 | 없음 | 가능 |
| 요청 특성 고려 | 제한적 | 가능 |
| Endpoint 선택 | 일반 LB 방식 | Filter → Score → Pick |
| 목적 | 트래픽 분산 | 추론 효율과 지연시간까지 고려한 Routing |

핵심 차이는 **Backend가 살아 있는지를 보는 것에서 끝나는가, 아니면 Backend가 현재 해당 추론 요청을 얼마나 효율적으로 처리할 수 있는지까지 보는가**에 있다.

## 10. Load-Aware Routing

LLM Serving에서는 특정 서버 하나에 긴 요청이 몰리면 다른 서버가 여유가 있어도 전체 응답 지연시간이 증가할 수 있다.

예를 들어 다음과 같은 상태가 있을 수 있다.

```text
Pod A
████████████████████
Queue가 길고 GPU가 바쁨

Pod B
████
상대적으로 여유 있음
```

일반적인 요청 분산 방식에서는 이 차이를 충분히 반영하지 못할 수 있다.

반면 Load-Aware Routing에서는 각 Endpoint의 Queue나 실행 중인 요청 등의 정보를 활용하여 상대적으로 여유 있는 Endpoint를 선택할 수 있다.

```text
새로운 Request
      |
      v
     EPP
      |
      +-- Pod A : Busy
      |
      +-- Pod B : Available
                 |
                 v
              Pod B 선택
```

이렇게 하면 단순히 요청 개수를 균등하게 나누는 것이 아니라 **실제 추론 부하를 균형 있게 분산하는 것**을 목표로 할 수 있다.

## 11. Prefix-Cache-Aware Routing

Load-Aware Routing보다 한 단계 더 LLM 특화된 방식이 Prefix-Cache-Aware Routing이다.

동일하거나 유사한 Prefix를 가진 요청이 들어왔을 때 기존 KV Cache가 있는 Endpoint를 우선 선택한다.

```text
Request
Prefix = A

Pod A
Prefix A Cache: HIT

Pod B
Prefix A Cache: MISS
```

이 경우 Pod A를 선택하면 이미 계산된 Prefix를 재사용할 가능성이 있다.

결국 Routing 자체가 모델 추론 성능 최적화의 일부가 된다.

## 12. 인프라 관점에서 본 의미

LLM Serving을 공부하면서 처음에는 GPU 개수나 Model Size처럼 하드웨어 자원 자체가 가장 중요한 요소라고 생각하기 쉽다.

하지만 실제 Serving 시스템에서는 단순히 GPU를 추가하는 것만으로 성능 문제가 해결되지 않는다.

GPU가 여러 개 있더라도 요청이 특정 서버에 몰리면 다른 GPU가 놀고 있을 수 있다.

또 Prefix Cache가 여러 서버에 분산되어 있다면 어떤 서버로 요청을 보내느냐에 따라 같은 Prompt를 다시 계산할 수도 있다.

따라서 LLM Serving에서 인프라가 담당해야 하는 범위는 다음처럼 넓어진다.

```text
GPU Resource
      +
Model Server
      +
Scheduling
      +
Caching
      +
Routing
      +
Observability
```

즉 Kubernetes에서 Replica를 늘리고 Service를 붙이는 것만으로 끝나는 것이 아니라, **추론 엔진 내부 상태와 Gateway/Router 계층을 함께 고려해야 한다.**

llm-d가 흥미로운 이유도 이 지점에 있다.

Kubernetes의 기존 Gateway 및 Routing 구조를 활용하면서도 Queue, KV Cache, Prefix Cache와 같은 LLM Serving 특유의 정보를 Endpoint 선택 과정에 반영하려고 한다.

## 마무리

일반적인 Kubernetes Load Balancing은 여러 애플리케이션 인스턴스에 트래픽을 분산하는 데 효과적이다.

하지만 LLM Serving에서는 요청별 작업량이 크게 다르고, KV Cache와 Queue처럼 추론 엔진 내부 상태가 성능에 직접적인 영향을 준다.

따라서 단순히 Backend가 Ready 상태인지 확인하는 것만으로는 최적의 Endpoint를 선택하기 어렵다.

llm-d는 Proxy와 EPP를 분리하고, InferencePool 안의 Model Server들을 대상으로 LLM 특화 상태를 반영해 Endpoint를 선택한다.

특히 `Filter → Score → Pick` 구조를 통해 Queue, KV Cache, Prefix Cache, Latency 등의 정보를 Routing에 활용할 수 있다.

이번 내용을 정리하면서 LLM Serving에서는 **Load Balancing 자체도 모델 추론의 특성을 이해해야 하는 영역**이라는 점이 가장 중요하게 느껴졌다.

이전에 vLLM 내부의 Scheduler나 KV Cache를 모델 서버 한 개의 관점에서 봤다면, llm-d는 이를 여러 Model Server가 존재하는 Kubernetes Cluster 전체의 관점으로 확장한 구조라고 볼 수 있다.

## 참고

- [llm-d Documentation](https://llm-d.ai/docs)
- [llm-d Architecture](https://llm-d.ai/docs/architecture)
- [llm-d Router](https://llm-d.ai/docs/architecture/core/router)
- [llm-d EPP](https://llm-d.ai/docs/architecture/core/router/epp)
- *Hands-On LLM Serving and Optimization*, Chi Wang, Peiheng Hu, O'Reilly, 2026
