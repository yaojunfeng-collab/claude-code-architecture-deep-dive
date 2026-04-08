# Part 6: Agent 架构 -- 什么时候该开子 Agent？Fork 模式的 Cache 魔法

> **系列**: Claude Code 2.1.88 Architecture Deep Dive (6/10)
> **前置阅读**: Part 1 (Conversation Engine), Part 4 (Context Lifecycle)
> **关键源文件**: `src/tools/AgentTool/AgentTool.tsx`, `src/tools/AgentTool/runAgent.ts`, `src/tools/AgentTool/forkSubagent.ts`, `src/tools/AgentTool/builtInAgents.ts`, `src/tools/AgentTool/loadAgentsDir.ts`

---

## 0. 开场: 一个反直觉的设计

很多 AI Agent 框架（AutoGPT、CrewAI）的核心理念是 "自动分解任务、并行 dispatch 给多个 agent"。Claude Code 恰好相反:

> **默认行为是自己干完，不要轻易开 subagent。**

这不是懒，是精心考量的架构决策。每开一个 subagent 意味着:
- 一次全新的 API 调用（system prompt + context 从零构建）
- 额外的 token 消耗（subagent 没有父级上下文）
- 结果需要序列化回主线程（可能丢失 nuance）
- 增加了用户等待时间

但到了 2.1.88 版本，Claude Code 引入了一个颠覆性的新模式 -- **Fork Subagent**，让"开子 agent"的成本接近于零。这就是本文要深入探讨的 cache 魔法。

---

## 1. 全景: Agent 系统架构图

```
┌──────────────────────────────────────────────────────────────┐
│                     User Input (REPL)                        │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                    Main Agent (query loop)                    │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐ │
│  │ Bash Tool   │  │ Read Tool   │  │ Agent Tool (dispatch) │ │
│  └─────────────┘  └─────────────┘  └──────────┬───────────┘ │
└──────────────────────────────────────────────────────────────┘
                                                │
              ┌────────────────┬────────────────┼────────────────┐
              │                │                │                │
              ▼                ▼                ▼                ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────┐
     │ general-     │ │  Explore     │ │  Plan        │ │   Fork     │
     │ purpose      │ │  (haiku)     │ │  (inherit)   │ │  (inherit) │
     │              │ │  read-only   │ │  read-only   │ │  full ctx  │
     │ tools: [*]   │ │  omitClaudeMd│ │  omitClaudeMd│ │  tools: [*]│
     └──────────────┘ └──────────────┘ └──────────────┘ └────────────┘
              │                │                │                │
              ▼                ▼                ▼                ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────┐
     │ verification │ │ claude-code- │ │ statusline-  │ │ Custom     │
     │  (inherit)   │ │ guide(haiku) │ │ setup(sonnet)│ │ (.claude/  │
     │  background  │ │ dontAsk perm │ │ tools:[R,E]  │ │  agents/)  │
     │  adversarial │ │ limited tools│ │              │ │            │
     └──────────────┘ └──────────────┘ └──────────────┘ └────────────┘
```

[核心概念] AgentTool 是一个**统一的 dispatch 入口**，所有 subagent 调用都走同一个 tool。模型不是被框架"安排"去调用 subagent，而是**自主决定**是否调用以及选择哪种 subagent type。

---

## 2. 核心哲学: 模型自主决策，框架不强制

### 2.1 "When NOT to use" -- prompt 里的反模式教育

在 `src/tools/AgentTool/prompt.ts` 中，AgentTool 的 prompt 专门有一节告诉模型**不该用 Agent**的场景:

```
When NOT to use the Agent tool:
- If you want to read a specific file path, use the Read tool or the Glob tool
  instead of the Agent tool, to find the match more quickly
- If you are searching for a specific class definition like "class Foo", use
  the Glob tool instead, to find the match more quickly
- If you are searching for code within a specific file or set of 2-3 files,
  use the Read tool instead of the Agent tool, to find the match more quickly
```

[设计决策] 这是一种"反用户增长"的 prompt 设计。大多数框架希望 agent 被更多使用，而 Claude Code 明确告诉模型: **你自己能干的，就别开 subagent**。原因:
1. 直接工具调用只需 1 个 turn，subagent 至少需要 3-5 个 turn
2. 结果直接在主上下文，不需要序列化/反序列化
3. 避免不必要的 token 开销

### 2.2 Fork 模式: "不该开 subagent" 的例外

当 Fork Subagent 实验开启时（`isForkSubagentEnabled()`），prompt 风格完全变了。"When NOT to use" 整节被删除，替换为:

```typescript
// prompt.ts
const whenNotToUseSection = forkEnabled
  ? ''  // Fork 模式下完全不显示 "When NOT to use"
  : `When NOT to use the ${AGENT_TOOL_NAME} tool: ...`
```

取而代之的是 "When to fork" 指导:

```
Fork yourself (omit subagent_type) when the intermediate tool output isn't
worth keeping in your context. The criterion is qualitative -- "will I need
this output again" -- not task size.
```

[权衡] Fork 模式的核心判断标准不是"任务大不大"，而是 **"中间产物有没有必要留在主上下文"**。这是一个语义层面的判断，而非机械性的规则。

---

## 3. 六大内置 Agent 详解

### 3.1 数据结构: BuiltInAgentDefinition

在 `src/tools/AgentTool/loadAgentsDir.ts` 中定义了 Agent 的完整类型:

```typescript
export type BaseAgentDefinition = {
  agentType: string              // 唯一标识符，如 'Explore', 'Plan'
  whenToUse: string              // 模型用来决定是否选择此 agent 的描述
  tools?: string[]               // 允许的工具列表 (undefined = 全部, ['*'] = 全部)
  disallowedTools?: string[]     // 禁止的工具列表 (从 tools 中排除)
  skills?: string[]              // 预加载的 skill 名称
  mcpServers?: AgentMcpServerSpec[] // agent 专属 MCP 服务器
  hooks?: HooksSettings          // session-scoped hooks
  color?: AgentColorName         // UI 显示颜色
  model?: string                 // 模型选择 ('inherit' | 'haiku' | 'sonnet' | 具体模型ID)
  effort?: EffortValue           // 推理努力级别
  permissionMode?: PermissionMode // 权限模式 ('bubble' | 'dontAsk' | 等)
  maxTurns?: number              // 最大轮次限制
  background?: boolean           // 是否强制后台运行
  omitClaudeMd?: boolean         // 是否省略 CLAUDE.md
  isolation?: 'worktree' | 'remote'  // 隔离模式
  memory?: AgentMemoryScope      // 持久化记忆范围
  // ... 更多字段
}

export type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  baseDir: 'built-in'
  callback?: () => void
  getSystemPrompt: (params: {
    toolUseContext: Pick<ToolUseContext, 'options'>
  }) => string  // 动态生成 system prompt
}
```

