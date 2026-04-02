# C# OOP Interview Q&A (Intermediate to Senior, 30 Questions)

This guide focuses on Object-Oriented Programming concepts in C#.
Each question includes:
- Short answer
- Explanation
- Code snippet with comments

---

## 1) What are the four pillars of OOP?

**Short answer:**
Encapsulation, Abstraction, Inheritance, and Polymorphism.

**Explanation:**
These pillars help organize code into reusable, maintainable, and extensible units.

```csharp
// Encapsulation: hide internal state
// Abstraction: expose only meaningful operations
// Inheritance: reuse base behavior
// Polymorphism: same method call, different behavior
public abstract class Animal
{
    public string Name { get; }

    protected Animal(string name)
    {
        Name = name;
    }

    public abstract void Speak(); // polymorphic contract
}

public sealed class Dog : Animal
{
    public Dog(string name) : base(name) { }

    public override void Speak() => Console.WriteLine($"{Name} barks");
}
```

---

## 2) What is encapsulation in C#?

**Short answer:**
Encapsulation means bundling data and behavior together, and restricting direct access to internal state.

```csharp
public class BankAccount
{
    // Private field cannot be modified directly from outside
    private decimal _balance;

    public decimal Balance => _balance; // controlled read access

    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive.");
        _balance += amount;
    }

    public void Withdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive.");
        if (amount > _balance) throw new InvalidOperationException("Insufficient funds.");
        _balance -= amount;
    }
}
```

---

## 3) What is abstraction, and how is it different from encapsulation?

**Short answer:**
Abstraction hides complexity and shows essential behavior; encapsulation protects internal state.

```csharp
public interface IPaymentProcessor
{
    Task<bool> PayAsync(decimal amount);
}

public class StripePaymentProcessor : IPaymentProcessor
{
    public Task<bool> PayAsync(decimal amount)
    {
        // Complex external API logic hidden behind interface
        return Task.FromResult(true);
    }
}
```

---

## 4) What is inheritance in C#?

**Short answer:**
Inheritance allows a derived class to reuse and extend a base class.

```csharp
public class Employee
{
    public string Name { get; set; } = string.Empty;

    public virtual decimal CalculateBonus() => 1000m;
}

public class Manager : Employee
{
    // Override base behavior
    public override decimal CalculateBonus() => 3000m;
}
```

---

## 5) What is polymorphism in C#?

**Short answer:**
Polymorphism allows one interface/base type to represent many concrete implementations.

```csharp
public class Circle : Shape
{
    public double Radius { get; set; }

    public override double Area() => Math.PI * Radius * Radius;
}

public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }

    public override double Area() => Width * Height;
}

public abstract class Shape
{
    public abstract double Area();
}

// Runtime polymorphism
Shape[] shapes =
{
    new Circle { Radius = 2 },
    new Rectangle { Width = 3, Height = 4 }
};

foreach (var shape in shapes)
{
    Console.WriteLine(shape.Area());
}
```

---

## 6) Difference between method overloading and overriding?

**Short answer:**
Overloading: same method name, different signatures (compile-time).
Overriding: derived class replaces virtual base method (runtime).

```csharp
public class Printer
{
    // Overloading
    public void Print(string value) => Console.WriteLine(value);
    public void Print(int value) => Console.WriteLine(value);
}

public class BaseLogger
{
    public virtual void Log(string message) => Console.WriteLine($"Base: {message}");
}

public class FileLogger : BaseLogger
{
    // Overriding
    public override void Log(string message) => Console.WriteLine($"File: {message}");
}
```

---

## 7) When do you use an abstract class vs an interface?

**Short answer:**
Use interface for contract-only behavior. Use abstract class when shared implementation/state is needed.

```csharp
public interface INotifier
{
    void Notify(string message);
}

public abstract class NotifierBase : INotifier
{
    protected string Prefix => "[Notify]";

    public abstract void Notify(string message);

    protected string Format(string message) => $"{Prefix} {message}";
}

public class EmailNotifier : NotifierBase
{
    public override void Notify(string message)
    {
        Console.WriteLine(Format(message));
    }
}
```

---

## 8) What is the `sealed` keyword used for?

**Short answer:**
`sealed` prevents inheritance (for class) or overriding (for method override).

```csharp
public class Vehicle
{
    public virtual void Start() => Console.WriteLine("Vehicle starting...");
}

public class Car : Vehicle
{
    public sealed override void Start() => Console.WriteLine("Car starting...");
}

// public class SportsCar : Car
// {
//     public override void Start() { } // Not allowed because Start is sealed
// }
```

---

## 9) What are access modifiers in C#?

**Short answer:**
They control visibility of types and members: `public`, `private`, `protected`, `internal`, `protected internal`, `private protected`.

```csharp
public class Parent
{
    private int _a = 1;              // only in this class
    protected int B = 2;             // this class + derived types
    internal int C = 3;              // same assembly
    protected internal int D = 4;    // derived OR same assembly
    private protected int E = 5;     // derived in same assembly

    public int GetA() => _a;
}
```

---

## 10) What is composition and why is it often preferred over inheritance?

**Short answer:**
Composition means building objects from smaller objects. It reduces tight coupling and deep inheritance chains.

```csharp
public interface IEngine
{
    void Start();
}

public class ElectricEngine : IEngine
{
    public void Start() => Console.WriteLine("Electric engine started.");
}

public class CarWithComposition
{
    private readonly IEngine _engine;

    // Inject behavior instead of inheriting from many base types
    public CarWithComposition(IEngine engine)
    {
        _engine = engine;
    }

    public void Start() => _engine.Start();
}
```

---

## 11) What is constructor chaining?

**Short answer:**
Constructor chaining reuses initialization logic between constructors using `this(...)` or `base(...)`.

```csharp
public class User
{
    public string Name { get; }
    public string Role { get; }

    public User(string name) : this(name, "User")
    {
        // Calls main constructor with default role
    }

    public User(string name, string role)
    {
        Name = name;
        Role = role;
    }
}
```

---

## 12) What is an object initializer and when to use it?

**Short answer:**
Object initializers allow setting properties during object creation, improving readability.

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
}

var product = new Product
{
    Id = 1,
    Name = "Keyboard",
    Price = 99.99m
};
```

---

## 13) What are static classes and static members?

**Short answer:**
Static members belong to the type, not to instances. Static classes cannot be instantiated.

```csharp
public static class MathHelper
{
    public static int Square(int x) => x * x;
}

int value = MathHelper.Square(5); // 25
```

---

## 14) What is the difference between `const` and `readonly`?

**Short answer:**
- `const`: compile-time constant.
- `readonly`: runtime-assigned once (constructor or declaration).

```csharp
public class Config
{
    public const int MaxRetryConst = 3; // compile-time
    public readonly int MaxRetryReadonly;

    public Config(int retry)
    {
        MaxRetryReadonly = retry; // can set in constructor
    }
}
```

---

## 15) Explain `virtual`, `override`, and `new`.

**Short answer:**
`virtual` allows override. `override` replaces base implementation polymorphically. `new` hides base member.

```csharp
public class BaseService
{
    public virtual void Execute() => Console.WriteLine("Base Execute");
    public void Ping() => Console.WriteLine("Base Ping");
}

public class DerivedService : BaseService
{
    public override void Execute() => Console.WriteLine("Derived Execute");

    public new void Ping() => Console.WriteLine("Derived Ping"); // hides, not polymorphic override
}
```

---

## 16) What is object slicing, and does C# have it?

**Short answer:**
C# does not have classic C++-style object slicing because classes are reference types.

```csharp
public class AnimalBase
{
    public virtual string Kind => "Animal";
}

public class DogDerived : AnimalBase
{
    public override string Kind => "Dog";
}

AnimalBase a = new DogDerived();
Console.WriteLine(a.Kind); // Dog (runtime dispatch preserved)
```

---

## 17) What are properties, and why avoid public fields?

**Short answer:**
Properties provide controlled access and validation. Public fields expose implementation details.

```csharp
public class Temperature
{
    private double _celsius;

    public double Celsius
    {
        get => _celsius;
        set
        {
            // Validation in setter
            if (value < -273.15) throw new ArgumentOutOfRangeException(nameof(value));
            _celsius = value;
        }
    }
}
```

---

## 18) What is immutability in OOP, and when is it useful?

**Short answer:**
Immutable objects cannot change after creation. They are safer for concurrency and easier to reason about.

```csharp
public sealed class Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    // Return new object instead of mutating
    public Money Add(decimal amountToAdd) => new Money(Amount + amountToAdd, Currency);
}
```

---

## 19) What is aggregation vs composition?

**Short answer:**
Aggregation: weak ownership (part can exist independently).
Composition: strong ownership (part lifecycle tied to whole).

```csharp
public class Department
{
    public string Name { get; set; } = string.Empty;
}

public class University
{
    // Aggregation: Department can exist independently
    public List<Department> Departments { get; } = new();
}

public class Engine
{
    public int HorsePower { get; set; }
}

public class Car
{
    // Composition: Engine created and owned by Car
    public Engine Engine { get; } = new Engine { HorsePower = 150 };
}
```

---

## 20) What is the Liskov Substitution Principle (LSP) with example?

**Short answer:**
Derived classes must be substitutable for base classes without breaking behavior.

```csharp
public abstract class Bird
{
    public abstract void Move();
}

public class Sparrow : Bird
{
    public override void Move() => Console.WriteLine("Sparrow flies.");
}

public class Penguin : Bird
{
    public override void Move() => Console.WriteLine("Penguin swims.");
}

// Both are substitutable for Bird in this abstraction
void MakeBirdMove(Bird bird) => bird.Move();
```

---

## 21) What is interface segregation principle (ISP)?

**Short answer:**
Clients should not depend on methods they do not use.

```csharp
public interface IPrinter
{
    void Print(string content);
}

public interface IScanner
{
    string Scan();
}

public class SimplePrinter : IPrinter
{
    public void Print(string content) => Console.WriteLine(content);
}

public class MultiFunctionDevice : IPrinter, IScanner
{
    public void Print(string content) => Console.WriteLine(content);
    public string Scan() => "scanned-content";
}
```

---

## 22) What is dependency inversion principle (DIP)?

**Short answer:**
High-level modules should depend on abstractions, not concrete implementations.

```csharp
public interface IMessageSender
{
    Task SendAsync(string msg);
}

public class SmsSender : IMessageSender
{
    public Task SendAsync(string msg)
    {
        Console.WriteLine($"SMS: {msg}");
        return Task.CompletedTask;
    }
}

public class NotificationService
{
    private readonly IMessageSender _sender;

    public NotificationService(IMessageSender sender)
    {
        _sender = sender;
    }

    public Task NotifyAsync(string message) => _sender.SendAsync(message);
}
```

---

## 23) What is explicit interface implementation?

**Short answer:**
It allows a class to implement interface members that are accessible only through interface reference.

```csharp
public interface IReader
{
    void Open();
}

public interface IWriter
{
    void Open();
}

public class FileHandler : IReader, IWriter
{
    // Explicit implementations resolve naming conflict
    void IReader.Open() => Console.WriteLine("Reader Open");
    void IWriter.Open() => Console.WriteLine("Writer Open");
}

var handler = new FileHandler();
((IReader)handler).Open();
((IWriter)handler).Open();
```

---

## 24) What are partial classes and when to use them?

**Short answer:**
Partial classes split a class definition across multiple files. Useful for generated code separation.

```csharp
// File A
public partial class Customer
{
    public int Id { get; set; }
}

// File B
public partial class Customer
{
    public string Name { get; set; } = string.Empty;
}

// Combined at compile time into one class
```

---

## 25) How do operator overloads fit in OOP design?

**Short answer:**
Operator overloads can improve domain readability, but should be intuitive and not surprising.

```csharp
public readonly struct Vector2
{
    public double X { get; }
    public double Y { get; }

    public Vector2(double x, double y)
    {
        X = x;
        Y = y;
    }

    // Overload + for vector addition
    public static Vector2 operator +(Vector2 a, Vector2 b) => new(a.X + b.X, a.Y + b.Y);
}

