---
layout: post
title: "[eBPF] cgroup/connect4와 BPF Map을 활용한 네트워크 접근 제어 구현"
date: 2026-10-24 19:50:00 +0900
categories: [eBPF, Project]
tags: [eBPF, Linux, cgroup, cgroup-connect4, BPF-Map, Ring-Buffer, Go, cilium-ebpf, Network-Security, Access-Control]
published: true
---
Linux에서 특정 프로세스 그룹의 외부 TCP 연결을 기본 차단하고, 필요한 목적지에만 일정 시간 접근을 허용하는 기능을 구현해 본다. 이후 Go 기반 웹 화면에서 정책을 관리하고, eBPF 이벤트를 확인한다.

## 1. 들어가며

『Learning eBPF』를 공부하면서 eBPF 프로그램이 커널의 여러 Hook에 부착되어 실행된다는 개념을 배웠다. 하지만 교재에서 프로그램 유형이나 Map을 각각 살펴보는 것과, 실제로 이 기능들을 연결해서 하나의 시스템을 만드는 것은 다르게 느껴졌다.

그래서 이번에는 **특정 워크로드가 외부 서버에 접근할 수 있는 시간을 제한하는 기능**을 직접 구현해 보기로 했다.

예를 들어 애플리케이션이 외부 API에 잠깐 접근해야 할 때, 관리자에게 5분 동안만 권한을 부여받는 상황을 생각했다. 권한이 없을 때는 연결을 거부하고, 허용 시간이 끝나면 다시 차단하도록 하는 것이다.

처음에는 목적지 IP와 포트를 C 코드에 직접 적어 차단하는 작은 실습으로 시작했다. 이후 BPF Map, TTL, Go, 웹 화면, Ring Buffer를 순서대로 추가했고, 마지막에는 **서로 다른 워크로드에 다른 접근 정책이 적용되는지** 확인했다.

## 2. 무엇을 구현하려고 했을까?

필요한 기능을 네 가지로 정리했다.

1. **기본 차단:** 보호 대상 워크로드가 등록되지 않은 목적지로 TCP 연결을 시도하면 거부한다.
2. **임시 허용:** 관리자가 목적지 IP, 포트, TTL을 지정하면 해당 시간 동안 새 연결을 허용한다.
3. **워크로드별 격리:** A 워크로드에 부여한 권한이 B 워크로드에는 적용되지 않아야 한다.
4. **이벤트 확인:** eBPF가 어떤 연결을 허용하거나 거부했는지 관리 화면에서 확인한다.

이번 실습에서 TTL은 **이미 맺어진 TCP 연결을 강제로 종료하는 시간**이 아니다. 만료 이후 **새롭게 시도하는 연결을 거부하는 기준**이다.

## 3. 개발 환경

Apple Silicon Mac에서 UTM으로 Ubuntu Server VM을 만들고 실습했다.

| 구분 | 환경 |
|---|---|
| 가상화 | UTM (Apple Silicon, ARM64) |
| OS | Ubuntu Server 24.04.5 LTS |
| Linux Kernel | `6.8.0-147-generic` |
| CPU 아키텍처 | `aarch64` |
| cgroup | cgroup v2 (`cgroup2fs`) |
| BTF | `/sys/kernel/btf/vmlinux` 확인 |
| 컴파일러 | Clang 18.1.3 |
| Go | 초기 설치 Go 1.22.2 (이후 의존성 해석 과정에서 자동 툴체인 다운로드 발생) |
| eBPF 도구 | bpftool 7.4.0, libbpf |
| Go 라이브러리 | `github.com/cilium/ebpf` |

먼저 eBPF 기능을 확인했다.

```bash
stat -fc %T /sys/fs/cgroup
ls -lh /sys/kernel/btf/vmlinux
sudo bpftool feature probe kernel | grep 'program_type cgroup_sock_addr is available'
```

결과에서 `cgroup2fs`와 BTF 파일을 확인했고, 다음 메시지도 확인했다.

