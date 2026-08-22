---
layout: post
title: "[Learning eBPF] Chapter 6: The eBPF Verifier"
date: 2026-08-22 19:05:00 +0900
categories: [eBPF, Linux, Kernel]
tags: [eBPF, BPF, Verifier, eBPF Verifier, BPF Helper, XDP, BPF Map, Kernel]
published: true
---

# Learning eBPF Chapter 5

## The eBPF Verifier

이번 Chapter 6에서는 eBPF 프로그램이 Kernel에서 실제로 실행되기 전에 반드시 통과해야 하는 **eBPF Verifier**에 대해 다룬다.

처음 eBPF를 공부하면서 Verifier에 대해서는 단순히 "eBPF 프로그램이 안전한지 검사하는 단계" 정도로만 이해하고 있었다.

그런데 이번 Chapter를 읽어보니 Verifier는 생각보다 훨씬 많은 일을 하고 있었다.

프로그램의 명령어만 훑어보는 것이 아니라 **가능한 실행 경로를 따라가면서 eBPF Register의 상태, Pointer의 종류, 값의 범위, Memory 접근 범위 등을 계속 추적하고, 어떤 실행 경로에서도 위험한 동작이 발생하지 않는지를 확인한다.**

특히 eBPF 프로그램은 일반적인 User Space 프로그램이 아니라 **Linux Kernel 내부에서 실행되는 코드**이기 때문에 이러한 검증 과정이 중요하다.

잘못된 Pointer로 Kernel Memory에 접근하거나, 배열 범위를 벗어나거나, 프로그램이 끝나지 않고 계속 실행된다면 단순히 하나의 Process가 Crash 나는 것으로 끝나는 것이 아니라 Kernel 전체의 안정성에 영향을 줄 수 있기 때문이다.

---

# 1. eBPF 프로그램은 컴파일됐다고 바로 실행되는 것이 아니다

먼저 eBPF 프로그램이 실행되기까지의 전체 흐름을 다시 정리해보면 다음과 같다.

```text
eBPF C Source
     │
     │ Clang / LLVM
     ▼
eBPF Bytecode
(.o Object File)
     │
     │ bpf() syscall
     │ BPF_PROG_LOAD
     ▼
┌──────────────────────┐
│    eBPF Verifier     │
│                      │
│  프로그램 안전성 검사     │
└──────────┬───────────┘
           │
       검증 성공
           │
           ▼
     JIT Compilation
           │
           ▼
      Machine Code
           │
           ▼
 Kernel Hook에 연결
           │
           ▼
          실행
```

![eBPF 프로그램의 전체 실행 구조](/img/study-Learning_eBPF/eBPF%20프로그램의%20전체%20실행%20구조.png)

---

여기서 중요한 것은 **Compile 성공과 Verifier 통과는 전혀 다른 단계**라는 것이다.

예를 들어 우리가 다음과 같은 파일을 작성했다고 하자.

```text
hello.bpf.c
```

Clang을 이용해서 이를 컴파일하면

```text
hello.bpf.o
```

라는 Object File이 만들어질 수 있다.

이 `.o` 파일 안에는 Kernel에서 실행할 **eBPF Bytecode**가 들어 있다.

하지만 `.o` 파일이 정상적으로 만들어졌다고 해서 이 프로그램을 바로 Kernel에서 실행할 수 있다는 뜻은 아니다.

```text
C 코드 작성
   │
   ▼
Compiler
   │
   ├── C 문법상 문제가 있는가?
   ├── 타입을 처리할 수 있는가?
   └── Bytecode를 생성할 수 있는가?
   │
   ▼
.o 생성 성공
   │
   │
   │ 아직 끝이 아님
   ▼
Verifier
   │
   ├── Pointer 사용이 안전한가?
   ├── Memory 범위를 벗어나지 않는가?
   ├── Helper 사용이 올바른가?
   ├── 모든 실행 경로가 안전한가?
   └── 프로그램이 정상적으로 끝나는가?
```

따라서

```text
Compile 성공 ≠ Kernel에서 실행 가능
```

이다.

Compiler와 Verifier가 검사하는 목적 자체가 다르기 때문이다.

---

## Verifier가 보는 것은 C Source가 아니다

이번 Chapter에서 특히 중요했던 점은 **Verifier가 우리가 작성한 C 코드를 직접 검사하지 않는다는 것**이다.

Verifier가 실제로 보는 것은 Compiler가 만들어낸 **eBPF Bytecode**이다.

```text
hello.bpf.c
    │
    │ Clang / LLVM
    ▼
hello.bpf.o
    │
    └── eBPF Bytecode
             │
             ▼
          Verifier
```

따라서 Source Code를 조금 수정했다고 해서 반드시 Verifier의 분석 결과가 우리가 예상한 대로 달라지는 것은 아니다.

중간에 **Compiler Optimization**이 있기 때문이다.

예를 들어 실행될 수 없는 코드가 C Source에 존재한다고 하더라도 Compiler가 "이 코드는 어차피 실행될 일이 없다."

라고 판단하여 Bytecode를 생성할 때 제거해버렸다면 Verifier는 그 코드를 아예 보지 못한다.

이 때문에 Verifier 오류를 제대로 이해하려면

```text
C Source
    ↓
Compiler
    ↓
eBPF Bytecode
    ↓
eBPF VM
    ↓
Verifier
```

의 흐름을 같이 생각해야 한다.

---

# 2. 실행하지 않고 어떻게 안전성을 검사할까?

여기서 가장 궁금했던 부분은 이것이었다.

> 프로그램을 실제로 실행하지도 않았는데 어떻게 Memory Access가 안전한지 알 수 있을까?

Verifier는 프로그램을 실제 데이터로 실행하는 것이 아니라 **프로그램의 명령어를 따라가면서 실행 결과를 추론한다.**

이를 **정적 분석(Static Analysis)​**이라고 볼 수 있다.

예를 들어 다음과 같은 코드가 있다고 하자.

```c
if (x > 10) {
    A();
} else {
    B();
}
```

