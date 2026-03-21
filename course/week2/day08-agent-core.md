# Day 8：AI Agent 引擎（上）— 核心 Runner

> **目標**：理解 pi-embedded-runner 的完整流程、System Prompt 組裝、模型選擇與 Auth Profile
> **C# 對照**：Semantic Kernel、IChatCompletionService、IConfiguration
> **預計時間**：6 小時（本課程最硬的一天，慢慢來）

---

## 8.1 Agent 架構全景

```
使用者訊息進來
    ↓
[pi-embedded-runner.ts]        ← 指揮官：組裝一切、呼叫 AI、管理生命週期
  ├── [system-prompt.ts]       ← 組裝 system prompt
  ├── [model-catalog.ts]       ← 可用模型清單
  ├── [model-selection.ts]     ← 選擇最佳模型
  ├── [auth-profiles.ts]       ← API Key 輪替與 failover
  ├── [pi-embedded-subscribe.ts] ← 串流訂閱（Day 9）
  ├── [pi-tools.ts]            ← AI Tool 定義（Day 9）
  └── [compaction.ts]          ← 上下文壓縮
    ↓
AI 模型回覆
```

```csharp
// C# 等價架構
public class AgentRunner  // = pi-embedded-runner.ts
{
    private readonly SystemPromptBuilder _promptBuilder;
    private readonly ModelCatalog _modelCatalog;
    private readonly ModelSelector _modelSelector;
    private readonly AuthProfileManager _authManager;
    private readonly StreamSubscriber _subscriber;
    private readonly ToolRegistry _tools;
    private readonly ContextCompactor _compactor;
}
```

---

## 8.2 pi-embedded-runner.ts — AI 呼叫的指揮官

這是整個 AI 引擎的**核心中核心**。

```typescript
// 概念化的 Runner 流程
async function runEmbeddedPiAgent(params: RunParams) {
  // 1. 解析 session → 取得歷史訊息
  const history = await loadSessionMessages(params.sessionKey);

  // 2. 組裝 system prompt
  const systemPrompt = buildSystemPrompt({
    identity: params.identity,       // AI 的人設
    skills: params.skills,           // 可用技能
    channelContext: params.channel,   // 哪個 channel 來的
    tools: params.tools,             // 可用工具
  });

  // 3. 選擇模型
  const model = selectModel({
    preferred: params.model,
    available: await scanAvailableModels(),
    fallbacks: params.fallbacks,
  });

  // 4. 選擇 Auth Profile（API Key）
  const authProfile = await resolveAuthProfile({
    model,
    profiles: params.authProfiles,
    cooldowns: getCooldownState(),
  });

  // 5. 檢查上下文長度，必要時壓縮
  const messages = await maybeCompact(history, systemPrompt);

  // 6. 呼叫 AI 模型（串流）
  const stream = callModel({
    model,
    auth: authProfile,
    messages: [
      { role: "system", content: systemPrompt },
      ...messages,
    ],
    tools: toolDefinitions,
  });

  // 7. 訂閱串流事件
  return subscribeToStream(stream, {
    onText: (chunk) => { /* 文字回覆 */ },
    onToolCall: (call) => { /* 工具呼叫 */ },
    onError: (err) => { /* 錯誤處理 + failover */ },
  });
}
```

```csharp
// C# Semantic Kernel 等價
public class AgentRunner
{
    public async IAsyncEnumerable<AgentEvent> RunAsync(
        RunParams p,
        [EnumeratorCancellation] CancellationToken ct)
    {
        // 1. 載入歷史
        var history = await _sessions.LoadAsync(p.SessionKey);

        // 2. 組裝 prompt
        var systemPrompt = _promptBuilder.Build(p);

        // 3-4. 選模型 + 選 API Key
        var (model, apiKey) = _authManager.Resolve(p.Model);

        // 5. 壓縮
        var messages = await _compactor.MaybeCompactAsync(history, systemPrompt);

        // 6-7. 呼叫 + 串流
        var chatHistory = new ChatHistory(systemPrompt);
        chatHistory.AddRange(messages);

        await foreach (var chunk in _kernel.InvokeStreamingAsync(chatHistory, ct: ct))
        {
            yield return MapEvent(chunk);
        }
    }
}
```

