# _AIDocs 變更紀錄

## 2026-03-21 — 知識包 v1.0 匯入
- 新增 10 份 KR-（Knowledge Reference）前綴文件，來源：`openclaw-knowledge-v1.0.tar.gz`
- KR 系列與既有 deep-dive 文件互補：deep-dive 側重逐檔原始碼分析，KR 側重跨模組參考速查
- 涵蓋：概覽（76 常數）、Agent 引擎、Discord 40 gates、Session/ACP、Memory/LanceDB、20 Channels、17 Providers、Security/SSRF、44 Extensions + CLI、其他模組
- 更新 _INDEX.md 新增 KR 區段

## 2026-03-14 — Phase 7-2：Build + Test + CI/CD（最終 Session）
- 掃描 `pnpm-workspace.yaml` + `tsdown.config.ts` + `tsconfig*.json` + `scripts/` 40+ files + `.github/workflows/` 9 workflows + `vitest.*.config.ts` 8 configs + `Dockerfile` + `docker-compose.yml` + `git-hooks/` + `.oxlintrc.json` + `.oxfmtrc.jsonc` + `.swiftlint.yml` + `.swiftformat` + `knip.config.ts`
- Build Pipeline：11 步驟完整管線（A2UI SHA256 cached bundle → tsdown 60+ entries → tsc plugin-sdk DTS → facade .d.ts → asset copy → metadata → legacy shim），`pnpm build` 單指令全流程
- tsdown 設定：platform=node、60+ entry points（core + CLI + daemon + 7 lazy channels + 48 plugin SDK + extension API + bundled hooks）
- TypeScript：strict=true、target=es2023、module=NodeNext、noEmit=true（tsc 只檢查，tsdown 產出）
- Dev Runner：`run-node.mjs` 6 觸發條件 smart rebuild（.buildstamp git HEAD + mtime tracking），`watch-node.mjs` Node.js --watch mode
- UI Build：Lit 3 + Vite 7.3.1 → dist/control-ui/，prepack hook 串接
- Vitest 測試系統：8 config files（base/unit/e2e/live/gateway/channels/extensions/ui），V8 coverage 70% lines/functions/statements + 55% branches
- test-parallel.mjs：智慧平行化引擎 — 4 test profiles（normal/low/serial/max）、vmForks vs forks 自動偵測（Node 24+/Windows/memory）、89 isolated files 分離 lane、host-aware worker scaling（CI macOS 1 worker → local ≥96GiB 14 workers）、load-aware throttling、sharding
- Live Test 5 層：Direct Model + Gateway Agent + Android Node + Setup Token + CLI Backend
- Docker Tests 5 scripts + E2E 5 scripts（onboard/doctor/gateway-network/plugins/qr-import）
- Performance：test-perf-budget.mjs（wall-clock budget + regression %）、test-hotspots.mjs（top-N slowest files）
- CI 主管線（ci.yml）：smart scope detection → build-artifacts → checks(Node+Bun) + check(types+lint+format) + checks-windows(6 shards) + macos(TS+Swift) + android(Gradle) + skills-python + secrets + release-check，Blacksmith fleet（16/32 vCPU Ubuntu/Windows/ARM）
- Docker Release：multi-stage build（ext-deps → build → runtime），default+slim variant，multi-arch(amd64+arm64) manifest，digest pinning
- 9 GitHub Workflows：ci/docker-release/install-smoke/labeler/auto-response/codeql/stale/sandbox-smoke/workflow-sanity
- Code Quality：Oxlint(type-aware) + Oxfmt + 12 custom lint scripts + pre-commit hook(auto-fix+re-stage) + SwiftLint/SwiftFormat
- Dead Code：knip + ts-prune + ts-unused-exports 三工具
- Scripts 工具鏈：committer(安全 commit helper) + release-check(5 檢查項) + sync-plugin-versions + package-mac-app.sh(multi-arch Swift build+sign) + docs-i18n(Go+Claude PI) + protocol-gen(JSON Schema+Swift codegen)
- Release SOP：version bump(5 位置) → plugins:sync → build+check+test → release:check → npm publish → macOS package+notarize+appcast → GitHub release+tag → Docker auto
- 15 個邊界條件/陷阱 + 20+ 關鍵常量 + C# 概念對照
- **全計畫 7 Phase / 16 Sessions 完成**

