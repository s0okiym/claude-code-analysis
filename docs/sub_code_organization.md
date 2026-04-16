# 子领域分析：代码组织系统

## 1. 功能概述
代码组织定义了文件结构、模块划分和命名约定。

## 2. 目录结构
```
claude-code/
├── src/                    # 源代码
│   ├── __init__.py        # 包入口
│   ├── main.py            # CLI 入口
│   ├── models.py          # 数据模型
│   ├── port_manifest.py   # 清单生成
│   ├── query_engine.py    # 查询引擎
│   ├── commands.py        # 命令数据
│   ├── tools.py           # 工具数据
│   └── task.py            # 任务模型
│
├── tests/                  # 测试
│   └── (测试文件)
│
├── kimi_docs/             # 文档
│   ├── total.md
│   └── sub_*.md
│
├── README.md              # 项目说明
├── cc.md                  # 架构文档
├── context.md             # 上下文文档
└── .gitignore           # Git 忽略
```

## 3. 命名约定

### 3.1 文件命名
| 类型 | 示例 | 说明 |
|------|------|------|
| 模块 | `main.py` | snake_case |
| 包 | `src/` | 小写 |
| 测试 | `test_*.py` | test_前缀 |
| 文档 | `*.md` | 小写 |

### 3.2 代码命名
| 类型 | 示例 | 说明 |
|------|------|------|
| 类 | `PortManifest` | PascalCase |
| 函数 | `build_manifest` | snake_case |
| 常量 | `PORTED_COMMANDS` | UPPER_SNAKE |
| 变量 | `manifest` | snake_case |
| 私有 | `_helper` | 下划线前缀 |

## 4. 模块划分
### 4.1 按职责划分
```
src/
├── main.py          # 入口：CLI
├── models.py        # 数据：模型
├── port_manifest.py  # 功能：清单
├── query_engine.py   # 功能：查询
├── commands.py      # 数据：命令
├── tools.py         # 数据：工具
└── task.py          # 数据：任务
```

### 4.2 依赖关系
```
main.py
  ├── port_manifest.py
  │   └── models.py
  ├── query_engine.py
  │   ├── port_manifest.py
  │   ├── commands.py
  │   │   └── models.py
  │   └── tools.py
  │       └── models.py
  └── models.py (通过其他模块)
```

## 5. 关键设计
### 5.1 为什么扁平结构？
code
| 优点 | 说明 |
|--------|--------|
| 简单 | 易于理解 |
| 导航 | 快速定位 |
| 导入 | 路径短 |
### 5.2 为什么 models 单独？
- 共享数据
- 避免循环
- 统一类型
### 5.3 为什么 tests 分离？
- 生产代码干净
- 测试可独立运行
- 包大小
## 6. 评价
| 维度 | 评分 | 说明 |
|------|--------|------|
| 清晰 | ★★★★★ | 结构明确 |
| 可维护 | ★★★★★ | 简单 |
| 扩展 | ★★★☆☆ | 需重构 |
| 一致 | ★★★★★ | 统一 |
## 7. 与原始对比
| 特性 | 原始 (TS) | 本实现 (Py) |
|------|-----------|-------------|
| 规模 | 大 | 小 |
| 深度 | 深 | 扁平 |
| 模块 | 多 | 少 |
| 组织 | 复杂 | 简单 |
## 8. 代码片段
```python
# 文件模板
""""模块描述。

详细说明...
"""
from __future__ import annotations

# 导入
from .models import SomeClass

# 常量
DEFAULT_VALUE = 10

# 类
class MyClass:
    """类描述"""
    pass

# 函数
def public_function():
    """函数描述"""
    pass

def _private_helper():
    """私有辅助函数"""
    pass
```
