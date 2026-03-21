# 20-NATIVE-APPS.md — iOS + macOS + Android 原生 App 深入

> Phase 6-2 | 掃描 `apps/ios/` + `apps/macos/` + `apps/android/` + `apps/shared/`
> 共 ~500+ files | iOS (Swift 6.0) + macOS (Swift 6.2) + Android (Kotlin 2.2) + Shared (Swift SPM)

---

## 一、三平台架構鳥瞰

```
┌─────────────────────────────────────────────────────────────────────┐
│                        OpenClaw Gateway                            │
│              (Node.js, ws://host:port/api/gateway)                 │
└────────┬──────────────┬──────────────┬──────────────────────────────┘
         │ WS (node)    │ WS (operator)│ WS (node+operator)
         │              │              │
    ┌────▼────┐   ┌─────▼─────┐   ┌───▼────────┐
    │ iOS App │   │ macOS App │   │ Android App│
    │ SwiftUI │   │ MenuBar   │   │ Compose    │
    │ iOS 18+ │   │ macOS 15+ │   │ SDK 31+    │
    └─────────┘   └───────────┘   └────────────┘
         │              │              │
         └──────────────┼──────────────┘
                        │
              ┌─────────▼─────────┐
              │ OpenClawKit       │
              │ (Shared Swift SPM)│
              │ + CanvasA2UI JS   │
              └───────────────────┘
```

### 角色模型

三平台 App 都扮演 **Gateway Node**（裝置能力端點），不內嵌 Node.js runtime：

| 角色 | Session | 用途 |
|------|---------|------|
| **Node** | `role=node`, `scopes=read:device` | 接收 `node.invoke.*` 指令（相機、位置、螢幕等） |
| **Operator** | `role=operator`, `scopes=read:chat,write:chat` | 聊天、Talk mode、config sync |

iOS/macOS 使用 **雙 WebSocket session**（分離職責），Android 也是雙 session 架構。

---

## 二、Shared 共用層（OpenClawKit）

### 2.1 SPM 結構

```
apps/shared/OpenClawKit/
├── Package.swift                  (Swift 6.2, iOS 18+ / macOS 15+)
├── Sources/
│   ├── OpenClawProtocol/          (3 files — Gateway 通訊協定 v3)
│   ├── OpenClawKit/               (85+ files — 核心 SDK)
│   └── OpenClawChatUI/            (15 files — SwiftUI Chat 元件)
└── Tools/CanvasA2UI/              (Lit-based A2UI renderer)
```

### 2.2 三個 Library Module

| Module | 檔案數 | 職責 |
|--------|-------|------|
| **OpenClawProtocol** | 3 | Gateway Protocol v3 models：`ConnectParams`/`HelloOk`/`RequestFrame`/`ResponseFrame`/`Snapshot` + `AnyCodable` 彈性 JSON + `WizardHelpers` |
| **OpenClawKit** | 85+ | Gateway 連線（`GatewayChannelActor`/`GatewayNodeSession`）、Device Identity（Curve25519）、38+ Device Command 定義、Auth、Media、TTS、Bonjour、Canvas A2UI |
| **OpenClawChatUI** | 15 | SwiftUI Chat View/ViewModel/Composer/MarkdownRenderer + `OpenClawChatTransport` protocol |

### 2.3 Device Command 協定（共用定義）

所有命令遵循 `OpenClawXyzCommand` enum + `OpenClawXyzParams` struct + `OpenClawXyzPayload` struct 三元組：

| 分類 | 命令 | 說明 |
|------|------|------|
| camera | list / snap / clip | 列出裝置、拍照（JPEG/PNG）、錄影（MP4） |
| canvas | present / hide / navigate / eval / snapshot | Canvas WebView 控制 |
| canvas.a2ui | push / pushJSONL / reset | A2UI 渲染指令（v0.8 schema） |
| location | get | GPS + network 位置 |
| screen | record | 螢幕錄影（ReplayKit/MediaProjection） |
| device | status / info | 電池/溫度/儲存/網路 |
| contacts | search / add | 通訊錄 |
| calendar | events / add | 日曆 |
| reminders | list / add | 提醒（iOS only） |
| motion | activity / pedometer | 運動感測 |
| photos | latest | 最新照片 |
| talk | ptt.* | Push-to-talk 語音 |
| watch | notify / quick.reply | Apple Watch（iOS only） |
| system | run / notify / which | 系統指令（macOS only） |
| chat | push | 文字訊息 + TTS |
| sms | send | 簡訊（Android only） |
| notifications | list / actions | 通知欄讀取（Android only） |

### 2.4 Gateway 連線層（Shared）

```
GatewayChannelActor (actor)
  ├─ WebSocket 連線管理（URLSessionWebSocketTask）
  ├─ RequestFrame → ResponseFrame RPC
  ├─ Auto-reconnect（800ms → ×1.7 → 15s max）
  ├─ Heartbeat/keepalive
  └─ AsyncStream<GatewayPush> 事件訂閱

GatewayNodeSession
  ├─ 封裝 GatewayChannelActor
  ├─ connect(url, token, password, role, scopes, caps)
  ├─ request(method, params, timeout) → Data
  └─ subscribeServerEvents() → AsyncStream
```