프로그램을 실제로 실행한다면 `x`의 실제 값에 따라서 A와 B 중 하나만 실행된다.

하지만 Verifier는 실제 실행 시점의 `x` 값을 알 수 없다.

따라서 다음 두 경우를 모두 확인해야 한다.

```text
                   x > 10 ?
                   /      \
                 YES      NO
                  │        │
                  ▼        ▼
                 A()      B()
                  │        │
                  └───┬────┘
                      ▼
                    EXIT
```

Verifier는

```text
x > 10인 경로는 안전한가?

x <= 10인 경로는 안전한가?
```

를 각각 분석한다.

그리고 **가능한 실행 경로 중 하나라도 안전하다고 증명할 수 없다면 프로그램 전체를 거부한다.**

여기서 중요한 표현은 단순히 "위험하면 거부한다"보다

> **안전하다는 것을 Verifier가 증명할 수 있어야 한다.**

에 가깝다고 생각한다.

실제로는 안전하게 실행될 가능성이 높더라도 Verifier가 그 사실을 논리적으로 확인할 수 없다면 프로그램은 Load되지 않을 수 있다.

---

# 3. Verifier는 Register를 계속 추적한다

그렇다면 각 실행 경로에서 무엇을 추적할까?

여기서 Chapter 3에서 공부했던 **eBPF Virtual Machine의 Register**가 다시 등장한다.

eBPF VM에는 `R0`부터 `R10`까지 총 11개의 Register가 있다.

```text
┌──────────┬─────────────────────────────────┐
│ Register │ 역할                             │
├──────────┼─────────────────────────────────┤
│ R0       │ 함수 / eBPF Program Return 값     │
│ R1 ~ R5  │ 함수의 Argument                   │
│ R6 ~ R9  │ Callee-saved Register           │
│ R10      │ Stack Frame Pointer             │
└──────────┴─────────────────────────────────┘
```

Verifier는 Bytecode Instruction을 하나씩 따라가면서 이 Register들이 현재 어떤 상태인지 기록한다.

Kernel 내부에서는 이를 위해 `bpf_reg_state`라는 구조체를 사용한다.

여기서 Verifier가 단순히

```text
R2 = 10
```

처럼 현재 값 하나만 기록하는 것은 아니다.

크게 두 가지를 추적한다.

```text
                  Register State
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
         Value Type            Value Range
             │                     │
     어떤 종류의 값인가?     어떤 값을 가질 수 있는가?
```

즉,

**① 이 Register가 어떤 종류의 값을 담고 있는지**

그리고

**② 이 Register가 어느 범위의 값을 가질 수 있는지**

를 함께 추적한다.

---

## Register Type

대표적인 Register Type에는 다음과 같은 것들이 있다.

### `NOT_INIT`

아직 값이 설정되지 않은 Register이다.

즉 Verifier 입장에서는 아직 읽어서는 안 되는 상태이다.

### `SCALAR_VALUE`

Pointer가 아닌 일반적인 숫자 값이다.

예를 들어 정수 계산 결과 등이 여기에 해당할 수 있다.

### `PTR_TO_CTX`

eBPF 프로그램에 전달된 Context를 가리키는 Pointer이다.

eBPF 프로그램이 시작될 때 `R1`에는 일반적으로 해당 프로그램의 Context가 전달된다.

예를 들어 XDP 프로그램이라면 다음과 같다.

```c
int xdp_program(struct xdp_md *ctx)
```

여기서 `ctx`가 처음에는 `R1`에 들어온다.

### `PTR_TO_PACKET`

Network Packet의 Data를 가리키는 Pointer이다.

### `PTR_TO_MAP_VALUE`

BPF Map의 Value를 가리키는 Pointer이다.

### `PTR_TO_MAP_VALUE_OR_NULL`

BPF Map의 Value를 가리키고 있을 수도 있고 `NULL`일 수도 있는 Pointer이다.

뒤에서 살펴볼 `bpf_map_lookup_elem()`에서 특히 중요하다.

---

이 Type Tracking이 중요한 이유는 Verifier 입장에서 Pointer를 단순한 숫자 주소로 보면 안 되기 때문이다.

예를 들어 Register 안에

```text
0xffff888012345678
```

이라는 값이 들어 있다고 생각해보자.

그 숫자만 봐서는 이 주소가

- Packet Memory인지
- Stack인지
- Map Value인지
- Context인지

알 수 없다.

하지만 Verifier는 이를 다음처럼 구분해서 관리한다.

```text
R1 → PTR_TO_CTX

R2 → SCALAR_VALUE

R3 → PTR_TO_PACKET

R6 → PTR_TO_MAP_VALUE
```

덕분에

- 이 Pointer를 이 Helper에 전달해도 되는가?
- 이 Pointer에서 값을 읽어도 되는가?
- Pointer Arithmetic을 해도 되는가?

를 판단할 수 있다.


---

# 4. Type뿐 아니라 값의 범위도 추적한다

Verifier가 추적하는 것은 Type만이 아니다.

Register가 **가질 수 있는 값의 범위**도 추적한다.

예를 들어 어떤 시점에 Verifier가 다음과 같이 알고 있다고 해보자.

```text
R2 = 0 ~ 100
```

그리고 Bytecode에서

```text
R3 = R2
R3 += 1
```

이 수행된다면 Verifier는 다음과 같이 추론할 수 있다.

```text
R2
0 ───────────────────── 100

        copy
          ↓

R3
0 ───────────────────── 100

        + 1
          ↓

R3
1 ───────────────────── 101
```

Verifier Log에서는 이러한 정보가 `umin_value`, `umax_value` 등으로 나타난다.

예를 들어 책에서는 다음과 같은 상태를 확인한다.

```text
R2_w=inv(
    umax_value=4294967295,
    var_off=(0x0; 0xffffffff)
)
```

그리고 R2를 R3에 복사한 다음 1을 더하면 R3의 최소값 역시 변경된다.

이 범위 정보가 중요한 이유는 **Memory Bounds Check** 때문이다.

예를 들어 배열의 크기가 12인데 Index로 사용하는 Register가

```text
0 ~ 12
```

