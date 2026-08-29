---
layout: post
title: "[LLM Serving] Chapter 8: vLLM은 GPU에 요청을 어떻게 스케줄링할까?"
date: 2026-08-30 01:20:00 +0900
categories: [LLM, LLMOps, vLLM, GPU]
tags: [LLM Serving, vLLM, Scheduler, GPU, KV Cache, Continuous Batching, Token Scheduling, LLMEngine, GPUWorker]
published: true
---

지난주 과제를 하면서, 에서는 Kubernetes GPU 환경에 vLLM을 배포하고 `max_num_seqs` 값을 바꾸면서
Throughput, TTFT, TPOT, ITL이 어떻게 달라지는지 확인했다.

당시에는 `max_num_seqs`가 vLLM이 동시에 처리할 수 있는 Sequence 수를 조절하는 값이라는 점과,
Continuous Batching이 GPU 활용률을 높이는 방식이라는 부분을 중심으로 봤다.

이번 Chapter 8에서는 여기서 한 단계 더 들어가서,

> **그렇다면 vLLM 내부에서는 실제로 여러 Request와 Token을 어떻게 관리하고 GPU에 전달할까?**

를 중심으로 정리했다.

Chapter 8에는 vLLM 외에도 TensorRT-LLM, SGLang, llama.cpp가 나오지만,
이번에는 전부 비교하기보다는 **vLLM의 내부 구조와 Scheduler**에 집중해서 봤다.

특히 인프라 관점에서는 vLLM이 단순히 모델을 실행하는 라이브러리가 아니라
**GPU Memory, KV Cache, Token Budget, 여러 사용자의 Request를 관리하는 LLM 전용 Runtime**이라는 점이 중요하게 느껴졌다.

---

## 1. 왜 LLM에는 전용 Serving Framework가 필요할까

일반적인 ML Model Serving은 비교적 단순하게 생각할 수 있다.

```text
Request
   ↓
Model Forward
   ↓
Prediction
   ↓
Response
```

이미지 분류처럼 입력 하나를 받아 한 번의 Forward Pass를 수행하고 결과를 반환하는 형태라면
정적인 Batch 처리만으로도 어느 정도 효율적으로 운영할 수 있다.

하지만 LLM은 동작 방식이 다르다.

```text
Request
   ↓
Tokenization
   ↓
Prefill
   ↓
KV Cache 생성
   ↓
Decode
   ↓
Decode
   ↓
Decode
   ↓
...
   ↓
Streaming Response
```

LLM은 답변 전체를 한 번에 생성하지 않고 **Token을 하나씩 Autoregressive하게 생성**한다.

또 Request마다 다음 조건도 모두 다르다.

- Prompt 길이
- 생성할 Output Token 수
- 요청 도착 시점
- KV Cache 사용량
- Prefill / Decode 진행 상태

예를 들어 세 Request가 동시에 들어와도 실제 실행 시간은 모두 다를 수 있다.

```text
Request A → Prompt 100 Tokens  → Output 20 Tokens
Request B → Prompt 1000 Tokens → Output 300 Tokens
Request C → Prompt 20 Tokens   → Output 50 Tokens
```

이런 요청을 일반적인 Static Batch처럼 묶어 처리하면 먼저 끝난 요청이 있어도
다른 요청 때문에 GPU 자원이 비효율적으로 사용될 수 있다.

그래서 vLLM 같은 LLM 전용 Serving Framework는 단순 Model Execution 외에도 다음을 함께 처리한다.

```text
Request Queue
Token Scheduling
Continuous Batching
KV Cache Management
Streaming
Distributed Execution
GPU Resource Utilization
```

즉 vLLM은 단순히 `model.generate()`를 편하게 호출하는 Wrapper라기보다
**LLM 추론을 실제 서비스 형태로 운영하기 위한 Serving Runtime**에 가깝다.

---

## 2. vLLM의 전체 구조

Chapter 8에서 가장 먼저 잡아야 할 구조는 다음 흐름이다.

```text
Client Request
      ↓
  LLMEngine
      ↓
  EngineCore
      ↓
  Scheduler
      ↓
SchedulerOutput
      ↓
ModelExecutor
      ↓
  GPUWorker
      ↓
GPUModelRunner
      ↓
     GPU
```

