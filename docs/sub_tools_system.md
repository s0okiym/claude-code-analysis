# 子领域分析：工具元数据管理系统

## 1. 功能概述

Tools System 负责管理已移植工具的元数据，与 Commands System 结构相同但语义不同。工具代表可被 Agent 调用的功能单元，而命令代表用户可直接调用的 CLI 命令。

## 2. 核心组件

### 2.1 工具元数据结构

复用 `PortingModule` 数据类：

```python
@dataclass(frozen=True)
class PortingModule:
    name: str            # 工具名，如 "port_manifest"
    responsibility: str  # 功能描述
    source_hint: str     # 源文件提示
    status: str = 'planned'
```

### 2.2 工具注册表

```python
PORTED_TOOLS = (
    PortingModule(
        'port_manifest',
        'Inspect the active Python source tree and summarize the current rewrite surface',
        'src/port_manifest.py',
        'implemented'
    ),
    PortingModule(
        'backlog_models',
        'Represent subsystem and backlog metadata as Python dataclasses',
        'src/models.py',
        'implemented'
    ),
    PortingModule(
        'query_engine',
        'Coordinate Python-facing rewrite summaries and reporting',
        'src/query_engine.py',
        'implemented'
    ),
)
```

### 2.3 清单构建函数

```python
def build_tool_backlog() -> PortingBacklog:
    """构建工具移植待办清单"""
    return PortingBacklog(title='Tool surface', modules=list(PORTED_TOOLS))
```

## 3. 命令 vs 工具的语义区分

### 3.1 概念对比

| 维度 | 命令 (Command) | 工具 (Tool) |
|------|----------------|-------------|
| **调用者** | 用户（CLI） | Agent（AI） |
| **入口** | `python3 -m src.main [cmd]` | Agent 通过 tool_use 调用 |
| **交互** | 直接 | 间接（通过 Agent） |
| **示例** | `summary`, `manifest` | `port_manifest`, `query_engine` |
| **粒度** | 粗粒度（完整功能） | 细粒度（原子操作） |

### 3.2 当前重叠

当前实现中，命令和工具有大量重叠：

```
命令: main, summary, subsystems
工具: port_manifest, backlog_models, query_engine

重叠分析:
- summary 命令 → 调用 query_engine 工具
- subsystems 命令 → 调用 port_manifest 工具
- main 命令 → CLI 入口，不直接对应工具
```

**设计说明**:
- 当前阶段，命令和工具一一对应
- 未来工具可能更细粒度（如 `read_file`, `write_file`）
- 命令可能组合多个工具

## 4. 流程分析

### 4.1 工具调用流程（概念）

```
Agent 循环（未来实现）
  │
  ├─ 1. LLM 决定调用工具
  │     └─ tool_use: port_manifest
  │
  ├─ 2. 系统查找工具实现
  │     └─ PORTED_TOOLS 中查找
  │
  ├─ 3. 执行工具
  │     └─ 调用 src/port_manifest.py 功能
  │
  └─ 4. 返回结果给 Agent
        └─ tool_result
```

### 4.2 当前状态

当前工具系统仅作为**元数据注册表**，实际功能由对应模块直接实现。这是移植早期阶段的简化设计。

## 5. 设计决策

### 5.1 为什么复用 PortingModule?

```python
# 命令和工具使用相同的数据结构
PORTED_COMMANDS: tuple[PortingModule, ...]
PORTED_TOOLS: tuple[PortingModule, ...]
```

**理由**:
- 命令和工具具有相同的属性需求
- 复用减少代码重复
- 统一处理逻辑（`summary_lines()`）

**未来可能的分化**:

```python
@dataclass(frozen=True)
class Tool:
    name: str
    description: str
    source_file: Path
    parameters: dict  # 新增：工具参数定义
    returns: str      # 新增：返回值描述
    examples: list    # 新增：使用示例
```

### 5.2 为什么工具数量少于命令?

当前: 3 个工具 vs 3 个命令

**原因**:
- 当前阶段功能简单，命令和工具一一对应
- 未来工具数量可能超过命令（细粒度拆分）

**原始 Claude Code 对比**:
- 命令：数十个（`session`, `mcp`, `config` 等子命令）
- 工具：30+ 个（`BashTool`, `FileReadTool`, `GrepTool` 等）

### 5.3 为什么工具名使用下划线?

```python
# 工具名
'port_manifest'
'backlog_models'
'query_engine'

# 对比命令名
'main'
'summary'
'subsystems'
```

**命名约定**:
- 工具：snake_case（Python 风格，函数名）
- 命令：kebab-case 或单个词（CLI 风格）

**理由**:
- 工具对应 Python 函数/模块
- 命令对应 CLI 子命令

## 6. 扩展性分析

### 6.1 添加工具参数定义

```python
@dataclass(frozen=True)
class ToolParameter:
    name: str
    type: str  # 'string', 'number', 'boolean', 'path'
    required: bool
    description: str
    default: Any = None

@dataclass(frozen=True)
class Tool:
    name: str
    responsibility: str
    source_hint: str
    parameters: tuple[ToolParameter, ...] = ()
    status: str = 'planned'

PORTED_TOOLS = (
    Tool(
        'port_manifest',
        'Inspect the active Python source tree',
        'src/port_manifest.py',
        parameters=(
            ToolParameter('src_root', 'path', False, 'Source root path'),
        ),
        'implemented'
    ),
)
```

