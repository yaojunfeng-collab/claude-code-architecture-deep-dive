# Part 2: Prompt 组装 — 一个 API 请求是怎么被精心拼装出来的？

> **系列**: Claude Code 2.1.88 架构深度剖析 (10/10)
> **作者**: Senior Engineer Tech Talk
> **前置阅读**: Part 1 — 对话引擎

---

## 开场

> 你有没有想过，当你在 Claude Code 里输入一句话，到它调用 Anthropic API 之间，到底发生了什么？

一个看似简单的 API 调用，背后其实藏着一个精密的五步组装流水线。system prompt 被切分成 15+ 个 section，其中有些全球共享、有些每用户独立、有些每轮重算。CLAUDE.md 不放在 system prompt 里而是塞进 messages[0]。git status 被追加到 system prompt 的尾巴上但放在一个叫 "dynamic boundary" 的标记之后。一个叫 `DANGEROUS_uncachedSystemPromptSection` 的函数名在时刻提醒开发者：你加的每一行动态内容，都在摧毁全局缓存。

这一切，都是为了一个目标：**在保证行为正确性的前提下，最大化 prompt cache 命中率**。

本篇将从源码级别，拆解 Claude Code 的 Prompt 组装系统。

---

## 全景：五步组装流水线

先看全局流程图：

```
用户输入 "help me fix this bug"
          │
          ▼
┌─────────────────────────────────────────────────┐
│  Step 1: fetchSystemPromptParts()               │
│  并行获取三大原材料 (Promise.all)                   │
│  ┌──────────────┬──────────────┬──────────────┐  │
│  │defaultSystem │ userContext  │systemContext  │  │
│  │Prompt        │ {claudeMd,  │ {gitStatus}   │  │
│  │ (string[])   │  currentDate}│              │  │
│  └──────────────┴──────────────┴──────────────┘  │
└─────────────────────────┬───────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────┐
│  Step 2: QueryEngine 中组装 systemPrompt         │
│  customPrompt 可以完全替换 defaultSystemPrompt     │
│  + memoryMechanicsPrompt + appendSystemPrompt    │
│  → asSystemPrompt([...])                         │
└─────────────────────────┬───────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────┐
│  Step 3: query loop 中消息预处理和上下文注入         │
│  ┌──────────────────────────────────────────┐    │
│  │ toolResultBudget → snip → microcompact   │    │
│  │ → contextCollapse → autocompact          │    │
│  └──────────────────────────────────────────┘    │
│  appendSystemContext(systemPrompt, systemContext) │
│  prependUserContext(messages, userContext)        │
└─────────────────────────┬───────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────┐
│  Step 4: queryModel 中最终组装 (claude.ts:1017)  │
│  - Tool schema 构建 (含 defer_loading)           │
│  - System prompt header (attribution + prefix)  │
│  - Message 规范化 + tool_result 配对修复           │
│  - 最终 API params 构建                          │
└─────────────────────────┬───────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────┐
│  Step 5: 三层缓存标记                             │
│  splitSysPromptPrefix() → SystemPromptBlock[]   │
│  ┌────────────────────────────────────────┐      │
│  │ Attribution header   │ cacheScope=null │      │
│  │ CLI prefix           │ cacheScope=null │      │
│  │ Static sections [0-6]│ cacheScope='global' │  │
│  │ Dynamic sections     │ cacheScope=null │      │
│  └────────────────────────────────────────┘      │
│  buildSystemPromptBlocks() → TextBlockParam[]   │
└─────────────────────────────────────────────────┘
          │
          ▼
    Anthropic Messages API
```

**[核心概念]** 整个流水线的设计核心是一个理念：**数据获取并行化，组装延迟化，缓存最大化**。三大原材料在 Step 1 就并行获取完毕，但它们各自被注入到不同位置（system prompt vs messages[0]），在不同时机（Step 2 vs Step 3）完成组装，以实现不同的缓存策略。

---

## 深入一：三大原材料

### 1.1 fetchSystemPromptParts() — 并行获取

入口在 `src/utils/queryContext.ts:44`：

```typescript
// src/utils/queryContext.ts
export async function fetchSystemPromptParts({
  tools, mainLoopModel, additionalWorkingDirectories,
  mcpClients, customSystemPrompt,
}): Promise<{
  defaultSystemPrompt: string[]
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
}> {
  const [defaultSystemPrompt, userContext, systemContext] = await Promise.all([
    customSystemPrompt !== undefined
      ? Promise.resolve([])                    // 自定义 prompt 完全替换
      : getSystemPrompt(tools, mainLoopModel, additionalWorkingDirectories, mcpClients),
    getUserContext(),                            // CLAUDE.md + currentDate
    customSystemPrompt !== undefined
      ? Promise.resolve({})                    // 自定义模式不需要 git context
      : getSystemContext(),                     // gitStatus
  ])
  return { defaultSystemPrompt, userContext, systemContext }
}
```

**[可借鉴]** `Promise.all` 并行获取三大原材料。`getSystemPrompt()` 涉及文件系统扫描（skill commands, output style config, env info），`getUserContext()` 要遍历 8 层 CLAUDE.md，`getSystemContext()` 要跑 5 个 git 命令。三者互不依赖，并行执行能显著减少启动延迟。

**[设计决策]** 当 `customSystemPrompt` 被设置时（SDK 调用者传入自定义 prompt），`getSystemPrompt()` 和 `getSystemContext()` 都被跳过。这是因为自定义 prompt 完全替换了默认 prompt，而 `systemContext`（git status）是追加到默认 prompt 尾部的，如果默认 prompt 不存在，追加就没有意义。

---

### 1.2 原材料 A：defaultSystemPrompt (string[])

来源：`getSystemPrompt()` at `src/constants/prompts.ts:444`

这是一个 **字符串数组**，不是单个字符串。每个元素是一个 section，最终会被 `\n\n` 连接。为什么用数组？因为中间需要插入一个 **boundary marker** 来分割静态和动态内容。

```typescript
// src/constants/prompts.ts:560-577
return [
  // --- Static content (cacheable) ---
  getSimpleIntroSection(outputStyleConfig),        // [0] Intro
  getSimpleSystemSection(),                         // [1] System
  outputStyleConfig === null ||
    outputStyleConfig.keepCodingInstructions === true
    ? getSimpleDoingTasksSection()                  // [2] Doing Tasks
    : null,
  getActionsSection(),                              // [3] Actions
  getUsingYourToolsSection(enabledTools),           // [4] Using Tools
  getSimpleToneAndStyleSection(),                   // [5] Tone/Style
  getOutputEfficiencySection(),                     // [6] Output Efficiency

  // === BOUNDARY MARKER - DO NOT MOVE OR REMOVE ===
  ...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),

  // --- Dynamic content (registry-managed) ---
  ...resolvedDynamicSections,
].filter(s => s !== null)
```

**[核心概念]** `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 是整个 prompt 组装系统中最重要的优化标记。它的值就是一个简单的字符串常量 `'__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'`，但它划分了两个世界：

- **Boundary 之前** = 静态内容 = 所有 Claude Code 用户共享 = `cacheScope: 'global'`
- **Boundary 之后** = 动态内容 = 每用户/每会话不同 = `cacheScope: null`

这意味着，全球所有使用 Claude Code 的用户，只要用同一个版本，boundary 之前的 prompt 内容完全相同。Anthropic API 端可以全局缓存这部分 KV cache，新用户只需要计算 boundary 之后的增量。

```
// src/constants/prompts.ts:114-115
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

