# Extensions + Skills

## Extensions（40 個，位於 extensions/）

### 頻道 Extensions（17 個）
```
bluebubbles / copilot-proxy / discord / feishu / googlechat / imessage /
irc / line / matrix / mattermost / msteams / nextcloud-talk / nostr /
signal / slack / synology-chat / telegram / tlon / twitch / whatsapp /
zalo / zalouser
```

### 功能 Extensions
| Extension | 用途 |
|-----------|------|
| acpx | Agent Client Protocol 擴展 |
| device-pair | 裝置配對 |
| diagnostics-otel | OpenTelemetry 診斷 |
| diffs | 差異比較 |
| google-gemini-cli-auth | Gemini CLI 認證 |
| llm-task | LLM 任務執行 |
| lobster | Lobster UI 整合 |
| memory-core | 記憶核心 |
| memory-lancedb | LanceDB 向量儲存 |
| minimax-portal-auth | Minimax 認證 |
| open-prose | 開放文本處理 |
| phone-control | 電話控制 |
| qwen-portal-auth | Qwen 認證 |
| shared | 共用工具 |
| talk-voice | 語音對話 |
| test-utils | 測試工具 |
| thread-ownership | 執行緒所有權 |
| voice-call | 語音通話 |

### Extension 結構（每個 extension）
```
extensions/{name}/
  ├── package.json      ← openclaw 欄位定義 manifest
  ├── src/
  │   ├── index.ts      ← register(api) + activate(api)
  │   ├── channel.ts    ← ChannelPlugin 實作（頻道類）
  │   └── ...
  └── tsconfig.json
```

---

## Skills（52 個，位於 skills/）

### 完整清單
```
1password / apple-notes / apple-reminders / bear-notes / blogwatcher /
blucli / bluebubbles / camsnap / canvas / clawhub / coding-agent /
discord / eightctl / gemini / gh-issues / gifgrep / github / gog /
goplaces / healthcheck / himalaya / imsg / mcporter / model-usage /
nano-banana-pro / nano-pdf / notion / obsidian / openai-image-gen /
openai-whisper / openai-whisper-api / openhue / oracle / ordercli /
peekaboo / sag / session-logs / sherpa-onnx-tts / skill-creator /
slack / songsee / sonoscli / spotify-player / summarize / things-mac /
tmux / trello / video-frames / voice-call / wacli / weather / xurl
```

### Skill 元資料結構
```typescript
SkillEntry = {
  skill: Skill                    // Pi framework Skill 物件
  frontmatter: {}                 // YAML metadata
  metadata?: {
    always?: boolean              // 永遠載入
    skillKey?: string             // 識別符
    emoji?: string                // UI icon
    requires?: {
      bins?: string[]             // 需要的二進位檔
      env?: string[]              // 需要的環境變數
      config?: string[]           // 需要的設定鍵
    }
    install?: [{
      kind: "brew"|"node"|"go"|"uv"|"download"
      package?: string
      bins?: string[]
    }]
  }
  invocation?: {
    userInvocable: boolean        // 使用者可呼叫 /skill
    disableModelInvocation: boolean
  }
}
```

### Skill 解析流程
1. 從 agent workspace 載入：`_SKILL.md` / `SKILL.md` / `skills/` dir
2. 根據 agent config 過濾：`agents[*].skills`（allow-pattern 或 exclusion）
3. 組裝 skills prompt 注入 system prompt

### Skill 呼叫機制
System prompt 指示 agent：
> Scan available_skills. If exactly one clearly applies, read its SKILL.md using `read`, then follow it.

Agent 讀取 skill → 遵循嵌入指令（自動化工作流）。
