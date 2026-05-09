# CQRS, Event Sourcing & MediatR — A Complete Guide

---

## Table of Contents

1. [CQRS Pattern](#1-cqrs-pattern)
2. [Event Sourcing Pattern](#2-event-sourcing-pattern)
3. [MediatR Library](#3-mediatr-library)
4. [CQRS + Event Sourcing with MediatR](#4-cqrs--event-sourcing-with-mediatr)

---

## 1. CQRS Pattern

### What is CQRS?

**CQRS** stands for **Command Query Responsibility Segregation**.

The core idea is simple:

> **Split your application into two sides — one that WRITES data (Commands) and one that READS data (Queries). Never mix them.**

Think of a restaurant:
- The **waiter** takes your order (Command → writes to the kitchen)
- The **menu board** shows available items (Query → reads from a display)

They are completely separate responsibilities.

---

### The Problem CQRS Solves

In a traditional application, you have one model that does everything:

```csharp
// ❌ Traditional approach — one model does it all
public class OrderService
{
    public void PlaceOrder(Order order) { /* write */ }
    public Order GetOrder(int id) { /* read */ }
    public List<Order> GetAllOrders() { /* read */ }
    public void UpdateOrder(Order order) { /* write */ }
}
```

This causes problems:
- Your **read model** is cluttered with write logic
- Reads and writes have **different performance needs** (reads are often 10x more frequent)
- Adding business rules to writes breaks reads and vice versa
- **Hard to scale** reads and writes independently

---

### The CQRS Solution

Split into two distinct models:

```
┌────────────────────────────────────────┐
│              Application               │
│                                        │
│   ┌──────────────┐  ┌──────────────┐  │
│   │   COMMANDS   │  │   QUERIES    │  │
│   │  (Write Side)│  │  (Read Side) │  │
│   │              │  │              │  │
│   │ PlaceOrder   │  │ GetOrder     │  │
│   │ CancelOrder  │  │ GetOrders    │  │
│   │ UpdateOrder  │  │ SearchOrders │  │
│   └──────┬───────┘  └──────┬───────┘  │
│          │                 │          │
│     Write DB          Read DB/Cache   │
└────────────────────────────────────────┘
```

---

### Commands (Write Side)

A **Command** is an instruction to **do something**. It changes state.

Rules:
- Named as an **action**: `PlaceOrderCommand`, `CancelOrderCommand`
- Returns nothing (or just a confirmation ID)
- Contains all the data needed to perform the action

```csharp
// Command — represents an intention to change state
public class PlaceOrderCommand
{
    public Guid CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public string DeliveryAddress { get; set; }
}

// Command Handler — executes the business logic
public class PlaceOrderCommandHandler
{
    private readonly IOrderRepository _repo;

    public PlaceOrderCommandHandler(IOrderRepository repo)
    {
        _repo = repo;
    }

    public async Task<Guid> Handle(PlaceOrderCommand command)
    {
        // Business logic lives here
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = command.CustomerId,
            Items = command.Items,
            Status = OrderStatus.Pending,
            PlacedAt = DateTime.UtcNow
        };

        await _repo.SaveAsync(order);

        return order.Id; // Just returns the new ID
    }
}
```

---

### Queries (Read Side)

A **Query** is a request to **fetch data**. It never changes state.

Rules:
- Named as a **question**: `GetOrderByIdQuery`, `GetCustomerOrdersQuery`
- Always returns data
- Should be **side-effect free** (calling it 100 times has the same result)

```csharp
// Query — represents a request for data
public class GetOrderByIdQuery
{
    public Guid OrderId { get; set; }
}

// A lightweight DTO specifically for reading (not the full domain model)
public class OrderDto
{
    public Guid Id { get; set; }
    public string CustomerName { get; set; }
    public decimal TotalAmount { get; set; }
    public string Status { get; set; }
}

// Query Handler — just fetches and returns data
public class GetOrderByIdQueryHandler
{
    private readonly IReadOnlyOrderRepository _repo;

    public GetOrderByIdQueryHandler(IReadOnlyOrderRepository repo)
    {
        _repo = repo;
    }

    public async Task<OrderDto> Handle(GetOrderByIdQuery query)
    {
        // No business logic — just retrieve and map
        return await _repo.GetOrderDtoAsync(query.OrderId);
    }
}
```

---

### Key Benefits of CQRS

| Benefit | Explanation |
|---|---|
| **Scalability** | Scale reads and writes independently |
| **Performance** | Optimise read models with caching, denormalised views |
| **Clarity** | Each class has one job — easier to understand |
| **Security** | Restrict read/write access separately |
| **Flexibility** | Use different databases for reads and writes |

---

## 2. Event Sourcing Pattern

### What is Event Sourcing?

**Event Sourcing** stores every change to your data as an **immutable sequence of events**, rather than just saving the current state.

> Instead of saving "Order status = Cancelled", you save "OrderPlaced → ItemsAdded → OrderCancelled"

Think of a **bank account**:
- Traditional: stores the **balance** (`$500`)
- Event Sourcing: stores every **transaction** (`Deposit $1000 → Withdraw $300 → Withdraw $200`) → balance is derived

---

### Traditional Storage vs Event Sourcing

**Traditional approach** — only current state is saved:

```
Orders Table:
| id  | status    | total  | updated_at  |
|-----|-----------|--------|-------------|
| 001 | Cancelled | $99.00 | 2024-01-15  |
```
❌ You've lost all history. Why was it cancelled? Who cancelled it? When?

**Event Sourcing approach** — every change is an event:

```
Events Table:
| id | aggregate_id | event_type       | data                        | timestamp           |
|----|-------------|------------------|-----------------------------|---------------------|
| 1  | 001         | OrderPlaced      | { total: 99, items: [...] } | 2024-01-10 09:00:00 |
| 2  | 001         | PaymentReceived  | { amount: 99 }              | 2024-01-10 09:05:00 |
| 3  | 001         | OrderShipped     | { carrier: "FedEx" }        | 2024-01-12 10:00:00 |
| 4  | 001         | OrderCancelled   | { reason: "Lost package" }  | 2024-01-15 14:00:00 |
```
✅ Complete history. You know everything that happened.

---

### How Events Work

**Step 1: Define your events**

```csharp
// Base class for all events
public abstract class DomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
    public int Version { get; set; }
}

// Specific events — each represents ONE thing that happened
public class OrderPlacedEvent : DomainEvent
{
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public decimal Total { get; set; }
}

public class OrderShippedEvent : DomainEvent
{
    public Guid OrderId { get; set; }
    public string TrackingNumber { get; set; }
    public string Carrier { get; set; }
}

public class OrderCancelledEvent : DomainEvent
{
    public Guid OrderId { get; set; }
    public string Reason { get; set; }
    public Guid CancelledBy { get; set; }
}
```

**Step 2: Build your Aggregate (domain object that raises events)**

```csharp
public class Order
{
    // Current state — rebuilt from events
    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public decimal Total { get; private set; }
    public List<OrderItem> Items { get; private set; }

    // Uncommitted events waiting to be saved
    private readonly List<DomainEvent> _uncommittedEvents = new();
    public IReadOnlyList<DomainEvent> UncommittedEvents => _uncommittedEvents;

    public int Version { get; private set; } = 0;

    // ─── Command Methods (raise events) ───────────────────────────

    public static Order Place(Guid customerId, List<OrderItem> items)
    {
        var order = new Order();

        // Don't change state directly — raise an event instead!
        order.RaiseEvent(new OrderPlacedEvent
        {
            OrderId = Guid.NewGuid(),
            CustomerId = customerId,
            Items = items,
            Total = items.Sum(i => i.Price * i.Quantity)
        });

        return order;
    }

    public void Ship(string trackingNumber, string carrier)
    {
        if (Status != OrderStatus.Paid)
            throw new InvalidOperationException("Order must be paid before shipping.");

        RaiseEvent(new OrderShippedEvent
        {
            OrderId = Id,
            TrackingNumber = trackingNumber,
            Carrier = carrier
        });
    }

    public void Cancel(string reason, Guid cancelledBy)
    {
        if (Status == OrderStatus.Shipped)
            throw new InvalidOperationException("Cannot cancel a shipped order.");

        RaiseEvent(new OrderCancelledEvent
        {
            OrderId = Id,
            Reason = reason,
            CancelledBy = cancelledBy
        });
    }

    // ─── Apply Methods (update state from events) ─────────────────

    private void Apply(OrderPlacedEvent e)
    {
        Id = e.OrderId;
        Items = e.Items;
        Total = e.Total;
        Status = OrderStatus.Pending;
    }

    private void Apply(OrderShippedEvent e)
    {
        Status = OrderStatus.Shipped;
    }

    private void Apply(OrderCancelledEvent e)
    {
        Status = OrderStatus.Cancelled;
    }

    // ─── Infrastructure ───────────────────────────────────────────

    private void RaiseEvent(DomainEvent @event)
    {
        @event.Version = ++Version;
        _uncommittedEvents.Add(@event);
        ApplyEvent(@event); // Update current state immediately
    }

    // Replay events from storage to rebuild state
    public void LoadFromHistory(IEnumerable<DomainEvent> history)
    {
        foreach (var @event in history)
        {
            ApplyEvent(@event);
            Version = @event.Version;
        }
    }

    private void ApplyEvent(DomainEvent @event)
    {
        // Dynamic dispatch to the correct Apply() method
        ((dynamic)this).Apply((dynamic)@event);
    }
}
```

**Step 3: The Event Store — saves and loads events**

```csharp
public interface IEventStore
{
    Task SaveEventsAsync(Guid aggregateId, IEnumerable<DomainEvent> events, int expectedVersion);
    Task<List<DomainEvent>> GetEventsAsync(Guid aggregateId);
}

public class SqlEventStore : IEventStore
{
    private readonly AppDbContext _db;

    public async Task SaveEventsAsync(Guid aggregateId, IEnumerable<DomainEvent> events, int expectedVersion)
    {
        foreach (var @event in events)
        {
            var storedEvent = new StoredEvent
            {
                AggregateId = aggregateId,
                EventType = @event.GetType().Name,
                Data = JsonSerializer.Serialize(@event),  // Serialise to JSON
                Version = @event.Version,
                OccurredAt = @event.OccurredAt
            };

            _db.StoredEvents.Add(storedEvent);
        }

        await _db.SaveChangesAsync();
    }

    public async Task<List<DomainEvent>> GetEventsAsync(Guid aggregateId)
    {
        var stored = await _db.StoredEvents
            .Where(e => e.AggregateId == aggregateId)
            .OrderBy(e => e.Version)
            .ToListAsync();

        // Deserialise back to event objects
        return stored.Select(e => DeserializeEvent(e.EventType, e.Data)).ToList();
    }

    private DomainEvent DeserializeEvent(string eventType, string data)
    {
        return eventType switch
        {
            nameof(OrderPlacedEvent) => JsonSerializer.Deserialize<OrderPlacedEvent>(data),
            nameof(OrderShippedEvent) => JsonSerializer.Deserialize<OrderShippedEvent>(data),
            nameof(OrderCancelledEvent) => JsonSerializer.Deserialize<OrderCancelledEvent>(data),
            _ => throw new UnknownEventException(eventType)
        };
    }
}
```

**Step 4: Rebuilding state by replaying events**

```csharp
public class OrderRepository
{
    private readonly IEventStore _eventStore;

    public async Task<Order> GetByIdAsync(Guid orderId)
    {
        // Load all events for this order from the store
        var events = await _eventStore.GetEventsAsync(orderId);

        if (!events.Any())
            throw new OrderNotFoundException(orderId);

        // Create a blank order and replay all events to rebuild current state
        var order = new Order();
        order.LoadFromHistory(events);

        return order;
    }

    public async Task SaveAsync(Order order)
    {
        // Only save NEW events that haven't been persisted yet
        await _eventStore.SaveEventsAsync(
            order.Id,
            order.UncommittedEvents,
            order.Version - order.UncommittedEvents.Count
        );

        order.ClearUncommittedEvents();
    }
}
```

---

### Key Benefits of Event Sourcing

| Benefit | Explanation |
|---|---|
| **Full Audit Trail** | Every change is recorded — perfect for compliance |
| **Time Travel** | Rebuild state at any point in the past |
| **Debugging** | Replay events to reproduce bugs exactly |
| **Event-driven** | Events can trigger other systems (notifications, analytics) |
| **No Data Loss** | Nothing is ever deleted or overwritten |

---

## 3. MediatR Library

### What is MediatR?

**MediatR** is a .NET library that implements the **Mediator design pattern**.

> The Mediator pattern: instead of objects talking to each other directly, they all talk through a central **mediator** (middleman).

```
❌ Without MediatR — direct coupling:
Controller → OrderService → NotificationService
                         → InventoryService
                         → EmailService

✅ With MediatR — everything goes through the mediator:
Controller → Mediator → OrderCommandHandler
                     → NotificationHandler
                     → InventoryHandler
```

---

### Installing MediatR

```bash
dotnet add package MediatR
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection
```

```csharp
// Program.cs — register MediatR
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
```

---

### Core MediatR Concepts

MediatR has three main building blocks:

| Concept | Interface | Purpose |
|---|---|---|
| **Request** | `IRequest<T>` | A message (Command or Query) |
| **Handler** | `IRequestHandler<TRequest, TResponse>` | Processes the request |
| **Notification** | `INotification` | A broadcast event (multiple handlers) |

---

### MediatR in Action — Basic Example

```csharp
// 1. Define a Request (Command)
public class CreateProductCommand : IRequest<Guid>
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// 2. Define the Handler
public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, Guid>
{
    private readonly IProductRepository _repo;

    public CreateProductCommandHandler(IProductRepository repo)
    {
        _repo = repo;
    }

    public async Task<Guid> Handle(CreateProductCommand request, CancellationToken ct)
    {
        var product = new Product
        {
            Id = Guid.NewGuid(),
            Name = request.Name,
            Price = request.Price
        };

        await _repo.SaveAsync(product);
        return product.Id;
    }
}

// 3. Use it in a Controller — controller knows NOTHING about handlers
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;

    public ProductsController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateProductCommand command)
    {
        // Controller just sends the command — MediatR routes it to the right handler
        var productId = await _mediator.Send(command);
        return Ok(productId);
    }
}
```

---

### MediatR Notifications (Publish/Subscribe)

Notifications are like announcements — **multiple handlers** can react to the same event.

```csharp
// 1. Define a Notification
public class OrderPlacedNotification : INotification
{
    public Guid OrderId { get; set; }
    public string CustomerEmail { get; set; }
}

// 2. Multiple handlers can subscribe
public class SendConfirmationEmailHandler : INotificationHandler<OrderPlacedNotification>
{
    public async Task Handle(OrderPlacedNotification notification, CancellationToken ct)
    {
        // Send email to customer
        await _emailService.SendAsync(notification.CustomerEmail, "Order Confirmed!");
    }
}

public class UpdateInventoryHandler : INotificationHandler<OrderPlacedNotification>
{
    public async Task Handle(OrderPlacedNotification notification, CancellationToken ct)
    {
        // Deduct inventory
        await _inventoryService.DeductAsync(notification.OrderId);
    }
}

// 3. Publish — all handlers run
await _mediator.Publish(new OrderPlacedNotification
{
    OrderId = orderId,
    CustomerEmail = "user@example.com"
});
```

---

### MediatR Pipeline Behaviours (Middleware)

One of MediatR's killer features — add cross-cutting concerns like logging, validation, and caching without touching your handlers.

```csharp
// Logging Behaviour — logs every request and response
public class LoggingBehaviour<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly ILogger<LoggingBehaviour<TRequest, TResponse>> _logger;

    public LoggingBehaviour(ILogger<LoggingBehaviour<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        _logger.LogInformation("Handling {RequestName}: {@Request}", typeof(TRequest).Name, request);

        var response = await next(); // Call the actual handler

        _logger.LogInformation("Handled {RequestName} → {@Response}", typeof(TRequest).Name, response);

        return response;
    }
}

// Validation Behaviour — validates before handling
public class ValidationBehaviour<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehaviour(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (_validators.Any())
        {
            var context = new ValidationContext<TRequest>(request);
            var results = await Task.WhenAll(_validators.Select(v => v.ValidateAsync(context, ct)));
            var failures = results.SelectMany(r => r.Errors).Where(f => f != null).ToList();

            if (failures.Any())
                throw new ValidationException(failures); // Stop here if invalid
        }

        return await next(); // Proceed to handler if valid
    }
}

// Register behaviours — they run in order for every request
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehaviour<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehaviour<,>));
```

The pipeline looks like this:

```
Request
   │
   ▼
LoggingBehaviour (before)
   │
   ▼
ValidationBehaviour (before)
   │
   ▼
YourCommandHandler  ← actual work happens here
   │
   ▼
ValidationBehaviour (after)
   │
   ▼
LoggingBehaviour (after)
   │
   ▼
Response
```

---

## 4. CQRS + Event Sourcing with MediatR

Now let's put it all together — MediatR naturally implements CQRS by routing Commands and Queries separately, and we add Event Sourcing to record every change.

### The Full Architecture

```
HTTP Request
     │
     ▼
 Controller
     │
     │  _mediator.Send(command)
     ▼
  MediatR
  ┌─────────────────────────────────────┐
  │  Pipeline Behaviours                │
  │  (Logging → Validation → ...)       │
  └────────────────┬────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
   Command Handler       Query Handler
        │                     │
        │                     │
   Aggregate               Read Model
   (raises events)         (returns DTOs)
        │
        ▼
   Event Store
        │
        ▼
   Publish Events via MediatR
        │
   ┌────┴─────┐
   │          │
Email      Analytics
Handler    Handler
```

---

### Complete Working Example

**Define Commands, Queries, and Events with MediatR interfaces:**

```csharp
// ─── COMMANDS ────────────────────────────────────────────────────

public class PlaceOrderCommand : IRequest<Guid>
{
    public Guid CustomerId { get; set; }
    public List<OrderItemDto> Items { get; set; }
    public string DeliveryAddress { get; set; }
}

public class CancelOrderCommand : IRequest<bool>
{
    public Guid OrderId { get; set; }
    public string Reason { get; set; }
    public Guid CancelledBy { get; set; }
}

// ─── QUERIES ─────────────────────────────────────────────────────

public class GetOrderByIdQuery : IRequest<OrderDetailDto>
{
    public Guid OrderId { get; set; }
}

public class GetCustomerOrdersQuery : IRequest<List<OrderSummaryDto>>
{
    public Guid CustomerId { get; set; }
}
```

---

**Command Handler — uses Event Sourcing to persist changes:**

```csharp
public class PlaceOrderCommandHandler : IRequestHandler<PlaceOrderCommand, Guid>
{
    private readonly IEventStore _eventStore;
    private readonly IMediator _mediator;

    public PlaceOrderCommandHandler(IEventStore eventStore, IMediator mediator)
    {
        _eventStore = eventStore;
        _mediator = mediator;
    }

    public async Task<Guid> Handle(PlaceOrderCommand request, CancellationToken ct)
    {
        // 1. Create the aggregate — this raises the OrderPlacedEvent internally
        var items = request.Items.Select(i => new OrderItem(i.ProductId, i.Quantity, i.Price)).ToList();
        var order = Order.Place(request.CustomerId, items);

        // 2. Save the events to the Event Store
        await _eventStore.SaveEventsAsync(order.Id, order.UncommittedEvents, 0);

        // 3. Publish events via MediatR so other systems react
        foreach (var domainEvent in order.UncommittedEvents)
        {
            // Convert domain events to MediatR notifications
            if (domainEvent is OrderPlacedEvent placed)
            {
                await _mediator.Publish(new OrderPlacedNotification
                {
                    OrderId = placed.OrderId,
                    CustomerId = placed.CustomerId,
                    Total = placed.Total
                }, ct);
            }
        }

        order.ClearUncommittedEvents();
        return order.Id;
    }
}
```

---

**Cancel Order Command Handler:**

```csharp
public class CancelOrderCommandHandler : IRequestHandler<CancelOrderCommand, bool>
{
    private readonly IEventStore _eventStore;
    private readonly IMediator _mediator;

    public CancelOrderCommandHandler(IEventStore eventStore, IMediator mediator)
    {
        _eventStore = eventStore;
        _mediator = mediator;
    }

    public async Task<bool> Handle(CancelOrderCommand request, CancellationToken ct)
    {
        // 1. Rebuild the order from its event history
        var events = await _eventStore.GetEventsAsync(request.OrderId);
        var order = new Order();
        order.LoadFromHistory(events);

        // 2. Apply business logic — raises OrderCancelledEvent internally
        order.Cancel(request.Reason, request.CancelledBy);

        // 3. Save only the NEW events
        await _eventStore.SaveEventsAsync(order.Id, order.UncommittedEvents, order.Version);

        // 4. Notify other parts of the system
        await _mediator.Publish(new OrderCancelledNotification
        {
            OrderId = request.OrderId,
            Reason = request.Reason
        }, ct);

        order.ClearUncommittedEvents();
        return true;
    }
}
```

---

**Query Handler — reads from a denormalised Read Model (not the event store):**

```csharp
// Separate read model — optimised for fast reads
public class OrderDetailDto
{
    public Guid Id { get; set; }
    public string CustomerName { get; set; }
    public string Status { get; set; }
    public decimal Total { get; set; }
    public List<OrderItemDto> Items { get; set; }
    public DateTime PlacedAt { get; set; }
    public string? TrackingNumber { get; set; }
}

public class GetOrderByIdQueryHandler : IRequestHandler<GetOrderByIdQuery, OrderDetailDto>
{
    private readonly IReadDbContext _readDb; // Separate read database/view

    public GetOrderByIdQueryHandler(IReadDbContext readDb)
    {
        _readDb = readDb;
    }

    public async Task<OrderDetailDto> Handle(GetOrderByIdQuery request, CancellationToken ct)
    {
        // Simple, fast read — no business logic, no event replaying
        return await _readDb.OrderViews
            .Where(o => o.Id == request.OrderId)
            .Select(o => new OrderDetailDto
            {
                Id = o.Id,
                CustomerName = o.CustomerName,
                Status = o.Status,
                Total = o.Total,
                Items = o.Items,
                PlacedAt = o.PlacedAt
            })
            .FirstOrDefaultAsync(ct);
    }
}
```

---

**Projections — keep the Read Model in sync with events:**

When a Command saves events, we also update the Read Model via MediatR Notifications:

```csharp
// This handler updates the read model whenever an order is placed
public class OrderPlacedReadModelProjection : INotificationHandler<OrderPlacedNotification>
{
    private readonly IReadDbContext _readDb;

    public OrderPlacedReadModelProjection(IReadDbContext readDb)
    {
        _readDb = readDb;
    }

    public async Task Handle(OrderPlacedNotification notification, CancellationToken ct)
    {
        // Write a denormalised, query-friendly view of the order
        var orderView = new OrderView
        {
            Id = notification.OrderId,
            CustomerId = notification.CustomerId,
            Status = "Pending",
            Total = notification.Total,
            PlacedAt = DateTime.UtcNow
        };

        _readDb.OrderViews.Add(orderView);
        await _readDb.SaveChangesAsync(ct);
    }
}

// Update read model when cancelled
public class OrderCancelledReadModelProjection : INotificationHandler<OrderCancelledNotification>
{
    private readonly IReadDbContext _readDb;

    public async Task Handle(OrderCancelledNotification notification, CancellationToken ct)
    {
        var view = await _readDb.OrderViews.FindAsync(notification.OrderId);
        if (view is not null)
        {
            view.Status = "Cancelled";
            await _readDb.SaveChangesAsync(ct);
        }
    }
}
```

---

**The Controller — stays thin, knows nothing about implementations:**

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;

    public OrdersController(IMediator mediator)
    {
        _mediator = mediator;
    }

    // COMMAND — Place an order
    [HttpPost]
    public async Task<IActionResult> PlaceOrder([FromBody] PlaceOrderCommand command)
    {
        var orderId = await _mediator.Send(command);
        return CreatedAtAction(nameof(GetOrder), new { id = orderId }, new { orderId });
    }

    // COMMAND — Cancel an order
    [HttpDelete("{id}")]
    public async Task<IActionResult> CancelOrder(Guid id, [FromBody] CancelOrderRequest req)
    {
        await _mediator.Send(new CancelOrderCommand
        {
            OrderId = id,
            Reason = req.Reason,
            CancelledBy = req.UserId
        });
        return NoContent();
    }

    // QUERY — Get a single order
    [HttpGet("{id}")]
    public async Task<IActionResult> GetOrder(Guid id)
    {
        var order = await _mediator.Send(new GetOrderByIdQuery { OrderId = id });
        return order is null ? NotFound() : Ok(order);
    }

    // QUERY — Get all orders for a customer
    [HttpGet("customer/{customerId}")]
    public async Task<IActionResult> GetCustomerOrders(Guid customerId)
    {
        var orders = await _mediator.Send(new GetCustomerOrdersQuery { CustomerId = customerId });
        return Ok(orders);
    }
}
```

---

### Event Time Travel — Rebuild Past State

One of Event Sourcing's superpowers — see what an order looked like at any point in time:

```csharp
public class GetOrderAtPointInTimeQuery : IRequest<OrderDetailDto>
{
    public Guid OrderId { get; set; }
    public DateTime AsOf { get; set; } // What did the order look like at this time?
}

public class GetOrderAtPointInTimeQueryHandler
    : IRequestHandler<GetOrderAtPointInTimeQuery, OrderDetailDto>
{
    private readonly IEventStore _eventStore;

    public async Task<OrderDetailDto> Handle(GetOrderAtPointInTimeQuery request, CancellationToken ct)
    {
        // Get only events that occurred BEFORE the requested time
        var events = await _eventStore.GetEventsAsync(request.OrderId);
        var relevantEvents = events.Where(e => e.OccurredAt <= request.AsOf);

        // Replay only those events
        var order = new Order();
        order.LoadFromHistory(relevantEvents);

        return new OrderDetailDto
        {
            Id = order.Id,
            Status = order.Status.ToString(),
            Total = order.Total
        };
    }
}
```

---

### Full Data Flow Summary

```
POST /api/orders
       │
       ▼
OrdersController.PlaceOrder()
       │
       │  _mediator.Send(new PlaceOrderCommand {...})
       ▼
  MediatR Pipeline
  ┌──────────────────────────┐
  │ 1. LoggingBehaviour      │
  │ 2. ValidationBehaviour   │
  │ 3. PlaceOrderCommandHandler │
  │    ├── Order.Place()     │  ← Domain logic, raises OrderPlacedEvent
  │    ├── EventStore.Save() │  ← Persist events
  │    └── mediator.Publish()│  ← Notify other systems
  └──────────────────────────┘
       │
       ├──► OrderPlacedReadModelProjection  → Updates Read DB
       ├──► SendConfirmationEmailHandler    → Sends email
       └──► UpdateInventoryHandler          → Deducts stock
```

---

## Quick Reference Cheat Sheet

| Concept | What it does | MediatR Interface |
|---|---|---|
| **Command** | Changes state, no return (or ID) | `IRequest<T>` |
| **Query** | Reads state, returns data | `IRequest<T>` |
| **Event** | Records what happened | `INotification` |
| **Command Handler** | Executes the command | `IRequestHandler<,>` |
| **Query Handler** | Returns the data | `IRequestHandler<,>` |
| **Event Handler** | Reacts to what happened | `INotificationHandler<>` |
| **Pipeline Behaviour** | Cross-cutting (logging, validation) | `IPipelineBehavior<,>` |

---

## When to Use These Patterns?

**Use CQRS when:**
- Your reads and writes have very different loads (reads >> writes)
- You need different models for reading and writing
- Your domain has complex business rules

**Use Event Sourcing when:**
- You need a full audit trail (finance, healthcare, compliance)
- You need to debug by replaying history
- Your business logic is inherently event-driven

**Use MediatR when:**
- You want to decouple your controllers from business logic
- You need a clean pipeline for cross-cutting concerns
- You're implementing CQRS and want a clean handler structure

> 💡 **Start with MediatR + CQRS** for most applications. Add **Event Sourcing** only when you truly need the audit trail or time-travel capabilities — it adds meaningful complexity.