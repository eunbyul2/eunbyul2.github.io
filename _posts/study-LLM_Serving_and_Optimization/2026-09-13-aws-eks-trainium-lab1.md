---
layout: post
title: "[LLM Serving] AWS EKS + Trainium 환경 구성 - Lab 1"
date: 2026-09-13 01:00:00 +0900
categories: [LLM, LLMOps, AWS, Kubernetes]
tags: [LLM Serving, vLLM, AWS Trainium, EKS, Neuron, Helm, S3 CSI, Kubernetes]
published: true
---

## 1. 실습 개요

이번 Lab에서는 **vLLM + NxD 기반 LLM Serving 환경을 배포하기 전에 필요한 EKS 인프라를 구성**했다.

최종적으로 구성한 항목은 다음과 같다.

```text
Amazon EKS 1.33
└── Managed Node Group
    └── EC2 trn1.2xlarge
        └── AWS Trainium
            ├── Neuron Device Plugin
            └── Neuron Scheduler Extension

Amazon S3
└── Neuron / Model Cache Bucket
    └── Mountpoint for Amazon S3 CSI Driver
```

이번 Lab의 핵심은 단순히 EKS Worker Node를 추가하는 것이 아니라,  
**Kubernetes가 Trainium 자원을 인식하고 이후 vLLM Pod가 이를 사용할 수 있는 기반 환경을 구성하는 것**이다.

---

## 2. 실습 환경

Workshop에서 제공된 EC2 Instance에 SSH로 접속한 뒤 실습을 진행했다.

### Workshop EC2

```text
OS: Ubuntu 22.04
Instance Type: t3.2xlarge
```

이 EC2는 실제 LLM Inference를 수행하는 노드가 아니라,  
AWS CLI / kubectl / Helm 등을 사용해 EKS를 관리하는 **관리용 Workshop Instance** 역할을 한다.

실제 LLM Inference를 수행할 Worker Node는 이후 생성하는 `trn1.2xlarge`이다.

---

## 3. 필수 도구 설치

```bash
sudo apt update
sudo apt install -y python3-pip jq unzip
```

AWS CLI v2 설치:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
  -o "awscliv2.zip"

unzip awscliv2.zip

sudo ./aws/install --update
```

Helm 설치:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

설치 확인:

```bash
python3 --version
pip3 --version
jq --version
aws --version
helm version --short
kubectl version --client
```

실습 당시 확인된 버전은 다음과 같았다.

```text
Python 3.10.12
pip 22.0.2
jq 1.6
AWS CLI 2.36.44
Helm 3.22.0
kubectl Client v1.37.0
```

---

## 4. AWS 인증 상태 확인

```bash
aws sts get-caller-identity
```

Workshop EC2에는 IAM Role이 연결되어 있었고, 해당 Role을 통해 AWS API를 호출할 수 있었다.

```text
assumed-role/...WorkshopInstanceRole...
```

> 공개 블로그에는 AWS Account ID, ARN 등 식별 가능한 값은 마스킹했다.

---

## 5. EKS 실습 변수 설정

```bash
export AWS_REGION=us-west-2
export CLUSTER_NAME=ai-infra-summit-test-cluster
export EKS_VERSION=1.33
export INSTANCE_TYPE=trn1.2xlarge
export DESIRED_NODES=1
```

Neuron Optimized AMI는 직접 AMI ID를 하드코딩하지 않고  
**AWS Systems Manager Parameter Store**에서 현재 권장 AMI를 조회했다.

```bash
export WORKER_AMI=$(aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.33/amazon-linux-2023/x86_64/neuron/recommended/image_id \
  --region $AWS_REGION \
  --query "Parameter.Value" \
  --output text)
```

확인:

```bash
echo "$WORKER_AMI"
```

실습 당시:

```text
ami-0e08c07b0376ba3f8
```

AMI 정보를 추가로 확인했다.

```bash
aws ec2 describe-images \
  --image-ids "$WORKER_AMI" \
  --region "$AWS_REGION" \
  --query 'Images[0].{
    ImageId:ImageId,
    Name:Name,
    Description:Description,
    Architecture:Architecture,
    CreationDate:CreationDate
  }' \
  --output table
```

확인 결과:

```text
Name:
amazon-eks-node-al2023-x86_64-neuron-1.33-v20260903

Description:
EKS-optimized Kubernetes node based on Amazon Linux 2023
(k8s: 1.33.13, containerd: 2.*)
```

즉 일반 Amazon Linux AMI가 아니라  
**EKS + Neuron 환경에 최적화된 Amazon Linux 2023 AMI**를 사용했다.

---

## 6. EKS Cluster 연결

기존에 Workshop에서 생성된 EKS Cluster에 kubeconfig를 연결했다.

```bash
aws eks update-kubeconfig \
  --region "$AWS_REGION" \
  --name "$CLUSTER_NAME"
