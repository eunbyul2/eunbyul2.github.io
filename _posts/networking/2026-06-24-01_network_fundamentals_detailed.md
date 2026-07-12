---
layout: post
title: "네트워크 기초: Ethernet, OSI, TCP/IP, IP, CIDR, DNS, Routing, ARP, NAT"
date: 2026-06-24 18:20:00 +0900
categories: [Network, Infrastructure]
tags: [Network, OSI, TCPIP, Ethernet, IP, CIDR, DNS, Routing, ARP, NAT, Conntrack]
published: true
---

## 1. 이 글에서 다루는 범위

이 글은 Kubernetes, Ingress, Load Balancer, HAProxy, Observability를 이해하기 전에 반드시 알아야 하는 네트워크 기본기를 정리한 글입니다.

네트워크를 처음 볼 때 가장 중요한 관점은 다음 한 문장입니다.

> 요청은 애플리케이션에서 시작하지만, 실제 전송은 패킷, 프레임, 라우팅, ARP, NAT, 포트, 세션 상태를 거쳐 목적지까지 이동한다.

브라우저에서 다음 주소를 호출한다고 가정합니다.

```text
https://app.example.com/api/users
```

겉으로는 웹사이트 하나를 여는 것처럼 보입니다. 하지만 실제로는 아래 단계들이 순서대로 동작합니다.

```text
사용자 입력
  ↓
DNS 조회
  ↓
목적지 IP 확인
  ↓
Routing Table 조회
  ↓
Gateway 또는 직접 통신 여부 결정
  ↓
ARP로 다음 홉 MAC 주소 확인
  ↓
Ethernet Frame 생성
  ↓
IP Packet 전달
  ↓
TCP 3-Way Handshake
  ↓
TLS Handshake
  ↓
HTTP Request 전송
  ↓
HTTP Response 수신
```

Kubernetes, HAProxy, Ingress, Cilium, kube-proxy, Ceph Dashboard, Prometheus, Whatap 같은 시스템도 결국 이 흐름 위에서 동작합니다.

공식 참고자료:

- IETF RFC 894 - IP Datagrams over Ethernet: <https://datatracker.ietf.org/doc/html/rfc894>
- IETF RFC 791 - IPv4: <https://datatracker.ietf.org/doc/html/rfc791>
- IETF RFC 8200 - IPv6: <https://datatracker.ietf.org/doc/html/rfc8200>
- IETF RFC 1122 - Host Requirements: <https://datatracker.ietf.org/doc/html/rfc1122>
- IETF RFC 9293 - TCP: <https://datatracker.ietf.org/doc/html/rfc9293>
- IETF RFC 768 - UDP: <https://datatracker.ietf.org/doc/html/rfc768>
- IETF RFC 792 - ICMP: <https://datatracker.ietf.org/doc/html/rfc792>
- IETF RFC 826 - ARP: <https://datatracker.ietf.org/doc/html/rfc826>
- IETF RFC 1918 - Private IPv4: <https://datatracker.ietf.org/doc/html/rfc1918>
- IETF RFC 4632 - CIDR: <https://datatracker.ietf.org/doc/html/rfc4632>
- Linux ip-route manual: <https://man7.org/linux/man-pages/man8/ip-route.8.html>
- Linux ip-address manual: <https://man7.org/linux/man-pages/man8/ip-address.8.html>
- Linux ip-link manual: <https://man7.org/linux/man-pages/man8/ip-link.8.html>
- Netfilter NAT HOWTO: <https://www.netfilter.org/documentation/HOWTO/NAT-HOWTO.txt>
- Netfilter conntrack-tools: <https://conntrack-tools.netfilter.org/>

## 2. 네트워크 통신을 이해하는 핵심 관점

### 2-1) 네트워크는 계층적으로 포장해서 보낸다

네트워크 데이터는 한 번에 그대로 전송되지 않습니다. 상위 계층의 데이터가 하위 계층으로 내려가면서 계속 포장됩니다. 이 과정을 Encapsulation이라고 합니다.

예를 들어 HTTP 요청은 다음처럼 포장됩니다.

```text
HTTP Request
  ↓ TCP Header 추가
TCP Segment
  ↓ IP Header 추가
IP Packet
  ↓ Ethernet Header/Trailer 추가
Ethernet Frame
  ↓ NIC를 통해 전기/광 신호로 전송
Physical Signal
```

반대로 수신 측에서는 포장을 하나씩 벗깁니다. 이 과정을 Decapsulation이라고 합니다.

```text
Physical Signal
  ↓
Ethernet Frame
  ↓ Ethernet Header 제거
IP Packet
  ↓ IP Header 제거
TCP Segment
  ↓ TCP Header 제거
HTTP Request
```

### 2-2) 계층별로 보는 정보가 다르다

각 계층은 자기 계층의 정보만 주로 봅니다.

| 계층 | 보는 정보 | 대표 장비/기능 |
|---|---|---|
| L2 | MAC Address, Ethernet Frame | Switch, Bridge, veth |
| L3 | Source IP, Destination IP, TTL | Router, Gateway, Routing Table |
| L4 | TCP/UDP Port, SYN/ACK, 세션 | Firewall, L4 Load Balancer |
| L7 | Host, Path, Header, Body, Status Code | Reverse Proxy, Ingress, L7 Load Balancer |

이 구분이 중요한 이유는 장애를 잘라서 볼 수 있기 때문입니다.

```text
DNS는 되는데 ping이 안 된다
→ L3 경로 문제 가능성

ping은 되는데 nc -vz 443이 실패한다
→ L4 포트/방화벽 문제 가능성

443 연결은 되는데 404가 나온다
→ L7 Host/Path 라우팅 문제 가능성
```

### 2-3) Kubernetes에서도 결국 Linux 네트워크다

Kubernetes의 Pod, Service, Ingress는 특별한 마법이 아닙니다.

실제로는 Linux Kernel의 기능을 조합합니다.

- Network Namespace
- veth pair
- Linux Bridge 또는 eBPF datapath
- Routing Table
- ARP/Neighbor Table
- iptables/nftables/IPVS/eBPF
- conntrack
- DNS/CoreDNS

따라서 Kubernetes 네트워크 장애를 분석하려면 Linux 네트워크 기본기를 알아야 합니다.

## 3. OSI 7 Layer

OSI 7 Layer는 네트워크 통신을 7개의 계층으로 나누어 설명하는 모델입니다. 실제 인터넷이 OSI 모델 그대로 구현된 것은 아니지만, 장애 위치를 구분할 때 매우 유용합니다.

| 계층 | 이름 | 핵심 역할 | 예시 |
|---|---|---|---|
| L7 | Application | 애플리케이션 데이터 의미 처리 | HTTP, DNS, SMTP, gRPC |
| L6 | Presentation | 표현, 인코딩, 암호화 | TLS, JSON, 압축 |
| L5 | Session | 세션 관리 | 세션 유지, 인증 세션 |
| L4 | Transport | 프로세스 간 전송 | TCP, UDP, Port |
| L3 | Network | IP 기반 경로 결정 | IP, Routing, ICMP |
| L2 | Data Link | 같은 네트워크 안에서 프레임 전달 | Ethernet, MAC, ARP |
| L1 | Physical | 물리 신호 전달 | 케이블, 광 모듈, NIC |

실무에서는 L3, L4, L7이라는 표현을 가장 자주 사용합니다.

```text
L3 문제
- IP가 맞는가?
- 라우팅 경로가 있는가?
- Gateway로 갈 수 있는가?
- ICMP 응답이 오는가?

L4 문제
- TCP 포트가 열려 있는가?
- SYN/SYN-ACK/ACK가 정상인가?
- UDP 패킷이 도착하는가?
- 방화벽에서 포트를 막는가?

L7 문제
- HTTP Host가 맞는가?
- Path가 Ingress Rule과 일치하는가?
- Header가 누락되지 않았는가?
- 애플리케이션이 4xx/5xx를 반환하는가?
```

