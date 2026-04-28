---
name: pi-subagents
description: |
  将工作委托给内置或自定义子代理，支持单代理、链式、并行、异步、分叉上下文和
  对讲机协调工作流。用于顾问式审查、实现交接，以及单代理应保持控制而其他代理
  提供上下文、规划或执行的多步骤任务。
---

# Pi 子代理

当你需要启动专门的子代理、将多个代理组合成工作流，或按需创建/编辑代理和链时，使用此技能。

## 何时使用

- **顾问式审查**：使用全新上下文的 `reviewer` 代理进行对抗性代码审查，或在需要审查继承决策和漂移时分叉到 `oracle`
- **实现交接**：先让 `oracle` 提供建议，然后在方向获批后让 `oracle-executor` 或 `worker` 执行
- **侦察和规划**：使用 `scout` 或 `context-builder`，然后使用 `planner`
- **并行探索**：并发运行多个无冲突的任务
- **长时间运行的工作**：启动异步/后台运行，稍后检查
- **子代理控制**：关注需要关注的信号，仅当委托运行真正被阻塞时才进行软中断
- **代理编写**：为项目创建、更新或覆盖代理和链

## 工具 vs 斜杠命令

代理可以直接使用 `subagent(...)` 工具进行执行、管理、状态查看和控制。
人类通常使用斜杠命令层：

- `/run` — 启动单个代理
- `/chain` — 启动链式步骤
- `/parallel` — 启动顶层并行任务
- `/agents` — 打开代理管理器 TUI
- `/run-chain` — 启动保存的 `.chain.md` 工作流
- `/subagents-status` — 检查活动/最近的异步运行
- `/subagents-doctor` — 诊断设置、发现、异步路径和对讲机桥接状态

编写代理逻辑时优先使用工具。引导人类进行交互式流程时优先使用斜杠命令。

还提供打包的提示快捷方式用于可重复的工作流：
- `/parallel-review` — 具有不同审查角度的全新上下文审查员，然后综合
- `/parallel-research` — 结合 `researcher` 和 `scout` 获取外部证据和本地代码上下文
- `/gather-context-and-clarify` — 先进行侦察/研究，然后使用 `interview` 向用户询问澄清问题
- `/oracle-executor` — 将明确批准的实现任务发送给 `oracle-executor`

## 内置代理

内置代理以最低优先级加载。项目代理覆盖用户代理，用户/项目代理覆盖同名内置代理。

| 代理 | 用途 | 模型 | 典型输出/角色 |
|------|------|------|--------------|
| `scout` | 快速代码库侦察 | `openai-codex/gpt-5.4-mini` | 写入 `context.md` 交接材料 |
| `planner` | 创建实现计划 | `openai-codex/gpt-5.5` | 写入 `plan.md` |
| `worker` | 通用实现 | `openai-codex/gpt-5.5` | 直接编辑代码 |
| `reviewer` | 审查和修复专家 | `openai-codex/gpt-5.5` | 可以编辑/修复审查的代码 |
| `context-builder` | 需求/代码库交接构建器 | `openai-codex/gpt-5.5` | 写入结构化上下文文件 |
| `researcher` | 网络研究简报生成器 | `openai-codex/gpt-5.5` | 写入 `research.md` |
| `delegate` | 轻量级通用委托 | 继承父模型 | 无固定输出；通用委托工作 |
| `oracle` | 决策一致性顾问审查 | `openai-codex/gpt-5.5` | 顾问审查、对讲机协调 |
| `oracle-executor` | 批准后执行 | `openai-codex/gpt-5.5` | 批准后单作者实现 |

只需小调整时，优先覆盖内置默认值，而不是复制完整的代理文件。

对于单次运行，使用内联配置：

```text
/run reviewer[model=anthropic/claude-sonnet-4] "Review this diff"
```

对于持久调整，优先使用 `/agents`：选择内置，按 `e`，更改模型或其他字段，然后保存为用户或项目覆盖。用户覆盖全局生效。项目覆盖仅在该仓库生效，并优先于用户覆盖。

设置位置：
- 用户范围：`~/.pi/agent/settings.json`
- 项目范围：`.pi/settings.json`

直接设置示例：

```json
{
  "subagents": {
    "agentOverrides": {
      "reviewer": {
        "model": "anthropic/claude-sonnet-4",
        "thinking": "high",
        "fallbackModels": ["openai/gpt-5-mini"]
      }
    }
  }
}
```

有用的覆盖字段：`model`、`fallbackModels`、`thinking`、
`systemPromptMode`、`inheritProjectContext`、`inheritSkills`、`disabled`、
`skills`、`tools` 和 `systemPrompt`。仅当你想要一个显著不同的代理时，才创建同名用户或项目代理。

## 发现和作用域规则

