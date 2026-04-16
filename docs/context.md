# Claude Code — Context 定义、管理与生命周期

> 深度分析 Claude Code 如何定义、构建、使用、调整和管理 context（上下文）

---

## 一、什么是 Context

在 Claude Code 中，**context** 指的是发送给 Anthropic API 的所有信息，它决定了模型"看到"什么、能做什么。Context 由三个层面构成：

| 层面 | 内容 | 位置 | 缓存策略 |
|---|---|---|---|
| **System Prompt** | 角色定义、工具使用指南、环境信息、记忆指令 | API `system` 字段 | 静态部分可跨用户缓存 (global scope) |
| **User Context** | CLAUDE.md 指令、当前日期 | API `system` 字段前缀 | 会话级缓存 |
| **System Context** | Git 状态、缓存破坏标记 | API `system` 字段前缀 | 会话级缓存 |
| **Messages** | 对话历史（用户消息、助手响应、工具结果） | API `messages` 字段 | 压缩/截断管理 |

**核心矛盾**: 模型能看到的 context 越多，回答越准确；但 context 窗口有限（200K~1M tokens），必须持续管理。

---

## 二、Context 的完整生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                    Context 生命周期                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 构建 (Build)                                                │
│     ├─ System Prompt 组装                                       │
│     ├─ User Context 收集 (CLAUDE.md + 日期)                      │
│     └─ System Context 收集 (Git 状态)                            │
│              │                                                  │
│              ▼                                                  │
│  2. 使用 (Use) — API 调用                                       │
│     ├─ prependUserContext() — 前缀注入                           │
│     ├─ appendSystemContext() — 后缀注入                          │
│     └─ messages[] — 对话历史                                     │
│              │                                                  │
│              ▼                                                  │
│  3. 增长 (Grow) — 工具结果、助手回复、附件                         │
│     ├─ 工具执行结果追加到 messages                                │
│     ├─ 附件注入 (文件、记忆、技能发现)                              │
│     └─ 流式输出逐步填充                                           │
│              │                                                  │
│              ▼                                                  │
│  4. 感知 (Monitor) — Token 计数                                  │
│     ├─ tokenCountWithEstimation() — 实时估算                      │
│     ├─ tokenUsageFromAPI — 精确计数                               │
│     └─ calculateTokenWarningState() — 阈值判断                    │
│              │                                                  │
│              ▼                                                  │
│  5. 压缩 (Compact) — 上下文管理                                  │
│     ├─ 自动压缩 (autoCompact)                                    │
│     ├─ 微压缩 (microCompact)                                     │
│     ├─ 工具结果清除 (function result clearing)                    │
│     ├─ Snip 压缩                                                 │
│     └─ 上下文折叠 (context collapse)                              │
│              │                                                  │
│              ▼                                                  │
│  6. 持久化 (Persist) — 会话恢复                                  │
│     └─ recordTranscript() → JSONL 文件                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、Context 构建：三层组装

### 3.1 System Prompt 构建 (`constants/prompts.ts`)

System Prompt 是最复杂、最大的 context 组件，由多个 section 按优先级组装：

```
buildEffectiveSystemPrompt()
  │
  ├─ 优先级 0: Override (完全替换，如 loop 模式)
  ├─ 优先级 1: Coordinator 系统提示词 (多 Agent 协调模式)
  ├─ 优先级 2: Agent 系统提示词 (主线程 Agent)
  │    proactive 模式: 追加到默认
  │    普通模式: 替换默认
  ├─ 优先级 3: 自定义系统提示词 (--system-prompt)
  ├─ 优先级 4: 默认系统提示词 (getSystemPrompt)
  └─ 末尾追加: appendSystemPrompt
```

#### 默认 System Prompt 的内部结构

默认 system prompt 分为**静态区**和**动态区**，由边界标记 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 分隔：

