---
layout: post
title: "[Learning eBPF] Chapter 1: What Is eBPF, and Why Is It Important?"
date: 2026-07-11 19:40:00 +0900
categories: [eBPF, Linux, Kernel]
tags: [eBPF, BPF, Cilium, Linux Kernel]
published: true
---

# Learning eBPF Chapter 1

## What Is eBPF, and Why Is It Important?

eBPF(extended Berkeley Packet Filter)는 Linux Kernel 내부에서 사용자 정의 프로그램을 안전하게 실행하기 위한 Runtime이다. 프로그램은 특정 Kernel Hook에 연결(Attach)되어 이벤트가 발생할 때 실행되며, Kernel을 재컴파일하거나 Kernel Module을 개발하지 않고도 Networking, Observability, Security 기능을 확장할 수 있다.

eBPF 프로그램은 독립적으로 실행되는 프로세스가 아니라 Event-driven 방식으로 동작한다. Packet 수신, System Call 실행, Tracepoint 호출, Cgroup Event, LSM Hook 등 특정 Kernel Event가 발생하면 연결된 eBPF Program이 실행되고 작업을 수행한 후 종료된다.

```text
Kernel Event
    │
    ├── Network Packet
    ├── System Call
    ├── Tracepoint
    ├── Cgroup
    ├── LSM
    └── Kernel Function
            │
            ▼
       eBPF Program
            │
            ├── Context 조회
            ├── BPF Map 조회
            ├── Helper Function 호출
            ├── Packet 수정 또는 전달
            └── Event 생성
```

기존 Linux에서 새로운 기능을 추가하려면 Kernel Source 수정 또는 Kernel Module 개발이 필요했다.

```text
Kernel Source 수정
        │
        ▼
Kernel Build
        │
        ▼
System Reboot
        │
        ▼
새로운 기능 적용
```

eBPF는 기존 Kernel에 이미 존재하는 Hook을 활용한다. 따라서 프로그램을 동적으로 로드하여 즉시 기능을 추가할 수 있다.

```text
eBPF Program
        │
        ▼
Verifier
        │
        ▼
Kernel Load
        │
        ▼
Hook Attach
        │
        ▼
Runtime 실행
```

이 구조는 Cloud Native 환경에서 특히 중요하다. Kubernetes Node의 여러 Pod와 Container는 동일한 Linux Kernel을 공유하므로, Kernel Hook에 eBPF Program을 연결하면 Node 전체에서 발생하는 Packet, Process, System Call, Security Event를 관찰하거나 제어할 수 있다.

대표적인 활용 사례는 다음과 같다.

| 프로젝트 | 활용 목적 |
|---|---|
| Cilium | Kubernetes Networking, Network Policy, Load Balancing |
| Hubble | Network Flow Observability |
| Katran | XDP 기반 L4 Load Balancer |
| Tetragon | Runtime Security 및 Process Observability |
| bpftool | Program 및 Map 관리 |
| bpftrace | Kernel Tracing |

## eBPF's Roots: The Berkeley Packet Filter

eBPF는 Classic Berkeley Packet Filter(cBPF)를 기반으로 발전하였다.

Classic BPF는 Linux Socket Layer에서 Packet Filtering을 수행하기 위한 Virtual Machine으로, Packet Capture 과정에서 불필요한 Packet을 User Space까지 전달하지 않기 위해 설계되었다.

Packet은 Network Driver를 통해 Kernel로 전달되며 Socket Queue에 적재되기 전에 BPF Program이 실행된다.

```text
NIC
 │
 ▼
Network Driver
 │
 ▼
Socket Filter (cBPF)
 │
 ├── ACCEPT
 │      │
 │      ▼
 │  Socket Queue
 │      │
 │      ▼
 │  User Space
 │
 └── DROP
```

cBPF 프로그램은 Packet Header의 특정 Offset을 읽고 조건을 비교하여 Packet의 전달 여부를 결정한다.

예를 들어 IPv4 Packet 여부를 검사하는 경우 Ethernet Header의 EtherType 필드를 확인한다.

```text
Packet
   │
   ▼
Load EtherType
   │
   ▼
Compare
   │
   ▼
Accept / Drop
```

Packet Header 전체를 User Space에서 분석하는 대신 Kernel에서 필요한 필드만 검사하므로, 불필요한 Packet Copy와 User Space 처리 비용을 줄일 수 있다.

`tcpdump`와 `libpcap`은 사용자가 작성한 Filter Expression을 BPF Instruction으로 변환하여 Socket에 연결한다.

```bash
tcpdump 'tcp dst port 443'
```

```text
Filter Expression
        │
        ▼
libpcap Compiler
        │
        ▼
BPF Instruction
        │
        ▼
Kernel Socket Filter
```

