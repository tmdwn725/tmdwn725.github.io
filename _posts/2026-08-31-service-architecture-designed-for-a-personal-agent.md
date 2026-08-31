---
title: 개인 에이전트를 올리려고 설계한 서비스 구조
description: 코드를 쓰기 전에 구조부터 정했다. 서버 한 대에 무엇을 어떻게 올릴지, 그리고 애플리케이션 넷이 프로바이더를 직접 부르지 않고 게이트웨이를 지나게 한 이유를 적어둔다.
date: 2026-08-31 20:00:00 +0900
categories: [Agent Platform, Gateway]
tags: [openai-api, tool-calling, multi-tenancy, fastapi]
---

## 설계 전에 정해진 조건

개인 비서를 만들려고 [GPU 박스를 샀고](/posts/why-dgx-spark/), [기성 에이전트 둘을 한 달쯤 굴려보고](/posts/hermes-agent-vs-openclaw/) 직접 만들기로 했다. 그러고 나서 코드를 열기 전에 구조부터 정했다. 이 글은 그때 그린 그림이다.

정해진 조건이 넷 있었다.

**모델을 부르는 애플리케이션이 하나가 아니다.** 개인 비서 말고도 워크플로우 빌더, 관리 콘솔, 학습용 앱이 같은 모델을 부른다. 나중에 더 붙을 것도 같았다.

**로컬과 클라우드를 같이 쓴다.** 개인 데이터를 만지는 대화는 집에 둔 DGX Spark 에서 돌리고, 그 밖에는 클라우드 모델을 쓸 생각이었다. 둘을 요청마다 갈아탈 수 있어야 했다.

**돈이 어디로 나가는지 봐야 한다.** 클라우드 모델은 토큰이 곧 비용이다. 어느 애플리케이션이 얼마를 썼는지 모르면 모델을 올릴지 내릴지 판단할 근거가 없다.

**서버는 한 대다.** 작은 서버에 컨테이너로 다 올린다. 구조를 크게 잡아도 돌릴 기계가 없다.

## 개인 에이전트 설계 구조

이 조건에서 그린 구조다.

