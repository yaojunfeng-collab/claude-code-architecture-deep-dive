# Part 3: Cache 优化 — 如何让 90% 的 token 走缓存？

> **系列**: Claude Code 2.1.88 Architecture Deep Dive (3/10)
> **前置阅读**: Part 1 (Conversation Engine), Part 2 (Context Management)
> **关键词**: Prompt Cache, KV Cache, prefix matching, cache_control, scope, TTL

---

## 0. 开场：一个数字引发的思考

Claude Code 一次典型的 API 调用，system prompt + tool schemas + messages 加起来可以有 **50K-200K tokens**。如果每次都要从头计算 KV cache，那延迟和成本都是灾难性的。

Anthropic 的 Prompt Cache 机制让相同前缀可以复用 KV cache：
- **cache_creation**: 1.25x 正常 token 价格（写入）
- **cache_read**: 0.1x 正常 token 价格（读取，**便宜 12.5 倍**）

也就是说，如果一个 200K token 的请求有 180K 走 cache_read，成本从 200K 单位降到 `20K * 1.0 + 180K * 0.1 = 38K` 单位 — **节省 81%**。

但要达到 90% 的缓存命中率，需要精心的工程设计。Claude Code 2.1.88 在这方面做到了极致，本篇将逐层拆解。

---

## 1. Prompt Cache 基础原理

### 1.1 这不是客户端缓存

[核心概念] Prompt Cache 是 Anthropic **服务端**的 KV Cache 机制。客户端（Claude Code）的角色是：

1. 在请求中标记 `cache_control` breakpoint
2. 保持请求内容前缀尽量稳定
3. 检测缓存命中/丢失，诊断问题

服务端做的事情：
1. 对请求内容做 **逐字节（byte-by-byte）的严格前缀匹配**
2. 如果找到匹配的 KV cache entry，复用它（cache_read）
3. 如果没找到，创建新的 cache entry（cache_creation）

```
┌─────────────────────────────────────────────────────┐
│                  Anthropic Server                    │
│                                                     │
│  Cache Key = hash(                                  │
│    system blocks                                    │
│    + tool schemas                                   │
│    + model ID                                       │
│    + thinking config                                │
│    + beta headers                                   │
│    + messages prefix (up to cache_control marker)   │
│  )                                                  │
│                                                     │
│  Match? ──→ cache_read (0.1x price)                 │
│  Miss?  ──→ cache_creation (1.25x price)            │
└─────────────────────────────────────────────────────┘
```

### 1.2 严格前缀匹配的含义

[核心概念] "前缀匹配"意味着：

- 服务端从请求的第一个字节开始比较
- 任何位置的一个字节不同，**从该位置开始的所有内容都无法复用**
- cache_control marker 告诉服务端"到这里为止可以作为一个可复用的前缀"

```
请求 1: [AAAA BBBB CCCC DDDD]  ← cache_control 在末尾
请求 2: [AAAA BBBB CCCC EEEE]  ← 只有 DDDD→EEEE 变了

Server 比较:
  AAAA = AAAA ✓
  BBBB = BBBB ✓
  CCCC = CCCC ✓
  → 前缀匹配到 CCCC 结尾！
  → CCCC 之前的 KV 计算全部复用（cache_read）
  → 只有 EEEE 需要新计算（cache_creation）
```

这就是为什么 Claude Code 要如此执着地保持 system prompt 和 messages 前缀的稳定性。

### 1.3 Cache Scope 机制

[核心概念] Anthropic 支持三级 cache scope：

| scope | 含义 | 共享范围 |
|-------|------|----------|
| (无/per-user) | 默认 | 同一个 API key 的请求 |
| `'org'` | 组织级别 | 同一个 organization 下所有 key |
| `'global'` | 全球级别 | **所有用户共享** |

Global scope 的意义：**第一个发送某段 system prompt 的用户支付 cache_creation，全球后续所有用户直接 cache_read**。

对 Claude Code 来说，所有用户的核心 system prompt 都是一样的（身份介绍、系统规则、工具使用指南等），这部分完全可以 global cache。

---

## 2. 三层缓存策略全景

Claude Code 的 cache 优化分为三层，从外到内层层递进：

```
┌─────────────────────────────────────────────────────────────────┐
│                    一次完整的 API 请求                            │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Layer 1: System Prompt                                    │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────┐                      │  │
│  │  │ TextBlock 1 (attribution)       │ cacheScope: null     │  │
│  │  ├─────────────────────────────────┤                      │  │
│  │  │ TextBlock 2 (CLI prefix)        │ cacheScope: null     │  │
│  │  ├─────────────────────────────────┤                      │  │
│  │  │ TextBlock 3 (static content)    │ cacheScope: 'global' │  │
│  │  │  intro / system / tasks /       │ ← 全球所有用户共享    │  │
│  │  │  actions / tools / tone / eff   │                      │  │
│  │  ├── DYNAMIC_BOUNDARY ─────────────┤                      │  │
│  │  │ TextBlock 4 (dynamic content)   │ cacheScope: null     │  │
│  │  │  session / memory / env / lang  │ ← 每用户不同,不缓存   │  │
│  │  └─────────────────────────────────┘                      │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Tool Schemas (tools array)                                │  │
│  │  BashTool / FileEditTool / ... / MCP tools               │  │
│  │  → cache_control 在最后一个 tool 上                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Layer 2: Messages                                         │  │
│  │                                                           │  │
│  │  [msg 0] user: "修复这个 bug"                              │  │
│  │  [msg 1] assistant: "让我看看..."                          │  │
│  │  [msg 2] user: tool_result                                │  │
│  │  [msg 3] assistant: "找到了，问题在..."                     │  │
│  │  [msg 4] user: "好的，请修改" ← cache_control: ephemeral  │  │
│  │                                 ↑ Layer 2: 只标一个点      │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Layer 3: Beta Headers (request-level)                     │  │
│  │                                                           │  │
│  │  beta headers 参与 cache key 计算                          │  │
│  │  → sticky-on latch 保证 header 集合不变                    │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Layer 1: System Prompt Global/Org 分区

### 3.1 分区原理

System prompt 由两类内容组成：

**静态内容** — 全球所有 Claude Code 用户完全相同：
- 身份介绍 (`getSimpleIntroSection()`)
- 系统规则 (`getSimpleSystemSection()`)
- 任务指引 (`getSimpleDoingTasksSection()`)
- 操作规范 (`getActionsSection()`)
- 工具使用指南 (`getUsingYourToolsSection()`)
- 风格要求 (`getSimpleToneAndStyleSection()`)
- 输出效率 (`getOutputEfficiencySection()`)

**动态内容** — 每个用户/会话不同：
- session guidance (工具集相关)
- memory (用户的 CLAUDE.md)
- env info (操作系统、git 状态等)
- language (语言偏好)
- MCP instructions (MCP 服务器指令)

### 3.2 SYSTEM_PROMPT_DYNAMIC_BOUNDARY

`src/constants/prompts.ts` 定义了分界标记：

```typescript
// src/constants/prompts.ts:114-115
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

