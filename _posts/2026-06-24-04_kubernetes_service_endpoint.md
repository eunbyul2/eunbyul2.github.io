---
layout: post
title: "Kubernetes Service와 Endpoint/EndpointSlice: ClusterIP, NodePort, LoadBalancer, Headless, kube-proxy, Cilium"
date: 2026-06-24 17:00:00 +0900
categories: [Kubernetes, Network]
tags: [Kubernetes, Service, ClusterIP, NodePort, LoadBalancer, HeadlessService, Endpoint, EndpointSlice, kube-proxy, Cilium, Conntrack]
published: true
---

## 1. 이 글에서 다루는 범위

이 글은 Kubernetes 네트워크 시리즈 중 **Service와 Endpoint/EndpointSlice**를 집중적으로 다룹니다.

앞선 글에서 Pod Network, CNI, kube-proxy, CoreDNS의 큰 구조를 봤다면, 이번 글에서는 다음 질문을 끝까지 추적합니다.

```text
Pod IP는 계속 바뀌는데 애플리케이션은 어떻게 안정적으로 통신하는가?

ClusterIP는 실제 NIC에 없는 IP인데 왜 접속이 되는가?

Service는 어떻게 Backend Pod를 찾는가?

Endpoint와 EndpointSlice는 왜 필요한가?

Pod는 Running인데 왜 Endpoint에 안 들어가는가?

DNS는 되는데 Service 접속이 안 되는 이유는 무엇인가?

kube-proxy가 없으면 Cilium은 Service를 어떻게 처리하는가?
```

이 문서는 단순히 Service 타입을 외우는 용도가 아닙니다. 실제 장애 분석에서 `kubectl get svc`, `kubectl get endpoints`, `kubectl get endpointslice`, `curl`, `nslookup`, `ss`, `iptables`, `cilium service list` 결과를 어떻게 연결해서 봐야 하는지 이해하는 것을 목표로 합니다.

공식 참고자료:

- Kubernetes Service 공식 문서: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes EndpointSlice 공식 문서: https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/
- Kubernetes EndpointSlice API Reference: https://kubernetes.io/docs/reference/kubernetes-api/discovery/endpoint-slice-v1/
- Kubernetes DNS for Services and Pods: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- Kubernetes Topology Aware Routing: https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/
- Kubernetes Debugging DNS Resolution: https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/
- Kubernetes kube-proxy Reference: https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/
- Cilium kube-proxy replacement: https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/
- Cilium Load Balancing: https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#load-balancing
- MetalLB 공식 문서: https://metallb.universe.tf/

## 2. Service가 필요한 이유

### 2-1) Pod IP를 직접 바라보면 왜 안 되는가

Kubernetes에서 애플리케이션은 보통 Pod로 실행됩니다. Pod는 고유한 IP를 가집니다.

예를 들어 `backend` Deployment가 있고, 현재 Pod가 다음처럼 실행 중이라고 가정합니다.

```text
backend-7d9f8c9d6b-a1b2c  10.244.1.10
backend-7d9f8c9d6b-d3e4f  10.244.2.20
backend-7d9f8c9d6b-g5h6i  10.244.3.30
```

`frontend` 애플리케이션이 이 Pod IP를 직접 호출한다면 다음처럼 됩니다.

```text
frontend -> http://10.244.1.10:8080
```

처음에는 동작할 수 있습니다. 하지만 Kubernetes에서 Pod IP는 안정적인 식별자가 아닙니다.

Pod는 다음 상황에서 언제든 삭제되고 새로 만들어질 수 있습니다.

- Deployment Rolling Update
- Pod 장애로 인한 재시작
- Node 장애
- Drain / Cordon / Eviction
- HPA에 의한 Scale out / Scale in
- 이미지 변경
- 설정 변경
- 리소스 부족으로 인한 재스케줄링

예를 들어 Rolling Update가 발생하면 기존 Pod가 삭제되고 새 Pod가 생깁니다.

```text
기존 Pod
backend-7d9f8c9d6b-a1b2c  10.244.1.10

삭제됨

새 Pod
backend-6c8b7f5d4c-x9y8z  10.244.2.55
```

그러면 `frontend`가 계속 `10.244.1.10`을 호출하고 있었다면 장애가 발생합니다.

```text
frontend -> 10.244.1.10:8080
              ^
              이미 삭제된 Pod IP
```

즉 Pod IP는 개별 인스턴스의 주소일 뿐, 애플리케이션의 안정적인 진입점으로 사용하기 어렵습니다.

### 2-2) Service는 고정 진입점이다

Service는 변하는 Pod IP 앞에 고정된 네트워크 진입점을 제공하는 Kubernetes 리소스입니다.

```text
frontend
  |
  |  http://backend.default.svc.cluster.local:80
  v
Service backend
  |
  +-> Pod 10.244.1.10:8080
  +-> Pod 10.244.2.20:8080
  +-> Pod 10.244.3.30:8080
```

Service를 사용하면 클라이언트는 Pod IP를 알 필요가 없습니다.

클라이언트는 다음 중 하나만 알면 됩니다.

```text
Service DNS Name: backend.default.svc.cluster.local
Service Name:     backend
Service ClusterIP: 10.96.120.15
Service Port:     80
```

Pod가 바뀌어도 Service 이름과 ClusterIP는 유지됩니다.

```text
기존 Endpoint
10.244.1.10:8080
10.244.2.20:8080

Pod 재생성 후 Endpoint
10.244.2.55:8080
10.244.3.71:8080

Service 이름
backend.default.svc.cluster.local  그대로 유지
```

이것이 Service의 핵심 목적입니다.

### 2-3) Service가 해결하는 문제

Service는 다음 문제를 해결합니다.

| 문제 | Service가 제공하는 해결 방식 |
|---|---|
| Pod IP가 바뀜 | 고정 ClusterIP와 DNS 이름 제공 |
| Pod가 여러 개임 | Backend Pod 목록으로 분산 전달 |
| 클라이언트가 Pod 목록을 알기 어려움 | DNS 기반 Service Discovery 제공 |
| Pod Ready 상태를 반영해야 함 | Endpoint/EndpointSlice에 Ready Endpoint만 반영 |
| 외부에서 접근해야 함 | NodePort, LoadBalancer, Ingress와 연계 |
| Stateful Pod를 개별 식별해야 함 | Headless Service와 StatefulSet 조합 |

## 3. Service는 실제로 무엇인가

### 3-1) Service는 Pod가 아니다

Service는 컨테이너를 실행하지 않습니다.

Service를 만들었다고 해서 새로운 Pod가 생기는 것은 아닙니다. Service는 Kubernetes API 객체입니다.

```bash
kubectl get svc
```

예시:

```text
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
backend      ClusterIP   10.96.120.15    <none>        80/TCP    10m
```

여기서 `10.96.120.15`는 ClusterIP입니다. 하지만 이 IP가 특정 Pod나 특정 Node의 NIC에 붙어 있는 것은 아닙니다.

### 3-2) ClusterIP는 실제 인터페이스에 보이지 않는 VIP다

Node에서 다음 명령을 실행해도 ClusterIP는 보통 보이지 않습니다.

```bash
ip addr | grep 10.96.120.15
```

대부분 아무 결과가 나오지 않습니다.

이유는 ClusterIP가 실제 NIC에 할당된 IP가 아니라 **Virtual IP(VIP)** 이기 때문입니다.

```text
실제 IP
- Node IP
- Pod IP
- NIC에 붙은 IP

가상 IP
- Service ClusterIP
- kube-proxy / IPVS / eBPF 규칙으로 처리되는 VIP
```

즉 Service의 ClusterIP는 다음처럼 동작합니다.

```text
Client Pod
  |
  |  dst = 10.96.120.15:80
  v
Node Kernel Datapath
  |
  |  이 IP는 Service VIP다.
  |  실제 Endpoint 중 하나로 바꿔야 한다.
  v
DNAT or eBPF LB
  |
  v
Backend Pod 10.244.2.20:8080
```

따라서 Service 문제를 볼 때는 `ip addr`에서 ClusterIP가 보이지 않는다고 해서 이상한 것이 아닙니다.

중요한 것은 다음입니다.

```text
ClusterIP가 존재하는가?        kubectl get svc
ClusterIP가 Endpoint를 갖는가? kubectl get endpoints / endpointslice
Datapath가 Service VIP를 처리하는가? kube-proxy / iptables / IPVS / Cilium
```

