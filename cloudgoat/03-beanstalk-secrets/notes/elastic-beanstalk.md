# Beanstalk은 무엇인가

[← 시나리오 본문](../README.md)

`Beanstalk`, `Elastic Beanstalk`, `EB`는 여기서는 모두 같은 서비스를 뜻합니다.

정식 명칭은:

```
AWS Elastic Beanstalk
```

사람들이 대화나 코드에서 줄여서 다음처럼 부릅니다.

```
Beanstalk
EB
Elastic Beanstalk
```

AWS CLI에서는 서비스 이름을 붙여서 사용합니다.

```
aws elasticbeanstalk ...
```

## Elastic Beanstalk를 한 문장으로 정의하면

개발자가 애플리케이션 코드를 올리면, AWS가 그 코드를 실행하는 데 필요한 EC2·로드밸런서·Auto Scaling 등의 인프라를 대신 구성하고 운영해주는 배포 서비스입니다.

```
개발자
  │
  │ 애플리케이션 코드 업로드
  ▼
Elastic Beanstalk
  ├─ EC2 생성
  ├─ 운영체제와 Runtime 준비
  ├─ 코드 배포
  ├─ Load Balancer 구성
  ├─ Auto Scaling 구성
  ├─ 상태 검사
  └─ 로그와 배포 관리
```

