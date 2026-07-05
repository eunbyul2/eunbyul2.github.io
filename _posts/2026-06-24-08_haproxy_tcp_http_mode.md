---
layout: post
title: "HAProxy TCP Mode와 HTTP Mode 차이"
date: 2026-06-24 23:40:00 +0900
categories: [스터디, 인프라, Network, HAProxy]
tags: [HAProxy, TCPMode, HTTPMode, SSLPassthrough, TLSTermination, SNI, PROXYProtocol, Kubernetes]
published: true
---

## 1. HAProxy TCP Mode와 HTTP Mode를 구분해야 하는 이유

HAProxy는 대표적인 오픈소스 Load Balancer이자 Reverse Proxy입니다. HAProxy를 사용할 때 가장 먼저 결정해야 하는 값 중 하나가 `mode tcp`와 `mode http`입니다.

이 둘의 차이는 단순히 설정 문법의 차이가 아닙니다. HAProxy가 트래픽을 **어느 계층까지 이해하고 처리할 것인지**를 결정하는 핵심 기준입니다.

- `mode tcp`는 TCP 연결을 기준으로 동작합니다.
- `mode http`는 HTTP 요청과 응답을 해석하면서 동작합니다.

즉, 두 mode의 차이는 다음 한 문장으로 정리할 수 있습니다.

```text
TCP Mode  = 연결(Connection)을 보고 처리한다.
HTTP Mode = HTTP 요청(Request)과 응답(Response)을 보고 처리한다.
```

이 차이 때문에 실무에서는 다음과 같은 문제가 자주 발생합니다.

- HTTPS 트래픽을 `mode tcp`로 넘기면서 Path Routing을 기대함
- SSL Passthrough 환경에서 HTTP Header를 수정하려고 함
- TCP Health Check만 설정해놓고 애플리케이션 정상성을 보장한다고 오해함
- HTTP 서비스인데 `X-Forwarded-Proto`를 넣지 않아 무한 Redirect가 발생함
- Kubernetes API Server, Harbor, ArgoCD, Ceph Dashboard를 같은 방식으로 처리하려고 함

공식 참고자료:

- HAProxy Configuration Manual: https://www.haproxy.com/documentation/haproxy-configuration-manual/latest/
- HAProxy WebSocket Documentation: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/protocol-support/websocket/
- TLS 1.3 RFC 8446: https://datatracker.ietf.org/doc/html/rfc8446
- HTTP RFC 9110: https://datatracker.ietf.org/doc/html/rfc9110
- TLS Encrypted Client Hello RFC 9849: https://www.rfc-editor.org/rfc/rfc9849.html

----

## 2. 먼저 TCP를 이해해야 하는 이유

`mode tcp`를 이해하려면 TCP 자체를 먼저 알아야 합니다.

HTTP, HTTPS, gRPC, WebSocket 같은 대부분의 웹 트래픽은 결국 TCP 위에서 동작합니다.

```text
Application Layer : HTTP, HTTPS, gRPC, WebSocket
Transport Layer   : TCP
Network Layer     : IP
Link Layer        : Ethernet
```

HTTP는 애플리케이션 계층 프로토콜이고, TCP는 전송 계층 프로토콜입니다.

그래서 HAProxy가 `mode tcp`로 동작한다는 말은 다음 의미입니다.

```text
HAProxy가 HTTP 내용을 해석하지 않고 TCP 연결 단위로만 중계한다.
```

반대로 `mode http`는 다음 의미입니다.

```text
HAProxy가 TCP 연결 위에 실려 있는 HTTP 메시지까지 파싱한다.
```

----

## 2-1. OSI 계층 기준으로 보는 TCP Mode와 HTTP Mode

HAProxy의 `mode tcp`와 `mode http` 차이를 이해하려면 OSI 계층 관점이 필요합니다.

```text
L7 Application  : HTTP, HTTPS, gRPC, WebSocket
L6 Presentation : TLS/SSL, Encoding
L5 Session      : Session 관리 개념
L4 Transport    : TCP, UDP
L3 Network      : IP
L2 Data Link    : Ethernet, VLAN
L1 Physical     : Cable, NIC, 광케이블
```

HAProxy가 `mode tcp`로 동작하면 기본적으로 L4 수준의 연결을 기준으로 판단합니다.

```text
Client IP:Port  →  HAProxy  →  Backend IP:Port

HAProxy가 보는 핵심 정보:
- Source IP
- Source Port
- Destination IP
- Destination Port
- TCP 연결 상태
- 연결 시간
- 전송 바이트 수
```

반대로 `mode http`로 동작하면 L7 HTTP 메시지까지 분석합니다.

```text
GET /api/users HTTP/1.1
Host: api.example.com
Cookie: SESSIONID=abc
Authorization: Bearer xxx
```

HTTP Mode에서는 위 요청에서 다음 정보를 기준으로 판단할 수 있습니다.

- `GET`, `POST` 같은 Method
- `/api/users` 같은 Path
- `api.example.com` 같은 Host Header
- `SESSIONID=abc` 같은 Cookie
- `Authorization` 같은 인증 Header
- 응답의 `200`, `301`, `404`, `500` 같은 Status Code

정리하면 다음과 같습니다.

```text
TCP Mode
  ↓
L4까지만 보고 판단
  ↓
연결을 어느 Backend로 보낼지 결정

HTTP Mode
  ↓
L7 HTTP까지 보고 판단
  ↓
요청 내용을 기준으로 어느 Backend로 보낼지 결정
```

이 차이가 문서 전체의 핵심입니다.

----

## 3. TCP란 무엇인가

TCP는 Transmission Control Protocol의 약자입니다. 클라이언트와 서버 사이에 **신뢰성 있는 연결 기반 통신**을 제공하는 프로토콜입니다.

TCP의 핵심 특징은 다음과 같습니다.

| 특징 | 설명 |
|---|---|
| 연결 기반 | 데이터를 보내기 전에 연결을 만든다. |
| 신뢰성 보장 | 손실된 데이터는 재전송한다. |
| 순서 보장 | 보낸 순서대로 애플리케이션에 전달한다. |
| 흐름 제어 | 수신자가 처리 가능한 속도에 맞춘다. |
| 혼잡 제어 | 네트워크 혼잡을 고려해 전송량을 조절한다. |
| Stream 기반 | 데이터를 연속된 바이트 흐름으로 다룬다. |

TCP는 HTTP 요청 하나하나를 직접 이해하지 않습니다. TCP 입장에서 데이터는 단순한 바이트 흐름입니다.

예를 들어 HTTP 요청이 다음과 같더라도,

```http
GET /api/users HTTP/1.1
Host: api.example.com
Authorization: Bearer token
```

TCP 입장에서는 그냥 다음과 같은 바이트 데이터입니다.

```text
47 45 54 20 2f 61 70 69 2f 75 73 65 72 73 ...
```

즉, TCP는 `GET`, `/api/users`, `Host`, `Authorization`의 의미를 모릅니다.

----

## 4. TCP Segment, Socket, Connection, Stream

### 4-1) TCP Segment

TCP Segment는 TCP 계층에서 데이터를 나누어 보내는 단위입니다.

애플리케이션이 큰 데이터를 보내면 TCP는 이를 여러 조각으로 나누어 전송할 수 있습니다.

```text
HTTP Request
  ↓
TCP Segment 1
TCP Segment 2
TCP Segment 3
```

중요한 점은 HTTP 요청 하나가 TCP Segment 하나와 1:1로 대응하지 않는다는 것입니다.

- 하나의 HTTP 요청이 여러 TCP Segment로 나뉠 수 있습니다.
- 여러 HTTP 요청이 같은 TCP 연결에서 연속적으로 흐를 수 있습니다.
- TCP는 HTTP 메시지 경계를 모릅니다.

### 4-2) Socket

Socket은 네트워크 통신의 끝점입니다.

일반적으로 하나의 TCP 연결은 다음 4가지 정보로 식별됩니다.

```text
Client IP
Client Port
Server IP
Server Port
```

예시:

```text
192.168.10.50:53124 → 10.0.0.10:443
```

HAProxy가 TCP Mode에서 주로 보는 정보도 이 수준입니다.

- 출발지 IP
- 출발지 Port
- 목적지 IP
- 목적지 Port
- 연결 상태
- 전송 바이트 수
- 연결 시간
- 일부 초기 Payload 검사

### 4-3) TCP Connection

TCP Connection은 클라이언트와 서버 사이에 만들어진 논리적인 연결입니다.

연결이 만들어진 뒤에는 양쪽이 데이터를 주고받을 수 있습니다.

```text
Client <================ TCP Connection ================> Server
```

### 4-4) TCP Stream

TCP Stream은 연결 위에서 흐르는 연속된 바이트 데이터입니다.

TCP는 메시지 단위가 아니라 Stream 단위입니다.

```text
Byte 1 → Byte 2 → Byte 3 → Byte 4 → Byte 5 → ...
```

그래서 HAProxy가 `mode tcp`일 때는 기본적으로 이 Stream을 중계합니다.

----

## 5. TCP Connection이 만들어지는 과정: 3-Way Handshake

TCP는 데이터를 보내기 전에 연결을 먼저 만듭니다. 이 과정을 3-Way Handshake라고 합니다.

```text
Client                              Server
  |                                   |
  | -------- SYN -------------------> |
  |                                   |
  | <----- SYN + ACK ---------------- |
  |                                   |
  | -------- ACK -------------------> |
  |                                   |
  |        TCP Connection Established |
```

각 단계의 의미는 다음과 같습니다.

| 단계 | 의미 |
|---|---|
| SYN | 클라이언트가 연결 시작을 요청한다. |
| SYN/ACK | 서버가 요청을 수락하고 응답한다. |
| ACK | 클라이언트가 서버 응답을 확인한다. |

이후에 TLS 또는 HTTP 데이터가 흐릅니다.

HTTPS 요청의 전체 흐름은 다음과 같이 볼 수 있습니다.

```text
Client
  ↓
TCP SYN
  ↓
TCP SYN/ACK
  ↓
TCP ACK
  ↓
TCP Connection 생성
  ↓
TLS Handshake
  ↓
Encrypted HTTP Request
  ↓
Encrypted HTTP Response
```

이 흐름이 중요한 이유는 다음과 같습니다.

- TCP 연결이 먼저 만들어집니다.
- 그 다음 TLS Handshake가 진행됩니다.
- TLS 이후에 HTTP 요청이 암호화되어 흐릅니다.
- TLS를 종료하지 않으면 HAProxy는 HTTP Header, Path, Method를 볼 수 없습니다.

----

## 6. HTTP는 TCP 위에서 동작한다

HTTP는 애플리케이션 계층 프로토콜입니다. 일반적인 HTTP/1.1, HTTP/2, gRPC, WebSocket은 TCP 연결 위에서 동작합니다.

HTTP 요청은 다음과 같은 구조를 가집니다.

```http
GET /api/users HTTP/1.1
Host: api.example.com
User-Agent: curl/8.0
Authorization: Bearer xxx
```

HTTP 응답은 다음과 같은 구조입니다.

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"status":"ok"}
```

HAProxy가 HTTP Mode로 동작하면 위 구조를 파싱할 수 있습니다.

그래서 다음과 같은 조건을 사용할 수 있습니다.

```haproxy
acl is_api path_beg /api
acl is_admin hdr(host) -i admin.example.com
acl is_post method POST
```

하지만 TCP Mode에서는 이 구조를 해석하지 않습니다.

----

## 7. mode tcp란 무엇인가

`mode tcp`는 HAProxy가 Layer 4 수준에서 TCP 연결을 중계하는 방식입니다.

```haproxy
defaults
  mode tcp
  timeout connect 5s
  timeout client  1m
  timeout server  1m

frontend fe_tls
  bind *:443
  default_backend be_tls

backend be_tls
  server app1 10.0.0.11:443 check
  server app2 10.0.0.12:443 check
```

이 구성에서 HAProxy는 클라이언트의 TCP 연결을 받아 백엔드 서버로 전달합니다.

```text
Client --TCP--> HAProxy --TCP--> Backend
```

이때 HAProxy는 기본적으로 다음 정보만 확실하게 다룰 수 있습니다.

- 출발지 IP
- 출발지 Port
- 목적지 IP
- 목적지 Port
- 연결 성공 여부
- 연결 시간
- 전송 바이트 수
- 일부 초기 Payload
- TLS ClientHello의 SNI 일부 정보

반대로 다음 정보는 알 수 없습니다.

- HTTP Host Header
- HTTP Path
- HTTP Method
- HTTP Cookie
- HTTP Authorization Header
- HTTP Body
- HTTP Status Code

----

## 8. TCP Mode 내부 동작 흐름

TCP Mode의 동작 흐름은 다음과 같습니다.

```text
Client
  ↓
TCP 연결 요청
  ↓
HAProxy frontend 수신
  ↓
TCP ACL 평가
  ↓
backend 선택
  ↓
backend server와 TCP 연결
  ↓
양방향 TCP Stream 중계
  ↓
연결 종료
```

HTTPS를 SSL Passthrough로 처리하는 경우에는 다음과 같습니다.

```text
Client
  ↓ TCP
HAProxy
  ↓ TCP
Backend

TLS Handshake와 HTTP 암호화 데이터는 HAProxy를 통과만 한다.
```

중요한 점은 HAProxy가 TLS를 종료하지 않는다는 것입니다.

```text
Client == TLS Session == Backend
             ↑
          HAProxy는 중간에서 TCP만 중계
```

----

## 8-1. TCP Mode 내부 처리 과정을 더 자세히 보기

TCP Mode에서도 HAProxy가 단순히 아무 서버로나 넘기는 것은 아닙니다. 연결을 받은 뒤 여러 단계를 거칩니다.

```text
[1] Client가 HAProxy frontend로 TCP 연결 요청
    ↓
[2] HAProxy가 frontend bind 포트에서 연결 수락
    ↓
[3] tcp-request 규칙 또는 ACL 평가
    ↓
[4] balance 알고리즘으로 backend server 선택
    ↓
[5] HAProxy가 선택된 backend server와 TCP 연결 생성
    ↓
[6] Client ↔ Backend 사이의 TCP Stream 중계
    ↓
[7] 한쪽이 FIN/RST를 보내거나 timeout 발생 시 연결 종료
```

예를 들어 `balance roundrobin`이면 연결이 순서대로 서버에 분산됩니다.

```haproxy
backend be_tcp_app
  mode tcp
  balance roundrobin
  server app1 10.0.0.11:443 check
  server app2 10.0.0.12:443 check
```

`balance leastconn`이면 현재 연결 수가 적은 서버를 우선 선택합니다.

```haproxy
backend be_db
  mode tcp
  balance leastconn
  server db1 10.0.0.21:5432 check
  server db2 10.0.0.22:5432 check
```

`balance source`이면 Source IP를 기준으로 같은 클라이언트가 같은 서버로 가도록 만들 수 있습니다.

```haproxy
backend be_redis
  mode tcp
  balance source
  server redis1 10.0.0.31:6379 check
  server redis2 10.0.0.32:6379 check
```

중요한 점은 이 과정의 판단 기준이 대부분 **연결 단위**라는 것입니다.

```text
TCP Mode의 처리 단위 = TCP Connection
HTTP Mode의 처리 단위 = HTTP Request / Response
```

그래서 하나의 TCP 연결 안에 HTTP 요청이 여러 개 들어가더라도 TCP Mode는 각각의 HTTP 요청을 따로 분리해서 판단하지 않습니다.

----

## 9. Layer 4 Proxy

Layer 4 Proxy는 전송 계층에서 동작하는 프록시입니다.

Layer 4에서의 핵심 단위는 HTTP 요청이 아니라 TCP 연결입니다.

장점:

- HTTP가 아닌 프로토콜도 처리 가능
- PostgreSQL, Redis, MySQL, SSH, Kubernetes API Server 등에 사용 가능
- HTTPS를 복호화하지 않고 백엔드로 그대로 전달 가능
- 구조가 단순함
- L7 파싱 비용이 없음

단점:

- HTTP Path Routing 불가
- HTTP Header Rewrite 불가
- Cookie 기반 Session Persistence 불가
- HTTP Status Code 확인 불가
- Redirect 처리 불가
- X-Forwarded-For, X-Forwarded-Proto 추가 불가
- 애플리케이션 레벨 장애 감지가 제한적

----

## 10. TCP Pass Through

TCP Pass Through는 HAProxy가 TCP 연결을 거의 그대로 백엔드로 넘기는 방식입니다.

```text
Client --TCP--> HAProxy --TCP--> Backend
```

이 방식에서는 HAProxy가 애플리케이션 프로토콜을 자세히 보지 않습니다.

대표 사용 예시는 다음과 같습니다.

- PostgreSQL 5432
- Redis 6379
- MySQL 3306
- SSH 22
- Kubernetes API Server 6443
- SSL Passthrough가 필요한 HTTPS 서비스

----

## 11. TLS Handshake 흐름

HTTPS는 HTTP에 TLS 암호화를 적용한 형태입니다.

```text
HTTP + TLS = HTTPS
```

TLS 연결은 대략 다음 흐름으로 만들어집니다.

```text
Client                              Server
  |                                   |
  | -------- ClientHello ---------->  |
  |                                   |
  | <------- ServerHello ------------ |
  | <------- Certificate ------------ |
  | <------- Key Exchange ----------- |
  |                                   |
  | -------- Finished ------------->  |
  | <------- Finished --------------- |
  |                                   |
  | ===== Encrypted HTTP Data ======  |
```

TLS Handshake 이후에는 HTTP 요청과 응답이 암호화됩니다.

```text
GET /api/users HTTP/1.1
Host: api.example.com
```

위 HTTP 요청은 TLS 내부에서 암호화되어 흐릅니다.

따라서 HAProxy가 TLS를 종료하지 않으면 HTTP 내용을 볼 수 없습니다.

----

## 12. SSL Passthrough

SSL Passthrough는 TLS를 HAProxy에서 종료하지 않고 백엔드 서버까지 그대로 전달하는 방식입니다.

```text
Client == TLS == HAProxy == TLS == Backend
```

더 정확히 말하면 TLS 세션은 클라이언트와 백엔드 서버 사이에 만들어지고, HAProxy는 TCP 중계자 역할만 합니다.

```text
Client == TLS Session == Backend
             |
          HAProxy
        TCP Forwarding
```

특징:

- 인증서와 개인키는 백엔드 서버가 가집니다.
- HAProxy는 HTTP 요청 내용을 볼 수 없습니다.
- HAProxy는 Path Routing을 할 수 없습니다.
- HAProxy는 HTTP Header를 수정할 수 없습니다.
- SNI 기반 라우팅은 제한적으로 가능합니다.
- Client IP 전달은 PROXY Protocol이 필요할 수 있습니다.

사용하는 경우:

- 백엔드가 직접 TLS 인증서를 관리해야 하는 경우
- 규정상 로드밸런서에서 TLS 복호화를 하면 안 되는 경우
- Kubernetes API Server처럼 TLS 클라이언트 인증 또는 인증서 검증이 중요한 경우
- HTTP가 아닌 TLS 기반 프로토콜을 그대로 전달해야 하는 경우

----

## 13. TLS Termination

TLS Termination은 TLS 연결을 HAProxy에서 종료하는 방식입니다.

```text
Client == TLS == HAProxy --HTTP--> Backend
```

이 경우 인증서와 개인키는 HAProxy가 가집니다.

```haproxy
frontend fe_https
  bind *:443 ssl crt /etc/haproxy/certs/example.pem
  mode http
  http-request set-header X-Forwarded-Proto https
  default_backend be_app

backend be_app
  mode http
  server app1 10.0.0.11:8080 check
```

TLS Termination의 장점:

- HAProxy가 HTTP 내용을 볼 수 있습니다.
- Host Routing 가능
- Path Routing 가능
- Header Rewrite 가능
- Redirect 가능
- HTTP Health Check 가능
- X-Forwarded-For 추가 가능
- X-Forwarded-Proto 추가 가능
- WAF, Rate Limit, ACL 적용이 쉬움

단점:

- HAProxy가 인증서와 개인키를 관리해야 합니다.
- HAProxy와 Backend 사이가 평문 HTTP가 될 수 있습니다.
- 내부망도 암호화해야 하는 환경에서는 추가 구성이 필요합니다.

----

## 14. SSL Bridging / Re-encrypt

SSL Bridging 또는 Re-encrypt는 HAProxy에서 TLS를 한 번 종료한 뒤, 백엔드로 다시 TLS 연결을 맺는 방식입니다.

```text
Client == TLS == HAProxy == TLS == Backend
```

SSL Passthrough와 모양은 비슷해 보이지만 완전히 다릅니다.

### 14-1) SSL Passthrough

```text
Client == TLS Session == Backend
             |
          HAProxy는 복호화하지 않음
