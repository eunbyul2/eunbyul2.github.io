---
layout: post
title: "[eBPF Open Source] Cilium pwru로 보는 kprobe 기반 Packet Tracing"
date: 2026-09-05 18:05:00 +0900
categories: [eBPF, Cilium, Observability]
tags: [eBPF, pwru, Cilium, kprobe, kprobe-multi, BTF, CO-RE, BPF Map, sk_buff, Network Debugging]
published: true
---

# pwru의 eBPF 활용 분석

Learning eBPF Chapter 1~7에서는 eBPF Program이 Kernel Event에 어떻게 연결되는지, BPF Map으로 State를 어떻게 공유하는지, CO-RE/BTF로 Kernel 버전 차이를 어떻게 흡수하는지, Verifier가 무엇을 검증하는지, kprobe/fentry/fexit 같은 Attachment Type이 어떻게 다른지를 각각 따로 학습했다.

이번 스터디 주제는 "eBPF 오픈소스 분석"이었고, 이번에는 Cilium 산하의 [`pwru`](https://github.com/cilium/pwru)를 분석 대상으로 선택했다. Pixie의 Socket Tracer를 분석했을 때는 System Call Entry/Return을 연결하는 State Machine 관점이 중심이었다면, 

이번 `pwru`는 **"수천 개의 Kernel Function에 kprobe를 걸어 하나의 Packet(`sk_buff`)이 Kernel 내부를 어떻게 통과하는지 추적한다"**

---

## 0. 시작하기 전에: skb와 kprobe란?

**`sk_buff`(skb)**는 리눅스 커널이 Network Packet 하나를 표현하는 C Struct다. 
패킷이 Network Card로 들어오는 순간부터, 라우팅/방화벽/NAT 같은 여러 처리 단계를 거쳐 소켓으로 전달되거나 밖으로 나가거나 Drop될 때까지, **이 과정 내내 패킷은 하나의 `sk_buff` 포인터로 여러 Kernel Function 사이를 옮겨다닌다.** `ip_rcv(struct sk_buff *skb, ...)`, `tcp_v4_rcv(struct sk_buff *skb)`처럼 Kernel Function Signature에 실제로 이 타입이 등장한다. 택배 상자에 붙은 꼬리표라고 생각하면, 패킷이라는 상자가 커널이라는 컨베이어 벨트 위 여러 구간(Kernel Function)을 지나는 동안 이 꼬리표(skb 포인터)가 계속 따라다니는 셈이다.

**kprobe**는 특정 Kernel Function이 호출되는 시점에 임의의 코드를 끼워 실행할 수 있게 해주는 Kernel 기능이다. 원리는 디버거의 Breakpoint와 비슷한데(함수 시작 부분 명령어를 트랩 명령어로 바꿔치기해서 CPU가 그 지점에서 걸리게 만든다), 사람이 멈춰서 들여다보는 대신 **등록해둔 콜백(여기서는 eBPF Program)을 자동으로 실행하고 원래 함수 실행을 계속 이어간다**는 점이 다르다. 그래서 Kernel Source를 고치거나 재컴파일하지 않고도, 실행 중인 Kernel Function 하나하나에 관찰 코드를 동적으로 붙일 수 있다.

이 두 개를 합치면 `pwru`가 하는 일이 정확히 정의된다. 

**"`sk_buff`를 인자로 받는 Kernel Function을 찾아서 전부 kprobe를 걸어두고, 패킷이 그 함수들을 지날 때마다 관찰한다."**

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

## 4.0 실행 준비: RLIMIT과 Graceful Shutdown

BTF를 로드하기 전에, `run()` 맨 앞에는 eBPF와 직접 관련 없어 보이는 준비 코드가 먼저 나온다.

```go
if err := unix.Setrlimit(unix.RLIMIT_NOFILE, &unix.Rlimit{
	Cur: 8192,
	Max: 8192,
}); err != nil {
	return fmt.Errorf("failed to set temporary RLIMIT_NOFILE: %w", err)
}
if err := rlimit.RemoveMemlock(); err != nil {
	return fmt.Errorf("failed to set temporary RLIMIT_MEMLOCK: %w", err)
}

ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
```

처음 봤을 때는 그냥 상투적인 에러 처리 코드로 보였는데, 각 줄이 실제로 필요한 이유가 있었다.

- **`RLIMIT_NOFILE`**: Linux는 BPF Map과 BPF Program 하나하나를 File Descriptor로 관리한다. `pwru`는 곧 Kernel Function 수천 개에 kprobe를 걸 예정이라 그만큼 fd가 많이 필요한데, 기본 제한(보통 1024개)에 걸리지 않도록 미리 8192로 올려둔다.
- **`RemoveMemlock()`**: BPF Map은 Kernel Memory에 상주하는데, 예전 Kernel은 "잠긴(Locked, Swap 불가) 메모리를 얼마나 쓸 수 있는가"에도 제한을 뒀다. eBPF Program/Map을 위한 메모리가 이 제한에 걸려 로드가 실패하는 것을 막기 위해 미리 풀어둔다.
- **`signal.NotifyContext` + `defer stop()`**: `Ctrl+C`(`SIGINT`)를 누르면 기본적으로 프로그램은 정리할 시간 없이 즉시 종료된다. 하지만 `pwru`는 지금 Kernel에 kprobe 수천 개를 걸어놓은 상태라, 즉사 대신 **"종료 신호가 오면 `ctx`를 통해 알려달라"**고 등록해서 프로그램이 스스로 순서대로 kprobe를 떼어내고(Detach) 종료할 기회를 만든다. 이 `ctx`는 뒤에서 Event Loop의 `select { case <-ctx.Done(): ... }`에서 그대로 쓰인다. `defer stop()`은 이 신호 감시를 함수가 끝날 때 반드시 해제하기 위한 짝이다.

즉 이 블록은 eBPF 로직이 아니라, **"곧 대량의 Kernel 자원을 쓸 것이므로 시스템 제한을 미리 풀어두고, 강제 종료 대신 정상 종료(Graceful Shutdown)를 준비하는" 사전 작업**이다. 이후 §4.3~4.4에서 보듯 이 프로그램은 Verifier를 통과해야 하는 여러 Program과 다수의 Map을 한 번에 다루기 때문에, 이런 자원 준비가 실제로 없으면 중간에 실패하기 쉽다.

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

## 6.1 수천 개를 어떻게 빠르게 붙이는가 — Batch Attach와 kprobe-multi

Position별로 맞는 Program을 찾았다고 끝이 아니다. 실제로 수천 개의 Kernel Function 각각에 kprobe를 붙이는 작업 자체가 성능 이슈가 될 수 있는데, `internal/pwru/kprobe.go`를 보면 pwru는 두 가지 방식을 모두 지원한다.

**방식 1: kprobe — 개별 Attach를 병렬로**

```go
func attachKprobes(ctx context.Context, bar *pb.ProgressBar, kprobes []Kprobe) (...) {
	for _, kprobe := range kprobes {
		kp, err = link.Kprobe(kprobe.hookFunc, kprobe.Prog, nil)   // 함수 하나당 Syscall 한 번
		...
	}
}

func AttachKprobes(ctx context.Context, bar *pb.ProgressBar, kps []Kprobe, batch uint) (...) {
	...
	for i = 0; i+batch < uint(len(kprobes)); i += batch {
		kps := kprobes[i : i+batch]
		errg.Go(func() error { return attaching(kps) })   // batch(기본 10개)개씩 나눠 goroutine으로 동시 실행
	}
	...
}
```

`link.Kprobe()`는 함수 이름 하나당 별도의 Kernel Syscall이 필요하다. 함수 수천 개를 순서대로 하나씩 부르면 그만큼 느려지기 때문에, `errgroup`으로 `--filter-kprobe-batch`(기본 10개) 단위로 잘라 **여러 Batch를 동시에 병렬로 Attach**한다. Syscall 횟수 자체는 줄지 않지만, 병렬로 처리해서 전체 소요 시간을 줄이는 방식이다. `DetachKprobes()`도 같은 방식으로 배치를 나눠 병렬로 해제한다.

**방식 2: kprobe-multi — 한 번의 Syscall로 전부**

```go
func AttachKprobeMulti(ctx context.Context, bar *pb.ProgressBar, kprobes []Kprobe, a2n Addr2Name) (...) {
	for _, kp := range kprobes {
		addrs := make([]uintptr, 0, len(kp.HookFuncs))
		for _, fn := range kp.HookFuncs {
			addr, _ := a2n.Name2AddrMap[fn]
			addrs = append(addrs, addr...)          // 함수 이름들을 전부 주소로 변환해 하나의 배열로 모음
		}
		opts := link.KprobeMultiOptions{Addresses: addrs}
		l, err := link.KprobeMulti(kp.Prog, opts)     // Position별 Program 하나당 Syscall 한 번뿐
	}
}
```

Kernel 5.18+에서 지원하는 `kprobe-multi`는 접근 자체가 다르다. 개별 함수 이름이 아니라 **주소 배열 전체를 한 번의 Syscall로 Kernel에 넘긴다.** Position별 Program(`kprobe_skb_1`~`5`)마다 딱 한 번씩, 총 5번의 Syscall만으로 수천 개의 Attach Point를 한 번에 등록하는 셈이라, 개별 Attach보다 압도적으로 빠르다. `main.go`에서 `HaveBPFLinkKprobeMulti()`로 Kernel이 이 기능을 지원하는지 확인해서, 지원하면 자동으로 이 경로를 쓴다.

즉 "수천 개의 kprobe를 거는 것이 성능에 어떤 영향을 주는가"에 대한 pwru의 답은 두 겹이다 — **오래된 Kernel에서는 병렬화(Batch + Goroutine)로 완화하고, 최신 Kernel에서는 아예 API 차원에서 다건 처리(kprobe-multi)를 지원받아 Syscall 수 자체를 줄인다.**

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

위 코드에서 `...`로 생략한 부분에는 사실 추적 방식이 하나 더 있다. `skb_addresses`(주소 기반) 조회가 실패하면, `--filter-track-skb-by-stackid` 옵션이 켜져 있는 경우 `stackid_skb`라는 또 다른 Map을 Stack ID로 조회한다. 이게 필요한 이유는, Bridge를 통과하는 Packet처럼 원본 skb가 Kernel 내부에서 Free된 뒤 새로운 skb로 다시 생성되는 경우가 있기 때문이다. 이 경우 skb 주소 자체가 바뀌어버려서 `skb_addresses` Map으로는 더 이상 같은 Packet임을 알아볼 수 없다. 대신 `get_stackid()`가 계산한 Call Stack 기반 ID로 연결해 추적을 이어간다. 즉 `pwru`는 "주소가 바뀌지 않는 일반적인 경우"와 "주소 자체가 무의미해지는 경우"를 서로 다른 Key(주소 vs Stack ID)로 나눠 대응하고 있다.

## 7.1 `filter()`: 세 조건의 AND

`handle_everything`에서 호출하는 `filter(skb)`의 실제 정의는 아래와 같다.

```c
static __always_inline bool
filter(struct sk_buff *skb) {
	return filter_pcap(skb) && filter_meta(skb) && filter_skb_expr(skb);
}
```

세 개의 서로 다른 필터 조건을 `&&`로 묶은 것뿐이다.

- **`filter_pcap`**: `--filter-pcap`(tcpdump 문법으로 쓰는 pcap Expression)을 체크. 이 부분은 `libpcap` 패키지가 사용자의 pcap 문자열을 런타임에 별도 eBPF Bytecode로 컴파일해서 이 자리에 주입해둔 것이다.
- **`filter_meta`**: `--filter-mark`, `--filter-netns`, `--filter-ifname` 등 skb의 mark/netns/ifindex 값을 직접 비교.
- **`filter_skb_expr`**: `--filter-skb-expr`로 사용자가 지정한 임의의 skb Field 조건.

세 조건이 모두 참이어야 이 Packet을 추적 대상으로 판단한다. §6의 Position별 Program 분기와 마찬가지로, 사용자가 CLI에서 준 여러 종류의 조건이 결국 하나의 Boolean 함수로 합쳐지는 구조다.

## 7.2 `set_output()`: 출력 Option과 1:1 대응

Filter를 통과한 skb에 대해서만 실행되는 `set_output()`은 다음과 같다.

```c
static __always_inline void
set_output(void *ctx, struct sk_buff *skb, struct event_t *event) {
	if (cfg->output_meta)  set_meta(skb, &event->meta);
	if (cfg->output_tuple) set_tuple(skb, &event->tuple);
	if (cfg->output_skb)   set_skb_btf(skb, &event->print_skb_id);
	if (cfg->output_stack) event->print_stack_id = bpf_get_stackid(ctx, &print_stack_map, BPF_F_FAST_STACK_CMP);
	...
}
```

CLI Option(`--output-meta`, `--output-tuple`, `--output-skb`, `--output-stack` ...) 하나하나가 이 `if`문 하나와 정확히 대응한다. 사용자가 요청하지 않은 정보는 애초에 채우지도 않는다 — Kernel 안에서 도는 코드이니만큼, 필요 없는 작업을 줄여 Overhead를 최소화하려는 의도로 보인다.

`set_meta()` 내부를 보면 Chapter 5의 CO-RE가 가장 전형적으로 쓰이는 지점을 확인할 수 있다.

```c
static __always_inline void
set_meta(struct sk_buff *skb, struct skb_meta *meta) {
	meta->mark = BPF_CORE_READ(skb, mark);
	meta->ifindex = BPF_CORE_READ(skb, dev, ifindex);
	meta->mtu = BPF_CORE_READ(skb, dev, mtu);
	...
}
```

`skb->mark`를 `skb->mark`로 직접 읽지 않고 `BPF_CORE_READ(skb, mark)` 매크로로 읽는다. `sk_buff` 구조체 안 `mark` 필드의 실제 Offset은 Kernel Version마다 달라질 수 있는데, 이 매크로가 실행 시점에 BTF 정보로 현재 Kernel의 실제 Offset을 찾아 안전하게 읽어준다. §4.1에서 BTF를 "Function Signature 탐색"에 쓰는 것을 봤다면, 여기서는 BTF/CO-RE의 훨씬 더 흔한 용례인 "Struct Field Access"를 확인할 수 있다.

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