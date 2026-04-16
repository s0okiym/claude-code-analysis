# 子领域分析：导入系统

## 1. 功能概述
导入系统管理模块间的依赖关系，使用相对导入和绝对导入组织代码。

## 2. 核心用法

### 2.1 相对导入
```python
# src/query_engine.py
from __future__ import annotations

from .commands import build_command_backlog      # 同级模块
from .port_manifest import PortManifest          # 同级模块
from .tools import build_tool_backlog            # 同级模块
```

### 2.2 绝对导入
```python
# 外部库
from pathlib import Path
from dataclasses import dataclass
from collections import Counter
```

### 2.3 包导入
```python
# src/__init__.py
from .commands import PORTED_COMMANDS
from .port_manifest import PortManifest
from .query_engine import QueryEnginePort
```

## 3. 导入结构
```
src/
├── __init__.py          # 导出公共 API
│   └── from .commands import ...
│
├── main.py              # CLI 入口
│   └── from .port_manifest import ...
│
├── port_manifest.py     # 清单生成
│   └── from .models import Subsystem
│
├── query_engine.py      # 查询引擎
│   └── from .commands import ...
│
├── commands.py          # 命令数据
│   └── from .models import PortingBacklog
│
├── tools.py             # 工具数据
│   └── from .models import PortingBacklog
│
├── models.py            # 数据模型
│   └── from dataclasses import dataclass
│
└── task.py              # 任务模型
    └── from dataclasses import dataclass
```

## 4. 关键设计
### 4.1 为什么相对导入？
| 优点 | 说明 |
|------|------|
| 可移动 | 包可重命名 |
| 清晰 | 显示内部依赖 |
| 简洁 | 短路径 |

### 4.2 为什么 __future__ annotations?
```python
from __future__ import annotations
```
**优点**:
- 延迟求值
- 前向引用
- 新语法
### 4.3 循环导入处理
```python
# 当前无循环导入
# 如果出现：

# 方案 1：延迟导入
def function():
    from .other import something  # 函数内导入
    ...

# 方案 2：TYPE_CHECKING
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from .other import Something
```
## 5. 评价
| 维度 | 评分 | 说明 |
|------|--------|------|
| 清晰 | ★★★★★ | 结构清晰 |
| 无循环 | ★★★★★ | 无问题 |
| 标准 | ★★★★★ | 符合规范 |
| 可维护 | ★★★★★ | 简单 |
## 6. 与原始对比
| 特性 | 原始 (TS) | 本实现 (Py) |
|------|-----------|-------------|
| 方式 | ES import | import |
| 相对 | 支持 | 支持 |
| 动态 | 有限 | 灵活 |
| 循环 | 有处理 | 简单 |
## 7. 代码片段
```python
# 最佳实践
from __future__ import annotations

# 标准库
import os
from pathlib import Path

# 第三方
# (无)

# 本地
from .models import MyClass
from .utils import helper
```
