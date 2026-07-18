---
layout: post
title: "[Learning eBPF] Chapter 2: eBPF’s “Hello World”"
date: 2026-07-18 17:20:00 +0900
categories: [eBPF, Linux, Kernel]
tags: [eBPF, BPF, BCC, BPFMap, Kprobe, PerfBuffer, RingBuffer, TailCall]
published: true
---

Chapter 1에서는 eBPF가 Linux Kernel 내부에서 사용자 정의 프로그램을 안전하게 실행할 수 있도록 하는 기술이라는 점을 살펴봤다. 또한 eBPF 프로그램은 독립적으로 계속 실행되는 Process가 아니라, 특정 Kernel Event에 Attach된 뒤 해당 Event가 발생할 때 실행되는 Event-driven Program이라는 점도 확인했다.

Chapter 2에서는 이 개념을 실제 코드로 연결한다. 다만 이 장의 핵심은 단순히 `"Hello World"`를 출력하는 방법을 배우는 것이 아니다. 더 중요한 것은 하나의 eBPF Application이 어떤 구성 요소로 이루어지고, User Space와 Kernel Space가 어떤 역할을 나누며, Kernel에서 발생한 데이터를 다시 User Space로 어떻게 전달하는지를 이해하는 것이다.

이 장의 예제는 다음 순서로 발전한다.

```text
가장 단순한 Trace 출력
→ 여러 Event 사이에 상태 저장
→ 구조화된 Event 전달
→ Program 내부 코드 분리
→ 여러 eBPF Program 연결
```

책에서는 이를 `hello.py`, `hello-map.py`, `hello-buffer.py`, `hello-tail.py`와 같은 예제로 보여준다. 각 예제는 서로 독립적인 샘플처럼 보이지만 실제로는 앞선 방식의 한계를 다음 방식이 보완하는 구조로 연결되어 있다.

```text
trace_pipe
→ 문자열 출력은 가능하지만 상태 저장과 구조화된 데이터 전달이 어렵다.

BPF Hash Map
→ 상태를 저장할 수 있지만 개별 Event의 순서와 상세 정보는 남지 않는다.

Perf Buffer
→ 개별 Event를 구조체 형태로 User Space에 전달할 수 있다.

Function Call
→ Program 내부 로직을 여러 함수로 나눌 수 있다.

Tail Call
→ 실행을 다른 eBPF Program으로 전환하여 Program을 여러 단위로 분리할 수 있다.
```

결국 Chapter 2에서 형성해야 할 기본 구조는 다음과 같다.

```text
eBPF Application
=
User Space Program
+
Kernel Space eBPF Program
+
Attachment Point
+
BPF Map 또는 Buffer
```

## 1. Hello World는 무엇을 보여주기 위한 예제인가?

일반적인 Programming Language에서 `Hello World`는 문법과 실행 환경이 정상적으로 동작하는지 확인하기 위한 가장 단순한 Program이다. 그러나 eBPF의 `Hello World`는 단순한 문자열 출력 이상의 의미를 가진다.

eBPF Program은 일반 Application처럼 실행 파일을 직접 실행한다고 동작하지 않는다. 먼저 User Space Program이 eBPF 코드를 준비하고, 이를 Kernel이 이해할 수 있는 Bytecode로 Compile한 뒤 Kernel에 Load해야 한다. 이후 특정 Kernel Event에 Attach해야 하며, 실제 Event가 발생해야 eBPF Program이 실행된다.

따라서 eBPF의 `Hello World`는 다음 전체 과정을 한 번에 보여주는 예제다.

```text
eBPF C 코드 작성
→ Compile
→ Kernel Load
→ Event에 Attach
→ Event 발생
→ eBPF Program 실행
→ 결과를 User Space에서 확인
```

Chapter 2에서는 이 과정을 간단히 구현하기 위해 BCC를 사용한다.

## 2. BCC가 맡아주는 역할

BCC는 BPF Compiler Collection의 약자로, eBPF Program을 작성하고 Compile·Load·Attach하는 과정을 단순화하는 Framework다. Python이나 Lua 같은 User Space 언어에서 eBPF Program을 관리할 수 있으며, Chapter 2에서는 Python API를 사용한다.

첫 번째 예제의 구조는 다음과 같다.

```python
#!/usr/bin/python
from bcc import BPF

program = r"""
int hello(void *ctx) {
    bpf_trace_printk("Hello World!");
    return 0;
}
"""

b = BPF(text=program)

syscall = b.get_syscall_fnname("execve")
b.attach_kprobe(event=syscall, fn_name="hello")

b.trace_print()
```

이 코드는 하나의 Python 파일에 들어 있지만 실제로는 두 개의 실행 영역으로 구분된다.

```text
User Space
→ Python 코드

Kernel Space
→ C로 작성된 hello() eBPF Program
```

Python Program은 eBPF C 코드를 Compile하고 Kernel에 Load하며, 특정 Kernel Event에 Attach하고, Kernel에서 출력된 결과를 읽는다. 반면 `hello()` 함수는 Kernel 내부에서 Event가 발생했을 때 실행된다.

### 2-1. Kernel에서 실행될 C 코드

```python
program = r"""
int hello(void *ctx) {
    bpf_trace_printk("Hello World!");
    return 0;
}
"""
```

Python의 `program` 변수에는 문자열 형태로 C 코드가 들어 있다. 이 코드는 Python Interpreter가 직접 실행하지 않는다. BCC가 Clang/LLVM을 사용하여 eBPF Bytecode로 Compile한 뒤 Kernel에 Load한다.

```c
int hello(void *ctx) {
    bpf_trace_printk("Hello World!");
    return 0;
}
```

`hello()`는 독립적으로 실행되는 Main Function이 아니다. 특정 Event에 연결된 Callback과 유사하게 동작한다. Event가 발생할 때 Kernel이 이 Program을 실행한다.

`ctx`는 현재 eBPF Program을 실행시킨 Event의 Context를 가리킨다. Context의 구조는 Program Type과 Attachment Point에 따라 달라진다. kprobe, tracepoint, raw tracepoint, XDP, TC, LSM은 서로 다른 Context를 제공한다. 현재 예제에서는 Context 값을 읽지 않기 때문에 `void *ctx`로 선언한다.

### 2-2. Compile과 Kernel Load

```python
b = BPF(text=program)
```

이 한 줄은 단순히 Python Object를 생성하는 코드처럼 보이지만 내부에서는 여러 단계가 진행된다.

```text
C Source Code
→ Clang/LLVM Compile
→ eBPF Instruction 생성
→ bpf() System Call 호출
→ eBPF Verifier 검증
→ Kernel에 Program Load
```

Verifier는 eBPF Program이 Kernel에서 안전하게 실행될 수 있는지 검사한다. 잘못된 Memory Access, 초기화되지 않은 값 사용, 허용되지 않은 Helper 호출, 종료되지 않을 가능성이 있는 실행 경로 등이 발견되면 Program Load가 거부된다.

