# Plugin SDK + Plugin System 深入

> Phase 5-1 | 掃描 `src/plugin-sdk/` 110 files (~8.8K LOC) + `src/plugins/` 83 files (~15K LOC) + `extensions/` 33+ plugins
> 總計 ~193 files, ~24K LOC（不含 extensions 實作）

---

## 1. 架構鳥瞰

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Plugin Lifecycle                             │
│                                                                     │
│  Discovery ──→ Manifest ──→ Loader ──→ Registry ──→ Activation     │
│  (4 origins)   (validate)  (jiti)    (register)   (global state)   │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                        Plugin SDK                                   │
│                                                                     │
│  index.ts (799 lines, 600+ exports)                                │
│  ├── Channel types & adapters (20+ interfaces)                     │
│  ├── Webhook infrastructure (targets, guards, rate limit)          │
│  ├── Security (SSRF, auth scopes, allowlist)                       │
│  ├── File ops (lock, dedupe, JSON store)                           │
│  ├── Message flow (reply payload, envelope, dispatch)              │
│  ├── Access control (allow-from, group policy)                     │
│  ├── Config schemas (per-channel Zod schemas)                      │
│  ├── Context Engine registry                                       │
│  └── Channel-specific SDKs (discord/telegram/slack/...)            │
│                                                                     │
│  root-alias.cjs ── jiti Proxy 延遲載入                              │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                        Registration API                             │
│                                                                     │
│  registerTool / registerHook / registerHttpRoute / registerChannel │
│  registerGatewayMethod / registerCli / registerService             │
│  registerProvider / registerCommand / registerContextEngine        │
│  on<PluginHookName>()  ── 24 typed hooks                           │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                        Plugin Runtime                               │
│                                                                     │
│  config / system / media / tts / stt / tools / events / logging    │
│  state / subagent / channel (40+ methods)                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Plugin Lifecycle — 5 階段完整流程

```
Discovery ──→ Manifest ──→ Loading ──→ Registration ──→ Activation
  find          parse       jiti         API calls      global state
  candidates    validate    import       register()     hook runner
  4 origins     JSON        SDK alias    tools/hooks    Symbol key
```

### 2.1 Discovery（discovery.ts — 712 lines）

**核心函式**：`discoverOpenClawPlugins(params)`

**4 Origins 優先順序**（數字小優先）：

| Origin | 優先 | 來源路徑 | 說明 |
|--------|------|---------|------|
| `config` | 0 | `plugins.load.paths` 各 entry | 使用者明確指定 |
| `workspace` | 1 | `.openclaw/extensions/` | 工作區本地 |
| `global` | 2 | `~/.openclaw/extensions/` | 全域安裝 |
| `bundled` | 3 | `src/plugins/bundled-dir.ts` 指向 | 內建 |

**安全檢查**：
- `source_escapes_root`：symlink/realpath 逃逸偵測
- `path_world_writable`：目錄可被他人寫入（mode & 0o002）
- `path_suspicious_ownership`：非 bundled 且 UID 不符
- 異常僅發 `warn` diagnostic，不阻止載入

**Plugin Entry 解析順序**：
1. 有 `openclaw.plugin.json` → 讀 manifest + extensions 欄位
2. `package.json` 有 `openclaw.extensions` array → 用那些 entry
3. 否則找 `index.{ts,js,mjs,cjs}`
4. 或直接載入單檔 `.ts/.js`

**快取**：
- Key = workspace + uid + config dir + bundled dir + extra paths（排序）
- TTL = 1000ms（`OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` 可覆蓋）
- `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE` 可關閉

### 2.2 Manifest（manifest.ts + manifest-registry.ts — 461 lines）

**openclaw.plugin.json 結構**：
```json
{
  "id": "my-plugin",              // 必要
  "configSchema": { ... },        // 必要（JSON Schema）
  "name": "My Plugin",            // 選填
  "description": "...",           // 選填
  "version": "1.0.0",            // 選填
  "kind": "memory",              // 選填：slot 類型
  "channels": ["my-channel"],    // 選填
  "providers": ["my-provider"],  // 選填
  "skills": ["./skills"],        // 選填
  "uiHints": {                   // 選填：UI 顯示提示
    "key.path": {
      "label": "顯示名",
      "help": "說明文字",
      "sensitive": true,
      "advanced": true,
      "placeholder": "預設提示"
    }
  }
}
```

**package.json `openclaw` 欄位**：
```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "aliases": ["mc"],
      "preferOver": ["old-channel"]
    },
    "install": {
      "npmSpec": "@my/plugin",
      "localPath": "./local-override",
      "defaultChoice": "npm"
    }
  }
}
```

**去重邏輯**：同 ID 多 origin → realpath 比對 → 保留高優先 origin → 不同物理路徑發 warn

### 2.3 Loader（loader.ts — 886 lines）

**核心函式**：`loadOpenClawPlugins(options): PluginRegistry`

**載入步驟**：

