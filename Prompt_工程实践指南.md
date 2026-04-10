# Prompt 工程实践指南：基于 Claude Code CLI 源码

> 本指南从 Claude Code CLI (v2.1.88) 源码中提炼出的 prompt 工程最佳实践。

---

## 第一章：系统提示词的分层架构

### 1.1 分段设计原则

Claude Code 的系统提示词不是一大段文本，而是按功能职责拆分成独立模块：

```
┌─────────────────────────────────────────────────────┐
│  # Identity & Environment (静态，可缓存)              │
│  "You are Claude Code..." + CWD + Date              │
├─────────────────────────────────────────────────────┤
│  # System (静态)                                     │
│  工具输出机制、权限模式、系统提醒处理                   │
├─────────────────────────────────────────────────────┤
│  # Doing tasks (静态)                                │
│  任务执行指导、代码风格、安全编码                       │
├─────────────────────────────────────────────────────┤
│  # Using your tools (静态)                           │
│  工具使用策略、并行调用指导                            │
├─────────────────────────────────────────────────────┤
│  # Actions with care (静态)                          │
│  危险操作确认策略                                     │
├─────────────────────────────────────────────────────┤
│  ★ SYSTEM_PROMPT_DYNAMIC_BOUNDARY ★                 │
│  ─────────────────────────────────────────────────── │
│  # Session-specific guidance (动态)                  │
│  当前可用工具、skills、用户权限状态                     │
│  Memory 加载、MCP 指令                               │
│  Environment 信息                                    │
└─────────────────────────────────────────────────────┘
```

**关键洞察**：边界之上的内容可以跨用户缓存（`scope: 'global'`），边界之下的内容每会话动态生成。

### 1.2 源码实现参考

```typescript
// src/constants/prompts.ts
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'

// 静态部分 - 缓存友好
const staticSections = [
  getSimpleIntroSection(),
  getSimpleSystemSection(),
  getSimpleDoingTasksSection(),
  getActionsSection(),
  getUsingYourToolsSection(enabledTools),
  SYSTEM_PROMPT_DYNAMIC_BOUNDARY,
]

// 动态部分 - 每次计算
const dynamicSections = [
  systemPromptSection('session_guidance', () => getSessionSpecificGuidanceSection(...)),
  systemPromptSection('memory', () => loadMemoryPrompt()),
  systemPromptSection('env_info', () => computeEnvInfo()),
]
```

---

## 第二章：行为指导的编码方法

### 2.1 用否定约束定义边界

Claude Code 大量使用"不要做 X"来定义行为边界，比正面指导更有效：

```markdown
# Doing tasks
- Don't add features, refactor code, or make "improvements" beyond what was asked.
- Don't create helpers, utilities, or abstractions for one-time operations.
- Don't design for hypothetical future requirements.
- Don't add error handling for scenarios that can't happen.
- Don't use backwards-compatibility hacks like renaming unused _vars.
```

**原则**：告诉模型"边界之外是什么"，比只说"做什么"更能防止越界行为。

### 2.2 分层细化指导

每个主题用子要点展开，形成层级结构：

```markdown
# Doing tasks
 - The user will primarily request you to perform software engineering tasks...
 - Do not create files unless they're absolutely necessary...
   - Don't add features, refactor code, or make "improvements"...
     - A bug fix doesn't need surrounding code cleaned up
     - A simple feature doesn't need extra configurability
   - Don't create helpers, utilities, or abstractions for one-time operations...
 - Be careful not to introduce security vulnerabilities...
```

源码实现：
```typescript
// src/constants/prompts.ts
export function prependBullets(items: Array<string | string[]>): string[] {
  return items.flatMap(item =>
    Array.isArray(item)
      ? item.map(subitem => `  - ${subitem}`)  // 子层级缩进
      : [` - ${item}`]                          // 主层级
  )
}
```

### 2.3 IMPORTANT 标记关键约束

用 `IMPORTANT:` 前缀标记关键约束，提高模型注意权重：

```markdown
IMPORTANT: You must NEVER generate or guess URLs for the user unless...
IMPORTANT: Go straight to the point. Try the simplest approach first...
IMPORTANT: Avoid using this tool to run `find`, `grep`, `cat`...
```

### 2.4 安全指导的平衡写法

安全指导需要平衡"拒绝有害请求"与"支持合法用途"：

```typescript
// src/constants/cyberRiskInstruction.ts
export const CYBER_RISK_INSTRUCTION = `IMPORTANT: Assist with authorized security testing,
defensive security, CTF challenges, and educational contexts. Refuse requests for
destructive techniques, DoS attacks, mass targeting, supply chain compromise, or
detection evasion for malicious purposes. Dual-use security tools (C2 frameworks,
credential testing, exploit development) require clear authorization context:
pentesting engagements, CTF competitions, security research, or defensive use cases.`
```