```
┌──────────────────────────────┐
│         静态区 (Static)        │  ← 可跨用户缓存 (global scope)
│                              │
│ 1. 角色定义 (Intro)            │  "You are Claude Code..."
│ 2. 系统规则 (System)           │  工具权限、system-reminder 说明
│ 3. 任务执行 (Doing Tasks)      │  代码风格、安全注意事项
│ 4. 行为准则 (Actions)          │  什么操作需要确认
│ 5. 工具使用 (Using Tools)      │  优先用专用工具而非 Bash
│ 6. 语气风格 (Tone & Style)     │  简洁、无 emoji
│ 7. 输出效率 (Output)           │  直奔主题
│                              │
├── SYSTEM_PROMPT_DYNAMIC_BOUNDARY ──┤
│                              │
│         动态区 (Dynamic)        │  ← 会话/用户特定，不能跨用户缓存
│                              │
│ 8. 会话指导 (Session Guidance) │  Agent/fork/skill 使用指导
│ 9. 记忆系统 (Memory)          │  loadMemoryPrompt()
│10. 环境信息 (Environment)      │  OS、平台、模型、工作目录
│11. 语言偏好 (Language)         │  用户指定语言
│12. 输出风格 (Output Style)     │  自定义输出样式
│13. MCP 指令 (MCP Instructions)│  第三方 MCP 服务器指令
│14. Scratchpad 指令            │  临时文件目录说明
│15. 函数结果清除说明            │  工具结果自动清理提示
│16. Token 预算说明             │  用户指定 token 目标时
│17. Brief 指令                 │  KAIROS 模式的简报指令
│                              │
└──────────────────────────────┘
```

**为什么分静态/动态？**

目的是 **Prompt Cache 优化**。Anthropic API 支持 prompt caching —— 如果 system prompt 的前缀不变，API 不会重新计算这些 token，直接从缓存读取（快 ~10x，便宜 ~90%）。

- 静态区对所有用户都一样（角色定义、系统规则等），可以用 `global` scope 缓存
- 动态区因用户/会话而异（MCP 指令、语言偏好等），只能用 `session` scope

**取舍**: 把更多信息放入静态区能提高缓存命中率，但静态区对所有用户完全相同，所以任何用户特定信息都不能放进去。这是一个**通用性 vs 缓存效率**的权衡。

### 3.2 User Context (`context.ts` → `getUserContext()`)

```
getUserContext() → { claudeMd, currentDate }
  │
  ├─ CLAUDE.md 指令 (claudeMd 字段)
  │   └─ getClaudeMds() → 加载多层指令文件
  │
  └─ 当前日期 (currentDate 字段)
      └─ "Today's date is 2026-03-31."
```

#### CLAUDE.md 加载层次

CLAUDE.md 是用户自定义的持久指令，按优先级从低到高加载：

```
优先级 1 (最低): Managed Memory
  └─ /etc/claude-code/CLAUDE.md — 企业管理员全局指令

优先级 2: User Memory
  └─ ~/.claude/CLAUDE.md — 用户私人全局指令（跨项目）

优先级 3: Project Memory
  ├─ CLAUDE.md — 项目根目录
  ├─ .claude/CLAUDE.md — 项目 .claude 目录
  └─ .claude/rules/*.md — 项目规则文件

优先级 4 (最高): Local Memory
  └─ CLAUDE.local.md — 私人项目指令（不提交到 Git）
```

**发现机制**: 从当前目录向上遍历到根目录，**离当前目录越近的文件优先级越高**（加载越晚，模型更关注后面的内容）。

**@include 指令**: CLAUDE.md 支持 `@path` 语法引用其他文件，会递归展开，有循环引用保护。

**取舍**: 层次越多灵活性越高，但也增加 token 消耗。每个层次的内容都会注入到 context 中。设置了 40,000 字符上限（`MAX_MEMORY_CHARACTER_COUNT`）。

### 3.3 System Context (`context.ts` → `getSystemContext()`)

```
getSystemContext() → { gitStatus, cacheBreaker }
  │
  ├─ gitStatus (仅在 Git 仓库时)
  │   ├─ 当前分支
  │   ├─ 主分支名
  │   ├─ git status --short (截断到 2000 字符)
  │   ├─ 最近 5 条 commit
  │   └─ Git 用户名
  │
  └─ cacheBreaker (仅 ant 内部调试)
```

**关键设计**: `getSystemContext` 和 `getUserContext` 都用 `memoize` 缓存，**整个会话期间只计算一次**。这意味着：
- Git 状态是**快照**，不会在对话中更新（注释明确说明 "this status is a snapshot in time"）
- 日期也不会更新（通过 `getDateChangeAttachments` 附件机制处理跨午夜情况）

