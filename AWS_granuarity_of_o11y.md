# AWS 서비스별 보안 기능 세분화(Granularity) 수준

각 AWS 서비스가 **얼마나 세밀한 수준까지** 권한 제어, 로깅, 관제가 가능한지를 정리합니다.

---

## A. VPC Flow Logs — 네트워크 트래픽 로깅 세분화

**로깅 단위**: 플로우(Flow) 단위 (5-tuple: 소스IP, 목적지IP, 소스포트, 목적지포트, 프로토콜)

**기본 포맷 (v2) 필드**:
- `srcaddr` / `dstaddr` — 소스/목적지 IP 주소
- `srcport` / `dstport` — 소스/목적지 포트
- `protocol` — IANA 프로토콜 번호 (TCP=6, UDP=17 등)
- `packets` / `bytes` — 전송 패킷 수 및 바이트 수
- `action` — ACCEPT / REJECT (Security Group 또는 NACL에 의한 허용/거부)
- `log-status` — OK / NODATA / SKIPDATA

**커스텀 포맷으로 확장 가능한 필드 (v3~v10)**:
- `vpc-id`, `subnet-id`, `instance-id` — 리소스 식별
- `tcp-flags` — TCP 플래그 비트마스크 (FIN=1, SYN=2, RST=4, SYN-ACK=18). 단, ACK, PSH 등 일부 플래그는 미지원
- `pkt-srcaddr` / `pkt-dstaddr` — 패킷 레벨 원본 소스/목적지 IP (NAT 경유 시 원본 추적 가능)
- `pkt-src-aws-service` / `pkt-dst-aws-service` — 트래픽 소스/목적지가 AWS 서비스인 경우 서비스명 식별 (S3, DYNAMODB, EC2 등)
- `flow-direction` — ingress / egress
- `traffic-path` — 트래픽 경로 (인터넷 게이트웨이, VPC 피어링, VGW 등 경유 경로 식별)
- `ecs-cluster-arn`, `ecs-service-name`, `ecs-task-arn`, `ecs-container-id` — **ECS 컨테이너 레벨까지 트래픽 추적 가능** (v7)
- `reject-reason` — 거부 사유 (BPA=Block Public Access, EC=Encryption Controls) (v8)
- `encryption-status` — 암호화 상태 (0=미암호화, 1=Nitro 암호화, 2=애플리케이션 암호화, 3=양쪽 모두) (v10)

**세분화 수준 요약**:
- ✅ 패킷 헤더 수준 (IP, 포트, 프로토콜, TCP 플래그)
- ✅ 플로우 단위 ACCEPT/REJECT 판정
- ✅ NAT 경유 시 원본 IP 추적
- ✅ ECS 컨테이너/태스크 레벨 트래픽 식별
- ✅ AWS 서비스 간 트래픽 소스 식별
- ✅ 암호화 상태 확인
- ❌ 패킷 페이로드(본문) 캡처는 불가 — 헤더 메타데이터만 기록
- ❌ 일부 TCP 플래그(ACK, PSH) 미지원

**집계 간격**: 기본 10분, 최소 1분까지 설정 가능

---

## B. AWS IAM — 권한 관리 세분화

**권한 제어 모델**:

1. **리소스 레벨 권한 (Resource-Level Permissions)**
   - IAM Policy의 `Resource` 요소에서 특정 리소스 ARN을 지정하여 개별 리소스 단위로 권한 제어
   - 예: 특정 S3 버킷, 특정 DynamoDB 테이블, 특정 Bedrock 모델 ARN
   - ⚠️ **모든 서비스가 리소스 레벨 권한을 지원하지는 않음** — 서비스별로 지원 범위가 다름
   - 예: `bedrock:InvokeModel`은 모델 ARN 단위로 제한 가능, 일부 관리 API는 `*`만 지원

