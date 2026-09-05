---
layout: post
title: "[LLM Serving] Chapter 10 - The Evolution of LLM Serving"
date: 2026-09-05 23:36:00 +0900
categories: [LLM, LLMOps, vLLM, GPU]
tags: [LLM Serving, Semantic Routing, Semantic Cache, Performance Profiling, Multimodal Serving, vLLM, GPU]
published: true
---

## Chapter 10 개요

앞 장에서는 `KV Cache`, `Continuous Batching`, `Quantization`, `Speculative Decoding`, `Multi-GPU`처럼 **LLM 모델 자체를 얼마나 효율적으로 실행할 것인가**를 중심으로 살펴봤다.

Chapter 10에서는 여기서 범위를 더 넓혀, 앞으로의 LLM Serving 시스템이 어떤 방향으로 발전하고 있는지를 다룬다.

주요 주제는 다음과 같다.

- Semantic Caching / Semantic Routing
- Performance Profiling
- Multimodal Serving
- Edge AI
- Multi-LoRA
- Reinforcement Learning Inference

이 중 인프라 관점에서 직접 연결되는 내용인 **Semantic Routing, Performance Profiling, Multimodal Serving**을 중심으로 정리한다.

---

# 1. Semantic Routing

## 기존 Routing과의 차이

기존 LLM Serving 환경에서 Routing은 주로 여러 Model Replica 중 하나를 선택하는 역할이었다.

```text
Client
  ↓
Load Balancer
  ├─ Model Replica 1
  ├─ Model Replica 2
  └─ Model Replica 3
```

이 경우 Router가 보는 것은 보통 다음과 같은 정보다.

- 현재 Replica의 부하
- 요청 수
- 응답 시간
- 사용 가능한 Backend
- Cache 상태

반면 **Semantic Routing**은 요청의 의미까지 분석해 **어떤 모델 또는 어떤 처리 경로를 선택할지 결정**한다.

```text
User Request
      ↓
Semantic Router
      ├─ Small Model
      ├─ Domain Model
      └─ Large Reasoning Model
```

예를 들어 단순한 질문까지 항상 가장 큰 모델로 보내는 것은 비용 측면에서 비효율적이다.

Semantic Router를 사용하면 요청의 복잡도나 의미를 기준으로 모델을 나눠 사용할 수 있다.

```text
간단한 질문
→ Small Model

특정 업무/도메인 질문
→ Domain Fine-tuned Model

복잡한 Reasoning
→ Large Reasoning Model
```

즉 기존 Routing이 **Replica 선택**에 가까웠다면, Semantic Routing은 **Model Endpoint 선택**까지 범위가 확장된 형태라고 볼 수 있다.

---

## Semantic Cache

기존 Prefix Cache는 입력 Token의 앞부분이 동일하거나 유사할 때 이전 계산 결과를 재사용한다.

```text
[공통 System Prompt] + 질문 A
[공통 System Prompt] + 질문 B
```

반면 Semantic Cache는 문자열이 완전히 같지 않아도 **의미가 비슷한 요청을 판단해 이전 응답을 재사용**한다.

예를 들어 다음 두 문장은 표현은 다르지만 의미는 비슷하다.

```text
"비밀번호를 잊어버렸어."
"로그인 비밀번호를 재설정하고 싶어."
```

Semantic Cache에서는 일반적으로 요청을 Embedding Vector로 변환한 뒤 Vector Search를 수행한다.

```text
User Prompt
     ↓
Embedding Model
     ↓
Embedding Vector
     ↓
Vector Search
     ↓
유사한 기존 요청 존재?
     ├─ Yes → Cached Response 반환
     └─ No  → LLM 호출
```

이를 통해 동일한 의미의 요청에 대해 불필요한 LLM 호출을 줄일 수 있다.

결과적으로 다음 효과를 기대할 수 있다.

- LLM 호출 횟수 감소
- Latency 감소
- Token 사용량 감소
- API 비용 감소
- Backend 부하 감소

---

## Tool Filtering

Semantic Routing은 모델 선택뿐 아니라 Agent 환경의 Tool 선택에도 활용할 수 있다.

Agent가 사용할 수 있는 Tool이 많다고 가정하면 모든 Tool Schema를 LLM Prompt에 전달하는 것은 비효율적이다.

```text
Tool 100개 전체 전달
↓
Prompt Token 증가
↓
Context 증가
↓
Latency 증가
↓
Cost 증가
```

따라서 Router가 먼저 요청과 관련된 Tool만 골라낸 뒤 LLM에 전달할 수 있다.

