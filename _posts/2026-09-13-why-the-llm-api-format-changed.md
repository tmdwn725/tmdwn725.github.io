---
title: LLM API 형식이 바뀐 이유
description: Assistants API 가 끊긴 걸 보고 형식이 왜 바뀌었고 어떻게 바뀌었는지 파봤다. 벤더 넷의 요청·응답·오류 비교와 로컬 모델로 세 형식을 직접 불러본 결과.
date: 2026-09-13 14:00:00 +0900
categories: [Agent Platform, Gateway]
tags: [tool-calling, openai-api, responses-api, vllm]
---

## Assistants API 가 끊겼다

2026년 8월 26일에 Assistants API 가 종료됐다. 공지는 정확히 1년 전인 2025년 8월 26일에 나갔고, 그날로 `/v1/assistants` 와 `/v1/threads` 는 에러를 돌려준다. 유예는 없었다. OpenAI 의 [deprecations 문서](https://developers.openai.com/api/docs/deprecations)에 그렇게 적혀 있다.

Assistants API 는 2023년 11월에 나온 OpenAI 의 첫 에이전트용 API 다. 에이전트 설정을 Assistant 로 만들어 두고, 대화는 Thread 에 쌓고, 그 스레드 위에서 Run 을 돌리면 모델이 도구를 부르며 알아서 진행하고, 한 Run 안에서 무슨 일이 있었는지는 Run Step 으로 들여다보는 구조였다. 상태를 서버가 들고 있어서 클라이언트는 스레드 id 만 쥐고 있으면 됐다. 대체로 지목된 자리가 Responses API 와 Conversations API 다.

끊긴 게 그것만도 아니다. 11월 30일에는 재사용 프롬프트와 Agent Builder 가 닫히고, 그 뒤로도 모델 스냅샷과 레거시 오디오가 줄줄이 있다. 정작 Chat Completions 자체는 없앨 계획이 없다고 해 뒀다. 없어지는 건 채팅 형식 옆에 따로 세워뒀던 것들이다.

에이전트용으로 따로 있던 API 가 사라지고 그 자리가 형식 안으로 들어갔다는 뜻이다. 내 개인 비서도 같은 걸 하고 있으니 남 얘기가 아니었다. 새 모델이 에이전트 때문에 바뀌고 있다는 건 알고 있었는데, 정작 형식이 왜 바뀌었고 어떻게 바뀌었는지는 모르고 있었다. 그래서 파보기로 했다.

내 게이트웨이는 이미 절반쯤 그 변환을 하고 있었다. 입구로는 `/v1/chat/completions` 를 받고 Codex 로 나갈 때는 `/responses` 로 바꿔서 보낸다. 그 앞의 개인 비서 쪽 저장 구조를 열어보니 받아온 응답을 그대로 담고 있어서, 한 턴의 말과 도구 호출이 각각 다른 자리에 나뉘어 있었다.

#### OpenAI Chat Completions

```json
{
  "role": "assistant",
  "content": "서울 날씨를 볼게요\n그리고 부산도 볼게요",
  "tool_calls": [ {"id": "call_1"}, {"id": "call_2"} ]
}
```

한 턴에서 모델이 말하고 부르고 다시 말하고 또 불러도, 다시 읽으면 어느 말 다음에 어느 호출이 왔는지 알 방법이 없다.

내 저장 코드를 고치면 되는 줄 알았다. 그런데 받는 쪽을 따라가 보니 응답 자체에 그 정보가 없었다. Chat Completions 의 `choices[0].message` 가 `content`(문자열)와 `tool_calls`(배열)를 나란히 둔 칸막이 구조라서, 둘이 섞인 순서를 담을 자리가 애초에 없다.

그래서 이 형식이 왜 이렇게 생겼는지, 다른 벤더와 오픈소스 게이트웨이들은 어떻게 하고 있는지부터 찾아봤다.

## Chat Completions 의 문제

Chat Completions 는 채팅용 형식에 에이전트를 강제로 얹은 결과였다. 형식이 먼저 자리를 잡았고 에이전트 요구는 한참 뒤에 왔으니, 새 개념이 생길 때마다 이미 있는 자리에 밀어 넣는 수밖에 없었다.

### 이름 그대로 채팅용이었다

Chat Completions 는 2023년 3월에 나왔다. 그전의 completions 가 프롬프트 문자열 하나를 받던 것을, 사람과 모델이 번갈아 말하는 목록으로 바꾼 게 골자다. `messages` 가 배열인 이유도 여러 턴을 쌓기 위해서지 한 턴 안을 쪼개기 위해서가 아니었다. 한 턴은 한 사람이 한 번 말하는 것이고, 그게 채팅에서는 맞는 가정이다.

에이전트로 쓰기 시작한 건 그다음이다. 한 턴 안에서 모델이 생각하고 말하고 도구를 여러 번 부르고 그 결과를 받아 다시 말하는 건 채팅에 없던 일이라, 필요해질 때마다 필드가 하나씩 붙었다.

그 사정이 [Codex 쪽 공지](https://github.com/openai/codex/discussions/7782)에 한 줄로 적혀 있다. `chat/completions` 지원을 끊으면서 붙인 이유가, 그 프로토콜은 GPT-3.5 시절에 나온 것이라 지금의 에이전트 코딩과 추론 용도로 설계된 게 아니고 둘을 같이 받치느라 복잡도와 회귀가 늘었다는 것이다.

### functions 에서 tools 로

도구 호출은 2023년 6월에 `functions` 와 `function_call` 파라미터로 붙었고 한 번에 하나만 부를 수 있었다. 11월에 `tools` 와 `tool_choice` 로 바뀌는데, 바꾼 이유가 병렬 호출이었다. 여러 개를 동시에 부르려면 리스트가 필요했다.

그런데 기존 항목의 내부 구조는 그대로 두고 바깥만 감쌌다.

#### functions (2023년 6월)

```json
{
  "functions": [
    { "name": "get_weather", "description": "...", "parameters": {} }
  ]
}
```

#### tools (2023년 11월)

```json
{
  "tools": [
    {
      "type": "function",
      "function": { "name": "get_weather", "description": "...", "parameters": {} }
    }
  ]
}
```

`function` 래퍼 한 겹이 그때 생겨서 지금까지 남아 있다. 응답도 `message.function_call` 에서 `message.tool_calls[0].function` 으로 한 단계 깊어졌다.

### 옆에 따로 붙였다가 접은 Assistants

`tools` 와 Assistants API 가 같은 달에 나온 건 우연이 아니다. 한쪽은 채팅 형식에 필드를 붙여 도구를 넣는 길이었고, 다른 쪽은 형식은 손대지 않고 옆에 따로 층을 세우는 길이었다. 에이전트를 돌리려면 한 턴이 여러 단계로 쪼개져야 하는데, 그걸 `messages` 안에 넣는 대신 Run 과 Run Step 이라는 별도 개념으로 뺀 것이다.

3년을 그렇게 가보고 접은 것이 앞의 그 종료다. 옮겨간 자리를 보면 무엇이 문제였는지가 드러난다. Assistant 는 프롬프트로, Thread 는 Conversation 으로, Run 은 Response 로, Run Step 은 Item 으로 갔다. 별도 API 로 표현하던 걸 형식 안의 항목으로 끌어들인 것이다.

### role 로는 reasoning 을 못 담는다

덧붙이는 자리는 대개 role 이었다. 도구 결과가 필요해지자 `role: "tool"` 을 만들었고, 시스템 지시는 `messages` 안의 `role: "system"` 이었다. 채팅에서 물려받은 축이 role 하나뿐이니 새 개념도 거기에 걸 수밖에 없었다.

그 축이 안 통하는 게 사고 과정이다. reasoning 은 화자가 아니라 사건의 종류라서 role 로 표현이 안 된다. 서버가 실행하는 도구도, 컨텍스트 압축도 마찬가지다. role 은 누가 말하는지를 나타내는 축이지 무엇이 일어났는지를 나타내는 축이 아니다.

내 게이트웨이가 reasoning 을 받는 방식이 그 증거다. 요청에는 `extra_body` 라는 비표준 주머니에 넣고, 응답에서는 SDK 가 모르는 필드라 `model_extra` 에서 건져낸다. 표준에 자리가 없으니 우회할 수밖에 없었다.

### Anthropic 과 Google 의 출발점

칸막이형을 자기 형식으로 쓴 건 OpenAI 하나뿐이었다. 나머지 둘은 처음부터 배열이었다.

#### Anthropic Messages

```json
{
  "role": "assistant",
  "content": [
    { "type": "thinking", "thinking": "서울 날씨를 물었다...", "signature": "ErUBCkYIAR..." },
    { "type": "text", "text": "확인할게요" },
    { "type": "tool_use", "id": "toolu_01A", "name": "weather_forecast", "input": { "location": "서울" } }
  ]
}
```

role 을 user 와 assistant 둘로 고정하고 `content` 를 블록 배열로 뒀다. 생각도 말도 호출도 같은 배열에 순서대로 들어간다. 도구 결과는 새 role 을 만드는 대신 user 메시지 안의 `tool_result` 블록으로 넣었다.

#### Gemini generateContent

```json
{
  "role": "model",
  "parts": [
    { "text": "확인할게요" },
    { "functionCall": { "name": "weather_forecast", "args": { "location": "서울" } } }
  ]
}
```

`contents` 안에 `parts` 배열을 두고 role 은 user 와 model 을 쓴다. 키 이름이 `content`/`type` 대신 `parts`/`functionCall` 일 뿐 Claude 와 같은 구조다. 도구 결과도 같은 자리에 `functionResponse` 로 들어간다.

Gemini 가 OpenAI 형식처럼 보이는 건 호환 레이어를 따로 제공하기 때문이고, 그건 네이티브가 아니라 어댑터다.

## 프론티어 넷이 수렴한 구조

OpenAI 와 Anthropic 과 Google 과 xAI 가 각자 다른 시점에 다른 이유로 움직였는데, 도착한 자리가 같다.

### 같아진 여섯 가지

OpenAI 는 Responses API 로 옮겼다. `messages` 배열이 사라지고 `input` 과 `output` 에 타입 붙은 항목이 순서대로 들어간다. `message`, `reasoning`, `function_call`, `function_call_output` 이 전부 같은 층의 항목이다. 앞에서 본 Run Step 이 여기로 들어왔다. Chat Completions 자체는 없앨 계획이 없다고 해 뒀지만, 새 기능은 Responses 로 먼저 간다.

Google 은 Interactions API 를 2026년 6월에 GA 로 올리고 신규 프로젝트에 권장하고 있다. `input` 과 `steps` 를 쓰고 `previous_interaction_id` 로 서버가 상태를 잇는다. 이름까지 Responses 와 겹친다.

xAI 는 SDK 7 부터 Responses API 가 기본이고 Chat Completions 가 레거시 옵션으로 밀렸다.

네 벤더가 같아진 축이 여섯이다.

| 축 | 도착한 곳 |
|---|---|
| 항목 배열 | typed 항목이 시간 순서대로 |
| system 위치 | `instructions` · `system` · `system_instruction` 전부 최상위 |
| 사고 과정 | `reasoning` Item · `thinking` 블록 · thought step |
| 도구 정의 래퍼 | 없음 |
| 도구 종류 구분 | MCP·서버사이드 도구마다 별도 타입 |
| 서명 되돌려주기 | `encrypted_content` · `signature` · `thought_signature` |

베낀 게 아니라 같은 제약에 도달한 것으로 보인다. reasoning, 서버가 실행하는 도구, 도구 호출과 결과를 하나의 시간 순서로 표현하려면 타입 붙은 항목의 배열 말고 다른 답이 잘 없다. system 을 최상위로 뺀 것도 같다. system 은 대화 참여자가 아니라 설정이라는 걸 셋이 각자 정리했다.

### 도구 정의는 포장만 다르다

본체는 넷 다 같다. 이름과 설명과 JSON Schema. 다른 건 포장이다.

#### OpenAI Chat Completions

```json
{
  "type": "function",
  "function": {
    "name": "weather.forecast",
    "description": "Current weather plus forecast.",
    "parameters": {
      "type": "object",
      "properties": {
        "location": { "type": "string" },
        "days": { "type": "integer", "minimum": 1, "maximum": 7 }
      },
      "required": ["location"]
    }
  }
}
```

#### OpenAI Responses

```json
{
  "type": "function",
  "name": "weather.forecast",
  "description": "Current weather plus forecast.",
  "parameters": {
    "type": "object",
    "properties": {
      "location": { "type": "string" },
      "days": { "type": "integer", "minimum": 1, "maximum": 7 }
    },
    "required": ["location"],
    "additionalProperties": false
  },
  "strict": true
}
```

#### Anthropic Messages

```json
{
  "name": "weather_forecast",
  "description": "Current weather plus forecast.",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": { "type": "string" },
      "days": { "type": "integer", "minimum": 1, "maximum": 7 }
    },
    "required": ["location"]
  },
  "defer_loading": false
}
```

#### Gemini Interactions

```json
{
  "type": "function",
  "name": "weather.forecast",
  "description": "Current weather plus forecast.",
  "parameters": {
    "type": "object",
    "properties": {
      "location": { "type": "string" },
      "days": { "type": "integer" }
    },
    "required": ["location"]
  }
}
```

Chat Completions 만 `type` 을 밖에 두고 본체를 `function` 안에 넣는다. 나머지 셋은 `type` 이 본체와 같은 층에 있고 래퍼가 없다. 앞에서 본 2023년의 그 화석이 여기 남아 있는 것이다.

Claude 만 키 이름이 `input_schema` 이고, 이름에 점을 못 쓴다. `^[a-zA-Z0-9_-]{1,64}$` 라서 내 `weather.forecast` 는 `weather_forecast` 가 된다. 내보낼 때 바꾸고 받을 때 되돌리는 일이 어댑터에 하나 더 생긴다는 뜻이다.

### 같은 호출을 네 형식이 돌려주는 모양

"서울 날씨 알려줘" 에 모델이 도구를 부른 응답이다.

#### OpenAI Chat Completions

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "weather__forecast",
              "arguments": "{\"location\":\"서울\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ]
}
```

`choices` 배열을 열고 `message` 를 꺼내야 내용이 나온다. 사고 과정을 담을 자리는 없다.

#### OpenAI Responses

```json
{
  "id": "resp_...",
  "object": "response",
  "output": [
    { "type": "reasoning", "id": "rs_1", "summary": [] },
    {
      "type": "function_call",
      "id": "fc_12345xyz",
      "call_id": "call_12345xyz",
      "name": "weather.forecast",
      "arguments": "{\"location\":\"서울\"}"
    }
  ]
}
```

`output` 이 곧 항목 배열이다. `reasoning` 과 `function_call` 이 같은 층에 순서대로 있다.

#### Anthropic Messages

```json
{
  "id": "msg_...",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "thinking",
      "thinking": "서울 날씨를 물었다...",
      "signature": "ErUBCkYIAR..."
    },
    { "type": "text", "text": "확인할게요" },
    {
      "type": "tool_use",
      "id": "toolu_01A09q90qw",
      "name": "weather_forecast",
      "input": { "location": "서울" }
    }
  ],
  "stop_reason": "tool_use"
}
```

메시지 한 겹 안에 블록 배열이 있다. 인자가 문자열이 아니라 객체다.

#### Gemini Interactions

```json
{
  "steps": [
    {
      "type": "function_call",
      "id": "call_id_123",
      "name": "weather.forecast",
      "arguments": { "location": "서울" }
    }
  ]
}
```

이름이 `output` 대신 `steps` 일 뿐 Responses 와 같은 모양이다. 인자는 Claude 처럼 객체다.

넷을 겹쳐 보면 Chat Completions 만 두 겹을 열어야 내용이 나오고, 나머지 셋은 배열 하나에 타입 붙은 항목이 순서대로 들어 있다.

### 아직 갈리는 두 가지

구조는 모였는데 데이터 표현은 갈린다.

호출 인자가 Claude 와 Gemini 는 객체인데 OpenAI 는 Responses 로 넘어가면서도 문자열로 뒀다. 실패 표시도 갈린다. Responses 에는 실패를 나타내는 필드가 아예 없어서 결과 문자열에 그냥 쓴다.

#### OpenAI Responses

```json
{
  "type": "function_call_output",
  "call_id": "call_123",
  "output": "Error: invalid date, must be in the future."
}
```

#### Anthropic Messages

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_...",
  "content": [{ "type": "text", "text": "Invalid date: must be in the future." }],
  "is_error": true
}
```

