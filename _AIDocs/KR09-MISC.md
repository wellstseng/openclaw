# 09-MISC — 其他模組

> 來源：openclaw-knowledge-base.md §14 + F058 §14 + F060 src/ 模組詳細章節

---

## 1. Cron 系統

### CronSchedule 型別（3 種）

```typescript
type CronSchedule =
  | { type: "cron";     expression: string; }     // cron 表達式（6/7 欄位）
  | { type: "interval"; intervalMs: number; }      // 固定間隔（毫秒）
  | { type: "once";     runAt: number; };          // 單次執行（Unix ms）
```

### computeNextRunAtMs() / computePreviousRunAtMs()

```typescript
function computeNextRunAtMs(schedule: CronSchedule, fromMs?: number): number;
function computePreviousRunAtMs(schedule: CronSchedule, fromMs?: number): number;
// fromMs：從指定時間點計算（預設 now）
```

### croner year-rollback bug workaround

```typescript
// Bug 描述：croner 函式庫在特定時區（如 Asia/Shanghai）計算下次執行時
// 若跨年邊界，可能回傳過去的時間（year rollback）
// Workaround：
// 1. 若 computeNextRunAtMs 回傳值 < now → 觸發 bug
// 2. 重新計算：nextRun = computeNextRunAtMs(schedule, wrongResult + 1000)
// 3. 記錄警告日誌

// 受影響時區（已知）：
const CRONER_BUGGY_TIMEZONES = [
  "Asia/Shanghai",
  "Asia/Taipei",
  "Asia/Tokyo",
  "Asia/Seoul",
  // 其他 UTC+8 以上時區
];
```

### JobKind

```typescript
type JobKind =
  | "isolated-agent"    // 獨立 Agent 執行（最常用）
  | "session-reaper"    // Session 清理任務
  | "heartbeat-policy"  // Heartbeat 政策檢查
  | "custom";           // 自定義任務
```

### Session Reaper

```typescript
// session-reaper：定期清理過期 sessions
// 執行條件：Session TTL 超過 + refCount = 0
// 執行頻率：依配置（通常每小時）
```

---

## 2. Browser Automation

### ProfileRuntimeState

```typescript
interface ProfileRuntimeState {
  profileId: string;         // 瀏覽器 profile ID
  status: ProfileStatus;     // 目前狀態
  lastTargetId?: string;     // 最後訪問的 target（sticky 特性）
}

type ProfileStatus =
  | "idle"       // 閒置
  | "active"     // 活躍中
  | "loading"    // 載入中
  | "error";     // 發生錯誤
```

### BrowserProfileActions

```typescript
interface BrowserProfileActions {
  navigate(url: string): Promise<void>;
  click(selector: string): Promise<void>;
  type(selector: string, text: string): Promise<void>;
  screenshot(): Promise<Buffer>;
  evaluate<T>(fn: () => T): Promise<T>;
  waitForSelector(selector: string, options?: WaitOptions): Promise<void>;
  close(): Promise<void>;
}
```

### BrowserControl 介面

```typescript
interface BrowserControl {
  createProfile(options?: BrowserProfileOptions): Promise<string>; // 回傳 profileId
  getProfile(profileId: string): BrowserProfileActions;
  listProfiles(): ProfileRuntimeState[];
  destroyProfile(profileId: string): Promise<void>;
}
```

### lastTargetId sticky 特性

```typescript
// sticky 特性：記住最後一次導航的 target（tab）
// 再次使用同一 profile 時，繼續在同一 tab 操作
// 避免每次都開新 tab → 保留 session cookies / localStorage
```

---

## 3. Canvas Host

### CanvasHostOpts

```typescript
interface CanvasHostOpts {
  workingDir: string;         // canvas 工作目錄
  liveReload?: boolean;       // 啟用 live reload（chokidar 監聽）
  port?: number;              // HTTP server port
}
```

### A2UI / Canvas 架構

