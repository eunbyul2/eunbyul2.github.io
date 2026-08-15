---
layout: post
title: "[Learning eBPF] Chapter 5: CO-RE, BTF, and Libbpf"
date: 2026-08-15 18:40:00 +0900
categories: [eBPF, Linux, Kernel]
tags: [eBPF, BPF, CO-RE, BTF, libbpf, bpftool, vmlinux]
published: true
---

# Learning eBPF Chapter 5

## CO-RE, BTF, and Libbpf

Chapter 3에서는 eBPF C Source가 Clang을 통해 Bytecode가 포함된 ELF Object File로 만들어지고, Kernel에 Load된 뒤 Hook에 Attach되어 실행되는 과정을 살펴봤다.

Chapter 4에서는 이 과정에서 User Space Program이 `bpf()` System Call을 이용해 Map과 Program 같은 BPF Object를 실제 Kernel에 생성하고 관리한다는 것을 확인했다.

여기까지 공부하고 나면 한 가지 의문이 생긴다.

**한 환경에서 Compile한 eBPF Program을 다른 Linux Kernel에서도 그대로 실행할 수 있을까?**

Chapter 5는 이 문제에서 시작한다.

---

## eBPF Program은 왜 Kernel Version의 영향을 받을까?

일반적인 Application은 Kernel 내부의 Data Structure를 직접 읽는 경우가 많지 않다. User Space에서는 System Call이라는 비교적 안정된 Interface를 통해 Kernel 기능을 사용하기 때문이다.

하지만 eBPF Program은 Kernel 내부에서 실행되고, Observability나 Security 목적으로 Kernel Structure의 정보를 직접 읽는 경우가 많다.

예를 들어 Kernel 내부에 다음과 같은 Structure가 있다고 생각해보자.

```c
struct example {
    int pid;
    int state;
    long value;
};
```

Memory에서는 각 Field가 일정한 위치에 배치된다.

```text
struct example

+0   pid
+4   state
+8   value
```

eBPF Program에서 다음과 같이 접근하면,

```c
data = ptr->value;
```

최종적으로는 Structure의 시작 주소에서 `value`가 위치한 Offset을 이용해 Memory를 읽게 된다.

문제는 Linux Kernel의 내부 Data Structure가 고정되어 있지 않다는 점이다.

Kernel Version이 달라지면서 Field가 추가되거나 순서가 변경될 수 있다.

```text
Kernel A

+0   pid
+4   state
+8   value
```

다른 Kernel에서는 다음과 같을 수도 있다.

```text
Kernel B

+0   pid
+4   new_field
+8   state
+16  value
```

같은 `value`라는 Field라도 실제 Memory Offset이 달라졌다.

만약 Kernel A를 기준으로 Compile된 eBPF Instruction이 단순히 `+8` 위치를 읽도록 만들어졌다면 Kernel B에서는 전혀 다른 값을 읽게 된다.

즉 eBPF Program의 Portability 문제는 단순히 "Kernel Version이 다르다"는 문제가 아니라, **Compile할 때 알고 있던 Kernel Type의 Memory Layout과 실제 실행하는 Kernel의 Layout이 다를 수 있다는 문제**다.

---

## 기존 BCC 방식은 어떻게 해결했을까?

Chapter 2에서 처음 eBPF Program을 작성할 때 BCC를 사용했다.

당시 Python Program 안에 eBPF C Source를 문자열로 작성했다.

```python
program = r"""
int hello(void *ctx) {
    ...
}
"""

b = BPF(text=program)
```

처음에는 BCC가 eBPF Program을 쉽게 작성할 수 있게 해주는 Framework 정도로 볼 수 있었다.

Chapter 5까지 공부하고 나면 BCC가 왜 이런 구조를 사용했는지도 조금 더 명확해진다.

BCC는 eBPF Bytecode를 미리 만들어 배포하는 것이 아니라 **Program이 실행되는 Machine에서 eBPF Source를 Compile한다.**

```text
BCC Application 실행
        ↓
eBPF C Source
        ↓
현재 Machine의 Kernel Header 사용
        ↓
Clang Compile
        ↓
현재 Kernel에 맞는 eBPF Bytecode
        ↓
Kernel Load
```

