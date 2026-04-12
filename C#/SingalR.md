# SignalR in .NET & Angular — Real-Time Two-Way Communication

---
# Table of Contents
1. [What is signalR](#1-what-is-signalr)
2. [Core concepts of SignalR](#2-core-concepts)
3. [Demo App — Live Chat Room](#3-demo-app--live-chat-room)
4. [ Full Data Flow — Step by Step](#6-full-data-flow--step-by-step)
5. [Targeting specific clients](#7-targeting-specific-clients)
6. [Connection states & Lifecycles](#8-connection-states--lifecycle)
7. [Authentication with signalR](#9-authentication-with-signalr-jwt)
8. [Summary](#10-summary)
9. [Quick reference and cheetsheet](#11-quick-reference-cheatsheet)

## Common Interview questions asked
1. [How signalR can reduce latency in sending information](#1-what-causes-latency-in-signalr)
2. [How can signalR send huge amount of payload ? ](#1-how-signalr-handles-large-payloads)
---

## 1. What is SignalR?

**SignalR** is a .NET library that simplifies adding real-time, two-way communication between server and clients. Unlike traditional HTTP (request → response), SignalR keeps a **persistent connection** open, allowing the server to **push data to clients** at any time.

### How SignalR chooses its transport (automatically)

```
Client connects
      │
      ├── WebSockets available?  ──► ✅ Use WebSockets  (best)
      │
      ├── Server-Sent Events?    ──► ✅ Use SSE         (fallback)
      │
      └── Long Polling?          ──► ✅ Use Long Polling (last resort)
```

---

## 2. Core Concepts

| Concept | Description |
|---|---|
| **Hub** | The central class on the server. Clients connect to it and exchange messages. |
| **HubContext** | Used to push messages to clients from **outside** the Hub (e.g., from a service or controller). |
| **Connection** | Each browser tab gets a unique `connectionId` when it connects. |
| **Groups** | Named sets of connections. Broadcast to a group instead of all clients. |
| **Clients** | A property on the Hub to target specific clients (`All`, `Caller`, `Others`, `Group`). |

---

## 3. Demo App — Live Chat Room

We'll build a **Live Chat Room** where:
- Multiple users connect from Angular
- Any user sends a message → **all** connected users receive it in real time
- The server also broadcasts a **user joined/left** notification

---

###  4. Step 1 — .NET Backend Setup

#### 4.1 Create the project

```bash
dotnet new webapi -n SignalRDemo
cd SignalRDemo
```

SignalR is **built into ASP.NET Core** — no extra NuGet package needed.

---

#### 4.2 Create the Hub

A **Hub** is a class that inherits from `Hub`. Each public method on it becomes callable by clients.

```csharp
// Hubs/ChatHub.cs
using Microsoft.AspNetCore.SignalR;

public class ChatHub : Hub
{
    // ── Called BY the client → broadcasts to ALL clients ──────────────────
    public async Task SendMessage(string user, string message)
    {
        // Clients.All  → every connected client
        // "ReceiveMessage" → the method name Angular will listen for
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }

    // ── Called when a new connection is established ────────────────────────
    public override async Task OnConnectedAsync()
    {
        var connectionId = Context.ConnectionId;
        Console.WriteLine($"Client connected: {connectionId}");

        // Tell everyone else a new user joined
        await Clients.Others.SendAsync("UserJoined", connectionId);

        await base.OnConnectedAsync();
    }

    // ── Called when a client disconnects ──────────────────────────────────
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var connectionId = Context.ConnectionId;
        Console.WriteLine($"Client disconnected: {connectionId}");

        await Clients.Others.SendAsync("UserLeft", connectionId);

        await base.OnDisconnectedAsync(exception);
    }
}
```

---

#### 4.3 Register SignalR & map the Hub

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// 1️⃣ Register SignalR
builder.Services.AddSignalR();

// 2️⃣ Allow Angular (localhost:4200) to connect
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAngular", policy =>
    {
        policy
            .WithOrigins("http://localhost:4200")
            .AllowAnyHeader()
            .AllowAnyMethod()
            .AllowCredentials();   // ⚠️ Required for SignalR
    });
});

var app = builder.Build();

app.UseCors("AllowAngular");   // Must come BEFORE UseRouting

app.MapControllers();

// 3️⃣ Map the Hub to a URL endpoint
app.MapHub<ChatHub>("/hubs/chat");

app.Run();
```

---

#### 4.4 Push messages from OUTSIDE the Hub (HubContext)

Sometimes you need to send a message from a controller or background service — not from inside the Hub itself. Use `IHubContext<T>`:

```csharp
// Controllers/NotificationController.cs
[ApiController]
[Route("api/[controller]")]
public class NotificationController : ControllerBase
{
    private readonly IHubContext<ChatHub> _hubContext;

    public NotificationController(IHubContext<ChatHub> hubContext)
    {
        _hubContext = hubContext;
    }

    [HttpPost("broadcast")]
    public async Task<IActionResult> Broadcast([FromBody] string message)
    {
        // Push to ALL connected clients directly from the controller
        await _hubContext.Clients.All.SendAsync("ReceiveMessage", "System", message);
        return Ok();
    }
}
```

---

#### 4.5 Using Groups (bonus pattern)

Groups let you target a subset of connections — e.g., users in a specific chat room.

```csharp
public class ChatHub : Hub
{
    // Client calls this to join a named room
    public async Task JoinRoom(string roomName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomName);
        await Clients.Group(roomName).SendAsync("ReceiveMessage", "System", $"{Context.ConnectionId} joined {roomName}");
    }

    // Client calls this to send a message to a room only
    public async Task SendToRoom(string roomName, string user, string message)
    {
        await Clients.Group(roomName).SendAsync("ReceiveMessage", user, message);
    }

    public async Task LeaveRoom(string roomName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, roomName);
        await Clients.Group(roomName).SendAsync("ReceiveMessage", "System", $"{Context.ConnectionId} left {roomName}");
    }
}
```

---

### 5. Step 2 — Angular Frontend Setup

#### 5.1 Create the Angular project

```bash
ng new signalr-chat --standalone
cd signalr-chat
```

#### 5.2 Install the SignalR client package

```bash
npm install @microsoft/signalr
```

---

#### 5.3 Create the SignalR Service

The service owns the **connection lifecycle** and exposes **Observables** for components to subscribe to.

```typescript
// src/app/services/chat.service.ts
import { Injectable } from '@angular/core';
import * as signalR from '@microsoft/signalr';
import { BehaviorSubject } from 'rxjs';

export interface ChatMessage {
  user: string;
  message: string;
  timestamp: Date;
}

@Injectable({ providedIn: 'root' })
export class ChatService {
  private hubConnection!: signalR.HubConnection;

  // Streams that components subscribe to
  private messagesSubject = new BehaviorSubject<ChatMessage[]>([]);
  public messages$ = this.messagesSubject.asObservable();

  private notificationsSubject = new BehaviorSubject<string>('');
  public notifications$ = this.notificationsSubject.asObservable();

  // ── Build & start the connection ──────────────────────────────────────
  public startConnection(): Promise<void> {
    this.hubConnection = new signalR.HubConnectionBuilder()
      .withUrl('http://localhost:5000/hubs/chat')   // Must match app.MapHub<> URL
      .withAutomaticReconnect()                      // Auto-reconnect on drop
      .configureLogging(signalR.LogLevel.Information)
      .build();

    // Register listeners BEFORE starting
    this.registerListeners();

    return this.hubConnection
      .start()
      .then(() => console.log('✅ SignalR connected. ID:', this.hubConnection.connectionId))
      .catch(err => console.error('❌ SignalR connection error:', err));
  }

  // ── Listen for events pushed FROM the server ───────────────────────────
  private registerListeners(): void {

    // "ReceiveMessage" must match the string in Clients.All.SendAsync("ReceiveMessage", ...)
    this.hubConnection.on('ReceiveMessage', (user: string, message: string) => {
      const current = this.messagesSubject.getValue();
      this.messagesSubject.next([
        ...current,
        { user, message, timestamp: new Date() }
      ]);
    });

    this.hubConnection.on('UserJoined', (connectionId: string) => {
      this.notificationsSubject.next(`🟢 User joined: ${connectionId}`);
    });

    this.hubConnection.on('UserLeft', (connectionId: string) => {
      this.notificationsSubject.next(`🔴 User left: ${connectionId}`);
    });

    // Handle reconnecting state
    this.hubConnection.onreconnecting(() => {
      console.warn('⚠️ SignalR reconnecting...');
    });

    this.hubConnection.onreconnected((connectionId) => {
      console.log('✅ SignalR reconnected. New ID:', connectionId);
    });
  }

  // ── Send a message TO the server ──────────────────────────────────────
  public sendMessage(user: string, message: string): Promise<void> {
    // "SendMessage" must match the Hub method name
    return this.hubConnection.invoke('SendMessage', user, message);
  }

  // ── Join a named room ─────────────────────────────────────────────────
  public joinRoom(room: string): Promise<void> {
    return this.hubConnection.invoke('JoinRoom', room);
  }

  // ── Disconnect ────────────────────────────────────────────────────────
  public stopConnection(): Promise<void> {
    return this.hubConnection.stop();
  }
}
```

---

#### 5.4 Create the Chat Component

```typescript
// src/app/chat/chat.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { ChatService, ChatMessage } from '../services/chat.service';
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-chat',
  standalone: true,
  imports: [CommonModule, FormsModule],
  templateUrl: './chat.component.html',
  styleUrls: ['./chat.component.css']
})
export class ChatComponent implements OnInit, OnDestroy {
  username = '';
  messageInput = '';
  messages: ChatMessage[] = [];
  notification = '';
  isConnected = false;

  private subs = new Subscription();

  constructor(private chatService: ChatService) {}

  async ngOnInit(): Promise<void> {
    this.username = `User_${Math.floor(Math.random() * 1000)}`;

    await this.chatService.startConnection();
    this.isConnected = true;

    this.subs.add(
      this.chatService.messages$.subscribe(msgs => (this.messages = msgs))
    );

    this.subs.add(
      this.chatService.notifications$.subscribe(note => {
        this.notification = note;
        setTimeout(() => (this.notification = ''), 3000); // auto-clear after 3s
      })
    );
  }

  async sendMessage(): Promise<void> {
    if (!this.messageInput.trim()) return;

    await this.chatService.sendMessage(this.username, this.messageInput);
    this.messageInput = '';
  }

  async ngOnDestroy(): Promise<void> {
    this.subs.unsubscribe();
    await this.chatService.stopConnection();
  }
}
```

---

#### 5.5 Chat Component Template

```html
<!-- src/app/chat/chat.component.html -->
<div class="chat-container">

  <h2>💬 Live Chat</h2>

  <div *ngIf="notification" class="notification">
    {{ notification }}
  </div>

  <!-- Messages list -->
  <div class="messages-box">
    <div
      *ngFor="let msg of messages"
      class="message"
      [class.own]="msg.user === username"
    >
      <span class="user">{{ msg.user }}</span>
      <span class="text">{{ msg.message }}</span>
      <span class="time">{{ msg.timestamp | date:'HH:mm:ss' }}</span>
    </div>
  </div>

  <!-- Input area -->
  <div class="input-area">
    <input
      [(ngModel)]="messageInput"
      placeholder="Type a message..."
      (keyup.enter)="sendMessage()"
    />
    <button (click)="sendMessage()" [disabled]="!isConnected">Send</button>
  </div>

</div>
```

---

## 6. Full Data Flow — Step by Step

```
Angular (User A types a message and clicks Send)
  │
  │  hubConnection.invoke("SendMessage", "UserA", "Hello!")
  ▼
.NET Hub (ChatHub.SendMessage is called)
  │
  │  Clients.All.SendAsync("ReceiveMessage", "UserA", "Hello!")
  ▼
SignalR broadcasts to ALL connected clients
  │
  ├──► Angular (User A) → .on("ReceiveMessage") fires → message added to list
  ├──► Angular (User B) → .on("ReceiveMessage") fires → message added to list
  └──► Angular (User C) → .on("ReceiveMessage") fires → message added to list
```

---

## 7. Targeting Specific Clients

```csharp
// Inside a Hub method — you have access to these targets:

await Clients.All.SendAsync("Event");                        // Everyone
await Clients.Caller.SendAsync("Event");                     // Only sender
await Clients.Others.SendAsync("Event");                     // Everyone except sender
await Clients.Client("connectionId").SendAsync("Event");     // One specific client
await Clients.Clients(listOfIds).SendAsync("Event");         // A list of clients
await Clients.Group("roomName").SendAsync("Event");          // A named group
await Clients.OthersInGroup("roomName").SendAsync("Event");  // Group except sender
```

---

## 8. Connection States & Lifecycle

```
                   ┌─────────────────────────────────┐
                   │         HubConnection            │
                   │                                  │
    .start() ─────►│  Disconnected ──► Connecting     │
                   │                       │          │
                   │                       ▼          │
                   │                   Connected      │
                   │                       │          │
                   │            network drop / error  │
                   │                       │          │
                   │                       ▼          │
                   │    withAutomaticReconnect()       │
                   │          Reconnecting ──► Connected (success)
                   │                       │          │
                   │                       ▼          │
                   │              Disconnected (failed)│
                   └─────────────────────────────────┘
```

| Event hook (Angular) | When it fires |
|---|---|
| `onreconnecting()` | Connection dropped, attempting reconnect |
| `onreconnected()` | Successfully reconnected (new connectionId) |
| `onclose()` | Connection permanently closed |

---

## 9. Authentication with SignalR (JWT)

Pass the JWT token via query string (SignalR can't set Authorization headers over WebSockets):

```typescript
// Angular — attach token to the connection
this.hubConnection = new signalR.HubConnectionBuilder()
  .withUrl('http://localhost:5000/hubs/chat', {
    accessTokenFactory: () => localStorage.getItem('jwt_token') ?? ''
  })
  .build();
```

```csharp
// .NET — read token from query string for WebSocket requests
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];
                var path = context.HttpContext.Request.Path;

                if (!string.IsNullOrEmpty(accessToken) && path.StartsWithSegments("/hubs"))
                {
                    context.Token = accessToken;
                }
                return Task.CompletedTask;
            }
        };
    });