둘 다 실패 내용을 모델에게 텍스트로 돌려준다는 점은 같다. 다른 건 호출한 쪽이 성공과 실패를 구분할 수 있느냐다. Responses 로 받으면 `output` 문자열을 읽어 판단하는 수밖에 없고, 그러면 실패율 집계 같은 걸 문자열 매칭으로 하게 된다.

이쪽을 안 바꾼 이유는 짐작이 간다. 구조는 새 엔드포인트를 만들면서 바꿀 수 있지만, `arguments` 를 객체로 바꾸거나 `is_error` 를 추가하면 기존 파싱 코드가 조용히 깨진다.

인자가 문자열이라 치르는 비용은 내 코드에도 남아 있다. 모델이 JSON 텍스트를 직접 써내다 보니 닫는 중괄호가 빠지거나, 코드펜스로 감싸거나, Python 의 `None` 을 섞어 보낸다. 그걸 흡수하는 파일이 따로 있다. 객체로 받았으면 없어도 될 파일이다.

### 로컬 모델 한 대로 세 형식을 불러봤다

기준 형식을 정하기 전에 실제로 어떻게 오는지 보고 싶었다. 로컬에서 굴리는 35B 모델 한 대가 세 엔드포인트를 전부 노출하고 있어서 같은 질문을 세 번 보냈다. 도구는 `get_weather` 하나만 붙였다.

