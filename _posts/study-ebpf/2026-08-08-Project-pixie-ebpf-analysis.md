---
layout: post
title: "[eBPF Open Source] Pixie의 Stirling Socket Tracer 분석"
date: 2026-08-08 19:25:00 +0900
categories: [eBPF, Pixie, Observability]
tags: [eBPF, Pixie, Stirling, BCC, kprobe, BPF Map, Perf Buffer, Kubernetes]
published: true
---

# Pixie의 eBPF 활용 분석

Learning eBPF Chapter 1~4에서는 eBPF Program이 Kernel Event에 연결되어 실행되는 구조, BPF Map을 통한 상태 공유, BCC를 이용한 Program Load/Attach, Perf Buffer를 통한 Kernel과 User Space 사이의 Event 전달 등을 학습했다.

이번에는 책의 작은 예제를 넘어 실제 오픈소스에서는 이런 기능을 어떻게 조합해서 사용하는지 확인하기 위해 Pixie의 소스 코드를 분석했다. Pixie 전체를 분석하는 것이 아니라, **Node에서 Network Event를 수집하는 Stirling의 Socket Tracer를 중심으로 eBPF가 실제로 어떻게 사용되는지**를 따라가 보는 것을 목표로 했다.

---

## 1. Pixie란?

Pixie는 Kubernetes 환경에서 Application과 Infrastructure의 동작을 관찰하기 위한 Observability Open Source이다.

Pixie 내부에서 실제 Node의 데이터를 수집하는 구성요소 중 하나가 **Stirling**이다. Stirling README에서는 Stirling을 각 Node에서 실행되는 data collector로 설명하며, Linux API와 eBPF를 이용해 Linux Kernel, System Library, Application에서 데이터를 수집한다고 설명한다.

수집 대상에는 CPU, Memory, Network 관련 정보뿐 아니라 HTTP, MySQL, PostgreSQL 등 Application Network Message도 포함된다.

eBPF 관점에서 흥미로운 점은 Application에 별도의 계측 코드를 직접 추가하는 방식만을 사용하는 것이 아니라, Kernel의 socket 관련 동작을 추적해 Application의 Network Activity를 관찰한다는 점이다.

이번 분석에서는 그중 **Socket Tracer가 Network 관련 System Call을 어떻게 추적하고, Connection State와 Socket Payload를 어떻게 처리하여 User Space까지 전달하는지**를 확인했다.

---

## 2. 왜 Pixie를 분석했는가?

Chapter 1~4에서 본 예제들은 eBPF의 핵심 동작을 이해하기에는 좋았지만 비교적 단순했다.

예를 들어 하나의 Kernel Event에 eBPF Program을 Attach하고 PID, UID 등의 정보를 읽은 뒤 BPF Map이나 Perf Buffer를 이용해 User Space와 데이터를 주고받는 형태였다.

그래서 실제 Observability Tool에서는 다음 기능들이 어떻게 연결되는지가 궁금했다.

- 여러 Kernel Event를 동시에 어떻게 추적하는가?
- System Call의 Entry와 Return 사이의 정보는 어떻게 연결하는가?
- BPF Map은 실제 프로그램에서 어떤 상태를 저장하는가?
- Socket에서 얻은 Raw Data를 어떻게 Application Protocol과 연결하는가?
- Kernel에서 수집한 Event를 User Space로 어떻게 전달하는가?
- Verifier, Stack, Instruction Limit 같은 eBPF 제약은 실제 코드에 어떤 영향을 주는가?

Pixie의 Stirling Socket Tracer에서는 `connect`, `accept`, `send`, `recv`, `read`, `write` 등 Network 관련 동작을 추적하고, BPF Map으로 Connection과 Probe 사이의 상태를 유지하며, Perf Buffer를 이용해 Event를 User Space로 전달한다.

또한 Socket Payload를 기반으로 HTTP, MySQL, PostgreSQL, DNS, Kafka, Redis 등 Application Protocol을 추론하는 로직도 포함하고 있다.

따라서 Chapter 1~4에서 학습한 eBPF의 기본 요소들이 실제 규모가 큰 Observability Open Source에서 어떻게 조합되는지를 확인하기에 적합하다고 판단했다.

---

## 3. Pixie 저장소에서 왜 `src/stirling`을 봤는가?

Pixie 저장소는 eBPF 코드만 있는 작은 프로젝트가 아니다. 여러 Component와 Library, Build/Test 관련 파일이 함께 존재하기 때문에 저장소 전체를 순서대로 읽는 방식으로는 eBPF의 사용 지점을 찾기 어렵다.

이번 분석의 목적은 **Pixie 전체 Architecture 분석이 아니라 eBPF 활용 방식 분석**이므로, 먼저 실제 Node Data Collection을 담당하는 Stirling으로 범위를 좁혔다.

이번 글에서 직접 확인한 주요 파일은 다음과 같다.

```text
src/stirling/
└── source_connectors/
    └── socket_tracer/
        ├── socket_trace_connector.cc
        │   └── BCC 초기화 / Probe Attach / Perf Buffer User Space 처리
        │
        ├── bcc_bpf/
        │   ├── socket_trace.c
        │   │   └── Kernel에서 실행되는 Socket Tracing eBPF Program
        │   │
        │   └── protocol_inference.h
        │       └── Application Protocol Inference
        │
        └── bcc_bpf_intf/
            ├── socket_trace.h
            │   └── Connection State / Event Structure
            │
            └── common.h
                └── Protocol / Direction / Message / Connection 공통 Type
```
분석 범위는 Pixie 저장소의 `src/stirling/source_connectors/socket_tracer`를 중심으로 하며, Pixie 전체 Architecture나 Pixie의 User Space Protocol Parser, ConnTracker, PxL, Vizier 등 이후 Pipeline은 이번 eBPF 분석 범위에서는 제외했다. 해당 영역까지 확장하면 eBPF 자체보다 Pixie 전체 Observability Architecture 분석에 가까워지기 때문이다.