| 步驟 | 位置 | 動作 |
|------|------|------|
| 1. Enable State | config-state.ts:241 | 解析 `plugins.allow/deny`、`entries[id].enabled` |
| 2. Slot Early Exit | loader.ts:695 | kind=memory 但 slot 不符 → 跳過 |
| 3. Module Load | loader.ts:732 | `jiti(safeSource)` 動態 import |
| 4. Export Parse | loader.ts:750 | 取 definition/register/activate 函式 |
| 5. Slot Decision | config-state.ts:258 | 互斥 slot 判定（僅一個可啟用） |
| 6. Config Validation | loader.ts:802 | plugin config vs JSON Schema 驗證 |
| 7. API Creation | loader.ts:826 | 建立 `OpenClawPluginApi` 實例 |
| 8. Register Call | loader.ts:840 | 呼叫 `register(api)` 或 `activate(api)` |

**Plugin Module 三種格式**：
```typescript
// 1. Definition object（推薦）
export default {
  id: "my-plugin",
  configSchema: emptyPluginConfigSchema(),
  register(api) { ... }
}

// 2. Factory function
export default (api) => { api.registerTool(...) }

// 3. CommonJS
module.exports = { id: "my-plugin", register(api) { ... } }
```

**jiti 設定**：
```javascript
createJiti(import.meta.url, {
  interopDefault: true,
  extensions: [".ts", ".tsx", ".mts", ".cts", ".js", ".mjs", ".cjs", ".json"],
  alias: {
    "openclaw/plugin-sdk": <SDK path>,
    "openclaw/plugin-sdk/discord": <discord.ts>,
    "openclaw/plugin-sdk/telegram": <telegram.ts>,
    // ... 37+ scoped aliases
  }
})
```

**SDK Alias 解析**：
- 從 module 路徑向上掃描最多 6 層
- 開發模式優先 `src/`，production 優先 `dist/`
- Fallback chain：`.ts → .js`、`.mts → .mjs`

**Provenance Tracking**：
- 追蹤 plugin 來自 `plugins.load.paths` 或 `plugins.installs[id]`
- 不在信任路徑中 → 發 warn diagnostic

### 2.4 Registry（registry.ts — 625 lines）

**PluginRegistry 結構**：
```typescript
{
  plugins: PluginRecord[]              // 所有已載入 plugin + 狀態
  tools: PluginToolRegistration[]      // 工具
  hooks: PluginHookRegistration[]      // 內部 hook
  typedHooks: TypedPluginHookRegistration[]  // 類型安全 hook
  channels: PluginChannelRegistration[]     // 頻道
  providers: PluginProviderRegistration[]   // LLM Provider
  gatewayHandlers: GatewayRequestHandlers   // Gateway RPC
  httpRoutes: PluginHttpRouteRegistration[] // HTTP 路由
  cliRegistrars: PluginCliRegistration[]    // CLI 命令
  services: PluginServiceRegistration[]     // 背景服務
  commands: PluginCommandRegistration[]     // Direct 命令
  diagnostics: PluginDiagnostic[]           // 診斷訊息
}
```

**PluginRecord 結構**：
```typescript
{
  id, name, version, description, kind,
  source: string,           // 檔案路徑
  origin: "bundled" | "global" | "workspace" | "config",
  enabled: boolean,
  status: "loaded" | "disabled" | "error",
  error?: string,
  toolNames: string[],
  hookNames: string[],
  channelIds: string[],
  providerIds: string[],
  gatewayMethods: string[],
  cliCommands: string[],
  services: string[],
  commands: string[],
  httpRoutes: number,       // count
  hookCount: number,
  configSchema: boolean,
  configUiHints?: Record<string, PluginConfigUiHint>,
  configJsonSchema?: Record<string, unknown>
}
```

**衝突偵測**：
- HTTP Route：`findOverlappingPluginHttpRoute()` → auth 不同則 reject，同 auth + `replaceExisting` 可覆蓋
- Provider：同 ID 重複 → error
- Gateway Method：同 method 重複 → error（conflict check）
- Command：同 name 重複 → error + 72 保留命令名

### 2.5 Activation（runtime.ts — 50 lines）

**全域狀態**：
```typescript
const REGISTRY_STATE = Symbol.for("openclaw.pluginRegistryState")

globalThis[REGISTRY_STATE] = {
  registry: PluginRegistry | null,
  key: string | null,        // cache key
  version: number            // 每次更新遞增
}
```

**存取 API**：
- `getActivePluginRegistry()` → optional
- `requireActivePluginRegistry()` → 必有（空則建新）
- `getActivePluginRegistryVersion()` → 變更偵測用
- `activatePluginRegistry()` → 設定全域 + 初始化 hook runner

---

## 3. Plugin SDK（src/plugin-sdk/ — 110 files, 8.8K LOC）

### 3.1 主入口 index.ts（799 lines, 600+ exports）

**匯出分類**：

