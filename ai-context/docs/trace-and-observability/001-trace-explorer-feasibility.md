# Claude Code 消息浏览器可行性分析

> 分析目标：在不对 claude-code 原始代码做大幅改动的前提下，构建一个类似 `codex/external/codex-trace-explorer` 的会话/消息浏览器。

---

## 1. Codex Trace Explorer 架构回顾

```
Codex CLI (写 trace)
      │
      ▼
$CODEX_ROLLOUT_TRACE_ROOT/
   trace-<trace_id>-<thread_id>/
      ├── manifest.json
      ├── trace.jsonl          (事件流)
      └── payloads/<n>.json    (请求/响应原始 JSON)
      │
      ▼
Python Importer ──upsert──> SQLite (~/.codex/explorer.db)
                                  │
                             read-only
                                  │
                                  ▼
                       Tauri Rust Backend ──JSON──> React Frontend
```

**核心设计**：三层分离 — Python 导入器(写)、Rust 后端(只读)、React 前端(展示)。

**数据模型**：sessions → turns → inferences → inference_payloads / tool_calls。

---

## 2. Claude Code 现有数据源分析

### 2.1 Session JSONL 文件（核心数据源 ✅）

**位置**：`~/.claude/projects/<sanitized-cwd>/<session-id>.jsonl`

**特征**：
- 每个会话一个 JSONL 文件，append-only
- 每行是一个 `Entry` 对象（定义在 `src/types/logs.ts`）
- 文件名即 session UUID

**Entry 类型联合**（约 18 种）：

| 类型 | 说明 | 等价于 Codex |
|---|---|---|
| `TranscriptMessage` | user/assistant/system/attachment 消息 | Turn + Inference 合体 |
| `custom-title` | 用户自定义标题 | Session title |
| `ai-title` | AI 生成标题 | — |
| `last-prompt` | 最后用户输入 | — |
| `tag` | 会话标签 | — |
| `summary` | 对话摘要 | — |
| `file-history-snapshot` | 文件历史快照 | — |
| `attribution-snapshot` | 代码贡献归属 | — |
| `content-replacement` | 内容替换记录 | — |
| `speculation-accept` | 推测执行时间节省 | — |
| `marble-origami-commit/snapshot` | 上下文折叠 | — |
| `mode` | coordinator/normal | — |
| `worktree-state` | Git worktree 状态 | — |
| `pr-link` | PR 关联 | — |
| `agent-name/color/setting` | Agent 元数据 | — |

**TranscriptMessage 关键字段**（定义在 `src/types/logs.ts`）：
```typescript
type TranscriptMessage = SerializedMessage & {
  parentUuid: UUID | null         // 链式结构（非平坦列表）
  isSidechain: boolean            // 分支对话
  sessionId: UUID
  timestamp: string
  version: string
  promptId?: string
}

type SerializedMessage = Message & {
  cwd: string
  userType: string
  entrypoint?: string
}
```

**Message 内容**（来自 `@ant/model-provider`）：
- `role`: user / assistant / system
- `content`: string | ContentItem[]（含 text / tool_use / tool_result / thinking 等 block）
- `uuid`, `message.id`, `costUSD`, `durationMs` 等

### 2.2 子代理 Transcript

**位置**：`~/.claude/projects/<sanitized-cwd>/<session-id>/subagents/agent-<agent-id>.jsonl`

每个子代理有独立 JSONL，格式与主 transcript 一致。

### 2.3 远程代理元数据

**位置**：`~/.claude/projects/<sanitized-cwd>/<session-id>/remote-agents/remote-agent-<task-id>.meta.json`

### 2.4 全局历史

**位置**：`~/.claude/history.jsonl`  
**内容**：用户输入历史（display + pastedContents + project + timestamp），不含 assistant 回复。

### 2.5 统计缓存

`src/utils/stats.ts` / `src/utils/statsCache.ts` 已实现按 session 的 token 统计、模型用量等。

### 2.6 Langfuse / OpenTelemetry（可选集成）

