# Dependency Injection & IoC Container in C#
### A Complete Reference for .NET Developers

---

## Table of Contents

1. [What Problem Does DI Solve?](#1-what-problem-does-di-solve)
2. [Core Concepts: IoC vs DI vs IoC Container](#2-core-concepts-ioc-vs-di-vs-ioc-container)
3. [The Three Injection Patterns](#3-the-three-injection-patterns)
4. [Service Lifetimes](#4-service-lifetimes)
5. [Full Implementation in ASP.NET Core](#5-full-implementation-in-aspnet-core)
6. [Advanced Patterns](#6-advanced-patterns)
7. [Unit Testing With DI](#7-unit-testing-with-di)
8. [Common Pitfalls & Anti-Patterns](#8-common-pitfalls--anti-patterns)
9. [Interview Questions (4 YOE Level)](#9-interview-questions-4-yoe-level)

---

## 1. What Problem Does DI Solve?

Without DI, classes create their own dependencies internally. This causes tight coupling, makes unit testing nearly impossible, and violates the Single Responsibility Principle.

### Tightly Coupled (Bad)

```csharp
public class OrderService
{
    // Hardcoded — impossible to swap, impossible to test in isolation
    private readonly EmailService _emailService = new EmailService();
    private readonly SqlOrderRepository _repo = new SqlOrderRepository("Server=...");

    public void PlaceOrder(int productId)
    {
        var order = new Order { ProductId = productId };
        _repo.Save(order);
        _emailService.Send("Order confirmed!");
    }
}
```

**Problems:**
- You cannot unit test `OrderService` without a real database and SMTP server
- Swapping `SqlOrderRepository` for `MongoRepository` requires editing `OrderService`
- The class controls its own dependencies — it "pulls" them rather than having them "pushed" in

### With Dependency Injection (Good)

```csharp
public class OrderService
{
    private readonly IEmailService _emailService;
    private readonly IOrderRepository _repo;

    // Dependencies declared openly and injected from outside
    public OrderService(IEmailService emailService, IOrderRepository repo)
    {
        _emailService = emailService;
        _repo = repo;
    }

    public async Task PlaceOrderAsync(int productId)
    {
        var order = new Order { ProductId = productId };
        await _repo.SaveAsync(order);
        await _emailService.SendAsync("Order confirmed!");
    }
}
```

**Benefits:**
- `OrderService` depends on *abstractions* (`IEmailService`, `IOrderRepository`), not concrete classes
- Any implementation can be passed in — real in production, mock in tests
- Adding a new dependency is visible at the constructor level

---

## 2. Core Concepts: IoC vs DI vs IoC Container

These three terms are related but distinct. Developers often use them interchangeably, which is technically imprecise.

| Term | What it is | Scope |
|---|---|---|
| **Inversion of Control (IoC)** | A design principle — give up control of object creation to an external entity | Broad principle |
| **Dependency Injection (DI)** | One specific implementation of IoC — dependencies are passed ("injected") from outside | Pattern |
| **IoC Container** | A framework that automates DI at scale, resolving entire object graphs | Tool |

### The Dependency Inversion Principle (the "D" in SOLID)

DI is the mechanism that enforces the Dependency Inversion Principle:

> *High-level modules should not depend on low-level modules. Both should depend on abstractions.*
> *Abstractions should not depend on details. Details should depend on abstractions.*

```
WITHOUT DIP:                    WITH DIP:
OrderService                    OrderService
    └── SqlOrderRepository          └── IOrderRepository (abstraction)
                                            └── SqlOrderRepository (detail)
                                            └── MongoOrderRepository (detail)
                                            └── InMemoryOrderRepository (detail, for tests)
```

---

## 3. The Three Injection Patterns

### Constructor Injection (Preferred)

Dependencies are passed through the constructor. This is the standard and recommended approach.

```csharp
public class OrderService : IOrderService
{
    private readonly IOrderRepository _repo;
    private readonly IEmailService _email;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IOrderRepository repo,
        IEmailService email,
        ILogger<OrderService> logger)
    {
        _repo   = repo   ?? throw new ArgumentNullException(nameof(repo));
        _email  = email  ?? throw new ArgumentNullException(nameof(email));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }
}
```

**Why it's preferred:**
- All dependencies are explicit and visible
- Cannot instantiate the class without satisfying its requirements
- Works seamlessly with all IoC containers
- Makes circular dependencies fail fast at startup

### Property Injection (Optional Dependencies)

Used when a dependency is truly optional. The class works without it, but uses it when available.

```csharp
public class OrderService : IOrderService
{
    // Optional: used if provided, skipped if null
    public ILogger? Logger { get; set; }

    public async Task PlaceOrderAsync(int productId)
    {
        Logger?.LogInformation("Placing order for product {Id}", productId);
        // ...
    }
}
```

> **Caution:** Property injection hides dependencies. Use it sparingly, only for genuinely optional cross-cutting concerns.

### Method Injection (Per-Operation Dependencies)

A dependency is needed only for a specific operation, not by the class as a whole.

```csharp
public class ReportGenerator
{
    // IDataSource passed only to Generate — not needed elsewhere in the class
    public Report Generate(IDataSource dataSource, ReportConfig config)
    {
        var rawData = dataSource.Fetch();
        return new Report(rawData, config);
    }
}
```

---

## 4. Service Lifetimes

This is the most critical concept for a 4 YOE developer. Choosing the wrong lifetime causes bugs that are extremely difficult to diagnose.

### The Three Lifetimes

```csharp
// In Program.cs / Startup.cs

// TRANSIENT: New instance created every single time it is requested
builder.Services.AddTransient<IEmailService, SmtpEmailService>();

// SCOPED: One instance per HTTP request (or per scope)
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();

// SINGLETON: One instance for the entire application lifetime
builder.Services.AddSingleton<ICache, RedisCache>();
```

### Lifetime Comparison Table

| Lifetime | New Instance Created | Use Case | Thread-safe? |
|---|---|---|---|
| **Transient** | Every time it's requested | Stateless services, validators, mappers | Not required |
| **Scoped** | Once per HTTP request | DbContext, Unit of Work, shopping cart | Within scope |
| **Singleton** | Once at app startup | Caches, config readers, HTTP client wrappers | Required |

### Visualising Lifetimes Across Requests

```
Request 1:          Request 2:
────────────────    ────────────────
Transient A1 ──┐   Transient A3 ──┐   ← Different every time
Transient A2 ──┘   Transient A4 ──┘

Scoped    B1 ───   Scoped    B2 ───   ← Same within request, new across requests

Singleton C1 ───   Singleton C1 ───   ← Always the same instance
```

### The Captive Dependency Problem

The most common lifetime-related bug. Occurs when a longer-lived service holds a reference to a shorter-lived one.

```csharp
// BUG: Singleton capturing a Scoped service
// DbContext is Scoped, but MyBackgroundService is Singleton.
// The DbContext injected at startup gets held forever — it will be disposed
// at the end of the first scope, leaving the singleton with a stale reference.

public class MyBackgroundService : BackgroundService
{
    private readonly AppDbContext _db; // WRONG! DbContext is Scoped, not Singleton-safe

    public MyBackgroundService(AppDbContext db) => _db = db;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        // _db is disposed after the first scope ends — this will throw ObjectDisposedException
        var orders = await _db.Orders.ToListAsync(ct);
    }
}
```

**Correct approach — use `IServiceScopeFactory`:**

```csharp
public class MyBackgroundService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public MyBackgroundService(IServiceScopeFactory scopeFactory)
        => _scopeFactory = scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            // Create a fresh scope per operation — safe!
            using var scope = _scopeFactory.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            var orders = await db.Orders.Where(o => !o.Processed).ToListAsync(ct);
            // process orders...

            await Task.Delay(TimeSpan.FromSeconds(30), ct);
        } // scope disposed here — DbContext cleaned up correctly
    }
}
```

---

## 5. Full Implementation in ASP.NET Core

A complete, realistic example of the full DI pipeline.

### Step 1: Define Abstractions

```csharp
// Abstractions — the "what", not the "how"
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id);
    Task<IEnumerable<Order>> GetAllAsync();
    Task SaveAsync(Order order);
}

public interface IEmailService
{
    Task SendAsync(string to, string subject, string body);
}

public interface IOrderService
{
    Task PlaceOrderAsync(int productId, int quantity);
    Task<Order?> GetOrderAsync(int id);
}
```

### Step 2: Implement the Concrete Classes

```csharp
// Concrete repository — depends on EF Core DbContext (also injected)
public class SqlOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;

    public SqlOrderRepository(AppDbContext db) => _db = db;

    public async Task<Order?> GetByIdAsync(int id)
        => await _db.Orders.FindAsync(id);

    public async Task<IEnumerable<Order>> GetAllAsync()
        => await _db.Orders.ToListAsync();

    public async Task SaveAsync(Order order)
    {
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
    }
}

// Concrete email service — depends on IOptions<SmtpSettings> (also injected)
public class SmtpEmailService : IEmailService
{
    private readonly SmtpSettings _settings;

    public SmtpEmailService(IOptions<SmtpSettings> options)
        => _settings = options.Value;

    public async Task SendAsync(string to, string subject, string body)
    {
        using var client = new SmtpClient(_settings.Host, _settings.Port);
        var message = new MailMessage(_settings.From, to, subject, body);
        await client.SendMailAsync(message);
    }
}

// Concrete order service — depends on both abstractions above
public class OrderService : IOrderService
{
    private readonly IOrderRepository _repo;
    private readonly IEmailService _email;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IOrderRepository repo,
        IEmailService email,
        ILogger<OrderService> logger)
    {
        _repo   = repo;
        _email  = email;
        _logger = logger;
    }

    public async Task PlaceOrderAsync(int productId, int quantity)
    {
        var order = new Order
        {
            ProductId  = productId,
            Quantity   = quantity,
            PlacedAt   = DateTime.UtcNow
        };

        await _repo.SaveAsync(order);

        _logger.LogInformation("Order {OrderId} placed for product {ProductId}", 
            order.Id, productId);

        await _email.SendAsync(
            to:      "customer@example.com",
            subject: "Order Confirmed",
            body:    $"Your order #{order.Id} has been placed successfully.");
    }

    public async Task<Order?> GetOrderAsync(int id)
        => await _repo.GetByIdAsync(id);
}
```

### Step 3: Register Everything (Composition Root)

```csharp
// Program.cs — the ONLY place where concrete types are mentioned
var builder = WebApplication.CreateBuilder(args);

// Infrastructure
builder.Services.AddDbContext<AppDbContext>(opts =>
    opts.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// Strongly-typed configuration
builder.Services.Configure<SmtpSettings>(
    builder.Configuration.GetSection("Smtp"));

// Business logic registrations
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();   // per request
builder.Services.AddTransient<IEmailService,  SmtpEmailService>();    // stateless
builder.Services.AddScoped<IOrderService,     OrderService>();        // per request

// Built-in: logging, controllers, etc.
builder.Services.AddControllers();

var app = builder.Build();
app.MapControllers();
app.Run();
```

### Step 4: Consume in a Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;

    // Container injects IOrderService automatically
    public OrdersController(IOrderService orderService)
        => _orderService = orderService;

    [HttpPost]
    public async Task<IActionResult> PlaceOrder([FromBody] PlaceOrderRequest req)
    {
        await _orderService.PlaceOrderAsync(req.ProductId, req.Quantity);
        return Ok(new { message = "Order placed successfully" });
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetOrder(int id)
    {
        var order = await _orderService.GetOrderAsync(id);
        return order is null ? NotFound() : Ok(order);
    }
}
```

---

## 6. Advanced Patterns

### Factory Registration (Complex Construction Logic)

When a service requires runtime logic or external resources to initialise:

```csharp
builder.Services.AddSingleton<ICache>(serviceProvider =>
{
    var config  = serviceProvider.GetRequiredService<IConfiguration>();
    var logger  = serviceProvider.GetRequiredService<ILogger<RedisCache>>();
    var connStr = config["Redis:ConnectionString"]
        ?? throw new InvalidOperationException("Redis connection string missing");

    return new RedisCache(connStr, retries: 3, logger: logger);
});
```

### Open Generic Registration

Register a generic type once and it works for all type arguments:

```csharp
// One registration covers IRepository<Order>, IRepository<Product>, etc.
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));

// Usage in a class
public class ProductService
{
    public ProductService(IRepository<Product> repo) { }
}
```

### Keyed Services (.NET 8+)

Multiple implementations of the same interface, selected by a key:

```csharp
// Registration
builder.Services.AddKeyedScoped<IPaymentGateway, StripeGateway>("stripe");
builder.Services.AddKeyedScoped<IPaymentGateway, PayPalGateway>("paypal");
builder.Services.AddKeyedScoped<IPaymentGateway, BankTransferGateway>("bank");

// Consumption — resolved by key
public class CheckoutService
{
    private readonly IPaymentGateway _gateway;

    public CheckoutService(
        [FromKeyedServices("stripe")] IPaymentGateway gateway)
    {
        _gateway = gateway;
    }
}

// Dynamic resolution — chosen at runtime
public class PaymentRouter
{
    private readonly IServiceProvider _sp;

    public PaymentRouter(IServiceProvider sp) => _sp = sp;

    public IPaymentGateway Resolve(string method)
        => _sp.GetRequiredKeyedService<IPaymentGateway>(method);
}
```

### Decorator Pattern (via Scrutor)

Wrap a service with additional behaviour (caching, logging, retries) without modifying the original class or its consumers:

```csharp
// Install Scrutor: dotnet add package Scrutor

builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();

// CachedOrderRepository wraps SqlOrderRepository transparently
builder.Services.Decorate<IOrderRepository, CachedOrderRepository>();

// The decorator
public class CachedOrderRepository : IOrderRepository
{
    private readonly IOrderRepository _inner;  // the real one
    private readonly ICache _cache;

    public CachedOrderRepository(IOrderRepository inner, ICache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Order?> GetByIdAsync(int id)
    {
        var cacheKey = $"order:{id}";
        var cached = await _cache.GetAsync<Order>(cacheKey);
        if (cached is not null) return cached;

        var order = await _inner.GetByIdAsync(id); // delegates to SqlOrderRepository
        if (order is not null)
            await _cache.SetAsync(cacheKey, order, TimeSpan.FromMinutes(5));

        return order;
    }

    public Task SaveAsync(Order order)     => _inner.SaveAsync(order);
    public Task<IEnumerable<Order>> GetAllAsync() => _inner.GetAllAsync();
}
```

### IOptions, IOptionsSnapshot, IOptionsMonitor

Three ways to consume strongly-typed configuration — each with different refresh behaviour:

```csharp
// appsettings.json: { "Smtp": { "Host": "smtp.example.com", "Port": 587 } }
public class SmtpSettings
{
    public string Host { get; set; } = string.Empty;
    public int Port    { get; set; }
    public string From { get; set; } = string.Empty;
}

builder.Services.Configure<SmtpSettings>(builder.Configuration.GetSection("Smtp"));
```

| Type | Lifetime | Reloads Config? | Use In |
|---|---|---|---|
| `IOptions<T>` | Singleton | Never | Singletons, Transients |
| `IOptionsSnapshot<T>` | Scoped | Per request | Scoped services |
| `IOptionsMonitor<T>` | Singleton | On file change | Background services |

```csharp
// IOptionsMonitor — live config updates for long-running services
public class NotificationService
{
    private readonly IOptionsMonitor<SmtpSettings> _monitor;

    public NotificationService(IOptionsMonitor<SmtpSettings> monitor)
    {
        _monitor = monitor;
        // Subscribe to changes
        _monitor.OnChange(settings =>
            Console.WriteLine($"SMTP config changed: {settings.Host}"));
    }

    public SmtpSettings CurrentSettings => _monitor.CurrentValue;
}
```

### Injecting into Middleware

Middleware has two injection points depending on the lifetime of the service:

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger; // Singleton — constructor OK

    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next   = next;
        _logger = logger;
    }

    // Scoped services are injected via InvokeAsync parameters, NOT the constructor
    public async Task InvokeAsync(HttpContext context, IOrderService orderService)
    {
        _logger.LogInformation("Request: {Method} {Path}", 
            context.Request.Method, context.Request.Path);

        await _next(context);
    }
}

// Register
app.UseMiddleware<RequestLoggingMiddleware>();
```

---

## 7. Unit Testing With DI

This is the primary motivation for DI. By depending on interfaces, every service can be tested in complete isolation.

### Testing With Moq

```csharp
// dotnet add package Moq
// dotnet add package xunit

public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _repoMock;
    private readonly Mock<IEmailService>    _emailMock;
    private readonly OrderService           _sut; // System Under Test

    public OrderServiceTests()
    {
        _repoMock  = new Mock<IOrderRepository>();
        _emailMock = new Mock<IEmailService>();
        var logger = Mock.Of<ILogger<OrderService>>();

        // Wire up manually — no container needed in unit tests
        _sut = new OrderService(_repoMock.Object, _emailMock.Object, logger);
    }

    [Fact]
    public async Task PlaceOrderAsync_SavesOrderToRepository()
    {
        // Arrange
        _repoMock.Setup(r => r.SaveAsync(It.IsAny<Order>()))
                 .Returns(Task.CompletedTask);
        _emailMock.Setup(e => e.SendAsync(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()))
                  .Returns(Task.CompletedTask);

        // Act
        await _sut.PlaceOrderAsync(productId: 1, quantity: 2);

        // Assert — verify repo was called exactly once
        _repoMock.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);
    }

    [Fact]
    public async Task PlaceOrderAsync_SendsConfirmationEmail()
    {
        // Arrange
        _repoMock.Setup(r => r.SaveAsync(It.IsAny<Order>())).Returns(Task.CompletedTask);
        _emailMock.Setup(e => e.SendAsync(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()))
                  .Returns(Task.CompletedTask);

        // Act
        await _sut.PlaceOrderAsync(productId: 1, quantity: 2);

        // Assert — email sent once, to any recipient
        _emailMock.Verify(
            e => e.SendAsync(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()),
            Times.Once);
    }

    [Fact]
    public async Task PlaceOrderAsync_WhenRepositoryFails_ThrowsException()
    {
        // Arrange — simulate a DB failure
        _repoMock.Setup(r => r.SaveAsync(It.IsAny<Order>()))
                 .ThrowsAsync(new Exception("DB connection lost"));

        // Act & Assert
        await Assert.ThrowsAsync<Exception>(() =>
            _sut.PlaceOrderAsync(productId: 1, quantity: 2));

        // Email should NOT have been sent
        _emailMock.Verify(
            e => e.SendAsync(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()),
            Times.Never);
    }
}
```

### Integration Testing With WebApplicationFactory

For testing the full DI pipeline end-to-end:

```csharp
// dotnet add package Microsoft.AspNetCore.Mvc.Testing

public class OrdersIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrdersIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _client = factory
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // Replace real repository with in-memory test double
                    services.AddScoped<IOrderRepository, InMemoryOrderRepository>();
                    // Replace real email with a no-op test double
                    services.AddTransient<IEmailService, NoOpEmailService>();
                });
            })
            .CreateClient();
    }

    [Fact]
    public async Task POST_PlaceOrder_Returns200()
    {
        var body    = JsonContent.Create(new { productId = 1, quantity = 2 });
        var response = await _client.PostAsync("/api/orders", body);

        response.EnsureSuccessStatusCode();
    }
}
```

---

## 8. Common Pitfalls & Anti-Patterns

### Anti-Pattern 1: Service Locator

Calling `IServiceProvider.GetService<T>()` inside business logic hides dependencies and makes testing painful.

```csharp
// BAD — Service Locator anti-pattern
public class OrderService
{
    private readonly IServiceProvider _sp;

    public OrderService(IServiceProvider sp) => _sp = sp;

    public async Task PlaceOrderAsync(int productId)
    {
        // Hidden dependency — not visible from constructor
        var repo  = _sp.GetRequiredService<IOrderRepository>();
        var email = _sp.GetRequiredService<IEmailService>();
        // ...
    }
}

// GOOD — explicit constructor injection
public class OrderService
{
    public OrderService(IOrderRepository repo, IEmailService email) { }
}
```

> **Exception:** Service Locator is acceptable in framework infrastructure code (factories, middleware), never in domain/business logic.

### Anti-Pattern 2: Constructor Over-Injection

More than ~4-5 constructor parameters usually signals the class is doing too much (SRP violation).

```csharp
// Smell — too many dependencies
public class GodService
{
    public GodService(
        IOrderRepository orderRepo,
        IProductRepository productRepo,
        ICustomerRepository customerRepo,
        IEmailService emailService,
        ISmsService smsService,
        IPushNotificationService pushService,
        IPaymentGateway paymentGateway,
        IInventoryService inventoryService,
        ILogger<GodService> logger)
    { }
}
```

**Fix:** Extract cohesive sub-services (`NotificationService` wrapping email/SMS/push, `FulfillmentService` wrapping inventory/payment).

### Anti-Pattern 3: Captive Dependency (Recap)

Singleton holding a Scoped or Transient service — see [Section 4](#the-captive-dependency-problem).

### Anti-Pattern 4: Using `new` Inside Registered Services

```csharp
// BAD — bypasses DI entirely
public class OrderService
{
    private readonly IOrderRepository _repo;

    public OrderService(IOrderRepository repo)
    {
        _repo = repo;
        // This logger is unregistered, unconfigured, and can't be swapped
        _logger = new Logger<OrderService>(new LoggerFactory());
    }
}
```

### Anti-Pattern 5: Resolving in the Constructor

```csharp
// BAD — resolving services eagerly in the constructor defeats lazy resolution
public class OrderService
{
    private readonly IEnumerable<IOrderRepository> _repos;

    public OrderService(IServiceProvider sp)
    {
        // Don't do this — inject the enumerable directly
        _repos = sp.GetServices<IOrderRepository>();
    }
}

// GOOD
public class OrderService
{
    public OrderService(IEnumerable<IOrderRepository> repos) { }
}
```

---

## 9. Interview Questions (4 YOE Level)

### Conceptual Questions

**Q: What is the difference between IoC, DI, and an IoC Container?**

IoC (Inversion of Control) is the broader principle — instead of a class controlling its own dependencies, that control is *inverted* to an external entity. Dependency Injection is one concrete implementation of IoC — the external entity *pushes* dependencies into the class, rather than the class pulling them. An IoC Container is a framework (like `Microsoft.Extensions.DependencyInjection`) that automates this process at scale, resolving full object graphs, managing lifetimes, and wiring up all dependencies automatically.

---

**Q: Explain the three service lifetimes and a real-world scenario for each.**

- **Transient** — a new instance every request. Best for stateless, lightweight services: validators, formatters, email senders. They hold no state, so sharing them is pointless; creating them is cheap.
- **Scoped** — one instance per HTTP request. Best for `DbContext`, Unit of Work, shopping cart. These need to maintain state within a request (e.g. EF Core tracks entity changes) but must be fresh for each new request to avoid cross-request contamination.
- **Singleton** — one instance for the app lifetime. Best for expensive-to-create, inherently shared, thread-safe resources: in-memory caches, `HttpClient` wrappers (avoids socket exhaustion), configuration readers.

---

**Q: What is the Captive Dependency problem and how do you fix it?**

A Singleton holds a reference to a Scoped or Transient service. Since the Singleton outlives the scope, it ends up holding a stale, potentially disposed object. The fix is `IServiceScopeFactory` — the Singleton injects the factory, then creates and disposes a fresh scope for each operation (instead of holding a dependency directly).

---

**Q: What is the Dependency Inversion Principle and how does DI implement it?**

DIP states that high-level modules should not depend on low-level modules — both should depend on abstractions. DI implements this by having classes declare interface dependencies in their constructors. The IoC container then provides the concrete implementation at runtime. The class never knows or cares which concrete type it receives.

---

### Implementation Questions

**Q: How do you register multiple implementations of the same interface?**

Three approaches:
1. Register all of them — `GetServices<IEnumerable<T>>()` returns all; `GetRequiredService<T>()` returns the last registered one.
2. Keyed services (.NET 8+) — `AddKeyedScoped<IPaymentGateway, StripeGateway>("stripe")` and resolve with `[FromKeyedServices("stripe")]`.
3. Factory pattern — register a factory delegate that picks the right implementation based on runtime input.

---

**Q: What is the difference between `GetService<T>()` and `GetRequiredService<T>()`?**

`GetService<T>` returns `null` if the type is not registered. `GetRequiredService<T>` throws `InvalidOperationException`. In application code, always prefer `GetRequiredService` — failing loudly at startup is infinitely better than a `NullReferenceException` at runtime in production. `GetService` is only appropriate when a dependency is truly optional.

---

**Q: How do you inject a Scoped service into Middleware?**

Scoped services cannot go into the Middleware constructor (Middleware is effectively a Singleton). Inject them via the `InvokeAsync` method parameters instead — the framework passes them per-request:

```csharp
public async Task InvokeAsync(HttpContext context, IScopedService scopedService)
{
    // scopedService is resolved fresh per request
    await _next(context);
}
```

---

**Q: How do you resolve circular dependencies?**

The container throws `InvalidOperationException` or `StackOverflowException`. Circular dependencies always indicate a design problem — usually one class doing too much. The fix is to extract a shared abstraction (a third service that both depend on) or to use an event/mediator pattern (`MediatR`) to decouple the communication entirely.

---

**Q: What is the Service Locator anti-pattern and why is it bad?**

Calling `serviceProvider.GetService<T>()` from inside business logic classes. It is bad because: (1) it hides dependencies — the class's constructor no longer tells you what it needs; (2) it makes testing harder — you need to set up a full `ServiceProvider` instead of just passing a mock; (3) it couples your domain code to the DI infrastructure. Dependency injection should flow into classes, not be fetched by them.

---

**Q: Explain `IOptions<T>` vs `IOptionsSnapshot<T>` vs `IOptionsMonitor<T>`.**

- `IOptions<T>` — Singleton. Configuration is read once at startup and never refreshed. Use in Singletons where config is static.
- `IOptionsSnapshot<T>` — Scoped. Re-reads configuration per HTTP request, supporting hot-reload. Cannot be used in Singletons.
- `IOptionsMonitor<T>` — Singleton. Supports live change notifications via `OnChange()`. Use in background services that need to react to config file changes at runtime.

---

**Q: When would you use the Decorator pattern with DI, and how?**

When you want to add cross-cutting behaviour (caching, logging, retry, auditing) to a service without modifying the original implementation or its consumers. Use the `Scrutor` library's `.Decorate<TInterface, TDecorator>()` method. The decorator is injected with the original service and delegates calls to it, adding behaviour before or after. The consumer sees only `IOrderRepository` — it has no idea whether it's talking to the SQL implementation or the cached wrapper.

---

**Q: How do you test a service that uses `IServiceScopeFactory`?**

```csharp
[Fact]
public async Task MyService_CreatesScope_AndResolvesDbContext()
{
    // Arrange
    var dbContextMock = new Mock<AppDbContext>();
    
    var scopeMock = new Mock<IServiceScope>();
    scopeMock.Setup(s => s.ServiceProvider.GetService(typeof(AppDbContext)))
             .Returns(dbContextMock.Object);

    var factoryMock = new Mock<IServiceScopeFactory>();
    factoryMock.Setup(f => f.CreateScope()).Returns(scopeMock.Object);

    var sut = new MyBackgroundService(factoryMock.Object);

    // Act & Assert as needed
}
```

---

**Q: What happens if you register a service but forget to add a dependency it needs?**

At runtime, the container throws `InvalidOperationException` with a clear message: *"Unable to resolve service for type 'IEmailService' while attempting to activate 'OrderService'."* This fails at the point of first resolution — at startup for Singletons (if you validate on build), or on first request for Scoped/Transient services.

To catch all missing registrations at startup:

```csharp
// Program.cs
builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = true;    // catches captive dependencies
    options.ValidateOnBuild = true;   // catches missing registrations at startup
});
```

---

## Summary

The core philosophy to internalize:

> **Your classes declare what they need. The container figures out how to build it.**

- Business logic depends on **interfaces**, never concrete classes
- Concrete types are mentioned in **one place only**: the Composition Root
- The container owns **object creation** and **lifetime management**
- Classes are **never responsible** for creating or locating their own dependencies

When applied consistently, this makes every class independently testable, every dependency swappable, and the entire application far easier to reason about, extend, and maintain.

---

*Reference: Microsoft Docs — Dependency injection in ASP.NET Core*  
*https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection*