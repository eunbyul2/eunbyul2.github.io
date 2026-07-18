---
layout: post
title: "[Learning eBPF] Chapter 2: eBPF’s “Hello World”"
date: 2026-07-18 17:20:00 +0900
categories: [eBPF, Linux, Kernel]
tags: [eBPF, BPF, BCC, BPFMap, Kprobe, PerfBuffer, RingBuffer, TailCall]
published: true
---

Chapter 1에서는 eBPF가 Linux Kernel 내부에서 사용자 정의 로직을 실행하기 위한 Runtime이며, 특정 Kernel Event에 연결되어 Event-driven 방식으로 동작한다는 점을 살펴봤다. Chapter 2에서는 이 개념을 실제 코드와 실행 흐름으로 연결한다. 제목은 `Hello World`지만, 이 장의 핵심은 문자열을 한 번 출력하는 데 있지 않다. eBPF Program이 어떤 과정을 거쳐 Kernel에 Load되고, 특정 Hook에 Attach되며, Event가 발생했을 때 어떻게 실행되고, Kernel에서 수집한 데이터를 User Space로 전달하는지를 처음부터 끝까지 확인하는 것이 목적이다.

이 장의 예제는 단순한 출력에서 시작해 점차 실제 eBPF Application의 구조에 가까워진다. 처음에는 `bpf_trace_printk()`를 이용해 문자열을 출력하고, 그 다음에는 BPF Map을 이용해 Event 사이에서 상태를 유지한다. 이후 Perf Buffer를 이용해 구조화된 Event를 User Space로 전달하고, 마지막에는 Function Call과 Tail Call을 통해 Program을 분리하는 방법을 살펴본다.

```text
Hello World 출력
        ↓
Kernel Event에 Program Attach
        ↓
Kernel 내부 상태 저장
        ↓
Kernel → User Space Event 전달
        ↓
Program 구조화
```

Chapter 2를 이해하면 eBPF Application이 단일 Kernel Program만으로 구성되는 것이 아니라, User Space Loader와 Kernel Space Program, BPF Map, Event Buffer가 함께 하나의 시스템을 구성한다는 점을 알 수 있다.

## eBPF의 Hello World가 일반적인 Hello World와 다른 이유

일반적인 C나 Python Program은 실행 파일을 직접 실행하면서 시작된다. `main()` 함수가 호출되고, 코드가 순서대로 실행된 뒤 종료된다. 하지만 eBPF Program은 독립적인 Process가 아니다. Kernel 내부에 Load된 뒤 특정 Event에 연결되어 대기하고, 해당 Event가 발생했을 때 Kernel에 의해 호출된다.

따라서 eBPF에서 Hello World는 다음과 같은 전체 수명주기를 확인하는 예제다.

```text
User Space에서 eBPF Source 준비
        ↓
Clang/LLVM을 통한 Compile
        ↓
eBPF Bytecode 생성
        ↓
bpf() System Call을 통해 Kernel Load
        ↓
Verifier 검사
        ↓
Kernel Hook에 Attach
        ↓
Event 발생
        ↓
eBPF Program 실행
```

중요한 점은 Python Code가 eBPF Function을 직접 호출하지 않는다는 것이다. Python은 Program을 준비하고 Kernel에 등록하는 역할을 수행하며, 실제 실행 시점은 Kernel Event가 결정한다.

## BCC가 단순화하는 eBPF 개발 과정

Chapter 2의 예제는 BCC(BPF Compiler Collection)를 사용한다. BCC는 eBPF Program을 C 형태로 작성하고, User Space Loader를 Python으로 구성할 수 있도록 지원하는 Framework다.

다음 한 줄은 매우 단순해 보인다.

```python
b = BPF(text=program)
```

하지만 내부에서는 C Source를 Clang과 LLVM으로 Compile하고, eBPF Bytecode를 생성하고, `bpf()` System Call을 이용해 Program을 Kernel에 Load한 뒤, Verifier 검사를 통과시키는 과정이 수행된다.

```text
C Source
   ↓
Clang Frontend
   ↓
LLVM IR
   ↓
eBPF Bytecode
   ↓
bpf() System Call
   ↓
Verifier
   ↓
Kernel에 Program 등록
```

BCC를 사용하면 개발자는 ELF Section, Relocation, Map File Descriptor, Program Load Option 등을 직접 처리하지 않고도 eBPF Program을 실행할 수 있다. 대신 BCC가 제공하는 Macro와 Python API에 의존하게 된다. Chapter 2는 이러한 추상화를 이용해 eBPF의 핵심 실행 구조에 집중한다.

## User Space와 Kernel Space의 역할

BCC 예제는 하나의 Python File에 C Code가 문자열로 포함되어 있지만, 실제 실행 영역은 명확하게 나뉜다.

