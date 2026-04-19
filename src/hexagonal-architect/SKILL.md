---
name: hexagonal-architect
description: Use when designing .NET solutions following Hexagonal Architecture (Ports and Adapters) pattern with Azure Functions, Domain-Driven Design, CQRS, FluentValidation, and FluentResults. Do not use for imperative scripts or general .NET development.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Hexagonal Architect Skill

## Use this skill when

- Designing .NET solutions with Hexagonal Architecture
- Implementing Azure Functions (API and Worker projects)
- Building Domain-Driven Design patterns
- Creating CQRS-based application layers
- Setting up repository patterns with adapters
- Implementing message streaming with Azure Service Bus
- Implementing blob storage with Azure Blob Storage
- Creating new adapters for external services

## Do not use this skill when

- Writing imperative scripts or automation tasks
- Working with frontend frameworks only
- Performing database migrations without architecture context
- Using imperative directly without hexagonal patterns
- Building non-.NET applications

---

## Overview

This skill serves as the architectural guide for .NET development, implementing the Hexagonal Architecture (Ports and Adapters) pattern with a focus on Azure Functions, Domain-Driven Design (DDD), CQRS, FluentValidation, and FluentResults.

---

## Architecture Layers

### 1. Domain Layer (Core/Domain)

The Domain Layer is the core of the application, containing all business logic, entities, value objects, aggregate roots, repository interfaces (ports), and domain events.

**Location:**
- `Domain/<Company>.<Project>.Domain/`

**Responsibilities:**
- Define entities and value objects that represent business concepts
- Define aggregate roots for aggregate consistency
- Define repository interfaces (Ports) - never implement infrastructure here
- Define domain events
- Implement domain validation rules
- Business rules must not depend on any external framework

**Constraints:**
- NO external dependencies (no EF Core, no Azure SDK, no MediatR)
- NO implementation of repositories
- NO database access

**Example - Entity Definition:**

```csharp
namespace <Company>.<Project>.Domain.Entities;

public class EntityName : Entity<EntityName>
{
    public string PropertyOne { get; set; }
    public string PropertyTwo { get; set; }
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; }
    
    public bool IsValid() =>
        !string.IsNullOrEmpty(PropertyOne) && 
        !string.IsNullOrEmpty(PropertyTwo);
}
```

### 2. Application Layer

The Application Layer orchestrates the flow of data, implements use cases, commands, queries, and handlers. This layer sits between the Domain and Infrastructure layers.

**Location:**
- `Application/<Company>.<Project>.Application/`

**Responsibilities:**
- Define Commands (write operations) and Queries (read operations) following CQRS
- Implement handlers using MediatR
- Implement validators using FluentValidation
- Coordinate domain entities to fulfill use cases
- Return results using FluentResults

**Constraints:**
- Only reference Domain layer
- Use MediatR for all request handling
- Use FluentValidation for input validation
- Return Result<T> from FluentResults for all operations

**Example - Query (from Shared):**

```csharp
public interface IQuery<T> : MediatR.IRequest<Result<T>>;
```

**Example - Query Definition:**

```csharp
namespace <Company>.<Project>.Application.Domain.[Lote];

public class Buscar[Lote]Query : IQuery<Lote>
{
    public string Cod[Lote]Cia { get; set; } = string.Empty;
}

public class Listar[Lotes]Query : IQuery<IEnumerable<Lote>>
{
    public string CodCiaSusep { get; set; } = string.Empty;
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 20;
}
```

**Example - Query Handler:**

```csharp
using Confitec.Gepro.Application.Repositories;
using MediatR;

namespace Confitec.Gepro.Application.Domain.Lotes;

public class BuscarLoteQueryHandler(ILoteRepository loteRepository) : IRequestHandler<BuscarLoteQuery, Result<Lote>>
{
    public async Task<Result<Lote>> Handle(BuscarLoteQuery request, CancellationToken cancellationToken)
    {
        try
        {
            var lote = await loteRepository.GetAsync(l => l.CodLoteCia == request.CodLoteCia);
            return lote is null
                ? Result.Fail<Lote>(new Error("Lote não encontrado"))
                : Result.Ok(lote);
        }
        catch (Exception ex)
        {
            return Result.Fail<Lote>(new Error("Erro ao buscar lote").CausedBy(ex));
        }
    }
}

public class ListarLotesQueryHandler(ILoteRepository loteRepository) : IRequestHandler<ListarLotesQuery, Result<IEnumerable<Lote>>>
{
    public async Task<Result<IEnumerable<Lote>>> Handle(ListarLotesQuery request, CancellationToken cancellationToken)
    {
        try
        {
            var lotes = await loteRepository.GetManyAsync(l => l.CodCiaSusep == request.CodCiaSusep);
            return Result.Ok(lotes);
        }
        catch (Exception ex)
        {
            return Result.Fail<IEnumerable<Lote>>(new Error("Erro ao listar lotes").CausedBy(ex));
        }
    }
}
```

