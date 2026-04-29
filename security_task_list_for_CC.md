# AWS 위의 AI 보안 아키텍처 — 서비스별 보안 Task 정리

## 아키텍처 개요

```
[On-Prem 사내망]
  Claude Code (개발자 워크스테이션)
       │
  Client VPN 또는 H/W VPN
       │
[AWS Cloud]
  AWS S2S VPN / Client VPN
       │
  AWS VGW (Virtual Gateway)
       │
  AWS API Gateway
       │
  AWS ECS (LLM Gateway)
       │
  Amazon Bedrock
       │
  Claude Inference 호출
```

## 역할자 정의

| 역할 | 설명 |
|------|------|
| **AWS (솔루션 제공자)** | 보안 관련 기능 제공, 모범사례 교육 및 설계 지원 |
| **정책 담당자** | 사내 보안 정책 수립 (인증/권한관리/자산등급 등) |
| **구축·운영 담당자** | 정책을 기술적으로 구현, 솔루션 구성 및 설정 반영, 프로세스 관리 |
| **관제 담당자** | 이상 탐지 및 대응 (Splunk 활용, S3 로그 적재 기반) |

---

## 1. VPN (Client VPN / S2S VPN)

### 1-1. 인증 (Authentication)

**정책 담당자**
- VPN 접속 허용 대상 및 인증 방식 정책 수립 (인증서 기반 / AD 연동 / SAML 등)
- MFA 적용 여부 및 정책 결정
- VPN 접속 허용 시간대 및 조건 정의

**구축·운영 담당자**
- AWS Client VPN 엔드포인트 생성 및 인증서(ACM) 발급·관리
- Active Directory 또는 SAML IdP 연동 구성
- MFA 설정 적용 (Client VPN의 경우 AD + MFA 또는 SAML 기반)
- S2S VPN의 경우 IKE/IPSec 인증 파라미터(Pre-Shared Key 또는 인증서) 설정
- VPN 접속 Authorization Rule 구성 (대상 네트워크 CIDR 제한)

**관제 담당자**
- VPN 연결/해제 이벤트 모니터링 (CloudWatch Logs → S3 → Splunk)
- 비인가 접속 시도 탐지 알림 설정

### 1-2. 권한관리 (Authorization)

**정책 담당자**
- VPN 접속 후 접근 가능한 네트워크 대역(CIDR) 정책 수립
- 사용자/그룹별 접근 가능 리소스 범위 정의

**구축·운영 담당자**
- Client VPN Authorization Rule 설정 (그룹별 대상 네트워크 제한)
- Security Group 연결을 통한 VPN 엔드포인트 트래픽 제어
- Route Table 설정으로 VPN 트래픽 경로 제한
- S2S VPN의 경우 VGW Route Propagation 및 라우팅 정책 설정

**관제 담당자**
- 허용 범위 외 네트워크 접근 시도 탐지
- VPN 트래픽 흐름 로그(VPC Flow Logs) 기반 이상 패턴 모니터링

### 1-3. 로깅 (Logging)

**정책 담당자**
- VPN 관련 로그 보존 기간 및 저장 정책 수립
- 로그에 포함되어야 할 필수 항목 정의 (접속자, 시간, 소스IP, 대상 등)

**구축·운영 담당자**
- Client VPN 연결 로그 활성화 (CloudWatch Logs 그룹 지정)
- VPC Flow Logs 활성화 (VPN 관련 ENI 또는 서브넷 대상)
- 로그를 S3로 내보내기 설정 (Splunk 연동용)
- 로그 보존 주기 설정 (CloudWatch 보존 정책 + S3 Lifecycle)

**관제 담당자**
- Splunk에서 VPN 로그 대시보드 구성
- 비정상 접속 패턴(다수 실패, 비정상 시간대 접속 등) 알림 룰 설정

---

## 2. AWS VGW (Virtual Private Gateway)

### 2-1. 인증 (Authentication)

**정책 담당자**
- VGW 연결 대상 VPN 터널 인증 방식 정책 (PSK vs 인증서)
- 터널 갱신 주기 정책

