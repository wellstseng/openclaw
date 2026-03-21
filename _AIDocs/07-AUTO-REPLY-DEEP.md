# Auto-Reply 引擎深入

> Phase 1-1 產出 | 2026-03-12 | 掃描範圍：`src/auto-reply/` 281 .ts + `src/web/auto-reply/` 28 .ts

## 一句話定位

**Auto-Reply 是 Gateway 與 Agent Engine 之間的「訊息處理中間層」**：接收各頻道格式化好的 inbound message，經命令解析、Session 管理、指令解析、權限檢查後，呼叫 Agent 引擎產生回覆，再透過 Dispatcher 將回覆投遞回原頻道。

> C# 對照：相當於 ASP.NET Core 的 **Middleware Pipeline** + **MediatR Handler** — 每個 inbound 經過一連串中間處理（auth、session、directive、command），最後交給 Agent（≈ MediatR Handler）處理，回覆經 Dispatcher（≈ IActionResult + OutputFormatter）投遞。

---

## 架構全景

```
Channel Monitor (Telegram/Discord/WhatsApp/Web/...)
    │
    │ onMessage()
    ▼
┌─────────────────────────────────────────────────────┐
│              Auto-Reply Pipeline                     │
│                                                      │
│  ┌──────────┐  ┌───────────┐  ┌─────────────────┐  │
│  │ Envelope  │→│ Inbound   │→│  Session Init    │  │
│  │ Formatting│  │ Context   │  │  (reset/fork/    │  │
│  │           │  │ Finalize  │  │   group resolve) │  │
│  └──────────┘  └───────────┘  └────────┬────────┘  │
│                                         │           │
│  ┌──────────┐  ┌───────────┐  ┌────────▼────────┐  │
│  │ Command  │←│ Directive  │←│ Command Auth     │  │
│  │ Handling │  │ Parsing   │  │ + Send Policy    │  │
│  │ (/new    │  │ (/model   │  │                  │  │
│  │  /status │  │  /think   │  │                  │  │
│  │  /help)  │  │  /verbose │  │                  │  │
│  └────┬─────┘  │  /queue)  │  └──────────────────┘  │
│       │        └───────────┘                         │
│       │ (not a command)                              │
│       ▼                                              │
│  ┌─────────────────────────────────────────────┐    │
│  │         runPreparedReply()                   │    │
│  │                                              │    │
│  │  Group Context → Trigger Check → Queue       │    │
│  │  → Agent Runner → Block Streaming            │    │
│  │  → Followup → Memory Flush                   │    │
│  └──────────────────────┬──────────────────────┘    │
│                          │                           │
│  ┌──────────────────────▼──────────────────────┐    │
│  │         Reply Dispatcher                     │    │
│  │  sendToolResult / sendBlockReply / sendFinal │    │
│  │  (serialize, normalize, human delay)         │    │
│  └──────────────────────┬──────────────────────┘    │
│                          │                           │
└──────────────────────────┼───────────────────────────┘
                           │
                           ▼
                  Channel Delivery
           (deliverWebReply / routeReply)
```

---

## 目錄結構

