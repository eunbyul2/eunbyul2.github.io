---
layout: post
title: "S3 인증, 권한, 암호화"
date: 2026-07-01 20:550:00 +0900
categories: [Storage, S3, ObjectStorage]
tags: [S3Security, IAM, SigV4, SSE, KMS, OIDC, STS, PresignedURL]
published: true
---

## 1. S3 보안을 이해하기 전에 먼저 나눠야 하는 것

S3 보안을 공부할 때 가장 먼저 구분해야 하는 것은 **인증(Authentication)**, **인가/권한(Authorization)**, **암호화(Encryption)**, **보존/불변성(Immutability)**입니다. 이 네 가지를 섞어서 이해하면 Access Key, IAM Policy, Bucket Policy, SSE, Object Lock 같은 개념이 전부 비슷하게 느껴집니다. 하지만 실제 역할은 다릅니다.

```text
Authentication  = 너는 누구인가?
Authorization   = 네가 무엇을 할 수 있는가?
Encryption      = 데이터를 읽을 수 없게 보호하는가?
Immutability    = 데이터를 삭제/수정하지 못하게 막는가?
```

S3 Console을 만든다면 이 구분은 UI 설계에도 직접 영향을 줍니다. 예를 들어 로그인 화면은 인증 영역이고, 사용자/그룹/정책 메뉴는 권한 영역이고, SSE 설정은 암호화 영역이며, Object Lock/WORM은 보존 정책 영역입니다.

```text
S3 Console
├── Login / OIDC / Session                → Authentication
├── User / Group / Role / Policy / Quota  → Authorization
├── SSE / KMS                             → Encryption
└── Object Lock / Retention / Legal Hold  → Immutability
```

S3 API 호출은 일반 웹 로그인과도 다릅니다. 브라우저에서 ID/PW를 입력하고 세션 쿠키를 받는 방식만 있는 것이 아니라, CLI와 SDK에서는 Access Key와 Secret Key로 요청에 서명합니다. 이 서명 방식이 Signature Version 4, 즉 SigV4입니다.

---

## 2. HTTP

### 2-1) HTTP란?

HTTP는 HyperText Transfer Protocol입니다. 웹 브라우저와 서버가 데이터를 주고받기 위한 애플리케이션 계층 프로토콜입니다. S3 API도 HTTP를 사용합니다. 즉 사용자가 콘솔에서 “업로드” 버튼을 누르든, AWS CLI에서 `aws s3 cp`를 실행하든, 내부적으로는 HTTP 요청이 발생합니다.

```text
사용자 / CLI / SDK
  ↓ HTTP Request
S3 Endpoint
  ↓ HTTP Response
사용자 / CLI / SDK
```

S3는 파일시스템 API가 아니라 HTTP API를 제공합니다. 그래서 S3에서는 `open()`, `write()`, `close()` 같은 POSIX 파일시스템 호출이 아니라 `PUT Object`, `GET Object`, `DELETE Object`, `ListObjectsV2` 같은 API가 중심입니다.

### 2-2) HTTP 요청 구조

HTTP 요청은 크게 Method, Path, Header, Body로 구성됩니다.

```http
PUT /my-bucket/logs/app.log HTTP/1.1
Host: s3.example.com
Content-Type: text/plain
Content-Length: 1204
Authorization: AWS4-HMAC-SHA256 ...

<파일 데이터>
```

각 요소의 의미는 다음과 같습니다.

| 요소 | 의미 | S3 예시 |
|---|---|---|
| Method | 어떤 동작을 할 것인가 | GET, PUT, DELETE, HEAD |
| Path | 어떤 리소스를 대상으로 할 것인가 | `/bucket/key` |
| Header | 요청 부가 정보 | Authorization, Content-Type |
| Body | 실제 데이터 | 업로드할 파일 내용 |

S3 Console 개발에서는 이 구조를 알아야 합니다. 예를 들어 다운로드는 `GET Object`, 업로드는 `PUT Object`, 삭제는 `DELETE Object`, 메타데이터 조회는 `HEAD Object`로 대응됩니다.

### 2-3) S3에서 자주 쓰는 HTTP Method

| HTTP Method | S3에서의 대표 의미 |
|---|---|
| GET | 오브젝트 다운로드, 버킷/오브젝트 목록 조회 |
| PUT | 버킷 생성, 오브젝트 업로드, 정책 설정 |
| DELETE | 버킷 삭제, 오브젝트 삭제 |
| HEAD | 오브젝트 본문 없이 메타데이터만 조회 |
| POST | 일부 업로드/관리성 요청 |
| OPTIONS | CORS 사전 요청 |

S3는 REST 스타일 API를 사용하지만, 모든 기능이 단순히 CRUD로만 대응되는 것은 아닙니다. 예를 들어 Multipart Upload는 `CreateMultipartUpload`, `UploadPart`, `CompleteMultipartUpload`, `AbortMultipartUpload`처럼 여러 단계 API로 구성됩니다.

### 2-4) HTTP는 왜 보안상 위험한가?

HTTP는 평문 통신입니다. 네트워크 중간에서 패킷을 볼 수 있으면 요청 URL, Header, Body가 노출될 수 있습니다.

```text
Client ── HTTP 평문 ── 중간 네트워크 ── S3 Server
```

만약 HTTP로 S3 업로드를 수행하면 다음 정보가 노출될 수 있습니다.

```text
- 업로드/다운로드 대상 버킷명
- 오브젝트 Key
- 요청 Header
- 파일 내용
- Presigned URL
- SSE-C 사용 시 고객 제공 암호화 키 Header
```

특히 SSE-C는 고객이 직접 암호화 키를 요청 Header로 제공하기 때문에 HTTPS가 필수입니다. AWS S3 공식 문서에서도 SSE-C 요청은 HTTPS를 사용해야 하며, HTTP로 전송한 키는 손상된 것으로 간주하고 폐기해야 한다고 설명합니다.

### 2-5) 콘솔 개발 관점

S3 Console에서는 HTTP로 접속한 경우 경고하거나, 운영 배포에서는 HTTP를 HTTPS로 리다이렉트해야 합니다.

권장 정책은 다음과 같습니다.

```text
- 운영 콘솔은 HTTPS만 허용
- S3 API Endpoint도 HTTPS 사용
- Presigned URL도 HTTPS로 발급
- 브라우저 Mixed Content 발생 여부 확인
- Ingress TLS 인증서 만료 모니터링
```

---

## 3. HTTPS

### 3-1) HTTPS란?

HTTPS는 HTTP over TLS입니다. HTTP 메시지를 TLS로 암호화하여 전송합니다.

```text
HTTP  = 평문 HTTP
HTTPS = HTTP + TLS
```

사용자는 브라우저 주소창에서 `https://`를 보고 단순히 “보안 접속”이라고 이해하지만, 내부적으로는 서버 인증서 검증, 암호화 알고리즘 협상, 세션키 생성, 암호화 통신이 순서대로 진행됩니다.

### 3-2) HTTPS가 필요한 이유

S3는 파일 데이터 자체를 다룹니다. 단순 HTML 페이지보다 보호해야 할 데이터가 훨씬 민감할 수 있습니다.

```text
- 개인정보 파일
- 백업 데이터
- 로그 데이터
- AI 학습 데이터셋
- 고객사 업로드 파일
- Access Key 기반 API 요청
```

HTTPS를 사용하면 다음을 보호할 수 있습니다.

| 보호 항목 | 설명 |
|---|---|
| 기밀성 | 중간에서 내용을 볼 수 없음 |
| 무결성 | 중간에서 데이터를 바꾸기 어려움 |
| 서버 인증 | 접속한 서버가 진짜 서버인지 확인 |

### 3-3) HTTPS와 S3 Endpoint

S3 Endpoint는 다음처럼 표현됩니다.

