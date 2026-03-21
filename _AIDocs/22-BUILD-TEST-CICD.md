# 22 — Build + Test + CI/CD 深入

> Phase 7-2（最終 Session）| 2026-03-14
> 涵蓋：pnpm workspace、tsdown build pipeline、Vitest 測試系統、GitHub Actions CI/CD、Docker、Scripts 工具鏈、Release 流程、Code Quality

---

## 一、pnpm Workspace 結構

### Workspace 定義（`pnpm-workspace.yaml`）

```
packages:
  - .              ← 根包 (openclaw CLI + core)
  - ui             ← Control UI (Lit 3 + Vite)
  - packages/*     ← clawdbot / moltbot 相容 shim
  - extensions/*   ← 33+ channel/feature plugins
```

### 根包重點

| 欄位 | 值 | 備註 |
|------|---|------|
| `name` | `openclaw` | npm 發布名 |
| `version` | `2026.3.7` | 日期版號（YYYY.M.D） |
| `type` | `module` | ESM-first |
| `bin` | `openclaw` → `openclaw.mjs` | CLI 入口 |
| `main` | `dist/index.js` | Library 入口 |
| `engines.node` | `>=22.12.0` | 最低 Node 版本 |
| `packageManager` | `pnpm@10.23.0` | 鎖定 pnpm 版本 |

### Export Map（48 Plugin SDK Entries）

```
.                    → dist/index.js
./plugin-sdk         → dist/plugin-sdk/index.js    (+ .d.ts)
./plugin-sdk/core    → dist/plugin-sdk/core.js
./plugin-sdk/<ch>    → dist/plugin-sdk/<ch>.js      ← 每個 channel 一個
./cli-entry          → openclaw.mjs
```

### 相容 Shim（`packages/`）

| 包 | 用途 |
|----|------|
| `clawdbot` | Legacy 名稱轉發 → `openclaw` (`workspace:*`) |
| `moltbot` | Legacy 名稱轉發 → `openclaw` (`workspace:*`) |

### pnpm 特殊設定

- `minimumReleaseAge: 2880` — 新版套件必須發布 48h 後才可安裝（供應鏈安全）
- `overrides` — 12 個安全/相容性覆蓋（hono, fast-xml-parser, form-data, minimatch, qs, tar 等）
- `onlyBuiltDependencies` — 10 個需要 native build 的套件白名單（sharp, node-pty, esbuild 等）
- patched dependencies 必須使用精確版本（無 `^`/`~`）

---

## 二、Build Pipeline

### 管線總覽（`pnpm build`）

```
pnpm build
  │
  ├─ 1. canvas:a2ui:bundle        ← A2UI 前端 bundle（cached）
  ├─ 2. tsdown-build.mjs          ← TypeScript → JS（所有 entry points）
  ├─ 3. copy-plugin-sdk-root-alias ← 複製 CJS alias
  ├─ 4. build:plugin-sdk:dts      ← tsc 產生 .d.ts
  ├─ 5. write-plugin-sdk-entry-dts ← 產生 facade .d.ts（解決路徑嵌套問題）
  ├─ 6. canvas-a2ui-copy          ← 複製 A2UI 產物到 dist/
  ├─ 7. copy-hook-metadata        ← 複製 HOOK.md 到 dist/bundled/
  ├─ 8. copy-export-html-templates ← 複製 HTML export 模板
  ├─ 9. write-build-info          ← 產生 dist/build-info.json
  ├─ 10. write-cli-startup-metadata ← 產生 channel catalog JSON
  └─ 11. write-cli-compat         ← 產生 legacy daemon-cli shim
```

### tsdown 設定（`tsdown.config.ts`）

**Bundler：** tsdown（基於 Rolldown）
**Platform：** Node.js
**環境變數：** `NODE_ENV: "production"`

**Entry Points 分類：**

| 類別 | Entry | Output |
|------|-------|--------|
| 核心 | `src/index.ts` | `dist/index.js` |
| CLI | `src/entry.ts` | `dist/entry.js` |
| Daemon | `src/cli/daemon-cli.ts` | `dist/daemon-cli.js` |
| Warning Filter | `src/infra/warning-filter.ts` | `dist/warning-filter.js` |
| Lazy Channels (7) | `src/channels/plugins/*.ts` | `dist/channels/plugins/*.js` |
| Plugin SDK (48) | `src/plugin-sdk/*.ts` | `dist/plugin-sdk/*.js` |
| Extension API | `src/extensionAPI.ts` | `dist/extensionAPI.js` |
| Bundled Hooks | `src/hooks/bundled/*/handler.ts` | `dist/bundled/*/handler.js` |

**設定：** `fixedExtension: false`（自動偵測 .js/.mjs）

### TypeScript 設定

