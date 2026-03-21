# Day 1：TypeScript 基礎語法（C# 開發者速成版）

> **目標**：看完今天的內容後，你能看懂 OpenClaw 裡 60% 的 TS 程式碼。
> **預計時間**：4 小時

---

## 1.1 TypeScript 是什麼？一句話講完

**TypeScript = JavaScript + C# 風格的型別系統**

```
C#     → 編譯成 → IL (中間語言) → CLR 執行
TypeScript → 編譯成 → JavaScript   → Node.js 執行
```

你可以把 TS 想成：「有型別檢查的 JavaScript」，就像 C# 之於 IL。
編譯器（`tsc`）在編譯階段做型別檢查，產出的 JS 完全沒有型別資訊。

---

## 1.2 變數宣告：const / let / var

### C# 開發者只需要記住兩個

```typescript
const port = 3000;      // 宣告後不能重新賦值 → 類似 C# 的 readonly local
let count = 0;           // 可以重新賦值 → 類似 C# 的普通變數
count = 5;               // OK
// port = 4000;          // 編譯錯誤！const 不能改
```

```csharp
// C# 等價思維
readonly int port = 3000;  // 不能改（概念上）
int count = 0;             // 能改
```

**原則**：OpenClaw 的 code style 是 **能用 `const` 就用 `const`**，只有真的需要改值才用 `let`。

### 注意：const 只防「重新賦值」，不防「修改內容」

```typescript
const config = { port: 3000 };
config.port = 4000;   // OK！物件的屬性可以改
// config = {};        // 錯誤！不能重新指向另一個物件
```

```csharp
// C# 等價：
readonly var config = new Config { Port = 3000 };
config.Port = 4000;   // OK（如果 Port 不是 readonly）
// config = new Config();  // 錯誤！readonly 不能重新賦值
```

---

## 1.3 基本型別

```typescript
// TS 的型別標註寫在變數「後面」，用冒號分隔
const name: string = "openclaw";
const port: number = 18789;        // 不分 int/double/float，統一叫 number
const active: boolean = true;
const data: null = null;
const items: string[] = ["a", "b"];
const pair: [string, number] = ["port", 3000];  // Tuple
```

```csharp
// C# 對照
string name = "openclaw";
int port = 18789;              // C# 分 int / double / float / decimal
bool active = true;
string? data = null;
string[] items = ["a", "b"];
(string, int) pair = ("port", 3000);
```

### 型別推斷：通常不需要寫型別

```typescript
// TS 有型別推斷，跟 C# 的 var 一樣
const name = "openclaw";         // 自動推斷為 string
const port = 18789;              // 自動推斷為 number
const items = ["a", "b"];        // 自動推斷為 string[]
```

**OpenClaw 程式碼裡大多數都省略型別標註**（因為能推斷），你看到 `const x = ...` 時不要慌，編譯器知道它是什麼型別。

---

## 1.4 函式

### 基本函式

```typescript
// 寫法 1：function 宣告
function add(a: number, b: number): number {
  return a + b;
}

// 寫法 2：箭頭函式（你在 OpenClaw 裡會看到更多這個）
const add = (a: number, b: number): number => a + b;

// 寫法 3：省略回傳型別（自動推斷）
const add = (a: number, b: number) => a + b;
```

```csharp
// C# 對照
int Add(int a, int b) => a + b;

// 或 Lambda
Func<int, int, int> add = (a, b) => a + b;
```

**口訣**：箭頭函式 `() => {}` 就是 C# 的 Lambda `() => {}`。

### 選擇性參數 & 預設值

```typescript
function greet(name: string, greeting?: string): string {
  // greeting 可能是 undefined
  return `${greeting ?? "Hello"}, ${name}`;
}

greet("Wells");              // "Hello, Wells"
greet("Wells", "Hey");       // "Hey, Wells"

// 預設值
function connect(host: string, port: number = 3000): void { }
```

```csharp
// C#
string Greet(string name, string? greeting = null)
    => $"{greeting ?? "Hello"}, {name}";

void Connect(string host, int port = 3000) { }
```

---

## 1.5 介面 (Interface)

```typescript
// TS 的 interface
interface ChannelMeta {
  id: string;
  label: string;
  docsPath?: string;        // ? 表示這個屬性可有可無
  readonly blurb: string;   // 唯讀
}

// 使用：直接用物件字面值（不需要 new）
const meta: ChannelMeta = {
  id: "telegram",
  label: "Telegram",
  blurb: "simplest way to get started",
};
```

```csharp
// C# 對照
public interface IChannelMeta
{
    string Id { get; }
    string Label { get; }
    string? DocsPath { get; }   // nullable
    string Blurb { get; }
}

// C# 使用：需要一個實作類別
public record ChannelMeta : IChannelMeta
{
    public string Id { get; init; }
    public string Label { get; init; }
    public string? DocsPath { get; init; }
    public string Blurb { get; init; }
}
```

### TS Interface 的超能力：結構型別 (Structural Typing)

```typescript
// TS 不看「你是誰」，只看「你有什麼」
interface HasId { id: string; }

const channel = { id: "telegram", label: "Telegram", port: 443 };

function printId(item: HasId) {
  console.log(item.id);
}

printId(channel);  // OK！channel 有 id 屬性就行
```

```csharp
// C# 是名義型別 (Nominal Typing)：必須顯式實作介面
// channel 必須宣告 : IHasId 才能傳給 PrintId(IHasId item)
```

