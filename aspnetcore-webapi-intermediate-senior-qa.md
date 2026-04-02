# ASP.NET Core Web API Interview Q&A (Intermediate to Senior, 30 Questions)

This guide is for developers with intermediate to senior experience in ASP.NET Core Web API.
Each question includes:
- Short answer
- Explanation
- Practical C# code snippet with comments

---

## 1) What is ASP.NET Core Web API, and when should you use it?

**Short answer:**
ASP.NET Core Web API is a framework for building HTTP APIs (RESTful or otherwise) on .NET.

**Explanation:**
Use it for backend services consumed by web apps, mobile apps, microservices, and third-party integrations.

```csharp
var builder = WebApplication.CreateBuilder(args);

// Registers MVC controllers and API features
builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();

app.Run();
```

---

## 2) What is the request pipeline in ASP.NET Core?

**Short answer:**
The request pipeline is an ordered chain of middleware components processing each HTTP request.

**Explanation:**
Order matters. For example, authentication should run before authorization.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAuthentication();
builder.Services.AddAuthorization();
builder.Services.AddControllers();

var app = builder.Build();

app.UseRouting();
app.UseAuthentication(); // identify user first
app.UseAuthorization();  // then enforce policies

app.MapControllers();
app.Run();
```

---

## 3) What are the differences between `IServiceCollection`, `IServiceProvider`, and DI lifetimes?

**Short answer:**
- `IServiceCollection`: where services are registered.
- `IServiceProvider`: where services are resolved.
- Lifetimes: Singleton, Scoped, Transient.

```csharp
builder.Services.AddSingleton<IClock, SystemClock>();     // one for app lifetime
builder.Services.AddScoped<IUserContext, UserContext>();   // one per request
builder.Services.AddTransient<IEmailSender, EmailSender>(); // new every resolve
```

---

## 4) What is the role of `[ApiController]`?

**Short answer:**
`[ApiController]` enables API-specific conventions like automatic model validation and binding behavior.

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public IActionResult Create([FromBody] CreateProductRequest request)
    {
        // If model is invalid, framework automatically returns 400 with ProblemDetails
        return Ok(new { Message = "Created" });
    }
}

public class CreateProductRequest
{
    public string Name { get; set; } = string.Empty;
}
```

---

## 5) How does model binding work in Web API?

**Short answer:**
Model binding maps request data (route, query, headers, body, form) to action parameters.

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id:int}")]
    public IActionResult Get(
        [FromRoute] int id,
        [FromQuery] bool includeItems = false,
        [FromHeader(Name = "X-Correlation-Id")] string? correlationId = null)
    {
        return Ok(new { id, includeItems, correlationId });
    }
}
```

---

## 6) How do you validate request models?

**Short answer:**
Use data annotations and custom validators; return consistent validation errors.

```csharp
using System.ComponentModel.DataAnnotations;

public class RegisterRequest
{
    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;

    [Required]
    [MinLength(8)]
    public string Password { get; set; } = string.Empty;
}

[ApiController]
[Route("api/auth")]
public class AuthController : ControllerBase
{
    [HttpPost("register")]
    public IActionResult Register(RegisterRequest request)
    {
        // With [ApiController], invalid model auto returns 400
        return Ok(new { Message = "Registered" });
    }
}
```

---

## 7) What are action results (`IActionResult` vs `ActionResult<T>`) and when to use each?

**Short answer:**
Use `ActionResult<T>` for typed success responses plus flexible error responses.

```csharp
[HttpGet("{id:int}")]
public ActionResult<ProductDto> GetById(int id)
{
    var product = new ProductDto { Id = id, Name = "Laptop" };

    if (id <= 0)
    {
        return BadRequest("Invalid id");
    }

    return Ok(product); // typed success response
}

public class ProductDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}
```

---

## 8) How do you version APIs in ASP.NET Core?

**Short answer:**
Use URL versioning, query/header versioning, and clearly maintain backward compatibility.

```csharp
// NuGet: Asp.Versioning.Mvc
builder.Services.AddApiVersioning(options =>
{
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.DefaultApiVersion = new Asp.Versioning.ApiVersion(1, 0);
    options.ReportApiVersions = true;
});

