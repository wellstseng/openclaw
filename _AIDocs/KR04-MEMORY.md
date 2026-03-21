# 04-MEMORY — Memory 系統

> 來源：openclaw-knowledge-base.md §7 + F058 §9 + F060 memory modules

---

## 1. LanceDB Schema 完整定義

### MemoryEntry 所有欄位

```typescript
interface MemoryEntry {
  // 主鍵與識別
  id: string;                    // 唯一 ID（UUID v4）
  agentId: string;               // 所屬 Agent ID
  sessionKey?: string;           // 來源 Session Key（可選）

  // 內容
  content: string;               // 記憶文字內容
  summary?: string;              // 摘要（可選，自動生成）

  // 向量
  embedding: number[];           // 向量嵌入（float32 array）

  // 分類與標籤
  kind: MemoryKind;              // 記憶種類（見下）
  tags?: string[];               // 標籤陣列（可選）
  source?: string;               // 來源標識（可選）

  // 時間
  createdAt: number;             // 建立時間（Unix ms）
  updatedAt: number;             // 更新時間（Unix ms）
  accessedAt?: number;           // 最後存取時間（Unix ms，可選）

  // 統計
  accessCount?: number;          // 存取次數（用於 temporal decay）

  // 關聯
  parentId?: string;             // 父記憶 ID（hierarchical memory）
  relatedIds?: string[];         // 相關記憶 IDs

  // 後設資料
  metadata?: Record<string, unknown>; // 任意後設資料
}
```

### MemoryKind enum

```typescript
type MemoryKind =
  | "fact"          // 事實性資訊
  | "instruction"   // 使用者指令
  | "preference"    // 使用者偏好
  | "event"         // 事件記錄
  | "summary"       // 對話摘要
  | "skill"         // 技能描述
  | "entity"        // 實體資訊（人/地/物）
  | "general";      // 一般記憶
```

---

## 2. sentinel row 用途

```typescript
// sentinel row：id = "__schema__"
// 用途：追蹤 LanceDB schema 版本，偵測 schema 不一致
const SENTINEL_ID = "__schema__";

// sentinel row 結構：
{
  id: "__schema__",
  content: JSON.stringify({ schemaVersion: CURRENT_SCHEMA_VERSION }),
  embedding: new Array(EMBEDDING_DIM).fill(0),  // 零向量
  createdAt: INSTALL_TIME,
  // 其他欄位填預設值
}

// 使用方式：
// 1. 服務啟動時讀取 sentinel → 取得 schemaVersion
// 2. 若版本不符 → 觸發 migration
// 3. migration 完成 → 更新 sentinel
```

---

## 3. SQLite FTS5 Schema

### 全文搜尋表

```sql
-- FTS5 虛擬表（用於文字搜尋部分）
CREATE VIRTUAL TABLE memory_fts USING fts5(
  id UNINDEXED,      -- 主鍵（不索引）
  content,           -- 記憶內容（全文索引）
  tags,              -- 標籤（全文索引）
  agentId UNINDEXED, -- Agent ID（不索引）
  tokenize = "porter unicode61"  -- Porter Stemmer + Unicode 支援
);
```

### Embedding Cache 表

```sql
-- embedding cache（避免重複呼叫 embedding API）
CREATE TABLE embedding_cache (
  id TEXT PRIMARY KEY,        -- 輸入文字的 hash
  embedding BLOB,             -- 序列化的 float32 array
  model TEXT,                 -- embedding model 名稱
  createdAt INTEGER,          -- 建立時間（Unix ms）
  accessedAt INTEGER          -- 最後存取時間（Unix ms）
);
```

---

## 4. memory_recall / memory_store / memory_forget 完整參數

### memory_recall（搜尋）

```typescript
interface MemoryRecallParams {
  query: string;               // 搜尋查詢文字
  agentId?: string;            // 限定 Agent（不指定則全域搜尋）
  kind?: MemoryKind | MemoryKind[]; // 限定種類（可選）
  tags?: string[];             // 標籤過濾（可選，AND 邏輯）
  limit?: number;              // 回傳數量上限（預設 10）
  minScore?: number;           // 最低相關性分數（0-1）
  useMmr?: boolean;            // 啟用 MMR 多樣化（預設 false）
  useTemporalDecay?: boolean;  // 啟用時間衰減（預設 false）
  hybridWeight?: {             // 自訂 Hybrid 權重
    vector?: number;           // 向量搜尋權重（預設 0.7）
    text?: number;             // 文字搜尋權重（預設 0.3）
  };
}
```

