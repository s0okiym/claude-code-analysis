# 子领域分析：工作空间清单生成系统

## 1. 功能概述

Port Manifest（移植清单）系统负责动态扫描 Python 源代码树，统计文件分布，生成结构化的工作空间清单。这是整个移植工作空间的"自描述"机制。

## 2. 核心组件

### 2.1 PortManifest 数据类

```python
@dataclass(frozen=True)
class PortManifest:
    src_root: Path                           # 源代码根目录
    total_python_files: int                  # Python 文件总数
    top_level_modules: tuple[Subsystem, ...] # 顶层模块列表
```

### 2.2 清单构建函数

```python
def build_port_manifest(src_root: Path | None = None) -> PortManifest:
    root = src_root or DEFAULT_SRC_ROOT
    
    # 1. 递归扫描 Python 文件
    files = [path for path in root.rglob('*.py') if path.is_file()]
    
    # 2. 统计每个顶层模块的文件数
    counter = Counter(
        path.relative_to(root).parts[0] if len(path.relative_to(root).parts) > 1 
        else path.name
        for path in files
        if path.name != '__pycache__'
    )
    
    # 3. 人工注释映射
    notes = {
        '__init__.py': 'package export surface',
        'main.py': 'CLI entrypoint',
        'port_manifest.py': 'workspace manifest generation',
        ...
    }
    
    # 4. 构建 Subsystem 列表
    modules = tuple(
        Subsystem(name=name, path=f'src/{name}', file_count=count, 
                  notes=notes.get(name, 'Python port support module'))
        for name, count in counter.most_common()
    )
    
    return PortManifest(src_root=root, total_python_files=len(files), 
                        top_level_modules=modules)
```

## 3. 流程分析

### 3.1 扫描流程

```
build_port_manifest()
  │
  ├─ 1. 确定根目录
  │     └─ 参数传入 或 使用默认值 (src/)
  │
  ├─ 2. 递归扫描
  │     └─ Path.rglob('*.py') → 生成器遍历所有子目录
  │
  ├─ 3. 过滤文件
  │     └─ path.is_file() 确保是文件而非目录
  │     └─ 排除 __pycache__
  │
  ├─ 4. 提取模块名
  │     └─ path.relative_to(root).parts[0]
  │     └─ 单文件直接取文件名
  │
  ├─ 5. 统计计数
  │     └─ Counter 聚合统计
  │
  ├─ 6. 排序
  │     └─ counter.most_common() 按文件数降序
  │
  ├─ 7. 构建 Subsystem 对象
  │     └─ 添加人工注释
  │
  └─ 8. 返回 PortManifest
```

### 3.2 目录结构解析示例

```
src/
  ├─ main.py              → parts = ['main.py'] → name = 'main.py'
  ├─ models.py            → parts = ['models.py'] → name = 'models.py'
  ├─ commands.py          → parts = ['commands.py'] → name = 'commands.py'
  ├─ utils/
  │   ├─ helpers.py       → parts = ['utils', 'helpers.py'] → name = 'utils'
  │   └─ parser.py        → parts = ['utils', 'parser.py'] → name = 'utils'
  └─ core/
      └─ engine.py        → parts = ['core', 'engine.py'] → name = 'core'

统计结果:
  main.py: 1
  models.py: 1
  commands.py: 1
  utils: 2
  core: 1
```

## 4. 实现原理

### 4.1 Path.rglob 工作机制

```python
# rglob = recursive glob
for path in root.rglob('*.py'):
    # 递归遍历所有子目录，匹配 *.py 模式
    # 返回 Path 对象生成器
```

**性能特点**:
- 惰性求值：生成器按需产生路径
- 递归深度：受文件系统限制
- 符号链接：默认跟随（可能循环）

### 4.2 Counter 聚合机制

```python
from collections import Counter

counter = Counter(['a', 'b', 'a', 'c', 'a', 'b'])
# Counter({'a': 3, 'b': 2, 'c': 1})

counter.most_common()
# [('a', 3), ('b', 2), ('c', 1)]
```

**优势**:
- O(n) 时间复杂度
- 自动处理缺失键（返回 0）
- 支持数学运算（+, -, &, |）

### 4.3 相对路径处理

```python
root = Path('/project/src')
file = Path('/project/src/utils/helpers.py')

rel = file.relative_to(root)
# Path('utils/helpers.py')

rel.parts
# ('utils', 'helpers.py')

rel.parts[0]
# 'utils'
```

## 5. 设计决策

### 5.1 为什么使用动态扫描而非硬编码?

| 方案 | 优点 | 缺点 |
|------|------|------|
| 动态扫描 | 自动反映当前状态，无需维护 | 运行时开销 |
| 硬编码列表 | 零运行时开销 | 需手动更新，易遗漏 |

**选择理由**:
- 项目早期，文件变动频繁
- 扫描开销小（<100ms）
- 避免维护清单与实际代码不同步

### 5.2 为什么使用 parts[0] 作为模块名?

```python
# 方案对比
path.parent.name  # 父目录名（多级目录会丢失层级）
path.stem         # 文件名（不含扩展名）
path.parts[0]     # 相对路径第一部分
```

**选择理由**:
- 顶层目录或文件名作为模块标识
- 保留包结构信息
- 简单且符合直觉

### 5.3 为什么使用 tuple 存储模块列表?

