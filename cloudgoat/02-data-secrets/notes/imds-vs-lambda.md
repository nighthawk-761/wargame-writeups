# IMDS에서 찾은 것과 Lambda에서 찾은 것

[← 시나리오 본문](../README.md)

핵심은 서로 다른 세 종류의 AWS 신원을 차례로 사용했다는 것입니다. IMDS에서 찾은 자격 증명과 Lambda에서 찾은 자격 증명이 다른 것은 정상이며, 바로 그것이 시나리오의 권한 이동 과정입니다.

```
IMDS = Instance Metadata Service
```

## 전체 신원 이동

```
① start_user
   │ EC2 정보와 User Data만 조회 가능
   ▼
② EC2 Linux 사용자
   │ SSH로 서버 내부 진입
   ▼
③ EC2 Instance Role
   │ Lambda 설정 조회 가능
   ▼
④ Lambda 환경 변수에 저장된 IAM 사용자
   │ Secrets Manager 조회 가능
   ▼
⑤ 최종 Secret
```

각 단계에서 사용하는 신원과 권한이 다릅니다.

## 1. 처음의 `start_user`

CloudGoat이 처음 제공한 Access Key는 저권한 IAM 사용자의 키였습니다.

```
신원:
IAM User start_user

권한:
ec2:DescribeInstances
ec2:DescribeInstanceAttribute
ec2:DescribeTags
```

그래서 할 수 있었던 것은:

```
aws ec2 describe-instances
aws ec2 describe-instance-attribute
```

정도였습니다.

할 수 없었던 것은:

```
Lambda 설정 조회
Secrets Manager 조회
IAM 정보 조회
```

그래서 `iam:GetUser`도 `AccessDenied`가 발생했습니다.

## 2. SSH로 들어간 Linux 사용자

User Data에서 발견한 ID/PW는 AWS 신원이 아니었습니다.

```
AWS IAM User 아님
AWS Access Key 아님
EC2 운영체제의 Linux 사용자
```

이 계정으로 가능한 것은:

```
EC2 운영체제 로그인
파일과 프로세스 접근
EC2 내부 네트워크 접근
IMDS 접근
```

입니다.

따라서 SSH 진입만으로 AWS 관리자 권한을 얻은 것은 아닙니다.

## 3. IMDS에서 찾은 EC2 Role

EC2에는 Instance Profile을 통해 IAM Role이 연결되어 있었습니다.

```
EC2 Instance
  └─ Instance Profile
       └─ EC2 IAM Role
```

AWS 서비스나 애플리케이션은 이 Role을 이용해 별도의 장기 Access Key 없이 AWS API를 호출합니다.

EC2 내부에서 다음 경로를 조회했습니다.

```
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

이 명령은 EC2에 연결된 Role 이름을 알려줍니다.

그다음:

```
curl \
  "http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE_NAME"
```

을 통해 그 Role의 임시 자격 증명을 확인했습니다.

여기서 찾은 자격 증명의 신원은 대략 다음 형태입니다.

```
arn:aws:sts::계정ID:assumed-role/cg-ec2-role-.../인스턴스ID
```

이것은 IAM 사용자가 아니라 EC2가 임시로 맡은 Role 세션입니다.

### 왜 이 Role을 찾았나?

`start_user`에게는 Lambda를 조회할 권한이 없었지만 EC2 Role에는 다음 권한이 있었습니다.

```
lambda:ListFunctions
lambda:GetFunction
lambda:GetFunctionConfiguration
```

따라서:

```
start_user 권한으로 Lambda 조회
→ AccessDenied

