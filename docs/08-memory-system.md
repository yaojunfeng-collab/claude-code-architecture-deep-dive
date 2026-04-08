# Part 8: 记忆系统 — 如何让 AI 跨会话记住上下文？

> Claude Code 2.1.88 架构深度解析系列 (8/10)

---

## 开场：遗忘是 LLM 的原罪

每次你启动一个新的 Claude Code 会话，模型对你一无所知。它不知道你是 senior engineer 还是实习生，不知道你的项目用 bun 而不是 npm，不知道上周你花了三天修的 auth middleware bug 的来龙去脉。

这不是 bug，这是 LLM 的本质——每次对话都是无状态的。

但优秀的协作者应该 **记住** 你。不仅记住你说过什么，还要记住你是谁、你偏好什么、什么建议被验证过是好的。Claude Code 的记忆系统就是为此而建的。

**核心挑战：如何在无状态的 LLM API 之上，构建一个有状态的、跨会话持久的、且隐私安全的记忆层？**

答案是：文件系统 + 两阶段注入 + 后台提取 agent。

---

## 全景：记忆系统的架构鸟瞰

```
┌───────────────────────────────────────────────────────────────────┐
│                     Memory System 全景                            │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ 存储层 (~/.claude/projects/{slug}/memory/)                   │ │
│  │                                                             │ │
│  │  MEMORY.md (索引文件，非记忆本身)                              │ │
│  │  ├─ user_role.md       (type: user)                         │ │
│  │  ├─ feedback_testing.md (type: feedback)                    │ │
│  │  ├─ project_deadline.md (type: project)                     │ │
│  │  ├─ reference_linear.md (type: reference)                   │ │
│  │  └─ team/              (team memory, feature-gated)         │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                         ↕                                         │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ 注入层 (两阶段)                                               │ │
│  │                                                             │ │
│  │  Phase 1: Session Start (同步)                               │ │
│  │  loadMemoryPrompt() → systemPromptSection('memory')         │ │
│  │  ├─ 行为指令 (类型定义、何时存取、信任规则)                    │ │
│  │  ├─ MEMORY.md 索引内容 (≤200行/25KB)                         │ │
│  │  └─ cached by systemPromptSection                           │ │
│  │                                                             │ │
│  │  Phase 2: Per-Turn Async (异步 prefetch)                     │ │
│  │  startRelevantMemoryPrefetch()                              │ │
│  │  ├─ scanMemoryFiles (≤200 files, 30 lines each)             │ │
│  │  ├─ formatMemoryManifest → selectRelevantMemories           │ │
│  │  │  (Sonnet sideQuery, max 5 files)                         │ │
│  │  ├─ readMemoriesForSurfacing (200 lines/4KB each)           │ │
│  │  └─ → system-reminder injection                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                         ↕                                         │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ 写入层 (两条路径，互斥)                                       │ │
│  │                                                             │ │
│  │  Path A: 主 agent 直接写入                                   │ │
│  │  FileWrite/FileEdit → memory/*.md + MEMORY.md               │ │
│  │                                                             │ │
│  │  Path B: 后台提取 agent                                      │ │
│  │  stopHooks.ts → extractMemories()                           │ │
│  │  → runForkedAgent (maxTurns=5, fire-and-forget void)        │ │
│  │  → 典型 2 turns: read existing → write new                  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

---

## 存储结构

### 目录布局

```
~/.claude/
└── projects/
    └── {slug}/          ← sanitizePath(canonicalGitRoot)
        └── memory/
            ├── MEMORY.md                ← 索引文件 (非记忆本身！)
            ├── user_role.md             ← 独立记忆文件
            ├── feedback_testing.md
            ├── project_auth_rewrite.md
            ├── reference_linear.md
            └── team/                    ← Team memory (feature-gated)
                ├── MEMORY.md
                └── ...
```

### 路径解析

```typescript
// paths.ts:223-235
export const getAutoMemPath = memoize((): string => {
  const override = getAutoMemPathOverride() ?? getAutoMemPathSetting()
  if (override) {
    return override  // 支持 Cowork/SDK 覆盖
  }
  const projectsDir = join(getMemoryBaseDir(), 'projects')
  return (
    join(projectsDir, sanitizePath(getAutoMemBase()), AUTO_MEM_DIRNAME) + sep
  ).normalize('NFC')
}, () => getProjectRoot())
```

[设计决策] 路径使用 `sanitizePath(canonicalGitRoot)` 作为 slug，而不是 cwd。这意味着：
- 同一个 git repo 的不同目录共享记忆
- 同一个 repo 的不同 worktree 共享记忆（通过 `findCanonicalGitRoot`）
- 不同 repo 有独立的记忆空间

### MEMORY.md：索引而非记忆

这是最容易被误解的设计点。MEMORY.md **不是** 记忆内容——它是一个索引文件，每行指向一个具体的记忆文件：

```markdown
- [User Role](user_role.md) — senior backend engineer, Go/Rust focus
- [Testing Policy](feedback_testing.md) — integration tests use real DB, no mocks
- [Auth Rewrite](project_auth_rewrite.md) — legal compliance deadline 2026-03-15
```

[核心概念] MEMORY.md 每行不超过 ~150 chars，总行数上限 200 行，总大小上限 25KB。超出会被截断并附加警告。索引的职责是 **让模型快速扫描全局记忆概览**，而不是存储记忆本身。

```typescript
// memdir.ts:34-36
export const ENTRYPOINT_NAME = 'MEMORY.md'
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000
```

### 记忆文件的 YAML Frontmatter

每个独立记忆文件使用 YAML frontmatter 标注元数据：

```markdown
---
name: Testing Policy
description: Integration tests must hit real DB, no mocks — burned by mock/prod divergence
type: feedback
---

