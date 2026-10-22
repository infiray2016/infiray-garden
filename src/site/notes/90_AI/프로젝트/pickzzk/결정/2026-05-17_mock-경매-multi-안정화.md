---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/결정/2026-05-17_mock-경매-multi-안정화.md","permalink":"/프로젝트/pickzzk/결정/2026-05-17_mock-경매-multi-안정화/","tags":["프로젝트/pickzzk","결정","모의경매","multi"],"dg-note-properties":{"tags":["프로젝트/pickzzk","결정","모의경매","multi"],"project":"pickzzk","date":"2026-05-17","status":"박힘","commits":["d817458","961fa86","d556a62","2eab5bb","6078ed8","7637f81","ed0e981","e515aa0",9318470,"3dabf61","8d2b6e2","85e432d","e0e2e66"],"related-memory":["feedback_auction_unsold_relist_2026-05-11","feedback_auction_multi_free_bid_2026-05-12","feedback_auction_multi_team_budgets_2026-05-12","feedback_check_design_first_2026-05-15","feedback_obsidian_decision_log"]}}
---


# 모의 경매 (멀티 모드) 회귀 안정화 + UX 정합 (13 결정 묶음)

> 2026-05-17 세션. 회귀 검증 자동화 (mock-test-runner subagent) 로 멀티 모드에서 발견된 13 개 commit 묶음. 박힌 결정만 모음 — 각 결정마다 "어떤 옵션이 있었는지 / 무엇을 선택했는지 / 왜 그렇게 결정했는지 / 어떤 조건에서 다시 뒤집을 수 있는지" 5섹션 형식. 상세 commit narrative 는 git log 참조.

## 배경

지난 R20 C14 (모의 경매 멀티 모드) 박힌 이후 첫 본격 회귀 라운드. 자동화 매트릭스 + 직접 2-tab 브라우저 UI 흐름 (호스트 + 게스트 동시 참가 시나리오) 으로 검증한 결과, **이미 박혀 있던 자낳대 정합 spec 과 실제 멀티 모드 코드 동작이 안 맞는 부분이 다수 발견됨**. 추가로 지난 1주일 동안 누적된 UI noise (불필요한 라벨, 중복 입력 영역, viewport 별 가로 잘림) 도 함께 정리.

세션 마지막에 사용자 verbatim: "옵시디언도 작업 내용에 따라 폴더 내용에 맞춰서 작성해주길 기대했었는데" — 다음부터 큰 commit 묶음은 push 직전에 결정 파일로 박는 룰 추가 ([[90_AI/claude/개념/메모리\|memory]] `feedback_obsidian_decision_log`).

## 결정 13건

### 1. 팀장 가격만큼 budget 자동 차감 로직 (subtract-point-value) 엔진 차원에서 제거

**옵션:**
- A) 셋업 화면 (SetupCardRule) 의 default 값만 `'none'` 으로 바꿈 — 이미 저장된 셋업은 그대로
- B) 엔진 자체 (`auctionEngine.buildTeams` 안 `subtractCaptainCost`) 를 무조건 false 로 박아서 셋업 값 무관하게 차감 안 함
- C) 자낳대 시즌1 spec 유지 — 팀장 가격만큼 자동 차감 (marketValue 만큼)

**선택: B**

**근거:** A 만으로는 이미 fixture / RTDB 에 박혀있는 `budgetPolicy='subtract-point-value'` 가 그대로 적용 → 경매 시작 화면에서 "팀별 1000pt" 라고 안내해놓고 실제 시작하면 팀장 가격 100pt 차감되어 900pt 가 됨. 사용자 직관에 안 맞음. 자낳대 시즌2 의 자유 budget 패턴 정합 ([[90_AI/claude/개념/메모리\|memory]] `feedback_auction_multi_team_budgets_2026-05-12`).

**철회 조건:** "팀장이 비싸면 그 팀은 그만큼 budget 이 줄어야 자연스럽다" 는 spec 이 user 한테서 다시 나오면 — 현재로선 단순함 (0pt auto-pick) 이 우선.

---

### 2. 입찰 자격이 1팀만 남으면 경매 안 돌리고 0pt 즉시 배정 (멀티 엔진에도 박음)

