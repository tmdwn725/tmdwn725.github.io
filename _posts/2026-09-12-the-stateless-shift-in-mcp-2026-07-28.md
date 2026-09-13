---
title: MCP 의 stateless 전환
description: MCP SDK 가 메이저 버전을 올린 걸 보고 내 에이전트도 따라가야 하나 싶어 스펙을 읽었다. 세션과 initialize handshake 가 통째로 사라졌고, 내 쪽은 이미 deprecated 된 transport 를 쓰고 있었다.
date: 2026-09-12 20:00:00 +0900
categories: [Agent Platform, Integration]
tags: [mcp, streamable-http, python-sdk, hermes]
---

## 오픈소스 에이전트들의 MCP SDK 버전

MCP SDK 가 메이저 버전을 올렸다. 알고 나서 든 생각은 하나였다. **내 에이전트도 따라가야 하나.**

알게 된 건 Hermes 의존성에서였다. 오픈소스 에이전트를 몇 개 받아놓고 업데이트가 올라오면 훑어보는데, 이번엔 여기가 걸렸다.

```toml
mcp = ["mcp==2.0.0", "httpx2==2.7.0", "starlette==1.3.1"]
```

`mcp` 는 공식 Python SDK 다. 내 에이전트도 같은 걸 쓰는데 1.27.2 에 머물러 있었다. 메이저 버전이 하나 벌어진 것이다.

다른 데는 어떤지 궁금해서 몇 개 더 열어봤다.

| | MCP SDK |
|---|---|
| Hermes Agent 0.21.2 | `mcp==2.0.0` |
| OpenClaw 2026.9.4 | `@modelcontextprotocol/sdk` 1.30.0 |
| LiteLLM | `>=1.25.0,<2.0.0` |
| 내 에이전트 | `mcp` 1.27.2 |

2026년 9월 13일에 각 프로젝트 저장소에서 확인한 값이다. PyPI 의 `mcp` 최신은 이미 2.2.0 이다.

Hermes 만 넘어갔다. OpenClaw 는 2.0 이라는 큰 릴리스를 내면서도 MCP SDK 는 v1 에 뒀고, LiteLLM 은 아예 `<2.0.0` 으로 막아놨다. [두 프로젝트를 비교한 글](/posts/hermes-agent-vs-openclaw/)을 쓸 때는 이 차이를 못 봤다.

셋 중 하나만 움직였다. 먼저 간 쪽이 급했거나 나머지가 미룰 이유가 있거나 둘 중 하나다. **내가 어느 쪽에 서야 할지 정하려면 스펙을 봐야 했다.**

## MCP 가 무엇인가

### 도구를 연결하는 방식을 표준으로

Anthropic 이 2024년 11월에 공개한 오픈 프로토콜이다. 모델을 외부 도구와 데이터에 연결하는 방식을 하나로 맞추자는 것이다.

그전에는 연결하는 쪽마다 규약을 새로 만들었다. 도구를 열 개 연결하면 연동 코드가 열 벌 나오고, 에이전트를 바꾸면 처음부터 다시였다.

MCP 는 그 사이에 JSON-RPC 규약을 하나 끼운다. 서버가 `tools/list` 로 자기 도구를 광고하고 클라이언트가 `tools/call` 로 부른다. **서버를 한 번 만들면 MCP 를 말하는 클라이언트는 어디든 연결된다.**

지금은 Anthropic 만의 것이 아니다. 스펙이 공개돼 있고 여러 곳이 각자 SDK 를 낸다.

### 클라이언트와 서버

이름이 헷갈리기 쉽다. **MCP 서버는 남이 운영하는 무언가가 아니라 도구 묶음 하나다.** 내 노트북에서 도는 프로세스도 MCP 서버다.

```
MCP 클라이언트   도구를 부르는 쪽      에이전트
MCP 서버        도구를 내주는 쪽      도구 묶음
```

에이전트 하나가 서버 여럿에 연결된다.

