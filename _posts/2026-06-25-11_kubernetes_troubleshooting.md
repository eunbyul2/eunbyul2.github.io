---
layout: post
title: "Kubernetes 네트워크 트러블슈팅: logs, describe, svc, endpoints, port-forward"
date: 2026-06-25 08:30:00 +0900
categories: [Kubernetes, Troubleshooting]
tags: [Kubernetes, kubectl, logs, describe, ingress, svc, endpoints, endpointslice, port-forward, NetworkPolicy, CoreDNS]
published: true
---

## 1. 이 문서의 목적

이 문서는 Kubernetes 환경에서 네트워크 장애가 발생했을 때 `kubectl` 명령어를 단순히 나열하는 문서가 아니라, **요청이 Kubernetes 안에서 어떤 리소스를 거쳐 흐르는지 이해하고 어디에서 끊겼는지 찾는 방법**을 정리한 문서입니다.

앞선 문서에서 Linux 네트워크, HAProxy, `curl`, `tcpdump`, `ss`, `dig`, `nc` 같은 도구를 다뤘다면, 이 문서는 그 도구들을 Kubernetes 리소스 흐름에 맞춰 적용하는 단계입니다.

공식 참고자료:

- Kubernetes Debug Services: https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/
- kubectl Cheat Sheet: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- Kubernetes Ingress: https://kubernetes.io/docs/concepts/services-networking/ingress/
- Kubernetes Service: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes EndpointSlices: https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/
- Kubernetes DNS for Services and Pods: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- Kubernetes Network Policies: https://kubernetes.io/docs/concepts/services-networking/network-policies/
- Debug Running Pods: https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/

----

## 2. Kubernetes 네트워크 트러블슈팅의 기본 사고방식

Kubernetes 네트워크 장애는 대부분 한 리소스만 보고 판단하면 안 됩니다.

외부에서 애플리케이션으로 들어오는 요청은 일반적으로 다음 흐름을 거칩니다.

```text
Client
  ↓
External Load Balancer 또는 HAProxy
  ↓
Ingress Controller
  ↓
Ingress Resource
  ↓
Service
  ↓
EndpointSlice
  ↓
Pod IP
  ↓
Container Port
  ↓
Application Process
```

Pod 내부에서 다른 서비스로 나가는 요청은 다음 흐름을 거칩니다.

```text
Application Process
  ↓
Pod Network Namespace
  ↓
Pod eth0
  ↓
CNI
  ↓
Service DNS 또는 ClusterIP
  ↓
kube-proxy / eBPF Service 처리
  ↓
EndpointSlice
  ↓
Target Pod IP
  ↓
Target Container Port
```

따라서 문제를 볼 때는 다음 질문을 순서대로 던져야 합니다.

```text
1. DNS 이름은 IP로 해석되는가?
2. Ingress까지 요청이 도달하는가?
3. Ingress rule이 올바른 Service를 가리키는가?
4. Service selector가 Pod label과 맞는가?
5. EndpointSlice에 실제 backend Pod IP가 들어 있는가?
6. Pod가 Ready 상태인가?
7. targetPort가 실제 애플리케이션 listen port와 맞는가?
8. Pod 내부 애플리케이션은 정상 응답하는가?
9. NetworkPolicy가 막고 있지 않은가?
10. CNI, kube-proxy, CoreDNS, Ingress Controller 쪽 문제인가?
```

----

## 3. Kubernetes 리소스와 Linux 네트워크의 경계

Kubernetes 리소스만 보면 문제가 다 보일 것 같지만, 실제 패킷은 Linux 네트워크 기능을 통해 이동합니다.

```text
Kubernetes Resource Layer
  Ingress
  Service
  EndpointSlice
  Pod

Linux Network Layer
  Network Namespace
  veth pair
  routing table
  iptables / nftables
  eBPF
  NIC
```

예를 들어 Service는 Kubernetes API 리소스입니다. 하지만 Service ClusterIP로 들어온 패킷을 실제 Pod IP로 바꾸는 작업은 구현 방식에 따라 kube-proxy의 iptables/IPVS 또는 Cilium 같은 CNI의 eBPF 로직이 처리합니다.

```text
Service Resource
  ↓
kube-proxy iptables/IPVS 또는 Cilium eBPF
  ↓
Endpoint Pod IP
```

이 문서에서는 Kubernetes 리소스 관점의 확인을 중심으로 설명합니다. Linux 명령어 상세 해석은 앞선 Linux 네트워크 문서와 네트워크 트러블슈팅 도구 문서에서 다루는 것으로 분리합니다.

----

## 4. 전체 점검 흐름도

Kubernetes 네트워크 장애가 발생하면 다음 순서로 확인하는 것이 좋습니다.

```text
[외부 접속 실패]
  ↓
DNS 확인
  ↓
Ingress 주소 / Host / Path 확인
  ↓
Ingress Controller 로그 확인
  ↓
Service port / targetPort / selector 확인
  ↓
EndpointSlice backend 존재 여부 확인
  ↓
Pod Ready / Restart / Probe 상태 확인
  ↓
Pod 내부 애플리케이션 listen port 확인
  ↓
Pod 내부에서 curl/dig/ss 테스트
  ↓
NetworkPolicy / CNI / kube-proxy 확인
  ↓
필요 시 tcpdump / observability / RCA로 확장
```

