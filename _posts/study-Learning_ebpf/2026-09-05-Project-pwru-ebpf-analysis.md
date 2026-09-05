---
layout: post
title: "[eBPF Open Source] Cilium pwru로 보는 kprobe 기반 Packet Tracing"
date: 2026-09-05 19:30:00 +0900
categories: [eBPF, Cilium, Observability]
tags: [eBPF, pwru, Cilium, kprobe, kprobe-multi, BTF, CO-RE, BPF Map, sk_buff, Network Debugging]
published: true
---

# pwru의 eBPF 활용 분석

Learning eBPF Chapter 1~7에서는 eBPF Program이 Kernel Event에 어떻게 연결되는지, BPF Map으로 State를 어떻게 공유하는지, CO-RE/BTF로 Kernel 버전 차이를 어떻게 흡수하는지, Verifier가 무엇을 검증하는지, kprobe/fentry/fexit 같은 Attachment Type이 어떻게 다른지를 각각 따로 학습했다.

이번 스터디 주제는 "eBPF 오픈소스 분석"이었고, 이번에는 Cilium 산하의 [`pwru`](https://github.com/cilium/pwru)를 분석 대상으로 선택했다. Pixie의 Socket Tracer를 분석했을 때는 System Call Entry/Return을 연결하는 State Machine 관점이 중심이었다면, 이번 `pwru`는 **"수천 개의 Kernel Function에 kprobe를 걸어 하나의 Packet(`sk_buff`)이 Kernel 내부를 어떻게 통과하는지 추적한다"**는 훨씬 더 단순하고 날 것의 kprobe 활용 사례라서, Chapter 6(Verifier)와 Chapter 7(Attachment Type)에서 배운 내용을 실제 코드로 확인하기에 적합했다.

---

## 1. pwru란?

https://github.com/cilium/pwru

> pwru (packet, where are you?)

`pwru`는 Linux Kernel 내부에서 Network Packet이 어떤 경로로 흘러가는지 추적하는 eBPF 기반 CLI Tool이다. README의 설명을 요약하면 다음과 같다.

- Kernel 안에서 Packet은 `struct sk_buff`(줄여서 skb)라는 하나의 구조체 포인터로 표현되며, 이 포인터가 `ip_rcv`, `tcp_v4_rcv`, `nf_hook` 같은 여러 Kernel Function을 순서대로 통과한다.
- `pwru`는 **`sk_buff`를 인자로 받는 Kernel Function을 최대한 많이 찾아내서 전부 kprobe를 걸어두고**, 어떤 Packet이 그 함수들을 지날 때마다 Event를 발생시켜 User Space로 전달한다.
- iptables Rule에 의해 Packet이 Drop되는 지점, NAT 이후 주소가 바뀌는 지점 등을 실시간으로 확인할 수 있어 Network Connectivity 디버깅에 사용된다.

즉 하나의 Kernel Event Type에 국한되지 않고 **"skb를 인자로 받는다"는 조건 하나로 수천 개의 Attach Point를 동적으로 찾아내는 것**이 이 프로젝트의 핵심 아이디어다.

---

## 2. 왜 pwru를 분석했는가?

Chapter 1~7에서 본 예제들은 대부분 미리 정해진 하나(또는 소수)의 Kernel Function/Tracepoint에 Program을 Attach하는 형태였다. 예를 들어 `execve` Syscall 하나, `kfree_skb` Tracepoint 하나에 Program을 붙이는 식이다.

그래서 다음이 궁금했다.

- Attach 대상이 되는 Kernel Function을 **코드에 하드코딩하지 않고 런타임에 동적으로 찾아내려면** 어떻게 해야 하는가?
- kprobe Handler는 Kernel Function의 Argument를 어떻게 읽는가? Argument 순서가 함수마다 다르면 eBPF Program은 어떻게 대응하는가?
- 수천 개의 kprobe를 거는 것은 성능/구현상 어떤 제약을 만드는가?
- BTF는 단순히 CO-RE Struct Access뿐 아니라 "이 함수의 Signature가 무엇인지" 같은 정보도 제공하는가?
- Verifier 통과가 필요한 eBPF Program이 여러 개(kprobe, kprobe-multi, fentry, fexit) 공존할 때 실제 코드는 어떻게 구성되는가?

`pwru`의 소스를 보면 이 질문들에 대한 답이 거의 다 나온다.

---

## 3. 분석 범위와 파일 선정 이유

`pwru` 저장소는 Go User Space 코드와 eBPF C 코드가 함께 있는 구조다. 전체를 순서 없이 읽기보다는, "실행 흐름을 따라가며 필요한 파일만 선택"하는 방식으로 범위를 좁혔다.

| 파일 | 분석한 이유 |
|---|---|
| `main.go` | 전체 진입점. Flag 파싱부터 BTF 로드, BPF Program 로드, kprobe Attach, Event Loop까지 전체 흐름을 오케스트레이션 |
| `internal/pwru/utils.go` | `GetFuncs`: BTF를 순회하며 `sk_buff`를 인자로 받는 Kernel Function을 찾는 핵심 로직 |
| `internal/pwru/kprobe.go` | 찾은 함수 목록에 실제로 kprobe(또는 kprobe-multi)를 Attach/Detach하는 로직 |
| `bpf/kprobe_pwru.c` | Kernel에서 실행되는 실제 eBPF Program. Filter 체크, Event 생성, BPF Map 사용 방식 확인 |
| `internal/pwru/output.go` | Kernel에서 받은 Event를 사람이 읽는 로그로 변환하는 User Space 후처리 로직 |

`internal/libpcap/`(pcap 필터 문법을 eBPF Bytecode로 컴파일하는 부분)는 이번 분석에서는 제외했다. 이건 "Kernel Function Hooking"이라는 이번 주제보다는 별도의 작은 컴파일러에 가까워서, eBPF 자체를 이해하는 목적에서는 범위 밖이라고 판단했다.

전체 흐름을 먼저 정리하면 다음과 같다.

```text
User Space (Go)                         Kernel (eBPF C)
────────────────                        ────────────────
Flag 파싱
    │
    ▼
BTF 로드 (CO-RE)
    │
    ▼
sk_buff 받는 함수 찾기 (GetFuncs)
    │
    ▼
BPF Object 로드
    │
    ▼
Verifier 통과 ──────────────────▶  Kernel에 Program 존재
    │
    ▼
kprobe / kprobe-multi Attach ───▶  함수 호출 지점에 연결됨
                                          │
                                   (실제 Packet 발생)
                                          │
                                          ▼
                                   kprobe_skb_N 실행
                                          │
                                          ├── Filter 체크
                                          ├── Event 구성
                                          └── events Map에 push
    │
Event Loop (Polling) ◀──────────────────┘
    │
    ▼
output.go: Event → 로그 문자열
```

---

# 4. 진입점: `main.go`

## 4.1 BTF 로드 — CO-RE의 실제 사용처

```go
if flags.KernelBTF != "" {
    btfSpec, err = btf.LoadSpec(flags.KernelBTF)
} else {
    btfSpec, err = btf.LoadKernelSpec()
}
```

Chapter 5에서 CO-RE(Compile Once – Run Everywhere)는 "Kernel Struct의 Field Offset이 버전마다 달라도 재컴파일 없이 동작하게 해준다"는 개념으로 배웠다. `pwru`에서는 여기서 한 걸음 더 나아가 **BTF를 Struct Layout 확인뿐 아니라 "이 Kernel Function의 Signature가 무엇인지" 알아내는 데도 사용**한다. 이 부분은 뒤에서 `GetFuncs`를 보면서 자세히 확인했다.

## 4.2 Attach 대상 함수 찾기

```go
funcs, bpfmapFuncs, err := pwru.GetFuncs(flags.FilterFunc, btfSpec, flags.KMods, useKprobeMulti, flags.OutputBpfmap)
```

이 한 줄이 "skb를 인자로 받는 함수 수천 개를 찾는" 지점이다. 실제 구현은 5절에서 다룬다.

## 4.3 BPF Object 로드와 불필요한 Program 제거

```go
bpfSpec, err := LoadKProbePWRU()

if useKprobeMulti {
    for i := 1; i <= 5; i++ {
        delete(bpfSpec.Programs, fmt.Sprintf("kprobe_skb_%d", i))
    }
} else {
    for i := 1; i <= 5; i++ {
        delete(bpfSpec.Programs, fmt.Sprintf("kprobe_multi_skb_%d", i))
    }
}
```

`kprobe_pwru.c`(정확히는 컴파일된 `.o`)에는 kprobe용/kprobe-multi용, TC/XDP Tracing용(`fentry`/`fexit`), skb Lifetime 추적용 등 **하나의 ELF 안에 여러 Program이 함께 들어있고**, 사용자가 선택하지 않은 Backend/Feature에 해당하는 Program은 로드 전에 Spec에서 지워버린다. Chapter 7에서 배운 여러 Attachment Type(kprobe, kprobe-multi, fentry, fexit)이 실제로는 하나의 프로젝트 안에 전부 구현돼 있고, 실행 시점의 Kernel 지원 여부와 사용자 Option에 따라 그중 일부만 골라서 로드하는 구조라는 점이 흥미로웠다.

## 4.4 Verifier 통과

```go
opts.Programs.KernelTypes = btfSpec
opts.Programs.LogLevel = ebpf.LogLevelInstruction
coll, err := ebpf.NewCollectionWithOptions(bpfSpec, opts)
if err != nil {
    var ve *ebpf.VerifierError
    if errors.As(err, &ve) {
        verifierLog = fmt.Sprintf("Verifier error: %+v\n", ve)
    }
    ...
}
```

Chapter 6에서 배운 Verifier가 여기서 실제로 동작한다. 흥미로운 점은 `pwru`가 여러 개의 서로 다른 Program(5개의 kprobe Position별 변형 + fentry/fexit 변형들)을 **한 번에 하나의 Collection으로 로드**한다는 것인데, 이 경우 그중 단 하나라도 Verifier를 통과하지 못하면 전체 로드가 실패한다. 그래서 앞 단계(4.3)에서 필요 없는 Program을 미리 걸러내는 것이 단순히 로딩 속도 문제가 아니라 **Verifier 대상 자체를 줄이는 것**이라는 의미로도 읽혔다.

## 4.5 Event Loop

```go
events := coll.Maps["events"]
for i := flags.OutputLimitLines; i > 0 || runForever; i-- {
    for {
        if err := events.LookupAndDelete(nil, &event); err == nil {
            break
        }
        select {
        case <-ctx.Done():
            return nil
        case <-time.After(time.Microsecond):
            continue
        }
    }
    ...
    output.Print(&event)
}
```

Chapter 2/4에서 Perf Buffer/Ring Buffer로 Event를 받는 예제를 봤는데, `pwru`는 조금 다르게 **`BPF_MAP_TYPE_QUEUE`를 Polling**하는 방식을 사용한다(`LookupAndDelete`로 큐에서 하나씩 꺼내면서 지움). 뒤에서 `kprobe_pwru.c`의 Map 정의를 보면 왜 Perf/Ring Buffer 대신 Queue를 썼는지에 대한 힌트가 있다.

---

# 5. `GetFuncs`: BTF로 "skb를 인자로 받는 함수" 찾기

`internal/pwru/utils.go`에 있는 이 함수가 이번 분석에서 가장 흥미로웠던 부분이다.

```go
for typ, err := range it.iter {          // BTF의 모든 Type을 순회
    fn, ok := typ.(*btf.Func)
    if !ok {
        continue
    }
    fnName := string(fn.Name)

    if pattern != "" && reg.FindString(fnName) != fnName {
        continue                          // --filter-func 정규식 매치
    }

    if _, ok := availableFuncs[fnName]; !ok {
        continue                          // ftrace가 실제로 Attach 가능하다고 보고한 함수인지 확인
    }

    fnProto := fn.Type.(*btf.FuncProto)
    i := 1
    for _, p := range fnProto.Params {
        if ptr, ok := p.Type.(*btf.Pointer); ok {
            if strct, ok := ptr.Target.(*btf.Struct); ok {
                if strct.Name == "sk_buff" && i <= 5 {
                    funcs[fnName] = i      // 이 함수는 i번째 Argument로 sk_buff*를 받는다
                }
            }
        }
        i += 1
    }
}
```

이 코드에서 확인한 것들을 정리하면 다음과 같다.

1. **BTF는 Struct Layout 정보뿐 아니라 Function Signature(Parameter Type 목록)도 담고 있다.** `spec.All()`로 모든 BTF Type을 순회하다가 `*btf.Func` Type만 골라내고, 그 `FuncProto`의 Parameter 목록에서 "Pointer to Struct sk_buff"를 찾는다.
2. **BTF에 함수가 존재한다고 바로 kprobe를 걸 수 있는 것은 아니다.** `/sys/kernel/tracing/available_filter_functions`(ftrace가 관리하는 목록)에 있는 함수인지 한 번 더 확인한다. Inline된 함수 등은 BTF에는 보이지만 실제로 Attach할 수 없기 때문이다.
3. **sk_buff가 몇 번째 Argument인지까지 기록한다.** 그리고 이 위치(`i`)별로 함수를 그룹핑한다(`GetFuncsByPos`).

세 번째가 이번 분석의 핵심 발견이었다.

---

# 6. 왜 kprobe Program이 5개나 있는가

`internal/pwru/kprobe.go`의 `NewKprober`를 보면 Position별로 서로 다른 Program을 찾아 붙인다.

```go
fn, ok := coll.Programs[fmt.Sprintf("%s_skb_%d", probeMethod, pos)]
```

그리고 `bpf/kprobe_pwru.c`에는 이 Program들이 C Macro로 찍혀 있다.

```c
#define PWRU_ADD_KPROBE(X)                                                  \
SEC("kprobe/skb-" #X)                                                       \
    int kprobe_skb_##X(struct pt_regs *ctx) {                              \
        struct sk_buff *skb = (struct sk_buff *) PT_REGS_PARM##X(ctx);     \
        return kprobe_skb(skb, ctx, NULL, false);                          \
    }
    ...
PWRU_ADD_KPROBE(1)
PWRU_ADD_KPROBE(2)
PWRU_ADD_KPROBE(3)
PWRU_ADD_KPROBE(4)
PWRU_ADD_KPROBE(5)
```

kprobe Handler는 Kernel Function의 Argument를 CPU Register(`pt_regs`)에서 직접 읽는데, x86_64 Calling Convention에서는 Argument 순서에 따라 읽어야 할 Register가 다르다(1번째는 `rdi`, 2번째는 `rsi`, ...). `PT_REGS_PARM1(ctx)`, `PT_REGS_PARM2(ctx)`처럼 Argument 위치별로 다른 Macro를 써야 하기 때문에, eBPF Program은 실행 시점에 "이 함수는 몇 번째 Argument에 skb가 있는지"를 동적으로 판단할 수 없다.

그래서 `pwru`는 **컴파일 타임에 Position별 변형 Program을 5개(`kprobe_skb_1`~`kprobe_skb_5`) 미리 만들어두고, User Space에서 BTF로 알아낸 Position에 맞는 Program을 골라 Attach**하는 방식으로 이 제약을 해결한다.

```text
GetFuncs (BTF 분석)
    │
    ├── ip_rcv(skb)            → position 1
    ├── some_func(dev, skb)    → position 2
    └── other_func(a, b, skb)  → position 3
    │
    ▼
GetFuncsByPos (Position별 그룹핑)
    │
    ▼
kprobe_skb_1 ◀── position 1 함수들 Attach
kprobe_skb_2 ◀── position 2 함수들 Attach
kprobe_skb_3 ◀── position 3 함수들 Attach
```

Chapter 3(Anatomy of an eBPF Program)에서 "하나의 소스에 여러 Program이 들어갈 수 있다"는 내용을 배웠는데, 그 이유가 단순히 기능 분리가 아니라 **eBPF/kprobe의 저수준 제약(Register 기반 Argument 접근)을 우회하기 위한 설계**일 수도 있다는 걸 실제 코드로 확인한 셈이다.

---

# 7. Kernel 쪽 핵심 로직: `handle_everything`

`bpf/kprobe_pwru.c`의 모든 kprobe Handler는 결국 이 함수를 거친다.

```c
static __noinline bool
handle_everything(struct sk_buff *skb, void *ctx, struct event_t *event,
                   u64 *_stackid, const bool is_kprobe) {
    ...
    if (cfg->is_set) {
        if (cfg->track_skb && bpf_map_lookup_elem(&skb_addresses, &skb_addr)) {
            tracked_by = ...;
            goto cont;                 // 이미 추적 중인 skb라면 Filter 재검사 없이 통과
        }
        if (filter(skb)) {
            tracked_by = TRACKED_BY_FILTER;
            goto cont;
        }
        return false;                  // Filter 불통과 → Event 발생 안 함
cont:
        set_output(ctx, skb, event);
    }

    if (cfg->track_skb && tracked_by == TRACKED_BY_FILTER) {
        bpf_map_update_elem(&skb_addresses, &skb_addr, &TRUE, BPF_ANY);
    }
    ...
    return true;
}
```

여기서 확인한 것은 두 가지다.

**1) Filter는 매번 재평가되지 않는다.** 최초에 Filter 조건(`--filter-mark`, pcap 표현식 등)을 만족한 skb는 `skb_addresses`라는 BPF Map에 등록해두고, 이후 이 skb가 (NAT나 Tunnel Decapsulation으로 IP가 바뀐 뒤에도) 다른 Kernel Function을 지날 때는 **Filter 재검사 없이 Map 조회만으로 계속 추적**한다. Filter 조건 자체는 최초 진입점에서만 의미가 있고, 그 뒤로는 "이 Packet을 계속 따라가야 하는가"라는 State 문제로 바뀌는 것이다.