```
                  내 에이전트
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
    파일 도구       캘린더 도구     GitHub 도구     ← 각각이 MCP 서버
                       │              │
                       ▼              ▼
                  Google API     GitHub API      ← MCP 를 모른다
```

**맨 아랫줄이 중요하다.** Google API 도 GitHub API 도 MCP 를 전혀 모른다. 중간의 MCP 서버가 번역기다.

Microsoft Graph 를 쓰는 서버를 만든다면 이렇게 된다.

```
들어옴   tools/call { name: "send_mail", arguments: { to, subject, body } }
             ↓  내 MCP 서버가 변환
나감     POST https://graph.microsoft.com/v1.0/me/sendMail
         Authorization: Bearer ...
```

한 번 만들어두면 내 에이전트만 쓰는 게 아니다. **MCP 를 말하는 클라이언트는 다 붙는다.** 에이전트마다 Graph 연동 코드를 다시 쓰지 않아도 되는 게 MCP 를 두는 이유다.

### 서버를 누가 띄우나

세 가지 배치가 있고 지금은 앞의 둘이 훨씬 많다.

```
① 내가 만들고 내가 띄운다
   에이전트 ──stdio──▶ 내 MCP 서버 ──HTTPS──▶ 외부 API

② 공개된 걸 받아서 내가 띄운다
   에이전트 ──stdio──▶ 받아온 MCP 서버 ──HTTPS──▶ 외부 API

③ 그쪽이 운영하는 주소에 붙는다
   에이전트 ──HTTPS──▶ 그쪽 MCP 서버
```

①②는 내 기계 안에서 돌고 내 키를 쓴다. ③은 URL 만 알면 되는 대신 인증을 거치고 내 요청이 남의 서버를 지난다.

**stateless 전환이 제일 크게 걸리는 건 ③이다.** 수천 명이 붙는 서버에서 세션을 들고 있을 수가 없다. ①②도 여러 사람이 쓰게 되면 같은 문제를 만난다.

### 스펙 버전은 날짜로 적는다

보통 소프트웨어는 `2.0.0` 처럼 번호를 매긴다. MCP 스펙은 그런 번호가 없다. **개정한 날짜 문자열이 곧 버전이다.**

요청 헤더에 이렇게 들어간다.

```
MCP-Protocol-Version: 2026-07-28
```

헷갈리는 건 **SDK 는 보통 방식으로 번호를 매긴다**는 점이다. 앞에서 본 `mcp 2.0.0` 은 `2026-07-28` 리비전을 구현했다는 뜻이지 스펙이 2.0 이 됐다는 뜻이 아니다. 두 체계가 따로 논다.

리비전은 이렇게 이어져왔다.

| 리비전 | 그때 들어온 것 |
|---|---|
| 2024-11-05 | 첫 스펙. transport 는 stdio 와 HTTP+SSE |
| 2025-03-26 | Streamable HTTP 등장. HTTP+SSE 는 deprecated |
| 2025-06-18 | `MCP-Protocol-Version` 헤더 |
| 2025-11-25 | 직전 안정 리비전 |
| 2026-07-28 | stateless 전환 |


## transport 가 바뀌어온 순서

MCP 는 메시지를 어떻게 실어 나를지를 **transport** 로 떼어놨다. 여기가 제일 많이 바뀐 부분이다.

### stdio

로컬에서 서버를 자식 프로세스로 띄우고 표준 입출력으로 주고받는다. 첫 스펙부터 지금까지 그대로 있다.

연결이 단순하고 네트워크가 필요 없다. 대신 서버가 내 기계 안에 있어야 한다.

### HTTP+SSE

원격 서버를 연결하려고 첫 스펙에 같이 들어왔다. 엔드포인트가 둘이다.

```
GET  /sse        서버가 스트림을 열고 이쪽으로 메시지를 보낸다
POST /messages   클라이언트가 이쪽으로 요청을 보낸다
```

