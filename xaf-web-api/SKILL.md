---
name: xaf-web-api
description: XAF Backend Web API Service - OData v4 REST endpoints, Program.cs setup with JWT auth, business object exposure with [AllowedAction]/[IgnoreDataMember], OData query options ($filter/$expand/$select/$orderby/$top/$skip/$count), JWT authentication flow, custom OData actions/functions via [Action] attribute, Swagger integration, security integration, file upload, v24.2 vs v25.1 differences. Use when building or consuming the XAF Web API backend service.
license: MIT
compatibility: opencode, claude-code
metadata:
  domain: xaf
  topic: web-api
  versions: v24.2, v25.1
---

# XAF: Backend Web API Service

## Overview

XAF Web API Service provides OData v4 REST endpoints for XAF business objects. It can run standalone (no UI) or alongside a Blazor app. All XAF security, validation, and ObjectSpace features work the same way.

---

## Project Setup

### NuGet Packages

```xml
<PackageReference Include="DevExpress.ExpressApp.WebApi" Version="25.1.*" />
<PackageReference Include="DevExpress.ExpressApp.WebApi.Jwt" Version="25.1.*" />
<PackageReference Include="DevExpress.ExpressApp.WebApi.Swashbuckle" Version="25.1.*" />
```

### Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddXaf(builder.Configuration, b => {
    b.UseApplication<MyWebApiApplication>();

    b.AddObjectSpaceProviders(providers => {
        providers.UseEntityFramework(ef => {
            ef.DefaultDatabaseConnection("Default", p =>
                p.UseDbContext<MyDbContext>());
        });
        providers.AddNonPersistent();
    });

    b.Security
        .UseIntegratedMode(options => {
            options.RoleType = typeof(PermissionPolicyRole);
            options.UserType = typeof(ApplicationUser);
        })
        .AddPasswordAuthentication();

    b.AddWebApi(options => {
        options.BusinessObject<Contact>();      // expose specific type
        options.BusinessObject<Order>();
        // or expose all: options.BusinessObjects.AddFromAssembly(typeof(Contact).Assembly);
    });
});

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => {
        options.TokenValidationParameters = new TokenValidationParameters {
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };
    });

builder.Services.AddSwaggerGen();

var app = builder.Build();
app.UseXaf();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.UseSwagger();
app.UseSwaggerUI();
app.Run();
```

---

## Controlling Business Object Exposure

```csharp
// Allow/deny specific CRUD operations
[AllowedAction(Operations.Read)]            // read-only
[AllowedAction(Operations.Create | Operations.Read | Operations.Write)]
public class Contact : BaseObject { ... }

// Exclude property from API response
[IgnoreDataMember]
public virtual string InternalNotes { get; set; }

// Programmatic exposure in Program.cs:
b.AddWebApi(options => {
    options.BusinessObject<Order>()
           .AllowedActions(Operations.Read | Operations.Create);
});
```

---

## OData Queries

### GET collection
```
GET /api/odata/Contact
GET /api/odata/Contact?$filter=City eq 'Warsaw'
GET /api/odata/Contact?$select=FirstName,LastName,Email
GET /api/odata/Contact?$expand=Department
GET /api/odata/Contact?$orderby=LastName asc,FirstName asc
GET /api/odata/Contact?$top=10&$skip=20&$count=true
```

### GET single object
```
GET /api/odata/Contact(guid'00000000-0000-0000-0000-000000000001')
```

### POST (create)
```http
POST /api/odata/Contact
Content-Type: application/json
Authorization: Bearer <token>

{ "FirstName": "Jan", "LastName": "Kowalski", "Email": "jan@example.com" }
```

### PATCH (update)
```http
PATCH /api/odata/Contact(guid'...')
Content-Type: application/json
Authorization: Bearer <token>

