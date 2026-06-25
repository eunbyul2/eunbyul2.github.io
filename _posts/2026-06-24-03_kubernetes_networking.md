---
layout: post
title: "Kubernetes 네트워크 완전 정리: Pod, CNI, kube-proxy, CoreDNS, eBPF, NetworkPolicy"
date: 2026-06-24 17:00:00 +0900
categories: [Kubernetes, Network]
tags: [Kubernetes, Network, CNI, Cilium, kube-proxy, CoreDNS, eBPF, NetworkPolicy, PodNetwork]
published: true
---

## 1. 이 글에서 다루는 범위

이 글은 Kubernetes 네트워크를 처음부터 끝까지 이해하기 위한 기본 문서입니다.

앞선 글에서 OSI, TCP/IP, IP, CIDR, Gateway, Routing, ARP, NAT, conntrack 같은 일반 네트워크 개념을 정리했다면, 이번 글에서는 그 개념들이 Kubernetes 안에서 어떻게 사용되는지 설명합니다.

Kubernetes 네트워크를 이해할 때 가장 중요한 질문은 다음과 같습니다.

```text
Pod는 어떻게 IP를 받는가?

Pod 안의 eth0은 실제로 어디에 연결되어 있는가?

Pod가 다른 Pod와 통신할 때 패킷은 어디를 지나가는가?

ClusterIP는 실제 서버에 붙어 있는 IP가 아닌데 왜 통신되는가?

kube-proxy는 무엇을 하는가?

Cilium은 왜 kube-proxy 없이 Service를 처리할 수 있는가?

CoreDNS는 Service 이름을 어떻게 IP로 바꾸는가?

NetworkPolicy를 만들었는데 왜 통신이 막히거나 안 막히는가?

MTU가 잘못되면 왜 ping은 되는데 HTTP는 안 될 수 있는가?
```

이 질문들에 답할 수 있어야 Kubernetes Service, EndpointSlice, Ingress, LoadBalancer, HAProxy, Cilium, Observability, 장애 분석까지 이어서 이해할 수 있습니다.

공식 참고자료:

- Kubernetes Cluster Networking: https://kubernetes.io/docs/concepts/cluster-administration/networking/
- Kubernetes Service: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes DNS for Services and Pods: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- Kubernetes Network Policies: https://kubernetes.io/docs/concepts/services-networking/network-policies/
- Kubernetes Network Plugins: https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/
- Kubernetes kube-proxy: https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/
- Kubernetes Service API Reference: https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/
- Kubernetes NetworkPolicy API Reference: https://kubernetes.io/docs/reference/kubernetes-api/networking-resources/network-policy-v1/
- CNI Specification: https://github.com/containernetworking/cni
- CoreDNS Kubernetes Plugin: https://coredns.io/plugins/kubernetes/
- Cilium Kubernetes Networking: https://docs.cilium.io/en/stable/network/kubernetes/
- Cilium kube-proxy Replacement: https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/
- Cilium eBPF Datapath: https://docs.cilium.io/en/stable/network/ebpf/
- Linux Network Namespaces: https://man7.org/linux/man-pages/man7/network_namespaces.7.html
- Linux veth: https://man7.org/linux/man-pages/man4/veth.4.html
- Linux ip-route: https://man7.org/linux/man-pages/man8/ip-route.8.html
- Linux ip-netns: https://man7.org/linux/man-pages/man8/ip-netns.8.html

## 2. Kubernetes 네트워크가 어려운 이유

일반 서버 환경에서는 네트워크 구조가 비교적 단순합니다.

```text
Client
  |
  | 192.168.10.50 -> 192.168.10.10:80
  v
Server
```

서버에 IP가 있고, 서버 안의 프로세스가 포트를 열고, 클라이언트는 해당 IP와 Port로 접속합니다.

예를 들어 NGINX가 `192.168.10.10:80`에서 대기 중이면 클라이언트는 다음처럼 접근합니다.

```bash
curl http://192.168.10.10:80
```

하지만 Kubernetes에서는 구조가 훨씬 복잡해집니다.

```text
Node A
  Node IP: 192.168.10.83
  Pod A:   10.244.1.10
  Pod B:   10.244.1.11

Node B
  Node IP: 192.168.10.84
  Pod C:   10.244.2.20
  Pod D:   10.244.2.21
```

여기서 문제는 다음과 같습니다.

```text
Pod A가 Pod C로 통신하려면 어떻게 가야 하는가?

Pod는 계속 생성/삭제되는데 IP가 바뀌면 클라이언트는 어떻게 찾는가?

Pod가 죽고 새 Pod가 생겼을 때 Service는 어떻게 새 Pod로 트래픽을 보낼 수 있는가?

Node 밖에서 ClusterIP로 접근할 수 없는 이유는 무엇인가?

Pod 안에서 backend.default.svc.cluster.local을 입력하면 왜 IP로 바뀌는가?
```

Kubernetes 네트워크는 결국 Linux 네트워크 기능을 조합해서 이 문제를 해결합니다.

핵심 구성 요소는 다음과 같습니다.

| 구성 요소 | 역할 |
|---|---|
| Network Namespace | Pod마다 독립적인 네트워크 공간 제공 |
| Pause Container | Pod의 네트워크 네임스페이스를 소유하는 기반 컨테이너 |
| veth pair | Pod 네임스페이스와 Node 네임스페이스를 연결 |
| Linux Bridge 또는 CNI datapath | 같은 노드 안의 Pod 트래픽 전달 |
| Routing Table | 다른 노드의 Pod CIDR로 가는 경로 결정 |
| Overlay 또는 Native Routing | 노드 간 Pod 통신 구현 |
| CNI | Pod 네트워크 생성/삭제를 담당하는 플러그인 표준 |
| kube-proxy 또는 eBPF | Service IP를 실제 Pod IP로 전달 |
| CoreDNS | Service 이름을 ClusterIP 또는 Pod IP로 해석 |
| NetworkPolicy | Pod 간 L3/L4 통신 허용/차단 |
| conntrack | NAT 및 연결 상태 추적 |
| MTU | 패킷 크기 제한 및 단편화/드롭 문제와 관련 |

## 3. Kubernetes 네트워크 모델

Kubernetes 공식 네트워크 모델은 단순한 원칙을 갖습니다.

```text
1. 모든 Pod는 고유한 IP를 가진다.
2. Pod는 NAT 없이 다른 Node의 Pod와 통신할 수 있어야 한다.
3. Node는 NAT 없이 Pod와 통신할 수 있어야 한다.
4. Pod 내부의 컨테이너들은 localhost를 공유한다.
```

이 모델은 “Pod를 하나의 작은 서버처럼 취급한다”는 방향에 가깝습니다.

일반 Docker 단독 실행 환경에서는 컨테이너가 기본적으로 bridge network 뒤에 있고, 외부에서 접근하려면 포트 매핑이 필요합니다.

```bash
docker run -p 8080:80 nginx
```

그러나 Kubernetes는 Pod마다 고유 IP를 부여하고, 클러스터 내부에서는 Pod IP로 직접 통신할 수 있게 설계합니다.

예시:

```text
frontend Pod: 10.244.1.10
backend Pod:  10.244.2.20
```

frontend Pod는 backend Pod로 직접 요청할 수 있습니다.

```bash
curl http://10.244.2.20:8080
```

물론 실무에서는 Pod IP가 계속 바뀌므로 직접 Pod IP를 사용하는 방식은 권장되지 않습니다. 그래서 Service와 DNS를 사용합니다.

```bash
curl http://backend.default.svc.cluster.local:8080
```

하지만 Service와 DNS를 이해하려면 먼저 Pod IP가 어떻게 만들어지고, Pod 네트워크가 어떻게 연결되는지 알아야 합니다.

## 4. Cluster Network의 구성

Kubernetes Cluster Network는 하나의 대역만 의미하지 않습니다. 보통 다음 네트워크들을 함께 봐야 합니다.

| 네트워크 | 설명 | 예시 |
|---|---|---|
| Node Network | Kubernetes Node들이 실제로 붙어 있는 물리/가상 네트워크 | 192.168.10.0/24 |
| Pod Network | Pod IP가 할당되는 대역 | 10.244.0.0/16 |
| Service Network | ClusterIP가 할당되는 가상 IP 대역 | 10.96.0.0/12 |
| Control Plane Network | kube-apiserver, etcd, kubelet, controller-manager 통신 | 보통 Node Network 사용 |
| External Network | 인터넷 또는 외부 사용자망 | Public IP, 사내망, LB망 |

예시 구조:

```text
Node Network:    192.168.10.0/24
Pod CIDR:        10.244.0.0/16
Service CIDR:    10.96.0.0/12
DNS Service IP:  10.96.0.10
```

중요한 점은 이 대역들이 서로 겹치면 안 된다는 것입니다.

잘못된 예:

```text
Node Network: 10.0.0.0/16
Pod CIDR:     10.0.0.0/16
```

이렇게 겹치면 Linux 라우팅 테이블이 목적지를 판단할 수 없습니다. `10.0.1.20`이 실제 Node인지 Pod인지 모호해지고, 잘못된 인터페이스로 패킷이 나갈 수 있습니다.