Chapter 2에서는 BCC가 이 과정을 대부분 자동으로 처리한다. 따라서 사용자는 비교적 짧은 Python 코드만으로 eBPF Program을 실행할 수 있다. 이 편의성은 eBPF의 기본 실행 모델을 빠르게 학습하는 데 유리하지만, 실제로 어떤 Object File과 Bytecode가 만들어지고 `bpf()` System Call이 어떻게 호출되는지는 감춰진다. 이 부분은 다음 Chapter에서 더 자세히 다룬다.

## 3. eBPF Program은 언제 실행되는가?

eBPF Program은 Kernel에 Load된 것만으로 실행되지 않는다. Program이 언제 실행될지를 결정하는 Attachment Point가 필요하다.

첫 번째 예제에서는 새로운 Program을 실행할 때 사용되는 `execve()` System Call을 관찰한다.

```python
syscall = b.get_syscall_fnname("execve")
```

`execve()`는 현재 Process의 실행 이미지를 새로운 실행 파일로 교체하는 System Call이다.

예를 들어 Shell에서 `ls`를 실행하면 일반적으로 다음 흐름이 발생한다.

```text
Shell
→ 자식 Process 생성
→ execve("/usr/bin/ls", ...)
→ ls Program 실행
```

사용자가 알고 있는 System Call 이름은 `execve`지만 실제 Kernel 내부 Function 이름은 Architecture에 따라 달라질 수 있다. x86-64에서는 `__x64_sys_execve`와 같은 이름이 사용될 수 있다.

```python
b.get_syscall_fnname("execve")
```

BCC는 현재 Architecture에 맞는 실제 Kernel Function 이름을 찾아준다.

다음 코드는 그 Function의 진입 지점에 kprobe를 Attach한다.

```python
b.attach_kprobe(event=syscall, fn_name="hello")
```

각 인자의 의미는 다음과 같다.

```text
event=syscall
→ 관찰할 Kernel Function

fn_name="hello"
→ 해당 Function이 호출될 때 실행할 eBPF Program
```

결과적으로 다음 연결이 만들어진다.

```text
execve Kernel Function 진입
→ kprobe Trigger
→ hello() eBPF Program 실행
```

중요한 점은 Python Program이 `hello()`를 직접 호출하지 않는다는 것이다.

```text
Python Program
→ hello()를 Compile하고 Kernel에 Load
→ execve에 Attach

다른 Process
→ execve() 호출
→ Kernel이 hello() 실행
```

이것이 eBPF의 Event-driven 실행 모델이다.

## 4. Kernel에서 printf를 사용할 수 없는 이유

일반적인 User Space C Program에서는 `printf()`를 사용해 Terminal에 문자열을 출력할 수 있다. 그러나 eBPF Program은 Kernel Space에서 실행되므로 libc의 `printf()`를 사용할 수 없다.

```c
printf("Hello World");
```

`printf()`는 User Space Library Function이며 Kernel 내부 실행 환경에서는 사용할 수 없다. eBPF Program이 Kernel 기능과 상호작용하려면 Kernel이 제공하는 BPF Helper Function을 사용해야 한다.

첫 번째 예제에서는 다음 Helper를 사용한다.

```c
bpf_trace_printk("Hello World!");
```

`bpf_trace_printk()`는 문자열을 Kernel Tracing Buffer에 기록한다.

```text
hello() 실행
→ bpf_trace_printk()
→ Kernel Tracing Buffer
```

BPF Helper Function은 eBPF Program이 Kernel 기능에 접근하기 위한 제한된 API다. eBPF Program이 임의의 Kernel Function을 자유롭게 호출할 수 있도록 허용하면 Verifier가 안전성을 보장하기 어렵다. 따라서 Kernel은 검증 가능한 Helper Interface를 제공하고, Program Type별로 사용 가능한 Helper를 제한한다.

## 5. Trace 결과는 어떻게 User Space로 전달되는가?

Python 코드의 마지막 줄은 다음과 같다.

```python
b.trace_print()
```

이 코드는 Kernel Tracing Buffer에서 생성되는 출력을 읽어 화면에 표시한다. 내부적으로는 `trace_pipe`를 읽는다.

환경에 따라 다음 경로 중 하나에서 확인할 수 있다.

```text
/sys/kernel/debug/tracing/trace_pipe
```

또는

```text
/sys/kernel/tracing/trace_pipe
```

`trace_pipe`는 일반 Disk File이 아니라 Kernel이 제공하는 Pseudofile이다.

```text
일반 파일
→ Disk에 저장된 데이터를 읽음

trace_pipe
→ Kernel에서 새로 생성되는 Trace Event를 실시간으로 읽음
```

전체 실행 흐름은 다음과 같다.

```text
hello.py 실행
→ BCC가 hello() Compile
→ Kernel에 Load
→ execve kprobe에 Attach

다른 Process가 execve() 호출
→ kprobe Trigger
→ hello() 실행
→ bpf_trace_printk("Hello World!")
→ Kernel Trace Buffer 기록
→ Python이 trace_pipe를 읽어 출력
```

출력에는 사용자가 기록한 Message 외에도 Kernel Tracing Subsystem이 추가한 정보가 함께 나타난다.

```text
bash-5412    [001] .... 90432.904952: 0: bpf_trace_printk: Hello World
```

대략 다음 정보를 포함한다.

```text
bash-5412
→ Event를 발생시킨 Task 이름과 PID

[001]
→ Event가 실행된 CPU 번호

90432.904952
→ Timestamp

Hello World
→ eBPF Program이 기록한 Message
```

현재 eBPF Program이 직접 기록한 값은 `"Hello World"`뿐이다. Task 이름, PID, CPU 번호, Timestamp는 Tracing Infrastructure가 추가한 정보다.

## 6. 왜 실행하자마자 출력이 발생하는가?

`hello.py`를 실행한 뒤 별도의 명령을 입력하지 않았는데도 Trace가 출력될 수 있다. 이는 eBPF Program이 특정 Terminal만 관찰하는 것이 아니라 시스템 전체의 `execve()` Event를 관찰하기 때문이다.

```text
Shell
System Service
Monitoring Agent
Container Runtime
Cron Job
```

이들 중 어떤 Process라도 새로운 Program을 실행하면 `execve()`가 호출되고 `hello()`가 실행된다.

반대로 Shell Built-in Command는 `execve()`를 발생시키지 않을 수 있다.

```text
cd
export
alias
```

이 명령은 별도의 실행 파일을 실행하지 않고 현재 Shell Process 내부에서 처리되므로 `execve()`가 호출되지 않는다.

따라서 이 예제가 관찰하는 것은 사용자가 입력한 Command 자체가 아니라 Kernel에서 실제 발생한 `execve()` System Call이다.

## 7. trace_pipe만으로 충분하지 않은 이유

