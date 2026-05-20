# 构建与运行指南

## 构建

在项目根目录执行：

```bash
bun run build
```

### 产物说明

输出到 `dist/` 目录：

| 文件 | 说明 |
|------|------|
| `dist/cli.js` | 主 bundle（+ 约 450 个 chunk 文件） |
| `dist/cli-bun.js` | Bun shebang 入口（`#!/usr/bin/env bun`） |
| `dist/cli-node.js` | Node.js shebang 入口（`#!/usr/bin/env node`） |
| `dist/vendor/` | 原生依赖（audio-capture、ripgrep） |

> 构建**不会**生成 exe。产物是 JS bundle，需要 bun 或 node 运行时。

## 运行方式

### 方式一：`bun link`（推荐）

在项目根目录执行：

```bash
bun run build
bun link
```

`bun link` 会读取 `package.json` 的 `bin` 字段，在 Bun 全局 bin 目录（`~/.bun/bin/`）创建符号链接：

- `ccb` → `dist/cli-node.js`（Node.js 运行）
- `ccb-bun` → `dist/cli-bun.js`（Bun 运行）
- `claude-code-best` → `dist/cli-node.js`

之后在任意目录直接使用：

```bash
cd d:\your\project
ccb          # 工作目录 = d:\your\project
ccb-bun      # 同上，以 bun 运行
```

**前提**：`~/.bun/bin` 已在系统 PATH 中（安装 Bun 时通常自动添加）。

验证 PATH：

```powershell
# 查看 Bun 全局 bin 目录
bun pm bin -g

# 检查是否在 PATH 中
$env:PATH -split ';' | Select-String 'bun'
```

解除链接：

```bash
bun unlink
```

### 方式二：直接运行

```bash
cd d:\your\project
bun d:\codingnet\claude-code\dist\cli-bun.js     # 以 bun 运行
node d:\codingnet\claude-code\dist\cli-node.js    # 以 node 运行
```

### 方式三：npm 全局安装

```bash
npm i -g claude-code-best
ccb
```

## 工作目录机制

CLI 通过 `process.cwd()` 确定工作目录（`bootstrap/state.ts`），因此**在哪个目录运行命令，就以哪个目录为工作目录**。

> **注意**：`bun run dev` 开发模式下，`scripts/dev.ts` 硬编码了 `cwd: projectRoot`，工作目录始终是项目自身路径，无法指定其他目录。构建后的产物没有这个限制。