2. **태그 기반 접근 제어 (ABAC — Attribute-Based Access Control)**
   - `aws:ResourceTag`, `aws:RequestTag`, `aws:PrincipalTag` Condition Key 활용
   - 리소스에 부착된 태그 값을 기준으로 접근 허용/거부
   - 예: `"Condition": {"StringEquals": {"aws:ResourceTag/Environment": "Production"}}` → Production 태그가 붙은 리소스만 접근 허용
   - ✅ 리소스 레벨 권한이 제한적인 서비스에서도 태그 기반으로 세분화 가능
   - ✅ 새 리소스 추가 시 정책 수정 없이 태그만 부여하면 자동 적용 (확장성 우수)

3. **Condition Key 기반 세분화**
   - `aws:SourceIp` — 소스 IP 기반 제한
   - `aws:SourceVpc` / `aws:SourceVpce` — VPC 또는 VPC Endpoint 기반 제한
   - `aws:MultiFactorAuthPresent` — MFA 인증 여부
   - `aws:PrincipalOrgID` — Organizations 조직 ID 기반 제한
   - `aws:CalledVia` — 특정 서비스를 경유한 호출만 허용
   - `bedrock:InvokeModel` 시 `aws:ResourceTag` 또는 모델 ARN Condition으로 특정 모델만 허용

4. **Permission Boundary**
   - IAM 엔티티(User/Role)에 설정하는 권한 상한선
   - 위임된 관리자가 부여할 수 있는 최대 권한 범위를 제한

5. **SCP (Service Control Policy)**
   - AWS Organizations 레벨에서 계정 전체에 적용되는 권한 가드레일
   - 특정 서비스, 리전, 액션을 조직 수준에서 차단 가능

**세분화 수준 요약**:
- ✅ 개별 API 액션 단위 허용/거부
- ✅ 리소스 ARN 단위 제어 (서비스별 지원 범위 상이)
- ✅ 태그 기반 동적 접근 제어 (ABAC)
- ✅ IP, VPC, MFA, 시간 등 다양한 Condition 기반 제어
- ✅ 조직 수준 가드레일 (SCP)
- ❌ 일부 서비스/액션은 리소스 레벨 권한 미지원 (`*`만 가능)

### B-1. IAM × Bedrock Deep Dive — Bedrock 서비스에 대한 IAM 권한 세분화

Amazon Bedrock은 IAM 권한 제어가 매우 세분화되어 있어, 모델 단위, 리소스 유형 단위, 조건 기반으로 정밀한 접근 제어가 가능합니다.

#### 리소스 레벨 권한 지원 현황

Bedrock은 대부분의 주요 액션에서 **리소스 레벨 권한을 지원**합니다:

| IAM Action | 리소스 레벨 권한 | Resource ARN 형식 | 설명 |
|---|---|---|---|
| `bedrock:InvokeModel` | ✅ 지원 | `arn:aws:bedrock:{region}::foundation-model/{model-id}` | 특정 모델만 호출 허용 |
| `bedrock:InvokeModelWithResponseStream` | ✅ 지원 | `arn:aws:bedrock:{region}::foundation-model/{model-id}` | 스트리밍 호출 제한 |
| `bedrock:Converse` | ✅ 지원 | `arn:aws:bedrock:{region}::foundation-model/{model-id}` | Converse API 제한 |
| `bedrock:ApplyGuardrail` | ✅ 지원 | `arn:aws:bedrock:{region}:{account}:guardrail/{id}` | 특정 Guardrail만 적용 |
| `bedrock:InvokeAgent` | ✅ 지원 | `arn:aws:bedrock:{region}:{account}:agent-alias/{agent-id}/{alias-id}` | 특정 Agent만 호출 |
| `bedrock:Retrieve` (KB) | ✅ 지원 | `arn:aws:bedrock:{region}:{account}:knowledge-base/{kb-id}` | 특정 KB만 쿼리 |
| `bedrock:CreateAgent` | ❌ `*` 만 | — | 생성 시점에는 ARN 미존재 |
| `bedrock:ListFoundationModels` | ❌ `*` 만 | — | 목록 조회는 리소스 특정 불가 |

