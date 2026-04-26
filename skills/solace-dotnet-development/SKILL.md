---
name: solace-dotnet-development
description: 'Solace .NET API development guidance. Use for SolaceSystems.Solclient.Messaging C# patterns, async/await publishing, IHostedService integration, .NET Core dependency injection, and structured logging with Serilog/ILogger.'
argument-hint: 'async, dependency-injection, hosted-service, or logging'
---

# Solace .NET Development

Expert guidance for building .NET applications with Solace PubSub+ messaging.

## When to Use

- **.NET Core/5+/6+**: Modern .NET applications
- **ASP.NET Core**: Web APIs with messaging
- **Worker Services**: Background processing
- **Dependency Injection**: IServiceCollection integration
- **Async Patterns**: Task-based asynchronous programming

## Installation

```bash
dotnet add package SolaceSystems.Solclient.Messaging
```

## Basic Connection

```csharp
using SolaceSystems.Solclient.Messaging;

public class SolaceConnection : IDisposable
{
    private IContext _context;
    private ISession _session;

    public async Task ConnectAsync(SolaceConfig config)
    {
        // Initialize API
        ContextFactoryProperties cfp = new ContextFactoryProperties()
        {
            SolClientLogLevel = SolLogLevel.Warning
        };
        ContextFactory.Instance.Init(cfp);

        // Create context
        _context = ContextFactory.Instance.CreateContext(new ContextProperties(), null);

        // Session properties
        SessionProperties sessionProps = new SessionProperties()
        {
            Host = config.Host,
            VPNName = config.VpnName,
            UserName = config.Username,
            Password = config.Password,
            ReconnectRetries = -1,        // Infinite retries
            ReconnectRetriesWaitInMsecs = 3000,
            ConnectTimeoutInMsecs = 10000
        };

        // Create and connect session
        _session = _context.CreateSession(sessionProps, HandleMessage, HandleSessionEvent);
        
        var result = _session.Connect();
        if (result != ReturnCode.SOLCLIENT_OK)
        {
            throw new InvalidOperationException($"Connect failed: {result}");
        }
    }

    private void HandleMessage(object sender, MessageEventArgs args)
    {
        // Handle incoming messages
    }

    private void HandleSessionEvent(object sender, SessionEventArgs args)
    {
        Console.WriteLine($"Session event: {args.Event}");
    }

    public void Dispose()
    {
        _session?.Dispose();
        _context?.Dispose();
        ContextFactory.Instance.Cleanup();
    }
}
```

## Publishing

### Direct Publisher

```csharp
public class DirectPublisher
{
    private readonly ISession _session;

    public DirectPublisher(ISession session)
    {
        _session = session;
    }

    public void Publish<T>(string topicName, T payload)
    {
        using var message = ContextFactory.Instance.CreateMessage();
        
        message.Destination = ContextFactory.Instance.CreateTopic(topicName);
        message.DeliveryMode = MessageDeliveryMode.Direct;
        
        var json = JsonSerializer.Serialize(payload);
        message.BinaryAttachment = Encoding.UTF8.GetBytes(json);
        
        var result = _session.Send(message);
        if (result != ReturnCode.SOLCLIENT_OK)
        {
            throw new InvalidOperationException($"Send failed: {result}");
        }
    }

    public async Task PublishAsync<T>(string topicName, T payload)
    {
        // Wrap synchronous call in Task for async pattern
        await Task.Run(() => Publish(topicName, payload));
    }
}
```

### Guaranteed Publisher with Acknowledgments