**Example - Command Handler:**

```csharp
public class CreateEntityCommandHandler : 
    IRequestHandler<CreateEntityCommand, Result<EntityResponse>>
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

**Example - Validator:**

```csharp
namespace <Company>.<Project>.Application.Validators;

public class CreateEntityCommandValidator : 
    AbstractValidator<CreateEntityCommand>
{
    public CreateEntityCommandValidator()
    {
        RuleFor(x => x.PropertyOne)
            .NotEmpty()
            .WithMessage("PropertyOne is required");
            
        RuleFor(x => x.PropertyTwo)
            .NotEmpty()
            .WithMessage("PropertyTwo is required")
            .MaximumLength(50);
    }
}
```

### 3. Infrastructure/Adapters Layer

The Infrastructure layer implements the interfaces (ports) defined in the Domain layer. This includes database repositories, Azure service clients, and external API integrations.

**Location:**
- `Adapters/Repository/SQL/`
- `Adapters/Stream/AzureServiceBus/`
- `Adapters/Storage/AzureStorage/`

**Sub-Components:**

#### 3.1 SQL Adapter (Repository)

Implements database persistence using Entity Framework Core. All repositories inherit from SqlRepository<T> base class.

**Key Characteristics:**
- Generic base class handling CRUD operations
- Returns Result<T> for all operations
- Uses try-catch with proper error handling
- Never throws exceptions - returns Result.Fail() instead

**Example - Repository Implementation:**

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

#### 3.2 Azure Service Bus Adapter (Stream)

Implements message publishing using Azure Service Bus for asynchronous communication.

**Key Characteristics:**
- Implements IStream interface (port)
- Supports scheduled messages
- Supports session-based messaging
- Uses IDomainEvent interface

**Example - Service Bus Implementation:**

```csharp
public class AzureServiceBusService : IStream
{
    public async Task SendEventAsync<T>(T notification, string topicName) 
        where T : IDomainEvent
    {
        var sender = Get(topicName);
        var message = new ServiceBusMessage(
            JsonConvert.SerializeObject(notification));
        await sender.SendMessageAsync(message);
    }
}
```

#### 3.3 Azure Storage Adapter

Implements blob storage operations for file management in Azure Blob Storage.

**Key Characteristics:**
- Implements IStorage interface (port)
- Supports idempotent writes
- Adds invocation metadata
- Handles various input types (Stream, string, object)

### 4. API Layer (Azure Functions)

The API layer serves as the entry point for external requests. This project contains two Azure Functions projects:

**Projects:**

| Project | Purpose | Trigger Types |
|---------|---------|---------------|
| API | Publicly exposed HTTP endpoints | HttpTrigger |
| Worker | Background processing jobs | TimerTrigger, QueueTrigger, ServiceBusTrigger |

**Location:**
- `API/<Company>.<Project>.Functions.API/`
- `Worker/<Company>.<Project>.Functions.Worker/`

**Responsibilities:**
- Receive HTTP requests or Timer/Queue triggers
- Validate input using model validation
- Delegate to MediatR handlers
- Return appropriate HTTP responses
- Configure OpenAPI annotations for documentation

**Constraints:**
- Business logic MUST be in handlers (never in Functions)
- Functions should have minimal code (max ~50 lines recommended)
- Configuration must come from environment variables
- Use IUnitOfWork for transactional operations

**Example - HTTP Trigger Function (API Project):**

```csharp
public class FunctionEntities : 
    FunctionBase<IService, IUnitOfWork, IMediator, INotificationResult>
{
    [Function("post")]
    [OpenApiOperation(operationId: null, tags: ["Create Entity"], 
        Summary = "Create new entity")]
    [OpenApiRequestBody(contentType: "application/json", 
        bodyType: typeof(EntityRequest))]
    [OpenApiResponseWithBody(statusCode: HttpStatusCode.OK, 
        contentType: "application/json", 
        bodyType: typeof(EntityResponse), Description = "Ok")]

    public async Task<IActionResult> Post(
        [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequest req)
    {
        if (!TokenValidate.IsValid(req))
            return new UnauthorizedObjectResult(HttpStatusCode.Unauthorized);

        var contentBody = await new StreamReader(req.Body).ReadToEndAsync();
        var modelRequest = DeserializeRequest<EntityRequest>(contentBody);

        ValidateInput(modelRequest);
        if (_notificationResult.HasNotification())
            return ProcessBadRequest();

        ConfigureAppSettings();
        SetConnectionString(modelRequest.tenantId);

        var result = await _mediator.Send(modelRequest);

        if (result.Status == EStatus.BusinessRuleInvalid)
            return ProcessBadRequest(result.Status);

        await _uow.CommitAsync();

        return new OkObjectResult(result.Response);
    }
}
```

**Example - Timer Trigger Function (Worker Project):**

```csharp
public class FunctionScheduledTask : 
    FunctionBase<IService, IMediator, IUnitOfWork>
{
    [Function("ScheduledTask")]
    [FixedDelayRetry(5, "00:00:10")]
    public async Task RunAsync(
        [TimerTrigger("%TimerSchedule%")] TimerInfo timerInfo, 
        FunctionContext context)
    {
        var logger = context.GetLogger(nameof(ScheduledTask));
        logger.LogInformation($"{DateTime.Now:HH:mm:ss}");

        ConfigureAppSettings();
        
        var itemsToProcess = await _mediator.Send(
            new GetItemsToProcessRequest());

        foreach (var item in itemsToProcess)
        {
            var result = await _mediator.Send(new ProcessItemCommand
            {
                Item = item
            });

            await _uow.CommitAsync();
        }
    }
}
```

**Supported Triggers:**
- HttpTrigger - HTTP endpoints (API project)
- TimerTrigger - Scheduled jobs (Worker project)
- QueueTrigger - Azure Queue messages (Worker project)
- ServiceBusTrigger - Service Bus messages (Worker project)

### 5. Shared Layer

The Shared layer contains code that is shared across all projects. This is a READ-ONLY layer by design - any change impacts ALL projects in the solution.

**Location:**
- `Shared/<Company>.<Project>.Shared/`

**CRITICAL - Shared Layer Rules:**
1. This project is READ-ONLY for most teams
2. Any change can break multiple production Functions
3. Changes require thorough testing
4. Avoid adding dependencies
5. Keep interfaces stable
6. Document all new additions

### 6. Core Layer

The Core layer contains shared utilities and cross-cutting concerns used throughout the solution.

**Location:**
- `Core/<Company>.<Project>.Core/`

---

## Folder Structure

```
Solution Root/
├── Core/
│   └── <Company>.<Project>.Core/                    # Core utilities
├── Shared/
│   └── <Company>.<Project>.Shared/                  # Shared interfaces
├── Application/
│   └── <Company>.<Project>.Application/           # Application layer
├── Domain/
│   └── <Company>.<Project>.Domain/                # Domain entities
├── Adapters/
│   ├── Repository/
│   │   └── SQL/                          # SQL adapter
│   ├── Stream/
│   │   └── AzureServiceBus/              # Service Bus adapter
│   └── Storage/
│       └── AzureStorage/                 # Azure Storage adapter
├── API/
│   └── <Company>.<Project>.Functions.API/  # HTTP endpoints
├── Worker/
│   └── <Company>.<Project>.Functions.Worker/ # Background jobs
└── <Company>.<Project>.sln
```

---

## Best Practices

### 0. Core Interfaces (Shared/AggRoot.cs)

All interfaces are defined in the Shared project and used across all layers.

```csharp
// AggRoot.cs - Core interfaces for CQRS

public interface IDomainEvent : MediatR.INotification;
public interface IDomainEventHandler<T> : MediatR.INotificationHandler<T> where T : IDomainEvent;

public interface ICommand : MediatR.IRequest<Result>;
public interface ICommand<R> : MediatR.IRequest<Result<R>> where R : IDomainEvent;
public interface ICommandMany<R> : MediatR.IRequest<Result<IEnumerable<R>>> where R : IDomainEvent;

public interface ICommandHandler<T> : MediatR.IRequestHandler<T, Result> where T : ICommand;
public interface ICommandHandler<T, R> : MediatR.IRequestHandler<T, Result<R>> where T : ICommand<R> where R : IDomainEvent;
public interface ICommandManyHandler<T, R> : MediatR.IRequestHandler<T, Result<IEnumerable<R>>> where T : ICommandMany<R> where R : IDomainEvent;

public interface IQuery<T> : MediatR.IRequest<Result<T>>;
public interface IQueryMany<R> : MediatR.IRequest<Result<IEnumerable<R>>>;

public interface IQueryHandler<T, R> : MediatR.IRequestHandler<T, Result<R>> where T : IQuery<R> where R : class;
public interface IQueryManyHandler<T, R> : MediatR.IRequestHandler<T, Result<IEnumerable<R>>> where T : IQueryMany<R> where R : class;
```

### 1. FluentResults Usage

Always use FluentResults for return types in Application and Infrastructure layers.

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
        return Result.Fail(new Error(ex.Message).CausedBy(ex));
    }
}

// Never throw exceptions in repository
// Never return null - always use Result<T>
```

### 2. FluentValidation Usage

Always validate inputs using FluentValidation in the Application layer.

### 3. CQRS Pattern

Separate read and write operations using Commands and Queries. Uses IQuery<T> and ICommand<R> from Shared.

**Query Definition:**
```csharp
// Application/Domain/<Entity>/<Entity>Queries.cs
public class Buscar<Entity>Query : IQuery<<Entity>>
{
    public string Cod<Entity>Cia { get; set; } = string.Empty;
}

public class Listar<Entities>Query : IQuery<IEnumerable<<Entity>>>
{
    public string CodCiaSusep { get; set; } = string.Empty;
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 20;
}
```

**Query Handler:**
```csharp
// Application/Domain/<Entity>/<Entity>QueryHandlers.cs
public class Buscar<Entity>QueryHandler(I<Entity>Repository repository) : IRequestHandler<Buscar<Entity>Query, Result<<Entity>>>
{
    public async Task<Result<<Entity>>> Handle(Buscar<Entity>Query request, CancellationToken cancellationToken)
    {
        try
        {
            var entity = await repository.GetAsync(e => e.Cod<Entity>Cia == request.Cod<Entity>Cia);
            return entity is null
                ? Result.Fail<<Entity>>(new Error("<Entity> não encontrado"))
                : Result.Ok(entity);
        }
        catch (Exception ex)
        {
            return Result.Fail<<Entity>>(new Error("Erro ao buscar <Entity>").CausedBy(ex));
        }
    }
}
```

**Command with Event:**
```csharp
// Application/Domain/<Entity>/<Entity>GravadoEvent.cs - Event (output)
public class <Entity>GravadoEvent : ArquivoGravadoEvent, IDomainEvent { }

// Application/Domain/<Entity>/<Entity>Command.cs - Command (input)
public class <Entity>CommandRequest : ICommand<<Entity>GravadoEvent>
{
    public string InvocationId { get; set; } = string.Empty;
    public RemessaRequest remessa { get; set; } = new();
    public List<<Entity>Request> <entity>s { get; set; } = new();
}
```

### 4. Dependency Injection

Register dependencies in the correct order respecting the hexagonal architecture:

1. Domain interfaces (ports) - no registration needed
2. Application handlers - automatically by MediatR
3. Infrastructure implementations (adapters)
4. Azure Functions

### 5. Configuration

Always load configuration from environment variables.

---

## Testing Guidelines

When testing, follow these principles:

1. **Domain Layer** - Unit tests for business rules
2. **Application Layer** - Test handlers with mocked repositories
3. **Infrastructure Layer** - Integration tests with test databases
4. **API Layer** - Integration tests with in-memory pipelines

---

## Anti-Patterns to Avoid

1. **DO NOT** put business logic in Azure Functions
2. **DO NOT** throw exceptions in repositories - use Result.Fail()
3. **DO NOT** access database directly in Functions
4. **DO NOT** implement ports in Domain layer
5. **DO NOT** add circular dependencies between layers
6. **DO NOT** modify Shared layer without impact analysis

---

## Sub-Skills Index

This skill includes the following specialized sub-skills:

### 1. Repository Adapter

**Description:** Database persistence implementation using Entity Framework Core with repository pattern
**Location:** `adapter/repository/SKILL.md`

Key topics:
- SqlRepository<T> base class
- CRUD operations with FluentResults
- Entity mapping with EF Core
- Unit of Work pattern

### 2. Azure Service Bus Adapter

**Description:** Asynchronous message publishing using Azure Service Bus
**Location:** `adapter/azure-service-bus/SKILL.md`

Key topics:
- Topic publishing
- Session-based messages
- Scheduled message delivery
- Custom properties

### 3. Azure Storage Adapter

**Description:** Blob storage operations using Azure Blob Storage
**Location:** `adapter/azure-storage/SKILL.md`

Key topics:
- Idempotent writes
- Multiple input types (Stream, string, object)
- Metadata injection
- Container management

### 4. Azure Functions Implementation

**Description:** Azure Functions development patterns and best practices
**Location:** `azure-functions/SKILL.md`

Key topics:
- API vs Worker projects
- HTTP triggers and routing
- Timer triggers and scheduling
- Retry policies
- Dependency injection

### 5. DevOps

**Description:** CI/CD pipelines with GitHub Actions using Build Once Deploy Many strategy
**Location:** `devops/SKILL.md`

Key topics:
- Build pipeline (Continuous Integration)
- Multi-environment deployment (DEV → QA → PROD)
- Environment configuration and secrets
- Custom deployment targets

### 6. Adapter Creator

**Description:** Creates new adapter projects following the Hexagonal Architecture pattern
**Location:** `adapter-creator/SKILL.md`

Key topics:
- Project structure and naming conventions
- .csproj file template
- Service implementation patterns
- Startup registration (IStartupRegister)
- Settings configuration

### 7. Postman

**Description:** Postman collections for Azure Functions APIs
**Location:** `postman/SKILL.md`

Key topics:
- Collection structure (.postman/ folders)
- Environment configuration (Local, Dev, QA, Prod)
- Request file format (YAML)
- Agent for automatic API discovery

**Agent:** `postman/agent.md` - Agent behavior for automatic Postman generation

### 8. Infrastructure as Code

**Description:** Azure infrastructure deployment using Bicep with CAF naming conventions
**Location:** `infra/SKILL.md`

Key topics:
- Bicep modules per resource type
- Cloud Adoption Framework naming conventions
- Multi-environment parameters
- Service Bus, Storage, Functions modules
- Parameter files (.bicepparam)

**Additional Resources:**
- `infra/deploy.md` - Manual CLI/Portal deployment
- `infra/githubaction.md` - GitHub Actions pipeline
- `infra/azuredevops.md` - Azure DevOps pipeline
- `devops/iac.md` - IaC with GitHub Actions

### 9. Architecture Audit

**Description:** Comprehensive audit for hexagonal architecture compliance
**Location:** `audit/SKILL.md`

Key topics:
- Layer dependency verification
- MediatR abstractions (ICommand, IQuery, ICommandHandler)
- Result<T> return pattern
- Configuration from environment variables

**Layer Audits:**
- `audit/audit-functions.md` - Functions layer audit
- `audit/audit-adapters.md` - Adapters layer audit
- `audit/audit-application.md` - Application layer audit
- `audit/audit-domain.md` - Domain layer audit
- `audit/audit-shared.md` - Shared layer audit
- `audit/audit-services.md` - Services layer audit

---

## Additional Resources

For templates and examples, see:

- `templates/` - Ready-to-use code templates
- `examples/` - Complete working examples
- `adapter/repository/` - Repository implementation details
- `adapter/azure-service-bus/` - Service Bus implementation details
- `adapter/azure-storage/` - Storage implementation details

---

## Quick Reference

| Component | Interface | Implementation |
|-----------|-----------|---------------|
| SQL | IState<T> | SqlRepository<T> |
| Service Bus | IStream | AzureServiceBusService |
| Storage | IStorage | AzureStorageService |
| Functions | N/A | Function classes |
| Commands | IRequest<T> | Command classes |
| Queries | IRequest<T> | Query classes |
| Validators | AbstractValidator<T> | Validator classes |