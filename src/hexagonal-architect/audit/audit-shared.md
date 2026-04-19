---
name: audit-shared
description: Audit layer for Shared to verify MediatR abstractions, IState interface, and no implementation details.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Shared Layer Audit

## Overview

This audit verifies the Shared layer contains only abstractions and interfaces needed across the solution.

## Audit Checklist

### 1. Dependencies

- [ ] Does NOT reference Application project
- [ ] Does NOT reference Domain project
- [ ] Does NOT reference Adapter projects

### 2. Content

- [ ] Contains MediatR abstractions (ICommand, IQuery, ICommandHandler, IQueryHandler)
- [ ] Contains Result<T> extensions
- [ ] Contains IState<T> repository interface
- [ ] Contains IStream interface
- [ ] Contains IStorage interface
- [ ] Contains IStartupRegister

### 3. IState Pattern (Repository Port)

```csharp
public interface IState<T> where T : class
{
    Task<T> GetAsync(string id);
    Task<T> GetAsync(Expression<Func<T, bool>> filter);
    Task<IEnumerable<T>> GetManyAsync(Expression<Func<T, bool>> filter);
    Task<Result<T>> AddAsync(T entity);
    Task<Result<T>> AddOrUpdateAsync(string id, T entity);
    Task<Result> UpdateAsync(string id, T entity);
    Task<Result> DeleteAsync(string id);
}
```

### 4. IStream Pattern (Service Bus)

```csharp
public interface IStream
{
    Task SendEventAsync<T>(T notification, string topicName) where T : IDomainEvent;
    Task SendEventAsync<T>(T notification, string topicName, DateTime? scheduleMessage) where T : IDomainEvent;
}
```

### 5. IStorage Pattern (Blob Storage)

```csharp
public interface IStorage
{
    Task<Uri> WriteAsync(string invocationId, string containerName, string fileName, MemoryStream stream, bool overwrite, CancellationToken ct);
    Task<Uri> WriteAsync(string invocationId, string containerName, string fileName, string bytes, bool overwrite = false, CancellationToken ct = default);
}
```

### 6. IStartupRegister Pattern

```csharp
public interface IStartupRegister
{
    IServiceCollection Register(IServiceCollection services, IConfiguration configuration, IServiceProvider serviceProvider);
}
```

## Common Violations

| Violation | Description | Fix |
|-----------|-------------|-----|
| Implementation | Concrete implementations in Shared | Move to Adapters |
| Application ref | Reference to Application | Remove reference |
| Domain ref | Reference to Domain | Remove reference |

## Audit Steps

1. Check for no project references
2. List all interfaces
3. Verify no implementations

---

## Quick Reference

| Check | Expected |
|-------|----------|
| References | None (pure abstraction) |
| IState<T> | Repository port interface |
| IStream | Service Bus interface |
| IStorage | Storage interface |
| Result extensions | FluentResults helpers |