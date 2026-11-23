# 구성 설명

[← 시나리오 본문](../README.md)

응. 여기서는 **“VPC에 들어가는 자원”과 “AWS가 VPC 밖에서 제공하는 관리형 서비스”**를 구분하면 갑자기 구조가 깔끔해져.

네가 받은 환경을 먼저 그리면 대략 이렇게 보는 게 맞아.

```
AWS Account / Region
│
├─ VPC
│   │
│   ├─ EC2
│   │    └─ ENI + Private IP
│   │
│   ├─ RDS
│   │    └─ ENI + Private IP
│   │
│   └─ Lambda
│        └─ VPC 연결
│           (Hyperplane ENI를 통해 접근)
│
├─ SQS
│    └─ AWS가 운영하는 Regional Service
│       VPC에 직접 들어가 있지 않음
│
└─ IAM
     └─ IAM User
        AWS 계정의 Identity
        VPC와 무관
```

여기서 **Lambda만 조금 특이**하고, SQS는 아예 성격이 다르다.

---

# 1. RDS가 뭔가

RDS는 **Relational Database Service**야.

쉽게 말하면:

> **AWS가 OS와 DB 서버 운영을 상당 부분 대신해주는 관계형 데이터베이스 서비스**
> 

야. PostgreSQL, MySQL 같은 DB를 직접 EC2에 설치하는 것과 비교하면 이해하기 쉽다. AWS는 RDS에서 프로비저닝, 설정, 백업, 패치 같은 DB 운영 작업 상당 부분을 관리한다. ([Amazon Web Services, Inc.](https://aws.amazon.com/rds/?utm_source=chatgpt.com))

### EC2에 PostgreSQL을 직접 설치하면

```
EC2
├─ Linux
│
├─ PostgreSQL
│
├─ DB 데이터
│
└─ 네가 관리
   ├─ OS 패치
   ├─ DB 설치
   ├─ DB 패치
   ├─ 백업
   ├─ 스토리지
   └─ 장애 대응
```

### RDS PostgreSQL을 사용하면

```
RDS
├─ AWS가 관리하는 서버/OS
├─ PostgreSQL
├─ 스토리지
├─ 백업
└─ 장애복구 기능

사용자
├─ DB 생성
├─ Table 생성
├─ 계정/권한 관리
├─ SQL 실행
├─ 애플리케이션 연결
└─ 일부 DB 설정
```

즉 RDS도 결국 어딘가에서는 CPU, RAM, 스토리지 위에서 DB 엔진이 돌아가지만 **그 아래 OS에 SSH해서 들어가 직접 관리하는 서비스가 아니다.**

---

# 2. 왜 RDS는 VPC 안에 있나?

RDS DB 인스턴스를 만들 때 VPC와 **DB Subnet Group**을 지정한다.

RDS는 이 Subnet Group에서 실제 사용할 Subnet과 IP를 선택하고, 그 IP를 가진 ENI를 만들어 DB 인스턴스에 연결한다. Security Group도 함께 붙는다. ([AWS Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html?utm_source=chatgpt.com))

그래서 네 입장에서는 정말로:

```
VPC
│
├─ Subnet A
│   └─ EC2
│        10.0.1.10
│
└─ Subnet B
    └─ RDS
         10.0.2.15
```

같은 느낌이다.

EC2 애플리케이션에서:

```
10.0.1.10 EC2
     │
     │ TCP 3306
     ▼
10.0.2.15 RDS MySQL
```

처럼 같은 VPC 내부 네트워크로 연결할 수 있다. AWS도 EC2와 RDS를 같은 VPC에 배치하는 것을 일반적인 구성으로 설명한다. ([AWS Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.Scenarios.html?utm_source=chatgpt.com))

물론 실제로는 RDS endpoint DNS를 사용해서:

```
mydb.xxxxx.ap-northeast-2.rds.amazonaws.com:3306
```

처럼 접속하는 게 일반적이야.

---

# 3. EC2는 이미 알고 있듯 진짜 VPC 자원에 가깝다

EC2를 만들 때:

```
VPC
Subnet
Security Group
Private IP
ENI
```

를 정하지.

ENI 자체가 AWS 정의상 **VPC 안의 논리적인 가상 NIC**다. ([AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html?utm_source=chatgpt.com))

그래서:

```
EC2
 ↓
ENA / 가상 NIC
 ↓
ENI
 ↓
Subnet
 ↓
VPC
```

라는 구조로 이해하면 된다.

---

# 4. 그런데 Lambda는 조금 조심해야 한다

문제에서:

> `1 VPC with Lambda x1, RDS x1, EC2 x1`
> 

이라고 써놨더라도 **Lambda가 EC2처럼 VPC의 Subnet 안에 실제 서버로 들어가 있는 건 아니다.**

Lambda 자체는 AWS가 운영하는 Lambda 서비스에서 실행된다. AWS는 Lambda를 사용자가 서버를 프로비저닝하거나 관리하지 않는 serverless compute 서비스로 정의한다. ([AWS Documentation](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html?utm_source=chatgpt.com))

기본 Lambda라면:

```
AWS Lambda Service
      │
      └─ Lambda Function

VPC
├─ EC2
└─ RDS
```

이런 관계다.

그런데 Lambda가:

```
EC2의 private IP로 접속
RDS의 private endpoint로 접속
```

해야 한다면 VPC 접근이 필요하다.

그래서 Lambda를 VPC에 연결한다.

```
AWS Lambda Service
       │
       │
   Hyperplane ENI
       │
       ▼
┌────── VPC ──────┐
│                  │
│ EC2              │
│ RDS              │
│                  │
└──────────────────┘
```

AWS는 Lambda를 VPC에 연결하면 **Hyperplane ENI**를 생성해서 Lambda 함수와 VPC 자원 사이의 네트워크 인터페이스 역할을 하게 한다. ([AWS Documentation](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html?utm_source=chatgpt.com))

그러니까 정확히는:

```
EC2
→ VPC 안에 직접 존재

RDS
→ VPC Subnet에 ENI/IP를 가짐

Lambda
→ AWS Lambda 서비스에서 실행
→ Hyperplane ENI를 통해 VPC에 접속
```

이 차이가 있다.

문제 출제자가 편의상 셋을:

> `VPC with Lambda, RDS, EC2`
> 

라고 묶은 것 같아.

---

# 5. 그러면 SQS는 뭐길래 VPC 밖에 따로 적혀 있나?

SQS는 **Simple Queue Service**.

이름 그대로 **메시지 Queue 서비스**야.

예를 들어 웹 서비스가 있다고 하자.

사용자가 사진을 업로드하면 이미지 변환에 30초가 걸린다.

### SQS 없이

```
사용자
 ↓
EC2 Web Server
 ↓
이미지 변환 30초
 ↓
DB 저장
 ↓
응답
```

EC2가 작업을 끝낼 때까지 붙잡혀 있다.

그런데 SQS를 중간에 넣으면:

```
사용자
 ↓
EC2
 ↓
SQS Queue
“사진 123 변환하세요”
 ↓
사용자에게 빠르게 응답

            SQS
             ↓
        Worker / Lambda
             ↓
       실제 이미지 변환
```

즉 SQS는:

> **어떤 프로그램이 다른 프로그램에게 “이 작업 해줘”라는 메시지를 임시로 맡겨두는 곳**
> 

이다.

AWS도 SQS를 소프트웨어 구성요소 사이에서 메시지를 보내고 저장하고 받는 완전관리형 메시지 큐로 설명한다. ([Amazon Web Services, Inc.](https://aws.amazon.com/tw/sqs/?utm_source=chatgpt.com))

---

# 6. Queue라는 게 정확히 어떤 느낌인가

예를 들어 SQS에 현재:

```
Queue: jobs

Message 1
"사용자 321에게 이메일 보내기"

Message 2
"image.jpg 썸네일 만들기"

Message 3
"주문 #5923 처리하기"
```

가 있다고 하자.

Worker가:

```
ReceiveMessage
```

를 하면 메시지를 하나 받아간다.

```
Worker
   │
   │ ReceiveMessage
   ▼
SQS
   │
   └─ "image.jpg 썸네일 만들기"
```

작업 성공 후:

```
DeleteMessage
```

하면 큐에서 메시지를 제거한다.

아주 기본적으로는:

```
Producer
   ↓ SendMessage
  SQS
   ↓ ReceiveMessage
Consumer
   ↓ 작업 완료
DeleteMessage
```

구조다.

그래서 Lambda와도 굉장히 많이 연결된다.

```
EC2
 ↓ SendMessage
SQS
 ↓
Lambda 실행
 ↓
RDS 업데이트
```

같은 아키텍처가 가능하다.

---

# 7. 그런데 왜 SQS는 VPC 안에 안 들어가나?

이게 네 질문의 핵심이야.

SQS는 **너에게 특정 VM/네트워크 인터페이스 하나를 배정해주는 서비스가 아니기 때문**이야.

EC2는:

```
내 VM
내 ENI
내 Private IP
내 Subnet
```

이 존재한다.

RDS도:

```
내 DB Instance
내 ENI/IP
내 DB Subnet
```

이 존재한다.

하지만 SQS는:

```
AWS가 운영하는 거대한 SQS 서비스
          │
          ├─ 네 Queue A
          ├─ 다른 고객 Queue B
          ├─ 다른 고객 Queue C
          └─ ...
```

라는 서비스 모델이다.

그래서 네가 SQS를 만들 때:

```
VPC?
Subnet?
Private IP?
```

를 지정하지 않는다.

대신:

```
Region
Queue 이름
Queue 종류
Queue 정책
Encryption
```

등을 설정한다.

SQS API에는 리전별 서비스 endpoint가 존재한다. ([AWS Documentation](https://docs.aws.amazon.com/general/latest/gr/sqs-service.html?utm_source=chatgpt.com))

즉 구조상:

```
┌──────── 네 VPC ────────┐
│                        │
│ EC2 ───────────────┐    │
│ Lambda             │    │
│ RDS                │    │
│                    │    │
└────────────────────┼────┘
                     │
                     │ AWS Service API
                     ▼
                Amazon SQS
               ┌──────────┐
               │ Queue    │
               └──────────┘
```

가 된다.

---

# 8. 그럼 Private Subnet EC2가 SQS에 접근하면 인터넷을 나가야 하나?

여기서 **VPC Endpoint**가 나온다.

기본적으로 SQS는 VPC 내부 자원이 아니지만 AWS PrivateLink 기반의 **Interface VPC Endpoint**를 만들 수 있다.

```
VPC
│
│ EC2
│  ↓
│ SQS Interface VPC Endpoint
│  ↓
└──── AWS Private Network ───→ SQS
```

그러면:

```
Internet Gateway X
NAT Gateway X
Public IP X
```

여도 SQS와 사설 연결이 가능하다. AWS도 SQS용 Interface VPC Endpoint를 사용하면 인터넷 게이트웨이, NAT, VPN 없이 VPC에서 SQS로 연결할 수 있다고 설명한다. ([AWS Documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-internetwork-traffic-privacy.html?utm_source=chatgpt.com))

이건 앞으로 AWS 네트워킹 공부하면서 정말 자주 보게 될 구조야.

---

# 9. IAM User는 더더욱 VPC와 관련 없다

IAM User는 컴퓨팅이나 네트워크 자원이 아니야.

```
IAM User
├─ 사용자 이름
├─ Access Key
├─ Console Password
└─ IAM Policy
```

를 가진 **Identity**다.

즉:

```
IAM User
       │
       │ AWS API 호출
       ▼
AWS IAM:
“이 사람이 이 작업을 할 권한이 있나?”
       │
       ├─ Allow
       └─ Deny
```

역할이다.

IAM은 AWS 리소스 접근을 제어하는 서비스고, AWS 서비스 요청의 인증·인가에 사용된다. ([AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html?utm_source=chatgpt.com))

그래서 IAM User는:

```
Subnet 없음
Private IP 없음
Security Group 없음
VPC 없음
```

이다.

AWS 서비스 endpoint 관점에서도 IAM은 글로벌 endpoint를 제공하는 서비스다. ([AWS Documentation](https://docs.aws.amazon.com/general/latest/gr/rande.html?utm_source=chatgpt.com))

---

# 10. 그래서 네 문제 환경을 다시 해석하면

아마 대략 이런 상황일 거야.

```
                    IAM User
                       │
                       │ AWS API
                       ▼
              ┌──── AWS Account ────┐
              │                     │
              │     Amazon SQS      │
              │       Queue         │
              │         ▲           │
              │         │           │
              │  ┌──────┼── VPC ─┐ │
              │  │      │        │ │
              │  │   Lambda      │ │
              │  │      │        │ │
              │  │      ▼        │ │
              │  │     RDS       │ │
              │  │      ▲        │ │
              │  │      │        │ │
              │  │     EC2       │ │
              │  │               │ │
              │  └───────────────┘ │
              └─────────────────────┘
```

각 자원의 역할은 이렇게 잡으면 된다.

| 자원 | 역할 | VPC 관계 |
| --- | --- | --- |
| **VPC** | 네트워크 격리 영역 | 그 자체 |
| **EC2** | 서버/VM | VPC 내부 |
| **RDS** | 관리형 관계형 DB | VPC Subnet에 연결 |
| **Lambda** | 서버리스 코드 실행 | 필요하면 VPC에 연결 |
| **SQS** | 메시지 Queue | VPC 외부 AWS Regional Service |
| **IAM User** | AWS API 호출 Identity | VPC와 무관 |

그리고 네가 지금 문제를 풀면서 제일 먼저 머릿속에 잡아야 할 건 이 구분이야:

```
Compute
EC2 / Lambda

Database
RDS

Messaging
SQS

Network
VPC

Identity & Authorization
IAM
```

CloudGoat류 문제라면 이 구성은 상당히 의미심장해. 단순히 “EC2 하나 해킹하기”보다는 **IAM 권한 → Lambda 코드/권한 → SQS 메시지 → RDS 자격증명 또는 EC2**처럼 여러 AWS 서비스 사이의 **trust relationship과 권한 흐름**을 보게 만들 가능성이 높아. 다만 실제 문제의 공격 경로는 문제 설명이나 IAM 정책을 보기 전에는 단정하면 안 돼.