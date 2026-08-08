---
layout: post
title: "[Learning eBPF] Chapter 3: Anatomy of an eBPF Program"
date: 2026-07-25 19:10:00 +0900
categories: [eBPF, Linux, Kernel, Networking]
tags: [eBPF, XDP, BPF, LLVM, Clang, ELF, bpftool, JIT, BTF]
published: true
---

# Learning eBPF Chapter 3

## Anatomy of an eBPF Program

Chapter 2에서는 BCC를 이용해 Python 코드 안에 eBPF C 코드를 작성하고, `BPF()` 객체를 생성해 프로그램을 실행했다. 당시에는 특정 System Call이나 Kernel Event에 eBPF 프로그램을 연결하고 결과를 확인하는 데 집중했기 때문에, 작성한 C 코드가 실제로 어떤 형태로 변환되고 커널 안에서 어떻게 실행되는지는 거의 보이지 않았다.

Chapter 3에서는 그 뒤에 감춰져 있던 과정을 하나씩 직접 확인한다. 제한된 C로 eBPF 프로그램을 작성하고, Clang과 LLVM을 이용해 eBPF Bytecode가 포함된 ELF Object File을 만든다. 이후 `bpftool`로 프로그램을 Kernel에 Load하고, XDP Hook에 Attach한 뒤 실제 Packet이 들어왔을 때 실행되는 흐름을 확인한다. 마지막에는 Global Variable이 BPF Map으로 구현되는 방식과 Program의 Detach, Unload, BPF-to-BPF Function Call까지 살펴본다.

```text
eBPF C Source
      ↓
Clang / LLVM
      ↓
eBPF Bytecode
      ↓
ELF Object File
      ↓
Kernel에 Load
      ↓
Verifier 검증
      ↓
JIT Compilation
      ↓
Hook에 Attach
      ↓
Event 발생 시 실행
```

## eBPF Virtual Machine이란 무엇인가

이번 챕터에서 가장 먼저 이해해야 했던 개념은 eBPF Virtual Machine이었다. 처음에는 Virtual Machine이라는 이름 때문에 VMware나 OpenStack Instance처럼 운영체제 전체를 실행하는 가상 머신을 떠올렸는데, 여기서 말하는 VM은 그와는 성격이 다르다.

eBPF VM은 운영체제를 실행하는 환경이 아니라, **eBPF Program이 따라야 하는 Instruction Set, Register, Stack, Calling Convention을 정의한 가상의 CPU 실행 모델**이다. 개발자가 작성한 C 코드는 실제 CPU가 바로 실행하지 않는다. 먼저 eBPF VM이 이해할 수 있는 eBPF Bytecode로 변환되고, Kernel에서 Verifier 검증을 거친 뒤 JIT Compiler에 의해 현재 CPU Architecture에 맞는 Native Machine Code로 다시 변환된다.

```text
C Source Code
      ↓
eBPF Bytecode
      ↓
Verifier
      ↓
JIT Compiler
      ↓
x86-64 또는 ARM64 Machine Code
      ↓
CPU 실행
```

JVM이 Java Bytecode를 실행하고 WebAssembly Runtime이 WASM Bytecode를 실행하는 것과 비슷하게, eBPF 역시 CPU와 독립적인 Bytecode를 기준으로 동작한다. 다만 eBPF는 Linux Kernel 내부에서 안전하게 실행되어야 하므로 일반적인 VM보다 훨씬 강한 검증과 제한을 받는다.

초기에는 Kernel의 Interpreter가 eBPF Instruction을 한 줄씩 해석하는 방식이 사용되었다. 하지만 Network Packet처럼 매우 자주 발생하는 Event마다 Instruction을 반복해서 해석하면 비용이 커질 수 있다. 그래서 현재는 Program을 Load할 때 Bytecode를 한 번 Native Machine Code로 변환하고, 이후에는 변환된 Code를 직접 실행하는 JIT 방식이 일반적으로 사용된다.

같은 eBPF Bytecode라도 실행되는 시스템에 따라 결과는 달라진다.

```text
동일한 eBPF Bytecode
        ↓
x86-64 시스템 → x86-64 Machine Code
ARM64 시스템  → ARM64 Machine Code
```

이 구조 덕분에 개발자는 특정 CPU에 종속된 Assembly를 직접 작성하지 않고도, 하나의 eBPF Source를 여러 Architecture에서 사용할 수 있다.

## eBPF VM의 Register와 Calling Convention

eBPF VM에는 R0부터 R10까지 총 11개의 64-bit Register가 있다. CPU처럼 모든 Register를 자유롭게 사용하는 것은 아니며, 각각 정해진 역할이 있다.

| Register | 역할 |
|---|---|
| R0 | Function 또는 eBPF Program의 반환값 |
| R1~R5 | Function과 BPF Helper 호출 인자 |
| R6~R9 | Function Call 이후에도 값이 유지되는 Callee-Saved Register |
| R10 | Stack Frame Pointer |

