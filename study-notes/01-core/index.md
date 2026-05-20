# index.ts - 插件主入口详解

## 文件信息

- **文件名**: `index.ts`
- **大小**: 22K
- **位置**: 项目根目录
- **所属模块**: Core / 入口层

## 文件作用

`index.ts` 是 `pi-subagents` 插件的**主入口文件**，负责注册所有组件、初始化状态、设置事件监听和生命周期管理。

它是 Pi 扩展的标准入口，导出一个 `registerSubagentExtension(pi: ExtensionAPI)` 函数，Pi 在加载扩展时调用此函数。

---

## 核心内容

### 主要导出

```typescript
export default function registerSubagentExtension(pi: ExtensionAPI): void
```

**无命名导出**，只有默认导出。

---

## 九大组件注册

### 1. 消息渲染器（Message Renderers）

注册 3 个自定义消息渲染器，用于 TUI 中不同类型消息的显示：

| 渲染器 | 类型标识 | 说明 |
|--------|---------|------|
| Slash Result | `SLASH_RESULT_TYPE` | `/run` 命令执行结果卡片（带动画） |
| Subagent Notify | `"subagent-notify"` | 后台任务完成/失败/暂停通知 |
| Control Notice | `SUBAGENT_CONTROL_MESSAGE_TYPE` | 控制事件通知（暂停/恢复/中断） |

**核心辅助方法**（本文件内定义）：
- `rebuildSlashResultContainer()` - 重建 Slash 结果容器 UI
- `createSlashResultComponent()` - 创建带实时动画的结果组件
- `parseSubagentNotifyContent()` - 解析通知消息内容为结构化数据
- `SubagentControlNoticeComponent` - 控制通知组件类

---

### 2. 外部桥接注册（External Bridges）

#### 2.1 Slash Subagent Bridge
- **来源**: `./slash-bridge.ts`
- **作用**: 将 `/run`、`/chain`、`/parallel` 等 Slash 命令转换为子代理工具调用
- **机制**: 通过事件总线 `pi.events` 通信，解耦命令层和执行层

#### 2.2 Prompt Template Delegation Bridge
- **来源**: `./prompt-template-bridge.ts`
- **作用**: 支持在提示模板中使用 `{{#delegate agent="xxx"}}` 语法委托子代理
- **触发方**: `pi-prompt-template-model` 扩展（外部）
- **机制**: 监听 `PROMPT_TEMPLATE_SUBAGENT_REQUEST_EVENT` 事件

---

### 3. 核心工具（Core Tool）

注册 `subagent` 工具，这是 LLM 可调用的唯一接口：

```typescript
pi.registerTool({
    name: "subagent",
    label: "Subagent",
    description: "...",
    parameters: SubagentParams,  // 来自 ./schemas.ts
    execute: executor.execute,
    renderCall: (args, theme) => { ... },  // 自定义调用渲染
    renderResult: (result, options, theme, context) => { ... },  // 自定义结果渲染
});
```

**支持模式**:
- SINGLE: `{ agent, task }` - 单任务执行
- CHAIN: `{ chain: [...] }` - 链式执行
- PARALLEL: `{ tasks: [...] }` - 并行执行
- MANAGEMENT: `{ action: "list"|"get"|"create"|"update"|"delete" }` - Agent 管理
- CONTROL: `{ action: "status"|"interrupt" }` - 执行控制
- DIAGNOSTICS: `{ action: "doctor" }` - 诊断

---

### 4. Slash 命令（Slash Commands）

批量注册所有 `/subagent` 相关命令：
- `/run` - 运行单任务
- `/chain` - 运行链式任务
- `/run-chain` - 运行已保存的 chain
- `/parallel` - 并行运行
- `/agents` - Agent 管理界面
- `/subagents-status` - 查看异步作业状态
- `/subagents-doctor` - 诊断工具

**来源**: `./slash-commands.ts`

---

### 5. 通知系统（Notification System）

注册后台任务完成通知：

```typescript
registerSubagentNotify(pi);
```

**功能**:
- 监听 `SUBAGENT_ASYNC_COMPLETE_EVENT`
- 自动发送 `subagent-notify` 类型消息到 TUI
- 去重机制（10 分钟 TTL）
- 支持 completed/failed/paused 三种状态

**热重载处理**:
使用 `globalStore` 存储取消函数，避免重复订阅。

---

### 6. 事件订阅（Event Subscriptions）

#### 6.1 扩展间事件总线 (`pi.events`)
订阅 3 个内部事件：
- `SUBAGENT_ASYNC_STARTED_EVENT` → `handleStarted`（异步作业开始）
- `SUBAGENT_ASYNC_COMPLETE_EVENT` → `handleComplete`（异步作业完成）
- `SUBAGENT_CONTROL_EVENT` → `controlEventHandler`（控制事件）

#### 6.2 Pi 生命周期事件 (`pi.on`)
订阅 3 个生命周期事件：
- `tool_result` - 工具结果处理，更新 UI Widget
- `session_start` → `resetSessionState()` - 会话启动初始化
- `session_shutdown` - 会话关闭清理

**清理逻辑**:
取消所有订阅、停止监视器、清理定时器、销毁桥接、停止动画、清除 Widget。

---