[ApiController]
[Route("api/v{version:apiVersion}/products")]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
public class VersionedProductsController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IActionResult GetV1() => Ok(new { Version = "v1" });

    [HttpGet]
    [MapToApiVersion("2.0")]
    public IActionResult GetV2() => Ok(new { Version = "v2", Includes = "new fields" });
}
```

---

## 9) How do you implement global exception handling?

**Short answer:**
Use centralized exception middleware to avoid try/catch in every controller.

```csharp
using System.Net;
using System.Text.Json;

app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exceptionFeature = context.Features.Get<Microsoft.AspNetCore.Diagnostics.IExceptionHandlerPathFeature>();
        var ex = exceptionFeature?.Error;

        context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
        context.Response.ContentType = "application/json";

        var payload = new
        {
            title = "Unexpected error",
            detail = ex?.Message,
            status = context.Response.StatusCode
        };

        await context.Response.WriteAsync(JsonSerializer.Serialize(payload));
    });
});
```

---

## 10) What is `ProblemDetails` and why use it?

**Short answer:**
`ProblemDetails` is a standard RFC 7807 format for machine-readable API errors.

```csharp
[HttpGet("{id:int}")]
public IActionResult GetItem(int id)
{
    if (id < 1)
    {
        return Problem(
            title: "Invalid id",
            detail: "Id must be greater than zero.",
            statusCode: StatusCodes.Status400BadRequest);
    }

    return Ok(new { id });
}
```

---

## 11) How do authentication and authorization differ?

**Short answer:**
Authentication verifies identity; authorization checks permissions.

```csharp
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://identity.example.com";
        options.Audience = "api";
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));
});

[Authorize(Policy = "AdminOnly")]
[HttpDelete("{id:int}")]
public IActionResult Delete(int id) => NoContent();
```

---

## 12) How do you design secure JWT-based APIs?

**Short answer:**
Validate issuer/audience/signing key, enforce expiration, and avoid storing sensitive claims.

```csharp
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.TokenValidationParameters = new Microsoft.IdentityModel.Tokens.TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = "https://issuer.example.com",
            ValidAudience = "my-api",
            IssuerSigningKey = new Microsoft.IdentityModel.Tokens.SymmetricSecurityKey(
                System.Text.Encoding.UTF8.GetBytes("super-secret-signing-key-12345"))
        };
    });
```

---

## 13) How do you use policy-based authorization?

**Short answer:**
Define reusable policies with requirements and handlers for fine-grained access control.

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("MinimumAge18", policy =>
        policy.RequireClaim("age", "18", "19", "20"));
});

[Authorize(Policy = "MinimumAge18")]
[HttpGet("restricted")]
public IActionResult Restricted() => Ok("Allowed");
```

---

## 14) How do you implement CORS correctly?

**Short answer:**
Whitelist trusted origins and limit methods/headers instead of allowing everything.

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("FrontendPolicy", policy =>
    {
        policy.WithOrigins("https://app.example.com")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .AllowAnyHeader();
    });
});

app.UseCors("FrontendPolicy");
```

---

## 15) What is rate limiting and how do you apply it?

**Short answer:**
Rate limiting controls request volume to protect APIs from abuse and spikes.

```csharp
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", limiterOptions =>
    {
        limiterOptions.PermitLimit = 100; // max 100 requests
        limiterOptions.Window = TimeSpan.FromMinutes(1);
        limiterOptions.QueueLimit = 0;
        limiterOptions.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });
});

app.UseRateLimiter();

[EnableRateLimiting("fixed")]
[HttpGet("limited")]
public IActionResult Limited() => Ok("Rate limited endpoint");
```

---

## 16) How do you implement response caching and output caching?

**Short answer:**
Use output caching in .NET 7+ for API responses where data does not change often.

```csharp
builder.Services.AddOutputCache();

app.UseOutputCache();

