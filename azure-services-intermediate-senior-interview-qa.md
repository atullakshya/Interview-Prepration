# Azure Services - Interview Q&A (Intermediate to Senior Level)

## 1. What are Azure Functions and how do they differ from traditional application hosting?

**Short Answer:**
Azure Functions are serverless compute that run event-driven code without managing infrastructure. They scale automatically and charge by execution time, unlike Web Apps which run continuously and charge by compute resource allocation.

**Real-Time Example:**
```csharp
// Azure Function triggered by HTTP request
[Function("ProcessOrder")]
public static async Task<IActionResult> ProcessOrder(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "orders")] HttpRequest req,
    [ServiceBus("order-queue", Connection = "ServiceBusConnection")] IAsyncCollector<Order> orderQueue,
    ILogger log)
{
    var orderData = await req.Body.ReadAsAsync<Order>();
    orderData.CreatedAt = DateTime.UtcNow;
    
    // Process and queue for async handling
    await orderQueue.AddAsync(orderData);
    
    return new OkObjectResult(new { id = orderData.Id, status = "queued" });
}

// Timer-triggered function for scheduled tasks
[Function("DailyReconciliation")]
public static void DailyReconciliation(
    [TimerTrigger("0 2 * * *")] TimerInfo timer, // 2 AM daily
    ILogger log)
{
    log.LogInformation($"Running reconciliation at {DateTime.UtcNow}");
    // Perform batch operations
}
```

**Explanation:**
Azure Functions excel at event-driven scenarios (HTTP, queue triggers, blob changes) where you need auto-scaling without server management. Traditional Web Apps are better for continuous applications requiring persistent connections or complex stateful logic. Functions are cost-effective for variable workloads; Web Apps for predictable, steady-state traffic.

**Interview Tips:**
- Discuss cold starts and warm-up strategies for production scenarios
- Mention durable functions for complex orchestration
- Address limits: 10-minute timeout (consumption plan), 230MB memory
- Explain bindings reduce boilerplate code compared to SDKs

---

## 2. What are Azure Storage Account types and when would you use each?

**Short Answer:**
Five Azure Storage types: General Purpose v2 (blobs, queues, tables, files), Premium Block Blobs (high-throughput), Premium File Shares (SMB/NFS), Premium Append Blobs (logging), and Data Lake Storage Gen2 (analytics). Choose by access patterns, performance tiers, and redundancy needs.

**Real-Time Example:**
```csharp
// Using Azure Storage - Blob, Queue, Table
using Azure.Storage.Blobs;
using Azure.Storage.Queues;
using Azure.Data.Tables;

public class StorageService
{
    private readonly BlobContainerClient _blobClient;
    private readonly QueueClient _queueClient;
    private readonly TableClient _tableClient;

    public StorageService(string connectionString)
    {
        // Blob Storage for documents
        var blobServiceClient = new BlobServiceClient(connectionString);
        _blobClient = blobServiceClient.GetBlobContainerClient("documents");

        // Queue Storage for async messaging
        var queueServiceClient = new QueueServiceClient(connectionString);
        _queueClient = queueServiceClient.GetQueueClient("order-processing");

        // Table Storage for semi-structured data
        var tableServiceClient = new TableServiceClient(connectionString);
        _tableClient = tableServiceClient.GetTableClient("Orders");
    }

    // Upload blob with metadata and tier management
    public async Task UploadDocument(Stream fileStream, string fileName)
    {
        var blobClient = _blobClient.GetBlobClient(fileName);
        
        var uploadOptions = new BlobUploadOptions
        {
            Metadata = new Dictionary<string, string>
            {
                { "uploadedAt", DateTime.UtcNow.ToString("O") },
                { "department", "finance" }
            }
        };

        await blobClient.UploadAsync(fileStream, overwrite: true, options: uploadOptions);
        
        // Set access tier for archival (Premium = Hot, Cool, Archive)
        await blobClient.SetAccessTierAsync(AccessTier.Cool);
    }

    // Queue message for background processing
    public async Task QueueOrderForProcessing(Order order)
    {
        string messageBody = JsonSerializer.Serialize(order);
        
        // Visibility timeout prevents duplicate processing
        await _queueClient.SendMessageAsync(
            messageBody,
            visibilityTimeout: TimeSpan.FromMinutes(5));
    }

    // Table Storage for entities
    public async Task StoreOrderEntity(Order order)
    {
        var entity = new TableEntity("finance", order.Id.ToString())
        {
            { "Amount", order.Amount },
            { "Status", "pending" },
            { "CreatedAt", DateTime.UtcNow },
            { "Timestamp", DateTime.UtcNow }
        };

        await _tableClient.AddEntityAsync(entity);
    }
}
```

**Explanation:**
- **Blob Storage**: Large files (documents, media), cost-effective, supports hierarchical organization and tiering (Hot → Cool → Archive for cost savings)
- **Queue Storage**: Asynchronous messaging, guarantees delivery, supports visibility timeout and dead-letter queues
- **Table Storage**: NoSQL key-value store for entity data, cheaper than SQL Database for semi-structured data, good for logs
- **File Shares**: Network storage for on-premises access via SMB/NFS
- **Data Lake Storage Gen2**: Analytics workloads with Hadoop-compatible filesystem

**Interview Tips:**
- Explain redundancy options (LRS, GRS, RAGRS, ZRS) impact on RPO/RTO
- Discuss hot/cool/archive tiers for cost optimization over time
- Mention SAS tokens for secure access without connection strings
- Address Table Storage vs Cosmos DB trade-offs

---

## 3. How do you manage secrets in Azure Key Vault securely?

**Short Answer:**
Azure Key Vault stores secrets, keys, and certificates with encryption at rest and in transit. Access via managed identities (preferred), RBAC, and audit logging prevents credential exposure in code and logs.

**Real-Time Example:**
```csharp
// Secure secret management with Key Vault
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using Microsoft.Extensions.Azure;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

public static void ConfigureKeyVault(IServiceCollection services, IConfiguration config)
{
    // Option 1: Using Managed Identity (recommended for Azure-hosted apps)
    var keyVaultUrl = "https://myvault.vault.azure.net/";
    var credential = new DefaultAzureCredential(); // Auto-detects environment
    
    // Add Key Vault as configuration source
    config.AddAzureKeyVault(
        new Uri(keyVaultUrl),
        credential,
        new KeyVaultSecretManager()); // Optional: filter which secrets to load
}

public class DatabaseService
{
    private readonly SecretClient _secretClient;

    public DatabaseService(SecretClient secretClient)
    {
        _secretClient = secretClient;
    }

    // Retrieve secret (cached by Azure SDK)
    public async Task<string> GetConnectionString()
    {
        try
        {
            KeyVaultSecret secret = await _secretClient.GetSecretAsync("db-connection-string");
            return secret.Value;
        }
        catch (Azure.RequestFailedException ex) when (ex.Status == 403)
        {
            throw new UnauthorizedAccessException("Managed identity lacks Key Vault access", ex);
        }
    }

    // Retrieve certificate for mTLS
    public async Task<X509Certificate2> GetClientCertificate()
    {
        KeyVaultCertificate cert = await _secretClient.GetCertificateAsync("client-cert");
        // Certificate is automatically converted to X509Certificate2
        return cert.Properties.Id; // Use cert ID with Key Vault for decryption
    }
}

// Safe configuration - NO hardcoded secrets
public class OrderService
{
    private readonly IConfiguration _config;

    public OrderService(IConfiguration config)
    {
        _config = config; // Contains Key Vault secrets
    }

    public string ApiKey => _config["api-key"] ?? throw new InvalidOperationException("Missing api-key");
}

// Startup configuration with managed identity
public static class KeyVaultSetup
{
    public static WebApplicationBuilder AddSecureConfiguration(this WebApplicationBuilder builder)
    {
        var keyVaultUrl = builder.Configuration["KeyVault:Url"];
        
        if (!string.IsNullOrEmpty(keyVaultUrl))
        {
            // Managed Identity authentication (no credentials needed in code)
            var credential = new DefaultAzureCredential(
                new DefaultAzureCredentialOptions
                {
                    ExcludeManagedIdentityCredential = false,
                    ExcludeAzureCliCredential = true // Only use in development
                });

            builder.Configuration.AddAzureKeyVault(
                new Uri(keyVaultUrl),
                credential);

            // Attach Key Vault endpoint to logging
            builder.Services.AddApplicationInsightsTelemetry(options =>
            {
                options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
            });
        }

        return builder;
    }
}
```

