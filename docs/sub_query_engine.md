# 子领域分析：查询引擎与协调系统

## 1. 功能概述

Query Engine（查询引擎）是系统的协调层，负责整合多个数据源（清单、命令、工具），生成统一的 Markdown 摘要报告。它实现了**协调器模式 (Coordinator Pattern)**。

## 2. 核心组件

### 2.1 QueryEnginePort 类

```python
@dataclass
class QueryEnginePort:
    manifest: PortManifest  # 依赖：清单生成器
    
    @classmethod
    def from_workspace(cls) -> 'QueryEnginePort':
        """工厂方法：从工作空间自动创建"""
        return cls(manifest=build_port_manifest())
    
    def render_summary(self) -> str:
        """渲染完整摘要"""
        command_backlog = build_command_backlog()
        tool_backlog = build_tool_backlog()
        
        sections = [
            '# Python Porting Workspace Summary',
            '',
            self.manifest.to_markdown(),
            '',
            f'{command_backlog.title}:',
            *command_backlog.summary_lines(),
            '',
            f'{tool_backlog.title}:',
            *tool_backlog.summary_lines(),
        ]
        return '\n'.join(sections)
```

## 3. 流程分析

### 3.1 摘要渲染流程

```
QueryEnginePort.render_summary()
  │
  ├─ 1. 获取命令清单
  │     └─ build_command_backlog()
  │        └─ PortingBacklog(title='Command surface', modules=[...])
  │
  ├─ 2. 获取工具清单
  │     └─ build_tool_backlog()
  │        └─ PortingBacklog(title='Tool surface', modules=[...])
  │
  ├─ 3. 获取清单 Markdown
  │     └─ self.manifest.to_markdown()
  │        └─ 多行字符串
  │
  ├─ 4. 组合 sections
  │     ├─ 标题
  │     ├─ 清单 Markdown
  │     ├─ 空行
  │     ├─ 命令清单标题
  │     ├─ 命令清单行（展开）
  │     ├─ 空行
  │     ├─ 工具清单标题
  │     └─ 工具清单行（展开）
  │
  └─ 5. 拼接并返回
        └─ '\n'.join(sections)
```

### 3.2 输出示例

```markdown
# Python Porting Workspace Summary

Port root: `/project/src`
Total Python files: **8**

Top-level Python modules:
- `main.py` (1 files) — CLI entrypoint
- `models.py` (1 files) — shared dataclasses
...

Command surface:
- main [implemented] — Expose a Python CLI for manifest and backlog reporting (from src/main.py)
- summary [implemented] — Render a Markdown overview of the current porting workspace (from src/query_engine.py)
- subsystems [implemented] — List the current Python modules participating in the rewrite (from src/port_manifest.py)

Tool surface:
- port_manifest [implemented] — Inspect the active Python source tree and summarize the current rewrite surface (from src/port_manifest.py)
- backlog_models [implemented] — Represent subsystem and backlog metadata as Python dataclasses (from src/models.py)
- query_engine [implemented] — Coordinate Python-facing rewrite summaries and reporting (from src/query_engine.py)
```

## 4. 实现原理

### 4.1 协调器模式

```
┌─────────────────────────────────────────┐
│         QueryEnginePort                 │
│         (协调器/门面)                    │
├─────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────────┐  │
│  │ PortManifest│  │ PortingBacklog  │  │
│  │   (清单)    │  │   (命令)        │  │
│  └─────────────┘  └─────────────────┘  │
│  ┌─────────────────────────────────┐   │
│  │      PortingBacklog             │   │
│  │        (工具)                   │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**模式特点**:
- 单一入口：客户端只需与 `QueryEnginePort` 交互
- 封装复杂性：隐藏多个数据源的协调细节
- 灵活组合：可轻松添加新数据源

### 4.2 依赖注入

```python
@dataclass
class QueryEnginePort:
    manifest: PortManifest  # 构造函数注入
    
    @classmethod
    def from_workspace(cls) -> 'QueryEnginePort':
        """工厂方法：自动注入依赖"""
        return cls(manifest=build_port_mark())
