# 子领域 07：MCP 协议

> Model Context Protocol - Claude Code 的第三方工具扩展机制

---

## 一、概述

MCP (Model Context Protocol) 是 Claude Code 接入第三方工具的标准协议。通过 MCP，可以连接外部工具服务器，扩展 Claude 的能力。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/services/mcp/` | MCP 实现核心 |
| `src/services/mcp/MCPConnectionManager.ts` | 连接管理 |
| `src/services/mcp/transport/` | 传输层实现 |

---

## 二、MCP 架构

### 2.1 整体结构

```
┌─────────────────────────────────────────────────────────────┐
│                    Claude Code (MCP Client)                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              MCPConnectionManager                    │   │
│  │                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐          │   │
│  │  │ filesystem      │  │ github          │   ...    │   │
│  │  │ MCP Server      │  │ MCP Server      │          │   │
│  │  └────────┬────────┘  └────────┬────────┘          │   │
│  │           │                    │                    │   │
│  └───────────┼────────────────────┼────────────────────┘   │
│              │                    │                        │
└──────────────┼────────────────────┼────────────────────────┘
               │                    │
        ┌──────┴──────┐      ┌──────┴──────┐
        │ Stdio       │      │ SSE/HTTP    │
        │ Transport   │      │ Transport   │
        └──────┬──────┘      └──────┬──────┘
               │                    │
               ▼                    ▼
        ┌──────────────┐    ┌──────────────┐
        │ Local Server │    │ Remote Server │
        │ (stdio)      │    │ (HTTP/SSE)   │
        └──────────────┘    └──────────────┘
```

### 2.2 MCP 能力

| 能力 | 说明 |
|---|---|
| Tools | 注册工具供 Claude 调用 |
| Resources | 提供可读取的资源 |
| Prompts | 注册斜杠命令 |
| Elicitation | 启动外部认证流程 |

---

## 三、MCP 连接管理

### 3.1 MCPConnectionManager

```typescript
class MCPConnectionManager {
  private connections: Map<string, MCPConnection> = new Map()
  private mcpClients: MCPClient[] = []

  // 初始化所有 MCP 服务器
  async initialize(config: MCPConfig): Promise<void> {
    for (const [name, serverConfig] of Object.entries(config.servers)) {
      await this.connectServer(name, serverConfig)
    }
  }

  // 连接单个服务器
  private async connectServer(
    name: string,
    config: MCPServerConfig
  ): Promise<void> {
    const transport = this.createTransport(config)
    const client = new MCPClient({ name, transport })

    await client.connect()
    this.connections.set(name, { client, config })
    this.mcpClients.push(client)
  }

  // 创建传输层
  private createTransport(config: MCPServerConfig): Transport {
    if (config.command) {
      // Stdio 传输
      return new StdioClientTransport({
        command: config.command,
        args: config.args || [],
        env: config.env,
      })
    }

    if (config.url) {
      // HTTP/SSE 传输
      if (config.transport === 'sse') {
        return new SSEClientTransport({ url: config.url })
      }
      return new StreamableHTTPTransport({ url: config.url })
    }

    throw new Error('Invalid MCP server config')
  }
}
```

### 3.2 连接配置

```json
// ~/.claude/mcp.json
{
  "servers": {
    "filesystem": {
      "command": "mcp-server-filesystem",
      "args": ["/home/user/projects"]
    },
    "github": {
      "url": "https://mcp.github.com",
      "transport": "sse"
    },
    "postgres": {
      "command": "mcp-server-postgres",
      "args": ["postgresql://localhost/mydb"]
    }
  }
}
```

---

## 四、传输层

### 4.1 Stdio 传输

```typescript
class StdioClientTransport implements Transport {
  private process: ChildProcess
  private messageHandler: (message: MCPMessage) => void

  async connect(): Promise<void> {
    this.process = spawn(this.config.command, this.config.args, {
      env: { ...process.env, ...this.config.env },
      stdio: ['pipe', 'pipe', 'pipe'],
    })

    // 监听 stdout 获取消息
    this.process.stdout.on('data', (data) => {
      const message = this.parseMessage(data)
      this.messageHandler(message)
    })

    // 监听 stderr 获取日志
    this.process.stderr.on('data', (data) => {
      console.error(`[MCP ${this.name}] ${data}`)
    })
  }

  async send(message: MCPMessage): Promise<void> {
    const json = JSON.stringify(message)
    this.process.stdin.write(json + '\n')
  }

  async close(): Promise<void> {
    this.process.kill()
  }
}
```

### 4.2 SSE 传输

```typescript
class SSEClientTransport implements Transport {
  private eventSource: EventSource
  private messageHandler: (message: MCPMessage) => void

