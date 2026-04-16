# 子领域 08：多 Agent 系统

> Claude Code 的 Swarm 架构与 Agent 协作机制

---

## 一、概述

Claude Code 支持多 Agent 协作，允许主 Agent 创建子 Agent 并行处理任务。Swarm 架构提供了多种 Agent 类型和后端实现。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/utils/swarm/` | Swarm 核心实现 |
| `src/tools/TeamCreateTool.ts` | 团队创建工具 |
| `src/tools/SendMessageTool.ts` | 消息传递工具 |

---

## 二、Swarm 架构

### 2.1 整体结构

```
┌─────────────────────────────────────────────────────────────┐
│                    主 Agent (Leader)                         │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Team                               │   │
│  │                                                      │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐│   │
│  │  │Worker 1 │  │Worker 2 │  │Worker 3 │  │Worker 4 ││   │
│  │  │(Teammate)│  │(Teammate)│  │(Teammate)│  │(Teammate)│   │
│  │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘│   │
│  │       │            │            │            │      │   │
│  │       └─────┬──────┴─────┬──────┴─────┬──────┘      │   │
│  │             │            │            │              │   │
│  │         SendMessageTool  │            │              │   │
│  │         (Agent 间通信)    │            │              │   │
│  │             │            │            │              │   │
│  └─────────────┼────────────┼────────────┼──────────────┘   │
│                │            │            │                  │
└────────────────┼────────────┼────────────┼──────────────────┘
                 │            │            │
     ┌───────────┼────────────┼────────────┼───────────┐
     │           │            │            │           │
     ▼           ▼            ▼            ▼           ▼
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│InProcess│ │  Tmux   │ │ iTerm2  │ │  Shell  │ │ Remote  │
│ Backend │ │ Backend │ │ Backend │ │ Backend │ │ Backend │
│(同进程) │ │(独立终端)│ │ (macOS) │ │(子进程) │ │ (HTTP)  │
└─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘
```

### 2.2 Agent 类型

| 类型 | 说明 | 用途 |
|---|---|---|
| Leader | 主 Agent | 协调团队，分配任务 |
| Worker (Teammate) | 子 Agent | 执行具体任务 |
| Dream Agent | 后台 Agent | 自动执行预定任务 |

---

## 三、Agent 后端

### 3.1 后端选择

```typescript
function selectBackend(): AgentBackend {
  // 优先级
  // 1. InProcess (同进程，最快)
  // 2. Tmux (独立终端，可观察)
  // 3. iTerm2 (macOS 专用)
  // 4. Shell (子进程)

  if (canUseInProcess()) {
    return new InProcessBackend()
  }

  if (canUseTmux()) {
    return new TmuxBackend()
  }

  if (canUseITerm2()) {
    return new ITerm2Backend()
  }

  return new ShellBackend()
}
```

### 3.2 InProcess Backend

```typescript
class InProcessBackend implements AgentBackend {
  // 使用 AsyncLocalStorage 隔离上下文
  private asyncLocalStorage = new AsyncLocalStorage<AgentContext>()

  async spawn(config: AgentConfig): Promise<AgentHandle> {
    const agentContext: AgentContext = {
      id: generateId(),
      parentContext: getContext(),
      isolatedState: {},
    }

    // 在隔离上下文中运行
    return this.asyncLocalStorage.run(agentContext, async () => {
      const engine = new QueryEngine({
        systemPrompt: config.systemPrompt,
        tools: config.tools,
      })

      return {
        id: agentContext.id,
        submit: async (message: string) => {
          return engine.submitMessage(message)
        },
        kill: async () => {
          engine.abort()
        },
      }
    })
  }
}
```

### 3.3 Tmux Backend

```typescript
class TmuxBackend implements AgentBackend {
  async spawn(config: AgentConfig): Promise<AgentHandle> {
    // 创建新的 tmux 窗口
    const sessionId = `claude-agent-${generateId()}`
    await execCommand(`tmux new-session -d -s ${sessionId}`)

    // 在新窗口中启动 claude
    await execCommand(
      `tmux send-keys -t ${sessionId} 'claude --mode agent' Enter`
    )

    return {
      id: sessionId,
      submit: async (message: string) => {
        await execCommand(
          `tmux send-keys -t ${sessionId} '${escapeForShell(message)}' Enter`
        )
      },
      kill: async () => {
        await execCommand(`tmux kill-session -t ${sessionId}`)
      },
      capture: async () => {
        return await execCommand(`tmux capture-pane -t ${sessionId} -p`)
      },
    }
  }
}
```

### 3.4 iTerm2 Backend

```typescript
class ITerm2Backend implements AgentBackend {
  // 使用 AppleScript 控制 iTerm2

