# 子领域分析：数据类建模与不可变对象设计

## 1. 功能概述

数据类建模层定义了系统中所有核心数据结构的形状和行为，使用 Python 的 `@dataclass` 装饰器实现类型安全、简洁的数据模型。

## 2. 核心数据类

### 2.1 Subsystem — 子系统/模块

```python
@dataclass(frozen=True)
class Subsystem:
    name: str        # 模块名称，如 "main"
    path: str        # 相对路径，如 "src/main"
    file_count: int  # 包含的文件数量
    notes: str       # 人工注释说明
```

**设计要点**:
- `frozen=True`: 创建后不可修改
- 所有字段都有明确类型注解
- 无默认值，强制调用者提供所有信息

### 2.2 PortingModule — 移植模块

```python
@dataclass(frozen=True)
class PortingModule:
    name: str           # 模块名，如 "port_manifest"
    responsibility: str # 职责描述
    source_hint: str    # 源文件提示，如 "src/port_manifest.py"
    status: str = 'planned'  # 状态，默认 "planned"
```

**设计要点**:
- `status` 有默认值 `'planned'`，可选参数后置
- 表示移植工作项的元数据

### 2.3 PortingBacklog — 移植待办清单

```python
@dataclass
class PortingBacklog:
    title: str                    # 清单标题
    modules: list[PortingModule] = field(default_factory=list)
```

**设计要点**:
- 使用 `field(default_factory=list)` 避免可变默认参数陷阱
- 非 frozen，允许动态添加模块

### 2.4 PortingTask — 移植任务

```python
@dataclass(frozen=True)
class PortingTask:
    title: str       # 任务标题
    detail: str      # 任务详情
    completed: bool = False  # 是否完成
```

**设计要点**:
- 预留用于更细粒度的任务跟踪
- 当前未在核心流程中使用

## 3. 实现原理

### 3.1 @dataclass 工作机制

`@dataclass` 是 Python 3.7+ 的装饰器，自动为类生成特殊方法：

```python
# 手写等效代码
class Subsystem:
    def __init__(self, name: str, path: str, file_count: int, notes: str):
        self.name = name
        self.path = path
        self.file_count = file_count
        self.notes = notes
    
    def __repr__(self):
        return f'Subsystem(name={self.name!r}, ...)'
    
    def __eq__(self, other):
        if not isinstance(other, Subsystem):
            return NotImplemented
        return (self.name, self.path, self.file_count, self.notes) == \
               (other.name, other.path, other.file_count, other.notes)
    
    def __hash__(self):
        return hash((self.name, self.path, self.file_count, self.notes))
```

### 3.2 frozen=True 的语义

当 `frozen=True` 时：

1. **字段只读**: 实例创建后不能修改字段
2. **生成 __hash__**: 实例可哈希，可用作 dict key 或 set 元素
3. **防御性拷贝**: 不需要担心外部修改

```python
subsystem = Subsystem("main", "src/main", 1, "entry point")

# 以下操作会报错
subsystem.name = "new_name"  # dataclasses.FrozenInstanceError
```

### 3.3 default_factory 的必要性

```python
# 错误做法：使用可变默认值
@dataclass
class Backlog:
    modules: list = []  # 危险！所有实例共享同一个列表

# 正确做法：使用 default_factory
@dataclass
class Backlog:
    modules: list = field(default_factory=list)  # 每个实例独立列表
```

## 4. 设计决策分析

### 4.1 为什么使用 dataclass 而非普通类?

| 特性 | dataclass | 普通类 | NamedTuple |
|------|-----------|--------|------------|
| 样板代码 | 极少 | 多 | 少 |
| 可变性 | 可控 | 完全可控 | 不可变 |
| 类型注解 | 原生支持 | 手动 | 原生支持 |
| 继承 | 支持 | 支持 | 有限 |
| 性能 | 好 | 最好 | 最好 |

**选择理由**:
- 减少样板代码（`__init__`, `__repr__`, `__eq__`）
- 类型安全，IDE 友好
- 灵活性（可选择 frozen/mutable）

### 4.2 为什么混合使用 frozen 和 mutable?

| 类 | frozen | 理由 |
|----|--------|------|
| Subsystem | ✓ | 扫描结果，不应修改 |
| PortingModule | ✓ | 元数据定义，不应修改 |
| PortingTask | ✓ | 任务定义，完成后不应修改 |
| PortingBacklog | ✗ | 需要动态添加模块 |

