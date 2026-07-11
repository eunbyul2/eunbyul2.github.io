---
layout: post
title: "[Learning eBPF] 00. Chapter 1을 읽기 전에 알아야 하는 Linux와 eBPF 기초"
date: 2026-07-11 14:40:00 +0900
categories: [Study, eBPF, Linux]
tags: [eBPF, Linux, Kernel, Kubernetes, Cilium, Networking]
published: true
---

# Learning eBPF 스터디를 시작하며

CloudNet eBPF 스터디를 통해 **Learning eBPF**를 읽기 시작했다.

처음에는 eBPF를 단순히 "Cilium에서 사용하는 기술" 정도로만 알고 있었다. Kubernetes Networking을 공부하다 보면 Cilium을 많이 접하게 되고, Cilium을 이야기하면 항상 eBPF가 함께 등장하기 때문에 막연하게 "네트워크를 빠르게 처리하는 기술" 정도로 이해하고 있었다.

하지만 Chapter 1을 읽기 시작하자 생각보다 훨씬 많은 배경지식이 필요했다.

책은 eBPF 자체를 먼저 설명하지 않는다.

Linux Kernel이 무엇인지, User Space와 Kernel Space는 무엇인지, System Call은 어떻게 동작하는지, Container는 왜 Host Kernel을 공유하는지 등을 먼저 이해해야 비로소 eBPF가 왜 등장했고 왜 혁신적인 기술인지 이해할 수 있다.

실제로 Chapter 1을 읽으면서 가장 많이 했던 말이 있었다.

> "아... 그래서 이래서 eBPF가 필요한 거였구나."

반대로 Kernel을 이해하지 못하면 eBPF는 그냥 어려운 용어들의 나열처럼 느껴질 가능성이 크다.

그래서 Chapter 1을 정리하기 전에 먼저 공부하면서 새롭게 이해한 Linux와 eBPF의 기초 개념들을 따로 정리해보기로 했다.

이 글은 책 내용을 그대로 번역한 것이 아니라 **Chapter 1을 제대로 이해하기 위해 먼저 알고 있어야 하는 배경지식**을 정리한 글이다.

## 1. 운영체제와 Linux Kernel

### 1-1) 운영체제(OS)란 무엇인가?

컴퓨터에는 CPU, 메모리(RAM), SSD, HDD, 네트워크 카드(NIC), GPU 등 다양한 하드웨어가 존재한다.

애플리케이션이 이러한 하드웨어를 직접 제어할 수 있다면 어떨까?

예를 들어 Chrome이 메모리를 마음대로 사용하고, Python 프로그램이 디스크를 마음대로 수정하고, 어떤 프로그램이 네트워크 카드를 독점해서 사용한다면 다른 프로그램은 정상적으로 동작할 수 없을 것이다.

이러한 문제를 해결하기 위해 존재하는 것이 운영체제(Operating System)이다.

운영체제는 컴퓨터의 모든 하드웨어를 관리하며, 여러 프로그램이 동시에 실행될 수 있도록 자원을 효율적으로 분배하는 역할을 담당한다.

대표적인 운영체제로는 Windows, macOS, Linux가 있다.

우리가 서버 환경에서 가장 많이 사용하는 Linux 역시 운영체제이며, Kubernetes, Docker, PostgreSQL, Nginx 같은 프로그램도 모두 Linux 위에서 실행된다.

하지만 Linux 전체가 하나의 프로그램은 아니다.

Linux에서 가장 중요한 부분은 바로 **Kernel**이다.

### 1-2) Linux Kernel이란?

Kernel은 운영체제의 핵심(Core)이다.

애플리케이션과 하드웨어 사이에서 중간 관리자 역할을 수행하며 컴퓨터에서 가장 높은 권한을 가진 소프트웨어이다.

Kernel은 다음과 같은 핵심 기능을 담당한다.

- CPU 스케줄링
- 프로세스 생성 및 종료
- 메모리 관리
- 파일 시스템 관리
- 디스크 입출력
- 네트워크 통신
- 장치 드라이버 관리
- 보안 및 권한 관리

쉽게 말하면 **컴퓨터에서 일어나는 거의 모든 중요한 작업은 결국 Kernel을 거친다.**

예를 들어 브라우저에서 웹 페이지를 열었다고 생각해보자.

