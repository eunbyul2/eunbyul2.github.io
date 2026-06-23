---
layout: post
title: "Data Engine 비교: StarRocks, Trino, Milvus와 Capacity Management"
date: 2026-06-23 21:10:00 +0900
categories: [DataPlatform, Database, Infrastructure]
tags: [StarRocks, Trino, Milvus, MPP, OLAP, VectorDatabase, RAG, DataLake, CapacityPlanning, ThinProvisioning, ResourceGroup, RBAC, MultiTenancy, SharedCluster]
published: true
---

## 1. 글의 목적

데이터 플랫폼에서는 StarRocks, Trino, Milvus 같은 시스템이 함께 언급된다. 셋 다 “데이터를 다루는 시스템”이지만 역할은 다르다. StarRocks는 대규모 정형 데이터를 빠르게 분석하는 MPP OLAP 데이터베이스이고, Trino는 여러 데이터 소스를 SQL로 조회하는 분산 쿼리 엔진이며, Milvus는 embedding vector를 저장하고 유사도 검색을 수행하는 벡터 데이터베이스이다.

이 글은 StarRocks 구조(FE/BE/CN), Shared Nothing, Shared Data, MPP, OLAP, Resource Group, Multi-Tenancy, Trino 구조(Coordinator/Worker/Connector/Query Federation), Milvus 구조(Vector DB/Embedding/Similarity Search/ANN/RAG), Thin Provisioning, Thick Provisioning, Resource Reservation, Utilization, Capacity Planning을 연결해서 정리한다.

공식 문서:

- StarRocks Introduction: <https://docs.starrocks.io/docs/introduction/>
- StarRocks Architecture: <https://docs.starrocks.io/docs/introduction/Architecture/>
- StarRocks Shared-data Quick Start: <https://docs.starrocks.io/docs/quick_start/shared-data/>
- StarRocks Resource Group: <https://docs.starrocks.io/docs/administration/management/resource_management/resource_group/>
- StarRocks CREATE RESOURCE GROUP: <https://docs.starrocks.io/docs/sql-reference/sql-statements/cluster-management/resource_group/CREATE_RESOURCE_GROUP/>
- StarRocks User Privileges: <https://docs.starrocks.io/docs/administration/user_privs/authorization/User_privilege/>
- StarRocks GRANT: <https://docs.starrocks.io/docs/sql-reference/sql-statements/account-management/GRANT/>
- Trino Concepts: <https://trino.io/docs/current/overview/concepts.html>
- Trino Connectors: <https://trino.io/docs/current/connector.html>
- Milvus Architecture Overview: <https://milvus.io/docs/architecture_overview.md>
- Milvus Overview: <https://milvus.io/docs/overview.md>
- AWS What is OLAP: <https://aws.amazon.com/what-is/olap/>
- AWS What is a Data Lake: <https://aws.amazon.com/what-is/data-lake/>

## 2. StarRocks

### 2-1) StarRocks란 무엇인가

StarRocks는 대규모 실시간 분석을 위한 MPP 기반 OLAP 데이터베이스이다. 일반적인 트랜잭션 처리 DB보다는 대용량 분석 쿼리, 집계, 리포트, 대시보드, 데이터레이크 분석에 적합하다.

```text
OLTP DB:
회원가입, 주문, 결제, 게시글 저장

StarRocks:
수십억 건 로그 분석
실시간 대시보드
대규모 집계 쿼리
데이터레이크 분석
```

StarRocks는 고성능 분석을 위해 컬럼형 저장, 벡터화 실행, MPP 기반 병렬 처리, 쿼리 최적화, materialized view 같은 기능을 제공한다.

### 2-2) OLAP란 무엇인가

OLAP는 Online Analytical Processing이다. 분석 질의와 집계 처리를 위한 데이터 처리 방식이다.

```sql
SELECT region, SUM(sales)
FROM sales
WHERE date >= '2026-01-01'
GROUP BY region;
```

이 쿼리는 단건 조회가 아니라 많은 데이터를 읽고 집계한다. OLAP 시스템은 이런 질의에 최적화되어 있다. 반대 개념은 OLTP이다.

```text
OLTP = 짧은 트랜잭션 처리
OLAP = 대량 분석과 집계 처리
```

### 2-3) MPP란 무엇인가

MPP는 Massively Parallel Processing의 약자이다. 하나의 큰 작업을 여러 서버가 나눠서 병렬 처리하는 구조이다.

예를 들어 100TB 데이터를 집계해야 한다면 서버 1대가 전부 처리하는 것이 아니라 여러 노드가 데이터를 나눠 읽고, 부분 결과를 만든 뒤, 최종 결과를 합친다.

```text
Query
↓
Node 1: 일부 데이터 처리
Node 2: 일부 데이터 처리
Node 3: 일부 데이터 처리
↓
결과 병합
```

이 구조 덕분에 StarRocks 같은 MPP OLAP DB는 대량 데이터를 빠르게 분석할 수 있다.

## 3. StarRocks Architecture: FE / BE / CN

### 3-1) FE: Frontend

FE는 StarRocks의 두뇌 역할을 한다.

