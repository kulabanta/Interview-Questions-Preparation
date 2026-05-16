# Producer-Consumer Design Pattern in C#

## What is the Producer-Consumer Pattern?

The **Producer-Consumer** pattern is a classic concurrency design pattern where:

- **Producers** generate data / work items and place them into a shared buffer
- **Consumers** take items from the buffer and process them
- A **shared queue (buffer)** decouples producers from consumers so they operate independently

This decoupling means producers don't wait for consumers to finish, and consumers don't wait for producers to generate — they run concurrently.

```
 ┌──────────┐        ┌─────────────────┐        ┌──────────┐
 │ Producer │──────► │  Shared Buffer  │──────► │ Consumer │
 │  (Task)  │        │  (BlockingCol.) │        │  (Task)  │
 └──────────┘        └─────────────────┘        └──────────┘
 ┌──────────┐                                    ┌──────────┐
 │ Producer │──────────────────────────────────► │ Consumer │
 └──────────┘                                    └──────────┘
```

---

## Why Use It?

| Problem                          | Solution via Producer-Consumer        |
|----------------------------------|---------------------------------------|
| Producer is faster than consumer | Buffer absorbs the backlog            |
| Consumer is faster than producer | Consumer waits without busy-polling   |
| Tight coupling between threads   | Queue decouples them cleanly          |
| Thread coordination complexity   | Pattern handles sync automatically    |

---

## Approach 1: `BlockingCollection<T>` — The Standard C# Way ✅

`BlockingCollection<T>` is the go-to class in .NET for this pattern. It handles all thread-safety and blocking automatically.

```csharp
using System;
using System.Collections.Concurrent;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    // Bounded capacity = max 5 items in the buffer at once
    static BlockingCollection<int> _queue = new BlockingCollection<int>(boundedCapacity: 5);

    static void Main()
    {
        Task producer = Task.Run(() => Produce());
        Task consumer = Task.Run(() => Consume());

        Task.WaitAll(producer, consumer);
        Console.WriteLine("Done.");
    }

    static void Produce()
    {
        for (int i = 1; i <= 10; i++)
        {
            _queue.Add(i);                              // Blocks if queue is full
            Console.WriteLine($"[Producer] Produced: {i}");
            Thread.Sleep(100);                          // Simulate work
        }
        _queue.CompleteAdding();                        // Signal: no more items
    }

    static void Consume()
    {
        // GetConsumingEnumerable() blocks until an item is available
        // and exits automatically when CompleteAdding() is called
        foreach (var item in _queue.GetConsumingEnumerable())
        {
            Console.WriteLine($"  [Consumer] Consumed: {item}");
            Thread.Sleep(200);                          // Simulate slower processing
        }
    }
}
```

**Sample Output:**
```
[Producer] Produced: 1
  [Consumer] Consumed: 1
[Producer] Produced: 2
[Producer] Produced: 3
  [Consumer] Consumed: 2
...
Done.
```

---

## Approach 2: Multiple Producers & Multiple Consumers

Real-world systems often have many producers and many consumers running in parallel.

```csharp
using System;
using System.Collections.Concurrent;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static BlockingCollection<string> _queue = new BlockingCollection<string>(boundedCapacity: 10);

    static void Main()
    {
        int producerCount = 3;
        int consumerCount = 2;

        // Start multiple producers
        var producers = Enumerable.Range(1, producerCount)
            .Select(id => Task.Run(() => Produce(id)))
            .ToArray();

        // Start multiple consumers
        var consumers = Enumerable.Range(1, consumerCount)
            .Select(id => Task.Run(() => Consume(id)))
            .ToArray();

        // Wait for all producers to finish, then signal completion
        Task.WaitAll(producers);
        _queue.CompleteAdding();

        // Wait for consumers to drain the queue
        Task.WaitAll(consumers);
        Console.WriteLine("All done.");
    }

    static void Produce(int producerId)
    {
        for (int i = 1; i <= 5; i++)
        {
            string item = $"P{producerId}-Item{i}";
            _queue.Add(item);
            Console.WriteLine($"[Producer {producerId}] Produced: {item}");
            Thread.Sleep(new Random().Next(50, 150));
        }
    }

    static void Consume(int consumerId)
    {
        foreach (var item in _queue.GetConsumingEnumerable())
        {
            Console.WriteLine($"  [Consumer {consumerId}] Consumed: {item}");
            Thread.Sleep(new Random().Next(100, 300));
        }
    }
}
```

---

## Approach 3: Using `Channel<T>` — The Modern Async Way (Recommended for .NET 5+) ⭐

`Channel<T>` (from `System.Threading.Channels`) is the **async-first**, high-performance alternative to `BlockingCollection<T>`.

