---
layout: post
title: "[Hands-On LLM Serving and Optimization] Chapter 1·2 "
date: 2026-07-26 14:00:00 +0900
categories: [LLMOps, LLM Serving]
tags: [LLM, Model Serving, Kubernetes]
published: true
---

# Hands-On LLM Serving and Optimization

![Hands-On LLM Serving and Optimization book cover](/img/study-LLM_Serving_and_Optimization/book_cover.png)

# Chapter 1. Introduction to Model Serving and Optimization

최근 AI 분야에서는 성능이 뛰어난 LLM(Large Language Model)이 계속 등장하고 있다. 하지만 아무리 좋은 모델이라도 실제 서비스에서 사용할 수 없다면 그 가치는 크게 떨어진다.

예전에는 모델을 학습시키는 것 자체가 가장 중요한 목표였다면, 이제는 **학습이 완료된 모델을 얼마나 안정적으로 서비스할 수 있는지**가 더욱 중요해지고 있다.

이번 장에서는 Model Serving이 무엇인지, 왜 중요한지, 그리고 하나의 모델이 어떤 구성 요소로 이루어져 있는지 살펴본다.

---

# AI 모델은 어떻게 서비스될까?

우리가 ChatGPT나 Claude 같은 서비스를 사용할 때는 단순히 질문만 입력하면 된다.

하지만 내부적으로는 다음과 같은 과정이 이루어진다.

```
사용자 요청

↓

Model Server

↓

LLM Inference

↓

응답 반환
```

사용자는 모델을 직접 실행하는 것이 아니라 API를 호출하고, Model Server가 요청을 받아 모델을 실행한 뒤 결과를 반환한다.

즉, 우리가 사용하는 대부분의 AI 서비스는 **Model Serving**을 통해 제공된다.

책에서도 AI 모델을 만드는 것보다 실제 서비스 환경에서 안정적으로 운영하는 것이 점점 더 중요해지고 있다고 설명한다.

나 역시 지금까지는 Kubernetes 위에 AI 서비스를 올리는 것만 생각했는데, 앞으로는 Model Serving 자체를 이해해야 실제 운영 환경을 제대로 구성할 수 있을 것 같다는 생각이 들었다.

---

# Model Serving이란?

Model Serving은 학습이 완료된 모델을 실제 서비스에서 사용할 수 있도록 배포하고 운영하는 과정이다.

학습이 끝난 모델은 다양한 방식으로 서비스될 수 있다.

- REST API
- gRPC
- Batch Processing
- Streaming

사용자는 HTTP나 gRPC 요청을 보내고, Model Serving 시스템은 해당 요청을 모델에게 전달하여 추론 결과를 반환한다.

겉으로 보기에는 단순히 API 하나를 호출하는 것처럼 보이지만 실제로는

- 모델 로드
- 메모리 관리
- GPU 사용
- 요청 처리
- 응답 반환

등 여러 과정이 함께 이루어진다.

---

# Model은 무엇으로 구성될까?

처음에는 모델이라고 하면 Weight 파일 하나만 생각했다.

하지만 책에서는 모델을 하나의 프로그램이라고 설명한다.

실제로 모델은 크게 세 가지 요소로 구성된다.

- Model Data
- Model Architecture
- Model Execution Code

이 세 가지가 모두 있어야 하나의 모델을 정상적으로 실행할 수 있다.

---

## Model Data

Model Data는 학습 과정에서 생성되는 데이터이다.

대표적으로 다음 정보들이 포함된다.

- Weight
- Bias
- Configuration

Weight와 Bias는 모델이 학습을 통해 얻은 값이다.

Configuration에는 모델 실행에 필요한 다양한 설정이 저장된다.

예를 들어

- Input Shape
- Output 정보
- Label 정보
- Batch Size

등이 포함될 수 있다.

처음에는 Weight만 있으면 모델이라고 생각했는데, 실제로는 Configuration도 함께 관리되어야 동일한 환경에서 모델을 실행할 수 있다는 점을 알게 되었다.

---

## Model Architecture

Model Architecture는 모델의 구조를 정의한다.

입력이 들어왔을 때 어떤 Layer를 거쳐 결과를 만들 것인지 결정하는 부분이다.

예를 들어 Transformer, CNN, ResNet 같은 모델들도 각각 Architecture가 다르다.

PyTorch에서는 일반적으로 `nn.Module`을 이용하여 모델 구조를 정의한다.

같은 Weight라도 Architecture가 다르면 정상적으로 실행되지 않는다.

즉,

Model Data와 Model Architecture는 항상 함께 관리되어야 한다.

---

## Model Execution Code

Execution Code는 모델을 실제 실행하는 코드이다.

일반적으로 다음과 같은 순서로 동작한다.

1. 모델 생성
2. Weight 로드
3. Evaluation Mode 설정
4. 입력 데이터 전달
5. 추론 수행
6. 결과 반환

Weight만 가지고는 추론을 수행할 수 없으며, 이를 실행하는 코드가 반드시 필요하다.

책에서도 모델은 단순한 파일이 아니라 **실행 가능한 프로그램**이라는 점을 강조한다.

---

# Model을 프로그램으로 바라보기

이번 내용을 읽으면서 가장 인상 깊었던 부분은 모델을 단순한 AI 파일이 아니라 하나의 프로그램으로 바라봐야 한다는 점이었다.

지금까지는 모델을 다운로드해서 실행하는 정도로만 생각했지만, 실제 서비스에서는

- 모델 구조
- 학습된 데이터
- 실행 코드

모두가 하나의 구성 요소로 관리된다.

특히 Model Serving에서는 모델을 소프트웨어 Artifact처럼 배포하고 운영하기 때문에, 앞으로 Kubernetes나 MLOps 환경을 공부하면서도 이러한 관점이 중요할 것이라는 생각이 들었다.

# Model Lifecycle