```text
User Space
- eBPF Source 보관
- Compile 요청
- Kernel Load
- Hook Attach
- Map 조회
- Event Polling
- 결과 출력

Kernel Space
- Event 발생 감지
- eBPF Program 실행
- Context 정보 수집
- Map Update
- Buffer에 Event Submit
```

User Space Program은 eBPF Program의 Loader이자 Controller다. Kernel Space Program은 Event가 발생했을 때 짧게 실행되는 Handler다. 이 두 영역 사이에서 BPF Map과 Perf Buffer가 데이터 교환 통로로 사용된다.

## 첫 번째 Hello World Program

가장 단순한 eBPF Program은 다음과 같다.

```c
int hello(void *ctx) {
    bpf_trace_printk("Hello World");
    return 0;
}
```

이 Function은 일반적인 C Program의 `main()`처럼 자동 실행되지 않는다. User Space에서 특정 Kernel Event에 Attach해야 한다.

```python
b = BPF(text=program)
syscall = b.get_syscall_fnname("execve")
b.attach_kprobe(event=syscall, fn_name="hello")
b.trace_print()
```

`get_syscall_fnname("execve")`는 현재 Architecture와 Kernel에서 `execve` System Call을 구현하는 실제 Kernel Symbol을 찾는다. Architecture나 Kernel Version에 따라 Symbol Name이 다를 수 있기 때문에 직접 문자열을 고정하기보다 BCC API를 이용한다.

`attach_kprobe()`는 `execve` 구현 함수의 진입점에 Kprobe를 등록하고, 해당 지점에서 `hello()` eBPF Program이 실행되도록 연결한다. 이후 어떤 Process든 `execve()`를 호출하면 Kernel이 Kprobe를 처리하고 eBPF Program을 실행한다.

```text
Process
  ↓
execve()
  ↓
System Call Entry
  ↓
Kernel의 execve 구현 함수
  ↓
Kprobe Trigger
  ↓
hello() 실행
```

Python에서 `hello()`를 호출하지 않았는데도 Program이 실행되는 이유는 Kernel Hook에 Attach되어 있기 때문이다. 이 구조가 eBPF의 Event-driven 실행 모델이다.

## execve를 Hook으로 선택한 이유

`execve()`는 현재 Process Image를 새로운 Program Image로 교체하는 System Call이다. Shell에서 `ls`, `cat`, `curl`, `python`과 같은 Command를 실행할 때 일반적으로 최종적으로 `execve()`가 호출된다.

따라서 `execve()`에 eBPF Program을 Attach하면 새로운 Program이 실행되는 순간을 관찰할 수 있다. 테스트도 쉽다. 다른 Terminal에서 Command를 실행하면 곧바로 Event가 발생한다.

다만 Shell Built-in Command는 별도의 실행 파일을 `execve()`하지 않을 수 있다. 예를 들어 일부 Shell의 `cd`는 Shell Process 내부에서 처리되므로 동일한 방식으로 Event가 발생하지 않는다. 이는 eBPF Program이 Command 문자열 자체를 감시하는 것이 아니라 실제 Kernel Event를 감시한다는 점을 보여준다.

## bpf_trace_printk()와 Helper Function

eBPF Program은 Kernel 내부에서 실행되므로 일반 User Space Library Function을 호출할 수 없다. `printf()`, `malloc()`, `fopen()`과 같은 Function은 사용할 수 없다. eBPF Program은 Kernel이 허용한 BPF Helper Function을 통해서만 제한적으로 Kernel 기능에 접근한다.

`bpf_trace_printk()`는 Trace Buffer에 문자열을 기록하는 Helper Function이다.

```c
bpf_trace_printk("Hello World");
```

실행 흐름은 다음과 같다.

```text
eBPF Program
  ↓
bpf_trace_printk()
  ↓
Kernel Tracing Buffer
  ↓
trace_pipe
  ↓
User Space 출력
```

`b.trace_print()`는 `/sys/kernel/debug/tracing/trace_pipe`를 읽고 내용을 화면에 출력한다. 화면에 표시되는 Process Name, PID, CPU Number, Timestamp 등의 Metadata는 `Hello World` 문자열과 함께 Kernel Tracing Subsystem이 제공한다.

## trace_pipe가 Debugging 용도인 이유

`bpf_trace_printk()`는 Program이 정상적으로 Attach되었는지, 특정 분기문이 실행되는지 확인하는 데 유용하다. 그러나 실제 Observability, Networking, Security Application의 데이터 전달 방식으로 사용하기에는 한계가 크다.

첫째, `trace_pipe`는 System 전체에서 공유되는 단일 Trace Stream이다. 여러 Tracing Tool과 eBPF Program이 동시에 사용하면 출력이 섞일 수 있다.

