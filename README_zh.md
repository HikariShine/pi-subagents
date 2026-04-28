<p>
  <img src="banner.png" alt="pi-subagents" width="1100">
</p>

# pi-subagents

`pi-subagents` 让 Pi 将工作委托给专注的子代理。用于代码审查、侦察、实现、并行审计、保存工作流、后台作业，以及任何受益于额外模型视角的任务。

https://github.com/user-attachments/assets/702554ec-faaf-4635-80aa-fb5d6e292fd1

## 安装

```bash
pi install npm:pi-subagents
```

这是唯一必需的步骤。你可以稍后添加可选组件。

## 先试试这个

你不需要创建代理、编写配置或学习斜杠命令。安装后，用自然语言请求 Pi 进行委托：

```text
Use reviewer to review this diff.
```

```text
Ask oracle for a second opinion on my current plan.
```

```text
Use scout to understand this code based on our discussion then ask me clarification questions.
```

```text
Run parallel reviewers: one for correctness, one for tests, and one for unnecessary complexity.
```

这就足以开始了。

## 发生了什么

Pi 是父会话。子代理是具有自己任务的专注子 Pi 会话。

当你请求子代理时，Pi 启动子进程，分配任务，并将结果带回。前台运行在对话中流式显示。后台运行持续工作，可以稍后检查。

安装扩展不会自动在后台启动审查器。它给 Pi 提供了一个委托工具。如果你想让每个实现都被审查，在提示中说明或放在项目指令中：

```text
When you finish implementing, run a reviewer subagent before summarizing.
```

## 好的初始提示

这些覆盖了大部分日常使用：

```text
Ask oracle for a second opinion on my current plan. Challenge assumptions and tell me what I might be missing.
```

```text
Use oracle to help solve this hard bug. Have it inspect the code and propose the best next move before we edit anything.
```

```text
Run parallel reviewers on this diff. I want one focused on correctness, one on tests, and one on unnecessary complexity.
```

```text
Implement this plan with oracle-executor. Afterward, run parallel reviewers, summarize their feedback, and apply the fixes that make sense.
```

```text
Use scout to understand the auth flow, then have planner turn that into an implementation plan.
```

这些是普通的 Pi 请求。Pi 决定是否调用 `subagent`、使用哪个代理、以及链式或并行运行是否有意义。

## 常见工作流

| 想要 | 自然询问 |
|------|---------------|
| 获取第二意见 | "Ask oracle to review this plan and challenge assumptions." |
| 解决难题 | "Use oracle to investigate this bug before we edit." |
| 审查差异 | "Use reviewer to review this diff." |
| 运行并行审查 | "Run reviewers for correctness, tests, and cleanup." |
| 实现后审查 | "Implement this, then review it." |
| 谨慎执行计划 | "Use oracle-executor to implement this plan, then run reviewers and apply the feedback." |
| 规划前先侦察 | "Use scout to inspect the auth flow before planning." |
| 后台运行 | "Run this in the background." |
| 浏览代理 | "Show me the available subagents." |
| 使用保存的工作流 | "Run the review chain on this branch." |
| 查看运行中的工作 | "Show subagent status." |
| 检查设置 | "Check whether subagents are configured correctly." |

扩展附带了可立即使用的内置代理。

## 内置代理通俗解释

| 代理 | 何时使用... |
|-------|--------------------------|
| `scout` | 快速本地代码库侦察：相关文件、入口点、数据流、风险，以及另一个代理应该从哪里开始。 |
| `researcher` | 网络/文档研究并带来源：官方文档、规范、基准测试、最新变更，以及简洁的研究简报。 |
| `planner` | 从现有上下文制定具体的实现计划。它应该阅读并规划，不编辑代码。 |
| `worker` | 通用实现工作。它阅读上下文或计划，编辑文件，并在可能时运行验证。 |
| `reviewer` | 代码审查和小修复。它根据任务/计划、测试、边界情况和简洁性检查实现。 |
| `context-builder` | 规划前更强的准备阶段：收集代码上下文并编写交接材料，如 `context.md` 和 `meta-prompt.md`。 |
| `oracle` | 行动前的第二意见。它挑战假设、捕捉漂移，并推荐最安全的下一步，不编辑文件。 |
| `oracle-executor` | 方向获批后的谨慎实现。它是"去做批准的事"的代理。 |
| `delegate` | 当你想要一个行为接近父会话的子代理时的轻量级通用委托。 |

简单的经验法则：在你理解代码前使用 `scout`，在信任外部事实前使用 `researcher`，在大变更前使用 `planner`，用 `worker` 实现，用 `reviewer` 检查，当决策本身感觉有风险时使用 `oracle`。

## 更改内置代理的模型

你不需要复制内置代理文件来更改它们的模型。

对于单次运行，在命令中放入覆盖：

```text
/run reviewer[model=anthropic/claude-sonnet-4] "Review this diff"
```

对于持久覆盖，使用 `/agents`：

```text
/agents
```

