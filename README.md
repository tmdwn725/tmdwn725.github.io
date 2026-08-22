# tmdwn725.github.io

개인 블로그. 온프렘 LLM 인프라와 에이전트 런타임에 대한 기록입니다.

**https://tmdwn725.github.io**

[Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 테마를 사용합니다.

## 글 쓰기

`_posts/YYYY-MM-DD-slug.md` 형식으로 파일을 만들고 front matter를 넣습니다.

```yaml
---
title: 제목
description: 검색 결과와 링크 미리보기에 쓰이는 한 줄
date: 2026-08-22 10:00:00 +0900
categories: [Hardware, LLM]
tags: [dgx-spark, vllm]
---
```

커밋하면 GitHub Actions가 빌드해서 배포합니다.

## 로컬 미리보기

Docker로 실행합니다(Ruby 설치 불필요).

```bash
docker run --rm -it -v "$PWD":/srv -w /srv -p 4001:4000 ruby:3.3 \
  bash -c "bundle install && bundle exec jekyll serve --host 0.0.0.0 --future"
```

→ http://localhost:4001
