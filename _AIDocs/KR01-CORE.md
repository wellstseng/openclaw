# 01-CORE — Agent 核心引擎

> 來源：openclaw-knowledge-base.md §2 §3 §5 §6 + F058 §3 §4 §8 §13 + F059

---

## 1. Agent 型別完整定義

### AgentKind enum

```typescript
enum AgentKind {
  Run      = "run",       // 單次執行，無 session 持久化
  Session  = "session",   // 對話模式，session 持久化
  Subagent = "subagent",  // 子 Agent（由父 Agent 生成）
  Nested   = "nested",    // 巢狀執行（ACP nested mode）
}
```

### 三種執行模式差異

| 模式 | Session 持久化 | 使用場景 |
|------|--------------|---------|
| `run` | 否 | 一次性任務，無需記憶對話 |
| `session` | 是 | 互動式對話，保留對話歷史 |
| `subagent` | 是（新 session） | 父 Agent 生成的子任務 |

### CommandLane enum

```typescript
enum CommandLane {
  Main     = "main",      // 主要執行通道
  Cron     = "cron",      // 排程任務通道
  Subagent = "subagent",  // 子 Agent 通道
  Nested   = "nested",    // 巢狀執行通道
}
```

---

## 2. Agent Run Loop 完整邏輯（run.ts）

### 迭代上限計算公式

```typescript
// 精確公式（F059 修正版）
const scaledProfileCount = Math.max(1, profileCandidateCount);
const maxIterations = Math.max(
  MIN_ITERATIONS,                          // 32
  Math.min(
    MAX_ITERATIONS,                        // 160
    ITER_BASE + scaledProfileCount * ITER_PER_PROFILE  // 24 + n*8
  )
);
// 當 profileCandidateCount = 0 時，scaledProfileCount = 1，maxIterations = 32
// 當 profileCandidateCount = 10 時，maxIterations = 104
// 當 profileCandidateCount = 17+ 時，maxIterations = 160（上限）
```

| 常數 | 值 |
|------|-----|
| `MIN_ITERATIONS` | 32 |
| `MAX_ITERATIONS` | 160 |
| `ITER_BASE` | 24 |
| `ITER_PER_PROFILE` | 8 |

### Run Loop 流程（逐段）

```
1. 初始化
   ├─ 載入 Session（若 session mode）
   ├─ 解析 profileCandidateCount
   ├─ 計算 maxIterations（公式見上）
   └─ 初始化 Context Window Guard

2. 主迭代（每輪）
   ├─ 構建 System Prompt（27 sections）
   ├─ 組裝 messages array（含 context window 截斷）
   ├─ 呼叫 LLM Provider（streaming）
   ├─ 處理 LLM 輸出：
   │   ├─ text content → 累積
   │   ├─ tool_use → 執行工具
   │   └─ thinking → 累積（若 thinking mode）
   ├─ 檢查終止條件（見下節）
   └─ 更新 iteration 計數

3. 終止後
   ├─ 儲存 Session（若 session mode）
   ├─ 寫入 Memory（若 auto-capture）
   └─ 回傳結果
```

### 終止條件

```typescript
// run.ts L798-1487（F058 驗證行號）
終止條件（任一滿足即停止）：
1. iteration >= maxIterations
2. LLM 輸出 stop_reason = "end_turn"（無 tool_use）
3. 收到 SILENT_REPLY_TOKEN（"__silent_reply__"）
4. 收到 HEARTBEAT_TOKEN（"__heartbeat__"）且 heartbeat mode
5. NO_REPLY regex 匹配（/^(\.|\.\.)?\s*$/）
6. Context Overflow → hard-min 策略觸發
7. Overload Failover 最終失敗
```

---

## 3. Context Window 管理

### 三層閾值

```typescript
// context window guard/warn/hard-min 邏輯
const GUARD_RATIO  = 0.85;  // 超過 → 觸發截斷策略
const WARN_RATIO   = 0.75;  // 超過 → 發出警告
const HARD_MIN     = 4096;  // tokens，最小保留空間
```

### evaluateContextWindowGuard() 回傳值

```typescript
type ContextWindowGuardResult = {
  status: "ok" | "warn" | "guard" | "overflow";
  usedTokens: number;
  maxTokens: number;
  ratio: number;
};
```

### Context Overflow 三種策略

```typescript
// run.ts L958-1147（F058 驗證行號）
策略一：Truncate（截斷舊訊息）
  → 從最舊訊息開始移除，保留 system prompt + 最新 N 輪

策略二：Summarize（摘要壓縮）
  → 呼叫 LLM 對舊訊息生成摘要，以摘要取代原始訊息

策略三：Hard-Min（強制終止）
  → 當剩餘空間 < HARD_MIN 時，強制終止迴圈
```

