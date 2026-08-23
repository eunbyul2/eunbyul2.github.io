---
layout: post
title: "[Learning eBPF] Chapter 6: The eBPF Verifier"
date: 2026-08-22 19:33:00 +0900
categories: [eBPF, Linux, Kernel]
tags: [eBPF, BPF, Verifier, eBPF Verifier, BPF Helper, XDP, BPF Map, Static Analysis, Kernel]
published: true
---

# Learning eBPF Chapter 6

## The eBPF Verifier

Chapter 5에서는 BTF와 CO-RE를 통해 **컴파일한 eBPF 프로그램을 서로 다른 Kernel 환경에서도 동작시키는 방법**을 살펴봤다.

이번 Chapter 6의 주제는 **eBPF Verifier**이다.

이전까지 Verifier를 단순히 "eBPF Program을 Kernel에 Load하기 전에 안전한지 검사하는 단계" 정도로 생각했다. 하지만 이번 Chapter를 읽으면서 Verifier는 단순한 문법 검사기가 아니라, **컴파일된 eBPF Bytecode의 실행 경로를 따라가며 Register의 Type과 Value Range를 추적하고, 모든 가능한 경로가 안전한지를 정적으로 분석하는 검사기**라는 점을 이해하게 됐다.

eBPF Program은 Kernel 안에서 실행되기 때문에 잘못된 Pointer Access나 Memory Out-of-Bounds, 끝나지 않는 Loop 등이 그대로 실행되면 Kernel 전체에 영향을 줄 수 있다. 그래서 eBPF에서는 Program을 실제로 실행하기 전에 Verifier가 안전성을 증명할 수 있어야 한다.

이번 Chapter는 결국 다음 질문에 대한 내용이었다.

> **Kernel은 eBPF Program을 실제로 실행하지 않고 어떻게 이 Program이 안전한지 판단할까?**

---

# 1. Compile 성공과 Verifier 통과는 다르다

eBPF Program이 실행되기까지의 흐름을 단순화하면 다음과 같다.

```text
eBPF C Source
      │
      │ Clang / LLVM Compile
      ▼
eBPF Object File (.o)
 └─ eBPF Bytecode 포함
      │
      │ libbpf / bpftool 등 Loader
      ▼
bpf(BPF_PROG_LOAD, ...)
      │
      ▼
eBPF Verifier
      │
 ┌────┴────┐
 │         │
PASS      FAIL
 │         │
 ▼         ▼
Kernel    Load
Load      Reject
 │
 ▼
JIT Compilation
 │
 ▼
Native Machine Code
 │
 ▼
실행
```

![eBPF 프로그램의 전체 실행 구조](/img/study-Learning_eBPF/eBPF%20프로그램의%20전체%20실행%20구조.png)

`.o`는 ELF 형식의 Object File이고, 그 안에는 Compiler가 생성한 eBPF Bytecode가 들어 있다.

중요한 점은 **`.o` 파일이 만들어졌다고 해서 Kernel에서 실행 가능한 안전한 Program이라는 뜻은 아니라는 것**이다.

Compiler는 Source Code를 eBPF Bytecode로 변환한다. 그 이후 Kernel에 Load하려고 할 때 Verifier가 Bytecode의 안전성을 별도로 검사한다.

즉 다음 두 단계는 다르다.

```text
Compile 성공
→ Bytecode 생성 가능

Verifier 통과
→ Kernel에서 안전하게 실행 가능하다고 판단
```

또한 `BPF_PROG_LOAD`는 터미널에서 직접 실행하는 명령이 아니라 `bpf()` System Call에 전달되는 Command 중 하나이다. 일반적으로 libbpf나 bpftool 같은 User Space Loader가 Program을 Kernel에 Load하기 위해 사용한다.

---

## Verifier는 C Source가 아니라 Bytecode를 검사한다

Verifier가 직접 보는 것은 C Source가 아니라 Compiler가 만든 **eBPF Bytecode**이다.

```text
C Source
   ↓
Compiler
   ↓
eBPF Bytecode
   ↓
Verifier
```

