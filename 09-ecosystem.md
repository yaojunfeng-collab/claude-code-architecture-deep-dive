# Part 9: 生态扩展 — MCP、Hook、Plugin 如何分层解耦？

> Claude Code 不只是一个 agent，它是一个可扩展的 agent 平台。
> MCP 解决"能用什么工具"，Hook 解决"工具调用时做什么检查"，Skill 解决"如何编排工作流"，Plugin 解决"如何打包分发"。

---

## 开场：一个平台的四个问题

你在构建一个 coding agent 平台，面对四个问题：

1. **工具扩展**：用户想接入 Gmail、GitHub、Figma——你不可能全部内置
2. **运行时治理**：企业客户要求"所有 rm -rf 必须经过审批"——不能只靠 prompt
3. **工作流复用**：团队有一套标准的 code review 流程——不想每次手动输入
4. **分发管控**：管理员想把上述三者打包成"企业插件"——需要统一的管控面

Claude Code 用四层架构解决这四个问题。这一讲，我们逐层拆解。

---

## 1. 全景：四层架构

```
┌──────────────────────────────────────────────┐
│  Plugin（组合分发包）                           │
│  ┌──────────┐  ┌──────┐  ┌────┐  ┌───────┐  │
│  │ Skills   │  │Hooks │  │MCP │  │Agents │  │
│  └──────────┘  └──────┘  └────┘  └───────┘  │
└──────────────────────────────────────────────┘
         ↓             ↓          ↓
    prompt 注入    运行时拦截    工具桥接
```

| 层 | 核心职责 | 确定性 | 作用域 |
|----|---------|--------|--------|
| **MCP** | 工具桥 + 行为说明注入 | 100%（协议级） | 跨应用 |
| **Hooks** | 运行时治理 | 100%（harness 级） | 单应用 |
| **Skills** | 工作流单元 | ~90%（prompt 级） | 单应用 |
| **Plugins** | 分发打包 | 100%（配置级） | 企业 |

**[核心概念] 分层解耦是关键。** MCP 不关心 Hook 的存在，Hook 不关心 Skill 的存在。每一层独立工作，组合时产生强大的协同效果。

---

## 2. MCP：工具桥与行为注入

### 2.1 设计出发点：N×M 困境

```
之前：N 个 AI 应用 × M 个工具 = N×M 套集成代码
之后：N 个 AI 应用 → 1 个协议 ← M 个工具
```

类比 USB 之于外设、LSP 之于编辑器。MCP 解决的是**组织边界**问题——工具提供者和 AI 应用开发者不是同一个人。

### 2.2 MCP 在 Claude Code 中的三重角色

#### 角色 1：工具桥（Tool Bridge）

支持 6 种传输协议：

| 协议 | 场景 | 特点 |
|------|------|------|
| `stdio` | 本地进程 | 最常见，零网络开销 |
| `sse` / `http` | 远程 HTTP | 标准 REST |
| `ws` | WebSocket | 长连接，低延迟 |
| `sdk` | SDK 嵌入 | Agent SDK 内部用 |
| `claudeai-proxy` | claude.ai web 端 | 浏览器环境代理 |

MCP server 返回的工具被转换为内部 `Tool` 对象（`client.ts:1766-1832`）：

```typescript
{
  name: `mcp__${serverName}__${toolName}`,  // 全限定名，避免冲突
  mcpInfo: { serverName, toolName },
  isMcp: true,
  inputJSONSchema: tool.inputSchema,         // 直接透传，不二次验证
  isConcurrencySafe: () => tool.annotations?.readOnlyHint,
  isDestructive: () => tool.annotations?.destructiveHint,
}
```

**[设计决策] `inputSchema` 直接透传。** Claude Code 不验证 MCP 工具的输入参数——它用 `z.object({}).passthrough()` 接受任意输入。原因：MCP server 千变万化，客户端不可能预知所有 schema。让 server 自己验证更合理。

**[设计决策] 权限返回 `passthrough`。** MCP 工具不自己判断权限，交给 Claude Code 的统一权限系统处理。这保证了企业的权限策略对所有工具一致生效。

#### 角色 2：行为说明注入（Instructions Injection）

MCP server 在 `InitializeResult` 中可携带 `instructions` 字段：