## 4. TCP/IP 4 Layer

TCP/IP 모델은 실제 인터넷 프로토콜 스택을 더 현실적으로 설명합니다.

| TCP/IP 계층 | OSI 대응 | 대표 기술 |
|---|---|---|
| Application | L5~L7 | HTTP, DNS, TLS, SSH |
| Transport | L4 | TCP, UDP |
| Internet | L3 | IP, ICMP, Routing |
| Link | L1~L2 | Ethernet, ARP, NIC |

예를 들어 `https://app.example.com`으로 접속하는 흐름은 다음과 같습니다.

```text
Application: HTTP Request 생성
Transport: TCP 443 연결 생성
Internet: 목적지 IP로 라우팅
Link: 다음 홉 MAC 주소로 Ethernet Frame 전달
```

중요한 점은 "HTTP 요청을 보낸다"라는 말 안에 실제로는 TCP, IP, Ethernet이 모두 포함되어 있다는 것입니다.

## 5. Ethernet

### 5-1) Ethernet이란 무엇인가

Ethernet은 같은 LAN 안에서 데이터를 전달하기 위한 대표적인 L2 기술입니다.

우리가 흔히 말하는 유선 LAN, 스위치, NIC, MAC Address는 대부분 Ethernet 기반입니다.

```text
서버 NIC
  ↓
스위치
  ↓
다른 서버 NIC
```

Ethernet의 핵심은 다음과 같습니다.

- 같은 L2 네트워크 안에서 통신한다.
- MAC Address를 사용한다.
- 데이터를 Ethernet Frame 단위로 전송한다.
- Switch는 MAC Address를 기준으로 Frame을 전달한다.

### 5-2) Ethernet은 IP를 직접 보내지 않는다

IP Packet은 Ethernet 위에 실려서 전송됩니다.

```text
IP Packet
  ↓ Ethernet Header/Trailer로 감싸기
Ethernet Frame
```

RFC 894는 IP Datagram을 Ethernet Network 위에서 전송하는 방법을 설명합니다.

즉, 실제 전송 단위는 IP Packet 자체가 아니라 Ethernet Frame입니다.

### 5-3) Ethernet Header 구조

단순화하면 Ethernet Frame은 다음과 같습니다.

```text
+----------------------+--------------------+-----------+-------------------+-----+
| Destination MAC      | Source MAC         | EtherType | Payload           | FCS |
+----------------------+--------------------+-----------+-------------------+-----+
| 목적지 MAC            | 출발지 MAC          | 상위 프로토콜 | IP Packet 등       | 오류검사 |
+----------------------+--------------------+-----------+-------------------+-----+
```

대표 EtherType:

| EtherType | 의미 |
|---|---|
| 0x0800 | IPv4 |
| 0x86DD | IPv6 |
| 0x0806 | ARP |

### 5-4) Kubernetes에서 Ethernet이 왜 중요한가

Kubernetes Pod 통신도 결국 노드 내부에서는 Linux 가상 네트워크 장치를 지나갑니다.

예를 들어 일반적인 CNI 구조에서는 다음과 같은 흐름이 나옵니다.

```text
Pod eth0
  ↓
veth pair
  ↓
Host Network Namespace
  ↓
Linux Bridge 또는 CNI datapath
  ↓
Node NIC
  ↓
Switch
```

Cilium처럼 eBPF 기반 CNI를 사용하더라도, 최종적으로 물리 네트워크로 나갈 때는 NIC와 Ethernet을 사용합니다.

## 6. Frame

### 6-1) Frame이란 무엇인가

Frame은 L2 계층의 데이터 전송 단위입니다.

계층별 데이터 단위는 보통 다음처럼 구분합니다.

| 계층 | 데이터 단위 |
|---|---|
| L7 | Message |
| L4 | Segment(TCP), Datagram(UDP) |
| L3 | Packet |
| L2 | Frame |
| L1 | Bit/Signal |

즉, HTTP 요청은 최종적으로 Ethernet Frame 안에 들어가서 전송됩니다.

```text
HTTP Message
  ↓
TCP Segment
  ↓
IP Packet
  ↓
Ethernet Frame
```

### 6-2) Frame에는 MAC 주소가 들어간다

IP Packet에는 Source IP, Destination IP가 들어갑니다.

Ethernet Frame에는 Source MAC, Destination MAC이 들어갑니다.

```text
IP Header
- Source IP
- Destination IP

Ethernet Header
- Source MAC
- Destination MAC
```

같은 서버로 보내는 HTTP 요청이라도 다음 홉이 바뀌면 Destination MAC은 달라질 수 있습니다.

예를 들어 인터넷으로 나가는 요청은 최종 목적지 서버의 MAC으로 보내는 것이 아닙니다.

```text
내 서버 -> 인터넷 서버
Destination IP: 인터넷 서버 IP
Destination MAC: Gateway MAC
```

이 부분이 매우 중요합니다.

L3의 목적지는 최종 목적지 IP입니다.  
L2의 목적지는 다음 홉 MAC입니다.

## 7. MAC Address

### 7-1) MAC Address란 무엇인가

MAC Address는 네트워크 인터페이스가 가지는 L2 주소입니다.

예시:

```text
00:16:3e:ab:12:34
52:54:00:aa:bb:cc
```

IP Address가 L3 주소라면, MAC Address는 L2 주소입니다.

| 구분 | IP Address | MAC Address |
|---|---|---|
| 계층 | L3 | L2 |
| 용도 | 네트워크 간 라우팅 | 같은 L2 구간 프레임 전달 |
| 예시 | 192.168.10.25 | 00:16:3e:ab:12:34 |
| 변경 가능성 | 네트워크 설정에 따라 변경 | NIC/가상 NIC에 부여 |

### 7-2) IP만 있으면 안 되는 이유

많은 사람이 처음에 이렇게 생각합니다.

```text
어차피 IP로 통신한다면서 왜 MAC이 필요한가?
```

이유는 실제 LAN에서 데이터를 보내는 장비가 Ethernet이기 때문입니다.

컴퓨터가 같은 네트워크의 다른 컴퓨터로 데이터를 보낼 때는 Ethernet Frame을 만들어야 합니다. Ethernet Frame에는 Destination MAC이 반드시 필요합니다.

그래서 IP 주소를 알고 있어도, 실제 전송 전에는 다음 홉의 MAC 주소를 알아야 합니다.

이때 사용하는 것이 ARP입니다.

## 8. Switch와 Hub

### 8-1) Hub

Hub는 들어온 신호를 모든 포트로 뿌리는 장비입니다.

```text
A가 B에게 전송
  ↓
Hub는 모든 포트로 전달
  ↓
B도 받고 C도 받고 D도 받음
```

Hub는 목적지 MAC을 학습하지 않습니다. 그래서 충돌과 불필요한 트래픽이 많습니다. 현대 네트워크에서는 거의 사용하지 않습니다.

### 8-2) Switch

Switch는 MAC Address Table을 학습해서 필요한 포트로만 Frame을 전달합니다.

```text
MAC Address Table

00:11:22:aa:bb:cc -> port 1
00:11:22:dd:ee:ff -> port 2
00:11:22:11:22:33 -> port 3
```

A가 B에게 Frame을 보내면 Switch는 Destination MAC을 보고 B가 연결된 포트로만 전달합니다.

### 8-3) Switch가 모르는 MAC이면 어떻게 되는가

Switch가 Destination MAC을 모르면 Flooding합니다.

```text
Unknown Unicast
  ↓
Switch가 모든 포트로 전달
```

ARP Request도 Broadcast이므로 같은 VLAN/L2 도메인 전체로 전달됩니다.

### 8-4) 실무 장애 포인트

| 현상 | 가능 원인 |
|---|---|
| 같은 서브넷인데 통신 안 됨 | ARP 실패, VLAN 불일치, MAC Table 문제 |
| 특정 서버만 간헐적 통신 장애 | MAC Flapping, 이중화 장비 문제 |
| Broadcast 트래픽 과다 | 루프, ARP Storm, 잘못된 네트워크 구성 |

## 9. Broadcast와 Unicast

### 9-1) Unicast

Unicast는 하나의 출발지가 하나의 목적지로 보내는 통신입니다.

```text
A -> B
```

일반적인 TCP 연결, HTTP 요청, SSH 접속은 대부분 Unicast입니다.

### 9-2) Broadcast

Broadcast는 같은 L2 네트워크의 모든 장비에게 보내는 통신입니다.

Ethernet Broadcast MAC:

```text
ff:ff:ff:ff:ff:ff
```

ARP Request가 대표적인 Broadcast입니다.

```text
누가 192.168.10.1을 가지고 있나요?
가지고 있으면 192.168.10.25에게 알려주세요.
```

### 9-3) Broadcast Domain

Broadcast가 전달되는 범위를 Broadcast Domain이라고 합니다.

일반적으로 VLAN 또는 Subnet 단위로 생각하면 됩니다.

Broadcast가 너무 많으면 전체 네트워크 성능에 영향을 줄 수 있습니다.

## 10. Packet

### 10-1) Packet이란 무엇인가

Packet은 L3 계층의 데이터 단위입니다.

IPv4 Packet은 크게 Header와 Payload로 구성됩니다.

```text
+-----------------+------------------+
| IPv4 Header     | Payload          |
+-----------------+------------------+
```

IPv4 Header에는 다음 정보들이 포함됩니다.

- Source IP
- Destination IP
- TTL
- Protocol
- Fragment 관련 정보
- Header Checksum

### 10-2) Packet과 Frame의 차이

| 구분 | Packet | Frame |
|---|---|---|
| 계층 | L3 | L2 |
| 주소 | IP Address | MAC Address |
| 장비 | Router/Gateway | Switch |
| 범위 | 네트워크 간 전달 | 같은 L2 구간 전달 |

중요한 차이:

```text
Packet의 Destination IP는 최종 목적지다.
Frame의 Destination MAC은 다음 홉이다.
```

예시:

```text
내 서버: 192.168.10.25
Gateway: 192.168.10.1
인터넷 서버: 8.8.8.8

IP Packet
Destination IP = 8.8.8.8

Ethernet Frame
Destination MAC = 192.168.10.1 Gateway의 MAC
```

## 11. MTU

### 11-1) MTU란 무엇인가

MTU(Maximum Transmission Unit)는 하나의 L2 Frame이 실을 수 있는 최대 L3 Packet 크기입니다.

일반 Ethernet MTU는 보통 1500 bytes입니다.

```bash
ip link
```

예시:

```text
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
```

### 11-2) MTU가 중요한 이유

MTU보다 큰 Packet을 보내야 하면 두 가지 중 하나가 필요합니다.

1. Fragmentation
2. Path MTU Discovery를 통한 크기 조정

그런데 현대 네트워크에서는 Fragmentation을 피하는 것이 일반적입니다. 특히 터널링, VXLAN, Geneve, IPsec, WireGuard 같은 오버레이 네트워크에서는 원본 Packet 위에 추가 Header가 붙습니다.

예시:

```text
원본 IP Packet
  ↓ VXLAN Header 추가
외부 UDP/IP Header 추가
  ↓
물리 NIC로 전송
```

이때 물리 NIC MTU가 1500인데 오버레이 Header가 추가되면 실제로 실을 수 있는 Pod Packet 크기는 줄어듭니다.

### 11-3) Kubernetes에서 MTU 문제

Kubernetes CNI 환경에서는 MTU 문제가 자주 발생합니다.

대표 증상:

```text
작은 요청은 성공
큰 요청은 실패

ping은 성공
curl은 멈춤

TLS Handshake 중 멈춤

이미지 Pull 실패

Prometheus scrape 일부 실패

Ceph/RBD 통신 간헐 실패
```

Cilium, Calico, Flannel 같은 CNI는 각자 MTU 설정을 가집니다.

확인 예시:

```bash
ip link show

kubectl -n kube-system get cm cilium-config -o yaml | grep -i mtu

tracepath <destination>
```

### 11-4) MTU 확인 명령어

```bash
# 인터페이스 MTU 확인
ip link show

# 경로 MTU 추정
tracepath 8.8.8.8

# Don't Fragment 옵션으로 테스트
ping -M do -s 1472 8.8.8.8
```

IPv4에서 일반 Ethernet MTU 1500 기준으로 ICMP Payload 1472를 쓰는 이유:

```text
1500 - IPv4 Header 20 - ICMP Header 8 = 1472
```

단, 환경에 따라 Header 크기와 오버레이 여부가 다르므로 이 값은 항상 절대값으로 보면 안 됩니다.

## 12. Encapsulation과 Decapsulation

### 12-1) Encapsulation

Encapsulation은 상위 계층 데이터를 하위 계층 Header로 감싸는 과정입니다.

예시:

```text
HTTP Request
  ↓ TCP Header 추가
TCP Segment
  ↓ IP Header 추가
IP Packet
  ↓ Ethernet Header 추가
Ethernet Frame
```

각 계층은 자기 역할에 필요한 Header를 붙입니다.

| 계층 | 붙는 Header | 목적 |
|---|---|---|
| L7 | HTTP Header | 요청 의미 표현 |
| L4 | TCP/UDP Header | 포트, 연결, 순서 |
| L3 | IP Header | 출발지/목적지 IP |
| L2 | Ethernet Header | 출발지/목적지 MAC |

### 12-2) Decapsulation

수신 측에서는 반대로 Header를 하나씩 제거합니다.

```text
Ethernet Frame 수신
  ↓ MAC 확인
IP Packet 추출
  ↓ Destination IP 확인
TCP Segment 추출
  ↓ Destination Port 확인
HTTP Request 추출
  ↓ 애플리케이션 처리
```

### 12-3) 트러블슈팅에서 중요한 이유

장애 분석은 보통 Decapsulation 순서대로 확인하면 됩니다.

```text
L2: Frame이 NIC까지 도착했는가?
L3: IP Packet이 목적지까지 라우팅되는가?
L4: TCP 연결이 수립되는가?
L7: HTTP 응답이 정상인가?
```

도구로 보면 다음과 같습니다.

```bash
# L2/L3/L4 패킷 확인
tcpdump -i any -nn host 10.0.0.10

# L3 경로 확인
ip route get 10.0.0.10

# L4 포트 확인
nc -vz 10.0.0.10 443

# L7 확인
curl -vk https://app.example.com
```

## 13. IP Address

### 13-1) IP Address가 필요한 이유

IP Address는 네트워크에서 장비 또는 인터페이스를 식별하는 L3 주소입니다.

예시:

```text
192.168.10.25
10.0.0.5
172.16.1.10
8.8.8.8
```

IPv4는 32비트 주소이며, 사람이 보기 쉽도록 8비트씩 4개로 나누어 10진수로 표현합니다.

```text
192.168.10.25
= 11000000.10101000.00001010.00011001
```

IP 주소가 필요한 이유는 네트워크가 하나만 있는 것이 아니기 때문입니다.

MAC Address는 같은 L2 네트워크 안에서만 의미가 있습니다. 하지만 인터넷은 수많은 네트워크가 라우터로 연결된 구조입니다. 서로 다른 네트워크를 넘나들기 위해서는 계층적인 주소 체계가 필요합니다. 그 역할을 IP가 합니다.

### 13-2) IP는 인터페이스에 붙는다

정확히 말하면 IP는 서버 자체가 아니라 네트워크 인터페이스에 할당됩니다.