源码中有一段严厉的 WARNING 注释：

```
WARNING: Do not remove or reorder this marker without updating cache logic in:
- src/utils/api.ts (splitSysPromptPrefix)
- src/services/api/claude.ts (buildSystemPromptBlocks)
```

---

### 1.3 原材料 B：userContext ({claudeMd, currentDate})

来源：`getUserContext()` at `src/context.ts:155`

```typescript
// src/context.ts:155-189
export const getUserContext = memoize(async (): Promise<{ [k: string]: string }> => {
  const shouldDisableClaudeMd =
    isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
    (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)

  const claudeMd = shouldDisableClaudeMd
    ? null
    : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))

  setCachedClaudeMdContent(claudeMd || null)

  return {
    ...(claudeMd && { claudeMd }),
    currentDate: `Today's date is ${getLocalISODate()}.`,
  }
})
```

**注意这个 `memoize`**：`getUserContext` 在整个会话期间只计算一次。这是合理的，因为 CLAUDE.md 文件在会话期间不太可能变化。但也意味着会话中期修改 CLAUDE.md 不会立即生效（需要 `/clear` 或 `/compact` 来清除缓存）。

#### CLAUDE.md 的 8 层加载层级

`getMemoryFiles()` at `src/utils/claudemd.ts:790` 实现了一个多层级的配置文件加载系统：

```
加载优先级（从上到下，后加载的可以覆盖先加载的）：

