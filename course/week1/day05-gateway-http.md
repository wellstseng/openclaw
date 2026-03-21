# Day 5：Gateway Server（上）— HTTP + WebSocket + 認證

> **目標**：理解 Gateway 的 HTTP/WS 伺服器架構和認證機制
> **C# 對照**：ASP.NET Core Kestrel + Middleware Pipeline + SignalR + AuthenticationHandler
> **預計時間**：5 小時

---

## 5.1 Gateway 架構全景

```
Gateway Server = Express (HTTP) + ws (WebSocket) + 自建協定

C# 對照：
  Express     ≈ ASP.NET Core (Kestrel + Middleware)
  ws          ≈ SignalR (WebSocket transport)
  自建協定     ≈ 自訂 SignalR Hub Protocol
```

Gateway 是 OpenClaw 的心臟。所有客戶端（Web UI、macOS App、iOS App、CLI）都透過 WebSocket 連到 Gateway，Gateway 再把訊息轉發到各個 Channel 和 AI 模型。

---

## 5.2 server.ts — Gateway 主入口

```
src/gateway/server.ts
  └→ 匯出 Gateway 的公開 API + 型別定義

src/gateway/server.impl.ts
  └→ Gateway 的實際實作（分離介面和實作）
```

**設計模式**：這是 **Facade Pattern**。`server.ts` 是對外介面，`server.impl.ts` 是內部實作。

```csharp
// C# 等價
// server.ts ≈ IGatewayService (介面)
// server.impl.ts ≈ GatewayService : IGatewayService (實作)
```

---

## 5.3 server-http.ts — Express HTTP Server

### Express 的 Middleware Pipeline

```typescript
// Express 的 middleware 概念和 ASP.NET Core 的 middleware 完全一樣
import express from "express";

const app = express();

// Middleware 1: 解析 JSON body
app.use(express.json());                    // ≈ app.UseJsonBodyParser()

// Middleware 2: CORS
app.use(cors());                            // ≈ app.UseCors()

// Middleware 3: 認證
app.use(authMiddleware);                    // ≈ app.UseAuthentication()

// Route Handler
app.get("/health", (req, res) => {          // ≈ app.MapGet("/health", ...)
  res.json({ status: "ok" });
});

app.post("/api/chat", async (req, res) => { // ≈ app.MapPost("/api/chat", ...)
  const { message } = req.body;
  // ...
});

// 啟動
app.listen(18789);                          // ≈ app.Run("http://0.0.0.0:18789")
```

```csharp
// C# 完全對照
var builder = WebApplication.CreateBuilder();
var app = builder.Build();

app.UseJsonBodyParser();                    // express.json()
app.UseCors();                              // cors()
app.UseAuthentication();                    // authMiddleware

app.MapGet("/health", () =>                 // app.get("/health", ...)
    Results.Json(new { Status = "ok" }));

app.MapPost("/api/chat", async (ChatRequest req) => // app.post(...)
{
    // ...
});

app.Run("http://0.0.0.0:18789");
```

### Express Middleware 的執行順序

```
Request 進來 →
  [express.json()] → 解析 body
    [cors()] → 檢查跨域
      [auth()] → 檢查 token
        [route handler] → 處理請求
      ← 回傳
    ←
  ←
← Response 出去

跟 ASP.NET Core 的 middleware pipeline 完全一樣（洋蔥模型）
```

---

## 5.4 WebSocket Server

```typescript
// Gateway 用 ws 套件建立 WebSocket server
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ server: httpServer });

wss.on("connection", (ws, req) => {
  // 新連線進來
  // ws ≈ SignalR 的 ConnectionContext
  // req ≈ HttpContext

  ws.on("message", (data) => {
    // 收到訊息
    const message = JSON.parse(data.toString());
    handleMessage(ws, message);
  });

  ws.on("close", () => {
    // 連線關閉
    cleanup(ws);
  });
});
```

```csharp
// C# SignalR 等價
public class GatewayHub : Hub
{
    public override async Task OnConnectedAsync()
    {
        // 新連線
    }

    public async Task HandleMessage(JsonElement message)
    {
        // 收到訊息
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        // 連線關閉
    }
}
```

**差異**：SignalR 自動處理序列化/反序列化和重連，ws 是更底層的 WebSocket，需要自己處理協定。

---

## 5.5 auth.ts — 認證機制

```typescript
// src/gateway/auth.ts
// Gateway 支援兩種認證方式：

// 1. Token 認證
// 環境變數 OPENCLAW_GATEWAY_TOKEN
// 客戶端在 WebSocket 連線時帶上 token

// 2. 密碼認證
// 環境變數 OPENCLAW_GATEWAY_PASSWORD
// HTTP Basic Auth 或 WebSocket handshake 參數
```

```csharp
// C# 對照
// 1. Token ≈ Bearer Token Authentication
services.AddAuthentication("Bearer")
    .AddBearerToken(options => {
        options.Token = Environment.GetEnvironmentVariable("GATEWAY_TOKEN");
    });

// 2. Password ≈ Basic Authentication
services.AddAuthentication("Basic")
    .AddBasicAuth(options => { ... });
```