### 2.5 Device Identity & Auth

- **Curve25519 keypair** → 首次啟動生成 → 存 `~/.openclaw/identity/device.json`
- **deviceId** = SHA256(publicKey)
- **Auth Flow**：Gateway challenge → Ed25519 簽名回應 → token 授權
- **DeviceAuthStore**：role-based token 持久化（`device-auth.json`）
- **Keychain**（iOS/macOS）/ EncryptedSharedPreferences（Android）儲存 TLS fingerprints + auth tokens

### 2.6 Chat UI（SwiftUI 共用元件）

```swift
protocol OpenClawChatTransport {
    func requestHistory(sessionKey) async throws → OpenClawChatHistoryPayload
    func sendMessage(...) async throws → OpenClawChatSendResponse
    func events() → AsyncStream<OpenClawChatTransportEvent>
    func listSessions() → OpenClawChatSessionsListResponse
    func abortRun() / requestHealth()
}

@Observable class OpenClawChatViewModel  // 主狀態機
OpenClawChatView                         // SwiftUI 主 View
OpenClawChatComposer                     // 輸入區
ChatMarkdownRenderer                     // Markdown → SwiftUI (Textual 0.3.1)
```

iOS 和 macOS 各自實作 `OpenClawChatTransport`（`IOSGatewayChatTransport` / `WebChatManager`），底層都走 WebSocket RPC。

---

## 三、iOS App 深入

### 3.1 技術規格

| 項目 | 值 |
|------|-----|
| Language | Swift 6.0 (strict concurrency) |
| UI | SwiftUI (iOS 18+) |
| Build | XcodeGen (`project.yml`) |
| Bundle ID | `ai.openclaw.ios` |
| Version | 2026.3.7 / Build 20260307 |
| 依賴 | OpenClawKit (SPM) + Swabble (local, 語音) |
| Targets | App + ShareExtension + ActivityWidget + WatchApp + WatchExtension + Tests |

### 3.2 架構圖

```
OpenClawApp (@main)
  ├─ AppDelegate (UIApplicationDelegate)
  │   ├─ APNs token 註冊
  │   ├─ Silent push wake
  │   └─ BGAppRefreshTask
  │
  ├─ NodeAppModel (@Observable, @MainActor) ← 核心狀態機 (~2000 lines)
  │   ├─ nodeGateway: GatewayNodeSession      (role=node)
  │   ├─ operatorGateway: GatewayNodeSession   (role=operator)
  │   ├─ screen: ScreenController              (WebView canvas)
  │   ├─ camera: CameraController (actor)      (AVFoundation)
  │   ├─ locationService                       (CLLocationManager)
  │   ├─ contactsService / calendarService / remindersService / motionService
  │   ├─ voiceWake: VoiceWakeManager           (SFSpeechRecognizer)
  │   ├─ talkMode: TalkModeManager             (STT + TTS)
  │   ├─ liveActivity: LiveActivityManager     (iOS 16+ ActivityKit)
  │   └─ buildCapabilityRouter() → NodeCapabilityRouter
  │
  ├─ GatewayConnectionController (@Observable)
  │   ├─ GatewayDiscoveryModel (NWBrowser, _openclaw-gw._tcp)
  │   ├─ GatewayServiceResolver (NWServiceResolver)
  │   ├─ GatewayHealthMonitor
  │   ├─ GatewaySettingsStore (Keychain persistence)
  │   ├─ TCPProbe (port connectivity)
  │   └─ TLS TOFU trust prompt
  │
  └─ UI (SwiftUI)
      ├─ RootCanvas (onboarding vs main routing)
      ├─ RootTabs (ScreenTab / VoiceTab / ChatSheet / SettingsTab)
      ├─ OnboardingWizardView (Welcome → Mode → Connect → Auth → Success)
      ├─ StatusPill (connection indicator)
      └─ TalkOrbOverlay (voice feedback)
```

### 3.3 Gateway 連線流程

```
1. GatewayDiscoveryModel
   │  Bonjour NWBrowser → _openclaw-gw._tcp
   │  解析 TXT records (displayName, lanHost, port, tlsEnabled, sha256)
   ▼
2. 使用者選擇 Gateway
   │  GatewayConnectionController.connectDiscoveredGateway()
   │  DNS 解析 → TCP probe → TLS fingerprint 檢查
   ▼
3. TLS TOFU (Trust-On-First-Use)
   │  首次連線：顯示 TrustPromptAlert → 使用者核准 → 存 Keychain
   │  後續連線：pin 比對 fingerprint
   ▼
4. buildGatewayURL() → ws(s)://host:port/api/gateway
   ▼
5. 雙 Session 建立
   ├─ Node Session: GatewayNodeSession.connect(role=node)
   └─ Operator Session: GatewayNodeSession.connect(role=operator)
```

### 3.4 Canvas WebView + A2UI

**ScreenController** (`@Observable`): 管理 WKWebView lifecycle

