# 07-SECURITY — Security 安全機制

> 來源：openclaw-knowledge-base.md §11 + F058 §12

---

## 1. SSRF Blocked IPv4 Ranges（完整 8 種）

```typescript
// GuardedFetch 阻擋以下 IPv4 範圍（防 SSRF 攻擊）
const BLOCKED_IPV4_RANGES = [
  // 1. Loopback
  { start: "127.0.0.0",   end: "127.255.255.255", cidr: "127.0.0.0/8" },

  // 2. Private A 類（RFC 1918）
  { start: "10.0.0.0",    end: "10.255.255.255",  cidr: "10.0.0.0/8" },

  // 3. Private B 類（RFC 1918）
  { start: "172.16.0.0",  end: "172.31.255.255",  cidr: "172.16.0.0/12" },

  // 4. Private C 類（RFC 1918）
  { start: "192.168.0.0", end: "192.168.255.255", cidr: "192.168.0.0/16" },

  // 5. Link-local（APIPA）
  { start: "169.254.0.0", end: "169.254.255.255", cidr: "169.254.0.0/16" },

  // 6. Localhost alternative
  { start: "0.0.0.0",     end: "0.255.255.255",   cidr: "0.0.0.0/8" },

  // 7. Multicast
  { start: "224.0.0.0",   end: "239.255.255.255", cidr: "224.0.0.0/4" },

  // 8. Broadcast / Reserved
  { start: "240.0.0.0",   end: "255.255.255.255", cidr: "240.0.0.0/4" },
];
```

---

## 2. SSRF Blocked IPv6 Ranges（完整 5 種）

```typescript
const BLOCKED_IPV6_RANGES = [
  // 1. Loopback（::1）
  "::1/128",

  // 2. Link-local（fe80::/10）
  "fe80::/10",

  // 3. Unique local（fc00::/7，含 fc:: 和 fd::）
  "fc00::/7",

  // 4. Multicast（ff00::/8）
  "ff00::/8",

  // 5. Unspecified（::）
  "::/128",
];
```

---

## 3. BLOCKED_HOSTNAMES

```typescript
// 永遠阻擋的主機名稱（不論 IP 解析結果）
const BLOCKED_HOSTNAMES = [
  "localhost",
  "local",
  "internal",
  "metadata",              // 雲端 metadata 服務
  "169.254.169.254",       // AWS/GCP/Azure metadata
  "instance-data",         // AWS EC2 instance metadata
  "metadata.google.internal", // GCP metadata
];
```

---

## 4. IPv6 Tunnel 解包（5 種格式）

```typescript
// 防止透過 IPv6 tunnel 繞過 SSRF 防護
// 解包後再次驗證包含的 IPv4 地址

// 1. IPv4-mapped IPv6（::ffff:x.x.x.x）
function unpackIPv4Mapped(ipv6: string): string | null {
  const match = ipv6.match(/^::ffff:(\d+\.\d+\.\d+\.\d+)$/i);
  return match ? match[1] : null;
}

// 2. IPv4-compatible IPv6（::x.x.x.x，已廢棄但仍需處理）
function unpackIPv4Compatible(ipv6: string): string | null {
  const match = ipv6.match(/^::(\d+\.\d+\.\d+\.\d+)$/);
  return match ? match[1] : null;
}

// 3. 6to4（2002:xx:xx::/48）
function unpack6to4(ipv6: string): string | null {
  const match = ipv6.match(/^2002:([0-9a-f]{2})([0-9a-f]{2}):([0-9a-f]{2})([0-9a-f]{2}):/i);
  if (!match) return null;
  return `${parseInt(match[1], 16)}.${parseInt(match[2], 16)}.${parseInt(match[3], 16)}.${parseInt(match[4], 16)}`;
}

// 4. Teredo（2001:0000::/32）— 含 XOR 算法
function unpackTeredo(ipv6: string): string | null {
  // Teredo 格式：2001:0000:{server-ipv4}:{flags}:{port XOR}:{client-ipv4 XOR}
  const match = ipv6.match(/^2001:0{0,4}:([0-9a-f]{4}):([0-9a-f]{4}):([0-9a-f]{4}):([0-9a-f]{4}):([0-9a-f]{4}):([0-9a-f]{4})$/i);
  if (!match) return null;
  // Client IPv4 = 最後 32 bits XOR 0xffffffff（位元反轉）
  const hi = parseInt(match[5], 16) ^ 0xffff;
  const lo = parseInt(match[6], 16) ^ 0xffff;
  return `${hi >> 8}.${hi & 0xff}.${lo >> 8}.${lo & 0xff}`;
}

// 5. ISATAP（::5efe:x.x.x.x 或 ::200:5efe:x.x.x.x）
function unpackISATAP(ipv6: string): string | null {
  const match = ipv6.match(/::(?:200:)?5efe:(\d+\.\d+\.\d+\.\d+)$/i);
  return match ? match[1] : null;
}
```

---

## 5. GuardedFetch 完整 Options 型別