- `src/services/langfuse/tracing.ts` — 可选的 Langfuse 集成，记录 trace/span
- `src/utils/telemetry/perfettoTracing.ts` — Perfetto 格式的性能追踪
- `src/utils/telemetry/sessionTracing.ts` — Session 级别的 OTel span

这些是可选的外部集成，非默认启用。

---

## 3. 数据差异对比

| 维度 | Codex Trace Explorer | Claude Code |
|---|---|---|
| 数据格式 | trace.jsonl (事件流) + payloads/*.json | 单一 JSONL (消息流) |
| 数据粒度 | 结构化事件 (RolloutStarted, InferenceStarted...) | 扁平消息 (user/assistant/system) |
| Turn 概念 | 显式 CodexTurnStarted/Ended | 隐式：user → assistant 交替 |
| Inference 概念 | 显式 InferenceStarted/Completed + payload | 无独立 inference 记录；request/response 合在 assistant message 里 |
| Token 统计 | 每个 inference 独立记录 input/output/cached/reasoning tokens | 嵌在 assistant message 的 `costUSD`/token 字段 |
| 消息链 | 线性事件流 | parentUuid 链（有分支/sidechain） |
| 工具调用 | 独立 ToolCallStarted/Ended 事件 | 嵌入 content blocks：tool_use + tool_result |
| Session 元数据 | manifest.json + sessions/*.jsonl | 嵌入 JSONL（custom-title, tag, mode 等行） |
| 原始 payload | 完整保存 request/response JSON | 不保存原始 API 请求/响应 |

### 关键差距

1. **无系统提示词（System Prompt）**：claude-code 发送给 API 的完整 system prompt（包含所有行为规则、项目上下文、CLAUDE.md 内容等）不保存在 JSONL 中。JSONL 仅记录 `messages` 数组中的对话内容。
2. **无工具定义（Tools Schema）**：发送给 API 的 `tools` 参数（每个工具的 name、description、input_schema JSON Schema）不保存在 JSONL 中。JSONL 只记录了工具的**实际调用**（tool_use name + input）和**返回结果**（tool_result），但看不到工具的完整定义/描述。
3. **无原始 API payload**：claude-code 不保存发送给 API 的完整请求/响应 JSON，只保存解析后的消息。Codex 通过 `payloads/<n>.json` 保存完整 request/response JSON（含 system prompt + tools definition + messages），这是与 Codex 最大的区别。
4. **无显式 inference 事件**：没有 InferenceStarted/Completed 事件。不过每条 assistant message 的 `usage` 字段包含 token 统计，`system/turn_duration` 包含每轮耗时。
5. **链式结构**：parentUuid 链需要重建才能还原线性对话，且有 sidechain（分支）复杂性。

> **总结**：JSONL 记录的是"对话发生了什么"（消息内容 + 工具调用结果 + token 消耗），但**不记录"AI 是在什么上下文下工作的"**（系统提示词 + 可用工具列表）。如需后者，可通过**内置 OTel/Langfuse 集成**（仅需设置环境变量，包含完整 system prompt + tools definition，详见 `langfuse-setup.md`）、方案 B（Hook）或 HTTP 代理补充。

---

## 4. 可行性方案

### 方案 A：纯外部 JSONL 解析器（推荐 ✅）

**原理**：直接读取 `~/.claude/projects/` 下的 JSONL 文件，不修改 claude-code 代码。

**架构**：
```
~/.claude/projects/
   <sanitized-cwd>/
      <session-id>.jsonl          ← 直接读取
      <session-id>/subagents/     ← 子代理
      │
      ▼
Python/Node Importer ──> SQLite (~/.claude/cc-explorer.db)
                                │
                           read-only
                                │
                                ▼
                     Tauri/Electron/Web ──> React Frontend
```

**实现步骤**：

1. **Importer 层**
   - 扫描 `~/.claude/projects/` 下所有 `*.jsonl`
   - 解析每行 JSON，按 `type` 分类
   - 重建 parentUuid 链为线性对话
   - 提取 session 元数据（title, tag, cwd, timestamp）
   - 提取 tool_use/tool_result content blocks 为独立 tool_call 记录
   - 从 assistant messages 提取 token/cost 信息
   - 写入 SQLite

2. **数据库 schema（参考 Codex 简化）**

```sql
CREATE TABLE sessions (
    session_id    TEXT PRIMARY KEY,
    project_path  TEXT,
    cwd           TEXT,
    custom_title  TEXT,
    ai_title      TEXT,
    tag           TEXT,
    mode          TEXT,
    first_prompt  TEXT,
    started_at    INTEGER,
    last_modified INTEGER,
    file_size     INTEGER,
    file_path     TEXT
);

CREATE TABLE messages (
    uuid          TEXT PRIMARY KEY,
    session_id    TEXT REFERENCES sessions,
    parent_uuid   TEXT,
    type          TEXT,       -- user/assistant/system/attachment
    role          TEXT,
    content_text  TEXT,       -- 提取的纯文本
    content_json  BLOB,      -- 原始 content 数组
    timestamp     TEXT,
    cost_usd      REAL,
    duration_ms   INTEGER,
    is_sidechain  BOOLEAN,
    version       TEXT
);

CREATE TABLE tool_calls (
    id            TEXT PRIMARY KEY,
    message_uuid  TEXT REFERENCES messages,
    session_id    TEXT REFERENCES sessions,
    tool_name     TEXT,
    input_json    BLOB,
    output_json   BLOB,
    status        TEXT,
    is_error      BOOLEAN
);

CREATE TABLE metadata_entries (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id    TEXT REFERENCES sessions,
    entry_type    TEXT,       -- custom-title/tag/mode/pr-link/...
    payload_json  BLOB,
    timestamp     TEXT
);
```

3. **前端**：复用 Codex Trace Explorer 的三栏 UI 模式
   - 左栏：Session 列表（按项目分组，显示 title/tag/时间）
   - 中栏：消息时间线（按 turn 分组，turn = user → assistant 一轮）
   - 右栏：消息详情面板（Overview / Content / Tools / Raw JSON）

**优点**：
- **零侵入**：完全不修改 claude-code 代码
- 可复用 Codex Trace Explorer 的前端设计
- Session JSONL 已经包含足够的消息内容

**缺点**：
- 无原始 API request/response payload
- Token 统计精度有限（仅有 costUSD，部分 message 无 token 详情）
- 需处理 parentUuid 链重建和 sidechain 过滤

### 方案 B：轻量 Hook 增强（可选增强 ⚡）

在方案 A 基础上，通过 claude-code 的 hook 机制写入额外数据，而非修改源码。

claude-code 支持 user hook（`~/.claude/hooks/` 或 `.claude/hooks/`），可以在 `PostToolUse`、`PreToolUse` 等时机执行外部脚本。

**可利用的 hook 事件**：
- `PostToolUse` — 工具调用完成后，可获取工具名称、输入/输出
- `PreToolUse` — 工具调用前
- `SessionStart` — 会话开始

Hook 脚本通过 stdin 接收 JSON，可追加写入独立的 trace 文件。

**示例 hook（`~/.claude/hooks/PostToolUse.sh`）**：
```bash
#!/bin/bash
# 将工具调用详情追加到 trace 文件
cat | jq '{
  event: "tool_call",
  timestamp: now | todate,
  session_id: .session_id,
  tool_name: .tool_name,
  tool_input: .tool_input,
  tool_output: .tool_output
}' >> ~/.claude/traces/$(date +%Y%m%d).jsonl
```

**优点**：
- 仍不修改 claude-code 源码
- 可获取比 JSONL 更丰富的运行时信息

**缺点**：
- Hook 机制可能有性能影响
- 不同版本 hook API 可能变化
- 仍无法获取原始 API payload

### 方案 C：Langfuse 集成（如需完整 trace ⚡⚡）

claude-code 已内置 Langfuse 支持（`src/services/langfuse/`）。启用后可将完整的 LLM 调用 trace 导出到 Langfuse，包括：
- 每次 API 调用的 input/output
- Token 消耗
- 延迟和错误
- 工具调用链

可以利用 Langfuse 的 API 作为数据源，构建浏览器的后端。

**启用方式**（环境变量）：
```
LANGFUSE_PUBLIC_KEY=...
LANGFUSE_SECRET_KEY=...
LANGFUSE_BASEURL=...  (可自建)
```

**优点**：
- 最接近 Codex Trace Explorer 的数据完整度
- 包含完整 API payload

**缺点**：
- 需要额外的 Langfuse 服务
- 需要用户显式启用

---

## 5. 推荐实施路径

```
Phase 1: 方案 A — 纯外部 JSONL 解析器
   ├── Python importer 解析 ~/.claude/projects/**/*.jsonl
   ├── SQLite 数据库
   ├── Tauri 桌面应用（可复用 codex-trace-explorer 前端框架）
   └── 基础功能：浏览 session → 查看消息 → 查看工具调用

