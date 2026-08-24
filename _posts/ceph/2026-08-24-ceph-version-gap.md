---
layout: post
title: "ceph version과 ceph --version이 다른 이유: cephadm, ceph-common, Container와 Ubuntu APT"
date: 2026-08-24 14:50:00 +0900
categories: [Ceph, Ubuntu, Infrastructure]
tags: [Ceph, Cephadm, Ceph-Common, Ubuntu, APT, Repository, Podman, Container, Squid, Tentacle]
published: true
---

Ceph 버전을 확인하면서 다음과 같은 결과를 발견했다.

```bash
ceph version
```

```text
ceph version 20.2.3 (...) tentacle
```

그런데:

```bash
ceph --version
```

에서는:

```text
ceph version 19.2.3 (...) squid
```

가 출력됐다.

처음에는 동일한 Ceph Cluster 안에서 19.x와 20.x가 섞여 있는 것인지 의심했다.

하지만 실제로 확인해보니 **두 명령이 확인하는 대상 자체가 달랐다.**

이번 글에서는 실제 서버에서 명령을 하나씩 실행하면서 원인을 확인한 과정을 정리한다.

---

# 1. 먼저 두 명령의 의미가 다르다

가장 먼저 알아야 할 것은:

```bash
ceph version
```

과:

```bash
ceph --version
```

이 동일한 동작을 하는 명령이 아니라는 점이다.

Ceph 공식 Command API에서 `ceph version`은 **Monitor daemon version을 보여주는 Cluster command**로 정의되어 있고, `ceph --version`은 현재 실행 중인 `ceph` CLI 프로그램의 version을 출력한다.

즉:

```text
ceph version
     |
     v
MON에 명령 전달
     |
     v
MON daemon version 확인


ceph --version
     |
     v
현재 실행하는 ceph CLI 자체
     |
     v
Local binary version 확인
```

이다.

이 차이가 이번 문제의 핵심이었다.

---

# 2. 실제 결과 확인

```bash
ceph version
```

결과:

```text
ceph version 20.2.3 (...) tentacle
```

반면:

```bash
ceph --version
```

결과:

```text
ceph version 19.2.3 (...) squid
```

> **📸 캡처 1 - `ceph version`과 `ceph --version` 비교**
>
> 두 명령을 연속해서 실행한 화면을 한 장에 캡처한다.
>
> ```bash
> ceph version
> ceph --version
> ```
>
> 강조할 부분:
>
> ```text
> 20.2.3 Tentacle
> 19.2.3 Squid
> ```

---

# 3. Squid와 Tentacle은 무엇인가?

Ceph는 각 주요 Release Series마다 코드명을 사용한다.

현재 확인한 버전은:

```text
19.2.x → Squid
20.2.x → Tentacle
```

이다.

따라서:

```text
19.2.3 Squid
```

와:

```text
20.2.3 Tentacle
```

은 단순히 patch version 하나가 다른 것이 아니라 **서로 다른 Ceph Release Series**다.

그래서 처음에는 "왜 관리 CLI가 Cluster보다 한 release 낮은가?"를 확인할 필요가 있었다.

---

# 4. 현재 실행되는 `ceph`는 어디에 있는가?

먼저:

```bash
which ceph
```

를 실행했다.

결과:

```text
/usr/bin/ceph
```

그리고:

```bash
type -a ceph
```

결과:

```text
ceph is /usr/bin/ceph
ceph is /bin/ceph
```

였다.

즉 일반 Shell에서:

```bash
ceph ...
```

라고 실행하면 Host Ubuntu에 설치되어 있는 `/usr/bin/ceph`를 사용한다.

> **📸 캡처 2 - Host에서 실행하는 Ceph CLI 위치**
>
> ```bash
> which ceph
> type -a ceph
> ```

---

# 5. `/usr/bin/ceph`는 어디서 왔을까?

