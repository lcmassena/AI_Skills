# Hexagonal Architect

A comprehensive skill for designing .NET solutions following Hexagonal Architecture (Ports and Adapters) pattern with Azure Functions, Domain-Driven Design, CQRS, FluentValidation, and FluentResults.

## Use This Skill When

- Designing .NET solutions with Hexagonal Architecture
- Implementing Azure Functions (API and Worker projects)
- Building Domain-Driven Design patterns
- Creating CQRS-based application layers
- Setting up repository patterns with adapters
- Implementing message streaming with Azure Service Bus
- Implementing blob storage with Azure Blob Storage
- Creating new adapters for external services

## Do Not Use This Skill When

- Writing imperative scripts or automation tasks
- Working with frontend frameworks only
- Performing database migrations without architecture context
- Building non-.NET applications

## Quick Start

1. **Understand the layers:**
   - Domain (Core business logic, entities, interfaces)
   - Application (Commands, Queries, Handlers)
   - Adapters (Repository, Service Bus, Storage implementations)
   - API (Azure Functions HTTP triggers)
   - Worker (Azure Functions background jobs)

2. **Follow CQRS pattern:**
   - Queries use `IQuery<T>` and return `Result<T>`
   - Commands use `ICommand<E>` (Event) and return `Result<E>`

3. **Use FluentResults:**
   - Never throw exceptions
   - Always return `Result<T>` or `Result.Fail<T>()`

## Architecture Layers

```
Solution/
├── Core/                    # Core utilities
├── Shared/                  # Shared interfaces (ICommand, IQuery, IHandler)
├── Application/             # Application layer (CQRS)
├── Domain/                 # Domain entities and interfaces
├── Adapters/               # Infrastructure implementations
│   ├── Repository/SQL/     # SQL adapter
│   ├── Stream/             # Service Bus adapter
│   └── Storage/            # Blob Storage adapter
├── API/                   # HTTP endpoints
└── Worker/               # Background jobs
```

## Core Interfaces

```csharp
// From Shared/<Project>.Shared/AggRoot.cs
public interface IQuery<T> : IRequest<Result<T>>;
public interface ICommand<R> : IRequest<Result<R>> where R : IDomainEvent;
public interface ICommandHandler<T, R> : IRequestHandler<T, Result<R>>;
public interface IQueryHandler<T, R> : IRequestHandler<T, Result<R>>;
```

## Example: Query Handler

```csharp
public class BuscarLoteQueryHandler(ILoteRepository repository) : IRequestHandler<BuscarLoteQuery, Result<Lote>>
{
    public async Task<Result<Lote>> Handle(BuscarLoteQuery request, CancellationToken ct)
    {
        try
        {
            var lote = await repository.GetAsync(l => l.CodLoteCia == request.CodLoteCia);
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
```

## Sub-Skills

- **azure-functions**: Azure Functions implementation patterns
- **adapter/repository**: SQL Repository with EF Core
- **adapter/azure-service-bus**: Message publishing
- **adapter/azure-storage**: Blob storage operations
- **devops**: CI/CD pipelines
- **infra**: Azure infrastructure with Bicep
- **audit**: Architecture compliance checks
- **postman**: API testing collections

## Files

```
├── SKILL.md                 # Main skill documentation
├── README.md              # This file
├── templates/              # Code templates
├── examples/              # Working examples
├── azure-functions/       # Azure Functions guide
├── adapter/               # Adapter implementations
├── devops/               # CI/CD guides
├── infra/                # Infrastructure as Code
├── audit/                # Architecture audits
├── postman/              # API testing
└── LICENSE               # MIT License
```

## License

MIT License - See LICENSE file for details.

## Author

- Author: Lucas Massena
- Contact: lucas@massena.com.br
- GitHub: https://github.com/lcmassena