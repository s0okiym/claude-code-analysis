# 子领域 05：上下文压缩

> Claude Code 的五层压缩策略与实现细节

---

## 一、概述

对话历史会持续增长，Claude Code 通过五层压缩策略管理上下文窗口，确保长对话不爆 token 限制。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/services/compact/` | 压缩核心实现 |
| `src/services/compact/prompt.ts` | 压缩 Prompt |
| `src/services/compact/conversationCompact.ts` | 对话压缩 |
| `src/query.ts` | 自动压缩触发 |

---

## 二、五层压缩策略

### 2.1 压缩层次

| 层级 | 名称 | 触发条件 | 机制 |
|---|---|---|---|
| 1 | 工具结果清除 | 每轮微压缩 | 保留最近 N 个 |
| 2 | 自动压缩 | token 超阈值 | Fork 子 Agent 生成摘要 |
| 3 | Snip 压缩 | 长时间 SDK 会话 | 按边界标记删除 |
| 4 | 上下文折叠 | 实验功能 | 完整上下文管理 |
| 5 | Reactive Compact | API 返回 413 | 紧急压缩 |

### 2.2 触发流程

```
对话消息增长
  │
  ▼
┌──────────────────────────┐
│ Token 预算检查              │
│ vs effectiveContextWindow │
└──────────┬───────────────┘
           │ 超阈值
           ▼
┌──────────────────────────┐
│ 自动压缩 (autoCompact)     │ ← Layer 2
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│ 微压缩 (microCompact)      │ ← Layer 1
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│ Snip 压缩 (feature flag)  │ ← Layer 3
└──────────┬───────────────┘
           │
           ▼
       继续对话
```

---

## 三、工具结果清除

### 3.1 机制

```typescript
function microCompact(messages: Message[], keepRecent: number = 5): Message[] {
  let toolResultCount = 0

  return messages.map(msg => {
    if (msg.role === 'user' && msg.content.some(c => c.type === 'tool_result')) {
      toolResultCount++
      if (toolResultCount > keepRecent) {
        // 替换为清除标记
        return {
          ...msg,
          content: msg.content.map(c => 
            c.type === 'tool_result'
              ? { ...c, content: '[Old tool result content cleared]' }
              : c
          ),
        }
      }
    }
    return msg
  })
}
```

### 3.2 效果示例

```
压缩前 (180K tokens):
  [user] "帮我找 bug"
  [assistant] tool_use: GrepTool("error")
  [user] tool_result: "1000行 grep 输出..."     ← 占用大量 token
  [assistant] "找到了，让我看看..."
  [user] tool_result: "500行文件内容..."         ← 占用大量 token
  [assistant] tool_use: GrepTool("fix")
  [user] tool_result: "200行 grep 输出..."       ← 最近，保留

微压缩后:
  [user] "帮我找 bug"
  [assistant] tool_use: GrepTool("error")
  [user] tool_result: [Old tool result content cleared]   ← 已清除
  [assistant] "找到了，让我看看..."
  [user] tool_result: [Old tool result content cleared]   ← 已清除
  [assistant] tool_use: GrepTool("fix")
  [user] tool_result: "200行 grep 输出..."               ← 保留
