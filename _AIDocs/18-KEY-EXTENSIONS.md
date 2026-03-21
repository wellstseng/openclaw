# 關鍵 Extensions 深入

> Phase 5-2 | 掃描 `extensions/` 下 8 個關鍵 extensions
> voice-call (~68 files, ~9.9K LOC) + acpx (~23 files, ~4.8K LOC) + diffs (~28 files, ~4.2K LOC) + llm-task (6 files, ~537 LOC)
> + thread-ownership (3 files, ~341 LOC) + open-prose (skill-delivery, ~7.5K LOC) + diagnostics-otel (5 files, ~1.1K LOC) + shared (2 files, ~79 LOC)
> 總計 ~135+ files, ~28K+ LOC

---

## 1. 架構鳥瞰 — Extension 類型分佈

```
Extensions 按複雜度分層：

┌──────────────────────────────────────────────────────────────────┐
│  Layer 4: Full-Stack Extension（多 register + 背景服務）          │
│  voice-call: Tool + Gateway + CLI + Service + HTTP webhook       │
│  diffs:      Tool + HTTP route + Hook (before_prompt_build)      │
├──────────────────────────────────────────────────────────────────┤
│  Layer 3: Service Extension（背景服務 + 協定適配）                │
│  acpx:      Service（ACP runtime backend, process spawner）      │
│  diagnostics-otel: Service（OTel SDK 生命週期管理）               │
├──────────────────────────────────────────────────────────────────┤
│  Layer 2: Tool / Hook Extension（單一功能點）                     │
│  llm-task:         Tool（JSON-only LLM 呼叫）                    │
│  thread-ownership: Hook×2（message_received + message_sending）  │
├──────────────────────────────────────────────────────────────────┤
│  Layer 1: Passive Extension（無 runtime 邏輯）                    │
│  open-prose: Skill delivery（/prose command + .prose VM）         │
│  shared:     Test helpers only（無 register 邏輯）                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. voice-call — 語音通話系統

### 2.1 架構

```
┌────────────────────────────────────────────────────────┐
│               VoiceCallRuntime (index.ts)               │
│  config: VoiceCallConfig                                │
│  provider: VoiceCallProvider (Twilio/Telnyx/Plivo/Mock) │
│  manager: CallManager (state machine + persistence)     │
│  webhookServer: HTTP + WebSocket                        │
└──────────┬──────────┬──────────────┬───────────────────┘
           │          │              │
    ┌──────┴──┐  ┌────┴────────┐  ┌─┴─────────────────┐
    │Provider │  │ CallManager │  │ Webhook Server     │
    │(4 impl) │  │ • activeCalls│  │ • HTTP :3334       │
    │Twilio   │  │ • store JSONL│  │ • WS media stream  │
    │Telnyx   │  │ • timers     │  │ • OpenAI RT STT    │
    │Plivo    │  │ • transcript │  │ • Stale reaper     │
    │Mock     │  │   waiters    │  │                    │
    └─────────┘  └─────────────┘  └────────────────────┘
```

### 2.2 Plugin 註冊

| 註冊類型 | 名稱 | 說明 |
|---------|------|------|
| Gateway Method ×5 | `voicecall.{initiate,continue,speak,end,status}` | 外部系統可透過 RPC 控制通話 |
| Tool | `voice_call` | Agent 工具，union schema 支援 5 種 action |
| CLI | `voicecall` | 命令列：call/continue/speak/end/status/tail/expose |
| Service | voice-call | 啟動 webhook server + tunnel |

### 2.3 兩種通話模式

| 模式 | 流程 | 用途 |
|------|------|------|
| **Notify** | 播放 TTS → 延遲 hangup → 結束 | OTP、告警、通知 |
| **Conversation** | 開啟 Media Stream → STT → Agent 回應 → TTS → 循環 | 客服、互動語音 |

### 2.4 Outbound Conversation 資料流

```
initiateCall(to, message, mode:"conversation")
  │
  ├→ CallManager.initiateCall() → CallRecord(initiated)
  ├→ Provider REST API → 撥出電話
  │
  ▼ (電話接通)
