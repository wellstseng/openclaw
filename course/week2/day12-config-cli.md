# Day 12：Config + CLI + Routing

> **目標**：理解 YAML 設定管理、Session 系統、路由解析、CLI 指令架構
> **C# 對照**：IConfiguration + appsettings.json、Session Middleware、System.CommandLine
> **預計時間**：4 小時

---

## 12.1 Config 系統

### YAML 設定檔（vs appsettings.json）

```yaml
# ~/.openclaw/config.yaml — 主設定檔
identity:
  name: "Claw"
  systemPrompt: "You are a helpful AI assistant."

models:
  anthropic:
    apiKey: ${ANTHROPIC_API_KEY}     # 支援環境變數
    models:
      - id: claude-sonnet-4-5-20250929
        contextWindow: 200000
  openai:
    apiKey: ${OPENAI_API_KEY}
    models:
      - id: gpt-4o

channels:
  telegram:
    token: ${TELEGRAM_BOT_TOKEN}
    allowFrom: ["123456789"]          # 白名單
  whatsapp:
    allowFrom: ["+886912345678"]
  discord:
    token: ${DISCORD_BOT_TOKEN}
    dm:
      allowFrom: ["user-id-here"]

memory:
  enabled: true
  embeddingProvider: openai

cron:
  - schedule: "0 9 * * *"
    action: send
    channel: telegram
    target: "123456789"
    message: "Good morning!"
```

```csharp
// C# 等價：appsettings.json
{
  "Identity": {
    "Name": "Claw",
    "SystemPrompt": "You are a helpful AI assistant."
  },
  "Models": {
    "Anthropic": { "ApiKey": "...", "Models": [...] }
  },
  "Channels": {
    "Telegram": { "Token": "...", "AllowFrom": ["123456789"] }
  }
}

// 綁定到強型別
services.Configure<OpenClawConfig>(configuration);
```

### Config 載入流程

```typescript
// src/config/config.ts
async function loadConfig(): Promise<OpenClawConfig> {
  const configPath = resolveConfigPath();  // ~/.openclaw/config.yaml
  const raw = await readFile(configPath, "utf-8");
  const parsed = yaml.parse(raw);          // YAML → JS object

  // 環境變數替換 ${VAR_NAME}
  const expanded = expandEnvVars(parsed);

  // Schema 驗證
  const validated = validateConfig(expanded);

  return validated;
}
```

```csharp
// C# 等價
var config = new ConfigurationBuilder()
    .AddYamlFile(configPath)       // 需要 YamlDotNet.Extensions.Configuration
    .AddEnvironmentVariables()
    .Build()
    .Get<OpenClawConfig>();
```

### Hot Reload（config-reload.ts）

```typescript
// 監聽設定檔變更，自動重新載入
import { watch } from "chokidar";

function watchConfig(configPath: string, onReload: () => void) {
  const watcher = watch(configPath);
  watcher.on("change", async () => {
    const newConfig = await loadConfig();
    applyConfigChanges(newConfig);
    onReload();
  });
}
```

```csharp
// C# 等價：IOptionsMonitor<T>
services.Configure<OpenClawConfig>(configuration);
// IOptionsMonitor 自動監聽 appsettings.json 的變更
```

---

## 12.2 Session 系統

### Session Key 解析

```typescript
// src/routing/session-key.ts

// Session Key 格式：{channel}:{accountId}:{targetId}
// 例如：telegram:bot1:123456789
// 例如：whatsapp:default:+886912345678

function deriveSessionKey(context: ChatContext): string {
  const channel = context.channelId;
  const account = context.accountId ?? "default";
  const target = context.chatType === "direct"
    ? context.from
    : context.to;  // 群組用群組 ID

  return `${channel}:${account}:${target}`;
}
```

```csharp
// C# 等價
public string DeriveSessionKey(ChatContext context)
{
    var channel = context.ChannelId;
    var account = context.AccountId ?? "default";
    var target = context.ChatType == "direct"
        ? context.From : context.To;
    return $"{channel}:{account}:{target}";
}
```

### Session Store

