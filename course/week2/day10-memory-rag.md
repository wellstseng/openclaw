# Day 10：Memory / RAG 系統

> **目標**：理解 Embedding 產生、向量儲存、混合搜尋、記憶管理
> **C# 對照**：Semantic Kernel Memory、IMemoryStore、Vector DB
> **預計時間**：4 小時

---

## 10.1 RAG 概念（C# 開發者版）

**RAG = Retrieval Augmented Generation（檢索增強生成）**

```
傳統 AI 對話：使用者問 → AI 用訓練資料答（可能過時或缺少私有資訊）

RAG 對話：使用者問 → 搜尋相關文件/記憶 → 把搜尋結果塞進 prompt → AI 用最新資訊答
```

```csharp
// C# 概念（Semantic Kernel）
var memory = new MemoryBuilder()
    .WithOpenAITextEmbeddingGeneration("text-embedding-3-small", apiKey)
    .WithSqliteMemoryStore("memory.db")
    .Build();

// 儲存記憶
await memory.SaveInformationAsync("chat-history",
    "Wells 喜歡用 C# 寫後端", id: "fact-1");

// 搜尋相關記憶
var results = await memory.SearchAsync("chat-history", "Wells 的技術偏好");
// → 找到 "Wells 喜歡用 C# 寫後端"
```

---

## 10.2 embeddings.ts — 向量嵌入統一介面

OpenClaw 支援多個 Embedding Provider：

```typescript
// src/memory/embeddings.ts

// 統一的 Embedding 介面
interface EmbeddingProvider {
  embed(texts: string[]): Promise<number[][]>;
  // 輸入：多段文字
  // 輸出：每段文字對應一個向量 (number[])
}

// 支援的 Provider
type EmbeddingBackend =
  | "openai"       // text-embedding-3-small / text-embedding-3-large
  | "gemini"       // Gemini embedding
  | "voyage"       // Voyage AI
  | "node-llama";  // 本地 LLaMA 模型（離線使用）

async function createEmbedding(
  texts: string[],
  backend: EmbeddingBackend,
  apiKey: string,
): Promise<number[][]> {
  switch (backend) {
    case "openai":
      return embedWithOpenAI(texts, apiKey);
    case "gemini":
      return embedWithGemini(texts, apiKey);
    case "voyage":
      return embedWithVoyage(texts, apiKey);
    case "node-llama":
      return embedWithLocalLlama(texts);
  }
}
```

```csharp
// C# 等價
public interface IEmbeddingProvider
{
    Task<float[][]> EmbedAsync(string[] texts);
}

public class OpenAIEmbeddingProvider : IEmbeddingProvider { ... }
public class GeminiEmbeddingProvider : IEmbeddingProvider { ... }

// 用 DI 注入
services.AddSingleton<IEmbeddingProvider>(provider =>
    config.EmbeddingBackend switch
    {
        "openai" => new OpenAIEmbeddingProvider(apiKey),
        "gemini" => new GeminiEmbeddingProvider(apiKey),
        _ => throw new NotSupportedException(),
    });
```

---

## 10.3 sqlite-vec.ts — 向量資料庫

OpenClaw 用 **SQLite + sqlite-vec 擴充**做本地向量搜尋。

```typescript
// src/memory/sqlite-vec.ts

// 建立向量索引表
async function createVectorTable(db: Database) {
  await db.exec(`
    CREATE VIRTUAL TABLE IF NOT EXISTS vec_memory
    USING vec0(
      embedding float[1536]   -- OpenAI embedding 維度
    )
  `);

  await db.exec(`
    CREATE TABLE IF NOT EXISTS memory_entries (
      id TEXT PRIMARY KEY,
      content TEXT NOT NULL,
      metadata TEXT,           -- JSON metadata
      vec_rowid INTEGER,       -- 指向 vec_memory 的 rowid
      created_at TEXT DEFAULT (datetime('now'))
    )
  `);
}

// 向量搜尋
async function vectorSearch(
  db: Database,
  queryVector: number[],
  limit: number = 10,
): Promise<SearchResult[]> {
  return db.all(`
    SELECT m.id, m.content, m.metadata, v.distance
    FROM vec_memory v
    JOIN memory_entries m ON m.vec_rowid = v.rowid
    WHERE v.embedding MATCH ?
    ORDER BY v.distance
    LIMIT ?
  `, [JSON.stringify(queryVector), limit]);
}
```

```csharp
// C# 等價：EF Core + SQLite + 向量搜尋
// 或用 Microsoft.SemanticKernel.Connectors.Sqlite
public class SqliteVecStore
{
    public async Task<IList<SearchResult>> SearchAsync(
        float[] queryVector, int limit = 10)
    {
        return await _db.QueryAsync<SearchResult>(@"
            SELECT m.Id, m.Content, m.Metadata, v.distance
            FROM vec_memory v
            JOIN memory_entries m ON m.VecRowid = v.rowid
            WHERE v.embedding MATCH @vector
            ORDER BY v.distance
            LIMIT @limit",
            new { vector = JsonSerializer.Serialize(queryVector), limit });
    }
}
```

