# 子领域 06：System Prompt

> Claude Code 的系统提示词构建、结构与优化

---

## 一、概述

System Prompt 是发送给 API 的核心指令，决定了 Claude 的角色、行为和能力边界。Claude Code 的 System Prompt 采用分层结构，支持缓存优化。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/constants/prompts.ts` | 默认 System Prompt |
| `src/utils/systemPrompt.ts` | 构建逻辑 |
| `src/context.ts` | 上下文收集 |

---

## 二、分层结构

### 2.1 静态区与动态区

```
┌────────────────────────────────────────────────────┐
│                   System Prompt                      │
├────────────────────────────────────────────────────┤
│                                                     │
│  ┌────────────────────────────────────────────┐    │
│  │          静态区 (Static Area)               │    │
│  │          cacheScope: 'global'              │    │
│  │          对所有用户相同                      │    │
│  │                                            │    │
│  │  1. Intro (角色定义)                        │    │
│  │  2. System (系统规则)                       │    │
│  │  3. Doing Tasks (任务执行)                  │    │
│  │  4. Actions (行为准则)                      │    │
│  │  5. Using Tools (工具使用)                  │    │
│  │  6. Tone & Style (语气风格)                 │    │
│  │  7. Output (输出效率)                       │    │
│  │                                            │    │
│  └────────────────────────────────────────────┘    │
│                                                     │
│  SYSTEM_PROMPT_DYNAMIC_BOUNDARY                    │
│                                                     │
│  ┌────────────────────────────────────────────┐    │
│  │          动态区 (Dynamic Area)              │    │
│  │          cacheScope: 'session'             │    │
│  │          用户/会话特定                       │    │
│  │                                            │    │
│  │  8. Session Guidance (会话指导)             │    │
│  │  9. Memory (记忆系统)                       │    │
│  │ 10. Environment (环境信息)                  │    │
│  │ 11. Language (语言偏好)                     │    │
│  │ 12. Output Style (输出样式)                 │    │
│  │ 13. MCP Instructions (MCP 指令)             │    │
│  │ 14. Scratchpad (临时文件说明)               │    │
│  │ 15. Function Result Clearing               │    │
│  │ 16. Token Budget (预算说明)                 │    │
│  │ 17. Brief (KAIROS 简报)                    │    │
│  │                                            │    │
│  └────────────────────────────────────────────┘    │
│                                                     │
└────────────────────────────────────────────────────┘
```

### 2.2 分离原因

**Prompt Cache 优化**：

- 静态区对所有用户相同 → 可以用 `global` scope 缓存
- 动态区因用户/会话而异 → 只能用 `session` scope 缓存
- 缓存命中时，token 处理速度快 ~10x，成本降低 ~90%

---

## 三、静态区详解

### 3.1 Intro (角色定义)

```markdown
You are Claude Code, Anthropic's official CLI for Claude. You are an expert at:
- Software development, debugging, and code review
- Explaining complex technical concepts
- Writing clean, maintainable code
- Following best practices and coding standards

You are running inside a terminal and can interact with the local filesystem,
execute commands, and use various tools to help users with their tasks.
```

### 3.2 System (系统规则)

```markdown
## System Rules

### Tool Permissions
- Some tools require user confirmation before use
- You will be informed when a tool needs permission
- Do not ask for permission; the system handles this automatically

### system-reminder
- You may see system-reminder blocks in the conversation
- These contain important contextual information
- Pay attention to them but don't mention them to the user
```

### 3.3 Doing Tasks (任务执行)

```markdown
## Doing Tasks

### Code Style
- Follow the existing code style in the project
- Use appropriate naming conventions
- Write self-documenting code with clear variable names

### Safety
- Always read files before editing them
- Use exact string replacement, not line-based editing
- Back up important files before major changes

### Efficiency
- Use specialized tools instead of Bash when available
- Batch related operations when possible
- Avoid redundant reads of the same file
```

### 3.4 Actions (行为准则)

```markdown
## Actions

### Always Ask Before:
- Deleting files or directories
- Running potentially destructive commands
- Making changes outside the project directory
- Installing new packages

