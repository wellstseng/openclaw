# 03-SESSIONS-ACP — Sessions 與 ACP 協定

> 來源：openclaw-knowledge-base.md §5 §12 + F058 §4 + F059 §8

---

## 1. Session Key 所有格式

### 完整格式表

| 格式 | 範例 | 使用場景 |
|------|------|---------|
| `agent:{agentId}` | `agent:my-bot` | Agent 級別 session（無 channel） |
| `dm:{accountId}:{userId}` | `dm:acc1:user123` | Discord/Telegram DM |
| `channel:{accountId}:{channelId}` | `channel:acc1:ch456` | 平台頻道 session |
| `group:{accountId}:{groupId}` | `group:acc1:grp789` | 群組訊息 session |
| `thread:{accountId}:{threadId}` | `thread:acc1:thr012` | Thread session |
| `acp:{hash16}` | `acp:a1b2c3d4e5f6g7h8` | ACP session（SHA256 hash） |
| `subagent:{parentKey}:{depth}:{nonce}` | `subagent:channel:acc1:ch456:1:x9y8` | 子 Agent session |

### Session Key 規則

```typescript
// 1. Session Key 必須唯一識別一個對話
// 2. Key 不含敏感資訊（user ID 除外）
// 3. ACP session key 額外 hash 以縮短長度
// 4. Subagent key 包含 depth 防止無限遞迴
```

---

## 2. ACP Session Key 算法

```typescript
// 完整算法（F059 §8 驗證）
function generateAcpSessionKey(
  channel: string,
  accountId: string,
  conversationId: string
): string {
  const input = `${channel}:${accountId}:${conversationId}`;
  const hash = crypto.createHash("sha256").update(input).digest("hex");
  return hash.slice(0, 16);  // 取前 16 個 hex 字元
}

// 範例：
// input:  "discord:bot123:thread456"
// sha256: "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"
// result: "a1b2c3d4e5f6g7h8"（前 16 位 hex）
```

---

## 3. ACP Persistent Bindings

### 支援平台（只有兩個）

| 平台 | 支援 |
|------|------|
| `discord` | ✓ |
| `telegram` | ✓ |
| 其他所有平台 | ✗ |

### ConfiguredAcpBindingSpec 型別（完整 7 個欄位）

```typescript
type ConfiguredAcpBindingSpec = {
  channel: string;          // 平台名稱（"discord" | "telegram"）
  accountId: string;        // 帳號 ID
  conversationId: string;   // 對話/頻道 ID
  agentId: string;          // 目標 Agent ID
  sessionScope?: string;    // session 範圍（可選）
  expiresAt?: number;       // 過期時間（Unix timestamp，可選）
  metadata?: Record<string, unknown>; // 附加資料（可選）
};
```

### AcpBindingConfigShape

```typescript
// AcpBindingConfigShape 是 TypeBox schema，用於配置驗證
// 包含 ConfiguredAcpBindingSpec 的所有欄位 + 驗證規則
```

---

## 4. ACP Lifecycle 完整流程

### 建立（Create）

```
1. 接收 ACP 請求（來自外部 Agent 或 Gateway）
2. 驗證 ACP token
3. 生成 session key（SHA256 hash 前 16 位）
4. 建立新 Session（sessions.json 新增 entry）
5. 建立 transcript JSONL 檔
6. 初始化 ContextEngine
7. 回傳 { sessionKey, sessionId }
```

### Resume（恢復）

```
1. 接收帶有 sessionKey 的 ACP 請求
2. 查找 sessions.json 中的 entry
3. 驗證 session 未過期（TTL 檢查）
4. 載入 transcript JSONL
5. 恢復 ContextEngine 狀態
6. 繼續對話
```

### Cleanup（清理）

```
1. Agent 完成任務（end_turn）
2. 觸發 SessionEnd hook
3. 更新 sessions.json（lastActiveAt, turnCount 等）
4. 若 refCount 降至 0 → 刪除 transcript JSONL
5. Disk budget 檢查 → 清理超額 sessions
```

---

## 5. ACP 6 個檔案職責

| 檔案 | 職責 |
|------|------|
| `acp-session.ts` | Session 建立/resume/cleanup 核心邏輯 |
| `acp-context-engine.ts` | ContextEngine 介面實作（訊息組裝） |
| `acp-bindings.ts` | Persistent bindings CRUD |
| `acp-store.ts` | Session 持久化（sessions.json） |
| `acp-types.ts` | 所有型別定義 |
| `acp-schema.ts` | TypeBox schema（配置驗證） |

### ContextEngine 介面

```typescript
interface ContextEngine {
  appendUserMessage(content: string, metadata?: MessageMetadata): void;
  appendAssistantMessage(content: string, metadata?: MessageMetadata): void;
  appendToolResult(toolId: string, result: unknown): void;
  getMessages(): Message[];
  getTokenCount(): number;
  truncate(targetTokens: number): void;
}
```

---

## 6. Session Store（sessions.json）

### 結構

