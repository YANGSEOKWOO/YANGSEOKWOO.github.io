# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Jekyll 기반 GitHub Pages 개발 블로그 (Seokwoo's Dev Blog, https://yangseokwoo.github.io). `github-pages` gem으로 빌드되며 노션 스타일 카드 레이아웃을 쓴다.

## Commands

로컬에 Ruby/Bundler가 설치돼 있어야 한다 (이 저장소에는 포함되지 않음).

```bash
bundle install                 # 최초 1회 의존성 설치
bundle exec jekyll serve        # 로컬 미리보기 → http://localhost:4000
bundle exec jekyll build        # _site/ 로 정적 빌드
```

테스트·린트 설정은 없다. 배포는 `main`에 푸시하면 GitHub Pages가 자동 빌드한다.

## 글(post) 규약 — 중요

- 글은 `_posts/YYYY-MM-DD-<영문-슬러그>.md`에 둔다. (`_posts/` 폴더는 아직 없을 수 있으니 없으면 만든다.)
- 프론트매터는 **단수** `category`를 쓰며, 값은 정확히 `AI` / `CS` / `Project` / `Programming` 중 하나여야 한다. 이 값이 다음 세 곳과 **대소문자까지 일치**해야 동작한다:
  - `index.html`의 카테고리 필터 버튼 `data-category`
  - `categories/*.md`의 `category_key` (카테고리 페이지는 `where: "category", page.category_key`로 필터링)
  - `graph.html`의 그룹 색상 매핑
- 퍼머링크는 `_config.yml`의 `/:categories/:title/`.
- `_templates/`의 `ai.md`/`cs.md`/`project.md`/`programming.md`는 **발행되지 않는다** (`_config.yml`의 `exclude`). 새 글의 골격 템플릿이며, "내가 오해하고 있던 것 / trade-off / 관련 글 링크" 같은 학습 회고 구조를 강제한다.

## 지식 그래프 (`/graph`, `graph.html`)

옵시디언 스타일 글-연결 그래프. **노드와 엣지가 분리된 구조**라 이해가 필요하다:

- `graph.html` (`/graph/`): **노드는 Jekyll이 `site.posts`에서 빌드 시 자동 생성**한다 (항상 최신). 노드 id = `post.slug`. vis-network(CDN)로 force-directed 렌더, 카테고리별 색 + 필터. 노드 클릭 시 글로 이동.
- `assets/graph-data.json`: 글 사이의 **엣지**만 담는다 (`{ generated, edges: [{from, to, reason}] }`, slug로 키잉). 페이지는 양 끝이 실존하는 엣지만 그린다.
- `/graph` 커맨드: `_posts/`를 스캔해 **엣지만** 재계산하고 `graph-data.json`을 덮어쓴다 (노드는 페이지가 자동 생성하므로 건드리지 않는다). 새 글을 추가한 뒤 실행할 것.

## 세션 워크플로우 커맨드 (`.claude/commands/`)

이 블로그의 작업 방식은 **페르소나 기반 세션 커맨드**다 — "글 템플릿 빈칸 채우기"가 아니라, 커맨드가 한 세션의 모드(페르소나 + 흐름)를 정의하고 그 **과정 자체가 콘텐츠**가 된다.

- `/study-bottom <주제>`: 바닥부터 같이 배우는 학습 세션. "동료 + 검증" 페르소나(같이 배우되 모든 주장을 코드·문서·실험으로 검증, 불확실하면 솔직히 밝힘). 세션 끝에 학습 과정을 `_posts/`의 글 초안으로 정리하고, 관련 글 링크를 채워(= 그래프 엣지) `/graph` 실행을 안내한다.
- 새 세션 모드를 추가할 때도 같은 패턴(페르소나 + 흐름 + 산출물 규칙)을 따른다.

글을 쓸 때는 "다음에 봐도 잘 읽히는 구조"를 지킨다: 맨 위 TL;DR, 카테고리 템플릿을 따른 일관된 헤딩, **오해→정정 섹션을 비우지 않기**, 그리고 다른 글로의 링크를 반드시 채우기(그래프가 끊기지 않도록).

## Layout 구조

`_layouts/default.html`(헤더/푸터 + `.container`) → `post.html`(글) / `category.html`(카테고리별 카드 목록)로 상속. 스타일은 `assets/css/style.css` 한 파일이며 `:root` CSS 변수로 노션풍 팔레트(`--accent: #2eaadc` 등)를 정의한다.