### memory_store（儲存）

```typescript
interface MemoryStoreParams {
  content: string;             // 記憶內容
  agentId: string;             // 所屬 Agent ID
  kind?: MemoryKind;           // 記憶種類（預設 "general"）
  tags?: string[];             // 標籤（可選）
  sessionKey?: string;         // 來源 Session Key（可選）
  metadata?: Record<string, unknown>; // 後設資料（可選）
  captureMaxChars?: number;    // 最大字元數（預設 4096）
}
```

### memory_forget（刪除）

```typescript
interface MemoryForgetParams {
  id?: string;                 // 精確刪除（by ID）
  query?: string;              // 語意搜尋後刪除（需 confirm=true）
  agentId?: string;            // 限定 Agent
  kind?: MemoryKind;           // 限定種類
  tags?: string[];             // 標籤過濾
  confirm?: boolean;           // 確認刪除（語意搜尋時必須為 true）
  dryRun?: boolean;            // 模擬執行（不實際刪除）
}
```

---

## 5. Hybrid Search 權重與算法

### Score Fusion 公式

```typescript
// Hybrid Search 最終分數計算
function computeHybridScore(
  vectorScore: number,     // LanceDB cosine similarity（0-1）
  textScore: number,       // BM25 文字相關性分數
  weights = { vector: 0.7, text: 0.3 }
): number {
  // 1. 候選集生成：vector search top (limit * candidateMultiplier=4)
  //    + BM25 FTS5 search top (limit * candidateMultiplier=4)
  // 2. 合併候選集（dedup by id）
  // 3. 對每個候選計算混合分數
  const normalizedTextScore = bm25RankToScore(textScore);
  return weights.vector * vectorScore + weights.text * normalizedTextScore;
}

// BM25 rank → 0-1 正規化
function bm25RankToScore(rank: number): number {
  // FTS5 rank 是負數（越負 = 越相關）
  // 轉換為 0-1（越高 = 越相關）
  return 1 / (1 + Math.abs(rank));
}
```

### 關鍵常數

| 常數 | 值 |
|------|----|
| vector 權重 | 0.7 |
| text 權重 | 0.3 |
| `candidateMultiplier` | 4（候選集倍數） |

---

## 6. MMR 算法

### Maximal Marginal Relevance

```typescript
// MMR：在相關性與多樣性之間取得平衡
// lambda = 0.7（相關性比重）
// 1 - lambda = 0.3（多樣性比重）

function mmrRerank(
  query: number[],        // query 向量
  candidates: MemoryEntry[], // 候選記憶
  lambda = 0.7,
  k: number              // 最終回傳數量
): MemoryEntry[] {
  const selected: MemoryEntry[] = [];
  const remaining = [...candidates];

  while (selected.length < k && remaining.length > 0) {
    // 對每個候選計算 MMR 分數：
    // score = lambda * sim(query, candidate)
    //       - (1 - lambda) * max(sim(candidate, selected_i))
    // 選擇 MMR score 最高的候選
    const best = argmax(remaining, (c) => {
      const relevance = cosineSim(query, c.embedding);
      const redundancy = selected.length > 0
        ? Math.max(...selected.map(s => jaccardSim(c, s)))
        : 0;
      return lambda * relevance - (1 - lambda) * redundancy;
    });
    selected.push(best);
    remaining.splice(remaining.indexOf(best), 1);
  }

  return selected;
}

// 相似度度量：使用 Jaccard similarity（基於 tags/content tokens）
```

### MMR 配置

```typescript
// 預設：disabled（useMmr = false）
// 啟用：memory_recall 時傳入 useMmr: true
// 或在 Agent 配置中設定 memory.useMMR = true
```

---

## 7. Temporal Decay 公式

### 數學公式

```typescript
// 指數衰減（Half-life decay）
function applyTemporalDecay(
  score: number,
  entry: MemoryEntry,
  halfLifeDays: number
): number {
  const now = Date.now();
  const ageInDays = (now - entry.createdAt) / (1000 * 60 * 60 * 24);
  const decayFactor = Math.exp(-Math.LN2 / halfLifeDays * ageInDays);
  // Math.LN2 = ln(2) ≈ 0.693
  return score * decayFactor;
}
// 範例：halfLifeDays=7 時，7天前的記憶分數 × 0.5，14天前 × 0.25
```

### Evergreen Paths（不受衰減影響）

