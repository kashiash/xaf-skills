---
name: xaf-memory-leaks
description: XAF Memory Leak Prevention - event handler symmetry (OnActivated/OnDeactivated/Dispose), ObjectSpace scoped disposal with using statement, batch processing large datasets, IDisposable pattern for controllers with List<IDisposable> tracker, WeakEventSubscription, static reference anti-patterns, CollectionSource disposal, Session/HttpContext/Application anti-patterns (WebForms), ObjectSpacePool, controller lifecycle tracking, NavigationMonitor, warning signs, diagnostic tools (dotMemory, PerfView, XAF Tracing). Use when diagnosing memory leaks, auditing controller disposal, reviewing ObjectSpace lifetime, or reviewing Session usage in DevExpress XAF applications.
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
| ObjectSpace/objects in Session | Memory scales with active user count; session timeout failures |
| Controller holds undisposed resources | Finalizer queue pressure; slow GC |

---

## Event Handler Pattern — Most Common Leak

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

## Comprehensive Disposal Pattern — ResourceTrackingController

Use `List<IDisposable>` + `List<Action>` to track all resources safely:

```csharp
public class ResourceTrackingController : ViewController {
    private readonly List<IDisposable> _disposables = new();
    private readonly List<Action> _cleanups = new();
    private bool _disposed;

    protected T Track<T>(T resource) where T : IDisposable {
        _disposables.Add(resource);
        return resource;
    }

    protected void AddCleanup(Action cleanup) => _cleanups.Add(cleanup);

    protected override void OnActivated() {
        base.OnActivated();
        // Subscribe and auto-track cleanup
        View.SelectionChanged += OnSelectionChanged;
        AddCleanup(() => { if (View != null) View.SelectionChanged -= OnSelectionChanged; });

        // Track a CollectionSource
        var cs = Track(new CollectionSource(ObjectSpace, typeof(MyObject)));
    }

    protected override void OnDeactivated() {
        RunCleanups();
        base.OnDeactivated();
    }

    protected override void Dispose(bool disposing) {
        if (!_disposed && disposing) {
            RunCleanups();
            foreach (var d in _disposables)
                try { d?.Dispose(); } catch { }
            _disposables.Clear();
            _disposed = true;
        }
        base.Dispose(disposing);
    }

    private void RunCleanups() {
        foreach (var action in _cleanups)
            try { action(); } catch { }
        _cleanups.Clear();
    }
}
```

---

## Weak Event Subscription

For long-lived controllers that subscribe to events on short-lived objects:

```csharp
public class WeakEventSubscription : IDisposable {
    private WeakReference _sourceRef;
    private readonly string _eventName;
    private readonly EventHandler _handler;

    public WeakEventSubscription(object source, string eventName, EventHandler handler) {
        _sourceRef = new WeakReference(source);
        _eventName = eventName;
        _handler = handler;
        source.GetType().GetEvent(eventName)?.AddEventHandler(source, handler);
    }

    public void Dispose() {
        var source = _sourceRef?.Target;
        if (source != null)
            source.GetType().GetEvent(_eventName)?.RemoveEventHandler(source, _handler);
        _sourceRef = null;
    }
}

// Usage in controller:
private readonly List<WeakEventSubscription> _weakSubs = new();

protected override void OnActivated() {
    base.OnActivated();
    _weakSubs.Add(new WeakEventSubscription(
        View, nameof(View.SelectionChanged), OnSelectionChanged));
}

protected override void Dispose(bool disposing) {
    if (disposing) {
        foreach (var sub in _weakSubs) sub.Dispose();
        _weakSubs.Clear();
    }
    base.Dispose(disposing);
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

// ❌ Anti-patterns
private static IObjectSpace _globalOs;           // never
Session["XafObjectSpace"] = Application.CreateObjectSpace(); // never in Session
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

        // Force GC between large batches
        if (skip % 5000 == 0) {
            GC.Collect();
            GC.WaitForPendingFinalizers();
        }
    }
}
```

---

## ObjectSpace Pool (Advanced — High-Throughput Scenarios)