```

HAProxy는 HTTP 내용을 보지 못합니다.

### 14-2) SSL Bridging / Re-encrypt

```text
Client == TLS Session 1 == HAProxy == TLS Session 2 == Backend
```

HAProxy가 중간에서 TLS를 복호화하고 HTTP를 분석한 뒤, 다시 백엔드와 TLS 연결을 맺습니다.

예시:

```haproxy
frontend fe_https
  bind *:443 ssl crt /etc/haproxy/certs/example.pem
  mode http
  http-request set-header X-Forwarded-Proto https
  default_backend be_https_app

backend be_https_app
  mode http
  server app1 10.0.0.11:8443 ssl verify none check
```

장점:

- 외부 구간도 암호화
- 내부 구간도 암호화
- HAProxy에서 HTTP Routing 가능

단점:

- HAProxy 인증서 관리 필요
- 백엔드 인증서 검증 정책이 필요
- TLS 처리 비용이 증가

----

## 15. TLS 종료 위치 비교

| 방식 | 흐름 | HAProxy가 HTTP를 볼 수 있는가 | 인증서 위치 | 대표 사용처 |
|---|---|---:|---|---|
| SSL Passthrough | Client == TLS == Backend | X | Backend | Kubernetes API, 특수 TLS 서비스 |
| TLS Termination | Client == TLS == HAProxy --HTTP--> Backend | O | HAProxy | Harbor, Grafana, Keycloak, ArgoCD |
| Re-encrypt | Client == TLS == HAProxy == TLS == Backend | O | HAProxy + Backend | 내부망도 암호화해야 하는 웹 서비스 |

그림으로 보면 다음과 같습니다.

```text
[1] SSL Passthrough
Client == TLS == HAProxy == TLS == Backend
               HAProxy는 TCP만 중계

[2] TLS Termination
Client == TLS == HAProxy --HTTP--> Backend
               HAProxy가 HTTP 분석 가능

[3] Re-encrypt
Client == TLS == HAProxy == TLS == Backend
               HAProxy가 HTTP 분석 후 다시 TLS 연결
```

----

## 16. SNI란 무엇인가

SNI는 Server Name Indication의 약자입니다.

클라이언트가 TLS Handshake의 ClientHello 단계에서 접속하려는 도메인 이름을 서버에게 알려주는 TLS 확장입니다.

예를 들어 클라이언트가 다음 주소로 접속한다고 가정합니다.

```text
https://api.example.com
```

TLS Handshake 초기에 클라이언트는 다음과 같은 정보를 보냅니다.

```text
ClientHello
  SNI: api.example.com
```

서버는 이 값을 보고 어떤 인증서를 사용할지 결정할 수 있습니다.

----

## 17. 왜 TCP Mode에서도 SNI를 읽을 수 있는가

이 부분이 SSL Passthrough에서 가장 많이 헷갈리는 부분입니다.

질문은 다음과 같습니다.

```text
TLS는 암호화인데 왜 HAProxy가 SNI는 읽을 수 있는가?
```

이유는 SNI가 TLS Handshake의 초기 메시지인 ClientHello에 포함되기 때문입니다.

흐름은 다음과 같습니다.

```text
TCP Connection 생성
  ↓
ClientHello 전송
  ↓
ClientHello 안에 SNI 포함
  ↓
이 시점에서는 SNI가 평문으로 보일 수 있음
  ↓
TLS Handshake 완료
  ↓
이후 HTTP 데이터는 암호화됨
```

그래서 HAProxy는 TCP Mode에서도 초기 Payload를 잠깐 검사해서 SNI를 기준으로 백엔드를 나눌 수 있습니다.

```haproxy
frontend fe_tls_passthrough
  bind *:443
  mode tcp

  tcp-request inspect-delay 5s
  tcp-request content accept if { req.ssl_hello_type 1 }

  acl sni_api req.ssl_sni -i api.example.com
  acl sni_admin req.ssl_sni -i admin.example.com

  use_backend be_api_tls if sni_api
  use_backend be_admin_tls if sni_admin
  default_backend be_default_tls
```

단, SNI 기반 라우팅은 Host Header 기반 라우팅과 다릅니다.

| 구분 | SNI | HTTP Host Header |
|---|---|---|
| 위치 | TLS ClientHello | HTTP Header |
| 암호화 여부 | 일반적으로 초기 Handshake에서 노출 | HTTPS에서는 암호화됨 |
| TCP Mode 사용 가능 | O | X |
| HTTP Mode 사용 가능 | TLS 종료 후 O | O |
| Path 정보 포함 | X | X |

주의할 점도 있습니다.

- SNI에는 Path가 없습니다.
- SNI에는 HTTP Method가 없습니다.
- SNI에는 Cookie가 없습니다.
- SNI에는 Authorization Header가 없습니다.
- SNI는 도메인 기반 분기에만 제한적으로 사용할 수 있습니다.
- ECH(Encrypted Client Hello)가 활성화된 환경에서는 SNI 가시성이 달라질 수 있습니다.

----

## 18. TCP Mode에서 가능한 기능

`mode tcp`는 HTTP 기능을 사용할 수 없지만 아무것도 못 하는 것은 아닙니다.

가능한 기능은 다음과 같습니다.

| 기능 | 설명 |
|---|---|
| TCP Load Balancing | TCP 연결을 여러 백엔드로 분산 |
| SNI Routing | TLS ClientHello의 SNI를 기준으로 분기 |
| TCP Health Check | 포트 연결 가능 여부 확인 |
| SSL Hello Check | TLS Handshake 수준의 확인 |
| PROXY Protocol | 원본 Client IP를 백엔드에 전달 |
| Source Hash | Client IP 기반으로 같은 서버에 연결 |
| Least Connection | 연결 수가 적은 서버로 분산 |
| Connection Limit | 동시 연결 수 제한 |
| Timeout 설정 | connect/client/server timeout 제어 |
| TCP Logging | 연결 시간, 바이트 수, 종료 상태 기록 |
| Stick Table 일부 | Source IP 기반 추적/제한 |
| Rate Limit 일부 | 연결 수 또는 접속 빈도 기반 제한 |
| Connection Tracking | 연결 상태 기반 제어 |

예시:

```haproxy
backend be_postgres
  mode tcp
  balance leastconn
  server pg1 10.0.0.21:5432 check
  server pg2 10.0.0.22:5432 check
```

Source Hash 예시:

```haproxy
backend be_redis
  mode tcp
  balance source
  server redis1 10.0.0.31:6379 check
  server redis2 10.0.0.32:6379 check
```

----

## 19. TCP Mode에서 절대 못하는 것

TCP Mode에서는 HTTP를 파싱하지 않기 때문에 다음 기능은 사용할 수 없습니다.

| 항목 | TCP Mode 가능 여부 | 이유 |
|---|---:|---|
| HTTP Host Header 확인 | X | HTTP Header를 파싱하지 않음 |
| Path Routing | X | URL Path는 HTTP 요청 내부 정보 |
| Cookie 확인 | X | Cookie는 HTTP Header |
| Authorization Header 확인 | X | Authorization은 HTTP Header |
| Request Body 확인 | X | Body는 HTTP 메시지 내부 |
| HTTP Status Code 확인 | X | Status Code는 HTTP Response 정보 |
| HTTP Redirect | X | Redirect는 HTTP Response 생성 필요 |
| Header Rewrite | X | Header를 읽고 수정하지 않음 |
| X-Forwarded-For 추가 | X | HTTP Header 추가 기능 |
| X-Forwarded-Proto 추가 | X | HTTP Header 추가 기능 |
| Compression | X | HTTP Response 처리 필요 |

즉, 다음 설정은 `mode tcp`에서 기대하면 안 됩니다.

```haproxy
acl is_api path_beg /api
http-request set-header X-Forwarded-Proto https
http-request redirect scheme https
```

이런 기능이 필요하면 `mode http`와 TLS Termination을 고려해야 합니다.

----

## 20. TCP Health Check

TCP Health Check는 백엔드 포트에 TCP 연결이 가능한지 확인합니다.

```haproxy
backend be_tls
  mode tcp
  server app1 10.0.0.11:443 check
```

이 방식은 포트가 열려 있으면 정상으로 판단할 수 있습니다.

하지만 애플리케이션이 내부적으로 500 에러를 반환해도 TCP 포트가 열려 있으면 정상으로 보일 수 있습니다.

```text
TCP 연결 성공
  ↓
HAProxy는 정상으로 판단
  ↓
하지만 실제 HTTP /healthz는 500일 수 있음
```

따라서 HTTP 서비스라면 가능하면 HTTP Health Check를 사용하는 것이 더 정확합니다.

----

## 21. SSL Check / SSL Hello Check

TLS 서비스에서는 단순 TCP 연결뿐 아니라 TLS Handshake가 가능한지도 확인할 수 있습니다.

예시:

```haproxy
backend be_tls_app
  mode tcp
  option ssl-hello-chk
  server app1 10.0.0.11:443 check
```

또는 서버 라인에서 SSL 체크를 사용할 수 있습니다.

```haproxy
backend be_https
  mode tcp
  server app1 10.0.0.11:443 check ssl verify none
```

차이는 다음과 같습니다.

| Check | 확인 수준 |
|---|---|
| TCP Check | 포트 연결 가능 여부 |
| SSL Hello Check | TLS Handshake 가능 여부 |
| HTTP Check | HTTP 요청에 대한 응답 상태 |
| HTTPS Check | TLS 위에서 HTTP Health Check |
| Agent Check | 백엔드 Agent가 상태를 직접 보고 |
| External Check | 외부 스크립트로 상태 확인 |

----

## 22. PROXY Protocol

TCP Mode에서 가장 중요한 기능 중 하나가 PROXY Protocol입니다.

TCP Mode에서는 HTTP Header를 추가할 수 없습니다. 그래서 `X-Forwarded-For`로 Client IP를 전달할 수 없습니다.

대신 TCP 연결의 맨 앞에 원본 Client 정보를 실어 보내는 방식이 PROXY Protocol입니다.

```text
Client IP: 1.1.1.1
  ↓