## 2026-03-13 — Phase 7-1：CLI 命令全覽 + Infra 深入
- 掃描 `src/commands/` 223 files + `src/cli/` ~40 files + `src/infra/` 100+ files（含 outbound/ 52 files + update-*.ts 6 files + exec-*.ts 16 files），共 350+ files
- CLI Bootstrap：完整 14 步啟動序列（normalizeWindowsArgv → parseCliProfileArgs → loadDotEnv → tryRouteCli → buildProgram → parseAsync）
- Route-First 最佳化：health/status/sessions/config get 跳過 buildProgram 省 ~500ms
- Command Registry：三層載入策略（Route-First / Primary Only / Full），barrel export 模式，40+ CoreCliEntry
- 命令完整分類：Agent 執行(2) + Agent 管理(6) + Channel 管理(7) + Model(5) + Auth(30+ adapters) + Config(6) + Status/Doctor(5+10 submodules) + Onboarding(20+) + Infra/System(15+) + Messaging(5) + Utility(12+)
- 5 種 Handler 架構模式：Simple Flag / Interactive Wizard / Delegated Execution / Modular Diagnostic / Batch Operation
- Progress 系統：OSC → spinner → line → log → noop 五級 fallback，防巢狀 guard
- Profile 多實例隔離：--profile → 獨立 STATE_DIR + CONFIG_PATH，--dev → port 19001
- CLI DI：CliDeps lazy-loaded message senders + RuntimeEnv(log/error/exit) 注入
- Outbound 投遞管線：sendMessage → deliverOutboundPayloads → plugin adapter → message_sending/sent hooks
- Delivery Queue（Write-Ahead）：JSON file per message → ack rename → fail retry、指數退避（5s→25s→2m→10m）、MAX_RETRIES=5 → failed/
- Gateway 啟動恢復：recoverPendingDeliveries 60s budget、FIFO 排序、permanent error pattern 識別
- Update 機制：Git mode（fetch → worktree preflight 10 commits → checkout → build → doctor）+ Package mode（npm/pnpm/bun global install + --omit=optional fallback）
- Exec Safety：isSafeExecutableValue() shell injection 防護 + Windows CVE-2024-27980 緩解
- Exec Approvals：5 層架構（config → analysis → allowlist → forwarder → decision），socket 120s timeout，security/ask/askFallback 策略
- Path/FS Safety：O_NOFOLLOW symlink 阻擋、boundary check、hardlink 偵測(nlink>1)、atomic write(temp+rename)、post-write 驗證
- Network：SSRF DNS pinning + redirect 驗證(max 3) + cross-origin strip auth、HTTP/HTTPS proxy 支援
- Bonjour mDNS：@homebridge/ciao、openclaw-gw/openclaw-cli service type、TXT records、minimal mode
- Infra 工具模組：retry/backoff、fetch wrapper、JSON file safety、Git ops、archive、provider-usage、channel-activity、device pairing
- 18 個邊界條件/陷阱 + 30+ 關鍵常量 + C# 概念對照

## 2026-03-13 — Phase 6-2：iOS + macOS + Android 原生 App 深入
- 掃描 `apps/ios/` + `apps/macos/` + `apps/android/` + `apps/shared/OpenClawKit/`，共 ~500+ files
- Shared (OpenClawKit SPM)：3 module（OpenClawProtocol / OpenClawKit / OpenClawChatUI）、Gateway Protocol v3、38+ Device Command 定義（camera/canvas/location/screen/contacts/calendar/talk/watch/sms/notifications）、GatewayChannelActor WebSocket 連線、Curve25519 Device Identity、BridgeFrames IPC、A2UI v0.8 commands、ElevenLabs TTS shim
- 三平台共通：雙 WebSocket Session（node role + operator role）分離裝置指令與聊天、Capability 廣播、TLS TOFU 模式、A2UI 跨平台 JS bridge、Voice Wake + Talk Mode 互斥 mic 協調
- iOS (Swift 6.0, SwiftUI, iOS 18+)：NodeAppModel @Observable 核心狀態機 ~2000 行、GatewayConnectionController（Bonjour NWBrowser + TLS TOFU + Keychain）、ScreenController WKWebView + A2UI、VoiceWakeManager SFSpeechRecognizer、TalkModeManager STT+TTS 雙向語音、NodeCapabilityRouter 30+ 命令分派、APNs silent push + BGAppRefreshTask、ShareExtension App Group Container、WatchApp WatchConnectivity、LiveActivity ActivityKit、XcodeGen project.yml
- macOS (Swift 6.2, SwiftUI MenuBarExtra, macOS 15+)：AppState @Observable 中央狀態、三連線模式（unconfigured/local/remote）、GatewayProcessManager launchd plist 管理、GatewayConnection actor auto-recovery、CanvasSchemeHandler openclaw-canvas:// URL scheme + directory traversal 防護、WebChatManager panel/window 模式、VoiceWakeRuntime actor + RMS VAD、MacNodeModeCoordinator + Peekaboo screen/camera、DeepLinks ephemeral key 安全、Sparkle 2.8.1+ 自動更新、ConfigFileWatcher 即時 config 同步、openclaw-mac CLI tool
- Android (Kotlin 2.2, Jetpack Compose 100%, SDK 31-36)：NodeRuntime ~700 行核心 orchestrator、GatewaySession OkHttp3 WebSocket + JSON-RPC 2.0、DeviceIdentityStore Ed25519 BouncyCastle、GatewayDiscovery NSD + dnsjava Wide-Area DNS-SD、InvokeDispatcher 模組化 handler（CameraX/LocationManager/MediaProjection/SmsManager/NotificationListenerService）、CanvasController WebView + A2UI v0.8 hydration、NodeForegroundService START_STICKY、SecurePrefs EncryptedSharedPreferences、Material 3 theme
- 三平台功能矩陣對照：iOS 獨有（Watch/Share/LiveActivity/APNs/Reminders）、macOS 獨有（local gateway/exec approvals/deep links/auto-update/system exec）、Android 獨有（SMS/notification listener/foreground service）
- 版本管理 4 位置：Info.plist (iOS/macOS) + build.gradle.kts (Android) + package.json (CLI)
- 20 個邊界條件/陷阱 + 30+ 關鍵常量 + C# 概念對照

## 2026-03-13 — Phase 6-1：Web UI + Canvas Host + TUI 深入
- 掃描 `ui/` 184 files (~40.3K LOC) + `src/canvas-host/` 5 files (~1.1K LOC) + `src/tui/` 45 files (~8.4K LOC)，共 234 files ~49.8K LOC
- Web UI (Control UI)：Lit 3 Web Components（非 React）、`<openclaw-app>` 單一 LitElement 100+ @state()、無 Shadow DOM、Vite 7 build → dist/control-ui/
- Gateway 連線層：GatewayBrowserClient WS（Protocol 3）、Ed25519 device identity 簽名、token/password/device-token fallback、auto-reconnect (800ms→×1.7→15s max)
- Navigation：手寫 12-Tab 路由（chat/overview/channels/instances/sessions/usage/cron/agents/skills/nodes/config/debug/logs）、History.pushState + popstate、basePath 自動推斷
- View 體系：63 view files，純函數 render → TemplateResult，Chat view（marked+dompurify markdown、tool cards、sidebar split、streaming delta、queue）
- Controllers：18+ controller 模組，直接修改 host @state() 觸發 re-render，無 Redux/MobX
- CSS：純 CSS（無 Tailwind/CSS-in-JS），CSS variables 主題切換，layout 響應式
- Canvas Host：靜態檔案 serve + chokidar live reload（debounce 75ms）+ WS broadcast "reload"、scoped capability token 認證
- A2UI Bundle：Agent-to-UI 橋接（iOS webkit.messageHandlers / Android interface.postMessage）、sendUserAction() + postToNode() 跨平台
- TUI：@mariozechner/pi-tui 框架、GatewayChatClient WS 連接、Container/Text/Loader/ChatLog/CustomEditor 元件樹
- TUI 命令系統：30+ slash commands、! bang-line local shell、burst coalescer（50ms）、backspace deduper（8ms）、Ctrl-C 雙擊退出
- TUI 事件處理：stream assembler delta 組裝、chat/tool event → ChatLog 更新、session/agent 切換
- 三前端共通 API 對照：RPC method 矩陣 + event 矩陣 + 連線方式比較
- 16+ 邊界條件/陷阱（secure context、token persistence、chokidar 崩潰、Windows 貼上、EBADF setRawMode）
- 30+ 關鍵常量速查

## 2026-03-13 — Phase 5-2：關鍵 Extensions 深入
- 掃描 `extensions/` 下 8 個關鍵 extensions，共 ~135+ files, ~28K+ LOC
- voice-call (~68 files, ~9.9K LOC)：Twilio/Telnyx/Plivo/Mock 四 provider 語音通話、Notify+Conversation 雙模式、WebSocket Media Stream、OpenAI Realtime STT、mu-law G.711 audio pipeline、Embedded Pi Agent 自動回應、HMAC/Ed25519/SHA256 webhook 驗簽、10min replay cache、JSONL call state persistence、5 Gateway methods + Tool + CLI + Service
- acpx (~23 files, ~4.8K LOC)：ACP runtime backend adapter、acpx CLI child process spawner、base64url Handle 編碼、AsyncIterable event stream、JSON-RPC line parsing、MCP server injection via proxy（mcp-proxy.mjs stdin rewriter）、Windows .cmd wrapper 解析+快取、version pinning 0.1.15 + auto-install、session ensure/runTurn/cancel/close lifecycle
- diffs (~28 files, ~4.2K LOC)：@pierre/diffs SSR + Shadow DOM hydration、before/after(512KiB) + patch(2MiB) input、Playwright Chromium screenshot PNG/PDF、3 quality presets(standard 8MP/hq 14MP/print 24MP)、DiffArtifactStore TTL 30min~6hr、HTTP viewer CSP+rate-limit、Token 48 hex(24 bytes)、shared browser pool idle 30s
- llm-task (6 files, ~537 LOC)：JSON-only 單次 LLM 呼叫、disableTools:true、AJV schema validation、provider/model fallback chain + allowedModels 白名單、temp workspace auto-cleanup
- thread-ownership (3 files, ~341 LOC)：Slack thread ownership via external slack-forwarder、fail-open 設計、@-mention bypass、abTestChannels gating
- open-prose (~7.5K LOC)：Skill-delivery plugin（register no-op）、/prose command + .prose VM、49 examples + 8 lib modules
- diagnostics-otel (5 files, ~1.1K LOC)：NodeSDK lifecycle、12 event types → 20 metrics + traces + logs、secret redaction、Pino log parsing
- shared (2 files, ~79 LOC)：resolveTarget test helpers + Windows .cmd shim fixtures
- 跨 extension 共通模式：lazy runtime init、config resolution chain、TypeBox+Zod+JSON Schema 三 schema 系統、Embedded Pi Agent 整合、Service start/stop lifecycle
- 16 個邊界條件/陷阱 + 30+ 關鍵常量 + C# 概念對照 + 複雜度對照表

## 2026-03-13 — Phase 5-1：Plugin SDK + Plugin System 深入
- 掃描 `src/plugin-sdk/` 110 files (~8.8K LOC) + `src/plugins/` 83 files (~15K LOC) + `extensions/` 33+ plugins 開發模式分析
- 總計 ~193 files, ~24K LOC
- Plugin Lifecycle 5 階段：Discovery（4 origins: bundled/global/workspace/config, 優先排序, symlink/ownership 安全檢查, 1000ms cache）→ Manifest（openclaw.plugin.json, id+configSchema 必要, uiHints, package.json openclaw 欄位）→ Loading（jiti 動態 import, 37+ SDK aliases, dist→src fallback, 3 種 module 格式）→ Registration（10 種 register API, PluginRegistry 結構）→ Activation（globalThis Symbol, hook runner 初始化）
- Plugin SDK index.ts：799 行 600+ exports，Channel types(20+ adapter), ACP Runtime, Webhook infrastructure(target/guards/rate-limit), Security(SSRF/auth), File ops(lock/dedupe/JSON), Message flow(payload/envelope/dispatch), Access control(allowlist/group-policy), per-channel Zod schemas, Context Engine registry
- jiti Proxy 延遲載入：root-alias.cjs 200 行, fast exports(emptyPluginConfigSchema/resolveControlCommandGate) 即時可用，其餘 Proxy.get lazy-load，dist→src fallback
- Channel-specific SDKs：discord/telegram/slack/whatsapp/signal/line/imessage/bluebubbles thin wrappers
- OpenClawPluginApi 完整介面：10 個 register 方法 + runtime + logger + resolvePath + on()
- 24 Typed Plugin Hooks：before_model_resolve/before_prompt_build/llm_input/llm_output/agent_end/message_received/message_sending/before_tool_call/after_tool_call/session_start/session_end/subagent_spawning 等，priority-based 執行，prompt injection 防護（allowPromptInjection config）
- Slot System：memory + contextEngine 互斥插槽，先到先得，slot=null 停用全 slot
- Direct Commands：72 保留名, /^[a-z][a-z0-9_-]*$/ 命名規則, 4096 bytes args 上限
- Plugin Runtime API：config/system/media/tts/stt/tools/events/logging/state/subagent/channel(40+ methods)
- SubAgent API：run/waitForRun/getSessionMessages/deleteSession，sessionKey 穩定 vs sessionId 易變
- Extension 開發 SOP：register() vs activate()、singleton runtime vs closure state、TypeBox(tools) vs Zod(config) vs JSON Schema(manifest)
- Registry 衝突偵測：HTTP route auth mismatch reject、Provider/Gateway/Command duplicate reject
- Provenance tracking：非信任路徑 plugin warn diagnostic
- VITEST 環境預設：plugins.enabled=false、memory slot="none"
- 20 個邊界條件/陷阱 + 30+ 關鍵常量 + C# 概念對照

## 2026-03-13 — Phase 4-3：Memory/RAG + Context Engine 深入
- 掃描 `src/memory/` 111 files + `extensions/memory-core/` 3 files + `extensions/memory-lancedb/` 5 files + `src/context-engine/` 6 files，共 ~125 files
- Builtin Memory 核心：MemoryIndexManager 三層繼承（Manager → EmbeddingOps → SyncOps），SQLite + sqlite-vec + FTS5
- SQLite Schema：6 tables（meta/files/chunks/chunks_vec/chunks_fts/embedding_cache）
- 搜尋管線：雙層搜尋（vector cosine + FTS5 BM25）→ hybrid merge（weighted score）→ temporal decay（半衰期指數衰減）→ MMR 重排序（Jaccard diversity）→ fallback（FTS-only query expansion）
- 6 Embedding Providers：OpenAI/Ollama/Gemini/Voyage/Mistral，auto 模式不含 Ollama
- Batch Embedding：OpenAI Files API + Gemini + Voyage batch，2 concurrent, 2min timeout, 2 failures → auto-disable
- Sync 管線：file watcher + dirty tracking + 增量 chunking（~100 tokens/chunk, 50 overlap）+ embedding cache（PK: provider+model+key+hash）
- QMD 備援後端：FallbackMemoryManager 透明切換，QMD 失敗 → builtin
- memory-core extension：memory_search + memory_get tools + CLI 註冊
- memory-lancedb extension：LanceDB vector store, auto-recall（before_agent_start, limit=3, minScore=0.3）+ auto-capture（agent_end, max 3/conv）+ safety filters（MEMORY_TRIGGERS + PROMPT_INJECTION_PATTERNS）
- Context Engine：pluggable slot（default "legacy"），ContextEngine 介面（bootstrap/ingest/assemble/compact/afterTurn/subagent lifecycle）
- Context 組裝管線：sanitize → limitHistory → assemble → systemPrompt injection → model call → afterTurn
- Compaction：3 觸發點（overflow auto / manual / proactive afterTurn），reason 分類 8 種
- 20 個邊界條件/陷阱 + 35+ 關鍵常量 + C# 概念對照