| 엔드포인트 | 순서 | 사고 과정 | 인자 | 출력 토큰 |
|---|---|---|---|---|
| `/v1/responses` | 보존 | `reasoning` Item | 문자열 | 103 |
| `/v1/messages` | 보존 | `thinking` 블록 + `signature` | 객체 | 294 |
| `/v1/chat/completions` | 없음 | `message.reasoning` 비표준 필드 | 문자열 | 65 |

`/v1/messages` 가 있다는 걸 이때 알았다. 서빙 엔진이 Anthropic 형식 엔드포인트도 제공하고 `signature` 까지 만들어 준다. 같은 모델을 Claude 형식으로 부를 수 있다는 뜻이고, Anthropic 어댑터를 API 키 없이 개발할 수 있다는 뜻이기도 하다.

예상 못 한 건 따로 있었다. 같은 질문인데 엔드포인트마다 모델이 다른 언어로 생각했다. Chat Completions 로는 한국어로, 나머지 둘은 영어로 사고했다. 출력 토큰은 65 와 294 로 4.5배 벌어졌다. 엔드포인트별로 채팅 템플릿이 다르게 적용되기 때문으로 보인다.

옮길 때 실제 워크로드로 토큰을 다시 재기로 했다. 지금 숫자는 도구 하나짜리 질문 한 번이라 근거로 쓰기엔 약하다.

