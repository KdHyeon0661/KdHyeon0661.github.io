---
layout: post
title: AspNet - SignalR로 실시간 기능 구현
date: 2025-04-29 20:20:23 +0900
category: AspNet
---
# SignalR로 실시간 기능 구현하기

## 0. 빠른 개요

- **SignalR**: ASP.NET Core의 **양방향 실시간 통신** 프레임워크
- **전송 자동 선택**: WebSocket → SSE → Long Polling
- **핵심 추상화**: `Hub`(서버) ↔ 클라이언트(브라우저/모바일/콘솔)
- **현업 포인트**: 인증/권한, 그룹/DM, 재연결, 백플레인(Redis), 메시지 영속화, 성능(MessagePack), 배포 프록시 설정

---

## 1. 프로젝트 뼈대 & 의존성

### 1.1 디렉터리 구조(권장)

```
MyChatApp/
├── src/
│   ├── MyChatApp/               # ASP.NET Core App (Hub/Controllers/EF/DI)
│   │   ├── Hubs/
│   │   │   └── ChatHub.cs
│   │   ├── Services/
│   │   │   └── PresenceTracker.cs
│   │   ├── Data/
│   │   │   ├── AppDbContext.cs
│   │   │   └── Message.cs
│   │   ├── wwwroot/
│   │   │   └── chat.js
│   │   ├── Pages/Chat.cshtml
│   │   └── Program.cs
│   └── MyChatApp.Client.Console/ # .NET 클라이언트(선택)
└── tests/
    └── MyChatApp.Tests/          # xUnit + Moq로 Hub 테스트
```

### 1.2 패키지

```bash
dotnet add src/MyChatApp package Microsoft.AspNetCore.SignalR
dotnet add src/MyChatApp package Microsoft.AspNetCore.SignalR.Protocols.MessagePack
dotnet add src/MyChatApp package Microsoft.EntityFrameworkCore
dotnet add src/MyChatApp package Microsoft.EntityFrameworkCore.Sqlite
dotnet add src/MyChatApp package Microsoft.EntityFrameworkCore.Design
dotnet add src/MyChatApp package StackExchange.Redis
dotnet add tests/MyChatApp.Tests package Moq
dotnet add tests/MyChatApp.Tests package xunit
dotnet add tests/MyChatApp.Tests package Microsoft.AspNetCore.SignalR
```

> 실무에서는 로그/보안/검증 패키지를 추가(예: Serilog, FluentValidation 등).

---

## 2. 최소 동작 예제 — 브로드캐스트 채팅

### 2.1 Hub (서버)

```csharp
// src/MyChatApp/Hubs/ChatHub.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;

public interface IChatClient
{
    Task ReceiveMessage(string user, string message, DateTime utc);
    Task Typing(string user, bool isTyping);
}

[Authorize] // 인증 사용 시
public class ChatHub : Hub<IChatClient>
{
    public async Task SendMessage(string message)
    {
        var user = Context.User?.Identity?.Name ?? "Anonymous";
        await Clients.All.ReceiveMessage(user, message, DateTime.UtcNow);
    }

    public async Task SetTyping(bool isTyping)
    {
        var user = Context.User?.Identity?.Name ?? Context.ConnectionId;
        await Clients.Others.Typing(user, isTyping);
    }
}
```

> 포인트
> - **Strongly-Typed Hub**(`IChatClient`)로 런타임/리팩터링 안정성 ↑
> - 인증을 붙이면 `Context.User`로 사용자 식별 가능

### 2.2 Program.cs (맵핑/프로토콜)

```csharp
// src/MyChatApp/Program.cs
using Microsoft.AspNetCore.SignalR;
using Microsoft.AspNetCore.ResponseCompression;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorPages();
builder.Services.AddSignalR()
    .AddMessagePackProtocol(); // 성능 최적화(바이너리)

builder.Services.AddResponseCompression(opts =>
{
    opts.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
        new[] { "application/octet-stream" }); // MessagePack 압축
});

// (선택) 인증/권한/쿠키/JWT 구성 추가 가능
// builder.Services.AddAuthentication(...);
// builder.Services.AddAuthorization(...);

var app = builder.Build();
app.UseStaticFiles();
app.UseResponseCompression();
app.MapRazorPages();
app.MapHub<ChatHub>("/hubs/chat"); // 허브 엔드포인트

app.Run();
```