## 4. Service 생성 과정

### 4-1) 사용자가 Service YAML을 적용한다

예시 Service입니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
```

적용합니다.

```bash
kubectl apply -f backend-service.yaml
```

### 4-2) API Server에 Service 객체가 저장된다

Kubernetes API Server는 Service 객체를 검증한 뒤 etcd에 저장합니다.

Service 객체에는 다음 정보가 들어갑니다.

```text
metadata.name: backend
metadata.namespace: default
spec.type: ClusterIP
spec.selector: app=backend
spec.ports[].port: 80
spec.ports[].targetPort: 8080
spec.clusterIP: 자동 할당된 ClusterIP
```

만약 `spec.clusterIP`를 지정하지 않으면 Kubernetes가 Service CIDR 안에서 IP를 자동 할당합니다.

예시:

```text
Service CIDR: 10.96.0.0/12
할당된 ClusterIP: 10.96.120.15
```

### 4-3) EndpointSlice Controller가 Backend Pod를 찾는다

Service에 selector가 있으면 Kubernetes는 해당 selector와 일치하는 Pod를 찾습니다.

```yaml
selector:
  app: backend
```

Pod 라벨이 다음과 같다면 매칭됩니다.

```yaml
metadata:
  labels:
    app: backend
```

Kubernetes는 매칭되는 Pod 중 실제로 트래픽을 받을 수 있는 Pod를 Endpoint로 등록합니다.

여기서 중요한 조건이 있습니다.

```text
Pod가 Running이어도 Ready가 아니면 일반적으로 Service Endpoint로 사용되지 않는다.
```

즉 Service Backend가 되려면 보통 다음 조건이 필요합니다.

```text
1. Service selector와 Pod label 일치
2. Pod IP 존재
3. Pod가 삭제 중이 아님
4. Pod Ready 상태가 True
5. targetPort와 컨테이너 listen port가 맞음
```

### 4-4) kube-proxy 또는 Cilium이 Datapath를 구성한다

Service와 EndpointSlice가 준비되면 실제 패킷을 전달할 규칙이 필요합니다.

구성에 따라 다음 중 하나가 처리합니다.

```text
kube-proxy iptables mode
kube-proxy IPVS mode
Cilium eBPF kube-proxy replacement
기타 CNI / dataplane 구현체
```

일반적인 개념 흐름은 다음과 같습니다.

```text
Service VIP: 10.96.120.15:80
Endpoint:    10.244.1.10:8080
Endpoint:    10.244.2.20:8080
Endpoint:    10.244.3.30:8080

Datapath rule:
10.96.120.15:80 로 들어온 트래픽을 Endpoint 중 하나로 전달
```

## 5. Service 타입 전체 구조

Kubernetes Service 타입은 다음과 같습니다.

| Service Type | 주요 목적 | 접근 범위 |
|---|---|---|
| ClusterIP | 클러스터 내부 고정 진입점 | Cluster 내부 |
| NodePort | Node IP와 고정 포트로 노출 | Cluster 외부에서도 NodeIP 접근 가능 시 가능 |
| LoadBalancer | 외부 Load Balancer와 연계 | 외부 |
| ExternalName | 외부 DNS 이름을 Service처럼 참조 | DNS CNAME 방식 |
| Headless Service | ClusterIP 없이 Endpoint DNS 직접 제공 | 내부 Service Discovery, StatefulSet |

주의할 점은 Service 타입이 서로 완전히 독립된 기능이 아니라는 것입니다.

예를 들어 LoadBalancer 타입 Service는 내부적으로 보통 NodePort와 ClusterIP 기능을 함께 가집니다.

```text
LoadBalancer
  -> NodePort
    -> ClusterIP
      -> Endpoint Pod
```

구현체와 설정에 따라 세부 동작은 달라질 수 있지만, 기본적으로 Kubernetes Service 타입은 계층적으로 이해하면 쉽습니다.

## 6. ClusterIP

### 6-1) ClusterIP란 무엇인가

ClusterIP는 클러스터 내부에서만 접근 가능한 Service VIP입니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

생성 후 확인:

```bash
kubectl get svc backend
```

예시:

```text
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
backend   ClusterIP   10.96.120.15    <none>        80/TCP    1m
```

클러스터 내부 Pod는 다음 방식으로 호출할 수 있습니다.

```bash
curl http://backend.default.svc.cluster.local
curl http://backend
curl http://10.96.120.15
```

### 6-2) ClusterIP 요청 흐름

Pod에서 Service를 호출하면 다음 흐름으로 이동합니다.

```text
frontend Pod
  |
  |  curl http://backend:80
  v
CoreDNS
  |
  |  backend.default.svc.cluster.local -> 10.96.120.15
  v
frontend Pod
  |
  |  dst = 10.96.120.15:80
  v
Node datapath
  |
  |  Service VIP 확인
  |  Endpoint 중 하나 선택
  v
DNAT / eBPF LB
  |
  |  dst = 10.244.2.20:8080
  v
backend Pod
```

여기서 DNS와 Service 처리는 별개입니다.

```text
DNS 역할
backend.default.svc.cluster.local -> ClusterIP 반환

Service datapath 역할
ClusterIP:Port -> PodIP:TargetPort 전달
```

따라서 다음 현상이 가능합니다.

```text
DNS는 정상이다.
nslookup backend.default.svc.cluster.local 성공

하지만 curl은 실패한다.
curl http://backend:80 실패

원인 후보
- Endpoint 없음
- targetPort 오류
- Pod가 listen하지 않음
- NetworkPolicy 차단
- kube-proxy / Cilium Service 처리 문제
```

### 6-3) ClusterIP는 언제 쓰는가

ClusterIP는 가장 기본적인 Service 타입입니다.

주로 다음 용도로 사용합니다.

- 내부 API 서버
- 백엔드 서비스
- 데이터베이스 연결용 내부 이름
- Redis, Kafka, RabbitMQ 등 내부 컴포넌트 접근
- Ingress Controller가 Backend로 라우팅할 대상
- 마이크로서비스 간 통신

예시:

```text
frontend -> backend ClusterIP
backend -> redis ClusterIP
backend -> postgres ClusterIP
Ingress Controller -> app Service ClusterIP
```

## 7. NodePort

### 7-1) NodePort란 무엇인가

NodePort는 모든 Node의 특정 포트를 열고, 그 포트로 들어온 트래픽을 Service Backend로 전달하는 방식입니다.

예시:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
spec:
  type: NodePort
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

접속 방식:

```text
http://<NodeIP>:30080
```

예시:

```bash
curl http://192.168.10.83:30080
curl http://192.168.10.84:30080
curl http://192.168.10.85:30080
```

### 7-2) NodePort 포트 범위

기본 NodePort 범위는 일반적으로 다음입니다.

```text
30000-32767
```

이 범위는 kube-apiserver의 `--service-node-port-range` 설정으로 변경할 수 있습니다.

운영 환경에서 NodePort를 직접 외부에 열어두는 것은 관리 포인트가 많습니다.

주의할 점:

- 모든 Node에서 포트가 열린 것처럼 동작한다.
- 방화벽에서 NodePort 범위를 허용해야 할 수 있다.
- 노드가 많으면 접근 경로 관리가 어려워진다.
- 외부 사용자를 직접 받는 용도보다는 LoadBalancer, Ingress Controller 앞단 연결에 자주 사용된다.

### 7-3) NodePort 패킷 흐름

외부 클라이언트가 NodePort로 접근하는 흐름입니다.

```text
External Client
  |
  |  http://192.168.10.83:30080
  v
Node 192.168.10.83
  |
  |  NodePort rule
  v
Service backend
  |
  |  Endpoint 선택
  v
Pod 10.244.2.20:8080
```

중요한 점은 요청을 받은 Node에 Backend Pod가 없어도 다른 Node의 Pod로 전달될 수 있다는 것입니다.

```text
Client -> Node1:30080
          Node1에는 backend Pod 없음
          Endpoint 선택 결과 Node2의 Pod 선택
          Node1 -> Node2 -> Pod
```

이 동작 때문에 추가 Hop이 생길 수 있습니다.

### 7-4) NodePort와 Client IP

NodePort에서는 Client IP 보존 여부가 중요합니다.

기본적으로 `externalTrafficPolicy: Cluster`일 때는 트래픽이 클러스터 내부에서 다른 Node의 Pod로 전달될 수 있습니다. 이 과정에서 SNAT가 발생해 Backend Pod 입장에서 원본 Client IP가 보이지 않을 수 있습니다.

```yaml
spec:
  externalTrafficPolicy: Cluster
