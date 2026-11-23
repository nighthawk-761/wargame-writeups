# AWS에서 Lambda란?

[← 시나리오 본문](../README.md)

지금 핵심은 “관리자 권한을 얻는 것”이 아닙니다. CloudGoat 시나리오의 목표는 관리자 권한 획득이 아니라, 여러 제한된 신원을 거쳐 최종 Secret을 읽을 수 있는 신원까지 이동하는 것입니다.

현재까지의 신원 변화는 다음과 같습니다.

```
start_user
  └─ EC2 정보와 User Data를 읽는 권한

EC2의 Linux 사용자
  └─ 서버 운영체제 내부에 접속하는 권한

EC2 Instance Role
  └─ Lambda 목록과 설정을 읽는 AWS 권한
```

각 신원은 관리자 권한이 아니라 다음 단계로 가는 데 필요한 일부 권한만 갖습니다.

## Lambda란?

AWS Lambda는 서버를 직접 관리하지 않고 코드를 실행하는 서비스입니다.

일반 서버 프로그램은:

```
EC2 생성
→ Linux 관리
→ 프로그램 설치
→ 프로세스를 계속 실행
```

하지만 Lambda는:

```
코드 업로드
→ 어떤 이벤트가 발생
→ AWS가 코드를 잠깐 실행
→ 실행이 끝나면 종료
```

하는 방식입니다.

예를 들어 S3에 파일이 업로드될 때 이미지를 변환하는 코드가 있다면:

```
사용자가 S3에 이미지 업로드
        ↓
Lambda 실행
        ↓
이미지 크기 변환
        ↓
결과를 다른 S3 경로에 저장
```

Lambda를 “특정 이벤트가 발생할 때 AWS가 대신 실행해주는 작은 프로그램”이라고 이해하면 됩니다.

## Lambda 함수 하나를 구성하는 요소

Lambda 함수에는 대략 다음 정보가 들어 있습니다.

```
함수 이름
실행할 코드
Python/Node.js 같은 Runtime
메모리와 실행 제한시간
환경 변수
Lambda 실행용 IAM Role
이 함수를 실행시키는 Trigger
```

`aws lambda list-functions`를 실행하면 계정의 Lambda 프로그램 목록이 나옵니다.

예시 구조:

```
{
  "Functions": [
    {
      "FunctionName": "example-function",
      "FunctionArn": "arn:aws:lambda:us-east-1:...:function:example-function",
      "Runtime": "python3.12",
      "Role": "arn:aws:iam::...:role/lambda-execution-role",
      "Handler": "lambda_function.lambda_handler",
      "CodeSize": 1234,
      "Timeout": 3
    }
  ]
}
```

## 각 필드의 의미

### `FunctionName`

```
"FunctionName": "example-function"
```

Lambda 함수를 지정할 때 사용하는 이름입니다. EC2의 `InstanceId`와 비슷하게 다음 API의 입력값이 됩니다.

```
aws lambda get-function-configuration \
  --function-name "example-function" \
  --region us-east-1
```

### `Runtime`

```
"Runtime": "python3.12"
```

Lambda 코드를 어떤 실행 환경에서 실행하는지를 뜻합니다.

```
python3.x → Python
nodejsxx.x → JavaScript/Node.js
javaxx → Java
```

### `Handler`

```
"Handler": "lambda_function.lambda_handler"
```

실행이 시작될 함수 위치입니다.

Python이라면 대략:

```
# lambda_function.py

def lambda_handler(event, context):
    ...
```

를 의미합니다.

### `Role`

```
"Role": "arn:aws:iam::...:role/lambda-execution-role"
```

Lambda 코드가 실행될 때 사용하는 IAM Role입니다.

이 Role은 현재 당신이 사용 중인 EC2 Instance Role과 다릅니다.

```
현재 사용자
→ EC2 Instance Role

Lambda 실행 시
→ Lambda Execution Role
```

Lambda가 S3, DynamoDB, Secrets Manager 등에 접근해야 한다면 이 실행 Role에 해당 권한을 부여합니다.

단, 출력에서 Lambda의 Role ARN을 봤다고 그 Role을 바로 사용할 수 있는 것은 아닙니다.

### `Environment`

```
"Environment": {
  "Variables": {
    "KEY": "VALUE"
  }
}
```

이번 시나리오에서 가장 중요한 부분입니다.

## Lambda 환경 변수란?

Lambda 코드에 설정값을 전달하는 key-value 저장 공간입니다.

예를 들어:

```
환경 변수
DB_HOST = database.example.com
LOG_LEVEL = INFO
BUCKET_NAME = uploads
```

Lambda 코드에서는 다음처럼 읽습니다.

```
import os

bucket_name = os.environ["BUCKET_NAME"]
```

정상적인 목적은 코드와 설정을 분리하는 것입니다.

```
코드는 동일
개발 환경: LOG_LEVEL=DEBUG
운영 환경: LOG_LEVEL=WARNING
```

