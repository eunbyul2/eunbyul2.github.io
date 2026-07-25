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

Chapter 2에서는 BCC를 이용해 비교적 짧은 Python 코드로 eBPF 프로그램을 작성하고 실행했다. 당시에는 eBPF C 코드를 문자열로 작성한 뒤 `BPF()` 객체를 생성하면 BCC가 컴파일, 커널 로딩, Attach, 출력 수집까지 대부분 처리해주었다. 덕분에 eBPF 프로그램이 특정 Kernel Hook에서 실행된다는 흐름은 빠르게 확인할 수 있었지만, 실제로 C 코드가 어떤 형태로 변환되고 커널 내부에서 어떻게 관리되는지는 거의 드러나지 않았다.

Chapter 3에서는 BCC가 감추고 있던 이 과정을 직접 따라간다. 제한된 C로 eBPF 프로그램을 작성하고, Clang으로 eBPF Bytecode를 생성하고, ELF Object File을 분석한 뒤, `bpftool`을 이용해 커널에 Load하고 XDP Hook에 Attach한다. 이후 Verifier를 통과한 Bytecode와 JIT로 생성된 Native Machine Code를 확인하고, 전역 변수가 BPF Map으로 구현되는 방식과 Program의 Detach 및 Unload 과정까지 살펴본다.

이번 챕터의 전체 흐름은 다음과 같다.

```text
eBPF C Source
      ↓
Clang / LLVM
      ↓
eBPF Bytecode가 포함된 ELF Object File
      ↓
bpftool과 bpf() System Call
      ↓
Verifier 검증
      ↓
JIT Compilation
      ↓
Kernel에 Program Load
      ↓
XDP Hook에 Attach
      ↓
Packet 수신 시 Program 실행
```

## eBPF 프로그램은 가상 실행 모델을 기준으로 동작한다

eBPF Virtual Machine은 VMware나 OpenStack Instance처럼 운영체제 전체를 실행하는 가상 머신이 아니다. eBPF 명령어, 레지스터, 스택, 함수 호출 규약을 소프트웨어로 정의한 작은 가상 CPU 모델에 가깝다. 개발자가 작성한 C 코드는 커널에서 그대로 실행되지 않으며, 먼저 eBPF VM이 이해할 수 있는 Bytecode로 컴파일된다.

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

초기 구현에서는 커널의 Interpreter가 eBPF Instruction을 실행할 때마다 하나씩 해석했다. 이 방식은 매번 Opcode를 확인하고 대응 동작을 수행해야 하므로 반복 실행이 많은 네트워크 패킷 처리에서는 비용이 누적될 수 있다. 현재는 일반적으로 프로그램을 커널에 Load할 때 Bytecode를 한 번만 Native Machine Code로 변환하는 JIT 방식이 사용된다. JIT 결과는 CPU Architecture에 종속되기 때문에 동일한 eBPF Bytecode라도 x86-64에서는 x86-64 Machine Code로, ARM64에서는 ARM64 Machine Code로 변환된다.

eBPF VM에는 R0부터 R10까지 총 11개의 가상 레지스터가 있다. R0부터 R9까지는 일반 목적 레지스터이고, R10은 Stack Frame Pointer로 고정된다.

| Register | 역할 |
|---|---|
| R0 | 함수 및 eBPF Program의 반환값 |
| R1~R5 | Function 또는 Helper 호출 인자 |
| R6~R9 | 호출 이후에도 보존되는 Callee-Saved Register |
| R10 | Stack Frame Pointer |

eBPF Program이 시작될 때 Context Pointer는 R1에 전달된다. XDP Program이라면 R1은 일반적으로 `struct xdp_md *`를 가리키며, Program이 종료될 때는 R0에 XDP Verdict가 저장된다. Function이나 BPF Helper를 호출할 때는 최대 다섯 개의 인자를 R1부터 R5까지 배치하고, 반환값은 R0으로 받는다.

R10은 현재 Stack Frame의 기준점이며 프로그램이 임의로 변경할 수 없다. eBPF Stack은 R10에서 음수 Offset 방향으로 접근한다.

```text
R10          Stack Frame 기준점
R10 - 8      Local Variable
R10 - 16     임시 데이터
R10 - 24     구조체 또는 배열 일부
```