Service CIDR도 Node Network 또는 Pod CIDR과 겹치면 안 됩니다.

```text
Node Network: 192.168.10.0/24
Pod CIDR:     10.244.0.0/16
Service CIDR: 10.96.0.0/12
```

이런 식으로 서로 분리해야 합니다.

## 5. Pod는 작은 Linux 서버처럼 동작한다

Pod를 처음 보면 컨테이너 묶음처럼 보이지만, 네트워크 관점에서는 “고유 IP를 가진 작은 Linux 서버”에 가깝습니다.

Pod 안에서 다음 명령어를 실행해보면 일반 Linux 서버와 비슷한 정보가 보입니다.

```bash
kubectl exec -it <pod-name> -- ip addr
kubectl exec -it <pod-name> -- ip route
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
kubectl exec -it <pod-name> -- ss -lntp
```

예시:

```text
2: eth0@if123: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450
    inet 10.244.1.10/24 scope global eth0

default via 10.244.1.1 dev eth0
10.244.1.0/24 dev eth0 proto kernel scope link src 10.244.1.10
```

이 의미는 다음과 같습니다.

| 항목 | 의미 |
|---|---|
| eth0 | Pod 내부의 네트워크 인터페이스 |
| 10.244.1.10 | Pod IP |
| mtu 1450 | Pod 인터페이스가 한 번에 보낼 수 있는 최대 패킷 크기 |
| default via 10.244.1.1 | Pod 밖으로 나갈 때 사용하는 gateway |
| /etc/resolv.conf | DNS 서버와 search domain 정보 |

즉 Pod 안에서도 일반 Linux처럼 IP, route, DNS, socket이 존재합니다.

다만 이 eth0은 물리 NIC가 아닙니다. CNI가 만들어준 가상 인터페이스입니다.

## 6. Network Namespace

Linux Network Namespace는 네트워크 리소스를 분리하는 기능입니다.

분리되는 대표 리소스는 다음과 같습니다.

| 리소스 | 설명 |
|---|---|
| Network Interface | eth0, lo 같은 인터페이스 |
| IP Address | 네임스페이스별 IP |
| Routing Table | 네임스페이스별 라우팅 테이블 |
| ARP/Neighbor Table | 네임스페이스별 ARP 캐시 |
| iptables/nftables | 네임스페이스별 방화벽/NAT 규칙 |
| Socket | 네임스페이스별 listen/connect 상태 |

즉 Network Namespace가 다르면 같은 이름의 `eth0`이 있어도 서로 다른 인터페이스입니다.

Kubernetes Pod는 보통 Pod마다 하나의 Network Namespace를 가집니다.

```text
Node Network Namespace
  - ens3: 192.168.10.83
  - cni0
  - vethxxxx

Pod A Network Namespace
  - lo
  - eth0: 10.244.1.10

Pod B Network Namespace
  - lo
  - eth0: 10.244.1.11
```

Node에서 네임스페이스를 확인할 때는 다음 명령어를 사용할 수 있습니다.

```bash
lsns -t net
```

컨테이너 런타임에 따라 다음처럼 확인할 수도 있습니다.

```bash
crictl ps
crictl inspect <container-id>
```

주의할 점은 Kubernetes Pod의 네트워크 네임스페이스는 일반적으로 사람이 직접 관리하지 않습니다. kubelet과 container runtime, CNI plugin이 생성하고 삭제합니다.

## 7. Pause Container

Pod 안에 컨테이너가 여러 개 있을 수 있습니다.

예를 들어 다음 Pod를 생각해보겠습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
    - name: app
      image: nginx
    - name: sidecar
      image: busybox
      command: ["sleep", "3600"]
```

이 Pod에는 `app` 컨테이너와 `sidecar` 컨테이너가 있습니다. 그런데 두 컨테이너는 같은 Pod IP를 공유합니다.

```text
app container:     localhost 사용 가능
sidecar container: localhost 사용 가능
Pod IP:            하나만 존재
```

어떻게 가능한가?

Kubernetes는 Pod마다 네트워크 네임스페이스를 만들고, 그 네임스페이스를 유지하기 위해 pause container라는 아주 작은 컨테이너를 사용합니다.

구조는 다음과 같습니다.

```text
Pod
  ├─ pause container  -> Pod network namespace 소유
  ├─ app container    -> pause의 network namespace 공유
  └─ sidecar          -> pause의 network namespace 공유
```

그래서 같은 Pod 안의 컨테이너들은 다음을 공유합니다.

| 공유 대상 | 설명 |
|---|---|
| IP Address | Pod IP 하나를 공유 |
| Port Space | 같은 포트를 동시에 listen할 수 없음 |
| localhost | 서로 `localhost:<port>`로 접근 가능 |
| Routing Table | 같은 route 사용 |
| DNS 설정 | 같은 `/etc/resolv.conf` 사용 |

예를 들어 app 컨테이너가 `127.0.0.1:8080`에서 listen하고 있으면, sidecar 컨테이너는 `curl http://localhost:8080`으로 접근할 수 있습니다.

반대로 같은 Pod 안에서 두 컨테이너가 모두 `0.0.0.0:8080`을 열려고 하면 포트 충돌이 발생합니다.

## 8. veth pair

Pod의 `eth0`은 실제 물리 NIC가 아닙니다.

Pod 네트워크는 보통 veth pair로 Node와 연결됩니다.

veth pair는 양쪽 끝이 연결된 가상 케이블입니다.

```text
Pod Network Namespace              Node Network Namespace

eth0  ---------------------------  vethxxxx
10.244.1.10                         cni0 또는 datapath에 연결
```

Pod에서 패킷이 eth0으로 나가면 Node 쪽 veth peer로 나타납니다.

반대로 Node 쪽 veth peer로 들어간 패킷은 Pod 안의 eth0으로 들어갑니다.

확인 명령어:

```bash
ip link
```

Pod 안에서는 다음처럼 보일 수 있습니다.

```text
2: eth0@if123: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450
```

여기서 `@if123`은 Node 쪽 peer interface index를 의미합니다.

Node에서 해당 peer를 찾으면 `vethxxxx@if2` 같은 형태를 볼 수 있습니다.

veth pair를 이해하면 Pod 패킷 흐름이 훨씬 명확해집니다.

```text
Pod process
  |
  | socket write()
  v
Pod eth0
  |
  | veth pair
  v
Node veth peer
  |
  v
Bridge / Routing / eBPF datapath
```

## 9. Linux Bridge

많은 CNI는 같은 노드 안의 Pod들을 Linux Bridge에 연결합니다.

Bridge는 L2 Switch처럼 동작하는 가상 장치입니다.

예시:

```text
Node

        cni0 bridge
       /    |     \
  vethA  vethB  vethC
    |      |      |
  Pod A  Pod B  Pod C
```

Pod A가 같은 노드의 Pod B로 통신하는 흐름은 다음과 같습니다.

```text
Pod A eth0
  ↓
vethA
  ↓
cni0 bridge
  ↓
vethB
  ↓
Pod B eth0
```

이 경우 같은 노드 안에서는 물리 NIC를 거치지 않습니다.

확인 명령어:

```bash
ip link show type bridge
bridge link
```

일부 CNI는 `cni0`, `flannel.1`, `cilium_host`, `cilium_net` 등 다른 인터페이스 이름을 사용합니다.

Cilium eBPF datapath를 사용하는 경우 전통적인 Linux bridge 중심 구조와 다를 수 있습니다. 그래도 중요한 원리는 같습니다.

```text
Pod의 eth0은 Node의 datapath와 연결되어 있고,
CNI가 Pod 밖으로 나가는 경로를 만든다.
```

## 10. CNI란 무엇인가

CNI는 Container Network Interface의 약자입니다.

Kubernetes가 직접 Pod 네트워크를 구현하지 않고, CNI 플러그인에게 위임하기 위한 표준입니다.

Pod가 생성될 때 대략 다음 일이 발생합니다.

```text
1. kubelet이 container runtime에게 Pod sandbox 생성을 요청한다.
2. container runtime이 Pod network namespace를 만든다.
3. container runtime이 CNI plugin을 호출한다.
4. CNI plugin이 Pod에 eth0을 만들고 IP를 할당한다.
5. CNI plugin이 Node 쪽 veth, bridge, route, eBPF map 등을 구성한다.
6. Pod 컨테이너가 같은 network namespace 안에서 실행된다.
```

CNI가 담당하는 대표 작업은 다음과 같습니다.

| 작업 | 설명 |
|---|---|
| Interface 생성 | Pod eth0 및 Node 쪽 veth peer 생성 |
| IPAM | Pod IP 할당 |
| Routing | Pod CIDR 경로 설정 |
| Encapsulation | VXLAN/Geneve 등 overlay 구성 |
| Native Routing | 실제 라우팅 기반 Pod 통신 구성 |
| Policy | NetworkPolicy 적용 |
| Service Handling | 일부 CNI는 kube-proxy 기능까지 대체 |
| Observability | Flow log, drop reason, policy verdict 제공 |

대표 CNI:

| CNI | 특징 |
|---|---|
| Cilium | eBPF 기반, kube-proxy replacement, NetworkPolicy, Hubble 관측성 |
| Calico | BGP, IPIP/VXLAN, NetworkPolicy 강점 |
| Flannel | 비교적 단순한 Pod overlay 네트워크 |
| Weave Net | Overlay 기반 Pod 네트워크 |
| Antrea | Open vSwitch 기반 |
| AWS VPC CNI | Pod에 VPC IP를 직접 할당하는 구조 |
| Azure CNI | Azure VNet과 통합 |
| GKE Dataplane V2 | Google 관리형 eBPF 기반 dataplane |

중요한 점은 Kubernetes 네트워크의 실제 구현은 CNI마다 다를 수 있다는 것입니다.

따라서 장애 분석 시에는 반드시 현재 클러스터의 CNI를 확인해야 합니다.

```bash
kubectl -n kube-system get pods -o wide
kubectl -n kube-system get ds
kubectl -n kube-system get cm
```

Cilium 사용 환경에서는 다음도 확인합니다.

```bash
cilium status
cilium config view
kubectl -n kube-system get cm cilium-config -o yaml
```

## 11. Pod IP 할당과 IPAM

IPAM은 IP Address Management의 약자입니다.

Pod가 생성될 때 CNI는 Pod에 IP를 할당해야 합니다.

예를 들어 클러스터 Pod CIDR이 `10.244.0.0/16`이고, Node별 Pod CIDR이 다음처럼 나뉘었다고 가정합니다.

```text
node1: 10.244.1.0/24
node2: 10.244.2.0/24
node3: 10.244.3.0/24
```

node1에 생성되는 Pod는 보통 `10.244.1.0/24` 안에서 IP를 받습니다.

```text
pod-a: 10.244.1.10
pod-b: 10.244.1.11
```

node2에 생성되는 Pod는 `10.244.2.0/24` 안에서 IP를 받습니다.

```text
pod-c: 10.244.2.20
pod-d: 10.244.2.21
```

확인 명령어:

```bash
kubectl get pods -A -o wide
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.podCIDR}{"\n"}{end}'
```

주의할 점:

```text
Pod IP는 영구 주소가 아니다.
```

Pod가 재생성되면 IP가 바뀔 수 있습니다.

그래서 애플리케이션끼리 통신할 때는 Pod IP를 직접 쓰지 않고 Service DNS를 사용합니다.

```bash
curl http://backend.default.svc.cluster.local:8080
```

## 12. 같은 노드의 Pod-to-Pod 통신

같은 Node 안의 Pod끼리 통신하는 경우를 먼저 보겠습니다.

```text
Node A

Pod A: 10.244.1.10
Pod B: 10.244.1.11
```

Pod A가 Pod B로 요청합니다.

```bash
curl http://10.244.1.11:8080
```

흐름은 대략 다음과 같습니다.

```text
1. Pod A 애플리케이션이 socket을 통해 10.244.1.11:8080으로 데이터를 보낸다.
2. Pod A의 routing table이 목적지 10.244.1.11이 eth0으로 나가야 한다고 판단한다.
3. 패킷이 Pod A eth0으로 나간다.
4. veth pair를 통해 Node namespace의 vethA로 나온다.
5. Linux Bridge 또는 CNI datapath가 목적지 Pod B 방향으로 전달한다.
6. 패킷이 vethB를 통해 Pod B eth0으로 들어간다.
7. Pod B의 애플리케이션이 8080 포트에서 요청을 받는다.
```

그림:

```text
Pod A
10.244.1.10
  |
  | eth0
  v
vethA
  |
  v
cni0 / CNI datapath
  |
  v
vethB
  |
  | eth0
  v
Pod B
10.244.1.11
```

이때 물리 NIC를 거치지 않을 수 있습니다. 같은 노드 안에서는 Linux 내부에서 전달되기 때문입니다.

확인 명령어:

```bash
kubectl get pod -o wide
kubectl exec -it pod-a -- ip route
kubectl exec -it pod-a -- ping 10.244.1.11
```

## 13. 다른 노드의 Pod-to-Pod 통신

다른 Node에 있는 Pod끼리 통신하는 경우가 더 중요합니다.

```text
Node A
  Node IP: 192.168.10.83
  Pod A:   10.244.1.10

Node B
  Node IP: 192.168.10.84
  Pod C:   10.244.2.20
```

Pod A가 Pod C로 요청합니다.

```bash
curl http://10.244.2.20:8080
```

이때 Node A는 다음을 알아야 합니다.

```text
10.244.2.20은 Node B 쪽 Pod CIDR에 있다.
따라서 Node B로 보내야 한다.
```

구현 방식은 크게 두 가지입니다.

| 방식 | 설명 |
|---|---|
| Overlay Network | Pod 패킷을 Node IP 패킷 안에 다시 포장해서 전송 |
| Native Routing / Direct Routing | 실제 라우팅 테이블로 Pod CIDR 경로를 구성 |

## 14. Overlay Network

Overlay Network는 실제 물리/가상 네트워크 위에 또 다른 가상 네트워크를 얹는 방식입니다.

대표 기술은 VXLAN과 Geneve입니다.

예시:

```text
원래 Pod 패킷
Source: 10.244.1.10
Destination: 10.244.2.20
```

Node A는 이 Pod 패킷을 그대로 물리 네트워크에 흘려보내지 않고, Node B로 보내기 위한 외부 패킷으로 감쌉니다.

```text
Outer Packet
Source:      192.168.10.83
Destination: 192.168.10.84
Protocol:    UDP

Inner Packet
Source:      10.244.1.10
Destination: 10.244.2.20
```

이를 Encapsulation이라고 합니다.

Node B는 패킷을 받은 뒤 outer header를 벗겨내고 내부 Pod 패킷을 Pod C로 전달합니다. 이를 Decapsulation이라고 합니다.

```text
Pod A Packet
  ↓
Encapsulation
  ↓
Node A IP -> Node B IP
  ↓
Decapsulation
  ↓
Pod C로 전달
```

VXLAN은 일반적으로 UDP 4789 포트를 사용합니다.

확인 예시:

```bash
sudo tcpdump -i any udp port 4789
```

Overlay의 장점:

| 장점 | 설명 |
|---|---|
| 인프라 라우팅 변경이 적음 | 물리 네트워크가 Pod CIDR을 몰라도 됨 |
| 구축이 쉬움 | CNI가 노드 간 터널을 구성 |
| 클라우드/온프렘 모두 사용 가능 | 환경 의존성이 낮음 |

Overlay의 단점:

| 단점 | 설명 |
|---|---|
| Encapsulation 오버헤드 | 헤더가 추가되어 실제 payload 공간 감소 |
| MTU 문제 가능 | 1500 MTU에서 VXLAN 헤더만큼 줄어듦 |
| 디버깅 복잡도 증가 | inner/outer packet을 구분해야 함 |
| 성능 손실 가능 | 환경에 따라 CPU/네트워크 오버헤드 발생 |

## 15. VXLAN, Geneve, IPIP

Overlay 구현에는 여러 방식이 있습니다.

| 방식 | 설명 | 자주 쓰는 환경 |
|---|---|---|
| VXLAN | L2 frame을 UDP 위에 캡슐화 | Flannel, Calico, Cilium |
| Geneve | 확장성이 좋은 터널링 프로토콜 | OVN, 일부 CNI |
| IPIP | IP 패킷을 IP 패킷 안에 캡슐화 | Calico |

VXLAN 예시:

```text
Ethernet + IP + TCP
  ↓
VXLAN Header 추가
  ↓
UDP Header 추가
  ↓
Outer IP Header 추가
```

이 때문에 MTU를 고려해야 합니다.

예를 들어 실제 네트워크 MTU가 1500인데 VXLAN이 50바이트 정도의 오버헤드를 추가하면 Pod MTU는 1450 정도로 낮춰야 합니다.

```text
Underlay MTU: 1500
Overlay Header: 약 50
Pod MTU: 1450
```

MTU 설정이 맞지 않으면 다음 현상이 발생할 수 있습니다.

```text
작은 ping은 성공
큰 HTTP 응답은 실패
TLS handshake 중 멈춤
이미지 pull 중 timeout
gRPC stream 불안정
```

## 16. Native Routing / Direct Routing

Native Routing은 overlay로 패킷을 감싸지 않고, 실제 라우팅을 이용해 Pod CIDR 간 통신을 처리하는 방식입니다.

예시:

```text
Node A: 192.168.10.83
Pod CIDR: 10.244.1.0/24

Node B: 192.168.10.84
Pod CIDR: 10.244.2.0/24
```

Node A 라우팅 테이블에 다음 경로가 있어야 합니다.

```text
10.244.2.0/24 via 192.168.10.84
```

Node B 라우팅 테이블에는 다음 경로가 있어야 합니다.

```text
10.244.1.0/24 via 192.168.10.83
```

확인 명령어:

```bash
ip route
ip route get 10.244.2.20
```

Native Routing의 장점:

| 장점 | 설명 |
|---|---|
| Overlay 오버헤드 없음 | Encapsulation이 없어 성능상 유리 |
| MTU 문제 감소 | Underlay MTU를 더 온전히 활용 |
| 패킷 흐름이 명확 | 라우팅 테이블로 추적 가능 |

Native Routing의 단점:

| 단점 | 설명 |
|---|---|
| 라우팅 구성 필요 | 모든 Node/Router가 Pod CIDR 경로를 알아야 함 |
| 인프라 의존성 증가 | OpenStack, Bare Metal, Router 설정과 연계 필요 |
| 대규모 환경에서 route 관리 필요 | BGP 또는 CNI route 관리 필요 |

Cilium에서 `routing-mode: native`를 사용하는 경우 overlay 없이 직접 라우팅하는 구조입니다. 이때 중요한 것은 Node 간 Pod CIDR 경로가 정상적으로 구성되어야 한다는 점입니다.

## 17. BGP 기반 Pod Routing

BGP는 라우터끼리 경로 정보를 교환하는 프로토콜입니다.

일부 CNI는 BGP를 사용해 각 Node의 Pod CIDR을 네트워크에 광고합니다.

예시:

```text
Node A advertises: 10.244.1.0/24
Node B advertises: 10.244.2.0/24
Node C advertises: 10.244.3.0/24
```

그러면 라우터 또는 다른 Node는 각 Pod CIDR로 가는 경로를 자동으로 알 수 있습니다.

Calico가 대표적으로 BGP 기반 구성을 많이 사용합니다.

BGP 방식의 장점:

```text
Pod CIDR 경로를 동적으로 전파할 수 있다.
Overlay 없이 직접 라우팅할 수 있다.
대규모 Bare Metal 환경에서 유용하다.
```

단점:

```text
BGP 이해가 필요하다.
라우터/스위치와 연계될 수 있다.
잘못 설정하면 전체 네트워크 장애 범위가 커질 수 있다.
```

## 18. Pod-to-Service 통신

Kubernetes에서 애플리케이션은 보통 Pod IP가 아니라 Service를 통해 통신합니다.

예시:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: default
spec:
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080
```

Service가 생성되면 ClusterIP가 할당됩니다.

```bash
kubectl get svc backend
```

예시:

```text
NAME      TYPE        CLUSTER-IP      PORT(S)
backend   ClusterIP   10.96.120.15    8080/TCP
```

Pod는 다음처럼 요청합니다.

```bash
curl http://backend.default.svc.cluster.local:8080
```

DNS 해석 결과는 보통 Service의 ClusterIP입니다.

```text
backend.default.svc.cluster.local -> 10.96.120.15
```

그런데 중요한 점이 있습니다.

```text
ClusterIP는 실제 NIC에 붙어 있는 IP가 아니다.
```

Node에서 확인해도 보통 `10.96.120.15`가 인터페이스에 붙어 있지 않습니다.

```bash
ip addr | grep 10.96.120.15
```

안 나오는 것이 정상입니다.

그럼 어떻게 통신될까요?

kube-proxy 또는 eBPF datapath가 Service IP로 가는 트래픽을 가로채서 실제 Pod IP로 DNAT하기 때문입니다.

```text
Client Pod
  ↓
10.96.120.15:8080
  ↓
Service 처리 규칙
  ↓ DNAT
10.244.1.10:8080 또는 10.244.2.20:8080
  ↓
Backend Pod
```

## 19. kube-proxy

kube-proxy는 Kubernetes Service 트래픽을 실제 Backend Pod로 전달하기 위한 컴포넌트입니다.

이름 때문에 “프록시 서버”처럼 별도 프로세스가 모든 패킷을 직접 중계한다고 생각하기 쉽지만, 현대 Kubernetes의 kube-proxy는 주로 Linux 커널의 패킷 처리 기능을 설정하는 역할에 가깝습니다.

kube-proxy는 Kubernetes API를 watch합니다.

```text
Service 생성/변경/삭제
EndpointSlice 생성/변경/삭제
```

그리고 그 정보를 바탕으로 Node에 트래픽 전달 규칙을 구성합니다.

```text
Service ClusterIP:Port
  ↓
Backend PodIP:TargetPort 목록
```

예시:

```text
Service: 10.96.120.15:8080
Backend:
  - 10.244.1.10:8080
  - 10.244.2.20:8080
```

kube-proxy 동작 모드는 대표적으로 다음이 있습니다.

| 모드 | 설명 |
|---|---|
| userspace | 오래된 방식. kube-proxy 프로세스가 직접 중계. 현재 거의 사용하지 않음 |
| iptables | iptables NAT 규칙으로 ClusterIP를 Pod IP로 DNAT |
| IPVS | Linux IPVS 로드밸런서 기능 사용 |
| nftables | 최신 환경에서 nftables 기반 구현 가능 |
| eBPF 대체 | Cilium 등이 kube-proxy 없이 eBPF로 Service 처리 |

## 20. kube-proxy iptables 모드

iptables 모드에서는 Service 트래픽을 NAT 테이블에서 처리합니다.

흐름:

```text
Client Pod
  ↓
ClusterIP:Port
  ↓
iptables PREROUTING/OUTPUT
  ↓
KUBE-SERVICES Chain
  ↓
KUBE-SVC-xxxxx Chain
  ↓
KUBE-SEP-xxxxx Chain
  ↓
DNAT to PodIP:TargetPort
```

확인 명령어:

```bash
sudo iptables-save -t nat | grep KUBE-SVC
sudo iptables-save -t nat | grep KUBE-SEP
sudo iptables-save -t nat | grep 10.96.120.15
```

개념 예시:

```text
-A KUBE-SERVICES -d 10.96.120.15/32 -p tcp --dport 8080 -j KUBE-SVC-ABCDEF
-A KUBE-SVC-ABCDEF -m statistic --mode random --probability 0.5 -j KUBE-SEP-111111
-A KUBE-SVC-ABCDEF -j KUBE-SEP-222222
-A KUBE-SEP-111111 -j DNAT --to-destination 10.244.1.10:8080
-A KUBE-SEP-222222 -j DNAT --to-destination 10.244.2.20:8080
```

실제 규칙은 훨씬 복잡하지만 핵심은 다음입니다.

```text
ClusterIP로 들어온 패킷을 실제 Pod IP로 목적지 변환한다.
```

iptables 방식의 특징:

| 항목 | 설명 |
|---|---|
| 구현 | Linux iptables NAT |
| 장점 | 범용적이고 오래 사용됨 |
| 단점 | Service/Endpoint가 많아질수록 규칙 수 증가 |
| 로드밸런싱 | statistic match 기반 확률 분산 |
| 장애 분석 | iptables-save, conntrack 확인 필요 |

## 21. kube-proxy IPVS 모드

IPVS는 Linux 커널의 Layer 4 Load Balancer 기능입니다.

iptables 모드가 규칙 체인을 순차적으로 타는 방식이라면, IPVS는 Service를 가상 서버로 등록하고 backend Pod를 real server로 등록하는 방식입니다.

예시:

```text
Virtual Server:
  10.96.120.15:8080

Real Servers:
  10.244.1.10:8080
  10.244.2.20:8080
```

확인 명령어:

```bash
ipvsadm -Ln
```

예시 출력:

```text
TCP  10.96.120.15:8080 rr
  -> 10.244.1.10:8080 Masq 1 0 0
  -> 10.244.2.20:8080 Masq 1 0 0
```

IPVS 모드의 특징:

| 항목 | 설명 |
|---|---|
| 구현 | Linux IPVS |
| 장점 | 많은 Service/Endpoint에서 성능상 유리할 수 있음 |
| 알고리즘 | rr, lc, dh 등 다양한 L4 LB 알고리즘 |
| 확인 | ipvsadm |
| 주의 | 여전히 iptables와 함께 일부 처리가 사용될 수 있음 |

## 22. Cilium과 kube-proxy replacement

Cilium은 eBPF 기반 Kubernetes 네트워크, 보안, 관측성 솔루션입니다.

Cilium을 kube-proxy replacement 모드로 사용하면 kube-proxy 없이 Service 처리가 가능합니다.

일반 kube-proxy 방식:

```text
Service 변경
  ↓
kube-proxy가 iptables/IPVS 규칙 갱신
  ↓
커널 NAT/로드밸런싱 처리
```

Cilium eBPF 방식:

```text
Service 변경
  ↓
Cilium Agent가 Kubernetes API watch
  ↓
eBPF map에 Service/Backend 정보 저장
  ↓
