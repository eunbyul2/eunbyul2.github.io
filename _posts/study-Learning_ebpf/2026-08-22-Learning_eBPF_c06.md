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

Verifier 자체는 이전 Chapter에서도 계속 등장했다. eBPF 프로그램을 Kernel에 Load하려고 하면 Verifier가 먼저 프로그램을 검사하고, 안전하지 않다고 판단하면 Load를 거부한다는 정도로 알고 있었다.

그런데 이번 Chapter를 읽으면서 Verifier가 단순히 "위험한 명령어가 있는지 확인하는 검사기" 정도는 아니라는 것을 알게 됐다.

Verifier는 컴파일된 eBPF Bytecode를 따라가면서 **가능한 실행 경로를 분석하고, 각 시점의 Register가 어떤 종류의 값과 어느 범위의 값을 가질 수 있는지 추적한다.** 이 정보를 이용해 Pointer 접근, 배열 범위, Helper 호출, Loop 같은 동작이 안전한지를 판단한다.

eBPF 프로그램은 User Space Application과 달리 Kernel 안에서 실행된다. 잘못된 Pointer Access나 끝나지 않는 Loop가 그대로 실행된다면 Kernel 전체의 안정성에 영향을 줄 수 있다. 그래서 eBPF에서는 프로그램을 실행하기 전에 Verifier가 안전성을 확인하는 과정이 중요하다.

이번 장은 결국 다음 질문에 대한 내용이었다.

> **Kernel은 eBPF 프로그램을 실제로 실행하지 않고 어떻게 이 프로그램이 안전한지 판단할까?**

---

# 1. eBPF 프로그램은 컴파일됐다고 바로 실행되는 것이 아니다

먼저 eBPF 프로그램이 실행되기까지의 흐름을 다시 보면 다음과 같다.

```text
eBPF C Source
      │
      │ Clang / LLVM
      ▼
eBPF Object File (.o)
      │
      │ eBPF Bytecode
      ▼
bpf() system call
BPF_PROG_LOAD
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
Machine Code
 │
 ▼
실행
```

![eBPF 프로그램의 전체 실행 구조](/img/study-Learning_eBPF/eBPF%20프로그램의%20전체%20실행%20구조.png)

여기서 처음 정리해둘 부분은 **Compile 성공과 Verifier 통과는 별개의 문제**라는 점이다.

예를 들어 `hello-verifier.bpf.c`를 Clang으로 정상적으로 컴파일하면 `hello-verifier.bpf.o` 같은 Object File이 만들어진다.

```text
hello-verifier.bpf.c
        │
        │ clang
        ▼
hello-verifier.bpf.o
```

`.o` 파일 안에는 Kernel에서 실행할 eBPF Bytecode가 들어 있다.

하지만 `.o` 파일이 만들어졌다는 것은 Compiler가 해당 Source를 eBPF Bytecode로 변환하는 데 성공했다는 뜻이지, **그 Bytecode가 Kernel에서 안전하게 실행될 수 있다는 뜻은 아니다.**

이 안전성을 판단하는 것이 그다음 단계인 Verifier이다.

실제로 Chapter 6에 나오는 예제 중에는 C 코드 자체는 문제없이 컴파일되지만 Kernel에 Load하는 순간 Verifier가 거부하는 경우가 여러 번 등장한다.

이 차이는 뒤에 나오는 `bpf_map_lookup_elem()` 예제에서 특히 잘 보인다.

---

## Verifier는 C 코드를 직접 검사하지 않는다

또 하나 중요했던 부분은 **Verifier가 Source Code가 아니라 eBPF Bytecode를 검사한다는 것**이다.

```text
C Source
   │
   │ Compiler
   ▼
eBPF Bytecode
   │
   ▼
Verifier
```

그래서 Source를 수정했다고 해서 반드시 예상한 형태의 Bytecode가 만들어지는 것은 아니다. 중간에 Compiler Optimization이 있기 때문이다.

예를 들어 Source에 실행될 수 없는 코드가 있더라도 Compiler가 이를 최적화 단계에서 제거하면 Verifier 입장에서는 해당 Instruction 자체가 존재하지 않는다.

책에서도 Verifier는 unreachable instruction을 거부할 수 있지만, Compiler가 먼저 해당 코드를 없애버리면 Verifier는 그 코드를 볼 수 없다고 설명한다.

따라서 Verifier Error를 볼 때는 단순히 C 코드만 보는 것이 아니라