브라우저는 네트워크 카드를 직접 제어하지 않는다.

파일을 저장할 때도 SSD를 직접 제어하지 않는다.

CPU 메모리를 직접 할당하지도 않는다.

이 모든 작업은 Kernel에게 요청해서 수행된다.

즉, Kernel은 모든 프로그램이 공통으로 사용하는 운영체제의 핵심 기능을 제공하는 플랫폼이라고 볼 수 있다.

### 1-3) User Space와 Kernel Space

Linux를 공부하다 보면 반드시 등장하는 개념이 바로 **User Space**와 **Kernel Space**이다. 처음에는 단순히 "권한이 다른 두 공간" 정도로만 이해했지만, eBPF를 공부하면서 이 구조가 왜 필요한지 자연스럽게 이해하게 되었다.

Linux는 안정성과 보안을 위해 프로그램이 실행되는 영역을 크게 두 가지로 구분한다.

첫 번째는 **User Space**이다.

우리가 작성하는 대부분의 프로그램은 모두 User Space에서 실행된다.

예를 들면 다음과 같은 프로그램들이 모두 User Space에서 동작한다.

- Chrome
- Python 프로그램
- Java 프로그램
- Go 프로그램
- Docker
- Kubernetes Component
- Nginx
- MySQL
- PostgreSQL

즉, 우리가 평소에 사용하는 거의 모든 애플리케이션이 User Space에서 실행된다고 생각하면 된다.

반면 **Kernel Space**는 운영체제만 접근할 수 있는 특별한 공간이다.

Kernel이 직접 실행되는 공간이며 CPU, Memory, Storage, Network Device 같은 하드웨어를 제어할 수 있는 권한을 가지고 있다.

여기서 가장 중요한 점은 **User Space 프로그램은 하드웨어를 직접 제어하지 못한다는 것**이다.

예를 들어 Python 프로그램이 있다고 생각해 보자.

```python
file = open("test.txt")
```

우리는 단순히 `open()` 함수 하나를 호출했을 뿐이라고 생각한다.

하지만 실제로는 Python이 SSD를 직접 제어하는 것이 아니다.

내부적으로는

```text
Python

↓

System Call

↓

Linux Kernel

↓

File System

↓

SSD
```

이러한 과정을 거쳐 파일이 열린다.

즉, 실제 파일을 여는 작업은 Kernel이 수행하는 것이다.

네트워크도 마찬가지이다.

브라우저에서 Google에 접속한다고 해서 Chrome이 랜카드를 직접 제어하는 것이 아니다.

실제로는

```text
Chrome

↓

System Call

↓

Linux Kernel

↓

TCP/IP Network Stack

↓

NIC(Network Interface Card)

↓

Internet
```

과정을 거친다.

메모리도 마찬가지이다.

프로그램은 RAM에서 메모리를 직접 가져오는 것이 아니라 Kernel에게 메모리를 요청하고, Kernel이 적절한 공간을 할당해 준다.

결국 우리가 사용하는 거의 모든 프로그램은 실제 중요한 작업을 수행할 때 반드시 Kernel의 도움을 받아야 한다.

Linux가 이렇게 User Space와 Kernel Space를 분리한 이유는 안전성 때문이다.

만약 모든 프로그램이 Kernel 권한을 가진다면 어떤 일이 발생할까?

버그 하나만 있어도 운영체제가 종료될 수 있고, 악성 프로그램은 다른 프로그램의 메모리를 마음대로 읽거나 시스템 파일을 삭제할 수도 있다.

그래서 Linux는 일반 프로그램에게는 제한된 권한만 주고, 중요한 작업은 Kernel만 수행하도록 설계되어 있다.

이 구조 덕분에 Chrome이 종료되더라도 Linux 전체가 종료되지 않고, 하나의 프로그램이 메모리를 잘못 사용하더라도 다른 프로그램까지 영향을 받지 않는다.

### 1-4) System Call

앞에서 계속 System Call이라는 단어가 등장했다.

System Call은 **User Space 프로그램이 Kernel에게 작업을 요청하는 공식적인 인터페이스**이다.

쉽게 말하면

> "Kernel에게 부탁하는 창구"

라고 생각하면 된다.

예를 들어 우리가 아래와 같은 코드를 작성했다고 해보자.

