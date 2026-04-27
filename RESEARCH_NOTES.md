# pi-subagents 扩展项目研究笔记

> 研究时间: 2026-04-27
> 研究范围: pi-subagents 完整源码架构

---

## 1. 项目概述

**pi-subagents** 是 Pi 的一个扩展（Extension），实现了子代理（Sub-agents）功能。它允许 Pi 将工作委托给专门的子代理，支持代码审查、侦察、实现、并行审计、保存工作流、后台作业等场景。

### 核心概念
- **Pi**：父会话（Parent session）
- **Subagent**：专注的子 Pi 会话，有自己的任务
- 支持**前台运行**（实时流式输出）和**后台运行**（异步执行）

---

## 2. 核心功能模块

### 2.1 执行模式
| 模式 | 说明 |
|------|------|
| **Single** | 单代理执行 `{ agent, task }` |
| **Parallel** | 并行执行多个任务 `{ tasks: [...] }` |
| **Chain** | 链式执行 `{ chain: [...] }`，支持串行和并行步骤 |
| **Management** | 管理操作（list/get/create/update/delete） |

### 2.2 内置代理（Builtin Agents）
位于 `agents/` 目录：

| 代理 | 用途 |
|------|------|
| `scout` | 快速代码库侦察，返回压缩上下文 |
| `researcher` | 网页/文档研究，带来源引用 |
| `planner` | 从现有上下文生成实现计划 |
| `worker` | 通用实现工作 |
| `reviewer` | 代码审查和小修复 |
| `context-builder` | 构建代码上下文和交接材料 |
| `oracle` | 高上下文决策一致性检查，防止漂移 |
| `oracle-executor` | 谨慎实现（方向已批准后执行） |
| `delegate` | 轻量级通用委托代理 |

### 2.3 主要功能特性
- **Fork 上下文**：创建分支会话继承父状态
- **Worktree 隔离**：并行任务使用独立 git worktree
- **Intercom 桥接**：子代理与父会话通信
- **递归深度守卫**：防止无限嵌套（默认最大深度 2）
- **模型回退**：支持 fallback models
- **Skill 注入**：动态注入技能到系统提示
- **进度追踪**：实时显示工具调用、token 使用、耗时
- **后台执行**：异步运行，完成后通知

---

## 3. 主要文件作用

### 3.1 核心入口和注册
| 文件 | 作用 |
|------|------|
| `index.ts` | 扩展注册入口，注册 `subagent` 工具、消息渲染器、事件处理 |
| `subagent-executor.ts` | 主要执行路由，处理所有执行模式和管理操作 |

### 3.2 代理管理
| 文件 | 作用 |
|------|------|
| `agents.ts` | 代理发现、配置解析、内置代理覆盖机制 |
| `agent-management.ts` | 代理 CRUD 操作（创建、更新、删除） |
| `agent-manager*.ts` | Agents Manager TUI 界面 |
| `agent-selection.ts` | 代理选择逻辑 |
| `agent-scope.ts` | 代理作用域解析（user/project/both） |
| `agent-serializer.ts` | 代理序列化/反序列化 |
| `agent-overrides.ts` | 内置代理覆盖配置处理 |

### 3.3 执行引擎
| 文件 | 作用 |
|------|------|
| `execution.ts` | 核心同步执行逻辑（`runSync`） |
| `async-execution.ts` | 后台执行启动支持 |
| `subagent-runner.ts` | 分离的异步运行器 |
| `chain-execution.ts` | 链式执行编排 |
| `chain-serializer.ts` | `.chain.md` 文件解析 |

### 3.4 辅助功能
| 文件 | 作用 |
|------|------|
| `types.ts` | 所有类型定义和常量 |
| `schemas.ts` | 工具参数 Schema |
| `skills.ts` | Skill 发现和注入 |
| `worktree.ts` | Git worktree 隔离 |
| `fork-context.ts` | Fork 上下文处理 |
| `intercom-bridge.ts` | Intercom 通信桥接 |
| `model-fallback.ts` | 模型回退逻辑 |
| `pi-spawn.ts` | Pi 进程启动 |
| `pi-args.ts` | Pi 命令行参数构建 |
| `artifacts.ts` | 产物文件管理 |
| `render.ts` | 结果渲染 |
| `notify.ts` | 通知处理 |

### 3.5 UI/TUI
| 文件 | 作用 |
|------|------|
| `chain-clarify.ts` | 链式执行前的确认 UI |
| `subagents-status.ts` | 子代理状态查看界面 |
| `run-status.ts` | 运行状态检查 |
| `text-editor.ts` | 文本编辑器组件 |

---

