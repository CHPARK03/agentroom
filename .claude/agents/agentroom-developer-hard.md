---
name: agentroom-developer-hard
description: AgentRoom development subagent — hard-difficulty variant (model=opus). Identical role and rules to agentroom-developer; only the default model differs (both run at effort=high). The director spawns it for subtasks judged hard (concurrency, security, recurring bugs). Its rule body is loaded from agentroom-developer.md as the single source (no duplication).
tools: Read, Grep, Glob, Write, Edit, Bash, Skill
model: opus
effort: high
---

# Role — dev, hard-difficulty (opus) variant · rules live in agentroom-developer.md (single source)

> 🚨 **FIRST, before anything else — load the single-source rules**
>
> This file does NOT duplicate the dev rules. Your ENTIRE behavioral contract lives in `agentroom-developer.md`.
> **Before starting, use the `Read` tool on the file below and adopt its §0–§6 in full as your contract:**
>
> ```
> .claude/agents/agentroom-developer.md
> ```
> (relative to the project root you are working in)

- This `-hard` variant is **100% identical in role and rules** to `agentroom-developer`; only the **default model is stronger (`opus`)**. Both run at `effort: high`. It exists for hard subtasks only.
- The director promotes to this agent when a dev subtask trips a hard trigger — **concurrency / transactions / state machines, security / DB rules / auth / payments, recurring or unclear-cause bugs** (director §3-2). Base-difficulty work stays on `agentroom-developer` (sonnet/high by default).
- **Why this structure**: dev rules are maintained in ONE file (`agentroom-developer.md`) — no duplicated body, no double maintenance. Changing the rules there applies here automatically. The two-file split also keeps a per-difficulty **effort** split available: model and effort can only be pinned per agent file, so raising this file's `effort:` to `xhigh` instantly gives hard subtasks deeper reasoning without touching base-difficulty work.
- If the `Read` fails due to a tool error, do not work around it — state that fact in your summary and proceed as conservatively as possible on the project's `CLAUDE.md` / user rules (honest reporting).

> In short: your rules do not exist until you load `agentroom-developer.md`. **Read it first.**
