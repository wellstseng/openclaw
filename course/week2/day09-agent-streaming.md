# Day 9：AI Agent 引擎（下）— Streaming + Tool Calling + Sandbox

> **目標**：理解 AI 回覆的串流處理、Tool 定義與執行、Docker Sandbox 隔離
> **C# 對照**：IAsyncEnumerable<T>、OpenAI Function Calling、Docker SDK
> **預計時間**：5 小時

---

## 9.1 pi-embedded-subscribe.ts — 串流訂閱

AI 模型的回覆是**一段一段**流回來的（streaming），OpenClaw 需要：
1. 即時顯示（使用者看到「正在打字」）
2. 分塊送出（WhatsApp 4000 字限制）
3. 偵測 Tool Call（AI 要求呼叫工具）

```typescript
// 概念化的串流訂閱
async function* subscribeEmbeddedPiSession(
  stream: AsyncIterable<StreamEvent>,
  options: SubscribeOptions,
): AsyncGenerator<OutputEvent> {

  let currentText = "";
  let pendingToolCalls: ToolCall[] = [];

  for await (const event of stream) {
    switch (event.type) {
      case "text_delta":
        // AI 回了一小段文字
        currentText += event.text;

        // 如果累積夠多，先送一塊出去（block streaming）
        if (shouldFlush(currentText, options.chunkLimit)) {
          yield { type: "text_chunk", text: currentText };
          currentText = "";
        }
        break;

      case "tool_call":
        // AI 要求呼叫工具
        pendingToolCalls.push(event.toolCall);
        break;

      case "tool_call_complete":
        // 執行工具並把結果餵回 AI
        const result = await executeTool(event.toolCall);
        yield { type: "tool_result", result };
        break;

      case "message_end":
        // 訊息結束，送出剩餘文字
        if (currentText) {
          yield { type: "text_chunk", text: currentText };
        }
        yield { type: "done" };
        break;
    }
  }
}
```

```csharp
// C# 等價：IAsyncEnumerable<T>
public async IAsyncEnumerable<OutputEvent> SubscribeAsync(
    IAsyncEnumerable<StreamEvent> stream,
    SubscribeOptions options,
    [EnumeratorCancellation] CancellationToken ct)
{
    var currentText = new StringBuilder();

    await foreach (var evt in stream.WithCancellation(ct))
    {
        switch (evt)
        {
            case TextDelta td:
                currentText.Append(td.Text);
                if (ShouldFlush(currentText, options.ChunkLimit))
                {
                    yield return new TextChunk(currentText.ToString());
                    currentText.Clear();
                }
                break;

            case ToolCallComplete tc:
                var result = await ExecuteToolAsync(tc.ToolCall);
                yield return new ToolResult(result);
                break;

            case MessageEnd:
                if (currentText.Length > 0)
                    yield return new TextChunk(currentText.ToString());
                yield return new Done();
                break;
        }
    }
}
```

### AsyncGenerator（TS）vs IAsyncEnumerable（C#）

```typescript
// TS：async function* 就是 C# 的 async IAsyncEnumerable 方法
async function* generateNumbers(): AsyncGenerator<number> {
  yield 1;
  await delay(100);
  yield 2;
  await delay(100);
  yield 3;
}

// 消費
for await (const num of generateNumbers()) {
  console.log(num); // 1, 2, 3
}
```

```csharp
// C# 完全等價
async IAsyncEnumerable<int> GenerateNumbersAsync(
    [EnumeratorCancellation] CancellationToken ct)
{
    yield return 1;
    await Task.Delay(100, ct);
    yield return 2;
    await Task.Delay(100, ct);
    yield return 3;
}

await foreach (var num in GenerateNumbersAsync())
{
    Console.WriteLine(num);
}
```

**口訣**：`async function*` + `yield` = C# 的 `async IAsyncEnumerable` + `yield return`

---

## 9.2 pi-tools.ts — AI Tool 定義

OpenAI Function Calling / Tool Use 讓 AI 能呼叫外部工具。

```typescript
// pi-tools.ts：定義 AI 可以使用的工具
const tools: ToolDefinition[] = [
  {
    type: "function",
    function: {
      name: "execute_command",
      description: "Execute a shell command in the workspace",
      parameters: {
        type: "object",
        properties: {
          command: { type: "string", description: "The command to execute" },
          workdir: { type: "string", description: "Working directory" },
        },
        required: ["command"],
      },
    },
  },
  {
    type: "function",
    function: {
      name: "read_file",
      description: "Read the contents of a file",
      parameters: {
        type: "object",
        properties: {
          path: { type: "string", description: "File path" },
        },
        required: ["path"],
      },
    },
  },
  {
    type: "function",
    function: {
      name: "send_message",
      description: "Send a message to a channel",
      parameters: {
        type: "object",
        properties: {
          channel: { type: "string" },
          target: { type: "string" },
          text: { type: "string" },
        },
        required: ["channel", "target", "text"],
      },
    },
  },
];
```

```csharp
// C# Semantic Kernel 等價
public class WorkspacePlugin
{
    [KernelFunction("execute_command")]
    [Description("Execute a shell command in the workspace")]
    public async Task<string> ExecuteCommandAsync(
        [Description("The command to execute")] string command,
        [Description("Working directory")] string? workdir = null)
    {
        // ...
    }

    [KernelFunction("read_file")]
    [Description("Read the contents of a file")]
    public async Task<string> ReadFileAsync(
        [Description("File path")] string path)
    {
        // ...
    }
}
```

---

## 9.3 bash-tools.ts — Shell 指令執行

