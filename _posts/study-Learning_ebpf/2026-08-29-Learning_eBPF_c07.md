---
layout: post
title: "[Learning eBPF] Chapter 7: eBPF Program and Attachment Types"
date: 2026-08-29 19:06:00 +0900
categories: [eBPF, Linux, Kernel]
tags: [eBPF, BPF, kprobe, tracepoint, fentry, fexit, XDP, TC, cgroup, LSM]
published: true
---

# Learning eBPF Chapter 7

## eBPF Program and Attachment Types

Chapter 6에서는 eBPF Program이 Kernel에 Load되기 전에 Verifier가 어떤 내용을 검사하는지 살펴봤다.

이번 Chapter 7에서는 그 다음 단계인 **eBPF Program의 종류와 어디에 Attach되는지**를 다룬다.

eBPF Program은 모두 같은 방식으로 실행되는 것이 아니다.  
어떤 Event를 보고 싶은지에 따라 Program Type이 달라지고, Program Type에 따라 받을 수 있는 Context, 사용할 수 있는 Helper Function, Return Value의 의미도 달라진다.

예를 들어 Process 실행을 보고 싶다면 kprobe나 tracepoint를 사용할 수 있고, Network Interface로 들어오는 Packet을 아주 이른 시점에 처리하고 싶다면 XDP를 사용할 수 있다.

즉 이번 Chapter의 핵심은 다음과 같다.

```text
어떤 Event를 처리하고 싶은가?
        ↓
어디에 eBPF Program을 Attach할 것인가?
        ↓
Program Type / Attachment Type 결정
        ↓
사용 가능한 Context와 Helper Function 결정
        ↓
Return Value의 의미도 결정
```

---

# 1. eBPF Program Type과 Attachment Point

Kernel 안에는 eBPF Program을 연결할 수 있는 지점이 여러 곳 있다.

Chapter 2에서 사용했던 `execve()` kprobe도 하나의 Attachment Point였고, Chapter 3에서 사용한 XDP도 Network Interface에 존재하는 Attachment Point 중 하나였다.

책에서는 당시 기준으로 `uapi/linux/bpf.h`에 약 30개의 Program Type과 40개가 넘는 Attachment Type이 있다고 설명한다.

여기서 두 개념을 구분해야 한다.

- **Program Type**: 어떤 종류의 eBPF Program인지
- **Attachment Type / Attachment Point**: 그 Program을 Kernel의 정확히 어느 지점에 연결할 것인지

예를 들어 XDP Program은 XDP Hook에 연결되기 때문에 비교적 관계가 단순하다.

반면 Cgroup 관련 Program처럼 하나의 Program Type이 여러 위치에 Attach될 수 있는 경우에는 더 구체적인 Attachment Type이 필요하다.

---

# 2. Context는 Program Type마다 다르다

모든 eBPF Program은 실행될 때 Context를 전달받는다.

보통 코드에서 다음과 같이 볼 수 있다.

```c
int program(void *ctx)
```

처음에는 `ctx`가 그냥 모든 Program에서 비슷한 정보라고 생각하기 쉬운데 실제로는 그렇지 않다.

`ctx`가 가리키는 구조체는 **어떤 Event 때문에 Program이 실행되었는지**에 따라 달라진다.

예를 들어 XDP Program은 Network Packet을 처리해야 하므로 Packet과 관련된 Context를 받는다.

```c
struct xdp_md *ctx
```

반면 Tracepoint Program은 특정 Kernel Event에서 전달되는 Tracepoint 정보를 받는다.

따라서 Tracepoint Program에서 XDP처럼 `ctx->data`, `ctx->data_end`를 사용하려고 해서는 안 된다.

```text
XDP Event
→ Packet 관련 Context

Tracepoint Event
→ Tracepoint에서 정의된 Context

kprobe
→ Kernel Function 호출 시점의 Register / Argument 정보

fentry
→ 대상 Kernel Function의 Argument
```

이 구분은 Verifier와도 연결된다.

Verifier는 Program Type을 알고 있기 때문에 해당 Program에서 Context의 어떤 부분까지 접근할 수 있는지 검사한다. 잘못된 Context 영역에 접근하면 `invalid bpf_context access`와 같은 오류로 Load가 거부될 수 있다.