| 設定 | 值 | 備註 |
|------|---|------|
| `module` | `NodeNext` | ESM + Node.js 模組解析 |
| `target` | `es2023` | 現代 JS |
| `strict` | `true` | 嚴格型別檢查 |
| `noEmit` | `true` | 由 tsdown 處理產出，tsc 只做檢查 |
| `declaration` | `true` | 產生 .d.ts |
| `allowImportingTsExtensions` | `true` | 允許 `.ts` import |
| `experimentalDecorators` | `true` | TypeScript decorators |
| `lib` | `DOM, ES2023` | DOM + 最新 ES |

**Path Aliases：**
```
openclaw/plugin-sdk    → ./src/plugin-sdk/index.ts
openclaw/plugin-sdk/*  → ./src/plugin-sdk/*.ts
```

**Plugin SDK DTS（`tsconfig.plugin-sdk.dts.json`）：**
- `emitDeclarationOnly: true` → 只產 .d.ts
- `outDir: dist/plugin-sdk`
- 48 個 SDK 模組的 include 清單

### Build Scripts 詳解

| Script | 作用 |
|--------|------|
| `tsdown-build.mjs` | 薄 wrapper，spawn `pnpm exec tsdown --config-loader unrun`，`OPENCLAW_BUILD_VERBOSE` 控制 log level |
| `copy-plugin-sdk-root-alias.mjs` | 複製 `src/plugin-sdk/root-alias.cjs` → `dist/`（jiti Proxy 延遲載入入口） |
| `write-plugin-sdk-entry-dts.ts` | 解決 tsc 輸出嵌套問題：`dist/plugin-sdk/plugin-sdk/*.d.ts` → 產生 facade `dist/plugin-sdk/*.d.ts`（`export * from "./plugin-sdk/<entry>.js"`） |
| `canvas-a2ui-copy.ts` | 複製 A2UI bundle（index.html + a2ui.bundle.js）到 `dist/canvas-host/a2ui/`，Docker 可用 `OPENCLAW_A2UI_SKIP_MISSING=1` 跳過 |
| `copy-hook-metadata.ts` | 複製 `src/hooks/bundled/*/HOOK.md` → `dist/bundled/*/HOOK.md` |
| `copy-export-html-templates.ts` | 複製 `src/auto-reply/reply/export-html/`（template.html/css/js + vendor/）→ `dist/export-html/` |
| `write-build-info.ts` | 產生 `dist/build-info.json`：`{ version, commit, builtAt }` |
| `write-cli-startup-metadata.ts` | 掃描 `extensions/*/package.json` 的 `openclaw.channel` 中繼資料，產生 `dist/cli-startup-metadata.json`（按 order 排序的 channel 清單） |
| `write-cli-compat.ts` | 產生 `dist/cli/daemon-cli.js` legacy shim（解析 bundle 匯出 → re-export，缺失的產生 error-throwing stubs） |

### A2UI Bundle（`scripts/bundle-a2ui.sh`）

**用途：** 構建 Agent-to-UI 前端 bundle

**流程：**
1. 計算 SHA256 hash（inputs: package.json, pnpm-lock.yaml, vendor/a2ui/, apps/shared/）
2. 比對 `.bundle.hash` → 命中快取則跳過
3. tsc 編譯 A2UI renderer
4. rolldown 打包 Canvas app
5. 輸出 `src/canvas-host/a2ui/a2ui.bundle.js` + 更新 `.bundle.hash`
6. Docker 環境 graceful degradation（vendor/apps 缺失 → 保留預建 bundle）

### CLI 入口（`openclaw.mjs`）

**啟動序列：**
1. 驗證 Node.js 版本 ≥ 22.12.0（失敗 → nvm upgrade 提示）
2. `module.enableCompileCache()`（非阻塞）
3. 載入 `dist/warning-filter.js`（壓制 ExperimentalWarning 等）
4. 載入 `dist/entry.js`（主 CLI 邏輯）

### Dev Runner

| Script | 模式 | 特性 |
|--------|------|------|
| `run-node.mjs` | Smart rebuild | `.buildstamp` 追蹤 git HEAD + mtime，6 觸發條件（force/no stamp/no dist/config newer/HEAD changed/dirty tree/source newer），`--no-clean` 增量 rebuild |
| `watch-node.mjs` | File watch | Node.js `--watch` mode，監看 `src/`、`tsconfig.json`、`package.json`，`OPENCLAW_WATCH_SESSION` 追蹤 |

### UI Build（`ui/`）

| 項目 | 值 |
|------|---|
| 框架 | Lit 3 Web Components |
| Build | Vite 7.3.1 |
| 輸出 | `dist/control-ui/` |
| Dev port | 5173 |
| Source map | `true` |
| `pnpm prepack` | `pnpm build && pnpm ui:build` |

---

## 三、Test 系統

### Vitest Config 架構

```
vitest.config.ts              ← Base config（coverage thresholds, SDK aliases）
├── vitest.unit.config.ts     ← Unit/Integration（排除 gateway/channels/extensions）
├── vitest.e2e.config.ts      ← E2E（forks, adaptive workers）
├── vitest.live.config.ts     ← Live provider（serial, real API keys）
├── vitest.gateway.config.ts  ← Gateway 專屬
├── vitest.channels.config.ts ← Channel 專屬（telegram/discord/web/browser/line）
├── vitest.extensions.config.ts ← Extension 專屬
└── vitest.scoped-config.ts   ← Helper factory
```