```

---

## 四、自动压缩

### 4.1 触发检查

```typescript
function shouldAutoCompact(
  messages: Message[],
  contextWindow: number,
  reservedForSummary: number
): boolean {
  const currentTokens = tokenCountWithEstimation(messages)
  const threshold = contextWindow - reservedForSummary - 13000

  return currentTokens >= threshold
}
```

### 4.2 压缩流程

```typescript
async function autoCompactIfNeeded(
  messages: Message[],
  options: CompactOptions
): Promise<CompactResult> {
  if (!shouldAutoCompact(messages, options.contextWindow, options.reservedForSummary)) {
    return { compacted: false }
  }

  // 尝试会话记忆压缩
  const memoryResult = await trySessionMemoryCompaction(messages)
  if (memoryResult.success) {
    return { compacted: true, method: 'memory' }
  }

  // 执行对话压缩
  const result = await compactConversation(messages, options)

  return { compacted: true, method: 'summary', ...result }
}
```

### 4.3 compactConversation() 核心实现

```typescript
async function compactConversation(
  messages: Message[],
  options: CompactOptions
): Promise<CompactResult> {
  // 1. 移除图片（节省大量 token）
  const messagesWithoutImages = stripImagesFromMessages(messages)

  // 2. Fork 子 Agent 生成摘要
  const summary = await forkCompactAgent(messagesWithoutImages, options)

  // 3. 构建压缩后的消息列表
  const postCompactMessages = buildPostCompactMessages(
    messages,
    summary,
    options
  )

  // 4. 后清理
  await postCompactCleanup(messages, postCompactMessages)

  return {
    summary,
    originalTokenCount: tokenCountWithEstimation(messages),
    newTokenCount: tokenCountWithEstimation(postCompactMessages),
  }
}
```

### 4.4 压缩 Prompt

```typescript
const COMPACT_PROMPT = `
You are a conversation summarizer. Your task is to create a structured summary of the conversation.

Analyze the conversation and produce a summary with these sections:

<analysis>
[Your step-by-step analysis of the conversation]
</analysis>

<summary>
## 1. Primary Request and Intent
[What the user wanted to accomplish]

## 2. Key Technical Concepts
[Important technical concepts discussed]

## 3. Files and Code Segments
[Key files and code sections worked on]

## 4. Errors and Fixes
[Errors encountered and how they were resolved]

## 5. Problem-Solving Progress
[What was tried and what worked/didn't work]

## 6. All User Messages
[List of all user messages in order]

## 7. Pending Tasks
[Tasks that are not yet completed]

## 8. Current Work
[What is being worked on right now]

## 9. Optional Next Steps
[Suggested next steps if applicable]
</summary>
`
```

### 4.5 构建压缩后消息

```typescript
function buildPostCompactMessages(
  originalMessages: Message[],
  summary: string,
  options: CompactOptions
): Message[] {
  const result: Message[] = []

  // 1. 压缩边界标记
  result.push({
    role: 'system',
    type: 'compact_boundary',
    content: JSON.stringify({
      compactedAt: new Date().toISOString(),
      originalTokenCount: tokenCountWithEstimation(originalMessages),
    }),
  })

  // 2. 摘要消息
  result.push({
    role: 'user',
    type: 'compact_summary',
    content: `<compact_summary>\n${summary}\n</compact_summary>`,
  })

  // 3. 恢复最近修改的文件
  const recentFiles = getRecentModifiedFiles(originalMessages, {
    maxFiles: 5,
    maxTokensPerFile: 5000,
  })
  for (const file of recentFiles) {
    result.push({
      role: 'user',
      type: 'file_context',
      content: `Recent file: ${file.path}\n\`\`\`\n${file.content}\n\`\`\``,
    })
  }

  // 4. 保留最近的几条消息
  const recentMessages = originalMessages.slice(-options.keepRecentMessages)
  result.push(...recentMessages)

  return result
}
```

---

## 五、会话记忆压缩

### 5.1 机制

将重要信息存入记忆文件而非摘要：

```typescript
async function trySessionMemoryCompaction(
  messages: Message[]
): Promise<{ success: boolean }> {
  // 提取重要信息
  const importantInfo = extractImportantInfo(messages)

  // 写入记忆文件
  for (const info of importantInfo) {
    await writeMemoryFile({
      type: info.type, // user/feedback/project/reference
      content: info.content,
      metadata: {
        extractedAt: new Date().toISOString(),
        sessionContext: info.context,
      },
    })
  }

  return { success: importantInfo.length > 0 }
}
```

### 5.2 信息提取

```typescript
function extractImportantInfo(messages: Message[]): ImportantInfo[] {
  const result: ImportantInfo[] = []

  // 提取用户偏好
  const preferences = extractUserPreferences(messages)
  result.push(...preferences.map(p => ({
    type: 'user',
    content: p,
  })))

  // 提取反馈
  const feedback = extractUserFeedback(messages)
  result.push(...feedback.map(f => ({
    type: 'feedback',
    content: f,
  })))

  // 提取项目知识
  const projectInfo = extractProjectInfo(messages)
  result.push(...projectInfo.map(i => ({
    type: 'project',
    content: i,
  })))

  return result
}
```

---

## 六、Snip 压缩

### 6.1 机制

```typescript
async function snipCompactIfNeeded(
  messages: Message[]
): Promise<boolean> {
  // 检测 snip boundary message
  const snipIndex = messages.findIndex(m => 
    m.type === 'snip_boundary'
  )

  if (snipIndex === -1) return false

  // 移除边界之前的所有消息
  const removed = messages.splice(0, snipIndex + 1)

  // 清除过期标记
  clearExpiredMarkers(removed)

  return true
}
```

### 6.2 适用场景

- 长时间 SDK 会话
- 防止 `mutableMessages` 无限增长（内存泄漏防护）
- 更激进的历史裁剪

---

## 七、上下文折叠

### 7.1 机制（实验性）

```typescript
const CONTEXT_COLLAPSE_THRESHOLDS = {
  startSubmitting: 0.90,  // 90% 使用率
  blockNewTools: 0.95,    // 95% 使用率
}