Program이 시작될 때 R1에는 해당 Hook에서 제공하는 Context Pointer가 전달된다. XDP Program이라면 일반적으로 `struct xdp_md *ctx`가 R1에 들어가고, Program이 종료될 때는 R0에 `XDP_PASS`, `XDP_DROP` 같은 최종 Verdict가 저장된다.

Helper Function을 호출할 때도 같은 규칙이 적용된다.

```text
R1 = 첫 번째 인자
R2 = 두 번째 인자
R3 = 세 번째 인자
R4 = 네 번째 인자
R5 = 다섯 번째 인자
R0 = 반환값
```

이번 예제의 `bpf_printk()` 호출도 Bytecode에서는 이 Calling Convention에 맞게 인자가 배치된다.

```text
R1 = Format String 주소
R2 = Format String 길이
R3 = counter 값
CALL bpf_trace_printk
```

R10은 Stack Frame Pointer다. eBPF Stack은 R10을 기준으로 음수 Offset 방향으로 사용한다.

```text
R10          Stack 기준점
R10 - 8      Local Variable
R10 - 16     임시 데이터
R10 - 24     구조체 일부
```

R10은 프로그램이 임의로 변경할 수 없다. Verifier가 Stack 범위를 판단할 때 R10이 고정된 기준점이 되어야 하기 때문이다. 만약 Program이 R10 값을 자유롭게 바꿀 수 있다면 `R10 - 8`이 실제 Stack인지 다른 Kernel Memory인지 구분하기 어려워진다.

이 부분을 읽으면서 한 가지 궁금증이 생겼다. Function이나 Helper를 호출할 때 인자는 R1부터 R5까지 전달한다고 하는데, **인자가 5개보다 많으면 어떻게 되는지**가 궁금했다.

### 인자가 5개를 넘으면 어떻게 될까

일반적인 x86-64 C Calling Convention에서는 Register에 담을 수 있는 수보다 인자가 많으면 나머지 인자를 Stack으로 넘길 수 있다. 그래서 처음에는 eBPF도 R1부터 R5까지는 Register에 넣고, 여섯 번째 인자부터는 Stack에 저장할 것이라고 생각했다.

하지만 eBPF의 기본 Calling Convention은 그렇게 동작하지 않는다. eBPF에서는 Function Call Argument로 사용할 수 있는 Register가 R1부터 R5까지로 고정되어 있으며, 전통적인 eBPF Function과 BPF Helper 호출은 **최대 다섯 개의 인자만 직접 받을 수 있다**.

```text
R1 = argument 1
R2 = argument 2
R3 = argument 3
R4 = argument 4
R5 = argument 5

R6 = 여섯 번째 인자 용도가 아님
R7 = 일곱 번째 인자 용도가 아님
```

R6부터 R9는 추가 인자를 전달하는 Register가 아니라, Function Call 이후에도 값이 보존되어야 하는 Callee-Saved Register다. 따라서 여섯 번째 인자가 있다고 해서 자동으로 R6에 들어가는 것은 아니다. R10 역시 Stack Argument를 전달하는 용도가 아니라 현재 Stack Frame을 가리키는 Read-only Frame Pointer다.

예를 들어 다음처럼 인자가 여섯 개인 Function을 그대로 만들려고 하면 eBPF Calling Convention과 맞지 않는다.

```c
static __noinline int calculate(
    int a,
    int b,
    int c,
    int d,
    int e,
    int f)
{
    return a + b + c + d + e + f;
}
```

일반적인 eBPF 환경에서는 이 여섯 값을 각각 R1부터 R6에 배치하는 방식으로 호출할 수 없다. R6은 Argument Register가 아니기 때문이다. 이 경우에는 여러 값을 하나의 구조체로 묶고, 그 구조체를 가리키는 Pointer 하나를 Argument로 전달하는 방식이 가장 일반적이다.

```c
struct calculate_args {
    int a;
    int b;
    int c;
    int d;
    int e;
    int f;
};

static __noinline int calculate(struct calculate_args *args)
{
    return args->a + args->b + args->c
         + args->d + args->e + args->f;
}
```

호출하는 쪽에서는 구조체를 Stack에 만들고 Pointer 하나만 R1로 전달할 수 있다.

```c
struct calculate_args args = {
    .a = 1,
    .b = 2,
    .c = 3,
    .d = 4,
    .e = 5,
    .f = 6,
};

int result = calculate(&args);
```

Register 관점에서 보면 다음과 같다.

```text
R1 = &args
      ↓
calculate()가 Pointer를 통해
a, b, c, d, e, f를 읽음
```

즉, Argument 개수는 하나지만 그 Pointer가 가리키는 Memory 안에 여러 값을 함께 담는 것이다. 이 방법은 BPF Helper나 BPF-to-BPF Function에서 여러 정보를 한 번에 넘겨야 할 때 자주 사용된다.