```text
SQL 요청 접수
SQL 파싱
쿼리 최적화
실행 계획 생성
메타데이터 관리
클러스터 노드 관리
권한 관리
트랜잭션 관리
```

사용자가 SQL을 실행하면 먼저 FE가 요청을 받고, 어떤 방식으로 쿼리를 실행할지 계획을 세운다. 운영 관점에서 FE는 메타데이터와 쿼리 계획을 담당하므로 안정성이 중요하다. 일반적으로 고가용성을 위해 여러 FE를 구성한다.

### 3-2) BE: Backend

BE는 shared-nothing 구조에서 데이터 저장과 쿼리 실행을 담당한다. 즉 데이터를 저장하면서 동시에 쿼리 처리 작업도 수행한다.

```text
데이터 저장
데이터 스캔
집계
JOIN 처리
쿼리 실행
Tablet 관리
```

BE가 많을수록 저장 용량과 처리 성능을 확장할 수 있다. 다만 BE는 리소스를 많이 사용하는 컴포넌트이므로 사용자별로 BE 세트를 중복 배포하면 CPU/Memory request가 빠르게 증가한다.

### 3-3) CN: Compute Node

CN은 Compute Node이다. shared-data 구조에서 쿼리 실행을 담당한다. shared-data 구조에서는 데이터가 S3, MinIO, HDFS 같은 외부 스토리지에 저장되고, CN은 계산만 담당한다.

```text
Object Storage / HDFS / S3 / MinIO
↓
CN들이 데이터 읽고 쿼리 처리
↓
FE가 쿼리 계획 및 메타데이터 관리
```

공식 문서 기준 shared-data 구조는 FE와 CN으로 구성되고 backend object storage를 사용한다. 이 구조는 compute와 storage를 분리하기 때문에 탄력적인 확장에 유리하다.

## 4. Shared Nothing과 Shared Data

### 4-1) Shared Nothing

Shared Nothing 구조에서는 각 BE가 자기 데이터와 compute를 함께 가진다.

```text
FE
├ BE 1: 저장 + 계산
├ BE 2: 저장 + 계산
└ BE 3: 저장 + 계산
```

장점은 로컬 저장소 기반 고성능 처리에 유리할 수 있다는 점이다. 단점은 저장과 계산이 묶여 있어 리소스 확장과 축소가 상대적으로 덜 유연할 수 있다는 점이다.

### 4-2) Shared Data

Shared Data 구조에서는 데이터는 공용 스토리지에 두고, CN이 계산을 담당한다.

```text
FE
├ CN 1: 계산
├ CN 2: 계산
└ CN 3: 계산

Object Storage: 데이터 저장
```

Shared Data 구조는 compute와 storage를 분리한다. 계산 리소스가 더 필요하면 CN을 늘리고, 필요 없으면 줄이는 식으로 운영할 수 있다. Object Storage를 사용하면 저장 계층을 독립적으로 확장할 수 있다.

### 4-3) 어떤 구조가 더 좋은가

정답은 워크로드에 따라 다르다.

```text
Shared Nothing:
로컬 저장소 기반 성능
전통적인 MPP DB 구조
저장과 계산이 결합

Shared Data:
Object Storage 기반
저장과 계산 분리
탄력적 compute 운영
데이터레이크/레이크하우스 연계에 유리
```

대규모 멀티테넌트 환경에서는 Shared Data 구조가 리소스 탄력성 측면에서 검토 가치가 있다. 다만 실제 적용 가능성은 StarRocks 버전, 데이터 크기, 쿼리 패턴, 스토리지 성능, 운영 정책에 따라 달라진다.

## 5. StarRocks Resource Group, Database, Role, Tenant 설계

### 5-1) 왜 Shared Cluster에서 Resource Group이 필요한가

StarRocks를 사용자별로 각각 따로 배포하면 사용자 간 격리는 강해진다. 그러나 FE, BE, CN, Pod, PVC, Service, CPU/Memory request가 사용자 수만큼 반복되기 때문에 리소스 효율은 나빠진다. 반대로 하나의 큰 StarRocks 클러스터를 여러 사용자 또는 조직이 함께 사용하는 Shared Cluster 구조로 만들면 공통 인프라를 공유할 수 있어 훨씬 효율적이다.

다만 Shared Cluster에는 다른 문제가 생긴다. 여러 Tenant가 하나의 StarRocks 클러스터를 함께 쓰기 때문에 한 Tenant의 무거운 쿼리가 전체 클러스터의 CPU와 Memory를 과도하게 사용할 수 있다. 예를 들어 특정 사용자가 대량 JOIN, 대량 GROUP BY, 대량 scan 쿼리를 실행하면 다른 사용자의 대시보드 쿼리나 분석 쿼리가 느려질 수 있다.

이 문제를 제어하기 위해 필요한 것이 StarRocks의 Resource Group이다. StarRocks 공식 문서 기준 Resource Group은 단일 클러스터에서 여러 workload를 동시에 실행할 때 리소스를 격리하기 위한 기능이다. Resource Group은 쿼리를 특정 그룹으로 분류하고, 해당 그룹에 CPU, Memory, 동시성 같은 리소스 제한을 적용하는 방식으로 사용된다.

