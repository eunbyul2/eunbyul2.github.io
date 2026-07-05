---
layout: post
title: "Object Storage와 S3 기본 원리"
date: 2026-07-01 19:00:00 +0900
categories: [Storage, S3, ObjectStorage]
tags: [ObjectStorage, S3, Bucket, Object, Key, Prefix, Metadata, ETag, MultipartUpload, Versioning]
published: true
---

## 1. 스토리지를 먼저 분류해야 하는 이유

S3와 Object Storage를 이해하려면 먼저 “스토리지가 데이터를 어떤 단위로 제공하는가”를 구분해야 한다. 같은 저장소라고 해도 VM 디스크처럼 보이는 저장소, 리눅스 디렉터리처럼 보이는 저장소, HTTP API로 파일을 업로드하는 저장소는 내부 구조와 사용 방식이 다르다.

Kubernetes에서 PVC를 붙일 때 사용하는 볼륨, NFS 공유 디렉터리, Ceph RBD, CephFS, Ceph RGW, MinIO, RustFS는 모두 “데이터를 저장한다”는 점에서는 같지만, 애플리케이션이 바라보는 인터페이스가 다르다. 이 인터페이스를 구분하지 못하면 왜 S3에는 폴더가 없다고 하는지, 왜 Object를 수정하지 않고 다시 업로드한다고 하는지, 왜 대용량 업로드에 Multipart Upload가 필요한지 이해하기 어렵다.

스토리지는 크게 다음 세 가지 관점으로 나눠서 이해하는 것이 좋다.

```text
Block Storage  → 디스크처럼 보이는 저장소
File Storage   → 디렉터리/파일처럼 보이는 저장소
Object Storage → HTTP API로 Object를 저장하는 저장소
```

| 구분 | 애플리케이션이 보는 단위 | 대표 예시 | 주 사용처 |
|---|---|---|---|
| Block Storage | Block Device | Ceph RBD, AWS EBS, iSCSI, Fibre Channel | VM 디스크, DB 디스크, Kubernetes PVC |
| File Storage | File / Directory | NFS, SMB, CephFS, EFS | 공유 디렉터리, 홈 디렉터리, 파일 서버 |
| Object Storage | Object | AWS S3, Ceph RGW, MinIO, RustFS, OpenStack Swift | 백업, 로그, 이미지, 문서, 데이터레이크, AI 데이터셋 |

Object Storage는 Block Storage나 File Storage를 완전히 대체하는 기술이 아니다. 목적이 다르다. DB 데이터파일이나 VM 루트 디스크는 Block Storage가 적합하고, 여러 서버가 같은 디렉터리를 마운트해야 하면 File Storage가 적합하다. 반대로 수많은 이미지, 로그, 백업 파일, 모델 파일, 데이터셋을 HTTP API로 저장하고 관리하려면 Object Storage가 적합하다.

## 2. Block Storage

Block Storage는 저장 공간을 일정 크기의 블록 단위로 제공한다. 운영체제는 이 블록 장치를 일반 디스크처럼 인식하고, 그 위에 ext4, xfs 같은 파일시스템을 올려 사용한다.

```text
Application
  ↓
File System(ext4, xfs 등)
  ↓
Block Device(/dev/vdb, /dev/rbd0 등)
  ↓
Storage Backend
```

Block Storage의 핵심은 “운영체제 입장에서 디스크처럼 보인다”는 점이다. 애플리케이션은 보통 직접 블록을 다루지 않고, 운영체제 파일시스템을 통해 파일을 읽고 쓴다. 하지만 저장소가 제공하는 추상화 단위는 파일이 아니라 블록이다.

### 2-1) Block Storage가 적합한 경우

Block Storage는 낮은 지연 시간과 랜덤 I/O가 중요한 환경에 적합하다.

```text
- VM 루트 디스크
- 데이터베이스 디스크
- Kubernetes PVC
- 고성능 랜덤 읽기/쓰기 워크로드
- 파일시스템을 직접 선택하고 싶은 환경
```

Kubernetes에서 StatefulSet이 PostgreSQL, MySQL, Redis, Elasticsearch 같은 워크로드에 PVC를 붙일 때는 대부분 Block Storage 계열을 사용한다. Ceph에서는 RBD가 이 역할을 한다.

### 2-2) Block Storage의 한계

Block Storage는 디스크처럼 쓰기에는 좋지만, S3 같은 대규모 HTTP 기반 파일 저장소를 만들기에는 적합하지 않다.

```text
- HTTP API 기반 접근이 기본이 아니다.
- 객체별 Metadata, Policy, Lifecycle 관리가 어렵다.
- 수십억 개의 비정형 데이터를 이름 기반으로 관리하는 데 적합하지 않다.
- 여러 애플리케이션이 인터넷을 통해 직접 접근하는 저장소로 쓰기 어렵다.
```

즉 Block Storage는 “디스크가 필요한 워크로드”에 적합하고, Object Storage는 “API로 많은 파일성 데이터를 저장하고 관리하는 워크로드”에 적합하다.

## 3. File Storage

File Storage는 파일과 디렉터리 구조를 제공한다. 사용자는 경로를 기준으로 데이터를 다룬다.

```text
/shared
  ├── project-a
  │   └── report.pdf
  └── logs
      └── app.log
```

NFS, SMB, CephFS가 대표적인 File Storage이다. 여러 서버가 같은 경로를 마운트해서 파일을 공유해야 할 때 유용하다.

### 3-1) File Storage의 특징

File Storage는 POSIX에 가까운 파일시스템 동작을 제공하는 경우가 많다.

```text
- 디렉터리 구조
- 파일 생성/수정/삭제
- rename
- chmod/chown
- file lock
- open/read/write/close
```

물론 모든 File Storage가 완전한 POSIX 의미론을 동일하게 제공하는 것은 아니지만, 사용자는 일반적으로 “파일시스템처럼” 접근한다.

### 3-2) File Storage가 적합한 경우