이렇게 하면 Compile하는 시점과 실행하는 Kernel이 동일하므로 Kernel Structure 차이를 자연스럽게 해결할 수 있다.

하지만 배포 관점에서는 단점이 생긴다.

Target Machine에 BCC, Clang/LLVM, Kernel Header 등 Compile을 위한 구성 요소가 필요하고, Application 실행 시점에도 Compile 과정이 필요하다.

즉 BCC는 **Target 환경에서 다시 Compile하는 방식으로 Portability를 확보한다.**

---

## CO-RE는 무엇을 바꾼 것인가?

CO-RE는 **Compile Once – Run Everywhere**의 약자다.

이름만 보면 한 번 Compile한 Binary가 아무 Linux Kernel에서나 무조건 실행되는 것처럼 느껴질 수 있지만, 핵심은 조금 다르다.

CO-RE는 Kernel 차이를 무시하지 않는다.

오히려 **Kernel마다 Data Structure가 달라질 수 있다는 사실을 인정하고, 실행할 Kernel의 정보를 확인한 뒤 필요한 부분을 Load 시점에 수정한다.**

BCC 방식과 비교하면 차이가 명확하다.

```text
BCC

Target Machine
    ↓
현재 Kernel 기준으로
다시 Compile
    ↓
Load
```

CO-RE는 다음과 같다.

```text
Build Machine

eBPF Source
    ↓
미리 Compile
    ↓
eBPF Object File
    │
    │ 배포
    ▼
Target Machine
    ↓
실행 중인 Kernel 구조 확인
    ↓
필요한 부분 Relocation
    ↓
Load
```

즉 Portability를 해결하는 시점이 달라진 것이다.

```text
BCC
→ Compile 시점에 Target Kernel에 맞춘다.

CO-RE
→ 미리 Compile하고 Load 시점에 Target Kernel에 맞춘다.
```

Chapter 5의 전체 내용은 이 차이를 중심으로 이해하면 훨씬 정리하기 쉽다.

---

## Runtime Kernel의 구조는 어떻게 알 수 있을까?

여기서 필요한 것이 **BTF(BPF Type Format)**다.

BTF는 Kernel 내부의 Type 정보를 표현하는 Format이다.

예를 들어 Kernel에 다음 Structure가 있다고 해보자.

```c
struct task_example {
    int pid;
    long state;
};
```

BTF는 단순히 `task_example`이라는 이름만 저장하는 것이 아니라 Structure의 크기, Member 이름, Member Type, Offset과 같은 정보를 표현할 수 있다.

개념적으로 보면 다음과 같다.

```text
struct task_example

size = ...

member pid
  type   = int
  offset = ...

member state
  type   = long
  offset = ...
```

즉 BTF는 Binary 안의 Memory Layout을 **Type 정보와 연결해서 설명할 수 있게 해주는 Metadata**다.

Chapter 3에서도 BTF를 살펴봤지만, 당시에는 `bpftool`이 Program과 Map의 Type 정보를 해석하는 데 사용하는 Metadata 정도로 볼 수 있었다.

Chapter 5에서는 BTF가 CO-RE의 핵심 기반이라는 점이 드러난다.

```text
Compile 당시 Type 정보
        ↕
Runtime Kernel Type 정보
```

두 환경의 Type 정보를 비교할 수 있어야 실제 Kernel에서 Field가 어디에 존재하는지 판단할 수 있기 때문이다.

---

## Kernel 전체의 BTF 정보

BTF를 지원하는 Kernel에서는 일반적으로 다음 경로에서 실행 중인 Kernel의 BTF 정보를 확인할 수 있다.

```bash
/sys/kernel/btf/vmlinux
```

`bpftool`을 사용하면 이 정보를 사람이 확인할 수 있는 형태로 Dump할 수 있다.

```bash
bpftool btf dump file /sys/kernel/btf/vmlinux
```

여기서 중요한 것은 `/sys/kernel/btf/vmlinux`가 단순한 Kernel Image를 의미하는 것이 아니라 **현재 실행 중인 Kernel의 BTF Type 정보를 제공하는 객체**라는 점이다.

그리고 이 정보를 C Header 형태로 변환할 수도 있다.

