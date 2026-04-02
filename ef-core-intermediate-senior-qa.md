# Entity Framework Core Interview Q and A (Intermediate to Senior, 30 Questions)

This guide is designed for intermediate to senior engineers using Entity Framework Core.
Each question includes:
- Short answer
- Real-time example
- Explanation
- C# snippet with comments

---

## 1) What is Entity Framework Core and why use it?

Short answer:
Entity Framework Core is an ORM for .NET that maps C# classes to database tables.

Real-time example:
An e-commerce Order service uses EF Core to persist orders without writing repetitive SQL for every CRUD operation.

Explanation:
EF Core improves developer productivity, supports LINQ, migrations, and provider flexibility.

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
}
```

---

## 2) What are DbContext and DbSet?

Short answer:
DbContext is the unit-of-work and change-tracking boundary; DbSet represents a table-like collection for an entity.

Real-time example:
In a billing API, one request uses a single DbContext instance to create invoice and invoice items atomically.

```csharp
public class BillingDbContext : DbContext
{
    public DbSet<Invoice> Invoices => Set<Invoice>();
    public DbSet<InvoiceItem> InvoiceItems => Set<InvoiceItem>();
}

public class Invoice
{
    public int Id { get; set; }
    public DateTime CreatedUtc { get; set; }
}

public class InvoiceItem
{
    public int Id { get; set; }
    public int InvoiceId { get; set; }
    public string Description { get; set; } = string.Empty;
}
```

---

## 3) What is code-first approach in EF Core?

Short answer:
Code-first means you model entities in C# first, then generate database schema via migrations.

Real-time example:
A startup quickly iterates domain model in code and evolves schema through CI pipeline migrations.

```csharp
// Entity model first
public class Customer
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
}

// Then generate migration via CLI:
// dotnet ef migrations add InitialCreate
// dotnet ef database update
```

---

## 4) What are migrations and why are they important?

Short answer:
Migrations track schema changes incrementally and safely across environments.

Real-time example:
Adding a new Orders.Status column must be deployed consistently to dev, QA, and prod.

```csharp
// Example migration commands:
// dotnet ef migrations add AddOrderStatus
// dotnet ef database update

// In production, many teams run generated SQL scripts:
// dotnet ef migrations script
```

---

## 5) How do you configure relationships in EF Core?

Short answer:
Use navigation properties and Fluent API or conventions to model one-to-many, one-to-one, and many-to-many.

Real-time example:
Order has many OrderItems; each OrderItem belongs to one Order.

```csharp
using Microsoft.EntityFrameworkCore;

public class Order
{
    public int Id { get; set; }
    public List<OrderItem> Items { get; set; } = new();
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public Order? Order { get; set; }
}

public class OrdersDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>()
            .HasMany(o => o.Items)
            .WithOne(i => i.Order)
            .HasForeignKey(i => i.OrderId);
    }
}
```

---

## 6) What is eager loading vs lazy loading vs explicit loading?

Short answer:
- Eager loading: load related data with Include.
- Lazy loading: load on first access.
- Explicit loading: load related data manually.

Real-time example:
Order details page needs order and items immediately, so eager loading is best.

```csharp
// Eager loading
var orders = await db.Orders
    .Include(o => o.Items)
    .ToListAsync(ct);

// Explicit loading
var order = await db.Orders.FirstAsync(o => o.Id == id, ct);
await db.Entry(order)
    .Collection(o => o.Items)
    .LoadAsync(ct);

// Lazy loading requires proxies package and virtual navigation properties.
```

---

## 7) What is tracking vs no-tracking query?

Short answer:
Tracking queries track entity changes for updates. No-tracking queries are faster for read-only scenarios.

Real-time example:
Public product catalog endpoint should use no-tracking for performance.

```csharp
// Read-only endpoint optimization
var products = await db.Products
    .AsNoTracking() // no change tracker overhead
    .Where(p => p.Price > 100)
    .ToListAsync(ct);
```

---

## 8) What is change tracking in EF Core?

Short answer:
EF Core detects changes to tracked entities and generates SQL for inserts, updates, and deletes.

Real-time example:
Admin updates product price in UI; EF tracks changed property and sends UPDATE on SaveChanges.

```csharp
var product = await db.Products.FirstAsync(p => p.Id == id, ct);
product.Price = 199.99m; // tracked change

