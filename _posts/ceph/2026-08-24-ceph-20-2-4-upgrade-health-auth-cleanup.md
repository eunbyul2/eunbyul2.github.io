---
layout: post
title: "Tentacle 20.2.4 업그레이드 후 HEALTH_ERR/HEALTH_WARN 분석과 Stale CephX Auth 정리"
date: 2026-08-24 23:48:00 +0900
categories: [Ceph, Storage, Troubleshooting]
tags: [Ceph, Cephadm, Tentacle, CephX, Troubleshooting, OSD, MDS, RGW, iSCSI, NFS, RBD, CVE-2025-30156]
published: true
---

Ceph Cluster를 Tentacle 20.2.4로 업그레이드한 뒤 여러 Alert와 Health Error가 동시에 발생했다.

처음에는:

```text
CephOSDDownHigh
CephFilesystemDegraded
```

Alert가 발생했고, Cluster 내부에서는 OSD Down, PG Degraded/Peering, MDS 관련 상태와 함께 CephX Security Warning이 나타났다.

이번 작업의 핵심은 모든 Warning을 하나의 장애로 보는 것이 아니라:

```text
1. 실제 Storage/Data Plane 문제
2. Upgrade 과정의 일시적 Recovery
3. Ceph 20.2.4의 새로운 CephX Security Migration
4. 과거 서비스의 Stale Auth Entity
```

를 분리해서 확인하는 것이었다.

이 글에서는 실제 점검 순서와 판단 과정을 기록한다.

> 실제 Secret Key, UUID 일부, 내부 민감정보는 블로그에서 노출하지 않는다. `ceph auth ls`, `ceph auth get`, `ceph auth export`, keyring 파일은 실제 접근 Credential을 포함할 수 있다.

---

# 1. 환경

작업 당시 주요 Ceph 구성은 다음과 같았다.

```text
Ceph Version : Tentacle 20.2.4
Deployment   : cephadm

MON : 3
MGR : 3
OSD : 19
MDS : 5
RGW : 3

Pool : 24
PG   : 882
```

최종 정상 상태:

```text
OSD: 19 up / 19 in
PG : 882 active+clean
CephFS volumes: 3/3 healthy
RGW: 3 active
```

---

# 2. 최초 Alert

처음 확인한 Alert 중 하나는 OSD Down이었다.

```text
CephOSDDownHigh
```

당시 Alert에는 qcs02의 일부 OSD가 Down된 것으로 표시됐다.

또:

```text
CephFilesystemDegraded
```

가 발생해 MDS Rank 문제 가능성도 있었다.

이 상황에서 가장 먼저 해야 할 일은 **Alert 자체만 보고 원인을 확정하지 않는 것**이었다.

Prometheus Alert는 발생 시점의 상태를 나타내며, 직접 점검하는 순간에는 이미 상황이 바뀌어 있을 수 있기 때문이다.

---

# 3. 가장 먼저 확인한 `ceph -s`

```bash
ceph -s
```

초기에는:

- OSD 일부 Down
- degraded objects
- undersized/degraded/peering PG
- Recovery 진행
- Upgrade 진행 상태

등이 나타났다.

OSD가 Down되면 그 OSD에 존재하던 Replica를 사용할 수 없으므로 PG가 다음 상태로 바뀔 수 있다.

```text
OSD Down
   |
   v
Replica 일부 접근 불가
   |
   +--> degraded
   +--> undersized
   +--> peering
```

즉 PG Warning만 별개의 장애로 볼 것이 아니라 OSD 상태와 연결해서 봐야 한다.

---

# 4. OSD Tree 확인

```bash
ceph osd tree
```

를 통해 각 OSD가 어느 Host에 있고 현재 UP/DOWN인지 확인했다.

Alert에서는 qcs02의 OSD가 문제였지만 직접 확인 시점에는 해당 OSD가 복구됐고 다른 OSD가 일시적으로 Down 상태였다.

이를 통해 Upgrade/Daemon Restart 과정에서 OSD가 순차적으로 재기동되면서 Alert가 발생했을 가능성을 확인했다.

---

# 5. CephFS / MDS 확인

```bash
ceph mds stat
ceph fs status
```

로 MDS 상태를 확인했다.