```bash
ip addr
```

예시:

```text
2: ens3:
    inet 192.168.10.25/24 brd 192.168.10.255 scope global ens3
```

서버에 NIC가 여러 개 있으면 IP도 여러 개일 수 있습니다.

```text
ens3: 192.168.10.25/24
ens4: 10.10.0.25/24
```

Kubernetes Node도 마찬가지입니다.

```text
Node Internal IP
Node External IP
Pod IP
Service IP
```

각 IP는 용도와 라우팅 범위가 다릅니다.

## 14. Public IP와 Private IP

### 14-1) Public IP

Public IP는 인터넷에서 라우팅 가능한 IP입니다. 전 세계 인터넷에서 중복되면 안 됩니다.

예시:

```text
8.8.8.8
1.1.1.1
```

Public IP는 ISP, 클라우드 사업자, 데이터센터 네트워크 등을 통해 할당됩니다.

### 14-2) Private IP

Private IP는 사설망 내부에서 사용하는 IP입니다. RFC 1918에서 정의한 사설 IPv4 대역은 다음과 같습니다.

| 대역 | 범위 |
|---|---|
| 10.0.0.0/8 | 10.0.0.0 ~ 10.255.255.255 |
| 172.16.0.0/12 | 172.16.0.0 ~ 172.31.255.255 |
| 192.168.0.0/16 | 192.168.0.0 ~ 192.168.255.255 |

Private IP는 인터넷에 직접 라우팅되지 않습니다. 그래서 외부 인터넷으로 나갈 때는 보통 NAT가 필요합니다.

### 14-3) Kubernetes와 Private IP

Kubernetes에서는 보통 다음 대역들이 Private IP로 구성됩니다.

```text
Node IP:    192.168.10.0/24
Pod CIDR:   10.244.0.0/16
Service CIDR: 10.96.0.0/12
```

Pod IP와 Service IP는 대부분 클러스터 내부 전용입니다. 외부에서 직접 접근하려면 NodePort, LoadBalancer, Ingress 같은 추가 노출 방식이 필요합니다.

## 15. CIDR

### 15-1) CIDR이란 무엇인가

CIDR(Classless Inter-Domain Routing)은 IP 주소와 네트워크 크기를 함께 표현하는 방식입니다.

예시:

```text
192.168.10.0/24
```

여기서 `/24`는 앞의 24비트가 네트워크 주소라는 뜻입니다. IPv4는 총 32비트이므로 남은 8비트는 호스트 주소로 사용됩니다.

```text
Network bits: 24
Host bits: 8
주소 개수: 2^8 = 256
```

### 15-2) 자주 보는 CIDR

| CIDR | 주소 개수 | 일반적 의미 |
|---|---:|---|
| /32 | 1 | 단일 IP |
| /30 | 4 | P2P 링크 |
| /24 | 256 | 일반적인 작은 서브넷 |
| /16 | 65,536 | 큰 내부망 |
| /8 | 16,777,216 | 매우 큰 내부망 |

### 15-3) Network Address, Broadcast Address, Host Address

예를 들어 다음 CIDR이 있다고 가정합니다.

```text
192.168.10.0/24
```

구성은 다음과 같습니다.

```text
Network Address: 192.168.10.0
Usable Host:     192.168.10.1 ~ 192.168.10.254
Broadcast:       192.168.10.255
```

보통 Network Address와 Broadcast Address는 호스트에 할당하지 않습니다.

### 15-4) CIDR 계산이 중요한 이유

라우팅은 CIDR 단위로 동작합니다.

예를 들어 Linux 라우팅 테이블에 다음이 있다고 가정합니다.

```text
10.0.0.0/8 via 192.168.10.1
10.244.0.0/16 via 192.168.10.83
10.244.1.0/24 via 192.168.10.84
```

목적지가 `10.244.1.20`이면 어떤 경로가 선택될까요?

정답은 `10.244.1.0/24`입니다. 가장 구체적인 경로이기 때문입니다. 이것이 Longest Prefix Match입니다.

## 16. Subnet

### 16-1) Subnet이란 무엇인가

Subnet은 하나의 큰 네트워크를 작은 네트워크로 나눈 것입니다.

예를 들어 `192.168.0.0/16`을 여러 개의 `/24`로 나눌 수 있습니다.

```text
192.168.1.0/24
192.168.2.0/24
192.168.3.0/24
```

Subnet을 나누는 이유는 다음과 같습니다.

- Broadcast 범위 제한
- 보안 구간 분리
- 라우팅 정책 분리
- 장애 범위 축소
- IP 관리 단순화

### 16-2) 같은 Subnet과 다른 Subnet

같은 Subnet이면 Gateway 없이 직접 통신합니다.

```text
192.168.10.10/24 -> 192.168.10.20/24
```

이 경우 목적지 IP가 같은 Subnet에 있으므로 ARP로 목적지 MAC을 찾고 직접 Frame을 보냅니다.

다른 Subnet이면 Gateway를 거칩니다.

```text
192.168.10.10/24 -> 192.168.20.20/24
```

이 경우 목적지 IP는 최종 목적지 `192.168.20.20`이지만, Ethernet Frame의 Destination MAC은 Gateway의 MAC이 됩니다.

## 17. Gateway

### 17-1) Gateway란 무엇인가

Gateway는 현재 네트워크 밖으로 나갈 때 사용하는 다음 홉입니다.

일반적으로 기본 게이트웨이는 Default Route로 설정됩니다.

```bash
ip route
```

예시:

```text
default via 192.168.10.1 dev ens3
192.168.10.0/24 dev ens3 proto kernel scope link src 192.168.10.25
```

해석:

```text
192.168.10.0/24 안의 목적지
→ ens3로 직접 통신

그 외 목적지
→ 192.168.10.1 Gateway로 전달
```

### 17-2) Gateway로 보낸다는 말의 정확한 의미

Gateway로 보낸다는 것은 Destination IP를 Gateway IP로 바꾼다는 뜻이 아닙니다.

IP Packet의 Destination IP는 최종 목적지 그대로 유지됩니다.

```text
Destination IP = 8.8.8.8
```

다만 Ethernet Frame의 Destination MAC이 Gateway의 MAC으로 설정됩니다.

```text
Destination MAC = Gateway MAC
```

즉:

```text
L3 목적지: 최종 목적지 IP
L2 목적지: 다음 홉 MAC
```

이 차이를 이해해야 ARP와 Routing을 제대로 이해할 수 있습니다.

## 18. Routing

### 18-1) Routing이란 무엇인가

Routing은 목적지 IP까지 어떤 경로로 보낼지 결정하는 과정입니다.

Linux에서는 Kernel이 Routing Table을 보고 경로를 선택합니다.

```bash
ip route
ip route get 8.8.8.8
```

예시:

```text
default via 192.168.10.1 dev ens3
10.244.0.0/16 via 192.168.10.83 dev ens3
192.168.10.0/24 dev ens3 proto kernel scope link src 192.168.10.25
```

### 18-2) Route Lookup 과정

목적지 IP가 `10.244.1.20`이라고 가정합니다.

Routing Table:

```text
default via 192.168.10.1
10.0.0.0/8 via 192.168.10.1
10.244.0.0/16 via 192.168.10.83
10.244.1.0/24 via 192.168.10.84
```

매칭되는 Route:

```text
10.0.0.0/8
10.244.0.0/16
10.244.1.0/24
```

이 중 가장 구체적인 `/24`가 선택됩니다.

```text
10.244.1.0/24 via 192.168.10.84
```

### 18-3) Longest Prefix Match

Longest Prefix Match는 여러 Route가 동시에 매칭될 때 가장 긴 Prefix를 가진 Route를 선택하는 규칙입니다.

```text
/24는 /16보다 구체적이다.
/16은 /8보다 구체적이다.
/8은 /0보다 구체적이다.
```