```typescript
function getMcpInstructions(mcpClients) {
  return `# MCP Server Instructions
The following MCP servers have provided instructions:
## ${client.name}
${client.instructions}`
}
```

**这意味着 MCP server 可以影响模型的整体行为**，不仅仅是提供工具。比如 Chrome MCP server 可以告诉模型"截屏前先等 2 秒让页面加载完"。

**[权衡] Instructions 是双刃剑。** 它让 server 能提供使用指南（好的），但也让恶意 server 能注入 prompt（坏的）。Claude Code 有截断保护（`MAX_MCP_DESCRIPTION_LENGTH`），但信任边界需要用户自己把控。

#### 角色 3：能力协商与资源发现

连接后并发获取四种能力：

```typescript
const [tools, commands, skills, resources] = await Promise.all([
  fetchToolsForClient(client),       // tools → 可调用的工具
  fetchCommandsForClient(client),    // prompts → 转为 slash commands
  fetchMcpSkillsForClient(client),   // resources → 转为 skills
  fetchResourcesForClient(client),   // resources → 可读数据源
])
```

### 2.3 MCP vs 朴素 Function Calling

| 维度 | Function Calling | MCP |
|------|-----------------|-----|
| **执行位置** | in-process | out-of-process（独立进程/远程） |
| **工具发现** | 编译时写死 | 运行时 `tools/list` 动态发现 |
| **工具变更** | 改代码重部署 | server 推 `ToolListChanged` |
| **信任模型** | 自己写的，完全信任 | 第三方，需权限系统 |
| **连接状态** | 无状态 | 有状态持久连接 |
| **能力范围** | 只有工具调用 | tools + resources + prompts + instructions |

**[核心概念] 进程边界带来的复杂性。** Function calling 直接调函数，try/catch 即可。MCP 跨进程，需要处理：超时保护、OAuth 过期(401)、Session 过期(404)、URL Elicitation、Server 崩溃重连（指数退避 1s→2s→4s→8s→16s，最多 5 次）。

适用场景判断：
- **Function calling 就够了**：单 AI provider、工具逻辑在自己应用里、列表稳定
- **MCP 有价值**：工具由第三方提供、同一套工具接入多个 AI 应用、需要动态发现

### 2.4 MCP 完整生命周期（以 Gmail 为例）

#### 阶段 1：启动时初始化（用户还没打字之前）

```
Claude Code 启动
    │
    ├── 读配置：getClaudeCodeMcpConfigs()
    │     ├── enterprise (managed-mcp.json)
    │     ├── user (~/.claude/settings.json)
    │     ├── project (.claude/settings.json)
    │     ├── local (.claude/settings.local.json)
    │     ├── plugin configs
    │     └── 合并 + 去重 + 策略过滤
    │
    ├── 同时获取 claude.ai 远程 configs
    │
    ├── 并发连接所有 server
    │     │  对每个 server：
    │     ├── connectToServer("gmail", httpConfig)
    │     │     ├── new StreamableHTTPClientTransport(url)
    │     │     ├── client.connect(transport)  ← MCP initialize 握手
    │     │     ├── 获取 capabilities, instructions
    │     │     │
    │     │     ├── 认证未通过 → 注册 McpAuthTool
    │     │     └── 认证成功 → tools/list + prompts/list + resources/list
    │     │
    │     └── 其他 server 同时并发
    │
    └── 所有 MCP 工具在内存里 → AppState.mcp.tools
```

配置优先级：`plugin < user < project(需 approve) < local < enterprise(独占)`

**[设计决策] 企业级配置存在时排除所有其他来源。** 这确保了企业对 MCP 连接的完全控制。

#### 阶段 2：每次 query

```
tools = [...内置工具, ...AppState.mcp.tools]  ← 合并
system prompt += getMcpInstructions()          ← instructions 注入
→ 发送到 Claude API
```

#### 阶段 3：工具调用

```
findToolByName → 权限检查(passthrough) → PreToolUse hooks
  → callMCPTool(client.callTool, timeout, progress)
  → 结果大小检查(>100K 截断存盘)
  → PostToolUse hooks
```

**一句话总结时序**：启动时连接+发现工具 → 每次 query 合并到 tools 参数 → 模型调用时通过已建立连接转发。

