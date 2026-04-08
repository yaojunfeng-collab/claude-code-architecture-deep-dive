# Part 4: Context 生命周期 — 对话越来越长时怎么办？

> 200K token 的上下文窗口听起来很大，但一个深度 coding session 可以轻松用光它。
> Claude Code 设计了五层防线来管理这个问题。

---

## 开场：一个真实场景

你在 Claude Code 里重构一个大型项目。前 20 分钟一切顺利，模型读了 30 个文件，做了 15 次编辑。突然，你发现模型的回答开始变得"糊涂"——它忘记了之前讨论的架构决策，又去读已经读过的文件。

这不是模型"变笨了"。这是 **context window 快满了**，Claude Code 正在悄悄压缩你的对话历史。

这一讲，我们拆解 Claude Code 如何在"保留关键信息"和"控制 token 消耗"之间走钢丝。

---

## 1. 全景：五层防线

```
用量轻 ─────────────────────────────────────────── 用量重
  │                                                    │
  ▼                                                    ▼
Layer 1        Layer 2         Layer 3      Layer 4         Layer 5
Token 预警    MicroCompact    Snip        AutoCompact     ReactiveCompact
UI 提示       单条截断        删旧消息    API 摘要         prompt_too_long 兜底
  │              │               │            │                │
  └── 不影响 ────┴── 局部 ───────┴── 批量 ────┴── 全量 ────────┘
     对话质量       影响小          影响中        影响大          最后手段
```

**[核心概念] 渐进式降级。** 不是一到阈值就做全量压缩。先预警，再局部截断，再批量删除，最后才调 API 做摘要。每一层都尽量减少对对话质量的影响。

---

## 2. Layer 1: Token 计数预警

最轻量的一层——只在 UI 上提示用户。

```
每次 API 响应后：
  totalTokens = response.usage.input_tokens + response.usage.output_tokens
  if (totalTokens > softThreshold) {
    UI 显示："Context usage: 85% ⚠️"
  }
```

**不做任何实际处理。** 只是告诉用户"快满了，考虑 /compact 或 /clear"。

**[设计决策] 为什么要有纯提示层？** 因为用户可能有自己的判断——"我知道快满了，但这个分析马上就完成了，不要现在压缩"。给用户选择权比自动压缩更好。

---

## 3. Layer 2: MicroCompact — 单条截断

### 3.1 触发条件

单条 `tool_result` 的内容超过 `maxResultSizeChars` 时触发。

比如：你让 Claude 读一个 50KB 的日志文件，返回的 tool_result 有 50,000 字符。`FileReadTool` 的 `maxResultSizeChars` 是 `Infinity`（不截断），但 `BashTool` 是 30,000 字符。

### 3.2 处理方式

```
单条 tool_result 超限：
  → 保留头部内容（前 maxResultSizeChars 个字符）
  → 追加 "... [truncated N chars]"
  → 只影响这一条消息，不影响其他消息

如果输出特别大（超 64MB）：
  → 内容写入磁盘文件
  → 给模型一个路径 + 前 1KB 预览
  → <persisted-output path="..." size="...">前 1KB 内容</persisted-output>
```

**[权衡] FileReadTool 的 `maxResultSizeChars = Infinity`。** 看起来危险，但有原因：如果截断文件内容，模型可能基于不完整信息做出错误编辑。宁可占用更多 context，也不要给模型"半截"信息。BashTool 的输出更可能是冗余的（日志、构建输出），截断影响较小。

---

## 4. Layer 3: Snip — 删除旧消息

### 4.1 触发条件

当预估的 token 数量超过某个阈值，但还没到 autoCompact 的触发点。

### 4.2 处理方式

```
删除最旧的一批 assistant + user 消息对
  → adjustIndexToPreserveAPIInvariants()
    确保不拆散 tool_use / tool_result 配对
  → 不调 API，纯本地操作
  → 比 autoCompact 快得多
```

**[核心概念] API 的消息约束：tool_result 必须紧跟在对应的 tool_use 之后。** 如果只删了 assistant 消息（包含 tool_use），留下了 user 消息（包含 tool_result），API 会报错。`adjustIndexToPreserveAPIInvariants()` 确保删除操作不会打破这个约束。

