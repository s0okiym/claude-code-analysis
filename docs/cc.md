# Claude Code 源码分析文档

> 基于 2026-03-31 泄露的 Anthropic Claude Code CLI 源码（~1900 文件，~512,000 行 TypeScript）

---

## 一、项目概述

Claude Code 是 Anthropic 官方的终端 AI 编程助手。用户在命令行中与 Claude 对话，Claude 通过调用内置工具（文件读写、Shell 执行、代码搜索等）完成软件工程任务。支持交互式 REPL 和无头 SDK 两种运行模式。

### 技术栈

| 类别 | 技术 |
|---|---|
| 运行时 | Bun |
| 语言 | TypeScript (strict) |
| 终端 UI | React + Ink（React for CLI） |
| CLI 框架 | Commander.js |
| Schema 验证 | Zod v4 |
| 代码搜索 | ripgrep |
| 协议 | MCP SDK, LSP |
| API | Anthropic SDK (Beta Messages API) |
| 遥测 | OpenTelemetry + gRPC |
| Feature Flag | GrowthBook |
| 认证 | OAuth 2.0, JWT, macOS Keychain |

---

## 二、整体架构

```
┌─────────────────────────────────────────────────────────┐
│                      main.tsx (入口)                      │
│  Commander.js 解析 CLI 参数 → 启动 REPL / SDK / 打印模式    │
│  并行预取: MDM配置, Keychain, GrowthBook, API Preconnect   │
└─────────────┬───────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────┐
│                    REPL / SDK 入口层                       │
│  screens/REPL.tsx (交互)  │  QueryEngine (SDK无头)         │
│  Bridge (远程控制)          │  entrypoints/cli.tsx          │
└─────────────┬───────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────┐
│                   query() 核心循环                         │
│  src/query.ts — Agent 循环的主状态机                        │
│  循环: 用户输入 → API调用 → 工具执行 → 再调用 → ... → 结束    │
└──────┬──────────┬───────────┬───────────────────────────┘
       │          │           │
       ▼          ▼           ▼
┌────────────┐ ┌────────┐ ┌──────────────────┐
│ claude.ts  │ │ 工具层  │ │ 上下文/压缩管理    │
│ API 调用层  │ │ tools/  │ │ compact/         │
│ 流式响应    │ │ 30+工具 │ │ context.ts       │
│ 重试/回退   │ │        │ │ tokenBudget.ts   │
└────────────┘ └────────┘ └──────────────────┘
       │          │           │
       ▼          ▼           ▼
┌────────────┐ ┌────────┐ ┌──────────────────┐
│ Anthropic  │ │ 权限系统│ │ MCP Server 连接   │
│ Messages   │ │ permis-│ │ services/mcp/    │
│ API        │ │ sions/ │ │ 第三方工具扩展     │
└────────────┘ └────────┘ └──────────────────┘
```

---

## 三、核心模块详解

### 3.1 入口与启动 — `main.tsx`

**文件规模**: ~4,700 行，整个 CLI 的入口。

**启动流程**:

```
程序入口
  │
  ├─ 1. 并行预取（side-effect imports，在所有模块加载前触发）
  │     ├─ startMdmRawRead() — 读取企业 MDM 配置
  │     ├─ startKeychainPrefetch() — 预读 macOS Keychain (OAuth + API Key)
  │     └─ profileCheckpoint() — 启动性能追踪
  │
  ├─ 2. Commander.js 解析 CLI 参数
  │     ├─ --print / -p (单次问答模式)
  │     ├─ --resume (恢复上次会话)
  │     ├─ --dangerously-skip-permissions
  │     ├─ --model / --max-turns / --allowedTools
  │     └─ 数十个子命令 (session, mcp, config, plugin...)
  │
  ├─ 3. 初始化认证与配置
  │     ├─ GrowthBook Feature Flags 初始化
  │     ├─ OAuth/API Key 验证
  │     ├─ 远程托管设置加载
  │     └─ 策略限制加载 (policy limits)
  │
  ├─ 4. MCP 服务器发现与连接
  │
  ├─ 5. 工具与命令注册
  │     ├─ getTools() — 内置工具集
  │     ├─ getCommands() — 斜杠命令集
  │     └─ getMcpToolsCommandsAndResources() — MCP 工具
  │
  └─ 6. 启动 UI
        ├─ 交互模式 → launchRepl() → screens/REPL.tsx
        ├─ 打印模式 → ask() 一次性调用
        ├─ SDK 模式 → QueryEngine
        └─ Bridge 模式 → 远程控制 (Claude Desktop 等)
```