```
ScreenWebView (UIViewRepresentable)
  ├─ WKWebView (non-persistent data store, black background)
  ├─ ScreenNavigationDelegate → 攔截 openclaw:// deep links
  └─ CanvasA2UIActionMessageHandler → 接收 JS postMessage

A2UI 行動流：
  Canvas JS → webkit.messageHandlers.openclawCanvasA2UIAction.postMessage({...})
           → CanvasA2UIActionMessageHandler 驗證 origin
           → NodeAppModel.handleCanvasA2UIAction()
           → 組合 agent message → operator gateway agent.request RPC
           → 回傳 JS status: JS_openclawCanvasA2UIActionStatus(actionId, ok, error)

A2UI 自動載入：
  gateway 連線成功 → resolveA2UIHostURL() RPC
  → TCP probe canvas host (2.5s timeout)
  → navigate to https://canvas-host/__openclaw__/a2ui/?platform=ios
```

### 3.5 Voice Wake + Talk Mode

**VoiceWakeManager** (`@Observable`):
- `AVAudioEngine` 持續錄音 → `SFSpeechAudioBufferRecognitionRequest` 語音辨識
- 比對 `triggerWords[]`（可自訂）→ 觸發後擷取指令文字 → 送 gateway
- 與 TalkMode 互斥搶 mic：`setSuppressedByTalk(true)` / `suspendForExternalAudioCapture()`

**TalkModeManager** (`@Observable`):
- 雙向語音：STT（SFSpeechRecognizer）+ TTS（ElevenLabs PCM / MP3 fallback）
- 三模式：Continuous（自動靜音偵測 0.9s）、Push-to-Talk、Interrupt-on-speech
- Gateway RPC：`talk.send`（上行 PCM）、`talk` event（下行 text + audio）

### 3.6 Device Capability Routing

**NodeCapabilityRouter** — 命令分派器：

```swift
typealias Handler = (BridgeInvokeRequest) async throws -> BridgeInvokeResponse
// 由 NodeAppModel.buildCapabilityRouter() 構建
// 映射：camera.snap → CameraController.snap()
//       location.get → LocationService.currentLocation()
//       canvas.eval → ScreenController.eval(javaScript:)
//       ... 30+ 命令
```

### 3.7 背景 / 前景 生命週期

```
→ .background:
  ├─ 停止 discovery
  ├─ 暫停 voice wake（釋放麥克風）
  ├─ 開始 25s grace period
  └─ 設定 backgroundReconnectLease

→ .active:
  ├─ 恢復 discovery
  ├─ Healthcheck operator 連線（2s timeout）
  ├─ 失敗 → 重連雙 session
  ├─ 恢復 voice wake
  └─ 恢復 talk mode
```

### 3.8 Push Notifications (APNs)

```
App 啟動 → registerForRemoteNotifications()
         → didRegisterForRemoteNotifications → token 存 UserDefaults
         → Gateway 連線後 → push.apns.register RPC（operator session）
         → Gateway 發 silent push → didReceiveRemoteNotification
         → handleSilentPushWake() → 重連 if needed

BGAppRefreshTask: ai.openclaw.ios.bgrefresh → 每 15min health check
```

### 3.9 Share Extension

- `ShareViewController` 在任何 App 的 Share Sheet 中出現
- 提取 text/URL/images（最多 3 張, 5MB/張, JPEG 轉碼）
- 透過 **SharedContentPayload** 讀取 App Group Container 的 gateway config
- 建立臨時 GatewayNodeSession → 送 `agent.request` → 關閉 extension

### 3.10 Watch App

- WatchOS app via `WatchConnectivity.WCSession`
- iOS → Watch：`openclawWatchNotify` message（通知 + 最多 4 個 action buttons）
- Watch → iOS → Gateway：`watch.quick.reply` event
- Watch UI：`WatchInboxView` 顯示通知列表 + action 按鈕

---

## 四、macOS App 深入

### 4.1 技術規格

| 項目 | 值 |
|------|-----|
| Language | Swift 6.2 (strict concurrency) |
| UI | SwiftUI MenuBar Extra (macOS 15+) |
| Build | Swift Package Manager (`Package.swift`) |
| Bundle ID | `ai.openclaw.mac` |
| Version | 2026.3.7 |
| Products | `OpenClaw` (menubar app) + `openclaw-mac` (CLI tool) |
| Libraries | `OpenClawIPC` + `OpenClawDiscovery` + `OpenClawProtocol` |
| 依賴 | MenuBarExtraAccess + swift-subprocess + swift-log + Sparkle 2.8.1+ + Peekaboo + OpenClawKit + Swabble |

### 4.2 架構圖