// Protect the hub
app.MapHub<ChatHub>("/hubs/chat").RequireAuthorization();
```

---

## 10. Summary

| Step | What you do |
|---|---|
| **1. .NET** | Create a class inheriting `Hub`, add public methods clients can call |
| **2. .NET** | Register with `AddSignalR()`, map with `app.MapHub<T>("/url")` |
| **3. .NET** | Configure CORS with `AllowCredentials()` |
| **4. Angular** | Install `@microsoft/signalr` |
| **5. Angular** | Build `HubConnection` with `HubConnectionBuilder`, call `.start()` |
| **6. Angular** | Use `.on("EventName", handler)` to **receive** from server |
| **7. Angular** | Use `.invoke("HubMethod", args)` to **send** to server |
| **8. Both** | Use `withAutomaticReconnect()` for resilience |

---

## 11. Quick Reference Cheatsheet

```
SERVER (Hub)                            CLIENT (Angular)
────────────────────────────────────    ────────────────────────────────────
public Task MyMethod(args)          ←── hubConnection.invoke("MyMethod", args)

Clients.All
  .SendAsync("Event", data)         ──► hubConnection.on("Event", handler)

Clients.Caller
  .SendAsync("Event", data)         ──► (only the sender receives it)

Clients.Group("room")
  .SendAsync("Event", data)         ──► (only group members receive it)