까지 가능하다면 Verifier는

> 12가 들어오는 경우 배열의 끝을 넘어갈 수 있다.

고 판단할 수 있다.

즉 Verifier가 C Source의 변수 이름을 알고 판단하는 것이 아니라 **그 변수가 컴파일된 이후 어떤 Register에 들어가 있고 그 Register가 어느 범위의 값을 가질 수 있는지를 추적하면서 안전성을 판단하는 것이다.**

---

# 5. 조건문에서는 Register 상태도 갈라진다

이제 Type과 Range Tracking을 알고 나면 Verifier의 Branch 분석도 조금 더 이해하기 쉽다.

예를 들어 다음 코드가 있다고 하자.

```c
if (value < 10) {
    ...
}
```

조건문 이전까지 Verifier가

```text
R2 = 0 ~ 100
```

이라고 알고 있었다면 Branch 이후에는 두 개의 서로 다른 상태가 생긴다.

```text
                    R2 = 0 ~ 100
                          │
                     R2 < 10 ?
                    /        \
                  YES        NO
                   │          │
                   ▼          ▼
              R2 = 0~9    R2 = 10~100
```

즉 조건문은 단순히 실행 위치만 나누는 것이 아니라 **Verifier가 알고 있는 Register의 상태도 좁혀준다.**

이 부분은 뒤에서 살펴볼 NULL Pointer 검사에서 매우 중요하게 사용된다.

---

## 모든 경로를 무조건 다시 검사하는 것은 아니다 - State Pruning

그런데 Program에 `if`, `else`, Loop 등이 많아지면 가능한 실행 경로가 기하급수적으로 많아질 수 있다.

Verifier가 모든 경로를 처음부터 끝까지 계속 다시 검사한다면 검증 자체가 너무 비싸질 수 있다.

그래서 사용하는 최적화가 **State Pruning**이다.

예를 들어 Verifier가 이전에 Instruction 100에 다음 상태로 도달해서 이후 경로를 이미 검증했다고 하자.

```text
Instruction 100

R1 = PTR_TO_CTX
R2 = 0 ~ 10
R3 = SCALAR_VALUE
```

이후 다른 실행 경로를 따라가다 다시 Instruction 100에 왔는데 Register 상태 역시 이미 검증한 상태와 동일하거나, 이전에 검증한 상태 안에 포함되는 더 제한적인 상태라면 어떻게 될까?

```text
Path A
   │
   ▼
Instruction 100
State X
   │
   ▼
이후 경로 검증 완료


Path B
   │
   ▼
Instruction 100
State X
   │
   ▼
"이 상태는 이미 검증했다."
   │
   ▼
State Pruning
```

이후 경로를 다시 검증할 필요가 없다.

이것이 **State Pruning**이다.

즉 "모든 실행 경로를 확인한다"는 말을 정말 모든 경로를 무식하게 끝까지 반복해서 실행해본다는 의미로 이해하면 안 된다.

Verifier는 이전에 검증한 Register와 Stack State를 이용해서 **동일하거나 이미 안전성이 포함된 상태를 다시 만났을 때 중복 분석을 생략한다.**

---

# 6. Control Flow를 직접 눈으로 확인해보기

Verifier가 Branch를 따라간다고 설명해도 Bytecode에 익숙하지 않으면 처음에는 조금 추상적으로 느껴질 수 있다.

이럴 때 `bpftool`을 이용해 실제 eBPF 프로그램의 **Control Flow Graph**를 만들어볼 수 있다.

```bash
bpftool prog dump xlated name kprobe_exec visual > out.dot
```

`visual` 옵션을 사용하면 DOT 형식으로 Control Flow를 출력할 수 있다.

이를 Graphviz의 `dot`으로 변환하면 PNG 이미지를 만들 수 있다.

```bash
dot -Tpng out.dot > out.png
```

![eBPF Control Flow Graph](/img/study-Learning_eBPF/eBPF%20Control%20Flow%20Graph.png)


위 그림을 실제로 보면 Verifier가 분석한다는 "실행 경로"가 조금 더 명확해진다.

예를 들어 다음 Instruction에서

```text
28: if r0 == 0x0 goto pc+1
```

조건에 따라 실행 경로가 갈라진다.

한 경로에서는 Instruction 29를 거친 뒤 30으로 이동하고, 다른 경로에서는 바로 다음 Block으로 이동한다.

그리고 아래의

```text
37: if r8 == 0x0 goto pc+4
```

에서도 다시 실행 경로가 두 개로 나뉜다.

이 각각의 사각형은 여러 eBPF Instruction으로 구성된 **Basic Block**이고, 화살표는 해당 Block 사이에서 가능한 Control Flow를 나타낸다.

따라서 Verifier는 단순히

```text
23 → 24 → 25 → 26 → ...
```

처럼 Bytecode를 한 줄로만 읽는 것이 아니다.

```text
                   Block
                     │
             ┌───────┴───────┐
             ▼               ▼
          Branch A         Branch B
             │               │
             └───────┬───────┘
                     ▼
                  Next Block
```

처럼 실제 Control Flow를 따라가면서 각 경로의 Register State를 따로 분석한다.

이번 Chapter에서 State Pruning을 먼저 이해하고 이 그림을 보니 **Verifier가 왜 프로그램의 Control Flow와 Register State를 함께 추적해야 하는지**가 조금 더 명확해졌다.

---

# 7. Verifier Log

eBPF 프로그램이 Verifier를 통과하지 못했을 때 가장 중요한 디버깅 자료가 **Verifier Log**이다.

`bpftool prog load`로 프로그램을 Load하면 Verification 실패 시 로그가 `stderr`에 출력된다.

libbpf를 사용하는 프로그램에서는 `libbpf_set_print()` 등을 통해 libbpf가 출력하는 메시지를 처리할 수도 있다.

책의 성공한 프로그램에서는 다음과 같은 Verifier Summary가 나온다.

```text
processed 61 insns (limit 1000000)
max_states_per_insn 0
total_states 4
peak_states 4
mark_read 3
```

