---
name: firebase-event-tracking
description: Use when adding, changing, or auditing a Firebase/GA4 analytics event in this Unity rhythm engine — wiring a new song, onboarding, ad, personalization, or rating event through AnalyticHelper, AppsFlyerHelper, or AdEventTracking.IronSource into the AmaGDK analytics pipeline, or when the user mentions LogEvent, LogFunnelEvent, OnlyTo, TrackingEvent.All, or Firebase/AppsFlyer/Personalization tracking.
---

# Firebase Event Tracking

## Overview

One shared pipeline: `game code → helper (AnalyticHelper / AnalyticHelper.Personalization /
AppsFlyerHelper / AdEventTracking.IronSource) → AmaGDK.Analytics facade (LogEvent /
LogFunnelEvent) → adapters (Firebase, AppsFlyer, Personalization, ironSource) → vendor SDKs`.

**[`docs/Game-Event-Tracking-Mapping.md`](../../../docs/Game-Event-Tracking-Mapping.md) is the
exhaustive, authoritative map** of every existing event, funnel, and destination (§5 flows, §7
taxonomy, §8 params, §9 destinations matrix). Read the relevant section there before adding an
event — this skill is the "how to add one correctly" checklist, not a replacement for it.

## Where to add a new event

| Event is about… | File |
|---|---|
| Song lifecycle, `FN_*` onboarding funnel, survey, rating, user properties | `Assets/Piano7/ModuleFirebase/Events/AnalyticHelper.cs` |
| Tutorial calibration / difficulty assignment / `song_result_stats` | `AnalyticHelper.Personalization.cs` |
| Ad lifecycle (Firebase `fullads_*` / `videoads_*`, impressions, revenue) | `Assets/AmaGDK/Adapters/IronSource/AdEventTracking.IronSource.cs` |
| AppsFlyer-only mirror of an ad or song event | `Assets/Piano7/ModuleFirebase/Events/AppsFlyerHelper.cs` |

## Pick the right call

- **`LogEvent(name, params)`** — normal event, fires every time the state occurs (`song_start`,
  `song_fail`, `rating_popup_show`…).
- **`LogFunnelEvent(name)`** — fires once per install, deduped by name (funnel-signature,
  persisted to disk, survives restarts). Use only for an ordered onboarding/first-open step
  (`FN_*` family). No custom params beyond `accumulated_count`.

## Routing modifiers — get these wrong and the event silently doesn't reach where you think

| Modifier | Effect |
|---|---|
| *(none)* | Reaches **Firebase only** — it's the only adapter with `SendEventByDefault=true`. |
| `.OnlyTo(adapterIds)` | Explicit allow-list; the only way to also reach AppsFlyer or Personalization. `defaultModules = [FIREBASE_ANALYTICS, "PersonalizationAnalytics"]` is the pair `AnalyticHelper` uses by default. |
| `.SetAsAccumulated()` | Attaches lifetime `accumulated_count`; pair with `IncrementAccumulatedCount`. |
| `.LogImmediately()` | Bypasses the batch queue. **Required** (throws in editor otherwise) if the event name is in the `immediatelyLoggedEvents` allowlist in `AmaGDKConfig.asset` — used today for impression/revenue events. |

Adapter IDs are `const string`s on `Amanotes.Core.AmaGDK.AdapterID` (imported via
`using static Amanotes.Core.AmaGDK.AdapterID;` in `AnalyticHelper.cs`) — `FIREBASE_ANALYTICS`,
`APPSFLYER`, `IRONSOURCE`, `MAX`, `REVENUECAT`, `SQLITE_ANALYTICS`. **Personalization has no
constant** — the codebase passes the raw literal `"PersonalizationAnalytics"` instead.

**To reach both Firebase and AppsFlyer, don't rely on one multi-ID `.OnlyTo()` call — follow the
established two-call pattern** (see `LogSongStart` + `AppsFlyerHelper.OnSongStart`):
1. `AnalyticHelper` fires the Firebase-routed event: `AmaGDK.Analytics.LogEvent(name, params).OnlyTo(defaultModules)...`.
2. Separately call (or add) a matching method in `AppsFlyerHelper`, whose private `Fire(name, data)`
   always routes `.OnlyTo(APPSFLYER)` internally and auto-attaches `connection`. AppsFlyerHelper
   methods are a parallel, independent call — not a modifier on the first call.

**Personalization delivery is conditional even with `.OnlyTo(defaultModules)`.**
`PersonalizationAnalytisAdapter.LogEvent` silently drops any event name not present in
`Amanotes.Tracking.TrackingEvent.All` — an 11-name whitelist shipped in the
`com.amanotes.personalization` package (not this repo): `song_start, song_click, song_unlock,
song_ap, me_start, song_fail, song_revive_impression, song_revive, song_end, song_result,
tutorial_calibration`. Routing a new event through `defaultModules` does **not** make it reach
Personalization unless that whitelist is also updated (data-team owned) — mark it `PA*`
(conditional) in the mapping doc, never assume delivery.

## Reuse the param builders

Song-lifecycle events should reuse `GenerateEngagementEventParams` (and
`GeneratePlayerDeathParams` for death-adjacent events) instead of hand-building a dict — this
keeps `song_session_id`, `song_id`, `detailed_song_location`, etc. consistent across the funnel.
Hand-build a param dict only for one-off events (rating, onboarding, ads) that don't share the
engagement shape.

Known placeholders: `difficulty` and `song_mode` inside engagement params are hardcoded
(`'medium'` / `'normal'`) — don't rely on them for real difficulty; use the Personalization-side
`song_result_stats` / `difficulty_*` events for the resolved value.

## Hard platform limits (silent — nothing throws, data just gets dropped or mangled)

- Event name: ≤40 chars; non-alphanumeric → `_`; must start with a letter or gets prefixed `E`;
  reserved prefixes rewritten (`firebase_→frb_`, `google_→ggl_`, `ga_→gga_`).
- ≤25 params per event — the 26th is dropped (with a warning).
- String param values truncate at 100 chars.
- 500 distinct event-type cap process-wide.

## After adding the event

1. Add a row to the matching §7 taxonomy table in `docs/Game-Event-Tracking-Mapping.md` (event
   name, API method, key params, trigger file:line, destinations).
2. Update the §9 Destinations Matrix if it introduces a new category.
3. If you had to guess at AppsFlyer/Personalization delivery, or hit undocumented adapter
   behavior, log it under §10 (Open Questions & Drift) instead of silently assuming — that
   section exists specifically so the doc doesn't quietly go stale.

## Common mistakes

| Mistake | Fix |
|---|---|
| Assume AppsFlyer gets an event because it "looks important" | Add a second, separate call through `AppsFlyerHelper` (its `Fire()` always routes `.OnlyTo(APPSFLYER)`) — it is not covered by the Firebase call's `.OnlyTo(...)` list |
| Assume `.OnlyTo(defaultModules)` means Personalization receives it | Check it's in `TrackingEvent.All` first; if not, it's Firebase-only regardless of routing |
| Add an impression/revenue event without `.LogImmediately()` | Check `AmaGDKConfig.asset`'s `immediatelyLoggedEvents`; add the name there and call `.LogImmediately()` |
| Hand-roll song params instead of reusing `GenerateEngagementEventParams` | Reuse the builder so the funnel stays consistent |
| Forget to update the mapping doc | Add the row in §7 + §9 immediately — it's the team's single source of truth and drifts fast otherwise |