Classic BPF는 Packet Filtering에는 효과적이었지만, Packet 이외의 Kernel Event를 처리하거나 실행 사이의 상태를 유지하기에는 제한이 있었다.

### Classic BPF의 한계

Classic BPF는 Packet Filtering에 특화되어 설계되었기 때문에 활용 범위가 제한적이었다.

대표적인 제약은 다음과 같다.

- Packet 데이터 중심의 제한된 실행 Context
- Program 실행 사이의 상태 공유 제한
- Process 및 File System 정보 접근 불가
- 다양한 Kernel Event에 연결할 수 없음
- 현대 eBPF와 같은 범용 Map 및 Helper 구조 부재

즉, Packet 하나를 기준으로 전달 여부를 판단하는 데에는 적합했지만 Connection Tracking, Service Load Balancing, Runtime Tracing, Security Policy와 같은 기능을 구현하기에는 부족했다.

### seccomp-bpf

BPF는 이후 Packet Filtering 외에도 System Call Filtering에 활용되었다.

`seccomp-bpf`는 Process가 System Call을 호출할 때 BPF Filter를 실행하여 해당 호출의 허용 여부를 결정한다.

```text
User Process
      │
      ▼
 System Call
      │
      ▼
 seccomp-bpf
      │
      ├── Allow
      ├── Deny
      └── Kill
```

이는 BPF 실행 모델이 Network Packet뿐 아니라 다른 Kernel Event에도 적용될 수 있음을 보여준 사례다. 다만 `seccomp-bpf`는 System Call Filtering에 특화되어 있으며, 현대 eBPF의 다양한 Program Type과 Kernel Hook을 제공하는 범용 Runtime과는 구분된다.

## From BPF to eBPF

Classic BPF는 Packet Filtering만을 목적으로 설계된 Virtual Machine이었다. 따라서 Packet Header를 기반으로 조건을 검사하고 Packet의 전달 여부를 결정하는 작업이 중심이었다.

eBPF는 이러한 실행 모델을 확장하여 Linux Kernel의 여러 Subsystem에서 사용할 수 있는 범용 Runtime으로 발전하였다.

주요 변화는 다음과 같다.

| Classic BPF | eBPF |
|---|---|
| Packet Filtering 중심 | Networking, Tracing, Security 지원 |
| 제한된 실행 구조 | 64-bit Register 기반 실행 구조 |
| Packet Context 중심 | Program Type별 다양한 Context |
| 상태 공유 제한 | BPF Map 지원 |
| 제한된 Kernel 상호작용 | Helper Function 지원 |
| Socket Filter 중심 | XDP, TC, kprobe, Tracepoint, Cgroup, LSM 등 지원 |

가장 큰 변화는 Packet이라는 입력에 제한되지 않고 다양한 Kernel Context를 처리할 수 있게 되었다는 점이다.

대표적인 Hook은 다음과 같다.

- XDP
- TC
- Socket
- Tracepoint
- kprobe
- uprobe
- Cgroup
- LSM

Program은 Attach된 Hook에 따라 서로 다른 Context를 전달받는다.

예를 들어 XDP Program은 Network Driver 단계의 Packet 정보를 전달받고, Tracepoint Program은 Kernel 내부에서 정의된 Event 정보를 전달받는다. LSM Program은 Security 관련 Kernel Operation의 Context를 전달받는다.

```text
                     Linux Kernel

                          │

        ┌─────────────────┼─────────────────┐

        │                 │                 │

    Networking         Tracing          Security

        │                 │                 │

        ▼                 ▼                 ▼

   XDP / TC          kprobe /          LSM / Cgroup
                     Tracepoint
        │                 │                 │

        └─────────────────┼─────────────────┘

                          ▼

                    eBPF Program
```

### BPF Map

Classic BPF는 Program 실행 사이에 상태를 공유하기 어려웠다. eBPF는 이를 해결하기 위해 BPF Map을 제공한다.

BPF Map은 Kernel Memory에 생성되는 Data Structure이며 User Space Application과 eBPF Program이 함께 접근할 수 있다.

```text
          User Space Application
                    │
                    │ Update / Lookup
                    ▼
             +-------------+
             |   BPF Map   |
             +-------------+
                ▲       ▲
                │       │
           Program A  Program B
```

eBPF Program은 Event가 발생할 때마다 실행되고 종료되므로, Program 내부의 Stack과 Register만으로 장기적인 상태를 유지할 수 없다. BPF Map은 여러 실행 사이의 상태를 저장하고, 여러 Program이 정보를 공유하며, User Space에서 Kernel Datapath의 설정을 변경할 수 있도록 한다.

대표적인 활용 사례는 다음과 같다.

- Connection Tracking
- Service Backend 정보
- Network Policy
- Packet 및 Event 통계
- User Space Configuration
- Kernel Event 전달

