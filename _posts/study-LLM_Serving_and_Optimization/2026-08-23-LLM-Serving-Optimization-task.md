---
layout: post
title: "[LLM Serving] Task: vLLM Continuous Batching 성능 비교 - max_num_seqs 1/4/8/16"
date: 2026-08-23 03:14:00 +0900
categories: [LLM, LLMOps, vLLM, Kubernetes]
tags: [LLM Serving, vLLM, Continuous Batching, HAMi, Kubernetes, GPU, L40S, TTFT, TPOT, ITL, Throughput]
published: true
---

이번 Chapter 6에서는 LLM Serving 성능을 결정하는 핵심 요소 중 하나인 **Request Batching과 Scheduling**을 다뤘다. 책을 읽을 때는 Continuous Batching이 "여러 요청을 함께 처리하는 방식"이라는 정도로 이해했지만, 실제로 어느 정도의 효과가 있는지 감이 잘 오지 않았다.

그래서 이번에는 Kubernetes GPU 환경에 vLLM을 직접 배포하고, vLLM Scheduler가 동시에 처리할 수 있는 Sequence 수를 제한하는 `max_num_seqs` 값을 `1 → 4 → 8 → 16`으로 변경하면서 성능을 비교했다. 단순히 Throughput만 확인하는 것이 아니라 **TTFT, TPOT, ITL과 같은 LLM Serving 전용 Latency 지표까지 함께 측정**해 Continuous Batching이 실제 사용자 응답성과 GPU 처리량에 어떤 영향을 주는지 확인했다.

실습 과정에서는 Kubernetes에 HAMi Scheduler를 통해 L40S GPU를 할당하고, Qwen2.5-3B-Instruct를 vLLM으로 Serving했다. 중간에 Kubernetes Service Discovery와 vLLM의 `VLLM_PORT` 환경변수가 충돌하는 문제도 발생했는데, 이 문제를 직접 확인하고 수정하면서 Kubernetes에서 LLM Serving 서비스를 배포할 때 고려해야 할 부분도 같이 살펴볼 수 있었다.

---

## 1. 이번 실습에서 확인하려는 것

이번 실험의 질문은 단순하다.

> **동시에 처리할 수 있는 Sequence 수를 늘리면 vLLM의 Serving 성능은 어떻게 달라질까?**

실험에서 변경한 값은 `max_num_seqs` 하나다.

```text
max_num_seqs = 1
max_num_seqs = 4
max_num_seqs = 8
max_num_seqs = 16
```

반면 모델, GPU, Prompt 길이, Output 길이, Client Concurrency 등 다른 조건은 모두 동일하게 유지했다.

실험 관점에서 정리하면 다음과 같다.

| 구분 | 설정 |
|---|---|
| 독립변수 | `max_num_seqs` |
| 종속변수 | Throughput, TTFT, TPOT, ITL |
| 통제변수 | GPU, Model, dtype, Input/Output Length, Client Concurrency, Temperature 등 |

즉 `max_num_seqs`만 변경했을 때 성능 지표가 어떻게 달라지는지를 비교하는 실험이다.

---

## 2. 왜 LLM Serving에서 Batching이 필요한가

LLM 추론은 크게 **Prefill**과 **Decode** 두 단계로 나눌 수 있다.

### Prefill

Prefill은 사용자가 입력한 Prompt 전체를 처리하는 단계다.

```text
Prompt
  ↓
Tokenization
  ↓
Transformer Forward
  ↓
KV Cache 생성
  ↓
첫 Token 생성 준비
```

Prompt의 여러 Token을 병렬적으로 처리할 수 있기 때문에 Prefill은 상대적으로 **Compute-intensive**한 특성을 가진다. Prompt가 길어질수록 처리해야 할 계산량도 증가한다.

### Decode

Prefill이 끝난 뒤에는 실제 답변 Token을 생성한다.

```text
Token 1
  ↓
Token 2
  ↓
Token 3
  ↓
...
```

LLM은 답변 전체를 한 번에 만드는 것이 아니라 이전에 생성한 Token을 바탕으로 다음 Token을 하나씩 생성한다. KV Cache를 사용해 이전 Attention 계산 결과를 재사용하지만, Decode iteration마다 모델 Weight와 KV Cache를 계속 사용해야 한다.

이 때문에 Decode는 단일 요청만 처리할 경우 GPU의 병렬성을 충분히 활용하지 못할 수 있다.

예를 들어 요청 A 하나만 처리하면 GPU는 매 Decode iteration마다 A의 다음 Token 하나를 계산한다.

```text
GPU
 ↓
Request A
 ↓
Token 1

GPU
 ↓
Request A
 ↓
Token 2
```

하지만 요청 A, B, C, D를 Batch로 처리하면 하나의 iteration에서 여러 Sequence의 Token을 함께 계산할 수 있다.

```text
       GPU

A ─┐
B ─┼─ Batch → Next Token 계산
C ─┤
D ─┘
```

GPU는 병렬 계산에 강하기 때문에 여러 요청을 함께 처리할수록 모델 Weight를 읽은 뒤 더 많은 연산을 수행할 수 있고 전체 Throughput을 높일 수 있다.

---

## 3. Static Batching과 Continuous Batching

단순한 Static Batching에서는 여러 요청을 하나의 Batch로 묶은 뒤 Batch 전체가 끝날 때까지 같이 처리한다.

문제는 각 요청의 생성 길이가 다르다는 점이다.

```text
Request A → 20 Tokens
Request B → 100 Tokens
Request C → 500 Tokens
Request D → 50 Tokens
```

A가 먼저 끝나도 다른 요청 때문에 Batch가 유지된다면 GPU 자원을 효율적으로 활용하기 어렵다.

Continuous Batching은 이런 문제를 줄이기 위해 **완료된 Sequence를 Batch에서 제거하고, 대기 중인 새 요청을 그 자리에 즉시 투입하는 방식**이다.

