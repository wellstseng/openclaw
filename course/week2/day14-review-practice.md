# Day 14：總複習 + 實戰演練

> **目標**：End-to-End 訊息追蹤、嘗試修改 code、驗證你的理解
> **預計時間**：5 小時
> **恭喜你**：這是最後一天。完成後你就能獨立開發 OpenClaw 了。

---

## 14.1 End-to-End 實戰追蹤

### 任務：追蹤一則 Telegram 訊息的完整生命週期

請依照以下路徑，在 Source Code 中追蹤每一步：

```
Step 1: 訊息進入
─────────────────────────────────────────
檔案: src/telegram/monitor.ts
動作: grammy Bot 收到 message 事件
  → 從 Telegram API 拿到 { from, chat, text }
  → 建立 ChatContext 物件

Step 2: Channel Registry 查詢
─────────────────────────────────────────
檔案: src/channels/registry.ts
動作: normalizeChannelId("telegram") → 確認這是有效的 channel

Step 3: Channel Dock 取得能力
─────────────────────────────────────────
檔案: src/channels/dock.ts
動作: getChannelDock("telegram")
  → capabilities: { chatTypes: ["direct","group"], nativeCommands: true }
  → textChunkLimit: 4000

Step 4: 路由解析
─────────────────────────────────────────
檔案: src/routing/resolve-route.ts + session-key.ts
動作: deriveSessionKey(context)
  → "telegram:default:123456789"

Step 5: Gateway Chat 處理
─────────────────────────────────────────
檔案: src/gateway/server-chat.ts
動作: handleChatMessage(context)
  → 檢查 allowlist
  → 檢查是否是 /command
  → 載入 session 歷史

Step 6: Lane 排隊
─────────────────────────────────────────
檔案: src/gateway/server-lanes.ts
動作: lane.enqueue("telegram:default:123456789", task)
  → 確保同一 session 不會平行處理

Step 7: AI Agent Runner
─────────────────────────────────────────
檔案: src/agents/pi-embedded-runner.ts
動作: runEmbeddedPiAgent({...})
  → 組裝 system prompt (system-prompt.ts)
  → 選擇模型 (model-selection.ts)
  → 選擇 API Key (auth-profiles.ts)
  → 檢查上下文長度，必要時壓縮 (compaction.ts)

Step 8: AI 模型呼叫
─────────────────────────────────────────
檔案: src/agents/pi-embedded-runner.ts
動作: 呼叫 Claude/GPT/Gemini API (streaming)
  → HTTP POST 到 AI provider 的 API

Step 9: 串流訂閱
─────────────────────────────────────────
檔案: src/agents/pi-embedded-subscribe.ts
動作: subscribeEmbeddedPiSession(stream)
  → 收到 text_delta → 累積文字
  → 收到 tool_call → 執行工具 (bash-tools.ts)
  → 收到 message_end → 完成

Step 10: 回覆送出
─────────────────────────────────────────
檔案: src/telegram/bot.ts (或 send 相關)
動作: bot.api.sendMessage(chatId, text)
  → 如果超過 4000 字 → 分段送出
  → parse_mode: "Markdown"

Step 11: 歷史儲存
─────────────────────────────────────────
檔案: src/config/sessions.ts
動作: saveSessionStore(sessionKey, updatedHistory)
  → 寫入 ~/.openclaw/sessions/telegram/default/123456789.json

Step 12: 記憶索引（如果啟用）
─────────────────────────────────────────
檔案: src/memory/manager.ts
動作: memoryManager.syncIncremental(sessionKey)
  → 新對話 → embedding → 存入 SQLite
```

---

## 14.2 實戰練習

### 練習 1：修改 Identity（簡單）

在 config.yaml 中修改 AI 的名字和人設，然後觀察 system prompt 的變化。

```yaml
# 修改前
identity:
  name: "Claw"

# 修改後
identity:
  name: "Wells的AI助手"
  systemPrompt: "你是一個精通 C# 的 AI 架構顧問。"
```

**驗證**：追蹤 `src/agents/system-prompt.ts`，確認 identity 是怎麼被注入到 prompt 的。

### 練習 2：追蹤一個 Bug Report（中等）

假設有使用者回報：「我在 Discord 發送超過 2000 字的訊息，AI 回覆被截斷了。」