**구축·운영 담당자**
- VGW 생성 및 VPC 연결(Attach)
- VPN Connection 생성 시 인증 파라미터 설정 (PSK 또는 Private Certificate Authority 인증서)
- 터널 옵션 설정 (IKE 버전, DH 그룹, 암호화 알고리즘 등)

**관제 담당자**
- VPN 터널 상태(UP/DOWN) 모니터링 (CloudWatch 메트릭: `TunnelState`)
- 터널 다운 이벤트 알림 설정

### 2-2. 권한관리 (Authorization)

**정책 담당자**
- VGW를 통해 전파되는 라우팅 범위 정책
- On-prem에서 접근 가능한 AWS 리소스 범위 정의

**구축·운영 담당자**
- Route Table에 VGW Route Propagation 활성화/비활성화 설정
- 필요 시 Static Route 추가로 라우팅 범위 제한
- Network ACL 및 Security Group으로 VGW 경유 트래픽 제어

**관제 담당자**
- VGW 경유 트래픽 볼륨 모니터링 (CloudWatch: `TunnelDataIn`, `TunnelDataOut`)
- 비정상 트래픽 급증 탐지

### 2-3. 로깅 (Logging)

**정책 담당자**
- VGW 관련 네트워크 로그 보존 정책

**구축·운영 담당자**
- VPC Flow Logs 활성화 (VGW 연결 서브넷 대상)
- CloudTrail에서 VGW 관련 API 호출 로깅 확인 (CreateVpnConnection 등)
- 로그 S3 적재 설정

**관제 담당자**
- 터널 상태 변경 이력 대시보드 구성
- 트래픽 이상 패턴 알림 설정

---

## 3. AWS API Gateway

### 3-1. 인증 (Authentication)

**정책 담당자**
- API 호출 인증 방식 정책 수립 (IAM, Cognito, Lambda Authorizer, API Key 등)
- API 호출 주체(사용자/서비스) 식별 정책
- mTLS 적용 여부 결정

**구축·운영 담당자**
- API Gateway 인증 메커니즘 구성:
  - IAM Authorization: IAM 정책 기반 API 접근 제어
  - Lambda Authorizer: 커스텀 토큰 검증 로직 구현
  - Cognito User Pool Authorizer: 사용자 인증 연동
  - API Key + Usage Plan: API 키 발급 및 사용량 제한
- mTLS 설정 (Custom Domain + Truststore 구성)
- Resource Policy 설정 (VPC Endpoint 또는 소스 IP 기반 접근 제한)

**관제 담당자**
- 인증 실패(401/403) 응답 비율 모니터링
- 비인가 API 호출 시도 탐지

### 3-2. 권한관리 (Authorization)

**정책 담당자**
- API 리소스/메서드별 접근 권한 정책 수립
- Rate Limiting / Throttling 정책 수립
- API 호출 가능 소스(IP/VPC) 제한 정책

**구축·운영 담당자**
- API Gateway Resource Policy 설정 (소스 VPC/IP 제한)
- Usage Plan 및 Throttling 설정 (Rate Limit, Burst Limit)
- IAM Policy에서 `execute-api:Invoke` 권한 세분화
- WAF 연동 (API Gateway 앞단에 AWS WAF 적용)
- Private API 구성 (VPC Endpoint 경유만 허용)

**관제 담당자**
- API 호출량 및 에러율 모니터링 (CloudWatch: `Count`, `4XXError`, `5XXError`)
- Throttling 발생 빈도 모니터링
- WAF 차단 이벤트 모니터링

### 3-3. 로깅 (Logging)

**정책 담당자**
- API 호출 로그 보존 정책 (요청/응답 본문 포함 여부 결정)
- 민감 데이터 마스킹 정책

**구축·운영 담당자**
- API Gateway 실행 로그(Execution Logs) 활성화 (CloudWatch Logs)
- API Gateway 액세스 로그(Access Logs) 활성화 (커스텀 포맷 설정)
- CloudTrail에서 API Gateway 관리 API 호출 로깅 확인
- 로그 S3 내보내기 설정 (Splunk 연동)
- 요청/응답 본문 로깅 시 민감 데이터 마스킹 처리