Cilium은 Kubernetes Networking에 필요한 Runtime 정보를 BPF Map으로 관리한다.

Service가 생성되거나 Backend Pod가 변경되더라도 eBPF Program 자체를 다시 작성하거나 Load할 필요는 없다. User Space에서 실행되는 `cilium-agent`가 관련 Map을 갱신하면 다음 Packet부터 변경된 정보가 반영된다.

```text
Kubernetes API
      │
      ▼
cilium-agent
      │
      │ BPF Map Update
      ▼
+------------------+
| Service / Policy |
|      BPF Map     |
+------------------+
      ▲
      │ Lookup
      ▼
eBPF Datapath
```

이 구조는 Kubernetes 상태를 관리하는 Control Plane과 실제 Packet을 처리하는 Data Plane을 분리한다.

### bpf() System Call

eBPF에서는 User Space가 Kernel의 BPF Subsystem과 상호작용하기 위해 `bpf()` System Call을 사용한다.

`bpf()` System Call을 통해 다음과 같은 작업을 수행할 수 있다.

- BPF Map 생성
- eBPF Program Load
- Map Data 조회 및 갱신
- BPF Object 관리
- 일부 Program Type의 Attach 및 Detach

```text
User Space Loader
      │
      ├── Map 생성
      ├── Program Load
      ├── Map Update
      └── Object 관리
              │
              ▼
            bpf()
              │
              ▼
        Linux Kernel
```

실제 eBPF Application에서는 `bpf()` System Call을 직접 호출하기보다 libbpf, BCC, cilium/ebpf, libbpf-rs와 같은 Library를 사용한다.

### Helper Function

eBPF Program은 임의의 Kernel Function을 직접 호출할 수 없다.

대신 Kernel이 제공하는 Helper Function을 통해 필요한 기능을 수행한다.

대표적인 기능은 다음과 같다.

- BPF Map 조회
- BPF Map 갱신
- 현재 Process 정보 조회
- 현재 시간 조회
- Packet Redirect
- Event를 User Space로 전달

```text
eBPF Program
      │
      ▼
Helper Function
      │
      ▼
제한된 Kernel 기능 사용
```

사용할 수 있는 Helper Function은 Program Type에 따라 달라진다.

Networking Program에서는 Packet Redirect나 Packet 관련 Helper를 사용할 수 있고, Tracing Program에서는 현재 Process 정보나 Event 전달과 관련된 Helper를 사용할 수 있다.

이 구조는 eBPF Program이 Kernel 내부 기능을 무제한으로 호출하지 못하도록 제한한다.

### Verifier

eBPF Program은 Kernel에 Load되기 전에 반드시 Verifier를 통과해야 한다.

Verifier는 Program을 정적으로 분석하여 Kernel 안정성을 해칠 수 있는 동작이 있는지 검사한다.

대표적으로 다음 항목을 확인한다.

- 잘못된 Memory Access
- Stack 범위 초과
- 종료되지 않는 실행 경로
- 초기화되지 않은 값 사용
- 허용되지 않은 Helper 호출
- Program Type에 맞지 않는 Context 접근

```text
eBPF Program
      │
      ▼
Verifier
      │
      ├── Memory Access 검사
      ├── Control Flow 검사
      ├── Stack 검사
      └── Helper 호출 검사
              │
        ┌─────┴─────┐
        │           │
      Pass         Fail
        │           │
        ▼           ▼
 Kernel Load     Load 거부
```

Verifier를 통과한 Program만 Kernel에서 실행될 수 있다.

Kernel Module은 Native Kernel Code로 실행되며 잘못된 Memory Access나 Deadlock이 Kernel Panic으로 이어질 수 있다. eBPF는 Verifier와 제한된 실행 환경을 통해 이러한 위험을 줄인다.

## The Evolution of eBPF to Production Systems

eBPF가 Production 환경에서 사용될 수 있었던 이유는 하나의 기능만 추가되었기 때문이 아니다. 다양한 Kernel Hook, Map, Verifier, JIT Compiler, BTF와 같은 기능이 단계적으로 추가되면서 Packet Filter를 넘어 범용 Kernel Programmability Platform으로 발전하였다.

```text
1997
BPF가 Linux Kernel에 도입

2012
seccomp-bpf를 통한 System Call Filtering

2014
64-bit eBPF, BPF Map, bpf() System Call, Verifier 도입

2015
eBPF Program의 kprobe Attach 지원

2016
Cilium 공개
Container Networking에 eBPF Datapath 적용

2017
Katran 공개
XDP 기반 L4 Load Balancing

2018
eBPF가 Linux Kernel의 독립 Subsystem으로 정착
BTF 도입

2020
BPF LSM 도입
```

### Kernel Tracing으로의 확장

