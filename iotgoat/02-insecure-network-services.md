# 2. Insecure Network Services

[← IoTGoat 목록](README.md)

NW 서비스를 하면서 그냥 열려있는 포트에 대해서 원격 접근을 한다면, 정보 누출이나 미인증 RCE를 유발할 수도 있다. 

nmap 포트 스캐닝 도구를 통해 먼저 서비스중인 포트를 확인해보려고 한다. 

TCP-SYN을 활용한 스캔 옵션으로 nmap -sS 옵션을 사용하였다. 

TCP는 4개의 포트가 열려있는 것을 볼 수 있다. 

![image.png](assets/image%2021.png)

이번엔 nmap -sU -p- -T4 옵션을 활용하여 더 빠른 속도로 UDP 포트를 확인해보았다.
(다만 대상의 설정에 따라 -T4 옵션은 목표의 F/W, IPS, IDS, UTM의 작동을 야기할 수 있다.) 

시간이 너어어어어무 오래걸리는 것 같아서, UDP 0-1023으로 Well-known만 먼저 해보려고 한다.

![image.png](assets/image%2022.png)

![image.png](assets/image%2023.png)

시간에 비해 결과가…. 엄… 

일단 udp는 두고 tcp를 검사했던 저 옵션에는 -p-를 지정 하지않아 디폴트로 상위 1000개의 포트만을 검사한다는 것을 깨달아서, 다시 검사를 진행하였다. 

![image.png](assets/image%2024.png)

![image.png](assets/image%2025.png)

22번 은 ssh, 53번은 domain이니 DNS 포트, 80과 443은 각각 http와 https 포트로 well-known 포트다. 그리고 65534번과 5515번은 unknown으로 열려있었다.

먼저 5515번을 접속하였더니 

![image.png](assets/image%2026.png)

백도어를 발견했다면서 root 계정으로 접속에 성공하는 모습이었다. 

발견한 다른 포트인 65534번 포트에서는 로그인창이 나타났다. 그래서 1번에서 확보한 iotgoatuser로 그인을 진행해보았더니, 실제 로그인 창이 맞았다. 

![image.png](assets/image%2027.png)

## 백도어를 통한 관리자 계정탈취, 관리자 비밀번호 변경

그런데 우리는 여기서 더 나아가야만 했다… 백도어를 발견하고 히히 좋다 라고만 하고 있었지 실질적으로는 아무것도 하지 않았기에 3번에서 로그인창 어케 뚫지 이러고 있었다.

그래서 루트백도어로 진입, passwd 명령어를 활용해 root의 비밀번호를 00000000으로 바꾸었더니,

![image.png](assets/image%2028.png)

![image.png](assets/image%2029.png)

드디어 진입에 성공하였다. 

