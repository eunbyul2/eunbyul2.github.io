---
layout: post
title: "[Learning eBPF] Chapter 8. eBPF for Networking"
date: 2026-09-12 19:38:00 +0900
categories: [eBPF, Networking]
tags: [eBPF, XDP, TC, Cilium, Kubernetes, Network, Load Balancing, Network Policy, WireGuard, IPsec]
published: true
---

# Learning eBPF Chapter 8

## eBPF for Networking

Chapter 8에서는 **eBPF가 네트워크 패킷 처리에 어떻게 사용되는지**를 다룬다.

앞 장에서는 eBPF Program Type과 Attachment Point를 살펴봤다면, 이번 장에서는 실제 Network Stack의 여러 지점에 eBPF Program을 연결해서 패킷을 **확인하고, 버리고, 수정하고, 전달하는 방법**을 살펴본다.

네트워크 환경마다 필요한 기능은 다르다. 통신 환경에서는 SRv6 같은 특수한 기능이 필요할 수 있고, Kubernetes에서는 기존 시스템과의 연결이나 Pod 간 통신, Service Load Balancing, Network Policy 등이 필요하다. 또 별도의 Hardware Load Balancer가 하던 역할을 일반 서버에서 XDP Program으로 처리할 수도 있다.

이러한 기능을 모두 Linux Kernel 자체에 넣을 필요 없이, 필요한 기능만 eBPF Program으로 구현해서 동적으로 적용할 수 있다는 점이 eBPF의 장점이다.

실제로 Cilium은 Kubernetes Networking, Load Balancing, Network Policy 등에 eBPF를 사용하고 있으며 Meta와 Cloudflare 역시 대규모 환경에서 eBPF Networking을 활용하고 있다.

---

## 1. Packet Drops

네트워크 보안에서는 들어오는 Packet을 확인한 뒤 **허용할지 Drop할지 결정하는 기능**이 많이 사용된다.

대표적인 예시는 다음과 같다.

- Firewall
- DDoS Protection
- Packet-of-death 취약점 완화

### Firewall

Firewall은 Packet의 Source/Destination IP Address나 Port Number 등을 기준으로 해당 Packet을 통과시킬지 차단할지 결정한다.

즉 가장 단순하게 생각하면 다음과 같은 형태이다.

```text
Packet 도착
   ↓
Source / Destination / Port 확인
   ↓
Rule과 일치?
   ↓
허용 → PASS
차단 → DROP
```

### DDoS Protection

DDoS Protection은 단순히 특정 IP를 차단하는 것보다 조금 더 복잡하다.

예를 들어 특정 Source에서 Packet이 비정상적으로 빠른 속도로 들어오는지 확인하거나, Packet 내용에서 공격으로 의심되는 특징을 찾아낼 수도 있다.

즉 Packet 하나만 보는 것이 아니라 **Traffic의 패턴이나 상태를 같이 확인할 수도 있다.**

### Packet-of-death

Packet-of-death는 특정하게 조작된 Packet을 Kernel이 안전하게 처리하지 못해 문제가 발생하는 취약점의 한 종류이다.

이런 Kernel 취약점이 발견되면 전통적으로는 수정된 Kernel을 설치해야 하고, 경우에 따라 재부팅까지 필요할 수 있다.

하지만 eBPF Program으로 문제가 되는 Packet의 특징을 확인해서 미리 Drop한다면, Kernel 자체를 바로 교체하지 않더라도 해당 Host를 빠르게 보호할 수 있다.

즉 eBPF를 이용하면 **문제가 되는 Packet이 Kernel의 더 깊은 Network Stack까지 들어오기 전에 차단하는 방법**을 만들 수 있다.

---

# 2. XDP Program Return Codes

XDP Program은 Network Interface로 Packet이 들어오면 실행된다.

Program은 Packet을 확인하고 처리를 마친 뒤 **Return Code로 해당 Packet을 다음에 어떻게 처리할지 결정한다.**

| Return Code | 의미 |
|---|---|
| `XDP_PASS` | Packet을 일반 Linux Network Stack으로 전달 |
| `XDP_DROP` | Packet을 즉시 폐기 |
| `XDP_TX` | Packet이 들어온 동일 Interface로 다시 전송 |
| `XDP_REDIRECT` | 다른 Network Interface 등으로 전달 |
| `XDP_ABORTED` | 예상하지 못한 오류 상황으로 판단하고 Packet 폐기 |

예를 들어 Firewall처럼 Packet을 통과시킬지 버릴지만 판단한다면 다음과 같이 작성할 수 있다.

```c
SEC("xdp")
int hello(struct xdp_md *ctx)
{
    bool drop;

    drop = <패킷을 확인하고 Drop 여부 결정>;

    if (drop)
        return XDP_DROP;
    else
        return XDP_PASS;
}
```

여기서 중요한 점은 XDP Program의 `return` 값이 단순한 함수 종료 값이 아니라 **Packet의 처리 결과를 Kernel에 전달하는 값**이라는 것이다.

예를 들어 `XDP_DROP`을 반환하면 Packet은 그 자리에서 버려지고, `XDP_PASS`를 반환하면 원래처럼 Linux Network Stack의 다음 단계로 전달된다.

---

# 3. XDP Packet Parsing

Packet을 Drop할지 결정하려면 먼저 **Packet 안에 어떤 정보가 들어 있는지 읽을 수 있어야 한다.**

XDP Program은 다음과 같이 `struct xdp_md *ctx`를 Context로 전달받는다.