R10이 고정되어 있어야 Verifier가 Stack 접근 범위를 정확히 추적할 수 있다. 프로그램이 R10을 임의의 주소로 바꿀 수 있다면 `R10 - 8`이 실제 Stack인지 다른 Kernel Memory인지 판단할 수 없기 때문이다.

## eBPF Bytecode와 Instruction 구조

eBPF Program의 실제 본체는 C 코드가 아니라 eBPF Bytecode Instruction의 집합이다. Linux Kernel의 `include/uapi/linux/bpf.h`에는 하나의 eBPF Instruction을 표현하는 `struct bpf_insn`이 정의되어 있다.

```c
struct bpf_insn {
    __u8 code;
    __u8 dst_reg:4;
    __u8 src_reg:4;
    __s16 off;
    __s32 imm;
};
```

기본 eBPF Instruction 하나는 8바이트이며 다음 정보를 담는다.

| 필드 | 의미 |
|---|---|
| `code` | 수행할 연산을 나타내는 Opcode |
| `dst_reg` | 결과가 저장되거나 변경되는 Destination Register |
| `src_reg` | 입력값을 제공하는 Source Register |
| `off` | Memory Access 또는 Jump에 사용하는 Signed Offset |
| `imm` | Instruction 내부에 직접 포함되는 32-bit Immediate 값 |

대표적인 Instruction은 Register Load, Memory Store, 산술 연산, 조건부 Jump, Function Call, Exit 등으로 구분할 수 있다. 예를 들어 `R1 += 10`은 R1에 Immediate 값 10을 더하는 Instruction이고, `R1 += R2`는 R2의 값을 R1에 더하는 Register 연산이다.

기본 `bpf_insn`의 Immediate 영역은 32비트이므로 64비트 값을 하나의 Instruction에 모두 담을 수 없다. 이 경우에는 8바이트 Slot 두 개를 사용한 16바이트 Wide Instruction Encoding을 사용한다. Disassembly에서 Instruction Offset이 `0, 2, 3, 5`처럼 증가하는 이유도 Offset 0과 3의 Instruction이 각각 두 Slot을 차지하기 때문이다.

## XDP Hook에서 실행되는 Hello World

Chapter 2의 Hello World는 System Call에 연결된 kprobe나 Tracepoint에서 실행되었다. Chapter 3의 예제는 네트워크 인터페이스로 패킷이 들어올 때마다 실행되는 XDP Program이다.

XDP는 eXpress Data Path의 약자로, 네트워크 패킷이 Linux Network Stack의 깊은 단계로 진입하기 전에 실행되는 Hook이다. 일반적인 수신 경로를 단순화하면 다음과 같다.

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

XDP Program은 패킷을 검사하거나 수정한 뒤 Kernel이 해당 패킷을 어떻게 처리할지 Verdict를 반환한다.

| Verdict | 의미 |
|---|---|
| `XDP_ABORTED` | 비정상 종료 |
| `XDP_DROP` | 패킷 폐기 |
| `XDP_PASS` | 일반 Linux Network Stack으로 전달 |
| `XDP_TX` | 수신한 Interface로 다시 송신 |
| `XDP_REDIRECT` | 다른 Interface나 대상에 전달 |

이번 예제는 패킷 내용을 분석하지 않고 Trace Log를 출력한 뒤 항상 `XDP_PASS`를 반환한다.

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

`<linux/bpf.h>`에는 XDP Action과 BPF 관련 Kernel UAPI가 정의되어 있고, `<bpf/bpf_helpers.h>`에는 `SEC()` Macro와 BPF Helper 선언이 포함되어 있다.

```c
SEC("xdp")
int hello(void *ctx)
```

`SEC("xdp")`는 `hello()` 함수를 ELF Object File의 `xdp` Section에 배치한다. Loader는 이 Section Name을 이용해 해당 함수가 XDP Program이라는 사실을 파악한다. `hello`는 C 함수 이름이면서 커널에 로드된 뒤 표시되는 Program Name이다.

`ctx`는 XDP Event Context이지만 이번 예제에서는 패킷 내용을 읽지 않기 때문에 사용하지 않는다. 일반적인 XDP Program에서는 `struct xdp_md *ctx`를 사용해 Packet Data와 `data_end`를 가져온다.