```text
초기 Batch

[A][B][C][D]

A 완료
 ↓

[ ][B][C][D]

대기 중인 E 투입
 ↓

[E][B][C][D]
```

따라서 요청 도착 시점과 생성 길이가 제각각인 실제 Online Serving 환경에서 GPU를 지속적으로 활용할 수 있다.

---

## 4. `max_num_seqs`는 무엇인가

이번 실험에서 가장 중요한 설정은 vLLM의 `max_num_seqs`다.

`max_num_seqs`는 **vLLM Scheduler가 동시에 Active 상태로 관리할 수 있는 Sequence 수의 상한**이다.

예를 들어 다음과 같이 설정했다고 하자.

```text
max_num_seqs = 1
```

Client에서 여러 요청이 동시에 들어와도 Scheduler가 동시에 처리할 수 있는 Sequence 수는 최대 1개다.

```text
Request A ─┐
Request B ─┤
Request C ─┤
Request D ─┘
           ↓
         Queue
           ↓
        [ A ]
```

반면 다음과 같이 설정하면,

```text
max_num_seqs = 4
```

Scheduler가 최대 4개의 Sequence를 동시에 Active 상태로 처리할 수 있다.

```text
[A][B][C][D]
```

다만 `max_num_seqs=4`가 항상 실제 GPU Batch Size가 정확히 4라는 뜻은 아니다. 실제 Scheduling은 Prefill/Decode 상태, Token Budget, KV Cache 용량 등 여러 조건에 영향을 받는다.

따라서 정확하게는 다음과 같이 이해하는 것이 좋다.

```text
max_num_seqs
= Scheduler가 허용하는 동시 Active Sequence 수의 상한
```

---

## 5. `max_num_seqs`와 Client Concurrency의 차이

Benchmark에서는 다음 옵션도 사용했다.

```bash
--max-concurrency 16
```

이 값은 `max_num_seqs`와 역할이 다르다.

`max-concurrency`는 **Benchmark Client가 서버에 동시에 전송할 수 있는 Outstanding Request의 최대 개수**이고, `max_num_seqs`는 **Server 내부의 vLLM Scheduler가 동시에 처리할 수 있는 Sequence 수의 상한**이다.

```text
Client
max-concurrency=16
        │
        │ 최대 16개 동시 요청
        ▼
vLLM Server
max_num_seqs=1/4/8/16
        │
        ▼
GPU
```

이번 실험에서는 Client Concurrency를 16으로 고정하고 Server의 `max_num_seqs`만 변경했다.

---

## 6. LLM Serving에서 비교해야 할 성능 지표

일반적인 API Benchmark에서는 전체 Response Time과 RPS 정도만 보는 경우가 많지만, LLM은 Token을 Streaming 방식으로 생성하기 때문에 Latency를 조금 더 세분화해서 봐야 한다.

### Request Throughput

단위는 `req/s`이며 **서버가 1초 동안 완료할 수 있는 Request 수**를 의미한다.

```text
0.8 req/s
→ 초당 평균 0.8개의 요청 완료

9 req/s
→ 초당 약 9개의 요청 완료
```

값이 높을수록 동일한 시간에 더 많은 사용자 요청을 처리할 수 있다.

### Output Token Throughput

단위는 `tok/s`이며 **서버가 전체 요청에 대해 초당 생성한 Output Token 수**를 의미한다.

LLM은 요청마다 출력 길이가 달라질 수 있기 때문에 단순 Request Throughput뿐 아니라 Token Throughput도 중요한 지표다.

### TTFT(Time To First Token)

TTFT는 **사용자가 요청을 보낸 시점부터 첫 번째 Token을 받기까지 걸리는 시간**이다.

```text
Request
   │
   │ ← TTFT
   ▼
First Token
```

Chatbot처럼 Streaming 응답을 사용하는 서비스에서는 사용자가 가장 직접적으로 체감하는 지표 중 하나다. 전체 답변이 완료되기 전에 첫 Token이 빠르게 나오면 서비스가 훨씬 빠르게 느껴진다.

TTFT에는 단순 모델 계산 시간뿐만 아니라 **Request Queue에서 기다린 시간과 Prefill 처리 시간**도 포함될 수 있다.

### TPOT(Time Per Output Token)

TPOT은 첫 번째 Token 이후 **Output Token 하나를 생성하는 데 평균적으로 걸리는 시간**이다.

```text
Token 1
   │ 10ms
   ▼
Token 2
   │ 10ms
   ▼
Token 3
```

Decode 단계의 속도를 확인할 때 유용하다.

### ITL(Inter-Token Latency)

ITL은 **Streaming 중 연속된 두 Token 사이의 시간 간격**이다.

TTFT가 "답변이 언제 시작되는가"를 나타낸다면, ITL은 "답변이 시작된 뒤 얼마나 부드럽게 계속 생성되는가"를 나타낸다.

### Mean, Median, P99

Latency 지표를 볼 때는 평균만 보면 안 된다.

- **Mean**: 전체 요청의 평균
- **Median**: 정렬했을 때 중앙에 위치한 값
- **P99**: 약 99%의 요청이 이 값 이하에서 처리된다는 의미

특히 Production 환경에서는 평균 성능이 좋아도 일부 사용자가 매우 느린 응답을 경험할 수 있으므로 P95, P99와 같은 **Tail Latency**를 같이 확인하는 것이 중요하다.

---

# 실습

## 7. 실습 환경

이번 실습은 `aila-dev`에서 원격 Kubernetes Cluster를 관리하는 구조로 진행했다.

`aila-dev` 자체가 Kubernetes Node인 것은 아니다. `aila-dev`에는 `kubectl`과 GPU Cluster에 접근하기 위한 `gpu-kubeconfig.yaml`만 있고, 실제 Pod는 원격 Kubernetes Cluster의 Worker Node에서 실행된다.

```text
aila-dev
  │
  │ kubectl + gpu-kubeconfig.yaml
  ▼
Kubernetes API Server
  │
  ├─ 일반 Worker
  │
  └─ GPU Worker
       └─ NVIDIA L40S
```

