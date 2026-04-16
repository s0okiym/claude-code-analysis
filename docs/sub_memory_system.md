# 子领域分析：记忆系统

## 1. 功能概述
记忆系统允许 Claude Code 跨会话保持信息，包括用户偏好、项目决策和反馈。

## 2. 核心组件

### 2.1 记忆目录结构
```
~/.claude/
└── projects/
    └── <project-slug>/
        └── memory/
            ├── MEMORY.md           # 入口索引
            ├── user_preferences.md # 用户偏好
            ├── feedback.md         # 反馈记录
            ├── architecture.md     # 架构决策
            └── api_reference.md    # API 参考
```

### 2.2 记忆类型
```python
class MemoryType(Enum):
    USER = 'user'           # 用户偏好
    FEEDBACK = 'feedback'   # 反馈和纠正
    PROJECT = 'project'     # 项目知识
    REFERENCE = 'reference' # 外部参考

@dataclass
class Memory:
    type: MemoryType
    content: str
    created_at: datetime
    updated_at: datetime
```

### 2.3 记忆注入
```python
def load_memory_prompt() -> str:
    """加载记忆到系统提示词"""
    memory_dir = get_memory_dir()
    
    # 读取 MEMORY.md
    index = read_memory_file(memory_dir / 'MEMORY.md')
    
    # 读取相关主题文件
    relevant = find_relevant_memories(query)
    
    return format_memory_prompt(index, relevant)

def find_relevant_memories(query: str) -> list[Memory]:
    """语义搜索相关记忆"""
    all_memories = load_all_memories()
    
    # 简单关键词匹配（未来可用向量搜索）
    return [
        m for m in all_memories
        if any(kw in m.content for kw in extract_keywords(query))
    ][:5]  # 最多 5 个
```

## 3. 记忆生命周期
```
对话开始
  │
  ▼
加载 MEMORY.md
  │
  ▼
语义搜索相关记忆
  │
  ▼
注入到系统提示词
  │
  ▼
对话进行中
  │
  ▼
重要信息 → 写入记忆
  │
  ▼
对话结束
  │
  ▼
更新记忆文件
```

## 4. 关键设计
### 4.1 为什么分层存储？
| 层次 | 位置 | 用途 |
|------|------|------|
| Managed | /etc/claude-code/ | 企业全局 |
| User | ~/.claude/ | 用户个人 |
| Project | ./CLAUDE.md | 项目特定 |
| Local | ./CLAUDE.local.md | 私人（不提交） |

**优点**:
- 灵活性：不同范围不同策略
- 隐私：Local 不提交到 Git
- 共享：Project 可团队共享
### 4.2 为什么限制大小？
```python
MAX_MEMORY_SIZE = 25000  # 25KB
MAX_MEMORY_LINES = 200   # 200 行
```

**理由**:
- 控制 token 消耗
- 保持相关性
- 避免信息过载
### 4.3 为什么语义搜索？
| 方法 | 优点 | 缺点 |
|------|------|------|
| 关键词 | 快速 | 不准确 |
| 向量 | 准确 | 需要模型 |
| 全文 | 平衡 | 复杂 |

**当前选择**：简单关键词（原型阶段）
**未来**：向量搜索（如需要）

## 5. 评价
| 维度 | 评分 | 说明 |
|--------|------|------|
| 连续性 | ★★★★★ | 跨会话保持 |
| 智能性 | ★★★☆☆ | 简单关键词 |
| 隐私 | ★★★★☆ | 本地存储 |
| 可管理 | ★★★★☆ | 文件形式 |
| 准确性 | ★★★☆☆ | 依赖搜索质量 |
## 6. 关键代码
```python
# 记忆文件格式
MEMORY_TEMPLATE = """# Project Memory

## User Preferences
- Use Python 3.10+
- Prefer pathlib over os.path

## Architecture Decisions
- Use dataclasses for models
- Use argparse for CLI

## API References
- Internal API: src/
- Tests: tests/
"""

# 写入记忆
def write_memory(content: str, memory_type: MemoryType):
    memory_dir = get_memory_dir()
    file_path = memory_dir / f'{memory_type.value}.md'
    
    with open(file_path, 'a') as f:
        f.write(f'\n## {datetime.now()}\n')
        f.write(content)
```
## 7. 与原始对比
| 特性 | 原始 (TS) | 本实现 (Py) |
|------|-------------|---------------|
| 存储 | 文件 + 向量 | 仅文件 |
| 搜索 | 语义向量 | 关键词 |
| 层次 | 4 层 | 概念 |
| 自动写入 | 支持 | 手动 |
| 大小限制 | 有 | 概念 |
## 8. 代码片段
```python
# 加载记忆
def load_memory():
    memory_file = Path('~/.claude/memory/MEMORY.md').expanduser()
    if memory_file.exists():
        return memory_file.read_text()
    return ''

# 更新记忆
def update_memory(key: str, value: str):
    memory = load_memory()
    # 更新或追加
    ...
```
