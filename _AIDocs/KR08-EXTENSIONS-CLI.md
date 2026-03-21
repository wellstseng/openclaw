# 08-EXTENSIONS-CLI — Extensions 與 CLI

> 來源：openclaw-knowledge-base.md §8 §13 + F058 §11 §13 + F059 §10 §11 + F060

---

## 1. Extension 完整清單（44 個）

### 類別分組

```
總計：44 個 Extensions
  - 20 Channel Extensions
  -  7 AI Provider Extensions
  -  3 Memory Extensions
  - 14 Runtime Extensions
```

### 20 個 Channel Extensions

| ID | 顯示名稱 | 平台 |
|----|---------|------|
| `discord` | Discord | Discord |
| `telegram` | Telegram | Telegram |
| `whatsapp` | WhatsApp | WhatsApp |
| `signal` | Signal | Signal |
| `imessage` | iMessage | Apple iMessage |
| `slack` | Slack | Slack |
| `matrix` | Matrix | Matrix/Element |
| `mattermost` | Mattermost | Mattermost |
| `msteams` | Microsoft Teams | MS Teams |
| `irc` | IRC | IRC |
| `line` | LINE | LINE |
| `feishu` | Feishu（飛書） | Feishu/Lark |
| `googlechat` | Google Chat | Google Chat |
| `nextcloud-talk` | Nextcloud Talk | Nextcloud |
| `twitch` | Twitch | Twitch |
| `nostr` | Nostr | Nostr 去中心化 |
| `tlon` | Tlon（Urbit） | Urbit |
| `zalo` | Zalo | Zalo（越南） |
| `zalouser` | Zalo User | Zalo 個人（風險） |
| `synology-chat` | Synology Chat | Synology NAS |
| `bluebubbles` | BlueBubbles | iMessage bridge |

### 7 個 AI Provider Extensions

| ID | Provider |
|----|---------|
| `anthropic` | Anthropic Claude |
| `openai` | OpenAI GPT |
| `google` | Google Gemini |
| `aws-bedrock` | AWS Bedrock |
| `ollama` | Ollama（本地） |
| `copilot-proxy` | GitHub Copilot |
| `google-gemini-cli-auth` | Google Gemini（CLI 認證） |
| `sglang` | SGLang（本地） |
| `vllm` | vLLM（本地） |
| `qwen-portal-auth` | Alibaba Qwen |
| `minimax-portal-auth` | MiniMax |
| `openrouter` | OpenRouter |
| `nvidia` | NVIDIA API |
| `groq` | Groq |
| `mistral` | Mistral AI |
| `together` | Together AI |

### 3 個 Memory Extensions

| ID | 功能 |
|----|------|
| `memory-lancedb` | LanceDB + SQLite FTS5 後端 |
| `memory-core` | 記憶抽象介面 + 工具函式 |
| `memory-volatile` | 記憶體暫存（測試用，無持久化） |

### 14 個 Runtime Extensions

| ID | 功能 |
|----|------|
| `lobster` | Code 執行沙盒（LobsterEnvelope） |
| `browser` | 瀏覽器自動化 |
| `canvas-host` | A2UI canvas 介面 |
| `node-host` | Companion app 節點 |
| `device-pair` | QR/setup code pairing |
| `diffs` | 程式碼 diff 工具 |
| `diagnostics-otel` | OpenTelemetry 診斷 |
| `media-understanding` | 媒體理解（圖片/音訊） |
| `link-understanding` | URL 分析 |
| `tts` | 文字轉語音 |
| `wizard` | 新手引導 |
| `daemon` | 背景 process 管理 |
| `terminal` | Terminal UI |
| `web` | WhatsApp Web provider |

---

## 2. 各 Extension 的 registerXxx 呼叫

### memory-lancedb

```typescript
// extensions/memory-lancedb/index.ts
api.registerMemoryBackend({
  id: "lancedb",
  createBackend: (config) => new LanceDbMemoryBackend(config),
});
```

### memory-core

```typescript
// extensions/memory-core/index.ts
api.registerTools([
  "memory_recall",
  "memory_store",
  "memory_forget",
  "memory_list",
  "memory_update",
]);
api.registerHooks([
  "memory:before_capture",
  "memory:after_capture",
  "memory:before_recall",
  "memory:after_recall",
]);
```

### discord

