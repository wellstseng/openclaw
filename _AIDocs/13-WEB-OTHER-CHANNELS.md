# WhatsApp + Signal + LINE + iMessage + IRC + Google Chat 完整實作深入

> Phase 3-3 | 掃描範圍：`src/web/` 45 files (~6.2K LOC) + `src/whatsapp/` 2 files + `src/signal/` 17 files (~3.1K LOC) + `src/line/` 30 files (~6.1K LOC) + `src/imessage/` 19 files (~2.5K LOC) + `extensions/irc/` 17 files (~2.9K LOC) + `extensions/googlechat/` 15 files (~2.9K LOC) + 各頻道 extensions
> 更新：2026-03-13

---

## 目錄

1. [六頻道架構鳥瞰](#1-六頻道架構鳥瞰)
2. [WhatsApp（src/web/ — Baileys）](#2-whatsappsrcweb--baileys)
3. [Signal（src/signal/ — CLI Daemon + SSE）](#3-signalsrcsignal--cli-daemon--sse)
4. [LINE（src/line/ — Webhook + REST API）](#4-linesrcline--webhook--rest-api)
5. [iMessage（src/imessage/ — CLI RPC）](#5-imessagesrcimessage--cli-rpc)
6. [IRC（extensions/irc/ — Raw Socket）](#6-ircextensionsirc--raw-socket)
7. [Google Chat（extensions/googlechat/ — Webhook + REST）](#7-google-chatextensionsgooglechat--webhook--rest)
8. [九頻道總對照表](#8-九頻道總對照表)
9. [Extension Plugin 共通架構](#9-extension-plugin-共通架構)
10. [Auto-Reply 整合比較](#10-auto-reply-整合比較)
11. [Access Control 比較](#11-access-control-比較)
12. [邊界條件與陷阱](#12-邊界條件與陷阱)
13. [關鍵常量速查](#13-關鍵常量速查)
14. [C# 概念對照](#14-c-概念對照)

---

## 1. 六頻道架構鳥瞰

### 檔案分佈

| 頻道 | src/ 位置 | src/ files | src/ LOC | ext/ files | ext/ LOC | **Total** |
|------|----------|-----------|---------|-----------|---------|----------|
| **WhatsApp** | `src/web/` + `src/whatsapp/` | 45 | 6,221 | 3 | 504 | **48 / 6.7K** |
| **Signal** | `src/signal/` | 17 | 3,141 | 3 | 354 | **20 / 3.5K** |
| **LINE** | `src/line/` | 30 | 6,137 | 4 | 1,148 | **34 / 7.3K** |
| **iMessage** | `src/imessage/` | 19 | 2,518 | 3 | 349 | **22 / 2.9K** |
| **IRC** | — | 0 | 0 | 17 | 2,925 | **17 / 2.9K** |
| **Google Chat** | — | 0 | 0 | 15 | 2,912 | **15 / 2.9K** |
| **總計** | | **111** | **18,017** | **45** | **8,192** | **156 / 26.2K** |

### 連線模式一覽

| 頻道 | 連線方式 | SDK/Library | 持久連線 |
|------|---------|-------------|---------|
| WhatsApp | WebSocket（Baileys 封裝 WhatsApp Web） | @whiskeysockets/baileys | Yes |
| Signal | HTTP RPC + SSE（signal-cli daemon） | signal-cli | Yes（SSE stream） |
| LINE | Webhook（入）+ REST API（出） | @line/bot-sdk | No（stateless） |
| iMessage | CLI stdin/stdout JSON-RPC 2.0 | imsg 自訂 CLI | Yes（child process） |
| IRC | Raw TCP/TLS socket | Node.js net/tls | Yes |
| Google Chat | Webhook（入）+ REST API（出） | google-auth-library | No（stateless） |

### 共通架構模式

所有頻道遵循同一抽象：

```
Inbound: Platform Event → Access Control → Context Build → dispatchInboundMessage()
                                                                    ↓
                                                              Auto-Reply Engine
                                                                    ↓
Outbound: ReplyPayload[] → Format → Chunk → Send API → Platform Delivery
```

差異在每平台的 **連線模式**、**認證方式**、**訊息格式** 和 **互動能力**。

---

## 2. WhatsApp（src/web/ — Baileys）

### 2.1 架構概覽

`src/web/` 是 **WhatsApp Web** 的完整實作，透過 Baileys library 模擬 WhatsApp Web 協定。命名為 `web/` 因為底層連接 WhatsApp **Web** API（不是 Cloud API / Business API）。

```
src/web/
├── session.ts          # Baileys socket 建立、QR 登入、credential 備份
├── accounts.ts         # 帳號解析（channels.whatsapp config）
├── active-listener.ts  # 全域 listener registry（sendMessage/sendPoll/sendReaction）
├── auth-store.ts       # Baileys auth state 持久化（disk-based）
├── login.ts            # QR 登入流程 + logout detection
├── login-qr.ts         # QR code 渲染
├── logout.ts           # 登出清理
├── outbound.ts         # sendMessageWhatsApp()（Gateway 出站入口）
├── inbound.ts          # 入站整合
├── media.ts            # 媒體載入、壓縮、檔名推斷
├── reconnect.ts        # 指數退避策略
├── auto-reply.ts       # auto-reply 啟動入口
├── auto-reply.impl.ts  # monitorWebChannel() 主實作
├── inbound/
│   ├── monitor.ts      # Baileys event listener（messages.upsert）
│   ├── extract.ts      # 訊息解析（text/media/location/reply）
│   ├── send-api.ts     # Baileys sendMessage 封裝
│   ├── access-control.ts # DM/group 安全策略
│   ├── dedupe.ts       # 訊息去重
│   └── media.ts        # 媒體下載
├── auto-reply/
│   ├── monitor.ts          # 主迴圈（heartbeat + watchdog + reconnect）
│   ├── deliver-reply.ts    # 出站 delivery（chunk + retry）
│   ├── heartbeat-runner.ts # 保活心跳
│   ├── mentions.ts         # @提及處理
│   └── monitor/
│       ├── on-message.ts      # 入站路由（echo/access/gating/broadcast）
│       ├── process-message.ts # AI 回覆取得 + session 記錄
│       ├── broadcast.ts       # 多 agent 廣播（平行/串行）
│       ├── echo.ts            # Echo detection（LRU cache）
│       ├── group-gating.ts    # 群組 mention/activation 閘門
│       ├── group-activation.ts # 群組啟用偵測
│       ├── group-members.ts   # 群組成員快照
│       ├── ack-reaction.ts    # 已讀回應（emoji reaction）
│       ├── commands.ts        # 控制指令
│       ├── last-route.ts      # 最後路由快取
│       ├── message-line.ts    # 訊息行處理
│       └── peer.ts            # 對等端解析
```

### 2.2 連線機制

**Baileys WebSocket**

```
createWaSocket(printQr: true)
    → makeWASocket({ auth, browser, printQRInTerminal, ... })
    → waitForWaConnection()  // Promise: connection === "open"
    → sock.ev.on("messages.upsert", handler)
    → sock.ev.on("connection.update", handler)
    → sock.ev.on("creds.update", saveCreds)
```

**Credential 管理**：

| 階段 | 行為 |
|------|------|
| 首次登入 | QR code 顯示 → 手機掃碼 → creds 存入 authDir |
| 後續啟動 | 讀取 authDir creds → 自動連線 |
| Crash 恢復 | credential 備份/還原（atomic file write） |
| 被踢出 | 偵測 loggedOut → 清除 cached session → 提示重新掃碼 |

**Reconnect 策略**：指數退避，configurable base/max/jitter。

### 2.3 入站 Pipeline

```
Baileys messages.upsert
    ↓
monitorWebInbox()               # inbound/monitor.ts
    ↓ extract()                 # 解析 text/media/location/reply
WebInboundMessage
    ↓
createWebOnMessageHandler()     # monitor/on-message.ts
    ├── Echo detection          # LRU cache of recently sent text
    ├── Access control          # DM policy + group policy + allowFrom
    ├── Group gating            # mention/activation/owner check
    ├── Broadcast routing       # multi-agent dispatch
    ↓
processMessage()                # monitor/process-message.ts
    ├── Build inbound context
    ├── Resolve agent/session route
    ├── getReplyFromConfig()    # AI reply
    ↓
deliverWebReply()               # deliver-reply.ts
    ├── Suppress reasoning prefix
    ├── Chunk text (configurable limit, default ~4090 chars)
    ├── Send media with 3x retry
    ↓
Baileys sendMessage()           # 回到 WhatsApp Web API
```

### 2.4 出站 API

`sendMessageWhatsApp()` in `outbound.ts` — Gateway 呼叫入口：

```typescript
// 透過 active listener registry 取得當前連線的 Baileys socket
requireActiveWebListener(accountId)
    → sendMessage(to, text, mediaBuffer?, mediaType?, options?)
    → { messageId, toJid }
```

支援：Text / Image / Audio / Video / Document / Reactions / Polls / Presence（composing）

### 2.5 核心型別

```typescript
interface WebInboundMessage {
  from: string              // E.164 or group JID
  to: string                // Self E.164
  accountId: string
  body: string
  chatType: "direct" | "group"
  // Media
  mediaPath?, mediaType?, mediaFileName?, mediaUrl?
  location?: NormalizedLocation
  // Actions (bound to Baileys socket)
  sendComposing: () => Promise<void>
  reply: (text: string) => Promise<void>
  sendMedia: (payload: AnyMessageContent) => Promise<void>
  // Metadata
  senderJid?, senderE164?, senderName?
  replyToId?, replyToBody?
  groupSubject?, groupParticipants?, mentionedJids?
}

interface ActiveWebListener {
  sendMessage(to, text, mediaBuffer?, mediaType?, options?)
  sendPoll(to, poll)
  sendReaction(chatJid, messageId, emoji, fromMe, participant?)
  sendComposingTo(to)
  close?()
}

interface ResolvedWhatsAppAccount {
  accountId: string
  authDir: string
  enabled: boolean
  sendReadReceipts: boolean
  dmPolicy: "pairing" | "allowlist" | "open" | "disabled"
  groupPolicy: "open" | "allowlist" | "disabled"
  allowFrom?: string[]
  groupAllowFrom?: string[]
  textChunkLimit?: number
  mediaMaxMb?: number
  ackReaction?: { emoji, removeAfter?, onlyDM? }
}
```

### 2.6 獨有特色

| 特色 | 說明 |
|------|------|
| **Pairing Challenge** | 新發訊者自動收到配對請求，需 approve 才能對話 |
| **Broadcast Groups** | 單一入站訊息 → 多 agent 回覆（parallel/sequential 策略） |
| **Echo Detection** | LRU cache + broadcast-aware 防重複回覆 |
| **Group Mention Gating** | 精細控制群組何時回覆（@bot / @all / owner / command） |
| **Baileys 逆向工程** | 非官方 WhatsApp Web API，非 Cloud API |
| **Watchdog** | 30 分鐘無訊息 timeout → 強制 reconnect |
| **Heartbeat + Watchdog** | 雙層保活（heartbeat 60s log + watchdog 60s check） |

---

## 3. Signal（src/signal/ — CLI Daemon + SSE）

### 3.1 架構概覽

```
src/signal/
├── client.ts            # JSON-RPC 2.0 client, SSE event streaming
├── daemon.ts            # 啟動 signal-cli daemon（HTTP 介面）
├── monitor.ts           # 主訊息監控迴圈 + attachment fetching
├── send.ts              # 發送訊息 + text style + attachments
├── send-reactions.ts    # 發送/移除 emoji reactions
├── format.ts            # Markdown → Signal text styles 轉換
├── identity.ts          # sender 解析（phone/UUID）+ allowlist
├── accounts.ts          # channels.signal config 解析
├── monitor/
│   ├── event-handler.ts     # 入站事件處理（pairing/reaction/mention/access）
│   ├── event-handler.types.ts # Signal envelope 型別
│   ├── mentions.ts          # 群組 @提及處理
│   └── access-policy.ts     # DM access policy
```

### 3.2 連線機制

**signal-cli Daemon + HTTP**

```
signal-cli -a {account} daemon
    --http {host}:{port}           # Default: 127.0.0.1:8080
    --no-receive-stdout
    [--receive-mode on-start|manual]
    [--ignore-attachments]
    [--send-read-receipts]
```

**通訊協定**：
- **入站**：SSE stream from `/api/v1/events?account={account}`
- **出站**：HTTP JSON-RPC 2.0（`send`, `sendTyping`, `sendReceipt`, `getAttachment`）
- **健康檢查**：`/api/v1/check`

**啟動序列**：
1. Load Signal account config
2. Spawn signal-cli daemon（child process）
3. Wait for readiness via `/api/v1/check`
4. Open SSE stream → parse envelope events
5. Reconnect with backoff on disconnect

### 3.3 入站 Pipeline

```
SSE /api/v1/events
    ↓ parse Signal envelope
    ├── dataMessage    → 文字/媒體/群組訊息
    ├── reactionMessage → emoji reaction 通知
    ├── typingMessage   → typing indicator
    ↓
event-handler.ts
    ├── Access policy (DM allowlist + group policy)
    ├── Pairing challenge
    ├── Mention detection
    ├── Attachment fetching (RPC: getAttachment)
    ↓
dispatchInboundMessage()
```

### 3.4 出站 API

```typescript
// RPC 呼叫 signal-cli daemon
send({
  message: string,
  recipients?: string[],     // Phone numbers or UUIDs
  groupId?: string,
  username?: string,
  attachments?: string[],    // File paths
  textStyle?: SignalTextStyle[]  // Rich text ranges
})

sendTyping({ recipients?, groupId? })
sendReceipt({ recipients, targetTimestamp, type: "read" | "viewed" })
```

### 3.5 Text Styling

`format.ts` 將 Markdown 轉為 Signal text style ranges：

| Markdown | Signal Style |
|----------|-------------|
| `**bold**` | BOLD |
| `*italic*` | ITALIC |
| `~~strike~~` | STRIKETHROUGH |
| `` `code` `` | MONOSPACE |
| `\|\|spoiler\|\|` | SPOILER |

### 3.6 獨有特色

| 特色 | 說明 |
|------|------|
| **Dual Identity** | 同時支援 phone number + UUID 識別 |
| **Text Style Ranges** | Markdown → Signal rich text（非其他頻道的 plain text） |
| **Read Receipts** | Daemon flag + RPC 雙模式 |
| **Typing Indicators** | 即時 typing 狀態 |
| **SSE Reconnect** | 自動重連 with backoff |
| **Daemon Lifecycle** | OpenClaw 管理 signal-cli 程序生命週期 |

---

## 4. LINE（src/line/ — Webhook + REST API）

### 4.1 架構概覽

LINE 是六頻道中檔案最多（34 files）、LOC 最高（7.3K）的——因為豐富的 UI 元件支援。

```
src/line/
├── webhook.ts              # Express middleware, signature validation
├── webhook-node.ts         # Raw Node.js HTTP handler
├── monitor.ts              # Webhook 註冊 + replay cache + auto-reply dispatch
├── bot-handlers.ts         # 事件處理（message/postback/follow/join/leave）
├── bot-message-context.ts  # Webhook event → context
├── bot-access.ts           # Allowlist + DM access policy
├── accounts.ts             # LINE account config + token 解析
├── send.ts                 # Push/Reply messages, flex, templates, quick replies
├── auto-reply-delivery.ts  # Rich message delivery（flex/template/image/text chunks）
├── download.ts             # 從 LINE servers 下載媒體
├── markdown-to-line.ts     # Markdown → LINE flex messages or plain text
├── rich-menu.ts            # Rich menu 管理
├── template-messages.ts    # Template message 轉換
├── reply-chunks.ts         # Reply token vs Push 分拆
├── group-keys.ts           # Group-specific config
└── ...                     # 其餘輔助檔案
```

### 4.2 連線機制

**Webhook（入）+ Messaging API（出）**

| 方向 | 協定 | 說明 |
|------|------|------|
| 入站 | HTTPS POST webhook | LINE Platform → OpenClaw |
| 出站 | REST API via @line/bot-sdk | OpenClaw → LINE Platform |

**Webhook 驗證**：HMAC-SHA256（raw body, channel secret）→ X-Line-Signature header

**Token 來源**（優先順序）：
1. `channels.line.channelAccessToken` config
2. `LINE_CHANNEL_ACCESS_TOKEN` env
3. `channels.line.tokenFile` file path

### 4.3 入站 Pipeline

```
LINE Platform webhook POST
    ↓ HMAC-SHA256 signature validation
    ↓ parse events
bot-handlers.ts
    ├── message event → text/image/video/audio/location/sticker
    ├── postback event → button/menu 互動
    ├── follow event → 新增好友
    ├── join event → 加入群組/聊天室
    ├── leave/unfollow event → 離開
    ↓
Replay cache check (10 min window, 4096 entries)
    ↓
bot-message-context.ts → 建構 context
    ↓
dispatchInboundMessage()
```

### 4.4 出站 API

**Reply vs Push 二段式**：

| 模式 | 條件 | API |
|------|------|-----|
| Reply | 有 replyToken（3 分鐘內有效，一次性） | `replyMessage(token, messages)` |
| Push | 主動推送或 replyToken 過期 | `pushMessage(to, messages)` |

每次 API 呼叫最多 **5 則訊息**，超過自動分批。

### 4.5 Rich Message Types

```
Text           → 純文字（含 emoji shortcodes）
Flex Message   → JSON-based rich layout（carousel, box, button, bubble）
Template       → Buttons / Confirm / Carousel / Image Carousel
Quick Reply    → 最多 13 個按鈕，附於最後一則訊息
Image          → 含 preview URL
Location       → title + address + lat/lng
Loading Anim   → 20 秒 UI loading 回饋
```

**Markdown → Flex 轉換**（`markdown-to-line.ts`）：將 Markdown IR 轉為 LINE Flex Message JSON。當 Flex 不可用時 fallback 為 plain text。

### 4.6 獨有特色

| 特色 | 說明 |
|------|------|
| **Flex Messages** | 複雜 JSON-based UI layout（carousel, bubble, box） |
| **Quick Replies** | 最多 13 按鈕，case-sensitive labels |
| **Reply Token 機制** | 3 分鐘一次性 token → fallback push |
| **Rich Menu** | Persistent area-based menu |
| **Postback Actions** | Button/menu 觸發類訊息事件 |
| **Replay Cache** | 10 分鐘去重（4096 entries） |
| **Loading Animation** | 20 秒 UI 回饋 |
| **Markdown → Flex** | 結構化 IR → 原生 rich layout |

---

## 5. iMessage（src/imessage/ — CLI RPC）

### 5.1 架構概覽

最輕量的頻道實作，依賴 `imsg` CLI 工具 + macOS iMessage 系統。

```
src/imessage/
├── client.ts           # JSON-RPC 2.0 client（stdin/stdout bidirectional）
├── accounts.ts         # iMessage account config, service/region defaults
├── send.ts             # 發送訊息 + reply tag + file attachments
├── targets.ts          # 解析目標（handle/chat_id/chat_guid/chat_identifier）
├── target-parsing-helpers.ts  # 格式輔助 + allowlist validation
├── probe.ts            # 偵測 imsg binary + RPC 支援 + DB 存取
├── constants.ts        # Default probe timeout
└── monitor.ts          # 最小 event dispatch stub
```

### 5.2 連線機制

**CLI stdin/stdout JSON-RPC 2.0**

```
spawn("imsg", ["--db", dbPath, "rpc"])
    ↓ stdin: {"jsonrpc":"2.0","id":1,"method":"send","params":{...}}\n
    ↓ stdout: {"jsonrpc":"2.0","id":1,"result":{...}}
    ↓ stdout (notification): {"method":"message","params":{...}}
```

**Probe 流程**：
1. 偵測 `imsg` binary 是否存在
2. 測試 RPC 支援：`imsg rpc --help`
3. 連線測試：`chats.list({ limit: 1 })`
4. 快取 RPC 支援狀態（per cliPath）

### 5.3 Target 格式

| 格式 | 範例 | 說明 |
|------|------|------|
| Handle | `+14155552671`, `alice@icloud.com` | E.164 phone 或 email |
| Chat ID | `42` | macOS iMessage DB numeric ID |
| Chat GUID | `iMessage;-;+14155552671` | macOS GUID 格式 |
| Chat Identifier | `chat123456` | 內部識別碼 |
| Service-prefixed | `iMessage:alice@icloud.com` | 明確指定 iMessage/SMS |

### 5.4 獨有特色

| 特色 | 說明 |
|------|------|
| **macOS 限定** | 需要 macOS + iMessage app |
| **Multi-Service** | iMessage + SMS 共用一個帳號 |
| **Region Support** | US, CN 等地區設定 |
| **Custom DB** | 可指定自訂 iMessage SQLite DB |
| **Reply Threading** | 文字標記 `[[reply_to:{id}]]`（非協定層） |
| **Handle 多樣性** | Phone / email / SMS / FaceTime Audio 識別 |
| **無 Typing / 無 Read Receipts** | 受 CLI 工具限制 |

---

## 6. IRC（extensions/irc/ — Raw Socket）

### 6.1 架構概覽

純 extension 實作，無 `src/irc/`。使用 Node.js `net`/`tls` 直接實作 IRC 協定。

```
extensions/irc/
├── index.ts              # Plugin registration
├── src/
│   ├── channel.ts        # ChannelPlugin 實作
│   ├── client.ts         # Raw IRC client（net/tls socket）
│   ├── protocol.ts       # RFC 3659 parser
│   ├── monitor.ts        # 連線監控 + 入站 handler
│   ├── inbound.ts        # 入站訊息 dispatch
│   ├── send.ts           # 出站訊息（PRIVMSG/NOTICE）
│   ├── accounts.ts       # Account config 解析
│   ├── policy.ts         # Access control gates
│   ├── normalize.ts      # Target/allowlist 正規化
│   ├── onboarding.ts     # 設定精靈
│   ├── connect-options.ts # 連線選項建構
│   ├── config-schema.ts  # Zod schema validation
│   ├── probe.ts          # 連線測試
│   ├── control-chars.ts  # IRC 控制字元清理
│   ├── types.ts          # 型別定義
│   └── runtime.ts        # Runtime context
```

### 6.2 連線機制

**Raw TCP/TLS Socket**

```
net.connect(port, host)    // Plaintext（port 6667）
tls.connect(port, host)    // Encrypted（port 6697 default）
    → PASS, NICK, USER     // 認證
    → JOIN #channel         // 加入頻道
    → PRIVMSG/NOTICE        // 收發訊息
    → PING/PONG             // 保活
```

**Connection timeout**: 15 seconds（configurable）

**NickServ 認證**：
- IDENTIFY with password
- REGISTER with email
- GHOST recovery（nick 衝突時）
- Fallback nick（加 `_` 後綴）

**Nick 衝突處理**（ERR 433/436）：
1. 有 password → GHOST old nick → 重新 NICK
2. 無 password → 附加 `_` suffix

### 6.3 訊息處理

| 方向 | IRC Command | 說明 |
|------|-------------|------|
| 入站 | PRIVMSG | DM 或 channel 訊息 |
| 出站 | PRIVMSG | 發送訊息（max **350 chars/line**，word-boundary aware） |
| 入站 | NOTICE | 系統通知（不觸發 auto-reply） |

**Control char stripping**：移除 IRC 控制碼（0x00-0x1f, 0x7f）

### 6.4 獨有特色

| 特色 | 說明 |
|------|------|
| **Mention Gating** | IRC channel 預設需 bot nick 提及（可 per-channel disable） |
| **Strict Name Matching** | 預設 `nick!user@host` 全匹配；可開啟 `dangerouslyAllowNameMatching` 僅匹配 nick |
| **350 char limit** | IRC 協定限制，word-boundary 斷句 |
| **純文字** | 無 buttons / cards / reactions |
| **Persistent TCP** | 每帳號一個長連線 |
| **No implicit group allowlist** | 不自動繼承 allowFrom → groupAllowFrom |

---

## 7. Google Chat（extensions/googlechat/ — Webhook + REST）

### 7.1 架構概覽

純 extension 實作，無 `src/googlechat/`。使用 Google Chat API v1。

```
extensions/googlechat/
├── index.ts              # Plugin registration
├── src/
│   ├── channel.ts        # ChannelPlugin 實作
│   ├── api.ts            # Google Chat REST client
│   ├── auth.ts           # Service Account OAuth 2.0 + JWT 驗證
│   ├── monitor.ts        # Webhook handler + 入站 dispatch
│   ├── monitor-webhook.ts # HTTP request parser
│   ├── monitor-access.ts # 入站 access policy
│   ├── actions.ts        # Message actions（send/react/list reactions）
│   ├── targets.ts        # Space/user 解析
│   ├── accounts.ts       # Account config 解析
│   ├── onboarding.ts     # 設定精靈
│   ├── types.ts          # API 型別定義
│   ├── types.config.ts   # Config 型別 re-export
│   ├── monitor-types.ts  # Runtime 型別
│   └── runtime.ts        # Runtime context
```

### 7.2 連線機制

**Webhook（入）+ REST API（出）**

| 方向 | 協定 | URL |
|------|------|-----|
| 入站 | HTTPS POST webhook | `/googlechat`（configurable） |
| 出站 | REST API | `https://chat.googleapis.com/v1` |
| 上傳 | Multipart REST | `https://chat.googleapis.com/upload/v1` |

### 7.3 認證

**Service Account OAuth 2.0**（`google-auth-library`）

```
Scope: https://www.googleapis.com/auth/chat.bot
Sources: Config（inline JSON）/ File path / Env vars
Cache: LRU（max 32 accounts in-memory）
```

**Webhook 驗證**（兩種模式）：

| 模式 | 驗證方式 |
|------|---------|
| App URL (ID Token) | `OAuth2Client.verifyIdToken()` → 檢查 issuer + email_verified |
| Project Number (JWT) | 下載 Google service account 公鑰 → 驗證 JWT signature |

公鑰快取 10 分鐘。支援 Google Workspace Add-on payload 格式轉換。

### 7.4 API 操作

```typescript
// 訊息
sendMessage(space, text, threadName?)
updateMessage(messageName, text)
deleteMessage(messageName)

// Reactions
addReaction(messageName, emoji)
removeReaction(reactionName)
listReactions(messageName)

// Media
uploadMedia(space, buffer, mimeType, filename)  // Multipart/related, max 20MB
downloadMedia(url)

// Space
findDirectMessage(userId)  // 查找 DM space
```

### 7.5 獨有特色

| 特色 | 說明 |
|------|------|
| **Thread Support** | 訊息可回覆特定 thread（fallback 新 thread） |
| **Reactions CRUD** | 完整 add/remove/list emoji reactions |
| **Media Upload** | Multipart/related encoding，max 20MB |
| **Workspace Add-on** | 同時支援標準 Chat API 和 Add-on payload 格式 |
| **Mention Detection** | 檢查 annotations 中 USER_MENTION type |
| **Bot Filtering** | 可排除 bot 訊息（configurable `allowBots`） |
| **Service Account Only** | 無 user OAuth；reactions 需 message mode |
| **4000 char limit** | 訊息長度上限 |

---

## 8. 九頻道總對照表

涵蓋 Discord（11-DISCORD-DEEP.md）、Telegram + Slack（12-TELEGRAM-SLACK.md）和本文六頻道。

### 連線與認證

| | Discord | Telegram | Slack | WhatsApp | Signal | LINE | iMessage | IRC | Google Chat |
|--|---------|----------|-------|----------|--------|------|----------|-----|-------------|
| **連線** | WebSocket | Webhook/Polling | Socket/HTTP | WebSocket (Baileys) | HTTP+SSE (CLI) | Webhook | CLI RPC | TCP/TLS | Webhook |
| **SDK** | @buape/carbon | grammy | @slack/bolt | @whiskeysockets/baileys | signal-cli | @line/bot-sdk | imsg CLI | Node net/tls | google-auth-library |
| **認證** | Bot Token/OAuth | Bot Token | OAuth 3-token | QR scan | Account file | Channel Token | DB access | NickServ/PASS | Service Account |
| **持久連線** | Yes | Polling: Yes | Socket: Yes | Yes | Yes (SSE) | No | Yes (child) | Yes | No |

### 訊息能力

| | Discord | Telegram | Slack | WhatsApp | Signal | LINE | iMessage | IRC | Google Chat |
|--|---------|----------|-------|----------|--------|------|----------|-----|-------------|
| **Text Limit** | 2000 | 4096 | 4000 | ~4090 | configurable | 5 msgs/call | unlimited | 350/line | 4000 |
| **Rich Content** | Embeds | HTML/Markdown | Blocks | — | Text styles | Flex/Template | — | — | Cards |
| **Media** | Full | Full | Limited | Full | Attachments | Image+preview | File path | — | Upload 20MB |
| **Reactions** | Full | — | Limited | Emoji | Emoji | — | — | — | Full CRUD |
| **Threading** | Thread bindings | Forum topics | thread_ts | — | — | — | Reply tag | — | threadName |
| **Typing** | Yes | Yes | Yes | Yes | Yes | — | — | — | — |
| **Read Receipts** | — | — | — | Yes | Yes | — | — | — | — |
| **Interactive** | Buttons/Modals | Inline buttons | Modals/Blocks | Polls | — | Quick Reply/Postback | — | — | Reactions |

### 規模

| | Discord | Telegram | Slack | WhatsApp | Signal | LINE | iMessage | IRC | Google Chat |
|--|---------|----------|-------|----------|--------|------|----------|-----|-------------|
| **Total Files** | ~175 | ~64 | ~84 | 48 | 20 | 34 | 22 | 17 | 15 |
| **Total LOC** | ~42.4K | ~15.1K | ~12.7K | ~6.7K | ~3.5K | ~7.3K | ~2.9K | ~2.9K | ~2.9K |

---

## 9. Extension Plugin 共通架構

所有九頻道都透過 `extensions/{channel}/` 註冊為 ChannelPlugin：

```typescript
// extensions/{channel}/index.ts
export default {
  id: "{channel}",
  name: "{Channel Name}",
  register(api: OpenClawPluginApi) {
    setRuntime(api.runtime);
    api.registerChannel({ plugin: channelPlugin });
  }
};

// extensions/{channel}/src/channel.ts
const channelPlugin: ChannelPlugin<ResolvedAccount> = {
  id: "channel-id",
  meta: { label, docsPath, ... },
  capabilities: {
    chatTypes: ["direct", "group"],     // + "thread" for some
    media: true/false,
    reactions: true/false,
    blockStreaming: true/false,
  },
  pairing: { idLabel, normalizeAllowEntry, notifyApproval },
  config: { listAccountIds, resolveAccount, defaultAccountId },
  configSchema: ChannelConfigSchema,
  onboarding: OnboardingAdapter,
  send: (outboundContext) => Promise<SendResult>,
  messageActions?: ChannelMessageActionAdapter,
  agentTools?: () => Tool[],             // WhatsApp: login tool
  reload?: { configPrefixes, gatewayMethods },
};
```

### runtime.ts 模式

每個 extension 有 `runtime.ts` 實作延遲初始化：

```typescript
let _runtime: OpenClawRuntime | undefined;
export function setRuntime(r: OpenClawRuntime) { _runtime = r; }
export function getRuntime(): OpenClawRuntime {
  if (!_runtime) throw new Error("...");
  return _runtime;
}
```

### 特殊 Extension 行為

| 頻道 | 特殊行為 |
|------|---------|
| WhatsApp | `agentTools`: createLoginTool()、Gateway methods: web.login.start/wait |
| LINE | `createLineCardCommand()` 註冊 card 指令 |
| Discord | `subagent-hooks.ts` thread 整合 |

---

## 10. Auto-Reply 整合比較

| 頻道 | Delivery 模組 | Chunking | 特殊能力 |
|------|-------------|----------|---------|
| WhatsApp | `auto-reply/deliver-reply.ts` | length/newline（~4090 chars） | Media 3x retry, suppress reasoning |
| Signal | `monitor.ts` deliverReplies | length/newline（configurable） | Text styles, multiple URLs |
| LINE | `auto-reply-delivery.ts` | 5 msgs/batch, Markdown→Flex | Reply token priority, quick replies |
| iMessage | 委託 extension | Text only | Reply tag threading |
| IRC | `send.ts` | 350 chars/line, word-boundary | Plain text only |
| Google Chat | `actions.ts` send | 4000 chars | Thread reply, reactions |

**共通流程**：所有頻道最終都透過 `dispatchInboundMessage()` 進入 auto-reply engine，取得 AI 回覆後再透過各自的 delivery 機制送出。

---

## 11. Access Control 比較

所有頻道實作三層防護：

| 層級 | 說明 |
|------|------|
| **DM Policy** | `pairing` / `allowlist` / `open` / `disabled` |
| **allowFrom** | 白名單（phone/UUID/userId/nick/space） |
| **Group Policy** | `open` / `allowlist` / `disabled` + mention gating |

### 頻道差異

| 頻道 | DM 識別 | Group 識別 | 特殊 |
|------|---------|-----------|------|
| WhatsApp | E.164 phone / JID | group JID (@g.us) | Pairing challenge 自動通知 |
| Signal | Phone / UUID 雙重 | groupId | Dual identity support |
| LINE | userId | groupId / roomId | Follow/unfollow 事件 |
| iMessage | Phone / email / handle | chat_id / chat_guid | Multi-service (iMessage/SMS) |
| IRC | nick!user@host 完整匹配 | #channel | `dangerouslyAllowNameMatching` 選項 |
| Google Chat | user ID | spaces/{space} | Case-sensitive space name |

---

## 12. 邊界條件與陷阱

### WhatsApp

1. **Baileys 非官方 API** — 隨時可能被 WhatsApp 封鎖或 API 變動
2. **QR 登入狀態** — session 可能被遠端 logout，需偵測 `loggedOut` 並重新掃碼
3. **Echo detection 需 broadcast-aware** — 多 agent 回覆時 combined body 不能自我 echo
4. **Watchdog 30 分鐘** — 長時間無訊息會觸發強制 reconnect
5. **Group JID 格式** — `{timestamp1}-{timestamp2}@g.us`，非 phone number

### Signal

6. **signal-cli daemon 必須預裝** — 不自帶，需系統安裝 Java + signal-cli
7. **Dual identity** — 同一 sender 可能以 phone 或 UUID 出現，allowlist 需兩者都列
8. **SSE 連線中斷** — daemon crash 或 HTTP error 需 auto-reconnect
9. **Attachment 下載** — 透過 RPC getAttachment，需確保 daemon 未設 `--ignore-attachments`

### LINE

10. **Reply token 3 分鐘過期** — 過期後只能 push（需額外配額）
11. **每次 API 最多 5 則** — 超過自動分批，但 reply token 只能用一次
12. **Flex message alt text** — 限 400 字元，影響推播通知顯示
13. **Quick reply labels case-sensitive** — 大小寫不同視為不同按鈕
14. **Replay cache 10 分鐘** — 超過窗口的重複 webhook 無法過濾

### iMessage

15. **macOS 限定** — 需 macOS + iMessage app + imsg CLI 工具
16. **Reply tag 文字層級** — `[[reply_to:{id}]]` 是 text prepend，非協定層 threading
17. **Service 自動偵測不完美** — email 可能是 iMessage 或普通 mail
18. **無 typing / 無 read receipts** — CLI 工具限制

### IRC

19. **350 char/line limit** — IRC 協定硬限，過長訊息被截斷
20. **Nick collision** — 多實例用同一 nick 會互踢
21. **NickServ 認證時序** — IDENTIFY 需在 JOIN 之前，否則可能被 kick
22. **純文字** — 無法發送任何 rich content

### Google Chat

23. **Service Account only** — 無 user OAuth，某些操作受限
24. **Webhook 驗證公鑰 10 分鐘快取** — 金鑰輪替時可能短暫驗證失敗
25. **Mention 偵測** — 需精確匹配 annotations 中的 USER_MENTION（非文字比對）
26. **Space name case-sensitive** — 大小寫不同視為不同 space

---

## 13. 關鍵常量速查

### WhatsApp

| 常量 | 值 | 來源 |
|------|------|------|
| textChunkLimit (default) | ~4090 chars | accounts.ts |
| mediaMaxMb (default) | configurable | account config |
| heartbeat interval | 60s | auto-reply/monitor.ts |
| watchdog check interval | 60s | auto-reply/monitor.ts |
| watchdog message timeout | 30 min | auto-reply/monitor.ts |
| echo LRU size | configurable | monitor/echo.ts |

### Signal

| 常量 | 值 | 來源 |
|------|------|------|
| daemon HTTP default | 127.0.0.1:8080 | daemon.ts |
| mediaMaxMb (default) | 8 MB | accounts.ts |
| SSE endpoint | /api/v1/events | client.ts |
| health check | /api/v1/check | daemon.ts |

### LINE

| 常量 | 值 | 來源 |
|------|------|------|
| messages per API call | 5 max | send.ts |
| reply token TTL | ~3 min | LINE Platform |
| replay cache window | 10 min | monitor.ts |
| replay cache size | 4096 entries | monitor.ts |
| flex alt text limit | 400 chars | LINE Platform |
| quick reply max buttons | 13 | LINE Platform |
| loading animation duration | 20s | LINE Platform |

### iMessage

| 常量 | 值 | 來源 |
|------|------|------|
| probe timeout | configurable | constants.ts |
| RPC protocol | JSON-RPC 2.0 | client.ts |

### IRC

| 常量 | 值 | 來源 |
|------|------|------|
| message chunk limit | 350 chars | send.ts |
| connection timeout | 15s | client.ts |
| TLS default port | 6697 | connect-options.ts |
| plaintext default port | 6667 | connect-options.ts |

### Google Chat

| 常量 | 值 | 來源 |
|------|------|------|
| textChunkLimit | 4000 chars | channel.ts |
| media max upload | 20 MB (default) | api.ts |
| auth cache max | 32 accounts | auth.ts |
| cert cache TTL | 10 min | auth.ts |
| API base | chat.googleapis.com/v1 | api.ts |
| OAuth scope | googleapis.com/auth/chat.bot | auth.ts |

---

## 14. C# 概念對照

| OpenClaw 概念 | C# / .NET 近似 | 說明 |
|---------------|---------------|------|
| Baileys WebSocket | SignalR Client | 持久雙向連線 |
| SSE stream | EventSource / IAsyncEnumerable | Server-Sent Events |
| JSON-RPC 2.0 (iMessage) | StreamJsonRpc | stdin/stdout 雙向 RPC |
| Webhook + HMAC validation | ASP.NET Core Middleware + HMACSHA256 | 請求驗證 |
| signal-cli daemon spawn | Process.Start + EventWaitHandle | 子程序管理 |
| @line/bot-sdk MessagingApiClient | HttpClient + typed DTOs | REST API wrapper |
| google-auth-library GoogleAuth | Google.Apis.Auth ServiceAccountCredential | OAuth 2.0 |
| net.connect / tls.connect | TcpClient / SslStream | Raw socket |
| IRC PRIVMSG chunking | StreamWriter + word-boundary split | 協定限制下的分段 |
| ChannelPlugin<T> | IPlugin<T> interface | 通道抽象 |
| ResolvedAccount pattern | Options pattern (IOptions<T>) | 設定解析 |
| LRU echo cache | MemoryCache with SizeLimit | 去重快取 |
| Replay cache (LINE) | IDistributedCache | 冪等保護 |
| Flex Message JSON | Razor Components / Blazor | UI 描述語言 |
| Runtime lazy init | Lazy<T> + DI scope | 延遲初始化 |