  async spawn(config: AgentConfig): Promise<AgentHandle> {
    const tabId = generateId()

    const script = `
      tell application "iTerm2"
        tell current window
          set newTab to (create tab with default profile)
          tell newTab
            tell current session
              write text "claude --mode agent"
            end tell
          end tell
        end tell
      end tell
    `

    await execCommand(`osascript -e '${script}'`)

    return {
      id: tabId,
      submit: async (message: string) => {
        const sendScript = `
          tell application "iTerm2"
            tell current window
              tell current tab
                tell current session
                  write text "${escapeForAppleScript(message)}"
                end tell
              end tell
            end tell
          end tell
        `
        await execCommand(`osascript -e '${sendScript}'`)
      },
      kill: async () => {
        // 关闭标签页
      },
    }
  }
}
```

---

## 四、团队创建

### 4.1 TeamCreateTool

```typescript
class TeamCreateTool implements Tool {
  name = 'TeamCreate'

  parameters = z.object({
    name: z.string().describe('Team name'),
    teammates: z.array(z.object({
      name: z.string(),
      role: z.string().optional(),
      systemPrompt: z.string().optional(),
    })).describe('Teammate configurations'),
    mode: z.enum(['parallel', 'sequential']).default('parallel'),
  })

  async run(input: TeamCreateInput, context: ToolContext) {
    const team: Team = {
      id: generateId(),
      name: input.name,
      mode: input.mode,
      teammates: [],
    }

    // 创建每个 teammate
    for (const config of input.teammates) {
      const agent = await spawnAgent({
        name: config.name,
        systemPrompt: config.systemPrompt,
        tools: getTeammateTools(),
      })

      team.teammates.push({
        name: config.name,
        role: config.role,
        agent,
      })
    }

    // 注册团队
    registerTeam(team)

    return {
      content: `Team "${input.name}" created with ${input.teammates.length} teammates.`,
      metadata: { teamId: team.id },
    }
  }
}
```

### 4.2 团队模式

```typescript
type TeamMode = 'parallel' | 'sequential'

// 并行模式：所有 teammate 同时工作
async function runParallel(
  team: Team,
  task: string
): Promise<TeammateResult[]> {
  return Promise.all(
    team.teammates.map(t => t.agent.submit(task))
  )
}

// 顺序模式：teammate 依次工作
async function runSequential(
  team: Team,
  task: string
): Promise<TeammateResult[]> {
  const results: TeammateResult[] = []

  for (const teammate of team.teammates) {
    const result = await teammate.agent.submit(task)
    results.push(result)

    // 将结果传递给下一个 teammate
    if (team.teammates.indexOf(teammate) < team.teammates.length - 1) {
      task = `Previous result: ${result.content}\n\nContinue with: ${task}`
    }
  }

  return results
}
```

---

## 五、消息传递

### 5.1 SendMessageTool

```typescript
class SendMessageTool implements Tool {
  name = 'SendMessage'

  parameters = z.object({
    target: z.string().describe('Target agent name or ID'),
    message: z.string().describe('Message to send'),
    waitForResponse: z.boolean().default(true),
  })