```text
- 여러 서버가 같은 디렉터리를 공유해야 하는 경우
- 기존 애플리케이션이 파일 경로 기반으로 동작하는 경우
- 홈 디렉터리, 공유 폴더, 사내 파일 서버
- POSIX 파일 연산이 필요한 경우
```

### 3-3) File Storage의 한계

File Storage는 대규모 객체 저장소 관점에서는 몇 가지 한계가 있다.

```text
- HTTP API 기반 대규모 외부 접근에 최적화되어 있지 않다.
- 버킷 단위 정책, 객체 단위 Metadata, Lifecycle 같은 S3 기능이 기본 모델이 아니다.
- 수십억 개 Object를 Prefix 기반으로 API 조회하는 모델과 다르다.
- 클라우드 네이티브 애플리케이션에서 직접 API로 쓰기에는 S3가 더 일반적이다.
```

따라서 사용자 업로드 파일, 로그 아카이브, 백업, AI 데이터셋처럼 애플리케이션이 API로 저장하고 조회하는 데이터에는 Object Storage가 더 자연스럽다.

## 4. Object Storage

Object Storage는 데이터를 파일 경로나 블록 주소가 아니라 Object 단위로 저장하는 방식이다. Object는 일반적으로 다음 세 요소로 이해할 수 있다.

```text
Object = Data + Metadata + Key
```

- Data: 실제 파일 내용
- Metadata: Object에 붙는 부가 정보
- Key: Bucket 안에서 Object를 식별하는 이름

Object Storage는 대부분 HTTP/HTTPS 기반 API를 통해 접근한다.

```text
Application / Console / CLI / SDK
  ↓ HTTP(S) REST API
Object Storage Endpoint
  ↓
Distributed Storage Backend
```

AWS S3, Ceph RGW, MinIO, RustFS는 모두 이 모델에 가깝다. Ceph에서는 RGW가 S3 호환 API를 제공하고, 내부적으로는 RADOS에 데이터를 저장한다. Ceph 공식 문서는 Ceph Object Gateway를 librados 위에 구축된 Object Storage 인터페이스이며, 애플리케이션과 Ceph Storage Cluster 사이에 RESTful Gateway를 제공한다고 설명한다.

참고:

- https://docs.ceph.com/en/latest/radosgw/
- https://docs.ceph.com/en/latest/radosgw/s3/

## 5. Object Storage가 등장한 이유

Object Storage는 단순히 “파일을 저장하는 또 다른 방식”이 아니다. 클라우드와 대규모 웹 서비스가 성장하면서 기존 저장 방식으로 해결하기 어려운 문제가 생겼고, 그 문제를 해결하기 위해 Object Storage가 널리 사용되기 시작했다.

대표적인 배경은 다음과 같다.

```text
- 이미지, 영상, 로그, 백업, AI 데이터셋 같은 비정형 데이터 폭증
- 인터넷을 통한 API 기반 저장소 필요
- 수평 확장 가능한 저장소 필요
- 파일 경로보다 객체 단위 Metadata와 Policy가 중요해짐
- 오래된 데이터 자동 삭제/이동 필요
- 사용자, 서비스, 고객사 단위 접근 제어 필요
- 대규모 데이터레이크와 분석 워크로드 증가
```

기존 파일시스템은 디렉터리 트리를 중심으로 설계되었다. 하지만 클라우드 환경에서는 수많은 서비스가 네트워크를 통해 데이터를 저장하고 조회해야 한다. 이때 각 애플리케이션이 NFS를 마운트하거나 블록 디스크를 직접 다루는 방식은 확장성과 운영 측면에서 불리할 수 있다.

Object Storage는 이 문제를 HTTP API 모델로 풀었다.

```text
파일 열기(open) → 쓰기(write) → 닫기(close)
```

대신:

```text
PUT Object
GET Object
DELETE Object
LIST Objects
```

이렇게 API 단위로 데이터를 다룬다.

## 6. Object Storage와 POSIX 파일시스템의 차이

Object Storage를 처음 배울 때 가장 헷갈리는 부분은 “왜 폴더가 없다고 하는가”, “왜 파일 일부 수정이 어렵다고 하는가”, “왜 rename이 없다고 하는가”이다. 이는 Object Storage가 POSIX 파일시스템 모델이 아니기 때문이다.

File Storage에서는 다음과 같은 연산이 자연스럽다.

```text
open("/data/report.txt")
write(...)
rename("/data/report.txt", "/archive/report.txt")
chmod(...)
lock(...)
```

S3에서는 이런 방식이 아니다.

```text
PUT /bucket/key
GET /bucket/key
DELETE /bucket/key
COPY /bucket/source-key → bucket/destination-key
```

S3에는 일반적인 파일시스템의 rename API가 없다. 콘솔에서 “이동” 기능을 제공하더라도 내부적으로는 보통 CopyObject 후 DeleteObject 방식으로 처리한다.

```text
Move Object
  ↓
CopyObject
  ↓
DeleteObject
```

이 차이를 이해해야 S3 Console을 설계할 때 사용자가 보는 “폴더 이동”, “파일명 변경” 기능을 내부적으로 어떻게 구현해야 하는지 판단할 수 있다.

## 7. S3란 무엇인가

S3는 Amazon Simple Storage Service의 약자이다. AWS가 제공하는 Object Storage 서비스이며, 현재는 Object Storage API의 사실상 표준처럼 사용된다. 많은 제품이 “S3 호환”을 표방한다.

```text
S3 API 호환 제품 예시
- AWS S3
- Ceph RGW
- MinIO
- RustFS
- OpenStack Swift 일부 호환 계층
```

“S3 호환”은 AWS S3와 동일하거나 유사한 API 방식으로 Bucket과 Object를 다룰 수 있다는 뜻이다. 예를 들어 AWS CLI에서 endpoint만 바꾸면 AWS S3가 아닌 Ceph RGW나 RustFS에도 접근할 수 있다.

