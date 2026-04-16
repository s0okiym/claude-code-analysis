# 子领域 14：Feature Flag

> Claude Code 的功能控制与死代码消除

---

## 一、概述

Feature Flag 系统控制 Claude Code 的功能子集，支持编译时死代码消除和运行时动态开关。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/utils/featureFlags.ts` | Flag 定义与检查 |
| `src/services/analytics/growthbook.ts` | GrowthBook 集成 |

---

## 二、Feature Flag 类型

### 2.1 编译时 Flag

```typescript
// 使用 Bun 的 feature() 编译时剔除代码
const SleepTool = feature('PROACTIVE')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

### 2.2 运行时 Flag

```typescript
// 通过 GrowthBook 动态控制
if (isFeatureEnabled('WEB_BROWSER_TOOL')) {
  tools.push(new WebBrowserTool())
}
```

### 2.3 Flag 列表

| Flag | 说明 |
|---|---|
| `PROACTIVE` | 主动模式（后台任务） |
| `KAIROS` | 长期助手模式 |
| `BRIDGE_MODE` | Bridge 远程控制 |
| `DAEMON` | 守护进程模式 |
| `VOICE_MODE` | 语音模式 |
| `AGENT_TRIGGERS` | Agent 触发器 |
| `MONITOR_TOOL` | 监控工具 |
| `COORDINATOR_MODE` | 协调者模式 |
| `HISTORY_SNIP` | 历史 Snip 压缩 |
| `CONTEXT_COLLAPSE` | 上下文折叠 |
| `WEB_BROWSER_TOOL` | Web 浏览器工具 |

---

## 三、编译时死代码消除

### 3.1 机制

Bun 的 `feature()` 在编译时求值，未启用的代码会被完全剔除：

```typescript
// 源代码
if (feature('PROACTIVE')) {
  // 这段代码只在 PROACTIVE 启用时编译
  startBackgroundTasks()
}

// PROACTIVE=false 时编译结果
// (空，代码被剔除)
```

### 3.2 优势

1. **零运行时开销**：未启用功能的代码不进入 bundle
2. **减小包体积**：剔除无用代码
3. **性能优化**：避免不必要的条件判断

---

## 四、GrowthBook 集成

### 4.1 初始化

```typescript
import { GrowthBook } from '@growthbook/growthbook'

const growthbook = new GrowthBook({
  apiHost: 'https://cdn.growthbook.io',
  clientKey: process.env.GROWTHBOOK_CLIENT_KEY,
  attributes: {
    id: getUserId(),
    plan: getSubscriptionPlan(),
    region: getRegion(),
  },
})

await growthbook.loadFeatures()
```

### 4.2 Feature 检查

```typescript
function isFeatureEnabled(featureKey: string): boolean {
  return growthbook.isOn(featureKey)
}

function getFeatureValue<T>(featureKey: string, defaultValue: T): T {
  return growthbook.getFeatureValue(featureKey, defaultValue)
}
```

### 4.3 动态更新

```typescript
// GrowthBook 支持实时更新
growthbook.setRenderer(() => {
  // Feature 变化时重新渲染
  updateAvailableFeatures()
})
```

---

## 五、Feature Flag 使用模式

### 5.1 工具启用/禁用

```typescript
function getTools(): Tool[] {
  const tools: Tool[] = []

  // 基础工具
  tools.push(...getBaseTools())

  // 条件工具
  if (isFeatureEnabled('WEB_BROWSER_TOOL')) {
    tools.push(new WebBrowserTool())
  }

  if (isFeatureEnabled('VOICE_MODE')) {
    tools.push(new VoiceTool())
  }

  return tools
}
```

### 5.2 行为切换

```typescript
async function handleToolResult(result: ToolResult): Promise<void> {
  if (isFeatureEnabled('CONTEXT_COLLAPSE')) {
    // 新的上下文折叠逻辑
    await contextCollapse(result)
  } else {
    // 传统压缩逻辑
    await autoCompact(result)
  }
}
```

### 5.3 UI 变体