다만 구조체를 Stack에 배치할 경우 eBPF Stack 크기 제한도 고려해야 한다. 너무 큰 구조체를 Local Variable로 만들면 Verifier가 Stack 사용량 초과로 Program Load를 거부할 수 있다. 데이터가 크거나 여러 Function에서 공유해야 한다면 BPF Map, Per-CPU Map, Global Data 영역 등을 이용해 값을 저장하고 Pointer 또는 Key만 전달하는 방식도 사용할 수 있다.

```text
작은 데이터 묶음
→ Stack에 struct 생성 후 Pointer 전달

큰 데이터 또는 실행 간 공유 상태
→ BPF Map에 저장 후 Map Value Pointer 사용

고정된 설정값
→ Global Data 또는 .rodata 사용
```

또 다른 방법은 Function을 여러 단계로 나누는 것이다.

```c
static __noinline int step_one(int a, int b, int c)
{
    return a + b + c;
}

static __noinline int step_two(int partial, int d, int e, int f)
{
    return partial + d + e + f;
}
```

하지만 단순히 인자 수 제한을 피하려고 Function Call을 지나치게 늘리면 Call Depth와 Stack Frame 사용량이 증가할 수 있다. 따라서 관련 값들을 의미 있는 구조체로 묶어 Pointer로 전달하는 방식이 보통 더 자연스럽다.

BPF Helper도 같은 Calling Convention을 따르기 때문에 Prototype 자체가 최대 다섯 개의 인자로 정의된다.

```c
long bpf_helper(
    void *arg1,
    __u64 arg2,
    __u64 arg3,
    __u64 arg4,
    __u64 arg5);
```

여러 값을 요구하는 Helper는 개별 값을 계속 추가하기보다, 구조체 Pointer나 Context Pointer를 하나의 인자로 받도록 Interface가 설계되는 경우가 많다.

정리하면, 전통적인 eBPF Calling Convention에서는 여섯 번째 인자를 자동으로 Stack에 넘기는 규칙이 없다. 다섯 개가 넘는 값을 전달해야 한다면 보통 구조체로 묶어서 Pointer 하나를 넘기거나, Map이나 Global Data에 저장한 뒤 필요한 Reference만 전달한다.

```text
인자 5개 이하
→ R1~R5에 직접 전달

전달할 값이 5개 초과
→ struct로 묶고 Pointer 전달
→ Map 또는 Global Data 사용
→ 필요한 경우 Function을 의미 있게 분리
```

이 부분을 확인하면서 eBPF에서 Pointer와 구조체를 자주 사용하는 이유도 조금 더 이해할 수 있었다. 단순히 C 문법상의 선택이 아니라, 다섯 개의 Argument Register만 제공하는 eBPF Calling Convention과 제한된 Stack 안에서 여러 데이터를 안전하게 전달하기 위한 방식이었다.

이 부분을 읽으면서 eBPF Program이 단순히 “커널에서 실행되는 C 코드”가 아니라, 명확한 Register 규칙과 Calling Convention을 가진 별도의 실행 모델 위에서 동작한다는 점이 확실해졌다.

## Verifier는 Program의 안전성을 어떻게 확인하는가

eBPF Program은 Kernel 내부에서 실행되기 때문에 잘못된 Memory Access나 무한 Loop가 Kernel 전체 장애로 이어질 수 있다. 이를 막기 위해 Program은 Load 단계에서 반드시 Verifier 검사를 통과해야 한다.

Verifier는 단순히 문법이나 Opcode만 확인하는 것이 아니다. 모든 가능한 실행 경로를 따라가면서 Register가 어떤 값을 가지고 있는지, Pointer가 어느 영역을 가리키는지, Stack Access가 유효한지, Helper Function의 인자가 올바른지 등을 추적한다.

예를 들어 Verifier는 Register를 다음과 같은 성격으로 구분해서 본다.

```text
SCALAR_VALUE
PTR_TO_CTX
PTR_TO_STACK
PTR_TO_MAP_VALUE
PTR_TO_PACKET
```

같은 64-bit 값이라도 일반 정수와 Pointer는 다르게 취급된다. 일반 정수에는 산술 연산을 적용할 수 있지만 Pointer에는 허용된 범위 안에서만 Offset을 더하거나 Memory를 읽을 수 있다.

XDP Program에서 Packet Header를 읽을 때 항상 `data_end`와 경계 검사를 해야 하는 이유도 여기에 있다.

```c
void *data = (void *)(long)ctx->data;
void *data_end = (void *)(long)ctx->data_end;

struct ethhdr *eth = data;

if ((void *)(eth + 1) > data_end)
    return XDP_DROP;
```

이 조건을 통과한 실행 경로에서만 Verifier가 Ethernet Header 전체가 Packet 범위 안에 있다고 판단할 수 있다. 따라서 eBPF Program의 조건문은 단순히 로직 분기를 위한 것이 아니라, Verifier에게 해당 Memory Access가 안전하다는 사실을 증명하는 역할도 한다.

이번 Hello World 예제는 Packet Data를 직접 읽지 않기 때문에 이런 경계 검사가 나타나지는 않지만, 이후 실제 Network Program을 작성할 때 매우 중요하게 반복되는 패턴이다.

