---
layout: post
title: "Multi-Tenancy와 Data Platform Architecture: Data Lake, Warehouse, Lakehouse, Catalog"
date: 2026-06-23 20:33:00 +0900
categories: [스터디, 인프라, DataPlatform, Architecture, Kubernetes]
tags: [MultiTenancy, Tenant, SharedCluster, DataLake, DataWarehouse, DataLakehouse, ObjectStorage, Metadata, Catalog]
published: true
---

## 1. 글의 목적

대규모 Kubernetes 기반 데이터 플랫폼에서는 애플리케이션을 어떻게 배포하느냐만큼 “사용자를 어떤 단위로 격리할 것인가”가 중요하다. 사용자마다 애플리케이션 세트를 물리적으로 분리하면 격리는 강하지만 리소스 비용과 운영 복잡도가 급격히 증가한다. 반대로 하나의 Shared Cluster 안에서 사용자를 논리적으로 분리하면 리소스 효율은 좋아지지만 권한, 데이터, 리소스, 장애 격리를 정교하게 설계해야 한다.

이 글은 Multi-Tenancy, Tenant, Single Tenant, Multi-Tenant, Shared Cluster, Shared Resource, Namespace 기반 격리, Logical Isolation, Physical Isolation, Data Lake, Data Warehouse, Data Lakehouse, Object Storage, Metadata, Catalog를 연결해서 정리한다.

공식 문서:

- Kubernetes Namespaces: <https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/>
- Kubernetes Multi-tenancy: <https://kubernetes.io/docs/concepts/security/multi-tenancy/>
- Kubernetes Resource Quotas: <https://kubernetes.io/docs/concepts/policy/resource-quotas/>
- AWS What is a Data Lake: <https://aws.amazon.com/what-is/data-lake/>
- AWS Data Lakes and Analytics: <https://aws.amazon.com/big-data/datalakes-and-analytics/>
- Databricks Data Lakehouse: <https://www.databricks.com/glossary/data-lakehouse>
- Apache Iceberg: <https://iceberg.apache.org/>
- Apache Hive Metastore: <https://cwiki.apache.org/confluence/display/Hive/AdminManual+Metastore+Administration>

## 2. Tenant와 Multi-Tenancy

### 2-1) Tenant란 무엇인가

Tenant는 하나의 시스템을 나눠 쓰는 사용자, 팀, 조직, 고객 단위를 의미한다. 예를 들어 하나의 SaaS 시스템을 여러 회사가 함께 사용한다면 각 회사가 Tenant가 된다.

```text
하나의 서비스
├ A 회사 Tenant
├ B 회사 Tenant
└ C 회사 Tenant
```

Tenant는 Kubernetes만의 개념이 아니다. 서비스 아키텍처 또는 비즈니스 관점에서 “누구를 독립된 사용자 단위로 볼 것인가”를 표현하는 개념이다.

### 2-2) Single Tenant

Single Tenant 구조는 각 Tenant가 독립된 시스템을 갖는 방식이다.

```text
Tenant A → Application Cluster A
Tenant B → Application Cluster B
Tenant C → Application Cluster C
```

장점은 격리가 강하다는 점이다. 한 Tenant의 장애, 리소스 폭주, 데이터 문제, 권한 문제가 다른 Tenant에 영향을 덜 준다. 규제나 보안이 강한 환경에서는 Single Tenant 구조가 적합할 수 있다.

단점은 비용과 운영 복잡도다. Tenant 수가 늘어날수록 애플리케이션 인스턴스, Pod, PVC, 모니터링 대상, 백업 대상, 업그레이드 대상이 모두 증가한다.

### 2-3) Multi-Tenant

Multi-Tenant 구조는 하나의 공통 시스템을 여러 Tenant가 함께 사용하는 방식이다.

```text
Shared System
├ Tenant A
├ Tenant B
└ Tenant C
```

장점은 리소스 효율과 운영 효율이다. 공통 클러스터, 공통 애플리케이션, 공통 모니터링, 공통 업그레이드 체계를 사용할 수 있다.

