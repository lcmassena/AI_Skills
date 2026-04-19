---
name: hexagonal-templates
description: Code templates for Hexagonal Architecture implementation including entities, commands, queries, handlers, validators, and Azure Functions.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

This directory contains ready-to-use code templates for creating new components following the Hexagonal Architecture pattern.

## Available Templates

### 1. Entity Template

**File:** `entity-template.cs`

**Purpose:** Domain entity definition

**Placeholders to replace:**
- `[EntityName]` - Entity name in PascalCase (e.g., `Lote`, `Customer`)
- `[Company]` - Company name
- `[Project]` - Project name

```csharp
namespace <Company>.<Project>.Domain.Entities;

public class [EntityName]
{
    public long Id { get; set; }
    public string Cod[EntityName]Cia { get; set; } = string.Empty;
    public string CodCiaSusep { get; set; } = string.Empty;
    public DateTime Data { get; set; }
}
```

### 2. Query Template

**File:** `query-template.cs`

**Purpose:** Query definition (read operation)

**Placeholders to replace:**
- `[EntityName]` - Entity name (e.g., `Lote`)

```csharp
// Application/<Company>.<Project>.Application/Domain/<Entity>/<Entity>Queries.cs
public class Buscar[EntityName]Query : IQuery<[EntityName]>
{
    public string Cod[EntityName]Cia { get; set; } = string.Empty;
}

public class Listar[EntityName]sQuery : IQuery<IEnumerable<[EntityName]>>
{
    public string CodCiaSusep { get; set; } = string.Empty;
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 20;
}
```

### 3. Query Handler Template

**File:** `query-handler-template.cs`

**Purpose:** Query handler (read operation)

**Placeholders to replace:**
- `[QueryName]` - Query name (e.g., `BuscarLoteQuery`)
- `[EntityName]` - Entity name (e.g., `Lote`)
- `[EntityName]Repository` - Repository interface

```csharp
namespace <Company>.<Project>.Application.Domain.[EntityName]s;

public class [QueryName]Handler(I[EntityName]Repository repository) : 
    IRequestHandler<[QueryName], Result<[EntityName]>>
{
    public async Task<Result<[EntityName]>> Handle(
        [QueryName] request, 
        CancellationToken cancellationToken)
    {
        try
        {
            var entity = await repository.GetAsync(e => e.Cod[EntityName]Cia == request.Cod[EntityName]Cia);
            return entity is null
                ? Result.Fail<[EntityName]>(new Error("[EntityName] não encontrado"))
                : Result.Ok(entity);
        }
        catch (Exception ex)
        {
            return Result.Fail<[EntityName]>(new Error("Erro ao buscar [EntityName]").CausedBy(ex));
        }
    }
}
```

### 4. Command with Event Template

**File:** `command-event-template.cs`

**Purpose:** Command that writes to storage and dispatches event

**Placeholders to replace:**
- `[EntityName]` - Entity name (e.g., `RiscoEmissao`)
- `[ServiceName]` - Service name for logging

```csharp
// Event - Output of the command
public class [EntityName]GravadoEvent : ArquivoGravadoEvent, IDomainEvent
{
    public [EntityName]GravadoEvent() { }
}

// Command - Input request
public class [EntityName]CommandRequest : ICommand<[EntityName]GravadoEvent>
{
    public string InvocationId { get; set; } = string.Empty;
    public RemessaRequest remessa { get; set; } = new();
    public List<[EntityName]Request> [entity]s { get; set; } = new();
}

// Handler with base class
public class [EntityName]Handler
    : GravadorDeArquivoHandler<[EntityName]CommandRequest, [EntityName]GravadoEvent>, 
    ICommandHandler<[EntityName]CommandRequest, [EntityName]GravadoEvent>
{
    public [EntityName]Handler(IStream stream, IStorage storage, ILogger<[EntityName]CommandRequest> logger) : 
        base(stream, storage, logger) { }

    private static readonly string _CONTAINER_NAME = "[entity]";
    private static readonly string _QUEUENAME = "[entity]gravado";
    private static readonly string _SERVICE_NAME = "[ServiceName]";

    public async Task<Result<[EntityName]GravadoEvent>> Handle([EntityName]CommandRequest request, CancellationToken cancellationToken)
    {
        return await WriteToStorageAndDispatchEvent(request, _SERVICE_NAME, request.InvocationId, _CONTAINER_NAME, _QUEUENAME, cancellationToken);
    }
}
```

### 5. Repository Interface Template

**File:** `repository-interface-template.cs`

**Purpose:** Repository port interface

**Placeholders to replace:**
- `[EntityName]` - Entity name
- `[IEntityRepository]` - Interface name with 'I' prefix

```csharp
namespace <Company>.<Project>.Domain.Repositories;

public interface I[EntityName]Repository : IState<[EntityName]>
{
    Task<Result<[EntityName]>> GetAsync(Expression<Func<[EntityName], bool>> predicate);
    Task<Result<IEnumerable<[EntityName]>>> GetManyAsync(Expression<Func<[EntityName], bool>> predicate);
    Task<Result<[EntityName]>> AddAsync([EntityName] entity);
}
```

