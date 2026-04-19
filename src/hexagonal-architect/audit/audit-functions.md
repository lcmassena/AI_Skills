---
name: audit-functions
description: Audit layer for Azure Functions to verify return types, MediatR delegation, configuration patterns, and error handling.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Functions Layer Audit

## Overview

This audit verifies that Azure Functions follow the defined patterns and best practices.

## Audit Checklist

### 1. Return Type Verification

- [ ] All HTTP Functions return `IActionResult` with `Result<T>`
- [ ] All Timer Functions return task without throwing exceptions
- [ ] No Functions return `void` (except void async)

### 2. Function Pattern

```csharp
// Correct - Returns Result<T>
public async Task<IActionResult> Post([HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequest req)
{
    var result = await _mediator.Send(command);
    return new OkObjectResult(result.Response);
}

// ERROR - Returns void
public void Run([HttpTrigger] HttpRequest req)
{
    // Business logic here - WRONG!
}

// CORRECT - Delegates to MediatR
public async Task<IActionResult> Run([HttpTrigger] HttpRequest req)
{
    var result = await _mediator.Send(command);
    return ProcessResult(result);
}
```

### 3. Configuration

```csharp
// All configuration from environment
var connectionString = Environment.GetEnvironmentVariable("ConnectionStrings_MyDb")
    ?? throw new InvalidOperationException("ConnectionStrings_MyDb not configured");
```

### 4. Token Validation

```csharp
// Token validation for protected endpoints
if (!TokenValidate.IsValid(req))
    return new UnauthorizedObjectResult(HttpStatusCode.Unauthorized);
```

### 5. Error Handling

```csharp
// Proper error handling
if (result.Status == EStatus.BusinessRuleInvalid)
    return ProcessBadRequest(result.Status);

return new OkObjectResult(result.Response);
```

## Common Violations

| Violation | Description | Fix |
|-----------|-------------|-----|
| Business logic in Function | Code other than validation/deserialization | Move to handler |
| Direct database access | Using DbContext directly | Use repository |
| Throwing exceptions | `throw new Exception()` | Return Result.Fail() |
| Missing Result return | Returning domain object directly | Wrap in Result<T> |
| Hardcoded config | Connection strings in code | Environment variables |

## Audit Steps

1. List all Functions in project
2. Verify return types
3. Check for direct dependencies
4. Verify configuration sources
5. Check token validation

---

## Quick Reference

| Check | Expected |
|-------|----------|
| Return type | `IActionResult` with Result<T> |
| Business logic | In handlers only |
| Config source | Environment variables |
| Validation | Token or JWT |