在构建 system prompt 时，这个标记被插入到静态和动态内容之间：

```typescript
// src/constants/prompts.ts:560-576
return [
  // --- Static content (cacheable) ---
  getSimpleIntroSection(outputStyleConfig),
  getSimpleSystemSection(),
  getSimpleDoingTasksSection(),
  getActionsSection(),
  getUsingYourToolsSection(enabledTools),
  getSimpleToneAndStyleSection(),
  getOutputEfficiencySection(),
  // === BOUNDARY MARKER - DO NOT MOVE OR REMOVE ===
  ...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),
  // --- Dynamic content (registry-managed) ---
  ...resolvedDynamicSections,
].filter(s => s !== null)
```

注意：这个标记**不会**发送给模型。它是给 `splitSysPromptPrefix()` 看的内部标记。

### 3.3 splitSysPromptPrefix() 分区逻辑

`src/utils/api.ts:321-410` 的 `splitSysPromptPrefix()` 是 Layer 1 的核心。它有三种模式：

```
模式判定流程:

shouldUseGlobalCacheScope() ?
├── true + skipGlobalCacheForSystemPrompt=true
│   → 模式 1: MCP tools present，org-scope fallback
│
├── true + DYNAMIC_BOUNDARY found
│   → 模式 2: Global cache mode（理想状态）
│
└── false / boundary missing
    → 模式 3: Default org-scope
```

**模式 2（理想状态）的输出**：

```typescript
// src/utils/api.ts:387-404
const result: SystemPromptBlock[] = []
if (attributionHeader)
  result.push({ text: attributionHeader, cacheScope: null })
if (systemPromptPrefix)
  result.push({ text: systemPromptPrefix, cacheScope: null })
const staticJoined = staticBlocks.join('\n\n')
if (staticJoined)
  result.push({ text: staticJoined, cacheScope: 'global' })  // ← 全球共享!
const dynamicJoined = dynamicBlocks.join('\n\n')
if (dynamicJoined)
  result.push({ text: dynamicJoined, cacheScope: null })      // ← 不缓存
```

### 3.4 buildSystemPromptBlocks() 将 scope 转为 API 格式

`src/services/api/claude.ts:3213-3237` 将 `splitSysPromptPrefix()` 的输出转为 Anthropic API 的 `TextBlockParam[]`：

```typescript
// src/services/api/claude.ts:3213-3236
export function buildSystemPromptBlocks(
  systemPrompt: SystemPrompt,
  enablePromptCaching: boolean,
  options?: { ... },
): TextBlockParam[] {
  return splitSysPromptPrefix(systemPrompt, {
    skipGlobalCacheForSystemPrompt: options?.skipGlobalCacheForSystemPrompt,
  }).map(block => {
    return {
      type: 'text' as const,
      text: block.text,
      ...(enablePromptCaching &&
        block.cacheScope !== null && {
          cache_control: getCacheControl({
            scope: block.cacheScope,
            querySource: options?.querySource,
          }),
        }),
    }
  })
}
```

[设计决策] 注意 `cacheScope: null` 的 block 不会添加 `cache_control`。这意味着：
- attribution header → 不缓存（每个组织不同）
- CLI prefix → 不缓存（版本可能不同）
- static content → `scope: 'global'`（全球共享）
- dynamic content → 不缓存（每用户不同）

### 3.5 MCP 工具的退化效应

[权衡] 当用户连接了非 deferred 的 MCP 工具时，tool schemas 包含了用户自定义内容，无法用 global scope：

```typescript
// src/services/api/claude.ts:1207-1215
const useGlobalCacheFeature = shouldUseGlobalCacheScope()
const willDefer = (t: Tool) =>
  useToolSearch && (deferredToolNames.has(t.name) || shouldDeferLspTool(t))
// MCP tools are per-user → dynamic tool section → can't globally cache.
// Only gate when an MCP tool will actually render (not defer_loading).
const needsToolBasedCacheMarker =
  useGlobalCacheFeature && ...hasMcpTools...
```

退化路径：

```
全局缓存（理想）:
  system prompt static → global scope
  tool schemas         → global scope

MCP 工具存在时（退化）:
  system prompt        → org scope（skipGlobalCacheForSystemPrompt=true）
  tool schemas         → org scope
```

这就是为什么 Claude Code 对 MCP 工具做了 deferred loading — deferred 的 MCP 工具不进入初始 tool schemas，保住 global cache。

### 3.6 shouldUseGlobalCacheScope() 的条件

```typescript
// src/utils/betas.ts:227-232
export function shouldUseGlobalCacheScope(): boolean {
  return (
    getAPIProvider() === 'firstParty' &&
    !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS)
  )
}
```

仅限 firstParty provider（直接使用 Anthropic API 的用户）。Foundry、Bedrock、Vertex 不支持。

### 3.7 Global Cache 的经济学意义

[可借鉴] 假设 Claude Code static system prompt 有 ~15K tokens：

```
没有 global cache:
  每个用户首次请求: 15K * 1.25 = 18,750 token-units（cache_creation）
  后续请求: 15K * 0.1 = 1,500 token-units（cache_read，但只有同用户）

有 global cache:
  全球第一个用户: 15K * 1.25 = 18,750 token-units
  全球后续所有用户: 15K * 0.1 = 1,500 token-units

  如果有 100,000 个活跃用户:
    节省 = 99,999 * (18,750 - 1,500) = ~17.2 亿 token-units
```

---

## 4. Layer 2: Message-level 单点缓存

### 4.1 核心规则：只标一个 cache_control

`src/services/api/claude.ts:3078-3106` 的 `addCacheBreakpoints()` 实现了 message-level 缓存：

```typescript
// src/services/api/claude.ts:3078-3089 — 关键注释
// Exactly one message-level cache_control marker per request. Mycro's
// turn-to-turn eviction (page_manager/index.rs: Index::insert) frees
// local-attention KV pages at any cached prefix position NOT in
// cache_store_int_token_boundaries. With two markers the second-to-last
// position is protected and its locals survive an extra turn even though
// nothing will ever resume from there — with one marker they're freed
// immediately. For fire-and-forget forks (skipCacheWrite) we shift the
// marker to the second-to-last message: that's the last shared-prefix
// point, so the write is a no-op merge on mycro (entry already exists)
// and the fork doesn't leave its own tail in the KVCC.
```

```typescript
// src/services/api/claude.ts:3089-3106
const markerIndex = skipCacheWrite
  ? messages.length - 2  // fork child: 倒数第二条
  : messages.length - 1  // 正常: 最后一条

const result = messages.map((msg, index) => {
  const addCache = index === markerIndex
  if (msg.type === 'user') {
    return userMessageToMessageParam(msg, addCache, enablePromptCaching, querySource)
  }
  return assistantMessageToMessageParam(msg, addCache, enablePromptCaching, querySource)
})
```

### 4.2 为什么只标一个点？

[设计决策] 这是基于 Anthropic 服务端 **Mycro KV Cache** 的 page eviction 机制做出的选择。

