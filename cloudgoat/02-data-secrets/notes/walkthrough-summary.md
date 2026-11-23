# 내가 무엇을 한 것인가

[← 시나리오 본문](../README.md)

`Secrets Manager`는 권한이 아니라 AWS의 “비밀정보 보관 서비스”입니다. 이름 때문에 `Secret Manager 권한`처럼 느껴질 수 있지만, 구조를 분리해서 보면 명확합니다.

```
AWS Secrets Manager        서비스
Secret                     서비스 안에 저장된 리소스
SecretString               실제 비밀 값
IAM Policy                 누가 Secret을 읽을 수 있는지 정하는 권한
aws secretsmanager ...     서비스에 요청을 보내는 CLI 명령
```

## 금고로 비유하면

```
Secrets Manager = 금고실
Secret          = 금고실 안의 개별 금고
Secret Name     = 금고 번호
SecretString    = 금고 안의 내용물
IAM 사용자      = 금고실에 온 사람
IAM 정책        = 그 사람이 가진 출입 권한
AWS CLI         = 직원에게 요청을 전달하는 창구
```

예를 들면:

```
aws secretsmanager list-secrets
```

는:

```
“내 권한으로 볼 수 있는 금고 목록을 보여주세요.”
```

라는 요청입니다.

```
aws secretsmanager get-secret-value \
  --secret-id "금고이름"
```

은:

```
“이 금고를 열어서 안의 내용물을 보여주세요.”
```

라는 요청입니다.

## AWS CLI 명령을 분해하면

```
aws secretsmanager list-secrets \
  --profile lambda_user \
  --region us-east-1
```

각 부분의 의미:

```
aws
└─ AWS CLI 실행

secretsmanager
└─ Secrets Manager 서비스에 요청

list-secrets
└─ Secret 목록 조회 API 호출

--profile lambda_user
└─ lambda_user의 자격 증명으로 요청

--region us-east-1
└─ 버지니아 북부 리전의 Secret 조회
```

즉, `secretsmanager`는 권한명이 아니라 `ec2`, `lambda`, `iam`과 같은 AWS 서비스 이름입니다.

```
aws ec2 describe-instances
aws lambda list-functions
aws secretsmanager list-secrets
```

모두 같은 구조입니다.

## 권한 이름은 어떻게 생기는가?

IAM 정책에서 사용하는 실제 권한 이름은 다음처럼 `서비스:동작` 형태입니다.

```
secretsmanager:ListSecrets
secretsmanager:DescribeSecret
secretsmanager:GetSecretValue
```

구분하면:

| CLI 명령 | 내부 AWS API/IAM 동작 |
| --- | --- |
| `aws secretsmanager list-secrets` | `secretsmanager:ListSecrets` |
| `aws secretsmanager describe-secret` | `secretsmanager:DescribeSecret` |
| `aws secretsmanager get-secret-value` | `secretsmanager:GetSecretValue` |

CLI 명령과 IAM Action의 이름이 비슷하지만 역할이 다릅니다.

- CLI 명령: 실행하려는 요청
- IAM Action: 그 요청을 실행할 자격이 있는지 판단하는 권한

## start_user에게는 Secret이 “없는” 것인가?

아니요. Secret은 AWS 계정 안에 동일하게 존재합니다. 다만 `start_user`에게는 Secrets Manager를 조회할 권한이 없습니다.

시나리오의 실제 정책을 확인해보면 `start_user`에게 허용된 것은 다음 세 가지뿐입니다.

```
ec2:DescribeInstances
ec2:DescribeInstanceAttribute
ec2:DescribeTags
```

따라서 `start_user`로 다음 명령을 실행하면:

```
aws secretsmanager list-secrets \
  --profile start_user \
  --region us-east-1
```

빈 목록이 아니라 보통 `AccessDenied`가 발생해야 합니다.

이 차이가 중요합니다.

### 권한이 없는 경우

```
AccessDenied
```

의미:

```
Secret이 있는지 없는지 확인할 권한조차 없음
```

### 권한은 있지만 Secret이 없는 경우

```
{
  "SecretList": []
}
```

의미:

```
목록 조회는 성공했지만 해당 리전에 Secret이 없음
```

`AccessDenied`를 보고 “리소스가 없다”고 판단하면 안 됩니다. “볼 권한이 없다”가 정확한 해석입니다.

## 이번 시나리오의 권한 분리

실제 시나리오 정책은 세 신원에게 서로 다른 권한을 줍니다.

### 1. start_user

허용된 작업:

```
EC2 목록 조회
EC2 속성 조회
EC2 태그 조회
```

그래서 할 수 있는 것:

```
aws ec2 describe-instances
aws ec2 describe-instance-attribute
```

할 수 없는 것:

```
IAM 조회
Lambda 조회
Secrets Manager 조회
```

### 2. EC2 Instance Role

허용된 작업:

```
lambda:ListFunctions
lambda:GetFunction
lambda:GetFunctionConfiguration
```

그래서 할 수 있는 것:

```
aws lambda list-functions
aws lambda get-function-configuration
```

할 수 없는 것:

```
Secrets Manager 값 조회
관리자 작업
```

### 3. Lambda 환경 변수에서 발견한 사용자