---

# 3. Helper Function도 아무거나 사용할 수 없다

eBPF Program에서 Kernel 기능을 직접 아무렇게나 호출할 수는 없다.

대신 Kernel이 eBPF Program에 허용한 **Helper Function**을 사용한다.

그런데 Helper Function도 Program Type마다 사용할 수 있는 범위가 다르다.

Chapter 6에서 봤던 예가 `bpf_get_current_pid_tgid()`이다.

이 Helper는 현재 Process와 Thread 정보를 가져오는 Function이다.

하지만 XDP Program에서는 사용할 수 없다.

왜냐하면 XDP Program은 Network Interface로 Packet이 들어오는 시점에 실행되며, 그 Packet을 발생시킨 현재 User Process라는 개념이 존재하지 않을 수 있기 때문이다.

```text
Process 실행
→ 현재 PID라는 Context가 의미 있음

NIC로 Packet 수신
→ 현재 PID라는 Context가 의미 없음
```

따라서 Verifier는 XDP Program에서 이 Helper를 호출하려 하면 허용하지 않는다.

현재 Kernel에서 Program Type별로 어떤 Helper를 사용할 수 있는지는 다음 명령으로 확인할 수 있다.

```bash
bpftool feature
```

Helper Function은 Linux Kernel의 UAPI 일부이기 때문에 한 번 공개된 Interface는 이후 Kernel에서도 가능한 한 호환성이 유지된다.

---

# 4. Return Value의 의미도 Program Type마다 다르다

eBPF Program은 일반적으로 마지막에 값을 Return한다.

```c
return 0;
```

그런데 이 Return Value의 의미 역시 Program Type마다 다르다.

대표적인 예가 XDP다.

XDP Program은 Packet을 처리한 뒤 Kernel에 **이 Packet을 어떻게 할 것인지** 알려줘야 한다.

```c
return XDP_PASS;
```

주요 XDP Return Code는 다음과 같다.

| Return Value | 의미 |
|---|---|
| `XDP_PASS` | 일반 Network Stack으로 Packet 전달 |
| `XDP_DROP` | Packet 폐기 |
| `XDP_TX` | 들어온 Interface로 다시 전송 |
| `XDP_REDIRECT` | 다른 Interface 또는 대상으로 Redirect |
| `XDP_ABORTED` | 비정상 처리 |

반면 단순 Tracepoint Program에서는 Network Packet 자체가 없기 때문에 `XDP_DROP` 같은 값은 아무 의미가 없다.

즉 Return Value는 단순한 Program 종료 값이 아니라 **Program Type에 따라 Kernel에게 내리는 Verdict가 될 수도 있다.**

---

# 5. Helper Function과 Kfunc의 차이

eBPF에서는 Helper Function 외에도 **kfunc**라는 방식으로 Kernel Function을 호출할 수 있다.

Helper Function은 eBPF를 위해 안정된 Interface로 제공되는 Function이다.

반면 kfunc는 Kernel 내부 Function 중 일부를 BPF Subsystem에 등록하여 eBPF Program에서 사용할 수 있게 만든 것이다.

차이는 호환성이다.

```text
Helper Function
→ eBPF용으로 제공되는 안정적인 UAPI
→ 호환성 보장

kfunc
→ Kernel 내부 Function을 eBPF에서 사용할 수 있도록 공개
→ Kernel Version에 따라 변경될 수 있음
```

또한 모든 Program Type이 모든 kfunc를 호출할 수 있는 것은 아니다.

특정 kfunc가 어떤 Program Type에서 사용 가능한지 Kernel에 등록되어 있으며, Verifier가 이를 검사한다.

---

# 6. Tracing 계열 Program

eBPF Program Type은 크게 보면 **Tracing 계열**과 **Networking 계열**로 나눠볼 수 있다.

먼저 Tracing 계열에는 다음과 같은 방식이 있다.

- kprobe / kretprobe
- fentry / fexit
- tracepoint
- raw tracepoint
- BTF-enabled tracepoint
- perf event
- uprobe / uretprobe

