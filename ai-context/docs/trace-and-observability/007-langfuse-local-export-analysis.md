# 007 — Langfuse Trace 本地导出方案分析

> 问题：claude-code 自身有 `dumpPrompts` (方案 D) 和 Langfuse OTel 集成两套 trace 体系。
> 如果目标是「记录到本地文件 → 给自研工具导入浏览」，到底用哪一套更合适？

---

## 1. TL;DR

| 维度 | 推荐 |
|---|---|
| 给"自研浏览器"喂数据 | **Langfuse OTel 链路 + 自定义本地 `SpanProcessor`**（最佳） |
| 给"API 录制 / 回放"喂数据 | `dumpPrompts` (方案 D) |
| 两者关系 | **互补，不替代**。Langfuse 走应用层语义；dumpPrompts 走 HTTP 层原始报文 |

**核心结论**：Langfuse 模块已经把 turn 边界、工具调用、子 agent、token 用量、TTFT 都按 trace tree 结构化好了。它构建在 OpenTelemetry 之上 (`BasicTracerProvider` + `LangfuseSpanProcessor`)，是标准 OTel pipeline，**可以平行接入一个本地 `SpanProcessor` 把 span 写成 JSONL**，不需要任何"反向解析 dumpPrompts JSONL 再重建 turn 树"的工作。

---

## 2. 两套 Trace 体系对比

### 2.1 dumpPrompts (方案 D，见 `005-trace-dump-implementation.md`)

```
HTTP fetch override (src/services/api/dumpPrompts.ts)
        │
        ▼
按请求顺序追加写入 <session>.jsonl
  ├── init          原始 system prompt + tools + 模型参数
  ├── system_update system/tools 变更
  ├── message       每条新增 user message
  └── response      原始 streaming chunks
```

- **数据形态**：扁平事件流（按 HTTP 调用时序）
- **粒度**：API 请求级（一次 API 调用 = 一个 request + 一个 response）
- **Turn 信息**：**没有**。`006-trace-turn-correlation.md` 整篇都在讨论怎么离线把它和 session JSONL join 出 turn 归属
- **工具调用**：只能看到嵌入在 message/response 里的 `tool_use` / `tool_result` content block，**没有工具实际执行的耗时 / 错误标记**
- **子 agent**：独立 `<agent-id>.jsonl` 文件，但是和主 trace 没有显式关联
- **优势**：100% 原始 API 字节，可用于回放和 prompt 调试
- **劣势**：作为"trace 浏览器"的数据源 → **需要重建 turn 树、关联 session JSONL、配对 tool_use ↔ tool_result**

### 2.2 Langfuse OTel 集成 (`src/services/langfuse/`)

```
应用层多点埋点
        │
        ▼
@langfuse/tracing.startObservation() — 创建 OTel Span
        │
        ▼
BasicTracerProvider
        │
        ▼
LangfuseSpanProcessor  ── HTTPS ──>  Langfuse Cloud / 自部署
```

埋点位置（已编织好 trace tree）：

| 位置 | 调用 | 产生的 span |
|---|---|---|
| `query.ts:295-305` | `createTrace()` | **agent root span**（每个 query turn 一个） |
| `runAgent.ts` | `createSubagentTrace()` | 子 agent root span |
| `claude.ts:2975` | `recordLLMObservation()` | LLM `generation` span (含 input/output/usage/TTFT/thinking) |
| `toolExecution.ts:1355,1675` | `recordToolObservation()` | `tool` span (含 input/output/isError/toolUseId) |
| `tracing.ts:255` | `createToolBatchSpan()` | 并发工具批的父 `span` |
| `query.ts:349` | `endTrace()` | 关闭 root，设置 status |

数据形态本身就是完整的 trace tree：

```
agent: agent-run                                    ← Root (= 一个 turn)
├── tools (batch, 仅当并发)
│   ├── ChatAnthropic           (generation)
│   ├── FileEditTool            (tool)
│   └── BashTool                (tool)
├── ChatAnthropic               (generation)
└── ...
```

并且每个 span 已经带了：
- `LangfuseOtelSpanAttributes.TRACE_SESSION_ID` — 关联到 session
- `LangfuseOtelSpanAttributes.TRACE_USER_ID` — 关联到用户
- `provider`, `model`, `agentType`, `toolUseId`, `isError`, `usageDetails` 等语义字段

### 2.3 完整度对比