즉 Resource Group은 Shared Cluster 구조에서 “공유는 하되 무제한으로 쓰게 두지는 않는 장치”에 가깝다.

```text
Shared StarRocks Cluster
├ Tenant A 쿼리 → Resource Group A
├ Tenant B 쿼리 → Resource Group B
├ Batch 쿼리    → Resource Group Batch
└ BI 쿼리       → Resource Group BI
```

이 구조를 사용하면 StarRocks 클러스터를 사용자별로 물리적으로 분리하지 않아도, 쿼리 실행 리소스는 논리적으로 나눠서 제어할 수 있다.

### 5-2) StarRocks Resource Group이 제어하는 대상

Resource Group은 StarRocks 안에서 실행되는 쿼리의 컴퓨팅 리소스를 제어하기 위한 기능이다. Kubernetes의 ResourceQuota가 Namespace 단위로 Pod의 CPU/Memory request 총량을 제한하는 기능이라면, StarRocks Resource Group은 StarRocks 내부에서 쿼리가 사용할 수 있는 실행 리소스를 제한하는 기능이다.

둘은 계층이 다르다.

```text
Kubernetes ResourceQuota
→ Namespace 안의 Pod, PVC, CPU/Memory request 총량 제어

StarRocks Resource Group
→ StarRocks 내부 쿼리의 CPU/Memory/동시성 사용 제어
```

따라서 Shared StarRocks Cluster를 설계할 때는 Kubernetes 계층과 StarRocks 계층을 분리해서 봐야 한다. Kubernetes에서는 StarRocks 클러스터 자체가 사용할 수 있는 Pod 리소스를 관리하고, StarRocks 내부에서는 Resource Group으로 Tenant 또는 workload별 쿼리 리소스를 관리한다.

Resource Group에서 일반적으로 고려할 수 있는 정책은 다음과 같다.

```text
중요 Tenant에게 더 높은 CPU quota 부여
Batch 쿼리는 낮은 우선순위 그룹으로 분리
BI Dashboard 쿼리는 짧은 응답 시간을 위해 별도 그룹으로 분리
Ad-hoc 분석 쿼리는 동시성 제한 적용
대량 scan 쿼리는 Memory 사용량과 concurrency 제한
```

공식 문서의 `CREATE RESOURCE GROUP` 문법에서는 classifier와 resource limit을 함께 지정하는 구조를 사용한다. classifier는 어떤 쿼리를 해당 Resource Group에 넣을지 판단하는 조건이고, resource limit은 해당 그룹에 적용할 리소스 정책이다.

### 5-3) Classifier란 무엇인가

Resource Group에서 classifier는 쿼리를 어떤 Resource Group에 배정할지 결정하는 조건이다. StarRocks 공식 FAQ 기준으로 classifier matching은 IP, user, db, role, query_type 같은 속성에 따라 동작할 수 있다.

예를 들어 다음과 같은 정책을 생각할 수 있다.

```text
user = 'tenant_a_user'
→ tenant_a_rg

role = 'bi_role'
→ bi_dashboard_rg

db = 'tenant_b_db'
→ tenant_b_rg

query_type = 'select'
→ interactive_query_rg
```

즉 Resource Group은 단순히 “그룹을 만든다”에서 끝나는 것이 아니라, 어떤 쿼리가 어떤 그룹으로 들어갈지 분류 기준을 함께 설계해야 한다.

Shared Cluster에서 classifier 설계는 매우 중요하다. classifier가 부정확하면 특정 Tenant의 쿼리가 잘못된 Resource Group에 들어갈 수 있고, 그 결과 리소스 격리가 깨질 수 있다. 예를 들어 Batch 쿼리가 Interactive 쿼리 그룹에 들어가면 대시보드 성능이 느려질 수 있고, 일반 사용자의 Ad-hoc 쿼리가 고우선순위 그룹에 들어가면 핵심 업무 쿼리가 영향을 받을 수 있다.

### 5-4) StarRocks Database는 Tenant 분리에 어떻게 쓰이는가

StarRocks에서 Database는 테이블, 뷰, Materialized View 같은 데이터 객체를 묶는 논리적 단위로 사용할 수 있다. Shared Cluster에서 Tenant를 분리할 때 가장 직관적인 방법 중 하나는 Tenant별 Database를 나누는 것이다.

예시는 다음과 같다.

```text
Shared StarRocks Cluster
├ database: tenant_a_db
│  ├ table: sales
│  ├ table: events
│  └ materialized view
├ database: tenant_b_db
│  ├ table: sales
│  ├ table: events
│  └ materialized view
└ database: tenant_c_db
   ├ table: sales
   ├ table: events
   └ materialized view
```

이 구조에서는 StarRocks 클러스터는 하나지만, 데이터 객체는 Tenant별 Database로 논리적으로 나뉜다. 운영자는 Tenant별 Database에 대해 권한을 다르게 부여할 수 있고, 사용자는 자신에게 허용된 Database 안의 데이터만 조회하도록 제한할 수 있다.

