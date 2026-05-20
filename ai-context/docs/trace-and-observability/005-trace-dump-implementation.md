# 方案 D：基于 dumpPrompts 机制实现完整 Trace 记录

> 目标：通过最小改动，实现类似 `CODEX_ROLLOUT_TRACE_ROOT` 的完整 API 请求/响应记录，包含 **system prompt + tool definitions + messages (turns) + responses**。

---

## 1. 核心发现：`dumpPrompts.ts` 已具备完整能力

源码分析发现 `src/services/api/dumpPrompts.ts` 已经实现了完整的 trace 机制：

```
query.ts (queryLoop)
    │
    ├── createDumpPromptsFetch(sessionId)  ← 创建 fetch 拦截器
    │       │
    │       ▼
    │   fetchOverride: dumpPromptsFetch    ← 注入到 API 调用选项
    │       │
    │       ▼
    └── queryModelWithStreaming() / queryModelWithoutStreaming()
            │
            ▼
        anthropic.beta.messages.create(params, { fetch: fetchOverride })
            │
            ▼
        dumpPromptsFetch 拦截 POST body ← **这里包含完整 API 请求**
            │
            ├── init.body = JSON.stringify({
            │     model, system, tools, messages, betas,
            │     max_tokens, thinking, temperature, ...
            │   })
            │
            ├── dumpRequest() → 解析 body → 写入 JSONL
            │     ├── type: "init"      → system + tools + model + 全部参数（首次/变更时）
            │     ├── type: "system_update" → system/tools 变更时
            │     └── type: "message"   → 增量 user messages
            │
            └── response handler → 克隆 response → 写入 JSONL
                  └── type: "response"  → 完整 streaming chunks 或 JSON response
```

### 1.1 当前限制

| 限制 | 位置 | 说明 |
|------|------|------|
| `config.gates.isAnt` | `query.ts:771` | 仅 `USER_TYPE=ant` 才创建 `dumpPromptsFetch` |
| `USER_TYPE === 'ant'` | `dumpPrompts.ts:100` | `dumpRequest()` 内部对详细写入的二次限制 |
| `USER_TYPE === 'ant'` | `dumpPrompts.ts:174` | response 写入的限制 |
| 固定路径 | `dumpPrompts.ts:59-65` | 写入 `~/.claude/dump-prompts/<id>.jsonl` |

### 1.2 `USER_TYPE=ant` 是构建时常量（非运行时环境变量）

`USER_TYPE` 通过 Bun bundler 的 `--define` 在编译期注入（见 `build.ts:25` + `scripts/defines.ts`）。
外部发布版构建时 `USER_TYPE` **不是** `'ant'`，bundler 会：

1. 将所有 `process.env.USER_TYPE === 'ant'` 常量折叠为 `false`
2. 对应分支被 **dead-code elimination** 移除——代码本身不存在于产物中

```typescript
// src/utils/undercover.ts 注释原文：
// All code paths are gated on process.env.USER_TYPE === 'ant'. Since USER_TYPE is
// a build-time --define, the bundler constant-folds these checks and dead-code-
// eliminates the ant-only branches from external builds.
```

**结论：已发布的 exe/npm 包无法通过设置 `USER_TYPE=ant` 环境变量来开启 dump。必须从源码重新构建。**

方案 D 的改动正是为此设计——添加独立的 `CLAUDE_CODE_TRACE_ROOT` **运行时**环境变量，
在源码中绕过编译时门控，重新构建后即可在运行时通过环境变量控制开关。

---

### 1.3 `init` 记录内容（已验证包含完整数据）

```typescript
// dumpPrompts.ts:110 — 去掉 messages 后的 initData 包含：
const { messages: _, ...initData } = req
// initData = {
//   model: "claude-sonnet-4-20250514",
//   system: [{ type: "text", text: "完整 system prompt...", cache_control: {...} }],
//   tools: [{ name: "Read", description: "...", input_schema: {...} }, ...],
//   betas: ["..."],
//   max_tokens: 16384,
//   thinking: { type: "adaptive" },
//   tool_choice: undefined,
//   metadata: {...},
//   ...其他 API 参数
// }
```

**这就是完整的 system prompt + tool definitions + 模型配置。**

---

## 2. 改动方案

### 2.1 总览

```
修改文件（3个）：
  src/query/config.ts           — 新增 traceEnabled gate
  src/query.ts                  — 放宽 dumpPromptsFetch 创建条件
  src/services/api/dumpPrompts.ts — 放宽写入条件 + 支持自定义路径

新增环境变量：
  CLAUDE_CODE_TRACE_ROOT        — 设置后启用 trace，值为输出目录
```

### 2.2 文件改动详情

#### 2.2.1 `src/query/config.ts`