```
Mycro KV Cache 内部结构:
┌─────────────────────────────────────────────┐
│ Dense Pages: 存储 cross-attention 计算结果    │
│ Local Pages: 存储 local-attention 计算结果    │
│                                             │
│ 驱逐规则:                                    │
│   cache_store_int_token_boundaries 集合中     │
│   的位置，其 local pages 被保护               │
│   不在集合中的位置，local pages 被释放         │
└─────────────────────────────────────────────┘
```

如果标了两个 cache_control marker（比如 msg[3] 和 msg[5]）：

```
两个 marker 的问题:

Turn N:   [msg0][msg1][msg2][msg3*][msg4][msg5*]
                                ↑              ↑
                          marker 1        marker 2

Turn N+1: [msg0][msg1][msg2][msg3*][msg4][msg5*][msg6][msg7*]
                                ↑                        ↑
                          old marker 1            new marker 2

问题: msg3 和 msg5 的 local KV pages 都被保护
      但 msg5 之后不会有任何请求从 msg5 处恢复
      → msg5 的 local pages 白白占用内存

一个 marker 的方案:

Turn N:   [msg0][msg1][msg2][msg3][msg4][msg5*]
                                            ↑
                                      只有这一个

Turn N+1: [msg0][msg1][msg2][msg3][msg4][msg5][msg6][msg7*]
                                                        ↑
                                                  移到最新

      → msg5 的 local pages 在 N+1 时立即释放
      → 只有 msg7 的 local pages 被保护（这是需要的）
```

[权衡] 一个 marker 意味着：如果前缀完全匹配，之前的所有 KV 都可以复用；但不能缓存中间的某个"检查点"。Claude Code 选择了内存效率优先。

### 4.3 Fork Child 的特殊处理

当 Claude Code fork 一个子 agent 时（`skipCacheWrite = true`）：

```typescript
const markerIndex = skipCacheWrite
  ? messages.length - 2  // ← 倒数第二条
  : messages.length - 1
```

[设计决策] Fork child 的最后一条消息是 fork 独有的（不同的 agent 有不同的 fork 指令），不应该写入 cache。

标在 `messages.length - 2`（也就是 fork 点之前的最后一条共享消息）意味着：
- 这个 cache entry 在 Mycro 中已存在（父进程写过），所以是 no-op merge
- Fork child 的尾部不会留在 KV Cache Controller (KVCC) 中
- Dense pages 通过 refcount 机制继续存活

### 4.4 三轮对话的 prefix cache 示例

```
Turn 1: (首次请求)
┌──────────────────────────────────────────────┐
│ system prompt (15K tokens)                   │  cache_creation
│ tool schemas (8K tokens)                     │  cache_creation
│ msg[0] user: "请帮我修改 foo.ts" [cache*]    │  cache_creation
└──────────────────────────────────────────────┘
Total: 23K cache_creation, 0 cache_read

Turn 2: (第二次请求)
┌──────────────────────────────────────────────┐
│ system prompt (15K tokens)                   │  cache_read ✓
│ tool schemas (8K tokens)                     │  cache_read ✓
│ msg[0] user: "请帮我修改 foo.ts"             │  cache_read ✓
│ msg[1] assistant: "我来看看..." (2K tokens)   │  cache_read ✓
│ msg[2] user: tool_result (3K tokens)         │  cache_read ✓
│ msg[3] assistant: "找到问题了..." (1K tokens) │  cache_read ✓
│ msg[4] user: "好的请修改" [cache*]            │  cache_creation
└──────────────────────────────────────────────┘
Total: 1K cache_creation, 29K cache_read (96.7% hit rate!)

Turn 3: (第三次请求)
┌──────────────────────────────────────────────┐
│ system prompt (15K tokens)                   │  cache_read ✓
│ ... (之前的所有消息)                          │  cache_read ✓
│ msg[6] user: "还有别的问题吗?" [cache*]       │  cache_creation
└──────────────────────────────────────────────┘
Total: ~1K cache_creation, ~35K cache_read (97%+ hit rate!)
```

---

## 5. Layer 3: Beta Header Latching

### 5.1 问题：Beta Headers 参与 Cache Key

Anthropic API 的 beta headers（如 `fast-mode-2025-04-18`、`afk-mode-2025-01-09`）**参与 server-side cache key 计算**。

这意味着：如果 Turn 1 发了 `[header-A, header-B]`，Turn 2 发了 `[header-A]`（因为某个条件变了），即使 system prompt 和 messages 完全一样，**整个 cache 也会 miss**。

### 5.2 Sticky-on Latch 解决方案

`src/services/api/claude.ts:1405-1440`：

```typescript
// src/services/api/claude.ts:1405-1410
// Sticky-on latches for dynamic beta headers. Each header, once first
// sent, keeps being sent for the rest of the session so mid-session
// toggles don't change the server-side cache key and bust ~50-70K tokens.
// Latches are cleared on /clear and /compact via clearBetaHeaderLatches().
// Per-call gates (isAgenticQuery, querySource===repl_main_thread) stay
// per-call so non-agentic queries keep their own stable header set.
```

三个 latched headers：

```typescript
// AFK mode header latch
let afkHeaderLatched = getAfkModeHeaderLatched() === true
if (!afkHeaderLatched && /* 条件满足 */) {
  afkHeaderLatched = true
  setAfkModeHeaderLatched(true)  // 永久 latch
}

// Fast mode header latch
let fastModeHeaderLatched = getFastModeHeaderLatched() === true
if (!fastModeHeaderLatched && isFastMode) {
  fastModeHeaderLatched = true
  setFastModeHeaderLatched(true)  // 永久 latch
}

// Cache editing header latch
let cacheEditingHeaderLatched = getCacheEditingHeaderLatched() === true
if (!cacheEditingHeaderLatched && /* cache editing enabled */) {
  cacheEditingHeaderLatched = true
  setCacheEditingHeaderLatched(true)  // 永久 latch
}
```

### 5.3 Latch 的关键特性

[核心概念] "Sticky-on" 意味着**只能开不能关**：

```
Session 时间线:

Turn 1: fast mode OFF  → headers: [base-headers]
Turn 2: fast mode ON   → headers: [base-headers, fast-mode]
                                                  ↑ latched!
Turn 3: fast mode OFF  → headers: [base-headers, fast-mode]
                                                  ↑ 仍然发送!
Turn 4: fast mode OFF  → headers: [base-headers, fast-mode]
                                                  ↑ 仍然发送!

实际行为: speed 字段控制是否真的用 fast mode
          header 只是为了保持 cache key 稳定
```

[设计决策] 这是 header 和 behavior 的解耦。Header 决定 cache key，behavior 由其他参数控制。

### 5.4 Latch 重置时机

```typescript
// src/constants/systemPromptSections.ts:65-68
export function clearSystemPromptSections(): void {
  clearSystemPromptSectionState()
  clearBetaHeaderLatches()  // ← /clear 和 /compact 时重置
}
```