실험에 사용한 주요 환경은 다음과 같다.

| 항목 | 값 |
|---|---|
| GPU | NVIDIA L40S |
| GPU Memory | 약 46GB Physical VRAM |
| GPU 공유 | HAMi Scheduler |
| vLLM GPU Memory 할당 | 16,000 MiB |
| Model | `Qwen/Qwen2.5-3B-Instruct` |
| vLLM | 0.27.1 |
| dtype | BF16 |
| max_model_len | 4096 |
| gpu_memory_utilization | 0.90 |
| Attention Backend | FlashAttention 2 |
| Prefix Caching | Enabled |
| Chunked Prefill | Enabled |
| Client Concurrency | 16 |

---

## 8. Kubernetes Cluster 접속 확인

먼저 `aila-dev`에 kubeconfig가 있는지 확인했다.

```bash
ls -lh ~/gpu-kubeconfig.yaml
```

kubectl Client도 정상적으로 설치되어 있었다.

```bash
kubectl version --client
```

결과:

```text
Client Version: v1.36.3
Kustomize Version: v5.8.1
```

kubeconfig의 Context를 확인했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml config get-contexts
```

```text
CURRENT   NAME   CLUSTER   AUTHINFO    NAMESPACE
*         gpu    gpu       gpu-admin
```

Cluster 연결 확인:

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml cluster-info
```

```text
Kubernetes control plane is running at https://gpu.qks.quantumcns.ai:16443
CoreDNS is running at https://gpu.qks.quantumcns.ai:16443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

---

## 9. GPU Node와 HAMi 상태 확인

Node별 GPU Resource를 확인했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml get nodes \
  -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\\.com/gpu,GPUMEM:.status.allocatable.nvidia\\.com/gpumem,GPUCORES:.status.allocatable.nvidia\\.com/gpucores
```

GPU Worker에는 NVIDIA GPU Resource가 노출되어 있었다.

HAMi Pod도 확인했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml get pods -n hami-system
```

HAMi Scheduler와 Device Plugin이 정상 실행 중이었다.

HAMi ConfigMap에서 Device Split 설정도 확인했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml \
  -n hami-system get configmap hami-device-plugin -o yaml \
  | grep -i -E -C 8 'devicesplitcount|devicememoryscaling|nodename|name'
```

확인한 설정:

```json
{
  "nodeconfig": [
    {
      "name": "your-node-name",
      "operatingmode": "hami-core",
      "devicememoryscaling": 1,
      "devicesplitcount": 10,
      "preconfigureddevicememory": 0,
      "migstrategy": "none"
    }
  ]
}
```

ConfigMap에는 `devicesplitcount: 10`이 있었지만 실제 Node의 `nvidia.com/gpu` allocatable은 15로 확인됐다. 이번 실습의 목적은 GPU Slot 수 자체를 검증하는 것이 아니었기 때문에 이 차이에 대한 추가 분석은 진행하지 않았다.

---

## 10. HAMi GPU Memory 격리 테스트

vLLM을 바로 배포하기 전에 HAMi를 통해 GPU Memory 제한이 실제로 적용되는지 간단한 Pod로 확인했다.

`hami-test.yaml`을 작성했다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hami-test
spec:
  schedulerName: hami-scheduler

  containers:
    - name: cuda
      image: nvcr.io/nvidia/gpu-operator:v25.10.1

      command:
        - sh
        - -c
        - |
          echo "=== NVIDIA SMI ==="
          nvidia-smi
          echo "=== KEEP ALIVE ==="
          while true; do sleep 3600; done

      resources:
        requests:
          cpu: 100m
          memory: 128Mi

        limits:
          cpu: "500m"
          memory: 512Mi
          nvidia.com/gpu: "1"
          nvidia.com/gpumem: "4000"
```

문법 확인:

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml \
  apply --dry-run=client \
  -f ~/hami-test.yaml
```

```text
pod/hami-test created (dry run)
```

실제 배포:

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml apply -f ~/hami-test.yaml
```

Pod 상태 확인:

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml get pod hami-test -o wide
```

```text
NAME        READY   STATUS    RESTARTS   AGE   IP            NODE
hami-test   1/1     Running   0          6s    10.100.1.65   gpu-gpu-pool-hrv5t-tfhps
```

Pod 내부에서 `nvidia-smi`를 실행했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml exec -it hami-test -- nvidia-smi
```

```text
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.105.08             Driver Version: 580.105.08     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
|   0  NVIDIA L40S                    Off |   00000000:09:00.0 Off |                    0 |
| N/A   31C    P8             34W /  350W |       0MiB /   4000MiB |      0%      Default |
+-----------------------------------------+------------------------+----------------------+
```

물리 L40S의 VRAM은 약 46GB이지만 Pod 내부에서는 `4000MiB`만 보였다. 즉 다음 HAMi Resource 설정이 실제 GPU Memory 제한으로 적용된 것을 확인할 수 있었다.

```yaml
nvidia.com/gpu: "1"
nvidia.com/gpumem: "4000"
```

Pod Annotation도 확인했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml describe pod hami-test
```

```text
hami.io/bind-phase: success
hami.io/vgpu-devices-allocated:
GPU-740968d1-8f42-f6c5-14ca-e8e83f6cd2e6,NVIDIA,4000,0:;
```

테스트가 끝난 뒤 Pod는 삭제했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml delete pod hami-test
```

---

## 11. Hugging Face 네트워크 연결 확인

Qwen 모델은 최초 실행 시 Hugging Face에서 다운로드해야 하므로 Cluster 내부에서 외부 연결이 가능한지 확인했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml run net-test \
  --image=curlimages/curl:latest \
  --restart=Never \
  --command -- \
  curl -I https://huggingface.co
```

Pod 확인:

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml get pod net-test
```

```text
NAME       READY   STATUS      RESTARTS
net-test   0/1     Completed   0
```

로그를 확인했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml logs net-test
```

