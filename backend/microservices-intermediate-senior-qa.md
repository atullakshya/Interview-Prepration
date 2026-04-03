# Web API and Microservices Interview Q&A (C#, Intermediate to Senior, 30 Questions)

This guide is designed for intermediate to senior engineers.
Each question includes:
- Short answer
- Real-time example
- Practical explanation
- C# code snippet with comments

---

## 1) What is a microservice, and how is it different from a monolith?

Short answer:
A microservice is a small, independently deployable service focused on one business capability.

Real-time example:
In an e-commerce platform, Product Catalog, Order, Payment, and Notification can be separate services.

Explanation:
Monoliths are easier to start. Microservices improve independent scaling, ownership, and release speed, but add distributed-system complexity.

```csharp
// Example: Order service endpoint separated from other domains
app.MapPost("/api/orders", (CreateOrderRequest req) =>
{
    // Order service only handles order concerns
    return Results.Created($"/api/orders/{Guid.NewGuid()}", new { req.CustomerId, req.Items });
});

public record CreateOrderRequest(Guid CustomerId, List<string> Items);
```

---

## 2) How do you decide service boundaries?

Short answer:
Use business capabilities and bounded contexts, not technical layers.

Real-time example:
In ride-sharing, Trip service and Driver service are separate bounded contexts.

Explanation:
Avoid splitting by CRUD tables only. Split by business ownership and change frequency.

```csharp
// Good boundary by capability
// DriverService: onboarding, status, availability
// TripService: matching, trip lifecycle, fares

public interface ITripService
{
    Task<TripResult> CreateTripAsync(CreateTripRequest request, CancellationToken ct);
}

public record CreateTripRequest(Guid RiderId, string Pickup, string Dropoff);
public record TripResult(Guid TripId, string Status);
```

---

## 3) What communication patterns are common between microservices?

Short answer:
Synchronous HTTP/gRPC and asynchronous messaging via broker.

Real-time example:
Order service calls Inventory synchronously to reserve stock, then publishes OrderPlaced event asynchronously.

Explanation:
Use synchronous for immediate responses. Use async events for decoupling and resilience.

```csharp
// Synchronous API call with HttpClient
public class InventoryClient
{
    private readonly HttpClient _http;

    public InventoryClient(HttpClient http) => _http = http;

    public async Task<bool> ReserveAsync(Guid productId, int qty, CancellationToken ct)
    {
        var response = await _http.PostAsJsonAsync("/api/inventory/reserve", new { productId, qty }, ct);
        return response.IsSuccessStatusCode;
    }
}
```

---

## 4) What is API Gateway and why use it?

Short answer:
API Gateway is a single entry point that routes requests to internal services.

Real-time example:
Mobile app calls one gateway endpoint instead of directly calling 10 services.

Explanation:
Gateway handles auth, rate limits, routing, aggregation, and hides internal topology.

```csharp
// Minimal reverse-proxy style example using YARP-like pattern omitted for brevity
// Gateway endpoint can aggregate data from multiple services
app.MapGet("/api/customer/{id:guid}/summary", async (Guid id, CustomerClient c, OrderClient o) =>
{
    var customer = await c.GetAsync(id);
    var orders = await o.GetRecentAsync(id);
    return Results.Ok(new { customer, orders });
});
```

---

## 5) How do you implement service discovery?

Short answer:
Use orchestrator/DNS-based discovery (Kubernetes service DNS) or dedicated registry.

Real-time example:
Order service calls http://inventory-service within cluster; address is resolved by platform.

Explanation:
Avoid hardcoded host/port in distributed environments.

```csharp
builder.Services.AddHttpClient("inventory", client =>
{
    // Kubernetes service DNS name
    client.BaseAddress = new Uri("http://inventory-service");
});
```

---

## 6) What is eventual consistency and where is it useful?

Short answer:
Data across services becomes consistent over time, not immediately.

Real-time example:
Order confirmed now, loyalty points updated a few seconds later via event consumer.

Explanation:
Distributed transactions are expensive. Event-driven consistency is common in microservices.

```csharp
public record OrderPlaced(Guid OrderId, Guid CustomerId, decimal Total);

// Consumer in Loyalty service
public class LoyaltyConsumer
{
    public Task HandleAsync(OrderPlaced evt, CancellationToken ct)
    {
        // Add points eventually after order event arrives
        Console.WriteLine($"Add points for customer {evt.CustomerId}");
        return Task.CompletedTask;
    }
}
```

---

## 7) How do you handle distributed transactions?

Short answer:
Prefer Saga pattern over two-phase commit for cloud-native microservices.

Real-time example:
Travel booking: reserve flight, reserve hotel, then payment; compensate if one step fails.

Explanation:
Saga coordinates local transactions with compensating actions.

```csharp
public class BookingSaga
{
    public async Task ExecuteAsync(CancellationToken ct)
    {
        try
        {
            await ReserveFlightAsync(ct);
            await ReserveHotelAsync(ct);
            await ChargePaymentAsync(ct);
        }
        catch
        {
            // Compensate completed steps
            await CancelHotelAsync(ct);
            await CancelFlightAsync(ct);
            throw;
        }
    }

    private Task ReserveFlightAsync(CancellationToken ct) => Task.CompletedTask;
    private Task ReserveHotelAsync(CancellationToken ct) => Task.CompletedTask;
    private Task ChargePaymentAsync(CancellationToken ct) => Task.CompletedTask;
    private Task CancelHotelAsync(CancellationToken ct) => Task.CompletedTask;
    private Task CancelFlightAsync(CancellationToken ct) => Task.CompletedTask;
}
```

---

## 8) What is the Outbox pattern?

Short answer:
Write business data and integration event in same local transaction, then publish events reliably from outbox table.

Real-time example:
Order saved but broker temporarily down. Outbox ensures OrderPlaced event is not lost.

Explanation:
Prevents dual-write inconsistency between DB and message broker.

```csharp
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; } = string.Empty;
    public string Payload { get; set; } = string.Empty;
    public DateTime CreatedUtc { get; set; }
    public DateTime? PublishedUtc { get; set; }
}

// In same DB transaction:
// 1) Save Order
// 2) Save OutboxMessage(OrderPlaced)
// Background worker publishes unpublished outbox rows.
```

---

## 9) How do you make consumers idempotent?

Short answer:
Track processed message IDs and ignore duplicates.

Real-time example:
PaymentProcessed event may be delivered twice; do not charge twice.

Explanation:
At-least-once delivery requires idempotent handlers.

```csharp
public async Task HandlePaymentProcessedAsync(Guid messageId, Guid orderId, CancellationToken ct)
{
    if (await AlreadyProcessedAsync(messageId, ct))
    {
        return; // duplicate delivery, safely ignore
    }

    // Apply state transition once
    await MarkOrderPaidAsync(orderId, ct);
    await SaveProcessedMarkerAsync(messageId, ct);
}

Task<bool> AlreadyProcessedAsync(Guid id, CancellationToken ct) => Task.FromResult(false);
Task MarkOrderPaidAsync(Guid orderId, CancellationToken ct) => Task.CompletedTask;
Task SaveProcessedMarkerAsync(Guid id, CancellationToken ct) => Task.CompletedTask;
```

---

## 10) What retry strategy should you use for transient failures?

Short answer:
Use bounded retries with exponential backoff and jitter.

Real-time example:
Inventory service has temporary 503 spikes during deployment.

```csharp
using Polly;
using Polly.Extensions.Http;

builder.Services.AddHttpClient("inventory", c => c.BaseAddress = new Uri("http://inventory-service"))
    .AddPolicyHandler(HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(3, attempt => TimeSpan.FromMilliseconds(100 * Math.Pow(2, attempt))));
```

---

## 11) What is a circuit breaker?

Short answer:
A circuit breaker stops repeated calls to a failing dependency for a cool-down window.

Real-time example:
If Payment service is down, Order API fails fast and returns a fallback response.

```csharp
builder.Services.AddHttpClient("payment", c => c.BaseAddress = new Uri("http://payment-service"))
    .AddTransientHttpErrorPolicy(policy =>
        policy.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30))); // open after 5 consecutive faults
```

---

## 12) How do you set timeout and cancellation correctly?

Short answer:
Set per-request timeouts and always pass CancellationToken.

Real-time example:
Client disconnects while report generation API is running.

```csharp
app.MapGet("/api/reports/{id:guid}", async (Guid id, HttpContext ctx, ReportClient client) =>
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ctx.RequestAborted);
    cts.CancelAfter(TimeSpan.FromSeconds(5)); // hard timeout

    var report = await client.GetAsync(id, cts.Token);
    return Results.Ok(report);
});

public class ReportClient
{
    public Task<object> GetAsync(Guid id, CancellationToken ct) => Task.FromResult<object>(new { id, status = "ready" });
}
```

---

## 13) How do you implement API versioning?

Short answer:
Version routes/contracts and keep backward compatibility.

Real-time example:
Mobile app still uses v1 while web frontend adopts v2.

```csharp
// Example route versioning
app.MapGet("/api/v1/products", () => Results.Ok(new[] { new { id = 1, name = "Phone" } }));
app.MapGet("/api/v2/products", () => Results.Ok(new[] { new { id = 1, name = "Phone", currency = "USD" } }));
```

---

## 14) How do you secure service-to-service communication?

Short answer:
Use mTLS or JWT access tokens with scopes and least privilege.

Real-time example:
Order service can call Payment capture endpoint, but not admin refund endpoint.

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("payments.capture", p => p.RequireClaim("scope", "payments.capture"));
});

app.MapPost("/api/payments/capture", [Microsoft.AspNetCore.Authorization.Authorize(Policy = "payments.capture")] () =>
{
    return Results.Ok(new { status = "captured" });
});
```

---

## 15) What is zero-trust in microservices?

Short answer:
Never trust network location; authenticate and authorize every call.

Real-time example:
Even internal pods must present valid identity tokens.

Explanation:
Internal network is not inherently safe in modern cloud environments.

```csharp
// Pseudo-check in middleware
app.Use(async (ctx, next) =>
{
    if (!ctx.User.Identity?.IsAuthenticated ?? true)
    {
        ctx.Response.StatusCode = StatusCodes.Status401Unauthorized;
        return;
    }
    await next();
});
```

---

## 16) How do you implement centralized configuration?

Short answer:
Use appsettings plus environment variables and secret stores.

Real-time example:
Different database/broker settings for dev, staging, production.

```csharp
var builder = WebApplication.CreateBuilder(args);

// appsettings.json + appsettings.{Environment}.json + env vars are loaded by default
string conn = builder.Configuration.GetConnectionString("OrdersDb") ?? "";

Console.WriteLine($"Connection configured: {!string.IsNullOrWhiteSpace(conn)}");
```

---

## 17) What is observability in microservices?

Short answer:
Observability combines logs, metrics, and traces to understand behavior.

Real-time example:
Checkout latency spike traced to slow Inventory dependency.

```csharp
app.Use(async (ctx, next) =>
{
    var sw = System.Diagnostics.Stopwatch.StartNew();
    await next();
    sw.Stop();

    // Structured log for latency analysis
    app.Logger.LogInformation("Path {Path} took {ElapsedMs} ms", ctx.Request.Path, sw.ElapsedMilliseconds);
});
```

---

## 18) Why are correlation IDs important?

Short answer:
Correlation IDs link logs across services for one business transaction.

Real-time example:
One order request traverses Gateway, Order, Inventory, Payment, Notification.

```csharp
app.Use(async (ctx, next) =>
{
    string correlationId = ctx.Request.Headers["X-Correlation-Id"].FirstOrDefault() ?? Guid.NewGuid().ToString();
    ctx.Response.Headers["X-Correlation-Id"] = correlationId;

    using (app.Logger.BeginScope(new Dictionary<string, object> { ["CorrelationId"] = correlationId }))
    {
        await next();
    }
});
```

---

## 19) How do you implement health checks and readiness?

Short answer:
Expose liveness and readiness endpoints and check downstream dependencies for readiness.

Real-time example:
Kubernetes should not route traffic until DB and broker connections are healthy.

```csharp
builder.Services.AddHealthChecks();

