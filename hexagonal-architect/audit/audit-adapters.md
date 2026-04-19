---
name: audit-adapters
description: Audit layer for Adapters to verify implementation of Shared interfaces, FluentResults usage, and no business logic.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Adapters Layer Audit

## Overview

This audit verifies that Adapter implementations follow the hexagonal architecture pattern.

## Audit Checklist

### 1. Dependencies

- [ ] References Shared (implements interfaces)
- [ ] Does NOT reference Application
- [ ] Does NOT reference Domain

### 2. Content

- [ ] Contains repository implementations
- [ ] Contains service implementations
- [ ] Implements IState<T> interface

### 3. Repository Pattern

```csharp
public class EntityRepository : SqlRepository<Entity>, IEntityRepository
{
    public EntityRepository(DataContext context) : base(context) { }

    public async Task<Result<Entity>> AddAsync(Entity entity)
    {
        try
        {
            await _dbSet.AddAsync(entity);
            await _context.SaveChangesAsync();
            return Result.Ok(entity);
        }
        catch (Exception ex)
        {
            return Result.Fail<Entity>(new Error(ex.Message).CausedBy(ex));
        }
    }
}
```

### 4. Service Bus Pattern

```csharp
public class AzureServiceBusService : IStream
{
    public async Task SendEventAsync<T>(T notification, string topicName) 
        where T : IDomainEvent
    {
        var sender = Get(topicName);
        var message = new ServiceBusMessage(JsonConvert.SerializeObject(notification));
        await sender.SendMessageAsync(message);
    }
}
```

### 5. Storage Pattern

```csharp
public class AzureStorageService : IStorage
{
    public async Task<Uri> WriteAsync(string invocationId, string containerName, string fileName, MemoryStream stream, bool overwrite, CancellationToken ct)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        await containerClient.CreateIfNotExistsAsync(cancellationToken: ct);
        
        var blobClient = containerClient.GetBlobClient(fileName);
        await blobClient.UploadAsync(stream, options: GetMetadata(invocationId, overwrite), cancellationToken: ct);
        
        return blobClient.Uri;
    }
}
```

### 6. Data Access Only

- [ ] No business logic
- [ ] No validation
- [ ] Uses Result<T>.Fail() for errors
- [ ] No throws exceptions

## Common Violations

| Violation | Description | Fix |
|-----------|-------------|-----|
| Business logic | Business rules in adapter | Move to Application |
| Throwing exceptions | `throw ex` | Return Result.Fail() |
| Validation | Validation in adapter | Move to FluentValidation |
| Domain reference | Access to Domain logic | Remove reference |

## Audit Steps

1. Verify Shared reference only
2. List implementations
3. Check for Result<T> usage
4. Ensure no business logic

---

## Quick Reference

| Check | Expected |
|-------|----------|
| Reference | Shared only |
| Return type | Result<T> from FluentResults |
| Error handling | Result.Fail(), no throws |
| Logic | Data access only |