POST /voice/webhook → provider.verifyWebhook()（HMAC 驗簽）
  ├→ parseWebhookEvent() → NormalizedEvent[]
  ├→ call.answered → state:answered
  │   └→ TwiML <Start><Stream> → 開啟 MediaStream
  │
WS /voice/stream → MediaStream WebSocket
  ├→ Token 驗證（constant-time compare）
  ├→ mu-law audio → OpenAI Realtime STT
  ├→ VAD 偵測語音結束 → transcript
  │   ├→ Barge-in：中斷正在播放的 TTS
  │   ├→ processEvent(call.speech)
  │   └→ generateVoiceResponse()
  │       └→ Embedded Pi Agent（完整工具支援）
  ├→ Response text → provider.playTts()
  │   └→ OpenAI TTS → PCM 24kHz → 重取樣 8kHz → mu-law G.711 → 20ms chunks
  └→ 循環直到 max duration 或 hangup
```

### 2.5 Audio Pipeline

```
Input: 16-bit PCM (variable sample rate)
  → resamplePcmTo8k()（線性插值）
  → pcmToMulaw()（指數量化, bias=132, clip=32635）
  → chunkAudio()（160 bytes = 20ms @ 8kHz）
  → Twilio media.media WebSocket events
```

### 2.6 Provider 差異

| 面向 | Twilio | Telnyx | Plivo |
|------|--------|--------|-------|
| 驗簽 | HMAC-SHA1 | Ed25519 | HMAC-SHA256 (v3+nonce) |
| 格式 | Form-encoded | JSON | XML (TwiML-like) |
| Media Stream | WebSocket (mu-law) | 不支援 | 不支援 |
| TTS | OpenAI→mu-law chunks | 原生 speaker.play_tts | XML `<Speak>` |
| 呼叫控制 | REST + TwiML | Call Control v2 | REST + XML answer_url |
| Client State | N/A | base64 callId 編碼 | N/A |

### 2.7 Webhook 安全

- **Replay 偵測**：10 分鐘快取窗口，constant-time compare，max 10K entries
- **URL 重建**：allowedHosts 白名單 OR trustForwardingHeaders + trustedProxyIPs
- **Media Stream 節流**：pre-start timeout 5s，max pending 32，per-IP 4，max total 128
- **Body 上限**：1MB（防 DoS）

### 2.8 狀態管理

**CallRecord**（JSONL 持久化）：
```
callId(UUID) + providerCallId + provider + direction + state
from/to(E.164) + transcript[] + processedEventIds(replay dedup)
startedAt/answeredAt/endedAt + endReason + metadata
```

**State Machine**：
- Non-terminal: initiated → ringing → answered → active → speaking/listening
- Terminal: completed, hangup-user, hangup-bot, timeout, error, failed, no-answer, busy, voicemail

**Restart Recovery**：Load JSONL → verify with provider → skip old/terminal → rebuild maps → restart timers

### 2.9 關鍵設定

| 設定 | 預設 | 說明 |
|------|------|------|
| maxDurationSeconds | 3600 | 最大通話時長 |
| maxConcurrentCalls | 10 | 並行通話上限 |
| notifyHangupDelaySec | (config) | Notify 模式自動掛斷延遲 |
| staleCallReaperSeconds | 0 (關閉) | 孤兒通話自動清理 |
| streaming.silenceDurationMs | 800 | VAD 靜音判定門檻 |
| streaming.vadThreshold | 0.5 | 語音活動偵測閾值 |

### 2.10 邊界條件

1. **Stale Media Stream**：WS 斷線但 provider 未送 stop → onDisconnect callback + maxDuration timer 雙重保護
2. **Transcript Timeout**：用戶靜音超過 transcriptTimeoutMs → reject promise，通話仍 active
3. **Per-turn nonce (Twilio)**：turnToken 防止 STT 回傳過時 transcript
4. **Session Key 碰撞**：`voice:{phone}` 作 session key，同號碼多通會共享 context
5. **Codec 固定**：只支援 mu-law G.711，TTS 非預期格式直接 throw

---

## 3. acpx — Agent Client Protocol 擴展

### 3.1 架構

```
┌───────────────────────────────────────────┐
│        OpenClaw Plugin Service             │
│  start() → AcpxRuntime + ensureAcpx()     │
│  stop() → unregister backend              │
├───────────────────────────────────────────┤
│        AcpxRuntime (implements AcpRuntime) │
│  ensureSession() → acpx sessions ensure    │
│  runTurn() → async generator → events      │
│  cancel() / close() / setMode()            │
│  getStatus() / doctor()                    │
├───────────────────────────────────────────┤
│        Process Layer                       │
│  spawn acpx CLI (child process)            │
│  Windows .cmd wrapper resolution           │
│  AbortSignal → SIGTERM → SIGKILL (250ms)  │
├───────────────────────────────────────────┤
│        Event Parser                        │
│  JSON lines → AcpRuntimeEvent             │
│  text_delta / tool_call / status / done    │
├───────────────────────────────────────────┤
│        MCP Proxy (optional)                │
│  mcp-proxy.mjs: JSON-RPC line rewriter    │
│  Intercept session/new → inject mcpServers │
└───────────────────────────────────────────┘
```

### 3.2 Plugin 註冊

只註冊一個 **Service**，service.start() 時：
1. 建立 `AcpxRuntime` 實例
2. 呼叫 `registerAcpRuntimeBackend()` 立即註冊
3. 背景異步：`ensureAcpx()` 安裝/驗證 binary + `probeAvailability()` 測試

### 3.3 Session 管理

**Handle 編碼**：`acpx:v1:{base64url(JSON state)}`
```typescript
AcpxHandleState = {
  name: string;              // session name
  agent: string;             // "codex" | "claude" | "gemini" | ...
  cwd: string;               // working directory
  mode: "persistent" | "oneshot";
  acpxRecordId?: string;
  backendSessionId?: string;
  agentSessionId?: string;
}
```

**Session Lifecycle**：
```
ensureSession(sessionKey, agent) → Handle
  → acpx sessions ensure --name <key>
  → (or) acpx sessions new --name <key>
  ↓