```bash
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

이렇게 생성되는 파일이 CO-RE 기반 eBPF Program에서 자주 등장하는 `vmlinux.h`다.

---

## `vmlinux.h`는 무엇인가?

eBPF Program에서 Kernel Structure를 사용하려면 Compiler도 해당 Type Definition을 알고 있어야 한다.

기존 방식에서는 필요한 Linux Kernel Header를 직접 Include해야 했다.

하지만 Kernel Header는 굉장히 많고 서로 의존하는 Header도 많기 때문에 eBPF Program의 Build 환경을 복잡하게 만들 수 있다.

CO-RE에서는 Kernel BTF를 이용해 Kernel Type Definition을 C Header 형태로 생성할 수 있다.

```text
Running Kernel
    ↓
/sys/kernel/btf/vmlinux
    ↓
bpftool
    ↓
vmlinux.h
```

eBPF Program에서는 다음처럼 사용할 수 있다.

```c
#include "vmlinux.h"
```

따라서 `vmlinux.h`는 Kernel Source 전체를 그대로 복사한 Header가 아니다.

**Kernel BTF에 들어 있는 Type 정보를 C Definition 형태로 변환한 Header File**이라고 이해하는 것이 정확하다.

---

## `vmlinux.h`만 있으면 CO-RE가 되는 것은 아니다

처음에는 다음과 같이 생각하기 쉽다.

```text
Kernel BTF
→ vmlinux.h 생성
→ Kernel Structure 정보 확보
→ 끝?
```

하지만 `vmlinux.h`는 Compile 단계에서 Kernel Type Definition을 제공하는 역할이다.

Portability를 구현하려면 Compiler가 나중에도 다음 의미를 알 수 있도록 정보를 남겨야 한다.

```text
"이 Instruction에서 +8을 읽는 이유는

단순히 +8이라는 주소가 필요해서가 아니라

어떤 struct의 특정 field를 읽기 위해서다."
```

일반적인 Compile 결과에서 Field Access가 단순 Offset으로 완전히 변환되어 버리면, Runtime에서는 `+8`이라는 숫자가 어떤 Type의 어떤 Field를 의미했는지 알 수 없다.

그래서 CO-RE에서는 **Field Access의 의미를 Object File에 Relocation Metadata로 함께 남긴다.**

---

## CO-RE Relocation

CO-RE의 핵심 동작은 **Relocation**이다.

예를 들어 Source Code가 다음과 같다고 해보자.

```c
value = task->field;
```

Compile 당시 `field`가 Structure 시작 위치에서 `+8`에 있었다고 가정한다.

```text
Compile-time Kernel

struct task
field → offset 8
```

단순한 Compile 결과라면 다음과 비슷한 Memory Access Instruction만 남을 수 있다.

```text
load [register + 8]
```

하지만 CO-RE에서는 이 Access가 **어떤 Type의 어떤 Field를 읽기 위한 것인지** 알 수 있는 Relocation 정보도 함께 Object File에 기록한다.

Target Machine에서는 Program을 Kernel에 Load하기 전에 Runtime Kernel BTF를 확인한다.

```text
Compile-time BTF
        +
CO-RE Relocation
        +
Runtime Kernel BTF
```

Runtime Kernel에서 같은 Field의 Offset이 `+16`으로 변경되어 있다면 Loader는 그 차이를 찾아낸다.

```text
Compile 당시

load [r1 + 8]

        ↓

CO-RE Relocation

        ↓

Runtime Kernel에 맞게

load [r1 + 16]
```

즉 CO-RE는 Source Code를 Runtime에 다시 Compile하는 것이 아니라 **이미 Compile된 eBPF Instruction 중 Kernel Layout에 의존하는 부분을 Runtime Kernel에 맞춰 수정한다.**

---

## Relocation은 Program 실행 때마다 일어나는 것이 아니다

CO-RE를 처음 보면 Program이 Kernel Structure에 접근할 때마다 BTF를 조회하고 Offset을 계산하는 것으로 오해할 수 있다.

하지만 그렇게 동작하면 eBPF의 Runtime 성능에 큰 부담이 된다.

CO-RE Relocation은 **Program을 Kernel에 Load하기 전에 한 번 수행된다.**

```text
eBPF Object Open
        ↓
