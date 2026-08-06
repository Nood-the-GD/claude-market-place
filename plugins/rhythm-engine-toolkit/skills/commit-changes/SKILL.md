---
name: commit-changes
description: Use when asked to commit pending changes in this Unity repo — splits a dirty working tree into clean, logically-grouped commits, keeping auto-generated Editor noise and unrelated asset drift out of C#/prefab/scene commits.
model: claude-sonnet-5
effort: medium
---

# Committing Code Cleanly — Rhythm Engine

The working tree in this project routinely mixes real source changes with Unity
Editor noise: reimport touches, ad-SDK version bumps, local dev toggles,
Play-mode save-state. Never `git add -A` / `git add .` here — always stage
explicit paths per commit.

## What belongs in a commit

- **C# files**: always in scope — they're the source of truth for intent.
- **`.prefab` / `.unity` (scenes)**: include if EITHER —
  (a) `git diff -- <path>` shows a change that traces to a modified C# file
  in the *same* commit — a component added/removed, a serialized field
  newly wired to a script member, a GUID/string that matches a renamed C#
  constant; OR
  (b) the diff is a coherent, deliberate-looking change on its own — an
  anchor/pivot moved, a component enabled/disabled, a color/size/text
  value clearly set on purpose — even with no C# link at all.
  Either way, still leave out a diff that's pure transform-number drift,
  GUID reordering, or fileID/modification-list churn with no discernible
  intent — that's noise regardless of which criterion got it this far.
- **`.asset` / other assets**: only if the diff content is a direct,
  intentional consequence of the C# change (e.g. a database asset's entry
  renamed to match a renamed constant in the paired C# diff). Default to
  excluding.

## What to leave out by default

Recurring noise patterns in this repo's working tree — these categories are
excluded on principle, even when the diff itself looks deliberate (a version
bump is still a version bump). The "coherent, deliberate-looking diff" leniency
above is for `.prefab`/`.unity` content changes, not for these:

- Runtime/save-state ScriptableObjects: `ModuleUserData/**/*.asset`
  (`UserData*.asset`) — Play-mode test artifacts, not source.
- Ad-SDK / mediation config: `LevelPlay/**`, `GDKSymbolDefine.asset` —
  version/symbol bumps unrelated to feature work.
- Addressables build output: `AddressableAssetsData/link.xml(.meta)`.
- Editor tool state: `vInspector Data.asset`, meta-only diffs on unrelated
  art (materials, sprite sheets).
- Local scene/remote-config toggles no C# in the diff reads (a lone
  `_forceLocalGameDesignConfig` flip in a `.unity` scene, an unreferenced
  `FirebaseConfig.asset` field).

## Process

1. `git status --porcelain=v1 -uno` and `git diff --stat` — never `-uall`.
2. Classify every path with the rules above. For each `.prefab` / `.unity` /
   `.asset` on the fence, run `git diff -- <path>` and check two things,
   don't guess from the filename or directory alone: (a) does the
   field/ID/component it touches also appear in a modified C# file's diff,
   and (b) failing that, does the diff itself read as a deliberate value
   change (anchor/pivot moved, component toggled, a value clearly set on
   purpose) rather than transform-number drift or fileID/GUID reordering.
   Either (a) or (b) is enough to include it.
3. Group the included files into logical commits: one concern per commit,
   in dependency order (a shared API/schema change lands before the callers
   that use it) so each commit compiles independently. Splitting further
   because two features got mixed together is expected, not a failure to
   avoid.
4. Anything genuinely ambiguous — a value changed but no clear C# link
   found — don't silently include or exclude it. List it separately and ask
   the user before deciding.
5. Stage each commit's files by explicit path, write a message matching
   this repo's convention (see `git log --oneline`: `type(scope): imperative
   summary`, e.g. `feat(search):`, `fix(search):`, `chore(search):`), then
   commit.
6. Report what was committed in each commit and what was left out (and
   why), so the user can review the exclusions.

## Common mistakes

| Mistake | Fix |
|---|---|
| `git add -A` / `git add .` | Stage explicit paths per commit |
| Committing a prefab on a guess, without reading its diff | Verify the diff either traces to a real C# change or is itself a coherent, deliberate value change |
| Excluding a prefab-only diff just because no C# file touches it | Check whether the diff reads as deliberate (anchor/pivot moved, component toggled, a value set on purpose) before calling it noise |
| Treating a version bump / save-state asset as includable because it "looks intentional" | The categorical exclusions (ad-SDK, Play-mode save-state, Addressables output, editor tool state) apply regardless of how deliberate the diff looks |
| Bundling unrelated features into one commit to avoid splitting work | Split — one concern per commit is the point |
| Silently dropping an ambiguous asset change | Surface it to the user instead of guessing |