runTurn(handle, text) → AsyncIterable<Event>
  → spawn acpx <agent> prompt --session <name> --file -
  → stdin: prompt text → close
  → stdout: JSON lines → parse → yield events
  ↓
cancel(handle) → acpx cancel --session <name>
close(handle) → acpx sessions close <name>
```

### 3.4 Event 類型映射

| acpx Output | AcpRuntimeEvent |
|-------------|-----------------|
| text / agent_message_chunk | `text_delta` (stream: "output") |
| thought / agent_thought_chunk | `text_delta` (stream: "thought") |
| tool_call / tool_call_update | `tool_call` (title, status, toolCallId) |
| usage_update | `status` (used, size) |
| done | `done` (stopReason) |
| error | `error` (message, code, retryable) |
| malformed JSON | `status` (raw line) |

### 3.5 MCP Server 注入

```
有 mcpServers config 時：
  resolveAcpxAgentCommand(agent)
    → 查 acpx config show 取 override 或 builtin defaults
    → 例: codex → "npx @zed-industries/codex-acp"
  buildMcpProxyAgentCommand(targetCommand, mcpServers)
    → "node <mcp-proxy.mjs> --payload <base64url>"

mcp-proxy.mjs 攔截 stdin JSON-RPC：
  method = session/new | session/load | session/fork
    → params.mcpServers = inject configured servers
  其他 → passthrough