Runtime Kernel BTF 확인
        ↓
CO-RE Relocation
        ↓
Instruction Patch
        ↓
bpf() System Call
        ↓
Verifier
        ↓
JIT
        ↓
Kernel에서 실행
```

Runtime에서는 이미 Target Kernel에 맞게 수정된 Instruction을 실행한다.

따라서 Packet이나 Trace Event가 발생할 때마다 BTF를 다시 비교하는 구조가 아니다.

---

## `preserve_access_index`는 왜 필요한가?

`vmlinux.h`를 살펴보면 다음과 같은 Clang Attribute를 볼 수 있다.

```c
#pragma clang attribute push (__attribute__((preserve_access_index)), \
                              apply_to = record)
```

이 Attribute는 CO-RE가 Structure Field Access의 의미를 Compile 이후에도 추적할 수 있게 하는 데 사용된다.

예를 들어 Source Code에 다음 Access가 있다고 해보자.

```c
task->field
```

Compiler가 이를 단순히 다음과 같은 Offset Access로만 바꿔버린다면,

```text
register + 8
```

Runtime에서는 이 `8`이라는 값이 원래 `task->field`를 의미했다는 사실을 알 수 없다.

`preserve_access_index`는 이러한 Structure Access 정보를 보존하여 Clang이 CO-RE Relocation 정보를 생성할 수 있도록 한다.

개념적으로 보면 다음과 같다.

```text
task->field
    ↓
preserve_access_index
    ↓
Field Access 의미 보존
    ↓
CO-RE Relocation Metadata 생성
```

---

## Object File은 이제 단순한 Bytecode 파일이 아니다

Chapter 3에서는 `.bpf.o` 안에 eBPF Bytecode와 ELF Section이 들어 있다는 것을 확인했다.

CO-RE 방식에서는 Object File에 더 많은 Metadata가 포함된다.

```text
program.bpf.o

├── eBPF Bytecode
├── Map Definition
├── BTF
├── Function / Line Information
└── CO-RE Relocation Information
```

BTF와 확장 BTF 정보는 ELF Section 안에서 확인할 수 있다.

예를 들어 다음 명령으로 관련 Section을 확인할 수 있다.

```bash
readelf -S program.bpf.o | grep BTF
```

즉 `.bpf.o`는 단순히 Kernel에서 실행할 Instruction만 저장하는 파일이 아니다.

**Loader가 Program의 Type 정보와 Relocation 정보를 이해하고 Target Kernel에 맞게 조정할 수 있도록 필요한 Metadata까지 함께 담고 있는 ELF Object File**이다.

---

## libbpf는 어디에서 동작할까?

여기까지 보면 다음 질문이 남는다.

**Runtime Kernel의 BTF를 읽고 CO-RE Relocation을 적용하는 것은 누가 하는가?**

C 기반 eBPF Application에서는 대표적으로 **libbpf**가 이 역할을 수행한다.

Chapter 4에서 확인했듯이 Kernel에 BPF Map이나 Program을 생성하는 최종 Interface는 `bpf()` System Call이다.

하지만 실제 Application에서 다음 작업을 모두 직접 구현하는 것은 복잡하다.

```text
ELF Object Parsing
BTF Parsing
Map Definition 확인
BPF Map 생성
CO-RE Relocation
BPF Program Load
Hook Attach
Link 관리
File Descriptor 관리
Cleanup
```

libbpf는 이런 작업을 담당하는 User Space Library다.

전체 관계는 다음처럼 볼 수 있다.

```text
User Space Application
        ↓
      libbpf
        ↓
ELF / BTF / CO-RE 처리
        ↓
   bpf() System Call
        ↓
      Kernel
```

따라서 Chapter 4와 Chapter 5는 다음처럼 연결된다.

```text
Chapter 4

bpf()
→ Kernel이 제공하는 Low-level eBPF Interface


Chapter 5

libbpf
→ bpf()를 이용해 실제 eBPF Application의
  Load와 Attach 과정을 구성하는 User Space Library
```

libbpf 자체가 eBPF Runtime인 것은 아니다.

최종적으로 BPF Object를 생성하고 Program을 Kernel에 Load하는 것은 여전히 `bpf()` System Call이다.

---

## Map 정의 방식도 달라진다

BCC에서는 Map을 다음과 같은 Macro로 간단하게 정의할 수 있었다.

```c
BPF_HASH(config);
```

libbpf 기반 Program에서는 ELF와 BTF에서 Map 정보를 해석할 수 있도록 Map Definition을 보다 명시적으로 작성한다.

대표적인 형태는 다음과 같다.

```c
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, u32);
    __type(value, struct user_msg_t);
} config SEC(".maps");
```

각 항목은 Map의 속성을 표현한다.

```text
type
→ Map Type

max_entries
→ 최대 Entry 수

key
→ Key Type

value
→ Value Type

SEC(".maps")
→ ELF의 .maps Section에 배치
```

Chapter 4에서 `BPF_MAP_CREATE`를 이용해 Kernel에 Map Object가 생성된다는 것을 배웠다면, Chapter 5에서는 libbpf가 **ELF의 `.maps` Section과 BTF 정보를 읽어서 Map 생성에 필요한 정보를 얻은 뒤 실제 `bpf()` 호출로 연결한다**고 이해할 수 있다.

---

## `SEC()`는 단순히 Section을 나누는 Macro가 아니다

libbpf 기반 eBPF Code에서는 `SEC()` Macro가 자주 등장한다.

예를 들면 다음과 같다.

```c
SEC("ksyscall/execve")
int hello(...)
{
    ...
}
```

`SEC()`는 Function이나 Data를 ELF Object File의 특정 Section에 배치한다.

하지만 libbpf에서는 Section Name 자체가 의미를 가진다.

libbpf는 Section Name을 보고 Program의 종류와 Attach 방식을 판단할 수 있다.

```text
SEC("...")
    ↓
ELF Section Name
    ↓
libbpf
    ↓
Program Type / Attach 방식 판단
```

즉 Chapter 3에서 배운 ELF Section이 단순히 Binary 내부의 구역을 나누는 데서 끝나는 것이 아니라, libbpf가 eBPF Program을 해석하는 중요한 Metadata로도 사용된다.

---

## BPF Skeleton

libbpf가 많은 작업을 처리해주지만 User Space Code에는 여전히 반복적인 작업이 생긴다.

예를 들어 다음과 같다.

```text
BPF Object Open
Program 찾기
Map 찾기
Load
Attach
Map FD 접근
Cleanup
```

이 Boilerplate Code를 줄이기 위해 **BPF Skeleton**을 사용할 수 있다.

Skeleton은 Compile된 BPF Object File을 기반으로 자동 생성되는 Header다.

```bash
bpftool gen skeleton program.bpf.o > program.skel.h
```

전체 Build 흐름은 다음과 같다.

```text
program.bpf.c
      ↓
    Clang
      ↓
program.bpf.o
      ↓
bpftool gen skeleton
      ↓
program.skel.h
      ↓
User Space Program
```

Skeleton을 사용하면 해당 BPF Object에 맞는 Structure와 Helper Function이 생성되기 때문에 User Space에서 BPF Object 내부의 Program이나 Map을 직접 찾아다니는 코드를 줄일 수 있다.

결과적으로 User Space Program에서는 다음과 같은 Lifecycle에 집중할 수 있다.

```text
Open
 ↓
Load
 ↓
Attach
 ↓
Map / Buffer 사용
 ↓
Destroy
```

---

## BCC와 libbpf + CO-RE 비교

Chapter 2에서 사용한 BCC와 Chapter 5에서 다루는 libbpf + CO-RE를 다시 비교하면 차이가 더 명확하다.

| 구분                     | BCC                       | libbpf + CO-RE            |
| ------------------------ | ------------------------- | ------------------------- |
| eBPF Compile             | Target Machine의 Runtime  | Build Time                |
| 배포 형태                | Source Code 포함          | Compile된 BPF Object      |
| Target Compiler          | 필요                      | 일반적으로 불필요         |
| Kernel Header            | Target Kernel Header 사용 | BTF + `vmlinux.h` 활용    |
| Kernel 차이 대응         | Target에서 다시 Compile   | Load 시 Relocation        |
| Loader                   | BCC Framework             | libbpf                    |
| 반복적인 User Space 코드 | BCC가 추상화              | Skeleton으로 줄일 수 있음 |

핵심적인 차이는 다음과 같다.

```text
BCC

