---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/위키/기능/streamerDB.md","permalink":"/프로젝트/pickzzk/위키/기능/streamerDB/","tags":["프로젝트/pickzzk","위키","기능","마스터-DB","LoL","OW"],"updated":"2026-05-14","dg-note-properties":{"tags":["프로젝트/pickzzk","위키","기능","마스터-DB","LoL","OW"],"project":"pickzzk","updated":"2026-05-14"}}
---


# streamerDB (마스터 컬렉션)

> pickzzk 의 단일 streamer truth. **LoL + OW 공유 — 시즌 거듭하며 누적**. alias / 채널 / 게임 ID / 평가가 한 docs 에 박힘.

## 한 줄

`streamers/{streamerId}` 컬렉션 = 모든 스트리머 정보 마스터. 시즌마다 새로 만들지 X — 기존 docs 갱신 / alias 추가.

## 스키마

```js
streamers/{streamerId}: {
  id: "peak",                       // unique short id
  name: "PEAK",                     // 표시 이름 (현재)
  nickname: "별명",                 // 표시용 별칭 (2026-05-22). 비우면 채널명 폴백.
                                    // 모든 화면 표시 이름. alias(OCR 매칭용)와 다름!
  channels: {
    chzzk: "channel-hash-or-id",    // 치지직 채널 (필수)
    twitch: "username",             // 옵션
    youtube: "@channel"             // 옵션
  },
  position: "Top",                  // LoL: Top/Jng/Mid/ADC/Sup
                                    // OW: Tank/DPS/Support (TBD)
  rating: 95,                       // 0-100 (운영자 박음 또는 누적 산출)
  marketValue: 570,                 // pt (auction 시 사용)
  profileImageUrl: "https://...",   // 옵션, 박혀 있으면 PlayerAvatar 가 사용
  riotId: "ABC#KR1",                // LoL Riot ID
  bnetId: "ABC#1234",               // 오버워치 battle.net
  alias: ["옛이름1", "별명", ...],  // OCR raw 매칭용 (nickname 과 혼동 X)
  meta: {
    firstSeen: "ISO datetime",
    lastSeen: "ISO datetime",
    seasonsActive: ["2024-spring", "2026-summer", ...]
  }
}
```

자세히는 [[90_AI/프로젝트/pickzzk/위키/데이터-모델\|데이터-모델]].

## 사용 영역

### 1) 운영자 셋업 ([[90_AI/프로젝트/pickzzk/위키/페이지/운영자-셋업\|운영자-셋업]])

`tournament.roster = [streamerId, ...]` — cherry-pick. streamersDB 에서 명단 박음.

```
1. 셋업 진입
2. streamersDB list 표시
3. 운영자 cherry-pick → tournament.roster
4. 필요한 정보 (position / rating / marketValue) lookup
```

### 2) OCR alias 매칭 ([[90_AI/프로젝트/pickzzk/위키/기능/OCR\|OCR]])

OCR raw 닉네임 → alias 매칭 → streamerId 박힘.

```js
useStreamersList() → streamers[] list
→ alias map 만듦: { "raw OCR name": "streamerId" }
→ visionToGameInput(parsed, streamers, teams, aliasesByName) 호출
```

> [!warning] aliasesByName 현재 `{}` (미연결)
> 후순위 잔여 ([[../할일/README\|../할일/README]]): `useStreamersList` → alias map 로드 → 4 번째 인자 전달. 매칭률 향상.

### 3) 활성 홈 표시

활성 홈 의 카드들이 streamerId 받아 streamersDB lookup:
- 챔프 picks → playerId → streamersDB → name / position / rating
- MVP → playerId → streamersDB → name + profileImage
- top 인기 매물 → streamerId → name

### 4) PlayerAvatar 컴포넌트

`src/components/shared/PlayerAvatar.vue`:
- `streamerId` 또는 `profileImageUrl` 받음
- 박혀 있으면 image 표시
- 없으면 fallback (initial + 색 — name 의 첫 글자)

## 시즌 거듭하며 누적

> 새 시즌 시작 시 streamersDB 초기화 X. 같은 streamerId 재사용.

- 옛 시즌 streamerId="peak" → 새 시즌도 "peak"
- alias 새 박힌 거 add (옛 이름 / 별명)
- rating / marketValue 갱신 가능 (시즌 결과 누적)
- meta.seasonsActive[] push

## LoL + OW 공유

> 하나의 streamer 가 LoL + OW 둘 다 박힘 가능.

```js
{
  id: "peak",
  position: "Top",   // LoL primary
  riotId: "PEAK#KR1",
  bnetId: "PEAK#1234",   // OW 도 같이 박힘
  ...
}
```

- position 은 LoL primary (OW position 별 박힘 검토 — `positionByGame` 같은 nested 가능)
- channel 은 공유

## composable

- `useStreamersList(filterRef?)` — list onSnapshot + filter
- `useStreamerEditor(streamerId)` — 1 streamer editor
- `useChzzkFetch()` — 치지직 비공식 API (채널 정보 가져옴)
- `usePlayerRatings(results)` — rating 누적 산출 (v2 — 후속 작업)