Phase 2: 方案 B — Hook 增强（可选）
   ├── PostToolUse hook 记录工具详情
   └── SessionStart hook 记录 session 元数据

Phase 3: 方案 C — Langfuse 集成（可选）
   └── 对需要完整 API trace 的用户提供 Langfuse 数据源适配
```

---

## 6. 技术要点

### 6.1 JSONL 解析注意事项

1. **文件可能很大**：session JSONL 可达数 GB（`MAX_TRANSCRIPT_READ_BYTES = 50MB` 限制只在部分读取路径生效），需流式读取。
2. **parentUuid 链重建**：
   - 从所有 `TranscriptMessage` 构建 `uuid → message` 映射
   - 找到 leaf node（没有被其他 message 的 parentUuid 指向的 uuid）
   - 沿 parentUuid 回溯构建线性链
   - 参考 `buildConversationChain()` 在 `sessionStorage.ts` 中的实现
3. **Sidechain 过滤**：`isSidechain: true` 的消息属于分支对话，主对话不应包含。
4. **元数据在文件尾部**：custom-title、tag 等通常在文件尾部（`reAppendSessionMetadata` 确保此行为），读取时需扫描尾部。
5. **已有轻量解析**：`readSessionLite()` / `parseSessionInfoFromLite()` 在 `sessionStoragePortable.ts` 中实现了 head+tail 快速读取元数据的方式，可直接复用其逻辑。

### 6.2 Turn 分割逻辑

claude-code 没有显式的 turn 边界。推荐的分割方式：
```
Turn N = user message → (0..N assistant messages with tool_use/tool_result) → final assistant message
```
即：每个 `user` 类型消息开始一个新 turn，直到下一个 `user` 消息。

### 6.3 Tool Call 提取

工具调用嵌入在 assistant message 的 `content` 数组中：
```json
{
  "type": "assistant",
  "content": [
    { "type": "text", "text": "Let me check..." },
    { "type": "tool_use", "id": "toolu_xxx", "name": "Read", "input": {...} }
  ]
}
```

对应的工具结果在下一条 user message 中：
```json
{
  "type": "user",
  "content": [
    { "type": "tool_result", "tool_use_id": "toolu_xxx", "content": "..." }
  ]
}
```

需要通过 `tool_use.id` ↔ `tool_result.tool_use_id` 配对。

### 6.4 项目路径反向解析

`sanitizePath()` 将 cwd 编码为目录名（替换 `/` 为 `-`，截断到 `MAX_SANITIZED_LENGTH`），可以从 JSONL 首行的 `cwd` 字段恢复原始项目路径。

---

## 7. 结论

**可行性：高**。claude-code 的 session JSONL 文件已经包含了构建消息浏览器所需的大部分数据。虽然缺少 Codex 的结构化事件和原始 API payload，但消息内容、工具调用、元数据等核心信息完整。

**推荐方案**：方案 A（纯外部解析器），无需修改 claude-code 任何源码。可直接复用 `codex-trace-explorer` 的 Tauri + React 前端框架，只需替换 Python importer 的解析逻辑即可。

**工作量估算**（方案 A）：
- Python Importer：~500 行（JSONL 解析 + SQLite 写入）
- Rust Backend：可直接复用 codex-trace-explorer 的结构，修改 SQL 查询 ~300 行
- React Frontend：复用 codex-trace-explorer 80%+ 组件，适配 ~500 行
- **总计：约 1300 行代码，1-2 周开发时间**