```

반대로 `externalTrafficPolicy: Local`을 사용하면 해당 Node에 존재하는 Local Endpoint로만 트래픽을 보냅니다.

```yaml
spec:
  externalTrafficPolicy: Local
```

장점:

```text
원본 Client IP 보존 가능
```

단점:

```text
해당 Node에 Ready Endpoint가 없으면 트래픽 처리 불가
부하 분산이 균등하지 않을 수 있음
```

정리:

| 설정 | 동작 | 장점 | 주의점 |
|---|---|---|---|
| externalTrafficPolicy: Cluster | 모든 Endpoint로 전달 가능 | 가용성, 분산 유리 | Client IP 보존 어려울 수 있음 |
| externalTrafficPolicy: Local | Local Endpoint만 사용 | Client IP 보존 유리 | Local Pod 없는 Node는 응답 불가 |

## 8. LoadBalancer

### 8-1) LoadBalancer Service란 무엇인가

LoadBalancer 타입은 Kubernetes Service를 외부 Load Balancer와 연결하기 위한 타입입니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

클라우드 환경에서는 Service를 만들면 클라우드 컨트롤러가 외부 Load Balancer를 생성하거나 연결합니다.

예시:

```text
AWS: NLB / ELB
GCP: Cloud Load Balancing
Azure: Azure Load Balancer
```

온프레미스 환경에서는 기본 Kubernetes만으로 외부 Load Balancer가 자동 생성되지 않습니다. 이 경우 MetalLB 같은 구현체가 필요합니다.

### 8-2) LoadBalancer 요청 흐름

일반적인 흐름은 다음과 같습니다.

```text
External Client
  |
  v
External Load Balancer
  |
  v
NodePort or Direct Pod Routing
  |
  v
Service
  |
  v
Endpoint Pod
```

많은 환경에서 LoadBalancer Service는 내부적으로 NodePort를 함께 사용합니다.

```text
LoadBalancer IP: 203.0.113.10:80
  -> NodeIP:30080
    -> Service ClusterIP
      -> PodIP:8080
```

다만 구현체에 따라 NodePort를 생략하거나, eBPF datapath에서 더 직접적으로 처리할 수도 있습니다.

### 8-3) 온프레미스와 MetalLB

온프레미스 Kubernetes에서는 클라우드 사업자가 제공하는 Load Balancer가 없습니다. 그래서 `type: LoadBalancer` Service를 만들면 `EXTERNAL-IP`가 계속 `<pending>` 상태가 될 수 있습니다.

```bash
kubectl get svc
```

예시:

```text
NAME   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
web    LoadBalancer   10.96.10.20    <pending>     80:30080/TCP   5m
```

이때 MetalLB를 사용하면 온프레미스에서도 LoadBalancer IP를 할당할 수 있습니다.

MetalLB는 대표적으로 두 방식으로 동작합니다.

| 방식 | 설명 |
|---|---|
| Layer2 모드 | ARP/NDP를 사용해 특정 Node가 LoadBalancer IP를 소유한 것처럼 응답 |
| BGP 모드 | 라우터와 BGP로 경로를 광고 |

Layer2 모드 예시:

```text
Client -> 192.168.10.200
          ^ MetalLB가 LoadBalancer IP로 광고

실제로는 특정 Node가 ARP 응답
```

BGP 모드 예시:

```text
Kubernetes Node
  |
  |  BGP route advertisement
  v
Router
  |
  v
Client traffic -> 적절한 Node로 라우팅
```

온프레미스, OpenStack, 베어메탈 환경에서는 LoadBalancer Service가 자동으로 동작하는지 반드시 확인해야 합니다.

### 8-4) LoadBalancer와 Ingress의 관계

Ingress Controller를 외부로 노출할 때 자주 사용하는 구조입니다.

```text
External Load Balancer
  |
  v
Ingress Controller Service(type=LoadBalancer or NodePort)
  |
  v
Ingress Controller Pod
  |
  v
Application Service(ClusterIP)
  |
  v
Application Pod
```

즉 일반 애플리케이션마다 LoadBalancer Service를 만드는 대신, 보통은 Ingress Controller만 LoadBalancer로 노출하고 여러 애플리케이션은 Ingress Host/Path 규칙으로 라우팅합니다.

## 9. ExternalName

### 9-1) ExternalName이란 무엇인가

ExternalName Service는 Kubernetes 내부에서 외부 DNS 이름을 Service처럼 참조하게 해주는 타입입니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: default
spec:
  type: ExternalName
  externalName: db.example.com
```

이 Service는 ClusterIP를 만들지 않습니다. 대신 DNS CNAME처럼 동작합니다.

```text
external-db.default.svc.cluster.local
  -> db.example.com
```

### 9-2) ExternalName의 특징

ExternalName은 L4 프록시가 아닙니다.

즉 Kubernetes가 외부 서버로 트래픽을 중계해주는 것이 아닙니다. DNS 이름을 다른 DNS 이름으로 바꿔줄 뿐입니다.

```text
Pod
  |
  | DNS lookup external-db.default.svc.cluster.local
  v
CoreDNS
  |
  | CNAME db.example.com
  v
Pod가 db.example.com으로 직접 접속
```

따라서 다음을 주의해야 합니다.

- 외부 DNS 이름이 실제로 해석되어야 한다.
- 외부 네트워크로 나가는 egress 경로가 있어야 한다.
- NetworkPolicy, 방화벽, NAT 정책을 별도로 고려해야 한다.
- ExternalName에는 selector가 없다.
- EndpointSlice를 자동 생성하지 않는다.

### 9-3) 언제 사용하는가

ExternalName은 다음 상황에서 사용할 수 있습니다.

```text
클러스터 내부 애플리케이션이 외부 DB를 Kubernetes Service 이름처럼 사용하고 싶을 때
환경별 외부 의존성을 DNS 이름으로 추상화하고 싶을 때
외부 SaaS API endpoint를 내부 이름으로 감싸고 싶을 때
```

예시:

```text
개발 환경
external-db -> dev-db.example.com

운영 환경
external-db -> prod-db.example.com
```

## 10. Headless Service

### 10-1) Headless Service란 무엇인가

Headless Service는 ClusterIP가 없는 Service입니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  clusterIP: None
  selector:
    app: database
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

핵심은 다음입니다.

```text
일반 ClusterIP Service
DNS 조회 결과: ClusterIP 반환

Headless Service
DNS 조회 결과: Endpoint Pod IP 직접 반환
```

### 10-2) 일반 Service와 Headless Service 비교

| 구분 | 일반 ClusterIP Service | Headless Service |
|---|---|---|
| ClusterIP | 있음 | 없음, `clusterIP: None` |
| DNS 결과 | Service VIP | Endpoint Pod IP 목록 |
| 로드밸런싱 | kube-proxy/eBPF 등 Datapath가 처리 | 클라이언트가 DNS 결과 중 선택 |
| 주요 용도 | 일반적인 내부 서비스 | StatefulSet, 개별 Pod 식별, DB/메시징 시스템 |

일반 Service:

```text
backend.default.svc.cluster.local
  -> 10.96.120.15
```

Headless Service:

```text
database.default.svc.cluster.local
  -> 10.244.1.10
  -> 10.244.2.20
  -> 10.244.3.30
```

### 10-3) StatefulSet과 Headless Service

Headless Service는 StatefulSet과 함께 자주 사용됩니다.

StatefulSet은 Pod 이름이 안정적입니다.

```text
postgres-0
postgres-1
postgres-2
```

Headless Service를 사용하면 각 Pod에 안정적인 DNS 이름을 부여할 수 있습니다.

```text
postgres-0.postgres.default.svc.cluster.local
postgres-1.postgres.default.svc.cluster.local
postgres-2.postgres.default.svc.cluster.local
```

이 구조는 다음 시스템에서 중요합니다.

- PostgreSQL HA
- MongoDB ReplicaSet
- Kafka
- RabbitMQ
- Elasticsearch
- ZooKeeper
- etcd
- Ceph 관련 구성요소

왜냐하면 이런 시스템들은 단순히 아무 Pod로 요청을 보내는 것보다, 각 멤버를 개별적으로 식별해야 하는 경우가 많기 때문입니다.