다만 Database 분리만으로 완전한 멀티테넌시가 완성되는 것은 아니다. Database는 데이터 객체를 나누는 단위일 뿐이고, 쿼리 실행 리소스는 별도로 제어해야 한다. 따라서 Database 분리와 Resource Group을 함께 설계해야 한다.

```text
Database
→ 데이터 객체 격리

Role / Privilege
→ 접근 권한 격리

Resource Group
→ 쿼리 리소스 격리
```

### 5-5) StarRocks Role과 권한 설계

StarRocks는 사용자, 역할, 권한을 관리하기 위해 RBAC(Role-Based Access Control)와 IBAC(Identity-Based Access Control)를 제공한다. 공식 문서에서는 StarRocks가 사용자, 역할, 권한을 통해 클러스터 내 객체에 대한 접근을 세밀하게 제어할 수 있다고 설명한다.

Shared Cluster에서 Role은 Tenant별 권한 분리에 핵심적으로 사용된다. 예를 들어 Tenant A 전용 역할과 Tenant B 전용 역할을 만들고, 각 역할에 특정 Database에 대한 권한만 부여할 수 있다.

개념 예시는 다음과 같다.

```sql
CREATE ROLE tenant_a_role;
CREATE ROLE tenant_b_role;

GRANT ALL ON DATABASE tenant_a_db TO ROLE tenant_a_role;
GRANT ALL ON DATABASE tenant_b_db TO ROLE tenant_b_role;

GRANT tenant_a_role TO USER tenant_a_user;
GRANT tenant_b_role TO USER tenant_b_user;
```

위 예시는 구조 이해를 위한 예시이다. 실제 운영에서는 `ALL` 권한을 무조건 부여하기보다 SELECT, INSERT, CREATE TABLE, ALTER, DROP 등 필요한 권한만 최소 권한 원칙에 따라 부여해야 한다.

Role 설계에서 중요한 원칙은 다음과 같다.

```text
Tenant별 Role을 분리한다.
운영자 Role과 사용자 Role을 분리한다.
읽기 전용 Role과 쓰기 가능 Role을 분리한다.
Batch 작업 Role과 Interactive Query Role을 분리할 수 있다.
권한은 최소 권한 원칙으로 부여한다.
권한 변경 이력을 감사할 수 있게 한다.
```

StarRocks의 GRANT 문서는 사용자, 역할, 외부 그룹에 권한 또는 역할을 부여할 수 있다고 설명한다. 따라서 실제 기업 환경에서는 내부 IAM, LDAP, Ranger, Group Provider 같은 외부 인증/권한 체계와 연결하는 방식도 검토할 수 있다.

### 5-6) StarRocks Tenant는 공식 리소스인가

주의할 점이 있다. “Tenant”는 StarRocks에서 반드시 별도의 Kubernetes Resource나 StarRocks Native Object로 존재하는 이름이라고 단정하면 안 된다. 여기서 Tenant는 아키텍처 설계 관점의 논리 단위이다.

즉 Tenant는 다음 요소들을 조합해서 구현하는 개념이다.

```text
Tenant
= 사용자 또는 조직 단위
+ Database / Table / View 분리
+ Role / Privilege 분리
+ Resource Group 분리
+ Query 제한
+ 모니터링 / 과금 / 감사 단위
```

따라서 “StarRocks Tenant를 만든다”는 말은 실제로는 다음 작업들의 조합일 수 있다.

```text
Tenant별 Database를 만든다.
Tenant별 Role을 만든다.
Tenant별 User 또는 External Group을 연결한다.
Tenant별 Resource Group을 만든다.
Tenant별 쿼리 제한과 모니터링 기준을 만든다.
Tenant별 사용량 집계 기준을 만든다.
```

이 관점을 이해해야 사용자별로 StarRocks 전체를 하나씩 배포하지 않고도, 하나의 Shared Cluster 안에서 여러 Tenant를 논리적으로 분리하는 설계를 할 수 있다.

### 5-7) Shared Cluster 설계 예시 1: 사용자별 개별 배포 구조

먼저 비효율적인 구조를 보자.

```text
User A Namespace
└ StarRocks Cluster A
   ├ FE
   ├ BE/CN
   ├ Service
   └ PVC

User B Namespace
└ StarRocks Cluster B
   ├ FE
   ├ BE/CN
   ├ Service
   └ PVC

User C Namespace
└ StarRocks Cluster C
   ├ FE
   ├ BE/CN
   ├ Service
   └ PVC
```

이 구조는 사용자 간 격리는 강하다. User A의 StarRocks와 User B의 StarRocks가 물리적으로 분리되어 있으므로 권한이나 리소스 간섭을 줄이기 쉽다. 그러나 사용자 수가 늘어날수록 리소스 낭비가 커진다. FE, BE/CN, Service, PVC, request, 모니터링 대상, 백업 대상, 업그레이드 대상이 모두 사용자 수만큼 늘어난다.

특히 Kubernetes에서는 각 Pod의 request가 Scheduler의 예약량으로 계산된다. 실제 사용하지 않는 사용자 환경이 많아도 StarRocks Pod가 Running 상태이고 request가 설정되어 있다면 클러스터는 계속 자원이 예약된 것으로 본다.