첫 번째 예제를 통해 eBPF Program이 Event에 Attach되어 실행되고, Kernel에서 User Space로 결과를 전달하는 기본 구조를 확인할 수 있다. 하지만 `bpf_trace_printk()`와 `trace_pipe`는 실제 eBPF Application의 주된 데이터 전달 방식으로 사용하기 어렵다.

### 7-1. 모든 Program이 하나의 전역 Trace 경로를 공유한다

```text
eBPF Program A ┐
eBPF Program B ├→ trace_pipe
eBPF Program C ┘
```

여러 Program이 동시에 Trace를 기록하면 출력이 한곳에 섞인다. Program별로 별도의 Event Stream을 만들기 어렵다.

### 7-2. 문자열 중심이라 구조화된 데이터를 전달하기 어렵다

실제 Observability Event에서는 하나의 문자열보다 여러 Field가 필요하다.

```text
PID
UID
Command
Source IP
Destination IP
Port
Protocol
Timestamp
Policy Verdict
```

이를 하나의 문자열로 조합하면 User Space에서 다시 Parsing해야 한다. 정수와 문자열의 Type 정보도 사라지고, Field가 추가되거나 순서가 변경되면 Parser도 함께 수정해야 한다.

### 7-3. 상태를 유지할 수 없다

`bpf_trace_printk()`는 Event가 발생할 때마다 Message를 출력할 뿐이다. 이전 Event에서 어떤 값이 있었는지 저장하지 않는다.

예를 들어 UID별 `execve()` 호출 횟수를 알고 싶다면 다음과 같은 누적 상태가 필요하다.

```text
UID 1000 → 15회
UID 0    → 4회
```

하지만 지역 변수는 Program 실행이 끝나면 유지되지 않는다.

```c
int hello(void *ctx) {
    int counter = 0;
    counter++;
    return 0;
}
```

이 코드는 Event가 발생할 때마다 `counter`가 다시 0으로 초기화된다.

```text
첫 번째 Event
→ 0에서 1

두 번째 Event
→ 다시 0에서 1
```

여러 Event 사이에서 값을 유지하려면 별도의 Kernel Data Structure가 필요하다. 여기에서 BPF Map이 등장한다.

## 8. BPF Map은 왜 필요한가?

BPF Map은 Kernel 내부에 존재하는 Data Structure로, eBPF Program 실행 사이에 상태를 유지할 수 있다. 또한 Kernel Space의 eBPF Program과 User Space Application이 함께 접근할 수 있다.

```text
Kernel eBPF Program
        ↕
      BPF Map
        ↕
User Space Application
```

BPF Map은 다음과 같은 역할을 수행한다.

```text
Event 사이의 상태 유지
Kernel에서 수집한 통계 저장
User Space와 eBPF Program 간 설정 공유
여러 eBPF Program 사이에서 데이터 공유
```

대부분의 BPF Map은 Key-Value 구조를 사용한다.

```text
Key → Value
```

어떤 값을 Key로 선택하는지에 따라 전혀 다른 통계를 만들 수 있다.

```text
UID → 실행 횟수
PID → File Open 횟수
IP Address → Packet 수
Syscall Number → 호출 횟수
```

Chapter 2의 두 번째 예제는 UID를 Key로 사용하여 사용자별 `execve()` 호출 횟수를 저장한다.

## 9. Hash Map으로 UID별 실행 횟수 저장하기

먼저 전체 실행 흐름을 보면 다음과 같다.

```text
execve() 발생
→ hello() 실행
→ 현재 UID 조회
→ UID를 Key로 Hash Map Lookup
→ 기존 Counter 읽기
→ Counter 1 증가
→ Hash Map Update
→ Python이 Map을 읽어 출력
```

Kernel Space 코드는 다음과 같다.

```c
BPF_HASH(counter_table);

int hello(void *ctx) {
    u64 uid;
    u64 counter = 0;
    u64 *p;

    uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;

    p = counter_table.lookup(&uid);

    if (p != 0) {
        counter = *p;
    }

    counter++;

    counter_table.update(&uid, &counter);

    return 0;
}
```

### 9-1. Hash Map 정의

```c
BPF_HASH(counter_table);
```

`BPF_HASH()`는 BCC가 제공하는 Macro다. `counter_table`이라는 이름의 Hash Type BPF Map을 생성한다.

Type을 명확히 작성하면 다음과 같다.

```c
BPF_HASH(counter_table, u64, u64);
```

```text
Key Type
→ u64 UID

Value Type
→ u64 Counter
```

`BPF_HASH()`는 표준 C 문법이 아니다. BCC가 Compile 전에 실제 BPF Map Definition으로 변환한다.

### 9-2. 현재 UID 추출

```c
uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
```

`bpf_get_current_uid_gid()`는 현재 Task의 UID와 GID를 하나의 64bit 값으로 반환한다.

```text
상위 32bit
→ GID

하위 32bit
→ UID
```

`0xFFFFFFFF`와 Bitwise AND 연산을 수행하면 하위 32bit만 남기 때문에 UID를 얻을 수 있다.

```text
[ GID ][ UID ]
AND
[  0  ][ 1...1 ]
=
[  0  ][ UID ]
```

### 9-3. 기존 Counter 조회

```c
p = counter_table.lookup(&uid);
```

현재 UID를 Key로 사용해 기존 Counter를 찾는다.

Map Lookup은 Value 자체가 아니라 Value가 저장된 위치를 가리키는 Pointer를 반환한다.

```text
Entry 존재
→ Value Pointer 반환

Entry 없음
→ null Pointer 반환
```

따라서 Pointer가 유효한지 확인해야 한다.

```c
if (p != 0) {
    counter = *p;
}
```

Entry가 존재하면 `*p`로 기존 Counter 값을 읽는다. Entry가 없으면 `counter`는 초기값 0을 유지한다.

### 9-4. Counter 증가와 Update

```c
counter++;
counter_table.update(&uid, &counter);
```

기존 Counter를 1 증가시킨 뒤 Map에 저장한다.

최초 Event에서는 다음 흐름으로 동작한다.

```text
UID 1000 Lookup
→ Entry 없음
→ counter = 0
→ counter = 1
→ 1000 → 1 저장
```

다음 Event에서는 기존 값을 읽어 증가시킨다.

```text
UID 1000 Lookup
→ 기존 Value 1
→ counter = 2
→ 1000 → 2 저장
```

### 9-5. BCC가 제공하는 C-like 문법

다음 코드는 일반 C Method 호출처럼 보인다.

```c
counter_table.lookup(&uid);
counter_table.update(&uid, &counter);
```

그러나 C에는 Object Method 문법이 없다. BCC가 편의를 위해 제공하는 표현이며 실제로는 Map Helper 호출로 변환된다.

개념적으로는 다음과 같다.

```c
bpf_map_lookup_elem(&counter_table, &uid);

bpf_map_update_elem(
    &counter_table,
    &uid,
    &counter,
    BPF_ANY
);
```