## 4. 与 Pi 的集成方式

### 4.1 扩展注册
在 `index.ts` 中通过 `ExtensionAPI` 注册：

```typescript
export default function registerSubagentExtension(pi: ExtensionAPI): void {
  // 注册工具
  pi.registerTool(tool);
  
  // 注册消息渲染器
  pi.registerMessageRenderer<SlashMessageDetails>(SLASH_RESULT_TYPE, ...);
  
  // 注册斜杠命令
  registerSlashCommands(pi, state);
  
  // 监听事件
  pi.on("tool_result", ...);
  pi.on("session_start", ...);
  pi.on("session_shutdown", ...);
}
```

### 4.2 工具定义
`subagent` 工具通过 `ToolDefinition` 定义：

```typescript
const tool: ToolDefinition<typeof SubagentParams, Details> = {
  name: "subagent",
  label: "Subagent",
  description: "...",
  parameters: SubagentParams,
  execute: ...,
  renderCall: ...,
  renderResult: ...,
};
```

### 4.3 依赖的 Pi 核心包
- `@mariozechner/pi-agent-core` - 核心类型（AgentToolResult 等）
- `@mariozechner/pi-ai` - AI 消息类型
- `@mariozechner/pi-coding-agent` - 扩展 API 和上下文
- `@mariozechner/pi-tui` - TUI 组件

### 4.4 配置集成
通过 `~/.pi/agent/extensions/subagent/config.json` 配置：
- `asyncByDefault` - 默认异步执行
- `forceTopLevelAsync` - 强制顶层异步
- `maxSubagentDepth` - 最大嵌套深度
- `intercomBridge` - Intercom 桥接配置
- `worktreeSetupHook` - Worktree 设置钩子

### 4.5 代理定义集成
代理存储位置（优先级从低到高）：
1. **内置**: `~/.pi/agent/extensions/subagent/agents/`
2. **用户**: `~/.pi/agent/agents/` 或 `~/.agents/`
3. **项目**: `.pi/agents/` 或 `.agents/`

代理文件格式为 Markdown，包含 YAML frontmatter：

```yaml
---
name: scout
description: Fast codebase recon
tools: read, grep, find, ls, bash, write
model: openai-codex/gpt-5.4-mini
systemPromptMode: replace
inheritProjectContext: true
output: context.md
---
```

---

## 5. 架构特点

### 5.1 分层架构
```
┌─────────────────────────────────────┐
│  UI Layer (TUI/Render)              │
│  - chain-clarify, agent-manager     │
├─────────────────────────────────────┤
│  Execution Layer                    │
│  - subagent-executor, execution     │
│  - async-execution, chain-execution │
├─────────────────────────────────────┤
│  Agent Management Layer             │
│  - agents, agent-management         │
│  - skills, worktree                 │
├─────────────────────────────────────┤
│  Infrastructure Layer               │
│  - pi-spawn, pi-args                │
│  - artifacts, fork-context          │
│  - intercom-bridge                  │
└─────────────────────────────────────┘
```

### 5.2 事件驱动
使用事件总线进行组件间通信：
- `SUBAGENT_ASYNC_STARTED_EVENT` - 异步任务开始
- `SUBAGENT_ASYNC_COMPLETE_EVENT` - 异步任务完成
- `SUBAGENT_CONTROL_EVENT` - 控制事件
- `INTERCOM_DETACH_REQUEST_EVENT` - Intercom 分离请求

### 5.3 状态管理
全局状态存储在 `SubagentState` 中：
- `asyncJobs` - 异步任务映射
- `foregroundControls` - 前台运行控制
- `currentSessionId` - 当前会话 ID

---

## 6. 使用方式

### 自然语言调用
```
"Use reviewer to review this diff"
"Ask oracle for a second opinion"
"Run parallel reviewers"
```

### 工具调用
```typescript
// 单代理
{ agent: "scout", task: "analyze codebase" }

// 并行
{ tasks: [{ agent: "scout", task: "audit frontend" }, { agent: "reviewer", task: "audit backend" }] }

// 链式
{ chain: [{ agent: "scout", task: "..." }, { agent: "planner" }, { agent: "worker" }] }
```

### 斜杠命令
- `/run <agent>` - 运行代理
- `/chain` - 链式执行
- `/parallel` - 并行执行
- `/agents` - 打开代理管理器
- `/subagents-status` - 查看状态

---

## 7. 总结

pi-subagents 是 Pi 扩展生态系统中一个非常成熟和功能丰富的子代理实现，提供了完整的代理生命周期管理、执行编排和监控能力。

---

*笔记生成时间: 2026-04-27*
