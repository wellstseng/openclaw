# TypeScript ↔ C# 語法速查表

> 給 C# 開發者的隨身翻譯卡。遇到看不懂的 TS 語法就來查這張表。

---

## 1. 變數宣告

```typescript
// TS
const name = "openclaw";        // 不可重新賦值 (類似 C# readonly local)
let count = 0;                   // 可重新賦值
var legacy = "avoid this";       // 舊語法，不要用
```

```csharp
// C#
readonly string name = "openclaw";  // 或 const（編譯期常數）
int count = 0;                      // 預設就是可變的
// var 在 C# 是型別推斷，不同於 JS 的 var
```

**口訣**：`const` = 不能改的 `var`、`let` = 能改的 `var`

---

## 2. 基本型別

| TypeScript | C# | 備註 |
|------------|-----|------|
| `string` | `string` | 一樣 |
| `number` | `int` / `double` | TS 不分整數/浮點 |
| `boolean` | `bool` | 小寫 |
| `null` | `null` | 一樣 |
| `undefined` | 無（最接近 `null`） | TS 獨有，表示「未定義」 |
| `any` | `dynamic` | 放棄型別檢查 |
| `unknown` | `object` | 安全的 any，用前要 type guard |
| `void` | `void` | 一樣 |
| `never` | 無 | 永遠不會回傳（如 throw） |
| `string[]` | `string[]` | 一樣 |
| `[string, number]` | `(string, int)` | Tuple |
| `Record<string, T>` | `Dictionary<string, T>` | 鍵值對 |
| `Map<K, V>` | `Dictionary<K, V>` | 幾乎相同 |
| `Set<T>` | `HashSet<T>` | 幾乎相同 |
| `Promise<T>` | `Task<T>` | 非同步返回值 |

---

## 3. 函式

```typescript
// TS：函式宣告
function add(a: number, b: number): number {
  return a + b;
}

// TS：箭頭函式 (Arrow Function) ← 你會看到超多這個
const add = (a: number, b: number): number => a + b;

// TS：選擇性參數 + 預設值
function greet(name: string, greeting?: string, times: number = 1): void { }

// TS：剩餘參數
function log(...args: string[]): void { }
```

```csharp
// C#：等價寫法
int Add(int a, int b) => a + b;

// C#：Lambda (最接近箭頭函式)
Func<int, int, int> add = (a, b) => a + b;

// C#：選擇性參數 + 預設值
void Greet(string name, string? greeting = null, int times = 1) { }

// C#：params
void Log(params string[] args) { }
```

**重點**：TS 的 `=>` 就是 C# 的 Lambda `=>`，只是用的地方更廣。

---

## 4. 介面 & 型別

```typescript
// TS：interface
interface ChannelMeta {
  id: string;
  label: string;
  docsPath?: string;          // ? 表示選擇性（可以沒有）
  readonly blurb: string;     // 唯讀
}

// TS：type alias（C# 沒有完全對等的）
type ChatType = "direct" | "group" | "channel";  // Union Type
type StringOrNumber = string | number;

// TS：intersection type
type FullChannel = ChannelMeta & ChannelCapabilities; // 合併兩個型別
```

```csharp
// C#：interface
public interface IChannelMeta
{
    string Id { get; }
    string Label { get; }
    string? DocsPath { get; }    // ? 表示 nullable
    string Blurb { get; }
}

// C#：最接近 Union Type 的做法
public enum ChatType { Direct, Group, Channel }
// 或在未來的 C# discriminated union

// C#：intersection → 多重介面繼承
public interface IFullChannel : IChannelMeta, IChannelCapabilities { }
```

**雷區**：TS 的 `?` 是「可能沒這個屬性」，C# 的 `?` 是「可能是 null」。微妙不同但實務上差不多。

---

## 5. Destructuring 解構

```typescript
// TS：物件解構
const { id, label, docsPath } = channelMeta;

// TS：陣列解構
const [first, second, ...rest] = items;

// TS：函式參數解構 ← OpenClaw 裡超常見
function configure({ port, host, token }: ServerConfig) {
  console.log(port, host);
}

// TS：解構 + 重新命名
const { id: channelId, label: displayName } = meta;

// TS：解構 + 預設值
const { port = 3000, host = "localhost" } = config;
```

```csharp
// C#：沒有直接等價，最接近的寫法
var id = channelMeta.Id;
var label = channelMeta.Label;
var docsPath = channelMeta.DocsPath;

// C# Tuple 解構
var (first, second) = (items[0], items[1]);

// C# 沒有參數解構，要寫完整
void Configure(ServerConfig config)
{
    var port = config.Port;
    var host = config.Host;
}

// C# 重新命名：就是換個 local variable name
var channelId = meta.Id;
```

