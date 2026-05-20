# prompt-template-bridge.ts - 提示模板桥接详解

## 文件信息

- **文件名**: `prompt-template-bridge.ts`
- **大小**: 13K
- **位置**: 项目根目录
- **所属模块**: Core / 外部桥接
- **九大组件**: #2 外部桥接注册

---

## 文件作用

让提示模板中的 `{{delegate:agent}}` 能调用子代理。

通过 **Event Bus** 实现模板与子代理执行器的通信。

---

## 核心架构

```
提示模板 {{delegate:researcher}}
    │
    ├─► emit PROMPT_TEMPLATE_SUBAGENT_REQUEST_EVENT
    │        │
    │        ▼
    │   prompt-template-bridge.ts (监听)
    │        │
    │        ├─ 构建执行参数
    │        ├─ 调用 executor.execute()
    │        │       │
    │        │       ▼
    │        │   内部执行 (async)
    │        │       │
    │        │       ├─ 创建 Promise
    │        │       ├─ 监听 result-watcher
    │        │       └─ 等待完成
    │        │
    │        └─► emit PROMPT_TEMPLATE_SUBAGENT_RESPONSE_EVENT
    │
    ├─► emit PROMPT_TEMPLATE_SUBAGENT_CANCEL_EVENT
    │        │
    │        ▼
    │   prompt-template-bridge.ts
    │        │
    │        └─ 取消执行
    │
    ▼
监听 PROMPT_TEMPLATE_SUBAGENT_RESPONSE_EVENT
    返回结果给模板渲染器
```

---

## 监听的事件

### 1. PROMPT_TEMPLATE_SUBAGENT_REQUEST_EVENT

```typescript
pi.events.on(PROMPT_TEMPLATE_SUBAGENT_REQUEST_EVENT, async (request) => {
    const { requestId, agent, task, context, model, cwd, worktree } = request;
    
    // 调用执行器（异步模式）
    const result = await executor.execute(
        requestId,
        {
            agent,
            task,
            context,
            model,
            cwd,
            async: true,  // 强制异步
            worktree,
        },
        signal,
        undefined,  // 无进度回调
        ctx,
    );
    
    // 发出完成事件
    pi.events.emit(PROMPT_TEMPLATE_SUBAGENT_RESPONSE_EVENT, response);
});
```

### 2. PROMPT_TEMPLATE_SUBAGENT_CANCEL_EVENT

```typescript
pi.events.on(PROMPT_TEMPLATE_SUBAGENT_CANCEL_EVENT, (data) => {
    const { requestId } = data;
    const task = activeTasks.get(requestId);
    if (task) {
        task.cancel();
        activeTasks.delete(requestId);
    }
});
```

---

## 发出的事件

### 1. PROMPT_TEMPLATE_SUBAGENT_STARTED_EVENT

```typescript
pi.events.emit(PROMPT_TEMPLATE_SUBAGENT_STARTED_EVENT, { requestId });
```

**触发时机**: 开始执行时

### 2. PROMPT_TEMPLATE_SUBAGENT_RESPONSE_EVENT

```typescript
pi.events.emit(PROMPT_TEMPLATE_SUBAGENT_RESPONSE_EVENT, {
    requestId,
    agent,
    task,
    messages,       // Agent 消息记录
    isError,
    errorText,
});
```

**触发时机**: 执行完成

### 3. PROMPT_TEMPLATE_SUBAGENT_UPDATE_EVENT

```typescript
pi.events.emit(PROMPT_TEMPLATE_SUBAGENT_UPDATE_EVENT, {
    requestId,
    currentTool,
    currentToolArgs,
    recentOutput,
    recentTools,
    model,
    toolCount,
    durationMs,
    tokens,
    taskProgress,
});
```

**触发时机**: 进度更新（用于模板内显示）

---

## 与 slash-bridge 的区别

| 特性 | slash-bridge.ts | prompt-template-bridge.ts |
|------|-----------------|---------------------------|
| **触发方式** | Slash 命令 `/subagent` | 提示模板 `{{delegate:agent}}` |
| **调用方** | 用户手动输入 | Agent 自动触发 |
| **参数** | 完整 SubagentParams | 简化的 agent/task/model |
| **进度更新** | 实时 | 批量/间隔 |
| **使用场景** | 用户主动触发 | Agent 委托子任务 |
| **执行模式** | 同步/异步可选 | 强制异步 |

---

## 核心设计

| 设计点 | 说明 |
|--------|------|
| **异步执行** | 模板委托必须异步，不阻塞主 Agent |
| **消息记录** | 返回完整的 Agent 消息历史 |
| **进度聚合** | 批量返回工具调用和输出 |
| **取消机制** | 支持取消长时间运行的委托任务 |

---

## 一句话总结

**`prompt-template-bridge` 是"委托代理"：让提示模板通过 `{{delegate:agent}}` 触发子代理执行，强制异步，返回消息历史和进度，实现 Agent 自动委托子任务的能力！**