### 4.3 信息损失

Snip 是**有损操作**——被删除的消息永远消失了。模型不知道这些消息曾经存在，可能会：
- 重新读已经读过的文件
- 忘记之前讨论过的架构决策
- 重复已经尝试过的失败方案

**这就是为什么 Layer 4（AutoCompact）更好——它生成摘要保留关键信息。**

---

## 5. Layer 4: AutoCompact — API 摘要

### 5.1 触发条件

```
autoCompactThreshold = effectiveContextWindow - 13,000

其中：
  effectiveContextWindow = min(模型上下文窗口, 用户配置的限制)
  模型默认：200,000 tokens
  13,000 = 留给当前轮次响应的余量

默认：200,000 - 13,000 = 187,000 tokens 触发
```

### 5.2 两条路径（优先 SessionMemory）

```
触发 AutoCompact
    │
    ├─ 优先：尝试 SessionMemory Compact
    │   │
    │   ├─ 成功 → 完成（不需要 API call！）
    │   └─ 失败 → 走传统路径
    │
    └─ 备选：传统 API Compact
        │
        ├─ 成功 → 完成
        └─ 连续失败 3 次 → 熔断，停止尝试
```

### 5.3 SessionMemory Compact（新型，优先使用）

**[可借鉴] 这是一个精彩的优化。**

```
传统 compact：
  把全部历史消息 → 发给 Claude API → 返回摘要 → 替换全部历史
  问题：额外的 API call（慢 + 贵 + 可能失败）

SessionMemory Compact：
  维护一个持续更新的 session_memory.md 文件
  compact 时直接用这个文件 + 保留最近 N 条消息
  不需要额外 API call！
```

`calculateMessagesToKeepIndex()` 决定保留多少：

```
minTokens = 10,000          最少保留这么多 token 的最新消息
minTextBlockMessages = 5    至少保留 5 条消息
maxTokens = 40,000          最多保留这么多 token 的历史
```

**[设计决策] 为什么同时有 minTokens 和 minTextBlockMessages？** minTokens 确保保留足够的信息量，minTextBlockMessages 确保保留完整的对话轮次（一轮可能只有几个 token 的简短消息，但包含重要的决策指令）。

### 5.4 传统 API Compact（备选）

调用 Claude API 生成 **9-section 结构化摘要**：

```
1. Primary Request and Intent       ← 用户最终想要什么
2. Key Technical Concepts           ← 涉及的核心技术
3. Files and Code Sections          ← 读/改了哪些文件
4. Errors and fixes                 ← 遇到的错误和解决方案
5. Problem Solving                  ← 解决问题的过程
6. All user messages                ← 用户所有消息（完整保留！不压缩！）
7. Pending Tasks                    ← 未完成的任务
8. Current Work                     ← 正在做的事
9. Optional Next Step               ← 下一步建议
```

**[核心概念] 第 6 节"All user messages"是完整保留的。** 用户的原始指令是最重要的上下文，不能被摘要掉。模型的中间推理可以压缩，但"用户说了什么"不能丢。

### 5.5 熔断机制

```
连续 autoCompact 失败次数 >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES (3)
  → 熔断：不再尝试 autoCompact
  → 退化到 Layer 5（ReactiveCompact）
```

**[设计决策] 为什么要熔断？** 如果 compact 本身一直失败（比如 API 不可用），反复尝试只会浪费时间和 token。不如放弃，让更暴力的 Layer 5 兜底。

### 5.6 Compact 对 Cache 的冲击

**AutoCompact 是 Cache 的"杀手"。** messages 被替换为摘要后：

```
Compact 前: [system] [tools] [msg1] [msg2] ... [msg50] ← 全部被缓存
Compact 后: [system] [tools] [summary] [msg48] [msg49] [msg50]
                              ↑ 完全不同的内容！

→ 下一次 API call：100% cache miss
→ notifyCompaction() 重置 cache break detection baseline（避免误报）
```