```python
top_level_modules: tuple[Subsystem, ...]
# 而非
# top_level_modules: list[Subsystem]
```

**理由**:
- 与 `frozen=True` 保持一致
- 防止意外修改扫描结果
- 可哈希，便于缓存

## 6. 扩展性分析

### 6.1 添加新统计维度

```python
@dataclass(frozen=True)
class Subsystem:
    name: str
    path: str
    file_count: int
    lines_of_code: int  # 新增：代码行数
    test_count: int     # 新增：测试文件数
    notes: str

def build_port_manifest(...) -> PortManifest:
    ...
    # 统计代码行数
    loc = sum(
        len(path.read_text().splitlines())
        for path in files
        if path.parent.name == name
    )
```

### 6.2 支持多语言

```python
def build_port_manifest(
    src_root: Path | None = None,
    extensions: tuple[str, ...] = ('*.py', '*.pyi')
) -> PortManifest:
    files = []
    for ext in extensions:
        files.extend(root.rglob(ext))
```

### 6.3 缓存机制

```python
from functools import lru_cache

@lru_cache(maxsize=1)
def build_port_manifest_cached(src_root: Path) -> PortManifest:
    """带缓存的版本，适合频繁调用"""
    return _build_port_manifest_impl(src_root)
```

## 7. 存在的问题

### 7.1 当前限制

1. **单层统计**: 只统计顶层，不统计嵌套子目录
2. **无文件内容分析**: 只统计数量，不分析导入关系、复杂度等
3. **硬编码注释**: notes 字典需手动维护
4. **无增量更新**: 每次全量扫描

### 7.2 边界情况

```python
# 空目录
build_port_manifest(Path('/empty'))  # total_python_files=0, modules=()

# 无 .py 文件但有很多其他文件
# 只统计 .py，其他忽略

# 符号链接循环
# rglob 可能无限递归（需设置 limit 或检测循环）
```

### 7.3 改进建议

1. 添加嵌套模块统计：

```python
@dataclass(frozen=True)
class ModuleTree:
    name: str
    file_count: int
    children: tuple[ModuleTree, ...]
```

2. 支持文件内容分析：

```python
def analyze_imports(path: Path) -> list[str]:
    """分析文件导入关系"""
    content = path.read_text()
    # 解析 import 语句
    ...
```

3. 自动注释生成：

```python
def auto_generate_notes(path: Path) -> str:
    """从文件 docstring 或类名推断注释"""
    ...
```

## 8. 测试策略

### 8.1 单元测试

```python
def test_build_port_manifest_empty(tmp_path):
    """测试空目录"""
    manifest = build_port_manifest(tmp_path)
    assert manifest.total_python_files == 0
    assert manifest.top_level_modules == ()

def test_build_port_manifest_with_files(tmp_path):
    """测试包含文件的目录"""
    (tmp_path / 'a.py').write_text('')
    (tmp_path / 'b.py').write_text('')
    (tmp_path / 'subdir').mkdir()
    (tmp_path / 'subdir' / 'c.py').write_text('')
    
    manifest = build_port_manifest(tmp_path)
    assert manifest.total_python_files == 3
    names = [m.name for m in manifest.top_level_modules]
    assert 'a.py' in names
    assert 'b.py' in names
    assert 'subdir' in names
```

### 8.2 集成测试

```python
def test_port_manifest_to_markdown():
    """测试 Markdown 输出格式"""
    manifest = build_port_manifest()
    md = manifest.to_markdown()
    assert 'Port root:' in md
    assert 'Total Python files:' in md
    assert 'Top-level Python modules:' in md
```

## 9. 与原始 Claude Code 对比

| 特性 | 原始 (TypeScript) | 本实现 (Python) |
|------|-------------------|-----------------|
| 文件扫描 | 可能使用 glob 或 fs.readdir | pathlib.Path.rglob |
| 统计 | 手动计数或使用库 | collections.Counter |
| 路径处理 | path.join / path.resolve | pathlib.Path |
| 缓存 | 可能有复杂缓存策略 | 无（或 functools.lru_cache） |
| 复杂度 | 高（大型项目统计） | 低（小型项目） |

原始 Claude Code 需要处理 ~1900 文件的复杂统计，本实现针对小型移植工作空间简化设计。

## 10. 评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 准确性 | ★★★★★ | 准确反映文件系统状态 |
| 性能 | ★★★★★ | 小型项目毫秒级 |
| 可维护性 | ★★★★☆ | 简单清晰，但注释需手动维护 |
| 扩展性 | ★★★☆☆ | 基础设计，需重构支持复杂需求 |
| 健壮性 | ★★★☆☆ | 缺少边界情况处理 |

## 11. 关键代码片段

### 11.1 完整实现

```python
from __future__ import annotations
from collections import Counter
from dataclasses import dataclass
from pathlib import Path
from .models import Subsystem

DEFAULT_SRC_ROOT = Path(__file__).resolve().parent

@dataclass(frozen=True)
class PortManifest:
    src_root: Path
    total_python_files: int
    top_level_modules: tuple[Subsystem, ...]

    def to_markdown(self) -> str:
        lines = [
            f'Port root: `{self.src_root}`',
            f'Total Python files: **{self.total_python_files}**',
            '',
            'Top-level Python modules:',
        ]
        for module in self.top_level_modules:
