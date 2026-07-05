---
layout: post
title: "Load Balancer와 Proxy: L4/L7, VIP, Health Check, Sticky Session, Failover"
date: 2026-06-24 22:15:00 +0900
categories: [스터디, 인프라, Network, LoadBalancer]
tags: [LoadBalancer, L4, L7, ReverseProxy, ForwardProxy, HealthCheck, Failover, HAProxy, NGINX, MetalLB, Envoy]
published: true
---

## 1. 이 글에서 다루는 범위

이 글은 Load Balancer와 Proxy를 실무 관점에서 이해하기 위한 글입니다.

앞선 글에서 네트워크 기본기, HTTP/HTTPS, Kubernetes Network, Service, Ingress를 정리했다면, 이 글에서는 그 앞단 또는 중간에 위치하는 Load Balancer와 Proxy가 실제로 어떤 역할을 하는지 정리합니다.

실무에서 Load Balancer는 단순히 트래픽을 여러 서버로 나누는 장비가 아닙니다. 실제로는 다음 역할을 동시에 수행합니다.

- 외부 사용자가 접근하는 단일 진입점 제공
- 여러 서버로 트래픽 분산
- 장애 서버 자동 제외
- TLS 종료 또는 TLS 통과 처리
- 원본 Client IP 전달
- HTTP Host/Path/Header 기반 라우팅
- Session Persistence 유지
- 배포 중 Connection Draining
- 장애 시 Failover
- 보안 정책 적용
- 관측성 지표 제공

Kubernetes 환경에서는 Load Balancer가 다음 구성요소들과 연결됩니다.

```text
Client
  ↓
DNS
  ↓
External Load Balancer
  ↓
Ingress Controller
  ↓
Service
  ↓
EndpointSlice
  ↓
Pod
```

따라서 Load Balancer를 이해하지 못하면 Ingress, HAProxy, NGINX, MetalLB, AWS NLB/ALB, Keycloak, Harbor, Grafana, ArgoCD 장애를 정확히 분석하기 어렵습니다.

공식 참고자료:

- HAProxy Configuration Manual: https://www.haproxy.com/documentation/haproxy-configuration-manual/latest/
- HAProxy Architecture Guide: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/core-concepts/architecture/
- NGINX Reverse Proxy: https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/
- NGINX Load Balancing: https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/
- Kubernetes Service LoadBalancer: https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer
- Kubernetes External Traffic Policy: https://kubernetes.io/docs/reference/networking/virtual-ips/#external-traffic-policy
- MetalLB Documentation: https://metallb.io/
- AWS Network Load Balancer: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html
- AWS Application Load Balancer: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html
- Envoy Proxy Documentation: https://www.envoyproxy.io/docs/envoy/latest/
- PROXY Protocol Specification: https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt
- NGINX PROXY Protocol: https://docs.nginx.com/nginx/admin-guide/load-balancer/using-proxy-protocol/
- RFC 9110 HTTP Semantics: https://datatracker.ietf.org/doc/html/rfc9110
- RFC 8446 TLS 1.3: https://datatracker.ietf.org/doc/html/rfc8446

## 2. Load Balancer가 필요한 이유

### 2-1) 서버 한 대 구조의 한계

가장 단순한 웹 서비스 구조는 다음과 같습니다.

```text
Client
  ↓
Web Server
```

초기에는 이 구조로도 충분합니다. 하지만 사용자가 늘어나면 한 대의 서버가 처리해야 하는 일이 많아집니다.

서버 한 대가 감당해야 하는 리소스는 다음과 같습니다.

- CPU
- Memory
- Disk I/O
- Network I/O
- TCP Connection
- TLS Handshake
- HTTP Request 처리
- DB Connection
- Application Thread 또는 Worker

어느 하나라도 한계에 도달하면 응답 지연, timeout, 5xx 오류가 발생합니다.

예를 들어 웹 서버 한 대가 초당 1,000 요청까지 안정적으로 처리할 수 있는데 초당 5,000 요청이 들어오면 서버를 수평 확장해야 합니다.

```text
Client
  ↓
Server A
Server B
Server C
```

하지만 여기서 새로운 문제가 생깁니다.

사용자는 어느 서버로 접속해야 할까요?

사용자가 직접 `Server A`, `Server B`, `Server C`를 골라 접속하게 만들 수는 없습니다. 서버가 장애 났는지, 부하가 높은지, 배포 중인지도 사용자는 알 수 없습니다.

그래서 서버 여러 대 앞에 단일 진입점이 필요합니다.

그 단일 진입점이 Load Balancer입니다.

```text
Client
  ↓
Load Balancer
  ├─ Server A
  ├─ Server B
  └─ Server C
```

### 2-2) Load Balancer의 핵심 역할

Load Balancer는 다음 문제를 해결합니다.

| 문제 | Load Balancer의 역할 |
|---|---|
| 서버가 여러 대라 사용자가 어디로 갈지 모름 | 단일 진입점 제공 |
| 특정 서버에 부하 집중 | 트래픽 분산 |
| 서버 장애 | Health Check 후 장애 서버 제외 |
| 배포 중 일부 서버 제외 필요 | Drain 또는 Maintenance 상태 처리 |
| TLS 인증서 중앙 관리 필요 | TLS Termination |
| 내부 서버 주소 숨김 | Reverse Proxy 역할 |
| 사용자 IP 추적 필요 | X-Forwarded-For 또는 PROXY Protocol 전달 |
| 로그인 세션 유지 필요 | Sticky Session 또는 Session Persistence |
| 전 세계 사용자 분산 | GSLB, Geo DNS, Anycast 사용 |

즉 Load Balancer는 단순 분산기가 아니라 서비스 가용성과 운영 안정성을 담당하는 핵심 네트워크 구성요소입니다.

## 3. Load Balancer와 VIP

### 3-1) VIP란 무엇인가

VIP는 Virtual IP의 약자입니다. 사용자가 접근하는 대표 IP입니다.

예를 들어 다음과 같은 구성이 있다고 가정합니다.

```text
VIP: 203.0.113.10

Backend Server A: 10.0.0.11
Backend Server B: 10.0.0.12
Backend Server C: 10.0.0.13
```

사용자는 `10.0.0.11`, `10.0.0.12`, `10.0.0.13`을 직접 알 필요가 없습니다.

사용자는 오직 다음 주소로만 접속합니다.

```text
https://app.example.com
  ↓ DNS
203.0.113.10
```

Load Balancer는 `203.0.113.10`으로 들어온 요청을 내부 Backend 서버 중 하나로 전달합니다.

```text
Client
  ↓
203.0.113.10:443
  ↓
Load Balancer
  ├─ 10.0.0.11:8080
  ├─ 10.0.0.12:8080
  └─ 10.0.0.13:8080
```

### 3-2) VIP가 중요한 이유

VIP가 있으면 Backend 서버가 바뀌어도 사용자는 같은 주소로 접근합니다.

예를 들어 `Server A`를 교체해도 사용자는 알 필요가 없습니다.

```text
Before:
VIP -> A, B, C

After:
VIP -> B, C, D
```

사용자는 계속 `app.example.com`으로 접속합니다.

이 구조가 운영에서 중요한 이유는 다음과 같습니다.

- Backend 서버 교체 가능
- Rolling Update 가능
- 장애 서버 자동 제외 가능
- 외부 DNS 변경 최소화
- 인증서와 보안 정책 중앙화 가능

### 3-3) VIP와 Kubernetes Service의 관계

Kubernetes Service의 ClusterIP도 일종의 Virtual IP입니다.

```text
Service backend
ClusterIP: 10.96.20.15

Endpoint:
10.244.1.10:8080
10.244.2.20:8080
```

차이점은 다음과 같습니다.

| 구분 | 일반 Load Balancer VIP | Kubernetes ClusterIP |
|---|---|---|
| 접근 범위 | 외부 또는 내부 | 클러스터 내부 |
| 구현 | LB 장비, 클라우드 LB, HAProxy, NGINX 등 | kube-proxy, IPVS, eBPF 등 |
| Backend | 서버 IP | Pod IP |
| 목적 | 사용자 또는 시스템 진입점 | Pod 집합에 대한 안정적 진입점 |

따라서 Service를 이해할 때도 Load Balancer 개념을 알고 있으면 훨씬 이해가 쉽습니다.

## 4. Load Balancer의 기본 구조

### 4-1) Frontend와 Backend

Load Balancer는 일반적으로 Frontend와 Backend로 나뉩니다.

```text
Client
  ↓
Frontend
  ↓
Backend Pool
  ├─ Server A
  ├─ Server B
  └─ Server C
```

Frontend는 클라이언트가 접속하는 진입점입니다.

예시:

```text
0.0.0.0:80
0.0.0.0:443
203.0.113.10:443
```

Backend는 실제 요청을 처리할 서버 목록입니다.

예시:

```text
10.0.0.11:8080
10.0.0.12:8080
10.0.0.13:8080
```

HAProxy에서는 이 구조가 매우 명확합니다.

```haproxy
frontend https_front
    bind *:443
    default_backend app_back

backend app_back
    server app1 10.0.0.11:8080 check
    server app2 10.0.0.12:8080 check
```

NGINX에서는 `server`와 `upstream`으로 유사한 구조를 만듭니다.

```nginx
upstream app_back {
    server 10.0.0.11:8080;
    server 10.0.0.12:8080;
}

server {
    listen 443 ssl;
    location / {
        proxy_pass http://app_back;
    }
}
```

