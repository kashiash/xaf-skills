---
name: xaf-multi-tenant
description: XAF Multi-tenancy (v24.2+) - separate database per tenant, AddMultiTenancy setup, ITenantProvider, built-in tenant resolvers (TenantByEmailResolver), custom resolvers, connection string per tenant, custom Tenant class, data isolation, host vs tenant UI, user-tenant association, scoped services, common pitfalls. Use when building multi-tenant SaaS applications with DevExpress XAF Blazor.
license: MIT
compatibility: opencode, claude-code
metadata:
  domain: xaf
  topic: multi-tenancy
  versions: v24.2, v25.1
---

# XAF: Multi-Tenancy

## Overview

XAF (v24.2+) built-in multi-tenancy for SaaS apps. One deployed instance serves multiple independent organizations (tenants) with **strict database isolation** (separate DB per tenant).

Key characteristics:
- Blazor Server (ASP.NET Core) — primary supported platform
- Two operating modes: **Host UI** (tenant management) and **Tenant UI** (tenant data)
- Tenant connection strings stored in the host database
- Built-in tenant resolver infrastructure

---

## Approaches

| Approach | Isolation | Notes |
|---|---|---|
| **Separate DB per tenant** | Full | Default XAF approach; maximum isolation |
| Shared DB with discriminator | Partial | **Not natively supported** — must implement manually |
| Hybrid | Mixed | Shared config in host DB; operational data in tenant DBs |

---

## Setup (Blazor / v24.2+)

```
dotnet add package DevExpress.ExpressApp.MultiTenancy
```

```csharp
// Program.cs
builder.Services.AddXaf(builder.Configuration, b => {
    b.UseApplication<MyBlazorApplication>();

    b.AddObjectSpaceProviders(providers => {
        providers.UseEntityFramework(ef => {
            ef.DefaultDatabaseConnection("HostConnection", p => p
                .UseDbContext<HostDbContext>());
        });
    });

    b.Security
        .UseIntegratedMode(options => {
            options.RoleType = typeof(PermissionPolicyRole);
            options.UserType = typeof(ApplicationUser);
        })
        .AddPasswordAuthentication();

    b.AddMultiTenancy()
        .WithTenantResolver<TenantByEmailResolver>(); // {User}@{Tenant}
});
```

---

## Built-in Tenant Resolvers

| Resolver | Login Pattern | Example |
|---|---|---|
| `TenantByEmailResolver` | `{User}@{Tenant}` | `john@acme` → tenant = "acme" |
| `TenantByUserNameResolver` | Custom regex | `acme//john` → tenant = "acme" |

---

## Custom Tenant Resolver

```csharp
using DevExpress.ExpressApp.MultiTenancy;

public class TenantBySubdomainResolver : ITenantResolver {
    readonly IHttpContextAccessor _httpContext;

    public TenantBySubdomainResolver(IHttpContextAccessor httpContext) {
        _httpContext = httpContext;
    }

    public Task<string?> GetTenantNameAsync(string? loginName) {
        var host = _httpContext.HttpContext?.Request.Host.Host ?? "";
        var subdomain = host.Split('.').FirstOrDefault();
        return Task.FromResult<string?>(subdomain);
    }
}

// Register:
b.AddMultiTenancy().WithTenantResolver<TenantBySubdomainResolver>();
```

---

## ITenantProvider Interface

```csharp
public interface ITenantProvider {
    Guid? TenantId { get; }        // null in Host UI or not logged in
    string? TenantName { get; }    // null in Host UI or not logged in
    object? TenantObject { get; }  // the Tenant persistent object
}
```

### Getting the Current Tenant

**From ObjectSpace:**
```csharp
var tenantProvider = objectSpace.ServiceProvider.GetService<ITenantProvider>();
Guid? tenantId     = tenantProvider?.TenantId;
string? tenantName = tenantProvider?.TenantName;
```

**In Controllers via DI:**
```csharp
public class TenantAwareController : ViewController {
    ITenantProvider _tenantProvider;

    [ActivatorUtilitiesConstructor]
    public TenantAwareController(IServiceProvider serviceProvider) : base() {
        _tenantProvider = serviceProvider.GetRequiredService<ITenantProvider>();
    }

    protected override void OnActivated() {
        base.OnActivated();
        var tenantName = _tenantProvider.TenantName;
        // null → Host UI context
    }
}
```

---

## Tenant-Aware Connection Strings