클라이언트가 먼저 GET 으로 스트림을 열면 서버가 첫 이벤트로 POST 주소를 알려준다. 그 뒤로 요청은 POST, 응답은 SSE 로 간다.

**문제는 스트림이 계속 열려 있어야 한다는 것이다.** 끊기면 처음부터 다시 열어야 하고, 서버는 그 스트림이 누구 것인지 기억하고 있어야 한다.

### Streamable HTTP

`2025-03-26` 에 나왔다. 엔드포인트를 하나로 합쳤다.

```
POST /mcp    요청 하나가 POST 하나
             응답은 JSON 한 덩어리이거나, 그 요청에 묶인 SSE 스트림
```

스트림을 미리 열어둘 필요가 없다. 요청을 보내면 서버가 그때 판단해서 한 번에 답하거나 스트림으로 흘린다.

HTTP+SSE 는 이때 deprecated 됐다. 표시만 된 채로 계속 동작했고, 이번 리비전에서 공식 등급이 매겨졌다.

## MCP 가 상태를 갖고 있던 구조

### 연결을 열고 시작하던 방식

기존 MCP 는 연결을 열면 먼저 handshake 를 했다. 클라이언트가 `initialize` 를 보내고, 서버가 자기 capability 를 답하고, 클라이언트가 `notifications/initialized` 로 끝났다고 알린다. 그 뒤부터 도구 목록을 받고 호출을 보낸다.

원격 연결에는 `Mcp-Session-Id` 헤더가 붙었다. 이 세션 안에서 서버는 상태를 들고 있을 수 있고, `tools/list` 결과가 연결마다 달라도 됐다.

### 처음엔 세션이 자연스러웠다

지금 보면 불편한 구조인데 첫 스펙에서는 합리적이었다. **MCP 가 로컬용으로 출발해서다.**

기본 transport 가 stdio 다. 서버를 자식 프로세스로 띄우고 표준 입출력으로 대화한다.

```
프로세스 하나 · 연결 하나
그 프로세스가 사는 동안이 곧 세션
```

**여기서는 상태가 공짜다.** 기억할 곳이 한 군데뿐이고 누가 누군지 헷갈릴 일이 없다. 오히려 세션을 안 쓰면 매번 같은 말을 반복하는 낭비가 된다.

구조도 LSP 와 같다. 에디터가 언어 서버를 자식으로 띄우고 `initialize` 로 시작하는 그 방식이다. 이미 잘 도는 설계를 가져왔다.

원격용 HTTP+SSE 도 그 모양을 그대로 옮겼다. **프로세스가 사는 동안이 세션이던 걸 스트림이 열려 있는 동안으로 바꿨을 뿐이다.**

### 세션이 있어야 되는 기능도 있었다

관성만은 아니었다. 코어 기능 셋이 세션을 필요로 했다.

| | 무엇 |
|---|---|
| sampling | 서버가 클라이언트에게 "모델 한 번 돌려줘" |
| elicitation | 서버가 사용자에게 되물음 |
| roots | 서버가 클라이언트에게 "어느 디렉토리 쓰냐" |

**셋 다 서버가 클라이언트를 거꾸로 부르는 것이다.** 열린 채널이 없으면 성립하지 않는다.

그래서 세션을 버리려면 이 셋을 같이 정리해야 했다. 이번 리비전에서 셋이 나란히 deprecated 된 게 그 결과다. 공짜로 버린 게 아니라 기능을 내주고 버렸다.

### 세션이 붙잡고 있던 것

**서버를 한 대 이상 두는 순간 걸린다.** `abc123` 을 기억하는 건 그 요청을 받았던 프로세스 하나뿐이다.

```
세션이 있을 때

  ① 클라이언트 ──▶ load balancer ──▶ 서버 A     abc123 을 서버 A 가 기억
  ② 클라이언트 ──▶ load balancer ──▶ 서버 B     abc123 ? 모른다   ✗

  → 같은 클라이언트는 계속 서버 A 로만 보내야 한다
```