최종적으로 모든 Filesystem Rank가 active였고 Standby MDS도 정상 존재했다.

현재 구성:

```text
cephfs
  rank 0 active
  rank 1 active

qksfs
  rank 0 active
  standby 존재

qks-ceph
  rank 0 active
```

따라서 최초 `CephFilesystemDegraded` Alert는 Recovery 이후 해소된 상태였다.

---

# 6. Storage Recovery 완료

이후 상태:

```text
19/19 OSD up
19/19 OSD in
882/882 PG active+clean
3/3 CephFS volumes healthy
3 RGW active
```

로 복구됐다.

여기서 중요한 점:

> Storage/Data Plane은 정상으로 돌아왔는데 `ceph -s`는 여전히 HEALTH_ERR 또는 HEALTH_WARN이었다.

따라서 남은 Health 상태는 Storage 장애가 아닌 다른 원인이었다.

---

# 7. 원인은 CephX Security Health Check

```bash
ceph health detail
```

에서 다음과 같은 항목이 확인됐다.

```text
AUTH_INSECURE_SERVICE_KEY_TYPE
AUTH_INSECURE_CLIENT_KEY_TYPE
AUTH_INSECURE_KEYS_ALLOWED
AUTH_INSECURE_KEYS_CREATABLE
AUTH_INSECURE_ROTATING_SERVICE_KEY_TYPE
```

Tentacle 20.2.4는 CVE-2025-30156 대응을 위해 새로운 CephX Key Type `aes256k`를 도입한다.

공식 Release:

https://www.ceph.io/en/news/blog/2026/v20-2-4-v19-2-6-combo-released/

CVE:

https://docs.ceph.com/en/latest/security/CVE-2025-30156/

공식 Release에서도 Upgrade 과정에서 6개의 신규 Health Warning/Error가 발생하는 것은 정상적인 Migration 과정이라고 설명한다.

즉:

```text
Health Warning 존재
        !=
Data가 깨짐
```

이다.

물론 Warning을 무시해도 된다는 의미는 아니다. Security Migration을 완료해야 한다.

---

# 8. Upgrade 완료 여부 확인

먼저 Ceph Version Upgrade 자체가 완료됐는지 확인했다.

```bash
ceph orch upgrade status
```

결과:

```text
There are no upgrades in progress currently.
```

그리고:

```bash
ceph versions
```

로 daemon version을 확인했다.

당시 총 33개 Ceph daemon이 모두 `20.2.4`였다.

```text
mon : 3
mgr : 3
osd : 19
mds : 5
rgw : 3
```

즉 Binary/Container Version Upgrade는 끝난 상태였다.

---

# 9. Service Key Warning에서 Stale Daemon Entity 발견

`ceph health detail`에는 현재 daemon과 맞지 않는 old MDS/MGR Entity도 존재했다.

현재 daemon:

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

두 목록을 비교:

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

현재 daemon과 매칭되지 않는 old MDS/MGR auth entity 10개가 확인됐다.

추가로:

```bash
ceph mgr metadata
ceph mds metadata
ceph fs dump
ceph orch ls
```

를 확인하여 실제 현재 service가 해당 old entity를 사용하지 않음을 검증했다.

---

# 10. 삭제 전 무조건 Backup

Stale로 보이는 Auth Entity도 즉시 삭제하지 않았다.

먼저:

```bash
ceph auth export <ENTITY>
```

로 개별 Backup을 생성했다.

예:

```bash
mkdir -p /root/ceph-auth-backup-20260824
chmod 700 /root/ceph-auth-backup-20260824
```

그리고:

```bash
ceph auth export <ENTITY> \
> /root/ceph-auth-backup-20260824/<ENTITY>.keyring
```

파일 권한:

```bash
chmod 600 /root/ceph-auth-backup-20260824/*.keyring
```

Secret이 들어있는 파일이므로 일반 사용자에게 읽기 권한을 주지 않았다.

---

# 11. Stale MDS/MGR Entity 제거

검증한 old Entity를:

```bash
ceph auth rm <ENTITY>
```

로 **한 개씩** 제거했다.

매번:

```bash
ceph -s
ceph health detail
ceph fs status
```

를 확인했다.

그 결과:

```text
AUTH_INSECURE_SERVICE_KEY_TYPE
```

가 해소됐고 Storage 상태는 정상 유지됐다.

---

# 12. Client Insecure Key가 49개 남음

Service Entity 정리 후:

```text
49 auth client entities with insecure key types
```

가 남았다.

목록 추출:

```bash
ceph health detail | \
sed -n '/AUTH_INSECURE_CLIENT_KEY_TYPE/,/AUTH_INSECURE_KEYS_ALLOWED/p' | \
grep 'entity ' | \
awk '{print $2}'
```

개수:

```bash
ceph health detail | \
sed -n '/AUTH_INSECURE_CLIENT_KEY_TYPE/,/AUTH_INSECURE_KEYS_ALLOWED/p' | \
grep 'entity ' | \
wc -l
```

결과:

```text
49
```

---

# 13. 49개를 전부 Rotate하지 않은 이유

처음 생각할 수 있는 방법은:

```bash
ceph auth rotate --key-type=aes256k <ENTITY>
```

이다.

하지만 49개 중에는 **현재 Service가 존재하지 않는 old Credential**도 있었다.

이런 Credential까지 새 Key로 Rotation할 이유는 없다.

그래서:

```text
49 Client Entity
      |
      +-- 현재 사용
      |
      +-- Stale
      |
      +-- 사용 여부 미확인
```

으로 먼저 분류했다.

---

# 14. Old RGW Entity 점검

현재 RGW:

```bash
ceph orch ps | grep '^rgw'
```

결과 실제 daemon은:

```text
rgw.quantumcns.qcs01.mnvwqh
rgw.quantumcns.qcs02.zxzixz
rgw.quantumcns.qcs03.ruabri
```

였다.

각 Host의 실제 Process 확인:

```bash
ps -ef | grep '[r]adosgw'
```

qcs01:

```text
radosgw -n client.rgw.quantumcns.qcs01.mnvwqh
```

qcs02:

```text
radosgw -n client.rgw.quantumcns.qcs02.zxzixz
```

qcs03:

```text
radosgw -n client.rgw.quantumcns.qcs03.ruabri
```

즉 실제 사용 Credential이 명확했다.

반면 Auth DB에는 old RGW Credential:

```text
client.rgw.
client.rgw.qcs01
client.rgw.qcs01.rgw0
client.rgw.qcs02
client.rgw.qcs02.rgw0
client.rgw.qcs03
client.rgw.qcs03.rgw0
```

가 있었다.

---

# 15. cephadm Config에서도 Old RGW 검색

현재 cephadm data:

```text
/var/lib/ceph/<FSID>/
```

안에서 old Credential을 검색했다.

```bash
grep -R -l -E \
'client\.rgw\.qcs01|client\.rgw\.qcs02|client\.rgw\.qcs03|client\.rgw\.qcs0[123]\.rgw0' \
/var/lib/ceph/<FSID> \
--include='config' \
--include='*.conf' \
--include='*.keyring' \
2>/dev/null
```

결과 없음.

현재 daemon directory도:

```text
rgw.quantumcns.qcs01.mnvwqh
rgw.quantumcns.qcs02.zxzixz
rgw.quantumcns.qcs03.ruabri
```

만 존재했다.

---

# 16. Old RGW 7개 Backup 후 삭제

별도 Backup:

```bash
mkdir -p /root/ceph-auth-backup-20260824/rgw
chmod 700 /root/ceph-auth-backup-20260824/rgw
```

각 Entity Export 후:

```bash
chmod 600 /root/ceph-auth-backup-20260824/rgw/*.keyring
```

그리고 한 개씩:

```bash
ceph auth rm client.rgw.qcs01
ceph -s
ceph orch ps | grep '^rgw'
```

형태로 수행했다.

숫자는:

```text
49 -> 48 -> ... -> 42
```

로 감소했고 RGW 3개는 계속 Running이었다.

---

# 17. iSCSI 12개 점검

남은 목록에는:

```text
client.iscsi.*
```

가 12개 존재했다.

이름에 주로 qcs04/qcs05가 포함되어 있어 두 Host를 우선 확인했다.

Process:

```bash
ps -ef | grep -Ei '[c]eph-iscsi|[r]bd-target|[t]cmu'
```

Systemd:

```bash
systemctl list-units --type=service | \
grep -Ei 'iscsi|rbd-target|tcmu'
```

Package:

```bash
dpkg -l | grep -Ei 'ceph-iscsi|tcmu-runner|targetcli|rtslib'
```

Podman:

```bash
podman ps -a --format '{{.Names}}' | \
grep -Ei 'iscsi|tcmu|rbd-target'
```

cephadm:

```bash
ceph orch ps | grep -i iscsi
ceph orch ls | grep -i iscsi
```

모두 현재 사용 흔적이 없었다.

따라서 12개를 별도 Backup한 뒤 하나씩 제거했다.

```text
42 -> 30
```

---

# 18. NFS와 NFS-Ganesha를 구분

남은 Entity:

```text
client.nfs.ceph-nfs.1
client.nfs.nfs.qcs.1
```

때문에 NFS 사용 여부를 조사했다.

qcs04에는 실제 NFS Service가 있었다.

```bash
systemctl list-units --type=service --all | grep -Ei 'ganesha|nfs'
```

여기서:

```text
nfs-server.service active
nfs-mountd.service active
...
nfs-ganesha.service masked/inactive/dead
```

였다.

Package:

```bash
dpkg -l | grep -Ei 'ganesha|nfs-ganesha'
```

결과:

```text
rc nfs-ganesha
```

Debian Package Status에서 `rc`는 Package Binary는 제거됐지만 Config가 남은 상태다.

즉:

```text
Kernel NFS      -> 사용 중
NFS-Ganesha     -> 사용하지 않음
```

이었다.

---

# 19. 현재 NFS Export가 Ceph인지 확인

```bash
exportfs -v
```

현재 Export:

```text
/data1
/nfs
```

이었다.

Backing Filesystem:

```bash
findmnt /data1
findmnt /nfs
df -Th /data1 /nfs
```

결과:

```text
/data1 -> local LVM/XFS
/nfs   -> root filesystem의 XFS directory
```

였다.

즉 현재 NFS는:

```text
Local XFS
   |
   v
Kernel NFS
```

구조였고 CephFS + Ganesha가 아니었다.

qcs01~07에서:

```bash
ps -ef | grep '[g]anesha'
systemctl is-active nfs-ganesha
```

도 모두 Ganesha 미사용으로 확인됐다.

따라서 NFS Credential 2개를 Backup 후 삭제:

```text
30 -> 28
```

---

# 20. RBD Mirror 확인

남은:

```text
client.rbd-mirror-peer
```

를 조사했다.

Daemon:

```bash
ceph orch ps | grep -i rbd-mirror
ceph orch ls | grep -i rbd-mirror
ps -ef | grep '[r]bd-mirror'
```

결과 없음.

모든 Pool:

```bash
ceph osd pool ls
```

그리고:

```bash
for pool in $(ceph osd pool ls); do
  echo "===== $pool ====="
  rbd mirror pool info "$pool" 2>&1
done
```

결과 모든 Pool:

```text
Mode: disabled
```

였다.

즉:

```text
rbd-mirror daemon 없음
+
mirroring enabled pool 없음
```

이므로 해당 Peer Credential을 Stale로 판단했다.

Backup 후 삭제:

```text
28 -> 27
```

---

# 21. client.guest 조사

마지막 Stale 후보:

```text
client.guest
```

Caps:

```text
mds allow rw path=/
mon allow r
osd allow rw pool=cephfs_data
```

과거 `cephfs` Client 용도로 보였다.

qcs01~07에는:

```text
/etc/ceph/ceph.client.guest.keyring
```

이 모두 존재했다.

하지만:

```bash
findmnt | grep -Ei 'ceph|fuse.ceph' | grep 'name=guest'
```

결과 없음.

Systemd/fstab/script:

```bash
grep -R -n -F 'client.guest' \
/etc/systemd /etc/fstab /etc/cron* /usr/local /opt \
2>/dev/null
```

결과 없음.

그리고 Keyring SHA256:

```bash
sha256sum /etc/ceph/ceph.client.guest.keyring
```

은 qcs01~07에서 모두 동일했다.

