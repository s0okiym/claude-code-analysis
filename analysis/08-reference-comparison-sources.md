# 第八章：外部对比资料

[返回总目录](../README.md)

## 1. 本章导读

这一章只记录对比分析使用的外部公开资料，避免把外部资料和源码证据混在一起。

## 2. Cursor

- Cursor Background Agents: <https://docs.cursor.com/en/background-agents>
- Cursor MCP: <https://docs.cursor.com/en/context/mcp>
- Cursor CLI MCP: <https://docs.cursor.com/cli/mcp>

用于支撑的判断：

- Cursor 强调 background agents
- Cursor 强调 IDE 协同和远程执行环境
- Cursor 有 MCP 扩展能力

## 3. Aider

- Aider 首页: <https://aider.chat/>
- Aider 文档总览: <https://aider.chat/docs/>

用于支撑的判断：

- Aider 强调终端 pair programming
- Aider 强调 repo map、git、lint/test 闭环
- Aider 更轻量，平台化程度低于当前项目

## 4. Gemini CLI

- Gemini CLI 官方仓库 README: <https://github.com/google-gemini/gemini-cli>

用于支撑的判断：

- Gemini CLI 是公开开源 CLI agent 的高标准基线
- 其公开能力包括 built-in tools、MCP、checkpointing、sandboxing、trusted folders、telemetry

## 5. 使用说明

这些资料只用于做“同类产品公开能力边界对比”，不是用于证明本项目源码细节。源码细节仍以本仓库中的 [`analysis/07-code-evidence-index.md`](./07-code-evidence-index.md) 和对应代码文件为准。

## 6. 本章小结

对比结论的证据来源已经单独分离，便于后续更新或替换同类产品参照物。