따라서 Source Code에 어떤 코드가 있더라도 Compiler Optimization 과정에서 제거되면 Verifier는 그 코드를 보지 못한다.

이 점 때문에 Verifier Error를 볼 때는 C Source만 보는 것이 아니라, **Source가 어떤 Bytecode로 변환되었는지**까지 생각해야 한다.

---

# 2. Verifier는 Register의 상태를 추적한다

Verifier는 Program을 실제로 실행해보는 대신 Bytecode Instruction을 따라가면서 Program의 상태를 **정적으로 추론**한다.

이 과정에서 Chapter 3에서 봤던 eBPF Virtual Machine의 Register가 중요하게 사용된다.

| Register | 주요 역할 |
|---|---|
| `R0` | Helper Function / eBPF Program의 Return Value |
| `R1 ~ R5` | Function Argument |
| `R6 ~ R9` | Callee-saved Register |
| `R10` | Stack Frame Pointer |

Verifier는 각 Register에 대해 단순히 "어떤 값이 들어 있다"만 보는 것이 아니라 크게 두 가지를 추적한다.

```text
Register State
     │
     ├── Type
     │   ├─ SCALAR_VALUE
     │   ├─ PTR_TO_CTX
     │   ├─ PTR_TO_PACKET
     │   └─ PTR_TO_MAP_VALUE ...
     │
     └── Value Range
         ├─ 최소값
         └─ 최대값
```

Pointer를 모두 같은 주소값으로 취급하지 않고 **어디에서 온 Pointer인지까지 구분한다는 것**이 중요하다.

예를 들어 Verifier가 R1을 `PTR_TO_CTX`라고 알고 있다면 현재 Program Type에서 허용된 Context Field만 접근하도록 검사할 수 있다. R3가 `PTR_TO_PACKET`이라면 Packet Boundary를 넘지 않는지 확인할 수 있다.

Value Range 역시 중요하다. 예를 들어 배열의 유효 Index가 `0~11`인데 Index로 사용되는 Register가 `0~12`까지 가능하다고 분석되면, Verifier는 실제 실행 여부와 관계없이 Out-of-Bounds 가능성이 있다고 판단한다.

즉 Verifier가 보는 것은:

> **"지금 실제 값이 무엇인가?"가 아니라 "이 Register가 가질 수 있는 값의 범위 안에서 위험한 경우가 존재하는가?"**

이다.

---

# 3. Branch가 나오면 가능한 경로를 따라간다

조건문이 나오면 실행 경로가 여러 개로 나뉜다.

Verifier는 한쪽 경로만 보는 것이 아니라 현재 Register State를 기준으로 가능한 Branch를 따라가며 각 경로를 검사한다.

```text
              Branch
                │
        ┌───────┴───────┐
        ▼               ▼
     Path A           Path B
        │               │
      검사              검사
```

이게 중요한 이유는 eBPF Program이 **어떤 경로로 실행되더라도 안전해야 하기 때문**이다.

특정 입력에서는 문제가 없더라도 다른 Branch에서 NULL Pointer Dereference나 Out-of-Bounds Access가 가능하다면 Verifier는 Program을 Reject할 수 있다.

Verifier는 모든 경로를 무조건 처음부터 다시 분석하지는 않는다. 이전에 검증한 State와 동일하거나 이미 검증한 범위 안에 포함되는 State로 같은 Instruction에 다시 도달하면 이후 분석을 생략하는 **State Pruning**을 사용한다.

즉 Verifier는 가능한 실행 경로를 분석하면서도, 이미 검증된 State를 재사용해 분석 비용을 줄인다.

---

# 4. Control Flow와 Verifier Log

`bpftool`을 사용하면 Load된 eBPF Program의 Control Flow를 DOT 형식으로 볼 수 있다.

```bash
bpftool prog dump xlated name kprobe_exec visual > out.dot
dot -Tpng out.dot > out.png
```

![eBPF Control Flow Graph](/img/study-Learning_eBPF/eBPF%20Control%20Flow%20Graph.png)

Control Flow Graph를 보면 Instruction이 단순히 위에서 아래로만 실행되는 것이 아니라 Branch에 따라 여러 경로로 나뉜다는 것을 확인할 수 있다.

