---
layout: post
title: "CephX 인증: Entity, Keyring, Capabilities와 인증 흐름"
date: 2026-08-24 21:30:00 +0900
categories: [Ceph, Storage, Security]
tags: [Ceph, CephX, Authentication, Authorization, Keyring, Capabilities, MON, OSD, MDS, cephadm]
published: true
---

Ceph 20.2.4 업그레이드 이후 `AUTH_INSECURE_CLIENT_KEY_TYPE`, `AUTH_INSECURE_SERVICE_KEY_TYPE` 같은 Health Check를 마주치면서 가장 먼저 막혔던 부분은 `client.admin`, `client.cinder`, `mgr.*`, `mds.*` 같은 이름들이 정확히 무엇인지 이해하지 못했다는 점이었다.

처음에는 이것들을 단순한 "Ceph 계정" 정도로 생각했지만, 실제로 CephX를 이해하려면 **Entity, Secret Key, Keyring, Capability, Ticket, Session Key**가 어떤 관계를 갖는지 먼저 알아야 한다.

이 글에서는 CephX를 처음 접하는 입장에서 인증 구조를 처음부터 정리한다.

> 이 글에서 실제 Key 값은 보안상 모두 생략한다. `ceph auth get`, `ceph auth export`, keyring 파일에는 실제 Cluster 접근에 사용할 수 있는 Secret이 포함될 수 있으므로 블로그나 사내 공개 문서에 그대로 올리면 안 된다.

---

# 1. CephX는 무엇인가

CephX는 Ceph에서 사용하는 기본 인증(Authentication) 시스템이다.

Ceph Cluster에는 사람만 접근하는 것이 아니다. 다음과 같은 여러 주체가 Ceph와 통신한다.

- `ceph` CLI를 실행하는 관리자
- OpenStack Cinder
- Kubernetes CSI
- RGW
- OSD
- MDS
- MGR
- crash collector
- ceph-exporter
- 외부 CephFS/RBD client

Ceph는 이런 주체가 Cluster에 접근할 때 "누구인지"를 확인해야 한다.

```text
Ceph Client / Daemon
        |
        |  "나는 client.cinder다"
        v
      MON
        |
        |  CephX 인증
        v
인증 성공 / 실패
```

Ceph 공식 Architecture 문서에서는 CephX가 **shared secret 기반의 mutual authentication**을 수행한다고 설명한다. Client와 Monitor가 동일한 secret을 가지고 있으면서도 실제 secret 자체를 네트워크로 전송하지 않고 서로 해당 secret을 알고 있음을 증명한다.

공식 문서:

- Ceph Architecture - High Availability Authentication  
  https://docs.ceph.com/en/latest/architecture/#high-availability-authentication
- CephX Config Reference  
  https://docs.ceph.com/en/latest/rados/configuration/auth-config-ref/

중요한 점은 CephX가 **TLS 같은 전송 암호화 자체를 의미하지는 않는다**는 것이다.

Ceph 공식 문서도 CephX가 transport encryption 또는 encryption at rest를 제공하는 기능은 아니라고 명시한다. CephX의 핵심 역할은 인증과 메시지 보호에 있다.

---

# 2. Authentication과 Authorization의 차이

CephX를 이해할 때 가장 먼저 구분해야 하는 두 개념이다.

## Authentication

"너는 누구인가?"

예를 들어:

```text
client.cinder
```

라는 주체가 실제 `client.cinder`의 Secret Key를 가지고 있는지를 검증한다.

## Authorization

"인증된 사용자가 무엇까지 할 수 있는가?"

예를 들어 `client.cinder`가 인증됐다고 해서 Ceph 전체를 마음대로 조작할 수 있게 해서는 안 된다.

Ceph에서는 **Capabilities(Caps)** 로 권한을 제한한다.

```text
client.cinder
 ├─ 인증: Secret Key
 └─ 권한: Caps
```

따라서 다음처럼 이해하면 된다.