```python
with open("test.txt") as f:
    print(f.read())
```

우리는 단순히 파일을 읽었다고 생각하지만 실제 내부에서는 훨씬 많은 일이 발생한다.

먼저 Python의 `open()` 함수가 호출된다.

Python Runtime은 내부적으로 `openat()`이라는 System Call을 호출한다.

Kernel은 파일이 존재하는지 확인하고, 현재 사용자가 해당 파일을 읽을 권한이 있는지 검사한다.

그 후 파일 시스템을 통해 SSD에서 데이터를 읽어 메모리에 올린다.

마지막으로 그 결과를 다시 Python 프로그램에게 반환한다.

즉, 실제 동작 과정은 다음과 같다.

```text
Python open()

↓

openat() System Call

↓

Linux Kernel

↓

File System

↓

SSD

↓

Linux Kernel

↓

Python
```

네트워크도 동일하다.

예를 들어 브라우저가 웹사이트에 접속할 때도

```text
Chrome

↓

socket()

↓

connect()

↓

send()

↓

recv()

↓

Linux Kernel

↓

TCP/IP Stack

↓

Network Card
```

과정을 거친다.

이처럼 User Space 프로그램은 직접 하드웨어를 사용하는 것이 아니라 항상 System Call을 통해 Kernel에게 요청한다.

그래서 eBPF를 이해하려면 System Call을 반드시 이해해야 한다.

Chapter 1에서도 계속 이야기하지만 eBPF는 Application 내부에서 실행되는 기술이 아니다.

Kernel 내부에서 실행되는 작은 프로그램이다.

즉, System Call이나 Kernel Function 같은 지점(Hook)에 eBPF Program을 연결하여 특정 이벤트가 발생할 때 원하는 동작을 수행하는 것이다.

예를 들어 모든 파일 열기를 기록하고 싶다면 `openat()` System Call에 eBPF Program을 연결하면 된다.

그러면 어떤 프로그램이든 파일을 열려고 하는 순간 Kernel이 해당 eBPF Program을 실행하게 된다.

이것이 eBPF 기반 Observability 도구가 애플리케이션을 수정하지 않고도 시스템 전체를 관찰할 수 있는 이유이다.

### 1-5) 왜 eBPF를 이해하려면 Kernel을 먼저 이해해야 할까?

처음에는 eBPF를 새로운 기술 하나 정도로 생각했다.

하지만 Chapter 1을 읽으면서 가장 크게 느낀 점은 **eBPF는 Kernel을 확장하는 기술이지 애플리케이션 기술이 아니라는 것**이었다.

즉, eBPF를 공부한다는 것은 새로운 프로그래밍 언어를 배우는 것이 아니라 Linux Kernel이 어떻게 동작하는지를 이해하는 과정에 더 가깝다.

앞에서 살펴본 것처럼 모든 프로그램은 결국 Kernel을 거쳐 하드웨어를 사용한다.

따라서 Kernel 안에서 실행되는 eBPF Program은 특정 애플리케이션 하나만 보는 것이 아니라 Kernel을 사용하는 모든 프로세스를 관찰할 수 있다.

이것이 바로 eBPF가 Observability, Networking, Security 분야에서 혁신적인 기술이라고 불리는 이유이다.

Chapter 1도 바로 이 개념을 이해시키기 위해 Kernel부터 설명을 시작한다.

## 2. Linux Networking 기초

Chapter 1을 읽다 보면 XDP, Cilium, Hubble, Sidecar 같은 네트워크 관련 용어가 계속 등장한다.

처음에는 "네트워크를 더 빠르게 처리하는 기술인가 보다." 정도로만 생각했는데, 공부를 하다 보니 결국 이 모든 기술은 Linux Network Stack 위에서 동작한다는 것을 알게 되었다.

즉, eBPF가 네트워크를 빠르게 처리한다고 해서 새로운 네트워크를 만드는 것이 아니라, **기존 Linux Network Stack의 특정 지점(Hook)에 eBPF Program을 연결하여 동작을 변경하거나 추가하는 것**이다.

그래서 먼저 Linux가 네트워크를 어떻게 처리하는지 이해할 필요가 있다.

### 2-1) Linux Network Stack이란?

