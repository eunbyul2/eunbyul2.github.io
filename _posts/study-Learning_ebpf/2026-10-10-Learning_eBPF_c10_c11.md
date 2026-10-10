---
layout: post
title: "[Learning eBPF] Chapter 10-11. eBPF Programming & The Future Evolution of eBPF"
date: 2026-10-10 13:00:00 +0900
categories: [eBPF, Programming]
tags: [eBPF, Programming, bpftrace, BCC, libbpf, CO-RE, Go, Rust, Cilium, Windows, eBPF Foundation]
published: true
---

# Learning eBPF Chapter 10–11

## eBPF Programming & The Future Evolution of eBPF

## 1. eBPF Programming의 기본 구조

일반적인 eBPF 애플리케이션은 크게 두 부분으로 구성된다.

- **Kernel Space**: 이벤트가 발생했을 때 실행할 eBPF 프로그램을 작성한다.
- **User Space**: 프로그램을 커널에 로드하고, 이벤트에 연결하고, Map을 통해 데이터를 주고받으며 결과를 출력한다.

예를 들어 특정 프로세스가 파일을 여는 동작을 관찰한다고 생각해 볼 수 있다.

```text
[User Space]
  eBPF 프로그램 로드 및 이벤트 연결
  Map에 기록된 데이터를 읽고 화면에 출력
                 ↑ ↓
             [eBPF Map]
                 ↑ ↓
[Kernel Space]
  open/openat 시스템 콜 이벤트 감지
  PID, 파일 이름 등의 정보 수집
```

이처럼 **커널에서 무엇을 수집할지**와 **사용자에게 어떻게 보여줄지**를 함께 구현해야 한다. 다만 `bpftrace`는 이 구분을 상당 부분 감춰 주기 때문에 입문자가 비교적 쉽게 사용할 수 있다.

---

## 2. bpftrace

`bpftrace`는 eBPF를 활용한 고수준 트레이싱 언어이자 명령줄 도구다. `awk`, C, DTrace 등의 영향을 받았으며, 별도의 복잡한 애플리케이션을 작성하지 않고도 커널 이벤트를 관찰할 수 있다.

### 이벤트 확인

```bash
sudo bpftrace -l '*execve*'
```

이 명령은 이름에 `execve`가 들어가는 tracepoint나 kprobe 등을 검색한다. 예를 들어 `tracepoint:syscalls:sys_enter_execve`, `kprobe:do_execve` 같은 이벤트 연결 지점을 찾을 수 있다. **출력되는 이벤트는 커널 버전과 환경에 따라 다를 수 있다.**

### 실행 횟수 추적

교재에서는 다음과 같은 한 줄짜리 예제를 소개한다.

```bash
sudo bpftrace -e 'kprobe:do_execve { @[comm] = count(); }'
```

- `kprobe:do_execve`: `do_execve` 함수 진입 시 실행한다.
- `comm`: 현재 작업의 명령 이름이다.
- `@[comm]`: 명령 이름을 Key로 사용하는 집계 Map이다.
- `count()`: 이벤트 발생 횟수를 누적한다.

예를 들어 `sh`와 `node`가 몇 번 감지됐는지 출력하는 방식이다. 한 줄만 작성했지만 내부에서는 eBPF 프로그램과 Map을 이용한다.

### opensnoop과 eBPF Map

교재의 `opensnoop.bt` 예제는 `open()`과 `openat()` 시스템 콜의 **진입(enter)** 및 **종료(exit)** 이벤트를 함께 추적한다.

```bpftrace
tracepoint:syscalls:sys_enter_open,
tracepoint:syscalls:sys_enter_openat
{
    @filename[tid] = args->filename;
}

tracepoint:syscalls:sys_exit_open,
tracepoint:syscalls:sys_exit_openat
/@filename[tid]/
{
    $ret = args->ret;
    $fd = $ret >= 0 ? $ret : -1;
    $errno = $ret >= 0 ? 0 : -$ret;

    printf("%-6d %-16s %4d %3d %s\n",
           pid, comm, $fd, $errno, str(@filename[tid]));
    delete(@filename[tid]);
}
```