또한 Verification에 실패했을 때 가장 중요한 자료는 **Verifier Log**이다.

Chapter 예제는 `-g` 옵션으로 Compile했기 때문에 Verifier Log에 C Source Line과 eBPF Instruction이 함께 나타난다.

```text
; c++;

4: (bf) r3 = r2
5: (07) r3 += 1
6: (63) *(u32 *)(r1 +0) = r3
```

이 Debug Information 덕분에 어떤 C Source가 어떤 Register 조작으로 변환되었는지를 훨씬 쉽게 추적할 수 있다.

Verifier Log를 볼 때는 처음부터 모든 Instruction을 읽기보다 다음 순서로 보는 것이 이해하기 쉬웠다.

```text
Error Message 확인
      ↓
문제가 된 Register 확인
      ↓
해당 Register의 Type / Range 확인
      ↓
어디에서 만들어진 값인지 역추적
      ↓
Source Code와 연결
```

---

# 5. Helper Function도 Verifier가 검사한다

eBPF Program은 Kernel 안에서 실행되지만 아무 Kernel Function이나 직접 호출할 수 있는 것은 아니다. 주로 Kernel에서 제공하는 BPF Helper Function을 사용하며, 일부 Kernel Function은 `kfunc`로 eBPF에 노출될 수 있다.

대표적인 Helper는 다음과 같다.

```c
bpf_map_lookup_elem()
bpf_get_current_pid_tgid()
bpf_probe_read_kernel()
bpf_printk()
```

그런데 모든 Helper를 모든 BPF Program Type에서 사용할 수 있는 것은 아니다.

예를 들어 XDP Program에서 `bpf_get_current_pid_tgid()`를 호출하면 다음과 같은 오류가 발생할 수 있다.

```text
unknown func bpf_get_current_pid_tgid#14
```

이 메시지는 Helper 자체가 존재하지 않는다는 의미가 아니라, **현재 BPF Program Type에서 사용할 수 없는 Helper라는 의미**이다.

XDP는 Network Interface에서 Packet을 처리하는 Context에서 실행되고 특정 User Space Process Context를 전제로 하지 않으므로, 이 Helper는 XDP Program Type에서 허용되지 않는다.

---

## Helper Argument Type도 검사한다

`bpf_map_lookup_elem()`은 개념적으로 다음과 같은 규칙을 가진다.

```text
Argument 1 → Map Pointer
Argument 2 → Map Key Pointer
Return     → Map Value Pointer 또는 NULL
```

정상적인 호출은 다음과 같다.

```c
p = bpf_map_lookup_elem(&my_config, &uid);
```

그런데 책에서는 일부러 첫 번째 Argument에 Map이 아닌 Local Variable의 주소를 전달한다.

```c
p = bpf_map_lookup_elem(&data, &uid);
```

이 코드는 Compile될 수 있지만 Kernel에 Load하면 Verifier가 다음과 같이 Reject한다.

```text
R1 type=fp expected=map_ptr
```

Helper의 첫 번째 Argument는 R1에 전달된다.

Verifier가 본 R1은 Stack 쪽 Local Variable Pointer인 `fp`였지만, `bpf_map_lookup_elem()`이 요구한 것은 `map_ptr`이었다.

```text
실제 R1   → fp
필요한 R1 → map_ptr

        ↓

Type 불일치

        ↓

Reject
```

이 예제를 통해 Register Type Tracking이 실제 Helper 검증에 어떻게 사용되는지 확인할 수 있었다.

---

## GPL License도 Helper 사용 조건 중 하나다

GPL(GNU General Public License)은 오픈소스 라이선스 중 하나이다.

일부 BPF Helper는 Kernel에서 `gpl_only=true`로 정의되어 있어 GPL-Compatible License를 선언한 eBPF Program에서만 사용할 수 있다.

예를 들어 Chapter의 `bpf_probe_read_kernel()`은 GPL 제한이 있는 Helper이기 때문에 Program의 GPL-Compatible License 선언을 제거하면 다음과 같은 오류가 발생한다.

```text
cannot call GPL-restricted function from non-GPL compatible program
```