```diff
 export type QueryConfig = {
   sessionId: SessionId
   gates: {
     streamingToolExecution: boolean
     emitToolUseSummaries: boolean
     isAnt: boolean
     fastModeEnabled: boolean
+    traceEnabled: boolean
   }
 }

 export function buildQueryConfig(): QueryConfig {
   return {
     sessionId: getSessionId(),
     gates: {
       streamingToolExecution: checkStatsigFeatureGate_CACHED_MAY_BE_STALE(
         'tengu_streaming_tool_execution2',
       ),
       emitToolUseSummaries: isEnvTruthy(
         process.env.CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES,
       ),
       isAnt: process.env.USER_TYPE === 'ant',
       fastModeEnabled: !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_FAST_MODE),
+      traceEnabled: !!process.env.CLAUDE_CODE_TRACE_ROOT,
     },
   }
 }
```

#### 2.2.2 `src/query.ts` （约第 771 行）

```diff
-    const dumpPromptsFetch = config.gates.isAnt
-      ? createDumpPromptsFetch(toolUseContext.agentId ?? config.sessionId)
+    const dumpPromptsFetch = (config.gates.isAnt || config.gates.traceEnabled)
+      ? createDumpPromptsFetch(toolUseContext.agentId ?? config.sessionId)
       : undefined
```

#### 2.2.3 `src/services/api/dumpPrompts.ts`

**改动 1：`getDumpPromptsPath` 支持自定义目录**

```diff
 export function getDumpPromptsPath(agentIdOrSessionId?: string): string {
+  const traceRoot = process.env.CLAUDE_CODE_TRACE_ROOT
+  if (traceRoot) {
+    return join(traceRoot, `${agentIdOrSessionId ?? getSessionId()}.jsonl`)
+  }
   return join(
     getClaudeConfigHomeDir(),
     'dump-prompts',
     `${agentIdOrSessionId ?? getSessionId()}.jsonl`,
   )
 }
```

**改动 2：`dumpRequest` 放宽 ant 限制**

```diff
 function dumpRequest(
   body: string,
   ts: string,
   state: DumpState,
   filePath: string,
 ): void {
   try {
     const req = jsonParse(body) as Record<string, unknown>
     addApiRequestToCache(req)

-    if (process.env.USER_TYPE !== 'ant') return
+    if (process.env.USER_TYPE !== 'ant' && !process.env.CLAUDE_CODE_TRACE_ROOT) return
     const entries: string[] = []
```

**改动 3：response 写入放宽限制**

```diff
-    if (timestamp && response.ok && process.env.USER_TYPE === 'ant') {
+    if (timestamp && response.ok && (process.env.USER_TYPE === 'ant' || process.env.CLAUDE_CODE_TRACE_ROOT)) {
```

---

## 3. 输出格式说明

启用后，每个 session 生成一个 JSONL 文件：

```
$CLAUDE_CODE_TRACE_ROOT/
  ├── <session-id>.jsonl           ← 主会话
  └── hook-agent-<uuid>.jsonl      ← hook agent（如有）
```

### 3.1 JSONL 记录格式

**`init` — 首次请求时写入（包含完整 system prompt + tools）**
```jsonl
{"type":"init","timestamp":"2025-05-20T14:52:00.000Z","data":{"model":"claude-sonnet-4-20250514","system":[{"type":"text","text":"You are Claude Code...完整 system prompt 含 CLAUDE.md/memory/env info 等所有动态段...","cache_control":{"type":"ephemeral"}}],"tools":[{"name":"Read","description":"Reads a file...","input_schema":{"type":"object","properties":{"file_path":{"type":"string","description":"..."},...},"required":["file_path"]}},{"name":"Write","description":"...","input_schema":{...}},...],"max_tokens":16384,"thinking":{"type":"adaptive"},"betas":["interleaved-thinking-2025-05-14","prompt-caching-2024-07-31"],"metadata":{...}}}
```

**`system_update` — system/tools 变更时写入（如 compact 后 prompt 变化）**
```jsonl
{"type":"system_update","timestamp":"...","data":{...同 init 格式，不含 messages...}}
```

**`message` — 每轮新 user message**
```jsonl
{"type":"message","timestamp":"...","data":{"role":"user","content":[{"type":"text","text":"请帮我重构这个函数"}]}}
{"type":"message","timestamp":"...","data":{"role":"user","content":[{"type":"tool_result","tool_use_id":"toolu_xxx","content":"文件内容..."}]}}
```

**`response` — API 响应**
```jsonl
{"type":"response","timestamp":"...","data":{"stream":true,"chunks":[{"type":"message_start","message":{"id":"msg_xxx","model":"claude-sonnet-4-20250514","usage":{"input_tokens":12345,"cache_creation_input_tokens":5000,"cache_read_input_tokens":8000},...}},{"type":"content_block_start","index":0,"content_block":{"type":"thinking","thinking":""}},{"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","thinking":"Let me analyze..."}},...]}}
```

### 3.2 数据完整性对比