**2) BPF Map이 여기서도 Cross-Invocation State Store로 쓰인다.** Pixie 분석 때 Entry/Return Probe 사이의 State를 Map으로 연결했던 것과 같은 패턴인데, `pwru`에서는 "서로 다른 kprobe(서로 다른 함수, 서로 다른 실행 시점)들 사이에서 같은 Packet을 추적"하는 데 쓰인다.

---

# 8. 고정 크기 Event와 가변 크기 Data — 사이드 Map 패턴

`event_t` 구조체와 Event Map 정의를 보면 흥미로운 설계가 있다.

```c
struct event_t {
    u32 pid;
    u32 type;
    u64 addr;
    u64 skb_addr;
    u64 ts;
    u64 print_skb_id;        // ← 실제 skb 덤프가 아니라 "ID"만 들어있음
    u64 print_shinfo_id;
    u64 print_bpfmap_id;
    struct skb_meta meta;
    struct tuple tuple;
    s64 print_stack_id;
    ...
} __attribute__((packed));

struct {
    __uint(type, BPF_MAP_TYPE_QUEUE);
    __type(value, struct event_t);
    __uint(max_entries, 10000);
} events SEC(".maps");
```

`BPF_MAP_TYPE_QUEUE`는 고정 크기 Value만 담을 수 있는데, `--output-skb`(skb 전체 덤프)나 `--output-stack`(Stack Trace) 같은 옵션은 길이가 가변적이라 `event_t` 안에 직접 넣을 수 없다. `pwru`는 이를 다음과 같이 해결한다.

