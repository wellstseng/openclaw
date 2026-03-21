# Day 4：程式進入點 — 從 Main() 開始

> **目標**：追蹤 OpenClaw 從啟動到 Gateway 就緒的完整流程
> **C# 對照**：Program.cs → Startup.cs → Host.Run()
> **預計時間**：4 小時

---

## 4.1 啟動流程全景圖

```
openclaw.mjs (npm bin 入口)
  → src/entry.ts (真正的 Main)
    → normalizeWindowsArgv()          // Windows 參數修正
    → ensureExperimentalWarningSuppressed()  // Node.js flag 處理
    → parseCliProfileArgs()           // 解析 --profile dev
    → import("./cli/run-main.js")     // 動態載入 CLI
      → runCli(process.argv)
        → import("../index.js")       // 載入主模組
          → loadDotEnv()              // 載入 .env
          → normalizeEnv()            // 環境變數正規化
          → enableConsoleCapture()    // 結構化日誌
          → assertSupportedRuntime()  // 檢查 Node.js 版本
          → buildProgram()            // 建立 CLI 指令樹
        → program.parseAsync()        // 解析指令並執行
          → "gateway" command
            → 啟動 Gateway Server
```

```csharp
// C# 等價概念
static async Task Main(string[] args)        // entry.ts
{
    NormalizeArgv(args);                      // normalizeWindowsArgv
    LoadDotEnv();                             // loadDotEnv
    NormalizeEnv();                           // normalizeEnv

    var host = Host.CreateDefaultBuilder()    // buildProgram
        .ConfigureServices(ConfigureServices)
        .Build();

    await host.RunAsync();                    // program.parseAsync
}
```

---

## 4.2 entry.ts 深入解讀

### 重點 1：Windows 參數修正

```typescript
// src/entry.ts:79-144
function normalizeWindowsArgv(argv: string[]): string[] {
  if (process.platform !== "win32") return argv;
  // Windows 的 argv 有時會包含額外的 node.exe 路徑
  // 這個函式負責清理它
  // ...
}
```

```csharp
// C# 對照：你很少需要處理這個問題
// 因為 dotnet run 的 args 已經是乾淨的
// 但如果你有用過 Process.Start() 傳參數，就知道 Windows 引號處理很麻煩
```

### 重點 2：Respawn 機制

```typescript
// src/entry.ts:34-77
function ensureExperimentalWarningSuppressed(): boolean {
  // Node.js 會印出 ExperimentalWarning，很煩
  // 解法：用正確的 flag 重新啟動自己
  const child = spawn(
    process.execPath,  // node.exe 路徑
    [EXPERIMENTAL_WARNING_FLAG, ...process.argv.slice(1)],
    { stdio: "inherit" }
  );
  return true; // 表示「我已經 respawn 了，父進程請退出」
}
```

```csharp
// C# 對照：類似用 Process.Start() 重新啟動自己並加上新參數
var psi = new ProcessStartInfo
{
    FileName = Environment.ProcessPath,
    Arguments = $"--suppress-warning {string.Join(" ", args)}",
    UseShellExecute = false,
};
Process.Start(psi);
```

### 重點 3：動態 import

```typescript
// src/entry.ts:162-170
import("./cli/run-main.js")
  .then(({ runCli }) => runCli(process.argv))
  .catch((error) => { ... });
```

**這是 TS 的動態 import**——在執行時才載入模組。

```csharp
// C# 等價：類似 Assembly.LoadFrom() + 反射
var assembly = Assembly.LoadFrom("OpenClaw.Cli.dll");
var runCli = assembly.GetType("RunMain")!.GetMethod("RunCli")!;
await (Task)runCli.Invoke(null, new object[] { args })!;

// 或更現代的：用 DI 的 Lazy<T>
```

---

## 4.3 index.ts 深入解讀

```typescript
// src/index.ts — 這是模組的「公開 API」

// 第 1 步：載入環境
loadDotEnv({ quiet: true });     // 讀 .env 檔案 → C# 的 ConfigurationBuilder.AddEnvironmentVariables()
normalizeEnv();                   // 統一環境變數格式
ensureOpenClawCliOnPath();        // 確保 CLI 在 PATH 裡

// 第 2 步：啟動結構化日誌
enableConsoleCapture();           // 攔截 console.log → 結構化 log
                                  // C# 等價：ILoggerFactory + Console Provider

// 第 3 步：檢查 runtime
assertSupportedRuntime();         // 確認 Node.js >= 22.12.0
                                  // C# 等價：RuntimeInformation.FrameworkDescription 檢查

// 第 4 步：建立 CLI
import { buildProgram } from "./cli/program.js";
const program = buildProgram();   // 建立指令樹
```