BCC는 이런 Low-level 세부사항을 감춰주기 때문에 코드가 짧고 읽기 쉬워진다.

## 10. User Space에서 Map 읽기

Python Program은 일정한 주기로 Kernel의 Hash Map을 읽는다.

```python
while True:
    sleep(2)

    s = ""

    for k, v in b["counter_table"].items():
        s += f"ID {k.value}: {v.value}\t"

    print(s)
```

BCC는 Kernel Map을 Python Object처럼 접근할 수 있게 해준다.

```python
b["counter_table"]
```

`.items()`를 호출하면 현재 Map에 저장된 모든 Key-Value Entry를 순회한다.

```text
k
→ UID

v
→ execve 호출 횟수
```

이 방식은 2초마다 Map의 현재 상태를 조회하는 Polling 방식이다.

```text
2초 대기
→ Map 전체 조회
→ 출력
→ 다시 대기
```

출력은 다음과 같이 나타날 수 있다.

```text
ID 501: 1
ID 501: 2
ID 501: 3    ID 0: 1
```

`sudo ls`와 같은 명령에서는 UID 501과 UID 0의 Counter가 각각 증가할 수 있다.

```text
일반 사용자
→ sudo Program 실행
→ UID 501 Counter 증가

sudo가 root 권한으로 ls 실행
→ UID 0 Counter 증가
```

이 예제가 세는 것은 사용자가 입력한 명령어 수가 아니라 정확히는 UID별 `execve()` System Call 횟수다.

## 11. Hash Map이면 충분하지 않을까?

Hash Map을 사용하면 여러 Event 사이의 상태를 저장할 수 있다. UID별 호출 횟수, Process별 Packet 수, IP별 Drop Counter처럼 누적 값을 관리하는 데 적합하다.

하지만 Hash Map에는 최종 상태만 남는다.

예를 들어 Counter가 5에서 8로 증가했다면 User Space는 세 번의 Event가 있었다는 사실은 알 수 있다.

```text
5 → 8
```

그러나 다음 정보는 알 수 없다.

```text
각 Event가 언제 발생했는가?
각 Event의 PID는 무엇인가?
어떤 Command가 실행되었는가?
Event가 발생한 순서는 무엇인가?
```

즉 Hash Map은 상태를 저장하는 데 적합하지만, 개별 Event의 상세 정보를 순서대로 전달하는 데는 적합하지 않다.

```text
Hash Map
→ 현재 상태와 누적 통계

Event Buffer
→ 개별 Event의 연속적인 전달
```

여기에서 Perf Buffer가 등장한다.

## 12. 상태와 Event는 무엇이 다른가?

상태와 Event를 구분하면 BPF Map과 Buffer의 역할을 이해하기 쉬워진다.

### 상태

```text
UID 1000의 execve 누적 횟수는 25회다.
```

상태는 특정 시점의 현재 값이다. 이전에 어떤 과정으로 25가 되었는지는 알 수 없다.

### Event

```text
12:01:01 UID 1000이 ls 실행
12:01:03 UID 1000이 curl 실행
12:01:05 UID 1000이 python 실행
```

Event는 개별 사건의 발생 순서와 Context를 보존한다.

Observability와 Security에서는 두 가지가 모두 필요하다.

```text
Metric
→ 상태와 누적 통계

Log 또는 Event Stream
→ 개별 사건의 상세 정보
```

Hash Map은 Metric에 가깝고, Perf Buffer나 BPF Ring Buffer는 Event Stream에 가깝다.

## 13. Ring Buffer의 기본 구조

Ring Buffer는 고정된 크기의 Memory 영역을 논리적으로 원형으로 사용하는 Buffer다.

일반적으로 다음 두 위치를 관리한다.

```text
Write Position
→ Producer가 다음 Event를 기록할 위치

Read Position
→ Consumer가 다음 Event를 읽을 위치
```

Producer가 Event를 기록하면 Write Position이 이동한다.

```text
Event A 기록
→ Write Position 이동
→ 다음 공간에 Event B 기록
```

Consumer가 Event를 읽으면 Read Position이 이동한다.

```text
Event A 읽기
→ Read Position 이동
→ Event B 읽기
```

Buffer 끝에 도달하면 처음 위치로 돌아가므로 논리적으로 원형처럼 동작한다.

```text
끝
→ 처음 위치로 Wrap Around
```

Producer가 Consumer보다 빠르면 Buffer가 가득 찰 수 있다.

```text
Kernel Event 생성 속도
>
User Space 소비 속도
```

이 경우 새 Event가 Drop될 수 있으므로 Buffer 크기와 User Space 처리 성능이 중요하다.

## 14. Perf Buffer로 구조화된 Event 전달하기

이번 예제에서는 문자열 하나를 출력하지 않고, 하나의 Event를 구조체로 정의하여 User Space로 전달한다.

수집할 정보는 다음과 같다.

```text
PID
UID
Command
Message
```

Kernel Space 코드는 다음과 같다.

```c
BPF_PERF_OUTPUT(output);

struct data_t {
    int pid;
    int uid;
    char command[16];
    char message[12];
};

int hello(void *ctx) {
    struct data_t data = {};
    char message[12] = "Hello World";

    data.pid = bpf_get_current_pid_tgid() >> 32;
    data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;

    bpf_get_current_comm(
        &data.command,
        sizeof(data.command)
    );

    bpf_probe_read_kernel(
        &data.message,
        sizeof(data.message),
        message
    );

    output.perf_submit(ctx, &data, sizeof(data));

    return 0;
}
```

전체 흐름은 다음과 같다.

```text
execve() 발생
→ hello() 실행
→ PID, UID, Command 수집
→ data_t Event 생성
→ Perf Buffer에 제출
→ User Space Callback 실행
```

### 14-1. Perf Output 생성

```c
BPF_PERF_OUTPUT(output);
```

BCC가 `output`이라는 이름의 Perf Buffer용 Map을 생성한다.

이전 Hash Map과 목적이 다르다.

```text
BPF_HASH
→ Key-Value 상태 저장

BPF_PERF_OUTPUT
→ 개별 Event 전달
```

### 14-2. Event Schema 정의

```c
struct data_t {
    int pid;
    int uid;
    char command[16];
    char message[12];
};
```

이 구조체는 하나의 Event가 어떤 Field로 구성되는지 정의한다.

```text
Event
=
PID
+
UID
+
Command
+
Message
```

문자열 한 줄을 전달하는 방식과 달리 각 Field의 위치와 Type이 유지된다.

예를 들면 다음과 같은 Event를 표현할 수 있다.

```text
{
    pid: 11654,
    uid: 1000,
    command: "node",
    message: "Hello World"
}
```

### 14-3. 구조체 초기화

```c
struct data_t data = {};
```

구조체 전체를 0으로 초기화한다.