```text
Kafka broker-0
Kafka broker-1
Kafka broker-2

각 broker는 고유한 identity와 주소가 필요
```

### 10-4) SRV Record

Kubernetes DNS는 Service와 named port에 대해 SRV record도 제공할 수 있습니다.

예를 들어 Service에 port name이 `grpc`이고 protocol이 TCP라면 다음과 같은 형식으로 조회할 수 있습니다.

```text
_grpc._tcp.my-service.default.svc.cluster.local
```

SRV record는 서비스의 포트 정보까지 DNS로 조회할 수 있게 해줍니다.

실무에서 직접 자주 쓰는 편은 아니지만, Stateful 서비스나 특정 클라이언트 라이브러리에서 사용될 수 있으므로 개념은 알아두는 것이 좋습니다.

## 11. Port, TargetPort, NodePort

### 11-1) port

`port`는 Service가 노출하는 포트입니다.

```yaml
ports:
  - port: 80
```

클라이언트는 Service에 대해 이 포트로 접근합니다.

```bash
curl http://backend:80
```

### 11-2) targetPort

`targetPort`는 실제 Backend Pod의 컨테이너가 listen하고 있는 포트입니다.

```yaml
ports:
  - port: 80
    targetPort: 8080
```

흐름:

```text
Service:80 -> Pod:8080
```

가장 흔한 장애 중 하나가 `targetPort` 불일치입니다.

예를 들어 Pod 안의 애플리케이션은 8000 포트에서 listen하고 있는데 Service targetPort가 8080이면 연결이 실패합니다.

```text
Service targetPort: 8080
Pod listen port:    8000

결과:
Connection refused 또는 timeout
```

확인:

```bash
kubectl exec -it <pod> -- ss -lntp
kubectl get pod <pod> -o yaml | grep -A10 ports
kubectl describe svc backend
```

### 11-3) named targetPort

`targetPort`는 숫자가 아니라 이름으로 지정할 수도 있습니다.

Pod:

```yaml
ports:
  - name: http
    containerPort: 8080
```

Service:

```yaml
ports:
  - port: 80
    targetPort: http
```

이 방식은 컨테이너 포트 번호가 변경될 가능성이 있을 때 유용합니다.

하지만 이름이 일치하지 않으면 Endpoint Port 해석에 문제가 생길 수 있으므로 반드시 확인해야 합니다.

### 11-4) nodePort

`nodePort`는 NodePort 또는 LoadBalancer 타입에서 Node가 노출하는 포트입니다.

```yaml
ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

흐름:

```text
NodeIP:30080 -> Service:80 -> Pod:8080
```

정리:

| 필드 | 의미 | 예시 |
|---|---|---|
| port | Service가 노출하는 포트 | Service:80 |
| targetPort | Pod 컨테이너가 실제로 listen하는 포트 | Pod:8080 |
| nodePort | Node가 외부로 여는 포트 | NodeIP:30080 |

## 12. Selector

### 12-1) Selector의 역할

Selector는 Service가 어떤 Pod를 Backend로 삼을지 결정하는 조건입니다.

Service:

```yaml
spec:
  selector:
    app: backend
    tier: api
```

Pod:

```yaml
metadata:
  labels:
    app: backend
    tier: api
```

두 조건이 모두 맞으면 해당 Pod는 Service의 Backend 후보가 됩니다.

### 12-2) Selector가 틀리면 어떤 일이 생기는가

Service는 생성되지만 Endpoint가 비어 있습니다.

```bash
kubectl get svc backend
```

```text
NAME      TYPE        CLUSTER-IP      PORT(S)
backend   ClusterIP   10.96.120.15    80/TCP
```

Service는 존재합니다.

하지만 Endpoint를 보면 다음과 같습니다.

```bash
kubectl get endpoints backend
```

```text
NAME      ENDPOINTS   AGE
backend   <none>      5m
```

EndpointSlice도 비어 있거나 Ready Endpoint가 없습니다.

```bash
kubectl get endpointslice -l kubernetes.io/service-name=backend
```

이 상태에서 Ingress가 이 Service로 라우팅하면 보통 503이 발생합니다.

```text
Ingress -> Service backend -> Endpoint 없음 -> 503 Service Unavailable
```

### 12-3) Selector 확인 방법

```bash
kubectl get svc backend -o yaml
kubectl get svc backend -o jsonpath='{.spec.selector}'
kubectl get pod --show-labels
kubectl get pod -l app=backend,tier=api -o wide
```

Service selector와 Pod label은 장애 분석에서 가장 먼저 확인해야 합니다.

## 13. Readiness와 Endpoint의 관계

### 13-1) Running과 Ready는 다르다

Pod 상태에서 `Running`은 컨테이너 프로세스가 실행 중이라는 의미에 가깝습니다. 하지만 이것이 애플리케이션이 요청을 받을 준비가 되었다는 뜻은 아닙니다.

예를 들어 Spring Boot, FastAPI, Node.js 애플리케이션은 프로세스가 떠도 다음 작업이 끝나기 전까지 정상 요청을 처리하지 못할 수 있습니다.

- DB 연결 초기화
- Migration 수행
- Cache warm-up
- 외부 API 연결 확인
- 모델 로딩
- 설정 파일 로딩

그래서 Kubernetes는 Readiness Probe를 사용합니다.

### 13-2) Readiness Probe가 실패하면 Endpoint에서 제외된다

예시:

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

Pod가 Running이어도 Readiness Probe가 실패하면 해당 Pod는 Service Endpoint로 사용되지 않습니다.

```text
Pod Running=True
Pod Ready=False

결과:
Service Endpoint에 포함되지 않음
```

확인:

```bash
kubectl get pod -o wide
kubectl describe pod <pod>
kubectl get endpoints backend
kubectl get endpointslice -l kubernetes.io/service-name=backend -o yaml
```

### 13-3) 실무에서 자주 나는 문제

#### 문제 1. Pod는 Running인데 Service 503

가능 원인:

```text
Readiness Probe 실패
Service selector 불일치
targetPort 불일치
Pod label 누락
```

확인:

```bash
kubectl get pod -o wide
kubectl describe pod <pod>
kubectl get endpoints backend
```

#### 문제 2. 배포 직후 잠깐 503

가능 원인:

```text
새 Pod가 Ready 되기 전에 기존 Pod가 먼저 종료됨
readinessProbe / rollingUpdate 설정 부적절
```

확인:

```bash
kubectl describe deploy backend
kubectl rollout status deploy/backend
kubectl get pod -w
```

Deployment에서는 다음 설정도 함께 봐야 합니다.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

## 14. Endpoint

### 14-1) Endpoint란 무엇인가

Endpoint는 Service가 실제로 트래픽을 보낼 수 있는 Backend 주소입니다.

```text
Service backend
  ClusterIP: 10.96.120.15
  Port: 80

Endpoint
  10.244.1.10:8080
  10.244.2.20:8080
  10.244.3.30:8080
```

Service는 안정적인 진입점이고, Endpoint는 실제 목적지 목록입니다.

```text
Client -> Service -> Endpoint -> Pod
```

### 14-2) Endpoints 리소스 확인

```bash
kubectl get endpoints backend
```

예시:

```text
NAME      ENDPOINTS                                            AGE
backend   10.244.1.10:8080,10.244.2.20:8080,10.244.3.30:8080   10m
```

상세 확인:

```bash
kubectl describe endpoints backend
```

### 14-3) Endpoints API의 한계

기존 Endpoints 리소스는 하나의 객체에 모든 Endpoint 주소를 저장합니다.

Pod 수가 적을 때는 괜찮지만, 수백~수천 개 Endpoint를 가진 Service에서는 문제가 생길 수 있습니다.

문제:

```text
Endpoints 객체가 너무 커짐
작은 변경에도 큰 객체 전체가 업데이트됨
API Server와 watch client 부하 증가
확장성 저하
```

이 문제를 해결하기 위해 EndpointSlice가 도입되었습니다.

## 15. EndpointSlice

### 15-1) EndpointSlice란 무엇인가

EndpointSlice는 Service Backend 정보를 여러 조각으로 나누어 저장하는 리소스입니다.

```text
Service backend
  |
  +-> EndpointSlice backend-abcde
  |     - 10.244.1.10:8080
  |     - 10.244.2.20:8080
  |
  +-> EndpointSlice backend-fghij
        - 10.244.3.30:8080
        - 10.244.4.40:8080
