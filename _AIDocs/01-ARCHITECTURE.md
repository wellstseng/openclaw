# OpenClaw 整體架構

## 一句話定位
Multi-channel AI Gateway：接收來自 Telegram/Discord/WhatsApp/Slack 等 9+ 頻道的訊息，經路由分配給 AI Agent 處理，再將回覆投遞回原頻道。

## 技術棧
- TypeScript ESM / Node 22+ / pnpm monorepo
- 框架核心：`@mariozechner/pi-coding-agent`（Pi Agent Framework）
- HTTP：Express 5 / WebSocket：ws
- 資料庫：node:sqlite（內建）+ sqlite-vec（向量擴展）
- 建置：tsdown / oxlint / oxfmt / vitest
- 包管理：pnpm 10.23（也支援 bun）

## 核心架構圖

```
                    ┌──────────────────────────────────┐
                    │          Client Layer             │
                    │  Web UI / macOS App / iOS / CLI   │
                    └──────────────┬───────────────────┘
                                   │ WebSocket / HTTP
                    ┌──────────────▼───────────────────┐
                    │         Gateway Server            │
                    │  server.impl.ts (4000+ lines)     │
                    │                                   │
                    │  ┌─────────┐ ┌─────────────────┐ │
                    │  │HTTP Stage│ │ Chat Event Pipe │ │
                    │  │Pipeline  │ │ (WebSocket↔Agent)│ │
                    │  │(13 stages)│ │ (150ms throttle)│ │
                    │  └─────────┘ └─────────────────┘ │
                    └───┬────────────┬──────────┬──────┘
                        │            │          │
           ┌────────────▼──┐   ┌────▼────┐  ┌──▼──────────┐
           │  Channel Layer │   │ Routing │  │ Agent Engine │
           │                │   │ 7-tier  │  │              │
           │ Registry(meta) │   │ binding │  │ runEmbedded  │
           │ Dock(behavior) │   │ match   │  │ PiAgent()    │
           │ Plugin(impl)   │   │         │  │              │
           │                │   │ Session │  │ Tools(20+)   │
           │ 9 built-in +   │   │ Key Gen │  │ Skills(52)   │
           │ 40 extensions  │   │         │  │ Memory/RAG   │
           └────────────────┘   └─────────┘  └──────┬──────┘
                                                     │
                                          ┌──────────▼──────────┐
                                          │   LLM Providers     │
                                          │ Anthropic / OpenAI  │
                                          │ Google / Bedrock    │
                                          │ Ollama / 20+ others │
                                          └─────────────────────┘
```

## 訊息完整生命週期

```
1. Channel 收到訊息（e.g. Telegram Bot API polling）
2. → Channel Plugin 建立 InboundMessage
3. → resolveAgentRoute() 決定交給哪個 Agent（7-tier 匹配）
4. → buildAgentPeerSessionKey() 生成 session key
5. → enqueueCommand("main", task) 加入命令隊列
6. → runEmbeddedPiAgent() 啟動 Agent 引擎
   a. buildAgentSystemPrompt() 組裝 system prompt（20+ sections）
   b. resolveModel() 選擇最佳模型
   c. ensureAuthProfileStore() 取得 API key
   d. streamSimple() 呼叫 LLM（streaming）
   e. tool_calls loop（執行工具 → 回傳結果 → 再呼叫 LLM）
   f. context overflow → compact()（摘要壓縮）
7. → buildEmbeddedRunPayloads() 格式化回覆
8. → deliverOutboundMessage() 分塊投遞（考慮 channel 限制）
9. → 儲存對話紀錄到 session file
```

## 目錄結構 TOP 10（按檔案數）

| 目錄 | 檔案數 | 職責 |
|------|--------|------|
| src/agents/ | 802 | AI Agent 引擎核心 |
| src/infra/ | 360 | 基礎設施（執行、網路、安全） |
| src/commands/ | 349 | CLI 命令實作 |
| src/gateway/ | 344 | HTTP/WS Gateway |
| src/cli/ | 285 | CLI 框架 |
| src/auto-reply/ | 281 | 自動回覆系統 |
| src/config/ | 226 | JSON5 設定系統 |
| src/channels/ | 174 | 頻道抽象層 |
| src/discord/ | 170 | Discord 實作 |
| src/browser/ | 145 | 瀏覽器自動化 |

## C# 概念對照

| OpenClaw 概念 | C# 等價 |
|--------------|---------|
| Gateway Server | ASP.NET Core + SignalR Hub |
| Channel Plugin | IHostedService + Strategy Pattern |
| Agent Runner | Semantic Kernel IChatCompletionService |
| Tool Registry | DI IServiceCollection |
| Session File (JSONL) | Entity State Store |
| Config (JSON5) | IConfiguration + appsettings.json |
| Commander.js CLI | System.CommandLine |
| Event Loop (single-thread) | ThreadPool（但不需 lock） |
| Plugin Loader (jiti) | MEF2 / Assembly.LoadFrom |
| Daemon Service | Windows Service / systemd unit |