```
src/auto-reply/
├── types.ts                    # ReplyPayload, GetReplyOptions, TypingPolicy
├── reply.ts                    # Barrel: re-exports from reply/
├── dispatch.ts                 # dispatchInboundMessage() — 頂層入口
├── envelope.ts                 # formatAgentEnvelope() — [Channel From Time] Body
├── templating.ts               # MsgContext + TemplateContext 型別定義（150+ 欄位）
├── command-detection.ts        # hasControlCommand / isControlCommandMessage
├── command-auth.ts             # resolveCommandAuthorization
├── commands-registry.ts        # Chat command registry + detection
├── commands-registry.data.ts   # 40+ 命令定義（/help, /status, /model, ...）
├── commands-args.ts            # 命令引數解析
├── chunk.ts                    # 出站文字分塊（ChunkMode: length | newline）
├── heartbeat.ts                # Heartbeat prompt + HEARTBEAT_OK stripping
├── heartbeat-reply-payload.ts  # Heartbeat 回覆格式化
├── tokens.ts                   # HEARTBEAT_OK, NO_REPLY 常量
├── model.ts                    # /model 指令解析
├── thinking.ts                 # /think, /verbose, /reasoning 等級解析
├── group-activation.ts         # 群組觸發模式 (mention | always)
├── send-policy.ts              # Send policy override (allow | deny)
├── inbound-debounce.ts         # 入站訊息防抖（per-channel configurable）
├── fallback-state.ts           # 模型 fallback 通知
├── model-runtime.ts            # Provider/model ref 格式化
├── media-note.ts               # Inbound 媒體附註
├── skill-commands.ts           # Skill command 處理
├── status.ts                   # /status 回覆構建
├── tool-meta.ts                # Tool 元資料
│
├── reply/                      # ★ 核心回覆引擎（~120 非測試檔）
│   ├── get-reply.ts            # getReplyFromConfig() — 回覆主入口
│   ├── get-reply-run.ts        # runPreparedReply() — 準備後執行
│   ├── get-reply-directives.ts # resolveReplyDirectives() — 指令解析
│   ├── get-reply-inline-actions.ts  # handleInlineActions() — 內聯命令
│   ├── dispatch-from-config.ts # dispatchReplyFromConfig() — 含 routing + TTS
│   ├── reply-dispatcher.ts     # ReplyDispatcher — 序列化出站 queue
│   ├── dispatcher-registry.ts  # 全域 dispatcher 註冊（gateway restart）
│   ├── agent-runner.ts         # runReplyAgent() — 橋接 Agent 引擎
│   ├── agent-runner-execution.ts  # runAgentTurnWithFallback()
│   ├── agent-runner-helpers.ts    # typing signals, followup helpers
│   ├── agent-runner-memory.ts     # 記憶體 flush
│   ├── agent-runner-payloads.ts   # buildReplyPayloads()
│   ├── agent-runner-reminder-guard.ts  # Cron reminder 防護
│   ├── agent-runner-utils.ts      # 嵌入式執行參數構建
│   ├── block-reply-pipeline.ts    # Block streaming 管線
│   ├── block-reply-coalescer.ts   # Block 合併器
│   ├── block-streaming.ts        # Block streaming 配置解析
│   ├── session.ts                # initSessionState() — Session 初始化核心
│   ├── session-*.ts              # Session 相關（reset, fork, hooks, usage）
│   ├── inbound-context.ts        # finalizeInboundContext()
│   ├── inbound-dedupe.ts         # 入站去重
│   ├── inbound-meta.ts           # 入站 meta system prompt
│   ├── inbound-text.ts           # 入站文字正規化
│   ├── commands-core.ts          # handleCommands() — 命令路由核心
│   ├── commands-*.ts             # 各命令實作（20+ 檔）
│   ├── directive-handling*.ts    # 指令解析實作（10+ 檔）
│   ├── directives.ts             # 指令萃取（/think, /verbose, /elevated）
│   ├── typing.ts                 # TypingController — 打字指示器
│   ├── typing-mode.ts            # 打字模式解析
│   ├── typing-policy.ts          # 打字策略（user/system/heartbeat）
│   ├── queue.ts                  # 佇列指令 + followup run
│   ├── queue-policy.ts           # 活躍 run queue action
│   ├── route-reply.ts            # 跨頻道回覆路由
│   ├── normalize-reply.ts        # 回覆正規化（strip tokens, prefix）
│   ├── response-prefix-template.ts  # 回覆前綴模板
│   ├── abort.ts                  # Fast abort (stop/cancel)
│   ├── abort-cutoff.ts           # Abort cutoff 邏輯
│   ├── history.ts                # 群組聊天歷史構建
│   ├── groups.ts                 # 群組上下文 + intro prompt
│   ├── mentions.ts               # @mention 解析
│   ├── model-selection.ts        # createModelSelectionState()
│   ├── followup-runner.ts        # Followup 自動執行
│   ├── memory-flush.ts           # Memory flush after run
│   ├── dispatch-acp.ts           # ACP (Agent Client Protocol) dispatch
│   ├── dispatch-acp-delivery.ts  # ACP 投遞
│   ├── exec.ts                   # /exec 指令
│   ├── export-html/              # Session export to HTML
│   ├── commands-acp/             # ACP 命令子模組
│   ├── commands-subagents/       # 子代理命令
│   └── queue/                    # 佇列實作
│
├── test-helpers/                 # 測試輔助
└── *.test.ts                    # 測試檔案

src/web/auto-reply/              # WhatsApp 頻道 auto-reply 實作
├── monitor.ts                   # monitorWebChannel() — WhatsApp 連線主迴圈
├── monitor/
│   ├── on-message.ts            # createWebOnMessageHandler — 訊息路由
│   ├── process-message.ts       # processMessage() — 訊息處理 + dispatch
│   ├── ack-reaction.ts          # 已讀確認 reaction
│   ├── broadcast.ts             # 廣播訊息
│   ├── commands.ts              # WhatsApp 特有命令
│   ├── echo.ts                  # 回音追蹤（避免處理自己的訊息）
│   ├── group-activation.ts      # 群組觸發
│   ├── group-gating.ts          # 群組策略（mention/always/allowlist）
│   ├── group-members.ts         # 群組成員解析
│   ├── last-route.ts            # 最後路由記錄
│   ├── message-line.ts          # Inbound 訊息格式化
│   └── peer.ts                  # Peer ID 解析
├── deliver-reply.ts             # deliverWebReply — 出站投遞
├── heartbeat-runner.ts          # runWebHeartbeatOnce — Heartbeat 執行
├── session-snapshot.ts          # Session 快照
├── mentions.ts                  # WhatsApp @mention 配置
├── loggers.ts                   # Channel 日誌
├── types.ts                     # WebInboundMsg, WebChannelStatus
├── constants.ts                 # 常量
└── util.ts                      # 工具函式
```