```text
C Source
   ↓
Compiler
   ↓
eBPF Bytecode
   ↓
Verifier
```

의 흐름을 같이 생각해야 한다.

---

# 2. Verifier는 무엇을 추적할까?

Verifier는 프로그램을 실제 값으로 실행해보는 대신 Bytecode의 Instruction을 따라가면서 **프로그램의 상태를 추론**한다.

이 과정에서 Chapter 3에서 봤던 eBPF Virtual Machine의 Register가 다시 중요해진다.

eBPF VM에는 R0부터 R10까지 Register가 있다.

| Register | 주요 역할 |
|---|---|
| `R0` | Helper Function / eBPF Program의 Return Value |
| `R1 ~ R5` | Function Argument 전달 |
| `R6 ~ R9` | Callee-saved Register |
| `R10` | Stack Frame Pointer |

Verifier는 Instruction을 하나씩 분석하면서 각 Register의 상태를 `bpf_reg_state`라는 구조를 통해 추적한다.

여기서 단순히 "R2에 숫자가 들어 있다" 정도만 보는 것이 아니다.

크게 두 가지를 같이 추적한다.

```text
Register State
     │
     ├── 어떤 종류의 값인가?  → Type
     │
     └── 어느 범위의 값인가?  → Value Range
```

이 두 정보가 Memory Safety를 판단하는 기반이 된다.

---

## 2.1 Register Type

책에서 소개하는 대표적인 Register Type에는 다음과 같은 것들이 있다.

- `NOT_INIT` : 아직 초기화되지 않은 Register
- `SCALAR_VALUE` : Pointer가 아닌 일반 값
- `PTR_TO_CTX` : eBPF Program에 전달된 Context Pointer
- `PTR_TO_PACKET` : Network Packet을 가리키는 Pointer
- `PTR_TO_MAP_KEY` : BPF Map Key를 가리키는 Pointer
- `PTR_TO_MAP_VALUE` : BPF Map Value를 가리키는 Pointer

Pointer를 모두 똑같은 주소값으로 취급하지 않고 **어디에서 온 Pointer인지까지 구분해서 추적한다는 것**이 핵심이다.

예를 들어 Verifier가 다음처럼 알고 있을 수 있다.

```text
R1 → PTR_TO_CTX
R2 → SCALAR_VALUE
R3 → PTR_TO_PACKET
R6 → PTR_TO_MAP_VALUE
```

그러면 단순히 "R1에 어떤 주소가 들어 있다"가 아니라, R1은 Context를 가리키는 Pointer이므로 어떤 Context Field에 접근할 수 있는지 판단할 수 있다.

마찬가지로 Packet Pointer라면 Packet Boundary를 벗어나는지 검사할 수 있고, Map Value Pointer라면 해당 Map Value의 크기 안에서만 접근하도록 제한할 수 있다.

---

## 2.2 Register가 가질 수 있는 값의 범위

Verifier는 Type뿐 아니라 Register가 가질 수 있는 **최소값과 최대값의 범위**도 추적한다.

Chapter의 Verifier Log에는 다음과 같은 정보가 나온다.

```text
R2_w=inv(id=1,umax_value=4294967295,
         var_off=(0x0; 0xffffffff))

R3_w=inv(id=0,umin_value=1,umax_value=4294967296,
         var_off=(0x0; 0x1ffffffff))
```

앞의 Instruction을 보면 R2의 값을 R3에 복사한 뒤 1을 더한다.

```text
4: (bf) r3 = r2
5: (07) r3 += 1
```

따라서 Verifier는 R3가 가질 수 있는 값의 범위도 그에 맞게 변경해서 추적한다.

이런 Range Tracking이 필요한 이유는 뒤에서 나오는 배열 접근 예제를 보면 바로 이해할 수 있다.

배열 크기가 12라면 정상적인 Index는 `0~11`이다.

Verifier가 Index로 쓰이는 Register가 `0~11` 범위라고 알고 있다면 접근을 허용할 수 있지만, `12`까지 가능하다고 알고 있다면 `message[12]`가 실행될 가능성이 생긴다.

즉 실제로 `12`가 들어왔는지를 실행해보는 것이 아니라 **Register가 가질 수 있는 최대 범위를 이용해 위험 가능성을 미리 판단하는 것**이다.

---

# 3. Branch가 나오면 가능한 경로를 모두 따라간다

조건문이 나오면 프로그램의 실행 경로가 둘 이상으로 나뉜다.

Verifier는 이때 한쪽 경로만 검사하지 않는다.

