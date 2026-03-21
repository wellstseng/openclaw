# 00-OVERVIEW — OpenClaw 入門總覽

> 版本：v1.0（2026-03-21）
> 來源：openclaw-knowledge-base.md + F058/F059/F060 五輪原始碼驗證整合

---

## 1. OpenClaw 是什麼

OpenClaw 是一套**多平台 AI Agent 框架**，核心設計目標：

- **Multi-channel**：單一 Agent 同時服務 Discord / Telegram / WhatsApp / iMessage / Signal 等 20+ 平台
- **Multi-provider**：支援 Anthropic / OpenAI / Google / AWS Bedrock / Ollama / sglang / vllm 等 17+ LLM provider
- **Extension-first**：所有平台整合、LLM 整合、記憶體後端均透過 Extension/Plugin 動態載入
- **ACP（Agent Communication Protocol）**：支援 Agent 間通訊與子 Agent 生成
- **Self-hosted**：可完全本地部署，無需雲端依賴

### 版本資訊
- 知識庫版本：v1.0（2026-03-21）
- 原始碼驗證輪次：5 輪（F057 × 3 + F058 + F059 + F060）
- 預設 Provider：`anthropic`
- 預設 Model：`claude-opus-4-6`

---

## 2. 系統架構圖（文字版）

```
┌─────────────────────────────────────────────────────────────┐
│                        CLI / TUI                            │
│   (src/commands/ src/cli/ src/tui/)                         │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                     Gateway HTTP Server                      │
│   13-stage Pipeline: Auth → Rate Limit → Route → Agent      │
│   (src/gateway/ + src/routing/)                             │
└──────────┬──────────────────────────────┬───────────────────┘
           │                              │
┌──────────▼──────────┐      ┌────────────▼────────────────────┐
│   Agent Run Loop    │      │    Channel Providers (~20)       │
│   (src/agent/       │      │  Discord / Telegram / WhatsApp  │
│    run.ts)          │      │  Signal / iMessage / Matrix ...  │
│  - System Prompt    │      │  (extensions/*/  src/discord/)  │
│  - Tool Execution   │      └────────────┬────────────────────┘
│  - Context Mgmt     │                   │
│  - Failover         │      ┌────────────▼────────────────────┐
└──────────┬──────────┘      │      Inbound Pipeline            │
           │                 │  Debounce → Preflight → Queue    │
           │                 │  → Worker → Process → Delivery  │
┌──────────▼──────────┐      └─────────────────────────────────┘
│   LLM Providers     │
│  (~17 providers)    │      ┌─────────────────────────────────┐
│  Anthropic / OAI /  │      │        Memory System            │
│  Google / Bedrock / │      │  LanceDB (vector) + SQLite FTS5 │
│  Ollama / sglang... │      │  Hybrid Search + Temporal Decay │
└─────────────────────┘      │  (src/memory/ extensions/       │
                             │   memory-lancedb/ memory-core/) │
                             └─────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                Plugin & Extension System                     │
│  44 Extensions = 20 Channel + 7 AI Provider + 3 Memory      │
│               + 14 Runtime                                   │
│  Priority: config > workspace > bundled > global            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   ACP (Agent Comm Protocol)                  │
│  Sessions: sessions.json + transcript JSONL                  │
│  Persistent Bindings: discord / telegram only               │
└─────────────────────────────────────────────────────────────┘
```

### 模組依賴關係

| 上層 | 依賴 |
|------|------|
| Gateway | Agent, Routing, Security |
| Agent run.ts | Providers, Memory, Plugin-SDK, Process |
| Channel Extensions | Plugin-SDK, Security (GuardedFetch) |
| ACP | Sessions, Agent |
| Memory | LanceDB, SQLite FTS5 |
| TUI | Agent, Sessions, Terminal |

---

## 3. 核心概念詞彙表

