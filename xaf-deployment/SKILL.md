---
name: xaf-deployment
description: XAF Deployment - Blazor to IIS/Azure App Service/Docker, WinForms ClickOnce/MSI, database initialization (UpdateDatabaseBeforeOpen, DatabaseVersionMismatch), connection strings in appsettings.json (XPO vs EF Core keys), DevExpress license deployment (DEVEXPRESS_LICENSE_KEY env var), Serilog/ILogger integration, ASP.NET health checks, environment configuration (Development vs Production). Use when deploying DevExpress XAF Blazor or WinForms applications to production.
license: MIT
compatibility: opencode, claude-code
metadata:
  domain: xaf
  topic: deployment
  versions: v24.2, v25.1
---

# XAF: Deployment

## Blazor App Deployment

### IIS

```xml
<!-- web.config (auto-generated on publish) -->
<system.webServer>
  <aspNetCore processPath="dotnet"
              arguments=".\MyApp.dll"
              stdoutLogEnabled="false"
              stdoutLogFile=".\logs\stdout"
              hostingModel="inprocess">
    <environmentVariables>
      <environmentVariable name="ASPNETCORE_ENVIRONMENT" value="Production" />
    </environmentVariables>
  </aspNetCore>
</system.webServer>
```

Steps:
1. Publish: `dotnet publish -c Release -o ./publish`
2. Create IIS App Pool → .NET CLR version: **No Managed Code**, Pipeline: **Integrated**
3. Set physical path to publish folder
4. Install [ASP.NET Core Hosting Bundle](https://dotnet.microsoft.com/download)
5. Ensure `ASPNETCORE_ENVIRONMENT=Production` in IIS env vars

---

### Azure App Service

```bash
# Publish from CLI
dotnet publish -c Release -o ./publish
az webapp deploy --resource-group MyRG --name MyXafApp --src-path ./publish
```

**Connection strings in Azure App Service:**

Set via Portal → App Service → Configuration → Connection strings (or Environment Variables):
```
ConnectionStrings__Default=Server=myserver.database.windows.net;Database=MyDb;User Id=...;Password=...;
```

Or in `appsettings.Production.json`:
```json
{
  "ConnectionStrings": {
    "Default": "Server=myserver.database.windows.net;Database=MyDb;..."
  }
}
```

**DevExpress license in Azure:**
```
DEVEXPRESS_LICENSE_KEY=<your-license-key>
```
(Set as Application Setting in Azure Portal)

---

### Docker

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApp.Blazor/MyApp.Blazor.csproj", "MyApp.Blazor/"]
COPY ["MyApp.Module/MyApp.Module.csproj", "MyApp.Module/"]
RUN dotnet restore "MyApp.Blazor/MyApp.Blazor.csproj"
COPY . .
RUN dotnet build "MyApp.Blazor/MyApp.Blazor.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApp.Blazor/MyApp.Blazor.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.Blazor.dll"]
```

```yaml
# docker-compose.yml
services:
  app:
    image: myxafapp
    build: .
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__Default=Server=db;Database=MyApp;...
      - DEVEXPRESS_LICENSE_KEY=${DEVEXPRESS_LICENSE_KEY}
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=MyPassword123!
```

---

## WinForms Deployment

### ClickOnce

```xml
<!-- In .csproj -->
<PublishUrl>publish\</PublishUrl>
<Install>true</Install>
<ApplicationVersion>1.0.0.*</ApplicationVersion>
<BootstrapperEnabled>true</BootstrapperEnabled>
```

```bash
dotnet publish -c Release /p:PublishProfile=ClickOnce
```

### MSI Installer (WiX / Visual Studio Setup Project)

1. Add Setup Project to solution
2. Add project output of WinForms project
3. Add prerequisites: .NET Runtime, DevExpress redistributables
4. Build → produces `.msi`

---

## Database Initialization

### Auto-migration on startup (Development)

```csharp
// In WinApplication / BlazorApplication
DatabaseUpdateMode = DatabaseUpdateMode.UpdateDatabaseAlways;
```

### Production — controlled migration

```csharp
// Handle DatabaseVersionMismatch to control when migration runs
application.DatabaseVersionMismatch += (s, e) => {
    if (IsProductionSafeToMigrate()) {
        e.Updater.Update();
        e.Handled = true;
    }
    else {
        MessageBox.Show("Database requires migration. Contact administrator.");
        e.Handled = true;
    }
};
```

### EF Core — run migrations on startup

```csharp
// In Program.cs (Blazor) before app.Run():
using (var scope = app.Services.CreateScope()) {
    var factory = scope.ServiceProvider.GetRequiredService<IDbContextFactory<ApplicationDbContext>>();
    using var ctx = factory.CreateDbContext();
    ctx.Database.Migrate();
}
```

---

## Connection Strings

### appsettings.json (EF Core)

```json
{
  "ConnectionStrings": {
    "Default": "Server=(localdb)\\mssqllocaldb;Database=MyApp;Trusted_Connection=True;"
  }
}
```

### appsettings.json (XPO)

```json
{
  "ConnectionStrings": {
    "Default": "XpoProvider=MSSqlServer;data source=(localdb)\\mssqllocaldb;initial catalog=MyApp;integrated security=True;"
  }
}
```

### Multiple environments

```json
// appsettings.Development.json
{ "ConnectionStrings": { "Default": "Server=(localdb)..." } }

