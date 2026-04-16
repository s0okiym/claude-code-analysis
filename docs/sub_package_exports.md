# 子领域分析：包导出与模块接口设计

## 1. 功能概述

`__init__.py` 定义了包的公共 API 接口，控制哪些模块和类对外暴露，是模块封装的边界。

## 2. 核心实现

### 2.1 当前导出设计

```python
"""Python porting workspace for the Claude Code rewrite effort."""

from .commands import PORTED_COMMANDS, build_command_backlog
from .port_manifest import PortManifest, build_port_manifest
from .query_engine import QueryEnginePort
from .tools import PORTED_TOOLS, build_tool_backlog

__all__ = [
    'PortManifest',
    'QueryEnginePort',
    'PORTED_COMMANDS',
    'PORTED_TOOLS',
    'build_command_backlog',
    'build_port_manifest',
    'build_tool_backlog',
]
```

## 3. 导出策略分析

### 3.1 __all__ 的作用

```python
__all__ = ['PortManifest', 'QueryEnginePort', ...]
```

**行为**:
- `from src import *` 只导入 `__all__` 中的名称
- 显式导入不受影响：`from src import models` 仍可访问
- 文档化公共 API

### 3.2 导出分类

| 类别 | 导出项 | 用途 |
|------|--------|------|
| 数据类 | `PortManifest`, `QueryEnginePort` | 类型注解、实例化 |
| 常量 | `PORTED_COMMANDS`, `PORTED_TOOLS` | 只读访问 |
| 构建函数 | `build_*` | 创建对象 |

### 3.3 未导出的模块

```python
# 显式导入可用
from src import models
from src import task

# 但不在 __all__ 中
# from src import *  # 不会导入 models 或 task
```

**理由**:
- `models`: 内部数据类，通常通过其他模块间接使用
- `task`: 当前未在核心流程中使用

## 4. 设计决策

### 4.1 为什么使用 __all__?

| 方案 | 行为 | 评价 |
|------|------|------|
| 使用 `__all__` | 显式控制 `from src import *` | 推荐 |
| 不使用 | 导出所有不以下划线开头的名称 | 可能暴露过多 |
| 空 `__all__` | `from src import *` 不导入任何内容 | 过于严格 |

**选择理由**:
- 显式优于隐式
- 控制公共 API 表面
- 文档化设计意图

### 4.2 为什么导出构建函数而非类构造函数?

```python
# 导出构建函数
__all__ = ['build_port_manifest', ...]

# 对比：导出类
__all__ = ['PortManifest', ...]
```

**当前设计**:
- 同时导出：`PortManifest`（类型）和 `build_port_manifest`（构建）
- 提供灵活性

**理由**:
- 类型注解需要类
- 构建函数可能有额外逻辑（缓存、验证等）
- 未来可能添加参数

### 4.3 为什么导出常量 PORTED_COMMANDS?

```python
from .commands import PORTED_COMMANDS

__all__ = ['PORTED_COMMANDS', ...]
```

**使用场景**:
```python
from src import PORTED_COMMANDS

# 直接访问命令列表
for cmd in PORTED_COMMANDS:
    print(cmd.name)
```

**对比仅导出构建函数**:
```python
from src import build_command_backlog

# 必须通过函数访问
backlog = build_command_backlog()
for cmd in backlog.modules:
    print(cmd.name)
```

**选择理由**:
- 常量访问更直接
- 无需创建中间对象
- 明确只读语义

## 5. 扩展性分析

### 5.1 版本化导出

```python
# 支持版本迁移
__all__ = [
    # v1 API
    'PortManifest',
    'build_port_manifest',
    # v2 API (新增)
    'PortManifestV2',
    'build_port_manifest_v2',
]

# 兼容性别名
PortManifestV1 = PortManifest
```

### 5.2 条件导出

```python
import sys

__all__ = ['PortManifest', 'QueryEnginePort']

# 条件导出（Python 3.10+）
if sys.version_info >= (3, 10):
    from .new_feature import NewFeature
    __all__.append('NewFeature')
```

### 5.3 分级导出

```python
# __init__.py - 核心导出
__all__ = ['PortManifest', 'QueryEnginePort']

# advanced.py - 高级导出
__all__ = ['AdvancedFeature', 'InternalUtility']

# 使用
from src import PortManifest  # 核心
from src.advanced import AdvancedFeature  # 高级
```

## 6. 存在的问题

### 6.1 当前限制