```text
AWS S3:      https://s3.ap-northeast-2.amazonaws.com
Ceph RGW:    https://s3.quantumcns.ai
RustFS:      https://rustfs.example.com 또는 http://rustfs-test:9000
```

개발 테스트 환경에서는 HTTP를 사용할 수 있지만, 운영 환경에서는 반드시 HTTPS가 맞습니다. 특히 회사 콘솔이 고객사 환경에 설치된다면 인증서 발급, 갱신, Ingress TLS 설정까지 자동화 대상에 포함해야 합니다.

### 3-4) Ceph/RustFS 환경에서 자주 발생하는 HTTPS 문제

| 증상 | 가능 원인 |
|---|---|
| 브라우저 인증서 경고 | 자체 서명 인증서, 만료 인증서, CN/SAN 불일치 |
| API 호출 CORS 실패 | HTTPS/HTTP 혼용, Origin 불일치 |
| Presigned URL 접속 실패 | 서명 시 Host와 실제 접속 Host 불일치 |
| Virtual Hosted Style 실패 | 와일드카드 인증서 누락 |
| 업로드 중 Mixed Content 경고 | 콘솔은 HTTPS인데 S3 Endpoint는 HTTP |

---

## 4. TLS

### 4-1) TLS란?

TLS는 Transport Layer Security입니다. SSL의 후속 표준이며, 현대 HTTPS는 TLS를 사용합니다. 실무에서는 아직 “SSL 인증서”라는 표현을 많이 쓰지만, 정확히는 TLS 인증서 또는 X.509 인증서라고 보는 것이 맞습니다.

```text
과거 표현: SSL
현재 표준: TLS
일상 표현: SSL 인증서
정확한 표현: TLS 인증서 / X.509 인증서
```

### 4-2) TLS가 해결하는 문제

TLS는 네트워크 중간 공격을 방어하기 위해 사용됩니다.

대표 위협은 다음과 같습니다.

```text
- 도청: 중간에서 데이터를 읽음
- 변조: 중간에서 요청/응답을 바꿈
- 위장: 공격자가 진짜 서버인 척함
```

TLS는 이를 위해 암호화, 무결성 검증, 인증서 기반 서버 인증을 제공합니다.

### 4-3) 대칭키와 비대칭키

TLS를 이해하려면 대칭키와 비대칭키를 구분해야 합니다.

| 구분 | 설명 | 장점 | 단점 |
|---|---|---|---|
| 대칭키 | 같은 키로 암호화/복호화 | 빠름 | 키 전달이 어려움 |
| 비대칭키 | 공개키/개인키 쌍 사용 | 키 교환에 유리 | 느림 |

TLS는 둘 중 하나만 쓰는 것이 아닙니다. 처음에는 비대칭키 기반 방식으로 안전하게 세션키를 만들고, 실제 데이터 전송은 빠른 대칭키로 처리합니다.

```text
TLS Handshake
  ↓ 비대칭키/키 교환으로 세션키 협상
Application Data
  ↓ 대칭키로 빠르게 암호화 통신
```

### 4-4) 인증서와 CA

인증서는 서버의 공개키와 도메인 정보를 담고 있습니다. CA는 Certificate Authority, 즉 인증기관입니다.

브라우저는 서버 인증서를 보고 다음을 확인합니다.

```text
- 인증서가 신뢰할 수 있는 CA에서 발급되었는가?
- 인증서의 도메인이 접속한 도메인과 일치하는가?
- 인증서가 만료되지 않았는가?
- 인증서가 폐기되지 않았는가?
```

S3 콘솔과 S3 Endpoint가 서로 다른 도메인을 사용할 경우 각각 인증서 설정을 확인해야 합니다.

```text
console.example.com
s3.example.com
*.s3.example.com
```

Virtual Hosted Style을 사용하면 `bucket.s3.example.com` 형태의 주소가 필요하므로 `*.s3.example.com` 와일드카드 인증서가 필요할 수 있습니다.

---

## 5. TLS Handshake

### 5-1) Handshake란?

Handshake는 클라이언트와 서버가 본격적으로 데이터를 주고받기 전에 연결 조건을 협상하는 과정입니다. TLS Handshake에서는 TLS 버전, 암호화 알고리즘, 인증서, 세션키가 결정됩니다.

```text
Client
  ↓ ClientHello
Server
  ↓ ServerHello + Certificate
Client
  ↓ 인증서 검증 + 키 교환
Server
  ↓ Finished
암호화 통신 시작
```

### 5-2) TLS Handshake 흐름

단순화하면 다음과 같습니다.

```text
1. ClientHello
   - 지원 TLS 버전
   - 지원 Cipher Suite
   - Random 값
   - SNI

2. ServerHello
   - 선택한 TLS 버전
   - 선택한 Cipher Suite
   - Random 값

3. Certificate
   - 서버 인증서 전달

4. Certificate Verification
   - 클라이언트가 인증서 검증

5. Key Exchange
   - 세션키 생성을 위한 정보 교환

6. Finished
   - 이후부터 암호화된 Application Data 전송
```

### 5-3) SNI

SNI는 Server Name Indication입니다. 하나의 IP에서 여러 TLS 인증서를 사용할 때 클라이언트가 접속하려는 도메인을 TLS Handshake 초기에 알려주는 기능입니다.

Ingress 환경에서는 SNI가 중요합니다.

```text
클라이언트가 ceph.quantumcns.ai로 접속
  ↓
TLS ClientHello에 SNI=ceph.quantumcns.ai 포함
  ↓
Ingress Controller가 해당 인증서를 선택
```

SNI나 인증서 설정이 잘못되면 브라우저에서는 인증서 불일치 오류가 발생하고, CLI에서는 TLS 검증 실패가 발생할 수 있습니다.

### 5-4) S3 운영에서 TLS Handshake가 중요한 이유

S3 자체 인증은 SigV4가 담당하지만, 네트워크 구간 암호화는 TLS가 담당합니다. 둘은 역할이 다릅니다.

```text
TLS   = 통신 채널 보호
SigV4 = 요청자가 올바른 Secret Key를 가진 주체인지 검증
IAM   = 요청자가 해당 작업을 할 권한이 있는지 판단
```

따라서 TLS가 정상이어도 IAM 권한이 없으면 403이 나고, SigV4가 틀리면 서명 오류가 나며, 인증서가 틀리면 API 요청 자체가 실패할 수 있습니다.

---

## 6. S3 인증 구조 전체 흐름

S3 API 요청은 보통 다음 단계를 거칩니다.

```text
Client / SDK / CLI
  ↓
1. Endpoint 결정
  ↓
2. HTTP 요청 생성
  ↓
3. Access Key / Secret Key로 SigV4 서명 생성
  ↓
4. HTTPS로 요청 전송
  ↓
5. S3 서버가 서명 검증
  ↓
6. IAM/Policy 권한 평가
  ↓
7. 요청 실행 또는 거부
```

Ceph RGW나 RustFS도 S3 호환 API를 제공한다면 비슷한 흐름을 따릅니다. 다만 관리 API, 정책 모델, 지원 기능은 구현체마다 차이가 있습니다.

---

## 7. Access Key

### 7-1) Access Key란?

Access Key는 S3 API 호출 주체를 식별하는 값입니다. 일반 로그인 ID와 비슷해 보이지만, 사람이 직접 로그인하는 계정보다는 프로그램, SDK, CLI, 서비스 계정이 API 호출에 사용하는 식별자라고 이해하는 것이 좋습니다.

```text
Access Key = API 호출 주체 식별자
Secret Key = 서명 생성용 비밀값
```

Access Key는 공개되어도 Secret Key가 없으면 서명을 만들 수 없습니다. 하지만 Access Key도 계정 식별에 사용되므로 불필요하게 노출하지 않는 것이 좋습니다.

### 7-2) Access Key가 필요한 이유

