# 子领域分析：Counter 聚合统计系统

## 1. 功能概述

Counter 是 Python `collections` 模块提供的专用计数器类，本项目中用于统计各模块的 Python 文件数量。

## 2. 核心用法

### 2.1 模块文件统计

```python
from collections import Counter
from pathlib import Path

def build_port_manifest(src_root: Path | None = None) -> PortManifest:
    root = src_root or DEFAULT_SRC_ROOT
    files = [path for path in root.rglob('*.py') if path.is_file()]
    
    # 使用 Counter 统计
    counter = Counter(
        path.relative_to(root).parts[0] if len(path.relative_to(root).parts) > 1 
        else path.name
        for path in files
        if path.name != '__pycache__'
    )
    
    # 按计数降序排列
    modules = tuple(
        Subsystem(name=name, path=f'src/{name}', file_count=count, notes=...)
        for name, count in counter.most_common()
    )
    ...
```

## 3. Counter 详解

### 3.1 基本用法

```python
from collections import Counter

# 创建
words = ['apple', 'banana', 'apple', 'orange', 'banana', 'apple']
counts = Counter(words)
# Counter({'apple': 3, 'banana': 2, 'orange': 1})

# 访问
counts['apple']     # 3
counts['grape']     # 0（不存在返回 0，不报错）

# 获取最常见
counts.most_common(2)  # [('apple', 3), ('banana', 2)]
```

### 3.2 数学运算

```python
c1 = Counter(a=3, b=1)
c2 = Counter(a=1, b=2, c=1)

# 加法（合并计数）
c1 + c2  # Counter({'a': 4, 'b': 3, 'c': 1})

# 减法（只保留正数）
c1 - c2  # Counter({'a': 2})

# 交集（取最小值）
c1 & c2  # Counter({'a': 1, 'b': 1})

# 并集（取最大值）
c1 | c2  # Counter({'a': 3, 'b': 2, 'c': 1})
```

### 3.3 与其他数据结构对比

| 特性 | Counter | dict | defaultdict | 手动计数 |
|------|---------|------|-------------|----------|
| 缺失键 | 返回 0 | KeyError | 返回默认值 | 需检查 |
| 计数 | 专用 | 通用 | 通用 | 通用 |
| 方法 | 丰富 | 基础 | 基础 | 自定义 |
| 性能 | 高 | 高 | 高 | 取决于实现 |

## 4. 实现原理

### 4.1 Counter 继承关系

```python
class Counter(dict):
    """Dict subclass for counting hashable objects"""
    
    def __missing__(self, key):
        return 0  # 缺失键返回 0
    
    def most_common(self, n=None):
        """List the n most common elements"""
        return heapq.nlargest(n, self.items(), key=itemgetter(1))
```

### 4.2 生成器表达式

```python
counter = Counter(
    path.relative_to(root).parts[0] 
    for path in files
)
```

**等价展开**:
```python
counter = Counter()
for path in files:
    key = path.relative_to(root).parts[0]
    counter[key] += 1
```

**优势**:
- 惰性求值，内存友好
- 简洁，一行代码
- 直接传入 Counter 构造函数

### 4.3 most_common() 实现

```python
def most_common(self, n=None):
    """List the n most common elements and their counts.
    
    If n is None, list all elements.
    Ordered from most common to least.
    """
    if n is None:
        return sorted(self.items(), key=lambda x: x[1], reverse=True)
    return heapq.nlargest(n, self.items(), key=lambda x: x[1])
```

**时间复杂度**: O(m log n)，其中 m 是不同元素数量

## 5. 设计决策

### 5.1 为什么选择 Counter?

**场景**: 统计每个模块的文件数量

| 方案 | 代码 | 评价 |
|------|------|------|
| Counter | `Counter(keys)` | 简洁，专用 |
| dict | `{k: list.count(k) for k in set(keys)}` | O(n²)，低效 |
| defaultdict | 需手动累加 | 稍繁琐 |
| pandas | 重型依赖 | 过度设计 |

**选择理由**:
- 标准库，零依赖
- 专为计数设计
- 代码简洁
- 性能优秀

### 5.2 为什么使用 most_common()?

```python
# 按文件数降序排列
for name, count in counter.most_common():
    ...

# 对比：手动排序
for name, count in sorted(counter.items(), key=lambda x: x[1], reverse=True):
    ...
```

**理由**:
- 语义清晰
- 可能使用堆优化（对于 top-n）
- 代码更简洁

### 5.3 为什么过滤 __pycache__?

```python
counter = Counter(
    ...
    for path in files
    if path.name != '__pycache__'  # 过滤
)
```

**理由**:
- `__pycache__` 是 Python 字节码缓存目录
- 不是源代码，不应计入
- 可能包含 `.pyc` 文件（虽然 rglob('*.py') 已过滤）

## 6. 扩展性分析

### 6.1 多维度统计