## 2026-03-13 — Phase 4-2：Security + Secrets + Config 深入
- 掃描 `src/security/` 30 files (~13.7K LOC) + `src/secrets/` 47 files (~13K LOC) + `src/config/` 199 files (~25K LOC)
- 總計 276 files, ~51.7K LOC
- Security Audit：executeSecurityAudit() 整合 Sync + Async collectors，覆蓋 Gateway auth/Model risk/Node deny/Synced folders/Plugin trust/Skill code safety/FS permissions
- SSRF 防護：isBlockedHostnameOrIp() 封鎖 RFC 1918 + loopback + link-local + multicast + metadata，DNS pinning 防 rebinding，fetchWithSsrFGuard() redirect 驗證（max 3, cross-origin strip auth）
- External Content 隔離：wrapExternalContent() 隨機 marker + 17 injection pattern 偵測 + Unicode anti-spoofing
- Code Safety Scanner：7 rules（dangerous-exec/dynamic-code-execution/crypto-mining/suspicious-network/potential-exfiltration/obfuscated-code/env-harvesting），max 500 files/scan, 1MB/file
- ReDoS 防護：compileSafeRegex() + hasNestedRepetition() 保守解析器，cache 256 entries
- DM/Group 存取控制：4 DM policy modes（open/pairing/allowlist/disabled）+ mutable identity 偵測（Discord/Slack/Google Chat/MS Teams/Mattermost/IRC）
- Dangerous Tools：5 Gateway HTTP deny + 10 ACP deny，Dangerous Config Flags 6 項偵測
- FS Permissions：POSIX chmod + Windows ACL icacls 解析，localized principal 名稱（法/德/西/葡），auto-fix 支援
- Secrets：3 Provider（env/file/exec）無內建加密、路徑權限保護、JSON pointer 存取、exec stdin/stdout 協議（protocolVersion: 1）
- Secrets 生命週期：Configure（互動式）→ Plan（JSON）→ Apply（atomic mutation）+ Runtime Snapshot（prepareSecretsRuntimeSnapshot → activateSecretsRuntimeSnapshot）
- Secret Target Registry：~100+ 密鑰位置定義（target-registry-data.ts 749 lines），sibling_ref + secret_input 兩種 shape
- Secrets Audit：4 finding codes（PLAINTEXT_FOUND/REF_UNRESOLVED/REF_SHADOWED/LEGACY_RESIDUE），掃描 .env/config/auth-profiles/legacy auth.json
- Config 載入管道：JSON5 parse → $include 解析（max depth 10, max 2MB）→ ${VAR} 替換 → Zod 驗證 → 6 層 defaults → path normalization → runtime overrides
- Config Zod Schema：15+ schema 檔，35 type 子模組，.strict() + superRefine()，parseDurationMs/parseByteSize coercion
- Config Include：$include 單檔/多檔/glob，deepMerge（array concat, object recursive, prototype pollution 防護）
- Config Merge Patch：null 刪除 key，array by-id 合併，blocked prototype keys
- Config Runtime Override：setConfigOverride/unsetConfigOverride，最後步驟套用，path 驗證
- Config I/O 安全：audit log（config-audit.jsonl）、env var 寫回還原、SHA256 hash tracking、suspicious write 偵測
- 24 個邊界條件/陷阱 + 50+ 關鍵常量 + C# 概念對照