### 2.3 Razor Page

```html
<!-- src/MyChatApp/Pages/Chat.cshtml -->
@page
@{
    Layout = null;
}
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <title>SignalR Chat</title>
</head>
<body>
  <h2>SignalR Chat</h2>

  <input id="user" placeholder="Name" />
  <input id="message" placeholder="Message" />
  <button id="send">Send</button>
  <div id="typing"></div>
  <ul id="messages"></ul>

  <script src="https://cdnjs.cloudflare.com/ajax/libs/microsoft-signalr/8.0.0/signalr.min.js"></script>
  <script src="/chat.js"></script>
</body>
</html>
```

### 2.4 클라이언트 JS

```javascript
// src/MyChatApp/wwwroot/chat.js
const userInput = document.getElementById('user');
const messageInput = document.getElementById('message');
const sendBtn = document.getElementById('send');
const messages = document.getElementById('messages');
const typing = document.getElementById('typing');

const connection = new signalR.HubConnectionBuilder()
  .withUrl('/hubs/chat', {
    // accessTokenFactory: () => localStorage.getItem('jwt') // JWT 사용 시
  })
  .withAutomaticReconnect({
    nextRetryDelayInMilliseconds: ctx => Math.min(1000 * (ctx.previousRetryCount + 1), 10000)
  })
  .build();

connection.onreconnecting(err => {
  appendLine(`Reconnecting: ${err?.message ?? ''}`);
});
connection.onreconnected(id => {
  appendLine(`Reconnected. ConnectionId=${id}`);
});
connection.onclose(err => {
  appendLine(`Closed: ${err?.message ?? ''}`);
});

connection.on('ReceiveMessage', (user, msg, utc) => {
  appendLine(`[${new Date(utc).toLocaleTimeString()}] ${user}: ${msg}`);
});

connection.on('Typing', (user, isTyping) => {
  typing.textContent = isTyping ? `${user} is typing...` : '';
});

sendBtn.addEventListener('click', async () => {
  const name = userInput.value || 'Anonymous';
  const text = messageInput.value;
  if (!text) return;

  try {
    await connection.invoke('SendMessage', text);
    messageInput.value = '';
    await connection.invoke('SetTyping', false);
  } catch (e) {
    appendLine(`Send failed: ${e.message}`);
  }
});

let typingTimeout;
messageInput.addEventListener('input', async () => {
  clearTimeout(typingTimeout);
  await connection.invoke('SetTyping', true);
  typingTimeout = setTimeout(() => connection.invoke('SetTyping', false), 1200);
});

function appendLine(text) {
  const li = document.createElement('li');
  li.textContent = text;
  messages.appendChild(li);
}

(async () => {
  try {
    await connection.start();
    appendLine(`Connected. ConnectionId=${connection.connectionId}`);
  } catch (e) {
    appendLine(`Connect failed: ${e.message}`);
  }
})();
```

---

## 3. DM(1:1)과 방(그룹) — 실제 현업 패턴

### 3.1 UserIdentifier 매핑 (커스텀 사용자 키)

```csharp
// src/MyChatApp/Services/NameIdentifierProvider.cs
using Microsoft.AspNetCore.SignalR;

public class NameIdentifierProvider : IUserIdProvider
{
    public string? GetUserId(HubConnectionContext connection)
        => connection.User?.Identity?.Name ?? connection.ConnectionId;
}
```

```csharp
// Program.cs
builder.Services.AddSingleton<IUserIdProvider, NameIdentifierProvider>();
```

### 3.2 DM/그룹 메서드

```csharp
// Hubs/ChatHub.cs (일부)
public async Task JoinRoom(string room)
{
    await Groups.AddToGroupAsync(Context.ConnectionId, room);
    await Clients.Group(room).ReceiveMessage("SYSTEM", $"{Context.UserIdentifier} joined {room}", DateTime.UtcNow);
}

public async Task LeaveRoom(string room)
{
    await Groups.RemoveFromGroupAsync(Context.ConnectionId, room);
    await Clients.Group(room).ReceiveMessage("SYSTEM", $"{Context.UserIdentifier} left {room}", DateTime.UtcNow);
}

public async Task SendToRoom(string room, string message)
{
    var user = Context.UserIdentifier ?? "Anonymous";
    await Clients.Group(room).ReceiveMessage(user, message, DateTime.UtcNow);
}

public async Task SendDirect(string toUser, string message)
{
    var user = Context.UserIdentifier ?? "Anonymous";
    await Clients.User(toUser).ReceiveMessage(user, message, DateTime.UtcNow);
}
```