```typescript
// src/bootstrap/state.ts:1744-1747
export function clearBetaHeaderLatches(): void {
  STATE.afkModeHeaderLatched = null
  STATE.fastModeHeaderLatched = null
  STATE.cacheEditingHeaderLatched = null
}
```

[设计决策] `/clear` 和 `/compact` 重置 latch 是因为：
- `/clear` 清空整个对话，新的 cache key 从零开始
- `/compact` 替换所有 messages，cache prefix 完全变化
- 两者都意味着 "旧的 cache entry 已不可复用"，所以重新评估 headers 是安全的

---

## 6. Cache TTL

### 6.1 两档 TTL

| TTL | 条件 | 用途 |
|-----|------|------|
| 5 分钟 | 默认 | 免费用户、超额用户 |
| 1 小时 | ant 用户 / 付费订阅用户 / Bedrock 用户 opt-in | 长会话不丢失 cache |

### 6.2 should1hCacheTTL() 实现

```typescript
// src/services/api/claude.ts:393-434
function should1hCacheTTL(querySource?: QuerySource): boolean {
  // Bedrock 用户通过 env var opt-in
  if (getAPIProvider() === 'bedrock' &&
      isEnvTruthy(process.env.ENABLE_PROMPT_CACHING_1H_BEDROCK)) {
    return true
  }

  // Latch 资格判定到 bootstrap state — 防止 mid-session overage 翻转
  let userEligible = getPromptCache1hEligible()
  if (userEligible === null) {
    userEligible =
      process.env.USER_TYPE === 'ant' ||
      (isClaudeAISubscriber() && !currentLimits.isUsingOverage)
    setPromptCache1hEligible(userEligible)  // ← latch! 会话内不变
  }
  if (!userEligible) return false

  // 通过 GrowthBook allowlist 按 querySource 控制范围
  let allowlist = getPromptCache1hAllowlist()
  if (allowlist === null) {
    const config = getFeatureValue_CACHED_MAY_BE_STALE(
      'tengu_prompt_cache_1h_config', {})
    allowlist = config.allowlist ?? []
    setPromptCache1hAllowlist(allowlist)  // ← latch!
  }

  return querySource !== undefined &&
    allowlist.some(pattern =>
      pattern.endsWith('*')
        ? querySource.startsWith(pattern.slice(0, -1))
        : querySource === pattern,
    )
}
```

[设计决策] 资格和 allowlist 都 latch 到 bootstrap state。原因：
- 用户可能在会话中途超额（overage），如果 TTL 从 1h 变成 5min，`cache_control` 字段变化，cache key 变化，**整个 cache 丢失**
- GrowthBook 的 disk cache 可能在 mid-request 更新，latch 防止同一会话内出现混合 TTL

### 6.3 getCacheControl() 组装

```typescript
// src/services/api/claude.ts:358-374
export function getCacheControl({
  scope,
  querySource,
}: { scope?: CacheScope; querySource?: QuerySource }): {
  type: 'ephemeral'
  ttl?: '1h'
  scope?: CacheScope
} {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
    ...(scope === 'global' && { scope }),
  }
}
```

最终发送给 API 的 `cache_control` 对象可能是：
- `{ type: 'ephemeral' }` — 默认 5min
- `{ type: 'ephemeral', ttl: '1h' }` — 付费用户
- `{ type: 'ephemeral', scope: 'global' }` — 静态 system prompt (5min)
- `{ type: 'ephemeral', ttl: '1h', scope: 'global' }` — 付费用户的静态 system prompt

---

## 7. Cache Miss 检测系统

### 7.1 两阶段检测架构

`src/services/api/promptCacheBreakDetection.ts` 实现了一个精密的 cache miss 诊断系统。

```
Phase 1: Pre-call                    Phase 2: Post-call
recordPromptState()                  checkResponseForCacheBreak()
     │                                    │
     ▼                                    ▼
 快照当前状态                          比较 cache_read tokens
 与上次快照对比                        如果下降 >5% 且 >2000 tokens
 记录所有变化维度                      → 用 Phase 1 的变化信息
 存入 pendingChanges                     解释原因
```

### 7.2 Phase 1: recordPromptState()

记录 **12 个维度** 的状态快照：

```typescript
// src/services/api/promptCacheBreakDetection.ts:227-241
export type PromptStateSnapshot = {
  system: TextBlockParam[]           // 1. system prompt blocks
  toolSchemas: BetaToolUnion[]       // 2. tool schemas
  querySource: QuerySource           // 3. 请求来源
  model: string                      // 4. 模型 ID
  agentId?: AgentId                  // 5. agent ID
  fastMode?: boolean                 // 6. fast mode 状态
  globalCacheStrategy?: string       // 7. 缓存策略
  betas?: readonly string[]          // 8. beta headers
  autoModeActive?: boolean           // 9. auto mode 状态
  isUsingOverage?: boolean           // 10. 超额状态
  cachedMCEnabled?: boolean          // 11. cached microcompact
  effortValue?: string | number      // 12. effort 级别
}
```

每次 API 调用前，对当前状态和上次状态做 diff：

```typescript
// src/services/api/promptCacheBreakDetection.ts:332-346
const systemPromptChanged = systemHash !== prev.systemHash
const toolSchemasChanged = toolsHash !== prev.toolsHash
const modelChanged = model !== prev.model
const fastModeChanged = isFastMode !== prev.fastMode
const cacheControlChanged = cacheControlHash !== prev.cacheControlHash
const globalCacheStrategyChanged = globalCacheStrategy !== prev.globalCacheStrategy
const betasChanged = sortedBetas.length !== prev.betas.length || ...
const autoModeChanged = autoModeActive !== prev.autoModeActive
const overageChanged = isUsingOverage !== prev.isUsingOverage
const cachedMCChanged = cachedMCEnabled !== prev.cachedMCEnabled
const effortChanged = effortStr !== prev.effortValue
const extraBodyChanged = extraBodyHash !== prev.extraBodyHash
```

### 7.3 Phase 2: checkResponseForCacheBreak()

API 调用返回后，检查实际的 cache token 数据：

```typescript
// src/services/api/promptCacheBreakDetection.ts:485-492
// 检测条件: cache read 下降 >5% 且绝对值 >2000 tokens
const tokenDrop = prevCacheRead - cacheReadTokens
if (
  cacheReadTokens >= prevCacheRead * 0.95 ||
  tokenDrop < MIN_CACHE_MISS_TOKENS    // 2000
) {
  state.pendingChanges = null
  return  // 不算 cache break
}
```

### 7.4 诊断结果分类

```
优先级从高到低:

1. Client-side 可解释的变化:
   - model changed (model_A → model_B)
   - system prompt changed (+N chars)
   - tools changed (+N/-N tools)
   - fast mode toggled
   - global cache strategy changed
   - cache_control changed (scope or TTL)
   - betas changed (+header_X -header_Y)
   - auto mode toggled
   - overage state changed
   - cached microcompact toggled
   - effort changed
   - extra body params changed

2. TTL 过期（client-side 无变化）:
   - possible 1h TTL expiry (>1h since last assistant msg)
   - possible 5min TTL expiry (>5min since last assistant msg)

3. 服务端原因:
   - likely server-side (prompt unchanged, <5min gap)
     → routing/eviction 或 billed/inference disagreement

4. unknown cause
```

