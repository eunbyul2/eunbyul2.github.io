---
layout: post
title: "[Ceph] RBD Live Migration 동작 원리와 중단 시 대응"
date: 2026-09-13 16:00:00 +0900
categories: [Infrastructure, Ceph, Storage]
tags: [Ceph, RBD, Live Migration, Block Storage, Troubleshooting, OpenStack]
published: true
---

# Ceph RBD Live Migration 동작 원리와 중단 시 대응

Ceph에서 RBD Image를 다른 Pool이나 다른 Layout으로 옮길 때는 `RBD Live Migration` 기능을 사용할 수 있다. 이 작업은 단순히 Source Image를 Target Image로 복사하는 것이 아니라, Source와 Target을 연결한 상태에서 Client I/O를 Target으로 전환하고 데이터를 백그라운드로 옮기는 방식이다.

이 글에서는 다음 내용을 중심으로 정리한다.

- RBD Live Migration이 필요한 이유
- `Prepare → Execute → Commit/Abort` 단계별 동작
- Migration이 중단되었을 때 확인해야 할 항목
- `Commit`과 `Abort`를 선택하는 기준
- OpenStack Cinder Backend에서 작업할 때의 주의사항

> 이 글의 Live Migration 동작과 명령어는 Ceph 공식 문서를 기준으로 정리했다. 장애 원인 점검 순서와 OpenStack 관련 주의사항은 공식 Live Migration 절차를 바탕으로 보완한 운영 관점의 내용이다.
https://docs.ceph.com/en/latest/rbd/rbd-live-migration/?utm_source=chatgpt.com

---

## 1. RBD와 RBD Image

Ceph는 하나의 Cluster에서 Block, File, Object Storage Interface를 제공한다.

| 구분 | Ceph Interface | 주요 용도 |
|---|---|---|
| Block Storage | RBD | VM Disk, OpenStack Cinder Volume, Kubernetes PV 등 |
| File Storage | CephFS | 여러 Client가 Mount하여 사용하는 File System |
| Object Storage | RGW | S3 또는 Swift API 기반 Object Storage |

**RBD(RADOS Block Device)** 는 Ceph가 제공하는 Block Storage Interface이다. RBD에서 하나의 가상 Block Device를 **RBD Image**라고 한다.

예를 들어 OpenStack Cinder의 Ceph Backend가 `volumes_hdd` Pool을 사용한다면, 다음과 같은 형태로 Volume이 저장될 수 있다.

```text
Ceph Cluster
└── Pool: volumes_hdd
    └── RBD Image: volume-<Cinder Volume UUID>
```

- **Pool**: Ceph Object가 저장되는 논리적인 구역
- **RBD Image**: Client가 하나의 Block Device처럼 사용하는 가상 디스크
- **Object**: RBD Image의 데이터가 내부적으로 분할되어 저장되는 단위
- **OSD**: 실제 Object를 저장하고 복제하는 Ceph Daemon

RBD Image의 기본 정보는 다음 명령으로 확인한다.

```bash
rbd info <POOL>/<IMAGE>
```

예시는 다음과 같다.

```bash
rbd info volumes_hdd/volume-<UUID>
```

---

## 2. RBD Live Migration이란?

RBD Live Migration은 Source RBD Image를 새로운 Target Image로 옮기는 기능이다. 같은 Ceph Cluster 안에서 다음과 같은 변경에 사용할 수 있다.

- 다른 Pool로 Image 이동
- Image Format 변경
- Object Size, Stripe Unit, Stripe Count와 같은 Image Layout 변경
- Data Pool 변경
- Parent Image와의 연결 유지 또는 Flatten

Ceph는 다른 Cluster의 RBD Image나 외부 File, HTTP(S), S3, NBD Source를 가져오는 Import-only 방식도 지원하지만, 이 글에서는 **같은 Ceph Cluster 내부의 일반 RBD Live Migration**만 다룬다.

예를 들면 다음과 같다.

```text
Source: volumes_hdd/test-volume
                ↓
Target: volumes_ssd/test-volume
```

