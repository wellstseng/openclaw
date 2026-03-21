# OpenClaw 14 天精通課程

> **對象**：Wells 課長（資深 C#/.NET 架構師）
> **目標**：兩週內掌握整個 OpenClaw TypeScript 專案全貌，具備獨立開發能力
> **方法**：每個 TS 概念都用 C# 對照解釋，從「已知」推到「未知」

---

## 課程地圖

```
第一週：基礎 + 核心                    第二週：引擎 + 進階
┌─────────────────────────┐      ┌─────────────────────────┐
│ Day 1  TS 基礎語法       │      │ Day 8  AI Agent 核心     │
│ Day 2  TS 進階語法       │      │ Day 9  Streaming + Tools │
│ Day 3  Node.js 生態系    │      │ Day 10 Memory / RAG      │
│ Day 4  程式進入點        │      │ Day 11 Plugin 系統       │
│ Day 5  Gateway HTTP      │      │ Day 12 Config + CLI      │
│ Day 6  Gateway Chat      │      │ Day 13 Extensions+Skills │
│ Day 7  Channel 系統      │      │ Day 14 總複習 + 實戰     │
└─────────────────────────┘      └─────────────────────────┘
```

---

## 每日節奏（建議 4-6 小時/天）

| 時段 | 活動 | 時長 |
|------|------|------|
| 上午 | 閱讀課程筆記 + C# 對照理解 | 1-1.5 hr |
| 上午 | 閱讀指定 Source Code | 1.5-2 hr |
| 下午 | 實作練習 / 動手改 code | 1-2 hr |
| 下午 | Checkpoint 自我檢核 | 15-30 min |

---

## 第一週：基礎建設 + 核心架構

### Day 1 — TypeScript 基礎語法（C# 開發者速成版）
- **檔案**：[week1/day01-ts-basics.md](week1/day01-ts-basics.md)
- **目標**：掌握 TS 基本型別、變數、函式、類別、介面
- **C# 對照**：var → const/let、class/interface 差異、enum
- **閱讀作業**：`src/utils.ts`、`src/globals.ts`
- **預計時間**：4 小時

### Day 2 — TypeScript 進階語法 + 非同步
- **檔案**：[week1/day02-ts-advanced.md](week1/day02-ts-advanced.md)
- **目標**：Destructuring、Spread、Union Type、Generic、async/await、Module
- **C# 對照**：Deconstruct、record with、enum flags、Task<T>、namespace/using
- **閱讀作業**：`src/version.ts`、`src/channels/registry.ts`
- **預計時間**：5 小時

### Day 3 — Node.js 生態系 + 專案建置
- **檔案**：[week1/day03-nodejs-ecosystem.md](week1/day03-nodejs-ecosystem.md)
- **目標**：Event Loop、npm/pnpm、package.json、tsconfig、Docker
- **C# 對照**：ThreadPool、NuGet、.csproj、MSBuild
- **實作**：把專案跑起來 (docker-compose up)
- **預計時間**：4 小時

### Day 4 — 程式進入點：從 Main() 開始
- **檔案**：[week1/day04-entry-points.md](week1/day04-entry-points.md)
- **目標**：理解 entry.ts → index.ts → CLI → Gateway 的啟動流程
- **C# 對照**：Program.cs → Startup.cs → Host.Run()
- **閱讀作業**：`src/entry.ts`、`src/index.ts`、`src/cli/program/`
- **預計時間**：4 小時

### Day 5 — Gateway Server（上）：HTTP + WebSocket
- **檔案**：[week1/day05-gateway-http.md](week1/day05-gateway-http.md)
- **目標**：Express middleware、WebSocket server、認證
- **C# 對照**：ASP.NET Middleware Pipeline、SignalR、AuthenticationHandler
- **閱讀作業**：`src/gateway/server.ts`、`server-http.ts`、`auth.ts`
- **預計時間**：5 小時

### Day 6 — Gateway Server（下）：Chat + 廣播 + 排程
- **檔案**：[week1/day06-gateway-chat.md](week1/day06-gateway-chat.md)
- **目標**：聊天核心邏輯、WebSocket 廣播、Cron 排程、OpenAI 相容 API
- **C# 對照**：SignalR Hub、IHubContext<T>.Clients.All、IHostedService+Timer
- **閱讀作業**：`server-chat.ts`、`server-broadcast.ts`、`server-cron.ts`、`openai-http.ts`
- **預計時間**：5 小時

### Day 7 — Channel 系統：多通道架構
- **檔案**：[week1/day07-channels.md](week1/day07-channels.md)
- **目標**：Channel 介面設計、Registry、Dock、實作一個 Channel 的完整流程
- **C# 對照**：IHostedService、Strategy Pattern、Factory Pattern
- **閱讀作業**：`src/channels/`、`src/telegram/`（挑一個完整讀）
- **預計時間**：5 小時