처음 보면 이름이 많아서 복잡하지만 역할을 크게 두 영역으로 나누면 이해하기 쉽다.

```text
Scheduling / Control 영역
-------------------------
LLMEngine
EngineCore
Scheduler

Execution 영역
-------------------------
ModelExecutor
GPUWorker
GPUModelRunner
GPU
```

앞쪽은 **무엇을 실행할지 결정**하고,
뒤쪽은 **결정된 작업을 실제 GPU에서 실행**한다.

---

## 3. LLMEngine — vLLM의 상위 진입점

`LLMEngine`은 vLLM 추론 시스템의 상위 Interface다.

사용자가 vLLM을 Python Library 형태로 사용하거나 API Server를 통해 요청을 보내면,
결국 이 계층에서 Request Lifecycle이 관리된다.

간단히 표현하면 다음과 같다.

```text
Client
  ↓
LLMEngine
  ↓
vLLM 내부 실행 흐름
```

LLMEngine은 다음과 같은 상위 수준의 작업을 담당한다.

- Request 수신
- Request 상태 관리
- 내부 실행 Loop 연결
- 결과 처리
- Streaming Response 연결

다만 LLMEngine이 직접 GPU에서 Transformer 연산을 수행하는 것은 아니다.

실제 내부 실행 조율은 `EngineCore`로 내려간다.

---

## 4. EngineCore — 내부 실행 흐름을 연결하는 중심

`EngineCore`는 vLLM 내부의 주요 Component를 연결하는 중앙 실행 Loop에 가깝다.

개념적으로는 다음 역할을 한다.

```text
Scheduler에게
"이번에는 무엇을 실행할지 결정해줘"
        ↓
SchedulerOutput 생성
        ↓
ModelExecutor에게 전달
        ↓
GPU 실행
        ↓
실행 결과 회수
        ↓
다음 Scheduling Iteration
```

즉 LLM Request가 한 번 GPU에서 실행되고 끝나는 것이 아니라,
Decode 과정 동안 이 Cycle이 반복된다.

```text
Scheduling
   ↓
GPU Forward
   ↓
Token 생성
   ↓
Scheduling
   ↓
GPU Forward
   ↓
Token 생성
   ↓
...
```

이 구조가 일반적인 Request/Response 기반 Application과 LLM Serving의 큰 차이 중 하나다.

---

# 5. Scheduler — 이번 Chapter에서 가장 중요하게 본 부분

vLLM의 `Scheduler`는 여러 Request가 경쟁하는 상황에서

> **이번 Iteration에 어떤 Request의 몇 개 Token을 GPU에서 계산할지 결정하는 역할**

을 한다.

처음에는 Scheduler라고 해서 단순히 Request 순서를 정하는 Queue 정도로 생각했는데,
실제로는 훨씬 많은 자원을 같이 고려한다.

```text
현재 실행 중인 Request
대기 중인 Request
Token Budget
KV Cache Block
GPU Memory
Sequence 수
Prefill / Decode 상태
Priority
```

즉 Scheduler는 LLM Serving 내부에서 일종의 **Resource Manager** 역할을 한다.

---

## 6. RUNNING Queue와 WAITING Queue

여러 Request가 들어와도 GPU 자원은 한정되어 있기 때문에
모든 Request를 무조건 동시에 실행할 수는 없다.

vLLM Scheduler는 Request를 크게 `RUNNING`과 `WAITING` 상태로 관리한다.

```text
RUNNING
├── Request A
├── Request B
└── Request C

WAITING
├── Request D
├── Request E
└── Request F
```

### RUNNING

이미 실행 중이며 GPU Resource와 KV Cache를 사용하고 있는 Request다.

### WAITING

아직 실행 Resource를 할당받지 못하고 대기 중인 Request다.

Scheduling Cycle에서는 우선 현재 RUNNING Request를 살펴보고,
남아 있는 Token Budget과 KV Cache Resource 등을 확인한 뒤
추가로 실행 가능한 WAITING Request를 Batch에 포함시킨다.

개념적으로는 다음과 같다.

```text
1. 현재 Resource 확인
      ↓
2. RUNNING Request 처리
      ↓
3. 남은 Token / KV Cache 확인
      ↓
4. WAITING Request 추가
      ↓
5. GPU에 전달할 실행 계획 생성
```