```text
Authentication
    |
    | "정말 client.cinder가 맞는가?"
    v
Authorization
    |
    | "맞다면 어떤 Pool까지 읽고 쓸 수 있는가?"
    v
Ceph Resource 접근
```

---

# 3. CephX Entity란 무엇인가

CephX에서 인증을 받는 주체를 **Entity**라고 생각하면 이해하기 쉽다.

예를 들면:

```text
client.admin
client.cinder
client.crash.qcs01
client.ceph-exporter.qcs01
client.rgw.quantumcns.qcs01.mnvwqh

mgr.qcs01.oosstf
mds.qksfs.qcs03.zgoutb
osd.0
```

같은 것들이다.

여기서 `client.*`라고 해서 반드시 사람이 사용하는 계정인 것은 아니다.

`client.cinder`는 OpenStack Cinder가 사용할 수 있고, `client.crash.qcs01`은 Ceph crash daemon이 사용할 수 있다.

즉 Ceph에서 "user" 또는 "client"는 일반적인 OS 사용자 계정과 같은 의미로만 보면 안 된다.

---

# 4. Entity 이름은 어떻게 구성되는가

Ceph Entity는 대체로 다음과 같은 형태다.

```text
TYPE.ID
```

예:

```text
client.admin
mgr.qcs01.oosstf
mds.qksfs.qcs03.zgoutb
osd.11
```

`client`는 일반적인 Ceph Client Identity이고, `mgr`, `mds`, `osd` 등은 Ceph daemon identity다.

예를 들어:

```text
client.admin
```

에서:

- Type: `client`
- ID: `admin`

이다.

CLI에서 다음처럼 `--id`를 사용할 때는 일반적으로 `client.`를 제외한 ID를 사용한다.

```bash
ceph --id admin -s
```

반면 full entity name을 지정하는 옵션에서는:

```bash
ceph -n client.admin -s
```

처럼 사용할 수 있다.

---

# 5. Monitor의 Auth DB

CephX Entity에 대한 인증 정보는 Monitor의 Auth Database에 저장된다.

조회:

```bash
ceph auth ls
```

특정 Entity:

```bash
ceph auth get client.cinder
```

예시는 다음 형태다.

```text
[client.example]
    key = <REDACTED>
    caps mon = "allow r"
    caps osd = "allow rw pool=example"
```

여기에는 크게 세 가지 정보가 있다.

```text
Entity
  |
  +-- Secret Key
  |
  +-- Monitor Capability
  |
  +-- OSD / MDS / MGR Capability
```

---

# 6. Secret Key는 무엇인가

CephX는 shared secret 기반 인증을 사용한다.

즉:

```text
Monitor Auth DB
    |
    | Secret A
    |
Client
    |
    | Secret A
```

처럼 양쪽이 동일한 Secret을 가지고 있어야 한다.

중요한 것은 실제 Secret을 그대로 네트워크로 보내서 비교하는 것이 아니라는 점이다.

Client는 Secret을 이용해 Monitor와 인증 과정을 수행하고, Monitor는 Client가 올바른 Secret을 보유했다는 것을 확인한다.

---

# 7. Keyring은 무엇인가

Keyring은 CephX Secret을 파일 형태로 저장하는 대표적인 방법이다.

예:

```text
/etc/ceph/ceph.client.admin.keyring
/etc/ceph/ceph.client.cinder.keyring
```

형식은 대략 다음과 같다.

```ini
[client.example]
    key = <REDACTED>
    caps mon = "allow r"
    caps osd = "allow rw pool=example"
```

Ceph 공식 문서에서 keyring은 INI 스타일 파일이며 section name이 client/daemon name이고 `key` 항목에 CephX key가 들어간다고 설명한다.

공식 문서:

https://docs.ceph.com/en/latest/rados/configuration/auth-config-ref/#keys

기본 keyring search path도 존재한다.

