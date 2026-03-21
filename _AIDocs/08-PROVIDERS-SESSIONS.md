# Provider + Model + Session 深入

> Phase 1-2 產出 | 2026-03-12 | 掃描範圍：`src/agents/` model/provider 相關 + `src/sessions/` + `src/config/sessions/` + `src/providers/`

## 一句話定位

**Provider 層是「LLM 多供應商統一存取框架」**：透過 Provider Registry → Model Resolution → Auth Profile → Fallback Chain 四層抽象，讓上層 Agent 引擎用同一套介面存取 20+ LLM 供應商；Session 層是「對話狀態持久化引擎」，以 `sessions.json`（元資料）+ `.jsonl`（逐行 transcript）雙檔架構管理對話生命週期。

> C# 對照：Provider Registry ≈ `IServiceCollection` + Named `HttpClientFactory`（每個 provider 像一個 named HttpClient 配置）；Model Resolution ≈ `IOptions<T>` 的多層 override（alias → config → default）；Auth Profile ≈ `ICredentialStore` + Round-Robin `ILoadBalancer`；Session Store ≈ Entity Framework 的 `DbContext` + Unit of Work pattern（atomic read-modify-write）。

---

## 架構全景

```
User Request ("claude-sonnet-4-5" / "my-alias" / "openai/gpt-5.2")
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Model Resolution Layer                        │
│                                                                 │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────────────────┐ │
│  │ Alias      │→│ Provider ID  │→│ Model ID Normalization   │ │
│  │ Lookup     │  │ Normalization│  │ (anthropic shorthand,    │ │
│  │ (config    │  │ (z.ai→zai,  │  │  google suffix, etc.)    │ │
│  │  aliases)  │  │  bedrock→   │  │                          │ │
│  │            │  │  amazon-    │  │ + Auth Profile Suffix    │ │
│  │            │  │  bedrock)   │  │   (@profile-id split)    │ │
│  └────────────┘  └─────────────┘  └────────────┬─────────────┘ │
│                                                 │               │
│                            ModelRef { provider, model, profile }│
└─────────────────────────────────────────────────┼───────────────┘
                                                  │
    ┌─────────────────────────────────────────────▼───────────────┐
    │                  Provider Registry                          │
    │                                                             │
    │  ┌──────────────┐    ┌──────────────┐                      │
    │  │ Implicit     │ +  │ Explicit     │  → mergeProviders()  │
    │  │ (auto-disc.) │    │ (user config)│                      │
    │  │ 20+ built-in │    │ custom URLs  │                      │
    │  └──────────────┘    └──────────────┘                      │
    │           │                                                 │
    │           ▼                                                 │
    │  normalizeProviders() → writes models.json                  │
    │  (resolve env vars, secret refs, API key markers)           │
    └─────────────────────────────────────────────┬───────────────┘
                                                  │
    ┌─────────────────────────────────────────────▼───────────────┐
    │                Auth Profile Layer                            │
    │                                                             │
    │  ┌─────────┐  ┌──────────┐  ┌───────────┐  ┌────────────┐ │
    │  │ Env Var  │→│ Profile  │→│ OAuth     │→│ External   │ │
    │  │ Check    │  │ Store    │  │ Refresh   │  │ CLI Sync   │ │
    │  │ (first)  │  │ (JSON)  │  │ (if exp.) │  │ (Claude/   │ │
    │  │          │  │          │  │           │  │  Codex CLI)│ │
    │  └─────────┘  └──────────┘  └───────────┘  └────────────┘ │
    │                      │                                      │
    │        Round-Robin + Cooldown + Priority Order              │
    └─────────────────────────────────────────────┬───────────────┘
                                                  │
    ┌─────────────────────────────────────────────▼───────────────┐
    │              Model Fallback Chain                            │
    │                                                             │
    │  Primary Model → Configured Fallbacks → Same-Provider Alt   │
    │       │              │                       │              │
    │       ▼              ▼                       ▼              │
    │  [attempt] ──fail→ [attempt] ──fail→ [attempt] ──fail→ ERR │
    │       │              │                       │              │
    │  Cooldown Mgmt   Smart Probe (30s)   Transient Retry (2.5s)│
    └─────────────────────────────────────────────────────────────┘
                                │
                    Streaming Protocol (per API type)
                    ────────────────────────────────
                    anthropic-messages │ openai-completions
                    google-generative  │ ollama │ bedrock
                    github-copilot     │ openai-responses
```