```typescript
// A2UI（Agent to UI）：Agent 輸出結構化 UI
// Canvas 架構：
// 1. Agent 輸出 canvas 指令（JSON）
// 2. CanvasHostServer 接收並渲染
// 3. chokidar 監聽檔案變化 → liveReload
// 4. injectCanvasLiveReload()：注入 WebSocket live reload 腳本
```

---

## 4. Media Understanding

### 預設模型

```typescript
// 多媒體理解的預設 LLM 模型
const MEDIA_UNDERSTANDING_DEFAULTS = {
  DEFAULT_AUDIO_MODELS: [
    "claude-opus-4-6",      // 主要（Anthropic）
    "gpt-5-mini",           // 備選（OpenAI）
    "gemini-3-flash-preview", // 備選（Google）
  ],
  DEFAULT_IMAGE_MODELS: [
    "claude-opus-4-6",
    "gpt-5-mini",
    "gemini-3-flash-preview",
  ],
};
```

### 重要常數

```typescript
const MEDIA_UNDERSTANDING = {
  DEFAULT_MAX_BYTES: /* 媒體最大大小 */,
  TIMEOUT_MS: /* 超時時間 */,
};
```

### AUTO_*_KEY_PROVIDERS

```typescript
// 自動選擇 provider 的 key 名稱
// AUTO_IMAGE_KEY_PROVIDERS：支援圖片理解的 provider 清單
// AUTO_AUDIO_KEY_PROVIDERS：支援音訊理解的 provider 清單
```

### MediaUnderstandingScopeDecision

```typescript
type MediaUnderstandingScopeDecision =
  | "inline"     // 直接嵌入 message（小媒體）
  | "tool_call"  // 透過工具呼叫
  | "skip";      // 跳過（不支援的格式）
```

---

## 5. Media

### MediaAttachment 型別

```typescript
interface MediaAttachment {
  id: string;                // 附件 ID
  mimeType: string;          // MIME type（如 "image/jpeg"）
  filename?: string;         // 原始檔名
  data?: Buffer;             // 原始資料（本地）
  url?: string;              // 遠端 URL
  size?: number;             // 大小（bytes）
  width?: number;            // 圖片寬度（px）
  height?: number;           // 圖片高度（px）
  duration?: number;         // 音訊/影片長度（秒）
}
```

### MEDIA_TOKEN_RE

```typescript
// 識別訊息中的媒體 token
const MEDIA_TOKEN_RE = /\[media:([^\]]+)\]/g;
// 格式：[media:{attachmentId}]
```

### normalizeMediaSource()

```typescript
// 正規化媒體來源（統一處理 URL/base64/Buffer/File）
function normalizeMediaSource(source: MediaSource): NormalizedMedia;
```

---

## 6. Link Understanding

### extractLinksFromMessage()

```typescript
// 從訊息文字中提取 URL
function extractLinksFromMessage(text: string): string[];
// 過濾：
// 1. 去重
// 2. 移除 SSRF 危險 URL（使用 GuardedFetch 預驗證）
// 3. 只保留 http:// 和 https://
```

---

## 7. TTS（Text-to-Speech）

### 4 種 TTS Provider

```typescript
type TtsProvider =
  | "elevenlabs"     // ElevenLabs（雲端，高品質）
  | "openai-tts"     // OpenAI TTS
  | "google-tts"     // Google Cloud TTS
  | "local";         // 本地 TTS（系統語音）
```

### 重要常數

```typescript
const TTS_CONSTANTS = {
  DEFAULT_ELEVENLABS_BASE_URL: "https://api.elevenlabs.io",
  TEMP_FILE_CLEANUP_DELAY_MS: /* 臨時檔案清理延遲 */,
  VALID_VOICE_ID_RE: /^[a-zA-Z0-9]{10,40}$/,  // ElevenLabs voice ID 格式
};
```

---

## 8. Terminal（LOBSTER_PALETTE 顏色）