**Explanation:**
- **Managed Identity**: Azure automatically manages credentials; no keys in code
- **RBAC**: Fine-grained access control (who can read secrets vs. manage them)
- **Audit Logging**: All Key Vault operations logged in Azure Monitor
- **Versioning**: Rotate secrets without downtime by versioning
- **Soft Delete**: Accidental deletions recoverable (90 days)

**Interview Tips:**
- Explain DefaultAzureCredential chain: Managed Identity → CLI → Device Code → Interactive
- Discuss key rotation strategies and versioning
- Address compliance requirements (encryption, audit trails)
- Mention certificate auto-renewal for PKI scenarios
- Discuss soft delete and purge protection for compliance

---

## 4. What is Application Insights and how would you instrument an application?

**Short Answer:**
Application Insights is Azure's APM (Application Performance Monitoring) service that collects telemetry: dependencies, performance metrics, exceptions, and custom events. Use the .NET SDK for automatic and manual instrumentation with minimal code.

**Real-Time Example:**
```csharp
// Application Insights instrumentation setup
using Microsoft.ApplicationInsights;
using Microsoft.ApplicationInsights.DataContracts;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Configuration;

public static class StartupExtensions
{
    public static WebApplicationBuilder AddApplicationInsights(
        this WebApplicationBuilder builder)
    {
        // Automatic instrumentation
        builder.Services.AddApplicationInsightsTelemetry(options =>
        {
            options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
            options.EnableAdaptiveSampling = true; // Reduce costs for high-volume apps
            options.EnableDependencyTrackingTelemetryModule = true;
            options.EnableRequestTrackingTelemetryModule = true;
        });

        // Add performance counters
        builder.Services.ConfigureTelemetryModule<DependencyTrackingTelemetryModule>(
            (module, options) =>
            {
                module.DisableTelemetryModule = false;
                module.EnableSqlCommandTextInstrumentation = true;
            });

        return builder;
    }
}

// Custom instrumentation in service
public class OrderProcessingService
{
    private readonly TelemetryClient _telemetryClient;

    public OrderProcessingService(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    public async Task ProcessOrder(Order order)
    {
        // Track custom operation with duration
        using (var operation = _telemetryClient.StartOperation<DependencyTelemetry>("ProcessOrder"))
        {
            operation.Telemetry.Type = "SQL";
            operation.Telemetry.Target = "OrderDB";

            try
            {
                var stopwatch = System.Diagnostics.Stopwatch.StartNew();
                
                // Business logic
                await PaymentGateway.ChargeOrder(order);
                await Database.SaveOrder(order);

                stopwatch.Stop();
                operation.Telemetry.Duration = stopwatch.Elapsed;
                operation.Telemetry.Success = true;

                // Custom event tracking
                _telemetryClient.TrackEvent("OrderProcessed", new Dictionary<string, string>
                {
                    { "OrderId", order.Id },
                    { "Amount", order.Amount.ToString() },
                    { "ProcessingTime", stopwatch.ElapsedMilliseconds.ToString() }
                }, new Dictionary<string, double>
                {
                    { "ProcessingMs", stopwatch.ElapsedMilliseconds },
                    { "OrderAmount", order.Amount }
                });
            }
            catch (Exception ex)
            {
                operation.Telemetry.Success = false;
                
                // Track exception with custom properties
                _telemetryClient.TrackException(ex, new Dictionary<string, string>
                {
                    { "OrderId", order.Id },
                    { "Stage", "OrderProcessing" }
                });

                throw;
            }
            finally
            {
                _telemetryClient.StopOperation(operation);
            }
        }
    }

    // Track availability (synthetic monitoring)
    public async Task<bool> HealthCheck()
    {
        var availabilityTelemetry = new AvailabilityTelemetry
        {
            Name = "OrderService.HealthCheck",
            RunLocation = "US-East",
            Success = false,
            Timestamp = DateTimeOffset.UtcNow
        };

        var stopwatch = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            var response = await Database.Query("SELECT 1");
            availabilityTelemetry.Success = response != null;
        }
        catch (Exception ex)
        {
            availabilityTelemetry.Message = ex.Message;
        }
        finally
        {
            stopwatch.Stop();
            availabilityTelemetry.Duration = stopwatch.Elapsed;
            _telemetryClient.TrackAvailability(availabilityTelemetry);
        }

        return availabilityTelemetry.Success;
    }
}

// Analytics Query (KQL - Kusto Query Language)
/*
// Query slow requests in App Insights
requests
| where duration > 5000
| summarize 
    count_slow = count(),
    avg_duration = avg(duration),
    p95_duration = percentile(duration, 95)
    by name, resultCode
| order by count_slow desc

// Track dependency failures
dependencies
| where success == false
| summarize count() by type, target
*/
```

**Explanation:**
- **Automatic Instrumentation**: HTTP requests, SQL queries, external dependencies tracked without code
- **Custom Events**: Business metrics (orders processed, users registered)
- **Custom Metrics**: Performance indicators (processing time, queue depth)
- **Sampling**: Reduce costs by sampling data (adaptive or fixed %)
- **KQL Queries**: Analyze telemetry with Kusto Query Language
- **Live Metrics**: Real-time view of application health

**Interview Tips:**
- Explain sampling strategies impact on cost vs. data quality
- Discuss correlation IDs for tracking distributed calls
- Address GDPR concerns with PII filtering
- Mention custom dimensions for business context
- Discuss alert creation for SLA violations

---

## 5. How does Azure Service Bus work and what is the difference between queues and topics?

**Short Answer:**
Azure Service Bus provides reliable messaging with FIFO guarantees, dead-letter queues, and distributed transaction support. Queues are 1-to-many (consumer consumes and deletes); Topics support pub-sub with multiple independent subscribers via subscriptions.