이 Program들은 Kernel이나 Application에서 어떤 Event가 발생했는지 관찰하는 데 많이 사용된다.

책의 예제에서는 모두 `execve()`와 관련된 지점에 여러 Program을 Attach한다.

```bash
sudo bpftool perf show
```

예시는 대략 다음과 같은 형태다.

```text
kprobe          __x64_sys_execve
kprobe          do_execve
tracepoint      sys_enter_execve
raw_tracepoint  sched_process_exec
```

같은 `execve()` 실행을 보고 있더라도 어디에 Attach하느냐에 따라 받아오는 Context와 Program 작성 방식이 달라진다.

---

# 7. Kprobe와 Kretprobe

## Kprobe

Kprobe는 Kernel Function의 특정 지점에 동적으로 Probe를 걸 수 있는 기능이다.

가장 흔한 방식은 Kernel Function의 시작점에 Attach하는 것이다.

```text
Kernel Function 시작
        ↓
      kprobe
        ↓
eBPF Program 실행
```

예를 들어 `execve()` System Call에 Attach할 수 있다.

```c
SEC("ksyscall/execve")
int BPF_KPROBE_SYSCALL(kprobe_sys_execve, char *pathname)
```

또는 System Call이 아닌 일반 Kernel Function에도 Attach할 수 있다.

```c
SEC("kprobe/do_execve")
int BPF_KPROBE(kprobe_do_execve, struct filename *filename)
```

여기서 중요한 차이가 있다.

`execve()` System Call에서 User Space가 넘긴 filename은 문자열 Pointer이다.

```c
char *pathname
```

하지만 Kernel 내부의 `do_execve()` Function에서는 Kernel 내부 자료구조인 `struct filename *`을 사용한다.

```c
int do_execve(
    struct filename *filename,
    const char __user *const __user *__argv,
    const char __user *const __user *__envp
)
```

즉 Kernel 내부 Function에 kprobe를 붙이려면 **그 Kernel Function의 실제 Function Signature를 알고 있어야 한다.**

이 부분에서 eBPF Programming이 단순히 C 문법만 아는 것으로 끝나는 것이 아니라 Kernel 내부 구조를 어느 정도 이해해야 한다는 점이 보인다.

---

## Kretprobe

kretprobe는 Function이 시작할 때가 아니라 **Return할 때** 실행된다.

```text
Kernel Function
      ↓
Function 실행
      ↓
    return
      ↓
  kretprobe
```

따라서 Function의 Return Value를 확인할 때 사용할 수 있다.

```c
SEC("kretprobe/do_unlinkat")
int BPF_KRETPROBE(do_unlinkat_exit, long ret)
```

다만 kretprobe에서는 보통 Function이 처음 호출될 때 전달된 Argument를 그대로 사용하기가 어렵다.

이 점을 개선할 수 있는 방식이 fentry/fexit이다.

---

# 8. Fentry / Fexit

Linux Kernel 5.5부터 x86에서 BPF Trampoline 기반의 fentry/fexit가 도입되었다.

최근 Kernel에서는 Kernel Function의 Entry/Exit를 추적할 때 kprobe/kretprobe보다 fentry/fexit를 우선적으로 고려할 수 있다.

fentry는 Function 시작점에 Attach한다.

```c
SEC("fentry/do_execve")
int BPF_PROG(fentry_execve, struct filename *filename)
```

구조만 보면 kprobe와 상당히 비슷하다.

```text
do_execve()
   ↓
fentry
   ↓
eBPF Program
```

fexit는 Function 종료 지점에서 실행된다.

특히 fexit는 kretprobe와 달리 **Return Value뿐 아니라 원래 Function Argument까지 같이 사용할 수 있다.**

kretprobe 예제는 다음과 같다.

```c
SEC("kretprobe/do_unlinkat")
int BPF_KRETPROBE(do_unlinkat_exit, long ret)
```

여기서는 Return Value인 `ret`만 받는다.

반면 fexit는 다음과 같다.

```c
SEC("fexit/do_unlinkat")
int BPF_PROG(
    do_unlinkat_exit,
    int dfd,
    struct filename *name,
    long ret
)
```