```

**优势**:
- 测试友好：可注入 mock 对象
- 灵活：支持不同配置
- 清晰：依赖关系显式

### 4.3 延迟求值

```python
# 每次调用 render_summary() 都重新获取数据
def render_summary(self) -> str:
    command_backlog = build_command_backlog()  # 重新构建
    tool_backlog = build_tool_backlog()        # 重新构建
    ...
```

**设计选择**:
- 不缓存结果，确保数据最新
- 适合小型项目，开销可忽略
- 大型项目可考虑缓存

## 5. 设计决策

### 5.1 为什么使用类而非函数?

| 方案 | 代码 | 适用场景 |
|------|------|----------|
| 函数 | `render_summary(manifest)` | 简单场景，无状态 |
| 类 | `QueryEnginePort(manifest).render_summary()` | 需要封装状态或配置 |

**选择理由**:
- 封装 `manifest` 依赖
- 支持工厂方法 `from_workspace()`
- 未来可扩展更多配置参数

### 5.2 为什么使用 dataclass?

```python
@dataclass  # 非 frozen，允许后续修改
class QueryEnginePort:
    manifest: PortManifest
```

**理由**:
- 自动生成 `__init__`
- 清晰声明依赖
- 非 frozen，支持运行时替换 manifest（如重新扫描）

### 5.3 为什么 sections 使用列表拼接?

```python
sections = [
    '# Title',
    '',
    content,
    '',
    *list_items,  # 展开操作符
]
return '\n'.join(sections)
```

**优势**:
- 结构清晰，易于修改
- 使用 `*` 展开可迭代对象
- 避免字符串拼接的 `+` 操作

## 6. 扩展性分析

### 6.1 添加新数据源

```python
def render_summary(self) -> str:
    command_backlog = build_command_backlog()
    tool_backlog = build_tool_backlog()
    test_backlog = build_test_backlog()  # 新增
    
    sections = [
        ...
        f'{test_backlog.title}:',
        *test_backlog.summary_lines(),
    ]
    return '\n'.join(sections)
```

### 6.2 支持多种输出格式

```python
from enum import Enum

class OutputFormat(Enum):
    MARKDOWN = 'md'
    JSON = 'json'
    YAML = 'yaml'

class QueryEnginePort:
    def render(self, format: OutputFormat = OutputFormat.MARKDOWN) -> str:
        if format == OutputFormat.MARKDOWN:
            return self.render_summary()
        elif format == OutputFormat.JSON:
            return self.render_json()
        ...
```

### 6.3 添加过滤和排序

```python
def render_summary(
    self,
    status_filter: str | None = None,
    sort_by: str = 'name'
) -> str:
    command_backlog = build_command_backlog()
    
    if status_filter:
        command_backlog.modules = [
            m for m in command_backlog.modules
            if m.status == status_filter
        ]
    
    if sort_by == 'name':
        command_backlog.modules.sort(key=lambda m: m.name)
    ...
```

## 7. 存在的问题

### 7.1 当前限制

1. **紧耦合**: 直接依赖 `build_command_backlog()` 等全局函数
2. **无缓存**: 每次调用都重新计算
3. **单一格式**: 仅支持 Markdown
4. **无错误处理**: 假设所有数据源都可用

### 7.2 改进建议

1. **接口抽象**: 定义数据源接口

```python
from abc import ABC, abstractmethod

class DataSource(ABC):
    @abstractmethod
    def get_backlog(self) -> PortingBacklog: ...

class CommandDataSource(DataSource):
    def get_backlog(self) -> PortingBacklog:
        return build_command_backlog()

class QueryEnginePort:
    def __init__(self, sources: list[DataSource]):
        self.sources = sources
```

2. **结果缓存**: 添加可选缓存

```python
from functools import cached_property

class QueryEnginePort:
    @cached_property
    def _command_backlog(self) -> PortingBacklog:
        return build_command_backlog()
```

3. **模板系统**: 支持自定义模板

```python
from string import Template

