# 方案 B：自建本地 Marketplace 接入 Roslyn Language Server

> **思路**：从零建一个本地 marketplace，里面挂一个自定义 `roslyn-ls` 插件，配置 `lspServers` 指向 `roslyn-language-server`。Claude Code 通过 `/plugin marketplace add /absolute/path` 注册。  
> **适用**：长期使用、稳定环境、需与官方 `csharp-lsp` 并存、希望完全可控。  
> **优势**：本地 marketplace 无 upstream，**不会被后台 autoUpdate 覆盖**。  
> **对比 [方案 A](./plan-a-shell-hack.md)**：步骤多 4–5 步，但稳定性显著优于壳 hack。

---

## 1. 前置条件

- Claude Code ≥ 2.0.74
- `.NET 10` SDK
- 路径里允许使用绝对路径（Windows、macOS、Linux 均可）

```bash
dotnet --version
claude --version
```

---

## 2. 步骤

### 2.1 启用 LSP Tool

`~/.claude/settings.json`（Windows：`%USERPROFILE%\.claude\settings.json`）：

```json
{
  "env": {
    "ENABLE_LSP_TOOL": "1"
  }
}
```

> 源码 `@/e:/claude-hub/claude-code/src/tools.ts:250` 明确：必须显式设置 truthy 值才注册 LSPTool；没有默认开启路径。

### 2.2 安装 Roslyn Language Server

```bash
dotnet tool install --global roslyn-language-server --prerelease
roslyn-language-server --version
```

二进制位置：

- **Windows**：`%USERPROFILE%\.dotnet\tools\roslyn-language-server.cmd`
- **macOS/Linux**：`~/.dotnet/tools/roslyn-language-server`

### 2.3 建本地 marketplace 目录骨架

```bash
# 任意位置，这里以 ~/.claude-custom-plugins 为例
mkdir -p ~/.claude-custom-plugins/.claude-plugin
mkdir -p ~/.claude-custom-plugins/plugins/roslyn-ls/.claude-plugin
```

最终结构：

```
~/.claude-custom-plugins/
├── .claude-plugin/
│   └── marketplace.json
└── plugins/
    └── roslyn-ls/
        ├── .claude-plugin/
        │   └── plugin.json
        └── .lsp.json
```

### 2.4 写 `marketplace.json`