**옵션:**
- A) 현재처럼 일단 경매 시작 → 입찰 안 들어오면 deadline 만료 후 finalize → 유찰되면 재경매 → 그것도 안 되면 random 배정
- B) 매물 등장 시점에 "그 매물 입찰 가능한 팀이 1팀뿐" 이면 경매 발동 없이 0pt 로 즉시 배정 (예: 마지막 한 자리만 남은 포지션)
- C) 한 번에 여러 매물이 자동 배정 조건일 때 — 한 매물씩이 아니라 batch 로 한꺼번에 처리

**선택: B + C**

**근거:** 박힌 결정 ([[90_AI/claude/개념/메모리\|memory]] `feedback_auction_unsold_relist_2026-05-11`) — "입찰 자격이 1팀만 있으면 포인트 안 받고 자동 배정". 솔로 엔진 (`useMockAuctionState`) 에는 이미 박혀 있었는데 멀티 엔진에는 빠져 있었음. `advanceLot` 안에 `pickNextLotCandidate(cur, historyExtraIdxs, consumedRelistIds)` 헬퍼 + while loop 으로 main 큐와 유찰 (relist) 큐 양쪽 처리. 자동 배정 여러 건을 single RTDB `update()` batch 로 묶어서 atomic.

**철회 조건:** 자낳대 spec 자체가 바뀌면 (예: "모든 매물 무조건 경매 통과해야 한다" 같은 룰). 현재로선 가능성 X.

---

### 3. 유찰 후 재경매 횟수 2회 → 1회로 축소

**옵션:**
- A) 자낳대 spec 정합 2회 (메인 라운드 + 재경매 1차 + 재경매 2차 → 그래도 안 팔리면 random 배정)
- B) 1회로 축소 (메인 라운드 + 재경매 1차 → 그래도 안 팔리면 random 배정)

**선택: B**

**근거:** user 보고 verbatim 2026-05-17: "유찰 매물 재경매시 카운트 2번함... 10초 끝나고 또 10초 연속으로". 자낳대 spec 정합 (2회) → user 정정 (1회). UX 가 spec 보다 우선. 코드 변경 최소 (`MAX_RELIST` 상수 1 로 변경 + 테스트 기대값 정정).

**철회 조건:** 자낳대 정합이 다시 사용자 핵심 가치로 박히면 (현재로선 UX 우선 > spec 정합).

---

### 4. 멀티 모드에서 "이 팀은 AI 인지 다른 시청자인지" 판정 기준 변경 — 팀 claim 기반

**옵션:**
- A) 기존 로직 그대로 — `coachId !== myCoachId` 면 AI (솔로 모드 패턴)
- B) 멀티 모드에서는 `roomTeamClaims[idx]` (그 팀 자리를 claim 한 시청자 ID) 가 없으면 AI

**선택: B (멀티 모드에서는 팀 idx 받아 claim 정보 검사)**

**근거:** 멀티 모드는 팀 claim 기반 (`myTeamIdx`, `roomTeamClaims`) — coachId 와 무관. 기존 로직 그대로 두니까 모든 팀이 AI 로 표시되는 회귀 발생 (user 보고: "다른 플레이어가 안 보이고 AI 로 표시됨"). `MockAuctionTeamCards` 에 `isAi(t, idx)` + `otherNickname(idx)` 새 prop 추가해서 ME / 다른 시청자 닉네임 / AI 3-state tag 표시.

**철회 조건:** 없음.

---

### 5. 진행 중인 방에 늦게 들어온 게스트는 대기방에서 멈춤 (lobby gate)

**옵션:**
- A) 기존처럼 방 상태가 '진행 중' 이면 즉시 sim 화면 자동 진입
- B) `myTeamIdx >= 0` 조건 (내가 팀 자리를 claim 했는지) 추가 → 자리 없는 관전자는 대기방에 머무름

**선택: B**

**근거:** 기존 동작에서 늦게 들어온 게스트가 대기방 거치지 못하고 sim 으로 자동 진입 → 내 팀 idx 가 -1 로 고정 → "입찰 가능 YOU" 표시는 뜨는데 모든 입찰 버튼이 disabled 됨. 빈 AI 자리 claim 할 기회도 없음. 관전자는 대기방에 남아서 빈 자리 claim 후 자동 진입하거나, "관전으로 입장" 버튼으로 명시적으로 들어옴.