**心法**：看到 `const { a, b } = obj` 就當成「一次性把 obj.a, obj.b 取出來」。

---

## 6. Spread Operator 展開

```typescript
// TS：物件展開（合併/複製）
const merged = { ...defaultConfig, ...userConfig };
const copy = { ...original, name: "new name" };

// TS：陣列展開
const all = [...list1, ...list2, newItem];

// TS：函式呼叫展開
Math.max(...numbers);
```

```csharp
// C#：物件合併（最接近的寫法）
var merged = defaultConfig with { /* 用 userConfig 的值覆蓋 */ };
// 或手動：
var merged = new Config
{
    Port = userConfig.Port ?? defaultConfig.Port,
    Host = userConfig.Host ?? defaultConfig.Host,
};

// C#：陣列合併
var all = list1.Concat(list2).Append(newItem).ToArray();
// 或 C# 12 collection expression:
int[] all = [..list1, ..list2, newItem];

// C#：展開呼叫
numbers.Max(); // LINQ
```

---

## 7. Optional Chaining & Nullish Coalescing

```typescript
// TS
const name = user?.profile?.name;          // 任何一層是 null/undefined 就回傳 undefined
const port = config.port ?? 3000;          // null 或 undefined 才用預設值
const label = channel?.label ?? "unknown";
```

```csharp
// C#：完全一樣！
var name = user?.Profile?.Name;
var port = config.Port ?? 3000;
var label = channel?.Label ?? "unknown";
```

**好消息**：C# 6+ 的 `?.` 和 `??` 跟 TS 的行為一模一樣。

---

## 8. 非同步 async/await

```typescript
// TS
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();
  return data as User;
}

// TS：同時等多個
const [user, config] = await Promise.all([
  fetchUser("123"),
  loadConfig(),
]);

// TS：錯誤處理
try {
  await riskyOperation();
} catch (error) {
  if (error instanceof AuthError) { ... }
}
```

```csharp
// C#：幾乎一模一樣
async Task<User> FetchUserAsync(string id)
{
    var response = await _httpClient.GetAsync($"/api/users/{id}");
    var data = await response.Content.ReadFromJsonAsync<User>();
    return data!;
}

// C#：同時等多個
var (user, config) = await (
    FetchUserAsync("123"),
    LoadConfigAsync()
);
// 或 Task.WhenAll

// C#：錯誤處理
try
{
    await RiskyOperationAsync();
}
catch (AuthException ex) { ... }
```

**唯一差異**：C# 慣例方法名加 `Async` 後綴，TS 不加。

---

## 9. 模組系統 (import/export)

```typescript
// TS：具名匯出
export function loadConfig() { ... }
export const VERSION = "1.0";
export interface Config { ... }

// TS：具名匯入
import { loadConfig, VERSION } from "./config.js";

// TS：全部匯入
import * as config from "./config.js";

// TS：預設匯出（較少用）
export default class Server { ... }
import Server from "./server.js";

// TS：type-only import
import type { Config } from "./config.js";
```

```csharp
// C#：namespace + using
namespace OpenClaw.Config;
public static Config LoadConfig() { ... }
public const string VERSION = "1.0";
public interface IConfig { ... }

// C#：using
using OpenClaw.Config;
// 或
using static OpenClaw.Config.ConfigHelper;  // ≈ import { loadConfig }
```

**重點**：TS 的 `import/export` = C# 的 `using/namespace + public`。每個 `.ts` 檔案就是一個模組。

---

## 10. 類別

```typescript
// TS
class Gateway {
  private readonly port: number;
  protected config: Config;

  constructor(port: number, config: Config) {
    this.port = port;
    this.config = config;
  }

  async start(): Promise<void> {
    // ...
  }

  static create(config: Config): Gateway {
    return new Gateway(config.port, config);
  }
}

class SecureGateway extends Gateway {
  constructor(config: Config) {
    super(config.port, config);
  }
}
```

```csharp
// C#：幾乎一模一樣
class Gateway
{
    private readonly int _port;
    protected Config Config;

    public Gateway(int port, Config config)
    {
        _port = port;
        Config = config;
    }

    public async Task StartAsync()
    {
        // ...
    }

    public static Gateway Create(Config config)
        => new(config.Port, config);
}

class SecureGateway : Gateway
{
    public SecureGateway(Config config) : base(config.Port, config) { }
}
```

**差異**：TS 用 `extends`，C# 用 `:`。其他幾乎一樣。

---

## 11. 泛型

```typescript
// TS
function first<T>(items: T[]): T | undefined {
  return items[0];
}

interface Repository<T> {
  findById(id: string): Promise<T | null>;
  save(entity: T): Promise<void>;
}

// TS：泛型約束
function getLabel<T extends { label: string }>(item: T): string {
  return item.label;
}
```

