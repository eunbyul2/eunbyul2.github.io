---
layout: post
title: "[MLOps] Kubeflow Pipelines는 Kubernetes에서 어떻게 실행될까? — Argo Workflows와 Pod 실행 구조"
date: 2026-11-01 01:41:00 +0900
categories: [MLOps, Kubernetes, Kubeflow]
tags: [MLOps, Kubeflow Pipelines, KFP, Kubernetes, Argo Workflows, Workflow, CRD, Pod, Artifact, MLflow, Feast]
published: true
---

## 1. 들어가며

이번 MLOps 스터디의 Chapter 4에서는 MLflow로 모델 학습 결과를 추적하고 Feast로 피처를 정의하고 조회하는 방법을 살펴봤다. Chapter 5에서는 앞에서 준비한 모델과 피처를 연결하여 **Kubeflow Pipelines(KFP)** 로 배치 추론을 실행하는 과정을 다뤘다.

MLflow는 모델과 실험을 관리하고, Feast는 모델 입력에 사용할 피처를 조회한다. 그런데 실제 추론을 수행하려면 예측 대상 데이터를 읽고, 피처를 조회하고, 모델을 가져와 예측한 뒤 결과를 저장하는 여러 작업을 실행해야 한다. KFP는 이 작업을 재사용 가능한 컴포넌트로 나누고 의존성에 따라 실행하도록 관리한다.

이번 글에서는 KFP 사용법을 나열하기보다 **KFP에 파이프라인 실행을 요청하면 Kubernetes 내부에서는 무엇이 일어나는가?** 를 중심으로 정리한다. 특히 스터디에 공유된 KFP 2.4.1 실습 기록을 분석하여 Argo Workflow, 드라이버 Pod, 실제 실행 Pod, 아티팩트 저장 구조를 살펴본다. 아래에 제시하는 실행 결과는 **스터디 참고 실습 기록의 결과이며 필자가 동일한 환경에서 새롭게 측정한 결과는 아니다.**

## 2. KFP는 무엇이며 왜 필요한가?

KFP(Kubeflow Pipelines)는 머신러닝 작업을 여러 단계로 정의하고 실행 및 추적할 수 있도록 지원하는 파이프라인 플랫폼이다. 데이터 준비, 피처 추출, 모델 학습, 평가, 배치 추론 같은 작업을 컴포넌트로 분리할 수 있다.

예를 들어 다음 코드는 네 단계의 작업을 순서대로 실행한다.

```python
data = read_data()
features = retrieve_features(data)
predictions = run_inference(features)
write_predictions(predictions)
```

단순한 로컬 스크립트로도 실행할 수 있지만 실제 운영에서는 작업별 실행 환경, 실패 상태, 재시도, 기록과 데이터 전달을 별도로 관리할 필요가 있다. 특히 어떤 단계는 CPU를 많이 사용하고 다른 단계는 스토리지 I/O에 민감할 수 있다. 이를 모두 하나의 프로세스로 관리하면 각 단계의 장애와 리소스를 독립적으로 다루기 어렵다.

KFP에서 중요한 개념은 다음과 같다.

| 개념 | 의미 | 이번 스터디 예시 |
| --- | --- | --- |
| Component | 재사용 가능한 작업 정의. 실행 코드, 이미지, 입력과 출력 등을 포함 | `read-data-from-minio` |
| Task | 파이프라인에서 컴포넌트를 호출한 작업 인스턴스 | 피처 조회 Task |
| Pipeline | 여러 Task의 연결과 의존성을 정의한 워크플로 | 소득 분류 배치 추론 |
| Run | 파이프라인 정의를 실제 실행한 기록 | `income-classifier-run-1` |
| Artifact | 컴포넌트가 생성하거나 전달하는 파일 등의 결과물 | Parquet 데이터셋 |

이 중 **Component는 실행할 작업의 정의, Pipeline은 작업 사이의 관계, Run은 실제 실행**이라는 차이를 구분해야 한다.

## 3. Kubernetes와 Argo Workflows, KFP의 관계

Kubernetes는 Pod를 배치하고 컨테이너를 실행하며 리소스 상태를 관리한다. 하지만 Pod 몇 개를 만들었다고 해서 '첫 번째 작업이 완료되면 두 번째 작업을 실행하고 이전 결과를 다음 작업에 전달한다'는 머신러닝 워크플로가 저절로 구성되지는 않는다.