```

# Common Interview questions
## Reducing Latency in SignalR

---

### 1. What Causes Latency in SignalR?

Latency in SignalR is the delay between a sender pushing a message and the receiver actually getting it. It compounds across multiple layers:

```
Client sends message
        │
        ▼
[ Network Travel ]              ← Physical distance, ISP routing
        │
        ▼
[ Transport Negotiation ]       ← Choosing WebSocket vs SSE vs Long Polling
        │
        ▼
[ Hub Method Execution ]        ← Slow business logic, DB calls, external APIs
        │
        ▼
[ Serialization ]               ← JSON vs MessagePack, payload size
        │
        ▼
[ Message Broadcast ]           ← Sending to 1 vs 10,000 clients
        │
        ▼
Client receives message
```

**Fix one layer, and you win some latency. Fix all layers, and you win a lot.**

---

### 2. Fix 1 — Force WebSockets, Skip Negotiation

By default, SignalR performs an HTTP negotiation round-trip **before** establishing the real connection. This adds ~100–300ms on every new connection and wastes time trying transports that aren't available.

```
Default behavior:
  Client ──► POST /negotiate ──► Server   (round-trip #1 — wasted)
  Client ──► Try WebSocket   ──► Server   (round-trip #2 — actual connection)

With skipNegotiation:
  Client ──► WebSocket directly ──► Server  ✅ One round-trip saved
```

```typescript
// Angular — skip negotiation entirely, go straight to WebSockets
this.hubConnection = new signalR.HubConnectionBuilder()
  .withUrl('https://yourserver.com/hubs/live', {
    transport: signalR.HttpTransportType.WebSockets,  // Force WebSockets
    skipNegotiation: true                              // Skip the negotiation POST
  })
  .withAutomaticReconnect()
  .build();
```

> ✅ Saves 100–300ms on every new connection.
> ⚠️ Only works if WebSockets are guaranteed to be available (modern browsers + server support).

---

### 3. Fix 2 — Switch to MessagePack (Binary Protocol)

SignalR uses **JSON by default** — human-readable but verbose and slow to serialize. **MessagePack** is a compact binary format that is smaller and faster to encode/decode.

```
JSON:     {"id":1,"price":99.99,"symbol":"AAPL","timestamp":"2026-04-12T10:00:00Z"}
           → 72 bytes, text parsing

MessagePack: [binary]
           → ~35 bytes, binary parsing   ← ~50% smaller, faster to process
```

```bash
dotnet add package Microsoft.AspNetCore.SignalR.Protocols.MessagePack
npm install @microsoft/signalr-protocol-msgpack
```

```csharp
// Server — enable MessagePack with compression
builder.Services.AddSignalR()
    .AddMessagePackProtocol(options =>
    {
        options.SerializerOptions = MessagePackSerializerOptions.Standard
            .WithCompression(MessagePackCompression.Lz4BlockArray);
    });
```

```typescript
// Angular — tell the client to use MessagePack
import { MessagePackHubProtocol } from '@microsoft/signalr-protocol-msgpack';

this.hubConnection = new signalR.HubConnectionBuilder()
  .withUrl('/hubs/live')
  .withHubProtocol(new MessagePackHubProtocol())   // ← binary protocol
  .build();
```

> ✅ Reduces payload size by 30–50%, cuts serialization time noticeably on high-frequency messages.

---

### 4. Fix 3 — Send Only What Changed (Delta / Minimal Payloads)

Every extra byte you send costs time — serialization time, network time, and deserialization time on the client. Broadcast **only the fields that changed**, not the entire object.

```csharp
// ❌ Sends the full order object every time — most fields unchanged
await Clients.All.SendAsync("OrderUpdated", order);

// ✅ Send only the delta — what actually changed
await Clients.All.SendAsync("OrderUpdated", new
{
    order.Id,
    order.Status,       // Only the fields that changed
    order.UpdatedAt
});
```

```csharp
// For live price feeds — strip it to the bare minimum
public record PriceTick(string Symbol, decimal Price);   // 2 fields only

await Clients.Group(symbol).SendAsync("Tick", new PriceTick(symbol, newPrice));
```

```typescript
// Angular — update only the changed property in local state, not full replace
hubConnection.on('OrderUpdated', (delta: Partial<Order>) => {
  const existing = this.orders.find(o => o.id === delta.id);
  if (existing) Object.assign(existing, delta);   // Merge delta, not replace
});
```

> ✅ Fewer bytes = less serialization time + faster network transfer + faster client rendering.

---

### 5. Fix 4 — Use Groups to Target Messages Precisely

Broadcasting to **all clients** when only a subset cares is wasteful. SignalR has to iterate every connection and send each one a copy. Groups limit who receives the message.

```csharp
// ❌ Sends to 50,000 clients when only 200 care about this stock
await Clients.All.SendAsync("PriceUpdate", ticker, newPrice);

// ✅ Only the 200 clients subscribed to AAPL receive this
await Clients.Group("AAPL").SendAsync("PriceUpdate", ticker, newPrice);
```

```csharp
// Hub — clients join the group for what they care about
public class MarketHub : Hub
{
    public async Task Subscribe(string symbol)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, symbol);
    }

    public async Task Unsubscribe(string symbol)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, symbol);
    }
}
```

```typescript
// Angular — subscribe only to tickers the user is watching
async watchStock(symbol: string): Promise<void> {
  await this.hubConnection.invoke('Subscribe', symbol);
}

