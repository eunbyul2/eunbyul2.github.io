---
layout: post
title: "[Learning eBPF] Chapter 9. eBPF for Security"
date: 2026-09-19 16:06:00 +0900
categories: [eBPF, Security]
tags: [eBPF, Security, Seccomp, Falco, BPF LSM, Tetragon, Syscall, Runtime Security, Kubernetes]
published: true
---

# Learning eBPF Chapter 9 

## eBPF for Security


## 1. Security Observability에는 Policy와 Context가 필요하다

일반적인 Observability 도구는 시스템에서 어떤 이벤트가 발생했는지를 보여주는 데 초점이 있습니다.

하지만 보안 도구는 단순히 이벤트를 기록하는 것에서 끝나지 않고, **정상적인 행동과 의심스러운 행동을 구분할 수 있어야 합니다.**

예를 들어 어떤 애플리케이션이 평소 `/home/<username>/<filename>`에 파일을 쓰는 것은 정상적인 동작일 수 있습니다. 반면 같은 애플리케이션이 `/etc/passwd` 같은 민감한 파일을 수정한다면 보안적으로 확인할 필요가 있습니다.

이처럼 **어떤 행동이 정상이고 어떤 행동이 비정상인지 정의하는 것이 Policy**입니다.

또한 Policy는 정상적인 실행 경로뿐 아니라 **Error Path**도 고려해야 합니다.

예를 들어 평소에는 외부 통신을 하지 않는 애플리케이션이라도 디스크가 가득 찼을 때 Alert를 보내기 위해 Network Message를 전송할 수 있습니다. 평소에는 보기 어려운 행동이지만 공격은 아닙니다.

즉,

```text
Unusual ≠ Malicious
```

입니다.

보안 이벤트를 조사할 때는 **Contextual Information**도 중요합니다.

단순히 Policy 위반 이벤트가 발생했다는 사실만으로는 부족하고, 실제로 어떤 Process가 실행했는지, 무엇이 영향을 받았는지, 언제 발생했는지 같은 정보가 있어야 원인을 파악할 수 있습니다.

따라서 Security Observability에서는 다음 두 가지가 중요합니다.

```text
Policy
+ 
Context
```

<!-- Figure 9-1 삽입 -->

---

## 2. Using System Calls for Security Events

System Call(syscall)은 **User Space Application과 Kernel 사이의 Interface**입니다.

애플리케이션이 사용할 수 있는 syscall을 제한하면, 그 애플리케이션이 할 수 있는 동작도 제한할 수 있습니다.

예를 들어 애플리케이션이 `open*()` 계열 syscall을 사용할 수 없게 만들면 파일을 열 수 없습니다.

만약 원래 파일을 열 필요가 없는 애플리케이션이라면, 이런 제한을 적용해서 애플리케이션이 침해되더라도 악의적으로 파일을 여는 행동을 막을 수 있습니다.

Docker나 Kubernetes를 사용했다면 이런 방식의 대표적인 기능인 **seccomp**를 접했을 가능성이 높습니다.

---

## 3. Seccomp

`seccomp`는 **SECure COMPuting**의 줄임말입니다.

초기 seccomp의 **strict mode**는 프로세스가 사용할 수 있는 syscall을 아주 작은 집합으로 제한했습니다.

```text
read()
write()
_exit()
sigreturn()
```

하지만 실제 애플리케이션에서는 더 많은 syscall이 필요합니다.

그렇다고 Linux에 존재하는 400개 이상의 syscall을 모두 사용할 필요도 없습니다.

그래서 더 유연한 방식인 **seccomp-bpf**가 사용됩니다.

### seccomp-bpf

seccomp-bpf는 고정된 syscall 집합 대신 **BPF Filter를 사용해서 어떤 syscall을 허용하고 어떤 syscall을 차단할지 결정**합니다.

syscall이 호출될 때마다 Filter가 실행되고, 결과에 따라 다음과 같은 Action을 할 수 있습니다.

- syscall 허용
- User Space Application에 Error 반환
- Thread 종료
- User Space Application에 Notification 전달

또한 syscall Argument도 판단에 사용할 수 있습니다.

다만 Pointer 형태로 전달되는 Argument는 dereference할 수 없기 때문에, Pointer가 가리키는 실제 내용을 확인할 수는 없습니다.

이 점은 seccomp Profile의 표현력을 제한합니다.

또 seccomp Profile은 Process가 시작될 때 적용해야 하며, 실행 중인 Process에 적용된 Profile을 자유롭게 수정하는 방식은 아닙니다.

### Docker의 seccomp Profile

seccomp-bpf를 사용할 때 직접 BPF 코드를 작성하지 않아도 됩니다.

보통 사람이 읽을 수 있는 seccomp Profile을 기반으로 Filter가 만들어집니다.

