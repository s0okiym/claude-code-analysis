# Claude Code 综合分析文档

> 基于 2026-03-31 泄露的 Anthropic Claude Code CLI 源码分析与 Python 移植工作空间

---

## 一、项目概述

### 1.1 Claude Code 是什么

Claude Code 是 Anthropic 官方的终端 AI 编程助手。用户在命令行中与 Claude 对话，Claude 通过调用内置工具（文件读写、Shell 执行、代码搜索等）完成软件工程任务。

**核心能力**：
- 交互式 REPL 对话模式
- 无头 SDK 模式（可编程调用）
- 30+ 内置工具
- MCP 协议扩展
- 多 Agent 协作（Swarm）
- 远程控制（Bridge 模式）

### 1.2 技术栈

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

### 1.3 规模

- **~1900 文件**
- **~512,000 行 TypeScript**
- **~140 个 UI 组件**
- **30+ 内置工具**
- **50+ 斜杠命令**

---

## 二、整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    main.tsx (入口)                           │
│  Commander.js 解析 CLI 参数 → 启动 REPL / SDK / 打印模式      │
│  并行预取: MDM配置, Keychain, GrowthBook, API Preconnect     │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│                  REPL / SDK 入口层                           │
│  screens/REPL.tsx (交互)  │  QueryEngine (SDK无头)           │
│  Bridge (远程控制)          │  entrypoints/cli.tsx            │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│                   query() 核心循环                            │
│  src/query.ts — Agent 循环的主状态机                          │
│  循环: 用户输入 → API调用 → 工具执行 → 再调用 → ... → 结束     │
└──────┬──────────┬───────────┬───────────────────────────────┘
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

### 2.1 分层架构

| 层级 | 职责 | 关键文件 |
|---|---|---|
| 入口层 | CLI 解析、启动、预取 | `main.tsx` |
| 会话层 | REPL/SDK/Bridge 模式切换 | `REPL.tsx`, `QueryEngine.ts` |
| 核心循环 | Agent ReAct 循环 | `query.ts` |
| API 层 | Anthropic API 调用 | `claude.ts`, `services/api/` |
| 工具层 | 30+ 内置工具 + MCP 扩展 | `tools/`, `services/mcp/` |
| 权限层 | 多层权限检查 | `utils/permissions/` |
| 上下文层 | Token 管理、压缩 | `services/compact/`, `context.ts` |
| UI 层 | 终端界面 | `components/`, `ink/` |

---

## 三、核心流程

### 3.1 启动流程

```
程序入口
  │
  ├─ 1. 并行预取（side-effect imports，在所有模块加载前触发）
  │     ├─ startMdmRawRead() — 读取企业 MDM 配置
  │     ├─ startKeychainPrefetch() — 预读 macOS Keychain
  │     └─ profileCheckpoint() — 启动性能追踪
  │
  ├─ 2. Commander.js 解析 CLI 参数
  │     ├─ --print / -p (单次问答模式)
  │     ├─ --resume (恢复上次会话)
  │     ├─ --dangerously-skip-permissions
  │     ├─ --model / --max-turns / --allowedTools
  │     └─ 数十个子命令
  │
  ├─ 3. 初始化认证与配置
  │     ├─ GrowthBook Feature Flags 初始化
  │     ├─ OAuth/API Key 验证
  │     └─ 远程托管设置加载
  │
  ├─ 4. MCP 服务器发现与连接
  │
  ├─ 5. 工具与命令注册
  │
  └─ 6. 启动 UI
        ├─ 交互模式 → launchRepl()
        ├─ 打印模式 → ask()
        ├─ SDK 模式 → QueryEngine
        └─ Bridge 模式 → 远程控制
```

### 3.2 Agent 循环 (ReAct Pattern)

```
query() 主循环
  │
  ▼
┌──────────────────────────────┐
│ 检查自动压缩 (autoCompact)     │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ 构建 API 请求参数              │
│ - system prompt + context     │
│ - messages (压缩后)           │
│ - tools schema               │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ 调用 Anthropic API (流式)     │
│ 处理 stream events            │
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
     │         │
     │         ▼
     │   ┌──────────────────────┐
     │   │ runTools() — 并发执行  │
     │   │ 1. 权限检查            │
     │   │ 2. 并发安全的工具并行   │
     │   │ 3. 收集 tool_result   │
     │   └──────────┬───────────┘
     │              │
     │              ▼
     │      添加 tool_result 消息
     │              │
     │              └──────→ 回到循环顶部
     │
     ▼
  yield result → 循环结束
```

