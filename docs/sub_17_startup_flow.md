# 子领域 17：启动流程

> Claude Code 的初始化与状态恢复机制

---

## 一、概述

Claude Code 的启动流程包含多个阶段，从进程启动到 REPL 就绪，涉及认证检查、配置加载、上下文构建等。

### 启动阶段

```
进程启动 → 认证检查 → 配置加载 → 上下文构建 → UI 初始化 → REPL 就绪
```

---

## 二、启动流程详解

### 2.1 进程入口

```typescript
// src/main.tsx (Bun 入口)

// 1. 并行预取（side-effect imports）
startMdmRawRead()      // MDM 配置预取
startKeychainPrefetch() // Keychain 预取
startApiPreconnect()   // API 预连接

// 2. 解析命令行参数
const args = parseArgs(process.argv.slice(2))

// 3. 设置全局配置
setupGlobals(args)

// 4. 根据模式启动
if (args.mode === 'doctor') {
  await launchDoctor()
} else if (args.mode === 'bridge') {
  await launchBridge()
} else {
  await launchRepl()
}
```

### 2.2 认证检查

```typescript
async function checkAuthentication(): Promise<AuthStatus> {
  // 1. 检查 OAuth token (从预取的 Keychain)
  const token = await keychainPrefetch
  if (token && !isTokenExpired(token)) {
    return { authenticated: true, method: 'oauth' }
  }

  // 2. 检查 API Key
  const apiKey = process.env.ANTHROPIC_API_KEY
  if (apiKey) {
    return { authenticated: true, method: 'api_key' }
  }

  // 3. 未认证
  return { authenticated: false }
}
```

### 2.3 配置加载

```typescript
async function loadConfig(): Promise<Config> {
  const config: Config = {}

  // 并行加载所有配置源
  const [globalConfig, projectConfig, mcpConfig, envConfig] = await Promise.all([
    readJson(path.join(os.homedir(), '.claude/config.json')),
    readJson('.claude/config.json'),
    readJson(path.join(os.homedir(), '.claude/mcp.json')),
    loadEnvConfig(),
  ])

  // 合并（优先级：env > project > global）
  return mergeConfigs(globalConfig, projectConfig, envConfig)
}
```

---

## 三、状态恢复

### 3.1 恢复点文件

```typescript
// 恢复点存储位置
const RESUME_FILE = path.join(
  os.homedir(),
  '.claude/resume/resume-v1.json'
)

interface ResumeState {
  sessionId: string
  timestamp: number
  messages: Message[]
  toolResults: Map<string, ToolResult>
  pendingToolCalls: ToolCall[]
}
```

### 3.2 恢复检查

```typescript
async function checkForResume(): Promise<ResumeState | null> {
  try {
    const state = await readJson(RESUME_FILE)

    // 检查是否过期（超过 24 小时）
    const age = Date.now() - state.timestamp
    if (age > 24 * 60 * 60 * 1000) {
      await fs.unlink(RESUME_FILE)
      return null
    }

    return state
  } catch {
    return null
  }
}
```

### 3.3 恢复确认

```typescript
async function promptForResume(state: ResumeState): Promise<boolean> {
  const ageMinutes = Math.floor((Date.now() - state.timestamp) / 60000)

  console.log(`Found previous session (${ageMinutes} minutes ago)`)
  console.log(`Last message: ${state.messages.slice(-1)[0]?.content?.slice(0, 50)}...`)

  const answer = await prompt('Resume? [Y/n]: ')
  return answer.toLowerCase() !== 'n'
}
```

### 3.4 恢复执行

```typescript
async function resumeSession(state: ResumeState): Promise<void> {
  // 1. 恢复消息
  messages = state.messages

  // 2. 恢复工具结果
  toolResults = new Map(Object.entries(state.toolResults))

  // 3. 处理未完成的工具调用
  if (state.pendingToolCalls.length > 0) {
    console.log(`Resuming ${state.pendingToolCalls.length} pending tool calls...`)
    await executePendingTools(state.pendingToolCalls)
  }

  // 4. 清理恢复点
  await fs.unlink(RESUME_FILE)
}
```

---

## 四、上下文构建

### 4.1 目录发现

```typescript
async function discoverDirectories(): Promise<Directories> {
  const cwd = process.cwd()

  // 查找项目根目录
  const projectRoot = await findProjectRoot(cwd)

  // 查找 CLAUDE.md 位置
  const claudeMds = await discoverClaudeMdPaths(cwd)

  // 确定记忆目录
  const memoryDir = path.join(
    os.homedir(),
    '.claude/projects',
    getProjectSlug(projectRoot),
    'memory'
  )

  return { cwd, projectRoot, claudeMds, memoryDir }
}
```

### 4.2 Git 状态获取

```typescript
async function getGitStatus(): Promise<GitStatus | null> {
  const isRepo = await isGitRepository()
  if (!isRepo) return null

  // 并行获取所有信息
  const [branch, mainBranch, status, commits, userName] = await Promise.all([
    git('rev-parse --abbrev-ref HEAD'),
    getMainBranch(),
    git('status --short').then(s => s.slice(0, 2000)),
    git('log -5 --oneline'),
    git('config user.name'),
  ])

  return { branch, mainBranch, status, recentCommits: commits, userName }
}
```

