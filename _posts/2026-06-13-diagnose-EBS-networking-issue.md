---
layout: post
title: "클라우드 환경에서 EBS 같은 네트워크 스토리지 성능 이슈는 어떻게 진단할 수 있는지?"
date: 2026-06-13 23:00:00 +0900
categories: []
tags: []
published: true
---
# 클라우드 환경에서 EBS 같은 네트워크 스토리지 성능 이슈는 어떻게 진단할 수 있는가?

클라우드에서 사용하는 EBS 같은 네트워크 기반 블록 스토리지는 겉으로는 일반 디스크처럼 보이지만, 실제로는 EC2 인스턴스와 스토리지 서비스 사이의 네트워크 경로를 통해 I/O가 처리된다. 그래서 성능 문제가 발생했을 때 단순히 “디스크가 느리다”고 판단하면 안 된다.

EBS 성능 문제는 보통 다음 중 하나에서 발생한다.

- EBS 볼륨 자체의 IOPS 한계
- EBS 볼륨 자체의 Throughput 한계
- gp2, st1, sc1 같은 크레딧 기반 볼륨의 BurstBalance 고갈
- EC2 인스턴스의 EBS 대역폭 한계
- 스냅샷에서 복원한 볼륨의 초기화 지연
- 파일시스템 또는 마운트 설정 문제
- 애플리케이션의 과도한 I/O 발생
- 데이터베이스 쿼리, 로그, 백업, 배치 작업 등 상위 계층 문제
- Nitro/NVMe 계층의 I/O timeout 또는 지연
- 실제 EBS 상태 이상 또는 AWS 측 이벤트

따라서 EBS 성능 이슈 진단의 핵심은 **볼륨, 인스턴스, 운영체제, 애플리케이션 계층을 순서대로 분리해서 보는 것**이다.

---

## 1. EBS 성능 진단을 위해 먼저 알아야 하는 개념

### 1-1) EBS는 EC2에 붙는 네트워크 기반 블록 스토리지다

EBS는 EC2 인스턴스에 연결해서 사용하는 블록 스토리지다. EC2 안에서는 `/dev/nvme1n1`, `/dev/xvdf` 같은 디스크처럼 보이지만, 물리적으로 EC2 내부 로컬 디스크가 아니라 AWS의 스토리지 인프라에 존재하는 네트워크 기반 스토리지다.

즉, EBS 성능은 다음 요소에 의해 결정된다.

- EBS 볼륨 유형
- 볼륨 크기
- 프로비저닝된 IOPS
- 프로비저닝된 Throughput
- EC2 인스턴스의 EBS 대역폭
- EC2 인스턴스가 Nitro 기반인지 여부
- 워크로드의 I/O 패턴
- 파일시스템
- 애플리케이션의 I/O 요청 방식

그래서 EBS 장애를 진단할 때는 “볼륨만” 보면 안 되고, EC2와 OS까지 같이 봐야 한다.

### 1-2) IOPS

IOPS는 `Input/Output Operations Per Second`의 약자다.

즉, 1초 동안 처리할 수 있는 읽기/쓰기 작업 횟수다.

예를 들어 다음과 같은 작업은 IOPS 영향을 많이 받는다.

- 데이터베이스 랜덤 읽기
- 작은 파일 다수 읽기/쓰기
- 메타데이터 접근
- 트랜잭션 처리
- 로그 인덱스 접근
- Elasticsearch 같은 검색 엔진의 랜덤 I/O

IOPS가 부족하면 다음 증상이 나타난다.

- DB 응답 지연
- API 응답 시간 증가
- `iostat`의 `await` 증가
- CloudWatch의 `VolumeQueueLength` 증가
- 애플리케이션 timeout 발생

### 1-3) Throughput

Throughput은 초당 전송 가능한 데이터 양이다.

보통 단위는 다음과 같다.

- MiB/s
- MB/s

다음과 같은 작업은 Throughput 영향을 많이 받는다.

- 대용량 파일 복사
- 백업
- 로그 아카이빙
- 데이터 웨어하우스 스캔
- ETL
- 대용량 순차 읽기/쓰기

Throughput이 부족하면 다음 증상이 나타난다.

- 백업 시간이 길어짐
- 대용량 파일 처리 속도 저하
- 로그 처리 지연
- 배치 작업 지연
- CloudWatch의 `VolumeThroughputPercentage`가 100%에 근접

### 1-4) Latency

Latency는 I/O 요청이 발생한 후 완료될 때까지 걸리는 시간이다.

보통 ms 단위로 본다.

예를 들어 애플리케이션이 디스크에 데이터를 쓰고 `fsync`를 호출했는데 응답까지 50ms가 걸린다면, 그 I/O의 지연 시간이 50ms인 것이다.

Latency가 증가하면 사용자는 다음과 같이 느낀다.

- 웹 페이지가 느리다
- DB 쿼리가 느리다
- API 응답이 느리다
- 배치가 오래 걸린다

성능 진단에서 Latency는 매우 중요하다. IOPS나 Throughput이 높아도 Latency가 튀면 애플리케이션은 느려질 수 있다.

### 1-5) Queue Length

Queue Length는 처리되지 못하고 대기 중인 I/O 요청 수다.

스토리지가 현재 들어오는 요청을 즉시 처리하지 못하면 요청이 큐에 쌓인다. EBS에서는 CloudWatch의 `VolumeQueueLength`로 확인할 수 있고, Linux에서는 `iostat`의 `aqu-sz` 또는 `avgqu-sz` 계열 지표로 볼 수 있다.