await db.SaveChangesAsync(ct); // EF sends UPDATE statement
```

---

## 9) How do transactions work in EF Core?

Short answer:
SaveChanges uses transaction by default for multiple operations; you can create explicit transactions when needed.

Real-time example:
Transfer wallet balance between accounts must be atomic.

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);

try
{
    from.Balance -= amount;
    to.Balance += amount;

    await db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
}
catch
{
    await tx.RollbackAsync(ct);
    throw;
}
```

---

## 10) How do you handle optimistic concurrency?

Short answer:
Use concurrency token (for example RowVersion) and handle DbUpdateConcurrencyException.

Real-time example:
Two admins edit same product simultaneously; one update should fail gracefully.

```csharp
using System.ComponentModel.DataAnnotations;

public class InventoryItem
{
    public int Id { get; set; }
    public int Quantity { get; set; }

    [Timestamp] // concurrency token
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}

try
{
    await db.SaveChangesAsync(ct);
}
catch (DbUpdateConcurrencyException)
{
    // Return 409 conflict or re-load and merge
}
```

---

## 11) How do you avoid N plus 1 query problem?

Short answer:
Project required data in one query and avoid looping with per-item lazy loads.

Real-time example:
Dashboard listing 100 orders should not trigger 100 additional customer queries.

```csharp
var data = await db.Orders
    .Select(o => new
    {
        o.Id,
        CustomerName = o.Customer.Name,
        ItemCount = o.Items.Count
    })
    .ToListAsync(ct);
```

---

## 12) Why is projection important in EF Core queries?

Short answer:
Projection fetches only needed columns and reduces memory/network overhead.

Real-time example:
Product listing page needs only Id, Name, Price, not full entity graph.

```csharp
var dto = await db.Products
    .AsNoTracking()
    .Select(p => new ProductListDto
    {
        Id = p.Id,
        Name = p.Name,
        Price = p.Price
    })
    .ToListAsync(ct);

public class ProductListDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
}
```

---

## 13) What is compiled query in EF Core?

Short answer:
Compiled queries cache query translation for hot paths to reduce overhead.

Real-time example:
High-traffic endpoint repeatedly fetches product by SKU.

```csharp
using Microsoft.EntityFrameworkCore;

public static class Queries
{
    // Compiled query for repeated execution
    public static readonly Func<AppDbContext, string, Task<Product?>> ProductBySku =
        EF.CompileAsyncQuery((AppDbContext db, string sku) =>
            db.Products.FirstOrDefault(p => p.Name == sku));
}

var product = await Queries.ProductBySku(db, "SKU-123");
```

---

## 14) What is split query and when do you use it?

Short answer:
Split queries break large Include graphs into multiple SQL queries to avoid cartesian explosion.

Real-time example:
Order with multiple related collections can create huge duplicated rows in single joined query.

```csharp
var orders = await db.Orders
    .Include(o => o.Items)
    .Include(o => o.Payments)
    .AsSplitQuery() // prevent very large join explosions
    .ToListAsync(ct);
```

---

## 15) How do global query filters work?

Short answer:
Global filters apply automatically to all queries for an entity (for example soft delete, tenant isolation).

Real-time example:
Multi-tenant SaaS must return only current tenant data.

```csharp
public class TenantDbContext : DbContext
{
    private readonly int _tenantId;

    public TenantDbContext(DbContextOptions<TenantDbContext> options, int tenantId) : base(options)
    {
        _tenantId = tenantId;
    }

    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>()
            .HasQueryFilter(o => o.TenantId == _tenantId && !o.IsDeleted);
    }
}

public class Order
{
    public int Id { get; set; }
    public int TenantId { get; set; }
    public bool IsDeleted { get; set; }
}
```

---

## 16) How do you implement soft delete in EF Core?

Short answer:
Mark records deleted with flag and filter them out rather than physically deleting.

Real-time example:
Compliance requires keeping user records for audit.

```csharp
public interface ISoftDelete
{
    bool IsDeleted { get; set; }
}

public class Customer : ISoftDelete
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public bool IsDeleted { get; set; }
}

// Soft delete operation
var customer = await db.Set<Customer>().FirstAsync(c => c.Id == id, ct);
customer.IsDeleted = true;
await db.SaveChangesAsync(ct);

// Combine with global query filter to hide deleted rows.
```

---

## 17) How do you execute raw SQL safely in EF Core?

Short answer:
Use parameterized methods like FromSqlInterpolated and ExecuteSqlInterpolated to avoid SQL injection.