## 2026-03-13 — Phase 4-1：Browser + Media Understanding + Media Pipeline 深入
- 掃描 `src/browser/` 145 files (~23.5K LOC) + `src/media-understanding/` 65 files (~6.9K LOC) + `src/media/` 39 files (~4.5K LOC)
- 總計 249 files, ~34.9K LOC
- Browser：Playwright-core + CDP 引擎、Express REST API（20+ routes）、Profile 系統（per-profile Chrome + port allocation + user-data-dir）
- Browser Interaction：snapshot（aria/ai/role）+ act（click/type/press/hover/drag/select/fill/resize/wait/evaluate/close）+ download + dialog/file-chooser hooks
- Browser Security：Loopback-only binding、fail-closed auth、SSRF navigation guard（含 post-navigation）、path traversal 防護、per-profile process 隔離
- Browser Extension Relay：WebSocket loopback bridge、per-connection token auth、CDP 指令轉發
- Chrome Lifecycle：平台特定 executable 搜尋（Windows Registry/macOS /Applications/Linux paths）、profile decoration（extensions+prefs+policies）
- Media Understanding：3 capability（image.description / audio.transcription / video.description）、10 providers（OpenAI/Anthropic/Google/Deepgram/Groq/Mistral/Minimax×2/Moonshot/Zai）
- Media Understanding Pipeline：normalizeAttachments → scope check → selectAttachments → resolveModelEntries → runAttachmentEntries（provider fallthrough）→ formatBody + extractFileBlocks
- Media Understanding Config：scope allow/deny rules、attachment prefer/mode、per-capability models、concurrency（預設 2）
- Media Pipeline：Inbound capture（fetchWithGuard + MIME sniff + HEIC→JPEG + PDF extract）→ store（{original}---{uuid}.{ext}）→ serve（TTL 2min, single-use cleanup）
- Media Pipeline：MIME 偵測四級（magic bytes → ext → header → fallback）、Image ops（Sharp/sips, EXIF orientation, resize grid）、PDF 提取（pdfjs-dist + canvas fallback）
- Media Pipeline：Base64 安全（zero-allocation size estimate）、FFmpeg 整合（45s timeout, 20min max audio）、Inbound path policy（iMessage sandbox roots）
- MEDIA token 格式解析（跳過 code fences）+ [[audio_as_voice]] tag + outbound attachment 解析
- 三模組整合：Agent Engine ↔ Browser（REST）/ Media Understanding（applyMediaUnderstanding）↔ Media（extractFileContent / store / serve）
- 20 個邊界條件/陷阱 + 45+ 關鍵常量 + C# 概念對照

## 2026-03-13 — Phase 3-3：WhatsApp + Signal + LINE + iMessage + IRC + Google Chat 完整實作深入
- 掃描 `src/web/` 45 files (~6.2K LOC) + `src/whatsapp/` 2 files + `src/signal/` 17 files (~3.1K LOC) + `src/line/` 30 files (~6.1K LOC) + `src/imessage/` 19 files (~2.5K LOC) + `extensions/irc/` 17 files (~2.9K LOC) + `extensions/googlechat/` 15 files (~2.9K LOC)
- 總計 156 files, 26.2K LOC（含各頻道 extensions）
- WhatsApp：`src/web/` = Baileys WhatsApp Web 逆向協定、QR 登入/credential 備份、pairing challenge、broadcast groups、echo LRU、heartbeat+watchdog 雙層保活
- Signal：signal-cli daemon HTTP+SSE、JSON-RPC 2.0、Markdown→text style ranges、dual identity（phone+UUID）、read receipts、typing indicators
- LINE：Webhook+REST @line/bot-sdk、reply token 3分鐘一次性→fallback push、5 msgs/batch、Flex Message JSON layout、Quick Reply 13按鈕、Markdown→Flex IR 轉換、Rich Menu、replay cache 10min/4096
- iMessage：imsg CLI stdin/stdout JSON-RPC 2.0、macOS限定、multi-service（iMessage+SMS）、reply tag 文字層級、handle 多格式（phone/email/chat_id/chat_guid）
- IRC：extensions-only、Node.js net/tls raw socket、RFC 3659 parser、NickServ 認證+GHOST recovery、350 chars/line word-boundary chunking、mention gating 預設需 nick 提及
- Google Chat：extensions-only、Webhook+REST chat.googleapis.com/v1、Service Account OAuth 2.0、JWT webhook 驗證+10min公鑰快取、thread support、reactions CRUD、media upload 20MB multipart/related
- 九頻道總對照表（連線/SDK/認證/訊息能力/規模）
- Extension Plugin 共通架構（ChannelPlugin<T> + runtime.ts 延遲初始化）
- Access Control 三層比較（DM policy + allowFrom + group policy）
- 26 個邊界條件/陷阱 + 30+ 關鍵常量 + C# 概念對照