---

## 核心資料流（逐步展開）

### 1. 入站：Channel → Auto-Reply

每個頻道的 monitor 負責格式化 `MsgContext` 並呼叫 auto-reply 入口：

| 頻道 | 呼叫方式 | 入口函式 |
|------|---------|---------|
| WhatsApp | `processMessage()` → `dispatchReplyWithBufferedBlockDispatcher()` | `getReplyFromConfig()` |
| Discord | `message-handler.process.ts` → `dispatchInboundMessage()` | `dispatchReplyFromConfig()` |
| Telegram | bot handler → `getReplyFromConfig()` 直接呼叫 | `getReplyFromConfig()` |
| Slack | socket mode handler | `dispatchInboundMessage()` |
| Web UI | gateway server chat event | `dispatchReplyFromConfig()` |

> C# 對照：各頻道 = IHostedService，MsgContext = HttpContext，auto-reply = middleware pipeline。

### 2. MsgContext — 統一訊息上下文

`templating.ts` 定義了 `MsgContext`（150+ 欄位），是所有頻道的統一訊息格式：

```typescript
type MsgContext = {
  Body?: string;              // 原始訊息
  BodyForAgent?: string;      // 給 Agent 的格式化 body（含 envelope/history）
  BodyForCommands?: string;   // 命令解析用 body
  RawBody?: string;           // 未處理原始 body
  CommandBody?: string;       // 命令偵測 body
  From?: string;              // 發送者
  To?: string;                // 接收者
  SessionKey?: string;        // Session 標識
  Provider?: string;          // 頻道 provider
  ChatType?: string;          // direct | group
  // ... 150+ 更多欄位
};
```

> C# 對照：相當於一個巨大的 `record` / `DTO`，各頻道填入各自的欄位。

### 3. Envelope — 訊息格式化

`envelope.ts` 的 `formatAgentEnvelope()` 將 inbound 訊息包裝成 Agent 看到的格式：

```
[WhatsApp +1234567890 Wed 2026-03-12 10:30:00] Hello, how are you?
```

格式：`[Channel From Timestamp] Body`

時區支援：`local` | `utc` | `user` | IANA timezone string。
經過時間（elapsed）：當有 `previousTimestamp` 時，顯示 `+5m` 之類的經過時間。

### 4. 命令系統

#### 4.1 命令定義（`commands-registry.data.ts`）

40+ 內建聊天命令，每個包含：
- `key`：內部鍵（如 `"model"`）
- `nativeName`：Native 命令名（如 `"model"`）
- `textAliases`：文字別名（如 `["/model"]`）
- `scope`：`"text"` | `"native"` | `"both"`
- `category`：`"status"` | `"management"` | `"tools"` | `"media"` | `"docks"`

核心命令清單：

