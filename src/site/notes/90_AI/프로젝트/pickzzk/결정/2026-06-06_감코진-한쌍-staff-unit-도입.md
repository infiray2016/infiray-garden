---
{"dg-publish":true,"dg-path":"프로젝트/pickzzk/결정/2026-06-06_감코진-한쌍-staff-unit-도입.md","permalink":"/프로젝트/pickzzk/결정/2026-06-06_감코진-한쌍-staff-unit-도입/","dg-note-properties":{}}
---

# 감코진 한쌍 (staff-unit) 도입 + 모의 경매 정돈 — 2026-06-06

## 무엇

자낳대 이번 시즌 룰 — 감독제 경매인데 팀 입찰 주체가 감독 1명이 아니라
"감독 + 코치 한쌍"(감코진). 한쌍이 한 팀 주체로 경매 참여. 감코진 자체는
매물 아님. 이걸 기존 픽지직 룰 모델에 직교 한 축으로 추가:

```
teamOwnerKind: 'single-coach' | 'staff-unit'  (default: 'single-coach')
staffUnits: [{ directorId, assistantCoachId, displayName, teamName }, ...]
```

기존 `pickAuthority: 'coach' | 'captain'` 축은 그대로. 감독제 일 때만 의미 있는
sub 축. opt-in (셋업에서 명시적으로 'staff-unit' 박을 때만 발동) — 기존
single-coach 동작 byte-identical 회귀 보장.

## 왜

- 기존 `coachAssignment` 축에 끼우면 "감독제인데 coaches 가 2명씩" 같은
  애매한 상태가 됨. 데이터 모델 깨짐.
- "감독제 + 팀 주체가 2인" 은 픽 주체 (감독/팀장) 와 다른 차원. 직교 분리가
  깔끔.
- `coachId` mirror=directorId 박으면 73곳 legacy reader 호환 자동.

## user 입장에서 무엇이 달라지는지

1. 자낳대식 감독+코치 한쌍 입찰을 셋업에서 박고 모의에서 끝까지 돌릴 수 있음.
2. 셋업 카드 02 ("룰 · 형식") 에서 감독제 선택 시 "감독 단독 / 감코진 한쌍"
   sub-radio 노출. staff-unit 선택 시 카드 04 출전 명단 에 감코진 짝짓기
   pair editor 노출.
3. 모의 경매 lobby + board + 결과 화면 모두 "감독 X · 코치 Y" 두 줄 표시.
   Lobby 슬롯 아바타 = director 40px + assistant 24px overlap.
4. 기존 호종컵 등 single-coach 대회는 100% 동일 동작. 셋업/UI 변화 없음.

## 발견 + fix (visual 검증 후)

8건 user 보고 — visual smoke 진입해서 발견:

| # | 이슈 | fix |
|---|---|---|
| 1 | 별칭(nickname) 적용 안됨 | enrichPlayer / coaches enrich 둘 다 streamerDisplayName 통일 (`if (p.id && p.name) return p` 가드 제거) |
| 2 | "감코진 1/2" fallback 라벨 | 항상 "감독 ? · 코치 ?" 형식 |
| 3 | Lobby 코치 사진 없음 | director 40px + assistant 24px overlap thumb |
| 4 | "평균 200 pt" UI 잡음 | TeamCards footer 라벨 제거 (AI 는 같은 값 자체 계산 — logic 영향 0) |
| 5 | 매물 큐 헤더 풀 오렌지 띠 | soft 배경 + 좌측 3px accent |
| 6 | AI 130포만 지름 | rating 보너스 (1.3/1.4/1.6) + jump bid + 잔여슬롯 1 가드 |
| 7 | Board 감코 사진 없음 | director 20px + assistant 20px inline thumb |
| 8 | 매물 큐 "~시세" 정보 부족 | 티어 약자 (E4) + ★rating + 모스트 1챔 으로 교체 |
| 9 | footer 보더 line 끊김 | MockBottomBar 자체 박스 자립 (`border-top` 복원) |

## 다음

- staff-unit 추가 시나리오 (스네이크 + 감독제, 팀장제 + staff-unit 변형) — 자낳대
  이번 시즌 범위 외, MVP 후속.
- 결과 공유 PNG (domToPng) 의 감코진 두 줄 표시 — Result 컴포넌트엔 박혀있지만
  export 캔버스 캡처 시 어떻게 그려지는지 확인 필요.
- 활성 홈 Stage1Hero rule badge 의 staff-unit 라벨 ("경매 · 감독제 · 감코진").
- AI bidding 의 다른 시나리오 (낮은 rating + high marketValue, 잔여 예산
  과반인 상태에서 매물 등장 등) 통합 시뮬.

## 산출물

- 12 commits: `094e666` C1 / `c1cc810` C2 / `6009d5c` C3 / `96c43b1` C4 /
  `ac28a30` dev fixture toggle / `f2ad816` MockAuctionSandbox staff display /
  `5ef3075` B1 / `74e06f4` B2 / `32b7fa9` B3 / `468195d` B4 / `198952f` B5 /
  `8233de2` B6
- governing 문서: `.claude/JANAHDAE_STAFF_PAIR_AUCTION_PLAN.md` (입력 spec) +
  `.claude/AUCTION_RULE_DESIGN.md` (직교 확장 박음) + `.claude/skills/feature-mock-simulator/SKILL.md`
- fixture: `src/data/mock-tournament-mock-staff.json` (자낳대식 4쌍 감코진 + 20 선수)
- 단위 테스트: normalize +5 / formation-view +8 / auction-engine +10 / decideBid +11 = +34
- 회귀: mock smoke 7/7 single-coach byte-identical 보장