```typescript
const REPL: FC = () => {
  return (
    <Box>
      {isFeatureEnabled('KAIROS') ? (
        <KAIROSHeader />
      ) : (
        <DefaultHeader />
      )}
      <Messages />
      <PromptInput />
    </Box>
  )
}
```

---

## 六、A/B 测试

### 6.1 实验

```typescript
// 定义实验
const experiment = {
  key: 'new-compact-algorithm',
  variations: ['control', 'treatment'],
  weights: [0.5, 0.5],
}

// 获取变体
const variation = growthbook.run(experiment)

if (variation === 'treatment') {
  useNewCompactAlgorithm()
} else {
  useOldCompactAlgorithm()
}
```

### 6.2 指标追踪

```typescript
// 追踪实验指标
function trackMetric(metric: string, value: number): void {
  growthbook.track({
    metric,
    value,
    experiment: experiment.key,
    variation,
  })
}
```

---

## 七、Feature Flag 配置

### 7.1 环境变量

```bash
# 禁用所有实验性功能
CLAUDE_CODE_EXPERIMENTAL=false

# 启用特定功能
CLAUDE_CODE_WEB_BROWSER_TOOL=true
```

### 7.2 用户覆盖

```typescript
// 用户可以通过 CLI 覆盖
claude --feature WEB_BROWSER_TOOL=true

// 或在配置文件中
// ~/.claude/config.json
{
  "features": {
    "WEB_BROWSER_TOOL": true,
    "VOICE_MODE": false
  }
}
```

### 7.3 企业策略

```typescript
// MDM 可以强制禁用某些功能
const MDM_DISABLED_FEATURES = ['VOICE_MODE', 'WEB_BROWSER_TOOL']

function isFeatureEnabled(featureKey: string): boolean {
  // MDM 优先
  if (MDM_DISABLED_FEATURES.includes(featureKey)) {
    return false
  }

  // 然后检查 GrowthBook
  return growthbook.isOn(featureKey)
}
```

---

## 八、Feature Flag 与缓存

### 8.1 Prompt Cache 影响

```typescript
// Feature Flag 变化会影响 Prompt Cache
// 因为工具列表可能改变

function buildToolsSection(): string {
  const tools = getTools() // 受 Feature Flag 影响

  return `Available tools: ${tools.map(t => t.name).join(', ')}`
}
```

### 8.2 缓存策略

```typescript
// Feature 变化时清除缓存
growthbook.setRenderer(() => {
  clearPromptCache()
  updateTools()
})
```

---

## 九、调试与监控

### 9.1 查看当前 Feature

```bash
# CLI 命令
claude /features

# 输出
PROACTIVE: enabled
KAIROS: disabled
WEB_BROWSER_TOOL: enabled
...
```

### 9.2 遥测

```typescript
// 上报 Feature 使用情况
function reportFeatureUsage(): void {
  for (const feature of ALL_FEATURES) {
    telemetry.record({
      metric: 'feature_usage',
      feature: feature.key,
      enabled: isFeatureEnabled(feature.key),
    })
  }
}
```

---

## 十、设计权衡

### 10.1 编译时 vs 运行时

| 方案 | 优点 | 缺点 |
|---|---|---|
| 编译时 | 零开销 | 无法动态切换 |
| 运行时 | 动态切换 | 有条件判断开销 |

### 10.2 GrowthBook vs 自建

| 方案 | 优点 | 缺点 |
|---|---|---|
| GrowthBook | 成熟、实时、A/B | 外部依赖 |
| 自建 | 完全控制 | 开发成本 |

---

## 十一、总结

Feature Flag 系统控制 Claude Code 的功能子集：

1. **编译时消除**：Bun feature() 剔除未启用代码
2. **运行时开关**：GrowthBook 动态控制
3. **A/B 测试**：支持实验和指标追踪
4. **多层配置**：环境变量、用户覆盖、企业策略
5. **缓存感知**：Feature 变化清除缓存

Feature Flag 让 Claude Code 成为可定制的平台。
