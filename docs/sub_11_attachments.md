# 子领域 11：附件系统

> Claude Code 的动态上下文注入机制

---

## 一、概述

附件系统是 Claude Code 的动态上下文注入机制，每轮对话时自动收集并注入相关上下文，包括文件引用、记忆、任务状态等。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/utils/attachments.ts` | 附件收集与注入 |
| `src/utils/fileReferenceParser.ts` | @文件引用解析 |

---

## 二、附件类型

### 2.1 类型列表

```typescript
type AttachmentType =
  | 'file_reference'      // @文件引用
  | 'ide_selection'       // IDE 选区
  | 'clipboard_image'     // 剪贴板图片
  | 'relevant_memory'     // 相关记忆
  | 'skill_discovery'     // 技能发现
  | 'task_state'          // 任务状态
  | 'mcp_resource'        // MCP 资源
  | 'agent_list'          // Agent 列表
  | 'mcp_instructions'    // MCP 指令增量
  | 'deferred_tools'      // 延迟工具
  | 'date_change'         // 日期变更
  | 'efficiency_reminder' // 效率提醒
  | 'post_compact_files'  // 压缩后恢复文件
```

### 2.2 附件结构

```typescript
interface Attachment {
  type: AttachmentType
  content: string | ImageBlock
  metadata?: Record<string, any>
}

interface ImageBlock {
  type: 'image'
  source: {
    type: 'base64'
    media_type: 'image/png' | 'image/jpeg' | 'image/gif'
    data: string  // base64
  }
}
```

---

## 三、getAttachments() 流程

### 3.1 主流程

```typescript
async function getAttachments(context: AttachmentContext): Promise<Attachment[]> {
  const attachments: Attachment[] = []

  // 1. @文件引用
  const fileRefs = await processFileReferences(context)
  attachments.push(...fileRefs)

  // 2. IDE 选区
  const ideSelection = await getIDESelection()
  if (ideSelection) attachments.push(ideSelection)

  // 3. 剪贴板图片
  const clipboardImage = await getClipboardImage()
  if (clipboardImage) attachments.push(clipboardImage)

  // 4. 相关记忆
  const memories = await findRelevantMemories(context)
  attachments.push(...memories)

  // 5. 技能发现
  const skills = await discoverSkills(context)
  attachments.push(...skills)

  // 6. 任务状态
  const tasks = await getTaskState()
  if (tasks) attachments.push(tasks)

  // 7. MCP 资源
  const mcpResources = await getMCPResources()
  attachments.push(...mcpResources)

  // 8. Agent 列表增量
  const agentDelta = await getAgentListDelta()
  if (agentDelta) attachments.push(agentDelta)

  // 9. MCP 指令增量
  const mcpDelta = await getMCPInstructionsDelta()
  if (mcpDelta) attachments.push(mcpDelta)

  // 10. 延迟工具
  const deferred = await getDeferredTools()
  if (deferred) attachments.push(deferred)

  // 11. 日期变更
  if (hasDateChanged(context.lastDate)) {
    attachments.push({ type: 'date_change', content: `Today is ${getCurrentDate()}` })
  }

  // 12. 效率提醒
  if (shouldInjectEfficiencyReminder(context)) {
    attachments.push({ type: 'efficiency_reminder', content: getEfficiencyReminder() })
  }

  // 13. 压缩后恢复文件
  const postCompact = await getPostCompactFiles(context)
  attachments.push(...postCompact)

  // 去重
  return filterDuplicateAttachments(attachments)
}
```

---

## 四、文件引用处理

### 4.1 @文件引用解析

```typescript
// 用户输入: "请修改 @src/main.ts 的代码"

