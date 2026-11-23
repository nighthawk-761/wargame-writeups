# 1. Hardcoded Credentials Compiled into Firmware

[← IoTGoat 목록](README.md)

binwalk -Mre 옵션을 통해 추출을 진행하였다.

![image.png](assets/image%205.png)

먼저 passwd 파일을 확인해 날먹이 가능한지 확인을 해보고

![image.png](assets/image%206.png)

비밀번호란이 x로 처리되어 있으니 /etc/shadow 파일을 확인한다.

![image.png](assets/image%207.png)

비밀번호가 MD5($1)로 암호화되어 있고, 이를 John the Ripper로 brute force를 진행할 것이다. 

John the Ripper 다운로드 

![image.png](assets/image%208.png)

rockyou.txt를 다운 받은 뒤, 진행해보았지만 실패하였다.

![image.png](assets/image%209.png)

그래서 그냥 seclists 파일을 다운 받아버렸다. 그중에서 mirai-botnet.txt를 활용할 것이다. 

hash 전용으로 보이는 파일도 보이긴 했는데, 찾아보니까 좀 기본 패스워드, 이런 느낌이라 무작위성 비번에는 약해서 건너뛰려고 한다.

![image.png](assets/image%2010.png)

![image.png](assets/image%2011.png)

그래서  medusa를 이용한 온라인 로그인형  brute force를 사용하려고 한다. 근데근데 진짜 인터넷 연결하는데 시간이 엄청 잡아먹히게 되었다. gpt 교수님 덕에 성공하였지만, 너무… 너무 많은 시간이 끌리게 되었다. 
(이제 가장 윗 내용이 해당내용이다.)

자 이제 다시 brute force의 시간이다.

![image.png](assets/image%2012.png)

/etc/passwd를 확인하고 , root와 iotgoatuser 유저가 있는 것을 볼 수 있다. 어차피 다른 거는 시스템계정이니 무시하자. 

그리고 원래라면 /etc/ssh/sshd.config 에서 PermitRootLogin No / Yes 옵션에 따라 root 원격접속을 차단할 수 있는데, 여기에는 따로 ssh 폴더도, 설정파일도 없었다. 그래서 root도 한 번 brute를 진행해보았는데, 

![image.png](assets/image%2013.png)

![image.png](assets/image%2014.png)

root 대상으로는 성공하지 못하였다.

이번엔 iotgoatuser를 대상으로 진행해보았는데, 

![image.png](assets/image%2015.png)

비밀번호가 7ujMko0vizxv 인 곳에서 찾은 걸을 확인할 수 있었다. 

![image.png](assets/image%2016.png)

iotgoatuser 아이디로는 접속에 성공하였다. 

root에 대해서는 일단 seclists에 동봉된 rockyou-75.txt 파일을 이용하여 brute를 진행해 보고 있다. 

한 40000번대를 진행하고 있을 때, 재부팅을 하게 되면서 나가리가 되었다… 근데 이정도 까지면 이 wordlist에는 없을 것 같다는 생각이 들기도 하곤 한다.

![image.png](assets/image%2017.png)

## 3번에서 복귀

자 순서가 좀 꼬이게 되었지만, 지금 web 계정을 찾아야 하는 상황이다. 

먼저 www 파일을 확인하였으나, /usr/bin/lua 파일을 참조한다고 나와있었다.
그런데 생각해보면 2번에서 찾은 파일들이 전부 /usr/bin/lua 에 있었다. 그리고 /luci까지 더 들어가고.
만약 이걸 먼저 찾았었다면 2번에서의 수색범위가 좁혀졌을 것 같다. 

![image.png](assets/image%2018.png)

```lua
user@lab:~/iotgoat/_IoTGoat-raspberry-pi2.img.extracted/squashfs-root/
usr/lib/lua/luci/controller/admin$ cat index.lua
-- Copyright 2008 Steven Barth <steven@midlink.org>
-- Licensed to the public under the Apache License 2.0.

module("luci.controller.admin.index", package.seeall)

function index()
        local root = node()
        if not root.target then
                root.target = alias("admin")
                root.index = true
        end

        local page   = node("admin")
        page.target  = firstchild()
        page.title   = _("Administration")
        page.order   = 10
        page.sysauth = "root"
        page.sysauth_authenticator = "htmlauth"
        page.ucidata = true
        page.index = true

        -- Empty services menu to be populated by addons
        entry({"admin", "services"}, firstchild(), _("Services"), 40).index = true

        entry({"admin", "logout"}, call("action_logout"), _("Logout"), 90)
end

function action_logout()
        local dsp = require "luci.dispatcher"
        local utl = require "luci.util"
        local sid = dsp.context.authsession

        if sid then
                utl.ubus("session", "destroy", { ubus_rpc_session = sid })

                luci.http.header("Set-Cookie", "sysauth=%s; expires=%s; path=%s/" %{
                        sid, 'Thu, 01 Jan 1970 01:00:00 GMT', dsp.build_url()
                })
        end

        luci.http.redirect(dsp.build_url())
end
```

admin 관련한 파일로 판단되어 gpt에게 분석 결과 root 계정으로 가능할 수도 있을 것 같다라는 판단이 나와 다시 한 번 root 계정에 대해 brute force를 해볼까 한다. 

그전에 혹시해서 sensordata.db 에 있나해서
sqlite3 sensordata.db 커맨드를 통해 진입해서 확인을 해보았으나, 유일한 sensors 테이블에는 아무 것도 없었다. 

![image.png](assets/image%2019.png)

![image.png](assets/image%2020.png)

→ 의미없는 짓이었다~ 

