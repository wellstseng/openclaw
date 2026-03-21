# 06-PROVIDERS — LLM Providers 完整細節

> 來源：openclaw-knowledge-base.md §10 + F058 §10 + F059 §2-6 §9 §12

---

## 1. 所有 Provider 完整清單（17 個）

| Provider ID | 類型 | 來源 |
|-------------|------|------|
| `anthropic` | 雲端 | 官方 |
| `openai` | 雲端 | 官方 |
| `google` | 雲端 | 官方 |
| `aws-bedrock` | 雲端 | AWS |
| `azure-openai` | 雲端 | Azure |
| `ollama` | 本地 | 開源 |
| `copilot-proxy` | 代理 | GitHub Copilot |
| `sglang` | 本地 | 開源 |
| `vllm` | 本地 | 開源 |
| `openrouter` | 代理 | OpenRouter |
| `nvidia` | 雲端 | NVIDIA |
| `minimax` | 雲端 | MiniMax（CN/Global） |
| `qwen-portal-auth` | 雲端 | Alibaba Qwen |
| `google-gemini-cli-auth` | 雲端 | Google（CLI 認證） |
| `groq` | 雲端 | Groq |
| `mistral` | 雲端 | Mistral AI |
| `together` | 雲端 | Together AI |

### KnownProvider 型別

```typescript
type KnownProvider =
  | "anthropic" | "openai" | "google" | "aws-bedrock" | "azure-openai"
  | "ollama" | "copilot-proxy" | "sglang" | "vllm"
  | "openrouter" | "nvidia" | "minimax" | "qwen-portal-auth"
  | "google-gemini-cli-auth" | "groq" | "mistral" | "together";
```

---

## 2. 預設值

```typescript
const DEFAULT_PROVIDER = "anthropic";
const DEFAULT_MODEL    = "claude-opus-4-6";
```

---

## 3. Anthropic

### 支援模型

```typescript
// Anthropic 可用模型（截至 2026-03）：
const ANTHROPIC_MODELS = [
  "claude-opus-4-6",      // 最強能力（預設）
  "claude-sonnet-4-6",    // 平衡性能
  "claude-haiku-4-5",     // 快速輕量
  "claude-opus-4-5",      // 前代 Opus
  "claude-3-7-sonnet",    // Claude 3.7
  "claude-3-5-haiku",     // Claude 3.5 Haiku
];
```

### Anthropic 特有選項

```typescript
interface AnthropicOptions {
  effort?: "low" | "medium" | "high";  // 推理努力程度
  interleavedThinking?: boolean;        // 交錯思考模式
  thinkingBudget?: number;             // thinking budget_tokens
  betas?: string[];                    // Beta API 功能
}
```

### SSE Streaming 實作

```typescript
// Anthropic 使用 SSE（Server-Sent Events）streaming
// Events：
// "message_start"       → 開始
// "content_block_start" → 內容塊開始（text/thinking/tool_use）
// "content_block_delta" → 增量更新
// "content_block_stop"  → 內容塊結束
// "message_delta"       → 訊息元資料更新
// "message_stop"        → 結束

// Thinking mode（extended thinking）：
// content_block type = "thinking"
// 包含 thinking text（可選顯示給用戶）
```

---

## 4. OpenAI

### 支援模型

```typescript
const OPENAI_MODELS = [
  "gpt-5",            // GPT-5
  "gpt-5.2",          // GPT-5.2
  "gpt-4o",           // GPT-4o
  "gpt-4o-mini",      // GPT-4o mini
  "gpt-4-turbo",      // GPT-4 Turbo
  "o1", "o1-mini",    // O1 series
  "o3", "o3-mini",    // O3 series
  "o4-mini",          // O4 mini
];
```

### OpenAI Streaming 差異

```typescript
// OpenAI 使用 SSE，但 event 格式不同：
// "data: {...}" → choices[0].delta
// choices[0].delta.content → 文字增量
// choices[0].delta.tool_calls → 工具呼叫增量
// choices[0].finish_reason → 完成原因
```

---

## 5. Google（Gemini）

### 支援模型

```typescript
const GOOGLE_MODELS = [
  "gemini-3-pro",           // Gemini 3 Pro
  "gemini-3-flash-preview", // Gemini 3 Flash
  "gemini-2.5-pro",         // Gemini 2.5 Pro
  "gemini-2.0-flash",       // Gemini 2.0 Flash
  "gemini-1.5-pro",         // Gemini 1.5 Pro
  "gemini-1.5-flash",       // Gemini 1.5 Flash
];
```