### 4-2) Backend Pool

Backend Pool은 같은 목적을 가진 서버 묶음입니다.

예를 들어 API 서버 3대를 하나의 Pool로 묶을 수 있습니다.

```text
Backend Pool: api
  ├─ api-1: 10.0.0.11:8080
  ├─ api-2: 10.0.0.12:8080
  └─ api-3: 10.0.0.13:8080
```

Load Balancer는 알고리즘과 Health Check 결과에 따라 Backend Pool 안의 서버 중 하나를 선택합니다.

Backend Pool에 서버가 하나도 없거나 모두 unhealthy이면 사용자는 보통 다음 오류를 받습니다.

- 502 Bad Gateway
- 503 Service Unavailable
- 504 Gateway Timeout

어떤 오류가 발생하는지는 Load Balancer 종류와 설정에 따라 다릅니다.

## 5. L4 Load Balancer

### 5-1) L4 Load Balancer란

L4 Load Balancer는 Transport Layer, 즉 TCP/UDP 계층에서 동작합니다.

L4에서 볼 수 있는 정보는 다음과 같습니다.

- Source IP
- Source Port
- Destination IP
- Destination Port
- Protocol: TCP 또는 UDP

L4 Load Balancer는 HTTP 내용을 해석하지 않습니다.

즉 다음 값은 보지 못합니다.

- HTTP Host
- HTTP Path
- HTTP Header
- Cookie
- HTTP Method
- HTTP Body

예시:

```text
Client -> 203.0.113.10:443
              ↓
        L4 Load Balancer
              ↓
        10.0.0.11:443
```

이때 Load Balancer는 단지 `203.0.113.10:443`으로 들어온 TCP 연결을 Backend `10.0.0.11:443` 또는 `10.0.0.12:443`으로 전달합니다.

### 5-2) L4 Load Balancer의 장점

L4 Load Balancer는 단순하고 빠릅니다.

장점은 다음과 같습니다.

- HTTP가 아닌 TCP 서비스에도 사용 가능
- TLS를 해제하지 않아도 전달 가능
- 상대적으로 성능이 좋음
- Layer 7 파싱 비용이 적음
- gRPC, HTTPS, SSH, DB, MQ 등 다양한 TCP 서비스에 사용 가능

사용 예시:

| 서비스 | 포트 | L4 LB 사용 가능 여부 |
|---|---:|---|
| HTTPS | 443 | 가능 |
| SSH | 22 | 가능 |
| PostgreSQL | 5432 | 가능 |
| MySQL | 3306 | 가능 |
| Redis | 6379 | 가능 |
| Kafka | 9092 | 가능 |
| Kubernetes API Server | 6443 | 가능 |
| DNS UDP | 53 | 가능 |

### 5-3) L4 Load Balancer의 한계

L4 Load Balancer는 HTTP 내용을 알 수 없기 때문에 다음을 할 수 없습니다.

```text
app.example.com/api     -> api backend
app.example.com/admin   -> admin backend
Host: grafana.example.com -> grafana backend
```

이런 기능은 L7 Load Balancer가 필요합니다.

또한 TLS Passthrough 상태에서는 내부 HTTP Header를 볼 수 없습니다.

```text
Client --HTTPS--> L4 LB --HTTPS--> Backend
```

이 구조에서는 Load Balancer가 암호화된 데이터를 그대로 전달하기 때문에 Host Header, Cookie, Path를 해석하지 못합니다.

단, TLS Handshake의 SNI 정보는 암호화 전 단계에서 확인 가능한 경우가 있어 SNI 기반 TCP 라우팅을 지원하는 제품도 있습니다. 그러나 일반적인 HTTP Header 기반 라우팅과는 다릅니다.

### 5-4) L4 Load Balancer 트러블슈팅 관점

L4 문제는 보통 다음 질문으로 좁힙니다.

```text
1. VIP까지 TCP 연결이 되는가?
2. LB가 Backend로 TCP 연결을 만들 수 있는가?
3. Backend port가 열려 있는가?
4. 방화벽이나 Security Group이 막고 있지 않은가?
5. Health Check가 Backend를 healthy로 보고 있는가?
6. SNAT/DNAT 때문에 return path가 깨지지 않았는가?
```

확인 명령어:

```bash
# VIP TCP 연결 확인
nc -vz app.example.com 443

# TLS까지 확인
openssl s_client -connect app.example.com:443 -servername app.example.com

# Backend 직접 확인
nc -vz 10.0.0.11 8080

# Linux listen port 확인
ss -lntp

# 패킷 확인
sudo tcpdump -i any host 10.0.0.11 and port 8080
```

## 6. L7 Load Balancer

### 6-1) L7 Load Balancer란

L7 Load Balancer는 Application Layer, 즉 HTTP/HTTPS 계층을 해석합니다.

L7 Load Balancer는 다음 정보를 볼 수 있습니다.

- Host Header
- URL Path
- Query String
- HTTP Method
- HTTP Header
- Cookie
- HTTP Status Code
- HTTP Body 일부 또는 전체
- TLS SNI
- 인증 관련 Header

예시:

```text
Host: api.example.com
Path: /v1/users
Method: GET
Cookie: session_id=abc123
```

이 정보를 기반으로 다양한 라우팅이 가능합니다.

```text
api.example.com       -> api backend
admin.example.com     -> admin backend
app.example.com/api   -> api backend
app.example.com/web   -> web backend
```

### 6-2) L7 Load Balancer의 장점

L7 Load Balancer는 HTTP 기반 서비스 운영에 강합니다.

가능한 기능은 다음과 같습니다.

- Host 기반 라우팅
- Path 기반 라우팅
- Header 기반 라우팅
- Cookie 기반 Sticky Session
- HTTP Redirect
- HTTP Rewrite
- CORS 제어
- Rate Limit
- WAF 연동
- Authentication 연동
- TLS Termination
- HTTP Health Check
- Compression
- Caching
- Access Log
- Request ID 생성
- Trace Header 전달

Ingress Controller가 L7 Reverse Proxy로 동작하는 이유도 이 때문입니다.

### 6-3) L7 Load Balancer의 단점

L7 Load Balancer는 L4보다 더 많은 일을 하므로 더 복잡합니다.

단점은 다음과 같습니다.

- HTTP 파싱 비용 존재
- 설정이 복잡함
- Header/Path/Rewrite 오류 가능성 큼
- TLS Termination 시 인증서 관리 필요
- X-Forwarded-* Header 처리 필요
- WebSocket/gRPC 같은 프로토콜별 설정 필요
- Timeout 불일치 시 502/504 발생 가능

따라서 L7 Load Balancer는 기능이 강력하지만 장애 원인도 다양합니다.

## 7. L4와 L7 비교

| 구분 | L4 Load Balancer | L7 Load Balancer |
|---|---|---|
| 기준 계층 | TCP/UDP | HTTP/HTTPS |
| 라우팅 기준 | IP, Port | Host, Path, Header, Cookie |
| HTTP 해석 | 불가 | 가능 |
| TLS 처리 | Passthrough 중심 | Termination 가능 |
| 성능 | 상대적으로 단순하고 빠름 | 기능 많지만 처리 비용 증가 |
| 사용 예시 | NLB, HAProxy TCP mode, F5 L4 | ALB, NGINX, HAProxy HTTP mode, Ingress |
| 장애 분석 | TCP 연결 중심 | HTTP 상태 코드, Header, Backend 응답 중심 |

실무에서는 L4와 L7을 조합하는 경우가 많습니다.

```text
Client
  ↓
L4 Load Balancer
  ↓
Ingress Controller(L7)
  ↓
Service
  ↓
Pod
```

예를 들어 Kubernetes 온프레미스 환경에서는 다음 구조가 자주 사용됩니다.

```text
Client
  ↓
MetalLB 또는 외부 HAProxy
  ↓
NGINX Ingress Controller
  ↓
Kubernetes Service
  ↓
Pod
```

## 8. Load Balancing Algorithm

### 8-1) Round Robin

Round Robin은 요청을 순서대로 분산합니다.

```text
1번째 요청 -> Server A
2번째 요청 -> Server B
3번째 요청 -> Server C
4번째 요청 -> Server A
```

장점:

- 단순함
- 서버 성능이 비슷할 때 적합
- 예측 가능

단점:

- 서버별 실제 부하를 고려하지 않음
- 요청 처리 시간이 다르면 불균형 가능
- 긴 연결이 많은 서비스에는 부적합할 수 있음

예를 들어 Server A에는 오래 걸리는 요청이 몰리고, Server B에는 짧은 요청만 몰릴 수 있습니다. 요청 수는 같아도 실제 부하는 다를 수 있습니다.

### 8-2) Weighted Round Robin

Weighted Round Robin은 서버별 가중치를 부여합니다.

```text
Server A weight 5
Server B weight 3
Server C weight 1
```

이 경우 A가 B보다 더 많은 요청을 받고, C는 적게 받습니다.

사용 사례:

- 서버 사양이 다를 때
- 신규 서버에 트래픽을 조금만 보내고 싶을 때
- Blue/Green 또는 Canary 배포와 유사한 비율 제어를 하고 싶을 때

예시:

```text
A: 고사양 서버
B: 중간 사양 서버
C: 테스트 서버
```

이런 경우 C에 동일한 트래픽을 보내면 문제가 생길 수 있으므로 weight를 낮춥니다.

### 8-3) Least Connection

Least Connection은 현재 연결 수가 가장 적은 서버를 선택합니다.