## 내 쪽을 어떻게 바꿀지

### canonical 을 메시지와 블록으로

여러 형식이 오가는 시스템에서는 내부적으로 하나로 통일해 쓰는 표현이 필요하다. 그걸 canonical 이라고 부른다. 외국어 문서가 여러 언어로 들어와도 사내에서는 한국어로 보관하고 나갈 때만 번역하는 것과 같고, 그 한국어에 해당하는 자리다.

풍부한 형식을 빈약한 쪽으로 접는 건 되는데 반대는 안 된다. 없는 정보를 만들어낼 수 없어서다. canonical 을 가장 풍부한 형식에 맞춰야 하는 이유가 이거다.

내가 고른 건 메시지 안에 블록 배열을 두는 구조다. Responses 의 평면 항목 배열도 후보였는데, 그쪽은 메시지 경계가 없다. `reasoning` 과 `function_call` 이 어느 턴에 속하는지가 데이터가 아니라 순서 규칙으로만 정해진다. Claude 나 Gemini 로 내보낼 때 그 경계를 다시 추론해야 하고, 어댑터마다 같은 추론이 반복된다. 메시지 경계를 가진 쪽을 펴서 평면으로 만드는 건 정보를 버리는 것이고, 평면을 다시 묶는 건 추측이다.