def render_with_template(self, template: Template) -> str:
    return template.substitute(
        manifest=self.manifest.to_markdown(),
        commands='\n'.join(self._command_backlog.summary_lines()),
        ...
    )
```

## 8. 测试策略

### 8.1 单元测试

```python
def test_render_summary_structure():
    """测试输出结构"""
    manifest = PortManifest(...)
    engine = QueryEnginePort(manifest)
    
    summary = engine.render_summary()
    
    assert '# Python Porting Workspace Summary' in summary
    assert 'Command surface:' in summary
    assert 'Tool surface:' in summary

def test_render_summary_includes_manifest():
    """测试包含清单内容"""
    manifest = PortManifest(
        src_root=Path('/test'),
        total_python_files=5,
        top_level_modules=()
    )
    engine = QueryEnginePort(manifest)
    
    summary = engine.render_summary()
    
    assert 'Total Python files: **5**' in summary
```

### 8.2 Mock 测试

```python
from unittest.mock import patch, MagicMock

def test_render_summary_with_mocked_backlog():
    """使用 mock 隔离依赖"""
    mock_backlog = MagicMock()
    mock_backlog.title = 'Mock Commands'
    mock_backlog.summary_lines.return_value = ['- cmd1', '- cmd2']
    
    with patch('src.query_engine.build_command_backlog', return_value=mock_backlog):
        engine = QueryEnginePort.from_workspace()
        summary = engine.render_summary()
        
        assert 'Mock Commands:' in summary
        assert '- cmd1' in summary
```

## 9. 与原始 Claude Code 对比

| 特性 | 原始 (TypeScript) | 本实现 (Python) |
|------|-------------------|-----------------|
| 架构模式 | 可能使用服务容器 | 简单协调器 |
| 依赖管理 | 依赖注入框架 | 构造函数注入 |
| 输出格式 | 多种（JSON, 文本） | Markdown |
| 缓存策略 | 复杂缓存 | 无缓存 |
| 复杂度 | 高 | 低 |

原始 Claude Code 的查询引擎需要处理复杂的 Agent 循环、工具调用等，本实现仅做简单的数据整合。

## 10. 评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 简洁性 | ★★★★★ | 代码精简，职责单一 |
| 可测试性 | ★★★★☆ | 依赖注入便于测试，但依赖全局函数 |
| 可扩展性 | ★★★☆☆ | 需修改代码添加新数据源 |
| 灵活性 | ★★★☆☆ | 输出格式固定 |
| 性能 | ★★★★☆ | 无缓存，但开销小 |

## 11. 关键代码片段

### 11.1 完整实现

```python
from __future__ import annotations
from dataclasses import dataclass
from .commands import build_command_backlog
from .port_manifest import PortManifest, build_port_manifest
from .tools import build_tool_backlog


@dataclass
class QueryEnginePort:
    manifest: PortManifest

    @classmethod
    def from_workspace(cls) -> 'QueryEnginePort':
        return cls(manifest=build_port_manifest())

    def render_summary(self) -> str:
        command_backlog = build_command_backlog()
        tool_backlog = build_tool_backlog()
        sections = [
            '# Python Porting Workspace Summary',
            '',
            self.manifest.to_markdown(),
            '',
            f'{command_backlog.title}:',
            *command_backlog.summary_lines(),
            '',
            f'{tool_backlog.title}:',
            *tool_backlog.summary_lines(),
        ]
        return '\n'.join(sections)
```

### 11.2 使用示例

```python
# 方法 1：显式传入 manifest
manifest = build_port_manifest(Path('/custom/src'))
engine = QueryEnginePort(manifest)
print(engine.render_summary())

# 方法 2：使用工厂方法
engine = QueryEnginePort.from_workspace()
print(engine.render_summary())
```

## 12. 设计模式总结

| 模式 | 应用 | 收益 |
|------|------|------|
| 协调器模式 | QueryEnginePort 整合多数据源 | 简化客户端调用 |
| 依赖注入 | manifest 通过构造函数传入 | 便于测试和配置 |
| 工厂方法 | from_workspace() | 封装创建逻辑 |
| 组合 | sections 列表组合 | 灵活构建输出 |