**关键优化策略**:
- **并行预取**: MDM/Keychain 在模块求值前启动，利用 import 阶段的 ~135ms 并行执行
- **懒加载**: OpenTelemetry (~400KB)、gRPC (~700KB) 通过动态 `import()` 延迟加载
- **Feature Flag 死代码消除**: 通过 Bun 的 `bun:bundle` 的 `feature()` 编译时剔除未启用功能

### 3.2 查询引擎 — `QueryEngine.ts` + `query.ts`

这是 Claude Code 最核心的部分，负责管理整个对话循环。

#### QueryEngine (SDK 入口)

`QueryEngine` 是一个有状态类，每个对话一个实例，支持多轮 `submitMessage()`。

```
QueryEngine
  ├─ mutableMessages: Message[]     // 对话消息历史
  ├─ abortController                // 中断控制
  ├─ totalUsage                     // 累计 token 用量
  ├─ permissionDenials[]            // 权限拒绝记录
  └─ submitMessage(prompt) → AsyncGenerator<SDKMessage>
       │
       ├─ 1. 构建系统提示词 (fetchSystemPromptParts)
       ├─ 2. 处理用户输入 (processUserInput — 解析斜杠命令/附件)
       ├─ 3. 持久化用户消息到 transcript
       ├─ 4. 加载 Skills + Plugins (缓存优先)
       ├─ 5. yield 系统初始化消息
       ├─ 6. 进入 query() 循环
       │     └─ for-await 逐消息 yield 给调用方
       ├─ 7. 检查预算/轮次上限
       └─ 8. yield 最终结果 (result message)
```

#### query() 核心循环 (`src/query.ts`)

`query()` 是 Agent 循环的状态机，实现了 **ReAct 模式**（Reason-Act-Observate）:

```
query() 主循环
  │
  ▼
┌──────────────────────────────┐
│ 检查自动压缩 (autoCompact)     │ ← 上下文超阈值时触发
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ 构建 API 请求参数              │
│ - system prompt + user context│
│ - messages (压缩后)           │
│ - tools schema               │
│ - thinking config            │
│ - max_tokens                 │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ 调用 Anthropic API (流式)     │  claude.ts → Anthropic SDK
│ 处理 stream events:           │
│ - content_block_start/delta  │
│ - message_start/delta/stop   │
│ - thinking blocks            │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ yield 助手消息给调用方         │
└──────────────┬───────────────┘
               │
        ┌──────┴──────┐
        │ stop_reason? │
        └──────┬──────┘
               │
     ┌─────────┼──────────┐
     ▼         ▼          ▼
  end_turn   tool_use  max_tokens
     │         │          │
     │         │     ┌────┘
     │         │     ▼
     │         │  尝试恢复
     │         │  (增大 max_tokens 重试)
     │         │
     │         ▼
     │   ┌──────────────────────┐
     │   │ runTools() — 并发执行  │
     │   │ 工具调用 (toolOrchestration)│
     │   │                      │
     │   │ 1. 权限检查 (canUseTool) │
     │   │ 2. 并发安全的工具并行执行  │
     │   │ 3. 非安全的顺序执行      │
     │   │ 4. 收集 tool_result   │
     │   └──────────┬───────────┘
     │              │
     │              ▼
     │      添加 user message (tool_result)
     │              │
     │              └──────→ 回到循环顶部
     │
     ▼
  yield result → 循环结束
```

**关键策略**:
- **Thinking 模式**: 支持自适应 thinking (`adaptive`)，根据模型能力自动启用 extended thinking
- **Token 预算**: `tokenBudget.ts` 管理每轮的 token 分配，防止超出上下文窗口
- **max_tokens 恢复**: 遇到 `max_output_tokens` 错误时，尝试增大输出上限重试（最多 3 次）
- **流式输出**: 所有 API 调用都是流式的，通过 AsyncGenerator 逐消息 yield

### 3.3 工具系统 — `tools.ts` + `Tool.ts`

#### 工具注册

`getAllBaseTools()` 返回所有内置工具的完整列表:

| 工具 | 用途 |
|---|---|
| `BashTool` | Shell 命令执行 |
| `FileReadTool` | 文件读取 |
| `FileEditTool` | 文件编辑 (精确替换) |
| `FileWriteTool` | 文件写入/创建 |
| `GlobTool` | 文件搜索 (glob pattern) |
| `GrepTool` | 内容搜索 (ripgrep) |
| `AgentTool` | 子 Agent 调用 |
| `WebFetchTool` | URL 内容抓取 |
| `WebSearchTool` | Web 搜索 |
| `NotebookEditTool` | Jupyter Notebook 编辑 |
| `TodoWriteTool` | 待办事项管理 |
| `AskUserQuestionTool` | 向用户提问 |
| `SkillTool` | 技能执行 |
| `TaskCreateTool` / `TaskGetTool` / `TaskUpdateTool` / `TaskListTool` | 任务管理 (V2) |
| `TeamCreateTool` / `TeamDeleteTool` | 多 Agent 团队创建/删除 |
| `SendMessageTool` | Agent 间消息传递 |
| `EnterPlanModeTool` / `ExitPlanModeV2Tool` | 计划模式切换 |
| `BriefTool` | 上下文简报 |
| `ToolSearchTool` | 工具搜索与发现 |
| `ListMcpResourcesTool` / `ReadMcpResourceTool` | MCP 资源访问 |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git Worktree 管理 |
| `ConfigTool` / `LSPTool` / `TungstenTool` | 内部/Ant 专用工具 |

**工具过滤链**:

```
getAllBaseTools()
  → filterToolsByDenyRules()     // 按 deny 规则过滤
  → getTools(permissionCtx)       // 按 mode 过滤 (simple/full)
  → assembleToolPool(mcpTools)    // 合并 MCP 工具，去重
  → 按 name 排序 (prompt-cache 稳定性)
```

#### 工具执行 — `toolOrchestration.ts`

```
runTools(toolUseBlocks, ...)
  │
  ├─ partitionToolCalls() — 分区
  │   ├─ 并发安全 (只读操作如 Read/Grep) → runToolsConcurrently()
  │   └─ 非安全 (写操作如 Edit/Write) → 顺序执行
  │
  └─ 每个工具调用:
      ├─ 权限检查 (canUseTool)
      │   ├─ allow → 执行
      │   ├─ deny → 返回错误
      │   └─ ask → 弹出确认对话框
      │
      ├─ 执行工具 (tool.run())
      ├─ Hook 前后处理 (pre/post tool hooks)
      └─ 返回 tool_result message
```

**并发控制**: 最大并发数 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`，默认 10。

### 3.4 权限系统 — `utils/permissions/`

Claude Code 有一个多层权限模型:

#### 权限模式

| 模式 | 说明 |
|---|---|
| `default` | 每次敏感操作询问用户 |
| `plan` | 计划模式，只允许读取 |
| `auto` | 自动批准（有限制） |
| `bypassPermissions` | 跳过所有权限检查 |

#### 权限检查流程

```
工具调用请求
  │
  ▼
┌───────────────────────────┐
│ 1. 检查 Deny 规则           │
│    - permissionRules.deny  │
│    - 匹配工具名/路径 pattern │
└──────────┬────────────────┘
           │ 未 deny
           ▼
┌───────────────────────────┐
│ 2. 检查 Allow 规则          │
│    - alwaysAllowRules      │
│    - 匹配命令 pattern       │
└──────────┬────────────────┘
           │ 未 allow
           ▼
┌───────────────────────────┐
│ 3. 分类器判定               │
│    - bashClassifier        │
│    - yoloClassifier        │
│    - 判断命令危险级别        │
└──────────┬────────────────┘
           │
           ▼
┌───────────────────────────┐
│ 4. 根据模式决策             │
│    - auto → 自动批准安全操作│
│    - default → 弹窗询问     │
│    - bypass → 全部批准      │
└───────────────────────────┘
```

#### Bash 命令安全分类

`bashClassifier` 和 `yoloClassifier` 将 Shell 命令分为不同安全级别:
- **安全**: `ls`, `cat`, `grep`, `git status` 等只读命令 → 自动批准
- **需确认**: `rm`, `npm install`, `git push` 等修改操作 → 询问用户
- **危险**: `rm -rf /`, 重定向覆盖等 → 明确拒绝或强烈警告

### 3.5 上下文管理与压缩 — `services/compact/`

对话历史会持续增长，Claude Code 通过多层压缩策略管理上下文窗口:

```
对话消息增长
  │
  ▼
┌──────────────────────────┐
│ Token 预算检查              │
│ tokenCountWithEstimation() │
│ vs effectiveContextWindow  │
└──────────┬───────────────┘
           │ 超阈值
           ▼