### Google Gemini Streaming 差異

```typescript
// Google 使用 Server-Sent Events，但格式為 JSON lines
// 每行：{ "candidates": [{ "content": {...} }] }
// 累積處理（非純增量）
```

---

## 6. AWS Bedrock

### 特殊邏輯

```typescript
// AWS Bedrock 特性：
// - 使用 AWS SDK（不用 fetch）
// - 需要 AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY + AWS_REGION
// - 支援 Anthropic Claude + Meta Llama + Mistral on Bedrock
// - 串流使用 InvokeModelWithResponseStreamCommand
// - 請求格式轉換：OpenAI-like → Bedrock 特有格式
```

---

## 7. Ollama（本地）

### 特性

```typescript
// Ollama 本地 provider：
// - DEFAULT_BASE_URL = "http://localhost:11434"（可覆寫）
// - 使用 OLLAMA_BASE_URL 環境變數
// - 支援 OpenAI 相容 API（/api/chat）
// - 自動 model discovery（/api/tags）
// - streaming 使用 NDJSON（不是 SSE）
```

---

## 8. copilot-proxy Extension

### 常數

```typescript
const COPILOT_PROXY = {
  PROVIDER_ID: "copilot-proxy",
  DEFAULT_BASE_URL: "https://api.githubcopilot.com",  // GitHub Copilot API
  API_KEY: "n/a",             // 不使用 API key（用 OAuth token）
  authHeader: false,          // 不傳 Authorization header
  DEFAULT_GATEWAY_PORT: 18789, // 本地 gateway port（device-pair）
};
```

### 13 個預設模型（完整清單）

```typescript
const COPILOT_DEFAULT_MODEL_IDS = [
  "gpt-5.2",                // GPT-5.2
  "gpt-4o",                 // GPT-4o
  "gpt-4o-mini",            // GPT-4o mini
  "gpt-4-turbo",            // GPT-4 Turbo
  "o1",                     // O1
  "o1-mini",                // O1 mini
  "o3",                     // O3
  "o3-mini",                // O3 mini
  "o4-mini",                // O4 mini
  "claude-opus-4.6",        // Claude Opus 4.6（via Copilot）
  "claude-sonnet-4.6",      // Claude Sonnet 4.6
  "gemini-3-pro",           // Gemini 3 Pro
  "gemini-3-flash-preview", // Gemini 3 Flash Preview
];
```

### 行為

```typescript
// normalizeBaseUrl()：正規化 BASE_URL（移除尾端 /）
// parseModelIds()：解析模型 ID 清單
// authHeader = false：不傳 Authorization header（由 proxy 處理認證）
```

---

## 9. GitHub Copilot 認證（copilot-proxy）

### 常數

```typescript
const GITHUB_COPILOT = {
  CLIENT_ID: "Iv1.b507a08c87ecfe98",
  DEVICE_CODE_URL: "https://github.com/login/device/code",
  ACCESS_TOKEN_URL: "https://github.com/login/oauth/access_token",
};
```

### Device Flow 認證（2 步驟）

```typescript
// 步驟 1：請求 device code
POST https://github.com/login/device/code
Body: { client_id: CLIENT_ID, scope: "copilot" }
回傳: { device_code, user_code, verification_uri, expires_in, interval }
→ 顯示 user_code 給用戶，引導至 verification_uri

// 步驟 2：輪詢 access token
POST https://github.com/login/oauth/access_token
Body: { client_id, device_code, grant_type: "device_code" }
每 interval 秒輪詢一次，直到：
- access_token 取得（成功）
- expires_in 超過（失敗）
- 用戶拒絕（失敗）
```

### Token 預先刷新邏輯

```typescript
// Copilot token 管理：
// - 偵測 token 過期時間
// - 在過期前 5 分鐘主動刷新
// - 刷新失敗 → 重新走 Device Flow
```

---

## 10. google-gemini-cli-auth Extension

```typescript
const GEMINI_CLI_AUTH = {
  PROVIDER_ID: "google-gemini-cli-auth",
  DEFAULT_MODEL: "gemini-3-pro",
  ALIASES: ["gemini", "google-gemini"],
};

// 4 個環境變數
const GEMINI_ENV_VARS = [
  "GOOGLE_GEMINI_CLI_ACCESS_TOKEN",   // OAuth access token
  "GOOGLE_GEMINI_CLI_REFRESH_TOKEN",  // OAuth refresh token
  "GOOGLE_GEMINI_CLI_PROJECT_ID",     // Google Cloud project ID
  "GOOGLE_GEMINI_CLI_EXPIRY",         // Token 過期時間（ISO 8601）
];
```

