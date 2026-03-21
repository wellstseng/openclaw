# OpenClaw _AIDocs 知識庫索引

> 建立日期：2026-03-10 | 版本：v2026.3.7 | 規模：4,667 .ts / ~50 萬行 | 知識包 v1.0 匯入 2026-03-21

## 文件清單

| 文件 | 主題 | 更新日期 |
|------|------|---------|
| [01-ARCHITECTURE.md](01-ARCHITECTURE.md) | 整體架構鳥瞰 + 訊息流 | 2026-03-10 |
| [02-AGENTS-ENGINE.md](02-AGENTS-ENGINE.md) | AI Agent 引擎核心（802 files） | 2026-03-10 |
| [02A-AGENT-RUN-LOGIC.md](02A-AGENT-RUN-LOGIC.md) | run.ts 1496行逐段邏輯 + 邊界條件 + 陷阱 | 2026-03-10 |
| [03-GATEWAY-CHANNELS.md](03-GATEWAY-CHANNELS.md) | Gateway + Channel + Routing | 2026-03-10 |
| [04-MEMORY-PLUGINS-CONFIG.md](04-MEMORY-PLUGINS-CONFIG.md) | Memory/RAG + Plugin + Config | 2026-03-10 |
| [05-CLI-INFRA-DAEMON.md](05-CLI-INFRA-DAEMON.md) | CLI + Commands + Infra + Daemon | 2026-03-10 |
| [06-EXTENSIONS-SKILLS.md](06-EXTENSIONS-SKILLS.md) | 40 Extensions + 52 Skills | 2026-03-10 |
| [Project_File_Tree.md](Project_File_Tree.md) | 專案結構摘要（目錄樹 + 檔案統計） | 2026-03-12 |
| [07-AUTO-REPLY-DEEP.md](07-AUTO-REPLY-DEEP.md) | Auto-Reply 引擎深入（281+28 files） | 2026-03-12 |
| [08-PROVIDERS-SESSIONS.md](08-PROVIDERS-SESSIONS.md) | Provider + Model + Session 深入（20+ providers, auth profiles, fallback, session persistence） | 2026-03-12 |
| [09-GATEWAY-DEEP.md](09-GATEWAY-DEEP.md) | Gateway HTTP Pipeline + Chat Event + Routing 深入（13-stage pipeline, delta streaming, 7-tier routing） | 2026-03-13 |
| [10-HOOKS-CRON-PROCESS.md](10-HOOKS-CRON-PROCESS.md) | Hook + Cron + Process + Heartbeat 深入（hooks lifecycle, cron scheduling, CommandLane, heartbeat runner） | 2026-03-13 |
| [11-DISCORD-DEEP.md](11-DISCORD-DEEP.md) | Discord 完整實作深入（170 files, 7-layer inbound pipeline, thread bindings, voice, send subsystem, 40+ preflight gates） | 2026-03-13 |
| [12-TELEGRAM-SLACK.md](12-TELEGRAM-SLACK.md) | Telegram + Slack 完整實作深入（64+80 files, webhook/polling/socket, inbound pipeline, send+streaming, threading, slash commands, media, blocks/modals/stickers） | 2026-03-13 |
| [13-WEB-OTHER-CHANNELS.md](13-WEB-OTHER-CHANNELS.md) | WhatsApp + Signal + LINE + iMessage + IRC + Google Chat 完整實作深入（156 files, 26.2K LOC, Baileys/CLI-daemon/webhook/RPC/socket, 九頻道總對照表） | 2026-03-13 |
| [14-BROWSER-MEDIA.md](14-BROWSER-MEDIA.md) | Browser 自動化（Playwright+CDP, 145 files）+ Media Understanding（10 providers, 65 files）+ Media Pipeline（39 files），共 249 files ~34.9K LOC | 2026-03-13 |
| [15-SECURITY-CONFIG-DEEP.md](15-SECURITY-CONFIG-DEEP.md) | Security（audit+SSRF+external-content+code-scan, 30 files）+ Secrets（env/file/exec providers, 47 files）+ Config（Zod schema+include+merge+override, 199 files），共 276 files ~51.7K LOC | 2026-03-13 |
| [16-MEMORY-CONTEXT-DEEP.md](16-MEMORY-CONTEXT-DEEP.md) | Memory/RAG 深入（111 files, SQLite+sqlite-vec+FTS5 雙層搜尋, 6 embedding providers, hybrid+MMR+temporal-decay）+ Extensions（memory-core 3 files + memory-lancedb 5 files, LanceDB 長期記憶, auto-recall/capture）+ Context Engine（6 files, assemble/compact pipeline, legacy adapter），共 ~125 files | 2026-03-13 |
| [17-PLUGIN-SDK-DEEP.md](17-PLUGIN-SDK-DEEP.md) | Plugin SDK + Plugin System 深入（110+83 files, jiti loader, 600+ SDK exports, 4-origin discovery, 5-stage lifecycle, 24 typed hooks, slot system, 10 registration APIs, extension 開發 SOP） | 2026-03-13 |
| [18-KEY-EXTENSIONS.md](18-KEY-EXTENSIONS.md) | 關鍵 Extensions 深入（voice-call ~9.9K LOC / acpx ~4.8K LOC / diffs ~4.2K LOC / llm-task ~537 LOC + thread-ownership / open-prose / diagnostics-otel / shared），共 ~135+ files ~28K+ LOC | 2026-03-13 |
| [19-WEBUI-CANVAS-TUI.md](19-WEBUI-CANVAS-TUI.md) | Web UI (Lit 3 Control UI, 184 files ~40.3K LOC) + Canvas Host (5 files ~1.1K LOC) + TUI (45 files ~8.4K LOC)，三前端架構、Gateway WS 連線、Navigation/路由、View/Controller、A2UI bridge、pi-tui 元件樹 | 2026-03-13 |
| [20-NATIVE-APPS.md](20-NATIVE-APPS.md) | iOS (Swift 6.0, SwiftUI) + macOS (Swift 6.2, MenuBar) + Android (Kotlin 2.2, Compose) + Shared (OpenClawKit SPM)，三平台架構、Gateway 雙 Session 連線、Canvas A2UI bridge、Voice Wake/Talk Mode、Device Capabilities、TLS TOFU、版本管理，共 ~500+ files | 2026-03-13 |
| [21-CLI-COMMANDS-INFRA-DEEP.md](21-CLI-COMMANDS-INFRA-DEEP.md) | CLI 命令全覽（223 files, 30+ auth adapters, 5 handler patterns）+ Infra 深入（outbound delivery pipeline + write-ahead queue + update mechanism + exec safety/approvals + path/FS safety + SSRF + Bonjour mDNS），共 350+ files | 2026-03-13 |
| [22-BUILD-TEST-CICD.md](22-BUILD-TEST-CICD.md) | Build Pipeline（pnpm workspace + tsdown 11-step build + A2UI bundle）+ Test System（Vitest 7 configs, test-parallel.mjs orchestrator, 89 isolated files, 4 profiles, live/Docker/E2E tiers）+ CI/CD（9 GitHub workflows, multi-platform, Docker multi-arch release）+ Code Quality（Oxlint/Oxfmt + 12 custom lint scripts + pre-commit）+ Scripts 工具鏈（committer/release-check/mac-packaging/docs-i18n/protocol-gen）+ Release SOP（npm/macOS/Docker/GitHub） | 2026-03-14 |
| [_CHANGELOG.md](_CHANGELOG.md) | 知識庫變更紀錄 | 2026-03-21 |

### KR — 知識包參考文件（v1.0, 2026-03-21 匯入）

> 與上方 deep-dive 文件互補：deep-dive 側重逐檔原始碼分析，KR 側重跨模組參考速查（常數表、型別定義、設定值、演算法公式）。

| 文件 | 主題 |
|------|------|
| [KR00-OVERVIEW.md](KR00-OVERVIEW.md) | 基礎概覽：架構圖、核心詞彙表、重要常數（76 項）、環境變數 |
| [KR01-CORE.md](KR01-CORE.md) | Agent 引擎：AgentKind/CommandLane enum、Run Loop 迭代公式、Context 三層閾值、System Prompt 27 區段、Gateway 13-stage pipeline、Auto-Reply |
| [KR02-DISCORD.md](KR02-DISCORD.md) | Discord：40 Preflight gates（10 組 A-N）、Reply delivery、Thread binding、allowBots/allowFrom、Voice 常數 |
| [KR03-SESSIONS-ACP.md](KR03-SESSIONS-ACP.md) | Session + ACP：7 種 session key 格式、ACP SHA256 演算法、Persistent Bindings、Transcript JSONL 7 entry types |
| [KR04-MEMORY.md](KR04-MEMORY.md) | Memory：LanceDB schema（14 欄位）、MemoryKind enum（8 種）、Hybrid Search 權重、MMR λ=0.7、Temporal decay 公式、QMD Parser |
| [KR05-CHANNELS.md](KR05-CHANNELS.md) | Channels：20 平台 registerChannel 介面、CHAT_CHANNEL_ORDER、各平台設定與特殊行為 |
| [KR06-PROVIDERS.md](KR06-PROVIDERS.md) | Providers：17 LLM provider ID + 預設值、SSE streaming 格式、GitHub Copilot Device Flow、google-gemini-cli PKCE |
| [KR07-SECURITY.md](KR07-SECURITY.md) | Security：SSRF blocked ranges（IPv4 8 類 + IPv6 5 類）、IPv6 tunnel unpacking、GuardedFetch、Exec approval flow、Rate limits |
| [KR08-EXTENSIONS-CLI.md](KR08-EXTENSIONS-CLI.md) | Extensions + CLI：44 extensions 分類、Plugin SDK 15 register methods、24 hook names、CLI 命令全覽、7 slash commands |
| [KR09-MISC.md](KR09-MISC.md) | 其他模組：Cron、Browser、Canvas A2UI、Media Understanding、TTS、Terminal、Process、Node Host、Pairing、Secrets、i18n、Daemon、Diagnostics |