### 7. 结果监视器（Result Watcher）

初始化文件系统监视器：

```typescript
const { startResultWatcher, primeExistingResults, stopResultWatcher } = createResultWatcher(
    pi, state, RESULTS_DIR, 10 * 60 * 1000
);
startResultWatcher();
primeExistingResults();
```

**作用**: 监视 `RESULTS_DIR` 目录变化，检测子代理输出文件变更。

---

### 8. 异步作业追踪器（Async Job Tracker）

创建异步作业追踪器：

```typescript
const { ensurePoller, handleStarted, handleComplete, resetJobs } = createAsyncJobTracker(
    pi, state, ASYNC_DIR
);
```

**返回函数用途**:
- `handleStarted` - 作业开始时添加到 `state.asyncJobs`
- `handleComplete` - 作业完成时更新状态
- `ensurePoller` - 确保轮询器运行，定期刷新 UI
- `resetJobs` - 会话切换时重置作业状态

---

### 9. 子代理执行器（Subagent Executor）

创建核心执行引擎：

```typescript
const executor = createSubagentExecutor({
    pi,
    state,
    config,           // 来自 config.json
    asyncByDefault,   // 默认是否异步
    tempArtifactsDir,
    getSubagentSessionRoot,  // 生成子会话目录
    expandTilde,            // 路径展开
    discoverAgents,         // Agent 发现
});
```

**被调用方**:
- Tool 的 `execute`
- Slash Bridge 的 `execute`
- Prompt Template Bridge 的 `execute`

---

## 辅助方法

### 配置加载

```typescript
function loadConfig(): ExtensionConfig
```

加载 `~/.pi/agent/extensions/subagent/config.json`，支持：
- `asyncByDefault` - 默认异步
- `maxSubagentDepth` - 最大嵌套深度
- `intercomBridge` - 进程间通信配置
- `worktreeSetupHook` - Git worktree 钩子

### 会话目录生成

```typescript
function getSubagentSessionRoot(parentSessionFile: string | null): string
```

从父会话路径派生子会话目录：
```
~/.pi/agent/sessions/abc123.jsonl
                    ↓
~/.pi/agent/sessions/abc123/{runId}/
```

### 状态管理

#### `resetSessionState(ctx)`
会话启动时调用，执行：
1. 设置 `baseCwd`
2. 生成/恢复 `currentSessionId`
3. 保存 `lastUiContext`
4. 清理旧产物
5. 重置异步作业
6. 恢复 Slash 快照

#### `session_shutdown` 处理
会话关闭时彻底清理：
- 取消所有事件订阅
- 停止结果监视器
- 停止轮询器
- 清理定时器
- 清空异步作业
- 销毁桥接
- 停止动画
- 清除 Widget

---

## 依赖关系

### 导入的外部模块

| 模块 | 来源 | 用途 |
|------|------|------|
| `@mariozechner/pi-agent-core` | Pi 核心 | `AgentToolResult` 类型 |
| `@mariozechner/pi-coding-agent` | Pi 核心 | `ExtensionAPI`, `ToolDefinition` 等 |
| `@mariozechner/pi-tui` | Pi TUI | `Text`, `Container`, `Box` 等组件 |

### 导入的内部模块

| 模块 | 文件 | 用途 |
|------|------|------|
| `discoverAgents` | `./agents.ts` | Agent 发现 |
| `createResultWatcher` | `./result-watcher.ts` | 结果监视器 |
| `createAsyncJobTracker` | `./async-job-tracker.ts` | 异步作业追踪 |
| `createSubagentExecutor` | `./subagent-executor.ts` | 子代理执行器 |
| `registerSlashSubagentBridge` | `./slash-bridge.ts` | Slash 命令桥接 |
| `registerPromptTemplateDelegationBridge` | `./prompt-template-bridge.ts` | 提示模板桥接 |
| `registerSubagentNotify` | `./notify.ts` | 通知系统 |
| `SubagentParams` | `./schemas.ts` | 工具参数 Schema |
| `renderSubagentResult` | `./render.ts` | 结果渲染 |

---

## 热重载支持

所有全局状态存储在 `globalThis`，支持 `/reload` 后恢复：

```typescript
const globalStore = globalThis as Record<string, unknown>;
const runtimeCleanupStoreKey = "__piSubagentRuntimeCleanup";
const eventUnsubscribeStoreKey = "__piSubagentEventUnsubscribes";
```

热重载时：
1. 调用旧的 `runtimeCleanup()` 清理资源
2. 取消旧的事件订阅
3. 重新初始化新实例

---

## 阅读笔记

- **阅读日期**: 2026-05-09
- **关键理解**: 
  - index.ts 是"注册中心"，不直接执行逻辑，而是组装各个组件
  - 通过事件总线实现松耦合（Slash Bridge、Prompt Template Bridge）
  - 热重载通过 globalThis 存储状态实现
  - 所有清理逻辑集中在 `session_shutdown`
- **待深入**: 
  - 子代理执行器的具体实现
  - 链式/并行执行的分发逻辑
  - 文件监视器的实现细节

---

## 相关文档

- [Pi Extensions 文档](../docs/extensions.md)
- [slash-bridge.md](./slash-bridge.md)
- [subagent-executor.md](./subagent-executor.md)