Integration tests in this project must always use a real database connection, never mocks.

**Why:** Q4 2025 incident where mocked tests passed but prod migration failed — mock/prod
divergence masked a broken schema change for 3 days.

**How to apply:** When reviewing or writing tests for any database-touching code, ensure the
test setup creates a real test database rather than using jest mocks or test doubles.
```

---

## 四种记忆类型

### 类型定义

```typescript
// memoryTypes.ts:14-19
export const MEMORY_TYPES = [
  'user',       // 用户信息
  'feedback',   // 行为反馈 (纠正 + 确认)
  'project',    // 项目上下文
  'reference',  // 外部资源指针
] as const
```

### 详细对比

| 维度 | user | feedback | project | reference |
|------|------|----------|---------|-----------|
| **存什么** | 用户角色、目标、技能水平、偏好 | 用户的行为指导（纠正 + 确认） | 进行中的工作、决策、截止日期 | 外部系统的指针 |
| **触发场景** | "I'm a data scientist" / "I've been writing Go for 10 years" | "don't mock the DB" / "yeah the bundled PR was right" | "we're freezing merges after Thursday" | "bugs are in Linear project INGEST" |
| **关键点** | 不存负面评价 | **同时记录纠正和确认！** | **必须用绝对日期** | 存指针而非内容 |
| **衰减速度** | 慢 (角色很少变) | 中 (偏好可能演化) | 快 (项目状态频繁变化) | 中 (URL 可能失效) |
| **body_structure** | 自由格式 | Rule → **Why:** → **How to apply:** | Fact → **Why:** → **How to apply:** | 自由格式 |

### feedback 类型：记录纠正 AND 确认

[核心概念] 这是记忆系统中最具洞察力的设计。源码中明确指出：

```
Record from failure AND success: if you only save corrections, you will
avoid past mistakes but drift away from approaches the user has already
validated, and may grow overly cautious.
```

大多数记忆系统只记录"错误"（用户纠正的地方）。但 Claude Code 同时记录"正确"——当用户说 "yeah the single bundled PR was the right call" 时，这个确认被保存为 feedback 记忆。

**为什么？** 如果只记错误，模型会变得 **过度谨慎**——它知道什么不该做，但忘记了什么是被验证过的好做法。

### project 类型：绝对日期的强制要求

```
Always convert relative dates in user messages to absolute dates when
saving (e.g., "Thursday" → "2026-03-05"), so the memory remains
interpretable after time passes.
```

[可借鉴] 这个约束解决了记忆衰减的经典问题："Thursday" 一周后就无法解读了。强制绝对日期让记忆在时间维度上自解释。

### 不应该保存的内容

```typescript
// memoryTypes.ts:183-195
export const WHAT_NOT_TO_SAVE_SECTION: readonly string[] = [
  '## What NOT to save in memory',
  '',
  '- Code patterns, conventions, architecture, file paths, or project structure',
  '- Git history, recent changes, or who-changed-what',
  '- Debugging solutions or fix recipes',
  '- Anything already documented in CLAUDE.md files.',
  '- Ephemeral task details',
  '',
  'These exclusions apply even when the user explicitly asks you to save.',
]
```

[设计决策] 最后一句话特别重要："这些排除规则即使用户明确要求保存也适用。" 这意味着如果用户说 "remember that the auth module is in src/auth/"，模型应该拒绝（因为这可以从代码中推导出来），而是问用户："这个信息有什么 surprising 或 non-obvious 的部分？那才是值得记住的。"

---

## 两阶段注入

### Phase 1: Session Start（同步加载）

```typescript
// memdir.ts:419-507
export async function loadMemoryPrompt(): Promise<string | null> {
  const autoEnabled = isAutoMemoryEnabled()

  // KAIROS 模式 (长期会话) 使用 daily log 而非 MEMORY.md 索引
  if (feature('KAIROS') && autoEnabled && getKairosActive()) {
    return buildAssistantDailyLogPrompt(skipIndex)
  }

  // TEAMMEM 模式 (团队记忆)
  if (feature('TEAMMEM') && teamMemPaths!.isTeamMemoryEnabled()) {
    await ensureMemoryDirExists(teamDir)
    return teamMemPrompts!.buildCombinedMemoryPrompt(extraGuidelines, skipIndex)
  }

  // 标准模式
  if (autoEnabled) {
    await ensureMemoryDirExists(autoDir)
    return buildMemoryLines('auto memory', autoDir, extraGuidelines, skipIndex).join('\n')
  }

  return null  // 记忆未启用
}
```

Phase 1 在会话启动时同步执行，产出被 `systemPromptSection('memory')` 缓存。它包含：

1. **行为指令**：完整的记忆类型定义、何时存取、信任规则等（~200 行 prompt）
2. **MEMORY.md 索引内容**：截断到 200 行 / 25KB
3. **目录存在性保证**：`ensureMemoryDirExists()` 确保模型可以直接写入

```typescript
// memdir.ts:116-117
export const DIR_EXISTS_GUIDANCE =
  'This directory already exists — write to it directly with the Write tool
   (do not run mkdir or check for its existence).'
