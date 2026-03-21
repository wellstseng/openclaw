# 15 — Security + Secrets + Config 深入

> Phase 4-2 | 掃描日期：2026-03-13
> 涵蓋：`src/security/` 30 files ~13.7K LOC + `src/secrets/` 47 files ~13K LOC + `src/config/` 199 files ~25K LOC
> 總計：276 files, ~51.7K LOC

---

## 目錄

1. [Security 安全框架](#1-security-安全框架)
2. [Secrets 密鑰管理](#2-secrets-密鑰管理)
3. [Config 設定系統](#3-config-設定系統)
4. [三模組整合關係](#4-三模組整合關係)
5. [邊界條件與陷阱](#5-邊界條件與陷阱)
6. [關鍵常量速查](#6-關鍵常量速查)
7. [C# 概念對照](#7-c-概念對照)

---

## 1. Security 安全框架

### 1.1 模組定位

**多層防禦體系**：審計 + 外部內容隔離 + SSRF 防護 + 代碼掃描 + 檔案系統權限 + DM 存取控制 + ReDoS 防護。非集中式 middleware，而是分散嵌入各子系統的安全閘門。

### 1.2 架構分層

```
┌─────────────────────────────────────────────────────────────────┐
│  Security Audit Orchestrator  (audit.ts, 1253 lines)            │
│  executeSecurityAudit() → SecurityAuditReport                   │
├──────────────────────┬──────────────────────────────────────────┤
│  Sync Collectors     │  Async Collectors                        │
│  (audit-extra.       │  (audit-extra.async.ts, 1314 lines)      │
│   sync.ts, 1349 ln)  │                                          │
│  ・Config-only 檢查  │  ・Filesystem 權限                       │
│  ・Model 風險        │  ・Plugin/Skill 代碼掃描                 │
│  ・Gateway 曝露      │  ・Docker image hash                     │
│  ・Node deny pattern │  ・Include 檔權限                        │
│  ・同步資料夾偵測    │  ・Symlink escape                        │
├──────────────────────┼──────────────────────────────────────────┤
│  Channel Security    │  External Content                        │
│  (audit-channel.ts   │  (external-content.ts, 341 lines)        │
│   725 lines)         │  ・Untrusted wrapper + markers           │
│  ・DM/Group policy   │  ・Injection pattern 偵測               │
│  ・Mutable identity  │  ・Unicode 反 spoofing                   │
│  ・Command gating    │  ・Sources: email/webhook/api/browser    │
├──────────────────────┴──────────────────────────────────────────┤
│  Foundation Layer                                                │
│  ・SSRF Guard (infra/net/ssrf.ts + fetch-guard.ts)              │
│  ・Skill Scanner (skill-scanner.ts, 583 lines)                  │
│  ・Safe Regex (safe-regex.ts, 332 lines)                        │
│  ・FS Permissions (audit-fs.ts + windows-acl.ts)                │
│  ・DM Policy (dm-policy-shared.ts, 332 lines)                  │
│  ・Dangerous Tools/Flags registry                               │
│  ・Timing-safe compare (secret-equal.ts)                        │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 Security Audit 系統

**入口**：`executeSecurityAudit(config)` — 整合 Sync + Async collectors。

**Sync Collectors**（config-only，無 I/O）：

| Collector | 檢查項 |
|-----------|--------|
| `collectAttackSurfaceSummaryFindings` | 整體曝露矩陣 |
| `collectExposureMatrixFindings` | 多使用者風險 |
| `collectGatewayHttpNoAuthFindings` | Gateway 無認證 = CRITICAL |
| `collectHooksHardeningFindings` | 外部 hook 安全 |
| `collectNodeDenyCommandPatternFindings` | Node 指令 allowlist 驗證 |
| `collectSecretsInConfigFindings` | 偵測硬編碼密鑰 |
| `collectSmallModelRiskFindings` | 小模型能力警告 |
| `collectSyncedFolderFindings` | iCloud/Dropbox/OneDrive 風險 |

**Async Collectors**（需 I/O）：

| Collector | 檢查項 |
|-----------|--------|
| `collectSandboxBrowserHashLabelFindings` | Docker image hash 驗證 |
| `collectIncludeFilePermFindings` | Include 檔案權限 |
| `collectInstalledSkillsCodeSafetyFindings` | 工作區 skill 掃描 |
| `collectPluginsCodeSafetyFindings` | 已安裝 plugin 掃描 |
| `collectPluginsTrustFindings` | Plugin allowlist/denylist |
| `collectStateDeepFilesystemFindings` | 深度檔案系統審計 |
| `collectWorkspaceSkillSymlinkEscapeFindings` | Skill symlink 逃逸 |

**Severity 分級**：critical → warn → info

### 1.4 SSRF 防護

**位置**：`src/infra/net/ssrf.ts` + `src/infra/net/fetch-guard.ts`

**封鎖規則**：

| 類型 | 範圍 |
|------|------|
| Private | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 |
| Loopback | 127.0.0.0/8, ::1 |
| Link-local | 169.254.0.0/16, fe80::/10 |
| Multicast | 224.0.0.0/4, ff00::/8 |
| Hostname | localhost, *.local, *.internal, metadata.google.internal |

**DNS Rebinding 防禦**：
1. DNS lookup → 驗證解析結果不在封鎖範圍
2. Pin resolved address → 後續請求使用 pinned IP（undici dispatcher）
3. 避免 TOCTOU：lookup ↔ connect 之間 DNS 不會被重新解析

**fetchWithSsrFGuard 流程**：
```
fetchWithSsrFGuard(url, options)
  → Parse URL
  → isBlockedHostnameOrIp(hostname) → 拒絕
  → resolvePinnedHostnameWithPolicy() → DNS 解析 + 驗證
  → createPinnedLookup() → 固定 IP
  → fetch with pinned dispatcher
  → Redirect 處理（max 3 次）
    → 每次 redirect 重新驗證目標 URL
    → Cross-origin redirect 移除 auth/cookie headers
    → 保留 safe headers: accept, content-type, range
```

**Policy 設定**：
```typescript
SsrFPolicy = {
  allowPrivateNetwork?: boolean,        // 允許私有網路
  dangerouslyAllowPrivateNetwork?: boolean,
  allowRfc2544BenchmarkRange?: boolean, // 198.18.0.0/15
  allowedHostnames?: string[],          // 明確允許
  hostnameAllowlist?: string[],         // 模式 "*.example.com"
}
```

**IPv6 安全**：embedded IPv4-in-IPv6（`::ffff:192.0.2.1`）正確偵測，malformed IPv6 fail-closed。

### 1.5 External Content 隔離

**入口**：`wrapExternalContent(source, content, metadata)`

**來源分類**：email, webhook, api, browser, channel_metadata, web_search, web_fetch

**防護機制**：
1. **Marker 系統**：隨機 16-char hex ID，標記 `<<<EXTERNAL_UNTRUSTED_CONTENT id="..."/>>>`
2. **Anti-spoofing**：偵測 + 替換 Unicode fullwidth、CJK 括號等仿冒 marker
3. **Injection 偵測**（17 patterns）：
   - "ignore all previous instructions"
   - "you are now a"
   - "system:"
   - "rm -rf", "delete all files"
   - `[system]`, `[assistant]` method injection
   - "elevated = true"
4. **Truncation**：channel_metadata 每項 400 chars，總計 800 chars

### 1.6 Code Safety Scanner

**入口**：`scanWorkspaceSkills(dir, options)`

**掃描規則**：

| Rule ID | Severity | 偵測 |
|---------|----------|------|
| `dangerous-exec` | CRITICAL | exec/spawn/execFile（requires child_process） |
| `dynamic-code-execution` | CRITICAL | eval(), new Function() |
| `crypto-mining` | CRITICAL | stratum+tcp/ssl, coinhive, xmrig |
| `suspicious-network` | WARN | WebSocket to 非標準 port |
| `potential-exfiltration` | WARN | readFile + fetch/post 組合 |
| `obfuscated-code` | WARN | 6+ hex escapes 或 大量 base64 + decode |
| `env-harvesting` | CRITICAL | process.env + fetch/post 組合 |

**限制**：max 500 files/scan, max 1MB/file, 跳過 node_modules + dotfiles

**快取**：FILE_SCAN_CACHE（5000 entries）+ DIR_ENTRY_CACHE（5000 entries），key = path + size + mtime

### 1.7 ReDoS 防護

**入口**：`compileSafeRegex(source, flags)`

**偵測邏輯**：`hasNestedRepetition()` — 保守的 token 解析器
- 阻擋：`(a+)+`, `(a|b?)*`, `(a+|b*)*`
- 放行：`a+`, `(a|b)+`, `a{1,10}`, `[a-z]*`

**快取**：SAFE_REGEX_CACHE（256 entries），LRU 淘汰

### 1.8 DM/Group 存取控制

**決策函式**：`resolveDmGroupAccessDecision()`

**DM Policy 模式**：

| 模式 | 行為 |
|------|------|
| `open` | 允許任何人（CRITICAL 風險） |
| `pairing` | 允許 configured + paired（from store）使用者 |
| `allowlist` | 僅 allowList 內使用者 |
| `disabled` | 忽略所有 DM |

**Mutable Identity 風險**：
- Discord name/tag → 可改，偏好 numeric ID / mention `<@ID>`
- Slack display name → 可改，偏好 `<@ID>` / U/W/B 開頭 ID
- 需 `dangerouslyAllowNameMatching` 明確 opt-in

### 1.9 Dangerous Tools Registry

**Gateway HTTP 預設 deny**（`dangerous-tools.ts`）：
```
sessions_spawn  — 遠端 agent spawn = RCE
sessions_send   — 跨 session 注入
cron            — 持久自動化控制面
gateway         — Gateway 重新設定
whatsapp_login  — 互動式 QR terminal
```

**ACP dangerous tools**：exec, spawn, shell, fs_write, fs_delete, fs_move, apply_patch

### 1.10 Filesystem 權限

**POSIX**：state dir symlink 偵測、world/group-writable 檢查、chmod 建議
**Windows**：`windows-acl.ts`（363 lines）— 解析 icacls 輸出，分類 principal（trusted/world/group），支援 localized 系統帳號名稱（法/德/西/葡）

**Auto-fix**（`fix.ts`, 477 lines）：`safeChmod()` + `safeAclReset()` + DM/Group policy 自動修正

---

## 2. Secrets 密鑰管理

### 2.1 模組定位

**Provider-based 密鑰解析框架**：無內建加密，靠三種 provider（env/file/exec）+ 路徑權限保護。完整生命週期：發現 → 設定 → 解析 → 審計。

### 2.2 三種 Provider

| Provider | 來源 | 持久化 | 安全機制 |
|----------|------|--------|---------|
| **env** | 環境變數 | 無（process memory） | Optional allowlist |
| **file** | JSON/單值檔案 | 磁碟 JSON 檔案 | 路徑權限檢查（owner/world/group） |
| **exec** | 外部腳本輸出 | 腳本自行管理 | 絕對路徑 + 權限驗證 + timeout |

### 2.3 Secret Resolution 資料流

```
SecretInput（config 值）
    │
    ▼
resolveSecretInputRef()
    ├─ Explicit ref: sibling "Ref" field（keyRef, tokenRef, serviceAccountRef）
    ├─ Inline ref: "${ENV_VAR}" 或 {source, provider, id} 物件
    └─ Plaintext: 直接字串（不需解析）
    │
    ▼
SecretRef { source, provider, id }
    │
    ▼
resolveSecretRefValues()
    ├─ 依 provider key 分組（source:provider）
    ├─ 並行呼叫 resolveProviderRefs()（concurrency limit: 4）
    │   ├─ env: process.env[id]
    │   ├─ file: JSON read → JSON pointer 存取
    │   └─ exec: spawn script → parse stdout JSON
    └─ 回傳: Map<refKey, resolved-value>
    │
    ▼
結果套用回 config / authStore（via assignment.apply()）
```

**快取策略**：
- Per-ref：`resolvedByRefKey: Map` — 避免重複解析
- Per-provider (file)：`filePayloadByProvider: Map` — 避免重複讀檔
- Exec provider：每次 spawn 新 process（不快取）

### 2.4 File Provider 詳解

**兩種模式**：
- `mode: "json"`（預設）：JSON 檔，用 JSON pointer id 存取（`/providers/openai/apiKey`）
- `mode: "singleValue"`：純文字檔，所有 ref id 必須為 `"value"`

**路徑安全檢查**：
1. 非 symlink（除非 `allowSymlinkPath=true`）
2. 當前使用者擁有（Unix only）
3. 非 world-readable/writable
4. 非 group-readable/writable（除非 `allowReadableByOthers=true`）
5. Windows：嘗試 ACL 驗證（或 ACL=unknown error）

### 2.5 Exec Provider 協議

**Request**（via stdin）：
```json
{ "protocolVersion": 1, "provider": "vault", "ids": ["key1", "key2"] }
```

**Response**（via stdout）：
```json
{
  "protocolVersion": 1,
  "values": { "key1": "secret-value" },
  "errors": { "key2": { "message": "not found" } }
}
```

**安全限制**：
- 必須絕對路徑
- 權限驗證同 file provider
- Timeout: 5s（可設定）
- No-output timeout：偵測 hang
- Max output: 1MB

### 2.6 Secret Target Registry

**`target-registry-data.ts`（749 lines）**：定義 ~100+ 密鑰位置。

**範例 target**：

| Target ID | Path Pattern | Shape |
|-----------|-------------|-------|
| `models.providers.*.apiKey` | models.providers.*.apiKey | secret_input |
| `channels.discord.token` | channels.discord.token | sibling_ref |
| `channels.slack.accounts.*.botToken` | channels.slack.accounts.*.botToken | sibling_ref |
| `gateway.auth.token` | gateway.auth.token | secret_input |
| `auth-profiles.api_key.key` | profiles.*.key | sibling_ref |

**Secret Shape**：
- `secret_input`：直接 `{source, provider, id}` 或 plaintext
- `sibling_ref`：獨立 `*Ref` 欄位，優先於 plaintext

### 2.7 Configure → Plan → Apply 生命週期

```
Interactive Configure (configure.ts, 977 lines)
  ├─ 載入 openclaw.json + auth-profiles.json
  ├─ buildConfigureCandidates() → 所有可發現密鑰
  ├─ 互動式提示 → 選擇 target + provider
  └─ 產出 plan.json

Plan Structure:
{
  version: 1, protocolVersion: 1,
  targets: [{ type, path, ref: {source, provider, id} }],
  providerUpserts?: { provider configs },
  providerDeletes?: [ aliases ]
}

Apply Plan (apply.ts, 777 lines)
  ├─ applyProviderPlanMutations()     — 新增/刪除 provider 設定
  ├─ applyConfigTargetMutations()     — 修改 openclaw.json
  ├─ applyAuthProfileTargetMutations() — 修改 auth-profiles.json
  ├─ scrubAuthStoresForProviderTargets() — 清理遷移殘留
  ├─ scrubLegacyAuthJsonStores()      — 刪除舊 auth.json
  ├─ scrubEnvFiles()                  — 從 .env 移除
  └─ validateProjectedSecretsState()  — Dry-run 驗證
```

### 2.8 Runtime Snapshot 生命週期

```
prepareSecretsRuntimeSnapshot()
  ├─ Clone config（sourceConfig + resolvedConfig）
  ├─ collectConfigAssignments() → 找出 config 中所有 secret ref
  ├─ collectAuthStoreAssignments() → auth profiles 中的 ref
  ├─ resolveSecretRefValues() → 批次解析（concurrent）
  ├─ applyResolvedAssignments() → 寫入 config + authStores
  └─ Return snapshot

activateSecretsRuntimeSnapshot(snapshot)
  → 存入 module-level activeSnapshot
  → getRuntimeConfigSnapshot() 可取得
```

### 2.9 Secrets Audit

**Finding Codes**：
- `PLAINTEXT_FOUND` — 密鑰以明文儲存
- `REF_UNRESOLVED` — ref 存在但 provider 未設定
- `REF_SHADOWED` — plaintext + ref 同時存在（ref 優先）
- `LEGACY_RESIDUE` — 舊 auth.json 仍存在

**掃描範圍**：.env、openclaw.json、auth-profiles.json、auth.json（legacy）、models.json

### 2.10 已知 Provider 環境變數

```
openai       → OPENAI_API_KEY
anthropic    → ANTHROPIC_API_KEY
google       → GEMINI_API_KEY
moonshot     → MOONSHOT_API_KEY
mistral      → MISTRAL_API_KEY
together     → TOGETHER_API_KEY
openrouter   → OPENROUTER_API_KEY
huggingface  → HUGGINGFACE_HUB_TOKEN / HF_TOKEN
volcengine   → VOLCANO_ENGINE_API_KEY
（26+ 總計）
```

---

## 3. Config 設定系統

### 3.1 模組定位

**JSON5 設定框架**：支援環境變數替換、遞迴 include、Zod schema 驗證、plugin 合併、runtime override。共 ~35 個 type 子模組，15+ Zod schema 檔案。

### 3.2 Config 載入管道（完整流程）

```
┌─ maybeLoadDotEnvForConfig()
│  └─ 載入 .env（僅限真實 process.env）
│
├─ fs.readFileSync(configPath) → JSON5.parse()
│
├─ resolveConfigIncludesForRead()
│  ├─ 遞迴 $include（max depth 10, max 2MB/file）
│  └─ deepMerge() 合併所有 include
│
├─ resolveConfigForRead()
│  ├─ applyConfigEnvVars() → config.env.vars 寫入 process.env
│  ├─ resolveConfigEnvVars() → ${VAR} 替換
│  └─ 捕獲 envSnapshotForRestore（寫回時還原）
│
├─ validateConfigObjectWithPlugins()
│  ├─ findLegacyConfigIssues() → 舊格式偵測
│  ├─ OpenClawSchema.safeParse() → Zod 驗證
│  ├─ validateIdentityAvatar() → Avatar 路徑安全
│  ├─ validateGatewayTailscaleBind() → 跨欄位約束
│  └─ Plugin manifest registry 驗證
│
├─ Apply runtime defaults（6 層）：
│  ├─ applyMessageDefaults()
│  ├─ applySessionDefaults()
│  ├─ applyLoggingDefaults()
│  ├─ applyAgentDefaults()
│  ├─ applyContextPruningDefaults() + applyCompactionDefaults()
│  ├─ applyTalkConfigNormalization()
│  └─ applyModelDefaults()
│
├─ normalizeConfigPaths() → ~ 和相對路徑解析
├─ normalizeExecSafeBinProfilesInConfig()
├─ findDuplicateAgentDirs() → 重複工作區偵測
├─ applyConfigEnvVars()（二次套用）
├─ loadShellEnvFallback()（可選）
├─ ensureOwnerDisplaySecret()（自動產生）
│
└─ applyConfigOverrides()
   └─ 套用 in-memory runtime overrides
```

### 3.3 Root Config 結構（OpenClawConfig）

| Top-level Key | 用途 |
|--------------|------|
| `meta` | 版本追蹤、timestamps |
| `env` | Shell env fallback、inline vars |
| `auth` | Auth provider profiles & order |
| `secrets` | Provider-based secret retrieval |
| `models` | Model 定義 & defaults |
| `agents` | Agent 定義 & defaults |
| `tools` | Tool 設定（最大區塊） |
| `channels` | 頻道設定（Discord/Slack/Telegram/...） |
| `gateway` | HTTP/TLS、auth、control-UI、nodes |
| `hooks` | HTTP hooks & transforms |
| `cron` | 排程任務 |
| `session` | Session limits、retention、pruning |
| `plugins` | Plugin manifest loading |
| `skills` | Skill loading、install、limits |
| `browser` | Browser automation config |
| `memory` | QMD memory system |
| `talk` | TTS provider config |
| `messages` | Ack reactions、defaults |
| `commands` | Native command control |
| `approvals` | Approval workflow |
| `bindings` | Agent routing/binding rules |
| `broadcast` | Multi-agent broadcast routing |
| `audio` | Audio config |
| `media` | File preservation & TTL |
| `diagnostics` | OTEL、cache traces |
| `logging` | Log level、file output、redaction |
| `acp` | Agent Control Protocol |
| `web` | WebSocket heartbeat |
| `discovery` | mDNS、wide-area discovery |
| `canvasHost` | Canvas server |
| `ui` | Accent color、assistant identity |
| `cli` | CLI banner |
| `update` | Update channel、auto-update |
| `nodeHost` | Node-host browser proxy |
| `wizard` | CLI wizard state |
| `sandbox` | Execution environment |

### 3.4 Zod Schema 組成

```
OpenClawSchema (zod-schema.ts, 888 lines)
  ├─ zod-schema.core.ts (730 lines)
  │  SecretRef, SecretInput, HexColor, DurationMs, ByteSize,
  │  SafeFilePath, NonEmptyTrimmedString, HostPattern...
  ├─ zod-schema.agent-runtime.ts (839 lines)
  │  AgentRuntimeConfig: heartbeat, context, model, tools...
  ├─ zod-schema.agent-defaults.ts (194 lines)
  │  Agent-level defaults (model, context limits)
  ├─ zod-schema.agents.ts (108 lines)
  │  Agents list + Audio/Broadcast
  ├─ zod-schema.providers-core.ts (1479 lines) ← 最大
  │  Discord, Slack, Telegram, WhatsApp, Signal, iMessage,
  │  Google Chat, IRC, MS Teams channel schemas
  ├─ zod-schema.providers-whatsapp.ts (170 lines)
  ├─ zod-schema.session.ts (214 lines)
  │  Session limits, send policy, commands
  ├─ zod-schema.hooks.ts (161 lines)
  │  HTTP hooks mapping & Gmail config
  ├─ zod-schema.allowdeny.ts (40 lines)
  ├─ zod-schema.approvals.ts (28 lines)
  ├─ zod-schema.channels.ts (10 lines)
  └─ zod-schema.installs.ts (22 lines)
```

**Schema 特性**：
- `.strict()` 禁止多餘 key
- `superRefine()` 跨欄位驗證（broadcast agentIds vs agents.list）
- Coercion：Unix timestamp → ISO string
- `parseDurationMs()`：`"30s"`, `"5m"`, `"2h"` → ms
- `parseByteSize()`：`"10mb"`, `"1gb"` → bytes

### 3.5 環境變數替換

**語法**：`${VAR_NAME}` → 大寫 env var
**跳脫**：`$${VAR_NAME}` → 字面 `${VAR_NAME}`
**Pattern**：`/^[A-Z_][A-Z0-9_]*$/`

**Write-time 還原**：
1. 讀取時記錄哪些路徑含 `${VAR}` 參照
2. 寫回時：未變動路徑 → 還原 `${VAR}`；已變動 → 寫入新值
3. 透過 `envSnapshotForRestore` 追蹤

### 3.6 Include 機制

**語法**：
```json5
{
  "$include": "./base.json5",                 // 單檔
  "$include": ["./a.json5", "./b.json5"],    // 多檔（合併）
  "$include": "#{glob-pattern}"              // Glob 展開
}
```

**Merge 策略**（`deepMerge()`）：
- Array：串接（非取代）
- Object：遞迴合併
- Primitive：source wins
- 封鎖 key：`__proto__`, `constructor`, `prototype`（防 prototype pollution）

**安全護欄**：
- Max depth: 10 層
- Max file: 2MB/include
- 循環偵測（CircularIncludeError）
- 路徑邊界檢查

### 3.7 三層驗證系統

| 層次 | 函式 | 時機 |
|------|------|------|
| 1. Legacy 偵測 | `findLegacyConfigIssues()` | Raw parse 後 |
| 2. Zod Schema | `OpenClawSchema.safeParse()` | Strict mode |
| 3. Custom 驗證 | Avatar 路徑、Tailscale bind、Plugin manifest | Post-Zod |

**Issue 映射**：`mapZodIssueToConfigIssue()` — Zod error → 可讀路徑 + 允許值提示

### 3.8 Plugin Config 合併

1. Base config 先驗證（`validateConfigObjectRaw`）
2. 載入 Plugin manifest registry
3. 每個 plugin 的 `config` 欄位依 plugin JSON schema 驗證
4. Plugin 自訂 channel ID 動態加入 allowlist
5. Plugin hooks config 驗證（`allowPromptInjection` per entry）

**無自動合併**：Plugin 透過 `normalizePluginsConfig()` 自行解析設定

### 3.9 Runtime Override

**API**：
```typescript
setConfigOverride("foo.bar.baz", value)    // 設定
unsetConfigOverride("foo.bar.baz")         // 移除
applyConfigOverrides(cfg)                  // 套用（最後步驟）
getConfigOverrides()                       // 讀取
resetConfigOverrides()                     // 清除全部
```

**特性**：套用在載入管道最後一步，path 解析禁止 prototype key

### 3.10 Merge Patch 系統

**策略**（`merge-patch.ts`）：
- `null` 值刪除 key
- Array：若所有項目有 `id: string` → 依 id 合併/追加
- Object：遞迴合併
- Blocked key：跳過（prototype pollution 防護）

### 3.11 Runtime Defaults

**套用順序**（`defaults.ts`, 534+ lines）：

| 順序 | 函式 | 內容 |
|------|------|------|
| 1 | `applyMessageDefaults()` | ackReactionScope = "group-mentions" |
| 2 | `applySessionDefaults()` | contextTokens, maxConcurrent |
| 3 | `applyLoggingDefaults()` | level: "info", consoleStyle: "pretty" |
| 4 | `applyAgentDefaults()` | Model, context, heartbeat |
| 5 | `applyContextPruningDefaults()` | Pruning strategy |
| 6 | `applyCompactionDefaults()` | Compaction window |
| 7 | `applyTalkConfigNormalization()` | Voice alias 展開 |
| 8 | `applyModelDefaults()` | Model aliases |

**Model Aliases**（`defaults.ts`）：
```
opus    → anthropic/claude-opus-4-6
sonnet  → anthropic/claude-sonnet-4-6
gpt     → openai/gpt-5.4
gpt-mini → openai/gpt-5-mini
gemini  → google/gemini-3.1-pro-preview
gemini-flash → google/gemini-3-flash-preview
```

### 3.12 Config I/O 安全

**寫入護欄**：
- 審計日誌：每次寫入記錄到 `logs/config-audit.jsonl`
- 可疑旗標：大小下降 >50%、missing meta、gateway mode 移除
- Env snapshot 驗證：僅在 expectedConfigPath 匹配時還原 `${VAR}`
- SHA256 hash tracking for change detection

**Config Snapshot 結構**：
```typescript
ConfigFileSnapshot = {
  path, exists, raw, parsed,
  resolved,    // After include + ${VAR}, BEFORE defaults
  config,      // Final with defaults
  hash,        // SHA256 of raw
  valid, issues, warnings, legacyIssues
}
```

---

## 4. 三模組整合關係

```
┌─────────────────────────────────────────────────────────────────┐
│                        Config System                             │
│  openclaw.json → load → validate → defaults → overrides          │
│  產出: OpenClawConfig（所有子系統的設定來源）                      │
└──────────┬──────────────────────────────────┬────────────────────┘
           │                                  │
           ▼                                  ▼
┌──────────────────────────┐    ┌──────────────────────────────────┐
│   Secrets System          │    │   Security System                │
│   (src/secrets/)          │    │   (src/security/)                │
│                           │    │                                  │
│  • 從 config 讀取        │    │  • 從 config 讀取設定            │
│    secret refs           │    │  • 審計 config 安全性            │
│  • 解析 → 寫回 config    │    │  • 審計 secrets 明文/未解析      │
│  • Provider 設定在       │    │  • SSRF guard 供全系統使用       │
│    config.secrets.*      │    │  • External content 隔離         │
│                           │    │  • Code safety 掃描             │
└──────────────────────────┘    └──────────────────────────────────┘
```

**關鍵整合點**：

| 呼叫方 | 被呼叫 | 介面 |
|--------|--------|------|
| Config loading | Secrets runtime | `prepareSecretsRuntimeSnapshot()` 解析 config 中的 refs |
| Security audit | Config | 讀取 config + includes snapshot |
| Security audit | Secrets | `collectSecretsInConfigFindings()` 偵測硬編碼密鑰 |
| Secrets audit | Config | 掃描 config 檔中的 plaintext secrets |
| Gateway/Browser/Media | SSRF guard | `fetchWithSsrFGuard()` |
| Hooks/Channels | External content | `wrapExternalContent()` |
| Plugin loader | Skill scanner | `scanWorkspaceSkills()` |
| Config validation | Security | 危險 flag 偵測 + DM policy 檢查 |

**依賴順序**：
```
Config 載入 → Secrets 解析（注入 config）→ Security 審計（驗證最終 config）
```

---

## 5. 邊界條件與陷阱

### Security

1. **SSRF DNS Rebinding** — pinned DNS 防禦，但 env proxy mode 信任 operator proxy，可能被繞過
2. **External content marker 碰撞** — 16-char hex 隨機 ID，碰撞機率極低但理論存在
3. **ReDoS false positive** — 保守解析器可能拒絕安全 regex（如複雜 character class）
4. **Mutable identity 降級** — `dangerouslyAllowNameMatching` opt-in 後審計僅 WARN 不 BLOCK
5. **Code scanner 盲區** — 不分析 minified/obfuscated code 語義，僅靠 pattern match
6. **Redirect header 洩漏** — cross-origin 移除 auth，但同 origin redirect 保留所有 headers
7. **Windows ACL 限制** — localized principal 名稱可能遺漏非支援語系

### Secrets

8. **無內建加密** — 密鑰明文儲存，安全靠路徑權限，exec provider 可包裝 KMS/Vault
9. **Ref 優先序** — explicit (sibling ref) > inline > plaintext，兩者同時存在只 warn 不 error
10. **File provider JSON pointer** — `~` → `~0`, `/` → `~1`（RFC 6901），容易忘記 escape
11. **Exec provider hang** — no-output timeout 偵測，但腳本寫 stderr 不觸發 kill
12. **Windows ACL 不可驗證** — 無可靠 API，除非 `allowInsecurePath=true` 否則 error
13. **Prototype pollution** — JSON pointer 路徑封鎖 `__proto__`, `prototype`, `constructor`
14. **Provider 遷移殘留** — apply 時 scrub 舊 provider credentials，但遺漏的路徑不會被清理

### Config

15. **Strict mode** — 未知 key → Zod error，Plugin 新增 key 需先 schema 註冊
16. **Include 循環** — 偵測 + 錯誤，但 glob include 可能意外引入自己
17. **Env var 二次套用** — 載入管道中 `applyConfigEnvVars()` 呼叫兩次（正規化前後），需注意副作用
18. **Duplicate agent dir** — fail-closed，兩個 agent 指向同一工作區 → error
19. **Config write 大小下降** — >50% drop 觸發 suspicious flag，但不阻擋寫入
20. **Model alias 解析** — `opus` → `claude-opus-4-6`，但若 provider 移除該模型 → silent fallback
21. **Array merge by id** — `deepMerge` array 串接，`mergeObjectArraysById` by id 合併，兩者語義不同
22. **Shell env fallback** — 可選 exec shell 取 env var，timeout 可設定，失敗 silent
23. **ownerDisplaySecret 自動產生** — 無密鑰時自動生成 + 非同步持久化，首次可能 race
24. **Include file permissions** — async audit 檢查 include 檔權限，但 sync 載入時不檢查

---

## 6. 關鍵常量速查

### Security

| 常量 | 值 | 位置 |
|------|-----|------|
| External content marker ID | 16 hex chars (8 bytes random) | `external-content.ts` |
| Channel metadata truncation | 400 chars/entry, 800 total | `channel-metadata.ts` |
| Skill scan max files | 500 (configurable) | `skill-scanner.ts` |
| Skill scan max file size | 1 MB | `skill-scanner.ts` |
| FILE_SCAN_CACHE max | 5,000 entries | `skill-scanner.ts` |
| DIR_ENTRY_CACHE max | 5,000 entries | `skill-scanner.ts` |
| SAFE_REGEX_CACHE max | 256 entries | `safe-regex.ts` |
| Suspicious patterns | 17 patterns | `external-content.ts` |
| Fetch redirect max | 3 | `fetch-guard.ts` |
| Dangerous gateway tools | 5 (sessions_spawn, sessions_send, cron, gateway, whatsapp_login) | `dangerous-tools.ts` |

### Secrets

| 常量 | 值 | 位置 |
|------|-----|------|
| Provider concurrency | 4 | `resolve.ts` |
| Max refs per provider | 512 | `resolve.ts` |
| Max batch bytes | 256 KB | `resolve.ts` |
| File max bytes | 1 MB | `resolve.ts` |
| File timeout | 5s | `resolve.ts` |
| Exec timeout | 5s | `resolve.ts` |
| Exec max output | 1 MB | `resolve.ts` |
| Provider alias pattern | `/^[a-z][a-z0-9_-]{0,63}$/` | `ref-contract.ts` |
| Env ref ID pattern | `/^[A-Z][A-Z0-9_]{0,127}$/` | `ref-contract.ts` |
| Auth profile ID pattern | `/^[A-Za-z0-9:_-]{1,128}$/` | `ref-contract.ts` |
| Known provider env vars | 26+ | `provider-env-vars.ts` |

### Config

| 常量 | 值 | 位置 |
|------|-----|------|
| Config file | `openclaw.json` | `paths.ts` |
| State dir | `~/.openclaw/` | `paths.ts` |
| Gateway port | 18789 | `paths.ts` |
| OAuth creds | `$STATE_DIR/credentials/oauth.json` | `paths.ts` |
| Include max depth | 10 | `includes.ts` |
| Include max file | 2 MB | `includes.ts` |
| Env var pattern | `/^[A-Z_][A-Z0-9_]*$/` | `env-substitution.ts` |
| Shell env fallback keys | 17 keys | `io.ts` |
| Prototype-blocked keys | `__proto__`, `constructor`, `prototype` | `prototype-keys.ts` |
| Zod schema files | 15+ | `zod-schema*.ts` |
| Type sub-modules | 35 | `types.*.ts` |

---

## 7. C# 概念對照

| OpenClaw (TypeScript) | C# 對應概念 |
|----------------------|-------------|
| `executeSecurityAudit()` collectors | ASP.NET Core `IHealthCheck` + `HealthCheckRegistration` 群組 |
| `wrapExternalContent()` markers | `[AllowHtml]` + `HtmlSanitizer`（但更嚴格的 LLM-specific 防護） |
| `fetchWithSsrFGuard()` | `HttpClient` + `DelegatingHandler` + DNS pinning via `SocketsHttpHandler` |
| `isPrivateIpAddress()` | `IPAddress.IsIPv6LinkLocal` / `IPAddress.IsLoopback` + RFC 1918 check |
| `compileSafeRegex()` | `Regex` constructor + `RegexOptions.None`（C# 無內建 ReDoS 防護） |
| `timingSafeEqual()` | `CryptographicOperations.FixedTimeEquals()` |
| `SsrFPolicy` | `HttpClientHandler.DangerousAcceptAnyServerCertificateValidator` 的更細粒度版 |
| `SecretRef { source, provider, id }` | Azure Key Vault `SecretClient.GetSecret(name, version)` |
| Exec provider（stdin/stdout JSON） | `Process.Start` + `StandardInput`/`StandardOutput` redirect |
| File provider JSON pointer | `System.Text.Json.JsonDocument.SelectElement(pointer)` |
| `resolveSecretRefValues()` batch | `Task.WhenAll` + `SemaphoreSlim` 控制並行 |
| `prepareSecretsRuntimeSnapshot()` | DI container `IServiceCollection.BuildServiceProvider()` 階段 |
| Secret target registry | `[ConfigurationKeyName]` attribute + `IOptions<T>` binding |
| `SecretsApplyPlan` | Entity Framework `Migration` 概念 |
| Zod schema validation | `FluentValidation.AbstractValidator<T>` |
| `OpenClawSchema.safeParse()` | `ModelState.IsValid` + `TryValidateModel()` |
| `.strict()` on schema | `JsonSerializerOptions.UnmappedMemberHandling = Disallow` |
| `deepMerge()` include | `IConfiguration.GetSection().Bind()` 多來源合併 |
| `${VAR}` env substitution | `IConfiguration` 的 `%ENV%` token 或 Azure App Configuration |
| `$include` directive | `IConfigurationBuilder.AddJsonFile()` 多檔案載入 |
| `ConfigFileSnapshot` | `IOptionsSnapshot<T>` + change tracking |
| `applyConfigOverrides()` | `IOptionsMonitor<T>.CurrentValue` runtime 更新 |
| `merge-patch.ts` | JSON Merge Patch（RFC 7386）via `JsonPatchDocument` |
| `parseConfigPath("foo.bar.baz")` | `ConfigurationPath.Combine()` / `GetSection()` |
| Config audit log | `ILogger` + `AuditLogMiddleware` |
| `findLegacyConfigIssues()` | Entity Framework `HasConversion()` migration |
| `normalizeConfigPaths()` | `Path.GetFullPath()` + `Environment.ExpandEnvironmentVariables()` |
| DM policy modes | ASP.NET Core Authorization `[Authorize(Policy = "...")]` |
| Dangerous tools registry | `[Authorize(Roles = "Admin")]` on controller actions |
| `inspectWindowsAcl()` icacls | `System.Security.AccessControl.FileSecurity.GetAccessRules()` |