**핵심 포인트**: `InvokeModel`은 Foundation Model ARN 단위로 제한 가능하므로, "Claude 3.5 Sonnet만 호출 가능" 같은 정책을 IAM으로 직접 구현할 수 있습니다.

#### Bedrock 전용 Condition Key

Bedrock은 AWS 글로벌 Condition Key 외에 **서비스 전용 Condition Key**를 제공합니다:

| Condition Key | 설명 | 사용 예시 |
|---|---|---|
| `bedrock:ThirdPartyKnowledgeBaseCredentialsSecretArn` | 3rd party KB 연동 시 Secret ARN 제한 | 특정 Secrets Manager 시크릿만 허용 |
| `bedrock:BearerTokenType` | Bearer Token 유형 제한 | 특정 토큰 유형만 허용 |
| `bedrock:ServiceTier` | Agent 서비스 티어 제한 | 특정 서비스 티어만 허용/거부 |
| `aws:RequestTag/${TagKey}` | 리소스 생성 시 태그 강제 | Agent/KB 생성 시 필수 태그 요구 |
| `aws:ResourceTag/${TagKey}` | 기존 리소스 태그 기반 접근 제어 | 특정 팀 태그가 붙은 Agent만 접근 |

#### ABAC (태그 기반 접근 제어) 지원

Bedrock은 **ABAC를 지원**합니다. 태그를 부착할 수 있는 Bedrock 리소스:
- Agent, Agent Alias, Agent Action Group
- Knowledge Base
- Guardrail
- Custom Model, Model Copy Job
- Evaluation Job
- Provisioned Model Throughput
- Flow, Flow Alias, Prompt

**ABAC 활용 예시**: 팀별로 Agent를 분리 관리
```json
{
  "Effect": "Allow",
  "Action": ["bedrock:InvokeAgent"],
  "Resource": "arn:aws:bedrock:*:*:agent-alias/*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/Team": "${aws:PrincipalTag/Team}"
    }
  }
}
```
→ 호출자의 `Team` 태그와 Agent의 `Team` 태그가 일치할 때만 호출 허용

#### 실전 IAM Policy 예시

**예시 1: 특정 모델만 호출 허용 (Claude 3.5 Sonnet)**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "bedrock:InvokeModel",
      "bedrock:InvokeModelWithResponseStream"
    ],
    "Resource": [
      "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0",
      "arn:aws:bedrock:us-west-2::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0"
    ]
  }]
}
```

**예시 2: SCP로 조직 전체에서 특정 모델 차단**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyNonApprovedModels",
    "Effect": "Deny",
    "Action": [
      "bedrock:InvokeModel",
      "bedrock:InvokeModelWithResponseStream"
    ],
    "Resource": "arn:aws:bedrock:*::foundation-model/*",
    "Condition": {
      "StringNotLike": {
        "bedrock:FoundationModel": "anthropic.claude-*"
      }
    }
  }]
}
```
→ Anthropic Claude 계열 외 모든 모델 호출 차단

**예시 3: VPC Endpoint 경유 + MFA 필수 조건**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "bedrock:InvokeModel",
    "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.claude-*",
    "Condition": {
      "StringEquals": {
        "aws:SourceVpce": "vpce-0123456789abcdef0"
      },
      "Bool": {
        "aws:MultiFactorAuthPresent": "true"
      }
    }
  }]
}
```

**예시 4: Guardrail 적용 강제 (Guardrail 없이 모델 직접 호출 차단)**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlyWithGuardrail",
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:*::foundation-model/*",
      "Condition": {
        "StringLike": {
          "bedrock:GuardrailIdentifier": "arn:aws:bedrock:*:*:guardrail/*"
        }
      }
    },
    {
      "Sid": "DenyWithoutGuardrail",
      "Effect": "Deny",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:*::foundation-model/*",
      "Condition": {
        "Null": {
          "bedrock:GuardrailIdentifier": "true"
        }
      }
    }
  ]
}
```