### Base Config 重點（`vitest.config.ts`）

| 設定 | 值 |
|------|---|
| Pool | `forks`（adaptive: 4-16 local, 2-3 CI） |
| Hook timeout | 180s (Windows) / 120s (others) |
| Test timeout | 120s |
| Setup | `test/setup.ts` |
| Plugin SDK aliases | 55 subpath mappings（src → dist bypass） |

### Coverage Thresholds（V8 provider）

| 指標 | 門檻 |
|------|------|
| **Lines** | 70% |
| **Functions** | 70% |
| **Branches** | 55% |
| **Statements** | 70% |

**Coverage 範圍：** `./src/**/*.ts`
**排除：** extensions/, apps/, ui/, CLI, daemon, hooks, channels, gateway server, tui, browser 等整合表面

### Custom Test Runner（`scripts/test-parallel.mjs`）

**核心功能：** 智慧平行化 + 環境偵測 + 效能最佳化

#### Pool 策略

| 條件 | Pool |
|------|------|
| Node 24+（ERR_VM_MODULE_LINK_FAILURE） | `forks` |
| Windows | `forks` |
| 記憶體 < 64 GiB | `forks` |
| 其他 | `vmForks` |
| `OPENCLAW_TEST_VM_FORKS=0` | 強制 `forks` |
| `OPENCLAW_TEST_VM_FORKS=1` | 強制 `vmForks` |

#### Test Profiles（`OPENCLAW_TEST_PROFILE`）

| Profile | 行為 |
|---------|------|
| `normal`（default） | 分割 `unit-fast` + `unit-isolated` 雙 lane |
| `low` | 單一 unit lane，2 workers |
| `serial` | 全部 serial，每 lane 1 worker |
| `max` | 激進平行化（高記憶體主機） |

#### Isolated Unit Tests（89 files）

高衝突測試分離到 `unit-isolated` lane：
- Filesystem 競爭：`security/audit.test.ts`, `temp-path-guard.test.ts`
- Docker/Archive：`docker-setup.test.ts`, `infra/archive.test.ts`
- Process 監控：`supervisor.test.ts`
- Git hooks：`git-hooks-pre-commit.test.ts`
- Browser：`extension-relay.test.ts`, `server.*.test.ts`
- Telegram/Slack bot：`telegram/bot.test.ts`, `slack/monitor/slash.test.ts`

#### Worker Scaling（Host-Aware）

| 環境 | Unit | Isolated | Gateway | Extensions |
|------|------|----------|---------|------------|
| CI macOS | 1 | — | — | — |
| CI Windows | 2 shards | — | — | — |
| Local ≥96 GiB | 7-14 | 1-2 | 2-6 | 1-4 |
| Local 64-95 GiB | 2-8 | 1 | 1 | 1-4 |
| Local <64 GiB | 2 | 1 | 1 | 4 |

**Load scaling：** load avg ≥1.0 → -15%, ≥1.1 → -25%

#### Sharding

- `OPENCLAW_TEST_SHARDS=<n>` + `OPENCLAW_TEST_SHARD_INDEX=<m>` → shard m/n
- Windows CI 自動 shard 為 2

### Test Suites 分類

```
┌─────────────────────────────────────────────────────────┐
│                 pnpm test（orchestrator）                 │
├────────────┬──────────────┬─────────────┬───────────────┤
│ unit-fast  │ unit-isolated│ extensions  │ gateway       │
│ vmForks/   │ forks        │ vmForks     │ forks, serial │
│ forks      │ 89 files     │ (opt-in)    │ (opt-in)      │
├────────────┴──────────────┴─────────────┴───────────────┤
│ Optional:                                                │
│ • pnpm test:e2e     (vmForks, serial, multi-instance)   │
│ • pnpm test:live    (serial, real providers)             │
│ • pnpm test:docker:* (container sandbox)                │
│ • pnpm test:channels (telegram/discord/web/browser/line)│
│ • pnpm test:ui      (Playwright browser)                │
└─────────────────────────────────────────────────────────┘
```

### Live Test 分層

| 層 | 觸發 | 內容 |
|----|------|------|
| Direct Model | `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_MODELS=modern` | 直接 model completion（無 gateway） |
| Gateway + Agent | `OPENCLAW_LIVE_TEST=1` | Gateway 啟動 + agent smoke（nonce read, exec+read, image） |
| Android Node | `pnpm android:test:integration` | 已配對 Android 裝置 capability 驗證 |
| Setup Token | `OPENCLAW_LIVE_SETUP_TOKEN=1` | Claude Code CLI setup-token auth |
| CLI Backend | `OPENCLAW_LIVE_CLI_BACKEND=1` | 本地 CLI 作為 backend |

### Docker Test Scripts