이렇게 특정 클라이언트를 특정 서버에 묶는 걸 sticky 라우팅이라고 한다. **서버를 늘려도 부하가 고르게 안 퍼지고, 서버 A 가 죽으면 그 클라이언트들은 세션을 잃는다.**

serverless 나 edge 는 아예 못 쓴다. 요청 사이에 프로세스가 살아 있어야 하는데 그 전제가 없는 환경이다.

앞에 게이트웨이를 두면 한 겹 더 불편해진다.

```
세션 시절   무엇을 부르는지 알려면 본문을 열어야 한다

  POST /mcp
  Mcp-Session-Id: abc123
  { "method": "tools/call",
    "params": { "name": "send_mail", ... } }
           └ 여기까지 파싱해야 라우팅과 rate limit 이 된다
```

요청마다 JSON-RPC 본문을 파싱한다. **중개자가 할 일이 아닌데 하게 된다.**

## 2026-07-28 이 바꾼 것

리비전 이름이 날짜다. MCP 스펙에는 2.0 같은 번호가 없고 `2026-07-28` 이 그 자체로 버전이다. SDK 의 2.0.0 은 이 리비전을 구현한 결과물이지 스펙 이름이 아니다.

### 세션과 handshake 가 사라졌다

`Mcp-Session-Id` 헤더가 삭제됐다. `initialize` 와 `notifications/initialized` 도 없어졌다.

실제로 오가던 요청을 보면 차이가 분명하다. **전에는 도구 하나 부르려고 왕복이 세 번이었다.**

```
① POST /mcp
   {"method": "initialize",
    "params": {"protocolVersion": "2025-11-25",
               "capabilities": {...},
               "clientInfo": {"name": "MyAgent"}}}
   ← 200  Mcp-Session-Id: abc123

② POST /mcp   Mcp-Session-Id: abc123
   {"method": "notifications/initialized"}
   ← 202

③ POST /mcp   Mcp-Session-Id: abc123
   {"method": "tools/call", ...}
   ← 200
```

`abc123` 하나가 "아까 자기소개한 그 클라이언트" 를 가리킨다. 서버는 이게 누구였는지 기억하고 있어야 한다.

**지금은 한 번에 끝난다.**

```
POST /mcp
  MCP-Protocol-Version: 2026-07-28
  Mcp-Method: tools/call
  Mcp-Name: get_weather

  {"method": "tools/call",
   "params": {"name": "get_weather", "arguments": {...},
              "_meta": {
                "io.modelcontextprotocol/protocolVersion": "2026-07-28",
                "io.modelcontextprotocol/clientInfo": {"name": "MyAgent"},
                "io.modelcontextprotocol/clientCapabilities": {}
              }}}
  ← 200
```

`initialize` 에서 한 번 말하던 것들이 **매 요청의 `_meta` 안으로 자리를 옮겼다.**

| initialize 에서 하던 말 | 지금 어디에 |
|---|---|
| 나는 이 버전으로 말한다 | `_meta` 의 `protocolVersion` |
| 나는 이런 걸 할 수 있다 | `_meta` 의 `clientCapabilities` |
| 나는 누구다 | `_meta` 의 `clientInfo` |

없어진 게 아니라 옮겨간 것이다. 요청 하나가 커지는 대신 **서버가 아무것도 안 들고 있어도 된다.**

목록 엔드포인트는 연결마다 달라지지 않게 됐다. 같은 서버에 물으면 누가 묻든 같은 도구 목록이 나온다.

버전 확인용으로 `server/discover` 가 새로 생겼다. 서버는 이걸 반드시 구현해서 자기가 지원하는 리비전과 capability 를 광고한다.

그럼 여러 호출에 걸친 상태는 어디에 두느냐가 남는다. 스펙의 답은 서버가 핸들을 발급하고 그걸 **평범한 도구 인자로** 주고받으라는 것이다. 프로토콜이 몰래 들고 있던 걸 도구 시그니처로 끌어올렸다.