```text
Kernel (kprobe_pwru.c)
────────────────────────
가변 길이 데이터(skb 덤프, Stack 등)
        │
        ▼
별도의 BPF Map (print_skb_map, print_stack_map, ...)
에 저장하고, 그 Map의 Key(ID)만
        │
        ▼
event_t.print_skb_id 필드에 기록
        │
        ▼
event_t 전체를 events Queue에 push

════════════════════════ Kernel / User Space ════════════════════════

User Space (output.go)
────────────────────────
event_t를 Queue에서 꺼냄
        │
        ▼
event.PrintSkbId로 print_skb_map을 재조회
        │
        ▼
실제 가변 길이 데이터 획득 → 출력 후 Map에서 삭제
```

```go
// internal/pwru/output.go
func getSkbData(event *Event, o *output) (skbData string) {
    id := uint64(event.PrintSkbId)
    b, err := o.printSkbMap.LookupBytes(&id)
    ...
    defer o.printSkbMap.Delete(&id)
    ...
}
```

Chapter 2/4에서 BPF Map을 "Kernel과 User Space가 값을 주고받는 Key-Value Store"로 배웠는데, 여기서는 **"고정 크기 채널로 가변 크기 데이터를 우회 전달하기 위한 간접 참조(Indirection)"**로 쓰이고 있었다. 이 패턴은 pwru만의 특이한 사례라기보다, 고정 크기 Event Channel(Perf/Ring Buffer, Queue 등)과 가변 길이 Payload를 함께 다뤄야 하는 eBPF Tool 전반에서 재사용 가능한 아이디어로 보인다.