// Listen for updates on subscribed symbols only
this.hubConnection.on('PriceUpdate', (symbol, price) => {
  this.updateChart(symbol, price);
});
```

> ✅ Dramatically reduces broadcast fan-out. Server does less work per message = lower latency for everyone.

---

### 6. Fix 5 — Keep the Hub Method Lean (No Slow Work Inside)

Every millisecond your hub method spends doing DB calls, HTTP requests, or heavy computation **is added directly to latency**. The hub should be a **thin dispatcher**, not a processing unit.

```csharp
// ❌ Hub does heavy work — every client waits while the hub is busy
public class ChatHub : Hub
{
    public async Task SendMessage(string message)
    {
        await _db.Messages.AddAsync(new Message { Text = message });  // DB write
        await _db.SaveChangesAsync();                                 // More DB
        await _spamFilter.CheckAsync(message);                        // External call
        await _analyticsService.TrackAsync(message);                  // Another call

        // Only NOW does the message get broadcast — very late
        await Clients.All.SendAsync("ReceiveMessage", message);
    }
}

// ✅ Hub broadcasts immediately, offloads slow work to a background queue
public class ChatHub : Hub
{
    private readonly IBackgroundTaskQueue _queue;

    public async Task SendMessage(string user, string message)
    {
        // 1. Broadcast first — zero latency for clients
        await Clients.All.SendAsync("ReceiveMessage", user, message);

        // 2. Enqueue slow work to run after — doesn't block the broadcast
        _queue.Enqueue(async ct =>
        {
            await _db.Messages.AddAsync(new Message { Text = message });
            await _db.SaveChangesAsync(ct);
            await _analyticsService.TrackAsync(message, ct);
        });
    }
}
```

> ✅ Clients receive the message instantly. Persistence and analytics happen asynchronously in the background.

---

### 7. Fix 6 — Throttle & Batch High-Frequency Messages

If your server produces updates faster than clients can usefully render them (e.g., 100 price ticks/second), sending every single one wastes bandwidth and causes client-side congestion. **Batch or throttle** to a rate the client can actually use.

#### Server-side batching (combine multiple updates into one send)

```csharp
public class PriceFeedService : BackgroundService
{
    private readonly IHubContext<MarketHub> _hub;
    private readonly ConcurrentDictionary<string, decimal> _pendingUpdates = new();

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            // Collect all price updates that arrived in the last 50ms
            await Task.Delay(50, ct);

