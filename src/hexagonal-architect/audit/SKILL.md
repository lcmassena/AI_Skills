---
name: audit
description: Use when auditing a .NET solution for hexagonal architecture compliance, verifying layer dependencies, CQRS patterns, MediatR abstractions, and best practices. Do not use for code review without architecture context.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Architecture Audit Skill

## Use this skill when

- Auditing the entire solution architecture
- Verifying hexagonal architecture layer dependencies
- Checking CQRS implementation patterns
- Validating MediatR abstractions usage
- Ensuring all Functions return Result<T>
- Auditing configuration and security
- Verifying .gitignore patterns

## Do not use this skill when

- Performing general code reviews without architecture context
- Auditing non-.NET applications
- Checking only business logic without architectural patterns

---

## Overview

This skill provides comprehensive audit guidance for verifying that a .NET solution follows the Hexagonal Architecture pattern with CQRS, MediatR abstractions, and the defined layer dependencies.

---

## Architecture Layers

### Layer Dependency Rules

```
┌─────────────────────────────────────────────────────┐
│              FUNCTIONS (API / Worker)                │
│  Can reference: Application, Domain, Shared, Adapters│
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                   APPLICATION                      │
│  Can reference: Domain, Shared                    │
│  MUST NOT reference: Adapters                      │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                      DOMAIN                       │
│  Can reference: Shared (interfaces only)           │
│  MUST NOT reference: Application, Adapters      │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                     SHARED                        │
│  Contains: ICommand, IQuery, ICommandHandler,    │
│            IQueryHandler, AggRoot, Entity         │
│  MUST NOT reference: Application, Domain         │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                    ADAPTERS                      │
│  Implements: Shared interfaces (ports)            │
│  Can reference: Shared                          │
└─────────────────────────────────────────────────────┘
```

---

## Shared Layer - MediatR Abstractions

### Required Interfaces (from AggRoot.cs)

All implementations MUST inherit from these interfaces:

```csharp
// Commands - Write Operations
public interface ICommand : MediatR.IRequest<Result>;
public interface ICommand<R> : MediatR.IRequest<Result<R>> where R : IDomainEvent;

// Commands with Multiple Results
public interface ICommandMany<R> : MediatR.IRequest<Result<IEnumerable<R>>> where R : IDomainEvent;

// Queries - Read Operations
public interface IQuery<T> : MediatR.IRequest<Result<T>>;
public interface IQueryMany<R> : MediatR.IRequest<Result<IEnumerable<R>>>;

// Handlers
public interface ICommandHandler<T> : MediatR.IRequestHandler<T, Result>;
public interface ICommandHandler<T, R> : MediatR.IRequestHandler<T, Result<R>>;
public interface ICommandManyHandler<T, R> : MediatR.IRequestHandler<T, Result<IEnumerable<R>>>;
public interface IQueryHandler<T, R> : MediatR.IRequestHandler<T, Result<R>>;
public interface IQueryManyHandler<T, R> : MediatR.IRequestHandler<T, Result<IEnumerable<R>>>;

// Domain Events
public interface IDomainEvent : MediatR.INotification;
public interface IDomainEventHandler<T> : MediatR.INotificationHandler<T>;

// Base Classes
public abstract class AggRoot;
public abstract class Entity;
```

### Command/Query/Event Patterns

| Type | Interface | Handler | Returns |
|------|-----------|---------|---------|
| Command (single) | `ICommand<R>` | `ICommandHandler<T, R>` | `Result<R>` |
| Command (many) | `ICommandMany<R>` | `ICommandManyHandler<T, R>` | `Result<IEnumerable<R>>` |
| Query (single) | `IQuery<T>` | `IQueryHandler<T, T>` | `Result<T>` |
| Query (many) | `IQueryMany<R>` | `IQueryManyHandler<T, R>` | `Result<IEnumerable<R>>` |
| Event | `IDomainEvent` | `IDomainEventHandler<T>` | N/A |

---

## Sub-Audits

This skill includes specialized audit sub-skills:

### 1. Function Audit

**Description:** Audits Azure Functions for proper patterns
**Location:** `audit-functions.md`

Key verification:
- All Functions return `Result<T>` or `Task<Result<T>>`
- Functions delegate to MediatR handlers
- No business logic in Functions
- Configuration from environment variables

### 2. Adapter Audit

**Description:** Audits Adapters implementation
**Location:** `audit-adapters.md`

Key verification:
- Implements Shared interfaces
- Returns Result<T> for all operations
- Uses try-catch with proper error handling
- No business logic (data access only)

### 3. Application Audit

**Description:** Audits Application layer
**Location:** `audit-application.md`

Key verification:
- Contains handlers (CommandHandler, QueryHandler)
- Contains validators (FluentValidation)
- Does NOT reference Adapters
- Uses MediatR for all requests

### 4. Domain Audit

**Description:** Audits Domain layer
**Location:** `audit-domain.md`

Key verification:
- Contains entities and value objects
- Does NOT reference Adapters
- Does NOT reference Application
- Defines repository interfaces (ports)

### 5. Shared Audit

**Description:** Audits Shared layer
**Location:** `audit-shared.md`

Key verification:
- Contains MediatR abstractions
- Contains IState<T> for repositories
- Contains IStream, IStorage interfaces
- Contains Result<T> extensions

### 6. Services Audit

**Description:** Audits Services layer
**Location:** `audit-services.md`

Key verification:
- Configuration from environment variables
- No hardcoded values
- Dependency injection registration
- Startup registration pattern

---

## Audit Checklist

### Layer Dependencies

| Layer | Can Reference | Must NOT Reference |
|-------|---------------|-------------------|
| Functions | Application, Domain, Shared, Adapters | None |
| Application | Domain, Shared | Adapters |
| Domain | Shared (interfaces only) | Application, Adapters |
| Shared | None | Application, Domain |
| Adapters | Shared | Application, Domain |

### Result Pattern

- [ ] All Functions return `Result<T>` or `Task<Result<T>>`
- [ ] No Functions throw exceptions
- [ ] All repositories return `Result<T>`
- [ ] All handlers return `Result<T>`

### MediatR Patterns

- [ ] Commands inherit from `ICommand<R>`
- [ ] Queries inherit from `IQuery<T>`
- [ ] Handlers inherit from appropriate interface
- [ ] Domain events inherit from `IDomainEvent`

### Configuration

- [ ] All configurations from environment variables
- [ ] No sensitive data in code
- [ ] .gitignore excludes sensitive files

---

## Quick Reference

| Check | Expected |
|-------|----------|
| Function return | `Result<T>` or `Task<Result<T>>` |
| Command base | `ICommand<R>` |
| Query base | `IQuery<T>` |
| Handler base | `ICommandHandler<T, R>` or `IQueryHandler<T, R>` |
| Application deps | Domain + Shared only |
| Domain deps | Shared interfaces only |
| Adapter deps | Shared interfaces only |

---

## Additional Resources

For detailed layer audits, see:

- `audit-functions.md` - Function layer audit
- `audit-adapters.md` - Adapter layer audit
- `audit-application.md` - Application layer audit
- `audit-domain.md` - Domain layer audit
- `audit-shared.md` - Shared layer audit
- `audit-services.md` - Services layer audit