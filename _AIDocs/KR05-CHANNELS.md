# 05-CHANNELS — 各平台 Channel 完整細節

> 來源：openclaw-knowledge-base.md §9 + F058 §11 + F060 各 extension 章節

---

## 1. Channel Plugin 架構

### registerChannel 介面

```typescript
// 每個 Channel Extension 必須實作 registerChannel
function registerChannel(api: ExtensionApi): void {
  api.registerChannel({
    id: "platform-name",          // 唯一識別符（如 "discord", "telegram"）
    displayName: "Platform Name", // 顯示名稱
    configSchema: TypeBoxSchema,  // 配置 TypeBox schema
    createRuntime: (config, api) => ChannelRuntime, // runtime 工廠
    icon?: string,                // 圖示（可選）
  });
}
```

### Channel Config 通用型別

```typescript
interface BaseChannelConfig {
  accountId: string;              // 帳號 ID
  dmPolicy?: DmPolicy;            // DM 政策
  groupPolicy?: GroupPolicy;      // 群組政策
  sessionScope?: SessionScope;    // Session 範圍
  allowFrom?: string[];           // 允許來源白名單
  allowBots?: "off" | "mentions" | "all"; // Bot 訊息政策
  injectHistory?: boolean;        // 是否注入歷史訊息
  streamingMode?: "partial" | "block" | "off"; // Streaming 模式
  timeout?: number;               // 回應超時（ms）
}

type DmPolicy    = "open" | "pairing" | "disabled";
type GroupPolicy = "open" | "allowlist" | "disabled";
type SessionScope = "channel" | "user" | "thread";
```

### CHAT_CHANNEL_ORDER（載入順序）

```typescript
// Channel 載入優先順序（越前面越優先處理）
const CHAT_CHANNEL_ORDER = [
  "discord", "telegram", "whatsapp", "signal", "imessage",
  "slack", "matrix", "mattermost", "msteams", "irc",
  "line", "feishu", "googlechat", "nextcloud-talk",
  "twitch", "nostr", "tlon", "zalo", "zalouser",
  "synology-chat", "bluebubbles"
];
```

---

## 2. WhatsApp

### Self-Chat Mode 判斷邏輯

```typescript
// WhatsApp self-chat 模式：使用者傳訊息給自己（作為筆記/指令）
function isSelfChatMode(message: WhatsAppMessage, account: WhatsAppAccount): boolean {
  // 判斷條件：
  // 1. 訊息 sender JID === account JID（自己傳給自己）
  // 2. 或 message.key.fromMe === true
  return message.key.remoteJid === account.jid
    || message.key.fromMe === true;
}
```

### isSelfChatMode Config

```typescript
interface WhatsAppConfig extends BaseChannelConfig {
  selfChatMode?: boolean;    // 啟用 self-chat 模式
  // 當 selfChatMode=true：
  // - 只處理自己傳給自己的訊息
  // - allowFrom="*" 選項被停用（安全考量）
}
```

### allowFrom="*" 停用規則

```typescript
// WhatsApp 中，若 selfChatMode=true，allowFrom="*" 被強制覆蓋
// 只允許 self-chat JID，防止被任意使用者控制
```

### Group Policy

| 政策 | 行為 |
|------|------|
| `"open"` | 接受所有群組訊息 |
| `"allowlist"` | 只接受 allowFrom 白名單中的群組 JID |
| `"disabled"` | 不處理群組訊息 |

### Group JID 繞過 allowFrom

```typescript
// WhatsApp Group JID 特殊處理：
// normalize.ts 中，群組 JID 後綴為 "@g.us"
// resolveWhatsAppOutboundTarget() 處理外送目標解析
// Group JID 可以繞過 allowFrom（透過特殊 policy 配置）
```

---

## 3. Telegram

### 特性

```typescript
// Telegram extension 特性：
// - 支援多帳號（createScopedChannelConfigBase + 多帳號模式）
// - Persistent bindings（ACP 支援）
// - Inline keyboard / callback query 支援
// - Media download 整合
// - Webhook 與 polling 兩種模式
```

### 配置

```typescript
interface TelegramConfig extends BaseChannelConfig {
  token: string;             // Bot token
  mode: "webhook" | "polling";
  webhookUrl?: string;       // Webhook URL（mode=webhook 時必填）
  pollIntervalMs?: number;   // Polling 間隔（mode=polling 時）
  allowedUpdates?: string[]; // 允許的 update 類型
}
```

---

## 4. Signal

### SignalEnvelope / SignalDataMessage / SignalAttachment 型別

