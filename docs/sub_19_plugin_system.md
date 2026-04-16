# 子领域 19：插件系统

> Claude Code 的扩展机制

---

## 一、概述

插件系统允许第三方开发者扩展 Claude Code 的功能，包括自定义工具、斜杠命令、Hook 和 UI 扩展。

### 核心概念

- **插件目录**：`~/.claude/plugins/` 或项目级 `.claude/plugins/`
- **插件清单**：`plugin.json` 定义插件元数据和能力
- **插件 API**：标准接口供插件调用

---

## 二、插件结构

### 2.1 目录结构

```
~/.claude/plugins/my-plugin/
├── plugin.json       # 插件清单
├── index.js          # 入口文件
├── tools/            # 自定义工具
│   └── myTool.js
├── commands/         # 自定义命令
│   └── myCommand.js
├── hooks/            # 生命周期 Hook
│   └── preToolUse.js
└── assets/           # 资源文件
    └── icon.png
```

### 2.2 插件清单

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "A sample plugin",
  "author": "developer",
  "main": "index.js",
  "capabilities": {
    "tools": ["myTool"],
    "commands": ["mycommand"],
    "hooks": ["PreToolUse", "PostToolUse"]
  },
  "permissions": [
    "fs.read",
    "fs.write",
    "network.fetch"
  ],
  "config": {
    "apiKey": {
      "type": "string",
      "required": true,
      "description": "API key for the service"
    }
  }
}
```

---

## 三、插件加载

### 3.1 发现插件

```typescript
async function discoverPlugins(): Promise<Plugin[]> {
  const pluginDirs = [
    path.join(os.homedir(), '.claude/plugins'),
    path.join(process.cwd(), '.claude/plugins'),
  ]

  const plugins: Plugin[] = []

  for (const dir of pluginDirs) {
    if (!await exists(dir)) continue

    const entries = await fs.readdir(dir, { withFileTypes: true })
    for (const entry of entries) {
      if (!entry.isDirectory()) continue

      const manifestPath = path.join(dir, entry.name, 'plugin.json')
      if (await exists(manifestPath)) {
        plugins.push(await loadPlugin(path.join(dir, entry.name)))
      }
    }
  }

  return plugins
}
```

### 3.2 加载插件

```typescript
async function loadPlugin(pluginDir: string): Promise<Plugin> {
  // 读取清单
  const manifest = await readJson(path.join(pluginDir, 'plugin.json'))

  // 验证权限
  const granted = await checkPermissions(manifest.permissions)
  if (!granted) {
    throw new Error(`Plugin ${manifest.name} permissions not granted`)
  }

  // 加载入口
  const entryPath = path.join(pluginDir, manifest.main || 'index.js')
  const module = await import(entryPath)

  return {
    manifest,
    module,
    dir: pluginDir,
  }
}
```

### 3.3 初始化插件

```typescript
async function initializePlugin(plugin: Plugin): Promise<void> {
  const { manifest, module } = plugin

  // 创建插件上下文
  const context: PluginContext = {
    config: await getPluginConfig(manifest.name),
    registerTool: (tool) => registerPluginTool(manifest.name, tool),
    registerCommand: (cmd) => registerPluginCommand(manifest.name, cmd),
    registerHook: (hook) => registerPluginHook(manifest.name, hook),
    logger: createPluginLogger(manifest.name),
  }

  // 调用插件初始化
  if (module.initialize) {
    await module.initialize(context)
  }
}
```

---

## 四、自定义工具

### 4.1 工具定义

```typescript
// tools/myTool.js
export default {
  name: 'myTool',
  description: 'A custom tool that does something',
  parameters: z.object({
    input: z.string().describe('Input to process'),
    options: z.object({
      verbose: z.boolean().optional(),
    }).optional(),
  }),
  isConcurrencySafe: true,

  async run(input, context) {
    // 工具逻辑
    const result = await processInput(input.input, input.options)

    return {
      content: result,
    }
  },
}
```

### 4.2 工具注册

```typescript
function registerPluginTool(pluginName: string, tool: Tool): void {
  // 添加前缀避免冲突
  const prefixedName = `plugin_${pluginName}_${tool.name}`

  toolRegistry.register({
    ...tool,
    name: prefixedName,
    _plugin: pluginName,
  })

  console.log(`Registered tool: ${prefixedName}`)
}
```

---

## 五、自定义命令

### 5.1 命令定义

```typescript
// commands/myCommand.js
export default {
  name: 'mycommand',
  aliases: ['mc'],
  description: 'A custom command',

  async handler(args, context) {
    console.log('Running my command with args:', args)

    // 可以调用工具或发送消息
    const result = await context.submitMessage(`Execute my command: ${args.join(' ')}`)

    return result
  },
}
```

### 5.2 命令注册

```typescript
function registerPluginCommand(pluginName: string, command: SlashCommand): void {
  // 添加前缀
  const prefixedName = `${pluginName}:${command.name}`

  commandRegistry.register({
    ...command,
    name: prefixedName,
    _plugin: pluginName,
  })
}
```

---

## 六、插件 Hook

### 6.1 Hook 定义

```typescript
// hooks/preToolUse.js
export default {
  type: 'PreToolUse',

  async handler(context) {
    const { toolName, toolInput } = context

    // 记录工具调用
    await logToolCall(toolName, toolInput)

    // 可以修改输入
    if (toolName === 'Bash') {
      context.toolInput.command = sanitizeCommand(toolInput.command)
    }
  },
}
```

### 6.2 Hook 注册

```typescript
function registerPluginHook(pluginName: string, hook: Hook): void {
  hookRegistry.register({
    ...hook,
    _plugin: pluginName,
  })
}
```

---

## 七、权限系统

### 7.1 权限列表

```typescript
const PLUGIN_PERMISSIONS = {
  // 文件系统
  'fs.read': 'Read files',
  'fs.write': 'Write files',

  // 网络
  'network.fetch': 'Make HTTP requests',
  'network.listen': 'Start HTTP server',

  // 子进程
  'process.spawn': 'Spawn child processes',

  // 系统
  'system.env': 'Access environment variables',
  'system.clipboard': 'Access clipboard',
}
```

### 7.2 权限检查

```typescript
async function checkPermissions(
  requested: string[],
  manifest: PluginManifest
): Promise<boolean> {
  const granted = await getGrantedPermissions(manifest.name)

  for (const permission of requested) {
    if (!granted.includes(permission)) {
      // 提示用户授权
      const approved = await promptPermission(permission, manifest)
      if (!approved) return false

      granted.push(permission)
    }
  }

  await saveGrantedPermissions(manifest.name, granted)
  return true
}
```

### 7.3 权限沙箱

```typescript
class PluginSandbox {
  private permissions: Set<string>

