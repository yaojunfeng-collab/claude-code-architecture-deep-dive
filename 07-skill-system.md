# Part 7: Skill 系统 — Prompt Engineering 可以模块化吗？

> Claude Code 2.1.88 架构深度解析系列 (7/10)

---

## 开场：一个被误解的系统

想象一下这个场景：你在 Claude Code 中输入 `/simplify`，然后 AI 自动 review 你的代码改动、启动三个并行 agent 分别做 reuse/quality/efficiency 检查、最后汇总修复。整个过程看起来像是调用了一个"插件"或者"脚本"。

但如果你去看 simplify 的源码，你会发现它的核心只有一个 Markdown 字符串——一段精心编写的 prompt。没有任何代码逻辑、没有 API 调用、没有状态管理。

**这就是 Skill 系统最深层的本质：它不是代码执行引擎，而是一个模块化的 Prompt Engineering 框架。**

[核心概念] 如果把 Agent 比作工人（worker），那 Skill 就是 SOP 手册（Standard Operating Procedure）。工人拿到手册后自己决定怎么执行，手册本身不包含任何执行逻辑。

这个理解非常关键。很多人以为 Skill 是某种 code plugin system，试图在 SKILL.md 里写条件判断、循环逻辑——这完全搞错了方向。Skill 的全部威力来自 prompt engineering：通过精确的自然语言指令，引导模型按照特定的步骤和约束完成复杂任务。

---

## 全景：Skill 系统的完整生命周期

```
┌─────────────────── Skill 全生命周期 ───────────────────┐
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Phase 0: Discovery (系统启动时)                    │  │
│  │  bundledSkills + loadSkillsDir + MCP skills       │  │
│  │  → system-reminder 注入 skill_listing             │  │
│  │  (预算: contextWindow × 1%)                       │  │
│  └──────────────────────────────────────────────────┘  │
│                         ↓                               │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Phase 1: Model Decision                           │  │
│  │  模型看到 system-reminder 中的 skill 列表          │  │
│  │  + SkillTool 的 description 中的 BLOCKING REQ     │  │
│  │  → 决定调用 SkillTool({skill:"simplify"})         │  │
│  └──────────────────────────────────────────────────┘  │
│                         ↓                               │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Phase 2: SkillTool.call()                         │  │
│  │  validate → checkPermissions (3-level) → mode判断 │  │
│  │  → fork? → executeForkedSkill                     │  │
│  │  → inline? → processPromptSlashCommand            │  │
│  └──────────────────────────────────────────────────┘  │
│                         ↓                               │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Phase 3: Load SKILL.md                            │  │
│  │  getPromptForCommand(args, context)               │  │
│  │  → substituteArguments ($ARGUMENTS, $arg_name)    │  │
│  │  → template vars (${CLAUDE_SKILL_DIR}, etc.)      │  │
│  │  → embedded shell (!command)                      │  │
│  └──────────────────────────────────────────────────┘  │
│                         ↓                               │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Phase 4-5: Injection & Return                     │  │
│  │  构建 newMessages (user meta messages)            │  │
│  │  + contextModifier (allowedTools, model, effort)  │  │
│  │  → return {data, newMessages, contextModifier}    │  │
│  └──────────────────────────────────────────────────┘  │
│                         ↓                               │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Phase 6-7: Main Loop Applies                      │  │
│  │  newMessages 追加到 conversation                   │  │
│  │  contextModifier 修改 toolUseContext               │  │
│  │  → 模型继续处理，把 skill prompt 当指令执行        │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 深入：Step 0 — Skill Discovery

### 模型如何知道有哪些 Skill？

Skill 发现机制依赖于 **system-reminder 注入**。每一轮对话开始时，系统会在 attachment messages 中注入一个 `skill_listing`，列出所有可用的 skill 名称和简短描述。

```typescript
// src/tools/SkillTool/prompt.ts
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01  // 上下文窗口的 1%
export const CHARS_PER_TOKEN = 4
export const DEFAULT_CHAR_BUDGET = 8_000          // Fallback: 1% of 200k × 4
export const MAX_LISTING_DESC_CHARS = 250          // 每个 entry 描述上限
```

[设计决策] 为什么只给 1% 的预算？这是一个精确的 tradeoff：

- **太多**：skill 列表占据大量 turn-1 cache_creation tokens，而且这些 tokens 每次都在 system prompt 中，造成持续的成本
- **太少**：模型无法发现足够的 skill 来匹配用户意图
- **1%**：对于 200K context window，大约 8000 chars，足以列出 ~30 个 skill 的名称和简短描述

### 预算内的降级策略

当 skill 数量增长，超出 budget 时，`formatCommandsWithinBudget` 实现了三级降级：

```
Level 1: 完整描述 (name + description + whenToUse)
  ↓ 超出 budget