            if (!_pendingUpdates.IsEmpty)
            {
                var batch = _pendingUpdates.ToArray();
                _pendingUpdates.Clear();

                // Send ONE message with all updates — not N individual messages
                await _hub.Clients.All.SendAsync("PriceBatch", batch, ct);
            }
        }
    }

    // Called by market data feed — just accumulates, doesn't send
    public void OnPriceUpdate(string symbol, decimal price)
    {
        _pendingUpdates[symbol] = price;   // Latest price wins within the window
    }
}
```

### Client-side throttling (requestAnimationFrame)

```typescript
// Angular — only render what the screen can actually show (60fps = ~16ms)
private pendingUpdates = new Map<string, number>();
private rafScheduled   = false;

onPriceBatch(updates: [string, number][]): void {
  // Accumulate updates
  updates.forEach(([symbol, price]) => this.pendingUpdates.set(symbol, price));

  // Schedule a single render pass — not one per update
  if (!this.rafScheduled) {
    this.rafScheduled = true;
    requestAnimationFrame(() => {
      this.pendingUpdates.forEach((price, symbol) => this.updateUI(symbol, price));
      this.pendingUpdates.clear();
      this.rafScheduled = false;
    });
  }
}
```

> ✅ Reduces message count dramatically. Each message carries more useful info. Client renders smoothly.

---

### 8. Fix 7 — Scale Out with Redis Backplane (Multi-Server)

When you run **multiple server instances** (behind a load balancer), clients on Server A can't receive messages sent from Server B — they're isolated. A **Redis backplane** routes all messages through a shared pub/sub bus.

```
Without Redis backplane:
  Client A (on Server 1) sends message
  Server 1 ──► only clients on Server 1 get it
  Clients on Server 2, 3 ──► ❌ Never receive it

With Redis backplane:
  Client A (on Server 1) sends message
  Server 1 ──► Redis pub/sub ──► Server 1, Server 2, Server 3
  All clients on all servers ──► ✅ Receive it
```

```bash
dotnet add package Microsoft.AspNetCore.SignalR.StackExchangeRedis
```

```csharp
builder.Services.AddSignalR()
    .AddStackExchangeRedis("redis-connection-string", options =>
    {
        options.Configuration.ChannelPrefix = RedisChannel.Literal("MyApp");
    });
```

> ✅ Enables horizontal scaling. All server instances stay in sync with minimal added latency (Redis pub/sub is sub-millisecond on a local network).

---

### 9. Fix 8 — Tune Keep-Alive and Timeout Intervals

Overly aggressive keep-alive pings add noise. Overly loose settings cause stale connections to linger. Tune them to your use case.

```csharp
builder.Services.AddSignalR(options =>
{
    // How often the server pings the client to confirm connection is alive
    // Default: 15s. Lower = detect drops faster but more overhead
    options.KeepAliveInterval = TimeSpan.FromSeconds(10);

    // How long of silence before the server considers the client disconnected
    // Default: 30s. Should be ~2x KeepAliveInterval
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(20);

    // Allow multiple concurrent hub calls per connection
    // Default: 1 (serialized). Raise for parallel message handling
    options.MaximumParallelInvocationsPerClient = 4;
});
```

> ✅ Lower `KeepAliveInterval` catches dropped connections faster and triggers `withAutomaticReconnect()` sooner.

---

### 10. Fix 9 — Use `IHubContext` from Background Services

Pushing messages from a **background service** via `IHubContext` is faster than routing through an HTTP controller → hub roundtrip. It's a direct call into the SignalR engine.

```csharp
// Background service pushes directly — no HTTP overhead
public class LiveScoreService : BackgroundService
{
    private readonly IHubContext<ScoreHub> _hub;