```bash
aws --endpoint-url https://s3.example.com s3 ls
```

Ceph 문서도 Ceph Object Gateway가 Amazon S3 RESTful API의 큰 부분과 호환되는 인터페이스를 제공한다고 설명한다.

참고:

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html
- https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html
- https://docs.ceph.com/en/latest/radosgw/s3/

## 8. S3의 기본 구성 요소

S3를 이해할 때 가장 먼저 외워야 하는 기본 구성은 다음이다.

```text
Bucket
  └── Object
        ├── Key
        ├── Data
        ├── Metadata
        ├── ETag
        ├── Version ID
        └── Storage Class
```

S3 Console에서 사용자가 보는 “파일 목록”은 실제로는 Bucket 안의 Object Key 목록이다. 사용자가 보는 “폴더”는 실제 디렉터리가 아니라 Prefix와 Delimiter를 이용해 UI에서 폴더처럼 표현한 것이다.

## 9. Bucket

Bucket은 Object를 담는 최상위 컨테이너이다.

```text
bucket-a
bucket-b
bucket-c
```

Bucket은 일반적인 폴더와 다르다. Bucket 안에 또 다른 Bucket을 만들 수 없다. Bucket은 S3에서 정책, 버전 관리, 라이프사이클, CORS, 기본 암호화, Object Lock, Replication, Event Notification 같은 설정의 기준이 된다.

### 9-1) Bucket이 필요한 이유

Bucket은 단순한 “파일 담는 그릇”이 아니라 관리 경계이다. 버킷을 기준으로 다음을 설정할 수 있다.

```text
- 접근 정책
- 버전 관리
- 라이프사이클
- 이벤트 알림
- 복제
- 암호화
- Object Lock
- CORS
- 정적 웹 호스팅
```

즉 Bucket은 Object를 묶는 논리적 단위이자 운영 정책을 적용하는 단위이다.

### 9-2) Bucket 설계 기준

실무에서는 보통 프로젝트, 서비스, 고객사, 데이터 유형, 보안 등급에 따라 Bucket을 나눈다.

```text
qcs-backup
qcs-ai-dataset
qcs-log-archive
customer-a-data
customer-b-data
model-registry
```

설계를 잘못하면 나중에 권한 정책과 라이프사이클 정책이 복잡해진다.

예를 들어 고객사별 접근 제어가 중요한데 하나의 Bucket에 모든 고객 데이터를 섞어 넣으면 Bucket Policy만으로 분리하기 어렵고, Prefix 기반 정책을 정교하게 작성해야 한다. 반대로 고객사별 Bucket을 나누면 정책과 쿼터를 Bucket 단위로 관리하기 쉬워진다.

### 9-3) Bucket Naming Rule

AWS S3의 일반 목적 버킷 이름은 3~63자여야 하고, 소문자/숫자/점/하이픈을 사용할 수 있으며, 문자나 숫자로 시작하고 끝나야 한다. IP 주소 형식처럼 보이는 이름도 허용되지 않는다.

참고:

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html

운영에서는 AWS 규칙을 그대로 따르는 것이 좋다. Ceph나 RustFS에서 일부 규칙이 다르게 동작할 수 있더라도, S3 호환성과 DNS 기반 접근을 고려하면 보수적으로 설계하는 편이 안전하다.

권장 예시:

```text
qcs-backup-prod
qcs-ai-dataset
customer-a-archive
```

비권장 예시:

```text
QCS_Backup       # 대문자/언더스코어
192.168.0.1      # IP 주소 형식
my..bucket       # 연속 점
-bucket          # 하이픈 시작
bucket-          # 하이픈 끝
```

### 9-4) Bucket과 DNS

Bucket 이름은 URL의 일부가 될 수 있다.

Virtual-hosted-style에서는 Bucket 이름이 서브도메인처럼 들어간다.

```text
https://my-bucket.s3.example.com/object.txt
```

이 구조 때문에 Bucket 이름은 DNS 호환 규칙을 따르는 것이 중요하다. 회사 환경에서 `*.s3.quantumcns.ai` 같은 와일드카드 도메인을 구성하는 이유도 Bucket 이름을 서브도메인으로 받기 위해서이다.

### 9-5) Bucket 생성 과정

콘솔에서 “Create Bucket” 버튼을 누르면 내부적으로는 CreateBucket API가 호출된다.

```text
사용자
  ↓
Console: Bucket 이름 입력
  ↓
Backend 또는 SDK
  ↓
CreateBucket API
  ↓
S3 Endpoint
  ↓
Bucket Metadata 생성
```

Ceph RGW에서는 Bucket 생성 요청이 RGW를 통해 들어오고, RGW가 내부적으로 RADOS에 필요한 Metadata와 Object 저장 구조를 만든다.

### 9-6) Bucket 삭제 과정

Bucket 삭제는 단순하지 않다. 일반적으로 Bucket 안에 Object가 남아 있으면 삭제가 실패한다.

```text
DeleteBucket
  ↓
Bucket이 비어 있는지 확인
  ↓
비어 있으면 삭제
  ↓
Object가 남아 있으면 실패
```

Versioning이 켜진 Bucket이라면 Delete Marker나 과거 Version까지 고려해야 한다. 콘솔에서 “버킷 강제 삭제” 기능을 만들려면 다음 순서가 필요하다.

```text
1. 모든 Object 목록 조회
2. 모든 Version 목록 조회
3. Delete Marker 조회
4. Object/Version/Delete Marker 삭제
5. Bucket 삭제
```

### 9-7) Bucket 관련 API

대표 API는 다음과 같다.

```text
CreateBucket
DeleteBucket
ListBuckets
HeadBucket
GetBucketLocation
GetBucketPolicy
PutBucketPolicy
GetBucketVersioning
PutBucketVersioning
GetBucketLifecycleConfiguration
PutBucketLifecycleConfiguration
```