Level 2: 截断描述 (bundled skill 保持完整，其余截断)
  ↓ 仍然超出
Level 3: 纯名称 (bundled skill 保持完整，其余只显示 name)
```

```typescript
// prompt.ts:92-142
// Partition into bundled (never truncated) and rest
const bundledIndices = new Set<number>()
// ...
// bundled skills always get full descriptions
if (bundledIndices.has(i)) return fullEntries[i]!.full
```

[设计决策] Bundled skill 的描述永远不被截断。这是因为 bundled skill 是 Anthropic 官方维护的核心功能（simplify, verify, remember 等），截断它们的描述会严重降低匹配率。第三方 skill 的发现则可以依赖用户主动输入 `/skill-name`。

---

## 深入：Step 1 — Model Decision（BLOCKING REQUIREMENT）

SkillTool 的 prompt 中包含一个关键的行为约束：

```typescript
// prompt.ts:190-191
"When a skill matches the user's request, this is a BLOCKING REQUIREMENT:
 invoke the relevant Skill tool BEFORE generating any other response about the task"
```

这个 "BLOCKING REQUIREMENT" 是 Skill 系统最重要的 prompt engineering 技巧之一。没有它，模型很可能会：
1. 先自己尝试完成任务
2. 做到一半发现有个 skill 可以用
3. 再调用 skill，但此时已经浪费了 tokens 和用户时间

[可借鉴] "BLOCKING REQUIREMENT" 模式：当你设计 tool/skill 系统时，如果某些 tool 应该优先被考虑，需要在 prompt 中明确用 **blocking/priority 语言** 而不是 "you can"/"you may" 这样的 permissive 语言。

### `<command-name>` Tag 防止重复调用

```typescript
// prompt.ts:194
`If you see a <${COMMAND_NAME_TAG}> tag in the current conversation turn,
 the skill has ALREADY been loaded - follow the instructions directly
 instead of calling this tool again`
```

当 skill 被加载后，会在注入的消息中包含 `<command-name>simplify</command-name>` 这样的 tag。模型看到这个 tag 就知道 skill 已经加载完毕，直接执行指令即可。

[权衡] 这种 dedup 机制是 **模型自律型** 的——它依赖模型遵循 prompt 指令，而不是 API 层面的强制限制。因为 SkillTool 返回的是 `newMessages`，一旦注入到 conversation 中，framework 层面无法阻止模型再次调用 SkillTool。这是一个有意的 tradeoff：强制阻止会增加系统复杂度，而模型自律在实践中足够可靠。

---

## 深入：Step 2 — SkillTool.call() 的完整链路

### 三级权限检查

```typescript
// SkillTool.ts:432-578 checkPermissions
async checkPermissions({ skill, args }, context): Promise<PermissionDecision> {
  // Level 1: Deny rules (最高优先级)
  const denyRules = getRuleByContentsForTool(permissionContext, SkillTool, 'deny')
  for (const [ruleContent, rule] of denyRules.entries()) {
    if (ruleMatches(ruleContent)) {
      return { behavior: 'deny', ... }
    }
  }

  // Level 2: Allow rules
  const allowRules = getRuleByContentsForTool(permissionContext, SkillTool, 'allow')
  for (const [ruleContent, rule] of allowRules.entries()) {
    if (ruleMatches(ruleContent)) {
      return { behavior: 'allow', ... }
    }
  }

  // Level 3: Safe properties auto-allow
  if (commandObj?.type === 'prompt' && skillHasOnlySafeProperties(commandObj)) {
    return { behavior: 'allow', ... }
  }

  // Default: ask user
  return { behavior: 'ask', ... }
}
```

**Safe properties 白名单机制** 特别值得注意：

```typescript
// SkillTool.ts:875-908
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'model', 'effort',
  'source', 'name', 'description', 'aliases', 'whenToUse',
  // ... etc
])
```

[设计决策] 这是一个 allowlist 而非 blocklist 策略：如果一个 skill 只包含安全属性（基本的元数据字段），则自动允许执行。**任何未来新增的 property 都默认需要权限**——这是 secure-by-default 的典型实践。关键的不安全属性包括 `allowedTools`（因为它可以授予额外的工具权限）和 `hooks`（因为它可以注入 shell 命令）。

### Validate → Permission → Execute 的流程

```
validateInput()
  ├─ skill 名称非空？
  ├─ 去掉前导 /
  ├─ 命令是否存在？(findCommand)
  ├─ disableModelInvocation 不为 true？
  └─ type === 'prompt'？

