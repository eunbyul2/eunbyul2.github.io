---
layout: post
title: "Kubernetes Ingress와 Ingress Controller 완전 정리: Host/Path Routing, TLS, NGINX, HAProxy, Traefik, Gateway API"
date: 2026-06-24 20:55:00 +0900
categories: [Kubernetes, Network]
tags: [Ingress, IngressController, NGINX, HAProxy, Traefik, TLS, GatewayAPI, OAuth, Keycloak]
published: true
---

## 1. 이 글에서 다루는 범위

이 글은 Kubernetes에서 외부 HTTP/HTTPS 트래픽을 애플리케이션으로 전달할 때 가장 많이 사용하는 **Ingress**와 **Ingress Controller**를 상세히 정리한다.

앞선 글에서 다음 내용을 다뤘다.

```text
01. 네트워크 기초
    IP, CIDR, Subnet, Gateway, ARP, Routing, NAT, conntrack

02. HTTP/HTTPS
    HTTP Request/Response, Header, Cookie, TLS, SNI, OAuth/OIDC

03. Kubernetes 네트워크
    Pod Network, CNI, veth, bridge, overlay, native routing, kube-proxy/eBPF

04. Kubernetes Service / Endpoint
    ClusterIP, NodePort, LoadBalancer, EndpointSlice, Headless Service
```

이번 글은 그 다음 단계다.

즉, 사용자가 브라우저에서 다음 주소를 입력했을 때,

```text
https://harbor.example.com
https://grafana.example.com
https://argocd.example.com
https://keycloak.example.com
```

요청이 어떻게 Kubernetes 내부의 Service와 Pod까지 전달되는지 설명한다.

Ingress는 단순히 “도메인으로 Service를 연결하는 리소스”가 아니다. 실무에서는 다음 문제들과 직접 연결된다.

```text
404 Not Found
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
400 Bad Request
TLS 인증서 오류
무한 Redirect
OAuth Callback 오류
Keycloak 로그인 실패
Harbor 로그인 실패
Grafana root_url 오류
ArgoCD HTTPS Redirect 오류
X-Forwarded-Proto 누락
Host Header 불일치
IngressClass 불일치
Path rewrite 오류
Backend protocol 불일치
```

따라서 Ingress를 이해하려면 단순 YAML 문법보다 다음 흐름을 먼저 이해해야 한다.

```text
Client
  ↓
DNS
  ↓
External Load Balancer
  ↓
Ingress Controller
  ↓
Ingress Rule
  ↓
Service
  ↓
EndpointSlice
  ↓
Pod
  ↓
Application
```

공식 참고자료:

- Kubernetes Ingress: https://kubernetes.io/docs/concepts/services-networking/ingress/
- Kubernetes Ingress Controllers: https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/
- Kubernetes Gateway API: https://kubernetes.io/docs/concepts/services-networking/gateway/
- Gateway API 공식 문서: https://gateway-api.sigs.k8s.io/
- Gateway API TLS Configuration: https://gateway-api.sigs.k8s.io/guides/user-guides/tls/
- ingress-nginx Documentation: https://kubernetes.github.io/ingress-nginx/
- ingress-nginx Annotations: https://github.com/kubernetes/ingress-nginx/blob/main/docs/user-guide/nginx-configuration/annotations.md
- ingress-nginx ConfigMap: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/
- NGINX Ingress Controller: https://docs.nginx.com/nginx-ingress-controller/
- HAProxy Kubernetes Ingress Controller: https://www.haproxy.com/documentation/kubernetes-ingress/
- Traefik Kubernetes Ingress: https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/
- RFC 9110 HTTP Semantics: https://datatracker.ietf.org/doc/html/rfc9110
- RFC 8446 TLS 1.3: https://datatracker.ietf.org/doc/html/rfc8446

## 2. Ingress가 필요한 이유

### 2-1) Service만으로 외부 노출하면 어떤 문제가 생기는가

Kubernetes에서 애플리케이션을 외부에 노출하는 가장 기본적인 방법은 Service다.

예를 들어 다음 서비스들이 있다고 가정한다.

```text
frontend Service
backend Service
grafana Service
harbor Service
argocd Service
keycloak Service
```

각 Service를 외부에 노출하려면 `type: LoadBalancer`를 사용할 수 있다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
spec:
  type: LoadBalancer
  selector:
    app: grafana
  ports:
    - port: 80
      targetPort: 3000
```

이 방식은 단순하다.

```text
Client
  ↓
LoadBalancer
  ↓
Service
  ↓
Pod
```

하지만 서비스가 늘어나면 문제가 생긴다.

```text
frontend.example.com  -> LoadBalancer 1개
api.example.com       -> LoadBalancer 1개
grafana.example.com   -> LoadBalancer 1개
harbor.example.com    -> LoadBalancer 1개
argocd.example.com    -> LoadBalancer 1개
keycloak.example.com  -> LoadBalancer 1개
```

이렇게 되면 서비스마다 외부 IP 또는 로드밸런서가 필요하다. 클라우드 환경에서는 비용이 증가하고, 온프레미스 환경에서는 외부 IP 할당과 L4 장비 설정이 복잡해진다.

또한 HTTP/HTTPS 서비스는 단순히 IP와 포트만으로 나누기보다 다음 기준으로 나누는 경우가 많다.

```text
Host 기준
  harbor.example.com
  grafana.example.com
  argocd.example.com

Path 기준
  example.com/api
  example.com/admin
  example.com/web

Header 기준
  X-Canary: true

Cookie 기준
  user-group=beta
```

Service는 기본적으로 L4 수준의 추상화다. 즉, IP와 Port 중심이다. 반면 웹 서비스 운영에서는 L7 정보인 Host, Path, Header, Cookie를 기준으로 라우팅해야 하는 경우가 많다.

이 문제를 해결하기 위해 Ingress가 사용된다.

### 2-2) Ingress를 사용하면 구조가 어떻게 달라지는가

Ingress를 사용하면 외부 진입점은 하나 또는 소수의 LoadBalancer로 줄이고, 그 아래에서 HTTP/HTTPS 라우팅을 수행할 수 있다.

```text
                         ┌─────────────── frontend Service
                         │
Client -> LoadBalancer -> Ingress Controller -> backend Service
                         │
                         ├─────────────── grafana Service
                         │
                         ├─────────────── harbor Service
                         │
                         └─────────────── keycloak Service
```

도메인 기준 라우팅 예시:

```text
harbor.example.com   -> harbor Service
grafana.example.com  -> grafana Service
argocd.example.com   -> argocd Service
keycloak.example.com -> keycloak Service
```

Path 기준 라우팅 예시:

```text
example.com/        -> frontend Service
example.com/api     -> backend Service
example.com/admin   -> admin Service
```

즉, Ingress는 Kubernetes 내부에서 HTTP/HTTPS 트래픽을 여러 Service로 나누어 전달하기 위한 표준 리소스다.

### 2-3) Ingress가 해결하는 문제

Ingress는 다음 문제를 해결한다.

| 문제 | Service만 사용할 때 | Ingress 사용 시 |
|---|---|---|
| 외부 IP 관리 | 서비스마다 IP 필요 가능 | 하나의 진입점에서 여러 서비스 라우팅 |
| HTTP 라우팅 | IP/Port 중심 | Host/Path 중심 |
| TLS 인증서 | 서비스별 개별 처리 필요 | Ingress Controller에서 중앙 처리 가능 |
| Rewrite | 애플리케이션별 처리 | 프록시 계층에서 처리 가능 |
| 인증/인가 연동 | 앱마다 구현 | 일부 Controller/Proxy에서 공통 처리 가능 |
| Rate Limit | 앱마다 구현 | Ingress에서 공통 적용 가능 |
| Canary | 별도 배포 설계 필요 | Controller 기능으로 일부 구현 가능 |

## 3. L4 Load Balancer와 L7 Ingress의 차이

### 3-1) L4는 IP와 Port를 본다

L4 Load Balancer는 Transport Layer 정보를 기준으로 트래픽을 전달한다.

주로 보는 정보는 다음이다.

```text
Source IP
Source Port
Destination IP
Destination Port
Protocol TCP/UDP
```

예시:

```text
Client -> 203.0.113.10:443
```

L4 로드밸런서는 이 요청이 `harbor.example.com`인지, `grafana.example.com`인지, `/api` 요청인지 기본적으로 모른다. 단지 `203.0.113.10:443`으로 들어온 TCP 연결을 어느 backend로 보낼지 결정한다.

### 3-2) L7은 HTTP 내용을 본다

L7 Proxy 또는 L7 Load Balancer는 HTTP Request를 해석한다.

다음 정보를 기준으로 라우팅할 수 있다.

```text
Host Header
URL Path
HTTP Method
HTTP Header
Cookie
Query Parameter
TLS SNI
```

예시:

```http
GET /api/users HTTP/1.1
Host: app.example.com
User-Agent: curl/8.0
```

L7 프록시는 이 요청을 보고 다음처럼 판단할 수 있다.

```text
Host가 app.example.com이고 Path가 /api로 시작한다.
따라서 backend-api Service로 보낸다.
```

Ingress는 Kubernetes에서 이 L7 라우팅 규칙을 표현하는 리소스다.

### 3-3) Ingress는 L7 라우팅이고, Service는 주로 L4 추상화다

Service는 다음처럼 IP와 Port 중심으로 트래픽을 전달한다.

```text
ClusterIP:80 -> PodIP:8080
```

Ingress는 다음처럼 HTTP 의미를 해석한다.

```text
Host: harbor.example.com
Path: /
  -> harbor Service