| 命令 | 功能 |
|------|------|
| `/help`, `/commands` | 顯示幫助/命令列表 |
| `/status` | 目前狀態（模型、session、uptime） |
| `/new`, `/reset` | 新建/重設 session |
| `/model` | 切換模型 |
| `/think` | 設定 thinking level |
| `/verbose` | 設定 verbose level |
| `/context` | 顯示 context 使用狀況 |
| `/export-session` | 匯出 session 為 HTML |
| `/tts` | 控制 text-to-speech |
| `/allowlist` | 管理允許清單 |
| `/approve` | 批准/拒絕 exec 請求 |
| `/skill` | 執行 skill |
| `/activation` | 群組觸發模式 |
| `/send` | 設定 send policy |
| `/stop`, `/cancel` | 中止當前 run |
| `/compact` | 壓縮 session |
| `/dock-*` | 切換回覆頻道 |

#### 4.2 命令偵測流程

```
hasControlCommand(text)
  → normalizeCommandBody(text) — 正規化（strip prefix, lowercase）
  → 比對 command.textAliases
  → 若有 acceptsArgs，檢查後續是否有空白

isControlCommandMessage(text)
  → hasControlCommand(text) || isAbortTrigger(text)

hasInlineCommandTokens(text)
  → /(?:^|\s)[/!][a-z]/i — 粗略偵測內聯指令
```

#### 4.3 命令權限

`command-auth.ts` 的 `resolveCommandAuthorization()` 決定使用者是否有權限執行命令。
各頻道各自實作權限解析（如 WhatsApp 的 `resolveWhatsAppCommandAuthorized()`）。

### 5. getReplyFromConfig() — 回覆主入口

這是整個 auto-reply 的核心函式，在 `reply/get-reply.ts`（~400 行）。流程：

```
getReplyFromConfig(ctx, opts, cfgOverride)
│
├─ 1. 載入 config，解析 agentId
├─ 2. 合併 skill filter（channel + agent）
├─ 3. 解析預設 model（defaultProvider, defaultModel）
├─ 4. Heartbeat model override（若 isHeartbeat）
├─ 5. 確保 agent workspace 目錄存在
├─ 6. 解析 timeout、typing interval
├─ 7. 建立 TypingController
├─ 8. finalizeInboundContext(ctx)
├─ 9. applyMediaUnderstanding() — 圖片/影片理解
├─ 10. applyLinkUnderstanding() — 連結理解
├─ 11. emitPreAgentMessageHooks()
├─ 12. resolveCommandAuthorization()
├─ 13. initSessionState() ★ — Session 初始化
│     ├─ 載入 session store
│     ├─ 解析 session key（含群組 key）
│     ├─ 偵測 reset trigger (/new, /reset, 自動 freshness)
│     ├─ Session fork（若有 parent）
│     └─ 回傳完整 session 狀態
├─ 14. applyResetModelOverride() — reset 時的 model override
├─ 15. resolveChannelModelOverride() — 頻道級 model override
├─ 16. resolveReplyDirectives() ★ — 解析所有指令
│     ├─ 解析 /model, /think, /verbose, /elevated, /reasoning
│     ├─ 解析 /queue, /exec
│     ├─ 構建 command context
│     └─ 若指令本身就是回覆 → return early
├─ 17. handleInlineActions() — 處理內聯動作
│     ├─ /status, /help, /commands 等即時命令
│     └─ 若命令產生回覆 → return early
├─ 18. stageSandboxMedia() — 媒體存入 sandbox
└─ 19. runPreparedReply() ★ — 執行 Agent run
```

### 6. runPreparedReply() — Agent 執行

`reply/get-reply-run.ts`，接手 `getReplyFromConfig` 準備好的所有參數：