```text
Server A: 100 connections
Server B: 20 connections
Server C: 30 connections

다음 요청 -> Server B
```

장점:

- 긴 연결이 많은 서비스에 적합
- WebSocket, gRPC streaming, DB proxy 등에 유리할 수 있음
- 단순 Round Robin보다 실제 부하를 더 반영

단점:

- 연결 수가 실제 CPU 부하와 항상 일치하지 않음
- 짧은 요청이 매우 많은 서비스에서는 효과가 제한적일 수 있음

### 8-4) Source Hash

Source Hash는 클라이언트 IP 등을 해시해서 Backend를 선택합니다.

```text
hash(client_ip) -> Server B
```

장점:

- 같은 클라이언트가 같은 서버로 갈 가능성이 높음
- Cookie 없이 간단한 Sticky Session 구현 가능

단점:

- NAT 뒤에 많은 사용자가 있으면 한 서버로 몰릴 수 있음
- 모바일 환경처럼 IP가 자주 바뀌면 세션 유지가 깨질 수 있음
- 프록시 뒤에서는 실제 사용자 IP가 아니라 프록시 IP 기준이 될 수 있음

### 8-5) Consistent Hash

Consistent Hash는 서버 추가/삭제 시 전체 매핑 변경을 최소화하는 방식입니다.

일반 Hash는 서버 수가 바뀌면 대부분의 매핑이 바뀔 수 있습니다.

```text
hash(key) % server_count
```

서버 수가 3에서 4로 바뀌면 많은 key의 대상 서버가 바뀝니다.

Consistent Hash는 이 문제를 줄입니다.

사용 사례:

- 캐시 서버
- CDN
- 분산 저장소
- 세션 분산
- Kafka-like partition routing
- 특정 tenant를 특정 backend에 유지

### 8-6) Maglev

Maglev는 Google에서 공개한 고성능 로드밸런싱 알고리즘으로 알려져 있으며, Cilium 같은 eBPF 기반 네트워킹에서도 관련 개념이 등장합니다.

Maglev의 목적은 다음과 같습니다.

- Backend 추가/삭제 시 연결 재분배 최소화
- 빠른 lookup
- 균등한 분산
- 대규모 서비스에 적합한 일관성 있는 분산

Kubernetes에서 Cilium kube-proxy replacement를 사용할 때 `loadBalancer.algorithm=maglev` 같은 설정을 볼 수 있습니다.

실무적으로는 다음 정도로 이해하면 됩니다.

```text
Round Robin:
매 요청 순서대로 분산

Source Hash:
같은 source를 같은 backend로 유도

Consistent Hash / Maglev:
backend 변화가 있어도 기존 연결 매핑 변화를 최소화
```

## 9. TCP Load Balancing

### 9-1) TCP Load Balancing 동작 흐름

TCP Load Balancer는 클라이언트의 TCP 연결을 Backend로 전달합니다.

```text
Client
  ↓ TCP SYN
Load Balancer
  ↓ TCP 연결
Backend Server
```

구현 방식은 제품과 설정에 따라 다릅니다.

대표적으로 다음 방식이 있습니다.

1. TCP Proxy 방식
2. NAT 방식
3. DSR 방식

### 9-2) TCP Proxy 방식

TCP Proxy 방식에서는 Load Balancer가 클라이언트와 Backend 사이에 두 개의 TCP 연결을 만듭니다.

```text
Client ── TCP Connection 1 ── Load Balancer ── TCP Connection 2 ── Backend
```

장점:

- LB가 연결 상태를 명확히 제어 가능
- Health Check, timeout, retry 적용 용이
- TLS Termination 또는 Passthrough 선택 가능

단점:

- LB가 양쪽 연결을 모두 처리하므로 부하가 큼
- 원본 IP가 Backend에 직접 보이지 않을 수 있음
- 원본 IP 전달을 위해 X-Forwarded-For 또는 PROXY Protocol 필요

### 9-3) NAT 방식

NAT 방식에서는 Load Balancer가 목적지 IP 또는 출발지 IP를 변환합니다.

대표적으로 DNAT와 SNAT가 사용됩니다.

```text
Client -> VIP
DNAT: VIP -> Backend IP
```

그리고 응답이 다시 LB를 거치도록 SNAT을 같이 사용하는 경우도 있습니다.

```text
Client -> LB -> Backend
Client <- LB <- Backend
```

장점:

- 구현이 단순한 편
- 네트워크 계층에서 처리 가능

단점:

- 원본 IP가 사라질 수 있음
- return path 설계가 중요
- conntrack 부하가 발생할 수 있음

### 9-4) DSR

DSR은 Direct Server Return의 약자입니다.

요청은 Load Balancer를 거치지만 응답은 Backend가 클라이언트에게 직접 보냅니다.

```text
Request:
Client -> Load Balancer -> Backend

Response:
Backend -> Client
```

장점:

- 응답 트래픽이 LB를 거치지 않아 성능상 유리
- 다운로드 트래픽이 큰 서비스에서 유리

단점:

- 네트워크 설계가 복잡함
- Backend가 VIP를 인식해야 함
- L7 기능 적용이 제한적
- Troubleshooting 난이도 높음

## 10. UDP Load Balancing

UDP는 TCP처럼 연결이 없습니다.

따라서 UDP Load Balancing은 TCP보다 상태 추적이 어렵습니다.

사용 사례:

- DNS
- DHCP
- RADIUS
- Syslog
- VoIP
- 게임 서버
- QUIC / HTTP/3

UDP Load Balancing에서 중요한 점은 다음입니다.

- 응답이 같은 경로로 돌아오는가?
- UDP timeout이 너무 짧지 않은가?
- Backend 상태를 어떻게 판단하는가?
- 패킷 손실이 허용 가능한가?
- 재전송은 애플리케이션이 처리하는가?

예를 들어 DNS UDP 53은 요청/응답이 짧기 때문에 비교적 단순합니다.

하지만 QUIC/HTTP/3는 UDP 위에서 연결 개념을 애플리케이션 계층에서 구현하므로 LB가 QUIC 특성을 이해해야 하는 경우가 있습니다.

## 11. Reverse Proxy

### 11-1) Reverse Proxy란

Reverse Proxy는 서버 앞에 위치해서 클라이언트 요청을 대신 받아 내부 서버로 전달하는 프록시입니다.

```text
Client
  ↓
Reverse Proxy
  ↓
Backend Server
```

사용자는 실제 Backend 서버를 모릅니다. 사용자는 Reverse Proxy만 바라봅니다.

대표 제품:

- NGINX
- HAProxy
- Traefik
- Envoy
- Apache HTTPD
- Kubernetes Ingress Controller

### 11-2) Reverse Proxy의 역할

Reverse Proxy는 다음 역할을 수행할 수 있습니다.

- TLS Termination
- HTTP Routing
- Load Balancing
- Header Rewrite
- URL Rewrite
- Compression
- Caching
- Rate Limit
- IP Whitelist
- Basic Auth
- OAuth/OIDC 연동
- WAF 연동
- Access Log
- Metrics
- Request ID 부여

Kubernetes Ingress Controller는 Kubernetes에 맞게 동작하는 Reverse Proxy라고 볼 수 있습니다.

```text
Client
  ↓
Ingress Controller
  ↓
Service
  ↓
Pod
```

### 11-3) Reverse Proxy와 Backend 애플리케이션의 관계

Reverse Proxy 뒤에 있는 애플리케이션은 다음 정보를 직접 알기 어렵습니다.

- 실제 Client IP
- 원래 요청 Scheme: http 또는 https
- 원래 Host
- 원래 Port

왜냐하면 애플리케이션 입장에서는 요청이 Proxy에서 온 것처럼 보이기 때문입니다.

그래서 다음 Header가 중요합니다.

```http
X-Forwarded-For: 203.0.113.10
X-Forwarded-Host: app.example.com
X-Forwarded-Proto: https
X-Forwarded-Port: 443
```

이 값이 잘못되면 다음 문제가 발생할 수 있습니다.

- 로그인 Redirect가 `http://`로 생성됨
- OAuth Callback URL 불일치
- Keycloak redirect_uri 오류
- Harbor 로그인 실패
- Grafana root_url 문제
- 무한 Redirect Loop
- Access Log의 Client IP가 모두 Proxy IP로 기록됨

## 12. Forward Proxy

### 12-1) Forward Proxy란

Forward Proxy는 클라이언트 앞에 위치해서 클라이언트의 외부 요청을 대신 수행합니다.

```text
Client
  ↓
Forward Proxy
  ↓
Internet
```

Reverse Proxy는 서버 앞에 있고, Forward Proxy는 클라이언트 앞에 있습니다.

| 구분 | Forward Proxy | Reverse Proxy |
|---|---|---|
| 위치 | 클라이언트 앞 | 서버 앞 |
| 보호 대상 | 클라이언트 | 서버 |
| 대표 목적 | 외부 접속 제어 | 내부 서비스 노출 |
| 예시 | 회사 인터넷 프록시, Squid | NGINX, HAProxy, Ingress |
| 사용자가 인식하는 대상 | 외부 사이트 | 서비스 도메인 |

### 12-2) Forward Proxy 사용 사례

Forward Proxy는 다음 상황에서 사용합니다.

- 회사 내부망에서 인터넷 접근 통제
- 특정 사이트 차단
- 다운로드 파일 검사
- 외부 접속 로그 감사
- 고정 출구 IP 제공
- 패키지 저장소 캐싱
- 폐쇄망 환경에서 제한적 인터넷 접근 제공

