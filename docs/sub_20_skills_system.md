# 子领域 20：技能系统

> Claude Code 的领域专用扩展机制

---

## 一、概述

技能系统（AgentSkills）是一种轻量级的领域专用扩展机制，比插件更简单，通过描述性配置定义自动激活规则和指令注入。

### 与插件的区别

| 特性 | 技能 (Skills) | 插件 (Plugins) |
|---|---|---|
| 复杂度 | 低（仅配置） | 高（需要代码） |
| 能力 | 指令注入、自动激活 | 工具、命令、Hook |
| 开发门槛 | 无需编程 | 需要 JavaScript |
| 适用场景 | 领域知识注入 | 功能扩展 |

---

## 二、技能结构

### 2.1 目录结构

```
.claude/skills/my-skill/
├── SKILL.md          # 技能定义（必需）
├── references/       # 参考文档
│   ├── api.md
│   └── examples.md
└── scripts/          # 脚本文件
    └── setup.sh
```

### 2.2 SKILL.md 格式

```markdown
# my-skill

## Description
Brief description of the skill. Used for auto-activation matching.

## Activation
- keywords: [api, rest, http]
- file_patterns: ["**/*.api.ts", "**/routes/**"]
- commands: ["/api"]

## Instructions
Detailed instructions for the skill...

## Tools
- api-tester: Test REST APIs
- schema-generator: Generate schemas from examples

## References
- [API Design Guide](./references/api.md)
- [Examples](./references/examples.md)
```

---

## 三、技能发现

### 3.1 发现路径

```typescript
const SKILL_PATHS = [
  // 内置技能
  path.join(__dirname, '../skills'),

  // 用户技能
  path.join(os.homedir(), '.claude/skills'),

  // 项目技能
  path.join(process.cwd(), '.claude/skills'),
]

async function discoverSkills(): Promise<Skill[]> {
  const skills: Skill[] = []

  for (const skillPath of SKILL_PATHS) {
    if (!await exists(skillPath)) continue

    const entries = await fs.readdir(skillPath, { withFileTypes: true })
    for (const entry of entries) {
      if (!entry.isDirectory()) continue

      const skillMdPath = path.join(skillPath, entry.name, 'SKILL.md')
      if (await exists(skillMdPath)) {
        skills.push(await loadSkill(path.join(skillPath, entry.name)))
      }
    }
  }

  return skills
}
```

### 3.2 加载技能

```typescript
async function loadSkill(skillDir: string): Promise<Skill> {
  const skillMdPath = path.join(skillDir, 'SKILL.md')
  const content = await readFile(skillMdPath, 'utf-8')

  // 解析 SKILL.md
  const parsed = parseSkillMd(content)

  return {
    name: path.basename(skillDir),
    dir: skillDir,
    description: parsed.description,
    activation: parsed.activation,
    instructions: parsed.instructions,
    tools: parsed.tools,
    references: parsed.references,
  }
}
```

### 3.3 解析 SKILL.md

```typescript
function parseSkillMd(content: string): ParsedSkill {
  const sections = parseMarkdownSections(content)

  return {
    description: sections.Description || sections.description || '',
    activation: parseActivation(sections.Activation || sections.activation),
    instructions: sections.Instructions || sections.instructions || '',
    tools: parseTools(sections.Tools || sections.tools),
    references: parseReferences(sections.References || sections.references),
  }
}

function parseActivation(section: string): Activation {
  const activation: Activation = {}

  const keywordsMatch = section.match(/keywords:\s*\[([^\]]+)\]/)
  if (keywordsMatch) {
    activation.keywords = keywordsMatch[1].split(',').map(s => s.trim().toLowerCase())
  }

  const patternsMatch = section.match(/file_patterns:\s*\[([^\]]+)\]/)
  if (patternsMatch) {
    activation.filePatterns = patternsMatch[1].split(',').map(s => s.trim())
  }

  const commandsMatch = section.match(/commands:\s*\[([^\]]+)\]/)
  if (commandsMatch) {
    activation.commands = commandsMatch[1].split(',').map(s => s.trim())
  }

  return activation
}
```