이번 스터디의 KFP Standalone 2.4.1 환경에서는 **Argo Workflows**가 워크플로 실행 엔진으로 사용된다. Argo Workflows는 Kubernetes의 CRD(CustomResourceDefinition)를 이용해 `Workflow`라는 사용자 정의 리소스를 제공하고, Workflow Controller가 정의된 작업의 실행을 관리한다.

```text
KFP Python SDK
    │  컴포넌트·파이프라인 정의
    ▼
Pipeline YAML 컴파일
    │
    ▼
KFP API / Run 생성
    │
    ▼
Argo Workflow (Kubernetes CR)
    │  Workflow Controller가 작업 진행 관리
    ▼
Kubernetes Pod 생성 및 컨테이너 실행
    │
    ▼
작업 결과·아티팩트·실행 상태 기록
```

따라서 KFP와 Argo Workflows, Kubernetes는 같은 기능을 수행하는 도구가 아니라 서로 다른 계층을 맡는다.

- **KFP:** 머신러닝 파이프라인의 정의, 제출, 실행 기록과 아티팩트 관리
- **Argo Workflows:** 이번 구성에서 작업 순서와 상태를 실제로 실행하는 워크플로 엔진
- **Kubernetes:** Pod 스케줄링, 컨테이너 실행과 리소스 관리

여기서 **Argo CD와 Argo Workflows도 구분**해야 한다. Argo CD는 Git에 선언된 Kubernetes 리소스 상태를 클러스터에 반영하는 GitOps 도구이고, Argo Workflows는 작업 실행을 담당한다. 이번 KFP Standalone 실습에서 Argo CD를 필수 실행 엔진으로 사용한 것은 아니다.

## 4. Chapter 5의 실제 배치 추론 파이프라인

스터디 실습은 Chapter 4에서 등록한 MLflow 모델과 Feast 피처 정의를 사용한다. **Chapter 5 파이프라인 자체가 모델을 다시 학습하지는 않는다.**

```text
S3 호환 저장소: entity_df.parquet
                 │
                 ▼
       read-data-from-minio
           예측 대상 읽기
                 │ Dataset Artifact
                 ▼
   retrieve-features-from-feast
       과거 시점 피처 조회
                 │ Dataset Artifact
                 ▼
          model-inference
      MLflow 모델 로드 및 예측
                 │ Dataset Artifact
                 ▼
        write-data-to-minio
                 │
                 ▼
 S3: inference-datasets/income_predictions.parquet
```

각 단계의 역할은 다음과 같다.

| 컴포넌트 | 입력 | 수행하는 일 | 출력 |
| --- | --- | --- | --- |
| `read-data-from-minio` | S3 버킷·파일명 | 예측 대상 데이터를 다운로드 | 엔티티 데이터셋 |
| `retrieve-features-from-feast` | 엔티티·Feature 목록 | Feast의 `get_historical_features()`로 시점에 맞는 피처 조회 | 피처 데이터셋 |
| `model-inference` | 피처·모델 이름·Stage | MLflow의 모델과 전처리용 컬럼 목록을 로드하여 예측 | 예측 데이터셋 |
| `write-data-to-minio` | 예측 데이터·대상 버킷 | 최종 결과 업로드 | Parquet 객체 |

여기서 KFP는 업무 데이터 자체를 생성하는 모델 학습 프레임워크가 아니다. KFP는 **어떤 컴포넌트를 언제 실행하고 앞 단계의 어떤 출력을 다음 단계에 전달할 것인지**를 관리한다.

## 5. Python 코드가 Pipeline YAML로 컴파일되는 과정

KFP Python SDK에서는 각 컴포넌트를 로드하고, 출력 참조를 이용해 Task의 의존성을 정의한다. 아래는 스터디에 나온 구조를 간결하게 표현한 예시다.

```python
fetch_data_task = fetch_data_op(...)
retrieve_features_task = retrieve_features_op(
    entity_df=fetch_data_task.outputs["data_output"],
    ...
)
run_inference_task = run_inference_op(
    input_data=retrieve_features_task.outputs["data_output"],
    ...
)
write_data_task = write_data_op(
    input_data=run_inference_task.outputs["data_output"],
    ...
)
```