### 3.4 整体组装流程

`fetchSystemPromptParts()` 并行获取三个部分：

```
fetchSystemPromptParts()
  │
  ├─ Promise.all([
  │    getSystemPrompt(tools, model, dirs, mcpClients),  // 默认 system prompt
  │    getUserContext(),                                   // CLAUDE.md + 日期
  │    getSystemContext()                                  // Git 状态
  │  ])
  │
  └─ QueryEngine 最终组装:
     systemPrompt = [
       ...defaultSystemPrompt (或 customSystemPrompt),
       ...(memoryMechanicsPrompt),  // SDK 模式的记忆指令
       ...(appendSystemPrompt),     // 追加系统提示词
     ]
```

在 API 调用时：
- `prependUserContext()` — User Context 作为 system prompt 前缀
- `appendSystemContext()` — System Context 追加到 system prompt

---

## 四、Context 增长：附件与工具结果

### 4.1 附件系统 (`utils/attachments.ts`)

附件是 Context 中最灵活的动态注入机制。每轮对话时，`getAttachments()` 收集并注入以下内容：

```
getAttachments() — 每轮 API 调用前执行
  │
  ├─ 1. @文件引用
  │   └─ 用户输入中的 @path → 读取文件内容注入
  │
  ├─ 2. IDE 选区
  │   └─ 用户在 IDE 中选中的代码片段
  │
  ├─ 3. 剪贴板图片
  │   └─ 粘贴的图片（base64 编码，自动压缩）
  │
  ├─ 4. 相关记忆 (Relevant Memories)
  │   └─ findRelevantMemories() — 语义搜索记忆文件
  │   └─ 最多 5 个文件，每个 ≤ 4KB，总计 ≤ 60KB/会话
  │
  ├─ 5. 技能发现 (Skill Discovery)
  │   └─ 根据用户任务自动推荐相关技能
  │
  ├─ 6. 任务状态 (Task Attachments)
  │   └─ Todo 列表、任务进度
  │
  ├─ 7. MCP 资源
  │   └─ MCP 服务器提供的资源内容
  │
  ├─ 8. Agent 列表 (Delta)
  │   └─ 可用 Agent 的变更通知
  │
  ├─ 9. MCP 指令 (Delta)
  │   └─ MCP 服务器指令的增量更新
  │
  ├─ 10. 延迟工具 (Deferred Tools)
  │   └─ 工具搜索发现的额外工具
  │
  ├─ 11. 日期变更
  │   └─ 跨午夜时注入新日期提醒
  │
  ├─ 12. 上下文效率提醒
  │   └─ 提醒模型注意 token 使用
  │
  └─ 13. 压缩后恢复文件
      └─ 压缩后重新注入最近修改的文件
```

**关键策略**:

1. **去重**: `filterDuplicateMemoryAttachments()` 确保不重复注入相同记忆
2. **预算控制**: 每种附件类型都有独立的 token/字节预算
3. **增量注入**: MCP 指令和 Agent 列表使用 delta 模式，只在变化时注入
4. **会话累计上限**: 记忆注入有 60KB 的会话总量限制

### 4.2 工具结果管理 (`utils/toolResultStorage.ts`)

工具执行结果是对话 context 增长的主要来源。管理策略：

```
工具结果生命周期:
  │
  ├─ 生成时:
  │   ├─ getPersistenceThreshold() — 获取该工具的持久化阈值
  │   ├─ 结果 ≤ 阈值: 直接内联到 messages
  │   └─ 结果 > 阈值: 持久化到磁盘文件，注入引用
  │
  ├─ 微压缩时:
  │   ├─ 保留最近 N 个工具结果 (默认 keepRecent=5)
  │   └─ 旧结果替换为 [Old tool result content cleared]
  │
  └─ 完全压缩时:
      └─ 所有工具结果随 messages 一起被摘要替换
```

**阈值配置**: 
- 全局默认: `DEFAULT_MAX_RESULT_SIZE_CHARS` (~50K chars)
- 每个工具可声明自己的 `maxResultSizeChars`
- GrowthBook 可按工具名覆盖阈值

---

## 五、Context 监控：Token 计数与阈值