```typescript
interface SignalEnvelope {
  source: string;            // 發送者電話號碼
  sourceDevice: number;      // 裝置編號
  timestamp: number;         // 時間戳
  message?: SignalDataMessage;
  syncMessage?: SignalSyncMessage;
}

interface SignalDataMessage {
  body?: string;             // 訊息文字
  attachments?: SignalAttachment[];
  groupInfo?: {
    groupId: string;
    type: "DELIVER" | "UPDATE" | "QUIT" | "REQUEST_INFO";
  };
  reaction?: SignalReactionMessage;
}

interface SignalAttachment {
  contentType: string;       // MIME type
  filename?: string;
  data?: Buffer;             // 附件內容
}
```

---

## 5. iMessage

### 特性（src/imessage/）

```typescript
// iMessage 子模組：
// deliver.ts      → 訊息傳送
// reflection-guard.ts → 防止迴聲（自己的訊息不觸發 Agent）
// echo-cache.ts   → 已傳送訊息快取（避免重複回應）
// monitor.ts      → 監聽新訊息
// channel.ts      → Channel 實作
// runtime.ts      → Runtime 管理

interface IMessageAttachment {
  mimeType: string;
  filename?: string;
  data: Buffer;
}

interface IMessagePayload {
  handle: string;            // iMessage handle（電話/email）
  text?: string;
  attachments?: IMessageAttachment[];
  isGroup: boolean;
  groupId?: string;
}
```

---

## 6. IRC

```typescript
// IRC extension
// CHANNEL_ID = "irc"
// control-chars.ts → 過濾 IRC 控制字元（顏色碼/粗體等）
// 特性：
// - 支援多頻道
// - CTCP 支援
// - NickServ 認證
```

---

## 7. Matrix

```typescript
interface MatrixConfig extends BaseChannelConfig {
  homeserverUrl: string;        // Matrix homeserver URL
  accessToken: string;          // 存取 token
  encryption?: boolean;         // 啟用 E2E 加密（需 libolm）
  autoJoin?: boolean;           // 自動接受房間邀請
  threadReplies?: boolean;      // 在 thread 中回覆
  chunkMode?: "default" | "block"; // Chunk 模式
}
```

---

## 8. Mattermost

```typescript
// MattermostChatMode：
type MattermostChatMode = "oncall" | "onmessage" | "onchar";
// oncall:    只有被呼叫時才回應
// onmessage: 每則訊息都回應
// onchar:    每個字元觸發（streaming 模式）

// 配置使用 TypeBox schema 驗證
```

---

## 9. MS Teams

```typescript
interface StoredConversationReference {
  // 儲存 MS Teams 對話引用，用於主動傳送訊息
  serviceUrl: string;
  channelId: string;
  conversationId: string;
  activityId?: string;
}

interface MSTeamsConversationStore {
  // 持久化 StoredConversationReference
  // 使用 file-lock.ts 避免並發寫入衝突
}
```

---

## 10. Nextcloud Talk

```typescript
// Nextcloud Talk 特性：
// - HMAC-SHA256 簽章驗證（Webhook 安全）
// - 預設 port: 8788
// - form-urlencoded webhook payload

interface NextcloudTalkWebhookPayload {
  token: string;             // 房間 token
  message: string;           // 訊息內容
  actor: {
    type: string;
    id: string;
    displayName: string;
  };
  timestamp: number;
}
```

---

## 11. Feishu（飛書）

```typescript
// Feishu（Lark）整合特性：
// CHAT_ACTION_VALUES / MEMBER_ID_TYPE_VALUES 常數
// 支援 Drive / Wiki 整合
// open_id / union_id / user_id 三種 ID 格式
```

---

## 12. Google Chat

```typescript
interface GoogleChatMessage {
  name: string;
  sender: { name: string; displayName: string; };
  text?: string;
  space: { name: string; type: "ROOM" | "DM"; };
}

interface GoogleChatEvent {
  type: "MESSAGE" | "ADDED_TO_SPACE" | "REMOVED_FROM_SPACE" | "CARD_CLICKED";
  message?: GoogleChatMessage;
}
```

---

## 13. LINE

```typescript
// LINE extension 特性：
// meta.id = "line"
// LINE 特有：allowFrom 前綴 strip（LINE userId 格式處理）
// template-messages/channel-access-token 子模組
// flex-templates/ 目錄（Flex Message 模板）
```

---

## 14. Slack

