# 子领域分析：类型注解系统

## 1. 功能概述

类型注解系统使用 Python 3.10+ 的类型提示功能，为代码提供静态类型检查和 IDE 支持。

## 2. 核心用法

### 2.1 基本类型注解

```python
from __future__ import annotations
from pathlib import Path
from dataclasses import dataclass

@dataclass(frozen=True)
class Subsystem:
    name: str        # str 类型
    path: str        # str 类型
    file_count: int  # int 类型
    notes: str       # str 类型
```

### 2.2 可选类型

```python
def build_port_manifest(src_root: Path | None = None) -> PortManifest:
    """Path | None 表示可选参数"""
    root = src_root or DEFAULT_SRC_ROOT
    ...
```

### 2.3 联合类型

```python
# Python 3.10+ 语法（需要 from __future__ import annotations）
argv: list[str] | None = None

# 旧语法（Python 3.9 及以下）
from typing import Optional, List
argv: Optional[List[str]] = None
```

### 2.4 返回类型

```python
def main(argv: list[str] | None = None) -> int:
    """-> int 表示返回整数（退出码）"""
    ...
    return 0

def to_markdown(self) -> str:
    """-> str 表示返回字符串"""
    ...
    return '\n'.join(lines)
```

### 2.5 泛型类型

```python
from collections.abc import Sequence

top_level_modules: tuple[Subsystem, ...]  # 变长元组
modules: list[PortingModule]              # 列表
```

## 3. __future__ annotations 的作用

### 3.1 解决的问题

```python
from __future__ import annotations

# 启用后，类型注解不会立即求值
# 允许使用 Python 3.10+ 语法在旧版本运行

class PortManifest:
    # 如果不启用 __future__，这行在类定义时会求值
    # 但 Subsystem 还未定义完成
    top_level_modules: tuple[Subsystem, ...]
```

### 3.2 延迟求值

```python
# 启用 __future__ 后，注解以字符串形式存储
# 在需要时才求值

class A:
    b: B  # 即使 B 在后面定义，也能正常工作

class B:
    pass
```

### 3.3 性能优化

```python
# 类型注解在运行时会被忽略（除非显式检查）
# 启用 __future__ 后，注解存储为字符串，减少运行时开销
```

## 4. 类型注解覆盖分析

### 4.1 函数签名

| 函数 | 参数类型 | 返回类型 | 完整度 |
|------|----------|----------|--------|
| `build_parser` | 无 | `ArgumentParser` | ✓ |
| `main` | `list[str] \| None` | `int` | ✓ |
| `build_port_manifest` | `Path \| None` | `PortManifest` | ✓ |
| `to_markdown` | `self` | `str` | ✓ |
| `render_summary` | `self` | `str` | ✓ |
| `summary_lines` | `self` | `list[str]` | ✓ |

### 4.2 数据类字段

| 类 | 字段 | 类型 | 默认值 |
|----|------|------|--------|
| `Subsystem` | `name` | `str` | - |
| `Subsystem` | `path` | `str` | - |
| `Subsystem` | `file_count` | `int` | - |
| `Subsystem` | `notes` | `str` | - |
| `PortingModule` | `name` | `str` | - |
| `PortingModule` | `responsibility` | `str` | - |
| `PortingModule` | `source_hint` | `str` | - |
| `PortingModule` | `status` | `str` | `'planned'` |

## 5. 设计决策

### 5.1 为什么使用 from __future__ import annotations?

| 因素 | 启用 | 不启用 |
|------|------|--------|
| 语法 | 可用 `\|` 联合类型 | 需用 `Union` 或 `Optional` |
| 前向引用 | 自动支持 | 需用字符串 `"ClassName"` |
| 运行时开销 | 更低 | 较高 |
| 兼容性 | Python 3.7+ | 所有版本 |

**选择理由**:
- 代码更简洁（`Path | None` vs `Optional[Path]`）
- 支持前向引用
- 符合现代 Python 风格

### 5.2 为什么使用 tuple[...] 而非 List[...]?

```python
# 当前使用
top_level_modules: tuple[Subsystem, ...]

# 对比
# top_level_modules: list[Subsystem]
```