| 詞彙 | 定義 |
|------|------|
| **Agent** | 核心 AI 執行單元。有三種模式：`run`（單次）、`session`（對話）、`subagent`（子 Agent）。對應 `AgentKind` enum |
| **Gateway** | HTTP 伺服器，接收所有入站請求，透過 13-stage Pipeline 路由到 Agent |
| **Channel** | 聊天平台的抽象介面（Discord / Telegram / WhatsApp 等）。每個 Channel 為獨立 Extension |
| **Provider** | LLM 後端抽象（Anthropic / OpenAI / Google 等）。每個 Provider 為獨立 Extension |
| **Extension** | 動態載入模組，實作 `registerXxx()` 介面。是 Plugin 的正式稱呼 |
| **Plugin** | 非官方術語，指第三方 Extension。優先順序：config > workspace > bundled > global |
| **ACP** | Agent Communication Protocol，定義 Agent 間如何傳遞訊息、共用 Session |
| **Session** | Agent 對話狀態，儲存為 `sessions.json`（metadata）+ JSONL（transcript）。有 TTL + Disk Budget |
| **CommandLane** | 執行上下文：Main / Cron / Subagent / Nested |
| **Preflight** | Channel 層的入站驗證閘門（Discord 有 40 個） |
| **GuardedFetch** | 帶 SSRF 防護的 fetch wrapper |
| **HEARTBEAT_TOKEN** | `__heartbeat__`，心跳訊息識別符 |
| **SILENT_REPLY_TOKEN** | `__silent_reply__`，靜默回覆識別符 |
| **MMR** | Maximal Marginal Relevance，記憶搜尋多樣化算法（lambda=0.7） |
| **QMD** | Query Memory Database，記憶查詢協定 |

---

## 4. 重要數字常數速查表

### Agent 系統

| 常數 | 值 | 來源 |
|------|----|------|
| `MIN_ITERATIONS` | 32 | run.ts |
| `MAX_ITERATIONS` | 160 | run.ts |
| `ITER_BASE` | 24 | run.ts 公式：`24 + Math.max(1, n) * 8` |
| `ITER_PER_PROFILE` | 8 | run.ts |
| `CONTEXT_GUARD_RATIO` | 0.85 | context window guard 觸發比例 |
| `CONTEXT_WARN_RATIO` | 0.75 | context window warn 比例 |
| `CONTEXT_HARD_MIN` | 4096 tokens | 最小保留空間 |
| `MAX_SUBAGENT_DEPTH` | 5 | 子 Agent 最大巢狀深度 |
| `DEFAULT_PROVIDER` | `"anthropic"` | provider 預設值 |
| `DEFAULT_MODEL` | `"claude-opus-4-6"` | model 預設值 |

### Memory 系統

| 常數 | 值 | 來源 |
|------|----|------|
| Hybrid Search vector 權重 | 0.7 | memory 搜尋 |
| Hybrid Search text 權重 | 0.3 | memory 搜尋 |
| `candidateMultiplier` | 4 | Hybrid Search 候選集倍數 |
| MMR lambda | 0.7 | MMR re-ranking |
| Temporal decay halfLife | 可設定 | 預設 disabled |
| `captureMaxChars` | 4096（預設） | auto-capture 最大字元 |
| sentinel row id | `"__schema__"` | LanceDB schema 版本追蹤 |

### Discord 系統

| 常數 | 值 | 來源 |
|------|----|------|
| `DEFAULT_MAX_CHARS` | 2000 | chunk 字元上限 |
| `DEFAULT_MAX_LINES` | 17 | chunk 行數上限 |
| `MENTION_TTL_MS` | - | Thread binding mention TTL |
| Thread sweep interval | 120s | thread-bindings sweep |
| Thread idle timeout | 24h | thread-bindings idle |
| Echo suppression window | 30s | unbind 後 webhook 抑制 |
| Delivery retry 次數 | 3 | reply-delivery.ts |
| Delivery retry backoff | 1s–30s exponential | reply-delivery.ts |

### Gateway 系統

| 常數 | 值 | 來源 |
|------|----|------|
| Delta throttle | - | chat event delta throttle |
| Voice transcript dedupe | 1.5s | gateway constants |
| Tool event TTL | 10min / 30s grace | gateway constants |
| Control plane rate limit | sliding window by deviceId\|clientIp | security |
| Auth rate limit | 10 req/60s，lock 5min | gateway |
| Hook auth rate limit | 20 req/60s | gateway |

### Auto-Reply 系統

