---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/위키/페이지/CLOSED.md","permalink":"/프로젝트/pickzzk/위키/페이지/CLOSED/","tags":["프로젝트/pickzzk","위키","페이지","시청자","closed"],"updated":"2026-05-14","dg-note-properties":{"tags":["프로젝트/pickzzk","위키","페이지","시청자","closed"],"project":"pickzzk","updated":"2026-05-14","view":"components/closed/*.vue (또는 HomeView 안 분기)","url":"/ (stage 가 'pre-launch' 또는 'archived' 일 때)","role":"viewer"}}
---


# CLOSED 안내

> `tournament.stage='pre-launch'` (또는 활성 토너먼트 없음) 시 시청자가 진입하면 보는 정적 안내 페이지. **JS 거의 없음** — 단순 1 장.

## URL / 뷰

- URL: `/`
- 조건: `tournament` 없거나 `stage='pre-launch' || 'archived'`
- View: `src/components/closed/` (또는 HomeView 안 v-if 분기)
- 권한: 공개

## 의도

> 95%+ 시간 CLOSED — pickzzk 의 운영 cadence.

- 다음 대회 시작 전까지 cron / 자동 폴링 OFF
- 시청자가 와도 "지금 활성 없음" 안내 필요
- 단순 정적 1 장 = 가장 적게 박는 화면
- 비용 0 지향

## 표시 내용

```
헤더: pickzzk
├── 안내 카피: "현재 활성 대회가 없어요"
├── 다음 대회 안내 (있으면)
│   ├── 시즌명 / 일정
│   └── 사전 알림 / SNS link
├── 옛 시즌 아카이브 link (`/archive/...`)
└── (옵션) 이메일 / 디스코드 / 트위터 link
```

## 우선순위 (CLAUDE.md)

> 화면 우선순위 #4 (활성 홈 / 운영자 셋업 / 스트리머 DB 다음).

자세히는 [[90_AI/프로젝트/pickzzk/위키/개요\|개요]] 의 "화면 우선순위" 섹션.

## 라우터 fallback

`useActiveTournament` composable:
- 활성 tournament 없음 → `stage='closed'` 반환 (legacy alias)
- `HomeView.vue` 가 분기 → CLOSED 안내 표시

```js
<ClosedPage v-if="isClosed || stage === 'pre-launch' || stage === 'archived'" />
```

## v10 톤

- 라이트 톤 유지
- 시그널 오렌지 강조 (다음 대회 안내 / link)
- 정적 = 동적 X (애니메이션 / 인터랙션 거의 없음)
- 시청자 친화 카피 (terminal 문법 ↓)

## 후속 작업 후보

- 다음 대회 일정 + countdown
- 옛 시즌 highlights 캐러셀 (3-4 archived 시즌)
- SNS / 디스코드 link 강화
- 이메일 알림 등록 (옵션)

## 헷갈리는 점

> [!warning] CLOSED ≠ archived
> - `pre-launch` = 활성 대회 X (다음 시즌 박힌 후 advance 전)
> - `archived` = 시즌 종료 (별도 [[90_AI/프로젝트/pickzzk/위키/페이지/아카이브\|아카이브]] 페이지 진입 가능)
> 활성 홈 진입 시 둘 다 CLOSED 안내 페이지 표시.

> [!warning] tournament 없음 = `closed` legacy alias
> `useActiveTournament` 의 `stage` computed 가 `tournament.value?.stage ?? "closed"`. 활성 없음 = closed 표시.

> [!warning] JS 거의 없음 = 의도
> 비용 0 + 빠른 로딩 + 단순. SSR / 정적 빌드 친화.

## 관련

- [[90_AI/프로젝트/pickzzk/위키/화면\|화면]] — stage 별 화면
- [[90_AI/프로젝트/pickzzk/위키/개요\|개요]] — 운영 cadence (95%+ CLOSED)
- [[90_AI/프로젝트/pickzzk/위키/페이지/활성-홈\|활성-홈]] — stage 활성 시 진입
- [[90_AI/프로젝트/pickzzk/위키/페이지/아카이브\|아카이브]] — `stage='archived'` 의 별도 페이지
- [[90_AI/프로젝트/pickzzk/위키/데이터-모델\|데이터-모델]] — tournament.stage / timelineEvents
- [[90_AI/claude/개념/스킬\|스킬]] / `screen-closed`
