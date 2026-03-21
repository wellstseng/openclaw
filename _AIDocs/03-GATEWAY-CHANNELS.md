# Gateway + Channel + Routing

## Gateway 三件套

| 檔案 | 角色 | 行數 |
|------|------|------|
| `gateway/server.impl.ts` | 主啟動器，初始化所有子系統 | 4000+ |
| `gateway/server-http.ts` | HTTP stage pipeline | 843 |
| `gateway/server-chat.ts` | WebSocket ↔ Agent 事件橋接 | 649 |
| `gateway/server-channels.ts` | Channel 生命週期管理 | — |

## HTTP Stage Pipeline（13 stages，順序決定優先級）
```
hooks → tools-invoke → slack → openresponses → openai →
canvas-auth → a2ui → canvas-http → plugin-auth → plugin-http →
control-ui-avatar → control-ui-http → gateway-probes → 404
```

## Chat Event 機制
- Delta streaming：150ms throttle
- Tool event filtering：根據 `toolEventRecipientRegistry`
- Heartbeat suppression：可配置隱藏

## Channel 三層架構

| 層 | 檔案 | 用途 | 何時載入 |
|----|------|------|---------|
| Registry（元資料）| `channels/registry.ts` | Channel ID/label/icon | 永遠 |
| Dock（行為配置）| `channels/dock.ts` (637行) | capabilities / textChunkLimit / threading | 輕量查詢 |
| Plugin（實作）| `channels/plugins/*.ts` | 實際連接 API、收發訊息 | 運行時 |

### 9 個內建 Channel
```
telegram / whatsapp / discord / irc / googlechat / slack / signal / imessage / line
```

### textChunkLimit（分塊策略）
- Telegram: 4000 chars
- Discord: 2000 chars
- 其他：各有不同

## Routing 引擎

### 7-Tier 綁定匹配（resolve-route.ts — 805 行）
```
1. binding.peer          → 精確 user match
2. binding.peer.parent   → 執行緒繼承
3. binding.guild+roles   → Discord guild + role
4. binding.guild         → Discord guild
5. binding.team          → Slack team
6. binding.account       → Channel account
7. binding.channel       → Channel 預設
→ default agent          → 全域預設
```

### Session Key 格式
```
agent:{agentId}:{scope}

DM:    agent:main:discord:default:direct:alice
Group: agent:main:telegram:group:engineering
Thread: agent:main:slack:channel:C123:thread:456
```

### dmScope 配置
| 模式 | Session Key 範例 | 效果 |
|------|-----------------|------|
| main | agent:main:main | 所有 DM 共用一個對話 |
| per-peer | agent:main:discord:direct:alice | 每人獨立對話 |
| per-channel-peer | agent:main:discord:direct:alice | channel+人 |
| per-account-channel-peer | agent:main:discord:default:direct:alice | 完全隔離 |

### Identity Links（多渠道同人）
```yaml
session:
  identityLinks:
    alice:
      - "alice@example.com"
      - "telegram:123456"
      - "discord:user789"
```

## Channel 生命週期（Gateway 視角）
1. Load plugins
2. For each enabled channel: `startAccount(ctx)`
3. Monitor: abort / errors / reconnects
4. Restart policy: exponential backoff 5s→5m，max 10 attempts

## 關鍵設計決策

| 決策 | 原因 |
|------|------|
| Dock 分離 | 避免載入 heavy channel code |
| 150ms delta throttle | 減少 WebSocket 流量 |
| Job queue per-sessionKey | 保證訊息順序 |
| WeakMap cache | GC-friendly，config 更新自動清除 |
| 7-tier routing | 覆蓋所有 multi-tenant 場景 |