```text
eBPF program_type cgroup_sock_addr is available
```

이것은 사용할 **프로그램 유형을 커널이 지원한다**는 뜻이다. 실제 로드·부착·동작 여부는 별도로 검증해야 한다.

## 4. 전체 구조 이해하기

<!-- 캡처 추가 권장: Overview 전체 화면 / 파일명 ebpf-dashboard-overview.png / 현재 패키지 미포함 -->
<!-- 직접 촬영 후 아래 주석을 풀어 사용:
![네트워크 접근 제어 대시보드 전체 화면](/img/study-Learning_eBPF/ebpf-dashboard-overview.png)
-->



처음에는 웹 화면에서 연결을 차단한다고 생각하기 쉬웠다. 하지만 실제로는 **정책을 관리하는 부분과 정책을 집행하는 부분이 분리**되어 있다.

```text
[관리자 / Web Dashboard]
            |
            | 정책 등록·조회·철회 (HTTP API)
            v
       [Go Backend]  ---------> [BPF Hash Map]
                                       ^
                                       | 정책 조회
[워크로드 프로세스]                       |
            |                          |
            | connect()                |
            v                          |
      [Linux Kernel]                    |
            |                          |
            v                          |
 [eBPF: cgroup/connect4] ---------------+
            |
            +---- return 1 : 허용
            |
            +---- return 0 : 거부
            |
            +---- Ring Buffer 이벤트 ----> [Go] ----> [Events 화면]
```

두 가지 흐름을 나눠서 보면 이해하기 쉽다.

- **정책 관리:** 브라우저에서 Grant 버튼을 누르면 Go가 BPF Map에 `목적지 IP + 포트 → 만료 시간`을 등록한다.
- **연결 제어:** 워크로드에서 `connect()`를 호출하면 eBPF 프로그램이 Map을 조회해 연결을 허용하거나 거부한다.

웹 화면을 닫는다고 커널의 연결 판정 로직이 브라우저에서 실행되는 것은 아니다. **실제 접근 제어는 Linux Kernel의 eBPF 프로그램이 수행한다.**

## 5. 핵심 eBPF 코드 살펴보기

### 5.1 `cgroup/connect4`를 선택한 이유

이번 목적은 모든 패킷을 분석하는 것이 아니라, **특정 프로세스 그룹이 IPv4 소켓 연결을 시도하는 시점**에 허용 여부를 결정하는 것이다.

그래서 다음 Hook을 사용했다.

```c
SEC("cgroup/connect4")
int egress_connect4(struct bpf_sock_addr *ctx)
```

- `cgroup`: 해당 cgroup 및 적용되는 하위 계층의 프로세스에 정책을 적용한다.
- `connect4`: IPv4 소켓의 `connect()` 시점에 프로그램을 실행한다.
- `ctx`: 연결하려는 목적지 IP, 포트, 프로토콜 등의 정보를 담는다.

이 방식은 XDP나 TC처럼 개별 패킷을 검사하는 방식과 다르다. **새 연결 요청을 검사하는 방식**이므로 이미 성립한 연결을 직접 종료하지 않는다.

### 5.2 BPF Map에 접근 정책 저장하기

목적지를 코드에 하드코딩하면 정책을 바꿀 때마다 다시 컴파일해야 한다. 그래서 Hash Map을 사용했다.

```c
struct policy_key {
    __u32 ip;
    __u32 port;
};

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, struct policy_key);
    __type(value, __u64);
} policies SEC(".maps");
```

Map의 Key는 **목적지 IPv4와 TCP 포트**, Value는 **만료 시간(`__u64`)**이다.

예를 들어 다음과 같은 정보를 관리할 수 있다.

| Key | Value의 의미 |
|---|---|
| `1.1.1.1:443` | 이 목적지에 대한 접근 허용 만료 시각 |
| `8.8.8.8:443` | 별도 목적지에 대한 접근 허용 만료 시각 |