app.MapHealthChecks("/health/live");
app.MapHealthChecks("/health/ready");
```

---

## 20) What is API composition?

Short answer:
API composition combines results from multiple services in one response.

Real-time example:
Customer dashboard needs profile, orders, and loyalty points in one call.

```csharp
app.MapGet("/api/dashboard/{customerId:guid}", async (Guid customerId, ProfileClient p, OrdersClient o, LoyaltyClient l) =>
{
    // Fan-out calls
    var profileTask = p.GetAsync(customerId);
    var ordersTask = o.GetRecentAsync(customerId);
    var pointsTask = l.GetPointsAsync(customerId);

    await Task.WhenAll(profileTask, ordersTask, pointsTask);

    return Results.Ok(new
    {
        profile = profileTask.Result,
        orders = ordersTask.Result,
        loyaltyPoints = pointsTask.Result
    });
});

public class ProfileClient { public Task<object> GetAsync(Guid id) => Task.FromResult<object>(new { id, name = "Alex" }); }
public class OrdersClient { public Task<object> GetRecentAsync(Guid id) => Task.FromResult<object>(new[] { new { orderId = 1, total = 99 } }); }
public class LoyaltyClient { public Task<int> GetPointsAsync(Guid id) => Task.FromResult(340); }
```

---

## 21) How do you handle file uploads in microservices?

Short answer:
Stream to object storage and publish an event for downstream processing.

Real-time example:
User uploads image, Media service stores file, Thumbnail service processes asynchronously.

```csharp
app.MapPost("/api/media/upload", async (IFormFile file, CancellationToken ct) =>
{
    if (file.Length == 0) return Results.BadRequest("Empty file.");

    // In real systems: stream directly to blob/S3/minio
    using var stream = file.OpenReadStream();
    // await blobClient.UploadAsync(stream, ct);

    // Publish MediaUploaded event (pseudo)
    // await bus.PublishAsync(new MediaUploaded(...), ct);

    return Results.Accepted();
});
```

---

## 22) What is CQRS and when would you apply it?

Short answer:
CQRS separates write and read models for scalability and clarity.

Real-time example:
Order write model is transactional; analytics read model is denormalized for fast dashboards.

```csharp
// Command side
public record CreateOrderCommand(Guid CustomerId, decimal Total);

// Query side DTO
public record OrderSummaryDto(Guid OrderId, string CustomerName, decimal Total, string Status);

// Separate handlers improve specialization and scaling
```

---

## 23) How do you prevent chatty service calls?

Short answer:
Use coarse-grained APIs, batching, caching, and event-driven updates.

Real-time example:
Checkout page making 20 service calls causes latency; replace with one aggregated endpoint.

```csharp
// Coarse-grained endpoint
app.MapGet("/api/checkout/{customerId:guid}", async (Guid customerId) =>
{
    // Instead of multiple client calls, return pre-aggregated payload
    return Results.Ok(new
    {
        customerId,
        cartItems = 3,
        shippingOptions = new[] { "standard", "express" },
        promo = "SPRING10"
    });
});
```

---

## 24) How do you use caching in microservices?

Short answer:
Use in-memory cache for local hot data and distributed cache for shared state.

Real-time example:
Catalog details change rarely and are read heavily.

```csharp
builder.Services.AddMemoryCache();

app.MapGet("/api/catalog/{id:int}", async (int id, IMemoryCache cache) =>
{
    if (!cache.TryGetValue($"catalog:{id}", out object? item))
    {
        // Simulate DB fetch
        item = new { id, name = "Laptop", price = 999 };
        cache.Set($"catalog:{id}", item, TimeSpan.FromMinutes(5));
    }

    return Results.Ok(item);
});
```

---

## 25) How do you implement rate limiting and throttling?

Short answer:
Apply policy-based limits by IP/user/API key to protect service capacity.

Real-time example:
Public search API is abused by bots.

```csharp
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("search", cfg =>
    {
        cfg.PermitLimit = 60; // 60 requests/minute
        cfg.Window = TimeSpan.FromMinutes(1);
        cfg.QueueLimit = 0;
    });
});

app.UseRateLimiter();

app.MapGet("/api/search", () => Results.Ok(new { result = "ok" }))
   .RequireRateLimiting("search");