[核心概念] 注意 `getSystemPrompt` 是一个**函数**，不是静态字符串。这允许 agent 根据运行时上下文动态调整 prompt（如 claude-code-guide 会注入当前配置的 MCP、plugins 等信息）。

### 3.2 六大 Agent 对比表

| Agent | model | tools | permissionMode | background | omitClaudeMd | 核心痛点 |
|-------|-------|-------|---------------|------------|-------------|---------|
| **general-purpose** | 默认subagent模型 | `['*']` (全部) | 默认 | false | false | 通用任务委派 |
| **Explore** | haiku(ext)/inherit(ant) | 除写入工具外全部 | 默认 | false | **true** | 噪音隔离: 搜索结果不污染主上下文 |
| **Plan** | inherit | 同 Explore | 默认 | false | **true** | 只读安全: 防止规划时误改文件 |
| **verification** | inherit | 除写入工具外全部 | 默认 | **true** | false | 对抗性测试: 独立验证实现正确性 |
| **claude-code-guide** | haiku | 仅 Glob/Grep/Read/WebFetch/WebSearch | **dontAsk** | false | false | 文档查询: 用小模型快速回答 |
| **statusline-setup** | sonnet | 仅 `['Read', 'Edit']` | 默认 | false | false | 专项配置: 最小权限原则 |

### 3.3 各 Agent 深入解析

#### (1) general-purpose -- 万金油

```typescript
// src/tools/AgentTool/built-in/generalPurposeAgent.ts
export const GENERAL_PURPOSE_AGENT: BuiltInAgentDefinition = {
  agentType: 'general-purpose',
  whenToUse: 'General-purpose agent for researching complex questions, '
    + 'searching for code, and executing multi-step tasks...',
  tools: ['*'],          // 全部工具
  source: 'built-in',
  baseDir: 'built-in',
  // model 故意省略 -- 使用 getDefaultSubagentModel()
  getSystemPrompt: getGeneralPurposeSystemPrompt,
}
```

[设计决策] 为什么 `model` 留空？因为 subagent 默认模型由 `getAgentModel()` 决定，通常是比主模型便宜的版本。`tools: ['*']` 表示拥有与主 agent 相同的完整工具集。

#### (2) Explore -- 快速搜索专家

```typescript
// src/tools/AgentTool/built-in/exploreAgent.ts
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  disallowedTools: [
    AGENT_TOOL_NAME,          // 不能嵌套调用 Agent
    EXIT_PLAN_MODE_TOOL_NAME, // 不需要退出 plan mode
    FILE_EDIT_TOOL_NAME,      // 不能编辑文件
    FILE_WRITE_TOOL_NAME,     // 不能写入文件
    NOTEBOOK_EDIT_TOOL_NAME,  // 不能编辑 notebook
  ],
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  omitClaudeMd: true,  // 关键: 省略 CLAUDE.md 节省 token
  getSystemPrompt: () => getExploreSystemPrompt(),
}
```

[可借鉴] `omitClaudeMd: true` 背后的数据: Explore agent 每周被调用 3400 万次以上，每次省略 CLAUDE.md 可以节省 ~5-15 Gtok/week。这是典型的"高频小优化，聚沙成塔"。

Explore 的 prompt 还包含三级 thoroughness:
```
When calling this agent, specify the desired thoroughness level:
"quick" for basic searches,
"medium" for moderate exploration, or
"very thorough" for comprehensive analysis
```

最低调用次数保证: `EXPLORE_AGENT_MIN_QUERIES = 3`。

#### (3) Plan -- 只读架构师

```typescript
// src/tools/AgentTool/built-in/planAgent.ts
export const PLAN_AGENT: BuiltInAgentDefinition = {
  agentType: 'Plan',
  disallowedTools: [/* 同 Explore: 禁止所有写入工具 */],
  tools: EXPLORE_AGENT.tools,  // 直接复用 Explore 的工具配置
  model: 'inherit',            // 继承父级模型
  omitClaudeMd: true,
  getSystemPrompt: () => getPlanV2SystemPrompt(),
}
```

[设计决策] Plan 的 `model: 'inherit'` 和 Explore 的 `haiku` 不同。原因: 规划需要更强的推理能力，Haiku 可能不够好；而搜索更重视速度。

Plan 的输出格式要求:
```
End your response with:
### Critical Files for Implementation
List 3-5 files most critical for implementing this plan
```

#### (4) verification -- 对抗性验证者

```typescript
// src/tools/AgentTool/built-in/verificationAgent.ts
export const VERIFICATION_AGENT: BuiltInAgentDefinition = {
  agentType: 'verification',
  color: 'red',          // UI 中以红色标识 -- 视觉警示
  background: true,       // 始终后台运行
  disallowedTools: [/* 同 Explore */],
  model: 'inherit',
  getSystemPrompt: () => VERIFICATION_SYSTEM_PROMPT,
  criticalSystemReminder_EXPERIMENTAL:
    'CRITICAL: This is a VERIFICATION-ONLY task. You CANNOT edit...',
}
```

[核心概念] verification agent 的 prompt 是所有 agent 中最长的（约 150 行），包含:
- **两个已知失败模式**: "verification avoidance" 和 "被前80%迷惑"
- **对抗性探测**: 并发、边界值、幂等性、孤儿操作
- **强制输出格式**: 每个 check 必须包含 Command run + Output observed
- **最终 VERDICT**: 必须输出 `VERDICT: PASS | FAIL | PARTIAL`

写入限制: 只能写 `/tmp` 或 `$TMPDIR`，不能修改项目文件。

`criticalSystemReminder_EXPERIMENTAL` 是一个特殊字段，会在**每个 user turn**注入，防止模型在长对话中"忘记"自己的角色约束。

#### (5) claude-code-guide -- 文档导航员

```typescript
// src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts
export const CLAUDE_CODE_GUIDE_AGENT: BuiltInAgentDefinition = {
  agentType: 'claude-code-guide',
  tools: [GLOB_TOOL_NAME, GREP_TOOL_NAME, FILE_READ_TOOL_NAME,
          WEB_FETCH_TOOL_NAME, WEB_SEARCH_TOOL_NAME],
  model: 'haiku',
  permissionMode: 'dontAsk',  // 不询问权限 -- 只有读取操作
  getSystemPrompt({ toolUseContext }) {
    // 动态注入: 当前 MCP 服务器、自定义 agents、plugins、settings
    // 使得 guide agent 能准确回答 "我配置了什么" 类问题
  },
}
```

[设计决策] `permissionMode: 'dontAsk'` -- 因为 guide agent 只有读取类工具，不会产生任何破坏性操作，所以跳过所有权限确认。这大幅减少了用户交互负担。