#### VPC Endpoint Policy를 통한 네트워크 수준 제어

Bedrock VPC Endpoint에 Policy를 설정하여 네트워크 레벨에서 추가 제어 가능:
```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::123456789012:role/ECS-LLM-Gateway-Role"
    },
    "Action": [
      "bedrock:InvokeModel",
      "bedrock:InvokeModelWithResponseStream"
    ],
    "Resource": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-*"
  }]
}
```
→ 특정 IAM Role만, 특정 모델만, VPC Endpoint를 통해서만 호출 가능

#### Bedrock IAM 세분화 수준 요약

- ✅ **모델 ARN 단위** 호출 제한 (Foundation Model, Provisioned Throughput, Custom Model 각각)
- ✅ **ABAC 지원** — Agent, KB, Guardrail 등에 태그 기반 동적 접근 제어
- ✅ **Bedrock 전용 Condition Key** — ServiceTier, ThirdPartyKB 등 서비스 특화 조건
- ✅ **AWS 글로벌 Condition Key** — SourceVpc, SourceIp, MFA, PrincipalOrgID 등 모두 사용 가능
- ✅ **SCP로 조직 수준 모델 차단** — 승인된 모델만 사용하도록 가드레일 설정
- ✅ **VPC Endpoint Policy** — 네트워크 레벨에서 Principal + Action + Resource 3중 제어
- ✅ **Guardrail 적용 강제** — IAM Condition으로 Guardrail 없는 직접 호출 차단 가능
- ⚠️ 일부 관리 API (`CreateAgent`, `ListFoundationModels` 등)는 리소스 레벨 권한 미지원 (`*`만 가능)
- ⚠️ Foundation Model은 AWS 소유 리소스이므로 `aws:ResourceTag` 사용 불가 (Custom Model, Agent 등 고객 소유 리소스에만 태그 가능)

---

## C. AWS CloudTrail — API 호출 로깅 세분화

**로깅 유형**:

1. **관리 이벤트 (Management Events)** — 기본 활성화
   - AWS 리소스에 대한 관리 작업 기록 (CreateBucket, RunInstances, PutBucketPolicy 등)
   - 읽기/쓰기 이벤트 분리 가능
   - 기록 내용: 호출자 ID, 소스 IP, 호출 시간, 요청 파라미터, 응답 요소

2. **데이터 이벤트 (Data Events)** — 별도 활성화 필요 (추가 비용)
   - 리소스 내부의 데이터 수준 작업 기록
   - **Bedrock 관련**: 모델 호출(InvokeModel), Guardrail 평가, Agent 호출, Knowledge Base 쿼리, Session 활동 등 매우 세분화된 데이터 이벤트 로깅 가능
   - **S3**: 객체 수준 API (GetObject, PutObject, DeleteObject)
   - **Lambda**: 함수 실행 (Invoke)
   - **DynamoDB**: 아이템 수준 API (PutItem, GetItem, DeleteItem)
   - 특정 리소스 ARN 또는 접두사(prefix)로 필터링 가능

3. **네트워크 활동 이벤트 (Network Activity Events)**
   - VPC Endpoint를 통한 네트워크 수준 API 호출 기록

4. **CloudTrail Insights**
   - 비정상적인 API 호출 볼륨 또는 에러율 자동 탐지
   - 기준선(baseline) 대비 이상치 자동 알림

**기록되는 상세 정보 (각 이벤트 레코드)**:
- `userIdentity` — 호출자 정보 (IAM User, Role, Federated User, AWS Service 등)
- `sourceIPAddress` — 호출 소스 IP
- `eventName` — 호출된 API 이름
- `requestParameters` — 요청 파라미터 전체 (어떤 리소스에 어떤 설정을 적용했는지)
- `responseElements` — 응답 내용 (생성된 리소스 ID 등)
- `resources` — 관련 리소스 ARN 목록
- `errorCode` / `errorMessage` — 실패 시 에러 정보