### 5-8) Shared Cluster 설계 예시 2: 공용 StarRocks + Tenant 논리 분리

더 효율적인 구조는 StarRocks 클러스터를 공용으로 운영하고, 내부에서 Tenant를 논리적으로 분리하는 방식이다.

```text
Shared StarRocks Cluster
├ FE x 3
├ BE/CN Pool
├ Database: tenant_a_db
├ Database: tenant_b_db
├ Database: tenant_c_db
├ Role: tenant_a_role
├ Role: tenant_b_role
├ Role: tenant_c_role
├ Resource Group: tenant_a_rg
├ Resource Group: tenant_b_rg
└ Resource Group: tenant_c_rg
```

이 구조에서는 StarRocks 클러스터 자체는 하나이지만, 데이터 객체, 권한, 쿼리 리소스를 Tenant별로 나눌 수 있다.

```text
데이터 격리
→ Tenant별 Database / Table / View

접근 제어
→ Tenant별 User / Role / Privilege

리소스 격리
→ Tenant별 Resource Group

운영 관측
→ Tenant별 Query Log / Usage Metrics / Audit Log
```

이 방식은 Pod 수와 request 총량을 줄이는 데 유리하다. 공통 FE와 BE/CN Pool을 여러 Tenant가 공유하므로 유휴 사용자 때문에 StarRocks 전체 세트가 계속 떠 있는 문제를 줄일 수 있다.

### 5-9) Shared Cluster 설계 예시 3: workload 유형별 Resource Group

Tenant별 Resource Group만으로 충분하지 않을 수 있다. 같은 Tenant 안에서도 쿼리 유형이 다르기 때문이다. 예를 들어 대시보드 쿼리는 짧고 빠른 응답이 중요하고, Batch 쿼리는 오래 걸려도 되지만 많은 리소스를 사용할 수 있다.

따라서 Resource Group을 Tenant별로만 나누지 않고 workload 유형별로도 나눌 수 있다.

```text
Resource Group 설계 예시

tenant_a_interactive_rg
→ Tenant A의 대시보드/짧은 조회 쿼리

tenant_a_batch_rg
→ Tenant A의 장시간 Batch 쿼리

tenant_b_interactive_rg
→ Tenant B의 대시보드/짧은 조회 쿼리

tenant_b_batch_rg
→ Tenant B의 장시간 Batch 쿼리
```

이 구조에서는 동일 Tenant 안에서도 쿼리 유형에 따라 리소스 정책을 다르게 가져갈 수 있다.

```text
Interactive Query
→ 낮은 latency, 적당한 concurrency, 빠른 응답 우선

Batch Query
→ 긴 실행 시간 허용, 낮은 우선순위, 리소스 상한 명확히 설정

Ad-hoc Query
→ 동시성 제한, scan량 제한, 과도한 resource 사용 방지
```

### 5-10) Shared Cluster 설계 예시 4: Kubernetes와 StarRocks 계층 분리

Shared Cluster를 설계할 때는 Kubernetes 계층과 StarRocks 계층을 분리해서 설계해야 한다.

```text
Kubernetes 계층
├ Namespace
├ StatefulSet / Deployment
├ Service
├ PVC
├ ResourceQuota
├ LimitRange
└ Pod Request / Limit

StarRocks 계층
├ Database
├ Table
├ User
├ Role
├ Privilege
├ Resource Group
├ Query Queue
└ Audit / Query Log
```

Kubernetes Namespace는 StarRocks 클러스터 자체를 운영하기 위한 인프라 격리 단위로 사용할 수 있다. 예를 들어 `starrocks-prod`, `starrocks-dev`, `starrocks-monitoring` 같은 Namespace를 둘 수 있다. 반면 Tenant 분리는 StarRocks 내부의 Database, Role, Privilege, Resource Group으로 처리할 수 있다.

즉 사용자별 Namespace를 무조건 만드는 것이 아니라, 어떤 수준의 격리가 필요한지에 따라 계층을 분리해야 한다.

```text
Namespace
→ StarRocks 운영 환경 단위 또는 클러스터 단위 격리

Database / Role / Resource Group
→ Tenant 단위 데이터/권한/리소스 격리
```

### 5-11) Shared Cluster 설계 시 반드시 고려해야 할 위험

Shared Cluster는 리소스 효율을 높일 수 있지만, 잘못 설계하면 전체 장애 범위가 커질 수 있다. 사용자별 개별 배포 구조에서는 한 사용자 환경 장애가 다른 사용자에게 영향을 덜 줄 수 있지만, Shared Cluster에서는 공통 FE, BE/CN, Object Storage, Network 문제가 여러 Tenant에게 영향을 줄 수 있다.

따라서 다음 요소를 반드시 설계해야 한다.