**[权衡] Compact 是必要之恶。** 它破坏了 cache，但避免了更糟糕的结果——prompt_too_long 错误导致对话完全中断。

---

## 6. Layer 5: ReactiveCompact — 最后防线

### 6.1 触发条件

API 返回 `prompt_too_long` 错误时触发。这意味着**前面四层都没能控制住 token 增长**。

### 6.2 处理方式

```
收到 prompt_too_long 错误
    │
    ▼
truncateHeadForPTLRetry()
    │ 删除最旧的消息
    │
    ▼
重试 API call
    │
    ├─ 成功 → 继续
    └─ 还是 prompt_too_long？
         │
         ▼
       继续删 → 重试 → 直到成功或消息删光
```

**特殊情况**：compact 请求本身也可能 prompt_too_long（因为 compact 需要把所有历史发给 API 生成摘要）。这时 ReactiveCompact 会对 compact 请求本身做截断。

---

## 7. 五层防线的协作

一个完整的场景模拟：

```
Turn 1-20:  正常对话，token 消耗 50K
            Layer 1: "Context usage: 25%" — 纯显示

Turn 20-40: 大量文件操作，一条 tool_result 有 50K 字符
            Layer 2: BashTool 输出截断到 30K

Turn 40-80: 对话越来越长，token 到 150K
            Layer 1: "Context usage: 75% ⚠️"

Turn 80:    token 到 187K，触发 autoCompact
            Layer 4: SessionMemory Compact 成功
            → 压缩到 50K，保留最近 40K 消息 + session_memory.md
            → Cache 全部失效（不可避免）

Turn 80-120: 又积累到 187K
            Layer 4: 再次 SessionMemory Compact

Turn 121:   SessionMemory 失败，传统 Compact 也失败（API 超时）
            Layer 4: 失败计数 +1

Turn 122:   再次失败，计数到 3
            Layer 4: 熔断，不再尝试

Turn 123:   API 返回 prompt_too_long
            Layer 5: truncateHeadForPTLRetry()
            → 删最旧的消息 → 重试 → 成功
```

---

## 8. 设计哲学分析

### 8.1 为什么是五层而不是一层自动 Compact？

```
方案 A（简单方案）：超过阈值 → 调 API 压缩 → 完事
  问题1：API 调用有延迟，用户体验差
  问题2：API 可能失败
  问题3：每次压缩都破坏 cache
  问题4：有时候"快满了"不需要压缩，用户马上就完成了

方案 B（五层渐进）：预警 → 局部 → 批量 → 摘要 → 兜底
  Layer 1 不做处理，给用户选择权
  Layer 2 局部影响最小
  Layer 3 不需要 API call，很快
  Layer 4 尽量保留信息完整性
  Layer 5 最后手段，宁可丢信息也不能中断对话
```

**[可借鉴] 渐进式降级是处理资源有限场景的通用模式。** 类比：内存管理中的 minor GC → major GC → OOM killer。

### 8.2 SessionMemory 为什么是更好的方案？

传统 compact 有一个本质问题：**每次压缩都是全量计算**。100 轮对话压缩一次，110 轮又要把 100 轮的摘要 + 10 轮新内容一起重新摘要。

SessionMemory 是**增量维护**的：

```
传统: [全部历史] → API → [摘要]    每次都从头算
新型: [session_memory.md] ← 每轮增量更新   compact 时直接用
```

**[核心概念] 增量 vs 全量是性能的根本差异。** 类似于数据库的 WAL（Write-Ahead Log）vs 每次重建整个表。

### 8.3 用户消息为什么不能压缩？

Compact 的 9-section 模板中，第 6 节明确要求"All user messages"完整保留。

```
可以压缩的：
  模型的中间推理（"让我分析一下..."）
  工具调用的细节（"读了这个文件，内容是..."）

不能压缩的：
  用户说的每一句话
```

原因：用户的指令包含了**意图**。如果把"帮我用 TypeScript strict mode 重写这个模块"压缩成"重写模块"，模型可能忘记 strict mode 的要求。

---

## 9. 常见误区

### 误区 1："Compact 后模型还记得所有内容"

