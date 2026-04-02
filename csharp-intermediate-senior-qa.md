# C# Intermediate to Senior Interview Q&A (30 Questions)

This guide is designed for developers with 3 to 10+ years of C# experience.
Each question includes:
- Short answer
- Interview explanation
- Practical code snippet with comments

---

## 1) What is the difference between `IEnumerable<T>` and `IQueryable<T>`?

**Short answer:**
- `IEnumerable<T>` executes in memory (LINQ to Objects).
- `IQueryable<T>` builds an expression tree that can be translated by a provider (for example, SQL by Entity Framework).

**Explanation:**
Use `IQueryable<T>` for database-side filtering/projection so the database does the heavy lifting. If you materialize early (for example, with `ToList()`), the rest runs in memory.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

var inMemory = new List<int> { 1, 2, 3, 4, 5 };

// IEnumerable: query executes in memory
IEnumerable<int> eQuery = inMemory.Where(x => x > 2);
Console.WriteLine(string.Join(",", eQuery)); // 3,4,5

// IQueryable example (typically EF DbSet<T>)
IQueryable<int> qQuery = inMemory.AsQueryable().Where(x => x > 2);
// Provider can translate expression tree (in EF, usually to SQL)
Console.WriteLine(qQuery.Expression);
```

---

## 2) What are value types and reference types in C#?

**Short answer:**
- Value types (`struct`, `int`, `double`, `bool`) store data directly.
- Reference types (`class`, `string`, arrays, delegates) store references to objects.

**Explanation:**
Copying a value type copies data. Copying a reference type copies the reference (both variables point to same object).

```csharp
public struct Point
{
    public int X;
    public int Y;
}

public class Person
{
    public string Name { get; set; } = string.Empty;
}

var p1 = new Point { X = 1, Y = 2 };
var p2 = p1;            // copy values
p2.X = 10;
Console.WriteLine(p1.X); // 1 (unchanged)

var person1 = new Person { Name = "Alice" };
var person2 = person1;   // copy reference
person2.Name = "Bob";
Console.WriteLine(person1.Name); // Bob (same object)
```

---

## 3) Explain `async`/`await` and common pitfalls.

**Short answer:**
`async`/`await` enables non-blocking asynchronous code. Use `Task`/`Task<T>` and avoid blocking calls like `.Result` and `.Wait()`.

**Explanation:**
`await` releases the thread while work is in progress. Common mistakes: fire-and-forget without handling, blocking async code, ignoring cancellation.

```csharp
using System;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

public static async Task<string> DownloadAsync(string url, CancellationToken ct)
{
    using var client = new HttpClient();

    // Non-blocking I/O call
    var content = await client.GetStringAsync(url, ct);
    return content;
}

// Pitfall to avoid:
// var text = DownloadAsync("https://example.com", CancellationToken.None).Result; // can deadlock in some contexts
```

---

## 4) What is dependency injection in C#/.NET and why use it?

**Short answer:**
Dependency Injection (DI) means supplying dependencies from outside a class, usually through constructor injection.

**Explanation:**
DI improves testability, loose coupling, and maintainability.

```csharp
public interface IEmailSender
{
    Task SendAsync(string to, string subject, string body);
}

public class EmailSender : IEmailSender
{
    public Task SendAsync(string to, string subject, string body)
    {
        // send email here
        return Task.CompletedTask;
    }
}

public class OrderService
{
    private readonly IEmailSender _emailSender;

    // Constructor injection
    public OrderService(IEmailSender emailSender)
    {
        _emailSender = emailSender;
    }

    public async Task PlaceOrderAsync(string email)
    {
        // business logic...
        await _emailSender.SendAsync(email, "Order Confirmed", "Thanks!");
    }
}

// In Program.cs (ASP.NET Core):
// services.AddScoped<IEmailSender, EmailSender>();
```

---

## 5) Explain `var`, `dynamic`, and explicit types.

**Short answer:**
- `var`: compile-time type inference.
- `dynamic`: runtime binding.
- Explicit type: directly specified at compile time.

**Explanation:**
Use `var` when type is obvious. Avoid overusing `dynamic` because runtime errors are more likely and tooling support is weaker.

```csharp
var count = 10; // inferred as int (compile-time)

