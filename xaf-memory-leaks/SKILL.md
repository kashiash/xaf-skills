---
name: xaf-memory-leaks
description: XAF Memory Leak Prevention - event handler symmetry (OnActivated/OnDeactivated/Dispose), ObjectSpace scoped disposal with using statement, batch processing large datasets, IDisposable pattern for controllers with List<IDisposable> tracker, static reference anti-patterns, CollectionSource disposal, warning signs (memory growth on navigation, duplicate event execution), diagnostic tools (dotMemory, PerfView, XAF Tracing). Use when diagnosing memory leaks, auditing controller disposal, or reviewing ObjectSpace lifetime in DevExpress XAF applications.
license: MIT
compatibility: opencode, claude-code
metadata:
  domain: xaf
  topic: memory-leaks
  versions: v24.2, v25.1
---

# XAF: Memory Leak Prevention

## Root Causes

| Cause | Symptom |
|---|---|
| Event handler not unsubscribed | Memory grows with navigation; handler fires multiple times |
| ObjectSpace not disposed | Memory grows proportional to data accessed |
| Static reference to instance | Objects never GC'd; growing memory profile |
| CollectionSource not disposed | Cached data retained after view closes |
| Controller holds undisposed resources | Finalizer queue pressure; slow GC |

---

## Event Handler Pattern — Controller (Most Common Leak)

Every `+=` in `OnActivated` needs a matching `-=` in **both** `OnDeactivated` and `Dispose`.

```csharp
public class SafeController : ViewController {
    private bool eventsSubscribed;

    protected override void OnActivated() {
        base.OnActivated();
        if (!eventsSubscribed) {
            View.SelectionChanged += View_SelectionChanged;
            View.CurrentObjectChanged += View_CurrentObjectChanged;
            View.ObjectSpace.ObjectChanged += ObjectSpace_ObjectChanged;
            eventsSubscribed = true;
        }
    }

    protected override void OnDeactivated() {
        UnsubscribeEvents();
        base.OnDeactivated();
    }

    protected override void Dispose(bool disposing) {
        if (disposing) UnsubscribeEvents(); // safety net
        base.Dispose(disposing);
    }

    private void UnsubscribeEvents() {
        if (!eventsSubscribed) return;
        if (View != null) {
            View.SelectionChanged -= View_SelectionChanged;
            View.CurrentObjectChanged -= View_CurrentObjectChanged;
        }
        if (View?.ObjectSpace != null)
            View.ObjectSpace.ObjectChanged -= ObjectSpace_ObjectChanged;
        eventsSubscribed = false;
    }
}
```

---

## Comprehensive Disposal Pattern

Use a `List<IDisposable>` + `List<Action>` tracker to manage resources safely:

```csharp
public class ResourceTrackingController : ViewController {
    private readonly List<IDisposable> _disposables = new();
    private readonly List<Action> _cleanups = new();
    private bool _disposed;

    // Call this instead of raw += to auto-track cleanup
    protected void Subscribe<T>(T source, Action<T> subscribe, Action<T> unsubscribe)
        where T : class {
        subscribe(source);
        _cleanups.Add(() => {
            try { unsubscribe(source); } catch { }
        });
    }

    protected T Track<T>(T resource) where T : IDisposable {
        _disposables.Add(resource);
        return resource;
    }

    protected override void OnActivated() {
        base.OnActivated();
        // Example:
        Subscribe(View, v => v.SelectionChanged += OnSelectionChanged,
                       v => v.SelectionChanged -= OnSelectionChanged);

        // Example: track CollectionSource
        var cs = Track(new CollectionSource(ObjectSpace, typeof(MyObject)));
    }

    protected override void Dispose(bool disposing) {
        if (!_disposed && disposing) {
            foreach (var action in _cleanups)
                try { action(); } catch { }
            foreach (var d in _disposables)
                try { d?.Dispose(); } catch { }
            _cleanups.Clear();
            _disposables.Clear();
            _disposed = true;
        }
        base.Dispose(disposing);
    }
}
```

---

## ObjectSpace — Scoped Disposal

**Never** store `IObjectSpace` in a static field, singleton, or Session.

