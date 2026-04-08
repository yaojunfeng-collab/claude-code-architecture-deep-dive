# Part 5: Tool 系统 — Claude 是怎么和外部世界交互的？

> Claude Code 2.1.88 Architecture Deep Dive Series (5/10)
>
> 上一篇我们探讨了 Context 生命周期管理，理解了对话窗口如何被精心管理。
> 这一篇我们深入 Tool 系统 —— Claude 唯一的 "手和脚"。

---

## 一、开场：为什么 Tool 系统是 Claude Code 的灵魂？

一个关键认知：**Claude 模型本身只有语言能力**。它不能读文件、不能执行命令、不能搜索代码。
所有与外部世界的交互，都必须通过 Tool 系统完成。

```
┌─────────────────────────────────────────────────┐
│                   Claude 模型                     │
│  (纯语言推理：理解意图、规划步骤、生成参数)        │
└──────────────────────┬──────────────────────────┘
                       │ tool_use blocks
                       ▼
┌─────────────────────────────────────────────────┐
│                  Tool 系统                        │
│  验证 → 权限 → 执行 → 结果处理 → 回传模型         │
└──────────────────────┬──────────────────────────┘
                       │ side effects
                       ▼
          文件系统 / Shell / 网络 / MCP 服务
```

> [核心概念] **Reasoning 和 Action 的明确分离**：模型负责 "想"，Tool 系统负责 "做"。
> 这不是实现细节，而是架构根基。它决定了安全模型、并发策略、权限体系的所有设计。

---

## 二、全景：Tool 系统的五层架构

```
Layer 5: Tool Orchestration (并发调度)
         toolOrchestration.ts — partitionToolCalls → parallel/serial batches

Layer 4: Execution Pipeline (单 tool 执行 9 步流水线)
         toolExecution.ts — checkPermissionsAndCallTool

Layer 3: Permission & Hooks (权限决策)
         toolHooks.ts — PreToolUse/PostToolUse hooks
         bashPermissions.ts — multi-layer permission chain
         resolveHookPermissionDecision — priority chain

Layer 2: Tool Pool Assembly (工具池组装)
         tools.ts — getAllBaseTools() + assembleToolPool()
         toolPool.ts — mergeAndFilterTools()

Layer 1: Tool Definition (工具定义)
         Tool.ts — Tool interface (7 大类字段)
         buildTool() — defaults + type inference
```

---

## 三、完整的内置 Tool 列表

### 3.1 getAllBaseTools() — 所有工具的 Source of Truth

源文件 `src/tools.ts` 中的 `getAllBaseTools()` 函数是**全部 tool 的注册中心**。

> [设计决策] 函数头部有一行关键注释：
> `NOTE: This MUST stay in sync with statsig global system caching config`
> 这意味着每次增删 tool，都要同步更新 Statsig 配置，否则会导致 system prompt 缓存失效。

**核心 Tool（无条件加载）：**

| Tool 名称 | 职责 | 文件 |
|-----------|------|------|
| BashTool | Shell 命令执行 | `BashTool/BashTool.tsx` |
| FileReadTool | 文件读取 | `FileReadTool/FileReadTool.ts` |
| FileEditTool | 文件编辑（精确替换） | `FileEditTool/FileEditTool.ts` |
| FileWriteTool | 文件写入（完整覆盖） | `FileWriteTool/FileWriteTool.ts` |
| GlobTool | 文件名模式匹配 | `GlobTool/GlobTool.ts` |
| GrepTool | 内容搜索 | `GrepTool/GrepTool.ts` |
| WebFetchTool | 网页内容获取 | `WebFetchTool/WebFetchTool.ts` |
| WebSearchTool | 网络搜索 | `WebSearchTool/WebSearchTool.ts` |
| AgentTool | 子 Agent 创建 | `AgentTool/AgentTool.tsx` |
| TaskOutputTool | Agent 输出结果 | `TaskOutputTool/TaskOutputTool.tsx` |
| TaskStopTool | 停止当前任务 | `TaskStopTool/TaskStopTool.ts` |
| SendMessageTool | Agent 间消息传递 | `SendMessageTool/SendMessageTool.ts` |
| AskUserQuestionTool | 向用户提问 | `AskUserQuestionTool/AskUserQuestionTool.tsx` |
| TodoWriteTool | 任务列表管理 | `TodoWriteTool/TodoWriteTool.ts` |
| EnterPlanModeTool | 进入规划模式 | `EnterPlanModeTool/EnterPlanModeTool.ts` |
| ExitPlanModeV2Tool | 退出规划模式 | `ExitPlanModeTool/ExitPlanModeV2Tool.ts` |
| SkillTool | 技能调用 | `SkillTool/SkillTool.ts` |
| NotebookEditTool | Jupyter Notebook 编辑 | `NotebookEditTool/NotebookEditTool.ts` |
| BriefTool | 简要回复模式 | `BriefTool/BriefTool.ts` |

**特殊说明：GlobTool/GrepTool 的条件加载**

```typescript
// 当 ant-native 构建中内嵌了 bfs/ugrep 时，这两个 tool 不需要
...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
```

> [设计决策] 这是为了内部 ant 用户的优化 —— 嵌入到 bun binary 中的搜索工具更快，
> 此时 Glob/Grep 的独立 tool 就多余了。

### 3.2 条件加载的 Tool（Feature Flag 控制）

```typescript
// Dead code elimination: conditional import for ant-only tools
const REPLTool = process.env.USER_TYPE === 'ant'
    ? require('./tools/REPLTool/REPLTool.js').REPLTool : null

const SleepTool = feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool : null

const cronTools = feature('AGENT_TRIGGERS') ? [
    CronCreateTool, CronDeleteTool, CronListTool
] : []

const WorkflowTool = feature('WORKFLOW_SCRIPTS')
    ? (() => {
        require('./tools/WorkflowTool/bundled/index.js').initBundledWorkflows()
        return require('./tools/WorkflowTool/WorkflowTool.js').WorkflowTool
      })() : null
```

