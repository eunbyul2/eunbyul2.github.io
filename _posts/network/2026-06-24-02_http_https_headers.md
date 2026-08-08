---
layout: post
title: "HTTP/HTTPS와 HTTP Header 완전 "
date: 2026-06-24 18:57:00 +0900
categories: [Network, HTTP]
tags: [HTTP, HTTPS, TLS, SSL, Header, Cookie, Session, JWT, REST, HTTP2, HTTP3, Ingress, Proxy]
published: true
---

## 1. 이 글에서 다루는 범위

이 글은 Kubernetes, Ingress, HAProxy, API Gateway, Harbor, Keycloak, Grafana, ArgoCD 같은 웹 기반 시스템을 운영할 때 반드시 알아야 하는 HTTP/HTTPS 기본기를 정리한다.

HTTP를 단순히 “웹 요청 프로토콜” 정도로만 이해하면 실제 장애를 분석하기 어렵다. 실무에서는 다음과 같은 질문에 답할 수 있어야 한다.

```text
브라우저에서 URL을 입력하면 실제로 어떤 순서로 요청이 이동하는가?
HTTP는 TCP 위에서 어떻게 동작하는가?
HTTPS는 HTTP와 무엇이 다른가?
TLS 인증서는 왜 필요한가?
Host Header가 틀리면 왜 Ingress 라우팅이 실패하는가?
X-Forwarded-Proto가 틀리면 왜 로그인 Redirect Loop가 발생하는가?
Cookie와 Session은 로그인 유지에 어떻게 사용되는가?
JWT는 Cookie/Session과 무엇이 다른가?
502, 503, 504는 각각 어느 구간의 문제인가?
HTTP/1.1과 HTTP/2는 운영 관점에서 무엇이 다른가?
```

이 글의 목적은 용어 암기가 아니라, 실제 요청 흐름을 기준으로 HTTP 계층을 이해하고 장애 지점을 좁히는 것이다.

공식 참고자료:

- MDN HTTP Overview: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview
- MDN HTTP Messages: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Messages
- MDN HTTP Methods: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods
- MDN HTTP Status Codes: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status
- MDN HTTP Headers: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers
- MDN Set-Cookie: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie
- MDN CORS: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- MDN Same-Origin Policy: https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy
- HTTP Core Specifications: https://httpwg.org/specs/
- RFC 9110 HTTP Semantics: https://datatracker.ietf.org/doc/html/rfc9110
- RFC 9111 HTTP Caching: https://datatracker.ietf.org/doc/html/rfc9111
- RFC 9112 HTTP/1.1: https://datatracker.ietf.org/doc/html/rfc9112
- RFC 9113 HTTP/2: https://datatracker.ietf.org/doc/html/rfc9113
- RFC 8446 TLS 1.3: https://datatracker.ietf.org/doc/html/rfc8446
- RFC 7301 ALPN: https://datatracker.ietf.org/doc/html/rfc7301
- RFC 6066 TLS Extensions, SNI: https://datatracker.ietf.org/doc/html/rfc6066
- RFC 9114 HTTP/3: https://datatracker.ietf.org/doc/html/rfc9114
- RFC 9000 QUIC: https://datatracker.ietf.org/doc/html/rfc9000
- Kubernetes Ingress: https://kubernetes.io/docs/concepts/services-networking/ingress/
- Kubernetes Service: https://kubernetes.io/docs/concepts/services-networking/service/
- ingress-nginx User Guide: https://kubernetes.github.io/ingress-nginx/user-guide/
- HAProxy Configuration Manual: https://docs.haproxy.org/

## 2. HTTP를 이해하기 전에 먼저 봐야 하는 전체 요청 흐름

HTTP는 혼자 동작하지 않는다. 사용자가 브라우저에 `https://harbor.example.com`을 입력하면 실제로는 여러 계층이 순서대로 동작한다.

```text
사용자 브라우저
  ↓
URL 해석
  ↓
DNS 조회
  ↓
TCP 연결
  ↓
TLS Handshake
  ↓
HTTP Request 생성
  ↓
Load Balancer / Ingress / Reverse Proxy
  ↓
Service
  ↓
Backend Pod 또는 Application Server
  ↓
HTTP Response 반환
  ↓
브라우저 렌더링 또는 API 응답 처리
```

이 흐름을 계층별로 나누면 다음과 같다.

| 단계 | 계층 | 주요 기술 | 장애 예시 |
|---|---|---|---|
| 도메인 해석 | DNS/L7 | DNS, CoreDNS | `Could not resolve host` |
| 연결 생성 | L4 | TCP, Port | `Connection refused`, `Connection timed out` |
| 암호화 협상 | TLS | Certificate, SNI, ALPN | 인증서 오류, TLS handshake failure |
| 요청 처리 | HTTP/L7 | Method, URI, Header, Body | 400, 401, 403, 404 |
| 프록시 라우팅 | L7 Proxy | Host, Path, X-Forwarded-* | 404, 502, Redirect Loop |
| 백엔드 연결 | L4/L7 | Service, Endpoint, Pod | 502, 503, 504 |
| 응답 반환 | HTTP/L7 | Status Code, Header, Body | 느린 응답, 잘못된 Content-Type |

즉 HTTP 장애를 볼 때는 단순히 `curl` 결과만 보는 것이 아니라, DNS → TCP → TLS → HTTP → Proxy → Backend 순서로 좁혀야 한다.

## 3. HTTP란 무엇인가

HTTP는 HyperText Transfer Protocol의 약자다. 웹에서 클라이언트와 서버가 리소스를 주고받기 위한 애플리케이션 계층 프로토콜이다.

HTTP에서 중요한 단어는 세 가지다.

| 용어 | 의미 |
|---|---|
| Client | 요청을 보내는 쪽. 브라우저, curl, 모바일 앱, SDK, 다른 서버 등이 될 수 있다. |
| Server | 요청을 받아 처리하고 응답하는 쪽. 웹 서버, API 서버, Ingress, Gateway 등이 될 수 있다. |
| Resource | 요청 대상. HTML, JSON, 이미지, 파일, API 결과 등이 될 수 있다. |

HTTP의 기본 모델은 요청/응답이다.

```text
Client  →  HTTP Request   →  Server
Client  ←  HTTP Response  ←  Server
```

예를 들어 다음 명령은 `example.com` 서버에 HTTP 요청을 보낸다.

```bash
curl -v http://example.com/
```

HTTP는 기본적으로 Stateless 프로토콜이다. Stateless란 서버가 이전 요청 상태를 HTTP 프로토콜 자체만으로 기억하지 않는다는 뜻이다.

```text
1번째 요청: GET /login
2번째 요청: POST /login
3번째 요청: GET /my-page
```

HTTP 자체만 보면 이 세 요청은 독립적이다. 사용자가 로그인했는지 계속 기억하려면 Cookie, Session, Token 같은 별도 메커니즘이 필요하다.

## 4. HTTP가 TCP 위에서 동작한다는 의미

HTTP/1.1과 HTTP/2는 일반적으로 TCP 위에서 동작한다. HTTPS는 여기에 TLS가 추가된다.

HTTP 요청 하나를 보낼 때 실제 계층은 다음과 같다.

```text
HTTP Request
  ↓
TCP Segment
  ↓
IP Packet
  ↓
Ethernet Frame
```

HTTPS에서는 다음과 같이 된다.

```text
HTTP Request
  ↓
TLS Record
  ↓
TCP Segment
  ↓
IP Packet
  ↓
Ethernet Frame
```

즉 `curl https://example.com`이 실패했을 때는 HTTP만 의심하면 안 된다.

```text
DNS가 안 되는가?
TCP 443 연결이 안 되는가?
TLS 인증서 검증이 실패하는가?
SNI가 맞지 않는가?
HTTP Host/Path가 틀렸는가?
백엔드 애플리케이션이 오류를 반환했는가?
```

이런 식으로 계층을 분리해야 한다.

## 5. HTTP와 HTTPS의 차이