따라서 다음 정보를 한 번에 볼 수 있다.

```text
dfd
name
ret
```

즉 "어떤 파일에 대해 요청이 들어왔고 결과가 성공했는지 실패했는지"를 같이 확인하기가 더 편하다.

---

# 9. Tracepoint

Tracepoint는 Kernel Source Code에 미리 만들어져 있는 Tracing 지점이다.

Kprobe가 비교적 자유롭게 Kernel Function에 Attach하는 방식이라면, Tracepoint는 Kernel 개발자가 **여기는 추적 지점으로 사용해도 된다**고 미리 정의해 둔 위치라고 볼 수 있다.

사용 가능한 Tracepoint는 다음 파일에서 확인할 수 있다.

```bash
cat /sys/kernel/tracing/available_events
```

예를 들어 다음과 같은 Event가 있다.

```text
syscalls:sys_enter_execve
syscalls:sys_exit_execve
```

eBPF Program은 다음과 같이 Attach할 수 있다.

```c
SEC("tp/syscalls/sys_enter_execve")
```

Tracepoint의 장점은 Kernel Version이 바뀌어도 임의의 Kernel 내부 Instruction에 kprobe를 거는 것보다 상대적으로 안정적이라는 점이다.

---

# 10. Tracepoint Context는 어떻게 알 수 있을까?

Tracepoint마다 전달하는 Context 구조가 다르다.

BTF를 사용하지 않는 경우에는 해당 Tracepoint의 `format` 파일을 확인할 수 있다.

```bash
cat /sys/kernel/tracing/events/syscalls/sys_enter_execve/format
```

여기에는 각 Field의 Offset과 Size가 나온다.

예를 들면 `sys_enter_execve`에는 다음과 같은 정보가 포함된다.

```text
filename
argv
envp
```

그리고 이 Format에 맞춰 eBPF 코드에서 구조체를 직접 만들 수 있다.

```c
struct my_syscalls_enter_execve {
    unsigned short common_type;
    unsigned char common_flags;
    unsigned char common_preempt_count;
    int common_pid;
    long syscall_nr;
    long filename_ptr;
    long argv_ptr;
    long envp_ptr;
};
```

Program에서는 이 구조체 Pointer를 Context로 받는다.

```c
int tp_sys_enter_execve(
    struct my_syscalls_enter_execve *ctx
)
```

그리고 필요한 값을 읽을 수 있다.

```c
bpf_probe_read_user_str(
    &data.command,
    sizeof(data.command),
    ctx->filename_ptr
);
```

다만 구조체를 직접 정의하면 Kernel Version이 달라졌을 때 실제 구조와 맞지 않을 가능성이 있다.

이 문제를 줄이는 방법이 BTF를 사용하는 것이다.

---

# 11. BTF-enabled Tracepoint

Chapter 5에서 배운 BTF는 CO-RE뿐만 아니라 Tracepoint Context에도 사용할 수 있다.

BTF가 제공되는 환경에서는 `vmlinux.h`에 Kernel의 Type 정보가 들어 있기 때문에 Tracepoint 구조체를 직접 다시 정의할 필요가 줄어든다.

예제는 다음과 같다.

```c
SEC("tp_btf/sched_process_exec")
int handle_exec(
    struct trace_event_raw_sched_process_exec *ctx
)
```

기존 방식은 다음과 같이 생각할 수 있다.

```text
Tracepoint format 확인
        ↓
직접 struct 작성
        ↓
Context 해석
```

BTF를 사용하면 다음에 가깝다.

```text
Kernel BTF
   ↓
vmlinux.h
   ↓
Kernel Type 그대로 사용
```

이 때문에 Kernel 구조와 eBPF 코드가 서로 맞지 않는 문제를 줄이는 데 도움이 된다.

---

# 12. Uprobe와 Uretprobe

지금까지는 Kernel 내부 Event를 살펴봤지만 eBPF는 User Space Function에도 Attach할 수 있다.

대표적인 것이 다음 두 가지다.

- `uprobe`: User Space Function Entry
- `uretprobe`: User Space Function Return