stdout → 直接 pipe（不修改）
```

**Builtin Agent Defaults**：
- codex → `npx @zed-industries/codex-acp`
- claude → `npx -y @zed-industries/claude-agent-acp`
- gemini → `gemini`
- opencode → `npx -y opencode-ai acp`
- pi → `npx pi-acp`

### 3.6 Windows 相容

- `.cmd`/`.bat` wrapper 自動解析 + 快取
- `strictWindowsCmdWrapper=true`（預設）：拒絕未解析 wrapper，強制修正
- Compat 模式：fallback `cmd.exe /c <command>`

### 3.7 關鍵設定

| 設定 | 預設 | 說明 |
|------|------|------|
| command | (bundled) | acpx 執行檔路徑 |
| expectedVersion | "0.1.15" | 釘選版本，"any" 跳過檢查 |
| permissionMode | "approve-reads" | approve-all / approve-reads / deny-all |
| queueOwnerTtlSeconds | 0.1 | Queue owner idle TTL（短→避免延遲完成） |
| timeoutSeconds | (none) | Per-turn timeout |
| mcpServers | {} | MCP server 定義 |

### 3.8 邊界條件

1. **Exit Code 5**：Permission denied → 提供 config 路徑提示
2. **ENOENT 區分**：command missing vs cwd missing（檢查 cwd 是否存在）
3. **Version Install**：單一全域 Promise 防並行安裝
4. **Stale Probe**：revision tracking 取消跨 start/stop 的過時探測
5. **Orphan Sessions**：OpenClaw crash 後 session 留存，ensureSession 同名可恢復

---

## 4. diffs — Diff 渲染器

### 4.1 架構

```
┌────────────────────────────────────────────────────┐
│  Agent Tool "diffs"                                 │
│  input: before/after OR unified patch               │
│  mode: view | file | image | both                   │
├────────────────────────────────────────────────────┤
│  Rendering Layer                                    │
│  ┌──────────────────┐  ┌────────────────────────┐  │
│  │ renderDiffDocument│  │ PlaywrightScreenshotter│  │
│  │ @pierre/diffs SSR │  │ Chromium → PNG/PDF     │  │
│  │ viewer + image    │  │ auto-scale retry       │  │
│  │ HTML variants     │  │ shared browser pool    │  │
│  └──────────────────┘  └────────────────────────┘  │
├────────────────────────────────────────────────────┤
│  Storage & HTTP Layer                               │
│  ┌──────────────────┐  ┌────────────────────────┐  │
│  │ DiffArtifactStore │  │ HTTP Handler           │  │
│  │ $TMP/openclaw-diffs│  │ GET /plugins/diffs/    │  │
│  │ TTL 30min~6hr     │  │ view/{id}/{token}      │  │
│  │ 10B id + 24B token│  │ CSP + rate limit       │  │
│  └──────────────────┘  └────────────────────────┘  │
├────────────────────────────────────────────────────┤
│  Browser Viewer (Shadow DOM)                        │
│  @pierre/diffs FileDiff hydration                   │
│  Toolbar: layout/wrap/background/theme toggle       │
└────────────────────────────────────────────────────┘
```

### 4.2 Plugin 註冊

| 註冊類型 | 說明 |
|---------|------|
| Tool `diffs` | before/after 或 patch → 生成 viewer URL + PNG/PDF |
| HTTP Route `/plugins/diffs` (prefix) | Serve viewer HTML + assets |
| Hook `before_prompt_build` | 注入 agent guidance（使用 diffs 而非手動 summary） |

### 4.3 Tool 執行流程

```
execute(params)
  ├→ normalizeDiffInput()
  │   before+after → kind:"before_after" (max 512 KiB each)
  │   patch → kind:"patch" (max 2 MiB)
  │
  ├→ renderDiffDocument(input, options)
  │   ├→ @pierre/diffs preloadMultiFileDiff()
  │   ├→ 平行渲染 viewer + image 兩個 HTML variant
  │   └→ buildHtmlDocument() (Shadow DOM + payload JSON)
  │
  ├→ store.createArtifact() → {id, token, viewerPath}
  ├→ buildViewerUrl() → gateway URL
  │
  ├→ (mode=file|both) renderDiffArtifactFile()
  │   ├→ PlaywrightDiffScreenshotter.screenshotHtml()
  │   │   ├→ acquireSharedBrowser() (Chromium, idle 30s)
  │   │   ├→ page.route() → 攔截 asset 請求本地 serve
  │   │   ├→ page.setContent() → waitForFunction(diffsReady)
  │   │   ├→ measure frame → adjust viewport
  │   │   ├→ PNG: screenshot with clip (pixel limit check, auto-reduce scale)
  │   │   └→ PDF: page.pdf() (max 50 pages)
  │   └→ store.updateFilePath()
  │
  └→ return { viewerUrl, filePath, fileFormat, ... }