  async connect(): Promise<void> {
    this.eventSource = new EventSource(this.config.url)

    this.eventSource.onmessage = (event) => {
      const message = JSON.parse(event.data)
      this.messageHandler(message)
    }

    this.eventSource.onerror = (error) => {
      console.error('SSE connection error:', error)
    }
  }

  async send(message: MCPMessage): Promise<void> {
    await fetch(this.config.url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(message),
    })
  }

  async close(): Promise<void> {
    this.eventSource.close()
  }
}
```

---

## 五、MCP 工具

### 5.1 工具发现

```typescript
async function loadMcpTools(mcpClients: MCPClient[]): Promise<Tool[]> {
  const tools: Tool[] = []

  for (const client of mcpClients) {
    try {
      const response = await client.request('tools/list', {})
      
      for (const tool of response.tools) {
        tools.push(convertMcpToolToLocal(tool, client))
      }
    } catch (error) {
      console.error(`Failed to load tools from ${client.name}:`, error)
    }
  }

  return tools
}

function convertMcpToolToLocal(mcpTool: MCPTool, client: MCPClient): Tool {
  return {
    name: `mcp_${client.name}_${mcpTool.name}`,
    description: mcpTool.description,
    parameters: convertMcpSchemaToZod(mcpTool.inputSchema),
    run: async (input, context) => {
      return await client.callTool(mcpTool.name, input)
    },
  }
}
```

### 5.2 工具调用

```typescript
class MCPClient {
  async callTool(name: string, args: any): Promise<ToolResult> {
    const response = await this.request('tools/call', {
      name,
      arguments: args,
    })

    return {
      content: response.content,
      isError: response.isError,
    }
  }
}
```

### 5.3 工具命名

```
格式: mcp_{serverName}_{toolName}

示例:
- mcp_filesystem_read_file
- mcp_filesystem_write_file
- mcp_github_create_issue
- mcp_github_create_pull_request
- mcp_postgres_query
```

---

## 六、MCP 资源

### 6.1 资源发现

```typescript
async function loadMcpResources(mcpClients: MCPClient[]): Promise<MCPResource[]> {
  const resources: MCPResource[] = []

  for (const client of mcpClients) {
    try {
      const response = await client.request('resources/list', {})
      resources.push(...response.resources.map(r => ({
        serverName: client.name,
        ...r,
      })))
    } catch (error) {
      console.error(`Failed to load resources from ${client.name}:`, error)
    }
  }

  return resources
}
```

### 6.2 资源读取

```typescript
class ReadMcpResourceTool implements Tool {
  name = 'ReadMcpResource'
  
  async run(input: { uri: string }, context: ToolContext) {
    const { serverName, resourcePath } = parseMcpUri(input.uri)
    const client = getMcpClient(serverName)
    
    const response = await client.request('resources/read', {
      uri: resourcePath,
    })
    
    return { content: response.contents }
  }
}
```

---

## 七、MCP Prompts

### 7.1 Prompt 注册

```typescript
async function loadMcpPrompts(mcpClients: MCPClient[]): Promise<Command[]> {
  const commands: Command[] = []

  for (const client of mcpClients) {
    try {
      const response = await client.request('prompts/list', {})
      
      for (const prompt of response.prompts) {
        commands.push({
          name: `mcp_${client.name}_${prompt.name}`,
          description: prompt.description,
          handler: async (args) => {
            const result = await client.request('prompts/get', {
              name: prompt.name,
              arguments: args,
            })
            return result.messages
          },
        })
      }
    } catch (error) {
      console.error(`Failed to load prompts from ${client.name}:`, error)
    }
  }

  return commands
}
```

---

## 八、Elicitation 认证

### 8.1 认证流程

```
┌───────────────┐         ┌──────────────┐         ┌──────────────┐
│ Claude Code   │         │ MCP Server   │         │ OAuth Provider│
│               │         │              │         │              │
│  1. 调用工具   │ ──────► │              │         │              │
│               │         │ 2. 需要认证   │         │              │
│               │ ◄────── │   elicitation │         │              │
│               │         │   request    │         │              │
│ 3. 打开浏览器 │ ────────────────────────►│ 4. 用户登录  │
│               │         │              │ ◄────── │              │
│               │         │ 5. 回调      │         │ 6. 授权码    │
│               │         │              │ ──────► │              │
│               │         │ 7. 获取 token│ ◄────── │ 7. 返回 token│
│               │ ◄────── │ 8. 认证完成  │         │              │
│ 9. 重试调用   │ ──────► │ 10. 成功执行 │         │              │
└───────────────┘         └──────────────┘         └──────────────┘
```

### 8.2 代码实现

```typescript
async function handleElicitation(
  elicitation: ElicitationRequest,
  context: ToolContext
): Promise<ElicitationResponse> {
  // 打开浏览器
  await openBrowser(elicitation.url)

  // 等待回调
  const callbackResult = await waitForCallback(elicitation.callbackUrl, {
    timeout: 120000, // 2 分钟超时
  })

  return {
    type: 'elicitation_response',
    callback: callbackResult,
  }
}
```

---

## 九、MCP 配置发现

### 9.1 配置来源

```
优先级 1 (最低): 系统配置
  └─ /etc/claude-code/mcp.json