## eBPF Instruction과 Bytecode 구조

개발자가 작성한 C 코드는 최종적으로 eBPF Instruction의 집합으로 변환된다. Linux Kernel의 `include/uapi/linux/bpf.h`에는 Instruction 하나를 표현하는 `struct bpf_insn`이 정의되어 있다.

```c
struct bpf_insn {
    __u8 code;
    __u8 dst_reg:4;
    __u8 src_reg:4;
    __s16 off;
    __s32 imm;
};
```

기본 eBPF Instruction 하나는 8바이트다.

| 필드 | 의미 |
|---|---|
| `code` | 수행할 연산을 나타내는 Opcode |
| `dst_reg` | 결과를 저장할 Destination Register |
| `src_reg` | 입력값을 가진 Source Register |
| `off` | Memory Access나 Jump에 사용하는 Offset |
| `imm` | Instruction에 직접 포함되는 Immediate 값 |

대표적인 Instruction은 Register 이동, 산술 연산, Memory Load와 Store, 조건부 Jump, Function Call, Exit 등으로 구분할 수 있다.

기본 Instruction의 `imm` 영역은 32-bit이므로 64-bit 값을 한 번에 담을 수 없다. 이 경우에는 8바이트 Slot 두 개를 사용하는 Wide Instruction을 사용한다. 이후 `llvm-objdump` 결과에서 Instruction Offset이 `0, 2, 3, 5`처럼 보이는 이유도 Offset 0과 3의 Instruction이 각각 두 Slot을 차지하기 때문이다.

## Packet이 들어올 때 실행되는 XDP Hello World

Chapter 2의 Hello World는 System Call에 연결된 kprobe 또는 Tracepoint에서 실행되었다. 이번 Chapter 3의 Hello World는 Network Interface에 Packet이 들어올 때마다 실행되는 XDP Program이다.

XDP는 eXpress Data Path의 약자로, Packet이 Linux Network Stack 깊숙이 들어가기 전에 실행되는 Hook이다.

```text
NIC
 ↓
Device Driver
 ↓
XDP Hook
 ↓
skb 생성
 ↓
TC / Netfilter
 ↓
IP, TCP, UDP 처리
 ↓
Socket
 ↓
Application
```

XDP는 `skb` 생성 이전의 비교적 이른 지점에서 실행될 수 있기 때문에 Packet을 빠르게 검사하거나 Drop할 수 있다. DDoS 공격 Packet을 XDP 단계에서 제거하면 이후 Routing, Netfilter, Socket Lookup 같은 작업을 수행할 필요가 없다.

XDP Program은 실행 후 Packet을 어떻게 처리할지 Verdict를 반환한다.

| Verdict | 의미 |
|---|---|
| `XDP_ABORTED` | 비정상 종료 |
| `XDP_DROP` | Packet 폐기 |
| `XDP_PASS` | 일반 Linux Network Stack으로 전달 |
| `XDP_TX` | 수신 Interface로 다시 송신 |
| `XDP_REDIRECT` | 다른 Interface나 대상에 전달 |

이번 예제의 목적은 실제 Packet 분석이 아니라, Packet이 들어올 때 Program이 실행되는지 확인하는 것이다.

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

int counter = 0;

SEC("xdp")
int hello(void *ctx)
{
    bpf_printk("Hello World %d", counter);
    counter++;
    return XDP_PASS;
}

char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

### `SEC("xdp")`

```c
SEC("xdp")
int hello(void *ctx)
```

`SEC("xdp")`는 `hello()` 함수를 ELF Object File의 `xdp` Section에 배치한다. Loader는 이 Section Name을 보고 해당 Function이 XDP Program이라는 것을 알 수 있다.

처음에는 `SEC("xdp")`를 단순한 Annotation 정도로 생각했는데, 실제로는 Program Type을 구분하고 Object File 내부의 Code 위치를 정하는 중요한 Metadata였다.

`hello`는 Function Name이며, Program을 Kernel에 Load한 뒤 `bpftool prog list`에서도 Program Name으로 확인할 수 있다.

### `ctx`

`ctx`는 XDP Event가 발생했을 때 Kernel이 전달하는 Context다. 일반적으로 XDP Program에서는 `struct xdp_md *ctx`를 사용하며, 여기에서 Packet의 시작 주소와 끝 주소를 가져온다.

이번 예제에서는 Packet 내용을 읽지 않기 때문에 `ctx`를 실제로 사용하지 않는다.

### `bpf_printk()`

```c
bpf_printk("Hello World %d", counter);
```

eBPF Program은 User Space의 libc를 사용할 수 없기 때문에 일반적인 `printf()`를 호출할 수 없다. `bpf_printk()`는 Kernel Trace Buffer에 문자열을 기록하기 위한 디버깅용 Interface다.

출력은 다음 경로 또는 명령으로 확인할 수 있다.