S3는 대개 브라우저 사용자가 직접 로그인해서만 쓰는 서비스가 아닙니다. 애플리케이션 서버, 백업 스크립트, 데이터 파이프라인, AI 학습 잡, Kubernetes CronJob 등이 자동으로 S3 API를 호출합니다.

```text
Application
  ↓ Access Key / Secret Key
S3 API
```

ID/PW 기반 로그인만 제공하면 자동화가 어렵습니다. 그래서 API 자격증명으로 Access Key/Secret Key 방식을 사용합니다.

### 7-3) Access Key 관리 원칙

```text
- 사용자별/서비스별로 분리한다.
- 하나의 키를 여러 시스템에서 공유하지 않는다.
- 사용하지 않는 키는 비활성화하거나 삭제한다.
- 정기적으로 Rotation한다.
- 권한은 최소 권한으로 제한한다.
- Secret Key는 생성 직후 1회만 표시한다.
```

### 7-4) 콘솔 개발 관점

Access Key 화면에는 최소 다음 기능이 필요합니다.

```text
- Access Key 목록
- 생성
- 비활성화/활성화
- 삭제
- Secret Key 1회 표시
- 만료일 설정
- 마지막 사용 시간 표시
- 연결된 사용자/정책 표시
- Export 시 보안 경고
```

특히 Secret Key를 DB에 평문으로 저장하거나, 콘솔에서 반복 조회 가능하게 만들면 위험합니다. 일반적으로 Secret Key는 생성 직후 사용자에게 보여주고 이후에는 다시 볼 수 없게 하는 방식이 안전합니다.

---

## 8. Secret Key

### 8-1) Secret Key란?

Secret Key는 요청 서명을 만들기 위한 비밀값입니다. 비밀번호와 유사하지만, 직접 서버에 전송하는 값이 아니라 HMAC 서명 생성에 사용되는 비밀 재료입니다.

```text
Secret Key
  ↓ HMAC 계산
Signature
  ↓ Authorization Header에 포함
S3 서버가 검증
```

AWS 공식 문서도 SigV4에서는 Secret Access Key 자체로 요청을 서명하지 않고, SigV4 서명 프로세스를 통해 서명을 계산한다고 설명합니다.

### 8-2) Secret Key를 직접 보내지 않는 이유

만약 Secret Key를 요청에 그대로 포함하면, 중간자 공격이나 로그 유출 시 즉시 계정이 탈취됩니다. SigV4는 Secret Key를 직접 보내지 않고 요청 내용 기반의 서명만 전송합니다.

```text
잘못된 방식:
Client → Secret Key 직접 전송 → Server

S3 방식:
Client → Secret Key로 Signature 생성 → Signature만 전송 → Server
```

서버는 자신이 알고 있는 Secret Key로 같은 서명을 계산해 비교합니다. 두 서명이 같으면 요청자가 Secret Key를 알고 있다고 판단합니다.

### 8-3) Secret Key 유출 시 영향

Secret Key가 유출되면 공격자는 해당 키의 권한 범위 내에서 API를 호출할 수 있습니다.

가능한 피해는 다음과 같습니다.

```text
- 데이터 다운로드
- 데이터 삭제
- 악성 데이터 업로드
- Presigned URL 생성
- 버킷 정책 변경
- 비용 증가
- 랜섬웨어성 암호화/삭제 시도
```

따라서 유출이 의심되면 즉시 다음을 수행해야 합니다.

```text
1. 해당 Access Key 비활성화
2. 새 키 발급
3. 애플리케이션 Secret 교체
4. Audit Log 확인
5. 권한 범위 재점검
```

---

## 9. Signature Version 4

### 9-1) SigV4란?

Signature Version 4는 AWS API 요청에 사용하는 서명 방식입니다. S3 API 요청을 보낼 때 클라이언트는 요청의 Method, URI, Query String, Header, Payload Hash 등을 정규화한 뒤 Secret Key 기반 HMAC 서명을 생성합니다.

공식 흐름은 크게 다음 단계입니다.

```text
1. Canonical Request 생성
2. String to Sign 생성
3. Signing Key 계산
4. Signature 계산
5. Authorization Header에 추가
```

AWS 공식 문서는 SigV4가 Canonical Request를 만들고, AWS credentials로 서명을 계산한 뒤 Authorization Header에 추가하는 과정이라고 설명합니다.

### 9-2) SigV4가 필요한 이유

HTTP 요청은 중간에서 바뀔 수 있고, 단순 Access Key만으로는 요청자가 실제 Secret Key를 알고 있는지 증명할 수 없습니다.

SigV4는 다음을 검증합니다.

```text
- 요청자가 올바른 Secret Key를 알고 있는가?
- 요청 Method/Path/Query/Header가 변조되지 않았는가?
- 요청 시간이 허용 범위 안에 있는가?
- Payload Hash가 일치하는가?
```

즉 SigV4는 인증과 무결성 검증 역할을 함께 수행합니다.

### 9-3) Canonical Request

Canonical Request는 HTTP 요청을 서명하기 위해 정해진 규칙으로 정규화한 문자열입니다.

형식은 대략 다음과 같습니다.

```text
HTTPMethod
CanonicalURI
CanonicalQueryString
CanonicalHeaders
SignedHeaders
HashedPayload
```

예시:

```text
GET
/my-bucket/report.pdf

host:s3.example.com
x-amz-content-sha256:UNSIGNED-PAYLOAD
x-amz-date:20260701T010000Z

host;x-amz-content-sha256;x-amz-date
UNSIGNED-PAYLOAD
```

여기서 중요한 점은 클라이언트와 서버가 **완전히 같은 Canonical Request**를 만들어야 한다는 것입니다. 공백, Header 순서, URI 인코딩, Host 값이 다르면 서명이 달라집니다.

### 9-4) String to Sign

String to Sign은 Canonical Request의 Hash와 날짜, Region, Service 정보를 조합한 문자열입니다.

```text
AWS4-HMAC-SHA256
20260701T010000Z
20260701/ap-northeast-2/s3/aws4_request
<HashedCanonicalRequest>
```

### 9-5) Signing Key

SigV4는 Secret Key를 바로 HMAC에 쓰지 않고 날짜, Region, Service 값을 단계적으로 섞어서 Signing Key를 만듭니다.

```text
kSecret  = "AWS4" + SecretKey
kDate    = HMAC(kSecret, Date)
kRegion  = HMAC(kDate, Region)
kService = HMAC(kRegion, Service)
kSigning = HMAC(kService, "aws4_request")
```

그 다음 `String to Sign`에 대해 HMAC을 계산하여 최종 Signature를 만듭니다.

### 9-6) Authorization Header

최종 요청에는 Authorization Header가 포함됩니다.

```http
Authorization: AWS4-HMAC-SHA256 \
 Credential=AKIAEXAMPLE/20260701/ap-northeast-2/s3/aws4_request, \
 SignedHeaders=host;x-amz-content-sha256;x-amz-date, \
 Signature=abcdef123456...
```

이 Header를 받은 S3 서버는 동일한 방식으로 Signature를 다시 계산하고, 클라이언트가 보낸 Signature와 비교합니다.

### 9-7) SigV4 실패 원인

S3/Ceph/RustFS 환경에서 서명 오류는 매우 흔합니다.

| 오류 원인 | 설명 |
|---|---|
| 시간 불일치 | 클라이언트/서버 시간이 크게 다름 |
| Host 불일치 | 서명한 Host와 실제 요청 Host가 다름 |
| HTTP/HTTPS 불일치 | Presigned URL 생성 시 Scheme 차이 |
| Path-style/Virtual-hosted-style 혼동 | 버킷 위치가 URI냐 Host냐에 따라 Canonical URI가 달라짐 |
| Region 불일치 | AWS S3에서 흔함. Ceph/RustFS는 구현별 차이 |
| Header 누락 | SignedHeaders에 포함한 Header가 실제 요청에 없음 |
| URI Encoding 차이 | 공백, 한글, 특수문자 Key에서 자주 발생 |
| 프록시가 Header 변경 | Ingress/Nginx/Proxy가 Host나 Header를 바꿈 |