async function checkContextCollapse(
  tokenUsage: TokenUsage,
  contextWindow: number
): Promise<CollapseAction> {
  const usage = tokenUsage.input / contextWindow

  if (usage >= CONTEXT_COLLAPSE_THRESHOLDS.blockNewTools) {
    return 'block_tools'
  }

  if (usage >= CONTEXT_COLLAPSE_THRESHOLDS.startSubmitting) {
    return 'submit_context'
  }

  return 'none'
}
```

---

## 八、Reactive Compact

### 8.1 触发场景

API 返回 `prompt_too_long` 错误（413 响应）时：

```typescript
async function handlePromptTooLong(
  error: APIError,
  messages: Message[],
  options: QueryOptions
): Promise<void> {
  if (error.statusCode !== 413) {
    throw error
  }

  // 紧急压缩
  await compactConversation(messages, {
    ...options,
    aggressiveRatio: 0.5, // 压缩到 50%
  })

  // 重试请求
  return query(options)
}
```

---

## 九、压缩效果评估

### 9.1 Token 回收

```
典型压缩效果:
  压缩前: 180K tokens
  压缩后: ~30K tokens
  回收率: ~83%

摘要大小: ~10K tokens
恢复文件: ~15K tokens (5个文件 × 3K)
最近消息: ~5K tokens
```

### 9.2 信息保留

| 类型 | 保留策略 |
|---|---|
| 用户意图 | 摘要第1节 |
| 技术概念 | 摘要第2节 |
| 文件代码 | 摘要第3节 + 恢复文件 |
| 错误修复 | 摘要第4节 |
| 用户消息 | 摘要第6节 |
| 待办任务 | 摘要第7节 |
| 当前工作 | 摘要第8节 |

---

## 十、断路器保护

### 10.1 连续失败保护

```typescript
let consecutiveCompactFailures = 0
const MAX_COMPACT_FAILURES = 3

async function safeCompact(
  messages: Message[],
  options: CompactOptions
): Promise<CompactResult> {
  if (consecutiveCompactFailures >= MAX_COMPACT_FAILURES) {
    // 停止尝试压缩，继续运行
    return { compacted: false, reason: 'circuit_breaker' }
  }

  try {
    const result = await compactConversation(messages, options)
    consecutiveCompactFailures = 0
    return result
  } catch (error) {
    consecutiveCompactFailures++
    console.error(`Compact failed (${consecutiveCompactFailures}/${MAX_COMPACT_FAILURES})`, error)
    return { compacted: false, reason: 'error', error }
  }
}
```

---

## 十一、设计权衡

### 11.1 压缩粒度 vs 信息保留

| 策略 | 优点 | 缺点 |
|---|---|---|
| 工具结果清除 | 轻量，快速 | 丢失工具结果细节 |
| 摘要压缩 | 全面压缩 | 可能丢失细节，耗时 |
| Snip 压缩 | 快速，激进 | 丢失大量历史 |

### 11.2 Fork vs 内联

```typescript
// Fork 方式 (当前实现)
const summary = await forkAgent({
  task: 'compact',
  messages: messagesWithoutImages,
  timeout: 60000,
})

// 内联方式 (未采用)
const summary = await compactInCurrentContext(messages)
```

**Fork 优势**：
- 隔离压缩上下文
- 不污染主会话
- 可并行执行

---

## 十二、总结

压缩是长对话的生命线：

1. **五层策略**：从轻到重逐层触发
2. **摘要结构化**：9节摘要确保完整性
3. **恢复机制**：压缩后恢复关键文件
4. **断路器保护**：连续失败时停止尝试
5. **Fork 隔离**：子 Agent 执行，不阻塞主循环

压缩系统在信息保留和 token 效率之间找到了平衡点。
