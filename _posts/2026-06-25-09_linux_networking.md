---
layout: post
title: "Linux 네트워크: NIC, Bridge, Namespace, veth, Routing Table, ip 명령어"
date: 2026-06-25 07:27:00 +0900
categories: [Linux, Network]
tags: [Linux, Network, NIC, Bridge, Namespace, veth, RoutingTable, iproute2, tcpdump, Kubernetes, CNI]
published: true
---

## 1. Linux 네트워크를 먼저 알아야 하는 이유

Kubernetes 네트워크는 별도의 마법 같은 기술이 아니라 Linux 네트워크 기능 위에 만들어집니다.

Pod가 IP를 가지고 통신하는 것, Service가 Pod로 트래픽을 전달하는 것, CNI가 Pod 네트워크를 구성하는 것, kube-proxy가 Service NAT를 처리하는 것, Cilium이 eBPF로 패킷을 처리하는 것 모두 Linux 네트워크 기능과 연결되어 있습니다.

이 문서에서 다루는 핵심 주제는 다음과 같습니다.

- NIC
- Network Interface
- MAC Address
- IP Address
- Socket
- Routing Table
- ARP / Neighbor
- Linux Bridge
- Network Namespace
- veth Pair
- MTU
- iproute2 명령어
- ss
- tcpdump
- iptables / nftables
- eBPF
- Kubernetes CNI와의 연결
- Linux 네트워크 장애 분석 순서

공식 참고자료:

- iproute2 `ip` man page: https://man7.org/linux/man-pages/man8/ip.8.html
- ip-address man page: https://man7.org/linux/man-pages/man8/ip-address.8.html
- ip-link man page: https://man7.org/linux/man-pages/man8/ip-link.8.html
- ip-route man page: https://man7.org/linux/man-pages/man8/ip-route.8.html
- ip-netns man page: https://man7.org/linux/man-pages/man8/ip-netns.8.html
- network_namespaces man page: https://man7.org/linux/man-pages/man7/network_namespaces.7.html
- ss man page: https://man7.org/linux/man-pages/man8/ss.8.html
- tcpdump man page: https://www.tcpdump.org/manpages/tcpdump.1.html
- bridge man page: https://man7.org/linux/man-pages/man8/bridge.8.html
- Linux Kernel Networking Documentation: https://docs.kernel.org/networking/index.html
- Kubernetes Networking: https://kubernetes.io/docs/concepts/cluster-administration/networking/

----

## 2. Linux Network Stack 전체 구조

Linux에서 애플리케이션이 네트워크로 데이터를 보낼 때는 대략 다음 흐름을 거칩니다.

```text
Application
  ↓
Socket
  ↓
TCP / UDP
  ↓
IP
  ↓
Routing Table
  ↓
ARP / Neighbor
  ↓
Ethernet Frame
  ↓
Network Interface
  ↓
NIC Driver
  ↓
NIC
  ↓
Switch / Router / Network
```

예를 들어 `curl https://example.com`을 실행하면 단순히 URL에 접속하는 것처럼 보이지만 내부적으로는 다음 과정이 일어납니다.

```text
curl 프로세스
  ↓
Socket 생성
  ↓
DNS 조회
  ↓
TCP 연결 생성
  ↓
TLS Handshake
  ↓
HTTP Request 전송
  ↓
Routing Table 조회
  ↓
Gateway MAC 주소 확인
  ↓
NIC로 Ethernet Frame 전송
```

Kubernetes Pod도 마찬가지입니다.

```text
Pod 내부 애플리케이션
  ↓
Pod Network Namespace의 Socket
  ↓
Pod eth0
  ↓
veth pair
  ↓
Host Network Namespace
  ↓
CNI / Routing / iptables / eBPF
  ↓
Node NIC
  ↓
외부 네트워크
```

따라서 Kubernetes 네트워크 장애를 분석하려면 Linux의 기본 네트워크 구성요소를 이해해야 합니다.

----

## 3. OSI 계층과 Linux 네트워크 관점

네트워크를 설명할 때 OSI 7 Layer를 자주 사용합니다.

| 계층 | 이름 | 예시 |
|---|---|---|
| L7 | Application | HTTP, DNS, gRPC |
| L6 | Presentation | TLS, Encoding |
| L5 | Session | Session 관리 개념 |
| L4 | Transport | TCP, UDP |
| L3 | Network | IP, ICMP |
| L2 | Data Link | Ethernet, MAC, VLAN, Bridge |
| L1 | Physical | Cable, NIC, Port |

Linux 네트워크를 실무적으로 볼 때는 다음 구조가 더 자주 사용됩니다.

```text
Application Layer : HTTP, DNS, SSH, PostgreSQL, Redis
Transport Layer   : TCP, UDP
Network Layer     : IP, Routing, ICMP
Link Layer        : Ethernet, MAC, ARP, Bridge, VLAN
Physical Layer    : NIC, Cable, Switch Port
```

Kubernetes 네트워크 문제도 결국 이 계층 중 어딘가에서 발생합니다.

예시:

| 증상 | 의심 계층 |
|---|---|
| DNS 조회 실패 | L7 / DNS / CoreDNS |
| TCP 연결 실패 | L4 / 방화벽 / 라우팅 |
| Ping 실패 | L3 / Routing / ICMP |
| 같은 대역인데 통신 안 됨 | L2 / ARP / Bridge / VLAN |
| 패킷이 조각나거나 끊김 | MTU / Overlay |
| Service는 안 되지만 Pod IP는 됨 | kube-proxy / iptables / eBPF |

----

## 4. Network Interface란 무엇인가

Network Interface는 Linux가 네트워크 패킷을 주고받기 위해 사용하는 장치입니다.

물리 장치일 수도 있고, 가상 장치일 수도 있습니다.

```bash
ip link
```

출력 예시:

```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
4: veth9a1b2c@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master cni0 state UP mode DEFAULT group default
```

대표적인 Interface 종류는 다음과 같습니다.

| Interface | 설명 |
|---|---|
| `lo` | Loopback 인터페이스. 자기 자신과 통신할 때 사용 |
| `eth0`, `ens3`, `enp1s0` | 물리 NIC 또는 VM의 가상 NIC |
| `docker0` | Docker 기본 bridge |
| `cni0` | 일부 CNI가 사용하는 Linux bridge |
| `vethxxx` | veth pair의 한쪽 끝 |
| `br0` | 사용자가 만든 Linux bridge |
| `bond0` | 여러 NIC를 묶은 Bonding 인터페이스 |
| `vlan100` | VLAN 인터페이스 |
| `tun0`, `tap0` | VPN, 터널링, 가상 네트워크 장치 |
| `vxlan.calico`, `flannel.1` | Overlay Network용 가상 인터페이스 |

Network Interface는 단순히 장치 이름만 있는 것이 아니라 다음 정보를 가집니다.

- Interface name
- MAC address
- MTU
- State
- Queue discipline
- Master bridge 여부
- IP address
- RX/TX statistics

----

## 5. `ip link` 출력 읽는 법

`ip link`는 인터페이스의 L2 상태를 확인하는 명령어입니다.

```bash
ip link show ens3
```

예시:

```text
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether fa:16:3e:aa:bb:cc brd ff:ff:ff:ff:ff:ff
```

중요 항목:

| 항목 | 의미 |
|---|---|
| `2:` | Interface index |
| `ens3` | Interface name |
| `BROADCAST` | Broadcast 지원 |
| `MULTICAST` | Multicast 지원 |
| `UP` | 관리자가 인터페이스를 활성화함 |
| `LOWER_UP` | 실제 Link가 살아 있음 |
| `mtu 1500` | 최대 전송 단위 |
| `qdisc` | Queue discipline |
| `state UP` | 인터페이스 상태 |
| `link/ether` | MAC address |
| `brd` | Broadcast MAC address |

