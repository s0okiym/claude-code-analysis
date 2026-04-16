# 子领域分析：Swarm 多 Agent 系统

## 1. 功能概述

Swarm 系统支持多 Agent 协作，允许创建多个工作 Agent 并行处理任务，通过消息传递协调。

## 2. 核心组件

### 2.1 团队创建

```python
class TeamCreateTool:
    """创建多 Agent 团队"""
    
    def run(self, args: TeamCreateInput) -> Team:
        team = Team(
            name=args.name,
            agents=[
                self.create_agent(role)
                for role in args.roles
            ]
        )
        return team
```

### 2.2 Agent 类型

```python
class InProcessTeammateTask:
    """同进程 Agent"""
    context: AsyncLocalStorage  # 上下文隔离
    
    async def execute(self, task: Task):
        async with self.context:
            return await self.run_task(task)

class LocalAgentTask:
    """本地子进程 Agent"""
    process: subprocess.Popen
    
    async def execute(self, task: Task):
        self.process = spawn_agent_process()
        return await self.communicate(task)
```

### 2.3 消息传递

```python
class SendMessageTool:
    """Agent 间通信"""
    
    def run(self, args: SendMessageInput) -> MessageResult:
        target = self.find_agent(args.to)
        target.inbox.put(Message(
            from_=args.from_,
            content=args.content
        ))
        return MessageResult(status='delivered')
```

## 3. Swarm 架构

```
┌─────────────────────────────────────────┐
│           Leader Agent                  │
│  (主 Agent，协调团队)                    │
└──────────────┬──────────────────────────┘
               │ TeamCreateTool
               ▼
┌─────────────────────────────────────────┐
│              Team                       │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ Agent 1 │ │ Agent 2 │ │ Agent 3 │   │
│  │(Worker) │ │(Worker) │ │(Worker) │   │
│  └────┬────┘ └────┬────┘ └────┬────┘   │
│       │           │           │         │
│       └───────────┼───────────┘         │
│                   │                     │
│              SendMessageTool            │
│              (消息传递)                  │
└─────────────────────────────────────────┘
```

## 4. 关键设计

### 4.1 后端选择策略

```python
def select_backend() -> Backend:
    """自动选择可用后端"""
    if InProcessBackend.is_available():
        return InProcessBackend()
    elif TmuxBackend.is_available():
        return TmuxBackend()
    elif ItermBackend.is_available():
        return ItermBackend()
    else:
        raise NoBackendAvailable()
```

| 后端 | 适用场景 | 特点 |
|------|----------|------|
| InProcess | 同进程 | 最快，共享内存 |
| Tmux | 独立终端 | 隔离，可见 |
| iTerm2 | macOS | 原生集成 |

## 5. 评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 并行性 | ★★★★★ | 多 Agent 并行 |
| 协调复杂度 | ★★★★☆ | 需要消息传递 |
| 资源占用 | ★★★☆☆ | 多进程/线程 |
| 调试难度 | ★★★☆☆ | 分布式调试难 |
| 扩展性 | ★★★★★ | 可动态添加 |

## 6. 关键代码

```python
# 创建团队
team = await agent.run_tool('team_create', {
    'name': 'refactoring-team',
    'roles': ['analyzer', 'modifier', 'tester']
})

# Agent 间通信
await agent.run_tool('send_message', {
    'to': 'analyzer',
    'content': 'Please analyze src/auth.py'
})
```
