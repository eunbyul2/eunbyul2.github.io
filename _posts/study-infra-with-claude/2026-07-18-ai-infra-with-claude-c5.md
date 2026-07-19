---
layout: post
title: "[AI 인프라 모각코] 5장: 무중단 배포"
date: 2026-07-18 21:00:00 +0900
categories: [스터디, AI인프라모각코, Kubernetes, GitOps]
tags: [Kubernetes, GatewayAPI, ArgoRollouts, BlueGreen, GitOps, ClaudeCode]
published: true
---

# Gateway API와 Argo Rollouts로 무중단 배포 구현하기

지난 장에서는 Claude Code를 활용하여 애플리케이션을 배포하고 GitOps 환경을 구성했다. 애플리케이션을 정상적으로 배포하는 것까지는 성공했지만, 운영 환경에서는 단순히 배포가 되는 것만으로는 부족하다.

서비스를 사용하는 도중 새로운 버전을 배포해야 할 수도 있고, 배포 직후 예상하지 못한 문제가 발생할 수도 있다. 이런 상황에서 사용자가 서비스를 이용하는 동안 장애를 최소화하면서 안전하게 새로운 버전을 배포하는 것이 중요하다.

이번 장에서는 이러한 문제를 해결하기 위해 Gateway API와 Argo Rollouts를 이용하여 Blue/Green 배포를 구성했다. 처음에는 단순히 새로운 기능을 배우는 챕터라고 생각했는데, 실습을 따라가다 보니 실제 운영 환경에서는 왜 이런 구조를 사용하는지 조금씩 이해할 수 있었다.

---

## 1. Rolling Update만으로는 충분하지 않을까?

Kubernetes를 처음 공부하면 Deployment와 Rolling Update를 가장 먼저 배우게 된다.

나도 지금까지는 이미지를 변경하면 Pod가 하나씩 교체되고, 서비스는 계속 유지되는 방식 정도로만 이해하고 있었다.

실제로 대부분의 예제에서도 Rolling Update를 사용하기 때문에 크게 불편하다는 생각은 해본 적이 없었다.

하지만 책에서는 먼저 Rolling Update의 한계를 이야기한다.

새로운 버전을 배포하는 순간 일부 사용자는 이미 새로운 버전에 접속하게 된다. 만약 새 버전에 치명적인 문제가 있다면 그 순간부터 장애가 발생할 수도 있다.

또한 이전 버전으로 되돌리는 과정도 생각보다 간단하지 않다. 이미 Deployment가 변경된 상태이기 때문에 다시 이미지를 수정하고 재배포하는 과정이 필요하다.

즉, **서비스는 계속 운영되지만 새로운 버전을 충분히 검증할 시간은 없다**는 것이 Rolling Update의 가장 큰 아쉬운 점이었다.

책에서는 바로 이 부분을 해결하기 위해 Blue/Green 배포를 소개한다.

---

## 2. 왜 Gateway API부터 구성할까?

처음에는 조금 의아했다.

Blue/Green 배포를 배우는 챕터인데 왜 가장 먼저 Gateway API를 설정하는지 이해가 되지 않았다.

Ingress 대신 Gateway를 생성하고, HTTPRoute를 만들고, HealthCheckPolicy까지 적용하는 과정을 보면서도 '이게 Blue/Green이랑 무슨 관계가 있지?'라는 생각이 들었다.

그런데 뒤쪽 실습까지 모두 따라가 보니 이유를 알 수 있었다.

Blue/Green 배포의 핵심은 **트래픽을 어느 버전으로 보낼 것인지**이다.

새로운 버전을 실행하는 것보다 더 중요한 것은 사용자의 요청을 원하는 버전으로 안전하게 전달하는 것이다.

Gateway API는 바로 그 역할을 담당한다.

이번 실습에서는 GatewayClass를 확인하고 Gateway를 생성한 뒤 HTTPRoute를 이용하여 애플리케이션으로 요청이 전달되도록 구성했다.

기존에는 Ingress 하나에서 처리하던 기능이 Gateway API에서는 여러 리소스로 나누어 관리된다.