```text
HTTP/2 200
content-type: text/html; charset=utf-8
```

`HTTP/2 200` 응답을 통해 Kubernetes Pod에서 Hugging Face까지 정상적으로 접근할 수 있음을 확인했다.

테스트 Pod 삭제:

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml delete pod net-test
```

---

## 12. vLLM Deployment 구성

Serving 모델은 `Qwen/Qwen2.5-3B-Instruct`를 사용했고 HAMi에서 16GB GPU Memory를 할당했다.

최종 `vllm.yaml`은 다음과 같이 구성했다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm

spec:
  replicas: 1

  selector:
    matchLabels:
      app: vllm

  template:
    metadata:
      labels:
        app: vllm

    spec:
      schedulerName: hami-scheduler

      # Kubernetes Service 관련 환경변수 자동 주입 방지
      enableServiceLinks: false

      containers:
        - name: vllm
          image: vllm/vllm-openai:latest

          args:
            - "Qwen/Qwen2.5-3B-Instruct"
            - "--host"
            - "0.0.0.0"
            - "--port"
            - "8000"
            - "--dtype"
            - "auto"
            - "--max-model-len"
            - "4096"
            - "--max-num-seqs"
            - "1"
            - "--gpu-memory-utilization"
            - "0.90"

          ports:
            - containerPort: 8000

          resources:
            requests:
              cpu: "2"
              memory: "8Gi"

            limits:
              cpu: "8"
              memory: "16Gi"
              nvidia.com/gpu: "1"
              nvidia.com/gpumem: "16000"

---
apiVersion: v1
kind: Service
metadata:
  name: vllm-service

spec:
  selector:
    app: vllm

  ports:
    - name: http
      port: 8000
      targetPort: 8000

  type: ClusterIP
```

문법 확인 후 배포했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml \
  apply --dry-run=client -f ~/vllm.yaml
```

```text
deployment.apps/vllm created (dry run)
service/vllm-service created (dry run)
```

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml apply -f ~/vllm.yaml
```

---

## 13. 트러블슈팅: `VLLM_PORT`와 Kubernetes Service 이름 충돌

처음에는 Service 이름을 다음처럼 `vllm`으로 만들었다.

```yaml
kind: Service
metadata:
  name: vllm
```

배포 후 Pod가 반복적으로 `CrashLoopBackOff` 상태가 됐다.

```text
vllm-774cf6d4db-lltxr   0/1   CrashLoopBackOff
```

로그를 확인했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml \
  logs vllm-774cf6d4db-lltxr --tail=100
```

핵심 오류는 다음과 같았다.

```text
ValueError: VLLM_PORT 'tcp://10.96.13.234:8000' appears to be a URI.
This may be caused by a Kubernetes service discovery issue
```

원인은 Kubernetes의 Service 환경변수 자동 주입이었다.

Service 이름을 `vllm`으로 생성하면 Kubernetes가 해당 Service 정보를 Pod에 환경변수 형태로 주입할 수 있다.

```text
VLLM_SERVICE_HOST=10.96.13.234
VLLM_SERVICE_PORT=8000
VLLM_PORT=tcp://10.96.13.234:8000
```

그런데 vLLM도 내부 프로세스 통신을 위해 `VLLM_PORT`라는 환경변수를 사용한다. vLLM은 여기에 정수 형태의 Port 값이 들어올 것을 기대하지만 Kubernetes가 `tcp://...` URI를 넣으면서 EngineCore 초기화가 실패했다.

이를 해결하기 위해 두 가지를 수정했다.

첫 번째는 Service 이름을 `vllm`에서 `vllm-service`로 변경한 것이다.

```yaml
metadata:
  name: vllm-service
```

두 번째는 Pod Spec에 다음 옵션을 추가해 Kubernetes의 Service Link 환경변수 자동 주입을 막았다.

```yaml
enableServiceLinks: false
```

기존 Service를 삭제했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml delete service vllm
```

수정한 YAML을 다시 적용했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml apply -f ~/vllm.yaml
```

이후 `VLLM_PORT` 환경변수가 더 이상 주입되지 않는 것을 확인했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml \
  exec deployment/vllm -- env | grep '^VLLM_PORT'
```

출력 없음.

새로운 vLLM Pod는 정상적으로 Running 상태가 됐다.

```text
vllm-65cdcc9dd6-xf9jw   1/1   Running   0
```

---

## 14. vLLM 모델 Loading 확인

vLLM 로그를 확인했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml \
  logs deployment/vllm --tail=100
```

주요 설정은 다음과 같이 확인됐다.

```text
version 0.27.1
model Qwen/Qwen2.5-3B-Instruct

dtype=torch.bfloat16
max_seq_len=4096
enable_prefix_caching=True
enable_chunked_prefill=True
```

Attention Backend는 FlashAttention 2가 선택됐다.

```text
Using FLASH_ATTN attention backend
Using FlashAttention version 2
```

Model Weight는 약 5.79GiB가 사용됐다.

```text
Model loading took 5.79 GiB
```

KV Cache 관련 정보도 확인했다.

```text
Available KV cache memory: 7.01 GiB
GPU KV cache size: 204,048 tokens
Maximum concurrency for 4,096 tokens per request: 49.82x
```

최종적으로 API Server가 정상적으로 시작됐다.

```text
Starting vLLM server on http://0.0.0.0:8000
Application startup complete.
API server: HTTP server started
```

---

## 15. HAMi 16GB GPU Memory 적용 확인

vLLM Pod 내부에서 GPU 상태를 확인했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml \
  exec -it deployment/vllm -- nvidia-smi
```

```text
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.105.08             Driver Version: 580.105.08     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
|   0  NVIDIA L40S                    On  |   00000000:09:00.0 Off |                    0 |
| N/A   39C    P0            105W /  350W |    6468MiB /  16000MiB |      0%      Default |
+-----------------------------------------+------------------------+----------------------+

