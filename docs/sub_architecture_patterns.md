# 子领域分析：架构模式系统

## 1. 功能概述
架构模式定义了代码的组织原则和交互方式。

## 2. 使用的模式

### 2.1 协调器模式
```python
class QueryEnginePort:
    """协调器：整合多个数据源"""
    
    def render_summary(self) -> str:
        # 协调多个模块
        manifest_md = self.manifest.to_markdown()
        commands = build_command_backlog()
        tools = build_tool_backlog()
        
        # 组合输出
        return '\n'.join([
            manifest_md,
            *commands.summary_lines(),
            *tools.summary_lines(),
        ])
```

### 2.2 工厂模式
```python
def build_port_manifest(src_root: Path | None = None) -> PortManifest:
    """工厂：创建 PortManifest"""
    files = scan_files(src_root)
    counter = count_by_module(files)
    return PortManifest(...)

def build_command_backlog() -> PortingBacklog:
    """工厂：创建命令清单"""
    return PortingBacklog(title='...', modules=list(PORTED_COMMANDS))
```

### 2.3 数据类模式
```python
@dataclass(frozen=True)
class Subsystem:
    """不可变数据对象"""
    name: str
    file_count: int
```

### 2.4 纯函数模式
```python
def summary_lines(backlog: PortingBacklog) -> list[str]:
    """纯函数：无副作用"""
    return [f'- {m.name}' for m in backlog.modules]
```

## 3. 架构层次
```
┌─────────────────────────────────────┐
│  Presentation Layer                 │
│  - main.py (CLI)                    │
├─────────────────────────────────────┤
│  Application Layer                  │
│  - query_engine.py (协调)           │
├─────────────────────────────────────┤
│  Domain Layer                       │
│  - port_manifest.py (清单)          │
│  - commands.py (命令)               │
│  - tools.py (工具)                  │
├─────────────────────────────────────┤
│  Data Layer                         │
│  - models.py (数据类)               │
└─────────────────────────────────────┘
```

## 4. 设计原则
### 4.1 SOLID 应用
| 原则 | 应用 |
|------|------|
| SRP | 每个模块单一职责 |
| OCP | 扩展通过新模块 |
| LSP | dataclass 替换 |
| ISP | 小接口 |
| DIP | 依赖抽象 |
### 4.2 函数式风格
- 纯函数优先
- 不可变数据
- 无副作用
### 4.3 分层架构
- 清晰的层次
- 单向依赖
- 接口隔离
## 5. 评价
| 维度 | 评分 | 说明 |
|------|--------|------|
| 清晰 | ★★★★★ | 结构明确 |
| 可测试 | ★★★★★ | 易于测试 |
| 可扩展 | ★★★★☆ | 基础良好 |
| 复杂度 | ★★☆☆☆ | 简单 |
## 6. 与原始对比
| 特性 | 原始 (TS) | 本实现 (Py) |
|------|-----------|-------------|
| 模式 | 复杂 | 简单 |
| 层次 | 多层 | 基础 |
| 交互 | 复杂 | 简单 |
| 状态 | 有 | 无 |
## 7. 代码片段
```python
# 协调器模式
class Coordinator:
    def __init__(self, *services):
        self.services = services
    
    def execute(self, task):
        results = [s.process(task) for s in self.services]
        return self.combine(results)

# 工厂模式
class Factory:
    @staticmethod
    def create(type_: str):
        creators = {
            'a': ClassA,
            'b': ClassB,
        }
        return creators[type_]()
```
