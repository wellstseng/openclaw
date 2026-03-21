# Telegram + Slack 完整實作深入

> Phase 3-2 | 掃描範圍：`src/telegram/` 64 files (~15,100 lines) + `src/slack/` ~80 files (~12,700 lines) + `extensions/telegram/` + `extensions/slack/` + auto-reply channel plugins
> 更新：2026-03-13

---

## 目錄

1. [架構鳥瞰與三頻道對照](#1-架構鳥瞰與三頻道對照)
2. [Telegram 連線架構：Webhook vs Polling](#2-telegram-連線架構webhook-vs-polling)
3. [Slack 連線架構：Socket Mode vs HTTP](#3-slack-連線架構socket-mode-vs-http)
4. [Telegram Inbound Pipeline](#4-telegram-inbound-pipeline)
5. [Slack Inbound Pipeline](#5-slack-inbound-pipeline)
6. [Access Control 比較](#6-access-control-比較)
7. [Telegram Outbound：Send + Streaming](#7-telegram-outboundsend--streaming)
8. [Slack Outbound：Send + Streaming](#8-slack-outboundsend--streaming)
9. [Threading 與 Session 模型](#9-threading-與-session-模型)
10. [Slash Commands 比較](#10-slash-commands-比較)
11. [Media Handling 比較](#11-media-handling-比較)
12. [Telegram 專屬：Sticker + Inline Buttons + Forum Topics](#12-telegram-專屬sticker--inline-buttons--forum-topics)
13. [Slack 專屬：Blocks + Modals + Interactions](#13-slack-專屬blocks--modals--interactions)
14. [Account 多帳號管理](#14-account-多帳號管理)
15. [Extension Plugin 架構](#15-extension-plugin-架構)
16. [Auto-Reply 頻道整合](#16-auto-reply-頻道整合)
17. [Monitor + Health Probe](#17-monitor--health-probe)
18. [Error Handling + Reconnect](#18-error-handling--reconnect)
19. [邊界條件與陷阱](#19-邊界條件與陷阱)
20. [關鍵常量速查](#20-關鍵常量速查)
21. [C# 概念對照](#21-c-概念對照)

---

## 1. 架構鳥瞰與三頻道對照

### 檔案分佈

| 區域 | Telegram | Slack | Discord（對照） |
|------|----------|-------|-----------------|
| Core | `src/telegram/*.ts` ~20 files | `src/slack/*.ts` ~15 files | `src/discord/*.ts` ~35 files |
| Monitor | `src/telegram/bot*` + `monitor.ts` | `src/slack/monitor/` ~20 files | `src/discord/monitor/` ~90 files |
| Events | 內嵌於 bot-handlers | `src/slack/events/` 5 files | 內嵌於 listeners |
| Delivery | `src/telegram/bot/delivery.*` | `src/slack/send.ts` + `streaming.ts` | `src/discord/send.*` 多檔 |
| Extension | `extensions/telegram/` 3 files | `extensions/slack/` 4 files | `extensions/discord/` 5 files |
| **總計** | **~64 files, ~15.1K LOC** | **~80 files, ~12.7K LOC** | **~170 files, ~41.6K LOC** |

### 依賴庫比較

| | Telegram | Slack | Discord |
|--|----------|-------|---------|
| SDK | **grammy** | **@slack/bolt** + **@slack/web-api** | **@buape/carbon** |
| 連線 | grammy Bot + @grammyjs/runner | Bolt SocketModeReceiver / HTTPReceiver | Carbon GatewayPlugin + ws |
| 限流 | @grammyjs/transformer-throttler | Bolt 內建 | 自建 rate limit |
| 並行 | @grammyjs/runner concurrency | Bolt 內建 event loop | Keyed async queue |

### 共通架構模式

三個頻道都遵循相同的 OpenClaw channel 抽象：

```
Inbound: Platform Event → Middleware/Preflight → Context Build → dispatchInboundMessage()
                                                                        ↓
                                                                  Auto-Reply Engine
                                                                        ↓
Outbound: ReplyPayload[] → Format → Chunk → Send API → Platform Delivery
```

差異在於每個平台的 **連線模式**、**訊息格式**、**互動元件** 和 **限制**。

---

## 2. Telegram 連線架構：Webhook vs Polling

### 2.1 Webhook 模式

**檔案**：`src/telegram/webhook.ts` (285 lines)

| 設定 | 值 | 說明 |
|------|------|------|
| 路徑 | `/telegram-webhook` | POST endpoint |
| Max body | 1 MB | `TELEGRAM_WEBHOOK_MAX_BODY_BYTES` |
| Body timeout | 30s | `TELEGRAM_WEBHOOK_BODY_TIMEOUT_MS` |
| Callback timeout | 10s | `TELEGRAM_WEBHOOK_CALLBACK_TIMEOUT_MS` |

**啟動流程**：
1. 建立 HTTP server（`host:port`，預設 `127.0.0.1:8787`）
2. 驗證 `x-telegram-bot-api-secret-token` header
3. 呼叫 `bot.api.setWebhook(publicUrl, { secret_token, allowed_updates })`
4. 支援自簽憑證（`InputFile`）

**關閉**：`deleteWebhook({ drop_pending_updates: false })` + AbortSignal 優雅終止

### 2.2 Polling 模式

**檔案**：`src/telegram/monitor.ts` (422 lines)

| 設定 | 值 |
|------|------|
| Runner | `@grammyjs/runner` 並行處理 |
| 並行數 | `resolveAgentMaxConcurrent(cfg)` |
| Fetch timeout | 30s（grammy 預設） |
| Max retry time | 60 min |
| Stall detection | 90s 無 `getUpdates` → 強制重啟 |
| Watchdog interval | 30s |
| 初始 backoff | 2s |
| Max backoff | 30s |
| Factor | 1.8x |
| Jitter | 25% |

**Update Offset 持久化**：
- 追蹤 `highestCompletedUpdateId` + `pendingUpdateIds` (Set)
- 只持久化 < min(pendingId) 的 offset
- 重啟時保證 exactly-once 語義

**Stall Detection**：
- 每 30s 檢查上次 `getUpdates` 時間
- 超過 90s → 強制 restart runner
- 偵測掛住的 polling loop

---

## 3. Slack 連線架構：Socket Mode vs HTTP

### 3.1 Socket Mode（預設）

**檔案**：`src/slack/monitor/provider.ts` (520 lines)

| 設定 | 值 |
|------|------|
| Receiver | Bolt `SocketModeReceiver` |
| Token | App Token (`xapp-...`) |
| 重連 backoff | 初始 2s → max 30s, factor 1.8, jitter 25% |
| Max attempts | 12 |

**流程**：
1. 建立 `@slack/bolt` App + SocketModeReceiver
2. WebSocket 連線（使用 App Token）
3. 註冊 event handlers（message, app_mention, reactions, interactions 等）
4. 自動重連 + exponential backoff
5. 不可恢復錯誤（account_inactive, invalid_auth, token_revoked）→ 永久關閉

### 3.2 HTTP 模式（Events API）

| 設定 | 值 |
|------|------|
| Receiver | Bolt `HTTPReceiver` |
| 路徑 | `/slack/events`（可設定 `webhookPath`） |
| 驗證 | `signingSecret` 簽名驗證 |
| Max body | 1 MB |
| Timeout | 30s |

**差異**：不需 App Token，只需 Bot Token + Signing Secret。

### 連線模式對照

| | Telegram Webhook | Telegram Polling | Slack Socket | Slack HTTP |
|--|-----------------|-----------------|-------------|-----------|
| 方向 | Push（Telegram → Server） | Pull（Server → Telegram） | Push（WebSocket） | Push（Slack → Server） |
| 需要公網 | **是** | 否 | 否 | **是** |
| 即時性 | 即時 | ~30s long-poll | 即時 | 即時 |
| Token | Bot Token | Bot Token | App Token | Bot + Signing Secret |
| 適用場景 | 生產環境 | 開發/除錯 | 絕大多數場景 | 需要 HTTP 整合 |

---

## 4. Telegram Inbound Pipeline

**主檔案**：`bot-handlers.ts`, `bot-message-context.ts` (~600 lines)

### Middleware Stack（循序）

```
Telegram Update (JSON)
  ↓
1. Update Dedup（optional, skip 已處理 update）
  ↓
2. Sequentialize（per-chat sequential key → 保證同 chat 順序處理）
  ↓
3. Update ID Tracking（追蹤 pending/completed → safe offset 持久化）
  ↓
4. Raw Update Logging（verbose mode, 含 sanitize）
  ↓
5. registerTelegramHandlers() → 事件分派
```

### 訊息聚合機制

**Text Fragment Coalescing**（處理 Telegram 4096 字拆分）：

| 參數 | 值 | 用途 |
|------|------|------|
| Max gap | 1500ms | 合併視窗 |
| Threshold | 4000 chars | 啟動緩衝 |
| Max parts | 12 | 最多合併數 |
| Max total | 50,000 chars | 合併上限 |

**Media Group Assembly**：
- 偵測 `media_group_id` → buffer 同組 media
- Timeout ~200ms（`MEDIA_GROUP_TIMEOUT_MS`）後 flush
- 收集所有 media 後統一處理

**Forward Burst Debounce**：
- 轉發訊息用 80ms 快速 debounce
- 正常訊息 80ms+ debounce（可設定）

### Context Building

`buildTelegramMessageContext()` 完整流程：

1. **Chat Type 偵測**：group / supergroup / private、forum flag、thread ID
2. **Access Control**：DM allowFrom、group allowFrom、forum topic enabled
3. **Session 解析**：`agent:{agentId}:telegram:{context}:{peerId}[:thread:{threadId}]`
4. **Group History**：per-chat in-memory（限 `historyLimit` 筆）
5. **Ack Reactions**：根據 scope 決定 emoji feedback
6. **Inbound Body**：text + entity parsing + mention detection + media file ID

---

## 5. Slack Inbound Pipeline

**主檔案**：`message-handler.ts` (256 lines) → `prepare.ts` (803 lines) → `dispatch.ts` (531 lines)

### Event Reception

| Event | 處理檔案 | 說明 |
|-------|---------|------|
| `message` | `events/messages.ts` (83) | 主要入站 |
| `app_mention` | `events/messages.ts` | 頻道中 @mention |
| `reaction_added/removed` | `events/reactions.ts` (72) | Emoji 反應 |
| `pin_added/removed` | `events/pins.ts` (81) | 釘選 |
| Channel lifecycle | `events/channels.ts` (162) | 頻道建立/改名/成員異動 |
| Block actions | `events/interactions.ts` (675) | UI 互動 |
| View submissions | `events/interactions.modal.ts` (262) | Modal 表單 |

### 處理流程

```
Slack Event (Socket/HTTP)
  ↓
1. App ID 驗證（drop mismatched workspace）
  ↓
2. Debounce（skip rapid successive from same sender）
  ↓
3. DM / Channel 分流（channel ID "D" 開頭 = DM）
  ↓
4. prepare()
   ├─ 授權檢查（DM policy, allowFrom, channel access）
   ├─ 內容萃取（text, mention stripping, media）
   ├─ Thread history（scope: thread or channel, limit 20）
   ├─ Context building（label, session key, mention gate）
   └─ Agent routing（mention or default）
  ↓
5. dispatch()
   ├─ Create reply dispatcher + typing callback
   ├─ dispatchInboundMessage()（auto-reply 核心）
   ├─ Streaming setup（native or draft-stream）
   └─ Thread participation cache update（24h TTL, 5K max）
```

### SlackMonitorContext

**檔案**：`context.ts` (431 lines)

中央 context 物件，貫穿整個 pipeline：
- Bolt App instance、tokens、account/team/bot IDs
- Config snapshots（history limits, DM policy, group policy, reaction mode）
- Caches：channel histories (Map)、mention regexes (WeakMap)、thread participation
- Resolvers：channel names、user names、session keys
- Status callbacks

---

## 6. Access Control 比較

### 三層防護模型

| 層級 | Telegram | Slack | Discord |
|------|----------|-------|---------|
| **DM Policy** | `pairing` / `allow` / `deny` | `pairing` / `open` / `closed` | `pairing` / `open` / `deny` |
| **AllowFrom** | userId / username list | userId / slug list（5s cache） | 6 種 token 格式 + slug |
| **Group Policy** | per-chat enabled + wildcard | `open` / `disabled` / `allowlist` | guild → channel 2 層 fallback |
| **Mention Gate** | `requireMention` per chat/topic | `requireMention`（預設 true） | `requireMention` + thread bypass |

### Telegram 特有

- **Forum Topic 控制**：per-topic `enabled` + `agentId` override
- **Group Config Inheritance**：Topic → Group → Wildcard（`"*"`）
- `evaluateTelegramGroupBaseAccess()` + `evaluateTelegramGroupPolicyAccessForPolicy()`

### Slack 特有

- **Per-Channel Config**：`channels[channelId]` 可 override（enabled, requireMention, tools, users, skills, systemPrompt）
- **Bot Message 開關**：`allowBots` config
- **Thread Participation Bypass**：bot 已回覆過的 thread 不需 mention
- **Reaction Notification Filter**：`"off"` / `"own"` / `"all"` / `"allowlist"`
- `auth.ts` (285 lines) + `dm-auth.ts` (67 lines) + `allow-list.ts` (109 lines) + `channel-config.ts` (151 lines)

---

## 7. Telegram Outbound：Send + Streaming

### Send Pipeline

**檔案**：`send.ts` (1,269 lines) → `bot/delivery.replies.ts` (661 lines) → `bot/delivery.send.ts`

**Send 函數**：
- `sendTelegramText()` — 純文字
- `sendTelegramMedia()` — photo/video/audio/document
- `sendTelegramPoll()` — 投票
- `sendTelegramReaction()` — emoji 反應

### 格式轉換

**Markdown → Telegram HTML**（`format.ts`）：
- bold/italic/code/code blocks/links
- 表格渲染為 code block（可設定）

### Chunking

**檔案**：`draft-chunking.ts`

| 參數 | 預設值 | 說明 |
|------|--------|------|
| Min chunk | 200 chars | 最小切塊 |
| Max chunk | 800 chars | 最大切塊（不超過 textLimit） |
| Break preference | `paragraph` / `newline` / `sentence` | 斷句策略 |

### Three-Lane Delivery 狀態機

**檔案**：`lane-delivery-text-deliverer.ts` (463 lines)

與 Discord 相同的三車道架構：

| Lane | 用途 | 行為 |
|------|------|------|
| **Draft** | 逐步 token streaming | 累積到 minChars 後發送 |
| **Reasoning** | think/reasoning blocks | 獨立訊息 |
| **Status** | 狀態指示（typing, reactions） | 即時更新 |

每個 lane 獨立追蹤已送出 message IDs，支援 edit-or-new 策略。

### Reply Threading

| Mode | 行為 |
|------|------|
| `"off"` | 不 thread |
| `"first"` | 只 thread 第一則回覆 |
| `"all"` | 所有回覆都 thread |

**Thread Params**：`message_thread_id`（forum topics）+ `reply_parameters`（reply to）

---

## 8. Slack Outbound：Send + Streaming

### Send Pipeline

**檔案**：`send.ts` (360 lines)

**`sendMessageSlack(target, text, opts)`**：
- target：`C...`（channel）/ `D...`（DM）/ `U...`（user）/ `W...`（workspace user）
- Text limit：**4000 chars**（vs Telegram 4096, Discord 2000）
- Chunking：by length or newline
- Identity 自訂：`chat.write.customize` scope → 自訂 username + icon

### 格式轉換

**Markdown → Slack mrkdwn**（`format.ts`, 150 lines）：
- 表格模式：none / plain / blocks
- 特殊字元跳脫：`&`, `<`, `>`
- Link 保留（angle-bracket tokens）

### Two Streaming Modes

| Mode | 機制 | 適用 |
|------|------|------|
| **Native Streaming** | `chat.startStream` → `appendStream` → `stopStream` | 逐字更新（Slack SDK ChatStreamer） |
| **Draft Stream** | 分塊更新（replace / status_final / append） | 舊版相容 |

**Native Streaming**（`streaming.ts`, 153 lines）：
- 需要 team ID、channel、thread TS
- 處理 cross-origin redirect（CDN 重導向時 strip auth header）

**Draft Stream**（`draft-stream.ts`, 140 lines）：
- `replace`：每次覆寫訊息
- `status_final`：先顯示狀態，完成後送最終文字
- `append`：每塊追加

### Blocks 支援

- `validateSlackBlocksArray()` 驗證
- `blocks-fallback.ts`（42 lines）— 從 blocks 產生 fallback text
- 可直接傳 Slack Block Kit JSON

---

## 9. Threading 與 Session 模型

### Session Key 結構比較

| 平台 | 格式 | 範例 |
|------|------|------|
| Telegram DM | `agent:{id}:telegram:dm:{peerId}` | `agent:default:telegram:dm:123456` |
| Telegram DM Thread | `...dm:{peerId}:thread:{threadId}` | `...dm:123456:thread:789` |
| Telegram Group | `agent:{id}:telegram:group:{chatId}` | `agent:default:telegram:group:-100123` |
| Telegram Forum Topic | `...group:{chatId}:topic:{topicId}` | `...group:-100123:topic:5` |
| Slack DM | `slack:{accountId}:{channelId}:{userId}` | `slack:default:D0123:U0456` |
| Slack Channel/Thread | `slack:{accountId}:{channelId}:{senderId}` | `slack:default:C0123:U0456` |
| Discord DM | `agent:{id}:discord:dm:{userId}` | `agent:default:discord:dm:123` |
| Discord Thread | `...:{channelId}:thread:{threadId}` | `...guild:456:thread:789` |

### Thread Binding 比較

| | Telegram | Slack | Discord |
|--|----------|-------|---------|
| 機制 | `thread-bindings.ts` (~150 lines) | Thread participation cache | 8 個檔案完整系統 |
| Idle timeout | 30min（預設） | 24h TTL | 24h idle → unbind |
| Max age | 7 days | N/A（TTL based） | sweep 120s + idle 24h |
| Max entries | N/A | 5,000 | globalThis 跨模組 |
| Binding key | `{chatId}:{threadId}` | `{accountId}:{channelId}:{threadTs}` | full binding object |

### Telegram Thread Spec

`resolveTelegramThreadSpec()`：
- Forum → `scope: "forum"`, `id: messageThreadId`
- DM with thread → `scope: "dm"`, `id: messageThreadId`
- Regular → `scope: "none"`

### Slack Thread Resolution

`resolveSlackThreadTargets()`：
- Top-level 訊息：無 `thread_ts`
- Thread 回覆：有 `thread_ts`（parent timestamp）
- `replyToMode` 控制回覆是否跟 thread

---

## 10. Slash Commands 比較

### Telegram Native Commands

**檔案**：`bot.ts` 中 `registerTelegramNativeCommands()`

- 透過 `BotCommand` API 註冊到 Telegram
- 設定：`commands.native` + `commands.nativeSkills`
- 格式：`/command` 直接在 chat 中使用

### Slack Slash Commands

**檔案**：`slash.ts` (881 lines) — **Slack 最大單檔**

| 功能 | 說明 |
|------|------|
| 註冊 | Native Slack slash command（`/openclaw`） |
| 設定 | name, ephemeral mode, session prefix |
| 引數 UI | Block-based（buttons, select menus, text inputs） |
| 按鈕上限 | 5 per row → overflow 轉 select menu |
| 確認 | Dangerous operation 確認 dialog |
| Dispatch | → `dispatchInboundMessage()` + 特殊 session key prefix |
| 自動註冊 | Workspace install 時自動註冊 |
| 技能命令 | `slash-skill-commands.runtime.ts` 支援 |

**支援檔案**：
- `slash-commands.runtime.ts` — 延遲載入 command registry
- `slash-dispatch.runtime.ts` — 延遲載入 dispatch resolver
- `slash-skill-commands.runtime.ts` — skill command 支援

### Discord（對照）

- `@buape/carbon` Command 框架
- Discord 100 commands 上限自動裁剪
- DM pairing flow 整合

---

## 11. Media Handling 比較

### Telegram Media

**檔案**：`bot/delivery.resolve-media.ts`, `sticker-cache.ts` (268 lines)

| 功能 | 說明 |
|------|------|
| 支援格式 | photo, video, video_note, audio, voice, document, sticker, poll |
| Input 來源 | URL / local file / Telegram file ID / base64 |
| 大小限制 | `mediaMaxMb`（預設 100MB） |
| Caption | max 1024 chars → 超過拆多則訊息 |
| Voice bubble | `asVoice: true`（相容格式才生效） |
| Video note | `asVideoNote: true`（圓形影片） |
| File ID cache | per message cache → 加速重複送 |

**Sticker 特殊處理**：
- 持久化 cache：`STATE_DIR/telegram/sticker-cache.json`
- 欄位：fileId, fileUniqueId, emoji, setName, description
- **Vision 描述生成**：用 Claude/GPT-4V 產生 1-2 句描述 → cache
- 模糊搜尋：exact match (10pts), word match (5pts), emoji (8pts), set (3pts)

### Slack Media

**檔案**：`monitor/media.ts` (519 lines)

| 功能 | 說明 |
|------|------|
| 來源 | Slack files array + message attachments |
| Max files | **5 per message**（`MAX_SLACK_MEDIA_FILES`） |
| Fetch | Authorization header + cross-origin redirect handling |
| 驗證 | HTTPS only, Slack domains only |
| Thread 過濾 | Thread reply 中 skip 父訊息的 files |
| Attachment unfurling | 從 unfurled links 萃取 preview content |

### Discord（對照）

- 25MB free / 100MB Nitro 上限
- Embed 自動 unfurl
- 自訂 emoji/sticker 管理

### 三平台 Media 限制對照

| | Telegram | Slack | Discord |
|--|----------|-------|---------|
| Text limit | 4096 chars | 4000 chars | 2000 chars |
| File upload | 50MB（Bot API）| workspace plan | 25MB/100MB |
| Caption | 1024 chars | N/A | N/A（embed desc） |
| Max files/msg | 10（media group） | 5 | 10 |

---

## 12. Telegram 專屬：Sticker + Inline Buttons + Forum Topics

### Inline Buttons

**檔案**：`inline-buttons.ts`, `model-buttons.ts`, `button-types.ts`

**Scope 設定**：

| Scope | 說明 |
|-------|------|
| `"off"` | 停用 |
| `"dm"` | 只在 DM |
| `"group"` | 只在群組 |
| `"all"` | 全部 |
| `"allowlist"` | 按帳號設定（預設） |

**Button 類型**：URL / Callback / Web App / Inline Query / Pay

**Model Picker UI**：
- 分頁 provider 列表 → 分頁 model 列表
- Callback data：encoded selection
- 可設定行列數

### Forum Topics

**偵測**：`isForum && messageThreadId`

**Config Inheritance**：
```
Topic Config（最高優先）
  ↓ 繼承
Group Config
  ↓ 繼承
Wildcard Config ("*")
```

**Topic 設定**：
- `enabled`：false → 拒絕所有訊息
- `agentId`：路由到特定 agent
- 繼承 group 的 allowFrom、policy 等

### Sticker Cache

- Vision-based 描述產生 + 持久化
- 自動偵測可用 vision model
- 模糊搜尋功能（分數加權）

---

## 13. Slack 專屬：Blocks + Modals + Interactions

### Block Actions

**檔案**：`events/interactions.ts` (675 lines)

- Prefix scoping：`openclaw:` action ID
- Payload sanitize：redact trigger_id, response_url, private_metadata
- Max event log：2400 chars
- Max interaction string：160 chars

### Modals

**檔案**：`events/interactions.modal.ts` (262 lines)

**Modal Lifecycle**：
1. 觸發 → 開啟 modal（`views.open`）
2. 使用者填寫 → `view_submission` event
3. 解碼 private metadata → 恢復 session context
4. Input 驗證 + error messaging
5. 表單 → command 執行

**Private Metadata**（`modal-metadata.ts`, 46 lines）：
- JSON encoded, max 3000 chars
- 包含：sessionKey, channelId, channelType, userId

### Reaction Notifications

| Mode | 說明 |
|------|------|
| `"off"` | 停用 |
| `"own"` | 只 bot 自己的訊息 |
| `"all"` | 所有 |
| `"allowlist"` | 指定清單 |

### Channel Lifecycle Events

- `channel_created`, `channel_rename`, `channel_id_changed`
- `member_joined_channel`, `member_left_channel`
- → `directory-live.ts` (183 lines) 即時同步頻道目錄

---

## 14. Account 多帳號管理

### 共通模式

三個頻道都支援多帳號，結構幾乎相同：

```typescript
ResolvedAccount = {
  accountId: string;
  enabled: boolean;
  name?: string;
  token: string;
  tokenSource: "env" | "config" | "tokenFile" | "none";
  config: AccountConfig;  // platform-specific
}
```

### Token 解析優先順序

| 優先順序 | Telegram | Slack |
|---------|----------|-------|
| 1 | 參數明確傳入 | 參數明確傳入 |
| 2 | `env.TELEGRAM_BOT_TOKEN` | `env.SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN` |
| 3 | `tokenFile` 檔案讀取 | Config per-account `botToken` |
| 4 | Config `botToken` | — |

### Slack 特有 Token

| Token | 前綴 | 用途 |
|-------|------|------|
| Bot Token | `xoxb-` | API 操作 |
| App Token | `xapp-` | Socket Mode 連線 |
| User Token | `xoxp-` | 使用者級操作（read-only） |
| Signing Secret | — | HTTP mode 簽名驗證 |

---

## 15. Extension Plugin 架構

### Telegram Extension

**路徑**：`extensions/telegram/`

```
extensions/telegram/
├── src/channel.ts    → ChannelPlugin 實作
├── src/runtime.ts    → 延遲初始化 runtime
├── openclaw.plugin.json
└── package.json
```

**ChannelPlugin 介面**：
- `id: "telegram"`
- Config schema 定義
- Account CRUD（add/remove/edit）
- Group management UI
- Token validation
- Health probes
- Onboarding flow

### Slack Extension

**路徑**：`extensions/slack/`

```
extensions/slack/
├── index.ts          → plugin 註冊入口
├── src/channel.ts    → ChannelPlugin 實作
├── src/runtime.ts    → runtime 注入
├── openclaw.plugin.json
└── package.json
```

**Plugin ID**：`"slack"`，透過 `api.registerChannel()` 註冊。

### 與 Discord 對照

Discord extension 多了 `subagent-hooks.ts`（thread 整合），Telegram/Slack 目前無此對等功能。

---

## 16. Auto-Reply 頻道整合

### Telegram Auto-Reply

**檔案**：`src/auto-reply/reply/telegram-context.ts`

**Conversation ID 解析**：
- 解析 `MessageThreadId` context var
- 從 `To`/`OriginatingTo` 萃取 chat ID
- Forum topic → `{chatId}:topic:{threadId}`
- 非 topic 群組 → `undefined`（不做全域追蹤）

**Route 建構**：
- Channel: `"telegram"`
- Peer ID: `buildTelegramGroupPeerId(chatId, threadId)`

### Slack Auto-Reply

**Outbound Adapter**：`src/channels/plugins/outbound/slack.ts`
- 實作 `ChannelOutboundAdapter` 介面
- Text chunk limit：4000 chars
- Delivery mode：direct（無額外 chunking layer）
- 透過 `sendMessageSlack()` 送出
- 套用 `message_sending` hooks

**Normalize Plugin**：`src/channels/plugins/normalize/slack.ts`
- 解析 `slack://...`、`@user`、`#channel` → ID
- `looksLikeSlackTargetId()` 模式識別（C..., D..., <@U...>）

---

## 17. Monitor + Health Probe

### Telegram Probe

**檔案**：`probe.ts` (121 lines)

```typescript
TelegramProbe = {
  ok: boolean;
  status?: number;
  error?: string;
  elapsedMs: number;
  bot?: { id, username, canJoinGroups, canReadAllGroupMessages, supportsInlineQueries };
  webhook?: { url, hasCustomCert };
}
```

**流程**：retry 3x（50-1000ms delay）→ `getMe` → HTTP status 檢查 → bot capabilities → webhook info

### Slack Provider Status

**檔案**：`monitor/provider.ts`

- Callbacks：`setStatus()` / `getStatus()`
- Connected：`{ connected: true, lastEventAt: timestamp }`
- Disconnected：`{ connected: false, lastDisconnect: { at, error? } }`
- Event tracking：`trackEvent()` for liveness/heartbeat

### Discord（對照）

- `probe.ts`：`/users/@me` endpoint
- Auto-Presence：health → Discord status mapping

---

## 18. Error Handling + Reconnect

### Telegram 網路錯誤

**檔案**：`network-errors.ts`

**可恢復代碼**：
`ECONNRESET, ECONNREFUSED, EPIPE, ETIMEDOUT, UND_ERR_*, ERR_NETWORK`

**安全重試代碼**（僅 pre-connection 失敗）：
`ECONNREFUSED, ENOTFOUND, EAI_AGAIN, ENETUNREACH, EHOSTUNREACH`

**特殊處理**：
- HTML parse error（`can't parse entities`）→ fallback to plain text
- Thread not found (400) → retry without thread ID
- Message not modified (400) → skip edit
- 409 Conflict（getUpdates conflict）→ 清除 webhook → restart

### Slack Reconnect Policy

**檔案**：`reconnect-policy.ts` (108 lines)

**不可恢復錯誤**（永久關閉）：
- `account_inactive`, `invalid_auth`, `token_revoked`
- `team_access_revoked`, `missing_scope`

**可恢復**：connection error → exponential backoff

| 參數 | 值 |
|------|------|
| 初始 | 2000ms |
| Max | 30000ms |
| Factor | 1.8x |
| Jitter | 25% |
| Max attempts | 12 |

### 三平台 Reconnect 對照

| | Telegram | Slack | Discord |
|--|----------|-------|---------|
| Backoff 初始 | 2s | 2s | 5s |
| Backoff 上限 | 30s | 30s | 5min |
| Factor | 1.8x | 1.8x | exponential |
| Max attempts | 60min 時限 | 12 次 | 50 次 |
| 不可恢復 | 手動處理 | 5 種 auth error | 類似 |

---

## 19. 邊界條件與陷阱

### Telegram

1. **Text Fragment 合併 race**：1500ms window 內的快速連續訊息會被合併成單一 synthetic message → 原始 message ID 被覆寫
2. **Update Offset 持久化**：只持久化 < min(pendingId) → pending 的 update 必須完成後才推進 offset
3. **Forum vs Regular Group**：`isForum` flag 決定 thread 是 topic 還是 reply → session key 結構不同
4. **409 Conflict**：多個 instance 同時 polling → 自動清 webhook 後 restart，但不保證收斂
5. **Webhook Secret Mismatch**：header `x-telegram-bot-api-secret-token` 不匹配 → 靜默丟棄
6. **Caption 1024 上限**：media + 長文本 → 自動拆成 media（含截斷 caption）+ 後續文字訊息
7. **Sticker Vision 延遲**：首次遇到新 sticker → 同步呼叫 vision model → 可能阻塞 pipeline
8. **HTML Parse Fallback**：格式轉換失敗 → 自動降級 plain text，但 styling 全部丟失
9. **Media Group 200ms Window**：過慢的 grouped media 會被拆成個別訊息處理
10. **Group Config Wildcard 陷阱**：`groups["*"]` 設 `enabled: false` 會關閉所有未明確設定的群組

### Slack

1. **Thread Participation Cache TTL**：24h 過期 → bot 必須重新被 mention 才會回 thread
2. **Thread Participation Max Entries**：5000 上限 → 高流量 workspace 可能 evict 活躍 threads
3. **Modal Metadata 3000 chars**：超過 → Slack API error，需精簡 encoded data
4. **Interaction String 160 chars**：Block action value 上限 → 長 command 需拆分
5. **Native Streaming CDN Redirect**：auth header 被 strip → 需手動處理 cross-origin
6. **Socket Mode Reconnect 12 次**：用完後不再重試 → 需外部 supervisor 重啟
7. **Signing Secret 遺漏**：HTTP mode 必須有 signingSecret → 否則所有 request 被拒
8. **Channel ID 格式依賴**：`D` = DM、`C` = channel、`G` = group → 錯誤前綴導致路由失敗
9. **5 Files/Message 上限**：超過的 files 被靜默丟棄
10. **Slash Command Ephemeral**：ephemeral 回覆只有呼叫者看得到 → 不適合群組共享場景

---

## 20. 關鍵常量速查

### Telegram

| 常量 | 值 | 位置 |
|------|------|------|
| MAX_MESSAGE_TEXT_LENGTH | 4096 | Telegram API 限制 |
| MAX_CAPTION_LENGTH | 1024 | Telegram API 限制 |
| TEXT_FRAGMENT_THRESHOLD | 4000 | bot-handlers.ts |
| TEXT_FRAGMENT_MAX_PARTS | 12 | bot-handlers.ts |
| TEXT_FRAGMENT_MAX_TOTAL | 50,000 | bot-handlers.ts |
| TEXT_FRAGMENT_MAX_GAP | 1500ms | bot-handlers.ts |
| MEDIA_GROUP_TIMEOUT_MS | ~200ms | bot-handlers.ts |
| FORWARD_BURST_DEBOUNCE_MS | 80ms | bot-handlers.ts |
| POLL_STALL_THRESHOLD_MS | 90,000ms | monitor.ts |
| POLL_WATCHDOG_INTERVAL_MS | 30,000ms | monitor.ts |
| WEBHOOK_MAX_BODY_BYTES | 1 MB | webhook.ts |
| WEBHOOK_BODY_TIMEOUT_MS | 30,000ms | webhook.ts |
| WEBHOOK_CALLBACK_TIMEOUT_MS | 10,000ms | webhook.ts |
| maxRetryTime (grammY) | 60 min | monitor.ts |
| Draft chunk min | 200 chars | draft-chunking.ts |
| Draft chunk max | 800 chars | draft-chunking.ts |
| Probe retries | 3x | probe.ts |
| Thread idle timeout | 30 min | thread-bindings.ts |
| Thread max age | 7 days | thread-bindings.ts |
| Default backoff initial | 2s | monitor.ts |
| Default backoff max | 30s | monitor.ts |

### Slack

| 常量 | 值 | 位置 |
|------|------|------|
| SLACK_TEXT_LIMIT | 4000 | send.ts |
| MAX_SLACK_MEDIA_FILES | 5 | media.ts |
| Thread cache TTL | 24h | sent-thread-cache.ts |
| Thread cache max | 5,000 | sent-thread-cache.ts |
| Modal metadata max | 3,000 chars | modal-metadata.ts |
| Interaction event max | 2,400 chars | interactions.ts |
| Interaction string max | 160 chars | interactions.ts |
| Initial history limit | 20 messages | types.slack.ts |
| Webhook max body | 1 MB | provider.ts |
| Webhook timeout | 30s | provider.ts |
| Backoff initial | 2,000ms | reconnect-policy.ts |
| Backoff max | 30,000ms | reconnect-policy.ts |
| Backoff factor | 1.8x | reconnect-policy.ts |
| Backoff max attempts | 12 | reconnect-policy.ts |
| Pairing store cache TTL | 5,000ms | auth.ts |

---

## 21. C# 概念對照

| OpenClaw 概念 | C# / .NET 類比 |
|---------------|----------------|
| grammy `Bot` | DI-registered `TelegramBotClient`（Telegram.Bot NuGet） |
| `@slack/bolt` App | DI-registered `ISlackApiClient` or custom middleware pipeline |
| Sequentialize middleware | `SemaphoreSlim` per-key（ConcurrentDictionary<string, SemaphoreSlim>） |
| Text Fragment Coalescing | `System.Reactive` Buffer with closing selector |
| Media Group Assembly | `GroupBy` + `Buffer(timespan)` |
| Three-Lane Delivery | 3 `Channel<T>` consumers + coordinator |
| Thread Participation Cache | `MemoryCache` with absolute expiration |
| Slash Command Handler | MediatR `IRequestHandler<SlashCommand, SlashResult>` |
| Modal Lifecycle | State machine pattern or Blazor modal component |
| Reconnect Policy | Polly `WaitAndRetryAsync` with jitter |
| ChannelPlugin interface | `IChannelPlugin` + DI registration |
| Monitor Context | Scoped DI container or explicit context object |
| Sticker Vision Cache | `IDistributedCache` with serialized descriptions |
| `apiThrottler()` | `System.Threading.RateLimiting.TokenBucketRateLimiter` |
| Webhook signature check | HMAC middleware (`IMiddleware`) |
| SlackMonitorContext | Scoped service lifetime in ASP.NET DI |

---

## 附錄：完整檔案清單

### Telegram (`src/telegram/` — 64 files, ~15.1K LOC)

| 檔案 | 行數 | 職責 |
|------|------|------|
| bot.ts | 416 | Bot factory + setup + config |
| webhook.ts | 285 | Webhook listener |
| monitor.ts | 422 | Polling monitor + recovery |
| send.ts | 1,269 | 出站主邏輯 |
| bot-handlers.ts | ~900 | 入站 handler + debounce |
| bot-message-context.ts | ~600 | Context building |
| bot/delivery.replies.ts | 661 | Text/media delivery |
| bot/delivery.send.ts | — | Low-level API calls |
| bot/delivery.resolve-media.ts | — | Media URL/file resolution |
| lane-delivery-text-deliverer.ts | 463 | Streaming state machine |
| sticker-cache.ts | 268 | Sticker metadata + vision |
| format.ts | — | Markdown → HTML |
| group-access.ts | ~150 | Access control |
| accounts.ts | ~200 | Multi-account |
| inline-buttons.ts | — | Button scope config |
| probe.ts | 121 | Health check |
| network-errors.ts | — | Error classification |
| thread-bindings.ts | ~150 | Thread binding manager |

### Slack (`src/slack/` — ~80 files, ~12.7K LOC)

| 檔案 | 行數 | 職責 |
|------|------|------|
| slash.ts | 881 | Slash command handler（最大檔案） |
| monitor/prepare.ts | 803 | Content + auth + routing |
| monitor/dispatch.ts | 531 | Reply dispatch + streaming |
| monitor/media.ts | 519 | File/attachment resolution |
| monitor/provider.ts | 520 | Main monitor entry |
| monitor/context.ts | 431 | SlackMonitorContext factory |
| actions.ts | 446 | Bot actions (react, pin, read) |
| send.ts | 360 | sendMessageSlack core |
| monitor/auth.ts | 285 | DM/channel authorization |
| monitor/message-handler.ts | 256 | Orchestration |
| events/interactions.ts | 675 | Block actions + modals |
| events/interactions.modal.ts | 262 | Modal lifecycle |
| resolve-users.ts | 190 | User ID resolution |
| directory-live.ts | 183 | Channel directory sync |
| account-inspect.ts | 183 | Account diagnostics |
| streaming.ts | 153 | Native Slack streaming |
| format.ts | 150 | Markdown → mrkdwn |
| monitor/channel-config.ts | 151 | Per-channel config |
| thread-resolution.ts | 134 | Thread targeting |
| resolve-channels.ts | 137 | Channel ID resolution |
| accounts.ts | 122 | Account resolution |
| scopes.ts | 116 | OAuth scopes |
| reconnect-policy.ts | 108 | Backoff + errors |
| allow-list.ts | 109 | Allowlist resolution |