    public LiveScoreService(IHubContext<ScoreHub> hub) => _hub = hub;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var scores = await _scoreDataSource.GetLatestAsync(ct);

            // Direct push — no HTTP round-trip, no controller overhead
            await _hub.Clients.All.SendAsync("ScoreUpdate", scores, ct);

            await Task.Delay(1000, ct);   // Push every second
        }
    }
}
```

---

### 11. End-to-End Latency Impact Summary

```
Technique                          Latency Saved       Effort
────────────────────────────────────────────────────────────────
Skip negotiation + force WS        100–300ms/connect   Low
MessagePack protocol               10–50ms/message     Low
Minimal payloads (delta only)      5–30ms/message      Low
Precise group targeting            Scales with users   Low
Lean hub methods (offload work)    50–500ms/message    Medium
Batch high-frequency messages      Reduces congestion  Medium
Redis backplane (multi-server)     Prevents split      Medium
Tuned keep-alive intervals         Faster reconnect    Low
IHubContext from background svc    Removes HTTP hop    Low
```

---

### 12. Quick Checklist

```
[ ] Force WebSockets + skipNegotiation on the client
[ ] Switch JSON → MessagePack on both server and client
[ ] Send delta/minimal payloads — not full objects
[ ] Use Groups — never broadcast All when a subset suffices
[ ] Hub methods broadcast first, enqueue slow work after
[ ] Batch rapid updates server-side, throttle renders client-side
[ ] Add Redis backplane if running more than one server instance
[ ] Tune KeepAliveInterval and ClientTimeoutInterval for your SLA
[ ] Push from background services via IHubContext — skip HTTP
```
## 1. How SignalR Handles Large Payloads

---
### 1. The Problem With Large Payloads

SignalR is designed for **real-time, frequent, small messages** — think chat messages, live notifications, cursor positions. When you start sending large payloads, several problems surface:

```
Small message (< 32KB):
  Client ──── "User typed: hello" ────► Server   ✅ Instant, no issues

Large message (> 1MB):
  Client ──── 5MB JSON blob ──────────► Server   ⚠️ Buffered entirely in memory
                                                  ⚠️ Blocks the connection
                                                  ⚠️ May hit size limits and drop
```

**What goes wrong with large payloads:**

| Problem | What Happens |
|---|---|
| **Message size limit hit** | SignalR drops the message and closes the connection |
| **Memory pressure** | Large messages buffered in memory spike RAM usage |
| **Head-of-line blocking** | One large message blocks all other messages on that connection |
| **Timeout** | Transfer takes too long, connection times out mid-send |
| **No progress visibility** | Client has no idea how much has arrived |

---

### 2. SignalR's Built-In Message Size Limit

By default, SignalR enforces a **32KB maximum incoming message size**. Any message larger than this is **rejected** and the connection is closed.

```csharp
// Default limit is 32KB — most large payloads will hit this immediately
// You must explicitly raise it
builder.Services.AddSignalR(options =>
{
    options.MaximumReceiveMessageSize = 102_400;        // 100 KB
    // or
    options.MaximumReceiveMessageSize = 10 * 1024 * 1024; // 10 MB
    // or
    options.MaximumReceiveMessageSize = null;            // ⚠️ No limit — dangerous
});

// Per-hub override
builder.Services.AddSignalR()
    .AddHubOptions<FileHub>(options =>
    {
        options.MaximumReceiveMessageSize = 10 * 1024 * 1024; // 10MB for this hub only
    });
```

> ⚠️ Raising the limit is a **band-aid**, not a solution. A better approach is to never send large payloads as a single message in the first place.

---

### 3. The Right Approach — Chunking (Streaming Chunks)

Instead of sending one large message, **break it into small chunks** and stream them piece by piece. The receiver reassembles them in order.

```
❌ One giant message:
Client ──── [====== 10MB payload ======] ────► Server   Risky, blocking

✅ Chunked:
Client ──── [chunk 1/50] ────► Server
       ──── [chunk 2/50] ────► Server
       ──── [chunk 3/50] ────► Server
       ...
       ──── [chunk 50/50] ───► Server   ✅ Reassemble complete
```

#### Server — Hub that receives chunks

```csharp
public class FileHub : Hub
{
    // In-memory store per upload session (use distributed cache in production)
    private static readonly ConcurrentDictionary<string, List<byte[]>> _uploadSessions = new();

    // Client sends each chunk individually
    public async Task UploadChunk(string uploadId, int chunkIndex, int totalChunks, byte[] data)
    {
        _uploadSessions.AddOrUpdate(
            uploadId,
            _ => [data],
            (_, chunks) => { chunks.Add(data); return chunks; }
        );

        // Acknowledge each chunk so client knows to send the next
        await Clients.Caller.SendAsync("ChunkReceived", uploadId, chunkIndex);

        // All chunks arrived — reassemble
        if (_uploadSessions[uploadId].Count == totalChunks)
        {
            var fullFile = _uploadSessions[uploadId]
                .SelectMany(c => c)
                .ToArray();

            _uploadSessions.TryRemove(uploadId, out _);

            // Process the complete file
            await ProcessFileAsync(uploadId, fullFile);
            await Clients.Caller.SendAsync("UploadComplete", uploadId);
        }
    }