여기서 `processed 61 insns`는 Verifier가 분석 과정에서 처리한 Instruction 수를 나타낸다.

Branch를 서로 다른 상태로 여러 번 방문할 수 있기 때문에 이것을 단순히 `.o` 안에 존재하는 Instruction 개수와 동일하다고 생각하면 안 된다.

그리고 여기서 보이는 `1,000,000`은 Verifier가 프로그램의 복잡도를 분석할 때 적용되는 처리 한도와 관련된 값이다.

---

## `-g` 옵션이 있으면 Source Code까지 같이 볼 수 있다

책의 Verifier Log를 보면 이런 식으로 C Source가 함께 나온다.

```text
; c++;

4: (bf) r3 = r2
5: (07) r3 += 1
6: (63) *(u32 *)(r1 +0) = r3
```

위의

```text
; c++;
```

가 원래 C 코드이고 그 아래가 실제 eBPF Instruction이다.

이것이 가능한 이유는 Compile할 때 `-g` 옵션을 사용해서 **Debug Information**을 Object File에 포함했기 때문이다.

따라서

```text
C Source
   │
   ▼
어떤 eBPF Instruction으로 변환됐는가?
```

를 같이 확인할 수 있다.

Verifier Error를 디버깅할 때는 이 정보가 상당히 도움이 된다.

---

# 8. 왜 R1의 Context를 R6에 복사할까?

Verifier Log 첫 부분을 보면 다음 Instruction이 나온다.

```text
0: (bf) r6 = r1
```

처음에는 '왜 굳이 R1의 값을 R6으로 복사하지?' 라는 생각이 들 수 있다.

eBPF Program이 호출될 때 R1에는 **Context Pointer**가 들어온다.

```text
Program 시작

R1
 │
 └── Context
```

그런데 BPF Helper Function을 호출할 때는 R1~R5가 Helper의 Argument 전달에 사용된다.

```text
Helper Function Call

R1 → Argument 1
R2 → Argument 2
R3 → Argument 3
R4 → Argument 4
R5 → Argument 5
```

Helper Call 이후 R1~R5의 기존 값을 그대로 보존한다고 기대할 수 없다.

반면 R6~R9는 **callee-saved register**이므로 Helper Call 이후에도 값이 유지된다.

그래서 Compiler가 Context Pointer를 계속 사용해야 할 경우 다음처럼 R6에 저장해놓을 수 있다.

```text
Program 시작

R1 = Context
 │
 │ copy
 ▼
R6 = Context
 │
 │
 ├──── Helper Call
 │
 ▼
R6 = Context 그대로 유지
```

---

# 9. Helper Function도 아무 것이나 호출할 수 없다

eBPF 프로그램은 Kernel 안에서 실행된다고 해서 Kernel Function을 마음대로 호출할 수 있는 것은 아니다.

Kernel에서 제공하는 **BPF Helper Function**을 통해 정해진 기능을 사용할 수 있다.

예를 들어

```c
bpf_map_lookup_elem()
bpf_get_current_pid_tgid()
bpf_probe_read_kernel()
bpf_printk()
```

같은 함수가 있다.

그런데 중요한 점은 **모든 BPF Program Type에서 모든 Helper를 사용할 수 있는 것은 아니라는 것**이다.

예를 들어

```c
bpf_get_current_pid_tgid()
```

는 현재 실행 중인 Process의 PID와 TGID를 가져오는 Helper이다.

Kprobe처럼 Process가 Kernel Function을 실행하는 시점과 연결된 Program에서는 자연스러운 정보이다.

하지만 XDP 프로그램은

```text
Network Interface
       │
       ▼
    Packet 도착
       │
       ▼
      XDP
```

시점에 실행된다.

특정 User Space Process와 반드시 연결되어 있는 상황이 아니다.

그래서 기존 Kprobe Program의 Section을 XDP로 변경한 뒤 이 Helper를 사용하면 다음과 같은 오류가 발생한다.

```text
16: (85) call bpf_get_current_pid_tgid#14
unknown func bpf_get_current_pid_tgid#14
```

여기서 `unknown func`는 Kernel에 이 Helper가 존재하지 않는다는 의미가 아니라

> **현재 BPF Program Type에서는 이 Helper를 사용할 수 없다.**

는 의미이다.

---

# 10. Helper에 넘기는 Argument까지 확인한다

Helper를 사용할 수 있다고 끝나는 것도 아니다.

각 Helper는 **어떤 종류의 Argument를 받아야 하는지와 어떤 종류의 값을 반환하는지가 정의되어 있다.**

예를 들어 `bpf_map_lookup_elem()`은 Kernel 내부에서 다음과 비슷한 Prototype 정보를 가지고 있다.

```c
const struct bpf_func_proto bpf_map_lookup_elem_proto = {
    .func      = bpf_map_lookup_elem,
    .gpl_only  = false,
    .pkt_access = true,
    .ret_type  = RET_PTR_TO_MAP_VALUE_OR_NULL,
    .arg1_type = ARG_CONST_MAP_PTR,
    .arg2_type = ARG_PTR_TO_MAP_KEY,
};
```

이 중 핵심만 보면 다음과 같다.

```text
bpf_map_lookup_elem()

Argument 1
    ↓
Map Pointer

Argument 2
    ↓
Map Key Pointer

Return
    ↓
Map Value Pointer
OR
NULL
```

그래서 정상적인 코드는 다음과 같다.

```c
p = bpf_map_lookup_elem(&my_config, &uid);
```

![eBPF Program ↔ Map 관계](/img/study-Learning_eBPF/eBPF%20Program와%20Map%20관계.png)

BPF Map은 Kernel 영역에 존재하면서 eBPF Program과 User Space Application이 데이터를 주고받는 데 사용된다.

eBPF Program이 Map을 직접 아무 주소로나 접근하는 것이 아니라 정해진 Helper를 통해 Value를 Lookup한다는 점이다.

---

그런데 다음처럼 Map이 아닌 Local Variable의 주소를 첫 번째 Argument로 전달했다고 하자.

```c
p = bpf_map_lookup_elem(&data, &uid);
```