HAProxy
  ↓ PROXY Protocol Header + TCP Stream
Backend
```

예시:

```haproxy
backend be_k8s_api
  mode tcp
  server api1 10.0.0.11:6443 send-proxy check
```

백엔드 애플리케이션이 PROXY Protocol을 이해해야 합니다.

주의:

- 백엔드가 PROXY Protocol을 지원하지 않으면 연결이 깨질 수 있습니다.
- PROXY Protocol은 HTTP 전용이 아닙니다.
- TCP Mode와 HTTP Mode 모두에서 사용할 수 있지만, TCP Mode에서 Client IP 전달 목적으로 특히 중요합니다.

----

## 23. TCP Mode timeout

TCP Mode에서 기본적으로 중요한 timeout은 다음과 같습니다.

```haproxy
defaults
  mode tcp
  timeout connect 5s
  timeout client  1m
  timeout server  1m
```

| Timeout | 의미 |
|---|---|
| timeout connect | HAProxy가 백엔드 서버에 연결할 때 기다리는 시간 |
| timeout client | 클라이언트 쪽에서 데이터가 없을 때 연결 유지 시간 |
| timeout server | 서버 쪽에서 데이터가 없을 때 연결 유지 시간 |

예를 들어 SSH, DB, Redis처럼 연결이 오래 유지되는 서비스는 timeout을 너무 짧게 잡으면 연결이 끊길 수 있습니다.

```haproxy
backend be_ssh
  mode tcp
  timeout server 1h
  server ssh1 10.0.0.50:22 check
```

----

## 24. TCP Mode Logging

TCP Log는 HTTP 요청 정보가 아니라 연결 정보를 중심으로 기록합니다.

```haproxy
defaults
  mode tcp
  option tcplog
```

주로 확인 가능한 정보:

- Client IP
- Frontend
- Backend
- Server
- 연결 시간
- Queue 시간
- 전송 바이트 수
- 종료 상태

TCP Mode에서는 다음 정보가 없습니다.

- HTTP Method
- URL Path
- HTTP Status Code
- User-Agent
- Request Header

따라서 TCP Mode 로그만 보고 `/api/users` 요청이 실패했는지 확인할 수 없습니다.

----

## 25. mode http란 무엇인가

`mode http`는 HAProxy가 Layer 7에서 HTTP 메시지를 해석하는 방식입니다.

```haproxy
frontend fe_http
  bind *:80
  mode http

  acl path_api path_beg /api
  use_backend be_api if path_api
  default_backend be_web
```

HTTP Mode에서는 HAProxy가 다음 정보를 파싱할 수 있습니다.

- Method
- Path
- Query String
- Host Header
- Request Header
- Cookie
- HTTP Version
- Response Status Code
- Response Header

그래서 다음 기능이 가능해집니다.

- Host Routing
- Path Routing
- Header Rewrite
- Redirect
- Cookie 기반 Sticky Session
- HTTP Health Check
- Rate Limit
- Compression
- gRPC 처리
- WebSocket Upgrade 처리

----

## 26. HTTP Mode 내부 동작 흐름

HTTP Mode의 내부 흐름은 TCP Mode보다 복잡합니다.

```text
Client
  ↓
TCP Connection 생성
  ↓
TLS Termination (HTTPS인 경우)
  ↓
HTTP Parser
  ↓
Request Line 분석
  ↓
Header Parsing
  ↓
ACL 평가
  ↓
Backend 선택
  ↓
Request Header Rewrite
  ↓
Backend로 요청 전달
  ↓
Response 수신
  ↓
Response Status / Header 분석
  ↓
Response Header Rewrite
  ↓
Client로 응답 반환
```

HTTPS에서 HTTP Mode를 사용하려면 일반적으로 HAProxy가 TLS를 종료해야 합니다.

```text
Client == TLS == HAProxy --HTTP--> Backend
```

또는 Re-encrypt를 사용할 수 있습니다.

```text
Client == TLS == HAProxy == TLS == Backend
```

----

## 26-1. HTTP Mode 내부 처리 과정을 더 자세히 보기

HTTP Mode에서는 HAProxy가 TCP 연결 위에 실려 있는 HTTP 메시지를 파싱합니다. 그래서 TCP Mode보다 처리 단계가 많습니다.

```text
[1] Client가 HAProxy로 TCP 연결
    ↓
[2] HTTPS라면 HAProxy에서 TLS Termination
    ↓
[3] HTTP Parser가 Request Line 분석
    ↓
[4] HTTP Parser가 Header 분석
    ↓
[5] 필요하면 Cookie, Host, Path, Method, Query String 평가
    ↓
[6] ACL 조건 평가
    ↓
[7] use_backend / default_backend 기준으로 Backend 선택
    ↓
[8] http-request 규칙 적용
    - Header 추가
    - Header 삭제
    - Path Rewrite
    - Redirect
    - Rate Limit
    ↓
[9] Backend Server로 HTTP 요청 전달
    ↓
[10] Backend Response 수신
    ↓
[11] Response Status Code / Header 분석
    ↓
[12] http-response 규칙 적용
    - Response Header 추가
    - 보안 Header 추가
    - Cache-Control 조정
    ↓
[13] Client로 응답 반환
```

그림으로 단순화하면 다음과 같습니다.

```text
Client
  ↓ TCP Connection
HAProxy
  ↓ TLS Termination
HTTP Parser
  ↓ Request Line / Header / Cookie 분석
ACL 평가
  ↓
Backend 선택
  ↓
Request Rewrite
  ↓
Backend
  ↓
Response
  ↓
Response Header Rewrite
  ↓
Client
```

이 구조 때문에 HTTP Mode에서는 다음 작업이 가능합니다.

- `/api` 요청은 API 서버로 전달
- `/static` 요청은 정적 파일 서버로 전달
- `Host: admin.example.com` 요청은 Admin 서버로 전달
- `X-Forwarded-Proto: https` Header 추가
- `Set-Cookie` 기반 Sticky Session 구성
- 응답 Status Code가 200인지 확인
- 80번 HTTP 요청을 443 HTTPS로 Redirect

반대로 말하면, 이런 기능이 필요하다면 TCP Mode가 아니라 HTTP Mode를 사용해야 합니다.

----

## 27. HTTP Parsing

HTTP Parsing은 HAProxy가 HTTP 메시지를 읽고 구조를 이해한다는 뜻입니다.

예시 요청:

```http
POST /api/users?id=10 HTTP/1.1
Host: app.example.com
Content-Type: application/json
Authorization: Bearer xxx
Cookie: SESSIONID=abc
```

HAProxy는 다음 기준으로 ACL을 만들 수 있습니다.

```haproxy
acl is_api path_beg /api
acl is_app hdr(host) -i app.example.com
acl is_post method POST
acl has_auth req.hdr(Authorization) -m found
```

이것이 TCP Mode와 HTTP Mode의 가장 큰 차이입니다.

----

## 27-1. HTTP Parser가 실제로 보는 구조

HTTP 요청은 크게 세 부분으로 나눌 수 있습니다.

```http
POST /api/users?id=10 HTTP/1.1
Host: api.example.com
User-Agent: curl/8.0
Content-Type: application/json
Authorization: Bearer xxx
Cookie: SESSIONID=abc

{"name":"eunbyul"}
```

이를 구조로 나누면 다음과 같습니다.

| 영역 | 예시 | HAProxy HTTP Mode에서 활용 가능 여부 | 대표 기능 |
|---|---|---:|---|
| Request Line | `POST /api/users?id=10 HTTP/1.1` | O | Method Routing, Path Routing |
| Method | `POST` | O | POST 요청만 별도 처리 |
| Path | `/api/users` | O | `/api`는 API backend로 전달 |
| Query String | `id=10` | O | 특정 query 포함 요청 분기 |
| Header | `Host`, `Authorization`, `Cookie` | O | Host Routing, 인증 Header 검사 |
| Body | `{"name":"eunbyul"}` | 제한적 | 일반 라우팅 기준으로는 보통 사용하지 않음 |

응답도 마찬가지로 구조를 가집니다.

```http
HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: SESSIONID=abc; Path=/; Secure

{"status":"ok"}
```

HTTP Mode에서는 응답의 Status Code와 Header도 다룰 수 있습니다.

```haproxy
http-response set-header X-Frame-Options DENY
http-response set-header X-Content-Type-Options nosniff
```

즉, HTTP Mode는 요청 방향뿐 아니라 응답 방향에서도 L7 제어가 가능합니다.

----

## 27-2. TCP Stream과 HTTP Request/Response의 처리 단위 차이

TCP는 Stream 기반입니다.

```text
TCP Stream
Byte 1 → Byte 2 → Byte 3 → Byte 4 → Byte 5 → ...
```

HTTP는 Request/Response 구조를 가집니다.

```text
HTTP Request 1  →  HTTP Response 1
HTTP Request 2  →  HTTP Response 2
HTTP Request 3  →  HTTP Response 3
```

HTTP Keep-Alive가 켜져 있으면 하나의 TCP 연결 안에서 여러 HTTP 요청과 응답이 오갈 수 있습니다.

```text
하나의 TCP Connection
  ↓