Docker의 Default Profile은 다양한 일반 Container에서 사용할 수 있도록 만든 범용 Profile이기 때문에 대부분의 syscall을 허용하고, 일반적인 Application에서 사용할 가능성이 낮은 일부 syscall만 차단합니다.

교재에서는 일반적인 Container Application이 실제로 사용하는 syscall은 대략 **40~70개 정도**라고 설명합니다.

따라서 보안성을 높이려면 각 Application이 실제로 사용하는 syscall만 허용하는 더 제한적인 Profile을 사용하는 것이 좋습니다.

---

## 4. Generating Seccomp Profiles

문제는 개발자가 자신의 Application이 실제로 어떤 syscall을 사용하는지 정확히 알기 어렵다는 점입니다.

개발자는 어떤 파일을 여는지는 알 수 있지만, 실제로 `open()`을 사용하는지 `openat()`을 사용하는지까지는 모를 수 있습니다.

따라서 seccomp Profile을 사람이 직접 작성하기보다, **Application이 실제로 사용하는 syscall을 관찰해서 자동으로 Profile을 생성하는 방식**이 유용합니다.

초기에는 `strace`를 이용해서 syscall 목록을 수집하기도 했습니다.

하지만 Cloud Native 환경에서는 특정 Container나 Kubernetes Pod를 대상으로 사용하기가 불편할 수 있습니다.

이 때문에 eBPF를 사용해서 syscall 정보를 수집하고, Kubernetes나 OCI Runtime이 사용할 수 있는 seccomp Profile을 생성하는 도구들이 등장했습니다.

교재에서는 다음 예를 소개합니다.

- Inspektor Gadget의 seccomp profiler
- Red Hat의 OCI Runtime Hook 기반 seccomp profiler

### `sys_enter` Tracepoint

이런 Profiler는 eBPF Program을 syscall 진입 지점에 Attach해서 어떤 syscall이 사용되었는지 기록합니다.

```text
Application
    ↓
syscall
    ↓
sys_enter
    ↓
eBPF Program
    ↓
사용된 syscall 기록
```

Red Hat OCI Runtime Hook 예제에서는 `enter_trace` eBPF Program을 Kernel에 Load하고 `raw_syscalls:sys_enter` Tracepoint에 Attach합니다.

```go
enterTrace, err := m.LoadTracepoint("enter_trace")

if err := m.AttachTracepoint("raw_syscalls:sys_enter", enterTrace); err != nil {
    return fmt.Errorf("error attaching to tracepoint: %v", err)
}
```

이 Profiler들은 eBPF를 사용해서 **어떤 syscall이 사용되었는지 추적하고 seccomp Profile을 생성**합니다.

실제로 해당 Profile을 Enforcement하는 것은 seccomp입니다.

### Error Path도 포함해야 한다

Profiler를 사용할 때는 Application을 충분히 실행해서 정상적으로 사용할 수 있는 syscall 목록을 모두 포함해야 합니다.

앞에서 본 것처럼 Error Path에서만 사용하는 syscall도 있을 수 있습니다.

필요한 syscall이 Profile에서 빠지면 장애 상황에서 Application이 정상적으로 동작하지 못할 수 있습니다.

---

## 5. Syscall-Tracking Security Tools

syscall을 추적하는 대표적인 보안 도구로 **Falco**가 있습니다.

Falco는 Security Alert를 제공하는 CNCF Project입니다.

사용자는 어떤 Event가 보안적으로 중요한지 Rule을 정의할 수 있고, 실제 Event가 해당 Policy와 맞지 않으면 Alert를 생성할 수 있습니다.

Falco의 Kernel Module Driver와 eBPF Driver는 모두 syscall에 Attach합니다.

```c
BPF_PROBE("raw_syscalls/", sys_enter, sys_enter_args)
BPF_PROBE("raw_syscalls/", sys_exit, sys_exit_args)
```

eBPF Program은 동적으로 Load할 수 있고 이미 실행 중인 Process의 Event도 감지할 수 있습니다.

따라서 Application이나 Application 설정을 수정하지 않고도 실행 중인 Workload에 Policy를 적용할 수 있습니다.

이 점은 Process가 시작될 때 적용해야 하는 seccomp Profile과 차이가 있습니다.

---

## 6. TOCTOU 문제

syscall entry point를 보안 도구에 사용하는 데는 문제가 있습니다.

바로 **TOCTOU(Time Of Check to Time Of Use)** 문제입니다.

eBPF Program이 syscall entry에서 실행되면 User Space가 전달한 Argument를 확인할 수 있습니다.

그런데 Argument가 Pointer라면 Kernel은 실제로 동작하기 전에 Pointer가 가리키는 데이터를 자신의 Kernel Data Structure로 복사합니다.

이 과정에서 다음과 같은 문제가 생길 수 있습니다.