커널 eBPF 프로그램이 패킷 처리
```

eBPF는 Linux 커널 안에서 안전하게 실행되는 프로그램입니다.

Cilium은 eBPF를 사용해 다음 기능을 구현할 수 있습니다.

| 기능 | 설명 |
|---|---|
| Pod-to-Pod routing | Pod 간 패킷 전달 |
| Service load balancing | ClusterIP, NodePort, LoadBalancer 처리 |
| NetworkPolicy | L3/L4/L7 정책 적용 |
| Observability | Flow, drop reason, policy verdict 관측 |
| kube-proxy replacement | iptables/IPVS 없이 Service 처리 |
| Host firewall | Node 레벨 방화벽 |
| Encryption | WireGuard/IPsec 기반 암호화 구성 가능 |

Cilium 상태 확인:

```bash
cilium status
cilium config view
kubectl -n kube-system get ds cilium
kubectl -n kube-system get cm cilium-config -o yaml
```

kube-proxy가 없는 환경에서는 다음을 확인합니다.

```bash
kubectl -n kube-system get ds kube-proxy
```

없거나 비활성화되어 있으면 Cilium이 Service를 처리하는 구성일 수 있습니다.

Cilium 설정에서 자주 보는 값:

```yaml
kubeProxyReplacement: "true"
routing-mode: "native"
tunnel: "disabled"
bpf-lb-algorithm: "maglev"
bpf-lb-mode: "snat"
```

각 의미는 대략 다음과 같습니다.

| 설정 | 의미 |
|---|---|
| kubeProxyReplacement | kube-proxy 기능을 Cilium eBPF로 대체 |
| routing-mode=native | Overlay 없이 직접 라우팅 사용 |
| tunnel=disabled | VXLAN/Geneve 터널 비활성화 |
| bpf-lb-algorithm=maglev | 일관성 있는 로드밸런싱 알고리즘 |
| bpf-lb-mode=snat | 특정 트래픽에서 source NAT 사용 |

정확한 의미는 Cilium 버전과 설치 옵션에 따라 달라질 수 있으므로, 실제 운영 환경에서는 `cilium config view`와 Helm values를 함께 확인해야 합니다.

## 23. eBPF datapath를 이해하는 기준

eBPF를 깊게 파고들면 커널 내부 주제라 어렵습니다. Kubernetes 네트워크 관점에서는 다음 정도를 먼저 이해하면 됩니다.

```text
기존 방식:
  패킷이 커널 네트워크 스택을 지나면서 iptables 규칙을 통과한다.

eBPF 방식:
  커널의 특정 지점에 작은 프로그램을 붙여 더 이른 시점에 패킷을 판단하고 처리한다.
```

Cilium이 사용하는 주요 hook 위치는 다음과 같이 설명할 수 있습니다.

| 위치 | 설명 |
|---|---|
| XDP | NIC에 매우 가까운 지점. 초고속 drop/redirect 가능 |
| TC | Linux traffic control 계층. Pod veth ingress/egress 처리에 자주 사용 |
| Socket LB | 애플리케이션 socket connect 단계에서 Service IP를 backend IP로 변환 가능 |
| cgroup hook | 프로세스/컨테이너 단위 네트워크 제어 가능 |

단순 그림:

```text
Pod process
  ↓
Socket
  ↓       ← Socket LB 가능
Kernel network stack
  ↓
veth eth0
  ↓       ← TC eBPF 가능
Node datapath
  ↓
NIC
  ↓       ← XDP 가능
Network
```

Cilium 사용 환경에서 장애 분석 시에는 iptables만 보면 안 됩니다.

```text
kube-proxy iptables 환경:
  iptables-save, conntrack 중심

Cilium eBPF 환경:
  cilium status, cilium monitor, hubble observe, bpftool, cilium bpf lb list 중심
```

예시:

```bash
cilium service list
cilium bpf lb list
cilium monitor
hubble observe
```

## 24. Service Network와 ClusterIP의 정체

Service Network는 ClusterIP가 할당되는 대역입니다.

예시:

```text
Service CIDR: 10.96.0.0/12
CoreDNS:      10.96.0.10
backend:      10.96.120.15
```

ClusterIP는 일반 서버의 NIC에 붙은 IP가 아닙니다.

```bash
ip addr | grep 10.96
```

대부분 나오지 않습니다.

ClusterIP는 Kubernetes Service abstraction을 위한 virtual IP입니다.

```text
Client
  ↓
ClusterIP:Port
  ↓
kube-proxy/eBPF Service handling
  ↓
PodIP:TargetPort
```

그래서 ClusterIP 장애를 볼 때는 다음을 구분해야 합니다.

| 확인 대상 | 명령어 |
|---|---|
| Service 존재 여부 | `kubectl get svc -A` |
| Service selector | `kubectl describe svc <svc>` |
| EndpointSlice 존재 여부 | `kubectl get endpointslice -A` |
| Backend Pod Ready 여부 | `kubectl get pod -o wide` |
| kube-proxy 규칙 | `iptables-save`, `ipvsadm` |
| Cilium Service map | `cilium service list`, `cilium bpf lb list` |

## 25. Endpoint와 EndpointSlice 개념 연결

Service는 selector를 통해 Backend Pod를 찾습니다.

```yaml
selector:
  app: backend
```

Pod에는 label이 있어야 합니다.

```yaml
labels:
  app: backend
```

Kubernetes는 이 관계를 바탕으로 EndpointSlice를 생성합니다.

```text
Service
  selector: app=backend
      ↓
Pod 목록
  10.244.1.10:8080
  10.244.2.20:8080
      ↓
EndpointSlice
```

Service는 “고정 진입점”이고, EndpointSlice는 “현재 실제 backend 목록”입니다.

```text
Service ClusterIP: 10.96.120.15
EndpointSlice:
  - 10.244.1.10:8080
  - 10.244.2.20:8080
```

Service가 있는데 EndpointSlice가 비어 있으면 다음과 같은 문제가 발생합니다.

```text
Service는 존재한다.
DNS도 된다.
ClusterIP도 있다.
하지만 실제로 보낼 Pod가 없다.
결과적으로 503, connection refused, timeout 등이 발생할 수 있다.
```

확인:

```bash
kubectl get svc backend
kubectl get endpoints backend
kubectl get endpointslice -l kubernetes.io/service-name=backend
kubectl describe svc backend
```

Service/EndpointSlice는 다음 글에서 더 자세히 다룰 수 있으므로, 여기서는 “Service 트래픽 처리의 backend 목록”으로 이해하면 됩니다.

## 26. Pod-to-Internet 통신

Pod가 외부 인터넷으로 나가는 경우를 보겠습니다.

```text
Pod: 10.244.1.10
Node: 192.168.10.83
Internet: 8.8.8.8
```

Pod가 외부로 요청합니다.

```bash
curl https://example.com
```

흐름:

```text
Pod 10.244.1.10
  ↓
Node로 패킷 전달
  ↓
Node 또는 CNI에서 SNAT/Masquerade
  ↓
Source IP가 Node IP 또는 Egress IP로 변환
  ↓
외부 인터넷
```

왜 SNAT이 필요할까요?

외부 인터넷은 `10.244.1.10`이라는 Pod IP로 응답을 보낼 경로를 모릅니다. Pod CIDR은 클러스터 내부 주소이기 때문입니다.

그래서 외부로 나갈 때 source IP를 Node IP 등 라우팅 가능한 IP로 바꿉니다.

```text
Before SNAT:
  Source:      10.244.1.10
  Destination: 93.184.216.34

After SNAT:
  Source:      192.168.10.83
  Destination: 93.184.216.34
```

응답이 돌아오면 conntrack 정보를 보고 다시 Pod IP로 되돌립니다.

```text
Response to 192.168.10.83
  ↓
conntrack NAT reverse
  ↓
Pod 10.244.1.10
```

확인:

```bash
sudo iptables-save -t nat | grep MASQUERADE
sudo conntrack -L | grep 10.244
```

Cilium 환경에서는 Cilium의 masquerade 설정을 확인해야 합니다.

```bash
cilium config view | grep -i masquerade
kubectl -n kube-system get cm cilium-config -o yaml | grep -i masquerade
```

## 27. conntrack

conntrack은 Linux 커널의 connection tracking 기능입니다.

NAT가 정상 동작하려면 연결 상태를 기억해야 합니다.

예를 들어 Pod가 외부로 나갈 때 SNAT이 발생합니다.

```text
10.244.1.10:52344 -> 93.184.216.34:443
SNAT
192.168.10.83:40001 -> 93.184.216.34:443
```

응답은 다음처럼 돌아옵니다.

```text
93.184.216.34:443 -> 192.168.10.83:40001
```

커널은 conntrack 테이블을 보고 이 응답이 원래 `10.244.1.10:52344`로 돌아가야 한다는 것을 압니다.

conntrack 장애 또는 테이블 고갈이 발생하면 다음 문제가 생길 수 있습니다.

```text
간헐적 연결 실패
Service timeout
DNS timeout
외부 API 호출 실패
502/504 증가
새 연결 생성 실패
```

확인 명령어:

```bash
sudo conntrack -S
sudo conntrack -L | head
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_count
```

운영 환경에서는 conntrack 사용량을 모니터링해야 합니다.

```text
nf_conntrack_count / nf_conntrack_max
```

비율이 너무 높으면 connection tracking table 고갈을 의심해야 합니다.

## 28. DNS와 CoreDNS

Kubernetes 내부 Service Discovery의 기본은 DNS입니다.

Service가 생성되면 Kubernetes는 DNS 레코드를 제공합니다.

일반 Service:

```text
backend.default.svc.cluster.local -> ClusterIP
```

예시:

```bash
kubectl exec -it <pod> -- nslookup backend.default.svc.cluster.local
```

Pod 내부의 `/etc/resolv.conf`를 보면 보통 다음과 유사합니다.

```text
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