**Real-Time Example:**
```csharp
// Azure Service Bus - Queues vs Topics
using Azure.Messaging.ServiceBus;
using Microsoft.Extensions.DependencyInjection;

public static class ServiceBusSetup
{
    public static IServiceCollection AddServiceBus(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddSingleton(new ServiceBusClient(connectionString));
        return services;
    }
}

// Queue pattern - Point-to-point messaging
public class OrderQueueService
{
    private readonly ServiceBusClient _client;
    private const string QueueName = "order-processing";

    public OrderQueueService(ServiceBusClient client) => _client = client;

    // Producer: Send message
    public async Task SendOrder(Order order)
    {
        var sender = _client.CreateSender(QueueName);
        try
        {
            var message = new ServiceBusMessage(JsonSerializer.Serialize(order))
            {
                MessageId = order.Id.ToString(),
                ContentType = "application/json",
                Subject = "NewOrder",
                TimeToLive = TimeSpan.FromHours(24), // Auto-delete if not processed
                CorrelationId = Guid.NewGuid().ToString() // Track across services
            };

            await sender.SendMessageAsync(message);
        }
        finally
        {
            await sender.DisposeAsync();
        }
    }

    // Consumer: Process messages sequentially
    public async Task ProcessOrders(CancellationToken cancellation)
    {
        var receiver = _client.CreateReceiver(QueueName);
        try
        {
            while (!cancellation.IsCancellationRequested)
            {
                // Receive with lock timeout (30 seconds)
                ServiceBusReceivedMessage message = await receiver.ReceiveMessageAsync(
                    timeoutInSeconds: 1,
                    cancellationToken: cancellation);

                if (message == null) continue;

                try
                {
                    var order = JsonSerializer.Deserialize<Order>(message.Body.ToString());
                    await ProcessOrderLogic(order);
                    
                    // Mark as processed - removes from queue
                    await receiver.CompleteMessageAsync(message);
                }
                catch (Exception ex)
                {
                    // Requeue for retry (max 10 times before dead-letter)
                    await receiver.AbandonMessageAsync(message);
                    
                    // Or move to dead-letter queue on final failure
                    if (message.DeliveryCount > 3)
                        await receiver.DeadLetterMessageAsync(message, "MaxRetriesExceeded", ex.Message);
                }
            }
        }
        finally
        {
            await receiver.DisposeAsync();
        }
    }
}

// Topic pattern - Publish-subscribe messaging
public class OrderEventService
{
    private readonly ServiceBusClient _client;
    private const string TopicName = "order-events";
    private const string InvoicingSubscription = "invoicing";
    private const string ShippingSubscription = "shipping";
    private const string AnalyticsSubscription = "analytics";

    public OrderEventService(ServiceBusClient client) => _client = client;

    // Publisher: Multiple subscribers get same message
    public async Task PublishOrderCreated(Order order)
    {
        var sender = _client.CreateSender(TopicName);
        try
        {
            var message = new ServiceBusMessage(JsonSerializer.Serialize(order))
            {
                MessageId = order.Id.ToString(),
                Subject = "OrderCreated",
                CorrelationId = Guid.NewGuid().ToString(),
                ApplicationProperties =
                {
                    { "Amount", order.Amount },
                    { "Priority", order.IsUrgent ? "High" : "Normal" }
                }
            };

            await sender.SendMessageAsync(message);
        }
        finally
        {
            await sender.DisposeAsync();
        }
    }

    // Subscriber 1: Invoicing service
    public async Task ProcessInvoicing()
    {
        var receiver = _client.CreateReceiver(TopicName, InvoicingSubscription);
        try
        {
            while (true)
            {
                var message = await receiver.ReceiveMessageAsync(TimeSpan.FromSeconds(5));
                if (message == null) continue;

                try
                {
                    var order = JsonSerializer.Deserialize<Order>(message.Body.ToString());
                    await GenerateInvoice(order);
                    await receiver.CompleteMessageAsync(message);
                }
                catch
                {
                    await receiver.AbandonMessageAsync(message);
                }
            }
        }
        finally
        {
            await receiver.DisposeAsync();
        }
    }

    // Subscriber 2: Shipping service (independent processing)
    public async Task ProcessShipping()
    {
        var receiver = _client.CreateReceiver(TopicName, ShippingSubscription);
        // Same pattern but independent logic
        // Failure in invoicing doesn't affect shipping
    }

    // Advanced: Subscription filters and correlation
    public async Task SetupSubscriptionFilters()
    {
        var managementClient = new ServiceBusAdministrationClient(
            _client.FullyQualifiedNamespace, new Azure.Identity.DefaultAzureCredential());

        // Only process high-priority orders in analytics
        await managementClient.CreateOrUpdateSubscriptionRuntimePropertiesAsync(
            TopicName,
            AnalyticsSubscription,
            new CreateSubscriptionOptions(AnalyticsSubscription)
            {
                Filter = new CorrelationRuleFilter { ApplicationProperties = { { "Priority", "High" } } }
            });
    }
}

// Sessions for ordered message processing
public class OrderSessionService
{
    private readonly ServiceBusClient _client;
    private const string SessionQueueName = "order-processing-sessions";

    public async Task SendOrderBatch(string customerId, List<Order> orders)
    {
        var sender = _client.CreateSender(SessionQueueName);
        
        foreach (var order in orders)
        {
            // SessionId ensures FIFO processing per customer
            var message = new ServiceBusMessage(JsonSerializer.Serialize(order))
            {
                SessionId = customerId, // Orders for same customer processed sequentially
                MessageId = order.Id.ToString()
            };

            await sender.SendMessageAsync(message);
        }

        await sender.DisposeAsync();
    }

    public async Task ProcessOrdersByCustomer()
    {
        var client = _client.CreateSessionReceiver(SessionQueueName);
        
        while (true)
        {
            // Gets all messages for single customer (session)
            var session = await client.AcceptSessionAsync(TimeSpan.FromSeconds(1));
            if (session == null) continue;

            try
            {
                while (true)
                {
                    var message = await session.ReceiveMessageAsync(TimeSpan.FromSeconds(5));
                    if (message == null) break;

                    var order = JsonSerializer.Deserialize<Order>(message.Body.ToString());
                    await ProcessOrder(order);
                    await session.CompleteMessageAsync(message);
                }
            }
            finally
            {
                await session.CloseAsync();
            }
        }
    }
}
```

**Explanation:**
- **Queue**: One consumer per message (point-to-point), FIFO ordering, good for task distribution
- **Topic + Subscriptions**: Multiple subscribers get same message (pub-sub), broadcasts to all, good for events
- **Sessions**: Force ordering per customer/group, prevents race conditions
- **Dead-Letter Queue**: Failed messages after retries, investigate pattern
- **Message TTL**: Auto-expiration prevents stale data
- **Delivery Count**: Tracks retry attempts, circuit-break after max

**Interview Tips:**
- Discuss transaction support with Service Bus vs. Event Hub
- Explain subscription filters for message routing
- Address idempotent processing to handle duplicate delivery
- Mention batching strategies for throughput
- Discuss competing consumers vs. parallel subscribers pattern

---

## 6. What is Azure App Service (Web App) and how do you scale applications?

**Short Answer:**
Azure App Service manages web application hosting with automatic scaling, deployment slots, and integrated monitoring. Scale vertically (larger instance) or horizontally (more instances) via autoscale rules based on metrics or schedules.

**Real-Time Example:**
```csharp
// App Service configuration and autoscaling
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.DependencyInjection;

// Startup configuration for App Service
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        
        // Distributed caching (required for session state in scaled apps)
        services.AddStackExchangeRedisCache(options =>
        {
            options.Configuration = "myrediscache.redis.cache.windows.net:6380?ssl=True";
            options.InstanceName = "OrderService_";
        });

        // Shared state for multi-instance deployments
        services.AddSession(options =>
        {
            options.IdleTimeout = TimeSpan.FromMinutes(20);
            options.Cookie.IsEssential = true;
            options.Cookie.SameSite = SameSiteMode.Lax;
        });

        // Application Insights integration
        services.AddApplicationInsightsTelemetry();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        app.UseRouting();
        app.UseSession(); // Session before auth/endpoints
        app.UseEndpoints(endpoints => endpoints.MapControllers());

        // Health check endpoint for autoscale
        app.MapHealthChecks("/health");
    }
}

// Health check implementation for App Service
public class HealthService
{
    private readonly IHealthChecksBuilder _health;

    public void ConfigureHealthChecks(IServiceCollection services)
    {
        services.AddHealthChecks()
            .AddCheck("database", async () =>
            {
                try
                {
                    await Database.PingAsync();
                    return HealthCheckResult.Healthy("Database responding");
                }
                catch
                {
                    return HealthCheckResult.Unhealthy("Database unavailable");
                }
            })
            .AddCheck("cache", async () =>
            {
                try
                {
                    var cache = new RedisHealthCheck();
                    await cache.TestConnectionAsync();
                    return HealthCheckResult.Healthy("Cache responsive");
                }
                catch
                {
                    return HealthCheckResult.Degraded("Cache slow");
                }
            })
            .AddCheck("external-api", async () =>
            {
                var response = await new HttpClient().GetAsync("https://api.partner.com/health");
                return response.IsSuccessStatusCode
                    ? HealthCheckResult.Healthy()
                    : HealthCheckResult.Unhealthy("Partner API down");
            });
    }
}

// Load-sensitive controller (demonstrates scaling trigger)
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IMemoryCache _cache;
    private readonly TelemetryClient _telemetry;

    public OrdersController(IMemoryCache cache, TelemetryClient telemetry)
    {
        _cache = cache;
        _telemetry = telemetry;
    }

    [HttpPost("process")]
    public async Task<ActionResult> ProcessOrder([FromBody] Order order)
    {
        // Heavy computational load triggers autoscale
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            // Cache frequently accessed data
            var cachedProduct = _cache.GetOrCreate(
                $"product-{order.ProductId}",
                entry =>
                {
                    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
                    return Database.GetProduct(order.ProductId);
                });

            // Simulate CPU-intensive work
            var result = await ProcessOrderLogic(order, cachedProduct);
            
            // Track metrics for autoscale rules
            _telemetry.TrackEvent("OrderProcessed", metrics: new Dictionary<string, double>
            {
                { "Duration", stopwatch.ElapsedMilliseconds },
                { "CPU", GC.GetTotalMemory(false) / 1024.0 } // Memory usage
            });

            return Ok(result);
        }
        catch (Exception ex)
        {
            _telemetry.TrackException(ex);
            return StatusCode(503, "Service temporarily unavailable - scaling");
        }
    }

    [HttpGet("{id}")]
    public ActionResult<Order> GetOrder(int id)
    {
        // Might return 202 Accepted if queue is backed up
        var queueLength = BackgroundJobQueue.Count;
        if (queueLength > 100)
            return StatusCode(503, "Service under load, please retry");

        return Ok(Database.GetOrder(id));
    }
}

// ARM Template configuration for autoscaling (JSON)
/*
{
  "type": "Microsoft.Insights/autoscaleSettings",
  "name": "OrderServiceAutoscale",
  "properties": {
    "enabled": true,
    "targetResourceUri": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/serverfarms/OrderServicePlan",
    "profiles": [
      {
        "name": "Auto scale based on CPU",
        "capacity": {
          "minimum": "2",
          "maximum": "10",
          "default": "3"
        },
        "rules": [
          {
            "metricTrigger": {
              "metricName": "CpuPercentage",
              "metricNamespace": "Microsoft.Web/serverfarms",
              "timeGrain": "PT1M",
              "statistic": "Average",
              "timeWindow": "PT5M",
              "timeAggregation": "Average",
              "operator": "GreaterThan",
              "threshold": 70
            },
            "scaleAction": {
              "direction": "Increase",
              "type": "ChangeCount",
              "value": "1",
              "cooldown": "PT5M"
            }
          },
          {
            "metricTrigger": {
              "metricName": "CpuPercentage",
              "operator": "LessThan",
              "threshold": 30,
              "timeWindow": "PT10M",
              "statistic": "Average"
            },
            "scaleAction": {
              "direction": "Decrease",
              "type": "ChangeCount",
              "value": "1",
              "cooldown": "PT10M"
            }
          }
        ]
      },
      {
        "name": "Schedule-based scaling for peak hours",
        "capacity": {
          "minimum": "5",
          "maximum": "15",
          "default": "8"
        },
        "recurrence": {
          "frequency": "Week",
          "schedule": {
            "timeZone": "Eastern Standard Time",
            "days": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"],
            "hours": [8, 9, 10, 11, 12, 13, 14, 15, 16, 17],
            "minutes": 0
          }
        }
      }
    ]
  }
}
*/

// Deployment slots for zero-downtime deployments
public class DeploymentSlotStrategy
{
    /*
    Staging slot receives new deployment:
    1. Deploy to staging
    2. Run integration tests
    3. Warm up application
    4. Perform slot swap (atomic, <1 second)
    5. Monitor for issues
    6. Swap back if problems detected
    
    Benefits:
    - Production traffic uninterrupted
    - Easy rollback via swap
    - Test environment mirrors production
    - Session/cache retained via sticky slots
    */
}
```