C Compiler 입장에서는 이 코드가 컴파일될 수도 있다.

따라서 `.o` 파일도 만들어진다.

하지만 Kernel에 Load할 때 Verifier는 다음 오류를 발생시킨다.

```text
R1 type=fp expected=map_ptr
```

이를 풀어보면 다음과 같다.

```text
bpf_map_lookup_elem()
        │
        │ Argument 1 = R1
        ▼

Verifier가 확인한 R1
        │
        └── fp
            Frame / Stack 쪽 Pointer

Helper가 요구하는 R1
        │
        └── map_ptr
            Map Pointer

        서로 다름
            │
            ▼
         Reject
```

Verifier가 Register Type을 계속 추적하고 있었기 때문에 가능한 검사이다.

이 사례에서도 다시 한 번

```text
Compiler가 받아준 코드

        ≠

eBPF에서 안전한 코드
```

라는 사실을 확인할 수 있다.

---

# 11. GPL License까지 확인한다

Verifier는 Helper를 사용할 때 License 조건도 확인한다.

eBPF Program 끝에서는 다음과 같이 License Section을 선언하는 경우가 많다.

```c
char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

Helper 중 일부는 GPL-Compatible Program에서만 사용할 수 있다.

책에서 사용하는

```c
bpf_probe_read_kernel()
```

이 그런 Helper에 해당한다.

License 선언을 제거하고 해당 Helper를 호출하면 다음과 같은 오류가 발생한다.

```text
cannot call GPL-restricted function
from non-GPL compatible program
```

즉 Verifier가 확인하는 범위는 단순히

> Memory를 이상하게 접근하는가?

정도에 그치지 않는다.

해당 Program Type에서 Helper를 사용할 수 있는지, Argument가 맞는지, License 조건을 만족하는지까지 Helper의 **호출 계약(Contract)** 을 검사한다.

---

# 12. 가장 중요한 Memory Access 검사

Verifier가 수행하는 검사 중 가장 중요한 부분 중 하나가 **Memory Safety**이다.

eBPF Program은 Kernel 내부에서 실행되지만 아무 Kernel Memory나 자유롭게 접근할 수 있는 것이 아니다.

특히 XDP 프로그램을 보면 이 구조를 이해하기 쉽다.

XDP Program에는 다음과 같은 Context가 전달된다.

```c
struct xdp_md *ctx
```

그리고 보통 Packet의 시작과 끝을 다음처럼 가져온다.

```c
void *data =
    (void *)(long)ctx->data;

void *data_end =
    (void *)(long)ctx->data_end;
```

`data`는 Packet의 시작을 가리키고 `data_end`는 Packet의 끝을 나타낸다.

```text
                 XDP Packet Buffer

ctx->data
   │
   ▼
┌────────────┬────────────┬────────────┬─────────────┐
│ Ethernet   │ IPv4       │ TCP / UDP  │ Payload     │
│ Header     │ Header     │ Header     │             │
└────────────┴────────────┴────────────┴─────────────┘
                                                   ▲
                                                   │
                                             ctx->data_end
```

따라서 XDP Program이 Packet Header를 읽으려면 읽으려는 데이터가 이 범위 안에 있다는 사실을 먼저 확인해야 한다.

인터넷에서 참고한 XDP Packet Buffer 그림에서도 `xdp_md`의 `data`와 `data_end`가 Packet Buffer의 시작과 끝을 나타내고 그 사이에 Ethernet, IP, TCP Header와 Payload가 배치되는 구조를 확인할 수 있다.

---

예를 들어 Packet을 읽을 때 Verifier에게 결국 다음 사실을 보여줘야 한다.

```text
읽으려는 주소 + 읽을 크기
               │
               ▼
          data_end 이하인가?
```

이것이 증명되어야 Packet Access가 허용된다.

---

## 그러면 `data_end`를 늘려버리면 되지 않을까?

당연히 안 된다.

예를 들어 다음처럼 작성한다고 해보자.

```c
data_end++;
```

Verifier는 이를 다음처럼 거부한다.

```text
R3 pointer arithmetic on pkt_end prohibited
```

왜냐하면 `data_end` 자체를 프로그램이 마음대로 변경할 수 있다면

```text
실제 Packet End
       │
       ▼
[ Packet ]

프로그램이 data_end를 변경

[ Packet ][ 실제 존재하지 않는 영역 ]
                               ▲
                               │
                       가짜 Packet End
```

처럼 Bounds Check 자체를 우회할 수 있기 때문이다.

Verifier는 `PTR_TO_PACKET_END`에 대한 이런 Pointer Arithmetic을 허용하지 않는다.

---

# 13. 배열에서 딱 한 칸만 잘못돼도 잡아낸다

책에는 Verifier의 Range Tracking이 실제로 어떻게 사용되는지 보여주는 좋은 예제가 있다.

다음과 같은 배열이 있다고 하자.

```c
char message[12];
```

12개의 Character를 가지고 있으므로 유효한 Index는

```text
0 ~ 11
```

이다.

```text
Index

 0   1   2   3   4   5   6   7   8   9  10  11
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│   │   │   │   │   │   │   │   │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘

                                               12
                                                ↓
                                      ❌ Out of Bounds
```

따라서 다음 코드는 안전하다.

```c
if (c < sizeof(message)) {
    char a = message[c];
    bpf_printk("%c", a);
}
```

`sizeof(message)`가 12라면 `c`는 최대 11까지만 들어간다.

그런데 다음처럼 실수했다고 하자.

```c
if (c <= sizeof(message)) {
    char a = message[c];
}
```

얼핏 보면 `<` 대신 `<=` 하나만 바뀌었을 뿐이다.

하지만 이제 `c == 12`도 조건을 통과한다.

따라서

```c
message[12]
```

가 실행될 가능성이 존재한다.

Verifier는 이를 다음처럼 거부한다.

```text
invalid access to map value,
value_size=16 off=16 size=1

