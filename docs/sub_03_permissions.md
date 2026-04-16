# 子领域 03：权限系统

> Claude Code 的多层权限检查与安全控制机制

---

## 一、概述

Claude Code 拥有一个精细的权限系统，在 Agent 自主执行工具的能力与用户安全之间取得平衡。系统采用四层检查机制，支持多种权限模式。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/utils/permissions/` | 权限检查核心 |
| `src/utils/permissions/bashClassifier.ts` | Bash 命令分类器 |
| `src/utils/permissions/yoloClassifier.ts` | 危险操作分类器 |
| `src/utils/permissions/rules.ts` | 权限规则管理 |

---

## 二、权限模式

### 2.1 模式列表

| 模式 | 说明 | 适用场景 |
|---|---|---|
| `default` | 每次敏感操作询问用户 | 日常使用 |
| `plan` | 只允许读取操作 | 计划模式 |
| `auto` | 自动批准安全操作 | 信任环境 |
| `bypassPermissions` | 跳过所有权限检查 | 危险！仅调试 |

### 2.2 模式切换

```typescript
// CLI 参数
--dangerously-skip-permissions  // bypassPermissions 模式

// 斜杠命令
/permissions mode auto
/permissions mode default
```

---

## 三、四层权限检查

### 3.1 检查流程

```
工具调用请求
  │
  ▼
┌───────────────────────────┐
│ Layer 1: Deny 规则检查      │
│ permissionRules.deny       │
│ 匹配工具名/路径 pattern     │
└──────────┬────────────────┘
           │ 未 deny
           ▼
┌───────────────────────────┐
│ Layer 2: Allow 规则检查     │
│ alwaysAllowRules           │
│ 匹配命令 pattern            │
└──────────┬────────────────┘
           │ 未 allow
           ▼
┌───────────────────────────┐
│ Layer 3: 分类器判定         │
│ - bashClassifier           │
│ - yoloClassifier           │
│ 判断操作危险级别            │
└──────────┬────────────────┘
           │
           ▼
┌───────────────────────────┐
│ Layer 4: 根据模式决策       │
│ - auto → 自动批准安全操作   │
│ - default → 弹窗询问        │
│ - bypass → 全部批准         │
└───────────────────────────┘
```

### 3.2 代码实现

```typescript
async function canUseTool(
  tool: Tool,
  input: any,
  context: PermissionContext
): Promise<'allow' | 'deny' | 'ask'> {
  // Layer 1: Deny 规则
  if (matchesDenyRules(tool.name, input, context)) {
    return 'deny'
  }

  // Layer 2: Allow 规则
  if (matchesAllowRules(tool.name, input, context)) {
    return 'allow'
  }

  // Layer 3: 分类器
  const classification = classifyToolUse(tool, input)

  // Layer 4: 模式决策
  switch (context.mode) {
    case 'bypassPermissions':
      return 'allow'

    case 'auto':
      return classification === 'safe' ? 'allow' : 'ask'

    case 'plan':
      return tool.isConcurrencySafe ? 'allow' : 'deny'

    case 'default':
    default:
      return classification === 'safe' ? 'allow' : 'ask'
  }
}
```

---

## 四、Bash 命令分类器

### 4.1 分类级别

| 级别 | 说明 | 示例 |
|---|---|---|
| `safe` | 安全，自动批准 | `ls`, `cat`, `git status` |
| `confirm` | 需要确认 | `rm`, `npm install`, `git push` |
| `deny` | 明确拒绝 | `rm -rf /`, `mkfs` |

### 4.2 安全命令模式

```typescript
const SAFE_COMMAND_PATTERNS = [
  // 文件系统只读
  /^ls(\s|$)/,
  /^cat(\s|$)/,
  /^head(\s|$)/,
  /^tail(\s|$)/,
  /^find(\s|$)/,
  /^tree(\s|$)/,

  // 搜索
  /^grep(\s|$)/,
  /^rg(\s|$)/,
  /^ag(\s|$)/,

  // Git 只读
  /^git\s+status/,
  /^git\s+log/,
  /^git\s+diff/,
  /^git\s+show/,
  /^git\s+branch/,

  // 包管理只读
  /^npm\s+list/,
  /^pip\s+list/,
  /^pip\s+show/,

  // 系统
  /^echo(\s|$)/,
  /^pwd$/,
  /^which(\s|$)/,
  /^whoami$/,
  /^date$/,
]
```

### 4.3 危险命令模式

```typescript
const DANGER_COMMAND_PATTERNS = [
  // 系统破坏
  /^rm\s+-rf\s+\//,
  /^rm\s+-rf\s+~/,
  /^rm\s+-rf\s+\*/,

  // 磁盘操作
  /^mkfs/,
  /^dd\s+if=/,
  />\s*\/dev\/sd/,

  // 权限提升
  /^sudo\s+rm/,
  /^chmod\s+777/,

  // 网络危险
  /^curl.*\|\s*bash/,
  /^wget.*\|\s*bash/,
]
```

### 4.4 分类逻辑

```typescript
function classifyBashCommand(command: string): 'safe' | 'confirm' | 'deny' {
  // 标准化命令
  const normalizedCmd = command.trim().toLowerCase()

  // 检查危险模式
  for (const pattern of DANGER_COMMAND_PATTERNS) {
    if (pattern.test(normalizedCmd)) {
      return 'deny'
    }
  }

  // 检查安全模式
  for (const pattern of SAFE_COMMAND_PATTERNS) {
    if (pattern.test(normalizedCmd)) {
      return 'safe'
    }
  }

  // 默认需要确认
  return 'confirm'
}
```

---

## 五、权限规则

### 5.1 规则结构

```typescript
interface PermissionRule {
  // 规则类型
  type: 'allow' | 'deny'

