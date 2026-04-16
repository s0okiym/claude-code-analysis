# 子领域分析：错误处理系统

## 1. 功能概述
错误处理系统管理程序中的异常情况，提供优雅降级、重试和用户反馈。

## 2. 核心组件

### 2.1 错误分类
```python
class ClaudeError(Exception):
    """基础错误"""
    pass

class APIError(ClaudeError):
    """API 调用错误"""
    def __init__(self, message: str, status_code: int):
        super().__init__(message)
        self.status_code = status_code

class ToolError(ClaudeError):
    """工具执行错误"""
    def __init__(self, message: str, tool_name: str):
        super().__init__(message)
        self.tool_name = tool_name

class PermissionError(ClaudeError):
    """权限错误"""
    pass
```

### 2.2 错误处理装饰器
```python
from functools import wraps
from typing import Callable

def handle_errors(f: Callable) -> Callable:
    """错误处理装饰器"""
    @wraps(f)
    def wrapper(*args, **kwargs):
        try:
            return f(*args, **kwargs)
        except APIError as e:
            print(f"API Error: {e}")
            return None
        except ToolError as e:
            print(f"Tool Error: {e.tool_name}: {e}")
            return None
        except Exception as e:
            print(f"Unexpected: {e}")
            raise
    return wrapper
```

### 2.3 重试机制
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10)
)
def call_with_retry(func: Callable, *args):
    """带重试的调用"""
    return func(*args)

def call_api_with_retry(prompt: str) -> str:
    """API 调用带重试"""
    return call_with_retry(call_claude_api, prompt)
```

## 3. 错误处理流程
```
操作执行
  │
  ├─ 成功
  │   └─ 返回结果
  │
  └─ 失败
      │
      ├─ API 错误
      │   ├─ 429: 等待重试
      │   ├─ 500: 记录并退出
      │   └─ 其他: 包装抛出
      │
      ├─ 工具错误
      │   ├─ 权限: 询问用户
      │   ├─ 未找到: 提示检查
      │   └─ 其他: 记录并继续
      │
      └─ 未知错误
          └─ 记录并抛出
```

## 4. 关键设计
### 4.1 为什么自定义异常？
| 方案 | 优点 | 缺点 |
|------|------|------|
| 自定义 | 语义清晰 | 需要定义 |
| 标准异常 | 无需定义 | 语义不清 |
| 两者结合 | 灵活 | 复杂 |

**选择理由**:
- 业务语义明确
- 可附加元数据
- 统一处理入口
### 4.2 为什么使用装饰器？
```python
@handle_errors
def process_tool(tool):
    return tool.run()

# 等价于
process_tool = handle_errors(process_tool)
```

**优点**:
- 复用处理逻辑
- 不修改原函数
- 可堆叠多个
### 4.3 重试策略
| 策略 | 适用 | 代码 |
|------|------|------|
| 固定间隔 | 简单 | `wait_fixed(5)` |
| 指数退避 | API | `wait_exponential` |
| 自定义 | 复杂 | 自定义 callable |

**选择指数退避**:
- API 限流友好
- 避免雪崩
- 自动调整
## 5. 评价
| 维度 | 评分 | 说明 |
|------|------|------|
| 健壮性 | ★★★★☆ | 多层处理 |
| 用户体验 | ★★★★☆ | 清晰反馈 |
| 可维护 | ★★★★☆ | 结构清晰 |
| 调试性 | ★★★☆☆ | 需日志 |
| 恢复 | ★★★☆☆ | 部分支持 |
## 6. 关键代码
```python
# 上下文管理器
def safe_execute(tool):
    try:
        return tool.run()
    except ToolError as e:
        print(f"Failed: {e}")
        return None
    finally:
        # 清理
        tool.cleanup()

```
def start(self):
    try:
        return self.tool.run()
    except ToolError as e:
        print(f"Tool failed: {e}")
        return None
```
## 7. 与原始对比
| 特性 | 原始 (TS) | 本实现 (Py) |
|------|-----------|-------------|
| 异常 | 完整层次 | 基础 |
| 重试 | 复杂策略 | 概念 |
| 日志 | 结构化 | print |
| 恢复 | 状态机 | 简单 |
| 用户反馈 | 丰富 | 基础 |
## 8. 代码片段
```python
# 使用上下文
def process_with_context(tool):
    with error_context() as ctx:
        result = tool.run()
        ctx.success()
        return result

# 自动重试
@retry(stop=3, wait=5)
def api_call():
    return requests.post(...)
```
