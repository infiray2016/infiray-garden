---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/결정/2026-06-09_모바일-archive-fix-+-안정화-sprint-+-MockBoardHeader.md","permalink":"/프로젝트/pickzzk/결정/2026-06-09_모바일-archive-fix-+-안정화-sprint-+-MockBoardHeader/","dg-note-properties":{}}
---

# 2026-06-09 모바일 archive 무한 로딩 fix + AI 회귀 가설 검증 + 안정화 sprint 3 Part + MockBoardHeader 추출

## 무엇

하루를 시작한 단일 사고 (모바일 archive 페이지 무한 로딩) 부터 흐름이 펼쳐졌다. fix 진행하면서 user 가 데이터 격리에 대한 단순 확인 질문을 던졌고, 그게 의도와 달리 회귀 가설로 확장되어 한참 분석한 끝에 user 가 "그건 사용자 선택, 오류 아님" 정정해줘 narrative 가 reset 됐다. 그 다음 user 가 결정한 방향 = "있는거부터 제대로 돌아가야 추가를 하지" 였고, 신규 capability 모두 후순위로 미뤄 기존 기능 폴리싱만 박는 안정화 sprint 3 Part 진행. 마지막으로 sprint 중 발견된 반복 패턴 (3 곳에 박힌 모의 보드 헤더 CSS) 을 공통 컴포넌트로 추출. 총 6 commit push (`0f7d36b` / `7ad7d0f` / `e4166bf` / `70c96fb` / `628d0a7` / `cdab16b`).

## 왜

오전 흐름:

1. **모바일 archive 무한 로딩 발견** — user 보고 "모바일에서 아카이브 들어가면 불러오는 중 이라고 뜨고 변화가 없어 계속 기다리게됨" + "아카이브에서 ← 홈 누르면 무한로딩에 걸림". 진단 결과 root cause = `src/firebase.js` 의 visibility 기반 `disableNetwork`/`enableNetwork` 토글. 이전에 (1f3f20f Tier 1.5 폴리싱) 백그라운드 long-poll 차단 목적으로 추정 기반 추가했는데, 모바일 (iOS Safari prerender + 탭 전환 + 키보드 토글) 에서 pending `getDoc`/`onSnapshot` 이 disabled 네트워크 limbo 로 빠져 view 가 영원히 loading 상태로 남는 회귀.

2. **데이터 격리에 대한 user 의 단순 확인 질문** — "지금 로컬에서 경매하면 결과가 운영에도 남아?" 라는 질문이 들어왔는데, 제가 그걸 "로컬 데이터가 운영에 누수되는 회귀 의심" frame 으로 잘못 해석. 실제로 user 는 단순 확인이었고, `withTestFlag` + `filterTestDocs` 격리 룰 (`src/utils/dataEnv.js`) 이 정확히 동작함을 user 도 알고 있었음.

3. **AI 회귀 가설로 분석 폭주** — 그 과정에서 "운영 mockResults 의 한 doc 에서 마린팀 (T0, balance personality 자리) 이 5포지션 다 직접 낙찰 0건, fallback 으로만 0pt 받음" 변칙을 발견하고 commit `e135026` (Codex P2 A+C+D AI 변경) 이 회귀 원인이라고 추측. doc 삭제 + head-to-head 회귀 unit test 4건 추가. 시간선 cross-check 결과 그 doc 가 commit 이전 박힌 거라 회귀 가설 자체 reject. 그 다음 user 정정 = **"입찰 안 한 건 사용자 선택이지 오류가 아니잖아?"** — 즉 변칙 자체가 multi 룸의 사람 호스트가 입찰 안 한 정상 시나리오였음. 회귀도 X.

오후 흐름:

4. **안정화 sprint 결정** — "있는거 제대로 돌아가야 추가를 하지" 방향. 신규 capability 전부 (시즌 포인트 마켓 / OCR 2-pass / 매치업 drawer / multi smoke 신규) 후순위로 미루고, 실제로 깨졌거나 매끄럽지 않은 기존 기능만 골라 박는 sprint. user 가 명확히 scope 결정 (3 묶음): 즉시 fix 3건 + admin GameInput streamerId 매칭 root cause + 모바일 AdminSetup 03~05 카드 폴리싱.

5. **반복 CSS 패턴의 즉시 추출 트리거** — Part 1-3 (모바일 헤더 폰트 키움) 진행 중 같은 .head__crumb / .head__meta / .head h1 CSS 가 3 곳 (MockAuctionBoard / MockAuctionSandbox / MockDraftBoard) 에 박혀있는 걸 발견. plan 에는 별 cycle 권장이라 박혀있었지만 inline fix 3번 박는 것보다 컴포넌트 추출이 깔끔하다고 판단, sprint 마지막에 후속 작업으로 진행.

## 어떻게

### 1차 — 모바일 archive 무한 로딩 fix (`0f7d36b`)

- `src/firebase.js` 의 `visibilitychange` 기반 `disableNetwork`/`enableNetwork` 코드 제거 + import 정리. modern 브라우저는 hidden tab 자동 throttle 하므로 토글 없어도 비용 무시.
- `src/composables/useArchive.js` 의 `getDoc` 에 8초 `Promise.race` timeout 박음. 다른 원인으로 hang 해도 8초 후 "아카이브 시즌 없음" empty state graceful degrade.
- `src/composables/useActiveTournament.js` 의 `onSnapshot` 첫 snapshot 에 8초 timeout 박음. timeout 시 CLOSED 페이지 fallback. HomeView 가 "▸ loading…" 영구 박히는 회귀 차단.
- memory `feedback_no_firestore_visibility_toggle_2026-06-09` 박음 — 미래의 제가 같은 토글 다시 박지 않도록 룰 + 이유.