### elicitation 이 역방향 호출에서 재시도로

이게 제일 크게 바뀐 지점이다.

전에는 서버가 클라이언트를 거꾸로 부를 수 있었다. 도구를 실행하다가 사용자 입력이 필요하면 `elicitation/create` 를 보내고, 모델 호출이 필요하면 `sampling/createMessage` 를 보냈다. 열린 세션이 있으니 역방향 채널이 성립했다.

세션이 없으면 그 채널이 없다. 그래서 Multi Round-Trip Requests 로 대체됐다.

```
전       client ──call_tool──▶ server
         client ◀─elicitation/create── server
         client ──응답──▶ server
         client ◀─결과── server

2026-07-28
         client ──call_tool──▶ server
         client ◀─input_required(inputRequests)── server
         client ──call_tool(inputResponses 동봉)──▶ server
         client ◀─결과── server
```

서버는 추가 정보가 필요하면 `resultType: "input_required"` 를 그냥 리턴한다. 클라이언트가 답을 모아 원래 요청을 다시 보낸다. 재시도 사이에 서버가 뭔가 기억해야 하면 `requestState` 에 자기 식별자를 넣어둔다.

모든 결과에 `resultType` 이 필수가 됐다. 구버전 서버가 이 필드를 안 주면 클라이언트는 `complete` 로 친다.

### GET 스트림이 subscriptions/listen 으로

HTTP GET 엔드포인트와 `resources/subscribe` 가 없어지고 `subscriptions/listen` 하나로 합쳐졌다.

클라이언트가 받고 싶은 알림 종류를 골라 요청하면, 그 응답 스트림이 열린 채로 남아 해당 알림만 흘려보낸다.

```
toolsListChanged · promptsListChanged · resourcesListChanged · resourceSubscriptions
```

진행률이나 로그처럼 **요청 하나에 딸린 알림은 여기로 안 온다.** 그 요청의 응답 스트림으로 간다. 긴 구독과 짧은 요청을 섞지 않겠다는 것이다.

### 스트림 재개가 없어졌다

`Last-Event-ID` 와 SSE 이벤트 ID 가 Streamable HTTP 에서 빠졌다. 스트림이 끊기면 진행 중이던 요청은 날아가고, 클라이언트가 새 request ID 로 다시 보낸다.

재개를 프로토콜이 보장하던 걸 애플리케이션 쪽으로 내린 셈이다.

### 지워진 것과 남은 것

`ping`, `logging/setLevel`, `notifications/roots/list_changed` 는 제거됐다. 로그 레벨은 요청마다 `_meta` 로 넘긴다.

deprecated 로 표시만 된 것들은 최소 12개월 더 동작한다.

| 대상 | 대체 |
|---|---|
| Roots | 경로를 도구 인자나 resource URI 로 넘김 |
| Sampling | LLM 프로바이더 API 에 직접 붙음 |
| Logging | stderr 또는 OpenTelemetry |
| HTTP+SSE transport | Streamable HTTP |
| OAuth 동적 클라이언트 등록 | Client ID Metadata Documents |

Sampling 이 빠진 게 눈에 띈다. 서버가 클라이언트의 모델을 빌려 쓰는 구조였는데, 역방향 채널이 사라지면서 같이 정리됐다.

### 코어에서 확장으로 옮긴 것

실험 상태였던 Tasks 가 코어에서 빠져 공식 확장(`io.modelcontextprotocol/tasks`)이 됐다. 오래 걸리는 작업을 polling 으로 받아가는 기능인데, 이것도 세션 없이 돌게 다시 설계됐다.

그리고 `ClientCapabilities` 와 `ServerCapabilities` 에 `extensions` 필드가 생겼다. **코어 밖 기능을 선언할 자리가 마련된 것이다.**

코어는 얇게 두고 나머지는 확장으로 미는 방향이다.