```csharp
public class GuaranteedPublisher : IDisposable
{
    private readonly ISession _session;
    private readonly ConcurrentDictionary<long, TaskCompletionSource<bool>> _pendingAcks;
    private long _correlationId;

    public GuaranteedPublisher(ISession session)
    {
        _session = session;
        _pendingAcks = new ConcurrentDictionary<long, TaskCompletionSource<bool>>();
    }

    public async Task<bool> PublishAsync<T>(string destination, T payload, bool isQueue = false)
    {
        var tcs = new TaskCompletionSource<bool>();
        var correlationId = Interlocked.Increment(ref _correlationId);
        
        _pendingAcks[correlationId] = tcs;

        using var message = ContextFactory.Instance.CreateMessage();
        
        if (isQueue)
        {
            message.Destination = ContextFactory.Instance.CreateQueue(destination);
        }
        else
        {
            message.Destination = ContextFactory.Instance.CreateTopic(destination);
        }
        
        message.DeliveryMode = MessageDeliveryMode.Persistent;
        message.CorrelationKey = correlationId;
        
        var json = JsonSerializer.Serialize(payload);
        message.BinaryAttachment = Encoding.UTF8.GetBytes(json);

        _session.Send(message);

        // Wait for ACK with timeout
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
        try
        {
            return await tcs.Task.WaitAsync(cts.Token);
        }
        catch (OperationCanceledException)
        {
            _pendingAcks.TryRemove(correlationId, out _);
            throw new TimeoutException("Publish acknowledgment timed out");
        }
    }

    public void HandleAck(object correlationKey, bool success)
    {
        if (correlationKey is long id && _pendingAcks.TryRemove(id, out var tcs))
        {
            tcs.SetResult(success);
        }
    }

    public void Dispose()
    {
        foreach (var pending in _pendingAcks.Values)
        {
            pending.TrySetCanceled();
        }
        _pendingAcks.Clear();
    }
}
```

## Subscribing

### Topic Subscriber

```csharp
public class TopicSubscriber
{
    private readonly ISession _session;
    private readonly ConcurrentDictionary<string, Action<IMessage>> _handlers;

    public TopicSubscriber(ISession session)
    {
        _session = session;
        _handlers = new ConcurrentDictionary<string, Action<IMessage>>();
    }

    public void Subscribe<T>(string topicPattern, Action<T, MessageMetadata> handler)
    {
        var topic = ContextFactory.Instance.CreateTopic(topicPattern);
        
        _handlers[topicPattern] = (message) =>
        {
            var payload = JsonSerializer.Deserialize<T>(
                Encoding.UTF8.GetString(message.BinaryAttachment));
            
            var metadata = new MessageMetadata
            {
                Topic = message.Destination.Name,
                CorrelationId = message.CorrelationId,
                Timestamp = message.SenderTimestamp
            };
            
            handler(payload, metadata);
        };

        var result = _session.Subscribe(topic, true);
        if (result != ReturnCode.SOLCLIENT_OK)
        {
            throw new InvalidOperationException($"Subscribe failed: {result}");
        }
    }

    public void ProcessMessage(IMessage message)
    {
        var topic = message.Destination.Name;
        
        foreach (var kvp in _handlers)
        {
            if (MatchesTopic(topic, kvp.Key))
            {
                kvp.Value(message);
            }
        }
    }

    private bool MatchesTopic(string topic, string pattern)
    {
        // Implement Solace wildcard matching
        var regex = pattern
            .Replace("*", "[^/]+")
            .Replace(">", ".*");
        return Regex.IsMatch(topic, $"^{regex}$");
    }
}

public record MessageMetadata
{
    public string Topic { get; init; }
    public string CorrelationId { get; init; }
    public long Timestamp { get; init; }
}
```

### Queue Consumer

```csharp
public class QueueConsumer : IDisposable
{
    private readonly ISession _session;
    private IFlow _flow;
    private readonly CancellationTokenSource _cts;

    public QueueConsumer(ISession session)
    {
        _session = session;
        _cts = new CancellationTokenSource();
    }

    public void StartConsuming<T>(string queueName, Func<T, Task> handler)
    {
        var flowProps = new FlowProperties()
        {
            AckMode = MessageAckMode.ClientAck,
            BindBlocking = true
        };

        var queue = ContextFactory.Instance.CreateQueue(queueName);

        _flow = _session.CreateFlow(
            flowProps,
            queue,
            null,
            async (sender, args) =>
            {
                try
                {
                    var payload = JsonSerializer.Deserialize<T>(
                        Encoding.UTF8.GetString(args.Message.BinaryAttachment));
                    
                    await handler(payload);
                    
                    // Acknowledge after successful processing
                    _flow.Ack(args.Message.ADMessageId);
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"Processing failed: {ex.Message}");
                    // Message will be redelivered or sent to DMQ
                }
            },
            (sender, args) =>
            {
                Console.WriteLine($"Flow event: {args.Event}");
            });

        _flow.Start();
    }

    public void Dispose()
    {
        _cts.Cancel();
        _flow?.Dispose();
    }
}
```