Host: grafana.example.com
Path: /
  -> grafana Service

Host: app.example.com
Path: /api
  -> backend Service
```

따라서 Ingress를 이해하려면 HTTP Header, 특히 `Host`, `X-Forwarded-For`, `X-Forwarded-Proto`, `X-Forwarded-Host`를 반드시 알아야 한다.

## 4. Ingress와 Ingress Controller의 차이

### 4-1) Ingress는 규칙이다

Ingress 리소스는 “어떤 요청을 어떤 Service로 보낼지”를 선언하는 Kubernetes API 객체다.

예시:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
```

이 YAML은 다음 의미다.

```text
Host가 app.example.com이고
Path가 /api로 시작하는 요청은
backend Service의 80번 포트로 전달한다.
```

하지만 이 YAML만 만든다고 실제로 트래픽이 처리되지는 않는다.

### 4-2) Ingress Controller가 실제 트래픽을 처리한다

Ingress Controller는 Kubernetes API Server를 감시하면서 Ingress 리소스를 읽고, 실제 프록시 설정으로 변환하는 컨트롤러다.

```text
Kubernetes API Server
  ↓ watch
Ingress Controller
  ↓ config generation
NGINX / HAProxy / Traefik / Envoy
  ↓
Client Request 처리
```

즉, Ingress는 설정이고 Ingress Controller는 실행체다.

비유하면 다음과 같다.

| 구성요소 | 역할 |
|---|---|
| Ingress YAML | 라우팅 규칙 문서 |
| Ingress Controller | 규칙을 읽고 실제 프록시를 설정하는 관리자 |
| NGINX/HAProxy/Traefik | 실제 HTTP 요청을 받아 처리하는 프록시 엔진 |

Ingress Controller가 없으면 Ingress 리소스를 만들어도 외부 요청은 처리되지 않는다.

### 4-3) Ingress Controller가 감시하는 리소스

Ingress Controller는 보통 다음 리소스를 감시한다.

```text
Ingress
IngressClass
Service
EndpointSlice
Secret
ConfigMap
Pod
Namespace
```

왜 Service와 EndpointSlice까지 보는가?

Ingress는 backend로 Service 이름만 지정한다.

```yaml
backend:
  service:
    name: backend
    port:
      number: 80
```

하지만 실제 요청을 처리하려면 최종 Pod IP가 필요하다.

```text
backend Service
  ↓
EndpointSlice
  ↓
10.244.1.10:8080
10.244.2.15:8080
```

따라서 Ingress Controller는 Ingress만 보는 것이 아니라 Service와 EndpointSlice까지 함께 본다.

## 5. Ingress Controller 내부 동작 구조

### 5-1) Controller Watch 구조

Ingress Controller는 Kubernetes API Server에 watch 요청을 걸어 리소스 변화를 감시한다.

```text
사용자가 Ingress 생성
  ↓
API Server에 저장
  ↓
Ingress Controller가 이벤트 감지
  ↓
Service/EndpointSlice/Secret 조회
  ↓
프록시 설정 생성
  ↓
NGINX Reload 또는 동적 설정 반영
```

NGINX Ingress Controller 기준으로 보면 다음 흐름이다.

```text
Ingress YAML
  ↓
Controller
  ↓
nginx.conf 생성
  ↓
NGINX reload
  ↓
요청 라우팅
```

Traefik이나 일부 Envoy 기반 컨트롤러는 설정을 더 동적으로 반영할 수 있다. 구현체마다 내부 반영 방식은 다르다.

### 5-2) 설정 반영 지연

Ingress를 수정하자마자 바로 트래픽이 바뀌지 않는 경우가 있다.

가능한 이유는 다음이다.

```text
Controller가 API 이벤트를 아직 처리하지 않음
프록시 설정 생성 중
NGINX reload 중
Secret 또는 Service 참조 오류
IngressClass 불일치
Controller 로그상 rejected 상태
```

따라서 Ingress 수정 후에는 다음을 확인해야 한다.

```bash
kubectl describe ingress <ingress-name>
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
kubectl get endpointslice -l kubernetes.io/service-name=<service-name>
```

## 6. IngressClass

### 6-1) IngressClass가 필요한 이유

하나의 Kubernetes 클러스터에는 여러 Ingress Controller가 동시에 존재할 수 있다.

예시:

```text
nginx ingress controller
haproxy ingress controller
traefik ingress controller
internal ingress controller
external ingress controller
```

이때 어떤 Ingress를 어떤 Controller가 처리할지 지정해야 한다. 이 역할을 하는 것이 `IngressClass`다.

Ingress 예시:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 80
```

IngressClass 확인:

```bash
kubectl get ingressclass
```

예시 출력:

```text
NAME     CONTROLLER                     PARAMETERS
nginx    k8s.io/ingress-nginx           <none>
haproxy  haproxy.org/ingress-controller <none>
traefik  traefik.io/ingress-controller  <none>
```

### 6-2) IngressClass가 틀리면 어떤 일이 생기는가

IngressClass가 맞지 않으면 Controller가 해당 Ingress를 무시할 수 있다.

증상:

```text
Ingress 리소스는 존재함
kubectl get ingress에도 보임
하지만 실제 라우팅 안 됨
외부 접속 시 404 또는 default backend 응답
Controller 로그에 해당 Ingress 반영 흔적 없음
```

확인:

```bash
kubectl get ingress <name> -o yaml
kubectl get ingressclass
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep <ingress-name>
```

### 6-3) 기본 IngressClass

클러스터에는 기본 IngressClass가 지정될 수 있다.

```yaml
metadata:
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
```

기본 IngressClass가 있으면 `spec.ingressClassName`을 생략한 Ingress가 해당 Controller에 의해 처리될 수 있다. 하지만 운영 환경에서는 명시적으로 지정하는 편이 안전하다.

## 7. Ingress Resource 기본 구조

### 7-1) 최소 Ingress 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
```

이 YAML은 다음 구조다.

| 필드 | 의미 |
|---|---|
| `metadata.name` | Ingress 리소스 이름 |
| `metadata.namespace` | Ingress가 존재하는 namespace |
| `spec.ingressClassName` | 처리할 Ingress Controller |
| `spec.rules[].host` | 매칭할 HTTP Host |
| `spec.rules[].http.paths[].path` | 매칭할 URL Path |
| `pathType` | Path 매칭 방식 |
| `backend.service.name` | 전달할 Service 이름 |
| `backend.service.port.number` | 전달할 Service Port |

중요한 점은 backend Service는 같은 namespace에 있어야 한다는 것이다. 기본 Ingress 리소스는 다른 namespace의 Service를 직접 backend로 지정할 수 없다. 이런 요구사항이 있으면 Gateway API, 별도 Controller 기능, ExternalName, 또는 구조 재설계를 검토해야 한다.

### 7-2) Ingress의 namespace 범위

Ingress는 namespace 리소스다.

```text
default namespace의 Ingress
  -> default namespace의 Service를 backend로 참조

monitoring namespace의 Ingress
  -> monitoring namespace의 Service를 backend로 참조
```

따라서 다음과 같은 실수가 자주 발생한다.

```text
Ingress는 default namespace에 있음
Service는 qks-harbor namespace에 있음
```

이 경우 Ingress가 Service를 찾지 못하거나, 의도한 backend로 연결되지 않는다.

확인:

```bash
kubectl get ingress -A
kubectl get svc -A
kubectl describe ingress -n <namespace> <name>
```

## 8. 실제 Request Flow

### 8-1) 사용자가 URL을 입력했을 때