**写法要点**：
- 先说明支持什么（授权测试、防御安全、CTF）
- 再说明拒绝什么（DoS、大规模攻击、供应链攻击）
- 明确边界条件（需要授权上下文）

---

## 第三章：工具描述的设计模式

### 3.1 工具描述的核心结构

每个工具描述遵循固定结构：

```
1. 功能概述（一句话）
2. 前提假设（用户输入假设）
3. 参数说明（用法指导）
4. 行为指导（何时用/何时不用）
5. 示例用法
```

### 3.2 BashTool 描述设计

```typescript
// src/tools/BashTool/prompt.ts
export function getSimplePrompt(): string {
  return [
    'Executes a given bash command and returns its output.',  // 功能概述
    '',
    "The working directory persists between commands, but shell state does not...",  // 前提
    '',
    `IMPORTANT: Avoid using this tool to run ${avoidCommands} commands...`,  // 行为指导
    '',
    '# Instructions',
    ...prependBullets(instructionItems),  // 详细指导
    getSimpleSandboxSection(),           // 安全边界
    getCommitAndPRInstructions(),        // 特定场景指导
  ].join('\n')
}
```

**关键设计**：
- 明确"优先用专用工具而非 Bash"的行为引导
- 用 `prependBullets` 呈现层级指导
- 将 git 操作单独成节，提供详细流程

### 3.3 工具偏好引导

明确告诉模型"用什么替代什么"：

```markdown
IMPORTANT: Avoid using the Bash tool to run `find`, `grep`, `cat`, `head`, `tail`,
`sed`, `awk`, or `echo` commands, unless explicitly instructed...

Instead, use the appropriate dedicated tool:
- To read files use Read instead of cat, head, tail, or sed
- To edit files use Edit instead of sed or awk
- To search for files use Glob instead of find or ls
- To search the content of files, use Grep instead of grep or rg
```

### 3.4 工具并行调用指导

```markdown
You can call multiple tools in a single response. If you intend to call multiple
tools and there are no dependencies between them, make all independent tool calls
in parallel. Maximize use of parallel tool calls where possible to increase efficiency.

However, if some tool calls depend on previous calls to inform dependent values,
do NOT call these tools in parallel and instead call them sequentially.
```

---

## 第四章：Prompt 缓存策略

### 4.1 缓存分段原理

Anthropic 的 Prompt Caching 支持 `ephemeral`（会话级）和 `global`（跨用户级）缓存。

```typescript
// src/services/api/claude.ts
export function buildSystemPromptBlocks(systemPrompt, enablePromptCaching) {
  return splitSysPromptPrefix(systemPrompt).map(block => {
    return {
      type: 'text',
      text: block.text,
      ...(enablePromptCaching && block.cacheScope !== null && {
        cache_control: getCacheControl({ scope: block.cacheScope }),
      }),
    }
  })
}
```

### 4.2 静态/动态边界设计

**静态部分设计原则**：
- 不包含用户特定信息（路径、用户ID、权限）
- 不包含会话特定信息（可用工具列表、MCP 状态）
- 不包含时间相关信息（当前日期用"今天"而非具体日期）

**动态部分设计原则**：
- 放在边界之后
- 用 `systemPromptSection` 包装，支持缓存复用

```typescript
// src/constants/systemPromptSections.ts
export function systemPromptSection(name: string, compute: ComputeFn) {
  return { name, compute, cacheBreak: false }  // 会话内缓存
}

export function DANGEROUS_uncachedSystemPromptSection(
  name: string,
  compute: ComputeFn,
  _reason: string,
) {
  return { name, compute, cacheBreak: true }  // 每轮重算
}
```

### 4.3 工具排序稳定化

工具列表按字母排序，保证缓存稳定性：

```typescript
// src/tools.ts
const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
return uniqBy(
  [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
  'name',
)
```

**原因**：如果工具顺序变化，会 bust 整个 prompt cache。

### 4.4 路径标准化

将用户特定路径标准化为通用符号：

```typescript
// src/tools/BashTool/prompt.ts
const normalizeAllowOnly = (paths: string[]): string[] =>
  [...new Set(paths)].map(p => (p === claudeTempDir ? '$TMPDIR' : p))
```

**效果**：`/private/tmp/claude-1001/` → `$TMPDIR`，跨用户缓存友好。

---

## 第五章：动态内容注入策略

### 5.1 Attachment vs Inline

Claude Code 将频繁变化的内容从工具描述中移出，改为 attachment 注入：

```typescript
// src/tools/AgentTool/prompt.ts
export function shouldInjectAgentListInMessages(): boolean {
  // 动态 agent list ~10.2% of fleet cache_creation tokens
  // MCP async connect, /reload-plugins 都会改变列表
  // 移到 attachment 避免 bust 工具 schema cache
  return getFeatureValue('tengu_agent_list_attach', false)
}
```