Stirling은 여러 Source Connector를 통해 서로 다른 Data Source의 정보를 수집한다. 그중 `socket_tracer`는 Socket 기반 Network Activity를 관찰하는 부분이기 때문에 이번 eBPF 분석 대상으로 선택했다.


---

### 분석한 파일과 역할

| 파일 | 분석한 이유 |
|---|---|
| `README.md` | Stirling이 어떤 Component이며 eBPF/BCC를 어디에 사용하는지 전체 구조 파악 |
| `socket_trace_connector.cc` | User Space에서 BPF Program을 초기화하고 Probe를 Attach하며 Perf Buffer를 여는 과정 확인 |
| `bcc_bpf/socket_trace.c` | 실제 Kernel에서 실행되는 eBPF Program의 핵심 로직 분석 |
| `bcc_bpf/protocol_inference.h` | Socket Payload를 이용해 Application Protocol을 추론하는 과정 확인 |
| `bcc_bpf_intf/socket_trace.h` | Connection State와 Kernel→User Space Event 구조 확인 |
| `bcc_bpf_intf/common.h` | Protocol, Traffic Direction, Message Type, Connection ID 등 공통 Type 확인 |

즉 파일 이름을 순서대로 읽은 것이 아니라 다음 실행 흐름을 따라가기 위해 필요한 파일을 선택했다.

```text
User Space에서 BPF Program 준비
        ↓
Kernel Hook에 Attach
        ↓
System Call 발생
        ↓
eBPF Program 실행
        ↓
BPF Map으로 State 관리
        ↓
Protocol 추론
        ↓
Event 생성
        ↓
Perf Buffer
        ↓
Stirling User Space
```

이 흐름을 따라가면 Chapter 1~4에서 각각 따로 학습했던 eBPF 개념이 실제 프로그램에서는 어떻게 하나의 Pipeline으로 연결되는지 확인할 수 있다.

---

## 4. Stirling과 Socket Tracer

Stirling은 Node에서 여러 종류의 데이터를 수집하는 Data Collector이며, Source Connector가 각각 특정 Data Source를 담당하는 구조를 사용한다.

이번에 분석한 `socket_tracer`는 Application의 Network Communication을 관찰하기 위한 Source Connector이다.

전체적인 흐름을 단순화하면 다음과 같다.

```text
Application
    │
    │ connect / accept
    │ send / recv
    │ read / write
    ▼
Linux Kernel
    │
    ▼
kprobe / kretprobe
    │
    ▼
Pixie eBPF Program
(socket_trace.c)
    │
    ├── System Call Argument 수집
    ├── BPF Map State 관리
    ├── Socket Payload 수집
    └── Protocol Inference
             │
             ▼
      socket_data_event_t
             │
             ▼
         Perf Buffer
════════════════════════════
       Kernel / User Space
════════════════════════════
             │
             ▼
       Stirling User Space
```

이후에는 이 흐름을 실제 소스 코드 기준으로 하나씩 따라가 보았다.

---

# 5. eBPF Program은 어떻게 Kernel Event에 연결되는가?

## `socket_trace.c`가 있다고 자동으로 실행되는 것은 아니다

`socket_trace.c`에는 다음과 같이 eBPF 함수들이 구현되어 있다.

```c
int syscall__probe_entry_send(...) {
    ...
}

int syscall__probe_ret_send(...) {
    ...
}
```

하지만 eBPF 함수가 정의되어 있다는 이유만으로 Kernel이 이 함수를 자동으로 실행하지는 않는다.

eBPF Program을 Kernel에 Load하고, 특정 Kernel Event가 발생했을 때 실행되도록 **Attach**해야 한다.

Chapter에서 BCC Python 예제로 봤던 구조와 동일한 개념이다.

```python
b.attach_kprobe(
    event=syscall,
    fn_name="hello"
)
```

Pixie에서는 이러한 연결을 User Space의 `socket_trace_connector.cc`에서 관리한다.

---

## `socket_trace_connector.cc`: User Space와 eBPF Program의 연결 지점

이 파일에서는 어떤 System Call과 어떤 eBPF 함수를 연결할 것인지 Probe Specification을 정의한다.

구조를 단순화하면 다음과 같다.

```cpp
{"connect", ProbeType::kEntry,  "syscall__probe_entry_connect"},
{"connect", ProbeType::kReturn, "syscall__probe_ret_connect"},

{"send", ProbeType::kEntry,  "syscall__probe_entry_send"},
{"send", ProbeType::kReturn, "syscall__probe_ret_send"},

{"recv", ProbeType::kEntry,  "syscall__probe_entry_recv"},
{"recv", ProbeType::kReturn, "syscall__probe_ret_recv"},
```

예를 들어 다음 관계가 만들어진다.

```text
send() 진입
    │
    │ kprobe
    ▼
syscall__probe_entry_send()


send() 반환
    │
    │ kretprobe
    ▼
syscall__probe_ret_send()
```