### 3.3 JS 예시

```javascript
await connection.invoke('JoinRoom', 'dev');
await connection.invoke('SendToRoom', 'dev', 'Hello devs!');
await connection.invoke('SendDirect', 'alice', 'DM hi!');
```

---

## 4. Presence(접속/상태) 추적

### 4.1 In-Memory 트래커(단일 인스턴스)

```csharp
// src/MyChatApp/Services/PresenceTracker.cs
using System.Collections.Concurrent;

public class PresenceTracker
{
    private readonly ConcurrentDictionary<string, HashSet<string>> _online
        = new(StringComparer.OrdinalIgnoreCase);

    public Task UserConnected(string userId, string connectionId)
    {
        var set = _online.GetOrAdd(userId, _ => new HashSet<string>());
        lock (set) set.Add(connectionId);
        return Task.CompletedTask;
    }

    public Task UserDisconnected(string userId, string connectionId)
    {
        if (_online.TryGetValue(userId, out var set))
        {
            lock (set) set.Remove(connectionId);
            if (set.Count == 0) _online.TryRemove(userId, out _);
        }
        return Task.CompletedTask;
    }

    public Task<string[]> OnlineUsers()
        => Task.FromResult(_online.Keys.OrderBy(x => x).ToArray());
}
```

```csharp
// Hubs/ChatHub.cs
public class ChatHub : Hub<IChatClient>
{
    private readonly PresenceTracker _presence;

    public ChatHub(PresenceTracker presence) => _presence = presence;

    public override async Task OnConnectedAsync()
    {
        await _presence.UserConnected(Context.UserIdentifier ?? Context.ConnectionId, Context.ConnectionId);
        await Clients.All.ReceiveMessage("SYSTEM", $"{Context.UserIdentifier} joined", DateTime.UtcNow);
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? ex)
    {
        await _presence.UserDisconnected(Context.UserIdentifier ?? Context.ConnectionId, Context.ConnectionId);
        await Clients.All.ReceiveMessage("SYSTEM", $"{Context.UserIdentifier} left", DateTime.UtcNow);
        await base.OnDisconnectedAsync(ex);
    }
}
```

> **스케일아웃(다중 인스턴스)** 환경에서는 이 Presence 정보를 **Redis** 같은 공유 저장소로 옮겨야 한다(아래 스케일아웃 참고).

---

## 5. 인증/권한(쿠키 or JWT)

### 5.1 쿠키 인증(간단)

```csharp
// Program.cs (개념 예시)
builder.Services.AddAuthentication("Cookies").AddCookie("Cookies");
builder.Services.AddAuthorization();
// 로그인 성공 시 SignInAsync("Cookies") 수행
```

```csharp
// ChatHub.cs
[Authorize] // 인증 사용자만
public class ChatHub : Hub<IChatClient> { ... }
```

### 5.2 JWT 토큰(모바일/SPA)

```csharp
// Program.cs
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", opts =>
    {
        opts.Authority = "https://issuer.example.com"; // 또는 로컬 검증
        opts.Audience = "mychat";
        // WebSocket에서 QueryString 토큰 허용
        opts.Events = new Microsoft.AspNetCore.Authentication.JwtBearer.JwtBearerEvents
        {
            OnMessageReceived = ctx =>
            {
                var accessToken = ctx.Request.Query["access_token"];
                if (!string.IsNullOrEmpty(accessToken) && ctx.HttpContext.Request.Path.StartsWithSegments("/hubs/chat"))
                    ctx.Token = accessToken;
                return Task.CompletedTask;
            }
        };
    });
builder.Services.AddAuthorization();
```

```javascript
// chat.js (JWT 사용 시)
const token = localStorage.getItem('jwt');
const connection = new signalR.HubConnectionBuilder()
  .withUrl('/hubs/chat?access_token=' + encodeURIComponent(token))
  .withAutomaticReconnect()
  .build();
```