**철회 조건:** 없음.

---

### 6. 진행 중인 방 대기방 UI 에 'LIVE · 진행 중' 태그 + '관전으로 입장' 버튼

**옵션:**
- A) 대기방 UI 그대로 — 'LIVE' 표시 없음
- B) 방 상태가 '진행 중' 일 때 'LIVE · 진행 중' 태그 + late-hint 박스 + '관전으로 입장' 버튼 노출

**선택: B**

**근거:** 늦게 들어온 게스트가 방이 LIVE 인지 모르고 대기방에 머물기만 함. 명시적으로 "이 방은 이미 시작됨" 안내 + 관전 전용 입장 옵션 (`emit('start', { myTeamIdx: -1 })`). 진입 후에는 `MockAuctionHouse` 의 `myEligible` prop 이 false 로 평가되어 입찰 버튼이 자동 disabled.

**철회 조건:** 없음.

---

### 7. 나가기 버튼 누르면 다른 참가자에게 즉시 표시 (leave 명시 브로드캐스트)

**옵션:**
- A) 기존처럼 `unsubscribeRoom()` 만 호출 — RTDB 의 `onDisconnect` 가 WebSocket 끊김에만 발동 (heartbeat 12초 뒤에야 alive=false 로 바뀜)
- B) `unsubscribeRoom()` 안에서 RTDB 에 `{disconnected: true}` 명시적으로 박은 뒤 unsubscribe

**선택: B**

**근거:** user 보고 "다른 플레이어 나가도 모름" — `onDisconnect` 는 WebSocket 이 실제로 끊겨야 발동하는데, 나가기 버튼은 WebSocket 을 안 끊고 sub 만 해제. 다른 client 는 heartbeat 12초 timeout 까지 alive 로 판정 유지. 명시적으로 `disconnected: true` 박으면 다른 client 가 즉시 인지. 드래프트 (snake) / 경매 (auction) 양쪽 동일 패턴 적용.

**철회 조건:** 없음.

---

### 8. 모의 화면 진입 시 stale 방 객체 정리 (auction singleton 도)

**옵션:**
- A) 드래프트만 화면 진입 시 unsubscribe (기존) — 경매는 빠져 있었음
- B) 경매도 동일하게 화면 진입 시 (`onMounted`) unsubscribe 처리

**선택: B**

**근거:** 경매 보드 → 진입 화면으로 빠져나갈 때 (`onExit`), singleton 으로 박혀 있던 stale roomCode 와 sub 가 그대로 남음 → 다음 join 시 conflict. 드래프트와 동일 패턴 적용. `MockAuctionBoard.onExit` 는 phase=entry 만 set 하고 singleton 정리 X → 진입 화면 `onMounted` 가 책임.

**철회 조건:** 없음.

---

### 9. "0pt 가격" / "막판!" / "예산 부족" / "CRITICAL" 라벨 모두 제거

**옵션:**
- A) 라벨 유지 (사용자에게 시각적 정보 제공 목적)
- B) 모두 제거 (UI 정리)
- C) 일부만 제거

**선택: B**

**근거:** user verbatim 2026-05-17: "예산 부족 라벨 제거해줘, 막판 뱃지도 제거해줘", "팀장 or 감독 0 포인트 부분 안 보이게 해줘". 모두 hard-coded threshold (남은 시간 < 5초 / 평균 잔여 300pt 미만 / 잔여 200pt 미만 / 가격 0pt) — 박힌 사용자 룰과 무관. anti-snipe (마감 직전 입찰 시 자동 연장) 기능 자체는 그대로 유지.

**철회 조건:** 사용자가 UI hint 가 필요하다고 다시 박으면.

---

### 10. 대기방의 중복 닉네임 입력란 제거

**옵션:**
- A) 대기방에 닉네임 입력란 유지 (수정 가능)
- B) 제거 — 첫 화면에서 박은 값 그대로, 멤버 리스트의 ME 태그로 본인 확인

**선택: B**

**근거:** user verbatim 2026-05-17: "만들기 전에도 닉네임 필수로 입력하는데 대기방에도 수정 가능한 입력창이라 입력을 두번 하는 거 같아 불필요해 보임". 드래프트 대기방 (`MockLobby`) + 경매 대기방 (`MockAuctionLobby`) 양쪽에서 닉네임 입력 블록 제거.