---

## 一、Provider Registry — 供應商註冊與解析

### 1.1 註冊機制

**沒有顯式 Registry 類別** — Provider 透過 config + auto-discovery 動態解析。

**主入口**：`ensureOpenClawModelsJson()` → `src/agents/models-config.ts`

```
ensureOpenClawModelsJson(config?, agentDirOverride?)
  │
  ├─ resolveProvidersForModelsJson({ cfg, agentDir })
  │   ├─ resolveImplicitProviders()    ← 20+ built-in
  │   ├─ resolveImplicitBedrockProvider()
  │   └─ resolveImplicitCopilotProvider()
  │
  ├─ mergeProviders(implicit + explicit)
  │
  ├─ normalizeProviders()   ← env var / secret ref 解析
  │
  └─ Write → agentDir/models.json
```

**Lazy Discovery**：Provider 僅在找到 apiKey/profile 時才建立（減少啟動時間）。

### 1.2 Provider 定義結構

**型別**：`ModelProviderConfig`（`src/config/types.models.ts`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `baseUrl` | `string` | API endpoint |
| `apiKey` | `SecretInput?` | API key（env var name / secret ref / plaintext） |
| `auth` | `"api-key" \| "aws-sdk" \| "oauth" \| "token"` | 認證方式 |
| `api` | `ModelApi` | 串流協定（見 1.4） |
| `authHeader` | `boolean?` | 是否用 Authorization header |
| `headers` | `Record<string, SecretInput>?` | 自訂 headers |
| `models` | `ModelDefinitionConfig[]` | 可用模型清單 |

**Model 定義**：`ModelDefinitionConfig`

| 欄位 | 說明 |
|------|------|
| `id` / `name` | 模型識別碼 / 顯示名稱 |
| `reasoning` | 是否支援 reasoning/thinking |
| `input` | 支援的輸入類型：`"text"` / `"image"` / `"document"` |
| `cost` | `{ input, output, cacheRead, cacheWrite }` — 每百萬 token 價格 |
| `contextWindow` | 上下文長度 |
| `maxTokens` | 最大輸出 token |
| `compat` | 協定相容性 flags |

### 1.3 完整 Provider 目錄

#### Implicit Providers（自動發現，20+）

| Provider | Auth | API 協定 | 發現方式 |
|----------|------|---------|---------|
| **anthropic** | API Key/OAuth | anthropic-messages | Env / Profile |
| **openai** | API Key | openai-completions | Env / Profile |
| **google** | API Key/OAuth | google-generative-ai | Env / Profile |
| **openai-codex** | OAuth (PKCE) | openai-codex-responses | Codex CLI |
| **amazon-bedrock** | AWS SDK | bedrock-converse-stream | Config + AWS creds |
| **github-copilot** | Token exchange | github-copilot | Env / Profile |
| **ollama** | Local (無需 key) | ollama | `localhost:11434` auto-discover |
| **openrouter** | API Key | openai-completions | Env / Profile |
| **together** | API Key | openai-completions | Env / Profile |
| **huggingface** | API Key | openai-completions | Env / Profile + HF API |
| **nvidia** | API Key | openai-completions | Env / Profile |
| **vllm** | API Key (opt) | openai-completions | Env / Profile |
| **minimax** | API Key / OAuth | anthropic-messages | Env / Profile |
| **moonshot** | API Key | openai-completions | Env (Kimi K2.5) |
| **kimi-coding** | API Key | anthropic-messages | Env / Profile |
| **qwen-portal** | OAuth | openai-completions | OAuth Profile |
| **volcengine** | API Key | openai-completions | Env (Doubao) |
| **byteplus** | API Key | openai-completions | Env |
| **xiaomi** | API Key | anthropic-messages | Env / Profile |
| **qianfan** | API Key | openai-completions | Env (Baidu) |
| **venice** | API Key | openai-completions | Env + fetch |
| **kilocode** | API Key | openai-completions | Env + gateway |
| **cloudflare-ai-gateway** | API Key | anthropic-messages | OAuth Profile |
| **synthetic** | API Key | anthropic-messages | Env / Profile |

