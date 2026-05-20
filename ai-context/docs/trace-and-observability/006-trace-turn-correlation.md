# Trace Turn 关联机制：识别 API Request 属于哪个 Turn

> 核心问题：`dumpPrompts.ts` 在 HTTP fetch 层拦截请求，没有上层 turn 信息。如何将每个 API request/response 关联到用户 turn？

---

## 方案 1：API Message ID 精确匹配（推荐，零改动）

### 原理

Claude API 每次返回的 response 带有唯一 `message.id`（如 `msg_56ee8d28...`），由服务端分配，全局唯一。
该 ID 同时存在于 session JSONL 和 dumpPrompts JSONL 中。

### 匹配路径

```
dumpPrompts JSONL                           session JSONL
─────────────────                           ─────────────
type:"response"                             type:"assistant"
 └─ data.chunks[0]                           └─ message.id = "msg_56ee8d28..."
     └─ type:"message_start"                      │
         └─ message.id = "msg_56ee8d28..." ◄──────┘ 精确匹配
                                                   │
                                            该 assistant message 所在的
                                            promptId 组 = 所属 turn
```

### 具体数据示例

**session JSONL（turn 信息）：**
```jsonl
{"promptId":"22a2de34-...","type":"user","message":{"content":"帮我写一个python版本"},"timestamp":"..."}
{"type":"assistant","message":{"id":"msg_56ee8d2853d54a109d9f10c5fe98fedd","content":[{"name":"Write","type":"tool_use",...}],"stop_reason":"tool_use"},...}
{"promptId":"22a2de34-...","type":"user","message":{"content":[{"type":"tool_result",...}]},...}
{"type":"assistant","message":{"id":"msg_97021290a20d44f9b46ae51cd170c0bc","content":[{"name":"Bash","type":"tool_use",...}],"stop_reason":"tool_use"},...}
{"type":"assistant","message":{"id":"msg_3e6483b4274740c8b9e88c16a6c70176","content":[{"type":"text",...}],"stop_reason":"end_turn"},...}
{"type":"system","subtype":"turn_duration","durationMs":75714,...}
```

**dumpPrompts JSONL（API payload）：**
```jsonl
{"type":"response","timestamp":"T1","data":{"stream":true,"chunks":[{"type":"message_start","message":{"id":"msg_56ee8d2853d54a109d9f10c5fe98fedd",...}},...]}}
{"type":"response","timestamp":"T2","data":{"stream":true,"chunks":[{"type":"message_start","message":{"id":"msg_97021290a20d44f9b46ae51cd170c0bc",...}},...]}}
{"type":"response","timestamp":"T3","data":{"stream":true,"chunks":[{"type":"message_start","message":{"id":"msg_3e6483b4274740c8b9e88c16a6c70176",...}},...]}}
```

### 关联算法

```python
import json

def correlate(session_jsonl_path, dump_jsonl_path):
    # 1. 从 session JSONL 建立 msg_id → promptId (turn) 映射
    msg_to_turn = {}
    for line in open(session_jsonl_path):
        entry = json.loads(line)
        if entry.get("type") == "assistant" and "message" in entry:
            msg_id = entry["message"].get("id")
            # promptId 在同组的 user message 上，需向上查找
            # 但更简单的方式是按顺序追踪当前 promptId
            if msg_id:
                msg_to_turn[msg_id] = current_prompt_id

    # 2. 从 dumpPrompts JSONL 提取每个 response 的 msg_id
    for line in open(dump_jsonl_path):
        entry = json.loads(line)
        if entry.get("type") == "response":
            chunks = entry["data"].get("chunks", [])
            for chunk in chunks:
                if chunk.get("type") == "message_start":
                    msg_id = chunk["message"]["id"]
                    turn = msg_to_turn.get(msg_id)
                    # 现在知道这个 response 属于哪个 turn
```

### 优缺点

| 优点 | 缺点 |
|------|------|
| 零代码改动 | 需要两个文件联合查询 |
| 精确唯一匹配 | 只能关联 response（request 无独立 ID） |
| 服务端分配，不可能冲突 | request 到 response 的关联靠顺序（同一文件内 N-th request 对应 N-th response） |

---

## 方案 2：在 dumpPrompts 中注入 promptId（轻量改动）

### 原理

在 `query.ts` 的 queryLoop 中，每次 API 调用前将当前 `promptId` 传递到 dumpPrompts 层。

### 改动位置

```typescript
// query.ts — 调用 createDumpPromptsFetch 时传入 promptId
const dumpPromptsFetch = (config.gates.isAnt || config.gates.traceEnabled)
  ? createDumpPromptsFetch(
      toolUseContext.agentId ?? config.sessionId,
      { promptId: currentPromptId }  // ← 新增
    )
  : undefined
```

**问题**：`dumpPromptsFetch` 在 queryLoop 开头创建一次（第 771 行注释明确说明 "create once per query session to avoid memory retention"），而 promptId 在 turn 之间会变。

### 替代：通过闭包共享可变引用

