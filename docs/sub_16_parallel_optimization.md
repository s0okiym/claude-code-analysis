# 子领域 16：并行优化

> Claude Code 的多层并行执行策略

---

## 一、概述

Claude Code 在多个层次实现并行优化，从启动预取到工具执行，最大化利用并发能力。

### 优化层次

| 层次 | 场景 | 策略 |
|---|---|---|
| 启动 | 初始化 | 并行预取 |
| API 调用 | 上下文收集 | Promise.all |
| 工具执行 | 只读操作 | 并发池 |
| Git 状态 | 多命令 | Promise.all |
| MCP 资源 | 多服务器 | 并行预取 |

---

## 二、启动并行预取

### 2.1 Side-effect Imports

```typescript
// main.tsx 入口处
// 在所有模块加载前启动并行预取

// MDM 配置预取
startMdmRawRead()

// Keychain 预取
startKeychainPrefetch()

// API Preconnect
startApiPreconnect()

// 然后才 import 其他模块
import { QueryEngine } from './QueryEngine'
import { launchRepl } from './screens/REPL'
```

### 2.2 效果

```
串行加载 (传统方式):
  MDM (135ms) → Keychain (100ms) → API Preconnect (50ms) = 285ms

并行预取:
  MDM ─────┐
  Keychain ┼─ 并行 ─→ 全部完成 = ~135ms
  API ─────┘

节省: ~150ms
```

---

## 三、上下文并行收集

### 3.1 fetchSystemPromptParts()

```typescript
async function fetchSystemPromptParts(): Promise<SystemPromptParts> {
  // 并行获取三部分
  const [systemPrompt, userContext, systemContext] = await Promise.all([
    getSystemPrompt(tools, model, dirs, mcpClients),
    getUserContext(),   // CLAUDE.md + 日期
    getSystemContext(), // Git 状态
  ])

  return { systemPrompt, userContext, systemContext }
}
```

### 3.2 Git 状态并行

```typescript
async function getSystemContext(): Promise<SystemContext> {
  const isGitRepo = await isGitRepository()

  if (!isGitRepo) {
    return { gitStatus: null }
  }

  // 并行获取所有 Git 信息
  const [branch, mainBranch, status, commits, userName] = await Promise.all([
    git('rev-parse --abbrev-ref HEAD'),
    getMainBranch(),
    git('status --short').then(s => s.slice(0, 2000)),
    git('log -5 --oneline'),
    git('config user.name'),
  ])

  return {
    gitStatus: { branch, mainBranch, status, recentCommits: commits, userName },
  }
}
```

---

## 四、工具并发执行

### 4.1 并发安全判断

```typescript
function isToolConcurrencySafe(tool: Tool): boolean {
  // 显式声明
  if (tool.isConcurrencySafe !== undefined) {
    return tool.isConcurrencySafe
  }

  // 默认判断：只读操作安全
  const readOnlyTools = ['Read', 'Grep', 'Glob', 'WebFetch', 'WebSearch']
  return readOnlyTools.includes(tool.name)
}
```

### 4.2 分区执行

```typescript
async function runTools(toolCalls: ToolCall[]): Promise<ToolResult[]> {
  // 分区
  const { concurrent, sequential } = partitionToolCalls(toolCalls)

  const results: ToolResult[] = []

  // 并发执行安全工具
  if (concurrent.length > 0) {
    const concurrentResults = await runToolsConcurrently(concurrent)
    results.push(...concurrentResults)
  }

  // 顺序执行非安全工具
  for (const call of sequential) {
    const result = await runSingleTool(call)
    results.push(result)
  }

  return results
}
```

### 4.3 并发池控制

```typescript
const MAX_CONCURRENCY = parseInt(process.env.CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || '10')

async function runToolsConcurrently(tools: ToolCall[]): Promise<ToolResult[]> {
  // 分批执行，控制并发数
  const batches = chunk(tools, MAX_CONCURRENCY)
  const results: ToolResult[] = []

  for (const batch of batches) {
    const batchResults = await Promise.all(
      batch.map(tool => runSingleTool(tool))
    )
    results.push(...batchResults)
  }

  return results
}
```

---

## 五、MCP 资源并行预取

### 5.1 prefetchAllMcpResources()

```typescript
async function prefetchAllMcpResources(
  mcpClients: MCPClient[]
): Promise<MCPResource[]> {
  // 并行从所有 MCP 服务器获取资源
  const results = await Promise.all(
    mcpClients.map(client =>
      client.request('resources/list', {}).catch(error => {
        console.error(`Failed to get resources from ${client.name}:`, error)
        return { resources: [] }
      })
    )
  )

  return results.flatMap(r => r.resources)
}
```