### 5.1 Context 窗口大小 (`utils/context.ts`)

```
getContextWindowForModel(model, betas)
  │
  ├─ 环境变量覆盖 (CLAUDE_CODE_MAX_CONTEXT_TOKENS) — 最高优先级
  ├─ [1m] 后缀检测 → 1,000,000 tokens
  ├─ 模型能力查询 (getModelCapability)
  ├─ Beta Header 检测 (1M context beta)
  ├─ Sonnet 1M 实验组判断
  ├─ Ant 内部模型配置
  └─ 默认值: 200,000 tokens
```

**有效窗口**（扣除输出预留）：

```
effectiveContextWindow = contextWindow - reservedForSummary
```

其中 `reservedForSummary = min(maxOutputTokens, 20,000)`，为压缩操作预留输出空间。

### 5.2 多级阈值体系

```
┌───────────────────────────────────────────────────┐
│           200K Token Context Window 示例           │
│                                                   │
│  0K ──────────── 可用空间 ────────────── 180K      │
│     ↑                                    ↑        │
│     │  13K buffer                        │        │
│     │  (AUTOCOMPACT_BUFFER)              │        │
│     │                                    │        │
│  167K ───── 自动压缩触发线 ─────────────────        │
│     ↑                                             │
│     │  20K buffer (WARNING)                        │
│     │                                              │
│  160K ───── 警告阈值 ─────────────────────          │
│     ↑                                             │
│     │  20K buffer (ERROR)                          │
│     │                                              │
│  160K ───── 错误阈值 ─────────────────────          │
│     ↑                                             │
│     │  3K buffer                                   │
│     │                                              │
│  197K ───── 阻塞限制 ─────────────────────          │
│                                                   │
│  200K ───── 上下文窗口上限 ───────────────          │
└───────────────────────────────────────────────────┘
```

`calculateTokenWarningState()` 返回四个布尔值：

| 状态 | 阈值 | 触发动作 |
|---|---|---|
| `isAboveWarningThreshold` | threshold - 20K | UI 显示黄色警告 |
| `isAboveErrorThreshold` | threshold - 20K | UI 显示红色错误 |
| `isAboveAutoCompactThreshold` | threshold - 13K | 触发自动压缩 |
| `isAtBlockingLimit` | effectiveWindow - 3K | 阻止继续，强制压缩 |

### 5.3 Token 预算系统 (`query/tokenBudget.ts`)

用户可指定 token 目标（如 "+500k tokens"），系统会持续工作直到达到目标：

```
checkTokenBudget()
  │
  ├─ 已用 < 90% 目标 且 无收益递减 → continue
  │   └─ 注入 nudge: "You've used X% of your token budget"
  │
  ├─ 已用 ≥ 90% 目标 → stop (完成)
  │
  └─ 连续 3+ 轮增量 < 500 tokens (收益递减) → stop
```

---

## 六、Context 压缩：多层管理策略

这是 context 管理的核心。Claude Code 实现了 **五层压缩策略**，从轻到重依次触发：

### 6.1 第一层：工具结果清除 (Function Result Clearing)

**触发时机**: 每轮 API 调用后的微压缩阶段
**机制**: 保留最近 N 个工具结果，旧的替换为 `[Old tool result content cleared]`
**目的**: 工具结果是对话中最大的 token 消费者，这是最轻量的回收方式

```
微压缩前:
  [user] "帮我找 bug"
  [assistant] tool_use: GrepTool("error")
  [user] tool_result: "1000行 grep 输出..."     ← 占用大量 token
  [assistant] "找到了，让我看看..."
  [user] tool_result: "500行文件内容..."         ← 占用大量 token
  [assistant] tool_use: GrepTool("fix")
  [user] tool_result: "200行 grep 输出..."       ← 最近，保留

微压缩后:
  [user] "帮我找 bug"
  [assistant] tool_use: GrepTool("error")
  [user] tool_result: [Old tool result content cleared]   ← 已清除
  [assistant] "找到了，让我看看..."
  [user] tool_result: [Old tool result content cleared]   ← 已清除
  [assistant] tool_use: GrepTool("fix")
  [user] tool_result: "200行 grep 输出..."               ← 保留
```

### 6.2 第二层：自动压缩 (Auto Compact)