> 운영에서는 **HTTPS** 필수, 토큰 유출 방지(만료/회전), 최소 권한 정책 준수.

---

## 6. 메시지 영속화(EF Core) & 최근 메시지 로드

### 6.1 모델 & DbContext

```csharp
// Data/Message.cs
public class Message
{
    public int Id { get; set; }
    public string Room { get; set; } = "lobby";
    public string FromUser { get; set; } = default!;
    public string Content { get; set; } = default!;
    public DateTime Utc { get; set; }
}

// Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public DbSet<Message> Messages => Set<Message>();
    public AppDbContext(DbContextOptions<AppDbContext> opts) : base(opts) { }
}
```

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(opts =>
    opts.UseSqlite("Data Source=app.db"));
```

### 6.2 Hub에 저장 로직

```csharp
public class ChatHub : Hub<IChatClient>
{
    private readonly AppDbContext _db;
    public ChatHub(AppDbContext db) => _db = db;

    public async Task SendToRoom(string room, string message)
    {
        var user = Context.UserIdentifier ?? "Anonymous";
        var msg = new Message { Room = room, FromUser = user, Content = message, Utc = DateTime.UtcNow };
        _db.Messages.Add(msg);
        await _db.SaveChangesAsync();

        await Clients.Group(room).ReceiveMessage(user, message, msg.Utc);
    }

    public async Task<Message[]> RecentMessages(string room, int take = 50)
    {
        take = Math.Clamp(take, 1, 200);
        return await _db.Messages
            .Where(m => m.Room == room)
            .OrderByDescending(m => m.Utc)
            .Take(take)
            .OrderBy(m => m.Utc)
            .ToArrayAsync();
    }
}
```

### 6.3 클라이언트에서 최근 메시지 로드

```javascript
const history = await connection.invoke('RecentMessages', 'dev', 50);
history.forEach(m => appendLine(`[${new Date(m.utc).toLocaleTimeString()}] ${m.fromUser}: ${m.content}`));
```

> 검색/필터/페이지네이션을 추가하면 채팅 로그 뷰어를 만들 수 있다.

---

## 7. 스트리밍 & 바이너리(대용량) 전송

### 7.1 서버→클라이언트 스트리밍

```csharp
using System.Runtime.CompilerServices;

public async IAsyncEnumerable<string> StreamTime([EnumeratorCancellation] CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        yield return DateTime.UtcNow.ToString("O");
        await Task.Delay(1000, ct);
    }
}
```

JS:

```javascript
for await (const item of connection.stream('StreamTime')) {
  appendLine(`tick: ${item}`);
}
```

### 7.2 클라이언트→서버 스트리밍

```csharp
public async Task UploadChunks(IAsyncEnumerable<byte[]> chunks, CancellationToken ct)
{
    await foreach (var chunk in chunks.WithCancellation(ct))
    {
        // 파일 저장/해시 누적/진행률 브로드캐스트 등
    }
}
```

> 대용량 업로드는 **일반 HTTP 업로드** + 업로드 진행 상태를 **SignalR 이벤트**로 보내는 혼합 구조가 운영에 더 안정적이다.

---

## 8. 성능 최적화: MessagePack / 전송량 절약 / 배치

- **MessagePack 프로토콜** 사용(위 Program.cs 참고)
- 메시지 **최대 길이 제한/필터링**(욕설/금지 단어)
- **서버 푸시 빈도 제한**(타이핑 이벤트 등): 스로틀/디바운스
- **송신 배치**: 다수 이벤트를 묶어 전송(서버/클라이언트 측 버퍼링)

---

## 9. 재연결/네트워크 회복 전략

- `withAutomaticReconnect()` + **백오프**
- **상태 표시**: `onreconnecting/onreconnected/onclose`에서 UI 갱신
- 서버에서 `KeepAliveInterval`, `HandshakeTimeout` 조정(특수 네트워크 환경)

---

## 10. 스케일아웃(다중 인스턴스): Redis / Azure SignalR Service

### 10.1 Redis 백플레인

```csharp
// Program.cs
builder.Services.AddSignalR()
    .AddStackExchangeRedis("localhost:6379", options => { /* channel opts */ })
    .AddMessagePackProtocol();