즉 `ProbeType::kEntry`는 함수 진입 시점, `ProbeType::kReturn`은 반환 시점을 추적하기 위한 설정이다.

---

## BCC를 이용한 Program 초기화와 Attach

`socket_trace_connector.cc`의 `InitBPF()`에서는 BCC Wrapper를 통해 BPF Program을 초기화하고 Probe를 Attach한다.

핵심 흐름은 다음과 같다.

```cpp
bcc_->InitBPFProgram(socket_trace_bcc_script, defines);

bcc_->AttachKProbes(kProbeSpecs);
```

이를 개념적으로 표현하면 다음과 같다.

```text
socket_trace.c
    │
    ▼
BCC
    │
    ├── BPF C Program 준비/컴파일 및 로드
    │
    └── kprobe / kretprobe Attach
              │
              ▼
       Linux Kernel Event
```

Chapter 4에서 직접 살펴본 `bpf()` System Call과 같은 저수준 처리는 Pixie 코드에서 매번 직접 호출하기보다 BCC 계층을 통해 추상화되어 있다.

---

# 6. `send()` 하나를 따라가며 Socket Tracing 이해하기

여러 System Call을 전부 개별적으로 분석하기보다는 `send()`를 대표 예제로 보면 전체 구조를 이해하기 쉽다.

```text
Application
    │
    │ send(fd, buf, len)
    ▼
Entry Probe
    │
    ├── fd
    ├── buffer pointer
    └── argument 저장
    │
    ▼
BPF Map
(active write args)
    │
    │ 실제 send 실행
    ▼
Return Probe
    │
    ├── return value 확인
    └── Entry에서 저장한 argument 복원
    │
    ▼
Socket Data 처리
    │
    ├── Connection 조회
    ├── Payload 확인
    ├── Protocol 추론
    └── Event 생성
    │
    ▼
Perf Buffer
```

여기서 중요한 것은 **왜 Entry와 Return을 모두 추적하는가**이다.

Entry 시점에는 `fd`, buffer pointer 등 System Call Argument를 얻을 수 있지만 아직 실제로 몇 byte가 처리됐는지는 알 수 없다.

반대로 Return 시점에는 실제 처리 결과를 알 수 있지만 Entry에서 받았던 Argument가 그대로 필요한 경우가 있다.

따라서 Pixie는 Entry 시점의 정보를 BPF Map에 저장하고 Return Probe에서 다시 조회한다.

---

# 7. BPF Map은 실제로 어떻게 사용되는가?

Chapter에서 BPF Map을 Kernel과 User Space 또는 eBPF Program 사이에서 데이터를 공유하는 Key-Value Store로 학습했다.

Pixie에서는 Map이 단순한 값 저장 이상의 역할을 한다.

## 7.1 Entry와 Return 사이의 State 저장

개념적으로 다음과 같다.

```text
Entry Probe
    │
    │ fd / buf / arguments
    ▼
active_*_args_map
    │
    │ lookup
    ▼
Return Probe
```

eBPF Program은 하나의 함수 호출 전체를 일반 Application 코드처럼 계속 붙잡고 기다리는 것이 아니다. Entry Probe 실행과 Return Probe 실행은 서로 다른 Event이기 때문에 두 시점의 Context를 연결하기 위한 State 저장 공간이 필요하다.

BPF Map이 이 역할을 수행한다.

---

## 7.2 Connection State 관리

`socket_trace.h`에는 Connection 정보를 나타내는 `conn_info_t`가 정의되어 있으며, `socket_trace.c`에서는 Connection 정보를 Map으로 관리한다.

개념적으로는 다음과 같다.

```text
Process + File Descriptor
          │
          ▼
    Connection ID
          │
          ▼
     conn_info_map
          │
          ├── Protocol
          ├── Endpoint / Role
          ├── Read/Write 상태
          ├── 이전 Payload 일부
          └── Connection 관련 상태
```

이 때문에 각각 독립적으로 발생하는 `read()`, `write()`, `send()`, `recv()` Event를 동일한 Network Connection의 흐름으로 연결할 수 있다.

---

## 7.3 User Space에서 eBPF 동작 제어

Pixie의 Socket Tracer에는 Control 용도의 Map도 존재한다.

이를 통해 User Space에서 설정한 값을 eBPF Program이 조회해 추적 여부 등을 판단할 수 있다.

```text
Stirling User Space
        │
        │ configuration
        ▼
     BPF Map
        │
        ▼
   eBPF Program
```

즉 Pixie에서는 BPF Map이 다음과 같이 여러 방향으로 활용된다.

```text
1. Probe Entry ↔ Return State
2. Connection State
3. User Space → Kernel Configuration
```

책에서 단순 Key-Value 저장소로 시작한 Map이 실제 프로그램에서는 **Event Correlation과 Stateful Tracing의 핵심 구성요소**가 된다는 점이 인상적이었다.

---

# 8. Socket Payload로 Application Protocol을 어떻게 알아내는가?

`socket_trace.c`에서 Socket Data를 얻은 이후에는 `protocol_inference.h`의 로직을 통해 Application Protocol을 추론한다.

지원되는 Protocol Type에는 HTTP, HTTP/2, MySQL, PostgreSQL, DNS, Redis, Kafka, MongoDB, AMQP 등이 정의되어 있다.

중요한 점은 Kernel 안에서 각 Protocol을 완전히 Parsing하는 것이 아니라 **Payload의 특징적인 Header와 Byte Pattern을 검사하여 Protocol과 Message Type을 추론한다는 것**이다.