checkPermissions()
  ├─ deny rules 匹配 → 拒绝
  ├─ allow rules 匹配 → 允许
  ├─ safe properties → 自动允许
  └─ default → 询问用户

call()
  ├─ context === 'fork' → executeForkedSkill()
  └─ default (inline) → processPromptSlashCommand()
```

---

## 深入：Step 3 — SKILL.md 加载与模板处理

### Bundled Skill 的注册方式

```typescript
// bundledSkills.ts
export type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  context?: 'inline' | 'fork'
  files?: Record<string, string>
  getPromptForCommand: (args: string, context: ToolUseContext)
    => Promise<ContentBlockParam[]>
}
```

核心是 `getPromptForCommand`——一个 async 函数，接收 args 和 context，返回 `ContentBlockParam[]`（即一组 text blocks）。

对于 bundled skill，prompt 内容是硬编码在 TypeScript 中的字符串。例如 simplify：

```typescript
// simplify.ts
const SIMPLIFY_PROMPT = `# Simplify: Code Review and Cleanup
Review all changed files for reuse, quality, and efficiency...

## Phase 2: Launch Three Review Agents in Parallel
Use the ${AGENT_TOOL_NAME} tool to launch all three agents concurrently...
`
```

注意 `${AGENT_TOOL_NAME}` 是 TypeScript template literal，在编译时就已替换。

### 磁盘 Skill 的模板变量

对于 `.claude/skills/` 目录下的用户自定义 skill，加载时支持以下模板变量：

| 变量 | 含义 | 替换时机 |
|------|------|---------|
| `$ARGUMENTS` | 用户传入的参数 | processSlashCommand |
| `$arg_name` | 命名参数 | substituteArguments |
| `${CLAUDE_SKILL_DIR}` | Skill 文件所在目录 | loadSkillsDir / executeRemoteSkill |
| `${CLAUDE_SESSION_ID}` | 当前会话 ID | loadSkillsDir / executeRemoteSkill |
| `!command` | 嵌入式 shell 命令 | processSlashCommand (执行后替换为输出) |

### Bundled Skill 的 `files` 机制

```typescript
// bundledSkills.ts:35-37
files?: Record<string, string>
// Keys are relative paths, values are content.
// Extracted to disk on first invocation via extractBundledSkillFiles
```

某些 bundled skill（如 verify）需要附带参考文件。`files` 字段允许将这些文件打包在代码中，首次调用时解压到临时目录：

```typescript
// bundledSkills.ts:120-145
export function getBundledSkillExtractDir(skillName: string): string {
  return join(getBundledSkillsRoot(), skillName)
}

