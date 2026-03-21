# OpenClaw — 專案結構摘要

> 更新日期：2026-03-12 | Branch: v2026.3.7
> 總計：5,822 .ts 檔 | 632 目錄（排除 node_modules/packages/.git）

---

## 頂層目錄

```
openclaw/
├── src/              — 核心原始碼（4,667 .ts）
├── extensions/       — 40 個擴充模組（780 .ts）
├── skills/           — 52 個 Skill（YAML + 腳本）
├── ui/               — Web 控制介面（185 .ts）
├── vendor/           — 第三方嵌入（111 .ts）
├── apps/             — 原生 App（iOS / Android / macOS）
├── test/             — 整合測試（29 .ts）
├── scripts/          — 建置/部署腳本（35 .ts）
├── docs/             — 官方文件（en + zh-CN + ja-JP）
├── assets/           — 靜態資源 + Chrome Extension
├── .agent/           — Agent workflow 定義
├── .agents/          — Agent 設定
├── .github/          — CI/CD workflows + actions
├── .pi/              — PI 設定（prompts / extensions / git）
├── git-hooks/        — Git hook 腳本
├── patches/          — pnpm patch 檔
├── Swabble/          — Swabble 整合
└── _AIDocs/          — AI 知識庫（本資料夾）
```

---

## src/ 子目錄（按規模排序）

| 目錄 | .ts 數 | 說明 |
|------|--------|------|
| `agents/` | 802 | AI Agent 引擎核心（run、tool、model、flow） |
| `infra/` | 360 | 基礎設施（DB、queue、storage、auth） |
| `commands/` | 349 | CLI 指令實作 |
| `gateway/` | 344 | Gateway 服務（HTTP/WS/API） |
| `cli/` | 285 | CLI 框架與進入點 |
| `auto-reply/` | 281 | 自動回覆引擎 |
| `config/` | 226 | 設定管理 |
| `channels/` | 174 | Channel 抽象層 |
| `discord/` | 170 | Discord 整合 |
| `browser/` | 145 | 瀏覽器控制 |
| `telegram/` | 132 | Telegram 整合 |
| `slack/` | 122 | Slack 整合 |
| `plugin-sdk/` | 109 | Plugin SDK |
| `cron/` | 102 | 排程/定時任務 |
| `memory/` | 95 | 記憶 / RAG |
| `plugins/` | 83 | Plugin 載入器 |
| `web/` | 77 | Web 服務 |
| `media-understanding/` | 65 | 多媒體理解 |
| `shared/` | 51 | 共用模組 |
| `acp/` | 51 | ACP 協定 |
| `line/` | 48 | LINE 整合 |
| `daemon/` | 47 | 背景 Daemon |
| `tui/` | 45 | 終端 UI |
| `secrets/` | 45 | 機密管理 |
| `hooks/` | 43 | Hook 系統 |
| `media/` | 39 | 媒體處理 |
| `signal/` | 32 | Signal 整合 |
| `test-utils/` | 31 | 測試工具 |
| `imessage/` | 31 | iMessage 整合 |
| `utils/` | 29 | 通用工具函式 |
| `security/` | 29 | 安全模組 |
| `logging/` | 29 | 日誌系統 |
| `process/` | 26 | Process 管理 |
| `terminal/` | 19 | Terminal 控制 |
| `wizard/` | 16 | 設定精靈 |
| `node-host/` | 15 | Node 主機 |
| `markdown/` | 14 | Markdown 處理 |
| `sessions/` | 12 | Session 管理 |
| `routing/` | 11 | 路由引擎 |
| `providers/` | 11 | LLM Provider 抽象 |
| `types/` | 9 | 型別定義 |
| `pairing/` | 9 | 裝置配對 |
| `link-understanding/` | 6 | 連結解析 |
| `context-engine/` | 6 | 上下文引擎 |
| `canvas-host/` | 5 | Canvas 主機 |
| `whatsapp/` | 4 | WhatsApp 整合 |
| `tts/` | 4 | 語音合成 |
| `test-helpers/` | 4 | 測試輔助 |
| 其他（scripts/i18n/docs/compat） | 5 | 雜項 |

---

## extensions/（40 個擴充）

```
acpx              discord           llm-task          nostr             talk-voice
bluebubbles       feishu            lobster           open-prose        telegram
copilot-proxy     google-gemini-*   matrix            phone-control     test-utils
device-pair       googlechat        mattermost        qwen-portal-auth  thread-ownership
diagnostics-otel  imessage          memory-core       shared            tlon
diffs             irc               memory-lancedb    signal            twitch
                  line              minimax-portal-*  slack             voice-call
                                    msteams           synology-chat     whatsapp
                                    nextcloud-talk                      zalo / zalouser
```

---

## skills/（52 個）

```
1password    coding-agent  gifgrep      nano-banana-pro  session-logs    tmux
apple-notes  discord       github       nano-pdf         sherpa-onnx-tts trello
apple-*      eightctl      gog          notion           skill-creator   video-frames
bear-notes   gemini        goplaces     obsidian         slack           voice-call
blogwatcher  gh-issues     healthcheck  openai-image-gen songsee         wacli
blucli                     himalaya     openai-whisper*  sonoscli        weather
bluebubbles  camsnap       imsg         openhue          spotify-player  xurl
canvas       clawhub       mcporter     oracle           summarize
                           model-usage  ordercli/peekaboo/sag  things-mac
```

---

## apps/（原生 App）

| 平台 | 語言 | 重點 |
|------|------|------|
| `android/` | Kotlin | Chat UI + Gateway client + Voice |
| `ios/` | Swift | Chat + Camera + Voice + Calendar + LiveActivity + Watch |
| `macos/` | Swift | Node 模式 + Discovery + IPC |
| `shared/` | Swift | OpenClawKit（共用 ChatUI + Protocol） |

---

## ui/（Web 控制台）

```
ui/
├── public/    — 靜態資源
└── src/       — React/TS 前端（185 .ts）
```