  async run(input: SendMessageInput, context: ToolContext) {
    const target = getAgent(input.target)

    if (!target) {
      return {
        content: `Agent "${input.target}" not found`,
        isError: true,
      }
    }

    // 发送消息
    const result = await target.agent.submit(input.message)

    if (input.waitForResponse) {
      // 收集结果
      let response = ''
      for await (const event of result) {
        if (event.type === 'assistant_message') {
          response += event.content
        }
      }

      return { content: response }
    } else {
      return { content: 'Message sent (async)' }
    }
  }
}
```

### 5.2 消息路由

```typescript
class MessageRouter {
  private agents: Map<string, AgentHandle> = new Map()
  private teams: Map<string, Team> = new Map()

  registerAgent(agent: AgentHandle): void {
    this.agents.set(agent.id, agent)
    this.agents.set(agent.name, agent) // 也支持按名称查找
  }

  registerTeam(team: Team): void {
    this.teams.set(team.id, team)
    for (const teammate of team.teammates) {
      this.registerAgent(teammate.agent)
    }
  }

  route(message: OutgoingMessage): void {
    const target = this.agents.get(message.target)
    if (!target) {
      throw new Error(`Unknown target: ${message.target}`)
    }

    target.submit(message.content)
  }
}
```

---

## 六、Agent 任务类型

### 6.1 InProcessTeammateTask

```typescript
class InProcessTeammateTask implements AgentTask {
  type = 'in_process'

  async run(config: AgentConfig): Promise<TaskResult> {
    const engine = new QueryEngine(config)

    // 使用 AsyncGenerator 收集结果
    const messages: Message[] = []
    for await (const event of engine.submitMessage(config.initialPrompt)) {
      messages.push(event)
    }

    return {
      status: 'completed',
      messages,
    }
  }
}
```

### 6.2 LocalAgentTask

```typescript
class LocalAgentTask implements AgentTask {
  type = 'local'

  async run(config: AgentConfig): Promise<TaskResult> {
    // 启动子进程
    const child = spawn('claude', ['--mode', 'agent', '--prompt', config.initialPrompt], {
      stdio: ['pipe', 'pipe', 'pipe'],
    })

    // 收集输出
    let output = ''
    child.stdout.on('data', (data) => {
      output += data
    })

    // 等待完成
    await new Promise(resolve => child.on('close', resolve))

    return {
      status: 'completed',
      output,
    }
  }
}
```

### 6.3 RemoteAgentTask

```typescript
class RemoteAgentTask implements AgentTask {
  type = 'remote'

  async run(config: AgentConfig): Promise<TaskResult> {
    const response = await fetch(`${config.remoteUrl}/api/agent/run`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        prompt: config.initialPrompt,
        systemPrompt: config.systemPrompt,
      }),
    })

    const result = await response.json()

    return {
      status: 'completed',
      messages: result.messages,
    }
  }
}
```

### 6.4 DreamTask

```typescript
// 后台自动任务
class DreamTask implements AgentTask {
  type = 'dream'

  async run(config: AgentConfig): Promise<void> {
    // 在后台持续运行
    const engine = new QueryEngine(config)

    while (!this.shouldStop) {
      // 等待触发
      await this.waitForTrigger()

      // 执行任务
      for await (const event of engine.submitMessage(config.prompt)) {
        // 处理事件
      }

      // 等待下一次触发
      await this.sleep(config.interval || 3600000) // 默认 1 小时
    }
  }
}
```

---

## 七、上下文隔离

### 7.1 隔离机制

```typescript
// 使用 AsyncLocalStorage 实现上下文隔离
const agentContextStorage = new AsyncLocalStorage<AgentContext>()

interface AgentContext {
  id: string
  parentContext: AgentContext | null
  isolatedState: Map<string, any>
  messages: Message[]
}

function getContext(): AgentContext | undefined {
  return agentContextStorage.getStore()
}

function setContextValue(key: string, value: any): void {
  const context = getContext()
  if (context) {
    context.isolatedState.set(key, value)
  }
}