eBPF Verifier는 초기화되지 않은 Stack Memory가 읽히거나 User Space로 전달되는 것을 허용하지 않는다. 초기화되지 않은 Memory에는 예측할 수 없는 값이나 기존 Kernel Data가 남아 있을 수 있기 때문이다.

### 14-4. PID와 TID

```c
data.pid = bpf_get_current_pid_tgid() >> 32;
```

`bpf_get_current_pid_tgid()`는 64bit 값을 반환한다.

```text
상위 32bit
→ TGID

하위 32bit
→ PID 또는 TID
```

Linux Kernel 내부에서 각 Thread는 개별 PID를 가지며, 같은 Process에 속한 Thread는 하나의 TGID를 공유한다. User Space에서 일반적으로 Process ID라고 보는 값은 TGID다.

따라서 32bit Right Shift를 수행하여 상위 32bit를 가져온다.

```c
>> 32
```

### 14-5. UID 조회

```c
data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
```

Hash Map 예제와 동일하게 하위 32bit를 추출하여 UID를 얻는다.

### 14-6. Command 이름 조회

```c
bpf_get_current_comm(
    &data.command,
    sizeof(data.command)
);
```

`bpf_get_current_comm()`은 현재 Task의 `comm` 값을 가져온다.

```text
bash
python3
curl
node
```

`comm`은 전체 Command Line이 아니다.

```bash
python3 server.py --port 8080
```

이 명령이 실행되어도 `comm`에는 일반적으로 다음 값만 들어간다.

```text
python3
```

전체 Argument까지 수집하려면 `execve()`의 Argument Pointer를 읽는 별도의 로직이 필요하다.

### 14-7. Message 복사

```c
bpf_probe_read_kernel(
    &data.message,
    sizeof(data.message),
    message
);
```

C에서는 배열을 다음처럼 직접 대입할 수 없다.

```c
data.message = message;
```

따라서 Helper를 사용해 Source Memory에서 Destination Buffer로 문자열을 복사한다.

### 14-8. Perf Buffer 제출

```c
output.perf_submit(ctx, &data, sizeof(data));
```

완성된 `data` 구조체를 Perf Buffer에 제출한다.

BCC의 `perf_submit()`은 표준 C Method가 아니다. BCC가 실제 `bpf_perf_event_output()` 계열 Helper 호출로 변환한다.

```text
data 구조체
→ Perf Buffer
→ User Space
```

## 15. User Space에서 Event 처리하기

User Space Python 코드는 다음과 같다.

```python
b = BPF(text=program)

syscall = b.get_syscall_fnname("execve")
b.attach_kprobe(event=syscall, fn_name="hello")

def print_event(cpu, data, size):
    data = b["output"].event(data)

    print(
        f"{data.pid} "
        f"{data.uid} "
        f"{data.command.decode()} "
        f"{data.message.decode()}"
    )

b["output"].open_perf_buffer(print_event)

while True:
    b.perf_buffer_poll()
```

### 15-1. Callback Function

```python
def print_event(cpu, data, size):
```

Perf Buffer에서 Event를 읽었을 때 호출할 Callback Function이다.

```text
cpu
→ Event가 발생한 CPU

data
→ Raw Event Data

size
→ Event 크기
```

Raw Data를 Kernel에서 정의한 `struct data_t` 형태로 해석한다.

```python
data = b["output"].event(data)
```

이후 각 Field에 접근할 수 있다.

```python
data.pid
data.uid
data.command
data.message
```

C의 `char[]`는 Python에서 Byte String으로 전달되므로 `.decode()`를 사용한다.

### 15-2. Callback 등록

```python
b["output"].open_perf_buffer(print_event)
```

`output` Buffer에 Event가 도착하면 `print_event()`를 호출하도록 연결한다.

```text
Event 도착
→ print_event() 실행
```

### 15-3. Event Polling

```python
while True:
    b.perf_buffer_poll()
```

User Space Program은 Perf Buffer에서 Event를 기다린다. Event가 있으면 읽어서 Callback에 전달한다.

앞선 Hash Map Polling과는 의미가 다르다.

```text
Hash Map Polling
→ 일정 주기로 현재 상태 조회

Perf Buffer Polling
→ Buffer에 쌓인 개별 Event 소비
```

## 16. trace_pipe, Hash Map, Perf Buffer 비교

Chapter 2의 앞부분은 Kernel에서 User Space로 데이터를 전달하는 방식을 단계적으로 확장한다.

| 방식         | 주요 목적             | 장점                 | 한계                             |
| ------------ | --------------------- | -------------------- | -------------------------------- |
| `trace_pipe` | Debug Message 출력    | 가장 단순함          | 전역 출력, 문자열 중심           |
| Hash Map     | 상태와 누적 통계 저장 | Event 사이 상태 유지 | 개별 Event 순서와 상세 정보 손실 |
| Perf Buffer  | 구조화된 Event 전달   | Event 단위 처리 가능 | Buffer Full 시 Event Drop 가능   |

각 방식은 서로 완전히 대체 관계가 아니다. 목적이 다르다.

```text
Debugging
→ trace_pipe

Metric과 현재 상태
→ BPF Map

Log와 Event Stream
→ Perf Buffer 또는 BPF Ring Buffer
```

실제 eBPF Application은 여러 Map과 Buffer를 함께 사용할 수 있다.

```text
Configuration Map
Counter Map
Process Cache Map
Event Ring Buffer
```

## 17. Perf Buffer와 BPF Ring Buffer는 같은가?

책에서는 Perf Buffer를 Ring Buffer처럼 설명하지만, `BPF_PERF_OUTPUT`이 사용하는 Perf Buffer와 Kernel 5.8에서 추가된 BPF Ring Buffer는 구분해야 한다.

### Perf Buffer

```c
BPF_PERF_OUTPUT(output);
```

Perf Event Subsystem을 기반으로 하며 일반적으로 CPU별 Buffer를 사용한다.

```text
CPU 0 → Buffer 0
CPU 1 → Buffer 1
CPU 2 → Buffer 2
```

CPU별 Buffer는 CPU 간 Lock 경쟁을 줄이는 데 유리하지만 여러 CPU에서 발생한 Event의 전역 순서를 재구성하기 어렵다.

### BPF Ring Buffer

BPF 전용 Map Type이다.

```text
BPF_MAP_TYPE_RINGBUF
```

여러 CPU가 하나의 공유 Buffer에 Event를 기록한다.

```text
CPU 0 ┐
CPU 1 ├→ Shared Ring Buffer
CPU 2 ┘
```

CPU 사이의 Event 순서를 유지하기 유리하고, CPU마다 별도 Buffer를 할당하지 않아 Memory 활용 측면에서도 장점이 있다.

| 구분        | Perf Buffer              | BPF Ring Buffer  |
| ----------- | ------------------------ | ---------------- |
| 기반        | Perf Event Subsystem     | BPF 전용 Map     |
| 구조        | CPU별 Buffer             | 공유 Buffer      |
| CPU 간 순서 | 재구성이 어려움          | 유지에 유리      |
| Memory 할당 | CPU별                    | 하나의 공유 영역 |
| Kernel 지원 | 오래된 Kernel에서도 사용 | Kernel 5.8 이상  |