<div style="overflow-x:auto;margin:1.6rem 0;">
<svg id="archdiag" viewBox="0 0 980 605" role="img" aria-label="사용자 기기, 서버 한 대, 모델 실행으로 나눈 구조도" style="width:100%;min-width:760px;max-width:980px;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,'Apple SD Gothic Neo','Noto Sans KR',sans-serif;color:inherit;">
  <defs>
    <marker id="ah" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path class="arw" d="M0,0 L10,5 L0,10 z"/>
    </marker>
    <style>
      #archdiag .bx  { fill: currentColor; fill-opacity: .05; stroke: currentColor; stroke-opacity: .36; }
      #archdiag .bxa { fill: #3b82f6; fill-opacity: .12; stroke: #3b82f6; stroke-opacity: .6; }
      #archdiag .bxo { fill: currentColor; fill-opacity: .02; stroke: currentColor; stroke-opacity: .4; stroke-dasharray: 5 4; }
      #archdiag .host{ fill: currentColor; fill-opacity: .022; stroke: currentColor; stroke-opacity: .4; }
      #archdiag .ttl { fill: currentColor; font-size: 14px; font-weight: 600; }
      #archdiag .sub { fill: currentColor; fill-opacity: .66; font-size: 11.5px; }
      #archdiag .lbl { fill: currentColor; fill-opacity: .5; font-size: 11.5px; letter-spacing: .09em; }
      #archdiag .edg { fill: none; stroke: currentColor; stroke-opacity: .45; stroke-width: 1.2; }
      #archdiag .edd { fill: none; stroke: currentColor; stroke-opacity: .4; stroke-width: 1.2; stroke-dasharray: 5 4; }
      #archdiag .arw { fill: currentColor; fill-opacity: .55; }
      html[data-mode="dark"] #archdiag .sub  { fill-opacity: .85; }
      html[data-mode="dark"] #archdiag .lbl  { fill-opacity: .7; }
      html[data-mode="dark"] #archdiag .bx   { stroke-opacity: .5; }
      html[data-mode="dark"] #archdiag .bxo  { stroke-opacity: .52; }
      html[data-mode="dark"] #archdiag .host { stroke-opacity: .55; }
      html[data-mode="dark"] #archdiag .edg  { stroke-opacity: .6; }
      html[data-mode="dark"] #archdiag .edd  { stroke-opacity: .55; }
      html[data-mode="dark"] #archdiag .arw  { fill-opacity: .7; }
      @media (prefers-color-scheme: dark) {
        html:not([data-mode="light"]) #archdiag .sub  { fill-opacity: .85; }
        html:not([data-mode="light"]) #archdiag .lbl  { fill-opacity: .7; }
        html:not([data-mode="light"]) #archdiag .bx   { stroke-opacity: .5; }
        html:not([data-mode="light"]) #archdiag .bxo  { stroke-opacity: .52; }
        html:not([data-mode="light"]) #archdiag .host { stroke-opacity: .55; }
        html:not([data-mode="light"]) #archdiag .edg  { stroke-opacity: .6; }
        html:not([data-mode="light"]) #archdiag .edd  { stroke-opacity: .55; }
        html:not([data-mode="light"]) #archdiag .arw  { fill-opacity: .7; }
      }
    </style>
  </defs>

  <text class="lbl" x="290" y="26">사용자 기기</text>
  <rect class="bx" x="290" y="40" width="310" height="48" rx="10"/>
  <text class="ttl" x="445" y="61" text-anchor="middle">Browser</text>
  <text class="sub" x="445" y="78" text-anchor="middle">각 서비스 UI</text>
  <rect class="bx" x="630" y="40" width="310" height="48" rx="10"/>
  <text class="ttl" x="785" y="61" text-anchor="middle">Mobile</text>
  <text class="sub" x="785" y="78" text-anchor="middle">PWA · Web Push</text>

  <path class="edg" d="M445,88 V152" marker-end="url(#ah)"/>
  <path class="edg" d="M785,88 V152" marker-end="url(#ah)"/>

  <rect class="host" x="16" y="112" width="948" height="326" rx="14"/>
  <text class="lbl" x="32" y="136">서버 한 대 · 전 서비스 컨테이너</text>

  <rect class="bx" x="290" y="158" width="650" height="44" rx="10"/>
  <text class="ttl" x="615" y="177" text-anchor="middle">Nginx</text>
  <text class="sub" x="615" y="193" text-anchor="middle">리버스 프록시 · TLS</text>

  <path class="edg" d="M366,202 V232" marker-end="url(#ah)"/>
  <path class="edg" d="M532,202 V232" marker-end="url(#ah)"/>
  <path class="edg" d="M698,202 V232" marker-end="url(#ah)"/>
  <path class="edg" d="M864,202 V232" marker-end="url(#ah)"/>

  <rect class="bx" x="290" y="238" width="152" height="54" rx="10"/>
  <text class="ttl" x="366" y="261" text-anchor="middle">Agent</text>
  <text class="sub" x="366" y="278" text-anchor="middle">개인 비서</text>
  <rect class="bx" x="456" y="238" width="152" height="54" rx="10"/>
  <text class="ttl" x="532" y="261" text-anchor="middle">Agent Builder</text>
  <text class="sub" x="532" y="278" text-anchor="middle">노코드 워크플로우</text>
  <rect class="bx" x="622" y="238" width="152" height="54" rx="10"/>
  <text class="ttl" x="698" y="261" text-anchor="middle">Admin Console</text>
  <text class="sub" x="698" y="278" text-anchor="middle">운영 · 사용량</text>
  <rect class="bx" x="788" y="238" width="152" height="54" rx="10"/>
  <text class="ttl" x="864" y="261" text-anchor="middle">Other Apps</text>
  <text class="sub" x="864" y="278" text-anchor="middle">학습 · 수집</text>

  <path class="edg" d="M366,292 V332" marker-end="url(#ah)"/>
  <path class="edg" d="M532,292 V332" marker-end="url(#ah)"/>
  <path class="edg" d="M698,292 V332" marker-end="url(#ah)"/>
  <path class="edg" d="M864,292 V332" marker-end="url(#ah)"/>

  <rect class="bxa" x="290" y="338" width="650" height="56" rx="10"/>
  <text class="ttl" x="615" y="362" text-anchor="middle">LLM Gateway</text>
  <text class="sub" x="615" y="380" text-anchor="middle">키 · 단가 · 사용량 · 워크스페이스 격리</text>

  <text class="lbl" x="40" y="162">공통 기반</text>
  <rect class="bx" x="40" y="172" width="210" height="52" rx="10"/>
  <text class="ttl" x="145" y="194" text-anchor="middle">Keycloak</text>
  <text class="sub" x="145" y="211" text-anchor="middle">인증 · 인가</text>
  <rect class="bx" x="40" y="234" width="210" height="52" rx="10"/>
  <text class="ttl" x="145" y="256" text-anchor="middle">Gitea</text>
  <text class="sub" x="145" y="273" text-anchor="middle">형상관리 · CI/CD · 저장소</text>
  <rect class="bx" x="40" y="296" width="210" height="52" rx="10"/>
  <text class="ttl" x="145" y="318" text-anchor="middle">PostgreSQL</text>
  <text class="sub" x="145" y="335" text-anchor="middle">스키마 분리 · pgvector</text>
  <rect class="bx" x="40" y="358" width="210" height="52" rx="10"/>
  <text class="ttl" x="145" y="380" text-anchor="middle">Redis</text>
  <text class="sub" x="145" y="397" text-anchor="middle">큐 · 스트림 버퍼</text>

  <path class="edd" d="M290,180 H270 V198 H254" marker-end="url(#ah)"/>
  <path class="edd" d="M270,198 V260 H254" marker-end="url(#ah)"/>

  <path class="edg" d="M430,394 V494" marker-end="url(#ah)"/>
  <path class="edg" d="M800,394 V494" marker-end="url(#ah)"/>

  <text class="lbl" x="290" y="488">모델 실행</text>
  <rect class="bx"  x="290" y="500" width="280" height="54" rx="10"/>
  <text class="ttl" x="430" y="523" text-anchor="middle">DGX Spark</text>
  <text class="sub" x="430" y="540" text-anchor="middle">vLLM · 로컬 모델</text>
  <rect class="bxo" x="660" y="500" width="280" height="54" rx="10"/>
  <text class="ttl" x="800" y="523" text-anchor="middle">Cloud Provider</text>
  <text class="sub" x="800" y="540" text-anchor="middle">외부 API</text>

  <rect class="bx"  x="290" y="580" width="22" height="14" rx="4"/>
  <text class="sub" x="320" y="591">내 서버</text>
  <rect class="bxo" x="410" y="580" width="22" height="14" rx="4"/>
  <text class="sub" x="440" y="591">외부 서비스</text>