내부 서비스 통신 장애라면 Ingress 단계를 건너뛰고 DNS, Service, EndpointSlice, Pod부터 확인합니다.

```text
[Pod → Service 통신 실패]
  ↓
Service DNS 확인
  ↓
ClusterIP 확인
  ↓
EndpointSlice 확인
  ↓
Target Pod Ready 확인
  ↓
NetworkPolicy 확인
  ↓
CNI / kube-proxy 확인
```

----

## 5. kubectl get 기본 사용법

`kubectl get`은 리소스의 현재 상태를 빠르게 확인하는 명령어입니다.

```bash
kubectl get pod -A
kubectl get svc -A
kubectl get ingress -A
kubectl get endpoints -A
kubectl get endpointslice -A
```

네트워크 장애 분석에서는 `-o wide`가 중요합니다.

```bash
kubectl get pod -n app -o wide
```

확인할 값:

| 항목 | 의미 |
|---|---|
| STATUS | Pod 상태 |
| READY | 컨테이너 Ready 여부 |
| RESTARTS | 재시작 횟수 |
| IP | Pod IP |
| NODE | 어느 노드에 배치되었는지 |

`-o yaml`은 리소스의 실제 설정을 확인할 때 사용합니다.

```bash
kubectl get svc -n app backend -o yaml
kubectl get ingress -n app app-ingress -o yaml
kubectl get pod -n app backend-xxx -o yaml
```

Label selector는 특정 앱 리소스만 볼 때 유용합니다.

```bash
kubectl get pod -n app -l app=backend -o wide
kubectl get endpointslice -n app -l kubernetes.io/service-name=backend
```

상태 변화를 실시간으로 볼 때는 `-w`를 사용합니다.

```bash
kubectl get pod -n app -w
kubectl get endpointslice -n app -w
```

----

## 6. kubectl describe로 보는 것

`kubectl describe`는 리소스의 상세 상태와 Event를 확인하는 명령어입니다.

```bash
kubectl describe pod -n app backend-xxx
kubectl describe svc -n app backend
kubectl describe ingress -n app app-ingress
```

Pod에서 봐야 할 핵심 항목:

| 항목 | 확인 내용 |
|---|---|
| Node | 어느 노드에 배치되었는가 |
| IP | Pod IP가 할당되었는가 |
| Containers | 컨테이너 이미지, 포트, 상태 |
| State | Running, Waiting, Terminated |
| Last State | 이전 종료 원인 |
| Ready | Ready 상태인가 |
| Restart Count | 반복 재시작 여부 |
| Liveness Probe | 재시작 유발 여부 |
| Readiness Probe | Service endpoint 포함 여부 |
| Environment | 환경변수 오류 여부 |
| Mounts | Secret/ConfigMap/PVC 마운트 여부 |
| Events | 스케줄링, 이미지, Probe, Mount 실패 |

특히 네트워크 장애에서는 `Readiness Probe`가 중요합니다. Pod가 Running이어도 Ready가 아니면 Service Endpoint에 포함되지 않을 수 있습니다.

```text
Pod Running
  ↓
Readiness Probe 실패
  ↓
Ready=False
  ↓
EndpointSlice에서 제외
  ↓
Service로 요청해도 backend 없음
  ↓
Ingress 503 발생 가능
```

----

## 7. kubectl logs

`kubectl logs`는 컨테이너 로그를 확인하는 명령어입니다.

```bash
kubectl logs -n app backend-xxx
kubectl logs -n app backend-xxx -c backend
kubectl logs -n app deploy/backend
kubectl logs -n app deploy/backend --tail=100
kubectl logs -n app deploy/backend -f
kubectl logs -n app backend-xxx --previous
```

`--previous`는 컨테이너가 재시작된 경우 중요합니다.

```text
현재 컨테이너 로그에는 원인이 없음
  ↓
이전 컨테이너가 죽기 직전 로그 필요
  ↓
kubectl logs --previous
```

Deployment 기준으로 로그를 볼 수도 있습니다.

```bash
kubectl logs -n app deploy/backend
```

여러 Pod의 로그를 같이 보려면 Label selector를 사용합니다.

```bash
kubectl logs -n app -l app=backend --tail=100
```

네트워크 장애에서 로그로 봐야 할 것:

- 애플리케이션이 실제로 요청을 받았는가
- 요청 경로가 맞는가
- upstream 연결 실패가 있는가
- DB/Redis 외부 의존성 연결 실패가 있는가
- TLS 인증서 오류가 있는가
- 인증/인가 실패가 있는가
- Health Check 요청이 들어오는가

주의할 점은 로그에 요청이 아예 없다면 애플리케이션까지 요청이 도달하지 않았을 가능성이 높다는 것입니다.

```text
curl 실패
  ↓
Application log에 요청 없음
  ↓
Ingress / Service / Endpoint / NetworkPolicy / CNI 쪽 확인
```

----

## 8. Events 확인

Event는 Kubernetes가 리소스에 대해 기록한 상태 변화입니다.

```bash
kubectl get events -n app
kubectl get events -n app --sort-by=.lastTimestamp
kubectl get events -A --sort-by=.lastTimestamp
```

자주 보는 Event:

| Event | 의미 |
|---|---|
| FailedScheduling | Pod를 배치할 노드가 없음 |
| FailedMount | Volume, Secret, ConfigMap 마운트 실패 |
| FailedCreatePodSandBox | CNI 또는 Pod sandbox 생성 실패 |
| Pulling | 이미지 Pull 중 |
| Failed | 이미지 Pull, 실행, Mount 등 실패 |
| BackOff | 반복 실패 후 재시도 지연 |
| Unhealthy | Probe 실패 |
| Killing | 컨테이너 종료 |

네트워크 장애처럼 보이지만 실제 원인이 CNI Pod sandbox 생성 실패일 수도 있습니다.

```text
Service 접속 실패
  ↓
Pod가 Running 아님
  ↓
Event 확인
  ↓
FailedCreatePodSandBox
  ↓
CNI 문제 가능성
```

----

## 9. Ingress 확인

Ingress는 HTTP/HTTPS 요청을 Service로 라우팅하는 Kubernetes 리소스입니다.

```bash
kubectl get ingress -A
kubectl describe ingress -n app app-ingress
kubectl get ingressclass
```

확인할 항목:

| 항목 | 확인 내용 |
|---|---|
| HOSTS | 요청 Host와 일치하는가 |
| ADDRESS | 외부 주소가 할당되었는가 |
| ingressClassName | 실제 Controller와 일치하는가 |
| Rules | Host / Path가 맞는가 |
| Backend Service | 연결된 Service가 존재하는가 |
| TLS Secret | Secret이 존재하고 인증서가 유효한가 |
| Annotations | Controller별 설정이 맞는가 |

Ingress 문제는 보통 다음 형태로 나타납니다.

| 증상 | 가능성 |
|---|---|
| 404 | Host/Path rule 불일치, default backend 응답 |
| 502 | Ingress Controller가 backend 연결 실패 |
| 503 | backend endpoint 없음 또는 Service backend unavailable |
| TLS 오류 | Secret, 인증서, SNI, Ingress TLS 설정 문제 |

Ingress는 리소스만 존재한다고 동작하지 않습니다. 반드시 Ingress Controller가 있어야 합니다.

```bash
kubectl get pod -A | grep -i ingress
kubectl get svc -A | grep -i ingress
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

컨트롤러가 NGINX, HAProxy, Traefik 중 무엇인지에 따라 annotation과 로그 형식이 다릅니다.

----

## 10. Ingress 404 분석

Ingress 404는 보통 요청이 Ingress Controller까지 도달했지만 적절한 rule을 찾지 못했을 때 발생합니다.

확인 순서:

```bash
kubectl get ingress -A
kubectl describe ingress -n app app-ingress
curl -v http://<ingress-address>/ -H 'Host: app.example.com'
```

확인할 것:

```text
1. curl의 Host Header가 Ingress host와 같은가?
2. Path가 rule과 일치하는가?
3. pathType이 Prefix인지 Exact인지 확인했는가?
4. ingressClassName이 Controller와 맞는가?
5. Controller가 해당 Ingress를 실제로 반영했는가?
```

예시:

```yaml
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

테스트:

```bash
curl -v http://<ingress-ip>/api/health -H 'Host: app.example.com'
```

`Host`를 빼고 IP로만 호출하면 Ingress rule이 맞지 않아 404가 날 수 있습니다.

----

## 11. Ingress 503 분석

Ingress 503은 Service 뒤에 정상 backend가 없을 때 자주 발생합니다.

확인 순서:

```bash
kubectl get svc -n app backend
kubectl describe svc -n app backend
kubectl get endpoints -n app backend
kubectl get endpointslice -n app -l kubernetes.io/service-name=backend
kubectl get pod -n app -l app=backend -o wide
```

대표 원인:

| 원인 | 설명 |
|---|---|
| Service selector 불일치 | Service가 Pod를 선택하지 못함 |
| Pod Ready=False | EndpointSlice에서 제외됨 |
| Pod 없음 | backend 자체가 없음 |
| Readiness Probe 실패 | Running이어도 Ready가 아님 |
| targetPort 오류 | Service가 잘못된 포트로 전달 |

Service selector와 Pod label을 비교합니다.

```bash
kubectl get svc -n app backend -o yaml
kubectl get pod -n app --show-labels
```

Service 예시:

```yaml
selector:
  app: backend
```

Pod label 예시:

```yaml
labels:
  app: backend
```

둘이 다르면 Endpoint가 비게 됩니다.

----

## 12. Ingress 502 분석

Ingress 502는 Ingress Controller가 backend에 연결하려 했지만 실패했을 때 자주 발생합니다.

가능한 원인:

- Service targetPort가 실제 앱 포트와 다름
- 앱이 `127.0.0.1`에만 listen하고 있음
- 앱 프로세스가 죽었음
- NetworkPolicy가 Ingress Controller → Pod 트래픽을 차단함
- 백엔드가 HTTP가 아닌데 HTTP로 호출함
- TLS backend인데 HTTP로 호출함

확인 명령어:

```bash
kubectl describe svc -n app backend
kubectl get endpointslice -n app -l kubernetes.io/service-name=backend -o wide
kubectl get pod -n app -o wide
kubectl logs -n app deploy/backend
```

Ingress Controller Pod 안에서 backend Pod IP로 직접 호출할 수 있으면 원인 범위를 좁힐 수 있습니다.

```bash
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- curl -v http://<pod-ip>:<target-port>/health
```

만약 여기서 실패하면 Ingress Controller와 Pod 사이 문제입니다.

----

## 13. Service 확인

