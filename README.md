# src 源码分析总览

## 总述

本目录中的分析文档基于 `src/` 源码静态阅读整理，目标是从“总分总”的方式回答以下问题：

1. 这个项目的软件架构是什么，程序从哪里启动。
2. 从用户视角看，项目收集了哪些信息，并如何使用这些信息。
3. 用户若希望规避或降低信息收集，应如何做。
4. agent memory 机制是怎么实现的。
5. 该项目与同类产品相比，架构亮点和差异在哪里。
6. 除上述要求外，还补充了权限、compact、MCP、remote、swarm 等额外发现。

总判断先放在这里：

- 该项目不是简单命令行聊天工具，而是一套本地代码 agent 平台。
- 它最突出的特征是：统一的执行内核、分层的 memory 系统、平台化扩展能力。
- 从用户视角，真正的风险重点不是单一 telemetry，而是“模型上下文 + 本地持久化 + 外部同步/远程能力”的组合。

## 分章目录

### 第一部分：总体架构

- [第一章：软件架构与程序入口](./analysis/01-architecture-overview.md)

### 第二部分：用户信息与隐私

- [第二章：从用户角度看，项目收集了哪些信息，以及如何使用](./analysis/02-user-data-and-usage.md)
- [第三章：从用户角度看，如何规避或降低信息收集](./analysis/03-privacy-avoidance.md)

### 第三部分：核心机制

- [第四章：Agent Memory 机制是怎么做的](./analysis/04-agent-memory.md)

### 第四部分：亮点与对比

- [第五章：程序架构的亮点，以及与同类产品的不同](./analysis/05-differentiators-and-comparison.md)

### 第五部分：扩展分析

- [第六章：额外探索与补充发现](./analysis/06-extra-findings.md)

### 第六部分：证据与资料

- [第七章：代码证据索引](./analysis/07-code-evidence-index.md)
- [第八章：外部对比资料](./analysis/08-reference-comparison-sources.md)

### 第七部分：总结

- [第九章：总结结论](./analysis/09-final-summary.md)

## 阅读建议

如果只想快速把握结论，建议按这个顺序阅读：

1. [第一章：软件架构与程序入口](./analysis/01-architecture-overview.md)
2. [第二章：从用户角度看，项目收集了哪些信息，以及如何使用](./analysis/02-user-data-and-usage.md)
3. [第四章：Agent Memory 机制是怎么做的](./analysis/04-agent-memory.md)
4. [第五章：程序架构的亮点，以及与同类产品的不同](./analysis/05-differentiators-and-comparison.md)
5. [第九章：总结结论](./analysis/09-final-summary.md)

如果要做源码复核，建议直接看：

1. [第七章：代码证据索引](./analysis/07-code-evidence-index.md)
2. 再跳转到对应源码文件

## 总结

整个分析的最终结论是：

- 这是一个具备平台属性的本地 agent runtime，而不是一次性问答工具。
- 它的 memory、transcript、compact、permission、MCP、remote、swarm 被统一进了一套执行内核。
- 这带来了很强的工程能力，也意味着用户和组织需要更严格地管理信息输入、持久化策略与远程能力开关。