```

Kubernetes 공식 문서 기준으로 EndpointSlice API는 Service가 많은 Backend를 효율적으로 다루기 위한 메커니즘입니다.

### 15-2) EndpointSlice 확인

```bash
kubectl get endpointslice -A
```

Service별 확인:

```bash
kubectl get endpointslice -l kubernetes.io/service-name=backend
```

YAML 확인:

```bash
kubectl get endpointslice -l kubernetes.io/service-name=backend -o yaml
```

예시 구조:

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  labels:
    kubernetes.io/service-name: backend
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 8080
endpoints:
  - addresses:
      - 10.244.1.10
    conditions:
      ready: true
      serving: true
      terminating: false
    nodeName: node1
    zone: ap-northeast-2a
```

### 15-3) EndpointSlice 주요 필드

| 필드 | 의미 |
|---|---|
| addressType | IPv4, IPv6, FQDN 등 주소 유형 |
| ports | Endpoint가 제공하는 포트 목록 |
| endpoints[].addresses | 실제 Backend IP |
| endpoints[].conditions.ready | Service 트래픽 대상으로 사용할 준비 여부 |
| endpoints[].conditions.serving | 트래픽 처리 가능 여부 |
| endpoints[].conditions.terminating | 종료 중인지 여부 |
| endpoints[].nodeName | Endpoint Pod가 위치한 Node |
| endpoints[].zone | Endpoint가 속한 Zone |

### 15-4) EndpointSlice와 Readiness

EndpointSlice에는 Endpoint condition이 들어갑니다.

```yaml
conditions:
  ready: true
  serving: true
  terminating: false
```

장애 분석 시 이 값을 보면 Pod가 Endpoint로 들어갔는지, 종료 중인지, 트래픽을 받을 수 있는지 판단할 수 있습니다.

예를 들어 `ready: false`이면 Service가 해당 Endpoint로 일반 트래픽을 보내지 않을 수 있습니다.

### 15-5) EndpointSlice와 Dual Stack

IPv4/IPv6 Dual Stack 환경에서는 주소 유형별로 EndpointSlice가 나뉠 수 있습니다.

```text
IPv4 EndpointSlice
IPv6 EndpointSlice
```

Kubernetes 문서에서도 EndpointSlice는 하나의 IP family에 대한 Endpoint를 표현한다고 설명합니다.

운영 환경에서 Dual Stack을 사용한다면 Service의 `ipFamilyPolicy`, `ipFamilies`, EndpointSlice의 `addressType`을 함께 봐야 합니다.

## 16. Endpoint Controller와 EndpointSlice Controller

### 16-1) Controller가 하는 일

Kubernetes는 선언형 시스템입니다.

사용자가 Service를 만들면, 컨트롤러가 현재 상태를 관찰하고 원하는 상태에 맞게 EndpointSlice를 관리합니다.

```text
Service 생성
  |
  v
Selector 확인
  |
  v
Matching Pod 조회
  |
  v
Pod Ready 상태 확인
  |
  v
EndpointSlice 생성/수정/삭제
```

### 16-2) Pod 변경 시 EndpointSlice 갱신

다음 상황에서 EndpointSlice가 변경됩니다.

- Pod 생성
- Pod 삭제
- Pod IP 변경
- Pod label 변경
- Pod Readiness 변경
- Service selector 변경
- Service port 변경

예를 들어 Pod가 Ready 상태가 되면 EndpointSlice에 추가됩니다.

```text
Pod Ready=False
EndpointSlice ready=false or 제외

Pod Ready=True
EndpointSlice ready=true
```

### 16-3) Endpoint Mirroring

Endpoint Mirroring은 기존 Endpoints API와 EndpointSlice API 사이의 호환성을 위한 개념입니다.

과거에는 많은 컴포넌트가 Endpoints 리소스를 직접 사용했습니다. 하지만 확장성 문제로 EndpointSlice가 도입되면서, Kubernetes는 기존 API와 새 API 사이의 호환성을 유지해야 했습니다.

실무적으로는 다음처럼 이해하면 됩니다.

```text
최신 Kubernetes 컴포넌트: EndpointSlice 중심
오래된 도구나 일부 스크립트: Endpoints를 볼 수 있음

트러블슈팅할 때는 둘 다 확인하는 것이 안전
```

```bash
kubectl get endpoints backend
kubectl get endpointslice -l kubernetes.io/service-name=backend
```

## 17. Service Discovery

### 17-1) DNS 기반 Service Discovery

Kubernetes에서 가장 일반적인 Service Discovery 방식은 DNS입니다.

Service가 생성되면 CoreDNS가 Kubernetes API를 보고 DNS 레코드를 제공합니다.

일반 형식:

```text
<service-name>.<namespace>.svc.cluster.local
```

예시:

```text
backend.default.svc.cluster.local
postgres.database.svc.cluster.local
redis.cache.svc.cluster.local
```

Pod 내부에서 다음처럼 접근할 수 있습니다.

```bash
curl http://backend.default.svc.cluster.local
```

같은 namespace라면 짧게도 가능합니다.

```bash
curl http://backend
```

### 17-2) 왜 짧은 이름이 동작하는가

Pod 내부의 `/etc/resolv.conf`를 보면 search domain이 있습니다.

```bash
kubectl exec -it <pod> -- cat /etc/resolv.conf
```

예시:

```text
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

`backend`를 조회하면 DNS resolver는 search domain을 붙여 순서대로 질의할 수 있습니다.

```text
backend.default.svc.cluster.local
backend.svc.cluster.local
backend.cluster.local
backend
```

그래서 같은 namespace에서는 `backend`만 써도 Service를 찾을 수 있습니다.

다른 namespace의 Service는 보통 다음처럼 호출합니다.

```bash
curl http://backend.api.svc.cluster.local
curl http://backend.api
```

### 17-3) 환경변수 기반 Service Discovery

Kubernetes는 Pod 생성 시점에 존재하던 Service 정보를 환경변수로 주입할 수 있습니다.

확인:

```bash
kubectl exec -it <pod> -- env | grep SERVICE
```

예시:

```text
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
BACKEND_SERVICE_HOST=10.96.120.15
BACKEND_SERVICE_PORT=80
```

하지만 환경변수 방식은 다음 한계가 있습니다.

```text
Pod 생성 이후 만들어진 Service는 기존 Pod 환경변수에 자동 반영되지 않음
Service가 많으면 환경변수가 많아짐
동적 서비스 발견에는 DNS가 더 적합
```

그래서 실무에서는 DNS 기반 Service Discovery를 더 일반적으로 사용합니다.

## 18. Session Affinity

### 18-1) Session Affinity란 무엇인가

Session Affinity는 같은 클라이언트의 요청을 가능하면 같은 Backend Pod로 보내는 기능입니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  sessionAffinity: ClientIP
  ports:
    - port: 80
      targetPort: 8080
```

기본값은 `None`입니다.

```yaml
sessionAffinity: None
```

`ClientIP`를 설정하면 클라이언트 IP 기준으로 동일 Backend에 붙도록 시도합니다.

### 18-2) Session Affinity가 필요한 경우

예를 들어 애플리케이션이 세션 상태를 Pod 메모리에 저장한다고 가정합니다.

```text
첫 요청 -> Pod A -> 로그인 세션 생성
두 번째 요청 -> Pod B -> 세션 없음 -> 로그아웃처럼 보임
```

이런 경우 같은 클라이언트를 같은 Pod로 보내고 싶을 수 있습니다.

하지만 근본적으로는 세션을 Pod 로컬 메모리에만 저장하지 않는 설계가 더 좋습니다.

권장 구조:

```text
Session Store
- Redis
- Database
- External Cache
```

### 18-3) ClientIP 기준의 한계

Session Affinity는 L4 수준의 Client IP 기반입니다.

Ingress, HAProxy, NAT, Load Balancer 뒤에서는 Backend Pod가 보는 Client IP가 실제 사용자 IP가 아닐 수 있습니다.

```text
User A
User B
User C
  |
  v
Load Balancer IP: 10.0.0.5
  |
  v
Service
```

Backend 기준 Client IP가 모두 Load Balancer IP로 보이면 Session Affinity가 기대한 대로 동작하지 않을 수 있습니다.

HTTP Cookie 기반 Sticky Session이 필요하면 Ingress Controller, HAProxy, 애플리케이션 레벨에서 처리해야 하는 경우가 많습니다.