```c
struct xdp_md {
    __u32 data;
    __u32 data_end;
    __u32 data_meta;

    __u32 ingress_ifindex;
    __u32 rx_queue_index;
    __u32 egress_ifindex;
};
```

여기서 가장 먼저 알아야 할 값은 `data`와 `data_end`이다.

- `data`: Packet이 시작되는 Memory 위치
- `data_end`: Packet이 끝나는 Memory 위치
- `data_meta`: Packet 앞쪽의 Metadata 영역을 가리키는 값

`data`와 `data_end`는 타입만 보면 `__u32`지만 eBPF Program 안에서는 Packet Memory의 시작과 끝을 나타내는 값으로 사용한다.

Packet은 하나의 연속된 Byte Data이며 대략 다음과 같은 순서로 Header가 배치되어 있다.

```text
data
 ↓
+-------------------+
| Ethernet Header   |
+-------------------+
| IP Header         |
+-------------------+
| TCP/UDP/ICMP      |
+-------------------+
| Payload           |
+-------------------+
                    ↑
                 data_end
```

따라서 eBPF Program은 `data`부터 시작해서 Ethernet Header, IP Header 등을 순서대로 해석한다.

---

## Verifier와 Packet 범위 검사

이 부분에서 Chapter 6에서 배운 **Verifier**가 다시 중요해진다.

예를 들어 다음처럼 Ethernet Header를 읽으려고 한다.

```c
struct ethhdr *eth = data;
```

하지만 `data`가 가리키는 Packet이 실제로 Ethernet Header 크기보다 짧을 수도 있다.

그래서 Header를 읽기 전에 다음과 같이 범위를 확인해야 한다.

```c
if (data + sizeof(struct ethhdr) > data_end)
    return 0;
```

즉,

```text
data + 읽으려는 Header 크기
```

가 `data_end`를 넘어가지 않는지 확인한다.

이 검사가 없다면 Packet 범위를 벗어난 Memory Access가 발생할 수 있기 때문에 Verifier가 Program Loading을 거부할 수 있다.

이전 Chapter에서 배운 **"Pointer를 Dereference하기 전에 안전한 범위인지 검사해야 한다"**는 내용이 Network Packet Parsing에서도 그대로 적용된다.

---

## data_meta

`data_meta`와 `data` 사이에는 Packet과 관련된 추가 Metadata를 저장할 수 있는 공간이 있다.

이 Metadata는 하나의 Packet이 Network Stack을 지나면서 여러 eBPF Program에서 처리될 때 **Program 간 정보를 전달하는 용도**로 사용할 수 있다.

즉 여러 eBPF Program이 동일한 Packet을 서로 다른 Hook에서 처리해야 하는 경우, 앞에서 처리한 Program의 정보를 뒤쪽 Program이 활용할 수 있다.

이 개념은 뒤에서 Cilium처럼 여러 eBPF Program이 함께 동작하는 구조와도 연결된다.

---

# 4. ICMP Packet 확인 예제

책에서는 `ping()`이라는 XDP Program을 이용해서 ICMP Packet이 들어오면 Trace를 남기는 간단한 예제를 보여준다.

```c
SEC("xdp")
int ping(struct xdp_md *ctx)
{
    long protocol = lookup_protocol(ctx);

    if (protocol == 1) // ICMP
    {
        bpf_printk("Hello ping");
    }

    return XDP_PASS;
}
```

`lookup_protocol()` 함수가 Packet을 Parsing해서 어떤 Layer 4 Protocol인지 확인한다.

대표적인 Protocol Number는 다음과 같다.

| Protocol Number | Protocol |
|---:|---|
| `1` | ICMP |
| `6` | TCP |
| `17` | UDP |

따라서 `protocol == 1`이면 ICMP Packet이라는 뜻이다.

책에서는 이 Program을 Loopback Interface인 `lo`에 연결하고 `ping localhost`를 실행한다.

이 경우 약 1초마다 Trace가 두 줄씩 발생한다.

그 이유는 Loopback Interface가 다음 두 Packet을 모두 받기 때문이다.

```text
ICMP Echo Request
        ↓
      localhost
        ↓
ICMP Echo Reply
```

즉 Ping 요청 Packet과 응답 Packet이 모두 Loopback Interface를 지나면서 XDP Program을 실행한다.

---

## Ping Packet Drop

여기에 다음과 같이 `XDP_DROP`을 추가할 수 있다.

```c
if (protocol == 1) // ICMP
{
    bpf_printk("Hello ping");
    return XDP_DROP;
}

return XDP_PASS;
```

이제 ICMP Echo Request가 들어오면 XDP 단계에서 바로 Drop된다.

그러면 Request가 Network Stack의 뒤쪽까지 도달하지 않기 때문에 Kernel이 Echo Reply를 만들지 못한다.

그래서 기존에는 Request와 Response 때문에 Trace가 두 줄씩 발생했지만, Drop 이후에는 **Request에 대한 Trace만 한 번씩 발생한다.**

이 예제는 코드 몇 줄만으로 실제 Network 동작을 바꿀 수 있다는 것을 보여준다.

---

# 5. lookup_protocol()과 Packet Header Parsing

책의 `lookup_protocol()` 함수는 다음과 같은 순서로 Packet을 확인한다.

