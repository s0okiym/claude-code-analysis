# 子领域分析：维护性系统

## 1. 功能概述
维护性确保代码易于修改和扩展。

## 2. 维护性实践
### 2.1 单一职责
```python
# 每个函数做一件事
def scan_files(): ...
def count_modules(): ...
def build_manifest(): ...
```
### 2.2 依赖注入
```python
class QueryEnginePort:
    def __init__(self, manifest: PortManifest):
        self.manifest = manifest
```
### 2.3 纯函数
```python
def summary_lines(backlog) -> list:
    """纯函数：无副作用，输出仅依赖输入"""
    return [f'- {m.name}' for m in backlog.modules]
```
### 2.4 类型安全
```python
def build_port_manifest(
    src_root: Path | None = None
) -> PortManifest:
    ...
```
## 3. 评价
| 维度 | 评分 | 说明 |
|------|--------|------|
| 修改 | ★★★★★ | 容易 |
| 理解 | ★★★★★ | 清晰 |
| 测试 | ★★★★☆ | 良好 |
| 扩展 | ★★★★☆ | 基础 |
## 4. 与原始对比
| 特性 | 原始 (TS) | 本实现 (Py) |
|------|-----------|-------------|
| 复杂度 | 高 | 低 |
| 耦合 | 紧 | 松 |
| 文档 | 完整 | 基础 |
| 测试 | 完整 | 基础 |
## 5. 代码片段
```python
# 易维护的代码
def process_data(data: Data) -> Result:
    """处理数据。
    
    步骤：
    1. 验证
    2. 转换
    3. 返回
    """
    # 验证
    if not validate(data):
        raise ValueError('Invalid data')
    
    # 转换
    transformed = transform(data)
    
    # 返回
    return Result(transformed)
```