| 条件 Tool | 控制条件 | 用途 |
|-----------|---------|------|
| REPLTool | `USER_TYPE === 'ant'` | 虚拟机内执行，包裹原始 tools |
| SleepTool | `PROACTIVE \|\| KAIROS` | 延迟执行 |
| CronTools | `AGENT_TRIGGERS` | 定时任务 |
| WorkflowTool | `WORKFLOW_SCRIPTS` | 工作流脚本 |
| PowerShellTool | `isPowerShellToolEnabled()` | Windows PowerShell |
| LSPTool | `ENABLE_LSP_TOOL` env | Language Server Protocol |
| WorktreeTools | `isWorktreeModeEnabled()` | Git worktree 隔离 |
| SnipTool | `HISTORY_SNIP` | 历史消息裁剪 |
| WebBrowserTool | `WEB_BROWSER_TOOL` | 浏览器自动化 |

> [权衡] 使用 `require()` 而非 `import` 是有意为之 —— 这样 bundler 可以做 dead code elimination。
> 如果 feature flag 为 false，整个模块不会被打包进最终产物。

### 3.3 ToolSearchTool 的特殊地位

```typescript
// Include ToolSearchTool when tool search might be enabled (optimistic check)
...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
```

ToolSearchTool 自身**永远不会被 defer**：

```typescript
// Never defer ToolSearch itself — the model needs it to load everything else
if (tool.name === TOOL_SEARCH_TOOL_NAME) return false
```

> [核心概念] ToolSearchTool 是 defer 机制的"锚点" —— 没有它，被 defer 的 tool 永远无法被发现。
> 这是典型的 bootstrap 问题：loader 自己不能被 lazy load。

---

## 四、MCP Tool 的 Defer 加载策略

### 4.1 isDeferredTool() 的决策逻辑

```typescript
// src/tools/ToolSearchTool/prompt.ts
export function isDeferredTool(tool: Tool): boolean {
  // 1. alwaysLoad 优先级最高 — 显式 opt-out
  if (tool.alwaysLoad === true) return false

  // 2. MCP tools 默认全部 defer
  if (tool.isMcp === true) return true

  // 3. ToolSearch 自身永不 defer
  if (tool.name === TOOL_SEARCH_TOOL_NAME) return false

  // 4. 特定 tool 的豁免（Agent、Brief 等）
  if (feature('FORK_SUBAGENT') && tool.name === AGENT_TOOL_NAME) {
    if (isForkSubagentEnabled()) return false
  }

  // 5. shouldDefer 标记
  return tool.shouldDefer === true
}
```

### 4.2 三类 MCP Tool 与缓存影响

```
┌──────────────────────────────────────────────────────────────────────┐
│                    MCP Tool 分类与缓存策略                            │
├─────────────┬──────────────┬─────────────────────────────────────────┤
│   分类       │ 在 tools[]？ │ 对全局缓存的影响                         │
├─────────────┼──────────────┼─────────────────────────────────────────┤
│ alwaysLoad  │ ✅ 在 tools  │ ⚠️ 破坏全局缓存（每用户不同）             │
│             │ 完整 schema  │ 因为不同用户的 MCP 集合不同                │
├─────────────┼──────────────┼─────────────────────────────────────────┤
│ 普通 MCP    │ ❌ defer     │ ✅ 不影响缓存                            │
│             │ 仅名称可见   │ 通过 ToolSearch 按需加载                  │
├─────────────┼──────────────┼─────────────────────────────────────────┤
│ 内置 defer  │ ❌ defer     │ ✅ 不影响缓存                            │
│ shouldDefer │ 仅名称可见   │ 如 NotebookEdit, TodoWrite 等           │
└─────────────┴──────────────┴─────────────────────────────────────────┘
```

### 4.3 Auto 模式的阈值计算

```typescript
// 当 deferred tool 总 token 数超过 context window 的 N% 时启用
const DEFAULT_AUTO_TOOL_SEARCH_PERCENTAGE = 10 // 10%

function getAutoToolSearchTokenThreshold(model: string): number {
  const contextWindow = getContextWindowForModel(model, betas)
  const percentage = getAutoToolSearchPercentage() / 100
  return Math.floor(contextWindow * percentage)
}
```

> [设计决策] 为什么默认 10%？因为 tool 定义在 context window 中占据的空间是固定成本。
> 当 MCP tool 太多时，留给实际对话的空间就太少了。defer 后只传名称，
> 按需加载 schema，把固定成本变成了按需成本。

---

## 五、Tool 注册与组装：assembleToolPool()

### 5.1 排序策略