Real-time example:
Complex report stored procedure not easily modeled in LINQ.

```csharp
int minPrice = 100;

var products = await db.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Price > {minPrice}")
    .ToListAsync(ct);

// Non-query SQL
await db.Database.ExecuteSqlInterpolatedAsync($"UPDATE Products SET Price = Price * 1.05 WHERE Price < {500}", ct);
```

---

## 18) How do you configure indexes in EF Core?

Short answer:
Use Fluent API to define indexes for query performance and uniqueness.

Real-time example:
User login lookup by Email should be fast and unique.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<User>()
        .HasIndex(u => u.Email)
        .IsUnique();
}

public class User
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
}
```

---

## 19) How do you seed data in EF Core?

Short answer:
Seed reference data via model configuration or custom startup logic.

Real-time example:
Countries and supported currencies are required at application startup.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Currency>().HasData(
        new Currency { Id = 1, Code = "USD" },
        new Currency { Id = 2, Code = "EUR" }
    );
}

public class Currency
{
    public int Id { get; set; }
    public string Code { get; set; } = string.Empty;
}
```

---

## 20) How do owned entity types work?

Short answer:
Owned types model value objects that exist only as part of owner entity.

Real-time example:
Customer address should not be an independent table entity in simple domain.

```csharp
public class CustomerProfile
{
    public int Id { get; set; }
    public Address Address { get; set; } = new();
}

[Owned]
public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<CustomerProfile>().OwnsOne(c => c.Address);
}
```

---

## 21) What is table-per-hierarchy inheritance in EF Core?

Short answer:
TPH stores entire inheritance hierarchy in one table with discriminator column.

Real-time example:
Payment table stores CardPayment and UpiPayment rows with type discriminator.

```csharp
public abstract class Payment
{
    public int Id { get; set; }
    public decimal Amount { get; set; }
}

public class CardPayment : Payment
{
    public string Last4 { get; set; } = string.Empty;
}

public class UpiPayment : Payment
{
    public string UpiId { get; set; } = string.Empty;
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Payment>()
        .HasDiscriminator<string>("PaymentType")
        .HasValue<CardPayment>("CARD")
        .HasValue<UpiPayment>("UPI");
}
```

---

## 22) How do you tune SaveChanges performance for bulk operations?

Short answer:
Use batching, disable unnecessary tracking, and consider bulk libraries for very large writes.

Real-time example:
Nightly import inserts 500k rows.

```csharp
db.ChangeTracker.AutoDetectChangesEnabled = false;

for (int i = 0; i < 10000; i++)
{
    db.Products.Add(new Product { Name = $"P-{i}", Price = i });

    if (i % 1000 == 0)
    {
        await db.SaveChangesAsync(ct); // flush in batches
        db.ChangeTracker.Clear();      // clear tracked entities
    }
}

await db.SaveChangesAsync(ct);
db.ChangeTracker.AutoDetectChangesEnabled = true;
```

---

## 23) What interceptors are available in EF Core?

Short answer:
Interceptors allow cross-cutting logic around commands, connections, and SaveChanges.

Real-time example:
Automatically populate audit fields CreatedUtc and UpdatedUtc.

```csharp
using Microsoft.EntityFrameworkCore.Diagnostics;

public class AuditSaveChangesInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var context = eventData.Context;
        if (context is null) return base.SavingChangesAsync(eventData, result, cancellationToken);

        foreach (var entry in context.ChangeTracker.Entries<IAuditable>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedUtc = DateTime.UtcNow;
            }
            entry.Entity.UpdatedUtc = DateTime.UtcNow;
        }

        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}

public interface IAuditable
{
    DateTime CreatedUtc { get; set; }
    DateTime UpdatedUtc { get; set; }
}
```

---

## 24) How do you log generated SQL in EF Core?

Short answer:
Enable EF Core logging and use ToQueryString for debugging query translation.

Real-time example:
Slow endpoint investigation needs actual SQL emitted by LINQ.

```csharp
var query = db.Products
    .Where(p => p.Price > 500)
    .OrderBy(p => p.Name);

string sql = query.ToQueryString();
Console.WriteLine(sql); // inspect generated SQL

var result = await query.ToListAsync(ct);
```

---

## 25) How do you use EF Core with dependency injection in ASP.NET Core?

Short answer:
Register DbContext as scoped and inject into services/controllers.