### 7.5 Cache Break 常见原因及对策表

| 原因 | 频率 | 对策 | 实现位置 |
|------|------|------|----------|
| System prompt 内容变化 | 中 | systemPromptSection() 缓存 + DANGEROUS 隔离 | `systemPromptSections.ts` |
| Tool schema 动态变化 | 中 | defer MCP tools + agent list 移到 attachment | `prompt.ts` |
| Beta header 翻转 | 低(已修) | sticky-on latch | `claude.ts:1405` |
| Model 切换 | 低 | 不可避免，直接接受 | - |
| TTL 过期(5min) | 高 | 升级到 1h TTL | `should1hCacheTTL()` |
| TTL 过期(1h) | 低 | 不可避免 | - |
| Overage 状态翻转 | 低(已修) | latch 到 session | `should1hCacheTTL()` |
| MCP 服务器 connect/disconnect | 中 | Delta mode（详见下文） | `mcpInstructionsDelta.ts` |
| Auto mode 翻转 | 低(已修) | sticky-on latch | `claude.ts:1412` |
| Effort value 变化 | 低 | 目前不可避免 | - |

### 7.6 diff 文件输出

当检测到 cache break 时，系统会生成一个 diff 文件供调试：

```typescript
// src/services/api/promptCacheBreakDetection.ts:708-727
async function writeCacheBreakDiff(prevContent, newContent) {
  const diffPath = getCacheBreakDiffPath()
  const patch = createPatch('prompt-state', prevContent, newContent, 'before', 'after')
  await writeFile(diffPath, patch)
  return diffPath
}
```

输出格式：
```
[PROMPT CACHE BREAK] system prompt changed (+142 chars)
  [source=repl_main_thread, call #5, cache read: 45000 → 12000,
   creation: 38000, diff: /tmp/claude-xxx/cache-break-a1b2.diff]
```

---

## 8. DANGEROUS_uncachedSystemPromptSection 深入分析

### 8.1 两种 System Prompt Section 的区别

```typescript
// src/constants/systemPromptSections.ts

// 普通 section — 计算一次，缓存到 /clear 或 /compact
export function systemPromptSection(
  name: string,
  compute: ComputeFn,
): SystemPromptSection {
  return { name, compute, cacheBreak: false }  // ← cacheBreak: false
}

// 危险 section — 每轮重新计算，可能破坏缓存
export function DANGEROUS_uncachedSystemPromptSection(
  name: string,
  compute: ComputeFn,
  _reason: string,                              // ← 必须说明原因!
): SystemPromptSection {
  return { name, compute, cacheBreak: true }    // ← cacheBreak: true
}
```

### 8.2 resolveSystemPromptSections() 的缓存逻辑

```typescript
// src/constants/systemPromptSections.ts:43-58
export async function resolveSystemPromptSections(
  sections: SystemPromptSection[],
): Promise<(string | null)[]> {
  const cache = getSystemPromptSectionCache()  // 内存 Map

  return Promise.all(
    sections.map(async s => {
      if (!s.cacheBreak && cache.has(s.name)) {
        return cache.get(s.name) ?? null  // ← 命中缓存，直接返回
      }
      const value = await s.compute()       // ← cacheBreak=true 每次都执行
      setSystemPromptSectionCacheEntry(s.name, value)
      return value
    }),
  )
}
```

普通 section 的 `compute()` 只在首次调用和 `/clear`/`/compact` 后执行。
DANGEROUS section 的 `compute()` **每次 API 调用都执行**。

### 8.3 为什么 mcp_instructions 是唯一的 DANGEROUS section

查看所有 dynamic sections 的定义：

```typescript
// src/constants/prompts.ts:491-555
const dynamicSections = [
  systemPromptSection('session_guidance', () => ...),      // 普通
  systemPromptSection('memory', () => ...),                 // 普通
  systemPromptSection('ant_model_override', () => ...),     // 普通
  systemPromptSection('env_info_simple', () => ...),        // 普通
  systemPromptSection('language', () => ...),               // 普通
  systemPromptSection('output_style', () => ...),           // 普通

  // ↓↓↓ 唯一的 DANGEROUS section ↓↓↓
  DANGEROUS_uncachedSystemPromptSection(
    'mcp_instructions',
    () => isMcpInstructionsDeltaEnabled()
      ? null
      : getMcpInstructionsSection(mcpClients),
    'MCP servers connect/disconnect between turns',         // ← 原因
  ),

  systemPromptSection('scratchpad', () => ...),             // 普通
  systemPromptSection('frc', () => ...),                    // 普通
  // ...
]
```

[设计决策] 为什么只有 `mcp_instructions` 是 DANGEROUS？

因为 MCP 服务器可以**在任何两轮对话之间**连接或断开：
- 用户在 settings.json 里配置了新的 MCP server
- MCP server 进程崩溃后重启
- 延迟加载的 MCP server 在后台完成连接

其他 section 不会在 turn 间变化：
- memory: 只在启动时加载
- env_info: OS/git 信息在会话内不变
- language: 用户设置在会话内不变

### 8.4 _reason 参数的代码审查价值

```typescript
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => ...,
  'MCP servers connect/disconnect between turns',  // ← 这个 string 在运行时不使用!
)
```

[可借鉴] `_reason` 参数以 `_` 开头，编译器不会报未使用警告。它的唯一目的是**作为代码审查提示**——当有人添加新的 DANGEROUS section 时，必须在 code review 中解释原因。

这是一种"类型系统之外的契约约束"模式，非常值得借鉴。

### 8.5 具体的缓存破坏场景

假设一个 50K token 的会话，在 Turn 5 时一个 MCP server 连接了：

```
Turn 1-4 (MCP 未连接):
  system prompt = [static...] + [session + memory + env + null(mcp)]
  → mcp_instructions compute() 返回 null
  → system prompt 内容稳定
  → cache hit ✓

Turn 5 (MCP 连接了):
  system prompt = [static...] + [session + memory + env + "## my-server\n..."]
  → mcp_instructions compute() 返回新内容!
  → system prompt 从变化点开始不匹配
  → 变化点之后所有内容 cache miss ✗

  损失: 假设 mcp_instructions 在 dynamic 部分的中间位置
        dynamic 部分 ~5K tokens 中，变化点之后约 3K tokens cache miss
        加上后续所有 messages ~40K tokens 全部 cache miss
        → 总计 ~43K tokens 需要 cache_creation（1.25x）

Turn 6+ (MCP 保持连接):
  → 但 DANGEROUS 意味着每轮都重新计算!
  → 如果 mcp_instructions 内容没变 → string 比较相同 → cache hit
  → 如果又有新 MCP server 连接 → 再次 cache miss
```

---

## 9. Delta Mode: MCP Instructions 的缓存友好方案

### 9.1 问题本质