```
runPreparedReply()
│
├─ 1. 解析 typing mode + policy
├─ 2. 構建 group context（群組聊天 intro/history）
├─ 3. 準備 command body（含 envelope、inbound meta）
├─ 4. 群組觸發檢查（mention 模式需 @mention）
├─ 5. 解析 queue settings（steer/followup/drop policy）
├─ 6. 檢查活躍 run（resolveActiveRunQueueAction）
│     → abort / queue / skip / run
├─ 7. Reset session notice（若 /new 且跨頻道）
└─ 8. runReplyAgent() ★ — 呼叫 Agent
      │
      ├─ runAgentTurnWithFallback()
      │   ├─ 構建 embedded run params（system prompt, context）
      │   ├─ runWithModelFallback() — 含 provider fallback
      │   │   └─ runEmbeddedPiAgent() ← 這是 Agent 引擎入口
      │   ├─ HEARTBEAT_OK / NO_REPLY token 處理
      │   └─ Block streaming pipeline
      │
      ├─ buildReplyPayloads() — 格式化回覆
      ├─ persistRunSessionUsage() — 記錄 usage
      ├─ runMemoryFlushIfNeeded() — 記憶體寫入
      └─ finalizeWithFollowup() — 排程 followup run
```

### 7. Reply Dispatcher — 出站序列化

`reply/reply-dispatcher.ts` 的 `ReplyDispatcher` 是回覆投遞的核心抽象：

```typescript
type ReplyDispatcher = {
  sendToolResult: (payload) => boolean;   // 工具結果（中間回覆）
  sendBlockReply: (payload) => boolean;   // Block streaming 片段
  sendFinalReply: (payload) => boolean;   // 最終回覆
  waitForIdle: () => Promise<void>;       // 等待所有投遞完成
  markComplete: () => void;               // 標記不再有新回覆
};
```

設計要點：
- **序列化 queue**：所有回覆按 tool → block → final 順序排隊
- **Pending 計數器**：初始 pending=1 作為 reservation，防止 gateway 過早重啟
- **Human delay**：block 回覆之間可配置隨機延遲（800-2500ms），模擬人類打字節奏
- **Global registry**：透過 `dispatcher-registry.ts` 全域追蹤活躍 dispatcher

> C# 對照：相當於 Channel<ReplyPayload> + BackgroundService，保證順序投遞。

### 8. dispatchReplyFromConfig() — 完整 Dispatch

`reply/dispatch-from-config.ts`（~590 行），是 `getReplyFromConfig` 的包裝層：

```
dispatchReplyFromConfig()
│
├─ 1. Diagnostic logging（channel, chatId, messageId）
├─ 2. shouldSkipDuplicateInbound() — 入站去重
├─ 3. Plugin hooks（message_received）
├─ 4. Internal hooks（HOOK.md discovery）
├─ 5. Cross-channel routing 判斷
│     → originatingChannel ≠ currentSurface → 路由到原始頻道
├─ 6. tryFastAbortFromMessage() — 快速中止
├─ 7. Send policy 檢查（deny → 靜默丟棄）
├─ 8. tryDispatchAcpReply() — ACP 優先處理
├─ 9. 呼叫 getReplyFromConfig()
│     ├─ onToolResult → dispatcher.sendToolResult / routeReply
│     ├─ onBlockReply → dispatcher.sendBlockReply / routeReply
│     └─ 回傳 ReplyPayload[]
├─ 10. TTS 處理（maybeApplyTtsToPayload）
├─ 11. Final reply → dispatcher.sendFinalReply / routeReply
└─ 12. Accumulated block TTS（streaming 完但無 final 時補 TTS）
```

---

## 關鍵子系統

### Session 管理（`reply/session.ts`）

`initSessionState()` 是 session 管理的核心，負責：

| 步驟 | 說明 |
|------|------|
| 載入 store | `loadSessionStore(storePath)` — JSON 檔案 |
| 解析 key | `resolveSessionKey()` — 含群組 key 解析 |
| Reset 偵測 | 檢查 `/new`、`/reset`、自動 freshness（configurable） |
| Session fork | `forkSessionFromParent()` — 從 parent session 繼承 context |
| 狀態回傳 | `SessionInitResult`（sessionCtx, sessionEntry, isNewSession, etc.） |

Reset triggers：`DEFAULT_RESET_TRIGGERS`（`/new`、`/reset`），可透過 config 擴展。

### Heartbeat 系統

Heartbeat 是定時檢查機制，每 30 分鐘（預設）觸發一次 Agent run：