### 6.2 支持工具分类

```python
@dataclass(frozen=True)
class ToolCategory:
    name: str
    description: str
    tools: tuple[PortingModule, ...]

FILE_TOOLS = ToolCategory(
    'file',
    'File system operations',
    (...)
)

SEARCH_TOOLS = ToolCategory(
    'search',
    'Search and query operations',
    (...)
)
```

### 6.3 工具注册装饰器（未来）

```python
# 未来可能的支持
from src.tool_registry import tool

@tool(
    name='port_manifest',
    description='Inspect the active Python source tree'
)
def port_manifest_tool(src_root: Path | None = None) -> PortManifest:
    return build_port_manifest(src_root)

# 自动注册到工具列表
```

## 7. 存在的问题

### 7.1 当前限制

1. **无参数定义**: 工具参数未在元数据中描述
2. **无返回值描述**: 调用者不知道返回什么类型
3. **无示例**: 缺少使用示例
4. **与命令重叠**: 当前命令和工具区分不清晰

### 7.2 与 Agent 系统的集成缺失

```python
# 当前：纯元数据
PORTED_TOOLS = (...)

# 未来：需要支持 Agent 调用
class ToolExecutor:
    def execute(self, tool_name: str, params: dict) -> Any:
        tool = self._find_tool(tool_name)
        # 验证参数
        # 执行工具
        # 返回结果
```

### 7.3 改进建议

1. 添加完整的工具定义：

```python
@dataclass(frozen=True)
class ToolDefinition:
    name: str
    description: str
    source_file: Path
    parameters: dict[str, ParameterSpec]
    returns: ReturnSpec
    examples: list[Example]
    status: str
```

2. 支持工具发现：

```python
def discover_tools(src_root: Path) -> list[ToolDefinition]:
    """自动从源码发现工具"""
    ...
```

3. 工具与命令的明确分离：

```python
# 命令：用户直接调用
# 工具：Agent 调用
# 映射关系：命令可能调用多个工具
```

## 8. 测试策略

### 8.1 单元测试

```python
def test_build_tool_backlog():
    """测试工具清单构建"""
    backlog = build_tool_backlog()
    
    assert backlog.title == 'Tool surface'
    assert len(backlog.modules) == 3
    
    # 验证工具名称
    names = [m.name for m in backlog.modules]
    assert 'port_manifest' in names
    assert 'query_engine' in names

def test_tool_names_snake_case():
    """验证工具名使用 snake_case"""
    for tool in PORTED_TOOLS:
        assert '_' in tool.name or tool.name.islower(), \
            f'Tool name should be snake_case: {tool.name}'
```

### 8.2 工具与命令对比测试

```python
def test_tools_vs_commands():
    """对比工具和命令的差异"""
    command_names = {c.name for c in PORTED_COMMANDS}
    tool_names = {t.name for t in PORTED_TOOLS}
    
    # 命令和工具应有不同命名风格
    for cmd in command_names:
        assert '-' not in cmd and '_' not in cmd, \
            f'Command should be simple word: {cmd}'
    
    for tool in tool_names:
        assert '_' in tool, \
            f'Tool should be snake_case: {tool}'
```

## 9. 与原始 Claude Code 对比

| 特性 | 原始 (TypeScript) | 本实现 (Python) |
|------|-------------------|-----------------|
| 工具数量 | 30+ | 3 |
| 工具粒度 | 细粒度（原子操作） | 粗粒度（模块级） |
| 参数定义 | Zod schema | 无 |
| 权限检查 | 有 | 无 |
| 并发执行 | 支持 | 无 |
| 结果处理 | 复杂（持久化、压缩） | 无 |

原始 Claude Code 的工具系统是其核心功能，本实现仅为元数据占位。

## 10. 评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 完整性 | ★★☆☆☆ | 仅为元数据，无实际工具框架 |
| 可扩展性 | ★★★☆☆ | 基础结构，需大量扩展 |
| 与命令的区分 | ★★☆☆☆ | 当前重叠较多 |
| 类型安全 | ★★★★☆ | 复用 PortingModule |
| 文档化 | ★★★☆☆ | 有描述，但缺少参数/返回值 |

## 11. 关键代码片段

### 11.1 完整实现

```python
from __future__ import annotations
from .models import PortingBacklog, PortingModule

PORTED_TOOLS = (
    PortingModule('port_manifest', 'Inspect the active Python source tree...', 
                  'src/port_manifest.py', 'implemented'),
    PortingModule('backlog_models', 'Represent subsystem and backlog metadata...', 
                  'src/models.py', 'implemented'),
    PortingModule('query_engine', 'Coordinate Python-facing rewrite summaries...', 
                  'src/query_engine.py', 'implemented'),
)


def build_tool_backlog() -> PortingBacklog:
    return PortingBacklog(title='Tool surface', modules=list(PORTED_TOOLS))
```

### 11.2 与命令系统对比

```python
# commands.py
PORTED_COMMANDS = (
    PortingModule('main', 'Expose a Python CLI...', 'src/main.py', 'implemented'),
    PortingModule('summary', 'Render a Markdown overview...', 'src/query_engine.py', 'implemented'),
    PortingModule('sub