# 코드 및 문서 자산 보안 등급 관리 — AWS 서비스 활용 제안

## 개요

정책 담당자가 수립한 자산 보안 등급(극비/대외비/사내한정/공개)을 AWS 환경에서 기술적으로 구현하는 방안입니다.
코드 자산은 **On-Prem Git**에서 관리하므로, AWS 측에서는 **S3에 저장되는 문서/로그/아티팩트**와 **On-Prem Git에서 AWS로 유입되는 빌드 결과물**에 대한 등급 관리를 다룹니다.

---

## 보안 등급 체계 (예시)

| 등급 | 태그 값 | 설명 | 예시 자산 |
|------|---------|------|-----------|
| **극비** | `Classification:TopSecret` | 유출 시 사업 치명적 영향, 최소 인원만 접근 | 핵심 알고리즘, 고객 개인정보 원본, 보안 키 |
| **대외비** | `Classification:Confidential` | 특정 부서/프로젝트만 접근 | LLM 프롬프트 로그, 아키텍처 설계서, API 키 설정 |
| **사내한정** | `Classification:Internal` | 전 임직원 접근 가능, 외부 유출 금지 | 운영 매뉴얼, 일반 빌드 아티팩트 |
| **공개** | `Classification:Public` | 외부 공개 가능 | 오픈소스, 마케팅 자료 |

---

## 1. S3 — 문서/데이터 자산 등급 관리

### 1-1. 태그 기반 접근 제어 (ABAC)

S3 객체에 `Classification` 태그를 부착하고, 사용자 IAM Role에 `ClearanceLevel` 태그를 부여하여 매칭:

```json
{
  "Effect": "Deny",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::assets-topsecret/*",
  "Condition": {
    "StringNotEquals": {
      "aws:PrincipalTag/ClearanceLevel": "TopSecret"
    }
  }
}
```

### 1-2. S3 Access Points — 등급별 접근 경로 분리

```
assets-bucket
  ├── ap-public       → prefix: public/     (인터넷 허용)
  ├── ap-internal     → prefix: internal/   (VPC 전용)
  ├── ap-confidential → prefix: confidential/ (VPC 전용 + 팀 제한)
  └── ap-topsecret    → prefix: topsecret/  (VPC 전용 + CISO Role만)
```

### 1-3. S3 Access Grants — AD 그룹 직접 매핑

IAM Identity Center 연동으로 AD 그룹 → S3 경로 직접 매핑 (IAM Policy 작성 불필요):
```
Grant: AD Group "CISO-Office"      → s3://assets/topsecret/    READ
Grant: AD Group "AI-Security-Team" → s3://assets/confidential/ READ
Grant: AD Group "All-Employees"    → s3://assets/internal/     READ
```

### 1-4. 등급별 KMS 키 분리

| 등급 | 암호화 | KMS 키 |
|------|--------|--------|
| 극비 | SSE-KMS | CISO 전용 CMK (MFA 필수 복호화) |
| 대외비 | SSE-KMS | 부서별 CMK |
| 사내한정 | SSE-S3 또는 SSE-KMS (AWS 관리 키) | — |
| 공개 | SSE-S3 (선택) | — |

### 1-5. S3 Object Lock — 극비 자산 변조 방지

- **Governance Mode**: 특정 IAM 권한 보유자만 삭제/수정 가능
- **Compliance Mode**: 보존 기간 내 누구도 삭제 불가 (규제 대응)

### 1-6. Amazon Macie — 자동 민감 데이터 탐지 및 등급 검증

**Macie의 동작 방식**:
- Macie는 S3 버킷 내 객체를 **샘플링 기반**으로 스캔합니다
- 자동 민감 데이터 검색(Automated Sensitive Data Discovery)은 매일 버킷당 **객체 샘플을 선택**하여 분석
- 샘플링 깊이(Sampling Depth)를 조정하여 스캔 범위를 제어할 수 있습니다 (기본 1%)
- 전수 조사가 필요한 경우 **민감 데이터 검색 작업(Discovery Job)**을 별도로 생성하여 특정 버킷/접두사의 전체 객체를 스캔

**활용 시나리오**:

| 시나리오 | Macie 기능 | 설명 |
|----------|-----------|------|
| 미분류 자산 탐지 | 자동 검색 (샘플링) | `Classification` 태그 없는 객체 중 민감 데이터 포함 여부 일일 샘플 스캔 |
| 등급 불일치 탐지 | 자동 검색 + 커스텀 식별자 | 태그가 `Public`인데 PII 포함 → 등급 불일치 알림 |
| 극비 버킷 전수 조사 | Discovery Job (전체 스캔) | 극비/대외비 버킷은 정기적으로 전체 객체 스캔 Job 실행 |
| 신규 업로드 검증 | EventBridge + Lambda | S3 PutObject 이벤트 → Lambda에서 Macie Job 트리거 → 등급 자동 제안 |

**커스텀 데이터 식별자 예시**:
```
이름: KoreanClassificationKeyword
정규식: (극비|대외비|기밀|Confidential|Top\s?Secret|RESTRICTED)
```

**Macie 결과 활용 파이프라인**:
```
Macie Finding → EventBridge → Lambda
  ├── 민감 데이터 발견 + 태그 미부여 → 자동 태그 부착 (Classification:Confidential)
  ├── 등급 불일치 (Public 태그 + PII 발견) → SNS 알림 → 정책 담당자 검토
  └── 극비 수준 데이터 발견 → SNS 긴급 알림 + S3 객체 접근 임시 차단
```

**비용 고려사항**:
- 자동 검색(샘플링): 버킷당 월 고정 비용 (스캔 대상 버킷 수 기준)
- Discovery Job(전수 스캔): 스캔한 데이터 GB당 과금
- 극비/대외비 버킷만 전수 스캔, 나머지는 샘플링으로 비용 최적화 권장