Request 1  → Response 1
Request 2  → Response 2
Request 3  → Response 3
```

TCP Mode에서는 위 전체를 하나의 연결 흐름으로 봅니다.

```text
TCP Mode 관점:
Client와 Backend 사이에 연결 하나가 있다.
```

HTTP Mode에서는 그 안의 HTTP 요청들을 각각 분석할 수 있습니다.

```text
HTTP Mode 관점:
이 연결 안에 /api 요청, /login 요청, /static 요청이 각각 들어 있다.
```

이 차이 때문에 다음 결과가 생깁니다.

| 상황 | TCP Mode | HTTP Mode |
|---|---|---|
| 같은 TCP 연결 안의 여러 HTTP 요청 구분 | X | O |
| 요청마다 다른 backend 선택 | X | O |
| 요청 Path 기준 라우팅 | X | O |
| 응답 Status Code 기반 판단 | X | O |

따라서 “연결을 분산할 것인지”, “요청을 분석해서 분산할 것인지”가 mode 선택의 핵심입니다.

----

## 28. HTTP Mode에서 가능한 기능

| 기능 | 설명 |
|---|---|
| Host Routing | Host Header 기준 분기 |
| Path Routing | URL Path 기준 분기 |
| Method Routing | GET/POST/PUT/DELETE 기준 분기 |
| Header Add | 요청/응답 헤더 추가 |
| Header Delete | 요청/응답 헤더 삭제 |
| Header Rewrite | 헤더 값 변경 |
| URI Rewrite | URI 또는 Path 변경 |
| Redirect | HTTP 301/302/308 응답 생성 |
| Compression | 응답 압축 |
| Cookie Insert | Sticky Session용 Cookie 삽입 |
| Cache-Control 조정 | 응답 캐시 정책 설정 |
| Authentication 연동 | 기본 인증 또는 ACL 기반 차단 |
| Rate Limit | 요청 빈도 제한 |
| Stick Table | IP, Header, Cookie 기반 추적 |
| HTTP Health Check | `/healthz` 등으로 앱 정상성 확인 |
| Response Header 제어 | 보안 헤더 추가 가능 |
| gRPC | HTTP/2 기반 gRPC 처리 가능 |
| WebSocket | Upgrade 이후 tunnel 처리 |

----

## 29. Host Routing

Host Routing은 도메인 이름 기준 라우팅입니다.

```haproxy
frontend fe_https
  bind *:443 ssl crt /etc/haproxy/certs/example.pem
  mode http

  acl host_api hdr(host) -i api.example.com
  acl host_admin hdr(host) -i admin.example.com

  use_backend be_api if host_api
  use_backend be_admin if host_admin
  default_backend be_web
```

Kubernetes Ingress의 Host-based Routing과 같은 개념입니다.

```yaml
rules:
  - host: api.example.com
  - host: admin.example.com
```

----

## 30. Path Routing

Path Routing은 URL 경로 기준 라우팅입니다.

```haproxy
frontend fe_web
  bind *:80
  mode http

  acl path_api path_beg /api
  acl path_static path_beg /static

  use_backend be_api if path_api
  use_backend be_static if path_static
  default_backend be_frontend
```

흐름:

```text
/api/users     → be_api
/static/app.js → be_static
/              → be_frontend
```

Path Routing은 HTTP 요청을 파싱해야 하므로 TCP Mode에서는 불가능합니다.

----

## 31. Header Rewrite

Header Rewrite는 요청 또는 응답 Header를 추가, 삭제, 변경하는 기능입니다.

```haproxy
http-request set-header X-Forwarded-Proto https
http-request set-header X-Forwarded-Port 443
http-request add-header X-Request-ID %[unique-id]
http-response set-header X-Frame-Options DENY
```

사용 사례:

- 원래 요청 프로토콜 전달
- 클라이언트 IP 전달
- Request ID 부여
- 보안 헤더 추가
- 백엔드가 HTTPS 여부를 인식하도록 함

----

## 32. X-Forwarded-For

`X-Forwarded-For`는 원본 Client IP를 백엔드 애플리케이션에 전달하기 위한 HTTP Header입니다.

```haproxy
frontend fe_http
  mode http
  option forwardfor
```

또는 직접 설정할 수 있습니다.

```haproxy
http-request set-header X-Forwarded-For %[src]
```

왜 HTTP Mode에서만 가능한가?

`X-Forwarded-For`는 HTTP Header입니다. TCP Mode에서는 HTTP Header를 파싱하거나 추가하지 않습니다.

TCP Mode에서 Client IP를 전달해야 하면 PROXY Protocol을 고려해야 합니다.

----

## 33. X-Forwarded-Proto

`X-Forwarded-Proto`는 클라이언트가 원래 어떤 프로토콜로 접근했는지 백엔드에 알려주는 Header입니다.

```haproxy
http-request set-header X-Forwarded-Proto https if { ssl_fc }
```

이 설정이 없으면 백엔드 애플리케이션이 자신에게 들어온 요청을 HTTP로 인식할 수 있습니다.

그 결과 다음 문제가 발생할 수 있습니다.

```text
Client → https://app.example.com
HAProxy → Backend로 HTTP 전달
Backend는 "현재 HTTP로 들어왔다"고 판단
Backend가 HTTPS로 Redirect
Client는 다시 HTTPS 요청
반복
```

즉, 무한 Redirect가 발생할 수 있습니다.

----

## 34. Redirect

HTTP Mode에서는 HAProxy가 직접 Redirect 응답을 만들 수 있습니다.

HTTP에서 HTTPS로 Redirect:

```haproxy
frontend fe_http
  bind *:80
  mode http
  http-request redirect scheme https code 301 unless { ssl_fc }
```

Host 변경 Redirect:

```haproxy
http-request redirect location https://www.example.com%[path] code 301 if { hdr(host) -i example.com }
```

TCP Mode에서는 Redirect가 불가능합니다. Redirect는 HTTP 응답이기 때문입니다.

----

## 35. URI Rewrite / Set Path

HTTP Mode에서는 요청 Path를 바꿔서 백엔드로 보낼 수 있습니다.

```haproxy
http-request set-path %[path,regsub(^/api/,/)] if { path_beg /api/ }
```

예시:

```text
Client Request : /api/users
Backend Request: /users
```

이 기능은 API Gateway 또는 Reverse Proxy에서 자주 사용됩니다.

----

## 36. Cookie 기반 Sticky Session

HTTP Mode에서는 Cookie를 기준으로 같은 클라이언트를 같은 서버로 보낼 수 있습니다.

```haproxy
backend be_app
  mode http
  balance roundrobin
  cookie SERVERID insert indirect nocache
  server app1 10.0.0.11:8080 check cookie app1
  server app2 10.0.0.12:8080 check cookie app2
```

TCP Mode에서는 Cookie를 읽을 수 없으므로 Cookie 기반 Sticky Session은 불가능합니다.

대신 Source IP 기반 분산을 사용할 수 있습니다.

```haproxy
backend be_tcp
  mode tcp
  balance source
```

----

## 37. HTTP Health Check

HTTP Health Check는 애플리케이션이 실제로 정상 응답하는지 확인합니다.

```haproxy
backend be_app
  mode http
  option httpchk GET /healthz
  http-check expect status 200
  server app1 10.0.0.11:8080 check
  server app2 10.0.0.12:8080 check
```

HTTP Health Check는 TCP Check보다 더 정확합니다.

| Check | 확인 가능 여부 |
|---|---|
| 포트 열림 | O |
| HTTP 응답 여부 | O |
| Status Code | O |
| `/healthz` 정상 여부 | O |
| DB 연결 상태까지 반영한 Health | 애플리케이션 구현에 따라 가능 |

HTTPS 백엔드 체크 예시:

```haproxy
backend be_https_app
  mode http
  option httpchk GET /healthz
  http-check expect status 200
  server app1 10.0.0.11:8443 ssl verify none check
```

----

## 37-1. Health Check 종류별 비교

HAProxy의 Health Check는 단순히 서버가 살아 있는지 확인하는 기능이 아닙니다. 어떤 계층까지 확인하느냐에 따라 정확도가 달라집니다.

| Check 종류 | 확인 계층 | 확인 내용 | 장점 | 한계 |
|---|---|---|---|---|
| TCP Check | L4 | 포트 연결 가능 여부 | 단순하고 빠름 | 앱 정상성 확인 불가 |
| SSL Check | TLS | TLS Handshake 가능 여부 | TLS 서비스 확인 가능 | HTTP 상태 확인 불가 |
| SSL Hello Check | TLS 초기 | SSL/TLS Hello 응답 확인 | TCP보다 한 단계 정확 | 최신 TLS 검증에는 제한 가능 |
| HTTP Check | L7 | HTTP 요청/응답 확인 | 앱 상태 확인 가능 | HTTP 서비스에만 적합 |
| HTTPS Check | TLS + HTTP | TLS 연결 후 HTTP Health 확인 | HTTPS 앱 상태 확인 가능 | 인증서 검증 정책 필요 |
| Agent Check | 별도 Agent | 서버가 자신의 상태를 직접 보고 | 앱 내부 상태 반영 가능 | Agent 구현 필요 |
| External Check | 외부 스크립트 | Shell Script 등으로 자유 검사 | 복잡한 조건 가능 | 운영 복잡도 증가 |

### TCP Check

```haproxy
backend be_tcp
  mode tcp
  server app1 10.0.0.11:443 check
```

의미:

```text
10.0.0.11:443 포트에 TCP 연결이 가능한가?
```

### SSL/TLS Check

```haproxy
backend be_tls
  mode tcp
  server app1 10.0.0.11:443 check ssl verify none
```

의미:

```text
TCP 연결뿐 아니라 TLS Handshake까지 가능한가?
```

### HTTP Check

```haproxy
backend be_app
  mode http
  option httpchk GET /healthz
  http-check expect status 200
  server app1 10.0.0.11:8080 check
```

의미:

```text
/healthz로 HTTP 요청을 보냈을 때 200이 오는가?
```

### HTTPS Check

```haproxy
backend be_https_app
  mode http
  option httpchk GET /healthz
  http-check expect status 200
  server app1 10.0.0.11:8443 ssl verify none check
```

의미:

```text
TLS로 백엔드에 연결한 뒤 /healthz HTTP Check를 수행한다.
```

실무 기준으로는 다음처럼 선택하면 됩니다.

```text
비HTTP TCP 서비스          → TCP Check
TLS 기반 비HTTP 서비스     → SSL Check
일반 HTTP 웹 서비스        → HTTP Check
HTTPS 백엔드 웹 서비스     → HTTPS Check
복잡한 내부 상태 반영 필요 → Agent Check 또는 External Check
```

----

## 38. Stick Table과 Rate Limit

HTTP Mode에서는 요청 단위로 Rate Limit을 적용하기 쉽습니다.

```haproxy
frontend fe_api
  bind *:443 ssl crt /etc/haproxy/certs/example.pem
  mode http

  stick-table type ip size 100k expire 10m store http_req_rate(10s)
  http-request track-sc0 src
  http-request deny if { sc_http_req_rate(0) gt 100 }

  default_backend be_api
