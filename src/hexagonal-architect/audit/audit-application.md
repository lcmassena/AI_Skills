---
name: audit-application
description: Audit layer for Application to verify CQRS patterns, MediatR handlers, and FluentValidation implementation.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Application Layer Audit

## Overview

This audit verifies that the Application layer follows CQRS and MediatR patterns correctly.

## Audit Checklist

### 1. Dependencies

- [ ] Does NOT reference Adapters projects
- [ ] References Domain
- [ ] References Shared

### 2. Content

- [ ] Contains Command classes
- [ ] Contains Query classes
- [ ] Contains CommandHandler
- [ ] Contains QueryHandler
- [ ] Contains Validators (FluentValidation)

### 3. Command Pattern

```csharp
// Command - Write operation
public record CreateEntityCommand : ICommand<EntityResponse>
{
    public string PropertyOne { get; init; }
    public string PropertyTwo { get; init; }
}
```

### 4. Query Pattern

```csharp
// Query - Read operation
public record GetEntityQuery(string EntityId) : IQuery<EntityResponse>
{
}
```

### 5. Handler Pattern

```csharp
public class CreateEntityCommandHandler : ICommandHandler<CreateEntityCommand, EntityResponse>
{
    private readonly IEntityRepository _repository;
    private readonly IUnitOfWork _uow;

    public async Task<Result<EntityResponse>> Handle(
        CreateEntityCommand request, 
        CancellationToken cancellationToken)
    {
        var entity = request.ToEntity();
        var result = await _repository.AddAsync(entity);
        
        if (result.IsFailed)
            return Result.Fail(result.Errors);
            
        await _uow.CommitAsync();
        return Result.Ok(entity.ToResponse());
    }
}
```

### 6. Validator Pattern

```csharp
public class CreateEntityCommandValidator : AbstractValidator<CreateEntityCommand>
{
    public CreateEntityCommandValidator()
    {
        RuleFor(x => x.PropertyOne)
            .NotEmpty()
            .WithMessage("PropertyOne is required");
    }
}
```

## Common Violations

| Violation | Description | Fix |
|-----------|-------------|-----|
| Adapter reference | Direct database access | Use domain repository |
| Business logic | Logic in handler | Ensure delegation only |
| Missing FluentValidation | No input validation | Add validator |
| No ICommand | Using request directly | Implement interface |

## Audit Steps

1. Check project dependencies
2. List all commands and queries
3. Verify handler implementations
4. Check validators

---

## Quick Reference

| Check | Expected |
|-------|----------|
| Dependencies | Domain + Shared |
| Commands | Inherit from ICommand<R> |
| Queries | Inherit from IQuery<T> |
| Handlers | Inherit from ICommandHandler/IQueryHandler |
| Validators | FluentValidation.AbstractValidator |