```typescript
// src/config/sessions.ts
// Session 資料存在 ~/.openclaw/sessions/ 目錄下
// 每個 session 一個 JSON 檔案

async function loadSessionStore(sessionKey: string): Promise<SessionData> {
  const filePath = resolveStorePath(sessionKey);
  // sessions/telegram/bot1/123456789.json
  const raw = await readFile(filePath, "utf-8");
  return JSON.parse(raw);
}
```

---

## 12.3 Routing — 路由解析

```typescript
// src/routing/resolve-route.ts

// 決定一則訊息要送到哪裡
function resolveRoute(context: ChatContext, config: Config): Route {
  // 1. 檢查是否有指定 agent
  const agentId = extractAgentId(context.text);
  if (agentId) {
    return { type: "agent", agentId };
  }

  // 2. 檢查是否是群組訊息
  if (context.chatType === "group") {
    // 群組需要 @mention 才回覆
    if (!context.isMentioned) {
      return { type: "ignore" };
    }
  }

  // 3. 預設路由到 main session
  return {
    type: "chat",
    sessionKey: deriveSessionKey(context),
  };
}
```

---

## 12.4 CLI 指令架構深入

```typescript
// src/cli/program/register.subclis.ts
// 各種子指令的註冊

// openclaw gateway [options]
registerGatewayCommand(program);

// openclaw agent [options]
registerAgentCommand(program);

// openclaw tui
registerTuiCommand(program);

// openclaw message <channel> <target> <text>
registerMessageCommand(program);

// openclaw status
registerStatusCommand(program);

// openclaw doctor
registerDoctorCommand(program);

// openclaw setup
registerSetupCommand(program);
```

### 指令範例

```bash
# 啟動 Gateway
openclaw gateway --port 18789 --bind lan

# 發送訊息
openclaw message telegram 123456789 "Hello!"

# 查看狀態
openclaw status

# 互動式設定
openclaw setup

# 健康檢查
openclaw doctor
```

---

## 12.5 【v2026.2.26 新增】openclaw secrets — Secret 管理工作流程

v2026.2.26 最大的新功能之一：完整的 Secret 生命週期管理，讓 API Key 的管理從「手動改 config.yaml」升級為「有稽核、有版控、有安全邊界」的正式流程。

### 概念：為什麼需要 secrets 管理？

```
舊做法：直接把 API Key 寫進 config.yaml
  → API Key 明文存在設定檔
  → 多人協作時容易洩漏
  → 沒有稽核記錄

新做法：openclaw secrets 工作流程
  → Key 儲存在受保護的 secret store
  → config.yaml 只存 reference（$secret:key-name）
  → 有完整稽核、套用、重載流程
```

```csharp
// C# 對照：類似 Azure Key Vault / ASP.NET Core Data Protection
// 或 .NET 的 Secret Manager (dotnet user-secrets)

// 舊做法（config.yaml）：
// anthropic:
//   apiKey: "sk-ant-xxxxxxxxxxxx"   ← 明文

// 新做法（config.yaml）：
// anthropic:
//   apiKey: "$secret:anthropic-api-key"  ← reference

// Secret 實際存在受保護的 store 裡（類似 dotnet user-secrets）
```

### 四個子指令

```bash
# 1. audit：掃描設定檔，找出哪些欄位應該改用 secret reference
openclaw secrets audit
# 輸出範例：
# ⚠ channels.telegram.token — 建議改為 secret reference
# ⚠ models.anthropic.apiKey — 建議改為 secret reference

# 2. configure：互動式設定 secret（輸入 key 名稱和值）
openclaw secrets configure
# 互動式：
# > Secret name: anthropic-api-key
# > Secret value: sk-ant-xxxxxx （隱藏輸入）
# ✓ Secret saved

# 3. apply：把 secret reference 套用到執行中的 Gateway
openclaw secrets apply
# 相當於「熱更新」：不需要重啟 Gateway 就能讓新 key 生效

# 4. reload：強制 Gateway 重新載入所有 secret
openclaw secrets reload
```

### Secret Reference 語法

