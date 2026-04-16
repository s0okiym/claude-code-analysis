# 子领域 09：Bridge 模式

> Claude Code 的远程控制机制

---

## 一、概述

Bridge 模式允许外部应用（如 Claude Desktop）远程控制 Claude Code。通过 HTTP/WebSocket/SSE 协议，外部客户端可以发送请求并接收响应。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/bridge/bridgeMain.ts` | Bridge 主循环 |
| `src/bridge/bridgeMessaging.ts` | 消息协议 |
| `src/bridge/jwtUtils.ts` | JWT 认证 |
| `src/bridge/replBridge.ts` | REPL 桥接 |

---

## 二、Bridge 架构

### 2.1 整体结构

```
┌─────────────────────────────────────────────────────────────┐
│                    外部客户端                                 │
│  (Claude Desktop / IDE Plugin / 其他应用)                    │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       │ HTTP/WebSocket/SSE
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Bridge Server                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Authentication (JWT)                                │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Message Handler                                     │   │
│  │  - Request routing                                   │   │
│  │  - Response streaming                                │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Session Runner                                      │   │
│  │  - QueryEngine integration                           │   │
│  │  - State management                                  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 通信协议

| 协议 | 用途 |
|---|---|
| HTTP POST | 同步请求 |
| WebSocket | 双向实时通信 |
| SSE | 服务器推送事件 |

---

## 三、认证机制

### 3.1 JWT 认证

```typescript
interface BridgeJWT {
  // 标准 claims
  iss: string  // 签发者
  sub: string  // 主题（用户 ID）
  aud: string  // 受众
  exp: number  // 过期时间
  iat: number  // 签发时间

  // 自定义 claims
  sessionId: string  // 会话 ID
  permissions: string[]  // 权限列表
}

function generateBridgeToken(sessionId: string): string {
  const payload: BridgeJWT = {
    iss: 'claude-code',
    sub: getUserId(),
    aud: 'bridge-server',
    exp: Math.floor(Date.now() / 1000) + 3600, // 1 小时
    iat: Math.floor(Date.now() / 1000),
    sessionId,
    permissions: ['read', 'write', 'execute'],
  }

  return jwt.sign(payload, getSecretKey(), { algorithm: 'HS256' })
}

function verifyBridgeToken(token: string): BridgeJWT {
  return jwt.verify(token, getSecretKey()) as BridgeJWT
}
```

### 3.2 Token 交换流程

```
客户端                          Bridge Server
  │                                   │
  │  1. 请求认证                      │
  │  ─────────────────────────────►  │
  │                                   │
  │  2. 返回授权 URL                   │
  │  ◄─────────────────────────────  │
  │                                   │
  │  3. 用户在浏览器中授权              │
  │  ─────────────────────────────►  │
  │                                   │
  │  4. 返回 JWT Token                 │
  │  ◄─────────────────────────────  │
  │                                   │
  │  5. 后续请求携带 Token             │
  │  ─────────────────────────────►  │
  │                                   │
```

---

## 四、消息协议

### 4.1 请求格式

```typescript
interface BridgeRequest {
  id: string           // 请求 ID
  type: RequestType
  payload: any
  timestamp: number
}

type RequestType =
  | 'query'            // 执行查询
  | 'abort'            // 中断当前操作
  | 'getStatus'        // 获取状态
  | 'getHistory'       // 获取历史
  | 'setContext'       // 设置上下文
```

### 4.2 响应格式

```typescript
interface BridgeResponse {
  id: string           // 对应请求 ID
  type: ResponseType
  payload: any
  timestamp: number
}

type ResponseType =
  | 'result'           // 成功结果
  | 'error'            // 错误
  | 'stream'           // 流式事件
  | 'status'           // 状态更新
```

### 4.3 流式响应

```typescript
// SSE 流式响应
interface StreamEvent {
  id: string
  event: string        // 事件类型
  data: string         // JSON 数据
}

// 示例
event: message_start
data: {"type":"message_start","message":{"id":"msg_xxx"}}

event: content_block_delta
data: {"type":"content_block_delta","delta":{"text":"Hello"}}

event: message_stop
data: {"type":"message_stop"}
```

---

## 五、Bridge Server 实现

### 5.1 HTTP Server

```typescript
class BridgeServer {
  private server: http.Server

  start(port: number): void {
    this.server = http.createServer(async (req, res) => {
      // CORS 处理
      res.setHeader('Access-Control-Allow-Origin', '*')
      res.setHeader('Access-Control-Allow-Methods', 'POST, GET, OPTIONS')

      if (req.method === 'OPTIONS') {
        res.writeHead(204)
        res.end()
        return
      }

      // 路由
      const url = new URL(req.url, `http://localhost:${port}`)
      const handler = this.getHandler(url.pathname)

      if (!handler) {
        res.writeHead(404)
        res.end('Not Found')
        return
      }

      await handler(req, res)
    })

    this.server.listen(port)
  }

  private getHandler(pathname: string): Handler | null {
    switch (pathname) {
      case '/query':
        return this.handleQuery
      case '/status':
        return this.handleStatus
      case '/abort':
        return this.handleAbort
      case '/events':
        return this.handleEvents // SSE
      default:
        return null
    }
  }
}
```

### 5.2 Query Handler

```typescript
async function handleQuery(req: IncomingMessage, res: ServerResponse): Promise<void> {
  // 验证 Token
  const token = extractToken(req)
  const claims = verifyBridgeToken(token)

  // 解析请求
  const body = await parseBody(req)
  const request: BridgeRequest = JSON.parse(body)

  // 执行查询
  const engine = getOrCreateSession(claims.sessionId)

  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  })

  try {
    for await (const event of engine.submitMessage(request.payload.message)) {
      const sseEvent: StreamEvent = {
        id: generateId(),
        event: event.type,
        data: JSON.stringify(event),
      }
      res.write(`event: ${sseEvent.event}\n`)
      res.write(`data: ${sseEvent.data}\n\n`)
    }
  } catch (error) {
    const errorEvent: StreamEvent = {
      id: generateId(),
      event: 'error',
      data: JSON.stringify({ error: error.message }),
    }
    res.write(`event: ${errorEvent.event}\n`)
    res.write(`data: ${errorEvent.data}\n\n`)
  }

  res.end()
}
```

### 5.3 SSE Handler

```typescript
async function handleEvents(req: IncomingMessage, res: ServerResponse): Promise<void> {
  const token = extractToken(req)
  const claims = verifyBridgeToken(token)

  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  })

  // 订阅会话事件
  const session = getSession(claims.sessionId)
  const unsubscribe = session.subscribe((event) => {
    res.write(`event: ${event.type}\n`)
    res.write(`data: ${JSON.stringify(event)}\n\n`)
  })

  // 心跳
  const heartbeat = setInterval(() => {
    res.write(': heartbeat\n\n')
  }, 30000)

  // 连接关闭时清理
  req.on('close', () => {
    clearInterval(heartbeat)
    unsubscribe()
  })
}
```

---

## 六、Session Runner

### 6.1 会话管理

```typescript
class SessionRunner {
  private sessions: Map<string, QueryEngine> = new Map()

  getOrCreateSession(sessionId: string): QueryEngine {
    if (!this.sessions.has(sessionId)) {
      const engine = new QueryEngine({
        mode: 'bridge',
        onEvent: (event) => this.broadcast(sessionId, event),
      })
      this.sessions.set(sessionId, engine)
    }
    return this.sessions.get(sessionId)!
  }

  broadcast(sessionId: string, event: Event): void {
    // 广播给所有订阅者
    const subscribers = this.getSubscribers(sessionId)
    for (const subscriber of subscribers) {
      subscriber(event)
    }
  }
}
```

### 6.2 状态同步

```typescript
interface SessionState {
  status: 'idle' | 'running' | 'error'
  currentTask?: string
  lastActivity: number
  messageCount: number
}

async function getSessionState(sessionId: string): Promise<SessionState> {
  const engine = getSession(sessionId)

  return {
    status: engine.status,
    currentTask: engine.currentTask,
    lastActivity: engine.lastActivity,
    messageCount: engine.messageCount,
  }
}
```

---

## 七、REPL Bridge

### 7.1 REPL 模式桥接

```typescript
class REPLBridge {
  private repl: REPLSession
  private messageQueue: Message[] = []

  connect(repl: REPLSession): void {
    this.repl = repl

    // 监听 REPL 输出
    repl.onOutput((output) => {
      this.broadcast({
        type: 'repl_output',
        content: output,
      })
    })

    // 监听 REPL 状态
    repl.onStateChange((state) => {
      this.broadcast({
        type: 'repl_state',
        state,
      })
    })
  }

