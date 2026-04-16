# 子领域 18：CLI 命令

> Claude Code 的命令行接口与斜杠命令系统

---

## 一、概述

Claude Code 提供两种命令系统：CLI 参数（启动时）和斜杠命令（运行时）。CLI 参数控制启动行为，斜杠命令控制会话行为。

---

## 二、CLI 参数

### 2.1 核心参数

| 参数 | 说明 |
|---|---|
| `-p, --print` | 非交互模式，打印结果后退出 |
| `--model` | 指定模型 |
| `--mode` | 启动模式（repl/bridge/daemon） |
| `--resume` | 恢复上次会话 |
| `--no-context` | 不加载上下文 |
| `--dangerously-skip-permissions` | 跳过权限检查 |
| `--verbose` | 详细日志 |
| `-v, --version` | 显示版本 |
| `-h, --help` | 显示帮助 |

### 2.2 参数解析

```typescript
const args = parseArgs(process.argv.slice(2), {
  boolean: ['print', 'resume', 'verbose', 'dangerously-skip-permissions'],
  string: ['model', 'mode'],
  alias: {
    p: 'print',
    v: 'version',
    h: 'help',
  },
  default: {
    mode: 'repl',
    verbose: false,
  },
})
```

### 2.3 非交互模式

```typescript
async function runPrintMode(prompt: string): Promise<void> {
  // 不启动 REPL，直接执行
  const engine = new QueryEngine({ mode: 'print' })
  const result = await engine.submitMessage(prompt)

  // 打印结果
  console.log(result.content)
  process.exit(0)
}
```

---

## 三、斜杠命令

### 3.1 命令列表

| 命令 | 说明 |
|---|---|
| `/help` | 显示帮助 |
| `/clear` | 清除对话 |
| `/compact` | 手动压缩 |
| `/compact-status` | 压缩状态 |
| `/model` | 切换模型 |
| `/config` | 配置管理 |
| `/permissions` | 权限管理 |
| `/mcp` | MCP 管理 |
| `/memory` | 记忆管理 |
| `/cost` | 显示费用 |
| `/doctor` | 运行诊断 |
| `/init` | 初始化 CLAUDE.md |
| `/bug` | 提交 bug 报告 |
| `/logout` | 登出 |
| `/exit` | 退出 |

### 3.2 命令注册

```typescript
interface SlashCommand {
  name: string
  aliases?: string[]
  description: string
  handler: (args: string[], context: CommandContext) => Promise<void>
}

const commands: Map<string, SlashCommand> = new Map()

function registerCommand(command: SlashCommand): void {
  commands.set(command.name, command)
  command.aliases?.forEach(alias => commands.set(alias, command))
}

// 注册内置命令
registerCommand({
  name: 'help',
  aliases: ['?', 'h'],
  description: 'Show available commands',
  handler: showHelp,
})
```

### 3.3 命令解析

```typescript
function parseSlashCommand(input: string): { command: string; args: string[] } | null {
  // 检查是否以 / 开头
  if (!input.startsWith('/')) return null

  const parts = input.slice(1).split(/\s+/)
  const command = parts[0].toLowerCase()
  const args = parts.slice(1)

  return { command, args }
}
```

### 3.4 命令执行

```typescript
async function handleSlashCommand(
  input: string,
  context: CommandContext
): Promise<boolean> {
  const parsed = parseSlashCommand(input)
  if (!parsed) return false

  const command = commands.get(parsed.command)
  if (!command) {
    console.log(`Unknown command: /${parsed.command}`)
    return true
  }

  await command.handler(parsed.args, context)
  return true
}
```

---

## 四、内置命令详解

### 4.1 /clear

```typescript
async function handleClear(args: string[], context: CommandContext): Promise<void> {
  // 确认清除
  if (args[0] !== '--yes') {
    const confirm = await prompt('Clear conversation? [y/N]: ')
    if (confirm.toLowerCase() !== 'y') return
  }

  // 清除消息
  context.messages = []
  context.toolResults.clear()

  // 清除恢复点
  await fs.unlink(RESUME_FILE).catch(() => {})

  console.log('Conversation cleared.')
}
```