  // 匹配模式
  pattern: string | RegExp

  // 适用工具
  tool?: string

  // 适用路径
  path?: string

  // 描述
  description?: string
}
```

### 5.2 规则配置

```json
// ~/.claude/permissions.json
{
  "allow": [
    {
      "tool": "Bash",
      "pattern": "npm run *",
      "description": "允许运行 npm scripts"
    },
    {
      "tool": "Edit",
      "path": "/home/user/project/**",
      "description": "允许编辑项目文件"
    }
  ],
  "deny": [
    {
      "tool": "Bash",
      "pattern": "rm -rf *",
      "description": "禁止递归删除"
    },
    {
      "path": "/etc/**",
      "description": "禁止操作系统目录"
    }
  ]
}
```

### 5.3 规则匹配

```typescript
function matchesDenyRules(
  toolName: string,
  input: any,
  context: PermissionContext
): boolean {
  for (const rule of context.permissionRules.deny) {
    // 工具名匹配
    if (rule.tool && rule.tool !== toolName) continue

    // 路径匹配
    if (rule.path && input.file_path) {
      if (!minimatch(input.file_path, rule.path)) continue
    }

    // 命令模式匹配
    if (rule.pattern && input.command) {
      const regex = typeof rule.pattern === 'string'
        ? new RegExp(rule.pattern)
        : rule.pattern
      if (regex.test(input.command)) return true
    }
  }

  return false
}
```

---

## 六、权限确认 UI

### 6.1 确认对话框

当权限检查返回 `ask` 时，弹出确认对话框：

```
┌──────────────────────────────────────────────┐
│ Claude wants to run:                          │
│                                               │
│   rm -rf node_modules                         │
│                                               │
│ This will permanently delete files.           │
│                                               │
│ [Allow] [Deny] [Allow always for this command]│
└──────────────────────────────────────────────┘
```

### 6.2 选项说明

| 选项 | 行为 |
|---|---|
| Allow | 本次允许，下次仍需确认 |
| Deny | 本次拒绝 |
| Allow always | 添加到 allow 规则，永久允许 |

### 6.3 权限拒绝记录

```typescript
interface PermissionDenial {
  toolName: string
  input: any
  reason: string
  timestamp: Date
}

