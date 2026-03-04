---
name: xaf-security
description: XAF Security System - SecurityStrategyComplex, role-based access control, TypePermission/ObjectPermission/MemberPermission, PermissionSettingHelper, authentication types (Standard, ActiveDirectory, OAuth2 v24.2+), programmatic permission checks via ISecurityStrategyBase, SecurityOperations constants, DenyAllByDefault policy, JWT Web API integration, custom ApplicationUser/Role classes, Updater seeding. Use when implementing security, roles, permissions or authentication in DevExpress XAF.
---

# XAF: Security System

## Overview

XAF Security System provides role-based access control (RBAC) via `SecurityStrategyComplex`. It controls:
- **Authentication** — who can log in
- **Authorization** — what they can see and do (type/object/member level)
- ORM-level filtering: queries auto-modified so users only retrieve permitted records
- UI: navigation items, editors, actions hidden/disabled based on permissions

---

## Setup in Program.cs (ASP.NET Core Blazor)

```csharp
builder.Services.AddXaf(builder.Configuration, b => {
    b.UseApplication<MyBlazorApplication>();
    b.AddObjectSpaceProviders(providers => {
        providers.UseEntityFramework(ef => {
            ef.DefaultDatabaseConnection("Default", p => p.UseDbContext<MyDbContext>());
        });
    });

    b.Security
        .UseIntegratedMode(options => {
            options.RoleType = typeof(PermissionPolicyRole);
            options.UserType = typeof(ApplicationUser);
        })
        .AddPasswordAuthentication(options => {
            options.IsSupportChangePassword = true;
        });
});
```

WinForms:
```csharp
winApplication.Security = new SecurityStrategyComplex(
    typeof(ApplicationUser),
    typeof(PermissionPolicyRole),
    new AuthenticationStandard());
```

---

## Authentication Types

| Type | Method | Use Case |
|---|---|---|
| Username + Password | `AddPasswordAuthentication` | Standard apps |
| Windows / AD | `AddWindowsAuthentication` | Internal enterprise, SSO |
| OAuth2 / Entra ID | `AddOAuth2Authentication` **(v24.2+)** | Cloud SaaS, Azure AD |
| Mixed | Combine multiple `.Add*()` calls | Local + external providers |

### Active Directory

```csharp
b.Security
    .UseIntegratedMode(options => {
        options.RoleType = typeof(PermissionPolicyRole);
        options.UserType = typeof(ApplicationUser);
        options.NewUserRoleName = "Default";
    })
    .AddWindowsAuthentication(options => {
        options.CreateUserAutomatically();
    });
```

### OAuth2 / Entra ID (v24.2+)

```csharp
b.Security
    .UseIntegratedMode(options => {
        options.RoleType = typeof(PermissionPolicyRole);
        options.UserType = typeof(ApplicationUser);
    })
    .AddPasswordAuthentication()
    .AddOAuth2Authentication(options => {
        options.Providers.AddMicrosoftAccount(entra => {
            entra.ClientId = builder.Configuration["AzureAd:ClientId"]!;
            entra.ClientSecret = builder.Configuration["AzureAd:ClientSecret"]!;
            entra.TenantId = builder.Configuration["AzureAd:TenantId"]!;
        });
        options.CreateUserAutomatically();
    });
```

> v24.2: OAuth2 preview. v25.1: stabilized API, Google/GitHub providers added.

---

## Permission Model

| Type | Scope | Criteria |
|---|---|---|
| `TypePermission` | Entire entity class | No |
| `ObjectPermission` | Specific records | Yes (criteria string/lambda) |
| `MemberPermission` | Individual property | Yes (combined with object criteria) |

### SecurityOperations Constants

```csharp
SecurityOperations.Read              // "Read"
SecurityOperations.Write             // "Write"
SecurityOperations.Create            // "Create"
SecurityOperations.Delete            // "Delete"
SecurityOperations.Navigate          // "Navigate"
SecurityOperations.FullAccess        // "Create;Read;Write;Delete;Navigate"
SecurityOperations.CRUDAccess        // "Create;Read;Write;Delete"
SecurityOperations.ReadOnlyAccess    // "Read;Navigate"
SecurityOperations.ReadWriteAccess   // "Read;Write"
```

### PermissionSettingHelper — Key Methods

```csharp
using DevExpress.ExpressApp.Security;

// Type-level
role.AddTypePermission<Order>(SecurityOperations.Read, SecurityPermissionState.Allow);
role.AddTypePermission(typeof(Order), SecurityOperations.FullAccess, SecurityPermissionState.Allow);

// Object-level with criteria string
role.AddObjectPermission<Order>(
    SecurityOperations.ReadWriteAccess,
    "[AssignedTo.UserName] = CurrentUserId()",
    SecurityPermissionState.Allow);

// Object-level with lambda
role.AddObjectPermissionFromLambda<Order>(
    SecurityOperations.ReadWriteAccess,
    o => o.Status == OrderStatus.Draft,
    SecurityPermissionState.Allow);

// Member-level: deny write on specific field
role.AddMemberPermission<Employee>(
    SecurityOperations.Write,
    nameof(Employee.Salary),
    null,   // null = all records
    SecurityPermissionState.Deny);
```

---

## Programmatic Role Setup (Updater.cs)

