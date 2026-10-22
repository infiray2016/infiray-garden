---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/결정/2026-05-18_R22-server-consensus-promotion.md","permalink":"/프로젝트/pickzzk/결정/2026-05-18_R22-server-consensus-promotion/","tags":["프로젝트/pickzzk","결정","scrim","server-consensus","R22"],"dg-note-properties":{"tags":["프로젝트/pickzzk","결정","scrim","server-consensus","R22"],"project":"pickzzk","date":"2026-05-18","status":"박힘 + production 검증 완료","commits":["06bf303","935f589","fd1f163",6227755,"6587bd3","14e2a3a","b31f7b7","f814f16","24d7080","0fdb15b"],"related-memory":["feedback_check_design_first_2026-05-15","feedback_obsidian_decision_log","feedback_doc_plain_language_2026-05-18"],"related-design":[".claude/CONSENSUS_AUTO_PROMOTION_DESIGN.md",".claude/R22_PLAN.md"]}}
---


# R22 — 시청자 제보 자동 승격을 server 로 옮긴 작업 (9 단계 묶음)

> 2026-05-15 ~ 2026-05-18 세션. 시청자가 제보한 게임 결과를 운영자가 일일이 처리하지 않아도 server 가 두 명 이상 합의 시 자동으로 정식 결과로 승격시키는 흐름을 박았다. legacy DraftPick 의 RTDB 기반 `tryConsensusPromotion` 로직을 Firestore batch atomic 으로 어댑트한 것이 핵심.

## 배경

활성 시즌 (호종컵 4기) 직전 마무리 작업. 이전까지의 흐름은 이랬다:

1. 시청자가 GameInput 화면에서 스크린샷 OCR + 매핑 + 교정 후 제보 버튼 누름
2. client 가 직접 Firestore 의 `submissions/{id}` 에 박음
3. 운영자가 admin 큐 페이지 (`/admin/submissions`) 를 **열어둔 상태에서만** client watcher 가 같은 운영일의 다른 제보들과 비교
4. 두 명 이상 일치하면 client 측 `setupAutoPromotion()` 이 `scrimDays/{date}/games/{id}` 로 승격

이 흐름의 결정적 문제: **운영자가 admin 큐 페이지를 닫아두면 자동 승격이 절대 안 돔.** 활성 시즌 중에는 운영자가 항상 켜둔다는 전제였지만, 운영자도 자고 다른 일도 함. 실제 호종컵 시즌 운영에서 "왜 결과가 안 올라오나" 가 반복되던 부분.

본 R22 는 그 의존성을 끊는 작업. 시청자 제보가 server (Vercel serverless 함수) 를 거치고, server 자체가 합의 판정 + 승격까지 동기로 처리한다. 운영자 페이지 mount 여부 무관.

> 사용자 verbatim (직전 세션, 2026-05-17): "이거 admin 큐 안 열어두면 자동 승격 안 되는 거 맞지? 시즌 들어가면 누구도 24시간 켜두진 못함."

## 결정 4건

### 1. 옵션 B (server-side Vercel 함수) 채택. 옵션 A (카피 완화) 거부

**옵션:**
- A) **client autoPromotion 유지 + 카피 완화** — admin 큐 페이지를 cron 처럼 백그라운드 탭으로 띄워두라고 운영자 가이드 박음. 코드 변경 최소.
- B) **server-side Vercel serverless 함수로 합의 판정 동기 처리** — 시청자가 `/api/scrim-submit` 로 보내고 server 가 admin SDK 로 submission 박은 직후 같은 운영일의 pending 들과 비교 → 두 명 이상 일치 시 즉시 `scrimDays/{date}/games/{id}` 로 승격 + 매칭 submissions delete. Firestore batch atomic.
- C) **Firestore Cloud Functions onCreate trigger** — server-side 인 점은 B 와 같지만 Firebase Functions 인프라 새로 도입. cold start + 빌드 파이프라인 + 비용 모델 별도.

**선택: B**

**근거:**
- A 는 운영자 행동 변화에 의존 — 안 켜두면 그대로 망. 활성 시즌에 운영자가 reliably 켜둘 보장 없음.
- C 는 인프라 새로 박는 비용이 큼. 픽지직은 이미 Vercel 에 serverless 함수가 박혀 있고 (`api/parse-scrim-screenshot.js` 등) 같은 인프라 안에서 끝낼 수 있음.
- B 는 기존 Vercel 인프라 위에서 한 endpoint 추가. legacy DraftPick 의 `tryConsensusPromotion` 로직이 이미 검증된 코드라 RTDB → Firestore 어댑트만 하면 됨.