选择内置代理，按 `e`，更改模型或其他字段，然后保存用户或项目覆盖。用户覆盖全局生效。项目覆盖仅在该仓库生效，并优先于用户覆盖。

你也可以直接编辑设置：

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

使用 `~/.pi/agent/settings.json` 作为用户覆盖，或 `.pi/settings.json` 作为项目覆盖。相同的 `agentOverrides` 块可以更改 `tools`、`skills`、继承上下文、提示文本或禁用内置。如果你想要完全不同的代理，创建同名用户或项目代理；对于普通调整，优先使用覆盖。

## 运行子代理的显示位置

前台运行在运行时流式显示进度。

后台运行在控制权返回给你后继续工作。它们显示完成通知，可以用以下方式检查：

```text
/subagents-status
```

你也可以自然询问：

```text
Show me the current subagent status.
```

如果感觉配置有误，运行：

```text
/subagents-doctor
```

或询问：

```text
Check whether subagents and intercom are set up correctly.
```

## 可选快捷方式

该包包含常见工作流的可重用提示模板。你不需要它们，但当你想要每次都相同的结构时很方便：

| 提示 | 用于 |
|--------|------------|
| `/parallel-review` | 启动具有不同角度的全新上下文审查员，然后综合要修复的内容。 |
| `/parallel-research` | 结合 `researcher` 和 `scout` 获取外部证据、本地代码上下文和实际权衡。 |
| `/gather-context-and-clarify` | 先侦察/研究，然后向用户询问重要的澄清问题。 |
| `/oracle-executor` | 将明确批准的实现任务发送给具有继承上下文的 `oracle-executor`。 |

## 可选的 pi-intercom 伴侣

`pi-subagents` 无需 `pi-intercom` 即可工作。仅当你希望子代理在运行时能与父 Pi 会话对话时才安装 `pi-intercom`。

```bash
pi install npm:pi-intercom
```

大多数用户不直接调用 `intercom`。安装 `pi-intercom` 后，`pi-subagents` 可以自动给子代理一个返回父会话的私有协调通道。

用于子进程可能需要决策而非猜测的工作：

```text
Run this implementation in the background. If the worker gets blocked or needs a product decision, have it ask me through intercom.
```

```text
Ask oracle to review this plan. If it sees a decision I need to make, have it ask me instead of assuming.
```

子进程可以使用两种消息：

- `ask`：子进程需要父会话的决策或澄清
- `send`：子进程在被阻塞或明确要求进度时发送简短更新

例行完成不通过对讲机。正常的子代理结果仍通过 `pi-subagents` 返回。

如果子进程似乎停滞，需要关注的通知可以显示在父会话中，带有有用的下一步操作，如检查 `/subagents-status`、中断运行或轻推子进程。

如果消息不显示，运行：

```text
/subagents-doctor
```

对于正常使用，你不需要配置任何东西。高级用户可以在下面的配置部分用 `intercomBridge` 调整桥接。

到目前为止，你已经足够了解如何使用该插件。README 的其余部分是参考材料，包括确切的命令语法、自定义代理、保存的链、工作树和配置。

## 直接命令

跳过此部分，直到你想要确切的语法。

| 命令 | 描述 |
|---------|-------------|
| `/run <agent> [task]` | 运行一个代理；对于自包含代理省略任务 |
| `/chain agent1 "task1" -> agent2 "task2"` | 顺序运行代理 |
| `/parallel agent1 "task1" -> agent2 "task2"` | 并行运行代理 |
| `/run-chain <chainName> -- <task>` | 启动保存的 `.chain.md` 工作流 |
| `/agents` | 打开代理管理器覆盖层 |
| `/subagents-status` | 打开活动/最近运行覆盖层 |
| `/subagents-doctor` | 显示只读设置诊断 |

命令在本地验证代理名称，支持标签补全，并将结果发送回对话。

### 每步任务

使用 `->` 分隔步骤，并为每一步分配自己的任务：

```text
/chain scout "scan the codebase" -> planner "create an implementation plan"
/parallel scanner "find security issues" -> reviewer "check code style"
```

单引号和双引号都有效。你也可以使用 `--` 作为分隔符：

```text
/chain scout -- scan code -> planner -- analyze auth
```

没有任务的步骤从执行模式继承行为。链步骤获得 `{previous}`，即前一步的输出。并行步骤使用第一个可用任务作为回退。

```text
/chain scout "analyze auth" -> planner -> worker
# scout 获得 "analyze auth"；planner 获得 scout 输出；worker 获得 planner 输出
```

对于共享任务，列出代理并在任务前放置一个 `--`：

```text
/chain scout planner -- analyze the auth system
/parallel scout reviewer -- check for security issues
```

### 内联每步配置

在代理名称后附加 `[key=value,...]` 以覆盖该步骤的默认值：

```text
/chain scout[output=context.md] "scan code" -> planner[reads=context.md] "analyze auth"
/run scout[model=anthropic/claude-sonnet-4] summarize this codebase
/parallel reviewer[skills=code-review+security] "review backend" -> reviewer[model=openai/gpt-5-mini] "review frontend"
```