**관제 담당자**
- Splunk에서 API 호출 대시보드 구성 (호출량, 지연시간, 에러율)
- 비정상 호출 패턴 알림 (급격한 호출 증가, 특정 IP 집중 호출 등)

---

## 4. AWS ECS (LLM Gateway)

### 4-1. 인증 (Authentication)

**정책 담당자**
- LLM Gateway 서비스 접근 인증 정책 (API Gateway에서 전달받는 인증 토큰 검증 방식)
- 컨테이너 이미지 신뢰성 검증 정책 (이미지 서명/스캔)
- ECS Task가 사용하는 서비스 계정(Task Role) 인증 정책

**구축·운영 담당자**
- ECS Task Definition에서 Task Role 및 Execution Role 설정
- ECR 이미지 스캔 활성화 (취약점 스캔)
- ECR 이미지 서명 검증 구성 (선택적)
- ECS Service 배포 시 Private Subnet 배치 (인터넷 직접 노출 방지)
- Application Load Balancer(ALB) 또는 API Gateway와의 연동 시 인증 전달 구성

**관제 담당자**
- ECR 이미지 스캔 결과 모니터링 (취약점 발견 시 알림)
- ECS Task 시작/중지 이벤트 모니터링

### 4-2. 권한관리 (Authorization)

**정책 담당자**
- ECS Task Role의 최소 권한 원칙 정책
- LLM Gateway가 호출 가능한 Bedrock 모델 범위 정책
- 네트워크 수준 접근 제어 정책 (Security Group, NACL)

**구축·운영 담당자**
- ECS Task Role IAM Policy 설정:
  - Bedrock `InvokeModel` 권한 (특정 모델 ARN으로 제한)
  - 필요한 최소한의 AWS 서비스 접근 권한만 부여
- ECS Execution Role 설정 (ECR Pull, CloudWatch Logs 등)
- Security Group 설정 (인바운드: API Gateway/ALB만 허용, 아웃바운드: Bedrock 엔드포인트만 허용)
- VPC Endpoint 구성 (Bedrock, ECR, CloudWatch Logs 등 — 인터넷 경유 없이 접근)
- Secrets Manager 또는 SSM Parameter Store를 통한 민감 설정값 관리

**관제 담당자**
- ECS Task 리소스 사용량 모니터링 (CPU, Memory)
- Bedrock API 호출 실패율 모니터링
- Security Group 규칙 변경 이벤트 탐지

### 4-3. 로깅 (Logging)

**정책 담당자**
- LLM Gateway 애플리케이션 로그 보존 정책
- LLM 요청/응답 로그 보존 여부 및 민감 데이터 처리 정책
- 컨테이너 로그 보존 기간 정책

**구축·운영 담당자**
- ECS Task Definition에서 `awslogs` 로그 드라이버 설정 (CloudWatch Logs)
- 애플리케이션 레벨 로깅 구성 (요청 ID, 사용자 식별자, 모델 호출 정보 등)
- LLM 입출력 로깅 시 PII/민감 데이터 마스킹 처리
- CloudWatch Logs → S3 내보내기 설정 (Splunk 연동)
- 로그 보존 주기 설정

**관제 담당자**
- Splunk에서 LLM Gateway 대시보드 구성 (호출량, 지연시간, 에러율)
- 애플리케이션 에러 로그 알림 설정
- 비정상 LLM 호출 패턴 탐지 (대량 호출, 비정상 프롬프트 등)

---

## 5. Amazon Bedrock (Claude Inference)

### 5-1. 인증 (Authentication)

**정책 담당자**
- Bedrock API 호출 주체 인증 정책 (ECS Task Role만 허용)
- Bedrock 모델 접근 활성화 정책 (사용 가능 모델 목록 관리)

**구축·운영 담당자**
- Bedrock Model Access 활성화 (콘솔에서 사용할 모델 승인)
- ECS Task Role에 Bedrock 관련 IAM 정책 연결
- VPC Endpoint for Bedrock 구성 (Private 네트워크 경유 호출)
- Bedrock VPC Endpoint Policy 설정 (특정 IAM Principal만 허용)

