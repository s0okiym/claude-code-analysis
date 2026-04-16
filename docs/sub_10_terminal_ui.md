# 子领域 10：终端 UI

> 基于 Ink (React for CLI) 的终端界面系统

---

## 一、概述

Claude Code 的终端 UI 基于 Ink 框架（React for CLI）构建，提供了丰富的交互式界面，包括消息渲染、权限对话框、进度显示等。

### 核心文件

| 目录/文件 | 职责 |
|---|---|
| `src/ink/` | 定制版 Ink 框架 |
| `src/components/` | ~140 个 UI 组件 |
| `src/screens/` | 全屏 UI 页面 |

---

## 二、Ink 框架

### 2.1 架构概述

```
┌─────────────────────────────────────────────────────────────┐
│                    React Application                         │
│  (React Components)                                          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    React Reconciler                          │
│  (适配 React 到 Ink)                                          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Ink Renderer                              │
│  - 计算 DOM 差分                                             │
│  - 生成 ANSI 输出                                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Terminal Output                           │
│  - ANSI/CSI/OSC 序列                                         │
│  - 光标控制                                                  │
│  - 颜色渲染                                                  │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心 API

```typescript
// 渲染 React 组件到终端
import { render } from 'ink'

const App = () => (
  <Box>
    <Text color="green">Hello, World!</Text>
  </Box>
)

render(<App />)
```

### 2.3 基础组件

```typescript
// Box: 容器组件
<Box flexDirection="column" padding={1}>
  <Text>Line 1</Text>
  <Text>Line 2</Text>
</Box>

// Text: 文本组件
<Text color="blue" bold underline>
  Important text
</Text>

// ScrollBox: 可滚动容器
<ScrollBox height={20}>
  {longContent}
</ScrollBox>
```

---

## 三、定制版 Ink

### 3.1 文件结构

```
src/ink/
├── reconciler.ts      # React Reconciler 适配
├── renderer.ts        # 渲染引擎
├── render-to-screen.ts # 屏幕输出
├── termio/            # 终端 I/O 解析
│   ├── parser.ts      # ANSI/CSI/OSC 解析
│   ├── encoder.ts     # 输出编码
│   └── mouse.ts       # 鼠标事件
├── components/        # 基础组件
│   ├── Box.tsx
│   ├── Text.tsx
│   ├── ScrollBox.tsx
│   └── Button.tsx
└── hooks/             # 自定义 Hooks
    ├── useInput.ts
    ├── useTerminalViewport.ts
    └── useFocus.ts
```

### 3.2 Reconciler

```typescript
// React Reconciler 适配
import Reconciler from 'react-reconciler'

const hostConfig = {
  createInstance(type, props) {
    return { type, props, children: [] }
  },

  appendChild(parent, child) {
    parent.children.push(child)
  },

  removeChild(parent, child) {
    const index = parent.children.indexOf(child)
    if (index > -1) parent.children.splice(index, 1)
  },

  commitUpdate(instance, updatePayload, type, oldProps, newProps) {
    instance.props = newProps
  },

  // ... 更多配置
}

export const reconciler = Reconciler(hostConfig)
```

### 3.3 Renderer

```typescript
class InkRenderer {
  private lastOutput: string = ''
  private root: InkNode

  render(node: InkNode): string {
    // 计算 diff
    const output = this.renderNode(node)
    const diff = this.computeDiff(this.lastOutput, output)

    // 应用变更
    process.stdout.write(diff)

    this.lastOutput = output
    return output
  }

  private renderNode(node: InkNode): string {
    switch (node.type) {
      case 'box':
        return this.renderBox(node)
      case 'text':
        return this.renderText(node)
      default:
        return node.children.map(c => this.renderNode(c)).join('')
    }
  }

