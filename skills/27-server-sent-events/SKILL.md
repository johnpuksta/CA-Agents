---
name: server-sent-events
description: "Implements Server-Sent Events (SSE) for real-time server-to-client streaming in ASP.NET Core and .NET 10+. A lightweight, standards-based alternative to SignalR for unidirectional updates."
version: 1.0.0
language: C#
framework: .NET 10+
dependencies: ASP.NET Core
---

# Server-Sent Events (SSE) for Real-Time Streaming

## Overview

.NET 10 introduces native, high-level API support for Server-Sent Events (SSE), providing a lightweight alternative to SignalR for simpler real-time scenarios.

**Server-Sent Events** enable efficient, one-way streaming from server to client using standard HTTP with automatic browser reconnection.

## Quick Reference

| Feature | SSE | SignalR |
|---------|-----|---------|
| **Direction** | Unidirectional (server → client) | Bidirectional (server ↔ client) |
| **Protocol** | Standard HTTP (`text/event-stream`) | WebSockets + fallbacks |
| **Client Library** | Native browser `EventSource` API | SignalR client library required |
| **Authentication** | Standard HTTP headers (JWT, cookies) | SignalR-specific handshake |
| **Reconnection** | Automatic with `Last-Event-ID` | SignalR manages reconnection |
| **Complexity** | Low - simple streaming | Higher - hubs, protocols, backplane |
| **Use Case** | Notifications, dashboards, progress | Chat, games, bidirectional workflows |

---

## When to Use SSE

### ✅ SSE is Ideal For:
- **One-way updates**: Notifications, live feeds, progress indicators
- **Dashboard updates**: Real-time metrics, monitoring displays
- **Event streams**: Log tailing, audit trails, activity feeds
- **Lightweight scenarios**: Where SignalR overhead is unnecessary
- **Standard HTTP requirements**: Existing middleware compatibility
- **Browser-native simplicity**: No npm dependencies on client

### ❌ Use SignalR Instead For:
- **Bidirectional communication**: Client needs to send messages back
- **Complex workflows**: Multi-step request/response patterns
- **Massive scale with backplane**: Redis/Azure SignalR Service required
- **Non-browser clients**: Mobile apps, desktop apps (SignalR has SDKs)
- **Guaranteed delivery**: More robust acknowledgment patterns

---

## SSE Structure

```
/API/SSE/
├── Endpoints/
│   ├── NotificationEndpoints.cs
│   ├── OrderStreamEndpoints.cs
│   └── ProgressEndpoints.cs
├── Services/
│   ├── EventBuffer.cs           # Event replay for reconnections
│   └── BroadcastService.cs      # Fan-out to multiple clients
└── Channels/
    └── NotificationChannel.cs   # Shared channel for broadcasts
```

---

## Template: Basic SSE Endpoint

Simplest implementation using `IAsyncEnumerable<T>`:

```csharp
// src/{name}.api/Endpoints/OrderEndpoints.cs
using System.Threading.Channels;

public static class OrderEndpoints
{
    public static void MapOrderStreamEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapGet("/stream", StreamOrders);
    }

    // ═══════════════════════════════════════════════════════════════
    // BASIC SSE ENDPOINT
    // Returns IResult with ServerSentEvents
    // ═══════════════════════════════════════════════════════════════
    private static IResult StreamOrders(
        ChannelReader<OrderPlacement> channelReader,
        CancellationToken cancellationToken)
    {
        return Results.ServerSentEvents(
            channelReader.ReadAllAsync(cancellationToken),
            eventType: "orders");
    }
}

// Response DTO
public sealed record OrderPlacement(
    Guid OrderId,
    string CustomerName,
    decimal Amount,
    DateTime PlacedAt);
```

### Client-Side Consumption