</svg>
</div>

위에서 아래로 세 구역이고, 구역이 바뀌면 도는 자리가 바뀐다. 사용자 기기, 서비스가 전부 컨테이너로 뜨는 서버 한 대, 그리고 모델이 실제로 도는 곳이다.

가운데 파란 상자가 이 글의 절반이다. 애플리케이션에서 맨 아래 구역으로 바로 가는 선이 하나도 없다. 나머지 절반은 그 상자를 받치는 왼쪽 넷과 맨 위의 Nginx 다.

왼쪽 넷은 서비스 하나에 속하지 않고 전부가 같이 쓴다. 위의 서비스 전부와 선을 그으면 그림이 안 읽혀서, 같은 서버 안이라는 자리만 두고 선은 생략했다.

## 서버 한 대에 무엇을 올릴지

### 문을 Nginx 하나로 모은다

서비스마다 포트를 열어두면 방화벽 규칙과 인증서를 서비스 수만큼 관리하게 된다. 밖에서 들어오는 문을 하나로 두고 경로와 서브도메인으로 가르기로 했다. 인증서도 한 곳에서 끝난다.

애플리케이션 UI 는 경로로 갈리고, Keycloak 과 Gitea 는 자기 서브도메인을 받는다. 그림에서 왼쪽으로 빠지는 점선 둘이 그 경로다. 사람이 브라우저로 직접 여는 화면이 있는 쪽이라 애플리케이션을 거치지 않는다.

컨테이너로 올리는 것도 서버가 작기 때문이다. 컨테이너마다 메모리 상한을 따로 걸 수 있어야, 한 서비스가 메모리를 다 먹고 나머지를 눕히는 일을 막는다.

### 로그인과 권한은 Keycloak 한 곳으로