```
OpenClawApp (@main, MenuBarExtra)
  ├─ AppDelegate (NSApplicationDelegateAdaptor)
  │   ├─ 重複執行個體偵測
  │   ├─ Background services 啟動
  │   └─ Termination cleanup
  │
  ├─ AppState (@Observable, @MainActor) ← 中央狀態
  │   ├─ connectionMode: .unconfigured / .local / .remote
  │   ├─ isPaused, launchAtLogin, showDockIcon
  │   ├─ swabble (voice wake) 設定
  │   ├─ talk mode 設定
  │   ├─ canvas 設定
  │   ├─ execApprovalMode (安全等級)
  │   └─ remote target/identity/url
  │
  ├─ GatewayProcessManager (local mode)
  │   ├─ Attach-first：先嘗試連接既有 gateway
  │   ├─ Spawn：寫 launchd plist → launchctl load → health check (6s)
  │   ├─ 日誌：~/Library/Logs/OpenClaw/
  │   └─ Disable marker: ~/.openclaw/disable-launchagent
  │
  ├─ GatewayConnection (actor)
  │   ├─ 單一 WebSocket 連線（GatewayChannelActor）
  │   ├─ RPC: request(method, params, timeoutMs) → Data
  │   ├─ AsyncStream<GatewayPush> 事件訂閱
  │   ├─ Auto-recovery (local mode): 失敗 → auto-spawn → retry 3×
  │   └─ Protocol v3
  │
  ├─ ConnectionModeCoordinator
  │   ├─ Local → GatewayProcessManager
  │   ├─ Remote SSH → RemoteTunnelManager → tunnel
  │   └─ Remote Direct → wss:// mTLS
  │
  ├─ CanvasManager + CanvasWindowController
  │   ├─ WKWebView (NSWindowController)
  │   ├─ CanvasSchemeHandler → openclaw-canvas://session/path
  │   ├─ A2UI JS bridge (WKScriptMessageHandler)
  │   ├─ File watcher auto-reload
  │   └─ Panel mode (menubar anchor) / Window mode
  │
  ├─ WebChatManager
  │   ├─ Panel mode / Window mode
  │   ├─ OpenClawChatView (from OpenClawChatUI)
  │   └─ Session persistence (UserDefaults)
  │
  ├─ VoiceWakeRuntime (actor)
  │   ├─ Swabble integration (SFSpeechRecognizer)
  │   ├─ RMS VAD (noise floor calibration)
  │   ├─ Chime playback on trigger
  │   └─ VoiceWakeGlobalSettingsSync
  │
  ├─ TalkModeController
  │   ├─ Phases: idle → listening → thinking → speaking
  │   └─ Overlay + live transcription
  │
  ├─ MacNodeModeCoordinator
  │   ├─ Node.js gateway 的 node 角色
  │   ├─ 暴露：canvas, screen, camera, location capabilities
  │   └─ MacNodeScreenCommands / CameraCaptureService / ScreenRecordService
  │
  └─ UI
      ├─ CritterStatusLabel (animated menubar icon)
      │   ├─ Working (pulse), Idle (static), Sleeping (paused)
      │   └─ Left-click → chat, Right-click → menu
      ├─ SettingsRootView (tabbed: General/Channels/Canvas/Voice/Exec/Cron/Instances/Debug)
      ├─ DeepLinks (openclaw://agent?message=...&key=...)
      └─ HoverHUD (contextual action card)
```

### 4.3 三種連線模式

| Mode | 機制 | 說明 |
|------|------|------|
| **unconfigured** | — | 未設定，顯示 onboarding |
| **local** | launchd + port 18789 | 本機 Node.js gateway，App 管理生命週期 |
| **remote** | SSH tunnel / Direct wss:// | 遠端 gateway，SSH identity 或 mTLS |

**Local Gateway Lifecycle**:

```
GatewayProcessManager
  1. resolveGatewayCommand() → 找 Node.js runtime
  2. 寫 ~/Library/LaunchAgents/ai.openclaw.gateway.plist
  3. launchctl load → 啟動 gateway
  4. Health check loop (每 1s, max 6s)
  5. 成功 → .running(port, pid)
  6. 失敗 → .failed(reason)

  Logs → ~/Library/Logs/OpenClaw/gateway.log
```

### 4.4 Canvas 方案處理器

```
URL Scheme: openclaw-canvas://session/path
  → CanvasSchemeHandler.webView(start:)
  → 解析 session + path
  → ~/Library/Application Support/OpenClaw/canvas/{session}/{path}
  → Directory traversal 防護（standardizedFileURL.hasPrefix()）
  → 回傳 URLResponse + MIME type

Per-session 隔離：每個 session 獨立目錄 + WKWebView
CanvasFileWatcher：CoalescingFSEventsWatcher 偵測檔案異動 → auto-reload
```

### 4.5 Deep Link 安全

```
openclaw://agent?message=...&sessionKey=...&deliver=...&channel=...&key=...

安全規則：
  - 無 key → 確認彈窗 + max 240 chars
  - 有 key → 驗證 ephemeral key (Canvas A2UI) 或 configured key → max 20K chars
  - 1s throttle 防 spam
  - Confirmation prompt for untrusted sources
```

### 4.6 Exec Approvals

- Shell 命令需要核准（configurable policy）
- `ExecApprovalsPromptServer` + `ExecApprovalsGatewayPrompter` → 彈窗確認
- 等級：Auto-approve / Prompt / Deny

### 4.7 自動更新（Sparkle）

- Sparkle v2.8.1+，僅 Developer ID 簽名版啟用
- 自動背景檢查 + 靜默下載 + 安裝提示
- Ad-hoc dev builds 自動停用

### 4.8 Config File Watcher

- 監控 `~/.openclaw/config.json`
- 外部工具可修改 → App 即時套用
- 支援 connection mode + remote 設定覆蓋

---

## 五、Android App 深入

### 5.1 技術規格