```text
eBPF Program이 User Space Data 확인
        ↓
공격자가 Data 변경
        ↓
Kernel이 변경된 Data를 복사
```

즉 **eBPF가 검사한 값과 Kernel이 실제로 사용하는 값이 달라질 수 있습니다.**

<!-- Figure 9-2 삽입 -->

이 때문에 syscall entry point는 Observability에는 편리하지만, 보안 Tool에서 신뢰할 수 있는 Enforcement 지점으로 사용하기에는 부족할 수 있습니다.

### `sys_exit`를 이용하는 방법

Sysmon for Linux는 syscall entry와 exit에 모두 Attach합니다.

syscall이 끝난 뒤에는 Kernel Data Structure를 이용해서 더 정확한 정보를 확인할 수 있습니다.

하지만 syscall이 이미 완료된 뒤이기 때문에 실제 동작을 막을 수는 없습니다.

따라서 Kernel이 실제로 사용할 정보가 준비된 뒤, 실제 동작을 수행하기 전에 검사할 수 있는 지점이 필요합니다.

이 역할을 할 수 있는 Interface가 **Linux Security Module(LSM) API**입니다.

---

## 7. BPF LSM

LSM Interface는 Kernel이 Kernel Data Structure를 이용해 실제 동작을 수행하기 직전에 호출되는 Hook을 제공합니다.

이 Hook에서 해당 동작을 허용할지 거부할지 결정할 수 있습니다.

원래 LSM은 보안 기능을 Kernel Module 형태로 구현할 수 있도록 제공된 Interface입니다.

**BPF LSM은 eBPF Program을 동일한 LSM Hook에 Attach할 수 있도록 확장한 기능**입니다.

<!-- Figure 9-3 삽입 -->

LSM Hook은 수백 개가 있으며, syscall과 LSM Hook이 1:1로 대응하는 것은 아닙니다.

하나의 syscall을 처리하는 과정에서 보안적으로 중요한 동작이 있다면 하나 이상의 LSM Hook이 실행될 수 있습니다.

### `path_chmod` 예제

교재에서는 `chmod` 처리 과정에서 실행되는 `path_chmod` Hook 예제를 보여줍니다.

```c
SEC("lsm/path_chmod")
int BPF_PROG(path_chmod, const struct path *path, umode_t mode)
{
    bpf_printk("Change mode of file name %s\n", path->dentry->d_iname);
    return 0;
}
```

이 Program은 변경하려는 파일 이름을 출력하고 항상 `0`을 반환합니다.

`0`을 반환하면 동작을 허용하고, Non-zero 값을 반환하면 Kernel이 해당 동작을 진행하지 않도록 할 수 있습니다.

`path`는 대상 File을 나타내는 Kernel Data Structure이고, `mode`는 변경하려는 Mode 값입니다.

파일 이름은 다음 Field에서 확인할 수 있습니다.

```c
path->dentry->d_iname
```

이처럼 BPF LSM을 이용하면 Kernel 내부에서 직접 Policy를 확인하고 동작을 허용하거나 거부할 수 있습니다.

---

## 8. Cilium Tetragon

Tetragon은 Cilium Project의 일부이며, Kubernetes 환경에서 사용하는 eBPF 기반 Security Tool입니다.

Tetragon은 LSM Hook만 사용하는 대신 **Linux Kernel 내부의 여러 Function에 eBPF Program을 Attach할 수 있는 Framework**를 제공합니다.

Kubernetes에서는 `TracingPolicy`라는 Custom Resource를 사용합니다.

TracingPolicy에서는 다음을 정의할 수 있습니다.

- eBPF Program을 Attach할 Event
- eBPF Code가 검사할 Condition
- Condition이 만족됐을 때 수행할 Action

교재의 예시는 다음과 같습니다.

```yaml
spec:
  kprobes:
  - call: "fd_install"
    ...
    matchArgs:
    - index: 1
      operator: "Prefix"
      values:
      - "/etc/"
```

이 Policy는 Kernel 내부 Function인 `fd_install`에 Attach합니다.

`fd`는 File Descriptor를 의미하고, `fd_install`은 File Pointer를 File Descriptor Array에 설치하는 Function입니다.

이 Function은 File Data Structure가 Kernel에 준비된 뒤 호출되기 때문에 파일 이름을 확인하기에 적절한 지점입니다.

예제 Policy에서는 File Name이 `/etc/`로 시작하는 경우에 관심을 가집니다.

---

## 9. Attaching to Internal Kernel Functions

System Call Interface와 LSM Interface는 Linux Kernel에서 Stable Interface로 정의되어 있습니다.

반면 Kernel 내부 Function은 공식적으로 Stable하다고 보장되지 않을 수 있습니다.

하지만 오랫동안 변경되지 않아 사실상 안정적으로 사용되는 Function도 있습니다.