Default Route는 `0.0.0.0/0`입니다. 모든 IP에 매칭되지만 가장 덜 구체적입니다.

```text
0.0.0.0/0
= 모든 목적지
= 마지막 후보
```

### 18-4) Kubernetes에서 Routing

Kubernetes Pod Network도 결국 Routing이 필요합니다.

예시:

```text
Node A Pod CIDR: 10.244.1.0/24
Node B Pod CIDR: 10.244.2.0/24
```

Pod A가 Pod B로 통신하려면 Node A는 `10.244.2.0/24`로 가는 경로를 알아야 합니다.

CNI는 이 경로를 다음 중 하나로 구성합니다.

- 각 Node에 Route 추가
- Overlay Tunnel 사용
- eBPF datapath 사용
- BGP로 Pod CIDR 광고
- iptables/IPVS/eBPF로 Service NAT 처리

## 19. ARP

### 19-1) ARP란 무엇인가

ARP(Address Resolution Protocol)는 IP 주소에 해당하는 MAC 주소를 찾는 프로토콜입니다.

RFC 826은 ARP의 목적을 Protocol Address, 예를 들어 IP Address를 Local Network Address, 예를 들어 Ethernet Address로 변환하는 방법으로 설명합니다.

간단히 말하면 다음 질문을 해결합니다.

```text
192.168.10.1의 MAC 주소가 뭐야?
```

### 19-2) ARP 동작 과정

서버 `192.168.10.25`가 Gateway `192.168.10.1`로 패킷을 보내야 한다고 가정합니다.

1. 서버가 Routing Table을 확인합니다.
2. 다음 홉이 `192.168.10.1`임을 확인합니다.
3. Ethernet Frame을 만들려면 Gateway MAC이 필요합니다.
4. ARP Cache를 확인합니다.
5. 없으면 ARP Request를 Broadcast로 보냅니다.
6. Gateway가 ARP Reply로 자기 MAC을 알려줍니다.
7. 서버는 ARP Cache에 저장합니다.
8. 이후 Ethernet Frame을 Gateway MAC으로 전송합니다.

```text
ARP Request:
Who has 192.168.10.1? Tell 192.168.10.25

ARP Reply:
192.168.10.1 is at 00:11:22:aa:bb:cc
```

### 19-3) ARP Cache

ARP Cache는 IP와 MAC의 매핑을 임시로 저장하는 테이블입니다.

Linux에서는 Neighbor Table이라고도 합니다.

```bash
ip neigh
```

예시:

```text
192.168.10.1 dev ens3 lladdr 00:11:22:aa:bb:cc REACHABLE
192.168.10.84 dev ens3 lladdr 52:54:00:aa:bb:01 STALE
```

상태 예시:

| 상태 | 의미 |
|---|---|
| REACHABLE | 최근 통신이 정상 |
| STALE | 오래되어 재확인 필요 |
| DELAY | 확인 대기 |
| PROBE | ARP 재확인 중 |
| FAILED | MAC 확인 실패 |

### 19-4) ARP 장애 사례

#### 사례 1: Gateway ARP 실패

```text
default via 192.168.10.1 dev ens3
```

Gateway 경로는 있는데 ARP가 실패하면 외부 통신이 안 됩니다.

증상:

```text
같은 서버 안에서는 정상
같은 Subnet 일부 통신 가능
외부 인터넷 불가
이미지 Pull 실패
DNS 외부 질의 실패
```

확인:

```bash
ip route
ip neigh
arping 192.168.10.1
```

#### 사례 2: 잘못된 MAC 매핑

ARP Cache에 잘못된 MAC이 들어가면 패킷이 엉뚱한 장비로 갑니다.

```bash
ip neigh flush dev ens3
```

단, 운영환경에서 무작정 flush하지 말고 영향 범위를 확인해야 합니다.

#### 사례 3: MetalLB, VIP, HA 구성

Kubernetes bare-metal 환경에서 MetalLB L2 모드를 쓰면 VIP에 대한 ARP 응답을 특정 Node가 대신합니다.

```text
Client asks:
Who has 192.168.10.240?

MetalLB Speaker replies:
192.168.10.240 is at Node A MAC
```

이 구조에서는 ARP 동작을 이해하지 못하면 VIP 장애를 분석하기 어렵습니다.

## 20. ICMP

### 20-1) ICMP란 무엇인가

ICMP(Internet Control Message Protocol)는 IP 네트워크에서 오류, 제어, 진단 메시지를 전달하기 위한 프로토콜입니다.

대표적으로 `ping`이 ICMP Echo Request/Reply를 사용합니다.

```bash
ping 8.8.8.8
```

하지만 ICMP는 ping만을 의미하지 않습니다.

대표 메시지:

| ICMP 메시지 | 의미 |
|---|---|
| Echo Request/Reply | ping |
| Destination Unreachable | 목적지 도달 불가 |
| Time Exceeded | TTL 만료 |
| Fragmentation Needed | MTU 문제 |
| Redirect | 더 적절한 Gateway 안내 |

### 20-2) ping이 된다고 서비스가 되는 것은 아니다

ping은 L3 수준의 도달성 확인입니다.

```text
ping 성공
```

이 말은 보통 다음 정도를 의미합니다.

```text
목적지 IP까지 ICMP Echo Request/Reply가 오간다.
```

하지만 HTTP 서비스가 정상이라는 뜻은 아닙니다.

```text
ping 성공
nc 443 실패
→ L4 포트 문제

ping 성공
nc 443 성공
curl 404
→ L7 라우팅/애플리케이션 문제
```

### 20-3) ping이 실패한다고 항상 서버가 죽은 것도 아니다

방화벽이나 보안 정책에서 ICMP를 막을 수 있습니다.

```text
ping 실패
nc 443 성공
curl 성공
```

이 경우 네트워크 전체 장애가 아니라 ICMP만 차단된 것일 수 있습니다.

## 21. TTL

### 21-1) TTL이란 무엇인가

TTL(Time To Live)은 IP Packet이 네트워크 안에서 무한히 떠도는 것을 막기 위한 값입니다.

IPv4 Header에 TTL 필드가 있습니다.

라우터를 하나 지날 때마다 TTL은 1씩 감소합니다.

```text
TTL 64
  ↓ Router 1
TTL 63
  ↓ Router 2
TTL 62
```

TTL이 0이 되면 라우터는 Packet을 폐기하고 보통 ICMP Time Exceeded를 반환합니다.

### 21-2) traceroute와 TTL

`traceroute`는 TTL을 일부러 1부터 증가시키면서 경로를 추적합니다.

```bash
traceroute 8.8.8.8
```

동작 개념:

```text
TTL 1 → 첫 번째 라우터에서 만료 → ICMP Time Exceeded
TTL 2 → 두 번째 라우터에서 만료 → ICMP Time Exceeded
TTL 3 → 세 번째 라우터에서 만료 → ICMP Time Exceeded
```

이렇게 각 Hop을 확인합니다.

### 21-3) Kubernetes에서 TTL을 볼 일이 있는가

직접 자주 보지는 않지만, 다음 상황에서는 중요합니다.

- 라우팅 루프 의심
- Overlay Network 경로 확인
- Node 간 경로 확인
- 방화벽/라우터 경유 여부 확인
- traceroute/tracepath 분석

## 22. Port

### 22-1) Port란 무엇인가

Port는 하나의 IP 안에서 어떤 프로세스로 전달할지 구분하는 번호입니다.

IP가 서버 주소라면 Port는 서버 안의 서비스 번호입니다.

```text
192.168.10.10:22   -> SSH
192.168.10.10:80   -> HTTP
192.168.10.10:443  -> HTTPS
192.168.10.10:6443 -> Kubernetes API Server
```

### 22-2) Well-Known Port