```bash
cat /sys/kernel/debug/tracing/trace_pipe
```

```bash
bpftool prog tracelog
```

다만 `bpf_printk()`는 대량의 Event 데이터를 전달하기 위한 방식이 아니다. Packet마다 호출하면 출력 자체가 성능 저하를 만들 수 있다. 실제 Observability Program에서는 Ring Buffer, Perf Buffer, BPF Map 등을 이용해 User Space로 데이터를 전달하는 방식이 더 적합하다.

### `counter`

```c
int counter = 0;
```

처음에는 일반 C 전역 변수처럼 보였지만, eBPF Program 실행 간 값을 유지하기 위해 실제로는 `.bss` 영역을 나타내는 BPF Map으로 구현된다.

Packet이 들어올 때마다 다음 흐름이 반복된다.

```text
counter 읽기
      ↓
현재 값을 출력
      ↓
1 증가
      ↓
다시 저장
```

### `XDP_PASS`

```c
return XDP_PASS;
```

`XDP_PASS`는 Packet을 일반 Linux Network Stack으로 계속 전달하라는 의미다. 값은 2이며, eBPF Calling Convention에 따라 R0에 저장된다.

Bytecode에서는 다음처럼 보인다.

```text
r0 = 2
exit
```

### License

```c
char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

License Section은 단순한 문서용 정보가 아니다. 일부 BPF Helper는 GPL-compatible Program에서만 사용할 수 있기 때문에 Kernel은 Program의 License를 확인한다.

## Clang과 LLVM으로 eBPF Object File 만들기

eBPF Source Code는 Clang과 LLVM을 이용해 eBPF Bytecode가 포함된 ELF Object File로 컴파일한다.

```bash
clang \
  -target bpf \
  -I/usr/include/$(uname -m)-linux-gnu \
  -g \
  -O2 \
  -c hello.bpf.c \
  -o hello.bpf.o
```

가장 중요한 옵션은 `-target bpf`다. 이 옵션이 있어야 Host CPU용 Code가 아니라 eBPF VM이 이해할 수 있는 Bytecode가 생성된다.

| 옵션 | 의미 |
|---|---|
| `-target bpf` | eBPF ISA를 대상으로 컴파일 |
| `-I...` | Architecture별 Linux Header 검색 경로 추가 |
| `-g` | Debug Information과 BTF 관련 정보 생성 |
| `-O2` | 불필요한 Instruction과 Memory Access 최적화 |
| `-c` | Linking하지 않고 Object File 생성 |
| `-o` | 출력 파일 지정 |

전체 과정은 다음과 같다.

```text
hello.bpf.c
    ↓ Clang Frontend
LLVM IR
    ↓ LLVM BPF Backend
eBPF Bytecode
    ↓ ELF Packaging
hello.bpf.o
```

`file` 명령을 사용하면 생성된 Object File의 형식을 확인할 수 있다.

```bash
file hello.bpf.o
```

```text
hello.bpf.o: ELF 64-bit LSB relocatable, eBPF,
version 1 (SYSV), with debug_info, not stripped
```

여기서 `relocatable`이라는 표현이 중요했다. 아직 Kernel에 생성될 BPF Map ID나 실제 Memory Reference가 정해지지 않았기 때문에, 일부 값은 Load 시점에 다시 연결되어야 한다.

또한 ELF Object File에는 Bytecode만 들어 있는 것이 아니다.

```text
hello.bpf.o
├─ xdp
│   └─ hello Program Bytecode
├─ .bss
│   └─ counter
├─ .rodata
│   └─ "Hello World %d"
├─ license
│   └─ "Dual BSD/GPL"
├─ .BTF
└─ .BTF.ext
```

처음에는 Object File을 단순히 컴파일 결과물 정도로 생각했는데, 실제로는 Program Code와 Global Data, License, BTF, Relocation 정보까지 함께 담고 있는 Container에 가까웠다.

## Object File의 Bytecode 확인하기

다음 명령으로 Object File의 Bytecode를 확인할 수 있다.

```bash
llvm-objdump -S hello.bpf.o
```

핵심 부분은 다음과 같다.

```text
0:  r6 = 0 ll
2:  r3 = *(u32 *)(r6 + 0)
3:  r1 = 0 ll
5:  r2 = 15
6:  call 6

7:  r1 = *(u32 *)(r6 + 0)
8:  r1 += 1
9:  *(u32 *)(r6 + 0) = r1