```typescript
// src/tools.ts
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // 关键：built-in 和 MCP 分别排序，built-in 作为连续前缀
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

> [设计决策] **为什么不做全局 flat sort？**
>
> 服务端的 `claude_code_system_cache_policy` 在最后一个 built-in tool 之后放置了
> 全局 cache breakpoint。如果 flat sort 把 MCP tool 插入到 built-in 之间，
> 每次 MCP tool 变动（连接/断开）都会 invalidate 所有下游 cache keys。
>
> **分区排序保证了 built-in 前缀的稳定性 —— 这直接关系到 prompt cache 命中率。**

### 5.2 名字冲突解决

```typescript
// uniqBy 保留插入顺序，built-in 先插入所以优先
return uniqBy(
  [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
  'name',
)
```

如果一个 MCP tool 名字和 built-in 冲突，**built-in 胜出**。这防止了恶意 MCP server 覆盖核心 tool。

### 5.3 mergeAndFilterTools() — React 层的合并

```typescript
// src/utils/toolPool.ts
export function mergeAndFilterTools(
  initialTools: Tools,
  assembled: Tools,
  mode: ToolPermissionContext['mode'],
): Tools {
  // 分区: MCP vs built-in
  const [mcp, builtIn] = partition(
    uniqBy([...initialTools, ...assembled], 'name'),
    isMcpTool,
  )
  // 各自排序，built-in 在前
  const tools = [...builtIn.sort(byName), ...mcp.sort(byName)]

  // Coordinator 模式过滤
  if (coordinatorModeModule?.isCoordinatorMode()) {
    return applyCoordinatorToolFilter(tools)
  }
  return tools
}
```

> [可借鉴] 这个函数被特意放在 React-free 文件中，这样 print.ts（SDK 路径）
> 也能 import 而不会把 react/ink 拖入模块图。这种"逻辑与 UI 框架解耦"的做法值得学习。

---

## 六、Agent Tool 隔离策略

不同角色的 Agent 看到的 tool 集合截然不同：

### 6.1 四种隔离级别

```
┌───────────────────────────────────────────────────────────────────┐
│ 角色              │ 可用 Tool                │ 控制常量             │
├───────────────────┼─────────────────────────┼─────────────────────┤
│ Main Thread       │ ALL tools               │ getAllBaseTools()    │
│                   │ (经 isEnabled 过滤)      │                     │
├───────────────────┼─────────────────────────┼─────────────────────┤
│ Sub-agent         │ All MINUS 禁止列表       │ ALL_AGENT_          │
│ (built-in)        │ -TaskOutput             │ DISALLOWED_TOOLS    │
│                   │ -ExitPlanMode           │                     │
│                   │ -EnterPlanMode          │                     │
│                   │ -Agent* (外部用户)       │                     │
│                   │ -AskUserQuestion        │                     │
│                   │ -TaskStop               │                     │
│                   │ -WorkflowTool           │                     │
├───────────────────┼─────────────────────────┼─────────────────────┤
│ Coordinator       │ ONLY 4 tools:           │ COORDINATOR_MODE_   │
│                   │ Agent, TaskStop,         │ ALLOWED_TOOLS       │
│                   │ SendMessage,             │                     │
│                   │ SyntheticOutput          │                     │
├───────────────────┼─────────────────────────┼─────────────────────┤
│ Async Agent       │ 白名单制:               │ ASYNC_AGENT_        │
│                   │ Read/Edit/Write/Glob/    │ ALLOWED_TOOLS       │
│                   │ Grep/Bash/WebSearch/     │                     │
│                   │ WebFetch/Todo/Notebook/  │                     │
│                   │ Skill/ToolSearch/        │                     │
│                   │ Worktree tools           │                     │
└───────────────────┴─────────────────────────┴─────────────────────┘
```

### 6.2 filterToolsForAgent() 的逻辑

```typescript
// src/tools/AgentTool/agentToolUtils.ts
export function filterToolsForAgent({
  tools, isBuiltIn, isAsync = false, permissionMode,
}: { ... }): Tools {
  // built-in agent: 移除 ALL_AGENT_DISALLOWED_TOOLS
  // custom agent:   移除 CUSTOM_AGENT_DISALLOWED_TOOLS
  // async agent:    只保留 ASYNC_AGENT_ALLOWED_TOOLS
}
```

> [设计决策] **为什么 Coordinator 只有 4 个 tool？**
>
> Coordinator 模式下，主线程只做编排（dispatch work to agents），
> 不自己读写文件。这是 "separation of concerns" 的极致体现：
> 编排者不需要（也不应该有）直接操作环境的能力。

---

## 七、并发调度：toolOrchestration.ts

### 7.1 partitionToolCalls() 的分区算法

当模型一次返回多个 `tool_use` blocks 时，系统需要决定哪些可以并行、哪些必须串行。

```typescript
function partitionToolCalls(
  toolUseMessages: ToolUseBlock[],
  toolUseContext: ToolUseContext,
): Batch[] {
  return toolUseMessages.reduce((acc: Batch[], toolUse) => {
    const tool = findToolByName(toolUseContext.options.tools, toolUse.name)
    const parsedInput = tool?.inputSchema.safeParse(toolUse.input)

    // 关键判断：是否并发安全
    const isConcurrencySafe = parsedInput?.success
      ? (() => {
          try {
            return Boolean(tool?.isConcurrencySafe(parsedInput.data))
          } catch {
            return false // 保守策略：解析失败视为不安全
          }
        })()
      : false

    // 连续的 concurrencySafe calls 合并为一个并行 batch
    if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
      acc[acc.length - 1]!.blocks.push(toolUse)
    } else {
      acc.push({ isConcurrencySafe, blocks: [toolUse] })
    }
    return acc
  }, [])
}
```

### 7.2 执行模式

```
Model 返回: [Grep, Glob, FileEdit, Grep, Grep, FileWrite]

partitionToolCalls() 分区结果:

Batch 1: { concurrencySafe: true,  blocks: [Grep, Glob] }     → 并行
Batch 2: { concurrencySafe: false, blocks: [FileEdit] }       → 串行
Batch 3: { concurrencySafe: true,  blocks: [Grep, Grep] }     → 并行
Batch 4: { concurrencySafe: false, blocks: [FileWrite] }      → 串行

执行顺序: Batch1(并行) → Batch2 → Batch3(并行) → Batch4
```

### 7.3 并行度控制

```typescript
function getMaxToolUseConcurrency(): number {
  return parseInt(process.env.CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || '', 10) || 10
}
```

最大并发数默认 **10**，通过环境变量可调。底层使用自定义的 `all()` 工具函数控制并发。

### 7.4 contextModifier 的竞态处理

```typescript
// 并行执行时，contextModifier 被排队而非立即应用
const queuedContextModifiers: Record<
  string,
  ((context: ToolUseContext) => ToolUseContext)[]
> = {}

