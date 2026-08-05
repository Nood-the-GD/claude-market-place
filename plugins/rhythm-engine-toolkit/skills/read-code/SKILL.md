---
name: read-code
description: Use when the user asks to read, research, or explain how a named feature works in this repo (e.g. "read code about X", "how does X work", "spin up your team to look into X"), before producing a synthesized explanation of that feature.
---

# Read Code

## Overview

Dispatches the `code-researcher` subagent to answer "how does X work" questions. `code-researcher` already reads cheaply and in parallel on its own (it fans out to `code-reader`, a Haiku worker, for the actual file reading) and does exactly one synthesis pass itself — so the cost-efficient default is **one** `code-researcher` call, not several. Only escalate to multiple parallel `code-researcher` dispatches when the user explicitly wants extra depth or cross-checking: each additional researcher call re-pays a full synthesis pass and its angles often re-read overlapping files.

## When to use

- Trigger: any request to read/research/explain how a named feature works in this repo — "read code about [feature]", "how does [feature] work", "spin up your team to look into [feature]"
- `[feature]` is supplied by the user in their request — use it verbatim

## Process

1. Extract the feature name/description from the user's request.
2. **Default: dispatch exactly one `code-researcher` Agent call.** Give it the feature name/description verbatim and let it scope, dispatch its own `code-reader` reads, and synthesize on its own. `run_in_background: false` unless the user wants you to keep working on something else meanwhile.
3. **Escalate to multiple parallel `code-researcher` calls only when** the user explicitly asks for it (e.g. "spin up a team", "be thorough", "multiple angles", "cross-check", "second opinion") or the feature is clearly large/cross-cutting enough (spans several unrelated packages/games) that one pass would be shallow:
   - Split into 3-4 independent angles that fit the feature, e.g. entry point/trigger, data/state flow, UI/presentation, config/edge cases.
   - Send one `Agent` call per angle **in a single message** so they run concurrently — `subagent_type: "code-researcher"`, a self-contained prompt naming the feature and that agent's specific angle (each agent has no memory of the others).
   - When all agents return, synthesize their reports into one coherent narrative end-to-end, with file:line references. Do not just concatenate the raw reports back to back.
4. In the default single-dispatch case, `code-researcher`'s own report is already the final synthesis — pass it through (lightly edited for the conversation) rather than re-synthesizing it again.

## Quick reference

Angles to use only when escalating to a multi-researcher fan-out:

| Angle example | What it covers |
|---|---|
| Entry point / trigger | What calls into the feature, the event/state that activates it |
| Data / state flow | Where data lives, how it moves through the system |
| UI / presentation | GUI screens, prefabs, handler classes involved |
| Config / edge cases | Remote config, feature flags, error/edge-case paths |

## Common mistakes

| Mistake | Fix |
|---|---|
| Fanning out to 3-4 `code-researcher` calls by default | Default to one call — it already parallelizes its own reading cheaply via `code-reader` |
| Re-synthesizing a single `code-researcher`'s own report | Pass its report through — it's already the final synthesis |
| Escalating to a multi-researcher team for a small/narrow feature | Reserve escalation for an explicit ask or a genuinely cross-cutting feature |
| Overriding `model`/effort on the `code-researcher` call | Leave it unset — the agent is already pinned (Sonnet, medium effort) |
| Dispatching escalated agents one at a time | Send all `Agent` calls in one message so they run in parallel |
| Pasting each subagent's report back to back | Synthesize into one unified explanation |