### Automatic Actions (No Confirmation Needed):
- Reading files
- Searching for code
- Running read-only commands (ls, cat, grep, etc.)
```

### 3.5 Using Tools (工具使用)

```markdown
## Using Tools

### Tool Selection
- Prefer specialized tools over Bash
- Use Read for files, Grep for search, Glob for file patterns
- Bash is for operations not covered by other tools

### Tool Usage
- Read the tool description carefully
- Provide all required parameters
- Handle errors gracefully
```

### 3.6 Tone & Style (语气风格)

```markdown
## Tone & Style

### Communication
- Be concise and direct
- Avoid unnecessary explanations
- Don't use emojis in code or explanations
- Use code blocks for code, not for emphasis

### Response Structure
- Start with the answer, then explain if needed
- Break complex tasks into steps
- Use bullet points for lists
```

### 3.7 Output (输出效率)

```markdown
## Output

### Efficiency
- Get straight to the point
- Don't repeat information
- Use truncation for long outputs
- Keep tool outputs between tool uses ≤25 words
```

---

## 四、动态区详解

### 4.1 Session Guidance

```markdown
## Session Guidance

### Using Agents
- Spawn sub-agents for independent tasks
- Use TaskCreateTool for background work
- Coordinate multiple agents with SendMessageTool

### Skills
- Check available skills with /skills command
- Skills provide specialized capabilities
- Activate skills when relevant to the task
```

### 4.2 Memory

```markdown
## Memory

