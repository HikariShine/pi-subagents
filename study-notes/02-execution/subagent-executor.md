# subagent-executor.ts - 子代理执行器详解

## 文件信息

- **文件名**: `subagent-executor.ts`
- **大小**: 65K
- **位置**: 项目根目录
- **所属模块**: Execution / 执行层

## 文件作用

`subagent-executor.ts` 是 `pi-subagents` 插件的**核心执行引擎**，负责创建和运行子代理进程。它是所有子代理操作的统一入口，支持多种执行模式（单任务、链式、并行、异步）。

---

## createSubagentExecutor - 执行器工厂

### 函数签名

```typescript
export function createSubagentExecutor(deps: ExecutorDeps): {
    execute: (
        id: string,
        params: SubagentParamsLike,
        signal: AbortSignal,
        onUpdate: ((r: AgentToolResult<Details>) => void) | undefined,
        ctx: ExtensionContext,
    ) => Promise<AgentToolResult<Details>>;
}
```

### 参数详解（ExecutorDeps）

| 参数 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `pi` | `ExtensionAPI` | index.ts 传入 | Pi 扩展 API，用于事件通信 |
| `state` | `SubagentState` | index.ts 的 state | 全局状态存储（热重载兼容） |
| `config` | `ExtensionConfig` | loadConfig() | 用户配置（config.json） |
| `asyncByDefault` | `boolean` | config + 计算 | 默认是否异步执行 |
| `tempArtifactsDir` | `string` | index.ts 计算 | 临时产物目录路径 |
| `getSubagentSessionRoot` | `function` | index.ts 定义 | 从父会话派生子会话目录 |
| `expandTilde` | `function` | index.ts 定义 | 展开 `~` 为用户主目录 |
| `discoverAgents` | `function` | ./agents.ts | 发现并加载 Agent 配置 |

### 返回值

```typescript
{
    execute: (id, params, signal, onUpdate, ctx) => Promise<AgentToolResult<Details>>
}
```

---

## execute 函数 - 14步执行流程

### 整体流程图

```
execute(id, params, signal, onUpdate, ctx)
    │
    ├─ 1. 初始化状态（baseCwd, foregroundControls）
    │
    ├─ 2. 解析 cwd（resolveRequestedCwd）
    │
    ├─ 3. Action 分支判断
    │   ├─ doctor → 返回诊断报告
    │   ├─ status → 查询运行状态
    │   ├─ interrupt → 中断任务
    │   └─ list/get/create/update/delete → 管理操作
    │
    ├─ 4. 嵌套深度检查（checkSubagentDepth）
    │
    ├─ 5. 参数规范化（normalizeRepeatedParallelCounts）
    │   └─ 展开 count 参数为重复任务
    │
    ├─ 6. 应用强制异步覆盖（applyForceTopLevelAsyncOverride）
    │
    ├─ 7. 确定执行作用域（resolveExecutionAgentScope）
    │
    ├─ 8. 发现可用 Agent（discoverAgents）
    │
    ├─ 9. 配置 Intercom 桥接（resolveIntercomBridge）
    │   └─ applyIntercomBridgeToAgent（添加通信工具）
    │
    ├─ 10. 生成运行 ID（randomUUID().slice(0, 8)）
    │
    ├─ 11. 确定执行模式
    │   ├─ hasChain → chain 模式
    │   ├─ hasTasks → parallel 模式
    │   └─ hasSingle → single 模式
    │
    ├─ 12. 验证输入参数（validateExecutionInput）
    │
    ├─ 13. 创建会话目录（sessionRoot）
    │   ├─ 用户指定 → 使用指定目录
    │   ├─ 配置 defaultSessionDir → 使用配置目录
    │   └─ 默认 → getSubagentSessionRoot 派生
    │
    ├─ 14. 创建 Fork 上下文解析器（createForkContextResolver）
    │   └─ context: "fork" 时创建分支会话
    │
    ├─ 确定异步模式（effectiveAsync）
    │   └─ requestedAsync + clarify 冲突检查
    │
    ├─ 配置控制参数（controlConfig）
    │
    ├─ 配置产物参数（artifactConfig, artifactsDir）
    │   ├─ 异步 → tempArtifactsDir
    │   └─ 同步 → getArtifactsDir(parentSessionFile)
    │
    └─ 执行分发（try-finally 块）
        ├─ runAsyncPath → 异步执行
        ├─ runChainPath → 链式执行
        ├─ runParallelPath → 并行执行
        └─ runSinglePath → 单任务执行
```

