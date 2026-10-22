---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/결정/2026-06-08_시청자-경기-보기-완성-stats-detail-캘린더.md","permalink":"/프로젝트/pickzzk/결정/2026-06-08_시청자-경기-보기-완성-stats-detail-캘린더/","dg-note-properties":{}}
---

# 2026-06-08 시청자 경기 보기 완성 — Stats 5탭 + 경기 상세 + 운영일 캘린더

## 무엇

시청자가 "이 시즌 게임들이 어디까지 진행됐고, 누가 잘했고, 한 경기 안에서 무슨 일이 있었는지" 한 묶음으로 볼 수 있게 세 화면을 한 번에 정리. 지표 화면(`/stats`)에 선수/챔프 breakdown 카드 신설 + TOP3 카드 깨짐 fix + 내보내기(PNG 저장) 헤더/Safari 호환 정리. 운영일 화면(`/scrim/day`) 위쪽에 wwme.kr 패턴의 월 캘린더 grid 추가 — 시즌 전체 경기 분포를 한눈에. 경기 상세 화면(`/scrim/games/:date/:gameId`) 의 라인업 표를 9 컬럼 → 12 컬럼 풀 메트릭 (Lv/KDA/KP%/DPM/DPG/DMG%/GPM/G%/골드/딜량) 으로 확장해서 지표 화면과 컬럼 일관성 확보.

## 왜

지표 화면, 운영일 화면, 경기 상세 화면이 따로 만들어진 직후라 시청자가 한 곳에서 다른 곳으로 자연스럽게 흐를 수 있는 연결고리가 약했다. 사용자가 "지표에서 1등 카드가 잘림" + "내보내기 그림파일이 아냐" + "8일 도장 자리 7일로 옮겨야" + "월 캘린더 wwme 처럼 만들어줘" + "경기 상세에 지표 풀 컬럼 다 넣어줘" 같은 보고를 같은 흐름으로 묶어서 던졌고, 진단해보니 모두 "시청자가 시즌 흐름을 보는 동선" 한 줄에 있었다.

추가로 발견된 문제:

1. **TOP3 카드 깨짐** — CSS grid item 의 `min-width: auto` (browser default) 가 content 폭을 강제해 카드가 컨테이너 밖으로 튀어나옴. 내보내기 PNG 에서 더 심함
2. **내보내기 dataURL 실패** — Safari/mobile 에서 `<a download href="data:...">` 큰 dataURL 을 파일로 인식 못함. "그림파일이 아냐" 보고의 원인
3. **streamerId 미박힘 doc** — 수기 입력 또는 OCR 자동 매칭 누락 시 lineup row 의 streamerId 가 빈 채로 저장됨 → byId Map lookup 실패 → 아바타 빈 원
4. **legacy mvp string "B:3"** — 옛 DraftPick 시절 mvp 가 `"A:2" / "B:3"` 같은 팀:인덱스 형식. 새 형식 `{byStats: {name, champion, ...}}` 와 mixed → mvp.byStats.name 자리에 "B:3" 박혀서 viewer 가 KDA 와 합쳐 "B:3 / /" 어색하게 표시

## 어떻게

### 1차 — Stats 5탭 재설계 (선수 / 포지션 / 팀 / H2H / 챔프)

기존 단일 테이블에서 5 탭 분리. 선수 탭 = 12 컬럼 heatmap (rank/Lv/KDA/KP%/DPM/DPG/DMG%/GPM/G%/firstBloodRate/championCount/games). 포지션 탭 = 5 섹션 (포지션별 TOP3 카드 + 종합 TOP3). 팀 탭 = 카드 (승률·KDA·게임수). H2H 탭 = 4×4 매트릭스. 챔프 탭 = 챔프 sorting + breakdown 카드 (펼침). breakdown 카드 = 선수 클릭 시 챔프별 KDA/지표 표시, 챔프 클릭 시 사용 선수별 승률 표시. 두 시각 분리 (선수=상세 지표, 챔프=프로필 사진 + 승패 색).

상세 commit chain: `51f69e3` → `d24f1d6` → `eb18b33` → `5d555bd` → `14e2260` → `f617bc7`. PLAN: `.claude/PLAN_2026-06-07_stats-redesign.md` + `.claude/mockups/stats-redesign-5tabs.html`.

### 2차 — TOP3 카드 overflow + 내보내기 헤더 + Safari Blob URL fix

TOP3 카드의 `.top3__card` 에 `min-width: 0 + overflow: hidden`. `.top3__nm` 에 nowrap + ellipsis + padding-right. `.top3__stats` grid 3 col + span ellipsis. 내보내기 시 화면 표시와 다른 layout (탭/기간/필터 안 보임) 으로 캡처하기 위해 DraftPick legacy 의 `data-export-mode="light"` 패턴 차용 — 캡처 직전 `captureRef.setAttribute('data-export-mode', 'light')` + filter (`data-export-exclude` / `head` / `last-agg` skip). exportMeta computed 가 (대회명 · 탭 · 기간 · 종류 · 날짜) 헤더 동적 박음. 사용자 요구로 "출처 pickzzk" 라인 제거.

내보내기 그림파일이 아니라는 사용자 보고 = Safari 의 `<a download>` 가 큰 dataURL 을 인식 못함. fix = `src/utils/downloadDataUrl.js` 신규 — `fetch(dataUrl).then(r => r.blob())` 로 Blob 변환 → `URL.createObjectURL(blob)` → a.click → setTimeout 으로 revokeObjectURL. modern-screenshot domToPng + `document.fonts.ready` await 후 호출.

### 3차 — 포지션 표준 ADC → Bot 통일 + 한글 라벨 fix

