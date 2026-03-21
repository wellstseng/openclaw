# Hook + Cron + Process + Heartbeat 深入

> Phase 2-2 | 掃描範圍：`src/hooks/` 43 files + `src/cron/` ~8,800 lines + `src/process/` 25 files + `src/infra/heartbeat-*` 9 files
> 更新：2026-03-13

---

## 目錄

1. [Hook 系統架構](#1-hook-系統架構)
2. [Hook 生命週期：Discovery → Load → Register → Trigger](#2-hook-生命週期discovery--load--register--trigger)
3. [Hook Mapping 引擎](#3-hook-mapping-引擎)
4. [Gmail Watcher](#4-gmail-watcher)
5. [Bundled Hooks 實作](#5-bundled-hooks-實作)
6. [CronService 架構](#6-cronservice-架構)
7. [排程引擎核心（Timer + Schedule）](#7-排程引擎核心timer--schedule)
8. [Isolated Agent 執行](#8-isolated-agent-執行)
9. [Cron Delivery 系統](#9-cron-delivery-系統)
10. [Cron 持久化與 Run Log](#10-cron-持久化與-run-log)
11. [Process / CommandLane 系統](#11-process--commandlane-系統)
12. [ProcessSupervisor + exec 安全](#12-processsupervisor--exec-安全)
13. [Heartbeat Runner 引擎](#13-heartbeat-runner-引擎)
14. [Heartbeat Wake 機制](#14-heartbeat-wake-機制)
15. [System Event Bus](#15-system-event-bus)
16. [Gateway Restart 邏輯](#16-gateway-restart-邏輯)
17. [跨系統整合圖](#17-跨系統整合圖)
18. [邊界條件與陷阱](#18-邊界條件與陷阱)
19. [關鍵常量速查](#19-關鍵常量速查)
20. [C# 概念對照](#20-c-概念對照)

---

## 1. Hook 系統架構

### 核心檔案矩陣

| 檔案 | 行數 | 職責 |
|------|------|------|
| `hooks/types.ts` | 67 | Hook, HookEntry, OpenClawHookMetadata 型別 |
| `hooks/config.ts` | 85 | Runtime eligibility 評估（shouldIncludeHook） |
| `hooks/loader.ts` | 210 | Hook 動態載入 + 註冊（loadInternalHooks） |
| `hooks/internal-hooks.ts` | 422 | Event 註冊/分派系統（registerInternalHook, triggerInternalHook） |
| `hooks/workspace.ts` | 381 | 磁碟掃描發現 hooks（loadWorkspaceHookEntries） |
| `hooks/frontmatter.ts` | 82 | HOOK.md metadata 解析（resolveOpenClawMetadata） |
| `hooks/message-hook-mappers.ts` | 274 | Message canonical context 轉換 |
| `hooks/gmail-watcher.ts` | 247 | gog 子程序管理 |
| `hooks/gmail.ts` | 272 | Gmail config 解析 + gog 指令建構 |
| `hooks/gmail-ops.ts` | 374 | Gmail interactive setup + service running |
| `gateway/hooks-mapping.ts` | 527 | Webhook payload → agent action mapping |

### Hook 來源優先序

```
Extra dirs (config: hooks.internal.load.extraDirs)
  ↓
Bundled hooks（隨 app 發佈，不可修改）
  ↓
Managed hooks（~/.openclaw/hooks/）
  ↓
Workspace hooks（project/.../hooks/）
```

### 5 種 Event 類型

```typescript
type InternalHookEventType = "command" | "session" | "agent" | "gateway" | "message";
```

| Event | 常見 Action | 觸發時機 |
|-------|------------|---------|
| `command` | new, reset, stop | 使用者下 /new, /reset, /stop 指令 |
| `agent` | bootstrap | Agent 初始化前 |
| `gateway` | startup | Gateway 啟動 250ms 後（async） |
| `message` | received, sent, transcribed, preprocessed | 訊息接收/傳送/語音轉文字/預處理 |
| `session` | (保留) | 未來擴充 |

---

## 2. Hook 生命週期：Discovery → Load → Register → Trigger

### Phase 1: Discovery（`workspace.ts`）

遞迴掃描目錄，找含 `HOOK.md` 的子目錄。支援 npm `package.json` manifest。

### Phase 2: Parsing（`frontmatter.ts`）

解析 HOOK.md YAML frontmatter：

```yaml
---
name: session-memory
openclaw:
  events: ["command:new", "command:reset"]
  export: default
  requires:
    bins: ["node"]
    config: ["workspace.dir"]
  os: ["linux", "darwin", "win32"]
---
```

### Phase 3: Filtering（`config.ts` / `shouldIncludeHook()`）

```
1. OS 相容性 → os[] 包含 process.platform
2. Binary 需求 → bins[] 全在 PATH；anyBins[] 任一在 PATH
3. Config path 需求 → config 相應路徑有值
4. 明確設定 → hooks.internal.entries[hookKey].enabled
```

### Phase 4: Registration（`loader.ts` / `loadInternalHooks()`）

```
for each eligible hook:
  1. 驗證 handler path 在 hook 目錄內（boundary check 防 symlink escape）
  2. 建 cache-busted import URL:
     - Bundled → 無 bust（immutable between installs）
     - User/managed → ?t={mtime}&s={size}
  3. dynamic import(url) → 取 export function
  4. registerInternalHook(eventKey, handler)
```

### Phase 5: Dispatch（`internal-hooks.ts`）

```typescript
// 全域單例 Map
const handlers = (globalThis.__openclaw_internal_hook_handlers__ ??= new Map());

// 註冊：type 或 type:action
registerInternalHook("command", handler);      // 所有 command
registerInternalHook("command:new", handler);   // 只有 /new

// 觸發
triggerInternalHook(event) →
  1. 查 type handler[]
  2. 查 type:action handler[]
  3. 依序呼叫，error caught & logged，不阻塞其他 handler
```

### Hook Event 資料結構

```typescript
interface InternalHookEvent {
  type: InternalHookEventType;
  action: string;
  sessionKey: string;
  context: Record<string, unknown>;
  timestamp: Date;
  messages: string[];  // hooks 可 push 使用者可見訊息
}
```

---

## 3. Hook Mapping 引擎

**用途**：將外部 webhook payload 轉換為 wake/agent 內部 action。

### Mapping 解析流程（`gateway/hooks-mapping.ts`）

```
POST /hooks/{path}
  ↓
1. hooksConfig.mappings[] 逐筆匹配
   - match.path 比對 URL path suffix
   - match.method (optional) 比對 HTTP method
   - match.headers (optional) 比對 headers
  ↓
2. 找到 mapping →
   a. 解析 payload → extractBodyPayload(req)
   b. 渲染 template 變數: {{path}}, {{now}}, {{headers.X}}, {{query.key}}, {{payload.field}}
   c. 選擇性載入 transform module（~/.openclaw/hooks/transforms/*.ts）
   d. 覆寫 action 欄位（transform 回傳 partial override 或 null 跳過）
  ↓
3. action = "wake" → dispatchWakeHook()
   action = "agent" → dispatchAgentHook()
```

### Gmail Preset Mapping

```typescript
{
  id: "gmail",
  match: { path: "gmail" },
  action: "agent",
  wakeMode: "now",
  name: "Gmail",
  sessionKey: "hook:gmail:{{messages[0].id}}",
  messageTemplate: "New email from {{messages[0].from}}\nSubject: ..."
}
```

### Template 安全

- `__proto__`, `prototype`, `constructor` 被 allowlist 阻擋
- Transform module 限制在 `~/.openclaw/hooks/transforms/` 目錄

---

## 4. Gmail Watcher

### 架構：Google Pub/Sub + gog CLI push

```
Gmail API → Google Pub/Sub → gog serve (local) → HTTP POST → Gateway /hooks/gmail
```

### 三階段

1. **Setup**（`gmail-ops.ts`）
   - 確認 gcloud, gog, tailscale CLI 已安裝
   - 建立 GCP Pub/Sub topic + subscription（push mode）
   - 註冊 Gmail API Watch endpoint
   - 設定 Tailscale funnel/serve 穿 NAT

2. **Watch Registration**（`gmail-watcher.ts`）
   - `gog gmail watch start --account X --label INBOX --topic projects/ID/topics/NAME`
   - 每 12 小時更新（configurable renewEveryMinutes）
   - 120s timeout for setup phase

3. **Serve Process**
   - 長駐：`gog gmail watch serve --bind 127.0.0.1 --port 8788 --path /gmail-pubsub --hook-url <gateway> --hook-token <token>`
   - Crash → 5s backoff auto-restart（address-in-use 除外 → 不重啟）
   - SIGTERM/SIGKILL graceful 終止

### Config

```yaml
hooks:
  gmail:
    account: user@gmail.com
    label: INBOX
    renewEveryMinutes: 720  # 12h
    serve:
      bind: 127.0.0.1
      port: 8788
    tailscale:
      mode: "funnel" | "serve" | "off"
```

---

## 5. Bundled Hooks 實作

每個 hook 目錄含 `HOOK.md`（metadata）+ `handler.ts`（default export）。

### session-memory（334 行）

| 項目 | 內容 |
|------|------|
| Events | `command:new`, `command:reset` |
| 功能 | 讀取前一 session transcript，生成 slug（可選 LLM），寫入 `<workspace>/memory/YYYY-MM-DD-slug.md` |
| Config | `entries["session-memory"].messages`（default 15）, `llmSlug` |

### boot-md（44 行）

| 項目 | 內容 |
|------|------|
| Events | `gateway:startup` |
| 功能 | 對每個 agent 執行 workspace 內的 `BOOT.md` |

### bootstrap-extra-files（73 行）

| 項目 | 內容 |
|------|------|
| Events | `agent:bootstrap` |
| 功能 | 以 glob pattern 載入額外 bootstrap 檔案 |

### command-logger（68 行）

| 項目 | 內容 |
|------|------|
| Events | `command`（全部 command event） |
| 功能 | 每行 JSON append 至 `~/.openclaw/logs/commands.log` |

---

## 6. CronService 架構

### 核心檔案矩陣

| 檔案 | 行數 | 職責 |
|------|------|------|
| `cron/service.ts` | 56 | Public API facade |
| `cron/service/timer.ts` | 1218 | 排程引擎核心 + job 執行協調 |
| `cron/service/jobs.ts` | 900 | Job 生命週期 + schedule 計算 |
| `cron/service/store.ts` | 503 | JSON 持久化 + migration |
| `cron/service/ops.ts` | 473 | CRUD 操作 |
| `cron/service/state.ts` | 156 | State 定義 + 依賴注入 |
| `cron/isolated-agent/run.ts` | 864 | Isolated cron agent 執行器 |
| `cron/isolated-agent/delivery-dispatch.ts` | 553 | Delivery routing + webhook/announce |
| `cron/isolated-agent/delivery-target.ts` | 201 | Delivery 目標解析 |
| `cron/isolated-agent/subagent-followup.ts` | 192 | Subagent 結果聚合 |
| `cron/normalize.ts` | 506 | 輸入正規化 + 驗證 |
| `cron/schedule.ts` | 170 | Cron expression 解析（croner library） |
| `cron/run-log.ts` | 454 | JSONL run log 持久化 |
| `cron/delivery.ts` | 301 | Delivery plan 解析 |
| `cron/session-reaper.ts` | 152 | 暫態 session 清理 |
| `cron/types.ts` | 157 | CronJob, CronSchedule, CronDelivery 型別 |

### CronService Public API

```typescript
CronService {
  start()              // 啟動排程
  stop()               // 停止排程
  status()             // 狀態快照
  list(opts)           // 列出 jobs
  add(input)           // 新增 job
  update(id, patch)    // 更新 job
  remove(id)           // 移除 job
  run(id, mode)        // 手動觸發
  getJob(id)           // 查詢單筆
  wake(opts)           // 喚醒
}
```

### CronJob 完整資料結構

```typescript
CronJob {
  id: string;           // UUID
  name: string;
  description?: string;
  enabled: boolean;
  deleteAfterRun?: boolean;
  agentId?: string;
  sessionKey?: string;
  createdAtMs: number;
  updatedAtMs: number;

  schedule: CronSchedule;
  sessionTarget: "main" | "isolated";
  wakeMode: "next-heartbeat" | "now";
  payload: CronPayload;
  delivery?: CronDelivery;
  failureAlert?: CronFailureAlert | false;

  state: {
    nextRunAtMs?: number;
    runningAtMs?: number;
    lastRunAtMs?: number;
    lastRunStatus?: "ok" | "error" | "skipped";
    lastError?: string;
    lastDurationMs?: number;
    consecutiveErrors?: number;
    lastFailureAlertAtMs?: number;
    scheduleErrorCount?: number;
    lastDeliveryStatus?: CronDeliveryStatus;
    lastDeliveryError?: string;
  }
}
```

### CronSchedule 三種類型

```typescript
CronSchedule =
  | { kind: "at"; at: string }                          // 一次性 ISO/epoch
  | { kind: "every"; everyMs: number; anchorMs?: number } // 週期
  | { kind: "cron"; expr: string; tz?: string; staggerMs?: number } // Cron expression
```

### CronPayload 兩種類型

```typescript
CronPayload =
  | { kind: "systemEvent"; text: string }    // 主 session 系統事件
  | { kind: "agentTurn";                     // 獨立 agent 回合
      message: string;
      model?: string;
      thinking?: string;
      timeoutSeconds?: number;
      lightContext?: boolean;
      deliver?: boolean;
      channel?: ChannelId | "last";
      to?: string;
      bestEffortDeliver?: boolean;
      fallbacks?: string[];
    }
```

---

## 7. 排程引擎核心（Timer + Schedule）

### Timer 主循環（`service/timer.ts` onTimer()）

```
onTimer() fires →
  1. state.running = true（防重入）
  ↓
  2. 加鎖 locked():
     a. forceReload store from disk
     b. collectRunnableJobs(nowMs) → 找 enabled + 未 running + now ≥ nextRunAtMs
     c. 標記 job.state.runningAtMs = now
     d. persist store
  ↓
  3. 並行執行（worker pool + shared cursor）:
     concurrency = min(maxConcurrentRuns, dueJobCount)  // default: 1
     for each worker:
       cursor++ → executeJobCoreWithTimeout() →
         - race: runIsolatedAgentJob() vs Promise.timeout(AbortController)
  ↓
  4. 加鎖 locked():
     a. forceReload store
     b. applyOutcomeToStoredJob() for each result
     c. maintenance-only recompute（避免跳過 past-due）
     d. persist store
  ↓
  5. finally:
     - Session reaper sweep（5 min throttle, 在 lock 外避免 deadlock）
     - state.running = false
     - armTimer() → setTimeout(nextDue - now, clamp [2s, 60s])
```

### Schedule 計算（`service/jobs.ts`）

| Kind | 算法 |
|------|------|
| `at` | parse ISO/epoch → 大於 now 就跑，否則 undefined（一次性，跑完 disable） |
| `every` | `anchor + ceil((now - anchor) / everyMs) * everyMs`，anchor 缺省 createdAtMs |
| `cron` | croner library（512-entry LRU cache per tz/expr），workaround croner year-rollback bug |

### Stagger 防雷暴（`stagger.ts`）

- 自動套用到 `"0 * * * *"` 類 top-of-hour 排程
- 確定性偏移 = `SHA256(jobId)[0:4] % staggerMs`（default 5 min window）
- 穩定：同 jobId 永遠同偏移

### Job Result 處理（`applyJobResult()`）

**循環 job：**
- ok → 正常計算 next + MIN_REFIRE_GAP_MS 安全網
- error → backoff 加到 nextRunAtMs（max 與自然排程）

**一次性 "at" job：**
- ok/skipped → disable, 清 nextRunAtMs
- transient error → 指數退避重試 (30s → 1m → 5m → 15m → 60m)
- permanent error / max retries → disable + log warning

### Backoff 梯度

```
1st error:  30s
2nd error:  1min
3rd error:  5min
4th error:  15min
5th+ error: 60min (constant)
```

---

## 8. Isolated Agent 執行

### 入口：`runCronIsolatedAgentTurn(params)`

```
1. Agent Resolution
   - 依序: params.agentId → job.agentId → default
   - 載入 agent config，合併 defaults

2. Model Selection（優先序）
   ① job payload.model（顯式覆寫）
   ② session modelOverride（/model 指令）
   ③ hooks.gmail.model（Gmail hook）
   ④ subagents.model（if configured）
   ⑤ 預設 configured model

3. Session 建立
   - 暫態 key: cron:{jobId}:run:{uuid}
   - 基底 key: cron:{jobId}（跨 run 保留）
   - 建新或載入既有 session entry

4. Message 建構
   - "[cron:{jobId} {name}] {message}"
   - 附加格式化時間
   - 外部 hook 內容安全包裝（if not whitelisted）
   - 附加 delivery instruction（if announce delivery）

5. 執行
   - resolveDeliveryTarget() → channel, to, accountId
   - 設定 tool policy（require explicit message target）
   - lightContext mode（if payload.lightContext）
   - runEmbeddedPiAgent() 或 CLI runner
   - 捕獲 outputText, summary, usage

6. 回傳
   → { status, error?, summary?, sessionId, sessionKey, delivered?, model?, usage? }
```

### Session Reaper（`session-reaper.ts`）

- 在 timer finally 區塊執行（lock 外，避免 deadlock）
- 5 min 最小間隔 per store
- 預設 retention 24h
- 動作：刪除 store entry + archive/cleanup transcript

---

## 9. Cron Delivery 系統

### Delivery 模式

| Mode | 說明 |
|------|------|
| `none` | 不投遞 |
| `announce` | 發送到 messaging channel（subagent announce 或直接 message tool） |
| `webhook` | HTTP POST to URL |

### Delivery Plan 解析（`delivery.ts`）

```typescript
resolveCronDeliveryPlan(job) → {
  mode: "none" | "announce" | "webhook",
  channel?: ChannelId | "last",
  to?: string,
  accountId?: string,
  source: "delivery" | "payload",
  requested: boolean
}
```

### Announce 流程（`delivery-dispatch.ts`）

```
1. 嘗試 subagent announce（if expectsSubagentFollowup）
2. fallback → 直接 message tool sends
3. 目標匹配：normalize channel + to + accountId（忽略 :topic: suffix）
4. 重複抑制：如果 messaging tool 已送到同目標 → skip
```

### Webhook 流程

```
POST JSON → delivery.to URL
  payload: { text, mediaUrls, structured content }
  best-effort retry on transient errors
  timeout handling
```

### Fallback to Main Session

```
如果 announce 未嘗試或失敗:
  → enqueue summary to main session（unless skipHeartbeatDelivery）
  → wakeMode="now" → 觸發 heartbeat wake
```

---

## 10. Cron 持久化與 Run Log

### Store 檔案

```
路徑: {storePath}（e.g., ~/.openclaw/cron/jobs.json）
格式: { version: 1, jobs: CronJob[] }
操作: atomic write（tmpfile + rename）
Migration: 載入時自動 normalize legacy fields
```

### Concurrency Control

```
locked<T>(state, fn):
  - 以 state.op + storeLocks 實現 per-store-path 序列化
  - 確保同時只有 1 個 CRUD + 1 個 timer tick 執行
  - job 實際執行不持 lock（lock 在 execute 前釋放）
```

### Run Log（`run-log.ts`）

```
路徑: {cronDir}/runs/{jobId}.jsonl
格式: JSONL（一行一筆）
權限: 0600
Rotation: prune to keepLines=2000 when > maxBytes=2MB
寫入: 非同步 queue per file path，防並發
```

### Log Entry 結構

```typescript
{
  ts: number,           // epoch ms
  jobId: string,
  action: "finished",
  status?: "ok" | "error" | "skipped",
  error?: string,       // truncated to 200 chars
  summary?: string,
  delivered?: boolean,
  deliveryStatus?: CronDeliveryStatus,
  durationMs?: number,
  nextRunAtMs?: number,
  model?: string,
  provider?: string,
  usage?: { input_tokens, output_tokens, cache_read/write_tokens }
}
```

---

## 11. Process / CommandLane 系統

### CommandLane 定義

```typescript
const enum CommandLane {
  Main = "main",         // 主 agent 回覆（預設 serial）
  Cron = "cron",         // 背景 cron jobs
  Subagent = "subagent", // Subagent 執行
  Nested = "nested",     // 巢狀 agent spawn
}
```

### Queue 架構（`command-queue.ts` — 324 行）

```typescript
LaneState {
  lane: string;
  queue: QueueEntry[];           // FIFO 待執行
  activeTaskIds: Set<number>;    // 正在執行的 task ID
  maxConcurrent: number;         // 並行上限（≥1）
  draining: boolean;             // pump() 防重入
  generation: number;            // reset 遞增，使 stale task 無效化
}

QueueEntry {
  task: () => Promise<unknown>;
  resolve, reject;
  enqueuedAt: number;
  warnAfterMs: number;          // default 2000ms
  onWait?: (waitMs, queuedAhead) => void;
}
```

### Pump 機制

```
enqueueCommandInLane(lane, task) →
  1. check gatewayDraining → throw GatewayDrainingError
  2. queue.push(entry)
  3. pump():
     while activeTaskIds.size < maxConcurrent && queue.length > 0:
       shift entry from queue
       assign taskId, capture generation
       add to activeTaskIds
       spawn async: execute task → on complete:
         check generation match（mismatch = stale, no-op）
         remove taskId
         call pump() again
```

**Lane 間無排序；Lane 內 FIFO。**

### Gateway 並行度設定

```typescript
setCommandLaneConcurrency(CommandLane.Main, resolveAgentMaxConcurrent(cfg));
setCommandLaneConcurrency(CommandLane.Subagent, resolveSubagentMaxConcurrent(cfg));
setCommandLaneConcurrency(CommandLane.Cron, cfg.cron?.maxConcurrentRuns ?? 1);
```

### Queue API

```typescript
enqueueCommandInLane<T>(lane, task, opts?) → Promise<T>
setCommandLaneConcurrency(lane, maxConcurrent)
getQueueSize(lane) → number
getTotalQueueSize() → number
clearCommandLane(lane) → number         // reject pending tasks
getActiveTaskCount() → number
waitForActiveTasks(timeoutMs) → { drained: boolean }
markGatewayDraining()                   // reject new enqueues
resetAllLanes()                         // SIGUSR1 in-process restart
```

### Reset 安全（SIGUSR1）

```
resetAllLanes() →
  bump generation for each lane
  clear activeTaskIds
  → stale task completions check generation mismatch → no-op
  → queued entries preserved, pump immediately drains
  → gatewayDraining reset to false
```

---

## 12. ProcessSupervisor + exec 安全

### ProcessSupervisor（`supervisor/supervisor.ts` — 282 行）

```typescript
interface ProcessSupervisor {
  spawn(input: SpawnInput) → Promise<ManagedRun>;
  cancel(runId, reason?)
  cancelScope(scopeKey, reason?)   // scope-based 批量取消
  reconcileOrphans() → Promise<void>
  getRecord(runId) → RunRecord?
}

SpawnInput {
  runId?: string;
  sessionId: string;
  backendId: string;
  scopeKey?: string;
  replaceExistingScope?: boolean;  // 取消同 scope 的先前 run
  mode: "child" | "pty";
  // argv, timeouts, callbacks...
}

ManagedRun {
  runId: string;
  pid?: number;
  startedAtMs: number;
  stdin?: ManagedRunStdin;
  wait() → Promise<RunExit>;
  cancel(reason?) → void;
}
```

### 兩種執行模式

| Mode | 說明 |
|------|------|
| `child` | 標準 subprocess（argv-based, detached on POSIX） |
| `pty` | Interactive shell（@lydell/node-pty, 120x30 TTY） |

### exec 安全硬化（`exec.ts` — 342 行）

**Windows CVE-2024-27980 緩解**：
- `.cmd/.bat` 不直接 spawn（Node 18.20.2+ 限制）
- npm/npx → resolve 為 `node.exe <npm-cli.js>`
- 其他 → 加 `.cmd` extension
- **永遠 `shell: false`**

**注入防護**：
```typescript
const WINDOWS_UNSAFE_CMD_CHARS_RE = /[&|<>^%\r\n]/;
// 含不安全字元 → throw Error("Unsafe Windows cmd.exe argument")
```

### Timeout 機制

- Overall timeout: `timeoutMs` → SIGKILL after grace
- No-output timeout: `noOutputTimeoutMs` → resets on stdout/stderr activity
- Grace period: SIGTERM first → wait → SIGKILL

### Process Tree 終止（`kill-tree.ts` — 104 行）

| 平台 | Graceful | Force |
|------|----------|-------|
| Unix | SIGTERM to process group (-pid) | SIGKILL after 3s |
| Windows | `taskkill /T /PID` | `taskkill /F /T /PID` after 3s |

### RunRecord Registry（`registry.ts` — 154 行）

- 追蹤所有 spawn：starting → running → exiting → exited
- 保留上限 2000 筆 exited records
- 提供 `listByScope(scopeKey)` 查詢

---

## 13. Heartbeat Runner 引擎

### 核心檔案

| 檔案 | 行數 | 職責 |
|------|------|------|
| `infra/heartbeat-runner.ts` | 1238 | 主排程 + 單次執行邏輯 |
| `infra/heartbeat-wake.ts` | 262 | Event-driven wake + coalescing |
| `infra/heartbeat-events.ts` | 59 | Event bus + indicator type |
| `infra/heartbeat-events-filter.ts` | 97 | Event 分類（cron/exec/noise） |
| `infra/heartbeat-visibility.ts` | 74 | Channel/account 可見度解析 |
| `infra/heartbeat-reason.ts` | 55 | Reason kind 分類 |
| `infra/heartbeat-active-hours.ts` | 100 | Quiet hours 時區支援 |

### Timer 機制

- 單一 timer 對應最近到期的 agent
- 每個 `HeartbeatAgentState` 追蹤：agentId, config, intervalMs, lastRunMs, nextDueMs
- `scheduleNext()` → `setTimeout(Math.max(0, nextDue - now))`

### 單次 Heartbeat 執行（`runHeartbeatOnce`）

```
1. 早期退出:
   - 全域/agent/interval disabled
   - Quiet hours（isWithinActiveHours）
   - Main queue 忙碌（getQueueSize）

2. Preflight:
   - 載入 session, peek system events
   - 檢查 HEARTBEAT.md 是否空
   - 分類 reason（exec, cron, wake, interval）
   → skip if reason warrants

3. Delivery Setup:
   - 解析 target channel, account, visibility
   - 決定內容是否可達使用者

4. Model 執行:
   - 建構 prompt（注入 system events, cron reminders, exec results）
   - 嘗試 "heartbeat OK" silent ACK（if showOk enabled）
   - 執行 model

5. Reply 正規化:
   - Strip heartbeat token/prefix
   - 空或純 token → silent OK, prune transcript, advance schedule
   - 24h 窗口去重（lastHeartbeatText 比對）

6. Delivery:
   - 檢查 channel readiness
   - 投遞 normalized payload

7. Event 發射:
   - emitHeartbeatEvent() → status + indicator type
```

### Transcript Pruning

如果 heartbeat 只產出 `HEARTBEAT_OK` token → **truncate file back to pre-run size**，避免 context 汙染。

### Visibility 三層優先

```
Per-account heartbeat config
  ↓ (not set)
Per-channel heartbeat config
  ↓ (not set)
Global defaults: { showOk: false, showAlerts: true, useIndicator: true }
```

### Active Hours（`heartbeat-active-hours.ts`）

```yaml
activeHours:
  start: "09:00"
  end: "18:00"
  timezone: "Asia/Taipei"  # or "user" | "local"
```

- 支援跨午夜 wraparound（e.g., 22:00–06:00）
- `endMin === startMin` → 全部 disable

---

## 14. Heartbeat Wake 機制

### Reason 優先階層

```
RETRY (0) < INTERVAL (1) < DEFAULT (2) < ACTION (3)

ACTION = manual, exec-event, hook（最高優先）
```

### Coalescing 邏輯

```
queuePendingWakeReason(target, reason) →
  - target key = "agentId :: sessionKey"
  - 同 target 只保留一個 reason
  - 高優先 → 覆寫低優先
  - 同優先 + 較新 timestamp → 覆寫

schedule(delay?) →
  - default coalesce: 250ms
  - retry cooldown: 1000ms
  - 如果現有 timer 更早 → 保留
  - 否則 preempt timer

timer fires →
  1. handler not set → return
  2. already running → reschedule, return
  3. 取出所有 pending wakes, clear queue
  4. for each: call handler
  5. "requests-in-flight" → re-queue with retry timer
```

### 入口

```typescript
requestHeartbeatNow(opts?)           // 觸發 wake
hasHeartbeatWakeHandler() → boolean  // 是否已註冊
hasPendingHeartbeatWake() → boolean  // timer 或 queue 活躍
```

---

## 15. System Event Bus

### 架構（`system-events.ts` — 119 行）

**In-memory, session-scoped, ephemeral queue** — 無持久化。

```typescript
type SystemEvent = { text: string; ts: number; contextKey?: string | null };

type SessionQueue = {
  queue: SystemEvent[];        // FIFO, max 20
  lastText: string | null;     // 去重 marker
  lastContextKey: string | null;
};
```

### 操作

| 操作 | 說明 |
|------|------|
| `enqueueSystemEvent(sessionKey, text, contextKey?)` | 加入佇列，去重連續相同 text |
| `drainSystemEventEntries(sessionKey)` | 取出全部 + 清空（消耗性） |
| `peekSystemEventEntries(sessionKey)` | 唯讀取出（不消耗） |
| `isSystemEventContextChanged(sessionKey, contextKey)` | 偵測 context 切換 |

### 使用模式

- Heartbeat runner：preflight peek events → 注入 prompt → model 執行
- 分類為 cron / exec / noise
- Cap: 20 events per session, 超出 shift oldest

---

## 16. Gateway Restart 邏輯

### 兩條路徑（`restart.ts` — 505 行）

#### Path 1: Graceful Scheduled Restart

```
scheduleGatewaySigusr1Restart(opts) →
  1. 已有 restart in-flight → coalesce, return
  2. 已有 timer → 比較 due time:
     - 新的更早 → preempt + reschedule
     - 否則 coalesce
  3. 設 pending timer:
     delay = delayMs + cooldownMsApplied (30s)
     on fire → preRestartCheck() → deferGatewayRestartUntilIdle()
```

#### Path 2: Idle-Polling Deferred Restart

```
deferGatewayRestartUntilIdle(opts) →
  1. getPendingCount() = 0 → onReady() + emit restart
  2. getPendingCount() > 0 → 開始 polling:
     every 500ms check pending count
     drained → onReady() + emit restart
     elapsedMs ≥ maxWaitMs → timeout, emit anyway
  3. emit via emitGatewayRestart()
```

### SIGUSR1 Token 機制

```
emittedRestartToken    — 發射時的 cycle ID
consumedRestartToken   — run loop 消費時的 cycle ID
sigusr1AuthorizedCount + sigusr1AuthorizedUntil — grace window 5s
→ 防止 race condition
```

### Platform Restart（`triggerOpenClawRestart`）

| 平台 | 方法 |
|------|------|
| Linux | `systemctl --user restart <unit>` |
| macOS | `launchctl kickstart -k gui/<uid>/<label>` |
| Windows | `schtasks.exe` → `relaunchGatewayScheduledTask()` |

### Restart Cooldown

`RESTART_COOLDOWN_MS = 30s` — 防止快速重啟迴圈。

---

## 17. 跨系統整合圖

```
                  ┌─────────────┐
                  │   Gateway    │
                  │  Startup     │
                  └──────┬──────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   loadInternalHooks  startCron   startHeartbeat
          │              │              │
          ▼              ▼              ▼
   ┌──────────┐  ┌──────────────┐  ┌──────────────┐
   │  Hooks   │  │ CronService  │  │  Heartbeat   │
   │ Registry │  │   Timer      │  │   Runner     │
   └────┬─────┘  └──────┬───────┘  └──────┬───────┘
        │               │                 │
        │    ┌──────────┼──────────┐      │
        │    ▼          ▼          ▼      │
        │  schedule   execute    deliver  │
        │    │          │          │      │
        │    │    ┌─────┴─────┐   │      │
        │    │    ▼           ▼   │      │
        │    │  Isolated    Main  │      │
        │    │  Agent Run   Event │      │
        │    │    │           │   │      │
        │    │    │   enqueueSystemEvent  │
        │    │    │           │   │      │
        │    │    │           ▼   │      │
        │    │    │    ┌──────────┴──┐   │
        │    │    │    │ System Event│   │
        │    │    │    │    Bus      │   │
        │    │    │    └──────┬──────┘   │
        │    │    │           │          │
        │    │    │    peek events  ◄────┘
        │    │    │           │
        │    │    ▼           ▼
        │    │  Delivery   Heartbeat
        │    │  Dispatch   Wake
        │    │    │          │
        ▼    ▼    ▼          ▼
   ┌───────────────────────────────┐
   │      CommandLane Queue        │
   │  Main │ Cron │ Subagent │...  │
   └───────────────┬───────────────┘
                   │
                   ▼
   ┌───────────────────────────────┐
   │     ProcessSupervisor         │
   │  child / pty spawn            │
   │  timeout + kill-tree          │
   └───────────────────────────────┘
```

### 關鍵交互

| 觸發者 | 目標 | 路徑 |
|--------|------|------|
| Webhook POST | Hook Mapping → Agent Run | `/hooks/*` → dispatchAgentHook |
| Gmail Pub/Sub | Gmail Watcher → Hook Mapping | gog serve → `/hooks/gmail` |
| Cron Timer | Isolated Agent | onTimer → executeJobCore → runCronIsolatedAgentTurn |
| Cron systemEvent | Heartbeat | enqueueSystemEvent → requestHeartbeatNow |
| Heartbeat | System Events | peekSystemEventEntries → 注入 prompt |
| Config Reload | All | restartCron, reloadInternalHooks, restartHeartbeat |
| SIGUSR1 | Gateway | deferUntilIdle → emitGatewayRestart |

---

## 18. 邊界條件與陷阱

1. **Croner year-rollback bug**：nextRun() 可能回傳 ≤ nowMs 的值。Workaround：retry from +1s，then tomorrow UTC。

2. **MIN_REFIRE_GAP_MS (2s)**：Cron schedule 落在同 millisecond → 安全網防 spin-loop。

3. **Session Reaper lock 順序**：在 locked() 區塊外執行，避免 deadlock（reaper 需要存取 session store）。

4. **One-shot retry 條件**：只有 transient error（rate_limit, overloaded, network, timeout, 5xx）才重試，permanent error 直接 disable。

5. **Stagger 穩定性**：SHA256(jobId) 雜湊確保同 job 永遠同偏移，部署重啟不影響。

6. **Hook boundary check**：handler 路徑必須在 hook 目錄內，realpath 驗證防 symlink escape。

7. **Hook template 安全**：`__proto__`, `prototype`, `constructor` 在 template 變數中被阻擋。

8. **Gmail address-in-use**：gog serve crash 時 5s backoff 重啟，但 EADDRINUSE → 不重啟（已有 instance）。

9. **Heartbeat transcript pruning**：HEARTBEAT_OK only → truncate file back，防 context 無意義膨脹。但 event-driven reason 不做 pruning。

10. **Wake coalescing preempt**：新的高優先 wake 可以 preempt 既有低優先 timer，但 running 中的 handler 不會被打斷。

11. **Generation 防 stale task**：SIGUSR1 restart 後，舊 task 的 completion callback 因 generation mismatch 被忽略。

12. **Windows exec CVE-2024-27980**：.cmd/.bat 不直接 spawn，npm/npx resolve 為 node.exe + cli.js 路徑。`shell: false` 永遠不開。

13. **Cron store concurrent write**：locked() 序列化所有 CRUD + timer，但 job 執行期間不持 lock → 結果寫回時 forceReload 再鎖。

14. **Heartbeat 24h 去重**：同樣內容在 24h 內不重複投遞，但 event-driven reason（exec, cron）bypass 去重。

15. **Restart cooldown 30s**：防止 config 連續修改造成快速重啟循環。Idle poll 每 500ms，maxWaitMs 超時強制重啟。

---

## 19. 關鍵常量速查

| 常量 | 值 | 位置 |
|------|---|------|
| Timer max delay | 60s | `service/timer.ts MAX_TIMER_DELAY_MS` |
| Timer min refire gap | 2s | `service/timer.ts MIN_REFIRE_GAP_MS` |
| Cron max concurrent (default) | 1 | `service/timer.ts` |
| Cron backoff梯度 | 30s/1m/5m/15m/60m | `service/jobs.ts` |
| One-shot max retries (default) | 3 | `cron config retry.maxAttempts` |
| Failure alert threshold (default) | 2 consecutive | `cron config failureAlert.after` |
| Failure alert cooldown | 1h | `cron config failureAlert.cooldownMs` |
| Run log max bytes | 2MB | `run-log.ts maxBytes` |
| Run log keep lines | 2000 | `run-log.ts keepLines` |
| Session reaper throttle | 5 min | `session-reaper.ts` |
| Session retention (default) | 24h | `cron config sessionRetention` |
| Stagger window (default) | 5 min | `stagger.ts` |
| Croner LRU cache | 512 entries | `schedule.ts` |
| Wake coalesce delay | 250ms | `heartbeat-wake.ts DEFAULT_COALESCE_MS` |
| Wake retry cooldown | 1000ms | `heartbeat-wake.ts DEFAULT_RETRY_MS` |
| SIGUSR1 auth grace | 5s | `restart.ts SIGUSR1_AUTH_GRACE_MS` |
| Restart cooldown | 30s | `restart.ts RESTART_COOLDOWN_MS` |
| Idle poll interval | 500ms | `restart.ts pollMs` |
| Hook auth rate limit | 20/60s | `server-http.ts` |
| Kill tree grace period | 3s | `kill-tree.ts` |
| Queue warn threshold (default) | 2000ms | `command-queue.ts` |
| RunRecord exited max | 2000 | `supervisor/registry.ts` |
| PTY default size | 120x30 | `supervisor/adapters/pty.ts` |
| Gmail watch renew | 12h (720 min) | `gmail config renewEveryMinutes` |
| gog serve port (default) | 8788 | `gmail config serve.port` |
| System event queue max | 20 | `system-events.ts` |

---

## 20. C# 概念對照

| OpenClaw (TS) | C# / .NET 對照 |
|---------------|---------------|
| `CommandLane` + `command-queue.ts` | `Channel<T>` + `SemaphoreSlim` per lane |
| `enqueueCommandInLane` pump loop | 類似 `BoundedChannelOptions.SingleReader` + `Task.WhenAll` worker pool |
| `generation` stale invalidation | `CancellationToken` + 版本號比對 |
| `markGatewayDraining()` | `IHostApplicationLifetime.StopApplication()` + drain middleware |
| `ProcessSupervisor` | `IHostedService` + managed `Process.Start()` |
| `kill-tree.ts` | `Process.Kill(entireProcessTree: true)` (.NET 5+) |
| `exec.ts` Windows hardening | 類似 `ProcessStartInfo { UseShellExecute = false }` + 手動 CMD escape |
| `CronService` timer loop | `IHostedService` + `PeriodicTimer` (.NET 6+) + `System.Threading.Timer` |
| `locked()` serialization | `SemaphoreSlim(1, 1)` or `AsyncLock` |
| `croner` cron parser | NCrontab / Cronos library |
| `CronStoreFile` JSON | EF Core SQLite 或 `System.Text.Json` + file lock |
| `runCronIsolatedAgentTurn` | Background task via `IBackgroundTaskQueue` |
| `HeartbeatRunner` interval timer | `System.Timers.Timer` + agent state machine |
| `HeartbeatWake` coalescing | `System.Reactive` `Throttle` + priority queue |
| `SystemEvent` queue | `ConcurrentQueue<T>` per session key |
| `deferGatewayRestartUntilIdle` | Graceful shutdown via `IHostApplicationLifetime` + drain polling |
| `SIGUSR1` restart | Windows Service `OnCustomCommand()` or `Environment.FailFast` + auto-restart |
| Hook `globalThis` singleton Map | `static ConcurrentDictionary` or DI singleton |
| Hook `dynamic import()` | `Assembly.LoadFrom()` + MEF/`[Export]` |
| Hook frontmatter YAML | 類似 `AssemblyMetadataAttribute` + manifest 檔 |
| Gmail Watcher gog subprocess | `Process.Start()` + `OutputDataReceived` event |