| 数据 | JSONL transcript | dumpPrompts trace | Codex trace |
|------|:---:|:---:|:---:|
| 完整 system prompt | ❌ | ✅ | ✅ |
| 完整 tool definitions (schema) | ❌ | ✅ | ✅ |
| User messages | ✅ | ✅ | ✅ |
| Assistant messages | ✅ | ✅ (in response) | ✅ |
| Tool calls (input) | ✅ | ✅ | ✅ |
| Tool results (output) | ✅ | ✅ | ✅ |
| Thinking blocks | ❌ (redacted) | ✅ (in response stream) | ✅ |
| Token usage | 部分 | ✅ (message_start.usage) | ✅ |
| Cache statistics | ❌ | ✅ (cache_read/creation) | ✅ |
| Model config (betas/thinking/temp) | ❌ | ✅ | ✅ |
| 原始 API request body | ❌ | ✅ | ✅ |
| 原始 API response | ❌ | ✅ | ✅ |

---

## 4. 使用方式

```bash
# 设置环境变量启用 trace
export CLAUDE_CODE_TRACE_ROOT="$HOME/.claude/traces"

# 正常使用 claude-code
claude

# trace 自动写入到指定目录
ls ~/.claude/traces/
# → abc12345-6789-...jsonl
```

### 4.1 查看 trace 内容

```bash
# 查看 system prompt（init 记录的 data.system 字段）
cat ~/.claude/traces/<session>.jsonl | jq 'select(.type=="init") | .data.system[].text' | head -100

# 查看所有 tool definitions
cat ~/.claude/traces/<session>.jsonl | jq 'select(.type=="init") | .data.tools[] | {name, description}'

# 查看对话 turns
cat ~/.claude/traces/<session>.jsonl | jq 'select(.type=="message") | .data'

# 查看 token 使用
cat ~/.claude/traces/<session>.jsonl | jq 'select(.type=="response") | .data.chunks[] | select(.type=="message_start") | .message.usage'
```

---

## 5. 与 Codex CODEX_ROLLOUT_TRACE_ROOT 对比

| 维度 | Codex 方案 | 本方案 |
|------|-----------|--------|
| 启用方式 | `CODEX_ROLLOUT_TRACE_ROOT=<dir>` | `CLAUDE_CODE_TRACE_ROOT=<dir>` |
| 写入时机 | 运行时 hot-path 写入 | 运行时 fetch 拦截写入 |
| 文件格式 | `trace.jsonl` + `payloads/*.json` | 单一 `<session>.jsonl` |
| 记录粒度 | 细粒度事件（TurnStarted, InferenceStarted...） | 请求级（init + message + response） |
| 数据分离 | 事件流 + 独立 payload 文件 | JSONL 内联（payload 嵌在 data 字段） |
| Reducer | 需要 `codex debug trace-reduce` 离线处理 | 直接可读（jq 即可） |
| 代码改动量 | 新建 rollout-trace crate ~2000 行 | **修改 3 个文件 ~15 行** |
| 侵入性 | 低（独立 crate） | **极低（复用已有机制）** |

---

## 6. 关键源码路径

| 文件 | 作用 |
|------|------|
| `src/services/api/dumpPrompts.ts` | fetch 拦截器 + JSONL 写入 |
| `src/query.ts:771-773` | 创建 dumpPromptsFetch 并注入 |
| `src/query.ts:913` | `fetchOverride: dumpPromptsFetch` 传入 API 选项 |
| `src/services/api/claude.ts:697-730` | `Options` 类型定义（含 `fetchOverride`） |
| `src/services/api/claude.ts:1608-1802` | `paramsFromContext()` 构建完整请求参数 |
| `src/services/api/claude.ts:1039-1050` | `queryModel()` 入口 |
| `src/utils/systemPrompt.ts:41-123` | `buildEffectiveSystemPrompt()` 组装 system prompt |
| `src/constants/prompts.ts:408-507` | `getSystemPrompt()` 构建默认 prompt 段落 |
| `src/utils/api.ts:119-218` | `toolToAPISchema()` 将 Tool 转为 API schema |
| `src/query/config.ts` | QueryConfig gate 定义 |

---

## 7. 风险与注意事项

1. **文件大小**：长会话的 trace 文件可能很大（response 含完整 streaming chunks）。可考虑后续优化为 Codex 式的 payload 分离。
2. **性能**：`dumpPrompts.ts` 已使用 `setImmediate` 延迟解析 + 异步写入，对主流程无阻塞。response clone 是额外内存开销但短暂。
3. **隐私**：trace 含完整 prompt 和代码内容，与 Codex 一样需标注为敏感数据。
4. **并发安全**：`appendToFile` 是 append-only 写入，多个异步写入不会交错（Node.js 单线程 + append mode）。

---

## 8. 后续扩展

- **Phase 2**：编写 Python importer 解析 trace JSONL → SQLite，复用 codex-trace-explorer 的 Tauri 前端
- **Phase 3**：增加 `manifest.json`（trace_id, session_id, cwd, model, started_at）实现目录级索引
- **Phase 4**：可选 payload 分离（大 response 独立为 `payloads/<n>.json`）减小主 JSONL 体积