콘솔에서 Bucket 상세 화면을 만들 때는 단순히 Bucket 목록만 가져오는 것이 아니라 정책, Versioning, Lifecycle, CORS, Encryption, Replication 등 여러 API를 조합해야 한다.

### 9-8) Bucket Best Practice

```text
- Bucket 이름은 소문자, 숫자, 하이픈 중심으로 작성한다.
- 고객사/프로젝트/환경 기준을 미리 정한다.
- 권한 경계가 다르면 Bucket을 분리한다.
- Lifecycle 정책 기준이 다르면 Bucket 또는 Prefix를 분리한다.
- 공개 Bucket은 기본적으로 금지하고 예외적으로만 허용한다.
- 삭제 방지를 위해 Versioning/Object Lock 사용 여부를 검토한다.
```

## 10. Object

Object는 S3에 저장되는 실제 데이터 단위이다. 사용자가 파일을 업로드하면 S3 내부에서는 Object가 생성된다.

```text
Object
├── Key
├── Data
├── Metadata
├── ETag
├── Version ID
└── Storage Class
```

### 10-1) Object와 파일의 차이

파일시스템에서는 파일이 디렉터리 트리 안에 존재한다.

```text
/home/user/report.pdf
```

S3에서는 Bucket 안에 Key로 저장된다.

```text
bucket: docs
key: home/user/report.pdf
```

겉보기에는 경로처럼 보이지만 실제로는 `home/user/report.pdf`라는 문자열 Key를 가진 하나의 Object이다.

### 10-2) Object는 부분 수정이 어려운가

일반 파일시스템에서는 파일 일부를 열어서 수정할 수 있다. 하지만 S3 Object는 기본적으로 전체 Object 단위로 PUT/GET한다. 일부 Range를 읽을 수는 있지만, 임의 위치만 수정하는 POSIX 파일시스템 방식과는 다르다.

예를 들어 큰 CSV 파일의 중간 한 줄을 바꾸는 작업은 S3에서 자연스럽지 않다. 보통 새 Object를 만들어 다시 업로드하거나, 데이터를 작은 단위로 나누어 저장한다.

### 10-3) Object Rename이 없는 이유

S3에는 일반적인 rename API가 없다. Key가 Object의 식별자이기 때문이다.

```text
기존 Key: logs/a.log
새 Key: logs/archive/a.log
```

이동 또는 이름 변경은 보통 다음처럼 처리한다.

```text
1. CopyObject logs/a.log → logs/archive/a.log
2. DeleteObject logs/a.log
```

콘솔에서 Rename 기능을 제공할 때는 이 내부 동작을 고려해야 한다. 특히 큰 Object를 이동할 때 비용과 시간이 발생할 수 있다.

### 10-4) Object 관련 API

```text
PutObject
GetObject
HeadObject
DeleteObject
CopyObject
ListObjectsV2
ListObjectVersions
RestoreObject
```

콘솔 기능과 매핑하면 다음과 같다.

| 콘솔 기능 | 내부 API |
|---|---|
| 파일 업로드 | PutObject / Multipart Upload |
| 파일 다운로드 | GetObject |
| 파일 삭제 | DeleteObject |
| 파일 복사 | CopyObject |
| 파일 상세 정보 | HeadObject |
| 버전 목록 | ListObjectVersions |

## 11. Key

Key는 Bucket 안에서 Object를 식별하는 이름이다.

```text
images/cat.jpg
logs/2026/07/01/app.log
backup/mysql/full.sql
datasets/project-a/train/0001.parquet
```

S3에서 Key는 경로가 아니라 문자열이다. `/` 문자가 들어갈 수 있기 때문에 폴더처럼 보일 뿐이다.

```text
key = "logs/2026/07/01/app.log"
```

### 11-1) Flat Namespace

S3 Bucket은 디렉터리 트리 구조가 아니라 Flat Namespace에 가깝다.

```text
bucket
  ├── logs/2026/07/01/app.log
  ├── logs/2026/07/02/app.log
  └── images/cat.jpg
```

위 구조에서 `logs`, `2026`, `07`은 실제 디렉터리가 아니다. 콘솔이 Prefix와 Delimiter를 기준으로 폴더처럼 표현하는 것이다.

### 11-2) Key 설계가 중요한 이유

Key 구조는 조회, 권한, Lifecycle, 비용, 운영 자동화에 영향을 준다.

좋은 예:

```text
logs/service-a/2026/07/01/app.log
datasets/project-a/train/0001.parquet
backup/mysql/daily/2026-07-01.sql
models/llm/v1/model.safetensors
```

좋은 Key 구조는 다음에 유리하다.

```text
- Prefix 기반 목록 조회
- Prefix 기반 Lifecycle 정책
- Prefix 기반 권한 정책
- 날짜/서비스/고객사 기준 삭제 자동화
- 로그/백업/데이터셋 관리
```

나쁜 예:

```text
file1
file2
new_file_final_final_2
20260701logservicea
```

운영 기준이 없는 Key는 나중에 정책과 자동화가 어렵다.

### 11-3) Key 설계 패턴

로그:

```text
logs/{service}/{yyyy}/{mm}/{dd}/{filename}
```

백업:

```text
backup/{system}/{type}/{yyyy-mm-dd}/{filename}
```

AI 데이터셋:

```text
datasets/{project}/{split}/{version}/{filename}
```

모델 아티팩트:

```text
models/{project}/{model-name}/{version}/{artifact}
```

## 12. Prefix

Prefix는 Key의 앞부분이다.

```text
logs/2026/07/01/app.log
logs/2026/07/02/app.log
```

위 두 Object의 공통 Prefix는 다음이다.

```text
logs/2026/07/
```

Prefix 조회 예시:

```bash
aws s3api list-objects-v2 \
  --bucket qcs-logs \
  --prefix logs/2026/07/
```

Prefix는 폴더가 아니다. 하지만 S3 Console은 Prefix를 폴더처럼 보여준다.