```

[可借鉴] 这个 guidance 是典型的"减少模型无用行为"的 prompt engineering——Claude 经常在写文件前先 `ls` 或 `mkdir -p`，浪费一个 tool call。明确告知目录已存在可以省掉这个 round-trip。

### Phase 2: Per-Turn Async Prefetch

Phase 2 是更精密的机制——在每轮对话开始时异步预取相关记忆。

```
┌─────────────────────────────────────────────────────────────┐
│              Per-Turn Async Memory Prefetch                  │
│                                                             │
│  query.ts 开始一轮对话                                       │
│  │                                                          │
│  ├─ startRelevantMemoryPrefetch(messages, context)          │
│  │  │                                                       │
│  │  ├─ Entry checks (4 项)                                  │
│  │  │  ├─ isAutoMemoryEnabled()?                            │
│  │  │  ├─ feature flag 'tengu_moth_copse'?                  │
│  │  │  ├─ 用户消息 > 1 个词? (单词过滤)                      │
│  │  │  └─ surfaced bytes < 60KB? (session 限制)              │
│  │  │                                                       │
│  │  ├─ scanMemoryFiles(memoryDir, signal)                   │
│  │  │  └─ 读取 ≤200 个 .md 文件，每个前 30 行              │
│  │  │     (提取 frontmatter: name, description, type)        │
│  │  │                                                       │
│  │  ├─ formatMemoryManifest(memories)                       │
│  │  │  └─ 格式化为文本清单供 Sonnet 分类器使用               │
│  │  │                                                       │
│  │  ├─ selectRelevantMemories() ← Sonnet sideQuery          │
│  │  │  ├─ model: getDefaultSonnetModel()                    │
│  │  │  ├─ system: SELECT_MEMORIES_SYSTEM_PROMPT             │
│  │  │  ├─ output: JSON schema {selected_memories: string[]} │
│  │  │  ├─ max_tokens: 256                                   │
│  │  │  └─ 返回 ≤5 个文件名                                  │
│  │  │                                                       │
│  │  ├─ readMemoriesForSurfacing(selected, signal)           │
│  │  │  └─ 每个文件读取 ≤200 行 / ≤4KB                       │
│  │  │                                                       │
│  │  └─ → system-reminder injection                          │
│  │     (包含 memoryFreshnessText 陈旧警告)                   │
│  │                                                          │
│  ├─ ... 主 API 请求并行执行 ...                              │
│  │                                                          │
│  └─ consume prefetch result (poll settledAt)                │
│     └─ 如果 prefetch 已完成 → 注入; 否则不阻塞              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

[核心概念] **Memory prefetch 绝不阻塞主流程。** 它作为一个异步操作启动，主 API 请求同时进行。如果 prefetch 在主请求完成前完成，结果被注入；如果没完成，就跳过这一轮。这确保了记忆系统不会增加用户感知的延迟。

### Prefetch Entry Checks（四项前置检查）

```typescript
// attachments.ts:2361-2386
export function startRelevantMemoryPrefetch(
  messages: ReadonlyArray<Message>,
  toolUseContext: ToolUseContext,
): MemoryPrefetch | undefined {
  // Check 1: auto memory 是否启用
  if (!isAutoMemoryEnabled() || !getFeatureValue_CACHED_MAY_BE_STALE('tengu_moth_copse', false)) {
    return undefined
  }

  // Check 2: 找到最后一条真实用户消息
  const lastUserMessage = messages.findLast(m => m.type === 'user' && !m.isMeta)
  if (!lastUserMessage) return undefined

  // Check 3: 单词过滤 — 单词查询缺乏语义信息
  const input = getUserMessageText(lastUserMessage)
  if (!input || !/\s/.test(input.trim())) return undefined

  // Check 4: Session 累计字节限制 (60KB)
  const surfaced = collectSurfacedMemories(messages)
  if (surfaced.totalBytes >= RELEVANT_MEMORIES_CONFIG.MAX_SESSION_BYTES) {
    return undefined
  }
  // ...
}
```

[设计决策] 单词过滤（check 3）看起来简单但有深意。用户输入 "yes" 或 "ok" 时，没有足够的语义信息来做记忆检索——Sonnet sideQuery 几乎一定返回无关结果。跳过这种情况节省了一次 API 调用。

### Sonnet 分类器