```c
unsigned char lookup_protocol(struct xdp_md *ctx)
{
    unsigned char protocol = 0;

    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;

    if (data + sizeof(struct ethhdr) > data_end)
        return 0;

    if (bpf_ntohs(eth->h_proto) == ETH_P_IP)
    {
        struct iphdr *iph = data + sizeof(struct ethhdr);

        if (data + sizeof(struct ethhdr)
            + sizeof(struct iphdr) <= data_end)
            protocol = iph->protocol;
    }

    return protocol;
}
```

먼저 `ctx->data`와 `ctx->data_end`를 이용해서 Packet의 시작과 끝을 가져온다.

```c
void *data = (void *)(long)ctx->data;
void *data_end = (void *)(long)ctx->data_end;
```

그리고 Packet의 처음을 Ethernet Header라고 해석한다.

```c
struct ethhdr *eth = data;
```

하지만 곧바로 Ethernet Header 내용을 읽지는 않고, 먼저 Header 전체가 Packet 안에 들어 있는지 검사한다.

```c
if (data + sizeof(struct ethhdr) > data_end)
    return 0;
```

그다음 Ethernet Header의 `h_proto` 값을 보고 IPv4 Packet인지 확인한다.

```c
if (bpf_ntohs(eth->h_proto) == ETH_P_IP)
```

IPv4 Packet이라면 Ethernet Header 바로 뒤에서 IP Header가 시작된다.

```c
struct iphdr *iph = data + sizeof(struct ethhdr);
```

IP Header 역시 Packet 범위를 벗어나지 않는지 검사한 다음:

```c
if (data + sizeof(struct ethhdr)
    + sizeof(struct iphdr) <= data_end)
```

IP Header의 `protocol` 값을 읽는다.

```c
protocol = iph->protocol;
```

결국 이 함수가 하는 일은 다음과 같다.

```text
Packet 시작
   ↓
Ethernet Header 확인
   ↓
IPv4 Packet인가?
   ↓
IP Header 확인
   ↓
Protocol 값 확인
   ↓
ICMP / TCP / UDP 구분
```

---

# 6. bpf_ntohs()와 Byte Order

Network Packet Header 값을 읽을 때는 **Byte Order**도 고려해야 한다.

Network Protocol에서는 여러 Byte로 이루어진 값을 Big Endian 방식인 **Network Byte Order**로 표현한다.

반면 많은 CPU는 Little Endian을 사용한다.

따라서 Network Packet의 2Byte 값을 Host에서 그대로 읽으면 예상한 값과 다르게 보일 수 있다.

책의 예제에서는 다음 함수를 사용한다.

```c
bpf_ntohs(eth->h_proto)
```

`ntohs`는 Network To Host Short의 의미로, 2Byte 값을 Network Byte Order에서 Host Byte Order로 변환한다.

따라서 Packet Header에서 2Byte 이상의 Multi-byte 값을 읽을 때는 Byte Order를 신경 써야 한다.

---

# 7. Load Balancing and Forwarding

XDP Program은 Packet을 읽는 것뿐만 아니라 **Packet의 내용을 직접 수정할 수도 있다.**

책에서는 간단한 Load Balancer 예제를 통해 이를 설명한다.

구조는 다음과 같다.

```text
           Client
              ↓
       Load Balancer
         ↙       ↘
 Backend A     Backend B
```

각 구성 요소는 Container로 실행되고, Load Balancer Container의 `eth0`에 XDP Program을 연결한다.

Client에서 Packet이 들어오면 XDP Program이 Backend A 또는 Backend B를 선택한 다음 Packet Header의 주소를 변경해서 해당 Backend로 보낸다.

---

## Backend 선택

예제에서는 다음과 같이 간단한 Pseudorandom 방식으로 Backend를 선택한다.

```c
char be = BACKEND_A;

if (bpf_get_prandom_u32() % 2)
    be = BACKEND_B;
```

실제 Production Load Balancer라면 Health Check, Connection State, Hashing 등 훨씬 복잡한 요소를 고려하겠지만 이 예제는 XDP가 Packet을 직접 변경할 수 있다는 것을 이해하기 위한 코드이다.

---

## Destination Address 변경

선택된 Backend에 맞게 Destination IP와 MAC Address를 변경한다.

```c
iph->daddr = IP_ADDRESS(be);
eth->h_dest[5] = be;
```

반대로 Backend에서 돌아온 Response Packet이라고 판단되면 Destination을 다시 Client로 변경한다.

```text
Client → Load Balancer → Backend
Backend → Load Balancer → Client
```

또한 Packet의 Source Address는 Load Balancer 주소로 변경한다.

```c
iph->saddr = IP_ADDRESS(LB);
eth->h_source[5] = LB;
```

이렇게 하면 Client나 Backend의 입장에서는 Packet이 Load Balancer를 통해 전달되고 있다는 형태가 유지된다.

---

## IP Checksum 재계산

IP Header를 수정했기 때문에 Checksum도 다시 계산해야 한다.

```c
iph->check = iph_csum(iph);
```

IP Header의 Checksum은 Header 내용으로 계산되므로 Source/Destination IP를 바꾼 뒤 기존 Checksum을 그대로 사용하면 올바르지 않다.

따라서 Header를 수정한 후 Checksum을 다시 계산한다.

마지막으로 다음 값을 반환한다.

```c
return XDP_TX;
```

`XDP_TX`는 Packet을 **들어왔던 동일 Interface로 다시 내보내는 동작**이다.