```typescript
// bash-tools.ts：讓 AI 執行 shell 指令
import { spawn } from "node:child_process";

async function executeCommand(
  command: string,
  options: ExecOptions,
): Promise<ExecResult> {
  return new Promise((resolve, reject) => {
    const child = spawn("bash", ["-c", command], {
      cwd: options.workdir,
      timeout: options.timeout ?? 30_000,  // 30 秒超時
      env: { ...process.env, ...options.env },
    });

    let stdout = "";
    let stderr = "";

    child.stdout.on("data", (data) => { stdout += data; });
    child.stderr.on("data", (data) => { stderr += data; });

    child.on("close", (code) => {
      resolve({ stdout, stderr, exitCode: code ?? 1 });
    });
  });
}
```

```csharp
// C# 等價
public async Task<ExecResult> ExecuteCommandAsync(
    string command, ExecOptions options)
{
    using var process = new Process
    {
        StartInfo = new ProcessStartInfo
        {
            FileName = "bash",
            Arguments = $"-c \"{command}\"",
            WorkingDirectory = options.Workdir,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
        }
    };

    process.Start();

    var stdout = await process.StandardOutput.ReadToEndAsync();
    var stderr = await process.StandardError.ReadToEndAsync();

    await process.WaitForExitAsync();

    return new ExecResult(stdout, stderr, process.ExitCode);
}
```

---

## 9.4 sandbox.ts — Docker Sandbox 隔離

AI 執行的指令可能有危險，所以 OpenClaw 支援把指令放到 Docker 容器裡執行。

```typescript
// sandbox.ts：在 Docker 容器裡執行指令
async function executeInSandbox(
  command: string,
  sandboxConfig: SandboxConfig,
): Promise<ExecResult> {
  // docker run --rm -v workspace:/workspace image bash -c "command"
  const args = [
    "run", "--rm",
    "-v", `${sandboxConfig.workspaceDir}:/workspace`,
    "--workdir", "/workspace",
    "--network", sandboxConfig.network ?? "none",  // 預設無網路
    "--memory", sandboxConfig.memoryLimit ?? "512m",
    sandboxConfig.image,
    "bash", "-c", command,
  ];

  return executeCommand("docker", args);
}
```

```csharp
// C# 等價（用 Docker.DotNet NuGet）
public async Task<ExecResult> ExecuteInSandboxAsync(
    string command, SandboxConfig config)
{
    using var client = new DockerClientConfiguration().CreateClient();

    var container = await client.Containers.CreateContainerAsync(
        new CreateContainerParameters
        {
            Image = config.Image,
            Cmd = new[] { "bash", "-c", command },
            HostConfig = new HostConfig
            {
                Binds = [$"{config.WorkspaceDir}:/workspace"],
                NetworkMode = config.Network ?? "none",
                Memory = ParseMemoryLimit(config.MemoryLimit ?? "512m"),
            },
            WorkingDir = "/workspace",
        });

    await client.Containers.StartContainerAsync(container.ID, null);
    // ... 收集 stdout/stderr
}
```

---

## 9.5 Tool Call 的完整生命週期

```
AI 模型回覆
  │
  ├── 純文字 → 直接串流送出
  │
  └── Tool Call → 暫停文字輸出
        │
        ├── execute_command → bash-tools.ts → sandbox.ts
        │     └── 結果回傳給 AI
        │
        ├── read_file → 讀檔案內容
        │     └── 結果回傳給 AI
        │
        └── send_message → channel-tools.ts → 送訊息到指定 channel
              └── 結果回傳給 AI
        │
        ↓
  AI 繼續生成回覆（可能再呼叫其他工具）
        │
        ↓
  最終文字回覆 → 送給使用者
```

```csharp
// C# 等價概念：Semantic Kernel 的 Auto Function Calling
var settings = new PromptExecutionSettings
{
    FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
    // SK 會自動處理：
    // 1. AI 回覆 tool_call → 自動執行對應的 KernelFunction
    // 2. 把結果餵回 AI
    // 3. AI 繼續生成，直到不再呼叫工具
};
```

---

## 今日閱讀作業

### 作業 1：閱讀 `src/agents/pi-embedded-subscribe.ts`
- 找出串流事件的處理邏輯
- 理解 block streaming（分塊串流）的機制

### 作業 2：閱讀 `src/agents/pi-tools.ts`
- 列出所有 AI 可用的工具
- 找出工具定義的 JSON Schema 格式

### 作業 3：閱讀 `src/agents/bash-tools.ts`
- 理解指令執行的安全措施（timeout、sandbox）
- 找出 PTY（偽終端）的使用場景

### 作業 4：閱讀 `src/agents/sandbox.ts`
- 理解 Docker sandbox 的設定選項
- 找出哪些情況下會使用 sandbox

---

## 今日 Checkpoint

1. `async function*` 在 C# 裡等於什麼？
2. AI Tool Call 的結果要怎麼處理？
3. Sandbox 的 `--network none` 是在做什麼？
4. block streaming 解決什麼問題？
5. 一次 AI 對話可以呼叫多次 Tool 嗎？

---

## 答案

1. `async IAsyncEnumerable<T>` + `yield return`
2. 執行工具 → 取得結果 → 以 `tool_result` 角色回傳給 AI → AI 繼續生成。
3. **斷網**。防止 AI 執行的指令存取外部網路（安全隔離）。
4. 對話太長時不能一次送完（如 WhatsApp 4000 字限制），所以分塊即時送出。
5. **可以**。AI 可以連續呼叫多個工具，每個工具結果都回傳，直到 AI 決定停止。