You have access to a memory system:
- MEMORY.md: Your long-term memory (curated wisdom)
- memory/*.md: Daily notes and detailed logs
- Relevant memories are injected automatically

Use memory to:
- Remember user preferences across sessions
- Store project-specific knowledge
- Track decisions and their rationale
```

### 4.3 Environment

```markdown
## Environment

Current working directory: /home/user/project
Operating System: Linux (x64)
Current date: 2026-03-31
Model: claude-sonnet-4.6
Default model: claude-sonnet-4.6
```

### 4.4 Language

```markdown
## Language

User prefers responses in: English

When the user communicates in another language, respond in that language.
```

### 4.5 MCP Instructions

```markdown
## MCP Servers

### filesystem
- Provides file system operations
- Tools: read_file, write_file, list_directory

### github
- Provides GitHub API access
- Tools: create_issue, create_pr, search_code
```

### 4.6 Token Budget

```markdown
## Token Budget

When the user specifies a token target:
- Continue working until the target is reached
- The system will nudge you at 50%, 75%, 90%
- Stop when you reach 90% of the target
```

---

## 五、构建流程

### 5.1 优先级

```typescript
function buildEffectiveSystemPrompt(options: SystemPromptOptions): string {
  const { mode, customPrompt, appendPrompt, overridePrompt } = options

  // 优先级 0: Override (完全替换)
  if (overridePrompt) {
    return overridePrompt
  }

  // 优先级 1: Coordinator 模式
  if (mode === 'coordinator') {
    return COORDINATOR_PROMPT
  }

  // 优先级 2: Agent 系统提示词
  let prompt = options.agentPrompt || getDefaultSystemPrompt()

  // 优先级 3: 自定义系统提示词
  if (customPrompt) {
    if (mode === 'proactive') {
      // proactive 模式: 追加到默认
      prompt += '\n\n' + customPrompt
    } else {
      // 普通模式: 替换默认
      prompt = customPrompt
    }
  }

  // 优先级 4: 追加内容
  if (appendPrompt) {
    prompt += '\n\n' + appendPrompt
  }

  return prompt
}
```

### 5.2 Section 注册机制

```typescript
interface SystemPromptSection {
  name: string
  priority: number
  cacheScope: 'global' | 'session' | 'none'
  render: () => string | Promise<string>
}

const SYSTEM_PROMPT_SECTIONS: SystemPromptSection[] = [
  {
    name: 'intro',
    priority: 100,
    cacheScope: 'global',
    render: () => INTRO_SECTION,
  },
  {
    name: 'environment',
    priority: 500,
    cacheScope: 'session',
    render: async () => await renderEnvironmentSection(),
  },
  // ...
]

async function buildSystemPrompt(): Promise<string> {
  const sections = SYSTEM_PROMPT_SECTIONS
    .sort((a, b) => a.priority - b.priority)

  const parts: string[] = []
  let lastCacheScope = 'global'

  for (const section of sections) {
    const content = await section.render()

    // 插入边界标记
    if (section.cacheScope !== lastCacheScope) {
      parts.push(SYSTEM_PROMPT_DYNAMIC_BOUNDARY)
      lastCacheScope = section.cacheScope
    }

    parts.push(content)
  }

  return parts.join('\n\n')
}
```

---

## 六、缓存优化

### 6.1 工具排序

```typescript
// 工具按名称排序，保证缓存前缀稳定
function sortToolsForPromptCache(tools: Tool[]): Tool[] {
  return [...tools].sort((a, b) => a.name.localeCompare(b.name))
}
```

### 6.2 Delta 模式

```typescript
// MCP 指令使用 delta 模式，避免每次重连破坏缓存
function buildMcpInstructionsDelta(
  previous: MCPInstruction[],
  current: MCPInstruction[]
): string {
  const added = current.filter(c => !previous.some(p => p.name === c.name))
  const removed = previous.filter(p => !current.some(c => c.name === p.name))

  let result = ''

  if (added.length > 0) {
    result += `Added MCP servers:\n${added.map(a => `- ${a.name}`).join('\n')}\n`
  }

  if (removed.length > 0) {
    result += `Removed MCP servers:\n${removed.map(r => `- ${r.name}`).join('\n')}\n`
  }

  return result
}
```

### 6.3 空操作占位

```typescript
// Token 预算说明即使未启用也放入缓存
const TOKEN_BUDGET_SECTION = `
## Token Budget

When the user specifies a token target:
- Continue working until the target is reached
- The system will nudge you at 50%, 75%, 90%
- Stop when you reach 90% of the target
`

// 即使没有 token 目标，也包含这个 section（空操作措辞）
// 这样启用时不破坏缓存
```

---

## 七、自定义系统提示词

### 7.1 CLI 参数

```bash
claude --system-prompt "You are a Python expert."
claude --append-system-prompt "Always use type hints."
```

### 7.2 CLAUDE.md

```markdown
# CLAUDE.md

## Project Context
This is a React + TypeScript project using Vite.

## Code Style
- Use functional components with hooks
- Prefer named exports
- Use Tailwind CSS for styling

## Testing
- Use Vitest for unit tests
- Use Playwright for E2E tests
```

### 7.3 API 模式

```typescript
const query = new QueryEngine({
  systemPrompt: {
    type: 'custom',
    content: 'You are a Python expert.',
  },
})
```

---

## 八、Prompt 大小

### 8.1 典型大小

| Section | 大小 |
|---|---|
| 静态区 | ~8K tokens |
| 动态区 | ~5K tokens |
| 工具 Schema | ~2K tokens |
| **总计** | **~15K tokens** |

### 8.2 优化策略

1. **精简描述**：工具描述保持简洁
2. **按需加载**：只加载必要的 MCP 指令
3. **缓存复用**：最大化静态区
4. **工具过滤**：移除未使用的工具

---

## 九、设计权衡

### 9.1 静态区 vs 动态区

| 放入静态区 | 放入动态区 |
|---|---|
| 角色定义（固定） | 环境信息（变化） |
| 系统规则（固定） | Git 状态（变化） |
| 工具使用指南 | CLAUDE.md（用户特定） |

### 9.2 完整性 vs 效率

| 策略 | 优点 | 缺点 |
|---|---|---|
| 完整工具描述 | 模型理解工具 | 增加 token |
| 精简工具描述 | 减少 token | 可能误用工具 |

---

## 十、总结

System Prompt 是 Claude Code 的"大脑设定"：

1. **分层结构**：静态区 + 动态区
2. **缓存优化**：global scope 缓存静态区
3. **优先级系统**：Override → Coordinator → Agent → Custom → Default
4. **Delta 模式**：减少缓存失效
5. **自定义支持**：CLI 参数、CLAUDE.md、API

System Prompt 的设计在指导模型行为和 token 效率之间找到了平衡。