注意: 只在非 SDK 入口下加载:
```typescript
// builtInAgents.ts
const isNonSdkEntrypoint =
  process.env.CLAUDE_CODE_ENTRYPOINT !== 'sdk-ts' &&
  process.env.CLAUDE_CODE_ENTRYPOINT !== 'sdk-py' &&
  process.env.CLAUDE_CODE_ENTRYPOINT !== 'sdk-cli'
if (isNonSdkEntrypoint) {
  agents.push(CLAUDE_CODE_GUIDE_AGENT)
}
```

#### (6) statusline-setup -- 最小权限专家

```typescript
// src/tools/AgentTool/built-in/statuslineSetup.ts
export const STATUSLINE_SETUP_AGENT: BuiltInAgentDefinition = {
  agentType: 'statusline-setup',
  tools: ['Read', 'Edit'],  // 只有两个工具!
  model: 'sonnet',
  color: 'orange',
  getSystemPrompt: () => STATUSLINE_SYSTEM_PROMPT,
}
```

[可借鉴] 这是**最小权限原则**的完美示范。statusline 配置只需要读取 shell rc 文件 + 编辑 settings.json，所以只给了 Read 和 Edit。

---

## 4. 完整 Subagent 调用链 (8 步)

以下是模型调用 `Agent(...)` 后发生的完整流程:

### Step 1: Agent 选择与验证 (AgentTool.tsx:239-410)

```typescript
// AgentTool.tsx, call() 方法入口
async call({
  prompt, subagent_type, description, model: modelParam,
  run_in_background, name, team_name, mode: spawnMode,
  isolation, cwd
}: AgentToolInput, toolUseContext, canUseTool, assistantMessage, onProgress?)
```

**选择逻辑 (三路分支)**:

```
                subagent_type 参数
                      │
            ┌─────────┼──────────┐
            │         │          │
         已设置    未设置       未设置
            │    fork 开启    fork 关闭
            │         │          │
            ▼         ▼          ▼
     查找匹配的   FORK_AGENT   默认使用
     AgentDef               general-purpose
```

```typescript
// AgentTool.tsx:322-356
const effectiveType = subagent_type
  ?? (isForkSubagentEnabled() ? undefined : GENERAL_PURPOSE_AGENT.agentType);
const isForkPath = effectiveType === undefined;

if (isForkPath) {
  // 递归 fork 防护: 检测 querySource 或 message 中的 fork-boilerplate 标签
  if (toolUseContext.options.querySource === `agent:builtin:fork`
      || isInForkChild(toolUseContext.messages)) {
    throw new Error('Fork is not available inside a forked worker...');
  }
  selectedAgent = FORK_AGENT;
} else {
  // 正常路径: 从 activeAgents 中查找
  const agents = filterDeniedAgents(allAgents, ...);
  const found = agents.find(agent => agent.agentType === effectiveType);
  if (!found) throw new Error(`Agent type '${effectiveType}' not found...`);
  selectedAgent = found;
}
```

**嵌套约束**:
- Fork child 不能再 fork（`isInForkChild()` 检测 `<fork-boilerplate>` 标签）
- Teammate 不能 spawn 其他 teammate（扁平化团队结构）
- In-process teammate 不能 spawn background agent

**MCP 可用性轮询**:
```typescript
// AgentTool.tsx:370-410
// 如果 required MCP servers 还在 pending，最多等 30 秒
if (hasPendingRequiredServers) {
  const MAX_WAIT_MS = 30_000;
  const POLL_INTERVAL_MS = 500;
  while (Date.now() < deadline) {
    await sleep(POLL_INTERVAL_MS);
    // 检查是否已连接或失败
    if (hasFailedRequiredServer) break;
    if (!stillPending) break;
  }
}
```

### Step 2: System Prompt 与 Message 构建

**Fork vs Normal 两条路径**:

| 维度 | Fork Path | Normal Path |
|------|-----------|-------------|
| System Prompt | **复用父级** (`toolUseContext.renderedSystemPrompt`) | 调用 `selectedAgent.getSystemPrompt()` + `enhanceSystemPromptWithEnvDetails()` |
| Prompt Messages | `buildForkedMessages(prompt, assistantMessage)` | `createUserMessage({ content: prompt })` |
| 上下文继承 | 完整父级对话历史 | 空白开始 |
| Claude.md | 继承父级（已在 system prompt 中） | 按 `omitClaudeMd` 决定 |

```typescript
// AgentTool.tsx:495-541
if (isForkPath) {
  // Fork: 直接复用父级渲染好的 system prompt bytes
  if (toolUseContext.renderedSystemPrompt) {
    forkParentSystemPrompt = toolUseContext.renderedSystemPrompt;
  }
  // 构建 forked messages: 父级 assistant msg + placeholder results + directive
  promptMessages = buildForkedMessages(prompt, assistantMessage);
} else {
  // Normal: 构建独立的 system prompt
  const agentPrompt = selectedAgent.getSystemPrompt({ toolUseContext });
  enhancedSystemPrompt = await enhanceSystemPromptWithEnvDetails(
    [agentPrompt], resolvedAgentModel, additionalWorkingDirectories
  );
  promptMessages = [createUserMessage({ content: prompt })];
}
```

### Step 3: 工具池组装

```typescript
// AgentTool.tsx:571-577
// Worker 的工具池独立于父级! 用自己的 permissionMode 组装
const workerPermissionContext = {
  ...appState.toolPermissionContext,
  mode: selectedAgent.permissionMode ?? 'acceptEdits'
};
const workerTools = assembleToolPool(workerPermissionContext, appState.mcp.tools);
```

[核心概念] 每个 subagent 的工具池是**独立组装**的，不继承父级的工具限制。这意味着:
- 父级如果在 plan mode（只有只读工具），子 agent 仍然可以有写入工具
- 但如果子 agent 定义了 `disallowedTools`，则会从池中移除

特例: Fork path 使用 `useExactTools: true`，直接复用父级的**精确工具对象引用**（不是重新组装），这是 cache 优化的关键。

### Step 4: 同步/异步分支

```typescript
// AgentTool.tsx:556-567
const forceAsync = isForkSubagentEnabled();  // Fork 模式下强制全部异步
const shouldRunAsync =
  (run_in_background === true            // 显式请求后台
   || selectedAgent.background === true   // Agent 定义强制后台 (verification)
   || isCoordinator                       // Coordinator 模式
   || forceAsync                          // Fork 实验
   || assistantForceAsync                 // Kairos 模式
   || proactiveModule?.isProactiveActive()  // Proactive 模式
  ) && !isBackgroundTasksDisabled;
```