kprobe는 Kernel Function의 실행 지점에 동적으로 Probe를 연결하는 기능이다.

eBPF Program을 kprobe에 Attach하면 특정 Kernel Function이 호출될 때마다 Program을 실행할 수 있다.

```text
Kernel Function
      │
      ▼
    kprobe
      │
      ▼
eBPF Program
      │
      ▼
Event 수집
```

이를 통해 Kernel Source를 수정하지 않고도 Function 호출 횟수, 실행 시간, Process 정보 등을 관찰할 수 있다.

Tracepoint는 Kernel Source에 미리 정의된 Event Hook이다. System Call, Scheduler, Network, Block I/O 등 주요 Kernel Event에 eBPF Program을 연결할 수 있다.

```text
Kernel Subsystem
      │
      ▼
  Tracepoint
      │
      ▼
eBPF Program
```

`bpftrace`와 BCC 기반 Tool은 이러한 Hook을 이용해 Kernel 동작을 추적한다.

### Networking으로의 확장

eBPF는 Socket Filter를 넘어 XDP와 TC 같은 Networking Hook으로 확장되었다.

XDP는 Network Driver가 수신 Packet을 Kernel Network Stack으로 전달하기 전에 실행된다.

```text
NIC
 │
 ▼
Network Driver
 │
 ▼
XDP Program
 │
 ├── Drop
 ├── Pass
 ├── Transmit
 └── Redirect
```

Packet 처리 경로의 앞단에서 실행되므로 DDoS Filtering, L4 Load Balancing, Packet Redirect와 같은 기능에 적합하다.

Katran은 XDP 기반 L4 Load Balancer의 대표적인 사례다. Packet이 일반적인 Kernel Network Stack을 모두 통과하기 전에 Backend를 선택하고 전달한다.

TC eBPF Program은 Packet이 `sk_buff`로 변환된 이후 Traffic Control Hook에서 실행된다.

```text
Network Driver
      │
      ▼
    XDP
      │
      ▼
sk_buff 생성
      │
      ▼
 TC Ingress
      │
      ▼
Kernel Network Stack
      │
      ▼
 TC Egress
```

Cilium은 주로 TC Hook에 연결된 eBPF Program을 통해 Pod Traffic의 Routing, Network Policy, Service Load Balancing, NAT, Connection Tracking을 처리한다. XDP는 특정 Load Balancing 가속이나 Native Device 처리 기능에서 선택적으로 사용된다.

### Observability와 Security로의 확장

eBPF는 Kernel Function과 Tracepoint에 연결되어 Process, System Call, Network, File 관련 Event를 관찰할 수 있다.

Observability Tool은 Kernel에서 필요한 Event만 선별한 뒤 User Space로 전달한다.

```text
Kernel Event
      │
      ▼
eBPF Program
      │
      ├── 관심 없는 Event
      │       └── 종료
      │
      └── 필요한 Event
              └── User Space 전달
```

Security 영역에서는 BPF LSM을 통해 Linux Security Module Hook에 eBPF Program을 연결할 수 있게 되었다.

```text
Process Operation
      │
      ▼
  LSM Hook
      │
      ▼
BPF LSM Program
      │
      ├── Allow
      └── Deny
```

Tetragon은 eBPF를 이용해 Process 실행, File 접근, Network Connection과 같은 Runtime Event를 관찰하고 Policy에 따라 특정 동작을 제한할 수 있다.

### BTF의 도입

BTF(BPF Type Format)는 Kernel과 eBPF Program에서 사용하는 Type 정보를 표현하는 Metadata Format이다.

BTF는 Kernel Struct, Function, Variable 등의 Type 정보를 제공하며, eBPF Tool이 Kernel 내부 Data Structure를 보다 안정적으로 해석할 수 있도록 한다.

```text
Kernel Type Information
      │
      ▼
     BTF
      │
      ├── Struct
      ├── Enum
      ├── Function
      └── Variable
```

BTF는 이후 Kernel Version 차이를 처리하는 CO-RE와 효율적인 Function Tracing을 위한 기반이 된다. 구체적인 BTF와 CO-RE의 구조는 이후 Chapter에서 다룬다.

## Naming Is Hard

현재 `BPF`와 `eBPF`라는 용어는 문맥에 따라 혼용된다.

초기의 Packet Filter는 Classic BPF 또는 cBPF라고 부르고, 확장된 64-bit Runtime은 eBPF라고 부른다.

하지만 Linux Kernel Source와 API에서는 대부분 `eBPF`가 아니라 `BPF`라는 이름을 사용한다.

대표적인 예시는 다음과 같다.

```text
bpf() System Call

BPF Program

BPF Map

BPF_PROG_TYPE_XDP

BPF_MAP_TYPE_HASH

BPF Helper Function

bpftool
```

따라서 문맥에 따라 `BPF`는 다음과 같은 의미를 가질 수 있다.

