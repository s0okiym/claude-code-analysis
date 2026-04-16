# 子领域 13：Hook 系统

> Claude Code 的生命周期扩展机制

---

## 一、概述

Hook 系统允许在 Claude Code 的关键生命周期节点执行自定义代码，用于扩展、监控或修改行为。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/utils/hooks/` | Hook 注册与执行 |

---

## 二、Hook 类型

### 2.1 生命周期 Hook

```typescript
type HookType =
  | 'PreToolUse'      // 工具执行前
  | 'PostToolUse'     // 工具执行后
  | 'PreCompact'      // 压缩前
  | 'PostCompact'     // 压缩后
  | 'Stop'            // 循环结束时
  | 'PostSampling'    // API 采样后
  | 'SessionStart'    // 会话开始时
```

### 2.2 Hook 结构

```typescript
interface Hook {
  type: HookType
  handler: HookHandler
  priority?: number  // 执行优先级，数字越小越先执行
  once?: boolean     // 是否只执行一次
}

type HookHandler<T = any> = (context: HookContext) => Promise<T>

interface HookContext {
  // 工具相关
  toolName?: string
  toolInput?: any
  toolResult?: any

  // 压缩相关
  compactSummary?: string

  // 会话相关
  sessionId?: string
  messageCount?: number
}
```

---

## 三、Hook 注册

### 3.1 注册表

```typescript
class HookRegistry {
  private hooks: Map<HookType, Hook[]> = new Map()

  register(hook: Hook): void {
    const { type } = hook

    if (!this.hooks.has(type)) {
      this.hooks.set(type, [])
    }

    this.hooks.get(type)!.push(hook)

    // 按 priority 排序
    this.hooks.get(type)!.sort((a, b) => (a.priority || 100) - (b.priority || 100))
  }

  unregister(type: HookType, handler: HookHandler): void {
    const hooks = this.hooks.get(type)
    if (!hooks) return

    const index = hooks.findIndex(h => h.handler === handler)
    if (index > -1) {
      hooks.splice(index, 1)
    }
  }
}
```

### 3.2 注册方式

```typescript
// 代码注册
hookRegistry.register({
  type: 'PreToolUse',
  handler: async (context) => {
    console.log(`About to run: ${context.toolName}`)
  },
  priority: 1,
})

// 插件注册
// plugin/hooks/preToolUse.ts
export default {
  type: 'PreToolUse',
  handler: async (context) => {
    // 自定义逻辑
  },
}
```

---

## 四、Hook 执行

### 4.1 执行器

```typescript
async function runHooks(
  type: HookType,
  context: HookContext
): Promise<void> {
  const hooks = hookRegistry.get(type) || []

  for (const hook of hooks) {
    try {
      await hook.handler(context)

      // once 标记的 hook 执行后移除
      if (hook.once) {
        hookRegistry.unregister(type, hook.handler)
      }
    } catch (error) {
      console.error(`Hook ${type} failed:`, error)
      // 继续执行其他 hook
    }
  }
}
```

### 4.2 执行时机

```
Agent 循环
  │
  ├─ SessionStart Hook ← 会话开始时
  │
  ├─ for (;;) {
  │    │
  │    ├─ PreCompact Hook ← 压缩前 (如果需要压缩)
  │    ├─ 执行压缩
  │    ├─ PostCompact Hook ← 压缩后
  │    │
  │    ├─ API 调用
  │    ├─ PostSampling Hook ← API 响应后
  │    │
  │    ├─ for (tool of toolCalls) {
  │    │    ├─ PreToolUse Hook ← 工具执行前
  │    │    ├─ 执行工具
  │    │    └─ PostToolUse Hook ← 工具执行后
  │    │  }
  │    │
  │    └─ if (end_turn) {
  │         └─ Stop Hook ← 循环结束时
  │       }
  │  }
```

---

## 五、PreToolUse Hook

### 5.1 用途

- 日志记录
- 参数验证
- 权限增强检查
- 性能监控开始

### 5.2 示例

```typescript
// 工具调用审计
hookRegistry.register({
  type: 'PreToolUse',
  handler: async (context) => {
    const { toolName, toolInput } = context

    // 记录审计日志
    await writeAuditLog({
      timestamp: new Date(),
      tool: toolName,
      input: JSON.stringify(toolInput),
      sessionId: context.sessionId,
    })
  },
})