**4a: 异步路径**:
```typescript
// AgentTool.tsx:686-764
if (shouldRunAsync) {
  const agentBackgroundTask = registerAsyncAgent({
    agentId: asyncAgentId,
    description, prompt, selectedAgent,
    setAppState: rootSetAppState,
    toolUseId: toolUseContext.toolUseId
  });

  // Fire-and-forget! 不等待完成
  void runWithAgentContext(asyncAgentContext, () =>
    wrapWithCwd(() => runAsyncAgentLifecycle({
      taskId: agentBackgroundTask.agentId,
      abortController: agentBackgroundTask.abortController!,
      makeStream: onCacheSafeParams => runAgent({...}),
      // ...
    }))
  );

  // 立即返回: 只给 output_file 路径
  return {
    data: {
      status: 'async_launched',
      agentId: agentBackgroundTask.agentId,
      outputFile: getTaskOutputPath(agentBackgroundTask.agentId),
      canReadOutputFile
    }
  };
}
```

**4b: 同步路径** -- 包含 backgrounding 信号竞争:
```typescript
// AgentTool.tsx:865-900
while (true) {
  const nextMessagePromise = agentIterator.next();
  // Race: 下一条消息 vs 用户触发的 background 信号
  const raceResult = backgroundPromise
    ? await Promise.race([
        nextMessagePromise.then(r => ({ type: 'message', result: r })),
        backgroundPromise  // → { type: 'background' }
      ])
    : { type: 'message', result: await nextMessagePromise };

  if (raceResult.type === 'background' && foregroundTaskId) {
    // 用户按了 Ctrl+Z 或 auto-background 触发
    wasBackgrounded = true;
    // 关闭前台 iterator，用 runAgent 重新启动为后台
    await agentIterator.return(undefined);
    void runWithAgentContext(syncAgentContext, async () => {
      for await (const msg of runAgent({...isAsync: true...})) {
        // 后台继续运行...
      }
    });
    break; // 跳出同步循环
  }
}
```

### Step 5: 运行时初始化 (runAgent.ts)

`runAgent()` 是一个 `AsyncGenerator`，核心初始化步骤:

```typescript
// runAgent.ts:248-746 (精简)
export async function* runAgent({...}): AsyncGenerator<Message, void> {
  // 1. 解析模型
  const resolvedAgentModel = getAgentModel(
    agentDefinition.model, toolUseContext.options.mainLoopModel, model, permissionMode
  );

  // 2. 创建 AgentId
  const agentId = override?.agentId ?? createAgentId();

  // 3. Context messages 处理 (fork 路径会带入父级历史)
  const contextMessages = forkContextMessages
    ? filterIncompleteToolCalls(forkContextMessages) : [];

  // 4. 构建 UserContext & SystemContext
  // omitClaudeMd: Explore/Plan 省略 CLAUDE.md
  const shouldOmitClaudeMd = agentDefinition.omitClaudeMd && ...;
  // 省略 gitStatus: Explore/Plan 不需要 stale 的 git 信息
  const resolvedSystemContext = (type === 'Explore' || type === 'Plan')
    ? systemContextNoGit : baseSystemContext;

  // 5. Permission mode 覆写
  // agent 定义的 permissionMode 不会覆盖 bypassPermissions/acceptEdits/auto

  // 6. 工具解析
  const resolvedTools = useExactTools
    ? availableTools  // Fork: 直接用父级工具
    : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools;

  // 7. System prompt
  const agentSystemPrompt = override?.systemPrompt
    ?? asSystemPrompt(await getAgentSystemPrompt(...));

  // 8. SubagentStart hooks
  for await (const hookResult of executeSubagentStartHooks(...)) { ... }

  // 9. Frontmatter hooks 注册
  if (agentDefinition.hooks) registerFrontmatterHooks(...);

  // 10. Skill 预加载
  for (const skill of skillsToPreload) { ... }

  // 11. Agent MCP 初始化
  const { clients, tools: agentMcpTools, cleanup } =
    await initializeAgentMcpServers(agentDefinition, parentClients);

  // 12. 构建 agent options (thinkingConfig, tools, etc.)
  const agentOptions = {
    thinkingConfig: useExactTools
      ? toolUseContext.options.thinkingConfig  // Fork: 继承
      : { type: 'disabled' },                 // Normal: 禁用 (省 token)
    // ...
  };

  // 13. 创建 subagent context
  const agentToolUseContext = createSubagentContext(toolUseContext, {...});
}
```

### Step 6: 对话循环

```typescript
// runAgent.ts:748-806
try {
  for await (const message of query({
    messages: initialMessages,
    systemPrompt: agentSystemPrompt,
    userContext: resolvedUserContext,
    systemContext: resolvedSystemContext,
    canUseTool,
    toolUseContext: agentToolUseContext,
    querySource,
    maxTurns: maxTurns ?? agentDefinition.maxTurns,
  })) {
    // 转发 API metrics (TTFT/OTPS)
    if (message.type === 'stream_event' && message.event.type === 'message_start') {
      toolUseContext.pushApiMetricsEntry?.(message.ttftMs);
      continue;
    }
    // 处理 max_turns 到达
    if (message.type === 'attachment' && message.attachment.type === 'max_turns_reached') {
      break;
    }
    // 记录 sidechain transcript
    if (isRecordableMessage(message)) {
      await recordSidechainTranscript([message], agentId, lastRecordedUuid);
      yield message;
    }
  }
}
```

### Step 7: 结果返回

同步路径使用 `finalizeAgentTool()` 提取最终结果:
```typescript
// agentToolUtils.ts (简化)
function finalizeAgentTool(agentMessages, taskId, metadata) {
  const lastAssistant = getLastAssistantMessage(agentMessages);
  return {
    content: lastAssistant?.message.content ?? [],
    totalToolUseCount,
    totalDurationMs: Date.now() - metadata.startTime,
    // ...
  };
}
```

异步路径通过 notification 通知主线程:
```typescript
enqueueAgentNotification({
  taskId, description, status: 'completed',
  setAppState: rootSetAppState,
  finalMessage,
  usage: { totalTokens, toolUses, durationMs },
  toolUseId: toolUseContext.toolUseId,
});
```

### Step 8: 清理 (finally block)

`runAgent()` 的 `finally` 块执行 **9 项清理操作**:

```typescript
// runAgent.ts:816-858
finally {
  // 1. 清理 agent 专属 MCP 服务器连接
  await mcpCleanup()

  // 2. 清理 agent 的 session hooks
  if (agentDefinition.hooks) clearSessionHooks(rootSetAppState, agentId)

  // 3. 清理 prompt cache tracking
  if (feature('PROMPT_CACHE_BREAK_DETECTION')) cleanupAgentTracking(agentId)

  // 4. 释放克隆的 file state cache 内存
  agentToolUseContext.readFileState.clear()

  // 5. 释放 fork context messages 内存
  initialMessages.length = 0

  // 6. 注销 Perfetto tracing
  unregisterPerfettoAgent(agentId)

  // 7. 清除 transcript subdir mapping
  clearAgentTranscriptSubdir(agentId)

  // 8. 清理 agent 的 todos (防止长 session 中 AppState 泄漏)
  rootSetAppState(prev => {
    if (!(agentId in prev.todos)) return prev
    const { [agentId]: _removed, ...todos } = prev.todos
    return { ...prev, todos }
  })

  // 9. Kill agent 启动的后台 shell 任务 (防止 zombie 进程)
  killShellTasksForAgent(agentId, ...)
}
```