{ "Email": "new@example.com" }
```

### DELETE
```http
DELETE /api/odata/Contact(guid'...')
Authorization: Bearer <token>
```

---

## OData Filter Operators

| Operator | Example |
|---|---|
| Equals | `$filter=Status eq 'Active'` |
| Not equals | `$filter=Status ne 'Deleted'` |
| Greater/Less | `$filter=Amount gt 100` |
| Contains | `$filter=contains(Name, 'Smith')` |
| StartsWith | `$filter=startswith(Name, 'Jan')` |
| Null check | `$filter=Department eq null` |
| AND/OR | `$filter=City eq 'Warsaw' and Active eq true` |
| Date | `$filter=CreatedDate gt 2024-01-01` |

---

## Authentication

### Get Token
```http
POST /api/Authentication/Authenticate
Content-Type: application/json

{ "userName": "Admin", "password": "" }
```

Response: `"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."`

### Use Token
```http
GET /api/odata/Contact
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### HttpClient Example (C#)
```csharp
var httpClient = new HttpClient { BaseAddress = new Uri("https://myapp.com") };

// Get token
var tokenResponse = await httpClient.PostAsJsonAsync(
    "/api/Authentication/Authenticate",
    new { userName = "Admin", password = "" });
var token = await tokenResponse.Content.ReadAsStringAsync();

// Use token
httpClient.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", token.Trim('"'));

// Query data
var contacts = await httpClient.GetFromJsonAsync<ODataResult<Contact>>(
    "/api/odata/Contact?$filter=Active eq true");
```

---

## Custom Endpoints

### Custom OData Action (POST, modifies data)

```csharp
// In a controller (or XAF business logic class):
[ApiController]
[Route("api/[controller]")]
public class OrderController : ControllerBase {
    private readonly IObjectSpaceFactory _objectSpaceFactory;

    public OrderController(IObjectSpaceFactory objectSpaceFactory) {
        _objectSpaceFactory = objectSpaceFactory;
    }

    [HttpPost("api/odata/Order({key})/ProcessOrder")]
    public IActionResult ProcessOrder([FromRoute] Guid key) {
        using var os = _objectSpaceFactory.CreateObjectSpace(typeof(Order));
        var order = os.GetObjectByKey<Order>(key);
        if (order == null) return NotFound();
        order.Status = OrderStatus.Processed;
        os.CommitChanges();
        return Ok();
    }
}
```

Or via XAF's `[Action]` attribute on a non-persistent class:
```csharp
[DomainComponent]
public class OrderActions {
    [Action(Caption = "ProcessOrder")]
    public void ProcessOrder(Order order, IObjectSpace objectSpace) {
        order.Status = OrderStatus.Processed;
        objectSpace.CommitChanges();
    }
}
```

---

## Swagger / OpenAPI

Automatically available at `/swagger` when configured. All OData endpoints are documented.

---

## Security Integration

All OData endpoints automatically apply XAF security:
- Permissions filter which records are returned (object-level security)
- Member-level permissions hide specific properties
- `DenyAllByDefault` → explicit grants required for API access

No additional configuration needed — security works transparently via ObjectSpace.

---

## Pagination Pattern

```
GET /api/odata/Contact?$top=10&$skip=0&$count=true
```

Response includes `@odata.count` with total count. Client paginates by incrementing `$skip`.

---

## v24.2 vs v25.1 Differences

| Feature | v24.2 | v25.1 |
|---|---|---|
| OData version | v4 | v4 (same) |
| JWT auth | Available | Available (enhanced config) |
| Swagger | Available | Available |
| Batch requests | Limited | Improved |
| .NET target | .NET 8 | .NET 8 / .NET 9 |

---

## Source Links

- Backend Web API Service: https://docs.devexpress.com/eXpressAppFramework/403394/backend-web-api-service
- Get Started with Web API: https://docs.devexpress.com/eXpressAppFramework/403394/backend-web-api-service/get-started-with-the-web-api-service
- Use OData to Query Data: https://docs.devexpress.com/eXpressAppFramework/403551/backend-web-api-service/use-odata-to-query-data
- Authentication in Web API: https://docs.devexpress.com/eXpressAppFramework/403716/backend-web-api-service/authentication-in-web-api-projects
- Custom Endpoints: https://docs.devexpress.com/eXpressAppFramework/403715/backend-web-api-service/custom-endpoint