Branch를 만났을 때 현재 Register State를 저장해두고 한 경로를 따라간 다음, 다시 저장된 State로 돌아와 다른 경로도 분석한다.

```text
              Branch
                │
         현재 State 저장
                │
        ┌───────┴───────┐
        ▼               ▼
     Path A           Path B
        │               │
     검사              검사
        └───────┬───────┘
                ▼
              다음
```

이 과정이 중요한 이유는 eBPF Program이 **어떤 경로로 실행되더라도 안전해야 하기 때문**이다.

특정 입력에서는 문제가 없더라도 다른 Branch에서 잘못된 Pointer Access가 가능하다면 Verifier는 프로그램을 거부한다.

---

## State Pruning

물론 모든 경로를 끝까지 계속 다시 분석하면 Program이 조금만 복잡해져도 검증 비용이 크게 늘어난다.

그래서 Verifier는 **State Pruning**이라는 최적화를 사용한다.

Verifier가 이전에 어떤 Instruction에 특정 Register State로 도달했고, 그 이후 경로가 안전하다는 것을 이미 확인했다고 하자.

나중에 다른 경로를 통해 같은 Instruction에 도착했는데 Register State가 이전 상태와 동일하거나 이전에 검증한 상태의 범위 안에 포함된다면, 이후를 다시 검사할 필요가 없다.

```text
Path A
  │
  ▼
Instruction X
State A
  │
  ▼
이후 경로 검증 완료


Path B
  │
  ▼
Instruction X
State A
  │
  ▼
이미 확인한 상태
  │
  ▼
Prune
```

책에서는 과거에는 Jump Instruction 주변에 Pruning State를 자주 저장했지만, 실제 분석 결과 대부분의 State가 다시 Match되지 않았고, 이후에는 일정 Instruction 간격을 두고 State를 저장하는 쪽이 더 효율적이었다고 설명한다.

즉 Verifier는 단순히 모든 경로를 무작정 반복해서 검사하는 것이 아니라, **이미 검증한 상태를 재사용하면서 분석 비용을 줄인다.**

---

# 4. Control Flow를 직접 보면 조금 더 이해하기 쉽다

`bpftool`을 사용하면 Load된 eBPF Program의 Control Flow Graph를 DOT 형식으로 출력할 수 있다.

```bash
bpftool prog dump xlated name kprobe_exec visual > out.dot
```

Graphviz의 `dot` 명령으로 PNG로 변환할 수 있다.

```bash
dot -Tpng out.dot > out.png
```

Chapter 6에서 나온 Control Flow Graph의 일부는 다음과 같다.

![eBPF Control Flow Graph](/img/study-Learning_eBPF/eBPF%20Control%20Flow%20Graph.png)

그림을 보면 Instruction이 단순히 위에서 아래로만 이어지는 것이 아니라 조건에 따라 여러 Basic Block으로 나뉜다.

예를 들어 다음 Instruction에서는 R0의 값에 따라 경로가 나뉜다.

```text
28: (15) if r0 == 0x0 goto pc+1
```

한쪽 경로에서는 29번 Instruction을 거친 뒤 30번으로 이동하고, 다른 쪽은 바로 다음 Block으로 이어진다.

아래에서도 다시 Branch가 있다.

```text
37: (15) if r8 == 0x0 goto pc+4
```

Verifier가 "모든 실행 경로를 검사한다"는 말이 이 그림을 보면 좀 더 구체적으로 보인다.

각 Branch를 따라가면서 Register State를 별도로 추적하고, 다시 합쳐지는 지점에서는 이미 검증했던 상태인지 비교해 State Pruning을 수행할 수 있다.

---

# 5. Verifier Log를 읽어보자

프로그램이 Verification에 실패했을 때 가장 중요한 자료가 **Verifier Log**이다.

`bpftool prog load`를 사용하는 경우 Verification Error가 `stderr`에 출력된다.

libbpf를 직접 사용하는 프로그램에서는 `libbpf_set_print()`를 이용해 libbpf의 Log를 처리할 수 있다.

Chapter 예제의 Verifier Log 마지막에는 다음과 같은 Summary가 나온다.

```text
processed 61 insns (limit 1000000)
max_states_per_insn 0
total_states 4
peak_states 4
mark_read 3
```

`processed 61 insns`는 Verifier가 분석 과정에서 처리한 Instruction 수이다.

Branch를 통해 같은 Instruction을 서로 다른 State로 여러 번 방문할 수도 있기 때문에 Source Code의 줄 수나 Bytecode Instruction의 단순 개수와 반드시 같지는 않다.