**관제 담당자**
- Bedrock API 호출 인증 실패 이벤트 모니터링 (CloudTrail)
- 비인가 모델 접근 시도 탐지

### 5-2. 권한관리 (Authorization)

**정책 담당자**
- 사용 가능 Bedrock 모델 목록 정책 (예: Claude 3.5 Sonnet만 허용)
- Bedrock Guardrails 정책 수립 (콘텐츠 필터링, 토픽 제한, PII 감지 등)
- 모델 호출 한도(Quota) 정책

**구축·운영 담당자**
- IAM Policy에서 Bedrock 권한 세분화:
  - `bedrock:InvokeModel` — 특정 모델 ARN으로 Condition 제한
  - `bedrock:InvokeModelWithResponseStream` — 스트리밍 호출 허용 여부
- Bedrock Guardrails 구성:
  - Content Filter (유해 콘텐츠 차단)
  - Denied Topics (금지 주제 설정)
  - PII Detection (개인정보 감지 및 마스킹)
  - Word Filter (금지어 설정)
- Service Quotas 설정 확인 및 필요 시 한도 증가 요청
- Bedrock VPC Endpoint Policy로 접근 가능 Principal/Action 제한

**관제 담당자**
- Bedrock 모델 호출량 모니터링 (CloudWatch: `Invocations`, `InvocationLatency`)
- Guardrails 차단 이벤트 모니터링
- 토큰 사용량 모니터링 (비용 관점)
- Throttling 발생 모니터링

### 5-3. 로깅 (Logging)

**정책 담당자**
- Bedrock 모델 호출 로그 보존 정책
- LLM 입출력(프롬프트/응답) 로깅 여부 및 민감 데이터 처리 정책
- Bedrock Guardrails 차단 로그 보존 정책

**구축·운영 담당자**
- Bedrock Model Invocation Logging 활성화:
  - 대상: S3 버킷 및/또는 CloudWatch Logs
  - 입력/출력 데이터 로깅 여부 설정
  - 이미지 데이터 로깅 여부 설정
- S3 버킷 설정 (로그 저장용):
  - 버킷 정책 설정 (Bedrock 서비스 Principal 허용)
  - 서버 사이드 암호화(SSE-S3 또는 SSE-KMS) 설정
  - S3 Lifecycle Policy 설정 (로그 보존 주기)
- CloudTrail에서 Bedrock 관리 API 호출 로깅 확인

**관제 담당자**
- Splunk에서 Bedrock 호출 로그 대시보드 구성
- Guardrails 차단 이벤트 트렌드 분석
- 비정상 호출 패턴 알림 (대량 토큰 소비, 반복적 차단 등)

---

## 6. AWS IAM (Identity and Access Management)

### 6-1. 인증 (Authentication)

**정책 담당자**
- AWS 콘솔/API 접근 인증 정책 (SSO/Federation 방식)
- MFA 필수 적용 정책
- 패스워드 정책 (복잡도, 만료 주기 등)
- 서비스 계정(IAM Role) 관리 정책

**구축·운영 담당자**
- AWS IAM Identity Center(SSO) 구성 또는 외부 IdP Federation 설정
- MFA 필수 적용 (IAM Policy Condition: `aws:MultiFactorAuthPresent`)
- IAM 패스워드 정책 설정
- IAM User 최소화, IAM Role 기반 접근 구성
- 임시 자격 증명(STS AssumeRole) 활용 구성
- Access Key 관리 정책 적용 (주기적 교체, 미사용 키 비활성화)

**관제 담당자**
- 루트 계정 사용 이벤트 탐지 (CloudTrail)
- MFA 미적용 콘솔 로그인 탐지
- 비정상 로그인 시도 모니터링 (실패 횟수, 비정상 지역 등)

### 6-2. 권한관리 (Authorization)

**정책 담당자**
- 최소 권한 원칙(Least Privilege) 정책
- 역할 기반 접근 제어(RBAC) 정책 수립
- Permission Boundary 정책
- SCP(Service Control Policy) 정책 (Organizations 사용 시)

