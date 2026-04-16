# 子领域分析：命令元数据管理系统

## 1. 功能概述

Commands System 负责管理已移植命令的元数据，包括命令名称、职责描述、源文件位置和实现状态。这是一个轻量级的领域数据管理模块。

## 2. 核心组件

### 2.1 命令元数据结构

```python
@dataclass(frozen=True)
class PortingModule:
    name: str            # 命令名，如 "main"
    responsibility: str  # 职责描述
    source_hint: str     # 源文件提示，如 "src/main.py"
    status: str = 'planned'  # 实现状态
```

### 2.2 命令注册表

```python
PORTED_COMMANDS = (
    PortingModule(
        'main', 
        'Expose a Python CLI for manifest and backlog reporting',
        'src/main.py',
        'implemented'
    ),
    PortingModule(
        'summary',
        'Render a Markdown overview of the current porting workspace',
        'src/query_engine.py',
        'implemented'
    ),
    PortingModule(
        'subsystems',
        'List the current Python modules participating in the rewrite',
        'src/port_manifest.py',
        'implemented'
    ),
)
```

### 2.3 清单构建函数

```python
def build_command_backlog() -> PortingBacklog:
    """构建命令移植待办清单"""
    return PortingBacklog(
        title='Command surface',
        modules=list(PORTED_COMMANDS)
    )
```

## 3. 流程分析

### 3.1 数据流

```
commands.py
  │
  ├─ 1. 定义 PORTED_COMMANDS 元组
  │     └─ 硬编码已移植命令列表
  │
  ├─ 2. 提供 build_command_backlog()
  │     └─ 转换为 PortingBacklog 对象
  │
  └─ 3. 被 query_engine.py 调用
        └─ 生成 Markdown 输出
```

### 3.2 状态流转

```
命令生命周期:

planned → in-progress → implemented
   │          │            │
   │          │            └─ 已完成移植
   │          └─ 正在移植
   └─ 计划移植，尚未开始
```

## 4. 实现原理

### 4.1 元组作为常量集合

```python
PORTED_COMMANDS = (
    PortingModule(...),
    PortingModule(...),
)
```

**为什么选择元组?**

| 特性 | tuple | list | set |
|------|-------|------|-----|
| 不可变性 | ✓ | ✗ | ✗ |
| 有序性 | ✓ | ✓ | ✗ |
| 可哈希 | ✓ | ✗ | ✗ |
| 重复元素 | 允许 | 允许 | 自动去重 |

**理由**:
- 不可变性：命令列表定义后不应修改
- 有序性：保持定义顺序输出
- 语义清晰：表示只读集合

### 4.2 元组解包与展开

```python
sections = [
    f'{command_backlog.title}:',
    *command_backlog.summary_lines(),  # 展开操作符 *
]
```

**等价写法**:
```python
# 使用 extend
sections = [f'{command_backlog.title}:']
sections.extend(command_backlog.summary_lines())

# 使用列表拼接
sections = [f'{command_backlog.title}:'] + command_backlog.summary_lines()
```

### 4.3 延迟构建模式

```python
# 每次调用都新建对象
def build_command_backlog() -> PortingBacklog:
    return PortingBacklog(title='Command surface', modules=list(PORTED_COMMANDS))

# 对比：预构建（可能过早）
COMMAND_BACKLOG = PortingBacklog(title='Command surface', modules=list(PORTED_COMMANDS))
```

**选择理由**:
- 简单，无副作用
- 数据量小，构建开销可忽略
- 未来可支持动态过滤

## 5. 设计决策

### 5.1 为什么硬编码而非配置文件?

| 方案 | 优点 | 缺点 |
|------|------|------|
| 硬编码 (当前) | 类型安全，IDE 支持，版本控制 | 需修改代码添加命令 |
| JSON/YAML 配置 | 无需修改代码 | 无类型检查，需额外解析 |
| 数据库 | 动态更新 | 过度设计，增加复杂度 |

**选择理由**:
- 命令列表稳定，变动不频繁
- 类型安全，编译时检查
- 与代码一起版本控制

### 5.2 为什么使用 PortingModule 而非专用类?

```python
# 选项 1：复用 PortingModule（当前）
@dataclass(frozen=True)
class PortingModule:
    name: str
    responsibility: str
    source_hint: str
    status: str = 'planned'

# 选项 2：专用 Command 类
@dataclass(frozen=True)
class Command:
    name: str
    description: str
    source_file: Path
    implemented: bool = False
```

**选择理由**:
- 命令和工具具有相同结构（名称、描述、源文件、状态）
- 复用减少代码重复
- 统一处理逻辑（summary_lines 等）

### 5.3 为什么 status 使用字符串而非枚举?

```python
# 当前：字符串
status: str = 'planned'

# 替代：枚举
from enum import Enum

class Status(Enum):
    PLANNED = 'planned'
    IN_PROGRESS = 'in-progress'
    IMPLEMENTED = 'implemented'

status: Status = Status.PLANNED
```

**权衡**:

| 因素 | 字符串 | 枚举 |
|------|--------|------|
| 简洁性 | 高 | 中 |
| 类型安全 | 低 | 高 |
| 扩展性 | 高（任意字符串） | 中（需修改枚举） |
| 序列化 | 直接 | 需转换 |