---

## `-g` 옵션이 있으면 Source와 Bytecode를 같이 볼 수 있다

Chapter 예제는 Compile할 때 `-g` 옵션을 사용했기 때문에 Verifier Log에 C Source Line도 같이 나타난다.

```text
; c++;
4: (bf) r3 = r2
5: (07) r3 += 1
6: (63) *(u32 *)(r1 +0) = r3
```

`c++`라는 Source Code가 실제로 어떤 eBPF Instruction으로 변환됐는지를 바로 아래에서 볼 수 있다.

Verifier Error를 볼 때 이 Debug Information이 있으면 **어떤 Source Line이 어떤 Register 조작으로 바뀌었는지**를 추적하기 훨씬 쉬워진다.

---

## 왜 첫 Instruction에서 R1을 R6에 복사할까?

Verifier Log의 시작 부분에는 다음 Instruction이 나온다.

```text
0: (bf) r6 = r1
```

eBPF Program이 호출될 때 R1에는 Program의 Context가 전달된다.

그런데 BPF Helper Function을 호출할 때 R1~R5는 Helper Argument로 사용된다.

```text
R1 → Argument 1
R2 → Argument 2
R3 → Argument 3
R4 → Argument 4
R5 → Argument 5
```

반면 R6~R9는 Helper Call 이후에도 값이 보존되는 Callee-saved Register이다.

따라서 Context를 Helper 호출 이후에도 계속 사용해야 한다면 Compiler가 R1의 Context Pointer를 R6에 미리 복사해둘 수 있다.

```text
Program 시작
R1 = Context
     │
     └── copy
          ↓
     R6 = Context

Helper Call
R1~R5 사용

Helper Call 이후
R6 = Context 유지
```

Verifier Log를 읽을 때 R6에 `ctx`가 잡혀 있는 이유가 이것이다.

---

# 6. Helper Function도 Verifier의 검증 대상이다

eBPF Program은 Kernel 안에서 실행되지만 Kernel Function을 아무 것이나 직접 호출할 수 있는 것은 아니다.

대신 Kernel에서 제공하는 BPF Helper Function을 사용한다.

예를 들면 다음과 같다.

```c
bpf_map_lookup_elem()
bpf_get_current_pid_tgid()
bpf_probe_read_kernel()
bpf_printk()
```

그런데 모든 Helper를 모든 BPF Program Type에서 사용할 수 있는 것은 아니다.

책에서는 기존 Program의 `SEC()`를 `kprobe`에서 `xdp`로 변경한 뒤 `bpf_get_current_pid_tgid()`를 호출해본다.

그러면 Verifier에서 다음 오류가 발생한다.

```text
16: (85) call bpf_get_current_pid_tgid#14
unknown func bpf_get_current_pid_tgid#14
```

처음 보면 함수 자체가 존재하지 않는다는 뜻처럼 보이지만, 여기서는 **현재 BPF Program Type에서 사용할 수 없는 Helper**라는 의미이다.

`bpf_get_current_pid_tgid()`는 현재 실행 중인 Process의 PID/TGID를 가져오는 Helper인데, Network Interface에 Packet이 들어오는 시점에 실행되는 XDP Program에는 이런 User Space Process Context가 맞지 않기 때문이다.

---

## 6.1 Helper Argument의 Type도 확인한다

이 부분은 Register Type Tracking이 실제로 어디에 사용되는지 잘 보여주는 예제였다.

`bpf_map_lookup_elem()`의 Helper Prototype은 대략 다음과 같이 정의된다.

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

핵심만 보면 다음과 같다.

```text
Argument 1 → Map Pointer
Argument 2 → Map Key Pointer

Return     → Map Value Pointer 또는 NULL
```

정상적인 호출은 다음과 같다.

```c
p = bpf_map_lookup_elem(&my_config, &uid);
```

![eBPF Program ↔ Map 관계](/img/study-Learning_eBPF/eBPF%20Program와%20Map%20관계.png)

`my_config`는 BPF Map이고 `uid`는 조회할 Key이다.

책에서는 이 코드를 일부러 다음처럼 바꿔본다.

```c
p = bpf_map_lookup_elem(&data, &uid);
```

`my_config` 대신 Local Variable인 `data`의 주소를 첫 번째 Argument로 전달한 것이다.

흥미로운 점은 **이 코드는 컴파일된다.** 따라서 `.o` 파일까지 만들어진다.

