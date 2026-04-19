---
name: azure-functions
description: Use when implementing Azure Functions following Hexagonal Architecture patterns, including HTTP triggers, Timer triggers, Queue triggers, and Worker patterns. Do not use for non-Azure Functions projects.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Azure Functions Implementation Guide

## Overview

This guide covers the implementation patterns for Azure Functions. All functions follow the Hexagonal Architecture pattern, delegating business logic to MediatR handlers.

## Core Principles

1. **Minimal Code in Functions** - Business logic goes in handlers
2. **Use MediatR** - All operations flow through MediatR
3. **Environment Variables** - All configuration from environment
4. **FluentResults** - Handle errors gracefully
5. **OpenAPI** - Document all HTTP endpoints

## Project Structure

The solution contains TWO Azure Functions projects:

| Project | Purpose | Trigger Types |
|---------|---------|---------------|
| API | Publicly exposed HTTP endpoints | HttpTrigger |
| Worker | Background processing jobs | TimerTrigger, QueueTrigger, ServiceBusTrigger |

```
Solution/
├── API/
│   └── <Company>.<Project>.Functions.API/      # HTTP endpoints
│       ├── Program.cs
│       └── *.csproj
├── Worker/
│   └── <Company>.<Project>.Functions.Worker/ # Background jobs
│       ├── Program.cs
│       └── *.csproj
└── *.sln
```

## API Project (HTTP Triggers)

### HTTP Trigger

For REST API endpoints, defined in the API project.

```csharp
[Function("post")]
[OpenApiOperation(
    operationId: null, 
    tags: ["Create Entity"], 
    Summary = "Create new entity")]
[OpenApiRequestBody(
    contentType: "application/json", 
    bodyType: typeof(EntityRequest))]
[OpenApiResponseWithBody(
    statusCode: HttpStatusCode.OK, 
    contentType: "application/json", 
    bodyType: typeof(EntityResponse), 
    Description = "Ok")]
[OpenApiResponseWithBody(
    statusCode: HttpStatusCode.BadRequest, 
    contentType: "application/json", 
    bodyType: typeof(ErrorResponse), 
    Description = "BadRequest")]
[OpenApiSecurity(
    "Bearer", 
    SecuritySchemeType.ApiKey, 
    Description = "Add JWT token", 
    BearerFormat = "JWT", 
    In = OpenApiSecurityLocationType.Header, 
    Name = "Authorization")]

public async Task<IActionResult> Post(
    [HttpTrigger(
        AuthorizationLevel.Anonymous, 
        "post")] HttpRequest req)
{
    // 1. Validate token
    if (!TokenValidate.IsValid(req))
        return new UnauthorizedObjectResult(HttpStatusCode.Unauthorized);

    // 2. Deserialize request
    var contentBody = await new StreamReader(req.Body).ReadToEndAsync();
    var modelRequest = Deserialize<EntityRequest>(contentBody);

    // 3. Validation
    ValidateInput(modelRequest);
    if (_notificationResult.HasNotification())
        return ProcessBadRequest();

    // 4. Configure settings
    ConfigureAppSettings();
    SetConnectionString(modelRequest.tenantId);

    // 5. Delegate to MediatR handler
    var result = await _mediator.Send(modelRequest);

    // 6. Process response
    if (result.Status == EStatus.BusinessRuleInvalid)
        return ProcessBadRequest(result.Status);

    await _uow.CommitAsync();
    return new OkObjectResult(result.Response);
}
```

## Worker Project (Background Jobs)

### Timer Trigger

For scheduled jobs, defined in the Worker project.

```csharp
[Function("ScheduledTask")]
[FixedDelayRetry(5, "00:00:10")]
public async Task RunAsync(
    [TimerTrigger("%TimerSchedule%")] TimerInfo timerInfo, 
    FunctionContext context)
{
    var logger = context.GetLogger(nameof(ScheduledTask));
    logger.LogInformation($"{DateTime.Now:HH:mm:ss}");

    ConfigureAppSettings();
    
    // Delegate to MediatR handler
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
```

**Cron Format:** `{second} {minute} {hour} {day} {month} {day-of-week}`

```
Examples:
"0 */5 * * * *"    - Every 5 minutes
"0 0 0 * * *"     - Daily at midnight
"0 0 9 * * 1-5"   - Weekdays at 9 AM
```

### Queue Trigger

For Azure Queue message processing, defined in the Worker project.

```csharp
[Function("Process")]
public async Task RunAsync(
    [QueueTrigger("%RequestQueueName%", 
        Connection = "AzureWebJobsStorage")] QueueMessage message,
    FunctionContext context)
{
    var logger = context.GetLogger(nameof(FunctionProcess));
    logger.LogInformation($"Queue message received: {message.Body}");

    var command = JsonSerializer.Deserialize<MyCommand>(
        message.Body.ToString());

    var result = await _mediator.Send(command);

    if (result.IsSuccess)
        await _uow.CommitAsync();
}
```

### Service Bus Trigger

For Azure Service Bus message processing, defined in the Worker project.

