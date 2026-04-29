# 코드 및 문서 자산 보안 등급 관리 — AWS 서비스 활용 제안

## 개요

정책 담당자가 수립한 자산 보안 등급(예: 공개 / 사내 한정 / 대외비 / 극비)을 AWS 서비스를 활용하여 기술적으로 구현하는 방안을 정리합니다.

이 문서는 두 가지 자산 유형을 다룹니다:
1. **코드 자산** — Git 리포지토리 내 소스코드, IaC 템플릿, 설정 파일 등
2. **문서 자산** — S3에 저장되는 설계 문서, 회의록, LLM 프롬프트/응답 로그, 보안 감사 자료 등

---

## 자산 보안 등급 체계 (예시)

| 등급 | 레이블 | 설명 | 예시 |
|------|--------|------|------|
| Level 1 | **공개 (Public)** | 외부 공개 가능 | 오픈소스 코드, 공개 문서 |
| Level 2 | **사내 한정 (Internal)** | 사내 직원만 접근 | 일반 개발 코드, 내부 위키 |
| Level 3 | **대외비 (Confidential)** | 특정 프로젝트/팀만 접근 | AI 모델 파인튜닝 코드, 아키텍처 설계서, LLM 프롬프트 로그 |
| Level 4 | **극비 (Top Secret)** | 최소 인원만 접근, 감사 추적 필수 | 보안 키, 인증서, 핵심 알고리즘, 고객 데이터 처리 로직 |

---

## 1. S3 기반 문서 자산 등급 관리

### 1-1. 태그 기반 등급 분류 + ABAC

**구현 방식**: S3 객체 또는 버킷에 보안 등급 태그를 부착하고, IAM ABAC로 접근 제어

**태그 설계**:
```
Key: SecurityClassification
Value: Public | Internal | Confidential | TopSecret
```