즉 Verifier의 License 검사는 별도의 뜬금없는 검사가 아니라 **해당 Helper를 사용하기 위해 Kernel이 정의한 조건 중 하나를 확인하는 것**이다.

---

# 6. Memory Access는 어떻게 검사할까?

Verifier의 핵심 역할 중 하나는 Program이 허용된 Memory 범위를 벗어나지 않는지 검사하는 것이다.

Chapter에서는 XDP Packet과 배열 접근 예제를 통해 이를 설명한다.

---

## 6.1 XDP Packet Boundary

XDP Program에서는 `struct xdp_md *ctx`가 Context로 전달된다.

```c
void *data = (void *)(long)ctx->data;
void *data_end = (void *)(long)ctx->data_end;
```

`data`는 Packet Data가 시작하는 위치이고, `data_end`는 접근 가능한 Packet Data 영역의 **끝 경계(end boundary)** 를 나타낸다.

```text
data                             data_end
 ↓                                  ↓
[ Packet Data ...................... )
```

즉 `data`부터 `data_end` 전까지가 유효한 Packet Memory 범위이다.

그래서 Packet Header를 읽으려면 읽으려는 범위가 `data_end`를 넘지 않는다는 것을 먼저 검사해야 한다.

또한 Program이 `data_end` 자체를 임의로 변경해서 Boundary Check를 우회할 수 없도록 Verifier가 제한한다.

```c
data_end++;
```

와 같은 코드는 다음 오류로 Reject된다.

```text
R3 pointer arithmetic on pkt_end prohibited
```

---

## 6.2 배열 범위도 Register Range로 판단한다

Chapter에는 다음 코드가 나온다.

```c
if (c < sizeof(message)) {
    char a = message[c];
    bpf_printk("%c", a);
}
```

`message`가 12Byte 배열이라면 C 배열은 0부터 시작하므로 유효한 Index는 `0~11`이다.

따라서:

```c
c < sizeof(message)
```

이면 `c`는 최대 11까지 가능하다.

하지만 조건을 다음처럼 바꾸면 문제가 생긴다.

```c
if (c <= sizeof(message)) {
    char a = message[c];
}
```

`sizeof(message)`가 12이므로 이제 `c == 12`도 가능하다.

```text
c의 최대값 = 12
      ↓
message[12] 가능
      ↓
유효 Index = 0~11
      ↓
Out-of-Bounds 가능
      ↓
Reject
```

Verifier는 실제로 `c=12`인 상황을 실행해본 것이 아니다. Register의 가능한 Range를 추적한 결과 **Out-of-Bounds가 발생할 가능성을 배제할 수 없기 때문에** Reject한 것이다.

---

## 왜 Error에는 `map value`라고 나올까?

`message`는 Source Code에서는 Global Array이다.

하지만 libbpf 기반 eBPF Program의 `.data`, `.bss`, `.rodata` 같은 Global Data 영역은 Kernel에서 사용할 수 있도록 내부적으로 BPF Map을 이용해 구성된다.

```text
C Source

global char message[12];

        ↓

Compiler / libbpf

        ↓

BPF Global Data Map

┌─────────────────────────┐
│ message[0]              │
│ message[1]              │
│ ...                     │
│ message[11]             │
│ 기타 Global Data        │
└─────────────────────────┘
```

그래서 개발자 입장에서는 `message[12]`라는 Array Out-of-Bounds Access이지만, Verifier 입장에서는 Map Value Memory의 범위를 벗어난 접근으로 보일 수 있다.

따라서 다음과 같은 Error가 나타난다.

```text
invalid access to map value
```

---

# 7. NULL Pointer 예제로 이해하는 Verifier

`bpf_map_lookup_elem()`의 반환값 처리

```c
p = bpf_map_lookup_elem(&my_config, &uid);
```

이 Helper는 해당 Key가 있으면 Map Value Pointer를 반환하지만, Key가 없으면 `NULL`을 반환할 수 있다.

즉 Verifier는 반환값을 처음에는 다음처럼 본다.

```text
PTR_TO_MAP_VALUE_OR_NULL
```

