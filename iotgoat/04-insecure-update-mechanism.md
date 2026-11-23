# 4. Lack of Secure Update Mechanism

[← IoTGoat 목록](README.md)

- CVE-2020-7982
    
    CVE-2020-7982는 널리 사용되는 오픈소스 라우터 펌웨어인 **OpenWrt**의 **OPKG 패키지 관리자**에서 발견된 심각한 **원격 코드 실행(Remote Code Execution, RCE)** 취약점입니다.
    
    NIST NVD(국립 취약점 데이터베이스)에 따르면 이 취약점은 CVSS 3.1 기준 **8.1 (High)** 등급을 받았습니다.
    
    ---
    
    ### 🎯 취약점의 핵심 내용
    
    이 취약점의 핵심은 **패키지 무결성 검사 우회**입니다.
    
    1. **정상적인 과정:** OpenWrt에서 사용자가 `opkg install <패키지명>` 명령을 실행하면, OPKG는 저장소(repository)에서 패키지 목록 파일을 다운로드합니다. 이 목록에는 각 패키지의 `SHA-256` 체크섬(해시값)이 포함되어 있습니다. 그 후 실제 패키지(`.ipk` 파일)를 다운로드하고, 이 파일의 해시값과 목록의 해시값이 일치하는지 검사하여 패키지가 변조되지 않았는지 확인합니다.
    2. **취약점 발생:** 공격자가 **중간자 공격(Man-in-the-Middle, MITM)**이나 DNS 스푸핑 등을 통해 라우터와 다운로드 서버 간의 통신을 가로챌 수 있다고 가정합니다.
    3. **공격 방식:** 공격자는 변조된 패키지 목록을 라우터에 전송합니다. 이때, 악성 패키지에 해당하는 **SHA-256 체크섬 값의 맨 앞에 공백(leading spaces)을 추가**합니다.
    4. **결과:** 취약한 버전의 OPKG는 이 비정상적인(앞에 공백이 있는) 체크섬 문자열을 올바르게 파싱(해석)하지 못합니다. 결과적으로, **무결성 검사 자체를 건너뛰고** "검증에 성공한 것"처럼 처리하여 다운로드한 패키지를 그대로 설치합니다.
    
    ---
    
    ### ☣️ 공격 시 영향
    
    - **임의 코드 실행:** 공격자는 이 과정을 통해 합법적인 패키지 대신 악성 코드가 포함된 패키지를 라우터에 설치할 수 있습니다.
    - **루트(Root) 권한 탈취:** OpenWrt에서 OPKG 패키지 관리자는 **루트(root) 권한**으로 실행됩니다. 따라서 악성 패키지가 성공적으로 설치되면, 공격자는 해당 라우터의 **완전한 제어권**을 획득하게 됩니다.
    - **네트워크 장악:** 라우터는 네트워크의 관문(gateway)이므로, 라우터가 장악당하면 공격자는 해당 네트워크를 통과하는 모든 트래픽을 도청하거나, 내부 네트워크의 다른 장치를 공격하는 등의 추가적인 악의적 활동을 할 수 있습니다.

위 CVE에 해당되는 버전은 다음과 같다.
→ OpenWrt 18.06.0 to 18.06.6 and 19.07.0, and LEDE 17.01.0 to 17.01.7

![image.png](assets/image%2057.png)

그리고 여기에 설치된 opkg OpenWRT 버전은 18.06.2 이다. 

## CVE-2020-7982 PoC

먼저 ARP Spoofing을 통해 중간자 공격(MITM, Man in the Middle)을 진행할 것이다.  

중간자 공격툴로는 bettercap을 사용하여 ARP Spoofing을 진행할 것이다.  

L2 Data-Link에서 동작하는 네트워크 도구로, MITM 도구이다. (이거 전버전은 ettercap) 

### 사전 작업(ARP Spoofing - 중지)

bettercap 다운로드를 하고 

