# index.ts - 核心工具（subagent）详解

## 文件信息

- **文件名**: `index.ts`（核心工具部分）
- **所属模块**: Tools / 核心工具
- **九大组件**: #3 核心工具（1个）

---

## 核心工具定义

```typescript
const tool: ToolDefinition<typeof SubagentParams, Details> = {
    name: "subagent",
    label: "Subagent",
    description: "...",
    parameters: SubagentParams,
    
    execute(id, params, signal, onUpdate, ctx) {
        return executor.execute(id, params, signal, onUpdate, ctx);
    },
    
    renderCall(args, theme) { ... },
    renderResult(result, options, theme, context) { ... },
};

pi.registerTool(tool);
```

---

## 4 种执行模式

| 模式 | 参数 | 说明 | 示例 |
|------|------|------|------|
| **SINGLE** | `{ agent, task?, context? }` | 单任务执行 | `{ agent: "researcher", task: "分析代码" }` |
| **CHAIN** | `{ chain: [...], context? }` | 链式执行 | `{ chain: [{agent:"a"}, {agent:"b"}] }` |
| **PARALLEL** | `{ tasks: [...], concurrency?, worktree? }` | 并行执行 | `{ tasks: [...], concurrency: 3 }` |
| **MANAGEMENT** | `{ action: "list"/"get"/"create"/... }` | 管理操作 | `{ action: "list" }` |

---

## 执行入口

```typescript
execute(id, params, signal, onUpdate, ctx) {
    return executor.execute(id, params, signal, onUpdate, ctx);
}
//           │
//           ▼
//    subagent-executor.execute()
//           │
//           ├─ 9种执行路径
//           ├─ runSync / runAsyncPath / runChainPath ...
//           └─ ...
```

---

## renderCall - 调用摘要渲染

**作用**: LLM 调用工具时，显示简短摘要

```typescript
renderCall(args, theme) {
    // 管理操作
    if (args.action) {
        return new Text(`subagent ${args.action}`);
    }
    
    // 链式
    if (args.chain?.length) {
        return new Text(`subagent chain (${args.chain.length})`);
    }
    
    // 并行
    if (isParallel) {
        return new Text(`subagent parallel (${count})`);
    }
    
    // 单任务
    return new Text(`subagent ${args.agent}`);
}
```

**显示效果**:
- `{ action: "list" }` → `subagent list`
- `{ agent: "researcher" }` → `subagent researcher`
- `{ chain: [...] }` → `subagent chain (3)`
- `{ tasks: [...] }` → `subagent parallel (6)`

---

## renderResult - 结果渲染

**作用**: 执行完成后，显示详细结果

```typescript
renderResult(result, options, theme, context) {
    syncResultAnimation(result, context);
    return renderSubagentResult(result, options, theme);
}
```

**显示效果**:

### 单任务成功
```
✓ researcher · 12s
北京明天晴，25°C
```

### 单任务失败
```
✗ worker · 5s · failed

Error: API timeout after 30s
```

### 链式结果
```
✓ chain (3 steps) · 45s

Step 1: scout · 10s
  Found 3 sources

Step 2: planner · 15s
  Created plan

Step 3: worker · 20s
  ✓ completed
```

### 并行结果
```
✓ parallel (3 tasks) · 30s

Task 1: analyzer-a · 10s · ok
Task 2: analyzer-b · 12s · ok
Task 3: reviewer · 30s · ok
```

---

## 与 Slash 命令的区别

| | 核心工具 | Slash 命令 |
|--|---------|-----------|
| **调用方** | LLM 自动调用 | 用户手动输入 |
| **入口** | `pi.registerTool()` | `pi.registerCommand()` |
| **参数** | 结构化 JSON | 文本解析 |
| **执行** | `executor.execute()` | `runSlashSubagent()` |
| **结果** | Agent 继续处理 | 直接显示给用户 |

---

## 一句话总结

**核心工具是 LLM 的调用入口：定义 `subagent` 工具，4 种执行模式（SINGLE/CHAIN/PARALLEL/MANAGEMENT），`execute()` 调用 `subagent-executor.execute()`，`renderCall` 显示调用摘要，`renderResult` 显示详细结果，实现 LLM 自动触发子代理执行！**