| 键 | 示例 | 描述 |
|-----|---------|-------------|
| `output` | `output=context.md` | 将结果写入文件。对于 `/chain` 和 `/parallel`，相对路径位于链目录下；对于 `/run`，相对路径解析为 cwd。 |
| `reads` | `reads=a.md+b.md` | 执行前读取文件。`+` 分隔多个路径。 |
| `model` | `model=anthropic/claude-sonnet-4` | 覆盖此步骤的模型。 |
| `skills` | `skills=planning+review` | 覆盖注入的技能。`+` 分隔多个技能。 |
| `progress` | `progress` | 启用进度跟踪。 |

设置 `output=false`、`reads=false` 或 `skills=false` 以显式禁用该行为。

### 后台和分叉运行

添加 `--bg` 以在后台运行：

```text
/run scout "audit the codebase" --bg
/chain scout "analyze auth" -> planner "design refactor" -> worker --bg
/parallel scout "scan frontend" -> scout "scan backend" --bg
```

添加 `--fork` 以从父进程当前叶节点创建的真实的分支会话启动每个子进程：

```text
/run reviewer "review this diff" --fork
/chain scout "analyze this branch" -> planner "plan next steps" --fork
/parallel scout "audit frontend" -> reviewer "audit backend" --fork
```

你可以以任意顺序组合它们：

```text
/run reviewer "review this diff" --fork --bg
/run reviewer "review this diff" --bg --fork
```

`oracle` 和 `oracle-executor` 内置代理设计用于明确的决策循环。典型模式是先请 `oracle` 诊断并推荐执行提示，然后主代理批准该方向后再运行 `oracle-executor`。

## 澄清和启动 UI

链默认打开澄清 UI，以便你在运行前预览和编辑工作流。单代理和并行工具调用可以通过 `clarify: true` 选择加入相同流程；斜杠命令和 `/agents` 使用自己的启动屏幕。

常用澄清快捷键：

- `Enter` 在前台运行，或如果后台已切换则在后台运行
- `Esc` 取消或返回
- `↑↓` 在步骤或任务间移动
- `e` 编辑任务/模板
- `m` 选择模型
- `t` 选择思考级别
- `s` 选择技能
- `b` 切换后台执行
- `w` 编辑输出/写入行为（支持时）
- `r` 编辑读取（支持时）
- `p` 切换进度跟踪（支持时）
- `S` 将当前覆盖保存到代理 frontmatter
- `W` 将链配置保存到 `.chain.md`

选择屏幕使用 `↑↓`、`Enter`、`Esc` 和输入过滤。全屏编辑器支持自动换行、粘贴、`Esc` 保存和 `Ctrl+C` 丢弃。

## 代理管理器

按 `Ctrl+Shift+A` 或输入 `/agents` 打开代理管理器。这是浏览、检查、编辑、创建和启动代理及链的最简单方式。

用于查看存在的代理、调整内置覆盖、构建并行运行而无需编写斜杠语法，或保存链供以后使用。

主屏幕：

| 屏幕 | 功能 |
|--------|--------------|
| 列表 | 浏览代理和链，支持搜索、过滤、作用域标记和选择。 |
| 详情 | 查看解析后的提示、frontmatter 字段、运行历史和内置覆盖路径。 |
| 编辑 | 编辑模型、思考级别、提示模式、继承标志、技能和提示文本。 |
| 链详情 | 检查保存的链步骤。 |
| 并行构建器 | 构建并行槽，包括重复代理和每槽任务覆盖。 |
| 任务输入 | 输入共享任务并启动，支持分叉/后台/工作树切换（支持时）。 |
| 新建代理 | 从模板创建：Scout、Planner、Implementer、Code Reviewer、Blank Agent 或 Blank Chain。 |

常用快捷键：

- 输入以搜索列表
- `Enter` 打开详情屏幕
- `Alt+N` 从模板创建代理或链
- `Ctrl+R` 将选中的代理作为运行或链启动
- `Ctrl+P` 打开并行构建器
- `Tab` 在列表中选择代理或在任务输入中切换跳过澄清
- `Ctrl+A` 在并行构建器中添加槽
- `e` 编辑代理或内置覆盖
- `Ctrl+S` 保存覆盖；`r` 重置聚焦的覆盖字段；`D` 移除覆盖
- `Ctrl+K` 克隆当前项目
- `Ctrl+D` 或 `Del` 删除当前项目或移除并行槽
- `Esc` 返回上级屏幕

## 代理和链

代理是带有 YAML frontmatter 和系统提示正文的 markdown 文件。它们定义将在子 Pi 进程中运行的专家。

代理位置，从最低到最高优先级：

| 作用域 | 路径 |
|-------|------|
| 内置 | `~/.pi/agent/extensions/subagent/agents/` |
| 用户 | `~/.pi/agent/agents/{name}.md` |
| 项目 | `.pi/agents/{name}.md` |

