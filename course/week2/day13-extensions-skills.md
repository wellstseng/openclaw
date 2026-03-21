# Day 13：Extensions + Skills + UI

> **目標**：理解 Extension 架構、Skill 定義、Web UI、Native App 架構
> **C# 對照**：NuGet Plugin、Roslyn Source Generator、Blazor、MAUI
> **預計時間**：4 小時

---

## 13.1 Extensions — 可插拔擴充

### 目錄結構

每個 extension 是一個獨立的 pnpm package：

```
extensions/
  ├── matrix/
  │   ├── package.json      ← Plugin 描述 + 依賴
  │   ├── src/
  │   │   ├── index.ts      ← 入口：export plugin 定義
  │   │   ├── monitor.ts    ← Channel Monitor（持續監聽訊息）
  │   │   ├── send.ts       ← 發送訊息
  │   │   └── config.ts     ← 設定 schema
  │   └── tsconfig.json
  ├── slack/
  ├── telegram/
  ├── msteams/
  └── ... (35+ extensions)
```

### Extension 入口範例

```typescript
// extensions/matrix/src/index.ts
import type { ChannelPlugin } from "openclaw/plugin-sdk";

const matrixPlugin: ChannelPlugin = {
  id: "matrix",
  type: "channel",
  meta: {
    label: "Matrix",
    description: "Matrix chat protocol",
    docsPath: "/channels/matrix",
    aliases: ["element"],     // Element 也是 Matrix 客戶端
  },
  capabilities: {
    chatTypes: ["direct", "group"],
    reactions: true,
    media: true,
    threads: true,
  },
  outbound: { textChunkLimit: 4000 },

  // 建立 Monitor
  createMonitor: (config) => new MatrixMonitor(config),

  // 發送訊息
  sendMessage: async (params) => {
    await matrixClient.sendMessage(params.target, params.text);
  },
};

export default matrixPlugin;
```

```csharp
// C# 等價
[Export(typeof(IChannelPlugin))]
public class MatrixPlugin : IChannelPlugin
{
    public string Id => "matrix";
    public ChannelMeta Meta => new() { Label = "Matrix", ... };
    public ChannelCapabilities Capabilities => new() { ... };

    public IChannelMonitor CreateMonitor(Config config)
        => new MatrixMonitor(config);

    public async Task SendMessageAsync(SendMessageParams p)
        => await _client.SendMessageAsync(p.Target, p.Text);
}
```

---

## 13.2 Extension 的 35+ 種類

| 類別 | Extensions | 說明 |
|------|-----------|------|
| 聊天平台 | matrix, msteams, slack, telegram, discord, line, feishu, zalo, nostr, nextcloud-talk, mattermost, tlon, irc, googlechat | Channel plugins |
| 通訊 | whatsapp, signal, imessage, bluebubbles | 加密通訊 |
| 語音 | talk-voice, voice-call | 語音通話 |
| AI 整合 | llm-task, copilot-proxy | AI 工具 |
| 認證 | google-antigravity-auth, google-gemini-cli-auth, minimax-portal-auth, qwen-portal-auth | OAuth/API Key 管理 |
| 功能 | memory-core, memory-lancedb, device-pair, diagnostics-otel, open-prose | 輔助功能 |

---

## 13.3 Skills — AI 工具模組

### Skill 結構

```
skills/
  ├── github/
  │   ├── skill.yaml       ← Skill 描述檔
  │   └── gh.sh            ← 實際的工具腳本
  ├── notion/
  │   ├── skill.yaml
  │   └── notion-cli.sh
  ├── spotify-player/
  │   ├── skill.yaml
  │   └── sp.sh
  └── ... (50+ skills)
```

### skill.yaml 範例

```yaml
# skills/github/skill.yaml
name: github
description: "Interact with GitHub repositories, issues, and pull requests"
tools:
  - name: gh
    description: "Run GitHub CLI commands"
    command: "gh"
    allowedSubcommands:
      - "issue"
      - "pr"
      - "repo"
      - "api"
    examples:
      - "gh issue list"
      - "gh pr create --title 'Fix bug'"
```

### Skill 在 AI 對話中的使用

```
使用者: "幫我看一下 openclaw repo 上有沒有 open 的 issue"

AI 思考: 我有 github skill，可以用 gh CLI
AI Tool Call: gh issue list --repo openclaw/openclaw --state open
[執行結果回傳]

AI 回覆: "目前有 23 個 open issue，前 5 個是..."
```

---

## 13.4 Web UI — Lit + Vite

### 技術棧

```
ui/
  ├── package.json     ← Lit 3.x + Vite 7.x
  ├── vite.config.ts   ← 建構設定
  ├── index.html       ← SPA 入口
  └── src/
      ├── main.ts      ← App 入口
      ├── styles/      ← CSS
      └── ui/          ← Web Components
```