예시:

```bash
export HTTP_PROXY=http://proxy.company.local:3128
export HTTPS_PROXY=http://proxy.company.local:3128
```

이 설정이 필요한 환경에서는 `curl`, `apt`, `yum`, `docker pull`, `helm repo add` 같은 명령어가 프록시를 통해 외부로 나갑니다.

### 12-3) Kubernetes에서 Forward Proxy가 필요한 경우

Kubernetes 노드가 인터넷에 직접 접근할 수 없는 환경에서는 Forward Proxy가 필요할 수 있습니다.

예시:

- 이미지 Pull
- Helm Chart 다운로드
- OS 패키지 설치
- 외부 API 호출
- Git repository 접근

이 경우 다음 설정을 점검해야 합니다.

- Node의 HTTP_PROXY/HTTPS_PROXY
- containerd 또는 Docker proxy 설정
- Pod 환경변수
- NO_PROXY 설정
- Kubernetes API Server, ClusterIP, Pod CIDR이 proxy로 나가지 않도록 예외 처리

특히 `NO_PROXY` 설정을 잘못하면 클러스터 내부 통신까지 외부 proxy로 보내려 해서 장애가 발생할 수 있습니다.

예시:

```text
NO_PROXY=localhost,127.0.0.1,.svc,.cluster.local,10.96.0.0/12,10.244.0.0/16,192.168.10.0/24
```

## 13. Transparent Proxy

Transparent Proxy는 클라이언트가 프록시를 명시적으로 설정하지 않아도 중간에서 트래픽을 가로채 처리하는 프록시입니다.

```text
Client
  ↓
Transparent Proxy
  ↓
Server
```

사용자는 프록시 존재를 모를 수 있습니다.

사용 사례:

- 보안 장비
- 네트워크 필터링
- Service Mesh sidecar
- egress gateway
- 트래픽 미러링
- 기업망 HTTP inspection

Service Mesh의 Envoy Sidecar는 애플리케이션이 특별히 프록시를 지정하지 않아도 iptables 또는 eBPF 규칙으로 트래픽을 Envoy로 우회시킬 수 있습니다.

```text
Application Container
  ↓
iptables redirect
  ↓
Envoy Sidecar
  ↓
Network
```

이 구조를 이해하지 못하면 Service Mesh 환경에서 "애플리케이션은 직접 호출했는데 왜 Envoy 로그에 찍히는가"를 이해하기 어렵습니다.

## 14. TLS 처리 방식

Load Balancer와 Proxy에서 TLS 처리는 매우 중요합니다.

HTTPS 요청이 들어왔을 때 LB가 TLS를 어디에서 해제하느냐에 따라 구조가 달라집니다.

## 14-1) TLS Offloading / TLS Termination

TLS Offloading 또는 TLS Termination은 LB 또는 Proxy가 HTTPS를 해제하고 내부로 HTTP를 전달하는 방식입니다.

```text
Client --HTTPS--> Load Balancer --HTTP--> Backend
```

장점:

- Backend는 TLS 처리 부담이 줄어듦
- 인증서 관리를 LB에서 중앙화 가능
- L7 라우팅 가능
- Header 조작 가능
- HTTP Access Log 확보 가능

단점:

- LB와 Backend 사이 구간은 암호화되지 않을 수 있음
- 내부망 보안 요구가 높은 경우 부적합
- Backend가 원래 요청이 HTTPS였는지 모르므로 `X-Forwarded-Proto` 필요

예시:

```http
X-Forwarded-Proto: https
```

이 Header가 없으면 Backend가 `http://` redirect를 생성할 수 있습니다.

Keycloak, Harbor, Grafana 같은 서비스는 이 값이 틀리면 로그인 Redirect 문제가 자주 발생합니다.

## 14-2) SSL Bridging / Re-encrypt

SSL Bridging 또는 Re-encrypt는 LB에서 TLS를 해제한 뒤 Backend로 다시 HTTPS 연결을 만드는 방식입니다.

```text
Client --HTTPS--> Load Balancer --HTTPS--> Backend
```

장점:

- LB가 HTTP 내용을 볼 수 있어 L7 기능 사용 가능
- 내부 구간도 암호화 가능
- 보안 요구사항이 높은 환경에 적합

단점:

- LB와 Backend 양쪽 TLS 설정 필요
- 인증서 검증, SNI, CA 설정이 복잡해짐
- 성능 비용 증가

사용 사례:

- 금융권
- 공공기관
- 내부망도 암호화 요구가 있는 환경
- Zero Trust 네트워크

## 14-3) SSL Passthrough

SSL Passthrough는 LB가 TLS를 해제하지 않고 암호화된 TCP 스트림을 그대로 Backend로 전달하는 방식입니다.

```text
Client --HTTPS--> Load Balancer --HTTPS--> Backend
```

여기서 LB는 HTTP 내용을 볼 수 없습니다.

장점:

- Backend가 직접 인증서 처리
- End-to-End TLS 유지
- LB가 평문 HTTP를 보지 않음

단점:

- HTTP Header/Path 기반 라우팅 불가
- L7 기능 대부분 사용 불가
- Access Log가 제한적
- Sticky Cookie 사용 어려움
- WAF, Rate Limit 적용 제한

단, TLS SNI를 기준으로 어느 Backend로 보낼지는 가능한 경우가 있습니다.

예:

```text
SNI: harbor.example.com -> harbor backend
SNI: grafana.example.com -> grafana backend
```

하지만 이것은 HTTP Host Header 라우팅과 다릅니다. TLS Handshake 시점의 SNI 정보만 사용하는 것입니다.

## 15. Connection Lifecycle

### 15-1) HTTP 요청 하나의 흐름

L7 Reverse Proxy에서 HTTP 요청은 다음 흐름을 거칩니다.

```text
Client
  ↓
TCP Handshake
  ↓
TLS Handshake
  ↓
HTTP Request
  ↓
Load Balancer
  ↓
Backend 선택
  ↓
Backend TCP 연결
  ↓
Backend HTTP Request
  ↓
HTTP Response
  ↓
Client로 응답 반환
```

여기서 연결은 크게 두 구간으로 나뉩니다.

```text
Client-side connection:
Client ↔ Load Balancer

Server-side connection:
Load Balancer ↔ Backend
```

이 두 연결은 서로 다른 TCP 연결일 수 있습니다.

### 15-2) Connection Reuse

매 요청마다 Backend와 TCP 연결을 새로 만들면 비용이 큽니다.

```text
요청 1 -> TCP 연결 생성 -> 요청 처리 -> 연결 종료
요청 2 -> TCP 연결 생성 -> 요청 처리 -> 연결 종료
```

이렇게 하면 다음 비용이 반복됩니다.

- TCP 3-Way Handshake
- TLS Handshake
- Kernel socket 생성
- conntrack entry 생성
- Backend accept 처리

그래서 Proxy는 Backend와의 연결을 재사용합니다.

```text
요청 1 -> 기존 Backend 연결 사용
요청 2 -> 기존 Backend 연결 재사용
요청 3 -> 기존 Backend 연결 재사용
```

이것이 Keepalive 또는 Connection Reuse입니다.

### 15-3) Connection Pool

Connection Pool은 Backend별로 유지하는 연결 묶음입니다.

```text
LB -> Backend A: connection pool 100개
LB -> Backend B: connection pool 100개
LB -> Backend C: connection pool 100개
```

장점:

- 지연 시간 감소
- Backend 연결 생성 비용 감소
- TLS 비용 감소
- 높은 처리량 확보

주의점:

- Pool 크기가 너무 작으면 대기 발생
- 너무 크면 Backend connection 고갈
- idle timeout 불일치 시 502 발생 가능
- Backend가 연결을 먼저 끊으면 LB가 재사용 시 실패할 수 있음

### 15-4) Timeout 불일치

Load Balancer 장애에서 timeout 불일치는 매우 흔합니다.

예를 들어:

```text
LB backend keepalive timeout: 60s
Backend server keepalive timeout: 30s
```

Backend가 30초 후 연결을 끊었는데 LB는 그 연결을 살아 있다고 생각할 수 있습니다. 다음 요청을 그 연결로 보내면 실패할 수 있습니다.

결과:

- 502 Bad Gateway
- upstream prematurely closed connection
- connection reset by peer
- broken pipe

따라서 LB timeout과 Backend timeout은 함께 설계해야 합니다.

## 16. Health Check

### 16-1) Health Check란

Health Check는 Backend 서버가 요청을 받을 수 있는 상태인지 확인하는 절차입니다.

Load Balancer는 Health Check 결과를 기반으로 Backend를 Pool에 포함하거나 제외합니다.

```text
Healthy Backend   -> 트래픽 전달
Unhealthy Backend -> 트래픽 제외
```

Health Check가 잘못 설계되면 다음 문제가 발생합니다.

- 죽은 서버에 계속 트래픽 전달
- 정상 서버를 장애로 오판
- 일시적 지연으로 전체 서비스 장애
- 배포 중 서버가 너무 빨리 투입됨
- DB 장애가 모든 Web 서버 unhealthy로 전파됨

### 16-2) Active Health Check

Active Check는 Load Balancer가 주기적으로 Backend에 요청을 보내 상태를 확인하는 방식입니다.

```text
Load Balancer -> GET /healthz -> Backend
```

예시:

```text
GET /healthz
200 OK
```

정상 조건은 설정에 따라 다릅니다.

