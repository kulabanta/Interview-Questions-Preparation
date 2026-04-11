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
1. [How signalR can reduce latency in sending information]
2. [How can signalR send huge amounr]
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