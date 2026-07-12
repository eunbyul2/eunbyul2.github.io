---
layout: post
title: "네트워크 트러블슈팅 도구: curl, tcpdump, ss, traceroute, dig, nc"
date: 2026-06-25 07:55:00 +0900
categories: [Network, Troubleshooting]
tags: [curl, tcpdump, ss, netstat, traceroute, dig, nslookup, nc, openssl, mtr, Kubernetes, Troubleshooting]
published: true
---

## 1. 이 문서의 목적

네트워크 장애를 분석할 때 중요한 것은 명령어를 많이 아는 것이 아닙니다.

중요한 것은 다음 질문에 순서대로 답하는 것입니다.

```text
1. 이름 해석이 되는가?          DNS
2. 목적지 IP로 갈 경로가 있는가? Routing
3. 중간 네트워크에서 막히는가?   L3 / Hop / MTU
4. 포트가 열려 있는가?          TCP / UDP
5. TLS Handshake가 되는가?      TLS / Certificate
6. HTTP 응답이 정상인가?        HTTP / Application
7. 실제 패킷은 어디까지 가는가? Packet Capture
```

즉, 네트워크 트러블슈팅은 다음 흐름으로 진행해야 합니다.

```text
장애 발생
  ↓
DNS 확인
  ↓
Routing 확인
  ↓
L3 도달성 확인
  ↓
L4 포트 연결 확인
  ↓
TLS Handshake 확인
  ↓
HTTP 요청/응답 확인
  ↓
Packet Capture로 실제 흐름 확인
  ↓
Kubernetes라면 Service / Endpoint / Pod / NetworkPolicy 확인
```

이 문서는 `curl`, `tcpdump`, `ss`, `traceroute`, `dig`, `nc` 같은 도구를 단순히 나열하지 않습니다. 각 도구가 어느 계층을 확인하는지, 어떤 증상에서 먼저 써야 하는지, 출력 결과를 어떻게 해석해야 하는지, Kubernetes 환경에서는 어떻게 적용하는지까지 연결해서 설명합니다.

공식 참고자료:

- curl Documentation: https://curl.se/docs/
- tcpdump man page: https://www.tcpdump.org/manpages/tcpdump.1.html
- ss man page: https://man7.org/linux/man-pages/man8/ss.8.html
- iproute2 ip man page: https://man7.org/linux/man-pages/man8/ip.8.html
- dig man page: https://manpages.debian.org/bind9-dnsutils/dig.1.en.html
- OpenSSL s_client: https://docs.openssl.org/master/man1/openssl-s_client/
- Kubernetes Debug Services: https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/
- Kubernetes DNS for Services and Pods: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/

----

## 2. 트러블슈팅은 계층별로 해야 한다

네트워크 문제를 볼 때는 OSI 계층 또는 TCP/IP 계층을 기준으로 위에서 아래로, 또는 아래에서 위로 확인합니다.

```text
Application Layer  : HTTP, gRPC, DNS Query, Application Response
Transport Layer    : TCP, UDP, Port, Socket State
Network Layer      : IP, Routing, ICMP, TTL
Link Layer         : NIC, MAC, ARP, MTU, Bridge
Physical / Virtual : Cable, VM NIC, veth, CNI, Overlay
```

실무에서는 다음처럼 도구를 배치할 수 있습니다.

| 계층 | 확인 내용 | 대표 도구 |
|---|---|---|
| L7 Application | HTTP 응답, Header, Status Code, Redirect | `curl` |
| TLS | 인증서, SNI, TLS Handshake, ALPN | `openssl s_client`, `curl -v` |
| DNS | 도메인 해석, A/AAAA/CNAME, CoreDNS | `dig`, `nslookup` |
| L4 Transport | TCP/UDP 포트, Socket 상태 | `nc`, `telnet`, `ss` |
| L3 Network | IP 도달성, 라우팅, Hop | `ping`, `traceroute`, `mtr`, `ip route` |
| L2 Link | ARP/Neighbor, NIC, MTU | `ip neigh`, `ethtool`, `tracepath` |
| Packet | 실제 패킷 흐름 | `tcpdump`, Wireshark |
| Kubernetes | Service, Endpoint, Pod, DNS, NetworkPolicy | `kubectl`, `curl`, `dig`, `tcpdump` |

중요한 점은 `curl` 하나만 실패했다고 바로 애플리케이션 문제라고 판단하면 안 된다는 것입니다.

```text
curl 실패
  ├─ DNS 실패일 수 있음
  ├─ Routing 실패일 수 있음
  ├─ TCP 연결 실패일 수 있음
  ├─ TLS 인증서 문제일 수 있음
  ├─ HTTP Redirect 문제일 수 있음
  └─ 실제 애플리케이션 500 에러일 수 있음
```

그래서 장애를 볼 때는 항상 계층을 나누어 봐야 합니다.

----

## 3. 전체 장애 분석 흐름도

예를 들어 사용자가 다음 주소에 접속할 수 없다고 가정합니다.

```text
https://harbor.example.com
```

이때 바로 서버 로그부터 보면 안 됩니다. 먼저 네트워크 흐름을 분해합니다.

```text
Client
  ↓
DNS: harbor.example.com이 IP로 해석되는가?
  ↓
Routing: 그 IP로 나가는 경로가 있는가?
  ↓
L3: 목적지 IP까지 도달 가능한가?
  ↓
L4: 443 포트가 열려 있는가?
  ↓
TLS: 인증서와 TLS Handshake가 정상인가?
  ↓
HTTP: Status Code, Header, Redirect가 정상인가?
  ↓
Application: Harbor 자체가 정상인가?
```

명령어로는 다음 순서가 기본입니다.

```bash
DOMAIN=harbor.example.com
IP=$(dig +short $DOMAIN | tail -n1)
PORT=443

# 1. DNS
dig $DOMAIN

# 2. Routing
ip route get $IP

# 3. L3 도달성
ping -c 3 $IP

# 4. 중간 경로
traceroute $IP
mtr -rw $IP

# 5. TCP 포트 연결
nc -vz $IP $PORT

# 6. TLS Handshake
openssl s_client -connect ${DOMAIN}:${PORT} -servername $DOMAIN </dev/null

# 7. HTTP 응답
curl -vk https://${DOMAIN}/

# 8. 패킷 캡처
sudo tcpdump -i any -nn host $IP and port $PORT
```