### 12-1) Prefix와 Delimiter

콘솔에서 폴더처럼 보이게 하려면 보통 Prefix와 Delimiter를 사용한다.

```text
prefix: logs/
delimiter: /
```

그러면 S3 API는 `logs/service-a/`, `logs/service-b/` 같은 CommonPrefixes를 반환할 수 있다. 콘솔은 이 값을 폴더처럼 렌더링한다.

### 12-2) Prefix 기반 운영

Prefix는 다음 기능에서 자주 사용된다.

```text
- Lifecycle Rule
- IAM Policy Resource 조건
- ListObjectsV2
- Batch Delete
- 로그 보관 정책
- 데이터셋 분리
```

예를 들어 `logs/` 아래 데이터는 90일 후 삭제하고, `backup/` 아래 데이터는 1년 보관하는 식으로 설계할 수 있다.

## 13. Metadata

Metadata는 Object에 붙는 부가 정보이다.

```text
Object
├── Data
└── Metadata
```

Metadata는 크게 두 종류로 나눌 수 있다.

```text
System-defined metadata
User-defined metadata
```

AWS S3 문서는 Object Metadata를 시스템 정의 Metadata와 사용자 정의 Metadata로 구분한다.

참고:

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingMetadata.html

### 13-1) System-defined Metadata

대표적인 시스템 Metadata는 다음이다.

```text
Content-Type
Content-Length
Last-Modified
ETag
Storage Class
Server-Side Encryption 정보
Version ID
```

Content-Type은 콘솔에서 매우 중요하다. 이미지 파일을 다운로드할지, 브라우저에서 미리보기할지, 텍스트로 열 수 있을지 판단하는 데 쓰인다.

예시:

```text
image/png
application/pdf
text/plain
application/json
application/octet-stream
```

### 13-2) User-defined Metadata

사용자가 직접 붙이는 Metadata이다.

```text
x-amz-meta-owner: data-team
x-amz-meta-project: akashic
x-amz-meta-source: upload-api
```

사용자 Metadata는 검색 DB처럼 복잡한 조회를 위한 기능은 아니다. S3 자체에서 User Metadata 조건으로 자유롭게 검색하는 것은 제한적이다. 따라서 대규모 검색이 필요하면 별도 DB나 인덱스 시스템을 함께 설계해야 한다.

### 13-3) Metadata 수정 시 주의

S3에서 Metadata는 단순히 in-place update되는 개념이 아니다. 많은 경우 CopyObject를 통해 새 Metadata로 교체한다.

```text
기존 Object
  ↓ CopyObject with new metadata
새 Metadata가 적용된 Object
```

따라서 콘솔에서 “Metadata 수정” 기능을 만들 때는 내부적으로 CopyObject가 발생할 수 있음을 고려해야 한다.

## 14. ETag

ETag는 Object의 식별자 또는 변경 여부 확인에 사용되는 값이다. 하지만 ETag를 항상 파일의 MD5라고 이해하면 안 된다.

잘못된 이해:

```text
ETag = 항상 파일 MD5
```

정확한 이해:

```text
ETag는 Object를 식별하는 값으로 사용되지만, Multipart Upload나 암호화가 적용된 경우 단순 MD5가 아닐 수 있다.
```

### 14-1) 단일 파트 업로드와 ETag

암호화나 특수 조건이 없는 단일 파트 업로드에서는 ETag가 MD5처럼 보일 수 있다. 그래서 많은 사용자가 ETag를 파일 무결성 검증용 해시로 오해한다.

### 14-2) Multipart Upload와 ETag

Multipart Upload를 사용하면 ETag 형식이 달라질 수 있다. 예를 들어 뒤에 `-N`처럼 part 개수를 나타내는 값이 붙을 수 있다.

```text
"d41d8cd98f00b204e9800998ecf8427e-3"
```

이 값은 단순히 전체 파일의 MD5가 아니다.

### 14-3) 콘솔에서 ETag 표시 시 주의

콘솔에서 ETag를 표시할 때 다음처럼 단정하면 안 된다.

```text
ETag = MD5 Checksum
```

대신 다음처럼 표현하는 것이 안전하다.

```text
ETag = Object 식별/캐시 검증에 사용되는 값. 업로드 방식과 암호화 여부에 따라 단순 MD5가 아닐 수 있음.
```

## 15. Multipart Upload

Multipart Upload는 큰 Object를 여러 Part로 나누어 업로드하는 방식이다.

```text
Large File
  ↓
Part 1
Part 2
Part 3
  ↓
Complete Multipart Upload
  ↓
Single Object
```

AWS S3 문서는 Multipart Upload를 큰 Object를 여러 Part로 나누어 업로드할 수 있는 기능으로 설명한다.

참고:

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html

### 15-1) Multipart Upload가 필요한 이유

대용량 파일을 한 번에 업로드하면 다음 문제가 있다.

```text
- 네트워크 오류 시 처음부터 다시 업로드해야 한다.
- 업로드 시간이 길어진다.
- 진행률 표시가 어렵다.
- 병렬 업로드를 활용하기 어렵다.
```

Multipart Upload를 사용하면 실패한 Part만 다시 업로드할 수 있고, 여러 Part를 병렬로 업로드할 수 있다.

### 15-2) Multipart Upload 흐름

```text
1. CreateMultipartUpload
2. UploadPart
3. UploadPart
4. UploadPart
5. CompleteMultipartUpload
```

실패하거나 취소할 경우:

```text
AbortMultipartUpload
```

### 15-3) 콘솔 개발 관점

S3 Console에서 업로드 기능을 만들 때는 파일 크기에 따라 전략을 나누는 것이 좋다.

```text
작은 파일 → PutObject
큰 파일 → Multipart Upload
```

대용량 업로드 화면에는 다음이 필요하다.

```text
- Part 단위 진행률
- 재시도
- 업로드 취소
- 네트워크 오류 처리
- Complete 실패 처리
- 미완료 Multipart Upload 정리
```

### 15-4) 미완료 Multipart Upload

Multipart Upload를 시작한 뒤 Complete하지 않으면 미완료 업로드가 남을 수 있다. 운영 환경에서는 Lifecycle Rule을 통해 오래된 미완료 Multipart Upload를 정리하는 것이 좋다.

## 16. Versioning

Versioning은 같은 Key에 대해 여러 버전을 보존하는 기능이다.

```text
report.pdf
├── version-id: 111
├── version-id: 222
└── version-id: 333
```

AWS S3 문서는 Versioning을 하나의 Bucket 안에서 여러 Object 버전을 보관하는 수단으로 설명한다.

참고:

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html

### 16-1) Versioning이 필요한 이유

```text
- 실수로 덮어쓴 파일 복구
- 실수로 삭제한 파일 복구
- 랜섬웨어 대응
- 감사 추적
- 데이터 보존 정책
```

### 16-2) Delete Marker

Versioning이 켜진 Bucket에서 DeleteObject를 수행하면 실제 데이터를 바로 삭제하지 않고 Delete Marker가 생성될 수 있다.

```text
report.pdf
├── version-id: 111
├── version-id: 222
└── delete-marker: 333
```

사용자가 일반 GET을 하면 삭제된 것처럼 보이지만, 과거 Version은 남아 있을 수 있다.

### 16-3) 콘솔 개발 관점

Versioning이 켜져 있는 Bucket에서는 콘솔이 다음 기능을 제공해야 한다.

```text
- 현재 버전 보기
- 이전 버전 보기
- 특정 버전 다운로드
- 특정 버전 삭제
- Delete Marker 표시
- 복원 기능
```

단순 파일 목록만 보여주면 Versioning의 의미를 제대로 전달하기 어렵다.

## 17. REST API와 S3 API

S3는 HTTP 기반 REST API로 동작한다.

```http
PUT /bucket/key HTTP/1.1
Host: s3.example.com
Authorization: AWS4-HMAC-SHA256 ...
Content-Type: text/plain
```

콘솔에서 버튼을 누르는 행위는 내부적으로 HTTP API 호출로 바뀐다.

| 콘솔 동작 | API |
|---|---|
| 버킷 목록 | ListBuckets |
| 버킷 생성 | CreateBucket |
| 버킷 삭제 | DeleteBucket |
| 오브젝트 목록 | ListObjectsV2 |
| 업로드 | PutObject / Multipart Upload |
| 다운로드 | GetObject |
| 삭제 | DeleteObject |
| 복사 | CopyObject |
| 버전 조회 | ListObjectVersions |

### 17-1) HTTP Method

S3 API는 HTTP Method를 사용한다.

```text
GET     → 조회/다운로드
PUT     → 생성/업로드/설정
POST    → 일부 작업 요청
DELETE  → 삭제
HEAD    → Metadata 조회
OPTIONS → CORS Preflight
```

### 17-2) HEAD 요청

HeadObject는 Object 본문을 다운로드하지 않고 Metadata만 확인할 때 유용하다.

```bash
aws s3api head-object \
  --bucket qcs-data \
  --key reports/a.pdf
```

콘솔에서 파일 크기, Content-Type, ETag, Last-Modified를 표시할 때 사용될 수 있다.

## 18. Endpoint

Endpoint는 S3 API를 호출하는 주소이다.

```text
AWS S3:  https://s3.ap-northeast-2.amazonaws.com
Ceph RGW: https://s3.quantumcns.ai
RustFS:  http://rustfs-test:9000
MinIO:   http://minio.example.com:9000
```

우리 회사 S3 Console은 Ceph와 RustFS를 모두 지원해야 하므로 Endpoint 설정이 핵심이다.

```text
Console
  ↓
Backend Adapter
  ↓ Endpoint 설정에 따라 분기
Ceph RGW 또는 RustFS
```

### 18-1) Endpoint와 Region

AWS S3는 Region 개념이 중요하다.

```text
ap-northeast-2
us-east-1
eu-west-1
```

Ceph RGW나 RustFS에서는 AWS와 동일한 Region 모델이 아닐 수 있다. 하지만 AWS SDK는 Region 값을 요구하는 경우가 있으므로, 사설 S3에서는 보통 임의 Region을 설정하거나 서비스에서 요구하는 값을 사용한다.

## 19. Path-style과 Virtual-hosted-style

S3 URL 방식은 크게 두 가지가 있다.

Path-style:

```text
https://s3.example.com/my-bucket/object.txt
```

Virtual-hosted-style:

```text
https://my-bucket.s3.example.com/object.txt
```

AWS 문서는 path-style과 virtual-hosted-style URL 형식을 구분해 설명하며, virtual-hosted-style에서는 Bucket 이름이 URL의 도메인 일부가 된다고 설명한다.

참고:

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/VirtualHosting.html

### 19-1) 회사 환경과 와일드카드 도메인

`*.s3.quantumcns.ai` 같은 Ingress Host가 있다면 보통 다음 형태를 지원하기 위한 것이다.

```text
https://bucket-a.s3.quantumcns.ai/object.txt
https://bucket-b.s3.quantumcns.ai/object.txt
```

이 구조에서는 TLS 인증서도 와일드카드 인증서가 필요할 수 있다.

```text
*.s3.quantumcns.ai
```

### 19-2) 콘솔 개발 시 주의

콘솔은 Backend가 어떤 URL 스타일을 지원하는지 알아야 한다.

```text
Ceph RGW → path-style/virtual-hosted-style 설정 확인 필요
RustFS → 지원 방식 확인 필요
AWS S3 → virtual-hosted-style 권장
```

따라서 설정 화면에 다음 항목이 있으면 좋다.

```text
Endpoint URL
Region
Path-style 사용 여부
TLS 검증 여부
Backend Type: ceph / rustfs / minio / aws
```

## 20. S3 Consistency