Network Stack은 쉽게 말해 **Linux가 네트워크 통신을 처리하는 전체 과정**을 의미한다.

우리가 브라우저에서 Google에 접속하거나, Kubernetes Pod가 다른 Pod와 통신하거나, SSH로 서버에 접속할 때도 모두 Linux Network Stack을 거친다.

예를 들어 브라우저에서 https://google.com 에 접속한다고 가정해보자.

우리는 단순히 URL 하나를 입력했을 뿐이라고 생각하지만 실제로는 다음과 같은 과정이 일어난다.

```text
Chrome

↓

Socket 생성

↓

TCP 연결 생성

↓

Linux TCP/IP Stack

↓

Routing

↓

Network Interface

↓

NIC

↓

Internet
```

반대로 외부에서 응답 패킷이 들어오면

```text
Internet

↓

NIC

↓

Linux Network Stack

↓

TCP 처리

↓

Socket

↓

Chrome
```

과정을 거쳐 브라우저까지 데이터가 전달된다.

즉, 네트워크 패킷은 애플리케이션으로 바로 들어가는 것이 아니라 반드시 Linux Kernel의 Network Stack을 통과한다.

이것이 eBPF가 네트워크를 제어할 수 있는 가장 큰 이유이다.

### 2-2) Socket이란?

Socket은 애플리케이션과 네트워크를 연결해 주는 통신 인터페이스이다.

쉽게 말하면 파일을 읽기 위해 File Descriptor를 사용하는 것처럼, 네트워크를 사용하기 위해서는 Socket이 필요하다.

예를 들어 Python에서 HTTP 요청을 보내는 코드를 작성하면

```python
import requests

requests.get("https://google.com")
```

우리는 requests 라이브러리 하나만 사용했지만 내부적으로는

- socket()
- connect()
- send()
- recv()

같은 System Call이 호출된다.

즉,

```text
Python

↓

Socket 생성

↓

Kernel

↓

TCP 연결

↓

Packet 전송
```

이라는 흐름으로 동작한다.

결국 Socket 역시 Kernel이 관리하는 자원이다.

### 2-3) Network Interface와 NIC

Network Interface는 컴퓨터가 네트워크와 연결되기 위한 인터페이스이다.

대표적으로 Linux에서는 다음과 같은 인터페이스를 볼 수 있다.

- eth0
- ens160
- enp3s0
- lo

여기서 eth0나 ens160 같은 인터페이스는 실제 네트워크 카드(NIC)와 연결되어 있으며, lo는 Loopback Interface이다.

NIC(Network Interface Card)는 실제 하드웨어 장치이다.

랜선을 꽂거나 Wi-Fi를 연결하는 장치가 바로 NIC이다.

패킷은 일반적으로 다음과 같은 흐름으로 이동한다.

```text
Application

↓

Socket

↓

Linux Network Stack

↓

Network Interface

↓

NIC

↓

외부 네트워크
```

즉, NIC는 패킷이 컴퓨터를 드나드는 실제 출입구라고 생각하면 이해하기 쉽다.

### 2-4) veth란?

Kubernetes를 공부하다 보면 반드시 등장하는 것이 veth(Virtual Ethernet Pair)이다.

veth는 이름 그대로 가상의 Ethernet 장치이며 항상 두 개가 한 쌍으로 생성된다.

한쪽으로 들어온 패킷은 반드시 반대쪽으로 나온다.

Kubernetes에서는 이 특성을 이용하여 Pod와 Host를 연결한다.

예를 들어 하나의 Pod가 생성되면

```text
Pod

↓

eth0

↓

veth Pair

↓

Host Network Namespace
```

형태로 연결된다.

Pod 내부에서는 자신의 eth0만 보이지만 실제로는 Host와 veth Pair로 연결되어 있는 것이다.

따라서 Pod가 외부와 통신하려면 반드시 veth를 지나 Host Kernel의 Network Stack으로 이동한다.

이 부분은 나중에 Cilium을 이해할 때 매우 중요하다.

왜냐하면 Cilium은 바로 이 veth 지점이나 TC Hook 등에 eBPF Program을 연결하기 때문이다.

### 2-5) iptables란?

iptables는 Linux Kernel의 패킷 필터링 기능이다.

쉽게 말하면 "패킷을 어떻게 처리할 것인지 결정하는 규칙(rule)의 집합"이다.