```csharp
public class ObjectSpacePool : IDisposable {
    private readonly ConcurrentQueue<IObjectSpace> _pool = new();
    private readonly XafApplication _app;
    private readonly int _maxSize;
    private int _currentSize;

    public ObjectSpacePool(XafApplication app, int maxSize = 10) {
        _app = app; _maxSize = maxSize;
    }

    public IObjectSpace Rent() =>
        _pool.TryDequeue(out var os)
            ? (Interlocked.Decrement(ref _currentSize), os).os
            : _app.CreateObjectSpace();

    public void Return(IObjectSpace os) {
        if (os == null) return;
        if (os.IsModified) os.ReloadChangedObjects();
        if (_currentSize < _maxSize) {
            _pool.Enqueue(os);
            Interlocked.Increment(ref _currentSize);
        } else os.Dispose();
    }

    public void Dispose() {
        while (_pool.TryDequeue(out var os)) os.Dispose();
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
        _cs?.Dispose(); _cs = null;
        base.OnDeactivated();
    }

    protected override void Dispose(bool disposing) {
        if (disposing) { _cs?.Dispose(); _cs = null; }
        base.Dispose(disposing);
    }
}
```

---

## Session / HttpContext (WebForms Only)

### Anti-Patterns

```csharp
// ❌ ObjectSpace in Session — lives until session expires (30 min+ of retained objects)
Session["XafObjectSpace"] = Application.CreateObjectSpace();

// ❌ Large collection in Session
Session["AllCustomers"] = objectSpace.GetObjects<Customer>();

// ❌ No cleanup on session end
protected void Application_Session_End(object sender, EventArgs e) {
    // Missing: dispose XAF objects
}
```

### Correct Patterns

```csharp
// ✅ Request-scoped ObjectSpace (disposed at end of request)
public static IObjectSpace GetRequestObjectSpace(XafApplication app) {
    var ctx = HttpContext.Current;
    if (ctx == null) return app.CreateObjectSpace();

    var os = ctx.Items["XafOs"] as IObjectSpace;
    if (os == null) {
        os = app.CreateObjectSpace();
        ctx.Items["XafOs"] = os;
        ctx.DisposeOnPipelineCompleted(os); // auto-dispose at request end
    }
    return os;
}

// ✅ Session_End cleanup — dispose all IDisposable objects stored in session
public class SessionCleanupModule : IHttpModule {
    public void Init(HttpApplication context) {
        context.Session_End += (s, e) => {
            var session = HttpContext.Current?.Session;
            if (session == null) return;
            foreach (string key in session.Keys.Cast<string>().ToList()) {
                if (session[key] is IDisposable d) {
                    try { d.Dispose(); session.Remove(key); }
                    catch (Exception ex) { Tracing.Tracer.LogError(ex.ToString()); }
                }
            }
        };
    }
    public void Dispose() { }
}
```

---

## Controller Lifecycle Tracker (Diagnostic)

Detect duplicate activations or missing deactivation in development:

```csharp
public class ControllerLifecycleTracker {
    private static readonly Dictionary<string, int> _active = new();

    public static void TrackActivation(Controller controller) {
        var key = controller.GetType().Name;
        _active[key] = _active.GetValueOrDefault(key, 0) + 1;
        if (_active[key] > 1)
            Tracing.Tracer.LogWarning(
                $"Multiple active instances of {key}: {_active[key]}");
    }

    public static void TrackDeactivation(Controller controller) {
        var key = controller.GetType().Name;
        if (_active.ContainsKey(key))
            if (--_active[key] <= 0) _active.Remove(key);
    }
}

// In your controller (development only):
protected override void OnActivated() {
    base.OnActivated();
    ControllerLifecycleTracker.TrackActivation(this);
}
protected override void OnDeactivated() {
    ControllerLifecycleTracker.TrackDeactivation(this);
    base.OnDeactivated();
}
```

---

## Navigation Memory Monitor (Diagnostic)

