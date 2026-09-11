---
title: SSL 인증서 자동갱신 실패와 DNS-01 전환
description: certbot 이 12시간마다 갱신을 시도하는데 만료 10일이 지나서야 접속 불가로 알았다. 발급 방식이 manual 이라 애초에 자동으로 갱신될 수 없는 상태였다.
date: 2026-09-11 21:00:00 +0900
categories: [Infrastructure, Deployment]
tags: [certbot, lets-encrypt, duckdns, nginx]
---

## 접속이 막힌 상태

개인 서버에 nginx 리버스 프록시를 두고 서비스 네 개를 서브도메인으로 나눠 쓰고 있다. 어느 날 브라우저가 사이트를 아예 열어주지 않았다.

```
net::ERR_CERT_DATE_INVALID
```

인증서가 만료됐다는 뜻이다. 보통 이 화면에는 "고급"을 눌러 그냥 들어가는 링크가 붙어 있는데 그게 없었다. nginx 설정에 넣어둔 HSTS 때문이다.

```nginx
add_header Strict-Transport-Security "max-age=63072000" always;
```

HSTS는 브라우저에 "이 도메인은 앞으로 HTTPS로만 접속하라"고 알려주는 헤더다. 한 번 받으면 `max-age` 동안 기억한다. 63072000초는 730일이니 2년이다. 그 사이에는 주소창에 `http://` 를 쳐도 브라우저가 요청을 보내기 전에 `https://` 로 바꿔서 보낸다.

원래 목적은 중간자 공격을 막는 것이다. 주소를 칠 때 대부분 프로토콜을 생략하고, 그러면 첫 요청 한 번이 평문으로 나간다. 그 요청을 가로채면 이후 세션을 통째로 들여다볼 수 있다. HSTS는 그 첫 요청 자체를 없앤다.

인증서 오류를 무시하지 못하게 하는 것도 같은 맥락이다. 경고를 클릭으로 통과할 수 있으면 가짜 인증서를 내민 공격자도 같이 통과한다. 그래서 HSTS가 걸린 도메인에서는 예외 버튼을 아예 빼버린다.

급할 때는 답답하지만 설계대로 동작한 셈이다. 덕분에 우회로 때우지 못하고 원인을 봐야 했다. 브라우저에 남은 기록만 지우고 싶으면 크롬은 `chrome://net-internals/#hsts` 에서 도메인을 삭제할 수 있는데, 인증서를 고치면 필요 없는 작업이다.

서버에서 인증서 상태를 찍어봤다.

```bash
docker exec certbot certbot certificates
```

```
Expiry Date: 2026-09-01 08:07:11+00:00 (INVALID: EXPIRED)
```

9월 1일에 만료됐는데 발견한 게 9월 11일이었다. 열흘을 모르고 지났다.

## 진단

### 443과 80의 응답이 달랐다

먼저 요청이 어디까지 도달하는지 봤다. 80으로 보냈더니 nginx가 아닌 것이 응답했다.

```
HTTP/1.1 403 Forbidden
server: envoy
```

`server: envoy` 라서 내 nginx가 아니다. 공인 IP 앞단에 사내 게이트웨이가 있고 거기서 끊고 있었다.

이걸 보고 "서비스가 통째로 죽었다"고 판단했는데 틀렸다. 80만 보고 내린 결론이었다. 443을 따로 찍어보니 달랐다.

```bash
curl -skI https://내도메인/
```

```
HTTP/2 302
server: nginx/1.29.4
```

443은 게이트웨이를 통과해 nginx까지 정상 도달하고 있었다. 즉 서버도 nginx도 멀쩡했고, 브라우저가 막힌 이유는 만료된 인증서 하나였다.

### certbot 로그가 지목한 곳

갱신은 컨테이너가 12시간마다 돌고 있었다. compose에 이렇게 잡혀 있다.

```yaml
certbot:
  image: certbot/certbot
  entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```

돌고 있는데 갱신이 안 됐다면 실패하고 있다는 뜻이다. 로그를 봤다.

```bash
docker logs --tail 80 certbot
```

```
Failed to renew certificate with error: The manual plugin is not working
The error was: PluginError('An authentication script must be provided with
--manual-auth-hook when using the manual plugin non-interactively.')
```

같은 에러가 스크롤이 끝날 때까지 반복되고 있었다.

## manual authenticator