| Script | 用途 |
|--------|------|
| `test-live-models-docker.sh` | Docker 內直接 model completion |
| `test-live-gateway-models-docker.sh` | Docker 內 gateway + agent smoke |
| `test-install-sh-docker.sh` | 安裝腳本 smoke（root + non-root + CLI） |
| `test-install-sh-e2e-docker.sh` | 完整 E2E install + model 驗證 |
| `test-cleanup-docker.sh` | cleanup 命令 smoke |

### E2E Docker Scripts（`scripts/e2e/`）

| Script | 用途 |
|--------|------|
| `onboard-docker.sh` | Onboarding wizard 5 test cases（local/remote/reset/channels/skills），pseudo-TTY 驅動 |
| `doctor-install-switch-docker.sh` | `doctor --repair` 切換 npm↔git daemon entrypoint |
| `gateway-network-docker.sh` | 多容器 gateway + client WebSocket 連線測試 |
| `plugins-docker.sh` | Plugin load 4 種方式（in-memory/tgz/directory/file: spec） |
| `qr-import-docker.sh` | qrcode-terminal Node 22+ dynamic import 回歸測試 |

### Performance Testing

| Script | 用途 |
|--------|------|
| `test-perf-budget.mjs` | 斷言 wall-clock 時間在預算內（`--max-wall-ms`, `--baseline-wall-ms`, `--max-regression-pct 10%`） |
| `test-hotspots.mjs` | 識別最慢的 N 個測試檔（`--limit 20`） |

### Test 環境隔離

**`test/setup.ts`：**
- 非 live test → 隔離 HOME/state dir
- live test → 載入 `~/.profile`
- 預設 plugin registry（stub Discord/Slack/Telegram/WhatsApp/Signal/iMessage）
- 假 timer 洩漏防護（afterEach cleanup）
- max listeners 128

**`test/test-env.ts`：**
- `withIsolatedTestHome()` → 臨時 tmpdir + 清除所有 `OPENCLAW_*` env vars + 憑證 token

### 關鍵環境變數一覽

| 變數 | 用途 |
|------|------|
| `OPENCLAW_TEST_PROFILE` | 測試 profile（normal/low/serial/max） |
| `OPENCLAW_TEST_VM_FORKS` | 強制 pool（0=forks, 1=vmForks） |
| `OPENCLAW_TEST_WORKERS` | 總 worker 數覆蓋 |
| `OPENCLAW_TEST_WORKERS_UNIT` | Unit lane worker |
| `OPENCLAW_TEST_WORKERS_GATEWAY` | Gateway lane worker |
| `OPENCLAW_TEST_SERIAL_GATEWAY` | Gateway serial（default: 1） |
| `OPENCLAW_TEST_SHARDS` / `_SHARD_INDEX` | 分片 |
| `OPENCLAW_TEST_SHOW_PASSED_LOGS` | 顯示通過測試的 log |
| `OPENCLAW_TEST_MAX_OLD_SPACE_SIZE_MB` | Node heap 覆蓋 |
| `OPENCLAW_LIVE_TEST` | 啟用 live 測試 |
| `OPENCLAW_LIVE_MODELS` | Direct model allowlist（modern/all/list） |
| `OPENCLAW_LIVE_GATEWAY_MODELS` | Gateway model allowlist |

---

## 四、CI/CD（GitHub Actions）

### Workflow 總覽

| Workflow | 觸發 | 用途 |
|----------|------|------|
| `ci.yml` | push main, PR | **主管線**：build + lint + test + multi-platform |
| `docker-release.yml` | push main, tags `v*` | Docker 映像構建 + 多架構 manifest |
| `install-smoke.yml` | push main, PR | 安裝腳本 smoke 測試 |
| `labeler.yml` | PR opened/sync, issues | 自動標籤（size/channel/maintainer） |
| `auto-response.yml` | issues/comments/labels | 自動回應（support/skill/spam/dirty PR） |
| `codeql.yml` | 手動 | 安全分析（JS/TS/Actions/Python/Java/Swift） |
| `stale.yml` | 每日 3:17 UTC | Stale issue/PR 管理（7d stale, 5d close） |
| `sandbox-common-smoke.yml` | Dockerfile.sandbox 變更 | Sandbox 基礎映像 smoke |
| `workflow-sanity.yml` | push main, PR | Workflow 自身檢查（no tabs + actionlint） |

### CI 主管線（`ci.yml`）深入

**Runner Fleet：**
- Linux: `blacksmith-16vcpu-ubuntu-2404`
- Windows: `blacksmith-32vcpu-windows-2025`（6 shards）
- macOS: `macos-latest`
- ARM: `blacksmith-16vcpu-ubuntu-2404-arm`

**Smart Job Skipping：**
1. `docs-scope` → 偵測 docs-only 變更 → 跳過 build/test
2. `changed-scope` → 偵測觸及區域（Node/macOS/Android/Python/Windows）→ 條件跳過

**Jobs 流程：**