  // 处理来自 Bridge 的输入
  handleInput(input: string): void {
    this.repl.handleInput(input)
  }
}
```

### 7.2 UI 同步

```typescript
// 将 Bridge 消息同步到 UI
function syncToUI(event: BridgeEvent): void {
  // 更新 React 状态
  updateReactState({
    type: event.type,
    payload: event.payload,
  })
}
```

---

## 八、安全考虑

### 8.1 Token 安全

```typescript
// Token 过期检查
function checkTokenExpiry(claims: BridgeJWT): void {
  if (Date.now() / 1000 > claims.exp) {
    throw new Error('Token expired')
  }
}

// Token 刷新
async function refreshToken(oldToken: string): Promise<string> {
  const claims = verifyBridgeToken(oldToken)
  return generateBridgeToken(claims.sessionId)
}
```

### 8.2 权限控制

```typescript
function checkPermission(claims: BridgeJWT, action: string): void {
  if (!claims.permissions.includes(action)) {
    throw new Error(`Permission denied: ${action}`)
  }
}

// 使用中间件
function authMiddleware(requiredPermission: string): Middleware {
  return (req, res, next) => {
    const token = extractToken(req)
    const claims = verifyBridgeToken(token)
    checkTokenExpiry(claims)
    checkPermission(claims, requiredPermission)
    req.claims = claims
    next()
  }
}
```

### 8.3 速率限制

```typescript
const rateLimiter = new RateLimiter({
  windowMs: 60000,  // 1 分钟
  maxRequests: 100, // 最多 100 请求
})

function rateLimitMiddleware(req, res, next): void {
  const clientId = getClientId(req)
  if (!rateLimiter.check(clientId)) {
    res.writeHead(429, { 'Content-Type': 'application/json' })
    res.end(JSON.stringify({ error: 'Too many requests' }))
    return
  }
  next()
}
```

---

## 九、客户端集成

### 9.1 Claude Desktop 集成

```typescript
// Claude Desktop 端代码
class ClaudeDesktopBridge {
  private eventSource: EventSource

  connect(sessionId: string): void {
    // 建立 SSE 连接
    this.eventSource = new EventSource(
      `http://localhost:3000/events?session=${sessionId}`
    )

    this.eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data)
      this.handleEvent(data)
    }
  }

  async sendMessage(message: string): Promise<void> {
    const response = await fetch('http://localhost:3000/query', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.token}`,
      },
      body: JSON.stringify({
        id: generateId(),
        type: 'query',
        payload: { message },
        timestamp: Date.now(),
      }),
    })

    // 处理流式响应
    // ...
  }
}
```

### 9.2 IDE 插件集成

```typescript
// VSCode 扩展端代码
class VSCodeBridge {
  private client: BridgeClient

  async activate(context: ExtensionContext): Promise<void> {
    this.client = new BridgeClient({
      url: 'http://localhost:3000',
      onMessage: (msg) => this.showInOutputChannel(msg),
    })

    // 注册命令
    context.subscriptions.push(
      commands.registerCommand('claudeCode.sendSelection', async () => {
        const editor = window.activeTextEditor
        const selection = editor.selection
        const text = editor.document.getText(selection)
        await this.client.sendQuery(text)
      })
    )
  }
}
```

---

## 十、设计评价

### 10.1 优点

1. **远程控制**：外部应用可以控制 Claude Code
2. **实时同步**：SSE 提供实时事件推送
3. **安全认证**：JWT + 权限控制
4. **多客户端**：支持多个客户端同时连接

### 10.2 局限性

1. **网络依赖**：需要 HTTP 服务
2. **延迟**：网络传输增加延迟
3. **复杂度**：额外的认证和状态管理

### 10.3 最佳实践

1. **Token 短期有效**：减少泄露风险
2. **心跳机制**：检测连接状态
3. **优雅降级**：网络断开时保存状态

---

## 十一、总结

Bridge 模式扩展了 Claude Code 的使用场景：

1. **远程控制**：外部应用可发送请求
2. **SSE 流式**：实时推送事件
3. **JWT 认证**：安全的 Token 机制
4. **多客户端**：支持多个外部应用

Bridge 模式让 Claude Code 从 CLI 工具变为可集成的服务。
