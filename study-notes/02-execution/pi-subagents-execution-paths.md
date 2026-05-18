# pi-subagents 执行路径完整分析

> 文档版本: 2026-05-18
> 基于 pi-subagents 源码分析

---

## 目录

1. [概述](#概述)
2. [入口函数](#入口函数)
3. [九种执行路径](#九种执行路径)
4. [关键函数说明](#关键函数说明)
5. [同步 vs 异步对比](#同步-vs-异步对比)
6. [已知问题](#已知问题)

---

## 概述

`createSubagentExecutor` 是 pi-subagents 的核心执行器工厂函数，返回的 `execute` 方法根据参数自动选择 9 种不同的执行路径。

---

## 入口函数

### createSubagentExecutor

```typescript
// 位置: subagent-executor.ts
export function createSubagentExecutor(
    agents: AgentConfig[],
    ctx: ExtensionContext,
    options?: { maxOutput?: number; artifactsDir?: string }
): SubagentExecutor {
    return {
        execute: async (params: SubagentParamsLike) => {
            // 参数解析和分发逻辑
            // ...
        }
    };
}
```

### execute 参数结构

```typescript
interface SubagentParamsLike {
    // 执行模式（互斥）
    agent?: string;           // 单任务模式
    chain?: ChainStep[];      // 链式模式
    tasks?: SubagentStep[];   // 并行模式
    
    // 通用参数
    task?: string;            // 任务模板
    cwd?: string;             // 工作目录
    clarify?: boolean;        // 是否显示澄清UI
    async?: boolean;          // 是否强制异步
    context?: "fresh" | "fork"; // 执行上下文
    worktree?: boolean;       // 是否启用worktree隔离
    
    // 其他参数...
    output?: string;
    reads?: string[];
    skills?: string[];
    model?: string;
    // ...
}
```

---

## 九种执行路径

### 路径 1: 直接异步单任务

**触发条件**: `effectiveAsync === true && hasSingle === true`

**调用链**:
```
execute()
  └─ runAsyncPath()
       └─ executeAsyncSingle(id, params)
            └─ spawnRunner(cfg, suffix, cwd)
                 └─ subagent-runner.ts (独立进程)
```

**特点**:
- 立即分离进程 (`detached: true`)
- 返回 `{ pid }`，不等待结果
- 通过 `SUBAGENT_ASYNC_STARTED_EVENT` 通知

---

### 路径 2: 直接异步链式

**触发条件**: `effectiveAsync === true && hasChain === true`

**调用链**:
```
execute()
  └─ runAsyncPath()
       └─ executeAsyncChain(id, { chain, ... })
            └─ spawnRunner(cfg, id, cwd)
                 └─ subagent-runner.ts (多步骤顺序执行)
```

**特点**:
- 支持多步骤顺序执行
- 支持并行组嵌套
- `{previous}` 运行时动态替换

---

### 路径 3: 直接异步并行

**触发条件**: `effectiveAsync === true && hasTasks === true`

**调用链**:
```
execute()
  └─ runAsyncPath()
       └─ executeAsyncChain(id, {
              chain: [{ 
                  parallel: tasks, 
                  concurrency, 
                  worktree 
              }]
          })
            └─ spawnRunner()
                 └─ subagent-runner.ts
```

**特点**:
- 内部包装为单步骤并行组
- 复用 `executeAsyncChain` 逻辑

---

### 路径 4: 同步单任务

**触发条件**: `!effectiveAsync && hasSingle && !shouldClarify`

**调用链**:
```
execute()
  └─ runSinglePath()
       └─ runSync({ ... })
            └─ runSingleAttempt()
                 └─ spawn("pi", args, { stdio: ["ignore","pipe","pipe"] })
```

**特点**:
- 阻塞执行
- 实时输出通过 `proc.stdout`
- 支持 `intercom` 实时通信

---

### 路径 5: 澄清转异步单任务

**触发条件**: `!effectiveAsync && hasSingle && clarify === true && 用户选后台`

**调用链**:
```
execute()
  └─ runSinglePath()
       └─ executeAsyncSingle(id, { ... })
            └─ spawnRunner()
```

**关键点**:
```typescript
// runSinglePath 中的判断
if (singleResult.requestedAsync) {
    return executeAsyncSingle(id, { ... });  // 转异步
}
```

---

### 路径 6: 同步链式

**触发条件**: `!effectiveAsync && hasChain && !shouldClarify`

**调用链**:
```
execute()
  └─ runChainPath()
       └─ executeChain()
            └─ runChain()
                 ├─ 遍历 steps
                 │    ├─ 串行: runSync()
                 │    └─ 并行: runParallelChainTasks()
                 │              └─ mapConcurrent() → runSync()
                 └─ 聚合结果
```

**特点**:
- 阻塞顺序执行
- `{task}`、`{previous}` 调用前替换
- 支持并行组 (`isParallelStep`)

---

### 路径 7: 澄清转异步链式

**触发条件**: `!effectiveAsync && hasChain && clarify === true && 用户选后台`

**调用链**:
```
execute()
  └─ runChainPath()
       └─ executeChain()
            │   ├─ 显示 Clarify UI
            │   └─ return { requestedAsync: { chain, chainSkills } }
            │
            └─ if (chainResult.requestedAsync)
                    └─ executeAsyncChain(id, { ... })
                         └─ spawnRunner()
```

---

### 路径 8: 同步并行

**触发条件**: `!effectiveAsync && hasTasks && !shouldClarify`

**调用链**:
```
execute()
  └─ runParallelPath()
       └─ runForegroundParallelTasks()
            ├─ writeInitialProgressFile()
            └─ mapConcurrent(tasks, concurrency, async (task) => {
                   return runSync({ ... });
               })
```

**特点**:
- 纯并行执行，无顺序依赖
- 支持 worktree 隔离
- 结果聚合格式化输出

---

### 路径 9: 澄清转异步并行

**触发条件**: `!effectiveAsync && hasTasks && clarify === true && 用户选后台`

**调用链**:
```
execute()
  └─ runParallelPath()
       ├─ 显示 Clarify UI
       └─ executeAsyncChain(id, {
              chain: [{ parallel: tasks, concurrency, worktree }]
          })
            └─ spawnRunner()
```

---

## 关键函数说明

### 同步执行核心

| 函数 | 位置 | 作用 |
|------|------|------|
| `runSync` | `execution.ts` | 同步单任务执行 |
| `runSingleAttempt` | `execution.ts` | 底层 spawn 创建子进程 |
| `runChain` | `chain-execution.ts` | 同步链式循环执行 |
| `runParallelChainTasks` | `chain-execution.ts` | 链内并行执行 |
| `runForegroundParallelTasks` | `subagent-executor.ts` | 前台纯并行执行 |

### 异步执行核心

| 函数 | 位置 | 作用 |
|------|------|------|
| `executeAsyncSingle` | `async-execution.ts` | 异步单任务启动 |
| `executeAsyncChain` | `async-execution.ts` | 异步链式启动 |
| `spawnRunner` | `subagent-executor.ts` | 分离子进程启动器 |
| `runSubagent` | `subagent-runner.ts` | 异步子进程主函数 |

### 工具函数

| 函数 | 位置 | 作用 |
|------|------|------|
| `buildSeqStep` | `async-execution.ts` | 构建步骤完整配置 |
| `buildStepOverrides` | `async-execution.ts` | 提取步骤级覆盖配置 |
| `mapConcurrent` | `parallel-utils.ts` | 并发工作池调度 |
| `aggregateParallelOutputs` | `parallel-utils.ts` | 并行结果聚合 |
| `createWorktrees` | `worktree.ts` | Git worktree 隔离 |

---

## 同步 vs 异步对比

| 特性 | 同步执行 | 异步执行 |
|------|----------|----------|
| **进程模式** | `detached: false` | `detached: true` |
| **通信方式** | `proc.stdout` 实时流 | 文件系统 (status.json) |
| **占位符替换** | 调用前完成所有替换 | `{task}` 构建时替换，`{previous}` 运行时替换 |
| **结果获取** | 阻塞等待返回 | 立即返回 PID，轮询查询 |
| **Intercom** | 实时 JSONL 流 | 不支持（或受限）|
| **适用场景** | 交互式、短任务 | 后台、长任务、批量 |

---

## 已知问题

### Bug: 异步链式 `{task}` 占位符未替换

**位置**: `async-execution.ts` - `buildSeqStep`

**描述**:
```typescript
// buildSeqStep 中只处理了 {previous}
const task = injectSingleOutputInstruction(
    `${readInstructions.prefix}${s.task ?? "{previous}"}${progressInstructions.suffix}`,
    outputPath
);
// ❌ {task} 未替换！
```

**影响**:
- 异步链式执行时，第一步任务中的 `{task}` 保持原样
- Agent 无法获取用户原始输入
- 导致任务执行偏离预期

**修复建议**:
在 `buildSeqStep` 中增加 `{task}` 替换逻辑，或在 `subagent-runner.ts` 中补充替换。

---

## 附录: 调用关系图

```
createSubagentExecutor
    └── execute(params)
        │
        ├─ effectiveAsync? ──────────────┐
        │                                 │
        │   ├─ hasSingle ──► executeAsyncSingle ──► spawnRunner
        │   ├─ hasChain ───► executeAsyncChain ───► spawnRunner
        │   └─ hasTasks ───► executeAsyncChain ───► spawnRunner
        │                        (包装为并行组)
        │
        └─ !effectiveAsync
            │
            ├─ hasSingle ──► runSinglePath
            │                    ├─ clarify? ──► 返回 requestedAsync ──► executeAsyncSingle
            │                    └─ !clarify ──► runSync
            │
            ├─ hasChain ───► runChainPath
            │                    ├─ clarify? ──► 返回 requestedAsync ──► executeAsyncChain
            │                    └─ !clarify ──► executeChain ──► runChain
            │                              ├─ 串行: runSync
            │                              └─ 并行: runParallelChainTasks
            │
            └─ hasTasks ──► runParallelPath
                                 ├─ clarify? ──► executeAsyncChain
                                 └─ !clarify ──► runForegroundParallelTasks
                                                  └─ mapConcurrent ──► runSync
```

---

*文档完成时间: 2026-05-18*
