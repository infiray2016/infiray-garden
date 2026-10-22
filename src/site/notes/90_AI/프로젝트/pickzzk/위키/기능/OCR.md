---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/위키/기능/OCR.md","permalink":"/프로젝트/pickzzk/위키/기능/OCR/","tags":["프로젝트/pickzzk","위키","기능","OCR","AI"],"updated":"2026-05-14","dg-note-properties":{"tags":["프로젝트/pickzzk","위키","기능","OCR","AI"],"project":"pickzzk","updated":"2026-05-14","related_skill":"feature-game-input-ocr"}}
---


# OCR (게임 결과 입력)

> LoL 결과창 / 전적 페이지 스크린샷에서 챔프 / KDA / 결과 자동 추출. **공식 권한 (Riot API) 없이 데이터 만드는 핵심 인프라**. Claude Vision 1차 + 한글 text OCR + ONNX classifier cross-check.

## 핵심 가치

> pickzzk = "공식 권한 없이 데이터 만듦" 의 핵심.

- Riot API 사용 X (정식 권한 X)
- 시청자 / 운영자가 결과창 캡처 → AI 가 parse
- 매 시즌 cadence 작게 cron OFF — OCR 만으로 충분

자세히는 [[90_AI/프로젝트/pickzzk/위키/개요\|개요]].

## 흐름

```
1. 이미지 업로드 (시청자 [[../페이지/시청자-제보]] 또는 운영자 [[../페이지/게임-업로드]])
   ↓
2. POST /api/scrim-parse (Claude Vision)
   ↓
3. 응답: champions[], kda[], result, championColumnBboxA/B
   ↓
4. 클라이언트:
   ├── 챔프 = 한글 text OCR primary (Claude Vision)
   ├── ONNX classifier cross-check / fallback
   ├── chzzkPlayer alias 매칭 (streamersDB)
   └── 사용자 confirm / 정정 (chip click)
   ↓
5. POST /api/scrim-submit (시청자) 또는 직접 admin save (운영자)
   ↓
6. scrimSubmissions/ 또는 scrimDays/{date}/games/ 박힘
```

## API

`POST /api/scrim-parse`:

Request:
```json
{
  "image": "base64",
  "kind": "inhouse" | "official-scrim" | "unofficial-scrim" | "final",
  "date": "YYYY-MM-DD"
}
```

Response:
```json
{
  "champions": [
    { "name": "Aatrox", "team": "A", "position": "Top", "playerName": "PEAK", ... },
    ...
  ],
  "kda": [
    { "playerName": "PEAK", "k": 5, "d": 2, "a": 3, ... },
    ...
  ],
  "result": "A_win" | "B_win" | "tied",
  "championColumnBboxA": [x, y, w, h],
  "championColumnBboxB": [x, y, w, h],
  "meta": { ... }
}
```

자세히는 [[90_AI/프로젝트/pickzzk/위키/기능/AI-프롬프트\|AI-프롬프트]] 의 vision prompt 컨벤션.

## 챔프 인식 = 한글 text OCR primary

> user verbatim (`feedback_champion_text_ocr_primary.md`):
> "결과 스크린샷 챔프는 한글 텍스트 OCR 1차, ONNX classifier 는 cross-check / fallback"

### 1차: Claude Vision (한글 text)

결과창의 "아트록스 / 케일 / ..." 같은 한글 텍스트 → 영어 챔프 이름으로 매핑.

### 2차: ONNX classifier cross-check

`useChampionRecognition` 의 ONNX 모델 = 결과창 작은 챔프 아이콘 (32~40 px) 분류.

- 1 차 한글 OCR 결과와 일치 → confirmed
- 불일치 → candidate (운영자 / 시청자 정정)

자세히는 [[90_AI/프로젝트/pickzzk/위키/기능/픽밴-classifier\|픽밴-classifier]] / `feature-pickban-classifier` skill.

### 3차: fallback (사용자 직접 박음)

text + ONNX 둘 다 실패 → user 가 직접 chip click / 입력.

## championColumnBbox 의미

> Claude Vision 이 결과창에서 챔프 컬럼 위치 좌표 (bbox) 도 박음. 클라이언트가 cross-check 시 ONNX 분류기 input crop 으로 사용.

```
championColumnBboxA = [x, y, w, h]  // 팀 A 챔프 컬럼 (5 챔프 위에서 아래)
championColumnBboxB = [x, y, w, h]  // 팀 B
```

→ 클라이언트가 bbox crop 한 5 챔프 아이콘 → ONNX 분류 → text 와 cross-check.

## chzzkPlayer alias 매칭

> 닉네임 매칭 = streamersDB 의 `alias[]` 사용.

```
OCR raw: "Peakkk"
alias map: { "peakk": "peak", "peakkk": "peak", "PEAK": "peak", ... }
→ streamerId = "peak"
```

미매칭 시 → user 가 chip click / 직접 입력 → alias 자동 저장 (후순위).

자세히는 [[90_AI/프로젝트/pickzzk/위키/기능/streamerDB\|streamerDB]].

## OcrStatusFrame 패턴

> 박힘 상태별 보더 + status pill — `feature-game-input-ocr` 스킬 박힘.