---

# 7. vLLM은 Request가 아니라 Token 단위로 Scheduling한다

이번 Chapter에서 가장 중요하게 느껴진 부분이다.

처음에는 Scheduler가 다음처럼 동작한다고 생각하기 쉽다.

```text
Request A 실행
   ↓
Request A 완료
   ↓
Request B 실행
```

하지만 vLLM은 이렇게 Request 전체를 하나의 Scheduling Unit으로만 보지 않는다.

**Token 단위로 이번 Forward Pass에서 얼마나 계산할지를 결정한다.**

예를 들어 다음 세 Request가 있다고 하자.

```text
Request A
- Prompt: 100 Tokens
- 현재 80 Tokens까지 계산

Request B
- Prefill 완료
- Decode 진행 중

Request C
- Prompt: 1000 Tokens
- 현재 200 Tokens까지 계산
```

Scheduler는 개념적으로 이번 Iteration에 다음처럼 할당할 수 있다.

```text
Request A → 20 Tokens
Request B → 1 Decode Token
Request C → 일부 Prefill Tokens
```

위 숫자는 설명을 위한 예시지만 핵심은 같다.

> **Request 전체를 한 번에 처리하는 것이 아니라 현재 시스템 Resource 안에서 각 Request가 이번 Step에 처리할 Token 수를 결정한다.**

이 방식 덕분에 긴 Prefill Request와 짧은 Decode Request가 같이 존재하는 실제 Online Serving 환경에서도
GPU를 더 유연하게 활용할 수 있다.

---

# 8. `num_computed_tokens`와 `num_tokens_with_spec`

Chapter 8의 Scheduler Code를 보면 다음과 비슷한 계산이 나온다.

```python
num_new_tokens = (
    request.num_tokens_with_spec
    + request.num_output_placeholders
    - request.num_computed_tokens
)
```

처음 봤을 때는 변수 이름 때문에 어려웠는데 의미를 단순화하면 이해하기 쉽다.

### `num_computed_tokens`

현재까지 이미 계산이 완료된 Token 수다.

```text
전체 처리 대상
████████████████████

이미 계산
██████████
```

### `num_tokens_with_spec`

현재 Request에서 처리 대상으로 보고 있는 전체 Token 수다.

Prompt와 생성 과정의 Token뿐 아니라 Speculative Token과 관련된 상태도 포함될 수 있다.

결국 Scheduler는 큰 틀에서 다음 차이를 보고 있다.

```text
처리해야 할 Token
      -
이미 처리한 Token
      =
아직 계산해야 할 Token
```

그리고 이 차이를 한 Iteration에 모두 처리할 수도 있고,
Resource 상황에 따라 일부만 처리할 수도 있다.

---

# 9. Token Budget은 왜 필요한가

Token 단위 Scheduling을 한다고 해서 원하는 만큼 GPU에 넣을 수 있는 것은 아니다.

한 Iteration에서 GPU가 처리할 수 있는 양에는 한계가 있다.

그래서 Scheduler는 **Token Budget**을 사용한다.

개념적으로 다음과 같다.

```text
이번 Iteration Token Budget = 100

Request A → 20
Request B → 1
Request C → 79

----------------
Total = 100
```

실제 Scheduling은 이것보다 훨씬 복잡하지만,
인프라 관점에서는 Token Budget을 다음처럼 볼 수 있다.

> **한 번의 GPU 실행에 얼마만큼의 LLM 작업을 넣을 것인지 제한하는 Capacity 개념**

지난 실습에서 사용했던 `max_num_seqs`가 Sequence 개수의 상한과 관련됐다면,
Token Budget은 **실제 처리할 Token 양**과 관련된 제약이다.

따라서 vLLM의 Batch Size는 단순히 Request 개수 하나만으로 결정되지 않는다.

```text
Request 수
+
Token 수
+
KV Cache
+
GPU Memory
+
현재 Request 상태
```

가 같이 영향을 준다.

---

# 10. Scheduler가 KV Cache도 신경 써야 하는 이유

LLM Decode에서는 이전 Token의 Attention 결과를 다시 계산하지 않기 위해 KV Cache를 사용한다.

Request가 여러 개 실행되면 각 Request마다 KV Cache가 필요하다.