项目发现还读取遗留 `.agents/{name}.md` 文件。如果 `.agents/` 和 `.pi/agents/` 定义了相同的项目代理，`.pi/agents/` 获胜。使用 `agentScope: "user" | "project" | "both"` 控制发现；`both` 是默认值，项目定义在名称冲突时获胜。

内置代理以最低优先级加载，因此同名用户或项目代理会覆盖它们。`oracle` 是在编辑前批判方向并提出执行提示的顾问审查员。`oracle-executor` 是仅在主代理批准行动方案后才运行的实现升级器。

`researcher` 内置使用 `web_search`、`fetch_content` 和 `get_search_content`；这些需要 [pi-web-access](https://github.com/nicobailon/pi-web-access)：

```bash
pi install npm:pi-web-access
```

### 内置覆盖

你可以覆盖选定的内置字段而无需复制整个代理。覆盖位于设置中：

- 用户：`~/.pi/agent/settings.json`
- 项目：`.pi/settings.json`

示例：

```json
{
  "subagents": {
    "agentOverrides": {
      "reviewer": {
        "inheritProjectContext": false
      }
    }
  }
}
```

支持的覆盖字段有 `model`、`fallbackModels`、`thinking`、`systemPromptMode`、`inheritProjectContext`、`inheritSkills`、`disabled`、`skills`、`tools` 和 `systemPrompt`。项目覆盖优先于用户覆盖。

你也可以从 `/agents` 管理内置覆盖。在内置详情屏幕，按 `e`，根据需要选择用户或项目作用域，然后保存要覆盖的字段。

设置 `disabled: true` 以在运行时发现中隐藏内置，同时使其在 `/agents` 中可见。对于批量控制，在设置中设置 `subagents.disableBuiltins: true`。覆盖的内置显示如 `[builtin+user]` 或 `[builtin+project]` 的标记；禁用的内置在管理器中显示 `off` 标记。

### 提示组装

子代理默认设计为狭窄。自定义代理以干净的系统提示开始，只有你故意给它们的上下文。它们不会自动继承 Pi 的整个基础提示、项目指令文件或发现的技能目录。

当代理应该看到更多时使用这些字段：

| 字段 | 效果 |
|-------|--------|
| `systemPromptMode: append` | 将代理提示附加到 Pi 的正常基础提示。 |
| `inheritProjectContext: true` | 保留来自如 `AGENTS.md` 和 `CLAUDE.md` 等文件的继承项目指令。 |
| `inheritSkills: true` | 让子进程看到 Pi 发现的技能目录。 |

内置代理默认选择加入项目指令继承，因此它们开箱即用地遵循仓库特定规则。`delegate` 也使用附加模式，因为它的工作是在父工作流内部进行编排。

### 代理 frontmatter

典型的代理如下所示：

```yaml
---
name: scout
description: Fast codebase recon
tools: read, grep, find, ls, bash, mcp:chrome-devtools
extensions:
model: claude-haiku-4-5
fallbackModels: openai/gpt-5-mini, anthropic/claude-sonnet-4
thinking: high
systemPromptMode: replace
inheritProjectContext: false
inheritSkills: false
skills: safe-bash, chrome-devtools
output: context.md
defaultReads: context.md
defaultProgress: true
interactive: true
maxSubagentDepth: 1
---

Your system prompt goes here.
```

重要字段：

| 字段 | 说明 |
|-------|-------|
| `tools` | 内置工具白名单。`mcp:` 条目在 `pi-mcp-adapter` 安装时选择直接 MCP 工具。 |
| `extensions` | 省略表示正常扩展；空表示无扩展；逗号分隔值白名单特定扩展。 |
| `model` | 默认模型。裸 ID 尽可能优先当前提供商，然后是唯一注册表匹配。 |
| `fallbackModels` | 用于提供商/模型故障（如配额、认证、超时或不可用模型）的有序备份模型。普通任务故障不会触发回退。 |
| `thinking` | 除非已存在后缀，否则在运行时附加为 `:level` 后缀。 |
| `systemPromptMode` | 默认 `replace`；`append` 保留 Pi 的基础提示。 |
| `inheritProjectContext` | 保留或剥离继承的项目指令块。 |
| `inheritSkills` | 保留或剥离 Pi 发现的技能目录。 |
| `skills` | 直接注入特定技能，无论 `inheritSkills` 如何。 |
| `output` | 默认单代理输出文件。 |
| `defaultReads` | 链/并行行为运行前读取的文件。 |
| `defaultProgress` | 维护 `progress.md`。 |
| `interactive` | 为兼容性解析但 v1 中不强制执行。 |
| `maxSubagentDepth` | 收紧此代理子进程的嵌套委托。 |

### 工具和扩展选择

如果省略 `tools`，`pi-subagents` 不会传递 `--tools`，因此子进程获得 Pi 的正常内置工具。如果存在 `tools`，常规工具名称成为显式白名单。`mcp:` 条目被拆分并作为直接 MCP 选择转发。类似路径的 `tools` 条目（如扩展路径或 `.ts`/`.js` 文件）被视为工具扩展路径而非内置工具名称。

示例：

- 省略 `tools` 和省略 `extensions`：正常内置和正常扩展。
- `tools: mcp:chrome-devtools`：正常内置加直接 Chrome DevTools MCP 工具。
- `tools: read, bash, mcp:chrome-devtools`：仅 `read` 和 `bash` 作为内置，加直接 Chrome DevTools MCP 工具。

直接 MCP 工具需要 [pi-mcp-adapter](https://github.com/nicobailon/pi-mcp-adapter)。子代理仅当其 frontmatter 中列出 `mcp:` 条目时才接收直接 MCP 工具；`mcp.json` 中的全局 `directTools: true` 本身不足。通用的 `mcp` 代理工具在可用时仍可用于发现。适配器在启动时缓存工具元数据，因此首次连接新 MCP 服务器后，在依赖直接工具前重启 Pi。

`extensions` 控制子扩展加载：

```yaml
# 省略：加载所有正常扩展

# 空：无扩展
extensions:

# 白名单
extensions: /abs/path/to/ext-a.ts, /abs/path/to/ext-b.ts
```

当存在 `extensions` 时，它优先于 `tools` 条目暗示的扩展路径。

## 链文件

链是可重用的 `.chain.md` 工作流，存储在代理文件旁边。

| 作用域 | 路径 |
|-------|------|
| 用户 | `~/.pi/agent/agents/{name}.chain.md` |
| 项目 | `.pi/agents/{name}.chain.md` |

项目发现还读取遗留 `.agents/{name}.chain.md` 文件。如果两个位置定义了相同的解析链名称，`.pi/agents/` 获胜。

示例：

```md
---
name: scout-planner
description: Gather context then plan implementation
---

## scout
output: context.md

Analyze the codebase for {task}

## planner
reads: context.md
model: anthropic/claude-sonnet-4-5:high
progress: true

Create an implementation plan based on {previous}
```

每个 `## agent-name` 部分是一个步骤。配置行如 `output`、`reads`、`model`、`skills` 和 `progress` 紧跟在标题后。空行分隔配置与任务文本。

链支持三态行为：省略从代理继承，值覆盖，`false` 禁用。

从代理管理器模板选择器创建链，从链澄清 TUI 保存，或手写。用自然语言、`/agents` 或以下方式运行：

```text
/run-chain scout-planner -- refactor authentication
```

## 链变量

任务模板支持：

| 变量 | 描述 |
|----------|-------------|
| `{task}` | 第一步的原始任务。 |
| `{previous}` | 前一步的输出，或并行步骤的聚合输出。 |
| `{chain_dir}` | 链产物目录的路径。 |

在传递给下一步之前，并行输出用清晰分隔符聚合：

```text
=== Parallel Task 1 (worker) ===
...

=== Parallel Task 2 (worker) ===
...
```

## 技能

技能是注入代理系统提示的 `SKILL.md` 文件。

发现使用项目优先顺序：

1. `.pi/skills/{name}/SKILL.md`
2. 通过 `package.json -> pi.skills` 的项目包和项目设置包
3. 通过 `package.json -> pi.skills` 的当前任务 cwd 包
4. `.pi/settings.json -> skills`
5. `~/.pi/agent/skills/{name}/SKILL.md`
6. 通过 `package.json -> pi.skills` 的用户包和用户设置包
7. `~/.pi/agent/settings.json -> skills`

使用代理默认值，在运行时覆盖，或禁用：

```ts
{ agent: "scout", task: "..." }
{ agent: "scout", task: "...", skill: "tmux, safe-bash" }
{ agent: "scout", task: "...", skill: false }
```

对于链，顶层 `skill` 是附加的。步骤级 `skill` 覆盖该步骤；`false` 禁用该步骤的技能。

注入的技能使用此格式：

```xml
<skill name="safe-bash">
[来自 SKILL.md 的技能内容，frontmatter 已剥离]
</skill>
```

缺失的技能不会导致执行失败。结果摘要显示警告。

## 程序化工具使用

这些是 LLM 调用 `subagent` 工具时传递的参数。大多数用户使用自然语言或斜杠命令。

### 执行示例

```ts
// 单代理
{ agent: "worker", task: "refactor auth" }
{ agent: "scout", task: "find todos", maxOutput: { lines: 1000 } }
{ agent: "scout", task: "investigate", output: false }

// 分叉上下文
{ agent: "worker", task: "continue this thread", context: "fork" }

// 并行
{ tasks: [{ agent: "scout", task: "a" }, { agent: "reviewer", task: "b" }] }
{ tasks: [{ agent: "scout", task: "audit auth", count: 3 }] }
{ tasks: [{ agent: "scout", task: "audit frontend" }, { agent: "reviewer", task: "audit backend" }], context: "fork" }

// 链
{ chain: [
  { agent: "scout", task: "Gather context for auth refactor" },
  { agent: "planner" },
  { agent: "worker" },
  { agent: "reviewer" }
]}

// 无 TUI 的链，适合后台执行
{ chain: [...], clarify: false, async: true }

// 扇出/扇入链
{ chain: [
  { agent: "scout", task: "Gather context" },
  { parallel: [
    { agent: "worker", task: "Implement feature A from {previous}" },
    { agent: "worker", task: "Implement feature B from {previous}" }
  ], concurrency: 2, failFast: true },
  { agent: "reviewer", task: "Review all changes from {previous}" }
]}

// Worktree 隔离
{ tasks: [
  { agent: "worker", task: "Implement auth" },
  { agent: "worker", task: "Implement API" }
], worktree: true }
```

### 管理操作

代理定义默认不加载到上下文中。管理操作让 LLM 在运行时发现、检查、创建、更新和删除代理及链。

```ts
{ action: "list" }
{ action: "list", agentScope: "project" }
{ action: "get", agent: "scout" }
{ action: "get", chainName: "review-pipeline" }

{ action: "create", config: {
  name: "Code Scout",
  description: "Scans codebases for patterns and issues",
  scope: "user",
  systemPrompt: "You are a code scout...",
  systemPromptMode: "replace",
  inheritProjectContext: false,
  inheritSkills: false,
  model: "anthropic/claude-sonnet-4",
  fallbackModels: ["openai/gpt-5-mini", "anthropic/claude-haiku-4-5"],
  tools: "read, bash, mcp:github/search_repositories",
  extensions: "",
  skills: "parallel-scout",
  thinking: "high",
  output: "context.md",
  reads: "shared-context.md",
  progress: true
}}

{ action: "create", config: {
  name: "review-pipeline",
  description: "Scout then review",
  scope: "project",
  steps: [
    { agent: "scout", task: "Scan {task}", output: "context.md" },
    { agent: "reviewer", task: "Review {previous}", reads: ["context.md"] }
  ]
}}

{ action: "update", agent: "scout", config: { model: "openai/gpt-4o" } }
{ action: "update", chainName: "review-pipeline", config: { steps: [...] } }
{ action: "delete", agent: "scout" }
{ action: "delete", chainName: "review-pipeline" }
```

`create` 使用 `config.scope`，不是 `agentScope`。`update` 和 `delete` 仅在相同名称存在于多个作用域时使用 `agentScope`。要清除可选字符串字段，设置为 `false` 或 `""`。

### 参数参考

| 参数 | 类型 | 默认值 | 描述 |
|-------|------|---------|-------------|
| `agent` | string | - | 单模式代理名称，或管理操作目标。 |
| `task` | string | - | 单模式任务字符串。 |
| `action` | string | - | `list`、`get`、`create`、`update`、`delete`、`status`、`interrupt` 或 `doctor`。 |
| `chainName` | string | - | 管理操作的链名称。 |
| `config` | object/string | - | 创建/更新的代理或链配置。 |
| `output` | `string \| false` | 代理默认 | 覆盖单代理输出文件。 |
| `skill` | `string \| string[] \| false` | 代理默认 | 覆盖技能或禁用全部。 |
| `model` | string | 代理默认 | 覆盖模型。 |
| `tasks` | array | - | 顶层并行任务。支持 `agent`、`task`、`cwd`、`count`、`output`、`reads`、`progress`、`skill` 和 `model`。 |
| `concurrency` | number | 配置或 `4` | 顶层并行并发。 |
| `worktree` | boolean | false | 为并行任务创建隔离的 git worktree。 |
| `chain` | array | - | 顺序和并行链步骤。 |
| `context` | `fresh \| fork` | `fresh` | `fork` 从父叶节点创建真实的分支会话。 |
| `chainDir` | string | 临时链目录 | 链产物的持久目录。 |
| `clarify` | boolean | 链为 true | 显示 TUI 预览/编辑流程。 |
| `agentScope` | `user \| project \| both` | `both` | 代理发现作用域。项目在冲突时获胜。 |
| `async` | boolean | false | 后台执行。链需要 `clarify: false`。 |
| `cwd` | string | 运行时 cwd | 覆盖工作目录。 |
| `maxOutput` | object | 200KB, 5000 行 | 最终输出截断限制。 |
| `artifacts` | boolean | true | 写入调试产物。 |
| `includeProgress` | boolean | false | 在结果中包含完整进度。 |
| `share` | boolean | false | 上传会话导出到 GitHub Gist。 |
| `sessionDir` | string | 派生 | 覆盖会话日志目录。 |

`context: "fork"` 在父会话未持久化、当前叶节点缺失或无法创建分支子会话时快速失败。它从不会静默降级为 `fresh`。

顺序和并行链任务接受 `agent`、`task`、`cwd`、`output`、`reads`、`progress`、`skill` 和 `model`。并行任务还接受 `count`。并行步骤组接受 `parallel`、`concurrency`、`failFast` 和 `worktree`。

状态和控制操作：

```ts
subagent({ action: "status" })
subagent({ action: "status", id: "<run-id>" })
subagent({ action: "interrupt", id: "<run-id>" })
subagent({ action: "doctor" })
```

## Worktree 隔离

如果并行代理编辑相同的 checkout，它们可能会相互覆盖。`worktree: true` 为每个并行子进程提供从 `HEAD` 分支的自己的 git worktree。

```ts
{ tasks: [
  { agent: "worker", task: "Implement auth", count: 2 },
  { agent: "worker", task: "Implement API" }
], worktree: true }

{ chain: [
  { agent: "scout", task: "Gather context" },
  { parallel: [
    { agent: "worker", task: "Implement feature A from {previous}" },
    { agent: "worker", task: "Implement feature B from {previous}" }
  ], worktree: true },
  { agent: "reviewer", task: "Review all changes from {previous}" }
]}
```

要求：

- 在 git 仓库中运行
- 工作树必须干净
- 存在时 `node_modules/` 会符号链接到每个 worktree
- 任务级 `cwd` 覆盖必须省略或与共享 cwd 匹配
- 配置的 `worktreeSetupHook` 必须在超时前返回有效 JSON

Worktree 并行步骤完成后，每代理差异统计会附加到输出，完整补丁文件会写入产物。Worktree 和临时分支在 `finally` 块中清理。

## 配置

`pi-subagents` 从 `~/.pi/agent/extensions/subagent/config.json` 读取可选 JSON 配置。

### `asyncByDefault`

```json
{ "asyncByDefault": true }
```

使顶层调用在请求未显式设置 `async` 时使用后台执行。调用者仍可用 `async: false` 强制前台，除非启用了 `forceTopLevelAsync`。

### `forceTopLevelAsync`

```json
{ "forceTopLevelAsync": true }
```

将深度 0 的单、并行和链运行强制为后台模式，并通过强制 `clarify: false` 绕过澄清 UI。嵌套调用保留自己的继承设置。

### `parallel`

```json
{
  "parallel": {
    "maxTasks": 12,
    "concurrency": 6
  }
}
```

`maxTasks` 默认为 `8`；`concurrency` 默认为 `4`。每次调用 `concurrency` 优先。

### `defaultSessionDir`

```json
{ "defaultSessionDir": "~/.pi/agent/sessions/subagent/" }
```

会话目录优先级：首先 `params.sessionDir`，然后 `config.defaultSessionDir`，然后从父会话派生的目录。会话始终启用。

### `maxSubagentDepth`

```json
{ "maxSubagentDepth": 1 }
```

在没有继承的 `PI_SUBAGENT_MAX_DEPTH` 生效时控制嵌套委托。每代理 `maxSubagentDepth` 可以收紧该代理子进程的限制，但不能放宽继承的更严格限制。

### `intercomBridge`

```json
{
  "intercomBridge": {
    "mode": "always",
    "instructionFile": "./intercom-bridge.md"
  }
}
```

控制子代理是否接收运行时对讲机协调指令，以及是否在需要时自动将 `intercom` 添加到其工具白名单。

字段：

- `mode`：默认 `always`；使用 `fork-only` 仅对分叉运行注入，或 `off` 禁用桥接。
- `instructionFile`：可选 Markdown 模板替换默认桥接指令。`{orchestratorTarget}` 会被插值。相对路径从 `~/.pi/agent/extensions/subagent/` 解析。

桥接激活还需要 `pi-intercom` 已安装并启用、可定位的当前会话名称或回退别名，以及任何显式代理 `extensions` 白名单中的 `pi-intercom`。

默认注入的指导告诉子进程仅将对讲机用于协调：阻塞或需要决策时询问，仅在被阻塞或明确要求时发送更新，避免例行完成交接。

### `worktreeSetupHook`

```json
{
  "worktreeSetupHook": "./scripts/setup-worktree.mjs",
  "worktreeSetupHookTimeoutMs": 45000
}
```

钩子对每个创建的 worktree 运行一次。路径必须是绝对路径、`~/...` 或仓库相对路径；裸命令名被拒绝。

stdin 是具有 `repoRoot`、`worktreePath`、`agentCwd`、`branch`、`index`、`runId` 和 `baseCommit` 的 JSON 对象。stdout 必须是一个 JSON 对象，例如：

```json
{ "syntheticPaths": [".venv", ".env.local"] }
```

`syntheticPaths` 必须是相对于 worktree 根的路径。它们在捕获差异前被移除，因此辅助文件不会污染补丁。从不排除跟踪文件；将跟踪路径标记为合成会导致设置失败。默认超时为 `30000` ms。

## 文件、日志和可观测性

每个链运行创建类似如下的用户范围临时目录：

```text
<tmpdir>/pi-subagents-<scope>/chain-runs/{runId}/
```

它可能包含如 `context.md`、`plan.md`、`progress.md` 和 `parallel-{stepIndex}/.../output.md` 等文件。超过 24 小时的目录在扩展启动时清理。

调试产物位于 `{sessionDir}/subagent-artifacts/` 或用户范围临时产物目录。每项任务你可能会看到：

- `{runId}_{agent}_input.md`
- `{runId}_{agent}_output.md`
- `{runId}_{agent}.jsonl`
- `{runId}_{agent}_meta.json`

元数据记录时间、使用、退出代码、最终模型、尝试的模型和回退尝试结果。

会话文件存储在每运行会话目录下。使用 `context: "fork"` 时，每个子进程从父进程当前叶节点生成的 `--session <branched-session-file>` 开始。那是真实的会话分叉，不是注入的摘要。

异步完成仅通知发起会话。结果观察者发出 `subagent:async-complete`，扩展消费该事件以渲染完成通知。

异步运行写入：

```text
<tmpdir>/pi-subagents-<scope>/async-subagent-runs/<id>/
  status.json
  events.jsonl
  output-<n>.log
  subagent-log-<id>.md
```

`status.json` 驱动小部件和 `/subagents-status`。`events.jsonl` 包含包装事件以及带有运行和步骤元数据的子 Pi JSON 事件。`output-<n>.log` 是实时人类可读尾部。回退信息被持久化，因此后台运行完成后可调试。

## 实时进度

前台运行显示单、链和并行模式的紧凑实时进度：当前工具、最近输出、token 计数、持续时间、活动新鲜度和当前工具持续时间。

按 `Ctrl+O` 展开完整流视图，显示每步的完整输出。

顺序链显示如 `done scout → running planner` 的流程线。带并行步骤的链改为显示每步卡片。

## 会话分享

传递 `share: true` 以将完整会话导出为 HTML，通过 `gh` 凭证上传到秘密 GitHub Gist，并返回 `https://shittycodingagent.ai/session/?<gistId>` URL。

```ts
{ agent: "scout", task: "...", share: true }
```

默认禁用。会话数据可能包含源代码、路径、环境变量、凭证或其他敏感输出。需要已安装并认证的 `gh`。

## 递归守卫

子代理可以调用 `subagent`，这可能变得昂贵且难以观察。深度守卫防止无界嵌套。

默认情况下，嵌套限制为两级：主会话 → 子代理 → 子子代理。更深的调用被阻止，并指导直接完成当前任务。

配置限制：

1. 启动 Pi 前的 `PI_SUBAGENT_MAX_DEPTH`
2. `config.maxSubagentDepth`
3. 代理 frontmatter 中的 `maxSubagentDepth`，只能收紧继承的限制

```bash
export PI_SUBAGENT_MAX_DEPTH=3
export PI_SUBAGENT_MAX_DEPTH=1
export PI_SUBAGENT_MAX_DEPTH=0
```

`PI_SUBAGENT_DEPTH` 是内部自动传播的。不要手动设置。

## 事件

异步事件：

- `subagent:async-started`
- `subagent:async-complete`

结果观察者发出 `subagent:async-complete`；`index.ts` 注册消费它的通知处理程序。控制/注意事件作为可见的父通知显示并持久化用于异步运行。使用 `pi-intercom` 时，需要关注的通知也可以通过 intercom 到达编排器。

## 提示模板集成

`pi-subagents` 通过自然语言、`subagent` 工具、斜杠命令和本 README 顶部列出的打包提示快捷方式独立工作。如果使用 [pi-prompt-template-model](https://github.com/nicobailon/pi-prompt-template-model)，你也可以将子代理委托包装在你自己的可重用提示模板中。

示例：

```md
---
description: Take a screenshot
model: claude-sonnet-4-20250514
subagent: browser-screenshoter
cwd: /tmp/screenshots
---
Use url in the prompt to take screenshot: $@
```

然后 `/take-screenshot https://example.com` 切换到 Sonnet，委托给 `browser-screenshoter`，cwd 为 `/tmp/screenshots`，完成后恢复你的模型。运行时覆盖如 `--cwd=<path>` 和 `--subagent=<name>` 也有效。

对于子代理之上更多的可重用工作流，包括 `/chain-prompts` 和比较式提示如 `/best-of-n`，单独安装 `pi-prompt-template-model` 并将你想要的示例复制到 `~/.pi/agent/prompts/`。

## 运行时文件

主要运行时文件：

| 文件 | 用途 |
|------|---------|
| `index.ts` | 扩展注册、工具注册、消息/渲染连接。 |
| `agents.ts` | 代理和链发现、frontmatter 解析。 |
| `subagent-executor.ts` | 单、并行、链、管理、状态、中断和 doctor 操作的主要执行路由。 |
| `execution.ts` | 核心前台 `runSync` 处理。 |
| `subagent-runner.ts` | 分离的异步运行器。 |
| `async-execution.ts` | 后台启动支持。 |
| `async-status.ts` / `subagents-status.ts` | 状态发现和覆盖 UI。 |
| `chain-execution.ts` / `chain-serializer.ts` | 链编排和 `.chain.md` 解析。 |
| `agent-manager*.ts` | 代理管理器屏幕和编辑流程。 |
| `settings.ts` | 链行为、指令和配置辅助。 |
| `worktree.ts` | Git worktree 隔离。 |
| `intercom-bridge.ts` | 运行时 intercom 桥接指令和诊断。 |
| `schemas.ts` / `types.ts` | 工具模式、共享类型和事件常量。 |
| `test/unit/` / `test/integration/` | 单元和基于加载器的集成测试。 |