| 数据 | dumpPrompts | Langfuse 埋点 |
|---|:---:|:---:|
| 原始 API request body | ✅ | ❌（已转 OpenAI 兼容格式） |
| 原始 streaming chunks | ✅ | ❌（仅最终 output） |
| 完整 system prompt | ✅ | ✅（通过 `convertMessagesToLangfuse` 注入 system 消息） |
| Tool definitions (schema) | ✅ | ✅（`convertToolsToLangfuse` 输出 OpenAI tool 格式） |
| **Turn 边界** | ❌（要 join 两个 JSONL） | ✅（root span = 一个 turn） |
| **工具执行耗时** | ❌（只有 API 视角） | ✅（`startTime` → `end()`） |
| **工具错误标记** | ❌ | ✅（`isError` + `level: 'ERROR'`） |
| **子 agent 关联** | 独立文件，无关联 | ✅（独立 trace，统一 sessionId） |
| **Token cache 细分** | ✅ | ✅（`cache_read` / `cache_creation`） |
| **TTFT** | ✅（需算） | ✅（`completionStartTime`） |
| Thinking 配置 | ✅ | ✅ |
| 数据脱敏 | ❌（明文） | ✅（`sanitize.ts` 已就绪） |

**关键不对称**：
- 给"API 录制回放" → dumpPrompts 胜（原始字节）
- 给"会话浏览器 / 性能分析" → Langfuse 胜（结构化 + 语义 + 关联）

---

## 3. 为什么"改造 Langfuse 模块写本地文件"是好主意

Langfuse 模块构建在 OpenTelemetry 之上，关键代码：

```@g:\ai-devtools\claude-code\src\services\langfuse\client.ts:31-52
    processor = new LangfuseSpanProcessor({
      publicKey: process.env.LANGFUSE_PUBLIC_KEY,
      secretKey: process.env.LANGFUSE_SECRET_KEY,
      baseUrl: process.env.LANGFUSE_BASE_URL ?? 'https://cloud.langfuse.com',
      flushAt: parseInt(process.env.LANGFUSE_FLUSH_AT ?? '20', 10),
      flushInterval: parseInt(process.env.LANGFUSE_FLUSH_INTERVAL ?? '10', 10),
      mask: maskFn,
      environment: process.env.LANGFUSE_TRACING_ENVIRONMENT ?? 'development',
      release: MACRO.VERSION,
      exportMode:
        (process.env.LANGFUSE_EXPORT_MODE as
          | 'batched'
          | 'immediate'
          | undefined) ?? 'batched',
      timeout: parseInt(process.env.LANGFUSE_TIMEOUT ?? '5', 10),
    })

    provider = new BasicTracerProvider({
      spanProcessors: [processor],
    })
```

`BasicTracerProvider` 接收的是 **`spanProcessors` 数组**——OTel 的标准模式就是「多个 processor 同时消费同一份 span 流」。这意味着：

- 写一个 `FileSpanProcessor`（实现 `SpanProcessor` 接口）
- 把它和 `LangfuseSpanProcessor` 一起塞进数组
- 同一个 span 会被两个处理器各收到一份：一份发到 Langfuse Cloud，一份落盘

**完全不需要改 Langfuse SDK，也不需要改 tracing.ts 任何一处埋点**。所有现成的 `createTrace` / `recordLLMObservation` / `recordToolObservation` 都自动产生本地文件。

且能直接复用：
- `sanitize.ts` 的脱敏（通过 `mask` 钩子已经在 span 落地前生效）
- `convert.ts` 的 OpenAI 格式（统一 schema，自研浏览器一份解析逻辑就能复用 OpenAI 生态工具）
- `LangfuseOtelSpanAttributes.TRACE_SESSION_ID` 的语义字段
- Token usage / TTFT / 错误等级 / 工具错误标记

---

## 4. 实施方案

### 4.1 方案 A：新增 `FileSpanProcessor`（最小改动，推荐）

#### 4.1.1 新增文件 `src/services/langfuse/fileProcessor.ts`