### 9-8) Ceph/RustFS 콘솔 개발 관점

우리 콘솔에서 백엔드를 Ceph와 RustFS로 모두 지원하려면 다음을 고려해야 합니다.

```text
- Endpoint별 Path-style / Virtual-hosted-style 지원 여부
- 서명 생성 시 Host 값
- Presigned URL 생성 시 외부 접속 주소와 내부 API 주소 차이
- Ingress 뒤에 있을 때 X-Forwarded-Host 처리
- 와일드카드 도메인 사용 여부
- 브라우저 직접 업로드 시 CORS + SigV4 동시 처리
```

---

## 10. Pre-signed URL

### 10-1) Pre-signed URL이란?

Pre-signed URL은 서명이 포함된 임시 URL입니다. 이 URL을 가진 사람은 정해진 시간 동안 특정 S3 작업을 수행할 수 있습니다.

대표적으로 다운로드 링크로 사용됩니다.

```text
https://s3.example.com/my-bucket/report.pdf?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...
```

AWS 공식 문서는 Presigned URL이 보안 자격증명을 사용해 제한된 시간 동안 오브젝트 다운로드 권한을 부여하는 URL이라고 설명합니다.

### 10-2) 왜 필요한가?

모든 외부 사용자에게 Access Key/Secret Key를 줄 수는 없습니다. 예를 들어 고객이 10분 동안 파일 하나만 다운로드해야 한다면, 계정을 만드는 것보다 Presigned URL을 발급하는 것이 간단합니다.

사용 사례:

```text
- 임시 다운로드 링크
- 브라우저 직접 업로드
- 고객사 파일 공유
- 이메일 첨부 대신 링크 제공
- 대용량 파일 전달
```

### 10-3) URL 자체가 권한이다

Presigned URL은 URL에 권한이 담겨 있습니다. 따라서 URL이 유출되면 만료 전까지 접근이 가능합니다.

```text
Access Key를 몰라도
Secret Key를 몰라도
Presigned URL을 가진 사람은 접근 가능
```

그래서 운영에서는 다음 원칙이 필요합니다.

```text
- 만료 시간을 짧게 설정
- HTTPS 사용
- 필요한 Method만 허용
- 필요한 Object만 허용
- 가능하면 다운로드 횟수/감사 로그 확인
- URL 로그 노출 주의
```

### 10-4) Temporary Credential과 Presigned URL 만료

Temporary Credential로 Presigned URL을 만들면 URL 만료 시간보다 임시 자격증명 만료가 더 빠를 수 있습니다. AWS 문서에서는 임시 자격증명으로 만든 Presigned URL은 해당 자격증명이 만료되면 URL도 만료된다고 설명합니다.

```text
Presigned URL 만료: 1시간
STS Credential 만료: 15분
실제 접근 가능 시간: 15분
```

### 10-5) 콘솔 기능 설계

콘솔에서는 다음 기능을 제공할 수 있습니다.

```text
- 다운로드 Presigned URL 생성
- 업로드 Presigned URL 생성
- 만료 시간 선택
- URL 복사
- QR 코드 생성
- URL 폐기 기능 또는 키 비활성화 안내
- 생성자/대상/만료 시간 Audit Log 기록
```

단, S3 표준 자체는 이미 발급된 Presigned URL 하나만 골라서 폐기하는 기능을 제공하지 않습니다. 보통은 URL을 만든 자격증명을 비활성화하거나 정책 조건으로 제한합니다.

---

## 11. STS와 Temporary Credential

### 11-1) STS란?

STS는 Security Token Service입니다. 장기 Access Key 대신 일정 시간만 유효한 임시 자격증명을 발급하는 서비스입니다.

임시 자격증명은 보통 다음 값으로 구성됩니다.

```text
Access Key
Secret Key
Session Token
Expiration
```

장기 키와 다른 점은 Session Token과 만료 시간이 있다는 것입니다.

### 11-2) 왜 STS가 필요한가?

장기 Access Key는 유출되면 위험합니다. STS는 짧은 시간만 유효한 자격증명을 발급하여 위험을 줄입니다.

```text
사용자 로그인
  ↓
권한 확인
  ↓
임시 Access Key / Secret Key / Session Token 발급
  ↓
제한된 시간 동안 S3 API 호출
  ↓
만료 후 자동 사용 불가
```

### 11-3) OIDC와 STS 연동

OIDC 로그인과 STS를 함께 쓰면 회사 계정 기반 로그인 후 S3 임시 권한을 발급할 수 있습니다.

```text
사용자
  ↓ 회사 계정 로그인
IdP / Keycloak
  ↓ OIDC Token
S3 Console Backend
  ↓ STS 요청
S3 Backend / RustFS / Ceph
  ↓ Temporary Credential
브라우저 또는 Backend가 S3 API 호출
```

### 11-4) 콘솔 개발 관점

STS를 사용하면 콘솔 서버가 모든 사용자의 장기 Secret Key를 보관하지 않아도 됩니다. 대신 로그인된 사용자에게 제한된 임시 자격증명을 발급하고, 해당 권한으로 S3 API를 호출하게 할 수 있습니다.

다만 Ceph RGW와 RustFS의 STS/OIDC 지원 수준은 버전과 설정에 따라 다를 수 있으므로, 실제 기능 지원 여부를 Capability로 분리해야 합니다.

---

## 12. OIDC

### 12-1) OIDC란?

OIDC는 OpenID Connect입니다. OAuth 2.0 위에 신원 인증 계층을 추가한 표준입니다. 쉽게 말하면 외부 로그인 연동 방식입니다.

```text
OAuth 2.0 = 권한 위임 중심
OIDC      = OAuth 2.0 + 로그인/신원 정보
```

OIDC를 사용하면 S3 Console 자체 계정이 아니라 회사 계정, Keycloak, Azure AD, Google Workspace 같은 외부 IdP로 로그인할 수 있습니다.

### 12-2) OIDC 구성 요소

| 구성 요소 | 설명 |
|---|---|
| End User | 로그인하는 사용자 |
| Relying Party | OIDC 로그인을 사용하는 애플리케이션. 여기서는 S3 Console |
| OpenID Provider | IdP. 예: Keycloak, Okta, Azure AD |
| ID Token | 사용자 신원 정보가 담긴 토큰 |
| Access Token | API 접근 권한을 나타내는 토큰 |
| Refresh Token | 토큰 갱신에 사용하는 토큰 |

### 12-3) OIDC 로그인 흐름

```text
1. 사용자가 S3 Console 접속
2. Console이 IdP 로그인 페이지로 Redirect
3. 사용자가 회사 계정으로 로그인
4. IdP가 Authorization Code 발급
5. Console Backend가 Code를 Token으로 교환
6. ID Token 검증
7. 사용자 세션 생성
8. 사용자 권한에 따라 메뉴 표시
```

### 12-4) S3 Console에서 OIDC가 필요한 이유

Access Key/Secret Key만으로 콘솔 로그인을 처리하면 다음 문제가 생깁니다.

```text
- 사용자 퇴사/부서 이동과 계정 관리 연동이 어려움
- MFA 적용이 어려움
- 회사 계정 정책과 분리됨
- 감사 로그 연동이 약함
```

OIDC를 사용하면 회사 계정 체계와 연동할 수 있습니다.

```text
회사 계정 로그인
MFA
그룹 기반 권한
퇴사자 계정 비활성화
SSO
```

### 12-5) OIDC와 IAM 매핑

S3 Console에서는 OIDC 사용자/그룹을 내부 IAM 권한과 매핑해야 합니다.

