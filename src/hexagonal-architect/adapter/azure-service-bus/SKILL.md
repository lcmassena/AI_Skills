---
name: adapter-azure-service-bus
description: Azure Service Bus Adapter implementation with IStream interface, topic publishing, session-based messages, and scheduled message delivery.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

## Overview

The Azure Service Bus Adapter implements asynchronous message publishing using Azure Service Bus. This adapter supports topic publishing, session-based messages, and scheduled message delivery.

## Project Structure

```
/Adapters/Stream/AzureServiceBus/
├── Services/
│   └── AzureServiceBusService.cs    # Service Bus implementation
├── Adapters.Stream.AzureServiceBus.csproj
└── *.csproj
```

## Core Implementation

The AzureServiceBusService implements the IStream interface from the Shared layer and provides message publishing capabilities.

```csharp
public class AzureServiceBusService : IStream
{
    private readonly ConcurrentDictionary<string, ServiceBusSender> _instances = new();

    private ServiceBusSender Register(string topicName, ServiceBusSender sender)
    {
        _instances.TryAdd(topicName, sender);
        return sender;
    }

    private ServiceBusSender Get(string topicName)
    {
        return _instances.TryGetValue(topicName, out var instance) 
            ? instance 
            : Register(topicName, client.CreateSender(topicName));
    }

    public async Task SendEventAsync<T>(T notification, string topicName) 
        where T : IDomainEvent
    {
        var sender = Get(topicName);
        var message = new ServiceBusMessage(
            JsonConvert.SerializeObject(notification));
        await sender.SendMessageAsync(message);
    }

    public async Task SendEventAsync(
        string notification, 
        string topicName, 
        Dictionary<string, object>? properties = null, 
        string? sessionId = null)
    {
        var sender = Get(topicName);
        var message = new ServiceBusMessage(notification);

        if (!string.IsNullOrEmpty(sessionId))
        {
            message.SessionId = sessionId;
        }

        if (properties != null)
        {
            foreach (var property in properties)
            {
                message.ApplicationProperties[property.Key] = property.Value;
            }
        }

        await sender.SendMessageAsync(message);
    }

    public async Task SendEventAsync<T>(
        T notification, 
        string topicName, 
        DateTime? scheduleMessage) where T : IDomainEvent
    {
        var sender = Get(topicName);
        var message = new ServiceBusMessage(
            JsonConvert.SerializeObject(notification))
        {
            ScheduledEnqueueTime = scheduleMessage.HasValue 
                ? scheduleMessage.Value.ToUniversalTime() 
                : DateTime.UtcNow
        };
        await sender.SendMessageAsync(message);
    }
}
```

## Domain Events

Both publishers and consumers must implement the IDomainEvent interface.

```csharp
public interface IDomainEvent
{
    Guid Id { get; }
    DateTime OccurredOn { get; }
}
```

## Features

### Basic Message Publishing

```csharp
var domainEvent = new DomainEvent { Id = Guid.NewGuid() };
await _stream.SendEventAsync(domainEvent, "topic-name");
```

### Session-Based Messages

For ordered processing within a session.

```csharp
await _stream.SendEventAsync(
    notification, 
    "topic-name", 
    properties: null, 
    sessionId: "session-123");
```

### Scheduled Messages

Schedule messages for future delivery.

```csharp
await _stream.SendEventAsync(
    notification, 
    "topic-name", 
    DateTime.UtcNow.AddHours(1));
```

### Custom Properties

Add application properties to messages.

```csharp
await _stream.SendEventAsync(
    notification, 
    "topic-name", 
    new Dictionary<string, object>
    {
        { "CorrelationId", Guid.NewGuid() },
        { "Source", "my-function" }
    });
```

## Interface Definition

The IStream interface is defined in the Shared layer.

```csharp
namespace <Company>.<Project>.Shared.Stream;

public interface IStream
{
    Task SendEventAsync<T>(T notification, string topicName) 
        where T : IDomainEvent;
        
    Task SendEventAsync(
        string notification, 
        string topicName, 
        Dictionary<string, object>? properties = null, 
        string? sessionId = null);
        
    Task SendEventAsync<T>(
        T notification, 
        string topicName, 
        DateTime? scheduleMessage) where T : IDomainEvent;
}
```

## Best Practices

1. Use domain events for all message publishing
2. Handle session IDs for ordered processing
3. Use scheduled messages for deferred operations
4. Always serialize messages as JSON

## Anti-Patterns

1. **DO NOT** couple producers and consumers directly
2. **DO NOT** send sensitive data without encryption
3. **DO NOT** use topics for request-response patterns

## Registration

```csharp
// Program.cs
builder.Services.AddSingleton<IStream, AzureServiceBusService>();
```

## References

- Implementation: `/Adapters/Stream/AzureServiceBus/Services/AzureServiceBusService.cs`
- Interface: `Shared/Stream/IStream.cs`