```typescript
// sessions.json 儲存 metadata（非 transcript 內容）
interface SessionsStore {
  version: number;
  sessions: Record<string, SessionEntry>;
}

interface SessionEntry {
  sessionKey: string;
  agentId: string;
  channelId?: string;
  accountId?: string;
  createdAt: number;      // Unix timestamp（ms）
  lastActiveAt: number;   // Unix timestamp（ms）
  turnCount: number;
  refCount: number;       // 引用計數（多個 binding 共用同一 session）
  transcriptPath: string; // JSONL 檔案路徑
  diskBytes?: number;     // 估算的磁碟使用量（bytes）
  metadata?: Record<string, unknown>;
}
```

### TTL 規則

```typescript
// Session 過期條件（任一滿足）：
// 1. lastActiveAt + sessionTtlMs < now
// 2. 手動呼叫 session.expire()
// 3. Disk budget 超額觸發清理
```

### Disk Budget 清理邏輯

```typescript
// 清理觸發條件：總 diskBytes > diskBudgetBytes
// 清理策略：LRU（最久未使用優先刪除）
// 關鍵：refCount 降到 0 才能刪除 transcript JSONL
// refCount > 0：有 ACP binding 引用此 session，不得刪除
```

### measureStoreEntryChunkBytes() 估算

```typescript
// 估算一個 Session entry 的磁碟佔用（bytes）
function measureStoreEntryChunkBytes(entry: SessionEntry): number {
  // 讀取 transcript JSONL 的實際大小
  // 若無法讀取 → 返回估算值（turnCount * AVG_TURN_BYTES）
  const AVG_TURN_BYTES = 2048; // 每輪平均 2KB
  return entry.diskBytes ?? entry.turnCount * AVG_TURN_BYTES;
}
```

---

## 7. Transcript JSONL 格式（7 種 type）

```typescript
// transcript JSONL 每行為一個 JSON 物件
type TranscriptEntry =
  | { type: "user_message";      content: string; timestamp: number; metadata?: ... }
  | { type: "assistant_message"; content: string; timestamp: number; thinking?: string; }
  | { type: "tool_use";          toolId: string;  input: unknown;    timestamp: number; }
  | { type: "tool_result";       toolId: string;  result: unknown;   timestamp: number; }
  | { type: "system_event";      event: string;   data?: unknown;    timestamp: number; }
  | { type: "context_truncate";  reason: string;  removedCount: number; timestamp: number; }
  | { type: "session_meta";      key: string;     value: unknown;    timestamp: number; };
```

---

## 8. sessions_spawn 參數完整說明

```typescript
// sessions_spawn tool 的完整參數
interface SessionsSpawnParams {
  agentId: string;              // 目標 Agent ID
  sessionKey?: string;          // 指定 session key（可選，不指定則自動生成）
  channel?: string;             // Channel 名稱
  accountId?: string;           // 帳號 ID
  conversationId?: string;      // 對話 ID（ACP session 用）
  mode?: "run" | "session";     // 執行模式
  runtime?: "subagent" | "acp"; // runtime 類型
  instructions?: string;        // 附加指令
  context?: string;             // 初始 context
  depth?: number;               // subagent 深度（系統自動計算，通常不手動指定）
  // 禁用參數（ACP 安全限制）：
  // - credentials（禁止傳遞憑證）
  // - hooks（禁止覆寫 hooks）
}
```

### 禁用參數說明

```typescript
// ACP sessions_spawn 安全限制：
// 以下參數被禁用，傳入時會被靜默忽略或拋出錯誤：
// - credentials: 防止子 Agent 繼承父 Agent 的憑證
// - hooks: 防止子 Agent 覆寫 hook 行為
```

---

## 9. mode 差異（run vs session）

| 項目 | `run` | `session` |
|------|-------|-----------|
| Session 持久化 | 否 | 是 |
| Transcript 保存 | 否 | 是 |
| Memory auto-recall | 依配置 | 依配置 |
| 適用場景 | 一次性任務 | 互動對話 |
| Session TTL | N/A | 有效 |

---

## 10. runtime 差異（subagent vs acp）

| 項目 | `subagent` | `acp` |
|------|-----------|-------|
| 生成方式 | 父 Agent 主動生成 | 外部透過 ACP 協定 |
| Session Key | 含 depth + nonce | SHA256 hash |
| Persistent bindings | 否 | 可（discord/telegram） |
| 深度計數 | 是（max=5） | 否 |
| 適用場景 | Agent 內部子任務 | 跨 Agent 通訊 |

---

## 11. AcpRuntimeSessionMode

```typescript
type AcpRuntimeSessionMode =
  | "create"    // 建立新 session
  | "resume"    // 恢復既有 session
  | "transient" // 暫時性 session（不持久化）
  | "inherit";  // 繼承父 session 上下文
```

---

## 12. AcpSessionUpdateTag

```typescript
// ACP session 更新標籤（記錄 session 狀態變化）
type AcpSessionUpdateTag =
  | "turn_complete"        // 一輪對話完成
  | "context_truncated"    // Context 被截斷
  | "tool_executed"        // 工具已執行
  | "memory_stored"        // 記憶已儲存
  | "memory_recalled"      // 記憶已召回
  | "agent_spawned"        // 子 Agent 已生成
  | "session_expired"      // Session 已過期
  | "binding_created"      // Binding 已建立
  | "binding_removed"      // Binding 已移除
  | "error";               // 發生錯誤
```