```typescript
// extensions/discord/index.ts
api.registerChannel({
  id: "discord",
  createRuntime: (config) => new DiscordChannelRuntime(config),
});
api.registerSlashCommands(["/status", "/model", "/reasoning", "/approve", "/deny"]);
api.registerHooks([
  "discord:before_dispatch",
  "discord:after_delivery",
  "discord:on_thread_create",
  "discord:on_thread_bind",
]);
```

### telegram

```typescript
// extensions/telegram/index.ts
api.registerChannel({
  id: "telegram",
  createRuntime: (config) => new TelegramChannelRuntime(config),
});
```

### slack

```typescript
// extensions/slack/index.ts
api.registerChannel({
  id: "slack",
  createRuntime: (config) => new SlackChannelRuntime(config),
});
```

### ollama

```typescript
// extensions/ollama/index.ts
api.registerProvider({
  id: "ollama",
  createProvider: (config) => new OllamaProvider(config),
});
```

---

## 3. Plugin SDK 介面（ExtensionApi 所有 register* 方法）

```typescript
interface ExtensionApi {
  // Channel 相關
  registerChannel(def: ChannelDefinition): void;
  registerSlashCommands(commands: string[]): void;

  // Provider 相關
  registerProvider(def: ProviderDefinition): void;

  // Memory 相關
  registerMemoryBackend(def: MemoryBackendDefinition): void;
  registerTools(toolIds: string[]): void;

  // Hook 相關
  registerHooks(hookNames: string[]): void;
  onHook(name: string, handler: HookHandler): void;

  // 設定相關
  registerConfigSchema(schema: TypeBoxSchema): void;
  getConfig<T>(): T;

  // 事件相關
  onEvent(event: string, handler: EventHandler): void;
  emitEvent(event: string, data: unknown): void;

  // 工具相關
  registerTool(def: ToolDefinition): void;
  registerSkillBins(paths: string[]): void;

  // 節點相關
  registerNode(def: NodeDefinition): void;

  // 日誌
  getLogger(name: string): Logger;

  // 服務發現
  discover(): Promise<DiscoveryResult>;
}
```

---

## 4. Plugin 優先順序

```typescript
// Plugin 載入優先順序（高 → 低）：
// 1. config（配置檔明確指定的 plugin 路徑）
// 2. workspace（專案目錄下的 plugins/）
// 3. bundled（內建 extension）
// 4. global（~/.openclaw/plugins/）

// 優先順序規則：
// - 高優先度的 plugin 可覆蓋低優先度（相同 ID）
// - 所有優先度的 plugin 都被載入（不同 ID）
```

### Plugin Discovery Cache Key

```typescript
// Cache key 格式（含 Unix UID 確保多用戶隔離）：
const cacheKey = `plugin-discovery-${process.getuid()}-${configHash}`;
```

---

## 5. 24 個 Hook 名稱

```typescript
const ALL_HOOKS = [
  // Agent hooks
  "agent:before_run",          // Agent 執行前
  "agent:after_run",           // Agent 執行後
  "agent:before_tool",         // 工具執行前
  "agent:after_tool",          // 工具執行後
  "agent:on_error",            // 發生錯誤

  // Memory hooks
  "memory:before_capture",     // 記憶捕獲前
  "memory:after_capture",      // 記憶捕獲後
  "memory:before_recall",      // 記憶召回前
  "memory:after_recall",       // 記憶召回後

  // Discord hooks
  "discord:before_dispatch",   // Discord dispatch 前
  "discord:after_delivery",    // Discord 傳送後
  "discord:on_thread_create",  // Thread 建立
  "discord:on_thread_bind",    // Thread binding

  // Session hooks
  "session:created",           // Session 建立
  "session:resumed",           // Session 恢復
  "session:expired",           // Session 過期

  // Gateway hooks
  "gateway:before_route",      // 路由前
  "gateway:after_route",       // 路由後

  // Channel hooks
  "channel:inbound",           // 入站訊息
  "channel:outbound",          // 出站訊息

  // Provider hooks
  "provider:before_call",      // LLM 呼叫前
  "provider:after_call",       // LLM 呼叫後

  // Cron hooks
  "cron:before_job",           // Cron 任務前
  "cron:after_job",            // Cron 任務後
];
```

### PluginKind

```typescript
type PluginKind =
  | "channel"    // Channel 類型
  | "provider"   // LLM Provider 類型
  | "memory"     // Memory Backend 類型
  | "runtime"    // Runtime 類型（工具/功能）
  | "mixed";     // 混合類型
```

---

## 6. 所有 CLI 命令完整參數

### gateway