모델 개발은 단순히 학습만 하고 끝나는 과정이 아니다.

실제 서비스에서는 모델을 학습한 이후에도 지속적으로 운영하고 개선하는 과정이 반복된다. 책에서는 이러한 전체 과정을 **Model Lifecycle**이라고 설명한다.

Model Lifecycle은 일반적으로 다음과 같은 단계로 이루어진다.

```
Data Collection
        ↓
Training
        ↓
Evaluation
        ↓
Deployment
        ↓
Serving
        ↓
Monitoring
        ↓
Optimization
        ↓
Retraining
```

서비스가 운영되는 동안에도 새로운 데이터와 사용자 피드백이 계속 생성되기 때문에 이 과정은 한 번으로 끝나는 것이 아니라 지속적으로 반복된다.

---

## Data Collection

모델의 시작은 데이터 수집이다.

모델의 성능은 어떤 데이터를 학습하느냐에 따라 크게 달라지기 때문에 충분한 양의 데이터를 확보하는 것이 중요하다.

수집 대상은 프로젝트마다 다르다.

- 이미지
- 텍스트
- 로그 데이터
- 사용자 입력
- 문서 데이터

최근 LLM에서는 문서나 웹 데이터뿐 아니라 기업 내부 문서를 활용하는 경우도 많다.

---

## Training

수집한 데이터를 이용하여 모델을 학습하는 단계이다.

Training 과정에서는 Weight와 Bias가 계속 업데이트되면서 모델이 점점 원하는 결과를 출력하도록 학습된다.

최근에는 Foundation Model을 처음부터 학습하기보다는 기존 모델을 Fine-tuning하거나 LoRA 같은 기법을 이용하는 경우가 많다.

---

## Evaluation

학습이 끝났다고 바로 서비스할 수 있는 것은 아니다.

먼저 모델이 원하는 성능을 만족하는지 검증해야 한다.

평가에는 다양한 지표가 사용된다.

- Accuracy
- Precision
- Recall
- F1 Score
- Loss

LLM에서는 정답이 하나로 정해져 있지 않은 경우가 많기 때문에 사람이 직접 평가하거나 LLM을 이용해 평가하는 방식도 많이 사용된다.

---

## Deployment

평가가 완료된 모델은 실제 서비스 환경으로 배포된다.

이 단계에서는 모델을 단순한 파일이 아니라 하나의 소프트웨어처럼 관리하게 된다.

예를 들어

- Docker Image 생성
- Registry 업로드
- Kubernetes 배포

등의 과정이 이루어진다.

회사에서도 AI 서비스를 배포할 때 결국 컨테이너 형태로 관리하기 때문에 익숙한 과정이라는 생각이 들었다.

---

## Serving

Deployment가 끝난 모델은 실제 사용자 요청을 처리한다.

사용자는 REST API나 gRPC를 호출하고,

Model Server는 요청을 받아 모델을 실행한 뒤 결과를 반환한다.

Serving 단계에서는 단순히 모델이 실행되는 것뿐 아니라

- 요청 처리
- GPU 사용
- 메모리 관리
- Batch 처리
- 동시 요청 처리

등도 함께 고려해야 한다.

이번 책에서 가장 중요하게 다루는 부분도 바로 이 Serving 영역이다.

---

## Monitoring

서비스가 시작되면 모델이 정상적으로 동작하는지 계속 확인해야 한다.

대표적으로 다음과 같은 항목을 모니터링한다.

- CPU 사용률
- GPU 사용률
- Memory 사용량
- 응답 시간(Latency)
- Throughput
- Error Rate

실제 서비스에서는 모델보다 이러한 운영 지표를 더 자주 확인하는 경우도 많다.

특히 GPU는 비용이 매우 크기 때문에 GPU 사용률을 지속적으로 모니터링하는 것이 중요하다.

---

## Optimization

모델은 서비스를 시작했다고 끝나는 것이 아니다.

운영하면서 계속 성능을 개선해야 한다.

Optimization의 목적은 크게 두 가지이다.

- 더 빠르게 응답하기
- 더 적은 비용으로 운영하기

이를 위해 다양한 최적화 기법을 적용한다.

- Quantization
- Pruning
- Batch Inference
- KV Cache
- Tensor Parallelism

앞으로 이 책에서도 이러한 최적화 기법들을 하나씩 다루게 된다.

---

## Retraining

서비스를 운영하다 보면 새로운 데이터가 계속 생성된다.

이 데이터를 이용하여 모델을 다시 학습하면 이전보다 더 좋은 성능을 얻을 수 있다.

즉,

Model Lifecycle은

학습 → 서비스 → 개선 → 재학습

이라는 순환 구조를 계속 반복한다.

---

# 왜 Model Serving을 공부해야 할까?

예전에는 AI 모델을 만드는 것이 가장 중요한 기술이라고 생각했다.

하지만 최근에는 오픈소스 모델이 많아지고 Hugging Face처럼 이미 학습된 모델도 쉽게 사용할 수 있게 되었다.

이제는 **모델을 얼마나 잘 만드는가보다 모델을 얼마나 안정적으로 서비스할 수 있는가**가 더 중요한 경쟁력이 되고 있다.

예를 들어 ChatGPT를 사용할 때 사용자는

- 모델이 몇 개의 Layer인지
- 어떤 Optimizer를 사용했는지

에는 관심이 없다.

사용자가 원하는 것은

- 빠른 응답
- 안정적인 서비스
- 항상 동일한 품질

이다.

즉, 서비스 관점에서는 Training보다 Serving이 훨씬 중요할 수도 있다.

책에서도 이러한 이유 때문에 앞으로 AI 엔지니어라면 Model Serving을 반드시 이해해야 한다고 설명한다.

이 부분을 읽으면서 Kubernetes나 MLOps를 공부하는 이유도 결국 여기에 있다는 생각이 들었다.

