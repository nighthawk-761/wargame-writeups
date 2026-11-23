# User Data 사용의 맥락

[← 시나리오 본문](../README.md)

User Data는 한마디로 “새 EC2가 처음 부팅될 때 자동으로 실행할 초기 설정서”입니다.

맥락은 이렇습니다.

```
AWS가 기본 Linux 이미지로 EC2 생성
        ↓
아직 웹 서버, 사용자, 애플리케이션이 없음
        ↓
User Data 스크립트를 root 권한으로 실행
        ↓
설치와 설정이 끝난 사용 가능한 서버가 됨
```

## 누가 설정하는가?

EC2를 만드는 사람이 설정합니다. 보통 다음 주체 중 하나입니다.

- 관리자가 AWS 웹 콘솔에서 직접 입력
- 개발자나 DevOps 엔지니어가 Terraform에 작성
- CloudFormation 같은 IaC 템플릿에 작성
- Launch Template에 저장
- Auto Scaling이 Launch Template을 이용해 반복 적용
- 애플리케이션 배포 시스템이 EC2 생성 API에 전달

웹 콘솔에서 직접 EC2를 만들 경우 대략 다음 위치입니다.

```
EC2 → Launch instance → Advanced details → User data
```

Terraform에서는 다음과 비슷합니다.

```
resource "aws_instance" "server" {
  ami           = "ami-..."
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl enable nginx
    systemctl start nginx
  EOF
}
```

CloudGoat도 Terraform이 EC2를 만들면서 취약하게 작성된 User Data를 함께 전달한 것입니다.

## 언제 실행되는가?

Linux EC2에서는 일반적으로 다음 순서로 실행됩니다.

```
EC2 생성
→ 운영체제 부팅
→ cloud-init 실행
→ User Data 해석
→ root 권한으로 스크립트 실행
```

기본적으로 User Data는 인스턴스의 최초 부팅 과정에서 한 번 실행됩니다. 별도 설정을 하면 재부팅 때마다 실행하도록 만들 수도 있습니다. AWS 공식 문서도 Linux User Data가 셸 스크립트나 `cloud-init` 명령으로 전달되며 기본적으로 최초 부팅 때 root로 실행된다고 설명합니다. [AWS EC2 User Data 문서](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)

## 왜 필요한가?

AMI는 서버의 기본 디스크 이미지입니다. Ubuntu AMI를 선택하면 Ubuntu는 설치되어 있지만, 사용자가 원하는 애플리케이션까지 자동으로 구성되는 것은 아닙니다.

예를 들어 웹 서버를 10대 만들어야 한다면 매번 SSH로 들어가 다음 작업을 반복하는 것은 비효율적입니다.

```
패키지 업데이트
nginx 설치
설정 파일 작성
애플리케이션 다운로드
서비스 시작
모니터링 에이전트 설치
```

User Data를 사용하면 새 EC2가 스스로 초기 설정을 완료합니다.

```
#!/bin/bash

apt-get update
apt-get install -y nginx

echo "Hello" > /var/www/html/index.html

systemctl enable nginx
systemctl start nginx
```