허용된 작업:

```
secretsmanager:ListSecrets
secretsmanager:DescribeSecret
secretsmanager:GetSecretValue
```

그래서 이 사용자로 전환한 뒤에야 다음 명령이 성공한 것입니다.

```
aws secretsmanager list-secrets
aws secretsmanager get-secret-value
```

이 사용자도 관리자는 아닙니다. Secrets Manager 조회에 특화된 제한된 사용자입니다.

## 그럼 무엇을 찾은 것인가?

`list-secrets` 결과에서 찾은 것은 Secret이라는 AWS 리소스입니다.

결과를 분해하면:

```
{
  "ARN": "...",
  "Name": "...",
  "Description": "...",
  "Tags": [...],
  "SecretVersionsToStages": {...}
}
```

이 정보들은 실제 비밀값이 아니라 Secret의 메타데이터입니다.

### `Name`

Secret을 지정할 때 사용하는 이름입니다.

```
aws secretsmanager get-secret-value \
  --secret-id "이 Name"
```

### `ARN`

Secret의 AWS 전체 식별자입니다.

```
arn:aws:secretsmanager:리전:계정:secret:이름
```

`Name` 대신 ARN을 `--secret-id`에 넣어도 됩니다.

### `Description`

이 Secret이 무엇을 보관하는지 설명합니다. 이번 시나리오에서는 최종 플래그라는 사실을 알려줬습니다.

### `AWSCURRENT`

현재 사용 중인 Secret 버전이라는 표시입니다. Secrets Manager는 비밀번호가 변경될 때 여러 버전을 관리할 수 있습니다.

```
AWSCURRENT  → 현재 버전
AWSPREVIOUS → 이전 버전
AWSPENDING  → 교체 진행 중인 버전
```

### `SecretString`

`get-secret-value`를 실행해야 나오는 실제 내용입니다.

이번 시나리오에서는 플래그가 들어 있지만, 현실에서는 다음과 같은 값이 저장됩니다.

```
{
  "username": "application",
  "password": "실제 데이터베이스 비밀번호"
}
```

또는:

```
외부 API 키
OAuth Client Secret
DB 접속 정보
인증서 Private Key
애플리케이션 암호화 키
```

## 이 정보가 왜 가치가 있는가?

현실에서 공격자의 최종 목적은 반드시 관리자 권한일 필요가 없습니다.

예를 들어 데이터베이스 비밀번호 하나만 얻어도:

```
고객 개인정보 조회
데이터 변경
데이터 삭제
다른 시스템으로 이동
```

이 가능할 수 있습니다.

API Key를 얻으면:

```
외부 서비스 사용
결제 기능 호출
내부 API 접근
서비스 사칭
```

이 가능할 수 있습니다.

이번 시나리오의 Secret은 플래그지만, 현실의 플래그 자리에 실제 회사의 핵심 비밀정보가 있다고 생각하면 됩니다.

```
CloudGoat:
SecretString = CTF 플래그

현실:
SecretString = DB 비밀번호, API Key, 인증서 등
```

## Secrets Manager의 정상적인 사용법

Secrets Manager를 사용하는 것 자체는 좋은 보안 방식입니다. 문제는 누가 읽을 수 있는가입니다.

정상 구조:

```
애플리케이션 EC2/Lambda
        ↓
전용 IAM Role
        ↓
특정 Secret 하나에만 GetSecretValue 허용
        ↓
실행 중 필요한 순간에 Secret 조회
```

예를 들어 결제 애플리케이션에는 결제 DB 비밀번호만 허용합니다.

```
{
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "특정 Secret ARN"
}
```

이번 시나리오에서는 Lambda 환경 변수에 별도 IAM 사용자의 장기 Access Key를 저장했고, 그 사용자에게 모든 Secret을 읽을 수 있도록 `Resource: "*"` 권한을 부여한 것이 문제였습니다.

## 이번 공격의 핵심 가치

당신이 발견한 것은 단순히 문자열 하나가 아닙니다.

```
1. Secret 리소스가 존재한다는 사실
2. 그 Secret이 최종 목표라는 메타데이터
3. lambda_user가 그 Secret을 읽을 수 있다는 사실
4. get-secret-value를 통해 실제 보호 대상 데이터에 접근 가능하다는 사실
```

보안 보고서 관점에서는 다음처럼 정리됩니다.

```
Finding:
Lambda 환경 변수에 IAM 장기 자격 증명이 노출됨

Impact:
노출된 자격 증명은 Secrets Manager의 모든 Secret을
조회할 수 있으며, 실제 보호된 SecretString 접근이 가능함

Attack path:
EC2 User Data 노출
→ SSH 접근
→ Instance Role 탈취
→ Lambda 설정 조회
→ IAM 키 노출
→ Secrets Manager 데이터 유출
```

한 문장으로 정리하면, **Secrets Manager는 비밀정보를 보관하는 AWS 서비스이고, 당신은 Lambda에서 유출된 다른 사용자의 키를 통해 그 서비스 안에 저장된 최종 보호 데이터에 접근한 것**입니다. `start_user`에게 Secret이 안 보였던 것은 Secret이 없어서가 아니라 접근 권한이 없었기 때문입니다.