R2 max value is outside of the allowed memory range
```

이런 실수를 **Off-by-One Error**라고 한다.

이 사례가 흥미로운 이유는 Verifier가 프로그램을 실제로 `c=12`로 실행해본 것이 아니기 때문이다.

그동안 Register State를 추적한 결과

```text
이 Register의 최대값이 12가 될 수 있다.
        │
        ▼
12가 Index로 사용될 수도 있다.
        │
        ▼
배열은 0~11까지만 유효하다.
        │
        ▼
Out-of-Bounds 가능
        │
        ▼
Reject
```

라고 **정적으로 추론**한 것이다.

---

## 그런데 왜 Array인데 `map value` 오류가 나올까?

처음 오류를 보면 조금 이상하다.

우리가 접근한 것은 `message` 배열인데 오류에는

```text
invalid access to map value
```

라고 나온다.

이유는 Chapter 3에서 공부했던 **eBPF Global Variable 구현 방식**과 관련이 있다.

`message`가 Global Variable로 선언되어 있는데 eBPF의 Global Variable은 내부적으로 BPF Map을 이용해 구현된다.

따라서 C Source에서는

```c
message[c]
```

처럼 보이지만 Verifier가 실제 Bytecode를 분석할 때는 **Map Value Memory에 접근하는 형태**로 보일 수 있다.

그래서 오류 메시지가 Map Value Access라고 나타난다.

이 부분 역시 Verifier를 이해할 때 Source Code만 보는 것이 아니라

```text
C Source
   ↓
Compiler
   ↓
실제 eBPF Bytecode
```

까지 같이 생각해야 하는 이유이다.

---

# 14. `bpf_map_lookup_elem()`과 NULL Pointer

```c
p = bpf_map_lookup_elem(&my_config, &uid);
```

이 Helper는 Map에서 `uid`에 해당하는 Value를 찾는다.

문제는 해당 Key가 Map에 없을 수도 있다는 것이다.

따라서 반환값은 두 가지 가능성을 가진다.

```text
bpf_map_lookup_elem()
           │
           ▼
          R0
           │
     ┌─────┴──────┐
     │            │
     ▼            ▼
Map Value        NULL
 Pointer
```

그래서 Helper의 Return Type이

```text
RET_PTR_TO_MAP_VALUE_OR_NULL
```

로 정의되어 있다.

즉 Verifier 입장에서는 반환 직후 Pointer의 상태가

```text
PTR_TO_MAP_VALUE_OR_NULL
```

이다.

```text
bpf_map_lookup_elem()
          │
          ▼
PTR_TO_MAP_VALUE_OR_NULL
          │
          ▼
      p != NULL ?
       /       \
     YES       NO
      │         │
      ▼         ▼
PTR_TO_MAP    NULL
_VALUE
      │
      ▼
Dereference 가능
```

그런데 NULL Check 없이 바로 다음처럼 접근한다고 하자.

```c
char a = p->message[0];
```

C Compiler는 이 코드를 컴파일할 수 있다.

하지만 Verifier는 거부한다.

```text
R7 invalid mem access 'map_value_or_null'
```

왜냐하면 Verifier가 R7을 다음 상태로 알고 있기 때문이다.

```text
R7
 │
 └── PTR_TO_MAP_VALUE_OR_NULL
```

즉 `NULL`일 가능성이 아직 남아 있다.

이 Pointer를 Dereference하면 Kernel에서 잘못된 Memory Access가 발생할 수 있다.

---

## NULL Check를 하면 Verifier가 알고 있는 Type도 바뀐다

다음처럼 코드를 변경한다.

```c
if (p != 0) {
    char a = p->message[0];
}
```

이제 Verifier가 Branch를 따라갈 때 다음처럼 상태가 바뀐다.

```text
                  p
                  │
         MAP_VALUE_OR_NULL
                  │
              p != NULL
              /       \
            TRUE      FALSE
             │          │
             ▼          ▼
        MAP_VALUE      NULL
             │
             ▼
     Pointer Dereference
            허용
```

여기서 중요했던 점은

```c
if (p != NULL)
```

이 단순히 C 프로그램에서 Crash를 막기 위한 조건문으로만 존재하는 것이 아니라는 것이다.

**Verifier에게 새로운 정보를 제공하는 조건문이기도 하다.**

조건을 통과한 TRUE Branch에서는

```text
p는 NULL이 아니다.
```

라는 사실이 증명되었기 때문에 Verifier가 해당 Pointer를 안전한 Map Value Pointer로 다룰 수 있게 된다.

Linux Kernel 공식 문서에서도 `PTR_TO_MAP_VALUE_OR_NULL` Type이 NULL 여부를 검사한 후 `PTR_TO_MAP_VALUE`로 변경된다고 명시하고 있다.

이 사례를 통해

```text
Control Flow Analysis
        +
Register Type Tracking
        +
Pointer Safety
```

가 실제로 서로 어떻게 연결되는지를 확인할 수 있었다.

---

# 15. 그렇다면 Kernel Pointer는 어떻게 읽을까?

일부 경우에는 Kernel Pointer에서 데이터를 읽어야 할 때가 있다.

이때 사용하는 Helper 중 하나가

```c
bpf_probe_read_kernel()
```

이다.

Signature는 다음과 같다.

```c
long bpf_probe_read_kernel(
    void *dst,
    u32 size,
    const void *unsafe_ptr
);
```

특히 세 번째 Argument 이름이

```text
unsafe_ptr
```

인 것이 눈에 띈다.

일반적인 Pointer는 직접 Dereference하기 전에 안전성이 증명되어야 하지만, `bpf_probe_read_kernel()`은 잠재적으로 안전하지 않은 Kernel Pointer에서 데이터를 읽기 위한 Helper이다.

즉 eBPF가 위험한 Kernel Memory를 아무 방식으로나 직접 읽게 두는 것이 아니라

```text
잠재적으로 위험한 Kernel Pointer
               │
               ▼
     bpf_probe_read_kernel()
               │
               ▼
       Kernel이 통제한 방식으로
           Memory Read