```typescript
// findRelevantMemories.ts:18-24
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be
useful to Claude Code as it processes a user's query. You will be given the
user's query and a list of available memory files with their filenames and
descriptions.

Return a list of filenames for the memories that will clearly be useful to
Claude Code as it processes the user's query (up to 5). Only include memories
that you are certain will be helpful...`
```

```typescript
// findRelevantMemories.ts:97-122
const result = await sideQuery({
  model: getDefaultSonnetModel(),
  system: SELECT_MEMORIES_SYSTEM_PROMPT,
  skipSystemPromptPrefix: true,
  messages: [{
    role: 'user',
    content: `Query: ${query}\n\nAvailable memories:\n${manifest}${toolsSection}`,
  }],
  max_tokens: 256,
  output_format: {
    type: 'json_schema',
    schema: {
      type: 'object',
      properties: {
        selected_memories: { type: 'array', items: { type: 'string' } },
      },
      required: ['selected_memories'],
      additionalProperties: false,
    },
  },
  signal,
  querySource: 'memdir_relevance',
})
```

[设计决策] 为什么用 Sonnet 而不是更便宜的模型做分类？因为记忆选择需要 **理解用户意图与记忆描述之间的语义关系**——这不是简单的关键词匹配，而是推理任务。Sonnet 在成本和能力之间取得了最好的平衡。

**Recently used tools 降噪**：

```typescript
// findRelevantMemories.ts:91-95
const toolsSection = recentTools.length > 0
  ? `\n\nRecently used tools: ${recentTools.join(', ')}`
  : ''
```

如果模型正在使用 `mcp__X__spawn`，而记忆中有一个关于该工具的 API 文档，正常情况下会被选中（因为关键词高度匹配）。但此时模型 **已经在使用** 这个工具了——文档是多余的。通过传入 recently used tools，分类器可以跳过这些 "false positive"，但仍然选择包含 warnings/gotchas 的记忆。

### 陈旧性警告

```typescript
// memoryAge.ts:33-42
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''  // 今天或昨天的记忆不警告
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

[可借鉴] 为什么用 "47 days ago" 而不是 ISO timestamp？因为 **模型不擅长日期算术**。给它 "2026-02-19T10:30:00Z"，它很可能不会意识到这是 47 天前的信息。但 "47 days ago" 立刻触发了"这可能过时了"的推理。

### 三重去重

记忆注入有三层去重机制，防止同一记忆被重复展示：

```
Layer 1: Pre-selection (alreadySurfaced)
  findRelevantMemories 接收 alreadySurfaced Set
  → 在 Sonnet 选择之前就过滤掉已展示的记忆
  → 节省 Sonnet 的 5-slot 预算

Layer 2: Post-selection (readFileState)
  消费 prefetch 结果时，过滤掉已在 readFileState 中的路径
  → 防止 prefetch 和其他文件读取机制的重复

Layer 3: Session-level (60KB limit)
  collectSurfacedMemories 统计累计字节数
  → 超过 60KB 时完全停止 prefetch
  → 防止长会话中记忆不断累积
```

```typescript
// attachments.ts:279-288
export const RELEVANT_MEMORIES_CONFIG = {
  MAX_SESSION_BYTES: 60 * 1024,  // 60KB session 累计上限
} as const
```

[权衡] 60KB 的 session 限制意味着大约 15-20 个记忆文件后就不再注入新记忆。这是一个 token 预算 vs 记忆覆盖率的 tradeoff——更多记忆意味着更多 context 占用，更快触发 compaction。

---

## 两条写入路径

### Path A: 主 Agent 直接写入

当模型在对话中决定 "我应该记住这个"，它直接使用 FileWrite/FileEdit 工具：

1. 写入记忆文件 (如 `feedback_testing.md`)
2. 更新 MEMORY.md 索引 (添加一行指针)

这是最直接的路径——模型看到用户说了值得记住的话，主动创建或更新记忆。

### Path B: 后台提取 Agent（核心创新）