둘째, 데이터가 문자열로 전달된다. PID, UID, Command Name, Network Address와 같은 필드를 문자열로 조합하면 User Space에서 다시 Parsing해야 한다. 구조화된 자료형을 그대로 전달하는 방식보다 비효율적이고 오류 가능성이 높다.

셋째, 고빈도 Event에 적합하지 않다. Packet, System Call, Scheduler Event처럼 매우 자주 발생하는 Event마다 문자열 Formatting과 Trace Output을 수행하면 Overhead가 커진다.

따라서 `bpf_trace_printk()`는 운영 데이터 전송 수단이 아니라 개발과 디버깅을 위한 도구로 이해하는 것이 적절하다.

## Event마다 지역변수가 초기화되는 이유

eBPF Program은 항상 실행 중인 Daemon이 아니다. Event가 발생할 때 Kernel이 Function을 호출하고, Function이 Return하면 실행이 끝난다. Program의 Stack Frame도 해당 실행이 끝나면 더 이상 유지되지 않는다.

예를 들어 다음 코드는 누적 Counter를 만들지 못한다.

```c
int hello(void *ctx) {
    int counter = 0;
    counter++;
    return 0;
}
```

각 Event마다 `counter`가 0으로 초기화된 후 1이 되고, Program이 Return하면 사라진다.

```text
첫 번째 Event: counter = 0 → 1 → 종료
두 번째 Event: counter = 0 → 1 → 종료
세 번째 Event: counter = 0 → 1 → 종료
```

Observability에서는 Event 사이의 상태를 유지해야 하는 경우가 많다. UID별 System Call Count, PID별 Packet Count, IP별 Drop Count, File별 Access Count 등을 집계하려면 이전 실행의 값을 다음 실행에서 다시 읽을 수 있어야 한다. 이 문제를 해결하는 핵심 자료구조가 BPF Map이다.

## BPF Map이란

BPF Map은 Kernel이 관리하는 Persistent Kernel Object다. eBPF Program의 지역변수와 달리 Program 한 번의 실행이 끝나도 유지된다. eBPF Program은 Helper Function을 통해 Map을 조회하거나 수정하며, User Space Application도 `bpf()` System Call 또는 Framework API를 통해 동일한 Map에 접근할 수 있다.

```text
                User Space
                    │
                    │ Lookup / Update
                    ▼
             +---------------+
             |    BPF Map    |
             +---------------+
                ▲          ▲
                │          │
             eBPF A      eBPF B
```

따라서 BPF Map은 단순한 Key-Value 저장소가 아니라 Kernel Space와 User Space 사이의 공유 데이터 인터페이스다. 여러 eBPF Program이 동일한 Map을 공유하는 구조도 가능하다.

## UID별 execve 호출 횟수 집계

Chapter 2의 Map 예제는 UID를 Key로 하고 `execve()` 호출 횟수를 Value로 저장한다.

```text
UID 0    → 25
UID 1000 → 73
UID 1001 → 11
```

BCC에서는 다음 Macro로 Hash Map을 정의한다.

```c
BPF_HASH(counter_table);
```

완전한 Kernel Side Code는 다음과 같은 흐름을 가진다.

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

## 현재 UID를 구하는 과정

`bpf_get_current_uid_gid()`는 현재 실행 Context의 UID와 GID를 하나의 64-bit 값으로 반환한다.

```text
상위 32-bit: GID
하위 32-bit: UID
```

따라서 UID만 사용하려면 하위 32-bit만 남겨야 한다.

```c
uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
```

`0xFFFFFFFF`와 Bitwise AND를 수행하면 상위 32-bit가 제거되고 하위 32-bit UID만 남는다. 이 값이 Hash Map의 Key가 된다.

## Hash Map Lookup과 Update

현재 UID에 해당하는 Entry가 이미 존재하는지 조회한다.

```c
p = counter_table.lookup(&uid);
```

BCC가 제공하는 C-like Syntax이므로 일반적인 C Method Call처럼 보이지만, BCC는 Compile 전에 이를 적절한 BPF Map Helper 호출 형태로 Rewrite한다.

Lookup 결과는 Value를 가리키는 Pointer다. Entry가 존재하지 않으면 `NULL`이 반환된다.

```c
if (p != 0) {
    counter = *p;
}
```

기존 Entry가 있으면 Value를 읽어 `counter`에 복사하고, Entry가 없으면 초기값 0을 그대로 사용한다. 이후 Counter를 증가시켜 Map에 기록한다.

```c
counter++;
counter_table.update(&uid, &counter);
```

전체 동작은 다음과 같다.

