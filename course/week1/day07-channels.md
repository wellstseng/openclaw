# Day 7：Channel 系統 — 多通道架構

> **目標**：理解 Channel 的介面設計、註冊機制、一個 Channel 的完整實作
> **C# 對照**：IHostedService、Strategy Pattern、Factory Pattern、Plugin Architecture
> **預計時間**：5 小時
> **恭喜**：第一週最後一天！完成後你就掌握了 OpenClaw 的外殼。

---

## 7.1 Channel 架構三層設計

```
Layer 1: Registry (src/channels/registry.ts)
  → 「有哪些 channel？」靜態 metadata 清單

Layer 2: Dock (src/channels/dock.ts)
  → 「每個 channel 能做什麼？」能力描述 + 行為設定

Layer 3: Plugin (src/channels/plugins/ + src/telegram/ 等)
  → 「channel 怎麼跑起來？」實際的 Monitor + Message Handler
```

```csharp
// C# 對照
// Layer 1: 類似 enum + attribute 描述
[Channel("telegram", Label = "Telegram", DocsPath = "/channels/telegram")]

// Layer 2: 類似 interface + capability flags
interface IChannelDock { ChannelCapabilities Capabilities { get; } }

// Layer 3: 類似 IHostedService 的具體實作
class TelegramChannel : IChannelPlugin { ... }
```

---

## 7.2 Registry — Channel 註冊表

```typescript
// src/channels/registry.ts

// 所有內建 channel 的 ID 清單（順序決定 UI 顯示順序）
export const CHAT_CHANNEL_ORDER = [
  "telegram", "whatsapp", "discord", "irc",
  "googlechat", "slack", "signal", "imessage",
] as const;

// 從 array 衍生出 union type（Day 2 學的 as const + typeof）
type ChatChannelId = (typeof CHAT_CHANNEL_ORDER)[number];
// = "telegram" | "whatsapp" | "discord" | ...

// 每個 channel 的 metadata
const CHAT_CHANNEL_META: Record<ChatChannelId, ChannelMeta> = {
  telegram: {
    id: "telegram",
    label: "Telegram",
    selectionLabel: "Telegram (Bot API)",
    blurb: "simplest way to get started",
    systemImage: "paperplane",       // macOS/iOS SF Symbol 圖示名
  },
  whatsapp: { ... },
  // ...
};
```

```csharp
// C# 等價
public enum ChatChannelId
{
    Telegram, WhatsApp, Discord, IRC,
    GoogleChat, Slack, Signal, IMessage
}

public static readonly Dictionary<ChatChannelId, ChannelMeta> ChannelMeta = new()
{
    [ChatChannelId.Telegram] = new("Telegram", "Telegram (Bot API)",
        "simplest way to get started", "paperplane"),
    [ChatChannelId.WhatsApp] = new(...),
    // ...
};
```

### Channel Alias（別名系統）

```typescript
export const CHAT_CHANNEL_ALIASES: Record<string, ChatChannelId> = {
  imsg: "imessage",          // 使用者可以打 imsg 代替 imessage
  "google-chat": "googlechat",
  gchat: "googlechat",
};
```

---

## 7.3 Dock — Channel 能力與行為描述

```typescript
// src/channels/dock.ts

export type ChannelDock = {
  id: ChannelId;
  capabilities: ChannelCapabilities;       // 能做什麼
  commands?: ChannelCommandAdapter;         // 指令處理
  outbound?: { textChunkLimit?: number };   // 每則訊息最大字數
  streaming?: ChannelDockStreaming;         // streaming 設定
  elevated?: ChannelElevatedAdapter;        // 管理員功能
  config?: { ... };                         // 設定解析
  groups?: ChannelGroupAdapter;             // 群組行為
  mentions?: ChannelMentionAdapter;         // @mention 處理
  threading?: ChannelThreadingAdapter;      // 討論串支援
};

// 每個 channel 的能力旗標
type ChannelCapabilities = {
  chatTypes: Array<"direct" | "group" | "channel" | "thread">;
  polls?: boolean;           // 支援投票
  reactions?: boolean;       // 支援 emoji 反應
  media?: boolean;           // 支援圖片/檔案
  nativeCommands?: boolean;  // 支援 /command
  threads?: boolean;         // 支援討論串
  blockStreaming?: boolean;  // 支援分塊串流
};
```