예를 들어

- 특정 IP는 허용
- 특정 Port는 차단
- 특정 Pod로 전달
- NAT 수행

같은 작업을 수행한다.

기존 Kubernetes에서는 kube-proxy가 iptables Rule을 매우 많이 생성하여 Service Load Balancing과 Packet Forwarding을 수행했다.

하지만 Cluster 규모가 커질수록 Rule 개수가 수천~수만 개까지 증가하면서 성능 저하가 발생할 수 있었다.

Cilium이 등장하면서 이 부분을 eBPF Program으로 대체하게 된다.

즉,

기존에는

```text
Packet

↓

iptables Rule 1

↓

iptables Rule 2

↓

iptables Rule 3

↓

...

↓

Forward
```

였다면,

Cilium은

```text
Packet

↓

eBPF Program

↓

Forward
```

형태로 처리하여 더 빠른 성능을 제공한다.

### 2-6) XDP(eXpress Data Path)

XDP는 Linux Kernel에서 가장 빠르게 패킷을 처리할 수 있는 eBPF 실행 지점(Hook) 중 하나이다.

일반적으로 패킷은 NIC를 거쳐 Linux Network Stack으로 들어온 후 Routing, iptables, TCP/IP 처리 등을 수행한다.

하지만 XDP는 이보다 훨씬 앞단인 **NIC Driver 단계**에서 패킷을 처리한다.

즉,

기존 방식은

```text
NIC

↓

Linux Network Stack

↓

iptables

↓

Routing

↓

Application
```

이었다면,

XDP는

```text
NIC

↓

XDP(eBPF)

↓

Drop / Pass / Redirect

↓

필요한 경우만 Network Stack
```

형태로 동작한다.

필요 없는 패킷은 Network Stack에 들어오기 전에 바로 버릴 수 있기 때문에 CPU 사용량이 크게 줄어든다.

책에서 XDP가 기존 Linux Networking보다 2배 이상 빠르다고 설명하는 이유도 여기에 있다.

Cilium 역시 일부 기능에서 XDP를 사용하여 매우 빠른 Packet Processing을 수행할 수 있다.

## 3. Container와 Kubernetes

Chapter 1 후반부를 읽다 보면 갑자기 Kubernetes, Container, Sidecar, Cloud Native 이야기가 등장한다.

처음에는 "eBPF 책인데 왜 Kubernetes 이야기가 나오지?"라는 생각이 들었다.

하지만 읽다 보니 eBPF가 가장 강력한 이유가 바로 Cloud Native 환경에서 나타난다는 것을 알게 되었다.

그 이유를 이해하려면 먼저 Container가 어떻게 동작하는지 알아야 한다.

### 3-1) Container는 별도의 운영체제가 아니다

Docker나 Kubernetes를 처음 공부하면 Container를 작은 가상머신(VM)처럼 생각하기 쉽다.

하지만 Container는 VM과 다르다.

가상머신은 각각 독립적인 운영체제와 Kernel을 가진다.

예를 들어 하나의 서버에서 Ubuntu VM과 CentOS VM을 동시에 실행한다면 두 VM은 각각 자신의 Kernel을 사용한다.

반면 Container는 별도의 Kernel을 가지지 않는다.

Container 안에는 애플리케이션과 실행에 필요한 라이브러리만 존재하고, 실제 운영체제의 Kernel은 Host Linux Kernel을 그대로 공유한다.

예를 들어 하나의 Kubernetes Node에서 다음과 같은 Pod가 실행되고 있다고 가정해 보자.

```text
Node

├── Frontend Pod
├── Backend Pod
├── Database Pod
└── Monitoring Pod
```

각 Pod는 서로 독립적으로 실행되는 것처럼 보이지만 실제로는 모두 하나의 Host Linux Kernel을 사용한다.

즉,

```text
Frontend

↓

Host Linux Kernel

Backend

↓

Host Linux Kernel

Database

↓

Host Linux Kernel
```

와 같은 구조이다.

Container마다 Kernel이 하나씩 존재하는 것이 아니라 모든 Container가 같은 Kernel을 공유하는 것이다.

이것이 eBPF가 Kubernetes에서 강력한 가장 큰 이유이다.

### 3-2) Namespace란?