```text
execve Event
  ↓
현재 UID 조회
  ↓
counter_table Lookup
  ↓
Entry 존재?
 ├─ Yes → 기존 Counter 읽기
 └─ No  → Counter 0 사용
  ↓
Counter + 1
  ↓
Map Update
```

## BPF Map이 Kernel Object라는 의미

BPF Map은 eBPF Program의 Stack이나 Heap 일부가 아니다. Kernel이 별도 Object로 생성하고 관리한다. User Space에서는 Map File Descriptor를 통해 접근하고, eBPF Program에서는 Map Reference와 Helper Function을 통해 접근한다.

이 구조 덕분에 eBPF Program의 실행이 종료되어도 Map Entry는 유지된다. 또한 User Space Loader가 Map을 Pin하지 않고 종료하면 Reference가 사라져 Map도 정리될 수 있지만, bpffs에 Pin하면 Loader Process와 독립적으로 유지할 수 있다. Chapter 2에서는 Pinning까지 다루지 않지만, Map이 일반 지역변수와 다른 이유를 이해하는 데 중요한 배경이다.

## User Space에서 Map을 읽는 과정

Python Side에서는 BCC가 Map을 Python Object로 노출한다.

```python
while True:
    sleep(2)
    s = ""

    for k, v in b["counter_table"].items():
        s += f"ID {k.value}: {v.value}\t"

    print(s)
```

Kernel Side eBPF Program은 Event가 발생할 때마다 Map을 Update한다. User Space Program은 2초마다 Map 전체를 조회해 현재 누적값을 출력한다.

```text
Kernel Side
execve → Map Update → execve → Map Update

User Space
2초 대기 → Map 전체 조회 → 출력 → 2초 대기
```

이 방식은 집계값이나 현재 상태를 가져오는 데 적합하다. 예를 들어 UID별 호출 횟수, Interface별 Packet Count, Error Code별 발생 횟수처럼 최신 누적 상태가 중요할 때 사용할 수 있다.

## sudo ls에서 두 개의 execve가 관찰되는 이유

일반 사용자 UID가 1000이라고 가정하면 `ls`를 실행할 때 UID 1000의 Counter가 증가한다. 반면 `sudo ls`는 일반적으로 두 단계의 Program 실행을 포함한다.

```text
Shell
  ↓ execve
sudo Process 실행: UID 1000
  ↓ 인증 및 권한 전환
execve
ls Process 실행: UID 0
```

따라서 User UID와 Root UID의 Counter가 각각 증가할 수 있다. 이 결과는 eBPF가 Shell Command Text를 단순히 읽는 것이 아니라, 실제 Process 실행과 System Call Context를 Kernel에서 관찰한다는 점을 보여준다.

## BPF Map만으로는 Event의 순서를 알 수 없다

Hash Map은 현재 상태를 보관하는 데 적합하지만, 개별 Event 자체를 보존하는 것은 아니다. 예를 들어 다음 Entry가 있다고 가정한다.

```text
UID 1000 → 38
```

이 값으로는 UID 1000이 `execve()`를 38번 호출했다는 사실은 알 수 있다. 그러나 어떤 Command를 실행했는지, 언제 실행했는지, 어떤 순서로 Event가 발생했는지는 알 수 없다.

```text
Map이 제공하는 것
- 현재 Count
- Key별 누적 상태

Map만으로 제공하기 어려운 것
- Event 발생 순서
- Event별 Timestamp
- Event별 Command
- Event별 Context
```

실시간 Observability에서는 누적 상태보다 Event Stream 자체가 필요한 경우가 많다. Process 실행, File Open, Network Connection, Packet Drop과 같은 Event를 발생 순서대로 User Space에 전달하려면 별도의 Buffer가 필요하다.

## Perf Buffer와 Ring Buffer

Linux Kernel은 Perf Event Subsystem을 통해 Kernel에서 User Space로 Event Data를 전달할 수 있다. eBPF는 Perf Event Array Map을 이용해 이 경로를 사용할 수 있으며, BCC는 `BPF_PERF_OUTPUT` Macro로 이를 추상화한다.

Perf Buffer는 Producer인 eBPF Program이 Event Record를 쓰고, Consumer인 User Space Application이 이를 읽는 Queue 형태로 동작한다. 논리적으로는 Ring 구조를 사용하며, Write Position과 Read Position이 순환한다.

```text
eBPF Program
   ↓ Event Submit
+-----------------------+
| Perf Ring Buffer      |
| Event A | B | C | ... |
+-----------------------+
   ↓ Poll
User Space Application
```

Buffer가 비어 있으면 읽을 Event가 없다. Producer가 Consumer보다 빠르게 Event를 기록해 Buffer 공간이 부족하면 일부 Event가 Drop될 수 있다. 따라서 User Space Consumer의 처리 속도와 Buffer Size가 중요하다.

Linux 5.8 이후에는 BPF Ring Buffer가 추가되었으며, 일반적으로 새 Application에서는 Perf Buffer보다 BPF Ring Buffer가 선호된다. 다만 Chapter 2는 BCC의 `BPF_PERF_OUTPUT`을 이용해 Kernel-to-User Event 전달 개념을 설명한다.

## 구조화된 Event 정의

Perf Buffer 예제에서는 문자열 하나가 아니라 C Structure 전체를 User Space로 전달한다.

```c
BPF_PERF_OUTPUT(output);

struct data_t {
    int pid;
    int uid;
    char command[16];
    char message[12];
};
```

`data_t`는 하나의 Event Record Format이다. Kernel Side와 User Space Side가 동일한 Layout을 기준으로 데이터를 해석한다.

```text
data_t
├─ pid
├─ uid
├─ command[16]
└─ message[12]
```

이 방식의 장점은 문자열 Parsing이 필요하지 않다는 것이다. 각 Field가 고정된 Type과 Offset을 가지므로 User Space는 PID, UID, Command, Message를 각각 직접 읽을 수 있다.

## Context 정보 수집

Kernel Side Program은 Event가 발생할 때 현재 Process Context에서 필요한 정보를 수집한다.

```c
int hello(void *ctx) {
    struct data_t data = {};
    char message[12] = "Hello World";

    data.pid = bpf_get_current_pid_tgid() >> 32;
    data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;

    bpf_get_current_comm(&data.command, sizeof(data.command));
    bpf_probe_read_kernel(&data.message, sizeof(data.message), message);

    output.perf_submit(ctx, &data, sizeof(data));

    return 0;
}
```

`bpf_get_current_pid_tgid()`는 64-bit 값을 반환한다. Chapter 2 예제에서는 상위 32-bit를 Shift해 Process ID를 얻는다.

```c
data.pid = bpf_get_current_pid_tgid() >> 32;
```

`bpf_get_current_uid_gid()`는 앞의 Map 예제와 동일하게 하위 32-bit UID를 사용한다.

```c
data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
```

`bpf_get_current_comm()`은 현재 Task의 Command Name을 지정한 Buffer에 복사한다.

```c
bpf_get_current_comm(&data.command, sizeof(data.command));
```

eBPF Program은 임의의 Kernel Memory를 일반 Pointer Dereference 방식으로 자유롭게 읽을 수 없다. Verifier가 Memory Access의 안전성을 확인할 수 있어야 하며, 필요한 경우 적절한 Helper Function을 이용한다. 예제에서는 `bpf_probe_read_kernel()`을 사용해 Message를 Structure Field에 복사한다.

## Event Submit

Structure의 각 Field를 채운 뒤 Perf Buffer로 제출한다.

```c
output.perf_submit(ctx, &data, sizeof(data));
```

이 호출은 `data` Structure의 Byte를 Perf Buffer Event Record로 복사한다. eBPF Program이 Return한 뒤 지역변수 `data`는 사라지지만, Buffer에 제출된 Event Record는 User Space가 읽을 때까지 Buffer에 남는다.

```text
eBPF Stack의 data_t
       ↓ perf_submit
Perf Buffer의 Event Record
       ↓
eBPF Program Return
       ↓
User Space Consumer가 Event Read
```

## User Space Callback과 Polling

Python Side에서는 Perf Buffer를 열고 Callback Function을 등록한다.

```python
def print_event(cpu, data, size):
    event = b["output"].event(data)

    print(
        f"{event.pid} "
        f"{event.uid} "
        f"{event.command.decode()} "
        f"{event.message.decode()}"
    )

b["output"].open_perf_buffer(print_event)

while True:
    b.perf_buffer_poll()
```

`open_perf_buffer(print_event)`는 Buffer에서 Event가 도착했을 때 호출할 Callback을 등록한다. `perf_buffer_poll()`은 Event 도착을 기다리고, 읽을 Data가 있으면 BCC가 Callback을 호출한다.

```text
Kernel
  ↓ perf_submit()
Perf Buffer
  ↓ poll()
BCC
  ↓
print_event()
  ↓
Terminal 출력
```

Map Polling 예제는 2초마다 전체 Hash Map을 읽었다. 반면 Perf Buffer 예제는 Event가 발생할 때마다 개별 Record를 전달받는다. 두 방식은 목적이 다르다.

| 구분 | Hash Map | Perf Buffer |
|---|---|---|
| 목적 | 상태와 집계값 저장 | 개별 Event 전달 |
| 데이터 형태 | Key-Value | 순차 Event Record |
| User Space 접근 | 주기적으로 조회 | Event Polling |
| 적합한 예 | UID별 호출 횟수 | Process 실행 Event |
| Event 순서 | 보존하지 않음 | Buffer 순서로 전달 |