---

## 第二週：AI 引擎 + 進階系統

### Day 8 — AI Agent 引擎（上）：核心 Runner
- **檔案**：[week2/day08-agent-core.md](week2/day08-agent-core.md)
- **目標**：pi-embedded-runner 的完整流程、System Prompt 組裝、模型選擇
- **C# 對照**：Semantic Kernel、IChatCompletionService
- **閱讀作業**：`src/agents/pi-embedded-runner.ts`、`system-prompt.ts`、`model-catalog.ts`
- **預計時間**：6 小時（本課程最硬的一天）

### Day 9 — AI Agent 引擎（下）：Streaming + Tool Calling
- **檔案**：[week2/day09-agent-streaming.md](week2/day09-agent-streaming.md)
- **目標**：串流訂閱、Tool 定義與執行、Sandbox 沙箱
- **C# 對照**：IAsyncEnumerable<T>、Function Calling、Docker SDK
- **閱讀作業**：`pi-embedded-subscribe.ts`、`pi-tools.ts`、`bash-tools.ts`、`sandbox.ts`
- **預計時間**：5 小時

### Day 10 — Memory / RAG 系統
- **檔案**：[week2/day10-memory-rag.md](week2/day10-memory-rag.md)
- **目標**：Embedding 產生、向量儲存、混合搜尋、記憶管理
- **C# 對照**：Semantic Kernel Memory、IMemoryStore
- **閱讀作業**：`src/memory/embeddings.ts`、`sqlite-vec.ts`、`manager.ts`、`hybrid.ts`
- **預計時間**：4 小時

### Day 11 — Plugin 系統
- **檔案**：[week2/day11-plugins.md](week2/day11-plugins.md)
- **目標**：Plugin 載入、Registry、Discovery、Manifest、Hook、Slot
- **C# 對照**：MEF2、Assembly.LoadFrom、IServiceCollection
- **閱讀作業**：`src/plugins/registry.ts`、`loader.ts`、`discovery.ts`、`manifest.ts`
- **預計時間**：4 小時

### Day 12 — Config + CLI + Routing
- **檔案**：[week2/day12-config-cli.md](week2/day12-config-cli.md)
- **目標**：YAML 設定管理、Session 系統、路由解析、CLI 指令結構
- **C# 對照**：IConfiguration、appsettings.json、Session Middleware、System.CommandLine
- **閱讀作業**：`src/config/`、`src/routing/`、`src/cli/program/`
- **預計時間**：4 小時

### Day 13 — Extensions + Skills + UI
- **檔案**：[week2/day13-extensions-skills.md](week2/day13-extensions-skills.md)
- **目標**：Extension 架構、Skill 定義、Web UI (Lit)、Native App 架構
- **C# 對照**：NuGet Plugin、Roslyn Source Generator、Blazor、MAUI
- **閱讀作業**：挑 2-3 個 extension、2-3 個 skill、`ui/src/`
- **預計時間**：4 小時

### Day 14 — 總複習 + 實戰演練
- **檔案**：[week2/day14-review-practice.md](week2/day14-review-practice.md)
- **目標**：End-to-End 訊息追蹤、嘗試修改 code、架構總複習
- **實戰**：追蹤一則 WhatsApp 訊息從收到到回覆的完整路徑
- **預計時間**：5 小時

---

## 輔助資源

| 資源 | 說明 |
|------|------|
| [ts-csharp-cheatsheet.md](ts-csharp-cheatsheet.md) | TS ↔ C# 語法速查表（隨時翻閱） |
| [../OpenClaw-Architecture-Guide.md](../OpenClaw-Architecture-Guide.md) | 架構全覽圖（之前已產生） |

---

## 里程碑檢核

| 完成時間 | 你應該能做到 |
|----------|------------|
| Day 2 結束 | 看 TS code 不再恐慌，能「翻譯」成 C# 理解 |
| Day 4 結束 | 知道整個專案怎麼啟動、CLI 怎麼分發指令 |
| Day 7 結束 | 理解 Gateway + Channel 的完整訊息流 |
| Day 9 結束 | 理解 AI 怎麼收到訊息、產生回覆、串流回傳 |
| Day 14 結束 | 能獨立追蹤 bug、新增功能、修改設定 |

---

> 課程設計日期：2026-02-12，更新至 2026-02-27
> 基於 OpenClaw v2026.2.26
>
> v2026.2.26 新增章節：Day 4 §4.6（agents bindings）、Day 12 §12.5（secrets 管理）
