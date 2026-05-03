# Unit of Work Design Pattern in C#
### A Complete Reference for .NET Developers

---

## Table of Contents

1. [What Problem Does Unit of Work Solve?](#1-what-problem-does-unit-of-work-solve)
2. [Core Concept & Definition](#2-core-concept--definition)
3. [UoW + Repository: How They Work Together](#3-uow--repository-how-they-work-together)
4. [Implementation from Scratch (Without EF Core)](#4-implementation-from-scratch-without-ef-core)
5. [Implementation With EF Core](#5-implementation-with-ef-core)
6. [UoW With Transactions (Advanced)](#6-uow-with-transactions-advanced)
7. [Registering With Dependency Injection](#7-registering-with-dependency-injection)
8. [Unit Testing the Unit of Work](#8-unit-testing-the-unit-of-work)
9. [When NOT to Use Unit of Work](#9-when-not-to-use-unit-of-work)
10. [Common Pitfalls & Anti-Patterns](#10-common-pitfalls--anti-patterns)
11. [Interview Questions (4 YOE Level)](#11-interview-questions-4-yoe-level)

---

## 1. What Problem Does Unit of Work Solve?

Imagine a simple e-commerce order placement flow. You need to:

1. Deduct stock from `Inventory`
2. Create a new `Order`
3. Create an `OrderLineItem`
4. Deduct funds from the customer's `Wallet`

**Without Unit of Work**, each repository calls `SaveChanges()` independently:

```csharp
// DANGEROUS — four separate save points, four chances to fail mid-way
await _inventoryRepo.DeductStockAsync(productId, qty);  // SaveChanges() ← committed
await _orderRepo.CreateAsync(order);                    // SaveChanges() ← committed
await _lineItemRepo.AddAsync(lineItem);                 // SaveChanges() ← FAILS here
await _walletRepo.DeductAsync(customerId, amount);      // never reached
```

If step 3 fails, the database is now **partially updated** — inventory is deducted and an order exists, but no line items and no wallet deduction. You have corrupted data with no easy way to roll back.

**With Unit of Work**, all changes are collected and committed as a single atomic operation:

```csharp
// ALL-or-NOTHING — one commit at the end
await _inventoryRepo.DeductStockAsync(productId, qty);  // tracked, not saved
await _orderRepo.CreateAsync(order);                    // tracked, not saved
await _lineItemRepo.AddAsync(lineItem);                 // tracked, not saved
await _walletRepo.DeductAsync(customerId, amount);      // tracked, not saved

await _unitOfWork.CommitAsync(); // ONE atomic commit — succeeds or rolls back entirely
```

---

## 2. Core Concept & Definition

> **Unit of Work** maintains a list of objects affected by a business transaction and coordinates writing out changes and resolving concurrency problems.
> — *Martin Fowler, Patterns of Enterprise Application Architecture*

The pattern does three things:

| Responsibility | What it means |
|---|---|
| **Track changes** | Knows which objects were inserted, updated, or deleted during a business operation |
| **Coordinate writes** | Flushes all tracked changes to the database in a single round-trip |
| **Manage transactions** | Ensures all-or-nothing semantics — either everything succeeds or nothing is persisted |

### The Mental Model

Think of Unit of Work like a **shopping cart at a supermarket**:

- You walk around picking up items (tracking changes across multiple repositories)
- You don't pay for each item individually at separate tills (no individual `SaveChanges()` calls)
- You bring everything to one checkout at the end (single `CommitAsync()`)
- If your card declines, nothing is taken — atomicity guaranteed

```
Business Operation (e.g. Place Order)
│
├── InventoryRepository.DeductStock()  ──┐
├── OrderRepository.Create()           ──┤─→ UnitOfWork (change tracker)
├── LineItemRepository.Add()           ──┤         │
└── WalletRepository.Deduct()          ──┘         │
                                                    ▼
                                             CommitAsync()
                                                    │
                                         ┌──────────┴──────────┐
                                         │    Single DB         │
                                         │    Transaction       │
                                         │    BEGIN → COMMIT    │
                                         └─────────────────────┘
```

---

## 3. UoW + Repository: How They Work Together

Unit of Work and Repository are almost always used together. They have distinct, complementary roles:

| Concern | Repository | Unit of Work |
|---|---|---|
| **Purpose** | Abstract data access for a single aggregate/entity | Coordinate multiple repositories in one transaction |
| **Knows about** | One entity type (e.g. `Order`) | All repositories in a business operation |
| **Saves data?** | No — adds/updates/removes from the tracked set | Yes — the only one that calls `SaveChanges()` |
| **Scope** | Entity-level | Business-operation-level |

### The Classic Relationship

```csharp
// Repositories live inside the Unit of Work
public interface IUnitOfWork : IDisposable
{
    IOrderRepository     Orders     { get; }
    IInventoryRepository Inventory  { get; }
    IWalletRepository    Wallets    { get; }

    Task<int> CommitAsync(CancellationToken ct = default);
}
```

Consumers access repositories through the UoW — they never need to inject repositories directly.

---

## 4. Implementation from Scratch (Without EF Core)

Understanding a manual implementation builds intuition for what EF Core does automatically.

### Step 1: Generic Repository Interface

```csharp
public interface IRepository<T> where T : class
{
    Task<T?>              GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    void Update(T entity);
    void Remove(T entity);
}
```

### Step 2: Specific Repository Interfaces

```csharp
public interface IOrderRepository : IRepository<Order>
{
    Task<IEnumerable<Order>> GetOrdersByCustomerAsync(int customerId);
    Task<Order?>             GetWithLineItemsAsync(int orderId);
}

public interface IInventoryRepository : IRepository<InventoryItem>
{
    Task<InventoryItem?> GetByProductIdAsync(int productId);
    Task DeductStockAsync(int productId, int quantity);
}
```

### Step 3: The Unit of Work Interface

```csharp
public interface IUnitOfWork : IDisposable
{
    IOrderRepository     Orders    { get; }
    IInventoryRepository Inventory { get; }

    /// <summary>
    /// Persists ALL tracked changes as a single atomic database transaction.
    /// Returns the number of state entries written to the database.
    /// </summary>
    Task<int> CommitAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Discards all tracked changes without persisting them.
    /// </summary>
    void Rollback();
}
```

### Step 4: In-Memory Change Tracker (for illustration)

```csharp
// Simplified in-memory UoW to illustrate the concept
// (In real apps, use EF Core's DbContext — see Section 5)

public class InMemoryUnitOfWork : IUnitOfWork
{
    private readonly List<object> _newEntities     = new();
    private readonly List<object> _dirtyEntities   = new();
    private readonly List<object> _removedEntities = new();

    private IOrderRepository?     _orders;
    private IInventoryRepository? _inventory;

    public IOrderRepository     Orders    => _orders    ??= new InMemoryOrderRepository(this);
    public IInventoryRepository Inventory => _inventory ??= new InMemoryInventoryRepository(this);

    // Called by repositories to register work — NOT by consumers
    internal void RegisterNew(object entity)     => _newEntities.Add(entity);
    internal void RegisterDirty(object entity)   => _dirtyEntities.Add(entity);
    internal void RegisterRemoved(object entity) => _removedEntities.Add(entity);

    public Task<int> CommitAsync(CancellationToken ct = default)
    {
        // In a real implementation, this would:
        //   BEGIN TRANSACTION
        //   INSERT all _newEntities
        //   UPDATE all _dirtyEntities
        //   DELETE all _removedEntities
        //   COMMIT

        int count = _newEntities.Count + _dirtyEntities.Count + _removedEntities.Count;

        _newEntities.Clear();
        _dirtyEntities.Clear();
        _removedEntities.Clear();

        return Task.FromResult(count);
    }

    public void Rollback()
    {
        _newEntities.Clear();
        _dirtyEntities.Clear();
        _removedEntities.Clear();
    }

    public void Dispose()
    {
        Rollback();
        GC.SuppressFinalize(this);
    }
}
```

---

## 5. Implementation With EF Core

In practice, **EF Core's `DbContext` IS the Unit of Work**. It has a built-in change tracker and `SaveChangesAsync()`. The goal is to wrap it cleanly behind an interface for testability and maintainability.

### The Domain Models

```csharp
public class Order
{
    public int    Id         { get; set; }
    public int    CustomerId { get; set; }
    public DateTime PlacedAt { get; set; }
    public OrderStatus Status { get; set; }

    public ICollection<OrderLineItem> LineItems { get; set; } = new List<OrderLineItem>();
}

public class OrderLineItem
{
    public int    Id        { get; set; }
    public int    OrderId   { get; set; }
    public int    ProductId { get; set; }
    public int    Quantity  { get; set; }
    public decimal UnitPrice { get; set; }
}

public class InventoryItem
{
    public int ProductId    { get; set; }
    public int QuantityOnHand { get; set; }
}

public enum OrderStatus { Pending, Confirmed, Shipped, Cancelled }
```

### The DbContext

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Order>         Orders         => Set<Order>();
    public DbSet<OrderLineItem> OrderLineItems => Set<OrderLineItem>();
    public DbSet<InventoryItem> Inventory      => Set<InventoryItem>();

    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.Entity<Order>(e =>
        {
            e.HasKey(o => o.Id);
            e.HasMany(o => o.LineItems)
             .WithOne()
             .HasForeignKey(li => li.OrderId)
             .OnDelete(DeleteBehavior.Cascade);
        });

        mb.Entity<InventoryItem>(e =>
        {
            e.HasKey(i => i.ProductId);
        });
    }
}
```

### Generic Repository Implementation

```csharp
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly AppDbContext _context;
    protected readonly DbSet<T>     _set;

    public Repository(AppDbContext context)
    {
        _context = context;
        _set     = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id)
        => await _set.FindAsync(id);

    public async Task<IEnumerable<T>> GetAllAsync()
        => await _set.ToListAsync();

    public async Task AddAsync(T entity)
        => await _set.AddAsync(entity);

    public void Update(T entity)
        => _set.Update(entity);

    public void Remove(T entity)
        => _set.Remove(entity);
}
```

### Specific Repository Implementations

```csharp
public class OrderRepository : Repository<Order>, IOrderRepository
{
    public OrderRepository(AppDbContext context) : base(context) { }

    public async Task<IEnumerable<Order>> GetOrdersByCustomerAsync(int customerId)
        => await _set
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.PlacedAt)
            .ToListAsync();

    public async Task<Order?> GetWithLineItemsAsync(int orderId)
        => await _set
            .Include(o => o.LineItems)
            .FirstOrDefaultAsync(o => o.Id == orderId);
}

public class InventoryRepository : Repository<InventoryItem>, IInventoryRepository
{
    public InventoryRepository(AppDbContext context) : base(context) { }

    public async Task<InventoryItem?> GetByProductIdAsync(int productId)
        => await _set.FindAsync(productId);

    public async Task DeductStockAsync(int productId, int quantity)
    {
        var item = await _set.FindAsync(productId)
            ?? throw new InvalidOperationException($"Product {productId} not in inventory.");

        if (item.QuantityOnHand < quantity)
            throw new InvalidOperationException(
                $"Insufficient stock for product {productId}. " +
                $"Requested: {quantity}, Available: {item.QuantityOnHand}");

        item.QuantityOnHand -= quantity;
        // No SaveChanges() — the UoW will commit this along with everything else
    }
}
```

### The EF Core Unit of Work

```csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    // Lazy-initialised — repositories are created only when accessed
    private IOrderRepository?     _orders;
    private IInventoryRepository? _inventory;

    public UnitOfWork(AppDbContext context) => _context = context;

    public IOrderRepository     Orders
        => _orders    ??= new OrderRepository(_context);

    public IInventoryRepository Inventory
        => _inventory ??= new InventoryRepository(_context);

    public async Task<int> CommitAsync(CancellationToken cancellationToken = default)
    {
        // Single call — EF Core flushes all tracked changes in one transaction
        return await _context.SaveChangesAsync(cancellationToken);
    }

    public void Rollback()
    {
        // Discard all pending changes by resetting tracked entity states
        foreach (var entry in _context.ChangeTracker.Entries())
        {
            entry.State = entry.State switch
            {
                EntityState.Added    => EntityState.Detached,  // un-add it
                EntityState.Modified => EntityState.Unchanged, // revert modifications
                EntityState.Deleted  => EntityState.Unchanged, // un-delete it
                _                    => entry.State
            };
        }
    }

    private bool _disposed;

    public void Dispose()
    {
        if (!_disposed)
        {
            _context.Dispose();
            _disposed = true;
        }
        GC.SuppressFinalize(this);
    }
}
```

### Consuming the Unit of Work in a Service

```csharp
public class OrderService : IOrderService
{
    private readonly IUnitOfWork _uow;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IUnitOfWork uow, ILogger<OrderService> logger)
    {
        _uow    = uow;
        _logger = logger;
    }

    public async Task PlaceOrderAsync(PlaceOrderCommand cmd)
    {
        // 1. Deduct stock (tracked — not saved)
        await _uow.Inventory.DeductStockAsync(cmd.ProductId, cmd.Quantity);

        // 2. Create the order (tracked — not saved)
        var order = new Order
        {
            CustomerId = cmd.CustomerId,
            PlacedAt   = DateTime.UtcNow,
            Status     = OrderStatus.Pending,
        };
        await _uow.Orders.AddAsync(order);

        // 3. Add line items (tracked — not saved)
        order.LineItems.Add(new OrderLineItem
        {
            ProductId = cmd.ProductId,
            Quantity  = cmd.Quantity,
            UnitPrice = cmd.UnitPrice
        });

        // 4. ONE atomic commit — all or nothing
        try
        {
            await _uow.CommitAsync();
            _logger.LogInformation("Order {OrderId} placed successfully.", order.Id);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to place order. Rolling back.");
            _uow.Rollback();
            throw;
        }
    }

    public async Task CancelOrderAsync(int orderId)
    {
        var order = await _uow.Orders.GetByIdAsync(orderId)
            ?? throw new KeyNotFoundException($"Order {orderId} not found.");

        if (order.Status == OrderStatus.Shipped)
            throw new InvalidOperationException("Cannot cancel a shipped order.");

        order.Status = OrderStatus.Cancelled;

        await _uow.CommitAsync(); // single commit for a single operation
    }
}
```

---

## 6. UoW With Transactions (Advanced)

EF Core wraps `SaveChangesAsync()` in a transaction automatically. But sometimes you need **explicit transaction control** — for example, when spanning multiple `SaveChangesAsync()` calls or calling raw SQL alongside EF operations.

```csharp
public interface IUnitOfWork : IDisposable
{
    IOrderRepository     Orders    { get; }
    IInventoryRepository Inventory { get; }

    Task<int> CommitAsync(CancellationToken ct = default);
    void Rollback();

    // Explicit transaction support
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;
    private IDbContextTransaction? _currentTransaction;

    // ... repositories as before ...

    public async Task BeginTransactionAsync()
    {
        if (_currentTransaction is not null)
            throw new InvalidOperationException("A transaction is already in progress.");

        _currentTransaction = await _context.Database.BeginTransactionAsync();
    }

    public async Task CommitTransactionAsync()
    {
        if (_currentTransaction is null)
            throw new InvalidOperationException("No transaction in progress.");

        try
        {
            await _context.SaveChangesAsync();
            await _currentTransaction.CommitAsync();
        }
        catch
        {
            await RollbackTransactionAsync();
            throw;
        }
        finally
        {
            await _currentTransaction.DisposeAsync();
            _currentTransaction = null;
        }
    }

    public async Task RollbackTransactionAsync()
    {
        if (_currentTransaction is null) return;

        await _currentTransaction.RollbackAsync();
        await _currentTransaction.DisposeAsync();
        _currentTransaction = null;
    }

    public async Task<int> CommitAsync(CancellationToken ct = default)
        => await _context.SaveChangesAsync(ct);

    public void Rollback() { /* ... as before ... */ }

    public void Dispose()
    {
        _currentTransaction?.Dispose();
        _context.Dispose();
    }
}
```

### Usage With Explicit Transaction

```csharp
public async Task TransferInventoryAsync(int fromProductId, int toProductId, int quantity)
{
    await _uow.BeginTransactionAsync();

    try
    {
        await _uow.Inventory.DeductStockAsync(fromProductId, quantity);
        // Do something else that might fail...
        await _uow.Inventory.AddStockAsync(toProductId, quantity);
        // Execute raw SQL alongside EF tracked changes
        await _uow.CommitTransactionAsync();
    }
    catch
    {
        await _uow.RollbackTransactionAsync();
        throw;
    }
}
```

---

## 7. Registering With Dependency Injection

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// Register UoW as Scoped — one per HTTP request (same lifetime as DbContext)
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

// Do NOT register repositories separately — they're accessed through IUnitOfWork
// If you also need direct repo access, register them too:
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<IInventoryRepository, InventoryRepository>();
```

> **Important:** Always register `IUnitOfWork` as **Scoped**. It wraps a `DbContext` which is itself Scoped — registering the UoW as Singleton would cause a captive dependency and stale `DbContext` issues.

### Controller Usage

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;

    // OrderService gets IUnitOfWork injected — controller never touches the UoW directly
    public OrdersController(IOrderService orderService) => _orderService = orderService;

    [HttpPost]
    public async Task<IActionResult> PlaceOrder([FromBody] PlaceOrderCommand cmd)
    {
        await _orderService.PlaceOrderAsync(cmd);
        return Ok();
    }
}
```

---

## 8. Unit Testing the Unit of Work

### Option A: Mock IUnitOfWork Directly (Fast, Isolated)

```csharp
public class OrderServiceTests
{
    private readonly Mock<IUnitOfWork>           _uowMock;
    private readonly Mock<IOrderRepository>      _orderRepoMock;
    private readonly Mock<IInventoryRepository>  _inventoryRepoMock;
    private readonly OrderService                _sut;

    public OrderServiceTests()
    {
        _uowMock           = new Mock<IUnitOfWork>();
        _orderRepoMock     = new Mock<IOrderRepository>();
        _inventoryRepoMock = new Mock<IInventoryRepository>();

        // Wire the repo mocks into the UoW mock
        _uowMock.Setup(u => u.Orders).Returns(_orderRepoMock.Object);
        _uowMock.Setup(u => u.Inventory).Returns(_inventoryRepoMock.Object);
        _uowMock.Setup(u => u.CommitAsync(It.IsAny<CancellationToken>()))
                .ReturnsAsync(1);

        _sut = new OrderService(_uowMock.Object, Mock.Of<ILogger<OrderService>>());
    }

    [Fact]
    public async Task PlaceOrder_DeductsInventory_AndCommits()
    {
        // Arrange
        var cmd = new PlaceOrderCommand
        {
            CustomerId = 1,
            ProductId  = 42,
            Quantity   = 3,
            UnitPrice  = 19.99m
        };

        _inventoryRepoMock
            .Setup(r => r.DeductStockAsync(cmd.ProductId, cmd.Quantity))
            .Returns(Task.CompletedTask);

        _orderRepoMock
            .Setup(r => r.AddAsync(It.IsAny<Order>()))
            .Returns(Task.CompletedTask);

        // Act
        await _sut.PlaceOrderAsync(cmd);

        // Assert — inventory was deducted
        _inventoryRepoMock.Verify(
            r => r.DeductStockAsync(cmd.ProductId, cmd.Quantity), Times.Once);

        // Assert — changes were committed exactly once
        _uowMock.Verify(
            u => u.CommitAsync(It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task PlaceOrder_WhenInventoryFails_NeverCommits()
    {
        // Arrange
        _inventoryRepoMock
            .Setup(r => r.DeductStockAsync(It.IsAny<int>(), It.IsAny<int>()))
            .ThrowsAsync(new InvalidOperationException("Insufficient stock."));

        var cmd = new PlaceOrderCommand { CustomerId = 1, ProductId = 1, Quantity = 999 };

        // Act & Assert
        await Assert.ThrowsAsync<InvalidOperationException>(
            () => _sut.PlaceOrderAsync(cmd));

        // Commit must NEVER be called if inventory deduction failed
        _uowMock.Verify(
            u => u.CommitAsync(It.IsAny<CancellationToken>()), Times.Never);
    }
}
```

### Option B: In-Memory EF Core (Integration-Style, No Mocks)

```csharp
public class OrderServiceIntegrationTests : IDisposable
{
    private readonly AppDbContext _context;
    private readonly IUnitOfWork  _uow;
    private readonly OrderService _sut;

    public OrderServiceIntegrationTests()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString()) // unique per test
            .Options;

        _context = new AppDbContext(options);
        _uow     = new UnitOfWork(_context);
        _sut     = new OrderService(_uow, Mock.Of<ILogger<OrderService>>());

        // Seed test data
        _context.Inventory.Add(new InventoryItem { ProductId = 1, QuantityOnHand = 100 });
        _context.SaveChanges();
    }

    [Fact]
    public async Task PlaceOrder_PersistsOrder_AndDecrementsStock()
    {
        var cmd = new PlaceOrderCommand
        {
            CustomerId = 1, ProductId = 1, Quantity = 5, UnitPrice = 10.00m
        };

        await _sut.PlaceOrderAsync(cmd);

        // Verify order was persisted
        var order = await _context.Orders.FirstOrDefaultAsync();
        Assert.NotNull(order);
        Assert.Equal(1, order!.CustomerId);

        // Verify inventory was correctly decremented
        var inventory = await _context.Inventory.FindAsync(1);
        Assert.Equal(95, inventory!.QuantityOnHand); // 100 - 5
    }

    public void Dispose() => _context.Dispose();
}
```

---

## 9. When NOT to Use Unit of Work

The pattern has a real cost — extra abstractions, more boilerplate. It's not always the right choice.

### Skip UoW When...

**You're using a simple CRUD service.** If every operation maps to a single entity and a single `SaveChanges()`, UoW adds no value — just use the repository directly.

**You're using a document database (MongoDB, Cosmos DB).** These often have atomic operations at the document level. Multi-document transactions exist but are expensive and the UoW abstraction fits poorly.

**You're writing a microservice that owns a single bounded context.** A small microservice with two or three tables rarely needs the full Repository + UoW stack. EF Core directly in a service class is perfectly valid.

**You're building a read-heavy query service (CQRS read side).** Unit of Work is about coordinating writes. Read-side queries should bypass it entirely and query the `DbContext` directly for performance and simplicity.

### The Pragmatic Rule

> Use Unit of Work when a single business operation **must touch multiple aggregate roots atomically**. If your operations are naturally one-aggregate-at-a-time, the pattern is overhead.

---

## 10. Common Pitfalls & Anti-Patterns

### Pitfall 1: Calling SaveChanges Inside Repositories

The most common violation of the pattern. Repositories must never save — that is the UoW's job exclusively.

```csharp
// BAD — repository saving independently defeats the entire pattern
public class OrderRepository : IOrderRepository
{
    public async Task AddAsync(Order order)
    {
        await _context.Orders.AddAsync(order);
        await _context.SaveChangesAsync(); // WRONG! Breaks atomicity.
    }
}

// GOOD — only track, never save
public class OrderRepository : IOrderRepository
{
    public async Task AddAsync(Order order)
    {
        await _context.Orders.AddAsync(order);
        // No SaveChanges — the UoW coordinates this
    }
}
```

### Pitfall 2: Registering UoW as Singleton

```csharp
// BAD — UoW wraps DbContext (Scoped). Singleton captures a disposed DbContext.
builder.Services.AddSingleton<IUnitOfWork, UnitOfWork>();

// GOOD
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
```

### Pitfall 3: Creating Multiple UoW Instances Per Request

```csharp
// BAD — two separate units of work, two separate change trackers
// Changes in uow1 are invisible to uow2 — atomicity is broken
public class OrderService
{
    private readonly IUnitOfWork _uow1;
    private readonly IUnitOfWork _uow2;

    public OrderService(IUnitOfWork uow1, IUnitOfWork uow2)
    {
        _uow1 = uow1;
        _uow2 = uow2;
    }
}

// GOOD — one UoW per request, one shared change tracker
public class OrderService
{
    private readonly IUnitOfWork _uow;

    public OrderService(IUnitOfWork uow) => _uow = uow;
}
```

### Pitfall 4: Leaking DbContext Beyond the UoW Abstraction

```csharp
// BAD — consumer directly uses DbContext, bypassing the abstraction
public class OrderService
{
    private readonly IUnitOfWork _uow;
    private readonly AppDbContext _db; // Why is this here?

    public OrderService(IUnitOfWork uow, AppDbContext db)
    {
        _uow = uow;
        _db  = db;
    }

    public async Task PlaceOrderAsync(PlaceOrderCommand cmd)
    {
        // Using both — now you have two potential commit points
        await _uow.Orders.AddAsync(new Order());
        _db.Inventory.Update(item); // bypasses UoW tracking logic
        await _uow.CommitAsync();
    }
}
```

### Pitfall 5: Fat Unit of Work (Too Many Repositories)

```csharp
// SMELL — UoW knows about every entity in the system
public interface IUnitOfWork
{
    IOrderRepository       Orders       { get; }
    IProductRepository     Products     { get; }
    ICustomerRepository    Customers    { get; }
    IInvoiceRepository     Invoices     { get; }
    IShipmentRepository    Shipments    { get; }
    IWarehouseRepository   Warehouses   { get; }
    ISupplierRepository    Suppliers    { get; }
    IReturnsRepository     Returns      { get; }
    // ... 20 more
}
```

**Fix:** Define narrow, bounded UoWs per business operation or aggregate cluster. Use one for Order Fulfilment, one for Inventory Management, one for Invoicing.

---

## 11. Interview Questions (4 YOE Level)

### Conceptual Questions

**Q: What is the Unit of Work pattern and what problem does it solve?**

Unit of Work is a pattern that tracks all changes made to objects during a business transaction and writes them out to the database in a single atomic operation. It solves two problems: (1) preventing partially-committed state when multiple entities need to be saved together, and (2) reducing the number of database round-trips by batching all changes into one `SaveChanges()` call instead of one per repository.

---

**Q: What is the relationship between the Repository pattern and the Unit of Work pattern?**

They are complementary. The Repository pattern abstracts data access for a single entity or aggregate — it handles query and command operations but does *not* persist changes independently. The Unit of Work coordinates multiple repositories, owns the single `CommitAsync()` call, and ensures all their changes are flushed atomically. Repositories are typically exposed as properties on the Unit of Work interface.

---

**Q: Is EF Core's DbContext a Unit of Work?**

Yes, by design. `DbContext` has a built-in change tracker that monitors all entity modifications, and `SaveChangesAsync()` flushes all tracked changes to the database in a single transaction. It is literally an implementation of the Unit of Work pattern. The Repository + UoW wrappers we write around it are additional abstractions for testability and separation of concerns — not replacing what `DbContext` does internally.

---

**Q: Why should repositories never call SaveChanges()?**

Because it breaks atomicity. If `OrderRepository.AddAsync()` calls `SaveChanges()`, the order is committed before the inventory is deducted. If the inventory step then fails, the database is in a corrupt partial state — an order exists with no stock deducted. The Unit of Work exists precisely so that `SaveChanges()` is called *once*, *after all changes have been tracked*, ensuring all-or-nothing semantics.

---

### Implementation Questions

**Q: What lifetime should IUnitOfWork be registered with in ASP.NET Core DI?**

Scoped. The Unit of Work wraps `DbContext`, which is itself Scoped (one per HTTP request). Registering UoW as Transient means every injection gets a different change tracker — one repository's changes would be invisible to another. Registering as Singleton causes a captive dependency — the `DbContext` gets held across requests, leading to stale entities, threading bugs, and `ObjectDisposedException`.

---

**Q: How do you handle exceptions in a Unit of Work? What happens to uncommitted changes?**

In EF Core, if `SaveChangesAsync()` throws, the database transaction is automatically rolled back — nothing is persisted. However, the `DbContext` change tracker still holds the pending changes in memory. You should call `Rollback()` to reset tracked entity states, then either propagate the exception up or return an error. For long-lived operations, wrapping in a try-catch and explicitly resetting state prevents stale entities from accidentally being committed in a subsequent call.

---

**Q: How would you implement explicit transaction control in a Unit of Work?**

Expose `BeginTransactionAsync()`, `CommitTransactionAsync()`, and `RollbackTransactionAsync()` methods on the `IUnitOfWork` interface. Internally, they delegate to `_context.Database.BeginTransactionAsync()` and the resulting `IDbContextTransaction`. This is needed when you have multiple `SaveChanges()` calls within one business operation or when mixing EF tracked changes with raw SQL commands — you want all of them inside the same database transaction boundary.

---

**Q: How do you avoid the "Fat Unit of Work" problem?**

By applying bounded context thinking. Instead of one `IUnitOfWork` that exposes every repository in the application, define narrow, purpose-specific UoWs aligned to business operations: `IOrderFulfillmentUoW` for ordering, `IInventoryUoW` for stock management, `IBillingUoW` for invoicing. Each only exposes the repositories relevant to its domain. This also aligns well with DDD and microservices thinking.

---

**Q: How do you unit test a service that depends on IUnitOfWork?**

Using a mocking framework (e.g. Moq), mock `IUnitOfWork` and set up its repository properties to return mocked `IOrderRepository` and similar objects. Then verify that `CommitAsync()` was called the expected number of times and that the right repository methods were invoked. Alternatively, use an in-memory EF Core database (`UseInMemoryDatabase`) to write integration-style tests without mocking — this exercises the real repository logic while staying fast and self-contained.

---

**Q: What is the difference between a Unit of Work and a database transaction?**

A database transaction is a low-level database concept — it guarantees ACID properties at the SQL level. A Unit of Work is a higher-level application pattern that *uses* a database transaction internally but adds: object tracking (knowing which entities changed), coordination across multiple repositories, and a clean API for business code to work with objects rather than SQL. You can think of UoW as the application-level coordinator and the database transaction as the underlying mechanism it relies on.

---

**Q: Can you use Unit of Work in a CQRS architecture?**

Unit of Work belongs on the **write (command) side** of CQRS, not the read side. Commands mutate state across multiple aggregates — that's where atomicity is needed. On the read side, queries should bypass the UoW and repository abstractions entirely, hitting the `DbContext` directly (or even raw SQL / Dapper) for maximum query flexibility and performance. Applying UoW to reads is over-engineering and adds unnecessary overhead.

---

**Q: How does the Unit of Work pattern relate to the Saga pattern in microservices?**

Unit of Work handles atomicity within a single service's database — local ACID transactions. In a microservices architecture, you often need atomicity *across services*, each with their own database. That's where the Saga pattern comes in — it coordinates a sequence of local transactions across services using events or orchestration. If a step fails, compensating transactions undo the previous steps. Saga is effectively a distributed Unit of Work, but without a shared transaction — eventual consistency rather than ACID.

---

## Summary

The pattern in one sentence:

> **Collect all changes across multiple repositories during a business operation, then flush them to the database in a single atomic transaction.**

Key rules to always follow:

- Repositories **track**, never **save**
- The UoW is the **only** place `SaveChanges()` is called
- Register UoW as **Scoped** — always, without exception
- One UoW per request — never inject multiple instances in the same service
- EF Core's `DbContext` **is** a UoW — your wrapper adds testability, not functionality
- On the CQRS read side, **skip the UoW entirely**

---

*Reference: Martin Fowler — Patterns of Enterprise Application Architecture (2002)*
*https://martinfowler.com/eaaCatalog/unitOfWork.html*

*Reference: Microsoft Docs — Implementing the infrastructure persistence layer with Entity Framework Core*
*https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-implementation-entity-framework-core*