```text
권한 격리
Tenant가 다른 Tenant의 Database나 Table을 조회하지 못해야 한다.

리소스 격리
한 Tenant의 쿼리가 전체 CPU/Memory를 독점하지 못해야 한다.

성능 격리
대시보드 쿼리와 Batch 쿼리가 서로 영향을 덜 주도록 분리해야 한다.

장애 격리
특정 Tenant의 비정상 쿼리나 과도한 작업이 전체 클러스터 장애로 이어지지 않게 해야 한다.

감사 로그
누가 어떤 데이터에 접근했는지 추적할 수 있어야 한다.

사용량 집계
Tenant별 query count, scan bytes, CPU time, memory usage, storage usage를 집계해야 한다.

운영 자동화
Tenant 생성, 권한 부여, Resource Group 연결, 모니터링 등록이 자동화되어야 한다.
```

### 5-12) Shared Cluster 전환 시 단계적 접근

이미 사용자별 개별 배포 구조가 운영 중이라면 한 번에 Shared Cluster로 전환하기 어렵다. 데이터 위치, 권한, 접속 정보, 사용자별 설정, 쿼리 패턴, SLA가 모두 얽혀 있기 때문이다.

현실적인 접근은 단계적 전환이다.

```text
1단계: 현재 사용자별 StarRocks 사용량 분석
- 실제 사용 중인 사용자
- 장기간 미사용 사용자
- 쿼리 빈도
- 데이터 크기
- 리소스 사용량
- 권한 구조

2단계: Shared Cluster Pilot 구성
- 소수 Tenant만 이전
- Database / Role / Resource Group 모델 검증
- 성능 간섭 여부 확인
- 쿼리 호환성 검증

3단계: Tenant Provisioning 자동화
- Tenant Database 생성
- Role 생성
- User 또는 Group 연결
- Resource Group 생성
- 모니터링 등록

4단계: 저위험 Tenant부터 이전
- 사용량 낮은 Tenant
- 데이터 크기 작은 Tenant
- SLA 낮은 Tenant

5단계: 고위험 Tenant 이전
- 중요 업무 Tenant
- 대용량 Tenant
- 높은 동시성 Tenant

6단계: 개별 배포 구조 축소
- 미사용 StarRocks 정리
- 중복 PVC 정리
- Namespace 정리
- request 총량 감소 확인
```

이 방식은 단기적으로 리소스 최적화를 수행하면서, 장기적으로 아키텍처 구조를 개선하는 방향이다.

### 5-13) StarRocks Shared Cluster 설계 요약

StarRocks Shared Cluster의 핵심은 “하나의 클러스터를 여러 Tenant가 함께 쓰되, 데이터·권한·리소스·운영 관측을 논리적으로 분리하는 것”이다.

정리하면 다음과 같다.

```text
StarRocks Database
→ Tenant별 데이터 객체 분리

StarRocks Role / Privilege
→ Tenant별 접근 제어

StarRocks Resource Group
→ Tenant별 또는 workload별 쿼리 리소스 제어

Kubernetes Namespace
→ StarRocks 클러스터 운영 단위 격리

Kubernetes ResourceQuota / LimitRange
→ StarRocks Pod 자체의 인프라 리소스 상한 관리

Prometheus / Query Log / Audit Log
→ Tenant별 사용량과 접근 이력 관측
```

이 설계를 제대로 해야 사용자별 StarRocks 개별 배포 구조에서 발생하는 Pod 폭증, request 폭증, 운영 대상 폭증 문제를 줄일 수 있다.


## 6. Trino

### 6-1) Trino란 무엇인가

Trino는 분산 SQL 쿼리 엔진이다. 데이터를 직접 저장하는 데이터베이스라기보다 여러 데이터 저장소에 연결해서 SQL로 조회할 수 있게 해주는 엔진이다.

```text
Hive
Iceberg
Delta Lake
S3
HDFS
MySQL
PostgreSQL
Kafka
Elasticsearch
```

Trino는 데이터가 여러 곳에 흩어져 있어도 SQL 하나로 조회하게 해주는 계층으로 이해하면 된다.

### 6-2) Trino Architecture

Trino는 Coordinator와 Worker로 구성된다.

```text
Client
↓
Trino Coordinator
↓
Trino Workers
↓
Data Sources
```

Coordinator는 SQL을 파싱하고, 쿼리 계획을 만들고, Worker를 관리한다. Worker는 실제 쿼리 작업을 수행하고 데이터를 처리한다.

### 6-3) Coordinator

Coordinator는 Trino 클러스터의 두뇌이다.

```text
SQL 요청 수신
SQL 파싱
쿼리 계획 생성
Worker 관리
Task 분배
최종 결과 반환
```

Coordinator에 장애가 발생하면 쿼리 제출과 관리에 문제가 생길 수 있다. 운영 환경에서는 Coordinator의 안정성과 리소스 확보가 중요하다.

### 6-4) Worker

Worker는 실제 쿼리를 실행하는 노드이다.

```text
데이터 소스에서 데이터 읽기
필터링
집계
조인
중간 결과 교환
```

Worker 수를 늘리면 더 많은 쿼리나 더 큰 데이터를 처리할 수 있지만, Coordinator와 네트워크, 데이터 소스 성능도 함께 고려해야 한다.

### 6-5) Connector