```csharp
[Function("ServiceBusProcessor")]
public async Task RunAsync(
    [ServiceBusTrigger(
        "%TopicName%", 
        "%SubscriptionName%",
        Connection = "ServiceBusConnection")] ServiceBusReceivedMessage message,
    FunctionContext context)
{
    var logger = context.GetLogger(nameof(ServiceBusProcessor));
    logger.LogInformation($"Service Bus message received");

    var command = JsonSerializer.Deserialize<MyCommand>(
        message.Body.ToString());

    await _mediator.Send(command);
    
    await _uow.CommitAsync();
}
```

## Retry Policies

### Fixed Delay Retry

```csharp
[Function("MyFunction")]
[FixedDelayRetry(5, "00:00:10")]
public async Task RunAsync(...)
{
    // Retries up to 5 times with 10 second delay between retries
}
```

### Exponential Backoff Retry

```csharp
[Function("MyFunction")]
[ExponentialBackoffRetry(
    5,              // Max retry count
    "00:00:10",    // Initial delay
    "00:01:00")]    // Max delay
public async Task RunAsync(...)
{
    // Retries with exponential backoff: 10s, 20s, 40s, 80s, 160s
}
```

## Middleware Pattern

Use a base class for common function logic.

```csharp
public abstract class FunctionBase<T1, T2, T3>
    where T1 : class
    where T2 : class  
    where T3 : class
{
    protected readonly T1 _service1;
    protected readonly T2 _service2;
    protected readonly T3 _mediator;
    protected readonly INotificationResult _notificationResult;

    protected void ConfigureAppSettings()
    {
        AppSettings.Setting = Environment.GetEnvironmentVariable("SettingName")
            ?? throw new InvalidOperationException("SettingName not configured");
    }

    protected void SetConnectionString(string tenantId)
    {
        var connectionString = Environment.GetEnvironmentVariable(
            $"ConnectionStrings_{tenantId}");
        if (connectionString is null)
        {
            _notificationResult.Add(new ErrorResponse(
                "tenantId", "Tenant not found."));
        }
    }

    protected IActionResult ProcessBadRequest()
    {
        var errors = new List<ErrorDetail>();

        foreach (var error in _notificationResult.GetNotifications())
            errors.Add(new(error.Property, error.Message));

        var problemDetails = new ProblemDetails
        {
            Title = "One or more errors occurred.",
            Status = 400,
        };

        problemDetails.Extensions.Add("Errors", errors);
        return new BadRequestObjectResult(problemDetails);
    }
}
```

## Best Practices

### 1. Keep Functions Minimal

Function code should only:
- Validate input
- Deserialize request
- Configure settings
- Delegate to MediatR
- Return response

### 2. Use MediatR for All Operations

Never put business logic in Functions.

```csharp
// Correct
var result = await _mediator.Send(command);
return new OkObjectResult(result);

// Never do this
var repository = new EntityRepository(context);
// Direct database access - wrong!
```

### 3. Configuration from Environment

```csharp
var connectionString = Environment.GetEnvironmentVariable(
    $"ConnectionStrings_{tenantId}")
    ?? throw new InvalidOperationException(
        "ConnectionStrings not configured");
```

### 4. Use IUnitOfWork for Transactions

```csharp
await _uow.CommitAsync();
// or
await _uow.RollbackAsync();
```

### 5. Error Handling

```csharp
if (result.Status == EStatus.BusinessRuleInvalid)
    return ProcessBadRequest(result.Status);

return new OkObjectResult(result.Response);
```

## Dependency Injection

Register dependencies in Program.cs.

```csharp
// Shared
builder.Services.AddSingleton<ISharedInterface, SharedImplementation>();

// Infrastructure
builder.Services.AddScoped<IEntityRepository, EntityRepository>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

// MediatR
builder.Services.AddMediatR(cfg => 
    cfg.RegisterServicesFromAssembly(
        typeof(CreateEntityCommandHandler).Assembly));

// Application
builder.Services.AddValidatorsFromAssembly(
    typeof(CreateEntityCommandValidator).Assembly);
```

## Version Endpoint

Include a version endpoint.

```csharp
[Function("version")]
public IActionResult GetVersion([HttpTrigger(AuthorizationLevel.Anonymous, "get")] HttpRequest req)
{
    string location = Assembly.GetExecutingAssembly().Location;
    var fileVersion = FileVersionInfo.GetVersionInfo(location).FileVersion;
    var productVersion = FileVersionInfo.GetVersionInfo(location).ProductVersion;
    var data = $"Function Name\nVersion: {productVersion}\nBuild: {fileVersion}";
    return new OkObjectResult(data);
}
```

## Anti-Patterns

1. **DO NOT** put business logic in Functions
2. **DO NOT** access database directly in Functions
3. **DO NOT** throw exceptions - use error responses
4. **DO NOT** hardcode configuration

## References

- HTTP Function (API): `API/<Company>.<Project>.Functions.API/FunctionEntities.cs`
- Timer Function (Worker): `Worker/<Company>.<Project>.Functions.Worker/FunctionScheduledTask.cs`
- Queue Function (Worker): `Worker/FunctionQueueProcessor/FunctionQueueProcessor.cs`