| 項目 | 值 |
|------|-----|
| Language | Kotlin 2.2.21 (JVM target 17) |
| UI | Jetpack Compose (100%, 無 XML layouts) |
| Build | Gradle (Kotlin DSL) |
| Namespace | `ai.openclaw.app` |
| compileSdk | 36 (Android 15 preview) |
| minSdk | 31 (Android 12) |
| targetSdk | 36 |
| versionName | 2026.3.7 |
| versionCode | 202603070 |
| ABI | armeabi-v7a, arm64-v8a, x86, x86_64 |
| 核心依賴 | Compose BOM 2026.02 + CameraX 1.5.2 + OkHttp3 5.3.2 + BouncyCastle 1.83 + dnsjava 3.6.4 |

### 5.2 架構圖

```
NodeApp (Application)
  └─ lazy NodeRuntime singleton (~700 lines, core orchestrator)

MainActivity (Single Activity, Compose)
  ├─ MainViewModel → 暴露 NodeRuntime 給 UI
  ├─ Permission requesters (camera, SMS, screen)
  └─ setContent { RootScreen() }

NodeRuntime
  ├─ operatorSession: GatewaySession (OkHttp3 WS, role=operator)
  ├─ nodeSession: GatewaySession (OkHttp3 WS, role=node)
  ├─ InvokeDispatcher → routes node.invoke to handlers
  │   ├─ CameraHandler (CameraX)
  │   ├─ LocationHandler (GPS + network)
  │   ├─ CanvasController (WebView)
  │   ├─ A2UIHandler (v0.8 schema)
  │   ├─ ScreenHandler (MediaProjection)
  │   ├─ SmsHandler
  │   ├─ NotificationsHandler (NotificationListenerService)
  │   ├─ DeviceHandler
  │   ├─ ContactsHandler
  │   ├─ CalendarHandler
  │   ├─ PhotosHandler
  │   ├─ MotionHandler (SensorManager)
  │   └─ SystemHandler
  ├─ ChatController (session + message state)
  ├─ TalkModeManager (ElevenLabs TTS + speech recognition)
  ├─ VoiceWakeManager (on-device wake word)
  ├─ MicCaptureManager (VAD)
  └─ ConnectionManager (capability/TLS negotiation)

NodeForegroundService
  ├─ FOREGROUND_SERVICE_TYPE_DATA_SYNC + MICROPHONE
  ├─ Persistent notification (connection status)
  └─ START_STICKY (auto-restart)
```

### 5.3 Gateway 連線協定

**Discovery**:

```
GatewayDiscovery
  ├─ Android NSD API → _openclaw-gw._tcp (local mDNS)
  ├─ dnsjava → Wide-Area DNS-SD ($OPENCLAW_WIDE_AREA_DOMAIN)
  └─ 回傳 GatewayEndpoint (host, port, TLS hints, stableId)
```

**WebSocket RPC** (OkHttp3):

```json
{
  "jsonrpc": "2.0",
  "method": "node.connect",
  "params": { "role": "node", "scopes": [...], "caps": [...] },
  "id": "<uuid>"
}
```

**Device Identity (Ed25519, BouncyCastle)**:

```
首次啟動 → 生成 Ed25519 keypair
  publicKey: 32 bytes base64
  privateKey: PKCS#8 base64
  deviceId: SHA256(publicKey)
  存檔: filesDir/openclaw/identity/device.json (mode 0600)
```

**TLS TOFU**:
- 首次連線 → 儲存 SHA256 fingerprint
- 後續連線 → 強制 pin 比對
- mDNS TXT records 可提示 TLS 支援

### 5.4 Canvas WebView + A2UI

**CanvasController** (~250 lines):

```
WebView 管理:
  - @Volatile webView instance → Compose UI 掛載
  - 預設載入: file:///android_asset/CanvasScaffold/scaffold.html
  - URL 導航: canvas.present / canvas.navigate

命令:
  - canvas.eval(js) → evaluateJavascript() + suspendCancellableCoroutine
  - canvas.snapshot(format, quality, maxWidth) → WebView 截圖 base64
  - canvas.a2ui.push/pushJSONL → __openclaw.applyMessages()
  - canvas.a2ui.reset → 清除 A2UI state

A2UI Hydration:
  - 偵測 __openclaw.isReady JS global
  - 未就緒 → 每 120ms 重試, max 6s
  - A2UI host URL: {gateway-canvas-host}/__openclaw__/a2ui/?platform=android
```

**A2UIHandler**:
- 驗證 v0.8 schema（reject v0.9+ `createSurface`）
- 支援 `messages[]` array 和 JSONL（line-delimited JSON）

### 5.5 Foreground Service

```kotlin
NodeForegroundService : Service()
  - startForeground(DATA_SYNC + MICROPHONE)
  - Notification:
    - 連線狀態 ("Connected" / status text)
    - Server name
    - Mic status (if voice enabled)
    - Action: Tap → foreground app, "Disconnect"
  - START_STICKY → 被殺自動重啟
```

### 5.6 Notification Listener

- `DeviceNotificationListenerService` 繫結系統 NotificationListenerService
- 讀取所有 active notifications
- 支援 snapshot 查詢 + action dispatch（open/dismiss/reply）
- 需使用者手動在 Settings 授權

### 5.7 Permissions（36 項宣告）