1. **导出粒度**: 全部导出或全部不导出，无法部分导出
2. **无命名空间**: 所有导出项平铺，无分组
3. **无弃用机制**: 无法标记已弃用的 API

### 6.2 改进建议

1. 添加弃用警告：

```python
import warnings

def deprecated_build_port_manifest(*args, **kwargs):
    warnings.warn(
        'deprecated_build_port_manifest is deprecated, use build_port_manifest',
        DeprecationWarning,
        stacklevel=2
    )
    return build_port_manifest(*args, **kwargs)

__all__ = [
    'build_port_manifest',
    'deprecated_build_port_manifest',  # 仍导出但已弃用
]
```

2. 使用命名空间：

```python
# types.py
from dataclasses import dataclass

@dataclass(frozen=True)
class Types:
    """命名空间容器"""
    PortManifest = PortManifest
    PortingModule = PortingModule
    PortingBacklog = PortingBacklog

__all__ = ['Types', ...]

# 使用
from src import Types
manifest: Types.PortManifest = ...
```

3. 添加类型导出：

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from .models import Subsystem, PortingTask

__all__ = [...]

# 类型仅导出用于类型检查
```

## 7. 测试策略

### 7.1 导出测试

```python
def test_all_exports_exist():
    """测试所有导出项存在"""
    import src
    
    for name in src.__all__:
        assert hasattr(src, name), f'{name} not found in src'

def test_star_import():
    """测试 from src import *"""
    # 执行星号导入
    exec('from src import *', {})
    # 如果没有异常，说明 __all__ 定义正确

def test_no_underscore_exports():
    """测试没有以下划线开头的导出"""
    import src
    
    for name in src.__all__:
        assert not name.startswith('_'), f'{name} should not be exported'
```

### 7.2 API 稳定性测试

```python
def test_api_stability():
    """测试 API 稳定性"""
    import src
    
    expected_api = {
        'PortManifest',
        'QueryEnginePort',
        'PORTED_COMMANDS',
        'PORTED_TOOLS',
        'build_command_backlog',
        'build_port_manifest',
        'build_tool_backlog',
    }
    
    actual_api = set(src.__all__)
    
    assert expected_api == actual_api, f'API changed: {expected_api ^ actual_api}'
```

## 8. 与原始 Claude Code 对比

| 特性 | 原始 (TypeScript) | 本实现 (Python) |
|------|-------------------|-----------------|
| 导出机制 | export 语句 | __all__ |
| 命名空间 | 模块/文件级 | 包级 |
| 类型导出 | 自动 | 显式 |
| 分级导出 | 支持 | 需额外设计 |
| 弃用机制 | JSDoc @deprecated | warnings 模块 |

原始代码使用 ES 模块的 `export` 语法，Python 使用 `__all__` 列表。

## 9. 评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 显式性 | ★★★★★ | __all__ 明确声明 |
| 简洁性 | ★★★★★ | 简单列表 |
| 可维护性 | ★★★★☆ | 需手动维护 |
| 扩展性 | ★★★☆☆ | 基础功能 |
| 文档化 | ★★★★☆ | 自文档化 |

## 10. 关键代码片段

### 10.1 完整实现

```python
"""Python porting workspace for the Claude Code rewrite effort."""

from .commands import PORTED_COMMANDS, build_command_backlog
from .port_manifest import PortManifest, build_port_manifest
from .query_engine import QueryEnginePort
from .tools import PORTED_TOOLS, build_tool_backlog

__all__ = [
    'PortManifest',
    'QueryEnginePort',
    'PORTED_COMMANDS',
    'PORTED_TOOLS',
    'build_command_backlog',
    'build_port_manifest',
    'build_tool_backlog',
]
```

### 10.2 使用示例

```python
# 方式 1：星号导入
from src import *
manifest = build_port_manifest()

# 方式 2：显式导入
from src import PortManifest, build_port_manifest
manifest = build_port_manifest()

# 方式 3：导入模块
import src
manifest = src.build_port_manifest()

# 方式 4：访问常量
from src import PORTED_COMMANDS
for cmd in PORTED_COMMANDS:
    print(cmd.name)
```

## 11. 最佳实践

1. **始终定义 `__all__`** — 即使为空
2. **按类别分组** — 数据类、函数、常量
3. **保持稳定性** — 避免频繁修改
4. **文档化** — 模块 docstring 说明用途
5. **渐进导出** — 先导出核心，再扩展