- TCP 연결 성공
- HTTP 200 응답
- 특정 문자열 포함
- TLS Handshake 성공
- gRPC Health Check 성공

장점:

- 실제 트래픽이 없어도 상태 확인 가능
- 장애 서버를 사전에 제외 가능
- 배포 자동화와 잘 맞음

단점:

- Health Check endpoint 설계가 필요
- 너무 자주 검사하면 부하 발생
- 너무 느슨하면 장애를 못 잡음
- 너무 엄격하면 정상 서버도 제외될 수 있음

### 16-3) Passive Health Check

Passive Check는 실제 사용자 트래픽 결과를 보고 Backend 상태를 판단하는 방식입니다.

예를 들어 Backend가 계속 5xx를 반환하거나 connection reset을 발생시키면 unhealthy로 판단할 수 있습니다.

장점:

- 실제 사용자 요청 기준
- 별도 health endpoint가 없어도 일부 판단 가능

단점:

- 실제 트래픽이 있어야 판단 가능
- 장애 감지가 늦을 수 있음
- 일시적인 애플리케이션 오류와 서버 장애를 구분하기 어려움

### 16-4) TCP Health Check

TCP Health Check는 특정 포트로 TCP 연결이 가능한지만 확인합니다.

```text
LB -> Backend:8080 TCP connect
```

장점:

- 단순함
- HTTP가 아닌 서비스에도 사용 가능
- 빠름

단점:

- 애플리케이션 정상 여부는 모름
- 포트만 열려 있어도 healthy로 판단 가능

예를 들어 웹 서버 프로세스는 떠 있지만 DB 연결이 모두 실패하는 상태일 수 있습니다. TCP Health Check는 이 상태를 정상으로 볼 수 있습니다.

### 16-5) HTTP Health Check

HTTP Health Check는 특정 URL을 호출해서 응답 상태를 확인합니다.

```text
GET /healthz
GET /ready
GET /actuator/health
```

응답 예시:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"status":"UP"}
```

실무에서는 보통 다음 두 개를 구분합니다.

| Endpoint | 의미 |
|---|---|
| `/livez` 또는 `/healthz` | 프로세스가 살아 있는가 |
| `/readyz` 또는 `/ready` | 실제 트래픽을 받을 준비가 되었는가 |

Kubernetes의 livenessProbe와 readinessProbe 개념과 유사합니다.

### 16-6) Health Check 설계 기준

좋은 Health Check는 다음 조건을 만족해야 합니다.

- 빠르게 응답해야 함
- 너무 많은 dependency를 포함하지 않아야 함
- 트래픽 수신 가능 여부를 정확히 반영해야 함
- 배포 직후 warm-up 시간을 고려해야 함
- DB 장애를 전체 서비스 장애로 확대하지 않도록 신중히 설계해야 함

예를 들어 로그인 API 서비스의 readiness는 DB 연결이 필요할 수 있습니다. 하지만 정적 파일 서버의 readiness에 DB 연결을 넣는 것은 불필요합니다.

### 16-7) Rise/Fall Threshold

Health Check는 한 번 실패했다고 바로 unhealthy로 만들면 안 됩니다.

일시적인 네트워크 지연이나 순간적인 GC pause 때문에 한 번 실패할 수 있기 때문입니다.

그래서 보통 threshold를 둡니다.

```text
fall 3:
연속 3회 실패하면 unhealthy

rise 2:
연속 2회 성공하면 healthy
```

이 설정은 장애 감지 속도와 오탐 방지 사이의 균형입니다.

## 17. Connection Draining

### 17-1) Connection Draining이란

Connection Draining은 Backend 서버를 제거할 때 새 요청은 보내지 않되, 기존 연결은 일정 시간 유지하는 방식입니다.

```text
Backend A 상태 변경:
active -> draining

새 연결:
Backend B, C로 전달

기존 연결:
Backend A에서 마저 처리
```

### 17-2) 왜 필요한가

배포 중 서버를 바로 제거하면 기존 요청이 끊길 수 있습니다.

예를 들어 사용자가 파일 업로드 중인데 해당 Pod 또는 서버가 즉시 종료되면 업로드가 실패합니다.

```text
Client -> LB -> Backend A
파일 업로드 중

Backend A 즉시 종료
  ↓
Connection reset
```

Connection Draining을 사용하면 다음처럼 처리됩니다.

```text
Backend A는 새 요청을 받지 않음
기존 요청은 완료할 시간 제공
완료 후 종료
```

### 17-3) Kubernetes와 Draining

Kubernetes에서는 다음 개념들이 연결됩니다.

- readinessProbe 실패 시 Endpoint에서 제외
- preStop hook
- terminationGracePeriodSeconds
- Ingress Controller 또는 외부 LB의 endpoint 반영 지연
- keepalive connection 종료

정상적인 종료 흐름은 다음과 같습니다.

```text
1. Pod 종료 시작
2. readinessProbe 실패 또는 Ready=False
3. EndpointSlice에서 제거
4. Service/Ingress/LB가 더 이상 새 요청을 보내지 않음
5. preStop hook 실행
6. 기존 요청 완료
7. 컨테이너 종료
```

이 순서가 어긋나면 배포 중 502/503이 발생할 수 있습니다.

## 18. Sticky Session

### 18-1) Sticky Session이란

Sticky Session은 같은 클라이언트의 요청을 같은 Backend로 보내는 기능입니다.

```text
Client A -> Server 1
Client A -> Server 1
Client A -> Server 1
```

반대로 Sticky가 없으면 다음처럼 분산될 수 있습니다.

```text
Client A 요청 1 -> Server 1
Client A 요청 2 -> Server 2
Client A 요청 3 -> Server 3
```

### 18-2) Sticky Session이 필요한 경우

Sticky Session은 주로 세션 상태를 서버 메모리에 저장하는 애플리케이션에서 필요합니다.

예를 들어 로그인 세션이 Server 1 메모리에만 저장되어 있다면 Client A가 Server 2로 가는 순간 로그인 상태를 잃을 수 있습니다.

```text
Login -> Server 1 memory에 session 저장
다음 요청 -> Server 2
Server 2는 session 모름
  ↓
로그아웃된 것처럼 보임
```

### 18-3) Sticky Session 방식

대표 방식은 다음과 같습니다.

| 방식 | 설명 |
|---|---|
| Source IP | 클라이언트 IP 기준으로 같은 서버 선택 |
| Cookie | LB가 쿠키를 심고 해당 쿠키 기준으로 서버 선택 |
| Header | 특정 Header 값을 기준으로 서버 선택 |
| URL Parameter | 특정 query/path 값을 기준으로 서버 선택 |

### 18-4) Source IP 기반 Sticky의 문제

회사망, 통신사망, NAT 환경에서는 수많은 사용자가 하나의 공인 IP를 공유할 수 있습니다.

```text
User A
User B
User C
  ↓ NAT
203.0.113.10
```

Source IP 기반 Sticky를 사용하면 이 사용자들이 모두 같은 Backend로 갈 수 있습니다.

결과:

- 특정 서버 부하 집중
- 실제 사용자 단위 세션 분리 어려움
- 모바일 IP 변경 시 sticky 깨짐

### 18-5) Cookie 기반 Sticky

Cookie 기반 Sticky는 L7 Load Balancer가 쿠키를 사용해 Backend를 고정합니다.

```http
Set-Cookie: SERVERID=server1
```

다음 요청:

```http
Cookie: SERVERID=server1
```

Load Balancer는 이 값을 보고 Server 1로 라우팅합니다.

장점:

- 사용자 단위 제어가 Source IP보다 정확함
- HTTP 서비스에 적합

단점:

- L7에서만 가능
- 쿠키 보안 설정 필요
- 서버 장애 시 세션 복구 문제
- 무상태 구조보다 운영 복잡도 높음

### 18-6) 권장 구조

가능하면 애플리케이션을 Stateless하게 만들고, 세션은 외부 저장소에 두는 것이 좋습니다.

```text
Client
  ↓
Load Balancer
  ↓
App Pod A/B/C
  ↓
Redis 또는 DB Session Store
```

이 구조에서는 어느 Pod로 가도 로그인 상태가 유지됩니다.

## 19. Session Persistence

Session Persistence는 같은 사용자의 상태를 지속적으로 유지하는 설계 전체를 의미합니다.

Load Balancer 관점에서는 Sticky Session이고, 애플리케이션 관점에서는 세션 저장 구조입니다.

| 방식 | 설명 | 권장도 |
|---|---|---|
| 서버 메모리 세션 | 각 서버 메모리에 session 저장 | 낮음 |
| Sticky Session | LB가 같은 서버로 고정 | 제한적 |
| Redis Session | 외부 Redis에 session 저장 | 높음 |
| JWT | 클라이언트가 토큰 보유 | 상황에 따라 |
| DB Session | DB에 session 저장 | 가능하나 부하 고려 |

Kubernetes에서는 Pod가 언제든 재시작될 수 있으므로 서버 메모리에 중요한 세션을 저장하는 구조는 불안정합니다.

## 20. Retry

### 20-1) Retry란

Retry는 요청이 실패했을 때 Load Balancer 또는 Proxy가 다른 Backend로 다시 시도하는 동작입니다.

```text
Request -> Backend A 실패
Retry -> Backend B 성공
```

장점:

- 일시적 장애를 사용자에게 숨길 수 있음
- Backend 하나가 순간적으로 실패해도 성공률 향상

단점:

- 지연 시간 증가
- 중복 처리 위험
- Backend 장애를 더 악화시킬 수 있음
- POST, 결제, 주문 생성 같은 요청에는 위험

### 20-2) Retry에 안전한 요청과 위험한 요청

GET 요청은 보통 재시도하기 쉽습니다.

```http
GET /products/1
```

하지만 POST 요청은 재시도하면 중복 생성이 발생할 수 있습니다.

```http
POST /orders
```

첫 번째 요청이 실제로는 서버에서 처리되었지만 응답만 실패한 상황이라면, Retry가 주문을 두 번 만들 수 있습니다.

따라서 Retry 정책은 HTTP Method와 애플리케이션 idempotency 설계를 함께 고려해야 합니다.

## 21. Circuit Breaker

Circuit Breaker는 장애가 난 Backend로 계속 요청을 보내지 않도록 차단하는 패턴입니다.

전기 회로 차단기처럼 장애가 감지되면 회로를 열어 요청을 막습니다.

상태는 보통 다음과 같습니다.

```text
Closed:
정상 요청 허용