### 4.2 /compact

```typescript
async function handleCompact(args: string[], context: CommandContext): Promise<void> {
  const force = args.includes('--force')

  if (force || shouldCompact(context.messages)) {
    console.log('Compacting conversation...')
    const summary = await compactConversation(context.messages)
    console.log(`Compacted: ${summary}`)
  } else {
    console.log('No need to compact yet.')
  }
}
```

### 4.3 /model

```typescript
async function handleModel(args: string[], context: CommandContext): Promise<void> {
  if (args.length === 0) {
    // 显示当前模型
    console.log(`Current model: ${context.model}`)
    console.log('Available models:')
    MODELS.forEach(m => console.log(`  - ${m}`))
    return
  }

  const newModel = args[0]
  if (!MODELS.includes(newModel)) {
    console.log(`Unknown model: ${newModel}`)
    return
  }

  context.model = newModel
  console.log(`Switched to ${newModel}`)
}
```

### 4.4 /config

```typescript
async function handleConfig(args: string[], context: CommandContext): Promise<void> {
  const action = args[0]

  switch (action) {
    case 'list':
      const config = await loadConfig()
      console.log(JSON.stringify(config, null, 2))
      break

    case 'set':
      const key = args[1]
      const value = args[2]
      await setConfigValue(key, value)
      console.log(`Set ${key} = ${value}`)
      break

    case 'get':
      const getKey = args[1]
      const getValue = await getConfigValue(getKey)
      console.log(`${getKey} = ${getValue}`)
      break

    default:
      console.log('Usage: /config [list|set|get]')
  }
}
```

### 4.5 /permissions

```typescript
async function handlePermissions(args: string[], context: CommandContext): Promise<void> {
  const action = args[0]

  switch (action) {
    case 'list':
      console.log('Permission rules:')
      context.permissionManager.rules.forEach(rule => {
        console.log(`  ${rule.behavior}: ${rule.pattern}`)
      })
      break

    case 'allow':
      await context.permissionManager.addRule({
        pattern: args[1],
        behavior: 'allow',
      })
      console.log(`Added allow rule: ${args[1]}`)
      break

    case 'deny':
      await context.permissionManager.addRule({
        pattern: args[1],
        behavior: 'deny',
      })
      console.log(`Added deny rule: ${args[1]}`)
      break

    case 'reset':
      await context.permissionManager.reset()
      console.log('Permissions reset to defaults')
      break

    default:
      console.log('Usage: /permissions [list|allow|deny|reset]')
  }
}
```

### 4.6 /mcp

```typescript
async function handleMcp(args: string[], context: CommandContext): Promise<void> {
  const action = args[0]

  switch (action) {
    case 'list':
      console.log('MCP servers:')
      context.mcpClients.forEach(client => {
        console.log(`  - ${client.name}: ${client.connected ? 'connected' : 'disconnected'}`)
      })
      break

    case 'tools':
      const tools = await loadMcpTools(context.mcpClients)
      console.log('MCP tools:')
      tools.forEach(tool => {
        console.log(`  - ${tool.name}: ${tool.description}`)
      })
      break

    case 'restart':
      const serverName = args[1]
      await restartMcpServer(serverName)
      console.log(`Restarted ${serverName}`)
      break

    default:
      console.log('Usage: /mcp [list|tools|restart]')
  }
}
```

### 4.7 /memory

```typescript
async function handleMemory(args: string[], context: CommandContext): Promise<void> {
  const action = args[0]

  switch (action) {
    case 'list':
      const files = await listMemoryFiles()
      console.log('Memory files:')
      files.forEach(f => console.log(`  - ${f}`))
      break

    case 'show':
      const content = await readMemoryFile(args[1])
      console.log(content)
      break

    case 'add':
      await addToMemory({ type: 'user', content: args.slice(1).join(' ') })
      console.log('Added to memory')
      break

    case 'search':
      const results = await searchMemory(args.slice(1).join(' '))
      results.forEach(r => {
        console.log(`\n${r.path} (score: ${r.score}):`)
        console.log(r.content.slice(0, 500))
      })
      break

    default:
      console.log('Usage: /memory [list|show|add|search]')
  }
}
```