HTTP는 기본적으로 평문으로 데이터를 전송한다.

```text
Client → Server
GET /login HTTP/1.1
Host: example.com
Authorization: Bearer token
```

중간에서 패킷을 볼 수 있는 사람이 있다면 Header와 Body 내용을 읽을 수 있다. 그래서 로그인, 토큰, 개인정보, 결제 정보 같은 데이터를 HTTP 평문으로 전송하면 안 된다.

HTTPS는 HTTP를 TLS로 보호한다.

```text
HTTP + TLS = HTTPS
```

HTTPS가 제공하는 핵심 기능은 다음과 같다.

| 기능 | 설명 |
|---|---|
| 암호화 | 중간에서 패킷을 보더라도 HTTP 원문을 읽기 어렵게 한다. |
| 무결성 | 전송 중 데이터가 변조되었는지 확인한다. |
| 서버 인증 | 접속한 서버가 해당 도메인의 진짜 서버인지 인증서로 확인한다. |

HTTPS는 보통 TCP 443 포트를 사용한다. HTTP는 보통 TCP 80 포트를 사용한다.

## 6. TLS/SSL 기본 개념

SSL은 오래된 명칭이고 현재 표준은 TLS다. 실무에서는 여전히 “SSL 인증서”, “SSL 설정”, “SSL Passthrough”라는 표현을 많이 쓰지만 실제로는 TLS를 의미하는 경우가 대부분이다.

TLS를 이해하려면 다음 개념이 필요하다.

| 용어 | 설명 |
|---|---|
| Certificate | 서버의 신원을 증명하는 인증서 |
| Private Key | 인증서와 짝이 되는 개인키. 외부에 노출되면 안 된다. |
| Public Key | 인증서 안에 포함되는 공개키 |
| CA | 인증서를 발급하거나 신뢰 체인을 제공하는 인증기관 |
| Root CA | 운영체제/브라우저가 신뢰하는 최상위 인증기관 |
| Intermediate CA | Root CA와 서버 인증서 사이에 있는 중간 인증기관 |
| Chain of Trust | 서버 인증서가 신뢰 가능한 CA까지 이어지는 검증 체인 |
| SAN | Subject Alternative Name. 인증서가 유효한 도메인 목록 |
| SNI | TLS Handshake 때 클라이언트가 접속하려는 도메인을 서버에 알려주는 확장 |
| ALPN | TLS 위에서 HTTP/1.1, HTTP/2 같은 애플리케이션 프로토콜을 협상하는 확장 |

## 7. TLS Handshake 흐름

HTTPS 요청은 먼저 TCP 연결을 만들고, 그 다음 TLS Handshake를 수행한 뒤, 암호화된 HTTP 데이터를 주고받는다.

```text
1. TCP 3-Way Handshake
2. TLS Handshake
3. 암호화된 HTTP Request
4. 암호화된 HTTP Response
```

단순화한 TLS Handshake 흐름은 다음과 같다.

```text
Client → Server: ClientHello
  - 지원 TLS 버전
  - 지원 Cipher Suite
  - SNI: app.example.com
  - ALPN: h2, http/1.1

Server → Client: ServerHello
  - 선택된 TLS 버전
  - 선택된 Cipher Suite
  - 선택된 ALPN
  - 서버 인증서

Client:
  - 인증서가 신뢰 가능한 CA에서 발급되었는지 확인
  - 인증서 SAN에 접속 도메인이 포함되어 있는지 확인
  - 인증서 만료 여부 확인

Client ↔ Server:
  - 세션 키 생성
  - 이후 HTTP 데이터 암호화 전송
```

TLS 오류는 HTTP 요청이 서버 애플리케이션까지 도달하기 전에 발생한다. 따라서 TLS 오류가 나면 애플리케이션 로그에 아무것도 안 찍힐 수 있다.

확인 명령어:

```bash
# 인증서와 TLS Handshake 확인
openssl s_client -connect app.example.com:443 -servername app.example.com

# curl로 TLS 상세 확인
curl -vk https://app.example.com/
```

## 8. SNI가 중요한 이유

SNI는 Server Name Indication의 약자다. TLS Handshake 단계에서 클라이언트가 접속하려는 도메인을 서버에 알려주는 기능이다.

왜 필요할까?

하나의 IP에서 여러 HTTPS 도메인을 운영할 수 있기 때문이다.

```text
203.0.113.10:443
  ├─ harbor.example.com
  ├─ grafana.example.com
  ├─ argocd.example.com
  └─ keycloak.example.com
```

서버는 클라이언트가 어떤 도메인으로 접속하려는지 알아야 올바른 인증서를 선택할 수 있다. 이때 SNI가 사용된다.

SNI가 없거나 틀리면 다음 문제가 발생할 수 있다.

```text
인증서 도메인 불일치
기본 인증서 반환
TLS handshake failure
Ingress TLS Secret 불일치
브라우저 보안 경고
```

테스트 예시:

```bash
# SNI 없이 접근하면 기본 인증서가 나올 수 있음
openssl s_client -connect 203.0.113.10:443

# SNI를 명시하면 해당 도메인의 인증서를 확인할 수 있음
openssl s_client -connect 203.0.113.10:443 -servername harbor.example.com
```

## 9. TLS Termination, Re-encryption, SSL Passthrough

Ingress, HAProxy, Load Balancer를 다룰 때 반드시 구분해야 한다.

### 9-1) TLS Termination

TLS Termination은 프록시가 클라이언트와의 HTTPS 연결을 종료하고, 내부로는 HTTP로 전달하는 방식이다.

```text
Client
  ↓ HTTPS
Ingress / HAProxy
  ↓ HTTP
Backend Pod
```

장점:

```text
프록시가 HTTP Header, Host, Path를 읽을 수 있다.
Path-based Routing, Header Rewrite, Redirect 처리가 가능하다.
인증서 관리를 프록시 계층에서 집중할 수 있다.
```

주의점:

```text
백엔드 애플리케이션 입장에서는 직접 받은 요청이 HTTP로 보인다.
원래 요청이 HTTPS였다는 정보는 X-Forwarded-Proto 같은 Header로 전달해야 한다.
이 설정이 틀리면 Redirect Loop, 잘못된 Callback URL, Cookie Secure 문제 등이 발생한다.
```

### 9-2) Re-encryption

프록시에서 TLS를 한 번 해제한 뒤, 백엔드로 다시 HTTPS 연결을 맺는 방식이다.

```text
Client
  ↓ HTTPS
Ingress / HAProxy
  ↓ HTTPS
Backend Pod
```

장점은 내부 구간도 암호화할 수 있다는 것이다. 단점은 인증서와 신뢰 체인 관리가 더 복잡해진다는 것이다.

### 9-3) SSL Passthrough

SSL Passthrough는 프록시가 TLS를 해제하지 않고 TCP 그대로 백엔드로 넘기는 방식이다.

```text
Client
  ↓ HTTPS 그대로 통과
Ingress / HAProxy
  ↓ HTTPS 그대로 통과
Backend Pod
```

장점:

```text
프록시가 인증서 개인키를 가질 필요가 없다.
End-to-End TLS를 유지할 수 있다.
```

단점:

```text
프록시가 HTTP Path, Header, Cookie를 볼 수 없다.
Path-based Routing이 어렵거나 불가능하다.
L4 수준 라우팅 위주로 동작한다.
```

## 10. HTTP Request 구조

HTTP Request는 클라이언트가 서버에게 보내는 메시지다.

HTTP/1.1 기준 Request는 다음 구조를 가진다.

```http
GET /api/users?page=1 HTTP/1.1
Host: app.example.com
User-Agent: curl/8.0
Accept: application/json
Authorization: Bearer eyJ...

```

구성 요소는 다음과 같다.

| 구성 | 예시 | 설명 |
|---|---|---|
| Request Line | `GET /api/users?page=1 HTTP/1.1` | Method, Request Target, HTTP Version |
| Header | `Host: app.example.com` | 요청 메타데이터 |
| Empty Line | 빈 줄 | Header와 Body를 구분 |
| Body | JSON, Form, File 등 | 서버로 전송할 데이터 |

POST 요청은 Body를 포함할 수 있다.

```http
POST /api/users HTTP/1.1
Host: app.example.com
Content-Type: application/json
Authorization: Bearer eyJ...

{"name":"eunbyul","role":"admin"}
```

## 11. HTTP Response 구조

HTTP Response는 서버가 클라이언트에게 보내는 메시지다.

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 27
Set-Cookie: sessionid=abc123; HttpOnly; Secure; SameSite=Lax

{"status":"ok","data":[]}
```

구성 요소는 다음과 같다.

| 구성 | 예시 | 설명 |
|---|---|---|
| Status Line | `HTTP/1.1 200 OK` | HTTP Version, Status Code, Reason Phrase |
| Header | `Content-Type: application/json` | 응답 메타데이터 |
| Empty Line | 빈 줄 | Header와 Body 구분 |
| Body | JSON, HTML 등 | 실제 응답 데이터 |

## 12. HTTP Method

HTTP Method는 요청의 의도를 나타낸다.

| Method | 의미 | Body | Safe | Idempotent | 주요 용도 |
|---|---|---:|---:|---:|---|
| GET | 리소스 조회 | 보통 없음 | 예 | 예 | 목록 조회, 상세 조회 |
| POST | 리소스 생성 또는 처리 | 가능 | 아니오 | 보통 아니오 | 생성, 로그인, 명령 실행 |
| PUT | 리소스 전체 교체 | 가능 | 아니오 | 예 | 전체 수정 |
| PATCH | 리소스 일부 수정 | 가능 | 아니오 | 보장 아님 | 부분 수정 |
| DELETE | 리소스 삭제 | 가능 | 아니오 | 예로 설계 권장 | 삭제 |
| HEAD | GET과 같지만 Body 없음 | 없음 | 예 | 예 | 상태/헤더 확인 |
| OPTIONS | 지원 기능 확인 | 가능 | 예 | 예 | CORS Preflight |

Safe는 서버 상태를 변경하지 않는다는 의미다. GET, HEAD, OPTIONS는 일반적으로 Safe Method다.

Idempotent는 같은 요청을 여러 번 보내도 결과 상태가 같다는 의미다.

예시:

```text
DELETE /users/1
```

한 번 보내도 삭제되고, 두 번 보내도 이미 삭제된 상태다. 그래서 DELETE는 보통 idempotent하게 설계한다.

반면 다음 요청은 여러 번 보내면 주문이 여러 개 생성될 수 있다.

```text
POST /orders
```

그래서 결제, 주문, 배포 실행 같은 API는 중복 요청 방지를 위해 Idempotency-Key를 사용하기도 한다.

```http
POST /payments HTTP/1.1
Idempotency-Key: 9f2c1a3e-7b60-4c5a
```

## 13. URI, URL, Path, Query Parameter

URI는 리소스를 식별하는 더 넓은 개념이고, URL은 위치 정보를 포함한 URI다. 실무에서는 대체로 URL이라는 표현을 많이 쓴다.

예시:

```text
https://app.example.com:443/api/users?page=1&size=20#profile
```

구성은 다음과 같다.

| 구성 | 값 | 설명 |
|---|---|---|
| Scheme | `https` | 사용할 프로토콜 |
| Host | `app.example.com` | 요청 대상 도메인 |
| Port | `443` | 대상 포트. 기본 포트면 생략 가능 |
| Path | `/api/users` | 서버 안의 리소스 경로 |
| Query Parameter | `page=1&size=20` | 조회 조건 |
| Fragment | `profile` | 브라우저 내부 위치. 서버로 전송되지 않음 |

Query Parameter는 주로 조회 조건에 사용한다.

```text
/api/users?page=1&size=20&sort=createdAt,desc
```

주의할 점:

```text
민감정보를 Query Parameter에 넣으면 안 된다.
브라우저 히스토리, 프록시 로그, 웹 서버 접근 로그에 남을 수 있다.
토큰, 비밀번호, 주민번호, 인증 코드 등은 Query보다 Header 또는 Body 사용을 검토해야 한다.
```

## 14. HTTP Status Code

Status Code는 요청 처리 결과를 숫자로 표현한다.

| 범위 | 의미 | 예시 |
|---|---|---|
| 1xx | 정보 | 100 Continue, 101 Switching Protocols |
| 2xx | 성공 | 200 OK, 201 Created, 204 No Content |
| 3xx | 리다이렉션/캐시 | 301, 302, 304, 307, 308 |
| 4xx | 클라이언트 오류 | 400, 401, 403, 404, 409, 415, 422, 429 |
| 5xx | 서버/프록시 오류 | 500, 502, 503, 504 |

실무에서 자주 보는 코드는 다음과 같다.

| 코드 | 의미 | 주로 의심할 지점 |
|---:|---|---|
| 200 | 성공 | 정상 응답 |
| 201 | 생성됨 | POST 생성 API 정상 |
| 204 | 응답 Body 없음 | DELETE, update 성공 후 본문 없음 |
| 301 | 영구 이동 | HTTP→HTTPS, 도메인 변경 |
| 302 | 임시 이동 | 로그인 후 Redirect |
| 304 | 변경 없음 | 브라우저/프록시 캐시 |
| 400 | Bad Request | 요청 문법, Header, Body, Host 오류 |
| 401 | Unauthorized | 인증 필요 또는 인증 실패 |
| 403 | Forbidden | 인증은 되었지만 권한 없음 |
| 404 | Not Found | Path, Host Routing, 백엔드 라우트 없음 |
| 409 | Conflict | 중복 생성, 리소스 상태 충돌 |
| 415 | Unsupported Media Type | Content-Type 불일치 |
| 422 | Unprocessable Content | JSON 구조/검증 실패 |
| 429 | Too Many Requests | Rate Limit |
| 500 | Internal Server Error | 애플리케이션 내부 오류 |
| 502 | Bad Gateway | 프록시가 백엔드 응답을 정상적으로 받지 못함 |
| 503 | Service Unavailable | 백엔드 없음, Endpoint 없음, Ready 아님 |
| 504 | Gateway Timeout | 백엔드 응답 지연 또는 timeout |

Kubernetes Ingress 관점에서는 502/503/504를 반드시 구분해야 한다.

```text
502: Ingress가 Backend에 연결했지만 비정상 응답 또는 연결 실패
503: Service에 사용할 수 있는 Endpoint가 없음
504: Backend에 요청은 보냈지만 제한 시간 안에 응답이 없음
```

정확한 매핑은 Ingress Controller 구현체마다 다를 수 있다. 따라서 NGINX Ingress, HAProxy Ingress, Traefik 각각의 로그와 메트릭을 함께 봐야 한다.

## 15. HTTP Header란 무엇인가

Header는 HTTP 메시지의 메타데이터다. Body가 실제 데이터라면 Header는 그 데이터를 어떻게 해석하고 처리할지 알려주는 정보다.

```http
Host: app.example.com
User-Agent: curl/8.0
Accept: application/json
Content-Type: application/json
Authorization: Bearer eyJ...
Cookie: sessionid=abc123
X-Request-ID: 9f2c1a3e
```

Header는 다음 영역에서 중요하다.

```text
Ingress Host-based Routing
Path Routing
인증/인가
세션 유지
콘텐츠 협상
압축
캐싱
프록시 원본 정보 전달
분산 추적
보안 정책
```

HTTP Header는 대소문자를 구분하지 않는 필드 이름을 가진다. 예를 들어 `Host`, `host`, `HOST`는 같은 Header 이름으로 취급된다. 다만 실무에서는 관례적으로 `Content-Type`, `Authorization` 같은 표기를 사용한다.

## 16. Host Header

Host Header는 HTTP/1.1에서 매우 중요하다.

```http
Host: app.example.com
```

Host Header가 필요한 이유는 하나의 IP에서 여러 도메인을 서비스할 수 있기 때문이다.

```text
203.0.113.10
  ├─ harbor.example.com
  ├─ grafana.example.com
  ├─ argocd.example.com
  └─ keycloak.example.com
