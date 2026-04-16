# 子领域 01：Agent 循环 (ReAct 模式)

> Claude Code 核心的 Agent 循环实现分析

---

## 一、概述

Agent 循环是 Claude Code 的核心引擎，实现了经典的 **ReAct (Reason-Act-Observe)** 模式。这个循环持续在 LLM 推理和工具执行之间切换，直到任务完成。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/query.ts` | Agent 循环主状态机 |
| `src/QueryEngine.ts` | SDK 入口，状态管理 |
| `src/toolOrchestration.ts` | 工具并发执行 |

---

## 二、ReAct 模式原理

### 2.1 基本概念

```
ReAct = Reason + Act + Observe

Reason: 模型思考下一步该做什么
Act:    模型决定调用工具
Observe: 模型收到工具结果，继续思考
```

### 2.2 循环流程

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent 循环                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│   │  Reason  │ ──►│   Act    │ ──►│ Observe  │──┐          │
│   │ (思考)   │    │ (行动)   │    │ (观察)   │  │          │
│   └──────────┘    └──────────┘    └──────────┘  │          │
│        ▲                                         │          │
│        │                                         │          │
│        └─────────────────────────────────────────┘          │
│                                                             │
│   直到: end_turn 或 达到轮次上限                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、核心实现

### 3.1 query() 主循环

```typescript
async function* query(options: QueryOptions): AsyncGenerator<Message> {
  for (;;) {
    // 1. 检查是否需要自动压缩
    if (shouldAutoCompact(messages)) {
      await autoCompact(messages)
    }

    // 2. 构建 API 请求
    const request = {
      model: options.model,
      system: buildSystemPrompt(),
      messages: messages,
      tools: buildToolsSchema(),
      max_tokens: options.maxTokens,
    }

    // 3. 调用 API (流式)
    const stream = await api.messages.stream(request)

    // 4. 处理流式响应
    let stopReason: string
    for await (const event of stream) {
      if (event.type === 'content_block_delta') {
        yield { type: 'assistant_delta', ...event }
      }
      if (event.type === 'message_stop') {
        stopReason = event.message.stop_reason
      }
    }

    // 5. 根据 stop_reason 决定下一步
    if (stopReason === 'end_turn') {
      yield { type: 'result', status: 'success' }
      break
    }

    if (stopReason === 'tool_use') {
      // 6. 执行工具
      const toolResults = await runTools(toolUseBlocks)

      // 7. 添加工具结果到消息历史
      messages.push({
        role: 'user',
        content: toolResults.map(r => ({
          type: 'tool_result',
          tool_use_id: r.id,
          content: r.output,
        }))
      })

      // 8. 继续循环
      continue
    }

    if (stopReason === 'max_tokens') {
      // 尝试增大 max_tokens 重试
      continue
    }
  }
}
```

### 3.2 关键状态变量

| 变量 | 类型 | 说明 |
|---|---|---|
| `messages` | `Message[]` | 对话历史（动态增长） |
| `stopReason` | `string` | API 返回的停止原因 |
| `toolUseBlocks` | `ToolUseBlock[]` | 本轮需要执行的工具调用 |
| `totalUsage` | `TokenUsage` | 累计 token 用量 |

### 3.3 stop_reason 处理

| stop_reason | 处理方式 |
|---|---|
| `end_turn` | 循环结束，返回结果 |
| `tool_use` | 执行工具，添加结果，继续循环 |
| `max_tokens` | 尝试增大输出上限重试 |
| `stop_sequence` | 遇到停止序列，结束 |
| `refusal` | 模型拒绝，结束 |

---

## 四、工具执行

### 4.1 runTools() 流程

```
runTools(toolUseBlocks)
  │
  ├─ 1. partitionToolCalls() — 分区
  │     ├─ 并发安全 (只读) → concurrent queue
  │     └─ 非安全 (写) → sequential queue
  │
  ├─ 2. 并发执行安全工具
  │     └─ Promise.all() 最大并发 10
  │
  ├─ 3. 顺序执行非安全工具
  │
  └─ 4. 收集结果 → tool_result messages
```

### 4.2 并发控制

```typescript
const MAX_CONCURRENCY = 10

async function runToolsConcurrently(tools: ToolCall[]) {
  const results = await Promise.all(
    tools.map(tool => 
      runSingleTool(tool).catch(error => ({
        tool_use_id: tool.id,
        content: `Error: ${error.message}`,
        is_error: true,
      }))
    )
  )
  return results
}
```

### 4.3 权限检查

每个工具执行前都会进行权限检查：

```
runSingleTool(tool)
  │
  ├─ canUseTool(tool) — 权限检查
  │   ├─ deny → 返回错误
  │   ├─ allow → 执行
  │   └─ ask → 弹窗确认
  │
  ├─ preToolUseHooks — 执行前 Hook
  │
  ├─ tool.run(input) — 实际执行
  │
  └─ postToolUseHooks — 执行后 Hook
```

---

## 五、QueryEngine 状态管理

### 5.1 类结构

```typescript
class QueryEngine {
  // 可变状态
  private mutableMessages: Message[] = []
  private abortController: AbortController
  private totalUsage: TokenUsage = { input: 0, output: 0 }

  // 权限拒绝记录
  private permissionDenials: PermissionDenial[] = []

  // 会话 ID
  private sessionId: string

