---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/결정/2026-06-08_운영-staff-polish-lv-필드-codex-정리.md","permalink":"/프로젝트/pickzzk/결정/2026-06-08_운영-staff-polish-lv-필드-codex-정리/","dg-note-properties":{}}
---

# 2026-06-08 운영 입력 흐름 정리 — staff-polish + Lv 필드 + Codex 큐 정리

## 무엇

운영자가 게임 결과를 admin 으로 입력하는 흐름에서 누락/잡음 3건을 정리하고, Codex 리뷰어가 남긴 P2 큐 3건을 처리 또는 defer 결정으로 닫음. 자낳대식 감독+코치 한쌍(staff-unit) 의 viewer 표시 잡음 두 곳을 같은 컴포넌트로 통일. 게임 업로드 폼에 Lv (챔피언 레벨) 입력 셀이 누락돼 있던 것 보강 (지표 화면 Lv 컬럼 데이터 source). wwme 6/6 + 6/7 내전 12 게임 firestore seed.

## 왜

`/scrim/day` 경기 상세 화면을 12 컬럼 풀 메트릭으로 확장하면서 ([[90_AI/프로젝트/pickzzk/결정/2026-06-08_시청자-경기-보기-완성-stats-detail-캘린더\|2026-06-08_시청자-경기-보기-완성-stats-detail-캘린더]] 참조) viewer 가 표시할 데이터가 어디서 오는지 다시 본 결과:

1. **Lv 컬럼 = 입력 누락** — wwme seed 게임은 Lv 박혀 표시되는데, 수기 입력 게임은 폼에 Lv 셀 자체가 없어서 "—" 만 표시. 사용자 보고 "게임 업로드 레벨 입력 누락"
2. **staff-unit 의 viewer 표시 잡음** — 자낳대 감독+코치 한쌍 룰 도입 후 (2026-06-06) MockAuctionLobby / MockAuctionResult 의 director 아바타가 작은 단일 원 + assistant 옆에 별도 표시 — 한쌍 의도와 시각이 안 맞음. 또 셋업 카드의 `staffUnits[].displayName` 이 "감독 ? · 코치 ?" 처럼 fallback 박혀있는 옛 doc 데이터 (`SetupCardRoster.nameFor` 의 root cause = coachPool 만 lookup → 스트리머 마스터 fallback 누락)
3. **Codex 큐 3건이 큰 review 의 일부로 남아 있음** — `#6 P2 DraftPick 스크림 지표 이식` 큰 review 가 부분 처리 후 "남은" 4 항목으로 남았는데, 그 중 (경기 상세 10인 테이블) 은 12 컬럼 확장 commit 으로 이미 처리됨. (avgGdpm) (팀 퍼블률) 두 항목은 데이터량/의미 약함 → defer 결정 필요

추가로 wwme.kr 의 6/6 6/7 자낳대 시즌2 12 게임 실 데이터를 운영 firestore 에 seed → 시청자가 실데이터로 화면 검증 가능.

## 어떻게

### 1차 — 게임 입력 Lv 필드 추가 (1ef2080)

`GameInputLineup.vue` 의 head-row 에 `<span>Lv</span>` + row 에 `<input type="number" min=1 max=18 placeholder=18>` 셀. grid-template-columns 6 col (56px name champ 100 70 70) → 7 col (56px name champ **44px** 100 70 70). `useGameInputState.emptyLineup` 의 row default 에 `lv: null` 추가. `normalizeGameData.buildPlayer` 가 `row.lv` 박음 (server consensus + 지표 매칭).

→ 신규 game doc 의 lineup/players 에 lv 박혀 LineupCardReadonly 12 컬럼 표가 "—" 가 아닌 실 값 표시. wwme seed schema 와 동일.

### 2차 — StaffAvatarPair 공통 컴포넌트 추출 (dd019dd)

`src/components/shared/StaffAvatarPair.vue` 신규. props = `directorName / directorSrc / assistantName / assistantSrc / size`. layout = director (size px) + assistant (size × 0.6, overlap 우하단 size × -0.15px). MockAuctionLobby (size=40) + MockAuctionResult (size=40) 둘 다 사용. R6 (반복 패턴 컴포넌트화) 정합. MockAuctionLobby 의 `slot__staff` line 분리.

### 3차 — staff-unit displayName "감독 ? · 코치 ?" stale 데이터 fix (8664f0f)

root cause = `SetupCardRoster.buildStaffSavePayload.nameFor` 가 coachPool 만 lookup → 매칭 안되면 "?" fallback. fix = coachPool 1차 + `props.streamers` (스트리머 마스터) 2차 fallback. 기존 stale data 는 `scripts/fix-staff-unit-display-names.mjs` 일회용 admin script 로 일괄 정정. MockAuctionLobby 의 staff-unit derived name (displayName 무시) 도 같이 정리.

### 4차 — wwme 12 게임 firestore seed