```

### 4.4 Quality Presets

| Preset | Scale | Max Width | Max Pixels |
|--------|-------|-----------|-----------|
| standard | 2.0 | 960px | 8 MP |
| hq | 2.5 | 1200px | 14 MP |
| print | 3.0 | 1400px | 24 MP |

### 4.5 HTTP 安全

- **Access Control**：Loopback (127.0.0.1/::1) 永遠允許；remote 需 `allowRemoteViewer: true`
- **Rate Limit**：remote per-IP 40 failures / 60s window → 60s lockout
- **CSP**：`default-src 'none'; script-src 'self'; style-src 'unsafe-inline'`
- **Token 安全**：ID 20 hex + Token 48 hex（2^192 空間）
- **Proxy 偵測**：拒絕帶 X-Forwarded-For/X-Real-IP 的請求（除非配置允許）

### 4.6 Artifact Lifecycle

```
$TMPDIR/openclaw-diffs/{id}/
├── meta.json       ← DiffArtifactMeta (id, token, createdAt, expiresAt, ...)
├── viewer.html     ← Full HTML document
└── preview.png     ← Rendered PNG/PDF (optional)

TTL: 預設 30 min, 最大 6 hr
Cleanup: 每 5 min 背景清理 + 24hr orphan sweep
```

### 4.7 Viewer 前端

- **Shadow DOM hydration**：`<template shadowrootmode="open">` → attachShadow → FileDiff
- **Toolbar**：layout(unified/split) + wordWrap + background highlights + theme(light/dark)
- **State 同步**：toolbar 變更 → syncAllControllers() → 所有 diff card rerender

### 4.8 邊界條件

1. **Pixel 超限**：auto-reduce scale（最多 2 次 retry），仍超限 → throw
2. **Patch 檔案上限**：max 128 files per patch, max 120K total lines
3. **Browser 找不到**：搜尋 10+ 路徑（Windows/macOS/Linux），全部失敗 → 無法產生圖片
4. **Deprecated aliases**：imageFormat/imageQuality/imageScale/imageMaxWidth 仍可用

---

## 5. llm-task — LLM 任務執行

### 5.1 概念

**單次 JSON-only LLM 呼叫**。沒有工具、沒有多 turn、沒有 retry。適合 Lobster workflow 的結構化任務。

### 5.2 Plugin 註冊

只註冊一個 **Tool** (`llm-task`)，`optional: true`（需 agent config 明確 allow）。

### 5.3 執行流程

```
execute(prompt, input?, schema?, provider?, model?, ...)
  │
  ├→ Resolve provider/model (param → pluginCfg → global defaults)
  ├→ Check allowedModels allowlist (if configured)
  ├→ Serialize input → JSON
  │
  ├→ Build system prompt:
  │   "You are a JSON-only function.
  │    Return ONLY a valid JSON value.
  │    Do not wrap in markdown fences.
  │    Do not call tools.
  │    TASK: {prompt}
  │    INPUT_JSON: {inputJson}"
  │
  ├→ Create temp workspace: /tmp/openclaw-llm-task-{UUID}/
  │
  ├→ runEmbeddedPiAgent({
  │     disableTools: true,    ← 關鍵：不提供任何工具
  │     provider, model, authProfileId,
  │     timeoutMs (default 30s),
  │     streamParams: { temperature, maxTokens }
  │   })
  │
  ├→ collectText(payloads) → strip code fences → JSON.parse
  │
  ├→ (if schema) AJV validate(parsed, schema)
  │   allErrors: true, strict: false
  │
  └→ return { content: [text], details: { json, provider, model } }

  finally → rm -rf tmpDir