**IAM Policy 예시 — 대외비 이상만 특정 팀 접근 허용**:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::doc-assets/*",
    "Condition": {
      "StringEquals": {
        "s3:ExistingObjectTag/SecurityClassification": ["Confidential", "TopSecret"],
        "aws:PrincipalTag/Team": "security-team"
      }
    }
  }]
}
```

**정책 담당자 Task**:
- 보안 등급 체계 및 태그 값 표준 정의
- 등급별 접근 가능 역할/팀 매핑 정책 수립
- 등급 변경 승인 프로세스 정의

**구축·운영 담당자 Task**:
- S3 버킷 정책 및 IAM Policy에 태그 기반 Condition 적용
- 객체 업로드 시 태그 자동 부착 프로세스 구성 (Lambda 트리거 또는 업로드 정책)
- `s3:PutObjectTagging` 권한을 등급 관리자에게만 부여 (임의 등급 변경 방지)

### 1-2. S3 Access Points — 등급별 접근 경로 분리

**구현 방식**: 하나의 S3 버킷에 등급별 Access Point를 생성하여, 각 Access Point마다 다른 접근 정책 적용

```
doc-assets-bucket
  ├── ap-public      → 공개 등급 객체만 접근 가능 (prefix: public/)
  ├── ap-internal    → 사내 한정 이하 접근 가능 (prefix: internal/)
  ├── ap-confidential → 대외비 이하 접근 가능 (prefix: confidential/)
  └── ap-topsecret   → 극비 접근 가능 (prefix: topsecret/, VPC 경유 필수)
```

**극비 등급 Access Point — VPC 전용 + 태그 기반 제어**:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::123456789012:role/TopSecret-Reader"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:us-east-1:123456789012:accesspoint/ap-topsecret/object/topsecret/*",
    "Condition": {
      "StringEquals": {
        "s3:AccessPointNetworkOrigin": "VPC",
        "aws:PrincipalTag/ClearanceLevel": "TopSecret"
      }
    }
  }]
}
```

**장점**:
- 등급별 네트워크 경로 분리 (극비는 VPC 전용, 공개는 인터넷 허용)
- Access Point별 독립적인 접근 정책 관리
- 버킷 정책 복잡도 감소

### 1-3. S3 Access Grants — 사내 디렉토리 연동 등급 관리

**구현 방식**: IAM Identity Center(SSO)와 연동하여, 사내 디렉토리(AD/Okta 등)의 사용자/그룹을 S3 경로에 직접 매핑

```
S3 Access Grants Instance
  ├── Location: s3://doc-assets/confidential/
  │     └── Grant: AD Group "AI-Security-Team" → READ
  ├── Location: s3://doc-assets/topsecret/
  │     └── Grant: AD Group "CISO-Office" → READ
  └── Location: s3://doc-assets/internal/
        └── Grant: AD Group "All-Employees" → READ
```

**장점**:
- IAM Policy 작성 없이 사내 디렉토리 그룹 기반으로 S3 접근 제어
- 인사 이동 시 AD 그룹 변경만으로 자동 반영
- 임시 자격 증명(STS) 자동 발급 — Access Key 관리 불필요

**정책 담당자 Task**:
- 등급별 접근 가능 AD 그룹 매핑 정의
- Grant 생성/변경 승인 프로세스

### 1-4. Amazon Macie — 자동 등급 분류 및 민감 데이터 탐지

**구현 방식**: S3 버킷을 Macie로 스캔하여 민감 데이터를 자동 탐지하고, 등급 미부착 또는 등급 불일치 자산을 식별

**활용 시나리오**:
- **자동 탐지**: PII(개인정보), 금융 데이터, 인증 정보(API Key, 비밀번호) 등이 포함된 객체 자동 식별
- **커스텀 데이터 식별자**: 사내 보안 등급 키워드(예: "극비", "대외비", "Confidential") 패턴 정의
- **등급 검증**: 태그가 `Public`인데 PII가 포함된 객체 → 등급 불일치 알림
- **미분류 자산 탐지**: 보안 등급 태그가 없는 객체 식별

**Macie 커스텀 데이터 식별자 예시**:
```
이름: SecurityClassificationKeyword
정규식: (극비|대외비|Confidential|Top\s?Secret|기밀)
```

**정책 담당자 Task**:
- Macie 스캔 대상 버킷 범위 결정
- 커스텀 데이터 식별자 키워드 목록 정의
- 등급 불일치 발견 시 대응 프로세스 정의

**구축·운영 담당자 Task**:
- Macie 활성화 및 대상 S3 버킷 설정
- 커스텀 데이터 식별자 생성
- Macie Finding → EventBridge → SNS/Lambda 알림 파이프라인 구성
- 자동 태그 부착 Lambda 구성 (선택적)

### 1-5. S3 암호화 — 등급별 KMS 키 분리

**구현 방식**: 보안 등급별로 다른 KMS 키를 사용하여 암호화하고, KMS 키 정책으로 접근 제어

```
Level 1-2 (공개/사내 한정): SSE-S3 (AWS 관리형 키)
Level 3 (대외비):           SSE-KMS (팀 전용 CMK)
Level 4 (극비):             SSE-KMS (CISO 전용 CMK + 키 정책으로 접근 제한)
```

**극비 등급 KMS 키 정책 예시**:
```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::123456789012:role/CISO-Office-Role"},
    "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
    "Resource": "*",
    "Condition": {
      "Bool": {"aws:MultiFactorAuthPresent": "true"}
    }
  }]
}
```
→ 극비 문서 복호화 시 MFA 필수

---

## 2. Git 기반 코드 자산 등급 관리

### 2-1. AWS CodeCommit — 리포지토리/브랜치 레벨 접근 제어

> ⚠️ CodeCommit은 2024년 7월부터 신규 고객 가입이 중단되었습니다. 기존 사용 중인 고객은 계속 사용 가능하며, 신규 구축 시에는 2-2의 CodeConnections + 외부 Git 방식을 권장합니다.

**리포지토리 레벨 분리**:
```
repo-public        → Level 1 (공개)
repo-internal      → Level 2 (사내 한정)
repo-confidential  → Level 3 (대외비) — AI 모델 코드, LLM Gateway 코드
repo-topsecret     → Level 4 (극비) — 보안 키 관리, 핵심 알고리즘
```

**IAM Policy — 리포지토리 ARN 단위 접근 제어**:
```json
{
  "Effect": "Allow",
  "Action": ["codecommit:GitPull", "codecommit:GitPush"],
  "Resource": "arn:aws:codecommit:us-east-1:123456789012:repo-confidential",
  "Condition": {
    "StringEquals": {
      "aws:PrincipalTag/ClearanceLevel": "Confidential"
    }
  }
}
```

**브랜치 레벨 보호 — 극비 브랜치 Push/Merge 제한**:
```json
{
  "Effect": "Deny",
  "Action": [
    "codecommit:GitPush",
    "codecommit:MergeBranchesByFastForward",
    "codecommit:MergeBranchesBySquash",
    "codecommit:MergeBranchesByThreeWay",
    "codecommit:MergePullRequestByFastForward",
    "codecommit:MergePullRequestBySquash",
    "codecommit:MergePullRequestByThreeWay"
  ],
  "Resource": "arn:aws:codecommit:us-east-1:123456789012:repo-topsecret",
  "Condition": {
    "StringEqualsIfExists": {
      "codecommit:References": ["refs/heads/main", "refs/heads/release/*"]
    },
    "Null": {
      "codecommit:References": "false"
    }
  }
}
```
→ `main` 및 `release/*` 브랜치에 대한 직접 Push/Merge 차단 (PR 리뷰 강제)

### 2-2. CodeConnections + 외부 Git (GitHub/GitLab) — 연동 기반 등급 관리

**구현 방식**: AWS CodeConnections(구 CodeStar Connections)를 통해 GitHub/GitLab과 연동하고, 양쪽의 접근 제어를 조합

**등급별 구성 전략**:

| 등급 | Git 호스팅 | 접근 제어 |
|------|-----------|----------|
| Level 1-2 | GitHub/GitLab (SaaS) | GitHub/GitLab 자체 팀/역할 권한 |
| Level 3 | GitHub Enterprise / GitLab Self-Managed (VPC 내) | GitHub/GitLab 권한 + AWS VPC 네트워크 격리 |
| Level 4 | CodeCommit 또는 GitLab Self-Managed (Private Subnet) | IAM 기반 접근 제어 + VPC Endpoint + KMS 암호화 |

**CodeConnections IAM 제어**:
```json
{
  "Effect": "Allow",
  "Action": "codeconnections:UseConnection",
  "Resource": "arn:aws:codeconnections:us-east-1:123456789012:connection/{connection-id}",
  "Condition": {
    "StringEquals": {
      "codeconnections:FullRepositoryId": "org/repo-confidential"
    }
  }
}
```
→ 특정 리포지토리에 대한 Connection 사용만 허용

### 2-3. CodeCommit + CloudTrail — 코드 접근 감사 추적

**CloudTrail 데이터 이벤트로 Git 작업 추적**:
- `GitPull` — 누가 언제 어떤 리포지토리의 코드를 가져갔는지
- `GitPush` — 누가 언제 어떤 리포지토리에 코드를 올렸는지
- `MergePullRequestBy*` — PR 머지 이력

**기록되는 정보**:
- `userIdentity` — Git 작업 수행자 (IAM User/Role)
- `sourceIPAddress` — 작업 소스 IP
- `requestParameters.repositoryName` — 대상 리포지토리
- `requestParameters.branchName` — 대상 브랜치

**극비 리포지토리 감사 정책**:
- CloudTrail 데이터 이벤트에서 `repo-topsecret` 리포지토리 대상 모든 Git 작업 로깅
- 비인가 접근 시도(AccessDenied) 실시간 알림

---

## 3. 등급별 통합 보안 아키텍처 제안

### Level 1-2 (공개/사내 한정)

```
사용자 → SSO 인증 → GitHub/GitLab (SaaS) → 코드
사용자 → SSO 인증 → S3 (SSE-S3 암호화) → 문서
```
- 기본 IAM 역할 기반 접근 제어
- S3 태그: `SecurityClassification: Internal`
- 로깅: CloudTrail 관리 이벤트 (기본)

### Level 3 (대외비)

```
사용자 → SSO + MFA → VPN → VPC 내 Git (Self-Managed) → 코드
사용자 → SSO + MFA → S3 Access Point (VPC 전용) → 문서
```
- ABAC 태그 기반 접근 제어
- S3 태그: `SecurityClassification: Confidential`
- SSE-KMS (팀 전용 CMK)
- Macie 자동 스캔 활성화
- 로깅: CloudTrail 관리 + 데이터 이벤트

### Level 4 (극비)

```
사용자 → SSO + MFA → VPN → Private Subnet 내 Git → 코드
사용자 → SSO + MFA → VPN → S3 VPC Endpoint → S3 Access Point (극비) → 문서
```
- IAM + ABAC + VPC Endpoint + KMS 키 정책 다중 제어
- S3 태그: `SecurityClassification: TopSecret`
- SSE-KMS (CISO 전용 CMK, MFA 필수 복호화)
- S3 Object Lock (변조 방지)
- Macie 자동 스캔 + 커스텀 식별자
- 로깅: CloudTrail 전체 + S3 데이터 이벤트 + VPC Flow Logs
- 모든 접근 실시간 알림 (EventBridge → SNS)

---

## 4. 정책 담당자 결정사항 체크리스트

| # | 결정사항 | 비고 |
|---|---------|------|
| 1 | 보안 등급 체계 정의 (몇 단계, 각 등급 명칭) | 예: 4단계 (공개/사내한정/대외비/극비) |
| 2 | 등급별 접근 가능 역할/팀 매핑 | AD 그룹 또는 IAM Role 기준 |
| 3 | 등급별 인증 강도 (MFA 필수 여부) | Level 3 이상 MFA 필수 권장 |
| 4 | 등급별 네트워크 접근 경로 (인터넷/VPN/VPC 전용) | Level 4는 VPC 전용 필수 |
| 5 | 등급별 암호화 수준 (SSE-S3 / SSE-KMS / KMS+MFA) | Level 4는 전용 CMK + MFA |
| 6 | 등급 변경 승인 프로세스 | 누가 승인하는지, 이력 관리 방법 |
| 7 | 미분류 자산 기본 등급 정책 | 태그 미부착 시 기본 Level 2 등 |
| 8 | Macie 스캔 범위 및 커스텀 키워드 | 등급 불일치 탐지용 |
| 9 | 극비 자산 접근 로그 보존 기간 | 감사 요구사항에 따라 결정 |
| 10 | 코드 리포지토리 등급 분류 기준 | 리포지토리 단위 vs 브랜치 단위 |

---

## 5. 구축·운영 담당자 구현 Task 요약

| # | Task | AWS 서비스 | 등급 |
|---|------|-----------|------|
| 1 | S3 객체 태그 표준 적용 (`SecurityClassification`) | S3, IAM | 전체 |
| 2 | IAM ABAC Policy 설계 (태그 기반 접근 제어) | IAM | 전체 |
| 3 | S3 Access Point 등급별 생성 | S3 | Level 3-4 |
| 4 | S3 Access Grants + Identity Center 연동 | S3, IAM Identity Center | Level 2-4 |
| 5 | KMS 키 등급별 생성 및 키 정책 설정 | KMS | Level 3-4 |
| 6 | Macie 활성화 및 커스텀 데이터 식별자 설정 | Macie | Level 3-4 |
| 7 | Git 리포지토리 등급별 분리 및 IAM 정책 적용 | CodeCommit / CodeConnections | 전체 |
| 8 | 브랜치 보호 정책 (Push/Merge 제한) | CodeCommit IAM Condition | Level 3-4 |
| 9 | CloudTrail 데이터 이벤트 활성화 (S3 + Git) | CloudTrail | Level 3-4 |
| 10 | S3 Object Lock 설정 (변조 방지) | S3 | Level 4 |
| 11 | EventBridge 알림 파이프라인 구성 | EventBridge, SNS, Lambda | Level 4 |
| 12 | VPC Endpoint 전용 접근 경로 구성 | VPC, S3 | Level 4 |

---

> **참고**: 이 문서는 [ai-security-architecture-tasks.md](./ai-security-architecture-tasks.md)의 IAM 섹션(6-2. 권한관리)과 연계됩니다.
> 등급별 IAM 정책 세분화 수준은 [ai-security-granularity.md](./ai-security-granularity.md)의 B-1. IAM × Bedrock Deep Dive를 참조하세요.