### 4.8 /cost

```typescript
async function handleCost(args: string[], context: CommandContext): Promise<void> {
  const usage = context.tokenTracker.getUsage()
  const pricing = getPricing(context.model)

  const inputCost = usage.input * pricing.input / 1_000_000
  const outputCost = usage.output * pricing.output / 1_000_000
  const cacheReadCost = usage.cacheRead * pricing.cacheRead / 1_000_000
  const totalCost = inputCost + outputCost + cacheReadCost

  console.log(`Token usage:`)
  console.log(`  Input: ${usage.input.toLocaleString()}`)
  console.log(`  Output: ${usage.output.toLocaleString()}`)
  console.log(`  Cache read: ${usage.cacheRead.toLocaleString()}`)
  console.log()
  console.log(`Estimated cost: $${totalCost.toFixed(4)}`)
}
```

### 4.9 /doctor

```typescript
async function handleDoctor(args: string[], context: CommandContext): Promise<void> {
  console.log('Running diagnostics...\n')

  const checks = [
    checkAuthentication(),
    checkGitInstallation(),
    checkApiKey(),
    checkNetwork(),
    checkDiskSpace(),
    checkPermissions(),
  ]

  for (const check of checks) {
    const result = await check
    const icon = result.passed ? '✓' : '✗'
    console.log(`${icon} ${result.name}: ${result.message}`)
  }
}
```

### 4.10 /init

```typescript
async function handleInit(args: string[], context: CommandContext): Promise<void> {
  const claudeMdPath = path.join(process.cwd(), 'CLAUDE.md')

  if (await exists(claudeMdPath)) {
    console.log('CLAUDE.md already exists')
    return
  }

  const template = `# Project Instructions for Claude

This file contains project-specific instructions for Claude Code.

## Build Commands
- Build: \`npm run build\`
- Test: \`npm test\`
- Lint: \`npm run lint\`

## Code Style
- Use TypeScript
- Follow existing patterns
- Write tests for new features

## Notes
- Add project-specific notes here
`

  await writeFile(claudeMdPath, template)
  console.log('Created CLAUDE.md')
}
```

---

## 五、自定义命令

### 5.1 通过 CLAUDE.md 定义

```markdown
# CLAUDE.md

## Custom Commands

/test: Run all tests with verbose output
/review: Review the current changes for issues
/deploy: Deploy to staging environment
```

### 5.2 命令解析

```typescript
async function loadCustomCommands(claudeMd: string): Promise<SlashCommand[]> {
  const commands: SlashCommand[] = []
  const lines = claudeMd.split('\n')

  for (const line of lines) {
    const match = line.match(/^\/(\w+):\s*(.+)$/)
    if (match) {
      commands.push({
        name: match[1],
        description: match[2],
        handler: async (args, context) => {
          // 将命令转换为用户消息
          await context.submitMessage(match[2])
        },
      })
    }
  }

  return commands
}
```

---

## 六、命令自动补全

### 6.1 Tab 补全

```typescript
function getCommandCompletions(partial: string): string[] {
  const completions: string[] = []

  for (const [name, command] of commands) {
    if (name.startsWith(partial)) {
      completions.push(`/${name}`)
    }
  }

  return completions
}

// 在输入处理中
if (input.startsWith('/')) {
  const completions = getCommandCompletions(input.slice(1))
  if (completions.length === 1) {
    setInput(completions[0] + ' ')
  } else if (completions.length > 1) {
    console.log('\n' + completions.join('  '))
  }
}
```

---

## 七、总结

CLI 命令系统提供两种层次的控制：

1. **CLI 参数**：启动时控制
   - 模型选择
   - 启动模式
   - 权限控制

2. **斜杠命令**：运行时控制
   - 会话管理
   - 配置调整
   - 工具管理

命令系统让用户精确控制 Claude Code 的行为。
