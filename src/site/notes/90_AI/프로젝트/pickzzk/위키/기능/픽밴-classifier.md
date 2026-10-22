---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/위키/기능/픽밴-classifier.md","permalink":"/프로젝트/pickzzk/위키/기능/픽밴-classifier/","tags":["프로젝트/pickzzk","위키","기능","OCR","ONNX","AI"],"updated":"2026-05-14","dg-note-properties":{"tags":["프로젝트/pickzzk","위키","기능","OCR","ONNX","AI"],"project":"pickzzk","updated":"2026-05-14","related_skill":"feature-pickban-classifier","priority":"활성 시기 직전 필수"}}
---


# 픽밴 Classifier (ONNX)

> LoL 픽밴 화면의 챔프 아이콘 (32~40 px) 분류 — ONNX 모델 web inference. **활성 시기 직전 필수 이식 작업** (`feedback_pickban_classifier_pending.md` 메모리).

## 핵심

- legacy DraftPick 의 ban-classifier ONNX 모델 (~6.7 MB)
- pickzzk 로 미이식 — 운영 활성 시기 직전 필수
- 챔프 32~40 px 아이콘 분류 정확도 = 핵심
- 결과창 챔프 ([[90_AI/프로젝트/pickzzk/위키/기능/OCR\|OCR]]) cross-check 와 별개 또는 공유 (검증 필요)

## 왜 ONNX?

- 클라이언트 web inference (서버 사용 X) → 비용 0
- 빠른 분류 (모델 size 6.7 MB 도 부담 OK)
- 정확도 검증 = 픽밴 화면 (큰 아이콘 60+ px) 에 학습된 모델

## 결과창 OCR 과 차이

| 영역 | 픽밴 화면 | 결과창 |
|---|---|---|
| 아이콘 size | 60+ px | 32~40 px |
| 학습 모델 | ban-classifier (full) | 같은 모델 재사용? 또는 별 (재학습 검토) |
| 1차 인식 | ONNX (visual) | 한글 text OCR |
| 2차 인식 | text? | ONNX cross-check |

> [!info] 결과창 챔프 = text primary (verbatim)
> user 박힘 (`feedback_champion_text_ocr_primary.md`):
> "결과 스크린샷 챔프는 한글 텍스트 OCR 1차, ONNX classifier 는 cross-check / fallback"
>
> 픽밴 화면 = 한글 text 거의 없음 → ONNX primary.

## 이식 잔여 작업

> 현재 = `useBanRecognition` composable 박혀 있지만 실 ONNX 모델 미연결.

```js
// src/composables/useBanRecognition.js (현재 — TBD 코드 검색)
export function useBanRecognition() {
  // ONNX 모델 load
  // 픽밴 화면 crop 후 분류
  // result = { champ: "Aatrox", confidence: 0.95 }
}
```

### 박힐 단계

1. **legacy 모델 가져옴** — `~/workspace/draftpick/draftpick_vue/` 의 ONNX 파일 (.onnx ~6.7 MB)
2. **`public/models/ban-classifier.onnx`** 박음 (또는 `src/assets/`)
3. **onnxruntime-web 의존** 박음 (`bun add onnxruntime-web`)
4. **`useBanRecognition` 실 구현** — 모델 load + inference
5. **챔프 라벨 매핑** (모델 output index → 챔프 이름)
6. **테스트** — fixture 이미지 + expected 챔프
7. **결과창 적용 검증** (32~40 px 정확도)

## 활성 시기 직전 필수

> user 박힘 (`feedback_pickban_classifier_pending.md`):
> "활성 시기 직전 필수 이식"

- 95% 시간 CLOSED → 이식 미루기 OK
- 활성 1주 전 = 박혀 있어야 OCR 정확도 보장
- 현재 = R20+ 이후 후속 작업 후보

자세히는 [[../할일/README\|../할일/README]].

## composable

- `useBanRecognition()` — 픽밴 ONNX (현재 stub 상태)
- `useChampionRecognition()` — 결과창 챔프 (Claude Vision + ONNX cross-check)

## API / 데이터

- ONNX 모델 파일: `public/models/ban-classifier.onnx` (이식 후)
- 라벨 매핑: `src/constants/champions-onnx-labels.json` (이식 후)
- inference 결과: `{ champ, confidence }` per crop

## 헷갈리는 점

> [!warning] 픽밴 vs 결과창 = 다른 use case
> - 픽밴 화면 = 시청자 라이브 OCR (LIVE 카드 폐기 — 현재 안 함)
> - 결과창 = post-game OCR (현재 메인)
> 둘 다 같은 ONNX 사용? 또는 별 (검증 필요).

> [!warning] 활성 시기 직전 = 1 주 ~ 며칠 전
> "직전" 의 정의 = 활성 시작 전. 너무 늦으면 운영 차질.

> [!warning] 6.7 MB 모델 = 클라이언트 download
> 첫 로드 시 download. 캐싱 OK 지만 첫 인상 느림 가능. preload 검토.

> [!warning] LIVE 매치 카드 폐기
> 픽밴 라이브 OCR = 메인에 박지 X (실시간 추적 안 함). 결과창 OCR 만 메인.

## 박힘 후 적용 영역

- [[90_AI/프로젝트/pickzzk/위키/기능/시청자-제보\|90_AI/프로젝트/pickzzk/위키/기능/시청자-제보]] / [[90_AI/프로젝트/pickzzk/위키/페이지/게임-업로드\|게임-업로드]] — 결과창 OCR 의 cross-check
- (옵션) 픽밴 화면 OCR — 별 use case 검토

## 관련

- [[90_AI/프로젝트/pickzzk/위키/기능/OCR\|OCR]] — Claude Vision parse / 결과창 메인
- [[90_AI/프로젝트/pickzzk/위키/기능/AI-프롬프트\|AI-프롬프트]] — Vision prompt 컨벤션
- [[../할일/README\|../할일/README]] — "활성 시기 직전 필수" 항목
- [[90_AI/claude/개념/스킬\|스킬]] / `feature-pickban-classifier`
- [[90_AI/claude/개념/메모리\|메모리]] / `feedback_pickban_classifier_pending`
- 외부: `~/workspace/draftpick/draftpick_vue/` (legacy 모델 출처, freeze)