#### Explicit Providers（使用者自訂）

使用者可在 `openclaw.json` 的 `models.providers` 中定義任意 provider，指定 `baseUrl`、`models`、`auth`。

### 1.4 支援的 API 協定（8 種）

```typescript
const MODEL_APIS = [
  "openai-completions",         // OpenAI-compatible SSE（最多 provider 用）
  "openai-responses",           // OpenAI Responses 格式
  "openai-codex-responses",     // OpenAI Codex 專用
  "anthropic-messages",         // Anthropic Messages API
  "google-generative-ai",       // Google Generative AI
  "github-copilot",             // GitHub Copilot Token Exchange
  "bedrock-converse-stream",    // AWS Bedrock Converse
  "ollama",                     // Ollama 原生 API
] as const;
```

### 1.5 models.json 輸出格式

**位置**：`~/.openclaw/agents/<agentId>/models.json`

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://api.anthropic.com/v1",
      "apiKey": "ANTHROPIC_API_KEY",      // ← env var NAME, not value
      "api": "anthropic-messages",
      "authHeader": true,
      "models": [
        {
          "id": "claude-opus-4-6",
          "name": "Claude Opus 4.6",
          "reasoning": true,
          "input": ["text", "image"],
          "cost": { "input": 15, "output": 75 },
          "contextWindow": 200000,
          "maxTokens": 4096
        }
      ]
    }
  }
}
```

**寫入模式**：
- `"merge"`（預設）：保留現有 secrets/apiKey/baseUrl，只更新 model metadata
- `"replace"`：完全從 config + implicit discovery 重建

---

## 二、Model Resolution — 字串到 Provider+Model 的解析

### 2.1 主入口：`resolveModelRefFromString()`

**檔案**：`src/agents/model-selection.ts`

```
Input: "claude-sonnet-4-5@main-agent"
  │
  ├─ ① splitTrailingAuthProfile()
  │     → model = "claude-sonnet-4-5", profile = "main-agent"
  │
  ├─ ② Check Alias Index（no "/" → 查 alias）
  │     → "my-fast" → "anthropic/claude-sonnet-4-5"
  │
  ├─ ③ parseModelRef()（split on "/"）
  │     → { provider: "anthropic", model: "claude-sonnet-4-5" }
  │     → 無 "/" 時用 defaultProvider
  │
  ├─ ④ normalizeProviderModelId()
  │     Anthropic: "opus-4.6" → "claude-opus-4-6"
  │     Google:    "gemini-3-pro" → "gemini-3-pro-preview"
  │     OpenRouter: "aurora-alpha" → "openrouter/aurora-alpha"
  │
  └─ Output: ModelRef { provider, model, profile? }
```

### 2.2 Provider ID 正規化

`normalizeProviderId()` — 容錯對照表：

| 輸入 | 正規化結果 |
|------|-----------|
| `z.ai` / `z-ai` | `zai` |
| `opencode-zen` | `opencode` |
| `qwen` | `qwen-portal` |
| `kimi-code` | `kimi-coding` |
| `bedrock` / `aws-bedrock` | `amazon-bedrock` |
| `bytedance` / `doubao` | `volcengine` |
| 大小寫不敏感 | 全部 lowercase |

### 2.3 Alias 系統

**建構**：`buildModelAliasIndex(cfg, defaultProvider)` → `Map<aliasLowercase, { alias, ref }>`

**來源**：`config.agents.defaults.models[key].alias`

```yaml
# openclaw.json
agents:
  defaults:
    models:
      anthropic/claude-sonnet-4-5:
        alias: "my-fast"