```text
Request A
   └── KV Cache A

Request B
   └── KV Cache B

Request C
   └── KV Cache C
```

문제는 GPU Memory가 무한하지 않다는 점이다.

Scheduler가 Request를 무작정 많이 RUNNING 상태로 만들면 다음과 같은 문제가 생길 수 있다.

```text
Active Request 증가
        ↓
KV Cache 사용량 증가
        ↓
GPU Memory 압박
        ↓
추가 Request 실행 어려움
```

그래서 Scheduler는 단순히 Queue 순서만 관리하는 것이 아니라
**현재 KV Cache Block을 할당할 수 있는지까지 확인하면서 실행 대상을 결정**한다.

이 부분을 보면서 LLM Serving에서 Scheduling이 일반적인 Application Scheduling보다
Memory Management와 훨씬 강하게 연결되어 있다는 점이 이해됐다.

---

# 11. Continuous Batching은 Scheduler에서 어떻게 이어질까

지난 실습에서 Continuous Batching을 다음처럼 이해했다.

```text
초기 Batch
[A][B][C][D]

B 완료
 ↓

[A][ ][C][D]

새로운 E 투입
 ↓

[A][E][C][D]
```

Chapter 8을 보고 나니 이 기능도 결국 Scheduler의 동작과 연결됐다.

Request가 완료되거나 새 Request가 들어올 때마다 Scheduler는
현재 `RUNNING`, `WAITING`, Token Budget, KV Cache 상태를 다시 보고
다음 Iteration의 실행 Batch를 구성한다.

즉 Continuous Batching은 단순히

> "빈 Batch 자리에 새 Request를 넣는다"

정도의 기능이 아니라 실제로는

```text
Request 상태 변화
      ↓
Scheduler 재평가
      ↓
Token Budget 계산
      ↓
KV Cache 확인
      ↓
다음 GPU Batch 구성
```

과정의 반복이라고 이해할 수 있었다.

---

# 12. SchedulerOutput — 계획과 실행을 분리하는 Interface

Scheduler가 실행 대상을 결정했다고 해서 직접 GPU 연산을 수행하지는 않는다.

Scheduler는 결과를 `SchedulerOutput` 형태로 만들어 ModelExecutor에 전달한다.

```text
Scheduler
   ↓
SchedulerOutput
   ↓
ModelExecutor
```

SchedulerOutput에는 개념적으로 다음과 같은 정보가 들어간다.

```text
어떤 Request를 실행할 것인지
각 Request에서 몇 Token을 처리할 것인지
KV Cache 관련 정보
Encoder / Multimodal 관련 정보
실행에 필요한 Metadata
```

즉 SchedulerOutput은

> **Scheduler가 작성한 GPU 실행 작업 지시서**

처럼 이해하면 된다.

---

# 13. ModelExecutor — 실제 GPU Worker 실행을 조율한다

`ModelExecutor`는 SchedulerOutput을 받아 실제 Model Worker에서 실행할 수 있도록 연결한다.

특히 Multi-GPU나 Multi-Process 환경에서는 Worker가 여러 개일 수 있기 때문에
각 Worker를 조율하는 계층이 필요하다.

```text
SchedulerOutput
      ↓
ModelExecutor
   ├── Worker 0
   ├── Worker 1
   ├── Worker 2
   └── Worker 3
```

예를 들어 Tensor Parallel을 이용해 여러 GPU에서 하나의 모델을 실행한다면
ModelExecutor가 여러 Worker에 작업을 전달하고 실행을 조율한다.

---

# 14. GPUWorker와 GPUModelRunner

`GPUWorker`와 `GPUModelRunner`는 실제 GPU 실행에 가까운 계층이다.

### GPUWorker

GPU Device와 Model Lifecycle을 관리한다.

예를 들면 다음과 같은 부분이 해당한다.

- CUDA Device 설정
- Model Load
- Worker Process 상태
- Distributed Communication
- GPU Resource 관리

### GPUModelRunner

실제 Model Forward Pass를 수행한다.

```text
Input Tokens
    ↓
GPUModelRunner
    ↓
Transformer Forward
    ↓
Logits
    ↓
Next Token
```

따라서 구조를 다시 보면 역할이 분명해진다.

