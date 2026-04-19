---
name: audit-services
description: Audit layer for Services to verify configuration from environment variables, dependency injection, and .gitignore patterns.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Services Layer Audit

## Overview

This audit verifies that configuration and services follow best practices.

## Audit Checklist

### 1. Configuration

- [ ] All configuration from environment variables
- [ ] No hardcoded connection strings
- [ ] No hardcoded API keys
- [ ] No hardcoded secrets

```csharp
// Correct
var connectionString = Environment.GetEnvironmentVariable("ConnectionStrings_MyDb")
    ?? throw new InvalidOperationException("ConnectionStrings_MyDb not configured");

// ERROR - Hardcoded
var connectionString = "Server=localhost;Database=mydb;";
```

### 2. Program.cs Registration

- [ ] Dependency injection follows order
- [ ] MediatR registration
- [ ] Adapter registration
- [ ] Validator registration

```csharp
// Correct registration order
builder.Services.AddScoped<IEntityRepository, EntityRepository>();
builder.Services.AddSingleton<IStream, AzureServiceBusService>();
builder.Services.AddSingleton<IStorage, AzureStorageService>();

// MediatR
builder.Services.AddMediatR(cfg => 
    cfg.RegisterServicesFromAssembly(typeof(CreateEntityCommandHandler).Assembly));

// Validators
builder.Services.AddValidatorsFromAssembly(
    typeof(CreateEntityCommandValidator).Assembly);
```

### 3. Startup Registration

- [ ] IStartupRegister implementations
- [ ] Configuration loaded

```csharp
// Program.cs
builder.Services.RegisterFromAssemblies(typeof(IStartupRegister).Assembly);
```

## .gitignore Verification

Required exclusions:

```gitignore
# Sensitive files
*.appsettings.json
appsettings.*.json
!appsettings.example.json

# Secrets
*.pfx
*.key

# Local configuration
local.settings.json

# Environment files (optional - add to .gitignore if needed)
.env
.env.*
```

### Recommended .gitignore Additions

```gitignore
# App settings (sensitive)
**/appsettings.Development.json
**/appsettings.Local.json

# Secrets
*.pfx
*.key

# Azure publish profiles
*.publishproj
*.pubxml

# Environment variables
.env
.env.local
```

## Common Violations

| Violation | Description | Fix |
|-----------|-------------|-----|
| Hardcoded config | Connection strings in code | Environment variable |
| Missing validation | No FluentValidation registration | Add validator registration |
| Wrong DI order | Adapters before Shared | Fix registration order |
| Secrets in gitignore | Not tracked but committed | Verify .gitignore |

## Audit Steps

1. Check Program.cs registration order
2. Search for hardcoded values
3. Verify .gitignore entries

---

## Quick Reference

| Check | Expected |
|-------|----------|
| Config source | Environment.GetEnvironmentVariable |
| Registration | Shared → Adapters → MediatR |
| .gitignore | appsettings.*.json excluded |
| No secrets | Keys in env vars only |