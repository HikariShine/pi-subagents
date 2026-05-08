# pi-subagents 项目学习进度

## 📋 使用说明

**本文件是 pi-subagents 项目的学习进度跟踪文件。**

### 如果你是新 Session，请执行以下操作：

1. **阅读本文件**，了解项目结构和阅读顺序
2. **查看下方"文件列表"**，找到标记为 `❌ 未读` 的文件
3. **按"推荐阅读顺序"** 逐个阅读文件
4. **每读完一个文件后**：
   - 在文件列表中将 `❌ 未读` 改为 `✅ 已读`
   - 在"详解文件路径"列添加对应的 `.md` 文件路径
   - 创建对应的详解 `.md` 文件（位于 `study-notes/` 目录下）

---

## 📚 推荐阅读顺序

### 第一阶段：基础概念（入口和类型）
1. `package.json` - 了解项目配置和依赖
2. `README.md` - 项目整体介绍
3. `types.ts` - 核心类型定义（整个项目的基础）
4. `schemas.ts` - 参数 schema 定义

### 第二阶段：核心执行架构
5. `index.ts` - 插件入口和注册逻辑
6. `subagent-executor.ts` - 子代理执行器（核心）
7. `execution.ts` - 执行逻辑
8. `subagent-runner.ts` - 子代理运行器

### 第三阶段：Agent 定义和管理
9. `agents.ts` - Agent 定义和管理
10. `agents/*.md` - 内置 Agent 配置（oracle, reviewer, scout 等）
11. `agent-management.ts` - Agent 管理核心
12. `agent-manager.ts` - Agent 管理器

### 第四阶段：执行模式
13. `async-execution.ts` - 异步执行
14. `async-job-tracker.ts` - 异步作业追踪
15. `chain-execution.ts` - 链式执行
16. `chain-clarify.ts` - 链式澄清
17. `parallel-utils.ts` - 并行执行工具

### 第五阶段：支撑功能
18. `artifacts.ts` - 产物管理
19. `fork-context.ts` - Fork 上下文
20. `result-watcher.ts` - 结果监视器
21. `worktree.ts` - Git worktree 管理

### 第六阶段：UI 和交互
22. `render.ts` - 结果渲染
23. `slash-commands.ts` - Slash 命令
24. `slash-bridge.ts` - Slash 桥接
25. `intercom-bridge.ts` - 进程间通信

### 第七阶段：辅助工具
26. `utils.ts`, `formatters.ts` - 工具函数
27. `settings.ts` - 配置管理
28. `doctor.ts` - 诊断工具
29. 其他辅助文件

### 第八阶段：测试和文档
30. `test/` 目录下的测试文件
31. 其他文档文件

---

## 🔗 文件依赖关系