사용자가 브라우저에 다음 주소를 입력했다고 가정한다.

```text
https://harbor.example.com/
```

실제 흐름은 다음과 같다.

```text
1. DNS 조회
   harbor.example.com -> 203.0.113.10

2. TCP 연결
   Client -> 203.0.113.10:443

3. TLS Handshake
   SNI: harbor.example.com
   인증서 검증

4. HTTP Request 생성
   GET / HTTP/1.1
   Host: harbor.example.com

5. External Load Balancer 수신
   203.0.113.10:443

6. Ingress Controller로 전달
   ingress-nginx-controller Pod 또는 NodePort/LoadBalancer Service

7. Ingress Controller가 Host/Path 매칭
   Host: harbor.example.com
   Path: /

8. Backend Service 선택
   harbor Service

9. Service -> EndpointSlice 확인
   10.244.1.20:8080
   10.244.2.30:8080

10. Pod로 전달
    Harbor Pod

11. Application Response
    HTTP/1.1 200 OK
```

요청 흐름을 그림으로 보면 다음과 같다.

```text
Browser
  |
  |  https://harbor.example.com/
  v
DNS
  |
  |  harbor.example.com = 203.0.113.10
  v
External Load Balancer
  |
  v
Ingress Controller Service
  |
  v
Ingress Controller Pod
  |
  |  Host: harbor.example.com
  |  Path: /
  v
harbor Service
  |
  v
EndpointSlice
  |
  v
Harbor Pod
```

### 8-2) curl로 같은 흐름 테스트하기

DNS를 우회하고 LB IP에 직접 Host Header를 넣어 테스트할 수 있다.

```bash
curl -v http://203.0.113.10/ -H 'Host: harbor.example.com'
```

HTTPS라면 SNI까지 맞춰야 한다. `--resolve`를 사용하면 DNS 결과를 임시로 지정하면서 SNI와 Host Header를 함께 맞출 수 있다.

```bash
curl -vk --resolve harbor.example.com:443:203.0.113.10 https://harbor.example.com/
```

이 명령은 다음 의미다.

```text
harbor.example.com:443 접속을 203.0.113.10으로 보낸다.
URL은 https://harbor.example.com/ 이므로 Host와 SNI는 harbor.example.com으로 유지된다.
```

Ingress TLS 문제를 확인할 때 매우 유용하다.

## 9. Host-based Routing

### 9-1) Host Header가 중요한 이유

HTTP/1.1 요청에는 `Host` Header가 포함된다.

```http
GET / HTTP/1.1
Host: grafana.example.com
```

Ingress Controller는 이 Host Header를 보고 어느 Service로 보낼지 결정한다.

```text
Host: grafana.example.com -> grafana Service
Host: harbor.example.com  -> harbor Service
Host: argocd.example.com  -> argocd Service
```

예시 Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host
spec:
  ingressClassName: nginx
  rules:
    - host: grafana.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 80
    - host: harbor.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: harbor
                port:
                  number: 80
```

### 9-2) 하나의 IP에 여러 HTTPS 사이트가 가능한 이유

과거에는 HTTPS 사이트마다 별도 IP가 필요한 경우가 많았다. 하지만 지금은 SNI(Server Name Indication)를 사용한다.

TLS Handshake 단계에서 클라이언트는 접속하려는 hostname을 서버에 전달한다.

```text
TLS ClientHello
  SNI: harbor.example.com
```

Ingress Controller는 이 SNI를 보고 적절한 인증서를 선택할 수 있다.

```text
SNI: harbor.example.com  -> harbor.example.com 인증서
SNI: grafana.example.com -> grafana.example.com 인증서
```

따라서 하나의 LoadBalancer IP와 하나의 443 포트로 여러 HTTPS 도메인을 처리할 수 있다.

### 9-3) Host가 맞지 않으면 어떻게 되는가

Ingress에 다음 rule이 있다고 가정한다.

```yaml
rules:
  - host: harbor.example.com
```

그런데 요청이 다음처럼 들어온다.

```http
Host: 203.0.113.10
```

또는

```http
Host: wrong.example.com
```

그러면 rule이 매칭되지 않는다.

가능한 결과:

```text
default backend로 전달
404 Not Found
Controller 기본 응답 반환
다른 catch-all Ingress로 전달
```

테스트:

```bash
curl -v http://203.0.113.10/
curl -v http://203.0.113.10/ -H 'Host: harbor.example.com'
```

첫 번째는 실패하고 두 번째는 성공한다면 Host-based routing 문제일 가능성이 높다.

## 10. Path-based Routing

### 10-1) Path Routing 구조

Path-based routing은 URL Path를 기준으로 Service를 나눈다.

예시:

```text
https://app.example.com/       -> frontend Service
https://app.example.com/api    -> backend Service
https://app.example.com/admin  -> admin Service
```

Ingress YAML:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
```

### 10-2) pathType

Kubernetes Ingress의 `pathType`은 Path 매칭 방식을 나타낸다.

| pathType | 의미 |
|---|---|
| `Exact` | Path가 정확히 일치할 때만 매칭 |
| `Prefix` | `/`로 구분되는 path prefix 기준 매칭 |
| `ImplementationSpecific` | Controller 구현체에 위임 |

예시:

```yaml
path: /api
pathType: Exact
```

이 경우 `/api`만 매칭된다. `/api/users`는 매칭되지 않는다.

```yaml
path: /api
pathType: Prefix
```

이 경우 `/api`, `/api/`, `/api/users`가 매칭될 수 있다.

### 10-3) Path 우선순위

여러 path가 있을 때 더 구체적인 path가 우선되는 방식으로 동작한다. 다만 세부 동작은 Controller 구현체와 설정에 영향을 받을 수 있으므로 운영 환경에서는 명확한 path 설계가 필요하다.

권장 구조:

```text
/api      -> API
/admin    -> Admin
/static   -> Static
/         -> Frontend
```

주의할 구조:

```text
/app
/app2
/application
```

Prefix 매칭에서 의도하지 않은 라우팅이 발생할 수 있으므로 정확한 pathType과 rewrite 동작을 반드시 확인해야 한다.

## 11. Regex Path와 ImplementationSpecific

### 11-1) Regex Path가 필요한 경우

기본 Ingress는 Host와 Path 중심의 비교적 단순한 라우팅을 제공한다. 하지만 실제 운영에서는 다음처럼 정규식 기반 매칭이 필요할 수 있다.

```text
/api/v1/users
/api/v2/users
/files/123/download
```

ingress-nginx에서는 annotation을 통해 regex path를 사용할 수 있다.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
```

예시:

```yaml
paths:
  - path: /api(/|$)(.*)
    pathType: ImplementationSpecific
    backend:
      service:
        name: backend
        port:
          number: 80
```

### 11-2) Regex와 Rewrite는 주의해야 한다

regex와 rewrite를 함께 쓰면 요청 path가 예상과 다르게 바뀔 수 있다.

예를 들어 외부 요청은 다음과 같다.

```text
/api/users
```

Backend 애플리케이션은 다음 path를 기대한다.

```text
/users
```

이 경우 `/api` prefix를 제거하는 rewrite가 필요하다.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
```

```yaml
paths:
  - path: /api(/|$)(.*)
    pathType: ImplementationSpecific
```

하지만 rewrite 설정이 틀리면 다음 문제가 발생한다.

```text
/api/users -> /api/users 그대로 전달
/api/users -> / 로 전달
/api/users -> /users가 아니라 /api로 전달
```

결과적으로 backend에서 404가 발생한다.

## 12. Path Rewrite

### 12-1) Rewrite가 필요한 이유

Ingress는 외부 URL 구조와 내부 애플리케이션 URL 구조를 분리할 때 사용된다.

외부 노출 URL:

```text
https://app.example.com/api/users
```

Backend 애플리케이션이 기대하는 URL:

```text
http://backend.default.svc.cluster.local/users
```

즉, 외부의 `/api` prefix를 내부로 보낼 때 제거해야 한다.

### 12-2) ingress-nginx rewrite 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-example
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: backend
                port:
                  number: 80