Open:
요청 차단

Half-Open:
일부 요청만 보내 회복 여부 확인
```

Load Balancer 자체보다 Envoy, Istio, Service Mesh, API Gateway 영역에서 자주 등장합니다.

사용 이유:

- 장애 서비스로 요청 폭주 방지
- 장애 전파 차단
- 빠른 실패 처리
- 전체 시스템 보호

예시:

```text
Order Service -> Payment Service 장애
Circuit Breaker Open
  ↓
Payment 요청 즉시 실패 처리
  ↓
Order Service Thread 고갈 방지
```

## 22. Failover

### 22-1) Failover란

Failover는 장애 발생 시 정상 시스템으로 자동 전환하는 동작입니다.

Load Balancer에서는 보통 두 층위에서 Failover를 봅니다.

1. Backend Server Failover
2. Load Balancer 자체 Failover

### 22-2) Backend Failover

Backend 서버 중 하나가 장애 나면 LB가 해당 서버를 제외합니다.

```text
Before:
LB -> A, B, C

Server B 장애

After:
LB -> A, C
```

이 과정은 Health Check로 이루어집니다.

### 22-3) Load Balancer 자체 Failover

Load Balancer도 장애 날 수 있습니다.

그래서 운영 환경에서는 LB 자체를 이중화합니다.

```text
Client
  ↓
VIP
  ↓
LB Active
LB Standby
```

Active LB가 장애 나면 Standby LB가 VIP를 승계합니다.

대표 기술:

- VRRP
- Keepalived
- Pacemaker
- 클라우드 LB 자체 HA
- Anycast 기반 HA

### 22-4) Active-Active와 Active-Standby

| 구성 | 설명 | 장점 | 단점 |
|---|---|---|---|
| Active-Active | 여러 LB가 동시에 처리 | 자원 활용 좋음 | 설계 복잡 |
| Active-Standby | 하나만 Active, 하나는 대기 | 단순함 | Standby 자원 유휴 |

온프레미스에서는 Keepalived + HAProxy 조합으로 Active-Standby 구성을 많이 사용합니다.

## 23. Fail Open과 Fail Closed

장애 시 정책도 중요합니다.

### 23-1) Fail Open

Fail Open은 검사 시스템이 장애 나도 트래픽을 통과시키는 방식입니다.

장점:

- 서비스 가용성 유지

단점:

- 보안 위험 증가

사용 예:

- 보안 검사 장비 장애 시 서비스 중단을 피해야 하는 경우

### 23-2) Fail Closed

Fail Closed는 검사 시스템이 장애 나면 트래픽을 차단하는 방식입니다.

장점:

- 보안성 유지

단점:

- 서비스 중단 가능

사용 예:

- 인증/인가가 반드시 필요한 시스템
- 금융/보안 시스템
- 정책 위반 트래픽을 절대 허용하면 안 되는 환경

## 24. X-Forwarded-For

### 24-1) X-Forwarded-For가 필요한 이유

Reverse Proxy를 거치면 Backend는 클라이언트 IP를 직접 보지 못할 수 있습니다.

```text
Client: 203.0.113.10
Proxy: 10.0.0.5
Backend: 10.0.0.11
```

Backend 입장에서는 요청 출발지가 Proxy IP인 `10.0.0.5`로 보일 수 있습니다.

그래서 Proxy는 원본 IP를 Header에 담아 전달합니다.

```http
X-Forwarded-For: 203.0.113.10
```

### 24-2) 여러 Proxy를 거친 경우

여러 Proxy를 거치면 값이 누적됩니다.

```http
X-Forwarded-For: 203.0.113.10, 10.0.0.5, 10.0.0.6
```

일반적으로 왼쪽 첫 번째 값이 원래 클라이언트 IP로 간주됩니다.

하지만 주의해야 합니다.

클라이언트도 임의로 `X-Forwarded-For` Header를 보낼 수 있습니다.

따라서 Backend는 아무 Header나 믿으면 안 됩니다. 신뢰할 수 있는 Proxy가 설정한 값만 신뢰해야 합니다.

### 24-3) 관련 Header

```http
X-Forwarded-For: 203.0.113.10
X-Forwarded-Host: app.example.com
X-Forwarded-Proto: https
X-Forwarded-Port: 443
```

각 의미는 다음과 같습니다.

| Header | 의미 |
|---|---|
| X-Forwarded-For | 원본 Client IP |
| X-Forwarded-Host | 원래 Host |
| X-Forwarded-Proto | 원래 Scheme |
| X-Forwarded-Port | 원래 Port |

Keycloak, Harbor, Grafana, ArgoCD 같은 서비스는 이 Header를 올바르게 인식해야 Redirect URL을 정확히 생성합니다.

## 25. PROXY Protocol

### 25-1) PROXY Protocol이란

PROXY Protocol은 TCP 연결 시작 부분에 원본 Client 정보를 추가로 전달하는 프로토콜입니다.

X-Forwarded-For는 HTTP Header입니다. 따라서 HTTP를 해석할 수 있는 L7에서만 사용할 수 있습니다.

하지만 TCP 서비스에서는 HTTP Header가 없습니다.

예를 들어:

- PostgreSQL
- MySQL
- Redis
- Kafka
- SMTP
- raw TCP service

이런 서비스에서도 원본 Client IP를 전달해야 할 수 있습니다.

이때 PROXY Protocol을 사용합니다.

### 25-2) PROXY Protocol 흐름

```text
Client
  ↓
Load Balancer
  ↓ PROXY Protocol Header + Original Payload
Backend
```

예시 형태:

```text
PROXY TCP4 203.0.113.10 10.0.0.11 52344 443
```

이 정보에는 다음이 포함됩니다.

- 원본 Source IP
- 원본 Destination IP
- Source Port
- Destination Port
- Protocol

### 25-3) X-Forwarded-For와 PROXY Protocol 차이

| 구분 | X-Forwarded-For | PROXY Protocol |
|---|---|---|
| 계층 | HTTP Header | TCP 연결 메타데이터 |
| 사용 대상 | HTTP/HTTPS | TCP/HTTP 모두 가능 |
| Backend 지원 필요 | HTTP 앱 또는 Proxy | PROXY Protocol 파싱 필요 |
| TLS Passthrough | 어려움 | 가능 |
| 대표 사용 | NGINX, Ingress, Web App | HAProxy, NLB, Envoy, TCP Proxy |

중요한 점은 Backend가 PROXY Protocol을 지원하도록 설정되어 있어야 한다는 것입니다.

Backend가 PROXY Protocol을 모르는 상태에서 이 Header가 들어오면, 첫 요청 바이트를 이상한 데이터로 인식하여 오류가 발생할 수 있습니다.

## 26. SNAT, DNAT, NAT Mode

### 26-1) DNAT

DNAT는 Destination NAT입니다. 목적지 IP를 바꿉니다.

```text
Client -> VIP 203.0.113.10
DNAT
Client -> Backend 10.0.0.11
```

Kubernetes Service도 ClusterIP를 Pod IP로 바꾼다는 점에서 DNAT와 유사한 흐름을 가집니다.

### 26-2) SNAT

SNAT는 Source NAT입니다. 출발지 IP를 바꿉니다.

```text
Client 203.0.113.10 -> Backend 10.0.0.11

SNAT 후:
LB 10.0.0.5 -> Backend 10.0.0.11
```

SNAT을 쓰면 Backend는 원본 Client IP가 아니라 LB IP를 봅니다.

장점:

- return path가 단순해짐
- Backend 응답이 반드시 LB로 돌아옴

단점:

- 원본 Client IP가 사라짐
- 로그 분석 어려움
- IP 기반 정책 적용 어려움

### 26-3) NAT Mode의 return path 문제

SNAT 없이 DNAT만 사용하는 경우 Backend는 클라이언트에게 직접 응답하려고 할 수 있습니다.

```text
Request:
Client -> LB -> Backend

Response:
Backend -> Client
```

이때 클라이언트는 자신이 `VIP`에 연결했다고 생각하는데 응답은 `Backend IP`에서 오면 TCP 연결이 깨질 수 있습니다.

그래서 네트워크 경로 설계가 중요합니다.

## 27. MetalLB

### 27-1) MetalLB란

MetalLB는 Bare Metal 또는 온프레미스 Kubernetes 환경에서 `type: LoadBalancer` Service를 사용할 수 있게 해주는 Load Balancer 구현체입니다.

클라우드 환경에서는 다음처럼 Service를 만들면 클라우드 Load Balancer가 생성됩니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
spec:
  type: LoadBalancer
```