Queue Length가 지속적으로 높으면 다음 중 하나를 의심한다.

- 볼륨 IOPS 부족
- 볼륨 Throughput 부족
- 인스턴스 EBS 대역폭 부족
- 스토리지 지연 증가
- 애플리케이션에서 과도한 동시 I/O 발생

### 1-6) I/O Size

I/O Size는 한 번의 I/O 요청에서 읽거나 쓰는 데이터 크기다.

예를 들어

- 4KiB 랜덤 읽기
- 16KiB 랜덤 쓰기
- 1MiB 순차 읽기

는 서로 다른 I/O 패턴이다.

작은 I/O가 많으면 IOPS가 중요하고, 큰 I/O가 많으면 Throughput이 중요하다.

특히 st1, sc1 같은 HDD 계열 EBS는 대용량 순차 I/O에 최적화되어 있다. 작은 랜덤 I/O가 많으면 비효율적이다.

### 1-7) Burst Credit

일부 EBS 볼륨은 크레딧 기반으로 동작한다.

대표적으로 다음 볼륨이 해당된다.

- gp2
- st1
- sc1

이 볼륨들은 평소에 크레딧을 쌓아두었다가 순간적으로 더 높은 성능을 낼 수 있다. 하지만 크레딧이 고갈되면 기준 성능으로 떨어진다.

CloudWatch에서는 `BurstBalance`로 확인한다.

`BurstBalance = 0%`에 가까워지면 다음 현상이 나타날 수 있다.

- 갑자기 디스크 성능 저하
- DB 지연 증가
- Queue Length 증가
- await 증가

---

## 2. EBS 볼륨 유형별 성능 특성 이해

### 2-1) gp3

gp3는 현재 일반적인 신규 구성에서 가장 많이 사용하는 범용 SSD 볼륨이다.

핵심 특징은 다음과 같다.

- 기본 3,000 IOPS 제공
- 기본 125MiB/s Throughput 제공
- 볼륨 크기와 성능을 독립적으로 설정 가능
- BurstBalance 개념이 없음
- gp2보다 예측 가능한 성능 제공

gp3에서 성능 이슈가 나면 보통 다음을 확인한다.

- 프로비저닝한 IOPS를 다 쓰고 있는지
- 프로비저닝한 Throughput을 다 쓰고 있는지
- EC2 인스턴스의 EBS 대역폭이 부족한지
- 애플리케이션 I/O 패턴이 볼륨 성능을 초과하는지

### 2-2) gp2

gp2는 이전 세대 범용 SSD 볼륨이다.

특징은 다음과 같다.

- 볼륨 크기에 따라 기준 IOPS가 결정됨
- GiB당 3 IOPS
- 작은 볼륨은 기준 IOPS가 낮음
- Burst Credit을 사용하여 일시적으로 3,000 IOPS까지 버스트 가능
- BurstBalance 고갈 시 성능이 기준 성능으로 떨어짐

gp2에서 성능 이슈가 나면 반드시 `BurstBalance`를 확인해야 한다.

예를 들어 100GiB gp2는 기준 IOPS가 약 300 IOPS다. 순간적으로 3,000 IOPS까지 버스트할 수 있지만, 크레딧이 고갈되면 다시 300 IOPS 수준으로 떨어질 수 있다.

이 경우 해결 방법은 보통 다음 중 하나다.

- gp3로 변경
- 볼륨 크기 증가
- io2로 변경
- 애플리케이션 I/O 최적화

### 2-3) io1 / io2

io1, io2는 프로비저닝된 IOPS SSD 볼륨이다.

특징은 다음과 같다.

- 높은 IOPS 제공
- 낮은 지연 시간
- 고성능 데이터베이스 워크로드에 적합
- Oracle, SAP HANA, Microsoft SQL Server, 대형 MySQL/PostgreSQL 등에 적합
- io2는 io1보다 내구성과 성능 측면에서 더 강력함

io2 Block Express는 매우 높은 IOPS와 낮은 지연 시간이 필요한 미션 크리티컬 워크로드에 적합하다.

### 2-4) st1

st1은 Throughput Optimized HDD다.

특징은 다음과 같다.

- IOPS보다 Throughput 중심
- 대용량 순차 읽기/쓰기 적합
- 로그 처리
- 데이터 웨어하우스
- EMR
- ETL
- 부트 볼륨 불가

st1은 작은 랜덤 I/O에는 부적합하다.

### 2-5) sc1

sc1은 Cold HDD다.

특징은 다음과 같다.

- 가장 저렴한 HDD 계열
- 자주 접근하지 않는 데이터용
- 대용량 순차 콜드 데이터에 적합
- 부트 볼륨 불가
- 성능보다 비용이 중요한 경우 사용

sc1에 DB를 올리는 것은 거의 잘못된 설계다.

### 2-6) 볼륨 타입 선택과 성능 진단의 관계

성능 이슈를 진단할 때 가장 먼저 확인해야 하는 것은 볼륨 타입이다.

같은 “EBS가 느리다”라도 원인은 볼륨 타입마다 다르다.