## Dependency Injection

### Service Registration

```csharp
// SolaceServiceExtensions.cs
public static class SolaceServiceExtensions
{
    public static IServiceCollection AddSolaceMessaging(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.Configure<SolaceConfig>(configuration.GetSection("Solace"));
        
        services.AddSingleton<ISolaceConnection, SolaceConnection>();
        services.AddSingleton<IPublisher, DirectPublisher>();
        services.AddSingleton<ISubscriber, TopicSubscriber>();
        
        return services;
    }
}

// Configuration
public class SolaceConfig
{
    public string Host { get; set; }
    public string VpnName { get; set; }
    public string Username { get; set; }
    public string Password { get; set; }
}
```

### appsettings.json

```json
{
  "Solace": {
    "Host": "tcp://localhost:55555",
    "VpnName": "default",
    "Username": "admin",
    "Password": "admin"
  }
}
```

## IHostedService Integration

```csharp
public class SolaceBackgroundService : BackgroundService
{
    private readonly ILogger<SolaceBackgroundService> _logger;
    private readonly ISolaceConnection _connection;
    private readonly IServiceProvider _services;

    public SolaceBackgroundService(
        ILogger<SolaceBackgroundService> logger,
        ISolaceConnection connection,
        IServiceProvider services)
    {
        _logger = logger;
        _connection = connection;
        _services = services;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Starting Solace consumer service");

        try
        {
            await _connection.ConnectAsync();
            
            var consumer = new QueueConsumer(_connection.Session);
            consumer.StartConsuming<OrderEvent>("order-queue", async order =>
            {
                using var scope = _services.CreateScope();
                var handler = scope.ServiceProvider.GetRequiredService<IOrderHandler>();
                await handler.HandleAsync(order);
            });

            await Task.Delay(Timeout.Infinite, stoppingToken);
        }
        catch (OperationCanceledException)
        {
            _logger.LogInformation("Solace consumer stopping");
        }
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Disconnecting from Solace");
        _connection.Dispose();
        await base.StopAsync(cancellationToken);
    }
}
```

## Structured Logging

### Serilog Integration

```csharp
public class LoggingPublisher : IPublisher
{
    private readonly ISession _session;
    private readonly ILogger<LoggingPublisher> _logger;

    public LoggingPublisher(ISession session, ILogger<LoggingPublisher> logger)
    {
        _session = session;
        _logger = logger;
    }

    public void Publish<T>(string topic, T payload)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["Topic"] = topic,
            ["MessageType"] = typeof(T).Name,
            ["CorrelationId"] = Activity.Current?.Id
        }))
        {
            try
            {
                _logger.LogDebug("Publishing message to {Topic}", topic);
                
                // ... publish logic
                
                _logger.LogInformation("Message published successfully");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to publish message to {Topic}", topic);
                throw;
            }
        }
    }
}
```

## Error Handling

```csharp
public static class SolaceErrorHandler
{
    public static void HandleSessionEvent(SessionEventArgs args)
    {
        switch (args.Event)
        {
            case SessionEvent.ConnectFailedError:
                throw new SolaceConnectionException(
                    $"Connection failed: {args.Info}");
                
            case SessionEvent.Reconnecting:
                Log.Warning("Attempting to reconnect to Solace");
                break;
                
            case SessionEvent.Reconnected:
                Log.Information("Successfully reconnected to Solace");
                break;
                
            case SessionEvent.DownError:
                Log.Error("Session down: {Error}", args.Info);
                break;
                
            case SessionEvent.SubscriptionError:
                Log.Error("Subscription failed: {Error}", args.Info);
                break;
        }
    }
}

public class SolaceConnectionException : Exception
{
    public SolaceConnectionException(string message) : base(message) { }
}
```

## Reference Links

- **.NET API Documentation**: https://docs.solace.com/API/Messaging-APIs/Dot-Net-API/Dot-Net-home.htm
- **API Reference**: https://docs.solace.com/API-Developer-Online-Ref-Documentation/dotnet/index.html
- **Tutorials**: https://tutorials.solace.dev/dotnet

## Further Reading

- [Async Patterns](./references/async-patterns.md)
- [Integration Testing](./references/testing.md)
- [Health Checks](./references/health-checks.md)