이 흐름만 지켜도 대부분의 네트워크 장애는 원인 범위를 크게 좁힐 수 있습니다.

----

## 4. 장애 증상별 먼저 볼 도구

| 증상 | 먼저 확인할 것 | 도구 |
|---|---|---|
| 도메인 접속 불가 | DNS 해석 | `dig`, `nslookup` |
| IP로도 접속 불가 | Routing / L3 | `ip route get`, `ping`, `traceroute` |
| 특정 포트만 안 됨 | TCP/UDP 포트 | `nc`, `telnet`, `ss` |
| HTTPS만 안 됨 | TLS Handshake / 인증서 | `openssl s_client`, `curl -vk` |
| HTTP 404 | Path / Ingress / Backend Routing | `curl -v`, `kubectl get ingress` |
| HTTP 502/503 | LB, Proxy, Backend Down | `curl`, `ss`, `tcpdump`, `kubectl get endpoints` |
| HTTP 301/302 반복 | Redirect / X-Forwarded-Proto | `curl -vL`, Proxy 설정 |
| 간헐적 끊김 | Timeout / Packet Loss | `mtr`, `tcpdump`, `ss` |
| DNS는 되는데 Service 접속 실패 | Endpoint / kube-proxy / CNI | `kubectl get svc,endpoints`, `curl`, `tcpdump` |
| Pod 간 통신 실패 | NetworkPolicy / CNI / Route | `kubectl`, `ip route`, `tcpdump` |
| 외부로 나가지 못함 | NAT / Gateway / DNS | `ip route`, `curl`, `tcpdump`, `iptables` |

----

## 5. curl: HTTP/HTTPS 계층 분석 도구

`curl`은 HTTP/HTTPS 요청을 직접 보내고 응답을 확인하는 도구입니다.

단순히 “웹이 되는지” 보는 도구가 아니라 다음을 확인할 수 있습니다.

- DNS 해석 여부
- TCP 연결 여부
- TLS Handshake 여부
- 인증서 문제
- HTTP Request Header
- HTTP Response Header
- Status Code
- Redirect
- 응답 시간
- HTTP/2 협상 여부
- API 요청/응답

### 5-1) 기본 사용법

```bash
curl https://app.example.com/
curl -v https://app.example.com/
curl -I https://app.example.com/
curl -vk https://app.example.com/
```

| 옵션 | 의미 |
|---|---|
| `-v` | 상세 출력. DNS, TCP, TLS, HTTP Header 확인 |
| `-k` | TLS 인증서 검증 무시 |
| `-I` | `HEAD` 요청. Response Header만 확인 |
| `-L` | Redirect 따라가기 |
| `-H` | Header 추가 |
| `-X` | Method 지정 |
| `-d` | Body 전송 |
| `--resolve` | 특정 도메인을 특정 IP로 강제 매핑 |
| `--connect-timeout` | TCP 연결 제한 시간 |
| `-w` | 응답 시간, Status Code 등 포맷 출력 |

### 5-2) curl -v 출력 흐름 읽기

```bash
curl -v https://app.example.com/
```

`curl -v` 출력은 다음 흐름으로 읽어야 합니다.

```text
DNS Resolve
  ↓
TCP Connect
  ↓
TLS Handshake
  ↓
HTTP Request 전송
  ↓
HTTP Response 수신
```

예시 출력 일부:

```text
* Host app.example.com:443 was resolved.
*   Trying 10.0.0.10:443...
* Connected to app.example.com (10.0.0.10) port 443
* TLSv1.3 (OUT), TLS handshake, Client hello
* TLSv1.3 (IN), TLS handshake, Server hello
> GET / HTTP/1.1
> Host: app.example.com
< HTTP/1.1 200 OK
< content-type: text/html
```

해석:

| 출력 | 의미 |
|---|---|
| `was resolved` | DNS 해석 성공 |
| `Trying IP:PORT` | TCP 연결 시도 |
| `Connected` | TCP 연결 성공 |
| `TLS handshake` | TLS 협상 진행 |
| `> GET` | HTTP 요청 전송 |
| `< HTTP/1.1 200 OK` | HTTP 응답 수신 |

### 5-3) curl 오류 코드 해석

| 오류 | 의미 | 주로 볼 계층 |
|---|---|---|
| `curl: (6) Could not resolve host` | DNS 해석 실패 | DNS |
| `curl: (7) Failed to connect` | TCP 연결 실패 | L4 / Firewall / Service |
| `curl: (28) Operation timed out` | Timeout | L3/L4/L7 전체 가능 |
| `curl: (35) SSL connect error` | TLS Handshake 실패 | TLS |
| `curl: (51) certificate verification failed` | 인증서 검증 실패 | TLS/PKI |
| `curl: (52) Empty reply from server` | TCP 연결 후 응답 없음 | L7/Application/Proxy |
| `curl: (56) Recv failure` | 연결 중 수신 실패 | L4/L7 |
| `HTTP 404` | 요청 경로 없음 | L7 Routing/Application |
| `HTTP 502` | Proxy가 Backend 응답 실패 | Proxy/Backend |
| `HTTP 503` | Backend unavailable | LB/Service/Endpoint |
| `HTTP 504` | Gateway Timeout | Backend timeout/Network |

### 5-4) Host Header 테스트

IP로 직접 접근하면서 Host Header를 지정할 수 있습니다.

```bash
curl -v http://10.0.0.10/ -H 'Host: app.example.com'
```

이 명령은 다음 상황에서 유용합니다.

- DNS를 우회해서 Ingress/HAProxy Host Routing 확인
- 특정 VIP나 Load Balancer IP로 직접 테스트
- Host 기반 라우팅이 정상인지 확인

### 5-5) --resolve로 DNS 우회 테스트

`--resolve`는 특정 도메인을 특정 IP로 강제로 매핑합니다.

```bash
curl -vk --resolve app.example.com:443:10.0.0.10 https://app.example.com/
```

의미:

```text
app.example.com:443 요청을 10.0.0.10으로 보낸다.
하지만 TLS SNI와 HTTP Host는 app.example.com으로 유지한다.
```

Ingress, HAProxy, TLS 인증서 테스트에서 매우 유용합니다.

