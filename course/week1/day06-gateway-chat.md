# Day 6：Gateway Server（下）— Chat 核心 + 廣播 + 排程

> **目標**：理解聊天訊息的處理核心、WebSocket 廣播、排程系統
> **C# 對照**：SignalR Hub Methods、IHubContext<T>.Clients、IHostedService + Timer
> **預計時間**：5 小時

---

## 6.1 server-chat.ts — 聊天核心（整個系統最重要的檔案之一）

這個檔案處理所有聊天訊息的生命週期：收到訊息 → 找 session → 呼叫 AI → 回傳結果。

### 訊息處理流程

```typescript
// 概念化的 chat 處理流程
async function handleChatMessage(context: ChatContext) {
  // 1. 解析 session key（誰在哪個 channel 說了什麼）
  const sessionKey = resolveSessionKey(context);

  // 2. 檢查權限（allowlist）
  if (!isAllowed(context)) return;

  // 3. 檢查指令（如 /help, /reset）
  if (isCommand(context.text)) {
    return handleCommand(context);
  }

  // 4. 載入歷史對話
  const history = await loadSessionHistory(sessionKey);

  // 5. 呼叫 AI Agent
  const response = await runAgent({
    messages: [...history, { role: "user", content: context.text }],
    model: resolveModel(context),
    tools: resolveTools(context),
  });

  // 6. 發送回覆
  await sendReply(context, response);

  // 7. 儲存對話紀錄
  await saveToHistory(sessionKey, context.text, response);
}
```

```csharp
// C# 等價
public class ChatService
{
    public async Task HandleMessageAsync(ChatContext context)
    {
        var sessionKey = _routing.ResolveSessionKey(context);

        if (!_auth.IsAllowed(context)) return;

        if (context.Text.StartsWith("/"))
            return await HandleCommandAsync(context);

        var history = await _sessions.LoadHistoryAsync(sessionKey);

        var response = await _agentRunner.RunAsync(new AgentRequest
        {
            Messages = [..history, new UserMessage(context.Text)],
            Model = _modelSelector.Resolve(context),
            Tools = _toolResolver.Resolve(context),
        });

        await _channel.SendReplyAsync(context, response);
        await _sessions.SaveAsync(sessionKey, context.Text, response);
    }
}
```

---

## 6.2 server-broadcast.ts — WebSocket 廣播

Gateway 需要把事件廣播給所有連線的客戶端（Web UI、App 等）。

```typescript
// 廣播機制
function broadcast(clients: Set<WebSocket>, message: object) {
  const data = JSON.stringify(message);
  for (const client of clients) {
    if (client.readyState === WebSocket.OPEN) {
      client.send(data);
    }
  }
}

// 分組廣播（只送給特定 session 的訂閱者）
function broadcastToSession(sessionId: string, message: object) {
  const subscribers = sessionSubscriptions.get(sessionId) ?? [];
  for (const ws of subscribers) {
    ws.send(JSON.stringify(message));
  }
}
```

```csharp
// C# SignalR 等價
public class GatewayHub : Hub
{
    // 廣播給所有人
    await Clients.All.SendAsync("event", message);

    // 分組廣播（只送給特定 session 的訂閱者）
    await Clients.Group(sessionId).SendAsync("event", message);
}

// 在 Hub 外部廣播
public class SomeService(IHubContext<GatewayHub> hubContext)
{
    await hubContext.Clients.Group(sessionId).SendAsync("event", message);
}
```

**差異**：SignalR 內建 Group 管理；ws 要自己維護 `Map<sessionId, Set<WebSocket>>`。

---

## 6.3 server-channels.ts — Channel 生命週期管理

```typescript
// 管理所有 channel 的啟動和停止
class ChannelManager {
  private monitors: Map<string, ChannelMonitor> = new Map();

  async startAll(config: Config) {
    for (const channelId of config.enabledChannels) {
      const monitor = createMonitor(channelId);
      await monitor.start();
      this.monitors.set(channelId, monitor);
    }
  }

  async stopAll() {
    for (const [id, monitor] of this.monitors) {
      await monitor.stop();
    }
    this.monitors.clear();
  }

  // 動態啟停（hot reload）
  async restart(channelId: string) {
    const existing = this.monitors.get(channelId);
    if (existing) await existing.stop();
    const monitor = createMonitor(channelId);
    await monitor.start();
    this.monitors.set(channelId, monitor);
  }
}
```