하지만 Kernel에 Load할 때는 Verifier가 다음 오류를 출력한다.

```text
27: (85) call bpf_map_lookup_elem#1
R1 type=fp expected=map_ptr
```

Helper의 첫 번째 Argument는 R1에 들어간다.

Verifier가 추적한 R1은 `fp`, 즉 Stack Frame 쪽 Pointer였지만 `bpf_map_lookup_elem()`이 요구하는 것은 `map_ptr`이다.

```text
실제 R1      → fp
                Stack / Local Variable Pointer

필요한 R1    → map_ptr
                BPF Map Pointer

                ↓

             Type 불일치
                ↓
             Reject
```

앞에서 Register Type을 계속 추적한다고 했던 이유가 여기에서 드러난다.

Compiler가 Bytecode를 만들어낼 수 있느냐와, Helper의 계약에 맞는 안전한 Pointer를 전달했느냐는 다른 문제이다.

---

## 6.2 GPL License도 확인한다

일부 Helper는 GPL-Compatible Program에서만 사용할 수 있다.

Chapter의 예제 Program에는 다음과 같은 License Section이 있다.

```c
char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

`bpf_probe_read_kernel()`은 GPL 제한이 있는 Helper이기 때문에 License 선언을 제거하면 다음과 같은 오류가 발생한다.

```text
cannot call GPL-restricted function from non-GPL compatible program
```

즉 Verifier는 Helper 이름만 확인하는 것이 아니라 **Program Type, Argument Type, Return Type, License 조건까지 Helper의 사용 규칙을 확인한다.**

---

# 7. Memory Access는 어떻게 검사할까?

Verifier의 핵심 역할 중 하나는 eBPF Program이 허용되지 않은 Memory를 접근하지 못하게 하는 것이다.

Chapter에서는 XDP Packet과 배열 접근 예제를 이용해 이를 설명한다.

---

## 7.1 XDP의 `data`와 `data_end`

XDP Program에는 `struct xdp_md *ctx`가 Context로 전달된다.

```c
SEC("xdp")
int xdp_hello(struct xdp_md *ctx)
{
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    ...
}
```

`data`는 Packet이 시작하는 위치이고 `data_end`는 Packet의 끝을 나타낸다.

![XDP Packet data와 data_end](/img/study-Learning_eBPF/XDP%20Packet%20data-data_end.png)

개념적으로 보면 다음과 같다.

```text
ctx->data
   │
   ▼
┌──────────┬──────────┬───────────┬──────────┐
│ Ethernet │ IP       │ TCP / UDP │ Payload  │
└──────────┴──────────┴───────────┴──────────┘
                                             ▲
                                             │
                                      ctx->data_end
```

Packet Header를 읽으려면 읽으려는 위치가 `data_end`를 넘지 않는다는 것을 먼저 확인해야 한다.

Verifier는 Packet Pointer와 `data_end`를 이용해 이런 Bounds Check가 있었는지 확인한다.

책에서는 `data_end`에 다음 연산을 추가해본다.

```c
data_end++;
```

그러면 Verifier가 다음 오류를 출력한다.

```text
R3 pointer arithmetic on pkt_end prohibited
```

`data_end`는 Packet의 실제 끝을 나타내는 값인데, Program이 이를 임의로 수정할 수 있다면 Packet Boundary 검사 자체를 속일 수 있기 때문이다.

그래서 Verifier는 Packet End Pointer에 대한 이런 Pointer Arithmetic을 허용하지 않는다.

---

## 7.2 배열 범위도 Register Range로 확인한다

이번에는 다음과 같은 코드가 있다.

```c
if (c < sizeof(message)) {
    char a = message[c];
    bpf_printk("%c", a);
}
```

`message`가 12Byte 배열이라면 유효한 Index는 `0~11`이다.

`c < sizeof(message)`라는 조건 때문에 `c`는 12가 될 수 없으므로 정상적으로 Verification을 통과한다.

그런데 조건을 다음처럼 바꾸면 문제가 생긴다.

```c
if (c <= sizeof(message)) {
    char a = message[c];
    bpf_printk("%c", a);
}
```

`sizeof(message)`가 12라면 이제 `c == 12`도 조건을 통과한다.

그러면 `message[12]`가 실행될 가능성이 생기는데, 마지막 유효 Index는 11이므로 한 칸 범위를 벗어난다.

Verifier는 다음과 같은 오류를 낸다.

```text
invalid access to map value, value_size=16 off=16 size=1
R2 max value is outside of the allowed memory range
```

이 부분이 앞에서 본 Register Range Tracking과 연결된다.

Verifier Log를 거꾸로 따라가면 R1이 최대 12까지 될 수 있고, 그 값이 `message[c]`의 Index 계산에 사용된다는 것을 볼 수 있다.

즉 Verifier는 실제로 `c=12`를 넣어 실행해본 것이 아니라

```text
c의 최대값이 12일 수 있음
        ↓
