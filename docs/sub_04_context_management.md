# 子领域 04：上下文管理

> Claude Code 的 Context 构建、增长与监控机制

---

## 一、概述

Context（上下文）是发送给 API 的所有信息，决定了模型"看到"什么。Claude Code 的上下文管理涉及构建、增长、监控和压缩四个阶段。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/context.ts` | Context 收集与构建 |
| `src/utils/context.ts` | Context 窗口计算 |
| `src/utils/tokenBudget.ts` | Token 预算管理 |
| `src/utils/attachments.ts` | 附件系统 |

---

## 二、Context 构成

### 2.1 四层结构

| 层面 | 内容 | 缓存策略 |
|---|---|---|
| System Prompt | 角色定义、工具指南 | 静态区可跨用户缓存 |
| User Context | CLAUDE.md 指令、日期 | 会话级缓存 |
| System Context | Git 状态 | 会话级缓存 |
| Messages | 对话历史 | 压缩/截断管理 |

### 2.2 Token 分配

```
┌───────────────────────────────────────────────┐
│              200K Token Context Window         │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │         System Prompt (~15K)            │  │
│  │  - 角色定义                              │  │
│  │  - 工具使用指南                          │  │
│  │  - 环境信息                              │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │         User Context (~5K)              │  │
│  │  - CLAUDE.md 指令                        │  │
│  │  - 当前日期                              │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │        System Context (~2K)             │  │
│  │  - Git 状态                              │  │
│  │  - 缓存破坏标记                          │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │         Messages (动态增长)              │  │
│  │  - 用户消息                              │  │
│  │  - 助手响应                              │  │
│  │  - 工具结果                              │  │
│  │  - 附件                                  │  │
│  │  ...                                    │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │     Reserved for Output (~20K)          │  │
│  └─────────────────────────────────────────┘  │
└───────────────────────────────────────────────┘
```

---

## 三、System Prompt 构建

### 3.1 静态区与动态区

```
┌──────────────────────────────┐
│         静态区 (Static)        │  ← 可跨用户缓存 (global scope)
│                              │
│ 1. 角色定义 (Intro)            │
│ 2. 系统规则 (System)           │
│ 3. 任务执行 (Doing Tasks)      │
│ 4. 行为准则 (Actions)          │
│ 5. 工具使用 (Using Tools)      │
│ 6. 语气风格 (Tone & Style)     │
│ 7. 输出效率 (Output)           │
│                              │
├── SYSTEM_PROMPT_DYNAMIC_BOUNDARY ──┤
│                              │
│         动态区 (Dynamic)        │  ← 会话/用户特定
│                              │
│ 8. 会话指导 (Session Guidance) │
│ 9. 记忆系统 (Memory)          │
│10. 环境信息 (Environment)      │
│11. 语言偏好 (Language)         │
│12. MCP 指令 (MCP Instructions)│
│13. Token 预算说明             │
│                              │
└──────────────────────────────┘
```

### 3.2 构建优先级

```typescript
function buildEffectiveSystemPrompt(): string {
  // 优先级 0: Override (完全替换)
  if (hasOverride) {
    return overridePrompt
  }

  // 优先级 1: Coordinator 模式
  if (coordinatorMode) {
    return coordinatorPrompt
  }

  // 优先级 2: Agent 系统提示词
  let prompt = agentPrompt || defaultPrompt

  // 优先级 3: 自定义系统提示词
  if (customSystemPrompt) {
    prompt = customSystemPrompt
  }

  // 优先级 4: 追加内容
  prompt += appendSystemPrompt

  return prompt
}
```

---

## 四、User Context 收集

### 4.1 CLAUDE.md 加载层次

```
优先级 1 (最低): Managed Memory
  └─ /etc/claude-code/CLAUDE.md — 企业管理员全局指令

优先级 2: User Memory
  └─ ~/.claude/CLAUDE.md — 用户私人全局指令