---

## 10.4 hybrid.ts — 混合搜尋

純向量搜尋有時不夠精確，OpenClaw 用**混合搜尋**（向量 + 關鍵字）提高品質。

```typescript
// src/memory/hybrid.ts

async function hybridSearch(
  query: string,
  options: SearchOptions,
): Promise<SearchResult[]> {
  // 1. 向量搜尋（語意相似度）
  const queryVector = await embed([query]);
  const vectorResults = await vectorSearch(db, queryVector[0], options.limit * 2);

  // 2. 關鍵字搜尋（精確匹配）
  const keywordResults = await keywordSearch(db, query, options.limit * 2);

  // 3. 合併 + 重新排序（Reciprocal Rank Fusion）
  const merged = reciprocalRankFusion(vectorResults, keywordResults);

  return merged.slice(0, options.limit);
}

// RRF 演算法：合併兩個排序結果
function reciprocalRankFusion(
  ...resultSets: SearchResult[][]
): SearchResult[] {
  const scores = new Map<string, number>();

  for (const results of resultSets) {
    for (let i = 0; i < results.length; i++) {
      const id = results[i].id;
      const currentScore = scores.get(id) ?? 0;
      scores.set(id, currentScore + 1 / (60 + i)); // RRF 公式
    }
  }

  return [...scores.entries()]
    .sort((a, b) => b[1] - a[1])
    .map(([id]) => /* 找回完整結果 */);
}
```

---

## 10.5 manager.ts — 記憶管理器

```typescript
// src/memory/manager.ts

class MemoryManager {
  // 索引新的對話到記憶庫
  async indexConversation(sessionKey: string, messages: Message[]) {
    // 1. 把對話切成段落
    const chunks = chunkMessages(messages);

    // 2. Batch embedding（批次產生向量）
    const vectors = await batchEmbed(chunks.map(c => c.text));

    // 3. 存入 SQLite
    for (let i = 0; i < chunks.length; i++) {
      await this.store.insert({
        content: chunks[i].text,
        embedding: vectors[i],
        metadata: { sessionKey, timestamp: chunks[i].timestamp },
      });
    }
  }

  // 搜尋相關記憶
  async search(query: string, limit: number = 5): Promise<MemoryResult[]> {
    return hybridSearch(query, { limit });
  }

  // 增量同步（只索引新的對話）
  async syncIncremental(sessionKey: string) {
    const lastIndexed = await this.getLastIndexedTimestamp(sessionKey);
    const newMessages = await this.getMessagesSince(sessionKey, lastIndexed);
    if (newMessages.length > 0) {
      await this.indexConversation(sessionKey, newMessages);
    }
  }
}
```

---

## 10.6 Memory 在對話中的使用

```typescript
// 在 AI Agent 處理訊息時，會搜尋相關記憶注入 prompt
async function buildContextWithMemory(
  userMessage: string,
  sessionKey: string,
): Promise<string> {
  const memories = await memoryManager.search(userMessage, 5);

  if (memories.length === 0) return "";

  return `Relevant context from previous conversations:\n` +
    memories.map(m => `- ${m.content}`).join("\n");
}
```

---

## 今日閱讀作業

### 作業 1：閱讀 `src/memory/embeddings.ts`
- 找出支援哪些 embedding provider
- 理解 batch embedding 的機制

### 作業 2：閱讀 `src/memory/sqlite-vec.ts`
- 理解向量表的 schema
- 找出向量搜尋的 SQL 查詢

### 作業 3：閱讀 `src/memory/manager.ts`
- 理解增量同步的機制
- 找出 chunk（分段）的策略

### 作業 4：閱讀 `src/memory/hybrid.ts`
- 理解混合搜尋的合併策略

---

## 今日 Checkpoint

1. RAG 的三步驟是什麼？
2. Embedding 的輸入是什麼？輸出是什麼？
3. 為什麼要用混合搜尋而不是純向量搜尋？
4. sqlite-vec 相對於 Pinecone/Qdrant 等雲端向量 DB 的優勢是什麼？
5. 增量同步解決什麼問題？

---

## 答案

1. **Retrieve（檢索）→ Augment（增強 prompt）→ Generate（生成回覆）**
2. 輸入：文字字串。輸出：浮點數向量（如 1536 維的 `number[]`）。語意相近的文字，向量距離也會相近。
3. 向量搜尋擅長**語意相似度**，關鍵字搜尋擅長**精確匹配**。合併兩者結果更全面。
4. **完全本地、零成本、離線可用**。不需要雲服務、不需要網路、資料不出機器。
5. 避免重複索引。只索引上次同步後的新對話，省時省 API 費用（embedding API 要計費）。