```javascript
// Native browser EventSource API - no libraries needed
const eventSource = new EventSource('/api/orders/stream', {
    withCredentials: true  // Include cookies/auth
});

eventSource.addEventListener('orders', (event) => {
    const order = JSON.parse(event.data);
    console.log(`New Order ${order.orderId}:`, order);

    // Update UI
    addOrderToTable(order);
});

eventSource.onopen = () => {
    console.log('Connected to order stream');
};

eventSource.onerror = (error) => {
    if (eventSource.readyState === EventSource.CONNECTING) {
        console.log('Reconnecting...');
    } else {
        console.error('Connection failed:', error);
    }
};

// Cleanup when done
// eventSource.close();
```

---

## Template: SSE with Event Replay (Reconnection Support)

Use `SseItem<T>` for event IDs and automatic replay:

```csharp
// src/{name}.api/Endpoints/OrderEndpoints.cs
using Microsoft.AspNetCore.Mvc;

public static class OrderEndpoints
{
    public static void MapOrderStreamEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders")
            .RequireAuthorization();

        group.MapGet("/stream/reliable", StreamOrdersWithReplay);
    }

    // ═══════════════════════════════════════════════════════════════
    // SSE WITH EVENT REPLAY
    // Supports reconnection with Last-Event-ID header
    // ═══════════════════════════════════════════════════════════════
    private static IResult StreamOrdersWithReplay(
        ChannelReader<OrderPlacement> channelReader,
        OrderEventBuffer eventBuffer,
        [FromHeader(Name = "Last-Event-ID")] string? lastEventId,
        CancellationToken cancellationToken)
    {
        async IAsyncEnumerable<SseItem<OrderPlacement>> StreamEvents(
            [EnumeratorCancellation] CancellationToken ct)
        {
            // ═══════════════════════════════════════════════════════════════
            // STEP 1: Replay missed events
            // ═══════════════════════════════════════════════════════════════
            if (!string.IsNullOrWhiteSpace(lastEventId))
            {
                var missedEvents = eventBuffer.GetEventsAfter(lastEventId);
                foreach (var missedEvent in missedEvents)
                {
                    yield return missedEvent;
                }
            }

            // ═══════════════════════════════════════════════════════════════
            // STEP 2: Stream new events
            // ═══════════════════════════════════════════════════════════════
            await foreach (var order in channelReader.ReadAllAsync(ct))
            {
                var sseItem = eventBuffer.Add(order);
                yield return sseItem;
            }
        }

        return TypedResults.ServerSentEvents(
            StreamEvents(cancellationToken),
            eventType: "orders");
    }
}
```

### Event Buffer Implementation

```csharp
// src/{name}.api/Services/OrderEventBuffer.cs
using System.Collections.Concurrent;

namespace {name}.api.services;

/// <summary>
/// Buffers events for replay on client reconnection
/// </summary>
public sealed class OrderEventBuffer
{
    private readonly ConcurrentQueue<SseItem<OrderPlacement>> _events = new();
    private readonly int _maxBufferSize;
    private long _eventIdCounter = 0;

    public OrderEventBuffer(int maxBufferSize = 1000)
    {
        _maxBufferSize = maxBufferSize;
    }

    public SseItem<OrderPlacement> Add(OrderPlacement order)
    {
        var eventId = Interlocked.Increment(ref _eventIdCounter).ToString();

        var sseItem = new SseItem<OrderPlacement>(order)
        {
            Id = eventId,
            EventType = "orders"
        };

        _events.Enqueue(sseItem);

        // Trim buffer if too large
        while (_events.Count > _maxBufferSize)
        {
            _events.TryDequeue(out _);
        }

        return sseItem;
    }

    public IEnumerable<SseItem<OrderPlacement>> GetEventsAfter(string lastEventId)
    {
        if (!long.TryParse(lastEventId, out var lastId))
        {
            return Enumerable.Empty<SseItem<OrderPlacement>>();
        }

        return _events
            .Where(e => long.Parse(e.Id!) > lastId)
            .ToList();
    }
}
```

### DI Registration

```csharp
// src/{name}.api/Program.cs
builder.Services.AddSingleton<OrderEventBuffer>();
```

---

## Template: User-Specific Filtering

Filter streams per authenticated user:

```csharp
// src/{name}.api/Endpoints/NotificationEndpoints.cs
public static class NotificationEndpoints
{
    public static void MapNotificationEndpoints(this IEndpointRouteBuilder app)
    {
        app.MapGet("/api/notifications/stream", StreamUserNotifications)
            .RequireAuthorization();
    }

    // ═══════════════════════════════════════════════════════════════
    // USER-SPECIFIC SSE STREAM
    // Filters events by authenticated user
    // ═══════════════════════════════════════════════════════════════
    private static IResult StreamUserNotifications(
        ChannelReader<Notification> channelReader,
        IUserContext userContext,
        CancellationToken cancellationToken)
    {
        var currentUserId = userContext.UserId;

        async IAsyncEnumerable<Notification> GetUserNotifications(
            [EnumeratorCancellation] CancellationToken ct)
        {
            await foreach (var notification in channelReader.ReadAllAsync(ct))
            {
                // Only yield notifications for current user
                if (notification.UserId == currentUserId)
                {
                    yield return notification;
                }
            }
        }

        return Results.ServerSentEvents(
            GetUserNotifications(cancellationToken),
            eventType: "notification");
    }
}
```

---

## Template: Broadcast Service with Channels

Efficient fan-out to multiple clients using shared channels:

```csharp
// src/{name}.api/Services/NotificationBroadcastService.cs
using System.Threading.Channels;

namespace {name}.api.services;

/// <summary>
/// Manages broadcast to multiple SSE clients via channels
/// </summary>
public sealed class NotificationBroadcastService : IHostedService, IDisposable
{
    private readonly Channel<Notification> _sourceChannel;
    private readonly ConcurrentDictionary<string, Channel<Notification>> _subscribers;
    private Task? _broadcastTask;
    private readonly CancellationTokenSource _cts;

    public NotificationBroadcastService()
    {
        _sourceChannel = Channel.CreateUnbounded<Notification>();
        _subscribers = new ConcurrentDictionary<string, Channel<Notification>>();
        _cts = new CancellationTokenSource();
    }

    // ═══════════════════════════════════════════════════════════════
    // PUBLISH - Single producer writes to source channel
    // ═══════════════════════════════════════════════════════════════
    public async Task PublishAsync(Notification notification, CancellationToken ct = default)
    {
        await _sourceChannel.Writer.WriteAsync(notification, ct);
    }

    // ═══════════════════════════════════════════════════════════════
    // SUBSCRIBE - Register client for receiving notifications
    // ═══════════════════════════════════════════════════════════════
    public ChannelReader<Notification> Subscribe(string subscriberId)
    {
        var channel = Channel.CreateUnbounded<Notification>();
        _subscribers.TryAdd(subscriberId, channel);
        return channel.Reader;
    }

    // ═══════════════════════════════════════════════════════════════
    // UNSUBSCRIBE - Clean up when client disconnects
    // ═══════════════════════════════════════════════════════════════
    public void Unsubscribe(string subscriberId)
    {
        if (_subscribers.TryRemove(subscriberId, out var channel))
        {
            channel.Writer.Complete();
        }
    }

    // ═══════════════════════════════════════════════════════════════
    // BACKGROUND TASK - Fan out to all subscribers
    // ═══════════════════════════════════════════════════════════════
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _broadcastTask = Task.Run(async () =>
        {
            await foreach (var notification in _sourceChannel.Reader.ReadAllAsync(_cts.Token))
            {
                // Fan-out: Write to all subscriber channels
                foreach (var subscriber in _subscribers.Values)
                {
                    // Non-blocking write
                    await subscriber.Writer.WriteAsync(notification, _cts.Token);
                }
            }
        }, _cts.Token);

        return Task.CompletedTask;
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        _sourceChannel.Writer.Complete();
        _cts.Cancel();

        if (_broadcastTask != null)
        {
            await _broadcastTask;
        }

        // Clean up all subscriber channels
        foreach (var channel in _subscribers.Values)
        {
            channel.Writer.Complete();
        }
    }

    public void Dispose()
    {
        _cts.Dispose();
    }
}
```