message[12] 가능
        ↓
배열의 유효 범위는 0~11
        ↓
Out-of-Bounds 가능
        ↓
Reject
```

라고 정적으로 판단한 것이다.

여기서 Error가 `array`가 아니라 `map value`라고 나오는 이유도 Chapter 3에서 배운 내용과 연결된다.

`message`는 Global Variable이고, eBPF의 Global Variable은 내부적으로 Map을 통해 구현된다. 따라서 Verifier 입장에서는 이 접근이 Map Value Memory에 대한 접근으로 보인다.

---

# 8. NULL Pointer 검사는 Verifier의 동작을 가장 잘 보여준다

이번 Chapter에서 가장 이해하기 좋았던 예제는 `bpf_map_lookup_elem()`의 반환값을 검사하는 부분이었다.

```c
p = bpf_map_lookup_elem(&my_config, &uid);
```

Map에 해당 Key가 있으면 Value Pointer가 반환되지만, Key가 없으면 `NULL`이 반환될 수 있다.

그래서 Helper의 Return Type도 다음과 같이 정의되어 있다.

```text
RET_PTR_TO_MAP_VALUE_OR_NULL
```

Helper Return Value는 R0에 들어가고, Chapter 예제에서는 이 값이 R7로 복사된다.

```text
25: (18) r1 = ...
27: (85) call bpf_map_lookup_elem#1
28: (bf) r7 = r0
```

이 시점에서 Verifier가 알고 있는 R7의 상태는

```text
map_value_or_null
```

이다.

즉 Map Value Pointer일 수도 있지만 `NULL`일 수도 있다.

그런데 바로 다음과 같이 Pointer를 Dereference하면

```c
char a = p->message[0];
```

Verifier는 다음과 같이 거부한다.

```text
29: (71) r3 = *(u8 *)(r7 +0)
R7 invalid mem access 'map_value_or_null'
```

이 Log를 연결해서 보면 꽤 직관적이다.

```text
bpf_map_lookup_elem()
        │
        ▼
R0 = MAP_VALUE_OR_NULL
        │
        ▼
R7 = R0
        │
        ▼
R7 = MAP_VALUE_OR_NULL
        │
        ▼
*(R7 + 0)
        │
        ▼
NULL일 가능성이 남아 있음
        │
        ▼
Reject
```

반대로 다음처럼 명시적인 NULL Check를 넣으면 Verification을 통과한다.

```c
if (p != 0) {
    char a = p->message[0];
    bpf_printk("%c", a);
}
```

이 조건은 Runtime에서 Crash를 막는 역할도 하지만 **Verifier가 알고 있는 Pointer State를 바꾸는 역할**도 한다.

```text
p = MAP_VALUE_OR_NULL
        │
        ▼
    p != NULL ?
      /     \
    YES      NO
     │        │
     ▼        ▼
MAP_VALUE    NULL
     │
     ▼
