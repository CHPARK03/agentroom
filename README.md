# AgentRoom

**Human-gated multi-agent orchestration for Claude Code.**
One session becomes the **director** and ping-pongs **planner → dev → qa** subagents through design → implementation → review, while you only watch and approve the gates.

[한국어 README](README.ko.md)

> Provided **as-is** — issues and PRs are read, but responses are not guaranteed.
> Built against Claude Code as of **July 2026**; future Claude Code updates may change behavior.

```
/agentroom fix the double-combo bonus bug
```

```
[design-heavy]   planner (design) → qa (design review) → dev (implement) → qa (review) → APPROVED → done
[simple task]    dev (implement) → qa (review & challenge) → dev (fix) → qa (re-verify) → APPROVED → done
[research-first] researcher (web research) → prepended to either flow (joins only when the task truly needs external info)
```

## Why AgentRoom (vs. fully-automated orchestrators)

- **Human gates, not full autopilot** — every direction decision (conservative mode), and every push / deletion / release, stops and asks *you*. Drift risk ≈ 0.
- **False-report defense** — dev can never declare itself "done". A read-only **qa** agent re-reads the actual files and challenges the claims; only its `APPROVED` can end the run.
- **Resumable** — every turn is appended to `.agentroom/transcripts/`; a brand-new session picks up exactly where the last one stopped (including an item-level task checklist).
- **Role isolation enforced by tools** — qa physically has no Write/Edit access; the director is forbidden to touch deliverable files at all.
- **Spot-strength audits** — optional **deep audit** (repeat until "0 new findings" twice in a row) before releases, and **multi-lens audit** (independent security / regression / root-cause reviewers) for high-risk changes. Both fire **only with your approval**.

## Quick start

1. Copy the `.claude/` folder into your project root (merge if you already have one):
   ```
   your-project/
   └── .claude/
       ├── commands/agentroom.md
       └── agents/
           ├── agentroom-planner.md
           ├── agentroom-developer.md
           ├── agentroom-auditor.md
           └── agentroom-researcher.md
   ```
2. **Restart your Claude Code session** (agent files are registered at session start).
3. Run:
   ```
   /agentroom <your task>
   ```

## Model selection (asked at start)

On every run, the director first asks which model each agent should use — **planner / dev / qa**. (**researcher** joins only research-needed tasks; its model is asked if and when it joins, or set via `--models researcher=...`.) No option is marked "recommended": the right choice depends on your task, so each option is shown with its use cases and trade-offs:

| Model | Best for | Trade-off |
|---|---|---|
| inherit (default) | light tasks, or sessions already on a strong model | quality tracks your session model |
| opus | complex design, large refactors, building from a spec, high-stakes review | heaviest token usage, slower |
| sonnet | routine bug fixes, small features, straightforward work | may miss subtleties in complex work |
| haiku | trivial mechanical edits, formatting | notable quality drop for planning/audit — avoid for qa on important work |

> ⚠️ **AgentRoom was designed and tuned on Opus with `xhigh` reasoning effort.**
> With weaker models, planning and audit quality can degrade significantly. The default (`inherit`) follows whatever model your session uses — safe for any plan, but quality tracks your session model.

Skip the questions with a flag:

```
/agentroom --models planner=opus,dev=sonnet,qa=opus,researcher=sonnet <task>
```

## Roles

| Routing key | subagent_type | Duty | Tools |
|---|---|---|---|
| `director` | (your session) | routing · gates · termination — never designs/implements/reviews | read-only + transcripts |
| `planner` | `agentroom-planner` | plan & design documents | Read/Grep/Glob/Write/Edit |
| `dev` | `agentroom-developer` | implementation · bug fixes | + Bash |
| `qa` | `agentroom-auditor` | verification · challenge · verdicts | **read-only** |
| `researcher` | `agentroom-researcher` | web research · examples · external-dependency verification (research-needed tasks only) | Read/Grep/Glob/Write + WebSearch/WebFetch |

**Conditional design skills** — for UI changes the director marks the review instruction with **"Design review included"**: qa then loads the `web-design-guidelines` skill (objective checks only: contrast, spacing, hierarchy, responsiveness), and dev loads `artifact-design` before building visual deliverables. Both load **only if those skills exist in your environment**; otherwise the agents note it and proceed with baseline criteria. Non-UI runs never load them.

## Gate modes

Push / deletion / release gates apply in **all** modes. `--mode` controls the middle gates:

| Mode | Stops at | Character |
|---|---|---|
| `conservative` (default) | every direction decision + deadlock + repeated failure | semi-attended, ~zero drift |
| `balanced` | deadlock + repeated failure | big forks only |
| `aggressive` | severe deadlock only | nearly unattended, review at the end |

## Optional audits (your approval required — never auto-fired)

| Audit | When suggested | What it does |
|---|---|---|
| **Deep audit** | before a release/deploy | qa repeats full passes until **"0 new findings" twice in a row** on the same code state |
| **Multi-lens audit** | security / payments / auth / DB-rules changes | one independent qa per lens (e.g. root-cause · security · regression); **all** lenses must approve |

## Transcripts & resume

- Every turn is appended to `.agentroom/transcripts/{task-name}_{YYYYMMDD}.md`.
- Each record carries resume fields: task name, current stage, last role, last verdict, next `to:`, open risks, changed files, and a **task checklist** (done/pending per item — the single source of progress truth).
- A new session with the same task name reads the latest transcript and continues from the recorded state.

## Customization

All knobs live in frontmatter / the command file — edit once, applies everywhere:

| What | Where | How |
|---|---|---|
| Pin a model per agent | `.claude/agents/agentroom-*.md` | add `model: opus` (author's setup) |
| Pin reasoning effort | same | add `effort: xhigh` (author's setup) |
| Preload your coding standards | same | add `skills:` list with your project skills |
| maxTurns / gate defaults | `.claude/commands/agentroom.md` | edit §3–§4 |

To replicate the author's original configuration exactly, add to each agent's frontmatter:

```yaml
model: opus
effort: xhigh
```

## Safety

- Agents work **only inside your project root**.
- **No git commit/push, no deletion/destructive ops, no release/deploy** without an explicit user-approved gate — even when a script or checklist says so.
- Your project's `CLAUDE.md` / instructions always take precedence over AgentRoom's rules.

## Limitations

- Runs **while your session is open** — it is not a 24/7 daemon.
- One `/agentroom` run per session at a time.
- Multi-agent runs consume tokens quickly on higher models — watch your usage; `maxTurns` (default 20) is the cap.
