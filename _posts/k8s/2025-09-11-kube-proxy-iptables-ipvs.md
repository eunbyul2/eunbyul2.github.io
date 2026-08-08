---
layout: post
title: "쿠버네티스 kube-proxy와 iptables/IPVS, DNAT, RR/LC까지: 트래픽 분산"
date: 2025-09-11 15:25:00 +0900
categories: [kubernetes, Network]
tags: [kube-proxy, iptables, ipvs, kubernetes, network]
published: true
---

kube-proxy의 동작 원리와 iptables, ipvs의 차이점 정리

- kube-proxy: 서비스 트래픽을 노드로 라우팅
- iptables: 패킷 필터링 및 NAT
- ipvs: 고성능 로드밸런싱

---

# 개요

- **kube-proxy**는 직접 패킷을 중계하지 않는다. 대신 **리눅스 커널**에 규칙을 심어 **Service IP → Pod IP**로 트래픽을 분산시킨다.
- 구현 방식은 두 가지: **iptables 모드**(DNAT 규칙 체인) vs **IPVS 모드**(커널 L4 로드밸런서).
- **DNAT**는 목적지 IP/Port를 Pod로 바꾸는 동작. NodePort/ClusterIP/LoadBalancer 모두 이 원리를 쓴다.
- **IPVS**는 RR(라운드로빈), LC(최소 연결) 등 **여러 스케줄러**를 제공하며, **대규모 서비스/엔드포인트**에서 성능 우위.
- **세션 어피니티(ClientIP)**, **externalTrafficPolicy(Local/Cluster)**, **헤어핀(Hairpin) NAT**, **토폴로지 친화 라우팅** 등 운영 옵션까지 알아야 실무에서 삽질이 줄어든다.

---

## 1) kube-proxy가 하는 일

- 각 노드에서 실행되는 데몬(DaemonSet).
- API 서버에서 **Service** 및 **EndpointSlice**(혹은 Endpoints) 변화를 워치하고, 그에 맞춰 **커널 규칙**을 갱신한다.
- 모드:
  - `iptables`: NAT/필터 테이블 체인을 생성해 **DNAT**로 분산.
  - `ipvs`: 커널 **IPVS 가상 서비스**를 만들고 **스케줄러(RR/LC 등)** 로 분산.

> 확인: `kubectl -n kube-system get cm kube-proxy -o yaml` → `mode: "iptables" | "ipvs"`

---

## 2) Service 모델과 데이터플레인 연결

- **Service 타입**
  - `ClusterIP`: 클러스터 내부 IP. **기본값**.
  - `NodePort`: 각 노드의 고정 포트 노출(30000~32767).
  - `LoadBalancer`: 클라우드 LB와 연계(내부는 결국 NodePort/ClusterIP로 연결).
  - `ExternalName`: DNS CNAME. 데이터 플레인 라우팅은 없음.
- **EndpointSlice/Endpoints**: Service가 바라보는 **Pod IP 목록**(+ Port). kube-proxy가 이것을 읽어 규칙을 생성.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-svc
spec:
  selector: { app: demo }
  ports:
    - name: http
      port: 80 # Service(ClusterIP) Port
      targetPort: 8080 # Pod Container Port
  sessionAffinity: None # 혹은 ClientIP
