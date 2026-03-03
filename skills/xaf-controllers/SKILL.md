---
name: xaf-controllers
description: XAF Controllers and Actions - ViewController, ObjectViewController<TView,TObject>, WindowController, ApplicationController, controller lifecycle (OnActivated/OnDeactivated/OnViewControlsCreated/OnViewShown), SimpleAction/PopupWindowShowAction/SingleChoiceAction/ParametrizedAction full patterns, async actions in Blazor (async void + InvokeAsync), Frame/NestedFrame, Active/Enabled collections, finding/refreshing objects. Use when creating custom actions, controllers, or view logic in DevExpress XAF.
license: MIT
compatibility: opencode, claude-code
metadata:
  domain: xaf
  topic: controllers
  versions: v24.2, v25.1
---

# XAF: Controllers & Actions

## Controller Types

| Type | Base Class | Typical Use |
|---|---|---|
| `ViewController` | `Controller` | Any view; most common base |
| `ViewController<TView>` | `ViewController` | Constrained to specific view type |
| `ObjectViewController<TView, TObject>` | `ViewController<TView>` | View + business object type — no manual casting |
| `WindowController` | `Controller` | Window-level; not tied to a view |
| `ApplicationController` | `Controller` | Global; activates for all views |

**Generic shorthand:**
```csharp
// Instead of TargetViewType + TargetObjectType + casting:
public class MyController : ObjectViewController<DetailView, Employee> {
    // View is DetailView, ViewCurrentObject is Employee — no cast needed
}
```

Place controllers in the platform-agnostic Module project (not in Blazor/WinForms projects) unless platform-specific.

---

## Controller Lifecycle

| Method | When Called | Typical Use |
|---|---|---|
| `OnActivated()` | Controller becomes active | Subscribe to view/object events, set initial state |
| `OnDeactivated()` | Controller becomes inactive | **Unsubscribe events**, release resources |
| `OnViewControlsCreated()` | After all UI controls created | Access and modify native UI controls |
| `OnViewShown()` | After view shown to user | Logic requiring fully rendered view |

```csharp
public class MyController : ViewController {
    protected override void OnActivated() {
        base.OnActivated();
        View.CurrentObjectChanged += View_CurrentObjectChanged;
    }

    protected override void OnDeactivated() {
        View.CurrentObjectChanged -= View_CurrentObjectChanged; // always unsubscribe
        base.OnDeactivated();
    }

    protected override void OnViewControlsCreated() {
        base.OnViewControlsCreated();
        // Access View.Control here
    }

    private void View_CurrentObjectChanged(object sender, EventArgs e) { }
}
```

---

## Action Types

| Type | Class | UI | Use Case |
|---|---|---|---|
| Simple | `SimpleAction` | Button | Trigger an operation |
| Parametrized | `ParametrizedAction` | Button + text input | Search, filter by user-typed value |
| Single Choice | `SingleChoiceAction` | Dropdown / radio list | Pick from predefined options |
| Popup Window | `PopupWindowShowAction` | Button opens modal | Select objects or confirm via popup |

---

## SimpleAction — Full Pattern

```csharp
public class ClearTasksController : ViewController {
    private SimpleAction clearTasksAction;

    public ClearTasksController() {
        TargetViewType = ViewType.DetailView;
        TargetObjectType = typeof(Employee);

        clearTasksAction = new SimpleAction(this, "ClearTasksAction", PredefinedCategory.View) {
            Caption = "Clear Tasks",
            ConfirmationMessage = "Are you sure?",
            ImageName = "Action_Clear",
        };
        clearTasksAction.Execute += ClearTasksAction_Execute;
    }

    private void ClearTasksAction_Execute(object sender, SimpleActionExecuteEventArgs e) {
        var employee = (Employee)View.CurrentObject;
        while (employee.DemoTasks.Count > 0)
            employee.DemoTasks.Remove(employee.DemoTasks[0]);
        View.ObjectSpace.CommitChanges();
        View.ObjectSpace.Refresh();
    }
}
```

---

## SimpleAction — Async Pattern (Blazor Critical!)

```csharp
// CORRECT: async void, no ConfigureAwait(false) in Blazor Server
private async void MyAction_Execute(object sender, SimpleActionExecuteEventArgs e) {
    try {
        // Long-running async work
        var result = await myService.DoWorkAsync(); // NO ConfigureAwait(false)!
        View.ObjectSpace.CommitChanges();
        View.Refresh();
        Application.ShowViewStrategy.ShowMessage("Success");
    }
    catch (Exception ex) {
        throw new UserFriendlyException($"Error: {ex.Message}");
    }
}

// If updating UI from non-Blazor thread, use InvokeAsync:
await InvokeAsync(() => {
    View.Refresh();
});
// InvokeAsync is available in BlazorApplication and BlazorController
```

---

## PopupWindowShowAction — Full Pattern

```csharp
public class PopupNotesController : ViewController {
    private PopupWindowShowAction showNotesAction;

    public PopupNotesController() {
        TargetObjectType = typeof(DemoTask);
        TargetViewType = ViewType.DetailView;

        showNotesAction = new PopupWindowShowAction(this, "ShowNotesAction", PredefinedCategory.Edit) {
            Caption = "Show Notes"
        };
        showNotesAction.CustomizePopupWindowParams += Action_CustomizePopupWindowParams;
        showNotesAction.Execute += Action_Execute;
    }

    private void Action_CustomizePopupWindowParams(
        object sender, CustomizePopupWindowParamsEventArgs e) {
        // Option 1: show existing objects list
        e.View = Application.CreateListView(typeof(Note), true);

        // Option 2: show new-object detail view
        // IObjectSpace os = Application.CreateObjectSpace(typeof(Note));
        // e.View = Application.CreateDetailView(os, os.CreateObject<Note>());
        // e.DialogController.SaveOnAccept = true;
    }

    private void Action_Execute(
        object sender, PopupWindowShowActionExecuteEventArgs e) {
        var task = (DemoTask)View.CurrentObject;
        foreach (Note note in e.PopupWindowViewSelectedObjects) {
            if (!string.IsNullOrEmpty(task.Description))
                task.Description += Environment.NewLine;
            task.Description += note.Text;
        }
        View.ObjectSpace.CommitChanges();
    }
}
```

---

## SingleChoiceAction — Full Pattern

```csharp
public class SetTaskController : ViewController {
    private SingleChoiceAction setTaskAction;

    public SetTaskController() {
        TargetObjectType = typeof(DemoTask);

        setTaskAction = new SingleChoiceAction(this, "SetTaskAction", PredefinedCategory.Edit) {
            Caption = "Set Task",
            ItemType = SingleChoiceActionItemType.ItemIsOperation,
            SelectionDependencyType = SelectionDependencyType.RequireMultipleObjects
        };

        // Add choice items
        var setPriorityItem = new ChoiceActionItem("Set Priority", null);
        setTaskAction.Items.Add(setPriorityItem);
        foreach (Priority value in Enum.GetValues(typeof(Priority))) {
            setPriorityItem.Items.Add(new ChoiceActionItem(value.ToString(), value));
        }

        setTaskAction.Execute += SetTaskAction_Execute;
    }

    private void SetTaskAction_Execute(object sender, SingleChoiceActionExecuteEventArgs e) {
        if (e.SelectedChoiceActionItem.Data is Priority priority) {
            foreach (DemoTask task in e.SelectedObjects.OfType<DemoTask>())
                task.Priority = priority;
            View.ObjectSpace.CommitChanges();
            View.ObjectSpace.Refresh();
        }
    }
}
```

---

## ParametrizedAction

```csharp
public class FindController : ViewController {
    private ParametrizedAction findAction;

    public FindController() {
        findAction = new ParametrizedAction(this, "FindByName", PredefinedCategory.View, typeof(string)) {
            Caption = "Find",
            NullValuePrompt = "Enter name..."
        };
        findAction.Execute += FindAction_Execute;
    }

    private void FindAction_Execute(object sender, ParametrizedActionExecuteEventArgs e) {
        var searchText = e.ParameterCurrentValue as string;
        if (string.IsNullOrEmpty(searchText)) return;

        var obj = ObjectSpace.FindObject<Contact>(
            CriteriaOperator.Parse("Contains([Name], ?)", searchText));
        if (obj != null)
            View.SelectObject(obj);
    }
}
```

---

## Finding Objects from Controller

```csharp
// Current object in DetailView
var current = View.CurrentObject as Employee;

// Selected objects in ListView
var selected = View.SelectedObjects.OfType<Employee>().ToList();

// Find by criteria
var obj = ObjectSpace.FindObject<Contact>(
    CriteriaOperator.Parse("Email = ?", "user@example.com"));

// Create object
var newObj = ObjectSpace.CreateObject<Order>();
```

---

## Refreshing View

```csharp
// Refresh list
View.ObjectSpace.Refresh();

// Refresh detail view display
View.Refresh();

// Commit + refresh
ObjectSpace.CommitChanges();
View.ObjectSpace.Refresh();
```

---

## Action Customization

```csharp
new SimpleAction(this, "MyAction", PredefinedCategory.View) {
    Caption = "My Action",
    ToolTip = "Does something",
    ImageName = "Action_New",     // image from ImageLoader
    ShortCaption = "My",          // shown in mobile/compact mode
    Shortcut = "Control+Shift+M",
    Category = "Reports",          // action container
    PaintStyle = ActionItemPaintStyle.CaptionAndImage,
};
```

---

## Activation Conditions

```csharp
// Active: key-based dictionary — action shown if all values are true
MyAction.Active.SetItemValue("HasPermission", security.CanCreate(typeof(Order)));
MyAction.Active.SetItemValue("IsCorrectView", View is DetailView);

// Enabled: same pattern — action shown but grayed out if any value is false
MyAction.Enabled.SetItemValue("HasSelection", View.SelectedObjects.Count > 0);
```

---

## Frame & NestedFrame

```csharp
// Access nested frame (e.g., in a MasterDetail view)
if (Frame is NestedFrame nestedFrame) {
    var parentController = nestedFrame.ParentFrame
        .GetController<ParentViewController>();
}

// Find controller in current frame
var refreshCtrl = Frame.GetController<RefreshController>();
refreshCtrl?.RefreshAction.DoExecute();
```

---

## Common Patterns

```csharp
// Show message
Application.ShowViewStrategy.ShowMessage("Operation complete", InformationType.Success);

// Navigate to object detail view
var showViewParams = Application.CreateDetailViewShowViewParameters(obj, objectSpace);
Application.ShowViewStrategy.ShowView(showViewParams, new ShowViewSource(Frame, null));

// Navigate programmatically to ListView
var listViewId = Application.FindListViewId(typeof(Order));
var listView = Application.CreateListView(listViewId, true);
Application.ShowViewStrategy.ShowView(
    new ShowViewParameters(listView), new ShowViewSource(Frame, null));
```

---

## Source Links

- Controllers: https://docs.devexpress.com/eXpressAppFramework/112623/ui-construction/controllers-and-actions/controllers
- Actions: https://docs.devexpress.com/eXpressAppFramework/112622/ui-construction/controllers-and-actions/actions
- SimpleAction How-To: https://docs.devexpress.com/eXpressAppFramework/113538/ui-construction/controllers-and-actions/actions/how-to-create-and-use-a-simple-action
- PopupWindowShowAction How-To: https://docs.devexpress.com/eXpressAppFramework/113539/ui-construction/controllers-and-actions/actions/how-to-create-and-use-a-popup-window-action
- Controllers and Actions: https://docs.devexpress.com/eXpressAppFramework/112621/ui-construction/controllers-and-actions
- Views: https://docs.devexpress.com/eXpressAppFramework/113504/ui-construction/views