```

웹 서버나 Ingress Controller는 Host Header를 보고 어떤 Virtual Host 또는 Ingress Rule로 보낼지 결정한다.

Kubernetes Ingress 예시:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: harbor-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: harbor.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: harbor-service
                port:
                  number: 80
```

이때 실제 요청의 Host Header가 `harbor.example.com`이어야 해당 Rule에 매칭된다.

테스트:

```bash
# DNS를 거치지 않고 특정 IP로 Host Header만 강제 테스트
curl -v -H 'Host: harbor.example.com' http://10.0.0.10/
```

Host Header가 틀리면 다음 현상이 발생할 수 있다.

```text
Default Backend로 이동
404 Not Found
잘못된 서비스로 라우팅
인증 Callback URL 오류
TLS 인증서 도메인 불일치
```

## 17. User-Agent

User-Agent는 요청을 보낸 클라이언트 정보를 나타낸다.

```http
User-Agent: curl/8.0
User-Agent: Mozilla/5.0 ... Chrome/...
```

실무 활용:

```text
브라우저 요청인지 curl 요청인지 구분
모바일 앱 버전 구분
봇/크롤러 탐지
특정 SDK 버전 장애 분석
WAF 정책 적용
```

주의할 점은 User-Agent는 클라이언트가 임의로 조작할 수 있다는 것이다. 보안상 신뢰할 수 있는 인증 정보로 사용하면 안 된다.

## 18. Accept, Accept-Encoding, Accept-Language

### 18-1) Accept

Accept는 클라이언트가 받을 수 있는 응답 형식을 나타낸다.

```http
Accept: application/json
Accept: text/html
Accept: */*
```

API 서버는 이 값을 참고해 JSON, HTML, XML 등 응답 형식을 결정할 수 있다.

### 18-2) Accept-Encoding

Accept-Encoding은 클라이언트가 받을 수 있는 압축 방식을 나타낸다.

```http
Accept-Encoding: gzip, br, zstd
```

서버가 압축 응답을 보내면 응답 Header에 다음이 포함된다.

```http
Content-Encoding: gzip
```

장점:

```text
응답 크기 감소
네트워크 대역폭 절약
페이지 로딩 속도 개선
```

주의점:

```text
CPU 사용량 증가
프록시에서 압축/해제 정책 불일치 가능
이미 압축된 이미지/영상에는 효과가 작음
```

### 18-3) Accept-Language

Accept-Language는 클라이언트가 선호하는 언어를 나타낸다.

```http
Accept-Language: ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7
```

`q` 값은 선호도다. `ko-KR`이 가장 높고, 그 다음이 `ko`, `en-US` 순서다.

## 19. Content-Type과 Content-Length

### 19-1) Content-Type

Content-Type은 Body의 형식을 나타낸다.

```http
Content-Type: application/json
Content-Type: application/x-www-form-urlencoded
Content-Type: multipart/form-data
Content-Type: text/plain
```

API에서 매우 중요하다.

예를 들어 서버가 JSON을 기대하는데 클라이언트가 `Content-Type`을 보내지 않거나 다른 값을 보내면 다음 오류가 발생할 수 있다.

```text
400 Bad Request
415 Unsupported Media Type
422 Unprocessable Content
```

JSON POST 테스트:

```bash
curl -X POST https://app.example.com/api/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"test"}'
```

파일 업로드 테스트:

```bash
curl -X POST https://app.example.com/api/upload \
  -F 'file=@./test.png'
```

`-F`를 사용하면 curl이 `multipart/form-data`를 자동으로 구성한다.

### 19-2) Content-Length

Content-Length는 Body 크기를 바이트 단위로 나타낸다.

```http
Content-Length: 128
```

프록시, 웹 서버, 애플리케이션 서버의 최대 Body 크기 제한과 관련이 있다.

예시 장애:

```text
413 Payload Too Large
```

가능 원인:

```text
Ingress body size 제한
Nginx client_max_body_size 제한
Application upload size 제한
API Gateway payload 제한
```

## 20. Authorization Header

Authorization Header는 인증 정보를 전달한다.

```http
Authorization: Bearer eyJhbGciOi...
Authorization: Basic dXNlcjpwYXNz
```

대표 방식:

| 방식 | 설명 |
|---|---|
| Basic | username/password를 Base64로 인코딩해 전달. 반드시 HTTPS와 함께 사용해야 한다. |
| Bearer | 토큰을 소유한 사람에게 접근 권한을 부여하는 방식. JWT/OAuth2에서 자주 사용한다. |
| Digest | 오래된 인증 방식. 현재 일반 API에서는 덜 사용된다. |

주의:

```text
Authorization Header는 로그에 남기면 안 된다.
프록시 access log에서 마스킹해야 한다.
오류 디버깅 시에도 토큰 원문 공유를 피해야 한다.
```

## 21. Cookie와 Set-Cookie

Cookie는 서버가 브라우저에 저장시키고, 브라우저가 이후 요청에 다시 보내는 작은 데이터다.

로그인 흐름 예시:

```text
1. 사용자가 로그인 요청
2. 서버가 로그인 성공 처리
3. 서버가 Set-Cookie 응답
4. 브라우저가 Cookie 저장
5. 이후 요청마다 Cookie 자동 전송
6. 서버는 Cookie를 보고 로그인 상태 판단
```

응답 예시:

```http
HTTP/1.1 200 OK
Set-Cookie: sessionid=abc123; Path=/; HttpOnly; Secure; SameSite=Lax
```

다음 요청:

```http
GET /my-page HTTP/1.1
Host: app.example.com
Cookie: sessionid=abc123
```

중요 속성:

| 속성 | 설명 |
|---|---|
| Path | Cookie가 전송될 URL Path 범위 |
| Domain | Cookie가 전송될 도메인 범위 |
| Expires / Max-Age | 만료 시간 |
| HttpOnly | JavaScript에서 Cookie 접근 차단. XSS 피해 완화 |
| Secure | HTTPS 요청에서만 Cookie 전송 |
| SameSite | Cross-site 요청에서 Cookie 전송 제한 |

실무 장애 예시:

```text
HTTPS 서비스인데 X-Forwarded-Proto가 http로 들어감
↓
애플리케이션이 Secure Cookie를 제대로 설정하지 않음
↓
브라우저가 Cookie 저장 또는 전송을 거부
↓
로그인 유지 실패
```

또는

```text
Keycloak / Harbor / Grafana가 외부 URL을 https로 인식하지 못함
↓
http callback URL 생성
↓
브라우저가 redirect 또는 cookie 정책 때문에 로그인 실패
```

## 22. Session

Session은 서버가 사용자 상태를 저장하는 방식이다. Cookie에는 보통 Session ID만 저장하고, 실제 로그인 상태나 사용자 정보는 서버 또는 Session Store에 저장한다.

```text
Browser Cookie
sessionid=abc123

Server Session Store
abc123 → user_id=100, role=admin, expire=...
```

장점:

```text
민감한 사용자 정보를 브라우저에 직접 저장하지 않아도 된다.
서버에서 세션을 강제로 만료시킬 수 있다.
```

단점:

```text
서버 측 저장소가 필요하다.
여러 서버로 확장할 때 Session 공유가 필요하다.
Sticky Session이나 Redis Session Store가 필요할 수 있다.
```

Load Balancer 환경에서는 Session 저장 방식이 중요하다.

```text
Client
  ↓
Load Balancer
  ├─ App Pod 1
  ├─ App Pod 2
  └─ App Pod 3
```

Pod 1의 메모리에만 Session이 저장되어 있으면 다음 요청이 Pod 2로 갈 때 로그인이 풀릴 수 있다.

해결 방법:

```text
Redis 같은 외부 Session Store 사용
Sticky Session 사용
JWT 기반 Stateless 인증 사용
```

## 23. JWT