**决策依据**：如果内容变化频率 > 用户请求频率，用 attachment 而非 inline。

### 5.2 System Reminder 注入

动态提醒通过 `<system-reminder>` 标签注入：

```markdown
<system-reminder>
Available agent types:
- test-runner: use after writing code to run tests
- greeting-responder: respond to user greetings with a friendly joke
</system-reminder>
```

**优势**：
- 不修改系统提示词，避免 bust cache
- 模型被引导"这些是系统添加的，与具体消息无关"

---

## 第六章：示例用法设计

### 6.1 示例结构

示例遵循 `<example>` 标签结构，包含 user、assistant、commentary：

```xml
<example>
user: "Please write a function that checks if a number is prime"
assistant: I'm going to use the Write tool to write the following code:
<code>
function isPrime(n) {
  if (n <= 1) return false
  for (let i = 2; i * i <= n; i++) {
    if (n % i === 0) return false
  }
  return true
}
</code>
<commentary>
Since a significant piece of code was written and the task was completed,
now use the test-runner agent to run the tests
</commentary>
assistant: Uses the Agent tool to launch the test-runner agent
</example>
```

### 6.2 Commentary 的作用

commentary 解释"为什么这样做"，帮助模型理解决策逻辑：

```xml
<example>
user: "so is the gate wired up or not"
<commentary>
User asks mid-wait. The audit fork was launched to answer exactly this,
and it hasn't returned. The coordinator does not have this answer.
Give status, not a fabricated result.
</commentary>
assistant: Still waiting on the audit — that's one of the things it's checking.
</example>
```

### 6.3 多示例覆盖边界情况

```xml
<!-- 正常流程示例 -->
<example>
user: "What's left on this branch before we can ship?"
assistant: Agent tool called...
</example>

<!-- 边界情况：等待中 -->
<example>
user: "so is the gate wired up or not"
assistant: Still waiting on the audit...
</example>

<!-- 边界情况：委托理解 -->
<example>
<commentary>
Never delegate understanding. Don't write "based on your findings, fix the bug"...
</commentary>
</example>
```

---

## 第七章：模型特定调整

### 7.1 [MODEL LAUNCH] 标记

源码中使用 `[MODEL LAUNCH]` 标记追踪模型特定调整：

```typescript
// src/constants/prompts.ts
// @[MODEL LAUNCH]: Update the latest frontier model.
const FRONTIER_MODEL_NAME = 'Claude Opus 4.6'

// @[MODEL LAUNCH]: Remove this section when we launch numbat.
// @[MODEL LAUNCH]: capy v8 assertiveness counterweight (PR #24302)
```

**用途**：
- 追踪哪些 prompt 内容是为特定模型设计的
- 新模型发布时知道哪些内容需要调整

### 7.2 模型行为偏差补偿

针对模型特定行为偏差的补偿指导：

```typescript
// 针对 Capybara v8 的 false-claims 问题
...(process.env.USER_TYPE === 'ant' ? [
  `Report outcomes faithfully: if tests fail, say so with the relevant output;
   if you did not run a verification step, say that rather than implying it
   succeeded. Never claim "all tests pass" when output shows failures...`,
] : [])
```

**洞察**：模型能力变化时，prompt 需要相应调整，而非假设模型天然具备某些行为。

---

## 第八章：复杂任务的 Prompt 设计

### 8.1 Agent 工具的 Prompt 设计

Agent 工具需要指导模型"如何写 prompt 给子 agent"：

```markdown
## Writing the prompt

Brief the agent like a smart colleague who just walked into the room —
it hasn't seen this conversation, doesn't know what you've tried...

- Explain what you're trying to accomplish and why.
- Describe what you've already learned or ruled out.
- Give enough context about the surrounding problem...
- If you need a short response, say so ("report in under 200 words").

**Never delegate understanding.** Don't write "based on your findings, fix the bug"
or "based on the research, implement it." Those phrases push synthesis onto the
agent instead of doing it yourself.
```

### 8.2 Fork vs Subagent 区分

```markdown
## When to fork

Fork yourself (omit `subagent_type`) when the intermediate tool output
isn't worth keeping in your context. The criterion is qualitative —
"will I need this output again" — not task size.

- **Research**: fork open-ended questions.
- **Implementation**: prefer to fork implementation work that requires
  more than a couple of edits.

**Don't peek.** The tool result includes an `output_file` path — do not Read
or tail it unless the user explicitly asks for a progress check.
```

### 8.3 Git 操作的详细流程

复杂操作（如 commit、PR）需要分步骤指导：