Consistency는 쓰기 이후 읽기 결과가 언제 반영되는지를 의미한다. AWS S3는 현재 모든 S3 GET, PUT, LIST 작업과 Object Tag, ACL, Metadata 변경에 대해 strong consistency를 제공한다고 발표했다.

참고:

- https://aws.amazon.com/s3/consistency/
- https://aws.amazon.com/blogs/aws/amazon-s3-update-strong-read-after-write-consistency/

### 20-1) Strong Consistency란

Strong Consistency는 쓰기가 성공한 뒤 바로 읽으면 최신 데이터가 보여야 한다는 의미이다.

```text
PUT object-a 성공
  ↓
GET object-a
  ↓
방금 쓴 object-a가 보여야 함
```

### 20-2) S3 호환 제품에서의 주의

Ceph RGW, MinIO, RustFS가 AWS S3와 완전히 동일한 consistency 특성을 제공한다고 단정하면 안 된다. 각 제품의 내부 구조, 배포 방식, 복제 설정에 따라 다를 수 있다. 따라서 콘솔 개발자는 AWS S3 기준 동작과 사설 S3 구현체의 실제 동작을 구분해야 한다.

## 21. Object Storage를 쓰면 좋은 경우

Object Storage가 적합한 워크로드는 다음과 같다.

```text
- 이미지, 동영상, 문서 저장
- 사용자 업로드 파일
- 백업 파일
- 로그 아카이브
- 데이터레이크
- AI 학습 데이터셋
- 모델 아티팩트
- 정적 웹 리소스
- 장기 보관 데이터
```

특히 AI/데이터 플랫폼에서는 Object Storage가 자주 사용된다.

```text
Data Pipeline
  ↓
Raw Data 저장
  ↓
전처리 결과 저장
  ↓
학습 데이터셋 저장
  ↓
모델 파일 저장
  ↓
추론 결과 저장
```

## 22. Object Storage를 쓰면 안 좋은 경우

Object Storage가 모든 워크로드에 적합한 것은 아니다.

```text
- DB 데이터파일
- 초저지연 랜덤 I/O
- 파일 일부를 자주 수정하는 워크로드
- POSIX file lock이 필수인 워크로드
- rename이 매우 빈번한 워크로드
- 로컬 파일시스템 semantics가 필요한 애플리케이션
```

예를 들어 PostgreSQL의 데이터 디렉터리를 S3에 직접 올려 운영하는 것은 일반적인 사용 방식이 아니다. 이런 경우에는 Block Storage나 File Storage가 적합하다.

## 23. Ceph에서 Object Storage가 동작하는 방식

Ceph에서 S3 API를 제공하는 컴포넌트는 RADOS Gateway, 즉 RGW이다.

```text
S3 Client / Console / SDK
  ↓ S3 API
Ceph RGW
  ↓ librados
RADOS
  ↓
OSD
```

Ceph 공식 문서는 Ceph Object Gateway가 librados 위에 구축된 Object Storage 인터페이스이고, S3 호환 API를 제공한다고 설명한다.

참고:

- https://docs.ceph.com/en/latest/radosgw/
- https://docs.ceph.com/en/latest/radosgw/s3/

### 23-1) RADOS와 RGW의 관계

RADOS는 Ceph의 분산 Object Store이다. RGW는 이 RADOS 위에 S3 API Gateway 역할을 한다.

```text
RADOS = Ceph 내부 저장소 계층
RGW   = S3 API를 받아 RADOS에 저장하는 Gateway
```

사용자는 RADOS를 직접 쓰지 않고, S3 API를 통해 RGW에 요청한다. RGW가 내부적으로 RADOS에 데이터를 저장한다.

### 23-2) 우리 회사 콘솔과 Ceph

우리 회사 S3 Console이 Ceph를 지원하려면 다음 두 계층을 고려해야 한다.

```text
S3 API
  - Bucket 목록
  - Object 업로드/다운로드/삭제
  - Bucket Policy

RGW Admin API
  - 사용자 생성
  - Access Key 발급
  - Quota 설정
  - Bucket 통계 조회
```

따라서 Ceph Backend Adapter는 S3 API와 RGW Admin API를 모두 다룰 수 있어야 한다.

## 24. RustFS와 MinIO에서 Object Storage가 동작하는 방식

RustFS와 MinIO는 S3 호환 Object Storage를 제공한다. MinIO는 S3 호환 Object Storage로 널리 사용되며, RustFS는 Rust 기반 S3 호환 Object Storage를 지향한다.

```text
S3 Client / Console
  ↓
RustFS 또는 MinIO S3 API
  ↓
내부 Object Storage Engine
  ↓
Disk / Erasure Set / Pool
```

RustFS Console이나 MinIO Console은 S3 API와 Admin API를 조합해 사용자에게 웹 대시보드를 제공한다. 우리 회사 콘솔은 RustFS Console의 화면 구성을 참고하되, Ceph Backend에서는 지원되지 않는 기능을 숨기거나 Ceph 방식으로 바꿔야 한다.

참고:

- https://github.com/rustfs/rustfs
- https://rustfs.com/
- https://min.io/docs/minio/linux/index.html

## 25. S3 Console 개발 관점에서 01번 문서의 핵심

S3 Console을 만들 때 01번 문서에서 반드시 이해해야 하는 것은 다음이다.

```text
1. S3는 파일시스템이 아니다.
2. Bucket은 최상위 컨테이너이자 정책 적용 단위이다.
3. Object는 파일처럼 보이지만 Key 기반 객체이다.
4. Folder는 실제 폴더가 아니라 Prefix 기반 UI 표현이다.
5. Metadata는 Object의 부가 정보이며 검색 DB가 아니다.
6. ETag는 항상 MD5가 아니다.
7. 큰 파일은 Multipart Upload로 처리해야 한다.
8. Versioning이 켜지면 Delete Marker와 과거 Version을 고려해야 한다.
9. Endpoint와 URL 스타일은 Backend마다 다를 수 있다.
10. Ceph에서는 RGW가 S3 API Gateway 역할을 한다.
```

