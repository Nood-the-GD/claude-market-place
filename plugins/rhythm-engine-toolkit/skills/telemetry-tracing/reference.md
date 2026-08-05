# Telemetry Tracing — Reference

All types live in `MagicTiles.ModulePerformance` under
`Assets/Piano7/ModulePerformance/Tracing/`.

## File map

| File | Role |
|------|------|
| `PerfTrace.cs` | Core: start transactions/child spans, factory `TraceAsync`/`TraceChildAsync`, null-safe finish semantics. |
| `PerfTraceExtensions.cs` | `UniTask` extension methods to measure an already-started task in one line. |
| `*TraceNames.cs` | Centralized `const string` span/operation names. **Add new names here.** One class per subsystem/domain. |
| `../HoneycombPerformanceTracker.cs` | ACM `IPerformanceTracker` → OpenTelemetry OTLP exporter. Wires `ISpan`. |
| `../PerformanceTrackingBootstrap.cs` | Adds global default attributes (`app.produce_code`, `user.id`) to all spans. |

## API surface

### PerfTrace (static)

```csharp
// Start spans (null-safe; return null when no tracker / on failure)
ISpan StartTransaction(string name, string operation = "task");
ISpan StartChild(ISpan parent, string operation);   // null parent → null

// Factory overloads — body receives the span (may be null) for mid-flight SetData
UniTask<T> TraceAsync<T>(string name, Func<ISpan, UniTask<T>> body, string operation = "task");
UniTask    TraceAsync   (string name, Func<ISpan, UniTask>   body, string operation = "task");
UniTask<T> TraceChildAsync<T>(ISpan parent, string operation, Func<ISpan, UniTask<T>> body);
```

### PerfTraceExtensions (UniTask extension methods)

```csharp
UniTask<T> task.TraceAsync(string name, string operation = "task");        // new root
UniTask    task.TraceAsync(string name, string operation = "task");
UniTask<T> task.TraceChildAsync(ISpan parent, string operation);           // child; null parent → unmeasured
UniTask    task.TraceChildAsync(ISpan parent, string operation);
```

### ISpan (Amanotes.ACM)

```csharp
void  SetData(string key, object value);   // attribute; value stringified by the Honeycomb adapter
ISpan StartChild(string operation);
void  Finish();                            // status OK
void  Finish(Exception exception);         // status Error + exception.type/message/stacktrace attributes
```

## Finish semantics (guaranteed by PerfTrace factory + extension helpers)

| Outcome | Effect |
|---------|--------|
| success | `span.Finish()` |
| `OperationCanceledException` | `span.SetData("cancelled", true)` + `span.Finish(ex)`, rethrown |
| other exception | `span.Finish(ex)`, rethrown unchanged |
| `span == null` | body runs unmeasured |

## Complete templates

### Pattern 1 — Factory (root)

```csharp
using MagicTiles.ModulePerformance;

await PerfTrace.TraceAsync(MyTraceNames.WidgetLoad, async span =>
{
    span?.SetData("widget_id", id);
    var bytes = await LoadAsync();
    span?.SetData("bytes", bytes.Length);
});
```

### Pattern 1 — Factory (child, with nested children)

```csharp
return PerfTrace.TraceAsync(MyTraceNames.SdkInit, span =>
{
    var a = InitAAsync().TraceChildAsync(span, MyTraceNames.SdkInitA);
    var b = InitBAsync().TraceChildAsync(span, MyTraceNames.SdkInitB);
    return UniTask.WhenAll(a, b).AttachExternalCancellation(ct);
});
```

### Pattern 2 — Extension (measure existing task)

```csharp
await LoadAssetsAsync(ct)
    .TraceChildAsync(parentSpan, MyTraceNames.AssetLoad);
```

### Pattern 3 — Manual span (non-linear code)

```csharp
var span = PerfTrace.StartChild(parentSpan, MyTraceNames.ThingInit);
Exception error = null;
try
{
    span?.SetData("mode", mode);
    // ... work that isn't a single awaitable ...
}
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

## New-subsystem template

1. Centralize names — new domain gets its own class in `Tracing/`:

```csharp
namespace MagicTiles.ModulePerformance
{
    /// <summary>Centralized Honeycomb span/operation names for the Gameplay subsystem.</summary>
    public static class GameplayTraceNames
    {
        public const string SongLoad      = "gameplay.song_load";
        public const string AssetLoad     = "gameplay.asset_load";
        public const string SceneActivate = "gameplay.scene_activate";
    }
}
```

2. Instrument, childing off the owning span passed down the call chain:

```csharp
public UniTask LoadAsync(ISpan parent, CancellationToken ct)
{
    return PerfTrace.TraceChildAsync(parent, GameplayTraceNames.SongLoad, async span =>
    {
        await LoadAssetsAsync(ct).TraceChildAsync(span, GameplayTraceNames.AssetLoad);
        await ActivateSceneAsync(ct).TraceChildAsync(span, GameplayTraceNames.SceneActivate);
        span?.SetData("song_id", songId);
    });
}
```

3. If the subsystem owns its own top-level flow, start a root:

```csharp
var root = PerfTrace.StartTransaction(GameplayTraceNames.SongLoad, "gameplay");
// pass `root` to children; call root.Finish() / root.Finish(ex) when the flow ends
```

## Global attributes

`PerformanceTrackingBootstrap.Apply()` sets `app.produce_code` and `user.id` (backfilled via `AmaGDK.SetCallback_OnUserIdUpdated`) as defaults on every span — do not re-add per span. Add only subsystem-specific dimensions via `SetData`.

## Backend

- Dataset: `acm-sdk`; endpoint: `https://refinery.amanotes.net/v1/traces` (OTLP HTTP protobuf, batch export).
- Configured in `HoneycombPerformanceTracker` (`ApiKey`/`Dataset`/`HoneycombEndpoint`).
- WebGL: tracker is not installed; all helpers degrade to no-ops.