**触发时机**: token 使用超过 `autoCompactThreshold`（有效窗口 - 13K buffer）
**机制**: Fork 一个子 Agent，用完整的对话历史生成摘要，替换旧消息

```
autoCompactIfNeeded() 流程:
  │
  ├─ shouldAutoCompact() → 检查 token 是否超阈值
  │
  ├─ trySessionMemoryCompaction() — 先尝试会话记忆压缩
  │   └─ 将重要信息存入记忆文件而非摘要
  │
  ├─ compactConversation() — 核心压缩
  │   ├─ stripImagesFromMessages() — 移除图片（节省大量 token）
  │   ├─ fork 子 Agent 生成摘要
  │   │   └─ 使用专门的压缩 prompt (compact/prompt.ts)
  │   │       ├─ <analysis> 思考分析
  │   │       └─ <summary> 9 节摘要:
  │   │           1. 主要请求和意图
  │   │           2. 关键技术概念
  │   │           3. 文件和代码段
  │   │           4. 错误和修复
  │   │           5. 问题解决过程
  │   │           6. 所有用户消息
  │   │           7. 待办任务
  │   │           8. 当前工作
  │   │           9. 可选下一步
  │   │
  │   ├─ buildPostCompactMessages() — 构建压缩后的消息列表
  │   │   ├─ 保留压缩前最近的几条消息
  │   │   ├─ 插入 compact_boundary 系统消息
  │   │   └─ 恢复最近修改的文件 (最多5个，每个 ≤5K tokens)
  │   │
  │   └─ postCompactCleanup() — 后清理
  │       ├─ 清理内存中的旧消息引用
  │       └─ 重置状态
  │
  └─ 连续失败断路器: 最多连续失败 3 次
```

**压缩结果**:

```
压缩前 (180K tokens):
  [user] "帮我重构这个模块"
  [assistant] tool_use: Read → "500行代码..."
  [user] tool_result: "500行代码..."
  [assistant] tool_use: Edit → ...
  [user] tool_result: "..."
  ... (100+ 轮对话)
  [assistant] "我正在修复..."

压缩后 (~30K tokens):
  [system: compact_boundary] { 压缩元数据 }
  [user: compact_summary] "用户请求重构认证模块，已完成5个文件中的3个..."
  [assistant: 恢复的最近文件内容]  ← 保留关键上下文
  [user] "继续"
  [assistant] "我正在修复..."       ← 最近的消息保留
```

### 6.3 第三层：Snip 压缩 (History Snip)

**触发时机**: 长时间 SDK 会话中的额外压缩（feature flag `HISTORY_SNIP`）
**机制**: 比 autoCompact 更激进的消息裁剪，按边界标记删除整个历史段

```
snipCompactIfNeeded()
  │
  ├─ 检测 snip boundary message
  ├─ 移除边界之前的所有消息
  ├─ 清除过期标记
  └─ 防止 mutableMessages 无限增长（SDK 长会话内存泄漏防护）
```

### 6.4 第四层：上下文折叠 (Context Collapse)

**触发时机**: feature flag `CONTEXT_COLLAPSE`，Ant 内部实验
**机制**: 替代 autoCompact 的完整上下文管理系统

- 90% 使用率: 开始提交关键上下文
- 95% 使用率: 阻塞新工具调用

### 6.5 第五层：Reactive Compact

**触发时机**: API 返回 `prompt_too_long` 错误时（413 响应）
**机制**: 作为最后的救生圈，在 API 拒绝请求后紧急压缩

---

## 七、记忆系统：跨会话 Context

### 7.1 自动记忆 (`memdir/memdir.ts`)

记忆系统是 context 跨会话的延伸：

```
记忆目录: ~/.claude/projects/<project-slug>/memory/
  │
  ├─ MEMORY.md (入口索引，≤200行/25KB)
  │   └─ 指向各主题文件的索引
  │
  ├─ user_preferences.md    (type: user)
  ├─ feedback_testing.md    (type: feedback)
  ├─ architecture_decisions.md (type: project)
  └─ api_reference.md       (type: reference)
```

**四种记忆类型**:
- `user`: 用户偏好、工作风格、沟通偏好
- `feedback`: 用户纠正和反馈（"用 bun 不用 npm"）
- `project`: 项目级非代码知识（截止日期、决策原因）
- `reference`: 外部系统指针（仪表盘、Linear 项目）