**设计原则**: 默认 immutable，仅在需要修改时使用 mutable。

### 4.3 为什么使用 tuple 而非 list?

```python
# PortManifest 中使用 tuple
top_level_modules: tuple[Subsystem, ...]

# 而非 list
top_level_modules: list[Subsystem]
```

**理由**:
- 与 `frozen=True` 语义一致
- 防止意外修改（`append`, `extend` 等）
- 可哈希（如果元素也可哈希）

## 5. 类型安全分析

### 5.1 类型注解覆盖

```python
# 完整类型注解示例
from __future__ import annotations  # 支持 Python 3.9+ 的 | 语法
from pathlib import Path
from collections.abc import Sequence

@dataclass(frozen=True)
class PortManifest:
    src_root: Path
    total_python_files: int
    top_level_modules: tuple[Subsystem, ...]
    
    def to_markdown(self) -> str: ...
```

### 5.2 静态类型检查

使用 mypy 可进行静态类型检查：

```bash
mypy src/models.py
```

潜在问题：
- `PortingModule.status` 使用 `str`，可考虑 `Literal['planned', 'implemented', 'in-progress']`
- 部分返回类型如 `summary_lines() -> list[str]` 已正确注解

## 6. 扩展性分析

### 6.1 添加新字段

```python
@dataclass(frozen=True)
class Subsystem:
    name: str
    path: str
    file_count: int
    notes: str
    lines_of_code: int = 0  # 新增字段，带默认值
```

**注意**: frozen 类添加字段需小心，已有代码可能未提供新字段。

### 6.2 添加方法

```python
@dataclass(frozen=True)
class Subsystem:
    ...
    
    @property
    def is_large(self) -> bool:
        """判断是否为大模块（文件数 > 10）"""
        return self.file_count > 10
    
    def with_notes(self, new_notes: str) -> Subsystem:
        """创建带有新注释的副本（函数式更新）"""
        return Subsystem(
            name=self.name,
            path=self.path,
            file_count=self.file_count,
            notes=new_notes
        )
```

### 6.3 继承与组合

```python
# 继承（谨慎使用）
@dataclass(frozen=True)
class PythonSubsystem(Subsystem):
    has_tests: bool = False

# 组合（推荐）
@dataclass(frozen=True)
class SubsystemWithMetadata:
    subsystem: Subsystem
    metadata: dict[str, str]
```

## 7. 存在的问题

### 7.1 当前限制

1. **状态字符串**: `status: str` 没有限制取值范围
2. **验证缺失**: 无字段值验证（如 `file_count` 应为非负）
3. **文档不足**: 部分字段缺少 docstring

### 7.2 改进建议

1. 使用 `Literal` 类型限制状态取值：

```python
from typing import Literal

Status = Literal['planned', 'in-progress', 'implemented', 'deprecated']

@dataclass(frozen=True)
class PortingModule:
    status: Status = 'planned'
```

2. 添加 `__post_init__` 进行验证：

```python
@dataclass(frozen=True)
class Subsystem:
    ...
    
    def __post_init__(self):
        if self.file_count < 0:
            raise ValueError(f'file_count must be non-negative, got {self.file_count}')
```

## 8. 与原始 Claude Code 对比

| 特性 | 原始 (TypeScript) | 本实现 (Python) |
|------|-------------------|-----------------|
| 数据结构 | Interface + Class | @dataclass |
| 不可变性 | readonly 修饰符 | frozen=True |
| 验证 | Zod schema | 手动/__post_init__ |
| 序列化 | 手动 | 需额外库 (dataclasses-json) |
| 类型系统 | 编译时检查 | 运行时 + mypy |

原始代码使用 TypeScript 的严格类型系统和 Zod 进行运行时验证，本实现依赖 Python 的类型注解和可选的静态检查。

## 9. 评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 简洁性 | ★★★★★ | 大幅减少样板代码 |
| 类型安全 | ★★★★☆ | 注解完整，但运行时无强制 |
| 不可变性 | ★★★★★ | frozen 设计合理 |
| 可维护性 | ★★★★★ | 清晰易懂 |
| 性能 | ★★★★☆ | 略慢于普通类，可接受 |

## 10. 最佳实践总结

1. **默认使用 frozen=True**，除非确实需要修改
2. **使用 default_factory 处理可变默认值**
3. **提供完整类型注解**，启用 mypy 检查
4. **在 __post_init__ 中进行验证**
5. **优先使用组合而非继承**
