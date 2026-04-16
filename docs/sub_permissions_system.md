# 子领域分析：权限系统

## 1. 功能概述
权限系统控制 Agent 可以执行的操作，防止危险命令执行，保护用户数据安全。

## 2. 核心组件

### 2.1 权限检查流程
```python
def can_use_tool(
    tool: Tool,
    context: PermissionContext
) -> Decision:
    """四层权限检查"""
    # 1. 检查 Deny 规则
    if matches_deny_rule(tool, context):
        return Decision(action='deny')
    
    # 2. 检查 Allow 规则
    if matches_always_allow_rule(tool, context):
        return Decision(action='allow')
    
    # 3. 分类器判定
    level = classify_safety(tool, context)
    
    # 4. 根据模式决策
    if context.mode == 'auto':
        return auto_decide(level)
    elif context.mode == 'default':
        return ask_user(tool)
```

### 2.2 Bash 分类器
```python
def bash_classifier(command: str) -> SafetyLevel:
    """分类命令安全级别"""
    safe_patterns = [
        r'^ls\s',      # ls 命令
        r'^cat\s',     # cat 命令
        r'^git\sstatus',  # git status
    ]
    
    dangerous_patterns = [
        r'rm\s+-rf',    # 强制删除
        r'>\s*/dev/',   # 重定向到设备
        r'curl.*\|.*sh', # 管道到 shell
    ]
    
    for pattern in dangerous_patterns:
        if re.search(pattern, command):
            return SafetyLevel.DANGEROUS
    
    for pattern in safe_patterns:
        if re.match(pattern, command):
            return SafetyLevel.SAFE
    
    return SafetyLevel.REQUIRES_CONFIRMATION
```

### 2.3 权限模式
| 模式 | 说明 | 使用场景 |
|------|------|----------|
| default | 每次敏感操作询问 | 默认，最安全 |
| plan | 只允许读取操作 | 规划阶段 |
| auto | 自动批准安全操作 | 信任环境 |
| bypass | 跳过所有检查 | 自动化脚本 |

## 3. 权限架构
```
工具调用请求
  │
  ▼
┌─────────────────────────────┐
│ 1. Deny 规则检查          │
│    - 匹配工具名/路径       │
└───────────┬─────────────────┘
            │ 未 deny
            ▼
┌─────────────────────────────┐
│ 2. Allow 规则检查          │
│    - 匹配命令 pattern       │
└───────────┬─────────────────┘
            │ 未 allow
            ▼
┌─────────────────────────────┐
│ 3. 分类器判定               │
│    - bash_classifier      │
│    - yolo_classifier   │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ 4. 根据模式决策    │
│    - auto → 自动     │
│    - default → 询问  │
│    - bypass → 跳过   │
└─────────────────────────┘
```
## 4. 关键设计
### 4.1 为什么需要分类器？
| 代码 | 优点 | 缺点 |
|------|------|------|
| 静态规则 | 快速，确定 | 无法适应新情况 |
| 动态分类 | 智能，自适应 | 可能误判 |
| 用户确认 | 最安全 | 打断流程 |
```

### 4.2 为什么保留 bypass 模式？
```python
if context.mode == 'bypass':
    return Decision(action='allow')
```

**使用场景**:
- CI/CD 自动化
- 测试环境
- 已知安全操作

**风险**:
- 可能执行危险命令
- 无审计日志
- 难以追踪

## 5. 评价
| 维度 | 评分 | 说明 |
|--------|------|------|
| 安全性 | ★★★★★ | 多层检查 |
| 用户体验 | ★★★★☆ | 平衡安全和流畅 |
| 可配置性 | ★★★★☆ | 支持自定义规则 |
| 透明度 | ★★★★☆ | 用户知道原因 |
| 复杂性 | ★★★★☆ | 多层逻辑 |
## 6. 关键代码
```
def Decision(action: str, reason: str = ''):
    """权限决策结果"""
    action: str  # 'allow', 'deny', 'ask'
    reason: str   # 决策原因
```
## 7. 与原始对比
| 特性 | 原始 (TS) | 本实现 (Py) |
|----------|-------------------|-------------------|
| 规则引擎 | 复杂 | 简化版 |
| 分类器 | 机器学习 | 正则匹配 |
| 模式 | 4 种 | 基础 |
| 日志 | 完整 | 无 |
| UI 集成 | 完整 | 无 |
```

## 8. 关键代码片段
```python
# 权限检查结果
def check_permission(tool, context):
    if tool.name in context.deny_list:
        return Decision('deny', 'In deny list')
    
    if context.mode == 'bypass':
        return Decision('allow', 'Bypass mode')
    
    level = classify(tool.command)
    if level == SafetyLevel.SAFE:
        return Decision('allow')
    
    return Decision('ask', f'{level.name} action requires confirmation')
```