파일 열기 syscall이 시작되면 `tid`를 Key로 파일 이름을 Map에 기록한다.

syscall이 종료되면 같은 `tid`로 파일 이름을 조회하고, 반환값을 확인해서 File Descriptor 또는 Error Number를 출력한다.

마지막으로 `delete()`를 이용해 사용한 Entry를 삭제한다.

여기서 **같은 시스템 콜의 진입과 종료를 연결하기 위해 Map을 사용한다**는 점이 중요하다. 하나의 스크립트가 여러 eBPF 프로그램을 이벤트마다 연결할 수도 있다.

**예제 참고:** 위 코드는 학습용으로 교재 발췌문을 간소화한 것이다. 교재에 실린 `$ret > 0` 조건은 유효한 FD `0`을 성공으로 처리하지 못하므로, 여기서는 `>= 0`으로 표기했다. 실제 운영 환경에서 사용하기 전에는 스크립트 및 커널별 호환성을 확인해야 한다. 또한 시스템 콜 진입 시 읽은 파일 이름을 보안 판단의 유일한 근거로 사용하면 TOCTOU 문제가 발생할 수 있다.

`bpftool prog list`와 `bpftool map list`를 사용하면 `bpftrace`가 커널에 로드한 프로그램과 Map도 확인할 수 있다. 교재에서는 파일 이름을 저장하는 **Hash Map**과 결과 출력을 위한 **perf event array**를 확인한다.

`bpftrace`는 트레이싱에 유용하지만, eBPF의 모든 프로그램 유형과 사용 사례를 처리하는 범용 개발 환경은 아니다. 본격적인 도구를 개발하려면 언어와 라이브러리를 선택해야 한다.

---

## 3. Language Choices for eBPF

교재에서는 eBPF 바이트코드를 직접 작성할 수도 있지만, 실제로는 주로 **C 또는 Rust** 코드를 컴파일해 eBPF 바이트코드를 만든다고 설명한다.

C는 Clang/LLVM을 이용해 eBPF Bytecode를 생성하며, 커널 측 eBPF Program 작성에 널리 사용된다. 다만 Pointer와 Memory 접근을 안전하게 처리해야 한다.

Rust도 eBPF Program을 작성하는 데 사용할 수 있으며, 관련 Framework의 지원 범위를 확인해야 한다.

Go나 Java처럼 일반적인 런타임 또는 가비지 컬렉션을 전제로 하는 언어는 커널 내 eBPF 프로그램을 작성하는 데 제약이 있다. eBPF Verifier가 허용하는 실행 모델과 맞아야 하기 때문이다.

반면 **User Space 프로그램은 커널 코드와 다른 언어로 작성할 수 있다.** 예를 들어 커널 eBPF 프로그램은 C로 작성하고, 로딩 및 데이터 처리 애플리케이션은 Go로 작성하는 구성이 가능하다.

---

## 4. BCC와 libbpf

### BCC

**BCC(BPF Compiler Collection)**는 eBPF 프로그램을 작성하고 실행하는 데 필요한 도구와 라이브러리 모음이다. 교재에서는 Python, Lua, C++ 인터페이스를 소개한다.

BCC는 eBPF 코드를 실행 시점에 컴파일하는 방식과 전처리 기능을 제공하므로 빠르게 실험하고 도구를 제작하기에 편리한다. `bpftrace` 역시 교재에서 설명하는 구조상 BCC와 LLVM/Clang을 활용한다.

다만 대상 시스템에서 컴파일에 필요한 구성 요소를 다뤄야 한다는 부담이 있다. 그래서 교재에서는 배포용 도구를 만들 때 `libbpf`와 CO-RE 방식의 장점도 설명한다.

### libbpf와 CO-RE

**libbpf**는 eBPF 프로그램과 Map을 로드하고 관리하는 데 사용하는 C 라이브러리다.

```text
C로 eBPF 프로그램 작성
        ↓
Clang/LLVM으로 eBPF 오브젝트 빌드
        ↓
User Space 로더(libbpf)
        ↓
커널에 프로그램/Map 로드 및 이벤트 연결
```