모델 하나를 잘 만드는 것보다, 수많은 사용자의 요청을 안정적으로 처리하는 플랫폼을 만드는 것이 앞으로 더 중요한 기술이 될 것 같다.

# Why Model Serving Optimization?

최근 LLM의 크기는 계속 커지고 있다.

초기의 머신러닝 모델과 비교하면 수십 배, 수백 배 이상 커졌으며, 하나의 모델이 수십 GB에서 수백 GB까지 사용하는 경우도 흔하다.

모델이 커질수록 성능은 좋아질 수 있지만, 그만큼 서비스 환경에서는 새로운 문제가 발생한다.

예를 들어

- 모델을 메모리에 올리는 시간이 오래 걸리고
- GPU 메모리 사용량이 증가하며
- 추론 속도가 느려지고
- 운영 비용도 함께 증가한다.

결국 성능이 좋은 모델을 만드는 것과 실제 서비스에서 효율적으로 운영하는 것은 전혀 다른 문제라는 것을 알 수 있다.

그래서 Model Serving에서는 **Optimization**이 매우 중요한 역할을 한다.

---

## Optimization이 필요한 이유

모델이 점점 커지면서 다음과 같은 문제들이 발생한다.

### 높은 Latency

사용자가 요청을 보낸 후 응답을 받을 때까지 시간이 오래 걸릴 수 있다.

LLM은 일반적인 머신러닝 모델보다 연산량이 훨씬 많기 때문에 추론 시간이 길어질 수밖에 없다.

서비스에서는 이러한 응답 시간을 최소화하는 것이 매우 중요하다.

---

### GPU Memory 부족

LLM은 대부분 GPU에서 실행된다.

하지만 GPU 메모리는 무한하지 않으며, 하나의 모델만으로도 대부분의 메모리를 사용할 수 있다.

모델이 커질수록

- 더 큰 GPU가 필요하거나
- 여러 GPU를 함께 사용해야 하는 상황이 발생한다.

GPU는 비용이 매우 비싸기 때문에 메모리를 얼마나 효율적으로 사용하는지도 중요한 요소이다.

---

### Throughput 감소

추론 시간이 길어질수록 같은 시간 동안 처리할 수 있는 요청 수도 줄어든다.

예를 들어 하나의 요청을 처리하는 데 1초가 걸린다면 동시에 많은 사용자가 접속했을 때 병목 현상이 발생할 수 있다.

서비스에서는 단순히 응답 속도뿐 아니라 얼마나 많은 요청을 처리할 수 있는지도 중요하다.

---

### 운영 비용 증가

최근 AI 서비스의 가장 큰 고민 중 하나는 GPU 비용이다.

모델이 커질수록

- GPU 사용량 증가
- 전력 소비 증가
- 운영 비용 증가

로 이어진다.

따라서 성능을 유지하면서도 GPU 사용량을 줄이는 것이 중요한 과제가 된다.

책에서도 Optimization은 단순히 속도를 높이는 기술이 아니라 **서비스 비용을 줄이는 기술**이라는 점을 강조한다.

---

# Model Serving에서 중요한 지표

Model Serving에서는 여러 가지 성능 지표를 함께 확인한다.

대표적으로 다음과 같은 지표들이 있다.

## Latency

사용자가 요청을 보낸 후 응답을 받을 때까지 걸리는 시간이다.

Latency가 낮을수록 사용자는 더 빠른 서비스를 경험할 수 있다.

실시간 서비스에서는 가장 중요한 지표 중 하나이다.

---

## Throughput

일정 시간 동안 처리할 수 있는 요청의 개수를 의미한다.

예를 들어 초당 100개의 요청을 처리할 수 있다면 Throughput은 100 Requests/sec가 된다.

서비스 규모가 커질수록 Throughput의 중요성도 함께 증가한다.

---

## Resource Utilization

CPU와 GPU, 메모리를 얼마나 효율적으로 사용하는지를 의미한다.

GPU 사용률이 너무 낮다면 자원을 낭비하는 것이고,

반대로 너무 높으면 병목 현상이 발생할 수 있다.

따라서 적절한 자원 활용이 중요하다.

---

## Scalability

사용자가 많아졌을 때 서비스가 얼마나 쉽게 확장될 수 있는지를 의미한다.

요청이 증가하면 서버를 추가하여 처리량을 늘릴 수 있어야 한다.

Model Serving Framework에서도 이러한 확장 기능을 기본적으로 제공하는 경우가 많다.

---

## Availability

서비스가 얼마나 안정적으로 운영되는지를 나타낸다.

서비스 장애가 발생하거나 모델이 정상적으로 응답하지 못하면 사용자 경험에 큰 영향을 주게 된다.

실제 서비스에서는 성능뿐 아니라 안정성도 매우 중요한 요소이다.

---

이번 장에서는 Model Serving이 왜 필요한지와 Optimization이 왜 중요한지에 대해 살펴보았다.

이후 장에서는 이러한 문제들을 해결하기 위해 다양한 Model Serving Framework와 Optimization 기법들을 하나씩 살펴보게 된다.

대표적으로

- Model Server
- Dynamic Batching
- Quantization
- Tensor Parallelism
- KV Cache
- Continuous Batching

등이 등장하며, 이러한 기술들을 통해 LLM을 더 빠르고 효율적으로 서비스하는 방법을 배우게 된다.

이번 장을 읽으면서 가장 크게 느낀 점은 AI 서비스에서 중요한 것은 단순히 좋은 모델을 만드는 것이 아니라, **좋은 모델을 얼마나 효율적으로 운영할 수 있는가**라는 점이었다.

앞으로 책에서 다루게 될 여러 최적화 기법들도 결국 이 문제를 해결하기 위한 방법이라는 생각이 들었다.

---
# Model Serving Paradigms

Model Serving은 하나의 방식만 존재하는 것이 아니다.

서비스 규모와 운영 환경에 따라 다양한 구조를 사용할 수 있으며, 각각 장단점이 존재한다.