10: r0 = 2
11: exit
```

Offset 0과 3의 `ll` Instruction은 64-bit 값을 다루는 Wide Instruction이기 때문에 각각 두 개의 8-byte Slot을 사용한다. 그래서 다음 Instruction이 1이 아니라 2, 4가 아니라 5에서 시작한다.

Object File 단계에서는 다음 값이 아직 0으로 표시된다.

```text
r6 = 0 ll
r1 = 0 ll
```

이 값은 실제로 0을 사용하려는 것이 아니라 Loader가 나중에 실제 BPF Map Reference로 치환할 자리다.

`counter++`는 C 코드에서는 한 줄이지만 Bytecode에서는 다음 세 단계로 나뉜다.

```text
r1 = *(u32 *)(r6 + 0)   counter Load
r1 += 1                 1 증가
*(u32 *)(r6 + 0) = r1   증가한 값을 Store
```

이 부분을 보면서 C 코드 한 줄이 여러 개의 Load, Arithmetic, Store Instruction으로 변환된다는 점을 직접 확인할 수 있었다.

## `bpftool`로 Program을 Kernel에 Load하기

Object File을 만든 뒤에는 `bpftool`을 이용해 Program을 Kernel에 Load한다.

```bash
bpftool prog load hello.bpf.o /sys/fs/bpf/hello
```

처음에는 이 명령이 Object File을 Kernel Memory로 단순히 복사하는 정도라고 생각했는데, 실제로는 여러 과정이 함께 수행된다.

```text
ELF Object File 분석
        ↓
.bss와 .rodata Map 생성
        ↓
Relocation 처리
        ↓
bpf(BPF_PROG_LOAD) 호출
        ↓
Verifier 검증
        ↓
JIT Compilation
        ↓
Kernel Program Object 생성
        ↓
bpffs에 Pin
```

Object File에서 0으로 표시되던 Placeholder도 이 과정에서 실제 Map Reference로 변경된다.

```text
Load 전
r6 = 0 ll

Load 후
r6 = map[id:165][0]+0
```

`BPF_PROG_LOAD`가 성공하면 Kernel은 Program Object를 만들고 User Space에 File Descriptor를 반환한다. 다만 `bpftool` Process가 종료되면 해당 FD도 닫히기 때문에, Program을 계속 유지하려면 다른 Reference가 필요하다.

```text
Program FD
      ↓
bpffs에 Pin
      ↓
/sys/fs/bpf/hello
```

`/sys/fs/bpf/hello`는 Object File의 복사본이 아니라 Kernel Program Object를 가리키는 Reference다. 그래서 Program을 Load한 Process가 종료되어도 Pin이 남아 있으면 Program은 Kernel에 계속 존재한다.

## Load된 Program 정보 확인하기

```bash
bpftool prog list
```

예시 출력은 다음과 같다.

```text
540: xdp name hello tag d35b94b4c0c10efb gpl
    loaded_at 2022-08-02T17:39:47+0000 uid 0
    xlated 96B jited 148B memlock 4096B map_ids 165,166
    btf_id 254
```

| 항목 | 의미 |
|---|---|
| `540` | Kernel이 할당한 Program ID |
| `xdp` | Program Type |
| `name hello` | Source Function Name |
| `tag` | Program Instruction 기반 식별값 |
| `xlated` | Verifier 처리 이후 eBPF Bytecode 크기 |
| `jited` | JIT가 생성한 Native Machine Code 크기 |
| `map_ids` | Program이 사용하는 BPF Map |
| `btf_id` | Program과 연결된 BTF 정보 |

Program ID는 Load할 때마다 달라질 수 있지만, Tag는 동일한 Bytecode라면 같은 값을 가질 수 있다. Name은 여러 Program에서 중복될 수 있고, Pinned Path는 bpffs에서 특정 Kernel Object를 다시 찾을 수 있는 경로다.

```bash
bpftool prog show id 540
bpftool prog show name hello
bpftool prog show tag d35b94b4c0c10efb
bpftool prog show pinned /sys/fs/bpf/hello
```

## 같은 Program을 세 가지 형태로 확인하기

이번 챕터에서 가장 인상적이었던 부분은 같은 Program을 서로 다른 세 수준에서 확인할 수 있다는 점이었다.

### C Source Code

```c
counter++;
return XDP_PASS;
```

개발자가 작성하고 읽는 고수준 코드다.

### Translated eBPF Bytecode

```bash
bpftool prog dump xlated name hello
```

```text
0:  r6 = map[id:165][0]+0
2:  r3 = *(u32 *)(r6 + 0)
3:  r1 = map[id:166][0]+0
5:  r2 = 15
6:  call bpf_trace_printk

7:  r1 = *(u32 *)(r6 + 0)
8:  r1 += 1
9:  *(u32 *)(r6 + 0) = r1

10: r0 = 2
11: exit
```

Object File 단계에서는 0이었던 값이 실제 Map Reference로 바뀌고, Helper Call도 실제 Helper Name으로 확인된다.

### JIT-Compiled Machine Code

```bash
bpftool prog dump jited name hello
```

JIT Dump는 실제 CPU Architecture의 Assembly를 보여준다. ARM64라면 `ldr`, `add`, `str`, `blr`, `ret` 같은 Instruction이 나타나고, x86-64라면 `mov`, `add`, `call`, `ret` 같은 Instruction이 나타난다.

```text
C Source
      ↓
eBPF Bytecode
      ↓