| 類別 | 行範圍 | 主要匯出 |
|------|--------|---------|
| Channel 類型 | 1-62 | `ChannelPlugin`, 20+ adapter interfaces（Auth/Messaging/Security/Heartbeat/Threading/...） |
| ACP Runtime | 74-96 | `AcpRuntime`, `getAcpRuntimeBackend`, error codes |
| Plugin Core | 97-127 | `OpenClawPluginApi`, `PluginRuntime`, `emptyPluginConfigSchema` |
| File Ops | 133-342 | `acquireFileLock`, `withFileLock`, `createPersistentDedupe`, JSON store |
| Webhook | 148-175 | target routing, request guards, rate limiter, anomaly tracker |
| Config/Routing | 219-310 | per-channel Zod schemas, group policy, access control |
| Message Flow | 314-331 | outbound delivery, reply dispatch, media handling |
| Security | 406-449 | `fetchWithSsrFGuard`, SSRF utils, bearer auth fallback |
| Channel 特定 | 487-744 | Discord/Telegram/Slack/WhatsApp/LINE/iMessage/Signal normalization |
| Utilities | 597-625 | time format, dedupe cache, text utils, diagnostics |
| Context Engine | 783-800 | `ContextEngine`, `registerContextEngine`, `ContextEngineFactory` |

### 3.2 jiti Proxy 延遲載入（root-alias.cjs — 200 lines）

```
require('@clawdb/plugin-sdk')
        ↓
  root-alias.cjs (Proxy)
        ↓
  ┌─ Fast exports (即時可用)
  │   emptyPluginConfigSchema()
  │   resolveControlCommandGate()
  │
  └─ 其他 exports → Proxy.get → loadMonolithicSdk()
        ↓
    先嘗試 dist/plugin-sdk/index.js（快）
        ↓ 失敗
    fallback → jiti(src/plugin-sdk/index.ts)（慢但完整）
```

**Proxy 攔截**：
- `get()`：fast exports → lazy-load SDK → 取屬性
- `has()`：同上邏輯判斷屬性存在
- `ownKeys()`：只回傳已載入的 keys（確定性）
- `getOwnPropertyDescriptor()`：完整 descriptor 反射

### 3.3 Channel-Specific SDKs

每個頻道有獨立的 thin wrapper，只匯出該頻道相關 API：

| SDK 檔案 | 主要匯出 |
|---------|---------|
| `discord.ts` | `resolveDiscordAccount`, `autoBindSpawnedDiscordSubagent`, `collectDiscordStatusIssues` |
| `telegram.ts` | `resolveTelegramAccount`, `parseTelegramThreadId`, `TelegramProbe` |
| `slack.ts` | `resolveSlackAccount`, `extractSlackToolSend`, `handleSlackMessageAction` |
| `whatsapp.ts` | `normalizeWhatsAppTarget`, `resolveWhatsAppHeartbeatRecipients` |
| `signal.ts` | `resolveSignalAccount` |
| `line.ts` | `createInfoCard`, `createActionCard`, `processLineMessage`, Flex message builders |
| `imessage.ts` | `resolveIMessageAccount`, `resolveServicePrefixedChatTarget` |
| `bluebubbles.ts` | status + action helpers |
| `channel-plugin-common.ts` | 所有頻道共用基礎類型 |

Plugin import 方式：`import { ... } from "openclaw/plugin-sdk/discord"`

### 3.4 核心工具庫

#### File Lock（file-lock.ts — 161 lines）
- `acquireFileLock(filePath, opts)` → `FileLockHandle`
- Exponential backoff retry + PID staleness detection
- Reference counting 支援 re-entrant
- Process-scoped：`Symbol.for("openclaw.fileLockHeldLocks")`

#### Persistent Deduplication（persistent-dedupe.ts — 189 lines）
- Hybrid memory + disk cache
- Namespaced entries（如 `"webhooks:message-123"`）
- Auto-pruning（max entries limit）
- Inflight dedup 防並行重複

#### Keyed Async Queue（keyed-async-queue.ts — 49 lines）
- 同 key 的任務序列化執行
- Hooks：`onEnqueue`, `onSettle`
- 用於 message processing、state mutation

#### Webhook Infrastructure（3 files, 660+ lines）
- `registerWebhookTarget()` → target routing
- `createWebhookInFlightLimiter()` → 並行請求限制
- `createFixedWindowRateLimiter()` → 滑動視窗限流
- `createWebhookAnomalyTracker()` → 異常追蹤
- `readJsonWebhookBodyOrReject()` → body 大小+content-type 驗證

#### SSRF Protection（ssrf-policy.ts — 86 lines）
- `buildHostnameAllowlistPolicyFromSuffixAllowlist()` → suffix → pattern
- 允許 `example.com` + `*.example.com`
- 整合 `fetchWithSsrFGuard()`

---

## 4. OpenClawPluginApi — 完整介面