Auto Scaling에서도 중요합니다. 트래픽이 증가해 새 EC2가 자동 생성되면 관리자가 직접 들어갈 시간이 없습니다. Launch Template에 User Data를 넣어두면 새 인스턴스가 자동으로 같은 구성을 갖추게 됩니다. [AWS Auto Scaling Launch Template 문서](https://docs.aws.amazon.com/autoscaling/ec2/userguide/launch-templates.html)

## ID와 비밀번호를 넣을 맥락이 있나?

기술적으로는 있습니다. 과거나 단순한 개발 환경에서는 다음과 같은 목적으로 비밀번호를 넣기도 했습니다.

### 초기 운영체제 사용자 생성

```
useradd application
echo 'application:초기비밀번호' | chpasswd
```

새 서버에 애플리케이션용 Linux 사용자를 자동 생성하려는 경우입니다.

### SSH 비밀번호 접속 활성화

```
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' \
  /etc/ssh/sshd_config

systemctl restart sshd
```

SSH 키 관리가 번거롭다는 이유로 사용자명과 비밀번호 로그인을 활성화하는 경우입니다.

### 애플리케이션 초기 비밀번호 설정

```
export ADMIN_PASSWORD="..."
./install-application.sh
```

웹 관리 페이지의 최초 관리자 계정을 만들려는 경우입니다.

### 데이터베이스 연결 정보 전달

```
export DB_USERNAME="..."
export DB_PASSWORD="..."
```

애플리케이션을 처음 실행할 때 데이터베이스 접속 정보가 필요하다는 이유로 넣는 경우도 있습니다.

### 외부 저장소 다운로드

```
curl -u "user:password" https://example.com/application.tar.gz
```

사설 패키지나 설치 파일을 내려받기 위해 인증정보를 넣는 경우입니다.

이런 맥락은 존재하지만, 보안상 좋은 방식은 아닙니다.

## 왜 User Data에 비밀정보를 넣으면 안 되는가?

User Data의 Base64 인코딩은 암호화가 아닙니다.

```
평문 스크립트
→ Base64 인코딩
→ AWS에 전달
```

Base64는 전송 형식을 맞추기 위한 표현 방식일 뿐입니다. 누구나 다음처럼 되돌릴 수 있습니다.

```
echo "Base64문자열" | base64 --decode
```

또한 User Data는 다음 경로로 노출될 가능성이 있습니다.

- `ec2:DescribeInstanceAttribute` 권한이 있는 AWS 사용자
- EC2 내부에서 메타데이터에 접근할 수 있는 사용자나 프로세스
- Terraform 코드와 상태 파일
- Launch Template
- Cloud-init 로그
- User Data 스크립트가 저장된 로컬 파일
- Git에 잘못 커밋된 IaC 코드

따라서 User Data는 “누구도 볼 수 없는 비밀 저장소”가 아니라 “권한이 있는 주체가 다시 읽을 수 있는 서버 초기화 설정”으로 취급해야 합니다.

## 안전한 User Data 사용법

User Data에는 비밀값 자체를 넣지 않고, 서버를 구성하는 일반 명령만 넣습니다.

좋은 예:

```
#!/bin/bash

apt-get update
apt-get install -y nginx awscli
systemctl enable nginx
systemctl start nginx
```

피해야 할 예:

```
#!/bin/bash

DB_PASSWORD="실제비밀번호"
AWS_SECRET_ACCESS_KEY="실제AWS키"
echo 'admin:실제비밀번호' | chpasswd
```

비밀정보가 필요하다면 AWS Secrets Manager나 Systems Manager Parameter Store에서 부팅 시 가져오도록 구성합니다.

개념적으로는:

```
User Data
   ↓
EC2에 연결된 IAM Role로 인증
   ↓
Secrets Manager에서 필요한 비밀 조회
   ↓
메모리 또는 제한된 설정 파일에 적용
```

예시 형태:

```
DB_PASSWORD="$(aws secretsmanager get-secret-value \
  --secret-id production/database \
  --query SecretString \
  --output text)"
```

여기에도 비밀번호 자체는 들어 있지 않습니다. EC2의 IAM Role이 해당 Secret만 읽을 수 있도록 권한을 제한합니다.

AWS Access Key도 EC2에 직접 저장하지 않습니다.

```
나쁜 방식:
Access Key와 Secret Key를 User Data에 하드코딩

좋은 방식:
EC2 Instance Profile/IAM Role 사용
```

SSH 접속도 가능하면 다음 방법을 사용합니다.

- SSH Key Pair
- AWS Systems Manager Session Manager
- EC2 Instance Connect
- 필요할 때만 제한된 IP에 TCP 22 허용

고정 비밀번호를 User Data에 넣는 방식은 피합니다.

## 이번 CloudGoat 시나리오의 의도

CloudGoat은 학습을 위해 일부러 다음과 같이 구성했습니다.

```
1. 관리자가 User Data에 Linux 계정 비밀번호를 작성
2. start_user에게 User Data 조회 권한을 부여
3. EC2에 공인 IP 할당
4. 보안 그룹에서 SSH 접근 허용
5. SSH 비밀번호 인증 활성화
6. EC2에 별도 IAM Role 연결
```

각 요소가 단독으로는 즉시 치명적이지 않을 수 있지만, 연결하면 공격 경로가 생깁니다.

```
start_user는 EC2 목록을 볼 수 있다
        ↓
InstanceId를 알게 된다
        ↓
User Data를 읽는다
        ↓
Linux 사용자명과 비밀번호를 알게 된다
        ↓
공인 IP로 SSH 접속한다
        ↓
EC2 내부에서 더 강한 IAM Role 자격 증명을 얻는다
```

즉, 이 시나리오에서 User Data는 단순한 파일이 아니라 `AWS API 권한 → 운영체제 접근 권한`으로 넘어가는 연결 고리입니다.

핵심만 기억하면:

```
User Data = 서버 최초 부팅 자동화 입력값
기본 실행 권한 = root
기본 실행 시점 = 최초 부팅
용도 = 설치와 환경 구성
보안 성격 = 비밀 저장소가 아님
```

User Data에는 “서버를 어떻게 구성할 것인가”는 넣을 수 있지만, “누출되면 곤란한 실제 비밀번호와 키”는 넣지 않는 것이 원칙입니다.