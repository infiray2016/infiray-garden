---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/위키/페이지/스크림-DAY-편집.md","permalink":"/프로젝트/pickzzk/위키/페이지/스크림-DAY-편집/","tags":["프로젝트/pickzzk","위키","페이지","운영자"],"updated":"2026-05-14","dg-note-properties":{"tags":["프로젝트/pickzzk","위키","페이지","운영자"],"project":"pickzzk","updated":"2026-05-14","view":"admin/ScrimDayAdminView.vue (TBD)","url":"/admin/scrim/day/:date?","role":"admin"}}
---


# 스크림 DAY 편집 (운영자)

> 특정 date 의 스크림 / 본선 게임 결과 / 메타 편집. legacy DraftPick 의 핵심 운영자 화면 정합.

## URL / 뷰

- URL: `/admin/scrim/day/:date?` (date 옵션 — 없으면 오늘)
- View: `src/views/admin/ScrimDayAdminView.vue` (또는 admin views/)
- 권한: admin only (`useAdminGate`)

## 표시 / 박는 것

```
헤더: <date> 스크림 DAY 편집
├── kind 별 game list (inhouse / official / unofficial / final)
│   ├── 각 game row → 인라인 편집 또는 detail 진입
├── 새 game 박는 buttons (per kind)
│   ├── + 내전
│   ├── + 공식 스크림
│   ├── + 비공식 스크림
│   └── + 본선 (final)
└── kind 별 일괄 작업
    ├── 시리즈 자동 묶음
    └── 결과 일괄 확정 (status='done')
```

## 컴포넌트 (`src/components/admin/`)

(TBD — 정확한 위치 코드 검색 필요)

- `ScrimDayAdminHeader.vue`
- `ScrimDayGameRow.vue` — 인라인 편집
- `ScrimDayGameEditor.vue` — 모달 / drawer 편집
- `ScrimDayBatchActions.vue`

## 인라인 편집 vs 모달

- **빠른 편집** (status / score 등) = 인라인
- **깊은 편집** (picks / KDA / champion 정정) = 모달 → `/admin/scrim/upload` 와 같은 component 재사용

## composable

- `useScrimDayAdmin(dateRef)` — date 별 game list + 편집 actions
- `useScrimGameEditor(gameId)` — game 1 editor
- `useAdminGate()`

## 데이터 source

`scrimDays/{date}/games/{gameId}` (sub-collection):
- read = `collection('scrimDays/.../games').onSnapshot`
- write = admin SDK 경유 (서버 endpoint `/api/scrim-admin`)

자세히는 [[90_AI/프로젝트/pickzzk/위키/데이터-모델\|데이터-모델]] / `docs/firebase-rules.md`.

## 박힘 source 별 처리

| source | 처리 |
|---|---|
| `admin` | 운영자 직접 박은 거 — 자유 편집 |
| `submission` | 시청자 제보 promote — 정정 시 audit log 박힘 |

## 시리즈 자동 묶음

같은 teams + 가까운 시각 (1 시간 내) game = 시리즈 자동 묶음. 운영자가 수동 박을 필요 X.

자세히는 [[90_AI/프로젝트/pickzzk/위키/기능/스크림-집계\|스크림-집계]] / `series-status.test.js`.

## 박힌 R 시리즈

- R20 C7 ~ — legacy `scrim-day` UI 포팅

## 시청자 view 와 비교

| 영역 | 시청자 [[90_AI/프로젝트/pickzzk/위키/페이지/스크림-DAY\|스크림-DAY]] | 운영자 (여기) |
|---|---|---|
| URL | `/scrim/day/:date` | `/admin/scrim/day/:date?` |
| 편집 | X (read-only) | ✅ |
| game list | kind 분류 표시 | kind 분류 + 편집 |
| 진입 | 시청자 nav / 활성 홈 card | admin nav `DAY 편집` |

## 헷갈리는 점

> [!warning] :date 옵션
> URL `/admin/scrim/day` (date 없음) = 오늘 date default. ScrimDayView (시청자) 는 :date 필수.

> [!warning] 운영자 정정 / 시청자 제보 audit
> 시청자 제보 promote 후 운영자가 정정 시 audit log 박힘 (TBD — 코드 박힌 거 확인).

> [!warning] 시리즈 묶음 = 자동
> 수동으로 묶지 X. `useScrimAggregate` 가 같은 teams + 시각 기준 자동 묶음.

## 관련

- [[90_AI/프로젝트/pickzzk/위키/페이지/스크림-DAY\|스크림-DAY]] — `/scrim/day/:date` 시청자 view
- [[90_AI/프로젝트/pickzzk/위키/페이지/스크림-게임\|스크림-게임]] — game 1 detail
- [[90_AI/프로젝트/pickzzk/위키/페이지/게임-업로드\|게임-업로드]] — `/admin/scrim/upload` (새 game 박는 별 페이지)
- [[90_AI/프로젝트/pickzzk/위키/페이지/제보-큐\|제보-큐]] — `/admin/submissions` 큐
- [[90_AI/프로젝트/pickzzk/위키/기능/스크림-집계\|스크림-집계]] — aggregate / 시리즈
- [[90_AI/프로젝트/pickzzk/위키/데이터-모델\|데이터-모델]] — `scrimDays/{date}/games/`
- [[90_AI/claude/개념/스킬\|스킬]] / `screen-admin-scrim-day`