`UP`과 `LOWER_UP`은 다릅니다.

```text
UP       : Linux에서 인터페이스를 활성화한 상태
LOWER_UP : 실제 물리/가상 링크가 연결된 상태
```

예를 들어 케이블이 빠진 물리 NIC는 `UP`이어도 `LOWER_UP`이 없을 수 있습니다.

----

## 6. Network Interface 종류

### 6-1) Loopback Interface

`lo`는 자기 자신에게 접근할 때 사용하는 인터페이스입니다.

```bash
ip addr show lo
```

대표 주소:

```text
127.0.0.1/8
::1/128
```

애플리케이션이 `localhost`에 바인딩되면 외부에서는 접근할 수 없습니다.

```text
127.0.0.1:8080 → 같은 Network Namespace 안에서만 접근 가능
0.0.0.0:8080   → 모든 인터페이스에서 접근 가능
```

Kubernetes Pod에서도 이 개념은 중요합니다.

같은 Pod 안의 컨테이너들은 같은 Network Namespace를 공유합니다. 그래서 한 컨테이너가 `localhost:8080`으로 뜨면 같은 Pod 안의 다른 컨테이너가 접근할 수 있습니다.

### 6-2) Physical / Virtual NIC

`eth0`, `ens3`, `enp1s0` 같은 인터페이스는 물리 NIC 또는 VM의 가상 NIC입니다.

VM에서는 실제 물리 장치가 아니라 Hypervisor가 제공하는 가상 NIC일 수 있습니다.

예:

```text
OpenStack VM eth0
  ↓
Virtio NIC
  ↓
Hypervisor vSwitch
  ↓
Physical NIC
```

### 6-3) Bridge Interface

Bridge Interface는 Linux Bridge 자체를 나타내는 인터페이스입니다.

예:

```text
br0
cni0
docker0
```

Bridge는 여러 Port를 연결하는 소프트웨어 스위치 역할을 합니다.

### 6-4) veth Interface

veth는 Virtual Ethernet의 약자입니다.

두 개가 한 쌍으로 생성되며, 한쪽으로 들어간 패킷은 다른 쪽으로 나옵니다.

```text
vethA <------> vethB
```

Container와 Host Network Namespace를 연결할 때 자주 사용됩니다.

### 6-5) VLAN Interface

VLAN Interface는 하나의 물리 NIC 위에 VLAN Tag를 붙여 논리 네트워크를 나누는 방식입니다.

```bash
ip link add link ens3 name ens3.100 type vlan id 100
ip link set ens3.100 up
```

```text
ens3
 └─ ens3.100  VLAN 100
```

### 6-6) Bond Interface

Bonding은 여러 NIC를 하나로 묶는 기능입니다.

용도:

- 대역폭 증가
- 장애 시 Failover
- 링크 이중화

예:

```text
ens1 + ens2 → bond0
```

### 6-7) Tunnel Interface

`tun0`, `tap0`, `vxlan0` 같은 인터페이스는 터널링에 사용됩니다.

Kubernetes Overlay Network에서도 터널 인터페이스가 자주 등장합니다.

예:

- Flannel VXLAN: `flannel.1`
- Calico VXLAN: `vxlan.calico`
- WireGuard: `wg0`

----

## 7. NIC란 무엇인가

NIC는 Network Interface Card의 약자입니다.

물리 서버에서는 실제 랜카드이고, VM에서는 Hypervisor가 제공하는 가상 NIC입니다.

NIC는 단순히 케이블 꽂는 장치가 아니라 다음 요소와 연결됩니다.

- PCI Device
- Kernel Driver
- RX Queue
- TX Queue
- Interrupt
- Offloading
- MTU
- Link Speed
- Duplex
- MAC Address

확인 명령어:

```bash
ip link show ens3
ip addr show ens3
ip -s link show ens3
ethtool ens3
ethtool -i ens3
ethtool -k ens3
```

### 7-1) NIC Driver

NIC는 커널 드라이버를 통해 동작합니다.

```bash
ethtool -i ens3
```

예시:

```text
driver: virtio_net
version: 1.0.0
bus-info: virtio0
```

OpenStack, KVM, QEMU 환경에서는 `virtio_net` 같은 가상 NIC 드라이버를 많이 볼 수 있습니다.

### 7-2) Speed / Duplex

```bash
ethtool ens3
```

확인 항목:

```text
Speed: 10000Mb/s
Duplex: Full
Link detected: yes
```

장애 분석 시 다음을 봅니다.

- Link detected가 `yes`인지
- Speed가 예상보다 낮지 않은지
- Duplex mismatch가 없는지
- RX/TX error가 증가하는지

### 7-3) RX/TX Queue

NIC는 수신 패킷과 송신 패킷을 Queue로 처리합니다.

```text
Network → NIC RX Queue → Kernel → Application
Application → Kernel → NIC TX Queue → Network
```

고성능 환경에서는 Queue 수, RSS, Interrupt 분산이 성능에 큰 영향을 줍니다.

### 7-4) Offloading

NIC는 일부 네트워크 처리를 하드웨어에서 대신 수행할 수 있습니다.

```bash
ethtool -k ens3
```

대표 기능:

| 기능 | 설명 |
|---|---|
| checksum offload | 체크섬 계산을 NIC가 처리 |
| TSO | 큰 TCP 데이터를 NIC가 Segment로 분할 |
| GRO | 수신 Segment를 커널에서 묶어서 처리 |
| GSO | 송신 Segment 처리를 최적화 |

패킷 캡처에서 checksum 오류처럼 보이는 경우가 있는데, 실제 오류가 아니라 offload 때문에 그렇게 보일 수 있습니다.

----

## 8. MAC Address

MAC Address는 L2 계층에서 사용하는 주소입니다.

예:

```text
fa:16:3e:aa:bb:cc
```

확인:

```bash
ip link show ens3
```

MAC 주소는 같은 L2 네트워크 안에서 장비를 식별하는 데 사용됩니다.

IP 통신을 하더라도 실제 Ethernet Frame은 MAC 주소를 사용해서 전달됩니다.

```text
IP Packet
  Source IP      : 192.168.10.25
  Destination IP : 192.168.10.1

Ethernet Frame
  Source MAC      : fa:16:3e:aa:bb:cc
  Destination MAC : aa:bb:cc:dd:ee:ff
```

### 8-1) Unicast, Broadcast, Multicast

| 종류 | 설명 | 예시 |
|---|---|---|
| Unicast | 특정 한 장비로 전송 | 특정 MAC |
| Broadcast | 같은 L2 네트워크 전체로 전송 | ff:ff:ff:ff:ff:ff |
| Multicast | 특정 그룹으로 전송 | 01:00:5e:xx:xx:xx |

ARP 요청은 Broadcast로 나갑니다.

```text
Who has 192.168.10.1? Tell 192.168.10.25
```

----

## 9. IP Address와 CIDR

IP Address는 L3 계층 주소입니다.

확인:

```bash
ip addr
ip addr show ens3
```

예시:

```text
inet 192.168.10.25/24 brd 192.168.10.255 scope global ens3
```

이 값은 다음 의미를 가집니다.

| 항목 | 의미 |
|---|---|
| `192.168.10.25` | Interface에 할당된 IP |
| `/24` | Prefix length |
| `192.168.10.255` | Broadcast 주소 |
| `scope global` | 전역적으로 사용 가능한 주소 |
| `ens3` | IP가 붙은 인터페이스 |

### 9-1) CIDR

CIDR는 IP 주소와 네트워크 범위를 표현하는 방식입니다.

```text
192.168.10.25/24
```

`/24`는 앞의 24비트가 네트워크 주소라는 뜻입니다.

```text
Network   : 192.168.10.0
Host 범위 : 192.168.10.1 ~ 192.168.10.254
Broadcast : 192.168.10.255
Netmask   : 255.255.255.0
```

### 9-2) Private IP

사설 IP 대역:

| 대역 | 범위 |
|---|---|
| 10.0.0.0/8 | 10.0.0.0 ~ 10.255.255.255 |
| 172.16.0.0/12 | 172.16.0.0 ~ 172.31.255.255 |
| 192.168.0.0/16 | 192.168.0.0 ~ 192.168.255.255 |

Kubernetes Pod CIDR, Service CIDR, Node 내부망은 대부분 사설 IP 대역을 사용합니다.

### 9-3) Link-local Address

`169.254.0.0/16`은 IPv4 Link-local 대역입니다.

Cloud 환경에서는 Metadata Service 접근에 자주 사용됩니다.

```text
169.254.169.254
```

----

## 10. Socket

Socket은 애플리케이션이 네트워크 통신을 하기 위해 사용하는 커널 객체입니다.

TCP 연결 하나는 보통 다음 4가지 값으로 식별됩니다.

```text
Source IP
Source Port
Destination IP
Destination Port
```

예:

```text
192.168.10.25:53124 → 93.184.216.34:443
```

서버가 포트를 열고 대기하는 상태는 `LISTEN`입니다.

```bash
ss -lntp
```

예시:

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      4096   0.0.0.0:80        0.0.0.0:*     users:("nginx",pid=1234)
```

중요한 구분:

```text
127.0.0.1:8080  → 같은 Network Namespace 내부에서만 접근 가능
0.0.0.0:8080    → 모든 IPv4 Interface에서 접근 가능
192.168.10.25:8080 → 특정 IP로만 접근 가능
```

Kubernetes에서 애플리케이션이 `127.0.0.1`에만 바인딩되어 있으면 Service나 다른 Pod에서 접근할 수 없는 문제가 발생할 수 있습니다.

----

## 11. Routing Table

Routing Table은 목적지 IP로 패킷을 보낼 때 어느 경로를 사용할지 결정하는 표입니다.

확인:

```bash
ip route
```

예시:

```text
default via 192.168.10.1 dev ens3
10.244.0.0/16 via 192.168.10.83 dev ens3
192.168.10.0/24 dev ens3 proto kernel scope link src 192.168.10.25
```

각 항목의 의미:

| 항목 | 의미 |
|---|---|
| `default` | 일치하는 경로가 없을 때 사용하는 기본 경로 |
| `via` | 다음 Hop Gateway |
| `dev` | 패킷을 내보낼 Interface |
| `proto kernel` | 커널이 자동 생성한 경로 |
| `scope link` | 같은 L2 링크에 직접 연결된 대역 |
| `src` | 출발지 IP로 사용할 주소 |

### 11-1) Routing 결정 과정

Linux는 패킷을 보낼 때 다음 순서로 판단합니다.

```text
목적지 IP 확인
  ↓
Routing Table 조회
  ↓
Longest Prefix Match
  ↓
출구 Interface 결정
  ↓
Gateway 필요 여부 판단
  ↓
ARP / Neighbor 조회
  ↓
패킷 전송
```

### 11-2) Longest Prefix Match

여러 Route가 있을 때 더 구체적인 Prefix가 우선합니다.

예:

```text
default via 192.168.10.1 dev ens3
10.0.0.0/8 via 192.168.10.254 dev ens3
10.244.0.0/16 via 192.168.10.83 dev ens3
10.244.1.0/24 dev cni0
```

목적지가 `10.244.1.20`이면 가장 구체적인 `/24` 경로가 선택됩니다.

```text
10.244.1.20 → 10.244.1.0/24 경로 선택
```

### 11-3) `ip route get`

`ip route get`은 특정 목적지로 갈 때 실제 선택되는 경로를 보여줍니다.

```bash
ip route get 8.8.8.8
```

예시:

```text
8.8.8.8 via 192.168.10.1 dev ens3 src 192.168.10.25 uid 0
```

이 명령어는 장애 분석에서 매우 중요합니다.

```bash
ip route get <pod-ip>
ip route get <service-ip>
ip route get <external-ip>
```

----

## 12. Policy Routing과 `ip rule`

일반적으로 Linux는 main routing table을 사용합니다.

하지만 조건에 따라 다른 Routing Table을 사용할 수도 있습니다. 이것을 Policy Routing이라고 합니다.

확인:

```bash
ip rule
```

예시:

```text
0:      from all lookup local
32766:  from all lookup main
32767:  from all lookup default
```

특정 Source IP에서 나가는 트래픽만 다른 Routing Table을 사용하게 할 수도 있습니다.

```bash
ip rule add from 10.10.10.10/32 table 100
ip route add default via 192.168.20.1 dev ens4 table 100
```

Kubernetes CNI, Multus, 다중 NIC 환경에서는 Policy Routing이 중요해질 수 있습니다.

----

## 13. ARP와 Neighbor

같은 L2 네트워크 안에서 IP로 통신하려면 상대방의 MAC Address를 알아야 합니다.

IPv4에서는 이 역할을 ARP가 합니다.

```text
192.168.10.25가 192.168.10.1로 패킷을 보내고 싶음
  ↓
192.168.10.1의 MAC 주소를 모름
  ↓
ARP Broadcast 전송
  ↓
192.168.10.1이 자신의 MAC 주소를 응답
  ↓
Neighbor Table에 저장
  ↓
Ethernet Frame 전송
```

확인:

```bash
ip neigh
```

예시:

```text
192.168.10.1 dev ens3 lladdr aa:bb:cc:dd:ee:ff REACHABLE
192.168.10.50 dev ens3 lladdr fa:16:3e:11:22:33 STALE
```

상태 의미:

| 상태 | 의미 |
|---|---|
| REACHABLE | 최근 통신 성공 |
| STALE | 오래되었지만 사용 가능할 수 있음 |
| DELAY | 확인 대기 중 |
| PROBE | 재확인 중 |
| FAILED | MAC 주소 확인 실패 |
| INCOMPLETE | ARP 응답 대기 중 |

`ip neigh`에서 `FAILED`가 보이면 L2, ARP, VLAN, Gateway, Bridge 문제를 의심해야 합니다.

----

## 14. Linux Bridge

Linux Bridge는 Linux 커널이 제공하는 소프트웨어 스위치입니다.

여러 인터페이스를 같은 L2 네트워크에 연결합니다.

```text
          Linux Bridge br0
        ┌───────────────┐
veth1 ──┤               ├── ens3
veth2 ──┤               │
veth3 ──┤               │
        └───────────────┘
```

Bridge에 연결된 인터페이스는 Bridge Port가 됩니다.

```text
veth1, veth2, ens3 → br0의 port
```

확인:

```bash
ip link show type bridge
bridge link
bridge fdb show
```

### 14-1) Bridge는 L2 Switch처럼 동작한다

Bridge는 MAC 주소를 보고 Frame을 전달합니다.

동작 방식:

```text
Frame 수신
  ↓
Source MAC 학습
  ↓
Destination MAC 확인
  ↓
FDB 조회
  ↓
알면 해당 Port로 전달
  ↓