이 예제는 실제 Production Load Balancer보다 훨씬 단순하지만, XDP Program이 Packet Header를 직접 변경함으로써 Load Balancing과 Forwarding 기능을 구현할 수 있다는 점을 보여준다.

---

# 8. Network Namespace와 XDP Attachment

예제의 XDP Program은 `bpftool`을 이용해 Load Balancer Container의 `eth0`에 연결된다.

여기서 흥미로운 점은 eBPF Program 자체는 **Host의 하나뿐인 Kernel에 Load**된다는 것이다.

Container마다 별도의 Kernel이 있는 것이 아니다.

하지만 Container는 서로 다른 Network Namespace를 사용할 수 있기 때문에, 같은 이름인 `eth0`도 각 Namespace 안에서는 서로 다른 Network Interface를 의미할 수 있다.

즉:

```text
하나의 Linux Kernel
        │
        ├─ Host Network Namespace
        │
        ├─ Container A Network Namespace
        │      └─ eth0
        │
        └─ Load Balancer Network Namespace
               └─ eth0 ← XDP Attach
```

처럼 이해할 수 있다.

eBPF Program은 Kernel에 존재하지만 **Attachment Point는 특정 Network Namespace의 Interface가 될 수 있다.**

---

# 9. XDP Offloading

XDP는 처음부터 Network Packet을 가능한 한 빠르게 처리하기 위한 아이디어에서 출발했다.

일부 NIC는 **XDP Offload**를 지원해서 eBPF Program을 NIC 자체의 Processor에서 실행할 수도 있다.

이 경우 Packet은 Host Kernel이나 CPU까지 도달하기 전에 처리된다.

```text
Packet
  ↓
NIC
  ↓
XDP Program 실행
  ↓
DROP / REDIRECT / TX
```

예를 들어 Packet을 Drop하거나 같은 Physical Interface로 다시 보내는 작업이라면 Host Kernel이 해당 Packet을 전혀 처리하지 않아도 될 수 있다.

따라서 Host CPU Cycle도 사용하지 않는다.

모든 NIC가 Full XDP Offload를 지원하는 것은 아니다.

하지만 Full Offload가 없더라도 많은 NIC Driver가 XDP Hook을 지원하기 때문에 Packet Processing에 필요한 Memory Copy를 줄이고 일반 Linux Network Stack보다 이른 위치에서 처리할 수 있다.

이러한 특성이 XDP를 고성능 Load Balancing이나 DDoS Protection 같은 용도로 활용할 수 있게 한다.

---

# 10. Traffic Control (TC)

XDP가 Packet이 Network Stack에 들어가기 전 매우 이른 시점에 실행된다면, **TC는 Packet이 이미 Linux Network Stack으로 들어온 뒤의 단계에서 동작한다.**

이 시점에서 Packet은 Kernel의 `sk_buff` 구조체로 관리된다.

TC에 연결된 eBPF Program은 다음처럼 `struct __sk_buff *skb`를 Context로 받는다.

```c
int tc_drop(struct __sk_buff *skb)
{
    bpf_trace_printk("[tc] dropping packet\n");

    return TC_ACT_SHOT;
}
```

XDP에서 `xdp_md`를 사용한 것과 달리 TC에서는 `__sk_buff`를 사용한다.

그 이유는 XDP가 실행되는 시점에는 아직 `sk_buff`가 생성되기 전이기 때문이다.

---

## TC는 무엇을 위한 기능인가?

Traffic Control은 원래 Network Traffic의 **Scheduling과 QoS를 제어하기 위한 Linux 기능**이다.

예를 들어 Application마다 Bandwidth를 제한하거나, Latency에 민감한 Traffic을 우선 처리하는 것처럼 Packet 처리 순서나 정책을 세밀하게 조절할 수 있다.

기존 TC에서는 크게 다음 요소가 사용된다.

- `qdisc` : Queuing Discipline
- `classifier` : Packet을 분류
- `action` : 분류 결과에 따라 어떤 처리를 할지 결정

eBPF Program은 TC에서 Classifier로 연결될 수 있지만, Program 안에서 직접 Action까지 결정할 수도 있다.

---

## TC Return Code

대표적인 Return Code는 다음과 같다.

| Return Code | 의미 |
|---|---|
| `TC_ACT_SHOT` | Packet Drop |
| `TC_ACT_UNSPEC` | 현재 Program이 없었던 것처럼 다음 Classifier로 진행 |
| `TC_ACT_OK` | 다음 Network Stack 단계로 전달 |
| `TC_ACT_REDIRECT` | 다른 Network Device의 Ingress/Egress Path로 Redirect |

XDP와 마찬가지로 TC에서도 Program의 Return Code가 Packet의 다음 동작을 결정한다.

---

# 11. XDP와 TC 비교

둘 다 Packet을 읽고 수정하고 Drop하거나 Redirect할 수 있지만 실행 위치와 사용할 수 있는 Context가 다르다.

| 구분 | XDP | TC |
|---|---|---|
| 실행 위치 | Network Stack 진입 전의 매우 이른 단계 | Network Stack에 들어온 이후 |
| Context | `struct xdp_md` | `struct __sk_buff` |
| Packet 구조 | 아직 `sk_buff` 생성 전 | `sk_buff` 사용 가능 |
| Ingress | 가능 | 가능 |
| Egress | 일반적인 XDP Hook에서는 불가 | 가능 |
| 주요 장점 | 빠른 Drop/Redirect | Ingress/Egress 및 `sk_buff` 기반 처리 |