Service는 안정적인 가상 IP 또는 DNS 이름을 제공하고, 뒤의 Pod로 트래픽을 전달하는 리소스입니다.

```bash
kubectl get svc -A
kubectl describe svc -n app backend
kubectl get svc -n app backend -o yaml
```

확인할 항목:

| 항목 | 설명 |
|---|---|
| type | ClusterIP, NodePort, LoadBalancer, ExternalName |
| clusterIP | Service 가상 IP |
| port | Service가 노출하는 포트 |
| targetPort | Pod 컨테이너가 실제 listen하는 포트 |
| selector | 어떤 Pod를 backend로 선택할지 |
| sessionAffinity | 클라이언트 세션 고정 여부 |

`port`와 `targetPort`를 구분해야 합니다.

```yaml
ports:
  - port: 80
    targetPort: 8080
```

의미:

```text
Client → Service:80 → Pod:8080
```

`targetPort`가 틀리면 Service는 존재해도 애플리케이션으로 연결되지 않습니다.

----

## 14. Service type별 확인 포인트

| Type | 사용 목적 | 확인 포인트 |
|---|---|---|
| ClusterIP | 클러스터 내부 통신 | ClusterIP, selector, endpoint |
| NodePort | 노드 포트로 외부 노출 | nodeIP:nodePort, 방화벽, kube-proxy |
| LoadBalancer | 클라우드/외부 LB 연동 | EXTERNAL-IP, LB 상태 |
| ExternalName | DNS CNAME 매핑 | DNS 해석, 포트 직접 지정 필요 |
| Headless | ClusterIP 없이 Pod DNS 사용 | `clusterIP: None`, StatefulSet, Endpoint |

Headless Service 예시:

```yaml
spec:
  clusterIP: None
```

Headless Service는 로드밸런싱용 ClusterIP를 만들지 않고 Pod endpoint DNS를 직접 제공합니다. StatefulSet에서 자주 사용됩니다.

ExternalName은 selector나 endpoint를 만들지 않고 DNS CNAME처럼 동작합니다.

```yaml
spec:
  type: ExternalName
  externalName: db.example.com
```

ExternalName은 HTTP routing이나 port mapping을 자동으로 해주는 기능이 아닙니다. 단순 DNS 이름 매핑에 가깝습니다.

----

## 15. Endpoint와 EndpointSlice

Service가 실제로 트래픽을 보낼 backend 목록이 Endpoint 또는 EndpointSlice입니다.

```bash
kubectl get endpoints -n app backend
kubectl get endpointslice -n app -l kubernetes.io/service-name=backend
```

Endpoint가 비어 있으면 Service 뒤에 보낼 대상이 없다는 뜻입니다.

```text
Service exists
  ↓
Endpoint empty
  ↓
Service selector가 Pod를 못 찾거나 Pod가 Ready가 아님
```

EndpointSlice는 Endpoint를 확장성 있게 관리하기 위해 사용됩니다. 최신 Kubernetes에서는 EndpointSlice를 기준으로 backend를 확인하는 것이 더 정확합니다.

EndpointSlice에서 볼 것:

| 항목 | 의미 |
|---|---|
| addresses | backend Pod IP |
| conditions.ready | Ready 여부 |
| ports | backend port |
| nodeName | Pod가 있는 노드 |
| targetRef | 연결된 Pod |

```bash
kubectl get endpointslice -n app -l kubernetes.io/service-name=backend -o yaml
```

----

## 16. Endpoint가 비어 있을 때 확인 순서

```text
Endpoint Empty
  ↓
Service selector 확인
  ↓
Pod label 확인
  ↓
Pod Ready 확인
  ↓
Readiness Probe 확인
  ↓
Pod가 같은 namespace에 있는지 확인
```

명령어:

```bash
kubectl get svc -n app backend -o yaml
kubectl get pod -n app --show-labels
kubectl get pod -n app -l app=backend
kubectl describe pod -n app <pod-name>
```

자주 발생하는 실수:

```yaml
# Service selector
selector:
  app: backned

# Pod label
labels:
  app: backend
```

오타 하나만 있어도 Endpoint가 비게 됩니다.

----

## 17. Pod 상태 확인

```bash
kubectl get pod -n app -o wide
kubectl describe pod -n app <pod-name>
```

Pod 상태별 의미:

| 상태 | 의미 | 네트워크 영향 |
|---|---|---|
| Pending | 스케줄링 또는 이미지/볼륨 준비 전 | 서비스 불가 |
| ContainerCreating | 컨테이너 생성 중 | CNI/Volume 문제 가능 |
| Running | 컨테이너 실행 중 | Ready 여부 별도 확인 필요 |
| CrashLoopBackOff | 반복 종료 | 앱 로그 확인 필요 |
| ImagePullBackOff | 이미지 Pull 실패 | 앱 실행 전 단계 |
| Error | 컨테이너 종료 오류 | 로그/이벤트 확인 |
| Completed | Job 정상 완료 | 서비스 대상 아님 |

Running이라고 항상 Service 대상이 되는 것은 아닙니다. Ready가 중요합니다.

```bash
kubectl get pod -n app
```

예시:

```text
NAME        READY   STATUS    RESTARTS
backend-1   0/1     Running   0
```

이 경우 Pod는 실행 중이지만 Ready가 아니므로 Endpoint에 포함되지 않을 수 있습니다.

----