## 26. 실습: AWS CLI로 S3 호환 스토리지 확인하기

Ceph RGW, MinIO, RustFS 같은 S3 호환 스토리지에서는 AWS CLI의 endpoint-url을 바꿔서 테스트할 수 있다.

### 26-1) Bucket 목록 조회

```bash
aws --endpoint-url https://s3.example.com s3 ls
```

### 26-2) Bucket 생성

```bash
aws --endpoint-url https://s3.example.com s3 mb s3://qcs-test-bucket
```

### 26-3) Object 업로드

```bash
echo "hello object storage" > hello.txt

aws --endpoint-url https://s3.example.com \
  s3 cp hello.txt s3://qcs-test-bucket/hello.txt
```

### 26-4) Object 목록 조회

```bash
aws --endpoint-url https://s3.example.com \
  s3 ls s3://qcs-test-bucket/
```

### 26-5) Prefix 조회

```bash
aws --endpoint-url https://s3.example.com \
  s3api list-objects-v2 \
  --bucket qcs-test-bucket \
  --prefix logs/2026/07/
```

### 26-6) Object Metadata 조회

```bash
aws --endpoint-url https://s3.example.com \
  s3api head-object \
  --bucket qcs-test-bucket \
  --key hello.txt
```

## 27. 실습: mc로 S3 호환 스토리지 확인하기

MinIO Client인 `mc`는 S3 호환 스토리지를 다룰 때 자주 사용된다.

### 27-1) Alias 등록

```bash
mc alias set qcs-s3 https://s3.example.com ACCESS_KEY SECRET_KEY
```

### 27-2) Bucket 목록

```bash
mc ls qcs-s3
```

### 27-3) Bucket 생성

```bash
mc mb qcs-s3/qcs-test-bucket
```

### 27-4) Object 업로드

```bash
mc cp hello.txt qcs-s3/qcs-test-bucket/hello.txt
```

### 27-5) Object 다운로드

```bash
mc cp qcs-s3/qcs-test-bucket/hello.txt ./downloaded-hello.txt
```

## 28. 자주 헷갈리는 개념 정리

### 28-1) Bucket은 폴더인가?

아니다. Bucket은 Object를 담는 최상위 컨테이너이자 정책 적용 단위이다.

### 28-2) Prefix는 폴더인가?

아니다. Prefix는 Key의 앞부분이다. 콘솔이 Prefix를 폴더처럼 보여줄 뿐이다.

### 28-3) Object는 파일인가?

사용자 관점에서는 파일처럼 보일 수 있지만, S3 모델에서는 Key, Data, Metadata를 가진 Object이다.

### 28-4) ETag는 MD5인가?

항상 그렇지 않다. Multipart Upload나 암호화가 있으면 단순 MD5가 아닐 수 있다.

### 28-5) S3에서 파일명 변경은 rename인가?

일반적인 파일시스템 rename과 다르다. 보통 CopyObject 후 DeleteObject이다.

### 28-6) S3는 DB 디스크로 쓸 수 있는가?

일반적으로 적합하지 않다. DB 데이터파일에는 Block Storage가 적합하다.

## 29. Best Practice 요약

```text
- Bucket은 권한/수명주기/고객사/프로젝트 경계를 기준으로 설계한다.
- Bucket 이름은 DNS 호환 규칙에 맞춘다.
- Key는 Prefix 기반 운영이 가능하도록 설계한다.
- 사용자에게 보이는 폴더는 Prefix 기반 UI임을 이해한다.
- 큰 파일은 Multipart Upload를 사용한다.
- ETag를 무조건 MD5로 표시하지 않는다.
- Versioning이 켜진 Bucket은 삭제/복구 UI를 별도로 설계한다.
- Metadata 검색이 필요하면 별도 DB 인덱스를 고려한다.
- 사설 S3에서는 Endpoint, Region, Path-style 옵션을 명확히 관리한다.
- Ceph Backend에서는 S3 API와 RGW Admin API를 구분한다.
```

## 30. 01번 문서 핵심 요약

```text
Object Storage는 데이터를 Object 단위로 저장한다.
S3는 Object Storage API의 사실상 표준이다.
Bucket은 Object의 최상위 컨테이너다.
Object는 Data, Metadata, Key를 가진 저장 단위다.
Key는 경로가 아니라 문자열 식별자다.
Prefix는 폴더가 아니라 Key 앞부분이다.
Metadata는 Object 부가 정보다.
ETag는 항상 단순 MD5가 아니다.
Multipart Upload는 대용량 업로드의 핵심이다.
Versioning은 덮어쓰기/삭제 복구를 위한 기능이다.
Ceph에서는 RGW가 S3 API를 받고 RADOS에 저장한다.
RustFS/MinIO는 S3 호환 Object Storage로 동작한다.
```

## 공식 문서 링크 모음

- AWS S3 User Guide: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html
- AWS S3 API Reference: https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html
- AWS S3 Bucket Naming Rules: https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html
- AWS S3 Virtual Hosting: https://docs.aws.amazon.com/AmazonS3/latest/userguide/VirtualHosting.html
- AWS S3 Object Metadata: https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingMetadata.html
- AWS S3 Multipart Upload: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- AWS S3 Versioning: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
- AWS S3 Strong Consistency: https://aws.amazon.com/s3/consistency/
- AWS S3 Strong Consistency Announcement: https://aws.amazon.com/blogs/aws/amazon-s3-update-strong-read-after-write-consistency/
- Ceph Object Gateway: https://docs.ceph.com/en/latest/radosgw/
- Ceph Object Gateway S3 API: https://docs.ceph.com/en/latest/radosgw/s3/
- RustFS GitHub: https://github.com/rustfs/rustfs
- RustFS Official: https://rustfs.com/
- MinIO Documentation: https://min.io/docs/minio/linux/index.html
