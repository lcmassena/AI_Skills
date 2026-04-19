---
name: adapter-azure-storage
description: Azure Blob Storage Adapter implementation with IStorage interface, idempotent writes, and multiple input types.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

## Overview

The Azure Storage Adapter implements blob storage operations for file management using Azure Blob Storage. Supports idempotent writes, metadata injection, and multiple input types.

## Project Structure

```
/Adapters/Storage/AzureStorage/
├── Services/
│   └── AzureStorageService.cs    # Storage implementation
├── Adapters.Storage.AzureStorage.csproj
└── *.csproj
```

## Core Implementation

The AzureStorageService implements the IStorage interface from the Shared layer.

```csharp
public class AzureStorageService : IStorage
{
    public async Task<Uri> WriteAsync(
        string invocationId, 
        string containerName, 
        string fileName, 
        MemoryStream memoryStream, 
        bool overwrite, 
        CancellationToken cancellationToken)
    {
        if (string.IsNullOrWhiteSpace(containerName))
            throw new ArgumentException("containerName cannot be empty");

        if (string.IsNullOrWhiteSpace(fileName))
            throw new ArgumentException("fileName cannot be empty");

        if (memoryStream == null || memoryStream.Length == 0)
            throw new ArgumentException("invalid memoryStream");

        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
                
        // Idempotent - creates container if not exists
        await containerClient.CreateIfNotExistsAsync(
            cancellationToken: cancellationToken);

        var blobClient = containerClient.GetBlobClient(fileName);
        memoryStream.Position = 0;

        await blobClient.UploadAsync(
            memoryStream,            
            options: GetInvocationMetadata(invocationId, overwrite),
            cancellationToken);

        return blobClient.Uri;
    }

    public async Task<Uri> WriteAsync(
        string invocationId, 
        string containerName, 
        string fileName, 
        string bytes, 
        bool overwrite = false, 
        CancellationToken cancellationToken = default)
    {
        using var stream = new MemoryStream(Encoding.UTF8.GetBytes(bytes));
        return await WriteAsync(
            invocationId, containerName, fileName, 
            stream, overwrite, cancellationToken);
    }

    public Task<Uri> WriteAsync(
        string invocationId, 
        string containerName, 
        string fileName, 
        object objectToSave, 
        bool overwrite, 
        CancellationToken cancellationToken)
    {
        string jsonContent = JsonSerializer.Serialize(objectToSave);
        return WriteAsync(
            invocationId, containerName, fileName, 
            jsonContent, overwrite, cancellationToken);
    }

    private BlobUploadOptions GetInvocationMetadata(
        string invocationId, 
        bool overwrite)
    {
        var result = new BlobUploadOptions();

        if (overwrite == false)
        {
            result.Conditions = new BlobRequestConditions
            {
                IfNoneMatch = new ETag("*")
            };
        }
        
        if (invocationId is not null)
        {
            result.Metadata.Add("InvocationId", invocationId);
        }
        return result;
    }
}
```

## Features

### Idempotent Writes

The adapter supports idempotent operations through conditional uploads.

```csharp
// With overwrite = false:
// - Creates container if not exists
// - Fails if blob already exists (etag check)
// Use for: ensuring no duplicate writes

await _storage.WriteAsync(
    invocationId, 
    containerName, 
    fileName, 
    memoryStream, 
    overwrite: false);

// With overwrite = true:
// - Always creates/overwrites the blob
// Use for: retry scenarios

await _storage.WriteAsync(
    invocationId, 
    containerName, 
    fileName, 
    memoryStream, 
    overwrite: true);
```

### Multiple Input Types

Supports different input formats.

```csharp
// MemoryStream
await _storage.WriteAsync(
    invocationId, containerName, fileName, 
    memoryStream, true, cancellationToken);

// String (plain text)
await _storage.WriteAsync(
    invocationId, containerName, fileName, 
    "Hello World", true);

// Object (JSON serialization)
await _storage.WriteAsync(
    invocationId, containerName, fileName, 
    myObject, true);
```

### Metadata Injection

Automatically adds invocation tracking metadata.

```csharp
private BlobUploadOptions GetInvocationMetadata(
    string invocationId, 
    bool overwrite)
{
    var result = new BlobUploadOptions();

    if (overwrite == false)
    {
        result.Conditions = new BlobRequestConditions
        {
            IfNoneMatch = new ETag("*")
        };
    }
    
    if (invocationId is not null)
    {
        result.Metadata.Add("InvocationId", invocationId);
    }
    return result;
}
```

## Interface Definition

The IStorage interface is defined in the Shared layer.

```csharp
namespace <Company>.<Project>.Shared.Storage;

public interface IStorage
{
    Task<Uri> WriteAsync(
        string invocationId, 
        string containerName, 
        string fileName, 
        MemoryStream memoryStream, 
        bool overwrite, 
        CancellationToken cancellationToken);
        
    Task<Uri> WriteAsync(
        string invocationId, 
        string containerName, 
        string fileName, 
        string bytes, 
        bool overwrite = false, 
        CancellationToken cancellationToken = default);
        
    Task<Uri> WriteAsync(
        string invocationId, 
        string containerName, 
        string fileName, 
        object objectToSave, 
        bool overwrite, 
        CancellationToken cancellationToken);
}
```

## Best Practices

1. Use overwrite = false for production data to prevent accidental overwrites
2. Pass invocationId for traceability
3. Handle CancellationToken properly
4. Use MemoryStream for large files

## Anti-Patterns

1. **DO NOT** ignore overwrite parameter
2. **DO NOT** skip container creation check in production
3. **DO NOT** store sensitive data without encryption

## Registration

```csharp
// Program.cs
builder.Services.AddSingleton<IStorage, AzureStorageService>();
```

## References

- Implementation: `Adapters/Storage/AzureStorage/Services/AzureStorageService.cs`
- Interface: `Shared/Storage/IStorage.cs`