자세히는 [[90_AI/프로젝트/pickzzk/위키/구조\|구조]].

## chzzkPlayer alias 자동 저장 (후순위)

> 메모리 + 잔여:
> "사용자 confirm → alias 자동 저장 — chip click / 직접 입력 시 `scrimPlayerAliases/{chzzkName}.aliases` 박음 (legacy ScrimSubmitView 패턴). 시즌 거듭하면서 매칭률 자동 향상."

### 두 컬렉션 분리?

`scrimPlayerAliases/{chzzkName}` = alias 누적 (시즌 거듭하며)
`streamers/{streamerId}.alias[]` = streamerDB 내 alias 직접 박음

→ 둘 어떻게 통합? (TBD — 코드 검색 필요)

## 위키 우선순위 (CLAUDE.md)

> 화면 우선순위 #3 (활성 홈 / 운영자 셋업 다음).

자세히는 [[90_AI/프로젝트/pickzzk/위키/개요\|개요]].

## R20+ Codex 잔여

> AdminSubmissions winner 라벨 hard-code (`AdminSubmissionsView.vue:221`) — streamersDB 와 연관 (시즌별 동적 매핑).

자세히는 [[../할일/README\|../할일/README]] / `.claude/CODEX_REVIEW.md`.

## 후속 작업 후보

- 일괄 alias 정리 UI (관리 편의)
- 치지직 채널 자동 link / 갱신 (`useChzzkFetch`)
- 포지션 / 등급 변경 history (audit)
- 시즌별 streamer 출전 history
- LoL / OW 게임 ID 자동 검증 (Riot API / Bnet)
- alias 자동 학습 (scrimPlayerAliases 통합)

자세히는 [[../할일/README\|../할일/README]].

## 헷갈리는 점

> [!warning] streamerId = unique short id
> "peak" 같은 짧은 alias. uid 아님. user verbatim "uid 표시 / 로깅 금지" 룰.

> [!warning] streamerDB vs scrimPlayerAliases
> - **streamerDB** = 마스터 (운영자 박음)
> - **scrimPlayerAliases** = 시즌 OCR 누적 (자동 학습)
> 두 system 통합 = 후순위 잔여.

> [!warning] LoL + OW 공유 = 같은 streamerId
> 게임 분리 X. 한 docs 에 둘 다 박힘.

> [!warning] position = LoL primary
> OW position 별 박힘 검토. 현재 = LoL 만 (TBD).

> [!important] nickname(표시용) ≠ alias(OCR 매칭용) — 2026-05-22
> `nickname` 은 **사람에게 보여주는 별명** (비우면 채널명 폴백). 모든 화면 표시 이름이 여기서 나온다(helper `src/utils/displayName.js`).
> `alias` 는 **경기 스크린샷 자동 인식(OCR) 매칭에 쓰는 보조 키**. (OCR 매칭의 핵심 키는 라이엇 게임명 = riotId 의 #태그 뗀 부분. 채널명/alias 는 보조 타겟.) 별칭을 OCR 매칭에 섞으면 인식이 깨진다.
> 자세히 [[../결정/2026-05-22_표시용-별칭-nickname\|../결정/2026-05-22_표시용-별칭-nickname]]

> [!info] 솔로랭크 세부 지표 수집 — 2026-05-24
> `fetch:riot` 이 경기별 킬관여·분당CS·솔로킬·분당딜·시야·데스 시간대(deathBuckets)·최근 10경기·연승연패·챔프 폭 등을 뽑아 streamer 도큐에 저장하고, `fetch:ai` 가 그걸 rich aiSummary(AI 점수·스타일 분포·강약점 배열·데스 해석)로 분석한다. 선수 상세 모달 5탭이 이걸 표시. 활성 직전 1회 실행으로 최신화(Riot 키 24h 만료). 자세히 [[../결정/2026-05-24_선수-상세-모달-rich-파이프라인\|../결정/2026-05-24_선수-상세-모달-rich-파이프라인]] · [[90_AI/프로젝트/pickzzk/위키/기능/모의-시뮬\|모의-시뮬]]

## 관련

- [[90_AI/프로젝트/pickzzk/위키/페이지/스트리머-DB\|스트리머-DB]] — `/admin/streamers` 페이지
- [[90_AI/프로젝트/pickzzk/위키/페이지/운영자-셋업\|운영자-셋업]] — cherry-pick
- [[90_AI/프로젝트/pickzzk/위키/기능/OCR\|OCR]] — alias 매칭
- [[90_AI/프로젝트/pickzzk/위키/기능/시청자-제보\|90_AI/프로젝트/pickzzk/위키/기능/시청자-제보]] / [[90_AI/프로젝트/pickzzk/위키/페이지/시청자-제보\|90_AI/프로젝트/pickzzk/위키/페이지/시청자-제보]] — alias 학습
- [[90_AI/프로젝트/pickzzk/위키/데이터-모델\|데이터-모델]] — streamers/ schema
- [[90_AI/claude/개념/스킬\|스킬]] / `screen-streamers-db`
- [[90_AI/claude/개념/메모리\|메모리]] / `streamer_db_master_2026-04-29` (TBD)
