# AI Agent 引擎（src/agents/ — 802 files）

## 核心入口
- `pi-embedded-runner.ts` → re-exports `runEmbeddedPiAgent()`
- `pi-embedded-runner/run.ts` → 主執行迴圈（1500+ 行）
- `system-prompt.ts` → 動態 system prompt 組裝（726 行）

## Agent 生命週期

```
BOOTSTRAP → SYSTEM PROMPT → SESSION → TOOLS → EXECUTION → RESPONSE
```

### 1. Bootstrap
- `resolveRunWorkspaceDir()` — 工作目錄解析
- `ensureAuthProfileStore()` — API key 載入（多 profile 輪替）
- `resolveModel()` — 模型解析（plugin hooks → session → agent config → global → default）
- DEFAULT_MODEL = `claude-opus-4-6`

### 2. System Prompt（20+ sections）
動態組裝，根據 PromptMode（full/minimal/none）：
- Identity / Tooling / Tool Call Style / Safety
- Skills（scan → read → follow）
- Memory Recall（memory_search/memory_get）
- CLI Quick Reference / Model Aliases
- Workspace / Documentation / Sandbox Info
- Authorized Senders / Date & Time
- Reply Tags / Messaging / Voice(TTS) / Group Chat
- Reactions / Reasoning Format / Project Context
- Silent Replies / Heartbeats / Runtime Info

### 3. Tool 系統

**Pi 框架內建 Tools**：read / write / edit / apply_patch / grep / find / ls / exec / process

**OpenClaw 自有 Tools**：
| Tool | 用途 |
|------|------|
| message | 發送訊息到各頻道 |
| sessions_list/history/send/spawn | Session 管理 |
| subagents | 子代理管理 |
| browser | Chrome/Firefox 控制 |
| canvas | Canvas UI 呈現 |
| nodes | IoT 裝置控制 |
| cron | 排程提醒 |
| gateway | Gateway 重啟/更新 |
| web_search / web_fetch | 網頁搜尋/抓取 |
| image / pdf / tts | 多媒體處理 |

**Tool Policy 8 層 Pipeline**：
```
owner-only → global → group → subagent → provider → sandbox → channel → model
```

### 4. 執行迴圈
```
streamSimple() 呼叫 LLM
  ↓ tool_calls? → 執行工具 → 結果回寫 session → 再呼叫 LLM
  ↓ stop_reason = "completed"? → 結束
  ↓ context overflow? → compact()（摘要壓縮）→ 繼續
```

### 5. Failover 策略
- 錯誤分類：context_overflow / auth_error / billing_error / rate_limit / timeout
- 每個 auth profile 8 次重試
- 全域上限：base 24 + (8 × profile_count)，最大 160
- Backoff：250ms → 1.5s（含 jitter）
- 模型降級：opus → sonnet → haiku

### 6. Auth Profile 管理
- 多 API key 集合 per provider
- 輪替策略：primary → secondary → tertiary
- Cooldown tracking：失敗 → 退避
- 儲存：`~/.openclaw/auth-profiles.json`（加密）

## 關鍵檔案路徑

| 檔案 | 行數 | 職責 |
|------|------|------|
| `agents/pi-embedded-runner/run.ts` | 1500+ | 主執行迴圈 |
| `agents/system-prompt.ts` | 726 | System prompt 組裝 |
| `agents/pi-embedded-runner/run/attempt.ts` | 600+ | 單 turn 執行 |
| `agents/pi-tools.ts` | 600+ | Tool registry 建立 |
| `agents/agent-scope.ts` | 282 | Agent 設定解析 |
| `agents/model-selection.ts` | — | Provider/alias 處理 |
| `agents/auth-profiles.ts` | — | API key 輪替 |
| `agents/pi-embedded-runner/compact.ts` | — | Context overflow 處理 |
| `agents/openclaw-tools.ts` | — | OpenClaw tool 工廠 |
| `agents/failover-error.ts` | — | 錯誤分類 + 重試 |