```typescript
type OpenClawPluginApi = {
  // 身份
  id: string
  name: string
  version?: string
  description?: string
  source: string                    // 檔案路徑

  // 設定
  config: OpenClawConfig            // 全域設定
  pluginConfig?: Record<string, unknown>  // plugin 專屬設定（已驗證）

  // 執行時
  runtime: PluginRuntime
  logger: PluginLogger

  // 10 個註冊方法
  registerTool(tool, opts?): void
  registerHook(events, handler, opts?): void
  registerHttpRoute(params): void
  registerChannel(registration): void
  registerGatewayMethod(method, handler): void
  registerCli(registrar, opts?): void
  registerService(service): void
  registerProvider(provider): void
  registerCommand(command): void
  registerContextEngine(id, factory): void

  // 工具
  resolvePath(input: string): string   // ~/ 路徑解析
  on<K extends PluginHookName>(hookName, handler, opts?): void  // typed hook
}
```

---

## 5. Registration API 詳解

### 5.1 registerTool — AI 工具

```typescript
// 方式 1：直接工具定義
api.registerTool({
  name: "my_tool",
  description: "...",
  parameters: TypeBoxSchema,
  async execute(toolCallId, params) { return { content: [...] } }
})

// 方式 2：工廠函式（推薦 — 可依 context 決定是否啟用）
api.registerTool(
  (ctx: OpenClawPluginToolContext) => {
    if (!ctx.config?.myPlugin?.enabled) return null  // 不啟用
    return { name: "my_tool", ... }
  },
  { names: ["my_tool"], optional: true }
)
```

**ToolContext**：
```typescript
{
  config?, workspaceDir?, agentDir?, agentId?,
  sessionKey?,          // 穩定（跨 message）
  sessionId?,           // 易變（/new 或 /reset 重設）
  messageChannel?,      // "discord" | "telegram" | ...
  agentAccountId?,
  requesterSenderId?,   // 信任的 sender ID
  senderIsOwner?,       // 權限檢查用
  sandboxed?
}
```

### 5.2 registerHook — 事件鉤子（內部 hook 系統）

```typescript
api.registerHook(
  ["message_received", "message_sent"],
  async (event) => { /* handle */ },
  { name: "my-logger", description: "Log all messages" }
)
```

### 5.3 on() — Typed Plugin Hooks（24 種）

```typescript
api.on("before_prompt_build", async (event) => ({
  prependSystemContext: "Always respond in Chinese",  // 可快取
  prependContext: dynamicContext,                      // per-turn
}), { priority: 100 })  // 高 priority 先執行
```

**完整 24 Hook 列表**：

| Hook | 階段 | 可回傳 | 說明 |
|------|------|--------|------|
| `before_model_resolve` | Pre-session | `{ modelOverride?, providerOverride? }` | 選擇 LLM |
| `before_prompt_build` | Session | `{ systemPrompt?, prependContext?, prependSystemContext?, appendSystemContext? }` | 準備 prompt |
| `before_agent_start` | Session | 同上兩者合併 | Legacy 相容 |
| `llm_input` | Runtime | void | LLM 即將呼叫 |
| `llm_output` | Runtime | void | LLM 回應收到 |
| `agent_end` | Runtime | void | Agent 完成 |
| `before_compaction` | Cleanup | void | 即將壓縮歷史 |
| `after_compaction` | Cleanup | void | 壓縮完成 |
| `before_reset` | Session | void | /new 或 /reset |
| `message_received` | Inbound | void | 收到訊息 |
| `message_sending` | Outbound | `{ content?, cancel? }` | 發送前修改/取消 |
| `message_sent` | Outbound | void | 已發送 |
| `before_tool_call` | Runtime | `{ params?, block?, blockReason? }` | 工具呼叫前 |
| `after_tool_call` | Runtime | void | 工具呼叫後 |
| `tool_result_persist` | Runtime | `{ message? }` | 結果寫入前 |
| `before_message_write` | Runtime | `{ block?, message? }` | JSONL 寫入前 |
| `session_start` | Lifecycle | void | Session 建立 |
| `session_end` | Lifecycle | void | Session 結束 |
| `subagent_spawning` | SubAgent | `{ status, error? }` | 子 agent 請求 |
| `subagent_delivery_target` | SubAgent | `{ origin? }` | 投遞路由 |
| `subagent_spawned` | SubAgent | void | 子 agent 已建 |
| `subagent_ended` | SubAgent | void | 子 agent 完成 |
| `gateway_start` | Gateway | void | Server 啟動 |
| `gateway_stop` | Gateway | void | Server 關閉 |

**Prompt Injection 防護**：
- `plugins.entries[id].hooks.allowPromptInjection = false` →
  - 完全阻擋 `before_prompt_build`
  - 限制 `before_agent_start`（剝除 prompt mutation 欄位）

**Hook Runner 合併策略**：
- `before_model_resolve`：第一個非 null override 勝出
- `before_prompt_build`：context 字串串接
- subagent hooks：結果合併（error 優先）

### 5.4 registerHttpRoute — HTTP 路由

