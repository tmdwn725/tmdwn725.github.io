---
title: 셀프호스팅 Gitea 크립토 마이너 감염 대응과 재발 방지
description: 재시작하면 되고 며칠 뒤 다시 안 되던 git push. 브랜치 보호도 디스크도 아니었고, 컨테이너 안에서 채굴기가 CPU를 태우고 있었다.
date: 2026-08-22 20:00:00 +0900
categories: [Infrastructure, Security]
tags: [gitea, self-hosted, incident-response, docker]
---

## 재시작하면 되는 push 거부

개인 서버에 Gitea를 띄워놓고 여러 레포를 굴리고 있다. 어느 날부터 배포할 때마다 이게 났다.

```
remote: error: hook declined to update refs/heads/main
 ! [remote rejected] main -> main (pre-receive hook declined)
```

브랜치 보호 규칙이나 커밋 서명 문제인 줄 알았다. 그런데 컨테이너를 재시작하면 됐다.

```bash
ssh server 'docker restart gitea && sleep 15'
git push   # 성공
```

며칠 뒤에 또 안 됐다. 또 재시작하면 됐다.

이 패턴이 이상했다. 브랜치 보호였다면 재시작과 아무 상관이 없어야 한다. **뭔가가 시간이 지나면서 쌓여 push를 막고, 재시작으로 초기화되고 있었다.**

## 아닌 것부터 지웠다

브랜치 보호 규칙을 DB에서 직접 봤다.

```bash
docker exec postgres psql -U postgres -d gitea -c "
  SELECT r.name, b.branch_name, b.can_push
  FROM protected_branch b JOIN repository r ON r.id=b.repo_id;"
# (0 rows)
```

없었다. 다음은 디스크.

```bash
df -h /
# /dev/vda1  19G  17G  1.6G  92%
```

92%. 빠듯하긴 한데 지금 밀어 넣는 건 2KB짜리 커밋이다. 이게 직접 원인일 리는 없었다.

## CPU 357%

재시작하기 직전에 리소스를 봤다.

```bash
docker stats gitea --no-stream --format "cpu={{.CPUPerc}} mem={{.MemUsage}}"
# cpu=357.73% mem=2.562GiB / 7.755GiB
```

4코어 서버에서 3.5코어를 태우고 있었다.

Gitea 웹은 이런 CPU를 쓰지 않는다. 나 혼자 쓰는 서버고, 그 순간 push 말고는 아무 요청도 없었다. **컨테이너 안에서 Gitea가 아닌 뭔가가 돌고 있다는 뜻이었다.**

## docker top

```bash
docker top gitea
```

```
/tmp/javab                        294% CPU
/tmp/.rguard
/tmp/idle
find / -name esocket4 -exec rm ...
gitea web                          0.3%
```

`/tmp/javab`이 294%를 먹고 있었다. 채굴기였다. 정작 Gitea 본체는 0.3%였다.

나머지 프로세스들이 더 불쾌했다. `.rguard`는 경쟁 악성코드를 지우는 가드였고, `idle`은 본체가 죽으면 다시 띄우는 워치독이었다. `find`로 다른 감염 흔적을 청소하고 있었다. 자기가 이 서버를 차지하려고 남의 악성코드까지 지우고 있었던 것이다.

외부 연결도 확인했다.

```bash
sudo nsenter -t <PID> -n ss -tnp | grep javab
# ESTABLISHED  ...  141.x.x.x:8080
```

`/tmp/idle` 안에는 재다운로드 주소가 하드코딩돼 있었다. 지워도 다시 받아오게 되어 있었다.

## 재시작하면 되던 이유

여기서 풀렸다.

채굴기가 컨테이너의 `/tmp`에 있었다. 컨테이너 `/tmp`는 재시작하면 초기화된다. 그래서 `docker restart`를 하면 채굴기가 사라지고, CPU가 0%로 떨어지고, push의 pre-receive hook이 자원을 얻어 정상 실행됐다.

pre-receive hook은 별도 프로세스로 fork된다. CPU가 포화 상태면 이게 실패한다. 그 실패가 `pre-receive hook declined`로 나온 것이다.

침입 통로는 그대로였으니 며칠 뒤 다시 감염됐다. 나는 그때마다 재시작으로 증상만 지우고 있었다.

## 포트가 아니라 가입 페이지

처음엔 Gitea 웹(8091)과 SSH(2222) 포트가 열려서 뚫렸다고 생각했다. `docker inspect`에 `0.0.0.0`으로 바인딩돼 있었으니까.

그런데 `0.0.0.0` 바인딩은 컨테이너와 사설망 범위 얘기지 공인망 노출을 뜻하지 않는다. 실제로 밖에서 닿는지 노트북에서 찍어봤다.

```bash
nc -z -w 5 <공인IP> 8091   # 차단됨
nc -z -w 5 <공인IP> 2222   # 차단됨
nc -z -w 5 <공인IP> 443    # 열림
```

8091도 2222도 클라우드 보안그룹이 막고 있었다. 열린 건 nginx가 받는 443뿐이었다.

그럼 어디로 들어왔나. nginx 접근 로그에 그대로 찍혀 있었다.