`fetch_data_task.outputs["data_output"]`은 앞 Task의 출력이 다음 Task의 입력이라는 뜻이다. KFP는 이 참조를 이용해 실행 의존성을 표현한다.

이후 다음과 같이 파이프라인을 컴파일한다.

```python
from kfp import compiler

compiler.Compiler().compile(
    pipeline_func=income_classifier_pipeline,
    package_path="income_classifier_pipeline.yaml",
)
```

컴파일 결과 YAML에는 컴포넌트 정의, 입력·출력 파라미터, 이미지 참조, 작업 의존성과 실행 명세 등이 포함된다. **컴파일만으로 Kubernetes Pod가 생성되지는 않는다.** 컴파일된 YAML을 KFP에 등록하고 Run을 생성한 후에야 실제 작업 실행이 시작된다.

스터디 자료에서는 네 컴포넌트 모두 `book-ch5/components:0.1` 이미지를 사용하도록 변경했다. 하나의 이미지에서 작업별 Python 실행 파일만 다르게 호출한 것이다. 따라서 '네 컴포넌트'가 반드시 '네 이미지'라는 뜻은 아니다.

## 6. Run을 실행하면 왜 네 개보다 많은 Pod가 생성될까?

이번 글에서 가장 살펴볼 부분이다. 파이프라인은 네 개의 사용자 컴포넌트로 정의되어 있지만, 공유된 KFP 2.4.1 실행 기록에는 이를 실행하기 위한 **드라이버 Pod와 실제 실행 Pod**가 별도로 나타난다.

자료에서 확인된 Pod 이름의 예시는 다음과 같다.

```text
income-classifier-pipeline-65js8-system-dag-driver-...
income-classifier-pipeline-65js8-system-container-driver-...
income-classifier-pipeline-65js8-system-container-impl-...
```

이 Pod들은 모두 같은 일을 하는 것은 아니다.

### 6.1 Root DAG Driver

`system-dag-driver`는 전체 DAG(Directed Acyclic Graph) 실행을 준비·관리하는 KFP 내부 드라이버 작업이다. DAG는 작업의 방향과 의존성을 표현하고 순환 의존성이 없는 그래프다.

제공된 실행 기록에서 Root DAG Driver의 주요 이미지로 `ghcr.io/kubeflow/kfp-driver:2.4.1`이 확인되었다. 해당 Pod에서 사용자 데이터 처리 스크립트를 실행하는 것이 아니라 전체 파이프라인 실행과 하위 작업을 위한 준비에 관여한다.

### 6.2 Container Driver

`system-container-driver`는 개별 사용자 컴포넌트의 실행 준비를 수행한다. 스터디 기록에서는 입력 파라미터, 실행 메타데이터, 캐시 판단, 출력 아티팩트 위치 결정 및 작업 Pod에 필요한 실행 설정을 준비하는 내용이 확인되었다.

예를 들어 `read-data-from-minio`를 위한 Container Driver는 다음 실행 단계에서 사용할 이미지와 명령을 준비한다.

```text
image: book-ch5/components:0.1
command: /kfp-launcher/launch ... --
         python3 /app/src/read_data/read_data.py ...
```

따라서 **Driver Pod가 존재한다고 해서 그 Pod에서 사용자 Python 코드가 직접 실행되는 것은 아니다.**

### 6.3 Implementation Pod

`system-container-impl`은 실제 컴포넌트 구현이 실행되는 Pod다. 이번 실습에서 `read_data.py`와 같은 사용자 코드가 실행되는 위치가 이 단계다.

실행 기록에는 KFP Launcher와 함께 사용자 코드가 호출되는 명령이 나온다.

```text
/kfp-launcher/launch --pipeline_name ... --run_id ... --
python3 /app/src/read_data/read_data.py \
  --minio_host ... \
  --bucket_name ... \
  --file_name ... \
  --data_output_path ...
```

KFP Launcher는 실행에 필요한 입력·출력 경로를 설정하고 사용자 코드의 결과 및 아티팩트 메타데이터를 KFP의 관리 체계에 연결하는 역할을 한다. 자료에서는 Argo의 `argoexec` 컨테이너가 작업 준비 및 로그 수집 등에 관여하는 것도 확인된다.