[HttpGet("public-products")]
[OutputCache(Duration = 30)] // cache for 30 seconds
public IActionResult GetPublicProducts()
{
    return Ok(new[] { "Laptop", "Mouse", "Keyboard" });
}
```

---

## 17) What is the repository pattern in Web API, and when should you avoid extra abstraction?

**Short answer:**
Repository abstracts data access. Useful for complex domain boundaries, but avoid unnecessary wrappers over EF Core in simple apps.

```csharp
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(int id, CancellationToken ct);
}

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _db;

    public ProductRepository(AppDbContext db) => _db = db;

    public Task<Product?> GetByIdAsync(int id, CancellationToken ct)
        => _db.Products.FindAsync(new object?[] { id }, ct).AsTask();
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}

public class AppDbContext
{
    public FakeDbSet<Product> Products { get; } = new();
}

public class FakeDbSet<T> where T : class, new()
{
    public ValueTask<T?> FindAsync(object?[] keyValues, CancellationToken ct = default)
        => ValueTask.FromResult<T?>(new T());
}
```

---

## 18) How do you avoid N+1 query problems in EF Core APIs?

**Short answer:**
Use projection and eager loading carefully (`Select`, `Include`) to fetch required data in fewer queries.

```csharp
// Better: project only what API needs
// var orders = await _db.Orders
//     .Select(o => new OrderDto
//     {
//         Id = o.Id,
//         CustomerName = o.Customer.Name,
//         ItemCount = o.Items.Count
//     })
//     .ToListAsync(ct);

// Avoid lazy-loading each relationship in loops (N+1 risk)
```

---

## 19) How do you use DTOs and AutoMapper (or manual mapping)?

**Short answer:**
Use DTOs to avoid exposing domain entities directly and to control contract shape.

```csharp
public class ProductEntity
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal CostPrice { get; set; } // internal, do not expose
}

public class ProductResponseDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}

// Manual mapping example
public static ProductResponseDto ToDto(ProductEntity e) => new()
{
    Id = e.Id,
    Name = e.Name
};
```

---

## 20) How do you implement pagination, filtering, and sorting in APIs?

**Short answer:**
Accept query parameters, validate them, and apply on `IQueryable` before materializing.

```csharp
[HttpGet]
public IActionResult Search([FromQuery] int page = 1, [FromQuery] int pageSize = 20)
{
    page = Math.Max(page, 1);
    pageSize = Math.Clamp(pageSize, 1, 100);

    var all = Enumerable.Range(1, 500).Select(i => new { Id = i, Name = $"Item {i}" });

    var data = all.Skip((page - 1) * pageSize).Take(pageSize).ToList();

    return Ok(new
    {
        page,
        pageSize,
        total = all.Count(),
        data
    });
}
```

---

## 21) What are idempotency and safe HTTP methods?

**Short answer:**
- Safe: `GET`, `HEAD` (read-only).
- Idempotent: multiple same requests produce same effect (`PUT`, `DELETE`, `GET`, `HEAD`).

```csharp
[HttpPut("{id:int}")]
public IActionResult Upsert(int id, UpdateProductRequest request)
{
    // Repeating same PUT should produce same resource state
    return NoContent();
}

public class UpdateProductRequest
{
    public string Name { get; set; } = string.Empty;
}
```

---

## 22) How do you handle long-running operations in Web APIs?

**Short answer:**
Prefer asynchronous processing and return `202 Accepted` with operation tracking.

```csharp
[HttpPost("reports")]
public IActionResult GenerateReport()
{
    var operationId = Guid.NewGuid().ToString();

    // Enqueue background job (queue/message bus in real systems)

    return Accepted(new
    {
        operationId,
        statusUrl = $"/api/operations/{operationId}"
    });
}
```

---

## 23) How do background tasks work in ASP.NET Core (`IHostedService`)?

**Short answer:**
Use hosted services for recurring or queue-based background processing.

```csharp
using Microsoft.Extensions.Hosting;