```c
bpf_printk("Hello World %d", counter);
```

eBPF Program은 User Space의 libc를 사용할 수 없기 때문에 일반적인 `printf()`를 호출할 수 없다. `bpf_printk()`는 Kernel의 Trace Buffer에 문자열을 기록하는 디버깅용 Wrapper이며, 내부적으로 `bpf_trace_printk` Helper를 사용한다. 출력은 `/sys/kernel/debug/tracing/trace_pipe` 또는 `bpftool prog tracelog`에서 확인할 수 있다.

```c
counter++;
```

Counter는 Program이 패킷마다 실행될 때마다 증가한다. eBPF Program의 한 번의 실행이 종료된 뒤에도 값이 유지되어야 하므로 단순한 Stack Local Variable이 아니라 Global Data로 관리된다. 이 전역 변수는 실제로 `.bss` 영역을 나타내는 BPF Map으로 구현된다.

```c
return XDP_PASS;
```

`XDP_PASS`의 값은 2이며 eBPF Calling Convention에 따라 반환값은 R0에 저장된다. 따라서 Bytecode에서는 `r0 = 2`와 `exit`으로 변환된다.

마지막의 License Section도 단순한 설명용 문자열은 아니다.

```c
char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

일부 BPF Helper는 GPL-compatible Program에서만 사용할 수 있다. Loader와 Kernel은 Object File의 License Section을 확인해 Program이 사용하는 Helper와 License가 호환되는지 검사한다. BPF LSM과 같은 일부 Program Type도 GPL-compatible License를 요구한다.

## C 코드를 eBPF Object File로 컴파일하기

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

각 옵션의 역할은 다음과 같다.

| 옵션 | 의미 |
|---|---|
| `-target bpf` | Host CPU가 아닌 eBPF ISA를 대상으로 컴파일 |
| `-I...` | Architecture별 Linux Header 검색 경로 추가 |
| `-g` | Debug Information과 BTF 관련 정보 생성 |
| `-O2` | 불필요한 Instruction과 Memory Access를 줄이기 위한 최적화 |
| `-c` | Linking 없이 Object File까지만 생성 |
| `-o` | 출력 파일 지정 |

컴파일 과정을 내부적으로 보면 다음과 같다.

```text
hello.bpf.c
    ↓ Clang Frontend
LLVM IR
    ↓ LLVM BPF Backend
eBPF Bytecode
    ↓ ELF Packaging
hello.bpf.o
```

`file` 명령으로 생성된 Object File의 성격을 확인할 수 있다.

```bash
file hello.bpf.o
```

```text
hello.bpf.o: ELF 64-bit LSB relocatable, eBPF,
version 1 (SYSV), with debug_info, not stripped
```

이 결과에서 `ELF`는 Linux의 Executable and Linkable Format을 의미한다. `relocatable`은 아직 Map Reference나 Global Data의 최종 위치가 결정되지 않은 Object File이라는 뜻이며, `eBPF`는 내부 Code가 x86이나 ARM이 아닌 eBPF Instruction으로 구성되었다는 의미다. `debug_info`는 `-g` 옵션으로 Debug 및 Type 정보가 포함되었음을 나타낸다.

## Object File에서 eBPF Bytecode 확인하기

다음 명령으로 Object File의 Bytecode와 Source Line을 함께 확인할 수 있다.

```bash
llvm-objdump -S hello.bpf.o
```

핵심 Disassembly는 다음과 같다.

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

Offset 0과 3의 `ll` Instruction은 64비트 값을 다루는 Wide Instruction이므로 각각 16바이트, 즉 두 개의 Instruction Slot을 차지한다. 그래서 다음 Instruction이 1이 아니라 2, 4가 아니라 5에서 시작한다.

Object File 단계에서 다음 Instruction의 값은 0으로 보인다.

```text
r6 = 0 ll
r1 = 0 ll
```

이 값은 실제로 영구적인 숫자 0을 사용한다는 의미가 아니다. 아직 `.bss`와 `.rodata`를 담을 BPF Map이 커널에 생성되지 않았기 때문에 Loader가 나중에 실제 Map Reference로 치환할 Placeholder다.

`bpf_printk()` 호출을 위해 Register는 다음처럼 사용된다.

```text
R1 = Format String 주소
R2 = Format String 길이
R3 = counter 값
CALL bpf_trace_printk
```

`r2 = 15`에서 15는 `"Hello World %d"` 문자열과 종료 문자 `\0`을 포함한 길이다. 이후 `call 6`은 BPF Helper ID 6에 해당하는 Trace Print Helper 호출이다.

`counter++`는 C에서는 한 줄이지만 Bytecode에서는 세 단계로 분해된다.

```text
r1 = *(u32 *)(r6 + 0)   현재 Counter Load
r1 += 1                 1 증가
*(u32 *)(r6 + 0) = r1   증가한 값을 다시 Store
```

마지막 두 Instruction은 `return XDP_PASS;`에 대응한다.

```text
r0 = 2
exit
```

R0는 Program의 반환값이고 2는 `XDP_PASS`다.

## eBPF Program을 Kernel에 Load하기

컴파일된 Object File은 다음 명령으로 Kernel에 Load한다.

```bash
bpftool prog load hello.bpf.o /sys/fs/bpf/hello
```

이 명령은 단순히 파일을 Kernel Memory로 복사하는 것이 아니다. 내부에서는 다음 과정이 수행된다.

```text
bpftool이 ELF Object File 분석
        ↓