### 5-6) Redirect 확인

```bash
curl -v http://app.example.com/
curl -vL http://app.example.com/
```

`-L`은 Redirect를 따라갑니다. 하지만 장애 분석에서는 먼저 `-L` 없이 보는 것이 좋습니다.

```text
< HTTP/1.1 301 Moved Permanently
< Location: https://app.example.com/
```

무한 Redirect가 의심되면 다음을 봅니다.

```bash
curl -vkL --max-redirs 5 https://app.example.com/
```

주요 원인:

- `X-Forwarded-Proto` 누락
- Backend가 HTTPS 여부를 잘못 인식
- Ingress/HAProxy Redirect 설정 중복

### 5-7) JSON API 테스트

```bash
curl -v -X POST https://api.example.com/users \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer TOKEN' \
  -d '{"name":"test"}'
```

API 장애에서는 Status Code뿐 아니라 Request Header와 Body도 같이 확인해야 합니다.

### 5-8) 응답 시간 분석

```bash
curl -o /dev/null -s -w \
'DNS:%{time_namelookup}\nConnect:%{time_connect}\nTLS:%{time_appconnect}\nStartTransfer:%{time_starttransfer}\nTotal:%{time_total}\nHTTP:%{http_code}\n' \
https://app.example.com/
```

해석:

| 항목 | 의미 |
|---|---|
| `time_namelookup` | DNS 해석 시간 |
| `time_connect` | TCP 연결 완료 시간 |
| `time_appconnect` | TLS Handshake 완료 시간 |
| `time_starttransfer` | 첫 응답 바이트까지 시간 |
| `time_total` | 전체 요청 시간 |
| `http_code` | HTTP Status Code |

예를 들어 `time_connect`가 길면 네트워크/L4 문제를 의심하고, `time_starttransfer`가 길면 애플리케이션 처리 지연을 의심합니다.

----

## 6. dig: DNS 분석 도구

`dig`는 DNS 조회 결과를 자세히 확인하는 도구입니다.

```bash
dig app.example.com
dig A app.example.com
dig AAAA app.example.com
dig CNAME app.example.com
dig +short app.example.com
dig @8.8.8.8 app.example.com
```

### 6-1) dig 출력 구조

```text
;; QUESTION SECTION:
;app.example.com.       IN A

;; ANSWER SECTION:
app.example.com. 300    IN A 10.0.0.10

;; AUTHORITY SECTION:
...

;; Query time: 20 msec
;; SERVER: 168.126.63.1#53
```

| Section | 의미 |
|---|---|
| QUESTION | 어떤 레코드를 물어봤는지 |
| ANSWER | 실제 응답 |
| AUTHORITY | 권한 있는 DNS 서버 정보 |
| ADDITIONAL | 추가 정보 |
| SERVER | 질의한 DNS 서버 |
| Query time | DNS 응답 시간 |

### 6-2) DNS 장애 판단

| 증상 | 의미 |
|---|---|
| `NXDOMAIN` | 도메인이 존재하지 않음 |
| `SERVFAIL` | DNS 서버가 처리 실패 |
| 응답 없음 | DNS 서버 접근 불가 또는 차단 |
| 내부 DNS와 외부 DNS 결과 다름 | Split DNS 또는 내부 DNS 설정 문제 |
| A 레코드는 있는데 AAAA만 실패 | IPv6 관련 문제 가능 |

### 6-3) Kubernetes 내부 DNS 확인

Kubernetes에서는 Service DNS가 다음 형태로 생성됩니다.

```text
<service>.<namespace>.svc.cluster.local
```

예시:

```bash
dig backend.default.svc.cluster.local
```

Pod 안에서 확인:

```bash
kubectl run -it --rm dns-test --image=busybox:1.36 --restart=Never -- sh
nslookup kubernetes.default.svc.cluster.local
nslookup backend.default.svc.cluster.local
```

