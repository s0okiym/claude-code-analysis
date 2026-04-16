# 子领域分析：MCP 系统

## 1. 功能概述
MCP（Model Context Protocol）是 Anthropic 推出的开放协议，用于连接外部工具服务器，扩展 Claude 的能力。
## 2. 核心组件
### 2.1 CP 连接管理器
```python
class MCPConnectionManager:
    """管理所有 CP 服务器连接"""
    connections: dict[str, MCPServer]
    
    async def connect(self, config: ServerConfig):
        transport = self.create_transport(config)
        await self.initialize(transport)
    
    def get_tools(self) -> list[Tool]:
        """获取所有 CP 工具"""
        return [
            tool for conn in self.connections.values()
            for tool in conn.tools
        ]
```

### 2.2 传输层
```python
class StdioTransport:
    """本地进程通信"""
    process: subprocess.Popen
    
    async def send(self, message: JSONRPCMessage):
        self.process.stdin.write(message.serialize())
    
    async def receive(self) -> JSONRPCMessage:
        return JSONRPCMessage.parse(
            await self.process.stdout.readline()
        )

class SSETransport:
    """HTTP SSE 通信"""
    session: aiohttp.ClientSession
    
    async def connect(self, url: str):
        self.response = await self.session.get(url)
```

### 2.3 工具注册
```python
def register_mcp_tools(
    manager: MCPConnectionManager,
    permission_ctx: PermissionContext
) -> list[Tool]:
    """注册 CP 工具到系统"""
    tools = []
    for conn in manager.connections:
        for mcp_tool in conn.tools:
            tool = wrap_mcp_tool(mcp_tool, permission_ctx)
            tools.append(tool)
    return tools
```
## 3. CP 架构
```
┌─────────────────────────────────────────┐
│         Claude Code                   │
│  ┌─────────────────────────────┐    │
│  │ MCPConnectionManager        │    │
│  │ - connections: dict       │    │
│  │ - get_tools()           │    │
│  └──────────┬────────────┘    │
│               │                   │
│  ┌────────────┼────────────────┐ │
│  │            │                │ │
│  ▼            ▼                ▼ │
│  Stdio      SSE            StreamableHTTP
│  Transport  Transport       Transport
│  (本地)    (远程)       (HTTP)
│
│  ┌────────┐
│  │ MCP    │
│  │ Server │
│  │ 外部工具 │
│  └─────────┘
└─────────────────────────────────────────┘
```
## 4. 关键设计
### 4.1 为什么需要 CP?
| 特性 | 内置工具 | CP |
|------|--------|------|
| 扩展性 | 需修改代码 | 配置即可 |
| 生态 | 有限 | 丰富 |
| 安全 | 完全控制 | 需权限检查 |
## 5. 评价
| 维度 | 评分 | 说明 |
|--------|------|--------|
| 扩展性 | ★★★★★ | 配置即可添加工具 |
| 安全 | ★★★★☆ | 需权限检查 |
| 复杂性 | ★★★★☆ | 协议复杂 |
| 生态 | ★★★★★ | 丰富工具 |
| 稳定性 | ★★★★☆ | 依赖外部 |
## 6. 关键代码
```python
# CP 配置
mcp_config = {
    "mcpServers": {
        "filesystem": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-filesystem"],
            "description": "文件系统工具"
        }
    }
}
```
```
我看到代码有错乱，需要修复。让我修复并继续：<|tool_calls_section_begin|>