```typescript
interface GuardedFetchOptions extends RequestInit {
  // SSRF 防護設定
  mode?: "strict" | "trusted_env_proxy";

  // Redirect 設定
  maxRedirects?: number;         // 最大重定向次數（預設 3）
  followRedirects?: boolean;     // 是否跟隨重定向（預設 true）

  // Timeout 設定
  timeoutMs?: number;            // 請求超時（ms）

  // 信任設定
  allowPrivateIp?: boolean;      // 允許私有 IP（trusted_env_proxy 模式用）
  trustedHosts?: string[];       // 信任的主機名稱列表

  // 大小限制
  maxBodyBytes?: number;         // 回應 body 最大大小（bytes）

  // 日誌
  logSsrfAttempts?: boolean;     // 記錄 SSRF 嘗試
}
```

### GuardedFetch 模式

| 模式 | 說明 |
|------|------|
| `"strict"` | 預設模式，阻擋所有私有/loopback IP |
| `"trusted_env_proxy"` | 允許私有 IP（用於本地 proxy，如 Ollama） |

### DEFAULT_MAX_REDIRECTS

```typescript
const DEFAULT_MAX_REDIRECTS = 3;
```

---

## 6. ExecHost / ExecSecurity / ExecAsk 型別

### ExecHost

```typescript
type ExecHost =
  | "sandbox"   // 沙盒環境（隔離執行）
  | "gateway"   // Gateway 主機（有網路存取）
  | "node";     // Companion node 主機
```

### ExecSecurity

```typescript
type ExecSecurity =
  | "deny"       // 拒絕所有 exec（最嚴格）
  | "allowlist"  // 只允許 allowlist 中的命令
  | "full";      // 允許所有（最寬鬆）
```

### ExecAsk

```typescript
type ExecAsk =
  | "off"        // 不詢問，直接執行（依 ExecSecurity）
  | "on-miss"    // 只有 allowlist miss 時詢問
  | "always";    // 每次執行都詢問確認
```

---

## 7. Exec Approval 流程

```typescript
// 完整 exec approval 流程：
// 1. Agent 發出 exec 請求
// 2. 查找 ExecSecurity 配置：
//    - "deny" → 立即拒絕，回傳錯誤
//    - "allowlist" → 檢查命令是否在 allowlist
//    - "full" → 直接執行（跳過步驟 3-5）
// 3. 若 allowlist miss 且 ExecAsk = "always" 或 "on-miss"：
//    → 發送審批請求到 Channel（Discord/Telegram 等）
//    → 等待用戶 /approve 或 /deny
// 4. 用戶批准 → 執行命令
// 5. 用戶拒絕 → 回傳拒絕錯誤
// 6. 超時（預設 5min）→ 視同拒絕

interface ExecApprovalRequest {
  command: string;           // 要執行的命令
  args?: string[];           // 命令參數
  workingDir?: string;       // 工作目錄
  env?: Record<string, string>; // 環境變數
  reason?: string;           // Agent 說明執行原因
}
```

---

## 8. World-Writable Plugin 處理邏輯

```typescript
// 安全檢查：Plugin 目錄/檔案的權限
// 若 Plugin 路徑為 world-writable（任何人可寫入）：
// → 拒絕載入（安全風險：惡意程式碼注入）
// → 記錄警告到 log
// → 繼續載入其他 Plugin

// 檢查方式（Unix）：
// stat(path).mode & 0o002 !== 0 → world-writable
// 在 Windows 上略過此檢查
```

---

## 9. Control Plane Rate Limit

```typescript
// Sliding window rate limit
// by deviceId（優先）或 clientIp（fallback）

const CONTROL_PLANE_RATE_LIMIT = {
  windowMs: 60000,         // 1 分鐘視窗
  maxRequests: 100,        // 每視窗最大請求數（一般 API）
  strategy: "sliding",     // Sliding window（精確）
};

// Auth 端點（更嚴格）
const AUTH_RATE_LIMIT = {
  windowMs: 60000,         // 1 分鐘視窗
  maxRequests: 10,         // 每視窗 10 次
  lockDurationMs: 300000,  // 超限後 lock 5 分鐘
};

// Hook auth 端點
const HOOK_AUTH_RATE_LIMIT = {
  windowMs: 60000,
  maxRequests: 20,         // 每視窗 20 次
};
```

---

## 10. audit-channel / mutable-allowlist-detectors

```typescript
// src/security/audit-channel.ts：
// 審計頻道（記錄安全事件）

// src/security/mutable-allowlist-detectors.ts：
// 動態 allowlist 條目偵測

// Discord 可變 allowlist 條目格式
function isDiscordMutableAllowEntry(entry: string): boolean {
  // 形如 "discord:user:*"（萬用字元）
  return entry.startsWith("discord:") && entry.includes(":*");
}

// Zalouser 可變群組條目格式
function isZalouserMutableGroupEntry(entry: string): boolean {
  return entry.startsWith("zalouser:group:");
}
```

---

## 11. dangerous-config-flags

```typescript
// src/security/dangerous-config-flags.ts：
// 偵測危險的配置組合，在啟動時警告

const DANGEROUS_FLAGS = [
  { flag: "execSecurity=full",   severity: "warning", message: "允許任意命令執行" },
  { flag: "allowFrom=*",         severity: "warning", message: "允許任意來源（公開 Bot）" },
  { flag: "dmPolicy=open",       severity: "info",    message: "開放 DM 存取" },
  { flag: "allowBots=all",       severity: "info",    message: "允許所有 Bot 訊息" },
];
```