Helper Return Value는 R0에 들어가고, 예제에서는 이 값이 R7로 복사된다.

```text
call bpf_map_lookup_elem
        ↓
R0 = MAP_VALUE_OR_NULL
        ↓
R7 = R0
        ↓
R7 = MAP_VALUE_OR_NULL
```

그런데 바로 다음처럼 Dereference하면 문제가 된다.

```c
char a = p->message[0];
```

Verifier 입장에서는 아직 `p == NULL`일 가능성이 남아 있기 때문이다.

```text
R7 = MAP_VALUE_OR_NULL
        ↓
*(R7 + 0)
        ↓
NULL일 가능성 존재
        ↓
Reject
```

실제로 다음과 같은 Error가 발생한다.

```text
R7 invalid mem access 'map_value_or_null'
```

반대로 NULL Check를 넣으면 상황이 달라진다.

```c
if (p != NULL) {
    char a = p->message[0];
}
```

Branch 이전에는:

```text
p = MAP_VALUE_OR_NULL
```

이지만 `p != NULL`인 경로에 들어온 뒤에는 Verifier가 해당 Path에서 `p`가 NULL이 아니라는 사실을 알 수 있다.

```text
p = MAP_VALUE_OR_NULL

        ↓

    p != NULL ?

      /     \
    YES      NO
     │
     ▼
 MAP_VALUE
     │
     ▼
Dereference 허용
```

이 예제 하나에 Chapter 6의 핵심이 거의 다 들어 있다.

- Helper Return Type
- Register State Tracking
- Branch / Control Flow
- Pointer Type Refinement
- NULL Pointer Safety

즉 Verifier는 Bytecode를 위에서 아래로 단순히 읽는 것이 아니라, **각 Branch를 지나면서 Register에 대해 알고 있는 정보 자체를 계속 갱신한다.**

---

# 8. Loop와 Program 종료도 검사한다

Memory Access가 안전하더라도 eBPF Program이 끝없이 실행되면 Kernel Resource를 계속 점유할 수 있다.

그래서 Verifier는 Program이 제한된 복잡도 안에서 끝까지 도달할 수 있는지도 확인한다.

책에서 사용하는 Kernel 기준으로 Verifier는 실행 경로를 분석할 때 최대 1,000,000 Instruction의 Complexity Limit을 사용한다고 설명한다.

이 값은 Source Code 줄 수나 Program 자체의 단순 Instruction 개수를 의미하는 것이 아니라, **Verifier가 Branch와 Loop의 여러 State를 분석하면서 처리하는 Instruction 수와 관련된 제한**이다.

Kernel 5.3 이전에는 Loop를 만들기 위해 필요한 Backward Jump가 허용되지 않아 `#pragma unroll`을 이용해 Loop를 Compiler 단계에서 펼치는 방식을 많이 사용했다.

```c
#pragma unroll
for (int i = 0; i < 3; i++) {
    A();
}
```

개념적으로는 Bytecode에 다음과 같이 반복 코드를 직접 만드는 방식이다.

```c
A();
A();
A();
```

Kernel 5.3 이후에는 Verifier가 Backward Branch를 분석할 수 있게 되면서 **Bounded Loop**, 즉 유한한 반복이라고 증명할 수 있는 Loop를 허용할 수 있게 됐다.

```c
for (int i = 0; i < 10; i++) {
    bpf_printk("Looping %d", i);
}
```

또한 Kernel 5.17에서는 `bpf_loop()` Helper가 추가되어 반복되는 Callback의 Instruction을 효율적으로 검증할 수 있게 됐다.

여기서 중요한 것은 단순히 "eBPF에서 Loop를 사용할 수 있다/없다"가 아니라:

> **Verifier가 Loop가 유한하게 끝나며 분석 가능한 복잡도 안에 있다고 증명할 수 있어야 한다.**

는 점이다.

---

# 9. Return Value와 Instruction도 확인한다

eBPF Program의 Return Value는 R0에 저장된다.

Program이 종료될 때 R0가 초기화되지 않았다면 다음과 같은 Error가 발생할 수 있다.

```text
R0 !read_ok
```