따라서 무조건 XDP가 더 좋거나 TC가 더 좋다고 볼 수는 없다.

Packet을 최대한 빨리 Drop하거나 Redirect하고 싶다면 XDP가 유리할 수 있고, Egress Packet을 처리하거나 `sk_buff`의 정보가 필요하다면 TC가 적합할 수 있다.

---

# 12. TC에서 ICMP Packet Drop

TC에서도 XDP 예제와 비슷하게 ICMP Packet을 찾아서 Drop할 수 있다.

```c
int tc(struct __sk_buff *skb)
{
    void *data = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;

    if (is_icmp_ping_request(data, data_end)) {
        return TC_ACT_SHOT;
    }

    return TC_ACT_OK;
}
```

`__sk_buff`에도 Packet의 시작과 끝을 나타내는 `data`, `data_end`가 있기 때문에 Packet Parsing 방법 자체는 XDP와 비슷하다.

그리고 여기서도 Verifier를 통과하려면 Header를 읽기 전에 반드시 `data_end`를 이용해 Memory Range를 검사해야 한다.

---

# 13. TC에서 직접 Ping Response 만들기

책에서는 Packet을 단순히 Drop하는 것보다 한 단계 더 나아가, TC eBPF Program이 **ICMP Echo Request를 직접 Echo Reply로 변경해서 응답하는 예제**를 보여준다.

대략 다음 순서이다.

1. ICMP Echo Request인지 확인
2. Source/Destination MAC Address 교환
3. Source/Destination IP Address 교환
4. ICMP Type을 Echo Request에서 Echo Reply로 변경
5. 수정된 Packet의 Clone을 원래 Interface로 전송
6. Original Packet은 Drop

핵심 코드는 다음과 같다.

```c
swap_mac_addresses(skb);
swap_ip_addresses(skb);

update_icmp_type(skb, 8, 0);

bpf_clone_redirect(skb, skb->ifindex, 0);

return TC_ACT_SHOT;
```

ICMP Type `8`은 Echo Request, `0`은 Echo Reply를 의미한다.

`bpf_clone_redirect()`를 사용해서 수정된 Packet의 Clone을 해당 Interface로 다시 전송한다.

이미 Clone으로 Response를 보냈기 때문에 Original Packet은 `TC_ACT_SHOT`으로 Drop한다.

일반적인 Ping이라면 Request가 Kernel Network Stack의 더 뒤쪽까지 이동해서 Kernel이 Reply를 생성한다.

하지만 이 예제에서는 eBPF Program이 그 기능을 먼저 처리한다.

즉 eBPF가 기존 Network Stack의 일부 기능을 대신할 수 있다는 것을 보여준다.

---

# 14. Kernel에서 처리하는 것의 장점

Network 기능 중에는 User Space Service에서 처리되는 것들도 많다.

하지만 가능한 기능을 eBPF Program으로 Kernel 안에서 처리하면 Packet이 전체 Network Stack을 모두 통과해서 User Space까지 이동할 필요가 줄어든다.

```text
기존 방식

Packet
 ↓
Kernel Network Stack
 ↓
User Space Service
 ↓
다시 Kernel
 ↓
Network
```

반면 일부 처리를 eBPF에서 끝낼 수 있다면:

```text
Packet
 ↓
eBPF Program
 ↓
바로 처리
```

와 같이 경로를 줄일 수 있다.

그렇다고 모든 기능을 반드시 eBPF로 구현해야 하는 것은 아니다.

eBPF Program이 처리하기 어려운 복잡한 Packet은 `TC_ACT_OK` 등으로 기존 Network Stack이나 User Space Service에 넘길 수 있다.

즉 기존 방식과 eBPF 방식을 함께 사용할 수 있고, 시간이 지나면서 적합한 기능을 점진적으로 eBPF로 옮길 수도 있다.

---

# 15. Packet Encryption and Decryption

암호화된 Network Traffic은 Network Stack에서 보면 일반적으로 암호화된 데이터만 보인다.

하지만 암호화도 결국 Application 내부의 어느 시점에서는 평문으로 존재한다.

예를 들어:

```text
전송

Application Data
      ↓
평문
      ↓
SSL/TLS Library
      ↓
암호화
      ↓
Socket
      ↓
Network
```

수신은 반대이다.

```text
Network
   ↓
Socket
   ↓
암호화된 데이터
   ↓
SSL/TLS Library
   ↓
복호화
   ↓
평문
   ↓
Application
```

따라서 eBPF Program을 **암호화 직전이나 복호화 직후의 User Space 함수에 Attach**하면 평문을 관찰할 수 있다.

이때 인증서를 별도로 이용해 Traffic을 다시 복호화하는 것이 아니라, Application이 이미 복호화해서 사용하려는 순간의 데이터를 보는 방식이다.

---

# 16. User Space SSL Libraries

Application이 OpenSSL을 사용한다면 대표적으로 다음 함수가 사용된다.

- `SSL_write()` : 평문 데이터를 암호화해서 전송
- `SSL_read()` : 받은 암호화 데이터를 복호화해서 Application에 전달

eBPF에서는 User Space 함수에 연결하는 **uprobe / uretprobe**를 이용할 수 있다.

특히 `SSL_read()`는 함수가 시작될 때와 끝날 때 필요한 정보가 다르다.

```text
SSL_read() 진입
      ↓
uprobe
      ↓
Buffer Pointer 확인
      ↓
Map에 Pointer 저장
      ↓
SSL_read() 실행
      ↓
복호화된 평문이 Buffer에 저장
      ↓
uretprobe
      ↓
Map에서 Pointer 다시 조회
      ↓
평문 Data 읽기
```