// 所有并行 tool 完成后，按 block 顺序依次应用
for (const block of blocks) {
  const modifiers = queuedContextModifiers[block.id]
  if (!modifiers) continue
  for (const modifier of modifiers) {
    currentContext = modifier(currentContext)
  }
}
```

> [设计决策] **为什么不在并行执行中直接 apply contextModifier？**
>
> 因为 `ToolUseContext` 是共享状态。如果两个并行 tool 同时修改 context，
> 结果取决于执行顺序 —— 这是 race condition。
> 排队后按 block ID 顺序应用，保证了**确定性**。

---

## 八、单 Tool 执行流水线（9 步）

`checkPermissionsAndCallTool()` 是每个 tool 调用的核心路径。

```
┌─────────────────────────────────────────────────────────────────┐
│              checkPermissionsAndCallTool() 9 步                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Step 1: Abort Check                                              │
│    if (abortController.signal.aborted) → yield cancel message    │
│                                                                  │
│ Step 2: Zod Schema Validation                                    │
│    tool.inputSchema.safeParse(input)                             │
│    失败时: formatZodValidationError + buildSchemaNotSentHint     │
│    (deferred tool 专属 hint: 提示模型先 ToolSearch)              │
│                                                                  │
│ Step 3: Business Logic Validation                                │
│    tool.validateInput?.(parsedInput.data, toolUseContext)         │
│    BashTool: detectBlockedSleepPattern()                         │
│                                                                  │
│ Step 4: Security Defense                                         │
│    4a: Speculative Classifier (Bash only)                        │
│        startSpeculativeClassifierCheck() 并行启动分类器           │
│    4b: _simulatedSedEdit stripping                               │
│        从 model input 中移除内部字段，防止绕过权限               │
│    4c: backfillObservableInput — 添加衍生字段                    │
│                                                                  │
│ Step 5: PreToolUse Hooks                                         │
│    runPreToolUseHooks() → allow/deny/ask/modify/stop             │
│    Hook 可以: 修改 input、决定权限、阻止继续                     │
│                                                                  │
│ Step 6: Permission Decision                                      │
│    resolveHookPermissionDecision() 优先级链:                     │
│    hook result > tool.checkPermissions > canUseTool(UI)          │
│                                                                  │
│ Step 7: tool.call() Execution                                    │
│    processedInput 经过三版本演化:                                │
│    v1: 原始 API input (保留 prompt cache)                        │
│    v2: backfilled clone (hooks/observers 看到)                   │
│    v3: hook updatedInput (如有) 替换 callInput                   │
│                                                                  │
│ Step 8: PostToolUse Hooks                                        │
│    runPostToolUseHooks() — 可观察但不可阻止                      │
│                                                                  │
│ Step 9: Result Processing & Telemetry                            │
│    processToolResultBlock() — 大结果持久化                       │
│    logEvent('tengu_tool_use_*') — 遥测上报                      │
│    contextModifier — 上下文修改                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.1 Step 2: Deferred Tool 的 Schema 缺失处理

```typescript
// 当 deferred tool 被调用但 schema 未加载时
export function buildSchemaNotSentHint(
  tool: Tool, messages: Message[], tools: readonly { name: string }[],
): string | null {
  if (!isDeferredTool(tool)) return null
  const discovered = extractDiscoveredToolNames(messages)
  if (discovered.has(tool.name)) return null

  return `This tool's schema was not sent to the API...
  Load the tool first: call ${TOOL_SEARCH_TOOL_NAME} with
  query "select:${tool.name}", then retry this call.`
}
```

> [可借鉴] 这是一个优雅的自愈机制 —— 当模型"抢跑"调用还没加载 schema 的 tool 时，
> 错误消息会告诉它正确的补救步骤。

### 8.2 Step 4: _simulatedSedEdit 的双重防御

第一层防御：Schema 层面

```typescript
// inputSchema 中已经 omit 了 _simulatedSedEdit
const inputSchema = lazySchema(() =>
  fullInputSchema().omit({ _simulatedSedEdit: true })
)
```

第二层防御：运行时剥离

```typescript
// toolExecution.ts 中的 defense-in-depth
if (tool.name === BASH_TOOL_NAME && '_simulatedSedEdit' in processedInput) {
  const { _simulatedSedEdit: _, ...rest } = processedInput
  processedInput = rest
}
```

> [设计决策] 为什么需要两层？因为 `_simulatedSedEdit` 是权限系统内部注入的字段，
> 它允许绕过 sandbox 直接写文件。如果模型能构造这个字段，就能绕过所有安全检查。
> 双重防御确保即使一层失效，另一层仍然有效。

### 8.3 Step 6: resolveHookPermissionDecision 优先级链

```
Hook 返回 allow/deny → 直接采用 hook 结果
Hook 返回 undefined → 走 tool.checkPermissions()
tool.checkPermissions() 返回 ask → 走 canUseTool (UI prompt)
canUseTool 返回 allow/deny → 最终结果
```

---

## 九、完整的 Tool Interface 字段全景

`buildTool()` 为每个 tool 提供默认值，确保接口一致：

### 9.1 七大类字段

**Category 1: 核心执行**

| 字段 | 类型 | 用途 |
|------|------|------|
| `name` | `string` | 唯一标识符 |
| `call()` | `async` | 实际执行逻辑 |
| `inputSchema` | `z.ZodType` | Zod schema，用于验证和 API 传输 |
| `inputJSONSchema?` | `ToolInputJSONSchema` | MCP 工具直接用 JSON Schema |
| `maxResultSizeChars` | `number` | 超过此值时结果持久化到磁盘 |

**Category 2: 权限控制**

| 字段 | 默认值 | 用途 |
|------|--------|------|
| `isEnabled()` | `true` | 是否启用 |
| `isReadOnly()` | `false` | 是否只读（保守：假设写入） |
| `isConcurrencySafe()` | `false` | 是否并发安全（保守：假设不安全） |
| `isDestructive()` | `false` | 是否不可逆操作 |
| `checkPermissions()` | `allow` | 自定义权限检查 |

> [设计决策] **Fail-closed defaults**: `isReadOnly` 和 `isConcurrencySafe` 默认 `false`。
> 这意味着新 tool 如果忘记实现这些方法，不会被错误地并行执行或跳过权限检查。