```
resolveHeartbeatPrompt(raw?)
  → 預設："Read HEARTBEAT.md if it exists..."

stripHeartbeatToken(raw, opts)
  → 偵測 HEARTBEAT_OK token
  → 短回覆（≤300 chars）= 靜默丟棄
  → 長回覆 = 剝離 token 後投遞

isHeartbeatContentEffectivelyEmpty(content)
  → 檢查 HEARTBEAT.md 是否有實際任務
  → 空 = 跳過 API 呼叫
```

關鍵常量：
- `HEARTBEAT_TOKEN` = `"HEARTBEAT_OK"`
- `SILENT_REPLY_TOKEN` = `"NO_REPLY"`
- `DEFAULT_HEARTBEAT_EVERY` = `"30m"`
- `DEFAULT_HEARTBEAT_ACK_MAX_CHARS` = `300`

### Block Streaming

Block streaming 讓 Agent 的回覆分塊投遞，不等完整回覆：

```
Block Streaming Config:
  minChars: 800 (default)
  maxChars: 1200 (default)
  breakPreference: "paragraph" | "newline" | "sentence"
  flushOnParagraph: boolean

Block Reply Pipeline:
  create → coalesce (idle 1s) → chunk → deliver
```

可在以下層級配置：
- Global: `agents.defaults.blockStreaming`
- Per-channel: `channels.{channel}.blockStreaming`
- Per-account: `channels.{channel}.accounts.{id}.blockStreamingCoalesce`

### Inbound Debounce

防抖機制避免快速連續訊息觸發多次 Agent run：

```
resolveInboundDebounceMs({ cfg, channel })
  → 優先級：override > byChannel > base > 0

createInboundDebouncer({ debounceMs, buildKey, shouldDebounce, onFlush })
  → 按 key 分組（通常是 conversationId）
  → 同 key 的訊息合併後一次 flush
  → 媒體/位置/引用訊息不 debounce
  → 命令不 debounce
```

### Text Chunking

出站文字分塊，適應各頻道限制：

| 配置 | 預設值 | 說明 |
|------|--------|------|
| `textChunkLimit` | 4000 chars | 單塊最大長度 |
| `chunkMode` | `"length"` | `length` = 超長才切；`newline` = 段落邊界切 |

分塊保護 Markdown fence（````）不被切斷（`parseFenceSpans`）。

### Typing 指示器

三層控制：

1. **TypingController**（`reply/typing.ts`）：管理打字狀態生命週期
2. **TypingPolicy**（`reply/typing-policy.ts`）：按 run 類型決定策略
   - `auto` | `user_message` | `system_event` | `internal_webchat` | `heartbeat`
3. **TypingMode**（`reply/typing-mode.ts`）：按頻道/群組/mention 決定模式

### Model Selection 與 Fallback

```
resolveDefaultModel(cfg, agentId)
  → Provider + Model + AliasIndex

Heartbeat model override (per-agent)
  → agentCfg.heartbeat.model

Channel model override
  → resolveChannelModelOverride(cfg, channel, groupId)

Session model override
  → sessionEntry.modelOverride / providerOverride

Per-message /model directive
  → extractModelDirective(body)

Fallback chain
  → runWithModelFallback() → 自動切換到備用模型
  → buildFallbackNotice() → 通知使用者
```

### ACP (Agent Client Protocol)

`dispatch-acp.ts` 提供外部 Agent Client 直接接入的路徑：

```
tryDispatchAcpReply()
  → 檢查 session 是否有 ACP 配置
  → 若有 → 透過 ACP 投遞（bypass 一般 dispatch）
  → 若無 → fall through 到一般流程
```

---

## web/auto-reply/monitor.ts — WhatsApp 連線主迴圈

### 角色

`monitorWebChannel()` 是 WhatsApp（Web）頻道的 **連線生命週期管理器**，相當於一個永運行的 while(true) 迴圈。

> C# 對照：`BackgroundService.ExecuteAsync` + `IHostApplicationLifetime` — 含 reconnect、watchdog、graceful shutdown。

### 流程