| 용어 | 의미 |
|---|---|
| Classic BPF | 초기 Packet Filtering VM |
| eBPF | 확장된 Kernel Runtime |
| BPF Program | Kernel에 Load되는 eBPF Program |
| BPF Map | eBPF Program이 사용하는 Kernel Data Structure |
| BPF Subsystem | Linux Kernel 내부의 전체 BPF 구현 |

Kernel 내부 구현과 Tool에서는 `BPF`라는 표현이 일반적으로 사용되고, Cloud Native 및 User-facing 문서에서는 Classic BPF와 구분하기 위해 `eBPF`라는 표현을 주로 사용한다.

## The Linux Kernel

Linux Kernel은 Application과 Hardware 사이에서 System Resource를 관리한다.

Application은 User Space에서 실행되며 Hardware나 Kernel Memory에 직접 접근할 수 없다. File, Network, Process, Memory와 관련된 작업은 System Call을 통해 Kernel에 요청한다.

```text
User Space Application
        │
        │ System Call
        ▼
   Linux Kernel
        │
        ├── File System
        ├── Network
        ├── Memory
        ├── Scheduler
        └── Device Driver
                │
                ▼
             Hardware
```

Application은 실행 과정에서 반복적으로 Kernel 기능을 사용한다.

예를 들어 다음 작업은 모두 Kernel의 처리가 필요하다.

- File 열기 및 읽기
- Network Packet 송수신
- Process 생성
- Memory 할당
- Thread Scheduling
- Device 접근

eBPF는 이러한 Kernel 동작 경로에 연결될 수 있다.

```text
Application
      │
      ▼
Kernel Operation
      │
      ▼
Kernel Hook
      │
      ▼
eBPF Program
```

따라서 Application Source Code를 수정하지 않아도 Kernel과 Application 사이에서 발생하는 Event를 관찰할 수 있다.

예를 들어 File Open과 관련된 Kernel Event에 eBPF Program을 연결하면 어떤 Process가 어떤 File에 접근했는지 확인할 수 있다. Network Hook에 Program을 연결하면 Application이 송수신하는 Packet을 관찰하거나 제어할 수 있다.

다만 eBPF Program은 Kernel 전체에 자유롭게 접근하는 일반적인 Kernel Code가 아니다.

Program Type에 따라 다음 항목이 제한된다.

- 연결할 수 있는 Hook
- 전달받는 Context
- 사용할 수 있는 Helper Function
- 반환값의 의미
- 접근할 수 있는 Memory 영역

이 제한은 eBPF Program이 Kernel 내부에서 안전하게 실행될 수 있도록 하기 위한 구조다.

## Adding New Functionality to the Kernel

Linux Kernel에 새로운 기능을 직접 추가하려면 Kernel Source를 수정해야 한다.

일반적인 과정은 다음과 같다.

```text
Kernel Source 수정
      │
      ▼
Patch 작성 및 Review
      │
      ▼
Upstream Merge
      │
      ▼
Kernel Release
      │
      ▼
Linux Distribution 반영
      │
      ▼
Host Upgrade 및 Reboot
```

Linux Kernel은 다양한 Hardware, Architecture, Distribution에서 사용되므로 모든 기능이 Upstream에 포함되는 것은 아니다. 기능이 Kernel에 Merge되더라도 실제 Production 환경의 Linux Distribution에 포함되기까지 시간이 필요하다.

Enterprise Distribution은 최신 Kernel로 즉시 변경하기보다 안정적인 Kernel Version을 장기간 유지하면서 필요한 Patch를 Backport한다.

따라서 새로운 Kernel 기능이 개발되어도 사용자가 실제 시스템에서 사용하기까지 긴 시간이 걸릴 수 있다.

eBPF는 이미 Kernel에 구현된 Hook과 Runtime을 사용한다.

```text
eBPF Program 작성
      │
      ▼
Compile
      │
      ▼
Running Kernel에 Load
      │
      ▼
Hook에 Attach
      │
      ▼
즉시 기능 적용
```

Kernel Image를 교체하거나 System을 재부팅하지 않고도 Networking, Observability, Security 기능을 추가할 수 있다.

다만 eBPF가 모든 Kernel 변경을 대체하는 것은 아니다. 필요한 Hook, Helper Function, Map Type이 Kernel에 존재하지 않으면 해당 기능을 지원하기 위한 Kernel Patch가 필요하다.

즉, eBPF는 Kernel을 자유롭게 수정하는 기술이 아니라 Kernel이 제공하는 안전한 확장 지점 안에서 기능을 추가하는 Runtime이다.

## Kernel Modules

Kernel Module은 실행 중인 Linux Kernel에 Native Kernel Code를 Load하여 기능을 추가하는 방식이다.