### 2.5 MCP Instructions 与 Cache 的冲突

MCP instructions 在 system prompt 中，每次 server 连接/断开都会改变内容 → 破坏 cache prefix。

**Delta 方案**：把 instructions 从 system prompt 移到消息附件。详见 Part 3。

### 2.6 常用 MCP Server 生态

**自带**：Claude-in-Chrome（浏览器自动化）、computer-use（桌面 GUI）、claude.ai connectors（Gmail/Calendar）

**社区热门**：GitHub MCP、PostgreSQL/Supabase、Linear、Sentry、Brave Search、Playwright、Figma、Notion、Slack

**实际使用判断**：大部分日常场景内置工具就够了。MCP 在三类场景有增量价值：
1. Claude Code 自身没有的能力（浏览器、实时搜索）
2. 连接外部 SaaS（Gmail、Slack、Linear）
3. 安全隔离的执行环境（E2B 沙箱）

---

## 3. Hook：运行时治理

### 3.1 定位

> Hook = "每当 Claude 做了 X，就自动执行 Y"

**[核心概念] Hook vs CLAUDE.md 的本质区别：**

```
CLAUDE.md（自然语言引导）：
  "不要执行 rm -rf"
  → 模型可能忘记（prompt 被 compact 后丢失）
  → 模型可能被 prompt injection 绕过
  → 确定性：~90%

Hook（harness 级确定性执行）：
  PreToolUse → 检测 rm -rf → exit 2 阻止
  → harness 一定会执行
  → 无法被 prompt injection 绕过
  → 确定性：100%
```

### 3.2 四种 Hook 类型

| 类型 | 机制 | 最佳场景 |
|------|------|---------|
| **command** | shell 脚本，stdin/stdout JSON | 验证、格式化、本地检查（最常用） |
| **http** | POST 到外部 HTTP 端点 | 团队审计服务、云端策略 |
| **prompt** | 单轮 LLM 判断（默认 Haiku） | 基于判断的决策 |
| **agent** | 多轮子代理，可读文件、跑命令 | 复杂验证（跑测试套件） |

### 3.3 三步核心机制

1. **事件触发** — harness 在特定节点自动调用脚本
2. **数据传入** — stdin JSON（session_id, cwd, hook_event_name, tool_name, tool_input）
3. **脚本决策** — 退出码：`0` = 放行（stdout 可返回 JSON 控制行为），`2` = 阻止（stderr 反馈给 Claude），其他 = 记录错误不阻止

### 3.4 27 种 Hook 事件全览

**会话生命周期（7 种）**：
SessionStart, SessionEnd, Setup, InstructionsLoaded, CwdChanged, ConfigChange, FileChanged

**用户交互（3 种）**：
UserPromptSubmit, Elicitation, ElicitationResult

**工具执行（5 种）**：
PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest, PermissionDenied

**Agent 协作（5 种）**：
SubagentStart, SubagentStop, TaskCreated, TaskCompleted, TeammateIdle

**上下文管理（2 种）**：
PreCompact, PostCompact

**控制流（3 种）**：
Stop, StopFailure, Notification

**Worktree（2 种）**：
WorktreeCreate, WorktreeRemove

### 3.5 HookResult 控制协议

```json
{
  "blockingError": "禁止执行 rm -rf",    // 非空：阻断操作
  "systemMessage": "注意：...",           // 注入 system 消息
  "permissionBehavior": "allow",          // 覆盖权限决策
  "updatedInput": { "command": "..." },   // 修改工具输入
  "continue": false                       // 不执行后续 hook
}
```

### 3.6 实战示例集

**保护敏感文件（PreToolUse + command）**：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{ "type": "command", "command": ".claude/hooks/check-danger.sh" }]
    }]
  }
}
```

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')
if echo "$COMMAND" | grep -q "rm -rf"; then
  echo "禁止执行 rm -rf" >&2
  exit 2
fi
exit 0
```

**编辑后自动格式化（PostToolUse）**：
```json
{ "matcher": "Edit|Write", "hooks": [{ "type": "command",
  "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }] }
```

**compact 后重注入关键信息（SessionStart + compact matcher）**：
```json
{ "matcher": "compact", "hooks": [{ "type": "command",
  "command": "echo 'Reminder: use Bun, not npm. Run bun test before committing.'" }] }
```