```typescript
api.registerHttpRoute({
  path: "/plugins/my-plugin",
  auth: "plugin",        // "plugin"（需 API key）| "gateway"（Gateway 層 auth）
  match: "prefix",       // "exact" | "prefix"
  replaceExisting: false,
  handler: async (req, res) => {
    // return false → 不處理，交給下一個 handler
    // return true/void → 已處理
    res.writeHead(200, { "Content-Type": "application/json" })
    res.end(JSON.stringify({ ok: true }))
    return true
  }
})
```

### 5.5 registerChannel — 頻道

```typescript
api.registerChannel({
  plugin: myChannelPlugin,  // ChannelPlugin<TAccount, TProbe> 完整實作
  dock?: myDock              // 選填 UI 整合
})
```

**ChannelPlugin ~20 個 sub-adapter**：
- `config`：account CRUD（listAccountIds, resolveAccount, setEnabled, delete）
- `security`：DM policy, warnings
- `groups`：requireMention, toolPolicy
- `outbound`：sendText, sendMedia, chunker, textChunkLimit
- `directory`：self, listPeers, listGroups
- `gateway`：startAccount → `{ stop }`, probeAccount
- `status`：buildChannelSummary, defaultRuntime

### 5.6 registerProvider — LLM Provider

```typescript
api.registerProvider({
  id: "my-provider",
  label: "My Provider",
  aliases: ["mp"],
  envVars: ["MY_API_KEY"],
  auth: [{
    id: "api_key",
    label: "API Key",
    kind: "api_key",            // "oauth" | "api_key" | "token" | "device_code" | "custom"
    run: async (ctx) => ({
      profiles: [{ profileId: "default", credential: { apiKey: "..." } }],
      defaultModel: "my-model-v1",
      notes: ["Set MY_API_KEY in env"]
    })
  }],
  formatApiKey?: (cred) => `Bearer ${cred.apiKey}`,
  refreshOAuth?: (cred) => refreshedCred
})
```

### 5.7 registerCommand — Direct 命令（繞過 LLM）

```typescript
api.registerCommand({
  name: "my-cmd",                 // 使用 /my-cmd 觸發
  description: "Do something",
  acceptsArgs: true,
  requireAuth: true,              // 預設 true
  handler: async (ctx) => ({
    text: "Done!",
    error?: false
  })
})
```

**命令名規則**：`/^[a-z][a-z0-9_-]*$/`，72 個保留名（help, status, config, bash, exec, think, verbose...）

### 5.8 registerService — 背景服務

```typescript
api.registerService({
  id: "my-service",
  start: async (ctx) => { /* 啟動背景任務 */ },
  stop: async (ctx) => { /* 優雅關閉 */ }  // 選填
})
```

**ServiceContext**：`{ config, workspaceDir?, stateDir, logger }`

### 5.9 registerContextEngine — Context Engine（互斥 slot）

```typescript
api.registerContextEngine("my-engine", async () => ({
  bootstrap, ingest, assemble, compact, afterTurn, subagent
}))
```

只有一個 contextEngine 可啟用，slot 系統管理。

### 5.10 registerCli — CLI 命令

```typescript
api.registerCli(({ program }) => {
  program.command("my-cmd")
    .description("My CLI command")
    .argument("<query>")
    .option("--limit <n>", "Max results", "5")
    .action(async (query, opts) => {
      console.log(`Searching: ${query}, limit: ${opts.limit}`)
    })
}, { commands: ["my-cmd"] })
```

使用 commander.js，plugin 加子命令到頂層 program。

---

## 6. Plugin Runtime API

```typescript
runtime: {
  version: string                    // OCL 版本

  config: {
    loadConfig(): OpenClawConfig
    writeConfigFile(patch): void
  }

  system: {
    enqueueSystemEvent(event): void
    requestHeartbeatNow(): void
    runCommandWithTimeout(cmd, opts): Promise<Result>
    formatNativeDependencyHint(dep): string
  }

  media: {
    loadWebMedia(url): Promise<Buffer>
    detectMime(buffer): string
    mediaKindFromMime(mime): string
    isVoiceCompatibleAudio(path): boolean
    getImageMetadata(path): Promise<Metadata>
    resizeToJpeg(buffer, opts): Promise<Buffer>
  }

  tts: { textToSpeechTelephony(text, opts): Promise<Buffer> }
  stt: { transcribeAudioFile(path, opts): Promise<string> }

  tools: {
    createMemorySearchTool(opts): AgentTool | null
    createMemoryGetTool(opts): AgentTool | null
    registerMemoryCli(program): void
  }

  events: {
    onAgentEvent(handler): void
    onSessionTranscriptUpdate(handler): void
  }

  logging: {
    shouldLogVerbose(): boolean
    getChildLogger(bindings?, opts?): Logger
  }

  state: { resolveStateDir(): string }

  subagent: {
    run(params): Promise<{ runId }>
    waitForRun(params): Promise<{ status, error? }>
    getSessionMessages(params): Promise<{ messages }>
    deleteSession(params): Promise<void>
  }

  channel: {
    text: { chunkByNewline, chunkMarkdownText, ... }
    reply: { dispatchReply, withReplyDispatcher, ... }
    routing: { buildAgentSessionKey, resolveAgentRoute }
    pairing: { buildPairingReply, readAllowFromStore, ... }
    media: { fetchRemoteMedia, saveMediaBuffer }
    activity: { record, get }
    session: { resolveStorePath, readSessionUpdatedAt, ... }
    mentions: { buildMentionRegexes, matchesMentionPatterns, ... }
    reactions: { shouldAckReaction, removeAckReactionAfterReply }
    groups: { resolveGroupPolicy, resolveRequireMention }
    debounce: { createInboundDebouncer, ... }
    commands: { resolveCommandAuthorized, isControlCommand, ... }
    discord: { messageActions, auditChannelPermissions, ... }
    slack: { listDirectoryGroupsLive, probeSlack, ... }
    telegram: { auditGroupMembership, probeTelegram, ... }
    signal: { probeSignal, sendMessageSignal, ... }
    imessage: { monitorIMessageProvider, probeIMessage, ... }
    whatsapp: { getActiveWebListener, loginWeb, logoutWeb, ... }
    line: { listLineAccountIds, probeLineBot, sendMessageLine, ... }
  }
}
```