### resolveContextWindowInfo() 函式簽名

```typescript
function resolveContextWindowInfo(
  model: string,
  provider: KnownProvider,
  config?: AgentConfig
): {
  contextWindow: number;
  maxOutputTokens: number;
  reservedTokens: number;
}
```

---

## 4. System Prompt 27 個 Section 完整列表

> Full / Minimal / None 三種模式：Full = 全部 27 sections；Minimal = 僅核心 sections；None = 無 system prompt

| # | Section 名稱 | Full | Minimal | 說明 |
|---|-------------|------|---------|------|
| 1 | identity | ✓ | ✓ | Agent 身份與名稱 |
| 2 | current_time | ✓ | ✓ | 目前時間 |
| 3 | user_info | ✓ | ✓ | 使用者資訊 |
| 4 | capabilities | ✓ | ✓ | Agent 能力清單 |
| 5 | tools | ✓ | ✓ | 可用工具說明 |
| 6 | memory_context | ✓ | - | 記憶上下文（auto-recall） |
| 7 | session_context | ✓ | - | Session 資訊 |
| 8 | channel_context | ✓ | - | 頻道/平台資訊 |
| 9 | agent_config | ✓ | - | Agent 設定摘要 |
| 10 | profiles | ✓ | - | Profile 列表 |
| 11 | instructions | ✓ | ✓ | 主要指令（user-defined） |
| 12 | response_style | ✓ | - | 回應風格設定 |
| 13 | language | ✓ | - | 語言設定 |
| 14 | safety | ✓ | - | 安全規則 |
| 15 | tool_use_policy | ✓ | - | 工具使用政策 |
| 16 | memory_policy | ✓ | - | 記憶操作政策 |
| 17 | context_limits | ✓ | - | Context 限制說明 |
| 18 | heartbeat_policy | ✓ | - | Heartbeat 行為說明 |
| 19 | auto_reply_policy | ✓ | - | Auto-Reply 規則 |
| 20 | exec_policy | ✓ | - | 程式執行政策 |
| 21 | file_policy | ✓ | - | 檔案操作政策 |
| 22 | web_policy | ✓ | - | Web 存取政策 |
| 23 | subagent_policy | ✓ | - | 子 Agent 政策 |
| 24 | cron_policy | ✓ | - | Cron 排程政策 |
| 25 | pairing_info | ✓ | - | Pairing 資訊 |
| 26 | node_info | ✓ | - | Node 資訊 |
| 27 | footer | ✓ | ✓ | 結尾標記 |

---

## 5. Gateway HTTP Pipeline 13 個 Stage

> 原則：**first-match-wins**，每個 stage 可短路，不進入下一 stage

| # | Stage 名稱 | 說明 |
|---|----------|------|
| 1 | **CORS** | 處理跨域請求，注入 CORS headers |
| 2 | **Auth** | 驗證 Bearer token / API key |
| 3 | **Rate Limit** | Sliding window rate limit（by deviceId \| clientIp） |
| 4 | **Route Resolution** | 解析 URL → 找到對應 Agent 路由 |
| 5 | **Session Lookup** | 查找或建立 Session |
| 6 | **Profile Resolution** | 解析 Profile（決定 system prompt 變數） |
| 7 | **Agent Dispatch** | 分派到 AgentKind（run/session/subagent） |
| 8 | **Context Assembly** | 組裝 messages array |
| 9 | **LLM Call** | 呼叫 Provider streaming API |
| 10 | **Tool Execution** | 執行 tool_use 請求 |
| 11 | **Response Formatting** | 格式化輸出（markdown/plain/json） |
| 12 | **Delivery** | 傳遞到 Channel（Discord/Telegram 等） |
| 13 | **Cleanup** | 儲存 Session、觸發 hooks、清理資源 |

### Gateway 重要常數

| 常數 | 值 | 說明 |
|------|----|------|
| Voice transcript dedupe | 1.5s | 語音轉錄去重視窗 |
| Tool event TTL | 10min | tool event 存活時間 |
| Tool event grace period | 30s | grace period |
| Control plane rate limit | sliding window | by deviceId \| clientIp |
| Auth rate limit | 10 req / 60s | lock 5 min |
| Hook auth rate limit | 20 req / 60s | hook 認證限制 |

### Gateway 啟動/關閉順序

```
啟動：
1. Config 解析與驗證
2. Logger 初始化
3. Memory 後端初始化（LanceDB）
4. Plugin Discovery + Extension 載入
5. Provider 初始化
6. Channel Provider 初始化
7. HTTP Server 啟動
8. Gateway Pipeline 掛載
9. Background jobs 啟動（Cron / Heartbeat）

關閉：
1. 停止接收新請求
2. 等待進行中請求完成
3. 儲存所有 Sessions
4. 關閉 Channel connections
5. 關閉 Memory 後端
6. HTTP Server 關閉
```

