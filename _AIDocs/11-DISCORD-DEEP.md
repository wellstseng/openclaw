# Discord 完整實作深入

> Phase 3-1 | 掃描範圍：`src/discord/` 170 files (~41,600 lines) + `extensions/discord/` 5 files + `src/config/types.discord.ts`
> 更新：2026-03-13

---

## 目錄

1. [架構鳥瞰](#1-架構鳥瞰)
2. [Core Layer：Client + API + Token](#2-core-layerclient--api--token)
3. [Account 多帳號管理](#3-account-多帳號管理)
4. [Gateway + Socket 連線層](#4-gateway--socket-連線層)
5. [Provider 啟動序列](#5-provider-啟動序列)
6. [Inbound Pipeline：完整訊息處理流程](#6-inbound-pipeline完整訊息處理流程)
7. [Preflight 40+ 驗證閘門](#7-preflight-40-驗證閘門)
8. [Worker Queue + Message Processing](#8-worker-queue--message-processing)
9. [Reply Delivery + Draft Streaming](#9-reply-delivery--draft-streaming)
10. [Send 子系統：完整出站管線](#10-send-子系統完整出站管線)
11. [Components + UI 互動元件](#11-components--ui-互動元件)
12. [Thread Bindings 系統](#12-thread-bindings-系統)
13. [Voice 語音子系統](#13-voice-語音子系統)
14. [Slash Commands + Native Commands](#14-slash-commands--native-commands)
15. [Auto-Presence + Model Picker](#15-auto-presence--model-picker)
16. [Allow List + Access Control](#16-allow-list--access-control)
17. [Extension Plugin 架構](#17-extension-plugin-架構)
18. [Config Schema 完整結構](#18-config-schema-完整結構)
19. [PluralKit + 特殊整合](#19-pluralkit--特殊整合)
20. [邊界條件與陷阱](#20-邊界條件與陷阱)
21. [關鍵常量速查](#21-關鍵常量速查)
22. [C# 概念對照](#22-c-概念對照)

---

## 1. 架構鳥瞰

### 檔案分佈

| 區域 | 路徑 | 檔案數 | 行數 | 職責 |
|------|------|--------|------|------|
| Core | `src/discord/*.ts` | ~35 | ~6,000 | Client、API、Token、Send、Chunk、Account |
| Monitor | `src/discord/monitor/` | ~90 | ~32,000 | Inbound pipeline、Preflight、Routing、Thread Bindings |
| Voice | `src/discord/voice/` | 4 | ~1,700 | Voice channel 管理 + 指令 |
| Extension | `extensions/discord/` | 5 | ~800 | Channel plugin + Subagent hooks |
| Config | `src/config/types.discord.ts` | 1 | ~350 | 完整 config schema |

### 依賴圖

```
┌─ External Libraries
│  ├─ @buape/carbon ─── Client, RequestClient, GatewayPlugin, VoicePlugin, Commands
│  ├─ discord-api-types ─── REST/Gateway/Voice 型別
│  ├─ @discordjs/voice ─── Voice connection + audio player
│  ├─ ws ─── WebSocket client
│  └─ undici + https-proxy-agent ─── Proxy 支援
│
├─ Core Discord Layer
│  ├─ client.ts → REST client 初始化
│  ├─ api.ts → fetchDiscord<T>() 泛型 API 呼叫
│  ├─ token.ts → Token 解析/正規化
│  ├─ probe.ts → 連線健康檢查（/users/@me）
│  ├─ accounts.ts → 多帳號解析
│  └─ guilds.ts / audit.ts / directory-*.ts → Guild 與使用者目錄
│
├─ Monitor Layer（即時處理）
│  ├─ provider.ts ─── 啟動序列 + 生命週期（主入口）
│  ├─ listeners.ts ─── Gateway 事件監聽
│  ├─ message-handler.ts ─── Debounce + 路由
│  ├─ message-handler.preflight.ts ─── 40+ 驗證閘門
│  ├─ message-handler.process.ts ─── Agent dispatch + Draft streaming
│  ├─ inbound-worker.ts ─── Keyed async queue
│  ├─ reply-delivery.ts ─── 出站 chunk + retry
│  ├─ thread-bindings.*.ts ─── Thread 綁定系統（8 個檔案）
│  └─ allow-list.ts / route-resolution.ts / sender-identity.ts
│
├─ Send Layer（出站操作）
│  ├─ send.outbound.ts → 主出站邏輯
│  ├─ send.shared.ts → Chunk + Payload 建構
│  ├─ send.messages.ts / send.guild.ts / send.channels.ts
│  ├─ send.permissions.ts → Permission bitfield 計算
│  └─ send.components.ts → Component 互動元件
│
└─ Extension Layer
   ├─ extensions/discord/src/channel.ts → Channel plugin 實作
   └─ extensions/discord/src/subagent-hooks.ts → Subagent thread 整合
```

### 完整資料流概覽

```
Discord Gateway WebSocket
  ↓ MessageCreate / ReactionAdd / PresenceUpdate / ThreadUpdate
listeners.ts（事件分發）
  ↓ fire-and-forget
message-handler.ts（Debounce + 批次合併）
  ↓ per-channel:author key
message-handler.preflight.ts（40+ 驗證閘門）
  ↓ DiscordMessagePreflightContext
inbound-job.ts（Payload/Runtime 分離 + 序列化）
  ↓
inbound-worker.ts（KeyedAsyncQueue per-session 串行）
  ↓ 30min timeout
message-handler.process.ts（Media/History/Context/Draft → Agent dispatch）
  ↓ dispatchInboundMessage()
reply-delivery.ts（Chunk + Retry + Webhook 人格 → Discord API）
  ↓
Discord REST API → 使用者看到回覆
```

---

## 2. Core Layer：Client + API + Token

### 核心檔案矩陣

| 檔案 | 行數 | 職責 |
|------|------|------|
| `client.ts` | ~120 | `createDiscordClient()` → REST client（@buape/carbon RequestClient） |
| `api.ts` | ~180 | `fetchDiscord<T>()` → 泛型 Discord REST API 呼叫 + retry |
| `token.ts` | ~150 | Token 解析、正規化、Application ID 萃取 |
| `probe.ts` | ~200 | `probeDiscord()` → 健康檢查 + intent 偵測 |

### Token 解析流程

```
resolveDiscordToken(cfg, accountId)
  ├─ 1. 顯式傳入 → 直接使用
  ├─ 2. 帳號專屬 config → cfg.channels.discord.accounts[id].token
  ├─ 3. 全域 config → cfg.channels.discord.token
  └─ 4. 環境變數 → DISCORD_BOT_TOKEN（僅 default 帳號）

normalizeDiscordToken(raw)
  ├─ 去除 "Bot " 前綴
  ├─ 驗證格式：base64 ID 段解碼為 safe integer
  └─ 回傳清理後的 token string
```

### Application ID 萃取

```
parseApplicationIdFromToken(token)
  → base64 decode token 第一段 → 數字 ID
  → 若失敗 → fetchDiscordApplicationId() 走 REST /users/@me fallback
```

### REST API 呼叫模式

```typescript
fetchDiscord<T>(url, opts): Promise<T>
  ├─ Headers: Authorization: Bot {token}
  ├─ Base URL: https://discord.com/api/v10
  ├─ Retry: RetryRunner（3 次, 500ms-30s, 10% jitter）
  ├─ 429 Rate Limit → 讀取 Retry-After header → 等待後重試
  └─ Credential errors vs. Transport errors 分開處理
```

---

## 3. Account 多帳號管理

### 核心檔案

| 檔案 | 行數 | 職責 |
|------|------|------|
| `accounts.ts` | ~130 | Account 解析 + 啟用判定 |
| `account-inspect.ts` | ~100 | Token 狀態檢查（available / configured_unavailable / missing） |

### 架構

```
Config 結構：
  cfg.channels.discord = {
    ...DiscordAccountConfig        ← 基礎（default 帳號）
    accounts: {
      "account-2": { ... },       ← 覆寫特定欄位
      "account-3": { ... },
    }
  }

resolveDiscordAccount(cfg, accountId)
  ├─ 基礎 config + 帳號覆寫合併
  ├─ enabled = 基礎.enabled AND 帳號.enabled（兩層都需 true）
  ├─ token source 追蹤："env" | "config" | "none"
  └─ 回傳 ResolvedDiscordAccount

listEnabledDiscordAccounts(cfg)
  → 列舉所有 enabled=true 的帳號
```

### Token 來源追蹤

| 來源 | 條件 | 優先序 |
|------|------|--------|
| `"config"` | 帳號 or 全域 config 中有 token | 最高 |
| `"env"` | `DISCORD_BOT_TOKEN` 環境變數（僅 default 帳號） | 中 |
| `"none"` | 都沒有 | 最低（disabled） |

---

## 4. Gateway + Socket 連線層

### 核心檔案矩陣

| 檔案 | 行數 | 職責 |
|------|------|------|
| `monitor/gateway-plugin.ts` | ~200 | GatewayPlugin 工廠 + Intent 計算 + Proxy 支援 |
| `monitor/gateway-registry.ts` | ~50 | 活躍 Gateway 實例 registry（per account） |
| `monitor/gateway-error-guard.ts` | ~120 | 過早斷線偵測，防止假重連 |
| `monitor.gateway.ts` | ~80 | Gateway EventEmitter 萃取 + 等待停止 |
| `gateway-logging.ts` | ~100 | Gateway 事件日誌（debug/warning/metrics） |

### Intent 計算

```
resolveDiscordGatewayIntents(config)
  ├─ 永遠啟用：Guilds | GuildMessages | MessageContent | DirectMessages
  │            | MessageReactions | GuildVoiceStates
  ├─ config.intents.presence = true → + GuildPresences（需 Portal opt-in）
  └─ config.intents.guildMembers = true → + GuildMembers（需 Portal opt-in）
```

### Proxy 支援

```
ProxyGatewayPlugin extends GatewayPlugin
  ├─ WebSocket 連線：undici ProxyAgent → 自訂 WebSocket 建立
  ├─ REST 呼叫：https-proxy-agent → 代理 HTTP 請求
  └─ Gateway Info：透過 proxy 取得 gateway URL + session 限制
```

### 重連策略

- 最大重連次數：50（auto backoff）
- Gateway registry：per-accountId 儲存活躍 gateway → Agent tools 可存取 WebSocket

---

## 5. Provider 啟動序列

### 主入口：`monitor/provider.ts`（~800 lines）

```
monitorDiscordProvider(opts)
  │
  ├─ 1. Config 解析（L290-410）
  │  ├─ 載入 config, 解析 account
  │  ├─ 正規化 token
  │  ├─ 解析 allowlist（guilds/channels/users）
  │  ├─ 萃取 thread bindings 設定（account > root > session）
  │  └─ 計算 effective group policy + fallback
  │
  ├─ 2. 驗證 + Logger（L407-438）
  │  ├─ REST 取得 Application ID
  │  ├─ 驗證 command 數量（Discord 上限 100）
  │  ├─ 預載入 skill commands（native enabled 時）
  │  └─ 超出 100 → 自動裁剪
  │
  ├─ 3. Manager 初始化（L439-480）
  │  ├─ 建立 ThreadBindingManager（或 noop 若停用）
  │  ├─ ACP thread bindings 健康檢查
  │  └─ 初始化 gateway error guard
  │
  ├─ 4. Carbon Client 建構（L483-626）
  │  ├─ 建立 native command list
  │  ├─ 建立 voice command（若啟用）
  │  ├─ 建立 exec approval handler（若啟用）
  │  ├─ 建立 agent components（buttons/selects/modals）
  │  ├─ 組裝 components list：
  │  │   ├─ Command arg fallback button
  │  │   ├─ Model picker fallback button + select
  │  │   ├─ Exec approval button
  │  │   └─ Agent components
  │  └─ 建立 Carbon Client（appId, token, commands, listeners, components, plugins）
  │
  ├─ 5. Auto-Presence Controller（L631-637）
  │  ├─ 建立控制器（即使停用也回傳 noop）
  │  └─ 啟動輪詢（若啟用）
  │
  ├─ 6. Command 部署（L640）
  │  └─ 部署 native commands 到 Discord（含 retry）
  │
  ├─ 7. Bot Identity + Voice（L642-675）
  │  ├─ 取得 bot user（用於回覆過濾）
  │  ├─ 初始化 VoiceManager（若啟用）
  │  └─ 註冊 voice ready listener
  │
  ├─ 8. Message Handler + Listeners（L677-747）
  │  ├─ 建立 message handler（完整 config）
  │  ├─ 註冊 listeners：
  │  │   ├─ MessageCreateListener
  │  │   ├─ ReactionAddListener + ReactionRemoveListener
  │  │   ├─ ThreadUpdateListener
  │  │   └─ PresenceUpdateListener（若 presence intent 啟用）
  │  └─ 記錄 login 事件
  │
  ├─ 9. Gateway Lifecycle（L756-770）
  │  └─ runDiscordGatewayLifecycle() → 阻塞直到斷線或 abort
  │
  └─ 10. Cleanup（L771-779）
     ├─ 停用 message handler
     ├─ 停止 auto-presence
     └─ 停止 thread bindings
```

---

## 6. Inbound Pipeline：完整訊息處理流程

### 七層管線

```
┌───────────────────────────────────────────────────────────┐
│ Layer 1: INGESTION（listeners.ts）                         │
│  Discord Gateway 事件 → fire-and-forget 分發              │
│  Events: MessageCreate, ReactionAdd/Remove,               │
│          PresenceUpdate, ThreadUpdate                      │
└───────────────┬───────────────────────────────────────────┘
                ↓
┌───────────────────────────────────────────────────────────┐
│ Layer 2: DEBOUNCE（message-handler.ts）                    │
│  Key: discord:{accountId}:{channelId}:{authorId}          │
│  多訊息 → 合成為 synthetic message（含 MessageSids[]）    │
│  Early bot self-filter → 防止佔用 debounce 容量           │
└───────────────┬───────────────────────────────────────────┘
                ↓
┌───────────────────────────────────────────────────────────┐
│ Layer 3: PREFLIGHT（message-handler.preflight.ts）         │
│  40+ 驗證閘門（詳見第 7 節）                               │
│  輸出：DiscordMessagePreflightContext                      │
│  30+ early-exit 路徑，每個都有 log reason                  │
└───────────────┬───────────────────────────────────────────┘
                ↓
┌───────────────────────────────────────────────────────────┐
│ Layer 4: JOB QUEUEING（inbound-job.ts）                    │
│  分離 Payload（immutable）與 Runtime（session state）       │
│  移除 message.channel 循環引用                             │
│  Queue Key 衍生自 sessionKey / baseSessionKey              │
└───────────────┬───────────────────────────────────────────┘
                ↓
┌───────────────────────────────────────────────────────────┐
│ Layer 5: WORKER QUEUE（inbound-worker.ts）                  │
│  KeyedAsyncQueue → per-session 串行執行                    │
│  Timeout: 30 分鐘（DISCORD_DEFAULT_INBOUND_WORKER_TIMEOUT）│
│  Per-job abort signals: original + lifecycle + merged      │
│  deactivate() 優雅停止                                     │
└───────────────┬───────────────────────────────────────────┘
                ↓
┌───────────────────────────────────────────────────────────┐
│ Layer 6: PROCESSING（message-handler.process.ts）           │
│  Media 解析 → Status Reaction → Context 建構 →             │
│  Auto-Thread → Draft Stream → Agent dispatch              │
│  （詳見第 8 節）                                           │
└───────────────┬───────────────────────────────────────────┘
                ↓
┌───────────────────────────────────────────────────────────┐
│ Layer 7: DELIVERY（reply-delivery.ts）                      │
│  Chunk 2000 字 → Retry 3 次 → Webhook 人格 → Discord API  │
│  （詳見第 9 節）                                           │
└───────────────────────────────────────────────────────────┘
```

---

## 7. Preflight 40+ 驗證閘門

### 閘門清單（依執行順序）

| 階段 | 閘門 | Drop 條件 |
|------|------|-----------|
| **A. Identity** | 解析 sender identity | PluralKit vs 一般使用者 |
| | 建構 sender label / display name | — |
| **B. Basic Filter** | Own message detection | 自己的訊息 → abort |
| | Missing author/channel | 缺少 → silent drop |
| | Text-based slash command | 攔截 → drop |
| | System message detection | → enqueue system event |
| **C. Channel Type** | DM / Group DM / Guild 判定 | — |
| | Thread 偵測 → parent + starter | — |
| | Guild channel metadata fetch | — |
| **D. Guild Access** | Guild allowlist 檢查 | 未列入 → drop |
| | Channel config lookup + 驗證 | — |
| | Group policy (open/allowlist/disabled) | disabled → drop |
| | Owner access check | — |
| | Role/User 層級限制 | 不符 → drop |
| **E. DM Auth** | DM policy (open/pairing/allowlist/disabled) | disabled → drop |
| | Pairing request 建立 + 通知 | 未配對 → drop |
| | Store-based allow list 檢查 | — |
| **F. Mention** | Explicit mention 偵測（user/role/@everyone） | — |
| | Audio preflight 轉錄（語音 mention 偵測） | — |
| | Mention-gating + text command bypass | 未 mention + 需 mention → drop |
| | Implicit mention（reply-to-bot） | — |
| **G. Bot Filter** | allowBots policy (off/mentions/all) | off + is bot → drop |
| | PluralKit 豁免 | PK 成員不受 allowBots=off 影響 |
| **H. Thread Binding** | 解析 active session binding | — |
| | Webhook echo 過濾 | 是 echo → drop |
| | Bound thread system message 過濾 | ⚙️🤖🧰 前綴 → drop |
| | 套用 binding route 覆寫 | — |
| **I. Route + Media** | 解析 effective agent route | — |
| | 下載 media/attachments（SSRF policy） | — |
| | 轉錄 forwarded messages | — |
| | 空內容過濾 | 無內容 → drop |
| **J. History + Context** | 建構 guild history context | — |
| | 記錄 inbound session metadata | — |
| | 準備 thread starter text（cached, 5min TTL） | — |

### Abort 安全

每個 async 操作後檢查 `isPreflightAborted(abortSignal)`，任何中途取消都安全 return null。

---

## 8. Worker Queue + Message Processing

### Inbound Worker 架構

```
KeyedAsyncQueue
  ├─ Key = sessionKey → 同一 session 的訊息串行處理
  ├─ 跨 session 並行（不同 key 可同時執行）
  ├─ Timeout: 30 min（configurable via inboundWorker.runTimeoutMs）
  ├─ Run state machine: active → paused → deactivated
  └─ deactivate() → 優雅停止所有排隊中的 job
```

### Message Processing 九階段

```
message-handler.process.ts：processDiscordInboundMessage()
  │
  ├─ A. Media + History
  │  ├─ 解析 media list（含大小限制）
  │  ├─ 解析 forwarded media
  │  └─ 每個 async 操作後檢查 abort signal
  │
  ├─ B. Status Reaction
  │  ├─ createStatusReactionController
  │  ├─ Emoji 狀態機：queued → thinking → tool → done/error
  │  ├─ Scope gating: all | direct | group-mentions | off | none
  │  └─ removeAckAfterReply flag 控制清除/保留
  │
  ├─ C. Context 建構
  │  ├─ Guild/Direct labels
  │  ├─ Forum parent detection
  │  ├─ Thread name + parent session keys
  │  ├─ Group system prompt（untrusted channel metadata）
  │  └─ Owner allowFrom extraction
  │
  ├─ D. Auto-Thread
  │  ├─ autoThread config → 自動建立 thread
  │  ├─ Race condition handling → 使用既有 thread
  │  ├─ Thread 名稱清理（移除 mentions, 截斷 100 字）
  │  └─ 建構 delivery + reply targets
  │
  ├─ E. Payload 最終組裝
  │  ├─ Envelope 格式（channel + from + timestamp）
  │  ├─ History context 合併
  │  ├─ Reply context（referencedMessage）
  │  ├─ Media payload（attachments, embeds, audio）
  │  └─ Session key 持久化
  │
  ├─ F. Draft Stream
  │  ├─ resolveDiscordPreviewStreamMode → partial | block | off
  │  ├─ partial: 即時更新（edit 既有訊息）
  │  ├─ block: EmbeddedBlockChunker 分塊感知
  │  ├─ Throttle: 1200ms 預設（防 Discord rate limit）
  │  └─ 最終回覆：≤2000 字 → edit, 否則 → 新訊息
  │
  ├─ G. Typing
  │  ├─ 回覆開始時啟動 typing indicator
  │  ├─ 最長 20 分鐘持續
  │  └─ 錯誤或完成時停止
  │
  ├─ H. Agent Dispatch
  │  ├─ dispatchInboundMessage() → 路由到 runtime agent
  │  ├─ Skill 過濾（by channel config）
  │  ├─ Block streaming control（if draft active）
  │  ├─ Partial reply streaming（update draft）
  │  ├─ Tool start notifications（update emoji）
  │  └─ Reasoning tag 剝離（防洩漏）
  │
  └─ I. 錯誤處理 + 清理
     ├─ Draft cleanup on timeout/error
     ├─ Status reaction finalization
     ├─ History cleanup on success
     └─ Run state cleanup
```

---

## 9. Reply Delivery + Draft Streaming

### reply-delivery.ts 核心流程

```
deliverDiscordReply(text, opts)
  │
  ├─ 1. Binding 解析
  │  ├─ 檢查是否有 bound thread binding
  │  ├─ Webhook fallback → 保留 persona（agent 名稱/頭像）
  │  └─ Bot sender fallback → agent avatar
  │
  ├─ 2. Chunking
  │  ├─ 2000 字硬限制
  │  ├─ Markdown table 轉換（code/embed/split 模式）
  │  ├─ Code fence 自動平衡（跨 chunk 補開/補關）
  │  └─ Reasoning italics 特殊處理（_…_ 格式）
  │
  ├─ 3. 發送
  │  ├─ replyToMode: all（每條都 reply）| first（僅首條）| off
  │  ├─ Thread binding → webhook send（含 persona）
  │  ├─ Media attachment handling（files + voice）
  │  └─ Voice message 特殊路徑（audioAsVoice flag）
  │
  ├─ 4. Retry
  │  ├─ 3 次嘗試, 1s-30s exponential backoff
  │  ├─ 429 Rate Limit → Retry-After header 解析
  │  └─ 5xx → 自動重試
  │
  └─ 5. Touch Thread
     ├─ 更新 binding last-activity timestamp
     └─ 防止 idle timeout
```

### Draft Streaming 模式

| 模式 | 行為 | 適用場景 |
|------|------|---------|
| `off` | 無串流預覽 | 預設 |
| `partial` | Edit 即時更新，1200ms throttle | 快速回應 |
| `block` | EmbeddedBlockChunker 分塊感知 | 長回覆 |
| `progress` | 進度條式更新 | 特殊場景 |

最終化邏輯：
- ≤2000 字 + 純文字 + 無錯誤/media → edit 既有 draft
- 否則 → 清除 draft + 發送新訊息

---

## 10. Send 子系統：完整出站管線

### 核心檔案矩陣

| 檔案 | 行數 | 職責 |
|------|------|------|
| `send.ts` | ~50 | Re-export 所有 send 操作 |
| `send.types.ts` | ~200 | 完整型別定義（40+ 型別） |
| `send.shared.ts` | ~400 | Chunk + Payload 建構 + 底層發送 |
| `send.outbound.ts` | ~350 | 主出站邏輯（sendMessageDiscord 等） |
| `send.messages.ts` | ~300 | CRUD 操作（read/edit/delete/pin/search/thread） |
| `send.guild.ts` | ~250 | Guild 操作（member/role/kick/ban/timeout/events） |
| `send.channels.ts` | ~200 | Channel CRUD + permission set |
| `send.components.ts` | ~250 | Component 互動元件發送 |
| `send.permissions.ts` | ~250 | Permission bitfield 計算 |
| `send.reactions.ts` | ~120 | Reaction 操作 |
| `send.emojis-stickers.ts` | ~100 | Emoji/Sticker 上傳 |

### 出站訊息流程

```
sendMessageDiscord(to, text, opts)
  ├─ parseAndResolveRecipient() → 解析 user/channel + directory lookup
  ├─ resolveChannelId() → user target → POST /users/@me/channels 建立 DM
  ├─ resolveDiscordChannelType() → 偵測 Forum/Media channel
  │
  ├─ [Forum/Media Channel]
  │  ├─ 不支援直接 POST /messages
  │  ├─ 自動建立 thread + starter message
  │  └─ 後續 chunks 作為 follow-up 訊息
  │
  └─ [Regular Channel]
     ├─ buildDiscordTextChunks() → 2000 字 + 17 行限制
     ├─ resolveDiscordSendComponents() → 元件工廠 or 靜態陣列
     ├─ resolveDiscordSendEmbeds() → 正規化 embeds（僅套用首 chunk）
     ├─ buildDiscordMessagePayload() → content + components + embeds + flags + files
     ├─ sendDiscordText() or sendDiscordMedia()
     └─ recordChannelActivity()
```

### Permission 計算

```
fetchChannelPermissionsDiscord(channelId, opts)
  ├─ 取得 channel → guild → bot member → all roles
  ├─ 套用順序（重要！）：
  │  1. @everyone role + bot roles → base permissions
  │  2. Apply @everyone overwrites
  │  3. Apply bot role overwrites
  │  4. Apply bot user-specific overwrites
  ├─ Administrator → 全部 permission
  └─ 回傳 DiscordPermissionsSummary { permissions[], raw bigint, isDm, channelType }
```

### Message Search

```
searchMessagesDiscord(guildId, query, opts)
  → GET /guilds/{guildId}/messages/search
  → 支援: content, channelIds[], authorIds[], limit
```

---

## 11. Components + UI 互動元件

### Component 型別

| 型別 | 支援項目 |
|------|---------|
| **Buttons** | primary / secondary / success / danger / link, URL, emoji, disabled, allowedUsers |
| **Select Menus** | string / user / role / mentionable / channel, options, min/max values |
| **Text Inputs** | short / long（模態視窗用） |
| **Blocks** | text, section（含 thumbnail/button accessory）, separator, actions, media-gallery, file |
| **Modals** | title, trigger label, fields（text/checkbox/radio/select/role-select/user-select） |

### Component Registry

```
TTL-based registry（預設 30 分鐘）
  ├─ registerDiscordComponentEntries(params) → 儲存互動 + 過期時間
  ├─ resolveDiscordComponentEntry(id) → 取回 + 可選消費
  └─ resolveDiscordModalEntry(id) → Modal 提交

Custom ID 格式：
  ├─ occomp:{...} → 使用者互動（buttons/selects）
  └─ ocmodal:{...} → Modal 提交

自動清理：lookup 時清除過期項目
```

### Agent Components

```
agent-components.ts
  ├─ 建立 Discord components 連結到 agents
  ├─ Wildcard 支援：role/user/channel 過濾
  └─ 預設啟用（agentComponents.enabled = true）
```

---

## 12. Thread Bindings 系統

### 概念

**Thread Binding** = Discord thread ID ↔ Agent session key 的持久關聯。啟用：
- **Session 持久性**：thread 內訊息路由到正確的 agent session
- **Agent 人格**：Webhook-based messaging，agent 專屬名稱/頭像
- **生命週期管理**：idle timeout / 封存 / max-age 自動清理
- **1:N 關係**：一個 session 可綁定多個 threads

### 完整生命週期

```
1. CREATION（bindTarget）
   ├─ 觸發：subagent_spawning hook + threadRequested=true
   ├─ 解析/建立 thread ID
   ├─ 解析 parent channel ID
   ├─ 建立/重用 webhook（per account:channel）
   └─ 儲存 ThreadBindingRecord：
      ├─ accountId, threadId, channelId
      ├─ targetSessionKey, targetKind ("subagent" | "acp")
      ├─ webhookId, webhookToken
      ├─ boundAt, lastActivityAt
      ├─ idleTimeoutMs（預設 24h）
      └─ maxAgeMs（預設 0 = 停用）

2. ACTIVE（touchThread）
   ├─ 每個 inbound message → 更新 lastActivityAt = now
   ├─ 磁碟持久化限速：15s 間隔
   └─ Session binding adapter 已註冊

3. IDLE TIMEOUT（每 120s sweep）
   ├─ lastActivityAt + idleTimeoutMs → 過期？ → unbind
   ├─ boundAt + maxAgeMs → 過期？ → unbind
   ├─ Discord API: GET /channels/{threadId}
   ├─ archived=true → unbind
   └─ 404/403 → unbind

4. CLOSE（unbindThread）
   ├─ 移除 RAM state
   ├─ 存入 RECENT_UNBOUND_WEBHOOK_ECHOES（30s 視窗）
   ├─ 發送 farewell message（webhook 或 bot fallback）
   ├─ 持久化到磁碟
   └─ Session 標記 reset（updatedAt=0, 新對話但保留歷史）
```

### 狀態儲存

| 層 | 結構 | 說明 |
|---|------|------|
| RAM | `BINDINGS_BY_THREAD_ID` Map | `accountId:threadId` → Record |
| RAM | `BINDINGS_BY_SESSION_KEY` Map | sessionKey → Set<bindingKeys> |
| RAM | `REUSABLE_WEBHOOKS_BY_ACCOUNT_CHANNEL` Map | Webhook 重用快取 |
| RAM | `RECENT_UNBOUND_WEBHOOK_ECHOES` Map | 30s echo 抑制視窗 |
| Disk | `state/discord/thread-bindings.json` | `{ version: 1, bindings: {...} }` |

**globalThis 存放**：解決 Plugin(Jiti/CJS) vs Core(ESM) 不同模組實例問題。

### Webhook 人格機制

```
resolveThreadBindingPersona({ label, agentId })
  → "☁ {label or agentId or 'agent'}"（最長 80 字）

Webhook 重用策略：
  ├─ 一個 webhook per account:channel（跨所有 threads）
  ├─ 記憶體快取 → 掃描既有 bindings → 才建立新的
  └─ Token 失效 → fallback 到 bot send
```

### Subagent 整合

```
extensions/discord/src/subagent-hooks.ts

subagent_spawning event:
  ├─ 檢查 threadRequested + threadBindings.enabled + spawnSubagentSessions
  └─ autoBindSpawnedDiscordSubagent() → 建立 binding

subagent_ended event:
  └─ unbindThreadBindingsBySessionKey() → 清理 binding + farewell

subagent_delivery_target event:
  └─ 找到匹配的 binding → 回傳 thread 作為 completion target
```

### Config 層級

```
解析順序：Account → Root Discord → Session
  cfg.channels.discord.accounts[id].threadBindings  ← 最高
  cfg.channels.discord.threadBindings               ← 中
  cfg.session.threadBindings                        ← 最低
```

---

## 13. Voice 語音子系統

### 核心檔案

| 檔案 | 行數 | 職責 |
|------|------|------|
| `voice/manager.ts` | 902 | VoiceManager 主邏輯：連線、收音、TTS、播放 |
| `voice/command.ts` | 372 | Slash command 授權 + 執行 |
| `voice-message.ts` | ~120 | 語音訊息處理（OGG/Opus 轉換 + waveform 生成） |

### 音訊參數

| 參數 | 值 |
|------|-----|
| Sample Rate | 48 kHz（Discord 標準） |
| Channels | 2（stereo） |
| Bit Depth | 16 |
| Min Segment | 0.35s |
| Silence Padding | 1s |
| Playback Ready Timeout | 15s |
| Speaking Ready Timeout | 60s |

### Voice Manager 架構

```
VoiceSessionEntry（per guild:voice-channel）
  ├─ Connection: @discordjs/voice VoiceConnection
  ├─ Audio Player: 播放佇列
  ├─ Active Speakers: Set（目前說話的人）
  ├─ Decrypt failure tracking: 3 failures / 30s → reconnect
  ├─ Speaker context cache: 60s TTL
  └─ TTS config: base + account-specific overrides

VoiceManager 功能：
  ├─ joinChannel() → 加入語音頻道（含 auto-recover）
  ├─ leaveChannel() → 離開
  ├─ Audio ingestion → 收取使用者語音
  ├─ TTS synthesis → 文字轉語音回覆
  └─ WAV header 建構（44-byte header + PCM frames）
```

### Voice Message 處理

```
sendVoiceMessageDiscord(to, audioPath, opts)
  ├─ 1. 素材化音訊（URL/path → temp file, SSRF guards）
  ├─ 2. ffprobe 取得 duration
  ├─ 3. PCM 萃取 → 256 點取樣 → waveform（0-255 振幅）
  ├─ 4. ensureOggOpus() → 若非 OGG/Opus 則 ffmpeg 轉換
  ├─ 5. 發送含 flag 8192 (IS_VOICE_MESSAGE)
  └─ 注意：語音訊息不能有 text/embeds
```

### Voice Command 授權

```
VoiceCommand.authorize()
  ├─ 驗證 guild membership
  ├─ Channel allowlist config
  ├─ Group policy (open/disabled/allowlist)
  ├─ Channel-specific user/role allowlists
  └─ 支援頻道類型：GuildVoice, GuildStageVoice（拒絕 DM/thread/text）
```

### DAVE 加密

- `voice.daveEncryption`（預設 true）：Discord E2E 語音加密
- `decryptionFailureTolerance`（預設 24）：容忍解密失敗次數
- 超過容忍 → exponential backoff → reconnect

---

## 14. Slash Commands + Native Commands

### 架構

```
monitor/native-command.ts
  ├─ 基於 @buape/carbon Command 框架
  ├─ 從 Provider 規格建構 command list
  └─ Discord 上限：100 個 commands

monitor/native-command-context.ts
  ├─ 建構 command 執行上下文
  └─ 解析 guild/channel/user 資訊

monitor/commands.ts
  ├─ 解析 ephemeral flag（預設 true）
  └─ 完整 command 處理在 provider 層

monitor/dm-command-auth.ts + dm-command-decision.ts
  ├─ DM command 特殊授權流程
  ├─ dmPolicy: open | pairing | allowlist | disabled
  └─ Pairing flow: 生成配對碼 → DM 通知 → store-based 存取
```

### Slash Command 部署

- Provider 啟動時部署到 Discord（含 retry）
- 超過 100 → 自動裁剪 skill commands
- `slashCommand.ephemeral`（預設 true）→ 回覆僅發送者可見

---

## 15. Auto-Presence + Model Picker

### Auto-Presence 系統

```
auto-presence.ts
  │
  ├─ 設定
  │  ├─ enabled: boolean（預設 false）
  │  ├─ intervalMs: 30s 輪詢（最小 5s）
  │  ├─ minUpdateIntervalMs: 15s 最小更新間隔（最小 1s）
  │  ├─ healthyText / degradedText / exhaustedText
  │  └─ degradedText + exhaustedText 支援 {reason} template
  │
  ├─ 狀態判定
  │  ├─ resolveAuthAvailability(store)
  │  │  ├─ 無 profiles → degraded
  │  │  ├─ 有可用（非 cooldown）→ healthy
  │  │  └─ 全部 cooldown → exhausted（原因：rate_limit/overloaded/billing/auth）
  │  └─ Gateway 斷線 → 強制 degraded
  │
  ├─ 狀態 → Discord Presence
  │  ├─ healthy → online + healthyText（或 base activity）
  │  ├─ degraded → idle + degradedText
  │  └─ exhausted → dnd + exhaustedText
  │
  └─ 去重
     ├─ Signature = JSON(status, afk, since, activities)
     └─ 僅在 signature 改變 或 force=true 或 minUpdateInterval 已過時更新
```

### Model Picker UI

```
model-picker.ts（941 lines）

狀態機：
  DiscordModelPickerState {
    command: "model" | "models"
    action: "open" | "provider" | "model" | "submit" | "quick" | "back" | "reset" | "cancel" | "recents"
    view: "providers" | "models" | "recents"
    userId, provider?, page, providerPage?, modelIndex?, recentSlot?
  }

三個視圖：
  1. Providers View → 列出所有 provider（分頁, 5×5=25 per page）
  2. Models View → 列出 provider 下 models + dropdown
  3. Recents View → 最近使用的 models（快速切換）

Custom ID 格式（上限 100 字元）：
  mdlpk:c={cmd};a={action};v={view};u={userId};g={page};p={provider};pp={providerPage};mi={modelIndex};rs={recentSlot}

Layout：
  ├─ "v2": Carbon Container（TextDisplay + Separator + Row）
  └─ "classic": 純文字 + components payload
```

### Model Picker Preferences

```
model-picker-preferences.ts
  ├─ 持久化使用者 model 偏好
  ├─ Per-user per-command 儲存
  └─ Fallback → default model
```

---

## 16. Allow List + Access Control

### 核心檔案

| 檔案 | 行數 | 職責 |
|------|------|------|
| `monitor/allow-list.ts` | ~300 | Guild/Channel allowlist 解析 + 匹配 |
| `monitor/provider.allowlist.ts` | ~150 | Provider-level allowlist 解析 |
| `resolve-allowlist-common.ts` | ~80 | 共用 allowlist token 解析 |

### Allowlist 格式

| 格式 | 說明 |
|------|------|
| `*` | 允許全部 |
| `123456789` | Snowflake ID 精確匹配 |
| `discord:name` | Discord 平台前綴 |
| `user:name` | 使用者前綴 |
| `pk:systemId` | PluralKit 系統 ID |
| `guild-slug#channel-slug` | Guild/Channel slug 匹配 |

### Slug 正規化

```
normalizeDiscordSlug(name)
  ├─ lowercase
  ├─ 去除 #
  ├─ 非英數字元 → -
  └─ 忽略 #0000 discriminator（向後相容）
```

### Access Control 層級

```
Group Policy 解析順序：
  account config → channel defaults → base config
  值: "open" | "allowlist" | "disabled"

Guild 層級：
  ├─ guilds config → 列入的 guild 才能互動
  ├─ channels config → per-channel 覆寫
  └─ users/roles → per-guild + per-channel allowlists

Channel 層級（覆寫 guild）：
  ├─ enabled（預設 true）
  ├─ requireMention
  ├─ users[], roles[]
  ├─ tools, toolsBySender
  ├─ skills（allowlist of skill IDs）
  └─ systemPrompt, includeThreadStarter
```

### DM Access Control

```
DM Policy:
  ├─ "open" → 任何人可 DM
  ├─ "pairing" → 需配對碼（預設）
  ├─ "allowlist" → 僅 allowFrom 列表
  └─ "disabled" → 禁止 DM

Pairing Flow:
  1. 使用者 DM bot
  2. 若 policy=pairing + 未配對
  3. 生成配對碼 → DM 通知使用者
  4. 管理員確認 → 寫入 store
  5. 後續 DM 正常路由
```

---

## 17. Extension Plugin 架構

### 註冊

```typescript
// extensions/discord/index.ts
{
  id: "discord",
  name: "Discord",
  configSchema: emptyPluginConfigSchema(),
  register(api: OpenClawPluginApi) {
    setDiscordRuntime(api.runtime);           // 設定 runtime 引用
    api.registerChannel({ plugin: discordPlugin }); // 註冊 channel plugin
    registerDiscordSubagentHooks(api);        // 註冊 subagent hooks
  }
}
```

### Channel Plugin 能力

```
capabilities:
  ├─ direct DMs
  ├─ channel messages
  ├─ threads
  ├─ polls
  ├─ reactions
  ├─ media
  └─ native commands

streaming: block coalesce（min 1500 chars, 1s idle）
config reload prefix: "channels.discord"
```

### Runtime 隔離

```typescript
// extensions/discord/src/runtime.ts
let runtime: PluginRuntime | null = null;

export function setDiscordRuntime(next: PluginRuntime) {
  runtime = next;
}
export function getDiscordRuntime(): PluginRuntime {
  if (!runtime) throw new Error("Discord runtime not initialized");
  return runtime;
}
// 目的：防止循環依賴，延遲初始化
```

---

## 18. Config Schema 完整結構

### 頂層結構

```typescript
DiscordConfig = {
  accounts?: Record<string, DiscordAccountConfig>,
  defaultAccount?: string,
} & DiscordAccountConfig  // 基礎（default 帳號）欄位
```

### DiscordAccountConfig 完整欄位

```
基本控制
  ├─ name?: string                    // 顯示名稱
  ├─ enabled?: boolean                // 預設 true
  ├─ token?: SecretInput              // Bot token
  └─ capabilities?: string[]          // Provider capability tags

訊息處理
  ├─ textChunkLimit?: number          // 預設 2000
  ├─ chunkMode?: "length" | "newline" // 預設 "length"
  ├─ maxLinesPerMessage?: number      // 預設 17
  ├─ historyLimit?: number            // 預設 20
  ├─ dmHistoryLimit?: number          // DM 專用
  └─ mediaMaxMb?: number              // 預設 8 MB

串流 + Draft
  ├─ streaming?: "off" | "partial" | "block" | "progress" | boolean
  ├─ draftChunk?: BlockStreamingChunkConfig
  └─ blockStreamingCoalesce?: BlockStreamingCoalesceConfig

DM 設定
  ├─ dm?: { enabled?, policy?, allowFrom?, groupEnabled?, groupChannels? }
  ├─ dmPolicy?: DmPolicy             // 捷徑
  └─ allowFrom?: string[]            // 捷徑

Guild 設定
  └─ guilds?: Record<string, DiscordGuildEntry>
     ├─ slug?, requireMention?, ignoreOtherMentions?
     ├─ tools?, toolsBySender?
     ├─ reactionNotifications?: "off" | "own" | "all" | "allowlist"
     ├─ users?, roles?
     └─ channels?: Record<string, DiscordGuildChannelConfig>
        ├─ allow?, enabled?, requireMention?, ignoreOtherMentions?
        ├─ tools?, toolsBySender?, skills?
        ├─ users?, roles?
        ├─ systemPrompt?, includeThreadStarter?
        └─ （每個欄位都可覆寫 guild 層級）

Access Control
  ├─ groupPolicy?: "open" | "disabled" | "allowlist"
  ├─ allowBots?: boolean | "mentions"  // 預設 false
  ├─ dangerouslyAllowNameMatching?: boolean
  └─ configWrites?: boolean            // 預設 true

Actions（工具開關）
  └─ actions?: {
       reactions?, stickers?, polls?, permissions?, messages?,
       threads?, pins?, search?, memberInfo?, roleInfo?, roles?,
       channelInfo?, voiceStatus?, events?, moderation?,
       emojiUploads?, stickerUploads?, channels?, presence?
     }

Intents
  └─ intents?: {
       presence?: boolean,       // 需 Portal opt-in
       guildMembers?: boolean    // 需 Portal opt-in
     }

Voice
  └─ voice?: {
       enabled?: boolean,        // 預設 true
       autoJoin?: { guildId, channelId }[],
       daveEncryption?: boolean, // 預設 true
       decryptionFailureTolerance?: number, // 預設 24
       tts?: TtsConfig
     }

Bot Presence
  ├─ status?: "online" | "dnd" | "idle" | "invisible"
  ├─ activity?: string
  ├─ activityType?: 0-5          // 4=Custom 預設
  └─ activityUrl?: string

Auto-Presence
  └─ autoPresence?: {
       enabled?: boolean,         // 預設 false
       intervalMs?: number,       // 預設 30000（最小 5000）
       minUpdateIntervalMs?: number, // 預設 15000（最小 1000）
       healthyText?, degradedText?, exhaustedText?
     }

Thread Bindings
  └─ threadBindings?: {
       enabled?: boolean,
       idleHours?: number,        // 預設 24（0=停用）
       maxAgeHours?: number,      // 預設 0（停用）
       spawnSubagentSessions?: boolean, // 預設 false
       spawnAcpSessions?: boolean       // 預設 false
     }

Exec Approvals
  └─ execApprovals?: {
       enabled?: boolean,         // 預設 false
       approvers?: string[],      // Discord user IDs
       agentFilter?, sessionFilter?,
       cleanupAfterResolve?: boolean,
       target?: "dm" | "channel" | "both"
     }

Agent Components
  └─ agentComponents?: { enabled?: boolean }  // 預設 true

UI
  └─ ui?: { components?: { accentColor?: string } }

Commands
  ├─ commands?: ProviderCommandsConfig
  └─ slashCommand?: { ephemeral?: boolean }   // 預設 true

Outbound + Delivery
  ├─ retry?: OutboundRetryConfig
  ├─ responsePrefix?: string
  ├─ ackReaction?: string
  ├─ ackReactionScope?: "group-mentions" | "group-all" | "direct" | "all" | "off" | "none"
  ├─ replyToMode?: "off" | "first" | "all"
  └─ defaultTo?: string

Worker
  └─ inboundWorker?: { runTimeoutMs?: number }  // 預設 1,800,000 (30min)

Event Queue
  └─ eventQueue?: {
       listenerTimeout?: number,  // 預設 120,000 (2min)
       maxQueueSize?: number,     // 預設 10,000
       maxConcurrency?: number    // 預設 50
     }

Advanced
  ├─ proxy?: string               // HTTPS proxy URL
  ├─ markdown?: MarkdownConfig
  ├─ heartbeat?: ChannelHeartbeatVisibilityConfig
  └─ pluralkit?: DiscordPluralKitConfig
```

---

## 19. PluralKit + 特殊整合

### PluralKit

```
pluralkit.ts
  ├─ API: api.pluralkit.me/v2
  ├─ fetchPluralKitMessageInfo(messageId, config) → system/member info
  ├─ 需 config enabled + optional API token
  ├─ Webhook 訊息偵測 → 查詢 PK API → 取得真實身份
  └─ PK 成員豁免 allowBots=off 限制
```

### Directory Cache

```
directory-cache.ts（記憶體 LRU）
  ├─ rememberDiscordDirectoryUser() → 快取 handle:ID 映射
  ├─ resolveDiscordDirectoryUserId() → 查找 ID by handle
  ├─ 最大 4000 筆
  └─ 用於 @username mention 重寫

directory-live.ts（REST-backed）
  ├─ listDiscordDirectoryGroupsLive() → 匹配 channels
  └─ listDiscordDirectoryPeersLive() → 匹配 members（guild 遍歷搜尋）
```

### Mention 重寫

```
mentions.ts
  ├─ formatMention(id) → <@id>, <@&id>, <#id>
  ├─ rewriteDiscordKnownMentions(text) → 掃描 @username 模式
  │  ├─ 跳過 markdown code blocks
  │  ├─ 透過 directory cache 解析
  │  └─ 找不到 → 保留原文
  └─ 跳過 @everyone, @here
```

---

## 20. 邊界條件與陷阱

| # | 陷阱 | 影響 | 檔案 |
|---|------|------|------|
| 1 | **Forum/Media channel 拒絕直接 POST /messages** | 必須建立 thread，starter message 用 message body field | `send.outbound.ts` |
| 2 | **Thread webhook echo** | Unbind 後 30s 內 webhook 訊息會被誤認為 inbound | `thread-bindings.state.ts` |
| 3 | **Preflight abort 安全** | 每個 async 後都要 check abort signal，否則可能 process 已取消的訊息 | `message-handler.preflight.ts` |
| 4 | **Debounce 批次合併** | 多訊息合成為 synthetic message，丟失個別 message ID 追蹤 | `message-handler.ts` |
| 5 | **Bot self-filter 在 debounce 前** | 否則 bot 自己的訊息佔用 debounce 容量 | `message-handler.ts` |
| 6 | **Permission overwrite 順序** | @everyone → bot roles → bot user，順序錯誤會計算出錯誤權限 | `send.permissions.ts` |
| 7 | **globalThis 狀態共享** | Thread binding state 必須放 globalThis，否則 Plugin(Jiti/CJS) vs Core(ESM) 看到不同實例 | `thread-bindings.state.ts` |
| 8 | **Thread starter cache TTL** | 5min TTL + 500 entry LRU，高流量可能 evict 導致重複 API 呼叫 | `monitor/threading.ts` |
| 9 | **Voice decrypt failure tolerance** | 3 failures / 30s → reconnect，DAVE 加密環境更容易觸發 | `voice/manager.ts` |
| 10 | **Code fence 跨 chunk 平衡** | Chunker 必須自動補開/補關 fence，否則 Discord 渲染壞掉 | `chunk.ts` |
| 11 | **Reasoning tag 洩漏** | Process 階段必須剝離 reasoning tags，否則內部推理洩漏給使用者 | `message-handler.process.ts` |
| 12 | **DM pairing store-based 存取** | 配對碼寫入 store 但 store 是 per-session scoped → 確保正確的 store instance | `dm-command-decision.ts` |
| 13 | **Sweep 中的 touch race** | Sweeper 可能 snapshot stale state → unbind，但實際已被 touch 更新 | `thread-bindings.lifecycle.ts` |
| 14 | **Component custom ID 100 字元限制** | Model picker 等必須壓縮 state 到 100 字元內 | `model-picker.ts` |
| 15 | **Rate limit 429 + Retry-After** | 必須解析 header 為 ms（Discord 回傳 seconds），轉換錯誤會無限重試 | `reply-delivery.ts` |
| 16 | **Voice message flag 8192** | 缺少此 flag → Discord 不會以語音訊息格式顯示 | `send.outbound.ts` |
| 17 | **Sticker 上限 3** | 超過 3 個 sticker IDs → Discord API 拒絕 | `send.outbound.ts` |
| 18 | **Poll 上限 10 答案 / 768h** | 超出限制 → API 400 | `send.outbound.ts` |
| 19 | **Auto-presence 去重 signature** | 不去重會頻繁呼叫 gateway updatePresence → rate limit | `auto-presence.ts` |
| 20 | **ACP binding stale detection** | 啟動時必須 reconcile ACP bindings，否則 dead session 永不清理 | `thread-bindings.lifecycle.ts` |

---

## 21. 關鍵常量速查

| 常量 | 值 | 用途 |
|------|-----|------|
| `DISCORD_TEXT_LIMIT` | 2000 | 訊息字數上限 |
| `DISCORD_MAX_LINES_PER_MESSAGE` | 17 | 軟行數限制 |
| `DISCORD_MAX_STICKERS` | 3 | Sticker 數量上限 |
| `DISCORD_POLL_MAX_ANSWERS` | 10 | Poll 答案上限 |
| `DISCORD_POLL_MAX_HOURS` | 768 (32d) | Poll 持續時間上限 |
| `DISCORD_EMOJI_MAX_SIZE` | 256 KB | Emoji 上傳大小限制 |
| `DISCORD_STICKER_MAX_SIZE` | 512 KB | Sticker 上傳大小限制 |
| `SUPPRESS_NOTIFICATIONS_FLAG` | 1 << 12 | 靜音訊息 bit flag |
| `IS_VOICE_MESSAGE_FLAG` | 8192 | 語音訊息 flag |
| `DISCORD_CANNOT_DM` | 50007 | 無法 DM 錯誤碼 |
| `DISCORD_MISSING_PERMISSIONS` | 50013 | 權限不足錯誤碼 |
| `DISCORD_DEFAULT_LISTENER_TIMEOUT_MS` | 120,000 (2min) | Listener 超時 |
| `DISCORD_DEFAULT_INBOUND_WORKER_TIMEOUT_MS` | 1,800,000 (30min) | Worker 超時 |
| `DISCORD_SLOW_LISTENER_THRESHOLD_MS` | 30,000 | 慢 listener 警告門檻 |
| `DISCORD_TYPING_MAX_DURATION_MS` | 1,200,000 (20min) | Typing indicator 最長持續 |
| `DISCORD_THREAD_STARTER_CACHE_TTL_MS` | 300,000 (5min) | Thread starter cache TTL |
| `DISCORD_THREAD_STARTER_CACHE_MAX` | 500 | Thread starter cache 上限 |
| `DISCORD_DIRECTORY_CACHE_MAX_ENTRIES` | 4000 | 使用者目錄 cache 上限 |
| `DISCORD_DELIVERY_RETRY_ATTEMPTS` | 3 | Delivery 重試次數 |
| `DISCORD_DELIVERY_RETRY_MIN_DELAY_MS` | 1,000 | Delivery 最小延遲 |
| `DISCORD_DELIVERY_RETRY_MAX_DELAY_MS` | 30,000 | Delivery 最大延遲 |
| `THREAD_BINDINGS_SWEEP_INTERVAL_MS` | 120,000 (2min) | Binding sweep 間隔 |
| `DEFAULT_THREAD_BINDING_IDLE_TIMEOUT_MS` | 86,400,000 (24h) | Binding idle 超時 |
| `THREAD_BINDING_TOUCH_PERSIST_MIN_INTERVAL_MS` | 15,000 | Touch 持久化限速 |
| `RECENT_UNBOUND_WEBHOOK_ECHO_WINDOW_MS` | 30,000 (30s) | Echo 抑制視窗 |
| `COMPONENT_REGISTRY_TTL_MS` | 1,800,000 (30min) | Component 互動過期 |
| `DRAFT_STREAM_THROTTLE_MS` | 1,200 | Draft edit throttle |
| `MAX_GATEWAY_RECONNECT_ATTEMPTS` | 50 | Gateway 最大重連次數 |
| `MODEL_PICKER_CUSTOM_ID_MAX` | 100 | Custom ID 字元上限 |
| `AUTO_PRESENCE_DEFAULT_INTERVAL_MS` | 30,000 | Auto-presence 輪詢間隔 |
| `AUTO_PRESENCE_MIN_UPDATE_INTERVAL_MS` | 15,000 | 最小更新間隔 |
| `VOICE_SAMPLE_RATE` | 48,000 | 語音取樣率 |
| `VOICE_DECRYPT_FAILURE_TOLERANCE` | 24 | 解密失敗容忍次數 |
| `VOICE_WAVEFORM_SAMPLES` | 256 | Waveform 取樣點數 |

---

## 22. C# 概念對照

| Discord/TS 概念 | C# 對照 | 說明 |
|-----------------|---------|------|
| `@buape/carbon Client` | Discord.NET `DiscordSocketClient` | Bot WebSocket 連線管理 |
| `GatewayPlugin` | `DiscordSocketConfig` | Gateway intent + shard + 重連設定 |
| `fetchDiscord<T>()` | `DiscordRestClient.SendAsync<T>()` | 泛型 REST API 呼叫 |
| `RetryRunner` | Polly `RetryPolicy` | 指數退避重試策略 |
| `KeyedAsyncQueue` | `Channel<T>` + `SemaphoreSlim` per-key | Per-session 串行化佇列 |
| `createStatusReactionController` | 無直接對照 | 類似 `IProgress<T>` + 狀態機 |
| `ThreadBindingManager` | 自定義 `ConcurrentDictionary` + Timer | 持久化 binding 管理 |
| `globalThis` 狀態共享 | `static` singleton / `ServiceLocator` | 跨模組載入器共享 |
| `AbortSignal` | `CancellationToken` | 協作式取消 |
| `Carbon Command` | Discord.NET `SlashCommandBuilder` | Slash command 註冊框架 |
| `VoiceConnection` | Discord.NET `IAudioClient` | 語音頻道連線 |
| `AudioPlayer` | `AudioOutStream` | 音訊播放 |
| `Webhook send` | `DiscordWebhookClient.SendMessageAsync()` | Webhook 訊息發送 |
| `PermissionFlagsBits (bigint)` | `GuildPermission (enum flags)` | Permission bitfield 運算 |
| `LRU Cache` | `MemoryCache` + `CacheItemPolicy` | TTL + 容量限制快取 |
| `resolveAuthAvailability()` | Health Check pattern | 服務健康狀態判定 |
| `ProxyGatewayPlugin` | `WebProxy` + `HttpClientHandler.Proxy` | HTTP/WS 代理 |
| `ChannelInboundDebouncer` | Reactive Extensions `Throttle` | 訊息防抖 |
| `sendDiscordComponentMessage()` | `ComponentBuilder` + `ButtonBuilder` | Discord 互動元件 |
| `Thread sweep timer` | `System.Timers.Timer` + `Parallel.ForEachAsync` | 定期清理 |
| Extension `register()` | DI `IServiceCollection.AddSingleton()` | Plugin 註冊到容器 |
| `setDiscordRuntime()` | Service Locator / lazy `Lazy<T>` | 延遲初始化依賴注入 |