    private Task ProcessFileAsync(string uploadId, byte[] data)
    {
        // Save to disk, blob storage, etc.
        return Task.CompletedTask;
    }
}
```

#### Angular client — sends file in chunks

```typescript
async uploadFile(file: File): Promise<void> {
  const CHUNK_SIZE  = 64 * 1024;        // 64KB per chunk — well within limits
  const totalChunks = Math.ceil(file.size / CHUNK_SIZE);
  const uploadId    = crypto.randomUUID();

  for (let i = 0; i < totalChunks; i++) {
    const start = i * CHUNK_SIZE;
    const slice = file.slice(start, start + CHUNK_SIZE);
    const data  = await slice.arrayBuffer();

    // Send one chunk and wait for acknowledgement before sending next
    await this.hubConnection.invoke(
      'UploadChunk',
      uploadId,
      i,                    // chunkIndex
      totalChunks,
      new Uint8Array(data)  // raw bytes
    );

    const progress = Math.round(((i + 1) / totalChunks) * 100);
    console.log(`Upload progress: ${progress}%`);
  }
}
```

---

### 4. Server-to-Client Streaming with `IAsyncEnumerable`

When the **server** needs to push a large result to the client, use **server-to-client streaming**. The server yields results one at a time instead of building a giant response object.

```csharp
// Hub — streams results as they are produced
public class DataHub : Hub
{
    private readonly AppDbContext _db;

    public DataHub(AppDbContext db) => _db = db;

    // IAsyncEnumerable = stream, not a single return value
    public async IAsyncEnumerable<SalesRecordDto> StreamSalesData(
        int year,
        [EnumeratorCancellation] CancellationToken ct)
    {
        // EF Core streams rows from the DB — never loads all 100K rows into memory
        await foreach (var record in _db.SalesRecords
            .Where(r => r.Year == year)
            .AsAsyncEnumerable()
            .WithCancellation(ct))
        {
            yield return new SalesRecordDto(record);   // pushed to client immediately

            await Task.Delay(10, ct);  // optional throttle — prevent overwhelming client
        }
    }
}
```

```typescript
// Angular client — receives each item as it arrives
startStreaming(year: number): void {
  this.hubConnection
    .stream<SalesRecord>('StreamSalesData', year)
    .subscribe({
      next: (record) => {
        this.records.push(record);          // Render each record as it arrives
      },
      error: (err) => console.error(err),
      complete: () => console.log('Stream complete'),
    });
}

// Stop the stream early
stopStreaming(): void {
  this.streamSubscription?.dispose();
}
```

---

### 5. Client-to-Server Streaming with `ChannelReader`

When the **client** produces data continuously (e.g., live sensor readings, real-time audio chunks), it can open a **client-to-server stream** rather than invoking the hub repeatedly.

```csharp
// Hub — reads from the client's stream
public class SensorHub : Hub
{
    public async Task ProcessSensorStream(ChannelReader<SensorReading> stream)
    {
        // Hub reads each item as the client sends it
        await foreach (var reading in stream.ReadAllAsync())
        {
            // Process each reading in real time
            await SaveReadingAsync(reading);
            await Clients.Group("dashboard").SendAsync("NewReading", reading);
        }
    }
}
```

```typescript
// Angular client — pushes data into a Subject which backs the stream
import { Subject } from 'rxjs';

startSensorStream(): void {
  const subject = new Subject<SensorReading>();

  // Open the stream to the hub
  this.hubConnection.send('ProcessSensorStream', subject);

  // Push readings from the device
  this.sensorDevice.onData((reading) => {
    subject.next(reading);
  });

  // Close the stream when done
  this.sensorDevice.onStop(() => {
    subject.complete();
  });
}
```

---

### 6. Bidirectional Streaming

Both sides stream simultaneously — the client sends data while the server streams back processed results in real time.

```csharp
public class ProcessingHub : Hub
{
    // Receives a stream from client, sends a stream back
    public async IAsyncEnumerable<ProcessedResult> ProcessStream(
        ChannelReader<RawData> inputStream,
        [EnumeratorCancellation] CancellationToken ct)
    {
        await foreach (var raw in inputStream.ReadAllAsync(ct))
        {
            var result = await HeavyProcessAsync(raw);
            yield return result;      // Streamed back to client immediately
        }
    }
}
```

```typescript
// Angular — sends a stream, receives a stream back
const subject = new Subject<RawData>();

this.hubConnection
  .stream<ProcessedResult>('ProcessStream', subject)
  .subscribe({
    next: (result) => this.displayResult(result),
    complete: () => console.log('Processing complete'),
  });

// Feed data into the outgoing stream
rawDataSource.forEach(item => subject.next(item));
subject.complete();
```

---

### 7. Large Binary Data — Use Storage + URL, Not SignalR

For truly large files (images, videos, documents), **don't send the binary through SignalR at all**. Upload directly to blob storage, then use SignalR to send only a lightweight URL notification.

```
❌ Wrong:
Client ──── [20MB video binary] ──► SignalR Hub ──► Other clients