책에서는 대표적인 Model Serving 방식으로 Single Model Service와 Multi Model Service를 소개한다.

초기에는 하나의 서버가 하나의 모델만 담당하는 구조가 일반적이었지만, AI 서비스가 많아지고 운영해야 하는 모델 수도 증가하면서 보다 효율적인 구조가 필요하게 되었다.

---

# Single Model Service

Single Model Service는 하나의 서비스가 하나의 모델만 담당하는 구조이다.

모델마다 독립적인 API와 실행 환경을 가지며, 하나의 컨테이너에는 하나의 모델만 실행된다.

구조가 단순하기 때문에 가장 이해하기 쉬운 방식이기도 하다.

서비스 구조를 간단히 표현하면 다음과 같다.

```

Client

↓

API

↓

Model Server

↓

Single Model

```

각 모델이 독립적으로 동작하기 때문에 서로 영향을 주지 않는다는 것이 가장 큰 특징이다.

---

## 장점

Single Model Service는 운영이 단순하다.

모델별로 독립적인 환경을 가지기 때문에 문제가 발생하더라도 다른 모델에는 영향을 주지 않는다.

또한 모델별로 버전 관리나 배포를 수행하기도 쉽다.

대표적인 장점은 다음과 같다.

- 장애 격리
- 독립적인 배포
- 개별적인 Scale-out
- 관리가 비교적 쉬움

초기 AI 서비스를 구축하거나 모델 개수가 많지 않은 경우에는 지금도 많이 사용하는 방식이다.

---

## 단점

하지만 서비스해야 하는 모델이 많아질수록 단점도 나타난다.

예를 들어 100개의 모델을 서비스해야 한다면

100개의 Model Server가 필요할 수도 있다.

그만큼

- GPU 사용량 증가
- 메모리 낭비
- 운영 비용 증가

등의 문제가 발생한다.

같은 GPU 안에서도 모델 하나만 실행되고 남는 자원이 발생할 수도 있기 때문에 자원 활용 효율이 떨어질 수 있다.

책에서도 이러한 구조는 운영 규모가 커질수록 한계가 발생한다고 설명한다.

---

# Multi Model Service

Single Model Service의 단점을 해결하기 위해 등장한 구조가 Multi Model Service이다.

이 방식은 하나의 Model Server에서 여러 개의 모델을 함께 관리한다.

필요한 모델만 메모리에 올려 사용하고, 사용하지 않는 모델은 제거하면서 자원을 효율적으로 사용할 수 있다.

서비스 구조는 다음과 같은 형태가 된다.

```

Client

↓

Model Server

↓

Model A

Model B

Model C

Model D

```

즉, 하나의 Serving 시스템이 여러 모델을 함께 관리하는 구조이다.

---

## 장점

Multi Model Service의 가장 큰 장점은 자원 활용 효율이다.

모든 모델을 항상 메모리에 올려둘 필요가 없으며,

필요한 모델만 로드해서 사용할 수 있다.

또한

- GPU 공유
- 메모리 절약
- 운영 비용 감소

등의 효과도 얻을 수 있다.

모델 개수가 많아질수록 이러한 장점은 더욱 커진다.

---

## 단점

반대로 구조는 Single Model Service보다 복잡해진다.

Model Loading과 Unloading이 계속 발생할 수 있으며,

여러 모델이 하나의 GPU를 공유하기 때문에 자원 관리도 훨씬 어려워진다.

또한 모델 교체 과정에서 지연 시간이 발생할 수도 있다.

결국

운영 효율은 좋아지지만,

구현 난이도는 높아지는 구조라고 이해하면 될 것 같다.


---
# Model Serving Platform

Model의 개수가 많아지고 서비스 규모가 커질수록 단순히 Model Server만 실행해서는 운영하기 어려워진다.

실제 서비스에서는 모델을 배포하고, 버전을 관리하고, 자동으로 확장하며, 장애가 발생하면 복구하는 기능까지 함께 필요하다.

이러한 기능들을 제공하는 것이 **Model Serving Platform**이다.

Model Serving Platform은 단순히 모델을 실행하는 프로그램이 아니라 모델의 전체 생명주기를 관리하는 플랫폼이라고 볼 수 있다.

대표적으로 다음과 같은 기능을 제공한다.

- 모델 배포(Deployment)
- 모델 버전 관리
- Auto Scaling
- Load Balancing
- Monitoring
- Logging
- Traffic Routing
- Rollback

즉, Model Server 위에서 운영을 자동화하는 역할을 담당한다.

---

## Model Server와 Platform의 차이

처음에는 Model Server와 Model Serving Platform이 같은 의미라고 생각했다.

하지만 책에서는 두 개념을 명확하게 구분하고 있다.

Model Server는 말 그대로 모델을 실행하는 역할만 수행한다.

반면 Model Serving Platform은 여러 Model Server를 관리하면서 전체 서비스를 운영하는 역할을 담당한다.

쉽게 생각하면

```
Model Serving Platform

↓

Model Server

↓

AI Model
```

과 같은 구조라고 이해하면 된다.

실제 서비스에서는 대부분 Platform 위에서 여러 개의 Model Server가 함께 동작한다.

---

## 왜 Platform이 필요할까?

모델이 하나뿐이라면 크게 문제가 없다.

하지만 여러 팀에서 수십 개, 수백 개의 모델을 운영하게 되면 상황이 달라진다.

예를 들어

- 새로운 모델을 배포해야 하고
- 기존 모델을 새로운 버전으로 교체해야 하고
- 사용량이 증가하면 자동으로 서버를 늘려야 하며
- 장애가 발생하면 빠르게 복구해야 한다.

이러한 작업을 모두 수동으로 수행하는 것은 현실적으로 어렵다.

따라서 이러한 운영 작업을 자동화하기 위해 Model Serving Platform을 사용하게 된다.

---