예를 들어 OpenSSL의 `SSL_write()` Function을 추적할 수 있다.

```c
SEC("uprobe/usr/lib/aarch64-linux-gnu/libssl.so.3/SSL_write")
```

이 방식을 이용하면 Application 자체 코드를 수정하지 않고 User Space Library Function의 호출을 관찰할 수 있다.

다만 User Space Probe는 Kernel Probe보다 환경 영향을 많이 받는다.

예를 들어 다음을 고려해야 한다.

- Library 설치 Path
- CPU Architecture
- Library Version
- Static Linking 여부
- Container 내부 File System
- Application이 작성된 언어의 Calling Convention

특히 Container는 Host와 다른 Root Filesystem을 사용하므로 Host의 Library Path와 Container 안의 Library Path가 다를 수 있다.

---

# 13. LSM Program

`BPF_PROG_TYPE_LSM`은 Linux Security Module의 Hook에 eBPF Program을 Attach한다.

LSM은 Linux Kernel에서 보안 정책을 적용하는 Interface다.

일반적인 Tracing Program은 Event를 관찰하는 목적이 강하지만 LSM Program은 **Kernel의 동작을 실제로 차단할 수 있다.**

예를 들어 LSM BPF Program이 Security Check에서 0이 아닌 값을 Return하면 해당 동작이 허용되지 않을 수 있다.

```text
Process
  ↓
어떤 Kernel Operation 요청
  ↓
LSM Hook
  ↓
BPF LSM Program
  ↓
허용 / 거부
```

이런 특성 때문에 BPF LSM은 eBPF 기반 Runtime Security Tool에서 중요한 역할을 한다.

---

# 14. Networking Program

Tracing 계열 Program이 "무슨 일이 일어났는지 관찰"하는 데 많이 사용된다면, Networking 관련 Program은 **Packet의 처리 방식 자체를 바꾸는 데** 많이 사용된다.

Networking Program의 특징은 크게 두 가지다.

첫 번째는 Return Value가 Packet의 처리를 결정할 수 있다는 것이다.

```text
PASS
DROP
REDIRECT
```

두 번째는 Program에서 Packet이나 Socket 관련 정보를 수정할 수 있다는 것이다.

따라서 eBPF는 단순 Network Monitoring뿐 아니라 다음과 같은 기능을 구현할 수 있다.

```text
Packet Filtering
Load Balancing
Traffic Redirect
Network Policy
Traffic Control
```

---

# 15. Network Stack과 eBPF Hook

Chapter 7의 그림을 보면 eBPF Program이 Network Stack의 한 곳에서만 실행되는 것이 아니라 여러 Layer에 Attach될 수 있다는 점을 확인할 수 있다.

![BPF program types hook into various points in the network stack](./assets/img/learning-ebpf/chapter7-bpf-program-types.png)

대략적인 위치를 단순화하면 다음과 같다.

```text
Application
    ↓
Socket
    ↓        ← sockmap / sockops
TCP / UDP
    ↓
IP
    ↓        ← cgroup 관련 Hook
Ethernet
    ↓
Traffic Control
    ↓        ← TC Hook
Network Device / Driver
    ↓        ← XDP Hook
NIC
```

그림에서 중요한 점은 **XDP와 TC가 같은 eBPF Networking Program이라고 해도 실행되는 위치가 다르다**는 것이다.

XDP는 Network Driver에 가까운 매우 이른 단계에서 실행되고, TC는 그보다 위쪽의 Linux Traffic Control 계층에서 실행된다.

---

# 16. Socket 관련 Program

Network Stack의 위쪽에는 Socket 관련 Program Type이 있다.

## `BPF_PROG_TYPE_SOCKET_FILTER`

초기의 BPF 목적과 가까운 Socket Filtering Program이다.

여기서 "Socket Filtering"은 Application으로 들어오는 Traffic을 Firewall처럼 차단한다는 의미로만 이해하면 안 된다.

책에서 설명하는 대표적인 용도는 tcpdump 같은 Observability Tool로 전달할 Socket Data의 복사본을 Filter하는 것이다.

---

## `BPF_PROG_TYPE_SOCK_OPS`

