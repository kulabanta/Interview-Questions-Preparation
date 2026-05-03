# Table of Contents
1. [Middleware Pipeline](#lld-interview-walkthrough--middleware-pipeline)

# LLD Interview Walkthrough — Middleware Pipeline

---

## Step 1 — Clarify Requirements (5 min)

Always open by asking these. It signals senior-level thinking.

### Functional Requirements

- Middleware should be registered in order and execute in that order
- Each middleware can run logic **before** and **after** calling the next one (two-pass)
- A middleware must be able to **short-circuit** the pipeline (e.g. auth failure, early response)
- Middleware must have access to the request and response context
- Support both **sync** and **async** middleware execution
- The pipeline must terminate with a final handler (terminal middleware)

### Non-Functional Requirements

- Extensible — new middleware should be pluggable without modifying existing code
- Performant — no unnecessary allocations in the hot path
- Testable — each middleware should be independently unit-testable
- Framework-agnostic core design (can be adapted to HTTP, CLI, messaging pipelines)

### Scope (what we are NOT designing)

- Not designing a full HTTP server
- Not designing routing or controller dispatch internals
- Not designing DI container internals (just assuming DI exists)

---

## Step 2 — Core Abstractions (5 min)

Define the key types before drawing anything.

```
RequestDelegate      → delegate Task(HttpContext context)
                       The unit of execution — every middleware AND the terminal handler
                       is ultimately a RequestDelegate.

IMiddleware          → Task InvokeAsync(HttpContext ctx, RequestDelegate next)
                       The contract every middleware implements.

IApplicationBuilder  → Use(Func<RequestDelegate, RequestDelegate> middleware)
                     → Build() → RequestDelegate
                       Collects middleware registrations and composes them into a chain.

HttpContext          → Carries Request, Response, User, Items (shared state bag), CancellationToken

MiddlewarePipeline   → Wraps IApplicationBuilder, exposes UseMiddleware<T>() convenience method

RequestDelegate      → The terminal/final handler (controller, minimal API endpoint, etc.)
```

**Key insight to state clearly:** Every component in the pipeline — including the final handler — is just a `RequestDelegate`. This unification is what makes the design elegant. Middleware is a function that takes a `RequestDelegate` and returns a new `RequestDelegate` that wraps it.

---

## Step 3 — Class Design (15 min)

### HttpContext

```csharp
public class HttpContext
{
    public HttpRequest  Request   { get; }
    public HttpResponse Response  { get; }
    public ClaimsPrincipal User   { get; set; }
    public IDictionary<object, object> Items { get; }  // shared state between middlewares
    public CancellationToken RequestAborted { get; }
    public IServiceProvider RequestServices { get; }   // scoped DI container
}

public class HttpRequest
{
    public string Method   { get; }
    public string Path     { get; }
    public IHeaderDictionary Headers { get; }
    public Stream Body     { get; }
    public IQueryCollection Query { get; }
}

public class HttpResponse
{
    public int StatusCode  { get; set; }
    public IHeaderDictionary Headers { get; }
    public Stream Body     { get; }
    public bool HasStarted { get; }   // CRITICAL — can't set headers after this
}
```

---

### RequestDelegate

```csharp
// The fundamental unit — everything is this
public delegate Task RequestDelegate(HttpContext context);
```

---

### IMiddleware Interface

```csharp
public interface IMiddleware
{
    Task InvokeAsync(HttpContext context, RequestDelegate next);
}
```

---

### IApplicationBuilder

```csharp
public interface IApplicationBuilder
{
    // Register a middleware factory function
    IApplicationBuilder Use(Func<RequestDelegate, RequestDelegate> middleware);

    // Terminal handler — what runs when nothing short-circuits
    IApplicationBuilder Run(RequestDelegate handler);

    // Convenience: branch the pipeline conditionally
    IApplicationBuilder UseWhen(
        Func<HttpContext, bool> predicate,
        Action<IApplicationBuilder> configure);

    // Compose all registered middleware into a single RequestDelegate
    RequestDelegate Build();
}
```

---

### ApplicationBuilder (Concrete Implementation)

```csharp
public class ApplicationBuilder : IApplicationBuilder
{
    private readonly List<Func<RequestDelegate, RequestDelegate>> _components = new();

    public IApplicationBuilder Use(Func<RequestDelegate, RequestDelegate> middleware)
    {
        _components.Add(middleware);
        return this;
    }

    public IApplicationBuilder Run(RequestDelegate handler)
    {
        // Terminal: ignores `next`, always ends the chain
        return Use(_ => handler);
    }

    public RequestDelegate Build()
    {
        // Start with a 404 fallback — if no terminal handler, return 404
        RequestDelegate pipeline = context =>
        {
            context.Response.StatusCode = 404;
            return Task.CompletedTask;
        };

        // Compose in REVERSE order so first registered runs first
        foreach (var component in _components.AsEnumerable().Reverse())
        {
            pipeline = component(pipeline);
        }

        return pipeline;
    }
}
```

> **Explain this clearly in the interview:**
> `Build()` iterates the list in reverse and wraps each component around the
> accumulated chain. The result is a single `RequestDelegate` where registration
> order is preserved at execution time.

---

### Concrete Middleware Examples

```csharp
// --- Logging Middleware ---
public class LoggingMiddleware : IMiddleware
{
    private readonly ILogger<LoggingMiddleware> _logger;

    public LoggingMiddleware(ILogger<LoggingMiddleware> logger)
        => _logger = logger;

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        _logger.LogInformation("→ [{Method}] {Path}", context.Request.Method, context.Request.Path);
        var sw = Stopwatch.StartNew();

        await next(context);  // ← hand off to next middleware

        sw.Stop();
        _logger.LogInformation("← [{Status}] {Elapsed}ms",
            context.Response.StatusCode, sw.ElapsedMilliseconds);
    }
}

// --- Auth Middleware (short-circuit example) ---
public class AuthMiddleware : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var token = context.Request.Headers["Authorization"].FirstOrDefault();

        if (string.IsNullOrEmpty(token))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("Unauthorized");
            return;   // ← short-circuit: next() is NEVER called
        }

        context.User = ValidateToken(token);
        await next(context);
    }
}

// --- Exception Handling Middleware (outermost layer) ---
public class ExceptionMiddleware : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        try
        {
            await next(context);
        }
        catch (Exception ex)
        {
            context.Response.StatusCode = 500;
            await context.Response.WriteAsync("Internal Server Error");
            // Log ex
        }
    }
}
```

---

### MiddlewarePipeline (Convenience Wrapper)

```csharp
public static class MiddlewareExtensions
{
    // Resolves middleware from DI and wraps it in the factory signature
    public static IApplicationBuilder UseMiddleware<T>(this IApplicationBuilder app)
        where T : IMiddleware
    {
        return app.Use(next => async context =>
        {
            var middleware = context.RequestServices.GetRequiredService<T>();
            await middleware.InvokeAsync(context, next);
        });
    }
}
```

---

### Wiring It All Together

```csharp
// Registration (Startup / Program.cs equivalent)
var builder = new ApplicationBuilder();

builder
    .UseMiddleware<ExceptionMiddleware>()  // outermost — catches everything below
    .UseMiddleware<LoggingMiddleware>()
    .UseMiddleware<AuthMiddleware>()
    .Run(async ctx =>
    {
        await ctx.Response.WriteAsync("Hello from handler!");
    });

var pipeline = builder.Build();

// Execution — called per request
await pipeline(httpContext);
```

---

## Step 4 — Request Flow (Draw This) (5 min)

```
Incoming Request
       │
       ▼
┌─────────────────────┐
│  ExceptionMiddleware │  ← outermost: catches all exceptions from below
│   try {             │
│     await next() ──────────────────────────────────────────┐
│   } catch { 500 }   │                                      │
└─────────────────────┘                                      │
                                                             ▼
                                              ┌─────────────────────┐
                                              │  LoggingMiddleware   │
                                              │  log request         │
                                              │  await next() ───────┐
                                              │  log response ◄──────┘
                                              └─────────────────────┘
                                                             │
                                                             ▼
                                              ┌─────────────────────┐
                                              │  AuthMiddleware      │
                                              │  if no token → 401  │  (short-circuit)
                                              │  else next() ────────┐
                                              └─────────────────────┘│
                                                                      ▼
                                                        ┌─────────────────────┐
                                                        │  Terminal Handler    │
                                                        │  200 OK "Hello!"     │
                                                        └─────────────────────┘

Response unwinds back through each middleware in reverse ▲
```

**Two-pass nature:**
- Code before `await next()` runs on the way **in** (request processing)
- Code after `await next()` runs on the way **out** (response processing)
- This is what enables: request logging, response compression, timing, etc. in a single component

---

## Step 5 — Design Patterns Used (5 min)

| Pattern | Where Applied | Why |
|---|---|---|
| Chain of Responsibility | Core pipeline execution | Each middleware decides whether to pass to next |
| Decorator | `Build()` composition | Each component wraps the next delegate, adding behaviour |
| Builder | `IApplicationBuilder` | Fluent API collects middleware, `Build()` constructs the chain |
| Template Method | `UseMiddleware<T>()` extension | Handles DI resolution boilerplate; subclass provides behaviour |
| Strategy | Middleware implementations | Different policies (auth, logging, rate limit) are interchangeable |

---

## Step 6 — Senior-Level Deep Dives (10 min)

These are the questions that separate senior candidates. Have crisp answers ready.

---

### Q: How do you handle exceptions thrown mid-pipeline?

Wrap the entire `await next(context)` in the outermost `ExceptionMiddleware` with a try-catch.
Register it first so it wraps all subsequent middleware. Any unhandled exception bubbles up through
the awaited call stack and is caught there. Never catch exceptions inside inner middleware unless
they have a specific recovery strategy — let them propagate.

---

### Q: How do middlewares share data without coupling to each other?

Use `HttpContext.Items` — a `Dictionary<object, object>` scoped to the request lifetime.
Middleware A can write `context.Items["UserId"] = 42`, Middleware B reads it.
Keys should be typed constants or dedicated record types to avoid string collisions.
Avoid static fields — they are shared across all requests and create concurrency bugs.

---

### Q: How does DI scoping work per request?

`HttpContext.RequestServices` is an `IServiceProvider` whose scope is created at the start of each
request and disposed at the end. Services registered as `Scoped` get one instance per request.
The scope boundary is the pipeline invocation itself — `Build()` produces the pipeline,
and each call to `pipeline(context)` is one request scope.

---

### Q: How would you conditionally run middleware for some paths only?

Implement `UseWhen()` — it branches the pipeline:

```csharp
public IApplicationBuilder UseWhen(
    Func<HttpContext, bool> predicate,
    Action<IApplicationBuilder> configure)
{
    var branch = new ApplicationBuilder();
    configure(branch);
    var branchPipeline = branch.Build();

    return Use(next => async context =>
    {
        if (predicate(context))
            await branchPipeline(context);
        // Note: branch does NOT automatically rejoin — use MapWhen for that
        await next(context);
    });
}
```

Example use: only run auth middleware for paths under `/api/`.

---

### Q: How do you make the pipeline thread-safe?

The pipeline itself (`RequestDelegate`) is stateless after `Build()`. It is built once at startup
and is immutable. Per-request state lives in `HttpContext`, which is created fresh per request and
is never shared across threads. Middleware instances resolved from DI as `Transient` or `Scoped`
are also per-request. Only `Singleton` middleware must be thread-safe — so avoid mutable instance
state in Singleton middleware.

---

### Q: How would you add rate limiting to this pipeline?

```csharp
public class RateLimitMiddleware : IMiddleware
{
    private readonly IRateLimiter _limiter;  // injected — e.g. token bucket implementation

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var clientId = context.User?.Identity?.Name ?? context.Connection.RemoteIpAddress?.ToString();

        if (!await _limiter.TryAcquireAsync(clientId))
        {
            context.Response.StatusCode = 429;
            context.Response.Headers["Retry-After"] = "60";
            return;
        }

        await next(context);
    }
}
```

Register it early in the pipeline, after auth (so you have a user identity) but before the handler.

---

## Step 7 — .NET-Specific Callouts (5 min)

Mention these to demonstrate real-world depth:

- **`IMiddleware` vs Convention-based middleware:** ASP.NET Core supports two styles.
  `IMiddleware` is the strongly-typed interface shown above. Convention-based uses `Invoke(HttpContext)`
  or `InvokeAsync(HttpContext)` with `next` injected by the framework — no interface required.

- **`WebApplication` vs `IApplicationBuilder`:** In .NET 6+, `WebApplication` implements
  `IApplicationBuilder` directly. `app.Use()`, `app.Run()`, `app.UseMiddleware<T>()` are all on the
  same object.

- **`IHostedService` for background pipelines:** For non-HTTP pipelines (e.g. message processing),
  the same middleware pattern can be applied inside an `IHostedService` with a custom context object.

- **`Channel<T>` for async pipelines:** If building a message processing pipeline, use
  `System.Threading.Channels.Channel<T>` as the transport between middleware stages.

- **Polly integration:** Exception handling middleware is a natural place to integrate Polly retry
  and circuit breaker policies for outbound calls made inside the handler.

---

## Trade-offs & Alternatives

| Decision | Trade-off |
|---|---|
| Reverse composition in `Build()` | Preserves intuitive registration order at the cost of a less obvious `Build()` implementation |
| `RequestDelegate` as a delegate (not interface) | Enables lambda middleware (simpler) but loses type safety vs a typed interface |
| `HttpContext.Items` for shared state | Simple but untyped — alternative is typed middleware output objects passed through the context |
| DI resolution per request via `RequestServices` | Correct scoping but slightly slower than cached resolution — acceptable for most workloads |
| Immutable pipeline after `Build()` | Cannot hot-reload middleware without restarting — trade-off for thread safety and performance |

---

## Summary — What to Say in Closing

> "The core insight in this design is that everything — middleware and handlers alike — converges
> to a single `RequestDelegate` type. `IApplicationBuilder.Build()` composes them in reverse
> registration order using functional wrapping, giving us a clean chain where each layer controls
> exactly when (and whether) it hands off to the next. The Chain of Responsibility and Decorator
> patterns work together here, and the Builder pattern makes registration fluent and readable.
> For a senior .NET role, I'd also call out that this design maps directly to how ASP.NET Core's
> own pipeline works — so battle-tested at scale."