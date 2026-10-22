---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/위키/페이지/스트리머-DB.md","permalink":"/프로젝트/pickzzk/위키/페이지/스트리머-DB/","tags":["프로젝트/pickzzk","위키","페이지","운영자","마스터-DB"],"updated":"2026-05-14","dg-note-properties":{"tags":["프로젝트/pickzzk","위키","페이지","운영자","마스터-DB"],"project":"pickzzk","updated":"2026-05-14","view":"admin/StreamersDBView.vue (TBD)","url":"/admin/streamers","role":"admin"}}
---


# 스트리머 DB (운영자)

> 마스터 컬렉션 `streamers/{id}` 관리. LoL + OW 공유. **시즌 거듭하며 누적**. 운영자 우선순위 #3 (활성 홈 / 운영자 셋업 다음).

## URL / 뷰

- URL: `/admin/streamers`
- View: `src/views/admin/StreamersDBView.vue` (또는 admin views/)
- 권한: admin only

## 표시 / 박는 것

```
헤더: 스트리머 DB
├── 검색 / 필터 (이름 / 게임 / 포지션 / 등급)
├── streamers list (table 또는 grid)
│   ├── 각 row: 이름 / 채널 / 포지션 / rating / marketValue / 활성
│   └── row 클릭 → 편집 drawer / modal
├── 새 스트리머 박는 button
└── 일괄 작업 (alias 정리 / channel 갱신 등)
```

## 스키마 (`streamers/{streamerId}`)

```js
{
  id: "peak",                       // unique short id
  name: "PEAK",
  channels: {
    chzzk: "channel-hash",          // 치지직 채널
    twitch: "username",
    youtube: "@channel"
  },
  position: "Top",                  // LoL: Top/Jng/Mid/ADC/Sup, OW: 다른
  rating: 95,                       // 0-100
  marketValue: 570,                 // pt
  profileImageUrl: "...",
  riotId: "...#KR1",                // LoL Riot ID
  bnetId: "...",                    // 오버워치 battle.net
  alias: ["옛이름", "별명", ...],   // raw OCR 매칭용
  meta: {
    firstSeen: "ISO datetime",
    lastSeen: "ISO datetime"
  }
}
```

자세히는 [[90_AI/프로젝트/pickzzk/위키/데이터-모델\|데이터-모델]] / [[90_AI/프로젝트/pickzzk/위키/기능/streamerDB\|streamerDB]].

## 박는 흐름

```
1. /admin/streamers 진입
   ↓
2. 새 스트리머 박음 (id 고유 / name / 채널 / 포지션)
   ↓
3. (옵션) 게임 ID 박음 (Riot ID / Battle.net)
   ↓
4. alias 박음 (옛이름 / 별명) — OCR 매칭률 향상
   ↓
5. profile image url 박음 (Vue Avatar 가 fallback 처리)
   ↓
6. 저장 → streamers/ 박힘
   ↓
7. 운영자 [[운영자-셋업]] 의 출전 명단 cherry-pick 에서 사용
```

## chzzkPlayer alias 자동 저장 (후속)

> 후순위 잔여 작업 ([[../할일/README\|../할일/README]] 참조).

OCR 박힌 raw 이름 → user 가 chip click 으로 매칭 → `scrimPlayerAliases/{chzzkName}.aliases` append.

→ 시즌 거듭하며 매칭률 자동 향상.

자세히는 [[90_AI/프로젝트/pickzzk/위키/기능/streamerDB\|streamerDB]].

## LoL + OW 공유

- 한 streamer = 한 streamers/ 박힘
- LoL 시즌 박힘 후 OW 시즌 박힐 때 같은 streamerId 재사용
- `position` 은 LoL / OW 다름 — channel 별 분기 가능 (TBD — 코드 검색)

## profile image 정책

- `profileImageUrl` 박힘 = url
- 박힌 거 없음 = `PlayerAvatar` 컴포넌트가 fallback (initial + 색)
- 자세히는 [[90_AI/프로젝트/pickzzk/위키/v10-톤\|v10-톤]] 의 component 패턴

## composable

- `useStreamersList(filterRef)` — list onSnapshot + filter
- `useStreamerEditor(streamerId)` — 1 streamer editor
- `useChzzkFetch()` — 치지직 비공식 API (채널 정보 가져옴)
- `useAdminGate()` — 권한 체크

## CLAUDE.md 우선순위

> 화면 우선순위 #3 (활성 홈 / 운영자 셋업 다음).

자세히는 [[90_AI/프로젝트/pickzzk/위키/개요\|개요]].

## 후속 작업 후보

- 일괄 alias 정리 UI
- 치지직 채널 자동 link / 갱신 (`useChzzkFetch`)
- 포지션 / 등급 변경 history
- 시즌별 streamer 출전 history
- LoL / OW 게임 ID 자동 검증 (Riot API / Bnet)

## 헷갈리는 점

> [!warning] streamerId = unique short id
> "peak" 같은 짧은 alias. uid 아님. user verbatim "uid 표시 / 로깅 금지" 룰.

> [!warning] roster = streamerId list
> 토너먼트 `tournament.roster` 는 streamerId 만 박힘. 자세 정보는 streamers/ 에서 lookup.

> [!warning] alias 자동 저장 = 후순위
> 현재 수동 박음. R20+ 또는 운영 시 다듬을 후보.

> [!warning] OCR 결과 챔프 = 한글 text primary
> 챔프 인식은 streamerDB 와 별개. 한글 text OCR 1차, ONNX classifier cross-check / fallback.

## 관련

- [[90_AI/프로젝트/pickzzk/위키/페이지/운영자-셋업\|운영자-셋업]] — `/admin/setup` 출전 명단 cherry-pick
- [[90_AI/프로젝트/pickzzk/위키/페이지/시청자-제보\|90_AI/프로젝트/pickzzk/위키/페이지/시청자-제보]] / [[90_AI/프로젝트/pickzzk/위키/페이지/게임-업로드\|게임-업로드]] — OCR alias 매칭
- [[90_AI/프로젝트/pickzzk/위키/기능/streamerDB\|streamerDB]] — 마스터 DB 인프라
- [[90_AI/프로젝트/pickzzk/위키/기능/OCR\|OCR]] — chzzkPlayer alias 매칭
- [[90_AI/프로젝트/pickzzk/위키/데이터-모델\|데이터-모델]] — streamers/ schema
- [[90_AI/claude/개념/스킬\|스킬]] / `screen-streamers-db`