```csharp
// C# 等價
public record ChannelDock
{
    public string Id { get; init; }
    public ChannelCapabilities Capabilities { get; init; }
    public int? TextChunkLimit { get; init; }
    // ...
}

[Flags]
public enum ChannelCapabilities
{
    Direct = 1, Group = 2, Channel = 4, Thread = 8,
    Polls = 16, Reactions = 32, Media = 64,
    NativeCommands = 128, Threads = 256, BlockStreaming = 512
}
```

### 各 Channel 的 Dock 設定一覽

```typescript
const DOCKS: Record<ChatChannelId, ChannelDock> = {
  telegram: {
    id: "telegram",
    capabilities: {
      chatTypes: ["direct", "group", "channel", "thread"],
      nativeCommands: true,
      blockStreaming: true,
    },
    outbound: { textChunkLimit: 4000 },
    // Telegram 最多一則 4000 字，超過就分段
  },
  whatsapp: {
    id: "whatsapp",
    capabilities: {
      chatTypes: ["direct", "group"],
      polls: true, reactions: true, media: true,
    },
    outbound: { textChunkLimit: 4000 },
  },
  discord: {
    id: "discord",
    capabilities: {
      chatTypes: ["direct", "channel", "thread"],
      polls: true, reactions: true, media: true,
      nativeCommands: true, threads: true,
    },
    outbound: { textChunkLimit: 2000 },  // Discord 最短：2000 字
  },
  // ... 其他 channel
};
```

---

## 7.4 Plugin 介面 — Channel 的實作合約

```typescript
// src/channels/plugins/types.ts
interface ChannelPlugin {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;

  // 生命週期
  createMonitor(config: Config): ChannelMonitor;

  // 訊息處理
  sendMessage(params: SendMessageParams): Promise<void>;

  // 設定
  config?: {
    resolveAllowFrom(params: { cfg: Config }): string[];
  };

  // 群組
  groups?: ChannelGroupAdapter;

  // 提及
  mentions?: ChannelMentionAdapter;
}

interface ChannelMonitor {
  start(): Promise<void>;
  stop(): Promise<void>;
}
```

```csharp
// C# 等價
public interface IChannelPlugin
{
    string Id { get; }
    ChannelMeta Meta { get; }
    ChannelCapabilities Capabilities { get; }

    IChannelMonitor CreateMonitor(Config config);
    Task SendMessageAsync(SendMessageParams message);
}

public interface IChannelMonitor : IAsyncDisposable
{
    Task StartAsync(CancellationToken ct = default);
    Task StopAsync();
}
```

---

## 7.5 實例研究：Telegram Channel

讓我們追蹤 Telegram 從收到訊息到回覆的完整流程。

### 目錄結構

```
src/telegram/
  ├── accounts.ts       ← 帳號設定解析
  ├── bot.ts            ← grammy Bot 初始化
  ├── commands.ts       ← /start, /help 等 Telegram 指令
  ├── monitor.ts        ← 主要的 Monitor（持續監聽訊息）
  ├── media.ts          ← 處理圖片/影片/語音
  ├── reactions.ts      ← emoji 反應處理
  └── utils.ts          ← 工具函式
```

### Monitor 啟動流程

```typescript
// src/telegram/monitor.ts（概念化）
class TelegramMonitor implements ChannelMonitor {
  private bot: Bot;  // grammy Bot instance

  async start() {
    // 1. 建立 Bot instance
    this.bot = new Bot(this.token);

    // 2. 註冊訊息處理器
    this.bot.on("message:text", async (ctx) => {
      // 收到文字訊息
      await this.handleTextMessage(ctx);
    });

    this.bot.on("message:photo", async (ctx) => {
      // 收到圖片
      await this.handlePhotoMessage(ctx);
    });

    // 3. 開始 polling（長輪詢 Telegram API）
    await this.bot.start();
  }

  private async handleTextMessage(ctx: GrammyContext) {
    // 建立統一的 ChatContext
    const chatContext: ChatContext = {
      channelId: "telegram",
      from: String(ctx.from.id),
      to: String(ctx.chat.id),
      text: ctx.message.text,
      chatType: ctx.chat.type === "private" ? "direct" : "group",
    };

    // 交給 Gateway 的 chat handler 處理
    await this.gateway.handleChatMessage(chatContext);
  }
}
```