```

의미:

```text
같은 Source IP에서 10초 동안 100개 초과 요청이 오면 차단
```

TCP Mode에서도 연결 수 기반 제한은 가능하지만, HTTP 요청 단위 제한은 어렵습니다.

----

## 39. Compression

HTTP Mode에서는 응답 압축을 설정할 수 있습니다.

```haproxy
backend be_web
  mode http
  compression algo gzip
  compression type text/html text/plain text/css application/json application/javascript
```

Compression은 HTTP Response Body를 다뤄야 하므로 TCP Mode에서는 사용할 수 없습니다.

----

## 40. HTTP/2

HTTP/2는 HTTP/1.1과 달리 Binary Frame 기반으로 동작합니다.

HAProxy는 TLS Termination 시 ALPN을 통해 HTTP/2를 받을 수 있습니다.

```haproxy
frontend fe_https
  bind *:443 ssl crt /etc/haproxy/certs/example.pem alpn h2,http/1.1
  mode http
  default_backend be_app
```

HTTP/2를 사용하는 경우에도 HAProxy가 HTTP Mode로 동작하면 HTTP 요청의 의미를 처리할 수 있습니다.

단, HTTP/2는 내부적으로 Frame 구조를 사용하므로 단순 텍스트 HTTP/1.1과 구현 방식은 다릅니다.

----

## 40-1. HTTP/2의 Binary Frame과 Multiplexing

HTTP/1.1은 사람이 읽을 수 있는 텍스트 기반 메시지에 가깝습니다.

```http
GET /api/users HTTP/1.1
Host: api.example.com
```

반면 HTTP/2는 Binary Frame 기반입니다. 즉, 요청과 응답이 텍스트 줄 단위가 아니라 여러 종류의 Frame으로 나뉘어 전송됩니다.

```text
HTTP/2 Connection
  ├─ Stream 1
  │   ├─ HEADERS Frame
  │   └─ DATA Frame
  ├─ Stream 3
  │   ├─ HEADERS Frame
  │   └─ DATA Frame
  └─ Stream 5
      ├─ HEADERS Frame
      └─ DATA Frame
```

HTTP/2의 중요한 특징은 Multiplexing입니다.

```text
하나의 TCP Connection 안에서
여러 HTTP 요청/응답 Stream이 동시에 흐를 수 있다.
```

HTTP/1.1 Keep-Alive는 하나의 TCP 연결을 재사용할 수 있지만, 요청/응답 처리 방식에 제약이 있습니다. HTTP/2는 하나의 연결 안에서 여러 Stream을 병렬적으로 처리할 수 있습니다.

```text
HTTP/1.1 Keep-Alive
Request 1 → Response 1 → Request 2 → Response 2

HTTP/2 Multiplexing
Request Stream 1 ┐
Request Stream 3 ├─ 동시에 같은 TCP 연결에서 처리
Request Stream 5 ┘
```

HAProxy에서 HTTP/2를 사용하려면 보통 TLS Termination 지점에서 ALPN을 설정합니다.

```haproxy
frontend fe_https
  bind *:443 ssl crt /etc/haproxy/certs/example.pem alpn h2,http/1.1
  mode http
  default_backend be_app
```

주의할 점은 HTTP/2도 결국 TCP 위에서 동작한다는 것입니다.

```text
HTTP/2
  ↓
TLS
  ↓
TCP
```

따라서 TCP Mode로도 HTTP/2 트래픽을 단순 중계할 수는 있습니다. 하지만 그 경우 HAProxy는 HTTP/2 Frame의 의미를 해석하지 않고 TCP Stream으로만 전달합니다.

HTTP/2 기반 라우팅, Header 처리, gRPC 제어가 필요하면 HTTP Mode와 TLS Termination 또는 Re-encrypt 구성을 고려해야 합니다.

----

## 41. gRPC

gRPC는 일반적으로 HTTP/2 위에서 동작합니다.

```text
gRPC
  ↓
HTTP/2
  ↓
TLS
  ↓
TCP
```

HAProxy에서 gRPC를 처리할 때는 HTTP/2와 timeout을 함께 고려해야 합니다.

예시:

```haproxy
frontend fe_grpc
  bind *:443 ssl crt /etc/haproxy/certs/grpc.pem alpn h2
  mode http
  default_backend be_grpc

backend be_grpc
  mode http
  server grpc1 10.0.0.21:50051 proto h2 check
```

ArgoCD는 gRPC 관련 동작이 포함될 수 있으므로 Ingress나 HAProxy 설정에서 HTTP/2, gRPC-Web, TLS 종료 위치를 같이 고려해야 합니다.

----

## 42. WebSocket

WebSocket은 처음에는 HTTP 요청으로 시작합니다.

```http
GET /ws HTTP/1.1
Host: app.example.com
Upgrade: websocket
Connection: Upgrade
```

서버가 Upgrade를 승인하면 이후에는 WebSocket 터널처럼 동작합니다.

HAProxy에서는 보통 HTTP Mode에서 처리하며, 긴 연결을 위해 `timeout tunnel`을 설정합니다.

```haproxy
defaults
  mode http
  timeout connect 5s
  timeout client  1m
  timeout server  1m
  timeout http-request 10s
  timeout http-keep-alive 10s
  timeout tunnel 1h
```

WebSocket 장애에서 자주 보는 원인은 다음과 같습니다.

- `timeout tunnel`이 너무 짧음
- Upgrade Header가 백엔드까지 전달되지 않음
- 중간 Proxy가 HTTP/1.1 Upgrade를 제대로 처리하지 않음
- TLS 종료 위치가 잘못됨

----

## 42-1. WebSocket에서 timeout tunnel이 중요한 이유

WebSocket은 처음에는 HTTP 요청으로 시작하지만, Upgrade 이후에는 일반적인 HTTP 요청/응답 패턴과 다르게 동작합니다.

초기 요청은 HTTP입니다.

```http
GET /ws HTTP/1.1
Host: app.example.com
Upgrade: websocket
Connection: Upgrade
```

서버가 Upgrade를 승인하면 이후 연결은 장시간 유지되는 양방향 통신 채널처럼 동작합니다.

```text
Client  <================ WebSocket Tunnel ================>  Backend
```

이 상태에서는 일반적인 HTTP 요청처럼 짧게 요청하고 응답받은 뒤 끝나는 구조가 아닙니다.

그래서 `timeout client` 또는 `timeout server`만 짧게 잡혀 있으면 WebSocket 연결이 중간에 끊길 수 있습니다. HAProxy에서는 Upgrade 이후 터널 상태의 연결을 위해 `timeout tunnel`을 별도로 설정하는 것이 일반적입니다.

```haproxy
defaults
  mode http
  timeout connect 5s
  timeout client  1m
  timeout server  1m
  timeout tunnel  1h
```

정리하면 다음과 같습니다.

| 구간 | 설명 | 관련 timeout |
|---|---|---|
| WebSocket Upgrade 전 | 일반 HTTP 요청 | `timeout http-request`, `timeout client`, `timeout server` |
| WebSocket Upgrade 후 | 장시간 양방향 터널 | `timeout tunnel` |

WebSocket 장애를 볼 때는 다음을 확인해야 합니다.

- Upgrade Header가 유지되는가
- `Connection: Upgrade`가 백엔드까지 전달되는가
- `timeout tunnel`이 충분히 긴가
- 중간 Proxy나 Ingress가 WebSocket을 지원하는가
- Backend 애플리케이션의 idle timeout이 너무 짧지 않은가

----

## 43. HTTP Mode timeout

HTTP Mode에서는 TCP timeout 외에도 HTTP 전용 timeout을 고려해야 합니다.

```haproxy
defaults
  mode http
  timeout connect 5s
  timeout client  1m
  timeout server  1m
  timeout http-request 10s
  timeout http-keep-alive 10s
  timeout tunnel 1h
```

| Timeout | 설명 |
|---|---|
| timeout connect | 백엔드 연결 대기 시간 |
| timeout client | 클라이언트 측 비활성 시간 |
| timeout server | 서버 측 비활성 시간 |
| timeout http-request | HTTP 요청 Header를 모두 받을 때까지 기다리는 시간 |
| timeout http-keep-alive | Keep-Alive 연결 유지 시간 |
| timeout tunnel | WebSocket 등 터널 상태 연결 유지 시간 |

HTTP Mode에서는 요청 단위 처리와 연결 재사용이 있기 때문에 TCP Mode보다 timeout 종류를 더 세밀하게 봐야 합니다.

----

## 44. HTTP Mode Logging

HTTP Mode에서는 HTTP 요청/응답 정보를 로그에 남길 수 있습니다.

```haproxy
defaults
  mode http
  option httplog
```

HTTP Log에서 볼 수 있는 정보:

- Client IP
- Frontend
- Backend
- Server
- HTTP Method
- Path
- Status Code
- Response Time
- Bytes
- Termination State

TCP Log와 비교하면 다음과 같습니다.

| 항목 | TCP Log | HTTP Log |
|---|---:|---:|
| Client IP | O | O |
| Backend Server | O | O |
| 연결 시간 | O | O |
| 전송 바이트 | O | O |
| HTTP Method | X | O |
| Path | X | O |
| Status Code | X | O |
| Header 기반 분석 | X | O |

----

## 45. TCP Mode와 HTTP Mode의 핵심 차이

이 문서의 핵심입니다.

`mode tcp`와 `mode http`의 차이는 다음과 같습니다.

```text
TCP Mode
  - TCP 연결을 처리한다.
  - HTTP 내용을 보지 않는다.
  - 연결 단위 Load Balancing에 적합하다.
  - SSL Passthrough에 적합하다.
  - 비HTTP TCP 서비스에 적합하다.

HTTP Mode
  - HTTP 요청과 응답을 처리한다.
  - Host, Path, Header, Cookie, Status Code를 볼 수 있다.
  - 웹 서비스 Reverse Proxy에 적합하다.
  - TLS Termination과 함께 자주 사용한다.
  - Kubernetes Ingress와 유사한 L7 라우팅이 가능하다.
