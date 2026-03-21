# Day 2：TypeScript 進階語法 + 非同步模型

> **目標**：掌握 Destructuring、Spread、Union Type、Generic、async/await、Module 系統
> **完成後**：你能看懂 OpenClaw 裡 90% 的 TS 語法
> **預計時間**：5 小時

---

## 2.1 Destructuring（解構賦值）

OpenClaw 程式碼裡**幾乎每個函式**都用到解構。這是今天最重要的概念。

### 物件解構

```typescript
// 原本的寫法
const id = channelMeta.id;
const label = channelMeta.label;
const docsPath = channelMeta.docsPath;

// 解構寫法：一行搞定（OpenClaw 到處都是這個）
const { id, label, docsPath } = channelMeta;
```

```csharp
// C# 等價：就是一次取出多個屬性
var id = channelMeta.Id;
var label = channelMeta.Label;
var docsPath = channelMeta.DocsPath;
```

### 函式參數解構（OpenClaw 最愛用的 Pattern）

```typescript
// 你會在 OpenClaw 裡看到大量這種寫法：
function resolveAllowFrom({ cfg, accountId }: {
  cfg: OpenClawConfig;
  accountId?: string | null;
}): string[] {
  return cfg.channels?.telegram?.allowFrom ?? [];
}

// 呼叫時：
resolveAllowFrom({ cfg: myConfig, accountId: "abc" });
```

```csharp
// C# 等價：用一個 record/class 包參數
record ResolveParams(OpenClawConfig Cfg, string? AccountId = null);

string[] ResolveAllowFrom(ResolveParams p)
    => p.Cfg.Channels?.Telegram?.AllowFrom ?? [];

// 呼叫：
ResolveAllowFrom(new(myConfig, "abc"));
```

**心法**：看到 `function foo({ a, b, c }: SomeType)` → 想成「C# 的 method 接收一個物件，然後在 method 開頭解構它的屬性」。

### 解構 + 重新命名 + 預設值

```typescript
// 重新命名：id 取出來但改叫 channelId
const { id: channelId, label: displayName } = meta;

// 預設值：如果 port 不存在就用 3000
const { port = 3000, host = "localhost" } = config;

// 巢狀解構
const { channels: { telegram: { allowFrom } } } = config;
// 等於 config.channels.telegram.allowFrom
```

```csharp
// C# 等價
var channelId = meta.Id;
var displayName = meta.Label;

var port = config.Port ?? 3000;
var host = config.Host ?? "localhost";

var allowFrom = config.Channels.Telegram.AllowFrom;
```

### 陣列解構

```typescript
const [first, second, ...rest] = ["a", "b", "c", "d"];
// first = "a", second = "b", rest = ["c", "d"]
```

```csharp
// C# 等價
var first = items[0];
var second = items[1];
var rest = items[2..];  // Range syntax
```

---

## 2.2 Spread Operator（展開運算子 `...`）

### 物件展開：合併/複製

```typescript
// 複製物件並覆寫部分屬性
const updated = { ...original, name: "new name", port: 4000 };

// 合併兩個物件（後者覆蓋前者）
const merged = { ...defaultConfig, ...userConfig };
```

```csharp
// C# 等價概念
// 用 record with：
var updated = original with { Name = "new name", Port = 4000 };

// 手動合併：
var merged = new Config
{
    Port = userConfig.Port ?? defaultConfig.Port,
    Host = userConfig.Host ?? defaultConfig.Host,
    // ... 每個屬性都要寫
};
```

### 陣列展開：合併/複製

```typescript
const combined = [...list1, ...list2];
const withNew = [...existing, newItem];
const copy = [...original];
```

```csharp
// C# 12 collection expression
var combined = [..list1, ..list2];
// 或 LINQ
var combined = list1.Concat(list2).ToArray();
```

### Rest 參數（收集剩餘）

```typescript
function log(first: string, ...rest: string[]) {
  console.log(first, rest);
}
```

```csharp
void Log(string first, params string[] rest)
{
    Console.WriteLine($"{first} {string.Join(" ", rest)}");
}
```

---

## 2.3 Union Type（聯合型別）

C# 目前沒有原生 Union Type，這是 TS 的殺手特色。

```typescript
// 基本 union：可以是 string 或 number
type StringOrNumber = string | number;

// 字面值 union（OpenClaw 大量使用）
type ChatType = "direct" | "group" | "channel" | "thread";

function handle(chatType: ChatType) {
  switch (chatType) {
    case "direct": /* ... */ break;
    case "group":  /* ... */ break;
  }
}

// 區分 union（Discriminated Union）
type Result =
  | { ok: true; data: string }
  | { ok: false; error: string };

function process(result: Result) {
  if (result.ok) {
    console.log(result.data);   // TS 知道這裡有 data
  } else {
    console.log(result.error);  // TS 知道這裡有 error
  }
}
```

