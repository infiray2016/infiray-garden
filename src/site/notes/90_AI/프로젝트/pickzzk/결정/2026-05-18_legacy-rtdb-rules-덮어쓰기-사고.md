---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/결정/2026-05-18_legacy-rtdb-rules-덮어쓰기-사고.md","permalink":"/프로젝트/pickzzk/결정/2026-05-18_legacy-rtdb-rules-덮어쓰기-사고/","tags":["프로젝트/pickzzk","결정","rtdb","firebase-rules","incident","legacy-draftpick"],"dg-note-properties":{"tags":["프로젝트/pickzzk","결정","rtdb","firebase-rules","incident","legacy-draftpick"],"project":"pickzzk","date":"2026-05-18","status":"박힘 + production 복구 완료","commits":["6a50aa6","e4573f6"],"related-source":["~/workspace/draftpick/draftpick_vue/docs/firebase-rules.md"],"incident-window":{"start":"2026-05-16T19:38","end":"2026-05-18T12:25","duration":"~2일"},"impact":"legacy DraftPick 사이트 (호종컵 4기 운영 중) 의 모든 RTDB 기능 PERMISSION_DENIED"}}
---


# 옛 DraftPick RTDB 권한 룰 통째로 덮어씌워진 사고 (2 일 운영 중단)

> 2026-05-16 의 pickzzk 멀티 경매 회귀 fix 작업 중 `database.rules.json` 신설 + `firebase deploy --only database` 박은 게 같은 Firebase 프로젝트를 공유하는 옛 DraftPick 의 RTDB 권한 룰을 통째로 덮어씌웠다. 옛 사이트의 시청자 승부 예측 / 제보 / 일정 / 팀 / 별칭 / 매치업 분석 / 뉴스레터 / 일간 브리핑 등 9 개 경로가 2 일 동안 권한 거부 상태. 2026-05-18 user 가 옛 사이트 사용 중 발견 + 즉시 복구.

## 배경

같은 Firebase 프로젝트 (`draftpick-63456`) 를 두 repo 가 공유한다:

- **옛 DraftPick (`~/workspace/draftpick/draftpick_vue/`)** — 호종컵 4기 운영 중. RTDB 사용 (스크림 데이터 / 시청자 제보 / 시청자 예측 / 일정 / AI 매치업·뉴스레터 등 9-10 개 sub-tree).
- **픽지직 (`~/workspace/pickzzk/`)** — v2, 활성 시즌 직전. RTDB 사용은 `rooms/auction-XXXX` / `rooms/snake-XXXX` (모의 드래프트·경매 멀티 모드) 만.

Firebase Realtime Database 의 권한 룰은 **프로젝트당 단 하나의 트리만** 존재한다. 한쪽 repo 가 `firebase deploy --only database` 박으면 그 repo 의 `database.rules.json` 으로 production 룰이 통째로 교체된다. 즉 한쪽이 deploy 하면 다른 쪽의 기존 룰은 silent 하게 사라진다.

2026-05-16 (`e4573f6`) commit 메시지:

> fix(multi-auction): 실데이터 E2E 회귀 4건 fix — RTDB rules / 첫 lot / lot 이름 / 닉네임 중복
>
> 1. RTDB rules 미배포 — 방 생성 PERMISSION_DENIED
>    - database.rules.json 신설 (rooms/{auction|snake}-{4-8 char code} 만 read/write)
>    - firebase.json 에 database section 추가
>    - `firebase deploy --only database` 로 live 배포 완료 (draftpick-63456)

이 시점 픽지직의 `database.rules.json` 모양:

```json
{
  "rules": {
    ".read": false,
    ".write": false,

    "rooms": {
      "$roomKey": {
        ".read": "$roomKey.matches(/^(auction-)?[A-Z0-9]{4,12}$/) || $roomKey.matches(/^snake-[A-Z0-9]{4,12}$/)",
        ".write": "$roomKey.matches(/^(auction-)?[A-Z0-9]{4,12}$/) || $roomKey.matches(/^snake-[A-Z0-9]{4,12}$/)"
      }
    }
  }
}
```

root 가 deny + `rooms/*` 만 화이트리스트. **옛 DraftPick 의 모든 `scrim*` 경로는 매칭 안 됨 → root deny 로 떨어져서 PERMISSION_DENIED.**

발견 경위 — 2026-05-18 R22 + 옵시디언 narrative 작업 마무리 시점에 user 가 옛 DraftPick 사이트의 console error 를 공유:

> "viewerPredictions 구독 실패: permission_denied at /scrimViewerPredictions: Client doesn't have permission to access the desired data. 이런 오류가 드래프트 픽 프로젝트에서 나는데 왜 나는지 알 수 있어 ?"

즉시 원인 추적 → `e4573f6` 의 silent overwrite 사고 확정.

## 결정 3건

### 1. 옵션 A (픽지직 rules 에 옛 DraftPick path 합치기) 채택. B/C 거부