```text
/etc/ceph/$cluster.$name.keyring
/etc/ceph/$cluster.keyring
/etc/ceph/keyring
/etc/ceph/keyring.bin
```

---

# 8. Auth DB와 Keyring은 같은 것이 아니다

여기서 중요한 포인트가 있다.

```text
Monitor Auth DB
```

와:

```text
/etc/ceph/*.keyring
```

은 같은 저장소가 아니다.

예를 들어:

```text
Monitor Auth DB
client.cinder = Secret A

qcs01
/etc/ceph/ceph.client.cinder.keyring
client.cinder = Secret A
```

이면 정상이다.

그런데 Ceph Auth DB의 Key만 Rotation해서:

```text
Monitor Auth DB
client.cinder = Secret B

qcs01 keyring
client.cinder = Secret A
```

가 되면 다음 인증부터 실패할 수 있다.

이 때문에 Client Key Rotation은 단순히 다음 한 줄로 끝나는 작업이 아니다.

```bash
ceph auth rotate client.cinder
```

**새 Key가 저장되어야 하는 모든 Consumer를 같이 갱신해야 한다.**

---

# 9. Keyring 외에도 Secret은 다른 곳에 저장될 수 있다

실제 환경에서는 CephX Key가 `/etc/ceph/*.keyring`에만 존재하지 않는다.

예를 들어 OpenStack + Ceph 환경에서는:

```text
Ceph Auth DB
     |
     | client.cinder
     v
/etc/ceph keyring
     |
     +--> Cinder
     |
     +--> libvirt Secret
             |
             v
            VM
```

처럼 여러 위치에 복제될 수 있다.

실제 점검에서도:

```bash
grep -R "client.cinder" /etc/ceph /etc/cinder /etc/nova 2>/dev/null
```

을 통해:

```text
/etc/ceph/ceph.client.cinder.keyring
```

이 확인됐고:

```bash
virsh secret-list
```

에서는:

```text
ceph client.cinder secret
```

이 존재했다.

따라서 Client Credential을 Rotation하려면 "Ceph Auth DB의 Key가 어디에 복제되어 있는가"를 먼저 조사해야 한다.

---

# 10. Capabilities(Caps)

Ceph에서 인증된 Entity가 무엇을 할 수 있는지 결정하는 것이 Capabilities다.

대표적으로:

```text
mon
osd
mds
mgr
```

에 대한 Caps가 있다.

예를 들어:

```text
caps mon = "allow r"
```

이면 Monitor에 읽기 권한만 부여한다.

```text
caps osd = "allow rw pool=my-pool"
```

이면 특정 Pool에 읽기/쓰기를 허용한다.

---

# 11. 실제 client.guest Caps 해석

실제 점검 중 `client.guest`는 다음과 같은 형태였다.

```text
caps mds = "allow rw path=/"
caps mon = "allow r"
caps osd = "allow class-read object_prefix rbd_children, allow rw pool=cephfs_data"
```

해석하면:

### MDS

```text
allow rw path=/
```

CephFS Root Path 기준 Metadata Read/Write 권한이 있다.

### MON

```text
allow r
```

Cluster Map 등 Monitor 정보에 읽기 권한이 있다.

### OSD

```text
allow rw pool=cephfs_data
```

`cephfs_data` Pool에 대한 Read/Write 권한이 있다.

이 Caps와 실제 filesystem 상태를 비교해서 이 Credential이 과거 CephFS Client용이었음을 추론할 수 있었다.

---

# 12. CephX 인증 흐름

Ceph 공식 Architecture 문서를 기준으로 전체 과정을 단순화하면 다음과 같다.

```text
Client
  |
  | 1. Entity Name으로 MON 접속
  v
MON
  |
  | 2. Shared Secret 기반 인증
  |
  | 3. Session Key가 포함된 인증 정보 전달
  v
Client
  |
  | 4. Service Ticket 요청
  v
MON
  |
  | 5. OSD/MDS 등에 사용할 Ticket 발급
  v
Client
  |
  | 6. Ticket을 이용해 Service Daemon에 직접 접속
  +-----------------------------> OSD
  +-----------------------------> MDS
```

Ceph가 이렇게 동작하는 이유는 Ceph Client가 모든 I/O를 MON을 경유해서 처리하지 않기 때문이다.

Client는 실제 데이터 I/O 시 OSD와 직접 통신한다.

```text
            MON
           /   \
      인증/Map  \
         /       \
Client ----------> OSD
        실제 I/O
```

따라서 Client와 OSD/MDS 간 통신도 인증할 방법이 필요하고 CephX Ticket 구조가 사용된다.

---

# 13. Ticket과 Session Key

CephX를 아주 단순화하면:

```text
영구 Secret Key
      |
      | 초기 인증에 사용
      v
Session Key
      |
      | Service Ticket 요청
      v
Ticket
      |
      | OSD / MDS 접근
      v
Ceph Service
```

형태로 이해할 수 있다.

Client의 장기 Secret을 모든 요청에 직접 사용하는 것이 아니라, 인증 과정에서 Session Key와 Service Ticket을 사용한다.

Service Ticket에는 TTL이 존재한다.

CephX Config Reference에서 `auth_service_ticket_ttl`의 기본값은 1시간으로 설명되어 있다.

공식 문서:

https://docs.ceph.com/en/latest/rados/configuration/auth-config-ref/#time-to-live

---

# 14. Rotating Service Key는 또 무엇인가

이번 20.2.4 업그레이드에서 다음 Warning도 확인했다.

```text
AUTH_INSECURE_ROTATING_SERVICE_KEY_TYPE
```

여기서 "Rotating Service Key"는 개별 `client.*` Secret과 같은 개념이 아니다.

Ceph daemon들이 Client Ticket을 검증하는 데 사용하는 Service Secret은 주기적으로 Rotation된다.

공식 CephX 개발 문서에서는 daemon이 고정 Secret만 사용하는 대신 rotating secret을 사용함으로써 daemon key가 탈취됐을 때 피해 기간을 제한한다고 설명한다.

공식 문서:

https://docs.ceph.com/en/latest/dev/cephx/#rotating-service-secrets

기본적으로 기존 Key와 직전 Key를 일정 시간 같이 유지하여 Rotation 도중에도 기존 Ticket을 검증할 수 있게 한다.

---

# 15. Daemon Key와 Client Key의 차이

이번 20.2.4 업그레이드에서 매우 중요했던 구분이다.

## Service / Daemon Credential

예:

```text
osd.0
mds.qksfs.qcs03.zgoutb
mgr.qcs01.oosstf
```

## Client Credential

예:

```text
client.admin
client.cinder
client.crash.qcs01
client.rgw....
```

Cephadm 환경에서 20.2.4 upgrade 시 daemon key rotation은 상당 부분 자동화된다.

하지만 Client Key는 일반적으로 cephadm이 관리하지 않는다.

공식 문서:

https://docs.ceph.com/en/latest/rados/configuration/auth-config-ref/#upgrading-and-rotating-cephx-keys

---

# 16. Stale Auth Entity란 무엇인가

`stale auth entity`는 Ceph에 공식적으로 "Stale Entity"라는 별도 Object Type이 있다는 뜻은 아니다.

운영 관점에서 다음 상태를 의미한다.

```text
실제 Service / Daemon
        |
        X  이미 삭제됨

하지만

Monitor Auth DB
        |
        +-- 해당 Entity가 계속 존재
```

예를 들어 daemon 재배포 전:

```text
mgr.qcs01.jpqcei
```

를 사용했다고 하자.

재배포 후:

```text
mgr.qcs01.oosstf
```

가 새로 생성됐는데 Monitor Auth DB에:

```text
mgr.qcs01.jpqcei
mgr.qcs01.oosstf
```

둘 다 남아 있다면 old entity가 stale 후보가 된다.