|    0   N/A  N/A             141      C   VLLM::EngineCore       6556MiB |
```

HAMi에서 요청한 `16000MiB`가 실제 Pod 내부 GPU Memory 한도로 적용되어 있었다.

---

## 16. vLLM API 동작 확인

Port Forward를 실행했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml \
  port-forward service/vllm-service 8000:8000
```

다른 Terminal에서 Model API를 확인했다.

```bash
curl http://127.0.0.1:8000/v1/models
```

```json
{
  "object": "list",
  "data": [
    {
      "id": "Qwen/Qwen2.5-3B-Instruct",
      "object": "model",
      "owned_by": "vllm",
      "max_model_len": 4096
    }
  ]
}
```

실제 Chat Completion도 호출했다.

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-3B-Instruct",
    "messages": [
      {
        "role": "user",
        "content": "Explain Kubernetes in three sentences."
      }
    ],
    "max_tokens": 100
  }'
```

응답:

```json
{
  "model": "Qwen/Qwen2.5-3B-Instruct",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "Kubernetes is an open-source system for automating deployment, scaling, and management of containerized applications..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 36,
    "completion_tokens": 65,
    "total_tokens": 101
  }
}
```

여기까지 확인하면서 vLLM Serving 환경 구성이 완료됐다.

---

## 17. Benchmark Client Pod 구성

`aila-dev`에는 vLLM CLI를 별도로 설치하지 않았다.

```bash
vllm --version
```

```text
vllm: command not found
```

Benchmark를 위해 `aila-dev`에 vLLM 전체 패키지를 다시 설치하는 대신 Kubernetes Cluster 내부에 Benchmark 전용 Pod를 하나 구성했다.

`vllm-bench.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vllm-bench

spec:
  restartPolicy: Never

  containers:
    - name: bench
      image: vllm/vllm-openai:latest
      imagePullPolicy: IfNotPresent

      command:
        - /bin/bash
        - -lc
        - |
          sleep infinity

      resources:
        requests:
          cpu: "1"
          memory: "2Gi"

        limits:
          cpu: "4"
          memory: "8Gi"
```

Benchmark Pod는 GPU를 요청하지 않는다. 역할은 HTTP Client로서 vLLM Server에 부하를 발생시키는 것이다.

```text
vllm-bench Pod
      │
      │ HTTP Requests
      ▼
vllm-service:8000
      │
      ▼
vLLM Server Pod
      │
      ▼
HAMi
      │
      ▼
NVIDIA L40S
```

배포했다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml \
  apply -f ~/vllm-bench.yaml
```

상태 확인:

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml \
  get pod vllm-bench -o wide
```

```text
NAME         READY   STATUS    RESTARTS   IP             NODE
vllm-bench   1/1     Running   0          10.100.1.123   gpu-gpu-pool-hrv5t-tfhps
```

vLLM Version 확인:

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml \
  exec vllm-bench -- vllm --version
```

```text
0.27.1
```

---

## 18. Benchmark 조건

이번 실험에서는 `max_num_seqs` 외에는 모든 Benchmark 조건을 동일하게 유지했다.

```text
Model                 Qwen/Qwen2.5-3B-Instruct
Requests              100
Random Input Length   512 Tokens
Random Output Length  128 Tokens
Client Concurrency    16
Temperature           0
```

Benchmark 명령은 다음과 같다.

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml \
  exec vllm-bench -- \
  vllm bench serve \
  --backend openai-chat \
  --base-url http://vllm-service:8000 \
  --endpoint /v1/chat/completions \
  --model Qwen/Qwen2.5-3B-Instruct \
  --dataset-name random \
  --num-prompts 100 \
  --random-input-len 512 \
  --random-output-len 128 \
  --max-concurrency 16 \
  --temperature 0
```

`--temperature 0`을 지정해 Sampling에 의한 변동을 줄였고, Random Dataset의 Input/Output Length를 동일하게 유지해 `max_num_seqs` 변화에 따른 성능 차이를 비교하기 쉽게 만들었다.

---

# Benchmark 결과

## 19. `max_num_seqs=1`

먼저 Continuous Batching 효과를 거의 제한한 Baseline으로 `max_num_seqs=1`을 사용했다.

```yaml
- "--max-num-seqs"
- "1"
```

결과:

```text
============ Serving Benchmark Result ============
Successful requests:                     100
Failed requests:                         0
Maximum request concurrency:             16
Benchmark duration (s):                  124.43
Total input tokens:                      54100
Total generated tokens:                  12800
Request throughput (req/s):              0.80
Output token throughput (tok/s):         102.87
Peak output token throughput (tok/s):    106.00
Total token throughput (tok/s):          537.65

---------------Time to First Token----------------
Mean TTFT (ms):                          17200.42
Median TTFT (ms):                        18660.07
P99 TTFT (ms):                           18670.47

-----Time per Output Token (excl. 1st token)------
Mean TPOT (ms):                          9.58
Median TPOT (ms):                        9.58
P99 TPOT (ms):                           9.59

---------------Inter-token Latency----------------
Mean ITL (ms):                           9.51
Median ITL (ms):                         9.58
P99 ITL (ms):                            10.15
==================================================
```

Client에서는 최대 16개의 Request를 동시에 보내고 있지만 Server는 한 번에 하나의 Sequence만 Active하게 처리할 수 있다.

```text
16 Concurrent Requests
        ↓
      Queue
        ↓
max_num_seqs = 1
        ↓
      GPU
```

이 때문에 실제 Decode를 시작한 Request의 TPOT과 ITL은 낮았지만 대부분의 Request가 Queue에서 오래 기다리면서 Mean TTFT가 약 17.2초까지 증가했다.

---

## 20. `max_num_seqs=4`

다음으로 `max_num_seqs`를 4로 변경했다.

```yaml
- "--max-num-seqs"
- "4"
```

YAML 적용:

```bash
kubectl --kubeconfig ~/gpu-kubeconfig.yaml apply -f ~/vllm.yaml
```

새 Pod가 준비된 뒤 동일한 Benchmark를 실행했다.

결과:

```text
============ Serving Benchmark Result ============
Successful requests:                     100
Failed requests:                         0
Benchmark duration (s):                  34.42