```

---

## 3) DNAT 정확히 이해하기

- **정의**: 패킷의 **목적지 주소(IP/Port)** 를 다른 주소로 바꾸는 NAT.
- **kube-proxy(iptables 모드)** 는 `nat` 테이블에 **KUBE-SERVICES/KUBE-SVC-_/KUBE-SEP-_ 체인**을 만들고 **DNAT**으로 Pod로 보낸다.
- 흐름(ClusterIP 예):
  1. 패킷 도착 → `PREROUTING` (nat)
  2. `KUBE-SERVICES` → `KUBE-SVC-<hash>`
  3. 엔드포인트 선택 규칙(확률/해시)을 거쳐 `KUBE-SEP-<hash>`로 점프
  4. `DNAT --to-destination 10.244.x.y:port` 적용 → Pod로 라우팅
- **NodePort**의 경우 `KUBE-NODEPORTS` 체인에서 ClusterIP 경로로 연결되며, `externalTrafficPolicy`에 따라 SNAT 보존 여부가 달라진다.

> 예시(개념적 예; 실제 규칙은 환경마다 상이):

```bash
# ClusterIP에 대한 매칭
-A PREROUTING -d 10.96.0.1/32 -p tcp --dport 80 -j KUBE-SERVICES
-A KUBE-SERVICES -m comment --comment "svc demo-svc" -j KUBE-SVC-XYZ
# 엔드포인트 선택(확률 기반 등)
-A KUBE-SVC-XYZ -m statistic --mode random --probability 0.333333 -j KUBE-SEP-A
-A KUBE-SVC-XYZ -m statistic --mode random --probability 0.500000 -j KUBE-SEP-B
-A KUBE-SVC-XYZ -j KUBE-SEP-C
# 실제 DNAT
-A KUBE-SEP-A -j DNAT --to-destination 10.244.1.5:8080
```

- **Conntrack**: 첫 패킷에서 DNAT가 결정되면 **연결 추적** 덕분에 후속 패킷은 동일한 백엔드로 간다(세션 유지).
- **Hairpin NAT**: 같은 노드의 Pod가 **자기 자신이 속한 Service** 를 통해 자기/동료 Pod을 때릴 때 필요. kubelet `--hairpin-mode`(일반적으로 `hairpin-veth`)와 브리지/iptables 설정이 맞아야 한다.

---

## 4) IPVS: 커널 L4 로드밸런서

- kube-proxy `ipvs` 모드는 커널 IPVS에 **가상 서비스(VIP:PORT)** 를 만들고, 다수의 **리얼 서버(Pod)** 를 매핑한다.
- **전달 방식**: kube-proxy는 일반적으로 **NAT(-m, MASQ)** 모드를 사용.
- **주요 스케줄러(일부)**:
  - `rr`(Round Robin): 순차 분배
  - `lc`(Least Connection): 활성 연결 수 최소인 서버 선택
  - `wrr`/`wlc`: 가중치 기반 RR/LC
  - `sh`/`dh`: 소스/목적지 해시(세션 스티키에 유용)
  - `sed`/`nq`: 대기 시간 추정/Queue 기반 (환경에 따라 가용성 제한)
- **세션 어피니티**(Service `sessionAffinity: ClientIP`) 는 IPVS **persistence**(유지시간)로 구현된다.

> 상태 확인:

```bash
ipvsadm -Ln           # 가상 서비스/백엔드 리스트
ipvsadm -ln --stats   # 분배 통계
lsmod | egrep 'ip_vs|nf_conntrack'
```

**왜 IPVS가 빠른가?**

- iptables는 **규칙 수가 많아질수록** 체인 탐색 비용이 증가(선형/부분 확률 평가).
- IPVS는 **해시 테이블** 기반으로 **O(1)에 가까운 룩업**, 커널 레벨 스케줄러로 대량 엔드포인트에 유리.

---

## 5) 라운드로빈(RR) vs 최소 연결(LC) 깊게

### RR (Round Robin)

- **규칙**: 요청 도착 순서대로 서버를 **순환 분배**.
- **장점**: 단순, 공평. 단기 연결/유사 부하에 적합.
- **주의**: HTTP/2, keep-alive, 길게 유지되는 스트림에서는 실제 부하가 균등해지지 않을 수 있음.

### LC (Least Connection)

- **규칙**: **현재 활성 연결 수**가 가장 적은 서버 선택.
- **장점**: 연결 시간이 들쑥날쑥하거나 일부 요청이 무거울 때 유리.
- **주의**: 초단기·무상태 트래픽에서는 RR 대비 이점이 작고, 상태 추적 비용이 있다.

> 선택 가이드:
>
> - 짧고 균일한 요청 ➜ **RR**
> - 연결 길이/부하 편차 큼, WebSocket/HTTP2 혼재 ➜ **LC**(또는 `wlc`)
> - 클라이언트 IP 별 세션 스티키 필요 ➜ **`sh`** 또는 **`sessionAffinity: ClientIP`**

---

## 6) NodePort, 소스 IP 보존, SNAT

- `externalTrafficPolicy: Cluster`(기본): 어떤 노드로 오든 **클러스터 임의 Pod** 로 라우팅. 필요 시 **SNAT** 되어 **원 소스 IP가 사라질 수 있음**.
- `externalTrafficPolicy: Local`: **도착한 노드의 로컬 Pod** 로만 라우팅. 로컬 Pod가 없으면 드롭/ICMP. **클라이언트 소스 IP 보존**(로그/보안 정책에 유리).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-nodeport
spec:
  type: NodePort
  selector: { app: demo }
  externalTrafficPolicy: Local # 소스 IP 보존
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 31080
```