여기서 만료 시간은 일반적인 날짜 문자열이 아니라 **커널의 monotonic clock 기준 나노초 값**으로 저장했다. Go에서도 `CLOCK_MONOTONIC`을 사용해 같은 시간 기준으로 만료 값을 계산하도록 했다.

### 5.3 정책 조회와 TTL 검사

아래 화면에서는 `1.1.1.1:443`에 임시 접근 권한을 부여한 뒤 남은 시간이 표시되는 것을 확인할 수 있다.

![접근 정책이 활성화된 상태와 연결 성공 결과](/img/study-Learning_eBPF/ebpf-policy-active.png)

*그림 1. 허용 정책이 등록된 상태에서 TCP 연결이 성공한 결과. (초기 단일 워크로드 실습 화면)*



실제 소스에서 중요한 부분은 다음과 같다.

```c
struct policy_key key = {
    .ip = bpf_ntohl(ctx->user_ip4),
    .port = bpf_ntohs(ctx->user_port),
};

__u64 *expires_at = bpf_map_lookup_elem(&policies, &key);
__u64 now = bpf_ktime_get_ns();

__u32 verdict = 0;
__u32 reason = 1; // NO_POLICY

if (expires_at) {
    if (now < *expires_at) {
        verdict = 1;
        reason = 0; // VALID_POLICY
    } else {
        reason = 2; // EXPIRED
    }
}
```

동작 순서는 간단하다.

1. 연결하려는 IP와 포트를 추출한다.
2. 해당 Key로 BPF Map을 조회한다.
3. 정책이 없으면 거부한다.
4. 정책이 있어도 만료 시간이 지났다면 거부한다.
5. 정책이 유효하면 허용한다.

`bpf_ntohl()`과 `bpf_ntohs()`는 네트워크 바이트 순서 값을 비교와 Map 조회에 사용하기 편한 형태로 변환하기 위해 사용했다.

최종적으로 `return verdict;`를 호출한다. **이 Hook에서 `1`은 허용, `0`은 거부**를 의미한다.

### 5.4 Ring Buffer로 이벤트 전달하기

연결을 차단하는 것만으로는 관리자가 어떤 일이 발생했는지 알기 어렵다. 그래서 eBPF에서 이벤트를 Ring Buffer에 기록하도록 했다.

```c
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 1 << 18);
} events SEC(".maps");
```

연결을 판정한 뒤 다음과 같이 이벤트를 생성한다.

```c
struct connect_event *event = bpf_ringbuf_reserve(
    &events, sizeof(*event), 0
);

if (event) {
    event->timestamp_ns = now;
    event->ip = key.ip;
    event->port = key.port;
    event->pid = (__u32)(bpf_get_current_pid_tgid() >> 32);
    event->verdict = verdict;
    event->reason = reason;
    event->reserved = 0;
    bpf_ringbuf_submit(event, 0);
}
```

Go 프로그램은 Ring Buffer의 이벤트를 읽어 웹 화면에 전달한다. 현재 코드에서는 이벤트 버퍼에 공간이 없으면 기록을 건너뛸 수 있으므로, **모든 연결 시도가 빠짐없이 기록된다고 보장하는 구조는 아니다.** 또한 여기서 `ALLOWED`는 **eBPF가 연결을 허용했다는 판정**이지 상대 서버와 TCP 연결이 최종 성공했다는 뜻은 아니다.


<!-- 캡처 추가 권장: Kernel Events 목록에서 NO_POLICY / VALID_POLICY 이벤트가 동시에 보이는 화면 -->
<!-- 파일명 ebpf-kernel-events.png / 현재 패키지 미포함 -->
<!-- 직접 촬영 후 아래 주석을 풀어 사용:
![Ring Buffer를 통해 수집한 커널 연결 판정 이벤트](/img/study-Learning_eBPF/ebpf-kernel-events.png)
-->