CoreDNS 문제를 볼 때는 다음을 확인합니다.

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system deploy/coredns
kubectl get svc -n kube-system kube-dns
kubectl get endpoints -n kube-system kube-dns
```

----

## 7. nslookup: 간단한 DNS 확인 도구

`nslookup`은 간단한 DNS 확인에 유용합니다.

```bash
nslookup app.example.com
nslookup app.example.com 8.8.8.8
nslookup backend.default.svc.cluster.local
```

`dig`와 비교하면 다음과 같습니다.

| 도구 | 특징 |
|---|---|
| `nslookup` | 간단한 조회에 편함 |
| `dig` | 응답 Section, TTL, 서버, 레코드 분석에 좋음 |

실무에서는 빠른 확인은 `nslookup`, 상세 분석은 `dig`를 권장합니다.

----

## 8. ip route: 라우팅 확인

DNS가 정상이라면 다음은 라우팅입니다.

```bash
ip route
ip route get 10.0.0.10
```

`ip route get`은 특정 목적지로 갈 때 실제 선택되는 라우트를 보여줍니다.

예시:

```text
10.0.0.10 via 192.168.10.1 dev ens3 src 192.168.10.50 uid 0
```

해석:

| 항목 | 의미 |
|---|---|
| `10.0.0.10` | 목적지 IP |
| `via 192.168.10.1` | 게이트웨이 |
| `dev ens3` | 나가는 인터페이스 |
| `src 192.168.10.50` | 출발지 IP |

라우팅 문제에서는 다음을 확인합니다.

```bash
ip addr
ip link
ip route
ip rule
ip route get <destination-ip>
```

Kubernetes 노드에서는 Pod CIDR, Service CIDR, CNI route도 함께 확인해야 합니다.

```bash
ip route | grep -E 'cni|tunl|vxlan|cilium|calico|pod'
```

----

## 9. ping: L3 도달성 확인

`ping`은 ICMP Echo Request/Reply로 IP 수준 도달성을 확인합니다.

```bash
ping 8.8.8.8
ping -c 4 10.0.0.10
```

출력 예시:

```text
64 bytes from 10.0.0.10: icmp_seq=1 ttl=63 time=1.23 ms
```

해석:

| 항목 | 의미 |
|---|---|
| `bytes from` | 응답 수신 |
| `icmp_seq` | ICMP 순번 |
| `ttl` | 남은 TTL |
| `time` | RTT 왕복 시간 |

주의할 점:

- ping이 실패해도 HTTP/TCP는 정상일 수 있습니다. ICMP가 방화벽에서 차단될 수 있기 때문입니다.
- ping이 성공해도 HTTP가 정상이라는 뜻은 아닙니다. L3만 확인한 것입니다.
- Kubernetes Service ClusterIP는 환경에 따라 ping 대상이 아닐 수 있습니다.

----

## 10. traceroute: 경로 Hop 확인

`traceroute`는 목적지까지 가는 중간 Hop을 확인합니다.

```bash
traceroute 8.8.8.8
traceroute -n 8.8.8.8
```

원리는 TTL입니다.

```text
TTL=1 → 첫 번째 라우터에서 만료 → ICMP Time Exceeded
TTL=2 → 두 번째 라우터에서 만료 → ICMP Time Exceeded
TTL=3 → 세 번째 라우터에서 만료 → ICMP Time Exceeded
```

`* * *`가 나온다고 반드시 장애는 아닙니다.

가능한 이유:

- 중간 장비가 ICMP 응답을 차단
- Rate Limit
- 방화벽 정책
- 비대칭 라우팅

TCP 기반 traceroute가 필요할 때는 다음처럼 사용할 수 있습니다.

```bash
traceroute -T -p 443 app.example.com
```

----

## 11. mtr: 지속적인 경로 품질 확인

`mtr`은 ping과 traceroute를 합친 도구입니다.

```bash
mtr 8.8.8.8
mtr -rw 8.8.8.8
mtr -rwzc 100 8.8.8.8
```

확인할 항목:

| 항목 | 의미 |
|---|---|
| Loss% | 패킷 손실률 |
| Snt | 보낸 패킷 수 |
| Last | 마지막 RTT |
| Avg | 평균 RTT |
| Best | 최소 RTT |
| Wrst | 최대 RTT |
| StDev | 지연 편차 |

주의:

중간 Hop에서 Loss가 보여도 마지막 목적지 Loss가 0%라면 실제 서비스 장애가 아닐 수 있습니다. 중간 장비가 ICMP 응답만 제한하는 경우가 많습니다.

----

## 12. tracepath: PMTU 확인

`tracepath`는 경로 확인과 함께 PMTU(Path MTU) 정보를 볼 때 유용합니다.

```bash
tracepath 8.8.8.8
tracepath app.example.com
```

MTU 문제는 다음 환경에서 자주 발생합니다.

- VXLAN
- Geneve
- IP-in-IP
- VPN
- Kubernetes Overlay Network
- Cilium/Calico 터널링

증상:

```text
작은 요청은 성공
큰 응답 또는 파일 다운로드는 실패
TLS Handshake 이후 멈춤
간헐적 timeout
```

MTU 의심 시 같이 볼 명령어:

```bash
ip link
ip -d link show
tracepath <destination>
ping -M do -s 1472 <destination>
```

----

## 13. nc(netcat): L4 TCP/UDP 포트 확인

`nc`는 TCP/UDP 연결 확인에 매우 유용합니다.

```bash
nc -vz 10.0.0.10 443
nc -vzu 10.0.0.10 53
```

| 옵션 | 의미 |
|---|---|
| `-v` | 상세 출력 |
| `-z` | 데이터 전송 없이 포트 확인 |
| `-u` | UDP 사용 |
| `-l` | listen 모드 |
| `-p` | 포트 지정 |

### 13-1) TCP 포트 확인

```bash
nc -vz 10.0.0.10 443
```

성공:

```text
Connection to 10.0.0.10 443 port [tcp/https] succeeded!
```

실패:

```text
connect to 10.0.0.10 port 443 failed: Connection refused
```

해석:

| 결과 | 의미 |
|---|---|
| succeeded | TCP 연결 성공. 포트 열림 |
| refused | 목적지에서 RST. 포트 닫힘 또는 서비스 없음 |
| timed out | 방화벽 Drop, 라우팅 문제, 중간 차단 가능 |
| no route to host | 라우팅 또는 네트워크 도달성 문제 |

### 13-2) 간단한 서버/클라이언트 테스트

서버:

```bash
nc -l -p 9999
```

클라이언트:

```bash
nc 10.0.0.10 9999
```

이 테스트는 방화벽, 보안그룹, Kubernetes NetworkPolicy, Node 간 통신 확인에 유용합니다.

----

## 14. telnet: 단순 TCP 연결 확인

`telnet`도 TCP 포트 확인에 사용할 수 있습니다.

```bash
telnet 10.0.0.10 443
```

다만 `telnet`은 보안 접속 도구가 아니며, UDP 확인도 어렵습니다. 가능하면 `nc` 사용을 권장합니다.

----

## 15. ss: Socket 상태 확인

`ss`는 현재 시스템의 Socket 상태를 확인하는 도구입니다. `netstat`보다 현대적이고 빠릅니다.

```bash
ss -lntp
ss -antp
ss -s
ss -tan state established
ss -tan sport = :443
ss -tan dport = :5432
```

| 옵션 | 의미 |
|---|---|
| `-l` | Listening socket |
| `-n` | 숫자로 출력 |
| `-t` | TCP |
| `-u` | UDP |
| `-p` | Process 표시 |
| `-a` | All |
| `-s` | Summary |

### 15-1) LISTEN 포트 확인

```bash
ss -lntp
```

예시:

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      4096   0.0.0.0:443       0.0.0.0:*       users:(("haproxy",pid=1234,fd=7))
```

해석:

- `0.0.0.0:443`: 모든 인터페이스에서 443 대기
- `127.0.0.1:8080`: 로컬에서만 접근 가능
- Process가 없으면 권한 문제 또는 커널 소켓일 수 있음

### 15-2) TCP 상태 해석

| 상태 | 의미 | 장애 관점 |
|---|---|---|
| `LISTEN` | 서버가 연결 대기 | 서비스가 포트를 열고 있음 |
| `SYN-SENT` | 클라이언트가 SYN 보냄 | 응답 없음. 방화벽/라우팅 가능 |
| `SYN-RECV` | 서버가 SYN 받고 SYN/ACK 보냄 | Handshake 미완료, backlog 가능 |
| `ESTAB` | 연결 성립 | 통신 중 |
| `TIME-WAIT` | 정상 종료 후 대기 | 많아도 항상 장애는 아님 |
| `CLOSE-WAIT` | 상대가 닫았지만 로컬 앱이 안 닫음 | 애플리케이션 소켓 누수 의심 |
| `FIN-WAIT-1/2` | 종료 진행 중 | 네트워크/앱 종료 지연 |
| `LAST-ACK` | 마지막 ACK 대기 | 종료 과정 |