```typescript
// src/terminal/index.ts
const LOBSTER_PALETTE = {
  accent:      "#FF5A2D",  // Lobster 橘紅（品牌色）
  background:  "#1A1A1A",  // 深色背景
  foreground:  "#F0F0F0",  // 淺色前景
  success:     "#4CAF50",  // 成功（綠）
  warning:     "#FFC107",  // 警告（黃）
  error:       "#F44336",  // 錯誤（紅）
  info:        "#2196F3",  // 資訊（藍）
  dim:         "#666666",  // 暗色（次要文字）
  // ... 其他顏色
};
// 來源：Lobster 品牌設計系統
```

---

## 9. TUI（Terminal UI）

### TuiOptions

```typescript
interface TuiOptions {
  agentId: string;
  sessionKey?: string;
  streamingMode?: "on" | "off";
  useColor?: boolean;
  width?: number;
}
```

### 重要型別

```typescript
type ResponseUsageMode = "full" | "compact" | "hidden";

interface SessionInfo {
  sessionKey: string;
  agentId: string;
  turnCount: number;
  createdAt: number;
  lastActiveAt: number;
}

interface TuiStateAccess {
  getSession(): SessionInfo | null;
  getUsage(): TokenUsage;
  isStreaming(): boolean;
}
```

### 7 個 TUI 子模組

```
src/tui/
├── chat-view.ts       # 對話檢視
├── status-bar.ts      # 狀態列
├── input-handler.ts   # 輸入處理
├── streaming-view.ts  # Streaming 顯示
├── events.ts          # 事件處理
├── colors.ts          # 顏色主題（使用 LOBSTER_PALETTE）
└── renderer.ts        # 渲染引擎
```

---

## 10. Process 管理

### CommandLane enum（重申）

```typescript
enum CommandLane {
  Main     = "main",
  Cron     = "cron",
  Subagent = "subagent",
  Nested   = "nested",
}
```

### 5 個 Process 子模組

```
src/process/
├── child-process-bridge.ts  # 子程序通訊橋
├── command-queue.ts         # 命令佇列
├── kill-tree.ts             # 終止程序樹
├── restart-recovery.ts      # 重啟復原
└── index.ts                 # 主入口
```

---

## 11. Node Host（Companion App）

### SystemRunParams

```typescript
interface SystemRunParams {
  nodeId: string;
  capabilities: string[];    // 此 node 支援的能力
  gatewayUrl: string;        // 連接的 Gateway URL
  authToken: string;         // 認證 token
}
```

### RunResult

```typescript
interface RunResult {
  exitCode: number;
  stdout: string;
  stderr: string;
  durationMs: number;
}
```

### 7 個 Node Host 子模組

```
src/node-host/
├── runtime.ts           # Node runtime
├── capabilities.ts      # 能力宣告
├── exec-handler.ts      # 命令執行處理
├── file-handler.ts      # 檔案操作處理
├── process-handler.ts   # 程序管理
├── skill-bins.ts        # Skill bins 管理
└── index.ts
```

---

## 12. Pairing（配對）

### PairingChallengeParams

```typescript
interface PairingChallengeParams {
  agentId: string;
  channelId: string;
  expiresAt: number;         // 過期時間（Unix ms）
  nonce: string;             // 隨機 nonce
  metadata?: Record<string, unknown>;
}
```

### issuePairingChallenge()

```typescript
async function issuePairingChallenge(
  params: PairingChallengeParams
): Promise<{
  setupCode: string;  // 8位數 setup code
  qrData: string;     // QR code 資料（JSON encoded）
  deepLink: string;   // 深連結 URL
}>;
```

### QR 碼/通知服務特性

```typescript
// QR 碼：base64 PNG 圖片
// 通知服務：push notification 傳送 setup code
// 流程：
// 1. issuePairingChallenge() → 生成 setupCode + QR
// 2. 用戶掃描 QR 或輸入 setupCode
// 3. 驗證 challenge → 建立 pairing
// 4. 儲存 pairing 到 secrets store
```

---

## 13. Secrets Store

```typescript
// src/secrets/ 子模組：
// apply/                  → 套用 secrets 到環境
// auth-store-paths/       → 認證儲存路徑
// target-registry-*/      → 目標 registry（per platform）
// runtime-config-collectors/ → 執行時配置收集
```

---

## 14. Logging

### LoggerSettings / LogTransport