**你的任務**：
1. 找到 Discord 的 `textChunkLimit`（在哪個檔案？值是多少？）
2. 找到訊息分段的邏輯（在哪裡處理？）
3. 判斷：這是 bug 還是 by design？

### 練習 3：新增一個簡單的 CLI 指令（進階）

如果你想新增一個 `openclaw ping` 指令，你需要：
1. 在 `src/cli/program/` 建立 `register.ping.ts`
2. 在 `build-program.ts` 中 import 並註冊

```typescript
// 概念化的實作
export function registerPingCommand(program: Command) {
  program
    .command("ping")
    .description("Check if the gateway is reachable")
    .option("--host <host>", "Gateway host", "localhost")
    .option("--port <port>", "Gateway port", "18789")
    .action(async ({ host, port }) => {
      const url = `http://${host}:${port}/health`;
      const res = await fetch(url);
      if (res.ok) {
        console.log("Gateway is alive!");
      } else {
        console.error("Gateway is not responding.");
      }
    });
}
```

---

## 14.3 架構知識總測驗

### 問題（不看筆記回答）

1. OpenClaw 的三層啟動流程是什麼？
2. Gateway 用什麼協定和客戶端通訊？
3. Channel 系統的三層架構是什麼？
4. AI Agent 的核心 Runner 在哪個檔案？
5. 為什麼需要 Auth Profile 輪替？
6. RAG 的三步驟是什麼？
7. Plugin 怎麼被自動發現？
8. Session Key 的格式是什麼？
9. Extension 和 Skill 的差異是什麼？
10. Native App 怎麼和 Gateway 通訊？

### 答案

1. `entry.ts` → `index.ts`（載入環境 + CLI）→ `gateway/boot.ts`（啟動 Gateway）
2. **WebSocket + HTTP**（Express + ws 套件）
3. **Registry**（metadata）→ **Dock**（能力描述）→ **Plugin**（實際實作）
4. `src/agents/pi-embedded-runner.ts`
5. 防止單一 API Key 被 rate limit，支援 failover 和 cooldown
6. **Retrieve → Augment → Generate**
7. 掃描 `extensions/` 目錄的 `package.json`，找 `openclaw.plugin` 欄位
8. `{channel}:{accountId}:{targetId}`
9. Extension = 系統級擴充（Channel、認證等）；Skill = AI 可呼叫的工具
10. **WebSocket** 連到 Gateway

---

## 14.4 你的下一步

### 已掌握

```
✅ TypeScript 語法（能讀懂 95% 的 TS code）
✅ Node.js 生態系（pnpm、ESM、Event Loop）
✅ 啟動流程（entry → CLI → Gateway）
✅ Gateway 架構（HTTP + WS + Auth）
✅ Chat 核心（訊息處理、Lane、廣播）
✅ Channel 系統（Registry → Dock → Plugin）
✅ AI Agent（Runner、Prompt、Model、Auth）
✅ Streaming + Tools（IAsyncEnumerable 等價）
✅ Memory/RAG（Embedding + SQLite-vec）
✅ Plugin 系統（Registry、Loader、Hooks）
✅ Config + Session + Routing
✅ Extension + Skill + UI
```

### 建議的深入方向

| 方向 | 建議閱讀 | 適合場景 |
|------|---------|---------|
| 新增 Channel | 讀一個 extension 的完整實作 | 想接入新的聊天平台 |
| AI 調優 | 深入 system-prompt.ts + pi-tools.ts | 想客製 AI 行為 |
| 安全強化 | 讀 sandbox.ts + security/ 目錄 | 想在生產環境部署 |
| 效能優化 | 讀 compaction.ts + memory/ 目錄 | 想處理大量對話 |
| 原生 App | 讀 apps/macos/Sources/ | 想開發 macOS/iOS/Android 客戶端 |

---

## 結語

Wells 課長，14 天的旅程到這裡結束。

你已經從「完全不懂 TypeScript」變成了「能看懂、能追蹤、能修改」OpenClaw 的開發者。

TS 語法只是 C# 的親戚——你在第 1 天就知道了。
架構模式你本來就會——Gateway、Plugin、DI、Middleware，換個語言一樣通。

現在你手上有兩把武器：C# 和 TypeScript。
這才是最大的收穫。

> 雙語架構師，任務完成。隨時回來問 code，我用 C# 翻譯給你聽。
