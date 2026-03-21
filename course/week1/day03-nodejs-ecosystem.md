# Day 3：Node.js 生態系 + 專案建置

> **目標**：理解 Node.js 的執行模型、套件管理、專案設定，把 OpenClaw 跑起來
> **預計時間**：4 小時

---

## 3.1 Node.js Event Loop（vs C# ThreadPool）

這是 C# 開發者最需要轉換的心智模型。

### C# 的世界觀

```
C# ThreadPool:
┌──────────────────────────────┐
│  Thread 1: 處理 Request A    │
│  Thread 2: 處理 Request B    │  ← 多條線同時跑
│  Thread 3: 處理 Request C    │
│  Thread 4: 等待 I/O...       │
└──────────────────────────────┘
→ 需要 lock、Mutex、SemaphoreSlim 來防 race condition
```

### Node.js 的世界觀

```
Node.js Event Loop:
┌──────────────────────────────────────────┐
│  單一 Thread 輪流處理：                    │
│                                          │
│  1. 處理 Request A（執行 JS code）        │
│  2. A 要讀檔案 → 交給 OS，繼續下一個      │
│  3. 處理 Request B（執行 JS code）        │
│  4. B 要打 API → 交給 OS，繼續下一個      │
│  5. A 的檔案讀完了 → callback 排入佇列    │
│  6. 執行 A 的 callback                   │
│  ...                                     │
└──────────────────────────────────────────┘
→ 不需要 lock！因為同一時間只有一段 code 在跑
```

### 對你閱讀 OpenClaw 的影響

```typescript
// OpenClaw 的 code 裡你不會看到：
// - lock / Mutex / Semaphore
// - Thread.Start() / Task.Run()
// - ConcurrentDictionary / Interlocked

// 你會看到：
await someAsyncOperation();
// await 的意思：「我要等一個 I/O，先讓其他 code 跑，I/O 完成再回來」
```

```csharp
// C# 的 await 底層可能在不同 Thread 恢復
// Node.js 的 await 永遠在同一個 Thread 恢復 ← 這是關鍵差異！
```

**口訣**：Node.js = 單線程免 Lock 的 ThreadPool，await 只是「讓出 CPU 時間」。

---

## 3.2 pnpm / npm（vs NuGet）