```
docs-scope ─┬→ build-artifacts ──→ checks (Node + Bun)
changed-scope ┤                  ├→ check (types + lint + format)
              │                  ├→ check-docs (format + lint + links)
              │                  ├→ skills-python (ruff + pytest)
              │                  ├→ secrets (detect-secrets + zizmor)
              │                  ├→ checks-windows (6 shards)
              │                  ├→ macos (TS tests + Swift lint/build/test)
              │                  ├→ android (Gradle unit + debug build)
              │                  └→ release-check (push to main only)
              └→ ios (disabled)
```

**重要 CI 設定：**
- Heap: `--max-old-space-size=6144`（Windows）
- pnpm store caching: sticky disk + fallback
- Windows Defender exclusions（避免 scan 慢速）
- Concurrency: per-PR cancel in-progress

### Docker Release（`docker-release.yml`）

**Multi-stage Build：**

```
ext-deps (Node 22-bookworm)
  → build (install + build + ui:build)
    → runtime (default 或 slim)
      → 非 root user (node:node)
      → Healthcheck: GET /healthz
      → CMD: node openclaw.mjs gateway --allow-unconfigured
```

**Variant：**
- `default`：node:22-bookworm（完整）
- `slim`：node:22-bookworm-slim

**Build Args：**
- `OPENCLAW_EXTENSIONS`：空格分隔的 extension 名稱（opt-in deps）
- `OPENCLAW_INSTALL_BROWSER=1`：安裝 Chromium/Xvfb（~300MB）
- `OPENCLAW_INSTALL_DOCKER_CLI=1`：安裝 Docker CLI（~50MB）
- `OPENCLAW_DOCKER_APT_PACKAGES`：額外系統套件

**Tagging：**
- main → `main-amd64`, `main-arm64` → `main` manifest
- `v*` tag → 版本特定 tag → `latest`（semantic version）
- Slim → 加 `-slim` suffix

**OCI Labels：** revision, version, created, source, docs, licenses

### Auto-response（`auto-response.yml`）

**自動化規則：**
- `r:skill` → 關閉 + 建議 Clawhub
- `r:support` → 導向 Discord #help
- `r:too-many-prs` → 關閉（作者 >10 open PRs）
- Bug type 自動偵測 → 同步 labels（regression/crash/behavior）
- Spam-ping 偵測 → 警告（>3 maintainer mentions）
- Dirty PR → 關閉（>20 labels 或 "dirty" label）

### Labeler（`labeler.yml`）

**PR Size：**
- XS < 50, S < 200, M < 500, L < 1000, XL ≥ 1000（排除 lockfiles + docs/）

**自動標籤區域：**
- 20+ channel labels（src + extensions + docs 路徑）
- App labels（android/ios/macos/web-ui）
- Area labels（gateway/docs/cli/commands/scripts/docker/agents/security）
- Extension labels（20 個）

---

## 五、Code Quality

### Linting

| 工具 | 設定 | 指令 |
|------|------|------|
| **Oxlint** | `.oxlintrc.json` | `pnpm lint`（`oxlint --type-aware`） |
| **Oxfmt** | `.oxfmtrc.jsonc` | `pnpm format`（`oxfmt --write`） |
| **SwiftLint** | `.swiftlint.yml` | `pnpm lint:swift` |
| **SwiftFormat** | `.swiftformat` | `pnpm format:swift` |
| **ruff** | Python skills | CI `skills-python` job |

**Oxlint 重點規則：**
- Plugins: unicorn, typescript, oxc
- Categories: correctness, perf, suspicious → `error`
- `typescript/no-explicit-any: error`
- `curly: error`
- 排除：assets/, dist/, docs/, extensions/, node_modules/, patches/, skills/, vendor/

**Oxfmt 重點：**
- Tab width: 2, useTabs: false
- `experimentalSortImports: true`
- `experimentalSortPackageJson: true`

**SwiftLint 門檻：**
- file_length: warn 1500, error 2500
- function_body_length: warn 150, error 300
- cyclomatic_complexity: warn 20, error 120
- line_length: warn 120, error 250

**SwiftFormat：** Swift 6.2, indent 4 spaces, maxwidth 120, LF

### Custom Lint Scripts（12+）

| Script | 規則 |
|--------|------|
| `check-channel-agnostic-boundaries.mjs` | 禁止跨 channel 文字洩漏 |
| `check-no-monolithic-plugin-sdk-entry-imports.ts` | 禁止 root `openclaw/plugin-sdk` import |
| `check-no-register-http-handler.mjs` | 禁止原始 HTTP handler 註冊 |
| `check-no-raw-channel-fetch.mjs` | 強制使用 channel fetch wrapper |
| `check-no-raw-window-open.mjs` | 禁止直接 `window.open` |
| `check-no-random-messaging-tmp.mjs` | 禁止 messaging 中使用隨機 temp dir |
| `check-ingress-agent-owner-context.mjs` | 驗證 ingress agent context 擁有權 |
| `check-no-pairing-store-group-auth.mjs` | 禁止 pairing store 用於 group auth |
| `check-pairing-account-scope.mjs` | 驗證 account-scoped pairing |
| `check-webhook-auth-body-order.mjs` | Webhook auth field 順序合規 |
| `check-plugin-sdk-exports.mjs` | 保護 44+ 關鍵 SDK export |
| `check-ts-max-loc.ts` | 單檔 500 行上限（`--max` 可調） |