```text
Scheduler
"무엇을 얼마나 실행할까?"

        ↓

ModelExecutor
"어느 Worker에서 실행할까?"

        ↓

GPUWorker
"GPU와 Model 실행 환경을 관리"

        ↓

GPUModelRunner
"실제 Forward Pass 실행"
```

---

# 15. 왜 Scheduler와 GPU 실행 계층을 분리했을까

이번 Chapter에서 가장 중요한 설계 관점 중 하나다.

Scheduler는 Model 내부 Layer나 CUDA Kernel을 자세히 알 필요가 없다.

Scheduler가 관심 있는 것은 다음과 같다.

```text
Request 상태
Token 수
Queue
KV Cache
Resource Budget
Priority
```

반대로 GPUWorker나 GPUModelRunner는

```text
지금 WAITING Request가 몇 개인가?
어떤 Queue Policy를 쓰는가?
```

같은 Scheduling Policy를 자세히 알 필요가 없다.

이쪽은 실제 Model Execution에 집중하면 된다.

즉 다음처럼 **관심사를 분리**한 구조다.

```text
System-level Optimization
        ↓
Scheduler

Model Execution
        ↓
ModelExecutor / GPUWorker

Model Layer Optimization
        ↓
Attention / KV Cache / Operator

Hardware-specific Optimization
        ↓
CUDA / CustomOp
```

이 구조의 장점은 새로운 Model이나 GPU가 등장해도
모든 계층을 다시 작성할 필요가 없다는 점이다.

예를 들어 새로운 GPU Architecture가 추가되더라도
Scheduler의 Request Scheduling Policy 전체를 바꾸는 것이 아니라
하위 Hardware-specific 계층을 최적화할 수 있다.

반대로 새로운 Scheduling 기법이 추가되어도
Model Forward 구현 전체를 수정할 필요가 없다.

인프라 시스템에서도 흔히 볼 수 있는 **Control과 Execution의 분리**와 비슷한 구조라고 느꼈다.

---

# 16. Kubernetes와 비교해서 생각해보기

완전히 같은 구조는 아니지만, 인프라 관점에서는 Kubernetes와 비교하면 이해하기 쉬웠다.

Kubernetes에서는 Scheduler가 다음을 결정한다.

```text
Pod
 ↓
kube-scheduler
 ↓
어느 Node에 배치할지 결정
```

그 뒤 실제 Container 실행은 Node의 kubelet과 Container Runtime이 담당한다.

```text
kube-scheduler
= 배치 결정

kubelet / runtime
= 실제 실행
```

vLLM도 비슷한 역할 분리를 가지고 있다.

```text
vLLM Scheduler
= 어떤 Request의 몇 Token을 실행할지 결정

GPUWorker / GPUModelRunner
= 실제 GPU 연산 실행
```

물론 Kubernetes Scheduler는 Cluster의 Node Placement를 결정하고,
vLLM Scheduler는 하나의 LLM Runtime 내부 Token Execution을 관리하므로
서로 같은 Scheduler는 아니다.

하지만 **정책을 결정하는 계층과 실제 실행하는 계층을 분리한다**는 설계 관점에서는 비슷하게 볼 수 있다.

---

# 17. Kubernetes가 GPU를 할당한 뒤에도 Scheduling이 또 필요한 이유

Kubernetes에서 GPU Pod를 배포하면 보통 다음과 같이 Resource를 할당한다.

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

HAMi 같은 GPU Sharing 환경에서는 GPU Memory까지 나눠서 할당할 수도 있다.

```yaml
resources:
  limits:
    nvidia.com/gpu: "1"
    nvidia.com/gpumem: "16000"
```

여기까지만 보면 Kubernetes가 GPU를 할당했으니 Resource Scheduling이 끝난 것처럼 보인다.

하지만 실제 LLM Serving에서는 그 안에 한 단계가 더 있다.

```text
Kubernetes Scheduler
        ↓
vLLM Pod에 GPU Resource 할당
        ↓
vLLM Scheduler
        ↓
여러 User Request / Token Scheduling
        ↓
GPU Execution
```

즉 Scheduling Layer가 서로 다르다.

### Kubernetes / HAMi

```text
어떤 Pod에 어떤 GPU Resource를 줄 것인가?
```

### vLLM Scheduler

```text
할당받은 GPU 안에서 어떤 Request의 몇 Token을 실행할 것인가?
```