```

현재 Context 확인:

```bash
kubectl config current-context
```

Cluster 연결 확인:

```bash
kubectl cluster-info
kubectl version
```

확인 결과 Server Version은 다음과 같았다.

```text
Server Version: v1.33.13-eks-4cc7921
```

다만 Local kubectl Client는 v1.37이어서 다음 경고가 발생했다.

```text
Warning:
version difference between client (1.37)
and server (1.33)
exceeds the supported minor version skew of +/-1
```

현재 실습 명령은 정상 동작했지만, 운영 환경에서는 kubectl Client/Server Version Skew를 지원 범위 내로 맞추는 것이 적절하다.

---

## 7. EKS Endpoint 확인

EKS Cluster의 네트워크 설정을 확인했다.

```bash
aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.resourcesVpcConfig.{
    EndpointPublicAccess:endpointPublicAccess,
    EndpointPrivateAccess:endpointPrivateAccess,
    PublicAccessCidrs:publicAccessCidrs
  }' \
  --output json
```

결과:

```json
{
  "EndpointPublicAccess": false,
  "EndpointPrivateAccess": true,
  "PublicAccessCidrs": []
}
```

즉 이번 EKS API Endpoint는 **Private Endpoint**로 구성되어 있었다.

Workshop EC2가 같은 VPC 내부에 있기 때문에 kubectl을 통해 EKS API Server에 접근할 수 있었다.

---

## 8. Worker Node가 없는 초기 상태 확인

Node Group 생성 전:

```bash
kubectl get nodes
```

결과:

```text
No resources found
```

CoreDNS 상태:

```bash
kubectl get pods -n kube-system
```

```text
coredns-...   0/1   Pending
coredns-...   0/1   Pending
```

Pod 상세 상태를 확인했다.

```bash
kubectl describe pod \
  -n kube-system \
  coredns-75cb89d95b-km2w6
```

Event:

```text
FailedScheduling
no nodes available to schedule pods
```

EKS Control Plane은 존재하지만 Worker Node가 없기 때문에  
CoreDNS를 포함한 Pod를 스케줄링할 수 없는 상태였다.

```text
EKS Control Plane
        │
        └── Worker Node 없음
                │
                └── CoreDNS Pending
```

이후 Trainium Worker Node가 추가되면서 CoreDNS가 Running으로 전환되는 것을 확인했다.

---

## 9. Worker Node SSH Key 생성

Trainium Worker Node 접근용 SSH Key를 생성했다.

```bash
ssh-keygen -t rsa -b 4096 \
  -f ~/.ssh/id_rsa \
  -N ""
```

생성 결과:

```text
~/.ssh/id_rsa
~/.ssh/id_rsa.pub
```

Node Group에는 Public Key인 `id_rsa.pub`만 등록한다.

---

## 10. trn1.2xlarge 지원 Availability Zone 확인

AWS EC2 Instance Type은 모든 AZ에서 항상 제공되는 것이 아니므로  
`trn1.2xlarge`를 사용할 수 있는 AZ를 먼저 조회했다.

```bash
aws ec2 describe-instance-type-offerings \
  --region "$AWS_REGION" \
  --location-type availability-zone \
  --filters "Name=instance-type,Values=$INSTANCE_TYPE" \
  --query 'InstanceTypeOfferings[*].Location' \
  --output table
```

결과:

```text
us-west-2b
us-west-2d
```

즉 이번 Region에서는 `trn1.2xlarge`를 사용할 수 있는 AZ가 두 곳이었다.

---

## 11. Trainium 지원 AZ의 Public Subnet 선택

EKS VPC ID 조회:

```bash
VPC_ID=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.resourcesVpcConfig.vpcId' \
  --output text)
```

Trainium 지원 AZ를 배열로 저장:

```bash
SUPPORTED_AZS=($(aws ec2 describe-instance-type-offerings \
  --region "$AWS_REGION" \
  --location-type availability-zone \
  --filters "Name=instance-type,Values=$INSTANCE_TYPE" \
  --query 'InstanceTypeOfferings[*].Location' \
  --output text))
```

지원 AZ 내부의 Public Subnet 조회:

```bash
VALID_SUBNETS=()