Container가 각각 독립적인 환경처럼 보이는 이유는 Linux Namespace 덕분이다.

Namespace는 하나의 Linux 시스템 안에서 서로 다른 프로세스가 서로를 보지 못하도록 격리하는 기능이다.

대표적으로 다음과 같은 Namespace가 있다.

- Process Namespace
- Network Namespace
- Mount Namespace
- PID Namespace
- User Namespace

예를 들어 Process Namespace를 사용하면 Container A에서는 자신의 프로세스만 보인다.

Container B 역시 자신의 프로세스만 보인다.

하지만 실제로는 모두 같은 Host Kernel에서 실행된다.

즉, "격리"는 되어 있지만 "분리"된 운영체제를 사용하는 것은 아니다.

### 3-3) cgroup이란?

Namespace가 "보이는 범위"를 분리한다면 cgroup(Control Group)은 사용할 수 있는 리소스를 제한한다.

예를 들어

- CPU 2개만 사용
- Memory 4GB만 사용
- Disk I/O 제한

같은 설정을 cgroup으로 관리한다.

Kubernetes에서 Pod의 CPU Request, Limit, Memory Limit 역시 내부적으로 cgroup을 이용하여 구현된다.

### 3-4) Pod Packet Flow

Container Networking을 이해하려면 패킷이 어떻게 이동하는지 알아야 한다.

예를 들어 Pod가 외부와 통신하면 패킷은 다음과 같은 경로를 거친다.

```text
Application

↓

Socket

↓

Pod eth0

↓

veth Pair

↓

Host Linux Kernel

↓

Linux Network Stack

↓

NIC

↓

외부 네트워크
```

반대로 외부에서 들어오는 패킷도

```text
NIC

↓

Linux Network Stack

↓

veth Pair

↓

Pod eth0

↓

Application
```

순서로 이동한다.

중요한 점은 패킷이 반드시 Host Linux Kernel을 통과한다는 것이다.

바로 이 지점에 eBPF Program을 연결하면 해당 Node에서 실행되는 모든 Pod의 네트워크를 관찰하거나 제어할 수 있다.

이것이 Chapter 1 Figure 1-4에서 설명하는 핵심 내용이다.

## 4. BPF와 eBPF

Chapter 1을 읽으면서 가장 헷갈렸던 부분이 바로 "eBPF Program"이라는 표현이었다.

처음에는 새로운 프로그래밍 언어인가 싶었고, Java나 Python처럼 별도의 문법이 있는 줄 알았다.

하지만 공부하면서 eBPF는 언어가 아니라 Kernel에서 실행되는 작은 프로그램이라는 것을 이해하게 되었다.

### 4-1) BPF란?

BPF(Berkeley Packet Filter)는 원래 네트워크 패킷을 효율적으로 필터링하기 위해 만들어진 기술이다.

패킷을 Kernel 안에서 검사하여 필요한 패킷만 통과시키는 것이 목적이었다.

즉,

```text
Packet

↓

BPF Program

↓

Accept

또는

Drop
```

와 같은 구조였다.

### 4-2) eBPF란?

기존 BPF는 Packet Filtering만 가능했다.

하지만 Linux Kernel 3.18 이후 여러 기능이 추가되면서 eBPF가 등장했다.

대표적으로

- 64bit Instruction Set
- eBPF Map
- Helper Function
- bpf() System Call
- Verifier

등이 추가되었다.

이후 eBPF는 Packet Filter가 아니라 Kernel 전체를 확장하는 플랫폼으로 발전하게 된다.

### 4-3) eBPF Program이란?

eBPF Program은 일반적인 애플리케이션이 아니다.

Chrome처럼 계속 실행되는 프로그램도 아니고 Python Script처럼 main() 함수부터 시작하는 프로그램도 아니다.

오히려 특정 이벤트가 발생했을 때 잠깐 실행되는 Callback 함수에 가깝다.

예를 들어

- 파일 열기
- Process 실행
- Packet 수신
- TCP Connection 생성

같은 이벤트가 발생하면 해당 Hook에 연결된 eBPF Program이 실행된다.

즉,

```text
Packet 도착

↓

eBPF Program 실행

↓

종료
```

와 같은 형태이다.

### 4-4) Hook이란?