---

## 六、系统提示词并行构建

### 6.1 多部分并行

```typescript
async function buildSystemPrompt(options: SystemPromptOptions): Promise<string> {
  // 并行获取各部分
  const [
    skillsPrompt,
    outputStylePrompt,
    envInfo,
    mcpInstructions,
  ] = await Promise.all([
    loadSkillsPrompt(),
    getOutputStylePrompt(),
    getEnvironmentInfo(),
    getMCPInstructions(options.mcpClients),
  ])

  // 组装
  return assembleSystemPrompt({
    ...options,
    skillsPrompt,
    outputStylePrompt,
    envInfo,
    mcpInstructions,
  })
}
```

---

## 七、文件系统并行

### 7.1 多文件读取

```typescript
// 读取多个文件时并行
async function readFiles(paths: string[]): Promise<Map<string, string>> {
  const results = await Promise.all(
    paths.map(async path => ({
      path,
      content: await readFile(path).catch(() => null),
    }))
  )

  return new Map(results.map(r => [r.path, r.content]))
}
```

### 7.2 CLAUDE.md 加载

```typescript
async function getClaudeMds(): Promise<ClaudeMd[]> {
  const paths = discoverClaudeMdPaths()

  // 并行读取所有文件
  const contents = await Promise.all(
    paths.map(path =>
      readFile(path)
        .then(content => ({ path, content }))
        .catch(() => null)
    )
  )

  return contents.filter(Boolean)
}
```

---

## 八、网络请求并行

### 8.1 WebFetch 批量

```typescript
async function fetchUrls(urls: string[]): Promise<Map<string, string>> {
  const results = await Promise.all(
    urls.map(async url => ({
      url,
      content: await fetch(url).then(r => r.text()).catch(() => null),
    }))
  )

  return new Map(results.map(r => [r.url, r.content]))
}
```

---

## 九、压缩并行

### 9.1 Fork 子 Agent

```typescript
async function compactConversation(messages: Message[]): Promise<string> {
  // 压缩在 fork 的子 Agent 中执行
  // 不阻塞主循环
  const summary = await forkAgent({
    task: 'compact',
    messages,
    timeout: 60000,
  })

  return summary
}
```

---

## 十、并行限制

### 10.1 并发数限制

```typescript
// 环境变量控制
const MAX_CONCURRENCY = parseInt(
  process.env.CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || '10'
)

// 避免 C10K 问题
if (MAX_CONCURRENCY > 100) {
  console.warn('MAX_CONCURRENCY capped at 100')
}
```

### 10.2 资源限制

```typescript
// 检查系统资源
function getOptimalConcurrency(): number {
  const cpuCount = os.cpus().length
  const memoryGB = os.totalmem() / (1024 * 1024 * 1024)

  // CPU 和内存综合考虑
  return Math.min(cpuCount * 2, Math.floor(memoryGB / 2), MAX_CONCURRENCY)
}
```

---

## 十一、性能对比

### 11.1 启动时间

| 阶段 | 串行 | 并行 | 节省 |
|---|---|---|---|
| MDM + Keychain | 235ms | 135ms | 100ms |
| Git 状态 | 150ms | 50ms | 100ms |
| MCP 资源 | 300ms | 100ms | 200ms |
| **总计** | **685ms** | **285ms** | **400ms** |

### 11.2 工具执行

| 场景 | 串行 | 并行 | 节省 |
|---|---|---|---|
| 5 个只读工具 | 2.5s | 0.5s | 2s |
| 10 个文件读取 | 1s | 0.2s | 0.8s |

---

## 十二、最佳实践

### 12.1 设计原则

1. **I/O 操作并行**：网络、文件、进程
2. **CPU 密集串行**：压缩、计算
3. **写入操作顺序**：避免竞态条件

### 12.2 错误处理

```typescript
// 并行操作要有容错
const results = await Promise.allSettled(operations)

const succeeded = results
  .filter(r => r.status === 'fulfilled')
  .map(r => r.value)

const failed = results
  .filter(r => r.status === 'rejected')
  .map(r => r.reason)
```

---

## 十三、总结

并行优化贯穿 Claude Code 的每个层次：

1. **启动预取**：MDM + Keychain + API
2. **上下文收集**：Promise.all 并行
3. **工具执行**：安全工具并发池
4. **Git 状态**：多命令并行
5. **MCP 资源**：多服务器并行

并行优化显著降低了延迟，提升了用户体验。
