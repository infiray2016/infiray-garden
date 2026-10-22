---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/위키/기능/AI-프롬프트.md","permalink":"/프로젝트/pickzzk/위키/기능/AI-프롬프트/","tags":["프로젝트/pickzzk","위키","기능","AI","prompt"],"updated":"2026-05-14","dg-note-properties":{"tags":["프로젝트/pickzzk","위키","기능","AI","prompt"],"project":"pickzzk","updated":"2026-05-14","sources":["memory/ai_prompt_conventions.md"]}}
---


# AI 프롬프트 컨벤션

> pickzzk 가 Claude API (Vision / Text) 박을 때 쓰는 prompt 룰. **한국어 자연어 + 영어 LoL 용어 금지 + 수치 남발 방지 + OCR ★ 앵커**.

## 핵심 룰 (메모리 `ai_prompt_conventions.md`)

> user 박힘:
> - 한국어 자연어 prompt
> - 영어 LoL 용어 금지 (한글 명만)
> - 수치 (정확도 / confidence) 남발 방지
> - OCR ★ 앵커 (확신 정도 표시)

## 왜 이렇게?

### 1) 한국어 자연어

- pickzzk = 한국어 user base (치지직 시청자)
- LoL 용어도 한국어 (탑 / 정글 / 미드 / 원딜 / 서폿)
- 영어 prompt = 영문 챔프 이름 (Aatrox 등) 출력 위험 — 한글 변환 필요

### 2) 영어 LoL 용어 금지

prompt 안에 영어 용어 ("Top lane" / "ADC" / "Support") 박지 X. 한글 만 ("탑 / 원딜 / 서폿").

이유:
- Claude 가 영어로 응답 가능 (영어 prompt → 영어 chain)
- 출력 = 한글 챔프 이름 필요 (text OCR primary)

### 3) 수치 남발 방지

prompt 안에 "정확도 95%" / "confidence 0.9" 같은 숫자 박지 X.

이유:
- Claude 가 임의로 confidence 박음 (실제 정확도와 무관)
- 사용자 misleading (signal X)
- 대신 ★ 앵커 사용

### 4) OCR ★ 앵커

confidence 표시 = 별 (★) 단계로 박음. 자연어 + 시각 친화.

```
인식 정확도:
★★★★★  매우 확실 (한글 text + ONNX 둘 다 매치)
★★★★☆  확실 (한글 text 단독 매치, ONNX 다름)
★★★☆☆  애매 (cross-check 불일치 — 사용자 정정 필요)
★★☆☆☆  의심 (둘 다 약함)
★☆☆☆☆  못 박힘 (사용자 직접 입력)
```

prompt 예:
```
챔프 인식 결과를 ★ 단계로 박아줘:
- ★★★★★ = 확실
- ★★★ = 애매
- ★ = 못 박힘
```

## 사용 영역

### Claude Vision (`/api/scrim-parse`)

LoL 결과창 → 챔프 / KDA / 결과 parse:

```
당신은 LoL 결과창 이미지를 분석하는 시스템입니다.

주어진 이미지에서:
1. 양 팀 챔프 5명씩 (한글 이름)
2. 각 챔프의 KDA (k/d/a)
3. 양 팀 닉네임
4. 매치 결과 (어느 팀 승)
5. 챔프 컬럼 위치 (championColumnBboxA/B)

박을 때:
- 챔프는 한글 이름만 (Aatrox 박지 마, "아트록스" 박아)
- 정확도 = ★ 단계 (★★★★★ ~ ★)
- 위치 = [x, y, w, h] (이미지 좌상단 0,0)

응답 = JSON 만.
```

### Claude Text (스크림 일간 브리핑 / 뉴스레터)

```
호종컵 5기 N월 N일 스크림 결과 brief 박아줘:
- 게임 수 / 시리즈 수
- 인기 픽 챔프
- 최다 캐리 (MVP)
- 다음 일정

시청자 친화 카피로. 어조 = 가벼움, terminal 문법 ↓.
한글 챔프 이름만. 영문 X.
```

## API

`POST /api/scrim-parse` — Vision (이미지)
`POST /api/scrim-newsletter` — Text (브리핑 / 뉴스레터, TBD)

자세히는 `/api/` 서버 코드.

## 환경 변수

- `ANTHROPIC_API_KEY` — Claude API 인증 (서버 endpoint 만, 클라 X)

## 비용 관리

- 95%+ 시간 CLOSED — Claude API 사용 거의 X
- 활성 시기 = OCR 빈도 ↑ (시청자 제보)
- cron / 뉴스레터 = 운영 명시 결정 후 박음 (default OFF)

자세히는 [[90_AI/프로젝트/pickzzk/위키/개요\|개요]] 운영 cadence.

## 한국어 verbatim 출력

> Claude 출력 = user 가 박은 그대로 (verbatim) 박을 것.

예:
- 자낳대 (자내대 X / 자낣대 X)
- 호종컵 (한자 X)
- 모의 (시뮬레이션 X — "모의" verbatim)
- 매물 (매물 매물 — 자낳대 정합)
- 사전 내전 (pre-open)
- 스크림 (영어 보존 OK — 한국어 통용)

자세히는 [[90_AI/프로젝트/pickzzk/위키/자낳대-룰\|자낳대-룰]] / 한국어-용어.md (TBD — 위키 박을 후보).

## 시청자 친화 카피 룰

> CLAUDE.md / `v10-톤`:
> "시청자 친화 카피 (terminal 문법 ↓)"

지양:
- `> error: command not found`
- `[ERROR] 404 Not Found`
- raw enum (`kind=inhouse`)
- 영어 jargon
- 수치 남발 (`정확도 95%`)

지향:
- 자연 한국어
- "집계 대기" / "운영자 · 시청자 첫 게임 등록 대기 중"
- ★ 앵커 + 시각 친화

## 박힌 R 시리즈

- (legacy) OCR prompt 컨벤션 박힘
- R20+ — pickzzk 포팅

## 헷갈리는 점

> [!warning] 영어 LoL 용어 박지 마
> prompt 안에 "Top / Jungle / Mid / ADC / Support" 박지 마. 한글만.

> [!warning] confidence 숫자 박지 마
> "정확도 95%" / "확률 0.92" 같은 숫자 prompt 출력 = signal X. 대신 ★ 앵커.

> [!warning] 한국어 verbatim
> 자낳대 / 호종컵 / 모의 / 매물 verbatim. 영어 / 다른 표현 박지 마.

> [!warning] terminal 문법 지양
> v10 톤 = 시청자 친화. 운영자 SaaS 같은 terminal 카피 X.

## 관련

- [[90_AI/프로젝트/pickzzk/위키/기능/OCR\|OCR]] — Claude Vision 사용
- [[90_AI/프로젝트/pickzzk/위키/기능/픽밴-classifier\|픽밴-classifier]] — ONNX (Claude 와 별)
- [[90_AI/프로젝트/pickzzk/위키/기능/시청자-제보\|90_AI/프로젝트/pickzzk/위키/기능/시청자-제보]] / [[90_AI/프로젝트/pickzzk/위키/페이지/시청자-제보\|90_AI/프로젝트/pickzzk/위키/페이지/시청자-제보]] — OCR 활용
- [[90_AI/프로젝트/pickzzk/위키/v10-톤\|v10-톤]] — 시청자 친화 카피 룰
- [[90_AI/claude/개념/메모리\|메모리]] / `ai_prompt_conventions`