---

## 8.1 HTTP: Payload Signature 확인

HTTP는 비교적 직관적이다.

예를 들어 Payload의 앞부분이 다음과 같다면 Request로 판단할 수 있다.

```text
GET ...
POST ...
PUT ...
DELETE ...
HEAD ...
```

반대로 다음과 같이 시작하면 Response로 판단할 수 있다.

```text
HTTP/1.1 ...
```

따라서 eBPF 단계에서는 전체 HTTP Message를 완전히 Parsing하는 대신 앞부분의 특징적인 Byte Pattern을 이용해 빠르게 Protocol을 추론한다.

```text
Socket Payload
     │
     ├── "GET"  → HTTP Request
     ├── "POST" → HTTP Request
     └── "HTTP" → HTTP Response
```

---

## 8.2 MySQL과 Kafka: 한 번의 Event만으로 부족한 경우

Protocol Inference에서 특히 흥미로운 부분은 MySQL과 Kafka였다.

Network Data가 항상 하나의 `read()` 호출에서 완전한 Message 형태로 전달된다는 보장은 없다.

예를 들어 다음과 같이 나뉠 수 있다.

```text
read #1
└── 4-byte Header

read #2
└── Message Body
```

첫 번째 Event만 보면 Header만 있고, 두 번째 Event만 보면 Header가 없다.

Pixie는 Connection State에 이전 Data 일부를 저장해 다음 Event에서 다시 사용한다.

```text
eBPF Invocation #1
       │
       │ 4-byte Header
       ▼
conn_info.prev_buf
       │
       ▼
eBPF Invocation #2
       │
       │ Message Body
       ▼
Previous Header + Current Data
       │
       ▼
Protocol Inference
```

즉 **BPF Map을 이용해 서로 다른 eBPF 실행 사이에 상태를 유지하는 실제 사례**이다.

---

## 8.3 DNS: Header의 여러 Field를 함께 검사

DNS는 단순 문자열 Signature만 확인하는 것이 아니라 Header의 Flag와 Question/Answer Count 등 여러 Field가 정상적인 범위에 있는지 확인한다.

또한 QR Bit를 이용해 Query와 Response를 구분한다.

```text
QR = 0
→ Query

QR = 1
→ Response
```

여러 조건을 함께 검사하는 이유는 임의의 Socket Data를 특정 Protocol로 잘못 판단하는 False Positive를 줄이기 위해서이다.

---

## 8.4 Kernel에서 모든 Parsing을 하지 않는 이유

Protocol Inference 코드를 보면서 중요한 점은 **Kernel에서 모든 작업을 끝내려고 하지 않는다는 것**이다.

eBPF Program은 Verifier의 검증을 통과해야 하고 Stack과 Instruction 수 등에 제약이 있다. 또한 Kernel Context에서 실행되는 만큼 불필요하게 복잡하거나 오래 실행되는 로직을 넣는 것은 적절하지 않다.

따라서 Pixie는 Kernel에서 필요한 수준의 식별과 Filtering을 수행하고, 더 복잡한 처리는 User Space에서 담당하도록 역할을 나눈다.

```text
Kernel eBPF
────────────────────────
Event 감지
필요한 Metadata 수집
Connection State 관리
Protocol 추론
Filtering

            │
            ▼

User Space
────────────────────────
더 복잡한 처리
Protocol Parsing
Event Correlation
Observability Data 처리
```

---

# 9. Kernel에서 User Space로 Event 전달하기

eBPF Program이 Socket Event를 수집하고 Protocol을 추론한 뒤에는 결과를 User Space로 전달해야 한다.

Pixie에서는 Perf Buffer를 사용한다.

`socket_trace.c`에는 다음과 같은 Perf Output이 정의되어 있다.

```c
BPF_PERF_OUTPUT(socket_data_events);
BPF_PERF_OUTPUT(socket_control_events);
BPF_PERF_OUTPUT(conn_stats_events);
BPF_PERF_OUTPUT(mmap_events);
```

Socket Payload와 관련된 Event는 `socket_data_event_t`와 같은 구조체에 필요한 정보를 담아 Perf Buffer로 전달한다.

```text
eBPF Program
     │
     │ socket_data_event_t
     ▼
socket_data_events
     │
     │ Perf Buffer
═════╪══════════════════
     ▼
Stirling User Space
```

---

## User Space에서는 누가 받는가?

`socket_trace_connector.cc`에서는 Perf Buffer를 열고 각각의 Event에 Handler를 연결한다.

개념적으로 다음과 같은 관계이다.

```text
socket_data_events
       │
       ▼
HandleDataEvent()

socket_control_events
       │
       ▼
HandleControlEvent()

conn_stats_events
       │
       ▼
HandleConnStatsEvent()
```

따라서 Kernel과 User Space의 전체 흐름이 연결된다.

```text
socket_trace.c

perf_submit(...)
      │
      ▼
Perf Buffer
      │
══════╪════════ Kernel / User Space
      ▼
socket_trace_connector.cc

OpenPerfBuffers()
      │
      ▼
Event Handler
```

Chapter에서 본 간단한 Perf Buffer 예제와 기본 메커니즘은 동일하지만, Pixie에서는 여러 종류의 Event와 구조체, Handler가 사용되는 형태로 확장되어 있다.

---

# 10. 실제 코드에서 확인한 eBPF의 제약