하지만 온프레미스에는 AWS ELB 같은 관리형 Load Balancer가 없습니다.

이때 MetalLB가 외부 IP를 할당하고 광고합니다.

```text
Kubernetes Service type LoadBalancer
  ↓
MetalLB가 External IP 할당
  ↓
외부에서 해당 IP로 접근 가능
```

### 27-2) MetalLB L2 Mode

L2 Mode에서는 MetalLB가 ARP 또는 NDP를 사용해 IP 소유자를 알립니다.

```text
Who has 192.168.10.200?
  ↓
MetalLB Node replies:
192.168.10.200 is at node1 MAC
```

장점:

- 설정이 비교적 단순
- 별도 라우터 BGP 설정 불필요
- 작은 온프레미스 환경에서 사용하기 쉬움

단점:

- 하나의 IP는 한 노드가 담당
- 해당 노드 장애 시 failover 필요
- 대규모 트래픽 분산에는 한계

### 27-3) MetalLB BGP Mode

BGP Mode에서는 MetalLB가 라우터와 BGP 세션을 맺고 LoadBalancer IP 경로를 광고합니다.

장점:

- 라우터 기반 경로 제어 가능
- 여러 노드로 ECMP 분산 가능
- 대규모 환경에 적합

단점:

- 네트워크 장비와 BGP 지식 필요
- 설정 난이도 높음
- 네트워크 팀 협업 필요

### 27-4) MetalLB와 Ingress

온프레미스 Kubernetes에서 자주 쓰는 구조는 다음과 같습니다.

```text
Client
  ↓
MetalLB External IP
  ↓
Ingress Controller Service(type=LoadBalancer)
  ↓
Ingress Controller Pod
  ↓
Backend Service
  ↓
Pod
```

MetalLB는 L4 진입점을 제공하고, Ingress Controller는 L7 라우팅을 담당합니다.

## 28. AWS NLB와 ALB

### 28-1) AWS NLB

AWS Network Load Balancer는 L4 Load Balancer입니다.

특징:

- TCP/UDP/TLS 계층 중심
- 매우 높은 처리량
- 낮은 지연
- 고정 IP 또는 Elastic IP 사용 가능
- 원본 IP 보존 지원
- TLS listener 가능
- Kubernetes Service type LoadBalancer와 자주 연결

사용 예:

- Kubernetes Ingress Controller 앞단
- TCP 서비스 노출
- 고성능 API 진입점
- TLS Passthrough 또는 TCP 처리

### 28-2) AWS ALB

AWS Application Load Balancer는 L7 Load Balancer입니다.

특징:

- HTTP/HTTPS 기반
- Host/Path 기반 라우팅
- Header/Query 기반 조건 라우팅
- TLS Termination
- HTTP/2 지원
- WebSocket 지원
- Target Group 기반 Health Check
- AWS WAF 연동 가능

사용 예:

- 웹 애플리케이션
- REST API
- 마이크로서비스 라우팅
- Kubernetes AWS Load Balancer Controller와 연동

### 28-3) NLB와 ALB 비교

| 구분 | NLB | ALB |
|---|---|---|
| 계층 | L4 | L7 |
| 기준 | IP, Port, TCP/UDP/TLS | HTTP Host, Path, Header |
| HTTP 라우팅 | 제한적 | 강함 |
| 성능 | 매우 높음 | 기능 중심 |
| 원본 IP | 보존 가능 | X-Forwarded-For 사용 |
| 대표 용도 | TCP, Ingress 앞단, 고성능 | Web/API 라우팅 |
| Kubernetes 연결 | Service LoadBalancer | Ingress Controller 또는 AWS LB Controller |

## 29. F5, A10, Enterprise Load Balancer

엔터프라이즈 환경에서는 F5 BIG-IP, A10, Citrix ADC 같은 전용 ADC(Application Delivery Controller)를 사용하기도 합니다.

이들은 단순 Load Balancer보다 더 많은 기능을 제공합니다.

- L4/L7 Load Balancing
- SSL Offloading
- WAF
- Global Traffic Management
- DNS Load Balancing
- DDoS 방어
- TCP 최적화
- HTTP 압축
- Session Persistence
- iRule 또는 정책 기반 라우팅
- 상세 모니터링

금융권, 대기업, 공공기관에서는 이런 장비가 Kubernetes 앞단에 위치하는 경우가 많습니다.

```text
Client
  ↓
F5/A10
  ↓
Kubernetes Ingress
  ↓
Service
  ↓
Pod
```

이때 장애 분석 시 Kubernetes 내부만 보면 안 됩니다. 외부 ADC의 Health Check, Pool Member 상태, SNAT 정책, SSL profile, timeout도 함께 확인해야 합니다.

## 30. Envoy와 Service Mesh

### 30-1) Envoy란

Envoy는 클라우드 네이티브 환경에서 많이 사용되는 L7 Proxy입니다.

대표적으로 Istio, Consul, App Mesh 같은 Service Mesh에서 Data Plane으로 사용됩니다.

기능:

- L4/L7 Proxy
- HTTP/2, gRPC 지원
- Retry
- Circuit Breaker
- Outlier Detection
- mTLS
- Rate Limit
- Observability
- Distributed Tracing
- Dynamic Configuration

### 30-2) Service Mesh에서의 Load Balancing

Service Mesh에서는 각 Pod 옆에 Envoy Sidecar가 붙을 수 있습니다.

```text
App Container
  ↓
Envoy Sidecar
  ↓
Network
  ↓
Envoy Sidecar
  ↓
App Container
```

이 구조에서는 Load Balancing이 중앙 LB 한 곳에서만 일어나는 것이 아닙니다.

각 Envoy가 서비스 디스커버리 정보를 기반으로 요청을 분산합니다.

```text
frontend pod의 Envoy
  ↓
backend pod A/B/C 중 선택
```

### 30-3) Service Mesh와 기존 Load Balancer의 차이

| 구분 | 기존 Load Balancer | Service Mesh |
|---|---|---|
| 위치 | 서비스 앞단 | 서비스 간 통신 경로 |
| 범위 | North-South 중심 | East-West 중심 |
| 기능 | 외부 진입, 라우팅, TLS | mTLS, retry, circuit breaker, tracing |
| 대표 제품 | HAProxy, NGINX, F5 | Istio, Linkerd, Consul |
| Data Plane | LB/Proxy | Envoy 등 Sidecar 또는 Ambient |

North-South는 외부에서 클러스터 내부로 들어오는 트래픽이고, East-West는 클러스터 내부 서비스 간 트래픽입니다.

## 31. Anycast, GSLB, Geo Load Balancing

### 31-1) GSLB

GSLB는 Global Server Load Balancing입니다.

지역 또는 데이터센터 단위로 트래픽을 분산합니다.

```text
사용자 한국 -> 서울 IDC
사용자 미국 -> 버지니아 Region
사용자 유럽 -> 프랑크푸르트 Region
```

일반적인 L4/L7 LB가 하나의 데이터센터 내부 분산을 담당한다면, GSLB는 전 세계 또는 여러 데이터센터 사이의 분산을 담당합니다.

### 31-2) Geo DNS

Geo DNS는 사용자의 위치 또는 DNS resolver 위치를 기준으로 다른 IP를 응답합니다.

```text
app.example.com

한국 사용자 -> 203.0.113.10
미국 사용자 -> 198.51.100.20
```

단점은 DNS Cache 때문에 즉시 전환이 어려울 수 있다는 점입니다.

### 31-3) Anycast

Anycast는 여러 위치에서 같은 IP를 광고하고, 라우팅상 가까운 위치로 사용자를 보내는 방식입니다.

CDN, DNS, 글로벌 서비스에서 많이 사용됩니다.

```text
1.1.1.1
여러 지역에서 동시에 광고
사용자는 가까운 경로로 이동
```

Anycast는 네트워크 라우팅 기반이기 때문에 BGP와 연결됩니다.

## 32. Kubernetes에서 Load Balancer 흐름

### 32-1) Service type LoadBalancer

Kubernetes에서 `type: LoadBalancer` Service를 만들면 외부 Load Balancer를 요청합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
  ports:
    - port: 80
      targetPort: 80
    - port: 443
      targetPort: 443
```

클라우드에서는 Cloud Controller Manager가 이를 보고 LB를 생성합니다.

온프레미스에서는 MetalLB 같은 구현체가 필요합니다.

### 32-2) ExternalTrafficPolicy

`externalTrafficPolicy`는 외부 트래픽이 NodePort/LoadBalancer를 통해 들어올 때 어떻게 처리할지 결정합니다.

```yaml
spec:
  externalTrafficPolicy: Cluster
```

또는

```yaml
spec:
  externalTrafficPolicy: Local
```

#### Cluster

```text
Client
  ↓
Node A
  ↓
Pod가 Node B에 있으면 Node B로 전달
```

장점:

- 어느 노드로 들어와도 전체 Pod로 분산 가능
- 가용성이 좋음

단점:

- SNAT으로 원본 Client IP가 사라질 수 있음
- 노드 간 추가 홉 발생 가능

#### Local

```text
Client
  ↓
Node A
  ↓
Node A에 있는 Pod로만 전달
```

장점:

- 원본 Client IP 보존 가능
- 추가 노드 홉 감소

단점:

- 해당 노드에 Pod가 없으면 트래픽 처리 불가
- LB Health Check가 node-local endpoint를 고려해야 함
- Pod 분산과 DaemonSet/anti-affinity 설계가 중요

### 32-3) Ingress Controller 앞단 Load Balancer

실무 구조:

```text
Client
  ↓