.bss와 .rodata용 BPF Map 생성
        ↓
Bytecode의 Map Reference Relocation
        ↓
bpf(BPF_PROG_LOAD) System Call
        ↓
Verifier 검증
        ↓
JIT Compilation
        ↓
Kernel Program Object 생성
        ↓
/sys/fs/bpf/hello에 Pin
```

Verifier는 Bytecode의 모든 가능한 실행 경로를 분석한다. 유효하지 않은 Opcode, 잘못된 Jump, 초기화되지 않은 Register 또는 Stack Read, 허용되지 않은 Pointer Access, 잘못된 Helper Argument, Return Value 누락 등이 있으면 Load를 거부한다.

Program이 Verifier를 통과하면 JIT Compiler가 eBPF Bytecode를 현재 CPU Architecture의 Native Machine Code로 변환한다. 이후 `bpftool`은 Program을 `/sys/fs/bpf/hello`에 Pin한다.

`/sys/fs/bpf/hello`는 일반 Disk File이 아니라 BPF Filesystem에 생성된 Kernel Object Reference다. Program을 Load한 `bpftool` Process가 종료되더라도 Pin이 남아 있으면 Program은 Kernel에 유지된다.

## 커널에 Load된 Program 정보 확인하기

다음 명령으로 현재 커널에 Load된 eBPF Program을 확인할 수 있다.

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

각 항목은 Program의 Kernel Runtime 상태를 보여준다.

| 항목 | 의미 |
|---|---|
| `540` | Kernel이 Load 시 할당한 Program ID |
| `xdp` | Program Type |
| `name hello` | Source Function Name |
| `tag` | Instruction을 기반으로 계산한 Program Identifier |
| `gpl` | GPL-compatible License |
| `uid 0` | Root User가 Load |
| `xlated 96B` | Verifier 처리 이후 eBPF Bytecode 크기 |
| `jited 148B` | JIT가 생성한 Native Machine Code 크기 |
| `map_ids` | Program이 참조하는 BPF Map |
| `btf_id` | Program과 연결된 BTF 정보 |

Program ID는 현재 Kernel에서 고유하지만 Unload 후 다시 Load하면 변경될 수 있다. 반면 Tag는 Program Instruction을 기반으로 계산되므로 동일한 Bytecode라면 같은 값을 유지할 수 있다. 다만 동일한 Name이나 Tag를 가진 Program이 여러 개 존재할 수 있으므로 항상 유일한 식별자는 아니다.

Program은 ID, Name, Tag, Pinned Path로 조회할 수 있다.

```bash
bpftool prog show id 540
bpftool prog show name hello
bpftool prog show tag d35b94b4c0c10efb
bpftool prog show pinned /sys/fs/bpf/hello
```

## C 코드, Translated Bytecode, JIT Machine Code

Chapter 3의 핵심은 같은 Program을 서로 다른 세 수준에서 확인하는 것이다.

### C Source Code

```c
counter++;
return XDP_PASS;
```

사람이 읽고 작성하기 쉬운 고수준 표현이다.

### Verifier 이후 Translated Bytecode

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

Object File에서 0으로 보이던 Placeholder가 실제 Map Reference로 변경되었다.

```text
r6 = 0 ll
→ r6 = map[id:165][0]+0