### export（公開 API）

```typescript
// index.ts 的 export 就是這個套件的「公開介面」
export {
  loadConfig,           // 載入 YAML 設定
  monitorWebChannel,    // 啟動 Web channel monitor
  runExec,              // 執行外部指令
  // ... 其他 20+ 個 export
};
```

```csharp
// C# 等價：就是 public class / public method
// 這些 export 就像你 NuGet 套件裡的 public API
public static class OpenClaw
{
    public static Config LoadConfig() { ... }
    public static void MonitorWebChannel() { ... }
    public static ProcessResult RunExec() { ... }
}
```

---

## 4.4 CLI 指令系統

### Commander.js（vs System.CommandLine）

```typescript
// src/cli/program/build-program.ts
import { Command } from "commander";

const program = new Command("openclaw")
  .version(VERSION)
  .description("Multi-channel AI gateway");

// 子指令註冊
program
  .command("gateway")
  .description("Start the gateway server")
  .option("--port <number>", "port", "18789")
  .option("--bind <mode>", "bind mode", "loopback")
  .action(async (options) => {
    // 啟動 Gateway...
  });

program
  .command("agent")
  .description("Run agent mode")
  .action(async (options) => { ... });
```

```csharp
// C# System.CommandLine 等價
var rootCommand = new RootCommand("Multi-channel AI gateway");

var gatewayCommand = new Command("gateway", "Start the gateway server");
gatewayCommand.AddOption(new Option<int>("--port", () => 18789));
gatewayCommand.AddOption(new Option<string>("--bind", () => "loopback"));
gatewayCommand.SetHandler(async (port, bind) => {
    // 啟動 Gateway...
}, portOption, bindOption);

rootCommand.AddCommand(gatewayCommand);
await rootCommand.InvokeAsync(args);
```

### CLI 指令清單

看 `src/cli/program/` 目錄下的 `register.*.ts` 檔案：

| 檔案 | 指令 | 說明 |
|------|------|------|
| `register.agent.ts` | `openclaw agent` | AI Agent 模式 |
| `register.configure.ts` | `openclaw configure` | 設定管理 |
| `register.maintenance.ts` | `openclaw doctor`, `openclaw update` | 維護工具 |
| `register.message.ts` | `openclaw message` | 發送訊息 |
| `register.onboard.ts` | `openclaw setup` | 首次設定導引 |
| `register.setup.ts` | `openclaw config` | 組態設定 |
| `register.status-health-sessions.ts` | `openclaw status` | 狀態查詢 |
| `register.subclis.ts` | `openclaw gateway`, `openclaw tui` | 子 CLI |

---

## 4.5 從 CLI 到 Gateway 的連接

```typescript
// 當使用者執行 `openclaw gateway` 時：
// register.subclis.ts 裡會動態 import gateway 模組

program
  .command("gateway")
  .action(async (options) => {
    const { bootGateway } = await import("../../gateway/boot.js");
    await bootGateway(options);
  });
```

```csharp
// C# 等價
gatewayCommand.SetHandler(async (options) =>
{
    // 延遲載入 Gateway 模組（避免 CLI help 也要載入整個 Gateway）
    var gateway = new GatewayBootstrapper();
    await gateway.BootAsync(options);
});
```

**設計用意**：用動態 import 實現「延遲載入」，使用者只是打 `openclaw --help` 時不需要載入整個 Gateway。類似 C# 的 `Lazy<T>` 或 Assembly lazy loading。

---

## 4.6 【v2026.2.26 新增】openclaw agents bindings — Agent 路由管理

這是 v2026.2.26 新增的 CLI 功能，讓你可以把特定帳號的訊息綁定到指定的 Agent。

### 概念

```
Account-scoped Agent Binding：
  某個 Telegram 帳號的訊息 → 固定路由到某個指定的 Agent

用途：
  - 讓不同使用者觸發不同的 AI Agent（各自有不同人設/工具）
  - 多帳號多 Agent 場景的靜態路由管理
```

