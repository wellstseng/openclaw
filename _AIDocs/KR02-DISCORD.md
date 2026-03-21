# 02-DISCORD — Discord 整合完整細節

> 來源：openclaw-knowledge-base.md §4 §6 + F058 §5 §6 §7 §8 + F060 discord extension

---

## 1. Discord Preflight 40 個閘門（完整順序）

### 閘門執行架構

每個 async 閘門執行後，立即檢查 `isPreflightAborted(abortSignal)`，任一閘門 abort 即短路後續。

### A 群：Identity 驗證

| # | 閘門名稱 | 說明 |
|---|---------|------|
| 1 | `A1_IDENTITY_ACCOUNT` | 驗證 Discord account 存在且已配置 |
| 2 | `A2_IDENTITY_BOT` | 驗證 Bot identity 已解析 |
| 3 | `A3_IDENTITY_CHANNEL` | 驗證頻道 ID 有效 |

### B 群：Basic Filter

| # | 閘門名稱 | 說明 |
|---|---------|------|
| 4 | `B1_OWN_MESSAGE` | 過濾 Bot 自身訊息（在 debounce 前執行，避免佔用 debounce 容量，issue #15874） |
| 5 | `B2_MISSING_CONTENT` | 過濾空訊息（無文字、無媒體） |
| 6 | `B3_SLASH_COMMAND` | 過濾 Slash command interaction（另有處理路徑） |
| 7 | `B4_SYSTEM_MESSAGE` | 過濾系統訊息（pin/join 等） |

### C 群：Channel Type 判斷

| # | 閘門名稱 | 說明 |
|---|---------|------|
| 8 | `C1_DM_CHANNEL` | DM 頻道處理路徑 |
| 9 | `C2_GUILD_CHANNEL` | Guild 文字頻道處理路徑 |
| 10 | `C3_THREAD_CHANNEL` | Thread（公開/私有）處理路徑 |
| 11 | `C4_FORUM_CHANNEL` | Forum channel 處理路徑 |
| 12 | `C5_VOICE_CHANNEL` | Voice channel 訊息處理路徑 |

### D 群：Guild Access 控制

| # | 閘門名稱 | 說明 |
|---|---------|------|
| 13 | `D1_GUILD_ALLOWLIST` | Guild ID 白名單驗證 |
| 14 | `D2_GUILD_POLICY` | Guild 政策驗證（open/allowlist/disabled） |
| 15 | `D3_GUILD_ROLE` | Role 授權驗證 |

### E 群：DM Auth

| # | 閘門名稱 | 說明 |
|---|---------|------|
| 16 | `E1_DM_POLICY` | DM 政策驗證（open/pairing/disabled） |
| 17 | `E2_DM_PAIRING` | DM pairing 驗證（已 pair 的使用者） |

### F 群：Mention 驗證

| # | 閘門名稱 | 說明 |
|---|---------|------|
| 18 | `F1_MENTION_EXPLICIT` | 直接 @mention Bot |
| 19 | `F2_MENTION_AUDIO` | 語音頻道 mention 規則 |
| 20 | `F3_MENTION_GATING` | Mention gating 政策（是否需要 @） |
| 21 | `F4_MENTION_IMPLICIT` | 隱式 mention（如 reply thread） |

### G 群：Bot Filter

| # | 閘門名稱 | 說明 |
|---|---------|------|
| 22 | `G1_ALLOW_BOTS` | allowBots 政策驗證（off/mentions/all） |
| 23 | `G2_PLURAL_KIT` | PluralKit 代理訊息特殊處理（豁免 bot filter） |

### H 群：Thread Binding

| # | 閘門名稱 | 說明 |
|---|---------|------|
| 24 | `H1_THREAD_RESOLVE` | 解析 thread binding 狀態 |
| 25 | `H2_THREAD_ECHO` | 過濾 echo 訊息（unbind 後 30s 內） |
| 26 | `H3_THREAD_SYSTEM` | Thread 系統訊息過濾 |

### I 群：Route & Media

| # | 閘門名稱 | 說明 |
|---|---------|------|
| 27 | `I1_ROUTE_RESOLVE` | 解析最終 route（agent 路由） |
| 28 | `I2_MEDIA_DOWNLOAD` | 媒體附件下載與驗證 |
| 29 | `I3_EMPTY_AFTER_MEDIA` | 移除媒體後如果內容為空則 abort |

### J 群：History & Context

| # | 閘門名稱 | 說明 |
|---|---------|------|
| 30 | `J1_HISTORY_FETCH` | 獲取頻道歷史訊息（injectHistory） |
| 31 | `J2_CONTEXT_FINALIZE` | 最終化 InboundContext |

### 其餘 9 個閘門（擴充）

