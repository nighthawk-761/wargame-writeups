# 이 각 정보들이 가지는 보안적 의미

[← 시나리오 본문](../README.md)

(손님 맞을래요?) → 맞습니다. 다만 그 셸 스크립트가 직접 SSH로 접속하는 스크립트는 아닙니다. 정확히는 EC2가 처음 부팅될 때 실행되는 `User Data` 초기화 스크립트입니다.

이번 문제는 여러 개의 잘못된 설정이 연결되어 발생합니다.

```
저권한 AWS 사용자
  → EC2 User Data 읽기 가능
  → User Data에 Linux 계정 비밀번호 노출
  → 인터넷에서 SSH 접속 가능
  → EC2 내부의 IAM Role 자격 증명 탈취 가능
  → Lambda 환경 변수에서 또 다른 AWS 키 노출
  → Secrets Manager 플래그 접근
```

## 1. User Data란?

EC2를 생성할 때 초기 설정용 셸 스크립트를 전달할 수 있습니다. 

예를 들어 정상적인 사용 사례는 다음과 같습니다. 

```
#!/bin/bash

apt-get update
apt-get install -y nginx
systemctl enable nginx
systemctl start nginx
```

EC2가 처음 부팅될 때 `cloud-init`이라는 프로그램이 이 스크립트를 `root` 권한으로 실행합니다.

즉, 관리자가 서버에 직접 들어가지 않아도 다음 작업을 자동화할 수 있습니다.

- 패키지 설치
- 사용자 생성
- 설정 파일 변경
- 웹 서버 실행
- 애플리케이션 배포

User Data 자체가 취약한 기능은 아닙니다. 문제는 여기에 비밀번호나 AWS 키 같은 비밀정보를 넣는 것입니다.

## 2. 이번 User Data가 하는 일

디코딩한 스크립트에는 아마 다음과 비슷한 동작이 있을 것입니다.

### Linux 사용자 생성

```
useradd ...
```

또는:

```
adduser ...
```

EC2 운영체제 내부에 SSH로 로그인할 사용자를 만듭니다.

### 비밀번호 설정

```
echo '사용자명:비밀번호' | chpasswd
```

생성한 Linux 사용자에게 비밀번호를 부여합니다.

### SSH 비밀번호 인증 활성화

```
PasswordAuthentication yes
```

AWS EC2는 보통 SSH 키로 접속하도록 구성합니다. 그런데 이 스크립트는 사용자명과 비밀번호만으로 SSH 로그인을 허용하도록 SSH 서버 설정을 변경합니다.

### SSH 서비스 재시작

```
systemctl restart ssh
```

또는:

```
systemctl restart sshd
```

변경한 SSH 설정을 적용합니다.

따라서 이 스크립트는 “외부 서버로 SSH 접속”하는 것이 아니라, **현재 EC2가 외부 SSH 접속을 받아들이도록 서버를 설정**하는 것입니다.

## 3. 첫 번째 잘못된 설정: User Data에 비밀번호 저장

User Data에 다음과 같은 내용이 들어가면:

```
echo '사용자명:비밀번호' | chpasswd
```

EC2 생성 후에도 AWS API를 통해 User Data를 다시 읽을 수 있습니다.

이번 `start_user`에게는 다음과 같은 권한이 있는 것으로 보입니다.

```
ec2:DescribeInstances
ec2:DescribeInstanceAttribute
```

그래서 낮은 권한의 사용자임에도:

```
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute userData \
  --profile start_user \
  --region us-east-1
```

를 실행해 서버의 Linux 비밀번호를 알아낼 수 있습니다.

첫 번째 취약점은 다음 조합입니다.

```
비밀번호를 User Data에 저장
+
저권한 사용자에게 User Data 조회 권한 부여
```

## 4. 두 번째 잘못된 설정: 외부 SSH 접근 허용

EC2에는 공인 IP가 있습니다.

```
"PublicIpAddress": "..."
```

보안 그룹에서 TCP 22번 포트가 사용자 IP에 허용되어 있다면 인터넷에서 SSH로 접근할 수 있습니다.

```
사용자의 WSL
    |
    | TCP/22
    v
EC2 공인 IP
    |
    v
EC2 Linux SSH 서버
```

따라서 공격자는 다음 세 가지 정보를 결합합니다.

```
PublicIpAddress     → 접속할 서버
User Data 사용자명 → 로그인 계정
User Data 비밀번호 → 로그인 인증 정보
```

접속 형태:

```
ssh "사용자명@공인IP"
```

