# Circuit Breaker Pattern

## Table of Contents
1. [What Is the Circuit Breaker Pattern?](#what-is-the-circuit-breaker-pattern)
2. [The Problem It Solves](#the-problem-it-solves)
3. [States of a Circuit Breaker](#states-of-a-circuit-breaker)
4. [Real-World Analogy](#real-world-analogy)
5. [When to Use It](#when-to-use-it)
6. [When NOT to Use It](#when-not-to-use-it)
7. [Example: Without Circuit Breaker](#example-without-circuit-breaker)
8. [Example: With Circuit Breaker (Manual Implementation)](#example-with-circuit-breaker-manual-implementation)
9. [Implementing with Polly](#implementing-with-polly)
   - [Basic Circuit Breaker](#1-basic-circuit-breaker)
   - [Advanced Circuit Breaker](#2-advanced-circuit-breaker)
   - [Circuit Breaker + Retry](#3-combining-circuit-breaker-with-retry)
   - [Circuit Breaker + Fallback](#4-combining-circuit-breaker-with-fallback)
   - [Using ResiliencePipeline (Polly v8+)](#5-using-resiliencepipeline-polly-v8)
10. [Monitoring Circuit Breaker State](#monitoring-circuit-breaker-state)
11. [Key Configuration Knobs](#key-configuration-knobs)
12. [Summary](#summary)

---

## What Is the Circuit Breaker Pattern?

The **Circuit Breaker** is a resilience design pattern used in distributed systems to **detect failures and prevent cascading failures** across services. It acts as a proxy between a caller and a service, monitoring calls for failures. When failures exceed a threshold, the circuit "opens" and subsequent calls **fail immediately** without even attempting to reach the failing service.

The name is inspired by electrical circuit breakers that cut power when a fault is detected — preventing damage from spreading.

---

## The Problem It Solves

In a microservices architecture, services call each other over the network. When one service becomes slow or unavailable:

- Callers **block waiting** for timeouts (e.g., 30 seconds each)
- Thread pools get **exhausted**
- Upstream services degrade and eventually **fail too**
- The entire system can collapse — a **cascading failure**

```
User → API Gateway → Order Service → Payment Service (💥 DOWN)
                              ↑
               Threads pile up waiting... Order Service dies too
                              ↑
               API Gateway runs out of threads... it dies too
```

The Circuit Breaker stops this domino effect by **failing fast** instead of waiting.

---

## States of a Circuit Breaker

A Circuit Breaker has **three states**:

```
               failure threshold exceeded
    CLOSED ──────────────────────────────────► OPEN
      ▲                                           │
      │                                           │ after timeout period
      │    success                                ▼
      └──────────────────────────────── HALF-OPEN
                                    (probe allowed)
```

### 🟢 CLOSED (Normal Operation)
- All requests pass through to the service
- Failures are counted
- If failures exceed the **threshold** → circuit **opens**

### 🔴 OPEN (Failing Fast)
- All requests **immediately fail** with an exception (no network call made)
- After a configured **duration**, the circuit moves to HALF-OPEN

### 🟡 HALF-OPEN (Testing Recovery)
- A **limited number** of probe requests are allowed through
- If they succeed → circuit **closes** (service is healthy)
- If they fail → circuit **opens again** (service still down)

---

## Real-World Analogy

Imagine a restaurant where the kitchen is on fire:

- **CLOSED**: Kitchen is working. Orders go in, food comes out.
- **OPEN**: Kitchen is on fire. The manager immediately tells waiters *"Stop taking orders — the kitchen is down."* No waiter needs to walk to the kitchen to find out.
- **HALF-OPEN**: Fire is out. Manager allows one order through to test if the kitchen is really back. If it succeeds, full service resumes.

---

## When to Use It

✅ Use the Circuit Breaker pattern when:

| Scenario | Reason |
|---|---|
| Calling **external HTTP APIs** | Remote services can go down or slow down |
| Calling **downstream microservices** | Prevent cascading failures across services |
| Accessing **databases** under load | DB contention can cause timeouts |
| Integrating with **third-party services** | Payment gateways, SMS, email providers |
| High-traffic systems with **shared thread pools** | Avoid resource exhaustion |
| Operations with **expensive retries** | Avoid hammering an already-failing service |

---

## When NOT to Use It

❌ Avoid Circuit Breaker when:

- The operation is **local** (in-memory, no network) — overhead isn't justified
- Failures are **expected and transient** → use Retry instead
- The operation is **idempotent and fast** → retry is simpler
- You have a **monolith** with no distributed calls

---

## Example: Without Circuit Breaker

```csharp
// ❌ Without circuit breaker — threads pile up on failure
public async Task<Order> GetOrderAsync(int orderId)
{
    // If PaymentService is down, this hangs for 30 seconds
    // 100 concurrent users = 100 threads blocked for 30s each
    var result = await _httpClient.GetAsync($"https://payment-service/orders/{orderId}");
    result.EnsureSuccessStatusCode();
    return await result.Content.ReadFromJsonAsync<Order>();
}
```

**What happens when PaymentService goes down:**
1. Each request waits 30 seconds for a timeout
2. 100 simultaneous users → 100 threads blocked for 30s
3. Thread pool exhausted → OrderService stops responding
4. API Gateway starts timing out → API Gateway dies
5. 💥 Full system outage

---

## Example: With Circuit Breaker (Manual Implementation)

Here's a simplified circuit breaker to understand the mechanics:

```csharp
public class SimpleCircuitBreaker
{
    private CircuitState _state = CircuitState.Closed;
    private int _failureCount = 0;
    private DateTime _openedAt;

    private readonly int _failureThreshold = 5;
    private readonly TimeSpan _openDuration = TimeSpan.FromSeconds(30);

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        if (_state == CircuitState.Open)
        {
            // Check if it's time to try again
            if (DateTime.UtcNow - _openedAt > _openDuration)
                _state = CircuitState.HalfOpen;
            else
                throw new CircuitBreakerOpenException("Circuit is OPEN. Failing fast.");
        }

        try
        {
            var result = await action();

            // Success — reset
            _failureCount = 0;
            _state = CircuitState.Closed;
            return result;
        }
        catch (Exception)
        {
            _failureCount++;
            if (_failureCount >= _failureThreshold)
            {
                _state = CircuitState.Open;
                _openedAt = DateTime.UtcNow;
                Console.WriteLine("⚡ Circuit OPENED after too many failures.");
            }
            throw;
        }
    }
}

public enum CircuitState { Closed, Open, HalfOpen }
```

---

## Implementing with Polly

**Polly** is the de-facto .NET library for resilience and transient fault handling. It provides a production-grade circuit breaker with thread-safety, metrics, and composability.

### Install Polly

```bash
# For .NET / ASP.NET Core projects
dotnet add package Polly
dotnet add package Polly.Extensions.Http   # For HttpClient integration
dotnet add package Microsoft.Extensions.Http.Polly
```

---

### 1. Basic Circuit Breaker

```csharp
using Polly;
using Polly.CircuitBreaker;

// Circuit opens after 5 consecutive failures
// Stays open for 30 seconds, then goes HALF-OPEN
var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (exception, breakDuration) =>
        {
            Console.WriteLine($"⚡ Circuit OPENED for {breakDuration.TotalSeconds}s. Reason: {exception.Message}");
        },
        onReset: () =>
        {
            Console.WriteLine("✅ Circuit CLOSED. Service is healthy again.");
        },
        onHalfOpen: () =>
        {
            Console.WriteLine("🟡 Circuit HALF-OPEN. Sending probe request...");
        }
    );

// Usage
try
{
    var result = await circuitBreakerPolicy.ExecuteAsync(async () =>
    {
        return await _httpClient.GetStringAsync("https://api.example.com/data");
    });
}
catch (BrokenCircuitException ex)
{
    // Circuit is OPEN — fail fast, no HTTP call was made
    Console.WriteLine($"Circuit is open: {ex.Message}");
    return GetCachedOrDefaultData();
}
```

---

### 2. Advanced Circuit Breaker

The **Advanced Circuit Breaker** uses a **success/failure ratio** over a sampling period instead of just consecutive failure counts. This is more realistic for production.

```csharp
// Opens when 50% of calls in a 10-second window fail
// Requires at least 5 calls to evaluate (avoids tripping on low traffic)
// Stays open for 30 seconds
var advancedCircuitBreaker = Policy
    .Handle<HttpRequestException>()
    .Or<TimeoutException>()
    .AdvancedCircuitBreakerAsync(
        failureThreshold: 0.5,                        // 50% failure rate triggers open
        samplingDuration: TimeSpan.FromSeconds(10),   // Measure over 10-second window
        minimumThroughput: 5,                         // Need at least 5 calls to evaluate
        durationOfBreak: TimeSpan.FromSeconds(30),    // Stay open for 30 seconds
        onBreak: (ex, state, duration, context) =>
        {
            Console.WriteLine($"⚡ Circuit OPENED | State: {state} | Duration: {duration}");
        },
        onReset: (context) =>
        {
            Console.WriteLine("✅ Circuit CLOSED — service recovered");
        },
        onHalfOpen: () =>
        {
            Console.WriteLine("🟡 Circuit HALF-OPEN — testing service");
        }
    );
```

**Why Advanced > Basic:**
- Basic counts *consecutive* failures — a 50/50 success/fail pattern never triggers it
- Advanced measures failure *rate* over time — far more realistic
- Prevents tripping on low traffic (via `minimumThroughput`)

---

### 3. Combining Circuit Breaker with Retry

These two patterns complement each other perfectly:
- **Retry** handles transient glitches (network hiccup)
- **Circuit Breaker** handles prolonged outages (service is down)

```csharp
// ⚠️ ORDER MATTERS: Wrap Retry inside Circuit Breaker
// Retry is the inner policy — it retries on individual failures
// Circuit Breaker is the outer policy — it opens after too many total failures

var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)), // Exponential backoff
        onRetry: (exception, delay, attempt, context) =>
        {
            Console.WriteLine($"🔁 Retry {attempt} after {delay.TotalSeconds}s: {exception.Message}");
        }
    );

var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .AdvancedCircuitBreakerAsync(
        failureThreshold: 0.5,
        samplingDuration: TimeSpan.FromSeconds(30),
        minimumThroughput: 10,
        durationOfBreak: TimeSpan.FromSeconds(60)
    );

// Wrap: CircuitBreaker(Retry(action))
var combinedPolicy = Policy.WrapAsync(circuitBreakerPolicy, retryPolicy);

var result = await combinedPolicy.ExecuteAsync(async () =>
{
    return await _httpClient.GetStringAsync("https://api.example.com/data");
});
```

**Flow:**
```
Request
  └─► Circuit Breaker checks state
        ├─ OPEN? → Fail immediately (BrokenCircuitException)
        └─ CLOSED/HALF-OPEN? → Pass to Retry
              └─► Retry executes the action
                    ├─ Success? → Return result
                    └─ Failure? → Retry up to 3 times
                          └─ Still failing? → Circuit Breaker records failure
```

---

### 4. Combining Circuit Breaker with Fallback

A **Fallback** policy lets you return a default/cached response when the circuit is open instead of throwing an exception:

```csharp
var fallbackPolicy = Policy<string>
    .Handle<BrokenCircuitException>()
    .Or<HttpRequestException>()
    .FallbackAsync(
        fallbackValue: """{"status": "unavailable", "source": "cache"}""",
        onFallbackAsync: (exception, context) =>
        {
            Console.WriteLine($"⚠️ Falling back to cached response. Reason: {exception.Exception?.Message}");
            return Task.CompletedTask;
        }
    );

var circuitBreaker = Policy<string>
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30)
    );

// Fallback wraps CircuitBreaker
var resilientPolicy = Policy.WrapAsync(fallbackPolicy, circuitBreaker);

// Never throws — returns fallback on open circuit
var data = await resilientPolicy.ExecuteAsync(async () =>
    await _httpClient.GetStringAsync("https://api.example.com/data")
);
```

---

### 5. Using ResiliencePipeline (Polly v8+)

Polly v8 introduced a **new fluent API** with `ResiliencePipelineBuilder` — preferred for modern .NET 8+ apps:

```csharp
using Polly;
using Polly.CircuitBreaker;

// Build the pipeline
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
    {
        // Failure condition: non-success HTTP or exception
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .HandleResult(r => !r.IsSuccessStatusCode),

        FailureRatio = 0.5,                              // 50% failure rate
        SamplingDuration = TimeSpan.FromSeconds(10),
        MinimumThroughput = 5,
        BreakDuration = TimeSpan.FromSeconds(30),

        OnOpened = args =>
        {
            Console.WriteLine($"⚡ Circuit OPENED for {args.BreakDuration.TotalSeconds}s");
            return ValueTask.CompletedTask;
        },
        OnClosed = args =>
        {
            Console.WriteLine("✅ Circuit CLOSED");
            return ValueTask.CompletedTask;
        },
        OnHalfOpened = args =>
        {
            Console.WriteLine("🟡 Circuit HALF-OPEN");
            return ValueTask.CompletedTask;
        }
    })
    .AddRetry(new Polly.Retry.RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Exponential
    })
    .Build();

// Execute
try
{
    var response = await pipeline.ExecuteAsync(async ct =>
        await _httpClient.GetAsync("https://api.example.com/data", ct),
        CancellationToken.None
    );
}
catch (BrokenCircuitException)
{
    Console.WriteLine("Circuit is open — using fallback");
}
```

### Register with ASP.NET Core DI & HttpClient

```csharp
// Program.cs
builder.Services.AddHttpClient<IPaymentService, PaymentService>(client =>
{
    client.BaseAddress = new Uri("https://payment-service/");
    client.Timeout = TimeSpan.FromSeconds(5);
})
.AddResilienceHandler("payment-circuit-breaker", pipeline =>
{
    pipeline
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(10),
            MinimumThroughput = 5,
            BreakDuration = TimeSpan.FromSeconds(30)
        })
        .AddRetry(new Polly.Retry.RetryStrategyOptions<HttpResponseMessage>
        {
            MaxRetryAttempts = 3
        });
});
```

---

## Monitoring Circuit Breaker State

In production, you need to **observe** the circuit breaker state. Here's how to expose it:

```csharp
public class CircuitBreakerMonitor
{
    private readonly ICircuitBreakerPolicy _circuitBreaker;

    public CircuitBreakerMonitor(ICircuitBreakerPolicy circuitBreaker)
    {
        _circuitBreaker = circuitBreaker;
    }

    public CircuitBreakerState GetState() => _circuitBreaker.CircuitState;

    // Expose as a health check
    public HealthCheckResult CheckHealth()
    {
        return _circuitBreaker.CircuitState switch
        {
            CircuitState.Closed   => HealthCheckResult.Healthy("Circuit is closed"),
            CircuitState.HalfOpen => HealthCheckResult.Degraded("Circuit is half-open"),
            CircuitState.Open     => HealthCheckResult.Unhealthy("Circuit is open"),
            _ => HealthCheckResult.Unhealthy("Unknown state")
        };
    }
}
```

---

## Key Configuration Knobs

| Parameter | What It Controls | Typical Value |
|---|---|---|
| `exceptionsAllowedBeforeBreaking` | Consecutive failures before OPEN (basic) | 5–10 |
| `failureThreshold` / `FailureRatio` | % failure rate before OPEN (advanced) | 0.3–0.5 |
| `samplingDuration` | Time window to measure failure rate | 10–60s |
| `minimumThroughput` | Min calls before evaluating failure rate | 5–20 |
| `durationOfBreak` / `BreakDuration` | How long circuit stays OPEN | 15–60s |
| `retryCount` | Retries before counting as a failure | 2–3 |

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                   CIRCUIT BREAKER PATTERN                       │
├──────────────┬──────────────────────────────────────────────────┤
│ CLOSED       │ Normal. All calls go through. Failures counted.  │
│ OPEN         │ Fail fast. No network call. BrokenCircuitException│
│ HALF-OPEN    │ Probe allowed. Close on success, re-open on fail. │
├──────────────┴──────────────────────────────────────────────────┤
│ USE WHEN     │ Remote calls, microservices, external APIs        │
│ COMBINE WITH │ Retry (inner), Fallback (outer)                   │
│ POLLY API    │ CircuitBreakerAsync / AdvancedCircuitBreakerAsync │
│ MODERN POLLY │ ResiliencePipelineBuilder + CircuitBreakerStrategy│
└─────────────────────────────────────────────────────────────────┘
```

**Key Takeaways:**
- The Circuit Breaker **fails fast** when a service is unhealthy — protecting your system from cascading failures
- Use **Advanced Circuit Breaker** (failure ratio) in production over basic (consecutive count)
- Always combine with **Retry** (for transient faults) and **Fallback** (for graceful degradation)
- In Polly v8+, prefer `ResiliencePipelineBuilder` for a clean, composable API
- Register circuit breakers as **singletons** — they need shared state across requests