함수 진입 시점에는 Buffer Pointer를 알 수 있지만 아직 복호화된 Data가 들어 있지 않다.

반대로 함수가 끝난 시점에는 Buffer에 Data가 들어 있지만, 함수 Parameter가 저장돼 있던 Register 값은 이미 바뀌었을 수 있다.

그래서 Entry Probe에서 Buffer Pointer를 eBPF Map에 저장하고, Return Probe에서 다시 조회하는 방식을 사용한다.

이 예제는 eBPF Map이 단순한 설정 저장소가 아니라 **서로 다른 시점에 실행되는 eBPF Program 사이에서 상태를 전달하는 용도**로도 사용될 수 있다는 것을 보여준다.

---

## SSL uprobe 방식의 한계

이 방식은 모든 Application에서 무조건 사용할 수 있는 것은 아니다.

Application이 OpenSSL이 아닌 다른 SSL Library를 사용한다면 Hook Point가 달라진다.

또 Container 환경에서는 각 Container가 자체 User Space Library를 가지고 있을 수 있어서 어떤 Library Binary에 Probe를 걸어야 하는지 찾아야 한다.

Application이 Shared Library를 사용하지 않고 Static Linking된 하나의 Binary로 만들어져 있다면 접근 방법도 달라질 수 있다.

즉 Kernel은 Machine마다 하나지만 **User Space Library는 여러 복사본이 존재할 수 있다는 차이**가 있다.

---

# 17. eBPF and Kubernetes Networking

Kubernetes에서 Application은 Pod 안에서 실행된다.

Pod는 하나 이상의 Container로 구성되며, 일반적으로 Pod마다 **자신의 Network Namespace와 IP Address**를 가진다.

Pod와 Host는 보통 Virtual Ethernet Interface를 통해 연결된다.

외부에서 Pod로 Packet이 들어간다고 생각하면 대략 다음과 같은 경로를 거친다.

```text
External Network
       ↓
Physical NIC
       ↓
Host Network Stack
       ↓
Host 쪽 veth
       ↓
Pod 쪽 veth
       ↓
Pod Network Namespace
       ↓
Pod Network Stack
       ↓
Application
```

Host와 Pod가 서로 다른 Machine Kernel을 사용하는 것은 아니다.

동일한 Linux Kernel 안에서 Network Namespace가 분리되어 있기 때문에, Packet은 같은 Kernel의 Networking Processing을 서로 다른 Namespace에서 여러 번 지나갈 수 있다.

Packet이 처리해야 하는 코드와 경로가 길어질수록 Latency와 CPU 사용량이 늘어날 가능성이 있다.

Cilium 같은 eBPF 기반 CNI는 Network Stack의 적절한 지점에서 Packet을 가로채서 **불필요한 Network Processing Path를 줄이는 방식**을 사용할 수 있다.

---

# 18. Avoiding iptables

기존 Kubernetes에서는 `kube-proxy`가 Service Load Balancing을 구현하기 위해 iptables를 널리 사용해 왔다.

예를 들어 Service 하나에 여러 Pod가 연결되어 있다면:

```text
Client
  ↓
Service IP
  ↓
kube-proxy / iptables
  ↓
Pod A / Pod B / Pod C
```

와 같은 처리가 필요하다.

일부 CNI도 Kubernetes NetworkPolicy를 구현할 때 iptables Rule을 이용한다.

하지만 Kubernetes 환경에서는 Pod와 IP Address가 계속 생성되고 삭제된다.

Pod가 변경될 때마다 관련 Rule도 변경해야 하기 때문에 Cluster 규모가 커질수록 관리해야 할 Rule이 많아질 수 있다.

책에서는 특히 두 가지 문제를 설명한다.

### 1. Rule Update 비용

Pod나 Service 상태가 바뀌면 iptables Rule을 다시 갱신해야 한다.

대규모 환경에서는 Rule 갱신 자체가 상당한 비용이 될 수 있다.

### 2. Rule Lookup 비용

iptables에서는 Rule을 순서대로 확인하기 때문에 Rule 수가 많아질수록 Lookup 비용도 증가할 수 있다.

책에서는 이를 `O(n)`으로 설명한다.

반면 Cilium에서는 eBPF Hash Map을 이용해서 다음과 같은 정보를 저장할 수 있다.

- Network Policy
- Connection Tracking
- Load Balancer Lookup Table

Hash Table Lookup과 Insert는 일반적으로 **O(1)에 가까운 비용**으로 처리할 수 있기 때문에 Rule 수가 증가하는 환경에서 더 잘 확장될 수 있다.

이것이 Cilium이 eBPF를 이용해 **kube-proxy Replacement**를 구현할 수 있는 중요한 기반 중 하나이다.

---

# 19. Coordinated Network Programs

Cilium처럼 복잡한 Networking 기능은 하나의 거대한 eBPF Program으로 구현되지 않는다.

대신 **여러 eBPF Program을 서로 다른 Hook Point에 연결하고 서로 협력하게 만든다.**

예를 들어 Cilium에서는 Packet의 위치와 목적에 따라 다음과 같은 지점에서 eBPF Program이 동작할 수 있다.

- Socket Layer
- XDP
- TC
- Pod Virtual Ethernet Interface
- Tunnel Interface
- Physical Network Interface
- Host Network Interface

가능한 경우 Packet을 최대한 이른 시점에 처리해서 불필요한 경로를 줄인다.

예를 들어 Application에서 나가는 Traffic은 Socket Layer 가까이에서 처리할 수 있고, 외부에서 Host로 들어오는 Packet은 XDP에서 빠르게 처리할 수 있다.

---

## Cilium의 여러 Networking Mode

Cilium은 환경에 따라 여러 Networking Mode를 사용할 수 있다.

책에서는 크게 다음과 같은 형태를 설명한다.

### Direct Routing 방식

Pod IP들이 Routing 가능한 하나의 Network 영역에서 동작하고, Node 간 Traffic을 직접 Routing하는 방식이다.

```text
Pod A
 ↓
Node A
 ↓
Routing
 ↓
Node B
 ↓
Pod B
```

### Tunneling 방식

다른 Node의 Pod로 가야 하는 Packet을 Node 간 Tunnel Packet 안에 Encapsulation해서 전송한다.

```text
Pod A의 Packet
    ↓
Encapsulation
    ↓
Node A → Node B
    ↓
Decapsulation
    ↓
Pod B
```

이처럼 Packet이 Local Pod로 가는지, Host로 가는지, 다른 Node로 가는지, Tunnel을 이용하는지에 따라 실행되어야 할 eBPF Program도 달라진다.

그래서 Cilium에는 여러 Attachment Point의 eBPF Program이 존재한다.

---

## eBPF Program 간 정보 공유

서로 다른 위치에 Attach된 Program들은 완전히 독립적으로만 동작하는 것이 아니다.

다음과 같은 방법으로 정보를 공유할 수 있다.

- eBPF Map
- Packet Metadata

앞에서 XDP의 `data_meta`를 살펴본 것도 이와 연결된다.

즉 한 Program에서 Packet에 대한 정보를 기록하고, 뒤쪽 Hook에서 실행되는 다른 Program이 그 정보를 이용할 수 있다.

---

# 20. Network Policy Enforcement

Network Policy Enforcement의 가장 기본적인 동작은 결국 **Packet을 허용할지 Drop할지 결정하는 것**이다.

```text
Packet
   ↓
Policy 확인
   ↓
허용됨?
 ├─ YES → PASS
 └─ NO  → DROP
```

전통적인 Firewall은 주로 IP Address와 Port를 기준으로 Rule을 만든다.

하지만 Kubernetes에서는 Pod IP가 계속 바뀔 수 있다.

오늘 어떤 Application Pod가 사용한 IP Address가 내일은 다른 Application Pod에 할당될 수도 있다.

따라서 IP Address만을 기준으로 Firewall Rule을 관리하면 Kubernetes처럼 변화가 많은 환경에서는 운영이 어렵다.

---

## Kubernetes NetworkPolicy

Kubernetes는 `NetworkPolicy` Resource를 제공한다.

여기에서는 IP를 직접 지정하는 방식보다 **Pod Label을 기반으로 통신 허용 대상을 정의**할 수 있다.

예를 들어 개념적으로:

```text
role=frontend
      ↓
role=backend 로만 통신 허용
```

같은 정책을 만들 수 있다.

여기서 중요한 점은 Kubernetes가 `NetworkPolicy` Object를 정의하지만 **실제 Packet을 Drop하거나 허용하는 구현까지 Kubernetes 자체가 수행하는 것은 아니라는 것**이다.

실제 Enforcement는 CNI Plugin이 담당한다.

따라서 사용하는 CNI가 NetworkPolicy를 지원해야 정책이 실제 Network Traffic에 반영된다.

---

## Cilium Network Policy

Cilium은 eBPF Program을 이용해서 현재 Policy Rule에 맞지 않는 Packet을 Drop한다.

또 Kubernetes 기본 NetworkPolicy보다 더 확장된 정책도 사용할 수 있다.

예를 들면:

- DNS Name 기준 허용/차단
- HTTP Method 기준 허용/차단
- URL 기준 Layer 7 Policy

등이다.

Cilium에서는 Pod Label을 기반으로 **Security Identity**를 만들고, Packet이 어디에서 왔는지를 단순 IP 대신 Identity 기준으로 판단할 수 있다.

Policy 정보는 eBPF Hash Map에 저장해서 빠르게 조회할 수 있다.

이 구조는 IP가 자주 바뀌는 Kubernetes 환경과 잘 맞는다.

---

# 21. Encrypted Connections

Application 간 Traffic을 보호하기 위해 HTTPS나 gRPC 연결 위에서 mTLS를 사용하는 방법이 있다.

mTLS에서는 통신하는 양쪽 Application이 서로의 Certificate를 이용해 Identity를 확인한 뒤 Traffic을 암호화한다.

하지만 모든 Application이 직접 Certificate 관리와 암호화 설정을 담당하게 하면 운영 부담이 커질 수 있다.

Kubernetes에서는 이런 기능을 Application 자체가 아니라 다음 Layer에 맡길 수도 있다.

- Service Mesh
- Network Layer

Chapter 8에서는 Network Layer에서 제공하는 **Transparent Encryption**을 설명한다.

---

## Transparent Encryption

Transparent Encryption은 Application이 Encryption을 직접 처리하지 않아도 Network Layer에서 자동으로 Traffic을 암호화하는 방식이다.

Application 입장에서는 평소와 똑같이 통신하지만 실제 Node 사이의 Traffic은 암호화될 수 있다.