┌──────────────────────────┐
│ 自动压缩 (autoCompact)     │
│ 1. compactConversation()  │
│    - fork 子 Agent 生成摘要│
│    - 保留最近的 N 条消息    │
│    - 插入 compact_boundary │
│ 2. postCompactCleanup()   │
│    - 释放旧消息内存         │
│ 3. 会话记忆压缩             │
│    - sessionMemoryCompact  │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│ 微压缩 (microCompact)      │
│ - 折叠长工具输出            │
│ - 替换为摘要               │
│ - API 级别 micro-compact   │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│ Snip 压缩 (feature flag)  │
│ - 更激进的历史裁剪          │
│ - 按边界标记删除            │
│ - 用于长时间 SDK 会话       │
└──────────────────────────┘
```

**压缩流程图**:

```
┌────────────────┐     ┌──────────────┐     ┌───────────────┐
│ 原始消息列表    │ ──→ │ Fork 子Agent │ ──→ │ 压缩后消息列表  │
│ [msg1, msg2,   │     │ 生成对话摘要  │     │ [boundary,    │
│  msg3, ...]    │     │              │     │  summary,     │
│                │     │ 保留最近消息  │     │  recent_msgs] │
└────────────────┘     └──────────────┘     └───────────────┘
```

### 3.6 MCP (Model Context Protocol) — `services/mcp/`

Claude Code 通过 MCP 协议连接外部工具服务器:

```
┌─────────────────────────────────┐
│ MCPConnectionManager             │
│ 管理所有 MCP 服务器连接            │
├─────────────────────────────────┤
│ 传输层:                          │
│ ├─ StdioClientTransport  (本地)  │
│ ├─ SSEClientTransport    (远程)  │
│ └─ StreamableHTTPTransport(HTTP)│
├─────────────────────────────────┤
│ 发现:                            │
│ ├─ .claude/mcp.json (项目级)     │
│ ├─ ~/.claude/mcp.json (全局)     │
│ └─ claude.ai 同步的配置          │
├─────────────────────────────────┤
│ 功能:                            │
│ ├─ tools → 注册为可用工具        │
│ ├─ resources → ListMcpResources │
│ ├─ prompts → 命令扩展           │
│ └─ elicitation → URL 认证流     │
└─────────────────────────────────┘
```

### 3.7 多 Agent 系统 — Swarm/Team

Claude Code 支持多 Agent 协作:

```
┌──────────────────────────────────────────┐
│ 主 Agent (Leader)                         │
│ 通过 TeamCreateTool 创建团队               │
├──────────────────────────────────────────┤
│                                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │Worker 1 │  │Worker 2 │  │Worker 3 │  │
│  │(Teammate)│  │(Teammate)│  │(Teammate)│  │
│  └────┬────┘  └────┬────┘  └────┬────┘  │
│       │            │            │        │
│       └─────┬──────┘─────┬──────┘        │
│             │            │               │
│         SendMessageTool  │               │
│         (Agent间通信)     │               │
└─────────────────────────┼────────────────┘
                          │
              ┌───────────┼──────────┐
              ▼           ▼          ▼
         InProcess    Tmux       iTerm2
         Backend     Backend     Backend
         (同进程)    (独立终端)   (macOS)
```

**Agent 类型**:
- **InProcessTeammateTask**: 同进程内通过 `AsyncLocalStorage` 隔离上下文
- **LocalAgentTask**: 本地子进程
- **RemoteAgentTask**: 远程会话
- **DreamTask**: 后台自动任务
- **LocalShellTask**: Shell 任务

**后端选择**: 自动检测可用后端 → 优先 InProcess → Tmux → iTerm2

### 3.8 Bridge 模式 — `bridge/`

Bridge 允许外部应用（如 Claude Desktop）远程控制 Claude Code:

```
┌───────────────┐         ┌──────────────┐
│ Claude Desktop │ ◄─────► │ Claude Code  │
│ (或其他客户端)  │  HTTP/  │ Bridge 模式   │
│               │  WS/SSE │              │
└───────────────┘         └──────┬───────┘
                                 │
                          ┌──────┴───────┐
                          │ sessionRunner │
                          │ 执行会话任务    │
                          └──────────────┘
```

核心组件:
- `bridgeMain.ts` — Bridge 主循环
- `bridgeMessaging.ts` — 消息协议
- `jwtUtils.ts` — JWT 认证
- `replBridge.ts` — REPL 会话桥接
- `sessionRunner.ts` — 会话执行管理

### 3.9 系统提示词构建 — `utils/systemPrompt.ts` + `context.ts`

系统提示词按优先级构建:

```
优先级 0: Override (loop 模式等设置时完全替换)
    ↓