Pixie의 코드를 보면서 책의 예제보다 확실하게 드러난 부분은 **eBPF의 제약이 실제 구현 방식 자체를 바꾼다는 점**이었다.

## Stack 크기

큰 Event 구조체를 함수의 Local Variable로 계속 생성하는 대신 Per-CPU Array 등을 임시 저장 공간으로 사용하는 코드가 존재한다.

이는 eBPF Program이 제한된 Stack을 사용하기 때문에 일반적인 C Application과 같은 방식으로 큰 데이터를 자유롭게 Stack에 둘 수 없다는 점과 연결된다.

## Verifier

`socket_trace.c`에는 Bounds Check와 Loop 처리 등 Verifier가 Program의 안전성을 증명할 수 있도록 작성된 코드가 존재한다.

즉 논리적으로 올바른 C 코드라는 것만으로 충분하지 않고 **Verifier가 해당 Memory Access와 Control Flow가 안전하다는 것을 판단할 수 있는 형태**로 작성해야 한다.

## Instruction Limit

`socket_trace_connector.cc`에서는 Kernel Version과 활성화할 Protocol에 따라 eBPF Program의 크기와 Loop 관련 설정을 조절한다.

또한 Protocol별 Enable Define을 BPF Program 초기화 시 전달한다.

```text
ENABLE_HTTP_TRACING
ENABLE_MYSQL_TRACING
ENABLE_PGSQL_TRACING
ENABLE_KAFKA_TRACING
...
```

모든 기능을 무조건 하나의 거대한 BPF Program에 넣는 것이 아니라 Kernel Version과 eBPF 제약을 고려해 실행할 로직을 조절하는 구조이다.

---

# 11. TLS Traffic은 어떻게 처리하는가?

Socket Tracing만으로 모든 Application Payload를 동일하게 볼 수 있는 것은 아니다.

TLS가 적용된 Application에서는 Socket 계층으로 내려가기 전에 Application Data가 암호화되기 때문에 일반적인 Socket Probe에서는 암호화된 Data를 보게 된다.

Pixie의 Socket Tracer 관련 코드에는 OpenSSL, Go TLS 등 User Space TLS Library를 추적하기 위한 코드도 포함되어 있다.

개념적으로는 다음과 같다.

```text
Application Plaintext
       │
       ▼
TLS Library
       │
       │ uprobe
       ├──────────────→ eBPF
       │
       ▼
Encryption
       │
       ▼
send()
       │
       │ kprobe
       ├──────────────→ eBPF
       ▼
Kernel Socket
```

즉 Kernel Function을 추적하는 kprobe만 사용하는 것이 아니라, 필요하면 User Space Library Function에 uprobe를 연결해 Kernel로 내려가기 전의 Data를 관찰하는 방식까지 확장한다.

이번 분석에서는 TLS 관련 eBPF Program 내부까지 상세하게 따라가지는 않고, Socket Tracer가 kprobe뿐 아니라 User Space Probe와 결합될 수 있다는 구조까지만 확인했다.

---

# 12. Pixie Socket Tracer의 전체 eBPF 흐름

지금까지 분석한 내용을 하나로 연결하면 다음과 같다.

```text
┌──────────────────────────────┐
│ Application                  │
│ HTTP / MySQL / PGSQL / ...   │
└──────────────┬───────────────┘
               │
               │ send / recv
               │ read / write
               │ connect / accept
               ▼
┌──────────────────────────────┐
│ Linux Kernel                 │
│                              │
│ kprobe / kretprobe           │
└──────────────┬───────────────┘
               │
               ▼
        socket_trace.c
               │
        ┌──────┴────────┐
        │               │
        ▼               ▼
  BPF Maps      protocol_inference.h
        │               │
        │               ├── HTTP
        │               ├── MySQL
        │               ├── PostgreSQL
        │               ├── DNS
        │               ├── Kafka
        │               └── ...
        │               │
        └───────┬───────┘
                ▼
       socket_data_event_t
                │
                ▼
           Perf Buffer
════════════════╪════════════════
      Kernel    │    User Space
                ▼
    socket_trace_connector.cc
                │
                ▼
          Event Handler
                │
                ▼
       Stirling Data Pipeline
```

여기에 Program Lifecycle까지 포함하면 다음과 같이 볼 수 있다.

```text
socket_trace_connector.cc
        │
        ├── Protocol/Runtime 설정
        │
        ▼
BCC를 통한 BPF Program 초기화
        │
        ▼
eBPF Program Load
        │
        ▼
kprobe / kretprobe Attach
        │
        ▼
Kernel Event 발생
        │
        ▼
socket_trace.c 실행
        │
        ├── BPF Map State
        └── Protocol Inference
        │
        ▼
Perf Buffer
        │
        ▼
Stirling User Space
```

---

# 13. Learning eBPF Chapter 1~4와 연결해서 보기

이번 Pixie 분석을 통해 책에서 각각 따로 배웠던 개념들이 실제 프로그램에서는 하나의 흐름으로 연결되어 사용된다는 것을 확인할 수 있었다.

