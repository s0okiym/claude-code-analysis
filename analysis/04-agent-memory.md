# 第四章：Agent Memory 机制是怎么做的

[返回总目录](../README.md)

## 1. 本章导读

这一章专门回答“memory 机制怎么做的”。这里的结论非常明确：

这个项目没有把 memory 做成一个单一数据库，而是做成了一套多层次、文件化、可检索、可同步的记忆系统。

主要包括：

1. Auto Memory
2. Session Memory
3. Agent Memory
4. Team Memory

此外还配套了：

- relevant memory recall
- transcript
- compact
- snapshot

## 2. Auto Memory

相关实现：

- [`src/memdir/memdir.ts`](../src/memdir/memdir.ts)
- [`src/memdir/paths.ts`](../src/memdir/paths.ts)
- [`src/services/extractMemories/extractMemories.ts`](../src/services/extractMemories/extractMemories.ts)

### 2.1 基本形态

Auto Memory 是文件系统 memory，而不是隐藏数据库。

目录结构核心特征：

- 目录内有 `MEMORY.md` 作为索引入口
- 每条 memory 建议单独保存为 markdown 文件
- `MEMORY.md` 只作为入口索引，不承载全部正文

### 2.2 触发方式

Auto Memory 的提取并不是用户每次手动写，而是：

- 在主 query 结束后
- 由后台 forked agent
- 从当前 transcript 中提取 durable memories

### 2.3 类型体系

代码里明确给 memory 做了类型分类，典型包括：

- user
- feedback
- project
- reference

这说明 memory 不是简单“抓一句记一句”，而是带有结构化治理意识。

## 3. Relevant Memory Recall

相关实现：

- [`src/memdir/findRelevantMemories.ts`](../src/memdir/findRelevantMemories.ts)
- [`src/utils/attachments.ts`](../src/utils/attachments.ts)

这部分是 memory 机制非常成熟的一点。

系统不是每轮把全部 memory 都塞进 prompt，而是：

1. 扫描 memory 文件头信息
2. 通过 side query 判断哪些 memory 与当前问题相关
3. 只选择少量 memory 注入本轮上下文

这相当于一种轻量级检索式记忆机制，避免了“记忆越多越污染 prompt”。

## 4. Session Memory

相关实现：

- [`src/services/SessionMemory/sessionMemory.ts`](../src/services/SessionMemory/sessionMemory.ts)
- [`src/services/SessionMemory/sessionMemoryUtils.ts`](../src/services/SessionMemory/sessionMemoryUtils.ts)
- [`src/services/compact/sessionMemoryCompact.ts`](../src/services/compact/sessionMemoryCompact.ts)

### 4.1 定位

Session Memory 不是跨会话人格记忆，而是“当前会话摘要文件”。

它的目标是：

- 在长会话中沉淀当前状态
- 为 compact 提供高质量摘要源

### 4.2 运行方式

它通过 post-sampling hook 触发：

- 达到 token / tool call 阈值后
- 启动 forked subagent
- 只允许它编辑指定 session memory 文件

这说明 Session Memory 是一种受限、单文件、后台式摘要代理。

### 4.3 与 compact 的关系

当 session memory 足够有效时，compact 可以优先使用 session memory 结果，而不是每次重新对整段长历史做摘要。

这大幅提升了长会话稳定性。

## 5. Agent Memory

相关实现：

- [`src/tools/AgentTool/agentMemory.ts`](../src/tools/AgentTool/agentMemory.ts)
- [`src/tools/AgentTool/loadAgentsDir.ts`](../src/tools/AgentTool/loadAgentsDir.ts)
- [`src/tools/AgentTool/agentMemorySnapshot.ts`](../src/tools/AgentTool/agentMemorySnapshot.ts)

### 5.1 三种 scope

Agent Memory 支持三种 scope：

- `user`
- `project`
- `local`

这非常关键，因为它把“记忆作用域”做成了系统级概念。

含义分别是：

- `user`：跨项目长期通用
- `project`：项目内共享
- `local`：当前机器/当前工作区使用

### 5.2 注入方式

当 agent 定义里声明了 `memory`，系统会在 agent system prompt 后自动拼接 memory prompt。

对应代码见：

- [`src/tools/AgentTool/loadAgentsDir.ts`](../src/tools/AgentTool/loadAgentsDir.ts)

这意味着 agent 的“角色人格”不只来自静态 prompt，也来自持久记忆目录。

### 5.3 Snapshot 机制

Agent Memory 还支持 snapshot：

- 检查项目是否有更高版本 snapshot
- 首次使用时可从 snapshot 初始化本地 memory
- 更新时可提示替换或同步

这说明 agent 记忆被设计成可分发、可演进的角色资产。

## 6. Team Memory

相关实现：

- [`src/services/teamMemorySync/index.ts`](../src/services/teamMemorySync/index.ts)
- [`src/services/teamMemorySync/watcher.ts`](../src/services/teamMemorySync/watcher.ts)
- [`src/memdir/teamMemPaths.ts`](../src/memdir/teamMemPaths.ts)

### 6.1 定位

Team Memory 是 auto memory 下的共享子空间，用于 repo 级团队知识同步。

### 6.2 机制

它包含完整同步链路：

- pull
- push
- watcher
- checksum / optimistic locking
- path validation
- secret scanning

也就是说，team memory 不是“共享文件夹”这么简单，而是一套受控同步系统。

### 6.3 实际意义

这让项目从“个人 agent 记忆”扩展到了“团队知识基础设施”。

## 7. 与 agent 运行时的耦合

相关实现：

- [`src/tools/AgentTool/runAgent.ts`](../src/tools/AgentTool/runAgent.ts)

memory 机制不是独立外挂，而是和 agent 生命周期耦合：

- agent 可以继承上下文
- agent 可以拥有独立 transcript
- agent 可以有独立 permission mode
- agent 可以注入独立 memory prompt
- agent 可以绑定独立 MCP servers

所以这里的 memory，本质上是“agent runtime 的一部分”。

## 8. 整体评价

这套 memory 设计有两个显著优点：

1. 高透明度：文件化、可读、可编辑
2. 高工程性：有作用域、检索、同步、compact 协同、snapshot

它也有两个明显代价：

1. 长期沉淀用户画像与项目知识
2. 如果治理不严格，容易让旧偏好或错误知识持续影响后续行为

## 9. 本章小结

如果只用一句话总结本项目的 memory 机制：

它不是“聊天历史缓存”，而是一套围绕 agent 长期协作设计的、文件化的、多层记忆系统。