### 3.3 工具执行流程

```
runTools(toolUseBlocks)
  │
  ├─ partitionToolCalls() — 分区
  │   ├─ 并发安全 (只读操作) → runToolsConcurrently()
  │   └─ 非安全 (写操作) → 顺序执行
  │
  └─ 每个工具调用:
      ├─ 权限检查 (canUseTool)
      │   ├─ allow → 执行
      │   ├─ deny → 返回错误
      │   └─ ask → 弹出确认对话框
      │
      ├─ 执行工具 (tool.run())
      ├─ Hook 前后处理
      └─ 返回 tool_result message
```

---

## 四、核心设计原理

### 4.1 ReAct 模式

Claude Code 实现了经典的 **ReAct (Reason-Act-Observe)** 模式：

```
User Prompt → LLM Reasoning → Tool Call → Observation → LLM Reasoning → ... → Final Answer
                ↑                                              │
                └──────────────────────────────────────────────┘
```

**核心优势**：
- 模型可以自主决定何时需要工具
- 工具结果提供即时反馈
- 循环直到任务完成

### 4.2 流式架构

全链路 AsyncGenerator，从 API 到工具执行到 UI 都是流式处理：

```typescript
async function* query(): AsyncGenerator<Message> {
  for (;;) {
    const stream = await api.messages.stream(...)
    for await (const event of stream) {
      yield event
    }
    if (stop_reason === 'end_turn') break
    if (stop_reason === 'tool_use') {
      const results = await runTools(...)
      yield* results
    }
  }
}
```

**优势**：
- 用户更快看到响应
- 降低首字延迟
- 支持长时间操作的中断

### 4.3 权限分层

四层权限检查机制：

```
工具调用请求
  │
  ▼
┌───────────────────────────┐
│ 1. 检查 Deny 规则           │
└──────────┬────────────────┘
           │ 未 deny
           ▼
┌───────────────────────────┐
│ 2. 检查 Allow 规则          │
└──────────┬────────────────┘
           │ 未 allow
           ▼
┌───────────────────────────┐
│ 3. 分类器判定               │
│    - bashClassifier        │
│    - yoloClassifier        │
└──────────┬────────────────┘
           │
           ▼
┌───────────────────────────┐
│ 4. 根据模式决策             │
│    - auto/default/bypass   │
└───────────────────────────┘
```

### 4.4 上下文压缩

五层压缩策略：

| 层级 | 触发条件 | 机制 |
|---|---|---|
| 工具结果清除 | 每轮微压缩 | 保留最近 N 个结果 |
| 自动压缩 | token 超阈值 | Fork 子 Agent 生成摘要 |
| Snip 压缩 | 长时间 SDK 会话 | 按边界标记删除 |
| 上下文折叠 | 实验功能 | 完整上下文管理 |
| Reactive Compact | API 返回 413 | 紧急压缩 |

---

## 五、关键子系统

### 5.1 工具系统

30+ 内置工具：

| 类别 | 工具 |
|---|---|
| 文件操作 | FileReadTool, FileEditTool, FileWriteTool |
| 搜索 | GlobTool, GrepTool |
| 执行 | BashTool |
| 网络 | WebFetchTool, WebSearchTool |
| Agent | AgentTool, TeamCreateTool, SendMessageTool |
| 任务 | TodoWriteTool, TaskCreateTool, TaskGetTool |
| 交互 | AskUserQuestionTool |
| MCP | ListMcpResourcesTool, ReadMcpResourceTool |
| 其他 | NotebookEditTool, SkillTool, BriefTool |

### 5.2 权限系统

权限模式：
- `default` — 每次敏感操作询问
- `plan` — 只允许读取
- `auto` — 自动批准安全操作
- `bypassPermissions` — 跳过所有检查

### 5.3 MCP 协议

```
┌─────────────────────────────────┐
│ MCPConnectionManager             │
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
└─────────────────────────────────┘
```

### 5.4 多 Agent 系统

```
┌──────────────────────────────────────────┐
│ 主 Agent (Leader)                         │
├──────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │Worker 1 │  │Worker 2 │  │Worker 3 │  │
│  └────┬────┘  └────┬────┘  └────┬────┘  │
│       └─────┬──────┴─────┬──────┘        │
│             │            │               │
│         SendMessageTool  │               │
└─────────────────────────┼────────────────┘
                          │
              ┌───────────┼──────────┐
              ▼           ▼          ▼
         InProcess    Tmux       iTerm2
         Backend     Backend     Backend
```