Migration을 시작하면 Ceph는 Source의 초기화된 Block을 Target으로 Deep Copy한다. 이때 Snapshot History도 함께 가져오며, 가능한 경우 데이터의 Sparse Allocation을 유지한다.

### 2.1 단순 Copy와 다른 점

사용 중인 Block Device를 일반적인 방식으로 복사하면 복사 도중 Source에 새 Write가 발생하여 Source와 Target의 데이터 시점이 달라질 수 있다.

RBD Live Migration은 Prepare 단계에서 Source와 Target 사이에 연결 관계를 만든다. 아직 Target에 복사되지 않은 영역을 읽으면 Source에서 읽고, 해당 영역에 Write가 발생하면 겹치는 Source Block을 Target으로 먼저 Deep Copy한 뒤 Target에 기록한다.

따라서 전체 데이터를 한 번에 복사하지 않아도 Target을 사용할 수 있으며, 나머지 데이터 복사는 백그라운드에서 진행할 수 있다.

> 이름에 `Live`가 포함되어 있지만 처음부터 끝까지 완전한 무중단 작업이라는 의미는 아니다. 일반 Migration에서는 Prepare 전에 Source Image를 Read/Write로 사용하는 Client를 중지해야 한다.

---

## 3. 전체 동작 과정

RBD Live Migration은 다음 단계로 진행된다.

```text
Prepare
  └─ Target 생성, Source와 Target 연결, Client 전환 준비

Execute
  └─ Source의 초기화된 Block을 Target으로 백그라운드 Deep Copy

Finish
  ├─ Commit: Target을 최종 Image로 확정
  └─ Abort: Target을 제거하고 Source로 복구
```

| 단계 | 핵심 동작 | Source | Target |
|---|---|---|---|
| Prepare 전 | 기존 Image 사용 | Client가 사용 중 | 없음 |
| Prepare 후 | Source와 Target 연결 | Read-only, RBD Trash로 이동 | Client가 사용할 새 Image |
| Execute 중 | Block Deep Copy | 미복사 데이터의 원본 | Client I/O와 복사 데이터를 수용 |
| Commit 후 | Migration 확정 | 제거 | 독립된 최종 Image |
| Abort 후 | Migration 이전으로 복구 | 접근 복구 | 제거 |

---

## 4. Prepare Migration

### 4.1 Client 중지

일반적인 Live Migration에서는 Prepare 전에 Source Image를 사용하는 모든 Client를 중지해야 한다.

Source Image를 Read/Write Mode로 열고 있는 Client가 있으면 Prepare가 실패한다. 이는 Prepare 이후 Client가 기존 Source가 아니라 새 Target Image를 사용하게 하기 위한 과정이다.

Client 연결 상태는 다음과 같이 확인할 수 있다.

```bash
rbd status <SOURCE_POOL>/<SOURCE_IMAGE>
```

출력의 `Watchers`에 Source를 열고 있는 Client가 표시된다면 어떤 VM, Process 또는 Storage Consumer인지 먼저 식별해야 한다.

### 4.2 Prepare 실행

```bash
rbd migration prepare \
  <SOURCE_POOL>/<SOURCE_IMAGE> \
  <TARGET_POOL>/<TARGET_IMAGE>
```

예시는 다음과 같다.

```bash
rbd migration prepare \
  volumes_hdd/test-volume \
  volumes_ssd/test-volume
```

Prepare 단계에서는 다음 작업이 수행된다.

1. 새로운 Target Image를 생성한다.
2. Target을 Source와 연결한다.
3. Source를 Target과 연결하고 Read-only로 표시한다.
4. 같은 Cluster 내부의 일반 Migration에서는 Source Image를 RBD Trash로 이동한다.

Source가 Trash로 이동하는 이유는 Migration 중 기존 Image 이름으로 Client가 잘못 접근하는 것을 막기 위해서이다. 이 시점에 Source 데이터가 즉시 삭제된 것은 아니다.

