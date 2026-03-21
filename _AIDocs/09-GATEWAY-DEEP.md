# Gateway HTTP Pipeline + Chat Event + Routing 深入

> Phase 2-1 | 掃描範圍：`src/gateway/` 50+ server-*.ts + `src/routing/` 11 files
> 更新：2026-03-13

---

## 目錄

1. [Gateway HTTP Server 架構](#1-gateway-http-server-架構)
2. [13-Stage HTTP Pipeline 逐層深入](#2-13-stage-http-pipeline-逐層深入)
3. [WebSocket Upgrade 與 WS 連線生命週期](#3-websocket-upgrade-與-ws-連線生命週期)
4. [Chat Event 系統深入](#4-chat-event-系統深入)
5. [WS Method Dispatch 與 RPC 協定](#5-ws-method-dispatch-與-rpc-協定)
6. [Broadcasting 機制](#6-broadcasting-機制)
7. [Routing 引擎深入](#7-routing-引擎深入)
8. [Channel 生命週期管理](#8-channel-生命週期管理)
9. [Gateway Startup / Shutdown 完整流程](#9-gateway-startup--shutdown-完整流程)
10. [維護定時器與 Hot Reload](#10-維護定時器與-hot-reload)
11. [Auth 與 Rate Limiting](#11-auth-與-rate-limiting)
12. [Security Headers](#12-security-headers)
13. [邊界條件與陷阱](#13-邊界條件與陷阱)
14. [關鍵常量速查](#14-關鍵常量速查)

---

## 1. Gateway HTTP Server 架構

### 核心三件套

| 檔案 | 行數 | 職責 |
|------|------|------|
| `gateway/server.impl.ts` | ~4000+ | 主啟動器，組裝所有子系統 |
| `gateway/server-http.ts` | 843 | HTTP request pipeline + WebSocket upgrade |
| `gateway/server-chat.ts` | 649 | Agent event → Chat event 橋接，delta streaming |

### 輔助模組矩陣

| 模組 | 職責 |
|------|------|
| `server-broadcast.ts` | WS 廣播引擎，scope guard + slow consumer 保護 |
| `server-channels.ts` | Channel 生命週期（start/stop/restart/snapshot） |
| `server-startup.ts` | Sidecar 啟動（browser control, gmail, hooks, channels, plugins, ACP, memory） |
| `server-close.ts` | Graceful shutdown 全流程 |
| `server-methods.ts` | WS RPC dispatcher，25+ handler groups |
| `server-methods-list.ts` | 方法清單 + 事件清單宣告 |
| `server-ws-runtime.ts` | WS connection handler 組裝薄層 |
| `server-maintenance.ts` | tick / health refresh / dedupe cleanup / media cleanup 定時器 |
| `server-cron.ts` | Cron service builder + webhook + isolated agent |
| `server-reload-handlers.ts` | Config hot-reload handler（channels/hooks/cron/heartbeat/lanes） |
| `server-lanes.ts` | CommandLane 並行度設定（Main/Subagent/Cron） |
| `server-node-events.ts` | Node event handler（voice transcript, push, agent dispatch） |
| `server-session-key.ts` | Session key resolution helpers |

---

## 2. 13-Stage HTTP Pipeline 逐層深入

`server-http.ts` 的 `createGatewayHttpServer` 建立 `handleRequest` 函數，內部依序執行 `GatewayHttpRequestStage[]`。**第一個回傳 `true` 的 stage 吃掉請求**，後續不執行。

### Pipeline 總覽

```
Security Headers (always)
  ↓
WebSocket upgrade? → skip (ws handles 'upgrade' event)
  ↓
Canvas scoped URL normalize
  ↓
[Stage Pipeline — first-match-wins]
  ↓
1. hooks         — Webhook ingress (/hooks/*)
2. tools-invoke  — Tool invocation HTTP API (/api/tools/invoke)
3. slack         — Slack Events API callback
4. openresponses — OpenResponses /v1/responses (conditional)
5. openai        — OpenAI /v1/chat/completions (conditional)
6. canvas-auth   — Canvas path auth enforcement (conditional)
7. a2ui          — A2UI bundle serving (conditional)
8. canvas-http   — Canvas host HTTP (conditional)
9. plugin-auth   — Plugin route gateway auth (conditional)
10. plugin-http  — Plugin HTTP route handler (conditional)
11. control-ui-avatar — Agent avatar serving (conditional)
12. control-ui-http — Control UI SPA catch-all (conditional)
13. gateway-probes — /health, /healthz, /ready, /readyz
  ↓
None matched → 404 Not Found
  ↓
catch → 500 Internal Server Error
```

### 逐層細節

#### Stage 1: hooks
- **路徑**：`{basePath}/*`（預設 `/hooks`）
- **入口**：`createHooksRequestHandler()` → 閉包回傳 `HooksRequestHandler`
- **驗證流程**：
  1. 檢查 hooksConfig 是否存在
  2. 路徑 prefix 匹配 `basePath`
  3. 禁止 `?token=` query（強制 header auth）
  4. 僅允許 POST
  5. `extractHookToken()` + `safeEqualSecret()` constant-time 比對
  6. Rate limit：`hookAuthLimiter`（20次/60秒/IP，非 loopback 免除）
- **子路由**：
  - `/hooks/wake` → `dispatchWakeHook({ text, mode: "now"|"next-heartbeat" })`
  - `/hooks/agent` → 驗證 agent policy → resolve session key → `dispatchAgentHook()`
  - **Hook Mappings**：`hooksConfig.mappings[]` → `applyHookMappings()` 轉換外部 payload 為 wake/agent action
- **Body limit**：`hooksConfig.maxBodyBytes`

#### Stage 2: tools-invoke
- **路徑**：`/api/tools/invoke`
- **認證**：Gateway auth（Bearer token）
- **功能**：HTTP 呼叫 AI tools（memory_search 等），支援 `dryRun` 模式
- **安全**：`DEFAULT_GATEWAY_HTTP_TOOL_DENY` 黑名單 + tool policy pipeline

#### Stage 3: slack
- **路徑**：Slack Events API standard paths
- **委派**：`handleSlackHttpRequest()` from `../slack/http/`

#### Stage 4: openresponses（條件啟用）
- **條件**：`openResponsesEnabled === true`
- **路徑**：`/v1/responses`
- **認證**：Gateway auth
- **協定**：OpenResponses spec（`open-responses.com`）
- **支援 SSE streaming**

#### Stage 5: openai（條件啟用）
- **條件**：`openAiChatCompletionsEnabled === true`
- **路徑**：`/v1/chat/completions`
- **認證**：Gateway auth
- **功能**：OpenAI 相容 API，支援 conversation entries、image content、SSE streaming

#### Stage 6-8: canvas-auth / a2ui / canvas-http（條件啟用）
- **條件**：`canvasHost !== null`
- **canvas-auth**：`isCanvasPath()` → `authorizeCanvasRequest()`（支援 scoped capability token）
- **a2ui**：A2UI bundle 靜態資源
- **canvas-http**：Canvas host 的其他 HTTP 請求

#### Stage 9-10: plugin-auth / plugin-http（條件啟用）
- **條件**：`handlePluginRequest` 存在
- **plugin-auth**：
  - 跳過 Mattermost slash callback paths
  - `resolvePluginRoutePathContext()` 解析路徑
  - `shouldEnforceDefaultPluginGatewayAuth()` — malformed encoding / decode 上限 / protected path 需 auth
  - `enforcePluginRouteGatewayAuth()` — Bearer token 驗證
  - 成功 → 設 `pluginGatewayAuthSatisfied = true`
- **plugin-http**：`handlePluginRequest(req, res, pathContext, { gatewayAuthSatisfied })`

#### Stage 11-12: control-ui-avatar / control-ui-http（條件啟用）
- **條件**：`controlUiEnabled === true`
- **avatar**：`/api/avatar/{agentId}` → `resolveAgentAvatar()` → serve image
- **control-ui-http**：
  - `classifyControlUiRequest()` 路由分類
  - 靜態資源 + SPA catch-all（`index.html`）
  - CSP header：`buildControlUiCspHeader()`
  - Bootstrap config 注入：`/api/config`

#### Stage 13: gateway-probes
- **路徑**：`/health`, `/healthz`（liveness）; `/ready`, `/readyz`（readiness）
- **方法**：GET / HEAD only
- **Details 揭露**：local direct request 或 authenticated → 顯示完整 readiness details
- **readiness**：`getReadiness()` → 503 if not ready

### Pipeline 關鍵設計

| 設計 | 原因 |
|------|------|
| Stage array + first-match-wins | 優先級明確，避免路由衝突 |
| 條件推入而非全量宣告 | 未啟用的 stage 零成本 |
| plugin-auth 與 plugin-http 分離 | 先驗證再處理，避免未授權存取 plugin 端點 |
| Control UI SPA catch-all 排最後 | 避免吃掉 plugin/canvas/probe 路由 |
| Mattermost slash callback 豁免 | Mattermost webhook 自帶驗證 |

---

## 3. WebSocket Upgrade 與 WS 連線生命週期

### Upgrade Handler（`attachGatewayUpgradeHandler`）

```
httpServer.on("upgrade") →
  1. normalizeCanvasScopedUrl → malformed? → 401 + destroy
  2. rewrite URL if scoped
  3. Canvas WS path? (/canvas/ws) → authorizeCanvasRequest → fail? → 401 + destroy → canvasHost.handleUpgrade
  4. Default → wss.handleUpgrade → emit "connection"
```

### WS Connection 建立（`ws-connection.ts`）

```
wss.on("connection", (socket, upgradeReq)) →
  1. 生成 connId (UUID)
  2. 記錄 remoteAddr, host, origin, user-agent
  3. resolve canvasHostUrl
  4. 啟動 handshake timeout (getHandshakeTimeoutMs())
  5. 等待 "connect" method → authorizeGatewayConnect()
  6. 成功 → GatewayWsClient 加入 clients Set
  7. 註冊 message handler (attachGatewayWsMessageHandler)
  8. close → 從 clients 移除, cleanup presence
```

### GatewayWsClient 結構

```typescript
{
  connId: string;        // UUID
  socket: WebSocket;
  connect: {
    role: "operator" | "node" | "webchat";
    scopes: string[];    // operator.admin, operator.approvals, etc.
    version?: string;
    features?: string[];
  };
  openedAt: number;
  remoteAddr?: string;
}
```

---

## 4. Chat Event 系統深入

### 核心資料結構

#### ChatRunState
```typescript
{
  registry: ChatRunRegistry;     // sessionId → ChatRunEntry[] queue
  buffers: Map<string, string>;  // clientRunId → 累積 assistant text
  deltaSentAt: Map<string, number>;  // 上次 delta 廣播時間
  deltaLastBroadcastLen: Map<string, number>;  // 上次廣播的文字長度
  abortedRuns: Map<string, number>;  // 已 abort 的 run
}
```

#### ChatRunRegistry
- `add(sessionId, entry)` — 新增 chat run 到 queue
- `peek(sessionId)` — 查看 queue 首項
- `shift(sessionId)` — 彈出 queue 首項
- `remove(sessionId, clientRunId, sessionKey?)` — 按 ID 移除

#### ToolEventRecipientRegistry
- 追蹤每個 runId 對應的 WS connIds
- TTL 10 分鐘，finalized 後 30 秒 grace
- 用於 targeted broadcast（tool events 只送給相關 WS clients）

### Agent Event → Chat Event 轉換（`createAgentEventHandler`）

```
AgentEventPayload →
  1. resolve chatLink (registry.peek)
  2. resolve sessionKey, clientRunId
  3. check isAborted
  4. increment seq (with gap detection → broadcast seq-gap error)

  [tool event]
  5a. toolVerbose 決定 payload 是否包含 result/partialResult
  5b. tool-start → flushBufferedChatDeltaIfNeeded（確保 tool card 上方文字完整）
  5c. broadcastToConnIds → 只送給 registered recipients

  [non-tool event]
  5. broadcast("agent", agentPayload)

  [assistant stream]
  6. emitChatDelta:
     a. stripInlineDirectiveTagsForDisplay
     b. resolveMergedAssistantText（處理 text/delta 疊加）
     c. 過濾: SILENT_REPLY_TOKEN, silentLeadFragment, heartbeatHide
     d. 150ms throttle（now - last < 150 → skip）
     e. broadcast("chat", { state: "delta", ... })
     f. nodeSendToSession

  [lifecycle end/error]
  7. emitChatFinal:
     a. flushBufferedChatDeltaIfNeeded（確保最後 chunk 送出）
     b. normalizeHeartbeatChatFinalText → strip/suppress heartbeat ACK
     c. cleanup: buffers, deltaSentAt, deltaLastBroadcastLen
     d. broadcast("chat", { state: "final"|"error", ... })
     e. toolEventRecipients.markFinal + clearAgentRunContext
```

### Delta Streaming Protocol

```
Client ← Gateway:

{ event: "chat", payload: {
    runId: string,
    sessionKey: string,
    seq: number,
    state: "delta",
    message: {
      role: "assistant",
      content: [{ type: "text", text: "累積全文" }],
      timestamp: number
    }
}}

→ 150ms throttle（節省 WS 流量）
→ final 前 flush（確保不丟字元）

{ event: "chat", payload: {
    runId: string,
    sessionKey: string,
    seq: number,
    state: "final",
    stopReason?: string,
    message?: { ... }  // null if SILENT_REPLY
}}

{ event: "chat", payload: {
    runId: string,
    sessionKey: string,
    seq: number,
    state: "error",
    errorMessage?: string
}}
```

### 文字合併邏輯（`resolveMergedAssistantText`）

```
1. nextText startsWith previousText → 取 nextText（完整替換）
2. previousText startsWith nextText && !delta → 保留 previousText（避免回退）
3. has delta → appendUniqueSuffix（overlap detection 避免重複）
4. has nextText → 取 nextText
5. fallback → previousText
```

### Suppression 規則

| 條件 | 效果 |
|------|------|
| `isSilentReplyText(text, SILENT_REPLY_TOKEN)` | 完全靜默，不送 delta 也不送 final message |
| `isSilentReplyLeadFragment(text)` | 正在輸出 SILENT 前綴，暫不送 |
| `shouldHideHeartbeatChatOutput(runId)` | Heartbeat run + webchat visibility=off → 隱藏 |
| `normalizeHeartbeatChatFinalText` → `stripped.shouldSkip` | Heartbeat ACK 超出 maxAckChars → suppress final |

---

## 5. WS Method Dispatch 與 RPC 協定

### RPC 框架

```
Client → Gateway:
{ type: "request", id: "...", method: "chat.send", params: {...} }

Gateway → Client:
{ type: "response", id: "...", result: {...} }
or
{ type: "response", id: "...", error: { code, message } }
```

### Method Authorization

```
handleGatewayRequest →
  1. authorizeGatewayMethod(method, client):
     a. health → 免驗證
     b. parseGatewayRole(role) → null → "unauthorized role"
     c. isRoleAuthorizedForMethod(role, method) → false → "unauthorized role"
     d. role=node → pass (node has full access)
     e. scopes includes "operator.admin" → pass
     f. authorizeOperatorScopesForMethod(method, scopes)
  2. CONTROL_PLANE_WRITE_METHODS: config.apply / config.patch / update.run
     → consumeControlPlaneWriteBudget → 3 per 60s rate limit
  3. lookup handler → extraHandlers || coreGatewayHandlers
  4. withPluginRuntimeGatewayRequestScope → invokeHandler
```

### 25+ Handler Groups

| Group | 方法數 | 關鍵方法 |
|-------|--------|---------|
| connect | 1 | health |
| chat | 3 | chat.send, chat.abort, chat.history |
| channels | 2 | channels.status, channels.logout |
| config | 5 | config.get, config.set, config.apply, config.patch, config.schema |
| sessions | 5 | sessions.list, sessions.preview, sessions.patch, sessions.reset, sessions.delete |
| agents | 5 | agents.list, agents.create, agents.update, agents.delete, agents.files.* |
| models | 1 | models.list |
| cron | 5 | cron.list, cron.add, cron.update, cron.remove, cron.run |
| node | 7 | node.pair.*, node.rename, node.list, node.invoke, node.event |
| device | 5 | device.pair.*, device.token.rotate, device.token.revoke |
| send | 1 | send |
| agent | 3 | agent, agent.identity.get, agent.wait |
| wizard | 4 | wizard.start, wizard.next, wizard.cancel, wizard.status |
| exec-approvals | 6 | exec.approvals.get/set, exec.approval.request/waitDecision/resolve |
| tools-catalog | 1 | tools.catalog |
| skills | 3 | skills.status, skills.bins, skills.install/update |
| update | 1 | update.run |
| usage | 2 | usage.status, usage.cost |
| tts | 5 | tts.* |
| system | 2 | system-presence, system-event |
| push | 0+ | push registration |
| web | 1+ | web-specific handlers |
| browser | 1 | browser.request |
| voicewake | 2 | voicewake.get, voicewake.set |
| talk | 2 | talk.config, talk.mode |

### Gateway Events（WS push）

```
connect.challenge, agent, chat, presence, tick, shutdown,
health, heartbeat, cron, node.pair.requested/resolved,
node.invoke.request, device.pair.requested/resolved,
voicewake.changed, exec.approval.requested/resolved, update.available
```

---

## 6. Broadcasting 機制

### `createGatewayBroadcaster`（`server-broadcast.ts`）

```typescript
broadcast(event, payload, opts?) →
  1. clients.size === 0 → skip
  2. seq++（targeted broadcast 不遞增 seq）
  3. frame = JSON.stringify({ type: "event", event, payload, seq, stateVersion })
  4. for each client:
     a. targetConnIds filter（targeted 模式只送指定 connIds）
     b. hasEventScope(client, event) — scope guard
     c. bufferedAmount > MAX_BUFFERED_BYTES:
        - dropIfSlow=true → skip this client
        - else → close(1008, "slow consumer")
     d. socket.send(frame)
```

### Scope Guard

| Event | Required Scope |
|-------|---------------|
| exec.approval.requested/resolved | operator.approvals |
| device.pair.requested/resolved | operator.pairing |
| node.pair.requested/resolved | operator.pairing |
| 其他 | 無限制（任何 connected client） |

role=operator + scopes.includes("operator.admin") → bypass all scope checks

### Targeted Broadcast（`broadcastToConnIds`）

- Tool events 只送給 `toolEventRecipients` 登記的 connIds
- 不遞增全域 seq

---

## 7. Routing 引擎深入

### 核心檔案

| 檔案 | 行數 | 職責 |
|------|------|------|
| `routing/resolve-route.ts` | 805 | 7-tier binding match + session key 生成 |
| `routing/session-key.ts` | 254 | Session key 構建 + normalize |
| `routing/bindings.ts` | 115 | Binding 解析 + account lookup |
| `routing/account-id.ts` | — | Account ID normalize |
| `routing/account-lookup.ts` | — | Account lookup helpers |

### 7-Tier 匹配演算法完整流程

```
resolveAgentRoute(input) →
  1. normalize: channel, accountId, peer, guildId, teamId, memberRoleIds, dmScope
  2. check route cache (WeakMap per config object, max 4000 entries)
  3. getEvaluatedBindingsForChannelAccount(cfg, channel, accountId)
     → merge account-scoped + wildcard(*) bindings in source order
  4. buildEvaluatedBindingsIndex → 分桶索引

  [Tier 走訪 — first match wins]
  Tier 1: binding.peer      — collectPeerIndexedBindings(peer) → exact peer match
  Tier 2: binding.peer.parent — collectPeerIndexedBindings(parentPeer) → thread inherit
  Tier 3: binding.guild+roles — byGuildWithRoles[guildId] + memberRoleIds 交集 (any)
  Tier 4: binding.guild      — byGuild[guildId] (no roles constraint)
  Tier 5: binding.team       — byTeam[teamId]
  Tier 6: binding.account    — accountPattern !== "*" (exact account)
  Tier 7: binding.channel    — accountPattern === "*" (channel default)

  5. No match → resolveDefaultAgentId(cfg) → "default"
  6. choose(agentId, matchedBy):
     a. pickFirstExistingAgentId → verify agent exists in config
     b. buildAgentSessionKey → session key
     c. buildAgentMainSessionKey → main session key
     d. deriveLastRoutePolicy
```

### Binding Index 資料結構

```typescript
EvaluatedBindingsIndex {
  byPeer: Map<"direct:alice"|"group:123", EvaluatedBinding[]>;
  byGuildWithRoles: Map<guildId, EvaluatedBinding[]>;
  byGuild: Map<guildId, EvaluatedBinding[]>;
  byTeam: Map<teamId, EvaluatedBinding[]>;
  byAccount: EvaluatedBinding[];   // non-wildcard account bindings
  byChannel: EvaluatedBinding[];   // wildcard (*) account bindings
}
```

### Peer 匹配規則

- `group` ↔ `channel` 互通（`peerKindMatches`）
- Lookup keys: `peerLookupKeys(kind, id)` → `["group:123", "channel:123"]`

### Session Key 構建邏輯（`buildAgentPeerSessionKey`）

```
direct (DM):
  dmScope=main → "agent:{agentId}:{mainKey}"
  dmScope=per-peer → "agent:{agentId}:direct:{peerId}"
  dmScope=per-channel-peer → "agent:{agentId}:{channel}:direct:{peerId}"
  dmScope=per-account-channel-peer → "agent:{agentId}:{channel}:{accountId}:direct:{peerId}"

group/channel:
  "agent:{agentId}:{channel}:{peerKind}:{peerId}"

thread:
  "{baseSessionKey}:thread:{normalizedThreadId}"
```

### Identity Links

```yaml
session:
  identityLinks:
    alice: ["alice@example.com", "telegram:123", "discord:789"]
```

`resolveLinkedPeerId()` → 匹配 `peerId` 或 `channel:peerId` → 回傳 canonical name → 統一 session key

### 快取策略

| 快取 | Key | Max Size | 失效條件 |
|------|-----|----------|---------|
| Evaluated Bindings | channel\taccountId | 2000 | cfg.bindings 改變 |
| Resolved Routes | channel\t...7 fields | 4000 | bindings/agents/session 改變 |
| Agent Lookup | cfg object (WeakMap) | — | cfg.agents 改變 |

全部使用 `WeakMap<OpenClawConfig, ...>` → config 物件 GC 時自動清除

---

## 8. Channel 生命週期管理

### `createChannelManager`（`server-channels.ts`）

```
ChannelManager {
  startChannels()         — 啟動所有 channel plugins
  startChannel(id, acct?) — 啟動指定 channel
  stopChannel(id, acct?)  — 手動停止
  getRuntimeSnapshot()    — 取得全 channel 狀態快照
  markChannelLoggedOut()  — 標記登出
  isManuallyStopped()     — 是否手動停止
  resetRestartAttempts()  — 重設重啟計數
}
```

### 啟動流程

```
startChannelInternal(channelId, accountId?) →
  1. getChannelPlugin(channelId) → plugin.gateway.startAccount?
  2. plugin.config.listAccountIds(cfg) → accountIds
  3. for each accountId:
     a. 已有 task running → skip
     b. plugin.config.isEnabled → false → setRuntime(enabled:false)
     c. plugin.config.isConfigured → false → setRuntime(configured:false)
     d. resetDirectoryCache
     e. create AbortController
     f. setRuntime(running:true, lastStartAt)
     g. plugin.gateway.startAccount({cfg, accountId, account, runtime, abortSignal, log, ...})
     h. catch → setRuntime(lastError: message)
     i. finally → setRuntime(running:false, lastStopAt)
     j. auto-restart chain (see below)
```

### Auto-Restart 策略

```
ExponentialBackoff: 5s → 10s → 20s → ... → 5min (factor=2, jitter=0.1)
Max attempts: 10

on channel exit:
  1. manuallyStopped? → no restart
  2. attempts++ → exceed MAX_RESTART_ATTEMPTS? → give up
  3. computeBackoff → sleepWithAbort(delay)
  4. startChannelInternal(preserveRestartAttempts: true)
```

### Stop 流程

```
stopChannel(channelId, accountId?) →
  1. collect all known accountIds (aborts + tasks + config)
  2. for each:
     a. manuallyStopped.add(key)
     b. abort.abort()
     c. plugin.gateway.stopAccount?() — explicit shutdown hook
     d. await task
     e. cleanup: aborts.delete, tasks.delete
     f. setRuntime(running:false, restartPending:false)
```

### Runtime Snapshot 結構

```typescript
ChannelRuntimeSnapshot {
  channels: { [channelId]: ChannelAccountSnapshot };       // 預設 account
  channelAccounts: { [channelId]: { [accountId]: ... } };  // 全部 accounts
}

ChannelAccountSnapshot {
  accountId, enabled, configured, running, restartPending,
  connected?, lastStartAt?, lastStopAt?, lastError?,
  reconnectAttempts?
}
```

---

## 9. Gateway Startup / Shutdown 完整流程

### Startup（`startGatewaySidecars`）

```
1. Session lock cleanup (30min stale → remove)
2. Browser control server (if enabled)
3. Gmail watcher (hooks.gmail.account)
4. Validate hooks.gmail.model against model catalog
5. Load internal hook handlers (clearInternalHooks + loadInternalHooks)
6. Start channels (unless OPENCLAW_SKIP_CHANNELS=1)
7. Trigger internal hook: "gateway:startup" (250ms delay)
8. Start plugin services
9. ACP session identity reconcile (if acp.enabled)
10. Start memory backend
11. Restart sentinel wake check
```

### Shutdown（`createGatewayCloseHandler`）

```
close(opts?) →
  1. Bonjour stop
  2. Tailscale cleanup
  3. Canvas host close
  4. Canvas host server close
  5. Stop all channel plugins
  6. Stop plugin services
  7. Stop Gmail watcher
  8. Stop cron service
  9. Stop heartbeat runner
  10. Stop update checker
  11. Clear node presence timers
  12. broadcast("shutdown", { reason, restartExpectedMs })
  13. Clear intervals: tick, health, dedupe, media
  14. Unsubscribe: agent events, heartbeat events
  15. Clear chatRunState
  16. Close all WS clients (code 1012 "service restart")
  17. Stop config reloader
  18. Stop browser control
  19. Close WSS
  20. Close HTTP server(s): closeIdleConnections + close
```

---

## 10. 維護定時器與 Hot Reload

### 定時器（`startGatewayMaintenanceTimers`）

| 定時器 | 間隔 | 功能 |
|--------|------|------|
| tick | `TICK_INTERVAL_MS` | Keepalive broadcast |
| health | `HEALTH_REFRESH_INTERVAL_MS` | Health snapshot refresh + broadcast |
| dedupe | — | Dedupe cache cleanup (TTL `DEDUPE_TTL_MS`, max `DEDUPE_MAX`); agent run seq cleanup (max 10K) |
| media | `mediaCleanupTtlMs` | `cleanOldMedia()` 清理暫存媒體 |

### Config Hot Reload（`createGatewayReloadHandlers`）

```
applyHotReload(plan, nextConfig) →
  - reloadHooks → resolveHooksConfig
  - restartHeartbeat → heartbeatRunner.updateConfig
  - resetDirectoryCache
  - restartCron → stop + rebuild cron service
  - restartGmailWatcher → stop + start
  - reloadInternalHooks → clear + load
  - channelsToStop → stopChannel for each
  - channelsToStart → startChannel for each
  - updateLaneConcurrency → setCommandLaneConcurrency
  - restartBrowserControl → stop + start
  - channelHealthMonitor → restart if needed
  - restartGateway → deferGatewayRestartUntilIdle / emitGatewayRestart
```

### Lane 並行度

```
CommandLane.Main → resolveAgentMaxConcurrent(cfg)
CommandLane.Subagent → resolveSubagentMaxConcurrent(cfg)
CommandLane.Cron → cfg.cron.maxConcurrentRuns ?? 1
```

---

## 11. Auth 與 Rate Limiting

### Auth 模式（`ResolvedGatewayAuth`）

| Mode | 說明 |
|------|------|
| `none` | 無認證（不推薦） |
| `token` | Bearer token |
| `password` | 密碼 |
| `trusted-proxy` | 信任反代 |

附加：`allowTailscale` → Tailscale Whois 身份驗證

### Auth 驗證流程（`authorizeHttpGatewayConnect`）

```
1. rate limit check (per-IP)
2. mode=none → ok
3. mode=trusted-proxy → check trusted proxy list
4. Tailscale auth (if allowTailscale):
   a. Tailscale header auth (forwarded headers)
   b. Tailscale whois (IP → identity)
5. Token/password → safeEqualSecret (constant-time)
6. Device token auth
7. Failure → recordFailure
```

### Rate Limiter（`createAuthRateLimiter`）

```typescript
{
  maxAttempts: 10,      // default
  windowMs: 60_000,     // 1 min sliding window
  lockoutMs: 300_000,   // 5 min lockout
  exemptLoopback: true, // localhost 免除
}
```

**獨立 scope**：shared-secret / device-token / hook-auth 各自計數

### Hook Auth Limiter

```
maxAttempts: 20, windowMs: 60s, lockoutMs: 60s, exemptLoopback: false
```

### Control Plane Write Rate Limit

```
config.apply / config.patch / update.run → 3 per 60s
```

---

## 12. Security Headers

### 預設 Headers（`setDefaultSecurityHeaders`）

每個 HTTP response 都設定：

```
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
Permissions-Policy: camera=(), microphone=(), geolocation=()
Strict-Transport-Security: {configured}  // TLS only
```

**刻意省略**：X-Frame-Options / CSP — Canvas/A2UI 需要被 iframe 載入

### Control UI 特有

- CSP header：`buildControlUiCspHeader()`
- SPA index.html 注入 bootstrap config

---

## 13. 邊界條件與陷阱

1. **150ms Delta Throttle 丟字**：final 前必須 `flushBufferedChatDeltaIfNeeded`，否則最後一個 throttled chunk 會遺失。`deltaLastBroadcastLen` 防止重複 flush。

2. **SILENT_REPLY_TOKEN 前綴暫存**：正在輸出的文字如果是 SILENT token 的前綴（如 `"SILEN"`），不會送 delta。避免客戶端看到閃爍的殘字。

3. **Seq Gap Detection**：agent event seq 不連續時 broadcast seq-gap error event，但不中斷處理。

4. **WeakMap 快取失效**：Route/Binding 快取用 WeakMap 綁定 config 物件，config 重新載入（新物件）→ 快取自動失效。但快取上限 overflow → 全清重建。

5. **Tool-start flush**：tool 事件 phase=start 時先 flush 累積的 assistant text delta，確保 Control UI 能在 tool card 上方顯示完整的 pre-tool 文字。

6. **Heartbeat Visibility**：heartbeat run 的 chat output 是否顯示由 `resolveHeartbeatVisibility({ channel: "webchat" })` 決定，可被使用者配置隱藏。

7. **Mattermost Slash Callback 豁免**：Plugin auth stage 跳過 Mattermost callback paths，因為 Mattermost 自帶 token 驗證。配置中可自訂 callbackPath / callbackUrl。

8. **Canvas Scoped URL**：`normalizeCanvasScopedUrl()` 在 pipeline 最前面處理，malformed scoped path → 立即 401。合法 → rewrite `req.url`。

9. **Slow Consumer 保護**：`bufferedAmount > MAX_BUFFERED_BYTES` + `dropIfSlow=false` → 強制 close(1008)。Delta events 設 `dropIfSlow=true` → 只跳過不斷線。

10. **Hook Token 禁止 Query String**：`?token=` → 400。強制使用 `Authorization: Bearer` 或 `X-OpenClaw-Token` header，避免 token 洩漏到 access logs。

11. **Thread Tier 2（parent inherit）**：如果 thread peer 沒匹配到 binding，會用 parentPeer 再試一次，實現子討論串繼承父訊息的 agent routing。

12. **Identity Links 統一 Session**：多平台同一人（telegram:123 + discord:789）→ 解析為同一個 canonical peerId → 共用 session。需注意 dmScope=main 時 identity links 不生效。

---

## 14. 關鍵常量速查

| 常量 | 值 | 位置 |
|------|---|------|
| Delta throttle | 150ms | `server-chat.ts:373` |
| Hook auth limit | 20 / 60s | `server-http.ts:74-75` |
| Auth rate limit default | 10 / 60s, lockout 5min | `auth-rate-limit.ts:78-80` |
| Control plane write limit | 3 / 60s | `server-methods.ts:37` |
| Channel restart backoff | 5s → 5min, factor=2, jitter=0.1 | `server-channels.ts:13-18` |
| Channel max restart attempts | 10 | `server-channels.ts:19` |
| Tool event recipient TTL | 10 min | `server-chat.ts:246` |
| Tool event final grace | 30s | `server-chat.ts:247` |
| Binding cache max | 2000 entries | `resolve-route.ts:202` |
| Route cache max | 4000 entries | `resolve-route.ts:212` |
| Heartbeat ACK max chars | DEFAULT_HEARTBEAT_ACK_MAX_CHARS | `server-chat.ts:12` |
| Session lock stale threshold | 30 min | `server-startup.ts:32` |
| Body limit (tools-invoke) | 2 MB | `tools-invoke-http.ts:37` |
| WS slow consumer threshold | MAX_BUFFERED_BYTES | `server-broadcast.ts:1` (from constants) |
| Voice transcript dedupe window | 1.5s | `server-node-events.ts:27` |

---

## C# 概念對照

| OpenClaw (TS) | C# / .NET 對照 |
|---------------|---------------|
| HTTP stage pipeline (first-match) | ASP.NET Core Middleware Pipeline（但這裡是 first-match 而非 chain） |
| `runGatewayHttpRequestStages` | 類似 `app.Use()` 但回傳 bool 決定是否短路 |
| `GatewayWsClient` + broadcast | SignalR Hub + Groups |
| `ChatRunState` buffers + throttle | 類似自製 `BufferedStream` + `Throttle` |
| `ToolEventRecipientRegistry` | 類似 SignalR 的 per-connection subscription |
| `resolveAgentRoute` 7-tier | 類似 ASP.NET Policy-based Authorization + Custom Route Resolution |
| `EvaluatedBindingsIndex` | 類似 `Dictionary<string, List<T>>` 的多欄索引 |
| `WeakMap` cache | `ConditionalWeakTable<TKey, TValue>` |
| `AuthRateLimiter` sliding window | 類似 `SlidingWindowRateLimiter` (.NET 7+) |
| `ChannelManager` auto-restart | 類似 `IHostedService` + Polly retry policy |
| `CommandLane` concurrency | `SemaphoreSlim` 控制並行數 |
| `createGatewayCloseHandler` | `IHostApplicationLifetime.ApplicationStopping` + dispose chain |
