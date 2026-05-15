# 方案 B 快速配置 — Roslyn LSP 接入 Claude Code

> 5 分钟完成。基于源码分析后的最简配置：**单一来源** `.lsp.json` + **变量化路径**。  
> 详细原理见 [`plan-b-local-marketplace.md`](./plan-b-local-marketplace.md)。

---

## 前置

```bash
dotnet --version       # ≥ 10.0
claude --version       # ≥ 2.0.74
dotnet tool install --global roslyn-language-server --prerelease
```

---

## 配置（4 个文件 + 2 条命令）

### 1. `~/.claude/settings.json`

启用 LSPTool（**必须显式设置**，源码 `@/e:/claude-hub/claude-code/src/tools.ts:250` 无默认开启）：

```json
{
  "env": { "ENABLE_LSP_TOOL": "1" }
}
```

### 2. 建目录骨架

```bash
mkdir -p ~/.claude-custom-plugins/.claude-plugin
mkdir -p ~/.claude-custom-plugins/plugins/roslyn-ls/.claude-plugin
```

最终结构：

```
~/.claude-custom-plugins/
├── .claude-plugin/marketplace.json
└── plugins/roslyn-ls/
    ├── .claude-plugin/plugin.json
    └── .lsp.json                     ← 唯一的 LSP 配置源
```

### 3. `~/.claude-custom-plugins/.claude-plugin/marketplace.json`

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "local-csharp",
  "owner": { "name": "You" },
  "plugins": [
    {
      "name": "roslyn-ls",
      "version": "1.0.0",
      "source": "./plugins/roslyn-ls",
      "description": "Roslyn Language Server",
      "strict": false
    }
  ]
}
```

> **不写 `lspServers`**——Claude Code 自动读 plugin 根目录的 `.lsp.json`（`@/e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts:64`），避免重复注册。

### 4. `~/.claude-custom-plugins/plugins/roslyn-ls/.claude-plugin/plugin.json`

```json
{
  "name": "roslyn-ls",
  "version": "1.0.0",
  "description": "Roslyn Language Server"
}
```

### 5. `~/.claude-custom-plugins/plugins/roslyn-ls/.lsp.json`

**跨平台可移植版**（用变量代替硬编码路径，源码 `@/e:/claude-hub/claude-code/src/utils/plugins/lspPluginIntegration.ts:255-275` 支持变量替换）：

**Windows**：

```json
{
  "roslyn": {
    "command": "${USERPROFILE}/.dotnet/tools/roslyn-language-server.cmd",
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

**macOS / Linux**：把 `${USERPROFILE}` 换成 `${HOME}`、`.cmd` 后缀去掉即可。

### 6. 注册并安装

```bash
# 在 Claude Code REPL 里：
/plugin marketplace add /Users/YOU/.claude-custom-plugins   # 注意：必须绝对路径，正斜杠

# 退出 Claude Code，终端里：
claude plugin install -s user roslyn-ls
```

Windows 路径写 `C:/Users/YOU/.claude-custom-plugins` 或 `/Users/YOU/.claude-custom-plugins` 均可，**不要用反斜杠**。

---

## 验证

重启 Claude Code：

```
/plugin         # Installed 标签应见: roslyn-ls · local-csharp · ✔ enabled
```

让 Claude 跑：

> 对 src/Program.cs 跑 LSP 的 documentSymbol

输出含 `● LSP(operation: "documentSymbol", ...)` 即成功。

---

## 常见问题

| 现象 | 修复 |
|------|------|
| `/plugin` 显示 enabled 但 LSP 不工作 | `$env:ENABLE_LSP_TOOL` 是否为 `1`；用 `claude -p` 调用？**LSP 仅 REPL 可用**（源码 `@/e:/claude-hub/claude-code/src/services/lsp/manager.ts:148`） |
| `hover` / `findReferences` 挂起，`documentSymbol` 正常 | Roslyn LS 等 `client/registerCapability` 响应，装 [CSharpLspAdapter](https://github.com/Agasper/CSharpLspAdapter) 代理，把 `command` 改成 `csharp-ls-adapter` |
| Unity 项目 `Assembly-CSharp.csproj` 加载超时 | 装 VS Build Tools 或 .NET FX 4.7.1 Developer Pack；或考虑 DotRush |
| 启动 `roslyn-language-server` 命令找不到 | 用 `${USERPROFILE}/.dotnet/tools/...` 绝对路径形式，确认 `dotnet tool list -g` 能看到 `roslyn-language-server` |
| 大型 sln 内存爆涨 | `args` 加 `"--solutionPath", "${CLAUDE_PLUGIN_ROOT}/../../path/to/Backend.sln"` 限定范围 |

---

## 后续

| 想做 | 看 |
|------|----|
| 理解 server 注册 / 启动 / 路由原理 | [`plan-b-local-marketplace.md`](./plan-b-local-marketplace.md) §4 |
| 指定特定 Solution | [`plan-b-local-marketplace.md`](./plan-b-local-marketplace.md) §5.1 |
| 与官方 csharp-lsp 并存 | [`plan-b-local-marketplace.md`](./plan-b-local-marketplace.md) §5.2 |
| 用 CSharpLspAdapter 解决挂起 | [`plan-b-local-marketplace.md`](./plan-b-local-marketplace.md) §5.3 |
| 完整变量列表 | [`plan-b-local-marketplace.md`](./plan-b-local-marketplace.md) §5.4 |