```
┌─────────────────────────────────────────────────────────────┐
│                        基础设施层                            │
│  types.ts ◄─── schemas.ts ◄─── utils.ts ◄─── formatters.ts │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        入口层                                │
│                    index.ts (注册中心)                       │
│                     /        \                               │
│        subagent-executor    agents.ts                       │
│              |                  |                           │
│    ┌─────────┴─────────┐    ┌───┴───┐                       │
│    │                   │    │       │                       │
│ execution.ts    subagent-runner.ts  │                       │
│    │                        agents/*.md                     │
│    ▼                        agent-management.ts             │
│ async-execution.ts ◄── async-job-tracker.ts                │
│    │                        agent-manager.ts               │
│    ▼                        agent-manager-*.ts             │
│ chain-execution.ts ◄── chain-clarify.ts                    │
│ chain-serializer.ts                                         │
│    │                                                        │
│    ▼                                                        │
│ parallel-utils.ts                                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        支撑层                                │
│  artifacts.ts, fork-context.ts, result-watcher.ts          │
│  worktree.ts, settings.ts                                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        UI 层                               │
│  render.ts, render-helpers.ts                              │
│  slash-commands.ts, slash-bridge.ts                        │
│  slash-live-state.ts, intercom-bridge.ts                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 📁 文件列表

### 核心入口文件

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| index.ts | 22K | 插件主入口，注册所有组件 | ExtensionAPI, ToolDefinition | ❌ 未读 | - |
| package.json | 1.7K | 项目配置和依赖声明 | pi.extensions, pi.skills | ❌ 未读 | - |
| install.mjs | 2.6K | 安装脚本 | - | ❌ 未读 | - |

### 类型定义和 Schema

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| types.ts | 17K | 核心类型定义 | SubagentState, ExtensionConfig | ❌ 未读 | - |
| schemas.ts | 9.6K | 工具参数 schema | SubagentParams | ❌ 未读 | - |
| utils.ts | 15K | 工具函数集合 | - | ❌ 未读 | - |
| formatters.ts | 3.8K | 格式化工具 | formatDuration, shortenPath | ❌ 未读 | - |
| frontmatter.ts | 896 | frontmatter 解析 | - | ❌ 未读 | - |

### 子代理执行核心

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| subagent-executor.ts | 65K | 子代理执行器核心 | createSubagentExecutor | ❌ 未读 | - |
| subagent-runner.ts | 48K | 子代理运行器 | SubagentRunner | ❌ 未读 | - |
| execution.ts | 24K | 执行逻辑实现 | ExecutionEngine | ❌ 未读 | - |

### Agent 定义和管理

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| agents.ts | 27K | Agent 定义和管理 | discoverAgents, AgentDefinition | ❌ 未读 | - |
| agent-management.ts | 30K | Agent 管理核心 | AgentManager | ❌ 未读 | - |
| agent-manager.ts | 46K | Agent 管理器主逻辑 | - | ❌ 未读 | - |
| agent-manager-list.ts | 9.0K | Agent 列表功能 | - | ❌ 未读 | - |
| agent-manager-detail.ts | 9.4K | Agent 详情功能 | - | ❌ 未读 | - |
| agent-manager-edit.ts | 29K | Agent 编辑功能 | - | ❌ 未读 | - |
| agent-manager-parallel.ts | 11K | 并行 Agent 管理 | - | ❌ 未读 | - |
| agent-manager-chain-detail.ts | 7.6K | 链式 Agent 详情 | - | ❌ 未读 | - |
| agent-selection.ts | 739 | Agent 选择逻辑 | - | ❌ 未读 | - |
| agent-serializer.ts | 3.7K | Agent 序列化 | - | ❌ 未读 | - |
| agent-scope.ts | 219 | Agent 作用域定义 | - | ❌ 未读 | - |
| agent-templates.ts | 2.2K | Agent 模板 | - | ❌ 未读 | - |

### 异步执行

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| async-execution.ts | 17K | 异步执行实现 | AsyncExecutionEngine | ❌ 未读 | - |
| async-job-tracker.ts | 7.4K | 异步作业追踪 | AsyncJobTracker | ❌ 未读 | - |
| async-status.ts | 9.0K | 异步状态管理 | AsyncStatusManager | ❌ 未读 | - |

### 链式执行

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| chain-execution.ts | 28K | 链式执行实现 | ChainExecutionEngine | ❌ 未读 | - |
| chain-clarify.ts | 47K | 链式澄清逻辑 | ChainClarifier | ❌ 未读 | - |
| chain-serializer.ts | 4.1K | 链式序列化 | - | ❌ 未读 | - |

### 并行执行

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| parallel-utils.ts | 2.8K | 并行执行工具 | - | ❌ 未读 | - |

### 产物和上下文管理

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| artifacts.ts | 3.1K | 产物管理 | ArtifactsManager | ❌ 未读 | - |
| fork-context.ts | 1.9K | Fork 上下文 | ForkContext | ❌ 未读 | - |
| worktree.ts | 17K | Git worktree 管理 | WorktreeManager | ❌ 未读 | - |
| pi-spawn.ts | 2.9K | Pi 进程创建 | - | ❌ 未读 | - |
| pi-args.ts | 4.2K | Pi 参数处理 | - | ❌ 未读 | - |

### 结果和状态管理

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| result-watcher.ts | 3.2K | 结果文件监视器 | ResultWatcher | ❌ 未读 | - |
| run-status.ts | 5.0K | 运行状态查询 | RunStatus | ❌ 未读 | - |
| run-history.ts | 1.6K | 运行历史 | - | ❌ 未读 | - |
| settings.ts | 11K | 设置管理 | SettingsManager | ❌ 未读 | - |

### UI 和渲染

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| render.ts | 34K | 结果渲染器 | renderSubagentResult | ❌ 未读 | - |
| render-helpers.ts | 2.5K | 渲染辅助函数 | - | ❌ 未读 | - |

### Slash 命令系统

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| slash-commands.ts | 21K | Slash 命令实现 | registerSlashCommands | ❌ 未读 | - |
| slash-bridge.ts | 5.0K | Slash 桥接逻辑 | SlashSubagentBridge | ❌ 未读 | - |
| slash-live-state.ts | 8.8K | Slash 实时状态 | SlashLiveState | ❌ 未读 | - |

### 提示模板和通信

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| prompt-template-bridge.ts | 13K | 提示模板桥接 | PromptTemplateBridge | ❌ 未读 | - |
| intercom-bridge.ts | 9.5K | 进程间通信桥接 | IntercomBridge | ❌ 未读 | - |
| notify.ts | 2.9K | 通知系统 | SubagentNotify | ❌ 未读 | - |

### 控制和辅助

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| subagent-control.ts | 5.5K | 子代理控制 | ControlEvent | ❌ 未读 | - |
| subagents-status.ts | 19K | 子代理状态 | SubagentsStatus | ❌ 未读 | - |
| subagent-prompt-runtime.ts | 2.9K | 提示运行时 | - | ❌ 未读 | - |

### 辅助工具

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| model-fallback.ts | 3.0K | 模型回退 | ModelFallback | ❌ 未读 | - |
| single-output.ts | 3.1K | 单输出处理 | SingleOutput | ❌ 未读 | - |
| file-coalescer.ts | 1.0K | 文件合并 | FileCoalescer | ❌ 未读 | - |
| completion-dedupe.ts | 2.0K | 完成去重 | CompletionDedupe | ❌ 未读 | - |
| doctor.ts | 6.8K | 诊断工具 | Doctor | ❌ 未读 | - |
| jsonl-writer.ts | 1.9K | JSONL 写入 | JSONLWriter | ❌ 未读 | - |
| text-editor.ts | 7.2K | 文本编辑器 | TextEditor | ❌ 未读 | - |
| post-exit-stdio-guard.ts | 1.9K | 退出后 IO 保护 | - | ❌ 未读 | - |
| top-level-async.ts | 337 | 顶级异步 | - | ❌ 未读 | - |
| session-tokens.ts | 1.3K | 会话令牌 | SessionTokens | ❌ 未读 | - |

### 技能系统

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| skills.ts | 20K | 技能管理 | SkillsManager | ❌ 未读 | - |
| skills/pi-subagents/SKILL.md | - | 技能文档（英文） | - | ❌ 未读 | - |
| skills/pi-subagents/SKILL_zh.md | - | 技能文档（中文） | - | ❌ 未读 | - |

### Agent 配置文件

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| agents/oracle.md | 4.4K | Oracle Agent 配置 | - | ❌ 未读 | - |
| agents/reviewer.md | 1.4K | Reviewer Agent 配置 | - | ❌ 未读 | - |
| agents/scout.md | 1.6K | Scout Agent 配置 | - | ❌ 未读 | - |
| agents/planner.md | 1.5K | Planner Agent 配置 | - | ❌ 未读 | - |
| agents/worker.md | 1.3K | Worker Agent 配置 | - | ❌ 未读 | - |
| agents/oracle-executor.md | 2.6K | Oracle Executor 配置 | - | ❌ 未读 | - |
| agents/researcher.md | 1.7K | Researcher Agent 配置 | - | ❌ 未读 | - |
| agents/context-builder.md | 1.4K | Context Builder 配置 | - | ❌ 未读 | - |
| agents/delegate.md | 339 | Delegate 配置 | - | ❌ 未读 | - |
| agents_zh/*.md | - | 中文 Agent 配置 | - | ❌ 未读 | - |

### 提示模板

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| prompts/gather-context-and-clarify.md | 756 | 上下文收集提示 | - | ❌ 未读 | - |
| prompts/oracle-executor.md | 282 | Oracle Executor 提示 | - | ❌ 未读 | - |
| prompts/parallel-research.md | - | 并行研究提示 | - | ❌ 未读 | - |
| prompts/parallel-review.md | - | 并行审查提示 | - | ❌ 未读 | - |

### 项目文档

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| README.md | 40K | 项目主文档 | - | ❌ 未读 | - |
| README_zh.md | 37K | 中文项目文档 | - | ❌ 未读 | - |
| CHANGELOG.md | 69K | 更新日志 | - | ❌ 未读 | - |
| RESEARCH_NOTES.md | 8.3K | 研究笔记 | - | ❌ 未读 | - |

### 测试文件

| 文件名 | 大小 | 文件作用说明 | 重要字段 | 阅读状态 | 详解文件路径 |
|--------|------|-------------|----------|----------|-------------|
| test/unit/*.test.ts | - | 单元测试 | - | ❌ 未读 | - |
| test/integration/*.test.ts | - | 集成测试 | - | ❌ 未读 | - |
| test/support/*.ts | - | 测试支持文件 | - | ❌ 未读 | - |
| path-resolution.test.ts | 3.7K | 路径解析测试 | - | ❌ 未读 | - |

---

## 📊 进度统计

- 总文件数：约 100+ 个
- 已读文件：0
- 未读文件：100+

---

## 📝 阅读笔记目录

详解文件统一存放在 `study-notes/` 目录下：

```
study-notes/
├── 01-core/
│   ├── index.md
│   ├── types.md
│   └── schemas.md
├── 02-execution/
│   ├── subagent-executor.md
│   ├── execution.md
│   └── subagent-runner.md
├── 03-agents/
│   ├── agents.md
│   └── agent-management.md
├── ...
```

---

## ✅ 更新记录

| 日期 | 操作 | 文件 | 说明 |
|------|------|------|------|
| - | - | - | - |