r1 = 0 ll
→ r1 = map[id:166][0]+0
```

Map ID 165는 `counter`가 저장된 `.bss` Map이고, Map ID 166은 Format String이 저장된 `.rodata` Map이다. Helper Call도 단순한 숫자에서 실제 Helper Name이 표시되는 형태로 Resolution된다.

### JIT-Compiled Machine Code

```bash
bpftool prog dump jited name hello
```

JIT Dump는 실제 CPU Architecture의 Assembly를 보여준다. ARM64 환경에서는 `ldr`, `add`, `str`, `blr`, `ret` 같은 Instruction과 `x0`, `x19`, `sp` 등의 Register가 나타나며, x86-64 환경에서는 `mov`, `add`, `call`, `ret`과 `rax`, `rbx`, `rdi` 같은 Register가 나타난다.

예를 들어 Counter 증가는 ARM64에서 대략 다음과 같은 Load, Add, Store 흐름으로 변환된다.

```text
ldr    Memory에서 Counter Load
add    1 증가
str    Memory에 결과 Store
```

eBPF의 R0 반환값은 실제 CPU Calling Convention의 반환 Register로 매핑된다. ARM64에서는 최종적으로 `x0`에 `XDP_PASS` 값 2가 들어가고 `ret`으로 종료된다.

결국 동일한 로직은 다음 세 형태로 존재한다.

```text
개발자가 작성하는 형태
C Source Code

Kernel Verifier가 분석하는 형태
eBPF Bytecode

실제 CPU가 실행하는 형태
Native Machine Code
```

## Load와 Attach는 별개의 단계다

Program을 Kernel에 Load했다고 해서 바로 실행되는 것은 아니다. Load는 Program Object를 Kernel에 생성하는 작업이고, Attach는 어떤 Event가 해당 Program을 실행할지 연결하는 작업이다.

```text
Load
→ Kernel에 Program 존재

Attach
→ Program과 Event Hook 연결

Event 발생
→ Program 실행
```

XDP Program을 `eth0` Interface에 연결하려면 다음 명령을 사용한다.

```bash
bpftool net attach xdp id 540 dev eth0
```

연결 상태는 다음 명령으로 확인한다.

```bash
bpftool net list
```

```text
xdp:
eth0(2) driver id 540
```

`driver`는 Native Driver Mode로 Attach되었다는 의미다. `ip link`에서도 Program ID, Tag, JIT 여부를 확인할 수 있다.

```text
prog/xdp id 540 tag ... jited
```

Attach 이후에는 `eth0`로 패킷이 들어올 때마다 Program이 실행된다.

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

## XDP Trace에서 `<idle>-0`이 보이는 이유

Trace 출력은 다음 명령으로 확인한다.

```bash
cat /sys/kernel/debug/tracing/trace_pipe
```

또는 다음 명령을 사용할 수 있다.

```bash
bpftool prog tracelog
```

예시 출력은 다음과 같다.

```text
<idle>-0 [003] d.s.. 655370.944105:
bpf_trace_printk: Hello World 4531
```

Chapter 2의 System Call Trace에서는 특정 User Process가 System Call을 호출했기 때문에 Process Name과 PID가 표시되었다.

```text
User Process
    ↓
System Call
    ↓
Kernel Hook
    ↓
eBPF Program
```

반면 XDP Program은 네트워크 패킷이 NIC에 도착한 직후 Driver 또는 SoftIRQ Context에서 실행된다. 이 시점에는 아직 Packet Parsing, Routing, TCP/UDP 처리, Socket Lookup이 끝나지 않았으므로 해당 패킷을 어떤 User Process가 받을지 결정되지 않았다.

```text
Packet
  ↓
NIC
  ↓
Driver
  ↓
XDP Program
  ↓