---

## 8.3 system-prompt.ts — System Prompt 組裝

System Prompt 是 AI 行為的「設計圖」。OpenClaw 的 prompt 組裝非常精巧。

```typescript
// 概念化的 System Prompt 組裝
function buildSystemPrompt(params: PromptParams): string {
  const sections: string[] = [];

  // 1. 核心 Identity
  sections.push(`You are ${params.identity.name}.`);
  sections.push(params.identity.systemPrompt ?? "You are a helpful assistant.");

  // 2. Channel 特定指引
  if (params.channel === "whatsapp") {
    sections.push("Keep responses concise. WhatsApp users prefer short messages.");
  }

  // 3. 可用工具描述
  if (params.tools.length > 0) {
    sections.push("You have access to the following tools:");
    for (const tool of params.tools) {
      sections.push(`- ${tool.name}: ${tool.description}`);
    }
  }

  // 4. Skills 注入
  for (const skill of params.skills) {
    sections.push(skill.prompt);
  }

  // 5. 日期時間
  sections.push(`Current date and time: ${new Date().toISOString()}`);

  return sections.join("\n\n");
}
```

```csharp
// C# 等價
public class SystemPromptBuilder
{
    public string Build(PromptParams p)
    {
        var sb = new StringBuilder();

        sb.AppendLine($"You are {p.Identity.Name}.");
        sb.AppendLine(p.Identity.SystemPrompt ?? "You are a helpful assistant.");

        if (p.Channel == "whatsapp")
            sb.AppendLine("Keep responses concise.");

        foreach (var tool in p.Tools)
            sb.AppendLine($"- {tool.Name}: {tool.Description}");

        foreach (var skill in p.Skills)
            sb.AppendLine(skill.Prompt);

        sb.AppendLine($"Current date and time: {DateTime.UtcNow:O}");

        return sb.ToString();
    }
}
```

---

## 8.4 model-catalog.ts + model-selection.ts — 模型管理

### 模型掃描

```typescript
// model-catalog.ts：掃描所有可用模型
async function scanModels(config: Config): Promise<ModelEntry[]> {
  const models: ModelEntry[] = [];

  // 從設定檔讀取
  for (const [provider, providerConfig] of Object.entries(config.models ?? {})) {
    for (const model of providerConfig.models ?? []) {
      models.push({
        id: model.id,
        provider,               // "openai" | "anthropic" | "google" | "ollama" | ...
        contextWindow: model.contextWindow,
        capabilities: model.capabilities,
      });
    }
  }

  // 掃描 Ollama 本地模型
  const ollamaModels = await probeOllamaModels();
  models.push(...ollamaModels);

  return models;
}
```

### 模型選擇

```typescript
// model-selection.ts：根據需求選最佳模型
function selectModel(params: {
  preferred?: string;
  available: ModelEntry[];
  fallbacks?: string[];
}): ModelEntry {
  // 1. 使用者指定了模型 → 用它
  if (params.preferred) {
    const found = params.available.find(m => m.id === params.preferred);
    if (found) return found;
  }

  // 2. 嘗試 fallback 清單
  for (const fallback of params.fallbacks ?? []) {
    const found = params.available.find(m => m.id === fallback);
    if (found) return found;
  }

  // 3. 選第一個可用的
  return params.available[0];
}
```

---

## 8.5 auth-profiles.ts — API Key 輪替與 Failover

這是 OpenClaw 的一個亮點設計：支援**多個 API Key 輪替和自動 failover**。

```typescript
// auth-profiles.ts
interface AuthProfile {
  id: string;
  provider: string;           // "openai" | "anthropic" | ...
  apiKey: string;
  lastUsed?: Date;
  lastGood?: Date;
  failureCount: number;
  cooldownUntil?: Date;       // 失敗後冷卻期
}

// 選擇最佳 Auth Profile
function resolveAuthProfile(profiles: AuthProfile[]): AuthProfile {
  // 1. 排除在冷卻期的
  const available = profiles.filter(p => !isInCooldown(p));

  // 2. 優先選上次成功的
  const lastGood = available.find(p => p.lastGood);
  if (lastGood) return lastGood;

  // 3. Round-robin 輪替
  available.sort((a, b) =>
    (a.lastUsed?.getTime() ?? 0) - (b.lastUsed?.getTime() ?? 0)
  );
  return available[0];
}

// 標記失敗（觸發 cooldown）
function markFailure(profile: AuthProfile) {
  profile.failureCount += 1;
  profile.cooldownUntil = new Date(
    Date.now() + calculateCooldown(profile.failureCount)
  );
}
```