DANGEROUS_uncachedSystemPromptSection 的问题在于：**把动态内容放在了 system prompt 里**。system prompt 是 cache key 的一部分，任何变化都影响整个前缀。

### 9.2 Delta Mode 的核心思路

[设计决策] 将 MCP instructions 从 system prompt 移到 **message attachment**:

```
之前 (DANGEROUS mode):
  system prompt: [...static..., ..., "## my-server\n...", ...]
                                     ↑ 变化 → cache bust

之后 (Delta mode):
  system prompt: [...static..., ..., null, ...]  ← mcp_instructions 返回 null
  messages: [
    ...,
    { type: 'attachment',
      attachment: {
        type: 'mcp_instructions_delta',
        addedNames: ['my-server'],
        addedBlocks: ['## my-server\n...'],
        removedNames: []
      }
    },
    ...
  ]
```

Message attachment 出现在 messages 数组的末尾（或近末尾），不影响之前的 prefix cache。

### 9.3 getMcpInstructionsDelta() 算法

`src/utils/mcpInstructionsDelta.ts:55-130`:

```typescript
export function getMcpInstructionsDelta(
  mcpClients: MCPServerConnection[],
  messages: Message[],
  clientSideInstructions: ClientSideInstruction[],
): McpInstructionsDelta | null {

  // Step 1: 扫描历史消息，重建"已公告"的服务器集合
  const announced = new Set<string>()
  for (const msg of messages) {
    if (msg.type !== 'attachment') continue
    if (msg.attachment.type !== 'mcp_instructions_delta') continue
    for (const n of msg.attachment.addedNames) announced.add(n)
    for (const n of msg.attachment.removedNames) announced.delete(n)
  }

  // Step 2: 获取当前已连接且有 instructions 的服务器
  const connected = mcpClients.filter(c => c.type === 'connected')
  const connectedNames = new Set(connected.map(c => c.name))

  // Step 3: 构建指令块（server-authored + client-side 合并）
  const blocks = new Map<string, string>()
  for (const c of connected) {
    if (c.instructions) blocks.set(c.name, `## ${c.name}\n${c.instructions}`)
  }
  for (const ci of clientSideInstructions) {
    if (!connectedNames.has(ci.serverName)) continue
    const existing = blocks.get(ci.serverName)
    blocks.set(ci.serverName,
      existing ? `${existing}\n\n${ci.block}` : `## ${ci.serverName}\n${ci.block}`)
  }

  // Step 4: 计算 diff
  const added = []
  for (const [name, block] of blocks) {
    if (!announced.has(name)) added.push({ name, block })
  }
  const removed = []
  for (const n of announced) {
    if (!connectedNames.has(n)) removed.push(n)
  }

  // Step 5: 无变化则返回 null
  if (added.length === 0 && removed.length === 0) return null

  return {
    addedNames: added.map(a => a.name),
    addedBlocks: added.map(a => a.block),
    removedNames: removed.sort(),
  }
}
```

### 9.4 Before/After 对比

```
场景: 50K token 会话，Turn 5 时 MCP server 连接

=== Before (DANGEROUS mode) ===

Turn 5 API request:
  system: [static(15K)][dynamic-changed(5K)]  ← 变了!
  tools:  [tool schemas(8K)]
  msgs:   [msg0..msg4(22K)]

  cache_read:     15K (只有 static 之前的)
  cache_creation: 35K (dynamic + tools + msgs 全部重算)
  cost: 15K*0.1 + 35K*1.25 = 1.5K + 43.75K = 45.25K token-units

=== After (Delta mode) ===

Turn 5 API request:
  system: [static(15K)][dynamic(5K)]  ← 不变! mcp_instructions=null
  tools:  [tool schemas(8K)]
  msgs:   [msg0..msg4(22K)][delta-attachment(0.5K)]
                            ↑ 新增的，在末尾

  cache_read:     49.5K (之前所有内容)
  cache_creation: 0.5K  (只有 delta attachment)
  cost: 49.5K*0.1 + 0.5K*1.25 = 4.95K + 0.625K = 5.575K token-units

  节省: 45.25K - 5.575K = 39.675K token-units (87.7% 节省!)
```

### 9.5 Feature Gate 和 Rollout

```typescript
// src/utils/mcpInstructionsDelta.ts:37-44
export function isMcpInstructionsDeltaEnabled(): boolean {
  if (isEnvTruthy(process.env.CLAUDE_CODE_MCP_INSTR_DELTA)) return true
  if (isEnvDefinedFalsy(process.env.CLAUDE_CODE_MCP_INSTR_DELTA)) return false
  return (
    process.env.USER_TYPE === 'ant' ||  // ant 用户优先
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_basalt_3kr', false)  // GrowthBook rollout
  )
}
```

[设计决策] 渐进式 rollout：
1. 环境变量 `CLAUDE_CODE_MCP_INSTR_DELTA` 可以强制开/关（本地测试）
2. ant 用户默认开启（内部先行）
3. 外部用户通过 GrowthBook `tengu_basalt_3kr` 特性门控逐步开启

### 9.6 ClientSideInstruction 补充机制

有些 MCP server（如 claude-in-chrome）需要客户端补充 instructions：

```typescript
// src/utils/mcpInstructionsDelta.ts:23-27
export type ClientSideInstruction = {
  serverName: string
  block: string
}
```

这允许客户端为 server 注入额外的上下文信息，server 自身不知道这些信息。server-authored 和 client-side instructions 会合并：

```typescript
// 合并逻辑
for (const ci of clientSideInstructions) {
  if (!connectedNames.has(ci.serverName)) continue
  const existing = blocks.get(ci.serverName)
  blocks.set(ci.serverName,
    existing ? `${existing}\n\n${ci.block}`
             : `## ${ci.serverName}\n${ci.block}`)
}
```

---

## 10. Cache 与 Context 处理的关系

### 10.1 正常对话：前缀稳定

```
Turn 1: [sys][tools][msg0]                → 全部 cache_creation
Turn 2: [sys][tools][msg0][msg1][msg2]    → msg0 前 cache_read, msg1-2 creation
Turn 3: [sys][tools][msg0..4][msg5]       → msg0-4 前 cache_read, msg5 creation

特征: prefix 只增不变，cache 命中率随对话深度单调递增
```

### 10.2 compact 后：100% Cache Miss

compact 将所有 messages 替换为一条摘要消息：

```
Before compact:
  [sys][tools][msg0][msg1]...[msg20]
  → Turn 21 的 cache_read 可能有 100K+ tokens

After compact:
  [sys][tools][summary_msg]
  → summary_msg 是全新内容
  → 与之前的 cache entry 不匹配
  → 100% cache miss