| 볼륨 유형 | 주요 병목 | 핵심 확인 지표 |
|---|---|---|
| gp3 | IOPS, Throughput, EC2 EBS 대역폭 | VolumeIOPSPercentage, VolumeThroughputPercentage, VolumeQueueLength |
| gp2 | Burst Credit, 기준 IOPS | BurstBalance, VolumeQueueLength |
| io1/io2 | 프로비저닝 성능, EC2 한계 | IOPS/Throughput 사용률, Latency |
| st1 | Throughput, BurstBalance, I/O Size | BurstBalance, VolumeReadBytes, VolumeWriteBytes |
| sc1 | Throughput, 낮은 기준 성능 | BurstBalance, Throughput |
| standard | 이전 세대 성능 한계 | 교체 검토 |

---

## 3. 성능 이슈 발생 시 전체 진단 흐름

### 3-1) 1단계: 사용자가 말하는 “느리다”를 구체화한다

먼저 증상을 분리해야 한다.

확인할 질문은 다음과 같다.

- 전체 서버가 느린가?
- 특정 서비스만 느린가?
- DB만 느린가?
- 읽기만 느린가?
- 쓰기만 느린가?
- 특정 시간대에만 느린가?
- 배치나 백업 시간과 겹치는가?
- 최근 볼륨 타입이나 크기를 변경했는가?
- 최근 스냅샷에서 복원했는가?
- 최근 배포나 설정 변경이 있었는가?

이 질문 없이 바로 CloudWatch만 보면 원인을 잘못 판단할 수 있다.

### 3-2) 2단계: EBS 문제인지 아닌지 먼저 구분한다

스토리지 문제가 아니어도 사용자는 “느리다”고 표현한다.

먼저 다음을 확인한다.

```bash
uptime
free -h
top
vmstat 1
df -hT
```

확인 포인트는 다음과 같다.

- CPU 사용률이 높은가?
- Load Average가 비정상적으로 높은가?
- 메모리가 부족한가?
- Swap이 발생하는가?
- 디스크가 꽉 찼는가?
- 파일시스템이 read-only로 바뀌었는가?

스토리지 병목이 의심되면 다음 단계로 넘어간다.

### 3-3) 3단계: CloudWatch에서 EBS 볼륨 지표를 확인한다

CloudWatch에서 볼륨 단위 지표를 확인한다.

중요 지표는 다음과 같다.

- VolumeReadOps
- VolumeWriteOps
- VolumeReadBytes
- VolumeWriteBytes
- VolumeQueueLength
- VolumeIdleTime
- VolumeTotalReadTime
- VolumeTotalWriteTime
- BurstBalance
- VolumeThroughputPercentage
- VolumeIOPSPercentage
- VolumeStalledIOCheck
- VolumeStatusCheckFailed

### 3-4) 4단계: EC2 인스턴스 한계를 확인한다

EBS가 충분해도 EC2가 감당하지 못하면 성능이 안 나온다.

확인할 항목은 다음과 같다.

- 인스턴스 타입
- EBS Optimized 여부
- 최대 EBS 대역폭
- 최대 EBS IOPS
- 최대 EBS Throughput
- Nitro 기반 여부
- EBSIOBalance%
- EBSByteBalance%

### 3-5) 5단계: OS 내부에서 실시간 I/O 상태를 확인한다

CloudWatch는 1분 단위 지표가 많아서 순간적인 스파이크를 놓칠 수 있다.

Linux에서는 다음 명령어를 사용한다.

```bash
iostat -xz 1
```

추가로 다음 명령어도 본다.

```bash
iotop -o
pidstat -d 1
sar -d 1
dmesg -T
journalctl -k
```

### 3-6) 6단계: 애플리케이션 레벨 원인을 확인한다

EBS가 느린 것이 아니라 애플리케이션이 과도한 I/O를 만들고 있을 수 있다.

확인할 예시는 다음과 같다.

- MySQL slow query
- PostgreSQL checkpoint
- Elasticsearch merge
- Kafka flush
- Redis RDB/AOF 저장
- 백업 작업
- 로그 폭증
- 배치 작업
- 컨테이너 로그 적재
- Kubernetes PVC 사용량 증가

---

## 4. CloudWatch EBS 지표 해석

### 4-1) VolumeReadOps / VolumeWriteOps

읽기/쓰기 작업 횟수다.

이 값은 IOPS를 이해하는 데 사용한다.

예를 들어 1분 동안 `VolumeReadOps`가 60,000이면 평균 읽기 IOPS는 다음과 같다.

```text
60,000 / 60 = 1,000 IOPS
```

쓰기 작업도 동일하게 계산할 수 있다.

### 4-2) VolumeReadBytes / VolumeWriteBytes

읽거나 쓴 데이터 양이다.

Throughput을 계산할 때 사용한다.

예를 들어 1분 동안 `VolumeReadBytes`가 6GiB라면 평균 읽기 Throughput은 다음과 같이 계산한다.

```text
6GiB / 60초 = 약 102.4MiB/s
```

### 4-3) VolumeQueueLength

처리 대기 중인 I/O 요청 수다.

해석은 다음과 같다.

- 일시적으로 증가했다가 줄어듦: 순간 부하 가능성
- 지속적으로 높음: 병목 가능성
- IOPS/Throughput 사용률도 높음: 볼륨 성능 한계 가능성
- IOPS/Throughput은 낮은데 Queue가 높음: Latency, 인스턴스, OS, 파일시스템 문제 가능성

### 4-4) VolumeTotalReadTime / VolumeTotalWriteTime

읽기/쓰기 작업에 걸린 총 시간이다.

Ops와 함께 평균 지연 시간을 계산할 수 있다.

예시

