# 第一章：软件架构与程序入口

[返回总目录](../README.md)

## 1. 本章导读

本章回答两个问题：

1. 这个项目从哪里启动，主链路如何流转。
2. 它的整体架构是怎样分层的，各层分别承担什么职责。

结论先行：这个项目不是单纯的命令行聊天程序，而是一套本地 agent 平台。它采用“CLI 引导层 + TUI/REPL 交互层 + Query/Agent 执行内核 + Tool/Permission 层 + Memory/Persistence 层 + MCP/Remote/Swarm 扩展层”的分层结构。

先给出两个总图，后面各节再分别展开。

### 1.1 总体分层图

```text
+------------------------------+
| CLI 引导层                   |
| entrypoints/cli.tsx          |
| main.tsx                     |
+------------------------------+
               |
               v
+------------------------------+
| 初始化层                     |
| init.ts / setup.ts           |
+------------------------------+
      |                  |
      v                  v
+------------------+   +------------------------------+
| 控制面 / 命令层  |   | TUI / REPL 层                |
| commands.ts      |-->| replLauncher.tsx / REPL.tsx  |
| slash/menu       |   +------------------------------+
+------------------+                  |
                                      v
                         +------------------------------+
                         | 执行内核                     |
                         | query.ts / QueryEngine.ts    |
                         +------------------------------+
                           |            |            |
                           v            v            v
                 +---------------+ +-----------------+ +------------------+
                 | Tool/Perm 层  | | Memory/Persist  | | 扩展层           |
                 | Tool.ts       | | sessionStorage  | | MCP/Plugin/      |
                 | orchestration | | memdir/SM       | | Remote/Swarm     |
                 +---------------+ +-----------------+ +------------------+

说明：
- 初始化层先把运行环境建立起来。
- 控制面决定如何进入工作台。
- 执行内核向下调用工具、memory 与扩展层。
```

### 1.2 默认交互主链路图

```text
entrypoints/cli.tsx
  -> main.tsx
  -> init.ts + setup.ts
  -> launchRepl()
  -> App + REPL
  -> PromptInput / slash command / footer 菜单
  -> query()
     -> services/api/claude.ts
     -> runTools() / StreamingToolExecutor
     -> sessionStorage / SessionMemory / compact / hooks
     -> 返回到 query 主循环
```

## 2. 程序入口

### 2.1 轻量入口

主入口从 [`src/entrypoints/cli.tsx`](../src/entrypoints/cli.tsx) 开始。

这一层的职责不是运行完整系统，而是做“早期分流”：

- `--version` 等快路径直接返回
- `remote-control` / `bridge` / `daemon` 等模式做专门入口分发
- 某些内置 MCP server 或浏览器桥接模式提前切换执行路径

这种设计的好处是：普通快速命令不需要加载整个应用，提高了启动速度，也降低了初始化副作用。

用流程图看，这一层更像“入口分流器”而不是完整应用：

```text
进程启动
  -> 读取 argv
  -> 判断是否命中快路径
     -> 是：直接走专门入口并退出
        - --version
        - --dump-system-prompt
        - remote-control / daemon / bg / runner
     -> 否：进入 main.tsx 主启动器
```

### 2.2 主启动器

完整交互与主逻辑由 [`src/main.tsx`](../src/main.tsx) 接管。

这里是实际的总控入口，负责：

- 解析 CLI 参数
- 确定 permission mode
- 初始化模型、工具、命令、agents、MCP、plugins
- 选择进入 REPL、headless/SDK、bridge、remote 等路径

从结构上看，`main.tsx` 是系统编排中心，不是单纯的 UI 启动文件。

### 2.3 初始化与环境准备

初始化分为两部分：

- [`src/entrypoints/init.ts`](../src/entrypoints/init.ts)
- [`src/setup.ts`](../src/setup.ts)

`init.ts` 负责：

- 启用配置系统
- 应用安全环境变量
- 证书、代理、HTTP agent、遥测骨架初始化
- trust 建立后的 telemetry 延迟初始化

`setup.ts` 负责：

- 设置 cwd
- 启动 hooks 监听
- worktree / tmux / teammate snapshot / terminal 恢复
- session memory 初始化
- team memory watcher 启动

这说明项目把“逻辑初始化”和“工作目录/运行环境初始化”分开了，边界比较清晰。

## 3. 运行形态

### 3.1 默认 REPL/TUI

默认交互模式使用：

- [`src/replLauncher.tsx`](../src/replLauncher.tsx)
- [`src/screens/REPL.tsx`](../src/screens/REPL.tsx)

运行流大致是：

1. `main.tsx` 准备上下文
2. `launchRepl()` 动态加载 `App + REPL`
3. `REPL.tsx` 维护消息、输入框、权限弹窗、任务、远程状态
4. 用户输入进入 `query()`

### 3.2 Headless / SDK

非交互或 SDK 场景主要走：

- [`src/QueryEngine.ts`](../src/QueryEngine.ts)
- [`src/query.ts`](../src/query.ts)

这里的思路是：

- `query.ts` 管单次 query 循环
- `QueryEngine.ts` 管跨多轮会话状态

这意味着 UI 层和会话执行内核不是一体化耦合的，后续可复用于 SDK、后台任务、子 agent。