### Pre-commit Hook（`git-hooks/pre-commit`）

1. 取得 staged files（`git diff --cached`）
2. 過濾 lint/format 目標
3. `oxlint --type-aware --fix`
4. `oxfmt --write`
5. `git add` re-stage

**設定：** `pnpm prepare` → `git config core.hooksPath git-hooks`

### Dead Code Detection

| 工具 | 指令 |
|------|------|
| knip | `pnpm deadcode:knip`（production + isolate workspaces） |
| ts-prune | `pnpm deadcode:ts-prune` |
| ts-unused-exports | `pnpm deadcode:ts-unused` |

---

## 六、Scripts 工具鏈

### Commit Helper（`scripts/committer`）

**用法：** `committer [--force] "msg" file [file ...]`

**安全機制：**
- 拒絕 `.`（防止 stage 全 repo）
- 拒絕 node_modules 路徑
- 驗證 commit message 非空
- git lock retry（最多 5 秒）
- `--force` 可刪除 stale lock

**流程：** unstage all → stage specified → validate → commit

### Release 驗證（`scripts/release-check.ts`）

**檢查項：**
1. Plugin version 同步 — 所有 extensions 版本 = root version
2. Sparkle appcast 驗證 — 數字 `sparkle:version`，lane 單調遞增
3. 關鍵 SDK export 保護 — 23+ 必要 export（issue #27569 回歸防護）
4. npm pack dry-run — 驗證 tarball 包含所有必要路徑，禁止 app bundle
5. Monolithic import 禁止 — plugins 必須用 scoped import

### Plugin Version Sync（`scripts/sync-plugin-versions.ts`）

- 對齊所有 extension package.json version = root version
- 自動 prepend CHANGELOG entry
- 移除 workspace devDependency（`openclaw: workspace:*`）

### Protocol Generation

| Script | 輸出 |
|--------|------|
| `protocol-gen.ts` | `dist/protocol.schema.json`（Gateway Protocol JSON Schema） |
| `protocol-gen-swift.ts` | `apps/macos/Sources/OpenClawProtocol/GatewayModels.swift` + shared |

### macOS Packaging（`scripts/package-mac-app.sh`）

**Pipeline：**
1. pnpm deps → optional JS build
2. Swift build（multi-arch: arm64/x86_64/universal via `lipo`）
3. App bundle 結構（`.app/Contents/MacOS/` + `Frameworks/` + `Resources/`）
4. Info.plist 注入（version, bundle ID, Sparkle config）
5. Framework embedding（Sparkle.framework + Swift 6.2 compat libs）
6. Code signing（`codesign-mac-app.sh`）
7. Output: `dist/OpenClaw.app/`

### Docs 工具

| Script | 用途 |
|--------|------|
| `docs-i18n/` | Go 程式，Claude PI translator + translation memory + glossary |
| `docs-link-audit.mjs` | Markdown 連結驗證（routes, files, anchors, redirects） |
| `docs-spellcheck.sh` | codespell + 自訂字典 |
| `build-docs-list.mjs` | 產生 `bin/docs-list` CLI wrapper |

### 其他工具

| Script | 用途 |
|--------|------|
| `restart-mac.sh` | macOS 完整 dev restart（kill → clean → build → sign → launch） |
| `clawlog.sh` | macOS unified logging（subsystem `ai.openclaw`，category filter, stream/search/export） |
| `ios-configure-signing.sh` | 自動產生 `.local-signing.xcconfig`（Team ID → bundle IDs） |
| `generate-host-env-security-policy-swift.mjs` | JSON security policy → Swift enum transpile |
| `ghsa-patch.mjs` | GitHub Security Advisory programmatic update |

---

## 七、Release 流程

### Release SOP

```
1. Version Bump
   ├─ package.json version
   ├─ apps/android/app/build.gradle.kts (versionName/versionCode)
   ├─ apps/ios/Sources/Info.plist (CFBundleShortVersionString/CFBundleVersion)
   ├─ apps/macos/Sources/.../Info.plist
   └─ docs/install/updating.md (pinned npm version)

2. Plugin Sync
   └─ pnpm plugins:sync → 對齊所有 extension version + CHANGELOG

3. Build + Validate
   ├─ pnpm build
   ├─ pnpm check (format + types + lint)
   ├─ pnpm test + test:e2e
   ├─ pnpm release:check (SDK exports + appcast + npm pack)
   └─ pnpm test:install:smoke

4. npm Publish
   ├─ npm publish --access public --otp="<1password-otp>"
   └─ 驗證: npm view openclaw version

5. macOS App
   ├─ package-mac-app.sh (BUILD_ARCHS="arm64 x86_64")
   ├─ package-mac-dist.sh (notarize + create-dmg)
   ├─ make_appcast.sh (Sparkle ed25519 簽名)
   └─ 驗證: feed URL + enclosure + update flow

6. GitHub Release
   ├─ git tag vYYYY.M.D (or vYYYY.M.D-beta.N)
   ├─ gh release create (CHANGELOG excerpt)
   ├─ 附加: OpenClaw-YYYY.M.D.zip + .dSYM.zip + .dmg
   └─ Commit appcast.xml

7. Docker
   └─ 自動 (docker-release.yml on tag push)
```