```text
User Request
     ↓
Semantic Router
     ↓
관련 Tool 검색
     ↓
일부 Tool만 선택
     ↓
LLM에 전달
```

이렇게 하면 Prompt 크기를 줄이고, 모델이 잘못된 Tool을 선택할 가능성도 줄일 수 있다.

Chapter 10에서는 이러한 Routing Layer가 단순한 Load Balancer를 넘어 **보안, 정책, 컴플라이언스 검사를 수행하는 중앙 지점**으로도 확장될 수 있다고 설명한다.

---

## LLM Gateway 관점

이 구조는 LiteLLM이나 vLLM Semantic Router 같은 LLM Gateway와 연결해서 볼 수 있다.

일반적인 구조는 다음과 같다.

```text
Client
  ↓
LLM Gateway / Router
  ├─ Model A
  ├─ Model B
  └─ Model C
```

Gateway가 담당할 수 있는 기능은 다음과 같다.

- Routing
- Load Balancing
- Retry
- Fallback
- Rate Limit
- Authentication
- Usage Tracking
- Semantic Routing
- Caching

즉 LLM Serving에서 Router는 단순히 트래픽을 나누는 장비가 아니라 **요청을 해석하고 적절한 Backend를 선택하는 제어 계층**으로 발전하고 있다.

---

# 2. Performance Profiling

LLM Serving 성능이 좋지 않을 때 단순히 GPU 사용률만 보고 원인을 판단하면 안 된다.

예를 들어 다음과 같은 상황이 있을 수 있다.

```text
GPU Utilization 낮음
+
Throughput 낮음
```

이 경우 GPU 자체 성능이 부족한 것이 아니라 다음 문제일 수 있다.

- CPU 전처리 지연
- Python 실행 오버헤드
- 데이터 로딩 지연
- Scheduler 문제
- Batch가 충분히 구성되지 않음
- Kernel Launch 간격이 큼

즉 **병목은 항상 GPU에 있는 것이 아니다.**

Chapter 10에서는 성능 분석을 크게 세 단계로 나눈다.

```text
Serving Layer
     ↓
Framework Layer
     ↓
Runtime Layer
```

---

## 2.1 Serving Layer

가장 먼저 실제 서비스 수준의 지표를 확인한다.

대표적인 지표는 다음과 같다.

- Throughput
- TTFT
- ITL
- End-to-End Latency
- GPU Utilization
- GPU Memory Usage

예를 들어:

```text
GPU Utilization = 40%
Throughput = 낮음
```

이라면 GPU가 충분히 바쁘지 않은 상태이므로, 바로 CUDA Kernel부터 분석하기보다는 CPU나 Scheduling 쪽 문제를 먼저 의심하는 것이 맞다.

반대로:

```text
GPU Utilization = 높음
Latency = 높음
```

이라면 GPU 내부의 특정 연산이 병목일 가능성이 있다.

Serving Layer의 목적은 **어느 방향으로 더 깊게 분석해야 할지 판단하는 것**이다.

---

## 2.2 Framework Layer

Framework Layer에서는 PyTorch Profiler 같은 도구를 이용해 실제 모델 연산 중 어떤 Operator가 많은 시간을 사용하는지 확인한다.

예를 들어 Transformer 모델에서는 다음과 같은 연산이 존재한다.

```text
Attention
Matmul
LayerNorm
Embedding
Softmax
```

Profiler 결과가 다음과 같다고 가정하면:

```text
전체 Latency = 100 ms

Attention = 55 ms
Matmul    = 25 ms
LayerNorm = 5 ms
기타       = 15 ms
```

Attention이 전체 실행 시간에서 가장 큰 비중을 차지하므로, 해당 부분을 우선적으로 확인하는 것이 합리적이다.

PyTorch Profiler를 통해 다음을 확인할 수 있다.

- CPU Operator 실행 시간
- CUDA Operator 실행 시간
- CPU → GPU 데이터 이동
- Operator별 실행 비율
- 연산과 I/O가 겹쳐 실행되는지 여부

---

## 2.3 Runtime Layer

Framework 수준에서 특정 Operator가 병목이라는 것을 확인했다면 더 아래 Runtime Layer로 내려갈 수 있다.

이 단계에서 대표적으로 사용하는 도구가 NVIDIA Nsight이다.

### Nsight Systems

Nsight Systems는 시스템 전체 Timeline을 확인하는 데 사용한다.

확인할 수 있는 예시는 다음과 같다.

