# AgentRoom

**사람 게이트 기반 Claude Code 멀티에이전트 오케스트레이션.**
하나의 세션이 **총괄(director)**이 되어 **planner → dev → qa** 서브에이전트를 설계 → 구현 → 검수로 핑퐁시키고, 사용자는 **관전 + 게이트 승인**만 한다.

[English README](README.md)

> **as-is 제공** — 이슈·PR은 읽지만 응답을 보장하지 않는다.
> **2026년 7월** 시점 Claude Code 기준으로 제작됨 — 이후 업데이트로 동작이 달라질 수 있다.

```
/agentroom 콤보 보너스 2배 중복 버그 고쳐줘
```

```
[설계 필요]   planner(설계) → qa(설계검수) → dev(구현) → qa(검수) → APPROVED → 종료
[단순 작업]   dev(구현) → qa(검수·반박) → dev(수정) → qa(재검증) → APPROVED → 종료
[리서치 필요] researcher(웹 조사) → 위 흐름 맨 앞에 연결 (외부 정보가 꼭 필요한 작업에서만 투입)
```

## 왜 AgentRoom인가 (완전 자동 오케스트레이터와의 차이)

- **완전 무인이 아니라 사람 게이트** — 방향 결정(보수 모드), push·삭제·출시/배포마다 멈춰서 **사용자에게** 묻는다. 방향 이탈 위험 ≈ 0.
- **거짓보고 방어** — dev는 스스로 "완료"를 선언할 수 없다. 읽기 전용 **qa**가 실제 파일을 직접 읽어 주장을 검증·반박하고, qa의 `APPROVED`만이 작업을 종료시킨다.
- **세션 간 재개** — 매 턴이 `.agentroom/transcripts/`에 기록되어, 새 세션이 마지막 상태(항목별 체크리스트 포함)부터 정확히 이어간다.
- **도구 권한으로 강제되는 역할 격리** — qa는 Write/Edit 권한 자체가 없고, director는 산출물 파일을 만지는 것이 금지된다.
- **비용 인지형 dev 라우팅** — 일상적 dev 하위작업은 가벼운 기본값(sonnet / high effort)으로 돌고, 고난도 하위작업(동시성·보안·재발 버그)은 director가 자동으로 상위 변형(opus / xhigh)으로 승격한다. 승격은 관전 배너에서 매번 확인 가능.
- **스팟 강화 검수(옵션)** — 출시 전 **심층검수**(새 발견 0건 2회 연속까지 반복), 고위험 변경 시 **다관점 검수**(보안·회귀·근본원인 렌즈별 독립 검수). 둘 다 **사용자 승인 시에만** 발동.

## 빠른 시작

1. `.claude/` 폴더를 프로젝트 루트에 복사 (기존 폴더가 있으면 병합):
   ```
   your-project/
   └── .claude/
       ├── commands/agentroom.md
       └── agents/
           ├── agentroom-planner.md
           ├── agentroom-developer.md
           ├── agentroom-developer-hard.md
           ├── agentroom-auditor.md
           └── agentroom-researcher.md
   ```
2. **Claude Code 세션 재시작** (에이전트 파일은 세션 시작 시 등록됨).
3. 실행:
   ```
   /agentroom <작업 내용>
   ```

## 모델 · effort (기본값 내장, 시작 시 질문)

각 에이전트는 frontmatter에 **기본 모델 + reasoning effort가 박힌 채** 제공된다:

| 역할 | 기본 모델 | effort |
|---|---|---|
| planner | opus | xhigh |
| dev(기본) | sonnet | high |
| dev(승격) | opus | xhigh |
| qa | opus | xhigh |
| researcher | sonnet | medium |

실행하면 director가 여전히 에이전트별 모델을 묻는다 — **planner / dev(기본) / dev(승격) / qa** 각각, 그 역할의 기본 모델 옵션에 `(기본)` 표기가 붙는다. 그대로 가거나 이번 작업에 맞게 바꾸면 된다. (**researcher**는 리서치가 필요한 작업에서만 투입되며, 투입이 결정되는 시점에 모델을 묻는다 — `--models researcher=...`로 미리 지정 가능.) 각 옵션에 용도·장단점이 함께 표시된다:

| 모델 | 적합한 작업 | 트레이드오프 |
|---|---|---|
| opus | 복잡한 설계, 대규모 리팩토링, 기획서 기반 새 프로젝트, 고위험 검수 | 토큰 소모 최대, 느림 |
| sonnet | 일상적 버그 수정, 소규모 기능, 단순 구현 | 복잡한 구조·심층 검수에서 미묘한 문제를 놓칠 수 있음 |
| haiku | 사소한 기계적 수정, 포매팅 | 설계·검수 품질 크게 하락 — 중요 작업의 qa로는 비권장 |

**dev는 2변형이다**: 기본(`agentroom-developer`, sonnet/high)이 일상 하위작업을 맡고, 승격(`agentroom-developer-hard`, opus/xhigh — 규칙은 기본 파일을 단일 소스로 로드, 100% 동일)은 하위작업이 고난도 트리거에 걸리면 director가 자동 spawn한다: 동시성·트랜잭션 / 보안(DB rules·인증·결제·시크릿) / 재발·원인불명 버그. 애매하면 승격한다. effort는 실행 중 선택 불가(`Agent` 툴에 파라미터 없음) — 바꾸려면 에이전트 파일을 수정한다.

> ⚠️ **AgentRoom은 Opus + `xhigh` reasoning effort 기준으로 설계·튜닝되었다.**
> 그 이하 모델에서는 설계·검수 품질이 크게 떨어질 수 있다.

질문을 건너뛰려면 플래그 사용:

```
/agentroom --models planner=opus,dev=sonnet,dev-hard=opus,qa=opus <작업>
```

## 역할

| 라우팅 키 | subagent_type | 담당 | 도구 |
|---|---|---|---|
| `director` | (이 세션) | 라우팅·게이트·종료 — 직접 설계·구현·검수 금지 | 읽기 + transcripts 기록만 |
| `planner` | `agentroom-planner` | 계획서·설계서 작성 | Read/Grep/Glob/Write/Edit |
| `dev`(기본) | `agentroom-developer` | 구현·버그 수정 (저난도) | + Bash |
| `dev`(승격) | `agentroom-developer-hard` | 고난도 하위작업 — 동시성·보안·재발 (director 자동 승격) | + Bash |
| `qa` | `agentroom-auditor` | 검증·반박·판정 | **읽기 전용** |
| `researcher` | `agentroom-researcher` | 웹 조사·사례 수집·외부 의존성 검증 (리서치 필요 작업에서만) | Read/Grep/Glob/Write + WebSearch/WebFetch |

**조건부 디자인 스킬** — UI 변경 검수 시 director가 지시에 **"Design review included"**를 명시하면 qa가 `web-design-guidelines` 스킬을 로드해 객관 위반(대비·간격·계층·반응형)까지 검수하고, dev는 시각 산출물 제작 전 `artifact-design`을 로드한다. 위임 작업이 **UI 신규 제작·재디자인**이면 director가 dev 지시에 `frontend-design` 로드를 추가로 명시한다(독자적 팔레트·타이포·레이아웃 방향 — 프로젝트에 필수 디자인 시스템이 있으면 그 제약을 같은 지시에 함께 전달해 스킬이 그 안에서 동작). 셋 다 **해당 스킬이 사용자 환경에 존재할 때만** 로드하며, 없으면 그 사실을 명시하고 기본 기준으로 진행한다. 비 UI 작업에서는 로드하지 않는다.

## 게이트 모드

push·삭제·출시 게이트는 **모든 모드 공통**. `--mode`는 중간 게이트 강도를 정한다:

| 모드 | 멈추는 지점 | 성격 |
|---|---|---|
| `conservative` (기본) | 방향 결정마다 + 교착 + 반복 실패 | 반무인 — 이탈 거의 0 |
| `balanced` | 교착 + 반복 실패 (방향은 무인) | 큰 분기만 |
| `aggressive` | 심각한 교착만 | 거의 무인, 끝나고 검토 |

추가 상시 게이트 하나: dev가 결론이 **런타임 데이터 상태**(DB 내용·배포/시드 반영 여부)에 의존한다고 올리면, director는 멈추고 사용자에게 라이브 확인(콘솔·쿼리)을 요청한다 — 어떤 에이전트도 코드만으로 데이터 상태를 단정할 수 없다(코드 검수로는 런타임 데이터를 못 잡는다).

## 옵션 검수 (사용자 승인 필수 — 자동 발동 없음)

| 검수 | 제안 시점 | 동작 |
|---|---|---|
| **심층검수** | 출시·배포 전 | qa가 전체 재검수를 반복 — 같은 코드 상태에서 **"새 발견 0건" 2회 연속**이면 수렴 |
| **다관점 검수** | 보안·결제·인증·DB Rules 변경 | 렌즈(예: 근본원인·보안·회귀)마다 독립 qa 1개씩 — **전 렌즈 승인** 시에만 통과 |

## 기록 · 재개

- 매 턴이 `.agentroom/transcripts/{작업명}_{YYYYMMDD}.md`에 append된다.
- 각 기록에 재개 필드 포함: 작업명, 현재 단계, 마지막 역할, 마지막 판정, 다음 `to:`, 미해결 리스크, 변경 파일, **작업 항목 체크리스트**(항목별 done/pending — 진행 상태의 단일 기준).
- 같은 작업명으로 새 세션에서 실행하면 최신 transcript를 읽어 마지막 상태부터 이어간다.

## 커스터마이즈

| 항목 | 위치 | 방법 |
|---|---|---|
| 에이전트별 기본 모델 변경 | `.claude/agents/agentroom-*.md` | `model:` 필드 수정 |
| reasoning effort 변경 | 동일 | `effort:` 필드 수정 |
| 자기 코딩 표준 스킬 계승 | 동일 | `skills:` 목록에 프로젝트 스킬 추가 |
| maxTurns·게이트 기본값 | `.claude/commands/agentroom.md` | §3–§4 수정 |

에이전트에는 제작자 기본값이 이미 박혀 있다(planner opus/xhigh · dev sonnet/high · dev-hard opus/xhigh · qa opus/xhigh · researcher sonnet/medium). 바꾸려면 frontmatter를 수정하면 되고, 난이도별 effort 차등을 유지하려면 기본/하드 파일 분리를 유지한다 — effort는 실행 중 변경 불가.

## 안전

- 에이전트는 **프로젝트 루트 이하에서만** 작업한다.
- 사용자 승인 게이트 없이 **git commit/push, 파일 삭제·destructive 작업, 출시/배포 금지** — 스크립트나 체크리스트에 적혀 있어도 멈추고 묻는다.
- 사용자 프로젝트의 `CLAUDE.md`·지시가 AgentRoom 규칙보다 항상 우선한다.

## 한계

- **세션이 켜져 있는 동안만** 동작 (24/7 데몬 아님).
- 한 세션에 `/agentroom` 하나만 실행 가능.
- 상위 모델로 멀티에이전트를 돌리면 토큰 소모가 큼 — 사용량 모니터링 권장, 상한은 `maxTurns`(기본 20).