```

즉, 다음처럼 기억하면 됩니다.

```text
TCP Mode는 "연결을 어디로 보낼지" 결정한다.
HTTP Mode는 "요청 내용을 보고 어디로 보낼지" 결정한다.
```

----

## 46. TCP Mode와 HTTP Mode 기능 비교

| 기능 | TCP Mode | HTTP Mode |
|---|---:|---:|
| TCP 연결 중계 | O | O |
| HTTP Parsing | X | O |
| Host Header Routing | X | O |
| SNI Routing | O | O |
| Path Routing | X | O |
| Method Routing | X | O |
| Header Rewrite | X | O |
| Cookie 기반 Sticky Session | X | O |
| Source IP 기반 분산 | O | O |
| HTTP Status Code 확인 | X | O |
| HTTP Redirect | X | O |
| Compression | X | O |
| X-Forwarded-For | X | O |
| X-Forwarded-Proto | X | O |
| PROXY Protocol | O | O |
| TCP Health Check | O | O |
| HTTP Health Check | 제한적 | O |
| SSL Passthrough | O | X |
| TLS Termination | X | O |
| Re-encrypt | X | O |
| gRPC 처리 | 제한적 TCP 중계 | O |
| WebSocket 처리 | 제한적 TCP 중계 | O |

----

## 47. 실무 예제: Harbor는 왜 HTTP Mode인가

Harbor는 웹 UI와 API를 제공하는 Container Registry입니다.

일반적으로 다음 기능이 필요합니다.

- Host Routing
- TLS Termination
- X-Forwarded-Proto
- X-Forwarded-For
- API Path 처리
- HTTP Status 기반 Health Check

따라서 Harbor는 대체로 HTTP Mode가 적합합니다.

```haproxy
frontend fe_harbor
  bind *:443 ssl crt /etc/haproxy/certs/harbor.pem
  mode http
  option forwardfor
  http-request set-header X-Forwarded-Proto https

  acl host_harbor hdr(host) -i harbor.example.com
  use_backend be_harbor if host_harbor

backend be_harbor
  mode http
  option httpchk GET /api/v2.0/health
  server harbor1 10.0.0.101:80 check
```

Harbor에서 `X-Forwarded-Proto`가 빠지면 HTTPS Redirect나 로그인 흐름에서 문제가 생길 수 있습니다.

----

## 48. 실무 예제: Kubernetes API Server는 왜 TCP Mode인가

Kubernetes API Server는 기본적으로 `6443` 포트에서 TLS로 동작합니다.

```text
kubectl → https://api.example.com:6443 → kube-apiserver
```

API Server는 인증서 기반 인증, 토큰 인증, TLS 검증 등 보안 요소가 강하게 연결되어 있습니다.

HAProxy를 Control Plane 앞단에 둘 때는 보통 TCP Mode로 TLS를 그대로 전달합니다.

```haproxy
frontend fe_k8s_api
  bind *:6443
  mode tcp
  default_backend be_k8s_api

backend be_k8s_api
  mode tcp
  balance roundrobin
  option tcp-check
  server cp1 10.0.0.11:6443 check
  server cp2 10.0.0.12:6443 check
  server cp3 10.0.0.13:6443 check
```

이 경우 HAProxy는 Kubernetes API의 URL Path나 Header를 보지 않습니다.

```text
kubectl == TLS == HAProxy == TLS == kube-apiserver
```

----

## 49. 실무 예제: PostgreSQL, Redis, MySQL, SSH는 왜 TCP Mode인가

이 서비스들은 HTTP가 아닙니다.

| 서비스 | 포트 | 프로토콜 | 권장 mode |
|---|---:|---|---|
| PostgreSQL | 5432 | PostgreSQL Wire Protocol | tcp |
| Redis | 6379 | Redis Serialization Protocol | tcp |
| MySQL | 3306 | MySQL Protocol | tcp |
| SSH | 22 | SSH Protocol | tcp |

PostgreSQL 예시:

```haproxy
frontend fe_postgres
  bind *:5432
  mode tcp
  default_backend be_postgres

backend be_postgres
  mode tcp
  balance leastconn
  server pg1 10.0.0.21:5432 check
  server pg2 10.0.0.22:5432 check backup
```

Redis 예시:

```haproxy
frontend fe_redis
  bind *:6379
  mode tcp
  default_backend be_redis

backend be_redis
  mode tcp
  balance source
  server redis1 10.0.0.31:6379 check
  server redis2 10.0.0.32:6379 check
```

SSH 예시:

```haproxy
frontend fe_ssh
  bind *:2222
  mode tcp
  default_backend be_ssh

backend be_ssh
  mode tcp
  timeout server 1h
  server bastion1 10.0.0.40:22 check
```

----

## 50. 실무 예제: ArgoCD는 왜 HTTP Mode를 고려해야 하는가

ArgoCD는 웹 UI/API와 gRPC 성격의 통신을 함께 고려해야 합니다.

일반적인 접근 주소:

```text
https://argocd.example.com
```

필요한 기능:

- Host Routing
- TLS Termination
- X-Forwarded-Proto
- HTTP/2 또는 gRPC 관련 고려
- Web UI Redirect 처리

예시:

```haproxy
frontend fe_argocd
  bind *:443 ssl crt /etc/haproxy/certs/argocd.pem alpn h2,http/1.1
  mode http
  option forwardfor
  http-request set-header X-Forwarded-Proto https

  acl host_argocd hdr(host) -i argocd.example.com
  use_backend be_argocd if host_argocd

backend be_argocd
  mode http
  server argocd1 10.0.0.111:80 check
```

ArgoCD에서 로그인 Redirect가 이상하거나 CLI 접속이 실패하면 TLS 종료 위치, gRPC 처리, Ingress/HAProxy Header 설정을 함께 봐야 합니다.

----

## 51. 실무 예제: Ceph Dashboard는 왜 HTTP Mode인가

Ceph Dashboard는 웹 UI입니다.

따라서 다음 기능이 필요합니다.

- Host Routing
- TLS Termination 또는 Re-encrypt
- HTTP Health Check
- Header 전달
- Redirect 처리

예시:

```haproxy
frontend fe_ceph_dashboard
  bind *:443 ssl crt /etc/haproxy/certs/ceph.pem
  mode http
  option forwardfor
  http-request set-header X-Forwarded-Proto https

  acl host_ceph hdr(host) -i ceph.example.com
  use_backend be_ceph_dashboard if host_ceph

backend be_ceph_dashboard
  mode http
  option httpchk GET /
  server mgr1 10.0.0.201:8443 ssl verify none check
```

Ceph Dashboard가 HTTPS로 직접 떠 있다면 백엔드 연결에 `ssl` 옵션을 사용해 Re-encrypt 형태로 구성할 수 있습니다.

----

## 52. Kubernetes와 HAProxy Mode 연결

Kubernetes 환경에서는 다음처럼 구분하는 것이 좋습니다.

| 대상 | 권장 mode | 이유 |
|---|---|---|
| Kubernetes API Server 6443 | tcp | TLS를 그대로 전달하는 구성이 일반적 |
| Ingress Controller 앞단 | tcp 또는 http | TLS 종료 위치에 따라 다름 |
| Harbor | http | Host/Path/Header/Redirect 필요 |
| ArgoCD | http | Web UI, API, gRPC 고려 |
| Grafana | http | Web UI, Header, Redirect 필요 |
| Keycloak | http | 로그인 Redirect, X-Forwarded-Proto 중요 |
| Ceph Dashboard | http | Web UI와 HTTP Health Check 필요 |
| PostgreSQL | tcp | HTTP가 아님 |
| Redis | tcp | HTTP가 아님 |
| MySQL | tcp | HTTP가 아님 |
| SSH/Bastion | tcp | HTTP가 아님 |

Ingress와 비교하면 다음과 같습니다.

```text
Kubernetes Ingress
  - 보통 HTTP/HTTPS L7 Routing
  - Host/Path 기반
  - TLS Secret 사용

HAProxy HTTP Mode
  - Ingress와 유사한 L7 Reverse Proxy 역할 가능

HAProxy TCP Mode
  - Ingress보다 하위 계층 TCP Load Balancing
  - API Server, DB, Redis, SSH 등에 적합
```

----

## 53. 장애 사례 1: HTTP Path Routing이 안 됨

증상:

```text
/api 요청은 API 서버로 가야 하는데 계속 default backend로 감
```

잘못된 설정:

```haproxy
frontend fe_tls
  bind *:443
  mode tcp
  acl path_api path_beg /api
  use_backend be_api if path_api
```

원인:

```text
mode tcp에서는 HTTP Path를 볼 수 없다.
```

해결:

TLS Termination 후 HTTP Mode로 처리합니다.

```haproxy
frontend fe_https
  bind *:443 ssl crt /etc/haproxy/certs/example.pem
  mode http
  acl path_api path_beg /api
  use_backend be_api if path_api
  default_backend be_web
```

----

## 54. 장애 사례 2: X-Forwarded-Proto가 없어서 무한 Redirect

증상:

```text
브라우저에서 HTTPS로 접속했는데 계속 Redirect가 반복됨
```

흐름:

```text
Client --HTTPS--> HAProxy --HTTP--> Backend
Backend는 HTTP로 들어온 것으로 판단
Backend가 HTTPS로 Redirect
Client가 다시 HTTPS 요청
반복
```

해결:

```haproxy
http-request set-header X-Forwarded-Proto https if { ssl_fc }
```

또는 고정으로 설정합니다.

```haproxy
http-request set-header X-Forwarded-Proto https
```

----

## 55. 장애 사례 3: SSL Passthrough인데 Host Routing이 안 됨

증상:

```text
Host Header 기준으로 라우팅하려고 했는데 동작하지 않음
```

원인:

SSL Passthrough에서는 HTTP Header가 암호화되어 있습니다.

```text
Client == TLS == Backend
HTTP Host Header는 TLS 내부에 있음
HAProxy는 볼 수 없음
```

가능한 대안:

- SNI 기준으로 도메인 라우팅
- HAProxy에서 TLS Termination 후 Host Header Routing
- 서비스별 포트 분리

SNI 기반 예시:

```haproxy
frontend fe_tls
  bind *:443
  mode tcp
  tcp-request inspect-delay 5s
  tcp-request content accept if { req.ssl_hello_type 1 }

  acl sni_api req.ssl_sni -i api.example.com
  use_backend be_api_tls if sni_api
