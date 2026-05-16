# Thread-Safe Singleton Design Pattern in C#

## What is the Singleton Pattern?

The **Singleton** is a creational design pattern that ensures a class has **only one instance** throughout the application's lifetime and provides a **global access point** to that instance.

---

## The Problem with Naive Singleton in Multi-threaded Environments

A basic singleton is **not thread-safe**. Two threads can simultaneously check `_instance == null`, both pass, and each create a separate instance — violating the core guarantee.

```csharp
// ❌ NOT Thread-Safe
public class Singleton
{
    private static Singleton _instance;

    private Singleton() { }

    public static Singleton Instance
    {
        get
        {
            if (_instance == null)             // Thread A and B both pass this check
                _instance = new Singleton();   // Both create separate instances!
            return _instance;
        }
    }
}
```

---

## Thread-Safe Approaches in C#

### 1. Simple Lock (Correct but Slow)

Use a `lock` to allow only one thread at a time into the critical section.

```csharp
public class Singleton
{
    private static Singleton _instance;
    private static readonly object _lock = new object();

    private Singleton() { }

    public static Singleton Instance
    {
        get
        {
            lock (_lock)                           // Only one thread enters at a time
            {
                if (_instance == null)
                    _instance = new Singleton();
                return _instance;
            }
        }
    }
}
```

**Pros:** Simple and correct.  
**Cons:** Every `Instance` call acquires a lock — even after the object is created. Major performance overhead.

---

### 2. Double-Checked Locking (Performant & Thread-Safe)

Check before and after locking to avoid unnecessary synchronization on every call.

```csharp
public class Singleton
{
    // volatile ensures the instance is fully constructed before being visible to other threads
    private static volatile Singleton _instance;
    private static readonly object _lock = new object();

    private Singleton() { }

    public static Singleton Instance
    {
        get
        {
            if (_instance == null)                  // First check (no lock)
            {
                lock (_lock)
                {
                    if (_instance == null)          // Second check (with lock)
                        _instance = new Singleton();
                }
            }
            return _instance;
        }
    }
}
```

> ⚠️ The `volatile` keyword is **critical**. Without it, the CPU or JIT compiler may reorder instructions, causing a thread to see a partially initialized object.

**Pros:** Lock is only acquired during first initialization — very fast afterward.  
**Cons:** Slightly verbose; `volatile` is easy to forget.

---

### 3. `Lazy<T>` — The Modern C# Way (Recommended ✅)

.NET provides `Lazy<T>` which handles all thread-safety and lazy initialization for you.

```csharp
public class Singleton
{
    // LazyThreadSafetyMode.ExecutionAndPublication ensures only one instance is created
    private static readonly Lazy<Singleton> _lazy =
        new Lazy<Singleton>(() => new Singleton());

    private Singleton() { }

    public static Singleton Instance => _lazy.Value;

    public void DoWork()
    {
        Console.WriteLine("Singleton is working!");
    }
}

// Usage
Singleton.Instance.DoWork();
```

> `Lazy<T>` uses **double-checked locking internally** and is the idiomatic C# approach.

**Pros:** Clean, concise, lazy, thread-safe out of the box.  
**Cons:** Minimal — `Lazy<T>` has negligible overhead.

---

### 4. Static Constructor (Eager Initialization)

C# guarantees that a **static constructor** runs exactly once, before any static member is accessed — making it inherently thread-safe.

```csharp
public class Singleton
{
    // CLR ensures this is set before any thread accesses Instance
    private static readonly Singleton _instance = new Singleton();

    // Static constructor prevents beforefieldinit optimization (ensures laziness)
    static Singleton() { }

    private Singleton() { }

    public static Singleton Instance => _instance;

    public void DoWork()
    {
        Console.WriteLine("Singleton is working!");
    }
}

// Usage
Singleton.Instance.DoWork();
```

> The empty `static Singleton() { }` is important — it removes the `beforefieldinit` flag, ensuring the field isn't initialized until `Instance` is first accessed.

**Pros:** Simple, thread-safe with zero synchronization code.  
**Cons:** Instance is created even if never used (though the static constructor makes it nearly lazy).

---

### 5. Singleton with `Lazy<T>` + Interface (Production Pattern)

A more real-world approach using dependency injection readiness and interface abstraction.