한 가지 흥미로운 점은 Helper Function의 Return Value 역시 R0를 사용한다는 것이다.

```int prog(void *ctx)
{
    bpf_get_current_pid_tgid();
}
```

이 코드를 예로 들면:

```bpf_get_current_pid_tgid()
        ↓
Helper Return Value가 R0에 저장됨
        ↓
Program 종료 시
R0가 NOT_INIT 상태는 아님
```

즉 Source Code에서 명시적인 `return`이 없더라도 직전에 Helper를 호출했다면 Helper Return Value가 R0에 들어가 있기 때문에 단순한 `R0 !read_ok` Error가 발생하지 않을 수도 있다.

하지만 이것은 **`return`을 생략해도 된다는 뜻이 아니다.**

이 예제가 보여주는 핵심은 Verifier가 C Source의 `return` 문 자체를 보는 것이 아니라 **Bytecode 수준에서 R0의 상태가 유효한지를 추적한다는 것**이다.

또한 Verifier는 Program이 Kernel이 이해할 수 있는 올바른 eBPF Bytecode Instruction으로 구성되어 있는지도 확인한다. 오래된 Kernel이 지원하지 않는 새로운 eBPF Instruction을 사용한다면 Verification에 실패할 수 있다.

---

# 정리

이번 Chapter를 읽기 전에는 Verifier를 단순히 "Kernel에 Load하기 전에 안전한지 검사하는 단계" 정도로 생각했다.

하지만 내부 동작을 따라가보면 Verifier는 eBPF Bytecode의 가능한 실행 경로를 분석하면서 **각 Register의 Type과 Value Range를 계속 추적하는 정적 분석기**에 가깝다.

아래 흐름은 Kernel 내부에서 반드시 이 순서대로 독립적인 Phase가 실행된다는 뜻이 아니라, Chapter 6에서 살펴본 주요 검증 항목을 개념적으로 정리한 것이다.

```text
eBPF Bytecode
      │
      ▼
Control Flow 분석
      │
      ▼
Register State 추적
      │
      ├── Type
      └── Value Range
      │
      ▼
안전성 검증
      │
      ├── Helper 사용 규칙
      ├── Pointer / NULL
      ├── Packet Boundary
      ├── Array Bounds
      ├── Context Access
      ├── Loop / 종료 가능성
      └── Return / Instruction
      │
      ▼
모든 가능한 경로가 안전한가?
      │
 ┌────┴────┐
 │         │
YES       NO
 │         │
 ▼         ▼
Load      Reject
```

이번 Chapter에서 가장 중요하게 이해한 부분은 다음 한 문장으로 정리할 수 있었다.

> **Verifier는 실제로 문제가 발생했는지를 확인하는 것이 아니라, 모든 가능한 실행 경로에서 문제가 발생하지 않는다고 증명할 수 있는지를 확인한다.**

예를 들어 실제 Runtime에서는 `p`가 항상 정상 Pointer일 것 같더라도 Verifier가 NULL 가능성을 제거하지 못하면 Dereference를 허용하지 않는다. 마찬가지로 실제로 `c=12`가 자주 발생하지 않더라도 `c`의 가능한 Range에 12가 포함되어 있다면 Array Out-of-Bounds 가능성이 있기 때문에 Reject한다.

앞으로 Verifier Error를 확인할 때는 Error Message만 보는 것이 아니라 다음 흐름으로 추적하면 원인을 찾기 쉬울 것 같다.

```text
어느 Instruction에서 실패했는가?
        ↓
어떤 Register가 문제인가?
        ↓
Verifier는 해당 Register를 어떤 Type으로 보고 있는가?
        ↓
가능한 Value Range는 얼마인가?
        ↓
그 값은 어디에서 만들어졌는가?
        ↓
어떤 Branch를 지나 현재 State가 되었는가?
```

결국 eBPF가 Kernel 내부에서 강력한 기능을 수행하면서도 안전성을 유지할 수 있는 중요한 이유 중 하나가 바로 이 Verifier에 있다.

---

### 출처

- *Learning eBPF* — Liz Rice, Chapter 6: The eBPF Verifier