## 18. Container Port와 Application Listen 확인

Service의 `targetPort`가 Pod의 containerPort와 맞아 보이더라도, 실제 애플리케이션이 해당 포트에서 listen하지 않으면 연결은 실패합니다.

Pod 내부에서 확인합니다.

```bash
kubectl exec -n app -it <pod-name> -- ss -lntp
kubectl exec -n app -it <pod-name> -- curl -v http://127.0.0.1:8080/health
```

주의할 점:

```text
앱이 127.0.0.1:8080에만 listen
  ↓
Pod 내부 localhost에서는 접속 가능
  ↓
다른 Pod 또는 Service에서는 접속 불가
```

애플리케이션은 보통 컨테이너 안에서 `0.0.0.0`으로 listen해야 Service를 통해 접근 가능합니다.

```text
Good: 0.0.0.0:8080
Bad : 127.0.0.1:8080
```

----

## 19. kubectl exec로 Pod 내부에서 확인하기

`kubectl exec`는 Pod 내부에서 명령어를 실행합니다.

```bash
kubectl exec -n app -it <pod-name> -- sh
kubectl exec -n app <pod-name> -- curl -v http://backend:8080/health
kubectl exec -n app <pod-name> -- cat /etc/resolv.conf
kubectl exec -n app <pod-name> -- nslookup kubernetes.default
```

Pod 내부에서 확인할 것:

| 목적 | 명령어 |
|---|---|
| DNS 확인 | `nslookup`, `dig` |
| HTTP 확인 | `curl -v` |
| TCP 포트 확인 | `nc -vz` |
| Listening port 확인 | `ss -lntp` |
| route 확인 | `ip route` |
| interface 확인 | `ip addr`, `ip link` |
| resolv.conf 확인 | `cat /etc/resolv.conf` |

일반 앱 컨테이너에는 디버깅 도구가 없을 수 있습니다. 이럴 때는 netshoot이나 ephemeral container를 사용합니다.

----

## 20. netshoot 디버깅 Pod

`nicolaka/netshoot`은 네트워크 디버깅 도구가 많이 포함된 이미지입니다.

```bash
kubectl run netshoot -n app --rm -it --image=nicolaka/netshoot -- bash
```

자주 쓰는 명령어:

```bash
curl -v http://backend.app.svc.cluster.local/health
dig backend.app.svc.cluster.local
nslookup kubernetes.default
nc -vz backend.app.svc.cluster.local 80
ss -antp
ip addr
ip route
tracepath backend.app.svc.cluster.local
```

netshoot은 다음 상황에서 유용합니다.

- 앱 컨테이너에 `curl`, `dig`, `nc`가 없음
- Service DNS 확인 필요
- NetworkPolicy 영향 확인 필요
- 특정 namespace 내부에서 통신 테스트 필요

----

## 21. kubectl debug와 Ephemeral Container

운영 Pod 이미지를 수정하지 않고 디버깅 컨테이너를 붙이고 싶을 때 `kubectl debug`를 사용할 수 있습니다.

```bash
kubectl debug -n app -it pod/<pod-name> --image=nicolaka/netshoot --target=<container-name> -- bash
```

장점:

- 기존 Pod에 임시 디버깅 컨테이너를 추가
- 운영 이미지를 변경하지 않아도 됨
- 대상 컨테이너의 네임스페이스를 공유해 확인 가능

주의:

- 클러스터 버전과 권한에 따라 제한될 수 있음
- RBAC 권한이 필요할 수 있음
- 디버깅 후 불필요한 컨테이너가 남지 않도록 관리 필요

----

## 22. DNS와 CoreDNS 확인

Kubernetes 내부 Service Discovery는 CoreDNS를 통해 동작합니다.

Service DNS 형식:

```text
<service>.<namespace>.svc.cluster.local
```

예시:

```text
backend.app.svc.cluster.local
kubernetes.default.svc.cluster.local
```

Pod 내부에서 확인:

```bash
kubectl exec -n app <pod-name> -- cat /etc/resolv.conf
kubectl exec -n app <pod-name> -- nslookup backend.app.svc.cluster.local
kubectl exec -n app <pod-name> -- nslookup kubernetes.default
```

CoreDNS 상태 확인:

```bash
kubectl -n kube-system get pod -l k8s-app=kube-dns
kubectl -n kube-system get svc kube-dns
kubectl -n kube-system logs deploy/coredns
```

DNS 장애 증상:

| 증상 | 가능성 |
|---|---|
| NXDOMAIN | Service 이름/namespace 오류 |
| timeout | CoreDNS 장애, NetworkPolicy, CNI 문제 |
| 일부 Pod만 실패 | 해당 Node 또는 Pod DNS 설정 문제 |
| 외부 DNS만 실패 | upstream DNS 설정 문제 |

`/etc/resolv.conf`에서 확인할 값:

```text
nameserver <cluster-dns-ip>
search <namespace>.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

`ndots:5` 때문에 짧은 이름 조회 시 여러 search domain을 시도할 수 있습니다.

----

## 23. NetworkPolicy 확인

NetworkPolicy는 Pod 간 ingress/egress 트래픽을 제어합니다.

```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy -n app
```

NetworkPolicy는 CNI가 지원해야 실제로 적용됩니다. Calico, Cilium 등은 NetworkPolicy를 지원하지만, CNI 구성에 따라 동작이 달라질 수 있습니다.

확인할 것:

| 항목 | 의미 |
|---|---|
| podSelector | 어떤 Pod에 정책을 적용하는가 |
| policyTypes | Ingress, Egress |
| ingress.from | 어디에서 들어오는 트래픽을 허용하는가 |
| egress.to | 어디로 나가는 트래픽을 허용하는가 |
| ports | 어떤 포트를 허용하는가 |

장애 예시:

```text
Ingress Controller → backend Pod
  ↓