```

### 5.4 關鍵設定

| 設定 | 說明 |
|------|------|
| defaultProvider / defaultModel | Fallback provider/model |
| defaultAuthProfileId | Fallback auth profile |
| allowedModels | `["provider/model"]` 白名單 |
| maxTokens | 預設 max output tokens |
| timeoutMs | 預設 timeout（default 30s） |

### 5.5 邊界條件

1. **LLM 回傳非 JSON**：strip fences 後 JSON.parse 失敗 → throw
2. **Schema 驗證失敗**：AJV allErrors 收集所有問題 → 合併 error message
3. **Empty output**：所有 payload 都是 error → throw "LLM returned empty output"
4. **Dynamic import**：先嘗試 src/ 再 dist/，兩者都失敗 → throw
5. **Temp cleanup**：finally block 保證 rm -rf，cleanup error 被吞掉

---

## 6. thread-ownership — 執行緒所有權

### 6.1 概念

Slack 多 agent 環境下，防止多個 agent 同時回覆同一 thread。透過外部 slack-forwarder 服務做分散式鎖。

### 6.2 Plugin 註冊

| Hook | 事件 | 邏輯 |
|------|------|------|
| `message_received` | 收到訊息 | 偵測 @-mention → 快取 `{channel}:{thread}` (5min TTL) |
| `message_sending` | 發送前 | POST `/api/v1/ownership/{channel}/{threadTs}` → 200 允許 / 409 取消 |

### 6.3 關鍵設計

- **Fail-open**：網路錯誤 → 允許發送（不因外部服務故障阻斷回覆）
- **Mention-bypass**：被 @-mention → 跳過 ownership 檢查
- **A/B test gating**：`abTestChannels` 限定執行 channels
- **3s fetch timeout**：AbortSignal.timeout 防止阻塞
- **5 min mention TTL**：過期自動清理，每次 ownership 檢查前掃描

---

## 7. open-prose — OpenProse VM

### 7.1 概念

**Skill-delivery plugin**。`register()` 是 no-op，所有邏輯在 `.prose` 程式和 `/prose` slash command 中。

```json
{ "id": "open-prose", "skills": ["./skills"] }
```

### 7.2 內容

- **`/prose` slash command**：user-facing skill
- **OpenProse VM 語義**：定義 `.prose` 程式執行環境
- **49 個範例**：從 hello-world 到 RLM、habit mining、language self-improvement
- **8 個 lib 模組**：calibrator, cost-analyzer, error-forensics, ...
- **State backends**：filesystem / in-context / PostgreSQL / SQLite
- **Multi-agent orchestration** 支援

---

## 8. diagnostics-otel — OpenTelemetry 診斷

### 8.1 架構

```
onDiagnosticEvent() 訂閱
  ↓
12 種 event type → Metrics / Traces / Logs
  │
  ├→ model.usage → token counters + cost counter + duration histogram + span
  ├→ webhook.received/processed/error → counters + histograms
  ├→ message.queued/processed → counters + histograms + span
  ├→ queue.lane.enqueue/dequeue → counters + histograms
  ├→ session.state/stuck → counters + spans
  ├→ run.attempt → counter
  └→ diagnostic.heartbeat → queue depth histogram
  ↓
OTLP Export (http/protobuf)
  ├→ Traces: parent-based sampler + configurable ratio
  ├→ Metrics: periodic reader + flush interval
  └→ Logs: custom transport (Pino format → OTel severity)
```

### 8.2 Service Lifecycle

1. **start()**：讀 config → NodeSDK 初始化 → 建立 20 metrics → 訂閱 events → 設定 log transport
2. **stop()**：取消訂閱 → shutdown log provider → shutdown NodeSDK

### 8.3 安全

- **Secret Redaction**：所有 string attribute export 前過濾密鑰
- **Pino Log Parsing**：numeric-keyed format → 提取 JSON bindings → map severity

### 8.4 關鍵設定

| 設定 | 預設 | 說明 |
|------|------|------|
| endpoint | (env: OTEL_EXPORTER_OTLP_ENDPOINT) | OTLP 端點 |
| protocol | "http/protobuf" | 傳輸協定 |
| serviceName | "openclaw" | Service 識別 |
| sampleRate | (none) | 取樣率 [0,1] |
| traces / metrics / logs | true / true / false | 各信號開關 |

---

## 9. shared — 測試工具

2 個檔案，79 行：
- **resolve-target-test-helpers.ts**：4 個通用 `resolveTarget()` error case test
- **windows-cmd-shim-test-fixtures.ts**：Windows `.cmd` shim 檔案 fixture（CRLF 格式）

無 plugin registration，純測試基礎設施。

---

## 10. 跨 Extension 共通模式

### 10.1 Lazy Runtime 初始化

```typescript
// voice-call, diffs, diagnostics-otel 共通模式
let runtimePromise: Promise<Runtime> | null = null;