### 5.5 Bridge 模式

```
┌───────────────┐         ┌──────────────┐
│ Claude Desktop │ ◄─────► │ Claude Code  │
│ (客户端)       │  HTTP/  │ Bridge 模式   │
│               │  WS/SSE │              │
└───────────────┘         └──────────────┘
```

---

## 六、Context 管理

### 6.1 Context 构成

| 层面 | 内容 | 缓存策略 |
|---|---|---|
| System Prompt | 角色定义、工具指南 | 静态区可跨用户缓存 |
| User Context | CLAUDE.md 指令、日期 | 会话级缓存 |
| System Context | Git 状态 | 会话级缓存 |
| Messages | 对话历史 | 压缩/截断管理 |

### 6.2 System Prompt 结构

```
┌──────────────────────────────┐
│         静态区 (Static)        │  ← 可跨用户缓存
│ 1. 角色定义                    │
│ 2. 系统规则                    │
│ 3. 任务执行                    │
│ 4. 行为准则                    │
│ 5. 工具使用                    │
│ 6. 语气风格                    │
│ 7. 输出效率                    │
├── SYSTEM_PROMPT_DYNAMIC_BOUNDARY ──┤
│         动态区 (Dynamic)        │  ← 会话/用户特定
│ 8. 会话指导                    │
│ 9. 记忆系统                    │
│10. 环境信息                    │
│11. 语言偏好                    │
│12. MCP 指令                    │
└──────────────────────────────┘
```

### 6.3 CLAUDE.md 加载层次

```
优先级 1: Managed Memory  (/etc/claude-code/CLAUDE.md)
优先级 2: User Memory     (~/.claude/CLAUDE.md)
优先级 3: Project Memory  (CLAUDE.md, .claude/CLAUDE.md)
优先级 4: Local Memory    (CLAUDE.local.md)
```

### 6.4 附件系统

每轮对话时注入：
- @文件引用
- IDE 选区
- 剪贴板图片
- 相关记忆（语义搜索）
- 技能发现
- 任务状态
- MCP 资源

---

## 七、设计初衷与权衡

### 7.1 核心设计目标

1. **可扩展性**：MCP 协议允许第三方工具接入
2. **安全性**：多层权限检查防止危险操作
3. **效率**：流式架构、并行执行、分层缓存
4. **连续性**：会话持久化、压缩机制支持长对话
5. **可控性**：Feature Flag 控制功能子集

### 7.2 关键权衡

| 决策 | 收益 | 代价 |
|---|---|---|
| 静态/动态区分离 | Prompt Cache 最大化 | 个性化受限 |
| 工具结果持久化 | 节省 token | 需要额外 Read 操作 |
| Git 状态快照 | 降低延迟 | 状态可能过时 |
| 并行工具执行 | 加速只读操作 | 写操作仍需顺序 |
| 压缩摘要 | 延长对话能力 | 可能丢失细节 |

### 7.3 性能优化策略

- **启动并行**：MDM + Keychain + API Preconnect 同时发起
- **工具并行**：只读工具并发执行，写工具顺序执行
- **懒加载**：OpenTelemetry、gRPC 动态加载
- **Feature Flag 死代码消除**：编译时剔除未启用功能

---

## 八、目录结构总览

```
src/
├── main.tsx                 # 入口
├── QueryEngine.ts           # SDK 查询引擎
├── query.ts                 # 核心 Agent 循环
├── tools.ts                 # 工具注册表
├── Tool.ts                  # 工具类型定义
├── commands.ts              # 斜杠命令注册表
├── context.ts               # 上下文收集
│
├── commands/                # 斜杠命令实现 (~50个)
├── tools/                   # 工具实现 (~30+个)
├── components/              # UI 组件 (~140个)
├── hooks/                   # React Hooks
├── services/
│   ├── api/                 # API 客户端
│   ├── compact/             # 上下文压缩
│   ├── mcp/                 # MCP 协议
│   ├── analytics/           # 遥测
│   ├── oauth/               # 认证
│   ├── plugins/             # 插件管理
│   └── settings/            # 设置管理
├── utils/
│   ├── permissions/         # 权限系统
│   ├── plugins/             # 插件加载
│   ├── bash/                # Shell 解析
│   ├── swarm/               # 多 Agent
│   └── telemetry/           # 遥测导出
├── bridge/                  # Bridge 远程控制
├── coordinator/             # 协调者模式
├── memdir/                  # 记忆目录
├── skills/                  # 内置技能
├── tasks/                   # 异步任务
├── state/                   # 全局状态
├── screens/                 # 全屏 UI
├── ink/                     # 定制 Ink 框架
└── types/                   # 类型定义
```