**理由**:
- 与 `frozen=True` 语义一致
- 不可变，防止意外修改
- 可哈希（如果元素也可哈希）

### 5.3 为什么返回 int 而非 bool?

```python
def main(...) -> int:  # 而非 -> bool
    ...
    return 0  # 成功
```

**理由**:
- Unix 退出码惯例（0=成功，非0=错误）
- 支持多种错误码
- 与 `sys.exit()` 兼容

## 6. 静态类型检查

### 6.1 mypy 配置

```ini
# mypy.ini
[mypy]
python_version = 3.10
warn_return_any = True
warn_unused_configs = True
disallow_untyped_defs = True
disallow_incomplete_defs = True
```

### 6.2 运行检查

```bash
# 检查整个项目
mypy src/

# 检查单个文件
mypy src/models.py
```

### 6.3 预期输出

```
src/models.py:10: error: Need type annotation for "modules"
# 或
Success: no issues found in 8 source files
```

## 7. 存在的问题

### 7.1 当前限制

1. **无运行时检查**: 类型注解仅在静态检查时有效
2. **复杂类型**: 某些复杂类型难以表达
3. **第三方库**: 部分库缺少类型存根

### 7.2 改进建议

1. 添加运行时验证（可选）:

```python
from dataclasses import dataclass, field
from typing import get_type_hints

def validate_types(obj):
    """运行时类型验证"""
    hints = get_type_hints(type(obj))
    for name, expected_type in hints.items():
        value = getattr(obj, name)
        if not isinstance(value, expected_type):
            raise TypeError(f'{name} should be {expected_type}')
```

2. 使用更精确的类型:

```python
from typing import Literal

Status = Literal['planned', 'in-progress', 'implemented']

@dataclass(frozen=True)
class PortingModule:
    status: Status = 'planned'  # 更精确
```

## 8. 与原始 Claude Code 对比

| 特性 | 原始 (TypeScript) | 本实现 (Python) |
|------|-------------------|-----------------|
| 类型系统 | 编译时强制 | 静态检查 + 运行时忽略 |
| 语法 | 原生支持 | 注解语法 |
| 工具 | tsc | mypy, pyright |
| 严格程度 | 高 | 可配置 |
| 运行时 | 类型擦除 | 类型忽略 |

TypeScript 的类型系统在编译时强制执行，Python 的类型注解仅在静态检查时有效。

## 9. 评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 覆盖率 | ★★★★★ | 几乎 100% |
| 精确度 | ★★★★☆ | 可用 Literal 改进 |
| 现代性 | ★★★★★ | 使用 Python 3.10+ 语法 |
| 工具支持 | ★★★★☆ | mypy 支持良好 |
| 运行时 | ★★☆☆☆ | 无运行时检查 |

## 10. 关键代码片段

### 10.1 类型注解示例

```python
from __future__ import annotations
from pathlib import Path
from collections import Counter
from dataclasses import dataclass, field

# 数据类
@dataclass(frozen=True)
class PortManifest:
    src_root: Path
    total_python_files: int
    top_level_modules: tuple[Subsystem, ...]

# 函数签名
def build_port_manifest(src_root: Path | None = None) -> PortManifest:
    files: list[Path] = [...]
    counter: Counter[str] = Counter(...)
    ...

# 类方法
@classmethod
def from_workspace(cls) -> QueryEnginePort:
    ...
```

### 10.2 类型别名

```python
from typing import TypeAlias

# 定义类型别名
ModuleName: TypeAlias = str
FileCount: TypeAlias = int
Status: TypeAlias = Literal['planned', 'in-progress', 'implemented']

# 使用
@dataclass(frozen=True)
class PortingModule:
    name: ModuleName
    file_count: FileCount
    status: Status
```

## 11. 最佳实践

1. **始终使用 `from __future__ import annotations`** — 启用现代语法
2. **为所有公共 API 添加类型注解** — 提高可用性
3. **使用 mypy 进行静态检查** — 捕获类型错误
4. **避免 `Any`** — 失去类型安全
5. **使用 `None` 而非 `Optional`** — Python 3.10+ 风格