```typescript
// 以下種類的記憶免受時間衰減：
const EVERGREEN_KINDS: MemoryKind[] = [
  "instruction",  // 使用者指令永遠有效
  "preference",   // 使用者偏好不衰減
];

// 以下標籤的記憶免受衰減：
const EVERGREEN_TAGS = ["evergreen", "permanent", "core"];
```

### 預設狀態

```typescript
// Temporal decay 預設：disabled
// 啟用方式：
// 1. memory_recall 時傳入 useTemporalDecay: true
// 2. Agent 配置中設定 memory.useTemporalDecay = true + memory.halfLifeDays
```

---

## 8. QMD Query Parser 完整邏輯

### QMD（Query Memory Database）概述

```typescript
// QMD 是記憶查詢協定，允許 Agent 以結構化方式查詢記憶
// 使用 JSON array 格式表達查詢意圖
```

### extractFirstJsonArray 狀態機

```typescript
// 從 LLM 輸出中提取第一個完整的 JSON array
function extractFirstJsonArray(text: string): unknown[] | null {
  // 狀態機：
  // INITIAL → 找到 '[' → ARRAY_OPEN
  // ARRAY_OPEN → 找到完整 JSON → COMPLETE
  // 中途遇到 string/number/nested array/object → 正確跳過
  // 遇到 EOF 或無效字元 → FAILED
}
```

### isQmdNoResultsLine regex

```typescript
// 識別 QMD「無結果」回應
const QMD_NO_RESULTS_RE = /^(no\s+results?|nothing\s+found|empty|null|\[\])\s*$/i;
```

### QMD 所有常數

| 常數 | 值 | 說明 |
|------|----|------|
| `QMD_POLL_INTERVAL_MS` | - | 輪詢間隔 |
| `QMD_DEBOUNCE_MS` | - | 防抖延遲 |
| `QMD_TIMEOUT_MS` | - | 整體超時 |
| `QMD_COMMAND_TIMEOUT_MS` | - | 單一命令超時 |
| `QMD_UPDATE_TIMEOUT_MS` | - | 更新操作超時 |
| `QMD_EMBED_TIMEOUT_MS` | - | Embedding 生成超時 |

---

## 9. memory-core vs memory-lancedb 差異

| 項目 | `memory-core` | `memory-lancedb` |
|------|-------------|-----------------|
| 後端 | 抽象介面（可換後端） | LanceDB + SQLite FTS5 |
| 向量搜尋 | 依後端實作 | ✓（LanceDB） |
| 全文搜尋 | 依後端實作 | ✓（SQLite FTS5） |
| Hybrid Search | ✓（介面層） | ✓（實作層） |
| 持久化 | 依後端實作 | ✓（本地檔案） |
| 用途 | 抽象介面/工具函式 | 正式後端實作 |

---

## 10. Auto-Capture / Auto-Recall Lifecycle Hooks

### Auto-Capture（自動儲存）

```typescript
// 觸發時機：Agent 完成一輪對話後
// 觸發條件：
// 1. autoCaptureEnabled = true（Agent 配置）
// 2. 對話輪次包含有意義的資訊（LLM 判斷）
// 3. 內容長度 ≤ captureMaxChars（預設 4096）

// Hook 名稱：
"memory:before_capture"  // 捕獲前（可修改或取消）
"memory:after_capture"   // 捕獲後（記錄結果）
```

### Auto-Recall（自動召回）

```typescript
// 觸發時機：Agent 收到新訊息時
// 觸發條件：
// 1. autoRecallEnabled = true（Agent 配置）
// 2. 有足夠的查詢上下文

// Hook 名稱：
"memory:before_recall"   // 召回前（可修改查詢）
"memory:after_recall"    // 召回後（可過濾結果）
```

### captureMaxChars 預設值

```typescript
const CAPTURE_MAX_CHARS = 4096;  // 預設最大捕獲字元數
```

---

## 11. LanceDB 設定型別

```typescript
interface LanceDbConfig {
  uri: string;                    // 資料庫路徑（本地目錄）
  embeddingModel?: string;        // Embedding model（預設 text-embedding-3-small）
  embeddingDim?: number;          // 向量維度（依 model 決定）
  indexType?: "ivf_pq" | "flat";  // 索引類型（預設 ivf_pq 用於大量資料）
  numPartitions?: number;         // IVF 分區數
  numSubVectors?: number;         // PQ 子向量數
  cacheSize?: number;             // 記憶體 cache 大小
}
```