![image.png](assets/image%2058.png)

probe.net을 실행할려고 하다가…! 호오오오옥시 몰라 위법 소지를 확인했는데 위법 소지 가능성이 높다는 것을 보고 일단 올스탑, 그리고 추가로 더 찾아보니 애초에 IotGoat 기기가 게이트웨이 할당이 되어 있지 않고 내부망을 타는 것 같아 ARP Spoofing 대상도 아니었다. 리버스-행복 해질 뻔했다. 

### 사전 작업(DNS Spoofing)

어차피 원래 순서는 ARP Spoofing → DNS Spoofing → 검증 부족으로 인한 악성 패키지 삽입이었는데, 여기서 저 앞 과정이 빠지는 것이다. 

현재 IoTGoat 자체에는 192.168.99.1 주소가 할당되어 있고, GW는 없다. 

Host-Only 통신용으로 VM에서 설정된 가상 어댑터는 192.168.99.210이다.

WSL의 IP는 192.168.43.2/20, GW는 192.168.32.1이다. (ip addr, route -n 명령어 활용)

내 PC(윈도우)의 IP는 192.168.219.102, GW는 192.168.219.1 이다. 

### opkg로 설치될 악성 패키지(.ipk) 생성

- 악성 패키지용 preinst 스크립트 페이로드 작성

![image.png](assets/image%2059.png)

작업용 ./web-mitm 디렉터리 생성 및 preinst 스크립트 작성 , 작성된 스크립트에 실행권한 부여 
(preinst 스크립트는 Debian 기반 시스템에서 패키지가 설치 또는 업그레이드되기 전 실행되는 쉘 스크립트이다. 엄밀히 따지만 opkg 는 debian 기반은 아니지만, dpkg를 많이 참조하였고, preinst 스크립트 또한 사용한다.)  

- 패키지 정보 파일 생성

![image.png](assets/image%2060.png)

- control 및 data 아카이브 생성

![image.png](assets/image%2061.png)

control 아카이브는 위에서 작성했던 것들을 압축, data 아카이브는 비어도 상관없다(그냥 무지성 생성) 

- .ipk로 압축

![image.png](assets/image%2062.png)

### 생성한 pwned.ipk 파일 정보 저장

![image.png](assets/image%2063.png)

파일 크기 : 684

해쉬 checksum : dcc296dae2d90187408c3ca16949a39f9a6388ee52ac1cf2288abb0d323e26dc

### 위조 packages.gz 생성

- Iotgoat가 참조하는 실제 패키지 리스트 URL 확인

![image.png](assets/image%2064.png)

src/gz openwrt_base http://downloads.openwrt.org/releases/18.06.2/packages/i386_pentium4/base

- 여기서 Packages.gz를 뒤에 붙여서 다운받기

![image.png](assets/image%2065.png)

- gunzip으로 압축해제

![image.png](assets/image%2066.png)

- 앞에서 만들어둔 pwned.ipk의 패키지 정보를 작성

![image.png](assets/image%2067.png)

여기서 아까 확인한 파일 사이즈와 SHA256체크섬을 활용, 그리고 해쉬값 앞에 공백 한 칸 추가

→ 여기서 실수해서 저 ~dc 뒤에 공백을 한 칸 넣었었다. 그게 아니라 앞에 한 칸을 넣어야한다. (공격 수행에서 다시 돌아옴) 

- 작성한 조작된 정보를 압축해제된 Packages로 넣은 뒤 압축하여 Packages.gz로 위장

![image.png](assets/image%2068.png)

- 파일 준비 완료

![image.png](assets/image%2069.png)

### 준비된 공격용 파일들을 Windows로 이동

가상 어댑터 192.168.99.210으로 위장할 것이기 때문에 wsl이 아닌 windows(내 PC)에서 서버를 만들고 진행할 예정이다. 

### 공격자 서버 실행