---

## 6. Routing 解析邏輯（resolve-route.ts）

### 完整流程

```
1. 解析請求路徑（URL path）
   ├─ 提取 agentId（可能含 @version）
   ├─ 提取 channelId（若為 channel route）
   └─ 提取 sessionId（若為 session route）

2. 查找 Agent 定義
   ├─ 精確匹配 agentId
   ├─ wildcard 匹配（若有配置）
   └─ 找不到 → 404

3. 驗證 Channel 合法性
   ├─ 檢查 agent.channels 白名單
   └─ 不在白名單 → 403

4. 解析 Session Key
   ├─ 若 sessionId 存在 → 使用既有 session
   ├─ 若 channel route → 依 channel 規則生成
   └─ 新建 session → 生成新 session key

5. 回傳 RouteContext
   └─ { agentId, channelId, sessionKey, agentDef, ... }
```

---

## 7. Backoff Policy 型別

```typescript
type BackoffPolicy = {
  type: "exponential" | "linear" | "constant";
  baseMs: number;       // 初始退避時間（毫秒）
  maxMs: number;        // 最大退避時間（毫秒）
  multiplier?: number;  // 指數倍率（exponential 型別）
  jitter?: boolean;     // 是否加入隨機抖動
};
```

### FailoverReason 型別

```typescript
type FailoverReason =
  | "overload"          // Provider 過載（429）
  | "rate_limit"        // 超過 rate limit
  | "context_overflow"  // Context window 溢出
  | "timeout"           // 請求逾時
  | "error"             // 一般錯誤
  | "provider_error";   // Provider 特定錯誤
```

### ThinkLevel 型別

```typescript
type ThinkLevel =
  | "none"    // 不使用 thinking
  | "low"     // 低思考量（budget_tokens 較少）
  | "medium"  // 中等思考量
  | "high"    // 高思考量（budget_tokens 較多）
  | "auto";   // 自動根據問題複雜度決定
```

---

## 8. Auto-Reply 系統

### 核心 Token 定義

```typescript
const HEARTBEAT_TOKEN    = "__heartbeat__";
const SILENT_REPLY_TOKEN = "__silent_reply__";
const NO_REPLY_REGEX     = /^(\.|\.\.)?\s*$/;
```

### isSilentReplyText 邏輯

```typescript
function isSilentReplyText(text: string): boolean {
  return text.trim() === SILENT_REPLY_TOKEN
    || text.trim().startsWith(SILENT_REPLY_TOKEN + "\n");
}
```

### Heartbeat 邏輯

```typescript
// Heartbeat 觸發條件：
// 1. LLM 輸出第一個 token 之前超過 heartbeatIntervalMs
// 2. 工具執行超過 heartbeatIntervalMs
// Heartbeat 行為：
// - 發送 "__heartbeat__" 到 channel（防止超時）
// - 不算作有效回覆
// - 繼續等待真正的回覆
```

### Auto-Reply 函式鏈

```
inboundMessage
  → debounce
  → preflight（40+ gates）
  → job queue
  → worker queue（keyed async queue）
  → process（9 stages）
  → delivery
```

### TypingPolicy / SendPolicyOverride 型別

```typescript
type TypingPolicy = "always" | "on_tool" | "never";

type SendPolicyOverride = {
  forceNew?: boolean;    // 強制新訊息（不 edit）
  forceEdit?: boolean;   // 強制 edit 既有訊息
  skipDelivery?: boolean; // 跳過實際傳送（dry-run）
};
```

### Inbound 防抖優先順序

1. Bot self-filter（在 debounce 前，避免佔容量）
2. 有媒體附件 → 不 debounce
3. 有控制命令 → 不 debounce
4. 一般訊息 → debounce（key: `{platform}:{accountId}:{channelId}:{authorId}`）

---

## 9. retry 計算公式

```typescript
// 精確公式（來自 F058 §2）
const retryCount = Math.max(
  MIN_RETRY,        // 最小重試次數
  BASE_RETRY + profileCandidateCount * PER_PROFILE_RETRY
);
```

（注意：此公式與 maxIterations 公式相似但用途不同，maxIterations 用 `Math.max(1, n)` 確保 profileCount=0 時仍有合理值）

---

## 10. Subagent Depth 計算

```typescript
// Session key 含深度資訊
// 子 Agent depth = 父 Agent depth + 1
// MAX_SUBAGENT_DEPTH = 5
// 超過深度上限 → 拒絕建立子 Agent，回傳錯誤
```
