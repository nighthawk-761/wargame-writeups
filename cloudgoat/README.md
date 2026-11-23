# CloudGoat Writeups

[CloudGoat](https://github.com/RhinoSecurityLabs/cloudgoat)는 의도적으로 취약한 AWS Resource를 배포하는 학습 도구입니다. 아래 문서는 개인 Lab에서 직접 수행한 전체 과정과 당시 캡처를 시나리오별로 정리한 것입니다.

## Scenario 목록

| 순서 | Scenario | 핵심 학습 내용 |
| --- | --- | --- |
| 01 | [IAM Enum Basics](01-iam-enum-basics/README.md) | User·Group·Policy·Role 열거 |
| 02 | [Data Secrets](02-data-secrets/README.md) | User Data → EC2 → IMDS → Lambda → Secrets Manager |
| 03 | [Beanstalk Secrets](03-beanstalk-secrets/README.md) | 환경 변수 Secret과 `iam:CreateAccessKey` |
| 04 | [SNS Secrets](04-sns-secrets/README.md) | 과도한 구독 권한과 API Key 유출 |
| 05 | [IAM PrivEsc by Key Rotation](05-iam-privesc-by-key-rotation/README.md) | Tag 조건, Access Key, MFA와 Role 전환 |
| 06 | [IAM PrivEsc by Rollback](06-iam-privesc-by-rollback/README.md) | 관리형 Policy Version Rollback |
| 07 | [Lambda PrivEsc](07-lambda-privesc/README.md) | `iam:PassRole`과 Lambda Code 실행 |
| 08 | [SQS Flag Shop](08-sqs-flag-shop/README.md) | 신뢰되지 않은 Message와 Business Logic |
| 09 | [Static](09-static/README.md) | Public S3 Write와 JavaScript 공급망 변조 |
| 10 | [Vulnerable Cognito](10-vulnerable-cognito/README.md) | Client-side 검증과 Cognito Role Mapping |
| 11 | [Vulnerable Lambda](11-vulnerable-lambda/README.md) | Lambda SQL Injection과 권한 상승 |

각 시나리오는 다음 구조를 사용합니다.

```text
scenario/
├── README.md   # 전체 실습 과정
├── assets/     # 원본 캡처
└── notes/      # 개념·구성 보충 설명(있는 경우)
```

캡처와 출력에 등장하는 식별자와 자격 증명은 당시 CloudGoat 실습 환경에서 생성된 값입니다. 재사용하지 말고, 실습 완료 후 반드시 해당 Resource를 제거해야 합니다.

## 공통 시작·종료 절차

```bash
# 관리자 Profile로 Scenario 생성
cloudgoat create <SCENARIO> --profile <LAB_ADMIN_PROFILE>

# 공격자 Profile의 현재 신원 확인
aws sts get-caller-identity --profile <PROFILE>

# 완료 후 Resource 제거
cloudgoat destroy <DEPLOYED_SCENARIO> --profile <LAB_ADMIN_PROFILE>
cloudgoat list deployed
```

`start.txt`, Terraform State와 CLI Credential 파일에는 민감정보가 들어갈 수 있으므로 Commit하지 않습니다. `destroy` 전에 Scenario Directory와 State를 삭제하면 Resource 정리가 어려워질 수 있습니다.

## 참고

- [CloudGoat Repository](https://github.com/RhinoSecurityLabs/cloudgoat)
- [CloudGoat Wiki](https://github.com/RhinoSecurityLabs/cloudgoat/wiki)
- [AWS IAM Security Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