function getContextValue(key: string): any {
  const context = getContext()
  return context?.isolatedState.get(key)
}
```

### 7.2 状态共享

```typescript
// 子 Agent 继承父 Agent 的只读上下文
class InProcessBackend {
  spawn(config: AgentConfig): AgentHandle {
    const parentContext = getContext()

    const childContext: AgentContext = {
      id: generateId(),
      parentContext, // 继承父上下文
      isolatedState: new Map(),
      messages: [],
    }

    return this.asyncLocalStorage.run(childContext, async () => {
      // 子 Agent 运行在隔离上下文中
      // 但可以读取父上下文
    })
  }
}
```

---

## 八、Agent 协调

### 8.1 Coordinator 模式

```typescript
// Coordinator 作为中心协调者
class Coordinator {
  private agents: Map<string, AgentHandle> = new Map()

  async distributeTask(task: string): Promise<Map<string, TaskResult>> {
    const results = new Map<string, TaskResult>()

    // 分析任务，分配给合适的 Agent
    const assignments = await this.analyzeAndAssign(task)

    for (const [agentId, subtask] of assignments) {
      const agent = this.agents.get(agentId)
      const result = await agent.submit(subtask)
      results.set(agentId, result)
    }

    return results
  }

  private async analyzeAndAssign(task: string): Promise<Map<string, string>> {
    // 分析任务，决定分配策略
    // ...
  }
}
```

### 8.2 任务分发示例

```
主 Agent 收到: "帮我完成这个项目"

1. 分析任务:
   - 前端任务 → Worker 1 (前端专家)
   - 后端任务 → Worker 2 (后端专家)
   - 测试任务 → Worker 3 (测试专家)

2. 并行执行:
   Worker 1: 处理前端代码
   Worker 2: 处理后端代码
   Worker 3: 编写测试

3. 汇总结果:
   主 Agent 整合三个 Worker 的结果
```

---

## 九、错误处理

### 9.1 Agent 故障

```typescript
async function handleAgentFailure(
  agent: AgentHandle,
  error: Error
): Promise<void> {
  console.error(`Agent ${agent.name} failed:`, error)

  // 记录失败
  recordFailure(agent.id, error)

  // 决定是否重试
  if (shouldRetry(agent)) {
    await restartAgent(agent)
  } else {
    // 通知团队
    notifyTeam(`Agent ${agent.name} is down`)
  }
}
```

### 9.2 超时处理

```typescript
async function submitWithTimeout(
  agent: AgentHandle,
  message: string,
  timeout: number
): Promise<TaskResult> {
  const controller = new AbortController()

  const timeoutPromise = new Promise<never>((_, reject) => {
    setTimeout(() => {
      controller.abort()
      reject(new Error('Agent timeout'))
    }, timeout)
  })

  const resultPromise = agent.submit(message, { signal: controller.signal })

  return Promise.race([resultPromise, timeoutPromise])
}
```

---

## 十、设计评价

### 10.1 优点

1. **并行能力**：多个 Agent 同时工作
2. **上下文隔离**：AsyncLocalStorage 保证隔离
3. **多后端支持**：InProcess/Tmux/iTerm2/Remote
4. **灵活协调**：Parallel/Sequential 模式

### 10.2 局限性

1. **资源消耗**：多 Agent 增加 token 消耗
2. **协调复杂**：需要处理失败、超时、冲突
3. **调试困难**：多进程/多终端难以调试

### 10.3 最佳实践

1. **合理分配任务**：避免任务重叠
2. **设置超时**：防止 Agent 无限运行
3. **监控资源**：控制 Agent 数量

---

## 十一、总结

多 Agent 系统扩展了 Claude Code 的能力：

1. **Swarm 架构**：主 Agent + Worker Agents
2. **多后端**：InProcess/Tmux/iTerm2/Shell/Remote
3. **上下文隔离**：AsyncLocalStorage
4. **消息传递**：SendMessageTool
5. **协调模式**：Parallel/Sequential

多 Agent 系统让 Claude Code 从单线程变为并行处理平台。
