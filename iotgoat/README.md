# IoTGoat Write-up

[← Wargame Writeups](../README.md)

IoTGoat 실습 환경을 구성하고 펌웨어·네트워크·웹 인터페이스·업데이트 체인·오래된 구성요소를 분석한 기록입니다. 큰 실습 항목별로 원문과 캡처를 나누었습니다.

## 목차

| 순서 | 주제 | 주요 내용 |
| --- | --- | --- |
| 00 | [Lab Setup](00-lab-setup.md) | VMware의 VM–Host 네트워크 연결 문제 해결 |
| 01 | [Hardcoded Credentials](01-hardcoded-credentials.md) | 펌웨어 추출, 계정 탐색과 인증 정보 분석 |
| 02 | [Insecure Network Services](02-insecure-network-services.md) | 열린 서비스 조사와 백도어 접근 |
| 03 | [Insecure Ecosystem Interfaces](03-insecure-ecosystem-interfaces.md) | LuCI 웹 인터페이스와 입력 처리 분석 |
| 04 | [Lack of Secure Update Mechanism](04-insecure-update-mechanism.md) | 패키지 업데이트 체인과 위조 패키지 실습 |
| 05 | [Insecure or Outdated Components](05-insecure-outdated-components.md) | Kernel, BusyBox, dnsmasq, wpa_supplicant, pppd 점검 |

> 직접 구성한 격리형 실습 환경을 대상으로 진행했습니다. 캡처는 당시 실습 화면을 그대로 보존합니다.
