---
name: adapter-repository
description: Repository Adapter implementation using Entity Framework Core with generic SqlRepository base class, FluentResults, and Unit of Work pattern.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Repository Adapter (Hexagonal Architecture)

## Overview

The Repository Adapter implements database persistence for the Hexagonal Architecture pattern. This adapter handles all SQL Server operations using Entity Framework Core, following the repository pattern with a generic base class approach.

## Project Structure

```
/Adapters/Repository/SQL/
├── Data/
│   └── DataContext.cs           # EF Core DbContext
├── Mapping/
│   └── *.cs                   # Entity mappings
├── Models/
│   └── *.cs                   # Database entities
├── Repositories/
│   └── *.cs                   # Repository implementations
├── Services/
│   ├── SqlRepository.cs       # Generic base repository
│   └── SqlRepositoryExtensions.cs
├── Adapters.Repository.SQL.csproj
└── *.csproj
```

## Core Components

### SqlRepository<T> Base Class

The foundation of all repositories in the solution. This abstract class provides generic CRUD operations with proper error handling.

```csharp
public abstract class SqlRepository<T> : IState<T> where T : class
{
    protected readonly DataContext _context;
    private readonly DbSet<T> _dbSet;

    public SqlRepository(DataContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public virtual async Task<T> GetAsync(string id)
    {
        return await _dbSet.FindAsync(id);
    }

    public virtual async Task<T> GetAsync(Expression<Func<T, bool>> filter)
    {
        return await _dbSet.FirstOrDefaultAsync(filter);
    }

    public virtual async Task<IEnumerable<T>> GetManyAsync(
        Expression<Func<T, bool>> filter)
    {
        return await _dbSet.Where(filter).ToListAsync();
    }

    public virtual async Task<Result<T>> AddAsync(T entity)
    {
        try
        {
            await _dbSet.AddAsync(entity);
            await _context.SaveChangesAsync();
            return Result.Ok(entity);
        }
        catch (Exception ex)
        {
            return Result.Fail<T>(new Error(ex.Message).CausedBy(ex));
        }
    }

    public virtual async Task<Result<T>> AddOrUpdateAsync(
        string id, T entity)
    {
        try
        {
            var existing = await _dbSet.FindAsync(id);
            if (existing != null)
            {
                _context.Entry(existing).CurrentValues.SetValues(entity);
            }
            else
            {
                await _dbSet.AddAsync(entity);
            }
            await _context.SaveChangesAsync();
            return Result.Ok(entity);
        }
        catch (Exception ex)
        {
            return Result.Fail<T>(new Error(ex.Message).CausedBy(ex));
        }
    }

    public virtual async Task<Result> UpdateAsync(string id, T entity)
    {
        try
        {
            var existing = await _dbSet.FindAsync(id);
            if (existing == null)
                return Result.Fail("Entity not found");

            _context.Entry(existing).CurrentValues.SetValues(entity);
            await _context.SaveChangesAsync();
            return Result.Ok();
        }
        catch (Exception ex)
        {
            return Result.Fail(ex.Message);
        }
    }

    public virtual async Task<Result> DeleteAsync(string id)
    {
        try
        {
            var entity = await _dbSet.FindAsync(id);
            if (entity == null)
                return Result.Fail("Entity not found");

            _dbSet.Remove(entity);
            await _context.SaveChangesAsync();
            return Result.Ok();
        }
        catch (Exception ex)
        {
            return Result.Fail(ex.Message);
        }
    }

    public virtual async Task<IEnumerable<T>> GetPagedAsync(
        Expression<Func<T, bool>> filter, 
        int pageNumber, 
        int pageSize, 
        Expression<Func<T, object>>? sortBy = null, 
        bool ascending = true)
    {
        var query = _dbSet.Where(filter);

        if (sortBy != null)
        {
            query = ascending ? query.OrderBy(sortBy) 
                : query.OrderByDescending(sortBy);
        }

        return await query.Skip((pageNumber - 1) * pageSize)
            .Take(pageSize).ToListAsync();
    }
}
```

## Concrete Repository Implementation

All repository implementations should inherit from SqlRepository<T> and implement their specific interface.

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
            return Result.Fail<Entity>(
                new Error(ex.Message).CausedBy(ex));
        }
    }

    public async Task<Result<Entity>> GetByKeyAsync(
        string tenantId, 
        string entityId)
    {
        return await GetAsync(x => 
            x.TenantId == tenantId && 
            x.EntityId == entityId);
    }
}
```

## Best Practices

### 1. Always Return Result<T>

Never return null or throw exceptions. Use FluentResults for all operations.

```csharp
// Correct
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
        return Result.Fail<Entity>(
            new Error(ex.Message).CausedBy(ex));
    }
}

// Never do this
public async Task<Entity?> AddAsync(Entity entity)
{
    await _dbSet.AddAsync(entity);
    await _context.SaveChangesAsync();
    return entity; // Can return null - bad pattern
}
```

### 2. Use Try-Catch for All Operations

Wrap all database operations in try-catch blocks.

### 3. Implement AddOrUpdate for Upsert Operations

Use AddOrUpdateAsync for idempotent operations.

### 4. Use IState<T> Interface

All repositories must implement IState<T> from the Shared layer.

```csharp
public interface IEntityRepository : IState<Entity>
{
    Task<Result<Entity>> AddAsync(Entity entity);
    Task<Result<Entity>> GetByKeyAsync(
        string tenantId, 
        string entityId);
}
```

## Entity Mapping

Map domain entities to database models using EF Core Fluent API.

```csharp
public class EntityMapping : IEntityTypeConfiguration<Entity>
{
    public void Configure(EntityTypeBuilder<Entity> builder)
    {
        builder.ToTable("TB_ENTITY", "dbo");
        builder.HasKey(x => x.Id);
        
        builder.Property(x => x.TenantId)
            .HasColumnName("TENANT_ID")
            .HasMaxLength(10)
            .IsRequired();
            
        builder.Property(x => x.EntityId)
            .HasColumnName("ENTITY_ID")
            .HasMaxLength(50)
            .IsRequired();
            
        builder.Property(x => x.CreatedAt)
            .HasColumnName("CREATED_AT")
            .IsRequired();
            
        builder.Property(x => x.CreatedBy)
            .HasColumnName("CREATED_BY")
            .HasMaxLength(50)
            .IsRequired();
    }
}
```

## Unit of Work Pattern

Use IUnitOfWork for transactional operations.

```csharp
public interface IUnitOfWork
{
    Task CommitAsync();
    Task RollbackAsync();
}
```

## Anti-Patterns to Avoid

1. **DO NOT** throw exceptions - use Result.Fail()
2. **DO NOT** return null - return Result.Fail<T>() or Result.Ok(null) instead
3. **DO NOT** implement business logic in repositories
4. **DO NOT** access external services from repositories
5. **DO NOT** use async void - always use async Task

## Registration

Register repositories in the dependency injection container.

```csharp
// Program.cs
builder.Services.AddScoped<IEntityRepository, EntityRepository>();
builder.Services.AddScoped<DataContext>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
```

## References

- Base Repository: `/Adapters/Repository/SQL/Services/SqlRepository.cs`
- DataContext: `/Adapters/Repository/SQL/Data/DataContext.cs`
- Entity Mapping: `/Adapters/Repository/SQL/Mapping/*.cs`