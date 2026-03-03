---
name: xaf-xpo-models
description: XAF XPO persistent object models - base classes (XPObject, XPBaseObject, XPCustomObject, BaseObject), key attributes (Persistent, Size, Indexed, Association, Aggregated), one-to-many/many-to-many/one-to-one relations, PersistentAlias calculated fields, Session access, optimistic locking, common pitfalls. Use when defining business objects with XPO ORM in DevExpress XAF.
license: MIT
compatibility: opencode, claude-code
metadata:
  domain: xaf
  topic: data-modeling
  orm: xpo
  versions: v24.2, v25.1
---

# XAF: XPO Business Object Models

## Base Classes

| Class | Key Type | OID Field | Deferred Delete | Optimistic Lock | Use Case |
|---|---|---|---|---|---|
| `XPObject` | `int` (auto) | `OID` (int) | Yes (GCRecord) | Yes (auto) | Simple integer PK |
| `XPCustomObject` | Custom (`[Key]`) | User-defined | Yes (GCRecord) | Yes (auto) | Custom PK type (Guid, string, composite) |
| `XPBaseObject` | Custom (`[Key]`) | User-defined | No (immediate) | Yes (auto) | Physical deletion required |
| `XPLiteObject` | Custom (`[Key]`) | User-defined | No | No | Legacy/views/joins, no concurrency |
| `BaseObject` (BCL) | `Guid` (`Oid`) | `Oid` (Guid) | Yes | Yes | **Recommended** for new XAF projects |

**Inheritance chain:** `XPObject` → `XPCustomObject` → `XPBaseObject` → `PersistentBase`

**Deferred deletion:** XPO sets `GCRecord` to a non-null value instead of deleting the row.

**XAF recommendation:** Use `BaseObject` (from `DevExpress.Persistent.BaseImpl`) for new XPO entities.

```csharp
// Most common XAF XPO pattern
using DevExpress.Persistent.BaseImpl;
using DevExpress.Xpo;

public class Contact : BaseObject {
    public Contact(Session session) : base(session) { }

    private string firstName;
    public string FirstName {
        get => firstName;
        set => SetPropertyValue(nameof(FirstName), ref firstName, value);
    }
}

// XPObject (int key)
public class Product : XPObject {
    public Product(Session session) : base(session) { }
    private string name;
    public string Name {
        get => name;
        set => SetPropertyValue(nameof(Name), ref name, value);
    }
}

// XPCustomObject - custom key type
public class Setting : XPCustomObject {
    public Setting(Session session) : base(session) { }
    [Key]
    public string Key {
        get => key;
        set => SetPropertyValue(nameof(Key), ref key, value);
    }
    private string key;
}
```

---

## Key Attributes

| Attribute | Purpose | Example |
|---|---|---|
| `[Persistent("ColName")]` | Map to specific DB column/table name | `[Persistent("first_name")]` |
| `[NonPersistent]` | Exclude from DB (calculated/transient) | `[NonPersistent]` |
| `[Size(n)]` | String column length (default 100) | `[Size(255)]` |
| `[Size(SizeAttribute.Unlimited)]` | nvarchar(max) / TEXT | large strings/blobs |
| `[Indexed]` | Create DB index | `[Indexed]` |
| `[Indexed(Unique=true)]` | Unique constraint + index | `[Indexed(Unique=true)]` |
| `[Association("Name")]` | Declare relation endpoint (BOTH sides) | `[Association("Dept-Contacts")]` |
| `[Aggregated]` | Cascade delete child collection | on collection property |
| `[Key]` / `[Key(AutoGenerate=true)]` | Mark primary key | for XPBaseObject/XPLiteObject |
| `[PersistentAlias("expr")]` | Server-side calculated alias | `[PersistentAlias("Price*Qty")]` |
| `[DbType("nvarchar(50)")]` | Exact DB column type | `[DbType("decimal(18,4)")]` |
| `[Delayed]` | Defer loading (large blob/text) | on binary/ntext properties |
| `[FetchOnly]` | Read-only DB computed column | `[FetchOnly]` |
| `[OptimisticLocking(false)]` | Disable locking on class | class-level |
| `[OptimisticLockingIgnored]` | Exclude property from locking version | property-level |
| `[NullValue(0)]` | Default value for null reads | `[NullValue("")]` |
| `[DisplayName("Label")]` | UI display name | `[DisplayName("First Name")]` |
| `[VisibleInDetailView(false)]` | Hide in Detail View | property-level |
| `[NoForeignKey]` | Skip FK constraint generation | `[NoForeignKey]` |

---

## Relations

### One-to-Many

Both ends **MUST** use the same association name string.

```csharp
public class Department : BaseObject {
    public Department(Session session) : base(session) { }

    [Association("Department-Contacts")]
    public XPCollection<Contact> Contacts => GetCollection<Contact>(nameof(Contacts));
}

public class Contact : BaseObject {
    public Contact(Session session) : base(session) { }

    private Department department;
    [Association("Department-Contacts")]
    public Department Department {
        get => department;
        set => SetPropertyValue(nameof(Department), ref department, value);
    }
}
```

Add `[Aggregated]` to the collection for cascade delete.

### Many-to-Many

**Option A: Auto-generated intermediate table**

```csharp
public class Employee : BaseObject {
    public Employee(Session session) : base(session) { }

    [Association("Employee-Tasks")]
    public XPCollection<Task> Tasks => GetCollection<Task>(nameof(Tasks));
}

public class Task : BaseObject {
    public Task(Session session) : base(session) { }

    [Association("Employee-Tasks")]
    public XPCollection<Employee> Employees => GetCollection<Employee>(nameof(Employees));
}
// XPO auto-creates EmployeeTasks intermediate table
```

**Option B: Explicit intermediate class (extra fields)**

```csharp
public class EmployeeTask : BaseObject {
    public EmployeeTask(Session session) : base(session) { }

    private Employee employee;
    [Association("Employee-EmployeeTasks")]
    public Employee Employee {
        get => employee;
        set => SetPropertyValue(nameof(Employee), ref employee, value);
    }

    private Task task;
    [Association("Task-EmployeeTasks")]
    public Task Task {
        get => task;
        set => SetPropertyValue(nameof(Task), ref task, value);
    }
    public DateTime AssignedOn { get; set; }
}

public class Employee : BaseObject {
    [Association("Employee-EmployeeTasks"), Aggregated]
    public XPCollection<EmployeeTask> EmployeeTasks => GetCollection<EmployeeTask>(nameof(EmployeeTasks));

    [ManyToManyAlias(nameof(EmployeeTasks), nameof(EmployeeTask.Task))]
    public XPCollection<Task> Tasks => GetCollection<Task>(nameof(Tasks));
}
```

### One-to-One

```csharp
public class Person : BaseObject {
    public Person(Session session) : base(session) { }

    private Passport passport;
    public Passport Passport {
        get => passport;
        set {
            Passport prev = passport;
            SetPropertyValue(nameof(Passport), ref passport, value);
            if (!IsLoading) {
                if (prev != null && prev.Person == this) prev.Person = null;
                if (value != null && value.Person != this) value.Person = this;
            }
        }
    }
}
```

---

## Calculated / Non-Stored Properties

```csharp
// NonPersistent - in-memory only, NOT filterable server-side
[NonPersistent]
public string FullName => $"{FirstName} {LastName}";

// PersistentAlias - server-side, filterable, no extra DB column
[PersistentAlias("Price * Quantity * (1 - Discount)")]
public decimal TotalPrice {
    get { return Convert.ToDecimal(EvaluateAlias(nameof(TotalPrice))); }
}

// Aggregate alias
[PersistentAlias("Orders.Count()")]
public int OrderCount {
    get { return Convert.ToInt32(EvaluateAlias(nameof(OrderCount))); }
}

// FetchOnly - maps to DB computed column (read-only)
[FetchOnly]
[Persistent("computed_total")]
public decimal ComputedTotal {
    get { return GetPropertyValue<decimal>(nameof(ComputedTotal)); }
}
```