```csharp
// ✅ Short-lived operation
public void ProcessData() {
    using var os = Application.CreateObjectSpace(typeof(MyObject));
    var obj = os.FindObject<MyObject>(CriteriaOperator.Parse("Name = ?", "Test"));
    obj.Status = Status.Processed;
    os.CommitChanges();
} // disposed here — tracked objects released

// ❌ Anti-pattern
private static IObjectSpace _globalOs; // never do this
```

---

## ObjectSpace — Batch Processing Large Datasets

```csharp
public void ProcessAllRecords() {
    const int batchSize = 500;
    int skip = 0;

    while (true) {
        using var os = Application.CreateObjectSpace(typeof(MyObject));
        var batch = os.GetObjects<MyObject>()
            .Skip(skip).Take(batchSize).ToList();

        if (batch.Count == 0) break;

        foreach (var obj in batch)
            Process(obj);

        os.CommitChanges();
        skip += batch.Count;
    }
}
```

---

## CollectionSource Disposal

```csharp
public class ListController : ViewController<ListView> {
    private CollectionSource _cs;

    protected override void OnActivated() {
        base.OnActivated();
        _cs = new CollectionSource(ObjectSpace, typeof(MyObject));
        _cs.Criteria["Date"] = CriteriaOperator.Parse(
            "CreatedOn >= ?", DateTime.Today.AddDays(-30));
        View.CollectionSource = _cs;
    }

    protected override void OnDeactivated() {
        _cs?.Dispose();
        _cs = null;
        base.OnDeactivated();
    }

    protected override void Dispose(bool disposing) {
        if (disposing) { _cs?.Dispose(); _cs = null; }
        base.Dispose(disposing);
    }
}
```

---

## Static Reference Anti-Patterns

```csharp
// ❌ Never store controller/view instances in static fields
private static List<Controller> _allControllers = new();     // leak
private static IObjectSpace _sessionOs;                       // leak
private static MyController _instance;                        // leak

// ✅ Use instance fields; clean up in Dispose
private readonly List<IDisposable> _ownedResources = new();
```

---

## Warning Signs

| Sign | Likely Cause |
|---|---|
| Memory grows with each navigation | Event handlers not unsubscribed in OnDeactivated |
| Same handler fires multiple times | Controller activated multiple times without deactivation |
| `OutOfMemoryException` under normal load | ObjectSpace retained / large unconstrained query |
| GC pauses increase over time | Static collections or long-lived ObjectSpaces |
| `CollectionSource` queries slow down | Cached collection not disposed between views |

---

## Diagnostic Tools

```csharp
// Enable XAF tracing to monitor ObjectSpace create/dispose
Tracing.Tracer.Initialize(TraceLevel.Verbose,
    Environment.GetFolderPath(Environment.SpecialFolder.Desktop) + "\\xaf_trace.log");

// Quick memory snapshot (development only)
var before = GC.GetTotalMemory(false);
GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();
var after = GC.GetTotalMemory(true);
Tracing.Tracer.LogValue("Memory retained after GC", before - after);
```

**Profilers:**
- **dotMemory** (JetBrains) — best for XAF object tracking
- **PerfView** (Microsoft, free) — .NET GC and heap analysis
- **Application Insights** — production memory monitoring

---

## Code Review Checklist

```
✅ Every View.SomeEvent += in OnActivated has -= in OnDeactivated AND Dispose
✅ No static fields storing Controller, View, or IObjectSpace references
✅ All IObjectSpace created outside the view use `using` or explicit Dispose
✅ CollectionSource disposed in OnDeactivated and Dispose
✅ Large dataset operations use batching (Take/Skip per ObjectSpace)
✅ No ObjectSpace stored in Session, Application, or singleton services
✅ Dispose(bool) calls base.Dispose(disposing) as the last statement
```

---

## Source Links

- Memory management best practices: https://docs.devexpress.com/eXpressAppFramework/112569/getting-started
- IObjectSpace.Dispose: https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.IObjectSpace
- Controller lifecycle: https://docs.devexpress.com/eXpressAppFramework/112621/ui-construction/controllers-and-actions
- XAF Tracing: https://docs.devexpress.com/eXpressAppFramework/112649/debugging-testing-and-error-handling/tracing
- dotMemory: https://www.jetbrains.com/dotmemory/
- PerfView: https://github.com/microsoft/perfview