`CLOSE-WAIT`가 계속 누적되면 보통 애플리케이션이 소켓을 제대로 닫지 않는 문제를 의심합니다.

----

## 16. netstat: 오래된 도구

`netstat`은 과거에 많이 쓰였지만 현재는 `ss`, `ip route` 사용을 권장합니다.

```bash
netstat -lntp
netstat -antp
netstat -rn
```

대체 명령어:

| netstat | 대체 |
|---|---|
| `netstat -lntp` | `ss -lntp` |
| `netstat -antp` | `ss -antp` |
| `netstat -rn` | `ip route` |
| `netstat -i` | `ip -s link` |

----

## 17. openssl s_client: TLS 분석 필수 도구

HTTPS 장애에서는 `openssl s_client`가 매우 중요합니다.

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null
```

확인할 수 있는 것:

- TLS Handshake 성공 여부
- 서버 인증서 Subject
- SAN
- 인증서 체인
- 만료일
- TLS 버전
- Cipher Suite
- ALPN
- SNI 적용 여부

### 17-1) 인증서 확인

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

출력 예시:

```text
subject=CN = app.example.com
issuer=C = US, O = Let's Encrypt, CN = R3
notBefore=Jun 1 00:00:00 2026 GMT
notAfter=Aug 30 23:59:59 2026 GMT
```

### 17-2) SNI 문제 확인

SNI 없이 접속:

```bash
openssl s_client -connect app.example.com:443 </dev/null
```

SNI 포함:

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null
```

두 결과의 인증서가 다르면 SNI 기반 인증서 선택이 사용되고 있는 것입니다.

### 17-3) ALPN 확인

HTTP/2 또는 gRPC 문제에서는 ALPN을 봐야 합니다.

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com -alpn h2,http/1.1 </dev/null
```

출력에서 다음을 확인합니다.

```text
ALPN protocol: h2
```

`h2`가 선택되지 않으면 HTTP/2/gRPC 관련 문제가 발생할 수 있습니다.

----

## 18. tcpdump: 실제 패킷 확인

`tcpdump`는 네트워크 장애 분석의 최종 확인 도구입니다.

```bash
sudo tcpdump -i any -nn host 10.0.0.10
sudo tcpdump -i any -nn tcp port 443
sudo tcpdump -i ens3 -nn 'host 10.0.0.10 and port 443'
```

### 18-1) 기본 옵션

| 옵션 | 의미 |
|---|---|
| `-i` | 캡처 인터페이스 지정 |
| `any` | 모든 인터페이스 |
| `-n` | DNS reverse lookup 비활성화 |
| `-nn` | 포트 이름 변환도 비활성화 |
| `-s 0` | 패킷 전체 캡처 |
| `-w` | pcap 파일 저장 |
| `-r` | pcap 파일 읽기 |
| `-A` | ASCII 출력 |
| `-X` | Hex + ASCII 출력 |

### 18-2) BPF 필터

`tcpdump` 필터는 BPF 문법을 사용합니다.

```bash
sudo tcpdump -i any -nn host 10.0.0.10
sudo tcpdump -i any -nn src host 10.0.0.10
sudo tcpdump -i any -nn dst host 10.0.0.10
sudo tcpdump -i any -nn net 10.0.0.0/24
sudo tcpdump -i any -nn tcp port 443
sudo tcpdump -i any -nn udp port 53
sudo tcpdump -i any -nn 'host 10.0.0.10 and port 443'
```

| 필터 | 의미 |
|---|---|
| `host` | 출발지 또는 목적지 IP |
| `src host` | 출발지 IP |
| `dst host` | 목적지 IP |
| `net` | 네트워크 대역 |
| `port` | 출발지 또는 목적지 포트 |
| `src port` | 출발지 포트 |
| `dst port` | 목적지 포트 |
| `tcp` | TCP만 |
| `udp` | UDP만 |
| `icmp` | ICMP만 |
| `and` | AND 조건 |
| `or` | OR 조건 |
| `not` | NOT 조건 |

### 18-3) TCP Handshake 읽기

정상 TCP 연결:

```text
Client > Server: Flags [S]
Server > Client: Flags [S.]
Client > Server: Flags [.]
```

의미:

| Flag | 의미 |
|---|---|
| `S` | SYN |
| `S.` | SYN + ACK |
| `.` | ACK |
| `P.` | PSH + ACK, 데이터 포함 가능 |
| `F.` | FIN + ACK |
| `R` | RST |

정상 HTTP 흐름은 대략 다음과 같습니다.

```text
SYN
SYN/ACK
ACK
TLS ClientHello
TLS ServerHello
Encrypted HTTP Request
Encrypted HTTP Response
FIN/ACK
```

HTTP 평문이라면 `-A` 옵션으로 요청을 볼 수 있습니다.

```bash
sudo tcpdump -i any -nn -A 'tcp port 80'
```

HTTPS는 암호화되어 있으므로 HTTP Header/Body가 보이지 않습니다.

### 18-4) pcap 저장 후 Wireshark 분석

```bash
sudo tcpdump -i any -nn -s 0 host 10.0.0.10 -w capture.pcap
```

이후 Wireshark에서 열면 다음을 더 쉽게 볼 수 있습니다.

- TCP Retransmission
- TCP Reset
- TLS Handshake
- DNS Query/Response
- HTTP Request/Response
- Packet Loss 추정
- MTU 문제

----

## 19. ip neigh / ARP 확인

같은 L2 네트워크에서 목적지 MAC을 찾는 과정은 ARP 또는 Neighbor Table로 확인합니다.

```bash
ip neigh
ip neigh show dev ens3
```

상태 예시:

| 상태 | 의미 |
|---|---|
| `REACHABLE` | 최근 통신 가능 확인 |
| `STALE` | 오래됐지만 필요 시 재확인 가능 |
| `DELAY` | 재확인 대기 |
| `PROBE` | ARP/Neighbor Probe 중 |
| `FAILED` | MAC 확인 실패 |

`FAILED`가 보이면 같은 네트워크 대역의 L2/ARP 문제를 의심합니다.

----

## 20. ethtool: NIC 상태 확인

물리 서버 또는 VM NIC 문제는 `ethtool`로 확인합니다.

```bash
ethtool ens3
ethtool -S ens3
```

확인할 항목:

- Link detected
- Speed
- Duplex
- RX errors
- TX errors
- Drops
- Offload 설정

예시:

```bash
ip -s link show ens3
ethtool -S ens3 | grep -Ei 'error|drop|timeout|fail'
```

NIC error/drop이 증가하면 애플리케이션보다 하위 네트워크 문제를 먼저 의심해야 합니다.

----

## 21. lsof: 프로세스와 포트 확인

어떤 프로세스가 포트를 사용 중인지 확인할 수 있습니다.

```bash
sudo lsof -i :443
sudo lsof -iTCP -sTCP:LISTEN -P -n
```

`ss -lntp`와 함께 사용하면 서비스가 실제 포트를 열고 있는지 확인하기 좋습니다.

----

## 22. journalctl: 서비스 로그 확인

네트워크 도구로 포트가 닫힌 것을 확인했다면, 서비스 로그를 봐야 합니다.

```bash
journalctl -u haproxy -f
journalctl -u nginx -f
journalctl -u kubelet -f
journalctl -xe
```

예시:

```bash
systemctl status haproxy
journalctl -u haproxy --since '10 minutes ago'
```

네트워크 장애처럼 보여도 실제로는 서비스가 기동하지 않았거나 설정 오류로 포트를 열지 못한 경우가 많습니다.

----

## 23. Kubernetes 환경에서 도구 사용하기

Kubernetes에서는 “어디에서 테스트하느냐”가 중요합니다.

```text
내 PC에서 테스트
Node에서 테스트
Pod 안에서 테스트
Service 이름으로 테스트
ClusterIP로 테스트
Pod IP로 테스트
외부 LoadBalancer/Ingress로 테스트
```

같은 `curl`이라도 위치에 따라 의미가 다릅니다.

### 23-1) Pod 안에서 테스트용 컨테이너 실행

```bash
kubectl run -it --rm net-debug \
  --image=nicolaka/netshoot \
  --restart=Never -- bash