Socket에서 발생하는 여러 동작을 가로채고 Socket 설정을 조정할 수 있다.

예를 들어 TCP 관련 Parameter를 조정하는 데 사용할 수 있다.

```text
TCP Connection
   ↓
Socket Operation
   ↓
SOCK_OPS eBPF Program
```

Socket은 Connection의 Endpoint에 존재하므로 중간 Router 같은 장비에는 이런 Socket Context가 존재하지 않는다.

---

## `BPF_PROG_TYPE_SK_SKB`

Sockmap과 같이 사용하여 Socket Layer에서 Traffic을 다른 Socket으로 Redirect하는 데 사용할 수 있다.

```text
Socket A
   ↓
SK_SKB Program
   ↓
Sockmap Lookup
   ↓
Socket B로 Redirect
```

---

# 17. Traffic Control, TC

Linux에는 원래 Traffic Control Subsystem이 존재한다.

`tc` 명령으로 Queueing, Filtering, Classification 등을 설정할 수 있다.

eBPF Program도 이 TC Hook에 Attach할 수 있다.

특히 다음 두 방향에 Program을 적용할 수 있다.

```text
Ingress
→ Interface로 들어오는 Traffic

Egress
→ Interface에서 나가는 Traffic
```

TC에서는 Packet을 검사하고 Filter하거나 Redirect하는 등의 동작을 할 수 있다.

Cilium도 Network Datapath를 구성할 때 TC 계열 Hook을 중요하게 사용한다.

Cilium 환경에서 `bpftool net show`를 보면 Pod veth와 Host Device의 ingress/egress 지점에 여러 eBPF Program이 Attach된 것을 볼 수 있다.

---

# 18. XDP

XDP는 **eXpress Data Path**의 약자다.

Chapter 3에서도 사용했던 Program Type이다.

XDP는 Network Packet이 Linux Network Stack 깊숙이 들어가기 전에 Network Driver에 가까운 위치에서 실행된다.

```text
NIC
 ↓
Driver
 ↓
XDP
 ↓
Linux Network Stack
```

그래서 불필요한 Packet을 아주 일찍 Drop할 수 있다.

대표적인 사용 예시는 다음과 같다.

- DDoS Packet Drop
- High-performance Load Balancing
- Packet Redirect
- Early Packet Filtering

XDP Program은 특정 Network Interface에 Attach한다.

```bash
bpftool prog load hello.bpf.o /sys/fs/bpf/hello
bpftool net attach xdp id 540 dev eth0
```

또는 `ip` 명령으로도 Attach할 수 있다.

```bash
ip link set dev eth0 xdp obj hello.bpf.o sec xdp
```

확인은 다음과 같이 할 수 있다.

```bash
ip link show
```

제거는 다음과 같다.

```bash
ip link set dev eth0 xdp off
```

한 가지 중요한 점은 XDP Program이 **Interface 단위로 Attach된다**는 것이다.

따라서 하나의 Host에 여러 Network Interface가 있다면 Interface마다 서로 다른 XDP Program을 Attach하는 것도 가능하다.

---

# 19. Flow Dissector

Flow Dissector는 Packet Header에서 Network Flow를 구분하는 데 필요한 정보를 뽑는 기능이다.

예를 들면 Packet에서 다음과 같은 값을 읽을 수 있다.

```text
Source IP
Destination IP
Source Port
Destination Port
Protocol
```

`BPF_PROG_TYPE_FLOW_DISSECTOR`를 사용하면 이런 Packet 분석 로직을 eBPF로 구현할 수 있다.

---

# 20. Lightweight Tunnel

`BPF_PROG_TYPE_LWT_*` 계열 Program은 eBPF로 Network Encapsulation과 Routing 관련 처리를 구현할 때 사용할 수 있다.

`ip route`와 같이 Linux Routing 기능과 연결된다.

책에서는 실제 사용 빈도가 높은 Program Type으로 소개하지는 않고, 이런 방식도 존재한다는 정도로 설명한다.

---

# 21. Cgroup에 eBPF Attach하기

Cgroup은 Linux에서 Process Group별로 Resource를 관리하고 격리하는 기능이다.