### SubAgent API 詳解

```typescript
// 啟動子 agent
const { runId } = await api.runtime.subagent.run({
  sessionKey: "sub-task-1",
  message: "Summarize this document",
  extraSystemPrompt?: "You are a summarizer",
  lane?: "default",
  deliver?: true,           // 自動投遞結果
  idempotencyKey?: "unique"
})

// 等待完成
const { status, error } = await api.runtime.subagent.waitForRun({
  runId,
  timeoutMs: 30_000
})

// 讀取結果
const { messages } = await api.runtime.subagent.getSessionMessages({
  sessionKey: "sub-task-1",
  limit: 10
})

// 清理
await api.runtime.subagent.deleteSession({
  sessionKey: "sub-task-1",
  deleteTranscript: true
})
```

---

## 7. Slot System — 互斥插槽

### 概念

Slot = 同類型 plugin 只能啟用一個。目前兩種：

| Slot | 預設 | 用途 |
|------|------|------|
| `memory` | `"memory-core"` | Memory/RAG 後端 |
| `contextEngine` | `"legacy"` | Context 組裝引擎 |

### 設定

```json
{
  "plugins": {
    "slots": {
      "memory": "memory-lancedb",        // 切換記憶後端
      "contextEngine": "my-engine"       // 切換 context 引擎
    }
  }
}
```

`null` = 停用整個 slot，`"none"` = 同效果。

### 決策邏輯（config-state.ts:258-286）

```
kind 不是 slot 類型 → 永遠啟用
kind="memory" + slot=null → disabled（"memory slot disabled"）
kind="memory" + slot="plugin-id" →
  id === slot → enabled + selected
  id !== slot → disabled（"memory slot set to \"plugin-id\""）
首個未設定 slot → enabled + selected（先到先得）
```

### Slot Selection API（slots.ts:39-110）

`applyExclusiveSlotSelection()` → 互動式選擇時自動 disable 其他同 kind plugin，更新 config + 產生 warnings。

---

## 8. Extension 開發 SOP

### 8.1 檔案結構

```
extensions/my-plugin/
├── package.json              ← openclaw.extensions 指向 entry
├── openclaw.plugin.json      ← id + configSchema + uiHints
├── index.ts                  ← export default { register(api) { ... } }
└── src/
    ├── channel.ts            ← ChannelPlugin 實作（頻道類）
    ├── runtime.ts            ← singleton runtime state
    ├── config.ts             ← Zod schema
    ├── tool.ts               ← TypeBox schema + execute
    ├── http.ts               ← HTTP handler
    └── *.test.ts
```

### 8.2 register() vs activate()

| 函式 | 時機 | 用途 |
|------|------|------|
| `register(api)` | 載入時立即呼叫 | 註冊 tools/hooks/channels/services/routes/CLI |
| `activate(api)` | 所有 plugin 都 register 完後 | 跨 plugin 依賴、需要完整 registry 的邏輯 |

大多數 plugin 只用 `register()`。

### 8.3 常見模式

**Singleton Runtime**：
```typescript
// runtime.ts
let runtime: PluginRuntime | null = null
export function setMyRuntime(r: PluginRuntime) { runtime = r }
export function getMyRuntime(): PluginRuntime {
  if (!runtime) throw new Error("Not initialized")
  return runtime
}

// index.ts
register(api) {
  setMyRuntime(api.runtime)
  // ...
}
```

**Closure State**（替代 singleton）：
```typescript
register(api) {
  const db = new MyDB(api.pluginConfig.dbPath)
  // tools/hooks/CLI 都透過 closure 共享 db
  api.registerTool({ execute: () => db.query(...) })
  api.on("agent_end", () => db.flush())
}
```