```csharp
public override void UpdateDatabaseAfterUpdateSchema() {
    base.UpdateDatabaseAfterUpdateSchema();

    // Admin Role
    var adminRole = ObjectSpace.FirstOrDefault<PermissionPolicyRole>(r => r.Name == "Administrators");
    if (adminRole == null) {
        adminRole = ObjectSpace.CreateObject<PermissionPolicyRole>();
        adminRole.Name = "Administrators";
        adminRole.IsAdministrative = true; // bypasses ALL permission checks
    }

    // Users Role (deny-all + specific grants)
    var userRole = ObjectSpace.FirstOrDefault<PermissionPolicyRole>(r => r.Name == "Users");
    if (userRole == null) {
        userRole = ObjectSpace.CreateObject<PermissionPolicyRole>();
        userRole.Name = "Users";
        userRole.PermissionPolicy = SecurityPermissionPolicy.DenyAllByDefault;

        userRole.AddTypePermission<Order>(SecurityOperations.ReadOnlyAccess, SecurityPermissionState.Allow);
        userRole.AddObjectPermission<Order>(
            SecurityOperations.Write,
            "[Owner.UserName] = CurrentUserId()",
            SecurityPermissionState.Allow);
    }

    // Admin User
    var adminUser = ObjectSpace.FirstOrDefault<ApplicationUser>(u => u.UserName == "Admin");
    if (adminUser == null) {
        adminUser = ObjectSpace.CreateObject<ApplicationUser>();
        adminUser.UserName = "Admin";
        adminUser.SetPassword("");
        adminUser.Roles.Add(adminRole);
    }

    ObjectSpace.CommitChanges();
}
```

---

## Permission Policies

| Policy | Behavior |
|---|---|
| `AllowAllByDefault` | Grants all unless explicitly denied |
| `DenyAllByDefault` | Denies all unless explicitly allowed **(production recommended)** |
| `ReadOnlyAllByDefault` | Allows Read/Navigate; denies CUD unless granted |

---

## Checking Permissions in Code

### Dependency Injection (recommended)

```csharp
public class MyService {
    readonly ISecurityStrategyBase _security;
    public MyService(ISecurityStrategyBase security) => _security = security;

    public bool CanCreateOrder() =>
        _security.IsGranted(new PermissionRequest(typeof(Order), SecurityOperations.Create));
}
```

### Shorthand Extensions

```csharp
using DevExpress.ExpressApp.Security;

bool canRead   = security.CanRead(typeof(Order));
bool canCreate = security.CanCreate(typeof(Order));
bool canWrite  = security.CanWrite(typeof(Order));
bool canDelete = security.CanDelete(typeof(Order));
```

### In Controllers

```csharp
protected override void OnActivated() {
    base.OnActivated();
    var security = Application.ServiceProvider.GetRequiredService<ISecurityStrategyBase>();
    if (!security.CanCreate(typeof(Order)))
        NewAction.Active["Permission"] = false;
}
```

---

## Custom Security Objects

```csharp
// EF Core custom user
[DefaultClassOptions]
public class ApplicationUser : PermissionPolicyUser, ISecurityUserWithLoginInfo {
    public ApplicationUser(DbContext objectSpace) : base(objectSpace) { }

    public virtual string? Department { get; set; }
    public virtual DateTime? LastLoginDate { get; set; }

    IEnumerable<ISecurityUserLoginInfo> ISecurityUserWithLoginInfo.LoginProviderInfos
        => LoginProviderInfos.OfType<ISecurityUserLoginInfo>();
    public virtual IList<ApplicationUserLoginInfo> LoginProviderInfos { get; set; }
        = new List<ApplicationUserLoginInfo>();
}

// Register:
b.Security.UseIntegratedMode(options => {
    options.UserType = typeof(ApplicationUser);
    options.RoleType = typeof(ExtendedRole); // optionally custom role
});
```

---

## Object-Level Security

XAF rewrites ORM queries automatically — no extra filtering needed:

```csharp
// Returns only permitted records automatically
var myOrders = objectSpace.GetObjects<Order>();
```

- **EF Core:** additional WHERE clauses injected via secured DbContext
- **XPO:** criteria applied at session/unit-of-work level

---

## Web API Integration

```csharp
b.Security
    .UseIntegratedMode(options => { ... })
    .AddPasswordAuthentication();

builder.Services.AddAuthentication()
    .AddJwtBearer(options => {
        options.TokenValidationParameters = new TokenValidationParameters {
            ValidIssuer = config["Jwt:Issuer"],
            ValidAudience = config["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(config["Jwt:Key"]!))
        };
    });
```

Token endpoint: `POST /api/Authentication/Authenticate`
```json
{ "userName": "Admin", "password": "" }
```
Use as: `Authorization: Bearer <token>`
All OData endpoints automatically apply XAF security.

---

## Common Patterns

```csharp
// Deny-all + grant specific
role.PermissionPolicy = SecurityPermissionPolicy.DenyAllByDefault;
role.AddTypePermission<Order>(SecurityOperations.ReadOnlyAccess, SecurityPermissionState.Allow);

// Admin bypass
adminRole.IsAdministrative = true;

// Per-user row-level security
role.AddObjectPermission<Task>(
    SecurityOperations.ReadWriteAccess,
    "[AssignedUser.UserName] = CurrentUserId()",
    SecurityPermissionState.Allow);
```

---

## Source Links

- Security System Overview: https://docs.devexpress.com/eXpressAppFramework/113480/data-security-and-safety/security-system/security-system-overview
- Authentication: https://docs.devexpress.com/eXpressAppFramework/113504/data-security-and-safety/security-system/authentication
- OAuth2: https://docs.devexpress.com/eXpressAppFramework/403188/data-security-and-safety/security-system/authentication/oauth2-authentication
- Role-Based Access Control: https://docs.devexpress.com/eXpressAppFramework/113558/data-security-and-safety/security-system/role-based-access-control
- PermissionSettingHelper: https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.Security.PermissionSettingHelper._methods
- SecurityOperations: https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.Security.SecurityOperations._members
- IsGrantedExtensions: https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.Security.IsGrantedExtensions
- GitHub Examples: https://github.com/DevExpress-Examples/XAF_Security_E4908
