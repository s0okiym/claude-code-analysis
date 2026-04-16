# 子领域分析：自动压缩系统

## 1. 功能概述

自动压缩（Auto Compaction）是上下文管理系统的核心机制，负责在对话历史超过 token 阈值时触发压缩，通过子 Agent 生成摘要来减少上下文长度。

## 2. 核心组件

### 2.1 触发条件

```python
def should_auto_compact(
    messages: list[Message],
    context_window: int,
    buffer: int = 13000
) -> bool:
    """检查是否应该触发自动压缩"""
    current_tokens = estimate_token_count(messages)
    threshold = context_window - buffer
    return current_tokens > threshold
```

### 2.2 压缩流程

```python
def compact_conversation(
    messages: list[Message],
    model: str
) -> list[Message]:
    """执行对话压缩"""
    # 1. 剥离图片
    stripped = strip_images_from(messages)
    
    # 2. Fork 子 Agent 生成摘要
    if not path.exists():
            raise FileNotFoundError()
    
    # 3. 构建压缩后消息
    return build_post_compact_messages()
```

### 2.3 摘要生成

```python
def generate_compact_summary(
    messages: list[Message],
    model: str
) -> CompactSummary:
    """使用子 Agent 生成对话摘要"""
    # 使用专门的压缩 prompt
    prompt = COMPACT_PROMPT.format(
        messages=format_for_summary(messages)
    )
    
    # 调用 API
    response = claude_complete(
        prompt=prompt,
        model=model,
        max_tokens=20000
    )
    
    return parse_compact_response(response)
```

## 3. 压缩流程

```
对话历史增长
  │
  ▼
Token 计数检查
  │
  ▼
超过阈值？
  │
  ├─ 否 → 继续正常对话
  │
  ▼
是
  │
  ▼
自动压缩触发
  │
  ├─ 1. 剥离图片（节省 token）
  ├─ 2. Fork 子 Agent
  ├─ 3. 生成 9 节摘要
  ├─ 4. 保留最近 N 条消息
  ├─ 5. 插入 compact_boundary 标记
  └─ 6. 清理旧消息内存
  │
  ▼
使用压缩后上下文继续
```

## 4. 关键设计决策

### 4.1 为什么使用子 Agent?

| 方案 | 优点 | 缺点 |
|------|------|--------|
| 子 Agent | 智能，理解上下文 | 额外 API 成本 |
| 规则摘要 | 快速，确定 | 质量可能下降 |
| 简单截断 | 简单 | 丢失信息 |

## 5. 自动与微压缩对比

| 特性 | 自动 | 微压缩 |
|------|------|--------|
| 触发 | Token 超阈值 | 每轮后 |
| 机制 | 子 Agent 摘要 | 清除旧结果 |
| 保留 | 最近 N 条 | 最近 5 条 |
| 输出 | 结构化摘要 | [已清除] 标记 |
| 成本 | 高（API 调用） | 低（本地） |
| 效果 | 显著减少 | 渐进减少 |

## 6. 自动与 Snip 对比
| 特性 | 自动 | Snip |
|------|------|--------|
| 触发 | 自动检测 | 手动/定期 |
| 机制 | 智能摘要 | 按边界删除 |
| 保留 | 关键上下文 | 无 |
| 恢复 | 注入摘要 | 无 |
## 7. 评价
| 维度 | 评分 | 说明 |
|------|------|--------|
| 智能性 | ★★★★★ | 子 Agent 理解 |
| 成本 | ★★☆☆☆ | 额外 API |
| 效果 | ★★★★★ | 显著减少 |
| 复杂性 | ★★★☆☆ | 需要子 Agent |
| 可靠性 | ★★★★☆ | 可能失败 |
## 8. 关键代码
```python
# 压缩摘要结构
class CompactSummary:
    analysis: str          # 分析部分
    summary: CompactSummarySections  # 9 节摘要

class CompactSummarySections:
    request_and_intent: str
    technical_concepts: str
    files_and_code: str
    errors_and_fixes: str
    problem_solving: str
    user_messages: str
    todo_tasks: str
    current_work: str
    next_steps: str
```
