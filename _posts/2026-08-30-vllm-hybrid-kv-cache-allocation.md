---
title: vLLM 로그의 KV cache size 는 용량이 아니다
description: 기동 로그가 찍는 토큰 수는 블록을 그룹 수로 나눈 값이다. hybrid 모델에서는 실제로 받을 수 있는 양을 3배 넘게 낮게 말한다. 레이어를 그룹으로 묶고 페이지를 맞춰 블록을 나눠주는 과정을 vLLM v0.17.0 소스로 확인했다.
date: 2026-08-30 10:00:00 +0900
categories: [Infrastructure, Inference]
tags: [vllm, kv-cache, mamba, hybrid-attention]
mermaid: true
---

> LLM 추론 개념을 다시 정리한 기록.

## 로그 두 줄이 서로 안 맞는다

vLLM 기동 로그에 용량을 말해주는 줄이 둘 있다.

```
GPU KV cache size: 159,296 tokens
Maximum concurrency for 65,536 tokens per request: 8.00x
```

첫 줄대로면 컨텍스트 65,536 짜리 요청은 두 개하고 조금 더 받는다.

```
159,296 ÷ 65,536 = 2.43
```

그런데 둘째 줄은 여덟 개라고 한다. **같은 로그, 같은 메모리 풀 얘기인데 3.3배 어긋난다.**

내 서버 모델이 hybrid attention 이라 그렇다. 레이어 40개 중 10개만 full attention 이고 나머지 30개는 linear attention 이다. **첫 줄은 이 구조를 반영하지 못한다.**

[앞 글](/posts/hybrid-attention-kv-cache-size/)에서 이 모델의 토큰당 KV 크기를 계산했더니 숫자가 둘로 갈렸다. 공식대로 세면 40 KiB 인데, 토큰마다 K·V 를 쌓는 레이어만 세면 10 KiB 다. 둘 다 측정값과 맞는다. **그 차이도 여기서 풀린다.**