```csharp
using System;
using System.Threading.Channels;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        // Bounded channel — max 5 items buffered
        var channel = Channel.CreateBounded<int>(capacity: 5);

        Task producer = ProduceAsync(channel.Writer);
        Task consumer = ConsumeAsync(channel.Reader);

        await Task.WhenAll(producer, consumer);
        Console.WriteLine("Done.");
    }

    static async Task ProduceAsync(ChannelWriter<int> writer)
    {
        for (int i = 1; i <= 10; i++)
        {
            await writer.WriteAsync(i);              // Async: doesn't block a thread
            Console.WriteLine($"[Producer] Produced: {i}");
            await Task.Delay(100);
        }
        writer.Complete();                           // Signal no more items
    }

    static async Task ConsumeAsync(ChannelReader<int> reader)
    {
        // ReadAllAsync() awaits items as they arrive
        await foreach (var item in reader.ReadAllAsync())
        {
            Console.WriteLine($"  [Consumer] Consumed: {item}");
            await Task.Delay(200);
        }
    }
}
```

### Why prefer `Channel<T>` over `BlockingCollection<T>`?

| Feature                   | `BlockingCollection<T>` | `Channel<T>`        |
|---------------------------|-------------------------|---------------------|
| Async support             | ❌ (blocks threads)     | ✅ (true async/await)|
| Performance               | Good                    | Excellent            |
| Backpressure              | ✅ Bounded              | ✅ Bounded           |
| `IAsyncEnumerable` support| ❌                      | ✅                   |
| .NET version              | .NET 4+                 | .NET Core 2.1+       |

---

## Approach 4: Manual Implementation with `Monitor` / `lock`

Understanding the low-level mechanism is critical for interviews.

```csharp
using System;
using System.Collections.Generic;
using System.Threading;

class ProducerConsumer
{
    private readonly Queue<int> _buffer = new Queue<int>();
    private readonly int _maxSize = 5;
    private readonly object _lock = new object();

    public void Produce(int item)
    {
        lock (_lock)
        {
            // Wait while the buffer is full
            while (_buffer.Count >= _maxSize)
            {
                Console.WriteLine("[Producer] Buffer full, waiting...");
                Monitor.Wait(_lock);              // Releases lock and waits
            }

            _buffer.Enqueue(item);
            Console.WriteLine($"[Producer] Produced: {item} | Buffer size: {_buffer.Count}");
            Monitor.PulseAll(_lock);              // Notify waiting consumers
        }
    }

    public int Consume()
    {
        lock (_lock)
        {
            // Wait while the buffer is empty
            while (_buffer.Count == 0)
            {
                Console.WriteLine("[Consumer] Buffer empty, waiting...");
                Monitor.Wait(_lock);              // Releases lock and waits
            }

            int item = _buffer.Dequeue();
            Console.WriteLine($"  [Consumer] Consumed: {item} | Buffer size: {_buffer.Count}");
            Monitor.PulseAll(_lock);              // Notify waiting producers
            return item;
        }
    }
}

class Program
{
    static void Main()
    {
        var pc = new ProducerConsumer();

        Thread producer = new Thread(() =>
        {
            for (int i = 1; i <= 10; i++)
            {
                pc.Produce(i);
                Thread.Sleep(100);
            }
        });

        Thread consumer = new Thread(() =>
        {
            for (int i = 0; i < 10; i++)
            {
                pc.Consume();
                Thread.Sleep(200);
            }
        });

        producer.Start();
        consumer.Start();

        producer.Join();
        consumer.Join();
    }
}
```

> Key methods:
> - `Monitor.Wait(lock)` — releases the lock and suspends the current thread
> - `Monitor.Pulse(lock)` — wakes one waiting thread
> - `Monitor.PulseAll(lock)` — wakes all waiting threads

---

## Approach 5: Using `SemaphoreSlim` for Async Signaling

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static Queue<int> _buffer = new Queue<int>();
    static SemaphoreSlim _itemAvailable = new SemaphoreSlim(0);  // starts empty
    static SemaphoreSlim _spaceAvailable = new SemaphoreSlim(5); // capacity = 5
    static object _lock = new object();

    static async Task Main()
    {
        Task producer = Task.Run(async () =>
        {
            for (int i = 1; i <= 10; i++)
            {
                await _spaceAvailable.WaitAsync();     // Wait for space
                lock (_lock) { _buffer.Enqueue(i); }
                Console.WriteLine($"[Producer] Produced: {i}");
                _itemAvailable.Release();              // Signal item available
                await Task.Delay(100);
            }
        });

        Task consumer = Task.Run(async () =>
        {
            for (int i = 0; i < 10; i++)
            {
                await _itemAvailable.WaitAsync();      // Wait for item
                int item;
                lock (_lock) { item = _buffer.Dequeue(); }
                Console.WriteLine($"  [Consumer] Consumed: {item}");
                _spaceAvailable.Release();             // Signal space available
                await Task.Delay(200);
            }
        });

        await Task.WhenAll(producer, consumer);
    }
}
```

---

## Real-World Use Case: Order Processing System

```csharp
using System;
using System.Threading.Channels;
using System.Threading.Tasks;