```csharp
public interface ISingleton
{
    void DoWork();
}

public class Singleton : ISingleton
{
    private static readonly Lazy<Singleton> _lazy =
        new Lazy<Singleton>(() => new Singleton());

    public static ISingleton Instance => _lazy.Value;

    private int _callCount = 0;

    private Singleton()
    {
        Console.WriteLine("Singleton created.");
    }

    public void DoWork()
    {
        _callCount++;
        Console.WriteLine($"Working... Call #{_callCount}");
    }
}

// Usage
Singleton.Instance.DoWork();   // Output: Singleton created. Working... Call #1
Singleton.Instance.DoWork();   // Output: Working... Call #2
```

---

### 6. Thread-Safe Singleton with Async Initialization

When your singleton needs async setup (e.g., loading config from a DB), use `AsyncLazy<T>`.

```csharp
public class Singleton
{
    private static readonly Lazy<Task<Singleton>> _lazy =
        new Lazy<Task<Singleton>>(async () =>
        {
            var instance = new Singleton();
            await instance.InitializeAsync();
            return instance;
        });

    private Singleton() { }

    public static Task<Singleton> Instance => _lazy.Value;

    private async Task InitializeAsync()
    {
        // Simulate async setup (e.g., DB connection, config load)
        await Task.Delay(100);
        Console.WriteLine("Async initialization complete.");
    }

    public void DoWork() => Console.WriteLine("Singleton working!");
}

// Usage
var singleton = await Singleton.Instance;
singleton.DoWork();
```

---

## Comparison Table

| Approach                    | Thread-Safe | Lazy Init | Performance | C# Idiomatic |
|-----------------------------|:-----------:|:---------:|:-----------:|:------------:|
| Naive                       | ❌          | ✅        | ✅ Fast     | ❌           |
| Simple Lock                 | ✅          | ✅        | ❌ Slow     | ⚠️ Okay      |
| Double-Checked Locking      | ✅          | ✅        | ✅ Fast     | ⚠️ Verbose   |
| `Lazy<T>`                   | ✅          | ✅        | ✅ Fast     | ✅ Yes       |
| Static Constructor          | ✅          | ⚠️ Near   | ✅ Fast     | ✅ Yes       |
| `Lazy<T>` + Interface       | ✅          | ✅        | ✅ Fast     | ✅ Best      |

---

## Real-World Example: Logger Singleton

```csharp
public class Logger
{
    private static readonly Lazy<Logger> _lazy =
        new Lazy<Logger>(() => new Logger());

    public static Logger Instance => _lazy.Value;

    private readonly string _logFile = "app.log";

    private Logger()
    {
        Console.WriteLine("[Logger] Initialized.");
    }

    public void Log(string message)
    {
        string entry = $"[{DateTime.Now:HH:mm:ss}] {message}";
        Console.WriteLine(entry);
        File.AppendAllText(_logFile, entry + Environment.NewLine);
    }
}

// Usage — from multiple threads
Parallel.For(0, 5, i =>
{
    Logger.Instance.Log($"Message from thread {i}");
});

// Output:
// [Logger] Initialized.   ← Only once!
// [10:45:01] Message from thread 0
// [10:45:01] Message from thread 2
// ...
```

---

## When to Use Singleton

✅ **Good fits:**
- Logger / Audit trail
- App configuration manager
- Database connection pool
- Cache manager
- HTTP client factory

❌ **Avoid when:**
- Writing unit tests (global state makes mocking hard — prefer DI instead)
- The object has a short or scoped lifetime
- You're already using an IoC container like `Microsoft.Extensions.DependencyInjection`

> 💡 **Tip:** In ASP.NET Core, register your class as `services.AddSingleton<T>()` and let the DI container manage the lifetime — you rarely need to hand-roll a singleton in modern .NET apps.

---

## Key Takeaways

> 1. **Prefer `Lazy<T>`** — it's the cleanest, most idiomatic C# approach.
> 2. **Use `volatile`** if rolling your own double-checked locking.
> 3. **In ASP.NET Core**, use `AddSingleton<T>()` — let the DI container handle it.
> 4. **Async singletons** are possible via `Lazy<Task<T>>`.
> 5. Singleton = one instance, globally accessible, created once — don't overuse it.