Each tenant's connection string is stored as a property of the `Tenant` object in the host DB. XAF uses `IConnectionStringProvider` to route requests to the correct tenant DB automatically.

```csharp
// XAF handles routing automatically. Programmatic access if needed:
var connectionStringProvider = serviceProvider.GetService<IConnectionStringProvider>();
string? connStr = connectionStringProvider?.GetConnectionString();
```

---

## Custom Tenant Class

```csharp
public class CustomTenant : Tenant {
    public virtual string? LogoUrl { get; set; }
    public virtual string? PrimaryColor { get; set; }
    public virtual int MaxUsers { get; set; }
}

// Register:
b.AddMultiTenancy()
    .WithCustomTenantType<CustomTenant>()
    .WithTenantResolver<TenantByEmailResolver>();
```

Access:
```csharp
var tenant = tenantProvider.TenantObject as CustomTenant;
string? logoUrl = tenant?.LogoUrl;
```

---

## Data Isolation

```csharp
// In Tenant UI: queries ONLY the current tenant's database — automatic
var orders = objectSpace.GetObjects<Order>();

// Guard for type availability in current context
if (objectSpace.CanInstantiate(typeof(Order))) {
    // safe in this context
}
```

No built-in cross-tenant query. For aggregated data across tenants, implement outside XAF's ObjectSpace.

---

## User-Tenant Association

- **Host DB**: super admin accounts, tenant list, shared objects
- **Tenant DB**: tenant-specific users, roles, permissions, business data

```csharp
// Create tenant user (run in tenant's ObjectSpace context)
var tenantUser = objectSpace.CreateObject<ApplicationUser>();
tenantUser.UserName = "newuser";
tenantUser.SetPassword("SecurePassword1!");
tenantUser.Roles.Add(defaultRole);
objectSpace.CommitChanges();
```

---

## Admin Roles

| Role | DB | Responsibilities |
|---|---|---|
| Super Administrator | Host DB | Creates/deletes tenants, host config |
| Tenant Administrator | Tenant DB | Users, roles, permissions within tenant |
| Tenant User | Tenant DB | Normal end-user access |

```csharp
// Host Updater.cs
var superAdminRole = objectSpace.CreateObject<PermissionPolicyRole>();
superAdminRole.Name = "SuperAdministrators";
superAdminRole.IsAdministrative = true;

// Tenant Updater.cs (runs per-tenant DB)
var tenantAdminRole = objectSpace.CreateObject<PermissionPolicyRole>();
tenantAdminRole.Name = "TenantAdministrators";
tenantAdminRole.IsAdministrative = true;
```

---

## Modules in Multi-Tenant Apps

| Module | Host UI | Tenant UI |
|---|---|---|
| Business data modules | No | Yes |
| Dashboards | No | Yes |
| File Attachments | No | Yes |
| Reports V2 | No | Yes |
| Scheduler | No | Yes |
| Security module | Yes | Yes |

Modules that register persistent types cannot operate in Host UI mode.

---

## Common Pitfalls

| Pitfall | Solution |
|---|---|
| Singleton services holding tenant state | Use **scoped** services; never store tenant data in singletons |
| Same connection string for host and tenant | Always different connection strings — framework enforces this |
| Shared in-memory cache leaking across tenants | Use per-tenant cache keys: `$"{tenantId}:{cacheKey}"` |
| Querying business types in Host UI | Guard with `objectSpace.CanInstantiate(typeof(T))` |
| Missing DB migration per tenant | Run migration per tenant DB via XAF's updater infrastructure |

```csharp
// WRONG
services.AddSingleton<IMyService, MyService>(); // holds tenant-specific state

// CORRECT
services.AddScoped<IMyService, MyService>();

// CORRECT: per-tenant cache keys
string cacheKey = $"{tenantProvider.TenantId}:ProductList";
cache.Set(cacheKey, products);
```

---

## Source Links

- Multi-tenancy: https://docs.devexpress.com/eXpressAppFramework/404436/multitenancy
- Create Multitenant Application: https://docs.devexpress.com/eXpressAppFramework/404436/multitenancy/create-a-multitenant-application
- Multi-Tenant Architecture: https://docs.devexpress.com/eXpressAppFramework/404436/multitenancy/multi-tenant-application-architecture
- Get Current Tenant in Code: https://docs.devexpress.com/eXpressAppFramework/404668/multitenancy/get-the-current-tenant-in-code
- ITenantProvider API: https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.MultiTenancy.ITenantProvider