```typescript
import { mkdirSync, createWriteStream, type WriteStream } from 'fs'
import { join } from 'path'
import type {
  SpanProcessor,
  ReadableSpan,
  Span,
} from '@opentelemetry/sdk-trace-base'
import type { Context } from '@opentelemetry/api'
import { sanitizeGlobal } from './sanitize.js'

/**
 * OTel SpanProcessor that writes each finished span to a JSONL file.
 * One file per process; rotate by date to avoid unbounded growth.
 *
 * Output schema (per line):
 *   {
 *     traceId, spanId, parentSpanId,
 *     name, kind, status,
 *     startTime, endTime, durationNs,
 *     attributes,   // includes langfuse.observation.input/output, usage, sessionId, ...
 *     events,
 *     resource
 *   }
 */
export class FileSpanProcessor implements SpanProcessor {
  private stream: WriteStream

  constructor(filePath: string) {
    mkdirSync(require('path').dirname(filePath), { recursive: true })
    this.stream = createWriteStream(filePath, { flags: 'a' })
  }

  onStart(_span: Span, _ctx: Context): void {
    // no-op; Langfuse attributes are set after start, we serialize at onEnd
  }

  onEnd(span: ReadableSpan): void {
    try {
      const ctx = span.spanContext()
      const record = {
        traceId: ctx.traceId,
        spanId: ctx.spanId,
        parentSpanId: (span as unknown as { parentSpanId?: string }).parentSpanId,
        name: span.name,
        kind: span.kind,
        status: span.status,
        startTime: hrTimeToISO(span.startTime),
        endTime: hrTimeToISO(span.endTime),
        durationNs: hrTimeDeltaNs(span.startTime, span.endTime),
        attributes: sanitizeGlobal(span.attributes),
        events: span.events.map(e => ({
          name: e.name,
          time: hrTimeToISO(e.time),
          attributes: e.attributes,
        })),
        resource: span.resource.attributes,
      }
      this.stream.write(JSON.stringify(record) + '\n')
    } catch {
      /* ignore — observability must never throw */
    }
  }

  async forceFlush(): Promise<void> {
    await new Promise<void>(resolve => {
      // WriteStream has no public flush; wait for next drain
      if (!this.stream.writableNeedDrain) return resolve()
      this.stream.once('drain', () => resolve())
    })
  }

  async shutdown(): Promise<void> {
    await new Promise<void>(resolve => this.stream.end(() => resolve()))
  }
}

function hrTimeToISO([sec, nsec]: [number, number]): string {
  return new Date(sec * 1000 + nsec / 1e6).toISOString()
}
function hrTimeDeltaNs(
  [s1, ns1]: [number, number],
  [s2, ns2]: [number, number],
): number {
  return (s2 - s1) * 1e9 + (ns2 - ns1)
}
```

#### 4.1.2 在 `client.ts` 注册

```diff
+ import { FileSpanProcessor } from './fileProcessor.js'
+ import { getSessionId } from 'src/utils/sessionState.js'

  export function initLangfuse(): boolean {
    if (processor !== null) return true
-   if (!isLangfuseEnabled()) {
+   const wantLocal = !!process.env.LANGFUSE_LOCAL_TRACE_ROOT
+   if (!isLangfuseEnabled() && !wantLocal) {
      logForDebugging('[langfuse] No keys configured, running in no-op mode')
      return false
    }

    try {
      const maskFn: MaskFunction = ({ data }) => sanitizeGlobal(data)

-     processor = new LangfuseSpanProcessor({ ... })
-     provider = new BasicTracerProvider({
-       spanProcessors: [processor],
-     })
+     const processors = []
+     if (isLangfuseEnabled()) {
+       processor = new LangfuseSpanProcessor({ /* unchanged */ })
+       processors.push(processor)
+     }
+     if (wantLocal) {
+       const file = join(
+         process.env.LANGFUSE_LOCAL_TRACE_ROOT!,
+         `${getSessionId()}.jsonl`,
+       )
+       processors.push(new FileSpanProcessor(file))
+     }
+     provider = new BasicTracerProvider({ spanProcessors: processors })

      setLangfuseTracerProvider(provider)
      ...
    }
  }
```

并在 `isLangfuseEnabled()` 的语义上做一点选择：

```diff
- export function isLangfuseEnabled(): boolean {
-   return !!(process.env.LANGFUSE_PUBLIC_KEY && process.env.LANGFUSE_SECRET_KEY)
- }
+ /** True if any export sink (remote or local) is enabled. */
+ export function isLangfuseEnabled(): boolean {
+   return (
+     (!!process.env.LANGFUSE_PUBLIC_KEY && !!process.env.LANGFUSE_SECRET_KEY) ||
+     !!process.env.LANGFUSE_LOCAL_TRACE_ROOT
+   )
+ }
```

> `isLangfuseEnabled()` 被 `tracing.ts` 内所有 `recordXxx` 用作总开关，改为「任何一个出口启用就埋点」即可。

#### 4.1.3 启用方式