대표적으로 다음 기능이 Kernel Module로 구현된다.

- Device Driver
- File System
- Network Protocol
- Hardware 지원
- Kernel Subsystem 확장

```text
Kernel Module
      │
      ├── Kernel Symbol 사용
      ├── Kernel Memory 접근
      ├── Hardware 제어
      └── Kernel 기능 확장
```

Kernel Module은 Kernel과 동일한 Privilege Level에서 실행된다.

잘못된 Pointer Access, Memory Corruption, Deadlock, Infinite Loop가 발생하면 해당 Module만 종료되는 것이 아니라 System 전체에 영향을 줄 수 있다.

```text
Kernel Module Bug
      │
      ├── Kernel Panic
      ├── System Hang
      ├── Memory Corruption
      └── Security Vulnerability
```

eBPF는 Kernel Module보다 제한된 실행 환경을 사용한다.

| 항목 | Kernel Module | eBPF |
|---|---|---|
| 실행 형태 | Native Kernel Code | 검증된 BPF Program |
| Memory 접근 | Kernel Memory 접근 가능 | Verifier가 허용한 범위만 |
| Kernel 기능 호출 | Kernel Symbol 사용 | Helper Function 사용 |
| Load 전 검증 | eBPF Verifier 없음 | Verifier 필수 |
| Device Driver 구현 | 가능 | 일반적으로 불가능 |
| 실행 단위 | Module | Program과 Hook |
| 오류 영향 | System 전체 영향 가능 | 제한된 실행 모델 |

eBPF는 Kernel Module을 완전히 대체하지 않는다.

Device Driver, 새로운 File System, Kernel Scheduler 자체 변경처럼 Kernel 내부 기능을 직접 구현해야 하는 경우에는 Kernel Module 또는 Kernel Patch가 필요하다.

반면 다음처럼 기존 Kernel Event나 Packet Path에 Logic을 추가하는 작업에는 eBPF가 적합하다.

- Kernel Tracing
- Packet Filtering
- Network Policy
- Load Balancing
- Runtime Security
- Flow Observability

## Dynamic Loading of eBPF Programs

eBPF Program은 실행 중인 Linux Kernel에 동적으로 Load하고 제거할 수 있다.

일반적인 실행 흐름은 다음과 같다.

```text
eBPF Source
      │
      ▼
Compiler
      │
      ▼
BPF Object
      │
      ▼
User Space Loader
      │
      ├── Map 생성
      ├── Program Load
      └── Hook Attach
              │
              ▼
         Linux Kernel
```

User Space Loader는 `bpf()` System Call을 이용해 Program과 Map을 Kernel에 생성한다. 실제 Application에서는 libbpf, BCC와 같은 Library가 이 과정을 처리한다.

Program이 Kernel에 Load되었다고 바로 실행되는 것은 아니다. 특정 Event 또는 Hook에 Attach되어야 한다.

```text
Program Load
      │
      ▼
Hook Attach
      │
      ▼
Event 발생
      │
      ▼
eBPF Program 실행
```

대표적인 Attach 대상은 다음과 같다.

- Network Interface의 XDP
- TC Ingress 및 Egress
- Kernel Function의 kprobe
- Kernel Tracepoint
- Cgroup
- LSM Hook
- Socket Event

Program이 Attach된 이후에는 기존 Application이나 Process를 재시작하지 않아도 새로운 Event부터 즉시 적용된다.

예를 들어 Process 실행 Event에 eBPF Program을 Attach하면 이미 실행 중인 Process가 이후 새로운 Program을 실행할 때 해당 Event를 관찰할 수 있다.

```text
기존 Process
      │
      ▼
새로운 Program 실행
      │
      ▼
execve Event
      │
      ▼
eBPF Program 실행
```

Networking Program도 Interface나 Cgroup Hook에 Attach된 이후 들어오는 Packet부터 즉시 처리한다.

Cilium에서는 Node마다 실행되는 `cilium-agent`가 eBPF Program을 Load하고 Network Interface에 Attach하며, Kubernetes 상태 변경을 BPF Map에 반영한다.

```text
Kubernetes State
      │
      ▼
cilium-agent
      │
      ├── Program Load
      ├── Hook Attach
      └── BPF Map Update
              │
              ▼
       Kernel Datapath
```

## High Performance of eBPF Programs

eBPF가 높은 성능을 제공하는 이유는 Kernel 내부에서 실행된다는 점뿐 아니라 JIT Compilation, In-kernel Filtering, 적절한 Hook 위치를 함께 활용하기 때문이다.

### JIT Compilation

eBPF Program은 Bytecode로 Kernel에 Load된다.

Verifier를 통과한 Program은 JIT Compiler를 통해 현재 CPU Architecture의 Native Machine Code로 변환될 수 있다.

```text
eBPF Bytecode
      │
      ▼
BPF JIT Compiler
      │
      ▼
Native Machine Code
```

JIT Compilation을 사용하면 Program이 실행될 때마다 Bytecode를 해석하는 비용을 줄일 수 있다.

### Kernel과 User Space 전환 감소

모든 Event를 User Space로 전달한 뒤 Filtering하면 Event Copy, Context Switch, User Space Wake-up 비용이 발생한다.

```text
모든 Kernel Event
      │
      ▼
User Space 전달
      │
      ▼
User Space Filtering
```

eBPF는 Kernel 내부에서 필요한 Event만 선별할 수 있다.

```text
Kernel Event
      │
      ▼
eBPF Filter
      │
      ├── 필요 없음
      │       └── 종료
      │
      └── 필요함
              └── User Space 전달
```

예를 들어 특정 Process, Cgroup, Port, Protocol에 해당하는 Event만 수집할 수 있다.

이는 Observability와 Security Tool이 처리해야 하는 Event 양을 줄인다.

### Networking Path 단축

XDP는 Packet이 일반적인 Kernel Network Stack을 통과하기 전에 실행된다.

```text
Network Driver
      │
      ▼
XDP Program
      │
      ├── Drop
      ├── Pass
      └── Redirect
```

불필요한 Packet을 XDP 단계에서 Drop하면 이후의 Packet Buffer 생성, Routing, Protocol Processing 비용을 줄일 수 있다.

TC와 Socket Hook은 XDP보다 뒤에서 실행되지만 더 많은 Network Context를 사용할 수 있다. 따라서 eBPF에서는 무조건 가장 앞단의 Hook을 사용하는 것이 아니라 기능에 맞는 Hook을 선택한다.

### Kernel 내부 상태 조회

eBPF Program은 BPF Map을 통해 Packet 처리에 필요한 상태를 Kernel 내부에서 조회할 수 있다.

```text
Packet
      │
      ▼
BPF Map Lookup
      │
      ├── Service 정보
      ├── Policy 정보
      └── Connection State
              │
              ▼
       Forward 또는 Drop
```

Cilium은 Service, Policy, Endpoint, Connection Tracking 정보를 BPF Map에 저장하고 Packet 처리 시 조회한다.

다만 eBPF를 사용한다고 모든 기능이 자동으로 빨라지는 것은 아니다.

성능은 다음 요소에 따라 달라진다.

- Program이 Attach된 Hook 위치
- Program의 실행 복잡도
- BPF Map 조회 횟수
- User Space로 전달하는 Event 양
- Packet Parsing 범위
- CPU Cache와 동시성

특히 Packet마다 실행되는 eBPF Program은 매우 높은 빈도로 호출되므로 짧은 시간 안에 실행을 완료해야 한다.

## eBPF in Cloud Native Environments

Kubernetes의 Container는 각자 독립적인 Kernel을 사용하는 것이 아니라 Host의 Linux Kernel을 공유한다.

```text
Kubernetes Node
      │
      ├── Pod A
      │     ├── Container A1
      │     └── Container A2
      │
      ├── Pod B
      │     └── Container B1
      │
      └── Host Process
              │
              ▼
        Shared Linux Kernel
```

Node Kernel의 Hook에 eBPF Program을 Attach하면 해당 Node의 Container와 Host Process에서 발생하는 Event를 함께 관찰할 수 있다.

```text
Pod A Event ─┐
Pod B Event ─┼── Kernel Hook ── eBPF Program
Host Event ──┘
```

이 구조는 Application Source Code나 Container Image를 수정하지 않아도 Networking, Observability, Security 기능을 적용할 수 있게 한다.

### Application 변경 없이 관찰

Library 기반 Monitoring은 Application Source Code에 Agent나 SDK를 추가해야 한다.

```text
Application
      │
      ├── Monitoring Library
      └── Tracing SDK
```

Sidecar 방식은 Pod Spec에 별도의 Container를 추가한다.

```text
Pod
      │
      ├── Application Container
      └── Sidecar Container
```

eBPF는 Node Kernel의 Hook에서 Event를 관찰하므로 Application Binary와 Pod Spec을 변경하지 않아도 된다.

```text
Application / Container
          │
          ▼
      Linux Kernel
          │
          ▼
     eBPF Program
```

Program이 Kernel에 Attach된 이후에는 이미 실행 중인 Process와 새로 생성되는 Process에서 발생하는 Event를 모두 관찰할 수 있다.

### Sidecar 방식과의 차이

Sidecar 기반 Service Mesh에서는 Application Traffic이 User Space Proxy를 거쳐야 한다.

```text
Application Process
      │
      ▼
Kernel Network Stack
      │
      ▼
Sidecar Proxy
      │
      ▼
Kernel Network Stack
      │
      ▼
Network
```

