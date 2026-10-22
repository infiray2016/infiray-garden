---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/결정/2026-06-06_운영-로컬-데이터-분리-isTest-flag.md","permalink":"/프로젝트/pickzzk/결정/2026-06-06_운영-로컬-데이터-분리-isTest-flag/","dg-note-properties":{}}
---

# 운영 / 로컬 테스트 데이터 분리 — isTest flag 옵션 — 2026-06-06

## 무엇

`src/firebase.js` 가 legacy DraftPick 과 같은 firebase 프로젝트(`draftpick-63456`)
공유. dev/prod 분리 X — 로컬 dev 빌드에서 firestore 에 박는 모든 doc 가 운영
viewer 에 그대로 노출.

해결: **`isTest: true` flag** 박고 viewer query 가 client-side filter 로 자동
제외. 운영자 (super) 만 `?showTest=1` 또는 sessionStorage flag 로 디버그 시 전체 표시.

## 왜

user 보고 — "운영데이터와 로컬테스트 데이터 분리가 필요해 보임. 지금 로컬에서
개발하면서 테스트하는 데이터가 운영에도 적용되는 문제가 있음."

연쇄 영향:
- 지표(/stats) 가 dev test 게임에 묻힘
- 활성 홈에 자낳대 fixture (`janahdae-s2-staff-pair-mock`) 같은 test 도큐가 노출
- 운영 viewer 가 더러워짐

대안 검토 (DATA_ISOLATION_PLAN.md):
- **A 별도 firebase 프로젝트** — 완전 격리, user 셋업 부담 (firebase console
  생성 + billing + rules deploy + 데이터 채움). long-term 안전.
- **B isTest flag** ← 채택. 인프라 변경 0, 코드 변경만으로 즉시 적용. 인프라
  바꾸기 전 임시 해결이 아니라 본질 fix 로도 충분 (단독 운영 + dev write 양이
  많지 않음).
- C collection prefix — collectionGroup 부정확, 비추천.
- D dev firebase 자체 비활성 — 비현실적.

## user 입장에서 무엇이 달라지는지

1. 지금부터 dev 빌드에서 새로 만드는 tournament/games/mockResults/streamers 는
   자동으로 `isTest:true` 박힘. 운영 viewer 에서 안 보임.
2. 운영 (build production) 에선 `isTest:false` — 운영 viewer 에 정상 노출.
3. 운영자 (super) 가 디버그 필요 시 `?showTest=1` 박으면 전체 표시.
4. 기존 운영 firestore 에 박힌 dev pollution 은 user 가 `bun run mark:test-data
   --dry` 로 후보 확인 후 `--apply` 로 일괄 mark — 자동 viewer 제외.

## 구현 (4 commit + plan + 옵시디언)

- `ad7b725` plan — DATA_ISOLATION_PLAN.md 4 옵션 비교, B 추천
- `bun a409cd4` D1 — dataEnv + write 사용처 4건 + 단위 테스트
- `265365d` D2 — viewer query 4건 filter (streamers 제외)
- `4aebacf` D3 — scripts/mark-test-data.mjs + npm script

deferred (Phase 4):
- firestore.rules 의 isTest enum 가드 — dev 가 거짓말 못하게 강제. user 단독
  운영이라 우선순위 낮음. 인원 늘어나면 그 때 추가.
- 자동 status 'live'→'done' 전환 — 운영자 의도 명확화 필요.

## 산출물

- spec: `.claude/DATA_ISOLATION_PLAN.md`
- core: `src/utils/dataEnv.js` (write flag + viewer filter + showTest mode)
- write 4건: useScrimSubmit/useMockResults/useStreamerCrud/useTournamentSetup
- viewer 3건: useTournamentAggregate/useActiveTournament/useMockResults
- 마이그: `scripts/mark-test-data.mjs` + `bun run mark:test-data`
- 단위 테스트 7건 (`tests/unit/data-env.test.js`)

## 다음 user 직접

1. `.env` 의 `FIREBASE_SERVICE_ACCOUNT` 확인.
2. `bun run mark:test-data` (dry-run) — 후보 출력 확인.
3. 필요 시 `bun run mark:test-data --apply` (실제 mark).
4. 운영 viewer 진입 (활성 홈 / 지표) → test pollution 사라졌는지 visual 확인.
5. dev 작업 재개 시 자동으로 isTest:true 박힘 — 별도 행동 X.