Hook은 Kernel 안에서 특정 이벤트가 발생하는 지점을 의미한다.

예를 들어

- openat()

- execve()

- TCP Packet Receive

- XDP

- TC

같은 위치가 Hook이 될 수 있다.

eBPF는 이러한 Hook에 연결되어 이벤트가 발생할 때마다 실행된다.

### 4-5) Helper Function

eBPF Program에서는 일반 C 함수인 printf(), malloc() 등을 사용할 수 없다.

대신 Kernel이 제공하는 Helper Function을 사용한다.

대표적으로

- bpf_printk()
- bpf_map_lookup_elem()
- bpf_map_update_elem()
- bpf_get_current_pid_tgid()

등이 있다.

즉, 앞에 bpf_가 붙는 함수들은 대부분 eBPF Program에서 사용할 수 있도록 Kernel이 제공하는 API라고 이해하면 된다.

### 4-6) eBPF Map

Map은 eBPF Program과 User Space Program이 데이터를 공유하기 위한 저장소이다.

예를 들어 eBPF Program이 Packet 수를 계산하면 그 결과를 Map에 저장하고, User Space 프로그램인 Hubble이나 bpftool이 그 값을 읽어 화면에 출력할 수 있다.

즉,

```text
Packet

↓

eBPF Program

↓

Map 저장

↓

User Space

↓

CLI 또는 UI 출력
```

와 같은 구조이다.

### 4-7) Verifier

Kernel 안에서 실행되는 프로그램은 매우 위험할 수 있다.

버그 하나만 있어도 Kernel 전체가 종료될 수 있기 때문이다.

그래서 Linux Kernel은 eBPF Program을 실행하기 전에 Verifier라는 검사기를 이용하여 안전성을 확인한다.

Verifier는

- 무한루프 여부
- 잘못된 메모리 접근
- 허용되지 않은 함수 사용

등을 검사한다.

안전하지 않은 프로그램은 아예 Kernel에 로드되지 않는다.

### 4-8) JIT Compiler

Verifier를 통과한 Program은 JIT Compiler를 통해 현재 CPU에서 실행 가능한 Machine Code로 변환된다.

따라서 Interpreter 방식보다 훨씬 빠르게 실행할 수 있다.

Chapter 1에서 eBPF가 높은 성능을 제공한다고 설명하는 이유가 바로 여기에 있다.

## 5. Cilium은 eBPF를 어떻게 사용할까?

처음에는 "Cilium이 eBPF를 사용한다."는 말이 무슨 의미인지 이해하지 못했다.

공부하면서 이해한 내용을 한 문장으로 정리하면 다음과 같다.

**Cilium은 Go로 작성된 관리 프로그램과 Kernel에서 실행되는 여러 eBPF Program으로 구성된 CNI이다.**

Cilium Agent는 Go로 작성되어 있으며

- eBPF Program 생성
- Kernel 로드
- Map 관리
- Kubernetes API와 연동

같은 역할을 수행한다.

실제 Packet Processing은 Kernel 안의 eBPF Program이 담당한다.

즉,

```text
Cilium Agent (Go)

↓

eBPF Program 생성

↓

Kernel Load

↓

Packet 발생

↓

Kernel 안의 eBPF Program 실행

↓

Packet Forwarding
```

이라는 구조이다.

Hubble은 이러한 eBPF Program이 수집한 정보를 시각화하는 Observability 도구이고, Tetragon은 Process 실행이나 System Call 등을 모니터링하는 Security 도구이다.

즉, 세 프로젝트 모두 Kernel 안에서 실행되는 eBPF Program을 기반으로 동작한다.

## 6. 마무리

Chapter 1을 읽으면서 가장 크게 느낀 점은 eBPF를 이해하려면 새로운 기술 하나를 공부하는 것이 아니라 Linux 자체를 이해해야 한다는 것이었다.

Kernel이 무엇인지, System Call이 어떻게 동작하는지, Container가 왜 Kernel을 공유하는지 이해하고 나니 Chapter 1의 내용이 훨씬 자연스럽게 읽히기 시작했다.

다음 글에서는 Learning eBPF Chapter 1의 내용을 흐름에 맞게 따라가며 저자가 왜 eBPF를 혁신적인 플랫폼이라고 설명하는지 정리해보려고 한다.