Trino의 핵심은 Connector이다. Connector를 통해 다양한 데이터 소스를 SQL로 조회할 수 있다. 데이터가 S3, HDFS, Hive Metastore, Iceberg, MySQL, Kafka 등에 흩어져 있어도 Trino를 통해 하나의 SQL 계층에서 조회할 수 있다.

### 6-6) Query Federation

Query Federation은 여러 데이터 소스에 흩어진 데이터를 하나의 쿼리 계층에서 조회하는 방식이다.

```text
MySQL
S3/Iceberg
Kafka
PostgreSQL
Hive
↓
Trino
↓
SQL Query
```

Trino는 Query Federation에 강하다. 하나의 SQL로 여러 데이터 소스를 조합해 조회할 수 있다. 다만 모든 쿼리가 항상 빠른 것은 아니다. 데이터 소스의 성능, 네트워크, connector pushdown, join 방식, worker 리소스에 따라 성능이 크게 달라진다.

### 6-7) Trino와 StarRocks의 차이

```text
Trino:
여러 데이터 소스를 SQL로 연결 조회
Query Federation에 강함
저장보다는 조회 엔진 성격

StarRocks:
고성능 OLAP 분석 DB
데이터 적재 및 빠른 집계/분석
MPP + Columnar + Vectorized Engine
```

간단히 말하면 Trino는 “연결해서 조회”에 강하고, StarRocks는 “고성능 분석 DB”에 가깝다.

## 7. Milvus

### 7-1) Milvus란 무엇인가

Milvus는 오픈소스 벡터 데이터베이스이다. 대규모 고차원 벡터 데이터셋에 대한 similarity search를 위해 설계된 cloud-native vector database이다.

벡터 데이터베이스는 일반적인 SQL DB와 다르게 숫자 배열 형태의 embedding vector를 저장하고, 의미적으로 비슷한 데이터를 빠르게 찾는 데 사용된다.

### 7-2) Vector와 Embedding

AI 모델은 텍스트, 이미지, 음성 같은 비정형 데이터를 숫자 배열로 변환할 수 있다. 이 숫자 배열을 embedding vector라고 한다.

```text
"Kubernetes 리소스 최적화" → [0.12, -0.44, 0.98, ...]
```

비슷한 의미의 문장은 비슷한 벡터 위치에 놓인다. Milvus는 이런 벡터들을 저장하고, 입력 벡터와 가까운 벡터를 빠르게 찾는다.

### 7-3) Similarity Search

Milvus의 핵심은 similarity search이다.

```text
질문 입력
↓
질문을 embedding vector로 변환
↓
Milvus에서 가장 비슷한 vector 검색
↓
관련 문서 반환
```

이 구조는 RAG에서 많이 사용된다.

```text
사용자 질문
↓
Embedding
↓
Milvus 검색
↓
관련 문서 추출
↓
LLM에 문서 전달
↓
답변 생성
```

### 7-4) ANN

Milvus 같은 벡터 DB는 대규모 벡터에서 빠르게 유사한 벡터를 찾기 위해 ANN을 사용한다. ANN은 Approximate Nearest Neighbor의 약자이다. 정확히 모든 벡터와 비교하는 방식은 데이터가 커지면 너무 느리기 때문에, 근사적으로 가장 가까운 벡터를 빠르게 찾는 알고리즘을 사용한다.

정확도와 속도 사이에는 trade-off가 있다. 인덱스 설정, 거리 metric, top-k 값, 데이터 분포에 따라 검색 품질과 성능이 달라진다.

### 7-5) Milvus와 StarRocks의 차이

```text
StarRocks:
정형 데이터
테이블
SQL
집계/분석
OLAP

Milvus:
비정형 데이터의 embedding vector
유사도 검색
ANN
RAG/AI 검색
```

매출 합계를 구하는 것은 StarRocks가 맞고, 질문과 의미가 비슷한 문서를 찾는 것은 Milvus가 맞다.

## 8. Trino, StarRocks, Milvus 비교

```text
Trino:
여러 데이터 저장소를 SQL로 조회하는 분산 쿼리 엔진

StarRocks:
대용량 정형 데이터를 빠르게 분석하는 MPP OLAP 데이터베이스

Milvus:
Embedding vector를 저장하고 유사도 검색하는 벡터 데이터베이스
```

세 시스템은 서로 대체 관계라기보다 서로 다른 목적을 가진 데이터 플랫폼 구성요소이다. Trino는 Data Lake 위의 Query Federation 계층, StarRocks는 고성능 분석 저장/처리 계층, Milvus는 AI 검색/RAG 계층에 가깝다.

## 9. Thin Provisioning과 Thick Provisioning

### 9-1) Thick Provisioning

Thick Provisioning은 사용자에게 리소스를 할당할 때 물리 자원도 즉시 예약하는 방식이다.

```text
100GB 할당
↓
물리 스토리지 100GB 즉시 예약
```

장점은 예측 가능성이다. 단점은 실제 사용량이 적어도 물리 자원이 묶인다는 점이다.

### 9-2) Thin Provisioning

Thin Provisioning은 사용자에게는 큰 용량이 할당된 것처럼 보이지만, 실제 물리 스토리지는 사용한 만큼만 소비하는 방식이다.