**桌面通知（Notification）**：
```json
{ "hooks": [{ "type": "command",
  "command": "osascript -e 'display notification \"Claude Code 需要你的注意\"'" }] }
```

**完成前自动验证（Stop + agent）**：
```json
{ "hooks": [{ "type": "agent",
  "prompt": "Verify that all unit tests pass.", "timeout": 120 }] }
```

### 3.7 Matcher 与 `if` 条件

```json
{
  "matcher": "Bash",               // 正则匹配工具名
  "if": "Bash(git *)",             // 权限规则语法，更细粒度
  "hooks": [...]
}
```

`if` 避免不必要的进程开销——只在 git 命令时触发 hook，不是所有 Bash。

### 3.8 Hook 与权限系统的耦合

```
工具调用请求 → PreToolUse hooks 执行
  → hook 返回 permissionBehavior: allow/deny/ask/passthrough
  → exit code 2 → 阻止工具执行

权限弹窗 → PermissionRequest hooks
  → hook 可编程式 allow/deny，无需用户交互

权限被拒 → PermissionDenied hooks
  → hook 可请求 retry: true
```

**[可借鉴] 这三层 hook 让权限系统完全可编程化。** 企业可以实现：PreToolUse 检查内部审批系统 → PermissionRequest 自动批准白名单操作 → PermissionDenied 自动重试（换一种方式执行）。

---

## 4. Plugin：分发打包

### 4.1 Plugin 是 Skill 的超集

```
my-plugin/
├── plugin.json      # manifest 元数据
├── commands/        # 自定义命令
├── agents/          # 自定义 agent
├── skills/          # 嵌入的 skills
├── hooks/           # hooks.json
└── output-styles/   # 输出样式
```

| 维度 | Plugin | Skill |
|------|--------|-------|
| 粒度 | 多组件容器 | 单个 prompt |
| 组件 | skills + hooks + MCP/LSP + agents | 仅 markdown prompt |
| 状态 | enable/disable 持久化 | 无状态 |
| 策略控制 | 企业级（`strictPluginOnlyCustomization`） | 用户级 |

### 4.2 企业管控

```
strictPluginOnlyCustomization: true
  → 可指定 surfaces: skills/agents/hooks/mcp
  → 只有通过插件安装的定制才生效
  → 用户自己写的 skills/hooks 被忽略

strictKnownMarketplaces: true
  → 限定管理员批准的插件市场

blockedMarketplaces: ["evil-marketplace.com"]
  → 屏蔽特定来源
```

---

## 5. 产品"CLI 化"趋势与 MCP

### 5.1 "CLI 化"的真实含义

不是让用户敲命令行，而是**把 GUI 背后的能力拆出来，变成可被 AI agent 调用的 API**。

```
过去: 用户 → GUI → 功能
      开发者 → API → 功能（少数人）

现在: 用户 → 自然语言 → AI agent → API → 功能（所有人）
```

### 5.2 三条路径

| 路径 | 方式 | 对应用方要求 |
|------|------|-------------|
| **Computer Use** | 截屏 + 模拟点击 | 零（任何 GUI 都行） |
| **MCP Server** | 标准协议调 API | 提供 API |
| **产品内置 Agent** | 产品自己的 AI 入口 | 全部自己做 |

三者是演进关系也长期共存：`Computer Use（兜底）→ MCP（标准化）→ 内置 Agent（最终形态）`

### 5.3 核心矛盾：控制权之争

```
飞书的思路：我是平台，agent 在我这里
  → 用户留在飞书

Claude Code 的思路：agent 是独立的，通过协议连接一切
  → 用户在一个统一 agent 里跨产品编排
```

**MCP 的价值在中间那层——跨系统编排。** 这是单产品内置 agent 做不到的。

不管哪条路赢，底层都指向 **API 化**。飞书赢了要 API 化让内部 agent 调，MCP 赢了要 API 化暴露为 MCP server。

---

## 6. 四层协作的实际场景

**场景：企业部署 Claude Code，要求所有数据库操作经过审批**