## trace_pipe와 Perf Buffer의 차이

첫 번째 Hello World 예제는 모든 결과를 System-wide Trace Pipe로 보냈다. Perf Buffer 예제는 해당 Application이 생성한 전용 Buffer를 사용한다.

```text
bpf_trace_printk()
  ↓
공용 trace_pipe

perf_submit()
  ↓
Application 전용 Perf Buffer
```

Perf Buffer를 사용하면 다른 Tracing Output과 섞이지 않으며, 문자열이 아닌 Structure를 전달할 수 있다. 또한 Callback을 통해 Event를 Programmatically 처리할 수 있어 Log 저장, Metrics 변환, Policy Engine 연계 등의 후속 처리가 가능하다.

## Kernel에서 Context를 수집하는 장점

eBPF Program은 Event가 발생한 바로 그 Kernel Context에서 실행된다. 따라서 현재 PID, UID, Command Name 등 Event와 관련된 정보를 Kernel 내부에서 즉시 수집할 수 있다.

이 방식은 User Space Agent가 `/proc`를 반복해서 조회하는 방식과 다르다. User Space Polling은 짧게 생성되고 종료되는 Process를 놓칠 수 있고, Event 발생 시점과 조회 시점 사이에 상태가 변경될 수 있다. 반면 eBPF는 실제 Event 경로에서 실행되기 때문에 Event 발생 시점의 Context를 직접 캡처할 수 있다.

이 특성이 eBPF Observability의 핵심이다. 동일한 원리를 확장하면 System Call, File Access, TCP Connection, Packet, Scheduler Event 등의 Context를 수집할 수 있다.

## eBPF Program에서 Function을 나누는 문제

Program이 커지면 모든 로직을 하나의 Function에 작성하기 어렵다. 일반적인 Software에서는 공통 로직을 Function으로 분리하고 여러 위치에서 호출한다. 그러나 초기 eBPF는 Kernel Helper 이외의 Function Call을 허용하지 않았다.

당시에는 Function을 다음과 같이 강제 Inline하는 방식이 사용되었다.

```c
static __always_inline void my_function(void *ctx, int value) {
    // 공통 로직
}
```

Inline은 실제 Call Instruction을 만들지 않는다. Compiler가 Function Body를 호출 위치에 복사한다.

```text
일반 Function Call
Caller → Jump → Callee → Return

Inline
Caller 내부에 Callee Instruction 복사
```

Inline은 Call 제약을 우회하지만, 여러 위치에서 호출하면 동일한 Instruction이 반복되어 Program Size가 커질 수 있다.

Linux Kernel 4.16과 LLVM 6.0 이후에는 BPF-to-BPF Function Call, 즉 BPF Subprogram이 지원되기 시작했다. 이를 사용하면 일반적인 Function 분리가 가능하다. 다만 Chapter 2에서 사용하는 BCC Framework는 당시 이 기능을 직접 지원하지 않아 Inline Function을 중심으로 설명한다.

## Tail Call이란

Tail Call은 현재 eBPF Program에서 다른 eBPF Program으로 실행을 전환하는 메커니즘이다. 일반 Function Call과 달리 호출된 Program이 끝난 뒤 Caller로 돌아오지 않는다.

```text
Program A
  ↓ Tail Call
Program B
  ↓ Return
Hook 실행 종료
```

성공한 Tail Call은 현재 Program을 새로운 Program으로 교체한다. 이는 일반 Process에서 `execve()`가 현재 Process Image를 새로운 Program Image로 대체하는 것과 유사하다.

Tail Call은 다음 Helper Function을 사용한다.

```c
long bpf_tail_call(
    void *ctx,
    struct bpf_map *prog_array_map,
    u32 index
);
```

첫 번째 Argument는 현재 Event Context이며, 호출 대상 Program도 같은 Context를 전달받는다. 두 번째 Argument는 Program Array Map이다. 세 번째 Argument는 Program Array에서 실행할 Program을 선택하는 Index다.

## Program Array Map

Tail Call 대상은 `BPF_MAP_TYPE_PROG_ARRAY` Map에 저장한다. 일반 Data Map과 달리 Value로 eBPF Program을 참조한다.

```text
Program Array
0   → Program A
59  → execve Handler
222 → timer Handler
226 → timer Handler
```

Kernel Side Program은 Index를 계산해 Program Array를 조회하고 Tail Call을 시도한다. User Space Loader는 미리 각 eBPF Program을 Load한 뒤 Program Array Entry를 구성해야 한다.

BCC에서는 다음 Macro로 정의한다.

```c
BPF_PROG_ARRAY(syscall, 300);
```

## System Call Opcode를 이용한 Tail Call Dispatch