EC2 Role 권한으로 Lambda 조회
→ 성공
```

이 됩니다.

IMDS Role은 최종 목표가 아니라 다음 단계인 Lambda 설정을 읽기 위한 중간 권한이었습니다.

## 4. EC2 내부의 AWS CLI가 자동으로 Role을 사용한 이유

EC2 내부에서 AWS CLI를 실행할 때 별도 프로필을 지정하지 않았습니다.

```
aws sts get-caller-identity
```

그런데도 인증된 이유는 AWS CLI의 자격 증명 탐색 순서 때문입니다.

대략적으로:

```
명령행 옵션 확인
→ 환경 변수 확인
→ ~/.aws/credentials 확인
→ EC2의 IMDS 확인
```

앞쪽에 자격 증명이 없으면 AWS CLI가 IMDS에서 Instance Role의 임시 자격 증명을 자동으로 가져옵니다.

따라서 실제로는 다음처럼 동작한 것입니다.

```
EC2 내부 AWS CLI
→ IMDS 조회
→ EC2 Role 임시 키 획득
→ Lambda API 호출
```

우리가 `curl`로 IMDS를 직접 조회한 것은 이 과정을 눈으로 확인하고, 서버가 침해됐을 때 자격 증명이 어떻게 탈취될 수 있는지 배우기 위해서였습니다.

## 5. Lambda에서 찾은 IAM 사용자

EC2 Role 권한으로 Lambda의 설정을 조회했습니다.

```
aws lambda get-function-configuration ...
```

Lambda 환경 변수 안에는 별도의 Access Key와 Secret Key가 들어 있었습니다.

```
DB_USER_ACCESS_KEY
DB_USER_SECRET_KEY
```

이 키는 EC2 Role의 임시 자격 증명이 아닙니다. 별도로 생성된 IAM 사용자의 장기 자격 증명이었습니다.

신원 확인 결과도 다음과 같은 형태였을 것입니다.

```
arn:aws:iam::계정ID:user/cg-lambda-user-...
```

이 IAM 사용자는 다음 권한을 가지고 있었습니다.

```
secretsmanager:ListSecrets
secretsmanager:DescribeSecret
secretsmanager:GetSecretValue
```

따라서:

```
EC2 Role로 Secrets Manager 조회
→ 권한 없음

Lambda에서 발견한 IAM 사용자로 조회
→ 성공
```

이 됩니다.

## 두 자격 증명의 차이

| 구분 | IMDS에서 찾은 것 | Lambda에서 찾은 것 |
| --- | --- | --- |
| 신원 종류 | IAM Role의 STS 세션 | IAM User |
| 사용 주체 | EC2 인스턴스 | 별도의 사용자 |
| 키 성격 | 임시 자격 증명 | 장기 Access Key |
| Session Token | 필요 | 보통 없음 |
| 만료 | 자동 만료 | 삭제·비활성화 전까지 유효 |
| 주요 권한 | Lambda 설정 조회 | Secrets Manager 조회 |
| 목적 | 다음 자격 증명 발견 | 최종 Secret 접근 |

Access Key 앞부분도 보통 다릅니다.

```
ASIA... → STS 임시 자격 증명인 경우가 일반적
AKIA... → IAM 사용자의 장기 Access Key인 경우가 일반적
```

IMDS 자격 증명에는 세 값이 필요했습니다.

```
AccessKeyId
SecretAccessKey
Token
```

Lambda 환경 변수의 IAM 사용자 키에는 보통 두 값만 있었습니다.

```
AccessKeyId
SecretAccessKey
```

## Lambda의 실행 Role과도 구분해야 함

Lambda 설정에는 다음 필드도 있었습니다.

```
"Role": "arn:aws:iam::...:role/cg-lambda-exec-role-..."
```

이것은 Lambda 코드가 실행될 때 사용하는 Lambda Execution Role입니다.

이번 공격에서 사용한 것은 이 Role이 아닙니다.

구분하면:

```
EC2 Instance Role
→ 공격자가 Lambda 설정을 조회하는 데 사용

Lambda Execution Role
→ Lambda 코드가 실행될 때 사용하는 권한
→ 이번 공격에서 직접 사용하지 않음

Lambda 환경 변수의 IAM User 키
→ 공격자가 Secrets Manager를 읽는 데 사용
```

따라서 이 시나리오에는 눈에 보이는 AWS 신원이 여러 개 존재했습니다.

## 왜 이렇게 여러 신원을 거치는가?

AWS에서는 하나의 사용자에게 모든 권한을 주기보다 역할별로 권한을 나누는 것이 일반적입니다.

```
start_user
→ EC2 정보만 조회

EC2 Role
→ Lambda 정보만 조회

lambda_user
→ Secrets Manager만 조회
```

원래라면 이 분리가 보안을 높입니다. 하지만 각 단계에서 자격 증명이 잘못 노출됐기 때문에 분리된 권한이 공격 경로로 연결됐습니다.

```
User Data에 Linux 비밀번호 노출
        ↓
EC2 내부 접근
        ↓
EC2 Role 자격 증명 사용
        ↓
Lambda 환경 변수에 IAM 키 노출
        ↓
Secrets Manager 접근
```

즉, IMDS에서 Role을 찾은 목적은 관리자 권한을 얻기 위해서가 아니라 **현재 `start_user`에게 없던 Lambda 조회 권한을 얻기 위해서**였습니다. 그리고 Lambda에서 찾은 별도 사용자는 **EC2 Role에게 없던 Secrets Manager 권한을 얻기 위해서** 사용했습니다.

각 신원은 완전한 관리자가 아니었지만, 이전 신원이 갈 수 없던 다음 단계로 이동할 수 있게 해주는 징검다리였습니다.