**CO-RE(Compile Once – Run Everywhere)**는 BTF 등의 정보를 사용해 컴파일 환경과 실행 환경의 커널 자료구조 차이를 조정하도록 돕는다. 앞 장에서 공부한 CO-RE와 Chapter 10의 개발 도구가 이 지점에서 연결된다.

중요한 점은 CO-RE가 **모든 커널 버전에서 모든 프로그램이 무조건 동작함을 보장하지는 않는다**는 것이다. 필요한 기능과 타입이 실제 커널에 존재하는지는 별도로 확인해야 한다.

---

## 5. Go에서 eBPF 사용하기

Go는 인프라 및 클라우드 네이티브 도구 개발에 많이 사용된다. Chapter 10에서는 세 가지 관련 선택지를 살펴본다.

교재에서는 초기 Go 기반 라이브러리인 `gobpf`, Go로 Program과 Map을 관리하는 `cilium/ebpf`, `libbpf`의 Go Binding인 `libbpfgo`를 소개한다.

### cilium/ebpf와 bpf2go

특히 `cilium/ebpf`는 이름 때문에 **Cilium CNI를 설치해야만 사용하는 라이브러리**로 오해하기 쉽지만, 별도로 활용할 수 있는 Go eBPF 라이브러리다.

교재에서 소개하는 `bpf2go`는 eBPF C 코드를 빌드하고 Go 프로그램에서 사용할 수 있도록 Go 코드를 생성하는 도구다.

```text
[eBPF C 코드]
      ↓ bpf2go
[eBPF 오브젝트 + 자동 생성된 Go 코드]
      ↓
[Go 애플리케이션]
      ↓
[프로그램 로드 / 이벤트 Attach / Map 처리]
```

이런 구성을 이용하면 컴파일된 eBPF 코드를 Go 애플리케이션과 함께 다루기 편하다. 인프라 도구를 Go로 구현하면서 커널 관찰 기능만 eBPF로 작성하는 경우를 생각해 볼 수 있다.

---

## 6. Rust에서 eBPF 사용하기

교재에서는 Rust 기반 개발 도구도 비교한다.

`libbpf-rs`는 libbpf를 Rust에서 사용할 수 있게 하는 Wrapper다. `Redbpf`는 Rust 기반 eBPF 관련 Crate 모음이고, `Aya`는 libbpf에 의존하지 않는 개발 방식을 제공한다. `rust-bcc`는 BCC와 연동하는 Rust 라이브러리다.

특히 **Aya**는 Rust로 커널 측 eBPF 코드와 사용자 공간 코드를 개발하는 흐름을 보여준다. 교재에서는 XDP 프로그램이 패킷을 허용하는 예제와, 해당 프로그램을 로드하는 User Space 코드 구조를 소개한다.

```text
Rust eBPF 프로그램 (예: XDP)
            ↓
       eBPF 바이트코드
            ↓
Rust User Space 애플리케이션
            ↓
      로드 / Attach / Map 관리
```

각 프레임워크의 실제 API와 지원 상태는 교재 집필 시점 이후 변경되었을 수 있으므로, 따라 해 볼 때는 해당 프로젝트의 현재 문서를 확인하는 편이 좋다.

---

## 7. Testing eBPF Programs

프로그램을 작성했다면 정상적으로 동작하는지 확인해야 한다. 교재에서는 **BPF 프로그램 테스트 실행**과 **실행 통계** 기능을 소개한다.

### BPF_PROG_TEST_RUN

지원되는 프로그램 유형에 대해서는 `BPF_PROG_TEST_RUN` 기능을 사용해 입력 데이터를 전달하고 결과를 확인할 수 있다. 실제 이벤트가 발생할 때까지 기다리지 않고 특정 입력을 가지고 프로그램의 동작을 테스트하는 접근 방식이다.

단, **모든 eBPF 프로그램 유형에 똑같이 적용되는 기능은 아니다.** 테스트할 프로그램 유형 및 커널 지원 여부를 확인해야 한다.

### Runtime Statistics

