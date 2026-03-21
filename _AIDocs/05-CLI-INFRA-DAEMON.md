# CLI + Commands + Infra + Daemon

## 應用啟動流程

```
entry.ts
  ↓ 檢查主模組、設 process.title、安裝警告過濾
  ↓ 規範化環境變數、Windows argv
  ↓ 快速路徑：--version / --help
  ↓
runCli(argv)  [cli/run-main.ts]
  ↓ normalizeWindowsArgv()
  ↓ parseCliProfileArgs() → applyCliProfileEnv()
  ↓ loadDotEnv() → normalizeEnv()
  ↓ ensureOpenClawCliOnPath()
  ↓ assertSupportedRuntime()（Node/Bun 版本檢查）
  ↓ tryRouteCli(argv)  ← 快速路由（跳過 buildProgram）
  ↓
buildProgram()  [cli/program/build-program.ts]
  ↓ Commander.js 程式建構
  ↓ registerProgramCommands() → 18 個核心命令（lazy registration）
  ↓ program.parseAsync(argv)
```

## CLI 命令體系（18 類核心命令）

| 命令群 | 子命令 | 用途 |
|--------|--------|------|
| Setup | setup | 初始化配置 + agent workspace |
| Onboard | onboard | 互動式嚮導 |
| Configure | configure | 認證/頻道/gateway 互動設定 |
| Config | get/set/unset/file/validate | 非互動式配置 |
| Doctor | doctor | 健檢 + 快速修復 |
| Maintenance | start/stop/restart/logs/daemon/update/dashboard/reset/uninstall | 服務管理 |
| Message | send/read/manage | 訊息讀寫 |
| Memory | search/reindex | 記憶搜尋 |
| Agent | agent/agents | 單次執行 / 代理管理 |
| Status | status/health/sessions | 狀態監控 |
| Browser | — | 專用瀏覽器管理 |

### Lazy Registration 模式
1. 命令先註冊 placeholder
2. 首次執行時動態載入完整實作
3. 重新解析 argv → 完整執行

### Route-First 最佳化
常見命令（health/status/sessions/config get/models）直接路由，跳過 buildProgram，省 ~500ms。

## Infra 層（src/infra/ — 360 files）

| 模組 | 職責 |
|------|------|
| env.ts / dotenv.ts | 環境變數規範化 |
| exec.ts / spawn-utils.ts | 子流程執行 |
| ports.ts / bonjour.ts | 埠檢查 / mDNS 發現 |
| exec-approvals.ts / exec-safety.ts | 命令批准 / 危險偵測 |
| path-safety.ts / fs-safe.ts / file-lock.ts | 路徑驗證 / 安全 IO |
| outbound/ (17 files) | 多頻道投遞適配器 |
| heartbeat-*.ts | 心跳引擎 / 活躍時間 |
| update-*.ts | 版本檢查 / 更新執行 |
| provider-usage.*.ts | 供應商用量追蹤 |

### Outbound 投遞
```typescript
deliverOutboundMessage()
  → normalizePayloads()
  → loadChannelOutboundAdapter(channel)
  → sendMessage*(to, text, ...)
  → trackDeliveryStatus()
```

## Daemon 模式（src/daemon/ — 47 files）

### 跨平台服務管理
| 平台 | 實作 | 關鍵檔案 |
|------|------|---------|
| macOS | LaunchAgent | launchd.ts / launchd-plist.ts |
| Linux | systemd | systemd.ts / systemd-unit.ts |
| Windows | Scheduled Task | schtasks.ts |

### 統一 API
```typescript
GatewayService {
  install() / uninstall() / stop() / restart()
  isLoaded() / readCommand() / readRuntime()
}

resolveGatewayService() → 根據 process.platform 返回對應實作
```

## Process 管理（src/process/ — 14 files）

### Command Queue with Lanes
- `main` lane — 序列化所有操作
- `cron` lane — 低風險（定時任務），可與 main 並行
- `clearLane()` / `drainGateway()` — 重啟準備

### 重啟恢復
- restart-recovery.ts — 自動重啟故障服務
- restart-sentinel.ts — 重啟標記
- restart-stale-pids.ts — 清理陳舊 PID

## CLI DI 容器（cli/deps.ts）
```typescript
createDefaultDeps() → CliDeps {
  sendMessageWhatsApp, sendMessageTelegram,
  sendMessageDiscord, sendMessageSlack,
  sendMessageSignal, sendMessageIMessage
}
createOutboundSendDeps(deps) → OutboundSendDeps
```