Dereference 허용
```

TRUE Branch에 들어왔다는 사실 자체가 `p != NULL`임을 의미하므로 Verifier는 그 경로에서 Pointer를 안전한 Map Value Pointer로 취급할 수 있다.

앞에서 살펴본 **Control Flow Analysis와 Register Type Tracking이 실제로 어떻게 연결되는지**를 가장 잘 보여주는 예제라고 느꼈다.

---

# 9. `bpf_probe_read_kernel()`은 왜 `unsafe_ptr`을 받을 수 있을까?

일반적으로 Verifier는 안전성이 확인되지 않은 Pointer를 직접 Dereference하는 것을 허용하지 않는다.

그런데 Kernel Memory를 읽어야 하는 경우에는 `bpf_probe_read_kernel()` 같은 Helper를 사용할 수 있다.

```c
long bpf_probe_read_kernel(
    void *dst,
    u32 size,
    const void *unsafe_ptr
);
```

세 번째 Argument 이름 자체가 `unsafe_ptr`이다.

이 Helper는 잠재적으로 안전하지 않은 Kernel Pointer를 받아 안전하게 읽기 위한 함수이므로, 직접 Pointer를 Dereference하는 것과는 다르게 처리된다.

즉 eBPF Program이 위험한 Kernel Memory를 임의로 직접 읽게 하는 것이 아니라 **Kernel이 제공하는 Helper를 통해 제한된 방식으로 접근하도록 만든 것**이다.

---

# 10. Context도 아무 Field나 접근할 수 있는 것은 아니다

모든 eBPF Program은 실행될 때 Context를 전달받지만, 해당 Context의 모든 Field에 자유롭게 접근할 수 있는 것은 아니다.

Program Type과 Attachment Type에 따라 접근할 수 있는 Context가 다르다.

예를 들어 Tracepoint Program은 Tracepoint Data에 대한 Pointer를 받지만, Tracepoint Context의 모든 공통 Field를 eBPF Program에서 접근할 수 있는 것은 아니다.

허용되지 않은 Field에 접근하면 다음과 같은 오류가 발생한다.

```text
invalid bpf_context access
```

즉 Verifier는

```text
R1 = Context Pointer
```

라는 사실에서 끝나는 것이 아니라, **현재 Program Type에서 해당 Context의 그 Offset에 접근해도 되는지**까지 검사한다.

---

# 11. 프로그램이 끝날 수 있는지도 검증한다

Memory Access가 아무리 안전해도 eBPF Program이 끝없이 실행된다면 Kernel Resource를 계속 점유할 수 있다.

그래서 Verifier는 Program이 제한된 복잡도 안에서 끝까지 도달할 수 있는지도 확인한다.

Chapter에서 설명하는 Verifier의 처리 한도는 1,000,000 Instructions이다.

이 값은 단순한 Source Line 수가 아니라 Verifier가 Branch를 따라가면서 처리하게 되는 Instruction 수와 관련된 Complexity Limit이다.

---

## 11.1 과거에는 Loop를 쓰기 어려웠던 이유

일반적인 Loop는 Bytecode에서 이전 Instruction으로 돌아가는 Backward Jump가 필요하다.

```text
       ┌───────────────┐
       │               │
       ▼               │
Loop Condition         │
       │               │
       ▼               │
   Loop Body           │
       │               │
       └── Backward ───┘
```

Kernel 5.3 이전에는 이런 Backward Jump를 Verifier가 허용하지 않았다.

그래서 당시 eBPF Program에서는 `#pragma unroll`을 이용해 Loop를 Compiler 단계에서 펼치는 방법을 많이 사용했다.

```c
#pragma unroll
for (int i = 0; i < 3; i++) {
    A();
}
```

개념적으로는

```c
A();
A();
A();
```

처럼 반복되는 Instruction을 직접 만들어버리는 방식이다.

Loop는 없어지지만 반복 횟수만큼 Bytecode 크기가 증가한다.

---

## 11.2 Kernel 5.3 이후의 Bounded Loop

Kernel 5.3 이후에는 Verifier가 Backward Branch도 분석할 수 있게 되면서 종료 범위를 확인할 수 있는 Loop를 사용할 수 있게 됐다.

```c
for (int i = 0; i < 10; i++) {
    bpf_printk("Looping %d", i);
}
```

이 경우 반복 횟수가 최대 10회라는 것이 명확하므로 Verifier가 각 실행 경로를 분석할 수 있다.

하지만 반복 범위가 크고 Loop 내부 Branch가 많다면 가능한 State 수가 크게 증가한다.

따라서 "Loop를 사용할 수 있다"와 "Verifier가 쉽게 검증할 수 있다"는 같은 의미는 아니다.

---

## 11.3 `bpf_loop()`

Kernel 5.17에서는 `bpf_loop()` Helper도 추가됐다.

이 Helper는 최대 반복 횟수와 반복마다 호출할 Callback을 전달한다.

```text
bpf_loop(
    maximum_iterations,
    callback,
    ...
)
```

일반 Loop에서는 반복 과정의 많은 State를 Verifier가 따라가야 하지만 `bpf_loop()`를 사용하면 반복되는 Callback 자체를 검증하는 방식으로 처리할 수 있어 Verifier 입장에서도 훨씬 효율적이다.

Map Element를 순회할 때 사용할 수 있는 `bpf_for_each_map_elem()` Helper도 있다.

---

# 12. Return Value도 검사한다

eBPF Program의 Return Value는 R0에 저장된다.

Program이 종료될 때 R0가 초기화되지 않았다면 Verifier는 다음과 같은 오류를 발생시킨다.

```text
R0 !read_ok
```