그 밖에 Streamable HTTP POST 에 `Mcp-Method` 와 `Mcp-Name` 헤더가 필수가 됐다. 게이트웨이가 본문을 안 열고 라우팅하라는 것이다. 목록 결과에는 `ttlMs` 와 `cacheScope` 가 붙어서 클라이언트가 캐싱할 수 있게 됐고, `tools/list` 는 결정적 순서로 반환하는 게 권장된다. 도구 목록이 매번 같은 순서로 오면 프롬프트 캐시가 더 맞는다.

## SDK v2 에서 달라지는 코드

### 이름이 바뀐 것들

| v1 | v2 |
|---|---|
| `FastMCP` | `MCPServer` |
| `McpError` | `MCPError` |
| `tool.inputSchema` | `tool.input_schema` |
| `ClientSession` + transport 조립 | `Client` + `client.session()` |
| `streamablehttp_client` | 제거. `streamable_http_client()` |
| `timedelta` 타임아웃 | float 초 |
| `@server.list_tools()` 데코레이터 | `Server(on_list_tools=...)` |

필드명이 snake_case 로 바뀐 건 파이썬 쪽 접근만이고 와이어 JSON 은 그대로다. 직렬화할 때 `by_alias=True` 를 주면 예전 형태로 나간다.

`mcp.server.fastmcp` 의 FastMCP 와 별도 패키지인 `fastmcp` 는 다른 물건이다. 후자도 2.x 를 달고 있어서 검색하면 섞여 나온다.

### httpx2 는 나란히 설치된다

SDK 가 HTTP 스택을 `httpx` 에서 `httpx2` 로 옮겼다. 처음엔 이게 제일 부담스러워 보였다. 내 코드가 `httpx` 를 여기저기 쓰고 있어서 전부 갈아엎어야 하나 싶었다.

Hermes 쪽 주석에 답이 있었다.

```
Hermes' own httpx[socks]==0.28.1 in [dependencies] is unaffected
the two distributions install side by side under different module names
```

모듈명이 달라서 둘이 같이 설치된다. Hermes 도 자기 `httpx` 를 그대로 두고, SDK 에 넘길 클라이언트 객체를 만드는 파일에서만 `httpx2` 를 직접 import 한다. 전면 교체가 아니라 경계면만 갈아끼우는 일이었다.

## 내 에이전트는 어디쯤 있나

### 클라이언트가 세션을 전제로 짜여 있다

내 MCP 클라이언트는 860줄쯤 된다. 외부 서버마다 세션을 하나 열어 오래 붙들고, 연결할 때 `initialize` 를 부르고, 유휴 시간이 지나면 닫는다. 사용자별로 세션 키를 나눠서 풀에 담아둔다.

세션 풀 자체가 사라지는 개념 위에 서 있다. v2 로 옮기면 이 구조를 다시 짜게 된다.

다만 지금 당장 깨지지는 않는다. 업그레이드는 opt-in 이고, v2 클라이언트는 구버전 서버를 만나면 알아서 `initialize` 로 내려가고 v2 서버는 `server/discover` 와 레거시 `initialize` 를 둘 다 답한다.

### 서버 쪽 transport 가 이미 deprecated 였다

내 에이전트는 외부 서버를 소비하기만 하는 게 아니라 자기 도구를 MCP 서버로도 낸다. 그쪽을 보니 HTTP+SSE transport를 쓰고 있었다.

이건 2025-03-26 부터 deprecated 였다. Streamable HTTP 가 나온 게 그때다. 이번 리비전에서 공식 등급이 매겨지면서 확인했는데, 1년 반쯤 밀린 물건을 쓰고 있던 것이다.

이번 전환과 상관없이 갚아야 할 쪽이었고, v1 SDK 안에서도 옮길 수 있다.

### elicitation 자리가 비어 있었다