```typescript
interface LoggerSettings {
  level: "debug" | "info" | "warn" | "error";
  subsystem?: string;        // 子系統名稱（用於過濾）
  transports?: LogTransport[];
}

interface LogTransport {
  type: "console" | "file" | "external";
  filePath?: string;         // type=file 時的路徑
  externalFn?: Function;     // type=external 時的處理函式
}
```

### 重要常數

```typescript
const LOG_CONSTANTS = {
  MAX_LOG_AGE_MS: 24 * 60 * 60 * 1000,          // 24 小時（log 保留時間）
  DEFAULT_MAX_LOG_FILE_BYTES: 500 * 1024 * 1024, // 500MB（單檔上限）
};
```

### 外部 Transport

```typescript
// tslog 整合（使用 tslog 函式庫）
// externalTransports：支援掛載自訂 transport
// 用途：OpenTelemetry / Datadog / ELK 等監控整合
```

---

## 15. i18n（多語言支援）

```typescript
// 目前支援的語言（依配置）
// 預設語言：English
// 語言切換：透過 Agent 配置 language 欄位
// System Prompt section 13（language）控制回應語言
```

---

## 16. Daemon（背景 Process 管理）

### GatewayServiceRuntime

```typescript
interface GatewayServiceRuntime {
  start(): Promise<void>;
  stop(): Promise<void>;
  restart(): Promise<void>;
  status(): DaemonStatus;
  pid?: number;
}
```

### 平台特定實作

```typescript
// macOS：launchd
// - launchctl load/unload plist
// - launchd-restart-handoff：平滑重啟（不中斷連線）

// Windows：schtasks-exec
// - Task Scheduler 管理
// - schtasks /create /run /delete
```

---

## 17. Wizard（新手引導）

```typescript
// wizard extension：新手引導流程
// 觸發條件：首次啟動 + 無配置檔
// 流程步驟：
// 1. 選擇 Provider（Anthropic/OpenAI/其他）
// 2. 輸入 API Key
// 3. 設定 Agent 名稱
// 4. 選擇 Channel（Discord/Telegram/其他）
// 5. 配置 Channel credentials
// 6. 測試連線
// 7. 生成配置檔
```

---

## 18. Web（WhatsApp Web Provider）

```typescript
// src/web/：WhatsApp Web QR 登入
// 流程：
// 1. 開啟 Puppeteer 瀏覽器
// 2. 導航到 web.whatsapp.com
// 3. 掃描 QR code（WhatsApp app）
// 4. 擷取 session cookies
// 5. 持久化 cookies（避免重複掃碼）
// vcard：支援 WhatsApp vCard 格式解析
```

---

## 19. Diagnostics（OpenTelemetry）

```typescript
// diagnostics-otel extension：
// 整合 OpenTelemetry 追蹤與指標
// Spans：Gateway 請求、Agent 執行、LLM 呼叫
// Metrics：response time、token usage、error rate
// Exporters：OTLP/Jaeger/Zipkin（依配置）
```

---

## 20. Diffs Extension

```typescript
// diffs extension：
// 程式碼 diff 工具
// 支援：unified diff、side-by-side diff
// 用途：Agent 展示程式碼變更
```

---

## 21. Markdown 系統

### MarkdownStyle / MarkdownIR

```typescript
type MarkdownStyle = "default" | "discord" | "telegram" | "whatsapp" | "plain";

interface MarkdownIR {
  // 中間表示層（Intermediate Representation）
  // 平台無關的 markdown AST
  type: string;
  children?: MarkdownIR[];
  value?: string;
  attrs?: Record<string, string>;
}
```

### 6 個子模組

```
src/markdown/
├── render.ts        # 平台特定渲染（Discord/Telegram/WhatsApp）
├── whatsapp.ts      # WhatsApp markdown 格式（*bold*, _italic_）
├── tables.ts        # Table 處理（Discord code block 模式）
├── fences.ts        # Code fence 處理
├── code-spans.ts    # Inline code 處理
└── frontmatter.ts   # Frontmatter 解析（YAML）
```