예를 들어 XDP Program에서 Return을 모두 제거하면 R0가 초기화되지 않은 상태로 Program이 끝날 수 있다.

```c
SEC("xdp")
int xdp_hello(struct xdp_md *ctx)
{
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    // return XDP_PASS;
}
```

반면 다음과 같이 Return 값을 넣으면 R0에 해당 값이 저장된다.

```c
return XDP_PASS;
```

한 가지 재미있었던 부분은 **Helper Function의 Return Value 역시 R0를 사용한다는 점**이다.

그래서 명시적인 `return`을 제거했더라도 직전에 Helper를 호출해서 R0가 초기화되어 있다면 단순한 `R0 !read_ok` Error가 발생하지 않을 수도 있다.

물론 이것이 해당 BPF Program Type에서 의미적으로 올바른 Return Code라는 뜻은 아니다.

Verifier는 Program의 **안전성**을 검사하는 것이지, Program Logic이 개발자의 의도와 맞는지까지 확인하는 것은 아니다.

---

# 13. Invalid Instruction과 Unreachable Instruction

Verifier는 Bytecode가 올바른 eBPF Instruction으로 구성되어 있는지도 확인한다.

일반적으로 Clang/LLVM이 잘못된 eBPF Opcode를 생성하는 경우는 흔하지 않기 때문에 직접 Bytecode를 작성하지 않는다면 자주 만날 문제는 아니다.

다만 새로운 Kernel에서 추가된 Instruction을 사용하는 Bytecode를 오래된 Kernel에 Load한다면 해당 Instruction을 지원하지 않아 Verification에 실패할 수 있다.

실행될 수 없는 Unreachable Instruction도 Verifier가 거부할 수 있다.

다만 앞에서 살펴봤듯이 Compiler Optimization이 먼저 해당 코드를 제거하면 Verifier가 아예 보지 못할 수도 있다.

이 부분에서도 다시 **Verifier가 검사하는 것은 C Source가 아니라 최종 eBPF Bytecode**라는 점이 연결된다.

---

# 정리

이번 Chapter를 읽기 전에는 Verifier를 단순히 "Kernel에 Load하기 전에 안전한지 검사하는 단계" 정도로 생각했다.

하지만 내부 동작을 따라가보면 Verifier는 eBPF Bytecode의 가능한 실행 경로를 분석하면서 **각 Register의 Type과 Value Range를 계속 추적하는 정적 분석기**에 가깝다.

이번 장의 내용은 아래 흐름으로 연결할 수 있었다.

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
Pointer / Memory Access 검증
      │
      ├── Packet Boundary
      ├── Array Bounds
      ├── NULL Pointer
      └── Context Access
      │
      ▼
Helper 사용 규칙 검증
      │
      ├── Program Type
      ├── Argument Type
      └── License
      │
      ▼
Loop / Return / Instruction 검증
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

특히 이번 Chapter에서 가장 기억에 남는 부분은 `bpf_map_lookup_elem()`의 NULL Pointer 예제였다.

Helper가 반환한 값이 처음에는 `PTR_TO_MAP_VALUE_OR_NULL`이고, Verifier는 그 상태에서는 Pointer Dereference를 허용하지 않는다.

하지만

```c
if (p != NULL)
```

이라는 조건을 통과한 경로에서는 `p`가 NULL이 아니라는 사실이 확인되므로 Verifier가 안전한 Map Value Pointer로 취급할 수 있다.

이 예제를 통해 Verifier가 단순히 Bytecode를 위에서 아래로 읽는 것이 아니라 **Control Flow에 따라 Register에 대해 알고 있는 정보 자체를 계속 갱신하고 있다는 것**을 이해할 수 있었다.

앞으로 Verifier Error를 볼 때는 Error Message만 보고 끝내기보다 다음 순서로 확인하면 조금 더 원인을 찾기 쉬울 것 같다.

```text
어느 Instruction에서 실패했는가?
        ↓
어떤 Register가 문제인가?
        ↓
Verifier는 그 Register를 어떤 Type으로 보고 있는가?
        ↓
Value Range는 어떻게 되는가?
        ↓
그 Register는 어디에서 만들어졌는가?
        ↓
어떤 Branch를 지나 현재 State가 되었는가?
```

결국 eBPF가 Kernel 내부에서 강력한 기능을 수행하면서도 안전성을 유지할 수 있는 핵심 이유 중 하나가 이 Verifier에 있다.

---

### 출처
- https://ebpf.hamza-megahed.com/docs/chapter1/5-arch/