dynamic value = "hello";
Console.WriteLine(value.Length); // resolved at runtime

value = 123;
// RuntimeBinderException if property doesn't exist at runtime
// Console.WriteLine(value.Length);

string name = "Alice"; // explicit type
```

---

## 6) What are delegates, events, and `Action`/`Func`?

**Short answer:**
Delegates are type-safe function pointers. Events expose publish-subscribe patterns. `Action` and `Func` are built-in delegate types.

**Explanation:**
Use events when a class notifies external listeners without tight coupling.

```csharp
public class FileProcessor
{
    // Event to notify listeners
    public event Action<string>? FileProcessed;

    public void Process(string fileName)
    {
        // process file...
        FileProcessed?.Invoke(fileName); // notify subscribers
    }
}

var processor = new FileProcessor();
processor.FileProcessed += file => Console.WriteLine($"Processed: {file}");
processor.Process("report.csv");

Func<int, int, int> add = (a, b) => a + b;
Action<string> print = s => Console.WriteLine(s);
```

---

## 7) What is LINQ deferred execution?

**Short answer:**
Deferred execution means LINQ query runs only when enumerated.

**Explanation:**
This can be useful, but can also surprise people when source changes before enumeration.

```csharp
var numbers = new List<int> { 1, 2, 3 };
var query = numbers.Where(n => n > 1); // not executed yet

numbers.Add(4); // source changed before enumeration

foreach (var n in query)
{
    Console.WriteLine(n); // 2, 3, 4
}

// To execute immediately, materialize:
var immediate = numbers.Where(n => n > 1).ToList();
```

---

## 8) Difference between `First`, `FirstOrDefault`, `Single`, `SingleOrDefault`?

**Short answer:**
- `First`: first item; throws if none.
- `FirstOrDefault`: first item or default.
- `Single`: exactly one; throws if none or many.
- `SingleOrDefault`: one or default; throws if many.

```csharp
var users = new[] { "alice", "bob" };

var first = users.First(); // alice
var maybe = users.FirstOrDefault(x => x == "charlie"); // null

var one = new[] { 42 }.Single(); // 42
// var fail = users.Single(); // throws: more than one element
```

---

## 9) What are extension methods?

**Short answer:**
Extension methods allow adding methods to existing types without modifying original type.

**Explanation:**
They are static methods in static classes with `this` on first parameter.

```csharp
public static class StringExtensions
{
    public static bool IsNullOrTrimEmpty(this string? value)
    {
        // Extension method on string
        return string.IsNullOrWhiteSpace(value);
    }
}

string? input = "   ";
Console.WriteLine(input.IsNullOrTrimEmpty()); // True
```

---

## 10) What is `IDisposable` and why `using` matters?

**Short answer:**
`IDisposable` releases unmanaged resources deterministically. `using` ensures cleanup even when exceptions happen.

```csharp
using System.IO;

public static string ReadFile(string path)
{
    // using statement guarantees disposal
    using var reader = new StreamReader(path);
    return reader.ReadToEnd();
}
```

---

## 11) Explain `record` vs `class` in modern C#.

**Short answer:**
`record` is for immutable, value-based equality models; `class` defaults to reference-based equality.

```csharp
public record UserRecord(int Id, string Name);
public class UserClass
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}

var r1 = new UserRecord(1, "A");
var r2 = new UserRecord(1, "A");
Console.WriteLine(r1 == r2); // True (value equality)

var c1 = new UserClass { Id = 1, Name = "A" };
var c2 = new UserClass { Id = 1, Name = "A" };
Console.WriteLine(c1 == c2); // False (different references)
```

---

## 12) What is pattern matching and why is it useful?

**Short answer:**
Pattern matching allows concise and safe branching based on shape/type/value.

```csharp
public static string Describe(object value) => value switch
{
    null => "null",
    int i when i > 0 => "positive int",
    int => "non-positive int",
    string { Length: > 5 } s => $"long string: {s}",
    string s => $"short string: {s}",
    _ => "unknown"
};
```

---

## 13) Explain covariance and contravariance in generics.

**Short answer:**
Covariance (`out`) allows using more derived return types. Contravariance (`in`) allows less derived parameter types.

```csharp
IEnumerable<string> names = new List<string> { "Alice" };
IEnumerable<object> objs = names; // covariance on IEnumerable<out T>