정리하면 아래처럼 이해할 수 있다.

| Pod 종류 | 역할 | 사용자 Python 코드 실행 여부 |
| --- | --- | --- |
| Root DAG Driver | 파이프라인 전체 DAG 실행 준비·관리 | 아니오 |
| Container Driver | 개별 Task 실행 준비·캐시 및 아티팩트 위치 관리 | 아니오 |
| Implementation Pod | 작업 컨테이너에서 컴포넌트 코드 실행 | 예 |

이것이 **컴포넌트 네 개보다 Pod가 더 많이 관찰되는 이유**다. 단, 이 구조는 제공된 KFP 2.4.1 구성의 실행 기록에 기반하며 다른 버전이나 백엔드의 구현까지 동일하다고 일반화할 수 없다.

## 7. 서로 다른 Pod 사이에서 데이터는 어떻게 전달될까?

Kubernetes의 서로 다른 Pod는 같은 Python 프로세스의 메모리를 공유하지 않는다. 따라서 앞 작업에서 만든 DataFrame을 다음 Pod로 전달하려면 별도 저장 및 전달 메커니즘이 필요하다.

이번 구성에서는 KFP가 관리하는 **Artifact**가 그 역할을 한다. 컴포넌트가 출력 경로에 파일을 작성하면 KFP 실행 구성 요소가 이를 관리 저장소에 저장하고 다음 컴포넌트가 해당 데이터를 입력으로 사용할 수 있도록 한다.

실습에 등장하는 두 종류의 저장 경로를 구분해야 한다.

| 저장 위치 | 용도 |
| --- | --- |
| `mlpipeline` 버킷 | KFP 작업 사이의 중간 아티팩트와 실행 관련 파일 |
| `inference-datasets` 버킷 | 최종 업무 예측 결과 |

참고 실행 기록의 중간 아티팩트 경로는 아래와 같은 형태로 나타난다.

```text
minio://mlpipeline/v2/artifacts/
  income-classifier-pipeline/<run-id>/
  read-data-from-minio/<uuid>/data_output
```

여기서 `minio://`는 KFP가 표현하는 아티팩트 위치 형식이며, 실제 실습에서는 S3 호환 저장소 구현을 SeaweedFS로 교체했다. 기존 `minio-service` 서비스 이름을 유지했다고 해서 실행 중인 제품까지 반드시 MinIO인 것은 아니다.

KFP의 `Input[Dataset]`, `Output[Dataset]` 같은 타입 선언은 아티팩트 전달을 위한 인터페이스를 표현한다. 다만 **열 이름, 실제 컬럼 타입, 결측 여부 등 업무 데이터의 상세 스키마를 자동으로 검증한다는 의미는 아니다.** 그러한 검증은 별도의 코드나 데이터 검증 절차가 필요하다.

## 8. Kubernetes 관점에서 파이프라인 실행을 확인하는 방법

참고 실습에서는 KFP UI에서 Run을 시작한 뒤 Kubernetes의 Workflow와 Pod 상태를 함께 확인했다.

```bash
# Argo Workflow 확인
kubectl get workflows.argoproj.io -n kubeflow

# 컴포넌트 실행 관련 Pod를 생성 순서대로 확인
kubectl get pods -n kubeflow \
  -l pipelines.kubeflow.org/v2_component=true \
  --sort-by=.metadata.creationTimestamp

# Workflow 상세 명세 확인
kubectl get workflows.argoproj.io -n kubeflow -o yaml

# 특정 작업 Pod 상세 분석
kubectl describe pod <pod-name> -n kubeflow

# 특정 Job/Pod의 로그 확인 (실제 리소스 이름으로 변경)
kubectl logs -n kubeflow <pod-name> -c main
```

자료의 예시 Workflow는 최종적으로 다음 상태를 보였다.

```text
NAME                                STATUS
income-classifier-pipeline-65js8    Succeeded
```

하지만 **Workflow의 `Succeeded` 상태와 예측 품질이 검증되었다는 사실은 다르다.** 실습에서는 마지막으로 `inference-datasets/income_predictions.parquet` 파일의 생성 여부도 확인했다.

해당 가이드에서 기록된 파일 검증 출력은 다음과 같다.

```text
rows, columns: (30162, 92)
user_id in output: False
```