**Category 3: 权限集成**

| 字段 | 用途 |
|------|------|
| `preparePermissionMatcher()` | Hook `if` 条件匹配器（如 `Bash(git *)` 语法） |
| `backfillObservableInput()` | 为 hooks/observers 添加衍生字段 |
| `toAutoClassifierInput()` | 为 auto-mode 安全分类器提供紧凑输入 |

**Category 4: 延迟加载**

| 字段 | 用途 |
|------|------|
| `shouldDefer` | 是否需要 ToolSearch 才能调用 |
| `alwaysLoad` | 永不 defer（MCP 通过 `_meta['anthropic/alwaysLoad']` 设置） |
| `searchHint` | ToolSearch 关键词匹配提示 |

**Category 5: UI 渲染（6+ 方法）**

```typescript
renderToolUseMessage()           // tool 调用时的展示
renderToolResultMessage()        // 结果展示
renderToolUseProgressMessage()   // 执行中进度
renderToolUseQueuedMessage()     // 排队等待时
renderToolUseRejectedMessage()   // 权限拒绝时
renderToolUseErrorMessage()      // 错误时
renderGroupedToolUse()           // 多个同类 tool 分组展示
renderToolUseTag()               // 额外标签（timeout、model 等）
```

**Category 6: 描述与摘要**

| 字段 | 用途 |
|------|------|
| `prompt()` | 模型看到的 tool 描述 |
| `userFacingName()` | UI 中显示的名称（可动态） |
| `getToolUseSummary()` | 简洁摘要 |
| `getActivityDescription()` | Spinner 显示的活动描述 |
| `description()` | 单行描述 |

**Category 7: 结果处理**

| 字段 | 用途 |
|------|------|
| `mapToolResultToToolResultBlockParam()` | 将输出转为 API 格式 |
| `inputsEquivalent()` | 判断两次调用是否等效（推测执行用） |
| `extractSearchText()` | 为 transcript 搜索提取文本 |

### 9.2 buildTool() 的类型魔法

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,
  isReadOnly: (_input?: unknown) => false,
  isDestructive: (_input?: unknown) => false,
  checkPermissions: (input) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',
  userFacingName: (_input?: unknown) => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,  // 默认用 tool name
    ...def,                          // 用户定义覆盖默认
  } as BuiltTool<D>
}
```

> [可借鉴] `BuiltTool<D>` 类型利用了 TypeScript 的条件类型来精确推断：
> 如果 D 提供了某个字段，用 D 的类型；否则用默认类型。
> 这保证了 60+ 个 tool 的定义都通过类型检查，同时保持了 DX。

---

## 十、BashTool 深度剖析

BashTool 是整个 Tool 系统中**最复杂的单个 tool**，涉及安全、沙箱、并发、后台任务等多个关注点。

### 10.1 inputSchema 的隐藏字段

```typescript
// 完整 schema（内部使用）
const fullInputSchema = z.strictObject({
  command: z.string(),
  timeout: semanticNumber(z.number().optional()),
  description: z.string().optional(),
  run_in_background: semanticBoolean(z.boolean().optional()),
  dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional()),
  _simulatedSedEdit: z.object({         // ← 隐藏字段！
    filePath: z.string(),
    newContent: z.string()
  }).optional()
})

// 模型看到的 schema（移除了 _simulatedSedEdit）
const inputSchema = fullInputSchema().omit({ _simulatedSedEdit: true })
```

`_simulatedSedEdit` 是权限系统在用户批准 sed 编辑预览后注入的。它让 BashTool
跳过实际 sed 执行，直接写入预览内容，确保 "所见即所得"。

### 10.2 isConcurrencySafe / isReadOnly 逻辑

```typescript
isConcurrencySafe(input) {
  return this.isReadOnly?.(input) ?? false  // 只读才能并发
},

isReadOnly(input) {
  const compoundCommandHasCd = commandHasAnyCd(input.command)
  const result = checkReadOnlyConstraints(input, compoundCommandHasCd)
  return result.behavior === 'allow'
},
```

> [设计决策] `cd` 命令改变 CWD，虽然看起来 "只读"，但它修改了全局状态（Shell 工作目录），
> 所以包含 `cd` 的命令不能被视为 concurrencySafe。

### 10.3 isSearchOrReadCommand — UI 折叠判断

```typescript
const BASH_SEARCH_COMMANDS = new Set([
  'find', 'grep', 'rg', 'ag', 'ack', 'locate', 'which', 'whereis'
])
const BASH_READ_COMMANDS = new Set([
  'cat', 'head', 'tail', 'less', 'more',
  'wc', 'stat', 'file', 'strings',
  'jq', 'awk', 'cut', 'sort', 'uniq', 'tr'
])
const BASH_LIST_COMMANDS = new Set(['ls', 'tree', 'du'])
const BASH_SEMANTIC_NEUTRAL_COMMANDS = new Set([
  'echo', 'printf', 'true', 'false', ':'
])
```

判断规则：管道中的**所有** non-neutral 命令都必须是 search/read/list 才算可折叠。

```
ls dir && echo "---" && ls dir2    → isList: true  (echo 是 neutral)
cat file | jq '.data'              → isRead: true   (都是 read)
cat file | python script.py        → isRead: false  (python 不是 read)
```

### 10.4 preparePermissionMatcher — AST 级权限匹配

```typescript
async preparePermissionMatcher({ command }) {
  const parsed = await parseForSecurity(command)
  if (parsed.kind !== 'simple') {
    return () => true  // 无法解析 → 保守匹配所有
  }
  // 提取每个子命令的 argv（去除 VAR=val）
  const subcommands = parsed.commands.map(c => c.argv.join(' '))

  return pattern => {
    const prefix = permissionRuleExtractPrefix(pattern)
    return subcommands.some(cmd => {
      if (prefix !== null) {
        return cmd === prefix || cmd.startsWith(`${prefix} `)
      }
      return matchWildcardPattern(pattern, cmd)
    })
  }
}
```

这解决了复杂场景：

```
# 权限规则: Bash(git *)
# 命令: FOO=bar git push && echo done

