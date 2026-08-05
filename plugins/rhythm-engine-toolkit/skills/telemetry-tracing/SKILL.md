---
name: telemetry-tracing
description: Add or modify Honeycomb performance/error telemetry (spans, transactions) in the Piano7/MT3 rhythm engine. Use when instrumenting a subsystem, measuring async work, observing latency or failures, or when the user mentions PerfTrace, spans, transactions, Honeycomb, or telemetry.
---

# Telemetry Tracing

Instrument code with Honeycomb spans to observe performance and errors. The stack:

```
Your code
  → PerfTrace / PerfTraceExtensions               (Assets/Piano7/ModulePerformance/Tracing)
  → ACM ISpan / IPerformanceTracker               (Amanotes.ACM)
  → HoneycombPerformanceTracker (OpenTelemetry → OTLP)
  → Honeycomb (dataset "acm-sdk", refinery.amanotes.net)
```

Namespace for all tracing types: `MagicTiles.ModulePerformance`. WebGL is a no-op (OTLP HTTP exporter unsupported), so tracing must never be required for correctness.

## Golden rules

1. **Telemetry never breaks gameplay.** Span creation/finish failures are swallowed with a single warning; the measured work still runs. Never let a span throw into the game path. All `PerfTrace` helpers already guarantee this — preserve it.
2. **Centralize every trace/span name.** No inline string literals for span or operation names in business code. Put them as `public const string` in a `*TraceNames` class in `Assets/Piano7/ModulePerformance/Tracing/`. This is the single source of truth and prevents typo-fragmented traces in Honeycomb.
3. **Null spans are valid everywhere.** `StartTransaction`/`StartChild` return `null` when no tracker exists (WebGL, init failure). Every API accepts `null` and simply skips measurement — do not null-guard defensively at call sites unless you `SetData` on a span you hold (use `span?.SetData(...)`).
4. **Span start must match work start.** Start the span immediately before/as the work begins so duration reflects real work, not queueing.
5. **One `*TraceNames` class per subsystem/domain.** As subsystems grow, add a new centralized names class (e.g. `GameplayTraceNames`) rather than overloading one giant file — but keep them all in the `Tracing/` folder.

## The three usage patterns

Pick based on the code shape. See `reference.md` for full templates and the complete API.

### 1. Factory (body receives the span) — preferred for new roots/children with mid-flight `SetData`

```csharp
await PerfTrace.TraceAsync(MyTraceNames.WidgetLoad, async span =>
{
    span?.SetData("widget_id", id);
    await DoWorkAsync();
});
```

`TraceChildAsync(parent, op, body)` does the same under a parent span.

### 2. Extension (measure an already-started task) — cleanest one-liner

```csharp
await LoadAssetsAsync(ct)
    .TraceAsync(MyTraceNames.AssetLoad);

await FetchAsync(ct)
    .TraceChildAsync(parentSpan, MyTraceNames.Fetch);
```

Call the extension **immediately** after creating the task so the span start matches the work start. Note: `.Timeout()` on UniTask abandons (does not cancel) the underlying work — the span then measures wait time, not work time. Prefer passing a real `CancellationToken` to the work.

### 3. Manual span (try/catch/finally) — for non-linear code where a factory lambda is awkward

```csharp
var span = PerfTrace.StartChild(parentSpan, MyTraceNames.ThingInit);
Exception error = null;
try { /* work */ }
catch (Exception ex) { error = ex; throw; }
finally
{
    if (span != null)
    {
        if (error == null) span.Finish();
        else span.Finish(error);
    }
}
```

## Choosing a parent span

- **Independent flow** → start a root with `PerfTrace.StartTransaction(name, operation)` (or the `TraceAsync` factory/extension, which start a root for you).
- **Part of a larger operation** → child off the owning span. Pass the parent `ISpan` down through your call chain and use `TraceChildAsync` / `PerfTrace.StartChild(parent, op)`.
- Keep the tree shallow and meaningful: one span per unit of work you'd actually want to chart or alert on.

## Finish semantics (guaranteed by the factory/extension helpers)

- success → `span.Finish()`
- `OperationCanceledException` → `span.SetData("cancelled", true)` + `span.Finish(ex)`, rethrown
- other failure → `span.Finish(ex)`, rethrown unchanged
- `span == null` → body runs unmeasured

Errors reach Honeycomb only if the exception actually flows through the helper — so **always rethrow** from your own try/catch.

## Adding telemetry — checklist

```
- [ ] Add const names to the relevant *TraceNames class (create one if the subsystem is a new domain)
- [ ] Decide the parent span: a new root, or child off an owning span passed down the call chain
- [ ] Pick a pattern: extension (one-liner task), factory (needs SetData mid-work), or manual (non-linear code)
- [ ] Attach useful dimensions with span?.SetData(key, value) — ids, sizes, counts, mode, attempt index
- [ ] Ensure failures rethrow (factory/manual both do) so errors reach Honeycomb via Finish(ex)
- [ ] Confirm null-safety: no NRE when span is null (WebGL / no tracker)
```

## Anti-patterns

- Inline string span names in business code — centralize in a `*TraceNames` class instead.
- Wrapping a span in a `try` that swallows the exception — the error then never reaches `Finish(ex)`; always rethrow.
- Over-instrumenting: a span per trivial call floods the trace. Measure work worth charting/alerting on.
- Touching Unity APIs (`Time.*`, `Application.*`) inside span logic — spans may finish off the main thread; keep them Unity-free.
- Relying on a span being non-null for control flow.
- Re-adding global attributes (`app.produce_code`, `user.id`) per span — they're set as defaults (see `reference.md`).

## Additional resources

- `reference.md` — full API surface, complete templates for all three patterns, and the new-subsystem template.