## 5. 세 번째 잘못된 설정: EC2에 IAM Role 연결

EC2 조회 결과에 다음 항목이 있었습니다.

```
"IamInstanceProfile": {
  "Arn": "arn:aws:iam::...:instance-profile/..."
}
```

이는 해당 EC2에 IAM Role이 연결되어 있다는 뜻입니다.

정상적인 목적은 EC2가 AWS 서비스를 사용할 때 Access Key를 파일에 저장하지 않고 임시 자격 증명을 받도록 하는 것입니다.

예를 들어 EC2 애플리케이션이 S3를 읽어야 한다면:

```
EC2
→ Instance Profile
→ IAM Role 임시 자격 증명
→ S3 접근
```

형태로 동작합니다.

문제는 EC2 내부에 들어간 사용자가 이 Role의 임시 자격 증명도 조회할 수 있다는 점입니다.

## 6. 네 번째 잘못된 설정: IMDSv1 사용 가능

결과에 다음 설정도 있었습니다.

```
"MetadataOptions": {
  "HttpTokens": "optional",
  "HttpEndpoint": "enabled"
}
```

해석하면:

- `HttpEndpoint: enabled`: 메타데이터 서비스 사용 가능
- `HttpTokens: optional`: 별도 토큰 없이 IMDSv1 조회 가능

SSH로 EC2 안에 들어간 뒤 다음 주소를 조회할 수 있습니다.

```
curl http://169.254.169.254/latest/meta-data/
```

연결된 Role 이름:

```
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

Role 임시 자격 증명:

```
ROLE_NAME="$(curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/)"

curl "http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE_NAME"
```

여기서 나오는 것은 EC2 Linux 비밀번호와 다른 종류의 인증 정보입니다.

```
AccessKeyId
SecretAccessKey
Token
Expiration
```

이 자격 증명으로 AWS CLI에서 EC2에 부여된 Role 권한을 사용할 수 있습니다.

## 7. 권한 상승이 일어나는 원리

처음에는 `start_user`로 시작했습니다.

```
start_user
```

이 사용자는 IAM 정보도 조회하지 못할 만큼 권한이 제한되어 있었습니다.

하지만 User Data에서 Linux 계정을 발견하면서 EC2 내부로 이동합니다.

```
start_user
→ EC2 Linux 사용자
```

EC2 내부에서는 Instance Profile 자격 증명을 가져올 수 있습니다.

```
EC2 Linux 사용자
→ EC2 IAM Role
```

따라서 처음보다 더 강한 AWS 권한을 가진 신원으로 바뀝니다.

이것을 클라우드 환경에서의 `pivot`, 즉 다른 신원이나 리소스로 이동하는 과정으로 이해하면 됩니다.

## 8. 각 정보가 어떻게 연결되는가

| 정보 | 의미 | 활용 |
| --- | --- | --- |
| `InstanceId` | AWS API에서 EC2를 지정하는 ID | User Data 조회 |
| `UserData.Value` | EC2 초기화 스크립트 | OS 계정 정보 발견 |
| `PublicIpAddress` | 인터넷에서 접근할 주소 | SSH 접속 |
| Security Group TCP/22 | SSH 네트워크 접근 허용 | 외부에서 EC2 연결 |
| Linux 사용자명 | 운영체제 계정 | SSH 로그인 |
| Linux 비밀번호 | 운영체제 인증 정보 | SSH 로그인 |
| `IamInstanceProfile` | EC2에 IAM Role 연결 | 새로운 AWS 신원 존재 확인 |
| `HttpTokens: optional` | IMDSv1 허용 | Role 자격 증명 조회 |
| Role 임시 키 | EC2에 부여된 AWS 권한 | 다음 AWS 서비스 조사 |

핵심적으로 이번 문제는 하나의 치명적인 취약점이라기보다 다음 오설정들이 연결된 공격 경로입니다.

```
민감정보를 User Data에 저장
+ User Data 읽기 권한 과다 부여
+ 공인 IP와 SSH 허용
+ 비밀번호 로그인 허용
+ 서버 사용자가 Instance Profile 자격 증명에 접근 가능
+ EC2 Role에 다음 단계 권한 부여
```

지금 단계에서는 User Data에서 찾은 **Linux 사용자명과 비밀번호**를 선택하고, `PublicIpAddress`를 대상으로 SSH 접속하면 됩니다. SSH에 들어간 후에는 `169.254.169.254` 메타데이터 주소를 조사하는 것이 다음 연결입니다.