---

## 四、自动激活

### 4.1 激活规则

```typescript
interface Activation {
  keywords?: string[]      // 关键词匹配
  filePatterns?: string[]  // 文件路径匹配
  commands?: string[]      // 斜杠命令匹配
}

async function matchSkills(
  context: ActivationContext
): Promise<Skill[]> {
  const skills = await discoverSkills()
  const matched: Skill[] = []

  for (const skill of skills) {
    if (matchesSkill(skill, context)) {
      matched.push(skill)
    }
  }

  return matched
}

function matchesSkill(skill: Skill, context: ActivationContext): boolean {
  const { activation } = skill

  // 关键词匹配
  if (activation.keywords) {
    const inputLower = context.userInput.toLowerCase()
    if (activation.keywords.some(kw => inputLower.includes(kw))) {
      return true
    }
  }

  // 文件路径匹配
  if (activation.filePatterns) {
    const relevantFiles = context.recentFiles || []
    if (relevantFiles.some(file =>
      activation.filePatterns.some(pattern => minimatch(file, pattern))
    )) {
      return true
    }
  }

  // 命令匹配
  if (activation.commands) {
    const command = context.command
    if (command && activation.commands.includes(command)) {
      return true
    }
  }

  return false
}
```

### 4.2 激活时机

```
用户输入 → 技能匹配 → 注入指令 → 增强 Prompt
```

---

## 五、指令注入

### 5.1 构建指令

```typescript
async function buildSkillInstructions(skill: Skill): Promise<string> {
  let instructions = `## Skill: ${skill.name}\n\n`
  instructions += `${skill.description}\n\n`
  instructions += `${skill.instructions}\n\n`

  // 加载参考文档
  if (skill.references) {
    for (const ref of skill.references) {
      const refPath = resolveReferencePath(skill.dir, ref.path)
      const content = await readFile(refPath, 'utf-8')
      instructions += `### ${ref.name}\n\n${content}\n\n`
    }
  }

  return instructions
}
```

### 5.2 注入位置

```typescript
// 在 System Prompt 中注入
async function buildSystemPrompt(options: SystemPromptOptions): Promise<string> {
  // ...其他部分...

  // 技能指令
  const matchedSkills = await matchSkills({
    userInput: options.userInput,
    recentFiles: options.recentFiles,
  })

  let skillSection = ''
  for (const skill of matchedSkills) {
    skillSection += await buildSkillInstructions(skill)
  }

  return `
${basePrompt}

## Active Skills

${skillSection}

## Tools

${toolsSection}
`
}
```

---

## 六、技能工具

### 6.1 定义工具

```typescript
// 在 SKILL.md 中
## Tools
- api-tester: Test REST APIs with various methods
  - Parameters: url (string), method (string), body (object, optional)
  - Usage: api-tester --url "https://api.example.com" --method GET
```

### 6.2 工具加载

```typescript
async function loadSkillTools(skill: Skill): Promise<Tool[]> {
  const tools: Tool[] = []

  if (!skill.tools) return tools

  for (const toolDef of skill.tools) {
    tools.push({
      name: `skill_${skill.name}_${toolDef.name}`,
      description: toolDef.description,
      parameters: parseToolParameters(toolDef.parameters),
      run: async (input) => {
        // 执行脚本或调用内置实现
        return executeSkillTool(skill, toolDef, input)
      },
    })
  }

  return tools
}
```

### 6.3 脚本执行

```typescript
async function executeSkillTool(
  skill: Skill,
  toolDef: ToolDefinition,
  input: any
): Promise<ToolResult> {
  const scriptPath = path.join(skill.dir, 'scripts', `${toolDef.name}.sh`)

  if (await exists(scriptPath)) {
    // 执行脚本
    const result = await execFile(scriptPath, [JSON.stringify(input)])
    return { content: result.stdout }
  } else {
    // 使用内置实现
    return toolDef.implementation(input)
  }
}
```

---

## 七、技能管理

### 7.1 CLI 命令

```bash
# 列出所有技能
claude skill list

