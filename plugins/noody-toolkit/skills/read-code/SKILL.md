---
name: read-code
description: Use when the user asks to read, research, or explain how a named feature works in this repo (e.g. "read code about X", "how does X work", "spin up your team to look into X"), before producing a synthesized explanation of that feature.
---

# Read Code

## Overview

Dispatches the built-in `Explore` subagent to answer "how does X work" questions. A head-to-head test against the old `code-researcher`/`code-reader` fan-out found `Explore` faster, cheaper, and more reliable for this job: a single foreground `Explore` call returned a complete, accurately-cited report, while `code-researcher`'s multi-agent orchestration took much longer, cost more tokens, and in one run lost its final synthesis entirely (the background completion notification only delivered a truncated fragment). So the default is **one** `Explore` call, not a multi-agent dispatch.

## When to use

- Trigger: any request to read/research/explain how a named feature works in this repo — "read code about [feature]", "how does [feature] work", "spin up your team to look into [feature]"
- `[feature]` is supplied by the user in their request — use it verbatim

## Process

1. Extract the feature name/description from the user's request.
2. **Default: dispatch exactly one `Explore` Agent call.** `subagent_type: "Explore"`, `run_in_background: false` unless the user wants you to keep working on something else meanwhile. Give it the feature name/description verbatim, ask it to search "very thorough" (Explore's own breadth setting) since this is a full-feature explanation rather than a quick lookup, and ask for file:line citations.
3. **Escalate to multiple parallel `Explore` calls only when** the user explicitly asks for it (e.g. "spin up a team", "be thorough", "multiple angles", "cross-check", "second opinion") or the feature is clearly large/cross-cutting enough (spans several unrelated packages/games) that one agent's read window would plausibly miss things:
   - Split into 3-4 independent angles that fit the feature, e.g. entry point/trigger, data/state flow, UI/presentation, config/edge cases.
   - Send one `Agent` call per angle **in a single message** so they run concurrently — `subagent_type: "Explore"`, a self-contained prompt naming the feature and that agent's specific angle (each agent has no memory of the others).
   - When all agents return, synthesize their reports into one coherent narrative end-to-end, with file:line references. Do not just concatenate the raw reports back to back.
4. In the default single-dispatch case, `Explore`'s own report is already a complete answer — pass it through (lightly edited for the conversation) rather than re-synthesizing it from scratch.

## Quick reference

Angles to use only when escalating to a multi-agent fan-out:

| Angle example | What it covers |
|---|---|
| Entry point / trigger | What calls into the feature, the event/state that activates it |
| Data / state flow | Where data lives, how it moves through the system |
| UI / presentation | GUI screens, prefabs, handler classes involved |
| Config / edge cases | Remote config, feature flags, error/edge-case paths |

## Common mistakes

| Mistake | Fix |
|---|---|
| Fanning out to 3-4 `Explore` calls by default | Default to one call — escalate only for explicit asks or genuinely cross-cutting features |
| Re-synthesizing a single `Explore` call's own report | Pass its report through — it's already a complete answer |
| Escalating to a multi-agent team for a small/narrow feature | Reserve escalation for an explicit ask or a genuinely cross-cutting feature |
| Dispatching escalated agents one at a time | Send all `Agent` calls in one message so they run in parallel |
| Pasting each subagent's report back to back | Synthesize into one unified explanation |
| Using `code-researcher`/`code-reader` for this skill | Deprecated for this purpose — tested slower, costlier, and lost its final output in a background run |