### 5.5 처음에는 IP를 코드에 직접 넣어 차단했다

처음부터 Map과 웹 화면을 만든 것은 아니다. 가장 먼저는 **연결 차단이 정말 가능한지**만 확인하기 위해 목적지 하나를 하드코딩했다.

```c
SEC("cgroup/connect4")
int egress_connect4(struct bpf_sock_addr *ctx)
{
    if (ctx->protocol != IPPROTO_TCP)
        return 1;

    if (ctx->user_ip4 == bpf_htonl(0x01010101) &&
        ctx->user_port == bpf_htons(443))
        return 0;

    return 1;
}
```

`1.1.1.1:443`에 대해서만 `return 0`을 주는 예제다. 이 프로그램을 컴파일하고 테스트 cgroup에 붙인 뒤 실제 Python 소켓 연결에서 `PermissionError`를 확인했다. 이후 **코드 수정 없이 정책을 바꾸기 위해** 앞서 설명한 Hash Map 기반 구조로 변경했다.

### 5.6 eBPF C 파일을 컴파일하고 커널에 부착하는 과정

실습에서 사용한 컴파일 명령어는 다음과 같다.

```bash
clang -O2 -g -target bpf \
  -D__TARGET_ARCH_arm64 \
  -I/usr/include/aarch64-linux-gnu \
  -c bpf/egress.c \
  -o bpf/egress.o

file bpf/egress.o
llvm-objdump -h bpf/egress.o
```

`-target bpf`는 ARM64에서 실행하는 일반 바이너리가 아니라 **커널에 로드할 eBPF 바이트코드**를 생성하기 위한 옵션이다. 실습 결과는 다음과 같았다.

```text
ELF 64-bit LSB relocatable, eBPF

Sections:
  cgroup/connect4
  .maps
  .BTF
  .BTF.ext
```

- `cgroup/connect4`: 연결 요청을 검사하는 eBPF 프로그램 섹션
- `.maps`: 정책 Hash Map, 이벤트 Ring Buffer 정의
- `.BTF`: 타입 정보

초기 기능 검증에서는 `bpftool`을 이용해 직접 로드하고 부착했다.

```bash
sudo bpftool prog load bpf/egress.o /sys/fs/bpf/connect4_demo
sudo bpftool cgroup attach \
  /sys/fs/cgroup/<테스트-cgroup-경로> \
  connect4 pinned /sys/fs/bpf/connect4_demo
```

위 명령의 `<테스트-cgroup-경로>`는 설명용 자리표시자다. 실제로는 존재하는 cgroup 디렉터리 경로를 지정해야 한다. **컴파일 → 커널 로드(Verifier 검사) → Hook 부착**까지 완료해야 연결 요청이 발생할 때 프로그램이 실행된다. 이후에는 이 과정을 Go의 `cilium/ebpf` 라이브러리로 자동화했다.

## 6. 워크로드별로 다른 정책 적용하기

처음에는 하나의 테스트 cgroup에만 eBPF 프로그램을 붙였다. 이후 **서로 다른 cgroup을 사용하는 두 테스트 워크로드**로 확장했다.

```text
Ubuntu VM
├── egressleasea.slice
│   └── Workload A
│       └── 별도 BPF Map
│
├── egressleaseb.slice
│   └── Workload B
│       └── 별도 BPF Map
│
└── 일반 시스템 프로세스
    └── 위 테스트 cgroup의 정책 적용 대상 아님
```

이름 자체보다는 **서로 다른 cgroup에 서로 다른 eBPF 프로그램 인스턴스와 Map이 부착되어 있다는 것**이 중요하다.

Go에서는 `cilium/ebpf` 라이브러리로 eBPF 오브젝트를 로드하고 cgroup에 부착했다.

```go
link.AttachCgroup(link.CgroupOptions{
    Path:    "/sys/fs/cgroup/" + config.slice,
    Attach:  ebpf.AttachCGroupInet4Connect,
    Program: coll.Programs["egress_connect4"],
})
```

