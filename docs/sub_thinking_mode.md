# 子领域分析：Thinking 模式系统

## 1. 功能概述
Thinking 模式允许 Claude 在回复前进行更深入的推理，通过 extended thinking 提高复杂任务质量。

## 2. 核心组件

### 2.1 Thinking 配置
```python
class ThinkingConfig:
    """Thinking 模式配置"""
    enabled: bool           # 是否启用
    type: ThinkingType    # 类型：enabled/adaptive
    budget: int          # token 预算
    
    def to_api_param(self) -> dict:
        """转换为 API 参数"""
        if self.type == 'adaptive':
            return {'thinking': {'type': 'adaptive'}}
        return {'thinking': {'budget_tokens': self.budget}}
```

### 2.2 Adaptive Thinking
```python
def should_enable_thinking(
    prompt: str,
    model: str
) -> bool:
    """自适应决定是否启用 thinking"""
    # 启发式判断
    if is_complex_task(prompt):
        return True
    if model in HIGH_REASONING_MODELS:
        return True
    return False

def is_complex_task(prompt: str) -> bool:
    """判断任务复杂度"""
    indicators = [
        'refactor',
        'architect',
        'design',
        'analyze',
        'compare',
    ]
    return any(i in prompt.lower() for i in indicators)
```
### 2.3 Thinking 块处理
```python
class MessageBuilder:
    def add_thinking(self, content: str):
        """添加 thinking 块到消息"""
        self.blocks.append({
            'type': 'thinking',
            'thinking': content
        })
    
    def add_response(self, content: str):
        """添加正常响应"""
        self.blocks.append({
            'type': 'text',
            'text': content
        })
```
## 3. Thinking 流程
```
用户输入
  │
  ▼
┌─────────────────────┐
│ 判断是否启用 thinking │
│ - adaptive 模式？    │
│ - 任务复杂度？       │
│ - 模型能力？         │
└──────────┬──────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
  启用          禁用
     │           │
     ▼           ▼
┌─────────┐  ┌─────────┐
│ Extended │  │ Normal  │
│ Thinking │  │ Response│
│          │  │         │
│ 推理过程 │  │ 直接回复│
│ (可见) │  │         │
└────┬────┘  └─────────┘
       │
       ▼
┌─────────────┐
│ 最终响应      │
│ (基于 thinking)│
└─────────────┘
```
## 4. 设计决策
### 4.1 为什么支持 adaptive?
| 模式 | 优点 | 缺点 |
|------|------|------|
| enabled | 确定 | 可能过度使用 |
| adaptive | 智能 | 可能误判 |
| disabled | 快速 | 缺少推理 |

**选择 adaptive 理由**:
- 自动优化成本和延迟
- 只在需要时启用
- 用户无需手动配置
### 4.2 为什么显示 thinking?
```python
# 选项 1：显示（当前）
def build_response(self):
    if self.thinking:
        print(f"Thinking: {thinking}")
    return final_response

# 选项 2：隐藏
def build_response_hidden(self):
    if self.thinking:
        # 不显示，只用于内部
        return final_response
```

**显示的理由**:
- 透明度：用户知道 AI 在"思考"- 调试：可以看到推理过程- 学习：用户可学习推理模式
### 4.3 为什么设置 budget?
| budget | 效果 | 适用 |
|--------|--------|------|
| 低 (1K) | 快速推理 | 简单任务 |
| 中 (4K) | 平衡 | 一般任务 |
| 高 (16K+) | 深入 | 复杂任务 |

## 5. 评价
| 维度 | 评分 | 说明 |
|--------|------|------|
| 质量提升 | ★★★★★ | 复杂任务显著改善 |
| 成本 | ★★★☆☆ | 额外 token |
| 速度 | ★★★☆☆ | thinking 增加延迟 |
| 用户体验 | ★★★★★ | 透明 |
## 6. 关键代码
```python
# 配置 thinking
def configure_thinking():
    config = ThinkingConfig(
        enabled=True,
        type='adaptive',
        budget=4000
    )
    return config.to_api_param()

# 处理 thinking 块
def process_chunk(chunk):
    if chunk.type == 'thinking':
        display_thinking(chunk.content)
    elif chunk.type == 'text':
        display_response(chunk.content)
```
## 7. 与原始对比
| 特性 | 原始 (TS) | 本实现 (Py) |
|------|-------------|---------------|
| 模式 | 3 种 | 概念 |
| 预算 | 动态调整 | 固定 |
| 显示 | 可配置 | 概念 |
| 块处理 | 完整 | 简化 |
## 8. 代码片段
```python
# 配置 thinking
def configure_thinking_mode():
    return {
        'thinking': {
            'type': 'adaptive',
            'budget_tokens': 4000
        }
    }

# 显示 thinking
def display_thinking(content):
    print(f" Thinking: {content[:100]}...")
```