解析后: subcommands = ["git push", "echo done"]
匹配: "git push".startsWith("git ") → true → hook 触发
```

> [设计决策] 这里使用 AST parsing 而非正则，因为 shell 语法太复杂了。
> `ls && git push` 必须被 `Bash(git *)` hook 捕获，否则就是安全漏洞。

### 10.5 validateInput — sleep 检测

```typescript
async validateInput(input: BashToolInput): Promise<ValidationResult> {
  if (feature('MONITOR_TOOL') && !isBackgroundTasksDisabled && !input.run_in_background) {
    const sleepPattern = detectBlockedSleepPattern(input.command)
    if (sleepPattern !== null) {
      return {
        result: false,
        message: `Blocked: ${sleepPattern}. Run blocking commands in the
        background with run_in_background: true...`,
        errorCode: 10
      }
    }
  }
  return { result: true }
}
```

> [可借鉴] 不是简单地禁止 sleep，而是：
> - `sleep 0.5` → 允许（rate limiting）
> - `sleep 1` → 允许（< 2s）
> - `sleep 5` → 拦截，建议用 Monitor tool
> - `sleep 5 && check` → 拦截，建议用 Monitor tool

### 10.6 checkPermissions — 多层权限链

BashTool 的权限检查是所有 tool 中最复杂的：

```
bashToolHasPermission() 调用链:

1. Mode check (bypassPermissions → allow)
2. Sed edit detection → SedEditPermissionRequest
3. Path validation (working directory check)
4. Per-subcommand permission (compound command splitting)
5. Semantic analysis (is it destructive?)
6. Speculative classifier (async, parallel with hooks)
7. Prompt user (if needed)
```

### 10.7 userFacingName — 动态名称

```typescript
userFacingName(input) {
  if (!input) return 'Bash'

  // sed -i 编辑 → 显示为 "Edit(file.ts)"
  if (input.command) {
    const sedInfo = parseSedEditCommand(input.command)
    if (sedInfo) {
      return fileEditUserFacingName({ file_path: sedInfo.filePath, old_string: 'x' })
    }
  }

  // sandbox 模式 → "SandboxedBash"
  return shouldUseSandbox(input) ? 'SandboxedBash' : 'Bash'
}
```

### 10.8 mapToolResultToToolResultBlockParam — 结果多态

```typescript
mapToolResultToToolResultBlockParam({
  stdout, stderr, isImage, backgroundTaskId,
  persistedOutputPath, persistedOutputSize, structuredContent, ...
}, toolUseID) {
  // 1. Structured content（MCP 风格）
  if (structuredContent?.length > 0) {
    return { tool_use_id: toolUseID, type: 'tool_result', content: structuredContent }
  }

  // 2. Image output → image content block
  if (isImage) {
    const block = buildImageToolResult(stdout, toolUseID)
    if (block) return block
  }

  // 3. Large output → persisted-output 引用
  if (persistedOutputPath) {
    processedStdout = buildLargeToolResultMessage({
      filepath: persistedOutputPath,
      originalSize: persistedOutputSize ?? 0,
      ...
    })
  }

  // 4. Background task → 包含 task ID 和输出路径信息
  if (backgroundTaskId) {
    backgroundInfo = `Command running in background with ID: ${backgroundTaskId}...`
  }

  // 5. 标准文本输出
  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content: [processedStdout, errorMessage, backgroundInfo].filter(Boolean).join('\n')
  }
}
```

### 10.9 call() 核心执行

```typescript
async call(input: BashToolInput, toolUseContext, _canUseTool, parentMessage, onProgress) {
  // 1. _simulatedSedEdit 快速路径
  if (input._simulatedSedEdit) {
    return applySedEdit(input._simulatedSedEdit, toolUseContext, parentMessage)
  }

  // 2. 创建 AsyncGenerator 版 runShellCommand
  const commandGenerator = runShellCommand({
    input, abortController,
    setAppState: toolUseContext.setAppStateForTasks ?? setAppState,
    preventCwdChanges: !isMainThread,
    ...
  })

  // 3. 消费 generator，转发 progress events
  let generatorResult
  do {
    generatorResult = await commandGenerator.next()
    if (!generatorResult.done && onProgress) {
      onProgress({
        toolUseID: `bash-progress-${progressCounter++}`,
        data: { type: 'bash_progress', output: progress.output, ... }
      })
    }
  } while (!generatorResult.done)

  // 4. Git 操作追踪
  trackGitOperations(input.command, result.code, result.stdout)

  // 5. claude-code-hint 协议处理
  const extracted = extractClaudeCodeHints(strippedStdout, input.command)
  strippedStdout = extracted.stripped  // 从输出中移除 hint 标签

  // 6. 图片输出压缩
  if (isImage) {
    const resized = await resizeShellImageOutput(strippedStdout, ...)
  }

  // 7. 大输出持久化
  if (result.outputFilePath && result.outputTaskId) {
    await link(result.outputFilePath, dest)  // 硬链接避免拷贝
    persistedOutputPath = dest
  }

  return { data: { stdout, stderr, isImage, persistedOutputPath, ... } }
}
```

### 10.10 runShellCommand 引擎

```
┌─────────────────────────────────────────────┐
│          runShellCommand 时间线               │
│                                              │
│ T=0     spawn 子进程                          │
│ T=0~2s  静默等待（无 progress）               │
│ T=2s    PROGRESS_THRESHOLD_MS 触发            │
│         开始发送 bash_progress events         │
│ T=2s+   每次 output 变化 → yield progress    │
│ T=15s   ASSISTANT_BLOCKING_BUDGET_MS          │
│         (assistant 模式) auto-background      │
│ T=N     命令完成 → return ExecResult          │
│                                              │
│ [若命令被 Ctrl+B] → 转入后台任务              │
│ [若命令 timeout] → 中断并返回已有输出         │
└─────────────────────────────────────────────┘
```

---

## 十一、完整的 BashTool 执行时间线

以 `git log --oneline -5` 为例：

```
Timeline:
─────────────────────────────────────────────────────────────────