| Learning eBPF에서 학습한 개념 | Pixie에서 확인한 형태 |
|---|---|
| Event-driven eBPF | Socket/System Call Event 발생 시 BPF Program 실행 |
| BCC | BPF Program 초기화와 Probe Attach를 관리하는 계층 |
| kprobe | `send`, `recv`, `connect` 등 함수 진입 추적 |
| kretprobe | System Call Return Value와 처리 결과 추적 |
| BPF Map | Probe State, Connection State, Control 값 관리 |
| Helper Function | PID/TGID, Time, Memory Data 등을 얻는 데 사용 |
| Kernel/User Space | eBPF Program과 Stirling Connector의 역할 분리 |
| Perf Buffer | Socket Event를 Kernel에서 User Space로 전달 |
| Program Load | BCC를 통해 BPF Program을 Kernel에 준비/로드 |
| Attach | `kProbeSpecs`와 `AttachKProbes()`를 통해 Event와 BPF Program 연결 |
| Verifier | Bounds Check, Loop, Memory Access 구현에 직접 영향 |
| eBPF 제약 | Stack/Instruction Limit을 고려한 코드와 기능 구성 |

책의 예제에서는 하나의 기능을 이해하기 위해 각각의 요소를 단순한 형태로 사용했다면, Pixie에서는 이러한 요소들이 **Network Connection을 지속적으로 추적하는 Stateful Observability Pipeline**을 만들기 위해 함께 사용되고 있었다.

특히 BPF Map이 단순한 Kernel/User Space 공유 저장소를 넘어 Entry와 Return Event를 연결하고 Connection State를 유지하는 데 사용된다는 점, 그리고 Perf Buffer가 실제 Production Code에서도 Kernel→User Space Event 전달 경로로 사용된다는 점을 소스에서 직접 확인할 수 있었다.

---


# 14. Pixie를 분석하면서 다시 이해한 Learning eBPF Chapter 1~4

책으로 Chapter 1~4를 공부할 때는 `kprobe`, `BPF Map`, `Perf Buffer`, `bpf()` 같은 개념이 각각의 기능처럼 보였다. Pixie의 실제 코드를 따라가면서 가장 크게 달라진 점은, **이 기능들이 서로 독립적인 기술이 아니라 하나의 eBPF Application을 구성하기 위해 동시에 연결되어 사용된다는 점**이었다.

## Chapter 1: eBPF는 Kernel 안에서 실행되는 코드라는 말의 의미

Chapter 1에서는 eBPF가 Kernel Source를 수정하거나 Kernel Module을 직접 작성하지 않고도 Kernel Event에 사용자 정의 로직을 연결할 수 있다는 점을 배웠다.

책으로 볼 때는 이 설명이 다소 추상적으로 느껴졌지만 Pixie에서는 그 의미가 훨씬 구체적으로 보였다.

Pixie는 Application 내부 코드를 직접 수정해서 `send()`나 `recv()`를 기록하는 대신, 해당 Application이 결국 거쳐야 하는 Linux의 Socket/System Call 경로를 관찰한다.

```text
Application A ─┐
Application B ─┼─→ Linux Kernel의 Socket/System Call
Application C ─┘                 │
                                 ▼
                           Pixie eBPF Program
```

특히 Container나 Pod도 Host Kernel을 공유하기 때문에, Node의 Kernel에서 Event를 관찰하면 개별 Application마다 별도의 계측 코드를 넣지 않고도 여러 Workload의 동작을 볼 수 있다는 점이 실제 코드 구조와 연결되었다.

즉 Chapter 1에서 이야기한 **"Kernel에 동적으로 기능을 추가한다"**는 것은 단순히 Kernel에서 코드를 실행한다는 뜻만이 아니라, Application보다 아래 계층에 공통 Observation Point를 만들 수 있다는 의미로 이해하게 되었다.

---

## Chapter 2: BPF Map은 단순한 Key-Value 저장소가 아니었다

Chapter 2에서는 Hash Map에 Count를 저장하거나 Config 값을 넣는 예제를 통해 BPF Map을 처음 접했다.

당시에는 다음과 같이 이해하기 쉬웠다.

```text
Key → Value 저장
User Space ↔ Kernel 사이에서 값 공유
```

Pixie에서는 Map이 훨씬 다양한 역할을 하고 있었다.

### 1. 서로 다른 Probe 실행을 연결하는 임시 State

`send()`의 Entry Probe에서는 FD와 Buffer Pointer를 알 수 있지만 실제로 몇 Byte가 전송됐는지는 아직 알 수 없다.

Return Probe에서는 실제 Return Value를 알 수 있지만 Entry에서 얻은 Argument가 다시 필요하다.

Pixie는 이 두 실행을 BPF Map으로 연결한다.

```text
send() Entry
    │
    ├── fd
    └── buf
    │
    ▼
active_write_args_map
    │
    ▼
send() Return
    │
    └── 실제 전송 Byte와 결합
```

이 코드를 보면서 **BPF Map이 eBPF Program의 여러 실행 사이에 Context를 이어주는 State Store가 될 수 있다는 점**을 실제로 이해할 수 있었다.

### 2. Connection 전체의 장기 State

`conn_info_map`에는 단순 Count 하나가 아니라 Connection의 상태가 누적된다.

```text
Connection
├── Process Identity
├── File Descriptor
├── Local / Remote Address
├── Client / Server Role
├── Protocol
├── SSL 상태
├── Read / Write Byte
└── Protocol Inference 상태
```

즉 eBPF Program 자체는 Event가 발생할 때 짧게 실행되지만, Map을 사용하면 여러 Event를 하나의 Connection 흐름으로 묶어 **Stateful Tracing**을 구현할 수 있다.

### 3. Config Map과 Event Buffer는 방향이 다르다