## 19. Conntrack과 Service

### 19-1) Conntrack이란 무엇인가

Conntrack은 Linux 커널의 connection tracking 기능입니다. NAT나 방화벽이 연결 상태를 추적할 때 사용합니다.

Kubernetes Service에서 conntrack은 중요합니다.

예를 들어 ClusterIP가 Pod IP로 DNAT되면 커널은 연결 상태를 기억해야 응답 패킷도 올바르게 되돌릴 수 있습니다.

```text
Client Pod 10.244.1.10:52344
  -> Service 10.96.120.15:80

DNAT 후
Client Pod 10.244.1.10:52344
  -> Backend Pod 10.244.2.20:8080

응답 시
Backend Pod 10.244.2.20:8080
  -> Client Pod 10.244.1.10:52344

Client는 원래 Service 10.96.120.15와 통신한다고 생각해야 함
```

이 매핑을 추적하는 데 conntrack이 관여합니다.

### 19-2) conntrack 확인

노드에서 확인할 수 있습니다.

```bash
sudo conntrack -L
sudo conntrack -L -p tcp
sudo conntrack -S
```

conntrack 도구가 없다면 설치가 필요할 수 있습니다.

```bash
sudo apt-get install conntrack
```

### 19-3) conntrack 문제 증상

conntrack table이 가득 차거나 오래된 NAT 정보가 남아 있으면 이상한 네트워크 장애가 발생할 수 있습니다.

증상 예시:

```text
일부 요청만 실패
Pod 재배포 후에도 예전 Endpoint로 가는 것처럼 보임
간헐적인 connection timeout
Service 접속 불안정
```

확인:

```bash
sudo sysctl net.netfilter.nf_conntrack_max
sudo conntrack -S
sudo dmesg | grep -i conntrack
```

주의:

운영 환경에서 conntrack entry를 무작정 삭제하면 기존 연결이 끊길 수 있습니다. 반드시 영향 범위를 보고 수행해야 합니다.

## 20. kube-proxy Service 처리

### 20-1) kube-proxy의 역할

kube-proxy는 Kubernetes Service VIP를 실제 Endpoint로 전달하기 위한 노드별 네트워크 컴포넌트입니다.

각 Node에서 kube-proxy가 실행되며, API Server를 watch하면서 Service와 EndpointSlice 변경을 반영합니다.

```text
API Server
  |
  | watch Service / EndpointSlice
  v
kube-proxy on each Node
  |
  | program iptables or IPVS
  v
Node kernel datapath
```

### 20-2) iptables mode

iptables mode에서는 kube-proxy가 iptables NAT 규칙을 생성합니다.

개념 흐름:

```text
dst = ClusterIP:Port
  |
  v
iptables KUBE-SERVICES chain
  |
  v
Endpoint 선택
  |
  v
DNAT to PodIP:TargetPort
```

확인 명령:

```bash
sudo iptables-save | grep KUBE-SERVICES
sudo iptables-save | grep backend
sudo iptables-save | grep KUBE-SEP
```

iptables mode는 단순하고 널리 사용되지만, Service/Endpoint가 매우 많을 때 규칙 수가 많아져 성능과 관리 측면의 부담이 생길 수 있습니다.

### 20-3) IPVS mode

IPVS mode는 Linux IP Virtual Server 기능을 사용합니다.

개념적으로 Service VIP를 virtual server로 두고, Endpoint Pod를 real server로 등록합니다.

```text
Virtual Server
10.96.120.15:80

Real Server
10.244.1.10:8080
10.244.2.20:8080
10.244.3.30:8080
```

확인:

```bash
sudo ipvsadm -Ln
```

IPVS는 로드밸런싱 기능이 명확하고 대규모 Service에서 장점이 있을 수 있습니다.

### 20-4) kube-proxy 장애 시 증상

kube-proxy가 정상 동작하지 않으면 다음 증상이 나타날 수 있습니다.

```text
Pod IP로 직접 접근은 됨
Service ClusterIP로 접근은 안 됨
NodePort 접속 실패
일부 Node에서만 Service 접속 실패
```

확인:

```bash
kubectl -n kube-system get pod -l k8s-app=kube-proxy -o wide
kubectl -n kube-system logs <kube-proxy-pod>
kubectl -n kube-system get cm kube-proxy -o yaml
```

다만 Cilium kube-proxy replacement 환경에서는 kube-proxy Pod가 없을 수 있습니다. 이 경우 kube-proxy가 없다고 해서 장애가 아닙니다.

## 21. Cilium eBPF Service 처리

### 21-1) kube-proxy가 없는데 Service가 되는 이유

Cilium을 kube-proxy replacement 모드로 사용하면 kube-proxy 없이 eBPF로 Service 처리를 할 수 있습니다.

즉 다음 구조입니다.

```text
기존 kube-proxy 환경
Service VIP -> iptables/IPVS -> Endpoint

Cilium kube-proxy replacement 환경
Service VIP -> eBPF Load Balancer -> Endpoint
```

Cilium은 Linux 커널의 eBPF 기능을 사용해 패킷 처리 경로에 로드밸런싱 로직을 붙입니다.

### 21-2) Cilium Service 확인

Cilium CLI가 있다면 다음 명령으로 Service 상태를 볼 수 있습니다.

```bash
cilium status
cilium service list
cilium bpf lb list
```

Pod 내부나 노드에서 Hubble을 사용하는 경우 흐름을 볼 수도 있습니다.

```bash
hubble observe --to-service default/backend
```

환경에 따라 명령어 지원 여부는 다를 수 있습니다.

### 21-3) Maglev

Cilium은 Service Load Balancing에서 Maglev consistent hashing을 사용할 수 있습니다.

Maglev는 Google에서 제안한 consistent hashing 기반 로드밸런싱 방식으로, Backend 변화가 있을 때 전체 연결 매핑이 크게 흔들리지 않게 하는 데 목적이 있습니다.

개념적으로는 다음과 같습니다.

```text
Client flow A -> Backend 1
Client flow B -> Backend 2
Client flow C -> Backend 3

Backend 2 제거
일부 flow만 재배치
```

실무적으로 중요한 점은 다음입니다.

```text
Service Endpoint 변화 시 연결 분산 안정성
Session Affinity와는 다른 개념
L4 Load Balancing 알고리즘
```

### 21-4) DSR

DSR은 Direct Server Return의 약자입니다.

일반적인 로드밸런싱:

```text
Client -> Load Balancer -> Backend
Client <- Load Balancer <- Backend
```

DSR:

```text
Client -> Load Balancer -> Backend
Client <- Backend
```

응답 경로가 Load Balancer를 다시 거치지 않기 때문에 성능상 이점이 있을 수 있습니다. 하지만 네트워크 설계와 라우팅 조건이 맞아야 하므로 운영 난도가 올라갑니다.

Cilium 환경에서 DSR, SNAT, hybrid mode 같은 설정은 Service 동작과 원본 IP 보존, 리턴 패스에 영향을 줄 수 있으므로 장애 분석 시 Cilium 설정을 반드시 확인해야 합니다.

## 22. External Endpoint

### 22-1) Selector 없는 Service

Service는 반드시 Pod selector를 가져야 하는 것은 아닙니다.

Selector 없는 Service를 만들 수 있습니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

이 경우 Kubernetes는 자동으로 EndpointSlice를 만들 수 없습니다. 어떤 Backend를 사용할지 selector로 알 수 없기 때문입니다.

따라서 EndpointSlice를 수동으로 만들어야 합니다.

### 22-2) 외부 서버를 Service Backend로 등록

예시:

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-db-1
  labels:
    kubernetes.io/service-name: external-db
addressType: IPv4
ports:
  - name: postgres
    protocol: TCP
    port: 5432
endpoints:
  - addresses:
      - 192.168.10.50
```

이렇게 하면 Kubernetes 내부에서는 다음 이름으로 접근할 수 있습니다.

```bash
psql -h external-db.default.svc.cluster.local -p 5432
```

실제 목적지는 외부 서버입니다.

```text
Pod -> Service external-db -> Endpoint 192.168.10.50:5432
```

### 22-3) External Endpoint 주의점

External Endpoint는 편리하지만 다음을 주의해야 합니다.

```text
Kubernetes가 외부 서버의 Health를 Pod처럼 자동 관리하지 않음
외부 서버 장애 시 Endpoint가 자동 제거되지 않음
NetworkPolicy 적용 범위가 CNI 구현에 따라 다를 수 있음
외부 IP 라우팅과 방화벽을 별도로 확인해야 함
DNS만 된다고 연결이 되는 것은 아님
```

## 23. InternalTrafficPolicy와 ExternalTrafficPolicy

### 23-1) internalTrafficPolicy

`internalTrafficPolicy`는 클러스터 내부에서 Service로 들어온 트래픽을 어떻게 처리할지 제어합니다.

예시:

```yaml
spec:
  internalTrafficPolicy: Cluster