```

요청 변환:

```text
/app.example.com/api/users -> backend /users
/app.example.com/api       -> backend /
```

### 12-3) Rewrite 장애 패턴

| 증상 | 가능 원인 |
|---|---|
| Ingress는 매칭되지만 backend 404 | rewrite-target 오류 |
| `/api`는 되는데 `/api/users`는 안 됨 | regex capture group 오류 |
| static file 경로 깨짐 | 애플리케이션 base path와 Ingress path 불일치 |
| redirect가 잘못된 path로 감 | backend가 외부 prefix를 모름 |
| OAuth callback path 오류 | redirect_uri와 rewrite 결과 불일치 |

## 13. TLS 기본 구조

### 13-1) TLS가 Ingress에서 중요한 이유

Ingress는 보통 외부 HTTPS 진입점이다.

```text
Client --HTTPS--> Ingress Controller --HTTP or HTTPS--> Service --> Pod
```

사용자 관점에서는 주소창에 자물쇠가 보이는지, 인증서가 유효한지가 중요하다. 운영자 관점에서는 다음이 중요하다.

```text
인증서가 올바른 도메인을 포함하는가
Secret에 tls.crt/tls.key가 올바르게 들어있는가
Ingress tls.hosts와 rules.host가 일치하는가
SNI와 Host Header가 일치하는가
TLS Termination 위치가 어디인가
Backend로 HTTP를 보낼지 HTTPS를 보낼지 정했는가
```

### 13-2) TLS 인증서 구성 요소

TLS 인증서에는 보통 다음 정보가 들어있다.

| 항목 | 설명 |
|---|---|
| Subject | 인증서 대상 정보 |
| CN | Common Name. 과거에는 대표 도메인으로 많이 사용 |
| SAN | Subject Alternative Name. 현재 도메인 검증에서 중요 |
| Issuer | 인증서를 발급한 CA |
| Validity | 유효기간 |
| Public Key | 공개키 |
| Signature | CA 서명 |

도메인 검증에서 중요한 것은 SAN이다.

예를 들어 다음 인증서가 있다고 하자.

```text
SAN:
  DNS: harbor.example.com
  DNS: grafana.example.com
```

이 인증서는 `harbor.example.com`, `grafana.example.com`에 사용할 수 있다. 하지만 `argocd.example.com`에는 사용할 수 없다.

### 13-3) Wildcard Certificate

Wildcard 인증서는 여러 하위 도메인을 하나의 인증서로 처리할 때 사용한다.

예시:

```text
*.example.com
```

이 인증서는 다음에 사용할 수 있다.

```text
harbor.example.com
grafana.example.com
argocd.example.com
```

하지만 일반적으로 다음에는 직접 매칭되지 않는다.

```text
a.b.example.com
```

Wildcard 범위는 인증서 발급 정책과 브라우저 검증 규칙을 정확히 확인해야 한다.

## 14. TLS Secret

### 14-1) kubernetes.io/tls Secret

Kubernetes Ingress에서 TLS 인증서를 사용하려면 보통 `kubernetes.io/tls` 타입 Secret을 만든다.

```bash
kubectl create secret tls app-tls \
  --cert=./tls.crt \
  --key=./tls.key \
  -n default
```

생성 결과:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-tls
type: kubernetes.io/tls
data:
  tls.crt: <base64>
  tls.key: <base64>
```

Ingress에서 참조:

```yaml
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
```

### 14-2) Secret namespace 주의

TLS Secret은 Ingress와 같은 namespace에 있어야 한다.

```text
Ingress namespace: default
Secret namespace: default
```

다음처럼 되어 있으면 문제가 된다.

```text
Ingress namespace: default
Secret namespace: cert-manager
```

Ingress Controller에 따라 cross namespace 참조를 별도 기능으로 지원할 수 있지만, 기본 Ingress TLS Secret은 같은 namespace 참조가 원칙이다.

확인:

```bash
kubectl get secret -n default app-tls
kubectl describe ingress -n default app-ingress
```

### 14-3) 인증서 내용 확인

Secret에 들어있는 인증서를 확인하려면 다음처럼 한다.

```bash
kubectl get secret app-tls -o jsonpath='{.data.tls\.crt}' \
  | base64 -d \
  | openssl x509 -noout -text
```

유효기간 확인:

```bash
kubectl get secret app-tls -o jsonpath='{.data.tls\.crt}' \
  | base64 -d \
  | openssl x509 -noout -dates
```

SAN 확인:

```bash
kubectl get secret app-tls -o jsonpath='{.data.tls\.crt}' \
  | base64 -d \
  | openssl x509 -noout -text \
  | grep -A1 "Subject Alternative Name"
```

## 15. SNI

### 15-1) SNI가 필요한 이유

하나의 IP와 하나의 443 포트에서 여러 HTTPS 사이트를 운영하려면, 서버는 TLS Handshake 초기에 클라이언트가 어떤 도메인으로 접속하려는지 알아야 한다.

이 정보를 제공하는 TLS 확장이 SNI(Server Name Indication)다.

```text
ClientHello
  SNI: harbor.example.com
```

Ingress Controller는 SNI를 보고 적절한 인증서를 선택한다.

```text
SNI harbor.example.com  -> harbor-tls Secret
SNI grafana.example.com -> grafana-tls Secret
SNI argocd.example.com  -> argocd-tls Secret
```

### 15-2) SNI와 Host Header는 다르다

SNI는 TLS Handshake 단계에서 사용된다.

```text
TLS Layer
  SNI: harbor.example.com
```

Host Header는 HTTP Request 단계에서 사용된다.

```http
GET / HTTP/1.1
Host: harbor.example.com
```

HTTPS 요청에서는 보통 SNI와 Host Header가 같다. 하지만 테스트를 잘못하면 다를 수 있다.

잘못된 테스트:

```bash
curl -vk https://203.0.113.10/ -H 'Host: harbor.example.com'
```

이 경우 HTTP Host Header는 `harbor.example.com`일 수 있지만, TLS SNI는 IP 주소 기준으로 처리될 수 있다. 인증서 매칭이 깨질 수 있다.

올바른 테스트:

```bash
curl -vk --resolve harbor.example.com:443:203.0.113.10 https://harbor.example.com/
```

이 명령은 SNI와 Host를 모두 `harbor.example.com`으로 맞춘다.

## 16. TLS Termination, Re-encrypt, SSL Passthrough

### 16-1) TLS Termination

TLS Termination은 Ingress Controller가 HTTPS를 해제하고 내부로 HTTP를 전달하는 방식이다.

```text
Client --HTTPS--> Ingress Controller --HTTP--> Service --HTTP--> Pod
```

장점:

```text
Ingress에서 인증서 중앙 관리
Host/Path/Header 기반 L7 라우팅 가능
X-Forwarded-* Header 설정 가능
WAF, Rate Limit, Auth, Rewrite 적용 가능
```

주의점:

```text
Ingress와 Pod 사이 구간은 HTTP
내부망 암호화가 필요한 환경에서는 부적합할 수 있음
애플리케이션이 원래 요청이 HTTPS였는지 알 수 있도록 X-Forwarded-Proto 필요
```

### 16-2) Re-encrypt

Re-encrypt는 Ingress에서 외부 TLS를 해제한 뒤, backend로 다시 HTTPS를 사용하는 방식이다.

```text
Client --HTTPS--> Ingress Controller --HTTPS--> Service --HTTPS--> Pod
```

장점:

```text
외부 구간과 내부 구간 모두 암호화
Ingress에서 L7 라우팅 가능
```

주의점:

```text
Backend 인증서 검증 설정 필요
Service backend protocol을 HTTPS로 지정해야 할 수 있음
자체 서명 인증서 사용 시 Controller 설정 필요
```

ingress-nginx 예시:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
```

### 16-3) SSL Passthrough

SSL Passthrough는 Ingress Controller가 TLS를 해제하지 않고 TCP 그대로 backend로 전달하는 방식이다.

```text
Client --HTTPS--> Ingress Controller --HTTPS 그대로 전달--> Pod
```

장점:

```text
Backend가 직접 TLS 종료
Ingress가 인증서 개인키를 가지지 않아도 됨
mTLS처럼 backend에서 직접 인증을 처리하는 구조에 유리
```

제약:

```text
HTTP Path/Header 기반 라우팅 불가
SNI 기반 라우팅 정도만 가능
Controller별 별도 옵션 필요
L7 기능 사용 제한
```

즉, SSL Passthrough는 L7 Ingress 기능을 대부분 포기하고 L4/TLS 수준 전달에 가까워진다.

## 17. X-Forwarded-* Header

### 17-1) 왜 필요한가

Ingress Controller는 reverse proxy다. Client는 실제 backend Pod에 직접 접속하지 않는다.

```text
Client
  ↓
Load Balancer
  ↓
Ingress Controller
  ↓