**当前选择字符串的理由**:
- 简单，无需导入额外类型
- 输出直接可用（Markdown）
- 状态值可能来自外部（如 CI/CD）

**改进建议**: 未来可改为 Literal 类型：
```python
from typing import Literal

Status = Literal['planned', 'in-progress', 'implemented']
status: Status = 'planned'
```

## 6. 扩展性分析

### 6.1 添加新命令

```python
PORTED_COMMANDS = (
    ...,
    PortingModule(
        'config',
        'Manage configuration settings',
        'src/config.py',
        'planned'  # 或 'in-progress'
    ),
)
```

### 6.2 添加新字段

```python
@dataclass(frozen=True)
class PortingModule:
    name: str
    responsibility: str
    source_hint: str
    status: str = 'planned'
    priority: int = 0           # 新增：优先级
    dependencies: tuple = ()    # 新增：依赖的其他命令
    tags: tuple = ()            # 新增：标签
```

### 6.3 支持动态查询

```python
def build_command_backlog(
    status_filter: str | None = None,
    sort_by: str = 'name'
) -> PortingBacklog:
    modules = list(PORTED_COMMANDS)
    
    if status_filter:
        modules = [m for m in modules if m.status == status_filter]
    
    if sort_by == 'priority':
        modules.sort(key=lambda m: m.priority, reverse=True)
    
    return PortingBacklog(title='Command surface', modules=modules)
```

### 6.4 从配置文件加载

```python
import json

def load_commands_from_config(path: Path) -> tuple[PortingModule, ...]:
    """从 JSON 配置加载命令列表"""
    with open(path) as f:
        data = json.load(f)
    
    return tuple(
        PortingModule(**cmd) for cmd in data['commands']
    )

# 使用
COMMANDS_CONFIG = Path('commands.json')
if COMMANDS_CONFIG.exists():
    PORTED_COMMANDS = load_commands_from_config(COMMANDS_CONFIG)
```

## 7. 存在的问题

### 7.1 当前限制

1. **手动维护**: 添加命令需修改代码
2. **无验证**: 可能重复定义相同名称
3. **无关联**: 命令之间无依赖关系
4. **静态**: 不支持运行时动态注册

### 7.2 边界情况

```python
# 重复名称（无检测）
PORTED_COMMANDS = (
    PortingModule('main', ...),
    PortingModule('main', ...),  # 重复！
)

# 空描述（允许）
PortingModule('cmd', '', 'src/cmd.py')

# 无效状态（允许）
PortingModule('cmd', 'desc', 'src/cmd.py', 'invalid_status')
```

### 7.3 改进建议

1. 添加验证装饰器：

```python
def validate_commands(commands: tuple[PortingModule, ...]) -> None:
    names = [c.name for c in commands]
    if len(names) != len(set(names)):
        duplicates = [n for n in names if names.count(n) > 1]
        raise ValueError(f'Duplicate command names: {duplicates}')
    
    valid_statuses = {'planned', 'in-progress', 'implemented'}
    for cmd in commands:
        if cmd.status not in valid_statuses:
            raise ValueError(f'Invalid status for {cmd.name}: {cmd.status}')

PORTED_COMMANDS = validate_commands((...))
```

2. 支持命令分组：

```python
@dataclass(frozen=True)
class CommandGroup:
    name: str
    commands: tuple[PortingModule, ...]

CLI_COMMANDS = CommandGroup('cli', (...))
CONFIG_COMMANDS = CommandGroup('config', (...))
```

## 8. 测试策略

### 8.1 单元测试

```python
def test_build_command_backlog():
    """测试清单构建"""
    backlog = build_command_backlog()
    
    assert backlog.title == 'Command surface'
    assert len(backlog.modules) == 3
    
    # 验证每个模块
    for module in backlog.modules:
        assert module.name
        assert module.responsibility
        assert module.source_hint
        assert module.status in {'planned', 'in-progress', 'implemented'}

def test_ported_commands_unique_names():
    """测试命令名称唯一性"""
    names = [cmd.name for cmd in PORTED_COMMANDS]
    assert len(names) == len(set(names)), f'Duplicate names: {names}'
```

### 8.2 集成测试

```python
def test_command_backlog_in_summary():
    """测试命令清单在摘要中的呈现"""
    from src.query_engine import QueryEnginePort
    
    engine = QueryEnginePort.from_workspace()
    summary = engine.render_summary()
    
    assert 'Command surface:' in summary
    assert 'main [implemented]' in summary
    assert 'summary [implemented]' in summary
```

## 9. 与原始 Claude Code 对比

| 特性 | 原始 (TypeScript) | 本实现 (Python) |
|------|-------------------|-----------------|
| 命令定义 | 可能使用装饰器/注册机制 | 硬编码元组 |
| 命令数量 | 数十个 | 3 个 |
| 元数据 | 丰富（参数、选项、别名等） | 基础（名称、描述、状态） |
| 动态注册 | 支持 | 不支持 |
| 验证 | 编译时 + 运行时 | 无（需手动添加） |

原始 Claude Code 使用复杂的命令注册系统（可能基于