왜 그런지 [vLLM v0.17.0](https://github.com/vllm-project/vllm/tree/v0.17.0) 소스로 확인했다. 결론부터 적으면 이렇다.

```
GPU KV cache size    블록 수를 그룹 수로 나눠 토큰으로 환산한 값
                     그룹이 하나인 보통 모델에서만 실제 용량과 같다

Maximum concurrency  요청 하나가 실제로 잡는 블록으로 계산한 값
                     hybrid 에서는 이쪽을 봐야 한다
```

아래는 그 숫자들이 어디서 나오는지다. 내 서버 값으로 따라간다.

```
레이어 40 · KV 헤드 2 · 헤드 차원 256 · fp8 · 블록 2,096 토큰 · KV 6.08 GiB
```

## 레이어를 그룹으로 묶는다

vLLM 은 레이어를 하나씩 관리하지 않는다. **종류가 같은 것끼리 묶어 그룹을 만들고, 블록은 그룹 단위로 빌려준다.**

그룹 수는 레이어 종류 수가 아니라 **반복되는 패턴의 길이**다. `kv_cache_utils.py` 의 docstring 이 예시를 든다.

> full attention 10개와 sliding window 20개를 가진 모델은 (1 × full, 2 × sw) 패턴이 10번 반복되는 것으로 볼 수 있다. 그래서 3개의 kv_cache_group 으로 묶고, 각 그룹이 10개 레이어를 담당한다.

내 모델은 `linear × 3 + full × 1` 이 10번 반복된다.

<pre class="mermaid">
graph LR
  subgraph P["패턴 · 4 레이어"]
    direction TB
    L1["linear"]
    L2["linear"]
    L3["linear"]
    F["full attention"]
  end
  P -->|"× 10 반복"| G["그룹 4개<br/>각 10 레이어"]
</pre>

```
그룹 1    full attention   10 레이어
그룹 2    linear           10 레이어
그룹 3    linear           10 레이어
그룹 4    linear           10 레이어
```

**linear 30개가 한 덩어리가 아니라 셋으로 쪼개진다.** 그룹마다 레이어 수를 같게 맞추기 때문이다. 그룹이 3개가 아니라 4개라는 게 뒤의 숫자 두 개를 다 바꾼다.

## 페이지 크기를 강제로 맞춘다

로그가 linear attention 을 `mamba` 라고 부르는 데는 이유가 있다. `Mamba` 는 앞의 토큰을 전부 들고 있는 대신 고정 크기 상태 하나에 압축해 토큰마다 갱신하는 SSM(State Space Model) 계열 아키텍처다. 이 모델의 linear attention 은 정확히는 [Gated DeltaNet](https://sebastianraschka.com/llm-architecture-gallery/hybrid-attention/) 이지만, **고정 상태를 들고 간다는 성질이 같아서 vLLM 은 이 부류를 통틀어 mamba 로 부르고 같은 캐시 기계로 관리한다.**

그룹을 나눴어도 블록 하나가 차지하는 크기가 종류마다 다르면 한 풀에서 못 꺼내 쓴다. **그래서 vLLM 은 크기를 강제로 같게 만든다.**

먼저 어텐션 쪽 블록 크기를 mamba 상태가 들어갈 만큼 키운다.

```python
attn_block_size = kernel_block_alignment_size * cdiv(
    mamba_page_size, kernel_block_alignment_size * attn_page_size_1_token
)
```

`kernel_block_alignment_size` 는 16이다. 로그의 2,096 은 이 값의 배수다.

```
2,096 ÷ 16 = 131
```

보통 모델의 블록이 16 토큰인데 131배가 됐다. 기동할 때 이 줄이 찍힌다.

```
Setting attention block size to 2096 tokens
  to ensure that attention page size is >= mamba page size
```

그러고도 남으면 이번엔 반대로 mamba 쪽을 채운다.

```python
# pad mamba page size to exactly match attention
cache_config.mamba_page_size_padded = attn_page_size
```

**남는 공간은 버린다.** 얼마나 버렸는지도 로그로 찍는다.

```
Padding mamba page size by %.2f%% to ensure
that mamba page size and attention page size are exactly equal.
```

여기까지 오면 레이어 종류와 무관하게 **블록 하나 · 레이어 하나가 잡는 자리가 전부 같다.**

```
2,096 토큰 × 2(K,V) × 2(KV 헤드) × 256 × 1바이트 = 2.05 MiB
```

hybrid 를 하나의 할당자로 다루려고 치르는 비용이 이 패딩이다.

## 블록을 몇 개 만드나

```python
num_blocks = int(available_memory // page_size // num_layers)
```

`num_layers` 자리에 들어가는 건 전체 40이 아니라 **그룹당 레이어 수**다.

```python
group_size = max(len(group.layer_names) for group in kv_cache_groups)
num_blocks = get_num_blocks(vllm_config, group_size, available_memory, page_size)
```

`group_size` 는 10이다.

```
6.08 GiB ÷ 2.05 MiB ÷ 10 = 304 블록
```

**블록 하나가 레이어 10개분을 덮는다.** 그룹 하나가 통째로 쓰는 단위다.

```
304 블록 × 10 레이어 × 2.05 MiB = 6.08 GiB
```

> 나는 이걸 `76 블록 × 40 레이어` 로 짐작하고 있었다. 곱이 같아서 총량 검산으로는 안 걸렸다. **밖에서 본 총량이 맞아도 안이 맞는지는 확인이 안 된다.**

## 요청 하나가 블록을 몇 개 잡나

```python
num_block_per_request = cdiv(max_memory_usage_per_request, memory_per_block)
max_concurrency = kv_cache_config.num_blocks / num_block_per_request
```

요청당 사용량은 **그룹별로 따로 세서 더한다.**

```python
return sum(spec.max_memory_usage_bytes(vllm_config) for spec in kv_cache_specs)
```

세는 법이 그룹 종류마다 다르다. full attention 은 컨텍스트를 블록으로 나눈다.

```python
return cdiv(max_model_len, self.block_size) * self.page_size_bytes
```

mamba 는 컨텍스트를 아예 안 본다.

```python
elif vllm_config.cache_config.mamba_cache_mode == "align":
    return self.page_size_bytes * (2 + self.num_speculative_blocks)
```

**`2` 가 상수로 박혀 있다.** `align` 모드는 프리픽스 캐싱을 블록 경계에 맞춰야 해서 직전 블록까지 들고 있고, 그래서 그룹당 2블록이다. linear attention 의 상태가 컨텍스트에 따라 자라지 않으니 가능한 값이다.

세어보면 이렇게 된다.

```
그룹 1  full     cdiv(65,536 ÷ 2,096)  =  32 블록   ← 컨텍스트에 비례
그룹 2  linear   2                     =   2 블록
그룹 3  linear   2                     =   2 블록   ← 컨텍스트와 무관
그룹 4  linear   2                     =   2 블록
                                          38 블록

304 ÷ 38 = 8.00
```

로그의 `8.00x` 다.

컨텍스트를 늘려보면 두 쪽의 성질이 갈린다.

<pre class="mermaid">
xychart-beta
  title "요청 하나가 잡는 블록 수"
  x-axis "컨텍스트 길이" [8K, 16K, 32K, 65K]
  y-axis "블록" 0 --> 36
  bar [4, 8, 16, 32]
  line [6, 6, 6, 6]
</pre>

막대가 full attention 그룹이고 선이 linear 그룹 셋의 합이다. **컨텍스트가 8배로 늘어도 linear 쪽은 6블록에서 움직이지 않는다.** 자라는 건 막대뿐이다.

| 컨텍스트 | full | linear | 합 | linear 비중 |
|---|---|---|---|---|
| 8K | 4 | 6 | 10 | **60%** |
| 16K | 8 | 6 | 14 | 43% |
| 32K | 16 | 6 | 22 | 27% |
| 65K | 32 | 6 | 38 | **16%** |

**짧은 대화에서는 linear 쪽이 자리의 60% 를 먹는다.** hybrid 가 이득인 건 컨텍스트가 길 때뿐이다.

앞 글에서 본 GQA 와 fp8 은 **토큰 하나의 크기**를 줄인다. hybrid 는 성질이 다르다. 풀에 담기는 토큰 수는 hybrid 여도 안 바뀌고, **길이를 따라 자라는 부분**만 4분의 1로 준다. 그 결과가 동시 처리량이다.

```
전부 full attention 이라면   32 블록 × 4 그룹  =  128 블록   →  304 ÷ 128 = 2.38x
hybrid                       32 + 2 × 3        =   38 블록   →  304 ÷  38 = 8.00x
```

> `2` 를 나는 Gated DeltaNet 이 conv 상태와 순환 상태를 따로 들고 가기 때문이라고 짐작했었다. 1로 잡으면 8.69 가 나와서 2인 건 알았는데, **이유는 아는 걸로 지어냈다.**

## GPU KV cache size 는 어떻게 나온 값인가

처음의 두 줄로 돌아간다.

```python
num_tokens = (
    kv_cache_config.num_blocks
    // len(kv_cache_config.kv_cache_groups)
    * min_block_size
)
```

```
304 ÷ 4 × 2,096 = 159,296
```

**블록을 그룹 수로 균등하게 나눈 뒤 토큰으로 환산한 값이다.** 그룹이 하나뿐인 보통 모델에서는 나누기 1이라 실제 용량과 같다. hybrid 에서는 어긋난다.

linear 그룹 셋은 요청당 2블록이면 되는데 **76블록씩 배정받은 것처럼 세니**, 안 쓰는 몫이 숫자에서 사라진다.

| | 요청당 필요한 블록 | 이 식이 배정한 블록 |
|---|---|---|
| 그룹 1 full | 32 | 76 |
| 그룹 2~4 linear | 2씩, 합 6 | 228 |

**둘째 줄이 실제에 가깝다.**

```
159,296 ÷ 65,536  =  2.43     첫 줄로 계산한 값
304 ÷ 38          =  8.00     실제로 받을 수 있는 수
```

hybrid 모델에서 동시 처리량을 볼 때는 `GPU KV cache size` 가 아니라 `Maximum concurrency` 를 봐야 한다. 컨텍스트를 바꿔가며 확인하려면 `--max-model-len` 을 바꿔 띄우고 로그를 보면 된다.

## 정리

```
그룹 수          반복 패턴의 길이. 레이어 종류 수가 아니다
페이지 크기      전 그룹 동일. mamba 쪽을 패딩해서 맞춘다
블록 수          available ÷ page ÷ 그룹당 레이어 수
요청당 블록      full 은 컨텍스트에 비례, linear 는 그룹당 2 고정
GPU KV cache size    블록 ÷ 그룹 × 블록크기. hybrid 에서는 낮게 나온다
```

안 본 것도 있다. `align` 말고 `all` 모드로 돌리면 블록 크기를 mamba 커널의 chunk_size 에 맞춰 다르게 잡는다.

```python
attn_block_size = chunk_size * cdiv(attn_tokens_per_mamba_state, chunk_size)
```

프리픽스 캐싱 동작이 얼마나 달라지는지는 다음에 볼 생각이다.
