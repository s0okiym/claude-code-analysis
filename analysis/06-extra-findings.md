# 第六章：额外探索与补充发现

[返回总目录](../README.md)

## 1. 本章导读

这一章记录不在原始要求中、但从架构分析角度值得补充的发现。

## 2. Trust 边界前后是分阶段初始化的

相关实现：

- [`src/entrypoints/init.ts`](../src/entrypoints/init.ts)
- [`src/interactiveHelpers.tsx`](../src/interactiveHelpers.tsx)

项目明确区分了 trust 建立前后：

- 先只应用 safe env vars
- trust 通过后再应用完整环境变量
- 完整 telemetry 也在 trust 后再启用

这说明作者非常明确地意识到：

- 配置文件
- 环境变量
- 外部 includes

本身都可能是攻击面。

## 3. 多 Agent 后端不只一种

相关实现：

- [`src/utils/swarm/spawnInProcess.ts`](../src/utils/swarm/spawnInProcess.ts)
- [`src/utils/swarm/backends/registry.ts`](../src/utils/swarm/backends/registry.ts)

项目支持的 teammate backend 不止一种：

- in-process
- tmux
- iTerm2 pane

这说明 swarm 不是附属功能，而是认真设计过的运行后端抽象。

## 4. 工具并发模型做得很克制

相关实现：

- [`src/services/tools/toolOrchestration.ts`](../src/services/tools/toolOrchestration.ts)

策略是：

- 并发安全工具才并发
- 非安全工具串行

这比“全并发”更保守，但更符合真实工程环境下文件写入、状态更新和工具副作用的约束。

## 5. 长会话治理不是补丁，而是长期工程主题

相关实现：

- [`src/services/compact`](../src/services/compact)
- [`src/services/SessionMemory`](../src/services/SessionMemory)

从模块规模看，作者把下面这些都当成核心问题：

- context 窗口退化
- 摘要质量
- 会话恢复
- compact 后清理
- session memory 替代普通 compact

这和很多只在出现上下文超长时“临时补一个摘要”的实现很不一样。

## 6. MCP 在这里不是单点功能，而是平台扩展接口

相关实现：

- [`src/services/mcp/client.ts`](../src/services/mcp/client.ts)
- [`src/services/mcp/auth.ts`](../src/services/mcp/auth.ts)
- [`src/entrypoints/mcp.ts`](../src/entrypoints/mcp.ts)

项目既能：

- 作为 MCP client 消费外部能力
- 作为 MCP server 暴露自己的工具

这使得它不仅是产品本身，也可以成为其他 agent/工具链的一部分。

## 7. 隐私风险最大的点其实是“能力耦合”

单独看每个模块都不算夸张：

- transcript
- memory
- telemetry
- team sync
- remote

但耦合起来后会形成一个长期的用户工作画像系统：

- transcript 提供原始历史
- extract memories 提炼长期偏好
- relevant memory 反向注入后续 prompt
- team memory 扩展到团队知识同步
- remote/bridge 扩大会话边界

这不是某个模块的风险，而是系统性特征。

## 8. 本章小结

除用户明确提出的分析点之外，我认为这个项目额外最值得关注的三件事是：

1. trust 边界处理得比较成熟
2. swarm / remote / MCP 让它具备平台属性
3. 长期记忆与长期会话能力叠加后，会形成比普通 CLI 助手更强的“长期协作建模能力”