```csharp
// C#
T? First<T>(T[] items) => items.FirstOrDefault();

interface IRepository<T>
{
    Task<T?> FindByIdAsync(string id);
    Task SaveAsync(T entity);
}

// C#：泛型約束
string GetLabel<T>(T item) where T : IHasLabel => item.Label;
```

---

## 12. 常用陣列操作對照

| TS | C# (LINQ) | 說明 |
|----|-----------|------|
| `arr.map(x => x.name)` | `arr.Select(x => x.Name)` | 轉換 |
| `arr.filter(x => x.active)` | `arr.Where(x => x.Active)` | 篩選 |
| `arr.find(x => x.id === id)` | `arr.FirstOrDefault(x => x.Id == id)` | 找第一個 |
| `arr.some(x => x.valid)` | `arr.Any(x => x.Valid)` | 存在判斷 |
| `arr.every(x => x.valid)` | `arr.All(x => x.Valid)` | 全部判斷 |
| `arr.reduce((acc, x) => acc + x, 0)` | `arr.Aggregate(0, (acc, x) => acc + x)` | 累加 |
| `arr.flat()` | `arr.SelectMany(x => x)` | 攤平 |
| `arr.includes(item)` | `arr.Contains(item)` | 包含 |
| `arr.sort((a, b) => a - b)` | `arr.OrderBy(x => x)` | 排序 |
| `arr.slice(1, 3)` | `arr.Skip(1).Take(2)` | 切片 |
| `arr.forEach(x => ...)` | `arr.ForEach(x => ...)` / `foreach` | 遍歷 |
| `arr.length` | `arr.Length` / `arr.Count` | 長度 |
| `[...new Set(arr)]` | `arr.Distinct()` | 去重 |

---

## 13. 錯誤處理

```typescript
// TS：自訂 Error
class PortInUseError extends Error {
  constructor(public port: number) {
    super(`Port ${port} is in use`);
    this.name = "PortInUseError";
  }
}

// TS：throw + catch
try {
  throw new PortInUseError(3000);
} catch (err) {
  if (err instanceof PortInUseError) {
    console.error(`Port ${err.port} busy`);
  }
}
```

```csharp
// C#
class PortInUseException : Exception
{
    public int Port { get; }
    public PortInUseException(int port)
        : base($"Port {port} is in use") => Port = port;
}

try
{
    throw new PortInUseException(3000);
}
catch (PortInUseException ex)
{
    Console.Error.WriteLine($"Port {ex.Port} busy");
}
```

---

## 14. 常見 Pattern 速查

### 立即解構的 Import

```typescript
// 你會在每個檔案開頭看到這個
import { loadConfig, type Config } from "./config/config.js";
```

→ 就是 C# 的 `using static` + 型別引用。

### 物件字面值（Object Literal）

```typescript
const dock: ChannelDock = {
  id: "telegram",
  capabilities: { chatTypes: ["direct", "group"], media: true },
};
```

→ 就是 C# 的物件初始化器 `new ChannelDock { Id = "telegram", ... }`

### 型別斷言（Type Assertion）

```typescript
const value = something as string;       // 告訴編譯器「我確定這是 string」
const value = <string>something;          // 另一種寫法（JSX 會衝突）
```

→ 就是 C# 的 `(string)something` 強制轉型，但**不做運行時檢查**。

### 型別守衛（Type Guard）

```typescript
if (typeof value === "string") { ... }    // 基本型別
if (value instanceof Error) { ... }       // 類別
if ("port" in config) { ... }             // 屬性存在檢查
```

→ 就是 C# 的 `is` pattern matching：`if (value is string s) { ... }`

### 模板字串（Template Literal）

```typescript
const msg = `Hello ${name}, port is ${port}`;
```

→ 就是 C# 的 `$"Hello {name}, port is {port}"`，只是用反引號 `` ` `` 而非 `"`。

---

## 15. OpenClaw 專案常見縮寫

| 縮寫 | 全稱 | 說明 |
|------|------|------|
| `cfg` | config | 設定物件 |
| `ctx` | context | 上下文 |
| `ws` | WebSocket | WebSocket 連線 |
| `pi` | pi-agent | AI coding agent 引擎 |
| `e164` | E.164 | 國際電話號碼格式 |
| `jid` | Jabber ID | WhatsApp/XMPP 用戶識別 |
| `ct` | CancellationToken | 取消令牌（TS 用 AbortSignal） |
| `tui` | Terminal UI | 終端介面 |
| `acp` | Agent Client Protocol | Agent 通訊協定 |

---

> 本速查表涵蓋 OpenClaw 原始碼中 95% 以上的 TS 語法。
> 遇到未列出的語法，直接問雙語架構師。