Well-Known Port는 일반적으로 잘 알려진 서비스가 사용하는 포트입니다.

| Port | Protocol | 용도 |
|---:|---|---|
| 22 | TCP | SSH |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 6443 | TCP | Kubernetes API Server |
| 2379/2380 | TCP | etcd |
| 9090 | TCP | Prometheus |
| 3000 | TCP | Grafana |

Well-Known Port 범위는 일반적으로 0~1023입니다.

### 22-3) Ephemeral Port

Ephemeral Port는 클라이언트가 서버에 접속할 때 임시로 사용하는 출발지 포트입니다.

예를 들어 사용자가 HTTPS 서버에 접속합니다.

```text
Client: 192.168.10.25:52344
Server: 203.0.113.10:443
```

여기서 `52344`가 Ephemeral Port입니다.

서버는 443 포트를 사용하지만, 클라이언트는 매 연결마다 임시 포트를 사용합니다.

Linux에서 범위 확인:

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
```

예시:

```text
32768 60999
```

### 22-4) Ephemeral Port 고갈

대량의 Outbound 연결이 발생하면 Ephemeral Port가 부족해질 수 있습니다.

대표 상황:

- 프록시 서버
- NAT Gateway
- 대량 HTTP Client
- Prometheus가 많은 Target을 Scrape
- API Gateway가 많은 Backend로 연결
- Kubernetes Node에서 많은 Pod가 외부로 통신

증상:

```text
Cannot assign requested address
connection timeout 증가
일부 외부 호출 실패
```

확인:

```bash
ss -s
ss -ant | awk '{print $1}' | sort | uniq -c
cat /proc/sys/net/ipv4/ip_local_port_range
```

## 23. Socket

### 23-1) Socket이란 무엇인가

Socket은 네트워크 통신의 끝점입니다.

TCP 연결은 일반적으로 다음 4가지 값으로 식별됩니다.

```text
Source IP
Source Port
Destination IP
Destination Port
```

예시:

```text
192.168.10.25:52344 -> 203.0.113.10:443
```

이 4-tuple이 다르면 같은 서버의 같은 포트로도 여러 연결이 동시에 존재할 수 있습니다.

### 23-2) 소켓 확인 명령어

```bash
ss -lntp
ss -antp
```

예시:

```text
LISTEN 0 4096 0.0.0.0:443 0.0.0.0:* users:(("nginx",pid=1234,fd=6))
ESTAB  0 0 192.168.10.25:52344 203.0.113.10:443
```

해석:

```text
LISTEN
→ 서버가 해당 포트에서 연결을 기다리는 중

ESTAB
→ TCP 연결이 수립된 상태
```

## 24. TCP

### 24-1) TCP란 무엇인가

TCP는 신뢰성 있는 연결 지향 프로토콜입니다.

특징:

- 연결 수립 필요
- 순서 보장
- 손실 재전송
- 흐름 제어
- 혼잡 제어
- 바이트 스트림 기반

HTTP/1.1, HTTP/2, SSH, 대부분의 HTTPS 트래픽은 TCP 위에서 동작합니다.

### 24-2) 3-Way Handshake

TCP 연결은 3-Way Handshake로 시작합니다.

```text
Client -> Server: SYN
Server -> Client: SYN-ACK
Client -> Server: ACK
```

의미:

```text
SYN
→ 연결 요청

SYN-ACK
→ 요청 수락 및 응답

ACK
→ 응답 확인
```

### 24-3) tcpdump로 보는 Handshake

```bash
sudo tcpdump -i any -nn tcp port 443
```

예시:

```text
192.168.10.25.52344 > 203.0.113.10.443: Flags [S]
203.0.113.10.443 > 192.168.10.25.52344: Flags [S.]
192.168.10.25.52344 > 203.0.113.10.443: Flags [.]
```

해석:

```text
[S]  = SYN
[S.] = SYN-ACK
[.]  = ACK
```

### 24-4) TCP 장애 패턴

| tcpdump 패턴 | 가능 원인 |
|---|---|
| SYN만 반복 | 서버 미응답, 방화벽 Drop, 라우팅 문제 |
| SYN → RST | 포트 닫힘, 서비스 미기동, 방화벽 Reject |
| SYN/SYN-ACK 후 ACK 없음 | 비대칭 라우팅, 중간 장비 문제 |
| 연결 후 RST | 애플리케이션 종료, 프로토콜 불일치 |
| 연결은 되지만 느림 | MTU, 혼잡, 윈도우, 서버 처리 지연 |

## 25. UDP

### 25-1) UDP란 무엇인가

UDP는 연결 수립 없이 데이터를 보내는 비연결형 프로토콜입니다.

특징:

- 3-Way Handshake 없음
- 순서 보장 없음
- 재전송 보장 없음
- 오버헤드가 작음
- 지연이 낮음

대표 사용 예:

- DNS
- DHCP
- NTP
- QUIC/HTTP/3
- 일부 게임/스트리밍

### 25-2) UDP 트러블슈팅이 어려운 이유

TCP는 Handshake가 있어 연결 성공/실패를 비교적 쉽게 볼 수 있습니다.

반면 UDP는 연결 개념이 없기 때문에 다음을 직접 확인해야 합니다.

```text
패킷이 나갔는가?
상대가 받았는가?
응답이 돌아왔는가?
중간에서 Drop되었는가?
애플리케이션이 응답하지 않는가?
```

확인 예시:

```bash
dig @8.8.8.8 google.com

sudo tcpdump -i any -nn udp port 53
```

## 26. DNS

### 26-1) DNS란 무엇인가

DNS는 도메인 이름을 IP 주소로 변환하는 시스템입니다.

```text
app.example.com -> 203.0.113.10
```

애플리케이션은 보통 도메인으로 접근하지만, 실제 네트워크는 IP 주소로 통신합니다. 따라서 DNS가 실패하면 서버가 정상이어도 접속할 수 없습니다.

### 26-2) DNS 조회 흐름

단순화하면 다음과 같습니다.

```text
Client
  ↓
Local Resolver
  ↓
Recursive DNS
  ↓
Root DNS
  ↓
TLD DNS
  ↓
Authoritative DNS
  ↓
IP 응답
```

실무에서는 보통 `/etc/resolv.conf`에 설정된 DNS 서버로 질의합니다.

```bash
cat /etc/resolv.conf
dig app.example.com
nslookup app.example.com
```

### 26-3) Kubernetes DNS

Kubernetes에서는 CoreDNS가 Service Discovery를 담당합니다.

예시:

```text
my-service.default.svc.cluster.local
```

Pod 안에서 Service 이름을 조회하면 CoreDNS가 ClusterIP를 반환합니다.

```bash
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local
```

## 27. NAT

### 27-1) NAT란 무엇인가

NAT(Network Address Translation)는 IP 주소 또는 포트를 변환하는 기술입니다.

대표 유형:

| 유형 | 의미 |
|---|---|
| SNAT | Source IP 변환 |
| DNAT | Destination IP 변환 |
| PAT | Port Address Translation |
| Masquerade | 동적 SNAT |
| Port Forwarding | 특정 포트를 내부 서버로 전달 |
| Hairpin NAT | 내부에서 외부 주소로 다시 내부 서비스 접근 |

NAT는 Private IP 환경에서 매우 중요합니다.

### 27-2) SNAT

SNAT(Source NAT)는 출발지 IP를 변환합니다.

예시:

```text
Before:
192.168.10.25:52344 -> 8.8.8.8:53

After SNAT:
203.0.113.10:40001 -> 8.8.8.8:53
```

용도:

- 내부 서버가 인터넷으로 나갈 때
- Kubernetes Pod가 외부 API에 접근할 때
- NAT Gateway를 통해 외부 통신할 때

### 27-3) DNAT

DNAT(Destination NAT)는 목적지 IP 또는 포트를 변환합니다.

예시:

```text
Before:
203.0.113.10:80