- GPU가 실제로 Idle 상태인 구간
- CPU가 Kernel Launch를 늦게 수행하는지
- Memory Copy와 Compute가 겹치는지
- Kernel 사이에 긴 Gap이 있는지
- CPU와 GPU 사이의 동기화 문제

예를 들어 GPU Utilization이 높아 보이더라도 실제 Timeline에는 다음과 같은 Gap이 존재할 수 있다.

```text
Kernel
████████

       Idle

Kernel
████████

       Idle

Kernel
████████
```

이런 경우 단순한 GPU 연산 성능 문제가 아니라 Kernel Launch나 데이터 전달 과정이 병목일 수 있다.

---

### Nsight Compute

Nsight Compute는 특정 CUDA Kernel 내부를 더 자세히 분석하는 도구다.

대표적으로 다음을 확인한다.

- Occupancy
- Memory Stall
- Tensor Core 활용률
- Shared Memory 사용
- Global Memory 접근 패턴

즉:

```text
Serving Metric
    ↓
어느 Operator가 느린가?
    ↓
어느 CUDA Kernel이 느린가?
    ↓
그 Kernel은 왜 느린가?
```

순서로 점점 원인을 좁혀간다.

---

## Profiling의 핵심 흐름

Chapter 10의 Profiling 방법을 단순화하면 다음과 같다.

```text
Nsight Systems
     ↓
GPU가 실제 병목인지 확인
     ↓
GPU Utilization 낮음
→ PyTorch Profiler CPU Mode
→ Host / Preprocessing / Python Overhead 확인

GPU는 바쁜데 Latency 높음
→ PyTorch Profiler CUDA Mode
→ 어떤 Operator가 병목인지 확인

특정 Kernel이 대부분의 시간 차지
→ Nsight Compute
→ Occupancy / Memory Stall 분석
```

핵심은 처음부터 가장 낮은 수준의 Kernel 분석으로 들어가는 것이 아니라, **상위 지표부터 시작해서 병목 범위를 좁혀가는 방식**이다.

---

# 3. Multimodal Serving

기존 LLM Serving은 대부분 Text Input을 기준으로 설명할 수 있었다.

하지만 VLM(Vision-Language Model)처럼 이미지와 텍스트를 동시에 처리하는 모델에서는 Serving Pipeline이 더 복잡해진다.

예를 들어 사용자가 이미지와 질문을 함께 입력한다고 가정한다.

```text
Image
+
"이 이미지에 무엇이 있는지 설명해줘."
```

VLM에서는 이미지를 그대로 LLM에 넣지 않는다.

이미지는 먼저 Vision Encoder를 통해 처리된다.

```text
Image
  ↓
Image Preprocessing
  ↓
Vision Encoder
  ↓
Vision Embedding
  ↓
LLM
  ↓
Text Output
```

---

## Vision Input 처리

이미지는 내부적으로 여러 Patch로 나뉜다.

```text
Image
  ↓
Patch 분할
  ↓
각 Patch를 Vector로 변환
  ↓
Vision Embedding 생성
```

이 Vision Embedding은 LLM의 Hidden Dimension에 맞게 변환된 뒤 Text Embedding과 함께 모델에 입력된다.

결국 LLM 입장에서는 다음과 같은 하나의 긴 Embedding Sequence처럼 처리된다.

```text
Text Embedding
Vision Embedding
Text Embedding
```

---

## Multimodal Serving의 새로운 병목

텍스트 Tokenization은 상대적으로 가벼운 작업이다.

하지만 Image Input에는 다음과 같은 전처리가 필요할 수 있다.

```text
Image Decode
↓
Resize
↓
Crop
↓
Color Conversion
↓
Tensor Transform
↓
Normalize
```

이 작업 중 상당 부분은 CPU에서 수행된다.

따라서 요청량이 많아지면 다음과 같은 문제가 발생할 수 있다.

```text
Image Request 증가
      ↓
CPU Preprocessing 증가
      ↓
CPU Saturation
      ↓
GPU에 입력 전달 지연
      ↓
GPU Idle
      ↓
Throughput 감소
```

즉 GPU가 충분히 빠르더라도 **CPU가 데이터를 준비하지 못하면 전체 LLM Serving 성능이 떨어질 수 있다.**

이것이 Chapter 10에서 강조하는 `Bottleneck Shifting`의 대표적인 예다.

---

# 4. vLLM V0 → V1 Process Separation

Chapter 10에서는 Multimodal Serving에서 CPU 병목을 줄이기 위한 사례로 vLLM V1 구조를 설명한다.