```

- 모든 서버 인스턴스가 **서로의 메시지를 중계** → **Clients.All/Group/User**가 전 인스턴스에 반영
- Presence도 Redis로 공유(예: `HashSet` 대신 Redis SET/Hash)

### 10.2 Azure SignalR Service
- App에 연결 대신 Azure가 허브 역할(프록시/스케일/장애 조치)
- 간단히 **연결 문자열**만 설정하고 App은 Azure SignalR에 붙인다.

---

## 11. 보안/운영 체크리스트

| 항목 | 권장 |
|---|---|
| HTTPS | 필수 |
| 인증/권한 | `[Authorize]`, 정책/역할 기반 |
| 토큰 | 짧은 만료 + 회전 + HTTPS 전송 |
| 입력 검증 | 길이/금지어/HTML 인코딩 |
| 감사/로깅 | 누가/언제/어떤 메시지를 보냈는지 |
| 속도 제한 | 사용자별/연결별 rate limit(아래 참고) |
| 개인정보 | 로그/메시지 마스킹(필요 시) |

### Rate Limit 간단 예시

```csharp
// Hubs/ChatHub.cs (간략)
private static readonly TimeSpan Bucket = TimeSpan.FromSeconds(2);
private static readonly int MaxPerBucket = 10;
private static readonly Dictionary<string, (DateTime win, int count)> _limits = new();

public Task<bool> Allow(string user)
{
    lock (_limits)
    {
        var now = DateTime.UtcNow;
        if (!_limits.TryGetValue(user, out var slot) || now - slot.win > Bucket)
            _limits[user] = (now, 1);
        else if (slot.count + 1 <= MaxPerBucket)
            _limits[user] = (slot.win, slot.count + 1);
        else return Task.FromResult(false);
    }
    return Task.FromResult(true);
}

