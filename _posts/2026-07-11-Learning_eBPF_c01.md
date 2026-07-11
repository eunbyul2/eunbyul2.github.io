---
layout: post
title: "[Learning eBPF]Chapter 1: What Is eBPF, and Why Is It Important?"
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

eBPF는 기존 Kernel에 이미 존재하는 Hook를 활용한다. 따라서 프로그램을 동적으로 로드하여 즉시 기능을 추가할 수 있다.

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

이 구조는 Cloud Native 환경에서 특히 중요하다. Kubernetes Node는 수백 개의 Pod가 동일한 Kernel을 공유하므로 Kernel Hook 한 곳에 eBPF Program을 연결하면 Node 전체의 Packet, Process, System Call, Security Event를 관찰하거나 제어할 수 있다.

대표적인 활용 사례는 다음과 같다.

| 프로젝트 | 활용 목적 |
|----------|-----------|
| Cilium | Kubernetes Networking, Network Policy, Load Balancing |
| Hubble | Network Flow Observability |
| Katran | XDP 기반 L4 Load Balancer |
| Tetragon | Runtime Security 및 Process Observability |
| bpftool | Program 및 Map 관리 |
| bpftrace | Kernel Tracing |

---

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

예를 들어 IPv4 Packet 여부를 검사하는 경우 EtherType 필드만 확인한다.

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

Packet Header 전체를 Parsing하는 것이 아니라 필요한 Offset만 읽기 때문에 불필요한 연산을 줄일 수 있다.

### Classic BPF의 한계

Classic BPF는 Packet Filtering에 특화되어 설계되었기 때문에 활용 범위가 제한적이었다.

대표적인 제약은 다음과 같다.

- Packet 데이터만 처리 가능
- Program 간 상태 공유 불가
- Kernel Object 접근 불가
- Process 및 File System 정보 접근 불가
- 다양한 Kernel Event에 연결할 수 없음

즉, Packet 하나만 보고 전달 여부를 판단할 수 있을 뿐 Connection Tracking, Network Policy, Runtime Tracing과 같은 기능은 구현할 수 없었다.

### seccomp-bpf

Classic BPF는 이후 Packet Filtering 외에도 System Call Filtering에 활용되었다.

seccomp-bpf는 Process가 System Call을 호출할 때 BPF Filter를 실행하여 허용 여부를 결정한다.

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

이는 BPF가 Network Packet뿐 아니라 Kernel Event에도 적용될 수 있음을 보여준 사례이며, 이후 eBPF가 다양한 Kernel Hook으로 확장되는 기반이 되었다.

## From BPF to eBPF

Classic BPF는 Packet Filtering만을 목적으로 설계된 Virtual Machine이었다. 따라서 Packet Header를 기반으로 단순한 조건 분기와 Packet 전달 여부만 결정할 수 있었다.

eBPF는 이러한 실행 모델을 유지하면서 Linux Kernel 전반에서 사용할 수 있는 범용 Runtime으로 확장되었다.

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

Program은 Hook에 따라 서로 다른 Context를 전달받는다.

예를 들어 XDP Program은 Network Driver 단계의 Packet 정보를 전달받으며, Tracepoint Program은 Kernel Event 정보를 전달받는다.

즉, Program의 실행 방식은 동일하지만 입력 Context만 달라지는 구조이다.

```text
                Kernel

                  │

    ┌─────────────┼──────────────┐

    │             │              │

 Network      Tracepoint      Security

    │             │              │

    ▼             ▼              ▼

         eBPF Program
```

### BPF Map

Classic BPF는 Program 실행이 종료되면 모든 상태가 사라졌다.

eBPF는 이를 해결하기 위해 BPF Map을 도입하였다.

BPF Map은 Kernel Memory에 생성되는 Key-Value 자료구조이며 User Space와 eBPF Program이 동시에 접근할 수 있다.

```text
        User Space

             │

             ▼

      BPF Map Update

             │

             ▼

      +-------------+

      |  BPF Map    |

      +-------------+

         ▲       ▲

         │       │

 Lookup  │       │ Update

         │       │

      eBPF Program
```

Program은 Packet마다 Map을 조회하여 필요한 상태를 가져온다.

대표적인 활용 사례는 다음과 같다.

- Connection Tracking
- Network Policy
- Service Backend
- Packet Statistics

Cilium은 Kubernetes Networking에 필요한 대부분의 Runtime 정보를 BPF Map으로 관리한다.

Service가 생성되거나 Backend Pod가 변경되더라도 Program은 변경되지 않으며 Map만 갱신된다.

이러한 구조는 Control Plane과 Data Plane을 분리하는 핵심 요소이다.

### Helper Function

eBPF Program은 Kernel 내부 함수를 직접 호출하지 않는다.

대신 Kernel이 제공하는 Helper Function을 통해 필요한 기능을 수행한다.

대표적인 기능은 다음과 같다.

- BPF Map 조회
- BPF Map 갱신
- 현재 Process 정보 조회
- 현재 시간 조회
- Packet Redirect

Helper Function은 Program Type마다 사용할 수 있는 종류가 다르다.

이를 통해 Program이 접근할 수 있는 Kernel 기능을 제한하여 안정성을 유지한다.

### Verifier

eBPF Program은 Kernel에 Load되기 전에 반드시 Verifier를 통과해야 한다.

Verifier는 Program이 Kernel을 손상시키지 않는지 정적으로 분석한다.

대표적으로 다음 항목을 검사한다.

- 잘못된 Memory 접근
- Stack 범위 초과
- 종료되지 않는 실행 경로
- 허용되지 않은 Helper 호출

검증을 통과한 Program만 Kernel에 Load된다.

Kernel Module과 가장 큰 차이점이 바로 이 과정이다.

---