이후에야 Socket과 Process 결정
```

따라서 특정 User Process Context 대신 `<idle>-0`과 같은 값이 나타날 수 있다. 실제 표시 값은 Kernel Version과 Driver 처리 Context에 따라 달라질 수 있다.

## 전역 변수는 BPF Map으로 구현된다

Source Code에서는 전역 변수를 일반 C와 같은 방식으로 작성했다.

```c
int counter = 0;
```

하지만 eBPF Program은 Event가 발생할 때마다 독립적으로 실행되므로 실행 간 상태를 유지하려면 Kernel에 지속되는 저장 공간이 필요하다. libbpf는 Global Variable을 BPF Map Semantics로 구현한다.

```text
C Global Variable
        ↓
ELF Data Section
        ↓
Loader
        ↓
Internal BPF Map
```

이번 예제에는 두 개의 Map이 생성된다.

```text
hello.bss
→ counter

hello.rodata
→ "Hello World %d"
```

### `.bss` Map

`.bss`는 0으로 초기화되는 수정 가능한 Global 또는 Static Data를 저장하는 ELF Section이다.

```text
165: array name hello.bss
key 4B value 4B max_entries 1
```

Map Entry는 하나이며 Key 0의 Value 전체가 `.bss` 영역을 나타낸다. 이번 Program에는 4바이트 `int counter` 하나만 있으므로 Value Size도 4바이트다.

```bash
bpftool map dump name hello.bss
```

BTF 정보가 있으면 다음처럼 Source Variable Name과 값을 확인할 수 있다.

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

`-g` 없이 컴파일해 BTF 정보가 없다면 `bpftool`은 Variable Name과 Type을 알 수 없으므로 Raw Byte만 표시한다.

```text
key: 00 00 00 00 value: 19 01 00 00
```

Little Endian 환경에서 `19 01 00 00`은 `0x00000119`, 즉 Decimal 281이다.

### `.rodata` Map

`.rodata`는 읽기 전용 Global Data를 저장하는 Section이다. 이번 Program에서는 `bpf_printk()`의 Format String이 저장된다.

```text
166: array name hello.rodata
key 4B value 15B max_entries 1 frozen
```

Value Size 15바이트는 `"Hello World %d"`의 14바이트와 종료 문자 `\0` 1바이트를 합한 크기다. `frozen`은 Loader가 초기 값을 기록한 뒤 Map Update를 금지했다는 의미다.

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

BTF가 없으면 다음처럼 ASCII Byte가 표시된다.

```text
48 65 6c 6c 6f 20 57 6f 72 6c 64 20 25 64 00
```

이를 ASCII로 해석하면 `"Hello World %d\0"`이다.

이 예제의 `counter++`는 동시성 제어가 없는 단순한 학습용 코드다. 실제 환경에서는 여러 CPU가 동시에 같은 값을 읽고 갱신하면서 증가가 누락될 수 있으므로 Atomic 연산이나 Per-CPU Map을 고려해야 한다.

## Detach와 Unload는 다르다

XDP Hook에서 Program을 제거하려면 다음 명령을 사용한다.

```bash
bpftool net detach xdp dev eth0
```

Detach는 Event와 Program의 연결만 제거한다.

```text
Before

eth0 XDP Hook
      ↓
hello Program

After

eth0 XDP Hook
      ↓
Program 없음
```

하지만 Program은 여전히 `/sys/fs/bpf/hello`에 Pin되어 있으므로 Kernel에 남아 있다.

```bash
bpftool prog show name hello
```

eBPF Object의 수명은 Reference Count로 관리된다. 대표적인 Reference는 Open File Descriptor, BPF Link, Network Attach, bpffs Pin 등이 있다. Network Hook에서 Detach하더라도 Pin Reference가 남아 있으면 Program은 제거되지 않는다.

Pinned Path를 삭제하면 해당 Reference가 사라진다.

```bash
rm /sys/fs/bpf/hello
```

다른 Attach, Link, File Descriptor가 없다면 Reference Count가 0이 되고 Kernel이 Program Object를 해제한다.

```text
Detach
→ Event 연결만 제거

Pin 삭제
→ bpffs Reference 제거