모르면 Flooding
```

### 14-2) FDB

FDB는 Forwarding Database의 약자입니다.

Bridge가 어떤 MAC 주소가 어느 Port에 있는지 기억하는 테이블입니다.

```bash
bridge fdb show
```

예시:

```text
fa:16:3e:aa:bb:cc dev veth1 master br0
fa:16:3e:11:22:33 dev veth2 master br0
```

### 14-3) Flooding

Bridge가 Destination MAC을 모르면 모든 Port로 Frame을 뿌립니다.

```text
Unknown Unicast
  ↓
Flooding
  ↓
대상 장비가 응답
  ↓
Bridge가 MAC 위치 학습
```

Broadcast도 여러 Port로 전달됩니다.

ARP 요청이 대표적인 Broadcast입니다.

### 14-4) Bridge와 Kubernetes

일부 CNI는 Linux Bridge를 사용해 Pod를 연결합니다.

```text
Pod eth0
  ↓
veth pair
  ↓
cni0 bridge
  ↓
Host routing / NAT
  ↓
Node NIC
```

Docker 기본 네트워크의 `docker0`도 Linux Bridge입니다.

----

## 15. Network Namespace

Network Namespace는 Linux 네트워크 스택을 격리하는 기능입니다.

Namespace마다 독립적인 다음 요소를 가질 수 있습니다.

- Network Interface
- IP Address
- Routing Table
- ARP / Neighbor Table
- iptables / nftables rule
- Socket
- Port number space
- `/proc/net`

즉, Network Namespace가 다르면 같은 포트 번호를 동시에 사용할 수 있습니다.

```text
Namespace A: 0.0.0.0:8080 LISTEN
Namespace B: 0.0.0.0:8080 LISTEN
```

서로 다른 Namespace이므로 충돌하지 않습니다.

### 15-1) Namespace 종류

Linux Namespace에는 여러 종류가 있습니다.

| Namespace | 격리 대상 |
|---|---|
| Network | 인터페이스, IP, 라우팅, 포트 |
| PID | 프로세스 ID 공간 |
| Mount | 파일시스템 마운트 |
| UTS | hostname, domain name |
| IPC | IPC 리소스 |
| User | UID/GID |
| Cgroup | cgroup view |
| Time | 시간 namespace |

Container는 여러 Namespace를 조합해서 격리됩니다.

### 15-2) Pod와 Network Namespace

Kubernetes에서 Pod는 하나의 Network Namespace를 가집니다.

같은 Pod 안의 컨테이너들은 같은 Network Namespace를 공유합니다.

```text
Pod
 ├─ Container A
 ├─ Container B
 └─ pause container

세 컨테이너가 같은 Network Namespace 공유
```

그래서 같은 Pod 안에서는 `localhost`로 서로 접근할 수 있습니다.

```text
Container A: localhost:8080
Container B: curl localhost:8080 가능
```

반대로 다른 Pod는 다른 Network Namespace를 가지므로 `localhost`로 접근할 수 없습니다.

### 15-3) Namespace 확인

```bash
ip netns list
lsns -t net
```

컨테이너 런타임이 만든 Namespace는 `ip netns list`에 바로 보이지 않을 수 있습니다.

이 경우 프로세스 기준으로 확인합니다.

```bash
lsns -t net
readlink /proc/<pid>/ns/net
```

특정 프로세스의 Network Namespace 안에서 명령 실행:

```bash
nsenter -t <pid> -n ip addr
nsenter -t <pid> -n ip route
nsenter -t <pid> -n ss -lntp
```

----

## 16. veth Pair

veth pair는 두 개의 가상 Ethernet Interface가 한 쌍으로 연결된 구조입니다.

```text
vethA <================> vethB
```

한쪽으로 들어간 패킷은 다른 쪽으로 나옵니다.

Container 네트워크에서 가장 흔한 구조는 다음과 같습니다.

```text
Container Namespace               Host Namespace
┌───────────────────┐             ┌─────────────────────┐
│ eth0              │             │ vethxxxx            │
│ 10.244.1.10       │             │                     │
└───────┬───────────┘             └─────────┬───────────┘
        │                                   │
        └========== veth pair ==============┘
```

Kubernetes에서는 보통 veth 한쪽이 Pod의 `eth0`가 되고, 다른 한쪽이 Host Network Namespace에 남습니다.

```text
Pod eth0 <-> Host vethXXX
```

### 16-1) veth 생성 실습

```bash
sudo ip link add veth-a type veth peer name veth-b
ip link show type veth
```

인터페이스 활성화:

```bash
sudo ip link set veth-a up
sudo ip link set veth-b up
```

IP 할당:

```bash
sudo ip addr add 10.10.10.1/24 dev veth-a
sudo ip addr add 10.10.10.2/24 dev veth-b
```

테스트:

```bash
ping -c 3 10.10.10.2
```

### 16-2) veth를 Namespace에 넣기

```bash
sudo ip netns add ns1
sudo ip link add veth-host type veth peer name veth-ns
sudo ip link set veth-ns netns ns1

sudo ip addr add 10.10.20.1/24 dev veth-host
sudo ip link set veth-host up

sudo ip netns exec ns1 ip addr add 10.10.20.2/24 dev veth-ns
sudo ip netns exec ns1 ip link set veth-ns up
sudo ip netns exec ns1 ip link set lo up
```

통신 확인:

```bash
ping -c 3 10.10.20.2
sudo ip netns exec ns1 ping -c 3 10.10.20.1
```

삭제:

```bash
sudo ip netns del ns1
sudo ip link del veth-host
```

veth pair는 한쪽을 삭제하면 peer도 함께 삭제됩니다.

----

## 17. Pod Packet Flow

Pod에서 외부로 나가는 패킷 흐름을 단순화하면 다음과 같습니다.

```text
Pod Application
  ↓
Socket
  ↓
Pod Network Namespace
  ↓
Pod eth0
  ↓
veth pair
  ↓
Host Network Namespace의 veth
  ↓
CNI 처리
  ↓
Routing Table
  ↓
iptables / eBPF
  ↓
Node NIC
  ↓
Gateway
  ↓
External Network
```

외부에서 Pod로 들어오는 흐름은 반대입니다.

```text
External Network
  ↓
Gateway
  ↓
Node NIC
  ↓
Host Network Namespace
  ↓
Routing / iptables / eBPF
  ↓
Host veth
  ↓
Pod eth0
  ↓
Pod Application
```

CNI 구현 방식에 따라 중간 경로는 달라질 수 있습니다.

예:

| CNI 방식 | 특징 |
|---|---|
| Bridge 기반 | Pod veth를 bridge에 연결 |
| Routing 기반 | Pod CIDR를 Node 간 routing으로 전달 |
| Overlay 기반 | VXLAN, Geneve 등으로 캡슐화 |
| eBPF 기반 | iptables 대신 eBPF 프로그램으로 처리 |

----

## 18. MTU

MTU는 Maximum Transmission Unit의 약자입니다.

하나의 네트워크 Frame 또는 Packet이 한 번에 담을 수 있는 최대 크기입니다.

일반 Ethernet MTU:

```text
1500 bytes
```

확인:

```bash
ip link show ens3
```

변경:

```bash
sudo ip link set ens3 mtu 9000
```

### 18-1) MTU가 중요한 이유

Overlay Network에서는 원래 패킷 위에 추가 헤더가 붙습니다.

예:

```text
Original Packet
  + VXLAN Header
  + UDP Header
  + Outer IP Header
  + Ethernet Header