1. [Model Output] tool_use: Bash({ command: "git log --oneline -5" })
   │
2. [toolOrchestration] partitionToolCalls()
   │ → isConcurrencySafe("git log --oneline -5")
   │ → isReadOnly() → checkReadOnlyConstraints() → 是只读
   │ → 归入 concurrent batch
   │
3. [toolExecution] runToolUse() → streamedCheckPermissionsAndCallTool()
   │
4. [Step 2] Zod validation: inputSchema.safeParse({command: "git log..."})
   │ → success ✓
   │
5. [Step 3] validateInput: detectBlockedSleepPattern("git log...")
   │ → null (不是 sleep) → pass ✓
   │
6. [Step 4a] startSpeculativeClassifierCheck("git log --oneline -5")
   │ → 异步启动分类器（不阻塞）
   │
7. [Step 5] runPreToolUseHooks()
   │ → 检查用户配置的 hooks
   │ → 无匹配 hook → pass
   │
8. [Step 6] resolveHookPermissionDecision()
   │ → hookPermissionResult: undefined (无 hook 决策)
   │ → tool.checkPermissions() → bashToolHasPermission()
   │   → mode=default → checkReadOnly → is read-only → allow
   │
9. [Step 7] tool.call()
   │ → runShellCommand({ command: "git log --oneline -5" })
   │ → spawn 子进程
   │ → < 2s 完成，无 progress event
   │ → trackGitOperations("git log...", 0, stdout)
   │ → return { stdout: "abc1234 feat: ...\n...", stderr: "", interrupted: false }
   │
10. [Step 8] runPostToolUseHooks()
   │ → 遥测记录
   │
11. [Step 9] mapToolResultToToolResultBlockParam()
    │ → 标准文本: "abc1234 feat: ...\n..."
    │ → tool_result message → 回传给模型
```

---

## 十二、Tool 特殊逻辑对比表

| 维度 | BashTool | FileEditTool | AgentTool | TodoWriteTool | ToolSearchTool | FileReadTool |
|------|----------|-------------|-----------|---------------|----------------|-------------|
| **isConcurrencySafe** | `isReadOnly(input)` | `false` (总是写) | `false` | `false` | `true` | `true` |
| **isReadOnly** | AST 分析命令 | `false` | `false` | `false` | `true` | `true` |
| **shouldDefer** | `false` | `false` | 条件 | `true` | `false` (锚点) | `false` |
| **checkPermissions** | 多层链（最复杂） | 路径+stale write检查 | Agent 类型检查 | 默认 allow | 默认 allow | 默认 allow |
| **validateInput** | sleep 检测 | old_string 唯一性 | Agent 定义校验 | todo 格式检查 | - | 路径检查 |
| **userFacingName** | 动态(Bash/Edit/Sandboxed) | `Edit(file.ts)` | `Agent(type)` | `TodoWrite` | `ToolSearch` | `Read(file.ts)` |
| **maxResultSizeChars** | 30,000 | 30,000 | Infinity | 30,000 | 30,000 | Infinity |
| **preparePermissionMatcher** | AST 分词 + wildcard | 路径匹配 | - | - | - | - |
| **contextModifier** | CWD 变更 | readFileState 更新 | Agent 状态 | - | 发现 tool | readFileState |
| **特殊能力** | 后台任务、沙箱、图片 | stale write 保护 | 递归 agent | 无 UI 渲染 | tool_reference blocks | PDF/图片读取 |

---

## 十三、设计决策总结

### 13.1 为什么 Tool 而非 Function Call？

> [设计决策] Claude Code 的 Tool 概念比 OpenAI 的 Function Call 更丰富。
> 每个 Tool 不仅有 `call()`，还有完整的权限、UI、验证、并发控制生命周期。
> 这是因为 CLI 场景下，tool 的执行有真实的副作用（文件修改、命令执行），
> 需要比 API 层面的 function_call 更细致的控制。

### 13.2 Fail-Closed vs Fail-Open

> [设计决策] 所有默认值都选择了 fail-closed：
> - `isConcurrencySafe` → `false` (不并发)
> - `isReadOnly` → `false` (当作写操作)
> - `checkPermissions` → `allow` (但这只是默认，实际用 hook 链)
>
> 新 tool 忘记实现某个方法时，宁可慢（串行执行）、宁可问（权限检查），也不能出安全问题。

### 13.3 分区排序的 Cache 守护

> [设计决策] `assembleToolPool()` 的分区排序看似是小事，但直接影响了
> 生产环境中 **所有用户** 的 prompt cache 命中率。
> 一个随意的 `tools.sort()` 就能让 cache hit rate 从 90% 掉到 0%。
> 这是"基础设施级别的细节"。

### 13.4 Speculative Classifier 的并行策略

> [设计决策] 在 Step 4 而非 Step 6 启动分类器，因为分类器是网络调用（~1-2s）。
> 让它与 PreToolUse hooks 并行执行，用户感知的等待时间减少了近一半。
> 但 UI 指示器不在这里设置（避免闪烁），而是在 interactiveHandler 中延迟设置。

---

## 十四、常见误区

**误区 1: "Tool 越多越好"**

错。每个 tool 在 context window 中占据 token。20 个 tool 大约占 5-8K token。
当 MCP 工具达到 100+ 个时，tool 定义本身就占了 10% 以上的 context。
这就是 ToolSearch defer 机制存在的原因。

**误区 2: "isReadOnly 和 isConcurrencySafe 是一回事"**

不是。`cd` 命令 isReadOnly=false（修改 CWD），但它不会产生文件副作用。
BashTool 的实现中，`isConcurrencySafe` 直接委托给 `isReadOnly`，
但其他 tool 可能有不同的逻辑（例如一个 tool 可能是只读但有全局锁）。

**误区 3: "Hook 返回 allow 就一定 allow"**

不一定。`resolveHookPermissionDecision` 中，hook 的 allow 只是建议。
如果 tool 自己的 `checkPermissions` 返回 deny，hook 的 allow 不会覆盖它。
优先级是：hook deny > hook allow > tool checkPermissions > canUseTool。

**误区 4: "MCP tool 和 built-in tool 等价"**

不等价。MCP tool 默认 defer，名字冲突时 built-in 优先，
而且 MCP tool 的权限规则可以用 `mcp__serverName` 前缀整体 deny 一个 server 的所有 tools。

---

## 十五、跨模块联系

```
┌──────────────┐     tool 定义     ┌──────────────┐
│ System Prompt │◄────────────────│ Tool Pool    │
│ (Part 3)     │  prompt() 方法   │ (本篇)       │
└──────────────┘                  └───────┬──────┘
                                          │