| status | 보더 | 의미 |
|---|---|---|
| `auto` | 라이트 | Claude Vision 자동 추출 |
| `candidate` | 오렌지 | 후보 여러 개 (정정 필요) |
| `unknown` | 회색 + warning | 못 박힘 |
| `confirmed` | 초록 | user 가 confirm |

> [!info] 4 곳 중복 패턴 = 추출 후보
> LineupNameInput / GameInputChampionField / GameInputMeta / GameInputLineup 4 컴포넌트가 같은 패턴 박힘.
> 추출 = 후순위 잔여 ([[../할일/README\|../할일/README]]).

## 컴포넌트 (`src/components/game-input/`)

- `GameInputUpload.vue` — 이미지 업로드 / 진행
- `GameInputChampionField.vue` — 챔프 chip / 정정
- `GameInputLineup.vue` — 5+5 라인업
- `GameInputMeta.vue` — date / teams / score
- `LineupNameInput.vue` — chzzkPlayer 매칭

## composable

- `useGameInputState()` — 입력 상태 + 검증
- `useChampionRecognition()` — Claude Vision + ONNX cross-check
- `useBanRecognition()` — 픽밴 ONNX classifier (자세히는 [[90_AI/프로젝트/pickzzk/위키/기능/픽밴-classifier\|픽밴-classifier]])
- `useStreamersList()` — alias 매칭
- `useDuplicateCheck()` — 동일 제보 의심 check

## 검증 잔여

> 후순위 보강 ([[../할일/README\|../할일/README]]).

- **`/api/scrim-parse` 의 `championColumnBboxA/B` 실 응답** — Vision 이 prompt 따라 정확히 박스 잡는지. `ANTHROPIC_API_KEY` 필요.
- **3-2.png · 4-2.png fixture 회귀** — server 응답 fixture 화 시 `visionToGameInput` 룰 변경 자동 감지.
- **챔프 visual classifier 정확도** — ban-classifier 모델 (full 6.7MB) 을 결과창 작은 아이콘 (32~40px) 에 그대로 사용 중. 정확도 낮으면 결과 화면 전용 모델 재학습 고려.

## 후순위 보강 (활성 시기 직전 또는 운영 후)

- **`scrimPlayerAliases` 컬렉션 활성화** — `useStreamersList` / 별도 composable 로 alias map 로드 → `visionToGameInput(parsed, streamers, teams, aliasesByName)` 4 번째 인자 전달. 현재 `{}` 만 박혀 `matchRiotIdScored` step 3/6 (alias 정확/fuzzy) 가 dead code.
- **사용자 confirm → alias 자동 저장** — chip click / 직접 입력 시 `scrimPlayerAliases/{chzzkName}.aliases` 박음 (legacy ScrimSubmitView 패턴). 시즌 거듭하면서 매칭률 자동 향상.
- **`OcrStatusFrame` 추출** — auto/candidate/unknown/confirmed 보더 + status pill 패턴이 4 곳 중복. 공유 컴포넌트로 추출.

## 박힌 R 시리즈

- (legacy DraftPick) — OCR 인프라 박힘
- R20 이전 — pickzzk 포팅
- Codex 리뷰 잔여 — `feature-game-input-ocr` 스킬 박힘 잔여

## 헷갈리는 점

> [!warning] 한글 text OCR primary, ONNX secondary
> 결과창은 한글 text 가 가장 정확. ONNX 는 cross-check / fallback. user verbatim 박힘.

> [!warning] 픽밴-classifier ONNX ≠ 결과창 ONNX
> - 픽밴 화면 = ban-classifier (legacy ONNX) — 활성 시기 직전 이식 필수
> - 결과창 = 같은 ONNX 재사용? 또는 별 모델? (정확도 검증 필요)

> [!warning] 시청자 제보 vs 운영자 업로드 = 같은 OCR
> game-input/ 컴포넌트 공유. 차이는 큐 거치냐 / 즉시 박히냐.

> [!warning] chzzkPlayer alias 활성화 미완료
> `aliasesByName` 4 번째 인자가 현재 `{}` 만 박힘. 후순위 잔여.

## 관련

- [[90_AI/프로젝트/pickzzk/위키/기능/픽밴-classifier\|픽밴-classifier]] — legacy ONNX 이식 (잔여)
- [[90_AI/프로젝트/pickzzk/위키/기능/AI-프롬프트\|AI-프롬프트]] — Claude Vision prompt 컨벤션
- [[90_AI/프로젝트/pickzzk/위키/기능/시청자-제보\|90_AI/프로젝트/pickzzk/위키/기능/시청자-제보]] / [[90_AI/프로젝트/pickzzk/위키/페이지/게임-업로드\|게임-업로드]] — OCR 진입 페이지
- [[90_AI/프로젝트/pickzzk/위키/기능/streamerDB\|streamerDB]] — chzzkPlayer alias
- [[90_AI/프로젝트/pickzzk/위키/기능/스크림-집계\|스크림-집계]] — 결과 박힌 후 aggregate
- [[90_AI/claude/개념/스킬\|스킬]] / `feature-game-input-ocr`
- 외부: `docs/legacy-skills/` 의 OCR 관련 skill