var v1 = new Vector2(1, 2);
var v2 = new Vector2(3, 4);
var sum = v1 + v2; // intuitive math-like syntax
```

---

## 26) What is the difference between shallow copy and deep copy?

**Short answer:**
Shallow copy duplicates top-level object but shares nested references. Deep copy duplicates full graph.

```csharp
public class Address
{
    public string City { get; set; } = string.Empty;
}

public class Employee
{
    public string Name { get; set; } = string.Empty;
    public Address Address { get; set; } = new();

    public Employee ShallowCopy() => (Employee)MemberwiseClone();

    public Employee DeepCopy()
    {
        return new Employee
        {
            Name = Name,
            Address = new Address { City = Address.City } // clone nested reference
        };
    }
}
```

---

## 27) What is object equality in C# (`==`, `Equals`, `GetHashCode`)?

**Short answer:**
For value-based objects, override `Equals` and `GetHashCode` consistently. `record` does this by default.

```csharp
public sealed class ProductCode : IEquatable<ProductCode>
{
    public string Code { get; }

    public ProductCode(string code) => Code = code;

    public bool Equals(ProductCode? other)
        => other is not null && Code == other.Code;

    public override bool Equals(object? obj)
        => obj is ProductCode other && Equals(other);

    public override int GetHashCode()
        => Code.GetHashCode(StringComparison.Ordinal);
}
```

---

## 28) What is the template method pattern using abstract classes?

**Short answer:**
Template Method defines algorithm structure in base class, while subclasses implement specific steps.

```csharp
public abstract class ReportGenerator
{
    public void Generate()
    {
        // Fixed algorithm flow
        LoadData();
        ProcessData();
        Export();
    }

    protected abstract void LoadData();
    protected abstract void ProcessData();

    protected virtual void Export()
    {
        Console.WriteLine("Default export");
    }
}

public class SalesReportGenerator : ReportGenerator
{
    protected override void LoadData() => Console.WriteLine("Load sales data");
    protected override void ProcessData() => Console.WriteLine("Process sales metrics");
}
```

---

## 29) What are common OOP anti-patterns in C# codebases?

**Short answer:**
God classes, deep inheritance, anemic domain model, tight coupling, and misuse of static/global state.

```csharp
// Anti-pattern example: "God service" doing too many responsibilities
public class GodService
{
    public void ValidateUser() { }
    public void SaveInvoice() { }
    public void SendEmail() { }
    public void GenerateReport() { }
    public void CallExternalApi() { }
}

// Better approach: split into focused services and compose via interfaces.
```

---

## 30) How do you design OOP code for testability?

**Short answer:**
Depend on interfaces, isolate side effects, keep methods small, and use constructor injection.

```csharp
public interface IClock
{
    DateTime UtcNow { get; }
}

public class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}

public class SubscriptionService
{
    private readonly IClock _clock;

    public SubscriptionService(IClock clock)
    {
        _clock = clock;
    }

    public bool IsExpired(DateTime expiryUtc)
    {
        // Deterministic and unit-test friendly logic
        return _clock.UtcNow > expiryUtc;
    }
}
```

---

## Rapid Fire OOP Follow-ups

1. Why avoid deep inheritance hierarchies?
2. How do `record` types change OOP modeling in C#?
3. When should behavior be in entity vs service?
4. Interface vs abstract class in plugin architecture?
5. How to enforce invariants in domain objects?
6. Difference between rich and anemic domain model?
7. Why is composition usually more flexible than inheritance?
8. When is a static helper acceptable?
9. How to model aggregate roots in DDD with C#?
10. How do you keep constructors from becoming too large?

---

## Practice Plan

1. Pick 10 questions and answer each in under 2 minutes.
2. Implement 5 snippets in a console app.
3. Refactor one inheritance-heavy design into composition.
4. Add unit tests to verify polymorphic behavior.