**세분화 수준 요약**:
- ✅ 모든 AWS API 호출 기록 (관리 이벤트)
- ✅ 데이터 수준 작업까지 기록 가능 (데이터 이벤트)
- ✅ Bedrock 모델 호출, Guardrail, Agent, Session 등 AI 워크로드 세분화 로깅
- ✅ 호출자, 소스 IP, 요청 파라미터, 응답까지 전체 기록
- ✅ 리소스 ARN 단위 필터링
- ✅ 비정상 패턴 자동 탐지 (Insights)
- ❌ 데이터 이벤트는 추가 비용 발생
- ❌ 일부 서비스의 데이터 플레인 호출은 CloudTrail 미지원 (서비스별 확인 필요)

---

## D. Amazon Bedrock — AI 모델 호출 로깅 및 제어 세분화

**모델 호출 로깅 (Invocation Logging)**:
- 대상 API: `InvokeModel`, `InvokeModelWithResponseStream`, `Converse`, `ConverseStream`
- 로깅 가능 항목:
  - ✅ 전체 입력 프롬프트 (텍스트)
  - ✅ 전체 출력 응답 (텍스트)
  - ✅ 이미지/문서 데이터 (S3 로깅 시)
  - ✅ 모델 ID, 호출 시간, 지연시간
  - ✅ 입력/출력 토큰 수
  - ✅ 에러 정보
- 저장 대상: S3 (대용량, 이미지 포함) 및/또는 CloudWatch Logs (실시간 모니터링)
- ⚠️ `bedrock-runtime` 엔드포인트를 통한 호출만 로깅 (Responses API 등 다른 엔드포인트는 미지원)

**Bedrock Guardrails — 콘텐츠 제어 세분화**:
- Content Filter: 유해 콘텐츠 카테고리별 차단 강도 설정 (NONE/LOW/MEDIUM/HIGH)
- Denied Topics: 금지 주제 정의 (자연어로 주제 설명)
- PII Detection: 개인정보 유형별 감지 및 마스킹/차단 (이름, 이메일, 전화번호, 주민번호 등)
- Word Filter: 특정 단어/구문 차단
- Contextual Grounding: 환각(hallucination) 감지

**CloudTrail 데이터 이벤트를 통한 Bedrock 세분화 로깅**:
- `AWS::Bedrock::Model` — 모델 호출 이벤트
- `AWS::Bedrock::Guardrail` — Guardrail 평가 이벤트
- `AWS::Bedrock::AgentAlias` — Agent 호출 이벤트
- `AWS::Bedrock::KnowledgeBase` — Knowledge Base 쿼리 이벤트
- `AWS::Bedrock::Session` — 세션 활동 이벤트
- `AWS::Bedrock::FlowAlias` — Flow 실행 이벤트

**세분화 수준 요약**:
- ✅ 프롬프트/응답 전문 로깅 가능
- ✅ 토큰 단위 사용량 추적
- ✅ 콘텐츠 카테고리별 필터링 강도 조절
- ✅ PII 유형별 개별 감지/마스킹 설정
- ✅ CloudTrail로 모델/Agent/Guardrail/KB/Session 각각 독립 로깅
- ❌ Responses API 등 일부 엔드포인트는 Invocation Logging 미지원

---

## E. AWS API Gateway — API 호출 로깅 세분화

**액세스 로그 (Access Logs)** — 커스텀 포맷 지원:
- `$context.identity.sourceIp` — 호출자 소스 IP
- `$context.identity.caller` / `$context.identity.user` — IAM 호출자 식별
- `$context.identity.cognitoIdentityId` — Cognito 사용자 식별
- `$context.identity.apiKey` / `$context.identity.apiKeyId` — API 키 식별
- `$context.authorizer.principalId` — Lambda Authorizer에서 반환한 Principal
- `$context.authorizer.claims.*` — Cognito 토큰 클레임 (사용자 속성)
- `$context.httpMethod`, `$context.resourcePath` — 호출된 메서드 및 리소스 경로
- `$context.status` — 응답 상태 코드
- `$context.responseLatency` — 응답 지연시간
- `$context.requestId` / `$context.extendedRequestId` — 요청 추적 ID
- `$context.wafResponseCode` — WAF 평가 결과
- `$context.authorize.status` / `$context.authorize.error` — 인가 결과 및 에러
- `$context.authenticate.status` / `$context.authenticate.error` — 인증 결과 및 에러
- `$context.identity.clientCert.*` — mTLS 클라이언트 인증서 정보 (Subject DN, Issuer DN, Serial Number, 유효기간)