```csharp
// C# 等價（用 Telegram.Bot NuGet）
public class TelegramMonitor : IChannelMonitor
{
    private TelegramBotClient _bot;

    public async Task StartAsync(CancellationToken ct)
    {
        _bot = new TelegramBotClient(_token);
        _bot.StartReceiving(
            updateHandler: HandleUpdateAsync,
            errorHandler: HandleErrorAsync,
            cancellationToken: ct
        );
    }

    private async Task HandleUpdateAsync(
        ITelegramBotClient bot, Update update, CancellationToken ct)
    {
        if (update.Message?.Text is { } text)
        {
            var context = new ChatContext
            {
                ChannelId = "telegram",
                From = update.Message.From!.Id.ToString(),
                To = update.Message.Chat.Id.ToString(),
                Text = text,
                ChatType = update.Message.Chat.Type == ChatType.Private
                    ? "direct" : "group",
            };
            await _gateway.HandleChatMessageAsync(context);
        }
    }
}
```

### 訊息回覆流程

```typescript
// AI 回覆後，透過 Telegram Bot API 送出
async function sendReply(chatId: string, text: string) {
  // 如果超過 4000 字，分段送出
  const chunks = splitText(text, 4000);
  for (const chunk of chunks) {
    await bot.api.sendMessage(chatId, chunk, {
      parse_mode: "Markdown",
    });
  }
}
```

---

## 7.6 Extension Channel vs 內建 Channel

| 特性 | 內建 Channel (8 個) | Extension Channel (35+) |
|------|-------------------|----------------------|
| 位置 | `src/telegram/` 等 | `extensions/telegram/` 等 |
| 載入 | 靜態 import | Plugin 動態載入 |
| Dock | 在 `dock.ts` 硬編碼 | 由 Plugin 提供 |
| 升級 | 跟 core 一起升級 | 獨立版本管理 |

Extension 的載入流程：
```typescript
// Plugin Registry 在啟動時掃描 extensions/ 目錄
// 每個 extension 的 package.json 裡有 "openclaw" 欄位定義 plugin 類型
// Registry 把它載入後，和內建 channel 一起管理
```

---

## 7.7 第一週總回顧

```
你現在掌握的 OpenClaw 地圖：
                                                    ✅ 你在這裡
┌─────────────────────────────────────────────────────────────┐
│  entry.ts → index.ts → CLI Program                    Day 4 │
│      ↓                                                      │
│  Gateway Server (HTTP + WebSocket)                    Day 5 │
│      ↓                                                      │
│  Chat Handler → Lane → Session → AI Agent             Day 6 │
│      ↓                                                      │
│  Channel Registry → Dock → Plugin → Monitor           Day 7 │
│      ↓                                                      │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │Telegram │ │WhatsApp │ │Discord  │ │  ...    │          │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘          │
└─────────────────────────────────────────────────────────────┘

下週要進入的區域：
  → AI Agent Engine (Day 8-9)
  → Memory/RAG (Day 10)
  → Plugin System (Day 11)
  → Config + CLI + Routing (Day 12)
  → Extensions + Skills (Day 13)
  → 總複習 (Day 14)
```

---

## 今日閱讀作業

### 作業 1：閱讀 `src/channels/registry.ts`（完整閱讀）
- 特別注意 `as const` + `typeof` 的型別技巧
- 理解 `normalizeChannelId` 的邏輯

### 作業 2：閱讀 `src/channels/dock.ts`（完整閱讀）
- 比較不同 channel 的 capabilities 差異
- 找出哪個 channel 的 textChunkLimit 最小

### 作業 3：選一個 channel 深入閱讀
- 建議選 `src/telegram/`（最標準）或 `src/discord/`
- 追蹤從收到訊息到回覆的完整流程

---

## 今日 Checkpoint

1. Registry、Dock、Plugin 三層分別負責什麼？
2. `as const` + `typeof ARRAY[number]` 這個 pattern 在做什麼？
3. 哪個 channel 的 textChunkLimit 最小？為什麼？
4. Channel Monitor 的 `start()` 對應 C# 的什麼？
5. Extension Channel 和內建 Channel 的差異是什麼？

---

## 答案

1. Registry = metadata 清單（「有哪些 channel」）、Dock = 能力與行為設定（「能做什麼」）、Plugin = 實際實作（「怎麼跑」）。
2. 從一個常數陣列衍生出 Union Type。`["a", "b"] as const` + `typeof X[number]` = `"a" | "b"`。C# 等價大概是從 enum 衍生。
3. **Discord = 2000 字**。因為 Discord API 的訊息長度上限就是 2000。
4. `IHostedService.StartAsync()` — 啟動後持續監聽訊息。
5. 內建的靜態載入、在 dock.ts 硬編碼；Extension 的動態載入、由 Plugin 自行提供 Dock。