### Release Channels

| Channel | Tag | npm dist-tag | 說明 |
|---------|-----|-------------|------|
| `stable` | `vYYYY.M.D` | `latest` | 正式版 |
| `beta` | `vYYYY.M.D-beta.N` | `beta` | 預覽版（可能無 macOS app） |
| `dev` | 無 tag | — | main branch HEAD |

### Release Guards

- `pnpm release:check` — SDK export 保護 + appcast 驗證 + npm pack 驗證
- `pnpm test:install:smoke` — Docker 內安裝腳本測試
- Beta tag 必須用匹配 beta version suffix publish（`YYYY.M.D-beta.N`）
- 版本號變更需操作者明確同意

---

## 八、Docker

### Production Dockerfile（多階段）

| Stage | Base | 用途 |
|-------|------|------|
| `ext-deps` | node:22-bookworm | 收集 extension package.json |
| `build` | node:22-bookworm + Bun | install + build + ui:build |
| `runtime` | node:22-bookworm 或 -slim | 最終映像 |

**Runtime 特性：**
- 系統套件：procps, hostname, curl, git, openssl
- 非 root 執行（node:node）
- Healthcheck: `GET /healthz` every 3 min
- Optional: Chromium/Xvfb, Docker CLI
- Digest pinning（SHA256）

### docker-compose.yml

| Service | 用途 |
|---------|------|
| `openclaw-gateway` | Gateway（port 18789/18790, health check, restart: unless-stopped） |
| `openclaw-cli` | CLI（network_mode: service:openclaw-gateway, cap_drop: NET_RAW/NET_ADMIN, security_opt: no-new-privileges） |

### .dockerignore 重點

排除 .git, apps/（白名單 apps/shared/OpenClawKit/）, vendor/（白名單 vendor/a2ui/）, node_modules, dist, 圖片/影音

---

## 九、Build Pipeline 完整流程圖

```
                    ┌──────────────────────────────┐
                    │       Source Code (src/)       │
                    └──────────────┬───────────────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            │                      │                      │
            ▼                      ▼                      ▼
   ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────┐
   │ bundle-a2ui.sh  │  │ tsdown-build.mjs │  │   tsc (check)   │
   │ (SHA256 cached) │  │ (60+ entries)    │  │ tsgo / tsconfig  │
   └────────┬────────┘  └────────┬─────────┘  └─────────────────┘
            │                    │
            │         ┌──────────┼──────────────┐
            │         │          │              │
            ▼         ▼          ▼              ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐
   │ A2UI     │ │ dist/*.js│ │ plugin-  │ │ bundled/     │
   │ bundle   │ │ (core +  │ │ sdk/     │ │ hook HOOK.md │
   │ copy     │ │ CLI +    │ │ *.js +   │ │ + HTML tmpl  │
   └──────────┘ │ channels)│ │ *.d.ts   │ └──────────────┘
                └──────────┘ └──────────┘
                      │
            ┌─────────┼─────────┐
            ▼         ▼         ▼
   ┌──────────┐ ┌──────────┐ ┌──────────────┐
   │ build-   │ │ cli-     │ │ cli-startup- │
   │ info.json│ │ compat.js│ │ metadata.json│
   └──────────┘ └──────────┘ └──────────────┘
                      │
                      ▼
              ┌──────────────┐
              │  pnpm ui:build│  (Vite → dist/control-ui/)
              └──────────────┘
                      │
                      ▼
              ┌──────────────┐
              │   prepack     │  (build + ui:build → npm publish ready)
              └──────────────┘
```

---

## 十、邊界條件 / 陷阱

1. **A2UI bundle 快取** — SHA256 hash 失效才重建；Docker 環境無 vendor/ → `OPENCLAW_A2UI_SKIP_MISSING=1` 跳過
2. **Plugin SDK .d.ts 路徑嵌套** — tsc 輸出在 `dist/plugin-sdk/plugin-sdk/`，需要 `write-plugin-sdk-entry-dts.ts` 產生 facade
3. **Node 24+ vmForks 回歸** — ERR_VM_MODULE_LINK_FAILURE，test runner 自動 fallback 到 forks
4. **Windows pre-commit** — hook 用 `scripts/pre-commit/run-node-tool.sh` wrapper 確保 PATH 正確
5. **CI macOS OOM** — 強制 1 worker 避免 crash
6. **test-parallel 89 isolated files** — 不斷增長；新增衝突測試需手動加入清單
7. **Sparkle appcast lane 邏輯** — 2026-02-27 之後新 date-key 系統，之前 legacy 單調遞增
8. **npm pack bloat** — `files` 白名單嚴格限制（assets/, dist/, docs/, extensions/, skills/）
9. **Docker digest pinning** — base image 使用 SHA256 digest，更新 Node 版本時需同步更新
10. **pnpm overrides 精確版本** — patched dependencies 禁止 `^`/`~`
11. **CI Windows Defender** — PowerShell 排除 workspace/pnpm store 避免 scan 拖慢
12. **Live test 不穩定** — serial 執行 + 提供者 flaky → 不納入 CI 必要 gate
13. **`committer` script git lock retry** — 多 agent 場景下 git lock 衝突，最多 retry 5s
14. **Plugin version sync 必須在 npm publish 前** — `pnpm plugins:sync` 未執行 → `release-check` 會失敗
15. **Beta publish 必須用 beta suffix** — `npm publish --tag beta` 搭配 plain version → 版號被消耗