단점은 격리 설계가 어렵다는 점이다. Multi-Tenant 구조에서는 데이터 격리, 권한 격리, 리소스 격리, 성능 간섭, 장애 전파 범위를 반드시 설계해야 한다.

## 3. Namespace 기반 격리

### 3-1) Namespace란 무엇인가

Namespace는 Kubernetes 리소스를 논리적으로 분리하는 단위이다.

```text
Kubernetes Cluster
├ namespace-a
│  ├ Pod
│  ├ Service
│  ├ ConfigMap
│  └ Secret
└ namespace-b
   ├ Pod
   ├ Service
   ├ ConfigMap
   └ Secret
```

Namespace를 사용하면 팀, 환경, 사용자 단위로 Kubernetes 객체를 분리할 수 있다.

### 3-2) Namespace로 격리할 수 있는 것

Namespace는 다음과 같은 격리에 도움을 준다.

```text
리소스 이름 충돌 방지
RBAC 권한 분리
ResourceQuota 적용
LimitRange 적용
NetworkPolicy 적용
Secret/ConfigMap 분리
```

하지만 Namespace는 완전한 보안 경계가 아니다. 클러스터 수준 리소스, Node 리소스, CNI 설정, StorageClass, CRD, ClusterRole 등은 Namespace 경계를 넘어선다. 따라서 보안이 중요한 환경에서는 Namespace만으로 충분하다고 보면 안 된다.

### 3-3) Namespace와 Tenant의 차이

Namespace는 Kubernetes 계층의 리소스 격리 단위이고, Tenant는 서비스 또는 데이터 플랫폼 계층의 사용자 격리 단위이다.

```text
Namespace = Kubernetes 관점의 리소스 격리 단위
Tenant    = 서비스/데이터 플랫폼 관점의 사용자 격리 단위
```

Namespace 하나가 Tenant 하나에 대응될 수도 있다. 하지만 항상 그래야 하는 것은 아니다. 하나의 Tenant가 여러 Namespace를 사용할 수도 있고, 하나의 Namespace 안에 여러 Tenant가 논리적으로 존재할 수도 있다. 중요한 것은 어떤 계층에서 무엇을 격리할지 명확히 정하는 것이다.

## 4. Physical Isolation과 Logical Isolation

### 4-1) Physical Isolation

Physical Isolation은 사용자마다 독립된 물리 리소스나 애플리케이션 인스턴스를 제공하는 방식이다.

```text
User A → Application Cluster A
User B → Application Cluster B
User C → Application Cluster C
```

장점은 격리가 강하다는 점이다. 사용자별 장애 격리, 권한 격리, 데이터 격리가 상대적으로 명확하다. 단점은 리소스 낭비가 크다는 점이다. 애플리케이션 하나가 무거울수록 사용자 수 증가에 따른 비용이 급격히 증가한다. 사용자가 실제로 거의 사용하지 않아도 해당 애플리케이션 인스턴스는 계속 Running 상태로 남아 request를 점유할 수 있다.

### 4-2) Logical Isolation

Logical Isolation은 하나의 공용 시스템 안에서 권한, DB, Schema, Role, Resource Group 등으로 사용자를 분리하는 방식이다.

```text
Shared Application Cluster
├ Tenant A 논리 공간
├ Tenant B 논리 공간
└ Tenant C 논리 공간
```

장점은 리소스 효율이다. 공통 애플리케이션 인스턴스를 여러 사용자가 공유하므로 Pod 수, request 총량, 운영 대상이 줄어든다. 단점은 설계 난이도다. 사용자 간 데이터 접근 차단, 쿼리 리소스 제한, 성능 간섭 방지, 장애 범위 제한, 감사 로그, 과금 기준을 정교하게 설계해야 한다.

### 4-3) Physical과 Logical의 선택 기준

Physical Isolation이 맞는 경우는 다음과 같다.

