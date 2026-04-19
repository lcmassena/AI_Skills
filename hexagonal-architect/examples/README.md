---
name: hexagonal-examples
description: Complete working examples for Hexagonal Architecture following the actual pattern discovered in Confitec.Gepro.Application.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Hexagonal Architecture Examples

Complete working examples following the pattern discovered in Confitec.Gepro.Application.

## Pattern Sources

These examples are based on real code from:
- `Confitec.Gepro.Application.Domain.Lotes.LoteQueryHandlers.cs`
- `Confitec.Gepro.Application.Domain.Risco.Emissao.RiscoEmissaoHandler.cs`
- `Confitec.Gepro.Functions.Shared.AggRoot.cs`

---

## Example 1: Query (READ Operation)

### Query Definition

```csharp
// Application/<Company>.<Project>.Application/Domain/Lotes/LoteQueries.cs

namespace Confitec.Gepro.Application.Domain.Lotes;

public class LoteQuery : IQuery<Lote>
{
    public string CodLoteCia { get; set; } = string.Empty;
    public string CodCiaSusep { get; set; } = string.Empty;
}

public class BuscarLoteQuery : IQuery<Lote>
{
    public string CodLoteCia { get; set; } = string.Empty;
}

public class ListarLotesQuery : IQuery<IEnumerable<Lote>>
{
    public string CodCiaSusep { get; set; } = string.Empty;
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 20;
}
```

### Query Handler

```csharp
// Application/<Company>.<Project>.Application/Domain/Lotes/LoteQueryHandlers.cs

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

### Repository Interface (Port)

```csharp
// Application/<Company>.<Project>.Application/Repositories/ILoteRepository.cs

using Confitec.Gepro.Application.Domain;

namespace Confitec.Gepro.Application.Repositories;

public interface ILoteRepository : IState<Lote>
{
    Task<Result<Lote>> BuscarLote(string codLoteCia);
    Task<Result<RetornoLote>> AtualizarLote(long? id, long? qtdRegistrosRecebidos);
}
```

---

## Example 2: Command with Event (WRITE Operation)

### Event (Output)

```csharp
// Application/<Company>.<Project>.Application/Common/Events/ArquivoGravadoEvent.cs

namespace Confitec.Gepro.Application.Common.Events
{
    public class ArquivoGravadoEvent : IDomainEvent
    {
        public string Container { get; init; } = string.Empty;
        public string FileName { get; init; } = string.Empty;
        public string BlobPath => $"{Container}/{FileName}";
        public Uri BlobUrl { get; init; } = null;
        public string ProtocolNumber { get; init; } = string.Empty;
        public DateTime ProtocolDateTime { get; init; } = DateTime.Now;
    }
}
```

### Event Inheritance

```csharp
// Application/<Company>.<Project>.Application/Domain/Risco/Emissao/RiscoEmissaoGravadoEvent.cs

using Confitec.Gepro.Application.Common.Events;
using Confitec.Gepro.Functions.Shared.Common;

namespace Confitec.Gepro.Application.Domain.Risco.Emissao;

public class RiscoEmissaoGravadoEvent : ArquivoGravadoEvent
{
    public RiscoEmissaoGravadoEvent() { }
}
```

### Command Request

```csharp
// Application/<Company>.<Project>.Application/Domain/Risco/Emissao/RiscoEmissaoGravadoEvent.cs (same file)

public class RiscoRequest
{
    public string codCiaSusep { get; set; } = string.Empty;
    public int? codSucursalEmissora { get; set; }
    public string codGrupoRamoEmissao { get; set; } = string.Empty;
    public string codRamoEmissao { get; set; } = string.Empty;
    public string numApolice { get; set; } = string.Empty;
    public string numEndosso { get; set; } = string.Empty;
    public string? numCertificado { get; set; }
}

public class RiscoEmissaoCommandRequest : ICommand<RiscoEmissaoGravadoEvent>
{
    public string InvocationId { get; set; } = string.Empty;
    public RemessaRequest remessa { get; set; } = new();
    public List<RiscoRequest> riscos { get; set; } = new();
}
```

### Base Handler (GravadorDeArquivoHandler)

```csharp
// Application/<Company>.<Project>.Application/Common/Handler/GravadorDeArquivoHandler.cs

using Confitec.Gepro.Functions.Shared.Common;
using Confitec.Gepro.Functions.Shared.Storage;
using Confitec.Gepro.Functions.Shared.Stream;
using FluentResults;

namespace Confitec.Gepro.Application.Common.Handler;

public abstract class GravadorDeArquivoHandler<TRequest, TEvent> 
    where TRequest : ICommand<TEvent>
    where TEvent : ArquivoGravadoEvent, IDomainEvent
{
    protected readonly IStream _stream;
    protected readonly IStorage _storage;
    protected readonly ILogger _logger;

    protected GravadorDeArquivoHandler(IStream stream, IStorage storage, ILogger logger)
    {
        _stream = stream;
        _storage = storage;
        _logger = logger;
    }

    protected async Task<Result<TEvent>> WriteToStorageAndDispatchEvent(
        TRequest request,
        string serviceName,
        string invocationId,
        string containerName,
        string queueName,
        CancellationToken cancellationToken)
    {
        try
        {
            var fileName = $"{invocationId}_{DateTime.Now:yyyyMMddHHmmss}.json";
            var content = JsonSerializer.Serialize(request);
            
            await _storage.WriteAsync(containerName, fileName, content, cancellationToken);
            
            var domainEvent = (TEvent)Activator.CreateInstance(typeof(TEvent))!;
            domainEvent.Container = containerName;
            domainEvent.FileName = fileName;
            domainEvent.ProtocolNumber = request.InvocationId;
            
            await _stream.PublishAsync(domainEvent, queueName, cancellationToken);
            
            return Result.Ok(domainEvent);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro ao gravar arquivo");
            return Result.Fail<TEvent>(new Error(ex.Message).CausedBy(ex));
        }
    }
}
```

### Command Handler Implementation

```csharp
// Application/<Company>.<Project>.Application/Domain/Risco/Emissao/RiscoEmissaoHandler.cs

using Confitec.Gepro.Application.Common.Handler;
using Confitec.Gepro.Functions.Shared.Common;
using Confitec.Gepro.Functions.Shared.Storage;
using Confitec.Gepro.Functions.Shared.Stream;
using Microsoft.Extensions.Logging;

namespace Confitec.Gepro.Application.Domain.Risco.Emissao;

public class RiscoEmissaoHandler
    : GravadorDeArquivoHandler<RiscoEmissaoCommandRequest, RiscoEmissaoGravadoEvent>, 
    ICommandHandler<RiscoEmissaoCommandRequest, RiscoEmissaoGravadoEvent>
{
    public RiscoEmissaoHandler(IStream stream, IStorage storage, ILogger<RiscoEmissaoCommandRequest> logger) : 
        base(stream, storage, logger) { }

    private static readonly string _CONTAINER_NAME = "riscoemissao";
    private static readonly string _QUEUENAME = "riscoemitidogravado";
    private static readonly string _SERVICE_NAME = "Risco Emissão";

    public async Task<Result<RiscoEmissaoGravadoEvent>> Handle(RiscoEmissaoCommandRequest request, CancellationToken cancellationToken)
    {
        return await WriteToStorageAndDispatchEvent(
            request, 
            _SERVICE_NAME, 
            request.InvocationId, 
            _CONTAINER_NAME, 
            _QUEUENAME, 
            cancellationToken);
    }
}
```

---

## Example 3: Validator

```csharp
// Application/<Company>.<Project>.Application/Domain/Risco/Emissao/RiscoEmissaoValidator.cs

using FluentValidation;
using Confitec.Gepro.Application.Common.Validators;

namespace Confitec.Gepro.Application.Domain.Risco.Emissao;

public class RiscoEmissaoValidator : AbstractValidator<RiscoEmissaoCommandRequest>
{
    public RiscoEmissaoValidator()
    {
        RuleFor(x => x.remessa)
            .NotNull()
            .WithMessage("Remessa é obrigatória");

        RuleFor(x => x.remessa!.codCiaSusep)
            .SetValidator(new SeguradoraPermitidaValidator());

        RuleFor(x => x.remessa!.codRemessaCia)
            .NotEmpty()
            .WithMessage("Código da remessa é obrigatório");

        RuleFor(x => x.riscos)
            .NotNull()
            .WithMessage("Riscos são obrigatórios");

        RuleFor(x => x.riscos!)
            .Must(itens => itens != null && itens.Any())
            .WithMessage("Deve conter pelo menos um risco");
    }
}
```

---

## Key Patterns Summary

| Pattern | Interface | Handler | Return |
|--------|----------|---------|--------|
| Read | `IQuery<T>` | `IRequestHandler<Q, Result<T>>` | `Result<T>` |
| Write | `ICommand<E>` | `IRequestHandler<C, Result<E>>` | `Result<E>` |
| Event | `IDomainEvent` | `INotificationHandler<E>` | - |

### FluentResults Pattern

```csharp
// Always use try-catch in handlers
try
{
    var entity = await repository.GetAsync(predicate);
    return entity is null
        ? Result.Fail<T>(new Error("Not found"))
        : Result.Ok(entity);
}
catch (Exception ex)
{
    return Result.Fail<T>(new Error("Error").CausedBy(ex));
}

// Never throw exceptions
// Never return null directly
```

---

## References

- Main skill: `SKILL.md`
- Templates: `templates/README.md`
- Azure Functions: `azure-functions/SKILL.md`
- Repository adapter: `adapter/repository/SKILL.md`