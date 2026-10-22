---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/위키/페이지/스크림-DAY.md","permalink":"/프로젝트/pickzzk/위키/페이지/스크림-DAY/","tags":["프로젝트/pickzzk","위키","페이지","시청자","스크림"],"updated":"2026-05-14","dg-note-properties":{"tags":["프로젝트/pickzzk","위키","페이지","시청자","스크림"],"project":"pickzzk","updated":"2026-05-14","view":"ScrimDayView.vue","url":"/scrim/day/:date","role":"viewer"}}
---


# 스크림 DAY (시청자)

> 특정 date 의 스크림 / 본선 게임 모아 보기. read-only — 운영자만 편집 ([[90_AI/프로젝트/pickzzk/위키/페이지/스크림-DAY-편집\|스크림-DAY-편집]]).

## URL / 뷰

- URL: `/scrim/day/:date` (예: `/scrim/day/2026-07-08`)
- Query: `?kind=inhouse|official-scrim|unofficial-scrim|final` — 종류 filter
- View: `src/views/ScrimDayView.vue`
- 권한: 공개 (시청자 read-only)

## 표시 내용

```
헤더: <date> 스크림 DAY
├── 일정 / 결과 row 묶음 (kind 별 분류)
│   ├── 공식 스크림 (gold 배지)
│   ├── 비공식 스크림 (orange 배지)
│   ├── 본선 (gold 배지)
│   └── 내전 (orange 배지)
├── 시리즈 단위 묶음 (자동 — 같은 teams + 가까운 시각)
└── 게임별 detail → [[스크림-게임]] 으로
```

## 데이터 source

`scrimDays/{date}/games/{gameId}` (sub-collection):

```js
{
  date: "2026-07-08",
  kind: "inhouse" | "official-scrim" | "unofficial-scrim" | "final",
  tournamentId: "hojong-cup-5",
  teams: { teamA: "...", teamB: "..." },
  score: { teamA: 1, teamB: 0 },
  status: "live" | "done",
  picks: [...],
  bans: [...],
  mvp: "...",
  source: "admin" | "submission",
  approvedAt: serverTimestamp,
}
```

자세히는 [[90_AI/프로젝트/pickzzk/위키/데이터-모델\|데이터-모델]].

## 시리즈 자동 묶음

> legacy DraftPick 의 핵심 인프라. 같은 teams + 가까운 시각 (1 시간 내) 게임을 시리즈로 묶음.

자세히는 [[90_AI/프로젝트/pickzzk/위키/기능/스크림-집계\|스크림-집계]] (예정 — `series-status.test.js` 23 cases 참조).

## composable

- `useScrimAggregate(dateRef, kindRef)` — date / kind 별 aggregate
- `useScrimGame(gameId)` — 게임 1 detail
- `aggregateGames(games)` — pure aggregate

## 게임 enter

시리즈 row click → `/scrim/games/:gameId` ([[90_AI/프로젝트/pickzzk/위키/페이지/스크림-게임\|스크림-게임]]).

## kind filter

URL `?kind=` 추가 시:
- 해당 kind 만 표시
- 활성 홈 카드의 `kind=` link 에서 진입 시 자동 박힘

## 호종컵 일정

> 메모리 `scrim_tracker_architecture.md` 의 호종컵 패턴:
> - 매주 X 요일 19:00 ~ 23:00 정기
> - 공식 / 비공식 / 내전 혼재
> - 시리즈 자동 묶음 = 핵심 UX

## 박힌 R 시리즈

- R20 C7.6 — `/scrim/day/:date` ScrimDayView (legacy 로직 4 패턴 포팅) `7a600b5`

## 헷갈리는 점

> [!warning] 시청자 = read-only
> 편집은 [[90_AI/프로젝트/pickzzk/위키/페이지/스크림-DAY-편집\|스크림-DAY-편집]] (운영자 only).

> [!warning] :date format
> ISO 형식 `YYYY-MM-DD`. URL slash 그대로.

> [!warning] kind filter = 활성 홈 link 정합
> 활성 홈 의 ScrimResultCard / FinalResultCard 가 `kind=` 박은 link 로 진입.

## 관련

- [[90_AI/프로젝트/pickzzk/위키/페이지/스크림-게임\|스크림-게임]] — `/scrim/games/:gameId` 게임 detail
- [[90_AI/프로젝트/pickzzk/위키/페이지/스크림-DAY-편집\|스크림-DAY-편집]] — `/admin/scrim/day/:date?` 운영자 편집
- [[90_AI/프로젝트/pickzzk/위키/기능/스크림-집계\|스크림-집계]] — aggregate / 시리즈 묶음
- [[90_AI/프로젝트/pickzzk/위키/데이터-모델\|데이터-모델]] — `scrimDays/{date}/games/` schema
- [[90_AI/claude/개념/스킬\|스킬]] / `screen-admin-scrim-day`
- 외부: `docs/legacy-skills/` 의 scrim 관련 skill