```typescript
// Slack extension（F058 §11 驗證）
// 支援 Socket Mode 與 Event API
// 支援 Block Kit（message 格式）
// slash command 整合
// 多 workspace 支援
```

---

## 15. Twitch

```typescript
// Twitch extension：
// ChannelPlugin<TwitchAccountConfig>
// 子模組：
// client-manager-registry  → Twitch client 管理
// resolver                 → 頻道/用戶解析
// token                    → OAuth token 管理
```

---

## 16. Nostr

```typescript
interface NostrAccountConfig {
  privateKey: string;        // Nostr 私鑰（hex 或 nsec 格式）
  relays?: string[];         // 中繼伺服器列表
}

interface ResolvedNostrAccount {
  privateKey: string;
  publicKey: string;         // 從私鑰派生
  npub: string;              // Nostr 公鑰（npub 格式）
}

// DEFAULT_RELAYS：預設中繼伺服器列表
const DEFAULT_RELAYS = [
  "wss://relay.damus.io",
  "wss://relay.nostr.info",
  // ...
];
```

---

## 17. Tlon（Urbit）

```typescript
interface MonitorTlonOpts {
  ship: string;              // Urbit ship name（~sampel-palnet）
  url: string;               // Urbit URL
  code: string;              // 登入碼
}

interface ChannelAuthorization {
  groups?: string[];         // 允許的 group 列表
}

// Urbit SSE（Server-Sent Events）監聽
// normalizeShip() → 正規化 ship name
// parseChannelNest() → 解析 channel nest 格式（group/type/name）
```

---

## 18. Zalo vs ZaloUser 差異

| 項目 | `zalo`（官方） | `zalouser`（非官方） |
|------|-------------|-------------------|
| API | zalo Official API | 非官方 Zalo.js |
| 風險 | 低 | 高（違反 ToS，帳號可能被封） |
| Webhook | 必須 HTTPS | N/A |
| 功能 | OA（Official Account） | 個人帳號 |

### ZaloUser ACTIONS 清單

```typescript
const ZALO_USER_ACTIONS = [
  "send",    // 傳送訊息
  "image",   // 傳送圖片
  "link",    // 傳送連結
  "friends", // 獲取好友列表
  "groups",  // 獲取群組列表
  "me",      // 獲取個人資訊
  "status",  // 獲取狀態
];
```

---

## 19. Synology Chat

```typescript
interface ResolvedSynologyChatAccount {
  webhookUrl: string;        // Synology Chat Webhook URL
  token: string;             // 驗證 token
  secret?: string;           // HMAC-SHA256 簽章密鑰（可選）
}

interface SynologyWebhookPayload {
  token: string;
  user_id: number;
  username: string;
  post_id: number;
  timestamp: number;
  text: string;
}

// 特性：
// - form-urlencoded payload（非 JSON）
// - HMAC-SHA256 簽章驗證
```

---

## 20. BlueBubbles

```typescript
// BlueBubbles（macOS iMessage bridge）特性：
const REPLY_CACHE_MAX = 2000;       // 回覆快取最大條目數
const REPLY_CACHE_TTL  = 6 * 60 * 60 * 1000; // 6 小時（ms）

// 特性：
// - 作為 iMessage 的 bridge（macOS 必要）
// - REST API 通訊
// - 支援 Group chat
```

---

## 21. allow-list.ts 邏輯

```typescript
// allow-list.ts：通用 allowFrom 驗證
function checkAllowList(
  entry: string,           // 待驗證的值（userId/groupId/etc）
  allowFrom: string[],     // 配置的白名單
  prefix: string           // 平台前綴（如 "discord:user:"）
): boolean {
  // 1. 若 allowFrom 包含 "*" → 允許所有
  // 2. 若 allowFrom 包含 `!{entry}` → 拒絕（黑名單優先）
  // 3. 若 allowFrom 包含 `{prefix}{entry}` → 允許
  // 4. 若 allowFrom 包含 `{entry}`（無前綴）→ 允許
  // 5. 其他 → 拒絕
}
```

---

## 22. thread-bindings-policy.ts 常數

```typescript
// 各平台 Thread Binding 政策常數
const THREAD_BINDING_POLICY = {
  discord: {
    idleTimeoutMs:      24 * 60 * 60 * 1000,  // 24h
    sweepIntervalMs:    2 * 60 * 1000,          // 2min
    echoWindowMs:       30 * 1000,             // 30s
    touchPersistMs:     15 * 1000,             // 15s
    mentionTtlMs:       /* 依配置 */,
  },
  telegram: {
    // Telegram thread binding 政策
  },
};
```