실제로 로그에서 다음 상태를 확인했다.

```text
attached a -> egressleasea.slice
attached b -> egressleaseb.slice
```

이를 통해 A에 권한을 부여해도 B에는 동일한 권한이 적용되지 않도록 구성했다.


### 6.1 Go가 eBPF 프로그램을 로드하고 cgroup에 붙이는 방법

다음은 실제 Go 소스의 핵심 부분을 간추린 것이다.

```go
spec, err := ebpf.LoadCollectionSpec("bpf/egress.o")
if err != nil {
    log.Fatal(err)
}

coll, err := ebpf.NewCollection(spec)
if err != nil {
    log.Fatal(err)
}

policies := coll.Maps["policies"]
prog := coll.Programs["egress_connect4"]

attachment, err := link.AttachCgroup(link.CgroupOptions{
    Path:    "/sys/fs/cgroup/" + config.slice,
    Attach:  ebpf.AttachCGroupInet4Connect,
    Program: prog,
})
if err != nil {
    log.Fatal(err)
}
```

- `LoadCollectionSpec()`: 컴파일된 ELF 오브젝트의 프로그램·Map 정의를 읽는다.
- `NewCollection()`: 프로그램과 Map을 커널에 로드한다.
- `coll.Maps["policies"]`: 해당 워크로드에서 사용할 정책 Map의 핸들을 가져온다.
- `AttachCgroup()`: 지정한 cgroup에 IPv4 `connect()` 프로그램을 연결한다.

실제 구현에서는 **A와 B 각각에 대해 `NewCollection()`을 별도로 호출**했다. 그래서 두 워크로드는 이름이 같은 `policies` Map을 사용하더라도 **서로 다른 커널 Map 인스턴스**를 갖는다. 이것이 정책 격리의 핵심이다.

### 6.2 Go에서 IP·포트·TTL을 정책 Map에 등록하기

브라우저에서 정책을 생성하면 Go API가 요청을 받아 Key를 만들고, 만료 시각을 Value로 저장한다.

```go
// 실제 소스의 정책 등록 부분 발췌
key, err := getKey(req.IP, req.Port)
if err != nil {
    // 잘못된 IPv4/포트 입력 처리
    return
}

now, err := monotonicNS()
if err != nil {
    return
}

expiry := now + uint64(req.TTL)*1e9
if err := wl.Policies.Put(key, expiry); err != nil {
    // Map 업데이트 실패 처리
    return
}
```

예를 들어 TTL이 300초라면 **현재 monotonic time + 300초**를 Map에 저장한다. 커널의 eBPF 프로그램은 새 연결 요청 시 `bpf_ktime_get_ns()`와 이 값을 비교한다. 두 시간 기준을 맞추는 것이 중요하다.

```text
POST /api/policies
{ "workload": "a", "ip": "1.1.1.1", "port": 443, "ttl": 300 }
                |
                v
Go: monotonic 시간 계산 → policies.Put(key, expiry)
                |
                v
Kernel: BPF Hash Map 저장
                |
                v
새 connect() 호출 시 eBPF가 조회 → 허용 또는 차단
```

반대로 권한 철회는 `wl.Policies.Delete(key)`를 호출한다. TTL 연장은 기존 Map 값을 읽고 새로운 만료 시각으로 `Put()`한다. **정책 변경에 C 코드 재컴파일이 필요하지 않다.**

### 6.3 Ring Buffer 이벤트를 Go에서 수신하기

C 프로그램이 이벤트를 `bpf_ringbuf_submit()`으로 제출하면 Go는 별도 고루틴에서 이를 읽는다.

```go
reader, err := ringbuf.NewReader(coll.Maps["events"])
if err != nil {
    log.Fatal(err)
}

for {
    record, err := reader.Read()
    if err != nil {
        // 종료 또는 읽기 오류 처리
        break
    }
    // record.RawSample을 C의 connect_event 구조체에 맞춰 디코딩
    _ = record
}
```