代理文件可以存放在：
- `~/.pi/agent/agents/*.md` — 用户范围
- `.pi/agents/*.md` — 规范项目范围
- 遗留 `.agents/*.md` — 仍读取以保持兼容性，但 `.pi/agents/` 在冲突时获胜

链文件存放在：
- `~/.pi/agent/agents/*.chain.md`
- `.pi/agents/*.chain.md`
- 遗留 `.agents/*.chain.md`

优先级为：
1. 项目范围
2. 用户范围
3. 内置代理

## 运行子代理

### 单个代理

```typescript
subagent({
  agent: "oracle",
  task: "Review my current direction and challenge assumptions."
})
```

### 分叉上下文

```typescript
subagent({
  agent: "oracle",
  task: "Review my current direction and challenge assumptions.",
  context: "fork"
})
```

`context: "fork"` 从当前持久化的父会话创建分支子会话。它**不会**创建全新的最小审查上下文或将历史过滤到仅相关部分。当你想要一个独立的审查或执行线程但仍能引用父会话历史时使用它。

### 并行执行

```typescript
subagent({
  tasks: [
    { agent: "scout", task: "Explore the auth module" },
    { agent: "reviewer", task: "Review the API client" }
  ]
})
```

顶层并行任务可以覆盖每项任务的行为：

```typescript
subagent({
  tasks: [
    { agent: "scout", task: "Map auth", output: "auth-context.md", progress: true },
    { agent: "researcher", task: "Research OAuth best practices", output: "oauth-research.md" },
    { agent: "reviewer", task: "Review auth tests", model: "anthropic/claude-sonnet-4" }
  ],
  concurrency: 3
})
```

避免并行任务中的重复输出路径。并发子进程不应写入同一文件。

### 链式执行

```typescript
subagent({
  chain: [
    { agent: "scout", task: "Map the auth flow and summarize key files" },
    { agent: "planner", task: "Create an implementation plan from {previous}" },
    { agent: "worker", task: "Implement the approved plan based on {previous}" }
  ]
})
```

链步骤可以使用模板变量，如 `{task}`、`{previous}` 和
`{chain_dir}`。这是在步骤间传递结构化摘要的主要方式，无需强制每个步骤重新发现所有内容。

### 异步/后台

```typescript
subagent({
  agent: "worker",
  task: "Run the full test suite",
  async: true
})
```

使用 `subagent({ action: "status", id: "..." })`、`subagent({ action: "status" })` 检查活动运行，或使用 `/subagents-status` 斜杠命令检查异步运行。

当设置或子启动看起来有问题时，使用诊断：

```typescript
subagent({ action: "doctor" })
```

人类可以使用 `/subagents-doctor` 获取相同的只读报告。它检查运行时路径、发现计数、异步支持、当前会话上下文和对讲机桥接状态。

### 子代理控制

子代理控制是委托运行的运行时可见性和干预层。它与生命周期状态分开。生命周期状态表明子进程是 `queued`、`running`、`paused`、`complete` 还是 `failed`。活动报告是事实性的：它跟踪最后观察到的活动时间和当前工具（如果已知）。它不假装知道子进程是否真的卡住了。

默认行为是故意保守的。当超过配置的阈值没有观察到活动时，运行会发出 `needs_attention` 控制事件。前台运行可以将其作为 `subagent:control-event` 事件推送，异步运行将其持久化到 `events.jsonl`，以便父跟踪器可以在无需持续手动轮询的情况下显示它。值得通知的控制事件也会插入可见的转录中，以便用户和父代理都能看到，带有主动提示以及具体的 `nudge`、`status` 和 `interrupt` 选项。可见通知每个子运行和注意状态触发一次。

当子进程明显被阻塞或漂移且父进程需要重新获得控制时，使用软中断：

```typescript
subagent({ action: "interrupt" })
```

针对特定可控制运行时传递 `id`：

```typescript
subagent({ action: "interrupt", id: "abc123" })
```

软中断取消当前子进程轮次并使运行暂停。它并不意味着委托任务成功或失败。中断后，决定下一个明确的操作：以更清晰的指令恢复、替换任务、询问用户或停止工作流。

当任务合法地运行较长时间而没有可观察的输出时，可以覆盖每次运行的控制阈值：

```typescript
subagent({
  agent: "worker",
  task: "Run the slow migration test suite",
  control: {
    needsAttentionAfterMs: 300000,
    notifyOn: ["needs_attention"]
  }
})
```

如果运行已有活动的对讲机桥接目标，需要关注的通知也可以为编排器准备一个紧凑的对讲机 ping。当子路由可用时，ping 告诉编排器哪个代理需要关注，并包含用于轻推的确切 `intercom({ action: "send", to: "..." })` 目标。不要虚构目标或在没有桥接时要求子进程自我报告。

## 澄清 TUI