```text
100GB 할당처럼 보임
↓
실제 사용량 10GB
↓
물리 스토리지는 10GB만 소비
```

장점은 리소스 효율이다. 모든 사용자가 할당량을 동시에 전부 사용하는 경우는 드물기 때문에 실제 사용량 기준으로 물리 자원을 계획할 수 있다. 단점은 overcommit 위험이다. 모든 사용자가 갑자기 할당량을 다 사용하면 물리 스토리지가 부족해진다.

### 9-3) Kubernetes CPU/Memory와 Thin Provisioning

Kubernetes CPU/Memory에서도 “실제로 쓰는 만큼만 잡을 수 없나?”라는 고민이 생길 수 있다. 하지만 CPU/Memory request는 Scheduler가 예약 자원으로 사용하는 값이다. 스토리지 Thin Provisioning처럼 완전히 “논리 할당만 크게 하고 물리 사용량만큼만 소비”하는 구조와는 다르다.

Kubernetes에서 비슷한 효과를 내려면 request를 실제 사용량에 맞게 낮추고, limit은 필요 시 더 높게 두며, VPA/Goldilocks로 적정 request를 산정하고, In-place Resize로 유휴 Pod request를 축소하는 방식이 필요하다.

## 10. Capacity Management

### 10-1) Resource Reservation

Resource Reservation은 시스템이 논리적으로 예약했다고 보는 자원량이다. Kubernetes에서는 Pod의 request가 reservation 역할을 한다.

```text
request = Kubernetes Scheduler 관점의 예약량
```

### 10-2) Resource Utilization

Resource Utilization은 실제 사용량이다.

```text
usage = 실제 CPU/Memory 사용량
```

Reservation과 utilization이 크게 차이 나면 최적화 대상이 된다.

```text
request 높음 + usage 낮음 = over-provisioning
request 낮음 + usage 높음 = under-provisioning
```

### 10-3) Capacity Planning

Capacity Planning은 현재와 미래의 워크로드를 기준으로 필요한 CPU, Memory, Storage, Network, Node 수를 계획하는 작업이다.

대규모 데이터 플랫폼에서는 사용자 수 증가율, 쿼리 수 증가율, 데이터 증가율, 피크 시간대 사용량, 평균 사용량, p95/p99 사용량, request 총량, 실제 usage, OOMKilled, CPU throttling, Storage 증가 속도를 함께 봐야 한다.

### 10-4) 데이터 플랫폼 Capacity Planning의 특징

데이터 플랫폼은 일반 웹 서비스보다 사용량 변동이 크다. 특정 시간대에 대시보드 쿼리가 몰리거나, 대규모 batch 작업이 돌거나, 월말/분기말 리포트 쿼리가 증가할 수 있다.

따라서 평균 사용량만으로 capacity를 계획하면 위험하다. 피크, 쿼리 복잡도, scan bytes, concurrency, cache hit ratio, compaction, ingestion 작업까지 함께 봐야 한다.

## 11. Kubernetes에서 데이터 엔진을 운영할 때 주의할 점

### 11-1) StatefulSet, Helm, Operator

StarRocks, Trino, Milvus 같은 시스템은 Kubernetes에서 Deployment, StatefulSet, Helm Chart, Operator로 배포될 수 있다.

Pod를 직접 patch해도 Helm values.yaml이나 Operator Custom Resource에는 기존 값이 남아 있을 수 있다. 이후 `helm upgrade`나 Operator reconcile이 발생하면 patch한 값이 원복될 수 있다.

따라서 리소스 최적화를 적용하기 전에는 애플리케이션이 Deployment인지 StatefulSet인지, Helm Chart로 배포되었는지, Operator가 관리하는지, resources 설정의 원천이 어디인지, Pod 직접 patch가 reconcile로 원복되는지 확인해야 한다.

### 11-2) 모니터링

데이터 엔진에서는 다음 지표가 중요하다.

```text
CPU usage
Memory usage
CPU throttling
OOMKilled
query latency
query concurrency
scan bytes
cache hit ratio
compaction
ingestion throughput
storage usage
network throughput
```

단순 CPU/Memory만 보면 안 된다. 쿼리 성능과 데이터 처리 지표를 함께 봐야 한다.

## 12. 최종 정리

StarRocks, Trino, Milvus는 모두 데이터 플랫폼에서 중요한 시스템이지만 역할이 다르다.

```text
StarRocks:
MPP 기반 OLAP 분석 DB
FE/BE/CN 구조
Shared Nothing과 Shared Data 지원
Resource Group으로 쿼리 리소스 격리 가능

Trino:
분산 SQL 쿼리 엔진
Coordinator/Worker 구조
Connector 기반 Query Federation

Milvus:
Vector Database
Embedding 저장
Similarity Search와 ANN
RAG 아키텍처에 사용
```

데이터 플랫폼을 Kubernetes 위에서 운영할 때는 단순히 Pod가 Running인지 보는 것으로 충분하지 않다. request와 실제 사용량, 쿼리 지연, OOM, throttling, storage 증가율, Tenant별 사용량, Capacity Planning을 함께 봐야 한다.