[可借鉴] 9 项清理操作几乎涵盖了所有可能的资源泄漏。注意第 5 项 `initialMessages.length = 0` -- 这是一个 JS 技巧，清空数组同时释放对消息对象的引用，让 GC 可以回收。

---

## 5. Fork 模式深入: Cache 魔法的四层优化

Fork 是 Claude Code 2.1.88 最重要的新特性之一。理解它需要先理解 Anthropic API 的 **prompt cache** 机制:

> API 会缓存请求前缀。如果两个请求的 system prompt + messages 的前缀完全相同(逐字节)，
> 后续请求可以复用缓存，只对差异部分收费。

### 5.1 Fork 的消息构建

```typescript
// forkSubagent.ts:107-169
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  // Part 1: 克隆完整的父级 assistant message (保留所有 tool_use blocks)
  const fullAssistantMessage = {
    ...assistantMessage,
    uuid: randomUUID(),
    message: {
      ...assistantMessage.message,
      content: [...assistantMessage.message.content],
    },
  };

  // Part 2: 收集所有 tool_use blocks
  const toolUseBlocks = assistantMessage.message.content.filter(
    block => block.type === 'tool_use'
  );

  // Part 3: 为每个 tool_use 生成 placeholder result (全部相同文本!)
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result',
    tool_use_id: block.id,
    content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }],
    //                                  ^^^^^^^^^^^^^^^^^^^^^^^^
    //           关键! 所有 fork 共享同一个 placeholder 字符串
  }));

  // Part 4: 组合: placeholder results + 每个 child 独有的 directive
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,
      { type: 'text', text: buildChildMessage(directive) },
      //                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      //           唯一的差异点: 每个 fork child 的 directive 不同
    ],
  });

  return [fullAssistantMessage, toolResultMessage];
}
```

最终的消息结构:

```
┌─────────────────────────────────────────────────────┐
│              发送给 API 的完整请求                    │
│                                                     │
│  [System Prompt]  ← 父级的 system prompt bytes       │  ← cache hit
│  [...父级历史 messages]                               │  ← cache hit
│  [assistant msg (所有 tool_use blocks)]               │  ← cache hit
│  [user msg:                                          │
│    tool_result #1: "Fork started — processing..."    │  ← cache hit
│    tool_result #2: "Fork started — processing..."    │  ← cache hit
│    tool_result #N: "Fork started — processing..."    │  ← cache hit
│    text: "<fork-boilerplate>...\n你的指令..."          │  ← 差异!
│  ]                                                   │
└─────────────────────────────────────────────────────┘
         ▲                                      ▲
         │                                      │
    所有 fork child                        唯一差异
    这部分完全相同                          (每个 child 不同)
```

### 5.2 FORK_PLACEHOLDER_RESULT -- 所有 fork 共享的"魔法字符串"

```typescript
// forkSubagent.ts:93
const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'
```

[核心概念] 这个字符串看似简单，实际上是整个 cache 优化的基石:
- 所有 fork child 的 tool_result 都使用**完全相同**的文本
- API 请求中 tool_result 出现在 message 的固定位置
- 这确保了所有 fork 的前缀字节**完全一致**，直到最后的 directive

如果每个 tool_result 包含实际的执行结果（如文件内容、搜索结果），那么每个 fork 的前缀都会不同，cache 完全失效。

### 5.3 四层 Cache 优化详解

```
┌─────────────────────────────────────────────────┐
│              四层 Cache 优化                      │
│                                                  │
│  Layer 1: override.systemPrompt                  │
│  ┌──────────────────────────────────────────────┐│
│  │ 复用父级已渲染的 system prompt bytes          ││
│  │ 不重新调用 getSystemPrompt() 生成             ││
│  │ 避免 GrowthBook cold→warm 状态差异           ││
│  │ → 保证逐字节一致                              ││
│  └──────────────────────────────────────────────┘│
│                                                  │
│  Layer 2: useExactTools = true                   │
│  ┌──────────────────────────────────────────────┐│
│  │ 直接使用父级的工具对象引用                     ││
│  │ 不通过 resolveAgentTools() 重新过滤           ││
│  │ 保证 tool definition 序列化完全一致           ││
│  │ → 避免 permissionMode 差异导致工具列表不同    ││
│  └──────────────────────────────────────────────┘│
│                                                  │
│  Layer 3: FORK_PLACEHOLDER_RESULT                │
│  ┌──────────────────────────────────────────────┐│
│  │ 所有 fork 的 tool_result 使用相同占位符       ││
│  │ "Fork started — processing in background"    ││
│  │ → 确保 message 序列前缀完全共享               ││
│  └──────────────────────────────────────────────┘│
│                                                  │
│  Layer 4: skipCacheWrite = true (query 层)       │
│  ┌──────────────────────────────────────────────┐│
│  │ Fork child 不写入自己的 cache                 ││
│  │ 避免污染主线程的 prompt cache                 ││
│  │ → 保证主线程 cache 不被子 agent 干扰          ││
│  └──────────────────────────────────────────────┘│
└─────────────────────────────────────────────────┘
```

**Layer 1 的源码佐证**:
```typescript
// forkSubagent.ts:55-58 (注释)
// The getSystemPrompt here is unused: the fork path passes
// `override.systemPrompt` with the parent's already-rendered system prompt
// bytes, threaded via `toolUseContext.renderedSystemPrompt`. Reconstructing
// by re-calling getSystemPrompt() can diverge (GrowthBook cold→warm) and
// bust the prompt cache; threading the rendered bytes is byte-exact.
```

**Layer 2 的源码佐证**:
```typescript
// AgentTool.tsx:611-633
override: isForkPath ? {
  systemPrompt: forkParentSystemPrompt
} : ...,
availableTools: isForkPath
  ? toolUseContext.options.tools  // 直接引用父级工具!
  : workerTools,                 // Normal: 独立组装
...(isForkPath && { useExactTools: true }),
```

**Layer 2 防止的问题**:
```typescript
// runAgent.ts:680-684
// For fork children (useExactTools), inherit thinking config to match the
// parent's API request prefix for prompt cache hits. For regular
// sub-agents, disable thinking to control output token costs.
thinkingConfig: useExactTools
  ? toolUseContext.options.thinkingConfig  // Fork: 继承 thinking config
  : { type: 'disabled' },                 // Normal: 禁用
```