### 6. Repository Implementation Template

**File:** `repository-implementation-template.cs`

**Purpose:** SQL Repository implementation

```csharp
namespace <Company>.<Project>.Adapters.Repository.SQL.Repositories;

public class [EntityName]Repository : SqlRepository<[EntityName]>, I[EntityName]Repository
{
    public [EntityName]Repository(DataContext context) : base(context) { }
    
    public async Task<Result<[EntityName]>> GetAsync(Expression<Func<[EntityName], bool>> predicate)
    {
        try
        {
            var entity = await _dbSet.FirstOrDefaultAsync(predicate);
            return Result.Ok(entity);
        }
        catch (Exception ex)
        {
            return Result.Fail<[EntityName]>(new Error(ex.Message).CausedBy(ex));
        }
    }
}
```

### 2. Repository Interface Template

**File:** `repository-interface-template.cs`

**Purpose:** Repository port interface

**Use when:** Creating a new repository interface

**Placeholders to replace:**
- `[EntityName]` - Entity name
- `[IEntityRepository]` - Interface name with 'I' prefix

```csharp
namespace <Company>.<Project>.Domain.Interfaces;

public interface I[EntityName]Repository : IState<[EntityName]>
{
    Task<Result<[EntityName]>> AddAsync([EntityName] entity);
    Task<Result<[EntityName]>> GetByIdAsync([IdType] id);
    Task<Result<[EntityName]>> UpdateAsync([IdType] id, [EntityName] entity);
    Task<Result> DeleteAsync([IdType] id);
}
```

### 3. Repository Implementation Template

**File:** `repository-implementation-template.cs`

**Purpose:** Repository implementation with SQL

**Use when:** Implementing a new repository

**Placeholders to replace:**
- `[EntityName]` - Entity name
- `[IEntityRepository]` - Interface name

```csharp
namespace <Company>.<Project>.Adapters.Repository.SQL.Repositories;

public class [EntityName]Repository : SqlRepository<[EntityName]>, I[EntityName]Repository
{
    public [EntityName]Repository(DataContext context) : base(context) { }

    public async Task<Result<[EntityName]>> AddAsync([EntityName] entity)
    {
        try
        {
            await _dbSet.AddAsync(entity);
            await _context.SaveChangesAsync();
            return Result.Ok(entity);
        }
        catch (Exception ex)
        {
            return Result.Fail<[EntityName]>(new Error(ex.Message).CausedBy(ex));
        }
    }
}
```

### 4. Command Handler Template

**File:** `command-handler-template.cs`

**Purpose:** CQRS command handler

**Use when:** Creating a write operation handler

**Placeholders to replace:**
- `[CommandName]` - Command name (e.g., `CreateCustomerCommand`)
- `[EntityName]` - Entity name
- `[Response]` - Response DTO name

```csharp
namespace <Company>.<Project>.Application.Domain.[EntityName]s;

public class [CommandName]Handler : 
    IRequestHandler<[CommandName], Result<[Response]>>
{
    private readonly I[EntityName]Repository _repository;
    private readonly IUnitOfWork _uow;

    public async Task<Result<[Response]>> Handle(
        [CommandName] request, 
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

### 5. Query Handler Template

**File:** `query-handler-template.cs`

**Purpose:** CQRS query handler

**Use when:** Creating a read operation handler

**Placeholders to replace:**
- `[QueryName]` - Query name (e.g., `BuscarLoteQuery`)
- `[EntityName]` - Entity name (e.g., `Lote`)
- `[EntityName]Repository` - Repository interface

```csharp
namespace <Company>.<Project>.Application.Domain.[EntityName]s;

public class [QueryName]Handler(I[EntityName]Repository repository) : 
    IRequestHandler<[QueryName], Result<[EntityName]>>
{
    public async Task<Result<[EntityName]>> Handle(
        [QueryName] request, 
        CancellationToken cancellationToken)
    {
        try
        {
            var entity = await repository.GetAsync(e => e.Cod[EntityName]Cia == request.Cod[EntityName]Cia);
            return entity is null
                ? Result.Fail<[EntityName]>(new Error("[EntityName] não encontrado"))
                : Result.Ok(entity);
        }
        catch (Exception ex)
        {
            return Result.Fail<[EntityName]>(new Error("Erro ao buscar [EntityName]").CausedBy(ex));
        }
    }
}
```

### 7. Validator Template

**File:** `validator-template.cs`

**Purpose:** FluentValidation validator

**Placeholders to replace:**
- `[CommandName]` - Command name to validate
- `[Property1]` - Property to validate

```csharp
namespace <Company>.<Project>.Application.Validators;