function parseFileReferences(input: string): FileReference[] {
  const pattern = /@([^\s]+)/g
  const refs: FileReference[] = []

  let match
  while ((match = pattern.exec(input)) !== null) {
    refs.push({
      path: match[1],
      position: { start: match.index, end: match.index + match[0].length },
    })
  }

  return refs
}
```

### 4.2 文件读取与注入

```typescript
async function processFileReferences(
  context: AttachmentContext
): Promise<Attachment[]> {
  const refs = parseFileReferences(context.userInput)
  const attachments: Attachment[] = []

  for (const ref of refs) {
    try {
      const content = await readFile(ref.path)

      // 截断大文件
      const truncated = content.length > MAX_FILE_SIZE
        ? content.slice(0, MAX_FILE_SIZE) + '\n... [truncated]'
        : content

      attachments.push({
        type: 'file_reference',
        content: `<file path="${ref.path}">\n${truncated}\n</file>`,
        metadata: { path: ref.path, truncated },
      })
    } catch (error) {
      attachments.push({
        type: 'file_reference',
        content: `<file path="${ref.path}">\nError: ${error.message}\n</file>`,
        metadata: { path: ref.path, error: true },
      })
    }
  }

  return attachments
}
```

---

## 五、相关记忆注入

### 5.1 语义搜索

```typescript
async function findRelevantMemories(
  context: AttachmentContext
): Promise<Attachment[]> {
  // 收集语义信号
  const signals = collectSemanticSignals(context)

  // 在记忆目录中搜索
  const memories = await searchMemories(signals, {
    maxResults: 5,
    maxBytesPerResult: 4000,
    maxBytesPerSession: 60000,
  })

  // 去重
  const deduped = filterDuplicateMemoryAttachments(memories, context)

  return deduped.map(m => ({
    type: 'relevant_memory',
    content: m.content,
    metadata: { path: m.path, score: m.score },
  }))
}

function collectSemanticSignals(context: AttachmentContext): string[] {
  const signals: string[] = []

  // 最近成功的工具调用
  for (const call of context.recentToolCalls) {
    if (call.success && call.semanticContent) {
      signals.push(call.semanticContent)
    }
  }

  // 用户输入关键词
  signals.push(...extractKeywords(context.userInput))

  return signals
}
```

### 5.2 预算控制

```typescript
const MEMORY_BUDGET = {
  maxFiles: 5,           // 每轮最多 5 个文件
  maxBytesPerFile: 4000, // 每个文件最多 4KB
  maxBytesPerSession: 60000, // 每会话累计最多 60KB
}

let sessionMemoryUsage = 0

function checkMemoryBudget(bytes: number): boolean {
  return sessionMemoryUsage + bytes <= MEMORY_BUDGET.maxBytesPerSession
}

function recordMemoryUsage(bytes: number): void {
  sessionMemoryUsage += bytes
}
```

---

## 六、技能发现

### 6.1 自动发现

```typescript
async function discoverSkills(
  context: AttachmentContext
): Promise<Attachment[]> {
  // 根据用户输入匹配技能
  const matchedSkills = await matchSkills(context.userInput, {
    maxResults: 3,
    minScore: 0.7,
  })

  return matchedSkills.map(skill => ({
    type: 'skill_discovery',
    content: `Relevant skill available: ${skill.name}\n${skill.description}`,
    metadata: { skillName: skill.name, score: skill.score },
  }))
}
```

### 6.2 技能激活建议

```typescript
// 注入建议
<system-reminder>
The following skill may be relevant to your task:
- debug: Helps investigate and fix bugs in code
Use /skill debug to activate.
</system-reminder>
```

---

## 七、图片处理

### 7.1 剪贴板图片

```typescript
async function getClipboardImage(): Promise<Attachment | null> {
  // 检查剪贴板是否有图片
  const hasImage = await checkClipboardImage()
  if (!hasImage) return null

  // 读取图片
  const image = await readClipboardImage()

  // 压缩
  const compressed = await maybeResizeAndDownsampleImageBlock(image)

  return {
    type: 'clipboard_image',
    content: compressed,
    metadata: { originalSize: image.length, compressedSize: compressed.length },
  }
}
```

### 7.2 图片压缩

```typescript
async function maybeResizeAndDownsampleImageBlock(
  image: ImageBlock
): Promise<ImageBlock> {
  const maxBytes = 500000  // 500KB
  const maxDimension = 2048

  // 如果已经足够小，直接返回
  if (image.source.data.length <= maxBytes) {
    return image
  }

  // 解码图片
  const decoded = await decodeImage(image.source.data)

  // 计算新尺寸
  const scale = Math.min(
    maxDimension / decoded.width,
    maxDimension / decoded.height,
    Math.sqrt(maxBytes / image.source.data.length)
  )

  // 缩放
  const resized = await resizeImage(decoded, {
    width: Math.floor(decoded.width * scale),
    height: Math.floor(decoded.height * scale),
  })

  // 重新编码
  const encoded = await encodeImage(resized, { quality: 0.8 })

  return {
    type: 'image',
    source: {
      type: 'base64',
      media_type: 'image/jpeg',
      data: encoded,
    },
  }
}
```

---

## 八、增量更新

### 8.1 Agent 列表增量

```typescript
let lastAgentList: Agent[] = []