```

따라서 물리 NIC MTU가 1500인데 Overlay 헤더가 추가되면 실제 Pod에서 사용할 수 있는 MTU는 더 작아져야 합니다.

예시:

```text
Node NIC MTU: 1500
Overlay overhead: 50
Pod MTU: 1450
```

MTU가 맞지 않으면 다음 증상이 발생할 수 있습니다.

- 작은 ping은 되는데 큰 응답은 실패
- 특정 API 호출만 timeout
- TLS Handshake 실패
- 이미지 pull 중 끊김
- gRPC stream 불안정
- Pod 간 통신 일부 실패

### 18-2) MTU 테스트

Linux에서 DF 비트를 설정하고 ping 테스트:

```bash
ping -M do -s 1472 8.8.8.8
```

IPv4 ICMP 헤더와 IP 헤더를 고려하면 다음과 같습니다.

```text
Payload 1472 + ICMP 8 + IP 20 = 1500
```

MTU 1500 경로에서 이 값이 통과해야 합니다.

Overlay 환경에서는 더 작은 값으로 테스트해야 할 수 있습니다.

----

## 19. iproute2 명령어 개요

예전에는 `ifconfig`, `route`, `netstat`, `brctl`을 많이 사용했습니다.

현재 Linux에서는 `iproute2` 도구를 사용하는 것이 일반적입니다.

대표 명령어:

| 명령어 | 용도 |
|---|---|
| `ip addr` | IP 주소 확인/설정 |
| `ip link` | Interface 확인/제어 |
| `ip route` | Routing Table 확인/설정 |
| `ip rule` | Policy Routing 확인/설정 |
| `ip neigh` | ARP/Neighbor 확인 |
| `ip netns` | Network Namespace 관리 |
| `ip monitor` | 네트워크 이벤트 모니터링 |
| `ip -s link` | Interface 통계 확인 |
| `ip -d link` | Interface 상세 정보 확인 |
| `bridge` | Bridge/FDB/VLAN 확인 |
| `ss` | Socket 상태 확인 |

----

## 20. `ip addr`

IP 주소를 확인하고 설정하는 명령어입니다.

```bash
ip addr
ip addr show ens3
ip -br addr
```

`-br` 옵션은 간결한 출력입니다.

```bash
ip -br addr
```

예시:

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
ens3             UP             192.168.10.25/24
```

IP 추가:

```bash
sudo ip addr add 192.168.10.50/24 dev ens3
```

IP 삭제:

```bash
sudo ip addr del 192.168.10.50/24 dev ens3
```

주의:

`ip addr add`로 추가한 IP는 재부팅 후 사라질 수 있습니다. 영구 설정은 배포판별 네트워크 설정 파일이나 NetworkManager, netplan 등을 사용해야 합니다.

----

## 21. `ip link`

Interface 상태를 확인하고 제어하는 명령어입니다.

```bash
ip link
ip link show ens3
ip -d link show ens3
ip -s link show ens3
```

Interface up/down:

```bash
sudo ip link set ens3 up
sudo ip link set ens3 down
```

MTU 변경:

```bash
sudo ip link set ens3 mtu 9000
```

이름 변경:

```bash
sudo ip link set ens3 name net0
```

veth 생성:

```bash
sudo ip link add veth-a type veth peer name veth-b
```

bridge 생성:

```bash
sudo ip link add br0 type bridge
sudo ip link set br0 up
```

bridge에 interface 연결:

```bash
sudo ip link set veth-a master br0
```

----

## 22. `ip route`

Routing Table을 확인하고 수정하는 명령어입니다.

```bash
ip route
ip route show
ip route get 8.8.8.8
```

Route 추가:

```bash
sudo ip route add 10.10.0.0/16 via 192.168.10.1
```

Route 삭제:

```bash
sudo ip route del 10.10.0.0/16
```

Default Gateway 추가:

```bash
sudo ip route add default via 192.168.10.1 dev ens3
```

특정 Interface로 직접 연결된 대역 추가:

```bash
sudo ip route add 10.244.1.0/24 dev cni0
```

장애 분석에서 가장 자주 쓰는 명령어는 다음입니다.

```bash
ip route get <destination-ip>
```

이 명령어는 실제로 어떤 Interface와 Source IP가 선택되는지 보여줍니다.

----

## 23. `ip neigh`

ARP/Neighbor Table을 확인합니다.

```bash
ip neigh
ip neigh show dev ens3
```

Neighbor 삭제:

```bash
sudo ip neigh del 192.168.10.1 dev ens3
```

강제로 Neighbor 추가:

```bash
sudo ip neigh add 192.168.10.1 lladdr aa:bb:cc:dd:ee:ff dev ens3
```

일반적으로 수동 추가는 자주 하지 않습니다. 장애 분석에서는 `FAILED`, `INCOMPLETE` 상태를 확인하는 용도로 많이 사용합니다.

----

## 24. `ip netns`

Network Namespace를 관리합니다.

Namespace 생성:

```bash
sudo ip netns add ns1
```

목록 확인:

```bash
ip netns list
```

Namespace 안에서 명령 실행:

```bash
sudo ip netns exec ns1 ip addr
sudo ip netns exec ns1 ip route
sudo ip netns exec ns1 ping 8.8.8.8
```

Namespace 삭제:

```bash
sudo ip netns del ns1
```

주의:

Kubernetes Pod의 Network Namespace는 보통 `/var/run/netns`에 직접 이름이 잡히지 않을 수 있습니다. 이 경우 `crictl`, `ctr`, `nsenter`, `/proc/<pid>/ns/net`을 활용합니다.

----

## 25. `ip monitor`

네트워크 상태 변화를 실시간으로 볼 수 있습니다.

```bash
ip monitor
ip monitor link
ip monitor address
ip monitor route
```

Interface가 up/down되거나 IP가 추가/삭제되거나 Route가 바뀌는 상황을 실시간으로 확인할 때 유용합니다.

----

## 26. `bridge` 명령어

Bridge 상태를 확인할 때 사용합니다.

```bash
bridge link
bridge fdb show
bridge vlan show
```

Bridge Port 확인:

```bash
bridge link
```

FDB 확인:

```bash
bridge fdb show
```

VLAN 정보 확인:

```bash
bridge vlan show
```

Docker나 CNI Bridge 문제를 볼 때 유용합니다.

----

## 27. `ss` 명령어

`ss`는 Socket 상태를 확인하는 명령어입니다.

자주 쓰는 명령어:

```bash
ss -lntp
ss -antp
ss -lunp
ss -s
```

| 명령어 | 의미 |
|---|---|
| `ss -lntp` | TCP LISTEN 포트 확인 |
| `ss -antp` | 모든 TCP 연결 확인 |
| `ss -lunp` | UDP LISTEN 포트 확인 |
| `ss -s` | Socket 통계 요약 |

### 27-1) TCP 상태

| 상태 | 의미 |
|---|---|
| LISTEN | 서버가 연결 대기 중 |
| SYN-SENT | 클라이언트가 SYN을 보낸 상태 |
| SYN-RECV | 서버가 SYN을 받고 SYN/ACK를 보낸 상태 |
| ESTABLISHED | 연결 성립 |
| FIN-WAIT-1 | 연결 종료 시작 |
| FIN-WAIT-2 | 종료 응답 대기 |
| TIME-WAIT | 종료 후 일정 시간 대기 |
| CLOSE-WAIT | 상대가 종료했지만 애플리케이션이 아직 닫지 않음 |
| LAST-ACK | 마지막 ACK 대기 |

장애 예시:

| 증상 | 볼 상태 |
|---|---|
| 연결이 안 됨 | SYN-SENT 증가 |
| 서버 accept 문제 | SYN-RECV 증가 |
| 애플리케이션이 연결을 안 닫음 | CLOSE-WAIT 증가 |
| 짧은 연결이 너무 많음 | TIME-WAIT 증가 |

----

## 28. tcpdump

`tcpdump`는 패킷을 캡처하는 도구입니다.

기본 사용:

```bash
sudo tcpdump -i any
```

특정 IP:

```bash
sudo tcpdump -i any host 192.168.10.50
```

특정 Port:

```bash
sudo tcpdump -i any port 443
```

특정 Interface:

```bash
sudo tcpdump -i ens3 host 8.8.8.8
```

패킷 저장:

```bash
sudo tcpdump -i any -w capture.pcap
```

파일 읽기:

```bash
tcpdump -r capture.pcap
```

### 28-1) tcpdump로 확인할 수 있는 것

- SYN이 나가는지
- SYN/ACK가 돌아오는지
- DNS Query가 나가는지
- ARP Request가 나가는지
- ICMP Fragmentation Needed가 오는지
- 특정 Interface까지 패킷이 도달하는지
- Pod veth에서 패킷이 보이는지
- Node NIC에서 패킷이 보이는지

### 28-2) 어느 Interface에서 캡처해야 하는가

Kubernetes 장애 분석에서는 한 곳만 보면 안 됩니다.

예:

```text
Pod eth0
Host veth
cni0 또는 CNI Interface
Node NIC
```

각 지점에서 tcpdump를 떠야 패킷이 어디서 사라지는지 알 수 있습니다.

----

## 29. DNS 기본 확인

DNS는 애플리케이션 계층에 가깝지만 네트워크 장애 분석에서 자주 확인합니다.

확인 파일:

```bash
cat /etc/resolv.conf
```

조회 명령:

```bash
nslookup example.com
dig example.com
host example.com
```

Kubernetes Pod 안에서는 `/etc/resolv.conf`가 보통 Cluster DNS를 가리킵니다.

예:

```text
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Pod에서 외부 도메인이 안 되면 다음을 봅니다.

```bash
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes.default
kubectl exec -it <pod> -- nslookup google.com
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

----

## 30. iptables와 nftables

Linux에서 패킷 필터링과 NAT를 처리하는 대표 기능이 iptables와 nftables입니다.

iptables는 오래전부터 사용되어 왔고, nftables는 더 현대적인 대체 체계입니다.

Kubernetes에서는 kube-proxy가 iptables 또는 IPVS를 사용해 Service 트래픽을 처리할 수 있습니다.

확인:

```bash
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v
sudo iptables-save
```

nftables 확인:

```bash
sudo nft list ruleset
```

### 30-1) NAT

NAT는 IP 또는 Port를 변환하는 기능입니다.

Kubernetes Service는 보통 DNAT를 사용해 Service IP를 Pod IP로 바꿉니다.

```text
Client → Service IP:80
  ↓ DNAT
Client → Pod IP:8080
```

Pod가 외부로 나갈 때는 SNAT 또는 Masquerade가 사용될 수 있습니다.

```text
Pod IP → Node IP로 Source 변환
```

### 30-2) iptables Chain

대표 Chain:

| Table | Chain | 용도 |
|---|---|---|
| filter | INPUT | 로컬로 들어오는 패킷 |
| filter | OUTPUT | 로컬에서 나가는 패킷 |
| filter | FORWARD | 라우팅되어 지나가는 패킷 |
| nat | PREROUTING | 라우팅 전 NAT |
| nat | OUTPUT | 로컬 생성 패킷 NAT |
| nat | POSTROUTING | 라우팅 후 NAT |

Kubernetes에서는 `KUBE-SERVICES`, `KUBE-SVC-*`, `KUBE-SEP-*` 같은 Chain을 볼 수 있습니다.

----

## 31. eBPF 개요

eBPF는 Linux 커널 안에서 안전하게 실행되는 프로그램을 통해 패킷 처리, 관측, 보안 정책 등을 수행하는 기술입니다.

전통적인 Kubernetes 네트워크 처리:

```text
Packet
  ↓
iptables rule chain
  ↓
NAT / Filter
```

Cilium 같은 eBPF 기반 CNI:

```text
Packet
  ↓
eBPF program
  ↓
Policy / Load Balancing / NAT
```

장점:

- 커널 레벨에서 빠른 처리
- 세밀한 관측 가능
- iptables Chain 폭증 문제 완화
- Service Load Balancing 최적화 가능
- Network Policy 구현 가능

주의:

이 문서는 eBPF 자체를 깊게 다루는 문서가 아닙니다. 여기서는 Linux 네트워크 흐름 안에서 eBPF가 어디에 위치할 수 있는지만 이해하면 됩니다.

----

## 32. Kubernetes CNI와 Linux 네트워크

CNI는 Container Network Interface의 약자입니다.

Kubernetes는 Pod를 만들 때 CNI Plugin을 호출해서 Pod 네트워크를 구성합니다.

CNI가 하는 일은 대략 다음과 같습니다.

```text
Pod Network Namespace 생성 또는 사용
  ↓
veth pair 생성
  ↓
한쪽을 Pod Namespace에 넣음
  ↓
Pod eth0에 IP 할당
  ↓
Host 쪽 veth 설정
  ↓
Route 설정
  ↓
필요 시 Bridge 연결
  ↓
필요 시 iptables/eBPF 설정
```

CNI별 구현 방식은 다릅니다.

| CNI | 특징 |
|---|---|
| Flannel | 단순 Overlay 구성에 자주 사용 |
| Calico | Routing/BGP/Policy 기능 강함 |
| Cilium | eBPF 기반 네트워크/보안/관측 |
| Weave | Overlay 기반 |
| Bridge CNI | 단순 Linux bridge 기반 |

### 32-1) Pod 내부에서 보는 네트워크

```bash
kubectl exec -it <pod> -- ip addr
kubectl exec -it <pod> -- ip route
kubectl exec -it <pod> -- cat /etc/resolv.conf
```

Pod 내부에서는 보통 다음을 봅니다.

```text
lo
eth0
```

Pod의 `eth0`는 실제 물리 NIC가 아니라 veth pair의 한쪽입니다.

### 32-2) Node에서 보는 Pod veth

Node에서는 Pod와 연결된 veth를 볼 수 있습니다.

```bash
ip link show type veth
```

CNI에 따라 이름은 다릅니다.

```text
vethxxxx
calixxxx
lxcxxxx
```

Cilium에서는 `lxc...` 형태의 인터페이스가 보일 수 있습니다.

----

## 33. Kubernetes Service와 Linux Network

Kubernetes Service는 고정된 가상 IP를 제공하고, 실제 Pod로 트래픽을 전달합니다.

```text
Client
  ↓
Service IP:Port
  ↓
kube-proxy 또는 eBPF
  ↓
Pod IP:TargetPort
```

iptables 기반 kube-proxy에서는 다음 흐름이 대표적입니다.

```text
Packet to Service IP
  ↓
nat PREROUTING
  ↓
KUBE-SERVICES
  ↓
KUBE-SVC-xxxxx
  ↓
KUBE-SEP-xxxxx
  ↓
DNAT to Pod IP
```

Cilium eBPF 기반이면 iptables 대신 eBPF에서 Service Load Balancing을 처리할 수 있습니다.

따라서 Service 장애를 볼 때는 다음을 확인합니다.

```bash
kubectl get svc
kubectl get endpoints
kubectl get endpointSlice
kubectl get pods -o wide
ip route
sudo iptables -t nat -L -n -v | grep KUBE
```

Cilium 환경이라면:

```bash
cilium service list
cilium endpoint list
cilium monitor
```

----

## 34. 실습: Namespace + veth + Bridge로 작은 네트워크 만들기

아래 실습은 Linux 네트워크 구조를 이해하기 위한 예제입니다.

목표 구조:

```text
ns1                         host namespace                       ns2
┌─────────────┐             ┌───────────────┐             ┌─────────────┐
│ eth0        │             │ br0           │             │ eth0        │
│ 10.10.0.11  │---veth---→  │ 10.10.0.1     │  ←---veth---│ 10.10.0.12  │
└─────────────┘             └───────────────┘             └─────────────┘
```

### 34-1) Namespace 생성

```bash
sudo ip netns add ns1
sudo ip netns add ns2
```

### 34-2) Bridge 생성

```bash
sudo ip link add br0 type bridge
sudo ip addr add 10.10.0.1/24 dev br0
sudo ip link set br0 up
```

### 34-3) veth pair 생성

```bash
sudo ip link add veth1-host type veth peer name veth1-ns
sudo ip link add veth2-host type veth peer name veth2-ns
```

### 34-4) Namespace에 veth 이동

```bash
sudo ip link set veth1-ns netns ns1
sudo ip link set veth2-ns netns ns2
```

### 34-5) Host 쪽 veth를 Bridge에 연결

```bash
sudo ip link set veth1-host master br0
sudo ip link set veth2-host master br0
sudo ip link set veth1-host up
sudo ip link set veth2-host up
```

### 34-6) Namespace 내부 설정

```bash
sudo ip netns exec ns1 ip addr add 10.10.0.11/24 dev veth1-ns
sudo ip netns exec ns1 ip link set veth1-ns name eth0
sudo ip netns exec ns1 ip link set eth0 up
sudo ip netns exec ns1 ip link set lo up

sudo ip netns exec ns2 ip addr add 10.10.0.12/24 dev veth2-ns
sudo ip netns exec ns2 ip link set veth2-ns name eth0
sudo ip netns exec ns2 ip link set eth0 up
sudo ip netns exec ns2 ip link set lo up
```

### 34-7) 통신 확인

```bash
sudo ip netns exec ns1 ping -c 3 10.10.0.12
sudo ip netns exec ns2 ping -c 3 10.10.0.11
```

### 34-8) tcpdump로 확인

```bash
sudo tcpdump -i br0 icmp
```

다른 터미널에서:

```bash
sudo ip netns exec ns1 ping 10.10.0.12
```

### 34-9) 정리

```bash
sudo ip netns del ns1
sudo ip netns del ns2
sudo ip link del br0
```

이 실습은 Kubernetes Pod 네트워크의 기본 구조와 매우 비슷합니다.

```text
Pod Namespace = ns1/ns2
Pod eth0      = Namespace 안의 veth
Host veth     = Host Namespace에 남은 veth
CNI Bridge    = br0
```

----

## 35. 장애 분석 기본 순서

Linux 네트워크 장애는 무작정 명령어를 치는 것이 아니라 계층별로 좁혀가야 합니다.

기본 순서:

```text
1. Interface가 살아 있는가?
2. IP가 올바르게 붙어 있는가?
3. Routing이 맞는가?
4. Gateway 또는 대상의 MAC을 아는가?
5. Socket이 열려 있는가?
6. 방화벽/NAT가 막고 있는가?
7. DNS가 정상인가?
8. 패킷이 실제로 나가고 들어오는가?
9. MTU 문제는 아닌가?
10. Kubernetes라면 CNI/Service/Policy 문제는 아닌가?
```

### 35-1) Interface 확인

```bash
ip link
ip -br link
ip -s link
```

확인할 것:

- `UP`인가
- `LOWER_UP`인가
- RX/TX error가 증가하는가
- MTU가 예상과 같은가

### 35-2) IP 확인

```bash
ip addr
ip -br addr
```

확인할 것:

- IP가 붙어 있는가
- Prefix가 맞는가
- 중복 IP는 아닌가
- Pod IP와 Node IP 대역이 겹치지 않는가

### 35-3) Route 확인

```bash
ip route
ip route get <destination-ip>
```

확인할 것:

- default route가 있는가
- 목적지로 가는 route가 있는가
- 잘못된 interface로 나가지 않는가
- source IP가 이상하지 않은가

### 35-4) Neighbor 확인

```bash
ip neigh
```

확인할 것:

- Gateway가 `REACHABLE`인가
- `FAILED`, `INCOMPLETE`가 있는가
- MAC 주소가 계속 바뀌는가

### 35-5) Socket 확인

```bash
ss -lntp
ss -antp
```

확인할 것:

- 서버가 실제로 LISTEN 중인가
- 127.0.0.1에만 바인딩된 것은 아닌가
- SYN-SENT가 쌓이는가
- CLOSE-WAIT가 비정상적으로 많은가

### 35-6) 방화벽/NAT 확인

```bash
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v
sudo nft list ruleset
```

확인할 것:

- DROP rule이 있는가
- NAT rule이 예상대로 있는가
- Kubernetes Service Chain이 있는가

### 35-7) tcpdump 확인

```bash
sudo tcpdump -i any host <ip>
```

확인할 것:

- SYN이 나가는가
- SYN/ACK가 돌아오는가
- ARP 요청/응답이 있는가
- ICMP unreachable이 오는가
- Fragmentation needed가 오는가

----

## 36. 장애 사례 1: Ping이 안 됨

증상:

```bash
ping 192.168.10.1
```

실패.

점검:

```bash
ip link show ens3
ip addr show ens3
ip route get 192.168.10.1
ip neigh show 192.168.10.1
sudo tcpdump -i ens3 arp or icmp
```

가능한 원인:

- Interface down
- IP 없음
- Prefix 오류
- Gateway 장애
- VLAN 불일치
- ARP 실패
- 방화벽에서 ICMP 차단

판단:

```text
ARP 요청만 나가고 응답이 없다
  → L2/VLAN/Gateway 문제 가능성

ARP는 되는데 ICMP 응답이 없다
  → 대상 방화벽 또는 ICMP 차단 가능성

route get이 엉뚱한 interface를 가리킨다
  → Routing 문제
```

----

## 37. 장애 사례 2: 포트 접속이 안 됨

증상:

```bash
curl http://192.168.10.50:8080
```

실패.

점검:

서버 측:

```bash
ss -lntp | grep 8080
ip addr
sudo iptables -L -n -v
```

클라이언트 측:

```bash
ip route get 192.168.10.50
nc -vz 192.168.10.50 8080
sudo tcpdump -i any host 192.168.10.50 and port 8080
```

가능한 원인:

- 애플리케이션이 LISTEN하지 않음
- 127.0.0.1에만 바인딩
- 방화벽 차단
- 라우팅 문제
- 서버에서 응답 경로가 다름

특히 이 경우가 자주 발생합니다.

```text
LISTEN 127.0.0.1:8080
```

외부에서 접근하려면 보통 다음처럼 떠야 합니다.

```text
LISTEN 0.0.0.0:8080
```

또는 특정 외부 IP에 바인딩되어 있어야 합니다.

----

## 38. 장애 사례 3: DNS만 안 됨

증상:

```bash
ping 8.8.8.8        # 성공
ping google.com     # 실패
```

이 경우 IP 통신은 되지만 DNS가 실패하는 상황입니다.

점검:

```bash
cat /etc/resolv.conf
nslookup google.com
dig google.com
ss -lunp | grep :53
```

Kubernetes Pod라면:

```bash
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes.default
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc -n kube-system kube-dns
```