NetworkPolicy에서 허용 안 함
  ↓
Ingress 502 또는 timeout
```

테스트:

```bash
kubectl run netshoot -n ingress-nginx --rm -it --image=nicolaka/netshoot -- bash
curl -v http://<backend-pod-ip>:8080/health
```

Ingress Controller namespace에서 backend Pod로 접근 가능한지 확인하는 것이 중요합니다.

----

## 24. kube-proxy와 CNI 문제 의심 기준

Service, EndpointSlice, Pod가 모두 정상인데도 Service ClusterIP로 접근이 안 되면 kube-proxy 또는 CNI 문제를 의심할 수 있습니다.

확인 기준:

```text
Service 존재 O
EndpointSlice 존재 O
Pod 직접 접근 O
Service ClusterIP 접근 X
```

이 경우 확인할 것:

```bash
kubectl -n kube-system get pod -l k8s-app=kube-proxy
kubectl -n kube-system logs ds/kube-proxy
```

Cilium 사용 환경이라면 kube-proxy가 없을 수 있습니다. 이 경우 Cilium 상태를 봅니다.

```bash
kubectl -n kube-system get pod -l k8s-app=cilium
kubectl -n kube-system logs ds/cilium
cilium status
```

확실하지 않음: 클러스터별 CNI 구성에 따라 명령어와 라벨은 다를 수 있습니다. 실제 환경에서는 설치된 CNI와 namespace, label을 먼저 확인해야 합니다.

----

## 25. kubectl port-forward

`kubectl port-forward`는 로컬 포트를 Pod 또는 Service로 연결해 임시로 접근하는 기능입니다.

```bash
kubectl port-forward -n app svc/backend 8080:80
kubectl port-forward -n app pod/backend-xxx 8080:8080
```

흐름은 다음과 같습니다.

```text
Localhost
  ↓
kubectl
  ↓
Kubernetes API Server
  ↓
Pod 또는 Service
```

사용 목적:

- Ingress를 우회하고 Service 또는 Pod 자체 확인
- 외부 Load Balancer 문제와 앱 문제 분리
- 로컬에서 임시 테스트
- 내부 서비스 접근

주의:

- 운영 트래픽 처리용이 아님
- kubectl 세션이 끊기면 종료됨
- RBAC 권한 필요
- NetworkPolicy와 동일한 경로가 아닐 수 있으므로 최종 통신 검증과 혼동하면 안 됨

장애 분리 예시:

```text
Ingress 접속 실패
  ↓
port-forward svc/backend 성공
  ↓
앱과 Service는 정상 가능성 높음
  ↓
Ingress rule, Controller, TLS, 외부 LB 확인
```

----

## 26. Rollout 상태 확인

배포 직후 장애라면 Deployment rollout 상태를 확인해야 합니다.

```bash
kubectl rollout status deploy/backend -n app
kubectl rollout history deploy/backend -n app
kubectl describe deploy/backend -n app
```

잘못된 이미지, 잘못된 환경변수, Probe 실패로 새 ReplicaSet이 Ready가 되지 않으면 트래픽 장애가 발생할 수 있습니다.

롤백:

```bash
kubectl rollout undo deploy/backend -n app
```

주의: 롤백은 실제 운영 변경이므로 원인과 영향 범위를 확인한 뒤 수행해야 합니다.

----

## 27. kubectl wait

조건이 만족될 때까지 기다릴 때 사용합니다.

```bash
kubectl wait --for=condition=available deploy/backend -n app --timeout=120s
kubectl wait --for=condition=ready pod -l app=backend -n app --timeout=120s
```

CI/CD나 배포 후 검증에서 유용합니다.

```text
배포
  ↓
rollout status
  ↓
wait ready
  ↓
Service curl
  ↓
