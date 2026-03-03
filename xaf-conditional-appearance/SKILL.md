---
name: xaf-conditional-appearance
description: XAF Conditional Appearance - ConditionalAppearanceModule setup, [Appearance] attribute with all parameters (Criteria, TargetItems, Context, AppearanceItemType, Visibility, Enabled, FontColor, BackColor, FontStyle, CSS), criteria expression syntax, targeting multiple properties, coloring list view rows, hiding actions, dynamic appearance from code via IAppearanceEnabled/IAppearanceVisibility, model-based rules. Use when conditionally hiding, disabling, or styling UI elements in DevExpress XAF.
license: MIT
compatibility: opencode, claude-code
metadata:
  domain: xaf
  topic: conditional-appearance
  versions: v24.2, v25.1
---

# XAF: Conditional Appearance

## Overview

Conditional Appearance controls UI elements dynamically based on business rules: visibility, enabled state, colors, font style, and custom CSS — all without writing controller code for simple cases.

---

## Setup

```csharp
// In Module:
public class MyModule : ModuleBase {
    public MyModule() {
        RequiredModuleTypes.Add(typeof(ConditionalAppearanceModule));
    }
}

// Or in Program.cs:
b.AddModule<ConditionalAppearanceModule>();
```

---

## [Appearance] Attribute — All Parameters

```csharp
[Appearance(
    "RuleId",                                  // unique rule identifier (required)
    Criteria = "Status = 'Cancelled'",         // XAF criteria expression (required)
    TargetItems = "CancellationReason",        // property name(s), "*" = all, ";" = multiple
    Context = "DetailView",                    // "Any", "DetailView", "ListView", or custom
    AppearanceItemType = "ViewItem",           // "ViewItem", "Action", "LayoutItem"
    Visibility = ViewItemVisibility.Show,      // Show, Hide, ShowEmptySpace
    Enabled = false,                           // true/false
    FontColor = "Red",                         // named color or hex (#FF0000)
    BackColor = "LightYellow",                 // background color
    FontStyle = FontStyle.Bold,                // Bold, Italic, Strikeout, Underline
    Priority = 0                               // higher priority wins when multiple rules apply
)]
```

Apply to a **class** to affect properties within it:
```csharp
[Appearance("Rule", Criteria = "...", TargetItems = "PropertyName")]
public class Order : BaseObject { ... }
```

Apply to a **property** itself (targets that property):
```csharp
public class Order : BaseObject {
    [Appearance("RequiredWhenActive", Criteria = "Status = 'Active'",
        FontColor = "Red")]
    public virtual string ResponsiblePerson { get; set; }
}
```

---

## Common Patterns

### Hide field based on enum value

```csharp
[Appearance("HideCancellationReason",
    Criteria = "Status != 'Cancelled'",
    TargetItems = "CancellationReason",
    Visibility = ViewItemVisibility.Hide)]
[Appearance("ShowCancellationReason",
    Criteria = "Status = 'Cancelled'",
    TargetItems = "CancellationReason",
    Visibility = ViewItemVisibility.Show)]
public class Order : BaseObject {
    public virtual OrderStatus Status { get; set; }
    public virtual string CancellationReason { get; set; }
}
```

### Disable field based on boolean

```csharp
[Appearance("LockWhenApproved",
    Criteria = "IsApproved = true",
    TargetItems = "Amount;Description;Category",
    Enabled = false)]
public class Invoice : BaseObject {
    public virtual bool IsApproved { get; set; }
    public virtual decimal Amount { get; set; }
    public virtual string Description { get; set; }
    public virtual Category Category { get; set; }
}
```

### Color list view row based on status

```csharp
[Appearance("OverdueRow",
    Criteria = "DueDate < LocalDateTimeToday() AND Status != 'Completed'",
    Context = "Any",
    AppearanceItemType = "DataRow",
    BackColor = "#FFEEEE",
    FontColor = "DarkRed")]
public class Task : BaseObject {
    public virtual DateTime DueDate { get; set; }
    public virtual TaskStatus Status { get; set; }
}
```

### Hide action based on condition

