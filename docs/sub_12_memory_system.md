# 子领域 12：记忆系统

> Claude Code 的跨会话知识持久化机制

---

## 一、概述

记忆系统是 Claude Code 的跨会话知识持久化机制，允许在不同会话之间保留用户偏好、项目知识和重要决策。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/memdir/memdir.ts` | 记忆目录管理 |
| `src/utils/memorySearch.ts` | 记忆语义搜索 |

---

## 二、记忆目录结构

### 2.1 目录布局

```
~/.claude/projects/<project-slug>/memory/
├── MEMORY.md           # 入口索引 (≤200行/25KB)
│   └── 指向各主题文件的索引
│
├── user_preferences.md # type: user
├── feedback_testing.md # type: feedback
├── architecture_decisions.md # type: project
├── api_reference.md    # type: reference
└── logs/               # KAIROS 模式日志
    └── 2026/03/2026-03-31.md
```

### 2.2 项目标识

```typescript
// 项目 slug 生成
function getProjectSlug(): string {
  const cwd = process.cwd()
  const hash = createHash('md5').update(cwd).digest('hex').slice(0, 8)
  const name = path.basename(cwd).replace(/[^a-zA-Z0-9]/g, '-')
  return `${name}-${hash}`
}

// 示例
// /home/user/my-project → my-project-a1b2c3d4
```

---

## 三、记忆类型

### 3.1 四种类型

| 类型 | 用途 | 示例 |
|---|---|---|
| `user` | 用户偏好、工作风格 | "用 bun 不用 npm" |
| `feedback` | 用户纠正和反馈 | "不要用 emoji" |
| `project` | 项目级非代码知识 | 截止日期、决策原因 |
| `reference` | 外部系统指针 | 仪表盘 URL、Linear 项目 |

### 3.2 记忆文件格式

```markdown
# user_preferences.md
type: user
created: 2026-03-15
updated: 2026-03-31

## 用户偏好

### 编程语言
- 首选 TypeScript
- 使用 ES6+ 语法
- 优先函数式风格

### 工具
- 包管理器: bun
- 测试框架: vitest
- Linter: eslint

### 沟通风格
- 简洁，不要啰嗦
- 不要使用 emoji
- 代码块用 ts 标记
```

---

## 四、MEMORY.md 索引

### 4.1 索引结构

```markdown
# MEMORY.md

> 项目记忆索引 - 最后更新: 2026-03-31

## 索引

| 文件 | 类型 | 描述 |
|---|---|---|
| [user_preferences.md](./user_preferences.md) | user | 用户偏好设置 |
| [feedback_testing.md](./feedback_testing.md) | feedback | 测试反馈记录 |
| [architecture_decisions.md](./architecture_decisions.md) | project | 架构决策记录 |
| [api_reference.md](./api_reference.md) | reference | API 文档链接 |

## 快速参考

- 截止日期: 2026-04-15
- API 文档: https://api.example.com/docs
- Linear 项目: https://linear.app/team/project/123
```

### 4.2 索引限制

```typescript
const MEMORY_MD_LIMITS = {
  maxLines: 200,
  maxBytes: 25000,
}

function enforceMemoryLimits(content: string): string {
  const lines = content.split('\n')

  if (lines.length > MEMORY_MD_LIMITS.maxLines) {
    console.warn(`MEMORY.md exceeds ${MEMORY_MD_LIMITS.maxLines} lines, truncating`)
    return lines.slice(0, MEMORY_MD_LIMITS.maxLines).join('\n')
  }

  if (Buffer.byteLength(content) > MEMORY_MD_LIMITS.maxBytes) {
    console.warn(`MEMORY.md exceeds ${MEMORY_MD_LIMITS.maxBytes} bytes, truncating`)
    return content.slice(0, MEMORY_MD_LIMITS.maxBytes)
  }

  return content
}
```

---

## 五、记忆注入

### 5.1 System Prompt 注入

```typescript
// MEMORY.md 通过 loadMemoryPrompt() 注入 system prompt 的动态区
async function loadMemoryPrompt(): Promise<string> {
  const memoryPath = path.join(getMemoryDir(), 'MEMORY.md')

  if (!await exists(memoryPath)) {
    return ''
  }

  const content = await readFile(memoryPath)

  return `
## Memory

I have access to persistent memory for this project:

${content}

Use this memory to:
- Remember user preferences across sessions
- Store project-specific knowledge
- Track decisions and their rationale
`
}
```

### 5.2 相关记忆注入

```typescript
// 每轮对话时通过附件系统注入
async function findRelevantMemories(context: AttachmentContext): Promise<Memory[]> {
  const signals = collectSemanticSignals(context)

  // 在记忆目录中搜索
  const results = await searchMemories(signals, {
    maxResults: 5,
    maxBytesPerResult: 4000,
  })

  return results
}
```

---

## 六、记忆写入

### 6.1 自动写入

```typescript
async function writeMemory(entry: MemoryEntry): Promise<void> {
  const { type, content } = entry

  // 确定目标文件
  const fileName = type === 'user' ? 'user_preferences.md'
    : type === 'feedback' ? 'feedback.md'
    : type === 'project' ? 'project_notes.md'
    : 'reference.md'

  const filePath = path.join(getMemoryDir(), fileName)

  // 追加内容
  const timestamp = new Date().toISOString()
  const formatted = formatMemoryEntry(content, timestamp)

  await appendFile(filePath, formatted)

  // 更新索引
  await updateMemoryIndex()
}
```

### 6.2 用户手动写入

```typescript
// 通过斜杠命令
// /memory add "用户偏好 TypeScript"