Chapter 2에서 Config Map과 Perf Buffer를 따로 배웠지만 Pixie에서 두 방향이 더 명확하게 보였다.

```text
User Space
    │
    │ 설정
    ▼
control_map
    │
    ▼
eBPF Program
```

반대로 Event는:

```text
eBPF Program
    │
    │ Event
    ▼
Perf Buffer
    │
    ▼
User Space
```

이다.

즉 Map/Buffer를 단순히 "Kernel과 User Space가 데이터를 주고받는 방법"이라고 묶기보다, **설정과 State는 Map, 연속적으로 발생하는 Event는 Buffer**라는 역할 차이가 실제 코드에서 드러났다.

---

## Chapter 2: Perf Buffer가 필요한 이유도 더 명확해졌다

책의 `hello-buffer` 예제에서는 PID, UID, Command 같은 작은 Event를 Perf Buffer로 보냈다.

Pixie에서는 동일한 메커니즘으로 다음과 같은 훨씬 큰 Context를 전달한다.

```text
Timestamp
Connection ID
Protocol
Client / Server Role
Ingress / Egress
SSL 여부
Source Function
Message Position
Payload
```

이를 보면서 Perf Buffer는 단순히 `printk()`보다 편리한 출력 방법이 아니라, **Kernel에서 감지한 Event를 구조화된 Streaming Data로 User Space에 전달하는 핵심 통로**라는 점이 명확해졌다.

또한 Connection State처럼 계속 유지해야 하는 값은 Map에 두고, 실제 발생한 Network Event는 Perf Buffer로 내보내는 것을 보면서 Map과 Buffer의 사용 기준도 더 분명해졌다.

---

## Chapter 3: eBPF Program은 Source Code 하나만으로 완성되지 않는다

Chapter 3에서는 eBPF C Source가 Bytecode로 Compile되고 Kernel에 Load된 뒤 특정 Hook에 Attach되어야 실제로 실행된다는 흐름을 배웠다.

Pixie를 보기 전에는 `socket_trace.c`에 다음 함수가 있으면 이것이 곧 `send()`를 추적하는 eBPF Program이라고 생각하기 쉬웠다.

```c
int syscall__probe_entry_send(...) {
    ...
}
```

하지만 실제로는 이 함수가 존재하는 것과 실행되는 것은 별개의 문제였다.

User Space의 `socket_trace_connector.cc`가 다음 정보를 연결한다.

```text
Kernel Event
    send()

Probe Type
    Entry

eBPF Function
    syscall__probe_entry_send
```

그리고 BCC를 이용해 Program을 초기화한 뒤 Probe를 Attach한다.

```text
eBPF Source
    ↓
BCC
    ↓
Program Load
    ↓
kprobe / kretprobe Attach
    ↓
Kernel Event 발생
    ↓
eBPF Function 실행
```

이 코드를 보고 Chapter 3에서 배운 **Load와 Attach가 서로 다른 단계**라는 점이 훨씬 확실해졌다.

Program을 Kernel에 Load하는 것만으로는 아무 Event도 관찰하지 못하고, 실제 실행 시점을 결정하는 Hook과 연결되어야 비로소 Event-driven Program이 된다.

---

## Chapter 3: Entry와 Return을 같이 보는 이유

책에서 kprobe를 처음 볼 때는 특정 Kernel Function이 호출되는 순간을 관찰하는 기능으로 이해했다.

Pixie에서는 같은 System Call에 Entry와 Return Probe를 모두 사용하는 이유가 명확했다.

```text
Entry
→ 어떤 Argument로 호출됐는가?

Return
→ 실제 결과가 무엇인가?
```

예를 들어 `send()`라면:

```text
Entry
fd = 7
buf = 0x...
requested size = ...

            ↓

Return
actual bytes = 500
```

처럼 서로 다른 시점에 필요한 정보가 존재한다.

그리고 두 시점은 BPF Map으로 연결된다.

이를 통해 kprobe/kretprobe는 단순히 "함수 호출을 감시한다"는 기능이 아니라, **Kernel Function의 입력과 결과를 함께 관찰하여 하나의 의미 있는 Event로 재구성할 수 있는 Hook**이라는 점을 이해할 수 있었다.

---

## Chapter 4: `bpf()`를 직접 호출하지 않아도 Chapter 4의 구조는 그대로 존재한다

Chapter 4에서는 `bpf()` System Call을 따라가며 Map 생성, Program Load, Map Update 등의 실제 Kernel Interface를 살펴봤다.

Pixie의 코드에서는 `bpf(BPF_PROG_LOAD, ...)` 같은 호출을 직접 찾는 방식보다 BCC Wrapper가 사용된다.

처음에는 Chapter 4에서 본 저수준 흐름이 실제 프로젝트에서는 보이지 않는 것처럼 느껴졌지만, 오히려 그 반대였다.

```text
Pixie User Space
      │
      ▼
BCC Abstraction
      │
      ├── Program Load
      ├── Map 준비
      ├── Probe Attach
      └── Perf Buffer 준비
      │
      ▼
Linux Kernel
```

즉 Chapter 4에서 배운 작업들이 사라진 것이 아니라 **Library 아래로 추상화되어 있었던 것**이다.

이를 통해 `bpf()` System Call을 공부한 목적도 다시 이해할 수 있었다. 실제 Application에서는 BCC, libbpf 같은 Library를 사용할 가능성이 높지만, 그 아래에서 어떤 Kernel Object와 작업이 만들어지는지 알아야 높은 수준의 코드도 정확하게 해석할 수 있다.