```
┌────────────────────────────────────────────────────────────┐
│           Background Memory Extraction (9 步)               │
│                                                            │
│  Step 1: Entry check                                       │
│  └─ isAutoMemoryEnabled() && isExtractModeActive()         │
│     && feature('EXTRACT_MEMORIES')                         │
│                                                            │
│  Step 2: Mutual exclusion                                  │
│  └─ hasMemoryWritesSince(messages, lastMemoryMessageUuid)  │
│     如果主 agent 已经写过记忆 → 跳过，移动游标              │
│                                                            │
│  Step 3: Throttle                                          │
│  └─ turnsSinceLastExtraction < tengu_bramble_lintel?       │
│     默认每轮都提取，feature flag 可配置为每 N 轮            │
│                                                            │
│  Step 4: Pre-scan                                          │
│  └─ scanMemoryFiles → formatMemoryManifest                 │
│     列出现有记忆供 forked agent 参考，避免重复               │
│                                                            │
│  Step 5: Build prompt                                      │
│  └─ buildExtractAutoOnlyPrompt(newMessageCount, manifest)  │
│     "Review the last N messages, extract memories..."      │
│                                                            │
│  Step 6: Forked agent with restricted tools                │
│  └─ runForkedAgent({                                       │
│       canUseTool: createAutoMemCanUseTool(memoryDir),      │
│       maxTurns: 5,                                         │
│       querySource: 'extract_memories',                     │
│       skipTranscript: true,                                │
│     })                                                     │
│                                                            │
│  Step 7: Typical 2 turns                                   │
│  └─ Turn 1: Read existing memories (if any)                │
│     Turn 2: Write new memories + update index              │
│                                                            │
│  Step 8: Result processing                                 │
│  └─ extractWrittenPaths → filter MEMORY.md                 │
│     → createMemorySavedMessage → appendSystemMessage       │
│     → 用户看到 "memory saved" 通知                          │
│                                                            │
│  Step 9: Trailing extraction                               │
│  └─ 如果提取过程中有新的 context 到达                        │
│     → 从 pendingContext 中取出 → 递归执行                   │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### createAutoMemCanUseTool 权限限制

```typescript
// extractMemories.ts:171-222
export function createAutoMemCanUseTool(memoryDir: string): CanUseToolFn {
  return async (tool: Tool, input: Record<string, unknown>) => {
    // REPL: 允许 (REPL 内部仍受限于下面的检查)
    if (tool.name === REPL_TOOL_NAME) return { behavior: 'allow', ... }

    // Read/Grep/Glob: 无限制允许 (只读操作)
    if (tool.name === FILE_READ_TOOL_NAME || ...) return { behavior: 'allow', ... }

    // Bash: 仅允许 read-only 命令 (ls, find, grep, cat, stat...)
    if (tool.name === BASH_TOOL_NAME) {
      if (tool.isReadOnly(parsed.data)) return { behavior: 'allow', ... }
      return denyAutoMemTool(tool, 'Only read-only shell commands...')
    }

    // Edit/Write: 仅允许写入 memory 目录
    if ((tool.name === FILE_EDIT_TOOL_NAME || tool.name === FILE_WRITE_TOOL_NAME)
        && isAutoMemPath(filePath)) {
      return { behavior: 'allow', ... }
    }

    return denyAutoMemTool(tool, ...)
  }
}
```

[设计决策] 后台提取 agent 被严格限制在 "只读 + 只写 memory 目录"。这是安全边界——一个 fire-and-forget 的后台 agent 绝对不能修改用户的代码或执行任意命令。这个限制通过 `canUseTool` 函数在 tool 层面强制执行，而非靠 prompt 指令。

### 互斥机制

```typescript
// extractMemories.ts:346-360
// Mutual exclusion: when the main agent wrote memories, skip the
// forked agent and advance the cursor past this range
if (hasMemoryWritesSince(messages, lastMemoryMessageUuid)) {
  logForDebugging('[extractMemories] skipping — conversation already wrote to memory files')
  const lastMessage = messages.at(-1)
  if (lastMessage?.uuid) {
    lastMemoryMessageUuid = lastMessage.uuid
  }
  return
}
```

[核心概念] Path A 和 Path B 是 **互斥** 的。当主 agent 在这一轮中已经写过记忆文件，后台提取 agent 就跳过——因为主 agent 已经处理了记忆提取的职责。`hasMemoryWritesSince` 检查 assistant messages 中是否有 FileWrite/FileEdit 操作指向 memory 目录。

这个互斥是 **per-range** 的：游标 `lastMemoryMessageUuid` 会前进到最新消息，下一次只检查新增的消息范围。

### Fire-and-Forget 模式

```typescript
// stopHooks.ts 中触发 (概念代码)
void extractMemories(context, appendSystemMessage)
// ↑ void — 不 await，不阻塞用户
```

后台提取是 `void`（fire-and-forget）。它在主 query loop 的 stop hook 中触发，但不阻塞下一轮对话。即使提取失败，用户也不会感知到。

---

## MEMORY.md 更新机制

### 索引而非记忆

再次强调：MEMORY.md 是 **索引**，不是记忆内容。

```typescript
// memdir.ts:226-228
`**Step 2** — add a pointer to that file in \`${ENTRYPOINT_NAME}\`.
 \`${ENTRYPOINT_NAME}\` is an index, not a memory — each entry should be
 one line, under ~150 characters`