## Kubernetes와 Model Serving

최근 대부분의 Model Serving Platform은 Kubernetes 위에서 동작한다.

Kubernetes는

- Container 관리
- Auto Scaling
- Load Balancing
- Self Healing

등의 기능을 제공하기 때문에 AI 서비스를 운영하기에 적합한 환경이다.

책에서도 Model Serving과 Kubernetes가 함께 사용되는 이유를 설명하고 있으며, 이후 장에서도 Kubernetes 기반의 다양한 Model Serving Framework를 다루게 된다.

현재 회사에서도 Kubernetes 기반으로 AI 서비스를 운영하고 있기 때문에 앞으로 나올 내용들이 실무와도 많이 연결될 것 같다는 생각이 들었다.

---

## 앞으로 등장하는 Framework

책에서는 이후 장에서 다양한 Model Serving Framework를 소개한다.

대표적으로 다음과 같은 Framework들이 등장한다.

- KServe
- NVIDIA Triton Inference Server
- Ray Serve
- SGLang
- vLLM

각 Framework는 목적이 조금씩 다르지만 모두 Model Serving을 효율적으로 수행하기 위한 도구들이다.

특히 최근에는 LLM 서비스가 많아지면서 GPU 활용도를 높이기 위한 기능들이 계속 추가되고 있다.

앞으로는 이러한 Framework들이 어떤 특징을 가지고 있는지 하나씩 살펴보게 된다.

---

# Chapter 1을 읽고

이번 장은 본격적으로 Model Serving을 배우기 전에 전체적인 개념을 소개하는 내용이었다.

처음에는 Model Serving을 단순히 "모델을 실행하는 것" 정도로 생각했다.

하지만 책을 읽으면서 실제 서비스에서는 모델을 실행하는 것보다 운영하는 과정이 훨씬 중요하다는 것을 알게 되었다.

특히 하나의 모델도

- Model Data
- Model Architecture
- Execution Code

세 가지 요소가 함께 있어야 한다는 점이 인상적이었다.

또한 모델을 학습시키는 것과 서비스를 운영하는 것은 전혀 다른 영역이라는 것도 이해할 수 있었다.

Training은 좋은 모델을 만드는 과정이라면,

Serving은 만들어진 모델을 사용자에게 안정적으로 제공하는 과정이다.

LLM의 크기가 계속 커지는 만큼 앞으로는 Optimization의 중요성도 더욱 커질 것 같다.

이번 장에서는 Optimization이라는 개념만 간단히 소개했지만, 이후 장에서는 Quantization이나 Batching 같은 다양한 최적화 기법들을 배우게 된다.

현재 회사에서도 Kubernetes 기반으로 AI 플랫폼을 운영하고 있기 때문에 앞으로 등장하는 KServe나 Triton, vLLM 같은 Framework들이 실제 환경에서는 어떻게 사용되는지 함께 비교하면서 공부하면 이해가 더 잘 될 것 같다.

이번 Chapter는 Model Serving을 처음 접하는 사람을 위한 전체적인 개요를 설명하는 장이었다.

앞으로는 이러한 개념을 바탕으로 실제 Model Server와 다양한 Serving Framework들을 하나씩 학습하게 된다.

<br/><br/>

---
# Chapter 2. Large Language Model Serving

Chapter 1에서는 모델 서빙의 전체적인 구조와 Single-Model Service,
Multi-Model Service를 살펴봤다.\
Chapter 2에서는 그중에서도 **LLM을 실제로 서빙할 때 내부에서 어떤 일이
일어나는지**를 조금 더 자세히 다룬다.

이번 장에서 중요하게 본 내용은 Transformer의 기본 구조 자체보다는
**LLM이 토큰을 어떻게 생성하고, 그 과정에서 왜 KV Cache, Prefill/Decode,
Streaming, Batching 같은 기술이 필요한지**였다.

------------------------------------------------------------------------

## 1. LLM의 발전과 Transformer

언어 모델은 N-gram 같은 통계 기반 언어 모델에서 시작해 Word2Vec,
RNN/LSTM을 거쳐 Transformer 기반 모델로 발전했다.

RNN과 LSTM은 문장을 순차적으로 처리하기 때문에 이전 문맥을 반영할 수
있었지만, 입력을 하나씩 처리하는 구조라 병렬화가 어렵고 긴 문장의 관계를
처리하는 데 한계가 있었다.

2017년 Transformer가 등장하면서 이러한 구조가 크게 바뀌었다.
Transformer는 **Self-Attention**을 이용해 입력 토큰 사이의 관계를
계산하고, RNN처럼 모든 입력을 순차적으로 처리하지 않아도 되기 때문에
병렬 처리가 가능하다.

이후 Transformer를 기반으로 BERT와 GPT 계열 모델이 등장했다.

-   BERT: 양방향 Encoder 기반
-   GPT: 단방향 Decoder 기반
-   최근 생성형 LLM: 주로 Decoder-only Transformer 구조 사용

이 책에서도 이후 LLM이라고 할 때 주로 **GPT, Llama, Qwen과 같은
Decoder-only Transformer**를 기준으로 설명한다.

------------------------------------------------------------------------

## 2. LLM은 한 번에 문장을 만드는 것이 아니다

LLM은 완성된 문장을 한 번에 생성하지 않는다.

기본적으로 **Autoregressive 방식으로 토큰을 하나씩 생성한다.**

예를 들어 다음과 같은 Prompt가 있다고 한다.

``` text
Write a short introduction about US capital city.
```

모델이 첫 번째 토큰으로 `Washington`을 생성했다면 다음 입력은 다음과
같이 된다.

``` text
Write a short introduction about US capital city. Washington
```

그다음 `D.C.`를 생성하면 다시 이전 입력 뒤에 붙인다.

``` text
Write a short introduction about US capital city. Washington D.C.
```

이런 방식으로 이전에 생성된 모든 토큰을 문맥으로 사용하면서 다음 토큰을
계속 예측한다.