### 3.3 MCP Server 形态

项目还支持把自己暴露为 MCP server：

- [`src/entrypoints/mcp.ts`](../src/entrypoints/mcp.ts)

这条路径会把部分内部工具重新包装为 MCP tools，对外提供调用能力。也就是说，这个项目既能作为 MCP client，也能作为 MCP server。

### 3.4 Remote / Bridge 形态

远程与 bridge 模式核心在：

- [`src/bridge/bridgeMain.ts`](../src/bridge/bridgeMain.ts)

这条链路支持：

- 远程控制
- 会话桥接
- 远程环境会话拉起
- 心跳、重连、session ingress

这让项目从“本地终端工具”扩展成“本地与远程混合 agent 平台”。

## 4. 架构分层

### 4.1 命令与模式分发层

核心文件：

- [`src/commands.ts`](../src/commands.ts)

职责：

- 汇总所有 slash commands / local commands
- 按 feature gate、环境、模式进行启停控制
- 把命令系统和主执行内核解耦

这一层相当于“控制面”。

### 4.2 TUI 与状态层

核心文件：

- [`src/screens/REPL.tsx`](../src/screens/REPL.tsx)
- [`src/state/AppStateStore.ts`](../src/state/AppStateStore.ts)

特点：

- 不是简单 readline
- 有完整 AppState
- 管理任务、插件、MCP、agent registry、notifications、file history、permission context、remote bridge 状态

这一层相当于“终端工作台”。

### 4.3 Query / Agent 执行内核

核心文件：

- [`src/query.ts`](../src/query.ts)
- [`src/QueryEngine.ts`](../src/QueryEngine.ts)

职责：

- 组织消息历史
- 构造 system prompt / user context / system context
- 调用模型 API
- 处理 tool use / tool result
- compact / summarize / hooks / error recovery

这是系统的“执行面”。

### 4.4 Tool 与 Permission 层

核心文件：

- [`src/tools.ts`](../src/tools.ts)
- [`src/Tool.ts`](../src/Tool.ts)
- [`src/services/tools/toolOrchestration.ts`](../src/services/tools/toolOrchestration.ts)
- [`src/utils/permissions/permissionSetup.ts`](../src/utils/permissions/permissionSetup.ts)

特点：

- 工具统一 schema 化
- 支持 enablement、权限、prompt、执行接口
- 并发安全工具并发，非安全工具串行
- permission mode 是系统级对象，不是 UI 附加框

### 4.5 Persistence / Transcript / Memory 层

核心文件：

- [`src/utils/sessionStorage.ts`](../src/utils/sessionStorage.ts)
- [`src/memdir/*`](../src/memdir)
- [`src/services/SessionMemory/*`](../src/services/SessionMemory)
- [`src/services/extractMemories/*`](../src/services/extractMemories)

职责：

- transcript 落盘
- 会话恢复
- session memory
- auto memory
- agent memory
- team memory
- compact 辅助

这层是项目区别于普通 CLI 助手的关键。

### 4.6 MCP / Plugin / Remote / Swarm 扩展层

核心文件：

- [`src/services/mcp/client.ts`](../src/services/mcp/client.ts)
- [`src/utils/plugins/*`](../src/utils/plugins)
- [`src/bridge/*`](../src/bridge)
- [`src/utils/swarm/*`](../src/utils/swarm)

职责：

- 接入 MCP server
- 装配插件能力
- 远程桥接与控制
- 多 agent / teammate 协作

这层使项目具备平台化特征。

## 5. 典型主链路

典型默认 REPL 流程可概括为：

```text
entrypoints/cli.tsx
  -> main.tsx
  -> init.ts + setup.ts
  -> commands/tools/permissions/agents 初始化
  -> replLauncher.tsx
  -> screens/REPL.tsx
  -> query.ts
  -> services/api/claude.ts
  -> toolOrchestration.ts
  -> sessionStorage / memory / hooks / compact
```

这条链路非常完整，说明项目不是“把 API 包一层终端皮肤”，而是做了真正的本地 agent runtime。

如果再把“控制面”和“执行面”的边界画清楚，可以得到下面这个视图：

```text
控制面
  - commands.ts
  - REPL / PromptInput / Messages
  - Settings / MCP / Tasks / Teams / Hooks

执行面
  - query.ts / QueryEngine.ts
  - Tool / Permission
  - Transcript / Memory / Compact
  - MCP / Remote / Plugin / Swarm

关系
  commands.ts
    -> REPL / PromptInput / Messages
    -> query.ts / QueryEngine.ts

  Settings / MCP / Tasks / Teams / Hooks
    -> REPL / PromptInput / Messages
    -> 间接影响执行面状态

  query.ts / QueryEngine.ts
    -> Tool / Permission
    -> Transcript / Memory / Compact
    -> MCP / Remote / Plugin / Swarm
```

## 6. 本章小结

从架构上看，这个项目有三个明确特征：

1. 它是多入口系统，不只有一种运行方式。
2. 它把 UI、执行内核、工具层、memory 层、扩展层清晰拆开了。
3. 它的真正核心不是聊天，而是“围绕代码工作流的 agent 执行平台”。