```
31.x.x.x  "GET  /user/sign_up"              200
31.x.x.x  "POST /user/sign_up"              200
31.x.x.x  "POST /api/v1/user/repos"         500
31.x.x.x  "DELETE /api/v1/repos/..."        204
```

**가입하고, 로그인하고, API를 두드리고, 흔적을 지우고 나갔다.** 문을 부순 게 아니라 정문으로 걸어 들어왔다. 내가 무인 가입을 열어둔 채로 인터넷에 공개해두고 있었기 때문이다.

같은 패턴의 IP가 여러 개였다. 사람이 나를 노린 게 아니라 봇이 스캔하다 걸린 것이다.

## 계정 592개 중 590개가 봇

```bash
docker exec postgres psql -U postgres -d gitea -c "
  SELECT count(*) FILTER (WHERE lower_name ~ '^(testpoc|pwn|gitea_exp|user_)') AS bot,
         count(*) AS total FROM \"user\";"
#  bot 590 | total 592
```

진짜 사람은 내 계정 두 개뿐이었다. `testpoc18093`, `pwn09113b74`, `user_<hex>` 같은 이름이 몇 시간 간격으로 계속 생기고 있었고, 가장 최근 것이 그날 아침이었다. 조사하는 중에도 공격받고 있었다는 뜻이다.

## 지우기 전에 얼마나 뚫렸는지부터

채굴기를 당장 지우고 싶었는데 참았다. 지우면 증거가 사라진다.

읽기만 하는 쿼리로 범위부터 확인했다.

```sql
SELECT name FROM "user" WHERE is_admin=true;   -- 내 계정 2개뿐
SELECT count(*) FROM webhook;                  -- 0
SELECT t.name, u.name FROM access_token t JOIN "user" u ON u.id=t.uid;  -- 전부 내 것
SELECT name, is_private FROM repository;       -- 전부 private 유지
```

관리자 권한을 뺏기지 않았고, 백도어로 쓸 webhook도 없었고, 토큰도 안 털렸고, 레포도 계속 private이었다. 목적이 채굴이었지 데이터 탈취가 아니었다.

다만 RCE가 됐던 건 사실이다. 권한으로는 못 봤어도 파일시스템을 직접 읽었을 가능성까지 0이라고는 말 못 한다. 그게 걸리면 시크릿을 새로 발급하는 게 맞다.

## 막은 순서

**먼저 출혈을 멈췄다.** compose에 세 줄을 넣었다.

```yaml
- GITEA__service__DISABLE_REGISTRATION=true
- GITEA__openid__ENABLE_OPENID_SIGNUP=false
- GITEA__service__REQUIRE_SIGNIN_VIEW=true
```

설정을 넣고 나서 실제로 막혔는지 찍어봤다.

```bash
curl -s -o /dev/null -w "%{http_code}" -X POST https://<gitea>/user/sign_up \
  -d "user_name=zztest&email=zz@x.com&password=Ab1234!!&retype=Ab1234!!"
# 403
```

찍어보길 잘했다. 브라우저로 가입 페이지를 열면 화면이 그대로 뜬다. GET은 200이고 POST만 403이다. 화면만 봤으면 안 막혔다고 생각했을 것이다.

**그다음 봇 계정 590개를 지웠다.** DB에서 DELETE하면 연관 데이터가 깨진다. Gitea CLI로 지워야 레포와 star까지 같이 정리된다.

```bash
while read -r u; do
  docker exec gitea gitea admin user delete --username "$u" --purge
done < /tmp/bots.txt
```

개인 계정(`type=0`)만 대상으로 잡아서 조직 계정은 건드리지 않았다. 내 레포가 전부 조직 밑에 있어서 살았다. 목록을 뽑고 눈으로 확인한 다음 한 명으로 먼저 테스트하고 돌렸다.

**마지막으로 버전을 올렸다.** 1.22.6을 쓰고 있었다. 다섯 메이저나 밀린 버전이라 그동안 나온 보안 패치가 하나도 안 들어가 있었다. 가입을 막아도 취약점이 남으면 다른 경로로 또 뚫린다.

메이저 업그레이드는 DB 마이그레이션이 크니까 백업부터 했다.

```bash
docker exec postgres pg_dump -U postgres gitea | gzip > ~/gitea_db.sql.gz
tar czf ~/gitea_data.tar.gz data/gitea data/gitea-config
```

1.26.4로 올리고 마이그레이션 317번부터 330번까지 정상 완료를 로그로 확인했다.

## 남은 것

이 사고에서 제일 아팠던 건 침해 자체가 아니라, **몇 주 동안 재시작으로 증상만 지우고 있었다는 것**이다. "재시작하면 되는" 문제는 재시작으로 사라지는 무언가가 있다는 뜻이고, 그게 뭔지를 물었어야 했다.

그리고 컨테이너에 `cpus`나 `mem_limit`을 안 걸어둔 대가를 치렀다. 한도가 있었으면 채굴기가 아무리 돌아도 다른 프로세스가 쓸 몫이 남아서, 최소한 push는 됐을 것이다. 그러면 증상이 더 일찍 다른 모양으로 드러났을지도 모른다.

지금은 가입이 막혀 있고 버전도 최신이다. 그래도 인터넷에 공개된 셀프호스팅 서비스라는 사실은 그대로다. 다음은 신뢰하는 대역에서만 닿게 바꾸는 일이다.