```python
from collections import defaultdict, Counter

def multi_dimension_stats(root: Path) -> dict:
    """多维度文件统计"""
    stats = {
        'by_extension': Counter(),
        'by_directory': Counter(),
        'by_size': defaultdict(list),  # 大小分桶
    }
    
    for path in root.rglob('*'):
        if path.is_file():
            stats['by_extension'][path.suffix] += 1
            stats['by_directory'][path.parent.name] += 1
            
            size = path.stat().st_size
            bucket = 'small' if size < 1024 else 'medium' if size < 10240 else 'large'
            stats['by_size'][bucket].append(path.name)
    
    return stats
```

### 6.2 加权计数

```python
# 按文件行数加权
counter = Counter()
for path in files:
    lines = len(path.read_text().splitlines())
    module = path.relative_to(root).parts[0]
    counter[module] += lines  # 加权：行数而非文件数
```

### 6.3 时间序列统计

```python
from datetime import datetime

def stats_by_time(root: Path) -> Counter:
    """按修改时间统计"""
    counter = Counter()
    
    for path in root.rglob('*.py'):
        mtime = datetime.fromtimestamp(path.stat().st_mtime)
        key = mtime.strftime('%Y-%m')
        counter[key] += 1
    
    return counter
```

## 7. 存在的问题

### 7.1 当前限制

1. **单层统计**: 只统计顶层，不统计嵌套层级
2. **无去重**: 相同内容的文件会被分别计数
3. **无权重**: 所有文件同等计数（不论大小）

### 7.2 边界情况

```python
# 空列表
Counter([])  # Counter()

# 大量唯一值
Counter(range(1000000))  # 内存占用大

# 不可哈希元素
Counter([['a'], ['b']])  # TypeError: unhashable type: 'list'
```

### 7.3 改进建议

1. 嵌套统计：

```python
def nested_counter(root: Path) -> dict:
    """嵌套层级统计"""
    result = {}
    
    for path in root.rglob('*.py'):
        rel = path.relative_to(root)
        parts = rel.parts
        
        current = result
        for part in parts[:-1]:  # 除文件名外的路径
            if part not in current:
                current[part] = {'_count': 0, '_children': {}}
            current = current[part]['_children']
        
        # 增加计数
        if parts[0] in result:
            result[parts[0]]['_count'] += 1
    
    return result
```

2. 使用 Counter 子类：

```python
class FileCounter(Counter):
    """增强的文件计数器"""
    
    def total_lines(self, root: Path) -> 'FileCounter':
        """计算每个模块的总行数"""
        result = FileCounter()
        for path in root.rglob('*.py'):
            module = path.relative_to(root).parts[0]
            lines = len(path.read_text().splitlines())
            result[module] += lines
        return result
    
    def percentage(self) -> dict[str, float]:
        """计算百分比分布"""
        total = sum(self.values())
        return {k: v/total*100 for k, v in self.items()}
```

## 8. 测试策略

### 8.1 单元测试

```python
def test_counter_basic():
    """测试基本计数"""
    from collections import Counter
    
    items = ['a', 'b', 'a', 'c', 'a', 'b']
    counter = Counter(items)
    
    assert counter['a'] == 3
    assert counter['b'] == 2
    assert counter['c'] == 1
    assert counter['d'] == 0  # 缺失键返回 0

def test_counter_most_common():
    """测试最常见排序"""
    counter = Counter({'a': 3, 'b': 1, 'c': 2})
    
    assert counter.most_common() == [('a', 3), ('c', 2), ('b', 1)]
    assert counter.most_common(2) == [('a', 3), ('c', 2)]
```

### 8.2 集成测试

```python
def test_build_port_manifest_counter(tmp_path):
    """测试清单构建中的计数器"""
    # 创建测试结构
    (tmp_path / 'mod1').mkdir()
    (tmp_path / 'mod1' / 'a.py').write_text('')
    (tmp_path / 'mod1' / 'b.py').write_text('')
    (tmp_path / 'mod2').mkdir()
    (tmp_path / 'mod2' / 'c.py').write_text('')
    (tmp_path / 'root.py').write_text('')
    
    manifest = build_port_manifest(tmp_path)
    
    # 验证计数
    counts = {m.name: m.file_count for m in manifest.top_level_modules}
    assert counts.get('mod1') == 2
    assert counts.get('mod2') == 1
    assert counts.get('root.py') == 1
```

## 9. 与原始 Claude Code 对比

| 特性 | 原始 (TypeScript) | 本实现 (Python) |
|------|-------------------|-----------------|
| 计数器 | 可能使用 Map 或对象 | Counter |
| 排序 | 手动排序 | most_common() |
| 聚合 | 自定义实现 | Counter 运算 |
| 性能 | 取决于实现 | 优化过的 C 实现 |

原始代码可能使用 JavaScript 的 `Map` 或对象进行计数，手动实现排序和聚合。

## 10. 评价

| 维度 | 评分 | 说明 |
|