Action<object> handleObj = o => Console.WriteLine(o);
Action<string> handleString = handleObj; // contravariance on Action<in T>
handleString("hello");
```

---

## 14) What are nullable reference types?

**Short answer:**
Nullable reference types help catch null bugs at compile time using annotations and flow analysis.

```csharp
#nullable enable

public class Customer
{
    public string Name { get; set; } = string.Empty;
    public string? MiddleName { get; set; } // can be null
}

void PrintMiddle(Customer c)
{
    // Null-safe access
    Console.WriteLine(c.MiddleName?.ToUpper() ?? "N/A");
}
```

---

## 15) Difference between `Task`, `ValueTask`, and when to use each?

**Short answer:**
Use `Task` by default. Use `ValueTask` only in hot paths where synchronous completion is common and allocations matter.

```csharp
public Task<int> GetCountAsync()
{
    // Standard async contract
    return Task.FromResult(5);
}

public ValueTask<int> GetCachedCountAsync(bool fromCache)
{
    if (fromCache)
    {
        // No Task allocation when completed synchronously
        return new ValueTask<int>(5);
    }

    return new ValueTask<int>(Task.Run(() => 10));
}
```

---

## 16) How does exception handling best practice look in C#?

**Short answer:**
Catch only what you can handle, preserve stack trace, and add context.

```csharp
public static void ProcessPayment(string paymentId)
{
    try
    {
        // risky operation
        throw new InvalidOperationException("Gateway timeout");
    }
    catch (InvalidOperationException ex)
    {
        // Add meaningful context, keep original exception
        throw new ApplicationException($"Payment failed for {paymentId}", ex);
    }
}

// Avoid: catch (Exception) { }
// Avoid: throw ex; // resets stack trace
```

---

## 17) What is the difference between `throw` and `throw ex`?

**Short answer:**
`throw;` preserves original stack trace. `throw ex;` resets it.

```csharp
try
{
    DoWork();
}
catch (Exception ex)
{
    Log(ex);
    throw; // preferred to preserve stack trace
}

void DoWork() => throw new Exception("boom");
void Log(Exception ex) => Console.WriteLine(ex.Message);
```

---

## 18) What is `lock` and when should you use concurrent collections?

**Short answer:**
`lock` protects critical sections. Prefer thread-safe collections (`ConcurrentDictionary`, etc.) for shared mutable state patterns.

```csharp
using System.Collections.Concurrent;

var dict = new ConcurrentDictionary<string, int>();

// Thread-safe add/update
dict.AddOrUpdate("key", 1, (_, old) => old + 1);

// lock example for custom critical section
object gate = new();
int counter = 0;

void Increment()
{
    lock (gate)
    {
        // Only one thread enters this block at a time
        counter++;
    }
}
```

---

## 19) Explain `Span<T>` and `Memory<T>`.

**Short answer:**
`Span<T>` is a stack-only view over contiguous memory for high-performance slicing without allocations. `Memory<T>` is heap-friendly and can cross async boundaries.

```csharp
int[] data = { 1, 2, 3, 4, 5 };
Span<int> slice = data.AsSpan(1, 3); // 2,3,4

for (int i = 0; i < slice.Length; i++)
{
    // Modifies original array through span view
    slice[i] *= 10;
}

Console.WriteLine(string.Join(",", data)); // 1,20,30,40,5
```

---

## 20) What are `yield return` and iterator methods?

**Short answer:**
`yield return` enables lazy sequence generation without creating full collections upfront.

```csharp
public static IEnumerable<int> GetEvenNumbers(int max)
{
    for (int i = 0; i <= max; i++)
    {
        if (i % 2 == 0)
        {
            yield return i; // yields one value at a time
        }
    }
}