즉 하나의 old Credential이 과거 여러 Host에 공통 배포된 형태였다.

---

# 22. CephFS Client 상태와 비교

```bash
ceph fs status
```

결과:

```text
cephfs   : 0 clients
qksfs    : 29 clients
qks-ceph : 0 clients
```

`client.guest`가 권한을 가진 Pool은 old `cephfs_data`였고 현재 해당 `cephfs`에는 Client가 없었다.

반면 qksfs에는 Kubernetes CSI Client가 붙어 있었다.

따라서 `client.guest`를 Backup 후 제거했다.

```text
27 -> 26
```

---

# 23. 최종 Stale Auth 정리 결과

최초:

```text
49 auth client entities with insecure key types
```

정리:

| 구분 | 제거 수 |
|---|---:|
| Old RGW | 7 |
| Old iSCSI | 12 |
| Old NFS-Ganesha | 2 |
| RBD Mirror | 1 |
| guest | 1 |
| 합계 | 23 |

따라서:

```text
49 - 23 = 26
```

현재:

```text
26 auth client entities with insecure key types
```

가 남았다.

---

# 24. 남은 26개

```text
client.admin

client.bootstrap-mds
client.bootstrap-mgr
client.bootstrap-osd
client.bootstrap-rbd
client.bootstrap-rbd-mirror
client.bootstrap-rgw

client.ceph-exporter.qcs01
client.ceph-exporter.qcs02
client.ceph-exporter.qcs03
client.ceph-exporter.qcs04
client.ceph-exporter.qcs05
client.ceph-exporter.qcs06
client.ceph-exporter.qcs07

client.cinder

client.crash.qcs01
client.crash.qcs02
client.crash.qcs03
client.crash.qcs04
client.crash.qcs05
client.crash.qcs06
client.crash.qcs07

client.qks-gpu-cephfs

client.rgw.quantumcns.qcs01.mnvwqh
client.rgw.quantumcns.qcs02.zxzixz
client.rgw.quantumcns.qcs03.ruabri
```

이제부터는 삭제 단계가 아니다.

이 Entity들은 현재 사용 중이거나 Ceph 운영에 필요한 Credential일 가능성이 높다.

---

# 25. 실제 사용이 확인된 Client

## `client.cinder`

```text
/etc/ceph/ceph.client.cinder.keyring
```

존재.

또:

```bash
virsh secret-list
```

에서 `ceph client.cinder secret` 확인.

즉 OpenStack/libvirt와 연결된 실사용 Credential이다.

---

## current RGW

실제 Process:

```text
radosgw -n client.rgw.quantumcns....
```

로 확인됐으므로 사용 중이다.

---

## `client.admin`

Kubernetes Node의 CephFS CSI Mount를 확인했을 때:

```text
name=admin
mds_namespace=qksfs
```

가 확인됐다.

즉 `client.admin`을 단순 관리 CLI Key로만 생각해서는 안 되고 현재 Workload Mount에서도 사용 중인 상황을 고려해야 한다.

---

# 26. 왜 여기서 `auth_allowed_ciphers aes256k`로 바로 바꾸지 않았는가

현재 Monitor는 legacy `aes` Authentication도 허용한다.

이 때문에:

```text
AUTH_INSECURE_KEYS_ALLOWED
```

가 남는다.

하지만 아직:

```text
26 auth client entities with insecure key types
```

가 있다.

이 상태에서 먼저:

```bash
ceph mon set auth_allowed_ciphers aes256k
```

를 수행하면 기존 `aes` Key밖에 모르는 Client가 Authentication에 실패할 수 있다.

Ceph 공식 Health Check 문서도 `AUTH_INSECURE_KEYS_ALLOWED`를 너무 일찍 해소하면 Administrative Access를 잃을 수 있다고 경고한다.

공식 문서:

https://docs.ceph.com/en/latest/rados/operations/health-checks/#auth-insecure-keys-allowed

따라서 순서는:

```text
Client Compatibility 확인
        |
        v
Client Key Rotation
        |
        v
모든 Consumer에 New Key 배포
        |
        v
AUTH_INSECURE_CLIENT_KEY_TYPE 해소
        |
        v
마지막에 aes Authentication 차단
```

이어야 한다.

