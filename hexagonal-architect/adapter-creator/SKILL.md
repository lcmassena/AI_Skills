---
name: adapter-creator
description: Creates new adapter projects following Hexagonal Architecture pattern with proper structure, .csproj, service implementation, and IStartupRegister.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Adapter Creator Skill

## Overview

The Adapter Creator skill automates the creation of new adapter projects following the Hexagonal Architecture pattern. It creates a complete adapter structure with all necessary files, following the same patterns established by existing adapters (AzureServiceBus, AzureStorage, SQL).

## Adapter Project Structure

Every adapter project follows a standardized folder structure:

```
/Adapters/<Category>/<ServiceName>/
├── Adapters.<Category>.<ServiceName>.csproj    # Project file
├── Services/
│   └── <ServiceName>Service.cs           # Main implementation
└── Startup/
    ├── <ServiceName>Startup.cs          # DI registration
    └── <ServiceName>Settings.cs        # Configuration class
```

## Project Template

### .csproj File

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <AssemblyTitle><Company>.<Project>.Adapters.<Category>.<ServiceName></AssemblyTitle>
    <AssemblyName><Company>.<Project>.Adapters.<Category>.<ServiceName></AssemblyName>
    <RootNamespace><Company>.<Project>.Adapters.<Category>.<ServiceName></RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="<NuGetPackage>" Version="<Version>" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\..\Shared\<Company>.<Project>.Shared.csproj" />
  </ItemGroup>

</Project>
```

### Services/<ServiceName>Service.cs

```csharp
namespace <Company>.<Project>.Adapters.<Category>.<ServiceName>.Services;

public class <ServiceName>Service : I<PortInterface>
{
    public async Task<<ReturnType>> OperationAsync<<Parameters>>)
    {
        // Implementation
    }
}
```

### Startup/<ServiceName>Settings.cs

```csharp
namespace <Company>.<Project>.Adapters.<Category>.<ServiceName>.Startup;

public class <ServiceName>Settings
{
    public string ConnectionString { get; set; }
}
```

### Startup/<ServiceName>Startup.cs

```csharp
using <Company>.<Project>.Adapters.<Category>.<ServiceName>.Services;
using <Company>.<Project>.Shared.Common.Startup;
using <Company>.<Project>.Shared.<PortNamespace>;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace <Company>.<Project>.Adapters.<Category>.<ServiceName>.Startup;

public class <ServiceName>Startup : IStartupRegister
{
    public IServiceCollection Register(
        IServiceCollection services, 
        IConfiguration configuration, 
        IServiceProvider serviceProvider)
    {
        var settings = configuration.GetSection("<ServiceName>Settings")
            .Get<<ServiceName>Settings>();

        if (settings is not null && !string.IsNullOrWhiteSpace(settings.ConnectionString))
        {
            services.AddSingleton<<ServiceName>Settings>(settings);
            services.AddSingleton<<ClientType>>(new <ClientType>(settings.ConnectionString));
            services.AddSingleton<I<PortInterface>, <ServiceName>Service>();
        }

        return services;
    }
}
```

## Naming Convention

| Adapter Type | Category | ServiceName Example | Folder Path |
|-------------|---------|-------------|-------------|
| Message Queue | Stream | AzureServiceBus | /Adapters/Stream/AzureServiceBus/ |
| Blob Storage | Storage | AzureStorage | /Adapters/Storage/AzureStorage/ |
| SQL Database | Repository | SQLServer | /Adapters/Repository/SQLServer/ |
| PostgreSQL | Repository | Postgres | /Adapters/Repository/Postgres/ |
| MongoDB | Repository | MongoDb | /Adapters/Repository/MongoDb/ |
| Redis | Cache | Redis | /Adapters/Cache/Redis/ |

## Adapter Categories

### 1. Stream Adapter

For message queue and event streaming.

**Category:** Stream
**Interface:** IStream (from Shared.Stream)
**Port Interface Location:** `Shared/Stream/IStream.cs`

### 2. Storage Adapter

For blob and file storage.

**Category:** Storage
**Interface:** IStorage (from Shared.Storage)
**Port Interface Location:** `Shared/Storage/IStorage.cs`

### 3. Repository Adapter

For database persistence.

**Category:** Repository
**Interface:** IState<T> (from Shared.States)
**Port Interface Location:** `Shared/States/IState.cs`

### 4. Cache Adapter

For caching systems.

**Category:** Cache
**Interface:** ICache (custom or from library)

## Creation Steps

When creating a new adapter, follow these steps:

### Step 1: Create Project Folder

```
/Adapters/<Category>/<ServiceName>/
```

### Step 2: Create .csproj File

Set the correct namespace and package references.

### Step 3: Implement Service

Create the service implementation following Shared interface.

### Step 4: Create Settings

Define configuration class.

### Step 5: Create Startup

Implement IStartupRegister for automatic DI registration.

## Example: Creating a PostgreSQL Adapter

### Input

```
Category: Repository
ServiceName: Postgres
```

### Output Structure

```
/Adapters/Repository/Postgres/
├── Adapters.Repository.Postgres.csproj
├── Services/
│   └── PostgresService.cs
└── Startup/
    ├── PostgresSettings.cs
    └── PostgresStartup.cs