**Explanation:**
- **App Service Plans**: Shared (F1), Standard (1+ instances), Premium (isolated VNets), Dedicated (isolated hardware)
- **Autoscaling**: Metric-based (CPU, memory, HTTP queue) or schedule-based (peak hours)
- **Deployment Slots**: Staging environment identical to production for pre-production validation
- **Session Management**: Use Redis for distributed sessions across instances
- **Health Checks**: Critical for autoscale decisions and load balancer decisions

**Interview Tips:**
- Discuss deployment slot swap atomicity and pre-swap validation
- Explain scaling cooldown periods to prevent "flapping"
- Address session affinity vs. stateless architecture
- Discuss scaling UP (larger instance) vs. OUT (more instances)
- Mention auto-heal for recovering from transient issues

---

## 7. How do Azure Storage and Azure Database interact in a typical application?

**Short Answer:**
Storage handles unstructured data (files, blobs); databases handle structured transactional data. A typical architecture stores document references in Database, blobs in Storage, and uses managed identities for secure cross-service access.

**Real-Time Example:**
```csharp
// Storage + Database integration pattern
using Azure.Storage.Blobs;
using Azure.Identity;
using Microsoft.EntityFrameworkCore;

// Entity combining database + storage
public class DocumentEntity
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string BlobUri { get; set; } // Reference to blob
    public long SizeMb { get; set; }
    public DateTime UploadedAt { get; set; }
    public string ContentType { get; set; }
}

public class DocumentDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlServer("Server=tcp:myserver.database.windows.net,1433;...Database=DocDb;...");
        // Enable encryption at rest via database service
        options.UseSqlServer(builder => builder.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelaySeconds: 10,
            errorNumbersToAdd: new[] { 64 })); // Transient error
    }

    public DbSet<DocumentEntity> Documents { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<DocumentEntity>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.BlobUri).IsRequired();
            entity.HasIndex(e => e.UploadedAt).IsDescending();
            
            // Soft delete pattern
            entity.HasQueryFilter(e => e.IsDeleted == false);
        });
    }
}

// Add IsDeleted tracking
public abstract class AuditEntity
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; }
    public DateTime? ModifiedAt { get; set; }
    public string ModifiedBy { get; set; }
    public bool IsDeleted { get; set; }
}

// Service coordinating Storage + Database
public class DocumentService
{
    private readonly BlobContainerClient _blobClient;
    private readonly DocumentDbContext _dbContext;

    public DocumentService(
        BlobContainerClient blobClient,
        DocumentDbContext dbContext)
    {
        _blobClient = blobClient;
        _dbContext = dbContext;
    }

    // Upload document: Write to Storage, record in Database
    public async Task<DocumentEntity> UploadDocument(
        Stream fileStream,
        string fileName,
        string userId)
    {
        var blobName = $"{userId}/{Guid.NewGuid()}-{fileName}";
        var blobClient = _blobClient.GetBlobClient(blobName);

        // Upload to blob storage
        var uploadOptions = new BlobUploadOptions
        {
            Metadata = new Dictionary<string, string>
            {
                { "uploadedBy", userId },
                { "originalName", fileName }
            }
        };

        var uploadedBlob = await blobClient.UploadAsync(fileStream, options: uploadOptions);
        var properties = await blobClient.GetPropertiesAsync();

        // Record metadata in database
        var document = new DocumentEntity
        {
            Title = fileName,
            BlobUri = blobClient.Uri.ToString(),
            SizeMb = properties.Value.ContentLength / (1024 * 1024),
            ContentType = properties.Value.ContentType,
            UploadedAt = DateTime.UtcNow,
            CreatedBy = userId
        };

        _dbContext.Documents.Add(document);
        await _dbContext.SaveChangesAsync();

        return document;
    }

    // Download: Query Database, retrieve from Storage
    public async Task<(Stream data, DocumentEntity metadata)> DownloadDocument(int docId, string userId)
    {
        // Get metadata from database
        var document = await _dbContext.Documents
            .FirstOrDefaultAsync(d => d.Id == docId && d.CreatedBy == userId);

        if (document == null)
            throw new UnauthorizedAccessException("Document not found or access denied");

        // Retrieve blob from storage
        var blobClient = new BlobClient(new Uri(document.BlobUri), new DefaultAzureCredential());
        var download = await blobClient.DownloadAsync();

        return (download.Value.Content, document);
    }

    // Query documents with paging
    public async Task<PagedResult<DocumentEntity>> ListDocuments(
        string userId,
        int pageNumber = 1,
        int pageSize = 20)
    {
        var query = _dbContext.Documents
            .Where(d => d.CreatedBy == userId)
            .OrderByDescending(d => d.UploadedAt);

        var totalCount = await query.CountAsync();
        var documents = await query
            .Skip((pageNumber - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();

        return new PagedResult<DocumentEntity>
        {
            Items = documents,
            TotalCount = totalCount,
            PageNumber = pageNumber,
            PageSize = pageSize
        };
    }

    // Hard delete with cascade (database → storage)
    public async Task DeleteDocument(int docId, string userId)
    {
        using var transaction = await _dbContext.Database.BeginTransactionAsync();
        try
        {
            var document = await _dbContext.Documents
                .FirstOrDefaultAsync(d => d.Id == docId && d.CreatedBy == userId);

            if (document == null)
                throw new KeyNotFoundException("Document not found");

            // Delete blob first (if blob missing, transaction rolls back)
            var blobClient = new BlobClient(new Uri(document.BlobUri), new DefaultAzureCredential());
            await blobClient.DeleteAsync();

            // Delete database record
            _dbContext.Documents.Remove(document);
            await _dbContext.SaveChangesAsync();
            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }

    // Backup strategy: Export database + blob snapshots
    public async Task<BackupStatus> CreateBackup(string backupName)
    {
        var status = new BackupStatus { BackupName = backupName, StartedAt = DateTime.UtcNow };

        try
        {
            // Database backup via SQL
            await _dbContext.Database.ExecuteSqlAsync(
                $@"BACKUP DATABASE [DocDb] TO URL = 'https://backup.blob.core.windows.net/backups/{backupName}.bak'");

            // Blob versioning enabled for point-in-time recovery
            var blobs = _blobClient.GetBlobsAsync();
            status.BlobCount = 0;
            await foreach (var blob in blobs)
            {
                var blobClient = _blobClient.GetBlobClient(blob.Name);
                var snapshot = await blobClient.CreateSnapshotAsync();
                status.BlobCount++;
            }

            status.Status = "Success";
            status.CompletedAt = DateTime.UtcNow;
        }
        catch (Exception ex)
        {
            status.Status = "Failed";
            status.ErrorMessage = ex.Message;
        }

        return status;
    }
}

// Configuration for managed identity across services
public static class AzureServicesSetup
{
    public static IServiceCollection AddAzureServices(
        this IServiceCollection services,
        IConfiguration config)
    {
        var credential = new DefaultAzureCredential();

        // Blob Storage
        services.AddSingleton(sp =>
            new BlobContainerClient(
                new Uri(config["Azure:StorageUri"]),
                credential));

        // Database
        services.AddDbContext<DocumentDbContext>(options =>
            options.UseSqlServer(config["Azure:SqlConnectionString"]));

        return services;
    }
}
```

