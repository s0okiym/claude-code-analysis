# src 源码分析总览

## 总述

本目录中的分析文档基于 `src/` 源码静态阅读整理，目标是从“总分总”的方式回答以下问题：

1. 这个项目的软件架构是什么，程序从哪里启动。
2. 从用户视角看，项目收集了哪些信息，并如何使用这些信息。
3. 用户若希望规避或降低信息收集，应如何做。
4. agent memory 机制是怎么实现的。
5. 该项目自身的程序架构亮点是什么。
6. 除上述要求外，还补充了权限、compact、MCP、remote、swarm 等额外发现。
7. 对 `src/components/` 中各组件族、关键子组件以及叶子组件子函数进行专项拆解。
8. 该项目与 `Codex`、`Gemini CLI`、`Aider`、`Cursor` 等同类产品相比，有哪些差异。
9. 相关结论分别对应哪些代码证据和外部公开资料。

总判断先放在这里：

- 该项目不是简单命令行聊天工具，而是一套本地代码 agent 平台。
- 它最突出的特征是：统一的执行内核、分层的 memory 系统、平台化扩展能力。
- 从用户视角，真正的风险重点不是单一 telemetry，而是“模型上下文 + 本地持久化 + 外部同步/远程能力”的组合。

## 一图总览

```text
+---------------------------+
| CLI / 多入口              |
| entrypoints/cli.tsx       |
| main.tsx                  |
+---------------------------+
            |
            v
+---------------------------+
| 初始化与运行环境          |
| init.ts / setup.ts        |
+---------------------------+
      |                |
      v                v
+----------------+   +---------------------------+
| 命令与控制面   |   | TUI / REPL 工作台         |
| commands.ts    |-->| App / REPL / Messages     |
| PromptInput... |   | / PromptInput             |
+----------------+   +---------------------------+
                             |
                             v
                  +---------------------------+
                  | Query / Agent 执行内核    |
                  | query.ts / QueryEngine.ts |
                  +---------------------------+
                    |           |           |
                    v           v           v
          +---------------+ +--------------------+ +----------------------+
          | Tool/Perm     | | Transcript/Memory  | | 平台扩展层           |
          | Tool.ts       | | sessionStorage     | | MCP/Plugin/Remote/   |
          | orchestration | | memdir/SessionMem  | | Swarm                |
          +---------------+ +--------------------+ +----------------------+
                    \______________   |   ______________/
                                   \  |  /
                                    \ v /
                              回流到执行内核
```

## 分章目录

### 第一部分：总体架构

- [第一章：软件架构与程序入口](./analysis/01-architecture-overview.md)

### 第二部分：用户信息与隐私

- [第二章：从用户角度看，项目收集了哪些信息，以及如何使用](./analysis/02-user-data-and-usage.md)
- [第三章：从用户角度看，如何规避或降低信息收集](./analysis/03-privacy-avoidance.md)

### 第三部分：核心机制

- [第四章：Agent Memory 机制是怎么做的](./analysis/04-agent-memory.md)

### 第四部分：程序架构及亮点

- [第五章：程序架构及亮点](./analysis/05-differentiators-and-comparison.md)

### 第五部分：扩展分析

- [第六章：额外探索与补充发现](./analysis/06-extra-findings.md)

### 第六部分：组件体系详解

- [组件详解（一）：组件总览、分层与依赖主干](./analysis/components/01-component-architecture-overview.md)
- [组件详解（二）：核心交互组件与消息/输入主链路](./analysis/components/02-core-interaction-components.md)
- [组件详解（三）：平台能力组件与控制面实现](./analysis/components/03-platform-components.md)
- [组件详解（四）：组件索引、长尾组件与目录映射](./analysis/components/04-component-index.md)
- [组件详解（五）：核心组件函数级实现拆解](./analysis/components/05-function-level-core-walkthrough.md)
- [组件详解（六）：平台控制面函数级实现拆解](./analysis/components/06-function-level-platform-walkthrough.md)
- [组件详解（七）：叶子组件与子函数实现拆解](./analysis/components/07-function-level-leaf-walkthrough.md)

### 第七部分：同类产品对比

- [第八章：同类产品对比](./analysis/08-competitive-comparison.md)
- [附录A：外部对比资料](./analysis/08-reference-comparison-sources.md)

### 第八部分：证据与资料

- [第七章：代码证据索引](./analysis/07-code-evidence-index.md)

### 第九部分：总结

- [第九章：总结结论](./analysis/09-final-summary.md)
