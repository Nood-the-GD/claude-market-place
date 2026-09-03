---
name: read-code
description: Use when the user asks to read, research, or explain how a named feature works in this repo (e.g. "read code about X", "how does X work", "spin up your team to look into X"), before producing a synthesized explanation of that feature.
---

# Read Code

## Overview

Answers "how does X work" questions with a two-tier split that balances cost against quality:

- **Reading tier — cheap.** All file locating and extraction runs on `Explore` subagents pinned to `haiku` via the `model` override. This is the token-heavy part (sweeping many files) and haiku handles "find the files, pull the signatures and control flow, cite file:line" well enough.
- **Conclusion tier — your main model, inline.** You (the calling session) own the synthesis: turn the haiku report(s) into one coherent end-to-end explanation. This is cheap because only citations and notes are in context, never raw file bodies — and it's where reasoning quality actually matters.

Do **not** run the synthesis as its own subagent: a head-to-head test found multi-agent orchestration (`code-researcher`/`code-reader`) slower, costlier, and unreliable — in one run it lost its final synthesis entirely (the background completion notification only delivered a truncated fragment).

## When to use

- Trigger: any request to read/research/explain how a named feature works in this repo — "read code about [feature]", "how does [feature] work", "spin up your team to look into [feature]"
- `[feature]` is supplied by the user in their request — use it verbatim

## Process

1. Extract the feature name/description from the user's request.
2. **Default (narrow feature — one system, a handful of files): dispatch exactly one `Explore` Agent call, pinned to `haiku`.** `subagent_type: "Explore"`, `model: "haiku"`, `run_in_background: false` unless the user wants you to keep working on something else meanwhile. Give it the feature name/description verbatim, ask it to search "very thorough" (Explore's own breadth setting — haiku needs this push more than a larger model would), and ask for a factual report: file:line citations plus terse notes on entry points, control flow, data/state — not a polished prose explanation.
3. **Escalate to 3 parallel `Explore` calls (still each pinned to `haiku`) when** the user explicitly asks for it (e.g. "spin up a team", "be thorough", "multiple angles", "cross-check", "second opinion") **or** the feature is clearly cross-cutting (spans several unrelated packages/games) so one agent's read window would plausibly miss things. Parallel haiku calls recover the thoroughness a single haiku pass loses, at lower cost than one large-model pass.
   - Split into 3 independent angles that fit the feature, e.g. entry point/trigger, data/state flow, UI/presentation (or config/edge cases).
   - Send one `Agent` call per angle **in a single message** so they run concurrently — `subagent_type: "Explore"`, `model: "haiku"`, a self-contained prompt naming the feature and that agent's specific angle (each agent has no memory of the others).
4. **Synthesize yourself (main model), inline.** Connect the reported pieces into one coherent narrative end-to-end — trigger, data flow, state changes, edge cases — with file:line references throughout. Do not just concatenate the raw reports.
5. **Guardrail for haiku quality:** if a load-bearing claim in a report looks thin or the citations don't add up, spot-read 1–2 of the cited files yourself rather than re-dispatching a whole new agent.

Fallback: if `Explore` does not honor the `model` override in this harness, use `subagent_type: "general-purpose", model: "haiku"` with read-only instructions instead.

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
| Running the reading tier on the main/large model | Pin every `Explore` call to `model: "haiku"` — reading is the cheap tier |
| Running synthesis as a subagent | Synthesize inline on your main model — subagent synthesis tested slower, costlier, and lost output |
| Passing a haiku report straight through as the answer | The haiku report is raw findings; you write the coherent explanation from it |
| Fanning out to 3 `Explore` calls by default | Default to one — escalate only for explicit asks or genuinely cross-cutting features |
| Dispatching escalated agents one at a time | Send all `Agent` calls in one message so they run in parallel |
| Re-dispatching an agent over one shaky claim | Spot-read the cited file(s) yourself instead |
| Using `code-researcher`/`code-reader` for this skill | Deprecated for this purpose — tested slower, costlier, and lost its final output in a background run |