不是。Compact 生成的是摘要，必然有信息损失。模型可能忘记：
- 之前讨论过但没写入摘要的细节
- 某次失败的尝试（摘要可能只记了"遇到错误并解决"）
- 文件的具体内容（摘要只记路径）

### 误区 2："200K context window 足够任何任务"

一次 `git diff` 输出可能就有 50K token。读 5 个大文件就能用光 200K。实际上，深度重构任务经常触发 compact。

### 误区 3："Compact 对 cache 没影响"

**影响巨大。** Compact 后 messages 完全替换，100% cache miss。这是为什么 Claude Code 优先使用 SessionMemory（它保留最近的消息前缀，cache 部分命中）而不是全量 compact。

### 误区 4："Layer 3 (Snip) 和 Layer 4 (AutoCompact) 效果一样"

Snip 是**纯删除**，被删的消息彻底消失。AutoCompact 是**压缩为摘要**，关键信息（用户意图、文件列表、错误记录）被保留在摘要中。信息保真度完全不同。

---

## 10. 跨模块联系

| 涉及 | 详见 |
|------|------|
| Compact 破坏 cache | **Part 3: Cache 优化** — notifyCompaction() |
| 消息压缩链（步骤 3 的预处理） | **Part 2: Prompt 组装** — queryLoop 消息预处理 |
| tool_result 大小控制 | **Part 5: Tool 系统** — maxResultSizeChars |
| 子 Agent 的 maxTurns | **Part 6: Agent 架构** — 防止无限循环 |
| session_memory.md | **Part 8: 记忆系统** — 与 auto memory 的关系 |

---

## 11. 核心 Takeaway

1. **五层渐进式降级**：预警 → 局部截断 → 批量删除 → API 摘要 → 暴力截断。每层尽量减少影响。
2. **SessionMemory 是增量方案**，避免了传统 compact 的全量 API call，更快更省。
3. **Compact 是 Cache 的杀手**——messages 替换后 100% cache miss，这是不可避免的代价。
4. **用户消息永远不压缩**——它们包含意图，是不可替代的信息。
5. **熔断机制防止死循环**——compact 连续失败 3 次后放弃，退化到更暴力的兜底方案。

---

## 12. 思考题

1. **如果你在设计一个 coding agent，你会选择 compact（摘要替换）还是 sliding window（滑动窗口丢弃旧消息）？** 考虑以下场景：用户在第 5 轮定义了一个重要的架构约束，在第 50 轮时还需要遵守它。
2. **SessionMemory 的增量更新是在什么时机触发的？** 如果它在每轮对话后都更新，那更新本身的 token 消耗是否值得？
3. **Compact 后 cache 100% miss。有没有可能设计一种"部分保留 cache"的 compact 方案？** 比如只压缩消息的中间部分，保留头部和尾部？
4. **Layer 3 (Snip) 的"删最旧的消息"策略是否最优？** 是否有更好的策略，比如删除"信息密度最低"的消息？

---

## 关键源文件

| 文件 | 职责 |
|------|------|
| `src/services/compact/autoCompact.ts` | AutoCompact 触发与熔断 |
| `src/services/compact/compact.ts` | 传统摘要 compact |
| `src/services/compact/prompt.ts` | 9-section 摘要模板 |
| `src/services/compact/sessionMemoryCompact.ts` | SessionMemory 增量 compact |
| `src/utils/context.ts` | Context 常量（200K window） |

---

## 关键数值

| 常量 | 值 | 含义 |
|------|-----|------|
| MODEL_CONTEXT_WINDOW_DEFAULT | 200,000 | 默认上下文窗口 |
| AutoCompact 余量 | 13,000 | 留给响应的缓冲 |
| MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES | 3 | 熔断阈值 |
| SessionMemory minTokens | 10,000 | 至少保留 |
| SessionMemory maxTokens | 40,000 | 最多保留 |
| SessionMemory minTextBlockMessages | 5 | 至少保留消息数 |
| COMPACT_MAX_OUTPUT_TOKENS | 20,000 | 摘要最大输出 |