External LB
  ↓
Ingress Controller Service
  ↓
Ingress Controller Pod
  ↓
Backend Service
  ↓
Application Pod
```

이때 장애 분석은 다음 순서로 해야 합니다.

```text
1. DNS가 External LB IP를 가리키는가?
2. External LB가 Ingress Node/Pod를 healthy로 보는가?
3. Ingress Controller가 요청을 받는가?
4. Ingress rule이 Host/Path와 맞는가?
5. Backend Service Endpoint가 있는가?
6. Pod가 Ready인가?
7. Application이 정상 응답하는가?
```

## 33. 실전 장애 사례

### 33-1) 502 Bad Gateway

502는 Proxy 또는 LB가 Backend로부터 정상 응답을 받지 못했을 때 자주 발생합니다.

가능 원인:

- Backend port 미오픈
- targetPort 오류
- Backend connection refused
- Backend가 TLS를 기대하는데 LB가 HTTP로 접속
- Backend가 HTTP를 기대하는데 LB가 HTTPS로 접속
- Keepalive timeout 불일치
- Backend가 응답 전에 연결 종료
- PROXY Protocol 설정 불일치

확인:

```bash
curl -v http://backend-ip:port/health
nc -vz backend-ip port
ss -lntp
kubectl get endpoints
kubectl logs <ingress-controller>
```

### 33-2) 503 Service Unavailable

503은 healthy backend가 없을 때 자주 발생합니다.

가능 원인:

- Health Check 실패
- 모든 Backend down
- Kubernetes Endpoint 없음
- Pod Ready=False
- LB Pool member 전부 unhealthy
- NetworkPolicy로 차단
- 방화벽 차단

확인:

```bash
kubectl get endpoints <svc>
kubectl get endpointslice -l kubernetes.io/service-name=<svc>
kubectl describe pod <pod>
kubectl describe svc <svc>
```

### 33-3) 504 Gateway Timeout

504는 Proxy/LB가 Backend 응답을 기다리다가 timeout이 난 경우입니다.

가능 원인:

- Backend 처리 지연
- DB 쿼리 지연
- upstream read timeout 짧음
- 네트워크 지연
- Backend thread pool 고갈
- connection pool 고갈

확인:

```bash
curl -v --max-time 10 https://app.example.com/api/slow
kubectl logs <pod>
kubectl top pod
kubectl top node
```

### 33-4) Client IP가 모두 LB IP로 보임

가능 원인:

- SNAT 사용
- X-Forwarded-For 미설정
- 애플리케이션이 XFF를 신뢰하지 않음
- PROXY Protocol 미사용
- `externalTrafficPolicy: Cluster`

해결 방향:

- X-Forwarded-For 설정
- trusted proxy 설정
- PROXY Protocol 활성화
- externalTrafficPolicy Local 검토

### 33-5) 로그인 반복 또는 Redirect Loop

가능 원인:

- X-Forwarded-Proto 누락
- X-Forwarded-Host 오류
- Cookie Domain 오류
- SameSite/Secure Cookie 설정 오류
- HTTPS Termination 후 Backend가 HTTP로 인식
- 애플리케이션 base URL 설정 오류

관련 서비스:

- Keycloak
- Harbor
- Grafana
- ArgoCD
- OAuth/OIDC 연동 서비스

### 33-6) 파일 업로드 실패

가능 원인:

- Proxy body size 제한
- Request timeout
- Backend timeout
- Client body buffer 제한
- Ingress annotation 누락
- LB idle timeout 짧음

확인할 설정:

- NGINX `client_max_body_size`
- ingress-nginx `proxy-body-size`
- HAProxy timeout
- ALB idle timeout
- 애플리케이션 업로드 제한

### 33-7) WebSocket 끊김

가능 원인:

- Upgrade Header 미전달
- idle timeout 짧음
- L7 Proxy가 WebSocket을 지원하지 않거나 설정 누락
- Backend keepalive 불일치

필요 Header:

```http
Connection: Upgrade
Upgrade: websocket
```

## 34. Load Balancer 트러블슈팅 절차

### 34-1) 전체 흐름 기준

```text
Client
  ↓
DNS
  ↓
VIP / Load Balancer
  ↓
Frontend Listener
  ↓
Routing Rule
  ↓
Backend Pool
  ↓
Health Check
  ↓
Backend Port
  ↓
Application
```

이 순서로 확인해야 합니다.

### 34-2) 1단계: DNS와 VIP 확인

```bash
dig app.example.com
nslookup app.example.com
```

확인할 것:

- 도메인이 올바른 VIP를 가리키는가?
- 내부/외부 DNS 결과가 다른가?
- TTL 때문에 이전 IP를 보고 있지 않은가?

### 34-3) 2단계: L4 연결 확인

```bash
nc -vz app.example.com 443
telnet app.example.com 443
```

연결이 안 되면 HTTP 문제가 아닙니다.

이 경우 확인 대상은 다음입니다.

- 방화벽
- Security Group
- LB listener
- VIP 라우팅
- Backend Health
- NodePort
- Network ACL

### 34-4) 3단계: TLS 확인

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com
```

확인할 것:

- 인증서 CN/SAN
- 인증서 만료
- Chain 문제
- SNI 문제
- TLS version
- cipher negotiation

### 34-5) 4단계: HTTP 확인

```bash
curl -vk https://app.example.com/
curl -vk https://app.example.com/health
curl -vk -H 'Host: app.example.com' https://<VIP>/
```

확인할 것:

- Status Code
- Redirect Location
- Response Header
- X-Request-ID
- Set-Cookie
- Server Header
- Backend 응답 시간

### 34-6) 5단계: Backend 직접 확인

```bash
curl -v http://10.0.0.11:8080/health
nc -vz 10.0.0.11 8080
```

Kubernetes에서는:

```bash
kubectl get svc -A
kubectl get endpoints -A
kubectl get endpointslice -A
kubectl get pod -A -o wide
kubectl describe svc <svc>
kubectl logs <pod>
```

### 34-7) 6단계: 패킷 확인

```bash
sudo tcpdump -i any host <client-ip>
sudo tcpdump -i any host <backend-ip> and port 8080
sudo tcpdump -i any tcp port 443
```

패킷 캡처로 확인할 수 있는 것:

- SYN이 도착하는가?
- SYN-ACK가 나가는가?
- RST가 발생하는가?
- TLS Handshake가 진행되는가?
- Backend로 실제 요청이 가는가?
- 응답이 돌아오는가?

## 35. 운영 체크리스트

Load Balancer를 운영할 때는 다음을 기본 체크리스트로 삼을 수 있습니다.

```text
1. VIP/DNS 구조가 명확한가?
2. L4/L7 역할이 구분되어 있는가?
3. TLS Termination 위치가 명확한가?
4. Backend Health Check가 적절한가?
5. Health Check interval/rise/fall 값이 적절한가?
6. Connection timeout 값이 서비스 특성과 맞는가?
7. Keepalive timeout이 Backend와 충돌하지 않는가?
8. Connection Draining이 배포 절차와 연결되어 있는가?
9. Client IP 보존 방식이 정해져 있는가?
10. X-Forwarded-* 또는 PROXY Protocol 설정이 일관적인가?
11. Sticky Session이 정말 필요한가?
12. 세션 저장소가 외부화되어 있는가?
13. Retry가 중복 요청을 만들지 않는가?
14. LB 자체 HA 구성이 있는가?
15. 장애 시 Fail Open/Fail Closed 정책이 명확한가?
16. Access Log와 Metrics가 수집되는가?
17. 502/503/504에 대한 원인 분류 기준이 있는가?
18. Kubernetes Endpoint/Readiness와 외부 LB Health Check가 맞물려 있는가?
19. 대용량 업로드, WebSocket, gRPC에 대한 별도 설정이 되어 있는가?
20. 변경 전후 테스트 절차가 있는가?
```

## 36. 요약

Load Balancer는 단순히 요청을 나눠주는 장비가 아닙니다.

실무에서는 다음을 모두 이해해야 제대로 운영할 수 있습니다.

```text
VIP
L4
L7
Reverse Proxy
Forward Proxy
TLS Termination
SSL Bridging
SSL Passthrough
Health Check
Backend Pool
Connection Reuse
Connection Pool
Connection Draining
Sticky Session
Session Persistence
Retry
Circuit Breaker
Failover
X-Forwarded-For
PROXY Protocol
SNAT
DNAT
MetalLB
AWS NLB
AWS ALB
Envoy
Service Mesh
```

Kubernetes 환경에서는 Load Balancer가 Ingress, Service, EndpointSlice, Pod readiness와 직접 연결됩니다.

따라서 장애가 발생했을 때는 단순히 “서비스가 안 된다”가 아니라 다음처럼 계층별로 분리해서 봐야 합니다.

```text
DNS 문제인가?
VIP/LB 문제인가?
L4 TCP 문제인가?
TLS 문제인가?
L7 Routing 문제인가?
Health Check 문제인가?
Backend Pool 문제인가?
Kubernetes Endpoint 문제인가?
Pod/Application 문제인가?
```

이 관점이 잡혀야 HAProxy, NGINX, Ingress, MetalLB, Keycloak, Harbor, Grafana, ArgoCD 장애를 실무적으로 분석할 수 있습니다.