Application Pod
```

애플리케이션 입장에서 직접 연결한 상대는 Client가 아니라 Ingress Controller다.

따라서 애플리케이션이 원래 요청 정보를 알 수 있도록 프록시는 Header를 추가한다.

대표 Header:

```http
X-Forwarded-For: 203.0.113.10
X-Forwarded-Host: harbor.example.com
X-Forwarded-Proto: https
X-Real-IP: 203.0.113.10
```

### 17-2) X-Forwarded-For

원래 Client IP를 전달한다.

```http
X-Forwarded-For: 203.0.113.10
```

여러 프록시를 거치면 값이 누적된다.

```http
X-Forwarded-For: 203.0.113.10, 10.0.0.20, 10.0.0.30
```

일반적으로 왼쪽 첫 번째가 원래 Client IP에 가깝지만, 이 값은 클라이언트가 조작할 수 있으므로 신뢰할 수 있는 프록시가 설정한 값만 신뢰해야 한다.

### 17-3) X-Forwarded-Host

원래 요청의 Host를 전달한다.

```http
X-Forwarded-Host: harbor.example.com
```

애플리케이션이 절대 URL을 만들 때 사용될 수 있다.

예를 들어 로그인 후 redirect URL을 생성할 때 다음처럼 사용된다.

```text
https://harbor.example.com/callback
```

이 값이 잘못되면 redirect가 내부 Service 이름이나 Pod IP로 생성될 수 있다.

### 17-4) X-Forwarded-Proto

원래 요청이 HTTP였는지 HTTPS였는지 전달한다.

```http
X-Forwarded-Proto: https
```

이 Header는 OAuth/OIDC, Keycloak, Harbor, Grafana, ArgoCD에서 특히 중요하다.

예를 들어 외부 요청은 HTTPS인데 Ingress가 내부로 HTTP를 전달한다고 가정한다.

```text
Client --HTTPS--> Ingress --HTTP--> Harbor
```

Harbor 입장에서는 직접 받은 요청이 HTTP처럼 보일 수 있다. 이때 `X-Forwarded-Proto: https`가 없으면 Harbor는 redirect URL을 `http://harbor.example.com`으로 만들 수 있다.

결과:

```text
HTTPS 접속
  ↓
애플리케이션이 HTTP로 redirect
  ↓
Ingress가 다시 HTTPS로 redirect
  ↓
무한 redirect 또는 로그인 실패
```

### 17-5) Forwarded Header

표준 Header로는 `Forwarded`도 있다.

```http
Forwarded: for=203.0.113.10;proto=https;host=harbor.example.com
```

하지만 실무에서는 `X-Forwarded-*`가 더 널리 쓰인다. 어떤 Header를 신뢰할지는 애플리케이션과 프록시 설정을 함께 맞춰야 한다.

## 18. OAuth/OIDC와 Ingress

### 18-1) OAuth/OIDC 흐름에서 Ingress가 중요한 이유

Harbor, ArgoCD, Grafana, 사내 포털은 Keycloak 같은 IdP와 OIDC로 로그인할 수 있다.

단순화한 흐름:

```text
1. User -> Harbor 접속
2. Harbor -> Keycloak 로그인 페이지로 redirect
3. User -> Keycloak 로그인
4. Keycloak -> Harbor callback URL로 redirect
5. Harbor -> authorization code를 token으로 교환
6. 로그인 완료
```

이때 중요한 값은 redirect URI다.

```text
https://harbor.example.com/c/oidc/callback
```

이 URL은 Keycloak Client 설정과 애플리케이션 설정, Ingress Host/TLS 설정이 모두 일치해야 한다.

### 18-2) OAuth Callback 오류 원인

자주 발생하는 오류:

```text
Invalid redirect_uri
redirect_uri mismatch
400 Bad Request
로그인 후 다시 로그인 화면으로 돌아감
무한 redirect
callback 404
```

원인 후보:

```text
Ingress Host와 Keycloak Client redirect URI 불일치
X-Forwarded-Proto 누락
X-Forwarded-Host 누락
애플리케이션 external URL 설정 오류
Path rewrite로 callback path가 변경됨
TLS Termination 후 backend가 HTTP로 인식
Cookie SameSite/Secure 설정 불일치
```

### 18-3) Keycloak과 Forwarded Header

Keycloak은 reverse proxy 뒤에서 동작할 때 hostname과 proxy header 설정이 중요하다.

예를 들어 외부 접속 주소는 다음과 같다.

```text
https://keycloak.example.com
```

하지만 Keycloak Pod가 보는 요청은 다음일 수 있다.

```text
http://keycloak-http.keycloak.svc.cluster.local:8080
```

이때 Keycloak이 원래 외부 주소를 알지 못하면 잘못된 issuer, redirect URL, cookie domain을 만들 수 있다.

Ingress 또는 앞단 HAProxy는 다음 정보를 정확히 전달해야 한다.

```http
Host: keycloak.example.com
X-Forwarded-Host: keycloak.example.com
X-Forwarded-Proto: https
X-Forwarded-For: <client-ip>
```

그리고 Keycloak 자체도 reverse proxy 환경을 인식하도록 설정해야 한다. 정확한 옵션명은 Keycloak 버전과 배포 방식에 따라 다르므로 해당 버전 공식 문서를 확인해야 한다.

## 19. Cookie, Session, Sticky Session

### 19-1) 왜 Sticky Session이 필요한가

정상적인 클라우드 네이티브 애플리케이션은 세션 상태를 Pod 메모리에만 저장하지 않아야 한다. Redis, DB, 외부 세션 저장소를 사용하는 것이 일반적이다.

하지만 일부 애플리케이션은 로그인 세션이나 임시 상태를 특정 Pod 메모리에 저장한다.

```text
첫 번째 요청 -> Pod A 로그인 세션 생성
두 번째 요청 -> Pod B로 전달
Pod B는 세션을 모름
로그아웃처럼 보임
```

이 문제를 완화하기 위해 Sticky Session을 사용할 수 있다.

### 19-2) Cookie Affinity

ingress-nginx는 annotation으로 cookie 기반 affinity를 지원한다.

예시:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "INGRESSCOOKIE"
```

동작:

```text
첫 요청
  ↓
Ingress가 Pod A 선택
  ↓
Set-Cookie: INGRESSCOOKIE=...
  ↓
다음 요청에서 Cookie 전송
  ↓
Ingress가 같은 Pod A로 전달
```

주의점:

```text
Pod A가 죽으면 세션이 끊길 수 있음
수평 확장성과 장애 복구에 불리할 수 있음
가능하면 애플리케이션 세션을 외부 저장소로 분리하는 것이 좋음
```

## 20. Annotation

### 20-1) Annotation이 필요한 이유

Kubernetes Ingress 표준은 일부 기본 기능만 제공한다.

기본적으로 표현 가능한 것은 대략 다음이다.

```text
Host routing
Path routing
TLS Secret
Default backend
```

하지만 실제 운영에서는 다음 설정이 필요하다.

```text
rewrite
timeout
body size
rate limit
IP whitelist
CORS
authentication
sticky session
canary
backend protocol
proxy buffering
custom header
```

이런 기능은 Controller별 annotation으로 확장한다.

### 20-2) ingress-nginx 주요 annotation

대표적인 ingress-nginx annotation:

| Annotation | 용도 |
|---|---|
| `nginx.ingress.kubernetes.io/rewrite-target` | Path rewrite |
| `nginx.ingress.kubernetes.io/use-regex` | Regex path 사용 |
| `nginx.ingress.kubernetes.io/proxy-read-timeout` | backend 응답 대기 시간 |
| `nginx.ingress.kubernetes.io/proxy-send-timeout` | backend 전송 timeout |
| `nginx.ingress.kubernetes.io/proxy-body-size` | 요청 body 최대 크기 |
| `nginx.ingress.kubernetes.io/backend-protocol` | backend HTTP/HTTPS 지정 |
| `nginx.ingress.kubernetes.io/whitelist-source-range` | 허용 source CIDR |
| `nginx.ingress.kubernetes.io/limit-rps` | 초당 요청 제한 |
| `nginx.ingress.kubernetes.io/affinity` | session affinity |
| `nginx.ingress.kubernetes.io/canary` | canary Ingress 활성화 |

예시:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
```

### 20-3) Annotation은 Controller 종속적이다

중요한 점은 annotation은 표준 Kubernetes API 의미가 아니라 Controller 구현체별 확장이라는 것이다.

예를 들어 ingress-nginx annotation은 HAProxy Ingress나 Traefik에서 그대로 동작하지 않을 수 있다.

```text
nginx.ingress.kubernetes.io/rewrite-target
```

이 값은 ingress-nginx 전용이다.

따라서 Ingress Controller를 바꾸면 annotation 호환성을 반드시 검토해야 한다.

## 21. Rate Limit

### 21-1) Rate Limit이 필요한 이유