Tetragon Contributor 중에는 Kernel Developer도 있으며, 이들은 Kernel 내부 구조에 대한 지식을 이용해서 Security 목적으로 eBPF Program을 Attach하기 좋은 지점을 선택합니다.

이런 방식으로 다음과 같은 Event를 관찰할 수 있습니다.

- File Operation
- Network Activity
- Program Execution
- Privilege Change

이러한 Event들은 공격 과정에서 나타날 수 있는 대표적인 행동입니다.

또 Tetragon eBPF Program은 Kernel 내부에서 Contextual Information을 이용해 Security Policy를 판단할 수 있습니다.

모든 Event를 User Space로 보내지 않고, Policy를 벗어난 Security Event만 User Space로 전달할 수도 있습니다.

---

## 10. Preventative Security

많은 eBPF 기반 Security Tool은 악성 Event를 탐지한 뒤 User Space Application에 알리고, User Space에서 Action을 수행하는 방식으로 동작합니다.

하지만 이 과정은 **Asynchronous**합니다.

```text
Kernel에서 Event 탐지
    ↓
User Space에 Notification
    ↓
User Space에서 Action 수행
```

이 사이에 공격이 계속될 수 있습니다.

<!-- Figure 9-4 삽입 -->

Kernel 5.3 이상에서는 `bpf_send_signal()` Helper Function을 사용할 수 있습니다.

Tetragon은 이 Function을 이용해서 Preventative Security를 구현합니다.

TracingPolicy에서 `Sigkill` Action을 정의하면 조건에 맞는 Event가 발생했을 때 Tetragon eBPF Code가 `SIGKILL` Signal을 발생시켜 해당 Process를 종료할 수 있습니다.

```text
Policy 위반 Event
    ↓
Tetragon eBPF
    ↓
SIGKILL
    ↓
Process 종료
```

<!-- Figure 9-5 삽입 -->

이 방식은 Kernel 안에서 동기적으로 처리되기 때문에, Policy를 벗어난 행동이 완료되기 전에 Process를 종료할 수 있습니다.

다만 잘못된 Policy는 정상 Application까지 종료할 수 있기 때문에 주의해야 합니다.

처음에는 **Audit Mode**로 Security Event만 생성하고, Policy가 정상 동작을 방해하지 않는지 확인한 뒤 SIGKILL Enforcement를 적용할 수 있습니다.

---

## 11. Network Security

Chapter 8에서 살펴본 것처럼 eBPF는 Network Security에도 사용할 수 있습니다.

### Firewall과 DDoS Protection

Network Packet이 Ingress Path의 초기에 들어왔을 때 eBPF Program으로 검사하고 Drop할 수 있습니다.

특히 XDP Program을 Hardware에 Offload하는 경우 악성 Packet이 CPU에 도달하기 전에 처리할 수도 있습니다.

### Network Policy

Kubernetes에서 어떤 Service가 서로 통신할 수 있는지를 정의하는 Network Policy도 eBPF Program으로 Enforcement할 수 있습니다.

Policy를 벗어난 Packet이라고 판단되면 해당 Packet을 Drop할 수 있습니다.

Network Security에서는 단순히 악성 Traffic을 Audit하는 것보다 실제로 Packet을 Drop하는 **Preventative Mode**가 많이 사용됩니다.

반면 Runtime Security Tool은 False Positive 때문에 Audit Mode로 사용하는 경우가 많습니다.

교재에서는 eBPF가 더 정교하고 세밀한 Security Control을 가능하게 하면서 앞으로 Network Security 외의 영역에서도 Preventative Security가 더 많이 사용될 수 있다고 설명합니다.

---

# Summary

이번 장에서는 eBPF가 Security 분야에서 어떻게 발전해왔는지 살펴봤습니다.

처음에는 syscall을 관찰하거나 제한하는 낮은 수준의 방식에서 시작하지만, 점점 더 Kernel 내부의 Security Hook과 Context를 활용하는 방식으로 발전합니다.

전체 흐름을 정리하면 다음과 같습니다.

```text
Security Observability
        ↓
Policy + Context
        ↓
Syscall Tracking
        ↓
Seccomp
        ↓
Seccomp Profile 자동 생성
        ↓
Falco
        ↓
TOCTOU 문제
        ↓
BPF LSM
        ↓
Tetragon
        ↓
Preventative Security
```

eBPF를 이용하면 Security Event를 단순히 관찰하는 것뿐 아니라, Kernel 내부에서 Policy를 확인하고 Event를 Filter하거나 Runtime Enforcement를 수행할 수 있습니다.

Chapter 9의 핵심은 **eBPF Security가 단순 Event Detection에서 In-Kernel Policy Check와 Runtime Enforcement 방향으로 발전하고 있다는 점**입니다.