// 解压后在 prompt 前面加上 base directory
function prependBaseDir(blocks, baseDir) {
  const prefix = `Base directory for this skill: ${baseDir}\n\n`
  // ...
}
```

[设计决策] 为什么要解压到磁盘而不是直接在 prompt 里内联？因为参考文件可能很大（verify 的 schema 文件等），全部内联会浪费 token。解压到磁盘后，模型可以按需 Read/Grep，只加载实际需要的部分。

---

## 深入：Step 4-5 — 构建注入消息

### Inline 模式的返回结构

```typescript
// SkillTool.ts:767-841
return {
  data: {
    success: true,
    commandName,
    allowedTools: allowedTools.length > 0 ? allowedTools : undefined,
    model,
  },
  newMessages,              // ← 注入到 conversation 的新消息
  contextModifier(ctx) {    // ← 修改后续的 toolUseContext
    let modifiedContext = ctx

    // 1. 更新 allowed tools (Set 累加，只增不减)
    if (allowedTools.length > 0) {
      // ... merge into alwaysAllowRules.command
    }

    // 2. 模型覆盖
    if (model) {
      modifiedContext = { ...modifiedContext, options: {
        ...modifiedContext.options,
        mainLoopModel: resolveSkillModelOverride(model, ctx.options.mainLoopModel),
      }}
    }

    // 3. Effort level 覆盖
    if (effort !== undefined) {
      // ... override effortValue
    }

    return modifiedContext
  },
}
```

这个返回结构是 Skill 系统最精妙的设计之一。它有三个维度的副作用：

1. **newMessages**：注入 skill prompt 作为 user messages（标记为 `isMeta: true`）
2. **data**：告诉 ToolResult 渲染器显示什么（`Launching skill: simplify`）
3. **contextModifier**：一个闭包，在后续每次 main loop 迭代时修改 context

### "One Skill call = two API requests" 解释

这里有一个容易被忽视的性能特征：

```
Request 1: 模型决定调用 SkillTool
  → API call (模型输出 tool_use block)
  → SkillTool.call() 执行 (同步，几乎零耗时)
  → 返回 newMessages + contextModifier

Request 2: 模型处理 skill prompt
  → newMessages 被追加到 conversation
  → 再次调用 API (模型看到 skill prompt 并开始执行)
  → 这才是真正的 "work"