```text
강한 보안 격리가 필요함
Tenant별 규제 요구사항이 다름
데이터가 절대 섞이면 안 됨
Tenant별 커스터마이징이 큼
장애 전파를 극도로 제한해야 함
```

Logical Isolation이 맞는 경우는 다음과 같다.

```text
사용자 수가 많음
대부분의 사용자가 유휴 상태임
공통 기능이 많음
리소스 효율이 중요함
공통 모니터링과 운영 자동화가 필요함
```

대규모 데이터 플랫폼에서는 보통 두 방식을 혼합한다. 핵심 Tenant는 강하게 격리하고, 일반 Tenant는 Shared Cluster 안에서 논리 격리하는 방식이다.

## 5. Shared Cluster와 Shared Resource

### 5-1) Shared Cluster란 무엇인가

Shared Cluster는 하나의 큰 클러스터를 여러 Tenant가 함께 사용하는 구조이다.

```text
Shared Cluster
├ Tenant A
├ Tenant B
├ Tenant C
└ Tenant D
```

데이터 플랫폼에서는 하나의 분석 DB 클러스터 또는 쿼리 엔진 클러스터를 공통으로 운영하고, 그 안에서 Tenant별 DB, Schema, Role, Resource Group 등을 사용해 격리할 수 있다.

### 5-2) Shared Resource의 장점

Shared Resource 구조는 다음 장점이 있다.

```text
Pod 수 감소
request 총량 감소
운영 대상 감소
모니터링 대상 단순화
업그레이드 단순화
공통 캐시 활용 가능
리소스 풀링 효과
```

사용자별로 애플리케이션 세트를 따로 띄우는 구조에서는 유휴 사용자도 리소스를 점유한다. Shared Resource 구조에서는 유휴 사용자의 물리 인스턴스가 따로 존재하지 않으므로 리소스 효율이 좋아진다.

### 5-3) Shared Resource의 위험

Shared Resource 구조는 성능 간섭과 장애 전파 위험이 있다.

```text
한 Tenant의 무거운 쿼리가 전체 클러스터 성능에 영향
권한 설정 오류 시 데이터 노출 위험
공통 클러스터 장애 시 여러 Tenant 동시 영향
Tenant별 과금/사용량 산정 필요
```

따라서 Shared Cluster를 설계할 때는 권한, 리소스 그룹, 쿼리 제한, 모니터링, 감사 로그, 장애 격리 정책이 필요하다.

## 6. Multi-Tenancy 설계 체크리스트

### 6-1) 데이터 격리

Tenant별 데이터가 서로 노출되지 않도록 해야 한다. DB, Schema, Table, Row-level security, Column-level masking, Role 권한을 검토해야 한다.

### 6-2) 권한 격리

Tenant별 사용자와 서비스 계정이 자신에게 허용된 데이터와 기능만 접근하도록 해야 한다. Kubernetes 계층에서는 RBAC, 애플리케이션 계층에서는 Role/Privilege 설계가 필요하다.

### 6-3) 리소스 격리

Tenant별 CPU, Memory, Query concurrency, Queue, Storage 사용량을 제한해야 한다. Shared Cluster에서는 Resource Group이나 Query Queue 같은 기능이 중요하다.

### 6-4) 네트워크 격리

Namespace 기반 격리를 사용할 경우 NetworkPolicy로 Pod 간 통신 범위를 제한할 수 있다. 단, 데이터 플랫폼 내부 통신 구조를 잘못 막으면 서비스 장애가 발생할 수 있으므로 네트워크 흐름을 먼저 파악해야 한다.

### 6-5) 장애 격리

한 Tenant의 장애가 다른 Tenant로 전파되지 않도록 해야 한다. 무거운 쿼리, 폭주 요청, 대량 적재 작업, 잘못된 배치 작업이 전체 클러스터에 영향을 줄 수 있다.

### 6-6) 과금과 사용량 집계

Multi-Tenant 플랫폼에서는 Tenant별 사용량을 측정해야 한다. CPU 사용량, Memory 사용량, Storage 사용량, Query 수, Query 처리 시간, Scan bytes, 동시성, 에러율은 비용 배분, quota 정책, SLA 관리에 필요하다.