**這是 TS 和 C# 最大的差異之一**。TS 只看「形狀」(shape) 吻不吻合，不管你有沒有明確 `implements`。就像鴨子定型法 (duck typing)：「走起來像鴨子、叫起來像鴨子，那就是鴨子。」

---

## 1.6 類別 (Class)

```typescript
class Gateway {
  // 屬性直接宣告（不需要 field + property 分離）
  private port: number;
  public name: string;
  protected config: Record<string, unknown>;

  constructor(port: number, name: string) {
    this.port = port;
    this.name = name;
    this.config = {};
  }

  // 方法
  async start(): Promise<void> {
    console.log(`Starting on port ${this.port}`);
  }

  // getter（和 C# property get 幾乎一樣）
  get address(): string {
    return `localhost:${this.port}`;
  }

  // static
  static defaultPort(): number {
    return 18789;
  }
}

// 繼承
class SecureGateway extends Gateway {
  constructor(port: number) {
    super(port, "secure-gateway");  // 呼叫父建構式
  }
}
```

```csharp
// C# 對照
class Gateway
{
    private int _port;
    public string Name;
    protected Dictionary<string, object> Config;

    public Gateway(int port, string name)
    {
        _port = port;
        Name = name;
        Config = new();
    }

    public async Task StartAsync()
    {
        Console.WriteLine($"Starting on port {_port}");
    }

    public string Address => $"localhost:{_port}";

    public static int DefaultPort() => 18789;
}

class SecureGateway : Gateway
{
    public SecureGateway(int port) : base(port, "secure-gateway") { }
}
```

**差異**：
- TS 用 `extends`，C# 用 `:`
- TS 的 `super()` = C# 的 `base()`
- TS 預設 `public`，C# 預設 `private`（要注意！）

---

## 1.7 Enum

```typescript
// TS：數字 enum（和 C# 一樣）
enum Direction {
  Up,      // 0
  Down,    // 1
  Left,    // 2
  Right,   // 3
}

// TS：字串 enum（C# 沒有原生支援）
enum ChatType {
  Direct = "direct",
  Group = "group",
  Channel = "channel",
}
```

```csharp
// C#
enum Direction { Up, Down, Left, Right }

// C# 字串 enum：需要用 attribute 或 static class 模擬
public static class ChatType
{
    public const string Direct = "direct";
    public const string Group = "group";
    public const string Channel = "channel";
}
```

### OpenClaw 更常用 `as const` 代替 enum

```typescript
// 這在 OpenClaw 裡更常見：
const CHAT_CHANNEL_ORDER = [
  "telegram", "whatsapp", "discord", "slack",
] as const;

type ChatChannelId = (typeof CHAT_CHANNEL_ORDER)[number];
// 等於 type ChatChannelId = "telegram" | "whatsapp" | "discord" | "slack"
```

```csharp
// C# 等價理解：一個 readonly string array + 限制只能用這些值
public static readonly string[] ChatChannelOrder =
    ["telegram", "whatsapp", "discord", "slack"];
```

---

## 1.8 型別斷言 & 型別守衛

```typescript
// 型別斷言：「我比編譯器更清楚這是什麼型別」
const value = someFunction() as string;
const port = JSON.parse(data) as number;
```

```csharp
// C# 等價
var value = (string)SomeFunction();
var port = (int)JsonSerializer.Deserialize(data);
```

```typescript
// 型別守衛：安全的型別判斷
if (typeof value === "string") {
  // 在這個 block 裡，value 自動被當成 string
  console.log(value.toUpperCase());
}

if (error instanceof PortInUseError) {
  // error 自動被當成 PortInUseError
  console.log(error.port);
}
```

```csharp
// C# 等價：pattern matching
if (value is string s)
{
    Console.WriteLine(s.ToUpper());
}

if (error is PortInUseException ex)
{
    Console.WriteLine(ex.Port);
}
```

---

## 今日閱讀作業

### 作業 1：閱讀 `src/globals.ts`
- 這是一個非常小的檔案，展示了基本的 function、export、const
- 找出所有 export 的函式，用 C# 的概念理解它們

### 作業 2：閱讀 `src/utils.ts`
- 找到 `normalizeE164` 函式，用 C# 邏輯理解它在做什麼
- 找到 `escapeRegExp` 函式，對照 C# 的 `Regex.Escape()`
- 看看 `assertWebChannel` 怎麼做型別斷言

### 作業 3：閱讀 `src/version.ts`
- 理解它怎麼讀取 package.json 的版本號
- 對照你在 C# 裡怎麼讀 Assembly Version

---

## 今日 Checkpoint

完成下面的自我測驗，能回答就算過關：

1. `const x = { a: 1 }; x.a = 2;` 這樣合法嗎？為什麼？
2. `function foo(a: string, b?: number)` 中的 `?` 代表什麼？
3. TS 的 `interface` 和 C# 的 `interface` 最大的差異是什麼？
4. `extends` 在 TS 裡等於 C# 的什麼？
5. 為什麼 OpenClaw 裡很少看到型別標註？

---

## 答案

1. **合法**。`const` 只防重新賦值（`x = {}`），不防修改屬性。
2. **選擇性參數**，`b` 可以不傳，不傳時值為 `undefined`。
3. **結構型別 vs 名義型別**。TS 只看形狀是否匹配，C# 要顯式 `implements`。
4. `:` (冒號)。`class B extends A` → C# `class B : A`。
5. 因為 **型別推斷**。TypeScript 編譯器能從賦值自動推斷型別，等同 C# 的 `var`。