모든 Reference 소멸
→ Kernel Program Unload
```

## BPF-to-BPF Function Call

Chapter 2에서는 Tail Call을 통해 한 eBPF Program에서 다른 eBPF Program으로 실행을 넘기는 방식을 확인했다. Chapter 3에서는 하나의 eBPF Program 내부에서 일반 Function을 호출하는 BPF-to-BPF Call을 다룬다.

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

`get_opcode()`는 너무 단순해서 Compiler가 호출 자체를 없애고 Function Body를 호출 위치에 삽입할 가능성이 높다. 이를 Inlining이라고 한다. 이번 예제는 실제 Function Call Instruction을 확인하는 것이 목적이므로 `__attribute((noinline))`을 사용해 Inlining을 막는다.

Translated Bytecode는 다음과 같다.

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

Offset 0의 `call pc+7`은 다음 Instruction인 Offset 1을 기준으로 7개 앞으로 이동해 Offset 8의 `get_opcode()`를 실행한다.

```text
Offset 1 + 7
= Offset 8
```

Function Entry 시 R1에는 `ctx`가 전달되어 있고, `ctx->args[1]`은 첫 번째 64-bit Element에서 8바이트 떨어져 있으므로 다음 Instruction으로 표현된다.

```text
r0 = *(u64 *)(r1 + 8)
```

R0은 Function Return Value다. `get_opcode()`가 종료되면 Caller인 `hello()`로 돌아오고, 반환된 R0 값을 `bpf_printk()`의 세 번째 Argument로 사용하기 위해 R3으로 복사한다.

```text
r3 = r0
```

BPF-to-BPF Call과 Tail Call은 동작이 다르다.

| 구분 | BPF-to-BPF Call | Tail Call |
|---|---|---|
| 호출 대상 | 같은 Program 내부 Function | 다른 eBPF Program |
| 호출 후 복귀 | Caller로 복귀 | 성공하면 기존 Program으로 복귀하지 않음 |
| 대상 결정 | Compile된 상대 Offset | Program Array Map |
| 주요 목적 | 함수 분리와 코드 재사용 | Program Chaining과 Dynamic Dispatch |
| Stack | 추가 Call Frame 필요 | 일반 Function Call과 다른 전환 방식 |

일반 Function Call은 호출 후 원래 위치로 돌아와야 하므로 Return Address, Register 상태, Stack Frame을 관리해야 한다. eBPF Stack은 제한되어 있고 Function Call Depth도 Verifier가 검사하므로 일반 User Space Program처럼 깊은 호출 구조나 큰 Local Variable을 자유롭게 사용할 수 없다.

## Chapter 3 정리

Chapter 3을 읽기 전에는 eBPF Program을 C 코드와 Hook 정도로만 생각했다. 하지만 실제 Program은 Source Code 하나로 끝나는 것이 아니라 여러 단계를 거쳐 Kernel에서 실행된다.

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

eBPF VM은 R0부터 R10까지의 Register, Stack, Instruction Set, Calling Convention을 정의한다. Compiler는 C 코드를 이 모델에 맞는 Bytecode로 변환하고, Verifier는 Bytecode의 Register 상태와 Pointer Access, Stack Access, Control Flow를 분석한다. 검증된 Program은 JIT를 통해 실제 CPU용 Machine Code로 변환된다.

또한 Load와 Attach는 서로 다른 단계다. Kernel에 Program을 Load해도 Hook에 Attach하지 않으면 실행되지 않는다. 반대로 Detach는 Hook과의 연결만 해제할 뿐 Program 자체를 제거하지 않는다. Program은 File Descriptor, BPF Link, Attach, bpffs Pin과 같은 Reference가 모두 사라질 때 Unload된다.

Source Code에서 단순한 전역 변수로 보였던 `counter`와 Format String도 실제로는 `.bss`와 `.rodata`를 표현하는 BPF Map으로 구현된다. 이 과정에서 BTF는 Source Variable Name과 Type 정보를 유지해 `bpftool`이 Raw Byte가 아닌 구조화된 값을 보여줄 수 있도록 한다.

결국 이번 챕터의 핵심은 특정 명령어를 외우는 것이 아니라, **개발자가 작성한 C 코드가 어떤 단계와 자료구조를 거쳐 커널에서 안전하고 빠르게 실행되는지 이해하는 것**이다. 이 생명주기를 이해해야 이후에 등장하는 libbpf, CO-RE, BTF, XDP, TC, Cilium Datapath를 각각 독립된 기술이 아니라 하나의 eBPF 실행 체계 안에서 파악할 수 있다.