```powershell
# 仅本地（无需 Langfuse 账号）
$env:LANGFUSE_LOCAL_TRACE_ROOT = "$HOME\.claude\langfuse-traces"
claude

# 同时本地 + Langfuse Cloud
$env:LANGFUSE_LOCAL_TRACE_ROOT = "$HOME\.claude\langfuse-traces"
$env:LANGFUSE_PUBLIC_KEY = "pk-..."
$env:LANGFUSE_SECRET_KEY = "sk-..."
claude
```

输出：

```
~/.claude/langfuse-traces/
  <session-id>.jsonl            # 主 agent + LLM + tool span
  <subagent-id>.jsonl           # 子 agent（独立 trace，但同一 sessionId 关联）
```

### 4.2 方案 B：OTel 标准 `OTLPTraceExporter` + 收集器（适合既有 OTel 基建）

如果未来想接入 Jaeger / Tempo / Grafana：

```typescript
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base'

processors.push(new BatchSpanProcessor(new OTLPTraceExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
})))
```

claude-code 的 `package.json` 已经声明了 `@opentelemetry/exporter-trace-otlp-http`，零额外依赖。

> 缺点：自研浏览器要支持 OTLP/JSON protobuf 解析，比纯 JSONL 麻烦。

### 4.3 方案 C：双管齐下（最完备）

`LANGFUSE_LOCAL_TRACE_ROOT` 写结构化 span 文件 + `CLAUDE_CODE_TRACE_ROOT`(方案 D) 写原始 API 报文。

自研浏览器可以：
- 主视图基于 Langfuse JSONL（trace 树、turn、工具）
- 详情面板按 `langfuse.observation.input.message.id` 跳转到 dumpPrompts JSONL，展示原始 streaming chunks

两者通过 **API `message.id`** 关联（见 `006-trace-turn-correlation.md` 方案 1）。

---

## 5. 自研浏览器的数据模型设计

如果走方案 A，每行 JSONL 是一个 OTel span。建议落库 schema：

```sql
CREATE TABLE traces (
  trace_id      TEXT PRIMARY KEY,
  session_id    TEXT,           -- attributes['langfuse.session.id']
  user_id       TEXT,           -- attributes['langfuse.user.id']
  agent_type    TEXT,           -- 'main' | <subagent-type>
  root_name     TEXT,           -- 'agent-run' | 'agent:<type>'
  started_at    INTEGER,
  ended_at      INTEGER,
  status_level  TEXT,           -- normal / WARNING (interrupted) / ERROR
  input_json    BLOB,
  output_json   BLOB
);

CREATE TABLE spans (
  span_id       TEXT PRIMARY KEY,
  trace_id      TEXT REFERENCES traces,
  parent_id     TEXT,
  name          TEXT,           -- 'ChatAnthropic' / 'FileEditTool' / 'tools' / ...
  observation_type TEXT,        -- 'agent' | 'generation' | 'tool' | 'span'
  started_at    INTEGER,
  ended_at      INTEGER,
  duration_ns   INTEGER,
  attributes    BLOB
);

CREATE TABLE generations (        -- 视图便于查询
  span_id              TEXT PRIMARY KEY REFERENCES spans,
  trace_id             TEXT,
  model                TEXT,
  provider             TEXT,
  input_tokens         INTEGER,
  output_tokens        INTEGER,
  cache_read_tokens    INTEGER,
  cache_creation_tokens INTEGER,
  ttft_ms              INTEGER,
  completion_start     INTEGER,
  thinking_type        TEXT,
  thinking_budget      INTEGER
);

CREATE TABLE tool_calls (
  span_id      TEXT PRIMARY KEY REFERENCES spans,
  trace_id     TEXT,
  tool_name    TEXT,
  tool_use_id  TEXT,
  is_error     BOOLEAN,
  input_json   BLOB,
  output_text  TEXT
);
```

字段直接从 OTel attributes 提取，attribute key 是 Langfuse 约定的：
- `langfuse.observation.input` / `langfuse.observation.output`
- `langfuse.observation.usage_details`
- `langfuse.observation.completion_start_time`
- `langfuse.observation.metadata.*`
- `langfuse.session.id` / `langfuse.user.id`
- `langfuse.observation.type` (`agent` / `generation` / `tool` / `span`)

> 这些 attribute key 由 `@langfuse/tracing` 包内部赋值，可在 Langfuse 官方文档"OpenTelemetry attribute mapping"找到完整列表。

UI 沿用 `001-trace-explorer-feasibility.md` 提的三栏布局，把"消息"换成"span tree"，把"工具调用"换成 `tool_calls` 视图即可。

---