```markdown
# Committing changes with git

1. Run the following bash commands in parallel:
   - Run a git status command to see all untracked files
   - Run a git diff command to see both staged and unstaged changes
   - Run a git log command to see recent commit messages
2. Analyze all staged changes and draft a commit message:
   - Summarize the nature of the changes...
   - Do not commit files that likely contain secrets...
3. Run the following commands in parallel:
   - Add relevant untracked files to the staging area
   - Create the commit with a message ending with: ...
   - Run git status after the commit completes to verify success
4. If the commit fails due to pre-commit hook: fix the issue and create a NEW commit
```

---

## 第九章：实践建议总结

### 9.1 Prompt 设计清单

| 检查项 | 说明 |
|--------|------|
| 模块化 | 按功能职责拆分，而非单一大段文本 |
| 缓存边界 | 区分静态/动态内容，静态部分可缓存 |
| 行为边界 | 用否定约束定义"不要做什么" |
| 关键标记 | 用 IMPORTANT 标记关键约束 |
| 示例覆盖 | 提供正常流程和边界情况示例 |
| 工具引导 | 明确"用什么替代什么" |
| 安全平衡 | 先说支持什么，再说不支持什么 |
| 排序稳定 | 动态列表排序后注入，保证缓存稳定 |

### 9.2 常见错误

| 错误 | 正确做法 |
|------|----------|
| 单一提示词 | 分模块设计 |
| 包含用户路径 | 标准化为符号 ($TMPDIR) |
| 动态内容 inline | 用 attachment 或 system-reminder |
| 工具列表无序 | 按字母排序 |
| 缺少示例 commentary | 解释决策逻辑 |
| 只说"做什么" | 同时说"不要做什么" |

### 9.3 迭代追踪机制

建立 `[MODEL LAUNCH]` 或类似标记：
- 追踪哪些内容是为特定模型设计的
- 新模型发布时系统化审查需要调整的内容
- 结合 A/B 测试验证调整效果

---

## 附录 A：通用 Prompt 模板速查

### A.1 核心公式

```
好 Prompt = 角色 + 任务 + 上下文 + 约束 + 示例（可选）
```

### A.2 标准模板

```markdown
[角色] 你是一位专业的代码审查员。
[任务] 审查以下 Python 函数的生产安全性。
[上下文] 这是一个处理用户支付的函数，需要 PCI-DSS 合规。
[约束] 指出所有安全隐患，按严重程度排序，附上修复建议。

```python
def process_payment(card_num, amount):
    query = f"SELECT * FROM payments WHERE card='{card_num}'"
    # ...
```
```

### A.3 Few-Shot 模板

```markdown
[任务] 将自然语言转换为 SQL 查询。

示例：
问: 查找所有最近 7 天注册的用户
答: SELECT * FROM users WHERE created_at >= NOW() - INTERVAL '7 days'

问: [你的问题]
答:
```

### A.4 分步推理模板

```markdown
[任务] 分析这个系统为什么会频繁崩溃。

请按以下步骤思考：
1. 首先，列出可能导致崩溃的所有可能原因
2. 其次，根据以下日志信息逐个排除
3. 最后，给出最可能的根因和验证方法

[日志内容...]
```

---

## 附录 B：源码关键文件索引

| 文件 | 内容 |
|------|------|
| `src/constants/prompts.ts` | 系统提示词主模块，分段设计 |
| `src/constants/systemPromptSections.ts` | 缓存分段机制 |
| `src/tools.ts` | 工具列表组装与排序 |
| `src/tools/BashTool/prompt.ts` | Bash 工具描述，git 流程 |
| `src/tools/AgentTool/prompt.ts` | Agent 工具描述，fork/subagent 指导 |
| `src/services/api/claude.ts` | API 调用，缓存控制注入 |
| `src/constants/cyberRiskInstruction.ts` | 安全指导示例 |
| `src/utils/messages.ts` | 消息处理，system-reminder 注入 |

---

## 附录 C：场景速查表

| 场景 | 推荐 Pattern | 示例 Prompt 开头 |
|------|-------------|-----------------|
| 代码调试 | Role + Error + Context + 约束 | "你是一个 Python 专家。这个 `pandas` 操作报 `KeyError`..." |
| 方案设计 | Role + Goal + Constraints + 期望输出 | "作为系统架构师，设计一个支持日活千万的系统..." |
| 学习解释 | Role + 概念 + 深度 + 受众 | "用通俗语言解释什么是微服务，不要超过 200 字" |
| 文案创作 | Role + 风格 + 受众 + 关键点 | "以专业但亲切的风格，写给 25-35 岁程序员..." |
| 代码审查 | Role + Scope + 关注点 | "作为安全专家，审查这个登录模块，重点检查 XSS..." |

---

*本指南基于 Claude Code CLI v2.1.88 源码分析，源码从 npm 发布的 source maps 中恢复，用于研究和学习目的。*