**철회 조건:** 닉네임 자주 바꾸는 use-case 가 발견되면 다시 박음 (현재로선 첫 화면에서 박는 걸로 충분).

---

### 11. 경매 대기방 팀 카드 — 4-col 진입 breakpoint 700px → 980px

**옵션:**
- A) 700px 부터 4-col 유지 (드래프트 대기방과 동일)
- B) 980px 부터 4-col (경매만 — 시작 PT 입력 행 때문에 카드 폭이 더 필요)

**선택: B**

**근거:** 경매 대기방의 팀 카드는 시작 PT 입력 (~84px) 행 때문에 min-content 폭이 약 190px. 768px 화면을 4 컬럼으로 나누면 192px 이하라 가로 잘림. 드래프트 대기방은 시작 PT 입력 행 자체가 없으므로 700px 유지 가능. 경매만 breakpoint 를 980px 으로 올림.

**철회 조건:** 시작 PT 입력 자체가 제거되거나 (경매 룰 변경), 경매 카드 layout 이 바뀌면.

---

### 12. 모바일 (375px) 팀 카드 가로 잘림 — `min-width: 0` 표준 fix

**옵션:**
- A) grid item 의 default `min-width: auto` 그대로 유지
- B) `.team-card { min-width: 0 }` + `.budget__input { min-width: 0; width: 100% }` 박음

**선택: B**

**근거:** CSS grid 의 default `min-width: auto` 는 자식의 min-content (budget input 84px + padding) 을 보장하려고 1fr 컬럼 약속을 깸. 375px viewport 에서 2 컬럼 × 209px = 438px > 360px → 아리사 팀 카드가 화면 밖으로 잘림. 표준 grid/flex item gotcha. `min-width: 0` + 자식 `width: 100%` 로 해결.

**철회 조건:** 없음 (표준 CSS 패턴).

---

### 13. 백그라운드 탭에서 경매 진행이 멈춰 보이는 문제 — deadline 정확 setTimeout 3중 backup

**옵션:**
- A) 기존처럼 `setInterval(300ms)` 만 유지 — Chromium 이 background tab 의 timer 를 5분 idle 후 1/분 으로 throttle 하므로 사실상 멈춤
- B) `setInterval` 유지 + deadline 시각에 정확한 `setTimeout(remain+100ms)` 별도 trigger + 탭 visible 복귀 시 즉시 finalize 시도 (3중 backup)
- C) Web Worker 로 timer 옮김 (worker 안은 throttle 안 됨)
- D) Audio API 트릭 (background throttling 우회 — 무음 오디오 재생 등)

**선택: B (3중 backup)**

**근거:** Chromium 의 background tab timer throttling (5분 idle 후 1/분) 이 user 보고된 진행 지연의 근본 원인. C/D 는 over-engineering. B 는 multi-room.md 의 trap pattern 정합 — RTDB 의 `currentLot.deadline` 단일 소스를 유지하면서, deadline 시각에 정확한 setTimeout (short timeout 은 throttle 영향 덜 받음) + visible 복귀 시 catch-up. 별도 박혀 있는 `_resyncSeq` 가드와 함께 race-safe.

**철회 조건:** Chromium 정책 변경 (background timer throttle 안 함) 또는 server-side processing (Firebase Cloud Functions) 도입 시 client timer 자체를 단순화 가능.

---

## 후속 작업

- 멀티 방 trap pattern 가이드 (`multi-room.md`) 에 background tab throttling 케이스를 6번째 trap 으로 추가
- `feedback_obsidian_decision_log` 룰 따라 큰 commit 묶음 push 직전에 결정 파일 작성 ([[90_AI/claude/개념/메모리\|memory]] 박힘)
- 회귀 검증 자동화 (mock-test-runner subagent) 에 background throttling 시나리오 추가 — Playwright 는 실 throttling 을 재현 못 하므로 manual fallback 절차도 같이 박을 것

## 검증

- 자동 회귀 매트릭스 A/B/C/D + 직접 브라우저 (375 / 768 / 1440 viewport) 전부 PASS
- `bun run verify` (lint + regression 375 + build) 통과
- 두 탭 동기 + 완료 dialog 가 양쪽 탭에 동시 노출 확인
- console error 0