```csharp
// C# 最接近的對照：

// 1. 字面值 union → enum
enum ChatType { Direct, Group, Channel, Thread }

// 2. Discriminated union → abstract record + pattern matching
abstract record Result;
record Success(string Data) : Result;
record Failure(string Error) : Result;

void Process(Result result)
{
    switch (result)
    {
        case Success s: Console.WriteLine(s.Data); break;
        case Failure f: Console.WriteLine(f.Error); break;
    }
}
```

---

## 2.4 Utility Types（內建工具型別）

TS 內建了很多便利的型別工具，OpenClaw 裡常見的：

```typescript
// Partial<T>：所有屬性變成可選
type PartialConfig = Partial<Config>;
// 等於把 Config 裡每個屬性都加上 ?

// Required<T>：所有屬性變成必要（Partial 的反面）
type FullConfig = Required<Config>;

// Pick<T, Keys>：只取部分屬性
type IdAndLabel = Pick<ChannelMeta, "id" | "label">;

// Omit<T, Keys>：排除部分屬性
type NoId = Omit<ChannelMeta, "id">;

// Record<K, V>：鍵值對映射
type ChannelMap = Record<string, ChannelDock>;
// 等於 { [key: string]: ChannelDock }
```

```csharp
// C# 沒有這些內建的，通常需要自己定義新 interface/record
// Partial<T> ≈ 所有屬性都是 nullable 的版本
// Pick ≈ 建一個新 interface 只有需要的屬性
// Record<K,V> ≈ Dictionary<K,V>
```

---

## 2.5 泛型 (Generics)

```typescript
// 基本泛型
function first<T>(items: T[]): T | undefined {
  return items[0];
}

// 泛型介面
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  save(entity: T): Promise<void>;
}

// 泛型約束（和 C# 的 where 一樣）
function getLabel<T extends { label: string }>(item: T): string {
  return item.label;
}

// 多重泛型
function transform<TInput, TOutput>(
  input: TInput,
  fn: (item: TInput) => TOutput,
): TOutput {
  return fn(input);
}
```

```csharp
// C# 完全對照
T? First<T>(T[] items) => items.FirstOrDefault();

interface IRepository<T>
{
    Task<T?> FindByIdAsync(string id);
    Task SaveAsync(T entity);
}

string GetLabel<T>(T item) where T : IHasLabel
    => item.Label;

TOutput Transform<TInput, TOutput>(
    TInput input, Func<TInput, TOutput> fn)
    => fn(input);
```

**好消息**：泛型語法 TS 和 C# 幾乎一模一樣。

---

## 2.6 async/await 與 Promise

### 基本用法

```typescript
// TS 的 async/await（和 C# 幾乎一模一樣）
async function loadConfig(): Promise<Config> {
  const raw = await readFile("config.yaml", "utf-8");
  const config = parseYaml(raw);
  return config;
}
```

```csharp
async Task<Config> LoadConfigAsync()
{
    var raw = await File.ReadAllTextAsync("config.yaml");
    var config = ParseYaml(raw);
    return config;
}
```

### 平行等待

```typescript
// TS：同時發多個非同步操作
const [user, config, channels] = await Promise.all([
  fetchUser(),
  loadConfig(),
  listChannels(),
]);

// 等價於 C# 的：
```

```csharp
var userTask = FetchUserAsync();
var configTask = LoadConfigAsync();
var channelsTask = ListChannelsAsync();
await Task.WhenAll(userTask, configTask, channelsTask);
var (user, config, channels) = (userTask.Result, configTask.Result, channelsTask.Result);
```

### Promise 建構（較少見但要認識）

```typescript
// 手動建立 Promise（類似 C# 的 TaskCompletionSource）
function waitForEvent(): Promise<string> {
  return new Promise((resolve, reject) => {
    emitter.on("data", (value) => resolve(value));
    emitter.on("error", (err) => reject(err));
  });
}
```

```csharp
// C# 等價
Task<string> WaitForEventAsync()
{
    var tcs = new TaskCompletionSource<string>();
    emitter.OnData += value => tcs.SetResult(value);
    emitter.OnError += err => tcs.SetException(err);
    return tcs.Task;
}
```

### void Promise（不回傳值）

```typescript
async function doWork(): Promise<void> { }   // TS
// 注意：void 的 Promise 不是 Promise<undefined>
```

```csharp
async Task DoWorkAsync() { }                   // C#
```

---

## 2.7 Module 系統 (ESM)

OpenClaw 使用 ES Modules (ESM)，每個 `.ts` 檔案就是一個模組。

### export（匯出）