```

使用者輸入 `my-fast` → resolve 成 `anthropic/claude-sonnet-4-5`。

---

## 三、Auth Profile — 認證管理

### 3.1 架構

**目錄**：`src/agents/auth-profiles/`（10+ 檔案）

| 檔案 | 職責 |
|------|------|
| `types.ts` | 三種憑證型別定義 |
| `store.ts` | `auth-profiles.json` 讀寫 + legacy 遷移 |
| `order.ts` | Profile 排序、Round-Robin、Cooldown 管理 |
| `oauth.ts` | OAuth token refresh + API key 解析 |
| `session-override.ts` | 每 session 的 auth profile 選擇 |
| `usage.ts` | 失敗追蹤、Cooldown 計算 |
| `credential-state.ts` | Token 過期驗證、eligibility reason codes |
| `profiles.ts` | CRUD 操作 |
| `repair.ts` | Legacy profile ID 修復 |
| `external-cli-sync.ts` | 同步 Claude CLI / Codex CLI / Qwen CLI 憑證 |

### 3.2 三種憑證型別

```typescript
// ① API Key — 最簡單
{ type: "api_key", provider: "openai", key?: string, keyRef?: SecretRef }

// ② Token — Bearer token（可有過期時間）
{ type: "token", provider: "anthropic", token?: string, tokenRef?: SecretRef, expires?: number }

// ③ OAuth — Access + Refresh + 必定有過期
{ type: "oauth", provider: "openai-codex",
  access: string, refresh: string, expires: number,
  clientId?, email?, projectId?, accountId? }
```

### 3.3 Auth Profile Store

**位置**：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`

```json
{
  "version": 1,
  "profiles": {
    "anthropic:claude-cli": { "type": "token", "provider": "anthropic", ... },
    "openai:work":          { "type": "api_key", "provider": "openai", ... }
  },
  "order": {
    "openai": ["openai:work", "openai:default"]
  },
  "lastGood": {
    "openai": "openai:work"
  },
  "usageStats": {
    "openai:work": {
      "lastUsed": 1710000000000,
      "cooldownUntil": null,
      "errorCount": 0,
      "failureCounts": { "rate_limit": 0 }
    }
  }
}
```

**Profile ID 格式**：`<provider>:<suffix>`（如 `anthropic:claude-cli`、`openai:work`）

### 3.4 API Key 解析優先級

```
① Environment Variables → PROVIDER_ENV_API_KEY_CANDIDATES mapping
   e.g. ANTHROPIC_API_KEY, OPENAI_API_KEY, GEMINI_API_KEY...
②  Auth Profile Store → auth-profiles.json
③ Session Override → /model gpt-4@openai:work
④ External CLI Sync → Claude CLI / Codex CLI / Qwen CLI
```

### 3.5 Profile 選擇演算法（Round-Robin + Cooldown）

`resolveAuthProfileOrder()`:

1. 清除過期 cooldown
2. 解析排序偏好：`store.order[provider]` > `config.auth.order` > 全部 profiles
3. 過濾 eligibility（憑證完整性、token 過期、mode mismatch）
4. Cooldown 排序：可用的優先，cooldown 中的按過期時間排
5. 型別偏好（無顯式 order 時）：OAuth > Token > API Key
6. 同型別內：`lastUsed` 最舊的優先（round-robin）

**Ineligibility Reason Codes**：

| Code | 說明 |
|------|------|
| `ok` | 可用 |
| `missing_credential` | 無 key/token |
| `expired` | Token 已過期 |
| `invalid_expires` | expires 欄位格式錯誤 |
| `profile_missing` | Profile ID 不存在 |
| `provider_mismatch` | Provider 不匹配 |
| `mode_mismatch` | 認證型別不匹配 |
| `unresolved_ref` | SecretRef 無法解析 |

### 3.6 Session Override

```
/model gpt-4@openai:work    ← 指定此 session 用 "work" profile
```

- 驗證 profile 存在且 provider 匹配
- 標記 `authProfileIdSource: "user"`
- Cooldown 時自動切換下一個可用 profile
- 跨 compaction 存活