```csharp
public class NavigationMonitorController : WindowController {
    private static int _navCount;
    private static readonly Dictionary<string, int> _viewCounts = new();

    protected override void OnFrameAssigned() {
        base.OnFrameAssigned();
        if (Frame != null) Frame.ViewChanged += Frame_ViewChanged;
    }

    private void Frame_ViewChanged(object sender, ViewChangedEventArgs e) {
        _navCount++;
        if (e.View != null) {
            var t = e.View.GetType().Name;
            _viewCounts[t] = _viewCounts.GetValueOrDefault(t, 0) + 1;
        }

        if (_navCount % 10 == 0) {
            GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();
            Tracing.Tracer.LogValue("Navigations", _navCount);
            Tracing.Tracer.LogValue("Memory after nav",
                GC.GetTotalMemory(true));
        }
    }

    protected override void Dispose(bool disposing) {
        if (disposing && Frame != null)
            Frame.ViewChanged -= Frame_ViewChanged;
        base.Dispose(disposing);
    }
}
```

---

## Static Reference Anti-Patterns

```csharp
// ❌ Never store controller/view/ObjectSpace in static fields
private static List<Controller> _allControllers = new();  // leak
private static IObjectSpace _sessionOs;                    // leak
private static MyController _instance;                     // leak

// ✅ Use instance fields; clean up in Dispose
private readonly List<IDisposable> _ownedResources = new();
```

---

## Warning Signs

| Sign | Likely Cause |
|---|---|
| Memory grows with each navigation | Event handlers not unsubscribed in OnDeactivated |
| Same handler fires multiple times | Controller activated multiple times without deactivation |
| Memory scales with user count | ObjectSpace or objects stored in Session |
| `OutOfMemoryException` under normal load | ObjectSpace retained / large unconstrained query |
| GC pauses increase over time | Static collections or long-lived ObjectSpaces |
| IIS app pool recycling | Session memory pressure |
| Session timeout failures | Large objects (ObjectSpace) in session |

---

## Diagnostic Tools

```csharp
// Enable XAF tracing to monitor ObjectSpace create/dispose
Tracing.Tracer.Initialize(TraceLevel.Verbose,
    Environment.GetFolderPath(Environment.SpecialFolder.Desktop) + "\\xaf_trace.log");

// Memory snapshot (development only)
var before = GC.GetTotalMemory(false);
GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();
Tracing.Tracer.LogValue("Retained after GC", GC.GetTotalMemory(true));

// ObjectSpace internal tracking (debug reflection)
public static void LogObjectSpaceStats(IObjectSpace os) {
    if (os is not BaseObjectSpace baseOs) return;
    var modified = typeof(BaseObjectSpace)
        .GetField("modifiedObjects", BindingFlags.NonPublic | BindingFlags.Instance)
        ?.GetValue(baseOs) as IDictionary;
    Tracing.Tracer.LogValue("Modified objects in OS", modified?.Count ?? 0);
}
```

**Profilers:**
- **dotMemory** (JetBrains) — best for XAF object tracking
- **PerfView** (Microsoft, free) — .NET GC and heap analysis
- **Application Insights** — production memory monitoring

---

## Code Review Checklist

```
✅ Every View.Event += in OnActivated has -= in OnDeactivated AND Dispose
✅ No static fields storing Controller, View, or IObjectSpace references
✅ All IObjectSpace created outside the view use `using` or explicit Dispose
✅ CollectionSource disposed in OnDeactivated and Dispose
✅ Large dataset operations use batching (new ObjectSpace per batch)
✅ No ObjectSpace stored in Session, Application, or singleton services
✅ Session_End handler disposes all IDisposable objects in Session (WebForms)
✅ Dispose(bool) calls base.Dispose(disposing) as the last statement
✅ Double-disposal safe (guard with bool _disposed)
✅ Disposal methods wrap each operation in try/catch to ensure full cleanup
```

---

## Source Links

- Controller lifecycle: https://docs.devexpress.com/eXpressAppFramework/112621/ui-construction/controllers-and-actions
- IObjectSpace.Dispose: https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.IObjectSpace
- XAF Tracing: https://docs.devexpress.com/eXpressAppFramework/112649/debugging-testing-and-error-handling/tracing
- XafMemoryLeakDiagnosticMcp (source): https://github.com/egarim/XafMemoryLeakDiagnosticMcp
- dotMemory: https://www.jetbrains.com/dotmemory/
- PerfView: https://github.com/microsoft/perfview
