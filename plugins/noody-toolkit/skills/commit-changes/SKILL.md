---
name: commit-changes
description: Use when asked to commit pending changes in this Unity repo — splits a dirty working tree into clean, logically-grouped commits, keeping generated Editor noise out while defaulting to committing real source changes (C#, renames, packages, and asset changes that trace to them).
model: claude-sonnet-5
effort: medium
---

# Committing Code Cleanly

The working tree mixes real source changes with Unity Editor noise (reimport
touches, SDK version bumps, local toggles, Play-mode save-state). Never
`git add -A` / `git add .` — stage explicit paths per commit.

Classify **every** path `git status` reports by what `git diff -- <path>`
shows, not by whether this session touched it. A change from an earlier
session or another tool is exactly as eligible.

**Default: commit it.** Exclude a path only if it's generated/local-only
noise (below) or committing it would silently change logic, delete
something, or leave a referenced file missing — not as a general caution
reflex.

## Commit

- **C# files** — always.
- **Renames/moves** — same content and (for a `.meta`) same GUID. Unity
  renames often appear as *tracked delete + untracked add* under a new
  name, which `git status -uno` hides — the `??` pass in step 1 catches the
  add. A "rename" that also changed a serialized value is not pure; treat
  the value change on its merits.
- **New packages** — a new entry in `Packages/manifest.json` /
  `packages-lock.json` or a new package folder. Ship with the change that
  needs it. Removing a package: first grep the project for usages.
- **Prefab / scene / material / shader / `.asset`** — read the diff (don't
  guess from the filename), then include unless it's noise. If a path is
  included *because* it references another file (a GUID, a database entry
  pointing at a prefab), confirm that target is tracked or staged in the
  same commit — otherwise you ship a dangling reference.

## Leave out — generated / local-only noise

- Runtime/save-state ScriptableObjects: `ModuleUserData/**/*.asset`
  (`UserData*.asset`).
- Ad-SDK / mediation config: `LevelPlay/**`, `GDKSymbolDefine.asset`.
- Addressables build output: `AddressableAssetsData/link.xml(.meta)`, and
  a bare `m_currentHash` reset in `AddressableAssetSettings.asset`.
- Editor tool state: `vInspector Data.asset`.
- A lone local dev/config toggle no modified C# file reads (a
  `_forceLocalGameDesignConfig` flip in a scene, an unreferenced
  `FirebaseConfig.asset` field).
- Meta-only diffs on unrelated art (transform/GUID/fileID churn) with no
  matching C# change and no rename.

## Process

1. Run `git status --porcelain=v1 -uno` **and** `git status --porcelain=v1
   | grep '^??'` — never `-uall`. Both together are the candidate set.
2. Classify each path. `git diff` every prefab/scene/asset on the fence and
   resolve its references per the Commit rules.
3. Group into commits — one concern per commit, in dependency order so each
   compiles (shared schema before its callers). A store-door feature, a HUD
   redesign, and a package bump are three commits, not one.
4. Ambiguous after the rules (not a rename/package/noise, no clear C# link):
   don't silently include or drop it. If the user said not to ask ("just
   commit", "commit everything safe, decide yourself"), that holds for the
   whole task — decide with the closest default (noise-like → out, else →
   in) and report the call. "Decide yourself" raises the investigation you
   owe each path (diff, resolve references, confirm compile), it does not
   lower the safety bar.
5. Stage by explicit path; message `type(scope): imperative summary`
   (`feat`/`fix`/`chore`, scope from `git log --oneline`); commit.
6. Report each commit's contents and every exclusion with its reason.

## Common traps

- `-uno` hides half of a Unity rename → always also `grep '^??'`.
- Staging an asset whose GUID/path points at an untracked file → dangling
  reference; stage the target too or split differently.
- "Commit everything" read as "one big commit" → still one concern each.
- Excluding a plausible asset change for lack of a proven C# link → the bar
  is "not noise and safe", not "provably related".