가능한 원인:

- DNS 서버 주소 오류
- CoreDNS 장애
- NetworkPolicy 차단
- UDP 53 차단
- `/etc/resolv.conf` 설정 오류
- `ndots`로 인한 검색 지연

----

## 39. 장애 사례 4: 작은 요청은 되는데 큰 요청이 실패

증상:

```text
ping은 됨
curl 일부는 됨
이미지 pull은 실패
gRPC stream이 끊김
TLS Handshake가 간헐적으로 실패
```

MTU 문제일 수 있습니다.

점검:

```bash
ip link
ping -M do -s 1472 <destination-ip>
ping -M do -s 1400 <destination-ip>
sudo tcpdump -i any icmp
```

Overlay Network에서는 Pod MTU가 Node NIC MTU보다 작아야 할 수 있습니다.

예:

```text
Node NIC MTU 1500
VXLAN overhead 50
Pod MTU 1450
```

Cilium/Calico 설정에서 MTU를 확인해야 합니다.

예:

```bash
kubectl get ds cilium -n kube-system -o yaml | grep -i mtu
helm get values cilium -n kube-system | grep -i mtu
```

----

## 40. 장애 사례 5: Pod IP는 되는데 Service IP는 안 됨

증상:

```bash
kubectl exec -it test -- curl http://10.244.1.20:8080   # Pod IP 성공
kubectl exec -it test -- curl http://10.96.10.20:80      # Service IP 실패
```

이 경우 Pod 네트워크 자체는 되지만 Service 처리 계층이 문제일 수 있습니다.

점검:

```bash
kubectl get svc
kubectl get endpoints
kubectl get endpointslice
kubectl get pods -o wide
```

iptables 기반이면:

```bash
sudo iptables -t nat -L -n -v | grep KUBE
```

Cilium이면:

```bash
cilium service list
cilium endpoint list
cilium monitor
```

가능한 원인:

- Service selector가 Pod label과 불일치
- Endpoint 없음
- kube-proxy 장애
- Cilium agent 장애
- NetworkPolicy 차단
- Pod readiness 실패로 Endpoint 제외

----

## 41. 장애 사례 6: Pod에서 외부 인터넷이 안 됨

증상:

```bash
kubectl exec -it <pod> -- ping 8.8.8.8
```

실패.

점검:

Pod 내부:

```bash
ip addr
ip route
cat /etc/resolv.conf
```

Node:

```bash
ip route
sudo iptables -t nat -L -n -v | grep MASQUERADE
sudo tcpdump -i any host 8.8.8.8
```

가능한 원인:

- Pod default route 없음
- CNI masquerade 설정 문제
- Node default gateway 문제
- 외부 방화벽 차단
- NetworkPolicy Egress 차단
- MTU 문제

----

## 42. 자주 쓰는 Linux 네트워크 점검 명령어

```bash
# Interface
ip link
ip -br link
ip -s link
ip -d link show <interface>

# IP address
ip addr
ip -br addr
ip addr show <interface>

# Route
ip route
ip route get <destination-ip>
ip rule

# ARP / Neighbor
ip neigh
ip neigh show dev <interface>

# Socket
ss -lntp
ss -antp
ss -lunp
ss -s

# Bridge
ip link show type bridge
bridge link
bridge fdb show
bridge vlan show

# Namespace
ip netns list
lsns -t net
nsenter -t <pid> -n ip addr

# DNS
cat /etc/resolv.conf
nslookup <domain>
dig <domain>

# Packet capture
sudo tcpdump -i any host <ip>
sudo tcpdump -i <interface> port <port>

# iptables / nftables
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v
sudo iptables-save
sudo nft list ruleset

# NIC
ethtool <interface>
ethtool -i <interface>
ethtool -k <interface>
```

----

## 43. Kubernetes 네트워크 점검 명령어

```bash
# Pod 위치와 IP 확인
kubectl get pods -A -o wide

# Service / Endpoint 확인
kubectl get svc -A
kubectl get endpoints -A
kubectl get endpointslice -A

# Pod 내부 네트워크 확인
kubectl exec -it <pod> -- ip addr
kubectl exec -it <pod> -- ip route
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes.default

# Node 네트워크 확인
ip addr
ip route
ip neigh
ss -lntp

# CNI Pod 확인
kubectl get pods -n kube-system -o wide
kubectl get ds -n kube-system

# kube-proxy 사용 시
kubectl get ds kube-proxy -n kube-system
sudo iptables -t nat -L -n -v | grep KUBE

# Cilium 사용 시
kubectl get pods -n kube-system -l k8s-app=cilium
cilium status
cilium endpoint list
cilium service list
cilium monitor
```

----

## 44. 전체 흐름도: Linux 네트워크와 Kubernetes 연결

전체를 하나로 연결하면 다음과 같습니다.

```text
[Pod Application]
        ↓
[Socket]
        ↓
[TCP / UDP]
        ↓
[Pod IP]
        ↓
[Pod eth0]
        ↓
[veth pair]
        ↓
[Host Network Namespace]
        ↓
[CNI 처리]
        ↓
[Routing Table]
        ↓
[iptables / nftables / eBPF]
        ↓
[Node NIC]
        ↓
[Switch / Router]
        ↓
[Destination]
```

Service를 거치는 경우:

```text
Client Pod
  ↓
Service IP
  ↓
kube-proxy iptables 또는 Cilium eBPF
  ↓
Endpoint Pod IP
  ↓
Pod veth
  ↓
Target Pod
```

외부로 나가는 경우:

```text
Pod IP
  ↓
Node CNI
  ↓
SNAT / Masquerade
  ↓
Node IP
  ↓
Gateway
  ↓
Internet
```

Overlay Network인 경우:

```text
Pod Packet
  ↓
Encapsulation VXLAN/Geneve
  ↓
Node NIC
  ↓
Underlay Network
  ↓
Remote Node NIC
  ↓
Decapsulation
  ↓
Remote Pod
```

----

## 45. 최종 정리

Linux 네트워크는 Kubernetes 네트워크를 이해하기 위한 기반입니다.

핵심은 다음과 같습니다.

```text
Interface는 패킷이 드나드는 입구다.
NIC는 실제 또는 가상 네트워크 장치다.
IP는 L3 주소다.
MAC은 L2 주소다.
Routing Table은 목적지까지의 출구를 결정한다.
ARP/Neighbor는 IP를 MAC으로 바꿔준다.
Bridge는 Linux 안의 소프트웨어 스위치다.
Namespace는 네트워크 스택을 격리한다.
veth pair는 Namespace 사이를 잇는 가상 랜선이다.
CNI는 이 기능들을 조합해 Pod 네트워크를 만든다.
```

장애 분석은 다음 순서로 접근하면 됩니다.

```text
Interface
  ↓
IP Address
  ↓
Routing Table
  ↓
Neighbor / ARP
  ↓
Socket
  ↓
Firewall / NAT
  ↓
DNS
  ↓
tcpdump
  ↓
CNI / Kubernetes Service / NetworkPolicy
```

Kubernetes에서 네트워크 문제가 발생하더라도 결국 아래 Linux 구성요소를 확인해야 합니다.

- `ip link`
- `ip addr`
- `ip route`
- `ip neigh`
- `ss`
- `tcpdump`
- `iptables` / `nftables`
- `bridge`
- `ip netns`
- CNI 상태

이 문서를 이해하면 이후의 Kubernetes CNI, Calico, Cilium, kube-proxy, Service NAT, NetworkPolicy, eBPF 네트워크 구조를 훨씬 쉽게 이해할 수 있습니다.