Request throughput (req/s):              2.91
Output token throughput (tok/s):         371.84
Peak output token throughput (tok/s):    392.00
Total token throughput (tok/s):          1943.45

Mean TTFT (ms):                          3869.84
Median TTFT (ms):                        4174.25
P99 TTFT (ms):                           4377.27

Mean TPOT (ms):                          10.30
Median TPOT (ms):                        10.23
P99 TPOT (ms):                           10.54

Mean ITL (ms):                           10.22
Median ITL (ms):                         10.21
P99 ITL (ms):                            11.06
==================================================
```

`max_num_seqs=1`과 비교하면 Request Throughput은 약 3.6배 증가했고 Mean TTFT는 약 77.5% 감소했다. 반면 Mean TPOT과 ITL은 약 7% 정도 증가했다.

즉 동시에 더 많은 Sequence를 처리하면서 Queueing Delay가 크게 줄어들었고 전체 처리량은 증가했지만, 개별 Decode Token의 처리시간은 조금 증가하는 Trade-off가 나타났다.

---

## 21. `max_num_seqs=8`

다음 설정:

```yaml
- "--max-num-seqs"
- "8"
```

결과:

```text
============ Serving Benchmark Result ============
Successful requests:                     100
Failed requests:                         0
Benchmark duration (s):                  18.75

Request throughput (req/s):              5.33
Output token throughput (tok/s):         682.54
Peak output token throughput (tok/s):    784.00
Total token throughput (tok/s):          3567.35

Mean TTFT (ms):                          1430.87
Median TTFT (ms):                        1504.68
P99 TTFT (ms):                           1777.11

Mean TPOT (ms):                          10.63
Median TPOT (ms):                        10.68
P99 TPOT (ms):                           11.03

Mean ITL (ms):                           10.55
Median ITL (ms):                         10.26
P99 ITL (ms):                            24.94
==================================================
```

Throughput과 TTFT는 다시 크게 개선됐다.

특히 Mean TTFT는 약 1.43초까지 감소했다. 하지만 P99 ITL은 24.94ms로 올라가기 시작했다. 평균적인 Token 생성 간격은 크게 변하지 않았지만 일부 Token에서는 Scheduling과 Batch 처리 영향으로 지연이 더 크게 발생한 것을 볼 수 있었다.

---

## 22. `max_num_seqs=16`

마지막으로 Client Concurrency와 동일한 16까지 올렸다.

```yaml
- "--max-num-seqs"
- "16"
```

결과:

```text
============ Serving Benchmark Result ============
Successful requests:                     100
Failed requests:                         0
Maximum request concurrency:             16
Benchmark duration (s):                  11.09

Request throughput (req/s):              9.02
Output token throughput (tok/s):         1154.04
Peak output token throughput (tok/s):    1520.00
Total token throughput (tok/s):          6031.66

Mean TTFT (ms):                          178.24
Median TTFT (ms):                        166.92
P99 TTFT (ms):                           477.14

Mean TPOT (ms):                          11.30
Median TPOT (ms):                        11.24
P99 TPOT (ms):                           12.12

Mean ITL (ms):                           11.22
Median ITL (ms):                         10.48
P99 ITL (ms):                            52.34
==================================================
```

이번 실험 범위에서는 `max_num_seqs=16`이 가장 높은 Throughput과 가장 낮은 TTFT를 기록했다.

다만 P99 ITL은 52.34ms까지 증가했다. 따라서 단순히 Throughput만 보면 `16`이 가장 좋지만 모든 Latency 지표가 동시에 좋아진 것은 아니다.

---

# 결과 분석

## 23. 전체 결과 비교

| max_num_seqs | Request Throughput | Output Token Throughput | Mean TTFT | P99 TTFT | Mean TPOT | Mean ITL | P99 ITL |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | 0.80 req/s | 102.87 tok/s | 17,200 ms | 18,670 ms | **9.58 ms** | **9.51 ms** | **10.15 ms** |
| 4 | 2.91 req/s | 371.84 tok/s | 3,870 ms | 4,377 ms | 10.30 ms | 10.22 ms | 11.06 ms |
| 8 | 5.33 req/s | 682.54 tok/s | 1,431 ms | 1,777 ms | 10.63 ms | 10.55 ms | 24.94 ms |
| 16 | **9.02 req/s** | **1154.04 tok/s** | **178 ms** | **477 ms** | 11.30 ms | 11.22 ms | 52.34 ms |

`max_num_seqs=1`에서 `16`으로 증가시키자 Request Throughput은 `0.80 → 9.02 req/s`로 약 **11.3배 증가**했고 Output Token Throughput은 `102.87 → 1154.04 tok/s`로 약 **11.2배 증가**했다.

Mean TTFT는 `17.2초 → 178ms`까지 감소해 약 **99% 개선**됐다.

반면 Mean TPOT은 `9.58 → 11.30ms`, Mean ITL은 `9.51 → 11.22ms`로 소폭 증가했다. P99 ITL은 `10.15 → 52.34ms`로 더 크게 증가했다.

---

## 24. 왜 Throughput은 증가했는가

`max_num_seqs=1`에서는 Client가 동시에 16개 요청을 보내도 Server 내부에서는 한 Sequence만 Active하게 처리할 수 있었다.

```text
Client Concurrency = 16

R1 ─┐
R2 ─┤
R3 ─┤
... │
R16 ┘
     ↓
   Queue
     ↓
 [Sequence 1]
```

따라서 GPU의 Batch 병렬성을 거의 활용하지 못한다.

`max_num_seqs`를 높이면 여러 Sequence가 동시에 GPU Batch에 참여할 수 있다.

```text
max_num_seqs=16

[R1][R2][R3][R4]
[R5][R6][R7][R8]
[R9][R10][R11][R12]
[R13][R14][R15][R16]
          ↓
         GPU
```

GPU가 모델 Weight를 읽은 상태에서 여러 Sequence의 계산을 함께 수행할 수 있기 때문에 전체 Token Throughput이 증가한다.

---

## 25. 왜 TTFT는 크게 감소했는가

이번 실험에서 가장 큰 변화는 TTFT였다.

```text
max_num_seqs=1   → Mean TTFT 17.2 s
max_num_seqs=4   → Mean TTFT 3.87 s
max_num_seqs=8   → Mean TTFT 1.43 s
max_num_seqs=16  → Mean TTFT 0.178 s
```

`max_num_seqs=1`에서는 요청 대부분이 Queue에 들어가 앞선 요청이 처리되기를 기다렸다.

즉 TTFT 안에서 실제 Prefill 계산 시간보다 **Queueing Delay가 매우 큰 비중을 차지한 것**으로 볼 수 있다.

`max_num_seqs`가 증가하자 더 많은 요청이 Active Batch에 들어갈 수 있었고 Queue에서 기다리는 시간이 크게 줄어들었다. 그 결과 TTFT가 급격하게 감소했다.

---

## 26. 왜 TPOT과 ITL은 오히려 증가했는가

Batching이 증가한다고 모든 Latency가 좋아지는 것은 아니다.

`max_num_seqs=1`에서는 한 Sequence가 GPU를 거의 혼자 사용하지만, `max_num_seqs=16`에서는 여러 Sequence가 GPU Compute와 Memory Bandwidth를 공유한다.

```text
max_num_seqs=1

GPU
 ↓
A


max_num_seqs=16

GPU
 ↓
A B C D E F ...
```

따라서 전체 Throughput은 크게 증가하지만 각 Sequence 입장에서는 Decode 단계에서 하나의 Token을 생성하는 시간이 약간 증가할 수 있다.

이번 결과에서도 Mean TPOT은 다음과 같이 증가했다.

```text
9.58 → 10.30 → 10.63 → 11.30 ms
```

즉 이번 실험에서 확인한 핵심은 다음과 같다.

```text
Batching 증가
    ↓
GPU 병렬성 증가
    ↓
Throughput 증가
Queue 감소
TTFT 감소

하지만
    ↓
Sequence 간 GPU 자원 공유 증가
    ↓
TPOT / ITL 증가 가능
Tail Latency 증가 가능
```

---

## 27. P99 ITL이 증가한 이유를 어떻게 봐야 할까

P99 ITL은 다음과 같이 변했다.

```text
10.15 → 11.06 → 24.94 → 52.34 ms
```

Mean ITL은 9~11ms 수준으로 비교적 안정적이었지만 높은 `max_num_seqs`에서 P99가 크게 증가했다.

이는 대부분의 Token은 비슷한 속도로 Streaming되더라도 일부 Decode iteration에서는 다른 Sequence의 Scheduling이나 Batch 처리 때문에 Token 사이의 간격이 길어질 수 있다는 것을 보여준다.

따라서 Production Serving에서는 Mean Latency만 보는 것이 아니라 **P95/P99와 같은 Tail Latency를 반드시 같이 봐야 한다.**

---

## 28. `max_num_seqs=16`이 최적값인가

이번 실험에서는 16이 가장 높은 Throughput과 가장 낮은 TTFT를 보여줬다.

하지만 이를 두고 "L40S에서 최적값은 16이다"라고 결론 내리는 것은 정확하지 않다.

이번 Client Concurrency 자체가 16이었기 때문이다.

```text
Client max-concurrency = 16

Server max_num_seqs
1 → 4 → 8 → 16
```

즉 이번에 확인할 수 있는 것은 다음 정도다.

> **이번 Hardware, Model, Input/Output Length, Client Concurrency 조건과 1/4/8/16이라는 실험 범위에서는 `max_num_seqs=16`이 Throughput과 TTFT 기준 가장 좋은 결과를 보였다.**

실제 최적값을 찾으려면 다음과 같이 더 높은 Client Concurrency와 Server `max_num_seqs`를 조합해 Saturation Point까지 확인해야 한다.

```text
Client Concurrency
1 / 4 / 8 / 16 / 32 / 64

Server max_num_seqs
8 / 16 / 32 / 64
```

Throughput 증가가 멈추는 지점과 TTFT/P99가 급격히 악화되는 지점을 같이 보면 실제 서비스의 Capacity와 적절한 설정값을 판단할 수 있다.

---

## 29. Prefix Caching과 Chunked Prefill

이번 vLLM 로그에서는 다음 기능이 기본 활성화되어 있었다.

```text
enable_prefix_caching=True
enable_chunked_prefill=True
```

이번 실험에서는 이 기능들을 변경하지 않고 모든 Test에서 동일하게 유지했다.

### Prefix Caching

여러 Request가 동일한 Prompt Prefix를 공유할 경우 이미 계산한 KV Cache를 재사용하는 방식이다.

```text
공통 System Prompt
       ↓
KV Cache 생성
       ↓
후속 Request에서 재사용
```

반복되는 System Prompt가 많은 Chatbot이나 Agent 환경에서 중복 Prefill 계산을 줄일 수 있다.

### Chunked Prefill

긴 Prompt의 Prefill을 한 번에 모두 실행하면 기존 Decode Request가 오랫동안 대기할 수 있다.

```text
Long Prefill
████████████████████
```

Chunked Prefill은 긴 Prefill을 여러 Chunk로 나누고 Decode 작업과 섞어 Scheduling할 수 있도록 한다.

```text
Prefill Chunk
████

Decode
██

Prefill Chunk
████
```

이를 통해 긴 Prompt가 Decode Request를 장시간 Block하는 문제를 완화할 수 있다.

---

## 30. 이번 실습에서 이해한 LLM Serving 흐름

이번 실습을 하나의 흐름으로 연결하면 다음과 같다.

```text
LLM은 Autoregressive하게 Token을 하나씩 생성
        ↓