```

`netshoot` 이미지에는 `curl`, `dig`, `tcpdump`, `ss`, `ip`, `nc` 등 네트워크 도구가 많이 포함되어 있어 디버깅에 유용합니다.

BusyBox로 간단히 확인할 수도 있습니다.

```bash
kubectl run -it --rm busybox \
  --image=busybox:1.36 \
  --restart=Never -- sh
```

### 23-2) Service DNS 확인

```bash
nslookup backend.default.svc.cluster.local
nslookup backend
```

### 23-3) Service / Endpoint 확인

```bash
kubectl get svc -n default
kubectl get endpoints -n default
kubectl get endpointslices -n default
```

중요한 점:

```text
Service는 있는데 Endpoint가 없으면
Service로 들어온 트래픽을 보낼 Pod가 없다는 뜻입니다.
```

### 23-4) Pod 직접 접근

```bash
kubectl get pod -o wide
curl http://<pod-ip>:<port>
```

Service가 안 되는데 Pod IP는 된다면 다음을 의심합니다.

- Service selector 불일치
- Endpoint 생성 실패
- kube-proxy 또는 eBPF Service 처리 문제

### 23-5) Service 접근

```bash
curl http://<service-name>.<namespace>.svc.cluster.local:<port>
curl http://<cluster-ip>:<port>
```

### 23-6) Ingress 접근

```bash
curl -vk https://app.example.com/
curl -vk --resolve app.example.com:443:<ingress-ip> https://app.example.com/
```

Ingress 문제에서는 다음을 함께 봅니다.

```bash
kubectl get ingress -A
kubectl describe ingress -n <namespace> <name>
kubectl get svc -n <ingress-namespace>
kubectl logs -n <ingress-namespace> deploy/<ingress-controller>
```

----

## 24. Kubernetes Service 장애 분석 순서

Service 접속이 안 될 때는 다음 순서로 봅니다.

```text
Client Pod
  ↓
DNS 확인
  ↓
Service 존재 확인
  ↓
Endpoint 존재 확인
  ↓
Pod Ready 상태 확인
  ↓
Pod IP 직접 접근
  ↓
Service ClusterIP 접근
  ↓
NetworkPolicy 확인
  ↓
CNI / kube-proxy / eBPF 확인
```

명령어:

```bash
# DNS
kubectl run -it --rm dns-test --image=busybox:1.36 --restart=Never -- sh
nslookup backend.default.svc.cluster.local

# Service / Endpoint
kubectl get svc backend -n default
kubectl get endpoints backend -n default
kubectl get endpointslices -n default -l kubernetes.io/service-name=backend

# Pod 상태
kubectl get pod -n default -o wide
kubectl describe pod -n default <pod-name>

# Pod 직접 접근
curl http://<pod-ip>:8080/healthz

# Service 접근
curl http://backend.default.svc.cluster.local:8080/healthz

# NetworkPolicy
kubectl get networkpolicy -A
```

----

## 25. Kubernetes DNS 장애 분석 순서

Pod 안에서 Service 이름이 해석되지 않는다면 다음을 봅니다.

```bash
cat /etc/resolv.conf
nslookup kubernetes.default.svc.cluster.local
nslookup backend.default.svc.cluster.local
```

`/etc/resolv.conf` 예시:

```text
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

확인할 것:

| 항목 | 의미 |
|---|---|
| nameserver | CoreDNS Service IP |
| search | Service 짧은 이름 해석에 사용 |
| ndots | DNS 질의 순서에 영향 |

CoreDNS 확인:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc -n kube-system kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

----

## 26. Kubernetes NetworkPolicy 장애 분석

Pod 간 통신이 안 되는데 DNS, Service, Endpoint가 모두 정상이라면 NetworkPolicy를 의심해야 합니다.

```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy -n <namespace> <policy-name>
```

확인 순서:

```text
1. Source Pod label
2. Destination Pod label
3. Namespace label
4. Ingress policy
5. Egress policy
6. Port / Protocol
```

테스트:

```bash
kubectl exec -n <src-ns> <src-pod> -- curl -v http://<dst-service>.<dst-ns>.svc.cluster.local:<port>
```

----

## 27. 실제 장애 사례 1: curl timeout

증상:

```bash
curl -v https://app.example.com/
```

결과:

```text
* Trying 10.0.0.10:443...
* connect to 10.0.0.10 port 443 failed: Operation timed out
```

분석 순서:

```bash
# DNS는 되는지
dig app.example.com

# 라우팅은 있는지
ip route get 10.0.0.10

# ICMP는 되는지
ping -c 3 10.0.0.10

# TCP 포트는 열려 있는지
nc -vz 10.0.0.10 443

# 패킷은 나가는지
sudo tcpdump -i any -nn host 10.0.0.10 and port 443
```

해석:

| tcpdump 결과 | 의미 |
|---|---|
| SYN만 반복 | 응답 없음. 방화벽 Drop, 라우팅, 서버 미도달 가능 |
| SYN → RST | 서버가 포트 거부. 서비스 미기동 가능 |
| SYN → SYN/ACK → ACK 이후 멈춤 | TLS/Application 단계 문제 가능 |

----

## 28. 실제 장애 사례 2: DNS 실패

증상:

```text
curl: (6) Could not resolve host: app.example.com
```

분석:

```bash
dig app.example.com
cat /etc/resolv.conf
dig @8.8.8.8 app.example.com
```

Kubernetes 내부라면:

```bash
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local
kubectl logs -n kube-system -l k8s-app=kube-dns
```

판단:

| 결과 | 원인 후보 |
|---|---|
| 외부 DNS는 되는데 내부 DNS만 실패 | CoreDNS 문제 |
| 특정 도메인만 NXDOMAIN | DNS 레코드 없음 |
| 모든 DNS timeout | DNS 서버 접근 불가 |
| Pod에서만 실패 | Pod DNS 설정, NetworkPolicy, CoreDNS 접근 문제 |

----

## 29. 실제 장애 사례 3: TLS 인증서 오류

증상:

```text
curl: (60) SSL certificate problem
```

분석:

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

확인:

- 인증서 만료 여부
- CN/SAN에 도메인이 포함되어 있는지
- 중간 인증서 체인이 정상인지
- SNI를 넣었을 때와 안 넣었을 때 인증서가 다른지

임시 우회:

```bash
curl -vk https://app.example.com/
```

`-k`는 검증을 무시하는 테스트용 옵션입니다. 운영 해결책이 아닙니다.

----

## 30. 실제 장애 사례 4: 무한 Redirect

증상:

```bash
curl -vkL https://app.example.com/
```

결과:

```text
Maximum redirects followed
```

분석:

```bash
curl -vk https://app.example.com/
```

`Location` Header를 확인합니다.

```text
< HTTP/1.1 301 Moved Permanently
< Location: https://app.example.com/
```

원인 후보:

- HAProxy/Ingress에서 TLS Termination 후 백엔드로 HTTP 전달
- 백엔드가 원래 요청이 HTTPS였다는 것을 모름
- `X-Forwarded-Proto` 누락
- 애플리케이션 Redirect 설정 중복

해결 예시:

```haproxy
http-request set-header X-Forwarded-Proto https if { ssl_fc }
```

Nginx Ingress라면 관련 annotation과 애플리케이션 trusted proxy 설정도 확인해야 합니다.

----

## 31. 실제 장애 사례 5: Kubernetes Service는 있는데 접속 실패

증상:

```bash
curl http://backend.default.svc.cluster.local:8080
```

실패.

분석:

```bash
kubectl get svc backend -n default
kubectl get endpoints backend -n default
kubectl get pod -n default -o wide
kubectl describe svc backend -n default
```

자주 보는 원인:

| 상태 | 의미 |
|---|---|
| Service 있음, Endpoint 없음 | selector와 Pod label 불일치 또는 Pod NotReady |
| Endpoint 있음, Pod IP 직접 접근 실패 | Pod 앱 미기동, 컨테이너 포트 문제 |
| Pod IP는 되는데 Service 실패 | kube-proxy/eBPF Service 처리 문제 가능 |
| 특정 Pod에서만 실패 | NetworkPolicy 또는 Node/CNI 문제 |

----

## 32. 실제 장애 사례 6: Harbor 접속 장애

Harbor는 일반적으로 HTTPS, Host Routing, X-Forwarded-Proto, API Path가 중요합니다.

분석 순서:

```bash
DOMAIN=harbor.example.com
IP=<haproxy-or-ingress-ip>

# DNS
dig $DOMAIN

# 특정 IP로 강제 테스트
curl -vk --resolve ${DOMAIN}:443:${IP} https://${DOMAIN}/

# 인증서
openssl s_client -connect ${DOMAIN}:443 -servername ${DOMAIN} </dev/null

# Health/API
curl -vk --resolve ${DOMAIN}:443:${IP} https://${DOMAIN}/api/v2.0/health
```

확인할 것:

- 인증서가 Harbor 도메인과 맞는가?
- 301/302 Redirect가 반복되는가?
- `X-Forwarded-Proto`가 필요한 구조인가?
- HAProxy/Ingress가 올바른 backend로 보내는가?
- Harbor core/nginx 로그에 요청이 도달하는가?

----

## 33. 실제 장애 사례 7: ArgoCD 접속 장애

ArgoCD는 Web UI와 gRPC 성격의 통신을 함께 고려해야 합니다.

확인:

```bash
curl -vk https://argocd.example.com/
openssl s_client -connect argocd.example.com:443 -servername argocd.example.com -alpn h2,http/1.1 </dev/null
```

확인할 것:

- TLS Termination 위치
- HTTP/2 또는 gRPC 관련 처리
- Redirect URL
- `X-Forwarded-Proto`
- Ingress/HAProxy timeout

ArgoCD CLI 접속 문제가 UI 접속 문제와 다르게 나타날 수 있으므로, CLI와 UI 요청을 분리해서 분석해야 합니다.

----

## 34. 실제 장애 사례 8: Kubernetes API Server 6443 접속 장애

Kubernetes API Server는 일반적으로 6443/TCP에서 TLS로 동작합니다.