async function handleMemoryAdd(content: string): Promise<void> {
  await writeMemory({
    type: 'user',
    content,
    source: 'manual',
  })
}
```

---

## 七、语义搜索

### 7.1 索引构建

```typescript
// 为记忆文件构建向量索引
class MemoryIndex {
  private index: Map<string, Vector> = new Map()

  async buildIndex(memoryDir: string): Promise<void> {
    const files = await glob('**/*.md', { cwd: memoryDir })

    for (const file of files) {
      const content = await readFile(path.join(memoryDir, file))
      const chunks = this.chunkContent(content)

      for (const chunk of chunks) {
        const vector = await this.embed(chunk)
        this.index.set(`${file}:${chunk.id}`, vector)
      }
    }
  }

  private chunkContent(content: string): Chunk[] {
    // 按段落分割
    const paragraphs = content.split(/\n\n+/)
    return paragraphs.map((p, i) => ({
      id: i,
      content: p,
    }))
  }
}
```

### 7.2 搜索实现

```typescript
async function searchMemories(
  signals: string[],
  options: SearchOptions
): Promise<Memory[]> {
  const { maxResults, maxBytesPerResult } = options

  // 查询向量
  const queryVector = await embed(signals.join(' '))

  // 相似度搜索
  const results: SearchResult[] = []
  for (const [key, vector] of memoryIndex.index) {
    const similarity = cosineSimilarity(queryVector, vector)
    if (similarity > options.minScore) {
      results.push({ key, similarity })
    }
  }

  // 排序
  results.sort((a, b) => b.similarity - a.similarity)

  // 返回前 N 个
  const topResults = results.slice(0, maxResults)

  // 读取内容
  const memories: Memory[] = []
  for (const result of topResults) {
    const [file] = result.key.split(':')
    const content = await readMemoryChunk(result.key)

    memories.push({
      path: file,
      content: content.slice(0, maxBytesPerResult),
      score: result.similarity,
    })
  }

  return memories
}
```

---

## 八、KAIROS 助手模式

### 8.1 日志式记忆

在 KAIROS（长期助手）模式下，记忆改为追加式日志：

```
~/.claude/projects/<project-slug>/memory/logs/
├── 2026/
│   └── 03/
│       ├── 2026-03-30.md
│       └── 2026-03-31.md
```

### 8.2 日志格式

```markdown
# 2026-03-31

## 工作内容

### 上午
- 完成用户认证模块
- 修复登录页面样式问题
- 与产品确认 API 设计

### 下午
- 开始实现权限系统
- 代码审查通过

## 决策
- 使用 JWT 进行认证
- 权限系统采用 RBAC 模型

## 待办
- [ ] 完成权限系统
- [ ] 编写测试用例
```

### 8.3 夜间蒸馏

```typescript
// 夜间任务将日志蒸馏为 MEMORY.md
async function distillLogsToMemory(): Promise<void> {
  const logs = await readRecentLogs(7) // 最近 7 天

  // 提取重要信息
  const important = extractImportantInfo(logs)

  // 更新 MEMORY.md
  await updateMemoryIndex(important)

  // 清理旧日志
  await archiveOldLogs(30) // 30 天前
}
```

---

## 九、记忆管理

### 9.1 查看记忆

```bash
# 查看所有记忆文件
/memory list

# 查看特定文件
/memory show user_preferences.md
```

### 9.2 编辑记忆

```bash
# 编辑记忆文件
/memory edit user_preferences.md
```

### 9.3 搜索记忆

```bash
# 搜索记忆
/memory search "TypeScript"
```

### 9.4 清理记忆

```bash
# 清理过期记忆
/memory clean --older-than 30d
```

---

## 十、CLAUDE.md 与 Memory 的关系

### 10.1 CLAUDE.md

- **项目级指令**：随代码仓库分发
- **版本控制**：提交到 Git
- **团队共享**：所有开发者相同

### 10.2 Memory

- **用户级知识**：不随代码仓库分发
- **本地存储**：存储在 ~/.claude/
- **个人定制**：每个用户独立

### 10.3 配合使用

```
CLAUDE.md (项目规范)
  +
Memory (个人知识)
  =
完整的上下文
```

---

## 十一、设计权衡

### 11.1 本地 vs 云端

| 方案 | 优点 | 缺点 |
|---|---|---|
| 本地存储 | 隐私保护 | 无法跨设备 |
| 云端存储 | 跨设备同步 | 隐私风险 |

### 11.2 自动 vs 手动

| 方案 | 优点 | 缺点 |
|---|---|---|
| 自动写入 | 无需干预 | 可能记录噪音 |
| 手动写入 | 精确控制 | 用户负担 |

---

## 十二、总结

记忆系统是 Claude Code 跨会话的关键：

1. **四种类型**：user/feedback/project/reference
2. **双层注入**：MEMORY.md + 相关记忆
3. **语义搜索**：向量索引快速匹配
4. **KAIROS 模式**：日志式长期记忆
5. **预算控制**：防止 token 爆炸

记忆系统让 Claude Code 能够"记住"用户和项目。