| # | 閘門名稱 | 說明 |
|---|---------|------|
| 32 | `K1_ALLOW_FROM` | allowFrom 白名單/黑名單過濾 |
| 33 | `K2_RATE_LIMIT` | 用戶級 rate limit |
| 34 | `K3_CONTENT_FILTER` | 內容過濾（違禁詞等） |
| 35 | `L1_SESSION_CHECK` | Session 狀態驗證 |
| 36 | `L2_SESSION_LOCK` | Session 並發鎖 |
| 37 | `M1_WORKER_CAPACITY` | Worker queue 容量檢查 |
| 38 | `M2_DEBOUNCE_GATE` | Debounce 視窗控制 |
| 39 | `N1_DRAFT_CLEAN` | 清理舊 draft 訊息 |
| 40 | `N2_PRE_DISPATCH` | 最終 dispatch 前的驗證 |

---

## 2. Reply Delivery 機制

### DISCORD_DELIVERY_RETRY_DEFAULTS

```typescript
const DISCORD_DELIVERY_RETRY_DEFAULTS = {
  maxRetries: 3,
  initialDelayMs: 1000,     // 1s
  maxDelayMs: 30000,         // 30s
  multiplier: 2.0,           // exponential backoff
  jitter: true,              // 加入隨機抖動
};
```

### retry 流程

```
嘗試 1 → 失敗 → 等待 1s(±jitter) →
嘗試 2 → 失敗 → 等待 2s(±jitter) →
嘗試 3 → 失敗 → 等待 4s(±jitter)（capped at 30s）→
最終失敗 → 記錄錯誤
```

### 429 處理

```typescript
// 遇到 429 時讀取 Retry-After header
const retryAfterMs = parseInt(headers["retry-after"] ?? "1") * 1000;
await sleep(retryAfterMs);
```

### Webhook Persona 優先

```typescript
// 傳送優先順序：
// 1. Webhook persona（binding.webhookId + binding.webhookToken）
//    → 可自訂名稱/頭像
// 2. Bot 直接傳送（fallback，靜默失敗不 throw）
```

### Markdown Table Mode

```typescript
// 當訊息包含 markdown table 時切換模式
// Discord 不支援 markdown table → 轉為 code block 或其他格式
type MarkdownTableMode = "keep" | "code_block" | "csv";
```

---

## 3. Chunk 機制

### chunkDiscordTextWithMode() 函式簽名

```typescript
function chunkDiscordTextWithMode(
  text: string,
  mode: ChunkMode,
  options?: {
    maxChars?: number;   // 預設 2000
    maxLines?: number;   // 預設 17
  }
): string[];
```

### 常數

| 常數 | 值 |
|------|----|
| `DEFAULT_MAX_CHARS` | 2000 |
| `DEFAULT_MAX_LINES` | 17 |

### Chunk 切割邏輯

```
1. 優先在自然斷點切割：段落（\n\n）> 行（\n）> 句子（. ）> 字元
2. Code fence 跨 chunk 平衡：自動補開/補關 ```
3. Reasoning italics rebalance：確保 * 配對正確
4. 最終確保每個 chunk ≤ DEFAULT_MAX_CHARS 且 ≤ DEFAULT_MAX_LINES
```

### ChunkMode

```typescript
type ChunkMode = "default" | "table_aware" | "code_aware";
```

---

## 4. Thread Binding

### 8 個相關檔案

| 檔案 | 職責 |
|------|------|
| `thread-bindings.ts` | 核心 ThreadBindingManager，state maps |
| `thread-bindings-policy.ts` | idle timeout / maxAge / autoArchive 常數 |
| `thread-bindings-store.ts` | 持久化（JSON 讀寫） |
| `thread-bindings-sweep.ts` | 定時 sweep（每 120s） |
| `thread-bindings-webhook.ts` | Webhook 重用邏輯 |
| `thread-bindings-echo.ts` | Echo 抑制（30s 視窗） |
| `thread-bindings-migrate.ts` | 舊格式遷移（expiresAt → maxAgeMs） |
| `thread-bindings-types.ts` | 所有型別定義 |

### State Maps（儲存於 globalThis）

```typescript
// 使用 globalThis 解決 Plugin Jiti/CJS vs Core ESM 模組實例不一致問題
const BINDINGS_BY_THREAD_ID:   Map<string, ThreadBinding>
const BINDINGS_BY_SESSION_KEY: Map<string, ThreadBinding>
const REUSABLE_WEBHOOKS:       Map<string, WebhookInfo>   // key: account:channel
const RECENT_UNBOUND_ECHOES:   Set<string>
```

### 重要常數

| 常數 | 值 | 說明 |
|------|----|------|
| `MENTION_TTL_MS` | 依配置 | mention TTL |
| Sweep interval | 120s | 定時清理 |
| Idle timeout | 24h（預設） | 閒置超時 |
| Echo suppression | 30s | unbind 後抑制視窗 |
| Touch persist debounce | 15s | 持久化限速 |

### Sweep 清理條件（任一滿足）

1. idle timeout（閒置 > 24h）
2. maxAge 超過
3. Thread 已 archived
4. Thread 已刪除（HTTP 404/403）

### 遷移規則

```typescript
// 舊格式 → 新格式
binding.expiresAt → binding.maxAgeMs = expiresAt - boundAt
```

---

## 5. allowBots 三種模式

| 模式 | 行為 |
|------|------|
| `"off"` | 拒絕所有 Bot 訊息（包含 Webhook） |
| `"mentions"` | 只接受 @mention Bot 的 Bot 訊息 |
| `"all"` | 接受所有 Bot 訊息 |

---

## 6. allowFrom 配置邏輯

```typescript
// allowFrom 格式（支援前綴符號）：
// "user:123456"   → 允許特定使用者 ID
// "role:admin"    → 允許特定 Role 名稱
// "guild:789"     → 允許特定 Guild
// "*"             → 允許所有（WhatsApp self-chat 模式停用此選項）
// "!user:123"     → 排除特定使用者（黑名單語法）