After DNAT:
192.168.10.50:8080
```

용도:

- Port Forwarding
- Load Balancer
- Kubernetes Service
- NodePort
- Ingress Controller 앞단 전달

### 27-4) PAT

PAT(Port Address Translation)는 IP뿐 아니라 Port까지 함께 변환하는 NAT입니다.

여러 내부 클라이언트가 하나의 공인 IP를 공유할 수 있는 이유가 PAT입니다.

```text
192.168.10.10:50001 -> 8.8.8.8:53
192.168.10.11:50002 -> 8.8.8.8:53
192.168.10.12:50003 -> 8.8.8.8:53

SNAT/PAT 후:

203.0.113.10:40001 -> 8.8.8.8:53
203.0.113.10:40002 -> 8.8.8.8:53
203.0.113.10:40003 -> 8.8.8.8:53
```

응답이 돌아오면 NAT 장비는 변환 테이블을 보고 원래 내부 클라이언트로 되돌립니다.

### 27-5) Masquerade

Masquerade는 동적 SNAT입니다.

출구 인터페이스의 IP를 자동으로 사용합니다.

Linux iptables 예시:

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Kubernetes에서는 Pod가 외부로 나갈 때 CNI나 kube-proxy 설정에 따라 Masquerade가 사용될 수 있습니다.

### 27-6) Hairpin NAT

Hairpin NAT는 내부 클라이언트가 외부 주소 또는 LoadBalancer 주소로 접근했는데, 실제 목적지가 다시 내부 서버인 경우 발생합니다.

예시:

```text
Client: 192.168.10.25
접속 주소: 203.0.113.10
실제 Backend: 192.168.10.50
```

흐름:

```text
192.168.10.25
  ↓ 외부 IP 203.0.113.10으로 요청
Load Balancer/NAT
  ↓ DNAT
192.168.10.50
```

이때 응답 경로가 NAT 장비를 다시 지나지 않으면 클라이언트는 예상과 다른 출발지에서 응답을 받게 됩니다. 그러면 연결이 깨질 수 있습니다.

Kubernetes에서도 Service IP 또는 LoadBalancer IP로 자기 자신 또는 같은 클러스터 내부 Backend에 접근할 때 Hairpin NAT 개념이 중요합니다.

## 28. Connection Tracking(conntrack)

### 28-1) conntrack이란 무엇인가

conntrack은 Linux Netfilter가 네트워크 연결 상태를 추적하는 기능입니다.

TCP는 연결 상태가 명확하지만, NAT와 방화벽 입장에서는 다음 정보를 계속 기억해야 합니다.

```text
누가 누구에게 연결했는가?
어떤 Source Port를 썼는가?
NAT 전 주소는 무엇인가?
NAT 후 주소는 무엇인가?
응답 패킷은 어디로 돌려보내야 하는가?
```

이 정보를 추적하는 것이 conntrack입니다.

### 28-2) conntrack이 필요한 이유

SNAT 예시:

```text
Before:
192.168.10.25:52344 -> 8.8.8.8:53

After:
203.0.113.10:40001 -> 8.8.8.8:53
```

응답은 이렇게 돌아옵니다.

```text
8.8.8.8:53 -> 203.0.113.10:40001
```

NAT 장비는 이 응답을 보고 원래 내부 주소로 되돌려야 합니다.

```text
8.8.8.8:53 -> 192.168.10.25:52344
```

이 매핑을 기억하는 테이블이 conntrack table입니다.

### 28-3) conntrack 확인 명령어

```bash
conntrack -L
conntrack -S
```

conntrack-tools가 없으면 설치가 필요할 수 있습니다.

```bash
sudo apt install conntrack
```

상태 확인:

```bash
cat /proc/sys/net/netfilter/nf_conntrack_max
cat /proc/sys/net/netfilter/nf_conntrack_count
```

### 28-4) conntrack table full 장애

트래픽이 많거나 짧은 연결이 폭증하면 conntrack table이 가득 찰 수 있습니다.

대표 증상:

```text
간헐적 연결 실패
DNS 질의 실패
Service 통신 불안정
외부 API 호출 timeout
NodePort/Service 연결 실패
```

커널 로그에서 다음과 같은 메시지가 보일 수 있습니다.

```text
nf_conntrack: table full, dropping packet
```

확인:

```bash
dmesg | grep -i conntrack
conntrack -S
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
```

### 28-5) Kubernetes와 conntrack

Kubernetes Service는 NAT와 conntrack에 크게 의존합니다.

예를 들어 ClusterIP Service는 다음과 같이 동작할 수 있습니다.

```text
Client Pod
  ↓
Service IP:Port
  ↓ DNAT
Backend Pod IP:TargetPort
```

응답 패킷이 다시 원래 Client에게 돌아가려면 conntrack이 NAT 상태를 기억해야 합니다.

따라서 conntrack 문제가 생기면 Kubernetes Service 장애처럼 보일 수 있습니다.

## 29. NAT와 Kubernetes Service의 관계

### 29-1) Service는 가상 IP다

Kubernetes Service의 ClusterIP는 실제 Pod가 가진 IP가 아닙니다.

```text
Service IP: 10.96.10.20
Backend Pod:
- 10.244.1.10
- 10.244.2.15
- 10.244.3.21
```

Client가 Service IP로 요청하면 kube-proxy 또는 eBPF datapath가 Backend Pod IP로 전달합니다.

```text
10.96.10.20:80
  ↓
10.244.2.15:8080
```

이 동작은 개념적으로 DNAT와 유사합니다.

### 29-2) kube-proxy iptables 모드

iptables 모드에서는 Service IP로 들어온 패킷을 iptables NAT 규칙으로 Backend Pod에 분산합니다.

```text
PREROUTING/OUTPUT
  ↓
KUBE-SERVICES
  ↓
KUBE-SVC-xxxxx
  ↓
KUBE-SEP-xxxxx
  ↓
DNAT to Pod IP
```

확인:

```bash
iptables -t nat -L -n -v
iptables -t nat -S | grep KUBE-SVC
```

### 29-3) Cilium/eBPF 환경

Cilium에서는 kube-proxy를 제거하고 eBPF로 Service Load Balancing을 처리할 수 있습니다.

이 경우 iptables에 KUBE-SVC 규칙이 없을 수 있습니다.

따라서 환경에 따라 확인 방법이 달라집니다.

```bash
kubectl -n kube-system get cm cilium-config -o yaml | grep kube-proxy
cilium service list
cilium bpf lb list
```

명령어는 Cilium CLI 설치 여부와 버전에 따라 다를 수 있습니다.

## 30. 요청 흐름 전체 예시

### 30-1) 같은 Subnet 서버로 HTTP 요청

```text
Client: 192.168.10.25/24
Server: 192.168.10.50/24
Gateway: 192.168.10.1
```

요청:

```text
curl http://192.168.10.50
```

흐름:

```text
1. Destination IP가 192.168.10.50인지 확인
2. 내 IP 192.168.10.25/24 기준 같은 Subnet인지 판단
3. Gateway를 거치지 않음
4. ARP로 192.168.10.50의 MAC 확인
5. Ethernet Frame 생성
6. TCP 80 3-Way Handshake
7. HTTP Request 전송
8. HTTP Response 수신
```

### 30-2) 다른 Subnet 서버로 HTTP 요청

```text
Client: 192.168.10.25/24
Server: 192.168.20.50/24
Gateway: 192.168.10.1
```

흐름:

```text
1. Destination IP가 192.168.20.50인지 확인
2. 내 Subnet이 아님
3. Routing Table에서 default gateway 선택
4. ARP로 Gateway MAC 확인
5. Destination IP는 192.168.20.50 그대로 유지
6. Destination MAC은 Gateway MAC으로 설정
7. Gateway가 다음 경로로 라우팅
8. Server까지 전달
```

### 30-3) 인터넷으로 HTTPS 요청

```text
curl https://example.com
```

흐름:

```text
1. DNS로 example.com IP 조회
2. Routing Table에서 default route 선택
3. Gateway MAC을 ARP로 확인
4. TCP 443 연결
5. TLS Handshake
6. HTTP Request 전송
7. 응답 수신
```

사설망에서는 중간에 SNAT가 들어갈 수 있습니다.

```text
192.168.10.25:52344
  ↓ SNAT