---

## 关键辅助方法详解

### 1. checkSubagentDepth - 嵌套深度检查

**功能**：防止子代理无限递归调用

**机制**：
- 通过环境变量 `PI_SUBAGENT_DEPTH` 追踪当前嵌套层级
- 每启动一层子代理，深度 +1
- 超过 `maxSubagentDepth`（默认2）时阻止调用

**环境变量传递**：
```typescript
// 启动子进程时注入
env: {
    PI_SUBAGENT_DEPTH: String(parentDepth + 1),
    PI_SUBAGENT_MAX_DEPTH: String(maxDepth),
}
```

**最大深度优先级**：
1. `PI_SUBAGENT_MAX_DEPTH` 环境变量
2. `config.maxSubagentDepth` 配置
3. `DEFAULT_SUBAGENT_MAX_DEPTH`（=2）

---

### 2. normalizeRepeatedParallelCounts - 并行计数展开

**功能**：将 `count` 参数展开为重复的独立任务

**示例**：
```typescript
// 输入
{ agent: "reviewer", task: "检查", count: 3 }

// 展开后
[
    { agent: "reviewer", task: "检查" },
    { agent: "reviewer", task: "检查" },
    { agent: "reviewer", task: "检查" }
]
// count 字段被移除
```

**语法**：
```typescript
const { count, ...concreteTask } = task;
for (let repeat = 0; repeat < (rawCount ?? 1); repeat++) {
    expanded.push({ ...concreteTask });
}
```

**支持的参数位置**：
- `params.tasks`（顶层并行任务）
- `params.chain[].parallel`（链式中的并行步骤）

---

### 3. applyForceTopLevelAsyncOverride - 强制顶层异步

**功能**：强制顶层任务异步执行（无视用户设置）

**触发条件**：
- `depth === 0`（顶层调用）
- `forceTopLevelAsync === true`（配置开启）

**行为**：
```typescript
if (depth === 0 && forceTopLevelAsync) {
    return { ...params, async: true, clarify: false };
}
```

**为什么设置 `clarify: false`**：
- `clarify` 需要前台交互（暂停询问）
- `async` 是后台运行（无交互）
- 两者互斥，优先保证异步执行

---

### 4. applyIntercomBridgeToAgent - 应用 Intercom 桥接

**功能**：为 Agent 添加进程间通信能力

**修改内容**：
1. **Tools**：添加 `"intercom"` 工具
2. **System Prompt**：追加 Intercom 通信指令

**安全检查**（extensionSandboxAllowsIntercom）：
```typescript
// 检查 Agent 的 extensions 配置是否允许使用 Intercom
if (!extensionSandboxAllowsIntercom(agent.extensions, bridge.extensionDir)) {
    return agent;  // 不允许，保持原配置
}
```

**指令模板**：
```
To communicate with the orchestrator, use the intercom tool with target "{orchestratorTarget}".
```

---

### 5. createForkContextResolver - Fork 上下文解析器

**功能**：为 `context: "fork"` 模式创建分支会话

**两种模式对比**：

| 模式 | `fresh` | `fork` |
|------|---------|--------|
| 上下文 | 全新会话，无历史 | 继承父会话历史 |
| 使用场景 | 独立任务 | 需要上下文的任务 |
| 实现方式 | 创建新会话文件 | 从父会话分支 |