```
monitorWebChannel(verbose, listenerFactory, keepAlive, replyResolver, runtime, abortSignal)
│
├─ 初始化
│   ├─ 載入 config + WhatsApp account
│   ├─ 解析 heartbeat/reconnect/mention config
│   ├─ 建立 echo tracker（防止處理自己發出的訊息）
│   └─ 建立 onMessage handler
│
├─ while(true) — 連線迴圈
│   ├─ 1. 建立連線（monitorWebInbox → listener）
│   ├─ 2. 註冊系統事件（"WhatsApp gateway connected"）
│   ├─ 3. 設定 heartbeat interval
│   ├─ 4. 設定 watchdog timer（30min 無訊息 → 強制重連）
│   ├─ 5. await listener.onClose — 等待斷線
│   │
│   ├─ 斷線處理
│   │   ├─ loggedOut → 停止（要求使用者重新登入）
│   │   ├─ status 440（session conflict）→ 停止
│   │   ├─ max attempts reached → 停止
│   │   └─ 其他 → exponential backoff → 重連
│   │
│   └─ 重連 backoff
│       └─ computeBackoff(policy, attempts) → sleep → 回到迴圈頂部
│
└─ 清理（SIGINT / abort → closeListener）
```

### 關鍵常量

| 常量 | 值 | 說明 |
|------|-----|------|
| `MESSAGE_TIMEOUT_MS` | 30 min | 無訊息超時 → 強制重連 |
| `WATCHDOG_CHECK_MS` | 1 min | Watchdog 檢查間隔 |
| heartbeat interval | configurable | 預設 5 min |

### onMessage 處理鏈

```
onMessage(msg: WebInboundMsg)
  → resolveAgentRoute() — 路由到 agent
  → applyGroupGating() — 群組策略過濾
  → processMessage()
      → buildInboundLine() — 格式化入站訊息
      → buildHistoryContext() — 群組歷史
      → resolveWhatsAppCommandAuthorized() — 命令權限
      → dispatchReplyWithBufferedBlockDispatcher()
          → getReplyFromConfig() — 進入 auto-reply 主流程
      → deliverWebReply() — 投遞回覆到 WhatsApp
```

---

## auto-reply 與 Agent 引擎的關係

```
Auto-Reply                         Agent Engine (src/agents/)
─────────                          ─────────────────────────

runReplyAgent()
  │
  ├─ 構建 params (system prompt,
  │  context, tools, skills)
  │
  ├─ runAgentTurnWithFallback()
  │   │
  │   ├─ buildEmbeddedRunParams()  → resolves model, auth, context
  │   │
  │   ├─ runWithModelFallback()    → model-fallback.ts
  │   │   │
  │   │   └─ runEmbeddedPiAgent() ──→ pi-embedded.ts
  │   │       │                        │
  │   │       │                        ├─ buildAgentSystemPrompt()
  │   │       │                        ├─ resolveModel()
  │   │       │                        ├─ ensureAuthProfileStore()
  │   │       │                        ├─ streamSimple() → LLM call
  │   │       │                        ├─ tool_calls loop
  │   │       │                        └─ compact() on overflow
  │   │       │
  │   │       ←── result + usage ──────┘
  │   │
  │   ├─ HEARTBEAT_OK handling
  │   ├─ NO_REPLY handling
  │   └─ Block streaming
  │
  ├─ buildReplyPayloads()
  ├─ persistRunSessionUsage()
  ├─ runMemoryFlushIfNeeded()
  └─ finalizeWithFollowup()
```

> C# 對照：Auto-Reply = Controller + Application Service，Agent Engine = Domain Service + Repository。Auto-Reply 負責「如何呼叫」，Agent Engine 負責「如何執行」。

---

## 邊界條件與已知陷阱