优先级 1: Coordinator 系统提示词 (coordinator 模式)
    ↓
优先级 2: Agent 系统提示词 (主线程 agent)
    │  proactive 模式: 追加到默认
    │  普通模式: 替换默认
    ↓
优先级 3: 自定义系统提示词 (--system-prompt)
    ↓
优先级 4: 默认系统提示词
    │
    ↓
末尾追加: appendSystemPrompt
```

**上下文收集** (`context.ts`):
- Git 状态 (branch, status, recent commits)
- 项目结构 (CLAUDE.md 文件)
- 工作目录信息
- 系统环境信息

### 3.10 插件与技能系统

#### 插件 (`utils/plugins/`)

```
插件来源:
├─ 内置插件 (plugins/bundled/)
├─ 官方市场 (Marketplace)
├─ 本地目录安装
└─ Git 仓库安装

插件能力:
├─ 注册斜杠命令 (commands)
├─ 注册 MCP 服务器 (mcp)
├─ 注册 Hooks (hooks)
├─ 注册输出样式 (outputStyles)
└─ 注册 Agent (agents)
```

#### 技能 (`skills/`)

内置技能: `batch`, `claudeApi`, `debug`, `keybindings`, `loop`, `remember`, `scheduleRemoteAgents`, `simplify`, `skillify`, `stuck`, `verify` 等。

### 3.11 终端 UI — `ink/` + `components/`

基于 Ink（React for CLI）构建的完整终端 UI 系统:

```
ink/ (定制版 Ink 框架)
├─ reconciler.ts    — React Reconciler 适配器
├─ renderer.ts      — 渲染引擎
├─ render-to-screen.ts — 屏幕输出
├─ termio/          — 终端 I/O 解析 (ANSI/CSI/OSC/SGR)
├─ components/      — Box, Text, ScrollBox, Button 等
└─ hooks/           — useInput, useTerminalViewport 等

components/ (~140 个 UI 组件)
├─ PromptInput/     — 用户输入框 (含历史搜索、补全)
├─ Message.tsx      — 消息渲染
├─ Messages.tsx     — 消息列表 (虚拟滚动)
├─ permissions/     — 权限确认对话框
├─ Spinner/         — 加载动画
├─ mcp/             — MCP 配置界面
├─ agents/          — Agent 创建向导
├── tasks/          — 任务面板
└── diff/           — Diff 查看
```

---

## 四、关键设计模式与策略

### 4.1 Feature Flag 死代码消除

通过 Bun 的 `feature()` 编译时剔除代码:

```typescript
const SleepTool = feature('PROACTIVE')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

主要 Flag: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `MONITOR_TOOL`, `COORDINATOR_MODE`, `HISTORY_SNIP`, `CONTEXT_COLLAPSE`, `WEB_BROWSER_TOOL` 等。

### 4.2 Agent 循环 (ReAct Pattern)

```
User Prompt → LLM Call → Tool Use → Observation → LLM Call → ... → Final Answer
                ↑                                         │
                └─────────────────────────────────────────┘
```

核心在 `query.ts`，通过 `for-await` 循环实现流式输出。

### 4.3 并行优化

- **启动并行**: MDM + Keychain + API Preconnect 同时发起
- **工具并行**: 只读工具并发执行，写工具顺序执行
- **Git 状态并行**: branch + status + log + userName 并行 `Promise.all`
- **MCP 资源并行**: `prefetchAllMcpResources()`

### 4.4 分层缓存

- **系统提示词缓存**: 工具按 name 排序保证缓存稳定性
- **文件读取缓存**: `FileStateCache` 避免重复读取
- **设置缓存**: `settingsCache` + `completionCache`
- **插件缓存**: `loadAllPluginsCacheOnly()` 无头模式只读缓存

### 4.5 Hook 系统

生命周期 Hooks 在关键节点触发:
- `PreToolUse` — 工具执行前
- `PostToolUse` — 工具执行后
- `PreCompact` / `PostCompact` — 压缩前后
- `Stop` — 循环结束时
- `PostSampling` — API 采样后
- `SessionStart` — 会话开始时

---

## 五、核心数据流

### 5.1 单次查询完整流程