**注入方式**:
- `MEMORY.md` 通过 `loadMemoryPrompt()` 注入 system prompt 的动态区
- 主题文件通过 `findRelevantMemories()` 语义搜索，每轮注入最相关的 5 个

### 7.2 相关记忆注入 (Relevant Memories)

```
每轮对话开始时:
  │
  startRelevantMemoryPrefetch()
    │
    ├─ 收集最近成功工具调用的语义信号
    ├─ findRelevantMemories() — 在记忆目录中搜索
    │   ├─ 预算: 每轮最多 5 个文件
    │   ├─ 大小: 每个文件 ≤ 4KB
    │   └─ 总量: 每会话 ≤ 60KB
    │
    └─ 作为 <system-reminder> 附件注入
```

### 7.3 Assistant 模式：日志式记忆

在 KAIROS（长期助手）模式下，记忆改为**追加式日志**：
- 按日期写入 `logs/YYYY/MM/YYYY-MM-DD.md`
- 不维护索引，由夜间任务蒸馏为 `MEMORY.md`
- 适合永不结束的会话

---

## 八、关键设计目的与取舍

### 8.1 Prompt Cache 最大化

**目的**: 减少 API 成本和延迟。缓存命中时 token 处理速度快 ~10x，成本降低 ~90%。

**策略**:
1. 工具名按字母排序，保证缓存前缀稳定
2. 静态/动态区分离，静态区可跨用户缓存
3. `systemPromptSection()` 注册机制自动管理缓存失效
4. MCP 指令改用 delta 模式（`mcpInstructionsDelta`），避免每次重连破坏缓存
5. Token 预算说明即使未启用也放入缓存（"When the user specifies..." 空操作措辞）

**取舍**: 
- 静态区的内容必须对所有用户完全相同 → 限制了个性化
- 一些运行时信息（如 enabledTools 集合）被迫放在动态区 → 增加动态区大小
- `DANGEROUS_uncachedSystemPromptSection` 专门标记会破坏缓存的 section

### 8.2 Token 效率

**目的**: 在有限的 context 窗口内塞入最多的有用信息。

**策略**:
1. **工具结果持久化**: 大结果写磁盘，只注入引用
2. **图片压缩**: `maybeResizeAndDownsampleImageBlock()` 自动缩小图片
3. **CLAUDE.md 截断**: 40,000 字符上限
4. **记忆文件截断**: 200 行 / 25KB 上限
5. **Git 状态截断**: 2,000 字符上限
6. **工具结果清除**: 微压缩自动清除旧结果
7. **Ant 内部的数字长度锚点**: "保持工具间文本 ≤25 词"的 system prompt 指令

**取舍**:
- 截断意味着模型可能丢失信息 → 用 "truncated" 标记提醒
- 持久化到磁盘意味着需要额外 Read 操作 → 但节省了 context 空间
- 压缩摘要可能遗漏细节 → 压缩 prompt 设计了 9 个结构化节确保完整性

### 8.3 延迟优化

**目的**: 让用户尽快看到第一个字符。

**策略**:
1. `getUserContext` 和 `getSystemContext` 都 `memoize`，只计算一次
2. Git 状态并行获取（branch + status + log + userName）
3. System prompt 构建并行获取（skills + outputStyle + envInfo）
4. 压缩操作使用 fork 子 Agent，不阻塞主循环
5. 工具执行并发（只读工具并行，写工具顺序）

**取舍**:
- Memoize 意味着 Git 状态是过时的快照 → 用附件机制补偿
- 并行获取增加代码复杂度 → 但显著降低启动延迟

### 8.4 会话连续性

**目的**: 用户关闭终端后能恢复之前的对话上下文。

**策略**:
1. 每条消息 `recordTranscript()` 持久化到 JSONL
2. 用户消息在 API 调用前就持久化（防止中途崩溃丢失）
3. `--resume` 从 JSONL 恢复完整对话
4. 压缩边界（compact_boundary）标记压缩点
5. 会话 ID 贯穿整个生命周期

**取舍**:
- 每条消息都写磁盘增加 I/O → `--bare` 模式下改为 fire-and-forget
- JSONL 文件可能很大 → 压缩后清理旧文件