```text
평균 읽기 지연 시간 = VolumeTotalReadTime / VolumeReadOps
평균 쓰기 지연 시간 = VolumeTotalWriteTime / VolumeWriteOps
```

단, CloudWatch 통계 집계 방식에 따라 기간과 통계값을 올바르게 맞춰야 한다.

### 4-5) BurstBalance

gp2, st1, sc1에서 중요하다.

해석은 다음과 같다.

- 100%에 가까움: 크레딧 충분
- 점점 감소: 버스트 성능 사용 중
- 0% 근접: 곧 기준 성능으로 제한
- 0% 유지: 이미 성능 저하 발생 가능성 높음

해결 방법은 다음과 같다.

- gp2 → gp3 전환
- gp2 볼륨 크기 증가
- st1/sc1이면 워크로드가 적절한지 재검토
- HDD에 랜덤 I/O가 많다면 gp3/io2로 변경

### 4-6) VolumeThroughputPercentage

볼륨의 Throughput 한계 대비 사용률이다.

해석은 다음과 같다.

- 80% 이상 지속: 주의
- 100% 근접: Throughput 병목 가능성
- Queue Length 동반 상승: 처리량 부족 가능성 높음

조치 방법은 다음과 같다.

- gp3 Throughput 증가
- io2 검토
- 인스턴스 EBS 대역폭 확인
- 대용량 작업 시간 분산
- 압축/배치 방식 변경

### 4-7) VolumeIOPSPercentage

볼륨의 IOPS 한계 대비 사용률이다.

해석은 다음과 같다.

- 80% 이상 지속: 주의
- 100% 근접: IOPS 병목 가능성
- Queue Length 상승: I/O 요청이 밀리고 있음

조치 방법은 다음과 같다.

- gp3 IOPS 증가
- io2 사용
- DB 인덱스/쿼리 튜닝
- 캐시 도입
- 랜덤 I/O 감소

### 4-8) VolumeStalledIOCheck

I/O가 멈춘 상태를 확인하는 지표다.

AWS FIS로 I/O pause 실험을 하면 이 지표가 나타날 수 있다.

이 값이 1이면 스토리지 I/O가 정상 처리되지 않는 상태일 수 있으므로 즉시 확인해야 한다.

### 4-9) VolumeStatusCheckFailed

EBS 상태 확인 실패 여부를 나타낸다.

이 값이 발생하면 단순 성능 저하가 아니라 볼륨 상태 이상 가능성까지 고려해야 한다.

---

## 5. EBS Volume Status Check와 Event 확인

### 5-1) Volume Status Check

EBS는 볼륨 상태 확인을 제공한다.

주요 상태는 다음과 같다.

- ok
- warning
- impaired
- insufficient-data

### 5-2) ok

상태 검사를 통과한 상태다.

하지만 ok라고 해서 애플리케이션 성능 문제가 없다는 뜻은 아니다. 볼륨 자체 상태가 정상이라는 의미에 가깝다.

### 5-3) warning

성능이 예상보다 낮을 수 있다는 의미다.

특히 gp3, io1, io2 볼륨의 I/O 성능 상태 확인과 관련된다.

스냅샷에서 복원한 Provisioned IOPS SSD 볼륨은 초기화 중 성능이 낮아 warning이 발생할 수 있다. 이 경우에는 볼륨 초기화 상태를 함께 봐야 한다.

### 5-4) impaired

볼륨 상태가 손상되었거나 I/O가 비활성화되었을 수 있다.

이 경우 확인할 이벤트는 다음과 같다.

- Awaiting Action: Enable IO
- IO Enabled
- IO Auto-Enabled
- Degraded
- Severely Degraded
- Stalled

### 5-5) describe-volume-status

CLI로 상태 확인

```bash
aws ec2 describe-volume-status \
  --volume-ids vol-xxxxxxxxxxxxxxxxx
```

손상된 볼륨만 조회

```bash
aws ec2 describe-volume-status \
  --filters Name=volume-status.status,Values=impaired
```

### 5-6) impaired 상태 대응

기본 대응 흐름은 다음과 같다.

1. 애플리케이션 중지
2. 볼륨 상태 이벤트 확인
3. 필요 시 I/O 활성화
4. 파일시스템 일관성 검사
5. 스냅샷 복구 여부 판단
6. AWS Support 문의 필요 여부 판단

Linux 예시

```bash
aws ec2 enable-volume-io \
  --volume-id vol-xxxxxxxxxxxxxxxxx
```

파일시스템 검사는 마운트 해제 상태에서 수행해야 한다.

EXT 계열

```bash
sudo fsck -f /dev/nvme1n1
```

XFS

```bash
sudo xfs_repair /dev/nvme1n1
```

---

## 6. Linux에서 실시간으로 스토리지 병목 확인하기

### 6-1) lsblk

디스크 구조 확인

```bash
lsblk
```