## 6. 风险与注意事项

1. **`mask` 钩子的差异**
   `LangfuseSpanProcessor` 的 `mask` 在 export 时执行；自定义 `FileSpanProcessor` 在 `onEnd` 时调 `sanitizeGlobal`。两者效果一致，但若未来 Langfuse SDK 改成在 span 上**先 mask 再交给所有 processor**，那本地 processor 拿到的就是已脱敏的数据，**`sanitizeGlobal` 会变成空操作**——这是好事，不会重复脱敏。

2. **大 span 内存压力**
   `query.ts:349-364` 的注释明确警告："SpanImpl 会持有 hundreds of KB JSON，processor batch 不及时 flush 会 retain 内存"。本地 processor 用 append-only stream **同步写**，不存在 batch 滞留问题。

3. **OTel context propagation**
   `tracing.ts` 用 `parentSpanContext: rootSpan.otelSpan.spanContext()` 显式传递父子关系，**不依赖 AsyncLocalStorage**——本地 processor 拿到的 `span.parentSpanId` 是可靠的，trace 树重建零歧义。

4. **bundle 体积**
   `FileSpanProcessor` 只用 Node 内置 `fs` + `path`，零新增依赖。

5. **构建期常量**
   不像 `USER_TYPE=ant` 是 build-time `--define`，`LANGFUSE_*` 全部是运行时 env，已发布产物可直接启用。

---

## 7. 推荐路径

```
Step 1：实现 FileSpanProcessor（方案 4.1，约 80 行）
        + 改 client.ts 注册逻辑（约 15 行）
        + 改 isLangfuseEnabled 语义（约 3 行）
        → 总改动 < 100 行，零新增依赖

Step 2：写一个轻量 Python importer
        scan ~/.claude/langfuse-traces/*.jsonl
        按 trace_id 聚合 → SQLite
        ~200 行

Step 3：浏览器前端
        - 复用 codex-trace-explorer 三栏布局
        - 中间栏改为 span tree（可折叠）
        - 右侧详情面板按 observation_type 切换：
            generation → input/output diff + usage + TTFT
            tool       → input/output + isError + duration
            agent      → 整 turn 概览
        ~600 行 React

Step 4（可选）：和 dumpPrompts 联动
        若同时设置 CLAUDE_CODE_TRACE_ROOT，
        浏览器右侧加 "View raw API" 跳转，
        按 attributes['langfuse.observation.output'] 中的 message.id
        关联 dumpPrompts 的对应 response 记录。
```

---

## 8. 关键引用

| 来源 | 行号 | 说明 |
|---|---|---|
| `@g:\ai-devtools\claude-code\src\services\langfuse\client.ts:48-50` | — | `BasicTracerProvider` 接受 `spanProcessors` 数组，多 processor 是原生支持 |
| `@g:\ai-devtools\claude-code\src\services\langfuse\tracing.ts:41-66` | — | `createTrace` = 一个 turn 的 root span，已带 sessionId 关联 |
| `@g:\ai-devtools\claude-code\src\services\langfuse\tracing.ts:122-143` | — | `recordLLMObservation` 直接走 `startObservation`，含完整 input/output/usage/TTFT/thinking |
| `@g:\ai-devtools\claude-code\src\services\langfuse\tracing.ts:202-216` | — | `recordToolObservation` 含工具耗时 + 错误等级 |
| `@g:\ai-devtools\claude-code\src\query.ts:295-313` | — | 主 query loop 中 Langfuse trace 的创建与注入 |
| `@g:\ai-devtools\claude-code\src\services\api\claude.ts:2975-2991` | — | LLM 调用点的埋点，含 system prompt + tool schemas |
| `@g:\ai-devtools\claude-code\src\services\tools\toolExecution.ts:1355-1363` | — | 工具执行的埋点 |
| `@g:\ai-devtools\claude-code\ai-context\docs\trace-and-observability\005-trace-dump-implementation.md` | — | dumpPrompts 方案 D 全文 |
| `@g:\ai-devtools\claude-code\ai-context\docs\trace-and-observability\006-trace-turn-correlation.md` | — | dumpPrompts ↔ session 的 turn 关联难题 |

---

## 9. 一句话总结

**dumpPrompts 是"录音机"，Langfuse 埋点是"会议纪要"**。
要做 trace 浏览器 → 接 Langfuse 这条线、加一个本地 `SpanProcessor` 就足够；
要做 prompt / API 调试 → 用 dumpPrompts；
两者完全可以并存，唯一关联键是 API `message.id`。