```

처럼 정해진 Helper를 이용하게 하는 것이다.

이것도 eBPF의 안전성을 유지하는 중요한 설계 방식이라고 볼 수 있다.

---

# 16. Context라고 해서 전부 읽을 수 있는 것도 아니다

모든 eBPF Program은 실행될 때 어느 정도의 Context를 전달받는다.

예를 들어 XDP는 `struct xdp_md`, Tracepoint Program은 Tracepoint에 해당하는 Context Data를 전달받는다.

그렇다고 Context 구조 안의 모든 Memory를 마음대로 읽을 수 있는 것은 아니다.

Program Type과 Attachment Type에 따라 **허용된 Context Field가 정해져 있다.**

허용되지 않은 영역에 접근하면 다음과 같은 Verifier Error가 발생할 수 있다.

```text
invalid bpf_context access
```

즉 Verifier는 단순히

```text
"R1은 Context Pointer다."
```

까지만 확인하는 것이 아니다.

```text
R1 = Context Pointer
       │
       ▼
현재 Program Type은 무엇인가?
       │
       ▼
이 Context Offset에 접근할 권한이 있는가?
```

까지 판단한다.

---

# 17. 프로그램이 끝나는 것도 안전성의 일부다

Memory Access가 모두 안전해도 eBPF 프로그램이 영원히 실행된다면 문제가 된다.

eBPF는 Kernel 내부에서 실행되기 때문에 무한 루프에 빠져 CPU를 계속 점유한다면 시스템에 큰 영향을 줄 수 있다.

따라서 Verifier는 **프로그램이 제한된 복잡도 안에서 정상적으로 종료될 수 있는지**도 확인한다.

책에서 설명하는 Verifier의 Complexity Processing Limit은

```text
1,000,000 Instructions
```

수준이다.

즉 Branch나 Loop 때문에 가능한 실행 경로를 계속 분석하다가 Verifier가 처리할 수 있는 복잡도 한도를 넘어서면 프로그램을 거부할 수 있다.

---

# 18. 그래서 과거에는 Loop를 쓰기 어려웠다

일반적인 Loop를 Bytecode 수준에서 생각해보면 이전 Instruction으로 돌아가는 **Backward Jump**가 필요하다.

```text
           ┌───────────────────┐
           │                   │
           ▼                   │
     Loop Condition            │
           │                   │
           ▼                   │
       Loop Body               │
           │                   │
           └── Backward Jump ──┘
```

과거 Verifier는 이런 Backward Jump를 허용하지 않았다.

Kernel 5.3 이전에는 일반적인 Loop 사용이 제한되어 있었기 때문에 eBPF Programmer들이 흔히 사용한 방법이 **Loop Unrolling**이었다.

```c
#pragma unroll

for (int i = 0; i < 3; i++) {
    A();
}
```

Compiler에게 실제 Loop를 만들지 말고 반복되는 코드를 펼쳐달라고 요청하는 것이다.

개념적으로

```text
for (3번) {
    A();
}
```

를

```text
A();
A();
A();
```

로 바꾸는 것이다.

따라서 Bytecode에는 Backward Jump가 필요하지 않다.

단점은 반복 횟수만큼 Instruction이 증가한다는 것이다.

---

# 19. Kernel 5.3 이후에는 Bounded Loop 사용 가능

Kernel 5.3 이후에는 Verifier가 일부 Backward Branch를 분석할 수 있게 되면서 **종료 범위를 증명할 수 있는 Bounded Loop**를 사용할 수 있게 되었다.

예를 들어

```c
for (int i = 0; i < 10; i++) {
    bpf_printk("Looping %d", i);
}
```

에서는 반복 횟수가 최대 10회라는 것을 알 수 있다.

```text
i = 0
  │
  ▼
i = 1
  │
  ▼
i = 2
  │
 ...
  │
  ▼
i = 9
  │
  ▼
EXIT
```

따라서 Verifier가 이 Loop가 끝날 수 있다는 것을 분석할 수 있다.

다만 Loop 안에 많은 Branch가 존재하거나 반복 가능한 범위가 너무 크다면 가능한 Program State가 크게 늘어나면서 Verifier Complexity가 증가한다.

즉

```text
"Bounded Loop니까 무조건 가볍다"
```

는 의미는 아니다.

---

# 20. `bpf_loop()`가 등장한 이유

Linux Kernel 5.17에서는 Loop를 보다 쉽게 사용할 수 있도록

```c
bpf_loop()
```

Helper가 추가되었다.

개념적으로는 다음과 같다.

```text
bpf_loop(
    최대 반복 횟수,
    callback,
    ...
)
```

Verifier 입장에서도 일반 Loop의 모든 반복 State를 계속 분석하는 것보다 Callback의 안전성을 확인하는 방식이 훨씬 효율적일 수 있다.

```text
일반 Bounded Loop

Iteration 1
    │
Iteration 2
    │
Iteration 3
    │
...
    ▼
각 상태 분석


bpf_loop()

Callback
   │
   └── 안전성 검증

Maximum Iterations
   │
   └── 상한 존재
```

비슷하게 BPF Map의 Element를 순회할 때 사용할 수 있는

```c
bpf_for_each_map_elem()
```

Helper도 있다.

---

# 21. Return 값도 반드시 확인한다

eBPF Program의 Return Value는 **R0 Register**에 저장된다.

따라서 Program이 종료되려고 하는데 R0가 한 번도 초기화되지 않았다면 Verifier는 이를 허용하지 않는다.

대표적인 오류가

```text
R0 !read_ok
```

이다.

예를 들어 다음처럼 아무 것도 반환하지 않는 XDP Program이 있다고 하자.

```c
SEC("xdp")
int xdp_hello(struct xdp_md *ctx)
{
}
```

Program Exit 시점에 올바른 Return 값이 설정되어 있다는 보장이 없다.

정상적인 XDP Program이라면 예를 들어

```c
return XDP_PASS;
```

를 반환할 수 있다.

```text
return XDP_PASS
       │
       ▼
R0 = XDP_PASS
       │
       ▼
     EXIT