203.0.113.10:40001
```

### 30-4) Kubernetes Pod가 외부 API 호출

```text
Pod IP: 10.244.1.15
Node IP: 192.168.10.84
External API: 203.0.113.100:443
```

흐름:

```text
1. Pod에서 DNS 조회
2. Pod Routing Table 확인
3. veth를 통해 Node Network로 이동
4. Node에서 외부 경로 확인
5. 필요 시 Pod IP를 Node IP로 SNAT/Masquerade
6. Gateway로 전달
7. 외부 API 응답
8. conntrack을 통해 원래 Pod로 응답 복원
```

## 31. 실무 트러블슈팅 기본 순서

### 31-1) 전체 점검 흐름

네트워크 장애는 아래 순서로 좁혀가면 됩니다.

```text
1. 이름 해석 문제인가?
2. 목적지 IP가 맞는가?
3. Route가 있는가?
4. Gateway 또는 다음 홉에 도달하는가?
5. ARP가 되는가?
6. TCP/UDP Port가 열려 있는가?
7. NAT/conntrack 문제가 있는가?
8. MTU 문제인가?
9. TLS 문제인가?
10. HTTP Host/Path/Header 문제인가?
11. 애플리케이션 로그에 원인이 있는가?
```

### 31-2) DNS 확인

```bash
dig app.example.com
nslookup app.example.com
cat /etc/resolv.conf
```

Kubernetes 내부:

```bash
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local
```

### 31-3) L3 확인

```bash
ip addr
ip route
ip route get <destination-ip>
ping <destination-ip>
traceroute <destination-ip>
tracepath <destination-ip>
```

### 31-4) ARP 확인

```bash
ip neigh
arp -n
arping <gateway-ip>
```

### 31-5) L4 확인

```bash
ss -lntp
ss -antp
nc -vz <host> <port>
telnet <host> <port>
```

### 31-6) NAT/conntrack 확인

```bash
iptables -t nat -L -n -v
iptables -t nat -S

conntrack -S
conntrack -L | head

cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
```

### 31-7) MTU 확인

```bash
ip link show
tracepath <destination>
ping -M do -s 1472 <destination>
```

### 31-8) Packet Capture

```bash
tcpdump -i any -nn host <ip>
tcpdump -i any -nn port 443
tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0'
```

Kubernetes Node에서 확인할 때는 다음처럼 봅니다.

```bash
# Node 전체 인터페이스에서 특정 Pod IP 확인
tcpdump -i any -nn host <pod-ip>

# Service 통신 확인
tcpdump -i any -nn host <service-ip>

# DNS 확인
tcpdump -i any -nn port 53
```

## 32. 장애 패턴별 빠른 해석

### 32-1) DNS 실패

증상:

```text
Could not resolve host
Name or service not known
SERVFAIL
NXDOMAIN
```

확인:

```bash
dig <domain>
cat /etc/resolv.conf
```

Kubernetes:

```bash
kubectl -n kube-system get pod -l k8s-app=kube-dns
kubectl -n kube-system logs deploy/coredns
```

### 32-2) Routing 실패

증상:

```text
Network is unreachable
No route to host
```

확인:

```bash
ip route
ip route get <ip>
```

### 32-3) ARP 실패

증상:

```text
같은 Subnet인데 통신 안 됨
Gateway MAC 확인 실패
간헐적 통신 장애
```

확인:

```bash
ip neigh
arping <ip>
tcpdump -i any -nn arp
```

### 32-4) TCP 포트 실패

증상:

```text
Connection refused
Connection timed out
```

해석:

```text
Connection refused
→ 대상까지 도달했지만 포트가 닫혔거나 RST 반환

Connection timed out
→ 중간에서 Drop, 라우팅 실패, 방화벽 Drop 가능성
```

확인:

```bash
nc -vz <host> <port>
ss -lntp
tcpdump -i any -nn tcp port <port>
```

### 32-5) MTU 문제

증상:

```text
작은 요청은 됨
큰 요청은 멈춤
TLS Handshake 실패
파일 업로드 실패
일부 API만 timeout
```

확인:

```bash
tracepath <host>
ping -M do -s 1472 <host>
ip link show
```

### 32-6) conntrack 문제

증상:

```text
전체가 아니라 일부 연결만 실패
트래픽 많을 때만 장애
Service 통신 간헐 실패
DNS 간헐 실패
```

확인:

```bash
dmesg | grep -i conntrack
conntrack -S
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
```

## 33. Kubernetes, HAProxy, Ceph와 연결해서 이해하기

### 33-1) Kubernetes

Kubernetes 네트워크는 다음 기본 개념 위에 있습니다.

| Kubernetes 개념 | 기반 네트워크 개념 |
|---|---|
| Pod IP | IP Address, Routing |
| Service IP | DNAT, Load Balancing |
| NodePort | DNAT, Port |
| Ingress | L7 HTTP Routing |
| CNI | veth, routing, encapsulation, eBPF |
| CoreDNS | DNS |
| NetworkPolicy | L3/L4 Policy |
| kube-proxy | iptables/IPVS/eBPF, conntrack |

### 33-2) HAProxy

HAProxy는 L4 또는 L7 Proxy로 동작할 수 있습니다.

| HAProxy 모드 | 보는 정보 | 기반 개념 |
|---|---|---|
| mode tcp | IP, Port, TCP 연결 | L4, TCP, Session |
| mode http | Host, Path, Header | L7, HTTP |
| SSL Pass Through | TLS 내용을 해석하지 않음 | L4 TCP |
| TLS Termination | TLS 복호화 후 HTTP 확인 | L7 HTTP |

HAProxy Health Check 장애도 결국 다음을 확인해야 합니다.

```text
DNS
Routing
ARP
TCP Port
TLS
HTTP Status Code
```

### 33-3) Ceph

Ceph도 분산 시스템이므로 네트워크에 매우 민감합니다.

관련 개념:

| Ceph 상황 | 네트워크 개념 |
|---|---|
| MON 통신 실패 | Routing, Port, Firewall |
| OSD 간 통신 지연 | MTU, Packet Loss, TCP |
| Dashboard 접속 실패 | HTTP/HTTPS, TLS, L7 |
| RBD I/O 지연 | TCP, MTU, Packet Loss |
| Public/Cluster Network 분리 | Subnet, Routing, Interface |

Ceph에서는 MTU 불일치, 라우팅 오류, NIC 문제, 방화벽 문제 등이 성능 저하나 장애로 이어질 수 있습니다.

## 34. 핵심 요약

네트워크 기초를 단순 암기하면 실무 장애를 해결하기 어렵습니다.

반드시 다음 흐름으로 이해해야 합니다.

```text
HTTP 요청
  ↓
TCP Segment
  ↓
IP Packet
  ↓
Routing Table 조회
  ↓
Next Hop 결정
  ↓
ARP로 MAC 확인
  ↓
Ethernet Frame 생성
  ↓
Switch/Gateway 전달
  ↓
NAT/conntrack 처리 가능
  ↓
목적지 도착
```

장애 분석은 반대로 확인합니다.

```text
DNS가 되는가?
IP가 맞는가?
Route가 있는가?
ARP가 되는가?
TCP Port가 열려 있는가?
NAT/conntrack이 정상인가?
MTU 문제가 없는가?
HTTP 응답이 정상인가?
```