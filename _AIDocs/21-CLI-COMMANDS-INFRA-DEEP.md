# CLI 命令全覽 + Infra 深入

> Phase 7-1 | 掃描範圍：`src/commands/` 223 files + `src/cli/` ~40 files + `src/infra/` 100+ files（含 outbound/ 52 files + update-*.ts 6 files + exec-*.ts 16 files）
> 更新：2026-03-13

---

## 目錄

1. [CLI 啟動流程](#1-cli-啟動流程)
2. [Command Registry + Lazy Registration](#2-command-registry--lazy-registration)
3. [CLI 命令完整分類表](#3-cli-命令完整分類表)
4. [Handler 架構模式](#4-handler-架構模式)
5. [Progress 系統](#5-progress-系統)
6. [Profile + 多實例隔離](#6-profile--多實例隔離)
7. [CLI DI 容器](#7-cli-di-容器)
8. [Outbound 投遞管線](#8-outbound-投遞管線)
9. [Delivery Queue（Write-Ahead）](#9-delivery-queuewrite-ahead)
10. [Outbound Target 解析](#10-outbound-target-解析)
11. [Update 自動更新機制](#11-update-自動更新機制)
12. [Exec Safety + Approvals](#12-exec-safety--approvals)
13. [Path + FS 安全](#13-path--fs-安全)
14. [Network + SSRF + Proxy](#14-network--ssrf--proxy)
15. [Bonjour / mDNS 服務發現](#15-bonjour--mdns-服務發現)
16. [Infra 其餘工具模組](#16-infra-其餘工具模組)
17. [跨系統整合圖](#17-跨系統整合圖)
18. [邊界條件與陷阱](#18-邊界條件與陷阱)
19. [關鍵常量速查](#19-關鍵常量速查)
20. [C# 概念對照](#20-c-概念對照)

---

## 1. CLI 啟動流程

### 完整 Bootstrap 序列（`cli/run-main.ts`）

```
runCli(argv)
  ↓ [1] normalizeWindowsArgv() → 平台 argv 正規化
  ↓ [2] parseCliProfileArgs() → 提取 --profile / --dev
  ↓ [3] applyCliProfileEnv() → 設 OPENCLAW_PROFILE, STATE_DIR, CONFIG_PATH
  ↓ [4] loadDotEnv() → 讀 .env（quiet mode）
  ↓ [5] normalizeEnv() → 驗證/正規化環境變數
  ↓ [6] shouldEnsureCliPath() → 條件式加入 PATH
  ↓ [7] assertSupportedRuntime() → Node/Bun 版本檢查
  ↓ [8] tryRouteCli(argv) → 快速路由（跳過 buildProgram）
  ↓ [9] enableConsoleCapture() → 結構化 log
  ↓ [10] buildProgram() → Commander.js CLI 樹
  ↓ [11] installUnhandledRejectionHandler()
  ↓ [12] Register primary command（lazy）
  ↓ [13] Register plugin commands
  ↓ [14] program.parseAsync(argv)
```

### Route-First 最佳化（`cli/route.ts`）

常見命令跳過 buildProgram，省 ~500ms：

```typescript
tryRouteCli(argv) →
  1. 檢查 OPENCLAW_DISABLE_ROUTE_FIRST → 跳過
  2. hasHelpOrVersion(argv) → 跳過
  3. findRoutedCommand(path) → 查路由表
  4. emitCliBanner() + ensureConfigReady()
  5. route.run(argv) → 直接執行
```

**快速路由命令**：`health`, `status`, `sessions`, `config get/unset`, `models list/status`

### Pre-Action Hooks（`cli/program/preaction.ts`）

在每個命令執行前：

```
program.hook("preAction") →
  1. Set process.title = "${cliName}-${primaryCommand}"
  2. Skip if --help/--version
  3. Emit banner（unless disabled）
  4. Parse --verbose/--debug → setVerbose()
  5. Suppress Node warnings（unless verbose）
  6. ensureConfigReady()（unless doctor/completion/secrets）
  7. Load plugins（if command in PLUGIN_REQUIRED_COMMANDS）
```

---

## 2. Command Registry + Lazy Registration

### 註冊架構（`cli/program/command-registry.ts`）

```typescript
type CoreCliEntry = {
  commands: CoreCliCommandDescriptor[];
  register: (params: CommandRegisterParams) => Promise<void> | void;
};
```

### 三層載入策略

| 層級 | 說明 | 時機 |
|------|------|------|
| **Route-First** | 快速路由表直接執行 | tryRouteCli() 階段 |
| **Primary Only** | 只載入匹配的 primary command | shouldRegisterCorePrimaryOnly() |
| **Full** | 全部命令 + plugin | --help / --version |

### Barrel Export 模式

每個命令群用 barrel 檔案 re-export 子命令：

```typescript
// src/commands/agents.ts（barrel）
export * from "./agents.bindings.js";
export * from "./agents.commands.bind.js";
export * from "./agents.commands.add.js";
export * from "./agents.commands.delete.js";
export * from "./agents.commands.identity.js";
export * from "./agents.commands.list.js";
export * from "./agents.config.js";
```

### 命令 Handler 標準簽名

```typescript
export async function commandNameCommand(
  opts: { [flagName]: string | boolean | string[] },
  runtime: RuntimeEnv = defaultRuntime,
  params?: { hasFlags?: boolean },
): Promise<void>
```

---

## 3. CLI 命令完整分類表

### 3.1 Agent 執行

| 命令 | 檔案 | LOC | 關鍵選項 | 說明 |
|------|------|-----|---------|------|
| `agent [message]` | `agent.ts` | 1,152 | `--thinking`, `--model`, `--deliver`, `--timeout`, `--json` | 核心 agent 呼叫 |
| `agent --via-gateway` | `agent-via-gateway.ts` | 5,589 | 同上 + gateway 中繼 | 透過 gateway 遠端執行 |

### 3.2 Agent 管理

| 命令 | 檔案 | LOC | 說明 |
|------|------|-----|------|
| `agents list` | `agents.commands.list.ts` | 204 | 列出所有 agent（`--bindings`, `--json`） |
| `agents add` | `agents.commands.add.ts` | 368 | 建立 agent（`--name`, `--workspace`, `--model`, `--bind`） |
| `agents bind` | `agents.commands.bind.ts` | 386 | 設定路由規則（`discord:user:123`） |
| `agents unbind` | `agents.commands.bind.ts` | 386 | 移除路由規則（`--all`） |
| `agents delete` | `agents.commands.delete.ts` | 152 | 移除 agent |
| `agents identity` | `agents.commands.identity.ts` | 280 | Agent 身份/IDENTITY.md |

### 3.3 Channel 管理

| 命令 | 檔案 | LOC | 說明 |
|------|------|-----|------|
| `channels list` | `channels/list.ts` | 205 | 列出啟用頻道（`--usage`） |
| `channels add` | `channels/add.ts` | 311 | 新增頻道整合 |
| `channels status` | `channels/status.ts` | 338 | Token/auth 狀態 |
| `channels capabilities` | `channels/capabilities.ts` | 554 | 功能矩陣 |
| `channels logs` | `channels/logs.ts` | ~100 | 訊息歷史 |
| `channels remove` | `channels/remove.ts` | ~100 | 停用頻道 |
| `channels resolve` | `channels/resolve.ts` | ~100 | Channel ID 解析 |

### 3.4 Model / Provider 管理

| 命令 | 檔案 | LOC | 說明 |
|------|------|-----|------|
| `models list` | `models/list.status-command.ts` | 686 | 列出模型（`--check`, `--probe`, `--json`） |
| `models scan` | `models/scan.ts` | 359 | 自動偵測可用模型 |
| `models list --probe` | `models/list.probe.ts` | 620 | 並行驗證 provider 連線 |

### 3.5 認證（30+ Provider Adapters）

| 命令 | 說明 |
|------|------|
| `auth-choice.apply.anthropic.ts` | Anthropic token + OAuth |
| `auth-choice.apply.openai.ts` | OpenAI API key + Codex OAuth |
| `auth-choice.apply.google-gemini.ts` | Gemini API key + CLI OAuth |
| `auth-choice.apply.openrouter.ts` | OpenRouter |
| `auth-choice.apply.ollama.ts` / `vllm.ts` | Local models |
| `auth-choice.apply.qwen.ts` / `moonshot.ts` / `minimax.ts` | China region |
| ... | 20+ more providers |

每個 adapter 模式：`auth-choice.apply.{provider}.ts` → Credentials → `models.json` → Profile

### 3.6 Configuration

| 命令 | 檔案 | LOC | 說明 |
|------|------|-----|------|
| `configure` | `configure.commands.ts` | 38 | 互動式 wizard（分區） |
| `configure channels` | `configure.channels.ts` | 123 | 頻道設定 |
| `configure daemon` | `configure.daemon.ts` | 270 | systemd/launchd |
| `configure gateway` | `configure.gateway.ts` | 353 | 埠/auth/Tailscale |
| `configure gateway-auth` | `configure.gateway-auth.ts` | ~200 | OAuth/token 模式 |
| `config get/set/unset` | `config.ts` | — | 非互動式 config CRUD |
| `config validate` | `config.ts` | — | Config schema 驗證 |

### 3.7 狀態 / 診斷

| 命令 | 檔案 | LOC | 說明 |
|------|------|-----|------|
| `status` | `status.ts` | 225 | 快速健康檢查 |
| `status --all` | `status-all.ts` | 365 | 完整系統報告（11 probes） |
| `health` | `health.ts` | 751 | 持續健康監控（JSON） |
| `doctor` | `doctor.ts` | 363 | 互動式修復 wizard |
| `sessions` | `sessions.ts` | 200+ | 對話 session 列表 |
| `sessions --cleanup` | `sessions-cleanup.ts` | 468 | 清理過期 session |

**Doctor 子模組**（各 ~300 LOC）：
- `doctor-auth.ts` — Profile/OAuth 修復
- `doctor-gateway-*.ts` — Daemon/service 健康
- `doctor-config-flow.ts` — Config 驗證/遷移（2,122 LOC 最大）
- `doctor-sandbox.ts` — Container/browser
- `doctor-state-integrity.ts` — DB 完整性
- `doctor-state-migrations.ts` — Legacy state 升級
- `doctor-workspace.ts` — Workspace 結構
- `doctor-security.ts` — 權限/secret 檢查

### 3.8 Onboarding

| 命令 | 說明 |
|------|------|
| `onboard` | 主 setup wizard（interactive / non-interactive） |
| `onboard --flow quickstart` | 快速設定 |
| `onboard --flow advanced` | 進階設定 |
| `onboard-non-interactive/` | 非互動模式（10 files：api-keys, daemon, gateway, workspace） |
| `onboarding/` | Plugin onboarding（registry, install, types） |

### 3.9 Infrastructure / System

| 命令 | 說明 |
|------|------|
| `gateway` | Gateway 管理 |
| `daemon` | Daemon start/stop/status |
| `logs` | Log 查詢 |
| `sandbox` | Container 管理（list, recreate） |
| `update` | 版本更新 |
| `cron` | Cron job CRUD |
| `hooks` | Hook 管理 |
| `webhooks` | Webhook 設定 |
| `nodes` | Node 管理 |
| `devices` | Device 管理 |
| `dns` | DNS 設定 |
| `plugins` | Plugin 管理 |
| `secrets` | Secret 管理 |
| `pairing` | Device pairing |
| `approvals` | Exec approval 管理 |
| `security` | 安全檢查 |
| `skills` | Skill 列表 |

### 3.10 Messaging

| 命令 | 說明 |
|------|------|
| `message send` | 發送訊息到頻道 |
| `message read` | 讀取訊息 |
| `message manage` | 訊息管理 |
| `memory search` | 記憶搜尋 |
| `memory reindex` | 記憶重建索引 |

### 3.11 Utility

| 命令 | 檔案 | 說明 |
|------|------|------|
| `setup` | `setup.ts` (75) | 初始化 workspace |
| `reset` | `reset.ts` (150+) | 完整 state 清理（`--scope`, `--dry-run`） |
| `uninstall` | `uninstall.ts` (~150) | 移除 daemon + cleanup |
| `dashboard` | `dashboard.ts` (~300) | WebUI 啟動 |
| `tui` | — | TUI 啟動 |
| `qr` | — | QR 碼生成 |
| `docs` | `docs.ts` (~100) | 文件連結 |
| `completion` | — | Shell completion |
| `completion-fish` | — | Fish shell completion |
| `signal install` | `signal-install.ts` (302) | Signal CLI 安裝 |
| `vllm-setup` | `vllm-setup.ts` (~200) | vLLM 本地模型 |

---

## 4. Handler 架構模式

### Pattern 1：Simple Flag Handler

```typescript
// 大部分 read-only 命令
export async function channelsListCommand(
  opts: { json?: boolean; usage?: boolean },
  runtime: RuntimeEnv,
) {
  // Load → Format → Output → Exit
}
```

### Pattern 2：Interactive Wizard + Non-Interactive Fallback

```typescript
export async function agentsAddCommand(
  opts: AgentsAddOptions,
  runtime: RuntimeEnv,
  params?: { hasFlags?: boolean },
) {
  const nonInteractive = opts.nonInteractive || params?.hasFlags;
  if (nonInteractive) {
    // 驗證所有必要 flags
  } else {
    // @clack/prompts 互動式
  }
}
```

### Pattern 3：Delegated Execution（`agent.ts`）

```
agentCommand(opts) →
  resolveSession() → 建立/載入 session
  resolveAgentRunContext() → 組裝 context
  runCliAgent() → 執行 agent
  deliverAgentCommandResult() → 投遞結果
  updateSessionStoreAfterAgentRun() → 持久化
```

### Pattern 4：Modular Diagnostic（`doctor.ts`）

```typescript
// Dispatcher 模式，每個 check 獨立
await maybeRepairAuthProfiles(runtime, prompter);
await maybeRepairGatewayDaemon(runtime, prompter);
await maybeRepairConfigSchema(runtime, prompter);
// ... 10+ 更多 check
```

### Pattern 5：Batch Operation with Results

```typescript
// agents bind, channels add — 詳細結果報告
applyAgentBindings(cfg, bindings) → {
  config: updated,
  added: [...], updated: [...],
  skipped: [...], conflicts: [...]
}
```

---

## 5. Progress 系統

### API（`cli/progress.ts`）

```typescript
type ProgressReporter = {
  setLabel(label: string): void;
  setPercent(percent: number): void;
  tick(delta?: number): void;
  done(): void;
};

createCliProgress(options): ProgressReporter
withProgress<T>(options, work): Promise<T>
withProgressTotals<T>(options, work): Promise<T>
```

### Renderer 選擇瀑布

```
supportsOscProgress() → OSC native progress bar（if TTY）
  ↓ fallback
@clack/prompts spinner（if fallback !== "line"）
  ↓ fallback
Custom ANSI progress line（if fallback === "line"）
  ↓ fallback
Throttled stdout lines（non-TTY, fallback === "log"）
  ↓ fallback
Noop（disabled / nested progress）
```

### 防巢狀

全域 `activeProgress` 計數器，巢狀時回傳 noop reporter。

---

## 6. Profile + 多實例隔離

### Profile 解析（`cli/profile.ts`）

```typescript
parseCliProfileArgs(argv) → {
  ok: boolean,
  profile: string | null,  // /^[a-z0-9][a-z0-9_-]{0,63}$/i
  argv: string[]           // 清理後的 argv
}
```

### 環境隔離

```
--profile <name> 或 --dev
  ↓
OPENCLAW_PROFILE = <name>
OPENCLAW_STATE_DIR = ~/.openclaw-<name>（default = ~/.openclaw）
OPENCLAW_CONFIG_PATH = ${STATE_DIR}/openclaw.json
--dev 特別：OPENCLAW_GATEWAY_PORT = 19001（if not set）
```

**不覆寫**已明確設定的環境變數。

---

## 7. CLI DI 容器

### CliDeps（`cli/deps.ts`）

```typescript
type CliDeps = {
  sendMessageWhatsApp: typeof sendMessageWhatsApp;
  sendMessageTelegram: typeof sendMessageTelegram;
  sendMessageDiscord: typeof sendMessageDiscord;
  sendMessageSlack: typeof sendMessageSlack;
  sendMessageSignal: typeof sendMessageSignal;
  sendMessageIMessage: typeof sendMessageIMessage;
};
```

**Lazy Loading**：每個 sender 第一次使用時 `import()` 載入，Promise 快取。

### RuntimeEnv

```typescript
type RuntimeEnv = {
  log: (msg: string) => void;
  error: (msg: string) => void;
  exit: (code: number) => void;
};
```

所有命令 handler 注入此介面，測試時 mock。

---

## 8. Outbound 投遞管線

### 核心檔案矩陣（`src/infra/outbound/` — 52 files）

| 檔案 | 行數 | 職責 |
|------|------|------|
| `deliver.ts` | 817 | 核心投遞 + queue 生命週期 |
| `message.ts` | 347 | Public API（sendMessage, sendPoll） |
| `targets.ts` | 589 | 目標解析（channel + destination） |
| `channel-adapters.ts` | — | Plugin adapter 載入 |
| `delivery-queue.ts` | — | Write-ahead 佇列 |
| `session-binding-service.ts` | 382 | Session delivery 綁定 |
| `message-action-runner.ts` | 679 | Tool-level send/poll dispatch |

### 投遞資料流

```
sendMessage() / sendPoll()
  ↓
resolveMessageChannelSelection()
  ↓
deliverOutboundPayloads(params)
  ↓ enqueueDelivery() → {id}.json（write-ahead）
  ↓
deliverOutboundPayloadsCore()
  ├─ createChannelHandler() → 載入 plugin.outbound adapter
  ├─ normalizePayloadsForChannelDelivery()
  ├─ applyMessageSendingHook()    [plugin hook: message_sending]
  ├─ handler.sendText() / sendMedia() / sendPayload()
  └─ emitMessageSent()            [plugin hook: message_sent + internal:message:sent]
  ↓
ackDelivery() → rename .json → .delivered → unlink
  或
failDelivery() → increment retryCount, update lastAttemptAt
```

### Delivery 模式

| 模式 | 說明 |
|------|------|
| `direct` | 直接呼叫 deliverOutboundPayloads() |
| `gateway` | HTTP POST 到 callGatewayLeastPrivilege() |

### Channel Adapter 介面

```typescript
plugin.outbound = {
  sendText(ctx)           → 純文字
  sendMedia(ctx, mediaUrl) → 媒體 + caption
  sendPayload(ctx, payload) → 頻道原生格式（Telegram HTML 等）
}
```

Context 包含：cfg, to, accountId, replyToId, threadId, identity, gifPlayback, silent, mediaLocalRoots

### Payload Normalization

- 跨頻道 text limit（Telegram 4096, Discord 2000, Slack 4000）
- HTML → plain-text（WhatsApp, Signal）
- Code fence 平衡（chunking 時保持 fence 閉合）

---

## 9. Delivery Queue（Write-Ahead）

### 架構（`delivery-queue.ts`）

```
路徑：~/.openclaw/state/delivery-queue/
格式：{uuid}.json（pending）→ {uuid}.delivered（ack marker）
```

### 兩階段提交

```
1. enqueueDelivery() → 寫 {id}.json 到磁碟
2. deliverOutboundPayloadsCore() → 實際發送
3a. 成功 → ackDelivery() → rename .delivered → unlink
3b. 失敗 → failDelivery() → 遞增 retryCount + lastAttemptAt
```

### 重試策略（指數退避）

| 重試 | 延遲 |
|------|------|
| 1st | 5s |
| 2nd | 25s |
| 3rd | 2min |
| 4th | 10min |
| 5th | MAX_RETRIES → 移至 `failed/` |

### 永久錯誤識別

Pattern 比對：`"user not found"`, `"bot blocked"`, `"forbidden"` → 不重試，直接 fail。

### Gateway 啟動恢復

```
recoverPendingDeliveries() →
  1. loadPendingDeliveries() → 讀 {id}.json files
  2. 清理 stale .delivered markers
  3. Sort by enqueuedAt（FIFO）
  4. For each:
     - retryCount >= MAX_RETRIES → moveToFailed()
     - backoff 未到期 → defer
     - deliver() with skipQueue=true
  5. Return RecoverySummary {recovered, failed, skippedMaxRetries, deferredBackoff}
```

**Timeout budget**：恢復期 60s 上限。

---

## 10. Outbound Target 解析

### 核心函式（`targets.ts` — 589 lines）

```typescript
resolveOutboundTarget(params) → {
  channel: ChannelId,
  to: string,
  accountId?: string,
  mode: "explicit" | "implicit"
}

resolveSessionDeliveryTarget(params) → {
  channel, to, accountId,
  source: "explicit" | "session",
  // Telegram :topic:NNN 解析
}
```

### 解析優先序

```
1. 顯式指定（user --channel --to）
2. Session lastChannel（上次互動的頻道）
3. turnSourceChannel override（防跨頻道 reroute）
4. Fallback chain
```

### Session Binding Service（382 lines）

- Per-session delivery target 狀態
- 記錄 last channel/to/accountId
- Race protection（多頻道 session 競爭）

---

## 11. Update 自動更新機制

### 核心檔案

| 檔案 | 行數 | 職責 |
|------|------|------|
| `update-check.ts` | 490 | 版本偵測（Git/Package/Registry） |
| `update-runner.ts` | 927 | 更新執行器 |
| `update-channels.ts` | — | Channel → npm tag 映射 |
| `update-global.ts` | — | Global npm/pnpm/bun install |
| `update-startup.ts` | — | 啟動時檢查 |

### Version Check 流程（`update-check.ts`）

```
1. Installation 偵測
   - Git: git rev-parse → package.json version
   - Package: 偵測 lockfile（package-lock.json / bun.lockb / pnpm-lock.yaml）

2. Git Status
   - SHA, branch, upstream, ahead/behind
   - dirty working tree
   - Optional fetch with timeout

3. Deps Status
   - lockfile mtime vs install marker

4. Registry Status
   - fetch latest from npm registry
   - npm channel tag: latest / beta / @next
   - Semver 比較
```

### Update Runner 流程（`update-runner.ts`）

**Git mode（dev channel）：**

```
1. Clean check（git status）
2. git fetch --all
3. Worktree preflight（up to 10 commits）:
   - Build + lint on candidate
   - Select first good commit
4. Checkout tag/branch
5. pnpm install
6. pnpm build
7. ui:build
8. Doctor run（config auto-fix）
9. UI assets repair
```

**Package mode（stable/beta）：**

```
1. 偵測 global install（npm/pnpm/bun）
2. globalInstallArgs() → 建構安裝指令
3. 執行安裝
4. 失敗 → fallback（--omit=optional）
```

### Channel 映射

| Channel | npm tag | Branch |
|---------|---------|--------|
| stable | `latest` | detached（tag） |
| beta | `beta` | detached（tag） |
| dev | — | `main` |

### Progress Callbacks

```typescript
onStepStart(step: string): void;
onStepComplete(step: string, ok: boolean): void;
```

**Timeout**：20 min default，per-step configurable。

---

## 12. Exec Safety + Approvals

### 檔案矩陣（16 files）

| 檔案 | 職責 |
|------|------|
| `exec-safety.ts` (45) | Shell argument 安全驗證 |
| `exec-approvals.ts` (200+) | Approval config + schema |
| `exec-approvals-analysis.ts` | 混淆偵測 + payload 分析 |
| `exec-approvals-allowlist.ts` | Pattern 比對 |
| `exec-approval-forwarder.ts` | Async socket 請求 |

### Safety Check（`exec-safety.ts`）

```typescript
isSafeExecutableValue(value) →
  Rejects: null bytes, control chars, ; & | ` $ < > " '
  Rejects: leading - (flags)
  Allows: paths (/ \ . prefix, drive letters)
  Allows: bare names (alphanumeric + . _ + -)
```

### Approvals 架構

```
User command → Agent tool call
  ↓
loadExecApprovalsSnapshot()
  ↓
analyzeCommand()
  ├─ 混淆偵測（base64, hex encoding, escaping）
  ├─ env hash（SHA256 mutation detection）
  └─ mutable file operand tracking
  ↓
matchAllowlist()
  ├─ Case-insensitive glob + regex
  └─ Entry metadata: lastUsedAt, lastUsedCommand, lastResolvedPath
  ↓
Decision: allow → 執行
         deny → 拒絕
         not matched + ask=on-miss → requestExecApproval()
           → JSON-L over Unix socket / TCP
           → timeout: 120s
           → response: {decision, resolvedBy, ts}
```

### Approval Config（`~/.openclaw/exec-approvals.json`）

```json
{
  "version": 1,
  "defaults": {
    "security": "deny" | "allowlist" | "full",
    "ask": "off" | "on-miss" | "always",
    "askFallback": "deny" | "allowlist" | "full",
    "autoAllowSkills": false
  },
  "agents": {
    "agent-id": {
      "security": "...",
      "allowlist": [{
        "id": "...",
        "pattern": "python|pip|npm",
        "lastUsedAt": 1234567890,
        "lastUsedCommand": "pip install ...",
        "lastResolvedPath": "/usr/bin/pip"
      }]
    }
  }
}
```

**Defaults**：security=`"deny"`, ask=`"on-miss"`, askFallback=`"deny"`

---

## 13. Path + FS 安全

### Path Safety（`path-safety.ts`, `path-guards.ts`）

- **Symlink 阻擋**：`O_NOFOLLOW` flag
- **Boundary check**：路徑必須在 root 目錄內
- **Alias guard**：防 symlink chaining 逃逸
- **Post-write 驗證**：realpath 確認未被重導向

### FS Safe（`fs-safe.ts` — 637 lines）

```
openFileWithinRoot(root, path) →
  1. Pre-open: lstat（reject dir, symlink）
  2. Open: O_RDONLY | O_NOFOLLOW
  3. Post-open: stat + lstat match, realpath within root
  4. Hardlink check: nlink > 1 → reject
  5. Max file size enforcement
```

**Atomic Write**：
```
1. Write to temp file（pid + UUID suffix）
2. Rename to target（POSIX atomic）
3. Post-write verification（realpath, boundaries, identity）
4. Cleanup temp on failure
```

**Error 類型**：`"invalid-path"`, `"not-found"`, `"outside-workspace"`, `"symlink"`, `"not-file"`, `"path-mismatch"`, `"too-large"`

### File Lock

```typescript
acquireFileLock(path): Promise<FileLock>
withFileLock(path, fn): Promise<T>
// re-exported from plugin-sdk/file-lock.js
```

---

## 14. Network + SSRF + Proxy

### SSRF 防護（`infra/net/ssrf.ts`）

```typescript
isBlockedHostnameOrIp(host) →
  封鎖：RFC 1918 / loopback / link-local / multicast / metadata endpoints
  DNS pinning 防 rebinding
  fetchWithSsrfGuard(url) → redirect 驗證（max 3, cross-origin strip auth）
```

### Proxy 支援（`infra/net/proxy-env.ts`, `proxy-fetch.ts`）

- 讀取 `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`
- Undici global dispatcher 設定
- 所有 fetch() 自動走 proxy

---

## 15. Bonjour / mDNS 服務發現

### 架構（`infra/bonjour.ts`, `bonjour-*.ts`）

**Framework**：@homebridge/ciao（service responder）

**Service Types**：
- `"openclaw-gw"` — Gateway 發現
- `"openclaw-cli"` — CLI 發現

**TXT Records**：
- Core：role, gatewayPort, lanHost, displayName
- Optional：gatewayTls, gatewayTlsSha256, canvasPort, tailnetDns
- Minimal mode：省略 cliPath, sshPort（安全）

**Instance 命名**：`OPENCLAW_MDNS_HOSTNAME` env 或 `"openclaw"`

**停用**：`OPENCLAW_DISABLE_BONJOUR` env 或 `NODE_ENV=test`

---

## 16. Infra 其餘工具模組

### 環境

| 模組 | 說明 |
|------|------|
| `env.ts` | env var 正規化 + redacted logging |
| `dotenv.ts` | .env 載入（cwd → ~/.openclaw/.env fallback） |

### 網路 / Port

| 模組 | 說明 |
|------|------|
| `ports.ts` | ensurePortAvailable() + describePortOwner() |
| `net/` (6 files) | SSRF + proxy + Undici dispatcher |
| `tls/` | TLS fingerprinting + gateway TLS 管理 |

### 核心工具

| 模組 | 說明 |
|------|------|
| `retry.ts` + `retry-policy.ts` | 指數退避 + jitter（Discord/Telegram 用） |
| `backoff.ts` | 泛用退避計算（initialMs, maxMs, factor, jitter） |
| `fetch.ts` | fetch() wrapper + timeout |
| `json-file.ts`, `json-files.ts` | Safe JSON read/write + error handling |
| `git-root.ts`, `git-commit.ts` | Git 操作（detect root, create commits） |
| `archive.ts`, `archive-path.ts` | Zip/archive handling |
| `home-dir.ts` | `~` path 展開 |
| `machine-name.ts` | 平台特定 machine ID |
| `dedupe.ts` | Array 去重 |
| `plainobject.ts` | Plain object type guard |
| `abort-signal.ts` | AbortController tracking + cleanup |
| `process-respawn.ts` | Zero-downtime restart sentinel |

### 追蹤 / 狀態

| 模組 | 說明 |
|------|------|
| `provider-usage.ts` | API 用量追蹤（Claude, Gemini, Copilot 等） |
| `channel-activity.ts` | Per-channel 近期活動追蹤 |
| `agent-events.ts` | Channel event subscriptions |
| `device-pairing.ts` | Device identity + pairing state |
| `device-auth-store.ts` | Device auth 持久化 |

### Format

| 模組 | 說明 |
|------|------|
| `format-time/` | 時間格式化 utilities |
| `cli-root-options.ts` | 解析 --port, --token CLI args |

---

## 17. 跨系統整合圖

```
┌─────────────────────────────────────────────────┐
│                  CLI Entry                       │
│  entry.ts → runCli() → tryRouteCli() / buildProgram()  │
└──────────┬────────────────────────────┬─────────┘
           │                            │
     Route-First                  Commander.js
     (fast path)                  (full build)
           │                            │
           ▼                            ▼
┌──────────────┐              ┌──────────────────┐
│  Routed Cmd  │              │  Lazy-Registered │
│  (health,    │              │  Commands        │
│   status,    │              │  (223 files)     │
│   config get)│              │                  │
└──────┬───────┘              └────────┬─────────┘
       │                               │
       └───────────┬───────────────────┘
                   │
                   ▼
         ┌─────────────────┐
         │  Handler Execute │
         │  (RuntimeEnv DI) │
         └────────┬────────┘
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
┌────────┐  ┌──────────┐  ┌─────────┐
│ Agent  │  │ Channels │  │ Config  │
│ Exec   │  │ Manage   │  │ Doctor  │
└───┬────┘  └────┬─────┘  └────┬────┘
    │            │              │
    ▼            ▼              ▼
┌───────────────────────────────────┐
│         Infra Layer               │
├──────────┬──────────┬─────────────┤
│ Outbound │ Update   │ Exec Safety │
│ Delivery │ Checker  │ Approvals   │
│ Queue    │ Runner   │ Allowlist   │
├──────────┼──────────┼─────────────┤
│ Path/FS  │ Network  │ Bonjour     │
│ Safety   │ SSRF     │ mDNS        │
│ Atomic   │ Proxy    │ Discovery   │
└──────────┴──────────┴─────────────┘
```

### 關鍵交互路徑

| 觸發者 | 目標 | 路徑 |
|--------|------|------|
| `agent` command | Outbound Delivery | resolveSession → runAgent → deliverResult |
| `message send` | Outbound Delivery | sendMessage → deliverOutboundPayloads |
| `update` command | Update Runner | check → runner → global install / git rebase |
| Agent tool call | Exec Approvals | analyzeCommand → matchAllowlist → forwarder |
| Gateway startup | Delivery Recovery | recoverPendingDeliveries → retry eligible |
| Any file I/O | FS Safety | openFileWithinRoot → symlink check → boundary verify |
| Config reload | Bonjour | 更新 TXT records → re-announce |

---

## 18. 邊界條件與陷阱

1. **Route-First 與 plugin 命令**：Route-First 跳過 buildProgram，plugin 命令不在快速路由表 → 必須走 full build。

2. **Lazy Registration race**：命令首次執行時動態載入完整實作 → 重新解析 argv。如果模組 export 名不匹配 → silent no-op。

3. **Profile 環境變數優先**：`applyCliProfileEnv()` 不覆寫已設定的 env → 如果 `OPENCLAW_STATE_DIR` 已設定，`--profile` 不會改變它。

4. **Progress 巢狀**：全域 activeProgress 計數器防巢狀 → 內層回傳 noop reporter → 如果外層 done() 過早，內層永遠 silent。

5. **Write-ahead queue 磁碟空間**：每條訊息一個 JSON 檔案。大量失敗訊息累積 → 磁碟消耗。`MAX_RETRIES=5` 後移至 `failed/` 但不自動清理。

6. **Permanent error 辨識**：靠 string pattern 比對（`"user not found"` 等）→ 如果 API 改錯誤訊息格式，可能誤分類為 transient。

7. **Recovery timeout budget 60s**：啟動恢復期間只有 60s → 大量 pending 訊息時可能來不及全部重試。

8. **Exec approval socket timeout 120s**：如果使用者離開未回應 → 120s 後按 askFallback 處理（default: deny）。

9. **Exec safety Windows CVE-2024-27980**：`.cmd/.bat` 不直接 spawn → npm/npx resolve 為 `node.exe <npm-cli.js>`。`shell: false` 永遠不開。

10. **FS Safe hardlink 偵測**：`nlink > 1` → reject。但某些檔案系統（如 ZFS snapshots）正常檔案也有 nlink > 1 → false positive。

11. **Atomic write temp file 殘留**：寫入失敗時清理 temp，但如果 process crash → temp 可能殘留。無自動 gc。

12. **SSRF DNS rebinding**：fetchWithSsrfGuard 做 DNS pinning，但 redirect 時重新解析 → max 3 redirects + cross-origin strip auth 緩解。

13. **Bonjour instance naming**：`OPENCLAW_MDNS_HOSTNAME` env → 多實例同名會衝突。ciao 自動 rename 但可能造成 discovery 混亂。

14. **Update preflight worktree**：dev channel 最多測 10 commits → 如果 10 commit 都失敗，更新整個跳過。

15. **Update --omit=optional fallback**：第一次 global install 失敗 → fallback 省略 optional deps → 可能遺漏 native addon（如 sqlite-vec）。

16. **Target 解析 "last" 魔法**：`channel="last"` 用 session 上次頻道 → session 無歷史時 fallback 為 undefined → delivery 可能失敗。

17. **Delivery Queue FIFO vs backoff**：Recovery 照 enqueuedAt 排序 → backoff 未到期的被 defer → 實際執行順序可能亂序。

18. **Doctor config-flow 2122 行**：最大 command 檔案 → 遷移邏輯複雜，新 config version 容易漏 migration path。

---

## 19. 關鍵常量速查

| 常量 | 值 | 位置 |
|------|---|------|
| Route-First 節省 | ~500ms | cli/route.ts |
| Profile name pattern | `/^[a-z0-9][a-z0-9_-]{0,63}$/i` | cli/profile.ts |
| Dev gateway port | 19001 | cli/profile.ts |
| Progress nesting guard | global counter | cli/progress.ts |
| OSC progress support | terminal capability | cli/progress.ts |
| Delivery retry 1 | 5s | infra/outbound/delivery-queue.ts |
| Delivery retry 2 | 25s | infra/outbound/delivery-queue.ts |
| Delivery retry 3 | 2min | infra/outbound/delivery-queue.ts |
| Delivery retry 4 | 10min | infra/outbound/delivery-queue.ts |
| Delivery MAX_RETRIES | 5 | infra/outbound/delivery-queue.ts |
| Recovery timeout budget | 60s | infra/outbound/delivery-queue.ts |
| Exec approval timeout | 120s | infra/exec-approvals.ts |
| Exec approval default security | "deny" | infra/exec-approvals.ts |
| Exec approval default ask | "on-miss" | infra/exec-approvals.ts |
| FS Safe max nlink | 1 (hardlink reject) | infra/fs-safe.ts |
| SSRF max redirects | 3 | infra/net/ssrf.ts |
| Bonjour service type GW | "openclaw-gw" | infra/bonjour.ts |
| Bonjour service type CLI | "openclaw-cli" | infra/bonjour.ts |
| Update timeout | 20min | infra/update-runner.ts |
| Update preflight max commits | 10 | infra/update-runner.ts |
| Telegram text limit | 4096 chars | outbound/deliver.ts |
| Discord text limit | 2000 chars | outbound/deliver.ts |
| Slack text limit | 4000 chars | outbound/deliver.ts |
| Commands total files | 223 production TS | src/commands/ |
| Doctor config flow | 2,122 LOC | commands/doctor-config-flow.ts |
| Agent command | 1,152 LOC | commands/agent.ts |

---

## 20. C# 概念對照

| OpenClaw (TS) | C# / .NET 對照 |
|---------------|---------------|
| `Commander.js` program tree | `System.CommandLine` CommandHandler tree |
| Route-First tryRouteCli() | 手動 `if (args[0] == "status")` fast path |
| Lazy registration + barrel export | `Assembly.LoadFrom()` + MEF lazy activation |
| `RuntimeEnv` DI | `IConsole` / `IHost` abstraction |
| `CliDeps` message sender inject | `IServiceCollection.AddScoped<ISender>()` |
| `@clack/prompts` interactive | `Spectre.Console` prompts |
| OSC progress bar | `AnsiConsole.Progress()` |
| Profile 多實例隔離 | `IHostEnvironment.EnvironmentName` + separate appsettings |
| Write-ahead delivery queue | `IBackgroundTaskQueue` + persistent file queue |
| Delivery retry backoff | Polly `WaitAndRetry` policy |
| Permanent error pattern match | Exception type hierarchy + `PolicyResult.FinalException` |
| Recovery on startup | `IHostedService.StartAsync()` + scan pending tasks |
| `exec-approvals.json` allowlist | Process `StartInfo.FileName` whitelist + admin consent |
| Approval socket forwarder | Named pipe (`NamedPipeServerStream`) |
| `isSafeExecutableValue()` | `ProcessStartInfo { UseShellExecute = false }` + manual validation |
| `openFileWithinRoot()` O_NOFOLLOW | `FileStream` + `FileAttributes.ReparsePoint` check |
| Atomic write (temp + rename) | `File.Move(src, dst, overwrite: true)` |
| `acquireFileLock()` | `FileStream(path, FileMode.Open, FileAccess.ReadWrite, FileShare.None)` |
| SSRF guard fetchWithSsrfGuard | `HttpClient` + custom `HttpMessageHandler` with IP validation |
| `@homebridge/ciao` mDNS | `Makaretu.Dns.Multicast` or `Zeroconf` |
| `update-runner.ts` Git mode | `LibGit2Sharp` fetch/checkout + MSBuild |
| `update-global.ts` npm install | `dotnet tool update -g` |
| `provider-usage.ts` quota tracking | `IDistributedCache` + rate-limit middleware |
| `channel-activity.ts` tracking | `ConcurrentDictionary<channel, ActivityWindow>` |