```bash
openclaw gateway [options]
  --port <number>       # Gateway HTTP port（預設 3000）
  --host <string>       # 監聽 host（預設 0.0.0.0）
  --config <path>       # 配置檔路徑
  --daemon              # 以 daemon 模式執行
  --log-level <level>   # 日誌級別（debug/info/warn/error）
```

### config

```bash
openclaw config <subcommand>
  get <key>             # 取得配置值
  set <key> <value>     # 設定配置值
  list                  # 列出所有配置
  edit                  # 開啟編輯器編輯配置
  validate              # 驗證配置是否合法
  reset                 # 重置為預設值
```

### plugins

```bash
openclaw plugins <subcommand>
  list                  # 列出已安裝 plugins
  install <path|url>    # 安裝 plugin
  uninstall <id>        # 移除 plugin
  enable <id>           # 啟用 plugin
  disable <id>          # 停用 plugin
  info <id>             # 查看 plugin 資訊
```

### memory

```bash
openclaw memory <subcommand>
  list [--agent <id>]       # 列出記憶
  search <query>            # 搜尋記憶
  store <content>           # 儲存記憶
  forget <id|query>         # 刪除記憶
  export [--format json]    # 匯出記憶
  import <file>             # 匯入記憶
  stats                     # 統計資訊
```

### sessions

```bash
openclaw sessions <subcommand>
  list                      # 列出 sessions
  view <session-key>        # 查看 session 詳情
  delete <session-key>      # 刪除 session
  export <session-key>      # 匯出 session transcript
  cleanup [--older-than N]  # 清理舊 sessions
```

### pairing

```bash
openclaw pairing <subcommand>
  generate                  # 生成 pairing QR code
  list                      # 列出已 paired 裝置
  revoke <device-id>        # 撤銷 pairing
  status                    # 查看 pairing 狀態
```

### nodes

```bash
openclaw nodes <subcommand>
  list                      # 列出已知 nodes
  connect <url>             # 連接到 node
  disconnect <node-id>      # 斷開 node
  status                    # 查看 nodes 狀態
  register <type>           # 注冊 node
```

### status

```bash
openclaw status             # 查看系統整體狀態
  --json                    # JSON 格式輸出
  --verbose                 # 詳細資訊
```

---

## 7. Slash Command 完整列表

```typescript
// 在 Chat channel 中可使用的 slash commands
const SLASH_COMMANDS = [
  "/status",      // 查看 Agent 狀態
  "/model",       // 查看/切換模型
  "/reasoning",   // 切換 reasoning/thinking 模式
  "/approve",     // 批准 exec 請求
  "/deny",        // 拒絕 exec 請求
  "/memory",      // 記憶操作
  "/reset",       // 重置 session
  "/help",        // 顯示說明
  "/debug",       // 切換 debug 模式
  "/stream",      // 切換 streaming 模式
];
```

---

## 8. nodes-cli 9 個 register.*.ts 節點指令

```typescript
// src/cli/nodes-cli/ 下的 9 個 register.*.ts 檔案
const NODE_CLI_REGISTRATIONS = [
  "register.browser.ts",      // 瀏覽器節點
  "register.canvas.ts",       // Canvas 節點
  "register.exec.ts",         // 命令執行節點
  "register.file.ts",         // 檔案操作節點
  "register.memory.ts",       // 記憶節點
  "register.process.ts",      // 程序管理節點
  "register.search.ts",       // 搜尋節點
  "register.terminal.ts",     // Terminal 節點
  "register.web.ts",          // Web 請求節點
];
```

---

## 9. src/commands/ 目錄結構（175+ 非測試 ts 檔）

```
src/commands/
├── agent/              # Agent 相關命令
├── agents/             # 多 Agent 管理
├── auth-choice/        # 認證方式選擇
├── configure/          # 設定命令
├── doctor/             # 診斷命令
├── status/             # 狀態命令
├── daemon/             # Daemon 管理
├── sessions/           # Session 管理
├── backup/             # 備份命令
└── onboard/            # 新手引導
```

---

## 10. lobster Extension

```typescript
// lobster：程式碼執行沙盒
interface LobsterEnvelope {
  status: "ok" | "needs_approval" | "cancelled";
  command: string;           // 執行的命令
  output?: string;           // 執行輸出
  error?: string;            // 錯誤訊息
  exitCode?: number;
}

// cwd 防逃逸：
// - 強制限制工作目錄在允許的範圍內
// - 防止 ../../../ 路徑逃逸
// - 使用 realpath 正規化後再驗證
```