// 性能监控
hookRegistry.register({
  type: 'PreToolUse',
  handler: async (context) => {
    context.startTime = Date.now()
  },
  priority: 0, // 最先执行
})
```

---

## 六、PostToolUse Hook

### 6.1 用途

- 结果后处理
- 性能监控结束
- 错误处理
- 状态更新

### 6.2 示例

```typescript
// 性能统计
hookRegistry.register({
  type: 'PostToolUse',
  handler: async (context) => {
    const duration = Date.now() - context.startTime

    metrics.record({
      tool: context.toolName,
      duration,
      success: !context.toolResult?.isError,
    })
  },
  priority: 100, // 最后执行
})

// 结果过滤
hookRegistry.register({
  type: 'PostToolUse',
  handler: async (context) => {
    const { toolName, toolResult } = context

    if (toolName === 'Bash' && toolResult?.content) {
      // 过滤敏感信息
      toolResult.content = filterSensitiveInfo(toolResult.content)
    }
  },
})
```

---

## 七、PreCompact/PostCompact Hook

### 7.1 用途

- 压缩前保存状态
- 压缩后清理
- 记忆提取

### 7.2 示例

```typescript
// 压缩前提取重要信息到记忆
hookRegistry.register({
  type: 'PreCompact',
  handler: async (context) => {
    const messages = context.messages

    // 提取用户偏好
    const preferences = extractUserPreferences(messages)
    if (preferences.length > 0) {
      await writeToMemory({ type: 'user', content: preferences })
    }
  },
})

// 压缩后通知
hookRegistry.register({
  type: 'PostCompact',
  handler: async (context) => {
    console.log(`Context compressed: ${context.originalTokens} → ${context.newTokens} tokens`)
  },
})
```

---

## 八、Stop Hook

### 8.1 用途

- 清理资源
- 保存会话状态
- 生成报告

### 8.2 示例

```typescript
// 会话结束报告
hookRegistry.register({
  type: 'Stop',
  handler: async (context) => {
    const report = {
      sessionId: context.sessionId,
      messageCount: context.messageCount,
      toolCalls: context.toolCallCount,
      tokensUsed: context.totalTokens,
      duration: context.duration,
    }

    await saveSessionReport(report)
  },
})
```

---

## 九、PostSampling Hook

### 9.1 用途

- 响应监控
- 内容过滤
- Token 统计

### 9.2 示例

```typescript
// Token 使用统计
hookRegistry.register({
  type: 'PostSampling',
  handler: async (context) => {
    const usage = context.response.usage

    tokenTracker.record({
      input: usage.input_tokens,
      output: usage.output_tokens,
      cacheRead: usage.cache_read_input_tokens,
      cacheWrite: usage.cache_creation_input_tokens,
    })
  },
})
```

---

## 十、SessionStart Hook

### 10.1 用途

- 初始化
- 加载配置
- 恢复状态

### 10.2 示例

```typescript
// 加载项目特定配置
hookRegistry.register({
  type: 'SessionStart',
  handler: async (context) => {
    const projectConfig = await loadProjectConfig()

    if (projectConfig.customTools) {
      await registerCustomTools(projectConfig.customTools)
    }

    if (projectConfig.hooks) {
      for (const hook of projectConfig.hooks) {
        hookRegistry.register(hook)
      }
    }
  },
})
```

---

## 十一、Hook 与插件

### 11.1 插件定义 Hook

```typescript
// plugin/hooks/index.ts
export const hooks = [
  {
    type: 'PreToolUse',
    handler: async (context) => {
      // 插件逻辑
    },
  },
  {
    type: 'PostToolUse',
    handler: async (context) => {
      // 插件逻辑
    },
  },
]
```

### 11.2 加载插件 Hook

```typescript
async function loadPluginHooks(plugin: Plugin): Promise<void> {
  if (!plugin.hooks) return

  for (const hook of plugin.hooks) {
    hookRegistry.register(hook)
  }
}
```

---

## 十二、设计评价

### 12.1 优点

1. **扩展性强**：可在不修改核心代码的情况下扩展功能
2. **解耦**：核心逻辑与扩展逻辑分离
3. **优先级控制**：灵活控制执行顺序
4. **错误隔离**：Hook 错误不影响主流程

### 12.2 局限性

1. **执行顺序**：多个 Hook 可能产生冲突
2. **性能开销**：每个节点都有 Hook 调用开销
3. **调试困难**：Hook 链难以追踪

### 12.3 最佳实践

1. **优先级管理**：合理设置 priority
2. **错误处理**：Hook 内部捕获所有错误
3. **文档化**：清晰记录 Hook 的作用

---

## 十三、总结

Hook 系统是 Claude Code 的扩展基石：

1. **7 种类型**：覆盖关键生命周期节点
2. **优先级控制**：灵活的执行顺序
3. **插件集成**：插件通过 Hook 扩展功能
4. **错误隔离**：Hook 失败不影响主流程

Hook 系统让 Claude Code 成为可扩展的平台。