먼저 ‘관리자 권한으로’ PowerShell이나 cmd를 실행시켜준다. (80번 포트를 사용해야하기 때문에) 

- PowerShell에서 Python을 활용해 웹서버 실행(192.168.99.210:80) 후 유지

![image.png](assets/image%2070.png)

- 펌웨어는 기본적으로 RO이므로 RW로 재마운트

![image.png](assets/image%2071.png)

- /etc/host 에 도메인과 ip 추가(변경 전/후)

![image.png](assets/image%2072.png)

/etc/hosts 파일은 도메인 네임으로 접근할 때 /etc/hosts 파일을 먼저 확인한 후 있으면 해당 ip로 전개한다. 보통 DNS를 활용하면  Name Server에 질의를 하며 확인하는데, 그거 이전에 확인하는 로컬 NS하고 생각하면 된다. 사실 /etc/hosts가 더 먼저 있었으니 개쩌는 외장 hosts 파일이 NS 서버이지 않을까 싶다. 

### 공격 수행 - 실패

(현재 wsl에서 백도어로 root 계정 iotgoat에 들어와있는 상태다.)  

이제 iotgoat에서 opkg update를 진행 후 opkg install pwned를 해보았는데,

![image.png](assets/image%2073.png)

실패해 버렸다. 

이유를 분석한 결과 이제 도메인 부분은 /etc/hosts로 처리해버렸지만, 뒤의 디렉터리 주소 부분이 PowerShell에서 대기하고 있던 부분과 맞지 않았던 것이다. 

- 그래서 요청하는 디렉터리 대로 만들고 Packages.gz와 pwned.ipk를 저 base 안에 박아버렸다.

![image.png](assets/image%2074.png)

다시 대기한다.

![image.png](assets/image%2075.png)

- 그리고 해쉬 무결점을 깨뜨리는 것이 메인인데, 착각해서 앞에 공백이 아니라 뒤에 공백을 넣어서 해쉬체크섬이 계속 동작중이었다.  그래서 Packages.gz를 다시 만들었다.

![image.png](assets/image%2076.png)

제일 위에 있는 체크섬이 pwned의 해쉬체크섬인데, 혼자 한칸 더 간 것도 볼 수 있다.  

### 공격 수행 - 재시도

다시 한 번 opkg update를 시도하였고, iotgoat 측에서도 중간에 Update Avail이 하나 떴고, 

![image.png](assets/image%2077.png)

대기중이던 PowerShell에서도 http 성공 코드 200이 뜬 것을 볼 수 있다. 

![image.png](assets/image%2078.png)

그러나 마지막 opkg install pwned 에서는 되지 않고 있다.

![image.png](assets/image%2079.png)

원인을 확인해보니 

![image.png](assets/image%2077.png)

이 opkg update에서는 다운로드는 되었지만 후속하는 시그니쳐 파일 다운로드에 실패하였다. 그래서 검증에 실패하면서 Packages.gz 자체를 신뢰할 수 없는 파일로 처리하면서 제거해버린 것이다.

이는 지금 실습 과정 중 우회해야 하거나 그런 것은 아니기 때문에

- 그냥 /etc/opkg.conf 파일을 편집하여 저 option check_signature을 주석처리해버렸다.

![image.png](assets/image%2080.png)

- 다시 진행하여보니

![image.png](assets/image%2081.png)

초반에 제대로 …/base/Packages.gz가 다운로드 되고 아까처럼 Remove가 뜨지 않았다. 

- 드디어 opkg install pwned를 하니

![image.png](assets/image%2082.png)

다운로드에 성공하였다.

그리고 원래라면 /tmp/pwned.txt도 만들어져야 하지만.. 셸 스크립트를 짜는 과정에서

![image.png](assets/image%2083.png)

실수로 저 #!/bin/sh에서 /가 중간에 하나 빠졌다… 일단… 성공했다!! 와!! 

