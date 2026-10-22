---
{"dg-publish":true,"dg-path":"claude/INDEX.md","permalink":"/claude/INDEX/","tags":["index","claude","gardenEntry"],"updated":"2026-10-15","dg-note-properties":{"tags":["index","claude","gardenEntry"],"updated":"2026-10-15"}}
---


# 🧠 Claude 세컨드 브레인

Claude (Anthropic Claude Code CLI) 작업하면서 쌓이는 지식·결정·질문 정리.

> [!info] 사용 룰
> - 노트 만들 때 → 적절한 폴더 골라 박음. 헷갈리면 `프로젝트/<이름>/헷갈리는점/` 인박스.
> - 결정·논의·기억은 `YYYY-MM-DD_제목.md` (시간순 검색 쉬움).
> - 페이지 간 link 는 `[[페이지명]]`. 폴더 다르면 `[[폴더/페이지명]]`.

## 어디로 갈까?

| 폴더 | 들어갈 내용 |
|---|---|
| `개념/` | **프로젝트 무관** Claude Code mechanics (하네스, 에이전트, 스킬 등) |
| `프로젝트/<이름>/` | 프로젝트 단위 컨텍스트. 위키/결정/논의/기억/헷갈리는점/할일 nest |
| `아카이브/` | 끝난 프로젝트 / 폐기된 결정 / 옛 버전 위키 (보존용) |
| `../지식저장소/` | **프로젝트 무관** LLM Wiki 방식 지식 저장소의 설계 · 운영 규칙 · 결정 · 도구 정본(훅 · 스킬 · 템플릿). 프로젝트 폴더는 `--init` 으로 만든다 |

> [!abstract] 볼트 규칙
> 루트 `CLAUDE.md` — 클로디언/클로드코드가 매 세션 읽는 볼트 규칙 (25줄). 배경은 [[90_AI/claude/개념/클로디언\|개념/클로디언]].

## 개념 (프로젝트 무관)

- [[90_AI/claude/개념/하네스\|개념/하네스]] — R1-R22 룰 체계 / 반복 실수 방지 (HARNESS.md)
- [[90_AI/claude/개념/에이전트\|개념/에이전트]] — Claude (LLM) 자체. system / user / tool 흐름
- [[90_AI/claude/개념/서브에이전트\|개념/서브에이전트]] — Task tool 로 부르는 isolated 에이전트
- [[90_AI/claude/개념/스킬\|개념/스킬]] — `.claude/skills/<name>/SKILL.md` 자동 invoke
- [[90_AI/claude/개념/슬래시-커맨드\|개념/슬래시-커맨드]] — `/command` 호출 (스킬 또는 built-in)
- [[90_AI/claude/개념/훅\|개념/훅]] — SessionStart / PostToolUse / Stop 등 자동 실행
- [[90_AI/claude/개념/메모리\|개념/메모리]] — `~/.claude/projects/.../memory/` MEMORY.md 인덱스
- [[90_AI/claude/개념/클로디언\|개념/클로디언]] — Claude Code를 옵시디언 사이드바에 넣는 플러그인 (`realclaudian`, 2.2.5 설치됨)
- [[90_AI/claude/개념/옵시디언-플러그인\|개념/옵시디언-플러그인]] — 플러그인 고르는 기준(운영 비용) · 카탈로그 · 이 볼트에 깐 것과 안 깐 것
- [[90_AI/claude/개념/도표-만들기\|개념/도표-만들기]] — Excalidraw / 캔버스 / SVG 를 언제 쓰나 · 어떤 스킬·스크립트로 만드나

## 지식저장소 (프로젝트 무관 방법론 · 도구)

- [[90_AI/지식저장소/README\|지식저장소]] — 원자료 → 위키 → index · log 방식, 사용자 수준 훅이 자동 기록 · 검사, `--init` 으로 프로젝트 폴더 생성. 2026-10-15 모모에서 분리

## 프로젝트

- [[90_AI/프로젝트/pickzzk/README\|pickzzk]] — 치지직 스트리머 대회 시청자 트래커
- [[90_AI/프로젝트/dbdzzk/README\|dbdzzk]] — 데바데 내전 진행 도구 (2026-08-19 중단) · [[90_AI/프로젝트/dbdzzk/리서치/README\|경쟁 사이트 리서치]] · [[90_AI/프로젝트/dbdzzk/데바데-내전-생태계.excalidraw\|생태계 캔버스]]
- [[90_AI/프로젝트/momo-english-academy/README\|momo-english-academy]] — 모모 잉글리시 아카데미 · 초4부터 혼자 하는 영어 학습 웹 앱(설계 단계) · 원자료/위키(제품 · 방법론 · 매뉴얼 · 소개 · 공통)/일지/그림 구조(LLM Wiki 방식, 다른 프로젝트와 다름) · [[90_AI/프로젝트/momo-english-academy/index\|목차]] · [[90_AI/프로젝트/momo-english-academy/위키/문서-관계\|문서 관계]]

## 워크플로 예시

> [!note] "Claude 가 자꾸 같은 실수 함"
> `프로젝트/<이름>/기억/` 에 룰 박음. 다음 세션 prompt 에 직접 박거나 프로젝트 룰에 반영.

> [!note] "오늘 큰 결정 박힘"
> `프로젝트/<이름>/결정/YYYY-MM-DD_제목.md` 박음. 배경 / 옵션 / 선택 / 근거 / 철회조건.

> [!note] "이 개념 다시 헷갈림"
> `개념/<개념>.md` 정독. 부족하면 추가.

> [!note] "프로젝트 끝남"
> `프로젝트/<이름>/` 통째로 `아카이브/YYYY_<이름>/` 으로 옮김.

## 태그 컨벤션

- `#개념/하네스` — 개념 페이지
- `#프로젝트/pickzzk` — 프로젝트 노트
- `#결정` `#논의` `#기억` `#질문` `#할일` — 노트 종류
- `#open` `#resolved` `#archived` — 상태