  private renderText(node: InkNode): string {
    const { color, bold, underline } = node.props
    let text = node.props.children

    if (color) text = ansiColor(text, color)
    if (bold) text = ansiBold(text)
    if (underline) text = ansiUnderline(text)

    return text
  }
}
```

---

## 四、核心组件

### 4.1 PromptInput

```typescript
// 用户输入框（含历史搜索、补全）
const PromptInput: FC<PromptInputProps> = ({
  placeholder,
  onSubmit,
  history,
}) => {
  const [value, setValue] = useState('')
  const [historyIndex, setHistoryIndex] = useState(-1)

  useInput((input, key) => {
    if (key.return) {
      onSubmit(value)
      setValue('')
    } else if (key.upArrow) {
      // 历史导航
      const newIndex = historyIndex + 1
      if (newIndex < history.length) {
        setHistoryIndex(newIndex)
        setValue(history[history.length - 1 - newIndex])
      }
    } else if (key.downArrow) {
      // 历史导航
      const newIndex = historyIndex - 1
      if (newIndex >= -1) {
        setHistoryIndex(newIndex)
        setValue(newIndex === -1 ? '' : history[history.length - 1 - newIndex])
      }
    } else if (key.backspace) {
      setValue(value.slice(0, -1))
    } else {
      setValue(value + input)
    }
  })

  return (
    <Box>
      <Text color="cyan">❯ </Text>
      <Text dimColor={!value}>{value || placeholder}</Text>
    </Box>
  )
}
```

### 4.2 Message

```typescript
// 消息渲染
const Message: FC<MessageProps> = ({ message }) => {
  switch (message.role) {
    case 'user':
      return (
        <Box flexDirection="column" marginY={1}>
          <Text bold color="blue">You:</Text>
          <Text>{message.content}</Text>
        </Box>
      )

    case 'assistant':
      return (
        <Box flexDirection="column" marginY={1}>
          <Text bold color="green">Claude:</Text>
          <Markdown content={message.content} />
        </Box>
      )

    case 'system':
      return (
        <Box marginY={1}>
          <Text dimColor>{message.content}</Text>
        </Box>
      )

    default:
      return null
  }
}
```

### 4.3 Messages (虚拟滚动)

```typescript
// 消息列表（虚拟滚动）
const Messages: FC<MessagesProps> = ({ messages }) => {
  const { height } = useTerminalViewport()
  const containerRef = useRef()

  // 计算可见范围
  const visibleStart = Math.max(0, messages.length - height)
  const visibleMessages = messages.slice(visibleStart)

  return (
    <ScrollBox ref={containerRef} height={height - 5}>
      {visibleMessages.map((msg, i) => (
        <Message key={msg.id || i} message={msg} />
      ))}
    </ScrollBox>
  )
}
```

### 4.4 Permission Dialog

```typescript
// 权限确认对话框
const PermissionDialog: FC<PermissionDialogProps> = ({
  request,
  onAllow,
  onDeny,
  onAllowAlways,
}) => {
  const [selected, setSelected] = useState(0)

  useInput((input, key) => {
    if (key.leftArrow) {
      setSelected(Math.max(0, selected - 1))
    } else if (key.rightArrow) {
      setSelected(Math.min(2, selected + 1))
    } else if (key.return) {
      if (selected === 0) onAllow()
      else if (selected === 1) onDeny()
      else onAllowAlways()
    }
  })

  return (
    <Box flexDirection="column" borderStyle="round" borderColor="yellow" padding={1}>
      <Text bold>Claude wants to run:</Text>
      <Text color="cyan">{request.command}</Text>
      <Text dimColor>{request.description}</Text>
      <Box marginTop={1}>
        <Button selected={selected === 0}>Allow</Button>
        <Button selected={selected === 1}>Deny</Button>
        <Button selected={selected === 2}>Allow always</Button>
      </Box>
    </Box>
  )
}
```

---

## 五、Hooks

### 5.1 useInput

```typescript
// 处理键盘输入
function useInput(
  handler: (input: string, key: Key) => void,
  options?: { isActive?: boolean }
): void {
  const { isActive = true } = options || {}

  useEffect(() => {
    if (!isActive) return

    const stdin = process.stdin

    const onData = (data: Buffer) => {
      const { input, key } = parseInput(data)
      handler(input, key)
    }

    stdin.on('data', onData)
    stdin.setRawMode(true)
    stdin.resume()

    return () => {
      stdin.off('data', onData)
      stdin.setRawMode(false)
      stdin.pause()
    }
  }, [isActive, handler])
}