30,162개 행의 결과 파일이 생성된 점은 확인되지만, `user_id`가 결과에 남지 않은 점은 운영 관점의 보완 항목이다. 모델의 입력 피처에서 사용자 ID를 제외하는 것과 예측 결과에서 사용자 식별 정보를 보존하는 것은 다른 문제다. 결과의 사용자 식별, 기준 시각, 모델 버전, 피처 정의 정보를 남겨야 이후 추적과 재현이 용이하다.

## 9. 인프라 운영 관점에서 추가로 생각할 점

### 9.1 컨테이너 이미지와 노드 이미지 저장소

스터디 환경은 kind이므로 호스트 Docker에서 이미지를 빌드해도 Kubernetes 노드 내부의 containerd에 해당 이미지가 자동으로 존재하는 것은 아니다. 자료에서는 `python bookenv.py load book-ch5/components:0.1`로 이미지를 kind에 적재했다. `imagePullPolicy: Never` 설정에서 이미지를 적재하지 않으면 `ErrImageNeverPull`이 발생할 수 있다.

### 9.2 서비스 주소와 포트포워딩

로컬 PC에서 포트포워딩으로 접속할 때의 `localhost:5000`과 Kubernetes Pod에서 MLflow를 조회할 때의 주소는 다르다. Pod는 클러스터 내부 DNS 주소인 `http://mlflow-service.mlflow.svc.cluster.local:5000`을 사용한다. Pod 내부의 `localhost`는 해당 Pod의 네트워크 네임스페이스를 의미한다.

### 9.3 재현성과 실행 환경 버전

이번 실습은 모델을 저장한 환경과 추론 이미지의 `scikit-learn` 버전을 1.5.2로 맞추었다. 모델 파일이 존재해도 직렬화·역직렬화에 사용한 라이브러리가 호환되지 않으면 모델 로드에 실패할 수 있다. 모델 버전뿐 아니라 코드, 이미지, 라이브러리와 피처 정의도 함께 관리해야 한다.

### 9.4 캐싱과 재시도

참고 자료에서는 Container Driver 로그에서 `cached-decision=false`를 확인했다. 이는 해당 실행에서 캐시 재사용이 일어나지 않았다는 의미다. 캐싱이 활성화되어 있고 조건이 충족되면 이전 결과를 활용할 수 있지만, **같은 입력으로 재실행한다고 언제나 캐시가 적중하는 것은 아니다.** 입력, 실행 설정, 컴포넌트 명세, 캐시 정책 등을 확인해야 한다. 또한 재시도 가능한 일시적인 저장소 오류와 입력 스키마 오류처럼 재시도만으로 해결되지 않는 문제를 구분해야 한다.

### 9.5 자격 증명 관리

가이드의 실습용 Run 파라미터에는 S3 접근 키가 일반 문자열로 전달된다. 이는 학습용 환경에서만 허용할 수 있는 방식이며 운영 환경에서는 Secret, 적절한 ServiceAccount 및 접근 제어 등으로 개선해야 한다.

## 10. 정리

이번 Chapter 5의 핵심은 머신러닝 파이프라인을 Kubernetes에서 **어떤 구조로 실행하고 관리하는지** 이해하는 것이다.

KFP에서 네 개의 컴포넌트를 정의해도 실제 Kubernetes에서는 파이프라인 전체를 준비하는 Root DAG Driver, 개별 작업을 준비하는 Container Driver, 사용자 코드를 실행하는 Implementation Pod 등 내부 실행 리소스가 추가로 생성될 수 있다. 또 컴포넌트는 서로 다른 Pod에서 실행되기 때문에 중간 결과는 Artifact 저장소를 통해 전달된다.

이 구조를 이해하면 KFP UI에서 단순히 `Succeeded` 또는 `Failed`를 확인하는 데서 그치지 않고, 실제 실패한 계층이 **Workflow 제어, Pod 스케줄링, 이미지 실행, 사용자 코드, 서비스 통신, 아티팩트 저장** 중 어디인지 구분하는 데 도움이 된다.

결국 KFP는 머신러닝 알고리즘 자체가 아니라, **머신러닝을 이루는 여러 작업을 Kubernetes 위에서 재사용 가능하고 추적 가능한 형태로 실행하도록 연결하는 플랫폼**으로 이해할 수 있다.