record Order(int Id, string Product, int Quantity);

class OrderProcessingSystem
{
    private readonly Channel<Order> _orderChannel =
        Channel.CreateBounded<Order>(capacity: 20);

    // Producer: Receives orders from users
    public async Task AcceptOrdersAsync()
    {
        var orders = new[]
        {
            new Order(1, "Laptop", 2),
            new Order(2, "Mouse", 5),
            new Order(3, "Keyboard", 3),
        };

        foreach (var order in orders)
        {
            await _orderChannel.Writer.WriteAsync(order);
            Console.WriteLine($"[Order Received] #{order.Id} - {order.Product} x{order.Quantity}");
            await Task.Delay(50);
        }

        _orderChannel.Writer.Complete();
    }

    // Consumer: Processes orders from the queue
    public async Task ProcessOrdersAsync(int workerId)
    {
        await foreach (var order in _orderChannel.Reader.ReadAllAsync())
        {
            Console.WriteLine($"  [Worker {workerId}] Processing Order #{order.Id}...");
            await Task.Delay(300);  // Simulate processing time
            Console.WriteLine($"  [Worker {workerId}] Completed Order #{order.Id}");
        }
    }

    public async Task RunAsync()
    {
        await Task.WhenAll(
            AcceptOrdersAsync(),
            ProcessOrdersAsync(1),
            ProcessOrdersAsync(2)   // Two workers consuming
        );
    }
}

class Program
{
    static async Task Main() => await new OrderProcessingSystem().RunAsync();
}
```

---

## Key Concepts to Know

### Bounded vs Unbounded Buffer

```csharp
// Bounded — blocks/waits when full (prevents memory overflow)
var bounded = new BlockingCollection<int>(boundedCapacity: 10);

// Unbounded — never blocks producer (risk: memory grows indefinitely)
var unbounded = new BlockingCollection<int>();

// Same with Channel<T>
var boundedChannel   = Channel.CreateBounded<int>(10);
var unboundedChannel = Channel.CreateUnbounded<int>();
```

### Cancellation Support

```csharp
var cts = new CancellationTokenSource();
var queue = new BlockingCollection<int>(10);

Task.Run(() =>
{
    try
    {
        foreach (var item in queue.GetConsumingEnumerable(cts.Token))
            Console.WriteLine($"Consumed: {item}");
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Consumer cancelled.");
    }
});

// Cancel after 2 seconds
await Task.Delay(2000);
cts.Cancel();
```

---

## Interview: How to Implement It Step-by-Step

When asked in an interview, walk through these steps clearly:

```
Step 1 — Identify the shared buffer
         → Use BlockingCollection<T> or Channel<T>

Step 2 — Define the Producer
         → Loops, creates items, calls Add() or WriteAsync()
         → Calls CompleteAdding() or writer.Complete() when done

Step 3 — Define the Consumer
         → Loops using GetConsumingEnumerable() or ReadAllAsync()
         → Processes items until the queue is marked complete

Step 4 — Run them concurrently
         → Use Task.Run() + Task.WhenAll() or parallel threads

Step 5 — Handle edge cases
         → Cancellation (CancellationToken)
         → Bounded capacity (backpressure)
         → Multiple producers/consumers
         → Graceful shutdown
```

### Interview Code Template (Memorize This)

```csharp
// Minimal correct implementation — explain this confidently
var queue = new BlockingCollection<int>(boundedCapacity: 10);

Task producer = Task.Run(() =>
{
    for (int i = 0; i < 20; i++)
    {
        queue.Add(i);
        Console.WriteLine($"Produced: {i}");
    }
    queue.CompleteAdding();  // ← Don't forget this!
});

Task consumer = Task.Run(() =>
{
    foreach (var item in queue.GetConsumingEnumerable())
        Console.WriteLine($"  Consumed: {item}");
});