public class MetricsWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Periodic task
            Console.WriteLine("Collecting metrics...");
            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }
    }
}

// Registration:
// builder.Services.AddHostedService<MetricsWorker>();
```

---

## 24) How do you implement health checks?

**Short answer:**
Use health checks to expose service readiness/liveness endpoints.

```csharp
builder.Services.AddHealthChecks();

app.MapHealthChecks("/health");

// In production, also add DB/Redis/queue health checks and split readiness vs liveness.
```

---

## 25) How do you add logging and correlation IDs?

**Short answer:**
Use structured logging and propagate a correlation ID across requests.

```csharp
app.Use(async (context, next) =>
{
    var correlationId = context.Request.Headers["X-Correlation-Id"].FirstOrDefault()
                        ?? Guid.NewGuid().ToString();

    context.Response.Headers["X-Correlation-Id"] = correlationId;

    using (app.Logger.BeginScope(new Dictionary<string, object>
    {
        ["CorrelationId"] = correlationId
    }))
    {
        app.Logger.LogInformation("Handling request {Path}", context.Request.Path);
        await next();
    }
});
```

---

## 26) How do you produce OpenAPI/Swagger docs?

**Short answer:**
Use `AddEndpointsApiExplorer` + `AddSwaggerGen`, and document endpoints with XML comments/attributes.

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

---

## 27) What are Minimal APIs vs Controllers?

**Short answer:**
Minimal APIs are lightweight and concise; controllers provide richer organization for larger, structured APIs.

```csharp
// Minimal API endpoint
app.MapGet("/api/ping", () => Results.Ok(new { message = "pong" }));

// Controller style is preferred when API grows with many endpoints and filters.
```

---

## 28) How do filters differ from middleware?

**Short answer:**
Middleware runs for all requests in the pipeline; filters run inside MVC action execution pipeline.

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

public class AuditActionFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        // Runs before controller action
        Console.WriteLine("Action starting...");
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        // Runs after controller action
        Console.WriteLine("Action completed.");
    }
}

// Register globally:
// builder.Services.AddControllers(options => options.Filters.Add<AuditActionFilter>());
```

---

## 29) How do you test ASP.NET Core Web APIs?

**Short answer:**
Write unit tests for business logic and integration tests for real HTTP pipeline behavior.

```csharp
using System.Net;
using Microsoft.AspNetCore.Mvc.Testing;
using Xunit;

public class ApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Ping_ReturnsOk()
    {
        var response = await _client.GetAsync("/api/ping");

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
}
```

---

## 30) How do you design APIs for microservices and resilience?

**Short answer:**
Use clear bounded contexts, async communication where needed, retries/circuit breakers, observability, and backward-compatible contracts.

```csharp
using Polly;
using Polly.Extensions.Http;

builder.Services.AddHttpClient("CatalogApi", client =>
{
    client.BaseAddress = new Uri("https://catalog-service");
})
.AddPolicyHandler(HttpPolicyExtensions
    .HandleTransientHttpError()
    .WaitAndRetryAsync(3, retry => TimeSpan.FromMilliseconds(200 * retry))); // retry transient failures

// Add tracing/logging/metrics for end-to-end observability.
```

---

## Rapid Fire Senior Follow-ups

1. `UseRouting` vs endpoint mapping order in modern hosting model?
2. How do you return `409 Conflict` and when?
3. ETag and conditional requests in APIs?
4. How do you implement optimistic concurrency with EF Core `RowVersion`?
5. Why and when to use `IAsyncEnumerable<T>` in APIs?
6. How do you secure internal service-to-service calls?
7. How do you roll out breaking API changes safely?
8. Circuit breaker vs retry: when each?
9. When to use gRPC over REST?
10. How to handle distributed transactions without 2PC?

---

## Practice Plan

1. Implement 5 endpoints with validation + ProblemDetails.
2. Add JWT auth + one policy-based authorization rule.
3. Add pagination, global exception handling, and output caching.
4. Add Swagger docs and integration tests.
5. Add rate limiting + health checks + correlation ID logging.