```
Layer 1 — Plugin：安装"数据库安全"插件
  → 带来 skills（SQL 审查流程）+ hooks（拦截 SQL）+ MCP（审批系统连接）

Layer 2 — MCP：连接内部审批系统 MCP server
  → 工具：approve_sql, check_approval_status
  → Instructions："所有 SQL 操作需要先获得审批"

Layer 3 — Hook（PreToolUse + Bash matcher）：
  → 检测 SQL 命令 → 调用 MCP 审批工具 → 未通过则阻止

Layer 4 — Skill：/sql-review
  → 加载 SQL 审查流程 prompt → 分析变更影响 → 自动提交审批
```

四层各司其职，组合产生"自动化数据库操作审批"能力。

---

## 7. 常见误区

### 误区 1："MCP 和 Function Calling 是竞争关系"

不是。MCP 是 Function Calling 的**超集**。MCP 工具最终到模型手里时，形态和 function call 完全一样（都是 tool schema）。MCP 解决的是 function 之外的问题：发现、连接、能力协商、instructions。

### 误区 2："Hook 可以替代 CLAUDE.md"

不能。Hook 处理的是**确定性的、可编程的规则**（阻止/允许/修改）。CLAUDE.md 处理的是**模糊的、语义级的引导**（"优先使用 pnpm"、"代码风格偏好函数式"）。两者互补。

### 误区 3："Plugin 只是 Skill 的包装"

Plugin 可以包含 hooks、MCP server 配置、自定义 agent、输出样式——远超 Skill 的范围。Skill 是 Plugin 的一个组件。

### 误区 4："MCP server 需要在 Claude Code 启动前就运行"

不需要。Claude Code 启动时会**自己启动**配置的 stdio MCP server。远程 HTTP server 需要提前运行，但本地 stdio 是按需启动的。

---

## 8. 跨模块联系

| 涉及 | 详见 |
|------|------|
| MCP 工具的 defer 加载 | **Part 5: Tool 系统** — isDeferredTool() |
| MCP Instructions 破坏 cache | **Part 3: Cache 优化** — DANGEROUS section 和 Delta 方案 |
| Hook 的 PreToolUse 执行链 | **Part 5: Tool 系统** — 9 步执行管道 |
| Skill 的调用链路 | **Part 7: Skill 系统** — 完整 7 步链路 |
| 企业策略（Managed Settings） | **Part 10: 安全体系** — Layer 7 企业策略 |

---

## 9. 核心 Takeaway

1. **四层分层解耦**：MCP（工具桥）、Hook（治理）、Skill（工作流）、Plugin（分发）各自独立，组合时协同。
2. **MCP 解决组织边界问题**——当工具提供者和 AI 应用开发者不是同一个人时需要协议解耦。
3. **Hook 是确定性的 harness 级执行**，安全关键约束不应依赖模型遵循 prompt。
4. **MCP instructions 是双刃剑**——让 server 能提供使用指南，但也引入了 prompt injection 风险。
5. **产品 API 化是不可避免的趋势**——不管 MCP 还是内置 agent 赢，底层都需要 API。

---

## 10. 思考题

1. **如果你在设计 MCP 协议，你会让 server 提供 instructions 吗？** 考虑安全性和功能性的 tradeoff。
2. **Hook 的 `agent` 类型启动一个多轮子代理来做验证。这和 Verification Agent 是什么关系？** 如果冲突了怎么办？
3. **企业场景下 `allowManagedMcpServersOnly: true` 会不会过于限制？** 如果一个开发者需要连接一个临时测试用的 MCP server 怎么办？
4. **MCP 的有状态持久连接（vs function calling 的无状态）带来了什么好处和成本？** 在什么规模下状态管理的复杂性会成为瓶颈？

---

## 关键源文件

| 文件 | 职责 |
|------|------|
| `src/services/mcp/client.ts` | MCP 连接管理、工具转换、错误重试 |
| `src/utils/hooks.ts` | Hook 执行引擎 |
| `src/schemas/hooks.ts` | 27 个 Hook 事件定义 |
| `src/constants/prompts.ts` | MCP Instructions 注入 |
| `src/services/mcpInstructionsDelta.ts` | Delta 方案 |
| `src/skills/loadSkillsDir.ts` | Skill 加载与解析 |
| `src/commands.ts` | 命令/Skill 注册 |