单运行和并行运行支持澄清 TUI，当你想在启动前预览或编辑参数时：

```typescript
subagent({
  agent: "worker",
  task: "Implement feature X",
  clarify: true
})
```

链默认使用澄清模式，除非你明确设置 `clarify: false`。
对于程序化后台启动，使用 `clarify: false, async: true`。

`/agents` 管理器也有启动开关用于分叉上下文、后台执行和 worktree 隔离的并行运行。当引导想要在启动前检查或编辑启动的人类时使用它。

## Worktree 隔离

当多个代理可能并发写入时，使用 worktree 而不是让它们共享一个文件系统视图。

```typescript
subagent({
  tasks: [
    { agent: "worker", task: "Implement feature A" },
    { agent: "worker", task: "Implement feature B" }
  ],
  worktree: true
})
```

`worktree: true` 为每个并行任务提供从 HEAD 分支的自己的 git worktree。这需要干净的 git 状态，主要用于故意并行的写入工作流。如果你想要一个写入线程和几个顾问代理，优先使用单写入模式。

## Oracle 工作流

预期的 oracle 循环是：
1. 主代理分叉到 `oracle`
2. `oracle` 审查方向、漂移、假设和风险
3. `oracle` 可以通过 `intercom` 协调回编排器
4. 主代理决定批准什么方向
5. 只有那时才应该由 `oracle-executor` 实现

```typescript
// 分支线程中的顾问审查
subagent({
  agent: "oracle",
  task: "Review my current direction, challenge assumptions, and propose the best next move.",
  context: "fork"
})

// 明确批准后实现
subagent({
  agent: "oracle-executor",
  task: "Implement the approved approach: ...",
  context: "fork"
})
```

`oracle` 不是 Cognition 文章意义上的全新上下文审查员。它是一个继承父会话历史的分叉顾问线程，并将该历史作为基线契约。

## 子代理 + 对讲机协调

`pi-subagents` 无需 `pi-intercom` 即可工作。当 `pi-intercom` 已安装并启用时，对讲机桥接可以自动为子代理提供返回父会话的私有协调通道。

除非桥接指令提供目标，否则大多数代理不应直接调用 `intercom`。不要虚构目标。使用注入的桥接指令中的目标或从可见的需要关注通知中获取。

在以下情况使用 `intercom`：
- 子代理在决策上受阻
- 子进程需要澄清而不是猜测
- 分离的或异步的子进程需要协调而无需等待正常工具返回流程
- 顾问代理被明确要求发送简洁的进度更新

消息约定：
- `ask` 表示子进程需要父会话的决策或澄清。
- `send` 表示简短的阻塞/进度更新，仅在受阻或明确要求时。
- 例行完成不通过对讲机。正常的子代理结果仍通过 `pi-subagents` 返回。

如果桥接目标可用，子进程可以询问：

```typescript
intercom({
  action: "ask",
  to: "<bridge-provided-target>",
  message: "Should I optimize for readability or performance here?"
})
```

父进程回复：

```typescript
intercom({ action: "reply", message: "Optimize for readability." })
```

或首先检查未解决的询问：

```typescript
intercom({ action: "pending" })
```

如果对讲机消息不显示，运行 `subagent({ action: "doctor" })` 或 `/subagents-doctor`。

## 管理模式

`subagent(...)` 工具也支持管理操作。

### 列出可用代理和链

```typescript
subagent({ action: "list" })
```

### 创建代理

```typescript
subagent({
  action: "create",
  config: {
    name: "my-agent",
    description: "项目特定的实现助手",
    systemPrompt: "在此处填写你的系统提示。",
    systemPromptMode: "replace",
    model: "openai-codex/gpt-5.4",
    tools: "read,grep,find,ls,bash"
  }
})
```

### 更新代理

```typescript
subagent({
  action: "update",
  agent: "my-agent",
  config: {
    thinking: "high"
  }
})
```

### 删除代理

```typescript
subagent({ action: "delete", agent: "my-agent" })
```

当系统需要按需创建或编辑子代理而无需进入原始文件编辑时，使用管理操作。

管理操作创建或更新用户/项目代理文件。对于小的内置更改（如模型交换），优先使用 `/agents` 内置覆盖或设置中的 `subagents.agentOverrides`。

## 通过文件创建和编辑代理

最小的代理文件如下所示：

```markdown
---
name: my-agent
description: 此代理的作用
model: openai-codex/gpt-5.4
thinking: high
tools: read, grep, find, ls, bash
systemPromptMode: replace
inheritProjectContext: true
inheritSkills: false
---

在此处填写你的系统提示。
```

这只是一个起点。常见的可选字段包括：
- `defaultProgress`
- `defaultReads`
- `output`
- `fallbackModels`
- `maxSubagentDepth`

对于许多自定义，设置中的内置覆盖比复制完整的内置文件更简便。