renewal 설정 파일을 열어보니 원인이 그대로 적혀 있었다.

```bash
docker exec certbot cat /etc/letsencrypt/renewal/내도메인.conf
```

```ini
[renewalparams]
authenticator = manual
key_type = ecdsa
pref_challs = dns-01,
```

`authenticator = manual` 은 챌린지를 사람이 처리한다는 설정이다. certbot이 TXT 레코드 값을 화면에 띄우고 사용자가 DNS에 등록한 뒤 Enter를 치는 방식이다. 컨테이너의 `certbot renew` 는 입력을 받을 사람이 없으니 매번 같은 자리에서 멈춘다.

### 한 번도 갱신된 적이 없었다

Let's Encrypt 인증서는 90일짜리고 certbot은 만료 30일 전부터 갱신을 시도한다. 12시간 주기니까 30일이면 60번쯤 시도했고 전부 실패했다.

정확히 말하면 이번에 실패한 게 아니라 **발급 이후 단 한 번도 갱신된 적이 없다.** manual 로 발급한 순간부터 자동 갱신은 성립할 수 없는 상태였고, 90일이 지나서야 그게 드러났다.

### 80이 막혀 HTTP-01을 못 썼다

왜 manual 로 발급했는지는 앞의 403이 설명해준다.

certbot이 도메인 소유를 확인하는 방식은 크게 둘이다. HTTP-01은 `/.well-known/acme-challenge/` 경로에 파일을 두고 Let's Encrypt가 80 포트로 가져가게 한다. DNS-01은 `_acme-challenge` TXT 레코드를 등록하고 그걸 조회하게 한다.

nginx에는 HTTP-01 경로가 멀쩡히 잡혀 있었다.

```nginx
location /.well-known/acme-challenge/ {
    root /var/www/certbot;
}
```

설정해두고 쓰지 못하고 있었다. 게이트웨이가 80을 막고 있어서 Let's Encrypt가 파일을 가지러 와도 403을 받는다. 보안 정책이라 열 수 있는 것도 아니다.

그래서 처음 발급할 때 DNS-01을 손으로 처리했고, 그 설정이 renewal 파일에 남아 지금까지 온 것이다.

## DNS-01 자동화

80을 쓸 수 없으니 방향은 하나다. DNS-01을 유지하되 TXT 등록을 사람 대신 스크립트가 하게 만든다. certbot 에러 메시지가 요구한 것도 정확히 그거였다.

### 손으로 넣다 실패한 지점

먼저 수동으로 살려보려 했다. certbot을 대화형으로 띄우면 도메인마다 TXT 값을 하나씩 보여준다.

```bash
docker exec -it certbot certbot certonly --manual --preferred-challenges dns -d ...
```

여기서 실수했다. 값을 네 개 다 받아놓고 등록은 첫 번째만 한 채로 Enter를 계속 눌렀다.

```
Domain: 두번째도메인
Type:   unauthorized
Detail: Incorrect TXT record "LUi8-y3-..." found at _acme-challenge...
```

`Incorrect TXT record` 는 레코드가 없다는 게 아니라 **다른 값이 들어 있다**는 뜻이다. 지난번 발급 때 넣었던 옛 TXT가 그대로 남아 있었고, 새 값과 달라서 거절당했다. certbot은 마지막 Enter 이후에 네 도메인을 한꺼번에 검증하기 때문에, 각 Enter 전에 그 도메인 등록이 끝나 있어야 했다.

도메인이 넷이면 이 과정을 넷 반복하는데, 실패하면 Let's Encrypt 실패 카운트가 쌓인다. 손으로 반복할 일이 아니라고 판단하고 hook 으로 넘어갔다.

### hook 스크립트

`--manual-auth-hook` 은 챌린지마다 실행되는 스크립트를 받는다. certbot이 환경변수 두 개를 넘겨준다.

| 변수 | 값 |
|---|---|
| `CERTBOT_DOMAIN` | 지금 인증 중인 도메인 |
| `CERTBOT_VALIDATION` | TXT 에 넣어야 할 값 |

DNS는 DuckDNS를 쓰고 있다. 웹 화면에는 TXT 입력칸이 없고 API로만 설정된다.

```
https://www.duckdns.org/update?domains=<이름>&token=<토큰>&txt=<값>
```

certbot 이미지가 alpine 기반이라 curl이 있다고 보장할 수 없었다. certbot 자체가 파이썬이니 파이썬으로 썼다.