커널이 제공하는 BPF 실행 통계는 프로그램 실행 횟수와 실행 시간 등을 살펴보는 데 활용할 수 있다. 단순히 '프로그램이 로드됐다'는 사실뿐 아니라, **실제로 실행되고 있는지와 비용이 어느 정도인지**도 중요하다.

---

## 8. Multiple eBPF Programs

앞의 `opensnoop`만 보더라도 하나의 기능에 여러 eBPF 프로그램과 Map이 사용된다. 실무 도구는 이보다 복잡할 수 있다.

교재에서는 대표적인 예로 **Cilium**을 언급한다. Cilium은 네트워크 처리에 필요한 여러 eBPF 프로그램을 다양한 연결 지점에 배치한다.

```text
네트워크 이벤트
   ├─ eBPF 프로그램 A
   ├─ eBPF 프로그램 B
   └─ eBPF 프로그램 C
             ↓
       Map 등을 통해 상태 관리
             ↓
       네트워킹 기능 수행
```

따라서 eBPF 개발을 공부할 때는 개별 C/Rust 함수뿐 아니라 **프로그램 사이의 역할 분담, 이벤트 연결 지점, Map의 구조, User Space 제어 로직**을 함께 보아야 한다.

---

## 9. The eBPF Foundation

Chapter 11에서는 시선을 개별 프로그램 개발에서 eBPF 생태계 전체로 옮긴다.

**eBPF Foundation**은 2021년에 Google, Isovalent, Meta(당시 Facebook), Microsoft, Netflix가 Linux Foundation 산하에 설립한 조직으로 소개된다.

핵심 목적은 여러 기업이 특정 기업에 종속되지 않는 **중립적인 협력 구조**에서 eBPF 기술과 생태계에 기여할 수 있도록 하는 것이다.

교재에서는 재단이 Linux 커널 커뮤니티의 기존 개발 방식을 대체하려는 조직이 아니라고 설명한다. 기술적인 방향은 BPF Steering Committee를 통해 논의된다.

또한 eBPF 기반 프로젝트의 소속은 목적에 따라 다를 수 있다. 예를 들어 **Cilium, Pixie, Falco**처럼 클라우드 네이티브 환경을 대상으로 하는 프로젝트는 CNCF에 속한다.

---

## 10. eBPF for Windows

eBPF는 Linux 커널에서 발전했지만, Chapter 11에서는 **eBPF for Windows** 프로젝트도 소개한다.

운영체제가 다르면 커널 자료구조와 내부 기능은 다르다. 그럼에도 **네트워크 패킷 구조**나 **eBPF 바이트코드를 실행하는 가상 머신이라는 개념**은 다른 운영체제에 적용할 여지가 있다.

### eBPF for Windows Architecture

![Figure 11-1. eBPF for Windows 구조](/img/study-Learning_eBPF/ebpf-for-windows-architecture.png)

*Figure 11-1. 교재 Chapter 11의 eBPF for Windows 아키텍처. 사용자가 제공한 그림을 삽입했다.*

그림은 크게 **User Mode**와 **Kernel Mode**로 나뉜다.

그림에서 Clang은 eBPF Source를 Bytecode로 컴파일한다. `netsh.exe`와 `ebpfnetsh.dll`은 CLI 제어 경로이며, `libbpf`와 `ebpfapi.dll`은 User Mode에서 Program을 로드하고 관리하는 역할을 한다.

`ebpfsvc.dll` 서비스 내부의 **PREVAIL Verifier**는 Bytecode를 검증하고, **uBPF JIT**는 Native Code로 변환한다. `netebpfext.sys`는 Network Hook과 Helper를 제공하고, `ebpfcore.sys`는 Kernel Mode 실행 환경을 제공한다.

이 구조에서 눈여겨볼 부분은 **Verifier와 JIT 컴파일러의 위치**다.

Linux에서 배운 eBPF는 커널 내 Verifier 및 JIT와 연결되어 있었지만, 교재에 실린 Windows 아키텍처에서는 **사용자 모드의 보안 환경에서 검증과 JIT 컴파일이 이루어지는 구조**를 보여준다. 그 결과 커널 모드에서는 검증·변환된 코드를 실행한다.