## 7. Data Platform Architecture

### 7-1) Data Platform이란 무엇인가

Data Platform은 데이터를 수집, 저장, 처리, 분석, 검색, 제공하기 위한 전체 시스템이다. 단일 DB 하나만을 의미하지 않는다.

```text
Ingestion Layer: 데이터 수집, ETL/ELT, 스트리밍
Storage Layer: Object Storage, HDFS, Data Lake
Metadata/Catalog Layer: 테이블 정의, 스키마, 파티션, 권한, lineage
Query/Processing Layer: Trino, StarRocks, Spark
Serving/Analytics Layer: BI, Dashboard, API
AI Search Layer: Vector DB, RAG
Observability Layer: Metrics, Logs, Traces
```

## 8. Data Lake

### 8-1) Data Lake란 무엇인가

Data Lake는 정형, 반정형, 비정형 데이터를 원본에 가까운 형태로 대량 저장하는 저장소 아키텍처이다. 일반적으로 Object Storage나 HDFS 위에 구축된다.

```text
로그
CSV
Parquet
JSON
이미지
음성
문서
이벤트 데이터
```

Data Lake의 장점은 다양한 데이터를 유연하게 저장할 수 있다는 점이다. 데이터 형식이 완전히 정리되기 전에도 저장할 수 있고, 이후 분석 목적에 따라 가공할 수 있다.

### 8-2) Data Lake의 한계

Data Lake는 저장소 자체만으로는 충분하지 않다. 데이터가 어디에 있고, 어떤 스키마를 가지며, 누가 접근할 수 있고, 어떤 버전이 최신인지 관리하지 않으면 Data Swamp가 될 수 있다. 따라서 Data Lake에는 Metadata, Catalog, Table Format, 권한 관리, 품질 관리가 필요하다.

## 9. Data Warehouse

### 9-1) Data Warehouse란 무엇인가

Data Warehouse는 분석과 리포팅을 위해 정제된 데이터를 구조화해서 저장하는 시스템이다. Data Lake보다 스키마와 품질 관리가 강한 편이다.

```text
정제된 테이블
비즈니스 기준 지표
대시보드
정기 리포트
분석 쿼리
```

Data Warehouse는 OLAP 쿼리 성능과 데이터 신뢰성이 중요하다.

### 9-2) Data Lake와 Data Warehouse의 차이

```text
Data Lake:
원본 또는 다양한 형태의 데이터 저장
유연함
스키마 적용이 늦을 수 있음
저장 중심

Data Warehouse:
정제된 구조화 데이터 저장
분석과 리포팅 중심
스키마와 품질 관리 강함
쿼리 성능 중요
```

## 10. Data Lakehouse

### 10-1) Data Lakehouse란 무엇인가

Data Lakehouse는 Data Lake의 유연성과 Data Warehouse의 관리 기능을 결합하려는 아키텍처이다. Object Storage 같은 저렴하고 확장성 높은 저장소 위에 테이블 포맷, 트랜잭션, 스키마 진화, 메타데이터 관리를 더해 분석 시스템처럼 사용하는 방식이다.

대표적인 테이블 포맷으로는 Apache Iceberg, Delta Lake, Apache Hudi 등이 있다.

### 10-2) Lakehouse가 필요한 이유

Data Lake만 있으면 데이터 품질과 테이블 관리가 어려울 수 있다. Data Warehouse만 사용하면 저장 비용과 유연성에 한계가 있을 수 있다. Lakehouse는 Object Storage 기반 저장소를 활용하면서도 테이블 단위 관리와 분석 성능을 확보하려는 방향이다.

## 11. Object Storage

### 11-1) Object Storage란 무엇인가

Object Storage는 데이터를 파일 시스템의 디렉터리 구조가 아니라 객체 단위로 저장하는 스토리지이다. 대표적으로 Amazon S3, MinIO, Google Cloud Storage, Azure Blob Storage가 있다.