처음에는 리소스가 많아져서 오히려 복잡해 보였지만, 역할이 명확하게 나누어져 있기 때문에 구조를 이해하기는 더 쉬웠다.

특히 HealthCheckPolicy를 추가하는 과정도 인상적이었다.

단순히 외부에서 접속만 가능한 것이 아니라 Load Balancer가 Pod의 상태를 확인하면서 정상적인 Pod로만 요청을 전달하도록 구성한다는 점도 함께 이해할 수 있었다.

---

## 3. Argo Rollouts 설치

Gateway API 구성이 끝난 뒤 본격적으로 Argo Rollouts를 설치했다.

사실 이름만 들어봤지 직접 사용해 본 것은 처음이었다.

처음에는 Deployment를 완전히 대체하는 새로운 리소스라고 생각했는데, 실습을 진행하면서 보니 기존 Deployment와 사용 방법이 크게 다르지는 않았다.

YAML 구조도 상당히 비슷했고 Replica 개수나 이미지 설정도 익숙한 형태였다.

대신 Deployment에는 없던 Blue/Green이나 Canary와 같은 배포 전략을 사용할 수 있다는 점이 가장 큰 차이였다.

또한 kubectl argo rollouts 플러그인도 함께 설치했다.

기존에는 kubectl get pod 정도로만 상태를 확인했는데 Rollout 상태를 시각적으로 확인할 수 있어서 현재 어떤 단계까지 진행되고 있는지 이해하기 훨씬 쉬웠다.

이 부분은 실제 운영 환경에서도 꽤 유용하겠다는 생각이 들었다.

---

## 4. Rollout 리소스로 Blue/Green 배포 적용하기

Argo Rollouts 설치가 끝난 뒤에는 기존 Deployment를 Rollout 리소스로 변경하는 작업을 진행했다.

처음 YAML을 열어봤을 때는 Deployment와 거의 비슷하게 생겨서 "생각보다 별 차이가 없네?"라는 느낌이었다.

하지만 자세히 살펴보니 배포 전략을 지정하는 부분이 추가되어 있었고, 여기서부터 기존 Deployment와는 다른 방식으로 동작하기 시작했다.

이번 실습에서는 `strategy.blueGreen`을 사용하여 Blue/Green 배포를 구성했다.

가장 눈에 띄었던 설정은 `activeService`와 `previewService`였다.

Deployment만 사용할 때는 Service 하나만 연결하면 끝이었는데, Blue/Green에서는 Service를 두 개 사용한다는 점이 흥미로웠다.

### activeService와 previewService

처음에는 이름만 보고 두 Service의 차이를 잘 이해하지 못했다.

실습을 따라가면서 직접 배포해 보니 역할이 명확하게 구분되어 있었다.

`activeService`는 현재 실제 사용자가 접속하는 운영 서비스이다.

반면 `previewService`는 새 버전을 확인하기 위한 서비스이다.

즉, 새로운 버전이 배포되더라도 바로 운영 트래픽을 받는 것이 아니라 Preview 환경에서 먼저 실행된다.

```text
             User
              │
              ▼
      activeService
              │
           Blue(v1)

previewService
       │
       ▼
    Green(v2)
```

이 구조를 보고 나서야 왜 Service를 두 개 사용하는지 이해할 수 있었다.

Rolling Update에서는 새로운 Pod가 생성되는 순간부터 일부 사용자가 새 버전에 접속하게 되지만, Blue/Green에서는 운영 환경과 테스트 환경이 분리된다.

덕분에 운영자는 사용자에게 영향을 주지 않고 충분히 새로운 버전을 검증할 수 있다.

### autoPromotionSeconds

실습에서는 `autoPromotionSeconds` 옵션도 함께 사용했다.

처음에는 단순히 시간을 지정하는 옵션인 줄 알았는데, 실제로는 Preview 상태를 얼마나 유지할 것인지를 결정하는 설정이었다.

예를 들어 30초로 설정하면 새로운 버전이 생성된 뒤 바로 운영으로 전환되지 않는다.

먼저 Preview 상태를 유지한 뒤 일정 시간이 지나면 자동으로 운영 트래픽이 새로운 버전으로 변경된다.

배포 과정을 정리하면 다음과 같다.

```text
Blue(v1) 운영

↓

Green(v2) 생성

↓

Preview 상태

↓

자동 승격

↓

Green(v2) 운영
```

이 과정을 직접 보면서 "새 버전을 먼저 실행해 놓고 확인한 뒤 운영으로 전환하는구나."라는 흐름이 머릿속에 정리되기 시작했다.

책으로만 봤을 때보다 직접 배포 과정을 확인하는 것이 훨씬 이해하기 쉬웠다.

---

## 5. 실제 배포 과정 살펴보기

이제 새로운 이미지를 사용하도록 태그를 변경한 뒤 Rollout을 다시 적용했다.

기존에는 이미지를 변경하면 Pod가 순서대로 교체되는 모습만 봤는데, 이번에는 조금 다른 과정이 진행되었다.

먼저 새로운 ReplicaSet이 생성되고 Green 버전이 실행된다.

하지만 운영 트래픽은 여전히 기존 Blue 버전으로 전달된다.

```text
운영

activeService
      │
      ▼
 ReplicaSet(v1)

--------------------

Preview

previewService
       │
       ▼
 ReplicaSet(v2)
```

두 버전이 동시에 실행되고 있는 모습을 보니 왜 Blue/Green이라고 부르는지도 자연스럽게 이해가 되었다.

새 버전은 이미 실행되고 있지만 아직 사용자에게 공개되지 않은 상태인 것이다.

### 운영 트래픽 전환

Preview 환경에서 문제가 없다고 판단되면 운영 트래픽이 새로운 버전으로 변경된다.

처음에는 Pod를 다시 생성하거나 LoadBalancer 설정이 변경되는 줄 알았는데 그렇지 않았다.

실제로는 Service가 바라보는 대상만 변경된다.

```text
Before

activeService
      │
      ▼
ReplicaSet(v1)

↓

After

activeService
      │
      ▼
ReplicaSet(v2)
```

이 부분이 가장 신기했다.

Pod를 다시 만드는 것도 아니고 Service를 삭제하는 것도 아니다.

단순히 Service가 연결하는 ReplicaSet만 변경하면서 사용자 입장에서는 서비스가 계속 유지된다.

Blue/Green 배포가 "트래픽 전환"이라는 말을 계속 사용하는 이유를 이 부분에서 이해할 수 있었다.

### 문제가 발생하면?

새 버전에 문제가 생겼다고 가정하면 어떻게 될까?

Rolling Update였다면 다시 Deployment를 수정하고 이전 버전으로 재배포해야 한다.

하지만 Blue/Green은 이미 이전 버전이 그대로 실행되고 있기 때문에 Service만 다시 이전 ReplicaSet으로 연결하면 된다.

```text
Green(v2) 오류

↓

activeService

↓

Blue(v1)
```

복구 과정이 매우 단순하다는 점이 가장 큰 장점으로 느껴졌다.

실제 서비스에서는 장애가 발생했을 때 몇 분 차이도 큰 영향을 줄 수 있기 때문에 이런 구조가 많이 사용되는 이유를 조금은 이해할 수 있었다.

---

## 6. Rolling Update와 비교해 보기

이번 실습을 하기 전까지는 Rolling Update만으로도 충분하다고 생각했다.

하지만 Blue/Green 배포 과정을 직접 따라가 보니 두 방식의 목적 자체가 다르다는 것을 알게 되었다.

Rolling Update는 리소스를 효율적으로 사용하면서 점진적으로 업데이트하는 전략이다.

반면 Blue/Green은 운영 안정성을 가장 우선으로 생각하는 전략이다.

새로운 버전을 충분히 검증한 뒤 운영 트래픽을 전환할 수 있고, 문제가 생기면 빠르게 이전 버전으로 복구할 수도 있다.

물론 두 개의 환경을 동시에 운영해야 하므로 리소스는 더 많이 사용한다.

하지만 서비스 안정성이 중요한 환경이라면 충분히 고려할 만한 방식이라는 생각이 들었다.

---