```python
#!/usr/bin/env python3
import os, time, urllib.request
d = os.environ["CERTBOT_DOMAIN"].replace(".duckdns.org", "")
v = os.environ["CERTBOT_VALIDATION"]
t = "<DuckDNS 토큰>"
u = f"https://www.duckdns.org/update?domains={d}&token={t}&txt={v}&verbose=true"
print(urllib.request.urlopen(u).read().decode())
time.sleep(30)
```

`sleep(30)` 이 필요하다. 등록하자마자 Let's Encrypt가 조회하면 아직 전파 전이라 실패한다.

스크립트는 `/etc/letsencrypt` 로 마운트되는 디렉토리에 뒀다. 컨테이너를 다시 만들어도 남고, renewal 설정이 가리키는 경로와 같은 볼륨이다. 토큰이 평문으로 들어가므로 권한은 700으로 했다.

발급은 입력 없이 끝났다.

```bash
docker exec certbot certbot certonly --manual --preferred-challenges dns \
  --manual-auth-hook /etc/letsencrypt/duckdns-hook.py \
  --cert-name <이름> -d ... --key-type ecdsa
```

```
Hook '--manual-auth-hook' for 도메인A ran with output: OK ... UPDATED
Hook '--manual-auth-hook' for 도메인B ran with output: OK ... UPDATED
Successfully received certificate.
This certificate expires on 2026-12-10.
```

renewal 파일에 hook 경로가 저장되면서 자동 갱신 조건이 채워졌다.

```ini
manual_auth_hook = /etc/letsencrypt/duckdns-hook.py
```

`authenticator` 는 여전히 `manual` 이다. manual 은 "certbot 바깥에서 챌린지를 처리한다"는 뜻이지 "사람이 한다"는 뜻이 아니고, hook이 붙으면 비대화형으로 돈다.

### 갱신이 도는 경로

certbot 컨테이너와 nginx 컨테이너는 서로 통신하지 않는다. 인증서 디렉토리를 같은 볼륨으로 공유하고 각자 자기 주기로 돈다.

```
certbot 컨테이너  12시간마다 certbot renew
  │
  ├─ 만료 30일 전인가 확인, 아니면 종료
  └─ 맞으면 도메인마다 hook 실행
       ├─ DuckDNS API 로 TXT 등록
       ├─ 30초 대기
       └─ Let's Encrypt 가 TXT 조회해서 검증
            └─ 새 인증서를 live/ 에 저장
                     │
             공유 볼륨 (certbot/conf)
                     │
nginx 컨테이너    6시간마다 nginx -s reload
```

nginx는 파일이 바뀐 걸 감지하지 않는다. reload 할 때 다시 읽을 뿐이다. compose 커맨드에 6시간 루프가 들어 있는 이유가 이것이다.

```yaml
command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"
```

갱신과 반영 사이에 최대 6시간이 벌어지지만 만료 30일 전에 갱신되니 문제가 되지 않는다.

바로 갱신을 다시 돌려 확인할 수는 없다. 이미 새 인증서라 certbot이 건너뛴다. 대신 스테이징 서버로 리허설을 돌리면 hook 실행까지 그대로 재현된다.

```bash
docker exec certbot certbot renew --dry-run
```

```
Hook '--manual-auth-hook' for 도메인A ran with output: OK ... UPDATED
Hook '--manual-auth-hook' for 도메인B ran with output: OK ... UPDATED
Congratulations, all simulated renewals succeeded
```

네 도메인 모두 hook이 돌고 검증까지 통과했다. 90일 뒤에 같은 경로로 갱신된다.

## 열흘을 몰랐던 쪽

기술적인 원인은 manual 설정 하나였고 hook 스크립트 열 줄로 끝났다. 정작 손을 못 댄 쪽은 따로 있다.

certbot은 12시간마다 실패하면서 그 사실을 로그에만 남겼다. 아무도 그 로그를 보지 않았고, 만료 열흘 뒤 브라우저가 막히고 나서야 알았다. 갱신을 자동화해도 그게 실패하는 순간을 알 방법이 없으면 같은 일이 반복된다.

Let's Encrypt는 계정에 이메일이 등록돼 있으면 만료 20일 전쯤 알림을 보낸다. 확인해볼 지점이다.

```bash
docker exec certbot certbot show_account
```