### 3.7 外部 CLI 同步

| CLI 工具 | Profile ID | 來源 |
|----------|-----------|------|
| Claude Code | `anthropic:claude-cli` | macOS Keychain / `~/.claude/.credentials.json` |
| Codex CLI | `openai-codex:codex-cli` | macOS Keychain / `~/.codex/auth.json` |
| Qwen Code CLI | `qwen-portal:qwen-cli` | `~/.qwen/oauth_creds.json` |
| MiniMax CLI | `minimax-portal:minimax-cli` | `~/.minimax/oauth_creds.json` |

同步策略：每次 auth store 載入時、只在外部較新時更新、15 分鐘 cache TTL。

---

## 四、Model Fallback Chain — 失敗遞補

### 4.1 核心流程

**檔案**：`src/agents/model-fallback.ts`（651 行）

```
runWithModelFallback({ cfg, provider, model, run })
  │
  ├─ resolveFallbackCandidates()
  │   ├─ Primary: 使用者指定的 (provider, model)
  │   ├─ Configured: config.agents.defaults.model.fallbacks[]
  │   └─ Same-Provider: 同 provider 的替代模型
  │
  ├─ for each candidate:
  │   ├─ resolveCooldownDecision() ← 檢查 profile cooldown
  │   ├─ run(provider, model)      ← 呼叫 LLM
  │   └─ on error → collect FallbackAttempt, continue
  │
  └─ 全部失敗 → throw summary error with all attempts
```

### 4.2 Cooldown 管理

**失敗分類（FailoverReason）**：

| Reason | HTTP | 處理 |
|--------|------|------|
| `rate_limit` | 429 | 暫時 cooldown，5 小時 |
| `overloaded` | 503 (+overload marker) | 暫時 cooldown |
| `timeout` | 408/502/504/521-530 | 暫時 |
| `billing` | 402 | 長 cooldown（5h，max 24h） |
| `auth` | 401/403 | 暫時 auth 問題 |
| `auth_permanent` | 401/403 (+revoked) | 永久，需手動修復 |
| `model_not_found` | 404 | 跳過此模型 |
| `format` | 400 | 跳過 |
| `session_expired` | 410 | 跳過 |

**Smart Probe**：Primary 在 cooldown 中但存在 fallback 時，30 秒後允許重新探測（`MIN_PROBE_INTERVAL_MS = 30_000`），2 分鐘 margin。

### 4.3 Backoff 策略

```typescript
// src/infra/backoff.ts
const OVERLOAD_FAILOVER_BACKOFF = {
  initialMs: 250,
  maxMs: 1_500,
  factor: 2,
  jitter: 0.2
};
// 公式: min(1500, 250 × 2^(attempt-1) + random(0~20% of base))
```

### 4.4 Context Overflow 特例

**Context overflow 不走 fallback**（不同模型可能有更小的 context window）→ 直接觸發 auto-compaction 或 session reset。

### 4.5 Transient HTTP 重試

```typescript
const TRANSIENT_HTTP_ERROR_CODES = [500, 502, 503, 504, 521, 522, 523, 524, 529];
const TRANSIENT_HTTP_RETRY_DELAY_MS = 2_500;  // 重試一次，延遲 2.5 秒
```

重試的是**整條 fallback chain**（不是只有下一個模型），因為 transient error 通常影響整個 provider。

---

## 五、Streaming — 串流實作

### 5.1 多層串流架構

```
Layer 1: PI-AI SDK (streamSimple)
  │  per-protocol: anthropic SSE / openai SSE / ollama / bedrock
  │
  ▼
Layer 2: subscribeEmbeddedPiSession (pi-embedded-subscribe.ts, 800+ 行)
  │  事件類型: assistant message / tool execution / reasoning blocks / compaction
  │
  ▼
Layer 3: Block Chunker (pi-embedded-block-chunker.ts)
  │  deltaBuffer → blockBuffer → semantic chunking
  │  (paragraph / newline / sentence breaks)
  │
  ▼
Layer 4: Streaming Directives (streaming-directives.ts)
  │  解析 [[reply:id]] 等內嵌指令
  │  跨 chunk 邊界保持 context
  │
  ▼
UI / Channel Delivery
```