---

# 9. User Space: `output.go`의 후처리

Kernel에서 넘어온 `event_t`는 대부분 숫자(주소, ID)뿐이라 그 자체로는 읽기 어렵다. `output.go`의 `Print()`가 이를 사람이 읽는 한 줄로 바꾼다.

```go
func getOutFuncName(o *output, event *Event, addr uint64) string {
    if ksym, ok := o.addr2name.Addr2NameMap[addr]; ok {
        funcName = ksym.name
    } else {
        funcName = fmt.Sprintf("0x%x", addr)
    }
    ...
}
```

`event.Addr`는 그냥 숫자 Kernel Address일 뿐인데, 이걸 `"tcp_v4_rcv"` 같은 이름으로 바꾸는 것이 `main.go`에서 미리 만들어 둔 `addr2name`(`/proc/kallsyms`를 한 번 파싱해 만든 주소→이름 Map)이다. Event마다 Symbol Table을 다시 읽지 않고 재사용하는 구조다.

더 흥미로운 부분은 Drop Reason 처리다.

```go
if funcName == "kfree_skb_reason" {
    if reason, ok := o.kfreeReasons[event.ParamSecond]; ok {
        outFuncName = fmt.Sprintf("%s(%s)", funcName, reason)
    }
}
```

`event.ParamSecond`는 `kprobe_pwru.c`에서 `PT_REGS_PARM2(ctx)`로 읽어 그대로 넘긴 두 번째 Argument 값이다. `kfree_skb_reason` 함수에서 이 값은 Kernel의 `skb_drop_reason` Enum 값인데, `output.go`는 이 값을 BTF Enum 정보(`getKFreeSKBReasons`)로 이름을 복원해서 `kfree_skb_reason(NO_SOCKET)`처럼 보여준다. Packet이 "왜" Drop됐는지까지 보여주는 `pwru`의 대표 기능이 "Argument 값을 그대로 전달 → User Space에서 BTF Enum으로 재해석"하는 조합으로 구현돼 있었다.