실제 소스에는 `binary.Read(..., binary.LittleEndian, &raw)`로 C 구조체 필드를 해석하고, PID·목적지·판정·사유를 메모리 이벤트 목록에 저장하는 코드가 들어 있다. 웹의 `/api/events` API는 그 목록을 조회한다.

다만 **Kernel Events는 `connect4` Hook에서 발생한 정책 판정**이다. 예를 들어 `ALLOWED` 이벤트가 남아도 이후 TCP 핸드셰이크가 반드시 성공한 것은 아니다. 그 성공 여부는 별도의 Python 연결 테스트 결과인 `CONNECTED`로 확인했다.

### 6.4 HTTP API와 웹 화면은 어떻게 연결했을까?

Go의 `net/http` 패키지로 API를 만들고, 프런트엔드는 JavaScript의 `fetch()`로 호출하도록 구성했다. 실제 소스에 정의된 주요 엔드포인트는 다음과 같다.

| API | 용도 |
|---|---|
| `GET /api/workloads` | 테스트 워크로드 목록 |
| `GET /api/policies` | 현재 Map에 등록된 정책 조회 |
| `POST /api/policies` | IP·포트·TTL 정책 등록 |
| `POST /api/revoke` | 접근 권한 삭제 |
| `POST /api/extend` | 기존 TTL 60초 연장 |
| `POST /api/test` | 선택한 워크로드에서 실제 TCP 연결 시도 |
| `GET /api/events` | 커널 Ring Buffer로 수집한 이벤트 조회 |
| `GET /api/audit` | 관리 API의 작업 기록 조회(메모리) |
| `GET /api/status` | 서버가 제공하는 기본 상태 정보 |

API 예시 요청은 다음과 같다. 서버는 실습용으로 VM의 `127.0.0.1:8080`에서만 접근 가능하게 구성했다.

```bash
curl -X POST http://127.0.0.1:8080/api/policies \
  -H 'Content-Type: application/json' \
  -d '{"workload":"a","ip":"1.1.1.1","port":443,"ttl":300}'

curl http://127.0.0.1:8080/api/policies
```

위 요청은 **API 사용법을 설명하는 예시**이며, 블로그에 실린 `curl` 호출 자체를 별도로 실행·검증한 기록은 아니다. 웹 화면에서 정책을 등록하고 실제 연결 성공을 확인한 것은 검증된 결과다.

## 7. 실제 테스트 결과

### 7.1 권한이 없으면 연결 차단

먼저 테스트 cgroup에서 `1.1.1.1:443`으로 연결을 시도했다.

```bash
sudo systemd-run --wait --collect --pipe \
  --unit=egress-test-deny \
  --slice=egresslease.slice \
  /usr/bin/python3 -c 'import socket; socket.create_connection(("1.1.1.1",443),3)'
```

초기 단일 cgroup 실습에서 다음 결과가 발생했다.

```text
PermissionError: [Errno 1] Operation not permitted
```

일반 Ubuntu 셸에서는 같은 목적지 연결이 성공했고, eBPF 프로그램이 부착된 테스트 cgroup에서는 거부됐다. 따라서 해당 프로세스 그룹에 접근 제어가 적용된 것을 확인할 수 있었다.

### 7.2 임시 접근 권한을 부여하면 연결 허용

웹 화면에서 `1.1.1.1`, TCP `443`, TTL `300초` 정책을 등록한 뒤 같은 연결을 다시 시도했다.

```text
CONNECTED
Finished with result: success
Main processes terminated with: code=exited/status=0
```

C 소스코드를 다시 컴파일하지 않아도 BPF Map의 내용만 변경해 연결 판정이 달라졌다.

### 7.3 TTL 만료 후 다시 차단

60초짜리 접근 권한을 등록한 실습에서는 만료 전에는 `CONNECTED`가 표시됐고, 만료 후 새 연결 시도에서는 다음 오류가 나타났다.