for az in "${SUPPORTED_AZS[@]}"; do
  subnet=$(aws ec2 describe-subnets \
    --region "$AWS_REGION" \
    --filters \
    "Name=vpc-id,Values=$VPC_ID" \
    "Name=map-public-ip-on-launch,Values=true" \
    "Name=availability-zone,Values=$az" \
    --query 'Subnets[0].SubnetId' \
    --output text)

  [ "$subnet" != "None" ] && \
  [ "$subnet" != "" ] && \
  VALID_SUBNETS+=("$subnet")
done
```

결과:

```text
us-west-2d → subnet-08b018a95acc00045
us-west-2b → subnet-068e5c9051da134ef
```

결국 다음 조건을 만족하는 Subnet을 선택한 것이다.

```text
trn1.2xlarge 지원 AZ
        ∩
EKS VPC
        ∩
Public Subnet
        ↓
Managed Node Group 배치 대상
```

---

## 12. Trainium Managed Node Group 생성

Node Group 설정 파일을 작성했다.

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ai-infra-summit-test-cluster
  region: us-west-2
  version: "1.33"

vpc:
  id: <VPC_ID>
  subnets:
    public:
      us-west-2d:
        id: <PUBLIC_SUBNET_1>
      us-west-2b:
        id: <PUBLIC_SUBNET_2>

managedNodeGroups:
  - name: neuron-trn1-2x
    ami: <NEURON_OPTIMIZED_AMI>
    amiFamily: AmazonLinux2023

    subnets:
      - "<PUBLIC_SUBNET_1>"
      - "<PUBLIC_SUBNET_2>"

    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
        - arn:aws:iam::aws:policy/AmazonS3FullAccess
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

    instanceType: trn1.2xlarge
    desiredCapacity: 1

    volumeSize: 100
    volumeType: gp2

    ssh:
      allow: true
      publicKeyPath: ~/.ssh/id_rsa.pub
```

Node Group 생성:

```bash
eksctl create nodegroup \
  --config-file /home/ubuntu/eks_nodegroup.yaml
```

생성 과정에서 `eksctl`이 CloudFormation Stack을 생성하고  
EKS Managed Node Group을 구성했다.

```text
Managed Node Group
        ↓
EC2 trn1.2xlarge
        ↓
Neuron Optimized AMI
        ↓
EKS Worker Node 등록
```

---

## 13. Node Group 상태 확인

```bash
aws eks describe-nodegroup \
  --cluster-name "$CLUSTER_NAME" \
  --nodegroup-name neuron-trn1-2x \
  --region "$AWS_REGION" \
  --query 'nodegroup.{
    Status:status,
    Health:health,
    InstanceTypes:instanceTypes,
    Desired:scalingConfig.desiredSize
  }' \
  --output json
```

결과:

```json
{
  "Status": "ACTIVE",
  "Health": {
    "issues": []
  },
  "InstanceTypes": [
    "trn1.2xlarge"
  ],
  "Desired": 1
}
```

Kubernetes Node 확인:

```bash
kubectl get nodes -o wide
```

Trainium Worker Node가 `Ready` 상태로 EKS에 등록되었다.

CoreDNS도:

```text
Pending
→ Running
```

으로 변경되었다.

---

## 14. Neuron Device Plugin과 Trainium Resource 확인

`eksctl`을 통해 Accelerated AMI 기반 Node Group을 생성하면서  
Neuron Device Plugin이 자동 설치되었다.

DaemonSet 확인:

```bash
kubectl get ds neuron-device-plugin -n kube-system
```

Kubernetes에서 인식한 NeuronCore 확인:

```bash
kubectl get nodes \
  "-o=custom-columns=NAME:.metadata.name,NeuronCore:.status.allocatable.aws\.amazon\.com/neuroncore"
```

결과:

```text
NAME                                       NeuronCore
ip-10-0-5-132.us-west-2.compute.internal   2
```

Node Resource 상세 확인:

```bash
kubectl describe node | grep -A20 -E 'Capacity:|Allocatable:'
```

```text
Capacity:
  aws.amazon.com/neuron:      1
  aws.amazon.com/neuroncore:  2
  cpu:                        8
  memory:                     ...

Allocatable:
  aws.amazon.com/neuron:      1
  aws.amazon.com/neuroncore:  2
  cpu:                        7910m
  memory:                     ...
```

이 결과를 통해 Kubernetes가 Trainium 자원을  
**Extended Resource** 형태로 인식하고 있음을 확인할 수 있었다.

```text
Trainium Hardware
        ↓
Neuron Device Plugin
        ↓
kubelet
        ↓
Node Capacity / Allocatable
        ↓
aws.amazon.com/neuron
aws.amazon.com/neuroncore
```

이번 `trn1.2xlarge`에서는:

```text
Neuron Device: 1
NeuronCore: 2
```

가 노출되었다.

---

## 15. S3 Neuron Cache Bucket 생성

Neuron compile artifact 및 모델 캐시를 저장하기 위한 S3 Bucket을 생성했다.

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)

export BUCKET_NAME=ai-infra-summit-vllm-models-cache-${AWS_ACCOUNT_ID}
```

Bucket 생성:

```bash
aws s3 mb "s3://$BUCKET_NAME" \
  --region "$AWS_REGION"
```

확인:

```bash
aws s3 ls | grep ai-infra-summit-vllm-models-cache
```

S3 Bucket은 이후 Lab에서 Neuron compile 결과나 모델 캐시를 재사용하기 위한 저장소로 사용된다.

---

## 16. 기존 Neuron Component 정리

`eksctl`이 자동 설치한 Neuron Device Plugin을 제거한 뒤  
Helm으로 다시 관리하도록 구성했다.

```bash
kubectl delete daemonset neuron-device-plugin -n kube-system
kubectl delete clusterrole neuron-device-plugin
kubectl delete serviceaccount neuron-device-plugin -n kube-system
kubectl delete clusterrolebinding neuron-device-plugin
```

### 발생한 문제

처음에는 ServiceAccount 삭제를 빠뜨리고 Helm 설치를 진행해 다음 오류가 발생했다.

```text
ServiceAccount "neuron-device-plugin" in namespace "kube-system" exists
and cannot be imported into the current release:
invalid ownership metadata
```

기존 ServiceAccount에는 Helm 관리 metadata가 없었기 때문에  
Helm이 기존 리소스를 자신의 Release에 자동으로 포함할 수 없었다.

남은 ServiceAccount를 삭제했다.

```bash
kubectl delete serviceaccount neuron-device-plugin -n kube-system
```

기존 리소스가 모두 제거되었는지 확인:

```bash
kubectl get ds -n kube-system | grep neuron || true
kubectl get serviceaccount -n kube-system | grep neuron || true
kubectl get clusterrole | grep neuron || true
kubectl get clusterrolebinding | grep neuron || true
```

---

## 17. Helm으로 Neuron Device Plugin 재설치

```bash
helm upgrade --install neuron-helm-chart \
  oci://public.ecr.aws/neuron/neuron-helm-chart \
  --set "npd.enabled=false"
```

설치된 Chart:

```text
neuron-helm-chart-1.10.0
```

확인:

```bash
helm list -A
```

```text
NAME               NAMESPACE   REVISION   STATUS
neuron-helm-chart  default     1          deployed
```

DaemonSet:

```bash
kubectl get ds neuron-device-plugin -n kube-system
```

결과:

```text
DESIRED   CURRENT   READY   AVAILABLE
1         1         1       1
```

Pod:

```bash
kubectl get pods -n kube-system | grep neuron
```

```text
neuron-device-plugin-...   1/1   Running
```

NeuronCore도 다시 정상적으로 확인되었다.

```text
aws.amazon.com/neuroncore = 2
```

---

## 18. Neuron Scheduler Extension 설치

같은 Helm Release를 Upgrade하여 Scheduler Extension을 활성화했다.

```bash
helm upgrade --install neuron-helm-chart \
  oci://public.ecr.aws/neuron/neuron-helm-chart \
  --set "scheduler.enabled=true" \
  --set "npd.enabled=false"
```

Helm Revision:

```text
REVISION: 2
STATUS: deployed
```

관련 Pod 확인:

```bash
kubectl get pods -A | grep -Ei 'neuron|scheduler'
```

결과:

```text
k8s-neuron-scheduler-...   1/1   Running
my-scheduler-...           1/1   Running
neuron-device-plugin-...   1/1   Running
```

Deployment 확인:

```bash
kubectl get deployment -A | grep -Ei 'neuron|scheduler'
```

```text
k8s-neuron-scheduler   1/1
my-scheduler           1/1
```

### Device Plugin과 Scheduler의 역할 차이

```text
Neuron Device Plugin
= Trainium / Neuron Resource를 Kubernetes에 노출

Neuron Scheduler Extension
= Neuron Resource를 요청하는 Workload의 Scheduling을 보조
```

즉 Device Plugin은 **자원을 인식하게 만드는 역할**,  
Scheduler Extension은 **그 자원을 사용할 Pod의 배치를 지원하는 역할**로 볼 수 있다.

---

## 19. Mountpoint for Amazon S3 CSI Driver 설치

S3 Bucket을 Kubernetes Workload에서 사용할 수 있도록  
Mountpoint for Amazon S3 CSI Driver를 설치했다.

Helm Repository 추가:

```bash
helm repo add aws-mountpoint-s3-csi-driver \
  https://awslabs.github.io/mountpoint-s3-csi-driver