그림에 `uBPF interpreter`도 보이는데, 교재에서는 이것이 **디버그 빌드용이고 운영 환경에서는 사용되지 않는다**고 설명한다.

### PREVAIL Verifier와 uBPF

교재에서는 Linux 커널이 GPL 라이선스를 사용하고 Windows는 독점 운영체제이므로, Linux 커널의 Verifier 구현을 그대로 재사용하기 어렵다고 설명한다. 대신 허용적인 라이선스의 **PREVAIL**과 **uBPF**를 활용하는 방식이 제시된다.

### 플랫폼 간 호환성

**모든 프로그램이 그대로 실행되는 것은 아니다.**

같은 eBPF 바이트코드 형식을 사용하더라도 Linux의 kprobe 대상 함수, 커널 내부 자료구조, Helper, Attach 지점이 Windows에 동일하게 존재하는 것은 아니기 때문이다.

따라서 플랫폼 간 이식성을 생각할 때는 바이트코드뿐 아니라 **프로그램이 의존하는 커널 기능과 실행 환경**을 같이 살펴봐야 한다.

**시점 주의:** 이 부분은 교재가 2022년 말 무렵 설명한 Windows 프로젝트 구조와 당시의 진행 상황을 기준으로 한다. 현재 Windows eBPF의 지원 범위나 구현 상태를 검증한 내용은 아니다.

---

## 11. Linux eBPF Evolution

Chapter 11은 당시 커널 개발자들이 논의하던 발전 방향을 소개한다. **아래 항목은 교재의 전망 및 논의 내용을 정리한 것으로, 지금 모두 같은 상태의 '미구현 기능'이라는 뜻은 아니다.**

### 프로그램의 복잡도와 Verifier

초기 eBPF는 명령어 수와 실행 흐름에 제약이 많았지만, 반복문 지원과 Helper 함수 확장 등으로 더 다양한 작업을 수행할 수 있게 되었다.

교재에서는 eBPF용 C 언어 지원과 Verifier의 발전이 이어지면, eBPF가 안전성과 동적 로딩의 장점을 유지하면서 더욱 복잡한 커널 기능을 구현하는 방향으로 나아갈 수 있다고 본다.

### Signed eBPF Programs

서명은 코드가 신뢰할 수 있는 출처에서 왔는지, 중간에 변조되지 않았는지 확인하기 위한 방법이다.

그런데 eBPF 로더는 프로그램을 커널에 로드할 때 Map 위치를 조정하거나 CO-RE 재배치를 적용할 수 있다. 즉, **정상적인 로딩 과정에서도 바이트코드가 변경될 수 있으므로**, 단순한 서명 검증 방식만으로 해결하기 어렵다.

### Long-lived Kernel Pointers

일반적으로 eBPF 프로그램이 얻은 커널 객체 포인터는 해당 프로그램 실행 중 유효한 범위에서 다뤄야 한다. 교재에서는 포인터를 Map에 저장해 나중에 재사용하는 데 제약이 있음을 설명하며, **타입이 지정된 포인터 지원**을 발전 가능성으로 소개한다.

여기서 핵심은 포인터가 가리키는 객체가 이미 해제됐는데도 접근하는 것과 같은 메모리 안전성 문제를 막는 것이다.

### Memory Allocation

일반적인 커널 코드처럼 eBPF 프로그램이 `kmalloc()`을 임의로 호출할 수 있는 것은 아니다. 교재에서는 eBPF에 적합한 별도의 메모리 할당 방식을 논의하고 있다고 소개한다.

### Big TCP

교재에서는 고성능 네트워킹의 예로 **Big TCP**를 설명한다. Linux 5.19에 도입된 기능으로, 대규모 네트워크 환경에서 패킷 처리를 효율화해 100 Gbit/s 이상의 빠른 네트워크를 지원하려는 방향의 기술이다.

이 대목에서는 **기능이 커널에 추가되는 시점과 운영 환경에서 실제 사용할 수 있는 시점이 다르다**는 사실이 중요하다. 일반 기업은 안정성을 위해 지원되는 커널 버전을 장기간 유지하기 때문이다.