자세히 확인

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,SERIAL
```

확인할 것

- 어떤 디바이스가 루트 볼륨인지
- 어떤 디바이스가 데이터 볼륨인지
- 마운트 위치가 어디인지
- Nitro 환경에서 NVMe 이름이 어떻게 잡혔는지
- SERIAL에 EBS Volume ID가 보이는지

### 6-2) df -hT

파일시스템 사용량 확인

```bash
df -hT
```

확인할 것

- 디스크가 꽉 찼는지
- 파일시스템 타입이 XFS인지 EXT4인지
- 마운트가 정상인지

디스크가 100%에 가까우면 성능이 급격히 나빠질 수 있다.

### 6-3) iostat -xz 1

가장 중요한 실시간 분석 명령어다.

```bash
iostat -xz 1
```

주요 항목

- r/s
- w/s
- rkB/s
- wkB/s
- await
- r_await
- w_await
- aqu-sz
- %util

### 6-4) r/s, w/s

초당 읽기/쓰기 요청 수다.

IOPS와 직접 연결된다.

### 6-5) rkB/s, wkB/s

초당 읽기/쓰기 데이터 양이다.

Throughput과 연결된다.

### 6-6) await

I/O 요청이 완료되기까지 걸린 평균 시간이다.

해석 예시

- 1~5ms: 일반적으로 양호
- 10ms 이상 지속: 주의
- 20~50ms 이상 지속: 병목 가능성
- 수백 ms: 심각한 스토리지 지연 가능성

단, 절대 기준은 워크로드와 볼륨 타입에 따라 다르다.

### 6-7) r_await, w_await

읽기와 쓰기 지연을 분리해서 볼 수 있다.

예를 들어

- r_await만 높음: 읽기 병목
- w_await만 높음: 쓰기 병목
- 둘 다 높음: 전체 스토리지 병목 또는 인스턴스 병목

### 6-8) aqu-sz

I/O 큐 길이다.

CloudWatch의 VolumeQueueLength와 함께 해석한다.

### 6-9) %util

디스크가 I/O 처리로 바쁜 시간 비율이다.

단일 디바이스 기준으로 100%에 가까우면 포화로 해석할 수 있다.

하지만 클라우드/NVMe 환경에서는 %util만으로 병목을 단정하면 안 된다. await, queue, IOPS, throughput을 같이 봐야 한다.

### 6-10) iotop

어떤 프로세스가 I/O를 발생시키는지 확인한다.

```bash
sudo iotop -o
```

자주 발견되는 원인

- mysqld
- postgres
- java
- elasticsearch
- fluent-bit
- logrotate
- backup agent
- rsync
- tar
- gzip

### 6-11) pidstat

프로세스 단위 디스크 I/O 확인

```bash
pidstat -d 1
```

### 6-12) dmesg / journalctl

커널 로그 확인

```bash
dmesg -T | grep -i -E "nvme|blk|timeout|i/o error|ext4|xfs"
```

```bash
journalctl -k
```

확인할 메시지

- nvme timeout
- I/O error
- blk_update_request
- EXT4-fs error
- XFS error
- read-only filesystem
- reset controller

이런 로그가 있으면 단순 성능 문제가 아니라 I/O 오류 또는 파일시스템 문제일 수 있다.

---

## 7. EC2 인스턴스 병목 진단

### 7-1) EBS 볼륨 성능과 EC2 인스턴스 성능은 별개다

예를 들어 EBS 볼륨이 16,000 IOPS를 제공하더라도 EC2 인스턴스가 8,000 IOPS까지만 지원하면 실제 애플리케이션은 8,000 IOPS 이상을 사용할 수 없다.

즉 실제 성능은 다음 중 작은 값에 의해 제한된다.

```text
실제 성능 = min(EBS 볼륨 성능, EC2 인스턴스 EBS 성능)
```

### 7-2) EBS Optimized 인스턴스

EBS 최적화 인스턴스는 EC2와 EBS 사이에 전용 대역폭을 제공한다.

EBS 최적화를 지원하지 않는 인스턴스에서는 일반 네트워크 트래픽과 EBS 트래픽이 경합할 수 있다.

확인할 것

- 현재 인스턴스가 EBS Optimized인지
- 인스턴스 타입별 EBS 최대 대역폭
- 인스턴스 타입별 EBS 최대 IOPS
- 네트워크 트래픽과 EBS 트래픽이 경합하는지

### 7-3) EBSIOBalance%

인스턴스의 EBS I/O 크레딧 잔량이다.

낮아지면 인스턴스 측에서 EBS I/O가 제한될 수 있다.

### 7-4) EBSByteBalance%

인스턴스의 EBS 바이트 처리량 크레딧 잔량이다.

낮아지면 인스턴스의 EBS Throughput 한계를 의심해야 한다.

### 7-5) 인스턴스 병목 판단 패턴

다음 상황이면 인스턴스 병목 가능성이 있다.

- EBS 볼륨의 IOPS/Throughput 사용률은 낮음
- 그런데 OS에서는 await가 높음
- Queue도 증가
- EBSIOBalance% 또는 EBSByteBalance%가 낮음
- 인스턴스 타입의 EBS 한계가 낮음

조치 방법

- 더 큰 인스턴스로 변경
- EBS 대역폭이 높은 인스턴스로 변경
- EBS 전용 대역폭이 충분한 타입 선택
- 네트워크와 EBS 트래픽을 함께 고려

---

## 8. 스냅샷 복원 볼륨과 초기화 이슈

### 8-1) 스냅샷에서 생성한 볼륨은 초기 성능이 낮을 수 있다

스냅샷에서 EBS 볼륨을 생성하면 데이터 블록은 필요할 때 S3에서 EBS로 로딩된다.

이 과정에서 처음 접근하는 블록은 지연 시간이 증가할 수 있다.

증상

- 새로 복원한 볼륨이 처음에는 느림
- 특정 파일이나 영역 접근 시 지연
- I/O 성능 상태가 warning으로 보일 수 있음

### 8-2) 해결 방법

해결 방법은 다음과 같다.

- Fast Snapshot Restore 사용
- 볼륨 초기화 수행
- 운영 투입 전 워밍업
- fio/dd 등을 사용한 사전 읽기
- 복원 직후 성능 저하를 정상 동작으로 고려

### 8-3) 진단 포인트

다음 질문을 해야 한다.

- 최근 스냅샷에서 복원한 볼륨인가?
- 새로 생성한 볼륨인가?
- Fast Snapshot Restore가 활성화되었는가?
- 특정 블록에 처음 접근하는 상황인가?
- 모든 구간이 느린가, 처음 접근만 느린가?

---

## 9. NVMe와 Nitro 환경에서 확인할 것

### 9-1) Nitro 인스턴스에서는 EBS가 NVMe 디바이스로 보인다

Nitro 기반 인스턴스에서는 EBS가 다음과 같이 보인다.

```bash
/dev/nvme0n1
/dev/nvme1n1
```

AWS 콘솔에서 `/dev/sdf`로 붙였어도 OS에서는 `/dev/nvme1n1`처럼 보일 수 있다.

### 9-2) 볼륨 ID 매핑

확인 명령어

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,SERIAL
```