즉,

``` text
Prompt
  ↓
다음 Token 예측
  ↓
기존 입력 + 생성된 Token
  ↓
다음 Token 예측
  ↓
반복
```

의 구조다.

처음에는 LLM이 문장을 통째로 만들어서 반환한다고 막연하게 생각했는데,
실제로는 **다음 토큰 예측을 반복한 결과가 하나의 문장이 되는 구조**라는
점을 이해하는 것이 이후 서빙 구조를 이해하는 데 중요했다.

------------------------------------------------------------------------

## 3. Decoder-only Transformer 구조

책에서는 일반적인 Decoder-only Transformer를 크게 다음 세 부분으로
나눈다.

``` text
Input Text
    ↓
Tokenizer & Embedding
    ↓
Transformer Decoder Blocks
    ↓
LM Head
    ↓
Output Token
```

### Tokenizer와 Embedding

모델은 문자열 자체를 바로 계산할 수 없다.

먼저 Tokenizer가 입력 문장을 작은 단위인 **Token**으로 나누고, 각
Token을 숫자인 **Token ID**로 변환한다.

``` text
Text
→ Token
→ Token ID
→ Embedding Vector
```

Token ID는 다시 Embedding Layer를 통해 모델이 계산할 수 있는 벡터 형태로
변환된다.

즉 Tokenizer와 Embedding은 사람이 사용하는 텍스트를 Transformer가 처리할
수 있는 숫자 표현으로 변환하는 과정이라고 볼 수 있다.

### Transformer Decoder Block

실제 대부분의 계산은 여러 개의 Decoder Block에서 수행된다.

각 Decoder Block에는 대표적으로 다음과 같은 구성요소가 있다.

``` text
Decoder Block
├── Self-Attention
└── Feed Forward Neural Network (FFN)
```

이러한 Decoder Block이 여러 층으로 쌓여 있다.

책에서 사용한 Qwen2.5-0.5B 예시에서는 다음과 같은 모델 설정을 확인했다.

``` text
Hidden size: 896
Number of layers: 24
Number of attention heads: 14
Intermediate size: 4864
Vocabulary size: 151936
Maximum position embeddings: 32768
Total parameters: 494,032,768
```

이런 모델 설정을 미리 확인하면 모델 서빙에 필요한 GPU Memory나 병렬화
방식 등을 판단하는 데 도움이 된다.

### LM Head

Decoder Block을 통과한 결과는 Hidden State 형태로 나온다.

LM Head는 이 값을 Vocabulary 전체에 대한 점수인 **Logits**로 변환한다.

``` text
Hidden State
    ↓
LM Head
    ↓
Vocabulary Logits
    ↓
Probability
    ↓
Next Token
```

이 확률 분포를 이용해서 다음 Token을 선택하고, 선택된 Token을 다시
입력에 추가하면서 다음 생성 과정을 반복한다.

------------------------------------------------------------------------

## 4. Self-Attention

Self-Attention은 현재 Token을 이해할 때 다른 Token과의 관계를 계산하는
구조다.

예를 들어 `US capital city`라는 문장에서 `capital`이라는 단어가 나왔을
때, 모델은 주변 Token과의 관계를 확인하면서 여기서 capital이 금융 자본이
아니라 **수도**라는 의미라는 것을 파악할 수 있다.

Attention 계산에서는 각 Token으로부터 다음 세 가지 벡터를 만든다.

-   Query(Q)
-   Key(K)
-   Value(V)

개념적으로는 Query와 Key를 비교해 어떤 Token을 얼마나 참고할지 계산하고,
그 결과를 이용해 Value를 가중합한다.

책에서는 이를 다음 식으로 설명한다.

``` text
Attention(Q, K, V)
= softmax(QKᵀ / √dₖ)V
```

서빙 관점에서는 수식 자체를 외우는 것보다 **Attention 계산이 이전
Token의 정보를 계속 사용한다는 것**이 더 중요했다.

이 특징 때문에 뒤에서 나오는 KV Cache가 필요해진다.

------------------------------------------------------------------------

## 5. Multi-Head Attention

하나의 Attention만 사용하는 것이 아니라 여러 개의 Attention Head를
동시에 사용한다.

각 Head는 서로 다른 Q/K/V Projection을 가지고 있기 때문에 Token 사이의
관계를 서로 다른 관점에서 볼 수 있다.

예를 들어 어떤 Head는 문법적인 관계를 강하게 보고, 다른 Head는 위치나
의미 관계를 다르게 볼 수 있다.

각 Head의 결과는 마지막에 합쳐져 다음 Layer로 전달된다.

``` text
Input
 ├─ Head 1
 ├─ Head 2
 ├─ Head 3
 └─ ...
      ↓
   Concatenate
      ↓
   Output
```

LLM Serving 관점에서는 Attention이 계산량이 큰 부분이며, 이후 KV Cache나
Paged Attention 같은 최적화가 이 영역과 밀접하게 연결된다.

------------------------------------------------------------------------

## 6. LLM Token Generation

책에서는 Hugging Face의 `pipeline()`을 사용해 Qwen2.5 모델을 간단하게
실행한 뒤, 내부적으로 Token이 어떻게 생성되는지 직접 구현해본다.

간단하게 사용할 때는 다음처럼 실행할 수 있다.

``` python
generator = pipeline(
    "text-generation",
    model="Qwen/Qwen2.5-0.5B"
)
```

하지만 내부에서는 대략 다음 과정이 반복된다.

``` text
Prompt Tokenization
        ↓
Model Forward
        ↓
Logits 생성
        ↓
Softmax
        ↓
다음 Token 선택
        ↓
기존 Input 뒤에 Token 추가
        ↓
다시 Model Forward
```

문제는 생성할 때마다 이전 전체 입력을 다시 모델에 넣으면 이미 계산했던
Token까지 계속 다시 계산한다는 점이다.