Container와 Kubernetes Pod도 결국 Linux Cgroup을 기반으로 Resource가 분리된다.

eBPF Program을 Cgroup에 Attach하면 **특정 Cgroup에 속한 Process에만 적용되는 동작**을 만들 수 있다.

예를 들면 특정 Cgroup의 Process가 Socket을 만들거나 Network Traffic을 전송할 때 eBPF Program으로 허용 여부를 검사할 수 있다.

```text
Process
  ↓
Cgroup
  ↓
Socket 생성 / Packet 전송
  ↓
Cgroup BPF Hook
  ↓
Allow / Deny
```

대표적인 Program Type은 다음과 같다.

```text
BPF_PROG_TYPE_CGROUP_SOCK
BPF_PROG_TYPE_CGROUP_SKB
```

Kubernetes 환경에서는 이 구조가 특히 중요하다.

Pod마다 서로 다른 Network Policy를 적용해야 하는 경우 Process가 속한 Cgroup을 기준으로 Network 동작을 구분하는 방식이 가능하기 때문이다.

---

# 22. Program Type과 Attachment Type은 왜 따로 있을까?

Program Type만으로 Attach 위치가 완전히 결정되는 경우도 있다.

예를 들어 XDP Program은 XDP Hook에 Attach된다.

하지만 하나의 Program Type이 여러 Hook에서 사용될 수 있는 경우에는 **Attachment Type을 추가로 지정해야 한다.**

책에서는 `BPF_PROG_TYPE_CGROUP_SOCK`를 예로 든다.

이 Program Type은 다음과 같은 위치에 Attach할 수 있다.

```text
BPF_CGROUP_INET_SOCK_CREATE
BPF_CGROUP_INET_SOCK_RELEASE
BPF_CGROUP_INET4_POST_BIND
BPF_CGROUP_INET6_POST_BIND
```

즉 모두 Cgroup Socket Program이지만 실제 실행 시점은 다르다.

```text
Socket 생성 직후
Socket 해제 시점
IPv4 Bind 완료 후
IPv6 Bind 완료 후
```

Kernel은 Program을 Load할 때 Program Type과 Attachment Type 조합이 유효한지도 검사한다.

또 Attachment Type에 따라 사용할 수 있는 Helper Function이나 Context 접근 범위가 달라질 수 있다.

---

# 23. Chapter 7에서 정리해야 할 핵심

이번 Chapter에서 가장 중요한 것은 Program Type 이름을 전부 외우는 것이 아니다.

eBPF Program은 아무 Event에나 같은 방식으로 붙는 것이 아니라 **목적에 맞는 Hook과 Program Type을 선택해야 한다**는 점이 핵심이다.

전체 흐름을 다시 정리하면 다음과 같다.

```text
1. 관찰하거나 제어하고 싶은 Event를 정한다.

예)
Process 실행
Kernel Function 호출
Packet 수신
Socket 생성

        ↓

2. 적절한 Attachment Point를 찾는다.

예)
kprobe
tracepoint
fentry
XDP
TC
cgroup

        ↓

3. Program Type이 결정된다.

        ↓

4. Program Type에 따라 Context가 결정된다.

        ↓

5. 사용할 수 있는 Helper / kfunc 범위가 결정된다.

        ↓

6. Return Value가 Kernel에서 어떤 의미를 가지는지도 결정된다.
```

Tracing에서는 주로 다음 관계를 기억하면 된다.

```text
Kernel Function Entry
→ kprobe / fentry

Kernel Function Return
→ kretprobe / fexit

Kernel에 미리 정의된 Event
→ tracepoint

User Space Function
→ uprobe / uretprobe
```

Networking에서는 다음 정도로 먼저 구분하면 이해하기 쉽다.

```text
Network Driver 근처
→ XDP

Linux Traffic Control
→ TC

Socket Layer
→ SOCK_OPS / SK_SKB

특정 Process Group 기준
→ Cgroup BPF
```

다음 Chapter 8에서는 여기서 본 Networking Program Type들을 실제로 이용해 Packet Drop, Load Balancing, Forwarding, Network Policy 같은 기능을 어떻게 구현하는지 더 자세히 다룬다.