Object Storage는 대용량 데이터 저장에 적합하다. Data Lake와 Lakehouse에서 많이 사용된다.

### 11-2) Object Storage가 데이터 플랫폼에서 중요한 이유

Object Storage는 대용량 저장, 수평 확장, 상대적으로 낮은 비용, 내구성, Data Lake/Lakehouse와의 호환성 때문에 중요하다. StarRocks shared-data 구조, Trino + Iceberg/Hive 구조, Spark 기반 Lakehouse 구조 등에서 Object Storage는 중요한 저장 계층이 될 수 있다.

## 12. Metadata와 Catalog

### 12-1) Metadata란 무엇인가

Metadata는 데이터에 대한 데이터이다.

```text
테이블 이름
컬럼 이름
컬럼 타입
파티션 정보
파일 위치
데이터 생성 시각
소유자
권한
통계 정보
```

Metadata가 없으면 데이터가 어디에 있고 어떻게 읽어야 하는지 알기 어렵다.

### 12-2) Catalog란 무엇인가

Catalog는 Metadata를 관리하고 조회할 수 있게 해주는 계층이다. 데이터 플랫폼에서 Catalog는 테이블 목록, 스키마, 파티션, 권한, 위치 정보를 제공한다.

예를 들어 Trino가 Iceberg 테이블을 조회하려면 Catalog를 통해 테이블 메타데이터와 데이터 파일 위치를 알아야 한다.

### 12-3) Catalog가 중요한 이유

Catalog가 없으면 Data Lake는 파일 더미가 되기 쉽다. Catalog는 데이터 발견, 권한 관리, 스키마 관리, lineage, 품질 관리의 기반이다.

```text
데이터 발견
스키마 관리
권한 관리
파티션 관리
쿼리 엔진 연동
데이터 거버넌스
```

## 13. 데이터 플랫폼과 Kubernetes의 연결

### 13-1) Kubernetes가 담당하는 영역

Kubernetes는 데이터 플랫폼 컴포넌트를 실행하고 운영하는 오케스트레이션 계층이다.

```text
Trino Coordinator/Worker 실행
StarRocks FE/BE/CN 실행
Milvus 컴포넌트 실행
Prometheus/Grafana 실행
Airflow/Spark Operator 실행
```

### 13-2) Kubernetes와 데이터 플랫폼의 충돌 지점

Kubernetes는 stateless 웹 애플리케이션에 매우 잘 맞는다. 하지만 데이터베이스, 분석 엔진, 분산 저장소 같은 Stateful workload에서는 추가 고려가 필요하다.

```text
Pod 재시작이 서비스 영향으로 이어짐
PVC와 Pod identity가 중요함
리밸런싱 비용이 큼
쿼리 중단 위험이 있음
캐시 손실이 큼
CPU/Memory 피크가 불규칙함
```

따라서 데이터 플랫폼에서는 HPA/Knative 같은 일반적인 웹 서비스용 확장 방식보다 VPA/Goldilocks/In-place Resize/운영 정책 기반 right-sizing이 더 현실적인 경우가 많다.

## 14. 최종 정리

Multi-Tenancy와 Data Platform Architecture의 핵심은 “무엇을 공유하고 무엇을 격리할 것인가”이다.

```text
Namespace: Kubernetes 리소스 격리
Tenant: 서비스/데이터 플랫폼 사용자 격리
Physical Isolation: 격리는 강하지만 리소스 효율 낮음
Logical Isolation: 리소스 효율은 높지만 권한/성능/장애 격리 설계 필요
Shared Cluster: 공통 클러스터를 여러 Tenant가 공유
Data Lake: 다양한 데이터를 대량 저장
Data Warehouse: 정제된 분석 데이터 저장
Data Lakehouse: Data Lake + Warehouse 관리 기능
Object Storage: Data Lake/Lakehouse의 주요 저장 계층
Metadata/Catalog: 데이터를 찾고 읽고 통제하기 위한 관리 계층
```