Rate Limit은 특정 클라이언트 또는 경로에 대해 요청량을 제한하는 기능이다.

사용 목적:

```text
과도한 요청 방지
간단한 DDoS 완화
비정상 봇 제한
로그인 API 보호
비싼 API 보호
```

ingress-nginx 예시:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
```

의미:

```text
초당 요청 수 제한
```

정확한 동작은 Controller 설정과 annotation 조합에 따라 달라진다.

### 21-2) Rate Limit 주의점

프록시가 여러 단계일 때 source IP가 모두 동일하게 보일 수 있다.

```text
Client
  ↓
External LB
  ↓
Ingress
```

Ingress 입장에서 모든 요청 source가 External LB IP로 보이면, Rate Limit이 전체 사용자에게 동시에 적용될 수 있다. 이 경우 `X-Forwarded-For` 신뢰 설정과 real IP 설정이 중요하다.

## 22. IP Whitelist

### 22-1) 사내망만 접근 허용

관리 도구는 외부 전체에 노출하면 위험하다.

예시:

```text
Grafana
Prometheus
ArgoCD
Harbor Admin
Keycloak Admin Console
```

이런 서비스는 사내망, VPN, Bastion 대역만 접근하도록 제한하는 것이 좋다.

ingress-nginx 예시:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.10.0/24,10.0.0.0/8"
```

### 22-2) Whitelist가 안 먹는 경우

원인 후보:

```text
Ingress가 실제 client IP를 못 보고 LB IP만 봄
externalTrafficPolicy 설정 문제
real-ip-header 설정 문제
X-Forwarded-For 신뢰 범위 문제
앞단 HAProxy가 client IP를 덮어씀
```

확인:

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

Access log에 찍히는 remote address를 확인해야 한다.

## 23. Canary 배포

### 23-1) Canary란 무엇인가

Canary 배포는 새 버전을 일부 트래픽에만 노출해 안정성을 확인하는 방식이다.

```text
90% -> stable Service
10% -> canary Service
```

문제가 없으면 canary 비율을 점차 늘린다.

```text
10%
30%
50%
100%
```

### 23-2) ingress-nginx Canary 예시

Stable Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-stable
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-stable
                port:
                  number: 80
```

Canary Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-canary
                port:
                  number: 80
```

의미:

```text
app.example.com 요청 중 일부를 app-canary Service로 보낸다.
```

### 23-3) Header/Cookie 기반 Canary

특정 사용자 또는 테스트 요청만 canary로 보낼 수도 있다.

예시 개념:

```text
Header: X-Canary: always
Cookie: canary=true
```

이 방식은 QA, 내부 테스트, 점진적 배포에 유용하다.

## 24. Blue-Green 배포

### 24-1) Blue-Green 개념

Blue-Green 배포는 두 개의 환경을 준비하고 트래픽을 한 번에 전환하는 방식이다.

```text
Blue  = 현재 운영 버전
Green = 신규 버전
```

전환 전:

```text
Ingress -> blue Service
```

전환 후:

```text
Ingress -> green Service
```

### 24-2) Ingress 관점의 Blue-Green

Ingress backend Service를 바꾸거나, Service selector를 바꿔서 트래픽을 전환할 수 있다.

방법 1: Ingress backend 변경

```text
backend service: app-blue
  ↓ 변경
backend service: app-green
```

방법 2: Service selector 변경

```text
app Service selector:
  version: blue
  ↓
  version: green
```

방법 2는 Ingress를 건드리지 않고 Service 레벨에서 전환할 수 있다.

### 24-3) 주의점

```text
DB migration 호환성
세션 상태
캐시
파일 업로드 경로
Rollback 절차
Health check
```

Blue-Green은 네트워크 라우팅만 바꾸는 것이 아니라 애플리케이션 상태 전환까지 고려해야 한다.

## 25. Basic Auth와 외부 인증 연동

### 25-1) Basic Auth

간단한 내부 도구 보호에는 Basic Auth를 사용할 수 있다.

예시 용도:

```text
임시 개발 페이지
내부 문서
테스트 API
```

ingress-nginx는 annotation과 Secret으로 Basic Auth를 구성할 수 있다.

개념:

```text
Client
  ↓
Ingress에서 Authorization Header 확인
  ↓
인증 성공 시 backend 전달
```

### 25-2) 외부 인증

Ingress Controller에 따라 외부 인증 서버와 연동할 수 있다.

개념:

```text
Client -> Ingress
          ↓
       Auth Service 확인
          ↓
       Backend 전달 여부 결정
```

이 구조는 OAuth2 Proxy, OIDC Proxy, SSO 연동에서 자주 사용된다. 다만 Controller별 구현 방식이 다르고 보안 영향이 크므로 공식 문서를 기준으로 구성해야 한다.

## 26. Default Backend

### 26-1) Default Backend란

Default Backend는 어떤 Ingress rule에도 매칭되지 않는 요청을 처리하는 기본 backend다.

매칭 실패 예:

```text
Host 불일치
Path 불일치
IngressClass 불일치
TLS SNI 불일치
catch-all rule 없음
```

NGINX Ingress에서는 기본적으로 404를 반환하는 backend가 사용될 수 있다.

### 26-2) Default Backend 404 해석

다음 응답이 나온다고 하자.

```text
default backend - 404
```

이 경우 애플리케이션이 404를 준 것이 아닐 수 있다.

가능성이 높은 원인:

```text
Ingress rule까지 도달하지 못함
Host Header가 다름
Path가 매칭되지 않음
해당 Ingress Controller가 리소스를 처리하지 않음
```

확인:

```bash
curl -v http://<LB-IP>/ -H 'Host: app.example.com'
kubectl describe ingress <name>
kubectl get ingressclass
```

## 27. NGINX Ingress Controller

### 27-1) 특징

NGINX Ingress Controller는 Kubernetes에서 널리 쓰이는 Ingress Controller다.

주요 기능:

```text
Host/Path routing
TLS termination
Rewrite
Rate limiting
IP whitelist
Session affinity
Canary
Basic Auth
External Auth
Custom timeout
Body size 제한
WebSocket 지원
gRPC 지원
```

### 27-2) 구성 요소

일반적인 설치 구성:

```text
Namespace: ingress-nginx

Deployment:
  ingress-nginx-controller

Service:
  ingress-nginx-controller
  type: LoadBalancer 또는 NodePort

ConfigMap:
  controller global config

Admission Webhook:
  Ingress validation
```

확인:

```bash
kubectl get all -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get configmap -n ingress-nginx
```

### 27-3) NGINX 설정 확인

Controller Pod 안에서 NGINX 설정을 확인할 수 있다.

```bash
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- nginx -T
```

특정 host만 보고 싶으면 grep을 사용할 수 있다.

```bash
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- nginx -T | grep -A30 "server_name app.example.com"
```

이 명령은 Ingress YAML이 실제 NGINX 설정으로 어떻게 반영되었는지 확인할 때 유용하다.

## 28. HAProxy Ingress Controller

### 28-1) 특징

HAProxy Ingress Controller는 HAProxy를 기반으로 Kubernetes Ingress 리소스를 처리한다.

HAProxy의 강점:

```text
고성능 L4/L7 Proxy
정교한 ACL
다양한 Load Balancing Algorithm
상세한 Health Check
통계 페이지
TCP/HTTP 모드
운영 환경에서 검증된 프록시 엔진
```

Ingress와 함께 사용하면 Kubernetes 리소스 기반으로 HAProxy 설정을 동적으로 구성할 수 있다.

### 28-2) HAProxy와 Ingress

HAProxy Ingress Controller도 기본 구조는 같다.

```text
Ingress YAML
  ↓
HAProxy Ingress Controller
  ↓
HAProxy config
  ↓
frontend/backend routing
```

HAProxy 개념과 연결하면 다음과 같다.

| Kubernetes | HAProxy |
|---|---|
| Ingress host/path | ACL |
| Service | backend |
| EndpointSlice | server |
| TLS Secret | crt |
| Controller Service | frontend bind 대상 |

### 28-3) 언제 HAProxy Ingress를 고려할 수 있는가

```text
HAProxy 운영 경험이 많음
L4/TCP와 L7/HTTP를 함께 정교하게 다뤄야 함
ACL 기반 라우팅이 복잡함
HAProxy stats와 기존 모니터링 체계를 활용하고 싶음
NGINX annotation 호환성보다 HAProxy 기능이 중요함
```

## 29. Traefik

### 29-1) 특징

Traefik은 클라우드 네이티브 환경을 지향하는 reverse proxy이자 ingress controller다.