이 부분이 인프라 관점에서 가장 중요하게 느껴졌다.

GPU를 Pod에 잘 배치했다고 해서 그 GPU를 애플리케이션이 효율적으로 사용하는 것은 아니다.

**Cluster Resource Scheduling과 LLM Runtime Scheduling을 같이 봐야 실제 GPU 효율을 이해할 수 있다.**

---

# 18. 지난 `max_num_seqs` 실습을 다시 보면

지난 실습에서는 다음과 같이 `max_num_seqs`를 변경했다.

```text
1 → 4 → 8 → 16
```

당시에는 다음 관계를 중심으로 확인했다.

```text
max_num_seqs 증가
      ↓
동시에 Active한 Sequence 증가
      ↓
Continuous Batching 확대
      ↓
GPU 병렬 활용 증가
      ↓
Throughput 증가
```

Chapter 8을 보고 나니 `max_num_seqs`가 단독으로 GPU Batch Size를 결정하는 값이 아니라는 점이 더 명확해졌다.

실제 Scheduler는 다음 요소를 같이 본다.

```text
max_num_seqs
Token Budget
KV Cache
Request 상태
Prefill / Decode 상태
GPU Memory
```

따라서

```text
max_num_seqs = 16
```

이라고 설정했다고 해서 항상 정확히 16개의 Request가 같은 형태로 GPU에서 실행되는 것은 아니다.

이 값은 Scheduler가 동시에 Active하게 관리할 수 있는 Sequence 수의 상한 중 하나이고,
실제 실행 Batch는 현재 Runtime 상태에 따라 계속 달라진다.

지난 실습이 **설정값을 변경했을 때 성능이 어떻게 달라지는지 확인한 실험**이었다면,
이번 Chapter 8에서는 **그 설정 뒤에서 Scheduler가 실제로 어떤 판단을 하고 있는지 구조적으로 이해한 것**에 가깝다.

---

# 19. vLLM의 계층적 최적화 구조

Chapter 8에서는 vLLM의 최적화를 한 곳에 몰아넣지 않고
각 계층에서 나눠 처리하는 구조도 설명한다.

개념적으로 정리하면 다음과 같다.

| 계층 | 주요 역할 |
|---|---|
| Scheduler | Request / Token Scheduling, Batching, Cache 관련 시스템 수준 최적화 |
| ModelExecutor | Model Architecture와 실행 방식에 맞는 최적화 |
| Model Layer | Attention, KV Cache, Layer 단위 최적화 |
| CustomOp | CUDA Kernel, Quantization 등 Hardware-specific 최적화 |

위로 갈수록 전체 Serving System에 가깝고,
아래로 내려갈수록 Model과 Hardware에 가까워진다.

```text
Scheduler
   ↓
ModelExecutor
   ↓
Model Layer
   ↓
CustomOp
   ↓
GPU Hardware
```

이 구조 덕분에 Serving Framework가 새로운 Model Architecture나 새로운 GPU에 대응할 때
전체 시스템을 한 번에 바꾸지 않아도 된다.

---

# 20. LLM Serving을 인프라 관점에서 보면

이번 Chapter를 보고 나서 LLM Serving Infrastructure를 다음처럼 생각하게 됐다.

```text
Application / Client
        ↓
API / Gateway
        ↓
LLM Serving Engine
        ↓
vLLM Scheduler
        ↓
Request / Token / KV Cache 관리
        ↓
GPU Model Execution
        ↓
GPU Compute / Memory
```

그리고 Kubernetes까지 포함하면 한 단계 더 넓어진다.

```text
Kubernetes Cluster
        ↓
GPU Scheduling / HAMi
        ↓
vLLM Pod
        ↓
vLLM Scheduler
        ↓
Continuous Batching
        ↓
KV Cache / Token Budget
        ↓
GPUWorker
        ↓
GPU
```

일반적인 Web Application에서는 CPU와 Memory Resource Request/Limit을 설정하고
Pod Replica를 늘리는 것만으로도 Scale-out 전략을 생각할 수 있다.

하지만 LLM Serving에서는 같은 GPU 한 장을 할당받아도
vLLM 내부 설정과 Scheduling 방식에 따라 실제 처리량과 Latency가 크게 달라질 수 있다.