helm repo update
```

설치:

```bash
helm upgrade --install aws-mountpoint-s3-csi-driver \
  --namespace kube-system \
  aws-mountpoint-s3-csi-driver/aws-mountpoint-s3-csi-driver
```

설치 버전:

```text
Mountpoint for Amazon S3 CSI Driver v2.8.0
```

Pod 확인:

```bash
kubectl get pods \
  -n kube-system \
  -l app.kubernetes.io/name=aws-mountpoint-s3-csi-driver
```

결과:

```text
s3-csi-controller-...   1/1   Running
s3-csi-node-...         3/3   Running
```

구조는 다음과 같다.

```text
Amazon S3 Bucket
        ↓
Mountpoint for Amazon S3
        ↓
S3 CSI Driver
        ↓
Kubernetes Volume
        ↓
Pod
```

이 단계에서는 CSI Driver만 설치한 상태이며,  
실제 PV/PVC 구성은 이후 vLLM Deployment 과정에서 이어진다.

---

## 20. 최종 Cluster 상태 확인

### Node

```bash
kubectl get nodes
```

```text
NAME                                       STATUS
ip-10-0-5-132.us-west-2.compute.internal   Ready
```

### Neuron Resource

```bash
kubectl describe nodes \
  -l alpha.eksctl.io/nodegroup-name=neuron-trn1-2x \
  | grep "aws.amazon.com/neuron"
```

```text
aws.amazon.com/neuron:      1
aws.amazon.com/neuroncore:  2
```

### 전체 System Pod

```bash
kubectl get pods -A
```

최종적으로 다음 Pod들이 모두 `Running` 상태임을 확인했다.

```text
aws-node
coredns
kube-proxy
neuron-device-plugin
k8s-neuron-scheduler
my-scheduler
s3-csi-controller
s3-csi-node
```

---

## 21. Lab 1 최종 구성

Lab 1 완료 후 환경은 다음과 같다.

```text
EKS Kubernetes 1.33
│
├── CoreDNS
├── VPC CNI
├── kube-proxy
│
└── Managed Node Group
    │
    └── trn1.2xlarge
        │
        ├── Amazon Linux 2023 Neuron Optimized AMI
        ├── Trainium Device 1
        ├── NeuronCore 2
        │
        ├── Neuron Device Plugin
        │
        └── Neuron Scheduler Extension

Amazon S3
│
└── Model / Neuron Cache Bucket
    │
    └── Mountpoint for Amazon S3 CSI Driver
```

---

## 22. 핵심 정리

이번 Lab에서 가장 중요하게 확인한 부분은 다음과 같다.

### EKS Control Plane과 Worker Node는 별도이다

처음에는 EKS Control Plane만 존재했고 Worker Node가 없어서 CoreDNS가 `Pending` 상태였다.

Trainium Managed Node Group을 생성한 이후 Worker가 `Ready` 상태로 등록되면서 CoreDNS도 정상 실행되었다.

### Trainium은 Kubernetes에서 Extended Resource로 관리된다

Neuron Device Plugin이 설치되면 Kubernetes Node에 다음과 같은 Resource가 노출된다.

```text
aws.amazon.com/neuron
aws.amazon.com/neuroncore
```

이번 환경에서는:

```text
Neuron Device = 1
NeuronCore = 2
```

를 사용할 수 있었다.

### Neuron Device Plugin과 Scheduler Extension은 역할이 다르다

```text
Device Plugin
→ Trainium Resource를 Kubernetes에 등록

Scheduler Extension
→ 해당 Resource를 요청하는 Workload의 배치를 지원
```

### S3는 모델/컴파일 캐시 저장소로 사용한다

LLM Serving 환경에서는 모델 로딩 및 accelerator compile 과정에 시간이 필요할 수 있기 때문에  
S3 Bucket을 이용해 캐시를 영속적으로 보관하고 이후 재사용할 수 있도록 준비했다.

### Lab 1의 목적

Lab 1에서는 아직 vLLM이나 LLM 모델을 직접 Serving하지 않았다.

이번 단계의 목적은 이후 Lab에서:

```text
vLLM
  ↓
NxD
  ↓
Neuron
  ↓
Trainium
```

구조로 모델을 실행할 수 있도록 **Kubernetes + Trainium + Storage 기반 인프라를 준비하는 것**이었다.

다음 Lab에서는 이 환경 위에 실제 vLLM Workload를 배포한다.