각 항목의 의미:

| 항목 | 설명 |
|---|---|
| nameserver | CoreDNS Service IP |
| search | 짧은 이름을 자동으로 확장할 DNS suffix |
| ndots | 이름에 점이 몇 개 이상 있어야 FQDN으로 먼저 볼지 결정 |

예를 들어 Pod가 default namespace에 있고 다음을 요청한다고 가정합니다.

```bash
curl http://backend:8080
```

DNS resolver는 search domain을 이용해 다음 이름들을 시도할 수 있습니다.

```text
backend.default.svc.cluster.local
backend.svc.cluster.local
backend.cluster.local
backend
```

그래서 같은 namespace 안에서는 Service 이름만 써도 통신이 됩니다.

```bash
curl http://backend:8080
```

다른 namespace의 Service에 접근하려면 namespace를 포함하는 것이 좋습니다.

```bash
curl http://backend.database.svc.cluster.local:8080
curl http://backend.database:8080
```

## 29. CoreDNS 동작 흐름

CoreDNS는 Kubernetes API를 watch하면서 Service와 Pod 정보를 DNS 응답으로 제공합니다.

흐름:

```text
1. Service backend가 생성된다.
2. Kubernetes API Server에 Service 정보가 저장된다.
3. CoreDNS kubernetes plugin이 Service 정보를 watch한다.
4. Pod가 backend.default.svc.cluster.local을 질의한다.
5. CoreDNS가 ClusterIP를 응답한다.
6. Pod는 ClusterIP로 요청한다.
7. kube-proxy/eBPF가 실제 Pod IP로 전달한다.
```

그림:

```text
Pod
  |
  | DNS Query: backend.default.svc.cluster.local
  v
CoreDNS Service: 10.96.0.10
  |
  | Kubernetes plugin
  v
Kubernetes API Server
  |
  | DNS Answer: 10.96.120.15
  v
Pod
  |
  | HTTP Request to 10.96.120.15
  v
Service datapath
  |
  v
Backend Pod
```

CoreDNS 확인:

```bash
kubectl -n kube-system get deploy coredns
kubectl -n kube-system get pod -l k8s-app=kube-dns -o wide
kubectl -n kube-system logs deploy/coredns
kubectl -n kube-system get svc kube-dns
```

Pod 내부 테스트:

```bash
kubectl run dns-test --rm -it --image=busybox:1.36 -- nslookup kubernetes.default
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
dig kubernetes.default.svc.cluster.local
```

DNS 문제와 네트워크 문제를 분리하는 방법:

```text
Service DNS 이름이 안 된다.
  ↓
ClusterIP로 직접 curl 해본다.
  ↓
ClusterIP는 되면 DNS 문제.
  ↓
ClusterIP도 안 되면 Service/Endpoint/datapath 문제.
```

## 30. Headless Service

일반 Service는 DNS가 ClusterIP를 반환합니다.

```text
backend.default.svc.cluster.local -> 10.96.120.15
```

Headless Service는 `clusterIP: None`으로 만드는 Service입니다.

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
    - port: 5432
```

Headless Service에서는 DNS가 ClusterIP가 아니라 Pod IP 목록을 반환할 수 있습니다.

```text
database.default.svc.cluster.local
  -> 10.244.1.10
  -> 10.244.2.20
```

StatefulSet에서는 Pod별 고정 DNS 이름이 중요합니다.

```text
postgres-0.database.default.svc.cluster.local
postgres-1.database.default.svc.cluster.local
postgres-2.database.default.svc.cluster.local
```

이 구조는 다음과 같은 시스템에서 중요합니다.

```text
Kafka
RabbitMQ
PostgreSQL Cluster
MongoDB
Ceph 관련 컴포넌트
분산 DB
Stateful 애플리케이션
```

일반 Deployment 기반 웹 애플리케이션은 보통 ClusterIP Service를 사용하고, StatefulSet 기반 분산 시스템은 Headless Service를 자주 사용합니다.

## 31. Service Discovery 방식

Kubernetes Service Discovery는 크게 두 가지 방식이 있습니다.

### 31-1) DNS 기반 Service Discovery

가장 일반적이고 권장되는 방식입니다.

```bash
curl http://backend.default.svc.cluster.local:8080
```

장점:

```text
Service가 나중에 생성되어도 DNS로 조회 가능
Pod 재시작 없이 사용 가능
namespace 기반 이름 체계 제공
대부분의 애플리케이션과 호환
```

### 31-2) 환경변수 기반 Service Discovery

Pod 생성 시점에 존재하던 Service 정보는 환경변수로 주입될 수 있습니다.

```bash
env | grep SERVICE
```

예시:

```text
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
```

하지만 이 방식은 한계가 있습니다.

```text
Pod 생성 후 새로 만들어진 Service는 기존 Pod 환경변수에 자동 반영되지 않는다.
환경변수가 많아질 수 있다.
동적 환경에는 DNS가 더 적합하다.
```

따라서 일반적으로는 DNS 기반 Service Discovery를 사용합니다.

## 32. NetworkPolicy

Kubernetes 기본 네트워크 모델에서는 Pod 간 통신이 기본적으로 허용됩니다.

즉 아무 정책도 없으면 보통 다음이 가능합니다.

```text
frontend Pod -> backend Pod
backend Pod -> database Pod
임의 Pod -> 임의 Pod
```

NetworkPolicy는 Pod 간 트래픽을 L3/L4 기준으로 제한하는 Kubernetes 리소스입니다.

중요한 전제:

```text
NetworkPolicy는 CNI가 구현해야 실제로 동작한다.
```

NetworkPolicy 리소스를 만들었더라도 CNI가 NetworkPolicy를 지원하지 않거나 enforcement가 꺼져 있으면 효과가 없습니다.

예시 정책:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

의미:

```text
default namespace의 app=backend Pod에 대해
Ingress 트래픽을 제한한다.
app=frontend Pod에서 TCP 8080으로 오는 트래픽만 허용한다.
```

중요한 동작 방식:

```text
어떤 Pod가 NetworkPolicy에 의해 선택되기 전까지는 기본 허용 상태이다.

NetworkPolicy가 특정 Pod를 선택하면,
그 방향(Ingress/Egress)에 대해서는 명시적으로 허용된 트래픽만 허용된다.
```

즉 NetworkPolicy는 “차단 목록”이라기보다 “허용 목록” 방식으로 이해하는 것이 안전합니다.

## 33. Ingress Policy와 Egress Policy

NetworkPolicy에서 Ingress와 Egress는 방향을 의미합니다.

| 방향 | 기준 | 설명 |
|---|---|---|
| Ingress | 선택된 Pod로 들어오는 트래픽 | 누가 이 Pod에 접근할 수 있는가 |
| Egress | 선택된 Pod에서 나가는 트래픽 | 이 Pod가 어디로 나갈 수 있는가 |

예를 들어 backend Pod를 기준으로 보면:

```text
frontend -> backend
```

backend 입장에서는 Ingress입니다.

```text
backend -> database
```

backend 입장에서는 Egress입니다.

Egress 제한 예시:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-only-database
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

이 정책을 적용하면 backend Pod는 database Pod의 5432 포트로만 나갈 수 있습니다.

주의:

```text
Egress를 제한하면 DNS 질의도 막힐 수 있다.
```

DNS를 사용하려면 CoreDNS로 가는 UDP/TCP 53을 허용해야 할 수 있습니다.

예시:

```yaml
egress:
  - to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
    ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
```

## 34. MTU와 Kubernetes 네트워크

MTU는 Maximum Transmission Unit의 약자입니다.

한 번에 전송할 수 있는 최대 패킷 크기를 의미합니다.

일반 Ethernet MTU는 보통 1500입니다.

```bash
ip link
```

예시:

```text
ens3: mtu 1500
eth0: mtu 1450
```

Kubernetes에서 MTU가 중요한 이유는 overlay 때문입니다.

VXLAN을 사용하면 기존 패킷에 외부 헤더가 추가됩니다.

```text
Inner Packet
  +
VXLAN/UDP/Outer IP Header
```

이때 실제 물리 네트워크 MTU가 1500인데 Pod가 1500 크기의 패킷을 보내면, overlay 헤더가 추가되면서 1500을 초과할 수 있습니다.

결과:

```text
패킷 단편화 발생
또는
DF bit 때문에 drop
```

증상:

```text
ping은 된다.
curl 작은 응답은 된다.
큰 파일 다운로드는 실패한다.
TLS handshake가 멈춘다.
이미지 pull이 중간에 실패한다.
gRPC stream이 불안정하다.
Ceph/RBD/NFS 같은 스토리지 트래픽이 간헐적으로 느려진다.
```

확인 방법:

```bash
# Linux에서 DF bit를 세우고 큰 ping 테스트
ping -M do -s 1472 <target-ip>