public class [CommandName]Validator : AbstractValidator<[CommandName]>
{
    public [CommandName]Validator()
    {
        RuleFor(x => x.remessa)
            .NotNull()
            .WithMessage("Remessa é obrigatória");

        RuleFor(x => x.remessa!.codCiaSusep)
            .SetValidator(new SeguradoraPermitidaValidator());

        RuleFor(x => x.remessa!.codRemessaCia)
            .NotEmpty()
            .WithMessage("Código da remessa é obrigatório");

        RuleFor(x => x.[entity]s)
            .NotNull()
            .WithMessage("[Entity]s são obrigatórios");

        RuleFor(x => x.[entity]s!)
            .Must(itens => itens != null && itens.Any())
            .WithMessage("Deve conter pelo menos um [entity]");
    }
}
```

### 7. Azure Function Template (API)

**File:** `function-api-template.cs`

**Purpose:** HTTP trigger Azure Function

**Use when:** Creating new HTTP endpoint

**Placeholders to replace:**
- `[FunctionName]` - Function class name
- `[Verb]` - HTTP verb (get, post, put, delete)
- `[RequestType]` - Request DTO name
- `[ResponseType]` - Response DTO name

```csharp
namespace <Company>.<Project>.Functions.API;

public class [FunctionName] : 
    FunctionBase<IService, IUnitOfWork, IMediator, INotificationResult>
{
    [Function("[Verb]")]
    [OpenApiOperation(operationId: null, tags: ["[Entity]"], 
        Summary = "[Description]")]
    [OpenApiRequestBody(contentType: "application/json", 
        bodyType: typeof([RequestType]))]
    [OpenApiResponseWithBody(statusCode: HttpStatusCode.OK, 
        contentType: "application/json", 
        bodyType: typeof([ResponseType]), Description = "Ok")]

    public async Task<IActionResult> [Verb](
        [HttpTrigger(AuthorizationLevel.Anonymous, "[verb]")] HttpRequest req)
    {
        if (!TokenValidate.IsValid(req))
            return new UnauthorizedObjectResult(HttpStatusCode.Unauthorized);

        var contentBody = await new StreamReader(req.Body).ReadToEndAsync();
        var modelRequest = Deserialize<[RequestType]>(contentBody);

        ValidateInput(modelRequest);
        if (_notificationResult.HasNotification())
            return ProcessBadRequest();

        var result = await _mediator.Send(modelRequest);

        if (result.Status == EStatus.BusinessRuleInvalid)
            return ProcessBadRequest(result.Status);

        await _uow.CommitAsync();
        return new OkObjectResult(result.Response);
    }
}
```

### 8. Azure Function Template (Worker)

**File:** `function-worker-template.cs`

**Purpose:** Timer/Queue trigger Azure Function

**Use when:** Creating background job

**Placeholders to replace:**
- `[FunctionName]` - Function class name
- `[TimerSchedule]` - Cron expression

```csharp
namespace <Company>.<Project>.Functions.Worker;

public class [FunctionName] : 
    FunctionBase<IService, IMediator, IUnitOfWork>
{
    [Function("[FunctionName]")]
    [FixedDelayRetry(5, "00:00:10")]
    public async Task RunAsync(
        [TimerTrigger("%[TimerSchedule]%")] TimerInfo timerInfo, 
        FunctionContext context)
    {
        var logger = context.GetLogger(nameof([FunctionName]));
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

### 9. Adapter Template

**File:** `adapter-template.csproj`

**Purpose:** New adapter project

**Use when:** Creating a new adapter

**Placeholders to replace:**
- `[Category]` - Adapter category (Repository, Stream, Storage)
- `[ServiceName]` - Service name (e.g., Postgres, MongoDb)
- `[Package]` - NuGet package name
- `[Version]` - Package version

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <AssemblyTitle><Company>.<Project>.Adapters.[Category].[ServiceName]</AssemblyTitle>
    <AssemblyName><Company>.<Project>.Adapters.[Category].[ServiceName]</AssemblyName>
    <RootNamespace><Company>.<Project>.Adapters.[Category].[ServiceName]</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="[Package]" Version="[Version]" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\..\Shared\<Company>.<Project>.Shared.csproj" />
  </ItemGroup>

</Project>
```

---

## How to Use Templates

### Step 1: Copy the Template

Copy the template file you need to your project directory.

### Step 2: Replace Placeholders

Find and replace all placeholders (text in square brackets):

**Example:** Creating a Customer entity

- `[EntityName]` → `Customer`
- `[Company]` → `MyCompany`
- `[Project]` → `MyProject`
- `[Property1]` → `Name`
- `[Property2]` → `Email`

### Step 3: Customize

Modify the template to fit your specific needs:
- Add/remove properties
- Adjust validation rules
- Update method signatures

### Step 4: Register Dependencies

Make sure to register the new components in Program.cs:

```csharp
// Repository
builder.Services.AddScoped<ICustomerRepository, CustomerRepository>();

// MediatR
builder.Services.AddMediatR(cfg => 
    cfg.RegisterServicesFromAssembly(typeof(CreateCustomerCommandHandler).Assembly));
```

---

## Quick Reference

For complete implementation guidance, see:

- `SKILL.md` - Main skill documentation
- `examples/` - Complete working examples
- `adapter/repository/` - Repository patterns
- `adapter/azure-service-bus/` - Service Bus patterns
- `adapter/azure-storage/` - Storage patterns
- `azure-functions/` - Function patterns