```yaml
# config.yaml 的新寫法
models:
  anthropic:
    apiKey: "$secret:anthropic-api-key"   # reference，不是明文

channels:
  telegram:
    token: "$secret:telegram-bot-token"

  discord:
    token: "$secret:discord-token"
```

```csharp
// C# 對照：類似 ASP.NET Core 的 Configuration 引用
// appsettings.json 裡用 ${SECRET_NAME}，實際值從 Key Vault 取

{
  "Anthropic": {
    "ApiKey": "${anthropic-api-key}"   // Azure Key Vault reference
  }
}
```

### Runtime Snapshot 機制

```typescript
// secrets apply 的底層機制（概念化）

// 1. 讀取 secret store 裡的最新值
const secrets = await loadSecretStore();

// 2. 把 config.yaml 裡的 $secret:xxx reference 替換成實際值
const resolvedConfig = resolveSecretRefs(rawConfig, secrets);

// 3. 啟動一個新的 "snapshot"（快照），Gateway 切換到使用新快照
await gateway.activateConfigSnapshot(resolvedConfig);

// 4. 整個過程不重啟 Gateway，使用者感受不到中斷
```

```csharp
// C# 等價：IOptionsMonitor<T> 的 OnChange + 手動 reload
public class SecretAwareConfigMonitor
{
    public async Task ApplySecretsAsync()
    {
        var secrets = await _secretStore.LoadAllAsync();
        var resolved = _resolver.ResolveRefs(_rawConfig, secrets);
        _optionsMonitor.CurrentValue = resolved;  // 熱更新
    }
}
```

### Auth Profile 的 ref-only 模式

v2026.2.26 還讓 auth-profiles 支援純 reference 模式——API Key 完全不出現在任何設定檔，只存在 secret store 裡：

```yaml
# auth-profiles.json（ref-only 模式）
[
  {
    "id": "anthropic-main",
    "provider": "anthropic",
    "key": "$secret:anthropic-api-key"   # 只有 reference
  }
]
```

### 閱讀作業補充

找找看 `src/cli/program/` 或 `src/` 目錄下是否有 `secrets.ts` 或 `register.secrets.ts`，追蹤四個 subcommand 的實作。

---

## 今日閱讀作業

### 作業 1：閱讀 `src/config/config.ts`
- 理解設定檔的 schema 定義
- 找出環境變數替換的邏輯

### 作業 2：閱讀 `src/routing/resolve-route.ts` 和 `session-key.ts`
- 理解 session key 的組成
- 理解群組訊息的路由邏輯

### 作業 3：閱讀 `src/cli/program/` 目錄下的 register 檔案
- 挑 2-3 個指令深入理解

---

## 今日 Checkpoint

1. OpenClaw 的設定檔用什麼格式？存在哪裡？
2. Session Key 的格式是什麼？
3. 群組訊息預設什麼時候才會回覆？
4. Hot Reload 用什麼機制監聽檔案變更？
5. `openclaw doctor` 指令的用途是什麼？
6. 【v2026.2.26】`openclaw secrets apply` 和重啟 Gateway 有什麼差異？
7. 【v2026.2.26】`$secret:xxx` 在 config.yaml 裡是什麼意思？

---

## 答案

1. **YAML** 格式，存在 `~/.openclaw/config.yaml`。
2. `{channel}:{accountId}:{targetId}`，如 `telegram:bot1:123456789`。
3. 被 **@mention** 的時候才回覆（避免在群組裡對每則訊息都回）。
4. 用 **chokidar** 套件監聽檔案系統事件（類似 C# 的 `FileSystemWatcher`）。
5. 健康檢查——檢查 Node.js 版本、設定檔格式、API Key 是否有效、Channel 連線狀態等。
6. `secrets apply` 是**熱更新**——啟動新的 config snapshot，使用者感受不到中斷。重啟 Gateway 則會斷開所有 WebSocket 連線。
7. **Secret Reference**——不是明文 API Key，而是指向 secret store 裡某個 key 的引用。類似 Azure Key Vault reference 或 `dotnet user-secrets` 的概念。