**구축·운영 담당자**
- IAM Policy 설계 및 적용:
  - 관리형 정책 vs 인라인 정책 구분 사용
  - Resource 및 Condition 기반 세분화
- Permission Boundary 설정 (위임된 관리자의 권한 상한 제한)
- IAM Access Analyzer 활성화 (외부 접근 가능 리소스 탐지)
- IAM Access Advisor 활용 (미사용 권한 식별 및 제거)
- SCP 설정 (Organizations 레벨에서 서비스/리전 제한)

**관제 담당자**
- IAM Access Analyzer 결과 모니터링 (외부 공유 리소스 탐지)
- 권한 변경 이벤트 모니터링 (Policy 생성/수정/삭제)
- 비정상 API 호출 패턴 탐지 (GuardDuty 연동)

### 6-3. 로깅 (Logging)

**정책 담당자**
- IAM 관련 이벤트 로그 보존 정책
- 감사(Audit) 로그 요구사항 정의

**구축·운영 담당자**
- CloudTrail에서 IAM 관련 이벤트 로깅 확인 (기본 활성화)
- IAM Credential Report 주기적 생성 설정
- Access Advisor 데이터 주기적 리뷰 프로세스 수립
- 로그 S3 적재 및 Splunk 연동

**관제 담당자**
- Splunk에서 IAM 이벤트 대시보드 구성
- 고위험 IAM 변경 알림 (루트 키 생성, 정책 변경, 역할 생성 등)
- Credential Report 기반 미사용 계정/키 리포트

---

## 7. AWS CloudTrail

### 7-1. 인증 (Authentication)

**정책 담당자**
- CloudTrail 관리 접근 권한 정책 (누가 Trail을 생성/수정/삭제할 수 있는지)
- CloudTrail 로그 접근 권한 정책

**구축·운영 담당자**
- CloudTrail Trail 생성 (전체 리전, 관리 이벤트 + 데이터 이벤트)
- CloudTrail 관리 권한을 특정 IAM Role로 제한
- S3 버킷 정책에서 CloudTrail 서비스 Principal만 쓰기 허용
- CloudTrail 로그 파일 무결성 검증(Log File Validation) 활성화

**관제 담당자**
- CloudTrail Trail 설정 변경 이벤트 모니터링 (Trail 비활성화/삭제 시도 탐지)

### 7-2. 권한관리 (Authorization)

**정책 담당자**
- CloudTrail 로그 열람 권한 정책 (보안팀만 접근 가능 등)
- CloudTrail 설정 변경 권한 정책

**구축·운영 담당자**
- S3 버킷 정책 설정 (로그 읽기 권한을 보안팀 IAM Role로 제한)
- KMS 키 정책 설정 (CloudTrail 로그 암호화 키 접근 제어)
- IAM Policy로 CloudTrail 관리 API 접근 제한
- Organization Trail 설정 (멀티 계정 환경 시)

**관제 담당자**
- CloudTrail 로그 무결성 검증 결과 모니터링
- Trail 설정 변경 알림

### 7-3. 로깅 (Logging)

**정책 담당자**
- CloudTrail 로그 보존 기간 정책
- 로그 암호화 정책 (KMS 키 사용)
- 데이터 이벤트 로깅 범위 정책 (S3, Lambda, Bedrock 등)

**구축·운영 담당자**
- CloudTrail Trail 설정:
  - 관리 이벤트(Management Events) 로깅 활성화
  - 데이터 이벤트(Data Events) 로깅 활성화 (S3, Bedrock 등 대상 지정)
  - 멀티 리전 Trail 활성화
- S3 버킷 설정:
  - SSE-KMS 암호화 적용
  - S3 Object Lock 설정 (로그 변조 방지, 선택적)
  - S3 Lifecycle Policy (로그 보존 주기)
  - S3 버킷 버저닝 활성화
- CloudWatch Logs 연동 (실시간 알림용)
- CloudTrail Lake 활용 (쿼리 기반 분석, 선택적)