### 5.4 Fork Child 防递归

Fork child 不能再 fork。检测机制有两层:

```typescript
// 第一层: querySource 检查 (compaction-resistant)
if (toolUseContext.options.querySource === `agent:builtin:${FORK_AGENT.agentType}`) {
  throw new Error('Fork is not available inside a forked worker...');
}

// 第二层: message-scan fallback (检测 <fork-boilerplate> 标签)
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false;
    const content = m.message.content;
    if (!Array.isArray(content)) return false;
    return content.some(
      block => block.type === 'text'
            && block.text.includes(`<${FORK_BOILERPLATE_TAG}>`),
    );
  });
}
```

[设计决策] 为什么两层？因为 autocompact 会重写 messages（破坏第二层检测），但不会修改 `context.options`（第一层不受影响）。两层互补确保不会遗漏。

### 5.5 Fork 的完整数据流

```
Main Agent                    Fork Child                   User
    │                            │                          │
    │ Agent({prompt: "..."})     │                          │
    │ (omit subagent_type)       │                          │
    │                            │                          │
    ├───registerAsyncAgent()─────▶                          │
    │                            │                          │
    ├───return {                 │                          │
    │     status: 'async_launched',                         │
    │     outputFile: path       │                          │
    │   }                        │                          │
    │                            │                          │
    │   (主线程继续工作)           │ ← runAgent() 开始       │
    │                            │    使用父级 cache         │
    │                            │                          │
    │                            │ (独立执行 tools...)       │
    │                            │                          │
    │                            │ 完成!                    │
    │                            │                          │
    │   <task-notification>      │                          │
    │ ◀─enqueueAgentNotification─┤                          │
    │                            │                          │
    │ (处理 notification)        │                          │
    │ 给用户总结结果 ──────────────────────────────────────▶│
    │                            │                          │
```

---

## 6. SkillTool Fork vs AgentTool Fork 对比

两者都有 "fork" 概念但设计完全不同:

| 维度 | AgentTool Fork | SkillTool Fork (processSlashCommand) |
|------|---------------|--------------------------------------|
| **历史继承** | 完整父级对话历史 | 无 (skill 是独立命令) |
| **System Prompt** | 复用父级 rendered bytes | Skill 自己的 prompt |
| **Placeholder** | `FORK_PLACEHOLDER_RESULT` (全局统一) | 无此机制 |
| **Cache 设计** | 四层优化，共享父级 cache | 独立 cache，不共享 |
| **同步/异步** | Fork 模式下全部强制异步 | 取决于 slash command |
| **结果返回** | `<task-notification>` (异步通知) | 直接在主对话中输出 |
| **Anti-recursion** | `<fork-boilerplate>` 检测 | 无此需求 |
| **适用场景** | "中间产物不需要留在上下文" | "执行一个预定义的技能" |

---

## 7. 与 Codex 设计哲学对比

| 维度 | Claude Code | Codex |
|------|------------|-------|
| **执行模式** | 主 agent 优先，subagent 按需 | Everything is agent loop |
| **上下文管理** | 主线程保持完整上下文，subagent 隔离 | 每个 agent 独立上下文 |
| **用户交互** | 主线程直接交互，subagent 结果汇总返回 | Agent 之间协调 |
| **Cache 优化** | Fork 四层 cache，共享前缀 | 独立 cache |
| **任务分解** | 模型自主决定，框架不强制 | 框架提供分解策略 |
| **默认行为** | "自己做完" | "开 agent" |

[设计决策] Claude Code 的 "主 agent 优先" 设计适合 CLI 交互场景: 用户期望看到连贯的上下文和进度，而不是多个 agent 各自为政。Fork 模式在保留这种连贯性的同时，解决了 "中间产物膨胀" 的问题。

---

## 8. 什么时候 Subagent 真的会被触发

根据 prompt 和代码分析，subagent 在以下场景下最可能被触发:

### 8.1 确定性触发 (模型高概率调用)

1. **不确定的搜索范围**: "这个变量在哪里定义的？" -- 可能需要多次搜索
2. **需要上下文隔离**: "搜索一下相关文件" -- 搜索结果不需要留在主上下文
3. **并行独立任务**: "同时搜索 A 和 B" -- 可以并行 fork
4. **专业化工具集**: 用 claude-code-guide 回答 Claude Code 使用问题
5. **Post-implementation 验证**: 用 verification agent 对抗性测试

### 8.2 Fork 模式特有触发条件

```
Fork yourself (omit subagent_type) when the intermediate tool output
isn't worth keeping in your context.
```

- **Research**: fork open-ended questions
- **Implementation**: fork 需要多次编辑的实现工作
- **判断标准**: "will I need this output again?"

### 8.3 不应触发的场景

- 读取已知路径的文件 → 直接 Read
- 搜索 2-3 个文件中的代码 → 直接 Grep/Read
- 搜索特定 class 定义 → 直接 Glob

---

## 9. 自定义 Agents: 三种方式

### 9.1 方式一: `.claude/agents/*.md` 文件

```markdown
---
name: my-reviewer
description: Code review agent that checks for common patterns
tools:
  - Read
  - Glob
  - Grep
model: inherit
permissionMode: dontAsk
background: true
maxTurns: 50
memory: project
hooks:
  SubagentStart:
    - command: "echo Starting review..."
---

You are a code review agent. Review the changes and report issues.
```

### 9.2 方式二: `settings.json` 中定义

```json
{
  "agents": {
    "my-linter": {
      "description": "Run linting checks",
      "prompt": "You are a linting agent...",
      "tools": ["Bash", "Read"],
      "model": "haiku",
      "permissionMode": "dontAsk"
    }
  }
}
```

### 9.3 方式三: Plugin

Plugin agents 通过 `loadPluginAgents()` 加载，具有 `source: 'plugin'` 标识。

### 9.4 优先级覆盖链

同名 agent 的覆盖优先级:

```
built-in → plugin → userSettings → projectSettings → flagSettings → policySettings
   1         2          3               4                5              6 (最高)
```

```typescript
// loadAgentsDir.ts:203-221
const agentGroups = [
  builtInAgents,    // 最低优先级
  pluginAgents,
  userAgents,
  projectAgents,
  flagAgents,
  managedAgents,    // 最高优先级 (policySettings)
]
const agentMap = new Map<string, AgentDefinition>()
for (const agents of agentGroups) {
  for (const agent of agents) {
    agentMap.set(agent.agentType, agent)  // 后面的覆盖前面的
  }
}
```

[可借鉴] 这种优先级链设计允许:
- 组织管理员通过 `policySettings` 强制覆盖任何 agent 行为
- 项目级配置覆盖用户级偏好
- 内置 agent 作为 fallback

---

## 10. 120 秒自动 Backgrounding 机制