Ubuntu에서는 `ceph-common` Package가 여러 Ceph Client/Administration Utility를 제공한다.

현재 설치 버전을 확인했다.

```bash
dpkg -l | grep -E 'ceph-common|cephadm'
```

결과:

```text
ii  ceph-common          19.2.3-0ubuntu0.24.04.3
ii  cephadm              19.2.3-0ubuntu0.24.04.3
ii  python3-ceph-common  19.2.3-0ubuntu0.24.04.3
```

따라서 Host에 설치된:

```text
/usr/bin/ceph
```

는 `ceph-common 19.2.3`에 의해 제공되고 있었다.

즉:

```text
Ubuntu Host
   |
   +-- ceph-common 19.2.3 Squid
          |
          +-- /usr/bin/ceph
          +-- rados
          +-- rbd
          +-- mount.ceph
          +-- 기타 Ceph Client Utility
```

구조다.

> **📸 캡처 3 - ceph-common 및 cephadm Package Version**
>
> ```bash
> dpkg -l | grep -E 'ceph-common|cephadm'
> ```

---

# 6. 그렇다면 실제 Cluster daemon 버전은?

Cluster의 daemon별 version을 확인하기 위해 다음 명령을 사용했다.

```bash
ceph versions
```

초기 확인 시 결과는:

```text
MON  → 20.2.3 × 3
MGR  → 20.2.3 × 3
OSD  → 20.2.3 × 19
MDS  → 20.2.3 × 5
RGW  → 20.2.3 × 3
```

전체:

```text
20.2.3 Tentacle → 33 daemons
```

였다.

즉 실제 Cluster는 당시 모든 주요 Ceph daemon이 **20.2.3 Tentacle**이었다.

> **📸 캡처 4 - 실제 Cluster daemon Version**
>
> ```bash
> ceph versions
> ```
>
> 초기 20.2.3 상태 캡처가 있다면 사용한다.
>
> 현재는 20.2.4 Upgrade를 진행 중이므로 새로 실행하면 결과가 달라질 수 있다.

---

# 7. `ceph version`과 `ceph versions`도 다르다

두 명령도 구분할 필요가 있다.

```bash
ceph version
```

Ceph 공식 Command API 기준으로 Monitor daemon version을 표시한다.

반면:

```bash
ceph versions
```

은 실행 중인 Ceph daemon의 version 분포를 종류별로 보여준다.

따라서 실제 Cluster 전체의 version 분포를 확인하려면 `ceph versions`가 훨씬 유용하다.

예를 들어 Rolling Upgrade 중에는:

```text
mgr:
  20.2.3 → 1
  20.2.4 → 2

osd:
  20.2.3 → 19
```

처럼 서로 다른 version이 동시에 나타날 수 있다.

---

# 8. 실제 daemon은 Ubuntu Package로 실행되는 게 아니었다

여기서 가장 중요한 사실을 확인했다.

```bash
ceph orch ps
```

출력에서 Ceph daemon들은:

```text
VERSION: 20.2.3
IMAGE ID: f8a4970c6385
```

를 사용하고 있었다.

Image ID를 실제 Host에서 조회했다.

```bash
podman images | grep f8a4970c6385
```

결과:

```text
quay.io/ceph/ceph   v20.2.3   f8a4970c6385
```

즉 MON, MGR, OSD 등의 실제 Ceph daemon은 Host의 `ceph-common` Package로 실행되는 것이 아니라:

```text
quay.io/ceph/ceph:v20.2.3
```

Container Image로 실행되고 있었다.

> **📸 캡처 5 - Ceph Container Image 확인**
>
> ```bash
> podman images | grep f8a4970c6385
> ```
>
> 보여줄 부분:
>
> ```text
> quay.io/ceph/ceph
> v20.2.3
> ```

---

# 9. Podman이란?

Podman은 Container Runtime/Engine이다.

Docker와 유사하게 Container Image를 내려받고 Container를 실행한다.

현재 구조는:

```text
Ubuntu Host
   |
   v
Podman
   |
   v
quay.io/ceph/ceph:v20.2.3
   |
   +-- MON Container
   +-- MGR Container
   +-- OSD Container
   +-- MDS Container
   +-- RGW Container
```

이다.

따라서 Host의 APT Package와 Ceph daemon Runtime은 서로 독립적으로 version이 달라질 수 있다.

---

# 10. Package 기반과 Container 기반의 차이

## Package 기반

과거 또는 package 기반 deployment에서는 Host에 직접 daemon package를 설치할 수 있다.

```text
Ubuntu Host
├── ceph-mon
├── ceph-osd
├── ceph-mgr
└── ceph-common
```

이 경우:

```text
Host Package Version
≈
Daemon Runtime Version
```

관계가 매우 직접적이다.

## cephadm Container 기반

현재 환경은:

```text
Ubuntu Host
├── ceph-common 19.2.3
├── cephadm 19.2.3
└── Podman
     └── Ceph Container 20.2.x
```

이다.

그래서:

```text
Host Package Version
!=
Cluster Runtime Version
```

이 가능하다.

---

# 11. cephadm은 무엇인가?

`cephadm`은 Ceph daemon을 Container로 배포하고 관리하는 관리 도구다.

Ceph 공식 문서에서는 cephadm을 Ceph Cluster 설치 및 관리를 위한 도구로 설명하며 Podman 또는 Docker 같은 Container Runtime을 사용한다.

cephadm이 관리하는 대표적인 작업은:

- Ceph Cluster bootstrap
- Host 등록
- daemon 배치
- daemon 제거
- Container Image 관리
- Cluster Upgrade
- Orchestrator Backend

이다.

즉:

```text
cephadm
    |
    +-- qcs01에 MON 실행
    +-- qcs05에 OSD 실행
    +-- RGW 배포
    +-- daemon Image 변경
    +-- Cluster Rolling Upgrade
```

같은 역할을 담당한다.

---

# 12. `ceph orch`와 cephadm 관계

다음 명령을 자주 사용한다.

```bash
ceph orch ps
```

여기서 `orch`는 **Orchestrator**다.

개념적으로:

```text
ceph CLI
   |
   v
MGR
   |
   v
Orchestrator API
   |
   v
cephadm backend
   |
   v
Podman
   |
   v
Ceph Container
```

형태로 동작한다.

즉 `ceph orch ps`는 단순히 `podman ps`의 별칭이 아니다.

Ceph 관리 계층을 통해 daemon deployment 정보를 조회한다.

---

# 13. 현재 Ubuntu는 어떤 버전인가?

다음 명령을 사용했다.

```bash
cat /etc/os-release
```

결과:

```text
PRETTY_NAME="Ubuntu 24.04.3 LTS"
VERSION_ID="24.04"
VERSION_CODENAME=noble
UBUNTU_CODENAME=noble
```

현재 qcs01은:

```text
Ubuntu 24.04.3 LTS
Codename: Noble Numbat
```

이다.

Ubuntu release 이름을 정리하면:

```text
Ubuntu 22.04 → Jammy Jellyfish
Ubuntu 24.04 → Noble Numbat
```

이다.

> **📸 캡처 6 - Ubuntu Version**
>
> ```bash
> cat /etc/os-release
> ```

---

# 14. APT와 Repository

Ubuntu에서 package를 설치할 때 주로 사용하는 Package Manager가 APT다.

예를 들어:

```bash
apt install ceph-common
```

을 실행한다고 해서 APT가 인터넷 전체에서 최신 Ceph를 검색하는 것은 아니다.

APT는 **등록되어 있는 Repository만 검색**한다.

```text
apt
 |
 +-- /etc/apt/sources.list
 |
 +-- /etc/apt/sources.list.d/
 |
 v
Configured Repository
 |
 v
Package
```

Repository는 쉽게 말해:

> Ubuntu가 `.deb` Package를 다운로드할 수 있도록 등록해 놓은 Software 저장소

다.

---

# 15. 현재 활성 Repository 확인

실제 서버에서 다음과 같은 Ubuntu Repository가 활성화되어 있었다.

```text
archive.ubuntu.com/ubuntu noble
archive.ubuntu.com/ubuntu noble-updates
security.ubuntu.com/ubuntu noble-security
```

APT 설정에서:

```text
deb ...
```

처럼 `#` 없이 시작하는 line은 활성화된 Repository다.

반면:

```text
# deb ...
```

은 주석 처리된 상태이므로 사용되지 않는다.

---

# 16. 과거 Ceph Reef Repository 흔적

서버에는 다음 파일이 존재했다.

```text
/etc/apt/sources.list.d/download_ceph_com_debian_reef.list
```

내용은:

```text
# deb https://download.ceph.com/debian-reef jammy main
# deb https://download.ceph.com/debian-18.2.6/ noble main
```

였다.

두 line 모두 앞에 `#`이 있으므로 현재는 비활성화되어 있다.

파일에는:

```text
disabled on upgrade to noble
```

이라는 흔적도 있었다.

여기서 확실하게 말할 수 있는 것은:

> 현재 Ceph Reef Repository는 활성화되어 있지 않다.

이다.

반면:

> 과거 Ubuntu 22.04에서 24.04로 실제 release upgrade를 했기 때문에 정확히 이렇게 되었다.

는 해당 파일만으로 100% 확정할 수 없다.

`disabled on upgrade to noble`이라는 흔적으로 볼 때 Ubuntu Upgrade 과정에서 Third-party Repository가 비활성화되었을 가능성이 높지만, 정확한 변경 이력을 알려면 `/var/log/dist-upgrade` 등의 기록을 추가로 확인해야 한다.

> **📸 캡처 7 - 과거 Ceph Repository 흔적**
>
> ```bash
> grep -Rni 'ceph' \
>   /etc/apt/sources.list \
>   /etc/apt/sources.list.d/ 2>/dev/null
> ```
>
> 주의: Repository line만 캡처한다.

---

# 17. 현재 활성화된 Ceph Repository가 있는지 확인

주석 처리된 line까지 모두 출력하면 헷갈릴 수 있다.

실제로 활성화된 Ceph Repository만 확인하려면:

```bash
grep -RhE '^[[:space:]]*deb .*ceph' \
  /etc/apt/sources.list \
  /etc/apt/sources.list.d/ 2>/dev/null
```

현재 구성에서는 별도의 Ceph upstream Repository가 활성화되어 있지 않았다.

즉 현재 `ceph-common`과 `cephadm` Package는 Ceph upstream Repository가 아닌 Ubuntu Repository에서 제공받는다.

> **📸 캡처 8 - 활성 Ceph Repository 확인**
>
> ```bash
> grep -RhE '^[[:space:]]*deb .*ceph' \
>   /etc/apt/sources.list \
>   /etc/apt/sources.list.d/ 2>/dev/null
> ```
>
> 결과가 없다면:
>
> ```text
> 별도의 활성 Ceph APT Repository가 없음
> ```
>
> 을 의미한다.

---

# 18. `apt policy`로 Package 출처 확인

다음 명령을 실행했다.

```bash
apt policy ceph-common cephadm
```

결과:

```text
ceph-common:
  Installed: 19.2.3-0ubuntu0.24.04.3
  Candidate: 19.2.3-0ubuntu0.24.04.3

cephadm:
  Installed: 19.2.3-0ubuntu0.24.04.3
  Candidate: 19.2.3-0ubuntu0.24.04.3
```

그리고 package source는:

```text
archive.ubuntu.com/ubuntu noble-updates
security.ubuntu.com/ubuntu noble-security
```

였다.

Ubuntu Noble 공식 package 정보에서도 Ceph의 현재 package가 `19.2.3-0ubuntu0.24.04.3`으로 제공되고 있다.

> **📸 캡처 9 - Installed / Candidate / Package Source**
>
> ```bash
> apt policy ceph-common cephadm
> ```

---

# 19. Installed와 Candidate의 차이

APT에서 매우 중요한 개념이다.

## Installed

현재 Host에 실제 설치되어 있는 Package version이다.

```text
Installed: 19.2.3
```

## Candidate

현재 등록된 Repository와 APT 정책 기준으로:

> 지금 설치 또는 upgrade한다면 선택할 version

이다.

```text
Candidate: 19.2.3
```

따라서:

```text
Installed = 19.2.3
Candidate = 19.2.3
```

이면 단순히 `apt upgrade`한다고 Tentacle 20.x가 되는 것이 아니다.

현재 Ubuntu Noble Repository에서 APT가 선택할 수 있는 version 자체가 19.2.3이기 때문이다.

---

# 20. apt update와 apt upgrade도 다르다

## `apt update`

```bash
apt update
```

는 Repository Package Index를 최신 상태로 갱신한다.

실제 Package를 바로 업그레이드하는 명령은 아니다.

```text
Repository
    |
    v
Package metadata
    |
    v
Local APT cache
```

## `apt upgrade`

```bash
apt upgrade
```

는 Candidate가 현재 Installed version보다 높은 Package를 실제로 upgrade한다.

따라서 Repository에 Ceph 20.x가 없다면:

```text
apt update
→ Repository 정보를 최신화

Candidate
→ 여전히 19.2.3

apt upgrade
→ Ceph 20.x로 올라가지 않음
```

이 된다.

---

# 21. 왜 Ubuntu Repository와 Ceph 최신 Release가 다를까?

여기서 **Upstream Project와 Linux Distribution**의 차이를 알아야 한다.

Ceph Project에서는 자체적으로:

```text
Ceph Upstream
├── Squid 19.x
└── Tentacle 20.x
```

를 release한다.

하지만 Ubuntu는 upstream software가 release되는 즉시 모든 version을 기존 Ubuntu LTS repository에 그대로 넣는 것이 아니다.

Ubuntu Distribution에서 package를 별도로 빌드하고 테스트하며 해당 Ubuntu release에서 제공할 version을 관리한다.

그래서:

```text
Ceph upstream 최신 release
          !=
Ubuntu Noble package version
```

일 수 있다.

현재 운영 환경이 정확히 그런 경우다.

---

# 22. 그렇다면 왜 Cluster는 Tentacle인가?

실제 daemon은 APT가 아니라 Container Image에서 공급되기 때문이다.

```text
Host 관리 도구

Ubuntu Repository
      |
      +-- ceph-common 19.2.3
      +-- cephadm     19.2.3


Cluster Runtime

Container Registry
      |
      +-- quay.io/ceph/ceph:v20.2.x
```

즉 Software 공급 경로가 다르다.

---

# 23. cephadm shell

그렇다면 한 가지 문제가 생긴다.

Cluster는 Tentacle인데 Host의 `/usr/bin/ceph`는 Squid다.

기본 명령은 정상적으로 동작하지만, Tentacle에 새로 추가된 기능이나 CLI Option을 Squid CLI가 지원하지 못할 가능성은 존재한다.

cephadm에서는 이를 해결할 수 있는 방법으로 `cephadm shell`을 제공한다.

Ceph 공식 문서에서도 cephadm은 Host에 Ceph Package가 반드시 설치되어 있어야 하는 것은 아니며:

```bash
cephadm shell
```

을 통해 Ceph Package가 설치된 Container Shell을 사용할 수 있다고 설명한다.

명령 하나만 실행할 수도 있다.

```bash
cephadm shell -- ceph -s
```

---

# 24. 실제로 확인해볼 명령

다음은 **조회 명령**이다.