AWS는 Elastic Beanstalk가 용량 프로비저닝, 로드 밸런싱, 자동 확장, 애플리케이션 상태 모니터링을 처리한다고 설명합니다. [AWS Elastic Beanstalk FAQ](https://aws.amazon.com/elasticbeanstalk/faqs/)

## 일반 EC2 배포와 무엇이 다른가?

### EC2를 직접 사용하는 경우

개발자 또는 운영자가 직접 해야 합니다.

```
1. EC2 생성
2. 보안 그룹 설정
3. 운영체제 접속
4. Python, Java, Node.js 등의 Runtime 설치
5. nginx나 Apache 설치
6. 애플리케이션 코드 복사
7. 서비스 등록
8. 로그 설정
9. Load Balancer 설정
10. Auto Scaling 설정
11. 버전 업데이트와 롤백 처리
```

예를 들어 Python 웹 애플리케이션을 직접 배포한다면:

```
sudo dnf install python3 nginx
pip install flask gunicorn
scp application.py ec2-user@서버:/opt/app/
sudo systemctl enable application
sudo systemctl start application
```

같은 과정을 직접 설계해야 합니다.

### Elastic Beanstalk를 사용하는 경우

개발자는 보통 다음 정도만 지정합니다.

```
코드
사용할 플랫폼
인스턴스 크기
환경 변수
확장 정책
```

예:

```
application.zip 업로드
Platform: Python
Environment: production
```

그러면 Elastic Beanstalk가 그 아래의 EC2와 관련 인프라를 생성합니다.

중요한 점은 Elastic Beanstalk가 EC2를 대체하는 것은 아니라는 것입니다.

```
Elastic Beanstalk
= EC2 등의 인프라를 자동으로 구성하고 관리하는 상위 계층
```

웹 호스팅 관리 패널과 비슷한 역할을 하지만, 실제 리소스는 사용자 AWS 계정에 생성됩니다.

## 언제 사용하는가?

다음과 같은 경우에 적합합니다.

- 웹 애플리케이션을 빠르게 AWS에 배포하고 싶을 때
- EC2를 직접 하나씩 구성하고 싶지 않을 때
- 트래픽 증가에 따라 서버 수를 자동 조절하고 싶을 때
- 개발·테스트·운영 환경을 비슷한 구성으로 만들고 싶을 때
- 새 버전 배포와 롤백을 쉽게 관리하고 싶을 때
- 개발자는 애플리케이션 코드에 집중하고 싶은 경우

지원되는 대표적인 플랫폼은 다음과 같습니다.

```
Python
Node.js
Java
.NET
PHP
Ruby
Go
Docker
```

Elastic Beanstalk의 플랫폼은 단순히 언어만 의미하지 않습니다.

```
Platform
= 운영체제
+ 언어 Runtime
+ 웹 서버
+ 애플리케이션 서버
+ Beanstalk 관리 구성 요소
```

[AWS 공식 플랫폼 설명](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/concepts-all-platforms.html)

## 핵심 구성요소

Elastic Beanstalk를 조사하려면 다음 용어를 구분해야 합니다.

```
Application
  ├─ Application Version 1
  ├─ Application Version 2
  └─ Environment
       ├─ Platform
       ├─ Environment Configuration
       ├─ Environment Variables
       └─ 실제 AWS 리소스
            ├─ EC2
            ├─ Load Balancer
            ├─ Auto Scaling
            └─ Security Group
```

### Application

Elastic Beanstalk 리소스를 묶는 논리적인 상위 컨테이너입니다.

예를 들어 회사에 `shopping-mall`이라는 서비스가 있다면:

```
Application: shopping-mall
```

이 안에 코드 버전과 여러 환경을 구성합니다.

Application 자체가 실행 중인 서버는 아닙니다. 프로젝트나 제품 단위에 가깝습니다.

### Application Version

배포 가능한 특정 버전의 코드입니다.

```
shopping-mall-v1
shopping-mall-v2
shopping-mall-v3
```

실제 코드는 일반적으로 ZIP, WAR 등의 배포 패키지로 S3에 저장됩니다.

```
Application Version
→ S3에 저장된 배포 코드 패키지를 가리킴
```

새 버전에 문제가 있으면 이전 Application Version으로 되돌릴 수 있습니다.

### Environment

코드가 실제로 실행되는 AWS 리소스 묶음입니다.

예를 들어:

```
shopping-mall-dev
shopping-mall-staging
shopping-mall-production
```

각 환경은 독립적인 EC2와 설정을 가질 수 있습니다.

```
Application: shopping-mall

Environment: dev
  └─ 저렴한 EC2 1대

Environment: production
  ├─ Load Balancer
  ├─ EC2 여러 대
  └─ Auto Scaling
```

AWS 공식 정의상 Environment는 하나의 Application Version을 실행하는 AWS 리소스들의 모음입니다. [Elastic Beanstalk 개념 문서](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/concepts.html)

### Environment Tier

환경의 실행 방식을 나타냅니다.

#### Web server tier

HTTP 요청을 처리하는 일반적인 웹 애플리케이션입니다.

```
사용자
→ Load Balancer
→ EC2 웹 애플리케이션
```

#### Worker tier

웹 요청에 바로 응답하기보다 큐에서 작업을 가져와 백그라운드 처리를 합니다.

```
SQS Queue
→ Worker Environment
→ 이미지 처리, 이메일 발송, 데이터 변환
```

### Environment Configuration

환경이 어떻게 동작할지 정하는 설정 모음입니다.

예:

```
EC2 인스턴스 타입
최소·최대 인스턴스 개수
Load Balancer 종류
보안 그룹
배포 정책
로그 설정
상태 검사 URL
환경 변수
Instance Profile
```

### Environment Variable

애플리케이션 코드에 외부 설정을 전달하는 key-value 값입니다.

```
KEY=VALUE
```

예:

```
APP_ENV=production
LOG_LEVEL=info
API_ENDPOINT=https://api.example.com
DB_HOST=database.example.com
```

Python에서는:

```
import os

db_host = os.environ["DB_HOST"]
```

Node.js에서는:

```
const dbHost = process.env.DB_HOST;
```

로 값을 사용합니다.

AWS 공식 문서에서도 Environment Properties 또는 Environment Variables를 endpoint, debug 설정, 데이터베이스 연결 정보 등을 애플리케이션에 전달하는 용도로 설명합니다. [Elastic Beanstalk 환경 변수 문서](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environments-cfg-softwaresettings.html)

## 환경 변수는 누가 설정하는가?

일반적으로 개발자나 운영자가 다음 경로로 설정합니다.

- Elastic Beanstalk 웹 콘솔
- EB CLI
- AWS CLI
- Terraform
- CloudFormation
- 애플리케이션의 `.ebextensions` 설정 파일
- Launch Template이나 배포 파이프라인

웹 콘솔에서는 대략 다음 위치입니다.

```
Elastic Beanstalk
→ Environment
→ Configuration
→ Updates, monitoring, and logging
→ Runtime environment variables
```

예를 들어 운영자가 애플리케이션의 데이터베이스 주소를 코드에 넣고 싶지 않다면:

```
DB_HOST=production-db.example.com
```

을 환경 변수로 설정할 수 있습니다.

## 환경 변수에 ID/PW나 키를 넣는 맥락

실제 개발 현장에서도 다음과 같은 값을 환경 변수로 전달하는 경우가 있습니다.

```
DB_USERNAME
DB_PASSWORD
API_KEY
OAUTH_CLIENT_SECRET
SMTP_PASSWORD
```

그 이유는 코드를 환경별로 다시 작성하지 않기 위해서입니다.

나쁜 방식:

```
DB_PASSWORD = "실제 운영 비밀번호"
```

코드 안에 비밀번호가 들어가면 Git 저장소에 노출될 수 있습니다.

그래서 코드와 설정을 분리하려고:

```
DB_PASSWORD = os.environ["DB_PASSWORD"]
```

로 바꿉니다.

이 방향 자체는 맞지만, Elastic Beanstalk의 일반 평문 환경 변수에 장기 AWS Access Key를 직접 넣으면 다른 문제가 생깁니다.

```
환경 설정을 읽을 수 있는 IAM 사용자
→ Environment Variable 조회
→ Access Key와 Secret Key 획득
→ 다른 AWS 사용자로 인증
```

이것이 이번 CloudGoat 시나리오의 취약점입니다.

## 안전한 환경 변수 사용

일반 환경 변수에는 민감하지 않은 설정을 넣습니다.

```
APP_ENV
LOG_LEVEL
API_ENDPOINT
REGION
FEATURE_FLAG
```

비밀번호나 API 키는 Secrets Manager 또는 Systems Manager Parameter Store에 보관합니다.

현대적인 구조는 다음과 같습니다.

```
Elastic Beanstalk 애플리케이션
  │
  │ EC2 Instance Role로 인증
  ▼
Secrets Manager
  │
  └─ 필요한 Secret만 조회
```

최근 Elastic Beanstalk는 환경 변수의 값을 평문으로 직접 저장하는 대신 Secrets Manager 또는 Parameter Store의 ARN을 참조하는 기능도 제공합니다. 이 경우 EC2 Instance Profile에 해당 Secret을 읽는 권한이 필요합니다. [Elastic Beanstalk 환경 Secret 설정](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/command-options-general.html)

## Elastic Beanstalk의 IAM Role들

Elastic Beanstalk에서는 Role이 여러 개 나올 수 있어 구분이 중요합니다.

### 현재 조사하는 IAM 사용자

AWS CLI 명령을 보내는 사람입니다.

```
start_user
secondary_user
admin_user
```

### Elastic Beanstalk Service Role

Elastic Beanstalk 서비스가 환경 상태를 확인하고 AWS 리소스를 관리할 때 사용하는 Role입니다.

```
Elastic Beanstalk 서비스
→ Service Role
→ 환경 관리
```

### EC2 Instance Profile

Beanstalk가 생성한 EC2에서 실행되는 애플리케이션이 사용하는 AWS 권한입니다.

```
Beanstalk EC2
→ Instance Profile
→ S3, CloudWatch 등 접근
```

### 애플리케이션 환경 변수의 Access Key

이번 문제에서 잘못 저장된 별도 IAM 사용자의 장기 자격 증명입니다.

```
Environment Variable
→ IAM Access Key
→ 다른 IAM 사용자 권한
```

Role ARN이나 Access Key가 출력된다고 모두 같은 신원은 아닙니다. 매번 다음 명령으로 현재 신원을 확인하는 습관이 중요합니다.

```
aws sts get-caller-identity --profile "프로필명"
```

## 비용 구조

Elastic Beanstalk 서비스 자체에는 별도 추가 요금이 없습니다. 하지만 Beanstalk가 생성한 실제 리소스에는 요금이 발생할 수 있습니다.

```
Elastic Beanstalk 자체: 추가 요금 없음

생성된 리소스:
EC2            → 과금 가능
Load Balancer  → 과금 가능
NAT Gateway    → 과금 가능
RDS            → 과금 가능
S3             → 저장 및 요청 요금
데이터 전송     → 과금 가능
```

AWS도 Elastic Beanstalk 자체는 추가 비용이 없고, 실제 애플리케이션을 실행하는 EC2·로드밸런서 등의 비용을 지불한다고 설명합니다. [Elastic Beanstalk 요금](https://aws.amazon.com/elasticbeanstalk/pricing/)

따라서 이 시나리오도 완료 후 반드시 `destroy`해야 합니다.

## 이번 CloudGoat 시나리오의 구조

로컬 CloudGoat 설명에 따르면 `beanstalk_secrets`는 다음 리소스를 생성합니다.

```
VPC
Elastic Beanstalk Environment
저권한 시작 사용자
두 번째 IAM 사용자
Secrets Manager Secret
```

공식 공격 흐름은 다음과 같습니다.

```
1. 저권한 사용자의 Access Key로 시작
2. Elastic Beanstalk Application과 Environment 열거
3. Environment 설정 조회
4. Environment Variables에서 두 번째 사용자 키 발견
5. 두 번째 사용자의 IAM 권한 조사
6. 관리자 사용자에 대해 새 Access Key 생성 가능함을 발견
7. 관리자 사용자 키 생성
8. 관리자 권한으로 Secrets Manager의 최종 플래그 조회
```

이번에는 앞 시나리오보다 명확한 권한 상승이 포함됩니다.

```
시작 사용자
  └─ Beanstalk 읽기 권한

두 번째 사용자
  └─ 특정 관리자 사용자의 Access Key 생성 권한

관리자 사용자
  └─ Secrets Manager를 포함한 광범위한 권한
```

## 첫 번째 조사에서 보게 될 명령

시나리오를 생성하고 시작 키를 프로필에 등록한 다음, 먼저 신원을 확인합니다.

```
aws sts get-caller-identity \
  --profile beanstalk_start
```

Elastic Beanstalk Application 목록:

```
aws elasticbeanstalk describe-applications \
  --profile beanstalk_start \
  --region "실제_시나리오_리전"
```

여기서 선택할 정보:

```
ApplicationName
```

Environment 목록:

```
aws elasticbeanstalk describe-environments \
  --application-name "발견한_ApplicationName" \
  --profile beanstalk_start \
  --region "실제_시나리오_리전"
```

여기서 선택할 정보:

```
EnvironmentName
EnvironmentId
ApplicationName
Status
Health
PlatformArn 또는 SolutionStackName
```

그다음 환경 설정을 조회합니다.

```
aws elasticbeanstalk describe-configuration-settings \
  --application-name "ApplicationName" \
  --environment-name "EnvironmentName" \
  --profile beanstalk_start \
  --region "실제_시나리오_리전"
```

결과에서 다음 Namespace를 찾게 됩니다.

```
aws:elasticbeanstalk:application:environment
```

그 아래 `OptionSettings`가 애플리케이션 환경 변수입니다.

```
{
  "Namespace": "aws:elasticbeanstalk:application:environment",
  "OptionName": "변수 이름",
  "Value": "변수 값"
}
```

이번 시나리오에서는 이 위치에 다음 단계로 이어지는 자격 증명이 잘못 보관되어 있습니다.

## 정보 활용 흐름

이전 문제와 동일하게 하나의 출력에서 다음 API의 입력값을 고릅니다.

| 발견한 정보 | 의미 | 다음 사용처 |
| --- | --- | --- |
| `ApplicationName` | Beanstalk 프로젝트 이름 | Environment 조회 |
| `EnvironmentName` | 실제 실행 환경 이름 | 구성 설정 조회 |
| `OptionSettings` | 환경 설정 목록 | 민감한 환경 변수 식별 |
| Access Key/Secret Key | 두 번째 IAM 사용자 자격 증명 | 새 AWS CLI 프로필 |
| 두 번째 사용자의 ARN | 현재 권한 주체 | IAM 정책 조사 |
| 관리자 사용자 이름 | Access Key 생성 대상 | `iam:CreateAccessKey` |
| 관리자 Access Key | 관리자 신원으로 전환 | Secrets Manager 조회 |

핵심은:

```
Beanstalk Application
→ Environment
→ Configuration Settings
→ Environment Variables
→ 다음 IAM 신원
```

순서입니다.

## 이번 시나리오에서 배울 보안 문제

이 문제는 다음 세 가지를 보여줍니다.

### 민감정보를 평문 환경 변수에 저장

환경 설정 조회 권한만 있는 사용자에게 다른 사용자의 AWS 키가 노출됩니다.

### 장기 Access Key 사용

IAM 사용자의 Access Key는 삭제하거나 비활성화하기 전까지 계속 사용할 수 있습니다. EC2 Instance Role의 임시 자격 증명보다 노출 위험이 큽니다.

### 과도한 IAM 권한

두 번째 사용자에게 다른 고권한 사용자의 Access Key를 생성할 수 있는 `iam:CreateAccessKey` 권한이 있습니다.

이 권한은 단순한 사용자 관리 권한처럼 보이지만:

```
관리자 사용자의 Access Key 생성
→ 그 키로 관리자 인증
→ 관리자 계정 탈취
```

가 가능하므로 사실상 권한 상승 경로입니다.

정리하면, Elastic Beanstalk는 코드를 업로드하면 EC2 기반 실행 환경을 자동 구성해주는 PaaS에 가까운 관리형 배포 서비스입니다. 이번 CloudGoat 문제에서는 서비스 자체를 공격하는 것이 아니라, **Beanstalk Environment 설정을 읽을 수 있는 저권한 사용자가 평문 환경 변수에서 다른 IAM 자격 증명을 발견하고, 그 자격 증명의 과도한 IAM 권한을 이용해 관리자 신원으로 이동하는 과정**을 배우게 됩니다.