// appsettings.Production.json
{ "ConnectionStrings": { "Default": "Server=prod-server..." } }
```

Set environment: `ASPNETCORE_ENVIRONMENT=Production` (env var or `launchSettings.json`)

---

## DevExpress License Deployment

```bash
# Environment variable (recommended for CI/CD and containers)
DEVEXPRESS_LICENSE_KEY=your-devexpress-license-key

# Or file-based: %APPDATA%\DevExpress\license.key (Windows)
# Or: ~/.config/DevExpress/license.key (Linux/macOS)
```

Get your license key from [DevExpress Customer Center](https://www.devexpress.com/MyAccount/LicenseCenter/).

In CI/CD (GitHub Actions example):
```yaml
env:
  DEVEXPRESS_LICENSE_KEY: ${{ secrets.DEVEXPRESS_LICENSE_KEY }}
```

---

## Logging (Serilog)

```csharp
// Install: Serilog.AspNetCore, Serilog.Sinks.File, Serilog.Sinks.Console

builder.Host.UseSerilog((ctx, lc) => lc
    .WriteTo.Console()
    .WriteTo.File("logs/xaf-.log", rollingInterval: RollingInterval.Day)
    .ReadFrom.Configuration(ctx.Configuration));

// XAF uses ILogger<T> internally — logs to Serilog automatically
// Log from your own code:
public class MyController : ViewController {
    readonly ILogger<MyController> _logger;
    [ActivatorUtilitiesConstructor]
    public MyController(ILogger<MyController> logger) : base() {
        _logger = logger;
    }

    protected override void OnActivated() {
        base.OnActivated();
        _logger.LogInformation("Controller activated for view {ViewId}", View?.Id);
    }
}
```

---

## ASP.NET Health Checks

```csharp
// Add health checks
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>("database")
    .AddCheck("xaf", () => {
        // Custom XAF health check
        return HealthCheckResult.Healthy("XAF running");
    });

// Map health check endpoint
app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new HealthCheckOptions {
    Predicate = check => check.Tags.Contains("ready")
});
```

---

## Environment Configuration Checklist

| Setting | Development | Production |
|---|---|---|
| `DatabaseUpdateMode` | `UpdateDatabaseAlways` | `Never` or controlled |
| `ASPNETCORE_ENVIRONMENT` | `Development` | `Production` |
| Connection string | localdb / dev server | prod server |
| License key | from file | env var / secret |
| Logging level | Debug | Warning/Error |
| HTTPS | optional | required |
| HSTS | disabled | enabled |

---

## Source Links

- Deployment: https://docs.devexpress.com/eXpressAppFramework/113449/deployment
- Azure Deployment: https://docs.devexpress.com/eXpressAppFramework/113449/deployment/deployment-to-azure
- DevExpress License: https://docs.devexpress.com/eXpressAppFramework/403430/deployment/devexpress-license
- IIS Deployment (MS Docs): https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/
- Docker (MS Docs): https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/docker/