```

### 四种操作

| 操作 | 说明 | 示例 |
|------|------|------|
| **add** | 新增一行指针 | `- [New Topic](new_topic.md) — one-line hook` |
| **modify** | 更新现有行的描述 | 修改 hook 文字以反映记忆内容的变化 |
| **delete** | 删除一行 | 当对应的记忆文件被删除时 |
| **merge** | 合并重复条目 | 两个相似记忆文件合并后更新索引 |

### skipIndex 模式（tengu_moth_copse）

```typescript
// memdir.ts:422-425
const skipIndex = getFeatureValue_CACHED_MAY_BE_STALE(
  'tengu_moth_copse',
  false,
)
```

当 feature flag `tengu_moth_copse` 开启时，MEMORY.md 索引不再被使用。取而代之的是：
- 记忆文件直接写入目录（无索引步骤）
- 发现靠 scanMemoryFiles 全目录扫描
- Phase 2 prefetch 变为主要的记忆发现机制

这代表了一个架构演进方向：从 **索引式**（MEMORY.md 主导）到 **扫描式**（frontmatter + sideQuery 主导）。

[设计决策] 为什么要演进？因为索引维护是一个脆弱的操作——模型可能忘记更新索引、索引可能与实际文件不同步、索引占据宝贵的 system prompt 空间。而扫描式方法虽然启动时多一次 IO 操作，但 **自然保持一致性**（文件就是真相来源）。

---

## CLAUDE.md vs MEMORY.md 完整对比

| 维度 | CLAUDE.md | MEMORY.md (记忆系统) |
|------|-----------|---------------------|
| **谁写** | 人类开发者 | AI agent (主 agent 或后台提取 agent) |
| **内容性质** | 项目指令、约定、规则 | 跨会话的上下文、偏好、观察 |
| **Git 管理** | 通常纳入 git | 不纳入 git (在 ~/.claude/ 中) |
| **加载时机** | 每次会话启动 (sync) | 会话启动 (索引) + 每轮 prefetch (内容) |
| **信任级别** | 高 (人类编写+review) | 中 (AI 自动提取，可能有误) |
| **更新频率** | 低 (人工编辑) | 高 (每轮对话可能触发) |
| **大小限制** | 无硬限制 (但影响 token) | MEMORY.md: 200行/25KB, 单文件: 200行/4KB |
| **目录层次** | repo/.claude/, ~/.claude/ | ~/.claude/projects/{slug}/memory/ |
| **作用域** | 项目/用户/本地三级 | 项目级 (按 git root 隔离) |
| **主要目的** | 告诉 AI "怎么做" | 告诉 AI "关于用户和项目的背景" |

---

## 六层持久化层次

Claude Code 的持久化知识来自六个层次，从高信任到低信任排列：

```
┌───────────────────────────────────────────────────────────┐
│ Layer 1: Managed Instructions (最高优先级)                 │
│ 来源: Anthropic 控制的系统策略                              │
│ 加载: systemPromptSection('managed')                      │
│ 信任: 最高 (不可被用户覆盖)                                │
├───────────────────────────────────────────────────────────┤
│ Layer 2: User CLAUDE.md                                   │
│ 来源: ~/.claude/CLAUDE.md                                 │
│ 加载: getMemoryFiles → getUserContext                     │
│ 信任: 高 (用户全局配置)                                    │
├───────────────────────────────────────────────────────────┤
│ Layer 3: Project CLAUDE.md                                │
│ 来源: {repo}/.claude/CLAUDE.md + CLAUDE.md                │
│ 加载: getMemoryFiles (向上遍历到 git root)                 │
│ 信任: 中高 (可能来自第三方 repo)                            │
├───────────────────────────────────────────────────────────┤
│ Layer 4: Local CLAUDE.md                                  │
│ 来源: CLAUDE.local.md (gitignored)                        │
│ 加载: getMemoryFiles                                      │
│ 信任: 高 (个人本地配置，不纳入 git)                         │
├───────────────────────────────────────────────────────────┤
│ Layer 5: Auto Memory (记忆系统)                            │
│ 来源: ~/.claude/projects/{slug}/memory/*.md               │
│ 加载: Phase 1 (索引) + Phase 2 (prefetch)                 │
│ 信任: 中 (AI 自动提取，带陈旧性警告)                        │
├───────────────────────────────────────────────────────────┤
│ Layer 6: Team Memory (团队记忆, feature-gated)             │
│ 来源: ~/.claude/projects/{slug}/memory/team/*.md          │
│ 加载: 同 Layer 5                                          │
│ 信任: 中 (AI 提取 + 团队共享)                               │
└───────────────────────────────────────────────────────────┘
```

```typescript
// claudemd.ts:790 (概念)
export const getMemoryFiles = memoize(
  async (forceIncludeExternal: boolean = false): Promise<MemoryFileInfo[]> => {
    // 遍历 CWD 到 git root 的所有 CLAUDE.md, CLAUDE.local.md
    // + ~/.claude/CLAUDE.md
    // + auto memory MEMORY.md (如果启用)
    // ...
  }
)
```

---

## 为什么需要两个系统？

CLAUDE.md 和记忆系统解决的是 **不同层面** 的问题。以下是六个维度的分析：

### 1. 不同的问题域

| CLAUDE.md | 记忆系统 |
|-----------|---------|
| "这个项目用 bun 而不是 npm" | "这个用户是 senior Go 工程师，刚接触 React" |
| **项目知识** (what to do) | **用户知识** (who you're working with) |

### 2. 不同的信任边界

CLAUDE.md 可以包含在 git repo 中，被整个团队 review。记忆系统是 AI 自动生成的，质量参差不齐，需要陈旧性警告。

### 3. 不同的变更频率

CLAUDE.md 变化缓慢（人类编辑，经过 PR review）。记忆可能每轮对话都更新。这两种变更频率需要不同的存储和缓存策略。

### 4. 不同的作用域需求

CLAUDE.md 可以是全局的（~/.claude/CLAUDE.md）、项目级的、甚至是目录级的。记忆是严格项目级的（按 git root 隔离），因为记忆的语义依赖于项目上下文。

### 5. 不同的 Token 预算策略

CLAUDE.md 在 system prompt 中，享有 prompt cache 的长期命中。记忆通过 per-turn prefetch 按需注入，且有 60KB 的 session 上限。两者的 budget 策略完全不同。

### 6. 互补而非重叠

[核心概念] CLAUDE.md 告诉 AI "在这个项目中应该怎么做"。记忆系统告诉 AI "关于这个用户和项目的上下文"。前者是 **指令**，后者是 **知识**。`remember` skill 正是这两者之间的桥梁——它检查记忆中是否有内容应该"晋升"为 CLAUDE.md 中的正式指令。

---

## "Before recommending from memory" — 信任机制

记忆系统包含一个精妙的信任校准层：

```typescript
// memoryTypes.ts:240-256
export const TRUSTING_RECALL_SECTION: readonly string[] = [
  '## Before recommending from memory',
  '',
  'A memory that names a specific function, file, or flag is a claim that it
   existed *when the memory was written*. It may have been renamed, removed,
   or never merged. Before recommending it:',
  '',
  '- If the memory names a file path: check the file exists.',
  '- If the memory names a function or flag: grep for it.',
  '- If the user is about to act on your recommendation: verify first.',
  '',
  '"The memory says X exists" is not the same as "X exists now."',
]
```

[设计决策] 这段 prompt 是经过 eval 验证的（memory-prompt-iteration case 3, 0/2 → 3/3）。关键洞察是：

1. 记忆中的 `file:line` 引用会给模型虚假的"确定感"——有了具体引用，模型更倾向于断言而非验证
2. 将这段内容放在独立的 `## Before recommending from memory` section 比放在 `## When to access memories` 的 bullet 中效果更好——因为它描述的是 "how to use memory" 而非 "when to access"，位置上需要独立的触发上下文

---

## 关键开关

| 开关 | 类型 | 作用 | 优先级 |
|------|------|------|--------|
| `autoMemoryEnabled` | settings.json | 全局开关 | 最低 |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | env var | 禁用 auto memory | 最高 (1/true → OFF, 0/false → ON) |
| `CLAUDE_CODE_SIMPLE` (--bare) | env var | 极简模式，禁用记忆 | 高 |
| `CLAUDE_CODE_REMOTE` | env var | 远程模式 (需要 REMOTE_MEMORY_DIR) | 条件性 |
| `tengu_moth_copse` | feature flag | skipIndex 模式 (扫描式) | AB test |
| `tengu_passport_quail` | feature flag | 后台提取是否启用 | AB test |
| `tengu_bramble_lintel` | feature flag | 提取频率 (每 N 轮) | AB test |
| `tengu_herring_clock` | feature flag | Team memory 是否启用 | AB test |

```typescript
// paths.ts:30-55
export function isAutoMemoryEnabled(): boolean {
  // 1. env var 最高优先级
  if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY)) return false
  if (isEnvDefinedFalsy(envVal)) return true

  // 2. --bare 模式
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) return false

  // 3. 远程模式需要显式配置
  if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE)
      && !process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR) return false

  // 4. settings.json
  if (settings.autoMemoryEnabled !== undefined) return settings.autoMemoryEnabled

  // 5. 默认启用
  return true
}
```

---

## 常见误区

### 误区 1："MEMORY.md 就是记忆"

**正确**：MEMORY.md 是 **索引文件**。每行是一个指向独立记忆文件的指针。把记忆内容直接写入 MEMORY.md 是错误的——这会导致索引膨胀、超出 200 行限制后被截断。

### 误区 2："记忆系统会记住所有对话内容"

**正确**：记忆系统只保存 **不可从代码/git 推导的信息**。代码模式、文件路径、git 历史等都不应该被保存——它们是 derivable 的。记忆关注的是用户、偏好、项目上下文、外部资源等 non-derivable 信息。

### 误区 3："后台提取 agent 可能会修改我的代码"

**正确**：后台提取 agent 被 `createAutoMemCanUseTool` 严格限制——只能读取任意文件，只能写入 memory 目录。即使 prompt 中有恶意指令（prompt injection），tool 层面的硬限制确保它无法修改 memory 目录外的任何文件。

### 误区 4："记忆越多越好"

**正确**：记忆有 60KB 的 session 累计上限、每文件 4KB 的注入上限、每轮最多 5 个文件的选择上限。过多的记忆会稀释相关性——模型需要在 5 个 slot 中选择最相关的，slot 越紧张，选择质量越低。保持记忆精简和高信噪比更重要。

### 误区 5："CLAUDE.md 和记忆系统可以互相替代"

**正确**：它们服务于不同目的。CLAUDE.md 是人类编写的 **指令**（告诉 AI 怎么做），记忆是 AI 提取的 **知识**（告诉 AI 关于用户和项目的背景）。`/remember` skill 的存在恰恰说明需要一个机制来在两者之间"晋升"内容。

### 误区 6："记忆系统是一个数据库"

**正确**：记忆系统完全基于文件系统——Markdown 文件 + YAML frontmatter。没有数据库、没有向量索引、没有复杂的查询引擎。检索靠 Sonnet sideQuery 的语义理解。这种简单性是刻意的——文件系统是最可靠的持久化层，用户可以直接 `cat` 或 `vim` 来检查和编辑记忆。

---

## 跨模块联系

### Memory × Skill 系统

`remember` skill 是两个系统的桥梁：
- 使用 Skill 的模块化 prompt 能力来定义审查流程
- 操作 Memory 的文件结构来审查和"晋升"记忆到 CLAUDE.md

### Memory × Compaction

Compaction 会清除 `systemPromptSection` 和 `getUserContext` 的缓存（`postCompactCleanup.ts`），触发 Phase 1 的重新加载。同时 Phase 2 的 prefetch 因为基于消息扫描，compaction 后自然重置（旧 attachment 消失，surfaced bytes 归零）。

### Memory × Agent 系统

后台提取本质上是一个 forked agent（`runForkedAgent`）——它共享主 agent 的 prompt cache 前缀，使用相同的 system prompt + tools schema。这使得提取 agent 的第一次 API 调用几乎全部命中 cache（`cache_read_input_tokens`），大幅降低成本。

### Memory × Permission 系统

`createAutoMemCanUseTool` 是 permission 系统的一个特殊实例——它不走标准的 deny/allow rule 检查，而是硬编码了一个极度限制性的 allowlist。这是因为后台 agent 是无人监督的（fire-and-forget），必须从 tool 层面确保安全。

### Memory × CLAUDE.md

`getMemoryFiles`（在 claudemd.ts 中）统一管理 CLAUDE.md 文件和 MEMORY.md 的加载。MEMORY.md 的索引内容作为六层持久化层次的一部分，被 `getUserContext` 包装进 system prompt 的 user-context section。

---

## 核心 Takeaway

1. **记忆系统 = 文件系统 + 两阶段注入 + 后台提取**。没有数据库，没有向量搜索，纯粹基于 Markdown 文件和 Sonnet 的语义理解。

2. **四种记忆类型** 覆盖了非代码可推导的全部知识维度。特别注意 feedback 要 **同时记录纠正和确认**，project 要 **强制绝对日期**。

3. **两阶段注入确保了零延迟**：Phase 1 同步加载索引（cached），Phase 2 异步 prefetch 内容（never blocks）。

4. **两条写入路径互斥**：主 agent 写过就跳过后台提取，避免重复。后台提取被 tool-level 硬限制在 memory 目录内。

5. **MEMORY.md 是索引，不是记忆**。正在从"索引式"（MEMORY.md 主导）向"扫描式"（frontmatter + sideQuery）演进。

6. **CLAUDE.md 和记忆系统是互补的**：一个是人写的指令，一个是 AI 提取的知识。`/remember` skill 是它们之间的桥梁。

7. **陈旧性管理** 是记忆系统的隐藏复杂度：>1 天的记忆自动附带警告，模型被指导 "verify before asserting"。

8. **60KB session 上限 + 每轮 5 files + 4KB per file** = 精确控制记忆的 token 预算，在信息丰富度和 context 占用之间取得平衡。

---

## 思考题

1. **索引式 vs 扫描式的 tradeoff**：`tengu_moth_copse` flag 控制的 skipIndex 模式跳过 MEMORY.md 索引，改用全目录扫描 + Sonnet 选择。这样做的优缺点是什么？在什么规模下扫描式会变得不可行？

2. **记忆衰减策略**：当前系统只有陈旧性警告（>1 天提示"可能过时"），但没有自动清理。如果一个记忆 365 天没被访问，它还占据磁盘空间和 scan 时间。你会如何设计一个自动衰减机制？

3. **后台提取的成本问题**：每次 query 结束后都可能触发一次 Sonnet API 调用（forked agent）。假设用户每天 100 轮对话，后台提取的成本是多少？如何优化？

4. **记忆冲突解决**：如果 CLAUDE.md 说"用 npm"，而记忆系统中有一条 feedback 说"用户纠正我改用 bun"。模型应该遵循哪个？当前系统如何处理这种冲突？

5. **跨项目记忆**：当前记忆按 git root 隔离。但有些知识是跨项目的（如用户偏好"I prefer concise responses"）。User type 的记忆天然是跨项目的——如何设计一个机制来在项目级记忆中发现并"晋升"跨项目记忆到全局层？

6. **Sonnet 分类器的 failure mode**：如果 sideQuery 超时或返回错误结果怎么办？当前的 catch 块直接返回空数组。这意味着一个 Sonnet 故障 = 整个 prefetch 失效。你会如何增加 fallback 机制（例如 keyword-based 降级）？

---

## 关键源文件

| 文件 | 职责 |
|------|------|
| `src/memdir/memdir.ts` | 记忆系统核心：loadMemoryPrompt, buildMemoryLines, buildMemoryPrompt, truncateEntrypointContent |
| `src/memdir/memoryTypes.ts` | 四种记忆类型定义、frontmatter 示例、WHAT_NOT_TO_SAVE、TRUSTING_RECALL |
| `src/memdir/paths.ts` | 路径解析：getAutoMemPath, isAutoMemoryEnabled, isAutoMemPath, isExtractModeActive |
| `src/memdir/memoryScan.ts` | 文件扫描：scanMemoryFiles (200 files/30 lines), formatMemoryManifest |
| `src/memdir/findRelevantMemories.ts` | Sonnet 分类器：selectRelevantMemories, SELECT_MEMORIES_SYSTEM_PROMPT |
| `src/memdir/memoryAge.ts` | 陈旧性管理：memoryAgeDays, memoryAge, memoryFreshnessText |
| `src/utils/attachments.ts` | Per-turn prefetch：startRelevantMemoryPrefetch, readMemoriesForSurfacing, RELEVANT_MEMORIES_CONFIG |
| `src/services/extractMemories/extractMemories.ts` | 后台提取：initExtractMemories, runExtraction, createAutoMemCanUseTool, 9-step flow |
| `src/query/stopHooks.ts` | 提取触发点：handleStopHooks → extractMemories (fire-and-forget) |
| `src/utils/claudemd.ts` | 六层持久化：getMemoryFiles, resetGetMemoryFilesCache |
| `src/services/compact/postCompactCleanup.ts` | Compaction 后的缓存清理：resetGetMemoryFilesCache, getUserContext.cache.clear |
| `src/skills/bundled/remember.ts` | CLAUDE.md ↔ Memory 桥梁：remember skill |