---

## Lifecycle Hooks

```csharp
public class Order : BaseObject {
    public Order(Session session) : base(session) { }

    public override void AfterConstruction() {
        base.AfterConstruction();
        OrderDate = DateTime.Now;
        Status = OrderStatus.New;
    }

    public override void OnSaving() {
        base.OnSaving();
        if (IsNewObject) CreatedDate = DateTime.UtcNow;
        LastModified = DateTime.UtcNow;
    }

    public override void OnLoaded() {
        base.OnLoaded();
        // refresh in-memory caches
    }
}
```

---

## Session Access

```csharp
// Query in same session
public IList<LineItem> GetPendingItems() {
    return Session.Query<LineItem>()
                  .Where(li => li.Invoice == this && !li.IsProcessed)
                  .ToList();
}

// Find by criteria
public static Contact FindByEmail(Session session, string email) {
    return session.FindObject<Contact>(
        CriteriaOperator.Parse("Email = ?", email));
}
```

**Never** create `new Session()` inside a persistent object. Always use `this.Session`.

---

## Optimistic Locking

XPO automatically adds `OptimisticLockField` (int) to most base classes.

```csharp
// Disable for high-volume log tables
[OptimisticLocking(false)]
public class AuditLog : XPObject {
    public AuditLog(Session session) : base(session) { }
}

// Exclude property from bumping lock version
[OptimisticLockingIgnored]
public DateTime LastViewed {
    get => lastViewed;
    set => SetPropertyValue(nameof(LastViewed), ref lastViewed, value);
}
private DateTime lastViewed;
```

---

## Common Pitfalls

| Pitfall | Solution |
|---|---|
| Missing `Session` constructor | Every XPO class **must** have `public MyClass(Session session) : base(session) { }` |
| Auto-properties `{ get; set; }` | Breaks change tracking — always use `SetPropertyValue` with backing field |
| `GetCollection<T>()` in constructor | Don't access collections during construction |
| `new XPCollection<T>(session)` in getter | Creates unbound collection every access — use `GetCollection<T>(nameof(Prop))` |
| `[Association]` on one side only | Both ends MUST have `[Association("same-name")]` |
| Default `[Size]` is 100 | Without `[Size]`, strings become `nvarchar(100)` |
| Cross-session object assignment | Never assign an object from one Session to another |
| `Session.Save()` inside `OnSaving()` | Causes infinite recursion — don't call Save/CommitChanges in hooks |
| `Session`/`IObjectSpace` stored in static field | Memory leak — always scope to operation with `using var os = ...` |
| `GetObjects<T>()` without criteria on large tables | Loads entire table — add `CriteriaOperator` or `Take()` |

---

## Source Links

- Business Model Design with XPO: https://docs.devexpress.com/eXpressAppFramework/113469/business-model-design-orm/business-model-design-with-xpo
- Base Persistent Classes: https://docs.devexpress.com/XPO/2026/create-a-data-model/base-persistent-classes
- XPObject API: https://docs.devexpress.com/XPO/DevExpress.Xpo.XPObject
- BaseObject (BCL XPO): https://docs.devexpress.com/eXpressAppFramework/DevExpress.Persistent.BaseImpl.BaseObject
- PersistentAliasAttribute: https://docs.devexpress.com/XPO/DevExpress.Xpo.PersistentAliasAttribute
- Collections and Associations: https://docs.devexpress.com/XPO/2055/create-a-data-model/persistent-objects-and-references
- Property Change Notifications: https://docs.devexpress.com/eXpressAppFramework/117395/business-model-design-orm/property-changed-event-in-business-classes