```text
PermissionError: [Errno 1] Operation not permitted
```

즉, **정책 등록 → 임시 허용 → TTL 만료 후 새 연결 차단** 흐름을 확인했다.

![TTL 만료 후 정책이 만료됨으로 표시되고 연결이 거부된 화면](/img/study-Learning_eBPF/ebpf-ttl-expired.png)

*그림 2. TTL 만료 후 새 연결에서 `PermissionError`가 발생한 결과. (초기 단일 워크로드 실습 화면)*

### 7.4 워크로드 A/B 정책 격리

마지막으로 웹 화면에서 **Workload A에만** `1.1.1.1:443` 접근 권한을 등록했다. 동일 목적지에 대한 연결 테스트 결과는 다음과 같았다.

| 테스트 대상 | 정책 | 실제 확인 결과 |
|---|---|---|
| Workload A | `1.1.1.1:443` 허용 | `CONNECTED` |
| Workload B | 허용 정책 없음 | `POLICY_DENIED`, `Errno 1` |

이 실험이 의미 있었던 이유는 IP와 포트가 같더라도 **프로세스가 속한 cgroup에 따라 연결 판정이 달라졌기 때문**이다.

**Workload A — 허용 정책이 있는 경우**

![Workload A의 접근 정책 활성 상태와 TCP 연결 성공 결과](/img/study-Learning_eBPF/ebpf-workload-a-allowed.png)

*그림 3. Workload A에는 `1.1.1.1:443` 허용 정책이 등록되어 `CONNECTED`가 출력됐다.*

**Workload B — 허용 정책이 없는 경우**

![Workload B에서 동일 목적지에 대한 연결이 거부된 결과](/img/study-Learning_eBPF/ebpf-workload-b-denied.png)

*그림 4. Workload B에는 해당 정책이 없어서 `POLICY_DENIED`가 발생했다.*


### 7.5 테스트 프로세스를 특정 cgroup에서 실행한 이유

일반 셸에서 Python을 실행하면 실습용 cgroup 정책의 영향을 받지 않는다. 그래서 `systemd-run`의 `--slice` 옵션으로 **실제로 정책을 적용할 프로세스 그룹**을 지정했다.

```bash
sudo systemd-run --wait --collect --pipe \
  --unit=connect-test-a \
  --slice=egressleasea.slice \
  /usr/bin/python3 -c 'import socket; s=socket.create_connection(("1.1.1.1",443),3); print("CONNECTED"); s.close()'
```

위의 Slice 이름은 실습에서 실제 사용한 cgroup 식별자다. B에 대해 테스트할 때는 `--slice=egressleaseb.slice`로 변경했다. 두 테스트의 목적지와 코드는 동일하지만, **cgroup이 다르므로 참조하는 정책 Map도 달랐다.**

검증 결과는 A에서 `CONNECTED`, B에서 `POLICY_DENIED`로 달라졌다. 웹 화면에서 단순한 시뮬레이션을 한 것이 아니라 Linux의 별도 프로세스가 실제로 `connect()`를 호출한 결과다.

## 8. 개발하면서 겪은 문제

### 8.1 eBPF 코드를 컴파일했는데 실제 차단이 안 된다면?

처음에는 `clang -target bpf`로 `.o` 파일을 생성하는 것부터 진행했다. 그런데 컴파일이 성공했다고 곧바로 정책이 적용되는 것은 아니었다.

실제로는 다음 단계가 필요했다.

```text
C 코드 작성 → Clang 컴파일 → 커널 로드(Verifier 검사) → cgroup Hook 부착 → 연결 시도 시 실행
```

`bpftool`로 로드하고, 해당 cgroup에 프로그램을 부착한 뒤에야 Python 연결이 차단됐다.

### 8.2 systemd Slice 이름과 cgroup 경로