路径：`~/.claude-custom-plugins/.claude-plugin/marketplace.json`

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "local-csharp",
  "metadata": {
    "description": "Local plugins for C# / Unity development.",
    "version": "1.0.0",
    "license": "MIT"
  },
  "owner": {
    "name": "Your Name"
  },
  "plugins": [
    {
      "name": "roslyn-ls",
      "version": "1.0.0",
      "source": "./plugins/roslyn-ls",
      "description": "Microsoft Roslyn Language Server via LSP.",
      "category": "development",
      "author": { "name": "Your Name" },
      "tags": ["csharp", "dotnet"],
      "strict": false,
      "lspServers": "./plugins/roslyn-ls/.lsp.json"
    }
  ]
}
```

**关键设计（基于源码校准）**：

- **`lspServers` 用字符串**指向 `.lsp.json` 相对路径——避免与 `.lsp.json` 内容重复注册（见 [§4.4 重复注册风险](#44-重复注册风险与避免方式)）
- **`source`** 路径相对于 marketplace 根（不是相对于 `marketplace.json`）
- **`strict: false`** 让 plugin.json 可省略，marketplace entry 即完整 manifest

### 2.5 写 plugin 的 `plugin.json`

路径：`~/.claude-custom-plugins/plugins/roslyn-ls/.claude-plugin/plugin.json`

```json
{
  "name": "roslyn-ls",
  "description": "Microsoft Roslyn Language Server via LSP.",
  "version": "1.0.0",
  "license": "MIT",
  "author": { "name": "Your Name" }
}
```

> `name`、`version` 必须与 marketplace.json 中对应 entry 一致；不一致时 `claude plugin validate` 会给警告（源码 `@/e:/claude-hub/claude-code/src/utils/plugins/validatePlugin.ts:478-485`）。

### 2.6 写 `.lsp.json`

路径：`~/.claude-custom-plugins/plugins/roslyn-ls/.lsp.json`

```json
{
  "roslyn": {
    "command": "C:/Users/YourName/.dotnet/tools/roslyn-language-server.cmd",
    "args": [
      "--stdio",
      "--autoLoadProjects",
      "--logLevel", "Information",
      "--extensionLogDirectory", "C:/Users/YourName/.roslyn-lsp/logs"
    ],
    "extensionToLanguage": {
      ".cs": "csharp"
    },
    "startupTimeout": 120000
  }
}
```

**说明**：

- 顶层 key `"roslyn"` 是 **LSP server name**（不是语言名）。任意字符串，路由用 `extensionToLanguage`，server name 与之无关——见 [§4.2 server name scoping](#42-server-name-scoping)
- **路径必须用正斜杠 `/`**，包括 Windows
- `--extensionLogDirectory` 指向的目录需要提前存在
- `startupTimeout`：Roslyn 首次索引大 sln 可能 1-2 分钟，120000 ms 比较保险
- Razor 文件（`.cshtml`、`.razor`）不要映射——独立 Roslyn LS 不带 Razor 服务

### 2.7 注册本地 marketplace

启动 Claude Code，在 REPL 里：

```
/plugin marketplace add /Users/YourName/.claude-custom-plugins
```

**Windows 注意**：路径不要用反斜杠，写成 `C:/Users/YourName/...` 或 `/Users/YourName/...` 都可以。

> 源码 `@/e:/claude-hub/claude-code/src/utils/plugins/marketplaceManager.ts:1782` 的 `addMarketplaceSource()` 支持本地路径作为 source，**没有 upstream**，因此**不会触发 autoUpdate 覆盖**。

### 2.8 安装插件

退出 Claude Code，终端：

```bash
claude plugin install -s user roslyn-ls
```

`-s user` 把 plugin 装在 user scope（不污染项目级）。

### 2.9 验证

重启 Claude Code：

```
/plugin
```

切到 **Installed**，应看到：

```
roslyn-ls   Plugin · local-csharp · ✔ enabled
```

让 Claude 跑 LSP 操作：

```
对 src/Program.cs 运行 LSP 的 documentSymbol
```

输出含 `● LSP(operation: "documentSymbol", ...)` 即成功。

---

## 3. 字段 schema 参考

完整 `LspServerConfigSchema`（`@/e:/claude-hub/claude-code/src/utils/plugins/schemas.ts:710-790`）：

```@e:/claude-hub/claude-code/src/utils/plugins/schemas.ts:711-790
  z.strictObject({
    command: z.string().min(1).refine(
      cmd => {
        if (cmd.includes(' ') && !cmd.startsWith('/')) {
          return false
        }
        return true
      },
      { message: 'Command should not contain spaces. Use args array for arguments.' },
    ),
    args: z.array(nonEmptyString()).optional(),
    extensionToLanguage: z.record(fileExtension(), nonEmptyString())
      .refine(record => Object.keys(record).length > 0, {
        message: 'extensionToLanguage must have at least one mapping',
      }),
    transport: z.enum(['stdio', 'socket']).default('stdio'),
    env: z.record(z.string(), z.string()).optional(),
    initializationOptions: z.unknown().optional(),
    settings: z.unknown().optional(),
    workspaceFolder: z.string().optional(),
    startupTimeout: z.number().int().positive().optional(),
    shutdownTimeout: z.number().int().positive().optional(),
    restartOnCrash: z.boolean().optional(),
    maxRestarts: z.number().int().nonnegative().optional(),
  })