优先级 2: 用户配置
  └─ ~/.claude/mcp.json

优先级 3: 项目配置
  └─ .claude/mcp.json

优先级 4 (最高): claude.ai 同步
  └─ 云端同步的配置
```

### 9.2 配置合并

```typescript
async function loadMcpConfig(): Promise<MCPConfig> {
  const configs: MCPConfig[] = []

  // 系统配置
  const systemConfig = await readJson('/etc/claude-code/mcp.json')
  if (systemConfig) configs.push(systemConfig)

  // 用户配置
  const userConfig = await readJson(path.join(os.homedir(), '.claude/mcp.json'))
  if (userConfig) configs.push(userConfig)

  // 项目配置
  const projectConfig = await readJson('.claude/mcp.json')
  if (projectConfig) configs.push(projectConfig)

  // 云端同步
  const cloudConfig = await fetchCloudMcpConfig()
  if (cloudConfig) configs.push(cloudConfig)

  // 合并（后加载的覆盖前面的）
  return mergeConfigs(configs)
}
```

---

## 十、MCP 生命周期

### 10.1 连接状态

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Disconnected│ ──► │ Connecting  │ ──► │  Connected  │
└─────────────┘     └─────────────┘     └──────┬──────┘
       ▲                                       │
       │                                       │
       │           ┌─────────────┐             │
       │           │   Error     │ ◄───────────┤
       │           └─────────────┘             │
       │                                       │
       └───────────────────────────────────────┘
                    Reconnect
```

### 10.2 重连机制

```typescript
class MCPConnection {
  private reconnectAttempts = 0
  private maxReconnectAttempts = 3

  async handleDisconnect(): Promise<void> {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error(`MCP server ${this.name} failed to reconnect`)
      return
    }

    this.reconnectAttempts++
    const delay = Math.pow(2, this.reconnectAttempts) * 1000 // 指数退避

    await new Promise(resolve => setTimeout(resolve, delay))

    try {
      await this.connect()
      this.reconnectAttempts = 0
    } catch (error) {
      await this.handleDisconnect()
    }
  }
}
```

---

## 十一、MCP 指令注入

### 11.1 静态指令

MCP 服务器可以提供自定义指令：

```typescript
async function loadMcpInstructions(mcpClients: MCPClient[]): Promise<string[]> {
  const instructions: string[] = []

  for (const client of mcpClients) {
    if (client.instructions) {
      instructions.push(`## ${client.name}\n${client.instructions}`)
    }
  }

  return instructions
}
```

### 11.2 Delta 模式

```typescript
// 使用 delta 模式避免每次重连破坏缓存
function buildMcpInstructionsSection(
  previous: MCPInstruction[],
  current: MCPInstruction[]
): string {
  const delta = calculateDelta(previous, current)

  if (delta.added.length === 0 && delta.removed.length === 0) {
    return '' // 无变化
  }

  let section = '## MCP Server Changes\n\n'

  if (delta.added.length > 0) {
    section += 'Added:\n'
    for (const add of delta.added) {
      section += `- ${add.name}: ${add.instructions}\n`
    }
  }

  if (delta.removed.length > 0) {
    section += 'Removed:\n'
    for (const remove of delta.removed) {
      section += `- ${remove}\n`
    }
  }

  return section
}
```

---

## 十二、设计评价

### 12.1 优点

1. **标准化协议**：统一接口，易于扩展
2. **多种传输**：支持本地和远程服务器
3. **能力丰富**：Tools、Resources、Prompts
4. **认证支持**：Elicitation 流程处理外部认证

### 12.2 局限性

1. **连接复杂性**：需要管理多个连接状态
2. **错误传播**：MCP 服务器错误可能影响主进程
3. **缓存挑战**：动态 MCP 指令影响 Prompt Cache

### 12.3 最佳实践

1. **使用 Delta 模式**：减少缓存失效
2. **限制 MCP 服务器数量**：避免过多连接
3. **错误隔离**：MCP 服务器错误不应崩溃主进程

---

## 十三、总结

MCP 是 Claude Code 扩展能力的核心机制：

1. **标准协议**：统一的工具扩展接口
2. **多种传输**：Stdio、SSE、HTTP
3. **丰富能力**：Tools、Resources、Prompts
4. **认证支持**：Elicitation 流程
5. **配置发现**：多层次配置合并

MCP 让 Claude Code 从封闭系统变为开放平台。