## 2026-03-13 — Phase 3-2：Telegram + Slack 完整實作深入
- 掃描 `src/telegram/` 64 files (~15.1K LOC) + `src/slack/` ~80 files (~12.7K LOC) + `extensions/telegram/` + `extensions/slack/` + auto-reply channel plugins
- 三頻道架構對照表：Telegram vs Slack vs Discord（連線模式、限制、重連策略）
- Telegram 連線：Webhook（1MB max body, 30s timeout, secret token 驗證）vs Polling（grammY runner, 90s stall detection, 60min max retry）
- Telegram Inbound：5-stage middleware stack → text fragment coalescing（1500ms/12 parts/50K chars）→ media group assembly（200ms）→ forward burst debounce（80ms）
- Telegram Outbound：1,269 lines send.ts → three-lane delivery（draft/reasoning/status）→ 4096 char limit → 200-800 char draft chunking
- Telegram 專屬：Forum topic config inheritance（Topic→Group→Wildcard）、sticker vision cache、inline buttons 5 scope、thread bindings（30min idle/7d max）
- Slack 連線：Socket Mode（@slack/bolt + xapp- token, 2s→30s backoff, 12 max attempts）vs HTTP（signingSecret 驗證）
- Slack Inbound：event routing → debounce → prepare（803 lines: auth+content+thread+routing）→ dispatch（531 lines: streaming+delivery）
- Slack Outbound：4000 char limit → native streaming（ChatStreamer）vs draft stream（replace/status_final/append）→ blocks support
- Slack 專屬：slash commands（881 lines 最大檔）、modals（262 lines lifecycle）、block interactions（675 lines）、reaction notifications 4 mode、channel lifecycle sync
- Threading 對照：Telegram DM thread/forum topic vs Slack thread_ts participation cache（24h TTL, 5K max）vs Discord thread bindings
- Access control 對照：三層防護（DM policy + allowFrom + group policy）各平台實作差異
- Account 多帳號：共通 ResolvedAccount 模式 + Slack 特有 3 token（bot/app/user）
- Extension plugin：ChannelPlugin 介面 + runtime 延遲初始化
- Auto-reply 整合：Telegram conversation ID 解析 + Slack outbound adapter + normalize plugin
- 20 個邊界條件/陷阱（10 Telegram + 10 Slack）+ 35 個關鍵常量 + C# 概念對照

## 2026-03-13 — Phase 3-1：Discord 完整實作深入
- 掃描 `src/discord/` 170 files (~41,600 lines) + `extensions/discord/` 5 files + `src/config/types.discord.ts`
- 架構鳥瞰：Core(~6K) + Monitor(~32K) + Voice(~1.7K) + Extension(~800) + Config(~350)
- Core Layer：@buape/carbon Client、fetchDiscord<T>() 泛型 API、Token 多來源解析、Proxy 支援
- 多帳號管理：base config + per-account overrides、enabled 兩層 AND、token source 追蹤
- Gateway：Intent 計算（7 必要 + 2 可選）、ProxyGatewayPlugin、max 50 reconnect
- Provider 啟動序列：10 階段（Config → Validate → Manager → Client → Presence → Commands → Identity → Listeners → Gateway → Cleanup）
- 7-layer Inbound Pipeline：Ingestion → Debounce → Preflight(40+ gates) → Job Queue → Worker Queue(30min timeout) → Processing(9 stages) → Delivery
- Preflight 40+ 驗證閘門：Identity/Basic Filter/Channel Type/Guild Access/DM Auth/Mention/Bot Filter/Thread Binding/Route+Media/History
- Reply Delivery：2000 字 chunking + code fence 平衡 + 3x retry + webhook persona
- Draft Streaming：partial/block/progress 模式、1200ms throttle、edit-or-new-message 最終化
- Send 子系統：完整 CRUD（messages/channels/guild/permissions/reactions/emoji/stickers/polls/search）
- Thread Bindings：完整生命週期（bind → touch → sweep 120s → idle 24h → unbind）、webhook persona（☁ prefix）、globalThis 跨模組共享、30s echo 抑制
- Voice：@discordjs/voice 連線、48kHz stereo、DAVE 加密、waveform 256 點取樣、OGG/Opus 轉換
- Slash Commands：@buape/carbon Command 框架、Discord 100 上限自動裁剪、DM pairing flow
- Auto-Presence：健康狀態 → Discord status mapping（healthy→online/degraded→idle/exhausted→dnd）、signature 去重
- Model Picker：3 視圖狀態機（providers/models/recents）、100 字元 custom ID 壓縮
- Allow List：6 種 token 格式、slug 正規化、guild → channel 2 層 fallback、PluralKit 豁免
- Extension Plugin：channel.ts capabilities 註冊 + subagent-hooks.ts thread 整合 + runtime.ts 延遲初始化
- Config Schema 完整結構：DiscordAccountConfig 全部欄位含巢狀（DM/Guild/Voice/Presence/ThreadBindings/ExecApprovals）
- 20 個邊界條件/陷阱 + 35 個關鍵常量 + C# 概念對照