// 驗證順序：
// 1. 黑名單（!前綴）→ 拒絕
// 2. 白名單 → 允許
// 3. 預設政策（open/allowlist）
```

### allow-list.ts 邏輯

```typescript
// allow-list.ts 支援的前綴清單（Discord 特有）：
// "discord:user:" → 使用者 ID
// "discord:role:" → Role ID
// "discord:guild:" → Guild ID
// "discord:channel:" → 頻道 ID
```

---

## 7. Bot Message 處理規則

```
1. Bot self-message（自身）→ B1 閘門過濾（最早期，debounce 前）
2. 其他 Bot 訊息：
   - allowBots="off" → G1 閘門過濾
   - allowBots="mentions" → 需有 @mention → F1 閘門驗證
   - allowBots="all" → G1 閘門放行
3. Webhook 訊息：
   - 視同 Bot 處理
   - Echo suppression 視窗內的 Webhook → H2 閘門過濾
```

---

## 8. PluralKit 特殊處理

```typescript
// PluralKit 是 Discord 多重人格系統 Bot
// 問題：PluralKit 代理發送的訊息 author 是 Bot，但實際是人類用戶
// 解決：G2_PLURAL_KIT 閘門豁免 PluralKit 的 Bot filter
// 識別方式：訊息帶有 PluralKit 的特殊欄位或 application_id
```

---

## 9. Discord Subagent Hooks

```typescript
// Discord Extension 提供以下 subagent hooks：
// "discord:before_dispatch"  → Agent dispatch 前觸發
// "discord:after_delivery"   → 訊息傳送後觸發
// "discord:on_thread_create" → Thread 建立時觸發
// "discord:on_thread_bind"   → Thread binding 時觸發
```

---

## 10. Guild / Channel / Thread 路由差異

| 場景 | Session Key 格式 | 說明 |
|------|----------------|------|
| DM | `dm:{accountId}:{userId}` | 一對一私訊 |
| Guild 頻道 | `channel:{accountId}:{channelId}` | 公開頻道 |
| Thread | `channel:{accountId}:{threadId}` | Thread 視為獨立頻道 |
| Forum | `channel:{accountId}:{postId}` | Forum post 視為 Thread |

### autoArchiveDuration 支援值

```typescript
type AutoArchiveDuration = 60 | 1440 | 4320 | 10080; // 分鐘
// 60    = 1 小時
// 1440  = 1 天
// 4320  = 3 天
// 10080 = 1 週
```

---

## 11. Draft Streaming 三種模式

| 模式 | 行為 |
|------|------|
| `"partial"` | 即時 Edit 更新，1200ms throttle |
| `"block"` | EmbeddedBlockChunker 分塊感知 |
| `"off"` | 無預覽，完成後一次傳送 |

### 最終化邏輯

```typescript
// Draft 最終化判斷：
if (length <= 2000 && isPureText && !hasMedia) {
  // edit draft message → 最終版本
} else {
  // 清除 draft + 傳送新訊息
}
```

---

## 12. 外部庫依賴

```typescript
// Discord 系統主要依賴：
import { Client, RequestClient, GatewayPlugin, Commands } from "@buape/carbon";
import { ... } from "discord-api-types";  // REST/Gateway/Voice 型別
import { ... } from "@discordjs/voice";  // Voice connection + audio player
import { WebSocket } from "ws";           // WebSocket
import { fetch } from "undici";           // HTTP
import { HttpsProxyAgent } from "https-proxy-agent"; // Proxy 支援
```

---

## 13. Voice 相關常數

| 常數 | 值 |
|------|----|
| 音訊規格 | 48kHz stereo 16bit |
| DAVE E2E 加密 | 支援 |
| Decrypt failure tolerance | 24 次 / 30s → reconnect |
| Voice message format | OGG/Opus |
| Waveform 解析度 | 256 點 |
| Voice message flag | 8192 |