### 4.3 Prepare 상태 확인

```bash
rbd status <TARGET_POOL>/<TARGET_IMAGE>
```

정상적으로 준비되었다면 다음과 같이 `state: prepared`가 표시된다.

```text
Migration:
    source: volumes_hdd/test-volume (...)
    destination: volumes_ssd/test-volume (...)
    state: prepared
```

Source Image는 다음 명령으로 확인한다.

```bash
rbd trash ls --all
```

Prepare 완료 후에는 Client 설정을 새 Target Image 이름으로 변경한 뒤 다시 시작할 수 있다. Source의 기존 이름으로 다시 연결하려고 하면 실패한다.

---

## 5. Execute Migration

Prepare가 완료되면 Source의 초기화된 Block을 Target으로 복사한다.

```bash
rbd migration execute <TARGET_POOL>/<TARGET_IMAGE>
```

예시는 다음과 같다.

```bash
rbd migration execute volumes_ssd/test-volume
```

Execute는 Target Image를 Client가 사용하는 동안에도 백그라운드에서 실행할 수 있다.

진행률은 다음과 같이 확인한다.

```bash
rbd status volumes_ssd/test-volume
```

```text
Migration:
    source: volumes_hdd/test-volume (...)
    destination: volumes_ssd/test-volume (...)
    state: executing (32% complete)
```

Deep Copy가 모두 완료되면 상태가 `executed`로 변경된다.

```text
Migration:
    source: volumes_hdd/test-volume (...)
    destination: volumes_ssd/test-volume (...)
    state: executed
```

---

## 6. Commit Migration

모든 Block의 Deep Copy가 완료되고 Target의 정상 동작을 검증했다면 Migration을 확정한다.

```bash
rbd migration commit <TARGET_POOL>/<TARGET_IMAGE>
```

예시는 다음과 같다.

```bash
rbd migration commit volumes_ssd/test-volume
```

Commit하면 다음 작업이 수행된다.

- Source와 Target 사이의 Cross-link 제거
- 기존 Source Image 제거
- Target Image를 독립적인 최종 Image로 확정

Commit 이후에는 기존 Source로 간단히 되돌릴 수 없으므로 최소한 다음 항목을 확인한 뒤 실행해야 한다.

- `rbd status`가 `executed`인지
- Target을 사용하는 Client가 정상인지
- Application Read/Write가 정상인지
- Snapshot과 Clone 관계가 예상과 일치하는지
- 별도 Backup 또는 복구 수단이 준비되어 있는지

Source Image가 하나 이상의 Clone의 Parent라면, Ceph 공식 문서상 `--force`가 필요할 수 있다. 이 경우 Descendant Clone이 사용 중이지 않은지 확인해야 하며, Parent/Clone 관계를 충분히 검토하지 않고 `--force`를 사용해서는 안 된다.

---

## 7. Abort Migration

Prepare 또는 Execute 단계를 되돌려 기존 Source로 복구하려면 다음 명령을 사용한다.

```bash
rbd migration abort <TARGET_POOL>/<TARGET_IMAGE>
```

예시는 다음과 같다.

```bash
rbd migration abort volumes_ssd/test-volume
```

Abort는 단순히 복사 Process를 일시 정지하는 명령이 아니다. 다음과 같이 Migration을 되돌리는 작업이다.

- Source와 Target의 Cross-link 제거
- Target Image 삭제
- 기존 Source Image에 대한 접근 복구

따라서 Abort 전에 반드시 Target 사용 여부와 데이터 상태를 확인해야 한다. 공식 Live Migration 문서는 Abort 시 Target이 삭제되고 Source 접근이 복구된다고 설명한다. 즉, Target을 사용하던 Client가 있다면 서비스를 먼저 중지하고 데이터 영향 범위를 검토해야 한다.

> 장애가 발생했다는 이유만으로 즉시 Abort하면 안 된다. Abort의 직접적인 결과는 Target 삭제이므로, 현재 Client가 어느 Image에 연결되어 있는지 확인하지 않은 상태에서 실행하면 서비스와 데이터에 영향을 줄 수 있다.