function ensureRuntime() {
  if (!runtimePromise) {
    runtimePromise = createRuntime().catch(err => {
      runtimePromise = null;  // 失敗可重試
      throw err;
    });
  }
  return runtimePromise;
}
```

### 10.2 Config Resolution Chain

```
param override → pluginConfig → core config → hardcoded default
```

所有 extension 一致遵循此優先順序解析設定。

### 10.3 TypeBox + Zod + JSON Schema 三 schema 系統

| 用途 | Schema 庫 | Extension |
|------|-----------|-----------|
| Tool parameters (Agent) | TypeBox | voice-call, diffs, llm-task |
| Runtime config validation | Zod | voice-call |
| Plugin manifest configSchema | JSON Schema | 全部 |
| Output validation | AJV | llm-task |

### 10.4 Embedded Pi Agent 整合

voice-call 和 llm-task 都使用 `runEmbeddedPiAgent()` 執行獨立 LLM 呼叫：
- voice-call：conversation mode 自動回應，有完整工具
- llm-task：JSON-only，`disableTools: true`

### 10.5 背景 Service 模式

```typescript
// acpx, diagnostics-otel, voice-call
api.registerService({
  id: "my-service",
  start(ctx) {
    // 1. 初始化 runtime
    // 2. 註冊 backend / 訂閱 events
    // 3. 背景異步任務（不阻塞 startup）
  },
  stop(ctx) {
    // 1. 取消訂閱 / 取消註冊
    // 2. 優雅關閉連線
    // 3. 清理資源
  }
});
```

---

## 11. 邊界條件與陷阱總覽

### Discovery & Registration

1. **Tool optional flag**：llm-task 用 `{ optional: true }` 避免自動載入，需 agent config allow
2. **Hook priority**：thread-ownership 未設 priority → 預設排序，可能被其他 hook 搶先

### Runtime

3. **Embedded Pi Agent import**：動態 import src/ 或 dist/，路徑依安裝方式不同
4. **Browser auto-detection**：diffs 依賴本機 Chromium，無瀏覽器 → 只能用 view mode
5. **acpx binary version**：pinned 0.1.15，mismatch → 自動 npm install（如果允許）

### Security

6. **voice-call replay**：10min 快取，Twilio HMAC / Telnyx Ed25519 / Plivo SHA256+nonce
7. **diffs token**：48 hex chars（24 bytes），暴力破解不可行
8. **OTel secret redaction**：export 前過濾，防止密鑰外洩到 tracing backend
9. **thread-ownership fail-open**：forwarder 掛掉 → 允許發送（權衡 availability vs consistency）

### State

10. **voice-call JSONL store**：非 atomic write，crash 可能丟失最後一筆
11. **diffs artifact cleanup**：5 min interval，24hr orphan sweep，crash 不清理需等 OS temp
12. **llm-task temp dir**：finally block 保證清理，但 process kill → 留存 orphan
13. **acpx orphan sessions**：OpenClaw crash → session 留在 agent 端，同名 ensure 可恢復

### Integration

14. **Session key 碰撞**：voice-call 用 `voice:{phone}`，同號碼共享 context
15. **MCP proxy argv limit**：大量 mcpServers config base64 編碼可能超 OS 命令列限制
16. **Barge-in race**：voice-call STT transcript 和 TTS playback 可能時序衝突

---

## 12. 關鍵常量速查

| 常量 | 值 | 來源 |
|------|---|------|
| voice-call max concurrent | 10 | voice-call config |
| voice-call max duration | 3600s | voice-call config |
| voice-call replay cache | 10min / 10K entries | webhook-security.ts |
| voice-call body limit | 1MB | webhook.ts |
| voice-call media chunk | 160 bytes (20ms@8kHz) | telephony-audio.ts |
| acpx pinned version | 0.1.15 | ensure.ts |
| acpx queue TTL | 0.1s | config.ts |
| acpx abort kill delay | 250ms | process.ts |
| diffs artifact TTL | 1800s (30min) | store.ts |
| diffs max TTL | 21600s (6hr) | store.ts |
| diffs cleanup interval | 5min | store.ts |
| diffs ID length | 20 hex (10 bytes) | store.ts |
| diffs token length | 48 hex (24 bytes) | store.ts |
| diffs remote rate limit | 40 fails / 60s | http.ts |
| diffs patch file limit | 128 files | render.ts |
| diffs patch line limit | 120K lines | render.ts |
| diffs input limit | 512 KiB (before/after) | tool.ts |
| diffs patch limit | 2 MiB | tool.ts |
| llm-task default timeout | 30s | llm-task-tool.ts |
| thread-ownership mention TTL | 5min (300000ms) | index.ts |
| thread-ownership fetch timeout | 3s | index.ts |
| otel log severity mapping | 10-60 (Pino levels) | service.ts |

---

## 13. C# 概念對照

| Extension 概念 | C# 對應 |
|---------------|---------|
| voice-call CallManager | `StateMachine<CallState>` (Stateless library) |
| voice-call NormalizedEvent | Strategy pattern + Adapter（provider-agnostic） |
| voice-call Provider 介面 | `IVoipProvider`（interface + DI injection） |
| voice-call Webhook verify | `HMACSHA1`/`Ed25519` + `IAuthenticationHandler` |
| voice-call Media Stream | `WebSocketHandler` middleware |
| voice-call mu-law codec | `System.Buffers` + custom encoder |
| acpx AcpRuntime | `IHostedService` + `Process.Start()` |
| acpx Handle encoding | `Base64Url` + `JsonSerializer` |
| acpx async event stream | `IAsyncEnumerable<T>` |
| acpx Windows cmd resolution | `ProcessStartInfo.UseShellExecute` 替代方案 |
| acpx MCP proxy | `StreamReader/StreamWriter` pipe |
| diffs DiffArtifactStore | `ITempFileProvider` + TTL cache |
| diffs PlaywrightScreenshotter | `Selenium.WebDriver` + `Screenshot()` |
| diffs HTTP handler | ASP.NET `MapGet()` + `IResultFilter` (CSP) |
| diffs Shadow DOM hydration | Blazor component hydration |
| llm-task JSON-only LLM | `HttpClient.PostAsJsonAsync()` + `JsonDocument` |
| llm-task AJV validation | `JsonSchema.Net` / `NJsonSchema` |
| thread-ownership hook | `IMessageFilter` / middleware pipeline |
| diagnostics-otel | `System.Diagnostics` ActivitySource + Meter |
| open-prose skill delivery | Resource-embedded scripts（`EmbeddedResource`） |
| shared test helpers | xUnit `TheoryData` / shared fixtures |

---

## 14. Extension 複雜度對照表

| Extension | Files | LOC | 註冊項 | 外部依賴 | 複雜度 |
|-----------|-------|-----|--------|---------|--------|
| voice-call | ~68 | ~9.9K | Gateway×5 + Tool + CLI + Service | ws, commander, zod, typebox | ★★★★★ |
| diffs | ~28 | ~4.2K | Tool + HTTP + Hook | @pierre/diffs, playwright-core, typebox | ★★★★ |
| acpx | ~23 | ~4.8K | Service | acpx CLI (npm) | ★★★ |
| diagnostics-otel | 5 | ~1.1K | Service | @opentelemetry/* | ★★★ |
| llm-task | 6 | ~537 | Tool (optional) | typebox, ajv | ★★ |
| thread-ownership | 3 | ~341 | Hook×2 | (none) | ★★ |
| open-prose | 5+70 skills | ~7.5K | (none — skill delivery) | (none) | ★ (code) / ★★★★ (content) |
| shared | 2 | ~79 | (none — test only) | (none) | ★ |