여기서 **KV Cache**가 등장한다.

------------------------------------------------------------------------

## 7. KV Cache

LLM은 새로운 Token을 생성할 때 이전 Token의 Attention 계산 결과를 계속
사용한다.

KV Cache를 사용하지 않으면 다음 Token을 만들 때마다 이전 Token의 Key와
Value를 다시 계산해야 한다.

예를 들어 입력이 계속 길어진다면,

``` text
1번째 생성
Prompt 전체 계산

2번째 생성
Prompt + Token1 전체 계산

3번째 생성
Prompt + Token1 + Token2 전체 계산
```

처럼 이미 처리한 부분까지 반복 계산하게 된다.

KV Cache는 이전 Token에서 계산한 **Key와 Value를 Memory에 저장해
재사용**한다.

``` text
이전 Token
   ↓
K, V 계산
   ↓
KV Cache 저장
   ↓
다음 Token 생성 시 재사용
```

그러면 새로운 Token을 생성할 때는 새 Token에 필요한 계산을 수행하고 이전
Token의 K/V는 Cache에서 가져올 수 있다.

책의 실험에서도 KV Cache를 사용하지 않았을 때는 입력 Sequence가
길어질수록 Token 생성 시간이 증가했지만, KV Cache를 사용하면 첫 Token
이후의 생성 시간이 훨씬 안정적으로 유지됐다.

즉 KV Cache는 **LLM 추론 속도를 높이는 핵심 기술**이다.

다만 Cache를 저장해야 하므로 GPU Memory 사용량이 증가한다. 따라서 LLM
Serving에서는 단순히 모델 Weight만 GPU Memory를 사용하는 것이 아니라 KV
Cache가 사용하는 Memory도 함께 고려해야 한다.

------------------------------------------------------------------------

## 8. Prefill과 Decode

LLM 추론은 크게 **Prefill**과 **Decode** 두 단계로 나눌 수 있다.

### Prefill

Prefill은 사용자가 처음 전달한 Prompt 전체를 처리하는 단계다.

``` text
Prompt 전체
    ↓
Transformer
    ↓
첫 번째 Token 생성
```

Prompt에 포함된 모든 Token을 처리하면서 Attention을 계산하고 KV Cache를
만든다.

여러 Prompt Token을 병렬적으로 계산할 수 있지만 처리해야 할 Token이 많기
때문에 **Compute-intensive**한 특징이 있다.

특히 입력 Prompt가 길수록 Prefill 비용이 커진다.

### Decode

Prefill이 끝난 이후 실제 답변 Token을 하나씩 생성하는 단계다.

``` text
Token 1
 ↓
Token 2
 ↓
Token 3
 ↓
...
```

KV Cache가 있기 때문에 이전 Token 전체를 다시 계산할 필요는 없지만,
생성된 Token을 하나씩 처리해야 한다.

책에서는 Decode 단계가 KV Cache를 계속 읽고 유지해야 하기 때문에
**Memory-intensive**한 특징이 있다고 설명한다.

정리하면 다음과 같다.

  구분                  Prefill            Decode
  --------------------- ------------------ -------------------
  처리                  Prompt 전체 처리   Token 하나씩 생성
  특징                  병렬 처리 가능     순차적 생성
  주요 부담             Compute            Memory / KV Cache
  영향을 크게 받는 것   Prompt 길이        출력 Token 길이

이 구분을 이해하면 LLM Serving에서 어디가 병목인지 판단하기가 쉬워진다.

긴 문서를 입력하는 서비스라면 Prefill이 중요하고, 짧은 질문에 매우 긴
답변을 생성하는 서비스라면 Decode가 더 큰 영향을 줄 수 있다.

------------------------------------------------------------------------

## 9. Serving Framework가 필요한 이유

앞에서는 직접 모델을 Load하고 Token을 하나씩 생성하는 과정을 살펴봤다.

이 방식은 LLM 내부 동작을 이해하기에는 좋지만 실제 Production 환경에서
직접 구현하기에는 고려해야 할 부분이 너무 많다.

Model Serving Framework는 이러한 부분을 대신 처리한다.

책에서는 대표적인 Framework로 **vLLM과 SGLang**을 언급한다.

Serving Framework가 담당하는 기능에는 다음과 같은 것들이 있다.

-   KV Cache 재사용
-   Request Scheduling
-   Batching / Micro-batching
-   Multi-user Concurrency
-   Token Streaming
-   Request Cancellation
-   Serving 최적화

즉 Serving Framework는 단순히 모델을 실행해주는 라이브러리가 아니라
**여러 사용자의 요청을 효율적으로 처리하기 위한 LLM 추론 엔진**에
가깝다고 이해했다.

------------------------------------------------------------------------

## 10. vLLM으로 LLM Serving

책에서는 Qwen 모델을 vLLM으로 실행한다.

기본적인 사용 방식은 비교적 단순하다.

``` python
from vllm import LLM, SamplingParams

llm = LLM(
    model="Qwen/Qwen2.5-0.5B",
    dtype="float16"
)

sampling_params = SamplingParams(
    temperature=0.8,
    top_p=0.95,
    max_tokens=100
)

outputs = llm.generate([prompt], sampling_params)
```

직접 Token 생성 Loop를 구현하는 것보다 훨씬 단순하다.

vLLM은 내부적으로 KV Cache 관리, Scheduling, Batching 등의 Serving
기능을 제공하고 여러 최적화 기법을 적용한다.

책의 간단한 비교 실험에서는 동일한 모델과 Prompt를 사용했을 때 Hugging
Face 방식보다 vLLM의 실행 시간이 훨씬 짧게 측정됐다.

다만 이 수치는 책의 특정 실험 환경에서 나온 결과이므로 모든 환경에서
동일한 성능 차이가 난다는 의미는 아니다.