---

# 10. pwru 전체 eBPF 흐름

```text
┌───────────────────────────────┐
│ Kernel Function 수천 개        │
│ (ip_rcv, tcp_v4_rcv, ...)      │
└───────────────┬───────────────┘
                │ 함수 호출
                ▼
        kprobe_skb_1 ~ 5
     (BTF로 찾은 Argument Position에 맞춰 Attach됨)
                │
                ▼
        handle_everything()
                │
        ┌───────┴────────┐
        │                │
   이미 추적 중?      filter() 통과?
        │                │
        └───────┬────────┘
                ▼
          event_t 구성
    (가변 데이터는 별도 Map에 저장, ID만 기록)
                │
                ▼
          events (BPF_MAP_TYPE_QUEUE)
════════════════╪════════════════ Kernel / User Space
                ▼
       main.go: LookupAndDelete Polling
                │
                ▼
       output.go: addr2name / 별도 Map 재조회
                │
                ▼
          사람이 읽는 로그 한 줄
```

---

# 11. Learning eBPF Chapter 1~7과 연결해서 보기

| Learning eBPF에서 학습한 개념 | pwru에서 확인한 형태 |
|---|---|
| Chapter 1: eBPF는 Kernel 코드를 고치지 않고 동작을 관찰 | Kernel Function 재컴파일 없이 kprobe로 수천 개 지점에 Hook |
| Chapter 2: BPF Map, kprobe | `skb_addresses` Map으로 추적 State 유지, kprobe로 함수 진입 감지 |
| Chapter 3: 하나의 소스에 여러 Program | 하나의 `.c`에 kprobe/kprobe-multi/fentry/fexit 변형이 함께 존재, 필요한 것만 로드 |
| Chapter 4: bpf() System Call, BPF Map | `ebpf.NewCollectionWithOptions`(Go 라이브러리 위임), `BPF_MAP_TYPE_QUEUE`로 Event 전달 |
| Chapter 5: CO-RE, BTF | Struct Offset 흡수뿐 아니라 **Function Signature 조회**(`GetFuncs`)에도 BTF 사용 |
| Chapter 6: Verifier | 여러 Program을 한 Collection으로 로드 시 하나라도 실패하면 전체 실패 → 불필요한 Program 사전 제거 |
| Chapter 7: kprobe / kprobe-multi / fentry / fexit | Kernel 지원 여부(`HaveBPFLinkKprobeMulti`)에 따라 Backend를 런타임에 선택 |