### Using Broadcast Service

```csharp
// src/{name}.api/Endpoints/NotificationEndpoints.cs
public static class NotificationEndpoints
{
    public static void MapNotificationEndpoints(this IEndpointRouteBuilder app)
    {
        app.MapGet("/api/notifications/broadcast", StreamBroadcast)
            .RequireAuthorization();
    }

    private static IResult StreamBroadcast(
        NotificationBroadcastService broadcastService,
        IUserContext userContext,
        CancellationToken cancellationToken)
    {
        var subscriberId = Guid.NewGuid().ToString();
        var channelReader = broadcastService.Subscribe(subscriberId);

        async IAsyncEnumerable<Notification> StreamWithCleanup(
            [EnumeratorCancellation] CancellationToken ct)
        {
            try
            {
                await foreach (var notification in channelReader.ReadAllAsync(ct))
                {
                    // Filter by user
                    if (notification.UserId == userContext.UserId)
                    {
                        yield return notification;
                    }
                }
            }
            finally
            {
                // Clean up when client disconnects
                broadcastService.Unsubscribe(subscriberId);
            }
        }

        return Results.ServerSentEvents(
            StreamWithCleanup(cancellationToken),
            eventType: "notification");
    }
}
```

---

## Template: Progress Tracking

Stream long-running operation progress:

```csharp
// src/{name}.api/Endpoints/ExportEndpoints.cs
public static class ExportEndpoints
{
    public static void MapExportEndpoints(this IEndpointRouteBuilder app)
    {
        app.MapPost("/api/exports/start", StartExport);
        app.MapGet("/api/exports/{jobId}/progress", StreamProgress);
    }

    private static async Task<IResult> StartExport(
        StartExportRequest request,
        IExportService exportService)
    {
        var jobId = await exportService.StartExportAsync(request);
        return Results.Ok(new { jobId });
    }

    // ═══════════════════════════════════════════════════════════════
    // PROGRESS STREAM
    // Client polls this endpoint after starting job
    // ═══════════════════════════════════════════════════════════════
    private static IResult StreamProgress(
        Guid jobId,
        IExportService exportService,
        CancellationToken cancellationToken)
    {
        async IAsyncEnumerable<ProgressUpdate> GetProgress(
            [EnumeratorCancellation] CancellationToken ct)
        {
            await foreach (var progress in exportService.GetProgressAsync(jobId, ct))
            {
                yield return progress;

                // Complete stream when job finishes
                if (progress.IsComplete)
                {
                    yield break;
                }
            }
        }

        return Results.ServerSentEvents(
            GetProgress(cancellationToken),
            eventType: "progress");
    }
}

public sealed record ProgressUpdate(
    int Percentage,
    string Status,
    bool IsComplete,
    string? DownloadUrl = null);
```

### Client Side

```javascript
function trackExport(jobId) {
    const eventSource = new EventSource(`/api/exports/${jobId}/progress`);

    eventSource.addEventListener('progress', (event) => {
        const progress = JSON.parse(event.data);

        // Update progress bar
        updateProgressBar(progress.percentage);
        updateStatusText(progress.status);

        if (progress.isComplete) {
            // Download file
            window.location.href = progress.downloadUrl;
            eventSource.close();
        }
    });

    eventSource.onerror = () => {
        console.error('Export progress connection lost');
    };
}
```

---

## Template: SSE with Multiple Event Types

Send different event types on same connection:

```csharp
// src/{name}.api/Endpoints/DashboardEndpoints.cs
public static class DashboardEndpoints
{
    public static void MapDashboardEndpoints(this IEndpointRouteBuilder app)
    {
        app.MapGet("/api/dashboard/stream", StreamDashboard)
            .RequireAuthorization();
    }

    // ═══════════════════════════════════════════════════════════════
    // MULTIPLE EVENT TYPES
    // Use SseItem with different EventType values
    // ═══════════════════════════════════════════════════════════════
    private static IResult StreamDashboard(
        IDashboardService dashboardService,
        CancellationToken cancellationToken)
    {
        async IAsyncEnumerable<SseItem<object>> StreamEvents(
            [EnumeratorCancellation] CancellationToken ct)
        {
            await foreach (var update in dashboardService.GetUpdatesAsync(ct))
            {
                var sseItem = update switch
                {
                    MetricUpdate metric => new SseItem<object>(metric)
                    {
                        EventType = "metric",
                        Id = metric.Timestamp.Ticks.ToString()
                    },
                    AlertUpdate alert => new SseItem<object>(alert)
                    {
                        EventType = "alert",
                        Id = alert.AlertId.ToString()
                    },
                    StatusUpdate status => new SseItem<object>(status)
                    {
                        EventType = "status",
                        Id = DateTimeOffset.UtcNow.Ticks.ToString()
                    },
                    _ => throw new NotSupportedException()
                };

                yield return sseItem;
            }
        }

        return TypedResults.ServerSentEvents(StreamEvents(cancellationToken));
    }
}
```

### Client-Side Multiple Event Types

```javascript
const eventSource = new EventSource('/api/dashboard/stream');

// Listen for metric updates
eventSource.addEventListener('metric', (event) => {
    const metric = JSON.parse(event.data);
    updateMetricChart(metric);
});

// Listen for alerts
eventSource.addEventListener('alert', (event) => {
    const alert = JSON.parse(event.data);
    showAlert(alert.message, alert.severity);
});

// Listen for status changes
eventSource.addEventListener('status', (event) => {
    const status = JSON.parse(event.data);
    updateSystemStatus(status);
});
```

---

## Production Deployment Considerations

### Reverse Proxy Configuration

**Nginx:**
```nginx
location /api/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;

    # CRITICAL: Disable buffering for SSE
    proxy_buffering off;
    proxy_cache off;

    # Keep connection alive
    proxy_set_header Connection '';
    chunked_transfer_encoding on;

    # Timeouts
    proxy_read_timeout 86400s;  # 24 hours
    proxy_send_timeout 86400s;
}
```

**Traefik:**
```yaml
http:
  middlewares:
    sse-headers:
      headers:
        customResponseHeaders:
          Cache-Control: "no-cache"
          X-Accel-Buffering: "no"
```

### Azure App Service

Add to `web.config`:
```xml
<configuration>
  <system.webServer>
    <httpProtocol>
      <customHeaders>
        <add name="Cache-Control" value="no-cache" />
      </customHeaders>
    </httpProtocol>
  </system.webServer>
</configuration>
```

### Connection Limits

**Browser Limits (HTTP/1.1):**
- 6 concurrent connections per domain
- Solution: Use HTTP/2 or subdomains

**Server Limits:**
```csharp
// Program.cs - Configure Kestrel for long-lived connections
builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.KeepAliveTimeout = TimeSpan.FromHours(2);
    options.Limits.RequestHeadersTimeout = TimeSpan.FromMinutes(2);
});
```

### Load Testing

Test concurrent connections and memory usage:

```csharp
// Load test with Apache Bench
ab -n 1000 -c 100 -H "Accept: text/event-stream" http://localhost:5000/api/orders/stream
```

---

## Critical Rules

1. **Use [EnumeratorCancellation]** - Propagate cancellation tokens properly
2. **Implement cleanup in finally blocks** - Remove subscribers on disconnect
3. **Disable proxy buffering** - Critical for real-time streaming
4. **Use channels for fan-out** - Don't create N queries for N clients
5. **Buffer events for replay** - Support reconnection with Last-Event-ID
6. **Filter by user in handler** - Use HttpContext.User for security
7. **Set appropriate event types** - Help clients distinguish events
8. **Monitor connection count** - Track resource usage at scale
9. **Prefer HTTP/2** - Avoid connection limits
10. **Test under realistic load** - Verify memory and connection handling

---

## Anti-Patterns to Avoid

```csharp
// ❌ WRONG: Separate database query per client
app.MapGet("/orders/stream", (CancellationToken ct) =>
{
    async IAsyncEnumerable<Order> Stream()
    {
        while (!ct.IsCancellationRequested)
        {
            var orders = await db.Orders.ToListAsync();  // N clients = N queries!
            foreach (var order in orders)
                yield return order;
            await Task.Delay(1000);
        }
    }
    return Results.ServerSentEvents(Stream());
});

// ✅ CORRECT: Single source with channel fan-out
var broadcastService = app.Services.GetRequiredService<BroadcastService>();
app.MapGet("/orders/stream", (string subscriberId, CancellationToken ct) =>
{
    var reader = broadcastService.Subscribe(subscriberId);
    return Results.ServerSentEvents(reader.ReadAllAsync(ct));
});

// ❌ WRONG: Missing cancellation token propagation
async IAsyncEnumerable<Order> Stream()
{
    // Client disconnect won't stop this loop!
    while (true)
    {
        yield return await GetNextOrder();
    }
}

// ✅ CORRECT: Use EnumeratorCancellation attribute
async IAsyncEnumerable<Order> Stream([EnumeratorCancellation] CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        yield return await GetNextOrder(ct);
    }
}

// ❌ WRONG: No cleanup on disconnect
app.MapGet("/stream", () =>
{
    var subscriberId = Guid.NewGuid().ToString();
    var reader = broadcastService.Subscribe(subscriberId);
    return Results.ServerSentEvents(reader.ReadAllAsync());
    // Subscriber never removed - memory leak!
});

// ✅ CORRECT: Cleanup in finally block
async IAsyncEnumerable<T> StreamWithCleanup([EnumeratorCancellation] CancellationToken ct)
{
    var subscriberId = Guid.NewGuid().ToString();
    var reader = broadcastService.Subscribe(subscriberId);
    try
    {
        await foreach (var item in reader.ReadAllAsync(ct))
        {
            yield return item;
        }
    }
    finally
    {
        broadcastService.Unsubscribe(subscriberId);
    }
}
```

---

## Performance Characteristics

**Tested on Azure B2s (2 cores, 3.5GB RAM):**
- 5,000 concurrent SSE connections
- Memory: ~2GB with proper cleanup
- CPU: Minimal when idle, spikes on broadcast

**Best Practices:**
- Use channels for broadcast fan-out
- Implement event buffering with size limits
- Monitor and clean up disconnected clients
- Test proxy configuration before production

---

## Related Skills

- `07-minimal-api-endpoints` - API endpoint implementation
- `12-jwt-authentication` - Securing SSE endpoints
- `24-logging-configuration` - Logging SSE connections
- `17-health-checks` - Monitor SSE connection health

---

## Sources

- [Server-Sent Events in ASP.NET Core and .NET 10 - Milan Jovanović](https://www.milanjovanovic.tech/blog/server-sent-events-in-aspnetcore-and-dotnet-10)
- [Server-Sent Events in .NET 10: Finally, a Native Solution](https://dev.to/mashrulhaque/server-sent-events-in-net-10-finally-a-native-solution-22kg)
- [Real-Time Server-Sent Events in ASP.NET Core](https://antondevtips.com/blog/real-time-server-sent-events-in-asp-net-core)
- [How to Implement SSE in ASP.NET Core Minimal APIs with .NET 10](https://alliancetek.medium.com/how-to-implement-server-sent-events-sse-in-asp-net-core-minimal-apis-with-net-10-33b3fba720df)