**Git 类比**：
```
parent.jsonl
    │
    ├── branch-0.jsonl  ← 第1个子代理（fork）
    ├── branch-1.jsonl  ← 第2个子代理（fork）
    └── ...
```

**使用方式**：
```typescript
const sessionFileForIndex = createForkContextResolver(
    ctx.sessionManager,
    effectiveParams.context  // "fresh" | "fork"
).sessionFileForIndex;

// fork 模式：返回分支会话文件路径
// fresh 模式：返回 undefined（创建新会话）
```

---

### 6. withForkContext - Fork 上下文标记

**功能**：给执行结果添加 `context: "fork"` 标记

**代码**：
```typescript
function withForkContext(result, context) {
    if (context !== "fork" || !result.details) return result;
    return {
        ...result,
        details: { ...result.details, context: "fork" },
    };
}
```

**用途**：标识结果来自 Fork 分支，用于后续识别和特殊处理。

---

## 执行路径对比

| 路径 | 触发条件 | 特点 | 阻塞性 |
|------|---------|------|--------|
| **runAsyncPath** | `effectiveAsync === true` | 后台运行，立即返回 runId | 非阻塞 |
| **runChainPath** | `hasChain === true` | 按顺序执行多个步骤 | 阻塞 |
| **runParallelPath** | `hasTasks === true` | 同时执行多个任务 | 阻塞 |
| **runSinglePath** | `hasSingle === true` | 执行单个 Agent | 阻塞 |

### 异步 vs 同步的区别

**异步（runAsyncPath）**：
- 启动子进程后立即返回
- 结果通过 `result-watcher` 文件监视器检测
- 状态通过 `async-job-tracker` 追踪
- 支持跨会话查看结果

**同步（其他三个）**：
- 等待子进程完成
- 实时返回执行结果
- 通过 `foregroundControls` 注册控制状态
- 支持中断（`Ctrl+C`）

---

## 会话目录确定逻辑

### 优先级（从高到低）

```
sessionRoot 确定：
    │
    ├─ 1. 用户指定（params.sessionDir）
    │   └─ path.resolve(expandTilde(sessionDir))
    │
    ├─ 2. 配置指定（config.defaultSessionDir）
    │   └─ path.resolve(expandTilde(defaultSessionDir))
    │
    └─ 3. 从父会话派生（getSubagentSessionRoot）
        ├─ 有 parentSessionFile
        │   └─ path.dirname(parentSessionFile) + basename(parentSessionFile, ".jsonl")
        └─ 无 parentSessionFile
            └─ mkdtempSync(os.tmpdir() + "pi-subagent-session-")

最终：sessionRoot = path.join(baseSessionRoot, runId)
```

### 产物目录确定

| 模式 | 产物目录 | 说明 |
|------|---------|------|
| **异步** | `deps.tempArtifactsDir` | `~/.pi/agent/extensions/subagent/results/temp/` |
| **同步** | `getArtifactsDir(parentSessionFile)` | 会话目录下的 `subagent-artifacts/` |

---

## 阅读笔记

- **阅读日期**: 2026-05-11 ~ 2026-05-12
- **关键理解**:
  - `execute` 是统一的执行入口，通过参数分发到不同路径
  - 嵌套深度通过环境变量 `PI_SUBAGENT_DEPTH` 传递和检查
  - `count` 参数在内部被展开为重复任务，简化后续处理
  - `fork` 模式通过 `createForkContextResolver` 实现会话分支
  - Intercom 桥接通过修改 Agent 的 tools 和 systemPrompt 实现
  - 异步和同步使用不同的目录策略（临时 vs 会话绑定）
- **待深入**:
  - `runSinglePath` 的具体实现（子进程启动、通信、结果收集）
  - `runChainPath` 的链式调度逻辑
  - `runParallelPath` 的并发控制
  - `runAsyncPath` 的后台进程管理

---

## 相关文档

- [index.md](../01-core/index.md) - 插件入口和整体架构
- [agents.md](../03-agents/agents.md) - Agent 发现和配置
- [intercom-bridge.md