Native Machine Code
```

처음에는 eBPF Program이 C 코드 그대로 실행된다고 막연하게 생각했는데, 실제로는 Verifier가 분석하는 Bytecode와 CPU가 실행하는 Machine Code가 따로 존재한다는 점을 이번 챕터에서 명확하게 이해할 수 있었다.

## Load와 Attach는 다른 과정이다

Program을 Kernel에 Load했다고 해서 바로 실행되는 것은 아니다. 이번 챕터에서 가장 확실하게 구분하게 된 개념 중 하나가 Load와 Attach였다.

```text
Load
→ Program을 Kernel에 생성

Attach
→ Program을 Event Hook에 연결

Event 발생
→ Program 실행
```

XDP Program을 `eth0`에 연결하려면 다음 명령을 사용한다.

```bash
bpftool net attach xdp id 540 dev eth0
```

연결 상태는 다음과 같이 확인할 수 있다.

```bash
bpftool net list
```

```text
xdp:
eth0(2) driver id 540
```

Attach 이후에는 `eth0`로 Packet이 들어올 때마다 Program이 실행된다.

```text
eth0 Packet 수신
        ↓
XDP Hook
        ↓
hello Program
        ↓
Trace 출력
        ↓
counter 증가
        ↓
XDP_PASS
        ↓
일반 Linux Network Stack
```

## XDP Trace에서 `<idle>-0`이 나타나는 이유

```bash
cat /sys/kernel/debug/tracing/trace_pipe
```

예시 출력은 다음과 같다.

```text
<idle>-0 [003] d.s.. 655370.944105:
bpf_trace_printk: Hello World 4531
```

Chapter 2의 System Call 예제에서는 특정 User Process가 System Call을 호출했기 때문에 Process Name과 PID가 표시되었다.

```text
User Process
    ↓
System Call
    ↓
eBPF Program
```

하지만 XDP Program은 Packet이 NIC에 들어온 직후 Driver 또는 SoftIRQ Context에서 실행된다. 아직 Routing, TCP/UDP 처리, Socket Lookup이 끝나지 않았기 때문에 해당 Packet을 어떤 User Process가 받을지 정해지지 않은 상태다.

```text
Packet
  ↓
NIC
  ↓
Driver
  ↓
XDP Program
  ↓
이후에 Socket과 Process가 결정됨
```

그래서 특정 Process 대신 `<idle>-0`과 같은 Context가 나타날 수 있다.

## Global Variable은 BPF Map으로 구현된다

Source Code에서는 `counter`를 일반 C Global Variable처럼 선언했다.

```c
int counter = 0;
```

하지만 eBPF Program은 Event마다 다시 실행되므로 실행 간 상태를 유지하기 위한 Kernel 저장 공간이 필요하다. libbpf는 Global Data Section을 Internal BPF Map으로 구현한다.

```text
C Global Variable
        ↓
ELF Data Section
        ↓
Internal BPF Map
```

이번 예제에서는 두 개의 Map이 생성된다.

```text
hello.bss
→ counter

hello.rodata
→ "Hello World %d"
```

### `.bss`

`.bss`는 0으로 초기화되는 수정 가능한 Global Data를 저장한다.

```text
165: array name hello.bss
key 4B value 4B max_entries 1
```

```bash
bpftool map dump name hello.bss
```

BTF 정보가 있으면 다음처럼 Variable Name과 값을 확인할 수 있다.

```json
[
  {
    "value": {
      ".bss": [
        {
          "counter": 11127
        }
      ]
    }
  }
]
```

### `.rodata`

`.rodata`는 수정할 수 없는 Read-only Data를 저장한다.

```text
166: array name hello.rodata
key 4B value 15B max_entries 1 frozen
```

이번 예제에서는 `"Hello World %d"` 문자열이 들어 있다.

```bash
bpftool map dump name hello.rodata
```

```json
[
  {
    "value": {
      ".rodata": [
        {
          "hello.____fmt": "Hello World %d"
        }
      ]
    }
  }
]
```

`-g` 옵션으로 생성된 BTF 정보가 있으면 `bpftool`이 Raw Byte가 아니라 Variable Name과 Type을 기준으로 값을 보여줄 수 있다.

전역 변수는 초기화 방식에 따라 다른 Section에 들어갈 수 있다.

| Section | 의미 |
|---|---|
| `.bss` | 0으로 초기화되는 수정 가능한 Global Data |
| `.data` | 0이 아닌 값으로 초기화되는 수정 가능한 Global Data |
| `.rodata` | 수정할 수 없는 Read-only Data |

```c
int counter = 0;          // .bss
int threshold = 100;      // .data
const int limit = 1024;   // .rodata
```

## Detach와 Unload도 다른 과정이다

XDP Hook에서 Program을 제거하려면 다음 명령을 사용한다.

```bash
bpftool net detach xdp dev eth0
```

Detach는 Program과 Hook의 연결만 제거한다. Program은 여전히 `/sys/fs/bpf/hello`에 Pin되어 있기 때문에 Kernel에 남아 있다.

```text
Detach
→ Hook 연결 제거
→ Program Object는 유지
```

Pinned Path를 제거하면 bpffs Reference가 사라진다.

```bash
rm /sys/fs/bpf/hello
```

다른 File Descriptor, BPF Link, Attach Reference가 없다면 Program의 Reference Count가 0이 되고 Kernel에서 제거된다.

```text
Detach
      ↓
