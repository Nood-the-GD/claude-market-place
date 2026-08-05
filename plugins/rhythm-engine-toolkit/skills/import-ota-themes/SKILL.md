---
name: import-ota-themes
description: Runbook for importing new OTA themes from the piano_mt3 project into rhythm-engine via the ThemePianoImporter editor tooling. Use this whenever the user wants to import, port, or add MT3 OTA themes (e.g. "import theme X qua rhythm-engine", "port these piano_mt3 themes", batch names like HAZBIN/ICONIC/SPONGE_BOB), mentions the "Theme OTA" menu, ThemePianoImporter, NewMt3Names, or reports a theme showing wrong/phonk-sonic art on its long tiles. Covers the full add-names → run menu → handle the domain-reload disconnect → validate flow. It does NOT cover shipping (Addressables build, CDN upload, APK rebuild) — those stay manual.
---

# Import OTA Themes (piano_mt3 → rhythm-engine)

All the import logic already lives in one editor file plus a handful of menus. Your job is to drive that tooling correctly and read the results out of the Unity console — **not** to hand-build ThemeConfigs, prefabs, or sprites. This is a runbook, not a construction project.

**Boundary:** this gets themes into the project as `ThemeConfig` + Addressable entry + Firebase manifest row, validated. Shipping them (build Addressables, upload to CDN, rebuild the APK) is a separate manual process and is out of scope here — say so and stop when validation passes.

## Key file & menus

- Importer: `Assets/MT3/ModuleTheme/Ota/Editor/ThemePianoImporter.cs`
- `Tools/Theme OTA/Import Piano Themes - NEW (piano_mt3)` — imports every name in `NewMt3Names`
- `Tools/Theme OTA/Validate Themes For Publish` — validates all ThemeConfigs
- `Tools/Theme OTA/Fix Long-Note Body - NEW (piano_mt3)` — one-shot, only for themes imported before the long-note fix landed

## Prerequisites

- The **piano_mt3** repo checked out locally for the source art. There are two repos and it matters which one a theme ships in:
  - `PianoTexMt3` → `~/Desktop/piano_mt3/...` (newer themes usually ship here)
  - `PianoTex` → `~/Desktop/piano_mt3_android/...` (older/android batch)
  The NEW menu reads from `PianoTexMt3`. If a theme only exists in the android repo, it belongs to the old path, not this flow.
- Unity open with the MCP bridge. The server is named **`UnityMCP`** (capital U); the lowercase `unityMCP` alias has no connected instance — using it wastes a round-trip.

## Runbook

1. **Add the theme names.** In `ThemePianoImporter.cs`, append the folder names to `NewMt3Names` (UPPER_CASE, matching the folders under the source `TextureAssets` dir). Example:
   ```csharp
   private static readonly string[] NewMt3Names =
   {
       "HAZBIN","ICONIC","TFOL","YARARARA","SPONGE_BOB","MINECRAFT","SAKANACTION",
       // append the new batch here
   };
   ```

2. **Pin the Unity instance.** Read `mcpforunity://instances`, then `set_active_instance` to `rhythm-engine@<hash>`. The server errors if several editors are connected and none is pinned.

3. **Confirm a clean compile.** After your edit compiles, `read_console` for errors. Only proceed at 0 errors — the menu won't exist until the script compiles.

4. **Run the import** via `execute_menu_item` → `Tools/Theme OTA/Import Piano Themes - NEW (piano_mt3)`.
   **Expect the MCP call to fail with `disconnected while awaiting command_result` / hint `retry`.** That is normal: the import does a full `AssetDatabase.Refresh()` + domain reload, which drops the bridge mid-call. **Do not re-run the menu** — a second run would double-process. The import is still running; just wait and check the console.

5. **Read the outcome from the console, not the tool return.** `read_console` with `filter_text: "PianoImport"`. Success line: `[PianoImport] Done — N OK, M failed.` Each theme logs its own `OK '<id>' -> …` or a failure reason.

6. **Validate.** `execute_menu_item` → `Tools/Theme OTA/Validate Themes For Publish`, then `read_console` with `filter_text: "Validate"`. Success line: `[ThemeOTA validate] Done — X OK, 0 FAILED`. A `FAIL` names the theme and why (missing scripts / editor-only components) — fix that before shipping.

7. **Long-note body — usually nothing to do.** The importer now auto-creates `{id}_long_note.png` (a bottom-pivot duplicate of the short note) and swaps it onto the long tile, so long tiles no longer leak `tilelong_phonk_sonic` art. Only run `Tools/Theme OTA/Fix Long-Note Body - NEW (piano_mt3)` if a theme was imported *before* that fix landed and its long tiles show phonk-sonic art. See "Long-tile art leak" below for the diagnosis.

## Done looks like

- `[PianoImport] Done — N OK, 0 failed.` and `[ThemeOTA validate] Done — X OK, 0 FAILED.`
- New `Assets/MT3/Themes/<NAME>/` folders, each with a `ThemeConfig`, `Art/`, `Tile-Short`/`Tile-Long` prefabs, and 4 `BG` prefabs.
- `Assets/Piano7/ModuleFirebase/FirebaseConfig.asset` shows as modified (the manifest got the new rows).
- The tooling commits nothing — committing, plus the manual ship steps, are on the user.

## Gotchas

- **Verify via the console, never via the menu tool's return.** Any menu that touches assets triggers a domain reload that disconnects the bridge; the tool return being an error does not mean the operation failed.
- **Don't chain `sleep` in Bash** to "wait for the refresh" — it's blocked. Poll `read_console` instead.
- **A theme needs 1 short note + 1 hold + ≥4 backgrounds.** Short-note filenames vary (e.g. YARARARA's is `yararara_bg_short_note.png`); the classifier handles it, but if import errors with `need a short note + >=4 backgrounds`, inspect the source PNG names.
- **Re-running the NEW import skips themes whose ThemeConfig already exists** — it won't reprocess them. To redo one, delete its `Assets/MT3/Themes/<NAME>/` folder first, or use a targeted fix menu.

## Long-tile art leak (why the fix exists)

The long-tile template (`Tile-Long-phonk-sonic`) keeps its body sprite on a renderer whose `TileShape.bgRenderer` is **unassigned**, so `BuildTile`'s face swap misses it — the phonk body sprite would otherwise survive into every imported long tile (short tiles are fine; their `bgRenderer` is wired). To diagnose a suspected leak, grep the theme's `Tile-Long-<id>.prefab` for the phonk long-body GUID `ab33f9bf363e74aa7970c9afcacaec07` (`tilelong_phonk_sonic.png`): a hit means it's leaking. The importer's `MakeLongNoteSprite` + `SwapLongBody` fix this at import time; the Fix menu backfills already-imported themes.
