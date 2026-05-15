# 方案 A：复用官方 csharp-lsp 插件壳接入 Roslyn Language Server

> **思路**：装官方 `csharp-lsp` 插件作为容器，然后编辑本地缓存的 `marketplace.json` 把 `lspServers.<key>.command` 指向 `roslyn-language-server`。  
> **适用**：一次性试跑 / Demo / 不想自建 marketplace。  
> **不适用**：长期使用、稳定环境——本方案是 hack，**后台 autoUpdate 会覆盖**（见 [§5 风险](#5-风险与缓解)）。  
> **推荐替代**：生产环境用 [`plan-b-local-marketplace.md`](./plan-b-local-marketplace.md)。

---

## 1. 前置条件

- Claude Code ≥ 2.0.74
- `.NET 10` SDK（roslyn-language-server `--prerelease` 需要 .NET 10，.NET 9 装会报 `DotnetToolSettings.xml` 找不到）
- 已安装并启用过官方 `csharp-lsp` 插件（本地缓存目录已生成）

```bash
dotnet --version          # 检查 .NET 10
claude --version          # 检查 Claude Code
```

---

## 2. 步骤

### 2.1 启用 LSP 工具

在 `~/.claude/settings.json`（Windows：`%USERPROFILE%\.claude\settings.json`）添加：

```json
{
  "env": {
    "ENABLE_LSP_TOOL": "1"
  }
}
```

> **重要**：必须显式设置。源码层面没有"默认开启"路径——见 [§4.1 LSP Tool 注册门槛](#41-lsp-tool-注册门槛)。

### 2.2 安装 Roslyn Language Server

```bash
dotnet tool install --global roslyn-language-server --prerelease
roslyn-language-server --version   # 验证
```

可执行文件位置：

- **Windows**：`%USERPROFILE%\.dotnet\tools\roslyn-language-server.cmd`
- **macOS/Linux**：`~/.dotnet/tools/roslyn-language-server`

### 2.3 安装官方 csharp-lsp 插件作为壳

```bash
claude plugin marketplace update claude-plugins-official
claude plugin install csharp-lsp
```

本地 marketplace 缓存生成在：

- **Windows**：`%USERPROFILE%\.claude\plugins\marketplaces\claude-plugins-official\.claude-plugin\marketplace.json`
- **macOS/Linux**：`~/.claude/plugins/marketplaces/claude-plugins-official/.claude-plugin/marketplace.json`

### 2.4 ⚠️ 关闭该 marketplace 的 autoUpdate（教程版本通常漏掉）

在 Claude Code 里：

```
/plugin
```

进入 **Marketplaces** 标签，找到 `claude-plugins-official`，**关闭 autoUpdate**。

源码层面，官方 marketplace **默认 autoUpdate=true**：

```@e:/claude-hub/claude-code/src/utils/plugins/pluginAutoupdate.ts:1-11
/**
 * Background plugin autoupdate functionality
 *
 * At startup, this module:
 * 1. First updates marketplaces that have autoUpdate enabled
 * 2. Then checks all installed plugins from those marketplaces and updates them
 *
 * Updates are non-inplace (disk-only), requiring a restart to take effect.
 * Official Anthropic marketplaces have autoUpdate enabled by default,
 * but users can disable it per-marketplace.
 */
```

**不关掉的话每次启动 Claude Code 都会 git pull 覆盖你的修改**。

### 2.5 编辑本地 `marketplace.json` 把 `command` 改成 Roslyn LS

打开本地缓存的 `marketplace.json`，找到 `"name": "csharp-lsp"` 对应的 plugin entry，**整段替换**为：

```json
{
  "name": "csharp-lsp",
  "description": "C# language server (now backed by roslyn-language-server)",
  "version": "1.0.0",
  "author": { "name": "Anthropic", "email": "support@anthropic.com" },
  "source": "./plugins/csharp-lsp",
  "category": "development",
  "strict": false,
  "lspServers": {
    "roslyn": {
      "command": "roslyn-language-server.cmd",
      "args": ["--stdio", "--autoLoadProjects"],
      "extensionToLanguage": {
        ".cs": "csharp"
      }
    }
  }
}
```

**说明（已根据源码校准）**：

| 字段 | 必须值 | 可改否 | 备注 |
|------|------|-------|------|
| `name` | `csharp-lsp` | ❌ | plugin name，必须与 pluginId 对应 |
| `source` | `./plugins/csharp-lsp` | ❌ | 指向 plugin 目录 |
| `lspServers.<key>` | 任意 | ✅ | **server name，可随便改**（路由用 extensionToLanguage） |
| `command` | 绝对/PATH 中可见 | ✅ | Windows `.cmd`，*nix 无后缀 |
| `args` | 任意 | ✅ | 透传给二进制 |
| `extensionToLanguage` | 必填 | ✅ | 至少 1 项 |

> 之前流传的"`csharp-ls` 这个 key 不要改"是误传——见 [§4.4 server name 不影响路由](#44-server-name-不影响路由) 源码证据。

**Windows 注意**：若 `roslyn-language-server.cmd` 不在 `PATH`，需写绝对路径，**且必须用正斜杠**：

```json
"command": "C:/Users/YourName/.dotnet/tools/roslyn-language-server.cmd"
```

### 2.6 重启验证

```
/plugin
```

`csharp-lsp` 仍显示 enabled。让 Claude 对一个 `.cs` 文件跑 `documentSymbol`：

```
对 src/Program.cs 运行 LSP 的 documentSymbol
```

输出应有 `● LSP(operation: "documentSymbol", ...)`。

---

## 3. 字段 schema 参考

完整 `LspServerConfigSchema` 在 `@/e:/claude-hub/claude-code/src/utils/plugins/schemas.ts:710-790`：

```@e:/claude-hub/claude-code/src/utils/plugins/schemas.ts:711-790
  z.strictObject({
    command: z.string().min(1).refine(...),
    args: z.array(nonEmptyString()).optional(),
    extensionToLanguage: z.record(fileExtension(), nonEmptyString())
      .refine(record => Object.keys(record).length > 0, {...}),
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

注意 `z.strictObject` ——**多余字段会被拒绝**，schema 之外的字段不要写。

---

## 4. 源码加载原理

理解这个原理就能彻底想通"为什么改 marketplace.json 能改变启动的二进制"。

### 4.1 LSP Tool 注册门槛

LSPTool 是否对模型可见的双门槛：

**门槛 1 — 环境变量**：

```@e:/claude-hub/claude-code/src/tools.ts:250
    ...(isEnvTruthy(process.env.ENABLE_LSP_TOOL) ? [LSPTool] : []),
```

没设 `ENABLE_LSP_TOOL=1` 直接不注册。

**门槛 2 — LSP 服务器健康**：

```@e:/claude-hub/claude-code/packages/builtin-tools/src/tools/LSPTool/LSPTool.ts:137-139
  isEnabled() {
    return isLspConnected()
  },
```

`isLspConnected()` 检查至少有一个非 error 状态的 server：

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

**额外屏蔽**：`--bare` / `claude -p` headless 不启 LSP：

```@e:/claude-hub/claude-code/src/services/lsp/manager.ts:145-150
export function initializeLspServerManager(): void {
  // --bare / SIMPLE: no LSP. LSP is for editor integration (diagnostics,
  // hover, go-to-def in the REPL). Scripted -p calls have no use for it.
  if (isBareMode()) {
    return
  }
```

### 4.2 marketplace.json → plugin manifest 的合并机制

**关键**：`PluginMarketplaceEntrySchema` 继承自 `PluginManifestSchema`：

```@e:/claude-hub/claude-code/src/utils/plugins/schemas.ts:1244-1260
/**
 * Schema for individual plugin entries in a marketplace
 *
 * When strict=true (default): Plugin.json is required, marketplace fields supplement it
 * When strict=false: Plugin.json is optional, marketplace provides full manifest
 *
 * Unknown fields are silently stripped (zod default) rather than rejected.
 * ...
 */
export const PluginMarketplaceEntrySchema = lazySchema(() =>
  PluginManifestSchema()
    .partial()
    .extend({
```

意味着 **marketplace.json 中的 plugin entry 可以直接写 `lspServers`**，运行时会被合并到 plugin manifest 上——这是方案 A 能 work 的关键。

> `strict: false` 让 plugin.json 不是必需，marketplace entry 自身就能提供完整 manifest（包括 lspServers）。

### 4.3 LSP server 配置加载链路

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
    }
    ...
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

**方案 A 的 `lspServers` 写在 marketplace.json 里** → 通过 §4.2 的 supplement 机制变成 `plugin.manifest.lspServers` → 进入第 2 个分支被加载。

### 4.4 server name 不影响路由

加载完后，server name 被加 plugin scope 前缀：

```@e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts:298-315
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
      scope: 'dynamic', // Use dynamic scope for plugin servers
      source: pluginName,
    }
  }

  return scopedServers
}
```

**结论**：

- 你写 `"roslyn": {...}` → 最终 key 是 `plugin:csharp-lsp:roslyn`
- 你写 `"csharp-ls": {...}` → 最终 key 是 `plugin:csharp-lsp:csharp-ls`
- 都能跑，**路由用 `extensionToLanguage`，server 内部名字与之无关**

### 4.5 启动流程

LSP server manager 在 Claude Code 启动时初始化（trust dialog 后）：

```@e:/claude-hub/claude-code/src/main.tsx:2853
      initializeLspServerManager();
```

`initialize()` 调用 `getAllLspServers()`：

```@e:/claude-hub/claude-code/src/services/lsp/config.ts:15-43
export async function getAllLspServers(): Promise<{
  servers: Record<string, ScopedLspServerConfig>
}> {
  const allServers: Record<string, ScopedLspServerConfig> = {}
  try {
    const { enabled: plugins } = await loadAllPluginsCacheOnly()
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
    ...
```

聚合所有 enabled plugin 的 LSP server，注入 LSPServerManager，按 `extensionToLanguage` 建立路由表。**首次打开 `.cs` 文件时**，按扩展名 `.cs` 找到对应 server，启动 `command` 指向的二进制（即 `roslyn-language-server.cmd`），通过 stdio 通信。

### 4.6 后台 autoUpdate 覆盖机制（方案 A 的致命点）

```@e:/claude-hub/claude-code/src/utils/plugins/pluginAutoupdate.ts:213-225
/**
 * Auto-update marketplaces and plugins in the background.
 *
 * This function:
 * 1. Checks which marketplaces have autoUpdate enabled
 * 2. Refreshes only those marketplaces (git pull/re-download)
 * 3. Updates installed plugins from those marketplaces
 * 4. If any plugins were updated, notifies via the registered callback
 *
 * Official Anthropic marketplaces have autoUpdate enabled by default,
 * but users can disable it per-marketplace in the UI.
 *
 * This function runs silently without blocking user interaction.
 * Called from main.tsx during startup as a background job.
 */
```

**每次启动 Claude Code 都会跑这个 background job**，对 autoUpdate 开启的 marketplace（默认包括 `claude-plugins-official`）执行 git pull，把你改过的 marketplace.json **覆盖回上游版本**。这就是为什么 [§2.4](#24-️-关闭该-marketplace-的-autoupdate教程版本通常漏掉) 必须先关 autoUpdate。

---

## 5. 风险与缓解

| 风险 | 严重度 | 缓解 |
|------|-------|------|
| **后台 autoUpdate 覆盖** | 🔴 高 | [§2.4](#24) 关 autoUpdate；备份 marketplace.json |
| **手动 `claude plugin marketplace update`** | 🟡 中 | 用户不主动跑就行 |
| **csharp-ls 依赖（dotnet tool）** | 🟢 低 | 占点磁盘，不影响 |
| **plugin.json 字段不一致警告** | 🟢 低 | 改 marketplace.json 时同步顶层 `name`/`version` |
| **plugin entry 顶层 `name` 改错** | 🔴 高 | **不能改**，否则与 pluginId 对不上 |

**强烈建议**：把改过的 marketplace.json 备份成脚本，autoUpdate 触发后用脚本一键重写。

---

## 6. 排错清单

```bash
# 1. 二进制能跑
roslyn-language-server --version

# 2. ENABLE_LSP_TOOL 已启用
echo $env:ENABLE_LSP_TOOL      # PowerShell
echo $ENABLE_LSP_TOOL          # bash

# 3. marketplace.json 改对了
# Windows:
cat $env:USERPROFILE\.claude\plugins\marketplaces\claude-plugins-official\.claude-plugin\marketplace.json | findstr roslyn
# Linux/macOS:
grep -A 10 '"name": "csharp-lsp"' ~/.claude/plugins/marketplaces/claude-plugins-official/.claude-plugin/marketplace.json

# 4. autoUpdate 已关
# 在 Claude Code 里 /plugin → Marketplaces → claude-plugins-official → autoUpdate 应为 false

# 5. plugin 仍 enabled
claude plugin list

# 6. LSP 日志
# 看 ~/.claude/debug/latest 搜 "LSP" / "roslyn"
```

如果以上都对但 Claude 调用 LSP 工具挂起 / 返回空——是 Roslyn LS 的客户端协议兼容问题（`workspace/configuration`、`client/registerCapability`、`window/workDoneProgress/create`），见 [`csharp-lsp-adapter.md`](./csharp-lsp-adapter.md)（若有）或直接换 [方案 B 配 CSharpLspAdapter](./plan-b-local-marketplace.md#csharplspadapter-集成)。

---

## 7. 参考

- 源码核心位置：
  - `@/e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts` — plugin LSP 加载
  - `@/e:/claude-hub/claude-code/src/utils/plugins/schemas.ts` — 字段 schema
  - `@/e:/claude-hub/claude-code/src/services/lsp/manager.ts` — LSP 管理器
  - `@/e:/claude-hub/claude-code/src/utils/plugins/pluginAutoupdate.ts` — 后台更新
  - `@/e:/claude-hub/claude-code/src/tools.ts:250` — `ENABLE_LSP_TOOL` 门槛
- 上游 issue：
  - https://github.com/anthropics/claude-plugins-official/issues/539 — `@indy-singh` 原始 hack 思路
  - https://github.com/anthropics/claude-plugins-official/issues/463 — 请求官方 roslyn-lsp 插件
  - https://github.com/anthropics/claude-plugins-official/pull/464 — 外部 PR 被自动 close
  - https://github.com/anthropics/claude-code/issues/16360 — LSP 客户端缺失协议方法
- 二进制：
  - https://www.nuget.org/packages/roslyn-language-server
  - https://github.com/dotnet/roslyn
- 本仓库相关文档：`@/e:/claude-hub/claude-code/docs/lsp-integration.md`