`scripts/wwme-inhouse-raw.json` (gitignore) = wwme.kr/lol/scrim-records 6/6 6/7 12 게임 풀 메트릭 raw 데이터 (lv/kpPct/dpm/goldSharePct 등). `scripts/seed-wwme-inhouse.mjs` (gitignore) 가 `normalizeGameData` 거쳐 firestore 박음. POSITION_MAP 에 Bot/ADC 둘 다 'Bot' 정규화. seed 후 `/scrim/day/2026-06-06` + `/scrim/day/2026-06-07` 진입 → 12 경기 카드 정상 표시.

수기 입력 6/8 doc (`IfFE5pOkdBofHjnWdMKn`) 의 isTest=true 박혀있어 production viewer 가 안 보이던 것 → `scripts/move-day-doc.mjs` 로 7일 로 이동 + isTest=false update.

### 5차 — 포지션 표준 ADC → Bot 통일 (a12749e)

자세한 변경은 [[90_AI/프로젝트/pickzzk/결정/2026-06-08_시청자-경기-보기-완성-stats-detail-캘린더\|2026-06-08_시청자-경기-보기-완성-stats-detail-캘린더]] 4차 섹션. 운영 입력 측면 영향 = `GameInputLineup` 의 포지션 옵션 / `normalizeGameData` 의 role 매핑 / `usePlayerRatings` / Stats 의 ROLES key. legacy doc (ADC 박힌 game) 회귀 0 — alias 패턴 유지.

### 6차 — Codex 큐 P2 3건 정리 (39a58de + 이번 push)

`.claude/CODEX_REVIEW.md` 의 P2 "남은" 4 항목 중:
- **avgGdpm** (Gold Diff Per Minute vs 상대 같은 포지션) — 매칭 알고리즘 필요 + 시즌 게임 ~13 개 = 통계 약함. user 결정 = 제거 (defer)
- **팀 퍼블률** (`teamFirstBloodRate` 라벨 충돌 회피 — 선수 상세 `퍼블 관여` 와 별도) — user 결정 = 제거 (defer)
- **경기 상세 10인 지표 테이블** — 이번 세션 `53e6f96 feat(scrim-game): LineupCardReadonly 12 col 풀 메트릭` 으로 처리됨 — CLAUDE.md R3 (처리한 항목은 삭제하고 완료 로그 누적 X) 정합 으로 자동 정리
- **매치업 셀 클릭 → 상세 drawer** — 유지 (큰 작업, 별도 spec 필요)

## 결과

5 commits push (1ef2080, dd019dd, 8664f0f, a12749e, 9769100). 신규 파일 3 (StaffAvatarPair / downloadDataUrl / useScrimMonth) + script 3 (wwme seed / move-day-doc / fix-staff-unit-display-names — 모두 gitignore). 검증:

- `bun run lint` 0 errors
- `bun run regression` 526 pass (Lv field 회귀 영향 0, normalizeGameData 단위 테스트 통과)
- Playwright MCP 직접 검증: 게임 업로드 폼 Lv 셀 input 동작 + MockAuctionLobby/Result StaffAvatarPair (director 40px + assistant 24px overlap) 일관 표시
- production firestore: wwme 12 게임 + 6/8→6/7 이동 + isTest:true→false 적용 완료
- Codex 큐: 4 항목 → 1 항목 (매치업 drawer 만 남음)

사용자 흐름 검증:
- admin GameInput → 새 게임 업로드 → Lv 셀 입력 → 저장 → `/scrim/day` 경기 상세 → 12 컬럼 표에 Lv 실 값 표시 (이전엔 "—")
- staff-unit 모드 모의 경매 → lobby 에서 director+assistant 한쌍 시각 (StaffAvatarPair) + 결과창에도 동일 → 한쌍 의도 시각화 일관
- wwme 6/6 6/7 운영일 진입 → 12 게임 카드 + 메트릭 표시 (시청자 실데이터 검증)

## 남은

- admin GameInput 의 streamerId 자동 매칭 (root cause solve) — 현재는 viewer 의 byName fallback 으로 표시만 보완. 다음 cycle 에서 입력 시 streamer 마스터 lookup → 자동 박는 흐름. byName fallback 의존 줄이기
- Codex 매치업 drawer 화면 (큰 작업, 별도 spec) — viewer detail 진입 후 다른 선수와 비교 가능한 화면
- DAY 편집 화면이 filterTestDocs 안 거치는 일관성 점검 — viewer 만 isTest 제외하는 게 의도였는지 다시 확인 deferred
- staff-unit 의 코치 단독 (assistant 없는) 케이스 시각 — 현재 StaffAvatarPair 가 assistantSrc 없을 때 어떻게 보이는지 audit 필요

## 관련 메모리

- [[feedback_componentize_repeated\|feedback_componentize_repeated]] — 2회 이상 반복 패턴은 즉시 컴포넌트 추출 (StaffAvatarPair)
- [[feedback_check_design_first_2026-05-15\|feedback_check_design_first_2026-05-15]] — Codex 큐 처리 전 박힌 결정/spec 우선 (P2 avgGdpm defer 정합)
- [[feedback_no_uid_display\|feedback_no_uid_display]] — admin SDK script 의 doc id 노출 안 함 (script 결과 보고만)
- [[feedback_obsidian_decision_log\|feedback_obsidian_decision_log]] — 이 narrative 자체의 룰