```typescript
// AgentTool.tsx:72-77
function getAutoBackgroundMs(): number {
  if (isEnvTruthy(process.env.CLAUDE_AUTO_BACKGROUND_TASKS)
      || getFeatureValue_CACHED_MAY_BE_STALE('tengu_auto_background_agents', false)) {
    return 120_000;  // 120 秒
  }
  return 0;
}
```

运行流程:

```
0s        2s              120s
│         │                │
▼         ▼                ▼
启动      显示 "Press      自动触发
sync      Ctrl+Z to       backgrounding
agent     background"
          hint (BackgroundHint)
                           │
                           ▼
                    ┌──────────────────┐
                    │ backgroundSignal │
                    │    resolved!     │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Promise.race()   │
                    │ 选择 background  │
                    │ 分支             │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ 1. 关闭前台 iter │
                    │ 2. 重启为 async  │
                    │ 3. 返回 output   │
                    │    file 路径     │
                    └──────────────────┘
```

```typescript
// AgentTool.tsx:818-833
if (!isBackgroundTasksDisabled) {
  const registration = registerAgentForeground({
    agentId: syncAgentId,
    description, prompt, selectedAgent,
    setAppState: rootSetAppState,
    toolUseId: toolUseContext.toolUseId,
    autoBackgroundMs: getAutoBackgroundMs() || undefined  // 120s
  });
  foregroundTaskId = registration.taskId;
  backgroundPromise = registration.backgroundSignal.then(() => ({
    type: 'background' as const
  }));
  cancelAutoBackground = registration.cancelAutoBackground;
}
```

[设计决策] 120 秒的阈值选择: 大多数 subagent 在 120 秒内完成。超过 120 秒的通常是大型搜索或复杂实现，用户不太可能一直盯着等。自动 backgrounding 释放了用户的终端。

---

## 11. 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Main REPL Loop                             │
│                                                                     │
│  User Input → query() → model response → tool_use: Agent(...)      │
│                                              │                      │
│                                              ▼                      │
│                                    ┌─────────────────┐              │
│                                    │  AgentTool.call()│              │
│                                    │                 │              │
│                                    │ 1. select agent │              │
│                                    │ 2. build prompt │              │
│                                    │ 3. assemble tools│             │
│                                    │ 4. sync/async?  │              │
│                                    └────────┬────────┘              │
│                                             │                       │
│                              ┌──────────────┼──────────────┐        │
│                              │              │              │        │
│                              ▼              ▼              ▼        │
│                          Sync path    Auto-bg path    Async path   │
│                              │              │              │        │
│                              ▼              ▼              ▼        │
│                         ┌──────────────────────────────────┐        │
│                         │         runAgent()                │        │
│                         │                                   │        │
│                         │  ┌─ init context ────────────┐   │        │
│                         │  │ - resolve model            │   │        │
│                         │  │ - build system prompt      │   │        │
│                         │  │ - assemble tools           │   │        │
│                         │  │ - init MCP servers         │   │        │
│                         │  │ - register hooks           │   │        │
│                         │  │ - preload skills           │   │        │
│                         │  │ - createSubagentContext()   │   │        │
│                         │  └────────────────────────────┘   │        │
│                         │                                   │        │
│                         │  ┌─ conversation loop ────────┐   │        │
│                         │  │ for await (msg of query()) │   │        │
│                         │  │   yield message            │   │        │
│                         │  └────────────────────────────┘   │        │
│                         │                                   │        │
│                         │  ┌─ finally: 9 cleanups ──────┐   │        │
│                         │  │ 1. MCP cleanup              │   │        │
│                         │  │ 2. Session hooks cleanup    │   │        │
│                         │  │ 3. Cache tracking cleanup   │   │        │
│                         │  │ 4. File state cache clear   │   │        │
│                         │  │ 5. Messages array clear     │   │        │
│                         │  │ 6. Perfetto unregister      │   │        │
│                         │  │ 7. Transcript subdir clear  │   │        │
│                         │  │ 8. Todos cleanup            │   │        │
│                         │  │ 9. Kill shell tasks         │   │        │
│                         │  └────────────────────────────┘   │        │
│                         └──────────────────────────────────┘        │
│                                                                     │
│  ← finalizeAgentTool() or enqueueAgentNotification()               │
│  ← result back to main conversation                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 12. Feature Gate 控制

### 12.1 始终可用的 Agents

```typescript
// builtInAgents.ts:45-48
const agents: AgentDefinition[] = [
  GENERAL_PURPOSE_AGENT,  // 始终可用
  STATUSLINE_SETUP_AGENT, // 始终可用
]
```

### 12.2 条件可用的 Agents

| Agent | Gate | 条件 |
|-------|------|------|
| Explore | `BUILTIN_EXPLORE_PLAN_AGENTS` + `tengu_amber_stoat` | Feature flag + GrowthBook A/B test |
| Plan | 同 Explore | 与 Explore 绑定 |
| claude-code-guide | 入口点检查 | 非 SDK 入口 |
| verification | `VERIFICATION_AGENT` + `tengu_hive_evidence` | Feature flag + GrowthBook |

```typescript
// builtInAgents.ts:50-69
if (areExplorePlanAgentsEnabled()) {
  agents.push(EXPLORE_AGENT, PLAN_AGENT)
}

if (isNonSdkEntrypoint) {
  agents.push(CLAUDE_CODE_GUIDE_AGENT)
}

if (feature('VERIFICATION_AGENT')
    && getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false)) {
  agents.push(VERIFICATION_AGENT)
}
```

### 12.3 全局禁用

```typescript
// builtInAgents.ts:25-30
// SDK 用户可以禁用所有内置 agents，从空白开始
if (isEnvTruthy(process.env.CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS)
    && getIsNonInteractiveSession()) {
  return []
}
```

### 12.4 Fork Subagent Gate

```typescript
// forkSubagent.ts:32-39
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false  // 与 coordinator 互斥
    if (getIsNonInteractiveSession()) return false  // SDK 模式下禁用
    return true
  }
  return false
}
```

[设计决策] Fork 与 Coordinator 互斥: coordinator 已有自己的 delegation 模型，fork 会与其冲突。

---

## 13. 常见误区

### 误区 1: "Subagent 能看到父级对话历史"

**错!** 普通 subagent (非 fork) 从**空白上下文**开始。只有 Fork subagent 才继承父级历史。

```typescript
// Normal path:
promptMessages = [createUserMessage({ content: prompt })];
// 只有一条 user message!

// Fork path:
promptMessages = buildForkedMessages(prompt, assistantMessage);
// 包含完整的父级 assistant message + placeholder results + directive
forkContextMessages: toolUseContext.messages  // 还传了完整历史!
```

### 误区 2: "Fork child 的中间结果会流回主线程"