| 分類 | 權限 |
|------|------|
| 網路 | INTERNET, ACCESS_NETWORK_STATE |
| Foreground Service | DATA_SYNC, MICROPHONE, MEDIA_PROJECTION |
| 通知 | POST_NOTIFICATIONS (Android 13+) |
| Discovery | NEARBY_WIFI_DEVICES (neverForLocation), FINE/COARSE_LOCATION |
| 相機 | CAMERA, RECORD_AUDIO |
| SMS | SEND_SMS |
| Media | READ_MEDIA_IMAGES, VISUAL_USER_SELECTED |
| 通訊錄 | READ/WRITE_CONTACTS |
| 日曆 | READ/WRITE_CALENDAR |
| 感測器 | ACTIVITY_RECOGNITION |

### 5.8 Secure Storage

| 儲存 | 機制 | 內容 |
|------|------|------|
| 一般偏好 | SharedPreferences `openclaw.node` | gateway host, camera.enabled, etc. |
| 安全偏好 | EncryptedSharedPreferences `openclaw.node.secure` | gateway token, TLS fingerprints |
| Device Identity | 檔案 `filesDir/openclaw/identity/device.json` | Ed25519 keypair |

### 5.9 UI 結構（Jetpack Compose）

```
RootScreen
  ├─ onboardingCompleted = false → OnboardingFlow (4 steps)
  └─ onboardingCompleted = true → PostOnboardingTabs
      ├─ ConnectTab (discovery + manual + pairing)
      ├─ ChatTab (message list + composer + Markdown CommonMark)
      ├─ VoiceTab (mic mode + transcript)
      ├─ ScreenTab (WebView canvas host)
      └─ SettingsTab (camera/location/voice toggles + about)

State: StateFlow<T> → collectAsState() 驅動 recomposition
Theme: Material 3 (MobileUiTokens + OpenClawTheme)
```

---

## 六、三平台對照表

### 6.1 架構對照

| 維度 | iOS | macOS | Android |
|------|-----|-------|---------|
| Language | Swift 6.0 | Swift 6.2 | Kotlin 2.2 |
| UI Framework | SwiftUI | SwiftUI MenuBarExtra | Jetpack Compose |
| Min OS | iOS 18.0 | macOS 15.0 | Android 12 (SDK 31) |
| Build System | XcodeGen | SPM | Gradle KTS |
| State | @Observable | @Observable | StateFlow |
| Concurrency | async/await + actors | async/await + actors | Coroutines |
| WS Client | URLSessionWebSocketTask | URLSessionWebSocketTask | OkHttp3 |
| Crypto | CryptoKit (Curve25519) | CryptoKit (Curve25519) | BouncyCastle (Ed25519) |
| TLS Store | Keychain | Keychain | EncryptedSharedPreferences |
| Discovery | NWBrowser Bonjour | NWBrowser Bonjour | Android NSD + dnsjava |
| Camera | AVFoundation | AVFoundation/Peekaboo | CameraX |
| Location | CLLocationManager | CLLocationManager | LocationManager |
| TTS | ElevenLabs + AVSpeechSynth | ElevenLabs + NSSpeechSynth | ElevenLabs + system TTS |
| STT | SFSpeechRecognizer | SFSpeechRecognizer | Android SpeechRecognizer |
| Canvas | WKWebView | WKWebView | android.webkit.WebView |
| Local Gateway | — | launchd plist | — |

### 6.2 連線模式對照

| 模式 | iOS | macOS | Android |
|------|-----|-------|---------|
| Local gateway | ✗ | ✓ (launchd) | ✗ |
| Remote gateway | ✓ (WS direct) | ✓ (SSH tunnel / Direct wss) | ✓ (WS direct) |
| Bonjour discovery | ✓ | ✓ | ✓ (NSD) |
| Wide-area DNS-SD | ✗ | ✓ (Tailscale) | ✓ (dnsjava) |
| TLS TOFU | ✓ | ✓ | ✓ |

### 6.3 功能矩陣