```

또는

```yaml
spec:
  internalTrafficPolicy: Local
```

| 값 | 의미 |
|---|---|
| Cluster | 클러스터 전체 Endpoint로 전달 가능 |
| Local | 같은 Node의 Local Endpoint로만 전달 |

`Local`을 사용하면 같은 Node에 있는 Pod로만 트래픽을 보내므로 cross-node 트래픽을 줄일 수 있습니다. 하지만 Local Endpoint가 없으면 요청이 실패할 수 있습니다.

### 23-2) externalTrafficPolicy

`externalTrafficPolicy`는 외부에서 들어온 트래픽을 어떻게 처리할지 제어합니다.

```yaml
spec:
  externalTrafficPolicy: Local
```

주요 목적은 원본 Client IP 보존입니다.

하지만 Local Endpoint가 없는 Node로 트래픽이 들어오면 처리되지 않을 수 있으므로 Load Balancer Health Check와 함께 설계해야 합니다.

## 24. Topology Aware Routing

### 24-1) 왜 필요한가

클러스터가 여러 Zone에 걸쳐 있을 때 모든 요청이 무작위로 다른 Zone의 Pod로 가면 비용과 지연이 증가할 수 있습니다.

예시:

```text
Zone A frontend -> Zone B backend
```

같은 Zone에 backend가 있다면 굳이 Zone B로 보낼 필요가 없습니다.

Topology Aware Routing은 가능한 경우 트래픽을 같은 Zone 안의 Endpoint로 보내도록 힌트를 제공합니다.

```text
Zone A Client -> Zone A Endpoint 우선
Zone B Client -> Zone B Endpoint 우선
```

### 24-2) EndpointSlice와 Zone 정보

EndpointSlice에는 Endpoint가 위치한 zone 정보가 포함될 수 있습니다.

```yaml
endpoints:
  - addresses:
      - 10.244.1.10
    zone: ap-northeast-2a
```

kube-proxy 또는 dataplane 구현체는 이 정보를 활용해 topology-aware routing을 적용할 수 있습니다.

### 24-3) 주의점

Topology Aware Routing은 항상 강제로 같은 Zone만 사용하는 기능으로 이해하면 안 됩니다.

Endpoint 분포, 노드 수, 트래픽 정책, 구현체 지원 여부에 따라 동작이 달라질 수 있습니다.

장점:

```text
Zone 간 트래픽 감소
지연 시간 개선 가능
비용 절감 가능
```

주의점:

```text
Zone별 Pod 수 불균형이면 부하 불균형 발생 가능
모든 환경에서 동일하게 동작한다고 단정하면 안 됨
```

## 25. Hairpin Traffic

### 25-1) Hairpin이란 무엇인가

Hairpin Traffic은 트래픽이 나갔다가 다시 같은 곳으로 돌아오는 형태를 말합니다.

Kubernetes Service에서 흔한 예시는 다음입니다.

```text
Pod A
  |
  |  curl http://my-service
  v
Service my-service
  |
  |  Endpoint 선택 결과
  v
Pod A 자신
```

즉 Pod가 자기 자신이 포함된 Service를 호출했는데, 로드밸런싱 결과 자기 자신으로 다시 돌아오는 상황입니다.

### 25-2) 왜 문제가 될 수 있는가

Hairpin은 NAT, conntrack, bridge 설정과 관련이 있습니다.

구현체가 hairpin traffic을 제대로 처리하지 못하면 다음 문제가 발생할 수 있습니다.

```text
Pod가 자기 Service를 호출할 때만 실패
다른 Pod에서 호출하면 성공
간헐적인 connection reset 또는 timeout
```

현대 Kubernetes CNI에서는 대부분 처리되지만, 특수한 네트워크 설정이나 커스텀 bridge, 구버전 구성에서는 확인이 필요합니다.

## 26. Service와 NetworkPolicy

### 26-1) Service가 허용한다고 NetworkPolicy가 허용하는 것은 아니다

Service는 Backend Pod로 가는 주소 추상화입니다.

하지만 실제 패킷이 Pod로 들어갈 때 CNI의 NetworkPolicy가 적용될 수 있습니다.

즉 다음이 가능합니다.

```text
DNS 정상
Service 정상
Endpoint 정상
하지만 NetworkPolicy 때문에 Pod로 들어가는 트래픽 차단
```

### 26-2) 확인 방법

```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy <name>
```

테스트:

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
curl -v http://backend.default.svc.cluster.local
```

Cilium 환경이라면 다음도 확인할 수 있습니다.

```bash
cilium policy get
hubble observe --verdict DROPPED
```

환경에 따라 Cilium CLI와 Hubble 설치 여부는 다릅니다.

## 27. Service와 Ingress의 관계

Ingress는 직접 Pod로 가지 않습니다. 일반적으로 Ingress Controller는 Service를 통해 Backend Pod로 트래픽을 보냅니다.

```text
User
  |
  v
Load Balancer
  |
  v
Ingress Controller
  |
  v
Service backend
  |
  v
Endpoint Pod
```

Ingress YAML 예시:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
```

이때 Ingress에서 `service.name`과 `service.port.number`가 맞아야 합니다.

Service는 존재하지만 Endpoint가 없으면 Ingress Controller는 Backend로 보낼 대상이 없으므로 503이 발생할 수 있습니다.

```text
Ingress rule 정상
Service 정상
Endpoint 없음
결과: 503
```

반대로 Endpoint는 있는데 애플리케이션 path가 없으면 404가 날 수 있습니다.

```text
Ingress path 정상
Service 정상
Endpoint 정상
Backend app에 해당 route 없음
결과: 404
```

## 28. 실전 장애 사례

### 28-1) 503 Service Unavailable

증상:

```text
Ingress 접속 시 503
curl Service 시 실패
```

가능 원인:

```text
Service Endpoint 없음
Pod Ready=False
Selector와 Pod label 불일치
targetPort 오류
EndpointSlice Controller 문제
```

확인:

```bash
kubectl get svc backend
kubectl get endpoints backend
kubectl get endpointslice -l kubernetes.io/service-name=backend
kubectl get pod -l app=backend -o wide
kubectl describe pod <pod>
```

판단:

```text
endpoints <none> 이면 Service 문제가 아니라 Backend 선택 문제부터 확인
```

### 28-2) DNS는 되는데 curl이 안 됨

증상:

```bash
nslookup backend.default.svc.cluster.local
# 성공

curl http://backend.default.svc.cluster.local
# 실패
```

해석:

DNS는 Service 이름을 ClusterIP로 바꾸는 데 성공했습니다. 따라서 CoreDNS만의 문제는 아닐 가능성이 큽니다.

다음 후보를 봅니다.

```text
Endpoint 없음
targetPort 오류
Pod가 listen하지 않음
NetworkPolicy 차단
kube-proxy/Cilium Service datapath 문제
```

확인:

```bash
kubectl get endpoints backend
kubectl describe svc backend
kubectl exec -it <backend-pod> -- ss -lntp
kubectl exec -it <test-pod> -- curl -v http://<pod-ip>:8080
kubectl exec -it <test-pod> -- curl -v http://<cluster-ip>:80
```

### 28-3) Pod IP로는 되는데 Service로는 안 됨

증상:

```bash
curl http://10.244.2.20:8080
# 성공

curl http://backend:80
# 실패
```

가능 원인:

```text
Service selector 오류
EndpointSlice 없음
kube-proxy iptables/IPVS 문제
Cilium eBPF LB 문제
ClusterIP datapath 문제
```

확인:

```bash
kubectl get svc backend -o yaml
kubectl get endpoints backend
kubectl get endpointslice -l kubernetes.io/service-name=backend
sudo iptables-save | grep KUBE-SERVICES
sudo ipvsadm -Ln
cilium service list
```

Cilium 환경에서 kube-proxy가 없으면 iptables KUBE-SERVICES가 없을 수 있습니다. 이 경우 `cilium service list`, `cilium bpf lb list` 쪽을 봐야 합니다.

### 28-4) Service는 되는데 Ingress만 안 됨

증상:

```bash
kubectl exec -it test -- curl http://backend:80
# 성공