```text
OIDC Claim
  ↓
email: alice@example.com
roles: [storage-admin, data-user]
  ↓
Console Role Mapping
  ↓
storage-admin → consoleAdmin 정책
```

이 매핑이 없으면 로그인은 되었지만 어떤 버킷을 볼 수 있는지 결정할 수 없습니다.

---

## 13. IdP

### 13-1) IdP란?

IdP는 Identity Provider입니다. 사용자의 신원을 확인하고 인증 결과를 애플리케이션에 전달하는 서비스입니다.

대표 예시는 다음과 같습니다.

```text
Keycloak
Okta
Azure AD / Microsoft Entra ID
Google Workspace
Auth0
```

### 13-2) IdP가 담당하는 것

```text
- 사용자 계정 관리
- 비밀번호 정책
- MFA
- 그룹 관리
- 로그인 세션
- 토큰 발급
- 계정 비활성화
```

S3 Console이 직접 모든 사용자 계정과 비밀번호를 관리하지 않고 IdP에 위임하면 보안 관리가 단순해집니다.

### 13-3) Console과 IdP 연동 시 필요한 설정

```text
- Issuer URL
- Client ID
- Client Secret
- Redirect URI
- Scope
- Claim Mapping
- Logout URL
```

운영에서 가장 자주 발생하는 문제는 Redirect URI 불일치입니다. 콘솔 외부 도메인이 `https://s3-console.example.com`인데 IdP에는 내부 주소나 HTTP 주소가 등록되어 있으면 로그인 후 Callback이 실패합니다.

---

## 14. IAM

### 14-1) IAM이란?

IAM은 Identity and Access Management입니다. S3에서 IAM은 다음 질문에 답하는 체계입니다.

```text
누가?
어떤 리소스에?
무슨 동작을?
어떤 조건에서?
허용 또는 거부되는가?
```

### 14-2) Authentication과 Authorization 차이

IAM을 이해하려면 인증과 인가를 구분해야 합니다.

```text
Authentication = 너는 누구인가?
Authorization  = 너는 이것을 할 수 있는가?
```

예를 들어 Access Key/Secret Key로 서명 검증이 성공하면 인증은 된 것입니다. 하지만 해당 사용자가 `DeleteObject` 권한이 없으면 인가는 실패합니다.

```text
SigV4 검증 성공
  ↓
사용자 식별 성공
  ↓
Policy 평가
  ↓
권한 없으면 403 AccessDenied
```

### 14-3) IAM의 주요 구성 요소

| 구성 요소 | 설명 |
|---|---|
| User | 개별 사용자 또는 서비스 계정 |
| Group | 여러 User를 묶는 단위 |
| Role | 특정 상황에서 맡는 권한 집합 |
| Policy | 권한 규칙 |
| Principal | 요청 주체 |
| Action | 수행하려는 작업 |
| Resource | 대상 리소스 |
| Condition | 조건 |

### 14-4) Least Privilege

IAM의 핵심 원칙은 최소 권한입니다.

```text
필요한 대상에
필요한 작업만
필요한 기간 동안
허용한다.
```

나쁜 예:

```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

좋은 예:

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:ListBucket"],
  "Resource": [
    "arn:aws:s3:::project-a",
    "arn:aws:s3:::project-a/*"
  ]
}
```

### 14-5) Console 개발 관점

콘솔에서는 IAM을 JSON 편집기로만 제공하면 사용자가 이해하기 어렵습니다. 다음과 같은 비주얼 정책 빌더가 필요합니다.

```text
버킷: project-a
권한:
[✓] 목록 보기
[✓] 다운로드
[ ] 업로드
[ ] 삭제
[ ] 정책 변경

고급 설정:
- IP 제한
- 만료 시간
- Prefix 제한
```

그리고 내부적으로 IAM Policy JSON을 생성하도록 설계하는 것이 좋습니다.

---

## 15. User / Group / Role

### 15-1) User

User는 S3 API를 호출하는 개별 주체입니다. 사람일 수도 있고 애플리케이션일 수도 있습니다.

```text
alice
backup-service
airflow-pipeline
ml-training-job
```

User에는 보통 Access Key, Secret Key, Policy, Group, Quota가 연결됩니다.

### 15-2) Group

Group은 여러 User를 묶는 단위입니다.

```text
data-readers
backup-writers
storage-admins
```

Group에 정책을 연결하면 해당 그룹에 속한 사용자들이 같은 권한을 받습니다.

### 15-3) Role

Role은 특정 상황에서 임시로 맡는 권한 집합입니다. AWS에서는 Role과 STS가 강하게 연결됩니다. Ceph/RustFS에서는 AWS IAM Role과 동일한 수준으로 지원되는지 확인이 필요합니다.

Console 설계에서는 Role을 다음과 같이 표현할 수 있습니다.

```text
Console Admin
Storage Operator
Read Only User
Bucket Owner
Auditor
```

### 15-4) 사용자 관리 화면에 필요한 정보

```text
- 사용자 이름
- 상태
- 소속 그룹
- 연결 정책
- Access Key 개수
- 마지막 사용 시간
- 사용 용량
- Quota
- 생성일
- 수정일
```

기존 RustFS Console 화면에서 부족했던 부분은 사용자별 사용량/쿼터/마지막 접근 정보입니다. 우리 콘솔에서는 대표님 요구사항인 “사용자별 관리”를 이 영역에서 강화할 수 있습니다.

---

## 16. IAM Policy

### 16-1) IAM Policy란?

IAM Policy는 권한 규칙을 JSON 문서로 표현한 것입니다. S3에서는 사용자가 어떤 버킷과 오브젝트에 어떤 API를 호출할 수 있는지를 정의합니다.