Chapter 2의 예제는 BCC의 `BPF_PERF_OUTPUT`을 사용하므로 정확히는 Perf Buffer 예제다.

## 18. eBPF Program 내부 로직이 커지면 어떻게 할까?

초기 예제에서는 `hello()` 함수 하나만으로 충분했다.

```c
int hello(void *ctx) {
    ...
}
```

하지만 실제 eBPF Program은 다음과 같은 여러 작업을 수행할 수 있다.

```text
Context Parsing
Filter 조건 확인
Map Lookup
Policy 평가
Event 생성
통계 Update
```

모든 로직을 하나의 Function에 넣으면 코드가 길어지고 재사용하기 어렵다.

```text
hello()
→ 50줄
→ 200줄
→ 500줄
```

일반적인 C Program에서는 Function을 분리하여 해결한다.

```c
parse_context();
check_filter();
update_stats();
submit_event();
```

eBPF에서도 Function Call이 필요하지만 초기 eBPF에서는 일반적인 사용자 정의 Function Call을 사용할 수 없었다.

## 19. 초기 eBPF에서 Function Call이 제한된 이유

eBPF Program은 Kernel에 Load되기 전에 Verifier 검증을 받아야 한다.

Verifier는 다음을 분석한다.

```text
모든 실행 경로가 종료되는가?
Memory Access가 안전한가?
Stack 사용량이 제한을 넘지 않는가?
Pointer가 유효한가?
```

일반 Function Call이 있으면 Control Flow와 Stack Frame 분석이 복잡해진다.

```text
Caller
→ Callee
→ 또 다른 Function
→ Return
```

초기 eBPF는 Verifier 구현을 단순화하기 위해 사용자 정의 Function Call을 제한하고, Kernel이 제공하는 Helper Function만 호출할 수 있도록 했다.

따라서 개발자는 Function을 `inline`해야 했다.

```c
static __always_inline int helper_function(...) {
    ...
}
```

### 19-1. Inline이란?

일반 Function Call은 실행 중 다른 Function의 Instruction으로 이동한 뒤 다시 호출 지점으로 돌아온다.

```text
Caller
→ Callee
→ Return
→ Caller 계속
```

Inline은 Compiler가 함수 내용을 호출 위치에 직접 복사한다.

```text
Caller 내부에 Callee Instruction 삽입
```

따라서 Runtime Function Call이 발생하지 않는다.

### 19-2. `static __always_inline`

```c
static __always_inline int helper_function(...) {
    ...
}
```

`static`은 해당 Function의 Linkage 범위를 현재 Compilation Unit으로 제한한다.

`__always_inline`은 Compiler에게 이 Function을 반드시 Inline하도록 요청한다.

이 방식은 Function 단위로 코드를 작성하면서도 최종 eBPF Instruction에는 별도의 Function Call을 만들지 않는다.

다만 같은 Function을 여러 위치에서 호출하면 Instruction이 반복 복사되므로 Program 크기가 증가할 수 있다.

## 20. BPF-to-BPF Function Call

Kernel 4.16과 LLVM 6 이후에는 eBPF Program 내부의 사용자 정의 Function Call, 즉 BPF-to-BPF Call을 사용할 수 있게 되었다.

```text
eBPF Main Program
→ eBPF Subprogram
→ Return
```

Helper Function과는 구분해야 한다.

```text
BPF Helper Function
→ Kernel이 제공하는 API

BPF-to-BPF Function
→ 개발자가 작성한 Subprogram
```

Function Call은 Program 내부 로직을 분리하고 공통 코드를 재사용하는 데 적합하다.

```text
Program
├── parse_header()
├── check_policy()
└── update_counter()
```

그러나 Function Call은 동일한 eBPF Program 내부에서 실행 흐름을 잠시 다른 Subprogram으로 이동한 뒤 다시 돌아오는 구조다.

더 큰 기능 단위를 여러 eBPF Program으로 나누고, Runtime 조건에 따라 다른 Program으로 실행을 전환하려면 Tail Call이 필요하다.

## 21. Function Call과 Tail Call은 무엇이 다른가?

Function Call의 구조는 다음과 같다.

```text
Program A
→ Function B 호출
→ Function B 실행
→ Program A로 Return
```

Tail Call의 구조는 다르다.

```text
Program A
→ Program B로 실행 전환
→ Program A로 돌아오지 않음
```

Tail Call은 Function을 호출하는 것이 아니라 현재 eBPF Program의 실행을 다른 eBPF Program으로 교체한다.

개념적으로 User Space의 `execve()`와 비슷한 면이 있다.

```text
현재 Program
→ 새로운 Program으로 실행 이미지 교체
→ 이전 Program으로 Return하지 않음
```

다만 eBPF Tail Call과 `execve()`가 같은 구현이라는 의미는 아니다. Return하지 않고 다른 Program으로 실행을 전환한다는 실행 모델이 유사하다는 의미다.

## 22. Tail Call은 왜 필요한가?

Tail Call은 큰 eBPF Application을 여러 독립 Program으로 분리하는 데 사용한다.

예를 들어 모든 System Call 진입을 하나의 Program에서 처리한다고 가정할 수 있다.

```text
sys_enter
→ execve 처리
→ openat 처리
→ timer 처리
→ 네트워크 처리
→ 기타 처리
```

모든 로직을 하나의 Program에 넣으면 코드가 복잡해지고 Program Size와 Verifier Complexity가 커진다.

Tail Call을 사용하면 Dispatcher와 실제 처리 Program을 분리할 수 있다.

```text
sys_enter
→ Dispatcher

Dispatcher
├── execve Program
├── timer Program
├── ignore Program
└── default 처리
```

Dispatcher는 System Call Number를 읽고 어떤 Program으로 실행을 전환할지만 결정한다.

## 23. Program Array Map

Tail Call 대상 Program은 Program Array Map에 저장한다.

BCC에서는 다음과 같이 정의할 수 있다.

```c
BPF_PROG_ARRAY(syscall, 300);
```

`syscall`이라는 이름의 Program Array를 만들며 최대 300개의 Entry를 저장할 수 있다.

일반 Hash Map과 달리 Value는 일반 Data가 아니라 eBPF Program을 가리킨다.

```text
Key
→ Index 또는 Opcode

Value
→ eBPF Program
```

예를 들면 다음과 같다.

```text
59  → hello_exec
222 → hello_timer
223 → hello_timer
21  → ignore_opcode
```

User Space Program이 각 Index에 실제 eBPF Program File Descriptor를 등록한다.

## 24. Raw Tracepoint Dispatcher

Tail Call 예제는 모든 System Call 진입 시 발생하는 `sys_enter` Raw Tracepoint에 Attach된다.

