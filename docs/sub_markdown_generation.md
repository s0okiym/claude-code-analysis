# 子领域分析：Markdown 生成系统

## 1. 功能概述

Markdown 生成系统负责将结构化数据转换为 Markdown 格式文本，用于生成移植工作空间的摘要报告。

## 2. 核心组件

### 2.1 PortManifest.to_markdown()

```python
def to_markdown(self) -> str:
    lines = [
        f'Port root: `{self.src_root}`',
        f'Total Python files: **{self.total_python_files}**',
        '',
        'Top-level Python modules:',
    ]
    for module in self.top_level_modules:
        lines.append(f'- `{module.name}` ({module.file_count} files) — {module.notes}')
    return '\n'.join(lines)
```

### 2.2 QueryEnginePort.render_summary()

```python
def render_summary(self) -> str:
    command_backlog = build_command_backlog()
    tool_backlog = build_tool_backlog()
    
    sections = [
        '# Python Porting Workspace Summary',
        '',
        self.manifest.to_markdown(),
        '',
        f'{command_backlog.title}:',
        *command_backlog.summary_lines(),
        '',
        f'{tool_backlog.title}:',
        *tool_backlog.summary_lines(),
    ]
    return '\n'.join(sections)
```

### 2.3 PortingBacklog.summary_lines()

```python
def summary_lines(self) -> list[str]:
    return [
        f'- {module.name} [{module.status}] — {module.responsibility} (from {module.source_hint})'
        for module in self.modules
    ]
```

## 3. Markdown 语法使用

### 3.1 标题

```python
'# Python Porting Workspace Summary'  # H1
'## Section'                          # H2（未使用）
```

### 3.2 强调

```python
f'**{self.total_python_files}**'  # 粗体
f'`{self.src_root}`'              # 行内代码
```

### 3.3 列表

```python
# 无序列表
f'- `{module.name}` ({module.file_count} files) — {module.notes}'

# 生成结果
# - `main.py` (1 files) — CLI entrypoint
```

### 3.4 空行

```python
''  # 空行，用于段落分隔
```

## 4. 实现原理

### 4.1 字符串拼接策略

```python
# 策略 1：列表 + join（当前使用）
lines = ['line1', 'line2', 'line3']
result = '\n'.join(lines)

# 策略 2：累加（不推荐）
result = 'line1'
result += '\n' + 'line2'
result += '\n' + 'line3'

# 策略 3：f-string 多行
result = f'''line1
line2
line3'''

# 策略 4：io.StringIO
import io
buf = io.StringIO()
buf.write('line1\n')
buf.write('line2\n')
result = buf.getvalue()
```

**选择理由**:
- 列表 + join：清晰，易于修改，性能可接受
- 避免累加：多次创建字符串对象
- 避免 f-string 多行：缩进处理麻烦

### 4.2 列表展开操作符

```python
sections = [
    'header',
    *list_items,  # 展开操作符，将列表元素逐个插入
    'footer',
]
```

**等价于**:
```python
sections = ['header']
sections.extend(list_items)
sections.append('footer')
```

### 4.3 生成器表达式

```python
lines = [
    f'- {module.name}...'
    for module in self.modules
]
```

**优势**:
- 惰性求值（列表推导式是立即求值）
- 内存友好
- 简洁

## 5. 设计决策

### 5.1 为什么使用列表 + join?

| 方案 | 优点 | 缺点 |
|------|------|------|
| 列表 + join | 清晰，可修改，O(n) | 需要额外列表 |
| 累加 += | 直观 | O(n²)，创建多个中间字符串 |
| f-string | 简洁 | 多行缩进麻烦 |
| StringIO | 适合大量写入 | 稍繁琐 |

**选择理由**:
- 代码清晰，易于理解
- 数据量小（几十行），性能差异可忽略
- 便于动态添加/修改行

### 5.2 为什么使用 * 展开操作符?

```python
sections = [
    f'{command_backlog.title}:',
    *command_backlog.summary_lines(),  # 展开
]
```

**对比**:
```python
# 不使用展开
sections = [f'{command_backlog.title}:']
sections.extend(command_backlog.summary_lines())

# 使用展开（更简洁）
sections = [
    f'{command_backlog.title}:',
    *command_backlog.summary_lines(),
]
```

**选择理由**:
- 声明式风格，结构清晰
- 减少临时变量
- 现代 Python 特性（3.5+）

### 5.3 为什么每个类负责自己的 Markdown?

```python
# 方案 1：每个类负责（当前）
class PortManifest:
    def to_markdown(self) -> str: ...

# 方案 2：集中式渲染器
class MarkdownRenderer:
    def render_manifest(self, manifest: PortManifest) -> str: ...
    def render_backlog(self, backlog: PortingBacklog) -> str: ...
```