foreach (var n in GetEvenNumbers(10))
{
    Console.WriteLine(n);
}
```

---

## 21) What is the repository pattern and when to avoid overengineering?

**Short answer:**
Repository abstracts data access. Useful for domain boundaries and testability, but avoid unnecessary abstraction layers when ORM already provides repository-like behavior.

```csharp
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(int id);
    Task AddAsync(Product product);
}

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _db;

    public ProductRepository(AppDbContext db) => _db = db;

    public Task<Product?> GetByIdAsync(int id)
        => _db.Products.FindAsync(id).AsTask();

    public async Task AddAsync(Product product)
    {
        _db.Products.Add(product);
        await _db.SaveChangesAsync();
    }
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}

public class AppDbContext
{
    public FakeDbSet<Product> Products { get; } = new();
    public Task<int> SaveChangesAsync() => Task.FromResult(1);
}

public class FakeDbSet<T> where T : class, new()
{
    public ValueTask<T?> FindAsync(int id) => ValueTask.FromResult<T?>(new T());
    public void Add(T item) { }
}
```

---

## 22) Explain Entity Framework tracking vs no-tracking queries.

**Short answer:**
Tracking queries monitor entity changes for updates. `AsNoTracking()` is faster for read-only scenarios.

```csharp
// Tracking query (default): good when updating returned entity
// var user = await db.Users.FirstAsync(u => u.Id == id);
// user.Name = "Updated";
// await db.SaveChangesAsync();

// No-tracking: better for read-only APIs
// var users = await db.Users.AsNoTracking().ToListAsync();

Console.WriteLine("Use AsNoTracking for read-only to reduce overhead.");
```

---

## 23) How do you implement optimistic concurrency in EF Core?

**Short answer:**
Use a concurrency token (`rowversion`/`timestamp`) and handle `DbUpdateConcurrencyException`.

```csharp
public class Order
{
    public int Id { get; set; }
    public string Status { get; set; } = string.Empty;

    // Concurrency token (EF Core: [Timestamp])
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}

// In update flow (pseudo):
// try
// {
//     await db.SaveChangesAsync();
// }
// catch (DbUpdateConcurrencyException)
// {
//     // Handle conflict: reload, merge, retry, or return 409 Conflict
// }
```

---

## 24) What is middleware in ASP.NET Core?

**Short answer:**
Middleware are pipeline components handling request/response processing.

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Custom middleware in pipeline
app.Use(async (context, next) =>
{
    var started = DateTime.UtcNow;

    await next(); // pass control to next middleware

    var elapsed = DateTime.UtcNow - started;
    Console.WriteLine($"{context.Request.Path} took {elapsed.TotalMilliseconds}ms");
});

app.MapGet("/", () => "Hello");
app.Run();
```

---

## 25) How do you design resilient HTTP clients in .NET?

**Short answer:**
Use `IHttpClientFactory`, timeouts, retries (Polly), circuit breakers, and cancellation tokens.

```csharp
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

public class WeatherClient
{
    private readonly HttpClient _httpClient;

    public WeatherClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
        _httpClient.Timeout = TimeSpan.FromSeconds(5); // fail fast
    }

    public async Task<string> GetForecastAsync(CancellationToken ct)
    {
        // Pass cancellation token for cooperative cancellation
        using var response = await _httpClient.GetAsync("forecast", ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync(ct);
    }
}

// Registration example (Program.cs):
// services.AddHttpClient<WeatherClient>(client => client.BaseAddress = new Uri("https://api.weather.local/"));
```

---

## 26) What is the difference between abstraction and interface?

**Short answer:**
Interfaces define contracts only. Abstract classes can define contracts plus shared implementation/state.

```csharp
public interface INotifier
{
    Task NotifyAsync(string message);
}

public abstract class NotifierBase : INotifier
{
    // Shared implementation
    protected string Prefix => "[Notify]";

    public abstract Task NotifyAsync(string message);

    protected string Format(string message) => $"{Prefix} {message}";
}

public class EmailNotifier : NotifierBase
{
    public override Task NotifyAsync(string message)
    {
        Console.WriteLine(Format(message));
        return Task.CompletedTask;
    }
}
```

