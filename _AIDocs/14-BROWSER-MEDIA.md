# 14 — Browser + Media Understanding + Media Pipeline 深入

> Phase 4-1 | 掃描日期：2026-03-13
> 涵蓋：`src/browser/` 145 files ~23.5K LOC + `src/media-understanding/` 65 files ~6.9K LOC + `src/media/` 39 files ~4.5K LOC
> 總計：249 files, ~34.9K LOC

---

## 目錄

1. [Browser 自動化框架](#1-browser-自動化框架)
2. [Media Understanding 多媒體理解](#2-media-understanding-多媒體理解)
3. [Media Pipeline 媒體管線](#3-media-pipeline-媒體管線)
4. [三模組整合關係](#4-三模組整合關係)
5. [邊界條件與陷阱](#5-邊界條件與陷阱)
6. [關鍵常量速查](#6-關鍵常量速查)
7. [C# 概念對照](#7-c-概念對照)

---

## 1. Browser 自動化框架

### 1.1 技術選型

| 項目 | 選擇 |
|------|------|
| 引擎 | **Playwright-core** + CDP (Chrome DevTools Protocol) |
| HTTP 框架 | Express.js |
| 瀏覽器 | Chrome（平台特定 executable 搜尋） |
| 驅動模式 | `"openclaw"`（自行啟動 Chrome）或 `"extension"`（透過瀏覽器擴充轉發） |

### 1.2 架構分層

```
┌─────────────────────────────────────────────────────────┐
│  Client SDK  (client.ts / client-actions-*.ts)          │
│  REST 呼叫封裝 + 型別定義                                │
├─────────────────────────────────────────────────────────┤
│  HTTP Routes Layer  (routes/*.ts)                       │
│  /status, /tabs/*, /snapshot/*, /act/*, /storage/*      │
├─────────────────────────────────────────────────────────┤
│  Server Context  (server-context*.ts)                   │
│  Profile 管理 + Tab 操作 + Availability 管理             │
├─────────────────────────────────────────────────────────┤
│  Playwright Core  (pw-tools-core*.ts)                   │
│  Snapshot / Interaction / State / Downloads              │
├─────────────────────────────────────────────────────────┤
│  CDP Layer  (cdp.ts / cdp.helpers.ts)                   │
│  WebSocket 通訊 + Target 管理 + Accessibility Tree       │
├─────────────────────────────────────────────────────────┤
│  Chrome Lifecycle  (chrome.ts / chrome.executables.ts)   │
│  Process spawn / kill / user-data-dir / port allocation  │
└─────────────────────────────────────────────────────────┘
```

### 1.3 Server 架構

**主 Server** (`server.ts`, 122 lines)：
- Express app + auth middleware（token/password）
- 延遲載入 Playwright（`isPwAiLoaded()` 追蹤）
- 路由群組：basic（status）、tabs（list/open/close）、agent（snapshot/act/debug/storage）
- Port 透過 `resolveBrowserConfig()` 設定

**Bridge Server** (`bridge-server.ts`, 147 lines)：
- Loopback-only（127.0.0.1）沙盒隔離環境用
- Per-port auth 隔離
- noVNC observer HTML bootstrap

### 1.4 HTTP Route 完整列表

| Route | Method | 功能 |
|-------|--------|------|
| `/status` | GET | Browser 狀態（enabled/running/cdpPort/pid） |
| `/profiles` | GET | Profile 清單（name/cdpPort/tabCount/isDefault） |
| `/tabs` | GET | 列出所有 tab |
| `/tabs/open` | POST | 開新 tab → BrowserTab |
| `/tabs/close` | POST | 關閉 tab |
| `/tabs/focus` | POST | 聚焦 tab |
| `/navigate` | POST | 導航 + 解析 renderer swap 後的 targetId |
| `/screenshot` | POST | 截圖（PNG/JPEG，自動縮放） |
| `/snapshot/aria` | POST | Accessibility tree |
| `/snapshot/ai` | POST | AI 優化 DOM snapshot + role refs |
| `/snapshot/role` | POST | Role-based 元素參照 |
| `/act` | POST | 統一操作：click/type/press/hover/drag/select/fill/resize/wait/evaluate/close |
| `/download` | POST | 下載檔案 |
| `/wait/download` | POST | 等待下載完成 |
| `/hooks/dialog` | POST | 攔截對話框（accept/prompt） |
| `/hooks/file-chooser` | POST | 攔截檔案選擇器 |
| `/storage/get` | POST | 取得 cookies/localStorage/sessionStorage |
| `/storage/set` | POST | 設定 storage |
| `/storage/clear` | POST | 清除 storage |
| `/debug/console` | GET | Console 訊息（含 level 過濾） |
| `/debug/network` | GET | Network 請求（含 URL 過濾） |
| `/debug/errors` | GET | Page 錯誤 |

### 1.5 Page Interaction Pipeline

**Snapshot 請求流程：**
```
POST /snapshot/ai
  → readBody() → resolveTargetIdFromBody()
  → getPageViaPlaywright(cdpUrl, targetId)
    → Playwright.chromium.connectOverCDP(cdpWsUrl)
  → page._snapshotForAI()  [Playwright 內部 API]
  → buildRoleSnapshotFromAiSnapshot()
  → storeRoleRefsForTarget()  [cache in pageState WeakMap]
  → normalize screenshot size + format
  → JSON response with refs
```

**Action 請求流程：**
```
POST /act {kind: "click", ref: "e1", targetId}
  → resolveTargetIdFromBody()
  → getRestoredPageForTarget()
    → restoreRoleRefsForTarget()  [from cache]
  → refLocator(page, "e1") → Playwright Locator
  → locator.click({timeout, button, modifiers})
  → JSON response {ok: true, targetId, url}
```

**核心互動函式** (`pw-tools-core.interactions.ts`)：
- `clickViaPlaywright()` — DOMElement.click() + button/modifiers/timeout
- `typeViaPlaywright()` — fill + type + validation
- `hoverViaPlaywright()` — hover + delay
- `dragViaPlaywright()` — source→target locators

### 1.6 Profile 系統

- **ResolvedBrowserProfile**：name / cdpPort / cdpUrl / cdpHost / color / driver
- **Port 分配**：`allocateCdpPort()` — 在 cdpPortRange 內找可用 port
- **User Data Dir**：`resolveOpenClawUserDataDir(profileName)` — 每 profile 獨立 Chrome 資料目錄
- **Profile 裝飾**：`chrome.profile-decoration.ts` — 注入 extensions、prefs、policies
- **CRUD**：`profiles-service.ts` — create/delete/list + port/color 自動分配

**Profile 可用性生命週期：**
```
ensureBrowserAvailable()
  → profileState.running == null?
    → launchOpenClawChrome(profile, cdpPort, userDataDir)
      → spawn process with decoration
      → waitForCdpReady()
    → profileState.running = {pid, exe, cdpPort, ...}
  → isReachable()  [HTTP + WS 可達性檢查]
```

### 1.7 Chrome 生命週期

**chrome.ts** (11.7 KB)：
- `RunningChrome` = {pid, exe, userDataDir, cdpPort, startedAt, proc}
- `launchOpenClawChrome()` — spawn with profile decoration + port allocation
- `stopOpenClawChrome()` — graceful shutdown with exit probes

**chrome.executables.ts** — 平台搜尋：
- Windows：Registry lookup → Program Files
- macOS：/Applications → /opt/homebrew/Caskroom
- Linux：/usr/bin → /snap/bin

### 1.8 Extension Relay 系統

**extension-relay.ts** (80 lines)：
- WebSocket server on loopback
- 轉發 CDP commands：Extension ↔ control-server
- Ping/pong keepalive
- Target attach/detach 追蹤
- Per-connection token 驗證

### 1.9 安全邊界

| 層級 | 控制 | 模組 |
|------|------|------|
| 網路 | Loopback-only binding | `server.ts` / `bridge-server.ts` |
| 認證 | Fail-closed（無 token 不啟動） | `control-auth.ts` |
| SSRF | URL protocol + private network 驗證 | `navigation-guard.ts` |
| 路徑 | `writeViaSiblingTempPath()` 根目錄驗證 | `output-atomic.ts` |
| 隔離 | Per-profile Chrome process（獨立 user-data-dir） | `chrome.ts` |
| Relay | Per-connection token 驗證 | `extension-relay-auth.ts` |

### 1.10 資源管理

| 資源 | 策略 |
|------|------|
| Page state | WeakMap — GC 自動回收 |
| Console/Error/Network | Circular buffer（500/200/500） |
| Role ref cache | LRU max 50 per target |
| Playwright connection | Single cached browser per CDP URL |
| Screenshot | Adaptive JPEG quality（≤5MB） |
| Download | Atomic write to media dir |

---

## 2. Media Understanding 多媒體理解

### 2.1 模組定位

**統一抽象層**：處理多媒體附件（圖片/音訊/影片/文件），路由到合適的 AI provider API 或本地 CLI，回傳結構化文字供 Agent 使用。

### 2.2 三大 Capability

| Capability | Kind | 預設 Provider 優先序 | 預設上限 |
|-----------|------|---------------------|---------|
| Image | `image.description` | OpenAI → Anthropic → Google → Minimax → Zai | 10MB / 500 chars |
| Audio | `audio.transcription` | OpenAI → Groq → Deepgram | 20MB / 無限 |
| Video | `video.description` | Google → Moonshot | 50MB / 500 chars |

### 2.3 主流程

```
applyMediaUnderstanding()                    [apply.ts:466-580]
  ├── normalizeAttachments(ctx)              — 從 MsgContext 解析附件
  ├── MediaAttachmentCache                   — 延遲載入 + 快取 buffer
  ├── 並行執行 3 capability：
  │   └── runCapability(kind)                [runner.ts:659-805]
  │       ├── resolveScopeDecision()         — scope allow/deny 規則
  │       ├── selectAttachments()            — prefer: first|last|path|url
  │       ├── resolveModelEntries()          — 蒐集可用 model
  │       └── runAttachmentEntries()         — 依序嘗試 provider 直到成功
  │           └── runProviderEntry() / runCliEntry()
  ├── formatMediaUnderstandingBody()         — 包裝 [Audio]/[Image]/[Video] 區段
  ├── extractFileBlocks()                    — PDF/CSV/JSON 等文件提取
  └── return ApplyMediaUnderstandingResult
```

### 2.4 10 個 Provider

| Provider | Module | 支援 Capability | 後端 |
|----------|--------|----------------|------|
| openai | `providers/openai/` | image, audio | gpt-4o-mini-transcribe |
| anthropic | `providers/anthropic/` | image | Claude API |
| google | `providers/google/` | image, audio, video | Gemini API |
| deepgram | `providers/deepgram/` | audio | Deepgram API |
| groq | `providers/groq/` | audio | OpenAI-compatible |
| mistral | `providers/mistral/` | audio | OpenAI-compatible |
| minimax | `providers/minimax/` | image | VLM API |
| minimax-portal | `providers/minimax/` | image | VLM portal |
| moonshot | `providers/moonshot/` | image, video | Kimi API |
| zai | `providers/zai/` | image | GLM API |

### 2.5 Model 解析策略

`resolveKeyEntry()` (`runner.ts:340-422`)：

1. 檢查 context 中 active model
2. 嘗試 AUTO_IMAGE_KEY_PROVIDERS 順序：openai → anthropic → google → minimax → zai
3. Fallback 到 `agents.defaults.imageModel`
4. 檢查 Gemini CLI
5. 使用 config 中 configured models

**預設 Model** (`defaults.ts:53-60`)：
- Image：gpt-5-mini / claude-opus-4-6 / gemini-3-flash
- Audio：gpt-4o-mini-transcribe / whisper-large-v3 / nova-3
- Video：gemini-3-flash / Kimi

### 2.6 Attachment 管理

**normalize** (`attachments.normalize.ts:21-72`)：
```typescript
// 從 MsgContext 提取
ctx.MediaPath / ctx.MediaPaths    // 本地路徑
ctx.MediaUrl / ctx.MediaUrls      // 遠端 URL
ctx.MediaType / ctx.MediaTypes    // MIME hint
```

**select** (`attachments.select.ts:58-89`)：
- `prefer: "first" | "last" | "path" | "url"` — 偏好來源
- `mode: "first" | "all"` + `maxAttachments` — 數量控制

**cache** (`attachments.cache.ts`)：
- 延遲載入 buffer（filesystem/URL）
- 避免重複 fetch
- 完成後 cleanup

### 2.7 Scope 存取控制

```typescript
scope: {
  rules: [
    { match: { channel?: string, chatType?: string }, action: "allow" | "deny" }
  ],
  default: "allow" | "deny"
}
```

可依 channel/chatType 粒度開關特定 capability。

### 2.8 File Extraction Pipeline

`extractFileBlocks()` (`apply.ts:335-464`)：

1. 偵測 MIME type（副檔名 → heuristic → raw MIME）
2. UTF-8/UTF-16/CP1252 文字偵測
3. 允許列表驗證
4. 呼叫 `extractFileContentFromSource()` (來自 `../media/input-files.js`)
5. 輸出 XML `<file name="..." mime="...">...</file>` 區塊

支援格式：CSV, TSV, JSON, YAML, XML, Markdown, plain text, PDF（含 image fallback）

### 2.9 Decision 追蹤

```typescript
ctx.MediaUnderstandingDecisions[]  // 每 capability 的決策記錄
ctx.MediaUnderstanding[]           // 輸出文字陣列
ctx.Transcript                     // 音訊轉錄結果
ctx.Body                           // 更新後的 body（含 media + file blocks）
```

### 2.10 錯誤處理

`MediaUnderstandingSkipError` 原因：
- `maxBytes` — 檔案太大
- `timeout` — 請求超時
- `unsupported` — capability 不支援
- `empty` — 無輸出
- `tooSmall` — 音訊 < 1KB

Provider 失敗 → fallthrough 到下一個；全部失敗才 skip。

---

## 3. Media Pipeline 媒體管線

### 3.1 模組定位

**媒體底層管線**：安全的多頻道媒體管理——inbound 擷取、outbound 投遞、格式轉換、大小限制、安全儲存。

### 3.2 核心資料流

```
INBOUND:
┌──────────────────────────────────────────────┐
│ Discord / Telegram / iMessage / API          │
└─────────────────────┬────────────────────────┘
                      ↓
              input-files.ts
              ├─ Base64 decode (base64.ts)
              ├─ URL fetch (fetchWithGuard → SSRF 防護)
              ├─ MIME 偵測 (mime.ts: magic bytes + header + ext)
              ├─ 格式轉換 (image-ops.ts: HEIC→JPEG)
              ├─ PDF 提取 (pdf-extract.ts)
              └─ 大小驗證 (readResponseWithLimit)
                      ↓
              store.ts → saveMediaBuffer()
              ├─ mkdir mediaDir (0o700)
              ├─ 檔名清洗 → {original}---{uuid}.{ext}
              ├─ Write mode 0o644
              └─ return SavedMedia {id, path, size, contentType}

SERVING:
  GET /media/:id  (server.ts)
  ├─ ID 驗證（alphanumeric + ._- only, ≤200 chars）
  ├─ 讀取（max 5MB）
  ├─ TTL 檢查（預設 2 分鐘）→ 過期返回 410 Gone
  ├─ MIME sniff → Content-Type
  ├─ 回傳 + X-Content-Type-Options: nosniff
  └─ Single-use cleanup（50ms delay 後刪除）

OUTBOUND:
  Agent output → parse.ts → splitMediaFromOutput()
  ├─ Regex 提取 MEDIA: tokens（跳過 code fences）
  ├─ 解析 [[audio_as_voice]] tag
  └─ outbound-attachment.ts → resolveOutboundAttachmentFromUrl()
```

### 3.3 大小限制

| Media Kind | 上限 |
|-----------|------|
| Image | 6 MB |
| Audio | 16 MB |
| Video | 16 MB |
| Document | 100 MB |
| Serving (GET) | 5 MB |

### 3.4 MIME 偵測 (`mime.ts`, 192 lines)

**偵測優先序：**
1. `file-type` library（magic bytes sniffing）
2. 副檔名對照表 `EXT_BY_MIME`
3. Content-Type header
4. Generic fallback（zip / octet-stream）

**Telegram Voice 格式：**
```typescript
TELEGRAM_VOICE_MIME_TYPES = {
  "audio/ogg", "audio/opus", "audio/mpeg", "audio/mp3",
  "audio/mp4", "audio/x-m4a", "audio/m4a"
}
```

### 3.5 Image 處理 (`image-ops.ts`, 482 lines)

| 功能 | 函式 |
|------|------|
| HEIC→JPEG | `convertHeicToJpeg()` — Sharp 或 macOS sips |
| Metadata | `getImageMetadata()` — width/height |
| Resize | `resizeImage(buffer, maxSide, quality)` |
| EXIF | `readJpegExifOrientation()` — 手動解析 TIFF byte order |
| Resize Grid | 品質階梯 [85,75,65,55,45,35] × 尺寸階梯 [1800→800] |

**Backend 選擇**：
- 預設：Sharp（Node.js）
- macOS/Bun：`OPENCLAW_IMAGE_BACKEND=sips`

### 3.6 PDF 提取 (`pdf-extract.ts`, 104 lines)

1. `pdfjs-dist` 載入 PDF
2. 提取前 N 頁文字（預設 4 頁）
3. 文字 < 200 chars → fallback 到 canvas 渲染為圖片
4. 回傳 `{ text, images: PdfExtractedImage[] }`

限制：maxPages=4 / maxPixels=4M / minTextChars=200

### 3.7 Base64 安全 (`base64.ts`, 52 lines)

- `estimateBase64DecodedBytes()` — **不分配記憶體**估算解碼大小
- 單次掃描：追蹤字元數 + padding，避免大型 payload 的 OOM
- `canonicalizeBase64()` — 正規化（去空白、驗證字元）

### 3.8 FFmpeg 整合 (`ffmpeg-exec.ts`, 63 lines)

| 項目 | 限制 |
|------|------|
| Buffer | 10 MB |
| ffmpeg timeout | 45s |
| ffprobe timeout | 10s |
| 最大音訊長度 | 20 分鐘 |

### 3.9 Inbound Path Policy (`inbound-path-policy.ts`, 150 lines)

頻道特定的路徑沙盒：
- iMessage：`/Users/*/Library/Messages/Attachments`
- Per-account config：`cfg.channels.imessage.accounts[accountId].attachmentRoots`
- Wildcard pattern matching per segment

### 3.10 MEDIA Token 格式 (`parse.ts`, 261 lines)

```
MEDIA: https://example.com/file.jpg
MEDIA: /path/to/local/file.png
MEDIA: `https://... with spaces ...`
MEDIA: url1 url2 url3
[[audio_as_voice]]
```

- 跳過 fenced code block 內的 MEDIA token
- 支援 quoted paths（處理空格）
- 支援 file:// URL + Windows drive paths

### 3.11 儲存結構

```
$HOME/.config/openclaw/state/media/
├── {uuid}.{ext}                        # 隨機上傳
├── inbound/
│   └── {original}---{uuid}.{ext}       # 使用者上傳
├── outbound/
│   └── {original}---{uuid}.{ext}       # 頻道附件
└── [subdirs]/
```

### 3.12 安全邊界

| 層級 | 控制 | 模組 |
|------|------|------|
| Inbound Path | 頻道特定目錄沙盒 | `inbound-path-policy.ts` |
| SSRF | fetchWithSsrfGuard | `fetch.ts` / `input-files.ts` |
| Size | Pre-allocation check + stream abort | `base64.ts` / `store.ts` |
| ID | Alphanumeric + ._- only, 無 traversal | `server.ts` |
| TTL | 2 分鐘預設 + serve 時檢查 | `server.ts` / `store.ts` |
| 權限 | Dir 0o700 / File 0o644 | `store.ts` |

---

## 4. 三模組整合關係

```
┌────────────────────────────────────────────────────────────────┐
│                     Agent Engine                               │
│  (tools 呼叫 browser / media understanding 作為 capability)     │
└──────────┬──────────────────────────────┬──────────────────────┘
           │                              │
           ▼                              ▼
┌──────────────────────┐    ┌──────────────────────────────────┐
│   Browser Module     │    │   Media Understanding Module     │
│   (src/browser/)     │    │   (src/media-understanding/)     │
│                      │    │                                  │
│  • 截圖 → 存入       │    │  • 附件分析（image/audio/video） │
│    media store       │    │  • Provider 路由 + fallback      │
│  • 下載 → 存入       │    │  • 呼叫 extractFileContent       │
│    media dir         │    │    (from media module)            │
└──────────┬───────────┘    └──────────────┬───────────────────┘
           │                               │
           ▼                               ▼
┌──────────────────────────────────────────────────────────────┐
│                      Media Module                            │
│                      (src/media/)                            │
│                                                              │
│  • 儲存 + 清理（TTL-based）                                   │
│  • MIME 偵測 + 格式轉換（HEIC→JPEG / PDF 提取）              │
│  • Inbound/Outbound pipeline                                 │
│  • HTTP serving（GET /media/:id）                            │
│  • Base64 安全 + SSRF 防護                                   │
└──────────────────────────────────────────────────────────────┘
```

**關鍵整合點：**

| 呼叫方 | 被呼叫 | 介面 |
|--------|--------|------|
| Agent Engine | Browser | REST API（HTTP client） |
| Agent Engine | Media Understanding | `applyMediaUnderstanding()` |
| Media Understanding | Media | `extractFileContentFromSource()` |
| Media Understanding | Provider APIs | HTTP（OpenAI/Anthropic/Google/Deepgram 等） |
| Browser | Media store | Screenshot/download 存入 media dir |
| Channel（Discord/Telegram/...） | Media | `splitMediaFromOutput()` + `resolveOutboundAttachmentFromUrl()` |
| Channel | Media Understanding | 附件觸發 `applyMediaUnderstanding()` |

---

## 5. 邊界條件與陷阱

### Browser

1. **Playwright 延遲載入** — `isPwAiLoaded()` 追蹤，未載入時 snapshot/act 會 fail
2. **Role ref cache invalidation** — 頁面導航後 targetId 可能變（renderer swap），需 `resolveTargetIdFromBody()` 重新解析
3. **CDP WebSocket 連線** — Single cached per CDP URL，若 Chrome crash 需重新建立
4. **SSRF 導航防護** — 不只檢查初始 URL，也檢查導航後的最終 URL（post-navigation guard）
5. **Extension relay auth** — Per-port token 隔離，port 重用可能導致 stale token
6. **Screenshot 過大** — Adaptive JPEG quality reduction（max 2000px side, 5MB），可能失真
7. **evaluate 預設關閉** — `evaluateEnabled` 是安全閘門，需明確開啟
8. **headless 模式差異** — Container 環境需 `noSandbox`，否則 Chrome 無法啟動
9. **timeout 限制** — Action timeout clamp 500ms-60000ms，長操作可能超時
10. **Console/Error buffer 固定大小** — 500/200 條，超過會丟失早期記錄

### Media Understanding

11. **Provider fallthrough** — 依序嘗試，全部失敗才 skip；單一 provider 超時會拖慢整體
12. **Audio < 1KB 直接 skip** — `MIN_AUDIO_FILE_BYTES = 1024`，短音效被忽略
13. **Image output 截斷** — 預設 500 chars，複雜圖片描述可能被截短
14. **Scope 規則順序** — rules 陣列 first-match，順序影響最終 allow/deny
15. **Base64 video 70MB 上限** — `estimateBase64Size(bytes)` 4/3 ratio，50MB 原檔可能超限

### Media

16. **TTL 2 分鐘** — 預設 serve 後 50ms 刪除，agent 重試可能找不到檔案
17. **Single-use cleanup** — 檔案只 serve 一次就刪，多個消費者會衝突
18. **HEIC 轉換依賴** — Sharp 或 sips，Windows 環境 Sharp 需 native binding
19. **PDF text fallback** — < 200 chars 觸發 canvas 渲染，`@napi-rs/canvas` lazy load 可能失敗
20. **Filename sanitization** — `{original}---{uuid}.{ext}` 格式，original 含特殊字元會被清洗

---

## 6. 關鍵常量速查

### Browser

| 常量 | 值 | 位置 |
|------|-----|------|
| `DEFAULT_OPENCLAW_BROWSER_ENABLED` | configurable | `constants.ts` |
| `DEFAULT_BROWSER_EVALUATE_ENABLED` | false | `constants.ts` |
| Console buffer limit | 500 | `pw-tools-core.state.ts` |
| Error buffer limit | 200 | `pw-tools-core.state.ts` |
| Network buffer limit | 500 | `pw-tools-core.state.ts` |
| Role ref cache max | 50 per target | `pw-tools-core.snapshot.ts` |
| Screenshot max side | 2000px | `screenshot.ts` |
| Screenshot max size | 5MB | `screenshot.ts` |
| Action timeout range | 500-60000ms | `pw-tools-core.shared.ts` |
| CDP bootstrap timeout | configurable | `cdp-timeouts.ts` |

### Media Understanding

| 常量 | 值 | 位置 |
|------|-----|------|
| Image max size | 10 MB | `defaults.ts` |
| Audio max size | 20 MB | `defaults.ts` |
| Video max size | 50 MB | `defaults.ts` |
| Image/Video output max chars | 500 | `defaults.ts` |
| MIN_AUDIO_FILE_BYTES | 1024 | `audio-preflight.ts` |
| Default concurrency | 2 | `resolve.ts` |
| Provider timeout | 60s (image/audio) / 120s (video) | `defaults.ts` |
| Max base64 video | ~70 MB | `video.ts` |

### Media

| 常量 | 值 | 位置 |
|------|-----|------|
| Image limit | 6 MB | `constants.ts` |
| Audio limit | 16 MB | `constants.ts` |
| Video limit | 16 MB | `constants.ts` |
| Document limit | 100 MB | `constants.ts` |
| MEDIA_MAX_BYTES (serving) | 5 MB | `store.ts` |
| Default TTL | 2 min | `server.ts` |
| Media server port | 42873 | `server.ts` |
| PDF maxPages | 4 | `pdf-extract.ts` |
| PDF maxPixels | 4M | `pdf-extract.ts` |
| PDF minTextChars | 200 | `pdf-extract.ts` |
| FFmpeg buffer | 10 MB | `ffmpeg-exec.ts` |
| FFmpeg timeout | 45s | `ffmpeg-exec.ts` |
| FFprobe timeout | 10s | `ffmpeg-exec.ts` |
| Max audio duration | 20 min | `ffmpeg-exec.ts` |
| JPEG quality steps | [85,75,65,55,45,35] | `image-ops.ts` |
| Resize side steps | [1800,1600,1400,1200,1000,800] | `image-ops.ts` |

---

## 7. C# 概念對照

| OpenClaw (TypeScript) | C# 對應概念 |
|----------------------|-------------|
| Express routes (`routes/*.ts`) | ASP.NET Controller + [Route] attribute |
| `refLocator(page, ref)` | Selenium `By.CssSelector` / `FindElement` |
| Playwright `page._snapshotForAI()` | 類似 WebDriver Accessibility API |
| CDP WebSocket | SignalR / raw WebSocket client |
| `MediaAttachmentCache` (lazy load) | `Lazy<T>` + `ConcurrentDictionary` |
| `runWithConcurrency()` | `SemaphoreSlim` + `Task.WhenAll` |
| Provider registry pattern | `IServiceCollection.AddScoped<IProvider>()` |
| `MediaUnderstandingSkipError` | Custom Exception + error code enum |
| `scope.rules` match | Policy-based authorization (ASP.NET Core) |
| `saveMediaBuffer()` atomic write | `FileStream` + `File.Move(overwrite: true)` |
| `detectMime()` magic bytes | `System.IO.FileStream.Read` header sniff |
| `cleanOldMedia()` TTL | `BackgroundService` + `Timer` 定期清理 |
| `splitMediaFromOutput()` regex | `Regex.Matches` + `StringBuilder` |
| `fetchWithSsrfGuard()` | `HttpClient` + `DelegatingHandler` 攔截 |
| Browser Profile system | 類似 Selenium Grid Node 概念 |
| `WeakMap` page state | `ConditionalWeakTable<TKey, TValue>` |
| `convertHeicToJpeg()` Sharp | `System.Drawing` / ImageSharp / SkiaSharp |