// 记录拒绝
function recordDenial(denial: PermissionDenial) {
  permissionDenials.push(denial)

  // 超过阈值时警告
  if (permissionDenials.length > 10) {
    yield {
      type: 'warning',
      message: 'Multiple permission denials. Consider using /permissions to adjust settings.'
    }
  }
}
```

---

## 七、计划模式

### 7.1 模式特点

```typescript
// plan 模式下的权限逻辑
if (context.mode === 'plan') {
  // 只允许只读工具
  if (tool.isConcurrencySafe) {
    return 'allow'
  }
  return 'deny'
}
```

### 7.2 只读工具列表

| 允许 | 禁止 |
|---|---|
| Read | Edit |
| Grep | Write |
| Glob | Bash (写入类) |
| WebFetch | FileWrite |
| WebSearch | NotebookEdit |

---

## 八、企业策略

### 8.1 MDM 配置

```json
// /etc/claude-code/mdm-config.json
{
  "permissions": {
    "mode": "auto",
    "allowRules": [...],
    "denyRules": [
      {
        "pattern": "sudo *",
        "description": "禁止 sudo 命令"
      }
    ],
    "blockedTools": ["Bash"],
    "blockedPaths": ["/etc/**", "/var/**"]
  }
}
```

### 8.2 策略优先级

```
1. MDM 策略 (最高优先级，企业强制)
2. CLI 参数 (--dangerously-skip-permissions)
3. 项目配置 (.claude/permissions.json)
4. 用户配置 (~/.claude/permissions.json)
5. 默认规则 (最低优先级)
```

---

## 九、特殊场景

### 9.1 工具链执行

当工具调用其他工具时，权限如何传递？

```typescript
// AgentTool 调用子 Agent
async function runAgentTool(input) {
  // 子 Agent 继承权限上下文
  const subContext = {
    ...context,
    permissionContext: context.permissionContext, // 继承
  }

  return await runSubAgent(input, subContext)
}
```

### 9.2 MCP 工具权限

```typescript
// MCP 工具的权限由 MCP 服务器控制
// Claude Code 在调用前进行基础检查

async function runMcpTool(toolName: string, input: any) {
  // 基础权限检查
  const permission = await canUseTool(
    { name: toolName, isConcurrencySafe: false },
    input,
    context
  )

  if (permission !== 'allow') {
    return { error: 'Permission denied' }
  }

  // 调用 MCP 服务器
  return await mcpClient.callTool(toolName, input)
}
```

---

## 十、安全最佳实践

### 10.1 用户建议

1. **默认使用 default 模式**：保持安全默认
2. **谨慎添加 always allow 规则**：只在完全信任时添加
3. **定期审查规则**：清理不再需要的规则
4. **企业环境使用 MDM**：统一安全策略

### 10.2 开发建议

1. **新工具默认不安全**：`isConcurrencySafe = false`
2. **敏感操作需要确认**：不要绕过权限检查
3. **清晰的错误信息**：告诉用户为什么被拒绝
4. **审计日志**：记录所有权限决策

---

## 十一、设计评价

### 11.1 优点

1. **分层检查**：多层防护，逐级收紧
2. **灵活配置**：支持用户自定义规则
3. **安全默认**：默认需要确认敏感操作
4. **企业支持**：MDM 策略强制执行

### 11.2 局限性

1. **规则复杂**：多层级配置难以理解
2. **性能开销**：每次工具调用都需要权限检查
3. **绕过风险**：`bypassPermissions` 模式危险

### 11.3 潜在改进

1. **规则可视化**：UI 展示当前生效的规则
2. **智能分类**：ML 模型自动判断命令危险级别
3. **审计报告**：定期生成权限使用报告

---

## 十二、总结

权限系统是 Claude Code 安全的基石：

1. **四层检查**：Deny → Allow → Classifier → Mode
2. **多种模式**：default/plan/auto/bypass
3. **灵活配置**：用户自定义规则
4. **企业支持**：MDM 策略强制
5. **安全默认**：敏感操作需要确认

权限系统在 Agent 自主性和用户安全之间找到了平衡点。