# tracepath는 경로 MTU 추정에 유용
tracepath <target-ip>
```

MTU 계산 예:

```text
Ethernet MTU: 1500
VXLAN overhead: 약 50
Pod MTU: 1450
```

Cilium 설정 확인:

```bash
kubectl -n kube-system get cm cilium-config -o yaml | grep -i mtu
cilium config view | grep -i mtu
```

## 35. Hairpin Traffic

Hairpin traffic은 트래픽이 나갔다가 다시 같은 방향으로 돌아오는 형태를 말합니다.

Kubernetes에서 자주 보는 예는 다음입니다.

```text
Pod A
  ↓
Service ClusterIP
  ↓
Service가 선택한 backend가 다시 Pod A
```

즉 Pod가 자기 자신이 포함된 Service로 요청했는데, 로드밸런싱 결과 자기 자신에게 다시 돌아오는 상황입니다.

```text
Pod A -> backend Service -> Pod A
```

이런 트래픽은 CNI, kube-proxy, bridge hairpin mode, NAT 설정과 관련될 수 있습니다.

증상:

```text
Pod IP 직접 접근은 된다.
Service로 접근하면 특정 Pod에서만 실패한다.
자기 자신으로 돌아가는 요청만 실패한다.
```

확인:

```bash
kubectl get endpointslice -l kubernetes.io/service-name=<service>
kubectl exec -it <pod> -- curl -v http://<service-name>:<port>
```

일반적인 웹 서비스에서는 크게 문제되지 않을 수 있지만, 프록시, 사이드카, 로컬 콜백, self-check 구조에서는 중요해질 수 있습니다.

## 36. HostNetwork와 HostPort

### 36-1) HostNetwork

Pod에 `hostNetwork: true`를 설정하면 Pod는 별도 Pod network namespace가 아니라 Node의 network namespace를 사용합니다.

예시:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostnet-pod
spec:
  hostNetwork: true
  containers:
    - name: nginx
      image: nginx
```

특징:

```text
Pod가 Node IP를 사용한다.
Pod 안에서 보이는 네트워크 인터페이스가 Node와 같다.
Node의 포트 공간을 공유한다.
DNS policy를 별도로 고려해야 할 수 있다.
```

사용 사례:

```text
CNI agent
Node monitoring agent
일부 ingress controller
NodeLocal DNSCache
host-level network daemon
```

주의:

```text
보안 격리가 약해진다.
포트 충돌 가능성이 있다.
Pod IP 기반 NetworkPolicy 적용이 제한될 수 있다.
```

### 36-2) HostPort

HostPort는 Pod의 특정 containerPort를 Node의 특정 포트에 바인딩하는 기능입니다.

```yaml
ports:
  - containerPort: 8080
    hostPort: 8080
```

그러면 Node IP의 8080 포트로 들어온 트래픽이 해당 Pod로 전달됩니다.

주의:

```text
같은 Node에서 같은 hostPort를 사용하는 Pod는 동시에 실행될 수 없다.
스케줄링 제약이 생긴다.
대부분의 경우 Service 또는 Ingress가 더 적합하다.
```

## 37. NodeLocal DNSCache

NodeLocal DNSCache는 각 Node에 DNS cache agent를 두어 DNS 성능과 안정성을 개선하는 구성입니다.

일반 DNS 흐름:

```text
Pod
  ↓
CoreDNS Service IP
  ↓
CoreDNS Pod
```

NodeLocal DNSCache 사용 시:

```text
Pod
  ↓
Node local DNS cache
  ↓
CoreDNS
```

장점:

```text
DNS latency 감소
CoreDNS 부하 감소
conntrack DNS entry 증가 완화
Node 단위 캐싱
```

DNS timeout이 많거나 CoreDNS 부하가 높은 환경에서는 NodeLocal DNSCache를 고려할 수 있습니다.

확인:

```bash
kubectl -n kube-system get ds | grep node-local
kubectl -n kube-system get pods -o wide | grep node-local
```

## 38. Kubernetes 요청 흐름 전체 예시

사용자가 외부에서 애플리케이션에 접속하는 흐름을 보겠습니다.

```text
User Browser
  ↓
DNS
  ↓
External Load Balancer
  ↓
Ingress Controller Pod
  ↓
Service ClusterIP
  ↓
EndpointSlice
  ↓
Backend Pod IP
  ↓
Container Port
```

Kubernetes 내부에서는 다음 흐름도 함께 발생합니다.

```text
Ingress Controller Pod
  ↓ DNS 또는 Service ClusterIP
kube-proxy/eBPF
  ↓ DNAT / Load Balancing
Backend Pod
```

장애 분석은 이 흐름을 단계별로 쪼개야 합니다.

```text
1. 외부 DNS가 맞는가?
2. Load Balancer까지 도달하는가?
3. Ingress Controller가 요청을 받는가?
4. Ingress rule의 Host/Path가 맞는가?
5. Service가 존재하는가?
6. EndpointSlice에 Ready Pod가 있는가?
7. Pod IP로 직접 접근 가능한가?
8. Pod 내부 애플리케이션이 포트를 listen하는가?
9. NetworkPolicy가 막고 있지 않은가?
10. CoreDNS가 정상인가?
11. CNI datapath에 drop이 있는가?
```

## 39. Kubernetes 네트워크 트러블슈팅 기본 원칙

네트워크 장애는 한 번에 전체를 보려고 하면 어렵습니다.

항상 계층과 흐름을 나눠야 합니다.

```text
DNS 문제인가?
Service 문제인가?
Endpoint 문제인가?
Pod 문제인가?
CNI 문제인가?
Node 라우팅 문제인가?
NetworkPolicy 문제인가?
외부 LB/Ingress 문제인가?
```

가장 좋은 방식은 “직접 IP 접근 → Service 접근 → DNS 접근 → Ingress 접근” 순서로 확인하는 것입니다.

예시:

```text
1. Backend Pod IP 직접 curl
2. Service ClusterIP curl
3. Service DNS curl
4. Ingress Host curl
5. External LB curl
```

## 40. Pod IP 통신 확인

Pod IP 직접 통신이 되는지 확인합니다.

```bash
kubectl get pods -A -o wide
kubectl exec -it <client-pod> -- ping <backend-pod-ip>
kubectl exec -it <client-pod> -- curl -v http://<backend-pod-ip>:<port>
```

확인할 것:

| 증상 | 의심 지점 |
|---|---|
| ping 실패 | CNI, route, NetworkPolicy, ICMP 차단 |
| curl connection refused | Pod까지 도달했지만 애플리케이션 포트 미오픈 |
| curl timeout | route, NetworkPolicy, firewall, CNI drop |
| 특정 Node 간만 실패 | Node-to-Node routing, MTU, CNI agent 문제 |

Pod 내부 route 확인:

```bash
kubectl exec -it <pod> -- ip addr
kubectl exec -it <pod> -- ip route
kubectl exec -it <pod> -- ip neigh
```

## 41. Service 통신 확인

Service가 정상인지 확인합니다.

```bash
kubectl get svc -A
kubectl describe svc <service-name>
kubectl get endpoints <service-name>
kubectl get endpointslice -l kubernetes.io/service-name=<service-name>
```

확인할 것:

```text
Service selector가 Pod label과 맞는가?
EndpointSlice에 backend Pod IP가 있는가?
Pod가 Ready 상태인가?
Service port와 targetPort가 맞는가?
```

테스트:

```bash
kubectl exec -it <client-pod> -- curl -v http://<cluster-ip>:<port>
kubectl exec -it <client-pod> -- curl -v http://<service-name>.<namespace>.svc.cluster.local:<port>
```

Service는 되는데 Pod IP가 안 되면 이상합니다. 일반적으로 Service는 결국 Pod IP로 전달되기 때문입니다.

Pod IP는 되는데 Service가 안 되면 다음을 의심합니다.

```text
kube-proxy 규칙 문제
Cilium Service map 문제
EndpointSlice 문제
Service port/targetPort 불일치
SessionAffinity 문제
NetworkPolicy 문제
```

## 42. DNS 통신 확인

DNS 문제를 확인합니다.

```bash
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes.default
kubectl exec -it <pod> -- nslookup <service>.<namespace>.svc.cluster.local
```

netshoot 사용:

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
dig kubernetes.default.svc.cluster.local
dig <service>.<namespace>.svc.cluster.local
```

CoreDNS 상태:

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system logs deploy/coredns
kubectl -n kube-system describe deploy coredns
kubectl -n kube-system get cm coredns -o yaml
```

DNS 장애에서 자주 보는 원인:

| 증상 | 원인 후보 |
|---|---|
| 모든 DNS 실패 | CoreDNS 장애, kube-dns Service 장애, CNI 장애 |
| 외부 도메인만 실패 | CoreDNS forward/upstream 문제 |
| 특정 Service만 실패 | Service 없음, namespace 오류, search domain 오해 |
| DNS timeout | NetworkPolicy egress 53 차단, CoreDNS 과부하, conntrack 문제 |
| 간헐적 DNS 실패 | CoreDNS replica 부족, NodeLocal DNSCache 미사용, conntrack 고갈 |

## 43. CNI 상태 확인

CNI 문제는 Pod-to-Pod, Pod-to-Service, DNS, Ingress까지 광범위하게 영향을 줍니다.

공통 확인:

```bash
kubectl -n kube-system get pods -o wide
kubectl -n kube-system get ds
kubectl get nodes -o wide
kubectl describe node <node-name>
```

Cilium 환경:

```bash
cilium status
cilium connectivity test
cilium config view
cilium service list
cilium bpf lb list
hubble observe
```

Cilium Pod 로그:

```bash
kubectl -n kube-system logs ds/cilium -c cilium-agent
kubectl -n kube-system logs deploy/cilium-operator
```

자주 보는 문제:

```text
cilium-agent CrashLoop
Node 간 health check 실패
Pod CIDR route 누락
MTU 불일치
BPF map full
Policy drop
DNS proxy 문제
kube-proxy replacement 설정 불일치
```

## 44. NetworkPolicy 확인

NetworkPolicy 때문에 통신이 막히는 경우도 많습니다.

확인:

```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy <policy-name>
```

체크할 것:

```text
정책이 선택하는 podSelector가 맞는가?
namespaceSelector가 의도대로 동작하는가?
Ingress/Egress 방향을 헷갈리지 않았는가?
DNS egress UDP/TCP 53을 허용했는가?
CNI가 NetworkPolicy를 실제로 지원하는가?
```

테스트 Pod를 만들어 확인:

```bash
kubectl run test-client --rm -it --image=nicolaka/netshoot -- bash
curl -v http://backend:8080
dig backend.default.svc.cluster.local
```

Cilium에서는 drop reason을 볼 수 있습니다.

```bash
hubble observe --verdict DROPPED
cilium monitor --type drop
```

## 45. MTU 문제 확인

MTU 문제는 매우 까다롭습니다. 작은 요청은 성공하고 큰 요청만 실패할 수 있기 때문입니다.

확인:

```bash
kubectl exec -it <pod> -- ip link
kubectl exec -it <pod> -- tracepath <target-ip>
kubectl exec -it <pod> -- ping -M do -s 1400 <target-ip>
kubectl exec -it <pod> -- ping -M do -s 1450 <target-ip>
```

Node에서도 확인합니다.

```bash
ip link
tracepath <other-node-ip>
```

증상:

```text
curl -I는 되는데 파일 다운로드 실패
TLS handshake가 특정 구간에서 멈춤
Docker image pull이 중간에 멈춤
Ceph/RBD mount가 간헐적으로 지연
gRPC stream reset
```

Overlay 환경에서는 Pod MTU가 underlay MTU보다 작아야 합니다.

Native routing 환경에서는 overlay 오버헤드가 없지만, OpenStack, VLAN, Geneve, VXLAN 등 하부 인프라가 별도 오버헤드를 갖는지 확인해야 합니다.

## 46. Node 라우팅 확인

Pod 간 통신이 Node 경계를 넘을 때 라우팅이 중요합니다.

확인:

```bash
ip route
ip route get <pod-ip>
kubectl get nodes -o wide
kubectl get pods -A -o wide
```

예시:

```bash
ip route get 10.244.2.20
```

결과에서 어느 인터페이스로 나가는지 확인합니다.

```text
10.244.2.20 via 192.168.10.84 dev ens3 src 192.168.10.83
```

이런 경로가 있어야 native routing에서 정상 통신됩니다.

라우팅 문제가 의심되는 경우:

```text
같은 Node Pod끼리는 된다.
다른 Node Pod끼리는 안 된다.
특정 Node에 있는 Pod만 안 된다.
Node 재부팅 후 Pod 통신이 깨졌다.
CNI agent가 route를 제대로 못 넣었다.
```

## 47. 패킷 캡처

tcpdump는 최후의 확인 도구입니다.

Pod 내부에서 캡처:

```bash
kubectl exec -it <pod> -- tcpdump -nn -i any
```

Node에서 Pod IP 기준 캡처:

```bash
sudo tcpdump -nn -i any host <pod-ip>
```

Service 트래픽 확인:

```bash
sudo tcpdump -nn -i any host <cluster-ip>
sudo tcpdump -nn -i any host <backend-pod-ip>
```

VXLAN 확인:

```bash
sudo tcpdump -nn -i any udp port 4789
```

DNS 확인:

```bash
sudo tcpdump -nn -i any port 53
```

TCP handshake 확인:

```bash
sudo tcpdump -nn -i any tcp port 8080
```

해석 기준:

```text
SYN이 나갔는데 SYN-ACK이 안 온다.
  → route, firewall, NetworkPolicy, backend listen 상태 확인

SYN-ACK은 왔는데 ACK가 안 간다.
  → 비대칭 라우팅, conntrack, 중간 장비 확인

DNS query가 나갔는데 응답이 없다.
  → CoreDNS, NetworkPolicy, kube-dns Service 확인

Pod IP로는 응답이 있는데 Service IP로는 안 된다.
  → kube-proxy/eBPF Service handling 확인
```

## 48. Kubernetes 네트워크 장애 사례

### 48-1) Service는 있는데 503 발생

가능 원인:

```text
EndpointSlice가 비어 있음
Pod Ready가 아님
readinessProbe 실패
Service selector와 Pod label 불일치
```

확인:

```bash
kubectl describe svc <svc>
kubectl get endpointslice -l kubernetes.io/service-name=<svc>
kubectl get pod -l app=<label> -o wide
kubectl describe pod <pod>
```

### 48-2) DNS 이름만 실패

가능 원인:

```text
CoreDNS 장애
Pod /etc/resolv.conf 문제
NetworkPolicy가 UDP/TCP 53 차단
Service 이름 또는 namespace 오타
CoreDNS upstream 장애
```

확인:

```bash
kubectl exec -it <pod> -- nslookup <service>
kubectl exec -it <pod> -- curl http://<cluster-ip>:<port>
kubectl -n kube-system logs deploy/coredns
```

### 48-3) Pod IP는 되는데 Service가 안 됨

가능 원인:

```text
kube-proxy 장애
Cilium eBPF Service map 문제
EndpointSlice 미반영
Service port/targetPort 불일치
conntrack 문제
```

확인:

```bash
kubectl get svc,endpointslice
iptables-save -t nat | grep KUBE
ipvsadm -Ln
cilium service list
cilium bpf lb list
```

### 48-4) 같은 Node는 되는데 다른 Node는 안 됨

가능 원인:

```text
Node 간 Pod CIDR route 누락
CNI agent 장애
Overlay UDP 포트 차단
MTU 문제
OpenStack security group / firewall 문제
```

확인:

```bash
kubectl get pod -o wide
ip route get <remote-pod-ip>
ping <remote-node-ip>
tcpdump -i any host <remote-pod-ip>
```

### 48-5) 외부 API 호출만 실패

가능 원인:

```text
SNAT/Masquerade 문제
Egress firewall
Proxy 설정 누락
DNS upstream 문제
conntrack 고갈
```

확인:

```bash
kubectl exec -it <pod> -- curl -v https://example.com
kubectl exec -it <pod> -- nslookup example.com
iptables-save -t nat | grep MASQUERADE
conntrack -S
```

## 49. 실전 점검 명령어 모음

전체 리소스 확인:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get svc -A
kubectl get endpoints -A
kubectl get endpointslice -A
kubectl get ingress -A
kubectl get networkpolicy -A
```

Pod 내부 확인:

```bash
kubectl exec -it <pod> -- ip addr
kubectl exec -it <pod> -- ip route
kubectl exec -it <pod> -- ip neigh
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- ss -lntp
kubectl exec -it <pod> -- curl -v http://<target>
kubectl exec -it <pod> -- nslookup <service>
```

임시 디버그 Pod:

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
kubectl run busybox --rm -it --image=busybox:1.36 -- sh
```

Node 확인:

```bash
ip addr
ip route
ip neigh
bridge link
ss -lntp
iptables-save -t nat
conntrack -S
```

Cilium 확인:

```bash
cilium status
cilium connectivity test
cilium config view
cilium service list
cilium bpf lb list
hubble observe
```

CoreDNS 확인:

```bash
kubectl -n kube-system get svc kube-dns
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system logs deploy/coredns
kubectl -n kube-system get cm coredns -o yaml
```

## 50. 정리

Kubernetes 네트워크는 별도의 마법이 아닙니다.

실제로는 Linux 네트워크 기능을 Kubernetes 방식으로 자동 구성한 것입니다.

핵심 흐름은 다음과 같습니다.

```text
Pod 생성
  ↓
Network Namespace 생성
  ↓
Pause Container가 namespace 소유
  ↓
CNI가 veth pair 생성
  ↓
Pod eth0에 IP 할당
  ↓
Node datapath와 연결
  ↓
Pod-to-Pod 라우팅 구성
  ↓
Service는 kube-proxy/eBPF로 Pod에 전달
  ↓
CoreDNS는 Service 이름을 IP로 변환
  ↓
NetworkPolicy는 CNI가 트래픽을 허용/차단
```

가장 중요한 관점은 다음입니다.

```text
Pod IP 통신이 되는가?
Service ClusterIP 통신이 되는가?
Service DNS 통신이 되는가?
Ingress/LoadBalancer 통신이 되는가?
```

이 순서대로 보면 장애 범위를 좁힐 수 있습니다.