**철회 조건:** 활성 시즌 운영해보고 server 함수 호출 latency 가 viewer 체감 불편할 정도면 (~1초 이상 일관되게) C 의 Firestore Functions onCreate trigger 로 이동 검토. 다만 그건 분 단위 latency 가 OK 라는 전제 — viewer 가 제보 후 즉시 확인 못 해도 무방하면.

---

### 2. submissions 컬렉션의 create 권한을 viewer → admin only 로 좁힘 + client 측 autoPromotion 코드 제거

**옵션:**
- A) client 가 여전히 직접 `addDoc(submissions)` 가능하게 두고, server endpoint 는 옵션으로만 — viewer 가 둘 중 아무 거나 써도 됨.
- B) **rules 의 create 권한을 admin only 로 좁힘. viewer 는 server endpoint 만.** server 가 admin SDK 로 rules 우회. client 측 `setupAutoPromotion()` 코드도 같이 삭제 (server 가 담당하니까).
- C) viewer 는 직접 못 박고 server endpoint 만 쓰는 건 동일하지만, client 측 watcher 코드는 그대로 둠 (admin 큐 새로고침용).

**선택: B**

**근거:**
- A 는 두 경로가 공존하면 어느 쪽으로 박혀도 합의 판정 일관성 보장이 어려움. 자가 합의 방지 (같은 clientId 가 두 번 박아도 한 표) 룰이 server 안에 박혀 있는데 client 직접 박기는 그 룰을 우회.
- C 는 "server 가 처리한다" 와 "client 도 처리한다" 두 가지 진실 source 가 공존 — admin 큐 페이지에서 같은 promote 가 두 번 시도되면 batch 충돌. 디버깅 어려움.
- B 는 한 흐름으로 통합. admin 큐 페이지는 read-only (수동 승인/반려만) 로 단순화. server 가 single source of truth.

**철회 조건:** server endpoint 가 어떤 장애 상황 (Vercel 함수 outage 등) 으로 안 돌 때 viewer 가 제보를 못 박게 되는 게 운영상 큰 문제로 드러나면, fallback 으로 admin 모드 fetch 끊긴 경우 client direct write 허용 검토. 다만 그러려면 자가 합의 방지 로직을 client/server 양쪽에 박아야 해서 비용 큼.

---

### 3. C8 회귀 가드 범위 — pure function (gamesMatch) 만. 자가 합의 방지는 integration 영역으로 미룸

**옵션:**
- A) **`gamesMatch` 만 단위 테스트로 회귀 가드** — 두 game 객체가 "같은 경기인지" 판정하는 pure function. tolerance 룰 (KDA ±1, gold ±2%, totalDamage ±3%, duration ±3s, 10 슬롯 정확 일치) 을 case 단위로 가둠.
- B) **`gamesMatch` + `tryConsensusPromotion` 둘 다 단위 회귀** — Firestore admin SDK 를 mock 해서 자가 합의 방지 (같은 clientId 두 번 박아도 한 표 / 익명 총합 한 표) 룰도 가둠. Firestore mock infra 새로 도입.
- C) **`gamesMatch` 단위 + `tryConsensusPromotion` integration** — server endpoint 를 실제 Firebase emulator 에 띄우고 end-to-end. emulator 인프라 새로 도입.

**선택: A**

**근거:**
- B 는 Firestore admin SDK mock infra 가 기존 vitest 환경에 없음. 새로 박으려면 mock 패턴 결정 + util 추출 + 테스트 작성까지 ~수 시간. R22 C8 의 plan spec 이 "legacy 케이스 회귀" 만 박혀 있고 ~150 LOC 범위라 scope 넘침.
- C 는 emulator 인프라가 픽지직에 아직 없음 (할일 README 의 🟢 후순위 그룹에 박혀 있음). 별도 작업.
- A 는 pure function 회귀 만으로도 tolerance 룰 흔들리는 회귀를 99% 잡음. 자가 합의 방지는 server 안의 11 LOC 짜리 단순 로직 (`Set` + `filter`) 이라 회귀 가능성 낮음. 활성 시즌 전에 통합 테스트로 가두는 게 정합한데, 그건 R22 C8 가 아닌 별도 작업으로 분리.

**철회 조건:** 활성 시즌 들어가기 전에 통합 테스트 박는 작업 진입 시 — 자가 합의 방지 회귀 가드 박음. ([[90_AI/프로젝트/pickzzk/할일/2026-05-18 개선사항\|할일 2026-05-18]] 에 entry 박힘)

---

### 4. production 검증 방식 — UI 풀 플로우 X, server endpoint 에 직접 curl 두 번

**옵션:**
- A) **production URL (https://pickzzk.vercel.app) 에서 viewer 페이지를 시크릿 창 두 개로 열어서 OCR + 매핑 + 제보 풀 플로우** — 실제 유저 체험 그대로.
- B) **server endpoint 에 직접 curl 두 번** — minimal gameData payload 만들어서 서로 다른 clientId 로 POST. 응답의 `consensus.promoted` 만 확인.
- C) **로컬 dev server + Firestore emulator** — production data 영향 0. 다만 emulator 인프라 없음.