await Task.WhenAll(producer, consumer);
```

---

## Interview Questions & Answers

### ❓ Q1. What is the Producer-Consumer pattern and why is it used?

**Answer:**
It's a concurrency pattern where producers generate data and place it in a shared buffer, while consumers retrieve and process it. It decouples the production and consumption rates, prevents resource starvation, and allows both sides to work at their own pace without tight coupling.

---

### ❓ Q2. What happens if the producer is much faster than the consumer?

**Answer:**
Without a bounded buffer, the queue grows indefinitely, leading to high memory usage or `OutOfMemoryException`. With a **bounded** buffer (e.g., `BlockingCollection<T>(capacity: N)`), the producer is automatically **blocked** when the buffer is full, applying natural **backpressure** — slowing the producer to match the consumer's pace.

---

### ❓ Q3. How does `BlockingCollection<T>` work internally?

**Answer:**
It wraps a `ConcurrentQueue<T>` (or any `IProducerConsumerCollection<T>`) and adds blocking semantics. `Add()` blocks if the bounded capacity is reached. `Take()` / `GetConsumingEnumerable()` blocks when the collection is empty. Internally, it uses `SemaphoreSlim` for signaling between threads, making it safe without manual locking.

---

### ❓ Q4. What is `CompleteAdding()` and what happens if you forget it?

**Answer:**
`CompleteAdding()` marks the collection as complete — no more items will be added. `GetConsumingEnumerable()` monitors this flag and exits the `foreach` loop once the queue is empty AND `CompleteAdding()` has been called. **If you forget it**, the consumer loop blocks forever, waiting for items that will never come — causing a **deadlock**.

---

### ❓ Q5. What's the difference between `Monitor.Pulse()` and `Monitor.PulseAll()`?

**Answer:**
- `Monitor.Pulse()` wakes **one** thread waiting on the lock (chosen by the runtime)
- `Monitor.PulseAll()` wakes **all** waiting threads — each then re-acquires the lock and re-checks their condition

Use `PulseAll()` when multiple threads may be waiting and any of them could proceed. Use `Pulse()` when exactly one thread can make progress (more efficient but less safe if logic is complex).

---

### ❓ Q6. When would you use `Channel<T>` over `BlockingCollection<T>`?

**Answer:**
Use `Channel<T>` when:
- You're writing **async/await** code and don't want to block threads
- You need **high throughput** (Channel is significantly faster)
- You want `IAsyncEnumerable` support for consumers
- You're on .NET Core / .NET 5+

Use `BlockingCollection<T>` when:
- Working with synchronous code / legacy systems
- You need simpler, more familiar semantics
- Thread-blocking behavior is acceptable

---

### ❓ Q7. How would you gracefully shut down the consumer?

**Answer:**
Two clean approaches:
1. **`CompleteAdding()`** on `BlockingCollection<T>` — consumers drain remaining items then exit
2. **`CancellationToken`** — pass a `CancellationTokenSource.Token` to `GetConsumingEnumerable(token)` or `ReadAllAsync(token)` and call `cts.Cancel()` when you want to stop

---

### ❓ Q8. How do you handle exceptions in a producer or consumer without crashing the whole pipeline?

**Answer:**
Wrap each producer/consumer body in a `try/catch`. For `Task`-based implementations, collect all tasks and use `Task.WhenAll()` — exceptions aggregate into `AggregateException`. Alternatively, use a dedicated error channel or logging to capture failures without stopping other workers.

```csharp
try
{
    await Task.WhenAll(producer, consumer);
}
catch (AggregateException ex)
{
    foreach (var inner in ex.InnerExceptions)
        Console.WriteLine($"Error: {inner.Message}");
}
```

---

### ❓ Q9. What is backpressure and how does this pattern support it?

**Answer:**
Backpressure is a mechanism to **slow down a fast producer** when the consumer can't keep up. In this pattern, it's achieved with a **bounded buffer**: when the buffer is full, the producer blocks (sync) or awaits (async) until space is available. This prevents unbounded memory growth and naturally throttles throughput.

---

### ❓ Q10. Can you implement Producer-Consumer without any .NET concurrency primitives?

**Answer:**
Yes — using `lock`, `Monitor.Wait()`, and `Monitor.PulseAll()` on a plain `Queue<T>`:
- Producer: lock → check if full → if full `Monitor.Wait()` → Enqueue → `Monitor.PulseAll()`
- Consumer: lock → check if empty → if empty `Monitor.Wait()` → Dequeue → `Monitor.PulseAll()`

This is the raw low-level implementation and demonstrates understanding of the underlying synchronization model.

---

## Summary Cheat Sheet

```
Pattern:     Producer → [Shared Buffer] → Consumer

.NET Tools:
  ├── BlockingCollection<T>    → Sync, classic, easy
  ├── Channel<T>               → Async, modern, fast ⭐
  ├── Monitor + lock           → Manual, low-level (interview)
  └── SemaphoreSlim            → Async signaling

Key Concepts:
  ├── Bounded buffer           → Backpressure, prevents OOM
  ├── CompleteAdding()         → Signals end of production
  ├── GetConsumingEnumerable() → Blocking consumer loop
  ├── ReadAllAsync()           → Async consumer loop
  └── CancellationToken        → Graceful shutdown

Interview Tips:
  ├── Always use bounded capacity
  ├── Always call CompleteAdding() / writer.Complete()
  ├── Handle cancellation
  └── Prefer Channel<T> in async code
```