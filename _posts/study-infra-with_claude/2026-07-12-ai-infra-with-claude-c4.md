---
layout: post
title: "[AI 인프라 모각코] 4장: 관측 가능성 한 번에 구축하기"
date: 2026-07-13 10:00:00 +0900
categories: [Kubernetes, Observability]
tags: [Prometheus, Grafana, Loki, Fluent Bit, Alertmanager]
published: true
---

# Chapter 4. Observability 환경 구축하기

3장에서는 GitOps 기반으로 Kubernetes 애플리케이션을 배포하는 환경을 만들었다.

하지만 애플리케이션을 배포했다고 해서 운영이 끝나는 것은 아니다. 실제 운영에서는 서비스가 정상적으로 동작하는지 계속 확인해야 하고, 문제가 발생했을 때 빠르게 원인을 찾아야 한다.

이번 장에서는 Kubernetes에서 가장 많이 사용하는 Observability 스택인 **Prometheus, Grafana, Loki, Fluent Bit, Alertmanager**를 구축하면서 메트릭, 로그, 알림을 구성하는 방법을 학습했다.

---

# Prometheus와 Grafana

가장 먼저 메트릭 수집 환경을 구축했다.

Prometheus는 Kubernetes 클러스터의 메트릭을 일정 주기마다 수집하여 저장하는 역할을 한다.

CPU 사용률, Memory 사용량, Pod 재시작 횟수, Node 상태 같은 숫자 데이터를 지속적으로 저장하기 때문에 현재 시스템 상태를 빠르게 확인할 수 있다.

다만 Prometheus 자체는 데이터를 저장하는 역할에 집중되어 있기 때문에 시각화 기능은 부족하다.

그래서 Grafana를 함께 사용한다.

Grafana는 Prometheus에서 수집한 메트릭을 Dashboard 형태로 보여주기 때문에 운영자가 현재 클러스터 상태를 훨씬 쉽게 파악할 수 있다.

이번 장에서는 kube-prometheus-stack Helm Chart를 사용해서 Prometheus와 Grafana, Alertmanager까지 함께 설치했다.

Helm Chart 하나로 대부분의 구성이 끝나기 때문에 Kubernetes에서는 사실상 표준 설치 방법처럼 사용되고 있다는 점도 알게 되었다.

---

# 메트릭만으로는 원인을 찾을 수 없다

Prometheus를 설치하고 나니 CPU나 Memory 사용량은 쉽게 확인할 수 있었다.

하지만 메트릭만으로는 "무슨 문제가 발생했는지"까지만 알 수 있을 뿐, "왜 문제가 발생했는지"는 알 수 없었다.

예를 들어 CPU가 90%까지 올라갔다고 하더라도 어떤 에러가 발생했는지, 어떤 요청 때문에 그런 것인지는 메트릭만으로는 확인하기 어렵다.

결국 원인을 분석하려면 로그가 필요하다.

그래서 이번에는 로그 수집 환경도 함께 구축했다.

---

# Loki와 Fluent Bit

로그 수집 도구를 선택하면서 ELK Stack, Google Cloud Logging, Datadog, Loki 등을 비교해보았다.

ELK Stack은 기능이 매우 강력하지만 Elasticsearch가 사용하는 메모리 요구사항이 상당히 크다.

현재 실습 환경은 리소스가 많지 않았기 때문에 조금 부담스러운 선택이었다.

반면 Loki는 Elasticsearch처럼 로그 전체를 인덱싱하지 않고 Kubernetes 라벨을 기준으로 관리한다.

덕분에 메모리 사용량이 훨씬 적고 Grafana와도 자연스럽게 연동된다.

학습 환경에서는 Loki가 더 적합하다고 판단하여 Loki와 Fluent Bit 조합을 선택했다.

Fluent Bit는 모든 Node에서 실행되면서 컨테이너 로그를 수집한다.

처음에는 왜 Deployment가 아니라 DaemonSet으로 배포하는지 궁금했는데, 로그는 모든 Node에서 발생하기 때문에 Node마다 하나씩 실행되는 DaemonSet이 가장 적합하다는 것을 이해하게 되었다.

수집된 로그는 Loki에 저장되고 Grafana에서 LogQL을 이용해 바로 검색할 수 있다.

---

# Alertmanager와 PrometheusRule

관측 환경에서 마지막으로 필요한 것은 알림이다.

Dashboard를 계속 보고 있을 수는 없기 때문에 문제가 발생하면 자동으로 알려주는 기능이 필요하다.

이번 장에서는 PrometheusRule을 이용해 알림 규칙을 만들고 Alertmanager가 실제 알림을 전달하는 구조를 구성했다.

예를 들어 Pod 재시작 횟수가 일정 기준을 넘거나 CPU 사용률이 계속 높게 유지되면 Alertmanager가 Slack이나 Email 같은 채널로 알림을 보낼 수 있다.

Grafana에서도 Alert를 만들 수 있지만 GitOps 환경에서는 YAML로 관리할 수 있는 PrometheusRule 방식이 훨씬 관리하기 편하다는 점도 알게 되었다.

---

# Claude Code Memory

이번 장에서 조금 흥미로웠던 내용은 Claude Code의 Memory 기능이었다.

처음에는 CLAUDE.md만 사용하는 줄 알았는데 작업하면서 생성되는 TODO나 작업 패턴도 별도로 기억할 수 있다는 점이 인상적이었다.

다만 모든 내용을 Memory에 저장하는 것이 아니라 작업 컨텍스트만 저장하는 용도로 사용하는 것이 적절해 보였다.

프로젝트 설계나 아키텍처 결정처럼 팀과 공유해야 하는 내용은 Git으로 관리하고, 개인 작업 흐름은 Memory를 활용하는 방식이 더 자연스럽다고 느꼈다.

---

# 정리

이번 장에서는 단순히 Prometheus를 설치하는 것이 아니라 Kubernetes에서 사용하는 Observability 스택 전체를 구성해보았다.

메트릭은 Prometheus, 시각화는 Grafana, 로그는 Loki와 Fluent Bit, 알림은 Alertmanager와 PrometheusRule이 각각 담당한다.

각 도구의 역할이 명확하게 나누어져 있기 때문에 서로 조합해서 하나의 관측 환경을 만드는 구조라는 점을 이해하는 것이 가장 중요했다.

특히 단순히 설치 명령을 따라 하기보다 "왜 이 도구를 선택했는가"를 계속 생각하면서 진행하니 전체 구조가 훨씬 잘 이해되었다.

앞으로 Kubernetes를 운영하게 된다면 가장 먼저 구축해야 하는 환경 중 하나가 바로 Observability라는 것을 다시 한번 느낀 장이었다.