또는

```bash
sudo nvme id-ctrl -V /dev/nvme1n1
```

Amazon Linux 계열

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

### 9-3) NVMe I/O Timeout

Linux에서는 NVMe I/O timeout 설정이 있다.

확인

```bash
cat /sys/module/nvme_core/parameters/io_timeout
```

I/O가 timeout 값을 초과하면 OS는 I/O 오류를 반환할 수 있고, 심한 경우 파일시스템이 read-only로 전환될 수 있다.

### 9-4) timeout 관련 로그

확인

```bash
dmesg -T | grep -i nvme
```

의심 로그

- I/O timeout
- abort command
- reset controller

이 경우 단순 성능 저하가 아니라 I/O path 문제를 의심해야 한다.

---

## 10. 파일시스템과 마운트 설정 확인

### 10-1) 파일시스템 타입 확인

```bash
df -hT
lsblk -f
```

확인할 것

- XFS인지 EXT4인지
- 마운트 위치가 맞는지
- 파일시스템이 read-only로 바뀌었는지
- 디스크 사용률이 100%에 가까운지

### 10-2) fstab 확인

```bash
cat /etc/fstab
```

주의할 것

- `/dev/nvme1n1` 같은 디바이스 이름 직접 사용
- UUID 미사용
- nofail 미설정
- 잘못된 마운트 옵션
- 존재하지 않는 볼륨을 부팅 시 마운트하려는 설정

권장

```text
UUID=<UUID> /data xfs defaults,nofail 0 2
```

### 10-3) 디스크 사용량 확인

```bash
df -h
du -sh /data/*
```

디스크가 꽉 차면 I/O가 느려지고 애플리케이션 오류가 발생할 수 있다.

### 10-4) inode 확인

```bash
df -i
```

작은 파일이 너무 많으면 용량이 남아 있어도 inode 부족으로 쓰기 실패가 발생할 수 있다.

---

## 11. 애플리케이션 레벨 원인 분석

### 11-1) 데이터베이스

DB가 EBS 성능 문제의 가장 흔한 원인이다.

확인할 것

- Slow Query
- Full Scan 증가
- Index 누락
- Checkpoint
- Vacuum
- Autovacuum
- Binary Log 증가
- WAL 증가
- fsync 지연
- Lock 대기

MySQL 예시

```sql
SHOW PROCESSLIST;
SHOW ENGINE INNODB STATUS;
```

PostgreSQL 예시

```sql
SELECT * FROM pg_stat_activity;
SELECT * FROM pg_stat_bgwriter;
```

### 11-2) 로그 폭증

로그가 갑자기 증가하면 쓰기 I/O가 폭증한다.

확인

```bash
du -sh /var/log/*
iotop -o
```

원인 예시

- 애플리케이션 오류 로그 폭증
- debug 로그 활성화
- 컨테이너 로그 무제한 증가
- logrotate 실패

### 11-3) 백업 작업

백업은 Throughput을 많이 사용한다.

원인 예시

- rsync
- tar
- gzip
- mysqldump
- pg_dump
- snapshot 전후 작업
- S3 업로드

### 11-4) Kubernetes 환경

Kubernetes에서 EBS를 PVC로 사용할 경우 추가 확인이 필요하다.

확인할 것

- PVC가 어떤 StorageClass를 쓰는지
- gp2인지 gp3인지
- PVC 용량과 실제 요청량
- Pod가 어느 노드에 붙었는지
- EBS는 단일 AZ 리소스이므로 Pod 스케줄링이 AZ와 맞는지
- ReadWriteOnce 볼륨을 여러 Pod가 쓰려는지

---

## 12. 대표적인 원인별 진단 패턴

### 12-1) gp2 BurstBalance 고갈

증상

- 갑자기 느려짐
- 이전에는 괜찮았음
- CloudWatch BurstBalance 하락
- QueueLength 증가
- await 증가

원인

- gp2 크레딧 고갈

조치

- gp3 전환
- 볼륨 크기 증가
- I/O 패턴 최적화
- 캐시 도입

### 12-2) gp3 IOPS 부족

증상

- VolumeIOPSPercentage 100% 근접
- QueueLength 증가
- DB 랜덤 I/O 많음

조치