### 4.3 CLAUDE.md 加载

```typescript
async function loadClaudeMds(paths: string[]): Promise<string> {
  const contents = await Promise.all(
    paths.map(p => readFile(p).catch(() => ''))
  )

  return contents
    .filter(c => c.trim())
    .map((c, i) => `<!-- ${paths[i]} -->\n${c}`)
    .join('\n\n')
}
```

---

## 五、工具初始化

### 5.1 工具注册

```typescript
async function initializeTools(): Promise<Tool[]> {
  const tools: Tool[] = []

  // 核心工具
  tools.push(...getCoreTools())

  // MCP 工具
  const mcpTools = await loadMcpTools(mcpClients)
  tools.push(...mcpTools)

  // 技能工具
  const skillTools = await loadSkillTools()
  tools.push(...skillTools)

  // 过滤禁用的工具
  return tools.filter(t => !isToolDisabled(t.name))
}
```

### 5.2 MCP 初始化

```typescript
async function initializeMCP(config: MCPConfig): Promise<MCPClient[]> {
  const clients: MCPClient[] = []

  for (const [name, serverConfig] of Object.entries(config.servers || {})) {
    try {
      const client = await connectMcpServer(name, serverConfig)
      clients.push(client)
    } catch (error) {
      console.error(`Failed to connect to MCP server ${name}:`, error)
    }
  }

  return clients
}
```

---

## 六、UI 初始化

### 6.1 Ink 渲染

```typescript
async function initializeUI(): Promise<void> {
  const { waitUntilExit } = render(<REPL />)

  // 等待退出
  await waitUntilExit()

  // 清理
  await cleanup()
}
```

### 6.2 REPL 组件

```typescript
const REPL: FC = () => {
  const [messages, setMessages] = useState<Message[]>([])
  const [status, setStatus] = useState('idle')
  const [input, setInput] = useState('')

  // 处理输入
  const handleSubmit = async (userInput: string) => {
    setStatus('thinking')
    const response = await sendMessage(userInput)
    setMessages([...messages, { role: 'user', content: userInput }, response])
    setStatus('idle')
  }

  return (
    <Box flexDirection="column" height="100%">
      <Header status={status} />
      <Messages messages={messages} />
      <PromptInput onSubmit={handleSubmit} />
    </Box>
  )
}
```

---

## 七、启动优化

### 7.1 关键路径优化

```
关键路径（串行）:
认证 → 配置 → 上下文 → 工具 → UI

非关键路径（并行）:
├── MDM 预取
├── Keychain 预取
├── API 预连接
├── Git 状态
├── MCP 资源
└── CLAUDE.md 加载
```

### 7.2 延迟加载

```typescript
// 非关键组件延迟加载
const skillTools = feature('SKILLS_SYSTEM')
  ? await import('./skills').then(m => m.loadSkillTools())
  : []
```

---

## 八、启动模式

### 8.1 标准模式

```bash
claude
# 正常启动 REPL
```

### 8.2 单次模式

```bash
claude -p "解释这段代码"
# 执行一次查询后退出
```

### 8.3 Doctor 模式

```bash
claude doctor
# 运行诊断检查
```

### 8.4 Bridge 模式

```bash
claude --bridge
# 启动远程控制服务
```

### 8.5 Daemon 模式

```bash
claude daemon
# 后台守护进程
```

---

## 九、退出流程

### 9.1 正常退出

```typescript
async function handleExit(): Promise<void> {
  // 1. 保存恢复点
  if (messages.length > 0) {
    await saveResumePoint({
      sessionId,
      timestamp: Date.now(),
      messages,
      toolResults: Object.fromEntries(toolResults),
      pendingToolCalls: [],
    })
  }

  // 2. 清理资源
  await cleanupMCPConnections()
  await flushTelemetry()

  // 3. 退出
  process.exit(0)
}
```

### 9.2 异常退出

```typescript
process.on('uncaughtException', async (error) => {
  console.error('Uncaught exception:', error)

  // 尝试保存状态
  try {
    await saveResumePoint({ ... })
  } catch {}

  process.exit(1)
})

process.on('SIGINT', async () => {
  await handleExit()
})
```

---

## 十、启动时间分析

### 10.1 各阶段耗时

| 阶段 | 耗时（并行优化后） |
|---|---|
| 预取启动 | 0ms（并行） |
| 认证检查 | ~50ms |
| 配置加载 | ~30ms |
| 上下文构建 | ~100ms |
| 工具初始化 | ~50ms |
| UI 初始化 | ~20ms |
| **总计** | **~250ms** |

### 10.2 优化空间

- 配置缓存：减少重复解析
- 工具懒加载：按需加载
- UI 预渲染：减少首次渲染时间

---

## 十一、总结

启动流程是用户体验的第一道门槛：

1. **并行预取**：MDM/Keychain/API 预连接
2. **认证检查**：OAuth/API Key 双路径
3. **状态恢复**：支持断点续传
4. **上下文构建**：Git/CLAUDE.md/记忆
5. **工具初始化**：核心/MCP/技能工具
6. **UI 初始化**：Ink 渲染 REPL

优化的启动流程让 Claude Code 快速就绪。