---

## 8. Migration 중단 시 점검 순서

Migration 진행률이 장시간 변하지 않거나 `execute`가 오류로 종료되었다면, 바로 Commit 또는 Abort하지 않고 현재 상태와 중단 원인을 먼저 확인한다.

### 8.1 작업 변경 중지 및 정보 기록

우선 추가적인 `commit`, `abort`, `trash remove` 또는 수동 Image 삭제를 중지한다. 다음 정보를 기록한다.

- Source Pool/Image
- Target Pool/Image
- Migration State와 진행률
- 명령 실행 시각과 오류 메시지
- Source/Target Watcher
- Client가 현재 사용 중인 Image
- Ceph Health 상태

### 8.2 Migration State 확인

```bash
rbd status <TARGET_POOL>/<TARGET_IMAGE>
```

| 상태 | 의미 | 기본 판단 |
|---|---|---|
| `prepared` | Source와 Target 연결은 생성됐지만 전체 복사는 시작 전 | Client 전환과 Cluster 상태 확인 후 Execute 검토 |
| `executing` | Block Deep Copy 진행 중 | 진행률, Cluster Health, I/O와 Log 확인 |
| `executed` | Deep Copy 완료 | Target 검증 후 Commit 검토 |

CLI 오류가 발생했더라도 실제 Migration State가 남아 있을 수 있으므로, 명령의 종료 메시지만 보고 상태를 판단하면 안 된다.

### 8.3 Ceph Cluster Health 확인

```bash
ceph -s
ceph health detail
```

다음 문제를 중점적으로 확인한다.

- OSD `down` 또는 `out`
- PG가 `inactive`, `incomplete`, `stale`, `peering`, `degraded` 상태인지
- Pool 또는 OSD가 `nearfull`, `backfillfull`, `full` 상태인지
- Recovery/Backfill이 과도하게 진행 중인지
- MON Quorum이나 Network 문제가 있는지
- Slow Ops 또는 Storage I/O 오류가 있는지

`HEALTH_WARN` 자체만으로 Migration 장애라고 단정할 수는 없다. `ceph health detail`에서 경고 항목이 해당 Source/Target Pool의 I/O와 실제로 관련 있는지 확인해야 한다.

### 8.4 Source와 Target 확인

```bash
rbd info <TARGET_POOL>/<TARGET_IMAGE>
rbd status <TARGET_POOL>/<TARGET_IMAGE>
rbd trash ls --all
```

다음 항목을 확인한다.

- Target Image가 존재하는지
- Migration Metadata에 Source와 Target이 정상적으로 표시되는지
- Source가 RBD Trash에 존재하는지
- Source 또는 Target에 Watcher가 있는지
- 예상하지 않은 Snapshot이나 Clone 관계가 있는지

Migration이 끝나기 전에 Source를 Trash에서 강제로 제거하거나 Target을 수동 삭제하면 복구 경로를 훼손할 수 있다.

### 8.5 용량과 OSD 상태 확인

```bash
ceph df
ceph osd df tree
ceph osd tree
ceph pg stat
```

Target Pool에 논리적인 여유 공간이 보이더라도 실제 CRUSH 배치 대상 OSD 일부가 Full에 가까우면 쓰기가 지연되거나 실패할 수 있다. Cluster 전체 사용률뿐 아니라 Target Pool의 CRUSH Rule이 사용하는 OSD의 개별 사용률도 확인한다.

### 8.6 Log 확인

정확한 Log 위치는 Ceph 배포 방식과 버전에 따라 다르다. `cephadm` 환경이라면 관련 Daemon Log와 Cluster Log를 확인하고, 패키지 기반 배포라면 `/var/log/ceph/` 및 `journalctl`을 확인한다.

확인할 키워드의 예시는 다음과 같다.

- Source/Target Image 이름 또는 ID
- `migration`
- `rbd`
- `slow request`
- `I/O error`
- `No space left`
- `connection reset` 또는 `timeout`