**Config 解析優先**：
```typescript
register(api) {
  const cfg = mySchema.parse(api.pluginConfig)  // 失敗 → 停止載入
  const resolvedPath = api.resolvePath(cfg.dataDir)
  // ...
}
```

### 8.4 工具 Schema

- **TypeBox**：用於 AI 工具 schema（模型需要 description）
- **Zod**：用於 plugin config 驗證（runtime）
- **JSON Schema**：用於 `openclaw.plugin.json` 的 `configSchema`（UI 生成）

### 8.5 Extension 類型對照

| 類型 | 主要註冊 | 範例 |
|------|---------|------|
| 頻道 | `registerChannel()` | irc, discord, telegram, googlechat |
| 工具 | `registerTool()` | memory-core, diffs |
| 記憶 | `registerTool()` + `on()` + slot | memory-lancedb |
| Auth Provider | `registerProvider()` | google-gemini-cli-auth, copilot-proxy |
| 背景服務 | `registerService()` | acpx, diagnostics-otel |
| Context Engine | `registerContextEngine()` | （自訂 context 引擎） |
| 複合型 | 多個 register 組合 | diffs（tool + HTTP + hook） |

---

## 9. 邊界條件與陷阱

### Discovery & Loading

1. **Symlink 逃逸**：plugin 路徑 realpath 超出 root → `source_escapes_root` diagnostic（warn 不阻擋）
2. **World-writable 目錄**：Linux mode & 0o002 → warn，不阻擋但 provenance 追蹤
3. **同 ID 不同路徑**：多 origin 發現同 ID plugin → 高優先 origin 勝出，warn log
4. **jiti 失敗**：plugin 有 syntax error 或 import 錯誤 → status="error"，不影響其他 plugin
5. **SDK Alias 上限**：向上搜尋最多 6 層，超過 → alias 解析失敗
6. **Discovery 快取 1000ms**：連續呼叫不重新掃描，env `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE` 可關閉

### Registration

7. **HTTP Route 衝突**：同 path 不同 auth → reject；同 auth + replaceExisting → 允許覆蓋
8. **72 保留命令名**：registerCommand 不能用 help/status/config/bash/exec/think 等
9. **Provider 重複 ID**：同 ID 第二次 register → error diagnostic
10. **Hook allowPromptInjection=false**：`before_prompt_build` 被完全阻擋，`before_agent_start` 剝除 mutation

### Slots

11. **先到先得**：slot 未在 config 指定 → 第一個載入的同 kind plugin 勝出
12. **slot=null**：整個 slot 停用，所有同 kind plugin 都 disabled
13. **kind mismatch**：manifest 宣告 kind 與 module export 不同 → warn diagnostic

### Runtime

14. **sessionKey vs sessionId**：sessionKey 穩定（跨 message），sessionId 易變（/new /reset 重設）
15. **Tool factory 每次 agent invocation 呼叫**：不是每次 tool call，是每次 agent 啟動
16. **registerService.stop() 選填**：不提供 → 無優雅關閉，資源可能洩漏

### Testing

17. **VITEST 環境預設**：`plugins.enabled: false`、memory slot: `"none"` → 測試預設不載入 plugin
18. **Config schema 快取**：schema mtime 為 cache key，修改 schema 後需重啟

### Security

19. **Provenance tracking**：plugin 不在 `plugins.load.paths` 或 `plugins.installs` → warn
20. **命令 args 上限**：4096 bytes，超過 → 不 match

---

## 10. 關鍵常量速查

| 常量 | 值 | 說明 |
|------|---|------|
| Discovery cache TTL | 1000ms | `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` |
| Manifest cache TTL | 1000ms | `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS` |
| SDK alias search depth | 6 levels | 向上掃描找 SDK 路徑 |
| Reserved command names | 72 | help/status/config/bash/exec/think... |
| Command args max | 4096 bytes | 超過不 match |
| Command name pattern | `[a-z][a-z0-9_-]*` | 小寫英數+連字號 |
| Hook priority | higher first | 數字大先執行 |
| Tool factory names option | `names: string[]` | 多工具名宣告 |
| HTTP route auth types | 2 | `"plugin"` / `"gateway"` |
| HTTP route match modes | 2 | `"exact"` / `"prefix"` |
| Slot types | 2 | `"memory"` / `"context-engine"` |
| Default memory slot | `"memory-core"` | 未設定時 |
| Default contextEngine slot | `"legacy"` | 未設定時 |
| Plugin status values | 3 | `"loaded"` / `"disabled"` / `"error"` |
| Plugin origins | 4 | `"bundled"` / `"global"` / `"workspace"` / `"config"` |
| Diagnostic levels | 2 | `"error"` / `"warn"` |
| Total hook names | 24 | 見 Hook 列表 |
| Channel adapters | ~20 | config/security/groups/outbound/directory/gateway/status |
| SDK exports | 600+ | index.ts 799 lines |
| Global registry Symbol | `"openclaw.pluginRegistryState"` | `Symbol.for()` |
| File lock Symbol | `"openclaw.fileLockHeldLocks"` | process-scoped |

