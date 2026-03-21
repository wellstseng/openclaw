# Memory / Plugin / Config 系統

## 1. Memory/RAG 系統（src/memory/ — 95 files）

### 向量資料庫
- 核心：Node.js 原生 `node:sqlite` + `sqlite-vec` 擴展
- 向量搜尋：`vec_distance_cosine()` SQL 函式
- 環境變數：`OPENCLAW_SQLITE_VEC_EXTENSION`（自訂擴展路徑）

### 6 種 Embedding 提供者
```
openai (text-embedding-3-small/large) | local (node-llama-cpp) |
gemini | voyage | mistral | ollama
```

### 雙層搜尋引擎
| 層 | 方法 | 查詢方式 |
|----|------|---------|
| 向量搜尋 | `searchVector()` | 餘弦相似度 (1 - distance) |
| 全文搜尋 | `searchKeyword()` | BM25 + SQL MATCH |
| 降級搜尋 | `listChunks()` | 客戶端餘弦相似度 |

### MemorySearchManager 介面
```typescript
search(query, opts?) → MemorySearchResult[]
readFile(params) → { text, path }
status() → MemoryProviderStatus
sync?(params?) → void
probeEmbeddingAvailability() → MemoryEmbeddingProbeResult
probeVectorAvailability() → boolean
```

### 資料表結構
- `chunks_vec`: id + embedding (Float32 Blob)
- `chunks_fts`: FTS5 虛擬表 (id, path, start_line, end_line, text, source, model)
- `embedding_cache`: 加速重複查詢
- source 類型: `"memory"` | `"sessions"`

---

## 2. Plugin 系統（src/plugins/ — 83 files）

### 生命週期
```
Discovery → Validation → Loading → Registration → Activation
```

| 階段 | 檔案 | 功能 |
|------|------|------|
| Discovery | discovery.ts (20KB) | 掃描路徑 + 快取(1000ms) |
| Registry | registry.ts (18KB) | PluginRecord 維護 |
| Loader | loader.ts (28KB) | jiti 動態載入 + SDK 別名 |
| Runtime | runtime/index.ts | 執行時 API |
| Manifest | manifest-registry.ts | package.json 中的 openclaw 欄位 |

### Plugin API 表面
```typescript
api.registerTool(tool | factory)      // AI Tool
api.registerHook(events, handler)      // 事件鉤子
api.registerHttpRoute(params)          // HTTP 路由
api.registerChannel(plugin)            // 頻道
api.registerGatewayMethod(method, handler)
api.registerCli(registrar)             // CLI 命令
api.registerService(service)           // 背景服務
api.registerProvider(provider)         // LLM Provider
api.registerCommand(command)           // 繞過 LLM 的指令
api.registerContextEngine(id, factory) // Context Engine
```

### Plugin 來源
```
origin: "bundled" | "config" | "workspace" | "npm"
```

### SubAgent API（透過 plugin runtime）
```typescript
api.runtime.subagent.run(params)               → { runId }
api.runtime.subagent.waitForRun(params)         → { status, error? }
api.runtime.subagent.getSessionMessages(params) → { messages }
api.runtime.subagent.deleteSession(params)      → void
```

---

## 3. Plugin SDK（src/plugin-sdk/ — 109 files）

主要出口（index.ts — 800 行，600+ exports）：
- PluginRuntime / OpenClawPluginApi / GatewayRequestHandler
- 頻道配置幫手：createAccountListHelpers / buildChannelConfigSchema
- 安全工具：fetchWithSsrfGuard / isBlockedHostname / isPrivateIpAddress
- 檔案鎖：acquireFileLock / withFileLock
- 去重：createPersistentDedupe
- 每個頻道的專用 SDK（discord.ts, telegram.ts, slack.ts 等）

---

## 4. Config 系統（src/config/ — 226 files）

### 設定檔
- 格式：JSON5（允許註解、尾隨逗號）
- 路徑：`~/.openclaw/openclaw.json`
- 環境變數替換：`${ENV_VAR}` 語法
- 支援 `#include` 指令

### 載入管道
```
讀取 + JSON5 解析 → 環境變數替換 → include 解析 → 默認值 → Zod 驗證 → 快取
```

### 三層驗證
1. `validateConfigObjectRaw()` — 合併前
2. `validateConfigObjectWithPlugins()` — plugin Zod schema 合併後
3. `ConfigValidationResult` — { ok, config?, issues? }

### Secret 處理
```typescript
SecretInput = string | { env: string } | { prompt: string } | { file: string }
```

### 關鍵路徑常數
```
STATE_DIR = ~/.openclaw
CONFIG_PATH = ~/.openclaw/openclaw.json
OAUTH_DIR = ~/.openclaw/credentials
GATEWAY_PORT = 18789
```

### Config 型別模組（~35 個子模組）
types.base / types.agents / types.channels / types.models /
types.hooks / types.memory / types.plugins / types.secrets