도구가 실행 도중 사용자에게 되묻는 채널을 만들어뒀다. [권한 계약을 정한 글](/posts/the-tool-permission-contract-i-settled-on-for-my-agent/)에서 쓴 승인 요청이 이 배관을 탄다. 승인, 되묻기, 클라이언트 쪽 실행, 그리고 MCP elicit 까지 네 종류를 받게 해뒀는데 마지막 하나는 자리만 잡아두고 구현을 안 했다.

미룬 게 이득이 됐다. 구형 `elicitation/create` 를 만들어뒀으면 지금 걷어내야 한다.

게다가 내가 만든 채널이 MRTR 과 모양이 거의 같다. 도구가 `input_required` 에 해당하는 신호를 올리고, 사용자 응답을 받아 원래 호출을 이어간다. 스펙이 세션을 버리면서 도달한 구조와, 내가 4워커 환경에서 턴을 재개하려고 만든 구조가 같은 곳으로 갔다. 상태를 어디에 둘 수 없을 때 나오는 답이 하나여서 그런 것 같다.

## 언제 옮길지

지금 전면 마이그레이션은 안 하기로 했다. 세 가지가 받쳐준다.

업그레이드가 opt-in 이라 서버가 새 리비전을 추가로 광고해도 예전 걸 요청하는 클라이언트에는 아무 변화가 없다. SDK 가 양쪽으로 fallback 한다. deprecated 된 기능은 최소 12개월 유지가 재단 정책으로 박혀 있다.

MCP 는 작년 12월에 Linux Foundation 산하 Agentic AI Foundation 으로 넘어갔다. Anthropic 이 기증하고 Block 과 OpenAI 가 함께 세웠다. 한 회사가 날짜를 정하던 때와 달라진 부분이고, 제안서 절차와 12개월 창이 생긴 게 그 결과다. 급하게 따라가지 않아도 되는 근거를 여기서 찾았다.

대신 두 가지를 정했다.

transport만 먼저 옮긴다. HTTP+SSE 를 Streamable HTTP 로 바꾸는 건 이번 리비전과 무관하게 밀려 있던 일이고 지금 SDK 로도 된다.

본 마이그레이션은 조건이 생길 때 한다. 붙어야 할 서버가 새 리비전 전용으로 나오거나, 그 서버가 MRTR 로만 되묻거나, deprecated 창이 만료에 가까워질 때다. 마지막 건 2027년 7월 이후다.

설계 쪽에는 지금부터 반영한다. 도구 계약을 새로 쓰고 있는데 세션을 전제로 쓰면 반년 뒤에 버린다. 여러 호출에 걸친 상태는 서버가 발급한 핸들을 인자로 주고받는 쪽으로 잡았고, 지원 범위는 SDK 버전이 아니라 리비전 날짜로 적기로 했다.

## 앞으로 어디로 가나

이번 리비전에 기능 수명 정책이 같이 들어왔다. 기능마다 **Active · Deprecated · Removed** 등급을 매기고, deprecated 로 표시되면 최소 12개월은 더 동작한다. 지금 그 상태인 Roots · Sampling · Logging · HTTP+SSE 는 2027년 하반기 전에는 안 없어진다.

방향은 세 갈래로 보인다.

```
코어는 얇게      Tasks 처럼 확장으로 밀어낸다
상태는 밖으로    프로토콜이 들고 있던 걸 도구 인자와 서버 핸들로
중개자 친화적    본문을 안 열고 헤더만 보고 라우팅하게
```

세 번째가 특히 내 쪽 얘기다. `Mcp-Method` 와 `Mcp-Name` 을 헤더로 올리라는 건, 앞에 게이트웨이나 load balancer 를 두는 걸 전제로 깔았다는 뜻이다. 목록 결과에 `ttlMs` 와 `cacheScope` 가 붙은 것도 중개자가 캐싱하라는 신호다.

**세션을 버린 이유가 여기 다 모인다.** 서버를 여러 대로 늘리고, serverless 에 올리고, 앞에 게이트웨이를 세우는 것. 원격 MCP 서버를 운영 환경에서 굴리려면 어차피 넘어야 했던 문턱이다.
