# 16-MEMORY-CONTEXT-DEEP.md — Memory/RAG + Context Engine 深入

> Phase 4-3 | 2026-03-13 | 掃描 `src/memory/` 111 files + `extensions/memory-core/` 3 files + `extensions/memory-lancedb/` 5 files + `src/context-engine/` 6 files，共 ~125 files

## 目錄

1. [架構鳥瞰](#1-架構鳥瞰)
2. [Builtin Memory 核心（src/memory/）](#2-builtin-memory-核心)
3. [搜尋管線](#3-搜尋管線)
4. [Embedding 子系統](#4-embedding-子系統)
5. [Sync 管線](#5-sync-管線)
6. [QMD 備援後端](#6-qmd-備援後端)
7. [Extensions — memory-core](#7-extensions--memory-core)
8. [Extensions — memory-lancedb](#8-extensions--memory-lancedb)
9. [Context Engine](#9-context-engine)
10. [跨系統整合圖](#10-跨系統整合圖)
11. [邊界條件與陷阱](#11-邊界條件與陷阱)
12. [關鍵常量速查](#12-關鍵常量速查)
13. [C# 概念對照](#13-c-概念對照)

---

## 1. 架構鳥瞰

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Agent Prompt Loop                            │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │ Context       │    │ Memory Search    │    │ Memory LanceDB   │  │
│  │ Engine        │    │ Manager          │    │ (Extension)      │  │
│  │ (assemble/    │    │ (getMemory       │    │ (auto-recall/    │  │
│  │  compact)     │    │  SearchManager)  │    │  auto-capture)   │  │
│  └──────┬───────┘    └────────┬─────────┘    └────────┬─────────┘  │
│         │                     │                        │            │
│         ▼                     ▼                        ▼            │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │ Legacy /      │    │ MemoryIndex      │    │ LanceDB          │  │
│  │ Custom Engine │    │ Manager          │    │ (vector store)   │  │
│  │              │    │ (builtin)        │    │                  │  │
│  └──────────────┘    ├──────────────────┤    └──────────────────┘  │
│                      │ FallbackMemory   │                          │
│                      │ Manager          │                          │
│                      │ (QMD → builtin)  │                          │
│                      └────────┬─────────┘                          │
│                               │                                    │
│              ┌────────────────┼────────────────┐                   │
│              ▼                ▼                ▼                   │
│       ┌────────────┐  ┌────────────┐  ┌──────────────┐            │
│       │ SQLite +   │  │ FTS5       │  │ sqlite-vec   │            │
│       │ chunks     │  │ (keyword)  │  │ (vector)     │            │
│       └────────────┘  └────────────┘  └──────────────┘            │
└─────────────────────────────────────────────────────────────────────┘
```

**三大子系統**：

| 子系統 | 職責 | 入口 |
|--------|------|------|
| **Builtin Memory** (`src/memory/`) | SQLite + sqlite-vec + FTS5，雙層搜尋 + 6 embedding providers | `MemoryIndexManager.get()` |
| **Memory Extensions** (`extensions/`) | memory-core：Tool + CLI 註冊；memory-lancedb：LanceDB 長期記憶 | Plugin `register()` |
| **Context Engine** (`src/context-engine/`) | 訊息組裝 → Token 管理 → Compaction | `resolveContextEngine()` |

---

## 2. Builtin Memory 核心

### 2.1 類別繼承

```
MemoryIndexManager
  extends MemoryManagerEmbeddingOps        // Embedding 批次 + 快取
    extends MemoryManagerSyncOps           // 檔案監視 + 增量同步
```

### 2.2 核心檔案對照

| 檔案 | 行數 | 職責 |
|------|------|------|
| `manager.ts` | ~760 | 主類別，search()、get()、status()、生命週期 |
| `manager-sync-ops.ts` | ~1150 | 檔案掃描、chunking、增量同步、file watcher |
| `manager-embedding-ops.ts` | ~760 | embedding 執行、快取、retry、batch 調度 |
| `manager-search.ts` | ~200 | searchVector() + searchKeyword() SQL 執行 |
| `search-manager.ts` | ~210 | 後端路由：builtin vs QMD + FallbackMemoryManager |
| `hybrid.ts` | ~110 | 混合搜尋結果合併 + BM25 正規化 |
| `mmr.ts` | ~180 | Maximal Marginal Relevance 重排序 |
| `temporal-decay.ts` | ~130 | 時間衰減（半衰期指數衰減） |
| `query-expansion.ts` | ~400 | FTS-only 模式：關鍵字提取 + 停用詞過濾 |
| `memory-schema.ts` | ~85 | SQLite 建表（chunks / chunks_vec / chunks_fts / files / meta / embedding_cache） |
| `backend-config.ts` | ~300 | 後端設定解析（builtin vs QMD） |
| `embeddings.ts` | ~330 | Provider factory + 自動偵測 + fallback chain |
| `internal.ts` | ~250 | chunking、cosine similarity、路徑工具 |
| `session-files.ts` | ~110 | JSONL session 轉平文字 + lineMap 映射 |

### 2.3 Manager 取得 & 快取

```typescript
// manager.ts — singleton per (agentId + config hash)
static async get(opts): Promise<MemoryIndexManager> {
  const cacheKey = buildCacheKey(opts);
  if (INDEX_CACHE.has(cacheKey)) return INDEX_CACHE.get(cacheKey);
  if (INDEX_CACHE_PENDING.has(cacheKey)) return INDEX_CACHE_PENDING.get(cacheKey);
  // resolve config → create embedding provider → open SQLite → new instance
  const pending = create(opts);
  INDEX_CACHE_PENDING.set(cacheKey, pending);
  const mgr = await pending;
  INDEX_CACHE.set(cacheKey, mgr);
  INDEX_CACHE_PENDING.delete(cacheKey);
  return mgr;
}
```

### 2.4 後端選擇（search-manager.ts）

```
getMemorySearchManager({ cfg, agentId, purpose })
  │
  ├─ backend === "qmd" && configured
  │   └─ QmdMemoryManager.create()
  │       ├─ 成功 → FallbackMemoryManager(primary=QMD, fallback=builtin)
  │       └─ 失敗 → fallback to builtin only
  │
  └─ else → MemoryIndexManager 直接使用
```

**FallbackMemoryManager**：透明失敗恢復，primary 失敗時自動切換到 fallback，evict cache 重試。

### 2.5 SQLite Schema

```sql
-- 元資料
CREATE TABLE meta (key TEXT PRIMARY KEY, value TEXT);

-- 檔案追蹤
CREATE TABLE files (
  path TEXT PRIMARY KEY,
  hash TEXT, mtime REAL, size INTEGER
);

-- 文字 chunks
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,           -- "{path}::{hash[0:16]}"
  path TEXT, start_line INTEGER, end_line INTEGER,
  text TEXT, embedding TEXT,     -- JSON float array（fallback 用）
  model TEXT, source TEXT,       -- "memory" | "sessions"
  provider TEXT, provider_key TEXT
);
CREATE INDEX idx_chunks_path ON chunks(path);
CREATE INDEX idx_chunks_source ON chunks(source);

-- 向量索引（sqlite-vec virtual table）
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  embedding FLOAT[{dims}]       -- 維度由首次 embedding 決定
);

-- 全文索引（FTS5 virtual table）
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text, path, model, source,
  content='chunks', content_rowid='rowid'
);

-- Embedding 快取
CREATE TABLE embedding_cache (
  provider TEXT, model TEXT, provider_key TEXT,
  hash TEXT, embedding BLOB, updated_at REAL,
  PRIMARY KEY (provider, model, provider_key, hash)
);
CREATE INDEX idx_embedding_cache_updated_at ON embedding_cache(updated_at);
```

---

## 3. 搜尋管線

### 3.1 完整搜尋流程

```
search(query, { maxResults, minScore, sessionKey })
  │
  ├─ warmSession(sessionKey)              // 首次 session 觸發同步
  │
  ├─ if dirty → async sync()             // 背景增量同步
  │
  ├─ 計算參數：
  │   minScore = opts.minScore ?? settings.query.minScore
  │   maxResults = opts.maxResults ?? settings.query.maxResults
  │   candidates = min(200, max(1, floor(maxResults × candidateMultiplier)))
  │
  ├─ 【無 Provider】FTS-only 模式：
  │   ├─ extractKeywords(query)           // 分詞 + 停用詞過濾
  │   ├─ searchKeyword(term) per keyword  // 各詞獨立搜 FTS
  │   └─ 合併去重（同 chunk ID 取最高分）
  │
  └─ 【有 Provider】Embedding 模式：
      ├─ embedQuery(query) → queryVec
      │
      ├─ 並行：
      │   ├─ searchVector(queryVec, candidates)    // sqlite-vec
      │   └─ searchKeyword(query, candidates)      // FTS5（若啟用）
      │
      ├─ mergeHybridResults()：
      │   score = vectorWeight × vectorScore + textWeight × textScore
      │
      ├─ applyTemporalDecay()（若啟用）：
      │   score × exp(-λ × ageInDays)，λ = ln(2) / halfLifeDays
      │
      ├─ applyMMR()（若啟用）：
      │   MMR = λ × relevance - (1-λ) × max_sim_to_selected
      │
      ├─ filter: score ≥ minScore
      │
      └─ fallback: 若 strict=0 且 keyword 有結果 → 放寬 minScore 回傳
```

### 3.2 Vector Search — SQL 實作

```sql
-- manager-search.ts::searchVector()
SELECT c.id, c.path, c.start_line, c.end_line,
       substr(c.text, 1, :snippetMaxChars) AS text,
       c.source,
       vec_distance_cosine(v.embedding, :queryVec) AS dist
FROM chunks_vec v
JOIN chunks c ON c.id = v.id
WHERE c.model = :model AND c.source IN (:sources)
ORDER BY dist ASC
LIMIT :limit
```

**距離 → 分數轉換**：`score = 1 - distance`（cosine distance 0~2 → similarity -1~1）

**Fallback**（sqlite-vec 不可用）：
- 載入所有 chunks → 程式內 cosine similarity 計算
- `similarity(a, b) = dot(a,b) / (‖a‖ × ‖b‖)`

### 3.3 Keyword Search — FTS5 BM25

```sql
SELECT id, path, source, start_line, end_line, text,
       bm25(chunks_fts) AS rank
FROM chunks_fts
WHERE chunks_fts MATCH :query AND model = :model AND source IN (:sources)
ORDER BY rank ASC
LIMIT :limit
```

**FTS Query 建構**（hybrid.ts::buildFtsQuery）：
```
"that API thing" → tokenize → ["that", "API", "thing"]
→ '"that" AND "API" AND "thing"'
```

**BM25 → 正規化分數**：`textScore = relevance / (1 + relevance)`

### 3.4 混合結果合併（hybrid.ts）

```
1. 收集 vectorResults + keywordResults
2. 以 chunk ID 為 key 合併到 Map
3. score = vectorWeight × vectorScore + textWeight × textScore
4. 依 score 降序排列
```

### 3.5 時間衰減（temporal-decay.ts）

```
偵測日期檔案：memory/YYYY-MM-DD.md → 計算 ageInDays
一般記憶檔：MEMORY.md, memory/*.md → 常青（無衰減）

multiplier = exp(-λ × ageInDays)
λ = ln(2) / halfLifeDays

halfLifeDays=30 時：30 天 → 0.5x，60 天 → 0.25x

預設：disabled（opt-in）
```

### 3.6 MMR 重排序（mmr.ts）

```
目的：兼顧相關性與多樣性

演算法：
  1. 正規化分數到 [0, 1]
  2. 預先 tokenize 所有 snippet（Jaccard similarity 用）
  3. 迭代選取：
     selected = []
     for each slot:
       MMR(d) = λ × relevance(d) - (1-λ) × max_sim(d, selected)
       選最高 MMR 的 candidate

配置：
  enabled: false（opt-in）
  lambda: 0.7（偏向相關性）
  複雜度：~O(n² × token_count)
```

### 3.7 Query Expansion（FTS-only Fallback）

當無 embedding provider 時，`extractKeywords()` 將自然語言查詢轉為 FTS 關鍵字：

```
1. Unicode 分詞：match(/[\p{L}\p{N}_]+/gu)
2. 過濾停用詞（英/西/中/日）
3. 各關鍵字獨立搜 FTS
4. 合併去重，保留最高 BM25 分數
```

---

## 4. Embedding 子系統

### 4.1 Provider 清單

| Provider | 預設模型 | 維度 | Max Tokens | Batch API |
|----------|----------|------|-----------|-----------|
| OpenAI | text-embedding-3-small | 1536 | 8192 | ✅（Files API） |
| Ollama | nomic-embed-text | 模型依賴 | 模型依賴 | ❌（序列） |
| Gemini | gemini-embedding-001 | 自訂 | 自訂 | ✅ |
| Voyage | voyage-4-large | — | 32000 | ✅ |
| Mistral | mistral-embed | — | — | ❌（remote only） |

### 4.2 Provider 選擇邏輯（embeddings.ts）

```
provider === "auto"
  → 依序嘗試：["openai", "gemini", "voyage", "mistral"]
  → 失敗 → 使用 fallback provider
  → Ollama 不在 auto 內（需顯式設定）

provider === 明確指定
  → 使用指定 provider
  → 不可用 → fallback
  → 全部不可用 → FTS-only 模式（provider = null）
```

### 4.3 EmbeddingProvider 介面

```typescript
type EmbeddingProvider = {
  id: "openai" | "local" | "gemini" | "voyage" | "mistral" | "ollama"
  model: string
  maxInputTokens?: number
  embedQuery: (text: string) => Promise<number[]>
  embedBatch: (texts: string[]) => Promise<number[][]>
}
```

### 4.4 Embedding 快取

```
Table: embedding_cache
  PK: (provider, model, provider_key, hash)

provider_key = hash of:
  - Provider ID + Model
  - API key metadata（多租戶安全）
  - Headers（自訂端點）

操作：
  - 索引前批次載入：SELECT WHERE hash IN (...)
  - 索引後更新：INSERT OR REPLACE
  - 裁剪：保留最近使用的 maxEntries 筆
```

### 4.5 Batch Embedding 處理器

三家支援 Batch API：

| Provider | 流程 | 上限 |
|----------|------|------|
| **OpenAI** | 建 JSONL → 上傳 Files API → 提交 Batch → Poll → 下載結果 | 50,000/batch |
| **Gemini** | 類似 OpenAI：提交 → Poll → 取結果 | — |
| **Voyage** | 原生 batch endpoint → Poll → 取結果 | — |

**Batch 設定**：
```typescript
batch: {
  enabled: true,          // 預設開啟
  wait: false,            // 非阻塞
  concurrency: 2,         // 平行 batch 數
  pollIntervalMs: 10_000, // 狀態輪詢頻率
  timeoutMs: 120_000      // 等待超時 2min
}
```

### 4.6 重試 & 超時策略

```
重試：最多 3 次，指數退避 500ms → 8000ms
超時：
  - Remote query: 60s
  - Local query: 5min
  - Remote batch: 2min
  - Local batch: 10min
並行度：4 concurrent embedding tasks
```

---

## 5. Sync 管線

### 5.1 Dirty Tracking

```
dirty: boolean              // 全域：memory/ 檔案異動
sessionsDirty: boolean      // Session：sessions/ JSONL 異動
sessionsDirtyFiles: Set     // 哪些 session 檔異動
```

### 5.2 Sync 觸發點

| 觸發 | 時機 |
|------|------|
| 顯式 | `manager.sync({ reason, force })` |
| 搜尋時 | `if (settings.sync.onSearch && dirty)` → async sync |
| Session 開始 | `warmSession(sessionKey)` |
| 定時 | `ensureIntervalSync()`（如每 30s） |
| File Watcher | chokidar 監視 memory/ + sessions/ |

### 5.3 Sync 步驟

```
1. 列舉檔案
   - Memory：MEMORY.md + memory/*.md + extraPaths
   - Sessions：agent sessions/*.jsonl

2. 建立 Entry
   - buildFileEntry() → { path, hash, mtime, size }
   - buildSessionEntry() → { path, content, lineMap }

3. 偵測變更
   - hash 比對 DB → 新增/修改 → 加入佇列

4. Chunk 內容
   - chunkMarkdown()：預設 ~100 tokens/chunk
   - Overlap：前 50 tokens 帶入下一 chunk
   - 每 chunk：{ startLine, endLine, text, hash }

5. Embedding & 索引
   - 按 token 數批次分組
   - provider.embedBatch(texts) → vectors
   - 寫入：chunks + chunks_vec (if vec) + chunks_fts (if FTS)

6. 更新 Meta
   - model, provider, providerKey, sources,
     chunkTokens, chunkOverlap, vectorDims
```

### 5.4 Session 檔案處理

```
JSONL 格式（每行一筆）：
  { "type": "message", "message": { "role": "user|assistant", "content": ... } }

處理：
  1. 讀取 agent session 目錄
  2. 解析 JSONL → 提取 user + assistant 訊息
  3. 展平為 label + text
  4. 敏感資訊遮蔽
  5. lineMap：平文字行號 → 原始 JSONL 行號
  6. chunk 並回映行號
  7. 以 source="sessions" 儲存

增量追蹤：
  - sessionsDirtyFiles 紀錄異動檔
  - 僅重新索引有變更的檔案
  - delta read：SESSION_DELTA_READ_CHUNK_BYTES = 64KB
```

### 5.5 File Watcher 忽略清單

```
IGNORED_MEMORY_WATCH_DIR_NAMES = {
  ".git", "node_modules", ".pnpm-store",
  ".venv", "venv", ".tox", "__pycache__"
}
```

---

## 6. QMD 備援後端

外部 QMD 系統作為大規模部署的替代後端。

### 6.1 設定

```typescript
ResolvedQmdConfig = {
  command: string                              // QMD 執行檔
  mcporter: { enabled, serverName, startDaemon }
  searchMode: "search" | "query" | "hybrid"    // 速度 vs 召回
  collections: ResolvedQmdCollection[]
  sessions: { enabled, exportDir?, retentionDays? }
  update: { intervalMs, debounceMs, onBoot, embedIntervalMs }
  limits: {
    maxResults, maxSnippetChars,
    maxInjectedChars, timeoutMs
  }
  includeDefaultMemory: boolean
  scope?: SessionSendPolicyConfig
}
```

### 6.2 關鍵常量

| 常量 | 值 | 用途 |
|------|-----|------|
| `SEARCH_PENDING_UPDATE_WAIT_MS` | 500 | 搜尋前等待更新完成 |
| `MAX_QMD_OUTPUT_CHARS` | 200,000 | QMD 輸出上限 |
| `QMD_EMBED_BACKOFF_BASE_MS` | 60,000 | Embedding 退避基底 |
| `QMD_EMBED_BACKOFF_MAX_MS` | 3,600,000 | 退避上限（1 小時） |
| `QMD_BM25_HAN_KEYWORD_LIMIT` | 12 | 漢字查詢關鍵字上限 |

---

## 7. Extensions — memory-core

> `extensions/memory-core/`：3 files，薄層封裝

### 7.1 Plugin Manifest

```json
{ "id": "memory-core", "kind": "memory" }
```

### 7.2 註冊內容

```typescript
register(api: OpenClawPluginApi) {
  // Tool 註冊
  api.registerTool(createMemorySearchTool());  // memory_search
  api.registerTool(createMemoryGetTool());     // memory_get

  // CLI 註冊
  api.registerCli(program => {
    api.runtime.tools.registerMemoryCli(program);  // `memory` 指令群
  });
}
```

**Tool 行為**：
- `memory_search`：語義搜尋，回傳 `MemorySearchResult[]`
- `memory_get`：讀取特定檔案片段

兩者均透過 `getMemorySearchManager()` 取得後端，處理 manager 不存在的情況。

---

## 8. Extensions — memory-lancedb

> `extensions/memory-lancedb/`：5 files，LanceDB 長期記憶

### 8.1 架構

```
┌─────────────────────────────────────────────┐
│              memory-lancedb plugin          │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │ MemoryDB │  │ Embed    │  │ Safety    │ │
│  │(LanceDB) │  │(OpenAI)  │  │ Filters   │ │
│  └────┬─────┘  └────┬─────┘  └─────┬─────┘ │
│       │              │              │       │
│  ┌────┴──────────────┴──────────────┴─────┐ │
│  │ Tools: recall / store / forget         │ │
│  ├────────────────────────────────────────┤ │
│  │ Hooks: before_agent_start (recall)     │ │
│  │        agent_end (capture)             │ │
│  ├────────────────────────────────────────┤ │
│  │ CLI: ltm list / search / stats         │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### 8.2 LanceDB Schema

```typescript
Table: "memories"
{
  id: string,           // UUID
  text: string,         // 記憶內容
  vector: number[],     // Embedding 向量
  importance: number,   // 0-1 重要度
  category: "preference" | "fact" | "decision" | "entity" | "other",
  createdAt: number     // Unix timestamp
}
```

**搜尋**：L2 distance → `similarity = 1 / (1 + distance)`

### 8.3 設定

```typescript
{
  embedding: {
    apiKey: string,        // 必填（支援 ${ENV_VAR}）
    model?: string,        // 預設 "text-embedding-3-small"
    baseUrl?: string,      // 自訂端點
    dimensions?: number    // text-embedding-3-small=1536, 3-large=3072
  },
  dbPath?: string,         // 預設 ~/.openclaw/memory/lancedb
  autoCapture: boolean,    // 預設 false
  autoRecall: boolean,     // 預設 true
  captureMaxChars: number  // 100-10000，預設 500
}
```

### 8.4 三個 Tools

**memory_recall**：
```
參數：query (string), limit? (default 5)
流程：embed(query) → search(vector, limit, minScore=0.1)
回傳：格式化記憶清單（含 category + score）
```

**memory_store**：
```
參數：text (string), importance? (default 0.7), category?
流程：embed(text) → 重複偵測 (minScore=0.95) → store
回傳：確認 + ID，或重複警告
```

**memory_forget**：
```
參數：query? | memoryId?
  - by ID：直接刪除（UUID 驗證防注入）
  - by query：搜尋 → score > 0.9 直接刪 / 否則列候選
```

### 8.5 生命週期 Hooks

**before_agent_start**（Auto-Recall）：
```
autoRecall=true 時：
  1. embed(userPrompt)
  2. search(vector, limit=3, minScore=0.3)
  3. 回傳 prependContext：
     <relevant-memories>
       [1] [preference] (0.82) 使用者偏好 dark mode
       ...
     </relevant-memories>
  4. 標記為 "untrusted historical data for context only"
```

**agent_end**（Auto-Capture）：
```
autoCapture=true 時：
  1. 僅處理 user-role 訊息（避免自我中毒）
  2. shouldCapture() 過濾：
     - 長度：10 ≤ chars ≤ captureMaxChars
     - 排除：injected memories / system XML / markdown 摘要 / emoji-heavy
     - 匹配 MEMORY_TRIGGERS regex
     - 拒絕 PROMPT_INJECTION_PATTERNS
  3. detectCategory(text) → 自動分類
  4. 重複偵測 (minScore=0.95)
  5. 每次對話最多存 3 筆
```

### 8.6 安全過濾

**MEMORY_TRIGGERS**（觸發捕獲的 regex）：
- remember/preference 關鍵字
- 聯絡資訊（email、電話）
- 所有權 ("my X is")
- 偏好詞 (like, prefer, hate, love, want)
- 強調詞 (always, never, important)

**PROMPT_INJECTION_PATTERNS**（阻擋注入）：
- "ignore previous instructions"
- XML-like tags (`<system>`, `<tool>`)
- 指令調用模式

**escapeMemoryForPrompt()**：HTML entity 跳脫，防止注入 prompt

---

## 9. Context Engine

### 9.1 架構

```
┌───────────────────────────────────────────┐
│              Context Engine Slot           │
│                                           │
│  Registry: Map<id, ContextEngineFactory>  │
│                                           │
│  ┌─────────────────┐  ┌───────────────┐  │
│  │ "legacy"        │  │ Custom Engine │  │
│  │ (default)       │  │ (plugin slot) │  │
│  └────────┬────────┘  └───────┬───────┘  │
│           │                    │          │
│           └────────┬───────────┘          │
│                    ▼                      │
│           resolveContextEngine()          │
│           config.plugins.slots.           │
│              contextEngine ?? "legacy"    │
└───────────────────────────────────────────┘
```

### 9.2 ContextEngine 介面

```typescript
interface ContextEngine {
  readonly info: ContextEngineInfo;  // id, name, version, ownsCompaction

  // 生命週期
  bootstrap?(params): Promise<BootstrapResult>;
  afterTurn?(params): Promise<void>;
  dispose?(): Promise<void>;

  // 核心操作
  ingest(params: {
    sessionId, message: AgentMessage, isHeartbeat?
  }): Promise<IngestResult>;

  ingestBatch?(params: {
    sessionId, messages: AgentMessage[], isHeartbeat?
  }): Promise<IngestBatchResult>;

  assemble(params: {
    sessionId, messages: AgentMessage[], tokenBudget?
  }): Promise<AssembleResult>;

  compact(params: {
    sessionId, sessionFile, tokenBudget?, force?,
    currentTokenCount?, compactionTarget?, customInstructions?
  }): Promise<CompactResult>;

  // Subagent
  prepareSubagentSpawn?(params): Promise<SubagentSpawnPreparation | undefined>;
  onSubagentEnded?(params): Promise<void>;
}
```

### 9.3 Result Types

| Type | 關鍵欄位 |
|------|---------|
| `AssembleResult` | `messages`, `estimatedTokens`, `systemPromptAddition?` |
| `CompactResult` | `ok`, `compacted`, `reason?`, `result?{summary, firstKeptEntryId, tokensBefore, tokensAfter}` |
| `IngestResult` | `ingested: boolean` |
| `BootstrapResult` | `bootstrapped`, `importedMessages?`, `reason?` |

### 9.4 LegacyContextEngine

預設引擎，pass-through 行為：

| 方法 | 行為 |
|------|------|
| `ingest()` | no-op，`{ ingested: false }`（SessionManager 處理持久化） |
| `assemble()` | pass-through，原樣回傳 messages，`estimatedTokens: 0` |
| `afterTurn()` | no-op |
| `compact()` | 委派 `compactEmbeddedPiSessionDirect()`（pi-agent-core） |
| `dispose()` | no-op |

### 9.5 Context 組裝管線（attempt.ts）

```
RUN.TS
  └─ resolveContextEngine()
      └─ runEmbeddedAttempt({ contextEngine, contextTokenBudget })

ATTEMPT.TS
  ├─ SessionManager.open(sessionFile)
  │
  ├─ 1. SANITIZATION
  │   ├─ sanitizeSessionHistory()           // 修復損壞的 tool call 配對
  │   ├─ validateGeminiTurns()              // Google 專用驗證
  │   ├─ validateAnthropicTurns()           // Anthropic 專用驗證
  │   └─ sanitizeToolUseResultPairing()     // 孤兒偵測
  │
  ├─ 2. HISTORY TRUNCATION
  │   └─ limitHistoryTurns()                // DM history 上限
  │
  ├─ 3. CONTEXT ENGINE ASSEMBLY
  │   └─ contextEngine.assemble({
  │        sessionId,
  │        messages: activeSession.messages,
  │        tokenBudget: params.contextTokenBudget  // 預設 200,000
  │      })
  │      └─ { messages, estimatedTokens, systemPromptAddition? }
  │
  ├─ 4. SYSTEM PROMPT INJECTION
  │   └─ prependSystemPromptAddition()
  │
  ├─ 5. MESSAGE REPLACEMENT
  │   └─ activeSession.agent.replaceMessages(assembled.messages)
  │
  ├─ 6. BUILD & CALL MODEL
  │
  └─ 7. AFTER TURN
      └─ contextEngine.afterTurn({
           sessionId, sessionFile, messages,
           prePromptMessageCount, autoCompactionSummary?,
           tokenBudget?, legacyCompactionParams?
         })
```

### 9.6 Compaction 觸發

| 觸發條件 | 時機 | 備註 |
|----------|------|------|
| **Overflow** | Token/message 超過 context window | 自動，在 attempt 中 |
| **Manual** | 使用者/排程/`/new` 指令 | 顯式呼叫 |
| **Proactive** | `afterTurn()` hook | 背景，非阻塞 |

### 9.7 Compaction Reason 分類

| 原始 reason | 分類 |
|------------|------|
| "nothing to compact" | `no_compactable_entries` |
| "below threshold" | `below_threshold` |
| "already compacted" | `already_compacted_recently` |
| "guard" | `guard_blocked` |
| "summary" | `summary_failed` |
| "timed out" / "timeout" | `timeout` |
| 4xx HTTP | `provider_error_4xx` |
| 5xx HTTP | `provider_error_5xx` |

### 9.8 Subagent 生命週期

```typescript
// Spawn 前準備（可選）
prepareSubagentSpawn({
  parentSessionKey, childSessionKey, ttlMs?
}) → { rollback: () => void }  // spawn 失敗時回滾

// Child 結束通知（可選）
onSubagentEnded({
  childSessionKey,
  reason: "deleted" | "completed" | "swept" | "released"
})
```

---

## 10. 跨系統整合圖

```
User Message
  │
  ├─ [before_agent_start hook]
  │   └─ memory-lancedb: auto-recall → prependContext
  │
  ├─ Context Engine: assemble()
  │   └─ Legacy: pass-through
  │
  ├─ Agent Prompt Loop
  │   ├─ Tool call: memory_search (memory-core)
  │   │   └─ getMemorySearchManager() → MemoryIndexManager → search()
  │   │       └─ SQLite: vector (sqlite-vec) + keyword (FTS5) → hybrid merge
  │   │
  │   ├─ Tool call: memory_recall (memory-lancedb)
  │   │   └─ MemoryDB → LanceDB → L2 search
  │   │
  │   ├─ Tool call: memory_store (memory-lancedb)
  │   │   └─ shouldCapture() → embed() → dedup → store()
  │   │
  │   └─ Tool call: memory_get (memory-core)
  │       └─ readFile() → snippet
  │
  ├─ Context Engine: afterTurn()
  │   └─ 背景：可觸發 compaction / 索引更新
  │
  └─ [agent_end hook]
      └─ memory-lancedb: auto-capture → shouldCapture() → store()

Compaction 觸發：
  Context Engine → compact() → session.compact()
  → AI 摘要 → 截斷舊訊息 → 更新 session
```

### Memory 系統雙軌並行

| 面向 | Builtin (src/memory/) | LanceDB (extension) |
|------|----------------------|---------------------|
| 儲存 | SQLite + sqlite-vec + FTS5 | LanceDB (Arrow-based) |
| 搜尋 | vector + keyword hybrid | L2 distance vector-only |
| 資料來源 | MEMORY.md + memory/*.md + sessions | user 對話捕獲 |
| 觸發 | Tool call (memory_search) | Auto-recall hook + Tool call |
| 寫入 | 自動同步 (file watcher) | Auto-capture hook / Tool call |
| 重排序 | temporal decay + MMR | 無 |
| Embedding | 6 providers | OpenAI (可自訂 endpoint) |

---

## 11. 邊界條件與陷阱

### Memory 子系統

1. **sqlite-vec 載入超時**：`ensureVectorReady()` 有 30s timeout，超時後 vector 標記不可用，fallback 到 in-memory cosine 或 FTS-only
2. **首次 embedding 決定維度**：vector table 的 FLOAT[N] 維度由第一個 provider 的 embedding 決定，後續換 provider 若維度不同需重建
3. **Batch 失敗自動停用**：`batchFailureCount >= BATCH_FAILURE_LIMIT(2)` → 自動關閉 batch mode，fallback 到即時 embedding
4. **Provider auto 不含 Ollama**：auto 模式只嘗試 openai/gemini/voyage/mistral，Ollama 需顯式設定
5. **FTS query 全 AND**：buildFtsQuery 用 AND 連接所有詞，長查詢可能 0 結果（此時 fallback 到 keyword 放寬模式）
6. **Chunk ID 衝突**：ID = `{path}::{hash[0:16]}`，理論上 16 hex = 64 bit 可能碰撞，但實務上機率極低
7. **Session delta read**：增量讀取用 64KB chunks，超大 session 檔首次索引可能慢
8. **readonly DB recovery**：追蹤 readonlyRecoveryAttempts/Successes/Failures，指數退避重試
9. **embedding_cache PK 含 provider_key**：換 API key 會失去快取命中

### LanceDB 子系統

10. **重複偵測閾值 0.95**：太相似才算重複，微小改寫可能存兩份
11. **Auto-capture 每次最多 3 筆**：防止長對話暴寫
12. **PROMPT_INJECTION_PATTERNS**：阻擋含 `<system>` 等 tag 的文字，可能誤擋合法程式碼片段
13. **shouldCapture emoji 計數**：>3 emoji 就排除，可能遺漏含 emoji 的有效偏好
14. **LanceDB native bindings**：macOS 可能缺少原生綁定，lazy load 處理

### Context Engine

15. **Legacy engine estimatedTokens=0**：assemble 回傳 0，實際 token 估算在 caller（attempt.ts）
16. **compaction safety timeout**：compact 有超時保護，超時回傳 `{ ok: false, reason: "timeout" }`
17. **afterTurn 非阻塞**：afterTurn 是 async 但不等結果，compaction 可能在下次 prompt 前未完成
18. **Plugin slot 覆寫**：config.plugins.slots.contextEngine 可覆寫 "legacy"，但自訂引擎需實作完整介面
19. **Subagent context 隔離**：prepareSubagentSpawn 返回 rollback function，spawn 失敗時務必呼叫
20. **Sanitization 順序固定**：先 sanitize → limitHistory → assemble → compact，跳過任一步可能破壞 message 配對

---

## 12. 關鍵常量速查

### Memory — Embedding

| 常量 | 值 | 用途 |
|------|-----|------|
| `EMBEDDING_BATCH_MAX_TOKENS` | 8,000 | 單批次最大 token |
| `EMBEDDING_INDEX_CONCURRENCY` | 4 | 並行 embedding 任務 |
| `EMBEDDING_RETRY_MAX_ATTEMPTS` | 3 | 重試次數 |
| `EMBEDDING_RETRY_BASE_DELAY_MS` | 500 | 退避基底 |
| `EMBEDDING_RETRY_MAX_DELAY_MS` | 8,000 | 退避上限 |
| `EMBEDDING_QUERY_TIMEOUT_REMOTE_MS` | 60,000 | Remote query 超時 |
| `EMBEDDING_QUERY_TIMEOUT_LOCAL_MS` | 300,000 | Local query 超時 |
| `EMBEDDING_BATCH_TIMEOUT_REMOTE_MS` | 120,000 | Remote batch 超時 |
| `EMBEDDING_BATCH_TIMEOUT_LOCAL_MS` | 600,000 | Local batch 超時 |

### Memory — Search & Sync

| 常量 | 值 | 用途 |
|------|-----|------|
| `SNIPPET_MAX_CHARS` | 700 | 搜尋結果片段上限 |
| `VECTOR_TABLE` | "chunks_vec" | 向量表名 |
| `FTS_TABLE` | "chunks_fts" | 全文表名 |
| `SESSION_DIRTY_DEBOUNCE_MS` | 5,000 | Session 異動防抖 |
| `SESSION_DELTA_READ_CHUNK_BYTES` | 65,536 | 增量讀取大小 |
| `VECTOR_LOAD_TIMEOUT_MS` | 30,000 | sqlite-vec 載入超時 |
| `BATCH_FAILURE_LIMIT` | 2 | Batch 失敗自動停用門檻 |

### Memory — LanceDB

| 常量 | 值 | 用途 |
|------|-----|------|
| `minScore` (recall) | 0.1 | 召回最低相似度 |
| `minScore` (auto-recall) | 0.3 | 自動召回較嚴格 |
| `minScore` (dedup) | 0.95 | 重複偵測閾值 |
| `captureMaxChars` | 500 (default) | 捕獲最大字元 |
| `max captures/conv` | 3 | 每次對話捕獲上限 |
| `auto-recall limit` | 3 | 自動召回筆數上限 |

### Context Engine

| 常量 | 值 | 用途 |
|------|-----|------|
| `DEFAULT_CONTEXT_TOKENS` | 200,000 | 預設 context window |
| Default slot ID | "legacy" | Context engine 預設 |
| `candidates` | min(200, maxResults × multiplier) | 候選搜尋結果數 |

---

## 13. C# 概念對照

| OpenClaw (TypeScript) | C# 對應 | 說明 |
|----------------------|---------|------|
| `MemoryIndexManager` singleton cache | `ConcurrentDictionary<K, Lazy<T>>` | 避免重複建立 |
| `DatabaseSync` (node:sqlite) | `Microsoft.Data.Sqlite` | SQLite 同步 API |
| sqlite-vec virtual table | EF Core + pgvector | 向量搜尋擴展 |
| FTS5 `MATCH` + `bm25()` | Lucene.NET / Azure AI Search | 全文檢索 |
| `EmbeddingProvider` interface | `IEmbeddingGenerator<string, Embedding>` | .NET 9 AI Abstraction |
| `chunkMarkdown()` | `TextChunker` (Semantic Kernel) | 文本分塊 |
| cosine similarity in-process | `TensorPrimitives.CosineSimilarity()` | .NET 8 SIMD 加速 |
| LanceDB | Microsoft.SemanticKernel.Connectors.* | 向量 DB connector |
| `ContextEngine` interface | Middleware pipeline (ASP.NET) | 可插拔處理管線 |
| `CompactResult` | `IResult<T>` pattern | 結果封裝 |
| Plugin slot resolution | DI `IServiceProvider.GetService<T>()` | 依注入解析 |
| `FallbackMemoryManager` | Polly `FallbackPolicy` | 降級策略 |
| file watcher (chokidar) | `FileSystemWatcher` | 檔案監視 |
| batch embedding + polling | `BackgroundService` + `IHostedService` | 背景任務 |
| temporal decay half-life | Custom `IScoreModifier` | 分數衰減 |
| MMR Jaccard similarity | `HashSet.IntersectWith()` / `UnionWith()` | 集合運算 |