interface Key {
  upArrow: boolean
  downArrow: boolean
  leftArrow: boolean
  rightArrow: boolean
  return: boolean
  escape: boolean
  backspace: boolean
  delete: boolean
  tab: boolean
  ctrl: boolean
  meta: boolean
  shift: boolean
}
```

### 5.2 useTerminalViewport

```typescript
// 获取终端视口大小
function useTerminalViewport(): { width: number; height: number } {
  const [viewport, setViewport] = useState({
    width: process.stdout.columns || 80,
    height: process.stdout.rows || 24,
  })

  useEffect(() => {
    const onResize = () => {
      setViewport({
        width: process.stdout.columns || 80,
        height: process.stdout.rows || 24,
      })
    }

    process.stdout.on('resize', onResize)
    return () => process.stdout.off('resize', onResize)
  }, [])

  return viewport
}
```

### 5.3 useFocus

```typescript
// 焦点管理
function useFocus(): {
  isFocused: boolean
  focus: () => void
  blur: () => void
} {
  const [isFocused, setIsFocused] = useState(false)

  const focus = useCallback(() => setIsFocused(true), [])
  const blur = useCallback(() => setIsFocused(false), [])

  return { isFocused, focus, blur }
}
```

---

## 六、终端 I/O

### 6.1 ANSI 序列

```typescript
// ANSI 转义序列
const ANSI = {
  // 光标
  cursorUp: (n = 1) => `\x1b[${n}A`,
  cursorDown: (n = 1) => `\x1b[${n}B`,
  cursorForward: (n = 1) => `\x1b[${n}C`,
  cursorBack: (n = 1) => `\x1b[${n}D`,
  cursorPosition: (row, col) => `\x1b[${row};${col}H`,
  cursorHide: '\x1b[?25l',
  cursorShow: '\x1b[?25h',

  // 清除
  eraseLine: '\x1b[2K',
  eraseDisplay: '\x1b[2J',

  // 颜色
  color: {
    red: '\x1b[31m',
    green: '\x1b[32m',
    blue: '\x1b[34m',
    yellow: '\x1b[33m',
    cyan: '\x1b[36m',
    magenta: '\x1b[35m',
    white: '\x1b[37m',
    reset: '\x1b[0m',
  },

  // 样式
  style: {
    bold: '\x1b[1m',
    dim: '\x1b[2m',
    underline: '\x1b[4m',
    inverse: '\x1b[7m',
  },
}
```

### 6.2 解析器

```typescript
// 解析终端输入
class TerminalParser {
  parse(data: Buffer): ParsedInput {
    const str = data.toString('utf8')

    // 检测特殊键
    if (str === '\x1b[A') return { key: 'upArrow' }
    if (str === '\x1b[B') return { key: 'downArrow' }
    if (str === '\x1b[C') return { key: 'rightArrow' }
    if (str === '\x1b[D') return { key: 'leftArrow' }
    if (str === '\r') return { key: 'return' }
    if (str === '\x1b') return { key: 'escape' }
    if (str === '\x7f') return { key: 'backspace' }
    if (str === '\t') return { key: 'tab' }

    // 普通字符
    return { input: str }
  }
}
```

---

## 七、组件分类

### 7.1 布局组件

| 组件 | 用途 |
|---|---|
| Box | 容器 |
| ScrollBox | 可滚动容器 |
| SplitPane | 分割面板 |
| Tabs | 标签页 |

### 7.2 显示组件

| 组件 | 用途 |
|---|---|
| Text | 文本 |
| Markdown | Markdown 渲染 |
| CodeBlock | 代码块 |
| Table | 表格 |
| Diff | Diff 查看 |

### 7.3 交互组件

| 组件 | 用途 |
|---|---|
| Button | 按钮 |
| Input | 输入框 |
| Select | 选择列表 |
| Checkbox | 复选框 |
| Spinner | 加载动画 |

### 7.4 消息组件

| 组件 | 用途 |
|---|---|
| Message | 单条消息 |
| Messages | 消息列表 |
| ToolUseMessage | 工具调用消息 |
| ErrorMessage | 错误消息 |

---

## 八、屏幕页面

### 8.1 REPL

```typescript
// 主 REPL 界面
const REPL: FC = () => {
  const { messages, sendMessage, status } = useREPL()

  return (
    <Box flexDirection="column" height="100%">
      {/* 标题栏 */}
      <Box>
        <Text bold color="green">Claude Code</Text>
        <Text dimColor> | {status}</Text>
      </Box>

      {/* 消息列表 */}
      <Box flexGrow={1}>
        <Messages messages={messages} />
      </Box>

      {/* 输入框 */}
      <PromptInput
        onSubmit={sendMessage}
        placeholder="Type a message..."
      />
    </Box>
  )
}
```

### 8.2 Doctor

```typescript
// 诊断页面
const Doctor: FC = () => {
  const checks = useDoctorChecks()

  return (
    <Box flexDirection="column">
      <Text bold>Claude Code Doctor</Text>
      <Text dimColor>Checking your environment...</Text>

      {checks.map((check, i) => (
        <Box key={i}>
          <Text color={check.passed ? 'green' : 'red'}>
            {check.passed ? '✓' : '✗'}
          </Text>
          <Text> {check.name}</Text>
          {!check.passed && (
            <Text dimColor> - {check.message}</Text>
          )}
        </Box>
      ))}
    </Box>
  )
}
```

---

## 九、性能优化

### 9.1 虚拟滚动

```typescript
// 只渲染可见消息
const VirtualMessages: FC<{ messages: Message[] }> = ({ messages }) => {
  const { height } = useTerminalViewport()
  const [scrollTop, setScrollTop] = useState(0)

  // 计算可见范围
  const visibleStart = scrollTop
  const visibleEnd = scrollTop + height
  const visibleMessages = messages.slice(visibleStart, visibleEnd)

  return (
    <Box flexDirection="column" height={height}>
      {visibleMessages.map((msg, i) => (
        <Message key={msg.id} message={msg} />
      ))}
    </Box>
  )
}
```

### 9.2 增量渲染

```typescript
// 只更新变化部分
class IncrementalRenderer {
  private lastFrame: string = ''

  render(newFrame: string): void {
    const diff = computeDiff(this.lastFrame, newFrame)

    // 只输出差异
    if (diff.length > 0) {
      process.stdout.write(diff)
    }

    this.lastFrame = newFrame
  }
}
```

---

## 十、设计评价

### 10.1 优点

1. **React 模型**：组件化开发，状态管理清晰
2. **丰富组件**：~140 个组件覆盖各种场景
3. **定制 Ink**：针对 Claude Code 优化
4. **响应式**：支持终端大小变化

### 10.2 局限性

1. **性能开销**：React Reconciler 有额外开销
2. **终端限制**：受终端能力限制
3. **调试困难**：React DevTools 不适用

### 10.3 最佳实践

1. **虚拟滚动**：大量数据时使用虚拟滚动
2. **增量更新**：避免全屏重绘
3. **延迟渲染**：非可见区域延迟渲染

---

## 十一、总结

终端 UI 是 Claude Code 用户体验的基础：

1. **Ink 框架**：React for CLI
2. **~140 组件**：覆盖各种交互场景
3. **定制优化**：针对 Claude Code 需求
4. **虚拟滚动**：处理大量消息
5. **增量渲染**：优化性能

终端 UI 让 Claude Code 从命令行工具变为交互式应用。