- gp3 IOPS 증가
- io2 검토
- DB 쿼리 튜닝
- 인덱스 추가
- 캐시 사용

### 12-3) gp3 Throughput 부족

증상

- VolumeThroughputPercentage 100% 근접
- 대용량 처리 느림
- 백업/배치 시간대에 발생

조치

- gp3 Throughput 증가
- 작업 시간 분산
- 압축/전송 방식 개선
- 상위 인스턴스 검토

### 12-4) EC2 EBS 대역폭 부족

증상

- 볼륨 성능은 충분함
- 인스턴스 EBS 관련 지표 부족
- await 증가
- 여러 볼륨이 한 인스턴스에 몰림

조치

- 인스턴스 타입 변경
- EBS 대역폭 높은 타입 선택
- 여러 인스턴스로 워크로드 분산
- EBS 카드/대역폭 구조 고려

### 12-5) 스냅샷 복원 초기화 지연

증상

- 새로 만든 볼륨만 느림
- 처음 접근하는 데이터가 느림
- 시간이 지나면 개선

조치

- Fast Snapshot Restore
- 사전 초기화
- 운영 투입 전 워밍업

### 12-6) HDD 볼륨에 랜덤 I/O 발생

증상

- st1/sc1 사용
- 작은 랜덤 I/O 많음
- 응답 지연 심함

조치

- gp3/io2로 변경
- 워크로드 재배치
- 순차 I/O 중심으로 변경

### 12-7) 파일시스템 오류

증상

- dmesg에 EXT4/XFS 오류
- read-only filesystem
- I/O error

조치

- 서비스 중지
- 볼륨 detach
- 복구용 인스턴스 attach
- fsck/xfs_repair
- 스냅샷 복구 검토

---

## 13. fio를 이용한 벤치마킹

### 13-1) fio를 쓰는 이유

fio는 디스크 I/O 부하를 인위적으로 발생시켜 성능을 측정하는 도구다.

운영 볼륨에서 직접 수행하면 위험하다. 반드시 테스트 볼륨에서 수행해야 한다.

### 13-2) 설치

Amazon Linux / RHEL 계열

```bash
sudo yum install -y fio
```

Ubuntu 계열

```bash
sudo apt update
sudo apt install -y fio
```

### 13-3) 랜덤 읽기 테스트

```bash
sudo fio \
  --name=randread \
  --directory=/data \
  --rw=randread \
  --bs=16k \
  --size=1G \
  --numjobs=16 \
  --time_based \
  --runtime=180 \
  --direct=1 \
  --group_reporting
```

### 13-4) 랜덤 쓰기 테스트

```bash
sudo fio \
  --name=randwrite \
  --directory=/data \
  --rw=randwrite \
  --bs=16k \
  --size=1G \
  --numjobs=16 \
  --time_based \
  --runtime=180 \
  --direct=1 \
  --group_reporting
```

### 13-5) 순차 읽기 테스트

```bash
sudo fio \
  --name=seqread \
  --directory=/data \
  --rw=read \
  --bs=1M \
  --size=5G \
  --numjobs=4 \
  --time_based \
  --runtime=180 \
  --direct=1 \
  --group_reporting
```

### 13-6) fio 결과 해석

주요 항목

- IOPS
- BW
- lat
- clat
- percentiles
- jobs
- runtime

확인 포인트

- 기대 IOPS가 나오는가?
- 기대 Throughput이 나오는가?
- 지연 시간 분포가 안정적인가?
- p99 latency가 튀는가?
- EC2 한계에 먼저 걸리는가?
- EBS 한계에 먼저 걸리는가?

---

## 14. AWS FIS로 스토리지 장애 테스트

### 14-1) AWS FIS란

AWS Fault Injection Service는 장애 주입 테스트를 수행하는 서비스다.

EBS에서는 다음 테스트가 가능하다.

- I/O Pause
- Latency Injection

### 14-2) I/O Pause

EBS와 EC2 사이의 I/O를 일정 시간 중단한다.

확인할 수 있는 것

- 애플리케이션 timeout
- DB 장애 반응
- CloudWatch Alarm
- VolumeStalledIOCheck
- 운영 대응 절차

### 14-3) Latency Injection

일부 읽기/쓰기 I/O에 인위적으로 지연 시간을 주입한다.

확인할 수 있는 것

- 지연 증가 시 애플리케이션이 버티는지
- p95/p99 latency가 어떻게 변하는지
- 장애 조치가 필요한 임계값은 어디인지
- 알람이 제대로 울리는지

### 14-4) 운영 환경 주의사항

운영에서 FIS를 사용할 경우 반드시 준비해야 한다.

- 스냅샷
- 승인 절차
- 점검 시간
- Stop Condition
- CloudWatch Alarm
- Rollback 계획
- 담당자 대기

---

## 15. 문제 발생 시 조치 방법

### 15-1) gp2에서 gp3로 변경

가장 일반적인 개선 방법이다.

```bash
aws ec2 modify-volume \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --volume-type gp3
```

필요하면 IOPS와 Throughput도 지정한다.

```bash
aws ec2 modify-volume \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --volume-type gp3 \
  --iops 6000 \
  --throughput 250
```

### 15-2) gp3 성능 상향

```bash
aws ec2 modify-volume \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --iops 12000 \
  --throughput 500
```

### 15-3) io2로 변경

고성능 DB라면 io2를 검토한다.

```bash
aws ec2 modify-volume \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --volume-type io2 \
  --iops 20000
```

