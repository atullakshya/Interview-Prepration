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