### 5.2 串流狀態機

```
Raw tokens (deltaBuffer)
  → Grouped by block level (blockBuffer)
  → onPartialReply callback (typed to UI)
  → onBlockReply callback (structured delivery)
```

**Block Reply Break 模式**：
- `"text_end"`：文字區塊結束時 flush
- `"message_end"`：整則訊息結束才 flush

### 5.3 各 Provider 串流差異

| API 協定 | 串流格式 | 特點 |
|---------|---------|------|
| `anthropic-messages` | SSE（Anthropic 格式） | content_block_delta events |
| `openai-completions` | SSE（OpenAI 格式） | choices[0].delta.content |
| `google-generative-ai` | SSE | candidates[0].content.parts |
| `ollama` | NDJSON stream | `{ message: { content } }` |
| `bedrock-converse-stream` | AWS event stream | Binary frame protocol |
| `github-copilot` | Token exchange → SSE | 先 token exchange 再 SSE |

---

## 六、Session — 對話持久化

### 6.1 雙檔架構

```
~/.openclaw/agents/{agentId}/sessions/
  ├─ sessions.json                    ← 元資料 store（所有 session 的 metadata）
  ├─ {sessionId}.jsonl                ← 主 transcript（逐行 JSON）
  └─ {sessionId}-topic-{topicId}.jsonl ← Topic/Thread transcript
```

### 6.2 Session Key vs Session ID

