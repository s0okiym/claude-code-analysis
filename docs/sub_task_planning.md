# 子领域分析：任务级规划系统

## 1. 功能概述

Task 系统定义了移植工作空间中的任务级规划数据结构，用于更细粒度的任务跟踪。当前仅定义数据类，未在核心流程中使用。

## 2. 核心组件

### 2.1 数据类定义

```python
from __future__ import annotations
from dataclasses import dataclass

@dataclass(frozen=True)
class PortingTask:
    """Task-level planning structure for the porting workspace."""
    title: str       # 任务标题
    detail: str      # 任务详情
    completed: bool = False  # 是否完成
```

### 2.2 与 PortingModule 对比

| 特性 | PortingTask | PortingModule |
|------|-------------|---------------|
| 粒度 | 单个任务 | 功能模块 |
| 用途 | 细粒度跟踪 | 粗粒度模块跟踪 |
| 字段 | title, detail, completed | name, responsibility, source_hint, status |
| 使用状态 | 未使用 | 已使用 |
| 关系 | 可能属于某模块 | 独立功能单元 |

## 3. 设计意图

### 3.1 为什么存在未使用代码？

**可能原因**:
1. **未来扩展**: 为后续任务管理预留
2. **渐进开发**: 先定义接口，再实现功能
3. **原型验证**: 快速原型阶段

### 3.2 代码位置

```
src/
  ├─ task.py          # 定义 PortingTask
  ├─ main.py          # 未使用
  ├─ query_engine.py  # 未使用
  └─ __init__.py      # 未导出
```

## 4. 扩展性分析

### 4.1 集成到现有系统

```python
def build_port_manifest() -> PortManifest:
    """构建清单，带任务跟踪"""
    tasks = [
        PortingTask('Scan files', 'Find all Python files', False),
        PortingTask('Build manifest', 'Create manifest object', False),
    ]
    
    # 执行任务
    files = list(root.rglob('*.py'))
    tasks[0].completed = True  # 标记完成
    
    manifest = PortManifest(...)
    tasks[1].completed = True
    
    return manifest
```

### 4.2 完整任务系统

```python
@dataclass
class TaskManager:
    """任务管理器"""
    tasks: list[PortingTask] = field(default_factory=list)
    
    def add_task(self, title: str, detail: str) -> None:
        self.tasks.append(PortingTask(title, detail))
    
    def complete_task(self, title: str) -> None:
        for task in self.tasks:
            if task.title == title:
                # 需要重建（frozen）
                idx = self.tasks.index(task)
                self.tasks[idx] = PortingTask(title, task.detail, True)
    
    def get_progress(self) -> float:
        if not self.tasks:
            return 0.0
        completed = sum(1 for t in self.tasks if t.completed)
        return completed / len(self.tasks)
```

## 5. 与原始 Claude Code 对比

| 特性 | 原始 (TypeScript) | 本实现 (Python) |
|------|-------------------|-----------------|
| 任务系统 | 复杂（TodoTool, TaskManager） | 仅数据类 |
| 任务类型 | 多种（用户任务、Agent 任务） | 单一类型 |
| 状态管理 | 完整生命周期 | 布尔值 |
| 持久化 | 有 | 无 |

原始代码有完整的任务管理系统，本实现仅为占位。

## 6. 评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 简洁性 | ★★★★★ | 简单清晰 |
| 扩展性 | ★★★★☆ | 预留接口 |
| 完整性 | ★★☆☆☆ | 仅数据类 |
| 使用度 | ★☆☆☆☆ | 未使用 |

## 7. 关键代码片段

### 7.1 完整实现

```python
from __future__ import annotations
from dataclasses import dataclass


@dataclass(frozen=True)
class PortingTask:
    title: str
    detail: str
    completed: bool = False
```

### 7.2 使用示例

```python
# 创建任务
task = PortingTask(
    title='Implement file scanner',
    detail='Scan all Python files in src/',
    completed=False
)

# 函数式更新（frozen）
completed_task = PortingTask(
    title=task.title,
    detail=task.detail,
    completed=True
)
```

## 8. 总结

PortingTask 是一个预留的数据结构，当前未在核心流程中使用。它为未来的任务跟踪功能提供了基础接口，体现了渐进式开发的设计思路。