---

## 5.6 server-startup.ts — 啟動流程

Gateway 啟動時的初始化順序：

```typescript
// 概念化的啟動流程
async function bootGateway(options: GatewayOptions) {
  // 1. 載入設定
  const config = await loadConfig();

  // 2. 初始化 Plugin Registry
  await initPluginRegistry(config);

  // 3. 建立 HTTP + WS server
  const server = createServer(config);

  // 4. 註冊所有 Channel
  await registerChannels(config);

  // 5. 啟動 Channel Monitors
  await startChannelMonitors();

  // 6. 啟動排程任務
  await startCronJobs(config);

  // 7. 啟動 mDNS 服務發現
  await startDiscovery();

  // 8. 開始監聽
  server.listen(config.port);
}
```

```csharp
// C# 等價：IHostedService.StartAsync()
public class GatewayHostedService : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        var config = await _configLoader.LoadAsync();
        await _pluginRegistry.InitializeAsync(config);
        await _channelManager.RegisterAllAsync(config);
        await _channelManager.StartMonitorsAsync(ct);
        await _cronScheduler.StartAsync(ct);
        await _discovery.AnnounceAsync();
    }
}
```

---

## 5.7 openai-http.ts — OpenAI 相容 API

Gateway 還提供了 **OpenAI 格式的 HTTP API**，讓任何支援 OpenAI API 的客戶端都能連接。

```typescript
// POST /v1/chat/completions
app.post("/v1/chat/completions", async (req, res) => {
  const { model, messages, stream } = req.body;

  if (stream) {
    // SSE (Server-Sent Events) streaming
    res.setHeader("Content-Type", "text/event-stream");
    for await (const chunk of runChat(messages)) {
      res.write(`data: ${JSON.stringify(chunk)}\n\n`);
    }
    res.write("data: [DONE]\n\n");
    res.end();
  } else {
    const result = await runChat(messages);
    res.json(result);
  }
});
```

```csharp
// C# 等價
app.MapPost("/v1/chat/completions", async (
    ChatCompletionRequest req,
    HttpResponse response) =>
{
    if (req.Stream)
    {
        response.ContentType = "text/event-stream";
        await foreach (var chunk in RunChatAsync(req.Messages))
        {
            await response.WriteAsync($"data: {JsonSerializer.Serialize(chunk)}\n\n");
        }
        await response.WriteAsync("data: [DONE]\n\n");
    }
    else
    {
        return Results.Json(await RunChatAsync(req.Messages));
    }
});
```

---

## 5.8 origin-check.ts — CSRF 防護

```typescript
// 防止 WebSocket 的跨站請求偽造
function checkOrigin(req: IncomingMessage): boolean {
  const origin = req.headers.origin;
  if (!origin) return true; // 非瀏覽器請求允許
  // 檢查 origin 是否在白名單中
  return allowedOrigins.includes(origin);
}
```

```csharp
// C# 等價：CORS + Anti-Forgery
builder.Services.AddCors(options =>
{
    options.AddPolicy("GatewayPolicy", policy =>
        policy.WithOrigins(allowedOrigins));
});
```

---

## 今日閱讀作業

### 作業 1：閱讀 `src/gateway/server.ts`
- 找出 Gateway export 了哪些型別和函式
- 理解它和 `server.impl.ts` 的分離關係

### 作業 2：閱讀 `src/gateway/server-http.ts`
- 找出所有 HTTP route（`app.get`, `app.post` 等）
- 對照 ASP.NET Core 的 `MapGet`, `MapPost`

### 作業 3：閱讀 `src/gateway/auth.ts`
- 理解 Token 和 Password 兩種認證方式
- 找出認證失敗時的處理邏輯

### 作業 4：閱讀 `src/gateway/openai-http.ts`
- 理解 OpenAI 相容 API 的 endpoint 設計
- 找出 streaming 回應的實作方式

---

## 今日 Checkpoint

1. Express 的 `app.use(middleware)` 對應 ASP.NET Core 的什麼？
2. ws 套件和 SignalR 最大的差異是什麼？
3. Gateway 啟動時，Channel 是什麼時候被註冊的？
4. OpenAI 相容 API 的 streaming 用什麼格式？
5. `origin check` 在防禦什麼攻擊？

---

## 答案

1. `app.Use(middleware)` — 完全等價，都是洋蔥模型的 middleware pipeline。
2. ws 是**原始 WebSocket**，需自行處理序列化和協定；SignalR 內建了 Hub 抽象、自動序列化、重連機制。
3. 在 `server-startup.ts` 裡，Plugin Registry 初始化之後、server.listen() 之前。
4. **SSE (Server-Sent Events)**：`Content-Type: text/event-stream`，每個 chunk 用 `data: {...}\n\n` 格式。
5. **CSRF (Cross-Site Request Forgery)**：防止惡意網站透過瀏覽器發起未授權的 WebSocket 連線。