### 8.5 安全与隔离

**目的**: 防止 context 中的敏感信息泄露。

**策略**:
1. CLAUDE.md 加载时检查 `.gitignore`（不加载被忽略的文件）
2. 本地记忆 `CLAUDE.local.md` 不提交到 Git
3. 图片自动压缩降低隐私风险
4. 工具结果持久化到会话隔离目录
5. `isUndercover()` 模式隐藏模型名称/ID

---

## 九、关键状态流转

### 9.1 Context 状态转换图

```
                    ┌─────────────┐
                    │  空会话开始  │
                    └──────┬──────┘
                           │
                    构建 System Prompt
                    收集 User/System Context
                           │
                           ▼
                    ┌─────────────┐
                    │  Context 就绪 │ ← 每轮 API 调用前的状态
                    │  (Ready)     │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │             │
               正常增长      Token 超阈值
                    │             │
                    ▼             ▼
             ┌──────────┐  ┌───────────┐
             │ 对话进行中 │  │ 压缩触发   │
             │ (Active)  │  │ (Compact) │
             └──────┬───┘
                           │
                    ┌──────┴──────┐
                    │             │
               继续对话        压缩完成
                    │             │
                    ▼             ▼
             ┌──────────┐  ┌───────────┐
             │ 下一轮增长 │  │ Context 缩减│
             │ (Grow)    │  │ (Shrunk)  │
             └──────────┘  └─────┬─────┘
                                 │
                          恢复关键文件
                          注入压缩摘要
                                 │
                                 └──→ 回到 Ready
```

### 9.2 Context 容量管理关键数字

| 参数 | 值 | 说明 |
|---|---|---|
| 默认上下文窗口 | 200K tokens | 大部分模型 |
| 1M 上下文窗口 | 1M tokens | Sonnet 4.6 / Opus 4.6 + beta |
| 自动压缩 Buffer | 13K tokens | 到达 effectiveWindow - 13K 时触发 |
| 警告 Buffer | 20K tokens | UI 黄色警告 |
| 阻塞 Buffer | 3K tokens | 强制压缩 |
| 压缩摘要预留 | 20K tokens | max_output_tokens for compact |
| 压缩后恢复文件数 | 5 个 | 每个文件 ≤ 5K tokens |
| 压缩后技能预算 | 25K tokens | 每个技能 ≤ 5K |
| CLAUDE.md 上限 | 40K 字符 | 超过截断 |
| MEMORY.md 上限 | 200 行 / 25KB | 超过截断并警告 |
| 记忆注入上限 | 5 文件 × 4KB | 每轮最多 |
| 记忆会话上限 | 60KB | 每会话累计 |
| Git 状态上限 | 2K 字符 | 超过截断 |
| 连续压缩失败上限 | 3 次 | 断路器 |
| max_tokens 默认 | 8K / 32K / 64K | 视模型而定 |
| max_tokens 上限 | 64K / 128K | 视模型而定 |

---

## 十、总结

Claude Code 的 Context 管理可以归纳为一条主线：**在有限的 token 窗口内，尽可能多地保留对当前任务有用的信息，同时最小化 API 成本和延迟**。

围绕这条主线，有三个相互制约的目标：

```
        信息完整性
           △
          ╱ ╲
         ╱   ╲
        ╱     ╲
       ╱       ╲
      ╱    成本   ╲
     ╱    效率     ╲
    ▽───────────────▽
        响应延迟
```

- **信息完整性**: 模型需要看到足够的上下文才能正确执行任务
- **成本效率**: context 越大，API 调用越贵；prompt cache 可以大幅降低成本
- **响应延迟**: 首次响应延迟取决于 prompt 处理速度；缓存命中可以降低延迟

Claude Code 的核心策略是用**分层管理**来平衡这三者：

1. **构建时优化**: 静态/动态分离、并行获取、memoize — 降低首次构建成本
2. **运行时增长**: 附件系统按需注入、工具结果持久化 — 精确控制增长
3. **压缩时回收**: 五层压缩从轻到重 — 在信息丢失和空间回收之间找平衡
4. **跨会话延续**: 记忆系统 — 用持久化弥补 context 窗口的时间限制