  // 提交消息
  async *submitMessage(prompt: string): AsyncGenerator<SDKMessage> {
    // 1. 构建系统提示词
    const systemParts = await fetchSystemPromptParts()

    // 2. 处理用户输入
    const userMessage = await processUserInput(prompt)

    // 3. 持久化
    await recordTranscript(userMessage)

    // 4. 进入循环
    yield* query({ messages: this.mutableMessages, ... })

    // 5. 返回结果
    yield { type: 'result', ... }
  }
}
```

### 5.2 中断控制

```typescript
// 中断当前操作
abort() {
  this.abortController.abort()
}

// 在循环中检查中断
if (this.abortController.signal.aborted) {
  yield { type: 'result', status: 'aborted' }
  break
}
```

---

## 六、Thinking 模式

### 6.1 自适应 Thinking

```typescript
interface ThinkingConfig {
  type: 'enabled' | 'disabled' | 'adaptive'
  budget_tokens?: number
}

function getThinkingConfig(model: string): ThinkingConfig {
  if (modelSupportsThinking(model)) {
    return { type: 'adaptive' } // 根据任务复杂度自动启用
  }
  return { type: 'disabled' }
}
```

### 6.2 Thinking Block 处理

```
API 流式响应:
  ├─ content_block_start(type: 'thinking')
  ├─ content_block_delta(thinking_delta: '...')
  ├─ content_block_stop
  ├─ content_block_start(type: 'text')
  └─ ...

处理:
  ├─ thinking block → 不显示给用户，用于内部推理
  └─ text block → 显示给用户
```

---

## 七、max_tokens 恢复机制

### 7.1 问题场景

模型可能在输出过程中达到 `max_tokens` 限制，导致响应被截断。

### 7.2 恢复策略

```typescript
let retryCount = 0
const MAX_RETRIES = 3

async function handleMaxTokens(response: ApiResponse) {
  if (response.stop_reason === 'max_tokens' && retryCount < MAX_RETRIES) {
    retryCount++

    // 增大 max_tokens
    const newMaxTokens = Math.min(
      currentMaxTokens * 2,
      modelMaxOutputLimit
    )

    // 添加继续提示
    messages.push({
      role: 'user',
      content: 'Please continue.'
    })

    // 重试
    return query({ maxTokens: newMaxTokens, ... })
  }

  // 无法恢复，返回错误
  yield { type: 'result', status: 'max_tokens_exceeded' }
}
```

---

## 八、错误处理

### 8.1 错误类型

| 错误 | 处理方式 |
|---|---|
| API 错误 (4xx/5xx) | 重试或返回错误 |
| 网络超时 | 重试 |
| 权限拒绝 | 记录并返回 |
| 工具执行错误 | 作为 tool_result 返回 |
| 预算超限 | 停止循环 |

### 8.2 断路器

```typescript
// 连续压缩失败断路器
let consecutiveCompactFailures = 0
const MAX_COMPACT_FAILURES = 3

if (compactFailed) {
  consecutiveCompactFailures++
  if (consecutiveCompactFailures >= MAX_COMPACT_FAILURES) {
    // 停止尝试压缩，继续运行
  }
}
```

---

## 九、消息类型

### 9.1 消息结构

```typescript
type Message =
  | UserMessage
  | AssistantMessage
  | SystemMessage
  | ProgressMessage
  | StreamEvent

interface UserMessage {
  role: 'user'
  content: ContentBlock[]
}

interface AssistantMessage {
  role: 'assistant'
  content: ContentBlock[]
  stop_reason?: string
}

type ContentBlock =
  | TextBlock
  | ToolUseBlock
  | ToolResultBlock
  | ImageBlock
  | AttachmentBlock
```

### 9.2 消息流转

```
用户输入 → UserMessage
    ↓
API 调用 → AssistantMessage (含 ToolUseBlock)
    ↓
工具执行 → UserMessage (含 ToolResultBlock)
    ↓
API 调用 → AssistantMessage
    ↓
...循环...
    ↓
end_turn → 最终 AssistantMessage
```

---

## 十、设计评价

### 10.1 优点

1. **简洁的循环结构**：`for(;;)` 循环清晰表达 ReAct 模式
2. **流式输出**：AsyncGenerator 支持实时响应
3. **灵活的工具执行**：并发安全工具，顺序执行写工具
4. **健壮的错误处理**：多层重试和断路器

### 10.2 局限性

1. **状态管理复杂**：`mutableMessages` 需要小心维护
2. **压缩中断风险**：压缩失败可能导致上下文丢失
3. **并发限制**：最大并发 10 可能成为瓶颈

### 10.3 潜在改进

1. **更精细的并发控制**：根据工具类型动态调整并发数
2. **增量压缩**：边运行边压缩，减少一次性压缩开销
3. **工具结果缓存**：相同工具调用可复用结果

---

## 十一、总结

Agent 循环是 Claude Code 的核心，通过 ReAct 模式实现持续的任务执行能力。其设计要点：

1. **循环结构**：`for(;;)` + AsyncGenerator 实现流式 ReAct
2. **工具执行**：分区并发，权限检查，Hook 扩展
3. **状态管理**：QueryEngine 封装会话状态
4. **错误恢复**：max_tokens 恢复、断路器保护
5. **Thinking 支持**：自适应启用扩展思考

这个循环是整个系统的"心脏"，所有其他子系统（工具、权限、压缩）都围绕它运作。