이 방식은 Application별 Proxy를 통해 L7 Routing, TLS, Observability를 제공하지만 다음과 같은 운영 비용이 발생할 수 있다.

- Pod마다 추가 CPU와 Memory 사용
- Sidecar Injection 필요
- Pod Restart 필요
- Application과 Proxy의 Startup 순서 관리
- Traffic이 User Space Proxy를 추가로 통과

eBPF 기반 Networking은 L3/L4 Packet 처리를 Kernel Datapath에서 수행할 수 있다.

```text
Application Process
      │
      ▼
Kernel Network Stack
      │
      ▼
eBPF Datapath
      │
      ▼
Network
```

다만 HTTP Header, URL Path, gRPC Method처럼 L7 Parsing이 필요한 기능은 User Space Proxy가 필요하다.

Cilium은 L3/L4 기능을 eBPF Datapath에서 처리하고, L7 검사가 필요한 Traffic만 Node-local Envoy로 전달한다.

```text
Traffic
      │
      ▼
Cilium eBPF Datapath
      │
      ├── L3/L4 처리
      │       └── Kernel에서 처리
      │
      └── L7 처리 필요
              └── Envoy로 전달
```

따라서 Cilium의 Sidecarless 구조는 Proxy를 완전히 제거하는 것이 아니라, Pod마다 Proxy를 배치하지 않고 Kernel Datapath와 Node 단위 Proxy를 결합하는 방식이다.

### Cloud Native 프로젝트와의 연결

Cilium은 Kubernetes Network Datapath에 eBPF를 사용한다.

User Space의 `cilium-agent`가 Kubernetes Service, Endpoint, Network Policy 상태를 관리하고 이를 BPF Map에 반영한다. Kernel의 eBPF Program은 Packet마다 Map을 조회하여 Routing, Load Balancing, Policy Enforcement를 수행한다.

```text
Kubernetes API
      │
      ▼
cilium-agent
      │
      ▼
BPF Map
      │
      ▼
eBPF Datapath
```

Hubble은 Cilium Datapath에서 생성된 Flow Event를 수집하고 Kubernetes Pod, Namespace, Identity, Policy Verdict와 결합하여 Network Observability를 제공한다.

Katran은 XDP에 eBPF Program을 연결하여 Network Stack의 앞단에서 L4 Load Balancing을 수행한다.

Tetragon은 Process, System Call, File, Network Event를 Kernel에서 관찰하고 Kubernetes Metadata와 결합하여 Runtime Security 기능을 제공한다.

이들 프로젝트는 공통적으로 User Space에서 설정과 Metadata를 관리하고, Kernel의 eBPF Program이 실제 Event와 Packet을 처리하는 구조를 사용한다.

```text
User Space Control Plane
      │
      ├── Configuration
      ├── Metadata
      ├── Policy
      └── BPF Map Update
              │
              ▼
Kernel eBPF Data Plane
      │
      ├── Packet Processing
      ├── Event Collection
      ├── Policy Enforcement
      └── Load Balancing
```

## Summary

Classic BPF는 Network Packet을 Kernel 내부에서 필터링하여 필요한 Packet만 User Space로 전달하기 위한 Virtual Machine이었다.

```text
Classic BPF
      │
      ├── Packet Field 조회
      ├── 조건 비교
      └── Packet 전달 여부 결정
```

eBPF는 Classic BPF의 실행 모델을 확장해 Linux Kernel의 Networking, Tracing, Security 영역에서 사용할 수 있는 Runtime으로 발전하였다.

```text
eBPF
      │
      ├── 다양한 Program Type
      ├── BPF Map
      ├── Helper Function
      ├── Verifier
      ├── JIT Compiler
      └── 다양한 Kernel Hook
```

eBPF Program은 User Space Loader를 통해 실행 중인 Kernel에 동적으로 Load되고 특정 Hook에 Attach된다.

```text
Program Compile
      │
      ▼
Kernel Load
      │
      ▼
Verifier
      │
      ▼
Hook Attach
      │
      ▼
Event 발생 시 실행
```

Verifier는 Program의 Memory Access와 Control Flow를 검사하며, 검증을 통과한 Program만 Kernel에서 실행된다.

BPF Map은 Program 실행 사이의 상태를 유지하고, 여러 Program과 User Space 사이에서 데이터를 공유한다.

Cloud Native 환경에서는 여러 Container가 동일한 Linux Kernel을 공유하므로 Node Kernel에 Attach된 eBPF Program으로 Application 수정 없이 Node 전체의 Network, Process, System Call, Security Event를 관찰하거나 제어할 수 있다.

Cilium, Hubble, Katran, Tetragon은 이러한 실행 모델을 활용해 Kubernetes Networking, Network Observability, Load Balancing, Runtime Security 기능을 제공한다.