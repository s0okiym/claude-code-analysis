# 子领域分析：项目结构系统

## 1. 功能概述
项目结构定义了文件和目录的组织方式。

## 2. 完整结构
```
claude-code/
├── src/                      # 源代码
│   ├── __init__.py
│   ├── main.py
│   ├── models.py
│   ├── port_manifest.py
│   ├── query_engine.py
│   ├── commands.py
│   ├── tools.py
│   └── task.py
│
├── tests/                    # 测试
│   └── __init__.py
│
├── kimi_docs/               # 文档
│   ├── total.md
│   └── sub_*.md
│
├── assets/                   # 资源
│   └── omx/
│
├── README.md                # 说明
├── cc.md                    # 架构
├── context.md               # 上下文
├── .gitignore              # Git
└── 2026-03-09-essay.md     # 文章
```

## 3. 文件说明
| 文件 | 用途 | 大小 |
|------|------|------|
| main.py | CLI 入口 | ~40 行 |
| models.py | 数据类 | ~40 行 |
| port_manifest.py | 清单 | ~50 行 |
| query_engine.py | 引擎 | ~35 行 |
| commands.py | 命令 | ~20 行 |
| tools.py | 工具 | ~20 行 |
| task.py | 任务 | ~10 行 |

## 4. 设计原则
### 4.1 为什么扁平？
- 简单
- 易导航
- 适合小规模
### 4.2 为什么分离？
- 关注点分离
- 测试独立
- 文档独立
### 4.3 为什么文档丰富？
- 理解架构
- 便于维护
- 知识传承
## 5. 评价
| 维度 | 评分 | 说明 |
|------|--------|------|
| 清晰 | ★★★★★ | 明确 |
| 完整 | ★★★★☆ | 良好 |
| 可扩展 | ★★★☆☆ | 需调整 |
## 6. 与原始对比
| 特性 | 原始 (TS) | 本实现 (Py) |
|--------|-----------|-------------|
| 规模 | 大 | 小 |
| 结构 | 复杂 | 简单 |
| 文档 | 生成 | 手动 |
| 测试 | 完整 | 基础 |
## 7. 代码片段
```python
# 项目模板
PROJECT_TEMPLATE = '''
{project_name}/
├── src/
│   └── __init__.py
├── tests/
│   └── __init__.py
├── docs/
│   └── README.md
└── .gitignore
'''

def create_project(name: str):
    Path(name).mkdir()
    (Path(name) / 'src').mkdir()
    ...
```