기존 구조에서는 API 처리, Input Preprocessing, GPU Scheduling이 서로 영향을 줄 수 있었다.

```text
API Server
+
Preprocessing
+
GPU Scheduling
```

CPU에서 Multimodal Preprocessing이 오래 걸리면 GPU Scheduling까지 영향을 받아 GPU가 기다리는 상황이 발생할 수 있다.

vLLM V1에서는 CPU 중심 작업과 GPU 실행을 별도 Process로 분리한다.

```text
Process 0
- API Server
- Input Preprocessing
- Output Postprocessing

Process 1
- GPU Scheduling
- Kernel Launch
```

핵심은 **CPU 작업과 GPU 실행을 Decouple하는 것**이다.

CPU가 이미지 전처리를 수행하는 동안 GPU 실행 Process는 독립적으로 동작할 수 있기 때문에, CPU 작업이 GPU Kernel Launch를 직접 Blocking하는 상황을 줄일 수 있다.

이 구조는 단순히 멀티모달에만 적용되는 개념이라기보다 LLM Serving 시스템에서 자주 등장하는 설계 패턴으로 볼 수 있다.

```text
느린 작업
       ↓
Critical Path에서 분리
       ↓
비동기 처리
       ↓
CPU / GPU 작업 Overlap
       ↓
Resource Utilization 향상
```

---

# 5. Bottleneck Shifting

이번 Chapter에서 반복해서 등장하는 중요한 개념 중 하나가 **병목 이동(Bottleneck Shifting)**이다.

LLM Serving에서는 하나의 병목을 해결하면 다른 부분이 새로운 병목이 될 수 있다.

예를 들어:

```text
GPU Compute 최적화
       ↓
GPU 처리 속도 증가
       ↓
CPU가 데이터를 공급하지 못함
       ↓
CPU가 새로운 Bottleneck
```

또는:

```text
Model Quantization
       ↓
GPU Memory 여유 증가
       ↓
Concurrency 증가
       ↓
Scheduler / Network / CPU 부하 증가
```

따라서 특정 지표 하나만 보고 계속 같은 영역을 튜닝하면 안 된다.

최적화 이후에는 다시 전체 시스템을 측정해야 한다.

```text
Measure
  ↓
Bottleneck 확인
  ↓
Optimize
  ↓
Measure Again
  ↓
새로운 Bottleneck 확인
```

이 방식은 앞 Chapter의 LLM Optimization 실습과도 연결된다.

---

# 6. LLM Serving Engine의 역할 변화

Chapter 10 전체를 보면 LLM Serving Engine의 역할 자체가 계속 확장되고 있다.

기존에는 다음이 주요 역할이었다.

```text
Model Load
+
Inference
+
Batching
+
KV Cache
```

하지만 현재는 여기에 더 많은 기능이 추가되고 있다.

```text
Request Routing
Model Selection
Semantic Cache
Tool Filtering
Scheduling
Multimodal Processing
Observability
Policy Enforcement
Failover
```

즉 LLM Serving은 더 이상 **하나의 모델을 GPU에서 빠르게 실행하는 Runtime**만을 의미하지 않는다.

전체 요청 흐름을 관리하고 적절한 모델, Tool, Hardware, 실행 경로를 선택하는 **Inference Infrastructure**에 가까워지고 있다.

---

# 정리

Chapter 10에서 인프라 관점으로 중요하게 볼 수 있는 내용은 다음 세 가지로 정리할 수 있다.

## Semantic Routing

요청의 의미를 분석하여 적절한 모델이나 처리 경로를 선택한다.

기존 Load Balancing이 Replica 선택 중심이었다면, Semantic Routing은 Model Endpoint나 Tool 선택까지 확장된다.

## Performance Profiling

LLM Serving 병목은 GPU 하나만 보고 판단하면 안 된다.

```text
Serving
→ Framework
→ Runtime
```

순서로 내려가면서 CPU, Operator, CUDA Kernel까지 단계적으로 분석해야 한다.

## Multimodal Serving

이미지 같은 입력이 추가되면 CPU Preprocessing이 새로운 병목이 될 수 있다.

따라서 CPU 작업과 GPU 실행을 분리하고 비동기적으로 처리하는 구조가 중요해진다.

결국 앞으로의 LLM Serving은 단순한 Model Inference를 넘어 **Routing, Scheduling, Profiling, Resource Coordination을 포함하는 전체 시스템 최적화 문제**로 보는 것이 더 적절하다.