**Explanation:**
- **Dual Storage**: Database stores searchable metadata; Blob Storage stores actual files
- **Referential Integrity**: Database ensures consistency; blobs are eventually consistent
- **Transaction Patterns**: Optimize for ACID in database, eventual consistency in storage
- **Access Control**: Use managed identities for authentication without credentials
- **Backup Strategy**: Coordinate database backups with blob snapshots for point-in-time recovery

**Interview Tips:**
- Discuss when to use Database vs. Blob Storage vs. Files
- Explain eventual consistency impact on cascade deletes
- Address orphaned blobs detection and cleanup strategies
- Mention transactional outbox pattern for reliability
- Discuss pagination patterns for large document sets

---

## 8. What are Azure Functions triggers and bindings, and how do they simplify development?

**Short Answer:**
Triggers activate functions (HTTP, Timer, Queue, Blob). Bindings provide declarative access to services without boilerplate, reducing code and improving readability through Microsoft.Azure.WebJobs attributes.

**Real-Time Example:**
```csharp
// Azure Functions triggers and bindings
using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using Azure.Storage.Blobs;
using Azure.Messaging.ServiceBus;

public static class DocumentProcessingFunctions
{
    // HTTP Trigger - REST API endpoint
    [FunctionName("CreateDocument")]
    public static async Task<IActionResult> CreateDocument(
        [HttpTrigger(
            AuthorizationLevel.Function,
            "post",
            Route = "documents")] HttpRequest req,
        [Queue("document-processing")] IAsyncCollector<DocumentMessage> docQueue,
        [Blob("documents/{id}")] CloudBlockBlob outputBlob,
        ILogger log)
    {
        string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
        var document = JsonConvert.DeserializeObject<Document>(requestBody);
        document.Id = Guid.NewGuid().ToString();

        // Send to queue for async processing
        await docQueue.AddAsync(new DocumentMessage { DocumentId = document.Id });
        
        // Store metadata in blob
        await outputBlob.UploadTextAsync(JsonConvert.SerializeObject(document));

        return new CreatedResult($"/documents/{document.Id}", document);
    }

    // Queue Trigger - Process messages sequentially
    [FunctionName("ProcessDocument")]
    public static async Task ProcessDocument(
        [QueueTrigger("document-processing")] DocumentMessage docMessage,
        [Blob("documents/{DocumentId}")] CloudBlockBlob documentBlob,
        [ServiceBus("document-completed", Connection = "ServiceBusConnection")] IAsyncCollector<DocumentCompletedEvent> completedQueue,
        ILogger log)
    {
        try
        {
            log.LogInformation($"Processing document: {docMessage.DocumentId}");

            // Read document from blob
            string content = await documentBlob.DownloadTextAsync();
            var document = JsonConvert.DeserializeObject<Document>(content);

            // Process document
            await ProcessDocumentLogic(document);

            // Send completion event
            await completedQueue.AddAsync(new DocumentCompletedEvent
            {
                DocumentId = document.Id,
                Status = "Completed",
                ProcessedAt = DateTime.UtcNow
            });

            log.LogInformation($"Document processed: {docMessage.DocumentId}");
        }
        catch (Exception ex)
        {
            log.LogError($"Error processing document: {ex.Message}");
            throw; // Retry via dead-letter queue
        }
    }

    // Timer Trigger - Scheduled execution (CRON expression)
    [FunctionName("CleanupExpiredDocuments")]
    public static async Task CleanupExpiredDocuments(
        [TimerTrigger("0 2 * * *")] TimerInfo timer, // 2 AM daily
        [Blob("documents")] BlobContainerClient containerClient,
        ILogger log)
    {
        log.LogInformation($"Cleanup started at {DateTime.UtcNow}");

        var expiredDate = DateTime.UtcNow.AddDays(-30);
        int deletedCount = 0;

        // Iterate blobs older than 30 days
        await foreach (BlobItem blob in containerClient.GetBlobsAsync())
        {
            var properties = await containerClient.GetBlobClient(blob.Name).GetPropertiesAsync();
            
            if (properties.Value.CreatedOn < expiredDate)
            {
                await containerClient.DeleteBlobAsync(blob.Name);
                deletedCount++;
            }
        }

        log.LogInformation($"Deleted {deletedCount} expired documents");
    }

    // Blob Trigger - Activated by blob creation/modification
    [FunctionName("AnalyzeUploadedDocument")]
    public static async Task AnalyzeDocument(
        [BlobTrigger("uploads/{name}")] Stream blobStream,
        string name,
        [Blob("results/{name}.json")] CloudBlockBlob resultBlob,
        [Table("DocumentAnalysis")] CloudTable analysisTable,
        ILogger log)
    {
        log.LogInformation($"Analyzing blob: {name}");

        // Analyze document
        using (var streamReader = new StreamReader(blobStream))
        {
            string content = await streamReader.ReadToEndAsync();
            var analysis = new DocumentAnalysis
            {
                FileName = name,
                WordCount = content.Split(' ').Length,
                AnalyzedAt = DateTime.UtcNow
            };

            // Store results in blob
            await resultBlob.UploadTextAsync(JsonConvert.SerializeObject(analysis));

            // Store metadata in table
            var tableEntity = new DocumentAnalysisEntity(analysis.FileName, analysis.AnalyzedAt.ToString())
            {
                WordCount = analysis.WordCount
            };
            await analysisTable.ExecuteAsync(TableOperation.InsertOrReplace(tableEntity));
        }
    }

    // Multiple Input Bindings - Dependency Injection
    [FunctionName("GenerateReport")]
    public static async Task GenerateReport(
        [HttpTrigger(AuthorizationLevel.Function, "get", Route = "reports/{reportId}")] HttpRequest req,
        string reportId,
        [Blob("reports/{reportId}.json")] CloudBlockBlob reportBlob,
        [Queue("email-queue")] IAsyncCollector<EmailMessage> emailQueue,
        [CosmosDB(
            databaseName: "ReportDb",
            collectionName: "Reports",
            ConnectionStringSetting = "CosmosDbConnection",
            Id = "{reportId}")] dynamic reportDoc,
        ILogger log)
    {
        // Query from Cosmos DB
        if (reportDoc == null)
            return new NotFoundResult();

        // Retrieve from blob
        string reportContent = await reportBlob.DownloadTextAsync();

        // Send email notification
        await emailQueue.AddAsync(new EmailMessage
        {
            To = reportDoc.RecipientEmail,
            Subject = $"Report {reportId} is ready",
            Body = reportContent
        });

        return new OkObjectResult(reportDoc);
    }

    // Durable Functions - Long-running orchestration
    [FunctionName("DocumentProcessingOrchestrator")]
    public static async Task DocumentOrchestrator(
        [OrchestrationTrigger] IDurableOrchestrationContext context,
        ILogger log)
    {
        var documentId = context.GetInput<string>();

        try
        {
            // Step 1: Validate
            await context.CallActivityAsync("ValidateDocument", documentId);

            // Step 2: Scan for viruses
            var scanResult = await context.CallActivityAsync<bool>("ScanForViruses", documentId);
            if (!scanResult)
                throw new InvalidOperationException("Virus detected");

            // Step 3: Extract metadata
            var metadata = await context.CallActivityAsync<DocumentMetadata>("ExtractMetadata", documentId);

            // Step 4: Index for search
            await context.CallActivityAsync("IndexDocument", metadata);

            // Step 5: Send notification
            await context.CallActivityAsync("NotifyProcessingComplete", documentId);
        }
        catch (Exception ex)
        {
            await context.CallActivityAsync("LogError", ex.Message);
            throw;
        }
    }

    [FunctionName("ValidateDocument")]
    public static async Task ValidateDocument([ActivityTrigger] string documentId, ILogger log)
    {
        log.LogInformation($"Validating document: {documentId}");
        // Validation logic
        await Task.Delay(100);
    }

    [FunctionName("ScanForViruses")]
    public static async Task<bool> ScanForViruses([ActivityTrigger] string documentId, ILogger log)
    {
        log.LogInformation($"Scanning document: {documentId}");
        // Call antivirus API
        return true; // No virus
    }
}

// Supporting models
public class DocumentMessage
{
    public string DocumentId { get; set; }
}

public class DocumentCompletedEvent
{
    public string DocumentId { get; set; }
    public string Status { get; set; }
    public DateTime ProcessedAt { get; set; }
}

public class DocumentAnalysis
{
    public string FileName { get; set; }
    public int WordCount { get; set; }
    public DateTime AnalyzedAt { get; set; }
}

public class DocumentAnalysisEntity : TableEntity
{
    public int WordCount { get; set; }

    public DocumentAnalysisEntity() { }
    public DocumentAnalysisEntity(string partitionKey, string rowKey) 
        : base(partitionKey, rowKey) { }
}
```

**Explanation:**
- **Triggers**: HTTP, Queue, Blob, Timer, Service Bus, Event Hub, Cosmos DB change feed
- **Bindings**: Declaratively connect to services; types: Input, Output, Input-Output
- **Durable Functions**: Orchestration with retries, timeouts, human approvals
- **Advantages**: Reduces boilerplate, built-in error handling, automatic scaling tuning

**Interview Tips:**
- Discuss cold starts and premium plan benefits
- Explain function timeout limits and workarounds
- Address monitoring via Application Insights integration
- Mention local development with Azure Functions Core Tools
- Discuss retry policies and dead-letter handling

---

## 9. How do you handle authentication and authorization across Azure services?

**Short Answer:**
Use Azure AD (Entra ID) for identity, managed identities for Azure service-to-service auth, RBAC for permissions, and token caching to minimize latency. Avoid storing secrets; use Key Vault instead.

**Real-Time Example:**
```csharp
// Authentication & Authorization across Azure services
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Authorization;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Identity.Web;

public class AuthenticationSetup
{
    // Configure Azure AD authentication for web app
    public static WebApplicationBuilder AddAzureAuthentication(
        this WebApplicationBuilder builder)
    {
        builder.Services.AddMicrosoftIdentityWebApiAuthentication(
            builder.Configuration,
            configSectionName: "AzureAd");

        builder.Services.AddAuthorization(options =>
        {
            // Role-based authorization
            options.AddPolicy("AdminsOnly", policy =>
                policy.RequireRole("Admin"));

            // Claims-based authorization
            options.AddPolicy("FinanceTeam", policy =>
                policy.RequireClaim("department", "Finance"));

            // Scope-based (OAuth)
            options.AddPolicy("WriteAccess", policy =>
                policy.RequireAssertion(ctx =>
                    ctx.User.FindFirst("scope")?.Value.Contains("api://app/write") ?? false));
        });

        return builder;
    }

    // Service-to-service authentication with managed identity
    public static IServiceCollection AddServiceAuthentication(
        this IServiceCollection services,
        IConfiguration config)
    {
        // Managed Identity - no secrets in code
        var credential = new DefaultAzureCredential(
            new DefaultAzureCredentialOptions
            {
                ManagedIdentityClientId = config["Azure:ManagedIdentityClientId"],
                ExcludeManagedIdentityCredential = false
            });

        services.AddSingleton(credential);

        // Cached token provider for performance
        services.AddSingleton<ITokenProvider>(sp =>
            new CachedTokenProvider(credential));

        return services;
    }
}

// HTTP client with token injection
public interface ITokenProvider
{
    Task<string> GetTokenAsync(string resource);
}

public class CachedTokenProvider : ITokenProvider
{
    private readonly DefaultAzureCredential _credential;
    private readonly MemoryCache _cache = new MemoryCache(new MemoryCacheOptions());

    public CachedTokenProvider(DefaultAzureCredential credential)
    {
        _credential = credential;
    }

    public async Task<string> GetTokenAsync(string resource)
    {
        var cacheKey = $"token-{resource}";
        
        if (_cache.TryGetValue(cacheKey, out string cachedToken))
            return cachedToken;

        // Get token from Azure AD (scoped to resource)
        var token = await _credential.GetTokenAsync(
            new Azure.Core.TokenRequestContext(new[] { $"{resource}/.default" }));

        // Cache for 80% of token lifetime
        var cacheTime = token.ExpiresOn.UtcDateTime.Subtract(DateTime.UtcNow) * 0.8;
        _cache.Set(cacheKey, token.Token, TimeSpan.FromSeconds(cacheTime.TotalSeconds));

        return token.Token;
    }
}

// Controller with role and scope-based authorization
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class OrdersController : ControllerBase
{
    private readonly ITokenProvider _tokenProvider;
    private readonly IHttpClientFactory _httpFactory;

    public OrdersController(ITokenProvider tokenProvider, IHttpClientFactory httpFactory)
    {
        _tokenProvider = tokenProvider;
        _httpFactory = httpFactory;
    }

    [HttpPost]
    [Authorize(Policy = "FinanceTeam")]
    [Authorize(Policy = "WriteAccess")]
    public async Task<ActionResult> CreateOrder([FromBody] Order order)
    {
        // User is authenticated, in FinanceTeam role, with write scope
        var userId = User.FindFirst("oid")?.Value;
        var userEmail = User.FindFirst("preferred_username")?.Value;

        order.CreatedBy = userId;
        await Database.SaveOrder(order);

        return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
    }

    [HttpGet("{id}")]
    [Authorize(Policy = "AdminsOnly")] // Admin access only
    public async Task<ActionResult> GetOrder(int id)
    {
        var order = await Database.GetOrder(id);
        return Ok(order);
    }

    // Service-to-service call with managed identity
    [HttpPost("process")]
    public async Task<ActionResult> ProcessOrder([FromBody] Order order)
    {
        // Get token for downstream service (e.g., Payment Service)
        var token = await _tokenProvider.GetTokenAsync("https://payment-service.azurewebsites.net");

        var client = _httpFactory.CreateClient();
        client.DefaultRequestHeaders.Authorization = 
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);

        var response = await client.PostAsJsonAsync(
            "https://payment-service.azurewebsites.net/api/payments",
            new { OrderId = order.Id, Amount = order.Total });

        return Ok(new { ProcessingId = order.Id });
    }
}

// Azure AD B2C for external users (consumer identity)
public class B2CAuthenticationSetup
{
    public static WebApplicationBuilder AddB2CAuthentication(
        this WebApplicationBuilder builder)
    {
        builder.Services
            .AddMicrosoftIdentityWebAppAuthentication(
                builder.Configuration,
                configSectionName: "AzureAdB2C")
            .EnableTokenAcquisitionToCallDownstreamApi(
                builder.Configuration.GetValue<string[]>("Graph:Scopes"))
            .AddMicrosoftGraph(builder.Configuration.GetSection("Graph"))
            .AddInMemoryTokenCaches();

        builder.Services.AddAuthorization(options =>
        {
            options.AddPolicy("MustBeInvolved", policy =>
            {
                // Custom claim from B2C sign-up attributes
                policy.RequireClaim("emailVerified", "true");
            });
        });

        return builder;
    }
}

// Confidential client application for daemon processes
public class DaemonServiceExample
{
    private readonly IConfidentialClientApplication _app;

    public DaemonServiceExample(IConfidentialClientApplication app)
    {
        _app = app;
    }

    public async Task RunAsync()
    {
        // Acquire token for service principal
        var scopes = new[] { "https://management.azure.com/.default" };
        
        try
        {
            var result = await _app.AcquireTokenForClient(scopes).ExecuteAsync();
            
            // Use token to call Azure Management API
            using var client = new HttpClient();
            client.DefaultRequestHeaders.Authorization =
                new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", result.AccessToken);

            var response = await client.GetAsync(
                "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/resources?api-version=2021-04-01");
        }
        catch (MsalServiceException ex)
        {
            // Handle token acquisition failure
            Console.WriteLine($"Authentication failed: {ex.Message}");
        }
    }

    // Confidential client configuration
    public static IServiceCollection AddDaemonServices(
        this IServiceCollection services,
        IConfiguration config)
    {
        var app = ConfidentialClientApplicationBuilder
            .Create(config["AzureAd:ClientId"])
            .WithClientSecret(config["AzureAd:ClientSecret"])
            .WithAuthority(new Uri($"https://login.microsoftonline.com/{config["AzureAd:TenantId"]}"))
            .Build();

        services.AddSingleton(app);
        return services;
    }
}

// Configuration JSON structure
/*
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "Domain": "yourdomain.com",
    "TenantId": "{{TENANT_ID}}",
    "ClientId": "{{CLIENT_ID}}",
    "ClientSecret": "[Store in Key Vault once deployed]",
    "CallbackPath": "/signin-oidc",
    "Scopes": ["api://{{CLIENT_ID}}/ReadWrite"]
  },
  "AzureAdB2C": {
    "Instance": "https://{{TENANT}}.b2clogin.com/",
    "ClientId": "{{B2C_CLIENT_ID}}",
    "CallbackPath": "/signin-oidc",
    "SignUpSignInPolicyId": "B2C_1_susi",
    "ResetPasswordPolicyId": "B2C_1_reset"
  }
}
*/
```