| 概念 | NuGet (C#) | pnpm/npm (Node.js) |
|------|-----------|-------------------|
| 套件管理器 | `dotnet add package` | `pnpm add` / `npm install` |
| 套件定義檔 | `.csproj` | `package.json` |
| Lock 檔 | `packages.lock.json` | `pnpm-lock.yaml` |
| 套件倉庫 | nuget.org | npmjs.com |
| 安裝目錄 | `~/.nuget/packages` | `node_modules/` (專案內) |
| 全域安裝 | GAC / global tools | `pnpm add -g` |

### package.json = .csproj

```jsonc
// package.json 核心欄位
{
  "name": "openclaw",                    // ≈ <AssemblyName>
  "version": "2026.2.9",                 // ≈ <Version>
  "main": "dist/index.js",              // ≈ 入口 DLL
  "scripts": {                           // ≈ MSBuild targets
    "build": "tsdown",                   // ≈ dotnet build
    "test": "vitest",                    // ≈ dotnet test
    "dev": "node scripts/run-node.mjs"   // ≈ dotnet run
  },
  "dependencies": {                      // ≈ <PackageReference>（執行時需要）
    "express": "^5.2.1",
    "ws": "^8.19.0"
  },
  "devDependencies": {                   // ≈ 開發時才用的套件
    "typescript": "^5.9.3",
    "vitest": "^4.0.18"
  }
}
```

### pnpm Workspace = .NET Solution

```yaml
# pnpm-workspace.yaml
packages:
  - .              # 主專案
  - ui             # Web UI 子專案
  - packages/*     # clawdbot, moltbot
  - extensions/*   # 35+ 擴充套件
```

```xml
<!-- 等價的 .sln 概念 -->
<!-- OpenClaw.sln -->
<!--   ├── OpenClaw.Core.csproj -->
<!--   ├── OpenClaw.UI.csproj -->
<!--   ├── OpenClaw.ClawdBot.csproj -->
<!--   └── Extensions/*.csproj -->
```

### 版本符號

```
"^5.2.1"  → >= 5.2.1 且 < 6.0.0  (主版號鎖定)
"~5.2.1"  → >= 5.2.1 且 < 5.3.0  (次版號鎖定)
"5.2.1"   → 精確鎖定
```

---

## 3.3 tsconfig.json（vs .csproj 編譯設定）

```jsonc
// tsconfig.json 核心欄位
{
  "compilerOptions": {
    "target": "ES2022",          // ≈ <TargetFramework>net9.0</TargetFramework>
    "module": "ESNext",          // 模組系統 (ESM)
    "strict": true,              // ≈ <Nullable>enable</Nullable> + 嚴格型別
    "outDir": "./dist",          // ≈ <OutputPath>bin/Release</OutputPath>
    "rootDir": "./src",          // 原始碼根目錄
    "declaration": true,         // ≈ 產生 .d.ts（類似 XML doc）
    "esModuleInterop": true,     // CommonJS/ESM 互通
    "skipLibCheck": true         // 跳過 node_modules 型別檢查（加速編譯）
  },
  "include": ["src/**/*"],       // ≈ <Compile Include="..." />
  "exclude": ["node_modules"]
}
```

---

## 3.4 專案建置流程

```
OpenClaw 建置流程：
pnpm install          ← dotnet restore（安裝依賴）
pnpm build            ← dotnet build（編譯 TS → JS）
  └→ tsdown           ← 使用 tsdown 打包（類似 dotnet publish --self-contained）
pnpm test             ← dotnet test
pnpm dev              ← dotnet run（開發模式，帶 hot reload）

Docker 建置：
docker build .        ← 使用 Dockerfile
docker-compose up     ← 啟動 Gateway + CLI
```

### 建置產物

```
src/          ← TypeScript 原始碼
  ↓ (tsdown 編譯)
dist/         ← JavaScript 產物（這才是 Node.js 實際執行的）
  ├── index.js
  ├── index.d.ts    ← 型別定義（給其他 TS 專案引用）
  └── ...
```

---

## 3.5 Docker 部署

### Dockerfile 解讀

```dockerfile
FROM node:22-bookworm           # 基底映像 ≈ mcr.microsoft.com/dotnet/aspnet:9.0

RUN corepack enable             # 啟用 pnpm（corepack 是 Node.js 的套件管理器管理器）
WORKDIR /app

COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile    # ≈ dotnet restore

COPY . .
RUN pnpm build                        # ≈ dotnet publish
RUN pnpm ui:build

USER node                             # 非 root 執行（安全最佳實踐）

CMD ["node", "dist/index.js", "gateway", "--allow-unconfigured"]
# ≈ ENTRYPOINT ["dotnet", "OpenClaw.dll", "gateway"]
```

### docker-compose.yml

```yaml
services:
  openclaw-gateway:             # ≈ API Server
    ports: ["18789:18789"]      # Gateway HTTP/WS port
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw    # 設定目錄
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace  # 工作區
    command: ["node", "dist/index.js", "gateway"]

  openclaw-cli:                 # ≈ Console App Client
    stdin_open: true            # 互動式終端
    tty: true
    entrypoint: ["node", "dist/index.js"]
```

---

## 3.6 常用 Node.js 內建模組

OpenClaw 裡常見的 `node:` 前綴 import：

```typescript
import fs from "node:fs";              // ≈ System.IO（檔案操作）
import path from "node:path";          // ≈ System.IO.Path
import process from "node:process";    // ≈ System.Environment / System.Diagnostics.Process
import { spawn } from "node:child_process"; // ≈ Process.Start()
import { fileURLToPath } from "node:url";   // URL 處理
import crypto from "node:crypto";      // ≈ System.Security.Cryptography
import os from "node:os";              // ≈ System.Environment.OSVersion
import { EventEmitter } from "node:events"; // ≈ event keyword / IObservable
```

---

## 3.7 實作：把 OpenClaw 跑起來

### 方法 1：Docker（推薦）

```bash
# 1. 複製環境變數範本
cp .env.example .env
# 2. 編輯 .env，填入你的 AI API Key
# 3. 建置 Docker 映像
docker build -t openclaw:local .
# 4. 啟動
docker-compose up openclaw-gateway
```

### 方法 2：本機開發

```bash
# 前提：安裝 Node.js 22+ 和 pnpm
node --version    # 確認 >= 22.12.0
pnpm --version    # 確認已安裝

# 1. 安裝依賴（≈ dotnet restore）
pnpm install

# 2. 建置（≈ dotnet build）
pnpm build

# 3. 啟動開發模式（≈ dotnet run）
pnpm dev gateway
```

---

## 今日閱讀作業

### 作業 1：閱讀 `package.json`
- 找出 dependencies 裡每個套件的用途（對照前幾天的架構圖）
- 找出 `scripts` 裡的 build / test / dev 指令

### 作業 2：閱讀 `Dockerfile`
- 用你熟悉的 .NET Docker 映像對照每一行
- 理解為什麼先 COPY package.json 再 COPY 原始碼（Layer Cache）

### 作業 3：閱讀 `docker-compose.yml`
- 理解 gateway 和 cli 兩個 service 的差異
- 找出環境變數的用途

---

## 今日 Checkpoint

1. Node.js Event Loop 和 C# ThreadPool 最大的差異是什麼？
2. `pnpm install` 等於 C# 的什麼指令？
3. `node_modules/` 等於 C# 的什麼目錄？
4. 為什麼 Dockerfile 先 COPY package.json 再 COPY 原始碼？
5. OpenClaw 的 Gateway 預設跑在哪個 port？

---

## 答案

1. **單執行緒 vs 多執行緒**。Node.js 用 Event Loop 在一個 thread 上輪流處理，不需要 lock。
2. `dotnet restore`
3. `~/.nuget/packages/`（NuGet 全域快取）
4. **Docker Layer Cache**。package.json 不常改，先 install 可以快取這一層，後續只重建原始碼層。
5. **18789**（docker-compose.yml 裡的 ports 設定）