async function getAgentListDelta(): Promise<Attachment | null> {
  const currentList = await getAvailableAgents()

  const added = currentList.filter(a => !lastAgentList.some(l => l.id === a.id))
  const removed = lastAgentList.filter(a => !currentList.some(l => l.id === a.id))

  if (added.length === 0 && removed.length === 0) {
    return null
  }

  lastAgentList = currentList

  let content = 'Agent list updated:\n'
  if (added.length > 0) {
    content += `Added: ${added.map(a => a.name).join(', ')}\n`
  }
  if (removed.length > 0) {
    content += `Removed: ${removed.map(a => a.name).join(', ')}\n`
  }

  return {
    type: 'agent_list',
    content,
    metadata: { added, removed },
  }
}
```

### 8.2 MCP 指令增量

```typescript
let lastMCPInstructions: MCPInstruction[] = []

async function getMCPInstructionsDelta(): Promise<Attachment | null> {
  const current = await getMCPInstructions()

  const delta = computeDelta(lastMCPInstructions, current)

  if (delta.added.length === 0 && delta.removed.length === 0) {
    return null
  }

  lastMCPInstructions = current

  return {
    type: 'mcp_instructions',
    content: formatMCPDelta(delta),
    metadata: delta,
  }
}
```

---

## 九、去重机制

### 9.1 记忆去重

```typescript
function filterDuplicateMemoryAttachments(
  memories: Memory[],
  context: AttachmentContext
): Memory[] {
  // 检查是否已注入
  const injectedPaths = new Set(
    context.previousAttachments
      .filter(a => a.type === 'relevant_memory')
      .map(a => a.metadata?.path)
  )

  return memories.filter(m => !injectedPaths.has(m.path))
}
```

### 9.2 文件引用去重

```typescript
function filterDuplicateFileReferences(
  refs: FileReference[],
  context: AttachmentContext
): FileReference[] {
  const injectedPaths = new Set(
    context.previousAttachments
      .filter(a => a.type === 'file_reference')
      .map(a => a.metadata?.path)
  )

  return refs.filter(r => !injectedPaths.has(r.path))
}
```

---

## 十、注入方式

### 10.1 system-reminder 标签

```typescript
function formatAttachment(attachment: Attachment): string {
  return `<system-reminder type="${attachment.type}">
${attachment.content}
</system-reminder>`
}

// 示例输出
<system-reminder type="file_reference">
<file path="src/main.ts">
import { app } from './app'

app.listen(3000)
</file>
</system-reminder>

<system-reminder type="relevant_memory">
User prefers TypeScript over JavaScript.
</system-reminder>
```

### 10.2 消息注入位置

```typescript
function injectAttachments(
  messages: Message[],
  attachments: Attachment[]
): Message[] {
  if (attachments.length === 0) return messages

  // 注入到最新用户消息前
  const lastUserIndex = messages.findLastIndex(m => m.role === 'user')

  const attachmentMessage: Message = {
    role: 'user',
    content: attachments.map(formatAttachment).join('\n\n'),
    type: 'attachment_batch',
  }

  const result = [...messages]
  result.splice(lastUserIndex, 0, attachmentMessage)

  return result
}
```

---

## 十一、设计权衡

### 11.1 预算 vs 完整性

| 决策 | 收益 | 代价 |
|---|---|---|
| 记忆限制 | 控制 token 消耗 | 可能遗漏相关信息 |
| 文件截断 | 避免大文件 | 丢失部分内容 |
| 图片压缩 | 降低 token | 降低图片质量 |

### 11.2 实时性 vs 效率

| 决策 | 收益 | 代价 |
|---|---|---|
| 增量更新 | 减少重复注入 | 需要维护状态 |
| 预取 | 降低延迟 | 可能预取不必要内容 |

---

## 十二、总结

附件系统是 Claude Code 上下文动态性的核心：

1. **13 种附件类型**：文件、记忆、技能、任务等
2. **预算控制**：防止 token 爆炸
3. **增量更新**：减少重复注入
4. **自动发现**：技能和记忆自动匹配
5. **去重机制**：避免重复内容

附件系统让每轮对话都能获得最相关的上下文。
