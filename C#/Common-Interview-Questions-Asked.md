# C# Interview questions Asked

# Table of Contents
1. [Can we use `return` statement inside catch block in C#](#q-can-we-use-return-statement-inside-catch-block-in-c)
2. [why is string immutable in C# ](#q-why-is-string-immutable-in-c-)
3. [How can I implement a method with same signature which is already implemented by a parent class using an interface ?](#q-how-can-i-implement-a-method-with-same-signature-which-is-already-implemented-by-a-parent-class-using-an-interface-)
4. [How to run multiple threads in sequence ?](#q-how-to-run-multiple-threads-in-sequence-)
5. [What are the different ways for injecting dependency in .NET and give examples of some services ](#what-are-the-different-ways-for-injecting-dependency-in-net-and-give-examples-of-some-services)
6. [An api is taking so much time to respond ? what might be the reasons and How to fix it ?](#why-is-my-api-slow--diagnosis--fixes)
7. [Two apis are hitting the databases for write operations ? How can you maintain consistency in writing to the database?](#concurrent-api-write-consistency--keeping-your-database-safe)

## Q. Can we use `return` statement inside catch block in C#
**Answer:**<br>
- Yes, we absolutely can use a `return` statement inside a `catch`  block in C#.
- When the code hits a `return` statement inside a `catch` block, it evaluates the value to be returned and prepares to exit the method.
- If there is no `finally` block, the method terminates immediately and hands the value back to the caller.
```csharp
public int Divide(int a, int b)
{
    try
    {
        return a / b;
    }
    catch (DivideByZeroException)
    {
        Console.WriteLine("Cannot divide by zero!");
        // Returning a default/fallback value right from the catch
        return 0; 
    }
}
```
- If your `try-catch `structure includes a `finally` block, the `finally` block will always execute before the method actually returns to the caller, even if the `return` statement is inside the `try or catch` block.

```csharp
public string ProcessData()
{
    try
    {
        int x = 10;
        int y = 0;
        int result = x / y; // Throws exception
        return "Success";
    }
    catch (Exception ex)
    {
        Console.WriteLine("1. Inside Catch");
        return "Failed"; // Prepares to return "Failed", but waits...
    }
    finally
    {
        Console.WriteLine("2. Inside Finally");
    }
}
```
```
Output : 
1. Inside Catch
2. Inside Finally
```

## Q. why is string immutable in C# ?
**Answer:**
- In C#, a `string` is immutable, meaning once it is created in memory, its value can never be changed. When you concatenate, replace, or modify a string in your code, you aren't actually altering the original object; the runtime is creating a brand-new string in memory and updating your variable to point to it.

### here are four major reasons why:
#### 1. Memory Efficiency (The String Intern Pool)
- Because strings cannot be changed, the Common Language Runtime (CLR) can safely optimize memory by sharing string instances.

- This process is called `String Interning`. If you have three different variables in your code assigned the literal value "Hello", the CLR doesn't create three separate objects in the heap. It creates "Hello" once in a special area of memory called the `Intern Pool`, and makes all three variables point to that exact same memory address.

- If strings were mutable (changeable), changing one variable to "Help!" would accidentally change the other two variables as well! Immutability makes this memory-saving feature possible.

#### 2. Thread Safety
- Immutable objects are inherently thread-safe. Because a string's state cannot be modified after it's created, multiple threads can safely access and read the exact same string concurrently without needing complex `lock` statements or worrying about race conditions.

#### 3. Security
- Strings are heavily used for sensitive operations: database connection strings, network URLs, and file paths.

- Imagine a scenario where strings were mutable. You could pass a file path to a security validation method. After the validation passes, but right before the file is actually opened, a malicious background thread could mutate the string to point to a sensitive system file instead. By making strings immutable, C# guarantees that the string you validate is the exact same string you execute.

#### 4. Stability in Data Structures (Hash Codes)
- Strings are incredibly common as keys in data structures like `Dictionary<TKey, TValue>` and `HashSet<T>`.

- These collections rely on the object's Hash Code to sort and find data quickly. If you used a string as a dictionary key, and then later modified that string, its Hash Code would change. The dictionary would permanently lose track of where that data was stored, causing memory leaks and bugs. Immutability guarantees that a string's Hash Code will remain identical for its entire lifetime.
## Q. How can I implement a method with same signature which is already implemented by a parent class using an interface ?
- When a child class inherits from a parent class and implements an interface that shares the exact same method signature as the parent, you have a few different ways to handle it depending on the behavior you want.
### Scenario:
```csharp
public interface IWorker 
{ 
    void DoWork(); 
}

public class Parent 
{ 
    public void DoWork() 
    { 
        Console.WriteLine("Parent is working."); 
    } 
}
```
Now, you create a Child class: `public class Child : Parent, IWorker { }`.

How do you implement `DoWork()`? You have three main options.

### 1. Implicit Implementation (Do Nothing)
- If you simply declare the child class and do absolutely nothing, the C# compiler is smart enough to map the parent's method to the interface's requirement.

#### How it works: 
-  The Child class inherits Parent.DoWork(). Because that method's signature exactly matches IWorker.DoWork(), the interface is satisfied automatically.

#### When to use it: 
-  When the parent's logic is exactly what the interface requires, and the child doesn't need to change anything.

### 2. Explicit Interface Implementation
- If you want the Child class to have its own specific implementation for the interface, but leave the Parent method intact, you use `Explicit Interface Implementation`.

#### How it works: 
-  You implement the method by explicitly prefixing it with the interface name. You do not use access modifiers like `public`.

```csharp
public class Child : Parent, IWorker
{
    // Explicitly implementing the interface
    void IWorker.DoWork() 
    {
        Console.WriteLine("Child is working specifically for the interface.");
    }
}

Child myChild = new Child();
myChild.DoWork(); // Output: "Parent is working." (Calls the inherited method)

IWorker workerObj = myChild;
workerObj.DoWork(); // Output: "Child is working specifically for the interface."
```
- An explicitly implemented method is hidden from the normal class instance. 
- To call it, the caller must cast the object to the interface

### 3. Method Hiding (Using the new Keyword)
- If you want the `Child` class to completely replace the parent's method for all standard calls, but the parent's method wasn't marked as `virtual`, you can use the `new` keyword to "hide" or "shadow" the parent's method.

#### How it works: 
-  You declare the method in the child class with the `new` keyword. This method will automatically satisfy the IWorker interface.

#### When to use it: 
- When you don't control the Parent class (e.g., it's from a third-party library), it isn't virtual, and you need the child to have completely different default behavior.

```csharp
public class Child : Parent, IWorker
{
    // Hiding the parent's method
    public new void DoWork() 
    {
        Console.WriteLine("Child is working (hiding parent).");
    }
}
```
### 4. Overriding (If the Parent is virtual)
- If you actually own the Parent class code, the cleanest Object-Oriented approach is to mark the parent's method as `virtual` and `override` it in the child.
- Like method hiding, this overridden method will automatically satisfy the IWorker interface requirement.
```csharp
// In Parent class:
public virtual void DoWork() { ... }

// In Child class:
public override void DoWork() 
{ 
    Console.WriteLine("Child is overriding parent."); 
}
```
## Q. How to run multiple threads in sequence ?
Multiple approaches :

1. async / await : modern approach
2. Task.ContinueWith()
3. Thread.Join() 

## What are the different ways for injecting dependency in .NET and give examples of some services
### 1. What is Dependency Injection?
 
**Dependency Injection (DI)** is a design pattern where a class receives its dependencies from an external source rather than creating them itself. .NET has a built-in DI container (`IServiceProvider`) that manages object creation, lifetime, and disposal automatically.
 
```
Without DI                          With DI
──────────────────────────────      ──────────────────────────────
class OrderService {                class OrderService {
  private EmailService _email;        private IEmailService _email;
                                      
  public OrderService() {             public OrderService(IEmailService email) {
    _email = new EmailService();  ←     _email = email;   ← injected externally
  }                               }   }
}
// Tightly coupled                  // Loosely coupled, testable
// Hard to test or swap             // Easy to mock and replace
```
 
---
 
### 2. The Three Service Lifetimes
 
Before injection patterns, you must understand **lifetimes** — they control how long an instance lives.
 
```
Request comes in ──────────────────────────────────────────────► Request ends
│                                                                            │
│  Singleton  [═══════════════════ same instance, entire app lifetime ════] │
│                                                                            │
│  Scoped     [══════════ same instance within this request ═══════════]    │
│                                                                            │
│  Transient  [═══] new [═══] new [═══] new   ← new instance every resolve │
```
 
| Lifetime | Created | Disposed | Use for |
|---|---|---|---|
| **Singleton** | Once, at first resolve | App shutdown | Config, caches, HTTP clients |
| **Scoped** | Once per HTTP request | End of request | DbContext, unit of work |
| **Transient** | Every time resolved | After use | Lightweight, stateless services |
 
#### Registering lifetimes
 
```csharp
// Program.cs
builder.Services.AddSingleton<IConfigService, ConfigService>();
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddTransient<IEmailFormatter, EmailFormatter>();
 
// Also valid — register with a concrete type (no interface)
builder.Services.AddScoped<ReportGenerator>();
 
// Register with a factory for complex construction
builder.Services.AddSingleton<ICache>(sp =>
    new RedisCache(sp.GetRequiredService<IConfiguration>()["Redis:ConnectionString"]!));
```
 
---
 
### 3. Way 1 — Constructor Injection ⭐ (Most Common)
 
The container inspects the constructor of a class and **automatically resolves and injects** all parameters from the service registry.
 
#### How it works internally
 
```
1. You request OrderService from the container
2. Container inspects: OrderService(IEmailService, ILogger<OrderService>)
3. Container resolves IEmailService → finds EmailService registration
4. Container resolves ILogger<OrderService> → built-in logging factory
5. Container calls: new OrderService(emailService, logger)
6. Returns the fully constructed instance
```
 
#### Example
 
```csharp
// Service interfaces
public interface IEmailService
{
    Task SendAsync(string to, string subject, string body);
}
 
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id);
    Task SaveAsync(Order order);
}
 
// Implementation — all deps come via constructor
public class OrderService
{
    private readonly IEmailService _emailService;
    private readonly IOrderRepository _orderRepo;
    private readonly ILogger<OrderService> _logger;
 
    // ✅ Container reads this constructor and resolves each parameter
    public OrderService(
        IEmailService emailService,
        IOrderRepository orderRepo,
        ILogger<OrderService> logger)
    {
        _emailService = emailService;
        _orderRepo    = orderRepo;
        _logger       = logger;
    }
 
    public async Task PlaceOrderAsync(Order order)
    {
        _logger.LogInformation("Placing order {OrderId}", order.Id);
        await _orderRepo.SaveAsync(order);
        await _emailService.SendAsync(order.CustomerEmail, "Order Confirmed", $"Order {order.Id} placed.");
    }
}
```
 
#### Registration
 
```csharp
builder.Services.AddScoped<IEmailService, SmtpEmailService>();
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddScoped<OrderService>();
```
 
#### What kinds of services suit constructor injection?
 
| Service | Why |
|---|---|
| `DbContext` (EF Core) | Needed throughout the class lifetime |
| `ILogger<T>` | Logging used across multiple methods |
| `IConfiguration` | App settings read at startup and use |
| `IHttpClientFactory` | HTTP client access throughout the class |
| Repositories & domain services | Core business dependencies |
| `IOptions<T>` | Typed settings object |
 
> **Constructor injection is the default and preferred pattern.** It makes all dependencies explicit, visible, and easy to mock in unit tests.
 
---
 
### 4. Way 2 — Property (Setter) Injection
 
Dependencies are set via **public properties** rather than the constructor. .NET's built-in DI container does **not** support this natively — you must either set properties manually or use a third-party container like **Autofac**.
 
#### How it works internally
 
```
1. Container creates the object using a parameterless constructor
2. After construction, it sets the marked properties
3. (With Autofac): [Dependency] attribute marks injectable properties
4. Without a framework, you assign properties manually after resolution
```
 
#### Example — Manual property injection
 
```csharp
public class ReportGenerator
{
    // Optional dependency — has a default, replaced if available
    public IReportFormatter Formatter { get; set; } = new PlainTextFormatter();
    public ILogger<ReportGenerator>? Logger { get; set; }
 
    public string Generate(ReportData data)
    {
        Logger?.LogInformation("Generating report...");
        return Formatter.Format(data);
    }
}
 
// Manually wire it up
var generator = new ReportGenerator
{
    Formatter = new PdfFormatter(),
    Logger    = loggerFactory.CreateLogger<ReportGenerator>()
};
```
 
#### Example — With Autofac (attribute-based)
 
```csharp
// Install: Autofac.Extensions.DependencyInjection
using Autofac;
 
public class NotificationSender
{
    [Inject]   // Autofac resolves this property automatically
    public IEmailService EmailService { get; set; } = null!;
 
    [Inject]
    public ISmsService SmsService { get; set; } = null!;
 
    public void Notify(string message)
    {
        EmailService.Send(message);
        SmsService.Send(message);
    }
}
 
// Autofac container setup
var builder = new ContainerBuilder();
builder.RegisterType<SmtpEmailService>().As<IEmailService>();
builder.RegisterType<TwilioSmsService>().As<ISmsService>();
builder.RegisterType<NotificationSender>().PropertiesAutowired();
```
 
#### What kinds of services suit property injection?
 
| Service | Why |
|---|---|
| **Optional** dependencies with safe defaults | Won't break if not set |
| **Circular dependencies** (last resort) | Constructor injection breaks cycles |
| **Plugin architectures** | Properties set after object creation |
| Legacy code where constructors can't change | You can't modify the constructor |
 
> ⚠️ **Avoid property injection** unless necessary. It hides dependencies, makes classes harder to test, and creates risk of `NullReferenceException` if properties aren't set.
 
---
 
### 5. Way 3 — Method Injection
 
A dependency is passed **directly as a parameter to a method** rather than held for the class lifetime. It's not resolved by the container automatically — you pass it explicitly at the call site.
 
#### How it works internally
 
```
1. No container involvement at class construction
2. The caller resolves (or owns) the dependency
3. Caller passes it directly as a method argument
4. Class uses it for that single call only
```
 
#### Example
 
```csharp
public class InvoiceGenerator
{
    // No constructor deps — everything comes per method call
 
    // ✅ Dependency only needed for this one operation → pass it in
    public Invoice Generate(Order order, ITaxCalculator taxCalculator)
    {
        var tax = taxCalculator.Calculate(order.Total, order.Region);
 
        return new Invoice
        {
            OrderId    = order.Id,
            Subtotal   = order.Total,
            Tax        = tax,
            Total      = order.Total + tax,
            IssuedAt   = DateTime.UtcNow
        };
    }
}
 
// Caller resolves and passes it in
var taxCalc    = serviceProvider.GetRequiredService<ITaxCalculator>();
var generator  = new InvoiceGenerator();
var invoice    = generator.Generate(order, taxCalc);
```
 
#### ASP.NET Core — `[FromServices]` in Controllers (method injection by the framework)
 
ASP.NET Core supports method injection in controller actions natively using `[FromServices]`:
 
```csharp
[ApiController]
[Route("api/[controller]")]
public class ReportsController : ControllerBase
{
    // ✅ IReportService is resolved per-action, not held by the controller
    [HttpGet("{id}")]
    public async Task<IActionResult> GetReport(
        int id,
        [FromServices] IReportService reportService,   // ← method injection
        [FromServices] ILogger<ReportsController> logger)
    {
        logger.LogInformation("Generating report {Id}", id);
        var report = await reportService.GetAsync(id);
        return Ok(report);
    }
}
```
 
#### What kinds of services suit method injection?
 
| Service | Why |
|---|---|
| Services needed for **one specific operation** | No reason to hold it for the class lifetime |
| **Strategy pattern** implementations | Different algorithms passed per call |
| **Tax calculators, formatters, validators** | Operation-specific tools |
| `[FromServices]` in minimal API handlers | Lightweight endpoints with specific deps |
 
---
 
### 6. Way 4 — Resolving from `IServiceProvider` Directly (Service Locator)
 
Pull a service from the DI container **on demand**, at any point in code, using `IServiceProvider`.
 
#### How it works internally
 
```
1. IServiceProvider is injected into your class normally (constructor)
2. At runtime, you call .GetRequiredService<T>() or .GetService<T>()
3. The container resolves and returns the instance according to its lifetime
4. You are responsible for creating a scope if accessing Scoped services
```
 
#### Example — Basic service locator
 
```csharp
public class PluginRunner
{
    private readonly IServiceProvider _sp;
 
    public PluginRunner(IServiceProvider sp)
    {
        _sp = sp;
    }
 
    public void Run(string pluginName)
    {
        // Resolve the correct plugin dynamically at runtime
        var plugin = pluginName switch
        {
            "pdf"   => _sp.GetRequiredService<IPdfPlugin>(),
            "excel" => _sp.GetRequiredService<IExcelPlugin>(),
            "csv"   => _sp.GetRequiredService<ICsvPlugin>(),
            _       => throw new InvalidOperationException($"Unknown plugin: {pluginName}")
        };
 
        plugin.Execute();
    }
}
```
 
#### Example — Creating a scope manually (for Scoped services in Singletons)
 
This is a critical pattern for **Background Services**, which are Singletons but need Scoped services like `DbContext`:
 
```csharp
public class DataSyncService : BackgroundService
{
    private readonly IServiceProvider _sp;
 
    public DataSyncService(IServiceProvider sp) => _sp = sp;
 
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // ✅ Create a fresh scope → safe to resolve Scoped services
            using var scope  = _sp.CreateScope();
            var dbContext    = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            var syncService  = scope.ServiceProvider.GetRequiredService<ISyncService>();
 
            await syncService.SyncAsync(stoppingToken);
 
            // scope is disposed here → dbContext is cleaned up
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```
 
#### `GetService<T>` vs `GetRequiredService<T>`
 
```csharp
// Returns null if not registered — use when dep is truly optional
var logger = sp.GetService<ILogger<MyService>>();
 
// Throws InvalidOperationException if not registered — use when dep is required
var dbContext = sp.GetRequiredService<AppDbContext>();
```
 
#### What kinds of services suit service locator?
 
| Service | Why |
|---|---|
| **Dynamic/runtime selection** of implementations | Plugin systems, strategy patterns |
| **Background services** needing Scoped deps | Must manually create a scope |
| **Factory classes** that create varied types | Don't know the type at compile time |
| **Middleware** with conditional resolution | Resolve only when certain conditions met |
 
> ⚠️ **Avoid service locator** in regular application code. It hides dependencies, makes classes hard to test, and is considered an anti-pattern. Use it only where dynamic resolution is genuinely needed.
 
---
 
### 7. Way 5 — `IOptions<T>` Pattern (Configuration Injection)
 
A specialized injection pattern for **typed, strongly-typed configuration sections** from `appsettings.json`.
 
#### How it works internally
 
```
1. appsettings.json is loaded by the host builder
2. A POCO class is registered to bind to a config section
3. IOptions<T> / IOptionsSnapshot<T> / IOptionsMonitor<T> wraps it
4. Services receive the wrapper via constructor injection
5. .Value property returns the bound, typed configuration object
```
 
#### Configuration and POCO
 
```json
// appsettings.json
{
  "EmailSettings": {
    "SmtpHost": "smtp.example.com",
    "SmtpPort": 587,
    "FromAddress": "no-reply@example.com",
    "EnableSsl": true
  },
  "PaymentSettings": {
    "ApiKey": "sk_live_abc123",
    "Timeout": 30
  }
}
```
 
```csharp
// Strongly-typed POCOs
public class EmailSettings
{
    public string SmtpHost    { get; set; } = string.Empty;
    public int    SmtpPort    { get; set; }
    public string FromAddress { get; set; } = string.Empty;
    public bool   EnableSsl   { get; set; }
}
 
public class PaymentSettings
{
    public string ApiKey  { get; set; } = string.Empty;
    public int    Timeout { get; set; }
}
```
 
#### Registration
 
```csharp
// Program.cs
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection("EmailSettings"));
 
builder.Services.Configure<PaymentSettings>(
    builder.Configuration.GetSection("PaymentSettings"));
```
 
#### Three variants of IOptions
 
```csharp
// IOptions<T> — Singleton, config loaded once at startup, never refreshes
public class EmailService
{
    private readonly EmailSettings _settings;
 
    public EmailService(IOptions<EmailSettings> options)
    {
        _settings = options.Value;   // Reads once, never changes
    }
}
 
// IOptionsSnapshot<T> — Scoped, re-reads config per request (supports hot-reload)
public class PaymentService
{
    private readonly PaymentSettings _settings;
 
    public PaymentService(IOptionsSnapshot<PaymentSettings> options)
    {
        _settings = options.Value;   // Fresh per HTTP request
    }
}
 
// IOptionsMonitor<T> — Singleton, fires event when config file changes
public class FeatureFlagService
{
    private readonly IOptionsMonitor<FeatureFlags> _monitor;
 
    public FeatureFlagService(IOptionsMonitor<FeatureFlags> monitor)
    {
        _monitor = monitor;
 
        // React to config changes in real time
        _monitor.OnChange(flags =>
        {
            Console.WriteLine("Feature flags updated!");
        });
    }
 
    public bool IsEnabled(string flag) => _monitor.CurrentValue.EnabledFlags.Contains(flag);
}
```
 
| Variant | Lifetime | Hot Reload | Use for |
|---|---|---|---|
| `IOptions<T>` | Singleton | ❌ No | Static config (connection strings, SMTP) |
| `IOptionsSnapshot<T>` | Scoped | ✅ Per request | Per-request config (feature flags) |
| `IOptionsMonitor<T>` | Singleton | ✅ Real-time | Long-running services needing live config |
 
---
 
### 8. Way 6 — Factory Pattern Injection
 
Instead of injecting a service directly, inject a **factory** that creates instances on demand. Useful when you need multiple instances, runtime decisions, or instances with parameters the container can't know at registration time.
 
#### How it works internally
 
```
1. A factory interface/class is registered in the container
2. The factory itself is injected via constructor
3. At runtime, the factory creates and returns instances
4. The factory can use the container internally to resolve deps
```
 
#### Example — `Func<T>` as a factory
 
```csharp
// Register a Func<T> factory
builder.Services.AddTransient<IEmailService, SmtpEmailService>();
builder.Services.AddSingleton<Func<IEmailService>>(sp =>
    () => sp.GetRequiredService<IEmailService>());
 
// Consumer receives the factory
public class BulkEmailSender
{
    private readonly Func<IEmailService> _emailFactory;
 
    public BulkEmailSender(Func<IEmailService> emailFactory)
    {
        _emailFactory = emailFactory;
    }
 
    public async Task SendBulkAsync(List<string> addresses, string subject, string body)
    {
        foreach (var address in addresses)
        {
            // Fresh instance per email
            var emailService = _emailFactory();
            await emailService.SendAsync(address, subject, body);
        }
    }
}
```
 
#### Example — Named factory (select implementation by key)
 
```csharp
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(PaymentRequest request);
}
 
public class StripePaymentProcessor : IPaymentProcessor { /* ... */ }
public class PayPalPaymentProcessor : IPaymentProcessor { /* ... */ }
 
// Factory that picks by provider name
public class PaymentProcessorFactory
{
    private readonly IServiceProvider _sp;
 
    public PaymentProcessorFactory(IServiceProvider sp) => _sp = sp;
 
    public IPaymentProcessor Create(string provider) => provider switch
    {
        "stripe" => _sp.GetRequiredService<StripePaymentProcessor>(),
        "paypal" => _sp.GetRequiredService<PayPalPaymentProcessor>(),
        _        => throw new NotSupportedException($"Provider {provider} not supported")
    };
}
 
// Registration
builder.Services.AddScoped<StripePaymentProcessor>();
builder.Services.AddScoped<PayPalPaymentProcessor>();
builder.Services.AddScoped<PaymentProcessorFactory>();
 
// Usage
public class CheckoutService
{
    private readonly PaymentProcessorFactory _factory;
 
    public CheckoutService(PaymentProcessorFactory factory) => _factory = factory;
 
    public async Task CheckoutAsync(Cart cart, string provider)
    {
        var processor = _factory.Create(provider);   // Runtime decision
        await processor.ProcessAsync(new PaymentRequest(cart.Total));
    }
}
```
 
#### What kinds of services suit factory injection?
 
| Service | Why |
|---|---|
| **Multiple instances** needed at runtime | One-per-item in a bulk operation |
| **Named/keyed implementations** | Payment providers, storage backends |
| **Services with runtime parameters** | Can't register with unknown args upfront |
| **Short-lived objects** in long-lived services | Avoid holding stale scoped services |
 
---
 
### 9. Captive Dependency — The Lifetime Trap
 
The most dangerous DI mistake: injecting a **shorter-lived service into a longer-lived one**. The short-lived service gets "captured" and never refreshed.
 
```
❌ Danger Zone — Captive Dependencies
 
Singleton  ──injects──► Scoped    ← Scoped is now stuck alive for app lifetime
Singleton  ──injects──► Transient ← Transient is now stuck — defeats its purpose
Scoped     ──injects──► Transient ← Less dangerous, but Transient won't refresh
```
 
#### Example of the problem
 
```csharp
// DbContext is Scoped — one per HTTP request
public class AppDbContext : DbContext { }
 
// ❌ This will throw at runtime (or silently corrupt state)
public class DataCacheService   // Singleton
{
    private readonly AppDbContext _db;   // Scoped — captured forever!
 
    public DataCacheService(AppDbContext db)   // Runtime throws InvalidOperationException
    {
        _db = db;
    }
}
```
 
#### The fix — use `IServiceScopeFactory`
 
```csharp
public class DataCacheService   // Singleton
{
    private readonly IServiceScopeFactory _scopeFactory;   // ✅ Singleton-safe
 
    public DataCacheService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }
 
    public async Task RefreshCacheAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // use db safely — it's scoped correctly
    }
}
```
 
---
 
### 10. Full Comparison
 
| Pattern | DI Container? | Deps Visible? | Testability | Best For |
|---|---|---|---|---|
| **Constructor** | ✅ Automatic | ✅ Explicit | ✅ Easy to mock | Default — almost everything |
| **Property** | ⚠️ Manual/Autofac | ❌ Hidden | ⚠️ Harder | Optional deps, legacy code |
| **Method** | ❌ Manual | ✅ Explicit | ✅ Easy | Per-call deps, strategy pattern |
| **Service Locator** | ✅ On demand | ❌ Hidden | ❌ Hard | Dynamic resolution, background services |
| **IOptions\<T\>** | ✅ Automatic | ✅ Explicit | ✅ Easy | Configuration / settings |
| **Factory** | ✅ Via factory | ✅ Explicit | ✅ Easy | Runtime selection, multiple instances |
 
---
 
### 11. Summary
 
```
Register once in Program.cs
        │
        ▼
  AddSingleton / AddScoped / AddTransient
        │
        ▼
  IServiceProvider (the container)
        │
        ├─► Constructor Injection    → ✅ Use by default, always
        │
        ├─► Property Injection       → ⚠️ Optional deps, use sparingly
        │
        ├─► Method Injection         → ✅ Per-operation deps, [FromServices]
        │
        ├─► Service Locator          → ⚠️ Dynamic resolution only
        │
        ├─► IOptions<T>              → ✅ Always use for configuration
        │
        └─► Factory Pattern          → ✅ When type is known only at runtime
```
## Why Is My API Slow? — Diagnosis & Fixes

---

### 1. The Diagnostic Mindset

Before blindly optimizing, **measure first**. Every slow API has a bottleneck — and it's almost never where you think it is.

```
Slow API Request
      │
      ├── Is it always slow, or only sometimes?
      │         │                    │
      │      Always              Sometimes
      │         │                    │
      │    Structural           Load / Contention
      │    problem              problem
      │
      ├── Is it slow for all users, or just some?
      │         │                    │
      │       All               Some users
      │         │                    │
      │    Server-side          Network / Geography
      │    problem              / Auth problem
      │
      └── Did it become slow recently?
                │                    │
              Yes                    No
                │                    │
           Recent deploy /       Long-standing
           data growth            design flaw
```

---

### 2. The Layers Where Slowness Hides

```
Client Request
      │
      ▼
 [ Network / Latency ]          ← Possible slow layer
      │
      ▼
 [ Load Balancer / Reverse Proxy ]
      │
      ▼
 [ Middleware Pipeline ]         ← Possible slow layer
      │
      ▼
 [ Controller / Handler ]
      │
      ▼
 [ Business Logic / Services ]  ← Possible slow layer
      │
      ▼
 [ Database Queries ]           ← Most common slow layer
      │
      ▼
 [ External APIs / Services ]   ← Possible slow layer
      │
      ▼
 [ Response Serialization ]     ← Possible slow layer
      │
      ▼
Client Receives Response
```

---

### 3. Reason 1 — Slow or Inefficient Database Queries

The **#1 cause** of slow APIs. Even a single bad query can make an endpoint take seconds.

#### Symptoms
- API is slow but CPU and memory are fine
- Slow only when data volume grows
- Database CPU spikes during the slow request

#### Common culprits

**Missing indexes**
```sql
-- ❌ Full table scan — reads every row
SELECT * FROM Orders WHERE CustomerId = 42;

-- ✅ Fix: Add an index on the filtered column
CREATE INDEX IX_Orders_CustomerId ON Orders(CustomerId);
```

**N+1 Query Problem**
```csharp
// ❌ Fires 1 query for orders + N queries for each customer
var orders = await db.Orders.ToListAsync();
foreach (var order in orders)
{
    var customer = order.Customer.Name;  // New query per order!
}

// ✅ Fix: Eager load with Include()
var orders = await db.Orders
    .Include(o => o.Customer)
    .ToListAsync();
```

**Loading more data than needed**
```csharp
// ❌ Loads ALL columns and ALL rows into memory
var orders = await db.Orders.ToListAsync();
var recent = orders.Where(o => o.Date > DateTime.UtcNow.AddDays(-7)).ToList();

// ✅ Fix: Filter and project in the query, not in memory
var recent = await db.Orders
    .Where(o => o.Date > DateTime.UtcNow.AddDays(-7))
    .Select(o => new OrderSummaryDto(o.Id, o.Total, o.Date))
    .ToListAsync();
```

**Long-running transactions holding locks**
```csharp
// ❌ Transaction held open while doing unrelated work
using var tx = await db.Database.BeginTransactionAsync();
await db.Orders.AddAsync(order);
await SendEmailAsync(order);        // ← This is slow! Transaction blocks other readers
await db.SaveChangesAsync();
await tx.CommitAsync();

// ✅ Fix: Commit first, do side-effects after
using var tx = await db.Database.BeginTransactionAsync();
await db.Orders.AddAsync(order);
await db.SaveChangesAsync();
await tx.CommitAsync();
await SendEmailAsync(order);        // ← Now outside the transaction
```

#### Fixes

```csharp
// 1. Use AsNoTracking() for read-only queries (skips EF change tracker)
var orders = await db.Orders
    .AsNoTracking()
    .Where(o => o.Status == "Active")
    .ToListAsync();

// 2. Use compiled queries for hot paths
private static readonly Func<AppDbContext, int, Task<Order?>> GetOrderById =
    EF.CompileAsyncQuery((AppDbContext db, int id) =>
        db.Orders.FirstOrDefault(o => o.Id == id));

var order = await GetOrderById(db, 42);

// 3. Use raw SQL for complex reporting queries
var results = await db.Database
    .SqlQuery<SalesReportDto>($"EXEC GetMonthlySalesReport {month}, {year}")
    .ToListAsync();
```

---

### 4. Reason 2 — Synchronous Code Blocking Threads

Blocking threads with `.Result`, `.Wait()`, or missing `async/await` prevents the server from handling other requests — starving the thread pool.

#### Symptoms
- CPU is low but response time is high
- Slowness worsens under concurrent load
- Thread pool exhaustion warnings in logs

#### The problem

```csharp
// ❌ .Result blocks the calling thread synchronously
public IActionResult GetOrder(int id)
{
    var order = _orderService.GetByIdAsync(id).Result;     // Thread blocked here
    var customer = _customerService.GetAsync(order.CustomerId).Wait(); // Another block
    return Ok(order);
}

// ❌ Sync-over-async — looks async, still blocks
public async Task<IActionResult> GetOrder(int id)
{
    var order = _orderService.GetByIdAsync(id).Result;     // Still blocking!
    return Ok(order);
}
```

#### The fix

```csharp
// ✅ Fully async — thread is freed while awaiting I/O
public async Task<IActionResult> GetOrder(int id)
{
    var order = await _orderService.GetByIdAsync(id);
    return Ok(order);
}

// ✅ Run independent async calls in parallel
public async Task<IActionResult> GetDashboard(int userId)
{
    // ❌ Sequential — total time = A + B + C
    var orders   = await _orderService.GetAsync(userId);
    var profile  = await _profileService.GetAsync(userId);
    var alerts   = await _alertService.GetAsync(userId);

    // ✅ Parallel — total time = max(A, B, C)
    var ordersTask  = _orderService.GetAsync(userId);
    var profileTask = _profileService.GetAsync(userId);
    var alertsTask  = _alertService.GetAsync(userId);

    await Task.WhenAll(ordersTask, profileTask, alertsTask);

    return Ok(new {
        Orders  = ordersTask.Result,
        Profile = profileTask.Result,
        Alerts  = alertsTask.Result
    });
}
```

---

### 5. Reason 3 — No Caching

Recomputing or refetching the same data on every request is wasteful. If data doesn't change often, **cache it**.

#### Symptoms
- Same expensive query runs on every request
- High database load for data that rarely changes
- Response time drops immediately after first request (in testing)

#### Fix 1 — In-Memory Cache (`IMemoryCache`)

```csharp
public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly AppDbContext _db;

    public ProductService(IMemoryCache cache, AppDbContext db)
    {
        _cache = cache;
        _db    = db;
    }

    public async Task<List<Product>> GetFeaturedAsync()
    {
        const string cacheKey = "featured-products";

        // Return from cache if available
        if (_cache.TryGetValue(cacheKey, out List<Product>? cached))
            return cached!;

        // Otherwise fetch from DB and cache it
        var products = await _db.Products
            .Where(p => p.IsFeatured)
            .AsNoTracking()
            .ToListAsync();

        _cache.Set(cacheKey, products, new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
            SlidingExpiration               = TimeSpan.FromMinutes(2)
        });

        return products;
    }
}

// Registration
builder.Services.AddMemoryCache();
```

#### Fix 2 — Distributed Cache (Redis)

```csharp
// Registration
builder.Services.AddStackExchangeRedisCache(options =>
    options.Configuration = builder.Configuration["Redis:ConnectionString"]);

public class CatalogService
{
    private readonly IDistributedCache _cache;

    public async Task<CategoryDto?> GetCategoryAsync(int id)
    {
        var key      = $"category:{id}";
        var cached   = await _cache.GetStringAsync(key);

        if (cached is not null)
            return JsonSerializer.Deserialize<CategoryDto>(cached);

        var category = await FetchFromDbAsync(id);

        await _cache.SetStringAsync(key,
            JsonSerializer.Serialize(category),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
            });

        return category;
    }
}
```

#### Fix 3 — Response Caching (HTTP-level)

```csharp
// Cache entire HTTP responses at the middleware level
[HttpGet("products/featured")]
[ResponseCache(Duration = 60, VaryByHeader = "Accept")]
public async Task<IActionResult> GetFeatured()
{
    var products = await _productService.GetFeaturedAsync();
    return Ok(products);
}

// Program.cs
builder.Services.AddResponseCaching();
app.UseResponseCaching();
```

---

### 6. Reason 4 — Slow or Unreliable External HTTP Calls

If your API calls a third-party service (payment gateway, SMS provider, weather API), that external call becomes your bottleneck.

#### Symptoms
- Slow only when specific features are used
- Errors or timeouts correlate with third-party service issues
- Request duration matches exactly with third-party SLA

#### Problems and fixes

**No timeout set — waits forever**
```csharp
// ❌ Default HttpClient has no timeout → hangs indefinitely
var response = await _httpClient.GetAsync("https://slow-api.com/data");

// ✅ Always set a timeout
_httpClient.Timeout = TimeSpan.FromSeconds(5);
var response = await _httpClient.GetAsync("https://slow-api.com/data");
```

**Creating new HttpClient per request — socket exhaustion**
```csharp
// ❌ New HttpClient each time — exhausts sockets
public async Task<string> FetchAsync()
{
    using var client = new HttpClient();   // Socket not released immediately!
    return await client.GetStringAsync("https://api.example.com");
}

// ✅ Use IHttpClientFactory — manages pooled connections
public class WeatherService
{
    private readonly HttpClient _client;

    public WeatherService(IHttpClientFactory factory)
    {
        _client = factory.CreateClient("WeatherApi");
    }

    public async Task<WeatherDto?> GetWeatherAsync(string city)
    {
        var response = await _client.GetFromJsonAsync<WeatherDto>($"/weather?city={city}");
        return response;
    }
}

// Registration with typed client and timeout
builder.Services.AddHttpClient("WeatherApi", client =>
{
    client.BaseAddress = new Uri("https://api.weather.com");
    client.Timeout     = TimeSpan.FromSeconds(5);
});
```

**No retry / circuit breaker — fails fast or retries forever**
```csharp
// ✅ Use Polly for retry + circuit breaker (via Microsoft.Extensions.Http.Polly)
builder.Services.AddHttpClient("PaymentApi")
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3, attempt =>
            TimeSpan.FromSeconds(Math.Pow(2, attempt))))   // Exponential backoff: 2s, 4s, 8s
    .AddTransientHttpErrorPolicy(p =>
        p.CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30)));   // Open circuit for 30s after 5 failures
```

**Not running external calls in parallel**
```csharp
// ❌ Sequential — total = 400ms + 300ms + 200ms = 900ms
var weather   = await weatherClient.GetAsync();    // 400ms
var inventory = await inventoryClient.GetAsync();  // 300ms
var pricing   = await pricingClient.GetAsync();    // 200ms

// ✅ Parallel — total = max(400, 300, 200) = 400ms
await Task.WhenAll(
    weatherClient.GetAsync(),
    inventoryClient.GetAsync(),
    pricingClient.GetAsync()
);
```

---

### 7. Reason 5 — Heavy Serialization / Large Payloads

Serializing or deserializing huge JSON payloads consumes significant CPU and memory.

#### Symptoms
- High CPU during the slow request
- Memory spikes on large dataset endpoints
- Slow even when DB query is fast

#### Fixes

**Return only what the client needs**
```csharp
// ❌ Returns 50 fields when client needs 3
public async Task<IActionResult> GetOrders()
{
    var orders = await db.Orders.ToListAsync();   // Full entity
    return Ok(orders);
}

// ✅ Project to a lean DTO
public async Task<IActionResult> GetOrders()
{
    var orders = await db.Orders
        .Select(o => new OrderSummaryDto(o.Id, o.Status, o.Total))
        .ToListAsync();
    return Ok(orders);
}
```

**Use `System.Text.Json` with source generation (faster serialization)**
```csharp
// Define a serialization context — eliminates reflection at runtime
[JsonSerializable(typeof(List<OrderSummaryDto>))]
[JsonSerializable(typeof(ProductDto))]
public partial class AppJsonContext : JsonSerializerContext { }

// Register in Program.cs
builder.Services.ConfigureHttpJsonOptions(options =>
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));
```

**Enable response compression**
```csharp
// Program.cs
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
});

builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
    options.Level = CompressionLevel.Fastest);

app.UseResponseCompression();   // Before other middleware
```

**Stream large responses instead of buffering**
```csharp
// ✅ Stream a large dataset directly — no full buffer in memory
[HttpGet("export")]
public async IAsyncEnumerable<OrderDto> ExportOrders(
    [EnumeratorCancellation] CancellationToken ct)
{
    await foreach (var order in db.Orders.AsAsyncEnumerable().WithCancellation(ct))
    {
        yield return new OrderDto(order);
    }
}
```

---

### 8. Reason 6 — No Pagination on Large Result Sets

Returning thousands of records in one response is a major source of slowness.

#### Symptoms
- Endpoint is fast with little data, slow with lots of data
- Unbounded growth in response time as data grows

#### Fix — Implement cursor or offset pagination

```csharp
// Request model
public record PagedRequest(int Page = 1, int PageSize = 20);

// Response wrapper
public record PagedResponse<T>(
    List<T> Data,
    int     Page,
    int     PageSize,
    int     TotalCount,
    int     TotalPages);

// ✅ Paginated endpoint
[HttpGet("orders")]
public async Task<IActionResult> GetOrders([FromQuery] PagedRequest request)
{
    var query      = db.Orders.AsNoTracking().OrderByDescending(o => o.CreatedAt);
    var totalCount = await query.CountAsync();

    var orders = await query
        .Skip((request.Page - 1) * request.PageSize)
        .Take(request.PageSize)
        .Select(o => new OrderSummaryDto(o.Id, o.Status, o.Total))
        .ToListAsync();

    return Ok(new PagedResponse<OrderSummaryDto>(
        orders,
        request.Page,
        request.PageSize,
        totalCount,
        (int)Math.Ceiling(totalCount / (double)request.PageSize)));
}
```

---

### 9. Reason 7 — Middleware / Logging Overhead

Middleware runs on every request. Expensive or misconfigured middleware can silently add latency.

#### Symptoms
- All endpoints are equally slow
- Latency exists even for trivial endpoints
- Profiler shows time spent before the controller is even reached

#### Fixes

```csharp
// ❌ Logging everything synchronously with full detail
app.Use(async (context, next) =>
{
    var body = await new StreamReader(context.Request.Body).ReadToEndAsync(); // Reads entire body!
    _logger.LogInformation("Request body: {Body}", body);
    await next();
});

// ✅ Log only what you need, asynchronously
app.Use(async (context, next) =>
{
    _logger.LogInformation("Request: {Method} {Path}", context.Request.Method, context.Request.Path);
    await next();
});

// ✅ Use structured logging with minimum level filtering
builder.Logging.AddFilter("Microsoft.EntityFrameworkCore.Database.Command", LogLevel.Warning);
// Prevents EF Core from logging every SQL statement in production
```

**Order middleware correctly** — fast-exit middleware should come first:

```csharp
// Program.cs — correct order matters
app.UseResponseCompression();   // ① Compress early
app.UseResponseCaching();       // ② Return cached responses before auth
app.UseAuthentication();        // ③ Auth
app.UseAuthorization();         // ④ Authz
app.MapControllers();           // ⑤ Finally hit the controller
```

---

### 10. Reason 8 — Memory Pressure & Garbage Collection

Allocating too many short-lived objects forces frequent GC pauses, adding latency spikes.

#### Symptoms
- Latency spikes at irregular intervals (GC pauses)
- High Gen2 GC collections in metrics
- Memory steadily climbs then drops sharply

#### Fixes

```csharp
// ❌ Allocating new strings and lists in hot loops
public string BuildCsv(List<Order> orders)
{
    var result = "";
    foreach (var o in orders)
        result += $"{o.Id},{o.Total}\n";   // New string allocation every iteration
    return result;
}

// ✅ Use StringBuilder
public string BuildCsv(List<Order> orders)
{
    var sb = new StringBuilder(orders.Count * 30);
    foreach (var o in orders)
        sb.AppendLine($"{o.Id},{o.Total}");
    return sb.ToString();
}

// ✅ Use ArrayPool / MemoryPool for temporary buffers
var pool   = ArrayPool<byte>.Shared;
var buffer = pool.Rent(4096);
try
{
    // use buffer
}
finally
{
    pool.Return(buffer);    // Return to pool instead of GC
}

// ✅ Use object pooling for expensive objects (e.g., regex, StringBuilder)
var pool = ObjectPool.Create<StringBuilder>();
var sb   = pool.Get();
try { /* use sb */ }
finally { pool.Return(sb); }
```

---

### 11. How to Measure — Tools & Techniques

#### Add timing middleware (quick & zero-dependency)

```csharp
app.Use(async (context, next) =>
{
    var sw = Stopwatch.StartNew();
    await next();
    sw.Stop();

    var duration = sw.ElapsedMilliseconds;
    context.Response.Headers["X-Response-Time-ms"] = duration.ToString();

    if (duration > 500)   // Log slow requests only
        _logger.LogWarning("Slow request: {Method} {Path} took {Duration}ms",
            context.Request.Method, context.Request.Path, duration);
});
```

#### Use MiniProfiler (per-request breakdown)

```bash
dotnet add package MiniProfiler.AspNetCore.Mvc
dotnet add package MiniProfiler.EntityFrameworkCore
```

```csharp
builder.Services.AddMiniProfiler(options =>
    options.RouteBasePath = "/profiler"
).AddEntityFramework();

app.UseMiniProfiler();
```

Visit `/profiler/results-index` in dev to see SQL queries, timing per step, and duplicate queries.

#### Application Insights / OpenTelemetry

```csharp
// OpenTelemetry — production-grade distributed tracing
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter());
```

---

### 12. Quick Diagnostic Checklist

```
When an API is slow, go through this list in order:

[ ] 1. Check the database
        → Are there missing indexes?
        → Are there N+1 queries?  (Check logs for repeated queries)
        → Is a query doing a full table scan?

[ ] 2. Check for blocking code
        → Is .Result or .Wait() used anywhere?
        → Are independent calls sequential instead of parallel?

[ ] 3. Check caching
        → Is frequently-read, rarely-changed data being cached?
        → Is the cache configured correctly (TTL, size limits)?

[ ] 4. Check external calls
        → Is there a timeout set on all HTTP clients?
        → Are retries causing cascading slowness?
        → Can external calls be parallelized or made async?

[ ] 5. Check payload size
        → Is the endpoint returning more data than needed?
        → Is pagination missing on list endpoints?
        → Is compression enabled?

[ ] 6. Check middleware
        → Is expensive logic running in middleware on every request?
        → Is logging level too verbose in production?

[ ] 7. Check memory
        → Are there GC pause spikes?
        → Are large allocations happening in hot paths?
```

---

### 13. Summary — Causes & Fixes at a Glance

| Cause | Symptom | Fix |
|---|---|---|
| Missing DB index | Slow on large data, high DB CPU | Add index on filtered/joined columns |
| N+1 queries | Many identical queries in logs | `Include()`, `Select()` projection |
| Sync blocking | Slow under concurrency, thread starvation | Use `async/await` end-to-end |
| No caching | Same query on every request | `IMemoryCache`, Redis, `ResponseCache` |
| Slow external API | Slow only for specific features | Timeout, retry, Polly, parallelism |
| Large payloads | High CPU, memory spike | DTOs, pagination, compression, streaming |
| Middleware overhead | All endpoints equally slow | Audit middleware, fix logging level |
| GC pressure | Irregular latency spikes | Pooling, `StringBuilder`, `ArrayPool` |

## Concurrent API Write Consistency — Keeping Your Database Safe

---

### 1. The Problem

Two API requests arrive at the **same millisecond**, both read the same row, both modify it, and both try to save. One silently overwrites the other.

```
Time ──────────────────────────────────────────────────────────────►

Request A:  READ stock=100 ──────────────────── WRITE stock=90  ✅
                                                        │
Request B:  READ stock=100 ──── WRITE stock=95  ✅      │
                                      │                  │
                                      └── Overwrites A ──┘
                                          Final: stock=95
                                          Expected: stock=85  ❌ WRONG
```

This is called a **Lost Update** — one of the writes is silently discarded.

#### Three classic concurrent write problems

| Problem | What Happens |
|---|---|
| **Lost Update** | Both requests read the same value, last write wins and discards the first |
| **Dirty Read** | Request B reads data that Request A wrote but hasn't committed yet |
| **Phantom Write** | Request A checks a condition ("is username taken?"), Request B inserts in between, Request A proceeds assuming it's safe |

---

### 2. Solution 1 — Optimistic Concurrency (Row Versioning)

**Assume conflicts are rare.** Let both requests proceed, but detect at save time if the row was changed since it was read. If yes — reject the second writer.

No locks held. High throughput. Best for low-conflict data.

#### How it works

```
Request A: READ (stock=100, version=1) ────────────────── WRITE → version bumped to 2 ✅
Request B: READ (stock=100, version=1) ── WRITE → version=1 no longer matches ❌ REJECTED
```

#### Implementation with EF Core

```csharp
// 1. Add a RowVersion column to the entity
public class Inventory
{
    public int    Id       { get; set; }
    public int    Stock    { get; set; }

    [Timestamp]                           // EF Core manages this automatically
    public byte[] RowVersion { get; set; } = [];
}
```

```csharp
// 2. EF Core includes the RowVersion in the WHERE clause automatically
// Generated SQL:
// UPDATE Inventory SET Stock = 90
// WHERE Id = 1 AND RowVersion = 0x000000000000007E   ← must still match
// If 0 rows affected → someone else changed it → throws

public async Task DeductStockAsync(int productId, int quantity)
{
    try
    {
        var inventory = await db.Inventory.FindAsync(productId);
        inventory!.Stock -= quantity;
        await db.SaveChangesAsync();       // ✅ Succeeds only if RowVersion still matches
    }
    catch (DbUpdateConcurrencyException ex)
    {
        // Row was changed by another request between our read and write
        var entry = ex.Entries.Single();
        await entry.ReloadAsync();          // Reload latest values from DB

        // Option A — Retry with fresh data
        var fresh = (Inventory)entry.Entity;
        fresh.Stock -= quantity;
        await db.SaveChangesAsync();

        // Option B — Return 409 Conflict to the client and let them retry
        throw new ConflictException("Data was modified by another request. Please retry.");
    }
}
```

> ✅ **Use when:** Conflicts are rare — user profile updates, content edits, order status changes.
> ❌ **Avoid when:** High-contention rows hit by many requests per second.

---

### 3. Solution 2 — Pessimistic Locking (SELECT FOR UPDATE)

**Assume conflicts are likely.** Lock the row the moment you read it. Any other request trying to read that row **waits** until you commit or roll back.

#### How it works

```
Request A: READ + LOCK row ──── WRITE ──── COMMIT (lock released) ✅
Request B: READ + LOCK ──── WAITS ──────────────────────────────── Proceeds with fresh data ✅
```

#### Implementation

```csharp
public async Task BookSeatAsync(int seatId, string userId)
{
    using var tx = await db.Database.BeginTransactionAsync(IsolationLevel.RepeatableRead);

    // Lock this specific row — other requests block here until we commit
    var seat = await db.Seats
        .FromSqlRaw("SELECT * FROM Seats WITH (UPDLOCK, ROWLOCK) WHERE Id = {0}", seatId)
        .FirstOrDefaultAsync();

    if (seat is null || seat.IsBooked)
    {
        await tx.RollbackAsync();
        throw new InvalidOperationException("Seat unavailable.");
    }

    seat.IsBooked = true;
    seat.BookedBy = userId;

    await db.SaveChangesAsync();
    await tx.CommitAsync();    // Lock released — next request proceeds
}
```

> ✅ **Use when:** Conflicts are frequent and correctness is critical — seat booking, ticket sales, bank transfers.
> ❌ **Avoid when:** High concurrency with long-running operations — locks create queues and hurt throughput.

---

### 4. Solution 3 — Atomic Database Operations

Instead of read → modify → write (three steps a race can slip into), do it in **one atomic SQL operation**. The database engine handles the concurrency — no application-level locking needed.

```csharp
// ❌ Three steps — race condition window between READ and WRITE
var inventory = await db.Inventory.FindAsync(productId);  // Step 1: Read
inventory.Stock -= quantity;                               // Step 2: Modify in memory
await db.SaveChangesAsync();                               // Step 3: Write (stale!)

// ✅ One atomic step — no race window
await db.Inventory
    .Where(i => i.Id == productId && i.Stock >= quantity)  // Guard condition included
    .ExecuteUpdateAsync(s =>
        s.SetProperty(i => i.Stock, i => i.Stock - quantity));
```

```sql
-- What actually runs — atomic, no race condition possible
UPDATE Inventory
SET Stock = Stock - @quantity
WHERE Id = @productId AND Stock >= @quantity;
```

```csharp
// Check if the update actually applied (0 rows = stock was insufficient or race lost)
var rowsAffected = await db.Inventory
    .Where(i => i.Id == productId && i.Stock >= quantity)
    .ExecuteUpdateAsync(s => s.SetProperty(i => i.Stock, i => i.Stock - quantity));

if (rowsAffected == 0)
    throw new InvalidOperationException("Insufficient stock or concurrent update conflict.");
```

> ✅ **Use when:** Simple numeric updates — counters, stock levels, balance deductions, vote counts.
> ✅ Fastest option — no extra round-trips, no application locks, minimal overhead.

---

### 5. Solution 4 — Serializable Isolation Level

Tell the database to treat concurrent transactions **as if they ran one at a time**, in serial order. The DB engine handles all conflict detection automatically.

```csharp
public async Task CreateUniqueUsernameAsync(string username)
{
    using var tx = await db.Database.BeginTransactionAsync(IsolationLevel.Serializable);

    // Serializable prevents another transaction from inserting
    // the same username between this check and our insert
    var exists = await db.Users.AnyAsync(u => u.Username == username);

    if (exists)
    {
        await tx.RollbackAsync();
        throw new ConflictException("Username already taken.");
    }

    db.Users.Add(new User { Username = username });
    await db.SaveChangesAsync();
    await tx.CommitAsync();
}
```

> ✅ **Use when:** You need a check-then-insert guarantee — unique usernames, coupon codes, "only one winner" logic.
> ❌ Highest isolation cost — can cause deadlocks under heavy load. Add retry logic.

---

### 6. Solution 5 — Application-Level Locking (Distributed Lock)

When you need to ensure **only one request at a time** runs a critical section — across multiple server instances — use a distributed lock backed by Redis.

```bash
dotnet add package RedLockNet.SERedis
```

```csharp
public class InventoryService
{
    private readonly IDistributedLockFactory _lockFactory;

    public async Task DeductStockAsync(int productId, int quantity)
    {
        var resource  = $"lock:inventory:{productId}";  // Unique key per product
        var expiry    = TimeSpan.FromSeconds(10);        // Auto-release if process crashes

        await using var redLock = await _lockFactory.CreateLockAsync(resource, expiry);

        if (!redLock.IsAcquired)
            throw new ConflictException("Resource is locked by another request. Try again.");

        // ✅ Only ONE request executes this block at a time — across all server instances
        var inventory = await db.Inventory.FindAsync(productId);
        inventory!.Stock -= quantity;
        await db.SaveChangesAsync();

        // Lock auto-released when using block exits
    }
}
```

> ✅ **Use when:** Multiple server instances (load balanced) and you need process-level mutual exclusion.
> ✅ Good for flash sales, limited-quantity drops, one-time operations.
> ❌ Redis becomes a dependency — adds latency and a failure point.

---

### 7. Choosing the Right Solution

```
Are conflicts RARE or FREQUENT?
│
├── RARE (most user-facing writes)
│     └──► Optimistic Concurrency (RowVersion)
│           → Reject the loser, let client retry
│
├── FREQUENT + simple numeric update
│     └──► Atomic DB Operation (ExecuteUpdateAsync)
│           → Fastest, no locks, no round-trips
│
├── FREQUENT + complex logic before writing
│     └──► Pessimistic Locking (SELECT FOR UPDATE)
│           → Serialize access at the DB row level
│
├── CHECK then INSERT must be atomic
│     └──► Serializable Isolation
│           → DB prevents phantom writes
│
└── Multiple servers + mutual exclusion needed
      └──► Distributed Lock (Redis)
            → One request at a time across all instances
```

---

### 8. Quick Comparison

| Solution | Locks DB? | Throughput | Complexity | Best For |
|---|---|---|---|---|
| **Optimistic Concurrency** | ❌ No | ✅ High | Low | Low-conflict updates |
| **Pessimistic Locking** | ✅ Yes (row) | ⚠️ Medium | Low | High-conflict critical rows |
| **Atomic DB Operation** | ❌ No | ✅ Highest | Low | Counters, balances, stock |
| **Serializable Isolation** | ✅ Yes (range) | ❌ Low | Medium | Unique check-then-insert |
| **Distributed Lock** | ❌ No (app-level) | ⚠️ Medium | High | Multi-instance mutual exclusion |