중요한 점은 Production LLM Serving에서는 단순한 모델 실행보다 **동시
요청 처리, Throughput, Latency, GPU 활용률**까지 고려해야 하기 때문에
Serving Framework를 사용하는 것이 유리하다는 것이다.

------------------------------------------------------------------------

## 11. LLM Streaming Serving

LLM은 Token을 하나씩 생성하지만 모든 Token 생성이 끝난 다음 결과를
반환하면 사용자는 전체 생성 시간이 끝날 때까지 아무것도 볼 수 없다.

그래서 Chatbot 같은 서비스에서는 **Streaming**을 사용한다.

``` text
Token 생성
   ↓
즉시 사용자에게 전달
   ↓
다음 Token 생성
   ↓
즉시 전달
```

즉 전체 결과가 완성될 때까지 기다리는 것이 아니라 생성된 Token 또는 작은
Chunk를 바로 반환한다.

사용자 입장에서는 모델이 실제로 더 빨라진 것은 아니더라도 첫 번째 결과를
더 빠르게 확인할 수 있기 때문에 체감 응답성이 좋아진다.

vLLM에서는 비동기 방식의 Engine을 이용해 Streaming을 구현할 수 있다.

Streaming은 또 다른 장점도 있다. 사용자가 잘못된 응답이라고 판단하면
생성 중간에 Request를 취소할 수 있다.

불필요한 Token 생성을 중단할 수 있기 때문에 사용자 경험뿐만 아니라 GPU
자원 낭비를 줄이는 데도 도움이 된다.

------------------------------------------------------------------------

## 12. LLM Batch Serving

지금까지는 하나의 Prompt를 한 번에 처리하는 방식이었다.

하지만 실제 서비스에서는 동시에 많은 Request가 들어온다.

``` text
Request 1
Request 2
Request 3
Request 4
```

이 요청들을 하나씩 처리하면 GPU를 충분히 활용하지 못하고 Throughput이
낮아질 수 있다.

Batching은 여러 입력을 하나의 Batch로 묶어서 동시에 처리한다.

``` text
Request 1 ┐
Request 2 ├─ Batch → LLM → 여러 Output
Request 3 ┤
Request 4 ┘
```

Transformer의 Matrix 연산과 Attention 연산은 GPU에서 병렬화할 수 있기
때문에 여러 Sequence를 함께 처리하면 GPU 활용률을 높일 수 있다.

책의 간단한 실험에서도 4개의 Prompt를 각각 처리하는 것보다 Batch로
처리했을 때 전체 처리 시간이 감소했다.

다만 실제 Production 환경에서는 요청이 항상 같은 시점에 들어오지 않는다.

그래서 단순히 Batch가 채워질 때까지 기다리는 방식보다 **Continuous
Batching** 같은 방식이 사용된다.

Continuous Batching은 실행 중인 Batch에서 완료된 Request가 빠지면 새로운
Request를 계속 추가하면서 처리하는 방식이다.

이를 통해 GPU를 쉬지 않고 활용하면서 Throughput을 높일 수 있다.

------------------------------------------------------------------------

## 13. 이번 Chapter에서 연결된 흐름

이번 Chapter의 내용을 전체적으로 연결하면 다음과 같다.

``` text
사용자 Prompt
      ↓
Tokenizer
      ↓
Embedding
      ↓
Transformer Decoder Blocks
      ↓
Self-Attention
      ↓
LM Head
      ↓
Next Token 생성
      ↓
Autoregressive 반복
```

그런데 이 과정을 실제 서비스에서 그대로 수행하면 여러 문제가 발생한다.

``` text
반복 Attention 계산
→ KV Cache

Prompt 처리와 Token 생성의 특성이 다름
→ Prefill / Decode

전체 응답 완료까지 기다리면 사용자 경험 저하
→ Streaming

Request를 하나씩 처리하면 GPU 활용률 저하
→ Batching / Continuous Batching

이 모든 기능을 직접 구현하기 어려움
→ vLLM / SGLang 같은 Serving Framework
```

결국 이번 장은 Transformer 자체를 깊게 공부하는 장이라기보다는
**Transformer의 동작 방식이 실제 LLM Serving 시스템 설계에 어떤 영향을
주는지 연결해서 이해하는 장**에 가까웠다.

------------------------------------------------------------------------

## Chapter 2를 읽고

기존에는 vLLM이나 KV Cache 같은 용어를 각각 따로 접했는데, 이번 장을
보면서 왜 이런 기술이 필요한지가 조금 더 연결됐다.

특히 LLM이 Token을 하나씩 생성하는 Autoregressive 구조이기 때문에 이전
계산을 계속 재사용해야 하고, 그 과정에서 KV Cache가 중요해진다는 흐름이
이해됐다.

Prefill과 Decode도 단순히 LLM 추론의 두 단계라고만 알고 있었는데,
Prefill은 Prompt 전체를 처리하기 때문에 Compute 쪽 부담이 크고 Decode는
Token을 하나씩 생성하면서 KV Cache를 계속 사용하기 때문에 Memory 쪽
특성이 강하다는 차이가 있었다.

또 실제 LLM Serving에서는 단순히 모델을 GPU에 올려서 실행하는 것으로
끝나는 것이 아니라 여러 사용자의 Request를 어떻게 Scheduling하고,
Batch로 묶고, KV Cache를 관리하고, 생성되는 Token을 Streaming할지까지
같이 고려해야 한다.

그래서 vLLM 같은 Serving Framework가 단순히 모델 실행을 편하게 해주는
도구가 아니라 **GPU를 효율적으로 사용하면서 실제 서비스 형태로 LLM을
운영하기 위한 역할**을 한다는 점이 가장 크게 와닿았다.

Chapter 1에서 Model Serving의 전체적인 구조를 봤다면, Chapter 2에서는
그중 LLM Serving 내부에서 실제로 어떤 계산과 최적화가 필요한지를 조금 더
구체적으로 이해할 수 있었다.