---

## 9. 중단 이후 Continue, Commit, Abort 판단

중단 원인을 확인한 뒤 다음 기준으로 조치 방향을 결정한다.

| 상황 | 조치 방향 |
|---|---|
| 일시적인 Network·OSD·용량 문제가 해결되었고 Source/Target 관계가 정상 | 현재 상태와 버전 동작을 확인한 뒤 Execute 재시도 검토 |
| State가 `executed`이고 Target I/O가 정상 | 검증 후 Commit |
| Migration 계획을 취소했고 기존 Source로 복구해야 함 | Client 중지와 영향 확인 후 Abort |
| Target Pool 또는 Target Storage를 신뢰할 수 없음 | 데이터 영향 분석 후 Abort 검토 |
| Metadata, Source 또는 Target 상태가 서로 맞지 않음 | 수동 삭제 금지, Log와 Ceph 지원 절차를 통해 추가 분석 |

여기서 `Continue`는 별도의 `rbd migration continue` 명령을 의미하지 않는다. Ceph 공식 Live Migration 절차에 정의된 실행 명령은 `rbd migration execute`이다. 중단 후 같은 명령을 다시 수행할 수 있는지는 **현재 Migration State, Ceph Version, 오류 원인 및 Source/Target의 일관성**을 확인한 뒤 판단해야 한다.

공식 문서에는 모든 장애 유형에 공통으로 적용되는 자동 복구 또는 무조건적인 재실행 절차가 제시되어 있지 않다. 따라서 상태가 불명확하다면 운영 Image에서 즉시 재시도하지 말고 동일 Version의 Test 환경에서 동작을 검증하거나 Ceph 지원 절차를 따르는 것이 안전하다.

---

## 10. OpenStack Cinder 환경의 주의사항

OpenStack Cinder Volume이 Ceph RBD Image로 저장되어 있더라도, Backend RBD Image만 직접 옮기면 작업이 끝나는 것은 아니다.

Cinder와 Nova는 다음과 같은 Control Plane 정보를 별도로 관리한다.

- Volume ID와 Backend Host
- Volume Type
- Pool 정보
- Attachment 정보
- Connection 정보
- VM의 Block Device Mapping

따라서 Cinder가 관리하는 RBD Image에 `rbd migration prepare/execute/commit`을 직접 실행하면 Ceph의 실제 Image 위치와 Cinder DB의 Backend 정보가 달라질 수 있다. 이 상태에서는 Horizon이나 OpenStack CLI의 상태와 실제 Ceph/Libvirt 상태가 불일치할 수 있다.

OpenStack 관리 Volume을 이동할 때는 우선 다음과 같이 OpenStack이 제공하는 Volume Migration 또는 Retype 절차가 환경에서 지원되는지 검토해야 한다.

```bash
openstack volume migrate --host <HOST@BACKEND#POOL> <VOLUME_ID>
```

```bash
openstack volume set --type <TARGET_VOLUME_TYPE> \
  --migration-policy on-demand \
  <VOLUME_ID>
```

실제 지원 여부와 정확한 옵션은 사용 중인 OpenStack Release, Cinder Driver 및 Backend 설정에 따라 달라진다. 운영 Volume에 적용하기 전 다음 사항을 확인해야 한다.

- Cinder가 인식하는 현재 Backend와 Pool
- Source/Target Volume Type과 Backend 설정
- 해당 Ceph RBD Driver가 지원하는 Migration 방식
- Attached Volume의 Online Migration 지원 여부
- Nova, Cinder, Libvirt의 Attachment 상태
- 실패 시 Rollback과 DB 정합성 복구 절차

> OpenStack이 관리하는 Volume은 Ceph RBD 계층만 보고 직접 조작하지 않는다. Ceph 데이터와 OpenStack Control Plane Metadata를 함께 일관되게 유지하는 것이 중요하다.

---

## 11. RBD Live Migration과 Recovery/Backfill의 차이

두 기능 모두 데이터를 이동하지만 목적과 동작 계층이 다르다.