```

`z.strictObject` 意味着多余字段会被拒绝——只用上表中字段。

`marketplace.json` 中 `lspServers` 字段的多种合法格式（`@/e:/claude-hub/claude-code/src/utils/plugins/schemas.ts:799-820`）：

```@e:/claude-hub/claude-code/src/utils/plugins/schemas.ts:799-820
const PluginManifestLspServerSchema = lazySchema(() =>
  z.object({
    lspServers: z.union([
      RelativeJSONPath().describe(
        'Path to .lsp.json configuration file relative to plugin root',
      ),
      z
        .record(z.string(), LspServerConfigSchema())
        .describe('LSP server configurations keyed by server name'),
      z
        .array(
          z.union([
            RelativeJSONPath().describe('Path to LSP configuration file'),
            z
              .record(z.string(), LspServerConfigSchema())
              .describe('Inline LSP server configurations'),
          ]),
        )
        .describe(
          'Array of LSP server configurations (paths or inline definitions)',
        ),
    ]),
  }),
)
```

三种形态：

1. **字符串**：相对路径指向 `.lsp.json` 文件
2. **对象**：inline 写 server 配置
3. **数组**：混合上面两种

本教程用形态 1（字符串路径）+ 独立 `.lsp.json` 文件，原因见 [§4.4](#44-重复注册风险与避免方式)。

---

## 4. 源码加载原理

### 4.1 LSP server 三个来源

源码 `@/e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts:57-122`：

```@e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts:57-122
export async function loadPluginLspServers(
  plugin: LoadedPlugin,
  errors: PluginError[] = [],
): Promise<Record<string, LspServerConfig> | undefined> {
  const servers: Record<string, LspServerConfig> = {}

  // 1. Check for .lsp.json file in plugin directory
  const lspJsonPath = join(plugin.path, '.lsp.json')
  try {
    const content = await readFile(lspJsonPath, 'utf-8')
    const parsed = jsonParse(content)
    const result = z
      .record(z.string(), LspServerConfigSchema())
      .safeParse(parsed)

    if (result.success) {
      Object.assign(servers, result.data)
    } else {
      ...
    }
  } catch (error) {
    // .lsp.json is optional, ignore if it doesn't exist
    ...
  }

  // 2. Check manifest.lspServers field
  if (plugin.manifest.lspServers) {
    const manifestServers = await loadLspServersFromManifest(
      plugin.manifest.lspServers,
      plugin.path,
      plugin.name,
      errors,
    )
    if (manifestServers) {
      Object.assign(servers, manifestServers)
    }
  }

  return Object.keys(servers).length > 0 ? servers : undefined
}
```

**三个来源**（按加载顺序）：

| # | 来源 | 路径/字段 | 本方案使用 |
|---|------|----------|----------|
| 1 | `.lsp.json` 文件 | `<plugin.path>/.lsp.json` | ✅ |
| 2 | manifest `lspServers` = 字符串路径 | 指向另一个 JSON 文件 | ✅（marketplace.json 写字符串路径） |
| 3 | manifest `lspServers` = inline 对象/数组 | 直接写在 manifest 里 | ❌ 不用 |

**注意合并语义**：`Object.assign(servers, ...)`——同名 server 后者覆盖前者；不同名 server 累加。

### 4.2 server name scoping

```@e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts:294-315
/**
 * Add plugin scope to LSP server configs
 * This adds a prefix to server names to avoid conflicts between plugins
 */