확인:

```bash
nc -vz <api-server-ip> 6443
openssl s_client -connect <api-server-ip>:6443 -servername kubernetes </dev/null
curl -k https://<api-server-ip>:6443/version
```

HAProxy 앞단이 있다면:

```bash
sudo tcpdump -i any -nn port 6443
ss -lntp | grep 6443
```

확인할 것:

- HAProxy가 6443을 LISTEN 중인가?
- Backend control-plane 노드의 6443 포트가 열려 있는가?
- TCP Mode로 전달하고 있는가?
- 인증서 SAN에 접근 주소가 포함되어 있는가?
- kube-apiserver 프로세스가 정상인가?

----

## 35. Packet Flow 기준으로 다시 보기

`curl` 한 번은 내부적으로 많은 단계를 거칩니다.

```text
curl
  ↓
DNS Resolver
  ↓
Socket 생성
  ↓
TCP SYN
  ↓
TCP SYN/ACK
  ↓
TCP ACK
  ↓
TLS ClientHello
  ↓
TLS ServerHello / Certificate
  ↓
HTTP Request
  ↓
HTTP Response
  ↓
Connection Close or Keep-Alive
```

각 단계에서 사용할 도구는 다음과 같습니다.

| 단계 | 확인 도구 |
|---|---|
| DNS Resolver | `dig`, `nslookup`, `/etc/resolv.conf` |
| Socket 생성 | `ss`, `lsof` |
| TCP SYN/SYNACK | `tcpdump`, `nc` |
| TLS ClientHello | `openssl s_client`, `curl -v`, `tcpdump` |
| HTTP Request/Response | `curl -v`, Application log |
| Connection Close | `ss`, `tcpdump` |

----

## 36. 최종 Troubleshooting Tree

```text
접속 안 됨
  ↓
[1] DNS 확인
  ├─ 실패: dig, nslookup, resolv.conf, CoreDNS 확인
  └─ 성공
       ↓
[2] Routing 확인
  ├─ 실패: ip route, gateway, route table 확인
  └─ 성공
       ↓
[3] L3 도달성 확인
  ├─ ping 실패: ICMP 차단인지 실제 미도달인지 구분
  └─ 다음 단계
       ↓
[4] L4 포트 확인
  ├─ nc refused: 서비스 미기동/포트 닫힘
  ├─ nc timeout: 방화벽/라우팅/Drop 가능
  └─ 성공
       ↓
[5] TLS 확인
  ├─ 인증서 오류: openssl, SAN, 만료일, Chain 확인
  ├─ Handshake 실패: TLS 버전, SNI, ALPN 확인
  └─ 성공
       ↓
[6] HTTP 확인
  ├─ 301/302 반복: Redirect, X-Forwarded-Proto 확인
  ├─ 404: Path/Ingress/Route 확인
  ├─ 502/503: Backend/Endpoint 확인
  ├─ 504: Timeout/Backend 지연 확인
  └─ 200/정상
       ↓
[7] Kubernetes 확인
  ├─ Service/Endpoint
  ├─ Pod Ready
  ├─ NetworkPolicy
  ├─ CNI/kube-proxy/eBPF
  └─ CoreDNS
       ↓
[8] tcpdump로 실제 패킷 확인
```

----

## 37. 명령어 Cheat Sheet

### DNS

```bash
dig app.example.com
dig +short app.example.com
dig @8.8.8.8 app.example.com
nslookup app.example.com
cat /etc/resolv.conf
```

### Routing / L3

```bash
ip addr
ip link
ip route
ip route get <ip>
ping -c 3 <ip>
traceroute -n <ip>
mtr -rw <ip>
tracepath <ip>
```

### L4 TCP/UDP

```bash
nc -vz <ip> <port>
nc -vzu <ip> <port>
telnet <ip> <port>
ss -lntp
ss -antp
ss -s
```

### TLS

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null
openssl s_client -connect app.example.com:443 -servername app.example.com -alpn h2,http/1.1 </dev/null
curl -vk https://app.example.com/
```

### HTTP

```bash
curl -v http://app.example.com/
curl -vk https://app.example.com/
curl -I https://app.example.com/
curl -vL https://app.example.com/
curl -vk --resolve app.example.com:443:10.0.0.10 https://app.example.com/
```

### Packet Capture

```bash
sudo tcpdump -i any -nn host <ip>
sudo tcpdump -i any -nn tcp port 443
sudo tcpdump -i any -nn 'host <ip> and port 443'
sudo tcpdump -i any -nn -s 0 host <ip> -w capture.pcap
```

### Kubernetes

```bash
kubectl get pod -A -o wide
kubectl get svc -A
kubectl get endpoints -A
kubectl get endpointslices -A
kubectl describe svc -n <ns> <svc>
kubectl describe pod -n <ns> <pod>
kubectl logs -n <ns> <pod>
kubectl get networkpolicy -A
kubectl run -it --rm net-debug --image=nicolaka/netshoot --restart=Never -- bash
```

----

## 38. 결론

네트워크 트러블슈팅에서 가장 위험한 방식은 명령어를 무작위로 실행하는 것입니다.

도구는 계층별로 목적이 다릅니다.

```text
DNS 문제는 dig/nslookup
Routing 문제는 ip route
L3 도달성은 ping/traceroute/mtr
L4 포트는 nc/ss
TLS 문제는 openssl s_client/curl -v
HTTP 문제는 curl
실제 패킷은 tcpdump
Kubernetes 문제는 kubectl + Pod 내부 테스트
```

장애를 분석할 때는 다음 기준으로 판단하면 됩니다.

```text
이 도구가 지금 어느 계층을 확인하고 있는가?
이 결과가 성공이면 다음 계층은 무엇인가?
이 결과가 실패이면 가능한 원인 범위는 어디까지인가?
```

결국 실무 네트워크 장애 분석의 핵심은 다음입니다.

```text
증상 → 계층 분리 → 도구 선택 → 출력 해석 → 원인 범위 축소 → 실제 패킷 확인
```

이 흐름을 지키면 Kubernetes, HAProxy, Harbor, ArgoCD, Ceph Dashboard, 일반 웹 서비스, DB 서비스 장애를 훨씬 안정적으로 분석할 수 있습니다.