## 2026-03-13 — Phase 2-2：Hook + Cron + Process + Heartbeat 深入
- 掃描 `src/hooks/` 43 files + `src/cron/` ~8,800 lines + `src/process/` 25 files + `src/infra/heartbeat-*` 9 files
- Hook 系統：5 種 event type、4 階段 lifecycle（discovery → load → register → trigger）、hook mapping 引擎、template 安全
- Gmail Watcher：Google Pub/Sub + gog CLI push 架構、setup/watch/serve 三階段、auto-restart
- 4 bundled hooks 實作：session-memory / boot-md / bootstrap-extra-files / command-logger
- CronService：3 種 schedule type（at/every/cron）、timer 主循環、stagger 防雷暴、backoff 梯度（30s→60m）
- Isolated Agent 執行：model selection 5 層優先、session 建立、delivery dispatch
- Delivery 系統：announce / webhook / fallback to main session 三路徑
- 持久化：JSON atomic write + JSONL run log（2MB rotation, 2000 lines）+ locked() 序列化
- CommandLane：4 lane FIFO queue + per-lane semaphore + generation stale invalidation
- ProcessSupervisor：child/pty 雙模式、Windows CVE-2024-27980 緩解、kill-tree 跨平台
- Heartbeat Runner：interval timer + preflight gates + transcript pruning + 24h 去重
- Heartbeat Wake：priority-based coalescing（250ms）+ retry cooldown（1s）
- System Event Bus：session-scoped in-memory queue（max 20）、peek/drain 語義
- Gateway Restart：SIGUSR1 token 機制 + idle poll + 30s cooldown
- 15 個邊界條件/陷阱 + 30 個關鍵常量 + C# 概念對照
- 跨系統整合圖：Hook → Cron → SystemEvent → Heartbeat → CommandLane → Supervisor 完整資料流

## 2026-03-13 — Phase 2-1：Gateway HTTP Pipeline + Chat Event + Routing 深入
- 掃描 `src/gateway/` 50+ server-*.ts + `src/routing/` 11 files
- HTTP Pipeline：13-stage first-match-wins 架構逐層深入（hooks/tools-invoke/slack/openresponses/openai/canvas/plugin/control-ui/probes）
- WebSocket：upgrade handler + WS connection 建立 + GatewayWsClient 結構
- Chat Event：delta streaming protocol（150ms throttle + flush + suppress 規則）、完整資料結構
- WS RPC：25+ handler groups、method authorization（role + scope）、control plane write rate limit
- Routing：7-tier binding match 完整演算法 + 分桶索引 + session key 構建 + identity links
- Channel 生命週期：start/stop/auto-restart（exponential backoff 5s→5min, max 10）
- Startup/Shutdown 完整流程 + Hot Reload + Auth + Rate Limiting
- 12 個邊界條件/陷阱 + 15 個關鍵常量 + C# 概念對照

## 2026-03-12 — Phase 1-2：Provider + Model + Session 深入
- 掃描 `src/agents/` model/provider 系列 + `src/sessions/` + `src/config/sessions/` + `src/providers/`
- Provider Registry：20+ implicit providers、8 種 API 協定、models.json 雙模式寫入
- Model Resolution：alias → provider ID normalize → model ID normalize → ModelRef 完整流程
- Auth Profile：三種憑證型別、Round-Robin + Cooldown 選擇演算法、外部 CLI 同步、Session Override
- Fallback Chain：失敗分類（9 種 FailoverReason）、Smart Probe、Transient HTTP 重試
- Streaming：4 層架構（PI SDK → Subscribe → Block Chunker → Directives）
- Session：雙檔架構（sessions.json + .jsonl）、SessionEntry 完整欄位、生命週期、Store 維護
- 10 個邊界條件/陷阱 + 14 個關鍵常量速查
- 更新 _INDEX.md + _CHANGELOG.md

## 2026-03-12 — Phase 1-1：Auto-Reply 引擎深入
- 掃描 `src/auto-reply/` 281 .ts + `src/web/auto-reply/` 28 .ts
- 完整資料流追蹤：Channel → dispatch → getReplyFromConfig → Agent → Dispatcher → Channel
- 記錄：40+ 命令系統、Session 管理、Heartbeat、Block Streaming、Debounce、Typing、Fallback
- 10 個邊界條件/陷阱（NO_REPLY 誤殺、Dispatcher reservation、Echo protection 等）
- 15 個關鍵常量速查表 + 完整 C# 概念對照
- 更新 _INDEX.md + _CHANGELOG.md

## 2026-03-12 — 新增 Project_File_Tree.md
- 掃描全專案目錄結構，產生結構摘要（5,822 .ts / 632 目錄）
- 含 src/ 50+ 子目錄按規模排序、40 extensions、52 skills、4 apps 平台
- 更新 _INDEX.md 加入索引

## 2026-03-10 — 知識庫初始建立（Operation One-Shot Ingest）
- 建立完整 _AIDocs 知識庫（6 個主題文件 + 索引）
- 6 個並行 Agent 掃描全專案 + 課程教材
- 更新原子記憶 MEMORY.md + 6 個 atom files
- 涵蓋：架構 / AI 引擎 / Gateway / Memory / Plugin / CLI / Extensions / Skills