| 功能 | iOS | macOS | Android |
|------|-----|-------|---------|
| Chat UI | ✓ (OpenClawChatUI) | ✓ (OpenClawChatUI) | ✓ (Compose native) |
| Canvas A2UI | ✓ (WKWebView) | ✓ (WKWebView) | ✓ (WebView) |
| Voice Wake | ✓ (SFSpeech) | ✓ (Swabble) | ✓ (on-device) |
| Talk Mode | ✓ (STT+TTS) | ✓ (STT+TTS) | ✓ (STT+TTS) |
| Camera snap/clip | ✓ | ✓ (Peekaboo) | ✓ (CameraX) |
| Screen Record | ✓ (ReplayKit) | ✓ (Peekaboo) | ✓ (MediaProjection) |
| Location | ✓ | ✓ | ✓ |
| Contacts | ✓ | ✗ | ✓ |
| Calendar | ✓ | ✗ | ✓ |
| Reminders | ✓ | ✗ | ✗ |
| Motion sensors | ✓ | ✗ | ✓ |
| Photos | ✓ | ✗ | ✓ |
| Watch | ✓ (WatchConnectivity) | ✗ | ✗ |
| Share Extension | ✓ | ✗ | ✗ |
| Live Activity | ✓ (ActivityKit) | ✗ | ✗ |
| APNs Push | ✓ | ✗ | ✗ |
| SMS | ✗ | ✗ | ✓ |
| Notification listener | ✗ | ✗ | ✓ |
| System exec | ✗ | ✓ | ✗ |
| Deep links | ✗ | ✓ (openclaw://) | ✗ |
| Auto-update | ✗ | ✓ (Sparkle) | ✗ |
| Foreground Service | — | — | ✓ (START_STICKY) |
| Dock icon toggle | — | ✓ (LSUIElement) | — |
| Exec approvals | ✗ | ✓ | ✗ |

---

## 七、共通設計模式

### 7.1 雙 Session 模式

三平台皆使用兩個 WebSocket session 分離 node（裝置指令）和 operator（聊天/config）職責。好處：

- Scope 隔離（device access vs chat access）
- 獨立重連策略
- 單一 GatewayConnectConfig 作為兩 session 的連線參數來源

### 7.2 Capability 廣播

連線時 App 將可用能力清單送給 Gateway：

```json
{
  "caps": ["canvas", "camera", "screen", "location", "device", ...],
  "commands": ["camera.snap", "camera.clip", "location.get", ...],
  "permissions": {"camera": true, "location": "whenInUse"}
}
```

Gateway 根據此清單決定哪些 `node.invoke` 指令可發送。

### 7.3 A2UI 跨平台

| 平台 | JS bridge 機制 | postMessage 路徑 |
|------|---------------|-----------------|
| iOS | `webkit.messageHandlers.openclawCanvasA2UIAction` | WKScriptMessageHandler |
| macOS | `webkit.messageHandlers` + `openclaw://` fallback | WKScriptMessageHandler / Deep link |
| Android | `interface.postMessage` | WebView.addJavascriptInterface |

A2UI v0.8 message schema 三平台一致。CanvasA2UI bundle（Lit-based）共用同一份 JS。

### 7.4 TLS TOFU (Trust-On-First-Use)

三平台共通模式：
1. 首次連線 → 取得 server TLS 證書 SHA256 fingerprint
2. 顯示確認 UI → 使用者核准
3. 儲存 fingerprint → 後續連線 pin 比對
4. Fingerprint 不符 → 拒絕連線

### 7.5 Voice 整合模式

```
VoiceWake (持續背景辨識)
  ├─ Audio Engine → PCM buffer → Speech Recognizer → partial transcript
  ├─ 比對 trigger words → 觸發
  ├─ 擷取 post-trigger 語音 → 送 gateway agent.request
  └─ 與 TalkMode 互斥（同時搶 mic 時 voice wake 讓步）

TalkMode (雙向對話)
  ├─ STT: 持續辨識 or PTT → transcript → gateway talk.send
  ├─ TTS: gateway talk event → PCM/MP3 → audio player
  └─ Interrupt: 使用者說話 → 自動停止播放
```

---

## 八、版本管理對照

| 平台 | 檔案 | 欄位 |
|------|------|------|
| iOS | `apps/ios/Sources/Info.plist` | CFBundleShortVersionString / CFBundleVersion |
| iOS Tests | `apps/ios/Tests/Info.plist` | CFBundleShortVersionString / CFBundleVersion |
| macOS | `apps/macos/Sources/OpenClaw/Resources/Info.plist` | CFBundleShortVersionString / CFBundleVersion |
| Android | `apps/android/app/build.gradle.kts` | versionName / versionCode |
| CLI (npm) | `package.json` | version |

目前版本：**2026.3.7**（三平台同步）

---

## 九、邊界條件 / 陷阱

### iOS

1. **背景 25s grace period**：iOS 限制背景 WebSocket 維持，超時自動斷線，前景回來需重連
2. **Mic 互斥**：VoiceWake + TalkMode 搶 AVAudioSession，必須用 suppress/suspend 協調
3. **Share Extension App Group**：ShareGatewayRelaySettings 透過 App Group Container 共享 gateway config，Container ID 必須一致
4. **APNs silent push**：有 24h budget 限制，過度使用會被 iOS throttle
5. **WKWebView non-persistent**：每次啟動清空 cookie/cache，Canvas 不能依賴 localStorage

### macOS

6. **launchd plist 路徑硬編碼**：`~/Library/LaunchAgents/ai.openclaw.gateway.plist`，多使用者環境可能衝突
7. **Deep link 240 char 限制**：無 ephemeral key 時嚴格限制長度防惡意 URL
8. **Sparkle 僅 Developer ID**：Ad-hoc 簽名自動停用自動更新
9. **SSH tunnel 進程管理**：RemoteTunnelManager 依賴本機 Node.js 建 tunnel，Node.js 不可用 = 無法連遠端
10. **CanvasSchemeHandler directory traversal 防護**：用 `standardizedFileURL.hasPrefix()` 阻擋 `../` 攻擊

### Android

11. **Foreground Service 權限 Android 14+**：需 `FOREGROUND_SERVICE_DATA_SYNC` + `FOREGROUND_SERVICE_MICROPHONE` 分開宣告
12. **CameraX 相容性**：跨裝置相機行為不一致，用 CameraX 抽象但仍有邊角 case
13. **EncryptedSharedPreferences 金鑰遷移**：Android Keystore 金鑰損壞 = 無法讀取儲存資料
14. **MediaProjection 前景限制**：螢幕錄影必須在前景且有 user consent
15. **A2UI v0.8 限定**：A2UIHandler 明確 reject v0.9+ 語法（`createSurface`），升級需同步改

### 共通

16. **TLS TOFU 無 revocation**：fingerprint 一旦信任無法自動撤銷，server 換證書需使用者手動重信任
17. **Ed25519 keypair 無備份**：裝置重裝 = 新 deviceId，Gateway 上的舊 node 配對失效
18. **A2UI hydration 6s timeout**：Canvas host 載入慢 → A2UI push 靜默失敗
19. **雙 session 部分斷線**：node session 斷但 operator 正常（或反之）→ 功能部分失效，需偵測並全重連
20. **Protocol v3 版本鎖定**：App 和 Gateway 必須同版本 protocol，不相容 → 連線失敗

---

## 十、關鍵常量速查

### iOS

| 常量 | 值 | 位置 |
|------|-----|------|
| Bundle ID | `ai.openclaw.ios` | Info.plist |
| Bonjour service | `_openclaw-gw._tcp` | GatewayDiscoveryModel |
| Background refresh ID | `ai.openclaw.ios.bgrefresh` | AppDelegate |
| Background grace period | 25s | NodeAppModel |
| Canvas TCP probe timeout | 2.5s | NodeAppModel+Canvas |
| Share image limit | 3 images, 5MB each | ShareViewController |
| Silence detection window | 0.9s | TalkModeManager |
| Voice max capture | 120s | VoiceWakeManager |

### macOS

| 常量 | 值 | 位置 |
|------|-----|------|
| Bundle ID | `ai.openclaw.mac` | Info.plist |
| Default local port | 18789 | GatewayProcessManager |
| LaunchAgent label | `ai.openclaw.gateway` | GatewayProcessManager |
| Health check timeout | 6s | GatewayProcessManager |
| Deep link max (no key) | 240 chars | DeepLinks |
| Deep link max (with key) | 20,000 chars | DeepLinks |
| Sparkle min version | 2.8.1 | Package.swift |
| URL scheme | `openclaw://` | Info.plist |
| Canvas scheme | `openclaw-canvas://` | CanvasSchemeHandler |

### Android

| 常量 | 值 | 位置 |
|------|-----|------|
| Namespace | `ai.openclaw.app` | build.gradle.kts |
| compileSdk | 36 | build.gradle.kts |
| minSdk | 31 | build.gradle.kts |
| versionCode 格式 | YYYYMMDDxx | build.gradle.kts |
| WS connect timeout | 12s | GatewaySession |
| A2UI hydration retry | 120ms, max 6s | CanvasController |
| A2UI protocol version | v0.8 | A2UIHandler |
| Identity storage | `filesDir/openclaw/identity/device.json` | DeviceIdentityStore |

### Shared

| 常量 | 值 | 位置 |
|------|-----|------|
| Gateway Protocol | v3 | OpenClawProtocol |
| WS reconnect base | 800ms | GatewayChannelActor |
| WS reconnect max | 15s | GatewayChannelActor |
| WS reconnect multiplier | ×1.7 | GatewayChannelActor |
| Device auth file | `device-auth.json` | DeviceAuthStore |
| Identity file | `device.json` | DeviceIdentity |
| ElevenLabs dep | ElevenLabsKit 0.1.0 | OpenClawKit Package.swift |
| Chat markdown | Textual 0.3.1 | OpenClawChatUI Package.swift |

---

## 十一、C# 概念對照

| OpenClaw Native | C# / .NET 對等概念 |
|-----------------|-------------------|
| `@Observable` (Swift) | `INotifyPropertyChanged` / `ObservableObject` |
| `StateFlow<T>` (Kotlin) | `ReactiveProperty<T>` / `BehaviorSubject<T>` |
| `actor` (Swift) | `Channel<T>` + single-threaded `SynchronizationContext` |
| `Sendable` protocol | `ImmutableObject` / `readonly record struct` |
| `@MainActor` | `SynchronizationContext.Post()` / `Dispatcher.Invoke()` |
| `async/await` (Swift) | `async/await` (C#, 幾乎相同) |
| `AnyCodable` | `JsonElement` / `JToken` |
| `WKWebView` / `WebView` | `WebView2` (WPF/WinUI) |
| SPM `Package.swift` | `.csproj` + NuGet |
| XcodeGen `project.yml` | `.sln` 生成器 (Nuke Build) |
| `launchd` plist | Windows Service / `sc.exe` |
| Keychain | `ProtectedData` / DPAPI |
| `EncryptedSharedPreferences` | `ProtectedData` / Windows Credential Manager |
| Bonjour / NSD | `Dns.ServiceDiscovery` / mDNS 套件 |
| CameraX | MediaCapture (UWP) / EmguCV |
| `MediaProjection` | Desktop Duplication API |
| Jetpack Compose | MAUI / Avalonia UI |
| SwiftUI `@Environment` | DI Container (`IServiceProvider`) |
| `GatewayNodeSession` | `ClientWebSocket` wrapper |
| `BridgeInvokeRequest` | `JsonRpcRequest` / MediatR `IRequest` |