```c
int hello(struct bpf_raw_tracepoint_args *ctx) {
    int opcode = ctx->args[1];

    syscall.call(ctx, opcode);

    bpf_trace_printk("Another syscall: %d", opcode);

    return 0;
}
```

`ctx->args[1]`에는 System Call Number가 들어 있다. 이 값은 Architecture에 따라 달라질 수 있다.

Dispatcher는 다음 코드를 통해 Program Array의 해당 Index로 Tail Call을 시도한다.

```c
syscall.call(ctx, opcode);
```

BCC는 이를 실제 `bpf_tail_call()` Helper 호출로 변환한다.

개념적으로는 다음과 같다.

```c
bpf_tail_call(ctx, &syscall, opcode);
```

## 25. Tail Call 성공과 실패

Tail Call이 성공하면 현재 Program으로 돌아오지 않는다.

```text
hello()
→ Program Array Lookup
→ hello_exec()로 Tail Call
→ hello_exec() 실행
→ 종료
```

따라서 Tail Call 다음 줄은 실행되지 않는다.

```c
bpf_trace_printk("Another syscall: %d", opcode);
```

반대로 Program Array의 해당 Index에 Program이 등록되어 있지 않거나 Tail Call 제한에 도달한 경우 Tail Call은 실패한다.

```text
hello()
→ Tail Call 실패
→ 다음 Instruction 계속 실행
→ "Another syscall" 출력
```

이 동작을 이용하면 등록되지 않은 System Call에 대한 Default 처리도 구현할 수 있다.

## 26. Tail Call 대상 Program

예제에서는 여러 Tail Call Target을 정의한다.

```c
int hello_exec(void *ctx) {
    bpf_trace_printk("Executing a program");
    return 0;
}
```

```c
int hello_timer(void *ctx) {
    bpf_trace_printk("Creating or deleting a timer");
    return 0;
}
```

```c
int ignore_opcode(void *ctx) {
    return 0;
}
```

`ignore_opcode()`로 Tail Call하면 아무 Message도 출력하지 않고 종료한다. 특정 System Call의 Noise를 제거하는 Filter처럼 사용할 수 있다.

하나의 Target Program을 여러 Index에서 공유할 수도 있다.

```text
222 ┐
223 ├→ hello_timer
224 ┤
225 ┤
226 ┘
```

System Call Number는 다르지만 같은 Category로 처리하고 싶을 때 유용하다.

## 27. User Space에서 Program Array 구성하기

User Space에서는 각 Tail Call Target을 개별 eBPF Program으로 Load한다.

```python
hello_exec = b.load_func(
    "hello_exec",
    BPF.RAW_TRACEPOINT
)
```

`load_func()`는 Kernel에 Load된 Program을 가리키는 File Descriptor를 반환한다.

```text
File Descriptor
→ Kernel의 eBPF Program Object 참조
```

Program Array를 가져온 뒤 Index에 Program을 등록한다.

```python
syscall = b.get_table("syscall")

syscall[c_int(59)] = c_int(hello_exec.fd)
```

개념적으로 다음 Routing Table을 구성한다.

```text
59 → hello_exec
```

Timer 관련 System Call 여러 개는 같은 Program에 연결할 수 있다.

```python
syscall[c_int(222)] = c_int(hello_timer.fd)
syscall[c_int(223)] = c_int(hello_timer.fd)
syscall[c_int(224)] = c_int(hello_timer.fd)
```

이 방식의 중요한 특징은 Dispatcher Code를 변경하지 않고 User Space에서 Routing을 변경할 수 있다는 점이다.

```text
기존
59 → hello_exec_v1

변경
59 → hello_exec_v2
```

Program Array의 Entry만 교체하면 Dispatcher는 그대로 유지할 수 있다.

## 28. Tail Call은 동적 Dispatch다

Tail Call은 단순히 Program을 여러 개 연결하는 기능이 아니라 Runtime Dispatch Mechanism으로 볼 수 있다.

```text
Input
→ Index 계산
→ Program Array Lookup
→ 해당 Program으로 실행 전환
```

일반 Programming의 Function Pointer Table과 비슷한 역할을 한다.

```text
Index
→ Function 또는 Program 선택
```

다만 eBPF에서는 Program Array에 독립적으로 Load된 eBPF Program을 저장하며, `bpf_tail_call()`을 통해 실행을 전환한다.

이 구조는 다음과 같은 경우에 유용하다.

```text
Protocol별 Packet 처리
System Call별 Security Logic
Policy Version 교체
Feature별 Program 분리
```

## 29. Tail Call의 제한

Tail Call은 무제한으로 연결할 수 없다. Kernel은 Tail Call Chain 길이를 제한하여 무한 실행을 방지한다.

개념적으로 다음과 같은 Chain을 만들 수 있다.

```text
Program A
→ Program B
→ Program C
→ Program D
```

하지만 일정 횟수를 초과하면 다음 Tail Call이 실패한다.

이 제한이 필요한 이유는 다음과 같다.

```text
무한 Tail Call Loop 방지
Execution Time 제한
Kernel 안정성 유지
```

또한 각 eBPF Program은 독립적으로 Verifier 검증을 받는다. 따라서 하나의 거대한 Program을 작성하는 대신 여러 Program으로 나누어 검증할 수 있다.

하지만 이것이 전체 실행에서 무제한 Instruction을 사용할 수 있다는 의미는 아니다. Tail Call Chain과 Program Complexity 모두 Kernel 제한을 받는다.

## 30. Function Call과 Tail Call 비교

| 구분           | Function Call             | Tail Call                            |
| -------------- | ------------------------- | ------------------------------------ |
| 호출 대상      | 같은 Program의 Subprogram | 독립적으로 Load된 eBPF Program       |
| 호출 후 Return | 호출자에게 돌아옴         | 돌아오지 않음                        |
| 선택 방식      | Compile 시 코드에 정의    | Program Array Index로 Runtime 선택   |
| 주요 목적      | 코드 분리와 재사용        | Program Chain과 Dynamic Dispatch     |
| 실패 시        | 일반적인 호출 구조        | Tail Call 다음 Instruction 계속 실행 |
| 관리 단위      | 하나의 eBPF Program       | 여러 eBPF Program                    |

Function Call은 하나의 Program 내부를 구조화하는 방법이고, Tail Call은 Application을 여러 Program 단위로 분리하는 방법이다.

```text
Function Call
→ Program 내부 Modularization

Tail Call
→ Program 간 Modularization
```

## 31. Chapter 2의 예제는 어떻게 연결되는가?

Chapter 2의 예제는 각각 다른 기능을 소개하지만 하나의 흐름으로 연결된다.

### 첫 번째 단계: Event에 Attach하여 실행하기

```text
execve
→ kprobe
→ hello()
```

여기서 eBPF Program이 Event-driven 방식으로 실행된다는 점을 확인한다.

### 두 번째 단계: 결과를 Trace로 확인하기

