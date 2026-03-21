# Day 11：Plugin 系統

> **目標**：理解 Plugin 的載入、註冊、Discovery、Manifest、Hook、Slot
> **C# 對照**：MEF2 (System.Composition)、Assembly.LoadFrom、IServiceCollection
> **預計時間**：4 小時

---

## 11.1 Plugin 系統全景

```
src/plugins/
  ├── registry.ts       ← DI Container（Plugin 註冊中心）
  ├── loader.ts         ← Assembly.LoadFrom（動態載入）
  ├── discovery.ts      ← Assembly Scanning（自動發現 plugin）
  ├── manifest.ts       ← NuGet .nuspec（Plugin 描述檔）
  ├── hooks.ts          ← Middleware Pipeline（Hook 系統）
  ├── slots.ts          ← Service Slot（依賴注入插槽）
  ├── tools.ts          ← Plugin 提供的 AI Tools
  ├── services.ts       ← Plugin 提供的背景服務
  ├── types.ts          ← Plugin 型別定義
  ├── config-schema.ts  ← Plugin 設定 schema
  └── runtime/          ← Plugin 執行時環境
```

---

## 11.2 registry.ts — Plugin 註冊中心

```typescript
// src/plugins/registry.ts
class PluginRegistry {
  // 所有已註冊的 plugin
  private plugins: Map<string, RegisteredPlugin> = new Map();

  // Channel plugins
  channels: ChannelPluginEntry[] = [];

  // Tool plugins
  tools: ToolPluginEntry[] = [];

  // Service plugins
  services: ServicePluginEntry[] = [];

  // 註冊一個 plugin
  register(plugin: PluginDefinition) {
    this.plugins.set(plugin.id, {
      plugin,
      status: "registered",
    });

    // 按類型分類
    if (plugin.type === "channel") {
      this.channels.push({ plugin: plugin as ChannelPlugin });
    }
    if (plugin.tools) {
      this.tools.push({ plugin, tools: plugin.tools });
    }
  }

  // 查詢
  get(id: string): RegisteredPlugin | undefined {
    return this.plugins.get(id);
  }
}
```

```csharp
// C# 等價
public class PluginRegistry
{
    private readonly Dictionary<string, RegisteredPlugin> _plugins = new();
    public IReadOnlyList<ChannelPluginEntry> Channels => _channels;
    public IReadOnlyList<ToolPluginEntry> Tools => _tools;

    public void Register(IPluginDefinition plugin)
    {
        _plugins[plugin.Id] = new RegisteredPlugin(plugin);

        if (plugin is IChannelPlugin channel)
            _channels.Add(new ChannelPluginEntry(channel));

        if (plugin.Tools?.Any() == true)
            _tools.Add(new ToolPluginEntry(plugin, plugin.Tools));
    }
}
```

---

## 11.3 loader.ts — 動態載入

```typescript
// src/plugins/loader.ts

async function loadPlugin(pluginPath: string): Promise<PluginDefinition> {
  // 動態 import（runtime 載入模組）
  const module = await import(pluginPath);

  // Plugin 必須 export 一個 default 或 createPlugin 函式
  const plugin = module.default ?? module.createPlugin?.();

  if (!plugin || !plugin.id) {
    throw new Error(`Invalid plugin at ${pluginPath}`);
  }

  return plugin;
}
```

```csharp
// C# 等價
public async Task<IPluginDefinition> LoadPluginAsync(string dllPath)
{
    // 載入 Assembly
    var assembly = Assembly.LoadFrom(dllPath);

    // 用 MEF2 發現 Export
    var composition = new ContainerConfiguration()
        .WithAssembly(assembly)
        .CreateContainer();

    return composition.GetExport<IPluginDefinition>();
}
```

**映射**：TS 的 `import(path)` = C# 的 `Assembly.LoadFrom(path)` + 反射取得型別。

---

## 11.4 discovery.ts — 自動發現

```typescript
// src/plugins/discovery.ts

async function discoverPlugins(searchPaths: string[]): Promise<string[]> {
  const pluginPaths: string[] = [];

  for (const searchPath of searchPaths) {
    // 掃描目錄下所有有 package.json 的子目錄
    const entries = await readdir(searchPath);

    for (const entry of entries) {
      const pkgPath = path.join(searchPath, entry, "package.json");
      if (await fileExists(pkgPath)) {
        const pkg = await readJson(pkgPath);

        // 檢查是否有 openclaw plugin 標記
        if (pkg.openclaw?.plugin) {
          pluginPaths.push(path.join(searchPath, entry));
        }
      }
    }
  }

  return pluginPaths;
}
```

```csharp
// C# 等價
public IEnumerable<string> DiscoverPlugins(string[] searchPaths)
{
    foreach (var searchPath in searchPaths)
    {
        foreach (var dir in Directory.GetDirectories(searchPath))
        {
            var dllPath = Path.Combine(dir, "plugin.dll");
            if (File.Exists(dllPath))
                yield return dllPath;
        }
    }
}
```

---

## 11.5 manifest.ts — Plugin 描述檔

```typescript
// Plugin 的 package.json 裡定義 manifest
{
  "name": "@openclaw/extension-matrix",
  "version": "1.0.0",
  "openclaw": {
    "plugin": {
      "id": "matrix",
      "type": "channel",
      "label": "Matrix",
      "description": "Matrix chat protocol integration",
      "configSchema": { ... }
    }
  }
}
```

```csharp
// C# 等價：類似 NuGet .nuspec 或 AssemblyAttribute
[assembly: OpenClawPlugin(
    Id = "matrix",
    Type = PluginType.Channel,
    Label = "Matrix",
    Description = "Matrix chat protocol integration")]
```

---

## 11.6 hooks.ts — Hook 系統

Hook 讓 plugin 能在特定生命週期事件中插入邏輯。

```typescript
// src/plugins/hooks.ts

type HookPoint =
  | "before:chat"        // 聊天處理前
  | "after:chat"         // 聊天處理後
  | "before:tool_call"   // 工具呼叫前
  | "after:tool_call"    // 工具呼叫後
  | "on:message"         // 收到訊息時
  | "on:startup"         // Gateway 啟動時
  | "on:shutdown";       // Gateway 關閉時

class HookRunner {
  private hooks: Map<HookPoint, HookHandler[]> = new Map();

  register(point: HookPoint, handler: HookHandler) {
    const existing = this.hooks.get(point) ?? [];
    existing.push(handler);
    this.hooks.set(point, existing);
  }

  async run(point: HookPoint, context: HookContext) {
    const handlers = this.hooks.get(point) ?? [];
    for (const handler of handlers) {
      await handler(context);
      if (context.cancelled) break;  // 允許 hook 取消後續處理
    }
  }
}
```

```csharp
// C# 等價：ASP.NET Core Middleware 或 MediatR Pipeline
public class HookRunner
{
    private readonly Dictionary<HookPoint, List<IHookHandler>> _hooks = new();

    public void Register(HookPoint point, IHookHandler handler)
        => (_hooks[point] ??= []).Add(handler);

    public async Task RunAsync(HookPoint point, HookContext context)
    {
        foreach (var handler in _hooks.GetValueOrDefault(point, []))
        {
            await handler.HandleAsync(context);
            if (context.IsCancelled) break;
        }
    }
}
```

---

## 11.7 slots.ts — 服務插槽

Slot 允許 plugin 替換核心的預設行為。

```typescript
// src/plugins/slots.ts

// 定義可替換的 slot
type SlotId =
  | "embedding-provider"   // Embedding 提供者
  | "tts-engine"           // TTS 語音引擎
  | "browser-engine"       // 瀏覽器引擎
  | "memory-backend";      // 記憶儲存後端

class SlotRegistry {
  private slots: Map<SlotId, unknown> = new Map();

  provide<T>(slotId: SlotId, implementation: T) {
    this.slots.set(slotId, implementation);
  }

  resolve<T>(slotId: SlotId): T | undefined {
    return this.slots.get(slotId) as T | undefined;
  }
}
```

```csharp
// C# 等價：就是 DI 的 Replace
services.AddSingleton<IEmbeddingProvider, OpenAIEmbeddingProvider>();
// Plugin 可以替換：
services.Replace(ServiceDescriptor.Singleton<IEmbeddingProvider, CustomEmbeddingProvider>());
```

---

## 今日閱讀作業

### 作業 1：閱讀 `src/plugins/registry.ts`
### 作業 2：閱讀 `src/plugins/loader.ts`
### 作業 3：閱讀 `src/plugins/discovery.ts`
### 作業 4：挑一個 extension（如 `extensions/matrix/`），看它怎麼定義 plugin

---

## 今日 Checkpoint

1. PluginRegistry 對應 C# 的什麼概念？
2. 動態 `import()` 對應 C# 的什麼？
3. Hook 系統和 ASP.NET Middleware 有什麼相似之處？
4. Slot 系統解決什麼問題？
5. Plugin 怎麼被自動發現？

---

## 答案

1. **DI Container** (IServiceProvider) — 集中管理所有 plugin 的註冊和查詢。
2. **Assembly.LoadFrom()** + 反射 — 在執行時動態載入模組。
3. 都是 **Pipeline Pattern**：按順序執行 handler，每個 handler 可以修改 context 或取消後續執行。
4. 允許 plugin **替換核心的預設實作**。例如用自訂的 Embedding Provider 替換預設的 OpenAI。
5. 掃描 `extensions/` 目錄下所有子目錄的 `package.json`，找到含有 `openclaw.plugin` 欄位的。