┌──────────────┐   tool_use blocks ┌──────┴──────┐
│ Conversation │──────────────────►│ Orchestrator│
│ Engine       │◄──────────────────│ (本篇)      │
│ (Part 1)     │   tool_result    └─────────────┘
└──────┬───────┘                          │
       │                                  │
       ▼                          ┌───────▼──────┐
┌──────────────┐                  │ Permission   │
│ Context      │                  │ System       │
│ Lifecycle    │                  │ (Part 6)     │
│ (Part 4)     │                  └──────────────┘
└──────────────┘
```

- **与 Conversation Engine (Part 1)**：Tool 系统接收模型的 tool_use，返回 tool_result
- **与 System Prompt (Part 3)**：每个 tool 的 `prompt()` 方法贡献 system prompt 内容
- **与 Context Lifecycle (Part 4)**：tool 结果占 context 空间，大结果会被持久化
- **与 Permission System (Part 6)**：每次 tool 调用都经过权限链
- **与 MCP (Part 7)**：MCP tool 的 defer/load 策略与 ToolSearch 紧密耦合

---

## 十六、Takeaway

1. **Tool = Claude 的唯一 I/O 通道**。理解 tool 系统就理解了 Claude Code 的一半。

2. **分层防御是安全模型的核心**。从 schema validation 到 hook chain 到 classifier，
   每一层都独立工作，任何一层的失败都不会导致安全漏洞。

3. **并发控制建立在 "保守默认" 之上**。新 tool 默认串行、默认非只读，
   只有显式声明了 `isConcurrencySafe: true` 才会被并行执行。

4. **Defer 机制是 scalability 的关键**。没有 ToolSearch，
   100 个 MCP tool 会让 context window 爆炸。Defer 把 O(N) 的固定成本变成了 O(1)+按需。

5. **Cache 稳定性是隐性的但关键的约束**。assembleToolPool 的分区排序、
   getAllBaseTools 与 Statsig 的同步要求，都是为了 prompt cache 这一个目标。

---

## 十七、思考题

1. **如果你需要添加一个新的内置 Tool（比如 DatabaseQueryTool），
   需要修改哪些文件？需要注意哪些 cache/permission/concurrency 约束？**

2. **BashTool 的 `_simulatedSedEdit` 双重防御中，如果我们去掉 schema 层的 omit，
   只保留运行时 stripping，会有什么安全风险？**

3. **当前的并发模型是 "连续的 concurrencySafe 合并为并行 batch"。
   如果改成 "所有 concurrencySafe 不管位置都并行"，会有什么问题？**

   提示：考虑 `[Grep, FileEdit, Grep]` 的场景。

4. **ToolSearch 的 defer 机制对 subagent 有什么影响？
   Coordinator 的 4 个 allowed tools 中为什么没有 ToolSearch？**

5. **如果一个 MCP server 的 tool 设置了 `alwaysLoad: true`，
   会对其他用户的 prompt cache 产生什么影响？如何缓解？**

---

## 十八、关键源文件索引

| 文件 | 职责 | 重要度 |
|------|------|--------|
| `src/Tool.ts` | Tool interface 定义、buildTool | ★★★★★ |
| `src/tools.ts` | getAllBaseTools、assembleToolPool、getTools | ★★★★★ |
| `src/services/tools/toolExecution.ts` | checkPermissionsAndCallTool 9 步流水线 | ★★★★★ |
| `src/services/tools/toolOrchestration.ts` | partitionToolCalls、并发/串行调度 | ★★★★★ |
| `src/tools/BashTool/BashTool.tsx` | BashTool 完整实现 | ★★★★★ |
| `src/tools/BashTool/bashPermissions.ts` | Bash 多层权限链、speculative classifier | ★★★★☆ |
| `src/utils/toolSearch.ts` | ToolSearch 启用逻辑、defer 判断、阈值计算 | ★★★★☆ |
| `src/tools/ToolSearchTool/prompt.ts` | isDeferredTool()、formatDeferredToolLine | ★★★★☆ |
| `src/utils/toolPool.ts` | mergeAndFilterTools、coordinator filter | ★★★☆☆ |
| `src/constants/tools.ts` | Agent 隔离常量定义 | ★★★☆☆ |
| `src/tools/AgentTool/agentToolUtils.ts` | filterToolsForAgent | ★★★☆☆ |
| `src/services/tools/toolHooks.ts` | PreToolUse/PostToolUse hook 执行 | ★★★☆☆ |
| `src/tools/FileEditTool/FileEditTool.ts` | 文件编辑 tool（stale write 保护） | ★★☆☆☆ |
| `src/tools/TodoWriteTool/TodoWriteTool.ts` | 任务管理 tool（无 UI 渲染） | ★★☆☆☆ |

---

> 下一篇（Part 6）我们将深入 Permission System，探讨 Claude Code 如何在
> "给模型足够的自由" 和 "保护用户的安全" 之间取得平衡。