**选择理由**:
- 封装：每个类知道自己的结构
- 内聚：数据与渲染逻辑在一起
- 简单：当前场景不需要复杂渲染器

**未来可能需要分离**:
- 如果支持多种输出格式（JSON, YAML, HTML）
- 如果渲染逻辑变得复杂

## 6. 扩展性分析

### 6.1 支持多种输出格式

```python
from enum import Enum

class OutputFormat(Enum):
    MARKDOWN = 'md'
    JSON = 'json'
    YAML = 'yaml'

class PortManifest:
    def to_format(self, fmt: OutputFormat) -> str:
        if fmt == OutputFormat.MARKDOWN:
            return self.to_markdown()
        elif fmt == OutputFormat.JSON:
            import json
            return json.dumps({
                'src_root': str(self.src_root),
                'total_python_files': self.total_python_files,
                'modules': [
                    {'name': m.name, 'files': m.file_count}
                    for m in self.top_level_modules
                ]
            }, indent=2)
        ...
```

### 6.2 模板系统

```python
from string import Template

MANIFEST_TEMPLATE = Template('''
Port root: `$src_root`
Total Python files: **$total_files**

Top-level Python modules:
$modules
''')

def to_markdown_with_template(self) -> str:
    modules_md = '\n'.join(
        f'- `{m.name}` ({m.file_count} files) — {m.notes}'
        for m in self.top_level_modules
    )
    
    return MANIFEST_TEMPLATE.substitute(
        src_root=self.src_root,
        total_files=self.total_python_files,
        modules=modules_md
    )
```

### 6.3 Markdown 构建器

```python
class MarkdownBuilder:
    """流式 Markdown 构建器"""
    
    def __init__(self):
        self.lines = []
    
    def heading(self, text: str, level: int = 1):
        self.lines.append(f'{"#" * level} {text}')
        return self
    
    def paragraph(self, text: str):
        self.lines.append(text)
        return self
    
    def bullet(self, text: str):
        self.lines.append(f'- {text}')
        return self
    
    def code(self, text: str):
        self.lines.append(f'`{text}`')
        return self
    
    def bold(self, text: str):
        self.lines.append(f'**{text}**')
        return self
    
    def newline(self):
        self.lines.append('')
        return self
    
    def build(self) -> str:
        return '\n'.join(self.lines)

# 使用
md = MarkdownBuilder()
md.heading('Summary').newline()
md.paragraph(f'Port root: {md.code(str(root))}')
md.paragraph(f'Total: {md.bold("8")} files')
result = md.build()
```

## 7. 存在的问题

### 7.1 当前限制

1. **硬编码格式**: 无法自定义输出格式
2. **无转义**: 如果数据包含 Markdown 特殊字符，可能破坏格式
3. **无验证**: 不验证生成的 Markdown 是否有效
4. **单一格式**: 仅支持 Markdown

### 7.2 边界情况

```python
# 特殊字符
text = 'Use **bold** here'  # 会被解释为 Markdown
# 输出: Use **bold** here（bold 实际加粗）

# 空数据
PortingBacklog(title='Empty', modules=[]).summary_lines()
# 返回: []（空列表，可能不符合预期）

# 多行内容
notes = 'Line 1\nLine 2'
# 会破坏列表格式
```

### 7.3 改进建议

1. 添加转义：

```python
def escape_markdown(text: str) -> str:
    """转义 Markdown 特殊字符"""
    chars = ['\\', '`', '*', '_', '{', '}', '[', ']', '(', ')', '#', '+', '-', '.', '!']
    for char in chars:
        text = text.replace(char, f'\\{char}')
    return text
```

2. 添加验证：

```python
import re

def validate_markdown(md: str) -> bool:
    """基础 Markdown 验证"""
    # 检查未闭合的代码块
    if md.count('```') % 2 != 0:
        return False
    # 检查未闭合的粗体/斜体
    if md.count('**') % 2 != 0:
        return False
    return True
```

3. 支持自定义模板：

```python
class PortManifest:
    def to_markdown(self, template: str | None = None) -> str:
        if template:
            return Template(template).substitute(...)
        return self._default_markdown()
```

## 8. 测试策略

### 8.1 单元测试

```python
def test_port_manifest_to_markdown():
    """测试 Markdown 输出格式"""
    manifest = PortManifest(
        src_root=Path('/test/src'),
        total_python_files=3,
        top_level_modules=(
            Subsystem('main.py', 'src/main.py', 1, 'entry'),
            Subsystem('utils', 'src/utils', 2, 'helpers'),
        )
    )
    
    md = manifest.to_markdown()
    
    assert 'Port root: `/test/src`' in md
    assert 'Total Python files: **3**' in md
    assert '- `main.py` (1 files) — entry' in md
    assert '- `utils` (2 files) — helpers' in md

def test_summary_lines():
    """测试摘要行生成"""