```

---

## 26) How do you handle schema evolution in events?

Short answer:
Use versioned events and additive changes; avoid breaking consumers.

Real-time example:
OrderPlaced v2 adds currency while old consumers still process v1 fields.

```csharp
public record OrderPlacedV1(Guid OrderId, decimal Total);
public record OrderPlacedV2(Guid OrderId, decimal Total, string Currency);

// Consumer can support both versions during migration window
public Task HandleAsync(object evt)
{
    return evt switch
    {
        OrderPlacedV2 v2 => ProcessAsync(v2.OrderId, v2.Total, v2.Currency),
        OrderPlacedV1 v1 => ProcessAsync(v1.OrderId, v1.Total, "USD"),
        _ => Task.CompletedTask
    };
}

Task ProcessAsync(Guid id, decimal total, string currency) => Task.CompletedTask;
```

---

## 27) How do you deploy microservices safely?

Short answer:
Use rolling, blue-green, or canary deployments with health probes and automated rollback.

Real-time example:
Deploy new Payment version to 10 percent traffic before full rollout.

Explanation:
Gradual rollout lowers blast radius.

```csharp
// App-side support: expose readiness endpoint used by platform during rollout
app.MapGet("/ready", () => Results.Ok(new { ready = true }));

// Deployment strategy usually configured in platform (Kubernetes/Service Mesh), not in app code.
```

---

## 28) How do you test microservices effectively?

Short answer:
Combine unit tests, contract tests, integration tests, and end-to-end critical-path tests.

Real-time example:
Provider service changes response shape; consumer contract test catches break before production.

```csharp
// Minimal integration test style snippet
// using Microsoft.AspNetCore.Mvc.Testing;
// using Xunit;

// var factory = new WebApplicationFactory<Program>();
// var client = factory.CreateClient();
// var response = await client.GetAsync("/health/live");
// Assert.True(response.IsSuccessStatusCode);
```

---

## 29) How do you prevent cascading failures?

Short answer:
Use bulkheads, timeouts, retries with limits, circuit breakers, and fallback behavior.

Real-time example:
Recommendation service outage should not break checkout.

```csharp
app.MapGet("/api/checkout-summary", async (RecommendationClient rec) =>
{
    object? recommendations;

    try
    {
        recommendations = await rec.GetAsync();
    }
    catch
    {
        // Fallback response avoids full failure
        recommendations = Array.Empty<object>();
    }

    return Results.Ok(new
    {
        checkout = "ok",
        recommendations
    });
});

public class RecommendationClient
{
    public Task<object> GetAsync() => Task.FromResult<object>(new[] { "item-1" });
}
```

---

## 30) What are the top production metrics for Web API and microservices?

Short answer:
Track latency, error rate, throughput, saturation, plus dependency-level metrics.

Real-time example:
Order API p95 latency increased after new release, traced to Payment dependency.

Explanation:
Use SLOs and alerting for p95/p99 latency, 5xx rate, queue lag, and consumer retry counts.

```csharp
app.MapGet("/api/orders/{id:guid}", (Guid id, ILogger<Program> logger) =>
{
    var started = DateTime.UtcNow;

    // Simulated work
    var response = new { id, status = "confirmed" };

    var elapsed = DateTime.UtcNow - started;
    logger.LogInformation("Metric OrderGet latencyMs={LatencyMs}", elapsed.TotalMilliseconds);

    return Results.Ok(response);
});
```

---

## Rapid Fire Senior Follow-ups

1. gRPC vs REST in internal microservices?
2. What is consumer-driven contract testing?
3. How do you choose Kafka vs RabbitMQ?
4. Exactly-once delivery myth vs practical idempotency?
5. How to design dead-letter queue handling?
6. What is backpressure and where to enforce it?
7. How to avoid distributed monolith anti-pattern?
8. How to partition data by tenant or geography?
9. How to trace one request across async event chains?
10. How to run chaos tests for resilience?

---

## Practice Plan

1. Build three services: Order, Inventory, Payment.
2. Add synchronous reserve-stock call plus async OrderPlaced event.
3. Implement Outbox in Order service and idempotent consumer in Payment service.
4. Add gateway aggregation endpoint and correlation IDs.
5. Add retry, circuit breaker, rate limit, health checks, and logs.