서비스마다 로그인 화면을 만들면 사용자 테이블이 서비스 수만큼 생긴다. 비밀번호 재설정, 세션 만료, 권한 회수를 그때마다 여러 군데 고치게 된다. 그래서 로그인을 한 곳으로 모으고 각 서비스는 받은 토큰만 검증하기로 했다.

권한도 같은 토큰에서 꺼낸다. 토큰 안의 롤을 읽어서 관리자 화면을 열어줄지 서비스마다 판단한다. 표준 프로토콜을 쓰기 때문에 나중에 서비스가 하나 더 붙어도 인증 코드를 새로 쓸 일이 없다.

### 에이전트의 메모리와 스킬은 Git 저장소에

에이전트가 쌓는 메모리와 스킬은 결국 텍스트 파일이다. 이걸 DB에 넣으면 변경 이력과 되돌리기를 직접 만들어야 한다. Git 이 이미 갖고 있는 것을 다시 만들 이유가 없었다.

사용자별 private 저장소로 두면 격리도 파일 경로가 아니라 저장소 권한으로 처리된다. 같은 서버에 Git 을 올린 김에 배포도 여기서 건다. 푸시가 곧 배포다. 저장소를 밖에 두지 않은 이유가 이것이다.

### DB는 하나, 스키마로 나누고 벡터도 같이 넣는다

서비스마다 DB를 띄우면 인스턴스 수만큼 메모리가 나간다. 한 인스턴스 안에서 서비스마다 스키마를 나눠 쓰기로 했다.

RAG 벡터도 별도 벡터 DB 없이 같은 DB에 pgvector 로 넣는다. 벡터 전용 DB를 하나 더 올리면 백업과 접속과 운영이 두 벌이 된다. 문서 수가 적을 때는 pgvector 가 그 값을 한다.

내주는 건 분명하다. 한 서비스가 커넥션을 다 먹으면 나머지가 같이 느려진다. 스키마로 나눈 건 논리적인 구분이지 자원 격리가 아니다.

### Redis 는 큐와 스트리밍 버퍼

예약해둔 시간에 에이전트가 먼저 말을 걸게 하려면 워커와 큐가 필요하다. 그 큐를 Redis 에 둔다.

하나 더 있다. 응답을 스트리밍으로 받는 중에 브라우저가 끊기면 그때까지 흘린 토큰이 사라진다. 다시 들어왔을 때 이어 보게 하려면 어딘가 잠깐 담아둬야 하는데, 짧게 살고 빨리 읽히면 되는 데이터라 DB가 아니라 Redis 가 맞다.

## 모델 호출은 게이트웨이를 지난다

에이전트 하나만 쓸 거였으면 이건 안 그렸다. 키를 환경변수에 넣고 프로바이더 SDK를 직접 부르면 끝이다. 부르는 쪽이 넷이 되고 그중 어느 쪽이 얼마를 쓰는지 알아야 하는 순간부터, 각자 키를 들고 있는 구조로는 답이 안 나온다. 개인 비서를 만들려던 일이 밑단을 하나 그리는 일로 번진 지점이 여기다.

### 모델 호출 경로

애플리케이션 코드에 프로바이더를 두지 않기로 했다. OpenAI 호환 클라이언트를 게이트웨이 주소로 들고, 요청에는 모델 이름만 싣는다. 프로바이더가 무엇인지, 그 프로바이더의 키가 무엇인지, 백만 토큰당 단가가 얼마인지는 게이트웨이만 안다.

이렇게 잡으면 모델을 바꾸는 일이 에이전트를 다시 배포하는 일이 아니라 게이트웨이에서 고르는 일이 된다. 로컬 모델도 프로바이더 하나로 등록해서 클라우드와 같은 자리에 세운다.

요청 하나가 지날 경로는 여섯 단계로 잡았다.

```text
애플리케이션             게이트웨이                        프로바이더
   │  model=...            │ 1. API 키 검증                    │
   ├──────────────────────▶│ 2. 워크스페이스 확인                │
   │                       │ 3. 조직 유도                       │
   │                       │ 4. 프로바이더 자격증명 조회          │
   │                       ├───────────────────────────────────▶│
   │◀──────────────────────┤ 6. 요청 로그 기록  ◀ 5. 실행 ──────┤
```

애플리케이션이 아는 건 1번에 쓰이는 키 하나뿐이다.

### 워크스페이스가 격리 단위