"실행할 Kernel에서
그 Kernel에 맞게 다시 Compile한다."


libbpf + CO-RE

"미리 Compile하고
실행할 Kernel에서 달라진 부분만 보정한다."
```

단순히 Python과 C의 차이가 아니라 **eBPF Application을 Build하고 배포하는 방식 자체가 달라진다.**

---

## Chapter 3부터 Chapter 5까지 연결하기

Chapter 5까지 공부하고 나면 앞에서 배운 내용이 하나의 흐름으로 연결된다.

```text
                    Build Time

eBPF C Source
      │
      │
      ├── vmlinux.h
      │    Kernel Type Definition
      │
      ▼
Clang / LLVM
      │
      ├── eBPF Bytecode
      ├── BTF
      └── CO-RE Relocation
      │
      ▼
ELF Object File (.bpf.o)

════════════════════════════════════

                    Runtime

ELF Object File
      │
      ▼
libbpf
      │
      ├── ELF Parsing
      ├── BTF Parsing
      ├── Runtime Kernel BTF 확인
      ├── CO-RE Relocation
      ├── Map 생성
      └── Program Load / Attach
      │
      ▼
bpf() System Call
      │
      ▼
Verifier
      │
      ▼
JIT Compilation
      │
      ▼
Kernel eBPF Program
      │
      ▼
Hook에 Attach
      │
      ▼
Event 발생 시 실행
```

Chapter 3에서 살펴본 ELF Object File, Chapter 4에서 살펴본 `bpf()` System Call, Chapter 5에서 배우는 BTF와 CO-RE, libbpf는 각각 독립적인 주제가 아니다.

전부 **하나의 eBPF Application이 Source Code에서 시작해 Target Kernel에서 실행되기까지의 서로 다른 단계**다.

---

## 정리

Chapter 5에서 가장 중요하게 이해한 것은 BTF 명령이나 libbpf API 자체가 아니라 **eBPF Application의 Portability를 어떤 방식으로 해결하는가**였다.

BCC에서는 Target Machine의 Kernel Header를 사용해 eBPF Program을 다시 Compile함으로써 Kernel 차이를 해결한다.

CO-RE는 접근 방식이 다르다.

```text
BTF
→ Kernel Type과 Structure Layout 정보를 표현

vmlinux.h
→ BTF를 기반으로 Kernel Type Definition 제공

Clang
→ Compile 과정에서 BTF와 CO-RE Relocation 정보 생성

ELF Object File
→ Bytecode와 BTF, Relocation Metadata 저장

libbpf
→ Runtime Kernel BTF와 Program의 정보를 비교

CO-RE Relocation
→ Target Kernel에 맞게 필요한 Instruction 수정

bpf()
→ 수정된 Program과 Map을 Kernel Object로 생성

Skeleton
→ User Space Loader 코드의 반복 작업을 줄여줌
```

그래서 CO-RE의 `Compile Once – Run Everywhere`는 **모든 Kernel이 동일하다고 가정하는 방식이 아니다.**

오히려 Kernel마다 Type Layout이 달라질 수 있다는 점을 전제로 하고, 그 차이를 BTF를 통해 파악한 뒤 Load 시점에 Relocation하는 방식이다.

Chapter 3부터 연결해서 한 문장으로 정리하면 다음과 같다.

- eBPF Source는 Clang을 통해 Bytecode와 BTF, CO-RE Relocation 정보가 포함된 ELF Object File로 Compile되고, Runtime에서는 libbpf가 Target Kernel의 BTF를 기준으로 필요한 Relocation을 적용한 뒤 `bpf()` System Call을 통해 Program과 Map을 Kernel에 Load하고 Hook에 Attach한다.

이 흐름을 이해하고 나니 실제 eBPF 프로젝트에서 자주 등장하는 `vmlinux.h`, `.bpf.c`, `SEC()`, `.maps`, `libbpf`, BPF Skeleton이 각각 따로 존재하는 요소가 아니라 **CO-RE 기반 eBPF Application의 Build와 Load Pipeline을 구성하는 요소들**이라는 점이 정리됐다.