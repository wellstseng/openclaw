# Agent 主執行迴圈邏輯（run.ts — 1496 行）

> 這是 OpenClaw 最關鍵的檔案，所有 AI 呼叫都經過這裡。

## 函式簽名
```typescript
runEmbeddedPiAgent(params: RunEmbeddedPiAgentParams): Promise<EmbeddedPiRunResult>
```

## 完整邏輯流程圖

```
runEmbeddedPiAgent()
│
├── [1] Queue 管理（L253-273）
│   ├── resolveSessionLane() → 按 sessionKey 序列化
│   ├── resolveGlobalLane() → 全域 lane
│   └── enqueueSession → enqueueGlobal（雙層佇列嵌套）
│
├── [2] Workspace 解析（L274-294）
│   ├── resolveRunWorkspaceDir() → 工作目錄
│   ├── 若用了 fallback → log.warn
│   └── 記住 prevCwd（finally 時 chdir 回去）
│
├── [3] Model 解析（L292-369）
│   ├── provider = params.provider ?? "anthropic"
│   ├── modelId = params.model ?? "claude-opus-4-6"
│   ├── Hook: before_model_resolve → 可覆蓋 provider/model
│   ├── Hook: before_agent_start（legacy）→ 也可覆蓋
│   ├── resolveModel() → 取得 model 物件
│   └── 找不到 → throw FailoverError("model_not_found")
│
├── [4] Context Window 檢查（L371-402）
│   ├── resolveContextWindowInfo() → 計算可用 token
│   ├── effectiveModel = 若 config 限制 < 原生 → 用限制值
│   ├── evaluateContextWindowGuard()
│   │   ├── < WARN_BELOW(5000) → log.warn
│   │   └── < HARD_MIN(1000) → throw FailoverError（直接擋）
│
├── [5] Auth Profile 初始化（L404-706）
│   ├── ensureAuthProfileStore() → 從 ~/.openclaw/auth-profiles.json
│   ├── resolveAuthProfileOrder() → 排序 profile 優先級
│   ├── lockedProfileId = 若使用者指定 → 鎖定不輪替
│   ├── profileCandidates = locked ? [locked] : profileOrder
│   ├── Copilot 特殊處理：token 有效期 + refresh timer
│   │   ├── COPILOT_REFRESH_MARGIN_MS = 5min
│   │   ├── 到期前自動 refresh
│   │   ├── 失敗 60s 後 retry
│   │   └── scheduleCopilotRefresh() 遞迴排程
│   ├── 迴圈找第一個非 cooldown 的 profile
│   ├── 全部 cooldown → allowTransientCooldownProbe 可探測一次
│   └── 全部失敗 → throwAuthProfileFailover()
│
├── [6] 主執行迴圈 while(true)（L798-1487）
│   │
│   ├── 迴圈上限 = BASE(24) + profileCount × 8，最大 160
│   ├── 超過上限 → return error "Exceeded retry limit"
│   │
│   ├── [6a] runEmbeddedAttempt()（L840-911）
│   │   └── 傳入完整 params → 執行單次 agent turn
│   │
│   ├── [6b] Usage 累積（L931-944）
│   │   ├── mergeUsageIntoAccumulator() → 累加 input/output/cache
│   │   ├── 特殊：lastCacheRead/Write 只記最後一次（不累加）
│   │   │   原因：每次 tool loop 的 cacheRead ≈ 整個 context，
│   │   │   累加 N 次 = N × context，會膨脹（issue #13698）
│   │   └── lastTurnTotal = 最後 API 回報的 total
│   │
│   ├── [6c] Context Overflow 處理（L958-1147）
│   │   ├── 偵測：isLikelyContextOverflowError()
│   │   │   來源：promptError 或 assistantError
│   │   ├── 策略 1：若 attempt 內已 compact → 重試（不再 compact）
│   │   ├── 策略 2：explicit compact（最多 3 次）
│   │   │   └── contextEngine.compact({ force: true, target: "budget" })
│   │   ├── 策略 3：tool result truncation（只嘗試 1 次）
│   │   │   └── truncateOversizedToolResultsInSession()
│   │   └── 全部失敗 → return error "Context overflow"
│   │
│   ├── [6d] Prompt Error 處理（L1149-1255）
│   │   ├── Copilot auth error → refreshCopilotToken → retry
│   │   ├── Role ordering error → return 直接（不 retry）
│   │   ├── Image size error → return 直接（不 retry）
│   │   ├── Failover error → advanceAuthProfile() → retry
│   │   ├── Thinking level 不支援 → 降級 thinkLevel → retry
│   │   └── 有 fallback 配置 → throw FailoverError（讓外層處理）
│   │
│   ├── [6e] Assistant Error 處理（L1269-1375）
│   │   ├── 分類：auth / rateLimit / billing / failover / imageDimension
│   │   ├── shouldRotate = failover || (timeout && 非 compaction timeout)
│   │   ├── Rotate 成功 → advanceAuthProfile() → continue
│   │   ├── Rotate 失敗 + 有 fallback → throw FailoverError
│   │   └── Rotate 失敗 + 無 fallback → 繼續到成功路徑
│   │
│   └── [6f] 成功路徑（L1377-1487）
│       ├── toNormalizedUsage() → 計算最終 usage
│       ├── buildEmbeddedRunPayloads() → 格式化回覆
│       ├── timeout 且無 payload → return timeout error
│       ├── markAuthProfileGood() + markAuthProfileUsed()
│       └── return { payloads, meta, didSendViaMessagingTool, ... }
│
└── [7] Finally（L1488-1494）
    ├── contextEngine.dispose()
    ├── stopCopilotRefreshTimer()
    └── process.chdir(prevCwd)
```