단일 Request만 처리하면 GPU 병렬성 활용이 낮음
        ↓
Batching으로 여러 Sequence를 함께 처리
        ↓
실제 Online Traffic에서는 요청 길이와 도착 시점이 다름
        ↓
Continuous Batching 필요
        ↓
완료된 Sequence 제거 + 새 Request 즉시 투입
        ↓
vLLM Scheduler가 Active Sequence 관리
        ↓
max_num_seqs로 동시 Sequence 상한 조절
        ↓
max_num_seqs 증가
        ↓
Queue 감소 + GPU 병렬성 증가
        ↓
Throughput 증가
TTFT 감소
        ↓
하지만 GPU 자원을 여러 Sequence가 공유
        ↓
TPOT / ITL / Tail Latency 증가 가능
        ↓
결국 Throughput과 Latency의 Trade-off를 보고 튜닝
```

---

## 31. 실습을 마치며

이번 실습 전에는 `Continuous Batching`이나 `max_num_seqs`를 단순히 "동시에 여러 요청을 처리하기 위한 옵션" 정도로 생각했다. 실제 Benchmark를 해보니 이 설정 하나가 Throughput뿐 아니라 TTFT와 TPOT, ITL에 서로 다른 방식으로 영향을 준다는 점이 더 분명해졌다.

특히 처음 `max_num_seqs=1`로 실행했을 때 TPOT은 약 9.58ms로 빠른데도 Mean TTFT가 17초가 넘는 것이 처음에는 이상하게 느껴졌다. 하지만 Client에서는 16개의 요청이 동시에 들어오는데 Server는 한 Sequence만 처리하고 있었기 때문에 대부분의 Request가 실제 계산보다 Queue에서 더 오래 기다리고 있었다는 것을 이해하고 나니 결과가 연결됐다.

반대로 `max_num_seqs=16`에서는 Mean TTFT가 178ms까지 줄고 Request Throughput도 약 11배 증가했지만 P99 ITL은 52ms까지 증가했다. 이를 통해 LLM Serving에서 "빠르다"는 것이 하나의 지표로 결정되는 것이 아니라는 점도 확인했다. 사용자가 답변을 얼마나 빨리 보기 시작하는지, Token이 얼마나 부드럽게 생성되는지, 서버가 동시에 몇 명을 처리할 수 있는지를 모두 함께 봐야 한다.

또 이번 실습에서는 단순 Benchmark뿐 아니라 Kubernetes 환경에서 vLLM을 실제로 배포하면서 HAMi를 통한 GPU Memory 격리, Service 연결, Model Download, API Serving까지 전체 흐름을 경험했다. 특히 Service 이름을 `vllm`으로 사용하면서 Kubernetes가 자동 생성한 `VLLM_PORT` 환경변수와 vLLM 내부 환경변수가 충돌해 CrashLoopBackOff가 발생한 문제는 실제 운영 환경에서는 Serving Framework 자체뿐 아니라 Kubernetes 동작 방식까지 같이 이해해야 한다는 점을 보여줬다.

결국 LLM Serving 최적화는 특정 값을 무조건 크게 설정하는 작업이 아니라, **실제 Traffic Pattern과 서비스 SLA를 기준으로 Throughput과 Latency 사이의 균형점을 찾는 작업**이라고 이해했다.

---

## 32. 핵심 정리

| 개념 | 의미 |
|---|---|
| Prefill | Prompt 전체를 처리하고 초기 KV Cache를 생성하는 단계 |
| Decode | 다음 Token을 Autoregressive하게 하나씩 생성하는 단계 |
| Batching | 여러 Sequence를 묶어 GPU에서 함께 처리하는 방식 |
| Continuous Batching | 완료된 Sequence를 제거하고 새 Request를 지속적으로 Batch에 투입하는 방식 |
| `max_num_seqs` | vLLM Scheduler가 동시에 Active하게 관리할 수 있는 Sequence 수의 상한 |
| Client Concurrency | Client가 서버에 동시에 보낼 수 있는 Request 수 |
| Request Throughput | 초당 완료되는 Request 수 |
| Output Token Throughput | 초당 생성되는 Output Token 수 |
| TTFT | Request부터 첫 Token까지의 시간 |
| TPOT | 첫 Token 이후 Output Token 하나를 생성하는 평균 시간 |
| ITL | Streaming 중 연속 Token 사이의 시간 |
| P99 | 약 99%의 요청이 해당 값 이하에 위치하는 Tail Latency 지표 |
| Prefix Caching | 동일 Prompt Prefix의 KV Cache를 재사용하는 최적화 |
| Chunked Prefill | 긴 Prefill을 Chunk 단위로 나눠 Decode와 Scheduling하는 방식 |
| HAMi | Kubernetes에서 GPU Sharing과 GPU Memory 격리를 제공하는 Scheduler/Device 관리 계층 |

이번 실험의 핵심 결과는 다음과 같다.

```text
max_num_seqs
1 → 4 → 8 → 16

Request Throughput
0.80 → 2.91 → 5.33 → 9.02 req/s

Mean TTFT
17.2s → 3.87s → 1.43s → 0.178s

Mean TPOT
9.58 → 10.30 → 10.63 → 11.30ms

P99 ITL
10.15 → 11.06 → 24.94 → 52.34ms
```

Continuous Batching을 확대하면서 GPU 활용 효율과 전체 처리량이 크게 향상되고 Queueing Delay가 줄어 TTFT가 개선됐지만, 동시에 개별 Token Latency와 Tail Latency는 증가할 수 있었다.

따라서 vLLM의 `max_num_seqs`를 비롯한 Serving 설정은 하나의 지표만 최대화하는 방식으로 결정하기보다 **Throughput, TTFT, TPOT, ITL, P99 Latency와 실제 서비스의 SLA를 함께 고려해 조정해야 한다.**
