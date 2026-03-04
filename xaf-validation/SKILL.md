---
name: xaf-validation
description: XAF Validation Module - ValidationModule setup, built-in rules (RuleRequiredField, RuleRegularExpression, RuleRange, RuleStringLength, RuleUniqueValue, RuleCriteria, RuleValueComparison, RuleIsReferenced), rule contexts (DefaultContexts.Save/Delete), [RuleSet] attribute, custom validation rules via RuleBase, programmatic validation via Validator, conditional validation, cross-property validation, validation in Web API. Use when adding data validation rules to XAF business objects.
---

# XAF: Validation Module

## Setup

```csharp
// In Module (platform-agnostic):
public class MyModule : ModuleBase {
    public MyModule() {
        RequiredModuleTypes.Add(typeof(ValidationModule));
    }
}

// Or in Blazor Program.cs:
b.AddModule<ValidationModule>();
```

---

## Built-in Validation Rules

### RuleRequiredField

```csharp
[RuleRequiredField("Rule1", DefaultContexts.Save, "Name is required")]
public virtual string Name { get; set; }

// With custom message
[RuleRequiredField("ContactEmail_Required", DefaultContexts.Save,
    "Email address is required",
    SkipNullOrEmptyValues = false)]
public virtual string Email { get; set; }
```

### RuleRegularExpression

```csharp
[RuleRegularExpression("EmailRule", DefaultContexts.Save,
    @"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$",
    "Invalid email format")]
public virtual string Email { get; set; }
```

### RuleRange

```csharp
[RuleRange("DiscountRange", DefaultContexts.Save, 0.0, 1.0,
    "Discount must be between 0 and 1")]
public virtual double Discount { get; set; }

[RuleRange("AgePastRange", DefaultContexts.Save,
    minimumValue: "1900-01-01", maximumValue: "$today$",
    ParametersMode = ParametersMode.Expression,
    MessageTemplate = "Birth date must be in the past")]
public virtual DateTime BirthDate { get; set; }
```

### RuleStringLength

```csharp
[RuleStringLength("NameLength", DefaultContexts.Save, 100,
    MinimumLength = 2,
    MessageTemplate = "Name must be 2-100 characters")]
public virtual string Name { get; set; }
```

### RuleUniqueValue

```csharp
[RuleUniqueValue("UniqueEmail", DefaultContexts.Save,
    "This email is already in use")]
public virtual string Email { get; set; }
```

### RuleCriteria

```csharp
// Criteria-based rule (XAF criteria language)
[RuleCriteria("ActiveOrderRule", DefaultContexts.Save,
    "Status != 'Cancelled' OR CancellationReason != ''",
    "Provide a cancellation reason when cancelling an order")]
public virtual string CancellationReason { get; set; }

// Is null or empty check
[RuleCriteria("HasPhoneOrEmail", DefaultContexts.Save,
    "!IsNullOrEmpty([Phone]) OR !IsNullOrEmpty([Email])",
    "Phone or email is required")]
public virtual string Phone { get; set; }
```

### RuleValueComparison

```csharp
// Compare two dates
[RuleValueComparison("EndDateAfterStart", DefaultContexts.Save,
    ValueComparisonType.GreaterThanOrEqual,
    "@StartDate",
    ParametersMode = ParametersMode.Expression,
    MessageTemplate = "End date must be on or after start date")]
public virtual DateTime EndDate { get; set; }

// Compare to constant
[RuleValueComparison("MinAmount", DefaultContexts.Save,
    ValueComparisonType.GreaterThan, 0,
    "Amount must be greater than 0")]
public virtual decimal Amount { get; set; }
```

### RuleIsReferenced

```csharp
// Prevent deletion if referenced by other objects
[RuleIsReferenced("CannotDeleteCategory", DefaultContexts.Delete,
    typeof(Product), nameof(Product.Category),
    MessageTemplateMustNotBeReferenced = "Cannot delete category with products")]
public virtual IList<Product> Products { get; set; }
```

---

## Rule Contexts

| Context | When | Description |
|---|---|---|
| `DefaultContexts.Save` | On ObjectSpace.CommitChanges | Standard "save" validation |
| `DefaultContexts.Delete` | On ObjectSpace.Delete | Validates before deletion |
| Custom context string | On demand | Trigger programmatically |

Use custom contexts for multi-step workflows:
```csharp
[RuleRequiredField("SubmitApproval", "Submit",
    "Approver is required to submit")]
public virtual Employee Approver { get; set; }
```

---

## [RuleSet] Attribute

Group rules to run together as a named context:

```csharp
public class Order : BaseObject {
    [RuleRequiredField("CustomerRequired_Save", DefaultContexts.Save, "Customer required")]
    [RuleRequiredField("CustomerRequired_Submit", "Submit", "Customer required to submit")]
    public virtual Customer Customer { get; set; }

    [RuleRequiredField("ApproverRequired_Submit", "Submit", "Approver required to submit")]
    public virtual Employee Approver { get; set; }
}
```

---

## Programmatic Validation

```csharp
using DevExpress.Persistent.Validation;

// Validate using a specific context
ValidationResult result = Validator.RuleSet.ValidateObject(
    objectSpace, order, new ContextIdentifiers("Submit"));

if (!result.Valid) {
    var messages = result.Results
        .Where(r => r.State == ValidationResultType.Error)
        .Select(r => r.ErrorMessage);
    throw new UserFriendlyException(string.Join("; ", messages));
}

// Validate all objects in the context
ValidationResult allResult = Validator.RuleSet.ValidateAllTargets(
    objectSpace, new ContextIdentifiers(DefaultContexts.Save));
```

---

## Custom Validation Rule

```csharp
using DevExpress.Persistent.Validation;
using DevExpress.ExpressApp.Validation;

[CodeRule]
public class FutureDateRule : RuleBase<DateTime> {
    public FutureDateRule() { }
    public FutureDateRule(IRuleBaseProperties properties) : base(properties) { }

    protected override ValidationResult IsValidInternal(DateTime value) {
        if (value > DateTime.Today)
            return CreateValidResult();
        return CreateInvalidResult($"Date must be in the future. Got: {value:d}");
    }
}
```

Apply the custom rule:
```csharp
[FutureDateRule("DeliveryDateFuture", DefaultContexts.Save,
    MessageTemplate = "Delivery date must be in the future")]
public virtual DateTime DeliveryDate { get; set; }
```

---

## Validating on Action (Custom Context)

```csharp
private void SubmitAction_Execute(object sender, SimpleActionExecuteEventArgs e) {
    var result = Validator.RuleSet.ValidateObject(
        ObjectSpace, ViewCurrentObject, new ContextIdentifiers("Submit"));

    if (!result.Valid) {
        var errors = result.Results
            .Where(r => r.State == ValidationResultType.Error)
            .Select(r => r.ErrorMessage);
        throw new UserFriendlyException("Validation failed: " + string.Join(", ", errors));
    }

    // Proceed with submit logic
    (ViewCurrentObject as Order).Submit();
    ObjectSpace.CommitChanges();
}
```

---

## Conditional Validation (Criteria-Based)

Rules can be conditionally skipped using `SkipNullOrEmptyValues` or `TargetCriteria`:

```csharp
// Only validate when Status = Active
[RuleRequiredField("ActiveContactEmail", DefaultContexts.Save,
    "Email required for active contacts",
    TargetCriteria = "Status = 'Active'")]
public virtual string Email { get; set; }
```

---

## Cross-Property Validation

```csharp
// Compare EndDate to StartDate (on EndDate property)
[RuleCriteria("EndAfterStart", DefaultContexts.Save,
    "[EndDate] >= [StartDate]",
    "End date must be after start date")]
public virtual DateTime EndDate { get; set; }

// Or use RuleValueComparison with expression
[RuleValueComparison("EndDateRule", DefaultContexts.Save,
    ValueComparisonType.GreaterThanOrEqual, "@StartDate",
    ParametersMode = ParametersMode.Expression,
    MessageTemplate = "End date must be >= start date")]
public virtual DateTime EndDate { get; set; }
```

---

## Validation in Web API

Validation rules fire automatically on `CommitChanges()` in the Web API service. Invalid requests return HTTP 400 with validation error details in the response body.

---

## Source Links

- Validation Module: https://docs.devexpress.com/eXpressAppFramework/113684/validation
- Validation Rules Overview: https://docs.devexpress.com/eXpressAppFramework/113684/validation/validation-rules-overview
- Implement Validation in Code: https://docs.devexpress.com/eXpressAppFramework/113684/validation/implement-validation-rules-in-code
- RuleRequiredField: https://docs.devexpress.com/eXpressAppFramework/DevExpress.Persistent.Validation.RuleRequiredFieldAttribute
- RuleCriteria: https://docs.devexpress.com/eXpressAppFramework/DevExpress.Persistent.Validation.RuleCriteriaAttribute
- RuleUniqueValue: https://docs.devexpress.com/eXpressAppFramework/DevExpress.Persistent.Validation.RuleUniqueValueAttribute
- Validator API: https://docs.devexpress.com/eXpressAppFramework/DevExpress.Persistent.Validation.Validator