解决方案: notifyCompaction() 重置检测 baseline
```

```typescript
// src/services/api/promptCacheBreakDetection.ts:689-698
export function notifyCompaction(
  querySource: QuerySource,
  agentId?: AgentId,
): void {
  const key = getTrackingKey(querySource, agentId)
  const state = key ? previousStateBySource.get(key) : undefined
  if (state) {
    state.prevCacheReadTokens = null  // ← 重置! 下次不再对比
  }
}
```

### 10.3 Agent 列表变化的优化

[可借鉴] 当 MCP 工具加载后，available agents 列表会变化。如果 agent 列表在 tool schema 里，每次变化都破坏 tool schemas 的缓存。

解决方案：`shouldInjectAgentListInMessages()`

```typescript
// src/tools/AgentTool/prompt.ts:59-63
export function shouldInjectAgentListInMessages(): boolean {
  if (isEnvTruthy(process.env.CLAUDE_CODE_AGENT_LIST_IN_MESSAGES)) return true
  if (isEnvDefinedFalsy(process.env.CLAUDE_CODE_AGENT_LIST_IN_MESSAGES))
    return false
  // ... GrowthBook gate
}
```

开启后：
- AgentTool 的 tool schema 变为固定文本："Available agent types are listed in `<system-reminder>` messages in the conversation."
- 实际的 agent 列表通过 message attachment 注入

```
之前:
  tool schema: "AgentTool: ... Available agents: default, explore, custom:qa-agent"
  → MCP 加载新工具 → agent 列表变 → tool schema 变 → cache miss

之后:
  tool schema: "AgentTool: ... Available agent types are listed in system-reminder"
  → MCP 加载新工具 → agent 列表在 message attachment 中变化
  → tool schema 不变 → cache hit!

效果: 节省了 fleet 10.2% 的 cache_creation tokens
```

---

## 11. 完整 Cache 全景图

一次 API 请求中所有 cache breakpoint 的位置：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        API Request Body                             │
│                                                                     │
│  ┌─── system ──────────────────────────────────────────────────┐    │
│  │                                                             │    │
│  │  [Block 0] attribution header                               │    │
│  │    → cacheScope: null (不标记)                               │    │
│  │                                                             │    │
│  │  [Block 1] CLI prefix ("You are Claude Code...")            │    │
│  │    → cacheScope: null (不标记)                               │    │
│  │                                                             │    │
│  │  [Block 2] Static content (intro+system+tasks+actions+      │    │
│  │            tools+tone+efficiency, ~15K tokens)              │    │
│  │    → cache_control: { type:'ephemeral', scope:'global',     │    │
│  │                        ttl:'1h' }              ◄── BP #1    │    │
│  │                                                             │    │
│  │  [Block 3] Dynamic content (session+memory+env+lang+        │    │
│  │            mcp_instr(null when delta mode)+...)             │    │
│  │    → cacheScope: null (不标记)                               │    │
│  │                                                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─── tools ───────────────────────────────────────────────────┐    │
│  │                                                             │    │
│  │  [Tool 0] BashTool                                          │    │
│  │  [Tool 1] FileEditTool                                      │    │
│  │  [Tool 2] FileReadTool                                      │    │
│  │  ...                                                        │    │
│  │  [Tool N] GrepTool                                          │    │
│  │    → cache_control: { type:'ephemeral', ttl:'1h' }          │    │
│  │      (如果有 MCP tools → 没有 global scope)    ◄── BP #2    │    │
│  │                                                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─── messages ────────────────────────────────────────────────┐    │
│  │                                                             │    │
│  │  [msg 0] user: "请修复这个 bug"                              │    │
│  │  [msg 1] assistant: "让我看看代码..."                        │    │
│  │  [msg 2] user: { tool_result: "file content..." }           │    │
│  │  [msg 3] assistant: "找到了，问题在..."                      │    │
│  │  [msg 4] user: { tool_result: "修改成功" }                   │    │
│  │  [msg 5] assistant: "已经修复，还需要什么?"                   │    │
│  │  [msg 6] user: "帮我跑一下测试"                              │    │
│  │    → cache_control: { type:'ephemeral', ttl:'1h' }          │    │
│  │                                               ◄── BP #3    │    │
│  │                                (只有这一个 message marker)   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─── beta headers (request level) ───────────────────────────┐    │
│  │                                                             │    │
│  │  interleaved-thinking-2025-04-14                            │    │
│  │  prompt-caching-scope-2025-03-11                            │    │
│  │  [latched] fast-mode-2025-04-18                             │    │
│  │  [latched] afk-mode-2025-01-09                              │    │
│  │  [latched] cached-mc-2025-04-01                             │    │
│  │                                                             │    │
│  │  → 全部参与 cache key                                       │    │
│  │  → latched headers 保证会话内不变                            │    │
│  │                                                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─── other cache key participants ───────────────────────────┐    │
│  │  model: "claude-opus-4-6-20250430"                          │    │
│  │  thinking_config: { type: 'enabled', budget_tokens: 10000 } │    │
│  │  output_config: { effort: 'high' }                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Cache 匹配过程 (Turn N+1 vs Turn N):

  system prompt:   完全一样 → prefix cache hit ✓
  tool schemas:    完全一样 → prefix cache hit ✓ (累计)
  msg[0]..msg[6]:  完全一样 → prefix cache hit ✓ (累计)
  msg[7] (新消息): 新内容   → cache_creation

  结果: 只有 msg[7] 的 tokens 需要 cache_creation
        其余全部 cache_read
```

---

## 12. 设计决策总结

### 12.1 核心原则

[设计决策] Claude Code 的 cache 优化遵循一个核心原则：**把不变的东西放前面，把会变的东西移到末尾或移出 cache key 路径**。

具体实践：

| 技术 | 不变的放前面 | 会变的处理方式 |
|------|------------|--------------|
| System prompt | 静态内容 → global scope | 动态内容 → 不标 cache |
| Tool schemas | 内置工具 → 稳定 | MCP 工具 → deferred loading |
| Agent list | 固定描述文本 | 实际列表 → message attachment |
| MCP instructions | 返回 null | 实际指令 → message attachment (delta) |
| Beta headers | 一旦发送就锁定 | 行为由其他字段控制 |
| TTL eligibility | session 内 latch | 不跟随 overage 状态变化 |

### 12.2 权衡表

| 权衡点 | 选择 | 代价 | 收益 |
|--------|------|------|------|
| 一个 vs 多个 message cache marker | 一个 | 无法缓存中间"检查点" | Mycro KV page 内存效率高 |
| Latch header vs 精确控制 | Latch | 可能发送不需要的 header | 防止 ~50-70K tokens cache bust |
| Global scope vs per-user | Global for static | 所有用户看到相同 static prompt | 全球共享 cache_creation 成本 |
| Delta mode vs inline | Delta for MCP | 增加 attachment 处理复杂度 | 消除 MCP 连接导致的 cache break |
| Latch TTL eligibility | Session-stable | 超额用户可能多用 1h cache | 防止 mid-session TTL 翻转 |

---

## 13. 常见误区

### 13.1 "客户端可以控制 cache hit/miss"

错。Cache 完全是 server-side 行为。客户端能做的只有：
- 标记 `cache_control` breakpoint（告诉 server 哪些位置可以作为 cache prefix 端点）
- 保持请求内容稳定（让 server 有可能匹配到已有的 cache entry）

### 13.2 "多标几个 cache_control 点，cache 命中率更高"