Hook Reference 제거

Pin 삭제
      ↓
bpffs Reference 제거

모든 Reference 소멸
      ↓
Program Unload
```

처음에는 Detach와 Unload를 같은 의미로 생각했는데, 실제로는 Event 연결을 끊는 것과 Kernel Object 자체를 제거하는 것은 별개의 과정이었다.

## BPF-to-BPF Function Call

Chapter 2에서는 Tail Call을 통해 다른 eBPF Program으로 실행을 넘기는 방식을 다뤘다. Chapter 3에서는 하나의 eBPF Program 내부에서 일반 Function을 호출하는 BPF-to-BPF Call을 확인한다.

```c
static __attribute((noinline))
int get_opcode(struct bpf_raw_tracepoint_args *ctx)
{
    return ctx->args[1];
}
```

```c
SEC("raw_tp")
int hello(struct bpf_raw_tracepoint_args *ctx)
{
    int opcode = get_opcode(ctx);
    bpf_printk("Syscall: %d", opcode);
    return 0;
}
```

`get_opcode()`는 매우 짧은 Function이라 Compiler가 호출을 없애고 Function Body를 호출 위치에 직접 삽입할 수 있다. 이를 Inlining이라고 한다. 이번 예제에서는 실제 Call Instruction을 확인하기 위해 `__attribute((noinline))`을 사용한다.

Bytecode에서는 다음처럼 나타난다.

```text
0: call pc+7

1: r1 = map[id:...][0]+0
3: r2 = 12
4: r3 = r0
5: call bpf_trace_printk
6: r0 = 0
7: exit

8: r0 = *(u64 *)(r1 + 8)
9: exit
```

Offset 0에서 `get_opcode()`를 호출하고, Offset 8에서 `ctx->args[1]` 값을 R0에 넣어 반환한다. 이후 Caller로 돌아와 R0 값을 `bpf_printk()`의 인자로 사용한다.

BPF-to-BPF Call과 Tail Call의 차이는 다음과 같다.

| 구분 | BPF-to-BPF Call | Tail Call |
|---|---|---|
| 호출 대상 | 같은 Program 내부 Function | 다른 eBPF Program |
| 호출 후 복귀 | Caller로 복귀 | 성공하면 기존 Program으로 복귀하지 않음 |
| 대상 결정 | Compile된 Relative Offset | Program Array Map |
| 주요 목적 | Function 분리와 코드 재사용 | Program Chaining과 Dynamic Dispatch |

일반 Function Call은 Caller로 돌아와야 하므로 Return Address와 Register, Stack Frame 관리가 필요하다. eBPF는 Stack Size와 Call Depth에 제한이 있기 때문에 일반 User Space Program처럼 깊은 Function Call 구조를 자유롭게 만들 수 없다.

## Chapter 3 정리

이번 Chapter 3에서는 eBPF Program이 단순히 C 코드를 Kernel Hook에 연결해서 실행하는 구조가 아니라는 점을 이해할 수 있었다.

```text
C Source
    ↓
Clang / LLVM
    ↓
eBPF Bytecode
    ↓
ELF Object File
    ↓
Loader와 Relocation
    ↓
bpf() System Call
    ↓
Verifier
    ↓
JIT Machine Code
    ↓
Kernel Program Object
    ↓
Hook Attach
    ↓
Event 발생 시 실행
```

eBPF VM은 Register, Stack, Instruction Set, Calling Convention을 정의하고, Compiler는 C 코드를 이 실행 모델에 맞는 Bytecode로 변환한다. Verifier는 Bytecode의 모든 실행 경로와 Register, Pointer, Stack Access를 검사하고, 검증된 Program은 JIT를 통해 실제 CPU용 Machine Code가 된다.

또한 이번 챕터를 통해 Load와 Attach, Detach와 Unload가 각각 다른 과정이라는 점도 분명하게 구분할 수 있었다. Global Variable 역시 일반 Memory에 존재하는 것이 아니라 `.bss`와 `.rodata`를 나타내는 Internal BPF Map으로 구현된다.

결국 이번 챕터의 핵심은 개별 명령어나 `bpftool` 옵션을 외우는 것이 아니라, **개발자가 작성한 C 코드가 eBPF VM의 Bytecode로 변환되고, ELF Object File과 Verifier, JIT, Hook을 거쳐 실제 Kernel Event에서 실행되는 전체 흐름을 이해하는 것**이라고 정리할 수 있다.

이 흐름이 잡혀야 이후에 나오는 libbpf, BTF, CO-RE, XDP, TC, Cilium Datapath도 각각 별도의 기술이 아니라 하나의 eBPF 실행 구조 안에서 연결해 이해할 수 있다.