| 概念 | 格式 | 用途 |
|------|------|------|
| **Session Key** | `agent:<agentId>:<channel>:<chatType>:<identifier>` | 應用層路由 key，store 的 key |
| **Session ID** | UUID (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`) | 持久化識別碼，對應 .jsonl 檔名 |

**Key 內含的 chatType 推導**：
- 含 `:group:` → `"group"`
- 含 `:channel:` → `"channel"`
- 含 `:direct:` / `:dm:` → `"direct"`
- Legacy Discord: `discord:<accountId>:guild-<guildId>:channel-<channelId>` → `"channel"`

**特殊 key 判定**：

| 函式 | Pattern |
|------|---------|
| `isCronRunSessionKey()` | `cron:<name>:run:<id>` |
| `isSubagentSessionKey()` | `subagent:` 或含 `:subagent:` |
| `getSubagentDepth()` | 計算 `:subagent:` 出現次數（0=main） |
| `isAcpSessionKey()` | ACP session |
| `resolveThreadParentSessionKey()` | 從 `:thread:` / `:topic:` 取 parent |

### 6.3 SessionEntry 完整欄位

**`sessions.json` 中每個 entry 的結構**（`src/config/sessions/types.ts`）：

```typescript
SessionEntry {
  // 識別
  sessionId: string;              // UUID
  updatedAt: number;              // timestamp (ms)
  sessionFile?: string;           // → .jsonl 路徑

  // Model Override
  providerOverride?: string;      // AI provider
  modelOverride?: string;         // Model ID
  authProfileOverride?: string;   // Auth profile ID

  // Session 設定
  sendPolicy?: "allow" | "deny";  // 發送授權
  verboseLevel?: string;          // Thinking 輸出控制
  thinkingLevel?: string;         // Inline reasoning
  reasoningLevel?: string;        // Extended reasoning
  label?: string;                 // 顯示名稱（≤64 字元）

  // 頻道 & 來源
  channel?: string;               // discord / slack / telegram...
  chatType?: "direct" | "group" | "channel";
  origin?: SessionOrigin;         // provider, from, to, accountId, threadId

  // Token 統計
  inputTokens?: number;
  outputTokens?: number;
  totalTokens?: number;
  cacheRead?: number;
  cacheWrite?: number;

  // Runtime 狀態
  modelProvider?: string;         // 當前使用的 provider
  model?: string;                 // 當前使用的 model
  compactionCount?: number;       // Context compaction 次數
  contextTokens?: number;         // 可用 context 大小

  // 進階
  queueMode?: "steer" | "followup" | "collect" | ...;
  spawnedBy?: string;             // Parent session key
  spawnDepth?: number;            // Subagent 巢狀深度
  acp?: SessionAcpMeta;           // Agent Control Plane
}
```

### 6.4 Transcript JSONL 格式

**第一行（header）**：
```json
{
  "type": "session",
  "version": "<CURRENT_SESSION_VERSION>",
  "id": "<sessionId>",
  "timestamp": "2026-03-12T...",
  "cwd": "<process.cwd()>"
}
```

**後續行**：每則訊息一行 JSON（透過 `SessionManager.appendMessage()`）。

### 6.5 Session Store 操作

```
loadSessionStore(path)        ← 讀取 + Windows 重試（3次, 50ms backoff）
updateSessionStore(path, fn)  ← atomic read-modify-write（file lock queue）
saveSessionStore(path, store) ← temp file + rename（Windows 5 次重試）
```

**Cache**：in-memory TTL 45 秒（`OPENCLAW_SESSION_CACHE_TTL_MS` 可調）。

**Maintenance**（寫入後自動）：
- Pruning：超過 `pruneAfterMs` 的 entry 刪除
- Capping：超過 `maxEntries` 時刪最舊
- Rotation：`sessions.json` 超過 `rotateBytes` → rename 為 `sessions-{timestamp}.json`
- Disk budget：transcript 目錄總大小限制

### 6.6 `src/sessions/` 模組群

| 檔案 | 職責 |
|------|------|
| `input-provenance.ts` | 追蹤 user input 來源（external_user / inter_session / internal_system） |
| `level-overrides.ts` | 每 session verbose level override |
| `model-overrides.ts` | 每 session model/provider/authProfile override + stale 清理 |
| `send-policy.ts` | Allow/Deny 規則（channel + chatType + keyPrefix 匹配） |
| `session-id.ts` | UUID 格式驗證 |
| `session-key-utils.ts` | Session key 解析（chatType、subagent depth、cron、thread parent） |
| `session-label.ts` | Label 驗證（≤64 char, trimmed） |
| `transcript-events.ts` | JSONL 更新事件通知（listener pattern, swallow errors） |

### 6.7 Session 生命週期

```
① 訊息到達 → 從 channel/group/direct 推導 session key
② 查 sessions.json → 找到現有 entry 或建立新的（UUID 生成）
③ mergeSessionEntry(existing, patch) → updatedAt = max(existing, patch, now)
④ 首則訊息 → resolveAndPersistSessionFile() → ensureSessionHeader()
⑤ 後續訊息 → SessionManager.appendMessage() → emit transcript update
⑥ 維護：prune / cap / rotate / archive
```

---

## 七、Model Catalog — 模型目錄

**檔案**：`src/agents/model-catalog.ts`

**來源**（按順序）：
1. **PI SDK**：`pi-coding-agent.ModelRegistry`（從 `models.json` 載入）
2. **Configured Opt-In**：非 PI provider 的 `config.models.providers[p].models`
3. **Synthetic Fallbacks**：如 GPT-5.4 從 GPT-5.2 template 合成

```typescript
type ModelCatalogEntry = {
  id: string;
  name: string;
  provider: string;
  contextWindow?: number;
  reasoning?: boolean;
  input?: ("text" | "image" | "document")[];
};
```

**Context Window Guard**：

| 常量 | 值 | 用途 |
|------|---|------|
| `CONTEXT_WINDOW_HARD_MIN_TOKENS` | 16,000 | 低於此值拒絕使用 |
| `CONTEXT_WINDOW_WARN_BELOW_TOKENS` | 32,000 | 低於此值 log warning |

---

## 八、關鍵檔案速查

| 檔案 | 行數 | 職責 |
|------|------|------|
| `src/agents/models-config.ts` | ~300 | Config loader → models.json |
| `src/agents/models-config.providers.ts` | ~1390 | 20+ implicit provider builders |
| `src/agents/model-selection.ts` | — | 字串 → ModelRef 解析 + alias |
| `src/agents/model-fallback.ts` | 651 | Fallback chain + cooldown |
| `src/agents/model-catalog.ts` | — | ModelRegistry + catalog |
| `src/agents/model-ref-profile.ts` | — | `@profile` suffix 解析 |
| `src/agents/failover-error.ts` | 241 | Error 分類 + FailoverError |
| `src/agents/pi-embedded-subscribe.ts` | 800+ | Streaming subscription |
| `src/agents/pi-embedded-block-chunker.ts` | — | Block chunking |
| `src/config/types.models.ts` | — | Provider/Model 型別定義 |
| `src/config/sessions/store.ts` | — | Session store 核心 |
| `src/config/sessions/types.ts` | — | SessionEntry 定義 |
| `src/config/sessions/paths.ts` | — | Session 路徑解析 |
| `src/sessions/*.ts` | 14 files | Session 屬性模組群 |
| `src/agents/auth-profiles/*.ts` | 10+ files | Auth profile 完整系統 |
| `src/providers/*.ts` | 13 files | Provider-specific helpers |

---

## 九、邊界條件與陷阱

1. **apiKey 存的是 env var NAME 不是值** — `models.json` 中 `"apiKey": "ANTHROPIC_API_KEY"` 是變數名，runtime 才解析值
2. **OAuth marker** — OAuth provider 用 placeholder marker（如 `MINIMAX_OAUTH_MARKER`），runtime 才 refresh token 取真正的 access token
3. **models.json merge mode** — 預設 merge 不覆寫 secrets，debug 時容易誤以為 config 沒生效
4. **Provider ID 正規化靜默** — `z.ai` 靜默轉成 `zai`，使用者可能不知道
5. **Session key ≠ Session ID** — key 是路由用的（含 channel/chatType），ID 是 UUID（對應 .jsonl）
6. **Windows file lock retry** — Session store 讀寫在 Windows 有重試邏輯（3~5 次），多 agent 並行時可能遇到 lock contention
7. **Cooldown probe 不等於 recovery** — Smart probe 只是嘗試，不保證成功，失敗會延長 cooldown
8. **Context overflow 不走 fallback** — 不同模型 context window 不同，overflow 直接 compaction 或 reset
9. **Transient retry 重走整條 chain** — 不是只 retry 下一個模型，是重試全部（含 primary）
10. **Auth profile round-robin 按 lastUsed 排序** — 沒有顯式 order 時，最舊的先用，可能導致弱 profile 被優先選中

---

## 十、關鍵常量速查

| 常量 | 值 | 位置 |
|------|---|------|
| `SESSION_CACHE_TTL_MS` | 45,000 (45s) | sessions/store.ts |
| `MIN_PROBE_INTERVAL_MS` | 30,000 (30s) | model-fallback.ts |
| `TRANSIENT_HTTP_RETRY_DELAY_MS` | 2,500 (2.5s) | agent-runner-execution.ts |
| `CONTEXT_WINDOW_HARD_MIN_TOKENS` | 16,000 | context-window-guard.ts |
| `CONTEXT_WINDOW_WARN_BELOW_TOKENS` | 32,000 | context-window-guard.ts |
| `SESSION_LABEL_MAX_LENGTH` | 64 | session-label.ts |
| `BACKOFF_INITIAL_MS` | 250 | backoff.ts |
| `BACKOFF_MAX_MS` | 1,500 | backoff.ts |
| `BACKOFF_FACTOR` | 2 | backoff.ts |
| `COOLDOWN_DEFAULT_HOURS` | 5 | auth-profiles/usage.ts |
| `COOLDOWN_MAX_HOURS` | 24 | auth-profiles/usage.ts |
| `EXTERNAL_CLI_SYNC_TTL_MIN` | 15 | external-cli-sync.ts |
| `WINDOWS_WRITE_RETRIES` | 5 | sessions/store.ts |
| `WINDOWS_READ_RETRIES` | 3 | sessions/store.ts |