JWT는 JSON Web Token의 약자다. 인증/인가 정보를 토큰 형태로 전달할 때 많이 사용한다.

HTTP 요청에서는 보통 Authorization Header에 담긴다.

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

JWT는 세 부분으로 구성된다.

```text
Header.Payload.Signature
```

예시 구조:

```text
Header
- alg: 서명 알고리즘
- typ: JWT

Payload
- sub: 사용자 식별자
- exp: 만료 시간
- iss: 발급자
- aud: 대상 서비스
- role: 권한 정보

Signature
- Header와 Payload가 변조되지 않았는지 검증하기 위한 서명
```

JWT 장점:

```text
서버가 세션 저장소를 갖지 않아도 인증 정보를 검증할 수 있다.
마이크로서비스/API Gateway 구조에서 사용하기 좋다.
Keycloak, OAuth2, OIDC와 잘 맞는다.
```

주의:

```text
JWT Payload는 암호화가 아니라 Base64URL 인코딩이다.
민감정보를 Payload에 넣으면 안 된다.
토큰 탈취 시 만료 전까지 악용될 수 있다.
만료 시간, Refresh Token, Revocation 전략이 필요하다.
```

## 24. OAuth2와 OIDC 기초

OAuth2는 권한 위임 프로토콜이고, OIDC는 OAuth2 위에 인증 계층을 추가한 표준이다.

간단히 구분하면 다음과 같다.

| 구분 | 목적 |
|---|---|
| OAuth2 | 이 사용자가 어떤 리소스에 접근할 권한이 있는가 |
| OIDC | 이 사용자가 누구인가 |

Keycloak은 대표적인 Identity Provider다. Harbor, Grafana, ArgoCD 같은 서비스는 Keycloak을 통해 로그인할 수 있다.

일반적인 OIDC 로그인 흐름:

```text
1. 사용자가 Harbor 접속
2. Harbor가 Keycloak 로그인 페이지로 Redirect
3. 사용자가 Keycloak에서 로그인
4. Keycloak이 Authorization Code를 Harbor Callback URL로 전달
5. Harbor가 Code를 Token으로 교환
6. Harbor가 사용자 정보를 확인하고 세션 생성
```

이 흐름에서 자주 깨지는 부분:

```text
Callback URL 불일치
X-Forwarded-Proto 오류
X-Forwarded-Host 오류
Ingress Host 설정 오류
TLS Termination 이후 애플리케이션이 외부 Scheme을 잘못 인식
Cookie SameSite/Secure 문제
```

## 25. X-Forwarded-* Header

Reverse Proxy나 Load Balancer를 거치면 백엔드 애플리케이션은 원래 클라이언트 정보를 직접 알기 어렵다.

예를 들어 실제 요청 흐름이 다음과 같다고 하자.

```text
Client 203.0.113.10
  ↓ HTTPS
HAProxy 10.0.0.10
  ↓ HTTP
NGINX Ingress 10.0.0.20
  ↓ HTTP
Backend Pod 10.244.1.15
```

Backend Pod 입장에서 직접 연결한 상대는 Client가 아니라 Ingress다.

```text
Backend에서 보이는 Remote IP = 10.0.0.20
실제 사용자 IP = 203.0.113.10
```

그래서 프록시는 원래 정보를 Header로 전달한다.

### 25-1) X-Forwarded-For

원래 클라이언트 IP를 전달한다.

```http
X-Forwarded-For: 203.0.113.10, 10.0.0.10, 10.0.0.20
```

해석:

```text
가장 왼쪽: 원래 클라이언트 IP
그 뒤: 거쳐온 프록시 IP들
```

주의:

```text
클라이언트가 임의로 X-Forwarded-For를 조작할 수 있다.
신뢰할 수 있는 프록시가 설정한 값만 신뢰해야 한다.
애플리케이션에서 trusted proxy 설정이 필요할 수 있다.
```

### 25-2) X-Forwarded-Host

원래 요청 Host를 전달한다.

```http
X-Forwarded-Host: harbor.example.com
```

외부 Host 기준으로 Redirect URL을 만들 때 중요하다.

### 25-3) X-Forwarded-Proto

원래 요청 Scheme을 전달한다.

```http
X-Forwarded-Proto: https
```

TLS Termination 후 내부로 HTTP를 전달하는 경우 특히 중요하다.

장애 예시:

```text
사용자 → https://harbor.example.com
Ingress → Backend로 http 전달
Backend가 X-Forwarded-Proto를 보지 못함
Backend가 현재 요청을 http로 오해
http://harbor.example.com/callback 생성
브라우저 Redirect Loop 또는 Mixed Content 문제 발생
```

### 25-4) X-Forwarded-Port

원래 요청 포트를 전달한다.

```http
X-Forwarded-Port: 443
```

### 25-5) Forwarded Header

표준 Header로는 `Forwarded`가 있다.

```http
Forwarded: for=203.0.113.10;proto=https;host=harbor.example.com
```

다만 실무에서는 `X-Forwarded-*` 계열이 여전히 많이 사용된다.

## 26. X-Request-ID와 Correlation ID

X-Request-ID는 요청 하나를 추적하기 위한 식별자다.

```http
X-Request-ID: 9f2c1a3e-7b60-4c5a-9a65-111111111111
```

MSA나 Kubernetes 환경에서는 요청이 여러 컴포넌트를 거친다.

```text
Ingress
  ↓
Frontend API
  ↓
Backend API
  ↓
Database
  ↓
External API
```

각 구간 로그가 따로 남으면 하나의 요청을 추적하기 어렵다. 그래서 동일한 Request ID를 모든 로그에 남긴다.

예시 로그:

```text
[ingress] request_id=9f2c1a3e path=/api/users status=502
[backend] request_id=9f2c1a3e db_timeout=true
[db] request_id=9f2c1a3e query_time=10s
```

이렇게 하면 502가 단순 Ingress 문제가 아니라 DB timeout에서 시작되었음을 추적할 수 있다.

W3C Trace Context에서는 다음 Header도 사용한다.

```http
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-00
tracestate: vendor=value
```

OpenTelemetry 기반 분산 추적에서는 `traceparent`가 중요하다.

## 27. HTTP Body

Body는 실제 Payload다.

```http
POST /api/users HTTP/1.1
Host: app.example.com
Content-Type: application/json

{"name":"eunbyul"}
```

Body 형식은 Content-Type에 따라 달라진다.

| Content-Type | Body 예시 | 용도 |
|---|---|---|
| application/json | `{"name":"test"}` | REST API |
| application/x-www-form-urlencoded | `id=1&name=test` | HTML Form |
| multipart/form-data | 파일 + 필드 | 파일 업로드 |
| text/plain | 일반 텍스트 | 단순 텍스트 전송 |
| application/octet-stream | 바이너리 | 파일/바이너리 데이터 |

트러블슈팅 포인트:

```text
Body가 비어 있는가?
Content-Type이 맞는가?
JSON 문법이 올바른가?
서버가 기대하는 필드명이 맞는가?
Ingress 또는 Proxy에서 Body 크기 제한에 걸리지 않는가?
```

## 28. Keep-Alive와 Connection Reuse

HTTP 요청마다 TCP 연결과 TLS 연결을 새로 만들면 비용이 크다.

매번 새 연결을 만들 경우:

```text
Request 1
TCP Handshake
TLS Handshake
HTTP Request/Response
Connection Close

Request 2
TCP Handshake
TLS Handshake
HTTP Request/Response
Connection Close
```

Keep-Alive를 사용하면 하나의 연결을 재사용한다.

```text
TCP + TLS Connection 생성
  ├─ HTTP Request/Response 1
  ├─ HTTP Request/Response 2
  ├─ HTTP Request/Response 3
  └─ idle timeout 후 종료
```

장점:

```text
TCP Handshake 비용 감소
TLS Handshake 비용 감소
Latency 감소
서버 자원 효율 개선
```

운영 주의점:

```text
Client, Load Balancer, Ingress, Backend의 idle timeout 값이 서로 다르면 502/504가 발생할 수 있다.
Connection Pool이 고갈되면 요청 지연이 발생할 수 있다.
오래 열린 연결이 많으면 file descriptor를 많이 사용한다.
```

예시 장애:

```text
Load Balancer idle timeout = 60초
Backend keepalive timeout = 30초

LB는 연결이 살아있다고 생각하고 재사용
Backend는 이미 연결 종료
요청 전달 시 RST 또는 broken pipe
502 Bad Gateway 발생 가능
```

## 29. Connection Pool

애플리케이션 서버나 HTTP Client는 성능을 위해 Connection Pool을 사용한다.

```text
API Server
  ↓ connection pool
Payment API
```

매 요청마다 외부 API로 새 TCP/TLS 연결을 만들지 않고, 미리 만들어둔 연결을 재사용한다.

문제 상황:

```text
pool max size가 너무 작음 → 대기열 증가
idle connection 검증 없음 → 죽은 연결 재사용
timeout 설정 과도함 → 요청이 오래 묶임
DNS 변경 후 기존 연결 계속 사용
```

트러블슈팅 시 확인할 항목:

```text
connect timeout
read timeout
idle timeout
max connections
max idle connections
retry 정책
circuit breaker 여부
```

## 30. Chunked Transfer Encoding

HTTP/1.1에서는 응답 크기를 미리 알 수 없을 때 Chunked Transfer Encoding을 사용할 수 있다.

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked

4\r\n
Wiki\r\n
5\r\n
pedia\r\n
0\r\n
\r\n
```

의미:

```text
응답 Body를 여러 조각으로 나눠서 보낸다.
Content-Length를 미리 알 필요가 없다.
스트리밍 응답에 사용할 수 있다.
```

실무에서는 다음과 관련된다.

```text
대용량 응답
로그 스트리밍
SSE(Server-Sent Events)
프록시 buffering 설정
응답이 중간에 끊기는 문제
```

## 31. Compression

HTTP 응답은 gzip, br, zstd 같은 방식으로 압축할 수 있다.

요청:

```http
Accept-Encoding: gzip, br
```

응답:

```http
Content-Encoding: gzip
```

장점:

```text
응답 크기 감소
네트워크 비용 감소
페이지 로딩 개선
```

단점:

```text
CPU 사용 증가
압축이 필요 없는 파일에도 압축하면 비효율
프록시와 애플리케이션 중복 압축 가능
```

## 32. HTTP Caching

HTTP Cache는 동일 리소스를 반복 다운로드하지 않기 위한 메커니즘이다.

대표 Header:

```http
Cache-Control: max-age=3600
ETag: "abc123"
Last-Modified: Wed, 24 Jun 2026 08:00:00 GMT
```

브라우저는 이후 요청에서 조건부 요청을 보낼 수 있다.

```http
If-None-Match: "abc123"
If-Modified-Since: Wed, 24 Jun 2026 08:00:00 GMT
```

서버가 변경되지 않았다고 판단하면 다음 응답을 보낸다.

```http
HTTP/1.1 304 Not Modified
```

캐시 관련 장애:

```text
정적 파일이 갱신되지 않음
브라우저가 오래된 JS/CSS 사용
CDN 캐시가 오래된 API 응답 반환
ETag 불일치
Cache-Control 설정 오류
```

## 33. HTTP/1.1

HTTP/1.1은 텍스트 기반 메시지 형식을 사용한다.

특징:

```text
Host Header 필수
Keep-Alive 기본
Chunked Transfer Encoding 지원
요청/응답이 텍스트 라인 기반
하나의 연결에서 동시에 여러 응답을 섞어 보내기 어려움
```

요청 예시:

```http
GET / HTTP/1.1
Host: example.com
Accept: */*

```

HTTP/1.1의 한계:

```text
동일 연결에서 요청/응답 처리 효율이 낮음
여러 리소스를 받기 위해 여러 TCP 연결을 만들기도 함
Head-of-line blocking 문제가 발생할 수 있음
Header가 반복되어 오버헤드 발생
```

## 34. HTTP/2

HTTP/2는 HTTP 의미론은 유지하면서 전송 방식을 개선한 프로토콜이다.

중요한 점:

```text
GET, POST, Status Code, Header 같은 HTTP 의미는 유지된다.
하지만 전송 방식은 HTTP/1.1처럼 텍스트 라인 기반이 아니라 Binary Frame 기반이다.
```

HTTP/2 핵심 기능:

| 기능 | 설명 |
|---|---|
| Binary Framing | HTTP 메시지를 binary frame으로 쪼개 전송 |
| Multiplexing | 하나의 TCP 연결에서 여러 Stream을 동시에 처리 |
| HPACK | Header 압축 |
| Stream | 하나의 요청/응답 흐름을 나타내는 단위 |

HTTP/1.1 방식:

```text
Connection 1: Request A → Response A
Connection 2: Request B → Response B
Connection 3: Request C → Response C
```

HTTP/2 방식:

```text
Connection 1:
  Stream 1: Request A / Response A
  Stream 3: Request B / Response B
  Stream 5: Request C / Response C
```

운영 관점 주의:

```text
Ingress가 Client와는 HTTP/2로 통신하고 Backend와는 HTTP/1.1로 통신할 수 있다.
HTTP/2는 하나의 TCP 연결에 여러 요청이 몰리므로 connection 단위 메트릭만 보면 부하를 오해할 수 있다.
gRPC는 보통 HTTP/2 위에서 동작한다.
```

## 35. HTTP/3와 QUIC

HTTP/3는 QUIC 위에서 동작한다. QUIC은 UDP 기반 전송 프로토콜이다.

```text
HTTP/1.1, HTTP/2
  ↓
TCP

HTTP/3
  ↓
QUIC
  ↓
UDP
```

HTTP/3가 등장한 이유는 TCP 기반 HTTP/2에서도 TCP 계층의 Head-of-line blocking 문제가 남아 있기 때문이다.

HTTP/3 특징:

```text
UDP 기반 QUIC 사용
TLS 1.3 통합
연결 수립 지연 감소
네트워크 변경에 더 유연
Stream 단위 손실 복구
```

운영 관점:

```text
방화벽에서 UDP 443 허용 필요
Load Balancer가 HTTP/3를 지원해야 함
기존 HTTP/1.1/HTTP/2와 함께 fallback 운영되는 경우가 많음
```

## 36. REST API 기본

REST는 HTTP를 사용해 리소스를 표현하고 조작하는 API 설계 스타일이다.

리소스 중심 URL 예시:

```text
GET    /users        사용자 목록 조회
GET    /users/1      사용자 상세 조회
POST   /users        사용자 생성
PUT    /users/1      사용자 전체 수정
PATCH  /users/1      사용자 일부 수정
DELETE /users/1      사용자 삭제
```

좋은 API 설계의 기본:

```text
명사는 URI에 사용한다.
동사는 HTTP Method로 표현한다.
상태 코드를 의미에 맞게 반환한다.
오류 응답 형식을 일관되게 유지한다.
인증/인가 실패를 401/403으로 구분한다.
```

나쁜 예:

```text
POST /getUser
POST /deleteUser
```

좋은 예:

```text
GET /users/1
DELETE /users/1
```

## 37. gRPC와 HTTP

gRPC는 Google이 만든 RPC 프레임워크이며, 일반적으로 HTTP/2 위에서 동작한다.

특징:

```text
HTTP/2 사용
Protocol Buffers 사용
양방향 스트리밍 지원
마이크로서비스 내부 통신에 많이 사용
```

Ingress/Proxy 관점에서 gRPC는 일반 HTTP API와 다르게 봐야 한다.

```text
gRPC는 HTTP/2가 필요하다.
프록시가 HTTP/2 upstream을 지원해야 한다.
일반 REST처럼 curl만으로 테스트하기 어렵다.
grpcurl 같은 도구를 사용한다.
```

예시:

```bash
grpcurl -plaintext localhost:50051 list
```

## 38. CORS와 Same-Origin Policy

브라우저는 보안상 Same-Origin Policy를 적용한다.

Origin은 다음 세 가지 조합이다.

```text
Scheme + Host + Port
```

예시:

```text
https://app.example.com
https://api.example.com
```

Host가 다르므로 서로 다른 Origin이다.

프론트엔드가 다른 Origin의 API를 호출하려면 서버가 CORS Header를 허용해야 한다.

응답 예시:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Allow-Credentials: true
```

CORS 장애 특징:

```text
서버 로그에는 200이 찍히는데 브라우저에서는 실패로 보일 수 있다.
curl은 CORS를 강제하지 않으므로 curl에서는 정상인데 브라우저만 실패할 수 있다.
OPTIONS Preflight 요청이 실패하면 실제 요청이 보내지지 않을 수 있다.
```

Preflight 예시:

```text
Browser → OPTIONS /api/users
Server  → Access-Control-Allow-* Header 응답
Browser → 실제 POST /api/users 요청
```

## 39. CSRF와 XSS 기초

### 39-1) CSRF

CSRF는 사용자가 로그인한 상태를 악용해 원치 않는 요청을 보내게 만드는 공격이다.

Cookie 기반 인증에서는 브라우저가 Cookie를 자동으로 전송하기 때문에 CSRF 방어가 중요하다.

방어 방법:

```text
CSRF Token
SameSite Cookie
중요 요청에 대한 재인증
Origin / Referer 검증
```

### 39-2) XSS

XSS는 악성 스크립트가 웹 페이지에서 실행되는 공격이다.

XSS가 발생하면 다음 피해가 가능하다.

```text
사용자 세션 탈취
사용자 화면 조작
API 요청 위조
민감정보 탈취
```

방어 방법:

```text
입력값 검증
출력 시 escaping
Content Security Policy
HttpOnly Cookie 사용
```

## 40. Forward Proxy, Reverse Proxy, API Gateway

### 40-1) Forward Proxy

Forward Proxy는 클라이언트 대신 외부 서버에 요청을 보내는 프록시다.

```text
Client → Forward Proxy → Internet Server
```

주요 용도:

```text
회사 내부망 인터넷 접근 제어
캐싱
접속 로그 기록
보안 필터링
```

### 40-2) Reverse Proxy

Reverse Proxy는 서버 앞단에서 클라이언트 요청을 받아 내부 서버로 전달한다.

```text
Client → Reverse Proxy → Backend Server
```

Ingress, HAProxy, NGINX, Traefik, API Gateway는 Reverse Proxy 역할을 한다.

주요 기능:

```text
TLS Termination
Host-based Routing
Path-based Routing
Load Balancing
Header Rewrite
Authentication 연동
Rate Limiting
Observability
```

### 40-3) API Gateway

API Gateway는 Reverse Proxy보다 API 관리 기능이 강화된 계층이다.

주요 기능:

```text
인증/인가
Rate Limit
Request/Response 변환
API Key 관리
JWT 검증
서비스 라우팅
로깅/메트릭/추적
```

## 41. Kubernetes Ingress와 HTTP

Kubernetes Ingress는 외부 HTTP/HTTPS 요청을 클러스터 내부 Service로 라우팅하는 API 객체다.

중요한 점은 Ingress 자체가 트래픽을 처리하지 않는다는 것이다. 실제 트래픽 처리는 Ingress Controller가 한다.

```text
Ingress Resource = 라우팅 규칙 선언
Ingress Controller = 실제 프록시 구현체
```

흐름:

```text
Client
  ↓ HTTPS
Load Balancer
  ↓
Ingress Controller Pod
  ↓
Service
  ↓
EndpointSlice
  ↓
Pod
```

Ingress Rule은 주로 Host와 Path를 기준으로 동작한다.

```yaml
rules:
  - host: app.example.com
    http:
      paths:
        - path: /api
          pathType: Prefix
          backend:
            service:
              name: api-service
              port:
                number: 80
```

이때 요청이 다음과 같으면 매칭된다.

```http
GET /api/users HTTP/1.1
Host: app.example.com
```

요청이 다음과 같으면 Host가 다르므로 매칭되지 않는다.

```http
GET /api/users HTTP/1.1
Host: wrong.example.com
```

## 42. HTTP 기반 실전 장애 사례

### 42-1) 404 Not Found

가능 원인:

```text
Ingress Host 불일치
Ingress Path 불일치
Backend 애플리케이션 라우트 없음
잘못된 Context Path
Default Backend로 라우팅
```

확인:

```bash
kubectl get ingress -A
kubectl describe ingress -n <namespace> <name>
curl -v -H 'Host: app.example.com' http://<ingress-ip>/api/health
```

### 42-2) 502 Bad Gateway

가능 원인:

```text
Ingress가 Backend Pod에 연결 실패
Backend가 연결을 즉시 종료
HTTP/HTTPS 프로토콜 불일치
Backend Port 오설정
Keep-Alive timeout mismatch
```

예시:

```text
Ingress는 HTTP로 backend에 요청
Backend는 HTTPS만 수신
↓
502 발생 가능
```

확인:

```bash
kubectl get svc -n <namespace>
kubectl get endpoints -n <namespace>
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
curl -v http://<pod-ip>:<port>/health
```

### 42-3) 503 Service Unavailable

가능 원인:

```text
Service Selector가 Pod Label과 불일치
Ready 상태의 Pod 없음
EndpointSlice 없음
Deployment Replica 0
Readiness Probe 실패
```

확인:

```bash
kubectl get svc -n <namespace> <service-name> -o yaml
kubectl get endpoints -n <namespace> <service-name>
kubectl get endpointslice -n <namespace>
kubectl get pod -n <namespace> --show-labels
kubectl describe pod -n <namespace> <pod-name>
```

### 42-4) 504 Gateway Timeout

가능 원인:

```text
Backend 응답 지연
DB Query 지연
외부 API 지연
프록시 timeout 짧음
네트워크 지연 또는 패킷 드롭
```

확인:

```bash
curl -v --max-time 10 https://app.example.com/api/slow
kubectl logs -n <namespace> <backend-pod>
kubectl top pod -n <namespace>
```

### 42-5) Redirect Loop

가능 원인:

```text
HTTP → HTTPS Redirect 설정 중복
X-Forwarded-Proto 누락
애플리케이션 external URL 설정 오류
Ingress TLS Termination 후 backend가 http로 오해
Keycloak/Harbor/Grafana root_url 설정 오류
```

흐름 예시:

```text
Client → https://harbor.example.com
Ingress → Backend로 http 전달
Backend → http 요청으로 인식
Backend → https로 redirect
Client → 다시 https 요청
반복
```

확인:

```bash
curl -vkL https://harbor.example.com/
curl -vkI https://harbor.example.com/
```

응답 Header에서 `Location`을 확인한다.

```http
Location: http://harbor.example.com/login
```

외부 서비스가 HTTPS인데 Location이 HTTP로 나온다면 `X-Forwarded-Proto` 또는 external URL 설정을 의심한다.

### 42-6) 로그인은 되는데 새로고침하면 풀림

가능 원인:

```text
Cookie Domain 불일치
Cookie Path 불일치
Secure Cookie 설정 오류
SameSite 정책 문제
Session Store 공유 안 됨
Sticky Session 필요
```

확인:

```text
브라우저 개발자 도구 → Application → Cookies
브라우저 개발자 도구 → Network → Request Header Cookie 확인
Response Header Set-Cookie 확인
```

### 42-7) curl은 되는데 브라우저는 안 됨

가능 원인:

```text
CORS 오류
Cookie SameSite/Secure 정책
Mixed Content 차단
브라우저 캐시
HSTS
인증서 신뢰 문제
```

특히 CORS는 브라우저 보안 정책이므로 curl에서는 재현되지 않을 수 있다.

## 43. HTTP 트러블슈팅 명령어

### 43-1) 기본 요청 확인

```bash
curl -v https://app.example.com/
```

확인할 것:

```text
DNS 결과
TCP 연결 성공 여부
TLS Handshake
Request Header
Response Status Code
Response Header
Redirect Location
```

### 43-2) 인증서 검증 무시

```bash
curl -vk https://app.example.com/
```

`-k`는 인증서 검증을 무시한다. 운영에서 문제를 숨기는 설정이 아니라, 원인 분리를 위한 테스트에만 사용해야 한다.

### 43-3) Header만 확인

```bash
curl -I https://app.example.com/
```

### 43-4) Redirect 따라가기

```bash
curl -vkL https://app.example.com/
```

Redirect Loop 분석 시 `Location` Header를 반드시 본다.

### 43-5) Host Header 강제

```bash
curl -v -H 'Host: app.example.com' http://10.0.0.10/
```

DNS를 우회하고 Ingress IP로 직접 Host Routing을 테스트할 때 유용하다.

### 43-6) DNS를 curl에서 강제 지정

```bash
curl -v --resolve app.example.com:443:10.0.0.10 https://app.example.com/
```

이 방법은 Host Header와 SNI까지 `app.example.com`으로 유지하면서 실제 접속 IP만 `10.0.0.10`으로 바꾼다. Ingress TLS/Host 테스트에 매우 유용하다.

### 43-7) POST Body 테스트

```bash
curl -X POST https://app.example.com/api/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"test"}' \
  -v
```

### 43-8) TLS 인증서 확인

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com
```

확인할 것:

```text
인증서 Subject
SAN
Issuer
만료일
Chain
ALPN 결과
```

### 43-9) HTTP/2 확인

```bash
curl -vk --http2 https://app.example.com/
```

### 43-10) 응답 시간 확인

```bash
curl -o /dev/null -s -w '\nnamelookup:  %{time_namelookup}\nconnect:     %{time_connect}\nappconnect:  %{time_appconnect}\nstarttransfer:%{time_starttransfer}\ntotal:       %{time_total}\n' https://app.example.com/
```

해석:

| 값 | 의미 |
|---|---|
| time_namelookup | DNS 조회 시간 |
| time_connect | TCP 연결 완료까지 시간 |
| time_appconnect | TLS Handshake 완료까지 시간 |
| time_starttransfer | 첫 응답 바이트까지 시간 |
| time_total | 전체 요청 시간 |

이 값으로 DNS 문제인지, TCP/TLS 문제인지, 백엔드 응답 지연인지 분리할 수 있다.

## 44. 브라우저 개발자 도구로 확인할 것

브라우저 문제는 curl만으로 충분하지 않다. 개발자 도구 Network 탭을 봐야 한다.

확인 항목:

```text
Request URL
Request Method
Status Code
Remote Address
Request Headers
Response Headers
Cookie 전송 여부
Set-Cookie 수신 여부
Redirect Chain
CORS 오류 메시지
Timing
```

Timing에서 볼 것:

```text
Queueing
DNS Lookup
Initial Connection
SSL
Waiting for server response
Content Download
```

특히 `Waiting for server response`가 길면 백엔드 처리 지연 가능성이 높다.

## 45. HTTP 문제를 계층별로 나누어 보기

HTTP 장애는 다음 순서로 분리한다.

```text
1. URL이 맞는가?
2. DNS가 맞는 IP를 반환하는가?
3. TCP 포트가 열려 있는가?
4. TLS 인증서와 SNI가 맞는가?
5. HTTP Host Header가 Ingress Rule과 일치하는가?
6. Path가 올바른 Backend로 라우팅되는가?
7. Service Endpoint가 존재하는가?
8. Backend Pod가 Ready인가?
9. Backend 애플리케이션 로그에 오류가 있는가?
10. Cookie/Session/JWT 인증 흐름이 정상인가?
11. Proxy timeout, body size, header size 제한에 걸리지 않는가?
12. CORS 또는 브라우저 보안 정책 문제인가?
```

## 46. Kubernetes 환경 HTTP 장애 점검 체크리스트

```bash
# Ingress 확인
kubectl get ingress -A
kubectl describe ingress -n <namespace> <ingress-name>

# Service 확인
kubectl get svc -n <namespace>
kubectl describe svc -n <namespace> <service-name>

# Endpoint / EndpointSlice 확인
kubectl get endpoints -n <namespace>
kubectl get endpointslice -n <namespace>

# Pod 상태 확인
kubectl get pod -n <namespace> -o wide
kubectl describe pod -n <namespace> <pod-name>

# 애플리케이션 로그 확인
kubectl logs -n <namespace> <pod-name>

# 클러스터 내부에서 Service 호출
kubectl run curl-test --rm -it --image=curlimages/curl -- sh
curl -v http://<service-name>.<namespace>.svc.cluster.local:<port>/health
```

Ingress Controller 로그:

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

NGINX Ingress를 쓴다면 다음도 확인한다.

```text
nginx.ingress.kubernetes.io/proxy-connect-timeout
nginx.ingress.kubernetes.io/proxy-read-timeout
nginx.ingress.kubernetes.io/proxy-send-timeout
nginx.ingress.kubernetes.io/proxy-body-size
nginx.ingress.kubernetes.io/backend-protocol
nginx.ingress.kubernetes.io/ssl-redirect
```

## 47. HTTP와 Observability

HTTP 장애를 운영 환경에서 제대로 분석하려면 로그, 메트릭, 트레이싱이 함께 필요하다.

### 47-1) Logging

로그에 남기면 좋은 필드:

```text
timestamp
request_id
trace_id
client_ip
method
host
path
status_code
latency
upstream_addr
upstream_status
user_agent
```

민감정보는 남기면 안 된다.

```text
Authorization
Cookie
Set-Cookie
Password
Token
Personal Information
```

### 47-2) Metrics

HTTP 메트릭 예시:

```text
request count
status code count
latency histogram
request body size
response body size
active connections
upstream error count
```

Prometheus 예시 지표 이름은 구현체마다 다르지만 보통 다음 형태를 가진다.

```text
http_requests_total
http_request_duration_seconds_bucket
nginx_ingress_controller_requests
nginx_ingress_controller_request_duration_seconds
haproxy_frontend_http_responses_total
```

### 47-3) Tracing

분산 추적은 요청이 여러 서비스를 지날 때 전체 경로를 보여준다.

```text
Ingress Span
  ↓
API Server Span
  ↓
DB Query Span
  ↓
External API Span
```

이를 통해 다음을 확인할 수 있다.

```text
어느 서비스에서 시간이 오래 걸렸는가?
어느 구간에서 에러가 발생했는가?
재시도 때문에 지연이 늘었는가?
DB 쿼리 지연인가 외부 API 지연인가?
```

## 48. 최종 요약

HTTP/HTTPS를 실무적으로 이해하려면 다음 흐름을 기준으로 봐야 한다.

```text
URL
  ↓
DNS
  ↓
TCP
  ↓
TLS
  ↓
HTTP Request
  ↓
Header / Body
  ↓
Proxy / Ingress
  ↓
Service / Endpoint
  ↓
Backend Application
  ↓
HTTP Response
  ↓
Status Code / Header / Body
```

핵심 정리:

```text
HTTP는 요청/응답 기반의 Stateless 애플리케이션 프로토콜이다.
HTTPS는 HTTP를 TLS로 보호한 것이다.
TLS에서는 인증서, CA, SNI, ALPN이 중요하다.
Host Header는 Ingress와 Virtual Host 라우팅의 핵심이다.
Cookie/Session은 로그인 상태 유지에 사용된다.
JWT는 토큰 기반 인증/인가에서 자주 사용된다.
X-Forwarded-* Header는 프록시 뒤의 애플리케이션이 원래 요청 정보를 알기 위해 필요하다.
502/503/504는 각각 프록시-백엔드 연결, Endpoint, Timeout 관점으로 나눠 봐야 한다.
CORS는 서버가 아니라 브라우저 보안 정책 때문에 발생하는 문제일 수 있다.
HTTP 문제는 DNS → TCP → TLS → HTTP → Proxy → Backend 순서로 좁혀야 한다.
```