외부에서 https://app.example.com 접속
# 실패
```

가능 원인:

```text
Ingress host/path rule 오류
Ingress Controller 설정 오류
TLS/SNI 문제
Ingress Controller에서 Service port 참조 오류
X-Forwarded-* 또는 backend protocol 오류
```

이 경우 Service보다는 Ingress 문서에서 이어서 봐야 합니다.

### 28-5) NodePort가 일부 Node에서만 안 됨

가능 원인:

```text
externalTrafficPolicy: Local 설정
해당 Node에 Local Endpoint 없음
방화벽에서 NodePort 차단
kube-proxy/Cilium이 특정 Node에서 비정상
```

확인:

```bash
kubectl get svc backend -o yaml | grep externalTrafficPolicy
kubectl get pod -l app=backend -o wide
kubectl get endpointslice -l kubernetes.io/service-name=backend -o wide
kubectl -n kube-system get pod -o wide | grep kube-proxy
cilium status
```

### 28-6) LoadBalancer EXTERNAL-IP가 pending

가능 원인:

```text
클라우드 컨트롤러 없음
온프레미스에서 MetalLB 미설치
MetalLB IPAddressPool 미설정
L2/BGP 광고 실패
```

확인:

```bash
kubectl get svc
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

### 28-7) 원본 Client IP가 안 보임

가능 원인:

```text
externalTrafficPolicy: Cluster
SNAT 발생
Ingress/Proxy가 X-Forwarded-For를 덮어씀
애플리케이션이 X-Forwarded-For를 신뢰하도록 설정되지 않음
```

확인:

```bash
kubectl get svc ingress-nginx-controller -o yaml | grep externalTrafficPolicy
kubectl logs <backend-pod>
curl -H 'X-Forwarded-For: 1.2.3.4' ...
```

## 29. Service 트러블슈팅 절차

Service 문제는 다음 순서로 좁혀가면 됩니다.

```text
1. Service가 존재하는가?
2. Service type, ClusterIP, port, targetPort가 맞는가?
3. Selector가 Pod label과 일치하는가?
4. Endpoint/EndpointSlice가 존재하는가?
5. Endpoint condition ready가 true인가?
6. Pod가 실제 targetPort에서 listen 중인가?
7. Pod IP로 직접 접근은 되는가?
8. ClusterIP로 접근은 되는가?
9. DNS 이름으로 접근은 되는가?
10. NodePort/LoadBalancer라면 외부 경로와 trafficPolicy가 맞는가?
11. NetworkPolicy가 막고 있지 않은가?
12. kube-proxy 또는 Cilium Service datapath가 정상인가?
13. conntrack 문제가 아닌가?
```

### 29-1) 기본 확인 명령어

```bash
kubectl get svc -A
kubectl describe svc backend
kubectl get svc backend -o yaml
```

### 29-2) Selector와 Pod label 확인

```bash
kubectl get svc backend -o jsonpath='{.spec.selector}'
kubectl get pod --show-labels
kubectl get pod -l app=backend -o wide
```

### 29-3) Endpoint 확인

```bash
kubectl get endpoints backend
kubectl describe endpoints backend
kubectl get endpointslice -l kubernetes.io/service-name=backend
kubectl get endpointslice -l kubernetes.io/service-name=backend -o yaml
```

### 29-4) Pod Ready 상태 확인

```bash
kubectl get pod -l app=backend
kubectl describe pod <pod>
kubectl get pod <pod> -o jsonpath='{.status.conditions}'
```

### 29-5) 실제 listen port 확인

```bash
kubectl exec -it <backend-pod> -- ss -lntp
kubectl exec -it <backend-pod> -- curl -v http://127.0.0.1:8080/health
```

### 29-6) Pod IP 직접 접근 테스트

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
curl -v http://10.244.2.20:8080/health
```

### 29-7) ClusterIP 접근 테스트

```bash
curl -v http://10.96.120.15:80/health
```

### 29-8) DNS 접근 테스트

```bash
nslookup backend.default.svc.cluster.local
curl -v http://backend.default.svc.cluster.local:80/health
```

### 29-9) kube-proxy 확인

kube-proxy 사용 환경:

```bash
kubectl -n kube-system get pod -l k8s-app=kube-proxy -o wide
kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=100
sudo iptables-save | grep KUBE-SERVICES
sudo ipvsadm -Ln
```

### 29-10) Cilium 확인

Cilium kube-proxy replacement 환경:

```bash
cilium status
cilium service list
cilium bpf lb list
cilium endpoint list
hubble observe --to-service default/backend
```

명령어는 설치 방식과 권한에 따라 다를 수 있습니다.

## 30. Service 장애를 해석하는 기준

### 30-1) DNS 실패와 Service 실패를 분리한다

```bash
nslookup backend.default.svc.cluster.local
```

실패하면 DNS/CoreDNS부터 봅니다.

성공하면 다음으로 ClusterIP 접근을 봅니다.

```bash
curl -v http://<cluster-ip>:<port>
```

### 30-2) Service 실패와 Pod 실패를 분리한다

Pod IP 직접 접근이 되는지 확인합니다.

```bash
curl -v http://<pod-ip>:<targetPort>
```

판단:

```text
Pod IP 직접 접근 성공 + Service 접근 실패
=> Service selector, Endpoint, kube-proxy/Cilium datapath 의심

Pod IP 직접 접근 실패
=> 애플리케이션 listen, Pod 방화벽, NetworkPolicy, CNI Pod routing 의심
```

### 30-3) Endpoint가 없으면 Service는 할 일이 없다

Service 자체는 Backend를 자동으로 만들어주지 않습니다.

Endpoint가 없으면 Service는 트래픽을 보낼 목적지가 없습니다.

```text
Service 있음
ClusterIP 있음
DNS 있음
Endpoint 없음

결과:
대부분 503 또는 connection 실패
```

따라서 Service 장애에서 가장 중요한 명령은 다음입니다.

```bash
kubectl get endpoints <service>
kubectl get endpointslice -l kubernetes.io/service-name=<service>
```

## 31. 예제: 정상 Service 구성

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: nginx:1.27
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
```

Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
```

확인:

```bash
kubectl apply -f backend.yaml
kubectl get pod -l app=backend -o wide
kubectl get svc backend
kubectl get endpoints backend
kubectl get endpointslice -l kubernetes.io/service-name=backend
```

테스트:

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
curl -v http://backend.default.svc.cluster.local
```

## 32. 예제: Headless Service와 StatefulSet

Headless Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

StatefulSet 일부:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - name: postgres
              containerPort: 5432
```

DNS 이름:

```text
postgres-0.postgres.default.svc.cluster.local
postgres-1.postgres.default.svc.cluster.local
postgres-2.postgres.default.svc.cluster.local
```

이 구조는 각 Pod를 개별 멤버로 식별해야 하는 Stateful 시스템에서 사용합니다.

## 33. 최종 정리

Service를 이해할 때 핵심은 다음입니다.

```text
Service는 고정 진입점이다.
Endpoint/EndpointSlice는 실제 Backend 목록이다.
ClusterIP는 실제 NIC IP가 아니라 VIP다.
DNS는 Service 이름을 ClusterIP 또는 Endpoint로 바꿔준다.
Readiness가 실패하면 Pod는 일반적으로 Endpoint에서 제외된다.
NodePort는 NodeIP:Port를 Service로 연결한다.
LoadBalancer는 외부 LB와 Service를 연결한다.
Headless Service는 ClusterIP 없이 Pod IP를 직접 DNS로 노출한다.
kube-proxy 또는 Cilium eBPF가 Service VIP를 실제 Pod IP로 전달한다.
Service 장애는 DNS, Endpoint, Pod listen, NetworkPolicy, Datapath를 분리해서 봐야 한다.
```

가장 중요한 장애 분석 공식은 다음입니다.

```text
DNS 이름
  -> ClusterIP
    -> Service Port
      -> EndpointSlice
        -> Pod IP
          -> targetPort
            -> 애플리케이션 프로세스
```

이 흐름 중 어느 한 단계라도 틀리면 Service는 정상적으로 동작하지 않습니다.