특징:

```text
Kubernetes 리소스 자동 감지
동적 설정 반영
Middleware 개념
Let's Encrypt 연동 편의성
Ingress와 CRD 지원
Gateway API 지원
```

### 29-2) Middleware 개념

Traefik은 Middleware를 통해 요청 처리 중간 단계를 구성할 수 있다.

예:

```text
Redirect
Header 추가
Rate Limit
Auth
Strip Prefix
Retry
```

NGINX Ingress가 annotation 중심이라면, Traefik은 별도 리소스와 middleware 조합으로 표현하는 경우가 많다.

## 30. Gateway API

### 30-1) Gateway API가 등장한 이유

Ingress는 오랫동안 사용되었지만 한계가 있다.

대표 한계:

```text
Host/Path 외의 복잡한 라우팅 표현이 부족
Controller별 annotation 의존도 높음
L4/L7 정책 표현이 제한적
역할 분리가 어려움
여러 팀이 공통 Gateway를 공유하기 어려움
```

그래서 Gateway API가 등장했다.

Gateway API는 Ingress보다 더 명확한 역할 분리와 확장성을 목표로 한다.

### 30-2) Gateway API 핵심 리소스

| 리소스 | 역할 |
|---|---|
| GatewayClass | Gateway 구현체 종류 |
| Gateway | 실제 진입점, Listener 정의 |
| HTTPRoute | HTTP 라우팅 규칙 |
| TLSRoute | TLS 라우팅 규칙 |
| TCPRoute | TCP 라우팅 규칙 |
| UDPRoute | UDP 라우팅 규칙 |
| ReferenceGrant | namespace 간 참조 허용 |

### 30-3) Ingress와 Gateway API 비교

| 항목 | Ingress | Gateway API |
|---|---|---|
| 라우팅 | Host/Path 중심 | HTTP/TCP/TLS/UDP 등 확장 |
| 확장 방식 | Annotation 의존 | 명시적 API 리소스 |
| 역할 분리 | 제한적 | Infra owner / App owner 분리 용이 |
| Cross namespace | 제한적 | ReferenceGrant 등으로 명시 가능 |
| 정책 표현 | Controller별 차이 큼 | 표준화된 방향 |

### 30-4) Gateway API 예시

Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: external-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: wildcard-example-tls
```

HTTPRoute:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
spec:
  parentRefs:
    - name: external-gateway
  hostnames:
    - app.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: backend
          port: 80
```

Gateway API는 Ingress를 당장 완전히 대체해야 한다는 의미라기보다, 더 복잡한 트래픽 관리가 필요한 환경에서 점진적으로 고려할 수 있는 다음 세대 API로 이해하면 된다.

## 31. Backend Protocol

### 31-1) Ingress에서 backend로 HTTP를 보낼지 HTTPS를 보낼지

Ingress TLS 설정은 Client와 Ingress 사이의 TLS를 의미한다.

```text
Client --HTTPS--> Ingress
```

하지만 Ingress에서 backend Service로 보낼 때는 별도다.

```text
Ingress --HTTP--> Backend
Ingress --HTTPS--> Backend
```

기본은 보통 HTTP다.

Backend가 HTTPS만 받는다면 Controller에 backend protocol을 알려줘야 한다.

ingress-nginx 예시:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
```

### 31-2) Backend Protocol 오류 증상

| 상황 | 증상 |
|---|---|
| Backend는 HTTP인데 Ingress가 HTTPS로 보냄 | 502, SSL handshake failed |
| Backend는 HTTPS인데 Ingress가 HTTP로 보냄 | 400, plain HTTP request sent to HTTPS port |
| Backend 인증서 검증 실패 | 502 |
| targetPort가 틀림 | connection refused, 502 |

## 32. WebSocket과 gRPC

### 32-1) WebSocket

WebSocket은 HTTP Upgrade를 사용해 장기 연결을 만든다.

```http
Connection: Upgrade
Upgrade: websocket
```

Ingress에서 WebSocket을 사용할 때는 timeout이 중요하다.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```

### 32-2) gRPC

gRPC는 일반적으로 HTTP/2 위에서 동작한다.

Ingress Controller가 gRPC를 지원하고, backend protocol 설정이 맞아야 한다.

ingress-nginx 예시:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
```

TLS를 사용하는 gRPC는 `GRPCS`를 사용하는 경우도 있다. 정확한 값은 Controller 문서를 확인해야 한다.

## 33. 실전 장애 사례

### 33-1) 404 Not Found

#### 증상

```text
curl 결과 404
브라우저에서 Not Found
default backend - 404
```

#### 원인 후보

```text
Host Header 불일치
Path 불일치
IngressClass 불일치
Ingress가 잘못된 namespace에 있음
Controller가 Ingress를 처리하지 않음
rewrite로 backend path가 잘못 바뀜
```

#### 확인

```bash
kubectl get ingress -A
kubectl describe ingress -n <ns> <name>
kubectl get ingressclass
curl -v http://<LB-IP>/ -H 'Host: app.example.com'
```

### 33-2) 502 Bad Gateway

#### 의미

Ingress Controller가 backend와 통신하려고 했지만 실패했다.

#### 원인 후보

```text
Service targetPort 오류
Pod가 해당 포트에서 listen하지 않음
Backend protocol 불일치
Pod 네트워크 문제
NetworkPolicy 차단
Backend Pod가 연결 직후 RST
```

#### 확인

```bash
kubectl get svc -n <ns> <service> -o yaml
kubectl get endpointslice -n <ns> -l kubernetes.io/service-name=<service>
kubectl exec -n <ns> <pod> -- ss -lntp
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

### 33-3) 503 Service Unavailable

#### 의미

Ingress가 보낼 backend endpoint가 없거나 사용할 수 없다.

#### 원인 후보

```text
Service selector와 Pod label 불일치
Pod Ready가 아님
Readiness Probe 실패
EndpointSlice 비어 있음
Service port 이름/번호 불일치
```

#### 확인

```bash
kubectl get pod -n <ns> --show-labels
kubectl get svc -n <ns> <service> -o yaml
kubectl get endpoints -n <ns> <service>
kubectl get endpointslice -n <ns> -l kubernetes.io/service-name=<service>
kubectl describe pod -n <ns> <pod>
```

### 33-4) 504 Gateway Timeout

#### 의미

Ingress가 backend로 요청을 보냈지만 제한 시간 내 응답을 받지 못했다.

#### 원인 후보

```text
애플리케이션 응답 지연
DB 지연
외부 API 지연
proxy-read-timeout 부족
backend connection pool 고갈
Pod CPU throttling
```

#### 확인

```bash
kubectl top pod -n <ns>
kubectl logs -n <ns> <pod>
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

관련 annotation:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
```

단, timeout을 늘리는 것은 증상 완화일 뿐이고 애플리케이션 지연 원인을 같이 봐야 한다.

### 33-5) 400 Bad Request

#### 원인 후보

```text
Host Header가 애플리케이션 허용 목록에 없음
큰 Header 또는 Body
잘못된 HTTP 요청
HTTPS 포트에 HTTP 요청
Backend protocol 불일치
Keycloak/Harbor/Grafana의 external URL 설정 오류
```

예시:

```text
plain HTTP request was sent to HTTPS port
```

이 경우 HTTP/HTTPS 포트 또는 backend-protocol을 확인해야 한다.

### 33-6) TLS 인증서 오류

#### 증상

```text
NET::ERR_CERT_COMMON_NAME_INVALID
certificate has expired
unknown authority
self signed certificate
```

#### 원인 후보

```text
인증서 SAN에 도메인 없음
인증서 만료
Secret에 잘못된 인증서 저장
Secret namespace 불일치
Ingress tls.hosts와 rules.host 불일치
중간 인증서 chain 누락
```

확인:

```bash
openssl s_client -connect harbor.example.com:443 -servername harbor.example.com -showcerts
```

Kubernetes Secret 확인:

```bash
kubectl get secret -n <ns> <secret> -o jsonpath='{.data.tls\.crt}' \
  | base64 -d \
  | openssl x509 -noout -text
```

### 33-7) Redirect Loop

#### 증상

```text
ERR_TOO_MANY_REDIRECTS
http -> https -> http -> https 반복
로그인 페이지가 계속 반복
```

#### 원인 후보

```text
X-Forwarded-Proto 누락
애플리케이션 external URL이 http로 설정
Ingress ssl-redirect와 앱 redirect 정책 충돌
앞단 LB에서 TLS 종료 후 Ingress에 HTTP로 전달
Ingress도 다시 HTTPS redirect 수행
```