**Explanation:**
- **Managed Identity**: Azure automatically manages credentials for Azure resources
- **RBAC**: Role-based access control at subscription/resource group level
- **Azure AD**: Identity platform for enterprise authentication
- **Azure AD B2C**: Consumer identity for external applications
- **Token Caching**: Reduces latency and authentication service load
- **Service Principal**: Daemon applications with client secret or certificate

**Interview Tips:**
- Discuss managed identity benefits over connection strings
- Explain token expiration and refresh strategies
- Address RBAC roles (Reader, Contributor, Custom)
- Mention certificate-based authentication for enhanced security
- Discuss MFA for sensitive operations

---

## 10. What are Azure's disaster recovery and backup strategies?

**Short Answer:**
Azure provides geo-redundancy (GRS, GZRS), backup services (Azure Backup), and site recovery (ASR) for RPO/RTO objectives. Choose based on workload criticality: RPO (data loss tolerance) and RTO (downtime tolerance).

**Real-Time Example:**
```csharp
// Disaster Recovery and Backup strategies
using Microsoft.Azure.Management.RecoveryServices;
using Azure.Storage.Blobs;
using Microsoft.EntityFrameworkCore;

// Database backup and restore
public class DatabaseBackupService
{
    private readonly SqlManagementClient _sqlClient;
    private readonly string _resourceGroupName;
    private readonly string _serverName;
    private readonly string _databaseName;

    public DatabaseBackupService(
        SqlManagementClient sqlClient,
        string resourceGroup,
        string server,
        string database)
    {
        _sqlClient = sqlClient;
        _resourceGroupName = resourceGroup;
        _serverName = server;
        _databaseName = database;
    }

    // Automated backup (Azure SQL Server databases)
    public async Task<BackupInfo> GetLatestAutoBackup()
    {
        // Azure SQL Server provides automatic backups
        // Weekly full, daily differential, 5-minute transaction log
        var backups = await _sqlClient.Backups.ListByDatabaseAsync(
            _resourceGroupName,
            _serverName,
            _databaseName);

        var latest = backups.FirstOrDefault();
        return new BackupInfo
        {
            Id = latest.Id,
            CreatedTime = latest.CreatedTime,
            DeletedTime = latest.DeletedTime,
            BackupExpirationTime = latest.BackupExpirationTime
        };
    }

    // Point-in-time restore (PITR)
    public async Task<string> RestoreDatabaseToPointInTime(DateTime targetTime)
    {
        var restoredDbName = $"{_databaseName}-restore-{targetTime:yyyyMMddHHmmss}";

        var createParams = new Database
        {
            Location = "eastus",
            CreateMode = CreateMode.Restore,
            SourceDatabaseId = $"/subscriptions/[subId]/resourceGroups/{_resourceGroupName}/providers/Microsoft.Sql/servers/{_serverName}/databases/{_databaseName}",
            RestorePointInTime = targetTime
        };

        var result = await _sqlClient.Databases.CreateOrUpdateAsync(
            _resourceGroupName,
            _serverName,
            restoredDbName,
            createParams);

        return result.Id;
    }

    // Geo-restore to different region
    public async Task<string> GeoRestoreToDifferentRegion(string targetRegion)
    {
        var restoredDbName = $"{_databaseName}-geo-restore";

        var createParams = new Database
        {
            Location = targetRegion,
            CreateMode = CreateMode.Recovery,
            SourceDatabaseId = $"/subscriptions/[subId]/resourceGroups/{_resourceGroupName}/providers/Microsoft.Sql/servers/{_serverName}/databases/{_databaseName}"
        };

        var result = await _sqlClient.Databases.CreateOrUpdateAsync(
            _resourceGroupName,
            $"{_serverName}-{targetRegion}",
            restoredDbName,
            createParams);

        return result.Id;
    }

    // Active Geo-Replication for read-only replicas
    public async Task<ReplicationLink> SetupActiveGeoReplication(
        string targetRegion,
        string targetServerName)
    {
        var replicationLink = new ReplicationLink
        {
            PartnerLocation = targetRegion,
            PartnerDatabaseId = $"/subscriptions/[subId]/resourceGroups/{_resourceGroupName}/providers/Microsoft.Sql/servers/{targetServerName}/databases/{_databaseName}"
        };

        var link = await _sqlClient.ReplicationLinks.CreateAsync(
            _resourceGroupName,
            _serverName,
            _databaseName,
            replicationLink.PartnerDatabaseId);

        return link;
    }
}

// Blob Storage backup with versioning and snapshots
public class BlobBackupService
{
    private readonly BlobContainerClient _containerClient;

    public BlobBackupService(BlobContainerClient containerClient)
    {
        _containerClient = containerClient;
    }

    // Enable blob versioning (point-in-time recovery)
    public async Task EnableVersioning()
    {
        // Configure on storage account level
        var properties = await _containerClient.GetPropertiesAsync();
        // Versioning must be enabled via Azure Portal or ARM template
    }

    // Create blob snapshot (backup)
    public async Task<Uri> CreateSnapshot(string blobName)
    {
        var blobClient = _containerClient.GetBlobClient(blobName);
        var snapshot = await blobClient.CreateSnapshotAsync();
        return snapshot.Value.Snapshot; // Returns snapshot URI
    }

    // Restore blob from snapshot
    public async Task RestoreBlobFromSnapshot(string blobName, string snapshotDateTime)
    {
        var sourceUri = new Uri($"{_containerClient.Uri}/{blobName}?snapshot={snapshotDateTime}");
        var targetBlobClient = _containerClient.GetBlobClient($"{blobName}-restored");

        // Copy from snapshot to new blob
        var copy = await targetBlobClient.StartCopyFromUriAsync(sourceUri);
        while (copy.Value.CopyStatus == CopyStatus.Pending)
        {
            await Task.Delay(100);
            copy = await targetBlobClient.GetPropertiesAsync();
        }
    }

    // Enable immutable storage (WORM - Write Once Read Many)
    public async Task SetImmutabilityPolicy(string blobName, int retentionDays)
    {
        var blobClient = _containerClient.GetBlobClient(blobName);
        
        var policy = new BlobImmutabilityPolicy
        {
            ExpiresOn = DateTime.UtcNow.AddDays(retentionDays)
        };

        await blobClient.SetImmutabilityPolicyAsync(policy);
        // Blob cannot be deleted/modified until retention expires
    }
}

// Backup service for complete solution
public class AzureBackupService
{
    private readonly RecoveryServicesClient _recoveryClient;

    public AzureBackupService(RecoveryServicesClient recoveryClient)
    {
        _recoveryClient = recoveryClient;
    }

    // Backup VM
    public async Task BackupVirtualMachine(
        string resourceGroupName,
        string vaultName,
        string vmId,
        string policyName)
    {
        // Enable backup on VM using backup policy
        var protectionPolicy = new AzureIaaSVMProtectionPolicy
        {
            PolicyName = policyName,
            Schedule = new SchedulePolicy
            {
                ScheduleRuns = 1,
                ScheduleRunDaysOfWeek = new List<DayOfWeek> { DayOfWeek.Monday },
                ScheduleRunTimes = new List<DateTime> { DateTime.Parse("02:00") }
            },
            Retention = new LongTermRetentionPolicy
            {
                DailySchedule = new DailyRetentionSchedule { RetentionTimes = new List<DateTime> { DateTime.Parse("02:00") }, RetentionDuration = 30 },
                WeeklySchedule = new WeeklyRetentionSchedule { RetentionDuration = 104 }, // 2 years
                YearlySchedule = new YearlyRetentionSchedule { RetentionDuration = 1826 } // 5 years
            }
        };

        // Assign policy to VM
        var protectionIntent = new AzureResourceProtectionIntent
        {
            BackupManagementType = BackupManagementType.AzureIaaSVM,
            ItemId = vmId,
            PolicyId = await GetPolicyId(vaultName, policyName)
        };

        // Register VM for backup
        await _recoveryClient.RegisterIdentityResourceAsync(
            resourceGroupName,
            vaultName,
            protectionIntent);
    }

    private async Task<string> GetPolicyId(string vaultName, string policyName)
    {
        // Retrieve policy ID - placeholder
        return $"/subscriptions/[subId]/resourceGroups/[rg]/providers/Microsoft.RecoveryServices/vaults/{vaultName}/backupPolicies/{policyName}";
    }
}

// RTO/RPO Configuration
public class DisasterRecoveryConfiguration
{
    /*
    RPO (Recovery Point Objective) - How much data can you afford to lose?
    RTO (Recovery Time Objective) - How long can services be down?
    
    Service Tier vs. RTO/RPO:
    
    1. Backup (Daily snapshots)
       - RPO: 1 day
       - RTO: 1+ hours (manual restore)
       - Cost: Low
       - Use: Non-critical workloads
    
    2. Geo-Redundant Storage (GRS)
       - RPO: <1 hour (secondary sync)
       - RTO: 1+ hours (failover)
       - Cost: Moderate
       - Use: Moderate availability needs
    
    3. Active Geo-Replication + Failover Groups
       - RPO: <5 minutes (async replication)
       - RTO: <1 minute (automatic failover)
       - Cost: High
       - Use: Mission-critical applications
    
    4. Zone-Redundant Storage (ZRS)
       - RPO: Synchronous (zero data loss)
       - RTO: Automatic (multi-zone failover)
       - Cost: Moderate
       - Use: High-availability within region
    */
}

// Failover mechanism
public class FailoverService
{
    public async Task PerformManualFailover(
        string resourceGroupName,
        string serverName,
        string secondaryServerName)
    {
        // Failover groups provide automatic failover
        var failoverGroup = new FailoverGroup
        {
            PartnerServers = new List<PartnerInfo>
            {
                new PartnerInfo { Id = $"/subscriptions/[subId]/resourceGroups/{resourceGroupName}/providers/Microsoft.Sql/servers/{secondaryServerName}", Role = FailoverGroupPartnerRole.Secondary }
            },
            ReadWriteEndpoint = new FailoverGroupReadWriteEndpoint { FailoverPolicy = ReadWriteEndpointFailoverPolicy.Automatic, FailoverWithDataLossGracePeriodMinutes = 60 },
            ReadOnlyEndpoint = new FailoverGroupReadOnlyEndpoint { FailoverPolicy = ReadOnlyEndpointFailoverPolicy.Enabled }
        };

        // Create failover group for automatic failover
        // Primary and secondary automatically sync
        // On primary failure, application uses read-write endpoint that points to secondary
    }
}

// Monitoring backup health
public class BackupHealthMonitoring
{
    private readonly TelemetryClient _telemetry;

    public BackupHealthMonitoring(TelemetryClient telemetry)
    {
        _telemetry = telemetry;
    }

    public void MonitorBackupStatus(BackupInfo backup)
    {
        _telemetry.TrackEvent("BackupCompleted", new Dictionary<string, string>
        {
            { "BackupId", backup.Id },
            { "BackupTime", backup.CreatedTime.ToString() },
            { "ExpirationTime", backup.BackupExpirationTime.ToString() }
        });

        // Alert if backup is missing or expired
        if (backup.BackupExpirationTime < DateTime.UtcNow.AddDays(7))
        {
            _telemetry.TrackTrace(
                "Backup expiring soon",
                Microsoft.ApplicationInsights.DataContracts.SeverityLevel.Warning,
                new Dictionary<string, string> { { "BackupId", backup.Id } });
        }
    }
}
```