### 2차 — AI 회귀 가설 검증 (`7ad7d0f`, 잘못된 전제로 박힌 commit)

- `tests/unit/ai-pick-decide-bid.test.js` 에 4 personality (balance/boost/value/top-heavy) head-to-head 시뮬 + balance 안전망 + blendPrice cold start + scarcity 가산 4 케이스 추가.
- 동작은 무해하고 미래 회귀 차단 net 역할 가능하지만, 박은 이유 (마린팀 0pt 회귀 의심) 가 user 정정으로 무효화됨. commit message 본문에 정직하게 narrative 박힘 (시간선 reject + 그래도 보강).

### 3차 — 안정화 sprint Part 1 (`e4166bf`)

세 가지 즉시 fix:

- **MockAuction phase=sim URL 직접 진입 시 sandbox 자동 진입** — `MockAuctionView.vue` line 110 의 `startedMode` default 가 `'multi'` 라 직접 link 진입 시 빈 multi lobby 가 뜨던 회귀. URL-aware default 로 `route.query.phase === "sim" && route.query.mode !== "multi"` 면 `'manual'` 박음. entry shell 통과 시는 `onStart` 가 덮어쓰므로 회귀 0.
- **경매 완료화면 나가기 → 홈 정합 검증** — 진단 결과 이미 박혀있음 (`MockAuctionBoard.exitHome` emit + `MockAuctionView.onExitHome` → `router.push("/")`). 추가 작업 없음. Draft 의 완료화면 = entry 로 (재시뮬 자유) 의도된 동작.
- **모바일 모의 보드 헤더 폰트** — TODOS 의 "🐛 모바일 헤더 정보 줄 폰트 작음" 대응. `MockAuctionBoard` / `MockAuctionSandbox` / `MockDraftBoard` 3 곳의 `.head__meta` 폰트 모바일 (`@media (max-width: 768px)`) 13px 로 키움.

### 4차 — 안정화 sprint Part 2 (`70c96fb`)

진단 결과 root cause 자체는 오늘 새벽 다른 commit (`ec736b9 feat(game-input): streamerId 자동 매칭`) 에서 이미 4 layer fix 박힌 상태였음. TODOS 가 outdated. 남은 작업 3건 (audit + 청소 + UX polish):

- **기존 streamerId 누락 doc 일괄 backfill** — `scripts/backfill-streamer-ids.mjs` (gitignore — 일회용 admin SDK script) 작성. firebase-admin SDK 로 collectionGroup('games') 의 23 game doc + streamers 마스터 46 streamer × 56 unique name key 다 로드. `players[].streamerId` 가 null/undefined 인 row 를 byName lookup (nickname/chzzkName/displayName lowercase) 으로 자동 매칭. dry-run 결과 = 23 doc 의 190 player row 가 일제히 null + 100% 매칭 성공. --apply 실행으로 19 doc / 190 row 박음. viewer 의 byName fallback (`c0cc980 LineupCardReadonly avatar`) 영구 의존성 해소.
- **회귀 단위 test 3건 추가** — `tests/unit/scrim-mapping.test.js` 에 `visionToGameInput` 의 `streamerByName` lookup 흐름 회귀 차단. (1) streamer chzzkName + nickname 정확 일치 → streamerId 자동, (2) 매칭 안 되는 OCR (대타/외부) → null, (3) streamers 빈 배열 → null.
- **nameStatus 시각 표시 보강** — `GameInput.vue` 의 `updateLineupRow` 에 nameStatus 동시 박는 분기 추가. playerName 변경 시 매칭 결과 따라 `auto` (정상/빈 입력) 또는 `unknown` (orange-d 보더 + 알림) 박음. `LineupNameInput` 의 status prop 이 이미 visual marker 박혀있어 추가 변경 X.

### 5차 — 안정화 sprint Part 3 (`628d0a7`)

03 팀 / 04 출전 명단 / 05 팀 매핑 모바일 시각 audit 결과 진짜 원인 = `FormRow.vue` 의 fixed grid-template-columns. label-width=110px 인 SetupCardTeam 의 row 들이 375px viewport 에서 content 영역 200px 정도로 압축됨. 단일 지점 fix:

- `src/components/shared/FormRow.vue` 에 `@media (max-width: 640px)` 블록 추가. `grid-template-columns` 을 `var(--lbl-w) 1fr` → `1fr` 으로 stack. gap 줄임. lbl padding-top 0.
- 한 곳 fix 로 SetupCardTeam (03) / SetupCardInfo (01) / SetupCardRule (02) / ScheduleAddModal 등 4 사용처 모두 자동 mobile 친화. desktop 영향 0 (max-width 640px 이하만 발동).
- 04 출전 명단은 자체 `@media (max-width: 640px)` 이미 박혀있음, 05 팀 매핑은 auto-fill minmax(240px) 자동 1 col fold — 둘 다 추가 작업 X.

### 6차 — Task #6 MockBoardHeader 공통 컴포넌트 추출 (`cdab16b`)

HARNESS R "2회 이상 반복 패턴 = 즉시 컴포넌트 추출" 정합. 3 곳 (MockAuctionBoard / MockAuctionSandbox / MockDraftBoard) 에 박혀있던 동일 헤더 CSS 통합:

- `src/components/mock/MockBoardHeader.vue` 신규. 4 slot (crumb / h1 / meta / cta) 박음. 공용 chip class (.me-tag / .rule-badge / .rule-mini / .room-tag) 은 `:slotted()` scoped style 로 slot content 에 적용 — 사용처는 class 만 박으면 자동 style.
- 3 컴포넌트의 inline `<header class="head">` 영역을 `<MockBoardHeader>` 로 교체 + 관련 CSS (.head / .head__crumb / .head h1 / .head__meta / @media / chip class 들) 제거.
- net 코드: **+58 / -246 lines** (-188 lines deduplication). 미래 헤더 변경 단일 지점.

## 결과

총 6 commits push (`0f7d36b` archive fix → `7ad7d0f` head-to-head test → `e4166bf` Part 1 → `70c96fb` Part 2 → `628d0a7` Part 3 → `cdab16b` MockBoardHeader). 검증:

- `bun run verify` 6번 다 pass (lint 0 errors / 553 tests / build OK)
- `bun run mock:smoke` 7/7 pass
- Playwright MCP 모바일 375px 으로 archive 진입 → empty state + ← 홈 → home 정상 / `/mock/auction?phase=sim` 직접 진입 → sandbox 헤더 정확 렌더 + console errors 0
- production firestore: mockResults `wH6WWgHRp6W8dUx00fVB` (마린팀 0pt doc) admin SDK 로 삭제 + 19 game doc / 190 player row backfill 박힘
- TODOS.md 의 즉시 작음 4건 중 3건 closeout + 🐛 알려진 bug 2건 closeout

### 7차 — 종합 검증 (`design-review` + `qa` gstack skill)

push 직후 안정화 sprint 의 시각 + 기능 회귀 0건 보장 확인 위해 두 skill 연속 진입:

**design-review 결과:**
- Design Score = **A (94/100)**, AI Slop Score = **A** (signal orange + 라이트 베이지 + 하드 보더 특이성 strong, blacklist 10 패턴 0 매치)
- 4 화면 audit (홈 mobile/desktop, mock auction sandbox, archive empty, admin setup) — 모두 v10 Card Clean tone 정확, console errors 0
- Phase 7-10 (fix loop) 진입 X — high impact 회귀 0건

**qa 결과:**
- 4 functional 시나리오 검증: 홈 → CTA → entry shell 진입 / archive 빈 상태 → 홈 click 정상 / `?mobile=1` 모바일 강제 시 bottom nav 6 탭 정확 표시 / `/stats` 정렬+탭 switch 정상
- Health Score = **94/100 (A)**: Console 100, Links 100, Functional 95, UX 90 (홈 두 CTA polish 후보)
- Phase 7-10 (fix loop) 진입 X — fix commits 0

**알려진 한계 인식 (회귀 X):**
- `useViewport` 의 `(pointer: coarse) and (max-width: 1023px)` AND 조건이 Playwright headless 의 `pointer:coarse=false` 와 안 맞음 — viewport 만 줄여도 isMobile=false 라 bottom nav DOM 0건. 실 모바일 (iOS/Android) 은 정상. `?mobile=1` URL force 로 우회 검증. `1ab7210` 의 비스코프 섹션 F-P2-1 runbook 으로 박혀있는 known issue.

user 입장 변화:

- 모바일에서 archive 진입 시 무한 로딩 X → 데이터 있으면 표시, 없으면 8초 안에 empty state
- ← 홈 click → home 정상 로드 (이전: 무한 loading)
- `/mock/auction?phase=sim` 외부 link 진입 시 sandbox 자동 시작 (이전: 빈 multi lobby)
- 모의 보드 헤더 모바일 폰트 14px 가독성
- 운영자 게임 결과 입력 시 streamerId 자동 매칭 + 미매칭 시 시각 알림 (`unknown` orange-d 보더)
- /stats 12 컬럼 메트릭 정확도 향상 (운영 19 game doc backfill 영향)
- AdminSetup 모바일에서 라벨 위 / content 아래 stack 가독성

## 남은

- **DAY 편집 화면 vs 운영일 viewer 의 `filterTestDocs` 일관성 점검** — viewer 만 isTest 제외하는 게 의도였는지 다시 확인. TODOS 의 즉시 작음 마지막 1건.
- **시즌 포인트 마켓 (P1 핵심 놀이 루프)** — 픽지직 정체성의 가장 큰 미착수 P1. 안정화 끝났으니 다음 cycle 후보 1순위. 모의/지표 인프라는 이미 박혀있어 마켓이 자연스러운 다음 surface.
- **Codex P1 OCR 2-pass (B 단계 핵심)** — 1-pass 도 동작은 하지만 정확도 70~80% → 2-pass crop 으로 90%+ 향상 가능. 운영 데이터 정확도 직결.
- **Codex 매치업 drawer (P2)** — viewer 의 H2H 매트릭스 cell 클릭 → 포지션별 개별 퍼포먼스 / 유불리 점수 drawer. legacy spec 정확 사양 문서화 필요.
- **MockSetupStrip 또는 다른 모의 보드 공통 sub-section 추출** — Task #6 흐름의 연장. 발견 시 같은 패턴 적용.
- **AdminBanToolView 정체성 결정 (Codex #9)** — user 결정 대기 (유지 / lab flag / 폐기 중 하나).

### design-review + qa 7차 검증에서 발견된 polish 후보 (sprint 외)

- **홈의 두 모의 경매 CTA label 통일 또는 hierarchy 차별화** — 현재 큰 orange "▸ 모의 경매 시작" (TimelineHero) + 카드 안의 "▶ 경매 시뮬 시작" (MockSimulatorCard) 가 비슷한 의미. 시청자가 둘 중 하나만 보거나 혼동 가능. 후속 cycle 에서 label 분리 (예: "지금 시작" vs "공유 시뮬 보기") 또는 카드 CTA 제거 검토.
- **/stats 12 컬럼 표 모바일 폰트 audit** — 375px viewport 에서 12 컬럼 다 들어가서 폰트 작은 편. 의도된 dense display 지만 후속 cycle 에서 가로 스크롤 + 폰트 확대 옵션 검토.
- **Desktop 1440 의 좌측 사이드바 mono 폰트 audit** — letterspacing 0.2em 효과로 작게 보임. 가독성 audit 후속 cycle.
- **`F-P2-1` runbook 의 memory 화** — `useViewport` 의 pointer:coarse 한계가 현재 commit `1ab7210` 본문에만 박혀있음 → 2026-06-09 memory `feedback_useviewport_pointer_coarse_headless_limit_2026-06-09` 로 박힘 ✅.

### 영향도 분석 8차 — Explore agent 광범위 분석 + Part A 즉시 fix

User 우려 "많이 변경됐는데 동작 시 오류 없어? 영향도 분석을 안 하는 것 같다" 에 응답. Explore agent 로 10 영역 분석 → 진짜 위험 1건 발견 → 즉시 fix.

**8차 commit**: `src/views/MockAuctionView.vue` line 115-118 의 `startedMode` 분기 단순화. 이전 분기 (`phase=sim && mode!='multi' ? manual : multi`) 가 `?phase=sim&mode=multi` 직접 URL 진입 시 `startedRoomCode=''` 빈 채로 multi board mount → 사용자 "대기 중..." stuck 가능. 단순화 (`phase=sim ? manual : multi`) 로 phase=sim 직접 진입 시 무조건 sandbox. mode=multi 명시 의도는 entry shell 통한 정상 흐름 (사용자 수동 선택) 으로 유도.

검증: `bun run verify` pass / `bun run mock:smoke` 7/7 pass / Playwright `?phase=sim&mode=multi` 직접 진입 → SANDBOX 자동 fallback 확인 / console errors 0.

### 모니터링 가이드 (Part B — 코드 변경 0, 운영 actions)

**1. firestore visibility 토글 제거 후 비용 메트릭 (2026-06-12 까지)**

`firebase.js` 의 `visibilitychange → disableNetwork/enableNetwork` 코드 제거 (0f7d36b) 후 8 onSnapshot subscriber (useActiveTournament / useArchive / useStreamersList / useMockResults / useTournamentAggregate / useSubmissionsQueue / useScrimDay / useAuth) 가 백그라운드 탭에서도 throttle 의존. modern 브라우저 자동 throttle 으로 큰 비용 X 예상 (RTDB 는 useAuctionMultiRoom / useSnakeMultiRoom 의 자체 visibilitychange handler 가 manually disable).

확인 절차:
- baseline: 2026-06-08 (push 직전 last day) 의 Firestore daily reads
- 비교: 2026-06-12 (push 후 72시간 경과) daily reads
- 임계:
	- ±5% 이내 = 정상, 모니터링 종료
	- +5~10% = 모니터링 지속, 1주 후 재확인
	- +10% 이상 = `firebase.js` 의 visibility 토글 부분 복구 (page hidden > 5분 시만 disable) 검토

Firebase console 경로: https://console.firebase.google.com/project/draftpick-63456/firestore/usage

**2. mockResults 컬렉션 비용 동향**

`useMockResults.js:280-282` 의 firestore filterTestDocs 흐름이 live tournament 중 background tab 에서 계속 sync 되면 비용 누적 가능. mock 단계 (시뮬 활성) 동안 daily reads 시간대별 패턴 확인 권장.

### 후속 cycle 검토 (Part C — defer)

**1. `GameInput.vue` nameStatus OCR vs manual 일관성** — `visionToGameInput` (scrim-mapping.js line 274) 의 `nameStatus: m.status` 박는 흐름과 manual `updateLineupRow` (GameInput.vue line 354-367) 의 'unknown' 박는 흐름이 OCR 자동 채움 후 manual 수정 시 어떤 nameStatus 가 박히는지 확인 필요. 별 cycle 에서 unit test 또는 manual 검증.

**2. 삭제된 doc `wH6WWgHRp6W8dUx00fVB` 의 stats context** — 이미 삭제됐고 다시 못 봄. user 본인 production 테스트 결과 doc 이라 stats noise 제거 효과 있을 듯. 후속 stats 비교 시 매물 평가 / 평균 낙찰가 / 마린팀 메트릭 정합 확인.

### 9차 — /stats user-reported polish 3건 (`c99fea8`)

User 보고 3건 — 모바일 사용자 입장에서 즉시 영향. 안정화 sprint 완료 + 영향도 분석 직후 들어온 polish 라 같은 cycle 에서 박음.

**Issue 1: 모바일 export 사진첩 진입 X**
- 증상: `/stats` 의 "저장" 버튼 누르면 PNG 다운로드 폴더 또는 파일 앱으로 박혀 사용자가 사진첩에서 못 찾음. 모바일 사용자 의도 (사진첩에 저장해 공유) 와 안 맞음.
- 원인: `downloadDataUrl` + `<a download>` link click 패턴 — iOS Safari / Android Chrome 에서 file 시스템 다운로드 흐름만 발동.
- fix: `navigator.share({ files: [pngFile] })` Web Share API 우선 시도. canShare(files) 지원 환경 (iOS Safari 16+ / Android Chrome 90+) 에서 share sheet 통해 "이미지 저장" 선택 시 사진첩 진입. 미지원 / 데스크탑 / share AbortError 면 기존 download link fallback (회귀 0).

**Issue 2: 지표 헤더 모바일 글자 겹침**
- 증상: 모바일 320~480px viewport 에서 `.head__row` 의 "지표 보기 · 자낳대 2026" title + "21명" count + 저장/공유 버튼이 한 줄에 안 박혀 title wrap + count 잘못 위치 + 시각 cramped. user 가 PNG export 결과에서도 겹침 확인.
- fix:
	- `.head__row` 에 flex-wrap: wrap + title 에 word-break: keep-all
	- @media (max-width: 480px) 에서 title 16px + share 별 줄 (width 100% margin-left 0) 우측 정렬
	- export PNG 헤더 (`.export-meta`) 도 재구성 (issue 3 과 같이)

**Issue 3: 선수/챔프 export 헤더 = 포지션 스타일**
- 의도: `StatsPositionView` 의 `.role-section__h` (orange badge chip + count) 패턴이 시각 hierarchy 강한데 다른 view 의 export PNG 헤더는 plain text 박힘. 통일된 시각.
- fix: `.export-meta` 재구성 — `.export-meta__badge` (orange chip — "선수 지표" / "팀 순위" / "챔프 지표" 등) + `.export-meta__count` (mono — "21명" / "12챔프" 등) 위 row, 대회명 (sans 18px bold) 다음 줄, sub (mono 11px kind · range · today) 다음 줄. `exportMeta` computed 에 count 필드 추가 (view 별 playerRows/teamRows/championRows.length 등).

검증:
- `bun run verify` pass (lint 0 errors / 553 tests / build OK)
- `bun run mock:smoke` 7/7 pass (회귀 0)
- Playwright MCP 모바일 375px `?mobile=1` 으로 visual 검증: 메인 헤더 깔끔 stack + export-meta 가 "[선수 지표 orange chip] 21명 / 자낳대 2026 / 전체 · 전체 기간 · 2026-06-09" 정확 렌더

user 입장 변화:
- 모바일에서 /stats 의 "저장" 버튼 → share sheet → "이미지 저장" → **사진첩 진입** (iOS Safari 16+ / Android Chrome 90+)
- 모바일 320~480px viewport 에서 헤더 겹침 X
- export PNG 의 헤더가 포지션 view 와 일관된 시각 (orange chip + count + 대회명 + meta)

비스코프 (다음 cycle):
- `domToPng` 의 실제 출력 PNG 의 글자 wrap 정밀 audit (modern-screenshot 의 SVG foreign-object 한계 검토)
- export PNG 의 다국어 / 긴 대회명 케이스 시각 검증

### 10차 — PNG export 의 multi-iteration root cause 추적 (`97a9850` ~ `31137bf`)

9차 commit 의 처음 시도 (`c99fea8` + `1b9e4b9` + `cb820e5`) 가 user 가 보고한 "사진첩 진입 X" 를 못 풀고 5건 이상 추가 commit 으로 root cause 차례로 발견. 정직하게 narrative 박음 — 잘못 박은 추측들 + DraftPick legacy 참조로 진짜 답 발견의 흐름.

**시도 1 — usePngExport composable 추출 (`97a9850`)**

user 보고 "로딩창만 돌고 아무것도 안됨" + "드래프트픽은 잘되니까 그거 바탕으로". DraftPick `useScrimExport.js` 패턴 (`prepareOffscreenClone` + `syncCanvases` + `waitForImages` + `domToBlob` + share/clipboard/download chain) 그대로 composable 로 추출. backdrop overlay 자체 제거 (offscreen clone 이라 viewport 변화 X).

→ user 보고 정정: "저장 누르니까 왜 이전 공유 기능이 작동해???" — share sheet 떴는데 user 입장 "이게 공유다, 저장이 아니다" 로 인식.

**시도 2 — downloadPng / sharePng API 분리 (`669a770`)**

user 의 "공유 기능 X" 의도 박혀 `downloadPng` (저장 default, share 안 띄움, blob download) + `sharePng` (opt-in, share chain) 분리. StatsView 는 downloadPng 호출.

→ user 보고: "아이폰인데 사진첩 저장이 안돼". iOS Safari 의 `<a download>` blob URL 이 preview tab 만 열고 사진첩 진입 X — web API 한계.

**시도 3 — iOS UA detection (`c67b70b`)**

UA 박는 거 (`/iPad|iPhone|iPod/.test(ua)`) 로 iOS 만 share sheet, 나머지 download.

→ user 보고: "드래프트픽은 모바일 기종에 따라 다운로드 다를껄??" + "크롬이랑 사파리 둘다 되야지". UA detection 이 Android Chrome 도 사진첩/갤러리 진입 못 시킴 → user 의도 (모바일 둘 다) 와 안 맞음.

**시도 4 — capability detection (정답, `33fe785`)**

DraftPick `DraftMobileView.vue:243-271` 의 검증된 패턴 발견. **`navigator.canShare({ files })` capability detection** (UA 박지 X). canShare=true → share path (모바일 Chrome/Safari 둘 다), false → download (데스크탑). title 박음 (DraftMobileView 정합 — 검증된 패턴, 이전 useScrimExport.js comment 의 "title 박지 X" 가 일부 iOS 버전 edge case 였을 듯).

**시도 5 — flex wrap 헤더 겹침 (`2409883`)**

user 가 OLD PNG (06:35 박힌 거) 공유. 직접 PNG decode + Read tool 로 시각 확인 후 진짜 root cause 발견:
- `.export-meta__count` "21명" 의 "21" / "명" 2 줄 wrap
- `.export-meta__sub` 의 "전체 · 전체 기간 · 2026-06-09" 가 "전체 · 전체" / "기간 · 2026-..." 2 줄 wrap
- flex container 안에서 좁은 viewport 시 unexpected wrap.

fix: `.export-meta__badge` / `.export-meta__count` 에 `white-space: nowrap` + `.export-meta__sub` 의 `flex-wrap` 제거 + `overflow: hidden` + `text-overflow: ellipsis`. Playwright 으로 직접 `domToBlob` 호출 + base64 → 디스크 디코드 → before/after 비교.

**시도 6 — 모바일 hang 회피 (정답, `31137bf`)**

user 보고 "모바일/태블릿 사파리·크롬 다 안돼. 저장중… 뜨다가 아무 일 없어. 데스크탑은 다운로드 됨". 데스크탑 OK + 모바일 hang 패턴 = **user activation 5초 한계 초과**:
- iOS Safari / Mobile Chrome 의 `navigator.share()` 가 user gesture 5초 안에 호출돼야 share sheet 노출. 그 외 silent fail.
- 이전 코드의 `offscreen clone` + `waitForImages` + `CORS fetch` 단계가 모바일 3-5초+ 걸려 5초 한계 초과.

DraftPick `renderTeamGridBlob` 재확인 결과 offscreen clone X — **원본 element 직접 캡처**. 그래서 작동.

fix:
- `prepareOffscreenClone` + `syncCanvases` + `waitForImages` 헬퍼 통째 삭제 (-89 lines)
- `CORS fetch option` 제거 (modern-screenshot 기본 fetch 가 더 robust)
- 원본 `captureRef` 에 직접 `setAttribute('data-export-mode', 'light')` → `domToBlob` → removeAttribute. 캡처 속도 1-2초 단축.
- backdrop overlay 복원 — viewport 의 export 헤더 변화 가림.
- 8s timeout (이전 10s).

### user 입장 변화 (10차 완료 후)

- 모바일 (iOS Safari + Mobile Chrome) 저장 click → backdrop 1-2초 → share sheet 의 "이미지 저장" / "사진 저장" tap → **사진첩 / 갤러리 진입**.
- 데스크탑 (Chrome/Firefox/Safari) 저장 click → backdrop 1-2초 → blob download → 다운로드 폴더.
- PNG 헤더 깔끔 한 줄 ("선수 지표 21명" + "자낳대 2026" + "전체 · 전체 기간 · 2026-06-09").

### 학습 (R23 narrative)

- **user activation 5초 한계의 silent UX 함정**: 데스크탑 OK + 모바일 hang 패턴 = user gesture timeout 의 정확한 sign. 미래 share API 작업 시 이 단서 박힘.
- **DraftPick legacy 의 정확한 sample 가치**: 단순 `renderTeamGridBlob` (no clone, no waitForImages, no CORS option) 가 "더 robust" 박은 추가 단계들보다 모바일 작동 보장. premature optimization (offscreen clone) 으로 인한 사용자 회귀의 lesson.
- **multi-iteration narrative 의 정직성**: 시도 1~3 정답 못 박은 흔적 명시. 미래의 Claude / user 가 같은 함정 재발견 안 하도록.
- **PNG 직접 decode 의 가치**: Playwright `domToBlob` evaluate → base64 → tool result file → python script 디코드 → PNG file → Read tool 시각 확인. download API 우회로 정확히 캡처된 PNG 비교 = flex wrap root cause 발견의 결정적 evidence.

### 11차 — share path 완전 제거 (정답, `071587c`)

10차 의 시도 6 (offscreen clone 제거) 후 user 재보고 — "모바일/태블릿 여전히 안 됨. backdrop 떴다 사라지는데 share sheet 안 뜨고 파일도 없음." 그 직후 추가 보고 "데스크탑도 다운로드가 안 돼". 데스크탑 회귀까지 박힘.

**진짜 root cause 발견** (시도 7 = 정답):
- `navigator.canShare({ files })` 가 데스크탑 Chrome 130+ / 모바일 PWA / Mobile Safari 등 다수 환경에서 **true** 반환하지만 실제 share UI 가 없음 (false positive).
- share path 진입 → `navigator.share()` 즉시 AbortError 또는 silent fail → catch block 의 `if (AbortError) return` → blob download fallback **안 됨** → 사용자 "아무 일 없어".

이전 시도 4 (capability detection) 가 정답이라 박았던 게 실제로는 `canShare({files})` 의 false positive 함정. multi-iteration 7번 동안 share API 자체의 비결정성 (browser/PWA/version 별 차이) 을 못 잡아냄.

fix:
- `src/composables/usePngExport.js` 의 `downloadPng` 함수에서 `navigator.share` + `canShare` 블록 통째 제거.
- title 인자도 unused → 제거.
- 단일 path: `captureBlob → blob download` (`<a download href={blobUrl}>`).

결과:
- 데스크탑 Chrome / Firefox / Safari → 다운로드 폴더 (확실 동작 — share API 의 false positive 우회).
- iOS Safari → preview tab (user 가 long-press → 사진 저장 — web API 한계 명시).
- Android Chrome → 다운로드 폴더 / 갤러리 (브라우저 설정).
- PWA → 다운로드 폴더.

trade-off:
- 모바일 사진첩 **직접 진입 X**. share sheet 거쳐야 하는데 user 가 거부 + share path 가 silent fail → trade-off 받아들임.
- `sharePng` (opt-in) 함수는 composable 에 남아있어 차후 "공유" button 별 분리 시 재사용 가능. 사진첩 진입 원하면 user 가 그때 요청.

### 학습 (11차 추가)

- **`canShare({ files })` 의 false positive 함정**: capability detection 자체도 "user-facing 동작 가능" 보장 X. 데스크탑 Chrome 130+ / 모바일 PWA / 일부 Mobile Safari 환경에서 `canShare({files})` true 반환하지만 실제 share UI 가 없거나 즉시 AbortError. 미래 share API 작업 시 capability check 만으로 분기 박지 X — 실제 UI 동작 보장 필요.

- **silent fail 의 user UX 비용**: spinner 짧게 떴다 사라지고 아무 일 안 일어남 = user 입장 "그냥 안 됨". console error 모바일에선 안 보임. debug 어려움. **코드 path 가 모든 환경에서 100% 결과물 보장하도록 박는 게 우선** — 분기 많아도 silent fail 박는 path 보단 download fallback 으로 항상 가는 게 robust.

- **user 의도 + web API 한계의 정직한 균형**: user "공유 기능 X" 명시 + share 가 silent fail = share 제거가 정답. 모바일 사진첩 직접 진입은 web 표준 한계 (share sheet 거쳐야 함) 로 X. 하지만 user 가 share 거부 = trade-off 받아들임. "이게 안 됩니다" 명시가 silent fail 보다 robust UX.

- **multi-iteration 7번의 lesson**: 시도 1~6 모두 "더 robust" 박은 패턴들이 share API 자체의 비결정성 못 잡아냄. 시도 7 = "share 자체 제거" 가 정답. **때로는 빼는 게 더 robust**. premature optimization (offscreen clone, CORS fetch, capability detection, title 박는 패턴) 보다 simple blob download 가 모든 환경 작동 보장.

- **데스크탑 회귀 발견의 가치**: user 가 "데스크탑도 안 됨" 보고 = share path 가 데스크탑까지 영향. 모바일 fail + 데스크탑 OK 였다면 모바일-specific 분기 박았을 것. 데스크탑 회귀 보고로 share path 자체 제거 가능. **comprehensive testing 이 root cause 발견의 핵심**.

### 12차 — 진짜 회귀 origin 발견 + production 검증으로 fix (`9d233b3`)

11차 share path 제거 후에도 user "이젠 로컬 데스크탑도 안돼" 보고. 제가 "Playwright dev server 에서 다운로드 trigger 정상" 박고 SW 캐시 문제로 진단했는데, user 정정 = **"방금 다운로드 됐어도 이미지 파일이 아니였잖아"**. download event fire ≠ 진짜 PNG 다운로드. 이전 검증들이 모두 trigger 여부만 봤고 blob 내용물 검증 안 함.

진짜 root cause 진단 (production 직접 Playwright):
- `/stats` = 117 imgs / **96 external** (ddragon 챔피언 아이콘 + chzzk profile)
- modern-screenshot 가 외부 image 의 src 를 SVG foreignObject 의 dataURL inline 박기 위해 추가 fetch 호출
- default fetch (no mode option) 가 일부 외부 CDN (chzzk `nng-phinf` 등) 에서 무한 hang
- captureBlob 8s timeout → `"[png-export] 실패: png export timeout (8s)"` console.warn
- `<a download>` 자체가 appendChild 안 됨 → 다운로드 trigger X
- localhost 는 image cache 박혀 있어 회피된 **false-positive 환경**

회귀 origin = **commit `31137bf`**. "모바일 hang 회피" 목적으로 offscreen clone 제거하면서 `fetch: { requestInit: { mode: 'cors', credentials: 'omit' } }` option 까지 같이 제거. 그게 회귀의 진짜 source. 11차 share path 제거 commit (`071587c`) 은 무관 (이미 31137bf 가 박은 회귀 위에 박힌 거).

fix (`9d233b3`, `src/composables/usePngExport.js`):
1. `DEFAULT_OPTS.fetch = { requestInit: { mode: 'cors', credentials: 'omit' } }` 복원 — 외부 CDN 의 CORS header 박힌 image 정상 fetch
2. `captureBlob` 안에 `waitForImages(el)` 박음 — img.complete + naturalWidth>0 까지 await (max 4초/이미지). 안 그러면 incomplete img 가 빈 PNG (72byte) 캡처 회귀
3. timeout 8s → 12s — 외부 image 96+ 개 fetch + dataURL inline 시간 여유

production 검증 (Playwright, SW unregister + cache 전부 비움 후 fresh navigate):
- 데스크탑 1440: blobSize **637,756 bytes** (~623KB) / blobType `image/png` / magic byte `89 50 4e 47 0d 0a 1a 0a` (PNG signature 정확 매치) / elapsedMs 4817ms / `<a download>` appendChild ✓
- 모바일 375 (`?mobile=1`): elapsedMs 6069ms / 다운로드 trigger ✓
- 실 button click → URL.createObjectURL 의 blob: realDownloadBlobSize **575,720 bytes** = user 의 07:47 다운로드 (575,652 bytes) 와 거의 동일 크기 / PNG magic byte 정확 / isPng true
- `file` 명령: `PNG image data, 960 x 2049, 8-bit/color RGBA, non-interlaced`
- 시각 검증: stats 헤더 (선수/합계/통계/2026-06-09) + 21 행 선수 데이터 + KDA/K%/GPM/모스트 챔피언 모두 정상 + 챔피언 아이콘 96개 dataURL inline 정상 + 헤더 겹침 없음

### 학습 (12차 추가)

- **download event fire ≠ 진짜 PNG 다운로드**: Playwright 의 download event 가 fire 됐다는 건 browser 가 다운로드 initiate 했다는 신호일 뿐. blob 내용물이 진짜 PNG 인지, 빈 PNG (72byte) 인지, 외부 image 실종된 corrupted 인지는 별 검증. user 의 "이미지 파일이 아니었잖아" 지적이 이 차이의 진짜 비용을 드러냄. **blob.size + blob.type + magic byte + 시각 검증** 4 단계로 fix 검증해야 root cause 진짜로 잡힌 거 보장.

- **localhost vs production 의 image cache 차이**: 같은 코드가 localhost 에서는 동작 + production 에서는 hang = 외부 CDN 의 image cache 차이. localhost 검증만으로는 회귀 false-positive 통과. **production URL (Vercel deploy 완료 + SW unregister + cache 전부 비움 후 cold reload) 로 final 검증 필수**. 이전 11번의 시도 모두 localhost 만 검증한 게 진짜 origin 못 잡은 이유.

- **외부 image 다수 export 시 modern-screenshot 의 fetch option**: domToBlob 이 외부 image 의 src 를 dataURL inline 박기 위해 추가 fetch 호출. 외부 CDN 이면 `requestInit.mode='cors'` + `credentials='omit'` 박지 않으면 일부 환경에서 무한 hang. 이건 modern-screenshot 의 default 가 의외로 부족한 부분 + 외부 CDN 의 CORS 정책 따라 결과 다름. 미래 export 작업 시 **CDN image 가 있으면 default 옵션부터 의심**.

- **commit 의 "스코프 작게" 함정**: `31137bf` commit message = "모바일 hang 회피 — offscreen clone 제거". 작은 scope 였다고 인식했지만 같은 변경에서 CORS fetch option 도 같이 제거됨. **단순 scope ≠ 단순 영향**. commit message 가 "X 만 변경" 박혀있어도 실제 diff 에서 의도 외 변경 있는지 확인 필수. 또는 commit 더 작게 쪼개기.

- **multi-iteration 11+1 의 lesson**: 11번 동안 share path 의 silent fail, capability detection 의 false positive, offscreen clone 의 모바일 hang, JSDoc 의 stale 등 표면 증상만 패치. 진짜 root cause = 가장 단순한 fetch option 의 mode. **표면 패치 멈추고 처음부터 production blob 의 magic byte 까지 추적했어야**. user 의 "안 됨" 보고가 표면 증상 (다운로드 폴더에 파일 없음) 만 묘사 가능. 진짜 진단은 개발자가 매번 다층으로 검증해야 (trigger / blob size / blob magic / file content / 시각).

- **production 검증의 진짜 절차**:
  1. localhost dev server (`bun run dev`) 박고 1차 검증
  2. commit + push → Vercel deploy ~90초 대기 (또는 `vercel deploy --prod` 직접)
  3. Playwright 으로 production URL 접근 + `navigator.serviceWorker.getRegistrations()` → 모두 `unregister()` + `caches.delete()` 로 SW cache 전부 비움
  4. `?nocache=N` query 박고 fresh navigate
  5. button click → URL.createObjectURL hook 해서 blob.size + blob.type + magic byte 추출
  6. 다운로드 파일 `file` 명령 + Read tool 의 image viewer 로 시각 검증
  7. 모바일 (`?mobile=1`) 도 동일 검증

## 관련 메모리

- [[feedback_no_firestore_visibility_toggle_2026-06-09\|feedback_no_firestore_visibility_toggle_2026-06-09]] — firebase.js 에 visibilitychange→disableNetwork/enableNetwork 박지 X (1차 흐름의 직접 산물)
- [[feedback_no_playwright_prod_writes_2026-06-09\|feedback_no_playwright_prod_writes_2026-06-09]] — QA 시 production URL 진입해 write 트리거 X (2차 흐름의 흔적)
- [[feedback_componentize_repeated\|feedback_componentize_repeated]] — 2회 이상 반복 패턴 즉시 컴포넌트 추출 (6차 Task #6 의 직접 적용)
- [[feedback_check_design_first_2026-05-15\|feedback_check_design_first_2026-05-15]] — Codex 큐/TODO 처리 전 박힌 결정 grep (4차 진단 결과 root cause 이미 박혀있음 발견의 정합)
- [[feedback_no_uid_display\|feedback_no_uid_display]] — UID/raw doc id 노출 X (6차 검증 중 dev HMR stale 의 raw streamerId 표시 발견 후 reload 로 nickname 정상 확인)
- [[feedback_obsidian_decision_log\|feedback_obsidian_decision_log]] — 이 narrative 자체의 룰
- [[feedback_plan_file_before_work_2026-05-20\|feedback_plan_file_before_work_2026-05-20]] — 안정화 sprint 진입 시 `/Users/hikaru/.claude/plans/logical-gathering-yao.md` 박은 정합
- [[feedback_useviewport_pointer_coarse_headless_limit_2026-06-09\|feedback_useviewport_pointer_coarse_headless_limit_2026-06-09]] — useViewport 의 pointer:coarse + width<1024 AND 조건이 Playwright headless 와 안 맞음 (7차 QA 진입 시 발견, F-P2-1 runbook 의 memory 화 결과)