```csharp
// C# 等價：IHostedService 集合管理
public class ChannelManager : IHostedService
{
    private readonly Dictionary<string, IChannelMonitor> _monitors = new();

    public async Task StartAsync(CancellationToken ct)
    {
        foreach (var channelId in _config.EnabledChannels)
        {
            var monitor = _factory.Create(channelId);
            await monitor.StartAsync(ct);
            _monitors[channelId] = monitor;
        }
    }

    public async Task StopAsync(CancellationToken ct)
    {
        foreach (var monitor in _monitors.Values)
            await monitor.StopAsync(ct);
        _monitors.Clear();
    }
}
```

---

## 6.4 server-cron.ts — 排程任務

```typescript
// 使用 croner 套件做 cron 排程
import { Cron } from "croner";

// 設定檔裡定義排程
// cron:
//   - schedule: "0 9 * * *"      # 每天早上 9 點
//     action: "send"
//     message: "Good morning!"
//     target: "whatsapp:+886..."

function startCronJobs(config: Config) {
  for (const job of config.cron ?? []) {
    new Cron(job.schedule, async () => {
      await executeCronAction(job);
    });
  }
}
```

```csharp
// C# 等價：IHostedService + Timer 或 Quartz.NET
public class CronHostedService : IHostedService
{
    public Task StartAsync(CancellationToken ct)
    {
        foreach (var job in _config.CronJobs)
        {
            // 用 Quartz.NET
            var trigger = TriggerBuilder.Create()
                .WithCronSchedule(job.Schedule)
                .Build();
            await _scheduler.ScheduleJob(CreateJob(job), trigger);
        }
    }
}
```

---

## 6.5 server-lanes.ts — 請求分流

```typescript
// Lane = 一個並行處理的「車道」
// 用來控制同一個 session 不會同時跑兩個 AI 請求

class LaneManager {
  private lanes: Map<string, Promise<void>> = new Map();

  async enqueue(sessionKey: string, task: () => Promise<void>) {
    // 等待前一個同 session 的任務完成
    const previous = this.lanes.get(sessionKey) ?? Promise.resolve();
    const current = previous.then(task).catch(() => {});
    this.lanes.set(sessionKey, current);
    await current;
  }
}
```

```csharp
// C# 等價：SemaphoreSlim per session
public class LaneManager
{
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _lanes = new();

    public async Task EnqueueAsync(string sessionKey, Func<Task> task)
    {
        var semaphore = _lanes.GetOrAdd(sessionKey, _ => new SemaphoreSlim(1, 1));
        await semaphore.WaitAsync();
        try
        {
            await task();
        }
        finally
        {
            semaphore.Release();
        }
    }
}
```

**注意**：這裡 Node.js 不需要 SemaphoreSlim（因為單線程），用 Promise chain 就能保證順序。但概念相同。

---

## 6.6 server-discovery.ts — 區域網路服務發現

```typescript
// 使用 mDNS (Bonjour/Zeroconf) 讓 App 自動找到 Gateway
import { Responder } from "@homebridge/ciao";

const responder = Responder.create();
const service = responder.createService({
  name: "OpenClaw Gateway",
  type: "openclaw",
  port: 18789,
});
await service.advertise();  // 開始廣播「我在這裡」
```

```csharp
// C# 等價：用 Makaretu.Dns.Multicast 或 Tmds.MDns
// macOS/iOS App 掃描區域網路時能自動找到 Gateway
// 就像 AirPlay 設備被自動發現一樣
```

---

## 今日閱讀作業

### 作業 1：閱讀 `src/gateway/server-chat.ts`
- 追蹤一則訊息從收到到回覆的完整路徑
- 找出 session key 是怎麼解析的
- 找出 AI Agent 是在哪裡被呼叫的

### 作業 2：閱讀 `src/gateway/server-broadcast.ts`
- 理解 WebSocket 廣播的機制
- 對照 SignalR 的 Group 概念

### 作業 3：閱讀 `src/gateway/server-cron.ts`
- 理解 cron 排程是怎麼設定和執行的

---

## 今日 Checkpoint

1. Chat 處理的第一步是什麼？
2. 為什麼需要 Lane（車道）管理？
3. mDNS 服務發現解決什麼問題？
4. server-broadcast 和 SignalR 的 Clients.Group() 概念一樣嗎？
5. 如果同一個使用者快速發了 3 則訊息，Lane 會怎麼處理？

---

## 答案

1. **解析 Session Key**——確定「誰在哪個 channel 發了訊息」，這決定了用哪個 session 的歷史紀錄和設定。
2. 防止同一個 session 同時跑多個 AI 請求（避免回覆亂序或重複）。
3. 讓 App 不需要手動輸入 Gateway IP，自動在區域網路上發現 Gateway。
4. **概念一樣**。差異是 SignalR 內建 Group 管理，ws 要自己維護訂閱表。
5. 排隊依序執行：第 1 則處理完才處理第 2 則，第 2 則完才處理第 3 則。