기본 구조는 다음과 같습니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::project-a/*"]
    }
  ]
}
```

### 16-2) Statement 구성 요소

| 요소 | 의미 |
|---|---|
| Effect | Allow 또는 Deny |
| Action | 허용/거부할 API |
| Resource | 대상 리소스 ARN |
| Condition | 조건 |
| Principal | Bucket Policy 등 리소스 정책에서 요청 주체 지정 |

### 16-3) Action

Action은 허용할 API입니다.

```text
s3:ListBucket
s3:GetObject
s3:PutObject
s3:DeleteObject
s3:GetBucketPolicy
s3:PutBucketPolicy
s3:AbortMultipartUpload
```

주의할 점은 버킷 목록 조회와 오브젝트 다운로드가 서로 다른 권한이라는 것입니다.

```text
목록 조회: s3:ListBucket        → bucket ARN 필요
다운로드: s3:GetObject          → object ARN 필요
```

### 16-4) Resource

S3에서는 버킷 자체와 버킷 안의 오브젝트 ARN이 다릅니다.

```text
버킷 ARN:    arn:aws:s3:::project-a
오브젝트 ARN: arn:aws:s3:::project-a/*
```

따라서 목록 조회와 오브젝트 조회를 모두 허용하려면 둘 다 필요합니다.

```json
{
  "Effect": "Allow",
  "Action": ["s3:ListBucket", "s3:GetObject"],
  "Resource": [
    "arn:aws:s3:::project-a",
    "arn:aws:s3:::project-a/*"
  ]
}
```

### 16-5) Condition

Condition은 특정 조건에서만 허용/거부하는 기능입니다.

예시:

```text
- 특정 IP에서만 허용
- 특정 Prefix만 허용
- TLS 사용 시에만 허용
- MFA 사용 시에만 허용
- 특정 시간대만 허용
```

Console에서는 Condition을 고급 옵션으로 제공할 수 있습니다.

### 16-6) Deny 우선 원칙

명시적 Deny는 Allow보다 우선합니다. 따라서 어떤 정책에서 Allow가 있어도 다른 정책에서 명시적으로 Deny하면 요청은 거부됩니다.

```text
Explicit Deny > Allow > Default Deny
```

Default Deny는 아무 정책도 없으면 기본적으로 거부된다는 의미입니다.

---

## 17. Bucket Policy

### 17-1) Bucket Policy란?

Bucket Policy는 버킷에 직접 붙는 리소스 기반 정책입니다. IAM Policy가 사용자 중심이라면 Bucket Policy는 버킷 중심입니다.

```text
IAM Policy     = 이 사용자는 무엇을 할 수 있는가?
Bucket Policy  = 이 버킷은 누구에게 무엇을 허용하는가?
```

### 17-2) Bucket Policy 예시

특정 사용자에게 읽기 권한을 주는 예시입니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:user/alice"},
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::project-a/*"]
    }
  ]
}
```

### 17-3) Public Bucket 주의

Bucket Policy로 public read를 허용하면 외부 누구나 오브젝트를 읽을 수 있습니다.

```json
{
  "Effect": "Allow",
  "Principal": "*",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::public-bucket/*"
}
```

정적 웹사이트나 공개 이미지 저장소에서는 사용할 수 있지만, 일반 업무 데이터에서는 매우 위험합니다. 콘솔에서 public policy 설정 시 강한 경고가 필요합니다.

### 17-4) Console 개발 관점

Bucket Policy 화면은 다음 두 모드를 제공하는 것이 좋습니다.

```text
1. 쉬운 모드
   - 비공개
   - 공개 읽기
   - 특정 사용자/그룹만 허용
   - 특정 IP만 허용

2. 고급 모드
   - JSON 직접 편집
   - 문법 검증
   - 위험 정책 경고
```

---

## 18. ACL

### 18-1) ACL이란?

ACL은 Access Control List입니다. 버킷이나 오브젝트에 대해 누가 어떤 권한을 가지는지 목록 형태로 관리하는 오래된 접근 제어 방식입니다.

### 18-2) IAM Policy와 ACL 차이

| 구분 | IAM/Bucket Policy | ACL |
|---|---|---|
| 표현 방식 | JSON 정책 | 권한 목록 |
| 세밀도 | 높음 | 제한적 |
| 조건 지원 | 강함 | 약함 |
| 현대적 권장 방식 | 예 | 제한적 |

현대적인 S3 설계에서는 ACL보다 IAM Policy와 Bucket Policy 중심으로 권한을 관리하는 것이 일반적으로 더 명확합니다.

### 18-3) 콘솔에서 ACL을 어떻게 다룰 것인가

Ceph/RustFS 호환성을 고려하면 ACL 기능은 남겨둘 수 있지만, 기본 화면에서는 숨기고 고급 설정으로 제공하는 것이 좋습니다.

```text
기본 권한 관리: Policy 기반
고급 호환성 관리: ACL
```

---

## 19. Quota

### 19-1) Quota란?

Quota는 사용량 제한입니다. S3 환경에서는 사용자별, 버킷별, 테넌트별로 용량이나 오브젝트 수 제한을 걸 수 있습니다.

```text
사용자 alice: 최대 100GB
버킷 project-a: 최대 1TB
고객사 tenant-a: 최대 10TB
```

### 19-2) Quota가 필요한 이유

```text
- 특정 사용자의 과도한 사용 방지
- 고객사별 할당량 관리
- 비용/과금 기준 제공
- 장애 예방
- 용량 계획 수립
```

### 19-3) 콘솔에서 보여줘야 할 Quota 정보

```text
사용자명
현재 사용량
할당량
사용률
오브젝트 수
버킷 수
초과 여부
최근 증가량
```

예시:

```text
alice  72GB / 100GB  72%  정상
bob    98GB / 100GB  98%  경고
```

### 19-4) Ceph와 RustFS 차이

Ceph RGW는 RGW Admin API와 `radosgw-admin` 명령으로 사용자/버킷 쿼터를 다룰 수 있습니다. RustFS는 자체 Admin API/Console 기능을 통해 quota를 제공할 수 있습니다. 단, API 모양은 다르므로 우리 콘솔 백엔드에서 어댑터로 분리해야 합니다.

---

## 20. SSE

### 20-1) SSE란?

SSE는 Server-Side Encryption입니다. 클라이언트가 데이터를 업로드하면 S3 서버가 저장 전에 데이터를 암호화합니다.

```text
Client
  ↓ Plain Object Upload
S3 Server
  ↓ Encrypt
Encrypted Object Stored
```

SSE는 “저장 시 암호화”입니다. 네트워크 전송 구간 암호화인 TLS와 다릅니다.

```text
TLS = 전송 중 암호화
SSE = 저장 시 암호화
```

### 20-2) SSE가 필요한 이유

```text
- 디스크 유출 시 데이터 보호
- 규정 준수
- 고객사 보안 요구사항 충족
- 내부자 위험 완화
- 백업/스냅샷 데이터 보호
```

### 20-3) SSE 종류

| 방식 | 키 관리 주체 | 특징 |
|---|---|---|
| SSE-S3 | S3 시스템 | 가장 단순 |
| SSE-KMS | KMS | 키 제어/감사 강화 |
| SSE-C | 클라이언트 | 고객이 키 제공, 운영 복잡 |

AWS S3 공식 문서에 따르면 Amazon S3는 모든 버킷에 기본 암호화를 적용하며 기본 옵션은 SSE-S3입니다. 또한 SSE-KMS는 AWS KMS와 통합되어 더 세밀한 키 제어와 감사가 가능합니다.

---

## 21. SSE-S3

### 21-1) SSE-S3란?

SSE-S3는 S3 서비스가 관리하는 키로 서버 측 암호화를 수행하는 방식입니다. 사용자는 별도 KMS를 구성하지 않아도 됩니다.

```text
Client Upload
  ↓
S3가 자체 관리 키로 암호화
  ↓
Encrypted Object 저장
```

### 21-2) 장점

```text
- 설정이 단순함
- 운영 부담이 낮음
- 기본 암호화로 적합
- 별도 키 관리 시스템이 없어도 됨
```

### 21-3) 단점

```text
- 키 정책을 세밀하게 제어하기 어려움
- 키 사용 감사가 제한적일 수 있음
- 규정 준수 환경에서는 SSE-KMS가 더 적합할 수 있음
```

### 21-4) 콘솔 UI

```text
[버킷 기본 암호화]
(•) SSE-S3: 스토리지 시스템 관리 키 사용
( ) SSE-KMS: KMS 키 사용
( ) SSE-C: 클라이언트 제공 키 사용
```

---

## 22. SSE-KMS

### 22-1) SSE-KMS란?

SSE-KMS는 KMS가 관리하는 키를 사용하여 S3 오브젝트를 암호화하는 방식입니다.

```text
Client Upload
  ↓
S3 Server
  ↓ KMS에 키 사용 요청
KMS
  ↓
S3 Server가 데이터 암호화
  ↓
Encrypted Object 저장
```

### 22-2) 왜 SSE-KMS를 쓰는가?

```text
- 키 접근 권한을 별도 관리
- 키 사용 이력 감사
- 키 회전
- 키 비활성화/폐기
- 규정 준수 대응
```

AWS 문서에서도 SSE-KMS는 KMS와 통합되어 별도 키 관리, 정책 제어, CloudTrail을 통한 추적 등 더 많은 제어를 제공한다고 설명합니다.

### 22-3) Envelope Encryption

KMS 기반 암호화에서는 일반적으로 Envelope Encryption 개념이 중요합니다.

```text
Data Key로 실제 데이터를 암호화
Master Key/KMS Key로 Data Key를 암호화
암호화된 Data Key를 메타데이터와 함께 저장
```

이 방식은 대용량 데이터를 매번 KMS 키로 직접 암호화하지 않고, 데이터 암호화용 키를 안전하게 보호하기 위해 사용됩니다.

### 22-4) Ceph/RustFS에서의 KMS

Ceph와 RustFS에서 SSE-KMS를 구현하려면 외부 KMS 또는 호환 KMS 연동이 필요할 수 있습니다. 예를 들어 Vault/OpenBao 같은 키 관리 시스템과 연동하는 구조를 고려할 수 있습니다.

콘솔에서는 백엔드가 KMS를 지원하는지 Capability로 확인해야 합니다.

```text
capabilities.sseKms = true / false
```

---

## 23. SSE-C

### 23-1) SSE-C란?

SSE-C는 Server-Side Encryption with Customer-Provided Keys입니다. 클라이언트가 암호화 키를 요청 Header로 제공하고, S3 서버는 그 키로 암호화/복호화를 수행합니다.

```text
Client
  ↓ Object + Customer Key
S3 Server
  ↓ Customer Key로 암호화
Encrypted Object 저장
```

### 23-2) SSE-C의 위험성

SSE-C에서 S3 서버는 고객 키를 저장하지 않습니다. 따라서 사용자가 키를 잃어버리면 오브젝트를 복호화할 수 없습니다. AWS 공식 문서도 SSE-C 사용 시 S3가 암호화 키를 저장하지 않으며, 사용자가 어떤 오브젝트에 어떤 키를 썼는지 직접 추적해야 한다고 설명합니다.

### 23-3) HTTPS 필수

SSE-C는 고객 제공 키가 요청 Header에 포함됩니다. 따라서 HTTPS가 필수입니다. HTTP로 전송하면 키가 노출될 수 있습니다.

### 23-4) 콘솔에서 SSE-C를 어떻게 다룰 것인가

SSE-C는 일반 사용자에게 노출하면 운영 사고가 날 가능성이 큽니다.

권장 UI:

```text
- 기본 화면에서는 숨김
- 고급 설정으로 제공
- 강한 경고 문구 표시
- 키를 서버에 저장하지 않음을 명확히 표시
- 키 분실 시 복구 불가 안내
```

---

## 24. KMS

### 24-1) KMS란?

KMS는 Key Management Service입니다. 암호화 키를 생성, 저장, 사용, 회전, 폐기, 감사하는 시스템입니다.

KMS는 단순히 키를 저장하는 장소가 아닙니다. 키 사용 권한과 키 사용 이력까지 관리하는 보안 계층입니다.

### 24-2) KMS가 필요한 이유

암호화에서 가장 어려운 문제는 암호화 알고리즘이 아니라 키 관리입니다.

```text
- 키를 어디에 저장할 것인가?
- 누가 키를 사용할 수 있는가?
- 키가 유출되면 어떻게 할 것인가?
- 키를 언제 회전할 것인가?
- 키 사용 이력을 어떻게 감사할 것인가?
```

KMS는 이 문제를 해결하기 위한 서비스입니다.

### 24-3) Key Rotation

Key Rotation은 암호화 키를 주기적으로 교체하는 것입니다. 키가 장기간 사용되면 유출 위험이 커지고, 규정 준수 요구사항을 만족하기 어려울 수 있습니다.

### 24-4) Key Policy

KMS Key 자체에도 권한 정책이 필요합니다. S3 오브젝트에 접근할 권한이 있어도 KMS Key 사용 권한이 없으면 SSE-KMS 오브젝트를 읽을 수 없습니다.

```text
S3 GetObject 권한 있음
KMS Decrypt 권한 없음
  ↓
다운로드 실패
```

이 부분은 실무에서 자주 헷갈립니다.

### 24-5) Console 개발 관점

KMS 화면에서는 다음 정보를 표시하면 좋습니다.

```text
- 키 목록
- 키 상태
- 키 별칭
- 연결된 버킷
- 최근 사용 시간
- 키 회전 여부
- 키 정책
- 감사 로그 링크
```

단, KMS를 직접 구현할지 외부 Vault/OpenBao/KMS와 연동할지는 별도 아키텍처 결정이 필요합니다.

---

## 25. Object Lock

### 25-1) Object Lock이란?

Object Lock은 오브젝트를 일정 기간 삭제하거나 덮어쓰지 못하게 보호하는 기능입니다. 랜섬웨어 대응, 감사 로그 보존, 금융/공공 기록 보존에서 중요합니다.

```text
Object 저장
  ↓
Retention Period 설정
  ↓
기간 동안 삭제/수정 제한
```

### 25-2) Object Lock과 Versioning

Object Lock은 보통 Versioning과 연결됩니다. 특정 오브젝트 Key 전체가 아니라 특정 버전 단위로 보존 정책이 적용될 수 있습니다.

```text
report.pdf
├── version 1  retention: 1년
├── version 2  retention: 3년
└── version 3  retention: 없음
```

### 25-3) Governance Mode와 Compliance Mode

AWS S3 Object Lock에는 Governance Mode와 Compliance Mode가 있습니다.

| 모드 | 의미 |
|---|---|
| Governance Mode | 특별 권한이 있으면 우회 가능 |
| Compliance Mode | root에 가까운 권한도 기간 전 삭제/수정 불가 |

Compliance Mode는 매우 강력하므로 실수로 설정하면 운영자가 삭제하지 못하는 데이터가 생길 수 있습니다.

### 25-4) Legal Hold

Legal Hold는 기간과 관계없이 오브젝트 버전에 보존 잠금을 거는 기능입니다. 법무/감사 요구사항 때문에 특정 데이터를 삭제하지 못하게 해야 할 때 사용합니다.

### 25-5) 콘솔 설계 주의점

Object Lock 화면에는 강한 경고가 필요합니다.

```text
- Compliance Mode는 되돌리기 어려움
- 보존 기간 동안 삭제 불가
- 스토리지 비용 증가 가능
- Versioning과 함께 동작함
- 권한이 있어도 삭제 실패할 수 있음
```

---

## 26. WORM

### 26-1) WORM이란?

WORM은 Write Once Read Many입니다. 한 번 기록한 뒤 여러 번 읽을 수 있지만, 수정이나 삭제는 제한하는 보존 방식입니다.

```text
Write Once
Read Many
Delete/Modify Restricted
```

### 26-2) WORM이 필요한 곳

```text
- 감사 로그
- 금융 거래 기록
- 공공 기록물
- 의료 기록
- 보안 이벤트 로그
- 백업 데이터
```

### 26-3) Object Lock과 WORM 관계

S3에서 WORM 요구사항을 구현하는 대표 기능이 Object Lock입니다.

```text
WORM 요구사항
  ↓
S3 Object Lock으로 구현
```

---

## 27. 보안 트러블슈팅

### 27-1) 403 AccessDenied

가능 원인:

```text
- IAM Policy에 Action 없음
- Resource ARN이 잘못됨
- Bucket Policy에서 Deny
- KMS 권한 없음
- Object Lock/Retention으로 삭제 불가
- ACL/Ownership 문제
```

확인 순서:

```text
1. 어떤 API가 실패했는지 확인
2. 요청 주체 확인
3. IAM Policy 확인
4. Bucket Policy 확인
5. KMS 권한 확인
6. Object Lock 여부 확인
7. Audit Log 확인
```

### 27-2) SignatureDoesNotMatch

가능 원인:

```text
- Secret Key 오류
- Host 불일치
- Path-style/Virtual-hosted-style 불일치
- URI Encoding 문제
- 시간 불일치
- Header 변경
- Presigned URL 만료
```

### 27-3) TLS 인증서 오류

가능 원인:

```text
- 인증서 만료
- CN/SAN 불일치
- 자체 서명 인증서
- 중간 인증서 누락
- 와일드카드 인증서 범위 불일치
```

### 27-4) Presigned URL은 생성됐는데 다운로드 실패

가능 원인:

```text
- URL 만료
- 임시 자격증명 만료
- Host가 바뀜
- 프록시가 Query String 변경
- Object Key 인코딩 문제
- Method 불일치
```

### 27-5) SSE-KMS 오브젝트 다운로드 실패

가능 원인:

```text
- s3:GetObject는 있지만 kms:Decrypt 권한 없음
- KMS Key 비활성화
- KMS Key 정책에서 거부
- 다른 계정/테넌트 키 사용
```

---

## 28. S3 Console 보안 설계 체크리스트

```text
[인증]
- HTTPS 강제 여부
- OIDC/IdP 연동 여부
- 세션 만료 정책
- MFA 연동 가능성

[Access Key]
- Secret Key 1회 표시
- 키 만료일 설정
- 키 비활성화 기능
- 키 Rotation 가이드
- Export 시 경고

[권한]
- 사용자/그룹/정책 관리
- 최소 권한 템플릿
- 위험 정책 감지
- Public Bucket 경고
- Deny 정책 시각화

[암호화]
- 버킷 기본 암호화 표시
- SSE-S3/SSE-KMS/SSE-C 구분
- KMS Key 상태 표시
- SSE-C 사용 시 강한 경고

[보존]
- Object Lock 상태 표시
- Retention 기간 표시
- Legal Hold 표시
- Compliance Mode 경고

[감사]
- 누가 어떤 정책을 바꿨는지 기록
- 누가 키를 생성했는지 기록
- Presigned URL 생성 이력 기록
```

---

## 29. AWS / Ceph / RustFS / MinIO 관점 비교

| 기능 | AWS S3 | Ceph RGW | RustFS | MinIO |
|---|---|---|---|---|
| Access Key / Secret Key | 지원 | 지원 | 지원 | 지원 |
| SigV4 | 표준 | S3 호환 구현 | S3 호환 구현 | S3 호환 구현 |
| IAM Policy | AWS IAM | RGW IAM/Policy 호환 범위 확인 필요 | 자체 IAM/Policy | 자체 IAM/Policy |
| Bucket Policy | 지원 | 지원 범위 확인 필요 | 지원 | 지원 |
| STS | 지원 | 지원 범위/버전 확인 필요 | 지원 여부 확인 필요 | 지원 |
| OIDC | IAM Identity Center/외부 IdP 연동 | Keystone/OIDC 등 구성 확인 | 콘솔/서버 지원 확인 | 지원 |
| SSE-S3 | 지원 | 지원 | 지원 여부 확인 | 지원 |
| SSE-KMS | AWS KMS | 외부 KMS 연동 필요 | KMS 연동 확인 | KES/KMS 연동 |
| Object Lock | 지원 | 지원 범위 확인 | 지원 여부 확인 | 지원 |

주의: 위 표는 설계 관점의 비교입니다. 실제 지원 여부는 제품 버전, 배포 방식, 설정에 따라 달라질 수 있습니다. 운영 기능으로 넣기 전에는 반드시 현재 설치된 Ceph/RustFS 버전에서 API 테스트가 필요합니다.

---

## 30. 실습 명령어 예시

### 30-1) AWS CLI로 버킷 목록 확인

```bash
aws --endpoint-url https://s3.example.com s3 ls
```

### 30-2) Access Key 환경변수 설정

```bash
export AWS_ACCESS_KEY_ID="YOUR_ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="YOUR_SECRET_KEY"
export AWS_DEFAULT_REGION="us-east-1"
```

### 30-3) 오브젝트 업로드

```bash
aws --endpoint-url https://s3.example.com s3 cp ./test.txt s3://my-bucket/test.txt
```

### 30-4) 오브젝트 다운로드

```bash
aws --endpoint-url https://s3.example.com s3 cp s3://my-bucket/test.txt ./download.txt
```

### 30-5) Presigned URL 생성

```bash
aws --endpoint-url https://s3.example.com s3 presign s3://my-bucket/test.txt --expires-in 600
```

### 30-6) mc alias 설정

```bash
mc alias set qcs https://s3.example.com ACCESS_KEY SECRET_KEY
```

### 30-7) mc로 버킷 목록 확인

```bash
mc ls qcs
```

### 30-8) mc로 정책 확인

```bash
mc anonymous get qcs/my-bucket
```

---

## 31. 핵심 정리

```text
HTTP는 S3 API의 기반이다.
HTTPS는 HTTP에 TLS를 적용한 보안 통신이다.
TLS는 통신 채널을 보호하고, SigV4는 요청자를 검증한다.
Access Key는 식별자이고 Secret Key는 서명 생성용 비밀값이다.
Secret Key는 요청에 직접 보내지 않는다.
SigV4는 Canonical Request, String to Sign, HMAC Signature로 구성된다.
Presigned URL은 URL 자체가 임시 권한이다.
STS는 만료 시간이 있는 임시 자격증명을 발급한다.
OIDC는 외부 로그인 연동이며 IdP는 신원 제공자다.
IAM은 누가 어떤 리소스에 어떤 작업을 할 수 있는지 정의한다.
IAM Policy는 사용자 중심 권한이고 Bucket Policy는 리소스 중심 권한이다.
Quota는 사용자/버킷/테넌트별 사용량 제한이다.
SSE는 저장 시 서버 측 암호화다.
SSE-S3는 단순하고, SSE-KMS는 키 제어와 감사가 강하며, SSE-C는 운영 위험이 크다.
KMS는 키 생성, 사용, 회전, 감사, 폐기를 관리한다.
Object Lock과 WORM은 데이터 삭제/수정을 제한하는 보존 기능이다.
S3 Console은 인증, 권한, 암호화, 보존을 UI에서 명확히 분리해야 한다.
```

---

## 32. 공식 문서 링크 모음

### AWS S3 / IAM / SigV4

- Amazon S3 User Guide: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html
- Amazon S3 API Reference: https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html
- AWS Signature Version 4: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_sigv.html
- Create a signed AWS API request: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_sigv-create-signed-request.html
- AWS S3 Presigned URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- Sharing objects with presigned URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- AWS STS Temporary Security Credentials: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html
- AWS IAM Policies: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html
- AWS IAM JSON Policy Elements: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements.html
- AWS S3 Bucket Policy: https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html
- AWS S3 ACL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/acl-overview.html

### AWS S3 Encryption / Object Lock

- AWS S3 Server-Side Encryption: https://docs.aws.amazon.com/AmazonS3/latest/userguide/serv-side-encryption.html
- AWS S3 SSE-S3: https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingServerSideEncryption.html
- AWS S3 SSE-KMS: https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingKMSEncryption.html
- AWS S3 SSE-C: https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerSideEncryptionCustomerKeys.html
- AWS KMS Developer Guide: https://docs.aws.amazon.com/kms/latest/developerguide/overview.html
- AWS S3 Object Lock: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html

### OIDC / OAuth

- OpenID Connect Core: https://openid.net/specs/openid-connect-core-1_0.html
- OAuth 2.0 RFC 6749: https://datatracker.ietf.org/doc/html/rfc6749
- Keycloak Documentation: https://www.keycloak.org/documentation

### Ceph RGW

- Ceph Documentation: https://docs.ceph.com/en/latest/
- Ceph Object Gateway: https://docs.ceph.com/en/latest/radosgw/
- Ceph RGW Admin Ops API: https://docs.ceph.com/en/latest/radosgw/adminops/
- Ceph RGW STS: https://docs.ceph.com/en/latest/radosgw/STS/
- Ceph RGW Keystone: https://docs.ceph.com/en/latest/radosgw/keystone/

### RustFS / MinIO

- RustFS Official: https://rustfs.com/
- RustFS Docs: https://rustfs.org/
- RustFS GitHub: https://github.com/rustfs/rustfs
- MinIO Documentation: https://min.io/docs/minio/linux/index.html
- MinIO Client mc: https://min.io/docs/minio/linux/reference/minio-mc.html
- MinIO Security and Access Management: https://min.io/docs/minio/linux/administration/identity-access-management.html
- MinIO Server-Side Encryption: https://min.io/docs/minio/linux/administration/server-side-encryption.html