export function addPluginScopeToLspServers(
  servers: Record<string, LspServerConfig>,
  pluginName: string,
): Record<string, ScopedLspServerConfig> {
  const scopedServers: Record<string, ScopedLspServerConfig> = {}

  for (const [name, config] of Object.entries(servers)) {
    // Add plugin prefix to server name to avoid conflicts
    const scopedName = `plugin:${pluginName}:${name}`
    scopedServers[scopedName] = {
      ...config,
      scope: 'dynamic',
      source: pluginName,
    }
  }

  return scopedServers
}
```

本方案的 server 最终 key 是 `plugin:roslyn-ls:roslyn`。**两个不同 plugin 用同名 server 不冲突**（这就是 [§5.2 与 csharp-lsp 并存](#52-与官方-csharp-lsp-并存) 能 work 的原因）。

### 4.3 全局聚合与启动

```@e:/claude-hub/claude-code/src/services/lsp/config.ts:15-43
export async function getAllLspServers(): Promise<{
  servers: Record<string, ScopedLspServerConfig>
}> {
  const allServers: Record<string, ScopedLspServerConfig> = {}

  try {
    // Get all enabled plugins
    const { enabled: plugins } = await loadAllPluginsCacheOnly()

    // Load LSP servers from each plugin in parallel.
    const results = await Promise.all(
      plugins.map(async plugin => {
        const errors: PluginError[] = []
        try {
          const scopedServers = await getPluginLspServers(plugin, errors)
          return { plugin, scopedServers, errors }
        } catch (e) {
          ...
        }
      }),
    )

    for (const { plugin, scopedServers, errors } of results) {
      if (scopedServers && Object.keys(scopedServers).length > 0) {
        Object.assign(allServers, scopedServers)
      }
      ...
    }
```

并发遍历所有 enabled plugin，聚合 LSP servers。manager 初始化时机：

```@e:/claude-hub/claude-code/src/main.tsx:2853
      initializeLspServerManager();
```

在 trust dialog 之后（防止 untrusted 目录里的 plugin 启动陌生二进制）。

### 4.4 重复注册风险与避免方式

**典型坑**：marketplace.json 内联写 `lspServers`，**同时** plugin 根目录又放 `.lsp.json`，且两边 server name 不同 → 两个 server 都被**注册**，但运行时行为容易被误解。下面是精确的源码事实。

#### 4.4.1 注册阶段：两个 instance 对象都被创建

`@/e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts:73,117` 两个 `Object.assign(servers, ...)` 是累加语义；后续 `LSPServerManager` 给每个 server name 各创建一个 `LSPServerInstance`：

```@e:/claude-hub/claude-code/src/services/lsp/LSPServerManager.ts:121-123
        // Create server instance
        const instance = createLSPServerInstance(serverName, config)
        servers.set(serverName, instance)
```

#### 4.4.2 启动阶段：**lazy spawn**，只 spawn 第一个

`createLSPServerInstance` **只创建对象，不 spawn 子进程**。真正的 `spawn` 在 `@/e:/claude-hub/claude-code/src/services/lsp/LSPClient.ts:98`，由 `instance.start()` 触发；而 `start()` 的唯一触发点是 `ensureServerStarted(filePath)`：

```@e:/claude-hub/claude-code/src/services/lsp/LSPServerManager.ts:217-235
  async function ensureServerStarted(
    filePath: string,
  ): Promise<LSPServerInstance | undefined> {
    const server = getServerForFile(filePath)
    if (!server) return undefined

    if (server.state === 'stopped' || server.state === 'error') {
      try {
        await server.start()
```

`getServerForFile` 的路由规则是 **first registered wins**：

```@e:/claude-hub/claude-code/src/services/lsp/LSPServerManager.ts:189-208
  /**
   * Get the LSP server instance for a given file path.
   * If multiple servers handle the same extension, returns the first registered server.
   * Returns undefined if no server handles this file type.
   */
  function getServerForFile(filePath: string): LSPServerInstance | undefined {
    const ext = path.extname(filePath).toLowerCase()
    const serverNames = extensionMap.get(ext)

    if (!serverNames || serverNames.length === 0) {
      return undefined
    }

    // Use first server (can add priority later)
    const serverName = serverNames[0]
```

由于 `.lsp.json` 在 `loadPluginLspServers` 里**先**于 `manifest.lspServers` 合入，**`.lsp.json` 那份赢**：

- ✅ `.lsp.json` 配置的 server 被 spawn 一份 Roslyn LS 进程
- ❌ `marketplace.json` 配置的 server **永远 `state === 'stopped'`**，**不消耗 Roslyn LS 进程**

#### 4.4.3 真实代价（不是双倍内存）

| 代价 | 严重度 |
|------|------|
| 多一个 idle `LSPServerInstance` 对象 | 🟢 低（轻量 JS 对象） |
| 路由依赖 Object 迭代顺序（ES2015+ 稳定但隐式依赖） | 🟡 中 |
| 维护两份易漂移：改一边忘改另一边 → 重启后切到旧配置 | 🟡 中 |
| `isLspConnected()` 仍返 true（只要有一个非 error 即 true） → LSPTool 被注册但实际跑错配置 | 🟡 中 |

源码：

```@e:/claude-hub/claude-code/src/services/lsp/manager.ts:100-110
export function isLspConnected(): boolean {
  if (initializationState === 'failed') return false
  const manager = getLspServerManager()
  if (!manager) return false
  const servers = manager.getAllServers()
  if (servers.size === 0) return false
  for (const server of servers.values()) {
    if (server.state !== 'error') return true
  }
  return false
}
```

**结论**：两套不会"OOM 双倍"，但仍然是**坏实践**——配置漂移 + 隐式路由依赖 + 排错难度翻倍。

#### 4.4.4 避免方式（本方案做法）

- marketplace.json 里 `"lspServers": "./plugins/roslyn-ls/.lsp.json"`——**字符串路径形态**
- 真实配置只在 `.lsp.json` 一处

§4.1 步骤 1 读 `.lsp.json` 内容 → 步骤 2 读 manifest.lspServers 路径 → **路径指向同一个文件 → 两次读到的是相同内容 → `Object.assign` 同名覆盖等于不变**，结果幂等。

> 也可以反过来：marketplace.json 内联完整 `lspServers` 对象，**不放** `.lsp.json` 文件，结果一样幂等。**不要两边都写不同的配置**。

### 4.5 plugin manifest 与 marketplace entry 合并

```@e:/claude-hub/claude-code/src/utils/plugins/schemas.ts:1244-1260
/**
 * Schema for individual plugin entries in a marketplace
 *
 * When strict=true (default): Plugin.json is required, marketplace fields supplement it
 * When strict=false: Plugin.json is optional, marketplace provides full manifest
 *
 * Unknown fields are silently stripped (zod default) rather than rejected.
 * Marketplace entries are validated as an array — if one entry rejected
 * unknown keys, the whole marketplace.json would fail to parse and ALL
 * plugins from that marketplace would become unavailable. ...
 */
export const PluginMarketplaceEntrySchema = lazySchema(() =>
  PluginManifestSchema()
    .partial()
    .extend({
```

本方案用 `strict: false`，意味着 marketplace entry 自身就能提供 manifest 全部内容（连同 `lspServers`），不强依赖 `plugin.json`——`plugin.json` 写出来只是为了让 `claude plugin validate` 满意。

### 4.6 LSP 不受 marketplace autoUpdate 覆盖

```@e:/claude-hub/claude-code/src/utils/plugins/pluginAutoupdate.ts:213-225
/**
 * Auto-update marketplaces and plugins in the background.
 * ...
 * Official Anthropic marketplaces have autoUpdate enabled by default,
 * but users can disable it per-marketplace in the UI.
 * ...
 */
```

本地路径 source 的 marketplace **没有 upstream**，autoUpdate 对它是 no-op。这是方案 B 相比方案 A 最大的稳定性优势。

### 4.7 LSP 不在 bare/headless 模式下启用

```@e:/claude-hub/claude-code/src/services/lsp/manager.ts:145-150
export function initializeLspServerManager(): void {
  // --bare / SIMPLE: no LSP. LSP is for editor integration (diagnostics,
  // hover, go-to-def in the REPL). Scripted -p calls have no use for it.
  if (isBareMode()) {
    return
  }
```

`claude -p "..."` 这种非交互调用**完全不启** LSP。LSPTool 只在 REPL 交互模式下可用。

---

## 5. 进阶

### 5.1 指定 Solution

多 sln 项目用 `--solutionPath`：

```json
{
  "roslyn": {
    "command": "C:/Users/YourName/.dotnet/tools/roslyn-language-server.cmd",
    "args": [
      "--stdio",
      "--solutionPath", "./Backend.sln",
      "--logLevel", "Information"
    ],
    "extensionToLanguage": { ".cs": "csharp" }
  }
}
```

`--solutionPath` 显式指定时可省 `--autoLoadProjects`（Roslyn LS 自动加载指定 sln 的项目）。

### 5.2 与官方 csharp-lsp 并存

方案 B 完全可以与官方 `csharp-lsp` 共存：

- 官方 `csharp-lsp` 的 server scope key：`plugin:csharp-lsp:csharp-ls`
- 本方案 `roslyn-ls` 的 server scope key：`plugin:roslyn-ls:roslyn`

**但需要避免 extension 冲突**：两个 server 都 map `.cs` → `csharp` 会导致 LSPServerManager 同 extension 多 server 竞争。**实践做法**：

- 只在需要 Roslyn 时 enable `roslyn-ls`，disable `csharp-lsp`
- 或者反过来

`/plugin` UI 切换 enabled 状态即可，无需重装。

### 5.3 CSharpLspAdapter 集成

社区反馈 Roslyn LS 在 Claude Code 下 `hover`/`goToDefinition` 可能挂起，原因是 Claude Code LSP 客户端缺失对部分协议方法的响应。**精确事实**：

| 协议方法 | Claude Code 内置响应 | 需要 adapter 补 |
|---------|-------------------|----------------|
| `workspace/configuration` | ✅ 返回 `null` 数组（见下） | ❌ 不需要 |
| `client/registerCapability` | ❌ 缺失 | ✅ 需要 |
| `window/workDoneProgress/create` | ❌ 缺失 | ✅ 需要 |

`workspace/configuration` 由 `LSPServerManager` 在每个 server 注册时挂上 handler：

```@e:/claude-hub/claude-code/src/services/lsp/LSPServerManager.ts:125-137
        // Register handler for workspace/configuration requests from the server
        // Some servers (like TypeScript) send these even when we say we don't support them
        instance.onRequest(
          'workspace/configuration',
          (params: { items: Array<{ section?: string }> }) => {
            logForDebugging(
              `LSP: Received workspace/configuration request from ${serverName}`,
            )
            // Return empty/null config for each requested item
            // This satisfies the protocol without providing actual configuration
            return params.items.map(() => null)
          },
        )
```

所以 [CSharpLspAdapter](https://github.com/Agasper/CSharpLspAdapter) 的真正价值是**补另外两个协议方法**——它仍然有用（修复 `hover` / `findReferences` 挂起），但描述要准确。

安装：

```bash
dotnet tool install --global CSharpLspAdapter
```

修改 `.lsp.json`，把 `command` 指向适配器：

```json
{
  "roslyn": {
    "command": "csharp-ls-adapter",
    "extensionToLanguage": { ".cs": "csharp" },
    "env": {
      "LSP_SOLUTION_PATH": "/path/to/Solution.sln",
      "LSP_ADAPTER_DEBUG": "1"
    }
  }
}
```

`env` 字段由 schema 的 `z.record(z.string(), z.string()).optional()` 接受（`@/e:/claude-hub/claude-code/src/utils/plugins/schemas.ts:747-750`），无需额外配置。

### 5.4 使用变量替代硬编码路径

把 `.lsp.json` 里的绝对路径换成变量，配置可移植（跨平台、可随 plugin 目录搬家）。

源码 `@/e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts:229-291` 的 `resolvePluginLspEnvironment` 会对 `command`、`args`、`env`、`workspaceFolder` 字段做变量替换：

```@e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts:255-275
  // Resolve command path
  if (resolved.command) {
    resolved.command = resolveValue(resolved.command)
  }

  // Resolve args
  if (resolved.args) {
    resolved.args = resolved.args.map((arg: string) => resolveValue(arg))
  }

  // Resolve environment variables and add CLAUDE_PLUGIN_ROOT / CLAUDE_PLUGIN_DATA
  const resolvedEnv: Record<string, string> = {
    CLAUDE_PLUGIN_ROOT: plugin.path,
    CLAUDE_PLUGIN_DATA: getPluginDataDir(plugin.source),
    ...(resolved.env || {}),
  }
```

支持的变量类型：

| 变量 | 含义 | 例 |
|------|------|----|
| `${CLAUDE_PLUGIN_ROOT}` | 当前 plugin 目录的绝对路径（自动注入） | `~/.claude-custom-plugins/plugins/roslyn-ls` |
| `${CLAUDE_PLUGIN_DATA}` | plugin 持久化数据目录（自动按 source 隔离） | `~/.claude/plugins/data/<source>` |
| `${ENV_VAR}` | 父进程环境变量（包括 `HOME`、`USERPROFILE` 等） | `${HOME}/.dotnet/tools/...` |
| `${user_config.KEY}` | 来自 plugin manifest `userConfig` 的用户设置 | 需先在 plugin.json 声明 |

**可移植版 `.lsp.json` 示例**：

```json
{
  "roslyn": {
    "command": "${HOME}/.dotnet/tools/roslyn-language-server",
    "args": [
      "--stdio",
      "--autoLoadProjects",
      "--logLevel", "Information",
      "--extensionLogDirectory", "${CLAUDE_PLUGIN_DATA}/logs"
    ],
    "extensionToLanguage": { ".cs": "csharp" },
    "startupTimeout": 120000
  }
}
```

**Windows 注意**：`${HOME}` 在 Windows PowerShell 下通常未设，用 `${USERPROFILE}`；或用 `${CLAUDE_PLUGIN_ROOT}` 引相对路径。

**好处**：

- plugin 目录可以移动 / 跨机同步（路径不绑死）
- 一份 `.lsp.json` 跨 Windows / macOS / Linux 通用
- 日志默认进 `${CLAUDE_PLUGIN_DATA}/logs`，按 plugin 自动隔离（`CLAUDE_PLUGIN_DATA` 由 Claude Code 创建；子目录 `logs` 视 Roslyn LS 版本，必要时仍需手动创建）

**未解析变量的行为**：源码 `@/e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts:283-289` 会把缺失变量收集后输出 warn 日志，但**不会阻止启动**——server 拿到未解析的 `${X}` 字面量当路径，启动时报 ENOENT。

---

## 6. 风险与缓解

| 风险 | 严重度 | 缓解 |
|------|-------|------|
| 路径用反斜杠 → 解析失败 | 🟡 中 | 全部用 `/` |
| `.lsp.json` 和 marketplace.json 同时配置不同 server | 🔴 高 | 二选一（[§4.4](#44-重复注册风险与避免方式)） |
| plugin entry `name` 与 plugin.json 不一致 | 🟢 低 | `claude plugin validate` 会警告 |
| Roslyn LS 客户端协议挂起 | 🟡 中 | 用 CSharpLspAdapter（[§5.3](#53-csharplspadapter-集成)） |
| Unity net471 csproj 加载失败 | 🟡 中 | 装 VS Build Tools / .NET FX 4.7.1 Dev Pack |
| 大型 sln 内存爆炸 | 🟡 中 | 用 `--solutionPath` 限范围 |
| headless `claude -p` 调 LSP 失败 | 🟢 低 | LSP 仅 REPL 可用（[§4.7](#47-lsp-不在-bareheadless-模式下启用)） |

---

## 7. 排错清单

```powershell
# 1. 二进制可执行
roslyn-language-server --version

# 2. ENABLE_LSP_TOOL
$env:ENABLE_LSP_TOOL

# 3. marketplace.json 路径正确
Get-Content $env:USERPROFILE\.claude-custom-plugins\.claude-plugin\marketplace.json

# 4. plugin.json 与 .lsp.json 存在
Get-ChildItem $env:USERPROFILE\.claude-custom-plugins\plugins\roslyn-ls -Recurse

# 5. 已注册到 Claude Code
claude plugin marketplace list

# 6. plugin 已安装且 enabled
claude plugin list

# 7. LSP server 状态
# Claude Code 内 /plugin → Errors 标签看有无错误
# ~/.claude/debug/latest 搜 "LSP" / "roslyn"

# 8. Roslyn LS 自身日志
# 看 --extensionLogDirectory 指定目录
```

如 `documentSymbol` 工作但 `hover`/`goToDefinition` 挂起 → 用 CSharpLspAdapter（[§5.3](#53-csharplspadapter-集成)）。

---

## 8. 参考

- 源码核心位置：
  - `@/e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts:57-122` — `loadPluginLspServers` 三来源
  - `@/e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts:294-315` — server name scoping
  - `@/e:/claude-hub/claude-code/src/utils/plugins/schemas.ts:710-790` — LspServerConfigSchema
  - `@/e:/claude-hub/claude-code/src/utils/plugins/schemas.ts:799-820` — manifest lspServers union
  - `@/e:/claude-hub/claude-code/src/utils/plugins/schemas.ts:1244-1260` — marketplace entry schema
  - `@/e:/claude-hub/claude-code/src/services/lsp/config.ts:15-43` — 全局聚合
  - `@/e:/claude-hub/claude-code/src/services/lsp/manager.ts:145-150` — bare 模式判断
  - `@/e:/claude-hub/claude-code/src/tools.ts:250` — `ENABLE_LSP_TOOL` 注册门槛
  - `@/e:/claude-hub/claude-code/src/utils/plugins/marketplaceManager.ts:1782` — 本地 marketplace 注册
- 上游 issue / 资源：
  - https://gist.github.com/jrusbatch/1d2c539ef17476c8703f04a2e9148693 — 本方案原始参考
  - https://github.com/Agasper/CSharpLspAdapter — LSP 协议适配器
  - https://github.com/anthropics/claude-code/issues/16360 — Claude Code LSP 客户端缺失协议方法
  - https://github.com/anthropics/claude-plugins-official/issues/463 — 请求官方 roslyn-lsp 插件
  - https://www.nuget.org/packages/roslyn-language-server — 二进制下载
- 相邻方案：[`plan-a-shell-hack.md`](./plan-a-shell-hack.md)
- 本仓库 LSP 集成机制：`@/e:/claude-hub/claude-code/docs/lsp-integration.md`