---

## 11. C# 概念對照

| OpenClaw Plugin 概念 | C# 對應 |
|---------------------|---------|
| `OpenClawPluginApi` | `IServiceCollection` + `IHostBuilder`（DI container builder） |
| `registerTool()` | `services.AddTransient<ITool>()` |
| `registerHook()` / `on()` | `IMediator` pattern / event handlers |
| `registerHttpRoute()` | `app.MapGet("/path", handler)` 或 `[Route]` attribute |
| `registerChannel()` | `IHostedService` + `BackgroundService` 複合 |
| `registerProvider()` | `IAuthenticationHandler` 註冊 |
| `registerService()` | `IHostedService` |
| `registerCommand()` | `ICommand` pattern |
| `registerContextEngine()` | Strategy pattern factory registration |
| `registerCli()` | `CommandLineParser` subcommand |
| Plugin Manifest | Assembly `[Attribute]` metadata |
| Plugin Discovery | MEF `[Export]` / Assembly scanning |
| Slot System | `services.AddSingleton<IMemory>()` 唯一實例 |
| jiti Loader | `Assembly.LoadFrom()` + reflection |
| SDK Proxy | `DispatchProxy` / `Lazy<T>` |
| PluginRecord | `ServiceDescriptor`（DI descriptor） |
| PluginDiagnostic | `ILogger` structured log |
| Hook Priority | `[Order]` attribute / middleware pipeline |
| Tool Factory | `IServiceScopeFactory.CreateScope()` per-request |
| Config Schema (Zod) | `FluentValidation` / `DataAnnotations` |
| Config Schema (TypeBox) | `System.Text.Json` JsonSchema |
| Typed Hooks | `INotificationHandler<T>` (MediatR) |
| Global Registry State | `IServiceProvider` singleton |
| File Lock | `Mutex` / `SemaphoreSlim` |
| Persistent Dedupe | `IDistributedCache` |
| Keyed Async Queue | `Channel<T>` per-key |
| Webhook Guards | ASP.NET middleware pipeline |
| SSRF Policy | `SocketsHttpHandler` filter |
| ChannelPlugin adapters | Interface segregation（ISP）— 多小介面組合 |

---

## 12. 資料流圖

### Plugin 載入完整流程

```
Config Paths ──┐
               ├── Discovery ──→ Candidates[]
Workspace ─────┤                    │
               ├── (4 origins)      ▼
Global Dir ────┤              Manifest Load
               │              (openclaw.plugin.json)
Bundled Dir ───┘                    │
                                    ▼
                              Enable Decision
                              (allow/deny/entries)
                                    │
                              ┌─────┴─────┐
                              │ disabled   │ enabled
                              │ → skip     │
                              └────────────┤
                                           ▼
                                      Slot Check
                                      (memory / contextEngine)
                                           │
                                     ┌─────┴─────┐
                                     │ slot miss  │ slot hit
                                     │ → skip     │
                                     └────────────┤
                                                  ▼
                                             jiti Import
                                             (SDK aliases)
                                                  │
                                                  ▼
                                          Export Resolution
                                          (definition / factory)
                                                  │
                                                  ▼
                                          Config Validation
                                          (JSON Schema / Zod)
                                                  │
                                                  ▼
                                       Create OpenClawPluginApi
                                                  │
                                                  ▼
                                         register(api) 呼叫
                                         ┌────┬────┬────┐
                                         │    │    │    │
                                       tools hooks routes channels ...
                                         │    │    │    │
                                         └────┴────┴────┘
                                                  │
                                                  ▼
                                           PluginRegistry
                                                  │
                                                  ▼
                                        activatePluginRegistry()
                                        (globalThis Symbol)
                                                  │
                                                  ▼
                                        Global Hook Runner
```

### Extension 註冊到執行

```
Extension register()
    │
    ├── registerTool(factory) ──→ Agent Engine 每次 invocation 呼叫 factory
    │                              → factory(toolContext) → tool 實例
    │                              → LLM 決定呼叫 → execute(toolCallId, params)
    │
    ├── on("before_prompt_build") ──→ Hook Runner priority sort
    │                                  → 每次 agent 啟動前執行
    │                                  → 回傳 prependContext / systemPrompt
    │
    ├── registerHttpRoute() ──→ Gateway HTTP Pipeline
    │                            → first-match-wins
    │                            → auth 檢查 → handler(req, res)
    │
    ├── registerChannel() ──→ Gateway startAccount()
    │                          → 連線 + 收訊息 → inbound pipeline
    │                          → agent 回覆 → outbound adapter
    │
    ├── registerProvider() ──→ Config wizard
    │                           → auth flow → credentials
    │                           → model resolution
    │
    ├── registerService() ──→ 啟動時 start()
    │                         → 關閉時 stop()
    │
    └── registerCommand() ──→ /cmd 輸入
                               → match → handler(ctx)
                               → 直接回覆（繞過 LLM）
```