```

R0는 eBPF Program의 Return Value에만 사용되는 것이 아니라 **Helper Function의 Return Value 역시 R0에 저장된다.**

따라서 Source에 명시적인 `return`이 없더라도 직전에 Helper를 호출했다면 R0 자체는 이미 초기화된 상태일 수 있다.

그래서 단순한

```text
R0 !read_ok
```

오류는 발생하지 않을 수도 있다.

하지만 이것이 해당 Program Type에 맞는 올바른 의미의 Return Value라는 뜻은 아니다.

따라서 Verifier가 받아줬다 = 프로그램이 의도한 대로 동작한다고 생각해서는 안 된다.

Verifier는 **안전성**을 확인하는 것이지 Application Logic의 정답까지 검증해주는 것은 아니다.

---

# 22. Invalid Instruction과 Unreachable Instruction

마지막으로 Verifier는 Bytecode 자체가 올바른 eBPF Instruction으로 이루어져 있는지도 검사한다.

Clang/LLVM처럼 정상적인 Compiler를 사용한다면 잘못된 Opcode가 생성되는 경우는 흔하지 않다.

다만 비교적 최신 Kernel에서 지원되는 Instruction을 이용해 만든 eBPF Bytecode를 매우 오래된 Kernel에 Load하는 경우처럼 Kernel Compatibility 문제로 Verification이 실패할 수 있다.

또한 실행될 방법이 없는 **Unreachable Instruction**도 Verifier의 검사 대상이다.

다만 C Source에 Unreachable Code가 존재하더라도 Compiler Optimization 과정에서 제거되면 Verifier에게 전달되는 Bytecode에는 이미 존재하지 않을 수 있다.

결국 여기에서도 다시 한 번 중요한 점은 같다.

```text
Verifier가 검사하는 대상

C Source Code ❌

       ↓

eBPF Bytecode ⭕
```

---

# 23. Chapter 6을 하나의 흐름으로 정리해보면

처음에는 Verifier가 단순히 "위험한 eBPF 프로그램을 거부하는 기능"이라고 생각했는데, 이번 Chapter를 공부하면서 실제로는 **eBPF Virtual Machine의 상태를 계속 추적하는 정적 분석기**에 가깝다는 것을 알게 되었다.

전체 흐름을 정리하면 다음과 같다.

```text
                    eBPF C Program
                           │
                           ▼
                     Clang / LLVM
                           │
                           ▼
                     eBPF Bytecode
                           │
                           ▼
               ┌─────────────────────┐
               │    eBPF Verifier    │
               └──────────┬──────────┘
                          │
         ┌────────────────┼─────────────────┐
         │                │                 │
         ▼                ▼                 ▼
    Register Type    Value Range       Control Flow
       Tracking        Tracking          Analysis
         │                │                 │
         └────────────────┼─────────────────┘
                          │
              ┌───────────┼───────────┐
              │           │           │
              ▼           ▼           ▼
           Pointer      Memory      Helper
           Safety       Bounds     Validation
              │           │           │
              ├───────────┼───────────┤
              │           │           │
              ▼           ▼           ▼
           Context       Loop       Return
            Access     Completion    Value
              │           │           │
              └───────────┼───────────┘
                          │
                          ▼
                모든 실행 경로에서
                 안전성을 증명 가능?
                       /      \
                     YES      NO
                      │        │
                      ▼        ▼
                 Kernel Load  Reject
```

Verifier를 이해할 때 특히 핵심이 되는 개념은 다음 네 가지라고 생각한다.

```text
Register Type
      +
Value Range
      +
Pointer State
      +
Control Flow
```

이 네 가지를 Verifier가 함께 추적하기 때문에 다음과 같은 코드들을 실행하기 전에 찾아낼 수 있다.

```text
NULL Pointer Dereference
Array Out-of-Bounds
Packet Boundary 초과
잘못된 Context Access
잘못된 Helper Argument
허용되지 않은 Helper 사용
초기화되지 않은 Register 사용
종료를 보장할 수 없는 프로그램
```

특히 `bpf_map_lookup_elem()`의 NULL Check 예제가 이 구조를 가장 잘 보여주는 것 같다.

처음에는

```text
PTR_TO_MAP_VALUE_OR_NULL
```

이었던 Pointer가

```c
if (p != NULL)
```

이라는 조건을 통과하면서

```text
PTR_TO_MAP_VALUE
```

로 좁혀지고, 그제야 Memory Dereference가 허용된다.

즉 C 코드에 작성한 조건문 하나가 단순히 Runtime의 실행 흐름만 바꾸는 것이 아니라 **Verifier에게 Pointer가 안전하다는 사실을 증명하는 역할까지 한다.**

---

# 마무리

eBPF는 Linux Kernel 내부에서 동작할 수 있기 때문에 매우 강력하다.

Network Packet을 Kernel 단계에서 처리하거나, Kernel Function을 추적하고, System Call이나 Process 동작을 관찰하는 등 일반적인 User Space Application으로는 하기 어려운 작업을 수행할 수 있다.

하지만 그만큼 잘못된 코드가 Kernel에 미칠 수 있는 영향도 크다.

eBPF가 이러한 강력한 기능을 제공하면서도 비교적 안전하게 사용할 수 있는 중요한 이유 중 하나가 바로 **Verifier**이다.

```text
강력한 Kernel 접근 능력
          │
          ▼
     eBPF Verifier
          │
          ▼
"안전하다는 것을 증명할 수 있는
 Program만 Kernel에서 실행"
```

이번 Chapter를 통해 Verifier가 단순히 프로그램을 한 번 검사하는 단계가 아니라, eBPF Bytecode의 가능한 Control Flow를 따라가면서 **Register Type, Value Range, Pointer State, Memory Bounds를 지속적으로 추적하는 과정​**이라는 것을 이해할 수 있었다.

앞으로 Verifier Log를 볼 때도 복잡한 Register 정보 자체를 외우려고 하기보다는

> **Verifier가 현재 이 Register에 대해 무엇을 알고 있고, 무엇을 아직 증명하지 못했기 때문에 이 코드를 거부한 것인가?**

라는 관점으로 보는 것이 중요할 것 같다.

---

# 출처
- https://ebpf.hamza-megahed.com/docs/chapter1/5-arch/