Ingress curl
```

----

## 28. kubectl explain

리소스 필드 의미가 헷갈릴 때 사용합니다.

```bash
kubectl explain service.spec.ports
kubectl explain service.spec.selector
kubectl explain ingress.spec.rules
kubectl explain networkpolicy.spec
```

YAML을 외워서 쓰는 것보다, 필드 의미를 확인하면서 보는 습관이 좋습니다.

----

## 29. kubectl top

리소스 부족이 네트워크 장애처럼 보일 수 있습니다.

```bash
kubectl top pod -n app
kubectl top node
```

예를 들어 CPU throttling, OOMKilled, Node 자원 부족으로 애플리케이션 응답이 느려지고 timeout이 발생할 수 있습니다.

`kubectl top`은 metrics-server가 설치되어 있어야 동작합니다.

----

## 30. kubectl auth can-i

권한 문제도 장애처럼 보일 수 있습니다.

```bash
kubectl auth can-i get pods -n app
kubectl auth can-i create pods/exec -n app
kubectl auth can-i get secrets -n app
```

예를 들어 디버깅을 위해 `kubectl exec`를 실행하려는데 권한이 없으면 네트워크 확인 자체가 막힐 수 있습니다.

----

## 31. TLS Secret과 인증서 확인

Ingress TLS 문제는 Secret과 인증서 상태를 봐야 합니다.

```bash
kubectl get secret -n app
kubectl describe secret -n app app-tls
kubectl get ingress -n app app-ingress -o yaml
```

인증서 내용을 확인해야 할 때:

```bash
kubectl get secret -n app app-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -subject -issuer -dates
```

cert-manager를 사용한다면 다음도 확인합니다.

```bash
kubectl get certificate -A
kubectl describe certificate -n app app-tls
kubectl get certificaterequest -n app
kubectl get challenge -n app
kubectl get order -n app
```

증상:

| 증상 | 가능성 |
|---|---|
| 브라우저 인증서 경고 | 만료, CN/SAN 불일치, 체인 문제 |
| Ingress TLS 적용 안 됨 | Secret 이름 오류, namespace 오류 |
| cert-manager 발급 실패 | DNS/HTTP challenge 실패, issuer 오류 |

----

## 32. Kubernetes API Server 6443 접근 문제

Kubernetes API Server는 일반적인 Ingress HTTP 서비스와 다르게 보는 것이 좋습니다.

```text
kubectl
  ↓ TLS
Load Balancer / HAProxy
  ↓ TLS passthrough
kube-apiserver:6443
```

확인:

```bash
kubectl cluster-info
kubectl get --raw=/readyz
kubectl get --raw=/livez
```

외부에서 TLS 확인:

```bash
openssl s_client -connect <api-server-address>:6443 -servername <api-server-host>
```

API Server 앞단 LB는 보통 TCP mode로 구성되는 경우가 많습니다. HTTP Ingress 404/503 방식으로 접근하면 안 됩니다.

----

## 33. Harbor 장애 분석 예시

Harbor는 웹 UI와 API를 제공하므로 일반적으로 Ingress 또는 L7 Reverse Proxy를 거칩니다.

확인 순서:

```bash
curl -vk https://harbor.example.com/api/v2.0/health
kubectl get ingress -A | grep harbor
kubectl describe ingress -n harbor harbor
kubectl get svc -n harbor
kubectl get endpointslice -n harbor
kubectl get pod -n harbor -o wide
kubectl logs -n harbor deploy/harbor-core --tail=100
```

자주 보는 원인:

- Ingress Host 불일치
- TLS Secret 문제
- `X-Forwarded-Proto` 누락으로 redirect/login 문제
- harbor-core, registry, portal Service endpoint 없음
- DB 또는 Redis 연결 실패

주의: Harbor 자체 내부 컴포넌트 구조는 이 문서의 주제가 아닙니다. 여기서는 Ingress → Service → EndpointSlice → Pod 흐름으로 장애를 좁히는 데 집중합니다.

----

## 34. ArgoCD 장애 분석 예시

ArgoCD는 Web UI, API, gRPC 성격의 통신을 함께 고려해야 합니다.

확인 순서:

```bash
curl -vk https://argocd.example.com
kubectl get ingress -n argocd
kubectl describe ingress -n argocd argocd-server
kubectl get svc -n argocd
kubectl get endpointslice -n argocd
kubectl logs -n argocd deploy/argocd-server --tail=100
```

자주 보는 원인:

- TLS 종료 위치 문제
- gRPC 또는 HTTP/2 처리 문제
- Ingress annotation 누락
- `X-Forwarded-Proto` 누락
- argocd-server Service port와 targetPort 불일치
- Dex 또는 OIDC redirect URL 문제

CLI 접속 실패와 UI 접속 실패는 원인이 다를 수 있으므로 분리해서 봅니다.

----

## 35. Ceph Dashboard 장애 분석 예시

Ceph Dashboard는 웹 UI입니다.

확인 순서:

```bash
curl -vk https://ceph.example.com
kubectl get ingress -A | grep ceph
kubectl describe ingress -n ceph ceph-dashboard
kubectl get svc -n ceph
kubectl get endpointslice -n ceph
kubectl logs -n ceph <dashboard-pod> --tail=100
```

자주 보는 원인:

- Dashboard backend가 HTTPS인데 Ingress가 HTTP로 호출
- TLS verify 문제
- Service targetPort 불일치
- active mgr가 바뀌었는데 backend endpoint가 맞지 않음
- 인증서 문제

Ceph Dashboard 자체 상태는 Ceph 명령어와 mgr 상태 확인이 필요할 수 있습니다. 이 문서에서는 Kubernetes 경로 확인까지만 다룹니다.

----

## 36. 증상별 체크리스트

### 36-1) 404

```text
Ingress Controller까지 도달했지만 rule 매칭 실패 가능성
```

확인:

```bash
kubectl get ingress -A
kubectl describe ingress -n <ns> <ingress>
curl -v http://<ingress-ip>/<path> -H 'Host: <host>'
```

볼 것:

- Host Header
- Path
- pathType
- ingressClassName
- default backend 응답 여부

### 36-2) 503

```text
Service backend 없음 가능성
```

확인:

```bash
kubectl get svc -n <ns> <svc>
kubectl get endpoints -n <ns> <svc>
kubectl get endpointslice -n <ns> -l kubernetes.io/service-name=<svc>
kubectl get pod -n <ns> --show-labels
```

볼 것:

- Endpoint empty
- selector/label 불일치
- Pod Ready=False
- Readiness Probe 실패

### 36-3) 502

```text
Ingress Controller가 backend 연결 실패 가능성
```

확인:

```bash
kubectl describe svc -n <ns> <svc>
kubectl logs -n <ingress-ns> <ingress-controller-pod>
kubectl exec -n <ingress-ns> <ingress-controller-pod> -- curl -v http://<pod-ip>:<target-port>
```

볼 것:

- targetPort
- 앱 listen address
- NetworkPolicy
- HTTP/HTTPS mismatch

### 36-4) Timeout

```text
패킷이 어딘가에서 드롭되거나 응답이 지연됨
```

확인:

```bash
kubectl exec -n <ns> <pod> -- curl -v --connect-timeout 5 http://<service>
kubectl get networkpolicy -A
kubectl get pod -n kube-system
```

볼 것:

- NetworkPolicy
- CNI 상태
- Node 네트워크
- 백엔드 앱 응답 지연

### 36-5) Connection Refused

```text
대상 IP까지 도달했지만 포트에서 listen 중인 프로세스가 없음
```

확인:

```bash
kubectl exec -n <ns> <pod> -- ss -lntp
kubectl describe svc -n <ns> <svc>
```

볼 것:

- targetPort
- application listen port
- 127.0.0.1 bind 여부

### 36-6) NXDOMAIN

```text
DNS 이름 자체가 존재하지 않음
```

확인:

```bash
kubectl exec -n <ns> <pod> -- nslookup <svc>.<ns>.svc.cluster.local
kubectl get svc -n <ns>
```

볼 것:

- Service 이름
- namespace
- DNS suffix

### 36-7) TLS Error

```text
인증서, SNI, TLS 종료 위치 문제 가능성
```

확인:

```bash
openssl s_client -connect <host>:443 -servername <host>
kubectl get secret -n <ns>
kubectl describe ingress -n <ns> <ingress>
```

볼 것:

- Secret 이름
- 인증서 만료
- SAN
- TLS termination 위치

----

## 37. End-to-End 점검 예시

외부에서 `https://app.example.com/health`가 실패한다고 가정합니다.