그래서 LLM Infrastructure에서는 다음 세 영역을 같이 봐야 한다고 느꼈다.

```text
Cluster Resource Management
        +
Serving Engine Scheduling
        +
GPU Compute / Memory 특성
```

---

# Chapter 8을 읽고

지난 실습까지는 vLLM의 `max_num_seqs`나 Continuous Batching을
주로 **성능을 조절하는 설정**으로 봤다.

이번 Chapter 8에서는 그 설정들이 동작하는 vLLM 내부 구조를 보면서
왜 Serving Framework가 단순 Model Wrapper가 아닌지 조금 더 이해할 수 있었다.

특히 기억에 남은 부분은 vLLM이 Request 전체를 단순히 Queue에 넣고 순서대로 실행하는 것이 아니라,
**Token 단위로 이번 Iteration에서 얼마나 계산할지 결정한다는 점**이었다.

또 Scheduler가 단순 Request Queue만 보는 것이 아니라 Token Budget과 KV Cache까지 같이 관리한다는 점에서,
LLM Serving의 Scheduling은 GPU Memory Management와 상당히 밀접하게 연결되어 있었다.

인프라 관점에서는 Kubernetes나 HAMi가 GPU를 **어떤 Pod에 할당할지** 결정하는 것과,
vLLM이 그 GPU 안에서 **어떤 Request와 Token을 실행할지** 결정하는 것이 서로 다른 Scheduling Layer라는 점도 중요했다.

결국 GPU를 Pod에 할당했다고 해서 GPU Resource Management가 끝나는 것이 아니라,
그 위의 Serving Runtime이 Request, Token, KV Cache를 얼마나 효율적으로 관리하는지까지 봐야
실제 LLM Serving 성능을 이해할 수 있다는 점이 이번 Chapter에서 가장 크게 연결됐다.

---

## 핵심 정리

| 개념 | 의미 |
|---|---|
| LLMEngine | vLLM의 상위 Interface와 Request Lifecycle 관리 |
| EngineCore | Scheduler와 Model Execution을 연결하는 내부 실행 Loop |
| Scheduler | 어떤 Request의 몇 Token을 실행할지 결정하는 Resource Manager |
| RUNNING | 현재 Resource를 할당받아 실행 중인 Request |
| WAITING | 실행 Resource를 기다리는 Request |
| Token-level Scheduling | Request 전체가 아니라 Token 단위로 실행량을 결정하는 방식 |
| Token Budget | 한 Scheduling Iteration에서 처리 가능한 Token Capacity |
| `num_computed_tokens` | 현재까지 이미 계산한 Token 수 |
| `num_tokens_with_spec` | 현재 처리 대상으로 보는 전체 Token 수 |
| SchedulerOutput | Scheduler가 ModelExecutor에 전달하는 실행 계획 |
| ModelExecutor | 여러 Worker의 실제 Model Execution을 조율 |
| GPUWorker | GPU Device와 Model Lifecycle을 관리하는 Worker |
| GPUModelRunner | 실제 Model Forward Pass를 수행하는 실행 계층 |
| KV Cache | 이전 Attention 결과를 저장해 Decode 중 중복 계산을 줄이는 Cache |
| Continuous Batching | 완료된 Request를 제거하고 새 Request를 계속 실행 Batch에 투입하는 방식 |

전체 흐름은 다음 한 줄로 정리할 수 있다.

```text
Request
  ↓
LLMEngine
  ↓
EngineCore
  ↓
Scheduler
  ↓
Request / Token / KV Cache Scheduling
  ↓
SchedulerOutput
  ↓
ModelExecutor
  ↓
GPUWorker
  ↓
GPUModelRunner
  ↓
GPU
  ↓
Token
```

지난 실습에서는 `max_num_seqs` 값을 바꿔 성능 변화를 확인했다면,
이번 Chapter에서는 그 뒤에서 vLLM Scheduler가 **제한된 GPU Resource 안에 어떤 작업을 넣을지 계속 결정하고 있다**는 내부 구조를 이해한 것이 핵심이었다.

---

## 참고

- *Hands-On LLM Serving and Optimization*, Chapter 8
- vLLM Architecture Overview: https://docs.vllm.ai/en/stable/design/arch_overview/
- vLLM Scheduler source: https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/sched/scheduler.py