### Lit Web Component 範例

```typescript
// Lit：Google 的 Web Component 框架
import { LitElement, html, css } from "lit";
import { customElement, property } from "lit/decorators.js";

@customElement("chat-message")
class ChatMessage extends LitElement {
  @property() role: string = "user";
  @property() content: string = "";

  static styles = css`
    .message { padding: 8px; margin: 4px 0; }
    .user { background: #e3f2fd; }
    .assistant { background: #f5f5f5; }
  `;

  render() {
    return html`
      <div class="message ${this.role}">
        <strong>${this.role}:</strong>
        <span>${this.content}</span>
      </div>
    `;
  }
}
```

```csharp
// C# Blazor 等價
@* ChatMessage.razor *@
<div class="message @Role">
    <strong>@Role:</strong>
    <span>@Content</span>
</div>

@code {
    [Parameter] public string Role { get; set; } = "user";
    [Parameter] public string Content { get; set; } = "";
}
```

### Lit vs Blazor 對照

| Lit | Blazor | 說明 |
|-----|--------|------|
| `LitElement` | `ComponentBase` | 基底類別 |
| `@property()` | `[Parameter]` | 外部傳入的屬性 |
| `html\`...\`` | `.razor` 模板 | 模板語法 |
| `css\`...\`` | `::deep` / CSS isolation | 樣式隔離 |
| `@customElement("x")` | `<X />` | 自訂元素名稱 |

---

## 13.5 Native Apps — Swift / Kotlin

### macOS App (Swift)

```
apps/macos/
  ├── Package.swift          ← Swift Package Manager（≈ .csproj）
  ├── Sources/
  │   ├── OpenClawApp.swift  ← SwiftUI App 入口
  │   ├── GatewayClient.swift ← WebSocket 連線到 Gateway
  │   ├── ChatView.swift     ← 聊天介面
  │   └── SessionList.swift  ← Session 列表
  └── Tests/
```

### iOS App (Swift)

```
apps/ios/
  ├── project.yml            ← xcodegen 設定（≈ .csproj）
  ├── Sources/
  │   ├── OpenClawApp.swift
  │   └── ...
  └── Tests/
```

### Android App (Kotlin)

```
apps/android/
  ├── build.gradle.kts       ← Gradle（≈ .csproj）
  ├── app/src/main/
  │   ├── java/.../MainActivity.kt
  │   └── res/
  └── ...
```

### 共通點

所有 Native App 都是 **Gateway Client**——透過 WebSocket 連到 Gateway，取得 AI 對話。

```csharp
// C# MAUI 等價概念
// 如果用 MAUI 重寫，一個專案打三平台
public class GatewayClient
{
    private ClientWebSocket _ws = new();

    public async Task ConnectAsync(string gatewayUrl, string token)
    {
        _ws.Options.SetRequestHeader("Authorization", $"Bearer {token}");
        await _ws.ConnectAsync(new Uri(gatewayUrl), CancellationToken.None);
    }

    public async IAsyncEnumerable<GatewayEvent> ListenAsync(...)
    {
        // 持續接收 WebSocket 訊息
    }
}
```

---

## 今日閱讀作業

### 作業 1：挑 2 個 Extension 閱讀
- 建議：`extensions/line/` 和 `extensions/matrix/`
- 看 `package.json` 裡的 `openclaw.plugin` 欄位
- 看 `src/index.ts` 的 plugin 定義

### 作業 2：挑 3 個 Skill 閱讀
- 建議：`skills/github/`、`skills/weather/`、`skills/coding-agent/`
- 看 `skill.yaml` 的結構

### 作業 3：瀏覽 `ui/src/`
- 理解 Lit 的 Web Component 寫法
- 對照你熟悉的 Blazor

---

## 今日 Checkpoint

1. Extension 和 Skill 有什麼差異？
2. 一個 Channel Extension 至少要 export 什麼？
3. Skill 的工具定義寫在哪裡？
4. Lit 的 `@property()` 對應 Blazor 的什麼？
5. Native App 怎麼和 Gateway 通訊？

---

## 答案

1. **Extension 是系統級擴充**（Channel、認證、語音等）；**Skill 是 AI 可呼叫的工具**（GitHub、天氣、音樂等）。
2. Plugin 定義物件，包含 `id`、`type`、`capabilities`、`createMonitor()`、`sendMessage()`。
3. `skill.yaml` 檔案裡的 `tools` 區塊。
4. `[Parameter]` — 從父元件傳入的資料。
5. **WebSocket** — 連到 Gateway 的 WebSocket server，用 JSON 交換訊息。