`src/games/lol/index.js` 의 `roles = ["Top", "Jng", "Mid", "Bot", "Sup"]` + `roleLabels` 한글 매핑. `POSITION_KO_LOOSE` 에 ADC → "원딜" alias 유지 (legacy doc 회귀 0). `normalizeGameData.js` 의 `ROLE_MAP_SHORT_TO_LONG` 에 `Bot → Bottom` + `ADC → Bottom` 양쪽 박음. `positionKey('bot' / 'adc')` 둘 다 `'bot'` 반환. commit `a12749e`.

### 4차 — 운영일 월 캘린더 grid (wwme.kr 패턴)

`useScrimMonth` composable 신규 = 6×7 = 42 cell 배열 (YYYY-MM-DD + day + isCurMonth + isToday + isFuture + count + kinds:Set). `ScrimDayMonthGrid` 컴포넌트 = 헤더 (← 2026년 6월 → / 오늘 버튼) + 일~토 row + cell (count chip + kind dot 3색 — 내전/스크림/본선). cell 클릭 → 부모가 `router.push({ name: 'day', params: { date } })`. 오늘 = orange ring, 현재 선택 = ink ring, 미래 = dim opacity 0.5 + disabled. mobile 축소 layout. `ScrimDayView` 상단에 박음. PLAN: `.claude/plans/eager-imagining-wreath.md`. commit `3795ddf`.

### 5차 — 경기 상세 12 컬럼 풀 메트릭 + byName fallback

기존 9 컬럼 (포지션/플레이어/KDA/골드/딜량/기본 4종) → 12 컬럼 (포지션/플레이어/Lv/KDA/KP%/DPM/DPG/DMG%/GPM/G%/골드/딜량). 수기 입력 game 은 lv/kpPct/dpm 등 raw 값 누락이라 viewer 표시 시점에 fallback 계산 — `kpPct = (kills+assists)/teamKills * 100`, `dpm = dmg/durationMin`, `dpg = dmg/gold`, `dmgSharePct = dmg/teamDmg * 100`, `goldSharePct = gold/teamGold * 100`, `gpm = gold/durationMin`. 즉 raw 값 없어도 derived 표시 100%.

streamerId 미박힘 doc 의 avatar 빈 원 fix = `ScrimGameView` 에서 `streamerByName` Map 계산 (nickname/chzzkName/displayName lowercase → streamer). `LineupCardReadonly` 가 `:streamer-by-id` + `:streamer-by-name` prop 둘 다 받음. row 매칭 = byId 우선 → playerName lowercase byName fallback. PlayerAvatar src = `streamer?.profileImageUrl || chzzkProfileImageUrl`. commits `26eedb1` → `53e6f96` → `c0cc980`.

### 6차 — MVP legacy "B:3" 패턴 처리

`MvpBadge.vue` 에 `LEGACY_TEAM_MVP_PATTERN = /^[AB]:\d+$/` 박음. mvp.byStats.name 이 그 패턴 매칭하면 invalid 처리해서 빈 표시. commit `f617bc7` 의 일부.

## 결과

11 commits push (45477f6 → c0cc980). 모두 origin/main 동기화. 검증:

- `bun run lint` 0 errors
- `bun run regression` 526 + 신규 4 (scrim-month-aggregate) = 530 pass
- `bun --bun run build` OK
- Playwright MCP 직접 검증: `/stats` 5 탭 전부 정상 mount + 1440/768/375 viewport + 내보내기 PNG 다운로드 동작 (Safari/mobile 호환 Blob URL) + 운영일 캘린더 cell 클릭 → 일자 list 이동 + 경기 상세 12 컬럼 표 + 아바타 100% 표시 (byName fallback 작동)
- console errors 0 / warnings 0
- 6/8 운영일 doc 1건 (mGbCusdbHKDOQrdWRJQn) admin SDK script 로 6/7 로 이동 + isTest=true → false update (script `scripts/move-day-doc.mjs`, gitignore 박음)

사용자 흐름 검증:
- `/stats` 진입 → 선수 탭 → 1등 카드 깨짐 없이 표시 → 챔프 탭 → 챔프 클릭 → 사용 선수 breakdown → 내보내기 PNG → Safari/iOS 에서도 다운로드 OK
- `/scrim/day` 진입 → 상단 월 캘린더 → 다른 일자 cell 클릭 → 그 일자 list → 경기 row 클릭 → 경기 상세 12 컬럼 표 + 아바타 표시 + MVP 카드 + 밴 리스트

## 남은

- 듀오시너지 화면 (선수 쌍 승률 + 라인 조합 필터) — 캘린더 끝난 후 별도 commit 약속, 미진행
- 경기 상세 cell hover popover preview (캘린더) — deferred
- 월 단위 export PNG (캘린더) — deferred
- 캘린더 ICS export — deferred
- 수기 입력 game 의 streamerId 자동 매칭 (admin GameInput 의 root cause solve) — 현재는 viewer 의 byName fallback 으로 표시만 보완. 다음 cycle 에서 입력 시 자동 박는 흐름 정리 필요
- DAY 편집 화면 vs 운영일 viewer 데이터 차이 — viewer 가 filterTestDocs 거치는데 day 편집은 안 거침. 일관성 점검 deferred

## 관련 메모리

- [[user_role_viewer\|user_role_viewer]] — 시청자가 시즌 흐름을 한눈에 보는 사용자 흐름 정합
- [[feedback_doc_plain_language_2026-05-18\|feedback_doc_plain_language_2026-05-18]] — 풀어쓰기 라벨 (DPM → "분당 딜")
- [[feedback_check_design_first_2026-05-15\|feedback_check_design_first_2026-05-15]] — Codex 큐 spec 우선 (12 컬럼 정합)
- [[feedback_obsidian_decision_log\|feedback_obsidian_decision_log]] — 이 narrative 자체의 룰
