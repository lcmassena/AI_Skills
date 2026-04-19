---
name: audit-domain
description: Audit layer for Domain to verify hexagonal architecture principles, entity patterns, and repository interfaces.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Domain Layer Audit

## Overview

This audit verifies that the Domain layer follows hexagonal architecture principles.

## Audit Checklist

### 1. Dependencies

- [ ] Does NOT reference Application project
- [ ] Does NOT reference Adapters projects
- [ ] MAY reference Shared (interfaces only)
- [ ] MAY reference other Domain entities

### 2. Content

- [ ] Contains Entities
- [ ] Contains Value Objects
- [ ] Contains Aggregate Roots
- [ ] Contains Domain Events
- [ ] Contains Repository Interfaces (Ports)

### 3. Entity Pattern

```csharp
// Entity should inherit from Entity base
namespace <Company>.<Project>.Domain.Entities;

public class EntityName : Entity<EntityName>
{
    public string PropertyOne { get; set; }
    public DateTime CreatedAt { get; set; }
    
    public bool IsValid() =>
        !string.IsNullOrEmpty(PropertyOne);
}
```

### 4. Repository Interface Pattern

```csharp
// Repository interfaces (ports) - NOT implementations
namespace <Company>.<Project>.Domain.Interfaces;

public interface IEntityRepository : IState<Entity>
{
    Task<Result<Entity>> AddAsync(Entity entity);
    Task<Result<Entity>> GetByIdAsync(string id);
}
```

### 5. Domain Events

```csharp
public class EntityCreatedEvent : IDomainEvent
{
    public Guid Id { get; } = Guid.NewGuid();
    public DateTime OccurredOn { get; } = DateTime.UtcNow;
    public string EntityId { get; init; }
}
```

## Common Violations

| Violation | Description | Fix |
|-----------|-------------|-----|
| Adapter reference | Dependencies on Infrastructure | Remove reference |
| Application reference | Business logic in Application | Move to Application |
| EF Core usage | Database access in Domain | Remove, use port |
| Implementation | Repository implementation | Move to Adapters |

## Audit Steps

1. Check project dependencies
2. List all entities
3. Verify repository interfaces
4. Check for no infrastructure code

---

## Quick Reference

| Check | Expected |
|-------|----------|
| Dependencies | Shared interfaces only |
| Entities | Inherits from Entity/AggRoot |
| Ports | Interface definitions only |
| Events | Implements IDomainEvent |