优先级 3: Project Memory
  ├─ CLAUDE.md — 项目根目录
  ├─ .claude/CLAUDE.md — 项目 .claude 目录
  └─ .claude/rules/*.md — 项目规则文件

优先级 4 (最高): Local Memory
  └─ CLAUDE.local.md — 私人项目指令
```

### 4.2 发现机制

```typescript
async function getClaudeMds(): Promise<ClaudeMd[]> {
  const results: ClaudeMd[] = []

  // 从当前目录向上遍历
  let currentDir = process.cwd()
  while (currentDir !== '/') {
    // 检查各层级的 CLAUDE.md
    const possibleFiles = [
      path.join(currentDir, 'CLAUDE.md'),
      path.join(currentDir, '.claude', 'CLAUDE.md'),
      path.join(currentDir, 'CLAUDE.local.md'),
    ]

    for (const file of possibleFiles) {
      if (await exists(file)) {
        const content = await readFile(file)
        results.push({
          path: file,
          content: await processIncludes(content), // 处理 @include
        })
      }
    }

    currentDir = path.dirname(currentDir)
  }

  // 添加用户全局和系统全局
  // ...

  return results
}
```

### 4.3 @include 指令

```markdown
# CLAUDE.md

@include ./shared/instructions.md
@include ~/.claude/common.md
```

```typescript
async function processIncludes(content: string): Promise<string> {
  const includePattern = /@include\s+([^\n]+)/g

  let result = content
  let match
  while ((match = includePattern.exec(content)) !== null) {
    const includePath = match[1].trim()
    const includedContent = await readFile(includePath)
    result = result.replace(match[0], includedContent)
  }

  // 循环引用保护
  if (detectCircularInclude(result)) {
    throw new Error('Circular include detected')
  }

  return result
}
```

---

## 五、System Context 收集

### 5.1 Git 状态获取

```typescript
async function getSystemContext(): Promise<SystemContext> {
  const isGitRepo = await isGitRepository()

  if (!isGitRepo) {
    return { gitStatus: null, cacheBreaker: null }
  }

  // 并行获取 Git 信息
  const [branch, mainBranch, status, commits, userName] = await Promise.all([
    git('rev-parse --abbrev-ref HEAD'),
    getMainBranch(),
    git('status --short').then(s => s.slice(0, 2000)), // 截断
    git('log -5 --oneline'),
    git('config user.name'),
  ])

  return {
    gitStatus: {
      branch,
      mainBranch,
      status,
      recentCommits: commits,
      userName,
    },
    cacheBreaker: getCacheBreaker(),
  }
}
```

### 5.2 Memoize 缓存

```typescript
// 整个会话只计算一次
const getSystemContext = memoize(async () => {
  return await fetchSystemContext()
})

const getUserContext = memoize(async () => {
  return await fetchUserContext()
})
```

**关键设计**: Git 状态是**快照**，不会在对话中更新。

---

## 六、附件系统

### 6.1 附件类型

```typescript
interface Attachment {
  type: string
  content: string | ImageBlock
  metadata?: Record<string, any>
}

type AttachmentType =
  | 'file_reference'      // @文件引用
  | 'ide_selection'       // IDE 选区
  | 'clipboard_image'     // 剪贴板图片
  | 'relevant_memory'     // 相关记忆
  | 'skill_discovery'     // 技能发现
  | 'task_state'          // 任务状态
  | 'mcp_resource'        // MCP 资源
  | 'agent_list'          // Agent 列表
  | 'mcp_instructions'    // MCP 指令
  | 'deferred_tools'      // 延迟工具
  | 'date_change'         // 日期变更
  | 'efficiency_reminder' // 效率提醒
  | 'post_compact_files'  // 压缩后恢复文件
```

### 6.2 getAttachments() 流程

```typescript
async function getAttachments(context: AttachmentContext): Promise<Attachment[]> {
  const attachments: Attachment[] = []

  // 1. @文件引用
  const fileRefs = parseFileReferences(context.userInput)
  for (const ref of fileRefs) {
    const content = await readFile(ref.path)
    attachments.push({
      type: 'file_reference',
      content: content.slice(0, MAX_FILE_ATTACHMENT_SIZE),
      metadata: { path: ref.path },
    })
  }

  // 2. 相关记忆
  const memories = await findRelevantMemories(context.recentToolCalls)
  attachments.push(...memories.map(m => ({
    type: 'relevant_memory',
    content: m.content.slice(0, 4000),
    metadata: { path: m.path },
  })))

  // 3. 技能发现
  const skills = await discoverSkills(context.userInput)
  attachments.push(...skills.map(s => ({
    type: 'skill_discovery',
    content: s.description,
    metadata: { skillName: s.name },
  })))

  // 4. MCP 资源
  const mcpResources = await prefetchAllMcpResources(context.mcpClients)
  attachments.push(...mcpResources)

  // 5. 日期变更检测
  if (hasDateChanged(context.lastDate)) {
    attachments.push({
      type: 'date_change',
      content: `Today's date is ${getCurrentDate()}.`,
    })
  }

  // 去重
  return filterDuplicateAttachments(attachments)
}
```

### 6.3 附件预算控制

```typescript
const ATTACHMENT_BUDGETS = {
  relevant_memory: {
    maxFiles: 5,
    maxBytesPerFile: 4000,
    maxBytesPerSession: 60000,
  },
  file_reference: {
    maxBytes: 50000,
  },
  skill_discovery: {
    maxSkills: 3,
    maxBytesPerSkill: 5000,
  },
}
```

---

## 七、Token 监控

### 7.1 Context 窗口计算

```typescript
function getContextWindowForModel(model: string, betas: string[]): number {
  // 环境变量覆盖
  if (process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS) {
    return parseInt(process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS)
  }

  // 1M 后缀检测
  if (model.includes('[1m]')) {
    return 1_000_000
  }

  // 模型能力查询
  const capability = getModelCapability(model)
  if (capability?.contextWindow) {
    return capability.contextWindow
  }

  // Beta header 检测
  if (betas.includes('context-1m-beta')) {
    return 1_000_000
  }

  // 默认值
  return 200_000
}
```

### 7.2 阈值体系

```typescript
function calculateTokenWarningState(
  currentTokens: number,
  contextWindow: number,
  reservedForSummary: number
): TokenWarningState {
  const effectiveWindow = contextWindow - reservedForSummary

  // 各级阈值
  const autoCompactThreshold = effectiveWindow - 13000
  const warningThreshold = autoCompactThreshold - 20000
  const errorThreshold = warningThreshold - 20000
  const blockingLimit = effectiveWindow - 3000

  return {
    isAboveWarningThreshold: currentTokens >= warningThreshold,
    isAboveErrorThreshold: currentTokens >= errorThreshold,
    isAboveAutoCompactThreshold: currentTokens >= autoCompactThreshold,
    isAtBlockingLimit: currentTokens >= blockingLimit,
    warningThreshold,
    errorThreshold,
    autoCompactThreshold,
    blockingLimit,
  }
}
```

### 7.3 Token 计数

```typescript
// 实时估算
function tokenCountWithEstimation(messages: Message[]): number {
  let total = 0

  for (const msg of messages) {
    if (msg.cachedTokens) {
      total += msg.cachedTokens
    } else {
      // 本地估算
      total += estimateTokens(msg.content)
    }
  }

  return total
}

// API 精确计数
function tokenUsageFromAPI(response: ApiResponse): TokenUsage {
  return {
    input: response.usage.input_tokens,
    output: response.usage.output_tokens,
    cacheRead: response.usage.cache_read_input_tokens,
    cacheWrite: response.usage.cache_creation_input_tokens,
  }
}
```

---

## 八、Token 预算系统

### 8.1 用户指定目标

```typescript
interface TokenBudget {
  targetTokens: number  // 用户目标
  usedTokens: number    // 已使用
  diminishingReturns: number  // 连续低增量轮数
}

function checkTokenBudget(budget: TokenBudget): 'continue' | 'stop' {
  const percentage = budget.usedTokens / budget.targetTokens

  // 已用 < 90% 目标
  if (percentage < 0.9) {
    // 检查收益递减
    if (budget.diminishingReturns >= 3) {
      return 'stop'
    }
    return 'continue'
  }

  // 已用 ≥ 90% 目标
  return 'stop'
}
```

### 8.2 效率提醒注入

```typescript
function maybeInjectEfficiencyReminder(
  budget: TokenBudget,
  messages: Message[]
): void {
  if (!budget.targetTokens) return

  const percentage = Math.floor(budget.usedTokens / budget.targetTokens * 100)

  if (percentage >= 50 && percentage % 25 === 0) {
    messages.push({
      role: 'user',
      content: `You've used ${percentage}% of your token budget.`
    })
  }
}
```

---

## 九、Prompt Cache 优化

### 9.1 缓存策略

```typescript
// 静态区缓存
const STATIC_PROMPT_SECTIONS: SystemPromptSection[] = [
  { name: 'intro', cacheScope: 'global' },
  { name: 'system', cacheScope: 'global' },
  { name: 'doing_tasks', cacheScope: 'global' },
  { name: 'actions', cacheScope: 'global' },
  { name: 'using_tools', cacheScope: 'global' },
  { name: 'tone_style', cacheScope: 'global' },
  { name: 'output', cacheScope: 'global' },
]

// 动态区缓存
const DYNAMIC_PROMPT_SECTIONS: SystemPromptSection[] = [
  { name: 'session_guidance', cacheScope: 'session' },
  { name: 'memory', cacheScope: 'session' },
  { name: 'environment', cacheScope: 'session' },
  // ...
]
```

### 9.2 工具排序

```typescript
// 工具按名称排序，保证缓存前缀稳定
function sortToolsForCache(tools: Tool[]): Tool[] {
  return tools.sort((a, b) => a.name.localeCompare(b.name))
}
```

### 9.3 Delta 模式

```typescript
// MCP 指令使用 delta 模式，避免每次重连破坏缓存
interface MCPInstructionsDelta {
  added: MCPInstruction[]
  removed: string[]
}

function buildMCPInstructionsDelta(
  previous: MCPInstruction[],
  current: MCPInstruction[]
): MCPInstructionsDelta {
  return {
    added: current.filter(c => !previous.some(p => p.name === c.name)),
    removed: previous.filter(p => !current.some(c => c.name === p.name)).map(p => p.name),
  }
}
```

---

## 十、设计权衡

### 10.1 信息完整性 vs Token 效率

| 决策 | 收益 | 代价 |
|---|---|---|
| CLAUDE.md 截断 | 限制 token 消耗 | 可能丢失信息 |
| Git 状态快照 | 降低延迟 | 状态可能过时 |
| 记忆注入限制 | 控制 context 大小 | 相关信息可能缺失 |

### 10.2 缓存效率 vs 个性化

| 决策 | 收益 | 代价 |
|---|---|---|
| 静态/动态分离 | 最大缓存效率 | 静态区无法个性化 |
| 工具排序 | 缓存前缀稳定 | 工具顺序不直观 |

---

## 十一、关键数字

| 参数 | 值 |
|---|---|
| 默认上下文窗口 | 200K tokens |
| 1M 上下文窗口 | 1M tokens |
| 自动压缩 Buffer | 13K tokens |
| 警告 Buffer | 20K tokens |
| 阻塞 Buffer | 3K tokens |
| CLAUDE.md 上限 | 40K 字符 |
| Git 状态上限 | 2K 字符 |
| 记忆注入上限 | 5 文件 × 4KB |
| 记忆会话上限 | 60KB |

---

## 十二、总结

上下文管理是 Claude Code 的核心能力：

1. **四层结构**：System Prompt + User Context + System Context + Messages
2. **静态/动态分离**：优化 Prompt Cache
3. **附件系统**：动态注入相关上下文
4. **Token 监控**：多级阈值预警
5. **预算系统**：用户可控的 token 目标

上下文管理在信息完整性、Token 效率和响应延迟之间找到了平衡。