---

# 27. Kernel Client 호환성 확인 필요

Tentacle 20.2.4의 `aes256k`는 Client Software도 지원해야 한다.

Ceph 공식 Release:

https://www.ceph.io/en/news/blog/2026/v20-2-4-v19-2-6-combo-released/

에 따르면 Upstream Linux Kernel Ceph Client의 `aes256k` 지원은 Kernel 7.0부터 시작한다.

CentOS Stream 9/10에는 Backport가 들어갔다.

Ubuntu 등 다른 Distribution은 Vendor Backport 여부를 확인해야 한다.

따라서 CephFS Kernel Mount 또는 Kernel RBD가 있는 환경에서:

```bash
uname -r
```

만 보는 것으로 끝나는 것이 아니라 해당 Distribution의 Kernel Package가 `aes256k` 지원 Patch를 Backport했는지 확인해야 한다.

---

# 28. 다음 작업 계획

현재 상태:

```text
Storage             : 정상
Service Key Migration: 대부분 cephadm에 의해 완료
Stale Client Cleanup : 완료
Client Key Migration : 미완료
```

다음 단계:

```text
26 Client Entity
       |
       +-- 어디에서 사용되는지 Mapping
       |
       +-- Client Software/Kernel Version 확인
       |
       +-- 새 aes256k Key 지원 여부 확인
       |
       +-- Backup
       |
       +-- Key Rotation
       |
       +-- 모든 Consumer Secret 갱신
       |
       +-- Service Test
       |
       +-- AUTH_INSECURE_CLIENT_KEY_TYPE 해소
       |
       +-- auth_allowed_ciphers에서 aes 제거
```

---

# 29. 이번 작업에서 배운 Troubleshooting 원칙

## 1. Alert 발생 시점과 현재 상태를 구분한다

Alert에 나온 OSD와 현재 Down OSD가 달라질 수 있다.

## 2. `ceph -s`의 HEALTH_ERR만 보고 Storage 장애로 단정하지 않는다

Health Error가 Security Configuration 때문일 수 있다.

## 3. Auth Entity는 이름만 보고 삭제하지 않는다

반드시:

```text
orch
process
service
config
keyring
mount
consumer
```

를 확인한다.

## 4. 삭제 전에 `ceph auth export`

Credential은 복구 가능성을 확보하고 제거한다.

## 5. 한 개씩 삭제하고 Health를 본다

```text
remove one
   |
   v
ceph -s
   |
   v
Service 정상?
   |
   +-- Yes -> next
   +-- No  -> stop / restore
```

## 6. Key Rotation과 Entity Delete를 구분한다

```text
Stale     -> Delete
Active    -> Rotate + Redistribute
```

---

# 30. 참고 자료

## Ceph 20.2.4 Release

https://www.ceph.io/en/news/blog/2026/v20-2-4-v19-2-6-combo-released/

## CVE-2025-30156

https://docs.ceph.com/en/latest/security/CVE-2025-30156/

## CephX Key Upgrade / Rotation

https://docs.ceph.com/en/latest/rados/configuration/auth-config-ref/#upgrading-and-rotating-cephx-keys

## Ceph Health Checks

https://docs.ceph.com/en/latest/rados/operations/health-checks/

## CephX Architecture

https://docs.ceph.com/en/latest/architecture/#high-availability-authentication

---

# 마무리

이번 문제는 단순히 "Ceph Upgrade 후 Warning이 생겼다" 수준이 아니었다.

실제로는:

```text
20.2.4 Security Upgrade
        |
        +-- Storage daemon rolling update
        |
        +-- CVE-2025-30156 Fix
        |
        +-- aes256k 도입
        |
        +-- Legacy CephX Key Detection
        |
        +-- Stale Auth Entity 발견
        |
        +-- 49 -> 26 정리
        |
        +-- 앞으로 Active Client Key Migration 필요
```

라는 여러 단계가 겹쳐 있었다.

Storage가 정상으로 복구된 것과 Security Migration이 완료된 것은 별개의 상태다.

현재 Cluster의 데이터 경로는 정상화됐지만, 남은 26개 Client Credential에 대한 `aes -> aes256k` Migration은 별도 작업으로 계속 진행해야 한다.