다중 워크로드 기능을 만들면서 다음 오류를 만났다.

```text
can't open cgroup: ... no such file or directory
```

원인은 systemd의 Slice 이름에 있는 하이픈(`-`)이 계층 구조로 해석될 수 있다는 점이었다. 프로그램에서는 최상위 경로를 예상했지만, 실제 Slice의 경로가 달라졌다.

테스트 환경에서는 Slice 이름을 `egressleasea.slice`, `egressleaseb.slice`처럼 변경해 경로를 일치시켰다.

이 문제를 겪으면서 **eBPF 로드 성공과 올바른 cgroup 부착은 별도의 문제**라는 것을 알게 됐다.

### 8.3 웹 화면과 실제 커널 동작은 구분해야 한다

초기 웹 화면은 입력한 IP·포트와 관계없이 연결 테스트 대상이 고정되어 있었다. 이후 입력한 목적지로 실제 연결을 시도하도록 수정했다.

또한 화면에서 보여주는 테스트 결과와 eBPF가 Ring Buffer로 전달한 이벤트는 서로 다른 정보다. 웹 테스트의 TCP 연결 성공 여부와 커널의 `ALLOWED/DENIED` 판정을 구분해서 해석해야 한다.

## 9. 현재 한계와 개선할 부분

이번 프로젝트는 동작하는 프로토타입이지만 운영 환경에서 사용하려면 해결해야 할 점이 남아 있다.

| 항목 | 현재 범위 / 한계 | 개선 방향 |
|---|---|---|
| 적용 대상 | 지정된 테스트 cgroup | 실제 서비스·컨테이너 워크로드 등록 및 생명주기 관리 |
| 네트워크 범위 | IPv4 TCP 연결 | UDP, IPv6 등 별도 설계 |
| TTL | 만료 후 새 연결 차단 | 기존 연결에 대한 처리 정책 별도 설계 |
| 정책 데이터 | 서버 실행 중 커널 Map 및 메모리 중심 | 정책 영속화·재시작 복구 |
| 이벤트 | 메모리 기반 수집, 유실 가능 | 장기 저장·유실 지표·검색 |
| 관리자 보안 | 로컬 실습용, 인증 미구현 | 인증·RBAC·감사 로그·최소 권한 |
| Kubernetes | 직접 연계하지 않음 | Pod/cgroup 식별 및 연동 검토 |

정책 관리 화면 개선 작업도 진행했지만, **이번 글의 검증 결과는 실제로 실행해 확인한 기능까지만** 작성했다.

## 10. 구현 결과 요약과 마무리

이번 실습 전에는 eBPF 프로그램, Map, Hook이 각각 어떤 역할을 하는지 공부해도 전체 흐름이 잘 연결되지 않았다.

직접 구현하면서 가장 크게 이해한 점은 다음과 같다.

- **Hook:** eBPF 프로그램이 언제 실행되는지를 결정한다.
- **cgroup:** 정책을 어떤 프로세스 그룹에 적용할지 결정한다.
- **BPF Map:** User Space에서 관리하는 접근 정책을 커널의 eBPF 프로그램이 조회할 수 있게 한다.
- **Go와 `cilium/ebpf`:** 커널 프로그램을 로드·부착하고 정책을 수정하는 관리 계층을 구현한다.
- **Ring Buffer:** 커널의 연결 판정 결과를 User Space에 전달하는 통로가 된다.

무엇보다 **같은 목적지에 접근하더라도 워크로드 A는 허용되고 B는 거부되는 결과**를 직접 확인하면서 cgroup에 eBPF를 부착하는 방식이 어떤 의미인지 이해할 수 있었다.

아직은 실제 운영 환경에 적용할 수 있는 완성된 플랫폼이 아니라, 학습한 eBPF 개념들을 하나의 흐름으로 연결해 본 실습이다. 앞으로 정책 영속화와 인증, 워크로드 관리 기능을 보완하면서 확장해 보고 싶다.