새로운 eBPF 기능을 활용하고 싶어도 운영 환경의 커널 버전과 배포판 지원 여부를 함께 고려해야 한다.

### 운영 환경에서의 eBPF 활용

eBPF 프로그램을 동적으로 로드할 수 있다는 특징은 운영 환경에서 발생한 문제에 대응하는 데도 활용될 수 있다.

교재는 앞 장에서 설명한 커널 취약점 완화와 더불어, 마우스·키보드·게임 컨트롤러 같은 **HID(Human Interface Device)**를 지원하는 방향도 언급한다.

이처럼 eBPF는 단순한 네트워크 추적을 넘어 **커널 기능을 관찰하고 확장하는 범용적인 기반 기술**로 발전하고 있다.

---

## 12. eBPF Is a Platform, Not a Feature

Chapter 11에서 가장 인상적인 부분은 **“eBPF Is a Platform, Not a Feature”**라는 관점이다.

컨테이너를 사용하는 모든 사람이 Linux의 Namespace나 cgroup을 직접 구현하지는 않는다. Docker나 Kubernetes 같은 도구를 통해 해당 기능을 사용하는 경우가 대부분이다.

교재에서는 eBPF도 비슷한 방향으로 발전할 것이라고 본다. 사용자는 직접 eBPF 프로그램을 작성하지 않더라도, **네트워킹·보안·관측·성능 분석·디버깅 도구**를 통해 eBPF의 기능을 활용하게 된다는 것이다.

예를 들어 Cilium을 사용하는 입장에서는 CNI가 네트워크 정책과 패킷 처리 기능을 제공한다는 사실이 중요하다. 내부에서 몇 개의 eBPF 프로그램이 어떤 Hook에 연결되는지까지 모두 이해해야만 Cilium을 쓸 수 있는 것은 아니다.

다만 인프라나 플랫폼 엔지니어가 **문제를 진단하고 기술의 한계를 이해하려는 경우**에는 eBPF 프로그램 유형, Map, Verifier, Attach 지점에 관한 지식이 분명히 도움이 된다.

---

# Summary

이번 Chapter 10–11에서는 eBPF Program을 개발하는 방법과 앞으로의 발전 방향을 살펴봤다.

**eBPF는 커널 프로그램만 작성한다고 끝나는 기술이 아니다.** 프로그램 로딩, 이벤트 연결, Map 관리, 결과 출력처럼 User Space에서 수행할 작업도 함께 설계해야 한다.

**목적에 따라 개발 도구를 선택해야 한다.** 빠르게 시스템 이벤트를 확인하고 싶다면 `bpftrace`가 적합할 수 있고, 별도 프로그램으로 배포하거나 인프라 도구에 통합하려면 `libbpf`, `cilium/ebpf`, `Aya` 같은 라이브러리를 검토할 수 있다.

**eBPF는 특정 운영체제의 내부 구현을 완전히 감추는 마법 같은 추상화는 아니다.** Windows 지원 사례를 보면 공통 바이트코드와 실행 모델을 재사용할 수 있어도, 실제 Hook과 커널 기능은 플랫폼별로 다를 수 있다.

**기술의 발전 속도와 실무 도입 속도는 다르다.** 커널에 새 기능이 추가되더라도 운영 서버의 커널 버전과 배포판 지원 정책을 확인해야 한다.

**eBPF 자체를 직접 개발하는 사람은 일부일 수 있지만, eBPF로 만들어진 도구를 사용하는 사람은 훨씬 많아질 수 있다.** Cilium 같은 프로젝트를 공부하는 입장에서는 eBPF의 동작 원리를 이해하는 것이 문제 분석에 도움이 될 것 같다.

이로써 *Learning eBPF*의 마지막 두 장까지 정리했다. 앞으로는 프로그램 한 개의 문법만 보는 데 그치지 않고, 오픈소스에서 eBPF 프로그램과 Map, User Space 로더가 어떻게 연결되는지 함께 살펴보려고 한다.