public async Task SafeSend(string message)
{
    var user = Context.UserIdentifier ?? Context.ConnectionId;
    if (!await Allow(user)) return; // 드롭 또는 경고

    await Clients.All.ReceiveMessage(user, message, DateTime.UtcNow);
}
```

> 운영 환경에서는 **분산 rate-limit**(Redis/슬라이딩 윈도우)를 권장.

---

## 12. 프런트엔드 UX 패턴(채팅 특화)

- 타이핑 표시: `SetTyping`(디바운스 1~2초)
- 읽음 표시: 메시지 ID 기반 **read receipt** 이벤트
- 멘션/하이라이트: `@username` 파싱 → 유저별 알림
- 무한 스크롤: `RecentMessages` 페이지네이션

---

## 13. 리버스 프록시(Nginx) 설정

WebSocket 업그레이드 헤더 필수:

```nginx
server {
    listen 443 ssl;
    server_name example.com;
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass         http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "Upgrade";
        proxy_set_header   Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

> 다중 인스턴스면 **스티키 세션 불필요**(Redis 백플레인 or Azure SignalR 사용 시).

---

## 14. 컨테이너 & docker-compose

```yaml
# docker-compose.yml
version: "3.9"
services:
  redis:
    image: redis:7
    ports: [ "6379:6379" ]
  app:
    build: ./src/MyChatApp
    environment:
      - ASPNETCORE_URLS=http://+:8080
      - ConnectionStrings__Default=Data Source=/data/app.db
      - Redis=redis:6379
    ports: [ "8080:8080" ]
    depends_on: [ redis ]
```

---

## 15. 테스트(단위/통합)

### 15.1 Hub 단위테스트 (xUnit + Moq)

```csharp
// tests/MyChatApp.Tests/ChatHubTests.cs
using Moq;
using Microsoft.AspNetCore.SignalR;
using Xunit;

public class ChatHubTests
{
    [Fact]
    public async Task SendMessage_Broadcasts()
    {
        // Arrange
        var mockClients = new Mock<IHubCallerClients<IChatClient>>();
        var mockClient = new Mock<IChatClient>();
        mockClients.Setup(c => c.All).Returns(mockClient.Object);

        var mockContext = new Mock<HubCallerContext>();
        mockContext.SetupGet(c => c.UserIdentifier).Returns("alice");

        var hub = new ChatHub(null!) // DbContext 미사용 경로
        {
            Clients = mockClients.Object,
            Context = mockContext.Object
        };

        // Act
        await hub.SendMessage("hello");

        // Assert
        mockClient.Verify(c => c.ReceiveMessage("alice", "hello", It.IsAny<DateTime>()), Times.Once);
    }
}
```

> `Hub`은 `Clients`, `Context`, `Groups`가 virtual settable → Moq로 주입 가능.

### 15.2 통합 테스트(WebApplicationFactory)

- `Microsoft.AspNetCore.Mvc.Testing`으로 앱을 띄우고, `HubConnection`(클라이언트)로 실제 연결
- CI에서 **WebSocket 지원 러너** 필요(일반 ubuntu-latest OK)

---

## 16. 고급: 관리자/모더레이션/감사

- `KickUser(toUser)`, `MuteUser(toUser)` 등 **권한별 허브 메서드**
- 메시지 삭제/신고: 메시지 ID로 상태 업데이트 후 **이벤트 브로드캐스트**
- **감사 로그**: 누가 언제 어떤 제재를 가했는지 별도 테이블 저장

```csharp
[Authorize(Roles="Admin")]
public Task KickUser(string userId) =>
    Clients.User(userId).ReceiveMessage("SYSTEM", "You are kicked", DateTime.UtcNow);
```

---

## 17. 장애 대응 & 트러블슈팅

| 증상 | 점검 |
|---|---|
| WebSocket 400/Upgrade 실패 | 프록시 업그레이드 헤더/HTTPS/포트 충돌 확인 |
| 연결 자주 끊김 | 방화벽/프록시 아이들 타임아웃, KeepAlive 조정 |
| 메시지 유실 | 예외 로깅/재시도, 영속화/오프셋 재동기 |
| 스케일 후 그룹/DM 안감 | Redis 백플레인/SignalR Service 적용 여부 |
| CPU 폭주 | 브로드캐스트 남발 → 그룹/필터링, MessagePack, 배치 |

---

## 18. 확장 아이디어

- **알림 센터**: 주문 상태/빌드 상태/댓글 알림
- **협업 편집**: 문서/화이트보드 동시 편집(Operational Transform)
- **라이브 데이터**: 주가/센서/운동 기록 스트리밍
- **게임 서버**: 위치/상태 동기화 + 룸 매칭

---

## 부록 A) .NET 콘솔 클라이언트

```csharp
// src/MyChatApp.Client.Console/Program.cs
using Microsoft.AspNetCore.SignalR.Client;

Console.Write("Name: ");
var name = Console.ReadLine()?.Trim() ?? "cli";

var conn = new HubConnectionBuilder()
    .WithUrl("https://localhost:5001/hubs/chat")
    .WithAutomaticReconnect()
    .Build();

conn.On<string,string,DateTime>("ReceiveMessage", (u,m,utc) =>
{
    Console.WriteLine($"[{utc:HH:mm:ss}] {u}: {m}");
});

await conn.StartAsync();
Console.WriteLine($"Connected: {conn.ConnectionId}");

while (true)
{
    var line = Console.ReadLine();
    if (line is null || line.Equals("/quit", StringComparison.OrdinalIgnoreCase)) break;
    await conn.InvokeAsync("SendMessage", line);
}
```

---

## 부록 B) 클라이언트 권장 옵션 요약(JS)

```javascript
const connection = new signalR.HubConnectionBuilder()
  .withUrl('/hubs/chat', { transport: signalR.HttpTransportType.WebSockets })
  .withAutomaticReconnect([0, 1000, 3000, 5000, 10000])
  .configureLogging(signalR.LogLevel.Information)
  .build();
```

---

## 최종 요약

| 목표 | 구현 포인트 |
|---|---|
| 실시간 채팅 기본 | Hub + JS 클라이언트, `Clients.All/Group/User` |
| 그룹/DM | `Groups.AddToGroupAsync`, `Clients.User` |
| 인증/권한 | 쿠키/JWT + `[Authorize]` + `IUserIdProvider` |
| 안정성 | 자동 재연결, 타임아웃/KeepAlive, 에러 핸들링 |
| 영속화 | EF Core로 메시지 저장 + 히스토리 로드 |
| 스케일아웃 | **Redis 백플레인** 또는 **Azure SignalR Service** |
| 성능 | MessagePack, 압축, 배치/스로틀 |
| 배포 | Nginx 업그레이드 헤더, Docker/Compose |
| 테스트 | Hub 단위테스트 + 통합테스트(옵션) |