```bash
cephadm shell -- ceph --version
```

실행 후 Host CLI와 비교한다.

```bash
ceph --version
cephadm shell -- ceph --version
```

> **📸 캡처 10 - Host CLI와 cephadm shell CLI 비교**
>
> ```bash
> ceph --version
> cephadm shell -- ceph --version
> ```
>
> 이 부분은 실제 실행 결과를 캡처한 뒤 글에 추가한다.
>
> **주의:** `cephadm shell`이 어떤 Image를 선택하는지는 환경에 따라 달라질 수 있으므로 결과를 보기 전에 "무조건 Cluster와 동일 버전"이라고 단정하지 않는다. 실제 출력으로 확인한다.

---

# 25. 현재 운영 방식에 대한 판단

이번 확인을 통해 처음 생각했던:

```text
Cluster 20.x
Host CLI 19.x

→ 잘못됨
→ 즉시 Host Package도 20.x로 업데이트
```

라는 단순한 판단은 적절하지 않다는 것을 알 수 있었다.

현재 구조는:

```text
Ubuntu 24.04 Noble
 |
 +-- Ubuntu Repository
 |      |
 |      +-- ceph-common 19.2.3
 |      +-- cephadm     19.2.3
 |
 +-- Podman
        |
        +-- Ceph 20.x Tentacle Containers
```

다.

현재 기본적인 Cluster 관리 기능은 정상 동작하고 있으며, 필요한 경우 `cephadm shell`을 통해 Container 환경의 Ceph CLI를 사용할 수 있다.

따라서 현재 운영 정책은:

```text
1. Ubuntu Noble Repository 유지

2. ceph-common / cephadm 19.x 유지

3. 실제 Ceph daemon은 cephadm Container로 Tentacle 운영

4. Cluster version에 맞는 CLI가 필요할 경우
   cephadm shell 활용

5. 추후 Ubuntu Noble Repository에서
   적절한 Tentacle-compatible Package가 제공될 경우
   Host Package Update 검토
```

방향으로 이해할 수 있다.

---

# 26. 핵심 구조 다시 정리

```text
                Ubuntu 24.04 Noble
                        |
          +-------------+-------------+
          |                           |
          v                           v
    APT Package                  Container Runtime
          |                           |
          v                           v
 ceph-common 19.2.3                 Podman
 cephadm     19.2.3                   |
          |                           v
          |                 quay.io/ceph/ceph
          |                         20.x
          |                       Tentacle
          |                           |
          |                +----------+---------+
          |                |     |     |    |   |
          |               MON   MGR   OSD  MDS RGW
          |
          v
   /usr/bin/ceph

ceph --version
→ Host CLI 19.2.3

ceph version
→ Monitor daemon version

ceph versions
→ 실제 Cluster daemon version 분포
```

---

# 27. 이번 문제에서 얻은 결론

`ceph version`과 `ceph --version`의 결과가 다르다고 해서 바로:

```text
Ceph Cluster가 꼬였다
```

고 판단해서는 안 된다.

먼저 다음을 구분해야 한다.

```text
1. Host에 설치된 CLI Version인가?

2. Cluster daemon Runtime Version인가?

3. Package 기반인가?

4. Container 기반인가?

5. Package는 어느 Repository에서 공급되는가?

6. 현재 실제 기능 문제가 존재하는가?
```

현재 환경에서는 Host의 `ceph-common`과 `cephadm`은 Ubuntu Noble Repository의 Squid 19.2.3이고, 실제 Ceph daemon은 Podman Container의 Tentacle 20.x였다.

즉 **Host Package Layer와 Cluster Runtime Layer가 분리되어 있기 때문에 서로 다른 version이 나타난 것**이었다.

다음 글에서는 이 구조에서 Ceph 20.2.3 → 20.2.4 Rolling Upgrade가 어떻게 진행되는지 실제 운영 환경에서 확인한 과정을 정리한다.