### 1-7. SCP — 태그 미부여 업로드 차단

```json
{
  "Effect": "Deny",
  "Action": "s3:PutObject",
  "Resource": "*",
  "Condition": {
    "Null": { "s3:RequestObjectTag/Classification": "true" }
  }
}
```
→ `Classification` 태그 없이 S3 업로드 원천 차단

---

## 2. On-Prem Git 코드 자산 — AWS 연계 등급 관리

> 코드 자산의 접근 제어는 On-Prem Git 자체 기능(그룹/프로젝트 권한, Protected Branch 등)으로 수행합니다.
> AWS 측에서는 **코드가 AWS로 유입되는 경로**(빌드 아티팩트, 컨테이너 이미지, 배포 설정)에 대한 등급 관리를 담당합니다.

### 2-1. 빌드 아티팩트 등급 관리

On-Prem Git → 빌드 → AWS 저장 시 등급 태그 부여:

| 아티팩트 유형 | AWS 저장소 | 등급 적용 |
|--------------|-----------|----------|
| 컨테이너 이미지 (LLM Gateway) | ECR | ECR Repository Policy + 이미지 태그 |
| IaC 템플릿 | S3 | 객체 태그 `Classification` + KMS |
| 빌드 로그 | S3 / CloudWatch Logs | 등급별 S3 버킷 분리 |
| 배포 설정 (API Key 등) | Secrets Manager | 시크릿별 KMS 키 + IAM 접근 제어 |

### 2-2. ECR — 컨테이너 이미지 등급 관리

**ECR Repository Policy — 등급별 이미지 Pull 제한**:
```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::123456789012:role/ECS-LLM-Gateway-Role"},
    "Action": ["ecr:GetDownloadUrlForLayer", "ecr:BatchGetImage"],
    "Condition": {
      "StringEquals": { "aws:PrincipalTag/ClearanceLevel": "Confidential" }
    }
  }]
}
```

**ECR 이미지 스캔**:
- Push 시 자동 취약점 스캔 (CVE 기반)
- 극비/대외비 이미지: Critical/High 취약점 발견 시 배포 차단 정책
- EventBridge → SNS 알림

### 2-3. Secrets Manager — On-Prem Git 자격 증명 관리

- Git 접근 토큰/SSH 키를 Secrets Manager에 저장
- 등급별 별도 시크릿 + KMS 키로 암호화
- CodeBuild 등 AWS 빌드 서비스에서 On-Prem Git 접근 시 Secrets Manager에서 자격 증명 조회
- 자동 교체(Rotation) 설정 가능

---

## 3. 등급별 통합 아키텍처

### Level 1-2 (공개/사내한정)
```
사용자 → SSO → S3 (SSE-S3) → 문서
On-Prem Git → 빌드 → S3/ECR (기본 암호화) → 아티팩트
```
- 기본 IAM 역할 기반 접근 제어
- Macie 샘플링 스캔 (자동 검색)

### Level 3 (대외비)
```
사용자 → SSO + MFA → VPN → S3 Access Point (VPC 전용) → 문서
On-Prem Git → 빌드 → ECR (Repository Policy) + S3 (SSE-KMS) → 아티팩트
```
- ABAC 태그 기반 접근 제어 + 팀 전용 KMS CMK
- Macie 정기 Discovery Job (전수 스캔)
- CloudTrail 데이터 이벤트 활성화

### Level 4 (극비)
```
사용자 → SSO + MFA → VPN → S3 VPC Endpoint → Access Point (극비) → 문서
On-Prem Git → 빌드 → ECR (극비 리포지토리) + S3 (CISO CMK + Object Lock) → 아티팩트
```
- IAM + ABAC + VPC Endpoint + KMS(MFA 필수) 다중 제어
- S3 Object Lock (변조 방지)
- Macie 전수 스캔 + 커스텀 식별자
- CloudTrail 전체 + S3 데이터 이벤트 + VPC Flow Logs
- 모든 접근 실시간 알림 (EventBridge → SNS)

---

## 4. 역할별 Task 요약

### 정책 담당자
- 보안 등급 체계 수립 (등급 정의, 태그 값 표준화)
- 등급별 접근 가능 인원/역할/AD 그룹 매핑
- 등급별 암호화 수준 결정 (KMS 키 분리 정책)
- 등급별 로그 보존 기간 결정
- Macie 스캔 방식 결정 (샘플링 vs 전수 스캔, 대상 버킷)
- 등급 미부여 자산 대응 정책 (업로드 차단 vs 기본 등급 자동 부여)

### 구축·운영 담당자
- S3 객체/버킷에 `Classification` 태그 체계 적용
- IAM ABAC Policy 설계 (ClearanceLevel ↔ Classification 매칭)
- S3 Access Point / Access Grants 설정
- KMS 키 등급별 생성 및 키 정책 설정
- Macie 활성화 + 커스텀 데이터 식별자 + Discovery Job 스케줄
- ECR Repository Policy 및 이미지 스캔 설정
- Secrets Manager에 Git 자격 증명 저장
- SCP로 태그 미부여 업로드 차단
- Config Rule (`required-tags`) 설정
- EventBridge 알림 파이프라인 구성

### 관제 담당자
- 극비/대외비 자산 접근 이력 모니터링 (Splunk)
- Macie 탐지 결과 모니터링 (등급 불일치, 미분류 민감 데이터)
- ECR 이미지 스캔 취약점 알림 모니터링
- Config Rule Non-compliant (태그 미부여) 알림 모니터링

---

> **참고**: IAM ABAC 및 Bedrock 권한 세분화는 [ai-security-granularity.md](./ai-security-granularity.md)를 참조하세요.