| 구분 | RBD Live Migration | Recovery / Backfill / Rebalancing |
|---|---|---|
| 대상 | 특정 RBD Image | Ceph Object와 PG |
| 목적 | Pool, Image Format 또는 Layout 변경 | 장애 복구와 CRUSH 배치 균형 회복 |
| 실행 계층 | RBD Image 계층 | RADOS/OSD 계층 |
| 대표 원인 | 운영자가 Image 이동을 계획 | OSD 장애, OSD 추가·제거, CRUSH 변경 |
| 결과 | 새로운 Target RBD Image로 전환 | Object가 적절한 OSD로 재배치됨 |

OSD 장애로 Recovery/Backfill이 발생했다고 해서 RBD Image 자체가 다른 Pool로 Migration되는 것은 아니다. 반대로 RBD Live Migration은 특정 Image를 대상으로 하며 Cluster 전체의 OSD 배치를 재조정하는 기능이 아니다.

---

## 12. 최종 점검표

### Migration 전

- [ ] Source/Target Pool과 Image 이름 확인
- [ ] Source Image의 Client와 Watcher 확인
- [ ] Client 중지 및 작업 영향도 확인
- [ ] Source의 Snapshot/Clone/Parent 관계 확인
- [ ] Target Pool 용량과 CRUSH 배치 OSD 상태 확인
- [ ] Backup 또는 복구 절차 준비
- [ ] OpenStack/Cinder 관리 Volume인지 확인

### Migration 중단 시

- [ ] 추가 변경 작업 중지
- [ ] 오류 메시지와 실행 시각 기록
- [ ] `rbd status`로 Migration State 확인
- [ ] `ceph -s`와 `ceph health detail` 확인
- [ ] Source가 RBD Trash에 존재하는지 확인
- [ ] Target Image와 Watcher 확인
- [ ] Target Pool 및 OSD 용량 확인
- [ ] Ceph와 Client Log 확인
- [ ] 현재 Client가 사용하는 Image 확인

### Commit 또는 Abort 전

- [ ] Target에서 발생 중인 I/O 확인
- [ ] Client와 Application 상태 확인
- [ ] `executed` 상태 확인 후 Commit 결정
- [ ] Abort 시 Target이 삭제된다는 점 확인
- [ ] Snapshot/Clone과 OpenStack Metadata 영향 확인
- [ ] 결정 근거와 작업 결과 기록

---

## 13. 정리

RBD Live Migration의 핵심 흐름은 다음과 같다.

1. `Prepare`에서 Target을 만들고 Source와 연결한다.
2. Client가 새 Target Image를 사용하도록 전환한다.
3. `Execute`에서 Source의 초기화된 Block을 Target으로 Deep Copy한다.
4. 정상 완료 후 `Commit`하여 Target을 최종 Image로 확정한다.
5. Migration을 되돌려야 한다면 영향도를 확인한 뒤 `Abort`한다.

Migration 중단 시 가장 중요한 것은 곧바로 명령을 다시 실행하거나 Abort하는 것이 아니라, **Migration State, Ceph Cluster Health, Source/Target Image, Client I/O와 상위 Control Plane 상태를 함께 확인하는 것**이다.

특히 다음 세 가지를 기억해야 한다.

- `Live Migration`이어도 Prepare 전에는 Source를 사용하는 Client를 중지해야 한다.
- `Abort`는 일시 정지가 아니라 Target을 삭제하고 Source 접근을 복구하는 Rollback이다.
- OpenStack Cinder가 관리하는 Volume은 Backend RBD만 직접 변경하면 Control Plane 정보와 불일치할 수 있다.

## 참고자료

- [Ceph 공식 문서: RBD Image Live-Migration](https://docs.ceph.com/en/latest/rbd/rbd-live-migration/)
- [Ceph 공식 문서: rbd 명령어](https://docs.ceph.com/en/latest/man/8/rbd/)
- [OpenStackClient 공식 문서: volume migrate](https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/volume.html)