```typescript
// 具名匯出（OpenClaw 主要用這種）
export function loadConfig() { ... }
export const VERSION = "1.0";
export interface Config { ... }
export class Gateway { ... }

// 在檔案底部統一匯出
const a = 1;
const b = 2;
export { a, b };
```

```csharp
// C# 等價：public 修飾子 + namespace
namespace OpenClaw.Config;

public static Config LoadConfig() { ... }
public const string VERSION = "1.0";
public interface IConfig { ... }
public class Gateway { ... }
```

### import（匯入）

```typescript
// 具名匯入（最常見）
import { loadConfig, type Config } from "./config/config.js";

// 全部匯入到一個 namespace 下
import * as config from "./config/config.js";
config.loadConfig();

// 只匯入型別（編譯後會被移除）
import type { Config } from "./config/config.js";

// 副作用匯入（只執行，不取值）
import "./init.js";  // 類似 C# 的 static constructor 被觸發
```

```csharp
// C# 對照
using OpenClaw.Config;                          // import { ... } from
using Config = OpenClaw.Config;                 // import * as
using static OpenClaw.Config.ConfigHelper;      // import { loadConfig }
```

### 重要：TS import 路徑帶 `.js`

```typescript
// OpenClaw 裡你會看到：
import { loadConfig } from "./config/config.js";
// 注意是 .js 不是 .ts！
// 這是 ESM 的規定：import 路徑是「編譯後」的路徑
```

**不要困惑**：雖然原始碼是 `.ts`，但 import 時寫 `.js` 是正確的。這是 Node.js ESM 的規定。

---

## 2.8 Callback、Higher-Order Function

```typescript
// TS：函式當參數傳（OpenClaw 裡大量使用）
function processItems<T>(items: T[], processor: (item: T) => void): void {
  for (const item of items) {
    processor(item);
  }
}

// 常見的 Array method 都是 Higher-Order Function
const names = channels.map((ch) => ch.label);
const active = channels.filter((ch) => ch.active);
const found = channels.find((ch) => ch.id === "telegram");
```

```csharp
// C# 完全等價
void ProcessItems<T>(T[] items, Action<T> processor)
{
    foreach (var item in items) processor(item);
}

var names = channels.Select(ch => ch.Label);
var active = channels.Where(ch => ch.Active);
var found = channels.FirstOrDefault(ch => ch.Id == "telegram");
```

---

## 2.9 Template Literal（模板字串）

```typescript
const name = "Wells";
const msg = `Hello ${name}, welcome to OpenClaw v${version}!`;
// 用反引號 ` 而非 " 或 '

// 多行字串（不需要 @）
const sql = `
  SELECT *
  FROM channels
  WHERE id = '${channelId}'
`;
```

```csharp
// C# 等價
var msg = $"Hello {name}, welcome to OpenClaw v{version}!";

// 多行
var sql = $"""
  SELECT *
  FROM channels
  WHERE id = '{channelId}'
  """;
```

**映射**：反引號 `` ` `` + `${}` = C# 的 `$""` + `{}`

---

## 今日閱讀作業

### 作業 1：閱讀 `src/version.ts`
- 找出 destructuring 的用法
- 找出 async/await 的用法
- 理解 module export 的結構

### 作業 2：閱讀 `src/channels/registry.ts`
- 找出 `as const` 的用法（第 7-16 行）
- 理解 `type ChatChannelId = (typeof CHAT_CHANNEL_ORDER)[number]` 在做什麼
- 找出 `Record<ChatChannelId, ChannelMeta>` 的用法
- 看看函式參數解構在哪裡出現

### 作業 3：閱讀 `src/channels/dock.ts`
- 這個檔案用到了今天學的幾乎所有語法
- 找出 Spread、Destructuring、Union Type、Optional Chaining 的用法
- 理解 `ChannelDock` type 的定義方式

---

## 今日 Checkpoint

1. `const { a, b } = obj` 在做什麼？C# 怎麼表達？
2. `{ ...x, ...y }` 如果 x 和 y 有同名屬性，結果是什麼？
3. `string | number | null` 在 C# 怎麼表達？
4. `Promise.all([a(), b()])` 對應 C# 的什麼？
5. `import { X } from "./x.js"` 為什麼是 `.js` 不是 `.ts`？

---

## 答案

1. 把 `obj.a` 和 `obj.b` 分別取出放到 local 變數。C#：`var a = obj.A; var b = obj.B;`
2. **y 的屬性會覆蓋 x 的**（後者優先）。就像 C# 字典的 `dict[key] = newValue` 覆蓋舊值。
3. C# 沒有直接等價，最接近的：用 `object?` 或 method overload。
4. `Task.WhenAll(a(), b())`
5. ESM 規定 import 路徑是編譯後的路徑（.ts 編譯成 .js），這是 Node.js 的要求。