멀티테넌시 기준을 워크스페이스 하나로 잡았다. API 키가 워크스페이스에 매달리고, 자격증명·로그·예산이 전부 그 아래로 들어간다. 조직은 워크스페이스에서 역추적한다.

```text
조직 → 워크스페이스 → { API 키, 프로바이더 자격증명, 한도, 로그 }
```

키를 검증하면 워크스페이스와 조직과 키를 묶은 컨텍스트가 생기고, 그 뒤의 모든 조회와 기록이 이 컨텍스트를 달고 다니게 했다. 프로바이더 자격증명은 워크스페이스마다 따로 두고, 활성 자격증명은 워크스페이스·프로바이더당 하나만 남게 제약을 건다.

키 자체에도 제한을 매단다. 그 키가 부를 수 있는 모델 목록, 분당 요청 한도, 만료 시각이다. 남이 자기 에이전트를 올릴 때 받는 건 키 하나지만, 그 키에 이 셋이 같이 붙어 나가는 그림이다.

### 요청 하나가 남길 것

게이트웨이를 두는 이유의 절반이 이 로그다. 요청마다 워크스페이스, 키, 모델, 토큰 사용량, 비용, 지연, 성공 여부를 남긴다.

비용은 요청 시점의 단가로 계산해서 박아두기로 했다. 단가 테이블에 유효 기간을 두면 나중에 가격이 바뀌어도 과거 로그의 금액이 따라 움직이지 않는다. 단가를 모르는 모델은 비용을 0으로 떨어뜨린다. 직접 굴리는 로컬 모델이 여기 해당한다. 그래서 이 로그의 비용 합계는 외부에 나갈 돈이고, 전체 사용량은 토큰 수로 본다.

### 세션과 로그를 잇는 상관키

로그는 게이트웨이 쪽에, 대화 세션은 에이전트 쪽에 쌓인다. 서로 다른 데이터베이스라 나중에 붙일 키가 없다. 그래서 요청을 보낼 때 세션 번호와 몇 번째 왕복인지를 같이 실어 보내기로 했다.

왕복 번호까지 넣는 건 도구 호출 루프 때문이다. 한 번의 사용자 발화가 도구를 세 번 부르면 모델 호출은 네 번 일어난다. 세션 단위로만 묶으면 이 대화가 얼마 썼는지는 나와도 몇 번째 왕복에서 컨텍스트가 불었는지는 안 나온다.

## 내주기로 한 것

게이트웨이는 공짜로 얹히지 않는다. 그리면서 알고 내준 게 셋이다.

**요청이 지나는 자리가 하나 늘어난다.** 응답을 기다리고 스트리밍으로 흘려보내는 시간이 이 구간의 대부분인데, 로컬 모델은 응답이 몇 분씩 걸릴 수 있다. 그동안 게이트웨이가 DB 커넥션 같은 자원을 붙들고 있으면 그게 그대로 병목이 된다. 그래서 프로바이더로 요청을 넘기기 전에 필요한 걸 다 읽고 커넥션을 놓는 것을 규칙으로 잡았다. 비용 계산에 쓸 단가를 미리 받아두는 것도, 로그 기록을 요청 처리와 떼어놓는 것도 이 규칙에서 나온다.

**프로바이더 차이를 한 곳에서 흡수하게 된다.** 토큰 사용량 필드 이름부터 프로바이더 계열마다 다르다. 한 곳에 모으면 부르는 쪽이 편해지는 대신, 그 한 곳이 틀리면 전부 틀린다. 게다가 요청은 성공하니까 조용히 틀린다. 집계가 목적인 구조에서 집계만 비는 게 제일 위험한데, 값이 비었을 때 알려주는 장치는 이 설계에 아직 없다.

**모든 모델 호출이 한 곳을 지난다.** 게이트웨이가 멈추면 애플리케이션 넷의 LLM 기능이 같이 멈춘다. 다만 이 구조에는 그런 자리가 이미 여럿이다. Nginx 가 멈추면 아무 화면도 안 열리고, Keycloak 이 멈추면 로그인이 안 되고, PostgreSQL 이 멈추면 전부 선다. 서버가 한 대라 애초에 같이 눕는 그림이라, 게이트웨이가 그 목록에 하나 더 들어가는 것은 지금 받아들이기로 했다. 나눠 올릴 서버가 생기면 그때 다시 볼 자리다.