## 提示模板集成

该包包含常见工作流的提示快捷方式：`/parallel-review`、
`/parallel-research`、`/gather-context-and-clarify` 和 `/oracle-executor`。
当用户想要可重复的审查、研究、澄清或批准的执行模式时使用它们。

如果安装了 `pi-prompt-template-model`，额外的用户提示模板可以委托给
`pi-subagents`。当斜杠命令应始终通过特定代理或分叉上下文运行时，这很有用。

## 重要约束

- **分叉需要持久化的父会话。** 如果当前会话没有持久化的会话文件，分叉运行将失败。
- **分叉运行继承父历史。** 它们是分支线程，不是全新的过滤上下文。对于对抗性审查员使用全新上下文，除非用户明确要求分叉上下文。
- **默认子代理嵌套深度为 2。** 除非另行配置，否则更深层的递归委托被阻止。
- **关注信号不是生命周期状态。** `needs_attention` 表示超过配置的阈值未观察到活动。`paused` 表示子进程轮次被故意中断或正在等待方向；它与 `failed` 不同。
- **对讲机询问是阻塞的。** 会话一次只能维护一个待处理的出站询问等待状态。
- **保持对话权限清晰。** 顾问子代理不应默默地成为第二决策者。

## 最佳实践

### 默认保持单线程写入

一个强大的模式是一个主决策者加上围绕它的顾问/研究/审查子代理。使用 `oracle` 获取建议，使用 `oracle-executor` 或 `worker` 进行实际的写入路径。

### 使用分叉进行分支顾问或执行线程

分叉运行很有用，当子进程应在单独的线程中推理，同时仍然继承父进程累积的上下文时。它们对 `oracle` 特别有用，它审计继承的决策和漂移。对于对抗性代码审查，优先使用直接检查仓库和差异的全新上下文审查员，除非用户明确要求分叉上下文。

### 优先选择狭窄任务

给子代理具体任务而不是模糊的命令。
`Review auth.ts for null-check gaps` 比 `Review everything` 效果更好。

### 向上升级决策

如果子代理遇到未经批准的产品、架构或范围选择，它应通过对讲机协调返回，而不是独自决定。

### 仅在明确的控制信号时干预

当委托运行发出 `needs_attention` 或人类要求你重新获得控制时，主动使用子代理控制。不要仅因为子进程短暂没有输出就中断。长时间的工具调用、测试运行或模型推理期间，静默是正常的。

### 有意义地命名会话

使用 `/name` 使对讲机定位保持稳定。

## 常见工作流

### 侦察 → 规划 → 实现

```typescript
subagent({
  chain: [
    { agent: "scout", task: "Map the auth flow and summarize relevant files" },
    { agent: "planner", task: "Plan the migration from {previous}" },
    { agent: "worker", task: "Implement the approved plan from {previous}" }
  ]
})
```

### 审查循环

```typescript
subagent({ agent: "worker", task: "Add retry logic to the API client." })
subagent({
  agent: "reviewer",
  task: "Review the retry logic implementation. Inspect the repo and current diff directly. Look for edge cases and race conditions."
})
```

### 并行无冲突分析

```typescript
subagent({
  tasks: [
    { agent: "scout", task: "Audit frontend auth flow" },
    { agent: "researcher", task: "Research current retry/backoff best practices" }
  ]
})
```

### 保存的链

```text
/run-chain review-chain -- review this branch
```

当用户想要可重复的多代理流而无需每次都重写链时，使用保存的 `.chain.md` 工作流。

## 错误处理

**"Unknown agent"（未知代理）**
```typescript
subagent({ action: "list" })
// 检查可用代理和链，然后确认作用域/优先级。
```

**设置、发现或对讲机混乱**
```typescript
subagent({ action: "doctor" })
// 检查运行时路径、异步支持、发现计数、当前会话和对讲机桥接状态。
```

**"Max subagent depth exceeded"（超过最大子代理深度）**
```typescript
// 扁平化工作流或在配置中提高 maxSubagentDepth。
```

**"Session manager did not return a session file"（会话管理器未返回会话文件）**
```typescript
// 在使用 context: "fork" 之前持久化当前会话。
```

**对讲机 "Already waiting for a reply"（已在等待回复）**
```typescript
// 在开始另一个之前解决当前出站询问。
```

**并行输出路径冲突**
```typescript
// 为每个并行任务提供不同的输出路径，或为不需要的任务禁用输出。
```

**Worktree 启动失败**
```typescript
// 确保 git 工作树干净，并且任务 cwd 覆盖与共享 cwd 匹配。
```

**子进程在启动前失败**
```typescript
// 检查 /subagents-status 详细信息、产物元数据/输出日志，并运行 doctor。扩展加载器错误通常出现在子进程输出日志中。
```