개인 비서의 저장 구조를 먼저 이 형태로 바꿨다. `content` 컬럼이 이미 JSONB 여서 스키마 변경 없이 문자열 대신 배열을 넣을 수 있었다.

#### 메시지와 블록

```json
{
  "role": "assistant",
  "content": [
    {"type": "thinking", "text": "두 도시를 봐야겠다"},
    {"type": "text",     "text": "서울 볼게요"},
    {"type": "tool_use", "id": "c1", "name": "weather.forecast", "input": {"location": "서울"}},
    {"type": "text",     "text": "부산도 볼게요"},
    {"type": "tool_use", "id": "c2", "name": "weather.forecast", "input": {"location": "부산"}}
  ]
}
```

바꾸면서 딸려 나온 것도 있다. 그동안 `content` 에 `reasoning` 과 `finish_reason` 과 `stalled` 를 같이 얹고 있었다. 담을 칸이 없어서 한 필드에 쌓은 건데, 주석에도 piggyback 이라고 적혀 있었다. 모델에게 되돌려 보내는 것은 블록으로, 관측용은 컬럼으로 갈랐다.

### 입구는 둘, 출구는 프로바이더마다

게이트웨이 입구는 `/v1/responses` 를 기본으로 두고 `/v1/chat/completions` 도 열어 두기로 했다. 다른 서비스가 이미 후자를 쓰고 있어서 교체가 아니라 병존이다.