### 15-4) 볼륨 크기 확장

```bash
aws ec2 modify-volume \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --size 500
```

Linux에서는 이후 파티션과 파일시스템 확장이 필요하다.

파티션 확장

```bash
sudo growpart /dev/nvme0n1 1
```

XFS

```bash
sudo xfs_growfs -d /
```

EXT4

```bash
sudo resize2fs /dev/nvme0n1p1
```

### 15-5) 인스턴스 타입 변경

볼륨은 충분한데 인스턴스가 병목이면 인스턴스 타입을 변경해야 한다.

확인할 것

- vCPU
- Memory
- Network bandwidth
- EBS bandwidth
- EBS IOPS
- Nitro 여부

### 15-6) 애플리케이션 최적화

스토리지 스펙을 올리는 것만이 답은 아니다.

DB라면 다음이 더 효과적일 수 있다.

- 인덱스 추가
- 쿼리 튜닝
- 캐시 사용
- 배치 시간 조정
- 로그 레벨 조정
- 데이터 파티셔닝
- 읽기 복제본 사용

---

## 16. 실무용 진단 체크리스트

### 16-1) 기본 정보 확인

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,SERIAL
df -hT
df -i
```

확인 항목

- 볼륨 크기
- 파일시스템
- 마운트 위치
- 디스크 사용량
- inode 사용량
- EBS Volume ID

### 16-2) CloudWatch 확인

확인 지표

- VolumeReadOps
- VolumeWriteOps
- VolumeReadBytes
- VolumeWriteBytes
- VolumeQueueLength
- VolumeTotalReadTime
- VolumeTotalWriteTime
- BurstBalance
- VolumeIOPSPercentage
- VolumeThroughputPercentage
- VolumeStatusCheckFailed
- VolumeStalledIOCheck

### 16-3) 인스턴스 확인

확인 항목

- 인스턴스 타입
- EBS Optimized
- EBSIOBalance%
- EBSByteBalance%
- 최대 EBS bandwidth
- 최대 EBS IOPS

### 16-4) OS 확인

```bash
iostat -xz 1
iotop -o
pidstat -d 1
vmstat 1
dmesg -T
journalctl -k
```

### 16-5) 애플리케이션 확인

확인 항목

- DB slow query
- 로그 폭증
- 백업 실행 여부
- 배치 실행 여부
- 컨테이너 로그 증가
- 애플리케이션 timeout

---

## 17. 면접 답변용 정리

클라우드 환경에서 EBS 같은 네트워크 스토리지 성능 이슈가 발생하면 먼저 문제가 스토리지 자체인지, EC2 인스턴스 한계인지, OS 또는 애플리케이션 문제인지 분리해야 한다.

EBS 레벨에서는 CloudWatch의 VolumeReadOps, VolumeWriteOps, VolumeReadBytes, VolumeWriteBytes, VolumeQueueLength, BurstBalance, VolumeIOPSPercentage, VolumeThroughputPercentage 등을 확인한다. QueueLength가 지속적으로 높고 IOPS 또는 Throughput 사용률이 한계에 가까우면 볼륨 성능 부족을 의심할 수 있다. gp2, st1, sc1에서는 BurstBalance 고갈 여부도 반드시 확인해야 한다.

인스턴스 레벨에서는 EC2 인스턴스의 EBS 최대 대역폭과 IOPS 한계를 확인한다. 볼륨 성능은 충분하지만 await가 높거나 큐가 증가한다면 인스턴스의 EBS 대역폭 병목일 수 있다.

OS 레벨에서는 iostat -xz 1로 await, r_await, w_await, aqu-sz, %util을 확인하고, iotop이나 pidstat로 어떤 프로세스가 I/O를 발생시키는지 확인한다. dmesg와 journalctl에서는 NVMe timeout, I/O error, 파일시스템 오류 여부를 확인한다.

최종적으로 원인이 볼륨 성능 부족이면 gp3 IOPS/Throughput 증설, gp2에서 gp3 전환, io2 사용 등을 검토한다. 인스턴스 병목이면 더 높은 EBS 대역폭을 제공하는 인스턴스로 변경한다. 애플리케이션 문제라면 쿼리 튜닝, 캐시 도입, 백업 시간 조정, 로그 정책 변경 등을 수행한다.

---

## 18. 결론

EBS 성능 이슈 진단의 핵심은 다음 네 가지다.

- IOPS
- Throughput
- Latency
- Queue Length

하지만 이 네 가지 지표만 보는 것으로는 부족하다.

실무에서는 다음 순서로 봐야 한다.

1. 사용자가 느리다고 말하는 증상 구체화
2. CPU, Memory, Network 등 기본 리소스 확인
3. CloudWatch EBS 지표 확인
4. EBS 볼륨 타입과 프로비저닝 성능 확인
5. BurstBalance 확인
6. EC2 인스턴스의 EBS 대역폭 확인
7. Linux iostat으로 실시간 I/O 확인
8. iotop/pidstat로 원인 프로세스 확인
9. dmesg/journalctl로 I/O 오류 확인
10. 애플리케이션 로그와 DB 지표 확인
11. 볼륨 또는 인스턴스 스펙 조정
12. 필요 시 fio로 벤치마킹
13. FIS로 장애 대응 훈련

즉, EBS 성능 문제는 “스토리지 하나만 보는 문제”가 아니라 클라우드 인프라 전체를 계층별로 분리해서 보는 문제다.