---

## 7) 토폴로지-어웨어 라우팅(Topology Aware Hints)

- `service.kubernetes.io/topology-aware-hints: auto`  
  동일 노드/존/리전에 **가까운 엔드포인트** 를 우선 선택하도록 힌트를 제공(크로스 존 트래픽, 지연/비용 감소).

```yaml
metadata:
  annotations:
    service.kubernetes.io/topology-aware-hints: auto
```

---

## 8) 실무 설정: iptables ↔ ipvs 전환

1. **커널 모듈/툴 준비**
   ```bash
   sudo modprobe ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh
   sudo modprobe nf_conntrack
   sudo apt-get install -y ipvsadm ipset
   ```
2. **kube-proxy 설정 변경(예: kubeadm 기반)**
   ```bash
   kubectl -n kube-system get cm kube-proxy -o yaml > kp.yaml
   # kp.yaml 편집
   # mode: "ipvs"
   kubectl -n kube-system apply -f kp.yaml
   kubectl -n kube-system rollout restart ds kube-proxy
   ```
3. **확인**
   ```bash
   ipvsadm -Ln
   kubectl get endpointslice -A
   ```

> 주의: 운영 전환 시 **드레인/점진 롤아웃** 권장. 모듈/방화벽 정책(Cloud CNI/보안그룹)과의 충돌을 사전 점검.

---

## 9) 트러블슈팅 체크리스트

- 분배 확인
  ```bash
  ipvsadm -ln --stats        # IPVS 통계
  iptables-save | grep KUBE- # iptables 체인 확인
  kubectl get svc,ep,endpointslice -A -o wide
  ```
- 커넥션/세션
  ```bash
  ss -ntp | head
  sudo conntrack -L | head
  ```
- 소스 IP 보존
  - Service에 `externalTrafficPolicy: Local` 설정했는가?
  - 노드 로컬 Pod가 실제로 존재하는가?
- Hairpin 문제
  - kubelet `--hairpin-mode` 확인, 브리지/iptables 포워딩 설정(`net.ipv4.ip_forward=1`).
- 성능/쏠림
  - iptables 모드에서 엔드포인트 수가 많은가? ➜ IPVS 전환 고려
  - HTTP/2, 장수 연결 많음 ➜ LC/wlc 검토

---

## 10) 빠른 재현 레시피(테스트)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: demo, labels: { app: demo } }
spec:
  replicas: 3
  selector: { matchLabels: { app: demo } }
  template:
    metadata: { labels: { app: demo } }
    spec:
      containers:
        - name: web
          image: ghcr.io/nginxdemos/hello:plain-text
          ports: [{ containerPort: 80 }]
---
apiVersion: v1
kind: Service
metadata: { name: demo-svc }
spec:
  selector: { app: demo }
  ports: [{ port: 80, targetPort: 80 }]
```

```bash
# RR/LC 체감: 여러 번 호출하며 응답의 hostname 확인
for i in $(seq 1 12); do curl -sS demo-svc.default.svc.cluster.local | grep -E 'server address|server name'; done
```

---

## 11) 용어 요약(Glossary)

- **DNAT**: 목적지 주소 변경. Service IP → Pod IP.
- **SNAT/MASQUERADE**: 소스 주소 변경. 소스 IP 보존/감춤에 영향.
- **IPVS**: 커널 L4 LB. `ipvsadm`으로 조회/설정. 스케줄러: `rr`, `lc`, `wrr`, `wlc`, `sh`, `dh` 등.
- **RR**: 순차 분배. **LC**: 활성 연결 최소 서버 선택.
- **EndpointSlice**: Service 백엔드 목록(확장성 개선).
- **Hairpin NAT**: 같은 노드 Pod이 자기/동료 Pod을 **Service IP**로 접근 가능하게 하는 NAT.

---

## 12) 언제 무엇을 쓰나 (실무 선택 기준)

- **엔드포인트 수가 적고 단순**: iptables로 충분.
- **서비스/엔드포인트가 수백~수천**: IPVS 권장(해시 기반, 낮은 지연/CPU).
- **소스 IP가 꼭 필요**: `externalTrafficPolicy: Local` + 로컬 백엔드 확보.
- **세션 고정 필요**: `sessionAffinity: ClientIP` 또는 IPVS `sh`/persistence.
- **HTTP/2/스트림 다수**: LC/wlc가 더 안정적일 수 있음.

---