Responses 를 고른 건 표현력 때문만은 아니다. Claude 형식이 객체 인자와 `is_error` 를 갖고 있어서 표현력은 더 낫다. 다만 Anthropic 말고는 받아주는 곳이 없다. Responses 는 OpenAI 와 로컬 서빙 엔진들이 받고 Gemini 신규도 거의 같은 모양이다. 풍부하면서 가장 널리 받아주는 자리가 거기 하나였다.

출구는 프로바이더마다 다르다. Claude 로는 거의 그대로 나가고, 로컬 모델로는 두 형식 중 고를 수 있고, OpenAI 로는 메시지 껍질을 벗겨 펴고, Chat Completions 만 받는 모델로는 접는다. 접기는 이 지점에서만 한다. 한 번 접히면 다시 펼 수 없으니 가능한 한 바깥으로 미뤘다.

프로바이더별 차이는 코드 분기가 아니라 선언으로 두기로 했다. 순서를 주는지, 사고 과정을 어떤 모양으로 주는지, 이름에 점을 쓸 수 있는지, 스키마의 어떤 키를 버리는지 같은 것들이다. 참고한 건 LiteLLM 인데, 2,691개 모델의 지원 여부를 JSON 데이터로 두고 변환만 코드로 두고 있었다. 조건문을 코드 곳곳에 넣으면 어떻게 되는지는 다른 오픈소스 에이전트에서 봤다. URL 판정과 모델명 판정과 프로바이더 예외가 한 파일에 섞여 있었다.

### 아직 확인 못 한 것

중국 모델들이 Responses 를 지원하는지 아직 확인 못 했다. 미지원이면 그 경로는 접힌 채로 돌고, 그 사실을 선언에 적어두면 된다.

블록으로 바꿨다고 순서가 바로 생기는 것도 아니다. 그릇을 먼저 만든 것뿐이고, 채우려면 게이트웨이 출구부터 Responses 로 옮겨야 한다. 저장 구조는 그때 손대지 않아도 된다는 게 이 순서로 한 이유다.