```typescript
// query.ts
const traceContext = { promptId: '', turnIndex: 0 }
const dumpPromptsFetch = createDumpPromptsFetch(sessionId, traceContext)

// queryLoop 每次迭代开始时更新
traceContext.promptId = currentPromptId
traceContext.turnIndex++
```

```typescript
// dumpPrompts.ts — dumpRequest 中写入 traceContext
entries.push(
  `{"type":"turn_marker","timestamp":"${ts}","promptId":"${ctx.promptId}","turnIndex":${ctx.turnIndex}}`
)
```

### 输出效果

```jsonl
{"type":"init","timestamp":"T0","data":{system,tools,...}}
{"type":"turn_marker","timestamp":"T1","promptId":"02cb129b-...","turnIndex":1}
{"type":"message","timestamp":"T1","data":{"role":"user","content":"什么是快速排序"}}
{"type":"response","timestamp":"T1","data":{...}}
{"type":"turn_marker","timestamp":"T2","promptId":"22a2de34-...","turnIndex":2}
{"type":"message","timestamp":"T2","data":{"role":"user","content":[{"type":"tool_result",...}]}}
{"type":"response","timestamp":"T2","data":{...}}
```

### 优缺点

| 优点 | 缺点 |
|------|------|
| 单文件自包含 | 需改动 2 处代码（query.ts + dumpPrompts.ts） |
| promptId 直接可见 | traceContext 可变引用需仔细管理 |
| turnIndex 线性递增，易于理解 | |

---

## 方案 3：利用 messages 数组长度差推断 turn 边界（零改动，纯离线）

### 原理

`dumpPrompts.ts` 中 `state.messageCountSeen` 追踪了已写入的消息数量。每次 API 请求时，`req.messages` 的长度增长模式暗示 turn 结构：

- 新 user turn 开始：messages 增加 1 条 user message
- tool_result 循环：messages 增加 assistant + user(tool_result) 各 1 条
- compact 后：messages 数量骤减

### 离线推断算法

```python
# 连续 request 之间 messages 变化模式：
# [12] → [14] → [16]  = 同一 turn 内的 tool_use loop（每次 +2: assistant + tool_result）
# [16] → [17]          = 新 user turn 开始（+1: 新 user message）
# [17] → [5]           = compact 发生（骤减）
```

### 优缺点

| 优点 | 缺点 |
|------|------|
| 零改动，纯后处理 | 不精确，edge case 多 |
| | compact/snip 会打断连续性 |
| | 需要 dumpPrompts 记录 messages 数量（当前未记录） |

---

## 方案 4：利用 session JSONL 的 `parentUuid` 链（零改动，纯离线）

### 原理

session JSONL 中每条消息都有 `parentUuid`，形成链表。同一 turn 内消息共享 `promptId`：

```
user(promptId:A) → assistant → user/tool_result(promptId:A) → assistant → ... → turn_duration
```

turn 边界由 `type:"system", subtype:"turn_duration"` 明确标记。

### 重建方式

```python
turns = []
current_turn = {"promptId": None, "messages": [], "api_calls": []}

for entry in session_jsonl:
    if entry["type"] == "user" and "promptId" in entry:
        if entry["promptId"] != current_turn["promptId"]:
            # 新 turn
            if current_turn["promptId"]:
                turns.append(current_turn)
            current_turn = {"promptId": entry["promptId"], "messages": [entry], "api_calls": []}
        else:
            current_turn["messages"].append(entry)
    elif entry["type"] == "assistant":
        current_turn["messages"].append(entry)
        current_turn["api_calls"].append(entry["message"]["id"])  # msg_xxx
    elif entry.get("subtype") == "turn_duration":
        turns.append(current_turn)
        current_turn = {"promptId": None, "messages": [], "api_calls": []}
```

然后用每个 turn 的 `api_calls` 列表（msg_xxx IDs）去 dumpPrompts 中关联对应的 request/response。

### 优缺点

| 优点 | 缺点 |
|------|------|
| 零代码改动 | 需要两个文件联合处理 |
| turn 边界由 session JSONL 明确定义 | 解析逻辑较复杂（链表遍历） |
| 100% 精确（promptId + turn_duration 双重保证） | |

---

## 总结对比

| 方案 | 代码改动 | 精确度 | 单文件 | 推荐场景 |
|------|:--------:|:------:|:------:|----------|
| **1. msg ID 匹配** | 0 | ✅ 精确 | ❌ 需两文件 | 立即可用，最稳妥 |
| **2. 注入 promptId** | ~15 行 | ✅ 精确 | ✅ | 方案 D 的增强，最终目标 |
| 3. messages 长度推断 | 0 | ⚠️ 不精确 | ✅ | 不推荐 |
| **4. parentUuid 链** | 0 | ✅ 精确 | ❌ 需两文件 | 纯离线分析首选 |

### 推荐组合

**短期**：方案 1 或 4（零改动，离线工具解析两个 JSONL 文件）
**中期**：方案 2（在方案 D 改动中顺便加入 turn_marker，实现单文件自包含 trace）