```text
hello()
→ bpf_trace_printk()
→ trace_pipe
```

가장 단순한 Kernel-to-User Space 출력 경로를 확인한다.

### 세 번째 단계: Event 사이에 상태 유지하기

```text
execve
→ UID 조회
→ Hash Map Counter Update
```

BPF Map이 eBPF Program 실행 사이에 상태를 유지하며 User Space와 공유할 수 있다는 점을 확인한다.

### 네 번째 단계: 개별 Event 전달하기

```text
execve
→ Event 구조체 생성
→ Perf Buffer
→ Python Callback
```

누적 상태가 아니라 개별 Event를 구조화된 형태로 전달한다.

### 다섯 번째 단계: Program 내부 코드 분리하기

```text
Main Program
→ BPF-to-BPF Function
→ Main Program으로 Return
```

복잡한 로직을 Subprogram으로 분리한다.

### 여섯 번째 단계: 여러 Program 연결하기

```text
Dispatcher
→ Program Array
→ Tail Call Target
```

큰 Application을 여러 eBPF Program으로 분리하고 Runtime 조건에 따라 실행 대상을 선택한다.

## 32. Chapter 2에서 이해해야 할 eBPF Application 구조

Chapter 2의 예제를 모두 합치면 하나의 eBPF Application은 다음과 같이 구성된다.

```text
+--------------------------------------------------+
|                    User Space                    |
|--------------------------------------------------|
| Loader / Controller                              |
|                                                  |
| - eBPF C Source 준비                             |
| - Compile                                        |
| - Kernel Load                                    |
| - Attachment                                     |
| - Map 설정                                       |
| - Buffer Polling                                 |
| - Event Processing                               |
+--------------------------+-----------------------+
                           |
                           | bpf() System Call
                           | Map Access
                           | Buffer Read
                           v
+--------------------------------------------------+
|                   Kernel Space                   |
|--------------------------------------------------|
| Attachment Point                                 |
|                                                  |
| kprobe / tracepoint / raw tracepoint              |
|                                                  |
|                Event 발생                        |
|                     |                            |
|                     v                            |
|                eBPF Program                      |
|                     |                            |
|          +----------+----------+                 |
|          |                     |                 |
|          v                     v                 |
|       BPF Map             Perf/Ring Buffer       |
|    상태와 설정 저장          Event 전달           |
+--------------------------------------------------+
```

이를 더 간단히 정리하면 다음과 같다.

```text
User Space
→ Program을 준비하고 관리

Kernel Space
→ Event가 발생할 때 Program 실행

Attachment Point
→ Program 실행 시점을 결정

BPF Map
→ 상태와 설정 공유

Perf/Ring Buffer
→ 개별 Event 전달

Program Array
→ Tail Call 대상 관리
```

## 33. BCC가 숨겨준 것

Chapter 2에서는 BCC 덕분에 적은 코드로 eBPF Application을 실행할 수 있었다.

```python
b = BPF(text=program)
```

이 한 줄 뒤에는 다음 과정이 숨어 있다.

```text
Compile
→ Bytecode 생성
→ Map 생성
→ bpf() System Call
→ Verifier
→ Kernel Load
```

다음 코드도 마찬가지다.

```python
b.attach_kprobe(...)
```

내부적으로는 Kernel Function을 찾고, Probe를 생성하고, eBPF Program과 Event를 연결하는 작업이 필요하다.

Map 접근도 BCC가 추상화한다.

```python
b["counter_table"].items()
```

실제로는 Map File Descriptor를 사용하여 Key를 순회하고 Value를 조회하는 `bpf()` System Call이 필요하다.

BCC는 학습과 Prototype 작성에 편리하지만, 그만큼 eBPF Program의 Build·Load·Attach 과정이 감춰진다.

## 34. Chapter 3으로 이어지는 질문

Chapter 2를 끝내고 나면 다음 질문이 남는다.

```text
C 코드는 정확히 어떤 eBPF Instruction으로 변환되는가?
Kernel은 Program을 어떻게 Load하는가?
Verifier는 어떤 방식으로 안전성을 검사하는가?
JIT Compiler는 무엇을 하는가?
BCC 없이 eBPF Application을 작성하려면 무엇이 필요한가?
```

Chapter 2에서는 BCC가 이러한 과정을 대신 처리했다. 다음 Chapter에서는 같은 `Hello World`를 다른 방식으로 작성하면서 BCC가 감춰준 내부 과정을 더 자세히 살펴본다.

## 35. 정리

Chapter 2에서 중요한 것은 `Hello World`라는 문자열이 아니다. 이 장은 가장 단순한 Trace 출력 예제에서 시작하여 하나의 eBPF Application이 발전하는 과정을 보여준다.

```text
Event 발생
→ eBPF Program 실행
→ Kernel Context 수집
→ 상태 저장 또는 Event 생성
→ User Space로 전달
```

첫 번째 예제에서는 `execve()` Kernel Function에 kprobe를 Attach하고 `bpf_trace_printk()`로 Message를 출력했다. 이를 통해 eBPF Program이 Kernel Event에 의해 실행되는 구조와 `trace_pipe`를 통한 가장 단순한 결과 전달 방식을 확인했다.

다음 예제에서는 BPF Hash Map을 사용하여 UID별 `execve()` 호출 횟수를 저장했다. 이를 통해 eBPF Program의 지역 변수가 아니라 Kernel Map을 사용해야 Event 사이에 상태를 유지할 수 있다는 점을 확인했다.

Perf Buffer 예제에서는 PID, UID, Command, Message를 구조체로 정의하고 개별 Event를 User Space Callback으로 전달했다. Hash Map이 현재 상태를 저장한다면 Perf Buffer는 발생한 Event를 순서대로 전달하는 역할을 한다.

마지막으로 Function Call과 Tail Call을 통해 eBPF Logic을 구조화하는 방법을 확인했다. Function Call은 하나의 Program 내부에서 코드를 분리하고 재사용하는 방식이며, Tail Call은 Program Array를 사용하여 독립적으로 Load된 다른 eBPF Program으로 실행을 전환하는 방식이다.

Chapter 2를 통해 만들어야 할 최종 Mental Model은 다음과 같다.

```text
eBPF Application
=
User Space Loader
+
Kernel eBPF Program
+
Attachment Point
+
BPF Map
+
Event Buffer
```

User Space Program은 eBPF 코드를 Compile하고 Kernel에 Load하며, Event에 Attach하고, Map과 Buffer를 통해 결과를 읽는다. Kernel의 eBPF Program은 특정 Event가 발생했을 때만 실행되며, Helper Function을 통해 Context를 수집하고 BPF Map 또는 Buffer를 통해 User Space와 데이터를 공유한다.

결국 eBPF는 Kernel에서 실행되는 C 코드 하나가 아니라, User Space와 Kernel Space가 역할을 나누어 동작하는 하나의 Application Architecture다.
