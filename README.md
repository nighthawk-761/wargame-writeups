# Wargame Writeups

직접 구성하거나 명시적으로 허가받은 보안 실습 환경에서 학습한 내용을 정리합니다. 실습 과정의 캡처와 함께 공격 경로가 성립한 이유, 관련 개념, 방어 방법을 기록합니다.

## Writeups

| Wargame | Platform | 내용 |
| --- | --- | --- |
| [CloudGoat](cloudgoat/README.md) | AWS | IAM, EC2, S3, Lambda, Cognito, SNS·SQS의 잘못된 구성을 이용한 11개 시나리오 |

## 공개 원칙

- Write-up과 캡처는 직접 수행한 격리형 실습 환경의 기록만 사용합니다.
- 문서에 보이는 계정, 키, IP, Resource ID와 Flag는 CloudGoat가 실습용으로 생성한 값입니다.
- 명령은 해당 Wargame이 만든 격리된 Resource만 대상으로 합니다.
- 공격 단계와 함께 Root Cause, Detection, Mitigation과 Cleanup을 기록합니다.
- 실습이 끝나면 생성한 Resource와 로컬 Credential Profile을 제거합니다.

> 이 저장소의 내용은 교육과 방어 목적입니다. 소유하거나 명시적으로 허가받지 않은 시스템에 적용하면 안 됩니다.