### 1. NO_REPLY Token 處理
- Agent 回覆 `"NO_REPLY"` 表示不需要回覆（群組中常見）
- `isSilentReplyText()` 只匹配 **完全** 是 NO_REPLY 的文字，避免誤殺含 NO_REPLY 的真實回覆 (#19537)
- `isSilentReplyPrefixText()` 偵測 streaming 中的 `"NO"` 前綴（只在全大寫時觸發）

### 2. HEARTBEAT_OK 處理
- Heartbeat 回覆含 `HEARTBEAT_OK` 且 ≤300 chars → 靜默丟棄
- `stripHeartbeatToken()` 可處理 HTML/Markdown 包裹的 token（`<b>HEARTBEAT_OK</b>`）
- Token 在文字邊緣（開頭/結尾）才剝離，中間出現不處理

### 3. Cross-Channel Routing
- 當 `originatingChannel ≠ currentSurface` 時，回覆路由到 originating channel
- 例：Telegram 訊息透過 shared session 處理 → 回覆送回 Telegram，不送 Slack
- `isInternalWebchatTurn` 特殊處理 webchat 不跨頻道路由

### 4. Dispatcher Reservation
- `pending` 初始 = 1 作為 reservation，防止 gateway 在 Agent run 完成前重啟
- `markComplete()` 在所有 microtask 結束後才清除 reservation
- 若未 `markComplete()` 就丟棄 dispatcher → 永遠不觸發 `onIdle`

### 5. Inbound 去重
- `shouldSkipDuplicateInbound()` 防止同一訊息被處理兩次
- 特別重要於 WhatsApp（webhook 可能重發）

### 6. Group Activation
- 群組預設 `mention` 模式 → 只有 @mention 時才觸發
- `always` 模式 → 每條訊息都觸發
- 切換命令：`/activation mention` 或 `/activation always`

### 7. Send Policy
- Session 級別的 `send-policy` 可設為 `deny` → 靜默丟棄所有回覆
- 但命令（如 `/status`）仍然可以執行（`bypassAcpForCommand`）

### 8. Block Streaming 與 Final Reply
- Block streaming 成功時可能沒有 final reply payload
- 此時 accumulated block text 會用於 TTS 合成（`dispatch-from-config.ts` L525-578）

### 9. Echo Protection（WhatsApp）
- `createEchoTracker()` 記錄自己發出的訊息
- 收到自己的訊息時跳過處理，避免無限迴圈

### 10. Watchdog Timer（WhatsApp）
- 30 分鐘無訊息 → 強制重連
- 非 retryable status（440 = session conflict）→ 停止監聽

---

## 關鍵常量速查

| 常量 | 值 | 位置 |
|------|-----|------|
| `SILENT_REPLY_TOKEN` | `"NO_REPLY"` | tokens.ts |
| `HEARTBEAT_TOKEN` | `"HEARTBEAT_OK"` | tokens.ts |
| `DEFAULT_HEARTBEAT_EVERY` | `"30m"` | heartbeat.ts |
| `DEFAULT_HEARTBEAT_ACK_MAX_CHARS` | `300` | heartbeat.ts |
| `DEFAULT_CHUNK_LIMIT` | `4000` | chunk.ts |
| `DEFAULT_BLOCK_STREAM_MIN` | `800` | block-streaming.ts |
| `DEFAULT_BLOCK_STREAM_MAX` | `1200` | block-streaming.ts |
| `DEFAULT_BLOCK_STREAM_COALESCE_IDLE_MS` | `1000` | block-streaming.ts |
| `DEFAULT_HUMAN_DELAY_MIN_MS` | `800` | reply-dispatcher.ts |
| `DEFAULT_HUMAN_DELAY_MAX_MS` | `2500` | reply-dispatcher.ts |
| `BLOCK_REPLY_SEND_TIMEOUT_MS` | `15000` | agent-runner.ts |
| `MESSAGE_TIMEOUT_MS` | `30 * 60 * 1000` | monitor.ts |
| `WATCHDOG_CHECK_MS` | `60 * 1000` | monitor.ts |

---

## C# 概念對照總結

| Auto-Reply 概念 | C# 等價 |
|-----------------|---------|
| `MsgContext` | `HttpContext` / 巨大 DTO |
| `getReplyFromConfig()` | Controller Action + Application Service |
| `ReplyDispatcher` | `Channel<T>` + BackgroundService |
| `dispatch-from-config.ts` | ActionFilter + Middleware |
| `commands-registry` | `ICommandHandler<T>` registry |
| `initSessionState()` | EF Core DbContext + Unit of Work |
| `TypingController` | `CancellationTokenSource` + Timer |
| `monitorWebChannel()` | `BackgroundService.ExecuteAsync` + reconnect loop |
| `createInboundDebouncer()` | `System.Reactive.Throttle` / `Rx.Buffer` |
| `ReplyPayload` | `IActionResult` / response DTO |
| `envelope.ts` | `IOutputFormatter` |
| `block-streaming` | `IAsyncEnumerable<T>` streaming response |
| Heartbeat | `IHealthCheck` + periodic probe |
| `runEmbeddedPiAgent()` | `ISender.Send()` (MediatR) |