Chapter 2 예제는 모든 System Call 진입점에서 실행되는 `sys_enter` Raw Tracepoint에 Main Program을 Attach한다.

```c
int hello(struct bpf_raw_tracepoint_args *ctx) {
    int opcode = ctx->args[1];

    syscall.call(ctx, opcode);

    bpf_trace_printk("Another syscall: %d", opcode);
    return 0;
}
```

`ctx->args[1]`에서 System Call Opcode를 읽고, 해당 Opcode를 Program Array Index로 사용한다.

```text
sys_enter
  ↓
System Call Opcode 확인
  ↓
Program Array Lookup
  ↓
Entry 존재?
 ├─ Yes → Tail Call로 전환
 └─ No  → Main Program 계속 실행
```

BCC의 `syscall.call(ctx, opcode)`는 Compile 전에 `bpf_tail_call()` 형태로 Rewrite된다.

Tail Call이 성공하면 Main Program의 다음 줄은 실행되지 않는다. Tail Call이 실패하거나 해당 Index에 Entry가 없을 때만 기본 Trace Message가 출력된다.

## Tail Call 대상 Program

특정 System Call은 전용 Program에서 처리할 수 있다.

```c
int hello_execve(void *ctx) {
    bpf_trace_printk("Executing a program");
    return 0;
}
```

Timer 관련 여러 Opcode는 하나의 Program으로 연결할 수 있다.

```c
int hello_timer(struct bpf_raw_tracepoint_args *ctx) {
    if (ctx->args[1] == 222) {
        bpf_trace_printk("Creating a timer");
    } else if (ctx->args[1] == 226) {
        bpf_trace_printk("Deleting a timer");
    } else {
        bpf_trace_printk("Some other timer operation");
    }

    return 0;
}
```

아무 동작도 하지 않는 Program을 연결해 자주 발생하지만 관심 없는 System Call을 무시할 수도 있다.

```c
int ignore_opcode(void *ctx) {
    return 0;
}
```

Program Array의 여러 Index가 동일한 Program을 가리키는 것도 가능하다. 즉 Dispatch Table처럼 사용할 수 있다.

## User Space에서 Tail Call 구성

각 Tail Call 대상 Program은 독립적인 eBPF Program이므로 User Space에서 각각 Load해야 한다.

```python
b = BPF(text=program)
b.attach_raw_tracepoint(tp="sys_enter", fn_name="hello")

ignore_fn = b.load_func("ignore_opcode", BPF.RAW_TRACEPOINT)
exec_fn = b.load_func("hello_execve", BPF.RAW_TRACEPOINT)
timer_fn = b.load_func("hello_timer", BPF.RAW_TRACEPOINT)
```

이후 Program Array Map에 Program File Descriptor를 등록한다.

```python
prog_array = b.get_table("syscall")

prog_array[ct.c_int(59)] = ct.c_int(exec_fn.fd)
prog_array[ct.c_int(222)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(226)] = ct.c_int(timer_fn.fd)
```

Tail Call 대상 Program은 Caller와 동일한 Program Type이어야 한다. 이 예제에서는 Main Program이 Raw Tracepoint Program이므로 대상 Program도 `BPF.RAW_TRACEPOINT` Type으로 Load한다.

전체 구조는 다음과 같다.

```text
User Space Loader
  ├─ Main Program Load
  ├─ execve Program Load
  ├─ timer Program Load
  ├─ ignore Program Load
  └─ Program Array 구성

Kernel
sys_enter
  ↓
Main Program
  ↓ Opcode 기반 Tail Call
  ├─ execve Handler
  ├─ timer Handler
  ├─ ignore Handler
  └─ 기본 처리
```

## Function Call과 Tail Call의 차이

BPF-to-BPF Function Call은 하나의 eBPF Program 내부에서 Subprogram을 호출하고 다시 Caller로 돌아오는 구조다. Tail Call은 독립적인 다른 eBPF Program으로 실행을 넘기며 성공하면 Caller로 돌아오지 않는다.

| 구분 | BPF Function Call | Tail Call |
|---|---|---|
| 호출 대상 | 같은 Program의 Subprogram | 별도 eBPF Program |
| 호출 후 복귀 | Caller로 Return | Caller로 복귀하지 않음 |
| 대상 선택 | Compile Time | Program Array를 통한 Runtime 선택 |
| Context | Function Argument | 기존 Program Context 전달 |
| 주요 목적 | Code 재사용과 구조화 | Program 분리와 동적 Dispatch |

Tail Call은 Runtime에 Program Array Entry를 교체할 수 있기 때문에 동적인 Program 구성에도 활용할 수 있다. Main Program을 변경하지 않고 특정 Index의 Handler Program만 교체하는 구조가 가능하다.