```

[权衡] 一次 Skill 调用实际消耗两次 API 请求。第一次只是"选择调用哪个 skill"，第二次才是"执行 skill 的指令"。这是 tool-use 架构的固有开销。直接由用户 `/simplify` 触发时，可以跳过第一次 API 请求，直接注入 prompt——这就是下面要讲的两种调用路径的区别。

---

## 两种调用路径对比

### 用户 /skill-name vs 模型 SkillTool

| 维度 | 用户 `/skill-name` | 模型 `SkillTool({skill:"name"})` |
|------|--------------------|---------------------------------|
| 触发方式 | 用户在输入框键入 | 模型在 tool_use block 中输出 |
| 入口函数 | processSlashCommand() | SkillTool.call() → processSlashCommand() |
| API 请求数 | 1 (直接注入 prompt) | 2 (选择 + 执行) |
| 权限检查 | 无 (用户主动触发) | checkPermissions (3-level) |
| 发现依赖 | 无 (用户已知 skill 名称) | 依赖 system-reminder 中的 skill_listing |
| validateInput | 无 | 有 (类型检查、存在性检查等) |
| disableModelInvocation | 不影响 | 会阻止调用 |

[可借鉴] 双入口模式在很多 LLM 应用中都有价值：用户直接触发时跳过发现和权限步骤，AI 触发时走完整链路。这既保证了用户操作的流畅性，又维护了 AI 自主行为的安全边界。

---

## 两种执行模式：Inline vs Fork

### 模式选择

```typescript
// SkillTool.ts:622-632
// Check if skill should run as a forked sub-agent
if (command?.type === 'prompt' && command.context === 'fork') {
  return executeForkedSkill(command, commandName, args, context, ...)
}
// Otherwise inline (default)
```

### 对比表

| 维度 | Inline (默认) | Fork (`context: fork`) |
|------|--------------|----------------------|
| 执行位置 | 主 conversation 中 | 独立的 forked sub-agent |
| 历史共享 | 完全共享当前对话历史 | 共享 prompt cache 前缀，不共享对话 |
| System Prompt | 使用主 agent 的 system prompt | 基于主 agent 构建独立的 system prompt |
| Placeholder | 无 | 有 (agent 进度消息) |
| Cache 利用 | 完全利用主 agent cache | 利用 cache 前缀（tools + system prompt） |
| 同步/异步 | 同步 (注入后模型继续) | 同步等待完成（isAsync: false） |
| 返回值 | {newMessages, contextModifier} | {data: {result: string}} |
| Token 预算 | 共享主 conversation 预算 | 独立预算（maxTurns 限制） |
| 用户交互 | 可以与用户交互 | 无法与用户交互 |

### SkillTool Fork vs AgentTool Fork

```
┌──────────────────────────────────────────────────────────┐
│           SkillTool Fork vs AgentTool Fork               │
├─────────────┬──────────────────┬─────────────────────────┤
│ 维度         │ SkillTool Fork   │ AgentTool Fork          │
├─────────────┼──────────────────┼─────────────────────────┤
│ 历史传递     │ 共享 cache prefix │ 可选 fullHistory        │
│ System prompt│ 基于主 agent 构建 │ 基于 agent definition   │
│ Placeholder  │ skill_progress   │ agent_progress          │
│ Cache 共享   │ 是 (prefix)      │ 是 (cacheSafeParams)    │
│ Sync/Async   │ 总是同步          │ 可选同步/异步(background)│
│ 结果传递     │ extractResultText│ extractResultText       │
│ Max turns    │ 无特殊限制       │ maxTurns 可配置          │
│ 工具权限     │ 继承主 agent      │ availableTools 可限制    │
└─────────────┴──────────────────┴─────────────────────────┘
```

[设计决策] 为什么默认是 inline 而不是 fork？因为：
1. **Cache 效率**：Inline 完全利用已有 cache，fork 需要重建
2. **上下文感知**：Inline skill 能看到完整对话历史，做出更准确的判断
3. **用户交互**：Inline skill 可以通过 AskUserQuestion 与用户交互（如 skillify）
4. **简单性**：绝大多数 skill 只是一段 prompt，不需要隔离

Fork 模式的适用场景：当 skill 是完全自包含的、不需要用户中途干预、且输出是一个明确结果的场景（如自动化 CI/CD 流水线）。

---

## 三种方式启动 SubAgent

Skill 与 SubAgent 的关系是一个常被混淆的话题。以下是三种方式：

### 1. Inline skill prompt 中包含 AgentTool 指令（最常见）

```markdown
# simplify 的 prompt 片段
## Phase 2: Launch Three Review Agents in Parallel
Use the AgentTool tool to launch all three agents concurrently...
```

这不是 Skill 系统的功能——而是 prompt 中的自然语言指令。模型读到这段话后，自主决定调用 AgentTool。Skill 只是"告诉模型应该这么做"。

### 2. Fork skill（通过 `context: fork` 配置）

```yaml
# SKILL.md frontmatter
context: fork
```

Skill 自身作为 forked sub-agent 运行。整个 skill prompt 在隔离环境中执行。

### 3. Inline skill 中模型自主决定启动 agent

模型在处理 inline skill 的 prompt 时，可能自主判断"这个任务太复杂，我应该用 AgentTool 分解"。这完全是模型的自主行为，与 Skill 系统无关。

---

## 全部 Bundled Skills 一览

| Skill | 模式 | AgentTool 使用 | 核心用途 |
|-------|------|---------------|---------|
| **simplify** | inline | Prompt 中指示启动 3 个并行 agent | 代码审查：reuse/quality/efficiency |
| **skillify** | inline | 无 (交互式，用 AskUserQuestion) | 将当前会话的流程提取为可复用 skill |
| **remember** | inline | 无 | 审查 auto-memory 并建议 promotion |
| **verify** | inline | 无 (有附带 files) | 验证代码改动的正确性 |
| **loop** | inline | 无 (调用 ScheduleCronTool) | 设置周期性任务 |
| **batch** | inline | Prompt 中指示启动 5-30 个 worktree agent | 大规模并行重构 |

[核心概念] **没有一个 bundled skill 使用 `context: fork`**。这是一个重要的观察——Anthropic 自己的团队在实践中发现 inline 模式几乎总是更好的选择。Fork 模式存在是为了第三方 skill 的完整性，但 bundled skill 全部选择了 inline。

### Skill 的 ant-only 门控

注意 `skillify`, `remember`, `verify`, `batch` 都有 `process.env.USER_TYPE !== 'ant'` 的检查：

```typescript
export function registerSkillifySkill(): void {
  if (process.env.USER_TYPE !== 'ant') {
    return  // 外部用户看不到这些 skill
  }
  // ...
}
```

而 `simplify` 和 `loop` 对所有用户可用。这说明 Anthropic 正在逐步验证和发布 skill——先在内部使用，验证后再开放。

### disableModelInvocation 的含义

```typescript
// skillify, batch 的注册
disableModelInvocation: true,
```

这意味着模型不能主动调用这些 skill——只有用户输入 `/skillify` 或 `/batch` 时才会触发。原因是这些 skill 的触发条件太宽泛（任何"capture this"或"make changes across files"都可能误触），而且执行代价高昂。

---

## Multi-Skill Stacking：当多个 Skill 同时生效

### 堆叠全景图

```
┌───────────────────────────────────────────────────────┐
│              Multi-Skill Stacking 全景                 │
│                                                       │
│  Session starts                                       │
│  ├─ allowedTools: {} (空 Set)                         │
│  ├─ model: "sonnet" (默认)                            │
│  ├─ skill prompts: [] (空)                            │
│  └─ hooks: {} (无)                                    │
│                                                       │
│  User: /review-pr 123                                 │
│  ├─ allowedTools: {Bash(gh:*), Read} ← 只增不减       │
│  ├─ model: "sonnet" → "opus" ← 覆盖，不可恢复         │
│  ├─ skill prompts: [review-pr prompt] ← 堆积          │
│  └─ hooks: {postSampling: [...]} ← 叠加               │
│                                                       │
│  模型自主调用: /simplify                                │
│  ├─ allowedTools: {Bash(gh:*), Read} ← 保持           │
│  │  (simplify 没有额外 allowedTools)                   │
│  ├─ model: "opus" ← 保持 (simplify 没有 model)        │
│  ├─ skill prompts: [review-pr, simplify] ← 继续堆积   │
│  └─ hooks: {postSampling: [...]} ← 保持               │
│                                                       │
│  Token 持续增长 ↑↑↑                                    │
│                                                       │
│  Compaction 触发时:                                    │
│  ├─ 按 invokedAt 排序 (LRU)                           │
│  ├─ 每个 skill 截断到 5000 tokens                      │
│  └─ 总预算 25000 tokens                               │
│                                                       │
└───────────────────────────────────────────────────────┘
```

### allowedTools：Set 累加只增不减

```typescript
// SkillTool.ts:779-800
// contextModifier 内部
if (allowedTools.length > 0) {
  const previousGetAppState = modifiedContext.getAppState
  modifiedContext = {
    ...modifiedContext,
    getAppState() {
      const appState = previousGetAppState()
      return {
        ...appState,
        toolPermissionContext: {
          ...appState.toolPermissionContext,
          alwaysAllowRules: {
            ...appState.toolPermissionContext.alwaysAllowRules,
            command: [
              ...new Set([  // ← Set 去重 + 合并
                ...(appState.toolPermissionContext.alwaysAllowRules.command || []),
                ...allowedTools,
              ]),
            ],
          },
        },
      }
    },
  }
}
```

[权衡] `allowedTools` 只能增加，永远不能减少。这意味着如果 Skill A 授予了 `Bash(*)` 权限，即使 Skill A 的执行已经结束，后续所有操作都将拥有 `Bash(*)` 权限。这是一个安全考量上的 tradeoff：要么维护一个复杂的权限栈（谁授予、何时撤回），要么接受"只增不减"的简单性。当前选择了后者。

### Model 覆盖：后来者替换不可恢复

```typescript
// SkillTool.ts:810-819
if (model) {
  modifiedContext = {
    ...modifiedContext,
    options: {
      ...modifiedContext.options,
      mainLoopModel: resolveSkillModelOverride(model, ctx.options.mainLoopModel),
    },
  }
}
```

如果 Skill A 设置了 `model: opus`，然后 Skill B 设置了 `model: sonnet`，最终使用 sonnet。而且 **没有机制恢复到 Skill A 的设置**——这是 contextModifier 闭包链的特性，每个 modifier 修改的是"当前 context"而非"原始 context"。

### Skill Prompts：全部堆积

所有 inline skill 的 prompt 以 user message 的形式注入到 conversation 中。它们不会互相覆盖——全部保留在消息历史中。这意味着 token 持续增长。

---

## Compaction 时的 LRU Eviction

当 conversation 触发自动 compaction 时，需要决定哪些 skill prompt 保留、哪些丢弃。

```typescript
// compact.ts:129-130
export const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000
export const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000
```

```typescript
// compact.ts:1507-1521
const skills = Array.from(invokedSkills.values())
  .sort((a, b) => b.invokedAt - a.invokedAt)  // ← 按最近使用时间排序 (LRU)
  .map(skill => ({
    name: skill.skillName,
    content: truncateToTokens(
      skill.content,
      POST_COMPACT_MAX_TOKENS_PER_SKILL,     // ← 每个 skill 最多 5000 tokens
    ),
  }))
  .filter(skill => {
    const tokens = roughTokenCountEstimation(skill.content)
    if (usedTokens + tokens > POST_COMPACT_SKILLS_TOKEN_BUDGET) {
      return false                              // ← 总共最多 25000 tokens
    }
    // ...
  })