**실행 로그 (Execution Logs)**:
- 요청/응답 본문 전체 로깅 가능 (설정에 따라)
- 통합(Integration) 요청/응답 상세
- 인증/인가 과정 상세 로그

**세분화 수준 요약**:
- ✅ 호출자 식별 (IP, IAM, Cognito, API Key, mTLS 인증서)
- ✅ 인증/인가 과정 및 결과 상세 로깅
- ✅ WAF 평가 결과 포함
- ✅ 요청/응답 본문 로깅 가능 (실행 로그)
- ✅ 커스텀 포맷으로 필요한 필드만 선택 가능
- ✅ Lambda Authorizer 커스텀 컨텍스트 로깅

---

## F. AWS ECS — 컨테이너 워크로드 로깅 세분화

**컨테이너 로그**:
- `awslogs` 드라이버를 통해 stdout/stderr → CloudWatch Logs 전송
- 애플리케이션 레벨에서 구조화된 로그 출력 시 JSON 필드 단위 검색 가능

**VPC Flow Logs v7 ECS 메타데이터**:
- `ecs-cluster-arn` / `ecs-cluster-name` — 클러스터 식별
- `ecs-service-name` — 서비스 식별
- `ecs-task-arn` / `ecs-task-id` — 태스크 식별
- `ecs-task-definition-arn` — 태스크 정의 식별
- `ecs-container-id` / `ecs-second-container-id` — 컨테이너 레벨 식별
- `ecs-container-instance-arn` — 컨테이너 인스턴스 식별 (EC2 launch type)

**ECR 이미지 스캔**:
- 이미지 레이어 단위 취약점 스캔 (CVE 기반)
- 심각도별 분류 (Critical, High, Medium, Low, Informational)

**세분화 수준 요약**:
- ✅ 컨테이너 레벨 네트워크 트래픽 추적 (Flow Logs v7)
- ✅ 태스크/서비스/클러스터 단위 식별
- ✅ 애플리케이션 로그 구조화 가능
- ✅ 이미지 취약점 CVE 단위 스캔

---

## G. AWS Config — 리소스 구성 변경 추적 세분화

**기록 단위**: Configuration Item (CI) — 리소스의 전체 구성 스냅샷

**기록 내용**:
- 리소스의 모든 속성(Attribute) 변경 이력
- 리소스 간 관계(Relationship) 매핑 (예: EC2 인스턴스 ↔ Security Group ↔ VPC)
- 태그 변경 이력
- IAM Policy 변경 내용

**기록 빈도**: 연속(Continuous) 또는 일일(Daily) 선택 가능

**Config Rules 평가 세분화**:
- 리소스 유형별 규칙 적용 (300+ 리소스 유형 지원)
- 변경 트리거(Change-triggered) 또는 주기적(Periodic) 평가
- 커스텀 Lambda 규칙으로 자체 평가 로직 구현 가능
- Conformance Pack으로 프레임워크 기반 규칙 묶음 적용

**세분화 수준 요약**:
- ✅ 리소스 속성 단위 변경 추적
- ✅ 리소스 간 관계 매핑
- ✅ 300+ AWS 리소스 유형 지원
- ✅ 커스텀 평가 규칙 작성 가능
- ✅ 자동 수정(Remediation) 연동 가능
- ❌ 데이터 플레인 활동은 추적하지 않음 (컨트롤 플레인 구성 변경만)