---

## 十一、關鍵常量速查

| 常量 | 值 | 位置 |
|------|---|------|
| Node 最低版本 | 22.12.0 | `openclaw.mjs` |
| pnpm 版本 | 10.23.0 | `package.json` packageManager |
| Coverage lines threshold | 70% | `vitest.config.ts` |
| Coverage branches threshold | 55% | `vitest.config.ts` |
| Test hook timeout (Windows) | 180s | `vitest.config.ts` |
| Test hook timeout (others) | 120s | `vitest.config.ts` |
| Test timeout | 120s | `vitest.config.ts` |
| CI heap size | 6144 MB | `ci.yml` |
| Default gateway port | 18789 | 多處 |
| LOC limit per file | 500 | `check-ts-max-loc.ts` |
| SwiftLint file_length warn | 1500 | `.swiftlint.yml` |
| pnpm minimumReleaseAge | 2880 min (48h) | `package.json` |
| Docker healthcheck interval | 3 min | `Dockerfile` |
| Stale issue days | 7 stale + 5 close | `stale.yml` |
| Stale PR days | 5 stale + 3 close | `stale.yml` |
| PR size XS/S/M/L/XL | 50/200/500/1000 | `labeler.yml` |
| Plugin SDK entries | 48 | `tsdown.config.ts` |
| Isolated test files | 89 | `test-parallel.mjs` |

---

## 十二、C# / .NET 概念對照

| OpenClaw 概念 | C# / .NET 對應 |
|---------------|---------------|
| pnpm workspace | NuGet solution / multi-project .sln |
| tsdown bundler | MSBuild / dotnet publish 單檔發佈 |
| tsconfig.json strict | `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` |
| Vitest + V8 coverage | xUnit/NUnit + coverlet + ReportGenerator |
| vitest.config.ts pool=forks | `[assembly: CollectionBehavior(DisableTestParallelization = false)]` |
| GitHub Actions CI | Azure DevOps Pipelines / TeamCity |
| Oxlint + Oxfmt | Roslyn Analyzers + dotnet format |
| pre-commit hook | Husky.Net / pre-commit hooks |
| Docker multi-stage | Docker multi-stage（相同） |
| npm publish | `dotnet nuget push` |
| Sparkle auto-update | Squirrel / MSIX auto-update |
| pnpm overrides | `<PackageVersion>` in Directory.Packages.props (CPM) |
| knip dead code | JetBrains ReSharper Solution-Wide Analysis |
| `scripts/committer` | `git commit` wrapper + staged file control |
| `release-check.ts` | `dotnet pack --no-build` + NuGet metadata 驗證 |
| `.buildstamp` incremental | MSBuild incremental build（inputs/outputs） |

---

## 十三、常用指令速查

```bash
# Build
pnpm build                    # 完整 build（11 步）
pnpm build:strict-smoke       # 快速 smoke（前 4 步）
pnpm tsgo                     # TypeScript 型別檢查

# Dev
pnpm dev                      # Smart rebuild + run
pnpm gateway:watch            # Watch + rebuild + restart gateway
pnpm ui:dev                   # Vite dev server (:5173)

# Test
pnpm test                     # 主測試（adaptive parallelism）
pnpm test:coverage            # Unit + coverage report
pnpm test:e2e                 # E2E smoke
pnpm test:live                # Live provider（需 API keys）
pnpm test:docker:all          # 所有 Docker 測試
OPENCLAW_TEST_PROFILE=low pnpm test  # 低記憶體模式

# Lint / Format
pnpm check                    # format + types + lint + custom checks
pnpm lint:fix                 # Auto-fix lint + format
pnpm format                   # oxfmt --write

# Release
pnpm release:check            # Pre-release 驗證
pnpm plugins:sync             # 同步 extension versions
pnpm test:install:smoke       # 安裝腳本 smoke

# Docs
pnpm check:docs               # format + lint + link check
pnpm docs:dev                 # Mintlify dev server

# Dead Code
pnpm deadcode:knip            # knip production scan
pnpm deadcode:report          # 三工具完整掃描

# Protocol
pnpm protocol:check           # 產生 + diff 驗證
```