```

### 1. csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <AssemblyTitle><Company>.<Project>.Adapters.Repository.Postgres</AssemblyTitle>
    <AssemblyName><Company>.<Project>.Adapters.Repository.Postgres</AssemblyName>
    <RootNamespace><Company>.<Project>.Adapters.Repository.Postgres</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Npgsql" Version="8.0.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\..\Shared\<Company>.<Project>.Shared.csproj" />
  </ItemGroup>

</Project>
```

### 2. Services/IPostgresState.cs (Port Interface - Required in Shared layer first)

```csharp
namespace <Company>.<Project>.Shared.States;

public interface IPostgresState<T> : IState<T> where T : class
{
    // Additional PostgreSQL-specific methods if needed
}
```

### 3. Services/PostgresService.cs

```csharp
using <Company>.<Project>.Shared.States;
using Npgsql;

namespace <Company>.<Project>.Adapters.Repository.Postgres.Services;

public class PostgresService<T> : IPostgresState<T> where T : class
{
    private readonly NpgsqlConnection _connection;

    public PostgresService(NpgsqlConnection connection)
    {
        _connection = connection;
    }

    public async Task<Result<T>> AddAsync(T entity)
    {
        try
        {
            // Implementation
            return Result.Ok(entity);
        }
        catch (Exception ex)
        {
            return Result.Fail<T>(new Error(ex.Message).CausedBy(ex));
        }
    }

    public async Task<T> GetAsync(string id)
    {
        // Implementation
    }

    public async Task<Result> UpdateAsync(string id, T entity)
    {
        // Implementation
    }

    public async Task<Result> DeleteAsync(string id)
    {
        // Implementation
    }
}
```

### 4. Startup/PostgresSettings.cs

```csharp
namespace <Company>.<Project>.Adapters.Repository.Postgres.Startup;

public class PostgresSettings
{
    public string ConnectionString { get; set; }
}
```

### 5. Startup/PostgresStartup.cs

```csharp
using <Company>.<Project>.Adapters.Repository.Postgres.Services;
using <Company>.<Project>.Shared.Common.Startup;
using <Company>.<Project>.Shared.States;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Npgsql;

namespace <Company>.<Project>.Adapters.Repository.Postgres.Startup;

public class PostgresStartup : IStartupRegister
{
    public IServiceCollection Register(
        IServiceCollection services, 
        IConfiguration configuration, 
        IServiceProvider serviceProvider)
    {
        var settings = configuration.GetSection("PostgresSettings")
            .Get<PostgresSettings>();

        if (settings is not null && !string.IsNullOrWhiteSpace(settings.ConnectionString))
        {
            services.AddSingleton<PostgresSettings>(settings);
            services.AddSingleton(new NpgsqlConnection(settings.ConnectionString));
            services.AddSingleton(typeof(IPostgresState<>), typeof(PostgresService<>));
        }

        return services;
    }
}
```

## Adding to Solution

After creating the adapter, add it to the solution file:

```bash
dotnet sln add /Adapters/<Category>/<ServiceName>/Adapters.<Category>.<ServiceName>.csproj
```

## Configuration Required

After creating the adapter, configure it in the host project (usually the Function API or Worker):

### appsettings.json

```json
{
  "<ServiceName>Settings": {
    "ConnectionString": "Server=localhost;Database=mydb;..."
  }
}
```

## Verification

Verify the adapter was created correctly by building:

```bash
dotnet build /Adapters/<Category>/<ServiceName>/Adapters.<Category>.<ServiceName>.csproj
```

## Anti-Patterns

1. **DO NOT** create adapters without a port interface in Shared layer
2. **DO NOT** skip the IStartupRegister implementation
3. **DO NOT** hardcode connection strings (always use config)
4. **DO NOT** throw exceptions - use Result.Fail()
5. **DO NOT** implement business logic in adapters (only data access)

## References

- AzureServiceBus Adapter: `/Adapters/Stream/AzureServiceBus/`
- AzureStorage Adapter: `/Adapters/Storage/AzureStorage/`
- SQL Adapter: `/Adapters/Repository/SQLServer/`
- Shared layer: `Shared/`
- IStartupRegister: `Shared/Common/Startup/IStartupRegister.cs`