**관제 담당자**
- Splunk에서 CloudTrail 로그 대시보드 구성
- 고위험 API 호출 알림 설정 (루트 로그인, 보안 그룹 변경, IAM 변경 등)
- CloudTrail Insights 활성화 (비정상 API 호출 볼륨 자동 탐지)

---

## 8. AWS Config

### 8-1. 인증 (Authentication)

**정책 담당자**
- AWS Config 관리 접근 권한 정책
- Config Rule 생성/수정 권한 정책

**구축·운영 담당자**
- AWS Config 활성화 (전체 리전)
- Config Service-Linked Role 확인
- Config 관리 권한을 특정 IAM Role로 제한

**관제 담당자**
- Config 설정 변경 이벤트 모니터링 (Config Recorder 중지 시도 탐지)

### 8-2. 권한관리 (Authorization)

**정책 담당자**
- Config Rule 정의 정책 (어떤 규정 준수 규칙을 적용할 것인지)
- 비준수(Non-compliant) 리소스 대응 정책

**구축·운영 담당자**
- AWS Config Rules 설정 (이 아키텍처 관련 주요 규칙):
  - `vpc-flow-logs-enabled` — VPC Flow Logs 활성화 확인
  - `cloud-trail-enabled` — CloudTrail 활성화 확인
  - `iam-root-access-key-check` — 루트 Access Key 미사용 확인
  - `mfa-enabled-for-iam-console-access` — MFA 적용 확인
  - `iam-policy-no-statements-with-admin-access` — 과도한 권한 탐지
  - `api-gw-execution-logging-enabled` — API Gateway 로깅 확인
  - `ecs-task-definition-nonroot-user` — ECS 컨테이너 비루트 실행 확인
  - `encrypted-volumes` — EBS 암호화 확인
  - `s3-bucket-server-side-encryption-enabled` — S3 암호화 확인
  - `restricted-ssh` — SSH 접근 제한 확인
- Config Conformance Pack 활용 (보안 프레임워크 기반 규칙 묶음)
- 자동 수정(Remediation) 설정 (SSM Automation 연동, 선택적)

**관제 담당자**
- Config 규정 준수 대시보드 모니터링
- Non-compliant 리소스 발생 시 알림 설정
- 규정 준수 트렌드 리포트

### 8-3. 로깅 (Logging)

**정책 담당자**
- Config 기록 보존 정책
- 리소스 구성 변경 이력 보존 기간 정책

**구축·운영 담당자**
- Config Recorder 설정 (전체 리소스 유형 기록)
- Config 스냅샷 및 변경 이력 S3 저장 설정
- Config 알림 SNS Topic 설정
- S3 버킷 설정 (암호화, Lifecycle, 접근 제어)

**관제 담당자**
- Splunk에서 Config 규정 준수 대시보드 구성
- 리소스 구성 변경 이력 추적
- Non-compliant 리소스 트렌드 분석

---

## 요약: 관제 담당자를 위한 S3 → Splunk 로그 적재 대상

| 로그 소스 | S3 적재 경로 | 비고 |
|-----------|-------------|------|
| VPN 연결 로그 | CloudWatch Logs → S3 | Client VPN 연결/해제 이벤트 |
| VPC Flow Logs | 직접 S3 또는 CloudWatch → S3 | 네트워크 트래픽 로그 |
| API Gateway 액세스 로그 | CloudWatch Logs → S3 | API 호출 기록 |
| API Gateway 실행 로그 | CloudWatch Logs → S3 | 상세 실행 로그 |
| ECS 컨테이너 로그 | CloudWatch Logs → S3 | LLM Gateway 애플리케이션 로그 |
| Bedrock 모델 호출 로그 | 직접 S3 | 프롬프트/응답 로그 |
| CloudTrail 로그 | 직접 S3 | 전체 AWS API 호출 기록 |
| AWS Config 기록 | 직접 S3 | 리소스 구성 변경 이력 |
| WAF 로그 | 직접 S3 또는 Kinesis → S3 | API Gateway WAF 차단 로그 |
| IAM Credential Report | 수동/자동 생성 후 S3 | 계정/키 상태 리포트 |