```
用户输入 "帮我修复 bug"
  │
  ▼
processUserInput()
  ├─ 解析斜杠命令? → 执行本地命令
  └─ 普通文本 → createUserMessage()
  │
  ▼
QueryEngine.submitMessage()
  ├─ fetchSystemPromptParts() → 构建 system prompt
  ├─ processUserInput() → 解析输入
  ├─ recordTranscript() → 持久化
  │
  ▼
query() 循环
  ├─ autoCompactIfNeeded() → 检查是否需要压缩
  ├─ 构建 API 请求 (messages + tools + system + thinking)
  ├─ claude.ts → Anthropic SDK 流式调用
  │   ├─ yield stream_events (实时)
  │   └─ yield assistant message (完成时)
  │
  ├─ stop_reason == "tool_use"?
  │   ├─ YES → runTools()
  │   │   ├─ 权限检查
  │   │   ├─ 并发/顺序执行工具
  │   │   ├─ yield progress messages
  │   │   └─ yield tool_result → 回到循环
  │   │
  │   └─ NO → 结束
  │
  ▼
yield result (成功/错误/预算超限)
```

### 5.2 消息类型流转

```
Message 类型:
├── UserMessage        — 用户输入 / tool_result
├── AssistantMessage   — LLM 响应 (含 tool_use blocks)
├── ProgressMessage    — 工具执行进度
├── AttachmentMessage  — 附件 (文件/图片/structured_output)
├── SystemMessage      — 系统事件 (compact_boundary/api_error)
├── StreamEvent        — API 流事件
├── TombstoneMessage   — 删除标记
└── ToolUseSummaryMessage — 工具使用摘要
```

---

## 六、目录结构总览

```
src/
├── main.tsx                 # 入口 (Commander.js CLI 解析器)
├── QueryEngine.ts           # SDK 查询引擎 (对话状态管理)
├── query.ts                 # 核心 Agent 循环 (ReAct 状态机)
├── tools.ts                 # 工具注册表
├── Tool.ts                  # 工具类型定义
├── commands.ts              # 斜杠命令注册表
├── context.ts               # 系统/用户上下文收集
├── cost-tracker.ts          # Token 费用追踪
│
├── commands/                # 斜杠命令实现 (~50 个)
├── tools/                   # 工具实现 (~30+ 个)
├── components/              # React/Ink UI 组件 (~140 个)
├── hooks/                   # React Hooks
├── services/                # 服务层
│   ├── api/                 #   Anthropic API 客户端
│   ├── compact/             #   上下文压缩
│   ├── mcp/                 #   MCP 协议实现
│   ├── analytics/           #   遥测 (GrowthBook + Datadog)
│   ├── oauth/               #   OAuth 认证
│   ├── plugins/             #   插件管理
│   ├── settings/            #   设置管理
│   ├── lsp/                 #   LSP 集成
│   └── voice/               #   语音模式
├── utils/                   # 工具函数 (最大目录)
│   ├── permissions/         #   权限系统
│   ├── plugins/             #   插件加载/验证
│   ├── bash/                #   Shell 解析器
│   ├── swarm/               #   多 Agent Swarm
│   ├── model/               #   模型配置/别名
│   ├── hooks/               #   Hook 注册表
│   └── telemetry/           #   遥测导出
├── bridge/                  # Bridge 远程控制
├── coordinator/             # 协调者模式
├── memdir/                  # 记忆目录管理
├── skills/                  # 内置技能
├── tasks/                   # 异步任务类型
├── state/                   # 全局状态管理
├── screens/                 # 全屏 UI (REPL/Doctor/Resume)
├── ink/                     # 定制 Ink 框架
└── types/                   # TypeScript 类型定义
```

---

## 七、总结：核心要点

1. **ReAct 循环是骨架**: `query.ts` 实现的 `for-await` Agent 循环是整个系统的核心，不断在 LLM 推理和工具执行之间切换
2. **分层权限是安全基石**: Deny → Allow → Classifier → Mode 四层权限检查，Bash 命令有专门的安全分类器
3. **上下文压缩是生命线**: 自动压缩 + 微压缩 + Snip 三层策略确保长对话不爆上下文窗口
4. **并行优化贯穿始终**: 从启动预取到工具并发执行，系统在每一层都追求并行化
5. **Feature Flag 控制复杂度**: 通过编译时消除控制功能子集，避免运行时开销
6. **MCP 是扩展点**: 通过标准协议接入第三方工具服务器，实现能力扩展
7. **多 Agent 支持团队协作**: Swarm 架构允许多个 Agent 并行工作，通过消息传递协调
8. **流式架构**: 全链路 AsyncGenerator，从 API 到工具执行到 UI 都是流式处理