  constructor(permissions: string[]) {
    this.permissions = new Set(permissions)
  }

  check(permission: string): boolean {
    if (!this.permissions.has(permission)) {
      throw new Error(`Permission denied: ${permission}`)
    }
    return true
  }

  // 文件系统代理
  get fs() {
    return {
      readFile: async (path: string) => {
        this.check('fs.read')
        return fs.readFile(path, 'utf-8')
      },
      writeFile: async (path: string, content: string) => {
        this.check('fs.write')
        return fs.writeFile(path, content)
      },
    }
  }
}
```

---

## 八、插件配置

### 8.1 配置存储

```typescript
// ~/.claude/plugins/config/<plugin-name>.json
async function getPluginConfig(pluginName: string): Promise<Record<string, any>> {
  const configPath = path.join(
    os.homedir(),
    '.claude/plugins/config',
    `${pluginName}.json`
  )

  try {
    return await readJson(configPath)
  } catch {
    return {}
  }
}

async function setPluginConfig(
  pluginName: string,
  key: string,
  value: any
): Promise<void> {
  const config = await getPluginConfig(pluginName)
  config[key] = value

  const configPath = path.join(
    os.homedir(),
    '.claude/plugins/config',
    `${pluginName}.json`
  )

  await writeJson(configPath, config)
}
```

### 8.2 配置界面

```bash
# CLI 配置插件
claude plugin config my-plugin

# 输出
API Key: ****
Verbose: true
# 可编辑保存
```

---

## 九、插件管理

### 9.1 安装插件

```bash
# 从 npm 安装
claude plugin install my-plugin

# 从本地安装
claude plugin install ./path/to/plugin

# 从 URL 安装
claude plugin install https://example.com/plugin.tar.gz
```

### 9.2 管理命令

```typescript
async function handlePluginCommand(args: string[]): Promise<void> {
  const action = args[0]

  switch (action) {
    case 'list':
      const plugins = await discoverPlugins()
      plugins.forEach(p => console.log(`  - ${p.manifest.name} v${p.manifest.version}`))
      break

    case 'install':
      await installPlugin(args[1])
      break

    case 'uninstall':
      await uninstallPlugin(args[1])
      break

    case 'update':
      await updatePlugin(args[1])
      break

    case 'enable':
      await enablePlugin(args[1])
      break

    case 'disable':
      await disablePlugin(args[1])
      break

    default:
      console.log('Usage: /plugin [list|install|uninstall|update|enable|disable]')
  }
}
```

---

## 十、插件示例

### 10.1 完整示例

```javascript
// my-plugin/index.js
export default {
  name: 'my-plugin',
  version: '1.0.0',

  async initialize(context) {
    const { config, registerTool, registerCommand, registerHook, logger } = context

    // 注册工具
    registerTool({
      name: 'weather',
      description: 'Get weather for a location',
      parameters: z.object({
        location: z.string(),
      }),
      async run(input) {
        const response = await fetch(
          `https://api.weather.com/${input.location}?key=${config.apiKey}`
        )
        return { content: await response.text() }
      },
    })

    // 注册命令
    registerCommand({
      name: 'weather',
      description: 'Get weather for a location',
      async handler(args) {
        const location = args[0] || 'current'
        logger.info(`Getting weather for ${location}`)
        // ...
      },
    })

    // 注册 Hook
    registerHook({
      type: 'PreToolUse',
      async handler(context) {
        if (context.toolName === 'weather') {
          logger.debug('Weather tool called')
        }
      },
    })

    logger.info('Plugin initialized')
  },
}
```

---

## 十一、设计评价

### 11.1 优点

1. **标准接口**：统一的插件 API
2. **权限隔离**：保护用户安全
3. **热加载**：无需重启即可加载
4. **命名空间**：避免冲突

### 11.2 局限性

1. **性能开销**：每个插件独立加载
2. **依赖冲突**：插件间可能依赖冲突
3. **调试困难**：插件错误难以追踪

---

## 十二、总结

插件系统是 Claude Code 扩展性的核心：

1. **标准结构**：plugin.json + 模块
2. **能力注册**：工具、命令、Hook
3. **权限沙箱**：保护用户安全
4. **配置管理**：持久化配置
5. **CLI 管理**：安装/卸载/更新

插件系统让 Claude Code 成为可扩展的平台。