Level 1: Managed    → 内部策略文件（Anthropic 维护）
Level 2: User       → ~/.claude/CLAUDE.md（用户全局配置）
         User Rules → ~/.claude/rules/*.md
Level 3: Project    → {git-root}/CLAUDE.md（项目级配置）
         Proj Rules → {git-root}/.claude/rules/*.md
Level 4: Local      → {git-root}/CLAUDE.local.md（本地私有配置，gitignore）

加载方向：从根目录向 CWD 递归，越接近 CWD 的优先级越高。
```

```typescript
// src/utils/claudemd.ts:878 简化版
for (const dir of dirs.reverse()) { // 从根目录向 CWD
  // Project: CLAUDE.md（受版本控制）
  result.push(...processMemoryFile(join(dir, 'CLAUDE.md'), 'Project', ...))
  // Project rules: .claude/rules/*.md
  result.push(...processMdRules({ rulesDir: join(dir, '.claude/rules'), type: 'Project', ... }))
  // Local: CLAUDE.local.md（不受版本控制）
  result.push(...processMemoryFile(join(dir, 'CLAUDE.local.md'), 'Local', ...))
}
```

**[设计决策]** CLAUDE.md 为什么放在 `messages[0]` 而不是 `system prompt`？

这是本篇最重要的设计决策之一。来看 `prependUserContext()` 的实现：

```typescript
// src/utils/api.ts:449-474
export function prependUserContext(
  messages: Message[],
  context: { [k: string]: string },
): Message[] {
  if (Object.entries(context).length === 0) {
    return messages
  }

  return [
    createUserMessage({
      content: `<system-reminder>
As you answer the user's questions, you can use the following context:
${Object.entries(context)
  .map(([key, value]) => `# ${key}\n${value}`)
  .join('\n')}

      IMPORTANT: this context may or may not be relevant to your tasks.
      You should not respond to this context unless it is highly relevant to your task.
</system-reminder>`,
      isMeta: true,
    }),
    ...messages,
  ]
}
```

**[权衡]** 如果 CLAUDE.md 放在 system prompt 中：
- 每个用户的 CLAUDE.md 内容不同
- 这会让 boundary 之前的所有静态内容都无法全局缓存
- 因为 Anthropic API 的 prompt cache 要求前缀完全匹配
- 一个用户的 CLAUDE.md 改动会导致所有用户的缓存失效

把 CLAUDE.md 放在 `messages[0]` 中：
- system prompt 的静态部分保持全球一致 → 全局缓存有效
- `messages[0]` 作为 user message 不影响 system prompt 的缓存
- 代价：CLAUDE.md 的内容被包裹在 `<system-reminder>` 标签中，模型对它的注意力可能略低于直接在 system prompt 中

这是一个 **缓存效率 vs 指令权重** 的经典权衡。Claude Code 选择了缓存效率。

---

### 1.4 原材料 C：systemContext ({gitStatus})

来源：`getSystemContext()` at `src/context.ts:116`

```typescript
// src/context.ts:116-150
export const getSystemContext = memoize(async (): Promise<{ [k: string]: string }> => {
  const gitStatus =
    isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) ||
    !shouldIncludeGitInstructions()
      ? null
      : await getGitStatus()

  return {
    ...(gitStatus && { gitStatus }),
    // cache breaker injection (ant-only)
    ...(feature('BREAK_CACHE_COMMAND') && injection
      ? { cacheBreaker: `[CACHE_BREAKER: ${injection}]` }
      : {}),
  }
})
```

`getGitStatus()` 并行执行 5 个 git 命令：

```typescript
// src/context.ts:61-77
const [branch, mainBranch, status, log, userName] = await Promise.all([
  getBranch(),                          // git rev-parse --abbrev-ref HEAD
  getDefaultBranch(),                   // 检测 main/master
  execFileNoThrow(gitExe(),             // git status --short
    ['--no-optional-locks', 'status', '--short'], ...),
  execFileNoThrow(gitExe(),             // git log --oneline -n 5
    ['--no-optional-locks', 'log', '--oneline', '-n', '5'], ...),
  execFileNoThrow(gitExe(),             // git config user.name
    ['config', 'user.name'], ...),
])
```

输出格式：

```
This is the git status at the start of the conversation.
Note that this status is a snapshot in time, and will not update during the conversation.

Current branch: feature/my-branch

Main branch (you will usually use this for PRs): main

Git user: developer

Status:
M  src/app.ts
?? src/new-file.ts

Recent commits:
a1b2c3d Fix login bug
d4e5f6g Add user settings page
...
```

**[设计决策]** gitStatus 被 `appendSystemContext()` 追加到 system prompt 的尾部。尾部恰好在 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 之后（因为 `appendSystemContext` 在 query loop 中调用，此时 boundary 已经在数组中了）。这意味着 git status 是动态内容，不会影响静态部分的全局缓存。

```typescript
// src/utils/api.ts:437-447
export function appendSystemContext(
  systemPrompt: SystemPrompt,
  context: { [k: string]: string },
): string[] {
  return [
    ...systemPrompt,
    Object.entries(context)
      .map(([key, value]) => `${key}: ${value}`)
      .join('\n'),
  ].filter(Boolean)
}
```

**[权衡]** git status 也被 memoize 了（整个会话只计算一次）。prompt 中甚至明确告诉模型："this status is a snapshot in time, and will not update during the conversation"。这是一个 **准确性 vs 性能** 的权衡 —— 每轮都跑 git status 会浪费时间且破坏缓存，而且模型在工作过程中可以随时通过 BashTool 调用 `git status` 获取最新信息。

---

## 深入二：System Prompt 的 15 个 Section

### 静态 Sections (Boundary 之前，全局可缓存)

#### [0] Intro Section

```typescript
function getSimpleIntroSection(outputStyleConfig): string {
  return `
You are an interactive agent that helps users
${outputStyleConfig !== null
  ? 'according to your "Output Style" below...'
  : 'with software engineering tasks.'}
Use the instructions below and the tools available to you to assist the user.

${CYBER_RISK_INSTRUCTION}
IMPORTANT: You must NEVER generate or guess URLs for the user...`
}
```

**关键组成**：
- 角色定义（"interactive agent"）
- `CYBER_RISK_INSTRUCTION`：由 Anthropic 安全团队维护的网络安全指令。这个常量从 `./cyberRiskInstruction.js` 导入，是最高优先级的安全策略。
- URL 生成限制

#### [1] System Section

```typescript
function getSimpleSystemSection(): string {
  const items = [
    `All text you output outside of tool use is displayed to the user...`,
    `Tools are executed in a user-selected permission mode...`,
    `Tool results and user messages may include <system-reminder> tags...`,
    `Tool results may include data from external sources...`,
    getHooksSection(),  // hooks 机制说明
    `The system will automatically compress prior messages...`,
  ]
  return ['# System', ...prependBullets(items)].join('\n')
}
```

涵盖：权限模型、`<system-reminder>` 标签语义、hooks 行为、自动压缩机制。

#### [2] Doing Tasks Section

```typescript
function getSimpleDoingTasksSection(): string {
  const codeStyleSubitems = [
    // 核心编程哲学三原则:
    `Don't add features, refactor code, or make "improvements" beyond what was asked...`,
    `Don't add error handling, fallbacks, or validation for scenarios that can't happen...`,
    `Don't create helpers, utilities, or abstractions for one-time operations...`,
    // Ant-only: 注释原则、验证原则
    ...(process.env.USER_TYPE === 'ant' ? [
      `Default to writing no comments...`,
      `Before reporting a task complete, verify it actually works...`,
    ] : []),
  ]
  // ...
}
```

**[设计决策]** 这一段体现了 Claude Code 的编码哲学三原则：
1. **Don't over-engineer**: 不要超出需求范围
2. **Don't over-defend**: 不要为不可能发生的场景添加防御
3. **Don't premature-abstract**: 不要为一次性操作创建抽象

Ant-only 规则还包括诚实性验证：`Report outcomes faithfully: if tests fail, say so...`。这是专门针对内部使用时发现的虚假声明问题（False-claims）的对策。

#### [3] Actions Section

操作安全原则。核心理念："measure twice, cut once"（三思而后行）。

```
Carefully consider the reversibility and blast radius of actions.
Generally you can freely take local, reversible actions like editing files or running tests.
But for actions that are hard to reverse, affect shared systems beyond your local environment,
or could otherwise be risky or destructive, check with the user before proceeding.
```

**[核心概念]** "authorization is not generalizable" —— 用户批准一次操作（如 git push）并不意味着在所有上下文中都批准。除非在 CLAUDE.md 等持久化指令中预先授权。

#### [4] Using Tools Section

```typescript
function getUsingYourToolsSection(enabledTools): string {
  const providedToolSubitems = [
    `To read files use ${FILE_READ_TOOL_NAME} instead of cat, head, tail, or sed`,
    `To edit files use ${FILE_EDIT_TOOL_NAME} instead of sed or awk`,
    `To create files use ${FILE_WRITE_TOOL_NAME} instead of cat with heredoc...`,
    `To search for files use ${GLOB_TOOL_NAME} instead of find or ls`,
    `To search the content of files, use ${GREP_TOOL_NAME} instead of grep or rg`,
    // ...
  ]
  // ...
  return [`# Using your tools`,
    `Do NOT use the ${BASH_TOOL_NAME} to run commands when a relevant dedicated tool is provided.`,
    // ...
  ].join('\n')
}
```

**[设计决策]** 为什么要如此强调"使用专用工具而不是 Bash"？四个原因：
1. **权限控制**：专用工具有细粒度的权限检查（如 FileEditTool 检查文件路径），Bash 是通用 escape hatch
2. **用户体验**：专用工具的输出格式更好，用户更容易 review
3. **安全性**：Bash 可以执行任意命令，攻击面更大
4. **模型倾向**：模型天然倾向于使用 Bash（因为训练数据中大量的 shell 命令），需要明确引导

#### [5] Tone/Style Section

```typescript
function getSimpleToneAndStyleSection(): string {
  const items = [
    `Only use emojis if the user explicitly requests it.`,
    `When referencing specific functions... include the pattern file_path:line_number`,
    `When referencing GitHub issues... use the owner/repo#123 format`,
    `Do not use a colon before tool calls.`,  // 微妙但重要
  ]
  return [`# Tone and style`, ...prependBullets(items)].join('\n')
}
```

"Do not use a colon before tool calls" 这条规则值得关注：因为 tool call 在 UI 中可能不直接显示，如果模型写 "Let me read the file:" 然后紧跟一个 tool call，用户看到的就是一个不完整的句子加冒号。所以要求用句号："Let me read the file."

#### [6] Output Efficiency Section

这个 section 有两个完全不同的版本：

```typescript
function getOutputEfficiencySection(): string {
  if (process.env.USER_TYPE === 'ant') {
    // 内部版本：更详细的写作指南
    return `# Communicating with the user
When sending user-facing text, you're writing for a person, not logging to a console.
...Write so they can pick back up cold: use complete, grammatically correct sentences
without unexplained jargon...`
  }
  // 外部版本：简洁的效率指南
  return `# Output efficiency
IMPORTANT: Go straight to the point. Try the simplest approach first...
Keep your text output brief and direct. Lead with the answer or action, not the reasoning.`
}
```

**[设计决策]** 内外部版本差异体现了不同的用户群体需求：
- 外部用户更关心简洁（减少 token 消耗 = 减少费用）
- 内部用户更关心信息质量（假设成本不是主要顾虑）

---

### BOUNDARY MARKER

```typescript
// prompts.ts:572-573
...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),
```

**[核心概念]** 这一行代码是整个缓存系统的分水岭。`shouldUseGlobalCacheScope()` 检查是否使用 1P Anthropic API（第三方代理/Bedrock/Vertex 不支持 global cache scope）。只有直连 Anthropic API 时，才插入这个 boundary marker。

---

### 动态 Sections (Boundary 之后，每用户/每会话)

动态 sections 使用一个注册表模式：

```typescript
// prompts.ts:491-555
const dynamicSections = [
  systemPromptSection('session_guidance', () =>
    getSessionSpecificGuidanceSection(enabledTools, skillToolCommands)),  // [7]
  systemPromptSection('memory', () => loadMemoryPrompt()),                // [8]
  systemPromptSection('ant_model_override', () =>
    getAntModelOverrideSection()),
  systemPromptSection('env_info_simple', () =>
    computeSimpleEnvInfo(model, additionalWorkingDirectories)),           // [9]
  systemPromptSection('language', () =>
    getLanguageSection(settings.language)),                               // [10]
  systemPromptSection('output_style', () =>
    getOutputStyleSection(outputStyleConfig)),                            // [11]
  DANGEROUS_uncachedSystemPromptSection(                                  // [12] !!!
    'mcp_instructions',
    () => isMcpInstructionsDeltaEnabled()
      ? null
      : getMcpInstructionsSection(mcpClients),
    'MCP servers connect/disconnect between turns',
  ),
  systemPromptSection('scratchpad', () => getScratchpadInstructions()),   // [13]
  systemPromptSection('frc', () => getFunctionResultClearingSection(model)), // [14]
  systemPromptSection('summarize_tool_results',
    () => SUMMARIZE_TOOL_RESULTS_SECTION),                               // [15]
]

const resolvedDynamicSections = await resolveSystemPromptSections(dynamicSections)
```

#### systemPromptSection vs DANGEROUS_uncachedSystemPromptSection

这两个函数在 `src/constants/systemPromptSections.ts` 中定义：

```typescript
// src/constants/systemPromptSections.ts
export function systemPromptSection(
  name: string,
  compute: ComputeFn,
): SystemPromptSection {
  return { name, compute, cacheBreak: false }
}

export function DANGEROUS_uncachedSystemPromptSection(
  name: string,
  compute: ComputeFn,
  _reason: string,   // 必须解释为什么需要破坏缓存
): SystemPromptSection {
  return { name, compute, cacheBreak: true }
}
```

**[核心概念]** `cacheBreak: false` 的 section 只在首次访问时计算，之后从内存缓存中读取，直到 `/clear` 或 `/compact` 清除缓存。`cacheBreak: true` 的 section **每一轮都重新计算**。

```typescript
// systemPromptSections.ts:43-58
export async function resolveSystemPromptSections(
  sections: SystemPromptSection[],
): Promise<(string | null)[]> {
  const cache = getSystemPromptSectionCache()
  return Promise.all(
    sections.map(async s => {
      if (!s.cacheBreak && cache.has(s.name)) {
        return cache.get(s.name) ?? null      // 命中缓存，直接返回
      }
      const value = await s.compute()          // 未命中或 cacheBreak，重新计算
      setSystemPromptSectionCacheEntry(s.name, value)
      return value
    }),
  )
}
```

**[权衡]** 函数名中的 `DANGEROUS_` 前缀不是装饰。每当 MCP section 的值在两轮之间发生变化时，这个 section 之后的所有 system prompt 内容都无法命中 API 端的 prompt cache。这个前缀是一个工程文化信号：**添加动态 section 的工程师必须提供理由（`_reason` 参数），并且明确知道自己在承担什么代价**。

#### [7] Session Guidance Section

```typescript
function getSessionSpecificGuidanceSection(enabledTools, skillToolCommands): string | null {
  const items = [
    hasAskUserQuestionTool ? `If you do not understand why the user has denied a tool call, use ${ASK_USER_QUESTION_TOOL_NAME}` : null,
    getIsNonInteractiveSession() ? null : `If you need the user to run a shell command themselves, suggest they type \`! <command>\`...`,
    hasAgentTool ? getAgentToolSection() : null,
    hasSkills ? `/<skill-name> is shorthand for users to invoke a user-invocable skill...` : null,
    // ...
  ].filter(item => item !== null)
  if (items.length === 0) return null
  return ['# Session-specific guidance', ...prependBullets(items)].join('\n')
}
```

**[设计决策]** 这个 section 为什么在 boundary 之后？因为它依赖于 `enabledTools`（哪些工具可用）、`isNonInteractiveSession()`（是否交互模式）、`isForkSubagentEnabled()` 等运行时条件。这些条件在不同用户、不同会话类型下都不同。如果放在 boundary 之前，会导致 2^N 个静态 prompt 变体（N = 运行时条件数），每个变体都需要独立缓存，彻底摧毁全局缓存的价值。

#### [8] Memory Section (MEMORY.md)

```typescript
systemPromptSection('memory', () => loadMemoryPrompt()),
```

这是 Claude Code 的自动记忆系统。`loadMemoryPrompt()` 加载 `MEMORY.md` 的内容，告诉模型如何使用记忆工具。

#### [9] Environment Section

```typescript
export async function computeSimpleEnvInfo(modelId, additionalWorkingDirectories): Promise<string> {
  const [isGit, unameSR] = await Promise.all([getIsGit(), getUnameSR()])

  // Undercover mode: 隐藏模型名称
  let modelDescription: string | null = null
  if (process.env.USER_TYPE === 'ant' && isUndercover()) {
    // suppress
  } else {
    const marketingName = getMarketingNameForModel(modelId)
    modelDescription = marketingName
      ? `You are powered by the model named ${marketingName}. The exact model ID is ${modelId}.`
      : `You are powered by the model ${modelId}.`
  }

  const envItems = [
    `Primary working directory: ${cwd}`,
    [`Is a git repository: ${isGit}`],
    `Platform: ${env.platform}`,
    getShellInfoLine(),
    `OS Version: ${unameSR}`,
    modelDescription,
    knowledgeCutoffMessage,
    // 模型家族信息（帮助用户构建 AI 应用时选择正确模型）
    `The most recent Claude model family is Claude 4.5/4.6...`,
  ]
  return [`# Environment`, ...prependBullets(envItems)].join('\n')
}
```

**[设计决策]** Undercover 模式（`isUndercover()`）是一个 ant-only 功能，在这种模式下所有模型名称/ID 都从 system prompt 中移除。目的是防止内部评测时模型名称泄漏到公开的 commits/PRs 中。这是一个安全特性，不是功能特性。

#### [10] Language Section

```typescript
function getLanguageSection(languagePreference): string | null {
  if (!languagePreference) return null
  return `# Language
Always respond in ${languagePreference}. Use ${languagePreference} for all explanations...
Technical terms and code identifiers should remain in their original form.`
}
```

#### [11] Output Style Section

```typescript
function getOutputStyleSection(outputStyleConfig): string | null {
  if (outputStyleConfig === null) return null
  return `# Output Style: ${outputStyleConfig.name}\n${outputStyleConfig.prompt}`
}
```

**注意**：当 Output Style 被设置时，它可以通过 `keepCodingInstructions: false` 完全替换 [2] Doing Tasks section。这在 `prompts.ts:564` 的条件判断中体现：

```typescript
outputStyleConfig === null || outputStyleConfig.keepCodingInstructions === true
  ? getSimpleDoingTasksSection()
  : null,
```

#### [12] MCP Instructions Section (DANGEROUS!)

```typescript
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => isMcpInstructionsDeltaEnabled()
    ? null
    : getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns',
),
```

这是整个 system prompt 中唯一的 DANGEROUS section。原因：MCP 服务器可以在会话中途连接或断开，每个 MCP 服务器可能提供自己的 instructions。这意味着每一轮的 MCP instructions 都可能不同。

**[权衡]** 为了缓解缓存破坏的影响，Claude Code 2.1.88 引入了 "delta mode"（`isMcpInstructionsDeltaEnabled()`）：当 delta mode 启用时，MCP instructions 不再放在 system prompt 中（返回 null），而是以 attachment 的形式注入到 messages 中。这样 system prompt 就完全稳定了。

#### [13] Scratchpad Section

```typescript
export function getScratchpadInstructions(): string | null {
  if (!isScratchpadEnabled()) return null
  const scratchpadDir = getScratchpadDir()
  return `# Scratchpad Directory
IMPORTANT: Always use this scratchpad directory for temporary files instead of \`/tmp\`:
\`${scratchpadDir}\`
...`
}
```

提供一个会话特定的临时目录，隔离于用户项目之外。

#### [14] FRC (Function Result Clearing)

```typescript
function getFunctionResultClearingSection(model): string | null {
  if (!feature('CACHED_MICROCOMPACT') || !getCachedMCConfigForFRC) return null
  const config = getCachedMCConfigForFRC()
  if (!config.enabled || !config.systemPromptSuggestSummaries || !isModelSupported) return null
  return `# Function Result Clearing
Old tool results will be automatically cleared from context to free up space.
The ${config.keepRecent} most recent results are always kept.`
}
```

这个 section 告诉模型：旧的 tool result 会被自动清除。搭配 [15] 使用。

#### [15] Summarize Tool Results

```typescript
const SUMMARIZE_TOOL_RESULTS_SECTION =
  `When working with tool results, write down any important information you might need later
in your response, as the original tool result may be cleared later.`
```

[14] 和 [15] 配合工作：[14] 告知清除机制的存在，[15] 教模型如何应对 —— 及时将重要信息写到响应中。

---

## 深入三：QueryEngine 中的组装

### Step 2: systemPrompt 组装

在 `QueryEngine.ts:321`：

```typescript
// QueryEngine.ts:321-325
const systemPrompt = asSystemPrompt([
  ...(customPrompt !== undefined ? [customPrompt] : defaultSystemPrompt),
  ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
])
```

**关键点**：
1. `customPrompt` 完全替换 `defaultSystemPrompt`（不是追加）
2. `memoryMechanicsPrompt` 仅在自定义 prompt + 显式设置了 `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 时注入
3. `appendSystemPrompt` 永远追加在最后
4. 此时 `userContext` 和 `systemContext` 还**没有**合并进来

```
组装结果 (Step 2):
┌──────────────────────────────────────────┐
│ defaultSystemPrompt[0] (Intro)           │
│ defaultSystemPrompt[1] (System)          │
│ defaultSystemPrompt[2] (Doing Tasks)     │
│ defaultSystemPrompt[3] (Actions)         │
│ defaultSystemPrompt[4] (Using Tools)     │
│ defaultSystemPrompt[5] (Tone/Style)      │
│ defaultSystemPrompt[6] (Output Efficiency)│
│ __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__       │
│ defaultSystemPrompt[7..15] (Dynamic)     │
│ [appendSystemPrompt]  ← 如果有           │
└──────────────────────────────────────────┘
                    +
 userContext 和 systemContext 还在外面等着
```

---

## 深入四：Query Loop 中的预处理

### Step 3: 消息预处理 + 上下文注入

在 `src/query.ts` 的 query loop 中，每一轮迭代都执行以下步骤：

#### 3a. 消息压缩链

```
messagesForQuery (原始消息)
       │
       ▼
toolResultBudget  ─────→ 限制单条 tool result 的大小
       │                   (applyToolResultBudget)
       ▼
snip ─────────────→ 历史消息截断（HISTORY_SNIP feature）
       │                   (snipCompactIfNeeded)
       ▼
microcompact ─────→ 旧的 tool result 内容替换为摘要
       │                   (CACHED_MICROCOMPACT feature)
       ▼
contextCollapse ──→ 相关消息折叠为摘要
       │                   (CONTEXT_COLLAPSE feature)
       ▼
autocompact ──────→ 当 token 数接近限制时，自动压缩整个历史
       │
       ▼
messagesForQuery (压缩后的消息)
```

每个阶段的源码对应关系：

```typescript
// query.ts:379 - toolResultBudget
messagesForQuery = await applyToolResultBudget(
  messagesForQuery, toolUseContext.contentReplacementState, ...)

// query.ts:403 - snip
const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
messagesForQuery = snipResult.messages

// query.ts:414 - microcompact
const microcompactResult = await deps.microcompact(
  messagesForQuery, toolUseContext, querySource)
messagesForQuery = microcompactResult.messages

// query.ts:441 - contextCollapse
const collapseResult = await contextCollapse.applyCollapsesIfNeeded(
  messagesForQuery, toolUseContext, querySource)
messagesForQuery = collapseResult.messages

// query.ts:454 - autocompact
const { compactionResult } = await deps.autocompact(
  messagesForQuery, toolUseContext, {...}, querySource, tracking, snipTokensFreed)
```

#### 3b. appendSystemContext — 注入 gitStatus

```typescript
// query.ts:449-451
const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext),
)
```

这一步将 `systemContext`（主要是 `gitStatus`）追加到 system prompt 的末尾。追加后的结构：

```
┌──────────────────────────────────────────┐
│ [原 systemPrompt 内容, 含 boundary]       │
│ ...                                      │
│ "gitStatus: This is the git status..."   │  ← 追加在最后
└──────────────────────────────────────────┘
```

**[设计决策]** gitStatus 被追加在 system prompt 的最尾部（即 boundary 之后的动态区域的最尾部）。这意味着：
1. 不影响静态区域的全局缓存
2. 不影响动态区域中其他 section 的 org 级别缓存
3. 每个用户的 git 仓库状态不同，放在最后对缓存的影响最小

#### 3c. prependUserContext — 注入 CLAUDE.md

```typescript
// query.ts:660
messages: prependUserContext(messagesForQuery, userContext),
```

在发送给 API 的最后一刻，CLAUDE.md 和 currentDate 被作为 `messages[0]` 注入。此时消息数组变成：

```
[
  { type: 'user', content: '<system-reminder>...CLAUDE.md内容...</system-reminder>', isMeta: true },
  { type: 'user', content: '用户的第一条消息' },
  { type: 'assistant', content: '...' },
  ...后续对话
]
```

**[核心概念]** `isMeta: true` 标记这条消息不是真正的用户输入。UI 不会显示它，但 API 会看到它。

---

## 深入五：queryModel 最终组装

### Step 4: 最终 API 调用 (claude.ts:1017)

`queryModel()` 是整个流水线的最后一站。它负责：

#### 4a. Tool Schema 构建

```typescript
// claude.ts:1064 (简化)
queryCheckpoint('query_tool_schema_build_start')

// 检查是否启用 tool search（大量工具时使用 defer_loading）
let useToolSearch = await isToolSearchEnabled(...)

// 预计算哪些工具需要 defer
const deferredToolNames = new Set<string>()
if (useToolSearch) {
  for (const t of tools) {
    if (isDeferredTool(t)) deferredToolNames.add(t.name)
  }
}
```

**[可借鉴]** Tool search + defer_loading 是一个有趣的优化：当工具数量很多时（尤其是有大量 MCP 工具），不是一次性把所有工具 schema 发送给 API，而是标记部分工具为 `defer_loading: true`。API 端只在模型实际调用这些工具时才加载完整 schema。

每个工具的 schema 构建还有独立的缓存机制（`toolSchemaCache.ts`）：

```typescript
// api.ts:148-150
const cache = getToolSchemaCache()
let base = cache.get(cacheKey)
if (!base) {
  // 首次构建 schema（包含 prompt, input_schema, strict, eager_input_streaming）
  // 然后缓存
  cache.set(cacheKey, base)
}
```

**[设计决策]** 为什么工具 schema 需要 session-stable 缓存？因为 GrowthBook 的 feature flag 可能在会话中途翻转（如 `tengu_tool_pear`, `tengu_fgts`），如果每次都重新计算，工具 schema 的字节序列会变化，导致 API 端的缓存失效。Session-stable 缓存保证了一旦计算完成，工具 schema 在整个会话中保持不变。

#### 4b. System Prompt Header 注入

```typescript
// claude.ts:1358-1369
systemPrompt = asSystemPrompt([
  getAttributionHeader(fingerprint),                    // 计费归因 header
  getCLISyspromptPrefix({                               // CLI 标识前缀
    isNonInteractive: options.isNonInteractiveSession,
    hasAppendSystemPrompt: options.hasAppendSystemPrompt,
  }),
  ...systemPrompt,                                      // 前面组装好的 prompt
  ...(advisorModel ? [ADVISOR_TOOL_INSTRUCTIONS] : []), // advisor 指令
  ...(injectChromeHere ? [CHROME_TOOL_SEARCH_INSTRUCTIONS] : []),
].filter(Boolean))
```

**最终的 system prompt 结构**：

```
┌─────────────────────────────────────────────────────┐
│ "x-anthropic-billing-header: ..."                   │  Attribution header
│ "Claude Code <version> interactive|headless [+append]" │  CLI prefix
├─────────────────────────────────────────────────────┤
│ [0] Intro + CYBER_RISK_INSTRUCTION                  │
│ [1] System                                          │  Static Zone
│ [2] Doing Tasks                                     │  (cacheScope: 'global')
│ [3] Actions                                         │
│ [4] Using Tools                                     │
│ [5] Tone/Style                                      │
│ [6] Output Efficiency                               │
├─────────────────────────────────────────────────────┤
│ __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__                  │  Boundary
├─────────────────────────────────────────────────────┤
│ [7] Session Guidance                                │
│ [8] Memory (MEMORY.md)                              │  Dynamic Zone
│ [9] Environment                                     │  (cacheScope: null)
│ [10] Language                                       │
│ [11] Output Style                                   │
│ [12] MCP Instructions (DANGEROUS!)                  │
│ [13] Scratchpad                                     │
│ [14] FRC                                            │
│ [15] Summarize Tool Results                         │
│ [advisor instructions]                              │
│ "gitStatus: ..."                                    │
└─────────────────────────────────────────────────────┘
```

#### 4c. Message 规范化

```typescript
// 在 queryModel 内部，消息经过 normalizeMessagesForAPI() 处理
// 确保 user/assistant 交替出现
// 修复 tool_use / tool_result 的配对关系
```

#### 4d. 构建最终 API 请求参数

```typescript
// claude.ts:1376
const system = buildSystemPromptBlocks(systemPrompt, enablePromptCaching, {
  skipGlobalCacheForSystemPrompt: needsToolBasedCacheMarker,
  querySource: options.querySource,
})
```

---

## 深入六：三层缓存标记

### Step 5: splitSysPromptPrefix()

`splitSysPromptPrefix()` at `src/utils/api.ts:321` 是缓存标记的核心逻辑。它根据不同场景返回不同的缓存策略：

#### 场景 1: Global Cache Mode（1P 直连 + 找到 boundary）

```
┌──────────────────────┬─────────────────┐
│ Attribution header   │ cacheScope=null │
│ CLI prefix           │ cacheScope=null │
│ Static sections      │ cacheScope='global' │  ← 全球共享缓存！
│ Dynamic sections     │ cacheScope=null │
└──────────────────────┴─────────────────┘
```

```typescript
// api.ts:362-404 (简化)
if (useGlobalCacheFeature) {
  const boundaryIndex = systemPrompt.findIndex(
    s => s === SYSTEM_PROMPT_DYNAMIC_BOUNDARY)
  if (boundaryIndex !== -1) {
    for (let i = 0; i < systemPrompt.length; i++) {
      const block = systemPrompt[i]
      if (block.startsWith('x-anthropic-billing-header')) {
        attributionHeader = block
      } else if (CLI_SYSPROMPT_PREFIXES.has(block)) {
        systemPromptPrefix = block
      } else if (i < boundaryIndex) {
        staticBlocks.push(block)         // boundary 之前 = 静态
      } else {
        dynamicBlocks.push(block)        // boundary 之后 = 动态
      }
    }
    // 静态部分标记为 global scope
    result.push({ text: staticJoined, cacheScope: 'global' })
    // 动态部分不缓存
    result.push({ text: dynamicJoined, cacheScope: null })
  }
}
```

#### 场景 2: MCP Tools 场景（skipGlobalCacheForSystemPrompt=true）

当存在 MCP 工具时，system prompt 上不使用 global cache，改为在 tools 上标记缓存。

```
┌──────────────────────┬─────────────────┐
│ Attribution header   │ cacheScope=null │
│ CLI prefix           │ cacheScope='org' │  ← 降级为 org 级别
│ Everything else      │ cacheScope='org' │
└──────────────────────┴─────────────────┘
```

#### 场景 3: 默认模式（3P 代理/Bedrock/Vertex，或 boundary 缺失）

```
┌──────────────────────┬─────────────────┐
│ Attribution header   │ cacheScope=null │
│ CLI prefix           │ cacheScope='org' │
│ Everything else      │ cacheScope='org' │
└──────────────────────┴─────────────────┘
```

#### buildSystemPromptBlocks 的最终输出

```typescript
// claude.ts:3213-3237
export function buildSystemPromptBlocks(
  systemPrompt, enablePromptCaching, options
): TextBlockParam[] {
  return splitSysPromptPrefix(systemPrompt, {
    skipGlobalCacheForSystemPrompt: options?.skipGlobalCacheForSystemPrompt,
  }).map(block => ({
    type: 'text',
    text: block.text,
    ...(enablePromptCaching && block.cacheScope !== null && {
      cache_control: getCacheControl({
        scope: block.cacheScope,
        querySource: options?.querySource,
      }),
    }),
  }))
}
```

---

## 深入七：完整数据流图

```
┌─────────────────────────────────────────────────────────────────┐
│                    fetchSystemPromptParts()                       │
│                     (Promise.all 并行)                            │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐    │
│  │ getSystemPrompt()│ │ getUserContext() │ │getSystemContext()│   │
│  │                  │ │                  │ │                  │   │
│  │ prompts.ts:444   │ │ context.ts:155   │ │ context.ts:116   │   │
│  │                  │ │                  │ │                  │   │
│  │ ┌──── Static ──┐ │ │ getMemoryFiles() │ │ getGitStatus()   │   │
│  │ │ [0] Intro    │ │ │  8-level walk    │ │  5 git cmds      │   │
│  │ │ [1] System   │ │ │  ↓               │ │  branch, main,   │   │
│  │ │ [2] DoTasks  │ │ │ getClaudeMds()   │ │  status, log,    │   │
│  │ │ [3] Actions  │ │ │  filter+concat   │ │  user.name       │   │
│  │ │ [4] Tools    │ │ │  ↓               │ │  ↓               │   │
│  │ │ [5] Tone     │ │ │ { claudeMd,      │ │ { gitStatus }    │   │
│  │ │ [6] Output   │ │ │   currentDate }  │ │                  │   │
│  │ ├── BOUNDARY ──┤ │ │                  │ │                  │   │
│  │ │ [7] Session  │ │ │  memoized        │ │  memoized        │   │
│  │ │ [8] Memory   │ │ │  per session     │ │  per session     │   │
│  │ │ [9] Env      │ │ └─────────────────┘ └─────────────────┘    │
│  │ │ [10] Lang    │ │            │                   │            │
│  │ │ [11] Style   │ │            │                   │            │
│  │ │ [12] MCP (!) │ │            │                   │            │
│  │ │ [13] Scratch │ │            │                   │            │
│  │ │ [14] FRC     │ │            │                   │            │
│  │ │ [15] Summary │ │            │                   │            │
│  │ └──────────────┘ │            │                   │            │
│  └────────┬─────────┘            │                   │            │
│           │                      │                   │            │
└───────────┼──────────────────────┼───────────────────┼────────────┘
            │                      │                   │
            ▼                      │                   │
  ┌─────────────────────┐          │                   │
  │ QueryEngine.ts:321  │          │                   │
  │ asSystemPrompt([    │          │                   │
  │   ...defaultSysP,   │          │                   │
  │   ...memMechanics?, │          │                   │
  │   ...appendSysP?,   │          │                   │
  │ ])                  │          │                   │
  └─────────┬───────────┘          │                   │
            │                      │                   │
            ▼                      │                   │
  ┌─────────────────────────────────────────────────┐  │
  │ query.ts: query loop                            │  │
  │                                                 │  │
  │ 1. Messages 压缩链:                              │  │
  │    toolResultBudget → snip → microcompact       │  │
  │    → contextCollapse → autocompact              │  │
  │                                                 │  │
  │ 2. appendSystemContext(systemPrompt, ────────────┼──┘
  │                         systemContext)           │
  │    → gitStatus 追加到 prompt 尾部                 │
  │    → fullSystemPrompt                           │
  │                                                 │
  │ 3. callModel({                                  │
  │      messages: prependUserContext(msgs, ─────────┼── userContext
  │                                  userContext),   │   (claudeMd+currentDate)
  │      systemPrompt: fullSystemPrompt,            │   注入为 messages[0]
  │      ...                                        │
  │    })                                           │
  └──────────────────────┬──────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────┐
  │ claude.ts:1017 queryModel()                     │
  │                                                 │
  │ 1. Tool Schema Build                            │
  │    - toolToAPISchema() for each tool            │
  │    - defer_loading for deferred tools           │
  │    - session-stable schema cache                │
  │                                                 │
  │ 2. System Prompt Header                         │
  │    [attribution header]                         │
  │    [CLI prefix]                                 │
  │    [...systemPrompt]                            │
  │    [advisor instructions]                       │
  │                                                 │
  │ 3. buildSystemPromptBlocks()                    │
  │    → splitSysPromptPrefix()                     │
  │    → SystemPromptBlock[] with cacheScope        │
  │    → TextBlockParam[] for API                   │
  │                                                 │
  │ 4. Final API Call                               │
  │    model, messages, system, tools, betas, ...   │
  └──────────────────────┬──────────────────────────┘
                         │
                         ▼
               Anthropic Messages API
```

---

## 设计决策分析

### 决策 1: Static/Dynamic 分离实现全局缓存共享

**问题**: Anthropic 的 prompt cache 要求前缀精确匹配。如果 system prompt 有任何差异，缓存就无法命中。

**解法**: 将 system prompt 分为静态和动态两部分，用 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 标记分界。

**收益**:
- 全球所有 Claude Code 用户共享静态部分的 KV cache
- 静态部分约占 system prompt 的 60-70%
- 首次请求的延迟可能减少数秒（KV cache 命中意味着不需要重新计算 attention）

**代价**:
- 开发者在修改 prompt 时需要注意 section 的位置
- 如果不小心在静态区域添加了动态内容，会导致全局缓存碎片化
- 需要额外的工程规范来维护边界

### 决策 2: CLAUDE.md 在 messages 中而不是 system prompt 中

**问题**: CLAUDE.md 内容因项目而异，如果放在 system prompt 中会破坏全局缓存。

**解法**: 将 CLAUDE.md 包裹在 `<system-reminder>` 标签中，作为 `messages[0]` 注入。

**权衡分析**:

| 维度 | 放在 system prompt | 放在 messages[0] |
|------|-------------------|-----------------|
| 缓存影响 | 破坏全局缓存 | 不影响 |
| 指令权重 | 最高（system level） | 较高（user level） |
| 更新频率 | 每次 prompt 变化都重建 | 仅影响 messages |
| 实现复杂度 | 简单 | 需要 prependUserContext |

### 决策 3: gitStatus 在 dynamic zone 的尾部

**问题**: 每个用户的 git 仓库状态不同，但模型需要了解当前上下文。

**解法**: `appendSystemContext()` 在 query loop 中将 gitStatus 追加到 system prompt 尾部，位于 dynamic boundary 之后。

**为什么不放在 messages 中**（像 CLAUDE.md 那样）？因为 gitStatus 的信息密度较低（几十行文本），放在 system prompt 尾部的 overhead 很小，但好处是模型不需要从 messages 中"寻找"它 —— 它就在 system prompt 中。

### 决策 4: DANGEROUS_uncached 的命名约定

**问题**: 任何动态 section 都有破坏缓存的风险，但有些是不可避免的。

**解法**: 通过函数命名强制 code review 关注。

```typescript
// 普通 section - 只计算一次
systemPromptSection('env_info_simple', () => computeSimpleEnvInfo(...))

// 危险 section - 每轮重算，必须解释原因
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns',  // 强制解释
)
```

**[可借鉴]** 这是一个值得学习的工程实践：**用命名强制审查**。当你在 code review 中看到 `DANGEROUS_` 前缀，你自然会多看几眼。第三个参数 `_reason` 虽然在运行时不使用，但它在源码中记录了设计决策。

---

## 常见误区

### 误区 1: "system prompt 就是一个字符串"

**事实**: system prompt 是一个 `string[]`（字符串数组），每个元素是一个独立的 section。最终通过 `splitSysPromptPrefix()` 转换为 `SystemPromptBlock[]`，每个 block 有独立的 `cacheScope`。发送给 API 时，它们变成 `TextBlockParam[]`，每个 block 有独立的 `cache_control`。

### 误区 2: "CLAUDE.md 是 system prompt 的一部分"

**事实**: CLAUDE.md 不在 system prompt 中。它被 `prependUserContext()` 注入到 `messages[0]`，作为一条 `isMeta: true` 的 user message 存在。这是为了保护 system prompt 的全局缓存。

### 误区 3: "git status 每轮都会更新"

**事实**: `getGitStatus()` 和 `getSystemContext()` 都是 memoized 的，整个会话只计算一次。prompt 中甚至告诉模型这是一个 snapshot。模型如果需要最新状态，可以通过 BashTool 执行 `git status`。

### 误区 4: "添加一个 dynamic section 代价不大"

**事实**: 每个 `DANGEROUS_uncachedSystemPromptSection` 每轮都重新计算。如果返回值在两轮之间变化，整个动态区域的 prompt cache 都会失效。这可能意味着数万 token 的重新计算。

### 误区 5: "customSystemPrompt 只是在默认 prompt 前面/后面追加"

**事实**: `customSystemPrompt` 是**完全替换**，不是追加。设置后，`getSystemPrompt()` 和 `getSystemContext()` 都被跳过（返回空对象）。如果需要追加，应该使用 `appendSystemPrompt`。

### 误区 6: "所有用户看到的 system prompt 都一样"

**事实**: `process.env.USER_TYPE === 'ant'` 在多个 section 中引入了内部/外部版本差异：
- [2] Doing Tasks: 内部版本有更严格的注释规范和验证要求
- [6] Output Efficiency: 完全不同的两个版本
- 部分 section 有 ant-only 的额外指令

---

## 跨模块联系

### 与 Part 1 对话引擎的关系

prompt 组装是对话引擎中 query loop 的一个子系统。对话引擎负责循环执行 "发请求 → 解析响应 → 执行工具 → 继续" 的流程，而 prompt 组装负责每次发请求前的准备工作。

```
对话引擎 (Part 1)
├── query loop
│   ├── 消息压缩 (toolResultBudget, snip, microcompact, contextCollapse, autocompact)
│   ├── ★ prompt 组装 (本篇 Part 2) ★
│   │   ├── appendSystemContext
│   │   └── prependUserContext
│   ├── queryModel (API 调用)
│   ├── 响应解析
│   └── 工具执行
└── 会话管理
```

### 与 Tool System 的关系

prompt 组装中的 [4] Using Tools section 直接影响模型如何选择工具。工具 schema 在 `queryModel()` 中构建，与 system prompt 一起发送给 API。两者配合决定了模型的行为：
- system prompt 说"优先使用专用工具"
- tool schema 提供了工具的能力描述和输入格式
- 两者必须一致，否则模型会困惑

### 与 MCP 系统的关系

MCP instructions 是唯一的 DANGEROUS section，也是导入外部指令的主要通道。MCP 系统的连接/断开直接影响 prompt 内容，这也是为什么需要 delta mode 来减缓缓存破坏。

### 与缓存系统的关系

prompt 组装的核心目标之一就是优化缓存。`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 和 `splitSysPromptPrefix()` 的设计直接服务于 Anthropic API 的 prompt caching 功能。缓存层级：

```
Global cache (全球共享)
  └── 静态 sections [0-6]
Org cache (组织级共享)
  └── 降级场景（MCP, 3P providers）
Session cache (会话级)
  └── systemPromptSection 的 memoize
  └── getUserContext / getSystemContext 的 memoize
  └── toolSchemaCache
No cache
  └── DANGEROUS_uncachedSystemPromptSection
```

---

## 核心 Takeaway

1. **五步流水线**: fetchParts → QueryEngine 组装 → queryLoop 预处理 → queryModel 最终组装 → 三层缓存标记。每一步都有明确的职责边界。

2. **SYSTEM_PROMPT_DYNAMIC_BOUNDARY 是最重要的优化标记**: 它划分了全局可缓存的静态内容和每用户独立的动态内容。这一个字符串的存在与否，直接影响全球所有 Claude Code 用户的 API 延迟。

3. **三大原材料并行获取、分别注入**: defaultSystemPrompt 构成 system prompt 的主体，userContext 注入到 messages[0]（保护缓存），systemContext 追加到 system prompt 尾部（在 dynamic zone 中）。

4. **DANGEROUS 命名约定是工程文化**: 它不仅是技术约束，更是 code review 的信号。每个动态 section 都有缓存代价，用命名强制开发者思考这个代价。

5. **缓存 > 一切**: 整个设计的核心目标是最大化 prompt cache 命中率。CLAUDE.md 不放在 system prompt 里、gitStatus 放在 dynamic zone 尾部、工具 schema 有 session-stable 缓存、MCP instructions 有 delta mode —— 所有这些设计决策都指向同一个目标。

6. **消息压缩链保证了长对话的可行性**: 五层压缩机制（toolResultBudget → snip → microcompact → contextCollapse → autocompact）确保即使对话很长，token 消耗也在可控范围内。

---

## 思考题

1. **缓存粒度权衡**: 当前设计将 system prompt 分为"全局静态"和"动态"两部分。如果要引入第三层 —— 比如"组织级静态"（同一个组织的所有用户共享），你会把哪些 section 放在这一层？需要修改哪些模块？

2. **CLAUDE.md 位置的替代方案**: 如果 Anthropic API 未来支持"system prompt 中指定哪些部分不参与缓存 key 计算"（类似 `cache_control: { exclude_from_key: true }`），CLAUDE.md 还需要放在 messages[0] 吗？这个假设性的 API 变更会简化还是复杂化当前架构？

3. **MCP Delta Mode 的深层含义**: MCP instructions 从 system prompt 迁移到 message attachments 后，它的"指令权重"是否改变了？模型对 system prompt 中的指令 vs message 中的指令的注意力分配有什么不同？这是否会影响 MCP 工具的使用质量？

4. **动态 Section 的未来**: 目前只有一个 DANGEROUS section（MCP instructions）。如果未来需要添加更多实时变化的内容（如用户在线状态、协作者信息），你会如何设计来最小化缓存破坏？考虑 delta mode 的思路能否推广？

5. **压缩链的副作用**: 五层消息压缩在 Step 3 中执行，它会改变 messages 的内容。但 prependUserContext 在压缩之后执行，意味着 CLAUDE.md 的 `messages[0]` 不会被压缩。这是有意为之还是巧合？如果 CLAUDE.md 内容非常大（比如 10000 tokens），这个设计是否需要调整？

---

## 关键源文件

| 文件 | 核心职责 |
|------|---------|
| `src/utils/queryContext.ts` | `fetchSystemPromptParts()` —— 三大原材料并行获取入口 |
| `src/constants/prompts.ts` | `getSystemPrompt()` —— 15 个 section 的定义和组装 |
| `src/constants/systemPromptSections.ts` | `systemPromptSection()` / `DANGEROUS_uncachedSystemPromptSection()` —— section 注册机制 |
| `src/context.ts` | `getUserContext()` / `getSystemContext()` —— userContext 和 systemContext 的获取 |
| `src/utils/claudemd.ts` | `getMemoryFiles()` / `getClaudeMds()` —— CLAUDE.md 8 层加载 |
| `src/QueryEngine.ts` | `submitMessage()` —— Step 2 组装逻辑 |
| `src/query.ts` | query loop —— Step 3 消息预处理和上下文注入 |
| `src/utils/api.ts` | `appendSystemContext()` / `prependUserContext()` / `splitSysPromptPrefix()` —— 关键组装函数 |
| `src/services/api/claude.ts` | `queryModel()` / `buildSystemPromptBlocks()` —— Step 4-5 最终组装和缓存标记 |
| `src/constants/cyberRiskInstruction.ts` | `CYBER_RISK_INSTRUCTION` —— 安全团队维护的核心安全指令 |

---

> **下一篇预告**: Part 3 将深入 Claude Code 的工具系统 —— 从 Tool 接口定义到权限模型，从 BashTool 的沙箱机制到 MCP 工具的动态加载。