Pixie를 분석했을 때는 "여러 System Call의 Entry/Return을 Map으로 연결해 Connection State를 만드는" State Machine 관점이 두드러졌다면, `pwru`에서는 **"Attach 대상 자체를 코드에 고정하지 않고 BTF로 런타임에 동적으로 찾아낸다"**는 점과, **"Register 기반 Argument 접근이라는 kprobe의 저수준 제약을 Position별 Program 복제로 우회한다"**는 점이 가장 인상적이었다.

---

# 12. 정리

이번 분석에서 확인한 pwru의 핵심 아이디어는 다음 세 가지로 요약할 수 있다.

1. **BTF는 Struct Layout뿐 아니라 Function Signature 정보도 제공하며**, 이를 이용하면 "특정 Type의 Argument를 받는 Kernel Function"을 런타임에 동적으로 찾아낼 수 있다.
2. **kprobe는 Register 기반으로 Argument를 읽기 때문에, Argument 위치가 다른 함수들을 하나의 Program으로 처리할 수 없다.** pwru는 이를 Position별 Program 복제(`kprobe_skb_1`~`5`)로 해결한다.
3. **고정 크기 Event Channel(Queue/Perf/Ring Buffer)로 가변 길이 데이터를 보내야 할 때는, 가변 데이터를 별도 Map에 저장하고 ID만 고정 크기 Event에 실어 보내는 간접 참조 패턴이 반복적으로 쓰인다.**

Chapter 1~7에서 배운 개별 개념들이 "완성된 Production Tool 하나"를 이해하는 데 그대로 쓰였다는 점에서, 이번 분석은 개념을 다시 확인하는 과정이자 동시에 각 개념이 실제로 어떤 제약과 트레이드오프 아래에서 쓰이는지 확인하는 과정이었다.