| 常數 | 值 | 來源 |
|------|----|------|
| `HEARTBEAT_TOKEN` | `"__heartbeat__"` | auto-reply |
| `SILENT_REPLY_TOKEN` | `"__silent_reply__"` | auto-reply |
| NO_REPLY regex | `/^(\.|\.\.)?\s*$/` | auto-reply |

### Plugin 系統

| 常數 | 值 | 來源 |
|------|----|------|
| Plugin discovery cache key | 含 Unix UID | plugin discovery |
| Hook 名稱數量 | 24 個 | plugin registry |
| Extension 總數 | 44 個 | extension registry |

### 特殊 Provider 常數

| 常數 | 值 | 來源 |
|------|----|------|
| Copilot `DEFAULT_GATEWAY_PORT` | 18789 | device-pair extension |
| sglang `DEFAULT_BASE_URL` | `127.0.0.1:30000` | sglang extension |
| vllm `DEFAULT_BASE_URL` | `127.0.0.1:8000` | vllm extension |
| GitHub Copilot `CLIENT_ID` | `"Iv1.b507a08c87ecfe98"` | copilot-proxy |
| Copilot token 預先刷新 | 5 分鐘前 | copilot-proxy |
| Copilot `authHeader` | `false` | copilot-proxy |
| Security `DEFAULT_MAX_REDIRECTS` | 3 | guarded-fetch |

### Channels 特殊常數

| 常數 | 值 | 來源 |
|------|----|------|
| BlueBubbles `REPLY_CACHE_MAX` | 2000 | bluebubbles |
| BlueBubbles reply cache TTL | 6h | bluebubbles |
| Nextcloud Talk 預設 port | 8788 | nextcloud-talk extension |
| Logging `MAX_LOG_AGE_MS` | 24h | src/logging |
| Logging `DEFAULT_MAX_LOG_FILE_BYTES` | 500MB | src/logging |
| TTS `TEMP_FILE_CLEANUP_DELAY_MS` | - | src/tts |
| TTS voice ID regex | `/^[a-zA-Z0-9]{10,40}$/` | src/tts |

---

## 5. 重要環境變數一覽表

| 環境變數 | 用途 |
|----------|------|
| `ANTHROPIC_API_KEY` | Anthropic API 金鑰 |
| `OPENAI_API_KEY` | OpenAI API 金鑰 |
| `GOOGLE_API_KEY` | Google AI API 金鑰 |
| `AWS_ACCESS_KEY_ID` | AWS Bedrock 存取金鑰 |
| `AWS_SECRET_ACCESS_KEY` | AWS Bedrock 秘密金鑰 |
| `AWS_REGION` | AWS Bedrock 區域 |
| `OLLAMA_BASE_URL` | Ollama 服務 URL |
| `GOOGLE_GEMINI_CLI_*` | Google Gemini CLI 認證（4 個 ENV） |
| `GITHUB_COPILOT_TOKEN` | GitHub Copilot token |
| `DISCORD_TOKEN` | Discord Bot token |
| `TELEGRAM_BOT_TOKEN` | Telegram Bot token |
| `OPENROUTER_API_KEY` | OpenRouter API 金鑰 |
| `NVIDIA_API_KEY` | NVIDIA API 金鑰 |
| `MINIMAX_API_KEY` | MiniMax API 金鑰（CN/Global） |

---

## 6. 各層文件閱讀建議

| 情況 | 閱讀哪個檔 |
|------|----------|
| 剛接觸 OpenClaw，需要全面理解 | **00-OVERVIEW.md**（本文） |
| 調試 Agent 行為、理解迭代邏輯 | **01-CORE.md** |
| 調試 Discord 整合、preflight 問題 | **02-DISCORD.md** |
| ACP / 子 Agent / Session 管理 | **03-SESSIONS-ACP.md** |
| 記憶搜尋、向量資料庫、LanceDB | **04-MEMORY.md** |
| 新增平台 Channel / 調試特定平台 | **05-CHANNELS.md** |
| 新增 LLM Provider / 調試 streaming | **06-PROVIDERS.md** |
| SSRF 防護、exec 安全、rate limit | **07-SECURITY.md** |
| Extension 開發、CLI 命令參考 | **08-EXTENSIONS-CLI.md** |
| Cron、TTS、Daemon、Browser 等 | **09-MISC.md** |