**Explanation:**
- **RPO/RTO Trade-offs**: Lower RPO/RTO = higher costs
- **Geo-Redundancy**: Automatic replication to secondary region, manual failover
- **Active Geo-Replication**: Readable secondary replicas for reporting workloads
- **Immutable Storage**: Prevents data deletion (compliance requirement)
- **Backup Retention**: Daily, weekly, monthly, yearly schedules

**Interview Tips:**
- Calculate RPO/RTO for different service tiers
- Discuss failover group automatic failover benefits
- Explain point-in-time recovery window limits
- Address backup testing and restore drills
- Discuss cost implications of geo-redundancy

---

## 11. Rapid-Fire Follow-Up Questions

1. What's the difference between Azure Storage Account tiers and access tiers?
2. How would you handle secret rotation in Key Vault without downtime?
3. Explain the difference between Service Bus Topics and Event Hubs.
4. What are deployment slots and why use them?
5. How does Azure Functions cold start impact production?
6. What's the difference between managed identity and service principal?
7. How would you implement circuit breaker pattern across Azure services?
8. What are the constraints when using Azure Functions on a Consumption plan?
9. How do you secure blob storage against unauthorized access?
10. Explain Durable Functions deadletter pattern.
11. What's the maximum message size for Service Bus?
12. How would you monitor Azure Functions performance?
13. What's the difference between Azure Table Storage and Cosmos DB?
14. How do you implement multi-region deployments on App Service?
15. Explain soft delete and how it helps disaster recovery.

---

## 12. Practice Plan

**For Interview Preparation:**
- Create a three-tier application: API (App Service) → Database → Storage
- Implement managed identity authentication between services
- Set up Application Insights monitoring with custom events
- Deploy using deployment slots with automated testing
- Configure auto-scaling based on CPU and queue depth
- Implement backup strategy with PITR and geo-redundancy
- Create failover mechanism using failover groups

**For Senior-Level Assessment:**
- Design a multi-region disaster recovery plan
- Implement saga pattern for distributed transactions using Service Bus
- Optimize costs by choosing appropriate storage tiers and redundancy
- Implement comprehensive monitoring and alerting
- Design for compliance (encryption, audit logging, retention)
- Plan zero-downtime deployments and rollback strategies

---

## 13. Key Takeaways

- **Choose services for workload type**: Functions for event-driven, App Service for continuous, Storage for unstructured data
- **Security first**: Use managed identities, Key Vault, and RBAC exclusively
- **Cost optimization**: Right-size resources, use reserved instances, implement tiering strategies
- **Disaster recovery**: Define RPO/RTO upfront; match Azure services to requirements
- **Monitoring**: Instrument every service; correlate logs across services
- **Automation**: Infrastructure as Code, CI/CD, auto-scaling, health checks
