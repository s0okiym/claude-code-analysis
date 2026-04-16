# 子领域 02：工具系统

> Claude Code 的工具注册、执行与管理机制

---

## 一、概述

Claude Code 内置 30+ 工具，覆盖文件操作、代码搜索、Shell 执行、网络请求等场景。工具系统是 Agent 与外部世界交互的桥梁。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/tools.ts` | 工具注册表 |
| `src/Tool.ts` | 工具类型定义 |
| `src/tools/*.ts` | 各工具实现 |
| `src/toolOrchestration.ts` | 工具执行协调 |

---

## 二、工具注册

### 2.1 工具列表

```typescript
function getAllBaseTools(): Tool[] {
  return [
    // 文件操作
    new FileReadTool(),
    new FileEditTool(),
    new FileWriteTool(),

    // 搜索
    new GlobTool(),
    new GrepTool(),

    // 执行
    new BashTool(),

    // 网络
    new WebFetchTool(),
    new WebSearchTool(),

    // Agent
    new AgentTool(),
    new TeamCreateTool(),
    new SendMessageTool(),

    // 任务
    new TodoWriteTool(),
    new TaskCreateTool(),
    new TaskGetTool(),
    new TaskUpdateTool(),
    new TaskListTool(),

    // 交互
    new AskUserQuestionTool(),

    // MCP
    new ListMcpResourcesTool(),
    new ReadMcpResourceTool(),

    // 其他
    new NotebookEditTool(),
    new SkillTool(),
    new BriefTool(),
    new ToolSearchTool(),
    new EnterPlanModeTool(),
    new ExitPlanModeV2Tool(),
    new EnterWorktreeTool(),
    new ExitWorktreeTool(),
    new ConfigTool(),
    new LSPTool(),
    // ...
  ]
}
```

### 2.2 工具过滤链

```
getAllBaseTools()
  │
  ├─ filterToolsByDenyRules()
  │   └─ 按 permissionRules.deny 过滤
  │
  ├─ getTools(permissionCtx)
  │   └─ 按 mode (simple/full) 过滤
  │
  ├─ assembleToolPool(mcpTools)
  │   └─ 合并 MCP 工具，去重
  │
  └─ 按 name 排序 (prompt-cache 稳定性)
```

---

## 三、工具类型定义

### 3.1 Tool 接口

```typescript
interface Tool<Input, Output> {
  // 基本信息
  name: string
  description: string

  // 参数 Schema (Zod)
  parameters: z.ZodSchema<Input>

  // 执行函数
  run(input: Input, context: ToolContext): Promise<ToolResult<Output>>

  // 可选配置
  maxResultSizeChars?: number
  isConcurrencySafe?: boolean
  requiresPermissions?: boolean
  timeout?: number
}
```

### 3.2 工具上下文

```typescript
interface ToolContext {
  // 会话信息
  sessionId: string
  abortSignal: AbortSignal

  // 权限上下文
  permissionContext: PermissionContext

  // 文件状态缓存
  fileStateCache: FileStateCache

  // MCP 客户端
  mcpClients: MCPClient[]

  // 日志
  logger: Logger
}
```

### 3.3 工具结果

```typescript
interface ToolResult<T> {
  // 结果内容
  content: T | string

  // 是否错误
  isError?: boolean

  // 元数据
  metadata?: {
    tokensUsed?: number
    duration?: number
    filesModified?: string[]
  }
}
```

---

## 四、核心工具详解

### 4.1 FileReadTool

```typescript
class FileReadTool implements Tool<FileReadInput, string> {
  name = 'Read'
  description = 'Reads a file from the local filesystem...'

  parameters = z.object({
    file_path: z.string().describe('The path to the file to read'),
    offset: z.number().optional().describe('Line number to start reading from'),
    limit: z.number().optional().describe('Maximum number of lines to read'),
  })

  isConcurrencySafe = true // 可并发执行

  async run(input: FileReadInput, context: ToolContext) {
    const { file_path, offset, limit } = input

    // 权限检查
    await checkFileReadPermission(file_path, context)

    // 读取文件
    const content = await readFile(file_path, { offset, limit })

    // 更新缓存
    context.fileStateCache.set(file_path, content)

    return { content }
  }
}
```

### 4.2 FileEditTool

```typescript
class FileEditTool implements Tool<FileEditInput, string> {
  name = 'Edit'
  description = 'Performs exact string replacement in a file...'

  parameters = z.object({
    file_path: z.string(),
    old_text: z.string().describe('Exact text to replace'),
    new_text: z.string().describe('Replacement text'),
  })

  isConcurrencySafe = false // 需要顺序执行

  async run(input: FileEditInput, context: ToolContext) {
    const { file_path, old_text, new_text } = input

    // 检查文件存在
    if (!await fileExists(file_path)) {
      return { content: `Error: File not found: ${file_path}`, isError: true }
    }

    // 读取当前内容
    const content = await readFile(file_path)

    // 检查 old_text 是否存在
    if (!content.includes(old_text)) {
      return { content: `Error: old_text not found in file`, isError: true }
    }

    // 执行替换
    const newContent = content.replace(old_text, new_text)
    await writeFile(file_path, newContent)

    return { content: `Successfully edited ${file_path}` }
  }
}
```

### 4.3 BashTool

```typescript
class BashTool implements Tool<BashInput, BashOutput> {
  name = 'Bash'
  description = 'Executes a shell command...'

  parameters = z.object({
    command: z.string().describe('The command to execute'),
    timeout: z.number().optional().default(120000),
    workdir: z.string().optional(),
  })

  isConcurrencySafe = false // 默认不安全

  async run(input: BashInput, context: ToolContext) {
    const { command, timeout, workdir } = input

    // 权限检查（通过 bashClassifier 分类）
    const classification = classifyBashCommand(command)
    if (classification === 'deny') {
      return { content: `Error: Command not allowed`, isError: true }
    }

    // 执行命令
    const result = await execCommand(command, {
      cwd: workdir,
      timeout,
      signal: context.abortSignal,
    })

    return {
      content: result.stdout || result.stderr,
      metadata: {
        exitCode: result.exitCode,
        duration: result.duration,
      },
    }
  }
}
```

### 4.4 GrepTool

```typescript
class GrepTool implements Tool<GrepInput, string> {
  name = 'Grep'
  description = 'Searches for patterns in files using ripgrep...'

  parameters = z.object({
    pattern: z.string().describe('The pattern to search for'),
    path: z.string().optional().describe('Directory to search in'),
    glob: z.string().optional().describe('Glob pattern for file filtering'),
    case_insensitive: z.boolean().optional(),
    context_lines: z.number().optional(),
  })

  isConcurrencySafe = true // 只读操作

  async run(input: GrepInput, context: ToolContext) {
    const args = buildRipgrepArgs(input)
    const result = await execCommand(`rg ${args.join(' ')}`)
    return { content: result.stdout }
  }
}
```

---

## 五、工具执行协调

### 5.1 toolOrchestration.ts

```typescript
async function runTools(
  toolUseBlocks: ToolUseBlock[],
  context: ToolContext
): Promise<ToolResultMessage[]> {
  // 1. 分区
  const { concurrent, sequential } = partitionToolCalls(toolUseBlocks)

  const results: ToolResultMessage[] = []

  // 2. 并发执行安全工具
  if (concurrent.length > 0) {
    const concurrentResults = await runToolsConcurrently(concurrent, context)
    results.push(...concurrentResults)
  }

  // 3. 顺序执行非安全工具
  for (const tool of sequential) {
    const result = await runSingleTool(tool, context)
    results.push(result)
  }

  return results
}
```

### 5.2 并发控制

```typescript
const MAX_CONCURRENCY = 10

async function runToolsConcurrently(
  tools: ToolCall[],
  context: ToolContext
): Promise<ToolResultMessage[]> {
  // 使用 Promise.all 实现并发
  // 限制最大并发数
  const batches = chunk(tools, MAX_CONCURRENCY)
  const results: ToolResultMessage[] = []

  for (const batch of batches) {
    const batchResults = await Promise.all(
      batch.map(tool => runSingleTool(tool, context))
    )
    results.push(...batchResults)
  }

  return results
}
```

### 5.3 单工具执行

```typescript
async function runSingleTool(
  toolCall: ToolCall,
  context: ToolContext
): Promise<ToolResultMessage> {
  const { id, name, input } = toolCall

  try {
    // 1. 获取工具实例
    const tool = getToolByName(name)
    if (!tool) {
      return {
        type: 'tool_result',
        tool_use_id: id,
        content: `Unknown tool: ${name}`,
        is_error: true,
      }
    }

    // 2. 权限检查
    const permission = await canUseTool(tool, input, context)
    if (permission === 'deny') {
      return {
        type: 'tool_result',
        tool_use_id: id,
        content: 'Permission denied',
        is_error: true,
      }
    }

    // 3. 执行前 Hook
    await runPreToolUseHooks(tool, input)

    // 4. 执行工具
    const result = await tool.run(input, context)

    // 5. 执行后 Hook
    await runPostToolUseHooks(tool, input, result)

    // 6. 返回结果
    return {
      type: 'tool_result',
      tool_use_id: id,
      content: result.content,
      is_error: result.isError,
    }

  } catch (error) {
    return {
      type: 'tool_result',
      tool_use_id: id,
      content: `Error: ${error.message}`,
      is_error: true,
    }
  }
}
```

---

## 六、工具结果处理

### 6.1 结果持久化

```typescript
// 大结果写入磁盘
async function handleToolResult(result: ToolResult, toolName: string) {
  const threshold = getPersistenceThreshold(toolName)

  if (result.content.length > threshold) {
    // 持久化到文件
    const filePath = await persistResult(result.content)
    return {
      content: `[Result stored in ${filePath}]`,
      metadata: { persistedPath: filePath },
    }
  }

  return result
}
```

### 6.2 结果截断

```typescript
const MAX_INLINE_RESULT = 50000 // 字符

function truncateResult(content: string): string {
  if (content.length <= MAX_INLINE_RESULT) {
    return content
  }

  return content.slice(0, MAX_INLINE_RESULT) +
    `\n... [truncated ${content.length - MAX_INLINE_RESULT} chars]`
}
```

---

## 七、MCP 工具扩展

### 7.1 MCP 工具注册

```typescript
async function loadMcpTools(mcpClients: MCPClient[]): Promise<Tool[]> {
  const tools: Tool[] = []

  for (const client of mcpClients) {
    const serverTools = await client.listTools()
    for (const tool of serverTools) {
      tools.push({
        name: `mcp_${client.name}_${tool.name}`,
        description: tool.description,
        parameters: convertMcpSchemaToZod(tool.inputSchema),
        run: async (input, context) => {
          return await client.callTool(tool.name, input)
        },
      })
    }
  }

  return tools
}
```

### 7.2 MCP 工具命名

```
格式: mcp_{serverName}_{toolName}

示例:
- mcp_filesystem_read_file
- mcp_github_create_issue
- mcp_postgres_query
```

---

## 八、工具安全分类

### 8.1 并发安全性

| 安全 | 不安全 |
|---|---|
| Read | Edit |
| Grep | Write |
| Glob | Bash |
| WebFetch | NotebookEdit |
| WebSearch | FileWrite |

### 8.2 Bash 命令分类

```typescript
function classifyBashCommand(command: string): 'safe' | 'confirm' | 'deny' {
  // 安全命令列表
  const safePatterns = [
    /^ls(\s|$)/,
    /^cat(\s|$)/,
    /^grep(\s|$)/,
    /^git status/,
    /^git log/,
    /^git diff/,
    /^npm list/,
  ]

  // 危险命令列表
  const denyPatterns = [
    /^rm\s+-rf\s+\//,
    />\s*\/dev\/sd/,
    /^mkfs/,
    /^dd\s+if=/,
  ]

  for (const pattern of safePatterns) {
    if (pattern.test(command)) return 'safe'
  }

  for (const pattern of denyPatterns) {
    if (pattern.test(command)) return 'deny'
  }

  return 'confirm' // 需要确认
}
```

---

## 九、工具 Schema 生成

### 9.1 Zod → JSON Schema

```typescript
function zodToJsonSchema(schema: z.ZodSchema): JSONSchema {
  // Zod schema 转换为 JSON Schema
  // 用于 API tools 参数
  return zodToJson(schema)
}

// 示例
const readSchema = zodToJsonSchema(FileReadTool.parameters)
// 输出:
// {
//   type: 'object',
//   properties: {
//     file_path: { type: 'string', description: '...' },
//     offset: { type: 'number', description: '...' },
//     limit: { type: 'number', description: '...' },
//   },
//   required: ['file_path'],
// }
```

### 9.2 工具排序

```typescript
// 按名称排序，确保 prompt cache 稳定性
function sortToolsByName(tools: Tool[]): Tool[] {
  return tools.sort((a, b) => a.name.localeCompare(b.name))
}
```

---

## 十、设计评价

### 10.1 优点

1. **统一的接口**：所有工具实现相同的 `Tool` 接口
2. **类型安全**：Zod schema 提供编译时和运行时验证
3. **并发优化**：自动识别安全工具并发执行
4. **可扩展**：MCP 协议支持第三方工具

### 10.2 局限性

1. **工具数量限制**：过多工具增加 prompt token 消耗
2. **并发限制**：固定最大并发 10
3. **结果大小**：大结果需要持久化，增加 I/O

### 10.3 潜在改进

1. **动态加载**：根据任务需要动态加载工具
2. **工具压缩**：精简工具描述减少 token
3. **智能并发**：根据系统负载动态调整并发数

---

## 十一、总结

工具系统是 Claude Code 与外部世界交互的核心：

1. **30+ 内置工具**：覆盖常见开发场景
2. **统一接口**：`Tool<Input, Output>` 泛型接口
3. **并发执行**：安全工具自动并发
4. **权限控制**：每个工具调用都经过权限检查
5. **MCP 扩展**：标准协议接入第三方工具

工具系统的设计在功能丰富性和 token 效率之间找到了平衡。