✅ Right:
Client ──── [20MB video] ────────► Azure Blob / S3  (direct upload)
Client ──── "videoReady: url" ──► SignalR Hub ──────────────────────► Other clients
                                                 Other clients fetch video via URL
```

#### Implementation

```csharp
// Controller — handles the actual file upload to blob storage
[ApiController]
[Route("api/[controller]")]
public class MediaController : ControllerBase
{
    private readonly BlobServiceClient _blobClient;
    private readonly IHubContext<MediaHub> _hub;

    [HttpPost("upload")]
    public async Task<IActionResult> Upload(IFormFile file, string roomId)
    {
        // 1. Upload file directly to blob storage
        var containerClient = _blobClient.GetBlobContainerClient("media");
        var blobName        = $"{Guid.NewGuid()}{Path.GetExtension(file.FileName)}";
        var blobClient      = containerClient.GetBlobClient(blobName);

        await blobClient.UploadAsync(file.OpenReadStream());
        var url = blobClient.Uri.ToString();

        // 2. Notify all clients via SignalR — send only the URL, not the file
        await _hub.Clients.Group(roomId).SendAsync("MediaUploaded", new
        {
            Url      = url,
            FileName = file.FileName,
            Size     = file.Length
        });

        return Ok(new { url });
    }
}
```

---

### 8. MessagePack — Smaller Payload Size Over the Wire

By default SignalR serializes messages as **JSON** (text). Switching to **MessagePack** (binary) typically reduces payload size by **30–50%** — an easy win for large-ish payloads.

```bash
dotnet add package Microsoft.AspNetCore.SignalR.Protocols.MessagePack
npm install @microsoft/signalr-protocol-msgpack
```

```csharp
// Server — enable MessagePack
builder.Services.AddSignalR()
    .AddMessagePackProtocol(options =>
    {
        options.SerializerOptions = MessagePackSerializerOptions.Standard
            .WithCompression(MessagePackCompression.Lz4BlockArray);  // + compression
    });
```

```typescript
// Angular — use MessagePack protocol
import { MessagePackHubProtocol } from '@microsoft/signalr-protocol-msgpack';

this.hubConnection = new signalR.HubConnectionBuilder()
  .withUrl('/hubs/data')
  .withHubProtocol(new MessagePackHubProtocol())   // ← switch protocol
  .build();
```

---

### 9. Backpressure — Preventing the Client from Being Overwhelmed

When the server streams faster than the client can process, the client's buffer fills up and the connection stalls. Use **backpressure** to let the client control the flow.

```csharp
// Server — respects a pause/resume signal from the client
public class StreamHub : Hub
{
    public async IAsyncEnumerable<DataBatch> StreamWithBackpressure(
        int batchSize,
        [EnumeratorCancellation] CancellationToken ct)
    {
        var allData = GetAllData();

        foreach (var batch in allData.Chunk(batchSize))
        {
            if (ct.IsCancellationRequested) yield break;

            yield return new DataBatch(batch);

            // Yield control — let the SignalR pipeline flush before producing more
            await Task.Yield();
        }
    }
}
```

```typescript
// Angular — cancel the stream if the client can't keep up
let isPaused = false;
let subscription: ISubscription<DataBatch>;

subscription = this.hubConnection
  .stream<DataBatch>('StreamWithBackpressure', 100)
  .subscribe({
    next: (batch) => {
      if (this.queue.length > 500) {
        subscription.dispose();   // Stop the stream — client is full
        isPaused = true;
      }
      this.processQueue(batch);
    }
  });
```

---

### 10. Configuration Reference

```csharp
builder.Services.AddSignalR(options =>
{
    // Max size of a single incoming message (default: 32KB)
    options.MaximumReceiveMessageSize = 1 * 1024 * 1024;   // 1MB

    // How long client can be unresponsive before disconnect (default: 30s)
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(60);

    // How often keep-alive pings are sent (default: 15s)
    options.KeepAliveInterval = TimeSpan.FromSeconds(20);

    // Max parallel hub invocations per connection (default: 1)
    options.MaximumParallelInvocationsPerClient = 3;
});
```

---

### 11. Strategy Decision Guide

```
How large is your payload?
│
├── < 32KB
│     └──► Send normally — no special handling needed
│
├── 32KB – 1MB
│     ├── One-time send   → Raise MaximumReceiveMessageSize + chunk it
│     └── Streaming data  → IAsyncEnumerable (server) / ChannelReader (client)
│
├── 1MB – 10MB
│     ├── Structured data → Stream with IAsyncEnumerable + MessagePack
│     └── Binary/file     → Chunk upload (64KB chunks) + reassemble on server
│
└── > 10MB (files, video, documents)
      └──► Upload directly to Blob Storage / S3
           Use SignalR only to send the URL notification
```

---

### 12. Summary

| Technique | Direction | Best For |
|---|---|---|
| **Chunking** | Client → Server | File uploads, large binary data |
| **`IAsyncEnumerable` stream** | Server → Client | Large query results, live data feeds |
| **`ChannelReader` stream** | Client → Server | Sensor data, real-time input |
| **Bidirectional streaming** | Both | Processing pipelines |
| **Blob Storage + URL** | External + notify | Files > 10MB, media, documents |
| **MessagePack protocol** | Both | Reduce wire size by 30–50% |
| **Backpressure / cancellation** | Both | Prevent client/server overwhelm |