**선택: B**

**근거:**
- A 는 OCR + 매핑까지 풀 플로우라 ~5 분 / 케이스. dry-run 본질이 "server consensus 가 production 에서 잘 도냐" 이고 viewer UI 자체는 별도 회귀 (mock-test-runner subagent + 직접 브라우저 audit). 풀 플로우 안 돌려도 무방.
- C 는 inf 없음. 별도 작업.
- B 는 python urllib 로 두 번 POST 1 분 안에 끝. production 에 박히는 dry-run 데이터는 활성 시즌 아직 아니라 (submissions / scrimDays 컬렉션 거의 비어 있음) 영향 작고, cleanup 도 한 도큐만 삭제하면 됨 (matching submissions 는 server batch 가 promote 시 이미 삭제).

**철회 조건:** 활성 시즌 진입 후 또는 submissions 컬렉션에 실 운영 데이터 박힌 시점에는 production 직접 dry-run 위험. emulator 인프라 박힌 다음부터는 C 로.

## 검증 결과

production dry-run (2026-05-18 09:30) 결과:

| 차수 | clientId | HTTP | promoted | matchCount | uniqueClients | gameId |
|---|---|---|---|---|---|---|
| 1차 | r22-dryrun-A | 200 | `false` | 1 | 1 | — |
| 2차 | r22-dryrun-B | 200 | **`true`** | 2 | 2 | `XW61rZW6p5EfBigkFAO9` |

확인된 path:
- viewer fetch `/api/scrim-submit` → server admin SDK 가 submission 박음 (rules tighten 우회 정합)
- 1명만 박으면 `promoted: false` + `matchCount: 1`
- 두 번째 다른 clientId 박으면 즉시 `promoted: true` + 새 gameId 발급
- composite index `submissions (status ASC, operationalDate ASC)` 빌드 + 사용 (FAILED_PRECONDITION 안 떴음)
- Firestore batch atomic — game create + 매칭 submissions delete 한꺼번에

cleanup: `scrimDays/2026-05-18/games/XW61rZW6p5EfBigkFAO9` 삭제 후 Firestore REST GET 으로 404 NOT_FOUND 확인.

회귀 테스트: vitest 401 통과 (이전 369 → R22 C8 의 gamesMatch 32 케이스 추가 = +32).

## 결과 / 영향 / 다음 조심할 것

**user 입장에서 무엇이 달라지나:**
- 시청자가 게임 결과 제보 → 두 명 이상 같은 결과 박으면 운영자 손 안 거치고 즉시 정식 결과로 올라옴. 화면 새로고침 시 결과 카드 보임.
- 운영자가 admin 큐 페이지 안 켜둬도 자동 승격 돔. 운영자는 잠자도 됨.

**활성 시즌 (호종컵 4기) 진입 가능 상태:**
- C1~C9 전부 production 배포 + 검증 완료.
- 운영자가 직접 한 일은 `firebase deploy --only firestore:rules,firestore:indexes` 한 번 + dry-run 확인.

**다음 조심할 것:**
- 활성 시즌 들어가기 전에 자가 합의 방지 (clientId dedupe + 익명 1표) 통합 테스트 박아야 함. 지금은 server 안의 11 LOC 짜리 룰이 회귀 가드 없음.
- 합의 안 된 단일 제보 (한 명만 박은 거) 는 server batch 가 안 건드림 → DB 에 영원히 pending 으로 누적. 운영일 + 24h 만료 cron 또는 TTL 룰 박아야 함.
- 두 항목 다 [[90_AI/프로젝트/pickzzk/할일/2026-05-18 개선사항\|할일 2026-05-18]] 에 entry 박힘.

**rules 의존 (배포 정합):**
- `firestore.rules` — submissions create 가 admin only 로 좁혀짐. viewer client 가 직접 박으면 PERMISSION_DENIED. server endpoint 필수.
- `firestore.indexes.json` — submissions composite (status ASC, operationalDate ASC) 없으면 server consensus 가 FAILED_PRECONDITION. 둘 다 production 에 deploy 됨.

**legacy 참조:**
- `~/workspace/draftpick/draftpick_vue/api/_lib/scrim-submit-handler.js` — RTDB 기반 원본. R22 C2 의 source. RTDB multi-path update → Firestore batch 어댑트 부분에서 가장 자주 봤음.
- gamesMatch tolerance 룰 (KDA ±1, gold ±2%, dmg ±3%, duration ±3s, 10 슬롯) 은 legacy verbatim. 회귀 가드 32 케이스 (`tests/unit/games-match.test.js`) 가 본 룰을 가둠.