---

## 九、Python 移植工作空间

当前仓库的 Python 移植工作空间提供：

| 模块 | 职责 | 状态 |
|---|---|---|
| `port_manifest.py` | 工作空间结构摘要 | implemented |
| `models.py` | 数据类定义 | implemented |
| `commands.py` | 命令移植元数据 | implemented |
| `tools.py` | 工具移植元数据 | implemented |
| `query_engine.py` | 渲染移植摘要 | implemented |
| `main.py` | CLI 入口 | implemented |
| `task.py` | 任务结构 | implemented |

---

## 十、子领域文档索引

以下是 20 份子领域详细分析文档：

| 编号 | 文档 | 主题 | 说明 |
|---|---|---|---|
| 01 | [sub_01_agent_loop.md](./sub_01_agent_loop.md) | Agent 循环 | ReAct 模式的核心实现 |
| 02 | [sub_02_tools_system.md](./sub_02_tools_system.md) | 工具系统 | 30+ 内置工具与执行机制 |
| 03 | [sub_03_permissions.md](./sub_03_permissions.md) | 权限系统 | 四层权限检查与分类器 |
| 04 | [sub_04_context_management.md](./sub_04_context_management.md) | 上下文管理 | Token 预算与 Prompt 构建 |
| 05 | [sub_05_compact.md](./sub_05_compact.md) | 上下文压缩 | 五层压缩策略详解 |
| 06 | [sub_06_system_prompt.md](./sub_06_system_prompt.md) | System Prompt | 静态区/动态区分离设计 |
| 07 | [sub_07_mcp_protocol.md](./sub_07_mcp_protocol.md) | MCP 协议 | 第三方工具扩展机制 |
| 08 | [sub_08_multi_agent.md](./sub_08_multi_agent.md) | 多 Agent 系统 | Swarm 架构与协作机制 |
| 09 | [sub_09_bridge_mode.md](./sub_09_bridge_mode.md) | Bridge 模式 | 远程控制与 JWT 认证 |
| 10 | [sub_10_terminal_ui.md](./sub_10_terminal_ui.md) | 终端 UI | 基于 Ink 的 React 终端界面 |
| 11 | [sub_11_attachments.md](./sub_11_attachments.md) | 附件系统 | 动态上下文注入机制 |
| 12 | [sub_12_memory_system.md](./sub_12_memory_system.md) | 记忆系统 | 跨会话知识持久化 |
| 13 | [sub_13_hook_system.md](./sub_13_hook_system.md) | Hook 系统 | 生命周期扩展机制 |
| 14 | [sub_14_feature_flags.md](./sub_14_feature_flags.md) | Feature Flag | 功能控制与死代码消除 |
| 15 | [sub_15_authentication.md](./sub_15_authentication.md) | 认证系统 | OAuth 2.0 与 API Key |
| 16 | [sub_16_parallel_optimization.md](./sub_16_parallel_optimization.md) | 并行优化 | 多层并行执行策略 |
| 17 | [sub_17_startup_flow.md](./sub_17_startup_flow.md) | 启动流程 | 初始化与状态恢复 |
| 18 | [sub_18_cli_commands.md](./sub_18_cli_commands.md) | CLI 命令 | 命令行接口与斜杠命令 |
| 19 | [sub_19_plugin_system.md](./sub_19_plugin_system.md) | 插件系统 | 扩展机制与权限沙箱 |
| 20 | [sub_20_skills_system.md](./sub_20_skills_system.md) | 技能系统 | 轻量级领域专用扩展 |

---

## 十一、总结

Claude Code 是一个复杂的 AI 编程助手，其核心设计围绕：

1. **ReAct 循环** — 持续推理-行动-观察的闭环
2. **流式架构** — 全链路 AsyncGenerator
3. **多层权限** — 安全与灵活性的平衡
4. **上下文管理** — 五层压缩策略应对 token 限制
5. **MCP 扩展** — 标准协议接入第三方能力
6. **多 Agent 协作** — Swarm 架构支持并行工作
7. **Feature Flag** — 编译时控制功能子集

这些设计决策在信息完整性、成本效率和响应延迟之间找到了平衡点。