---

## 27) Explain SOLID with a practical C# example (OCP + SRP).

**Short answer:**
- SRP: one reason to change per class.
- OCP: open for extension, closed for modification.

```csharp
public interface IDiscountStrategy
{
    decimal Apply(decimal amount);
}

public class NoDiscount : IDiscountStrategy
{
    public decimal Apply(decimal amount) => amount;
}

public class SeasonalDiscount : IDiscountStrategy
{
    public decimal Apply(decimal amount) => amount * 0.9m;
}

public class CheckoutService
{
    // SRP: this class focuses on checkout pricing behavior
    public decimal CalculateTotal(decimal amount, IDiscountStrategy strategy)
    {
        // OCP: add new discount strategies without modifying this method
        return strategy.Apply(amount);
    }
}
```

---

## 28) How do you write unit tests effectively in C#?

**Short answer:**
Arrange-Act-Assert, test behavior not implementation details, mock external dependencies.

```csharp
using Xunit;

public class Calculator
{
    public int Add(int a, int b) => a + b;
}

public class CalculatorTests
{
    [Fact]
    public void Add_ReturnsSum()
    {
        // Arrange
        var sut = new Calculator();

        // Act
        var result = sut.Add(2, 3);

        // Assert
        Assert.Equal(5, result);
    }
}
```

---

## 29) What is `ConfigureAwait(false)` and when should you use it?

**Short answer:**
`ConfigureAwait(false)` avoids capturing context after await. In library code, it can reduce deadlock risk and improve throughput.

**Explanation:**
In ASP.NET Core, there is no legacy synchronization context, so impact is smaller, but still commonly used in reusable libraries.

```csharp
public static async Task<string> LoadDataAsync(HttpClient client)
{
    var response = await client.GetAsync("https://example.com")
        .ConfigureAwait(false); // continue without captured context

    response.EnsureSuccessStatusCode();
    return await response.Content.ReadAsStringAsync().ConfigureAwait(false);
}
```

---

## 30) How do you approach performance tuning in C# applications?

**Short answer:**
Measure first, optimize hotspots only, and verify gains. Focus on allocations, expensive I/O, database calls, and algorithmic complexity.

**Explanation:**
Senior engineers rely on profiling tools (dotTrace, PerfView, Application Insights, BenchmarkDotNet) and compare baseline vs changes.

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

public class StringBenchmarks
{
    private readonly string[] _items = Enumerable.Range(1, 1000)
        .Select(i => i.ToString())
        .ToArray();

    [Benchmark]
    public string ConcatWithPlus()
    {
        string result = string.Empty;
        foreach (var item in _items)
        {
            result += item; // many allocations
        }
        return result;
    }

    [Benchmark]
    public string ConcatWithStringBuilder()
    {
        var sb = new System.Text.StringBuilder();
        foreach (var item in _items)
        {
            sb.Append(item); // more allocation-friendly
        }
        return sb.ToString();
    }
}

// Run benchmarks in a console app entry point:
// BenchmarkRunner.Run<StringBenchmarks>();
```

---

## Rapid Fire Senior Follow-ups

1. `Task.WhenAll` vs `Task.WhenAny`?
2. How to prevent N+1 queries in EF Core?
3. `Any()` vs `Count() > 0` performance difference?
4. Why prefer `readonly` fields where possible?
5. What are source generators and when to use them?
6. How do you version APIs safely?
7. How do you design idempotent endpoints?
8. Difference between pessimistic and optimistic concurrency?
9. How do you handle distributed tracing in .NET microservices?
10. What metrics and logs do you monitor in production?

---

## Practice Plan

1. Pick 10 questions and answer each in under 2 minutes.
2. Implement 5 snippets in a sample .NET project.
3. Add unit tests for at least 3 examples.
4. Explain trade-offs (performance, readability, maintainability) for each design choice.