```text
Pod A
  ↓
CNI / Kernel
  ↓
Encrypted Traffic
  ↓
Network
  ↓
CNI / Kernel
  ↓
Pod B
```

대표적인 In-Kernel Encryption Protocol은 다음과 같다.

- IPsec
- WireGuard

Cilium과 Calico 모두 이러한 방식의 Encryption을 지원한다.

Node 사이에 Secure Tunnel을 만들고 Pod Traffic을 해당 Tunnel로 전달할 수 있다.

Application을 수정하지 않아도 된다는 점에서 `Transparent`라는 표현을 사용한다.

---

## NetworkPolicy와 Encryption

Transparent Encryption은 NetworkPolicy와 함께 사용할 수도 있다.

즉 다음 두 기능은 서로 다른 역할을 한다.

```text
NetworkPolicy
→ 누가 누구와 통신할 수 있는가?

Encryption
→ 허용된 Traffic을 어떻게 안전하게 전달할 것인가?
```

Policy로 Endpoint 간 통신 가능 여부를 제어하면서, 실제 Network를 통과하는 Data는 IPsec이나 WireGuard를 이용해 암호화할 수 있다.

---

## Application Identity까지 확장

책에서는 Node Identity뿐 아니라 **각 Application Endpoint를 인증한 뒤 Network Layer에서 암호화하는 방식**도 소개한다.

TLS를 이용해 먼저 Endpoint Identity를 인증하고, 이후 실제 Traffic은 Kernel의 IPsec이나 WireGuard를 이용해 암호화할 수 있다.

이때 Certificate와 Identity 관리는 다음과 같은 별도 도구가 담당할 수 있다.

- cert-manager
- SPIFFE/SPIRE

Application 자체가 모든 TLS 처리를 담당하지 않아도 되고, Network가 Encryption을 담당할 수 있다는 점이 특징이다.

또 Cilium은 SPIFFE ID를 기준으로 Network Policy Endpoint를 정의하는 방식도 지원한다.

---

# 정리

Chapter 8에서는 eBPF가 실제 Networking에 어떻게 사용되는지를 살펴봤다.

먼저 XDP에서는 Network Interface로 Packet이 들어오는 매우 이른 시점에 eBPF Program이 실행되고, Return Code를 통해 `PASS`, `DROP`, `TX`, `REDIRECT` 등의 동작을 결정할 수 있었다.

Packet을 처리하려면 `xdp_md`의 `data`, `data_end`를 이용해 Ethernet Header와 IP Header를 직접 Parsing해야 하며, 이 과정에서도 Verifier가 Memory Access 범위를 검사하기 때문에 Header를 읽기 전에 반드시 `data_end`를 확인해야 한다.

XDP는 Packet을 읽는 것뿐만 아니라 Header를 수정할 수도 있기 때문에 간단한 Load Balancing과 Forwarding 기능을 구현할 수 있다. 일부 NIC에서는 XDP Offload를 이용해 Host CPU까지 Packet이 도달하기 전에 NIC에서 직접 처리하는 것도 가능하다.

TC에서는 Packet이 이미 Network Stack에 들어와 `sk_buff`가 생성된 이후의 Traffic을 처리한다. XDP와 달리 Ingress뿐 아니라 Egress에도 Program을 연결할 수 있고, Packet을 Drop하거나 Redirect하는 것뿐 아니라 간단한 ICMP Reply처럼 일부 Network 기능을 직접 구현할 수도 있다.

또한 eBPF는 Kernel Network Hook에만 사용하는 것이 아니라 OpenSSL의 `SSL_read()` 같은 User Space 함수에 uprobe와 uretprobe를 연결해서 복호화된 Data를 관찰하는 데도 활용할 수 있다.

Kubernetes에서는 Pod마다 Network Namespace가 있고 Packet이 Host와 Pod의 Network Stack을 지나야 하기 때문에 Network Path가 길어질 수 있다. Cilium은 여러 eBPF Program을 Socket, XDP, TC 등 다양한 Hook Point에 배치해서 Packet을 가능한 한 빠르게 처리하고 불필요한 Network Processing을 줄인다.

기존 Kubernetes에서 많이 사용해 온 iptables는 Pod와 Service가 계속 바뀌는 환경에서 Rule Update와 Lookup 비용이 커질 수 있다. Cilium은 Network Policy, Connection Tracking, Load Balancing 정보를 eBPF Hash Map에 저장해서 더 효율적으로 조회하고, 이를 기반으로 kube-proxy Replacement 기능도 제공한다.

Network Policy에서도 eBPF의 Packet Drop 기능이 그대로 활용된다. Cilium은 변화가 많은 Pod IP 자체보다 Kubernetes Label을 기반으로 만든 Security Identity를 활용해서 Policy를 적용하고, DNS나 HTTP 수준까지 확장된 Policy도 지원한다.

마지막으로 IPsec이나 WireGuard를 이용한 Transparent Encryption을 통해 Application 수정 없이 Network Layer에서 Pod Traffic을 암호화할 수 있다는 점도 확인했다.

이번 장을 정리하면서 **eBPF Networking은 단순히 Packet을 빠르게 처리하는 기술 하나가 아니라, Linux Network Stack의 여러 지점에 Program을 연결하고 eBPF Map과 Metadata를 함께 사용해서 Load Balancing, Firewall, Network Policy, Kubernetes Networking, Encryption 같은 기능을 구성할 수 있는 기반**이라는 점을 이해할 수 있었다.
