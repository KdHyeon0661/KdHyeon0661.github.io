---
layout: post
title: AspNet - SignalR로 실시간 기능 구현
date: 2025-04-29 20:20:23 +0900
category: AspNet
---
# 💬 SignalR로 실시간 기능 구현하기 (채팅 예제 중심)

---

## ✅ 1. SignalR이란?

**SignalR**은 ASP.NET Core의 **실시간 기능**을 지원하는 라이브러리로,  
서버 → 클라이언트, 클라이언트 ↔ 클라이언트 간 **양방향 통신**을 쉽게 구현할 수 있어요.

### 🧠 핵심 특징

- WebSocket, Server-Sent Events, Long Polling 등 자동 fallback
- .NET ↔ JS 간 이벤트 기반 통신
- 채팅, 알림, 협업 도구 등에서 자주 사용

---

## 📦 2. SignalR 핵심 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| **Hub** | 서버 측 SignalR 허브 클래스 |
| **Client** | JavaScript/Blazor 클라이언트에서 SignalR 연결 |
| **ConnectionId** | 연결된 사용자 고유 식별자 |
| **Group** | 사용자 그룹화 (채팅방, 알림 채널 등)

---

## 🧱 3. 기본 예제: 실시간 채팅

### 📁 프로젝트 구조

```
MyChatApp/
├── Hubs/
│   └── ChatHub.cs
├── wwwroot/
│   └── chat.js
├── Pages/
│   └── Chat.cshtml
│   └── Chat.cshtml.cs
├── Startup.cs or Program.cs
```

---

## 💻 4. 서버 코드: Hub 정의

```csharp
// Hubs/ChatHub.cs
using Microsoft.AspNetCore.SignalR;

public class ChatHub : Hub
{
    public async Task SendMessage(string user, string message)
    {
        // 모든 클라이언트에게 메시지 전송
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }
}
```

---

## 🌐 5. 클라이언트 코드 (chat.js)

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/microsoft-signalr/7.0.5/signalr.min.js"></script>
<script>
    const connection = new signalR.HubConnectionBuilder()
        .withUrl("/chatHub")
        .build();

    // 서버로부터 메시지를 받으면 출력
    connection.on("ReceiveMessage", (user, message) => {
        const li = document.createElement("li");
        li.textContent = `${user}: ${message}`;
        document.getElementById("messages").appendChild(li);
    });

    connection.start().catch(err => console.error(err.toString()));

    document.getElementById("sendButton").addEventListener("click", event => {
        const user = document.getElementById("userInput").value;
        const message = document.getElementById("messageInput").value;
        connection.invoke("SendMessage", user, message).catch(err => console.error(err.toString()));
        event.preventDefault();
    });
</script>
```

---

## 📄 6. Razor Page (Chat.cshtml)

```html
<h2>SignalR Chat</h2>
<input type="text" id="userInput" placeholder="Name" />
<input type="text" id="messageInput" placeholder="Message" />
<button id="sendButton">Send</button>

<ul id="messages"></ul>

<script src="~/chat.js"></script>
```

---

## 🛠 7. 서버 설정

### ✅ Program.cs 또는 Startup.cs

```csharp
builder.Services.AddSignalR();

app.MapHub<ChatHub>("/chatHub");
```

> `MapHub<ChatHub>`는 클라이언트에서 연결할 URL 경로를 정의

---

## 🔐 8. 사용자 인증 & 그룹 기능 (선택)

### ▶ 인증된 사용자만 허용

```csharp
[Authorize]
public class ChatHub : Hub { ... }
```

### ▶ 사용자 그룹으로 분류

```csharp
public async Task JoinRoom(string room)
{
    await Groups.AddToGroupAsync(Context.ConnectionId, room);
}

public async Task SendToRoom(string room, string user, string message)
{
    await Clients.Group(room).SendAsync("ReceiveMessage", user, message);
}
```

---

## 📦 9. SignalR 클라이언트 옵션

| 플랫폼 | 패키지 |
|--------|--------|
| JavaScript | `microsoft-signalr` (CDN 또는 npm) |
| Blazor (WebAssembly/Server) | 내장 |
| Xamarin/MAUI | `Microsoft.AspNetCore.SignalR.Client` |
| 콘솔 앱 | 가능 (단, HubConnection 직접 구성)

---

## 📡 10. SignalR과 WebSocket

SignalR은 내부적으로 다음 순서로 연결 방식을 선택:

```
WebSocket → Server-Sent Events → Long Polling
```

WebSocket이 가능하면 이를 사용하고, 안되면 자동 fallback.

---

## 🧠 11. 실무 팁

| 항목 | 내용 |
|------|------|
| 연결 유지 | 클라이언트에서 `connection.start()` 재시도 필요 |
| 로그 | `builder.Logging.AddConsole()`로 서버 로그 확인 |
| 스케일 아웃 | Redis 백플레인 필요 (`AddStackExchangeRedis`) |
| 메시지 브로드캐스트 제한 | 그룹(Group) 또는 Caller 제외 등 전략 필요 |

---

## ✅ 요약

| 기능 | 구현 방식 |
|------|------------|
| 실시간 메시지 | Hub에서 `Clients.All.SendAsync` |
| 브라우저 통신 | `signalR.HubConnectionBuilder` |
| 채팅방 | `Groups.AddToGroupAsync()` |
| 인증 사용자 | `[Authorize]` + `Context.User` |
| 클라이언트 지원 | JS, Blazor, Xamarin 등 다양 |

---

## 🔜 추천 다음 주제

- ✅ Blazor Server + SignalR 채팅 구현
- ✅ SignalR + Redis로 분산 서버 확장
- ✅ SignalR에서 연결 상태 관리 (Reconnect 등)
- ✅ SignalR + 인증 연동 (JWT, Cookie)

---

**SignalR**은 복잡한 WebSocket 로직 없이  
쉽게 실시간 채팅, 알림, 스트리밍 같은 기능을 구현할 수 있게 해줍니다.

위 예제를 기반으로 **1:1 채팅**, **방별 채팅**, **알림센터** 등  
다양한 응용이 가능하니 확장해보세요!