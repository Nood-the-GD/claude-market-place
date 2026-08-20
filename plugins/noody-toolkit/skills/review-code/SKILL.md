---
name: review-code
description: Review C# code changes in the Rhythm Engine repository against engine architecture constraints, async patterns, and game mode conventions.
---

# Code Review Guide — Rhythm Engine

Use this skill when auditing, refactoring, or reviewing code changes in the Rhythm Engine project (both Magic Tiles 3 and Piano 7 modules).

---

## 1. Syntax & Core C# Rules

- **UniTask for Async Operations**: 
  - **Rule**: You MUST use `UniTask` or `UniTaskVoid` (from `Cysharp.Threading.Tasks`) for all asynchronous methods.
  - **Violation**: Standard C# `System.Threading.Tasks.Task` or `async void` is strictly prohibited.
- **JSON Serialization**: 
  - **Rule**: Use `Newtonsoft.Json` (JSON.NET) instead of standard Unity `JsonUtility` unless mapping raw remote configs via `JsonUtility.FromJsonOverwrite` on `FirebaseConfig`.
- **Animations**:
  - **Rule**: Use DOTween for gameplay visual animations and transitions.

---

## 2. Shared Engine Conventions

- **Assembly Definition Boundaries**:
  - **Rule**: Ensure new scripts align with their module's `.asmdef` boundaries. Avoid introducing circular dependencies.
- **RhythmContext (ScriptableObject)**:
  - **Rule**: Keep subsystems decoupled by utilizing `RhythmContext` public event Actions. Do not establish direct dependencies between distinct gameplay managers.
- **Prefab Pooling**:
  - **Rule**: Any reusable elements (tiles, hit effects, lists) must implement the `IPoolPrefab` interface and be managed via `PrefabPool` callbacks (`OnGet()`, `OnReturn()`).
- **Time Synchronization**:
  - **Rule**: Any audio-visual timing calculations or marker updates must route through the `MTCTimeMachine` instance.

---

## 3. Magic Tiles 3 (MT3) Rules

- **Custom Tile Prefabs**:
  - **Rule**: Every custom tile prefab must contain an `MT3Tile2` subclass (e.g. `TileShort`, `TileLong`) along with a `TileShape` component on the root.
- **Curved/Zigzag Trails**:
  - **Rule**: Curved or zigzag paths must utilize the `CurvyTile` package APIs to construct dynamic meshes from spline control points.

---

## 4. Piano 7 (P7) Rules

- **Modular Services**:
  - **Rule**: Any new subsystem in P7 must be structured as a service initialized asynchronously via `GameManager.DoInit()`.
- **UI & GUI Handlers**:
  - **Rule**: Page screens and panels must follow the P7 handler pattern and inherit from page controllers mapped to unique `GUIName` constants.
- **A/B Testing & Configs**:
  - **Rule**: Do not hardcode variant layout parameters. Read configuration sets via `FeatureManager.instance.designConfig` which parses `GameDesignConfigV2` from the remote settings block.
- **Content Resolution**:
  - **Rule**: Resolve content queries based on the config `useCase` using the centralized `ContentSourceResolver`.