### 新增指令

```bash
# 查看所有已綁定的路由
openclaw agents bindings

# 把某個帳號綁定到指定 agent
openclaw agents bind <channel>:<account> --agent <agentId>
# 例如：openclaw agents bind telegram:bot1 --agent coding-agent

# 解除綁定
openclaw agents unbind <channel>:<account>
# 例如：openclaw agents unbind telegram:bot1
```

### C# 對照理解

```csharp
// C# 概念：這相當於一個靜態路由表
// 類似 ASP.NET Core 的 Endpoint Routing，但針對 Agent

// 綁定表儲存在 ~/.openclaw/ 下的某個 JSON 檔
public class AgentBindingStore
{
    // Key: "{channel}:{account}", Value: agentId
    private Dictionary<string, string> _bindings = new();

    public void Bind(string channelAccount, string agentId)
        => _bindings[channelAccount] = agentId;

    public void Unbind(string channelAccount)
        => _bindings.Remove(channelAccount);

    public string? Resolve(string channelAccount)
        => _bindings.GetValueOrDefault(channelAccount);
}
```

### 對路由解析的影響

```typescript
// 新版的 resolve-route.ts 邏輯（概念化）
function resolveRoute(context: ChatContext, config: Config): Route {
  const bindingKey = `${context.channelId}:${context.accountId}`;

  // 【新增】先查 agent binding 表
  const boundAgentId = agentBindings.resolve(bindingKey);
  if (boundAgentId) {
    return { type: "agent", agentId: boundAgentId };
  }

  // 原有邏輯...
  const agentId = extractAgentId(context.text);
  if (agentId) return { type: "agent", agentId };

  if (context.chatType === "group" && !context.isMentioned) {
    return { type: "ignore" };
  }

  return { type: "chat", sessionKey: deriveSessionKey(context) };
}
```

### 閱讀作業補充

閱讀 `src/cli/program/` 目錄時，找看看是否有 `register.agents.ts` 或類似的新檔案，確認 `agents bindings` 指令的實作位置。

---

## 今日閱讀作業

### 作業 1：逐行閱讀 `src/entry.ts`（全文 172 行）
- 標記出你看不懂的語法，對照 Day 1-2 的筆記
- 特別注意 `normalizeWindowsArgv` 怎麼處理 Windows 路徑

### 作業 2：逐行閱讀 `src/index.ts`（全文 93 行）
- 注意啟動順序：loadDotEnv → normalizeEnv → enableConsoleCapture → buildProgram
- 理解 `isMainModule` 的判斷邏輯（類似 Python 的 `if __name__ == "__main__"`）

### 作業 3：瀏覽 `src/cli/program/` 目錄
- 讀 `build-program.ts`，理解 Commander.js 怎麼建立指令樹
- 掃一眼各個 `register.*.ts`，了解有哪些子指令

---

## 今日 Checkpoint

1. `entry.ts` 為什麼需要「respawn 自己」？
2. `index.ts` 裡的 `loadDotEnv()` 在做什麼？C# 的等價是什麼？
3. 為什麼 Gateway 用動態 `import()` 而不是靜態 `import`？
4. Commander.js 的 `.command().option().action()` 對應 C# System.CommandLine 的什麼？
5. `isMainModule` 在判斷什麼？
6. 【v2026.2.26】`openclaw agents bind` 解決什麼問題？

---

## 答案

1. 要加上 `--disable-warning=ExperimentalWarning` flag 來抑制 Node.js 的實驗性功能警告，但這個 flag 只能在啟動時傳，所以要 respawn。
2. 讀取 `.env` 檔案裡的環境變數。C# 等價：`ConfigurationBuilder.AddJsonFile("appsettings.json")` 或 `DotNetEnv.Env.Load()`。
3. **延遲載入**。使用者只打 `openclaw --help` 時不需要載入整個 Gateway 模組。類似 C# 的 `Lazy<T>`。
4. `new Command()` → `.AddOption()` → `.SetHandler()`
5. 判斷這個檔案是被「直接執行」還是被「其他模組 import」。類似 Python 的 `if __name__ == "__main__"` 或 C# 的判斷 Assembly 入口。
6. 把特定 channel 帳號的訊息**靜態綁定**到指定 Agent，讓路由在訊息進來前就決定好，不用依賴訊息內容中的 `@agent` 指令。