Real-time example:
One web request should use one DbContext instance.

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

public class ProductService
{
    private readonly AppDbContext _db;

    public ProductService(AppDbContext db)
    {
        _db = db;
    }

    public Task<List<Product>> GetAllAsync(CancellationToken ct) =>
        _db.Products.AsNoTracking().ToListAsync(ct);
}
```

---

## 26) What is DbContext pooling?

Short answer:
DbContext pooling reuses context instances to reduce allocation overhead in high-throughput apps.

Real-time example:
High-traffic read APIs benefit from lower allocation and improved throughput.

```csharp
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Use carefully: do not store DbContext beyond request scope.
```

---

## 27) How do you implement multi-tenancy with EF Core?

Short answer:
Use tenant-aware context with global query filters and tenant ID stamping.

Real-time example:
SaaS HR app must isolate each company's employee data.

```csharp
public interface ITenantProvider
{
    int TenantId { get; }
}

public class MultiTenantDbContext : DbContext
{
    private readonly ITenantProvider _tenant;

    public MultiTenantDbContext(DbContextOptions<MultiTenantDbContext> options, ITenantProvider tenant)
        : base(options)
    {
        _tenant = tenant;
    }

    public DbSet<Employee> Employees => Set<Employee>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Employee>().HasQueryFilter(e => e.TenantId == _tenant.TenantId);
    }
}

public class Employee
{
    public int Id { get; set; }
    public int TenantId { get; set; }
    public string Name { get; set; } = string.Empty;
}
```

---

## 28) How do you test EF Core code effectively?

Short answer:
Use integration tests with real provider (or sqlite in-memory) for realistic query behavior.

Real-time example:
A LINQ query passes with in-memory provider but fails in SQL translation; integration tests catch this.

```csharp
using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;

var connection = new SqliteConnection("DataSource=:memory:");
await connection.OpenAsync();

var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseSqlite(connection)
    .Options;

await using var db = new AppDbContext(options);
await db.Database.EnsureCreatedAsync();

db.Products.Add(new Product { Name = "Test", Price = 10 });
await db.SaveChangesAsync();

var count = await db.Products.CountAsync();
Console.WriteLine(count); // 1
```

---

## 29) When should you use EF Core and when not?

Short answer:
Use EF Core for most CRUD and domain-driven applications. Consider Dapper or raw SQL for very specific high-performance or SQL-heavy paths.

Real-time example:
Financial reporting query with complex window functions may be easier in raw SQL or specialized data pipeline.

```csharp
// Hybrid approach
// EF Core for normal transactional flows
var order = await db.Orders.FindAsync(new object?[] { id }, ct);

// Raw SQL/Dapper for specific heavy reporting endpoint
// Keep boundary clear and documented.
```

---

## 30) What are top EF Core production best practices?

Short answer:
Use no-tracking for reads, projection, proper indexes, bounded context DbContexts, migrations discipline, and observability.

Real-time example:
Checkout API latency dropped after switching to projections plus index on Order.CustomerId.

```csharp
// Production-oriented read query
var recent = await db.Orders
    .AsNoTracking()
    .Where(o => o.CreatedUtc >= DateTime.UtcNow.AddDays(-7))
    .OrderByDescending(o => o.CreatedUtc)
    .Select(o => new
    {
        o.Id,
        o.CreatedUtc,
        Total = o.Items.Sum(i => i.Quantity * i.UnitPrice)
    })
    .Take(50)
    .ToListAsync(ct);

// Combine with DB index and query monitoring.
```

---

## Rapid Fire Senior Follow-ups

1. How to handle migration conflicts across parallel feature branches?
2. How to monitor slow EF queries in production?
3. What is query plan cache and parameter sniffing impact?
4. When to use ExecuteUpdate and ExecuteDelete in EF Core?
5. How to implement audit trail with temporal tables?
6. Why can Include become expensive with many collections?
7. How to prevent accidental client-side evaluation?
8. How to model strongly-typed IDs in EF Core?
9. How to migrate large tables with near-zero downtime?
10. What is the tradeoff of single large DbContext vs multiple bounded DbContexts?

---

## Practice Plan

1. Build a small Order API with EF Core, migrations, and DTO projection.
2. Add no-tracking read endpoints and compare performance.
3. Implement RowVersion concurrency on one entity and handle conflict.
4. Add soft delete plus global query filter.
5. Write integration tests using sqlite in-memory and verify query behavior.