### PKCE + localhost OAuth 流程

```typescript
// PKCE（Proof Key for Code Exchange）認證流程：
// 1. 生成 code_verifier（隨機字串）
// 2. 計算 code_challenge = BASE64URL(SHA256(code_verifier))
// 3. 啟動 localhost callback server（監聽 localhost:PORT）
// 4. 開啟瀏覽器到 Google OAuth URL（含 code_challenge）
// 5. 用戶授權 → Google 重定向到 localhost:PORT?code=...
// 6. 用 code + code_verifier 換取 access_token + refresh_token
// 7. 儲存 token 到環境變數（GOOGLE_GEMINI_CLI_*）

// credentialExtra.projectId：Google Cloud project ID（必填）
```

---

## 11. sglang Extension

```typescript
const SGLANG = {
  PROVIDER_ID: "sglang",
  DEFAULT_BASE_URL: "http://127.0.0.1:30000",
  ENV_VARS: ["SGLANG_BASE_URL", "SGLANG_API_KEY"],
  DISCOVERY_ORDER: "late",  // 晚於內建 provider 載入
};
```

### Self-Hosted Provider 共用模式（sglang + vllm）

```typescript
// 3 個共用函式：
// promptAndConfigureSelfHosted()    → 互動式配置
// configureNonInteractiveSelfHosted() → 非互動式配置（CI/腳本）
// discoverOpenAICompatible()        → 探索 OpenAI 相容端點

// discovery.order = "late"：
// 確保在官方 provider 之後載入，避免衝突
```

---

## 12. vllm Extension

```typescript
const VLLM = {
  PROVIDER_ID: "vllm",
  DEFAULT_BASE_URL: "http://127.0.0.1:8000",
  ENV_VARS: ["VLLM_BASE_URL", "VLLM_API_KEY"],
  DISCOVERY_ORDER: "late",
};
// buildVllmProvider() → 建立 vllm provider 實例
// 與 sglang 完全對稱（共用相同模式）
```

---

## 13. minimax-portal-auth Extension

```typescript
const MINIMAX = {
  PROVIDER_ID: "minimax-portal-auth",
  DEFAULT_BASE_URL_CN:     "https://api.minimax.chat",  // 中國端點
  DEFAULT_BASE_URL_GLOBAL: "https://api.minimaxi.chat", // 全球端點
  DEFAULT_CONTEXT_WINDOW:  200000,  // 200K context window
};
// 支援 CN / Global 雙端點切換
```

---

## 14. qwen-portal-auth Extension

```typescript
const QWEN = {
  PROVIDER_ID: "qwen-portal-auth",
  DEFAULT_BASE_URL: "https://dashscope.aliyuncs.com/compatible-mode/v1",
  DEFAULT_CONTEXT_WINDOW: 128000,   // 128K context window
  OAUTH_PLACEHOLDER: "n/a",         // OAuth token placeholder
};
```

---

## 15. 環境變數對照表（完整）

| Provider | 環境變數 |
|----------|---------|
| `anthropic` | `ANTHROPIC_API_KEY` |
| `openai` | `OPENAI_API_KEY` |
| `google` | `GOOGLE_API_KEY` |
| `aws-bedrock` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION` |
| `azure-openai` | `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT` |
| `ollama` | `OLLAMA_BASE_URL`（可選，預設 localhost:11434） |
| `openrouter` | `OPENROUTER_API_KEY` |
| `nvidia` | `NVIDIA_API_KEY` |
| `minimax` | `MINIMAX_API_KEY` |
| `qwen-portal-auth` | Qwen portal OAuth token |
| `google-gemini-cli-auth` | 4 個 `GOOGLE_GEMINI_CLI_*` 變數 |
| `sglang` | `SGLANG_BASE_URL`（可選） |
| `vllm` | `VLLM_BASE_URL`（可選） |

---

## 16. Model Alias 解析

```typescript
// model alias 系統：允許使用別名（如 "opus" → "claude-opus-4-6"）
// google-gemini-cli-auth aliases: ["gemini", "google-gemini"]
// 解析順序：
// 1. 精確匹配 model ID
// 2. alias 查表
// 3. provider 內部 default
// 4. 系統 DEFAULT_MODEL
```