错。更多的 marker 意味着更多的 KV page 被保护不被回收，浪费 server-side 内存。Claude Code 的经验是**一个 message marker 足矣**。

### 13.3 "System prompt 的 global scope 是给当前用户的全部请求共享"

不完全对。`scope: 'global'` 是**全球所有用户共享**。这意味着全球第一个发送相同 static system prompt 的 Claude Code 用户支付 cache_creation，后续所有人都可以 cache_read。

### 13.4 "cache_control 参与 cache key 的 value 部分"

是的!`cache_control` 的 scope 和 TTL 参与 cache key 计算。如果 scope 从 `'global'` 变成 `'org'`，或 TTL 从 `'1h'` 变成默认，cache key 就不同了。这就是为什么需要 `cacheControlHash` 单独追踪。

### 13.5 "compact 后 cache 只是暂时丢失"

对。compact 后的下一次请求会 100% cache miss（cache_creation），但之后的请求会从新的 prefix 开始累积 cache_read。关键是 `notifyCompaction()` 要及时重置 baseline，避免误报。

---

## 14. 跨模块联系

### 14.1 与 Part 1 (Conversation Engine) 的关系

Conversation Engine 的 message 管理直接影响 cache：
- message 顺序必须稳定（前缀匹配）
- `ensureToolResultPairing()` 修复 tool_use/tool_result 配对，不能改变已有 messages 的内容
- `addCacheBreakpoints()` 在 message 序列化的最后一步执行

### 14.2 与 Part 2 (Context Management) 的关系

Context 处理是 cache 的最大威胁：
- auto-compact 替换 messages → cache miss（但 `notifyCompaction()` 处理了）
- micro-compact 使用 cache_edits → 预期的 cache drop（`notifyCacheDeletion()` 处理了）
- context window 溢出时的 truncation → 如果 truncate 了 prefix 的内容，cache miss

### 14.3 与 MCP 子系统的关系

MCP 是 cache 稳定性的最大挑战源：
- MCP tool schemas 在 tool array 中 → 影响 tools cache
- MCP instructions 在 system prompt 中 → 影响 system cache
- deferred loading + delta mode 是两个互补的解决方案

### 14.4 与 Agent 子系统的关系

子 agent 的 cache 处理：
- Fork agent 使用 `skipCacheWrite = true` → marker 放在 messages.length - 2
- 每个 agent 有独立的 tracking key (agentId)
- `MAX_TRACKED_SOURCES = 10` 防止内存无限增长

---

## 15. Takeaway

1. **Prompt Cache 是 server-side KV cache，严格前缀匹配**。客户端要做的是保持前缀稳定并标记 breakpoint。

2. **三层策略互补**：
   - Layer 1: System prompt 分区 — global scope 让全球用户共享静态部分
   - Layer 2: Message 单点标记 — 一个 marker 实现效率和内存的最优平衡
   - Layer 3: Beta header latching — 解耦 header 和 behavior，保持 cache key 稳定

3. **把变化的东西移到末尾**是核心原则：
   - 动态 system prompt 内容 → 不标 cache（放在 static 之后）
   - MCP instructions → delta mode（移到 message attachment）
   - Agent list → message attachment

4. **Cache break 检测是闭环的**：pre-call 记录状态 + post-call 对比实际 tokens，12 个维度的诊断让问题可追踪、可解释。

5. **Latch 模式是防御性编程的典范**：一旦开启就不关闭（beta headers、TTL eligibility），宁可多发一个不需要的 header，也不要因为翻转丢失 50K+ tokens 的 cache。

---

## 16. 思考题

1. **为什么 `splitSysPromptPrefix()` 不直接在 prompts.ts 里执行，而要在 buildSystemPromptBlocks() 里 lazy 执行？**
   提示：考虑 prompts.ts 的调用者除了 API 请求之外还有谁。

2. **如果 Anthropic 未来支持 "scatter cache"（可以缓存 prefix 中的非连续片段），Claude Code 的三层策略需要怎么改？**
   提示：思考哪些限制会被解除。

3. **Delta mode 有一个隐含假设："MCP server 的 instructions 在连接生命周期内是 immutable 的"。如果这个假设被打破，需要什么改动？**
   提示：看 `mcpInstructionsDelta.ts:99-104` 的注释。

4. **为什么 `notifyCompaction()` 将 `prevCacheReadTokens` 设为 `null` 而不是 `0`？两者的行为有什么不同？**
   提示：看 `checkResponseForCacheBreak()` 中对 `prevCacheRead === null` 的处理。

5. **Beta header latch 的一个边界问题：如果用户在 Turn 1 不需要 fast mode，Turn 2 开启 fast mode（latch），然后 Turn 3 执行 `/compact`。Turn 4 又不需要 fast mode。cache key 会怎样变化？**
   提示：`clearBetaHeaderLatches()` 在 compact 时调用。

---

## 17. 关键源文件索引

| 文件 | 职责 | 关键函数/常量 |
|------|------|--------------|
| `src/utils/api.ts` | System prompt 分区 | `splitSysPromptPrefix()` (L321) |
| `src/services/api/claude.ts` | API 调用 + cache 控制 | `buildSystemPromptBlocks()` (L3213), `addCacheBreakpoints()` (L3070), `getCacheControl()` (L358), `should1hCacheTTL()` (L393), beta latch (L1405-1440) |
| `src/services/api/promptCacheBreakDetection.ts` | Cache break 检测 | `recordPromptState()` (L247), `checkResponseForCacheBreak()` (L437), `notifyCompaction()` (L689), `notifyCacheDeletion()` (L673) |
| `src/constants/prompts.ts` | System prompt 构建 | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` (L114), `defaultSystemPrompt()` (L460) |
| `src/constants/systemPromptSections.ts` | Section 缓存机制 | `systemPromptSection()`, `DANGEROUS_uncachedSystemPromptSection()`, `resolveSystemPromptSections()`, `clearSystemPromptSections()` |
| `src/utils/mcpInstructionsDelta.ts` | MCP delta mode | `getMcpInstructionsDelta()` (L55), `isMcpInstructionsDeltaEnabled()` (L37), `ClientSideInstruction` type |
| `src/utils/betas.ts` | Beta header 管理 | `shouldUseGlobalCacheScope()` (L227) |
| `src/bootstrap/state.ts` | Latch 状态存储 | `get/setAfkModeHeaderLatched()`, `get/setFastModeHeaderLatched()`, `get/setCacheEditingHeaderLatched()`, `clearBetaHeaderLatches()` |
| `src/tools/AgentTool/prompt.ts` | Agent list 优化 | `shouldInjectAgentListInMessages()` (L59) |
| `src/services/compact/compact.ts` | Compact 与 cache 协调 | `notifyCompaction()` 调用 (L699, L1048) |
| `src/services/compact/autoCompact.ts` | Auto-compact 与 cache 协调 | `notifyCompaction()` 调用 (L303) |

---

> **Next**: Part 4 将深入 MCP (Model Context Protocol) 集成系统 — Claude Code 如何成为一个可扩展的 AI agent 平台。