확인:

```bash
curl -vk -L https://app.example.com/
curl -vk https://app.example.com/ -I
```

응답 Header의 `Location`을 본다.

```http
Location: http://app.example.com/login
```

외부는 HTTPS인데 Location이 HTTP라면 proxy header 또는 앱 external URL 설정 문제일 가능성이 높다.

### 33-8) Harbor OAuth 실패

#### 증상

```text
Harbor에서 OIDC 로그인 후 오류
redirect_uri mismatch
로그인 후 다시 로그인 화면
400 Bad Request
```

#### 원인 후보

```text
Harbor external_url 불일치
Keycloak Client redirect URI 불일치
X-Forwarded-Proto 누락
X-Forwarded-Host 누락
Ingress path rewrite로 callback path 변경
Cookie Secure/SameSite 설정 문제
```

점검 순서:

```text
1. Harbor external URL 확인
2. Keycloak Client redirect URI 확인
3. Ingress Host 확인
4. TLS Termination 위치 확인
5. X-Forwarded-Proto가 https인지 확인
6. Browser 개발자 도구에서 redirect Location 확인
```

### 33-9) Keycloak 로그인 실패

#### 증상

```text
Invalid parameter: redirect_uri
Invalid issuer
로그인 후 403/400
관리 콘솔 URL이 내부 주소로 생성
```

#### 원인 후보:

```text
Keycloak hostname 설정 오류
proxy header 설정 오류
X-Forwarded-* 누락
Ingress Host와 Keycloak frontend URL 불일치
TLS Termination 후 Keycloak이 HTTP로 인식
```

### 33-10) ArgoCD HTTPS Redirect 문제

ArgoCD는 자체적으로 HTTPS redirect를 수행할 수 있고, Ingress에서도 SSL redirect를 수행할 수 있다.

문제 구조:

```text
Client HTTPS
  ↓
Ingress TLS Termination
  ↓ HTTP
ArgoCD가 HTTP로 인식
  ↓
HTTPS로 redirect
  ↓
Ingress와 충돌
```

해결 방향은 환경마다 다르지만 일반적으로 다음 중 하나를 명확히 정해야 한다.

```text
Ingress에서 TLS 종료하고 ArgoCD는 insecure HTTP backend로 운영
또는 ArgoCD까지 HTTPS backend로 연결
```

## 34. Ingress 트러블슈팅 순서

### 34-1) 전체 흐름 기준

문제가 생겼을 때는 다음 순서로 확인한다.

```text
1. DNS
2. External Load Balancer
3. Ingress Controller Service
4. IngressClass
5. Ingress Rule
6. TLS Secret
7. Service
8. EndpointSlice
9. Pod Readiness
10. Application Log
```

### 34-2) 기본 확인 명령어

```bash
kubectl get ingress -A
kubectl describe ingress -n <namespace> <ingress-name>
kubectl get ingressclass
kubectl get svc -A
kubectl get endpoints -A
kubectl get endpointslice -A
kubectl get pod -A -o wide
```

Ingress Controller 확인:

```bash
kubectl get pod -n ingress-nginx -o wide
kubectl get svc -n ingress-nginx
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

### 34-3) Host Header 테스트

```bash
curl -v http://<LB-IP>/ -H 'Host: app.example.com'
```

HTTPS + SNI 테스트:

```bash
curl -vk --resolve app.example.com:443:<LB-IP> https://app.example.com/
```

Header 확인:

```bash
curl -vk -I https://app.example.com/
```

Redirect 확인:

```bash
curl -vk -L https://app.example.com/
```

### 34-4) Backend 직접 테스트

Ingress를 우회하고 Service를 직접 확인한다.

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
```

Pod 내부에서:

```bash
nslookup backend.default.svc.cluster.local
curl -v http://backend.default.svc.cluster.local:80/health
```

Service가 되지 않으면 Endpoint 확인:

```bash
kubectl get svc backend -o yaml
kubectl get endpoints backend
kubectl get endpointslice -l kubernetes.io/service-name=backend
```

Pod 포트 확인:

```bash
kubectl exec -it <backend-pod> -- ss -lntp
```

### 34-5) TLS 확인

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com -showcerts
```

확인할 것:

```text
인증서 CN/SAN
issuer
유효기간
chain
SNI에 따른 인증서 선택
```

### 34-6) NGINX Ingress 내부 설정 확인

```bash
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- nginx -T
```

특정 host 검색:

```bash
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- nginx -T | grep -A50 "server_name app.example.com"
```

### 34-7) Access Log / Error Log 확인

Ingress Controller 로그에서 다음을 본다.

```text
status code
upstream address
upstream response time
request time
host
path
user agent
client ip
```

ingress-nginx access log 예시 형태:

```text
203.0.113.10 - - [time] "GET /api HTTP/1.1" 502 ...
```

502/503/504는 애플리케이션 로그와 Ingress 로그를 함께 봐야 한다.

## 35. Ingress 설계 시 권장 원칙

### 35-1) Host와 Path를 단순하게 설계한다

권장:

```text
harbor.example.com/
grafana.example.com/
argocd.example.com/
api.example.com/
```

복잡한 path rewrite를 남발하면 장애 분석이 어려워진다.

가능하면 애플리케이션별로 별도 subdomain을 사용하는 것이 운영상 단순하다.

### 35-2) TLS 종료 위치를 명확히 한다

다음 중 하나를 명확히 정한다.

```text
Edge TLS Termination
Re-encrypt
SSL Passthrough
```

혼합하면 redirect, cookie secure, OAuth callback, backend protocol 문제가 자주 발생한다.

### 35-3) X-Forwarded-* Header 정책을 통일한다

앞단에 HAProxy나 외부 Load Balancer가 있고 그 뒤에 Ingress가 있다면 어느 계층에서 어떤 Header를 설정하고 신뢰할지 정해야 한다.

```text
HAProxy
  ↓ X-Forwarded-For 추가
Ingress
  ↓ X-Forwarded-Proto 설정
Application
```

### 35-4) 운영 도구는 접근 제어를 둔다

다음 서비스는 외부 전체 공개를 피하는 것이 좋다.

```text
Grafana
Prometheus
ArgoCD
Harbor Admin
Keycloak Admin
Kubernetes Dashboard
```

권장:

```text
VPN
Bastion
IP whitelist
SSO
mTLS
WAF
```

### 35-5) Ingress Controller별 annotation lock-in을 인지한다

ingress-nginx annotation에 강하게 의존하면 HAProxy, Traefik, Gateway API로 전환할 때 이관 비용이 생긴다.

가능하면 표준 Ingress 또는 Gateway API로 표현 가능한 부분과 Controller 종속 기능을 구분해서 문서화해야 한다.

## 36. 요약

Ingress는 Kubernetes에서 외부 HTTP/HTTPS 요청을 내부 Service로 라우팅하기 위한 표준 리소스다. 그러나 실제 요청을 처리하는 것은 Ingress 자체가 아니라 Ingress Controller다.

핵심 구조는 다음과 같다.

```text
Client
  ↓
DNS
  ↓
Load Balancer
  ↓
Ingress Controller
  ↓
Ingress Rule
  ↓
Service
  ↓
EndpointSlice
  ↓
Pod
```

Ingress를 제대로 이해하려면 다음 개념을 함께 알아야 한다.

```text
L4 vs L7
Reverse Proxy
Host Header
Path Routing
TLS Termination
SNI
TLS Secret
X-Forwarded-For
X-Forwarded-Host
X-Forwarded-Proto
Service
EndpointSlice
Readiness
Annotation
OAuth/OIDC
Gateway API
```

실무 장애는 대부분 다음 지점에서 발생한다.

```text
Host/Path 미매칭 -> 404
Endpoint 없음 -> 503
Backend 연결 실패 -> 502
Backend 지연 -> 504
TLS Secret 문제 -> 인증서 오류
X-Forwarded-Proto 문제 -> Redirect Loop
OIDC redirect_uri 불일치 -> 로그인 실패
```

따라서 Ingress 트러블슈팅은 단순히 `kubectl get ingress`만 보는 것이 아니라, DNS부터 Pod 로그까지 End-to-End로 따라가야 한다.

```bash
kubectl get ingress -A
kubectl describe ingress -n <ns> <name>
kubectl get ingressclass
kubectl get svc -n <ns>
kubectl get endpointslice -n <ns>
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
curl -vk --resolve app.example.com:443:<LB-IP> https://app.example.com/
```

Ingress는 이후 HAProxy, Load Balancer, Observability, 실전 RCA 문서와 직접 연결되는 핵심 주제다.