```csharp
// C# 等價
public class AuthProfileManager
{
    public AuthProfile Resolve(IReadOnlyList<AuthProfile> profiles)
    {
        var available = profiles.Where(p => !IsInCooldown(p)).ToList();

        // 優先選上次成功的
        var lastGood = available.FirstOrDefault(p => p.LastGood.HasValue);
        if (lastGood != null) return lastGood;

        // Round-robin
        return available.OrderBy(p => p.LastUsed ?? DateTime.MinValue).First();
    }

    public void MarkFailure(AuthProfile profile)
    {
        profile.FailureCount++;
        profile.CooldownUntil = DateTime.UtcNow
            + CalculateCooldown(profile.FailureCount);
    }
}
```

---

## 8.6 compaction.ts — 上下文壓縮

當對話太長超過模型的 context window 時，自動壓縮。

```typescript
// compaction.ts
async function maybeCompact(
  messages: Message[],
  systemPrompt: string,
  contextLimit: number,
): Promise<Message[]> {
  const totalTokens = estimateTokens(systemPrompt)
    + messages.reduce((sum, m) => sum + estimateTokens(m.content), 0);

  if (totalTokens <= contextLimit) {
    return messages; // 不需要壓縮
  }

  // 壓縮策略：用 AI 摘要前面的對話
  const [oldMessages, recentMessages] = splitAt(messages, messages.length / 2);
  const summary = await summarize(oldMessages);

  return [
    { role: "system", content: `Previous conversation summary: ${summary}` },
    ...recentMessages,
  ];
}
```

```csharp
// C# 等價
public async Task<IList<Message>> MaybeCompactAsync(
    IList<Message> messages, string systemPrompt, int contextLimit)
{
    var totalTokens = EstimateTokens(systemPrompt)
        + messages.Sum(m => EstimateTokens(m.Content));

    if (totalTokens <= contextLimit) return messages;

    var (old, recent) = SplitAt(messages, messages.Count / 2);
    var summary = await SummarizeAsync(old);

    return [new SystemMessage($"Previous summary: {summary}"), ..recent];
}
```

---

## 今日閱讀作業

### 作業 1：閱讀 `src/agents/pi-embedded-runner.ts`
- 這是最長最複雜的檔案之一，不需要逐行看
- 重點追蹤 `runEmbeddedPiAgent` 或類似的主函式
- 找出它在哪裡呼叫 AI 模型

### 作業 2：閱讀 `src/agents/system-prompt.ts`
- 理解 system prompt 是怎麼組裝的
- 找出 identity、skills、tools 是在哪裡被注入的

### 作業 3：閱讀 `src/agents/auth-profiles.ts`
- 理解 failover + cooldown 的機制
- 對照你在 C# 裡做過的 retry/circuit-breaker pattern

---

## 今日 Checkpoint

1. pi-embedded-runner 的主要職責是什麼？用一句話講。
2. System Prompt 包含哪些組成部分？
3. Auth Profile 的 cooldown 機制解決什麼問題？
4. 上下文壓縮（compaction）在什麼時候觸發？
5. 模型選擇的優先順序是什麼？

---

## 答案

1. **組裝 prompt + 選模型 + 選 API Key + 呼叫 AI + 管理串流回覆**。它是 AI 對話的「指揮官」。
2. Identity（人設）+ Channel 指引 + 工具描述 + Skills + 日期時間。
3. 防止同一個 API Key 被反覆重試（避免帳號被 rate limit 或封鎖）。失敗後冷卻，自動切換到其他 Key。
4. 當歷史對話的 token 數超過模型的 context window 限制時。
5. 使用者指定 → fallback 清單 → 第一個可用模型。