## Tail Call의 제약

Tail Call은 무한히 연속 호출할 수 없다. Kernel은 Tail Call 횟수에 제한을 두어 무한 Loop 형태의 Program Chain을 방지한다. 또한 Tail Call 대상은 동일한 Program Type과 호환 가능한 Attach Context를 사용해야 한다.

Tail Call이 실패할 수 있다는 점도 중요하다. Program Array에 해당 Index가 없거나 조건을 만족하지 못하면 Helper가 Return하고 Caller의 다음 Instruction이 계속 실행된다. 따라서 Tail Call 이후에 Default 처리 또는 Failure 처리 로직을 배치할 수 있다.

## Chapter 2 전체 실행 구조

Chapter 2의 예제들을 하나로 연결하면 eBPF Application의 기본 Architecture가 보인다.

```text
                 User Space
+------------------------------------------+
| BCC Python Application                   |
|                                          |
| - Source Compile                         |
| - Program Load                           |
| - Hook Attach                            |
| - Map Read                               |
| - Perf Buffer Poll                       |
| - Tail Call Program Array 구성           |
+--------------------+---------------------+
                     │
                     │ bpf() / Perf Event
                     ▼
                 Kernel Space
+------------------------------------------+
| Kernel Hook                              |
| - Kprobe                                 |
| - Raw Tracepoint                         |
+--------------------+---------------------+
                     │ Event
                     ▼
+------------------------------------------+
| eBPF Program                             |
| - Context 수집                           |
| - Map Lookup / Update                    |
| - Perf Buffer Submit                     |
| - Tail Call Dispatch                     |
+------------------------------------------+
          │                    │
          ▼                    ▼
+------------------+   +-------------------+
| BPF Map          |   | Perf/Ring Buffer  |
| 상태와 집계값     |   | 개별 Event Stream |
+------------------+   +-------------------+
```

eBPF Program 자체는 짧게 실행되지만, User Space Loader와 Map, Buffer를 결합하면 지속적으로 동작하는 Observability Application을 만들 수 있다.

## 각 데이터 전달 방식의 역할

Chapter 2에서는 Kernel에서 얻은 정보를 User Space로 전달하는 세 가지 방식을 확인한다.

### trace_pipe

`bpf_trace_printk()`를 이용하는 가장 단순한 방식이다. 개발 중 Program 실행 여부를 확인하는 데 적합하지만, System-wide Shared Buffer와 문자열 기반 출력이라는 한계가 있다.

### BPF Map

상태와 집계값을 보존한다. Event마다 값을 누적하거나 현재 상태를 공유할 때 적합하다. User Space는 필요할 때 Map을 조회한다.

### Perf Buffer

각 Event를 순서대로 User Space에 전달한다. PID, UID, Command와 같은 구조화된 Context를 실시간으로 처리할 때 적합하다.

```text
Debug Message      → trace_pipe
상태와 집계값       → BPF Map
실시간 Event Stream → Perf/Ring Buffer
```

실제 Application에서는 하나만 사용하는 것이 아니라 여러 방식을 조합할 수 있다. 예를 들어 Process 실행 Event는 Ring Buffer로 전달하고, UID별 누적 Count는 Hash Map에 저장할 수 있다.

## Chapter 2의 핵심

Chapter 2의 Hello World는 문자열 출력 예제에 그치지 않는다. 이 장은 eBPF Application을 구성하는 핵심 요소를 작은 예제로 분리해 보여준다.

eBPF Program은 독립 실행 Process가 아니라 Kernel Event에 Attach되는 Handler다. User Space Program은 eBPF Program을 Compile하고 Load하고 Attach하며, 실행 결과를 읽고 관리한다. Event 사이에서 상태를 유지하려면 BPF Map이 필요하고, 개별 Event를 실시간으로 전달하려면 Perf Buffer나 BPF Ring Buffer가 필요하다. Program이 복잡해지면 Inline Function, BPF-to-BPF Function Call, Tail Call을 이용해 로직을 구조화할 수 있다.

이 장에서 가장 중요한 흐름은 다음과 같다.

```text
Program Load
  ↓
Hook Attach
  ↓
Kernel Event 발생
  ↓
eBPF Program 실행
  ↓
Context 수집
  ↓
Map에 상태 저장 또는 Buffer에 Event 전달
  ↓
User Space에서 처리
```

이 구조는 이후 Cilium, Hubble, Tetragon, bpftrace와 같은 실제 eBPF 기반 도구를 이해하는 기반이 된다. 각각의 도구는 Attach Point와 Program Type, Map Type, Event Schema는 다르지만, Kernel Event를 관찰하고 BPF Map 또는 Buffer를 통해 User Space와 데이터를 교환한다는 기본 Architecture는 동일하다.