```

----

## 56. 장애 사례 4: HTTP Health Check 실패

증상:

```text
백엔드 서버는 살아 있는데 HAProxy에서 DOWN으로 표시됨
```

가능한 원인:

- `/healthz` endpoint가 없음
- Health Check 경로가 잘못됨
- Host Header가 필요한데 체크 요청에 Host Header가 없음
- HTTPS 백엔드인데 HTTP로 체크함
- 인증이 필요한 경로를 체크함

해결 예시:

```haproxy
backend be_app
  mode http
  option httpchk GET /healthz
  http-check send hdr Host app.example.com
  http-check expect status 200
  server app1 10.0.0.11:8080 check
```

HTTPS 백엔드라면:

```haproxy
server app1 10.0.0.11:8443 ssl verify none check
```

----

## 57. 장애 사례 5: WebSocket 연결이 자주 끊김

증상:

```text
WebSocket 연결이 몇 초 또는 몇 분 뒤 끊김
```

가능한 원인:

- `timeout tunnel`이 너무 짧음
- 중간 Proxy가 Upgrade를 처리하지 못함
- Backend timeout이 짧음

해결 예시:

```haproxy
defaults
  mode http
  timeout connect 5s
  timeout client  1m
  timeout server  1m
  timeout tunnel  1h
```

----

## 58. 장애 사례 6: Kubernetes API Server를 HTTP Mode로 처리함

증상:

```text
kubectl 접속 실패
certificate error
connection reset
```

원인:

Kubernetes API Server는 TLS 기반 API이며, 보통 HAProxy에서 HTTP로 복호화해 처리하지 않습니다.

권장 구성:

```haproxy
frontend fe_k8s_api
  bind *:6443
  mode tcp
  default_backend be_k8s_api
```

----

## 59. 선택 기준 정리

| 조건 | 권장 mode |
|---|---|
| HTTP Path Routing 필요 | http |
| Host Header 기반 Routing 필요 | http |
| HTTP Header 수정 필요 | http |
| X-Forwarded-For 필요 | http |
| X-Forwarded-Proto 필요 | http |
| HTTP Redirect 필요 | http |
| HTTP Status 기반 Health Check 필요 | http |
| TLS Termination을 HAProxy에서 수행 | http |
| 내부 구간 Re-encrypt 필요 | http |
| SSL Passthrough 필요 | tcp |
| 백엔드가 직접 TLS 인증서 관리 | tcp |
| Kubernetes API Server | tcp |
| PostgreSQL, Redis, MySQL, SSH | tcp |
| Client IP를 TCP 서비스에 전달 | tcp + PROXY Protocol |

----

## 60. 최종 비교표

| 항목 | TCP Mode | HTTP Mode |
|---|---|---|
| 계층 | L4 | L7 |
| 처리 단위 | TCP Connection | HTTP Request / Response |
| HTTP Parsing | X | O |
| Host Routing | X | O |
| SNI Routing | O | O |
| Path Routing | X | O |
| Header Rewrite | X | O |
| Cookie | X | O |
| Authorization Header | X | O |
| Request Body | X | O |
| HTTP Status Code 확인 | X | O |
| Redirect | X | O |
| Compression | X | O |
| X-Forwarded-For | X | O |
| X-Forwarded-Proto | X | O |
| Rate Limit | 연결/IP 기준 제한적 | 요청/Header/Cookie/IP 기준 가능 |
| Sticky Session | Source 기반 | Source, Cookie, Header 기반 |
| Health Check | TCP/SSL | HTTP/HTTPS/TCP |
| TLS Termination | X | O |
| SSL Passthrough | O | X |
| Re-encrypt | X | O |
| PROXY Protocol | O | O |
| Logging | 연결 중심 | 요청/응답 중심 |
| 대표 사용처 | PostgreSQL, Redis, MySQL, SSH, Kubernetes API Server | Harbor, Grafana, Keycloak, ArgoCD, Ceph Dashboard, 일반 웹 서비스 |

----



## 61. 전체 흐름으로 다시 보는 TCP Mode와 HTTP Mode

마지막으로 클라이언트 요청이 실제로 어떤 순서로 흘러가는지 전체 흐름으로 정리합니다.

### 61-1. HTTPS 요청의 일반적인 전체 흐름

```text
Client
  ↓
DNS 조회
  ↓
TCP 3-Way Handshake
  ↓
TLS Handshake
  ↓
HTTP Request
  ↓
HTTP Response
  ↓
TCP Connection 유지 또는 종료
```

이 흐름에서 HAProxy가 어디까지 개입하느냐에 따라 mode 선택이 달라집니다.

### 61-2. TCP Mode 관점

```text
Client
  ↓
TCP Connection
  ↓
HAProxy는 연결을 받음
  ↓
필요하면 SNI 같은 초기 Payload 일부 확인
  ↓
Backend 선택
  ↓
TCP Stream Relay
  ↓
Backend
```

TCP Mode의 핵심은 다음입니다.

```text
HAProxy는 TCP 연결을 중계한다.
HTTP 요청 내용은 보지 않는다.
```

### 61-3. HTTP Mode 관점

```text
Client
  ↓
TCP Connection
  ↓
TLS Termination
  ↓
HTTP Request Parsing
  ↓
ACL / Header / Path / Host 평가
  ↓
Backend 선택
  ↓
HTTP Request 전달
  ↓
HTTP Response 수신
  ↓
Response Header / Status 처리
  ↓
Client 응답
```

HTTP Mode의 핵심은 다음입니다.

```text
HAProxy는 HTTP 요청과 응답을 이해한다.
그래서 L7 Routing과 Header 제어가 가능하다.
```

### 61-4. 한 장으로 요약

```text
TCP Mode
  Client ── TCP ── HAProxy ── TCP ── Backend
                    │
                    └─ 연결 정보 중심 판단

HTTP Mode
  Client ── HTTPS ── HAProxy ── HTTP/HTTPS ── Backend
                     │
                     ├─ TLS Termination
                     ├─ HTTP Parsing
                     ├─ ACL 평가
                     ├─ Header Rewrite
                     └─ Response 처리
```

----

## 62. 확장 최종 비교표

| 항목 | TCP Mode | HTTP Mode |
|---|---|---|
| 계층 | L4 | L7 |
| 기본 처리 단위 | TCP Connection | HTTP Request / Response |
| 데이터 관점 | Byte Stream | 구조화된 HTTP Message |
| TCP 연결 중계 | O | O |
| HTTP Parsing | X | O |
| Request Line 확인 | X | O |
| HTTP Method 확인 | X | O |
| URL Path 확인 | X | O |
| Query String 확인 | X | O |
| Host Header 확인 | X | O |
| Cookie 확인 | X | O |
| Authorization Header 확인 | X | O |
| Request Body 확인 | X | 제한적 |
| Response Status Code 확인 | X | O |
| Response Header Rewrite | X | O |
| Host Routing | X | O |
| SNI Routing | O | O |
| Path Routing | X | O |
| Header Rewrite | X | O |
| URI Rewrite / Set Path | X | O |
| Redirect | X | O |
| Compression | X | O |
| X-Forwarded-For | X | O |
| X-Forwarded-Proto | X | O |
| PROXY Protocol | O | O |
| Sticky Session | Source IP 기반 | Source, Cookie, Header 기반 |
| Rate Limit | 연결 수/IP 기준 제한적 | 요청 수/Header/Cookie/IP 기준 가능 |
| Logging | 연결 중심 | 요청/응답 중심 |
| 로그에서 Method 확인 | X | O |
| 로그에서 Path 확인 | X | O |
| 로그에서 Status Code 확인 | X | O |
| 기본 Timeout | connect/client/server | connect/client/server/http-request/http-keep-alive/tunnel |
| TCP Health Check | O | O |
| SSL Check | O | O |
| HTTP Health Check | X 또는 제한적 | O |
| HTTPS Health Check | 제한적 | O |
| TLS Termination | X | O |
| SSL Passthrough | O | X |
| Re-encrypt | X | O |
| HTTP/2 인식 | 단순 중계 | O |
| gRPC 인식 | 단순 중계 | O |
| WebSocket Upgrade 처리 | 단순 중계 | O |
| WebSocket timeout tunnel | 상황에 따라 | 중요 |
| 대표 사용처 | PostgreSQL, Redis, MySQL, SSH, Kubernetes API Server | Harbor, Grafana, Keycloak, ArgoCD, Ceph Dashboard, 일반 웹 서비스 |
| 주요 장점 | 단순함, 비HTTP 지원, SSL Passthrough | L7 제어, Routing, Header, Redirect, Health Check |
| 주요 한계 | HTTP 정보 사용 불가 | TLS 종료/인증서 관리 필요 |

----

## 63. 결론

HAProxy의 `mode tcp`와 `mode http` 차이는 단순한 설정 옵션 차이가 아닙니다.

`mode tcp`는 TCP 연결을 기준으로 트래픽을 중계합니다. 그래서 HTTP가 아닌 서비스에도 사용할 수 있고, SSL Passthrough에도 적합합니다. 하지만 HTTP Header, Path, Cookie, Status Code를 볼 수 없기 때문에 L7 기능은 사용할 수 없습니다.

`mode http`는 HTTP 요청과 응답을 파싱합니다. 그래서 Host Routing, Path Routing, Header Rewrite, Redirect, HTTP Health Check, X-Forwarded-For, X-Forwarded-Proto, Cookie 기반 Sticky Session 같은 기능을 사용할 수 있습니다. 대신 HTTPS를 다루려면 일반적으로 HAProxy에서 TLS Termination 또는 Re-encrypt 구성이 필요합니다.

실무에서는 다음 기준으로 판단하면 됩니다.

```text
HTTP 내용을 보고 판단해야 한다면 mode http
HTTP 내용을 보면 안 되거나 HTTP가 아니라면 mode tcp
```

Kubernetes 환경에서는 특히 다음처럼 구분하는 것이 안전합니다.

```text
Kubernetes API Server → TCP Mode
PostgreSQL / Redis / MySQL / SSH → TCP Mode
Harbor / ArgoCD / Grafana / Keycloak / Ceph Dashboard → HTTP Mode
```

결국 핵심은 TLS 종료 위치입니다.

```text
TLS를 HAProxy에서 종료하면 HTTP Mode 기능을 사용할 수 있다.
TLS를 백엔드까지 그대로 넘기면 TCP Mode로 처리해야 한다.
```