```csharp
[Appearance("HideDeleteWhenLocked",
    Criteria = "IsLocked = true",
    AppearanceItemType = "Action",
    TargetItems = "DeleteAction",
    Visibility = ViewItemVisibility.Hide)]
public class Contract : BaseObject {
    public virtual bool IsLocked { get; set; }
}
```

### Custom CSS class in Blazor

```csharp
[Appearance("HighlightPremium",
    Criteria = "IsPremium = true",
    Context = "DetailView",
    CssClass = "premium-highlight")]
public class Customer : BaseObject {
    public virtual bool IsPremium { get; set; }
}
```

---

## Criteria Expression Syntax

| Operator | Example |
|---|---|
| Equals | `Status = 'Active'` |
| Not equals | `Status != 'Cancelled'` |
| Comparison | `Amount > 1000` |
| AND / OR / NOT | `Active = true AND Amount > 0` |
| IsNull | `IsNull([Manager])` |
| IsNullOrEmpty | `IsNullOrEmpty([Email])` |
| Contains | `Contains([Name], 'Smith')` |
| StartsWith | `StartsWith([Code], 'ORD')` |
| Date functions | `DueDate < LocalDateTimeToday()` |
| CurrentUser | `Owner.UserName = CurrentUserId()` |
| Enum | `Status = ##Enum#MyNamespace.Status,Active##` |

---

## Targeting Multiple Properties

```csharp
// Multiple properties: separate with semicolon
TargetItems = "FirstName;LastName;Email"

// All properties
TargetItems = "*"

// All properties except exclusions: no direct support
// Use separate rules with Priority to handle order
```

---

## Multiple Appearance Rules (Priority)

```csharp
[Appearance("Rule1", Criteria = "Amount > 1000", BackColor = "LightGreen", Priority = 1)]
[Appearance("Rule2", Criteria = "Amount > 5000", BackColor = "Gold", Priority = 2)]
public virtual decimal Amount { get; set; }
// Higher Priority wins when multiple criteria are true
```

---

## Dynamic Appearance from Code

### Using IAppearanceEnabled / IAppearanceVisibility

```csharp
public class OrderViewController : ObjectViewController<DetailView, Order> {
    protected override void OnActivated() {
        base.OnActivated();
        View.CurrentObjectChanged += UpdateAppearance;
        UpdateAppearance(null, null);
    }

    protected override void OnDeactivated() {
        View.CurrentObjectChanged -= UpdateAppearance;
        base.OnDeactivated();
    }

    private void UpdateAppearance(object sender, EventArgs e) {
        if (View.CurrentObject is not Order order) return;

        // Find view items and change appearance programmatically
        var amountItem = View.FindItem("Amount") as IAppearanceEnabled;
        if (amountItem != null)
            amountItem.Enabled = !order.IsLocked;

        var cancelReasonItem = View.FindItem("CancellationReason") as IAppearanceVisibility;
        if (cancelReasonItem != null)
            cancelReasonItem.Visibility = order.Status == OrderStatus.Cancelled
                ? ViewItemVisibility.Show
                : ViewItemVisibility.Hide;
    }
}
```

---

## Model-Based Rules (Application Model)

Define rules in the Application Model (Model Editor) without code changes:

Path: `Views > <ClassName>_DetailView > AppearanceRules`

Add `AppearanceRule` node with:
- `Criteria` — condition expression
- `TargetItems` — property names
- `Visibility`, `Enabled`, `FontColor`, `BackColor`, `FontStyle`

Useful for rules that business users need to modify without code deployment.

---

## AppearanceItemType Values

| Value | Targets |
|---|---|
| `ViewItem` | Property editor (field) |
| `DataRow` | Entire list view row |
| `Action` | Action button |
| `LayoutItem` | Layout group or tab |

---

## Source Links

- Conditional Appearance: https://docs.devexpress.com/eXpressAppFramework/113286/conditional-appearance
- AppearanceAttribute: https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.ConditionalAppearance.AppearanceAttribute
- IAppearanceEnabled: https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.ConditionalAppearance.IAppearanceEnabled
- IAppearanceVisibility: https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.ConditionalAppearance.IAppearanceVisibility
- Criteria Expression Syntax: https://docs.devexpress.com/CoreLibraries/4928/devexpress-data-library/criteria-language-syntax