**옵션:**
- A) **픽지직 `database.rules.json` 에 옛 DraftPick 10 path 추가 + 같이 deploy** — 한 rules 트리에서 두 영역 (`rooms/*` + `scrim*`) 다 살림. 옛 DraftPick 코드 변경 0 (freeze 룰 정합).
- B) **Firebase 프로젝트 분리** — 픽지직을 새 Firebase 프로젝트로 옮기고 옛 DraftPick 은 기존 프로젝트 그대로. RTDB rules 트리가 분리되니까 사고 재발 영구 차단.
- C) **옛 DraftPick repo 에 자체 `database.rules.json` 박고 별도 deploy** — 두 repo 가 같은 프로젝트의 rules 트리를 번갈아 deploy. 어느 쪽이든 마지막으로 deploy 한 쪽이 이김.

**선택: A**

**근거:**
- B 는 큰 작업. Firebase 새 프로젝트 + Vercel env vars + 도메인 + Firestore 데이터 마이그까지. 활성 시즌 (호종컵 4기) 직전이라 risk 큰 변경.
- C 는 사고 원인 자체를 해소 못 함. 마지막에 deploy 한 쪽이 다른 쪽 룰을 silent 하게 덮는 구조는 그대로 — 다음 누가 deploy 했는지 헷갈리면 또 발생.
- A 는 같은 rules 트리 안에서 namespace 만 분리. 픽지직 `rooms/*` 와 옛 DraftPick `scrim*` 가 path 겹침 0. 한 commit (`6a50aa6`) + 한 deploy 명령으로 끝.

**철회 조건:** 호종컵 4기 운영 끝나서 옛 DraftPick 폐기하는 시점에 영구적으로 B 로 이동. 본 commit (`6a50aa6`) 의 `scrim*` 10 path 도 같이 제거. → [[90_AI/프로젝트/pickzzk/할일/2026-05-18 개선사항\|2026-05-18 개선사항]] entry 박힘.

---

### 2. 복원 source 는 옛 DraftPick 의 `docs/firebase-rules.md` 권장 rules + 코드 grep 으로 누락 1 개 보강. 보수적 박기 (전부 read true) 거부

**옵션:**
- A) **보수적 빠른 fix** — 옛 DraftPick 이 박는 9 개 경로 전부 `.read: true, .write: true` 로 열어두기. 가장 빠르지만 보안 표면 큼.
- B) **옛 DraftPick 의 `docs/firebase-rules.md` 권장 rules 그대로** — `.read: true, .write: false` 가 default (서버 Admin SDK 만 우회로 쓰기). `scrimRateLimits` 만 read/write 양쪽 false. legacy 가 production 운영하던 시점의 spec 그대로.
- C) **A 박고 나중에 정정** — 일단 열어두고 운영 검증 후 좁히기.

**선택: B + 코드 grep 으로 누락 path 1 개 추가 (`scrimMatchups`)**

**근거:**
- A 는 `scrimSubmissions` 의 `.write: true` 가 viewer 가 직접 박는 길을 다시 열어줌 — 옛 DraftPick 이 박은 rate limit (IP 당 시간당 10회) + server 측 검증 (Anthropic API + Admin SDK + SCRIM_ADMIN_SECRET 검증) 를 우회. 보안 표면 큼.
- C 는 한 번 박힌 룰 좁히는 게 복구 후 운영 중 (호종컵 4기 시즌 중) 위험. 잘못 좁히면 다시 PERMISSION_DENIED.
- B 는 옛 DraftPick 이 e4573f6 직전에 production 으로 운영하던 spec 자체 — 옛 사이트 흐름 보장.
- 옛 DraftPick 의 가이드 문서가 작성 후 추가된 path (`scrimMatchups` — AI 매치업 분석) 가 빠짐. 옛 repo 의 `src/` + `api/` 의 실제 RTDB `ref()` 호출처 grep 으로 보강.

**철회 조건:** B 의 권장 spec 에 신뢰 못 가는 path 발견 시 (예: 옛 DraftPick 운영 중 새로 발생한 보안 사고) 그 path 만 추가 좁힘. 다만 옛 DraftPick 자체가 호종컵 4기 폐기 예정이라 보수적 좁힘은 비용 대비 효익 낮음.

---

### 3. 사고 재발 방지 가드는 hook 으로 박는다 — 후속 작업, 본 commit 에는 미포함

**옵션:**
- A) **본 복구 commit (`6a50aa6`) 에 pre-push hook 같이 박기** — `database.rules.json` 변경 시 legacy 10 path 누락 검사하는 자동 가드.
- B) **복구만 박고 hook 은 별도 commit** — 사고 발견 + 복구가 본질, 가드 박기는 별도 작업.
- C) **hook 박지 않음** — 활성 시즌 들어가면 옛 DraftPick 안 건드릴 거고 사고 재발 가능성 낮음.

**선택: B**