---

## 실제 프로젝트에서는 eBPF 제약을 피하는 것이 아니라 설계에 반영한다

책의 초반 예제에서는 eBPF의 Stack이나 Verifier 제약이 크게 체감되지 않았다.

Pixie의 `socket_trace.c`와 `protocol_inference.h`에서는 이런 제약이 코드 형태를 직접 바꾸고 있었다.

예를 들어 큰 Event 구조를 Stack에 두지 않기 위해 Per-CPU Array를 임시 Buffer처럼 사용하고, Loop를 제한하거나 Unroll하며, 오래된 Kernel Verifier를 만족시키기 위해 Bounds Check를 명시적으로 구성한다.

Protocol Inference도 모든 Protocol을 완전하게 Parsing하지 않는다.

```text
Kernel eBPF
→ 최소한의 Header/Pattern 검사
→ 필요한 Metadata와 Event만 전달

User Space
→ 더 복잡한 Parsing과 후속 처리
```

처음에는 eBPF가 Kernel 안에서 실행되기 때문에 원하는 로직을 Kernel 가까이에서 모두 처리하는 것이 장점이라고 생각하기 쉬웠다.

하지만 Pixie를 분석하면서 오히려 **무엇을 Kernel에서 처리하고 무엇을 User Space로 넘길지를 나누는 것이 eBPF Application 설계의 중요한 부분**이라는 점을 알 수 있었다.

---

## Protocol Inference를 보며 이해한 eBPF의 역할

Pixie가 HTTP나 MySQL을 인식한다고 해서 eBPF Program이 완전한 Application Protocol Parser 역할을 하는 것은 아니었다.

HTTP에서는 `GET`, `POST`, `HTTP` 같은 특징을 확인하고, DNS에서는 Header Field를 검사하며, MySQL과 Kafka에서는 이전 Event의 Header 일부를 State로 유지한다.

즉 Kernel에서는 다음 정도를 수행한다.

```text
Raw Bytes
    ↓
특징적인 Header / Pattern 확인
    ↓
Protocol 후보 추론
    ↓
필요한 Event만 User Space로 전달
```

이 과정에서 느낀 점은 **eBPF의 강점이 모든 분석을 Kernel에서 끝내는 데 있는 것이 아니라, Kernel에서만 얻기 쉬운 Context를 적절한 시점에 확보하고 필요 없는 데이터를 일찍 줄여 User Space와 결합하는 데 있다는 것**이다.

---

## Chapter 1~4가 실제로는 하나의 흐름이었다

Chapter별로 공부할 때는 다음처럼 각각의 주제가 나뉘어 있었다.

```text
Chapter 1
→ eBPF와 Kernel

Chapter 2
→ BCC / Map / Perf Buffer

Chapter 3
→ Compile / Load / Attach

Chapter 4
→ bpf() System Call과 Kernel Object
```

Pixie를 분석하면서 이 내용들이 다음 하나의 실행 흐름이라는 점이 보였다.

```text
User Space Loader
        │
        │ BCC
        ▼
BPF Program Compile / Load
        │
        ▼
Kernel Hook Attach
        │
        ▼
Event 발생
        │
        ▼
eBPF Program 실행
        │
        ├── Helper로 Context 수집
        ├── Map으로 State 유지
        ├── 필요한 Data Filtering
        └── Event 생성
        │
        ▼
Perf Buffer
        │
        ▼
User Space 처리
```

책에서 Chapter별로 나눠 학습한 이유는 각각의 구성요소를 이해하기 위해서였지만, 실제 eBPF 기반 프로젝트에서는 이 모든 요소가 함께 있어야 하나의 기능이 완성된다.

Pixie의 Socket Tracer를 분석하면서 가장 크게 정리된 부분도 바로 이 점이었다. **eBPF Program만 보는 것이 아니라 User Space Loader, Attachment Point, BPF Map, Event Buffer까지 함께 봐야 실제 eBPF Application을 이해할 수 있다.**

---


# 15. 정리

Pixie 전체 Source Code를 분석하려고 하면 범위가 매우 크지만, eBPF라는 관점에서 범위를 좁히면 핵심은 Stirling의 Socket Tracer에서 확인할 수 있었다.

이번에 분석한 흐름은 다음과 같다.

```text
Application Network I/O
        ↓
Kernel Function / System Call
        ↓
kprobe / kretprobe
        ↓
eBPF Socket Tracer
        ↓
BPF Map을 통한 State 관리
        ↓
Socket Payload 수집
        ↓
Application Protocol 추론
        ↓
Perf Buffer
        ↓
Stirling User Space
```

Chapter 1~4에서 배웠던 `Load → Attach → Event → eBPF Program → Map → Perf Buffer → User Space`라는 기본 구조가 실제 Pixie에서도 그대로 존재했다.

차이가 있다면 실제 Production Code에서는 하나의 Event만 처리하는 것이 아니라 여러 System Call을 동시에 추적하고, 서로 다른 Event 사이의 State를 BPF Map으로 연결하며, Application Protocol을 추론하고, Kernel Version과 Verifier/Instruction Limit까지 고려해 eBPF Program을 구성한다는 점이다.

따라서 이번 분석은 새로운 eBPF 기능을 많이 찾는 것보다는, **기본적인 eBPF Primitive들이 실제 Observability System을 만들 때 어떤 방식으로 조합되고 확장되는지 확인하는 과정**에 가까웠다.