# 显示技能详情
claude skill show my-skill

# 创建新技能
claude skill create my-new-skill

# 激活技能
claude skill activate my-skill

# 停用技能
claude skill deactivate my-skill
```

### 7.2 斜杠命令

```typescript
async function handleSkillCommand(args: string[]): Promise<void> {
  const action = args[0]

  switch (action) {
    case 'list':
      const skills = await discoverSkills()
      skills.forEach(s => console.log(`  - ${s.name}: ${s.description}`))
      break

    case 'show':
      const skill = await getSkill(args[1])
      console.log(`Name: ${skill.name}`)
      console.log(`Description: ${skill.description}`)
      console.log(`Instructions:\n${skill.instructions}`)
      break

    case 'activate':
      await activateSkill(args[1])
      console.log(`Skill ${args[1]} activated`)
      break

    case 'deactivate':
      await deactivateSkill(args[1])
      console.log(`Skill ${args[1]} deactivated`)
      break

    default:
      console.log('Usage: /skill [list|show|activate|deactivate]')
  }
}
```

---

## 八、内置技能

### 8.1 debug 技能

```markdown
# debug

## Description
Helps investigate and fix bugs in code.

## Activation
- keywords: [bug, error, fix, debug, crash, exception]
- file_patterns: ["**/*.log", "**/error*"]

## Instructions
When debugging:
1. First understand the error message
2. Check relevant log files
3. Trace the stack trace
4. Identify the root cause
5. Propose a fix

## Tools
- stacktrace-analyzer: Analyze stack traces
- log-search: Search log files for patterns
```

### 8.2 test 技能

```markdown
# test

## Description
Helps write and run tests.

## Activation
- keywords: [test, spec, coverage, jest, vitest]
- file_patterns: ["**/*.test.ts", "**/*.spec.ts", "**/__tests__/**"]

## Instructions
When writing tests:
1. Identify what needs to be tested
2. Choose appropriate test framework
3. Write clear test cases
4. Ensure good coverage
5. Run tests and verify

## Tools
- test-runner: Run tests with coverage
- snapshot-update: Update test snapshots
```

---

## 九、技能市场

### 9.1 ClawHub

```
ClawHub (https://clawhub.ai)
├── 官方技能
│   ├── debug
│   ├── test
│   └── refactor
├── 社区技能
│   ├── api-design
│   ├── security-audit
│   └── documentation
└── 企业技能
    ├── compliance
    └── deployment
```

### 9.2 安装技能

```bash
# 从 ClawHub 安装
claude skill install debug

# 从 Git 安装
claude skill install https://github.com/user/my-skill.git
```

---

## 十、设计评价

### 10.1 优点

1. **低门槛**：无需编程，仅配置
2. **自动激活**：基于上下文自动匹配
3. **轻量级**：比插件更轻
4. **可分享**：可通过 ClawHub 分享

### 10.2 局限性

1. **能力有限**：不能添加复杂功能
2. **指令膨胀**：过多技能增加 Prompt 大小
3. **冲突风险**：多个技能可能冲突

### 10.3 最佳实践

1. **精准关键词**：避免误激活
2. **精简指令**：减少 token 消耗
3. **命名空间**：使用前缀避免工具冲突

---

## 十一、总结

技能系统是 Claude Code 的轻量级扩展机制：

1. **配置驱动**：SKILL.md 定义技能
2. **自动激活**：关键词/文件/命令匹配
3. **指令注入**：增强领域知识
4. **工具定义**：简单工具支持
5. **市场分享**：ClawHub 社区

技能系统让领域专家可以轻松扩展 Claude Code 的能力。