**근거:**
- A 는 사고 복구의 긴급성 + 가드 박기의 별도 설계 (hook script 위치, 검사 룰, 실패 메시지 등) 가 한 commit 에 묶이면 review 어려움.
- C 는 너무 낙관적. e4573f6 박은 시점에도 사고를 의도하지 않았음 — 다음에 또 `database.rules.json` 만지면 같은 사고 재발 가능. 가드 박는 것이 cheap.
- B 는 복구 priority 높음을 우선 처리 + 가드는 옵시디언 [[90_AI/프로젝트/pickzzk/할일/2026-05-18 개선사항\|2026-05-18 개선사항]] 에 entry 로 박음. 활성 시즌 직전 마무리 작업에서 진입.

**철회 조건:** 호종컵 4기 폐기 + 픽지직 Firebase 프로젝트 분리 (결정 1 의 철회 조건) 가 진행되면 본 가드 자체가 dead — 두 repo 가 다른 프로젝트라 silent overwrite 가능성 0. 가드 hook 도 같이 제거.

## 결과 / 영향 / 다음 조심할 것

**옛 DraftPick 사이트 영향 (incident-window 2 일):**
- 시청자 승부 예측 (scrimViewerPredictions) — 투표 시도 시 PERMISSION_DENIED
- 시청자 제보 큐 (scrimSubmissions) — viewer 가 OCR 제보 박지 못함
- 스크림 일정 / 팀 / 별칭 / 매치업 / 뉴스레터 / 일간 브리핑 — 화면 빈 상태
- 호종컵 4기 시청자 경험 직접 손상. 운영자 (user) 가 사이트 직접 보지 않으면 ~수 일 단위로 미발견 가능했음.

**복구 후 (2026-05-18 12:25 deploy 완료):**
- 옛 DraftPick 의 9 개 viewer-facing path 즉시 정상 동작 (`.read: true`)
- 9 개 path 의 client write 는 여전히 막힘 (`.write: false`) — server Admin SDK 만 우회. 옛 DraftPick 의 보안 spec 정합.
- `scrimRateLimits` 는 read/write 양쪽 false — counter 노출 X (옛 DraftPick 권장 spec 그대로).

**픽지직 영향 (이번 복구로):**
- `rooms/*` 룰 그대로 유지 → 모의 드래프트 / 모의 경매 멀티 모드 동작 변화 0.
- root deny 유지 → 새 path 외 default deny.
- 회귀 (`bun run verify`) 통과: vitest 401 / lint 0 errors / build green.

**보안 표면 변화:**
- e4573f6 직전 (옛 DraftPick 단독 운영 시점) 의 rules spec 으로 되돌아간 것. 그동안 production 으로 운영하던 룰이라 새로 생기는 risk 0.
- 사이드 이펙트: 픽지직 자체 코드가 `scrim*` 경로 안 쓰니까 픽지직 viewer 가 read 해도 의미 없는 데이터. 보안 risk 0.

**다음 조심할 것:**
1. **`database.rules.json` 만지는 모든 작업에서 legacy 10 path 가 빠지지 않았는지 확인.** 다음에 픽지직이 `rooms/*` 룰 좁히거나 새 sub-tree 추가할 때 옛 DraftPick path 도 그대로 살아 있는지 review 단계에서 grep. 자동 가드 hook 박기 entry 가 [[90_AI/프로젝트/pickzzk/할일/2026-05-18 개선사항\|2026-05-18 개선사항]] 에.
2. **`firebase deploy --only database`** 박기 직전에 `git diff database.rules.json` 확인 — 의도한 변경만 있는지. 자동 가드 박히기 전까지는 manual review 필수.
3. **호종컵 4기 폐기 시점 = Firebase 프로젝트 분리 시점.** 이때 본 commit (`6a50aa6`) 의 `scrim*` 10 path 도 같이 제거. [[90_AI/프로젝트/pickzzk/할일/2026-05-18 개선사항\|2026-05-18 개선사항]] entry 박힘.
4. **legacy freeze 룰의 사각지대 발견** — 옛 DraftPick 코드는 freeze 라서 안 건드린다는 룰이 박혀 있는데, 같은 Firebase 프로젝트의 RTDB rules 는 픽지직 쪽에서 deploy 가능했음. freeze 룰이 "코드만" 가두고 production 환경 공유 자원 (Firebase 프로젝트 / 도메인 / env vars) 은 안 가둠. 다음에 옛 DraftPick 영향 있는 환경 변경 (예: storage rules / functions / IAM 등) 할 때도 같은 패턴 주의.

**관련:**
- [[90_AI/프로젝트/pickzzk/결정/2026-05-18_R22-server-consensus-promotion\|2026-05-18_R22-server-consensus-promotion]] — 같은 날 작업한 R22 narrative. 둘이 시간순 연결.
- [[90_AI/프로젝트/pickzzk/할일/2026-05-18 개선사항\|2026-05-18 개선사항]] — 이번 사고로 박힌 추가 entry 6 개.
- 옛 DraftPick legacy 권장 spec: `~/workspace/draftpick/draftpick_vue/docs/firebase-rules.md`
- 픽지직 firebase rules skill: [[90_AI/claude/개념/스킬\|스킬]] `firebase-rules` SKILL.md