## 關鍵常數

| 常數 | 值 | 用途 |
|------|-----|------|
| DEFAULT_PROVIDER | "anthropic" | 預設 LLM provider |
| DEFAULT_MODEL | "claude-opus-4-6" | 預設模型 |
| CONTEXT_WINDOW_HARD_MIN_TOKENS | ~1000 | 低於此直接擋 |
| CONTEXT_WINDOW_WARN_BELOW_TOKENS | 5000 | 低於此警告 |
| MAX_OVERFLOW_COMPACTION_ATTEMPTS | 3 | 最多 compact 3 次 |
| BASE_RUN_RETRY_ITERATIONS | 24 | 基礎重試 |
| RUN_RETRY_ITERATIONS_PER_PROFILE | 8 | 每個 profile 加 8 |
| MAX_RUN_RETRY_ITERATIONS | 160 | 全域上限 |
| OVERLOAD_BACKOFF | 250ms→1.5s, factor=2, jitter=0.2 | Overload 退避策略 |
| COPILOT_REFRESH_MARGIN_MS | 5 min | Copilot token 提前 refresh |
| COPILOT_REFRESH_RETRY_MS | 60s | Refresh 失敗後重試間隔 |

## 關鍵邊界條件 & 已知陷阱

1. **Usage 累積膨脹**（#13698）：每次 tool loop API call 都回報 cacheRead ≈ context size，N 次累加 = N × context。解法：只用最後一次 call 的 cache 值（lastCacheRead/lastCacheWrite）
2. **Compaction 後仍 overflow**：若 attempt 內已 auto-compact 但仍 overflow，不再立即 compact，而是 retry（避免無限 compact 迴圈，OC-65）
3. **Tool result truncation 只做一次**：`toolResultTruncationAttempted` flag 防止重複截斷
4. **Timeout 不記 profile failure**：timeout 是 network/model 問題，不是 auth 問題，記了會誤傷同 provider 的其他 model
5. **Copilot token 非同步 refresh**：用 `refreshInFlight` Promise 防止並發 refresh
6. **Anthropic magic string**：scrubAnthropicRefusalMagic() 過濾測試用 trigger string，防止污染 session
7. **雙層佇列嵌套**：session lane 包 global lane，確保同一 session 序列化，不同 session 可並行
8. **prevCwd restore**：finally 中 chdir 回原 cwd，防止 workspace 切換影響後續操作