**错!** Fork child 运行完毕后，只有最终结果通过 `<task-notification>` 通知主线程。中间的所有 tool 调用输出都不会出现在主上下文中。

```
主线程收到的只有:
1. 立即: output_file 路径 (可选择读取，但 prompt 说 "Don't peek")
2. 完成时: async notification 带最终消息
```

### 误区 3: "Background agent 和 sync agent 使用相同的 abort 控制"

**错!** Async agent 有独立的 AbortController:

```typescript
// runAgent.ts:524-528
const agentAbortController = override?.abortController
  ? override.abortController
  : isAsync
    ? new AbortController()     // 异步: 独立!
    : toolUseContext.abortController  // 同步: 共享父级
```

这意味着用户按 ESC 取消主线程时，background agent 继续运行。

### 误区 4: "Subagent 的 thinking 和主 agent 一样"

**错!** 普通 subagent 禁用 thinking 以节省 token:

```typescript
thinkingConfig: useExactTools
  ? toolUseContext.options.thinkingConfig  // Fork: 继承
  : { type: 'disabled' },                 // Normal: 禁用!
```

只有 Fork subagent (需要 cache 一致性) 才继承父级的 thinking config。

---

## 14. 跨模块联系

### 与 Part 1 (Conversation Engine) 的关系
- `query()` 函数是 subagent 对话循环的核心，与主 agent 使用同一个引擎
- `createSubagentContext()` 从 `utils/forkedAgent.ts` 创建隔离的 `ToolUseContext`

### 与 Part 4 (Context Lifecycle) 的关系
- Fork subagent 的 cache 优化依赖于 context 的精确控制
- `omitClaudeMd` 和 `omitGitStatus` 是 context 裁剪的实例
- `contentReplacementState` 控制 tool result 的 cache 稳定性

### 与 Permission 系统的关系
- 每个 agent 可以有独立的 `permissionMode`
- `bubble` mode: 权限请求冒泡到父级终端
- `dontAsk`: 跳过权限确认 (如 claude-code-guide)
- 父级的 `bypassPermissions` / `acceptEdits` / `auto` 不会被子 agent 覆盖

### 与 MCP 系统的关系
- Agent 可以定义自己的 MCP servers (frontmatter `mcpServers`)
- 与父级的 MCP clients 合并，agent 结束时清理
- `requiredMcpServers` 字段: agent 在所需 MCP 不可用时自动隐藏

---

## 15. Takeaway

1. **主 agent 优先**: 不要把 "能开 subagent" 等同于 "应该开 subagent"。Claude Code 的默认策略是自己干完。

2. **Fork 是 cache 的胜利**: 四层 cache 优化让 fork 的边际成本接近零。这是 "正确抽象 + 底层优化" 的典范。

3. **隔离是安全的基础**: 每个 agent 有独立的工具池、权限模式、abort 控制。这不是过度设计，是事故预防。

4. **清理是可靠性的保证**: 9 项 finally 清理操作确保无论 agent 如何终止，资源都不会泄漏。

5. **Feature gate 是渐进发布的关键**: 每个 agent 都可以通过 GrowthBook flag 独立控制上线，支持 A/B test。

6. **优先级覆盖链解决了多租户问题**: built-in → plugin → user → project → flag → managed，每层都可以覆盖前一层。

---

## 16. 思考题

1. **Cache 精确性**: Fork 的四层 cache 优化要求所有 fork child 的请求前缀逐字节一致。如果 GrowthBook flag 在两个 fork 之间变化了，会发生什么？系统如何应对？

2. **递归深度**: 当前 fork child 不能再 fork (硬限制)。如果允许有限层数的嵌套 fork (比如最多 2 层)，cache 共享还能工作吗？需要什么额外机制？

3. **Auto-backgrounding 的阈值**: 120 秒是硬编码的。如果你要把它做成自适应的（根据任务类型和历史数据调整），你会怎么设计？

4. **omitClaudeMd 的边界**: Explore 和 Plan agent 省略了 CLAUDE.md。但如果 CLAUDE.md 中包含了关键的代码规范信息（比如 "所有函数必须用 snake_case"），Explore 搜索到的代码如何保证符合规范？

5. **Fork vs Subagent 的决策边界**: 当前的判断标准是 "中间产物是否需要留在上下文"。能否设计一个更精确的启发式规则来自动化这个决策？

---

## 17. 关键源文件索引

| 文件 | 职责 | 行数参考 |
|------|------|---------|
| `src/tools/AgentTool/AgentTool.tsx` | AgentTool 主入口: schema 定义、call() dispatch、sync/async 分支 | ~1050 行 |
| `src/tools/AgentTool/runAgent.ts` | Agent 运行时: 初始化、对话循环、清理 | ~960 行 |
| `src/tools/AgentTool/forkSubagent.ts` | Fork 模式: buildForkedMessages、FORK_AGENT、反递归检测 | ~210 行 |
| `src/tools/AgentTool/builtInAgents.ts` | 内置 agent 注册: feature gate 控制 | ~73 行 |
| `src/tools/AgentTool/loadAgentsDir.ts` | Agent 定义加载: 类型系统、markdown/JSON 解析、优先级覆盖 | ~755 行 |
| `src/tools/AgentTool/prompt.ts` | AgentTool prompt 生成: whenToUse、fork 指导、examples | ~287 行 |
| `src/tools/AgentTool/agentToolUtils.ts` | 工具过滤、结果处理、agent lifecycle 辅助 | ~200 行 |
| `src/tools/AgentTool/built-in/generalPurposeAgent.ts` | general-purpose agent 定义 | ~35 行 |
| `src/tools/AgentTool/built-in/exploreAgent.ts` | Explore agent 定义: read-only、haiku、omitClaudeMd | ~84 行 |
| `src/tools/AgentTool/built-in/planAgent.ts` | Plan agent 定义: read-only、inherit | ~93 行 |
| `src/tools/AgentTool/built-in/verificationAgent.ts` | Verification agent 定义: adversarial、background | ~152 行 |
| `src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts` | Guide agent 定义: haiku、dontAsk、dynamic prompt | ~205 行 |
| `src/tools/AgentTool/built-in/statuslineSetup.ts` | Statusline agent 定义: minimal tools | ~145 行 |
| `src/tools/AgentTool/constants.ts` | 常量: AGENT_TOOL_NAME、ONE_SHOT_BUILTIN_AGENT_TYPES | ~13 行 |
| `src/constants/xml.ts` | FORK_BOILERPLATE_TAG、FORK_DIRECTIVE_PREFIX | 第63-66行 |
| `src/utils/forkedAgent.ts` | createSubagentContext: 隔离的 ToolUseContext 创建 | - |

---

> **Next**: Part 7 将探讨 MCP (Model Context Protocol) 集成架构 -- Claude Code 如何连接外部世界。