```bash
# 1. 외부 HTTP/TLS 확인
curl -vk https://app.example.com/health

# 2. Ingress 확인
kubectl get ingress -A
kubectl describe ingress -n app app-ingress

# 3. Ingress Controller 로그
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=100

# 4. Service 확인
kubectl get svc -n app backend
kubectl describe svc -n app backend

# 5. EndpointSlice 확인
kubectl get endpointslice -n app -l kubernetes.io/service-name=backend -o wide

# 6. Pod 상태 확인
kubectl get pod -n app -l app=backend -o wide
kubectl describe pod -n app <backend-pod>

# 7. 애플리케이션 로그
kubectl logs -n app deploy/backend --tail=100
kubectl logs -n app deploy/backend --previous --tail=100

# 8. Pod 내부 직접 확인
kubectl exec -n app <backend-pod> -- ss -lntp
kubectl exec -n app <backend-pod> -- curl -v http://127.0.0.1:8080/health

# 9. 같은 namespace에서 Service DNS 확인
kubectl run netshoot -n app --rm -it --image=nicolaka/netshoot -- bash
curl -v http://backend.app.svc.cluster.local/health

# 10. NetworkPolicy 확인
kubectl get networkpolicy -A
```

이 순서로 보면 어느 구간에서 실패하는지 좁힐 수 있습니다.

----

## 38. 11번 문서와 12번 문서의 역할 차이

이 문서는 Kubernetes 리소스 경로를 따라 장애 지점을 찾는 데 집중합니다.

```text
Ingress
  ↓
Service
  ↓
EndpointSlice
  ↓
Pod
  ↓
Application
```

반면 12번 Observability/RCA 문서는 다음을 다룹니다.

```text
로그, 메트릭, 트레이스 수집
  ↓
타임라인 작성
  ↓
원인 후보 검증
  ↓
Root Cause Analysis
  ↓
재발 방지 대책
```

즉, 11번은 **어디서 끊겼는지 찾는 문서**이고, 12번은 **왜 장애가 났는지 증거를 모아 설명하는 문서**입니다.

----

## 39. 최종 요약

Kubernetes 네트워크 장애는 다음 흐름으로 보는 것이 가장 안전합니다.

```text
Client
  ↓
Ingress
  ↓
Service
  ↓
EndpointSlice
  ↓
Pod
  ↓
Container Port
  ↓
Application
```

핵심 판단 기준은 다음과 같습니다.

| 증상 | 먼저 볼 곳 |
|---|---|
| 404 | Ingress Host/Path/IngressClass |
| 503 | Service Endpoint/EndpointSlice |
| 502 | targetPort, backend 연결, NetworkPolicy |
| Timeout | NetworkPolicy, CNI, Node 네트워크 |
| Connection Refused | 앱 listen port, targetPort |
| NXDOMAIN | Service DNS, namespace, CoreDNS |
| TLS Error | Secret, 인증서, SNI, TLS 종료 위치 |

명령어를 외우는 것보다 중요한 것은 흐름입니다.

```text
리소스가 존재하는가?
  ↓
리소스끼리 연결되어 있는가?
  ↓
실제 endpoint가 있는가?
  ↓
애플리케이션이 listen하고 있는가?
  ↓
정책이나 CNI가 막고 있지 않은가?
```