```

[设计决策] 为什么使用 LRU 而不是 FIFO？因为用户可能在长会话中反复回到某个 skill。LRU 确保"最近还在用"的 skill 被保留，而"很早用过但后来没用"的 skill 被淘汰。25000 tokens 大约能保留 5 个完整 skill。

### invoked_skills 在 Compaction 中的角色

```typescript
// postCompactCleanup.ts:17-18
// Note: We intentionally do NOT clear invoked skill content here.
// Skill content must survive across multiple compactions so that
// createSkillAttachmentIfNeeded() can include the full skill text
// in subsequent compaction attachments.
```

[核心概念] invoked_skills 的生命周期跨越多次 compaction。这是因为 compaction 产生的 summary 可能不完整地保留了 skill 的上下文——如果之后模型需要回到这个 skill 的指令，必须能够重新注入完整（或截断后的）prompt。

---

## 常见误区

### 误区 1："Skill 是代码插件"

**正确**：Skill 是模块化的 prompt template。它的全部内容就是给模型的自然语言指令。不要试图在 SKILL.md 中写"if-else"逻辑——应该用自然语言描述条件分支（"if the user provides X, do A; otherwise do B"）。

### 误区 2："Fork 模式更安全"

**正确**：Fork 模式的目的不是安全隔离（forked agent 仍然可以使用被允许的工具），而是 **上下文隔离**——避免 skill 的长输出污染主 conversation。安全性由 `allowedTools` 和 `checkPermissions` 控制，与执行模式无关。

### 误区 3："Skill 调用是免费的"

**正确**：每次 inline skill 调用至少增加一个 user message（skill prompt），这些 tokens 会持续存在直到 compaction。频繁调用不同的 skill 会快速消耗 context window。

### 误区 4："allowedTools 是作用域内的"

**正确**：Skill 授予的 `allowedTools` 是 session 级别的、只增不减的。即使 skill 的"执行"已经"结束"（模型已经处理完 skill prompt 并继续做其他事），skill 授予的权限仍然有效。

### 误区 5："所有 bundled skill 都对外部用户可用"

**正确**：只有 `simplify` 和 `loop` 对所有用户可用。其余（skillify, remember, verify, batch）目前仅对 Anthropic 内部用户开放（`USER_TYPE === 'ant'`）。

---

## 跨模块联系

### Skill × Agent 系统

Skill 和 Agent 是正交但互补的：
- Skill 提供 **what to do**（指令内容）
- Agent 提供 **how to execute**（执行环境）

`simplify` skill 就是最好的例子——它的 prompt 中明确指示模型使用 `AgentTool` 启动三个并行 sub-agent。Skill 是导演，Agent 是演员。

### Skill × Compaction

Compaction 系统通过 `invoked_skills` state 来保护 skill prompt 的生存。`postCompactCleanup.ts` 明确不清理 invoked_skills——这打破了"compaction 清理一切"的直觉。

### Skill × Permission 系统

SkillTool 是权限系统的一等公民——它有自己的 deny/allow rule 检查，还有独特的 `skillHasOnlySafeProperties` 自动放行机制。这使得 skill 权限比普通 tool 权限更精细化。

### Skill × Cache 系统

Skill 的 cache 利用是一个隐藏的优化。由于 skill_listing 在 system prompt 中（且内容基本不变），它会被 prompt cache 命中。但每次 skill 调用产生的 newMessages 是新内容，会触发 cache_creation。这就是为什么 prompt.ts 要控制 skill_listing 的大小——它直接影响 cache 效率。

### Skill × Memory 系统

`remember` skill 是 Skill 系统和 Memory 系统的桥梁——它使用 skill 的模块化 prompt 能力来审查和组织 auto-memory 条目。这展示了 Skill 作为"跨系统编排层"的价值。

---

## 核心 Takeaway

1. **Skill = 模块化 Prompt**，不是代码插件。理解这一点是正确使用 Skill 系统的前提。

2. **Discovery 靠 budget 控制**（1% context window），delivery 靠 message injection，lifecycle 靠 LRU eviction。三个阶段，三种策略。

3. **Inline 优于 Fork**（所有 bundled skill 都选择 inline），除非你明确需要上下文隔离。

4. **一次 Skill 调用 = 两次 API 请求**。用户直接 `/skill-name` 可以省掉一次。

5. **Multi-skill stacking 是 append-only 的**：allowedTools 只增不减，model 后来者覆盖，prompt 全部堆积。设计 skill 时要考虑与其他 skill 的交互。

6. **BLOCKING REQUIREMENT 模式** 是确保 AI 优先使用 skill 而非自行尝试的关键 prompt engineering 技巧。

7. **Safe properties allowlist** 是 secure-by-default 的典范——新属性默认需要权限，显式加入白名单才自动放行。

---

## 思考题

1. **Multi-skill 权限累加问题**：如果 Skill A 授予 `Bash(rm:*)` 权限用于清理临时文件，Skill B 只需要 `Read` 权限做代码审查。B 执行时是否能使用 `Bash(rm:*)`？这合理吗？如何改进？

2. **Skill 发现的 false positive**：当 skill_listing 中有 50 个 skill，模型如何避免把"help me review this PR"错误匹配到多个看似相关的 skill？MAX_LISTING_DESC_CHARS=250 的限制是否会加剧这个问题？

3. **Fork vs Inline 的选择**：假设你要实现一个"自动部署"skill，需要执行 10 分钟的 CI/CD 流程。选择 fork 还是 inline？为什么？

4. **Compaction 后的 Skill 恢复**：POST_COMPACT_MAX_TOKENS_PER_SKILL=5000 意味着长 skill 会被截断。如果 skill 的关键指令在第 5001 个 token 之后，会发生什么？如何设计 skill 来避免这个问题？

5. **Skill 作为 SOP 的极限**：Skill 是纯 prompt，没有代码逻辑。那什么样的自动化任务是 Skill 系统无法表达的？Skill + hooks 的组合能否弥补？

---

## 关键源文件

| 文件 | 职责 |
|------|------|
| `src/tools/SkillTool/SkillTool.ts` | SkillTool 核心：validate → permission → call 全链路 |
| `src/tools/SkillTool/prompt.ts` | Skill discovery：budget 计算、listing 格式化、SkillTool description |
| `src/skills/bundledSkills.ts` | BundledSkillDefinition 类型、注册机制、files 解压 |
| `src/skills/bundled/simplify.ts` | Bundled skill 示例：inline + AgentTool 指令 |
| `src/skills/bundled/skillify.ts` | Bundled skill 示例：inline + 交互式 |
| `src/skills/bundled/batch.ts` | Bundled skill 示例：大规模并行 worktree agent |
| `src/skills/bundled/remember.ts` | Bundled skill 示例：memory 审查 |
| `src/skills/loadSkillsDir.ts` | 磁盘 skill 加载：模板变量替换、frontmatter 解析 |
| `src/services/compact/compact.ts` | Compaction 中的 skill eviction：LRU + token budget |
| `src/bootstrap/state.ts` | invokedSkills 状态管理：addInvokedSkill、invokedAt 时间戳 |
| `src/utils/processUserInput/processSlashCommand.tsx` | 用户 /command 入口：跳过 validateInput/checkPermissions |
