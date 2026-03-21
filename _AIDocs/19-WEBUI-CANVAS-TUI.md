# Web UI + Canvas Host + TUI 深入

> Phase 6-1 | 掃描範圍：`ui/` 184 files ~40.3K LOC + `src/canvas-host/` 5 files ~1.1K LOC + `src/tui/` 45 files ~8.4K LOC
> 更新：2026-03-13

---

## 目錄

1. [三個前端總覽](#1-三個前端總覽)
2. [Web UI (Control UI) 架構深入](#2-web-ui-control-ui-架構深入)
3. [Gateway 連線層（瀏覽器端）](#3-gateway-連線層瀏覽器端)
4. [Navigation / 路由系統](#4-navigation--路由系統)
5. [View 體系（12 Tabs）](#5-view-體系12-tabs)
6. [Controllers 層](#6-controllers-層)
7. [狀態管理模式](#7-狀態管理模式)
8. [Build 設定（Vite）](#8-build-設定vite)
9. [CSS / 樣式方案](#9-css--樣式方案)
10. [Canvas Host 系統深入](#10-canvas-host-系統深入)
11. [A2UI Bundle 系統](#11-a2ui-bundle-系統)
12. [TUI（Terminal UI）架構深入](#12-tuiterminal-ui架構深入)
13. [TUI 命令系統](#13-tui-命令系統)
14. [TUI 事件處理](#14-tui-事件處理)
15. [三前端共通 API 層對照](#15-三前端共通-api-層對照)
16. [邊界條件與陷阱](#16-邊界條件與陷阱)
17. [關鍵常量速查](#17-關鍵常量速查)

---

## 1. 三個前端總覽

| 前端 | 目錄 | 技術棧 | LOC | 目標用戶 |
|------|------|--------|-----|---------|
| **Control UI** (Web) | `ui/` | Lit 3 (Web Components) + Vite 7 | ~40.3K | operator 透過瀏覽器 |
| **Canvas Host** | `src/canvas-host/` | Node HTTP + WS + chokidar | ~1.1K | 嵌入式自訂 HTML（iOS/Android node） |
| **TUI** | `src/tui/` | `@mariozechner/pi-tui` | ~8.4K | 開發者透過終端 |

### 共通 API 層

三者都透過 **Gateway WebSocket RPC** 與後端通訊：

```
Browser (Control UI)  ──WebSocket──┐
iOS/Android (Canvas)  ──WebSocket──┤──→ Gateway server-methods.ts (25+ handler groups)
Terminal (TUI)        ──WebSocket──┘     → server-chat.ts → Agent Engine
```

- **協定版本**：Protocol 3（`minProtocol: 3, maxProtocol: 3`）
- **認證**：token / password / device-identity (Ed25519 簽名)
- **角色**：`operator`（Web UI）| `node`（Canvas/mobile）| `webchat`（TUI 也歸 operator）
- **事件模型**：server → client push events，帶 `seq` 序號 + gap detection

---

## 2. Web UI (Control UI) 架構深入

### 技術選型

| 選擇 | 說明 |
|------|------|
| **Lit 3** | 非 React。使用 `@customElement` + `@state()` decorators 的 Web Components |
| **Signal** | `@lit-labs/signals` + `signal-polyfill` + `signal-utils` 為響應式基礎 |
| **Markdown** | `marked` 17.x + `dompurify` 3.x（XSS 防護） |
| **Ed25519** | `@noble/ed25519` — device identity 簽名 |
| **無路由庫** | 手寫 `navigation.ts` + `History.pushState` + `popstate` 事件 |
| **無狀態庫** | 所有 state 集中在 `OpenClawApp` 類別的 `@state()` 屬性上 |

### 元件結構

```
<openclaw-app>          ← ui/src/ui/app.ts (LitElement, ~625 LOC)
  ├── createRenderRoot() → return this  // 不用 Shadow DOM，直接掛全域 CSS
  ├── connectedCallback → handleConnected() → 啟動 WebSocket + polling
  ├── firstUpdated → observeTopbar
  ├── render() → renderApp(this as AppViewState)
  └── 100+ @state() 屬性（每個 tab 獨立 state 分區）
```

### 核心檔案矩陣

| 檔案 | 行數 | 職責 |
|------|------|------|
| `app.ts` | 625 | `<openclaw-app>` LitElement 主元件，所有 state 宣告 |
| `app-render.ts` | 1123 | `renderApp()` — 主渲染函數，組裝 nav + tab content |
| `app-view-state.ts` | 323 | `AppViewState` type — state 完整型別定義 |
| `app-gateway.ts` | 424 | `connectGateway()` — WebSocket event dispatch |
| `app-lifecycle.ts` | 95 | `connectedCallback` / `firstUpdated` / `disconnectedCallback` |
| `app-chat.ts` | - | chat send / abort / queue 邏輯 |
| `app-channels.ts` | - | WhatsApp / Nostr channel UI handlers |
| `app-settings.ts` | - | tab switch, theme, URL sync, settings persistence |
| `app-scroll.ts` | - | chat / logs 自動捲動 + export |
| `app-tool-stream.ts` | - | tool event stream 處理 + compaction status |
| `app-polling.ts` | - | nodes / logs / debug 定時 polling |
| `gateway.ts` | 393 | `GatewayBrowserClient` — 瀏覽器端 WS 客戶端 |
| `navigation.ts` | 166 | Tab 路由、URL 解析、basePath 推斷 |
| `storage.ts` | 123 | `UiSettings` localStorage 持久化 |
| `theme.ts` | 17 | ThemeMode 解析（system/light/dark） |
| `types.ts` | 350+ | 共享型別（channels, sessions, cron, etc.） |

---

## 3. Gateway 連線層（瀏覽器端）

### GatewayBrowserClient（`gateway.ts`）

```
start() → connect()
  ↓
new WebSocket(url)
  ↓ onopen
queueConnect() → sendConnect()
  ↓
  build connect payload:
    - minProtocol/maxProtocol: 3
    - client: { id: "control-ui", mode: "webchat", scopes: [operator.admin/approvals/pairing] }
    - auth: { token, password }
    - device: { id, publicKey, signature (Ed25519), signedAt, nonce }
  ↓
  send({ type: "req", method: "connect", ... })
  ↓ response
  hello-ok → onHello callback → store deviceToken → reset backoff
  error → pendingConnectError → close
  ↓ onclose
  isNonRecoverableAuthError? → stop
  else → scheduleReconnect() (backoff: 800ms → *1.7 → max 15s)
```

### Device Identity（`device-identity.ts`）

- 首次連線時用 `crypto.subtle` 生成 Ed25519 keypair
- 存入 `localStorage`（device ID + private key + public key）
- 每次 connect 時簽名 payload → Gateway 驗證
- **安全上下文限制**：HTTP plain text → 跳過 device identity，fallback 到 token-only

### 事件處理（`app-gateway.ts`）

```
GatewayBrowserClient.onEvent → switch(event) {
  "chat.delta"           → handleChatEvent → 更新 chatStream
  "chat.final"           → handleChatEvent → 載入完整 history
  "chat.tool"            → handleAgentEvent → tool card 更新
  "presence.update"      → 更新 presenceEntries
  "health.update"        → 更新 debugHealth
  "status.update"        → 更新 debugStatus
  "exec.approval.requested" → 加入 approval queue
  "exec.approval.resolved"  → 從 queue 移除
  "update.available"     → 顯示更新通知
  "agents.update"        → 重新載入 agents list
  "sessions.update"      → 重新載入 sessions
}
```

---

## 4. Navigation / 路由系統

### Tab 定義

```typescript
TAB_GROUPS = [
  { label: "chat",     tabs: ["chat"] },
  { label: "control",  tabs: ["overview", "channels", "instances", "sessions", "usage", "cron"] },
  { label: "agent",    tabs: ["agents", "skills", "nodes"] },
  { label: "settings", tabs: ["config", "debug", "logs"] },
]
```

共 **12 個 Tab**，各對應 URL path（`/chat`, `/overview`, `/channels`, ...）。

### 路由機制

- **無路由庫**：`navigation.ts` 手工維護 `TAB_PATHS` ↔ `PATH_TO_TAB` 雙向映射
- `History.pushState` 切換 tab（無頁面重載）
- `window.onpopstate` → `syncTabWithLocation()` 回復 tab
- `basePath` 自動推斷：從 `window.location.pathname` + `__OPENCLAW_CONTROL_UI_BASE_PATH__` 推算
- 默認路徑 `/` → chat tab

---

## 5. View 體系（12 Tabs）

`ui/src/ui/views/` 目錄，63 個檔案。每個 view 是純函數 `render*(state: AppViewState) → TemplateResult`。

| Tab | View 檔案 | 功能 |
|-----|----------|------|
| chat | `chat.ts` (636L) | 對話介面，markdown 渲染，tool cards，附件，queue，sidebar |
| overview | `overview.ts` | 總覽面板（health, presence, update） |
| channels | `channels.ts` + 8 子檔案 | 每頻道獨立 view（Discord/Telegram/Slack/WhatsApp/Signal/iMessage/Nostr/GoogleChat） |
| instances | `instances.ts` | 裝置配對、exec approvals |
| sessions | `sessions.ts` | session list，filter，patch |
| usage | `usage.ts` | token/cost 圖表，time series，session logs |
| cron | `cron.ts` | cron job CRUD，run logs，filters |
| agents | `agents.ts` | agent list，files 編輯，tools catalog，skills，channels per-agent，cron per-agent |
| skills | `skills.ts` | skill 安裝/啟用/API key |
| nodes | `nodes.ts` | 遠端 node 列表 |
| config | `config.ts` + `config-form.*.ts` | JSON schema-driven 設定表單 / raw JSON 編輯器 |
| debug | `debug.ts` | health, status, models, heartbeat, raw RPC call |
| logs | `logs.ts` | 日誌串流，level filter，auto-follow |

### Chat View 特殊機制

- **Markdown 渲染**：`marked` → `dompurify` → lit `html` 模板
- **Tool Cards**：`chat/tool-cards.ts` — tool-start / tool-result / partial-result 即時更新
- **Streaming**：`chatStream` + `chatStreamSegments` 即時 delta 拼接
- **Sidebar**：tool output 可在側邊欄展開，`splitRatio` 可拖曳調整（0.4~0.7）
- **Queue**：離線訊息 queue，重連後自動發送
- **Attachments**：支援附件（`ChatAttachment` 型別）

---

## 6. Controllers 層

`ui/src/ui/controllers/` — 業務邏輯與 Gateway RPC 呼叫的中間層。

| Controller | 檔案 | 主要 RPC calls |
|-----------|------|---------------|
| agents | `agents.ts` | `agents.list`, `agents.config.save` |
| agent-files | `agent-files.ts` | `agents.files.list`, `agents.files.get`, `agents.files.save` |
| agent-identity | `agent-identity.ts` | `agents.identity` |
| agent-skills | `agent-skills.ts` | `skills.status` |
| channels | `channels.ts` | `channels.status` |
| chat | `chat.ts` | `chat.history`, `chat.send`, `chat.abort` |
| config | `config.ts` | `config.get`, `config.save`, `config.apply`, `config.schema`, `update.run` |
| control-ui-bootstrap | `control-ui-bootstrap.ts` | `/api/config` HTTP GET（bootstrap config） |
| cron | `cron.ts` | `cron.list`, `cron.add`, `cron.update`, `cron.remove`, `cron.run`, `cron.runs` |
| debug | `debug.ts` | `status`, `health`, `models.list`, `heartbeat.status`, 任意 RPC |
| devices | `devices.ts` | `devices.list`, `devices.approve`, `devices.reject`, `devices.revoke`, `devices.rotate` |
| exec-approval | `exec-approval.ts` | `exec.approval.resolve` |
| exec-approvals | `exec-approvals.ts` | `exec.approvals.get`, `exec.approvals.save` |
| logs | `logs.ts` | `logs.get` |
| nodes | `nodes.ts` | `nodes.list` |
| presence | `presence.ts` | `presence.list` |
| sessions | `sessions.ts` | `sessions.list`, `sessions.patch`, `sessions.delete` |
| skills | `skills.ts` | `skills.status`, `skills.install`, `skills.update`, `skills.toggle`, `skills.apikey` |

### Controller 模式

```typescript
// 典型 controller 函數
export async function loadAgents(host: { client; agentsLoading; agentsList; agentsError }) {
  if (!host.client) return;
  host.agentsLoading = true;
  host.agentsError = null;
  try {
    const result = await host.client.request("agents.list", {});
    host.agentsList = result;
  } catch (err) {
    host.agentsError = String(err);
  } finally {
    host.agentsLoading = false;
  }
}
```

- 直接修改 host（OpenClawApp）的 `@state()` 屬性 → 觸發 Lit re-render
- 無 Redux / MobX / 任何外部狀態庫
- 錯誤統一用 `*Error: string | null` 欄位

---

## 7. 狀態管理模式

### 單一元件集中 State

```
OpenClawApp (LitElement)
  ├── 100+ @state() 屬性（按 tab 分區）
  ├── 不用 Shadow DOM（createRenderRoot → this）
  ├── render() → renderApp(this as AppViewState)
  │     └── 各 view render 函數收到完整 state snapshot
  └── controllers 直接修改 host 屬性觸發 re-render
```

### State 分區

| 分區 | 前綴 | 範例 |
|------|------|------|
| 連線 | `connected`, `hello`, `lastError` | 連線狀態 |
| Chat | `chat*` | `chatMessages`, `chatStream`, `chatLoading` |
| Config | `config*` | `configRaw`, `configForm`, `configSchema` |
| Channels | `channels*` | `channelsSnapshot`, `whatsapp*` |
| Sessions | `sessions*` | `sessionsResult`, `sessionsFilter*` |
| Usage | `usage*` | `usageResult`, `usageTimeSeries` |
| Cron | `cron*` | `cronJobs`, `cronForm`, `cronRuns` |
| Agents | `agents*` | `agentsList`, `agentFiles*` |
| Skills | `skills*` | `skillsReport`, `skillEdits` |
| Debug | `debug*` | `debugStatus`, `debugHealth` |
| Logs | `logs*` | `logsEntries`, `logsFilterText` |
| Exec | `execApprovals*` | `execApprovalsForm` |

### Settings 持久化

- `localStorage` key: `openclaw.control.settings.v1`
- 保存：`gatewayUrl`, `sessionKey`, `theme`, `chatFocusMode`, `chatShowThinking`, `splitRatio`, `navCollapsed`, `navGroupsCollapsed`, `locale`
- **不保存** token（in-memory only，安全考量）

---

## 8. Build 設定（Vite）

```typescript
// ui/vite.config.ts
export default defineConfig(() => ({
  base: process.env.OPENCLAW_CONTROL_UI_BASE_PATH ?? "./",  // 相對路徑部署
  publicDir: "public",
  build: {
    outDir: "../dist/control-ui",  // 產出到主專案 dist/
    sourcemap: true,
    chunkSizeWarningLimit: 1024,   // 刻意允許較大 chunk
  },
  server: {
    host: true,
    port: 5173,
    strictPort: true,
  },
}));
```

### 依賴策略

- **零 bundled React**：Lit 3 (~17KB gzip) 比 React+ReactDOM (~45KB) 輕
- 從 `../src/` 直接 import 共享型別（`session-types.js`, `config-ui-hints-types.js`, `protocol/*`）
- Production build 產出到 `dist/control-ui/`，Gateway 的 Stage 12 `control-ui-http` 直接 serve
- 測試：`vitest` + `@vitest/browser-playwright`（browser 環境測試）

---

## 9. CSS / 樣式方案

### 純 CSS（無 CSS-in-JS / 無 Tailwind）

```
ui/src/styles.css        — 入口，@import 其他
ui/src/styles/
  ├── base.css           — reset + CSS variables (colors, spacing, fonts)
  ├── layout.css         — 主 layout (621L)
  ├── layout.mobile.css  — 響應式 (374L)
  ├── components.css     — 通用 UI 組件
  ├── config.css         — config form 專用
  ├── chat.css           — chat view 專用
  └── chat/              — chat 子樣式
```

- `createRenderRoot() → return this`：不用 Shadow DOM，全域 CSS 直接生效
- CSS 變數控制主題切換（`data-theme="dark"` / `data-theme="light"`）
- i18n：`ui/src/i18n/` — 自建輕量 i18n，`locales/` 放翻譯檔

---

## 10. Canvas Host 系統深入

### 架構概觀

Canvas Host 是一個可選子系統，讓使用者部署自訂 HTML/JS 到 Gateway 上，供 iOS/Android node 的 WebView 嵌入顯示。

```
                    ┌──────────────────────────┐
                    │   Gateway HTTP Pipeline   │
                    │   Stage 6-8: canvas-*     │
                    └──────┬───────────────────-┘
                           │
            ┌──────────────┼──────────────────┐
            ▼              ▼                  ▼
     canvas-auth      canvas-http         canvas-ws
     (capability       (static files)     (live reload)
      token 驗證)
```

### 兩種運作模式

| 模式 | 入口 | 說明 |
|------|------|------|
| **嵌入模式** | `createCanvasHostHandler()` | Gateway 內建，共用 HTTP server，basePath: `/__openclaw__/canvas` |
| **獨立模式** | `startCanvasHost()` | 獨立 HTTP server，可指定 port 和 listenHost |

### 核心檔案

| 檔案 | 行數 | 職責 |
|------|------|------|
| `server.ts` | ~430 | Canvas host server，HTTP + WS + chokidar file watcher |
| `a2ui.ts` | ~210 | A2UI bundle serving + live-reload script 注入 |
| `file-resolver.ts` | ~40 | 安全路徑解析（防 path traversal） |

### Canvas Host Handler

```typescript
CanvasHostHandler = {
  rootDir: string;       // Canvas 根目錄（default: ~/.openclaw/canvas）
  basePath: string;      // URL 前綴（default: /__openclaw__/canvas）
  handleHttpRequest;     // HTTP GET → 靜態檔案 serve
  handleUpgrade;         // WebSocket upgrade → live reload
  close;                 // cleanup watcher + WS server
}
```

### Live Reload 機制

```
chokidar.watch(rootDir)
  ↓ file change
scheduleReload() (debounce 75ms)
  ↓
broadcastReload() → 所有 WebSocket clients 收到 "reload"
  ↓
瀏覽器/WebView 端：ws.onmessage → location.reload()
```

- **Debounce**：一般 75ms，測試模式 12ms
- **Write stability**：`awaitWriteFinish.stabilityThreshold: 75ms`（避免寫入中途觸發）
- **忽略**：dotfiles + `node_modules/`

### Auth 機制（Gateway 嵌入模式）

- Canvas 路徑受 **scoped capability token** 保護
- `isCanvasPath()` → `authorizeCanvasRequest()` 驗證 `?oc_cap=` query param
- WebSocket upgrade 同樣需要 capability token

---

## 11. A2UI Bundle 系統

A2UI（Agent-to-UI）是 Canvas Host 內建的 UI bundle，提供 iOS/Android node 與 Gateway 的橋接。

### 路徑

```
Canvas static:  /__openclaw__/canvas/*   ← 使用者自訂內容
A2UI bundle:    /__openclaw__/a2ui/*     ← 內建 bundle
Canvas WS:      /__openclaw__/ws         ← live reload WebSocket
```

### Action Bridge（注入到每個 Canvas HTML）

`injectCanvasLiveReload()` 在所有 HTML 回應的 `</body>` 前注入：

```javascript
// Cross-platform bridge
globalThis.OpenClaw = {
  postMessage(payload),       // raw message → node
  sendUserAction(userAction), // structured action → node
};
globalThis.openclawPostMessage = postToNode;
globalThis.openclawSendUserAction = sendUserAction;
```

### 平台偵測

| 平台 | 橋接介面 |
|------|---------|
| **iOS** | `window.webkit.messageHandlers.openclawCanvasA2UIAction.postMessage(raw)` |
| **Android** | `window.openclawCanvasA2UIAction.postMessage(raw)` |
| **Desktop** | 無原生橋接，僅 WebSocket live reload |

### Action 協定

```typescript
sendUserAction({
  name: "hello",              // action 名稱
  surfaceId: "main",          // 畫面區域
  sourceComponentId: "demo.hello", // 來源元件
  context: { t: Date.now() }, // 自訂 context
})
// → postToNode({ userAction: { id: crypto.randomUUID(), ...action } })
```

Gateway 監聽 `openclaw:a2ui-action-status` 事件回報結果。

---

## 12. TUI（Terminal UI）架構深入

### 技術棧

| 選擇 | 說明 |
|------|------|
| `@mariozechner/pi-tui` | Terminal UI 框架（TUI, Container, Text, Loader, CustomEditor, ChatLog） |
| `GatewayChatClient` | 基於 `GatewayClient`（Node WS）的 TUI 專用封裝 |
| 純 Node.js | stdin/stdout raw mode，無 blessed / ink |

### 元件樹

```
TUI (ProcessTerminal)
  └── root: Container
        ├── header: Text         ← "openclaw tui - ws://... - agent main - session main"
        ├── chatLog: ChatLog     ← 訊息歷史（user/assistant/system/tool）
        ├── statusContainer: Container
        │     └── Loader | Text  ← "sending • 3s | connected" / idle
        ├── footer: Text         ← 使用量摘要
        └── editor: CustomEditor ← 輸入框（自動完成、歷史）
```

### runTui() 主流程

```typescript
async function runTui(opts: TuiOptions) {
  // 1. 載入 config
  const config = loadConfig();

  // 2. 連接 Gateway
  const client = await GatewayChatClient.connect({ url, token, password });

  // 3. 建立 TUI
  const tui = new TUI(new ProcessTerminal());
  // ... 建立 header, chatLog, editor, footer, statusContainer

  // 4. 設定事件 handlers
  const eventHandlers = createEventHandlers({ chatLog, tui, state, ... });
  const commandHandlers = createCommandHandlers({ client, chatLog, tui, state, ... });

  // 5. 設定 editor submit handler
  editor.onSubmit = createEditorSubmitHandler({
    editor, handleCommand, sendMessage, handleBangLine
  });

  // 6. Gateway 事件監聽
  client.on("chat.delta", eventHandlers.handleChatDelta);
  client.on("chat.final", eventHandlers.handleChatFinal);
  client.on("chat.tool", eventHandlers.handleToolEvent);
  // ...

  // 7. 啟動 TUI render loop
  tui.start();

  // 8. Auto-message（if --message flag）
  if (autoMessage && !autoMessageSent) { ... }
}
```

### 核心檔案矩陣

| 檔案 | 行數 | 職責 |
|------|------|------|
| `tui.ts` | 752 | `runTui()` 主入口 + 各種 helper（burst coalescer, session key, ctrl-c） |
| `gateway-chat.ts` | 380 | `GatewayChatClient` — Gateway WS 連接、chat send/abort、session 管理 |
| `tui-command-handlers.ts` | 490 | 30+ slash command 處理器 |
| `tui-event-handlers.ts` | 310 | Gateway 事件 → ChatLog 更新 |
| `tui-formatters.ts` | 340 | Markdown → ANSI、token usage 格式化、text 清理 |
| `tui-session-actions.ts` | 400 | session 切換、agent 切換、history 載入 |
| `tui-stream-assembler.ts` | 190 | delta streaming 組裝器 |
| `tui-local-shell.ts` | 140 | `!command` bang-line → local shell 執行 |
| `tui-status-summary.ts` | 100 | status 格式化 |
| `tui-overlays.ts` | 16 | overlay 管理（agent picker, model picker 等） |
| `tui-types.ts` | 80 | TUI 專用型別定義 |
| `tui-waiting.ts` | 45 | 等待動畫訊息 |
| `commands.ts` | 155 | slash command 定義 + 自動完成 |

### TUI Components

| 元件 | 檔案 | 職責 |
|------|------|------|
| `ChatLog` | `components/chat-log.ts` | 訊息串列（user/assistant/system/tool） |
| `CustomEditor` | `components/custom-editor.ts` | 多行輸入框 + 歷史 + 自動完成 |
| `AssistantMessage` | `components/assistant-message.ts` | assistant 訊息渲染（markdown → ANSI） |
| `UserMessage` | `components/user-message.ts` | user 訊息渲染 |
| `ToolExecution` | `components/tool-execution.ts` | tool 呼叫結果卡片 |
| `MarkdownMessage` | `components/markdown-message.ts` | markdown 文字渲染 |
| `HyperlinkMarkdown` | `components/hyperlink-markdown.ts` | OSC8 terminal hyperlinks |
| `FilterableSelectList` | `components/filterable-select-list.ts` | 可篩選選擇列表（agent/model picker） |
| `SearchableSelectList` | `components/searchable-select-list.ts` | 搜尋選擇列表 |
| `FuzzyFilter` | `components/fuzzy-filter.ts` | fuzzy 搜尋 |

---

## 13. TUI 命令系統

### 輸入處理流程

```
editor.onSubmit(text)
  ├── text.startsWith("!") → handleBangLine → local shell 執行
  ├── text.startsWith("/") → handleCommand
  │     ├── parseCommand(input) → { name, args }
  │     └── switch(name) { case "help": ... case "agent": ... }
  └── else → sendMessage → client.send({ sessionKey, message, ... })
```

### Slash Commands 清單

| 命令 | 功能 |
|------|------|
| `/help` | 顯示幫助文字 |
| `/status` | Gateway 狀態摘要 |
| `/agent [id]` | 切換 agent（無參數 → picker） |
| `/agents` | 打開 agent picker overlay |
| `/session [key]` | 切換 session |
| `/sessions` | 打開 session picker overlay |
| `/model [name]` | 切換模型 |
| `/models` | 打開 model picker overlay |
| `/think [level]` | 設定 thinking level |
| `/verbose [on/off]` | tool 詳細輸出 |
| `/reasoning [on/off]` | 推理顯示 |
| `/elevated [on/off/ask/full]` | 提升模式 |
| `/activation [mention/always]` | 啟動模式 |
| `/usage [off/tokens/full]` | usage footer |
| `/deliver` | 設定 deliver mode |
| `/clear` | 清除 chat log |
| `/abort` | 中止當前回覆 |
| `/exit` | 退出 TUI |

### Burst Coalescer（多行貼上處理）

```
Windows Git Bash / macOS iTerm → 快速連續 submit 被合併
  ↓
createSubmitBurstCoalescer({
  burstWindowMs: 50,  // 50ms 內的連續 submit → 合併為一個
  enabled: true       // Windows Git Bash / macOS iTerm 自動啟用
})
```

### Ctrl-C 處理

```
resolveCtrlCAction({ hasInput, now, lastCtrlCAt, exitWindowMs: 1000 })
  ├── hasInput → "clear"（清空輸入框）
  ├── 1000ms 內連按兩次 → "exit"
  └── else → "warn"（顯示 "press again to exit"）
```

---

## 14. TUI 事件處理

### GatewayChatClient（`gateway-chat.ts`）

```typescript
class GatewayChatClient {
  connection: GatewayClient;  // 底層 WS client

  static async connect(opts) → GatewayChatClient
  send(opts: ChatSendOptions) → Promise  // chat.send RPC
  abort(sessionKey, runId)    → Promise  // chat.abort RPC
  listSessions(params)        → Promise  // sessions.list RPC
  patchSession(params)        → Promise  // sessions.patch RPC
  request(method, params)     → Promise  // generic RPC
  on(event, handler)                     // event listener
}
```

### Chat Event 處理（`tui-event-handlers.ts`）

```
createEventHandlers() → {
  handleChatDelta(event)
    → streamAssembler.append(text)
    → chatLog.updateAssistant(assembled, runId)
    → setActivityStatus("streaming")

  handleChatFinal(event)
    → chatLog.finalizeAssistant(text, runId)
    → setActivityStatus("idle")
    → refreshSessionInfo()  // 更新 usage 統計

  handleToolEvent(event)
    → chatLog.startTool(toolCallId, toolName, args)
    → chatLog.updateToolResult(toolCallId, result, { partial, isError })
    → setActivityStatus("running")
}
```

### TUI Stream Assembler

- 累積 delta 片段，組裝完整 assistant 回覆
- 處理 `seq` 序號，偵測 gap
- session 切換時 reset

---

## 15. 三前端共通 API 層對照

### 連線方式

| 前端 | Client 類別 | WebSocket 層 | 認證 |
|------|-----------|-------------|------|
| Control UI | `GatewayBrowserClient` | 瀏覽器 `WebSocket` API | token + password + Ed25519 device identity |
| TUI | `GatewayChatClient` → `GatewayClient` | `ws` npm package | token + password（config/env/CLI flag） |
| Canvas | 注入腳本直接 WS | 瀏覽器 `WebSocket` API | capability token (`?oc_cap=`) |

### RPC 共享方法

| 功能 | RPC Method | Control UI | TUI |
|------|-----------|-----------|-----|
| 聊天 | `chat.send` / `chat.abort` | ✅ | ✅ |
| 歷史 | `chat.history` | ✅ | ✅ |
| Agents | `agents.list` | ✅ | ✅ |
| Sessions | `sessions.list` / `sessions.patch` | ✅ | ✅ |
| Config | `config.get` / `config.save` | ✅ | ❌ |
| Channels | `channels.status` | ✅ | ❌ |
| Cron | `cron.*` | ✅ | ❌ |
| Skills | `skills.*` | ✅ | ❌ |
| Debug | `status` / `health` / `models.list` | ✅ | via `/status` |
| Logs | `logs.get` | ✅ | ❌ |
| Exec Approval | `exec.approval.resolve` | ✅ | ❌ |

### 事件共享

| 事件 | Control UI | TUI | Canvas |
|------|-----------|-----|--------|
| `chat.delta` | ✅ | ✅ | ❌ |
| `chat.final` | ✅ | ✅ | ❌ |
| `chat.tool` | ✅ | ✅ | ❌ |
| `presence.update` | ✅ | ❌ | ❌ |
| `health.update` | ✅ | ❌ | ❌ |
| `exec.approval.*` | ✅ | ❌ | ❌ |
| `update.available` | ✅ | ❌ | ❌ |
| Canvas live reload | ❌ | ❌ | ✅（`"reload"` raw message） |

---

## 16. 邊界條件與陷阱

### Web UI

| 陷阱 | 說明 |
|------|------|
| **無 Shadow DOM** | `createRenderRoot() → this` — 全域 CSS 有衝突風險，但簡化了開發 |
| **100+ @state** | 所有 state 集中在一個 LitElement，改任何 state 都觸發 `render()` |
| **Token 不持久化** | `localStorage` 不存 token，重開瀏覽器需重新輸入（安全設計） |
| **Secure Context 限制** | HTTP（非 localhost）→ 無 `crypto.subtle` → 跳過 device identity → 需 `controlUi.allowInsecureAuth` |
| **basePath 推斷** | 從 URL pathname 反推 basePath，嵌套在 reverse proxy 後需正確設定 `__OPENCLAW_CONTROL_UI_BASE_PATH__` |
| **Auto-reconnect 漏洞** | `AUTH_TOKEN_MISMATCH` 刻意不阻止重連（device token fallback 流程），但可能導致閃爍 |
| **Chunk size** | `chunkSizeWarningLimit: 1024` — build 產出可能 >500KB，設計上可接受 |

### Canvas Host

| 陷阱 | 說明 |
|------|------|
| **rootDir 安全** | `resolveFileWithinRoot()` 防 path traversal，但自訂 HTML 中的 JS 不受限制 |
| **Live reload 風暴** | 大量檔案同時改動 → debounce 75ms 緩解，但極端情況仍可能導致多次 reload |
| **chokidar watcher 崩潰** | 目錄過大 → watcher error → live reload 永久禁用（僅 log 警告） |
| **OPENCLAW_SKIP_CANVAS_HOST** | 環境變數可禁用，但 test 環境預設禁用（`NODE_ENV=test` / `VITEST`） |
| **A2UI root 解析** | 嘗試 10+ 候選路徑找 `a2ui/index.html` + `a2ui/a2ui.bundle.js`，找不到 → 503 |

### TUI

| 陷阱 | 說明 |
|------|------|
| **Windows Git Bash 貼上** | 多行貼上被拆成快速連續 submit → burst coalescer 合併（50ms window） |
| **EBADF setRawMode** | 終端關閉後 `setRawMode` 失敗 → `isIgnorableTuiStopError()` 安全忽略 |
| **Backspace 重複** | 某些終端快速 backspace 產生雙重事件 → `createBackspaceDeduper()` 8ms 去重 |
| **Session scope** | `per-sender`（default）vs `global` — 影響 session key 生成邏輯 |
| **localRunIds 上限** | 最多追蹤 200 個 runId，超過淘汰最舊的 |
| **Gateway 斷線** | pairing required → 需執行 `openclaw devices list` 核准配對 |

---

## 17. 關鍵常量速查

| 常量 | 值 | 位置 |
|------|---|------|
| `CANVAS_HOST_PATH` | `/__openclaw__/canvas` | `canvas-host/a2ui.ts` |
| `CANVAS_WS_PATH` | `/__openclaw__/ws` | `canvas-host/a2ui.ts` |
| `A2UI_PATH` | `/__openclaw__/a2ui` | `canvas-host/a2ui.ts` |
| WS reconnect backoff | 800ms → ×1.7 → max 15s | `ui/src/ui/gateway.ts` |
| WS connect close code | 4008 | `ui/src/ui/gateway.ts` |
| Protocol version | 3 | `gateway/protocol/index.ts` |
| Burst coalesce window | 50ms | `src/tui/tui.ts` |
| Backspace dedupe window | 8ms | `src/tui/tui.ts` |
| Ctrl-C exit window | 1000ms | `src/tui/tui.ts` |
| Canvas debounce | 75ms (test: 12ms) | `canvas-host/server.ts` |
| localStorage key | `openclaw.control.settings.v1` | `ui/src/ui/storage.ts` |
| Vite dev port | 5173 | `ui/vite.config.ts` |
| Build output | `dist/control-ui/` | `ui/vite.config.ts` |
| UI file count | 184 .ts files | `ui/src/` |
| TUI file count | 45 .ts files | `src/tui/` |
| Canvas file count | 5 .ts files | `src/canvas-host/` |