하지만 개발자가 편의를 위해 다음과 같은 민감정보를 환경 변수에 넣기도 합니다.

```
DB_PASSWORD
API_KEY
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

이것이 이번 시나리오의 의도된 취약점입니다.

## 지금 Lambda 목록을 보게 된 이유

EC2에 연결된 Instance Role이 다음과 비슷한 권한을 가진 것입니다.

```
lambda:ListFunctions
lambda:GetFunctionConfiguration
```

그래서 관리자 권한은 없지만:

```
Lambda 함수 목록 조회
Lambda 함수 설정 조회
Lambda 환경 변수 조회
```

가 가능합니다.

이를 권한 관점에서 보면:

```
관리자 권한: AWS의 거의 모든 작업 가능
현재 Role: Lambda의 일부 읽기 작업만 가능
```

현재 Role은 제한적이지만, Lambda 설정에 더 강한 자격 증명이 노출되어 있다면 다음 단계로 이동할 수 있습니다.

## 다음에 해야 할 일

`list-functions` 결과에서 `FunctionName`을 찾습니다.

예를 들어:

```
"FunctionName": "cg-something-function"
```

함수 이름을 변수에 넣습니다.

```
LAMBDA_NAME="실제_FunctionName"
```

상세 설정을 조회합니다.

```
aws lambda get-function-configuration \
  --function-name "$LAMBDA_NAME" \
  --region us-east-1
```

출력에서 다음 부분을 찾습니다.

```
"Environment": {
  "Variables": {
    ...
  }
}
```

특히 이름에 다음 단어가 들어간 변수를 확인하세요.

```
ACCESS
SECRET
KEY
TOKEN
CREDENTIAL
AWS
```

이번 문제 설명에 따르면 Lambda 환경 변수에는 또 다른 IAM 사용자의 AWS Access Key와 Secret Key가 하드코딩되어 있습니다.

## 왜 굳이 새로운 키가 필요한가?

현재 EC2 Role은 Lambda 설정을 읽을 수 있지만 최종 Secrets Manager의 값은 읽지 못할 가능성이 큽니다.

```
EC2 Role 권한:
lambda:ListFunctions
lambda:GetFunctionConfiguration

하지만:
secretsmanager:GetSecretValue 없음
```

그런데 Lambda 환경 변수에 들어 있는 다른 사용자의 키는:

```
secretsmanager:ListSecrets
secretsmanager:GetSecretValue
```

같은 권한을 가지고 있을 수 있습니다.

따라서 공격 흐름은:

```
EC2 Role로 Lambda 설정 읽기
        ↓
Lambda 환경 변수에서 새 AWS 키 발견
        ↓
새 키를 이용해 다른 IAM 사용자로 인증
        ↓
그 사용자의 Secrets Manager 권한 사용
```

입니다.

## Lambda에서 찾은 키의 역할 확인

Lambda 환경 변수에서 Access Key와 Secret Key를 찾으면 이를 채팅에 올리지 말고 기록해둡니다.

EC2에서 계속 진행할 수도 있지만, 프로필 혼동을 피하려면 로컬 WSL로 돌아가는 편이 편합니다.

```
exit
```

로컬에서 새 프로필을 만듭니다.

```
aws configure --profile lambda_user
```

입력:

```
AWS Access Key ID: Lambda 환경 변수에서 찾은 값
AWS Secret Access Key: Lambda 환경 변수에서 찾은 값
Default region name: us-east-1
Default output format: json
```

그 신원이 누구인지 확인합니다.

```
aws sts get-caller-identity \
  --profile lambda_user \
  --region us-east-1
```

현재까지의 신원과 다른 ARN이 나오면 정상입니다.

그다음 그 신원에게 어떤 접근 경로가 있는지 확인합니다.

```
aws secretsmanager list-secrets \
  --profile lambda_user \
  --region us-east-1
```

## 전체 과정의 의미

이번 시나리오는 각각의 서비스가 취약한 것이 아니라, 민감정보가 잘못된 위치에 저장되고 읽기 권한이 과도하게 연결된 문제입니다.

```
User Data
  └─ Linux 비밀번호를 저장하면 안 됨

EC2 SSH
  └─ 유출된 비밀번호로 서버 접근

Instance Metadata
  └─ 서버 접근자는 EC2 Role 권한을 사용할 수 있음

Lambda 환경 변수
  └─ 장기 AWS Access Key를 저장하면 안 됨

Secrets Manager
  └─ 유출된 키가 최종 Secret 읽기 권한을 가짐
```

지금 Lambda 목록에서 선택해야 하는 정보는 우선 `FunctionName`입니다. 그 이름을 `get-function-configuration`에 넣고, 반환된 `Environment.Variables`를 조사하는 것이 다음 단계입니다. 관리자 권한을 찾는 것이 아니라, **현재 Role이 읽을 수 있는 설정 안에서 다음 신원의 자격 증명이 유출됐는지 찾는 과정**입니다.