단, 이름이 오래됐다는 이유만으로 바로 삭제해서는 안 된다.

반드시:

- 현재 daemon 목록
- 실제 process
- config
- keyring
- service
- external consumer

등을 비교해야 한다.

---

# 17. Stale Entity 확인에 사용한 방법

현재 MDS/MGR daemon:

```bash
ceph orch ps | awk '$1 ~ /^(mds|mgr)\./ {print $1}' | sort
```

Auth DB의 MDS/MGR:

```bash
ceph auth ls -f json |
jq -r '.auth_dump[].entity' |
grep -E '^(mds|mgr)\.' |
sort
```

Auth DB에만 있고 현재 daemon에는 없는 항목:

```bash
comm -23 \
  <(ceph auth ls -f json |
    jq -r '.auth_dump[].entity' |
    grep -E '^(mds|mgr)\.' |
    sort -u) \
  <(ceph orch ps |
    awk '$1 ~ /^(mds|mgr)\./ {print $1}' |
    sort -u)
```

이 명령 하나로 "삭제 확정"이 되는 것은 아니지만 stale 후보를 찾는 데 매우 유용했다.

---

# 18. `ceph auth rm`과 `ceph auth rotate`의 차이

## Delete

```bash
ceph auth rm client.example
```

Entity 자체를 Auth DB에서 제거한다.

현재 사용 중인 Credential을 삭제하면 인증 장애가 발생한다.

따라서 **stale Entity일 때만** 사용해야 한다.

## Rotate

```bash
ceph auth rotate --key-type=aes256k client.example
```

Entity와 Caps는 유지하면서 Secret Key를 새로 만든다.

하지만 기존 Client가 보유한 Key는 자동으로 바뀌지 않는다.

즉:

```text
Auth DB        Client
New Key        Old Key
    \            /
     \          /
      Authentication FAIL
```

이 될 수 있다.

---

# 19. Key를 출력할 때 주의

다음 명령들은 실제 Secret을 출력할 수 있다.

```bash
ceph auth ls
ceph auth get client.admin
ceph auth export client.admin
cat /etc/ceph/ceph.client.admin.keyring
```

따라서:

- 화면 공유
- GitHub
- Confluence
- Slack
- Issue
- Ticket

등에 그대로 붙이지 않아야 한다.

필요하면:

```text
key = <REDACTED>
```

로 마스킹한다.

---

# 20. 정리

CephX를 이해할 때 다음 관계를 기억하면 된다.

```text
Entity
  |
  +-- Secret Key -------- Monitor Auth DB
  |                           ^
  |                           |
  +-- Keyring ----------------+
  |
  +-- Capabilities
  |
  +-- CephX Authentication
           |
           v
       Session Key
           |
           v
      Service Ticket
           |
           +--> OSD
           +--> MDS
           +--> MGR / 기타 Service
```

그리고 운영에서는 반드시 다음을 구분해야 한다.

```text
현재 사용 중인 Entity
        vs
Stale Auth Entity

Key Rotation
        vs
Entity Removal

Auth DB의 Key
        vs
Client에 배포된 Keyring/Secret
```

이 구분을 이해해야 Ceph 20.2.4 이후의 `AUTH_INSECURE_*` Warning을 안전하게 처리할 수 있다.

---

# 참고 자료

- Ceph Architecture - High Availability Authentication  
  https://docs.ceph.com/en/latest/architecture/#high-availability-authentication

- CephX Config Reference  
  https://docs.ceph.com/en/latest/rados/configuration/auth-config-ref/

- Upgrading and Rotating CephX Keys  
  https://docs.ceph.com/en/latest/rados/configuration/auth-config-ref/#upgrading-and-rotating-cephx-keys

- CephX Developer Documentation  
  https://docs.ceph.com/en/latest/dev/cephx/

- Ceph User Management  
  https://docs.ceph.com/en/latest/rados/operations/user-management/
