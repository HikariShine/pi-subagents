# slash-bridge.ts - Slash 命令桥接详解

## 文件信息

- **文件名**: `slash-bridge.ts`
- **大小**: 5K
- **位置**: 项目根目录
- **所属模块**: Core / 外部桥接
- **九大组件**: #2 外部桥接注册

---

## 文件作用

让其他扩展（如 `/subagent` Slash 命令）能调用 `pi-subagents` 的执行器。

通过 **Event Bus** 实现跨扩展通信。

---

## 核心架构

```
外部 Slash 命令 (/subagent)
    │
    ├─► emit SLASH_SUBAGENT_REQUEST_EVENT
    │        │
    │        ▼
    │   slash-bridge.ts (监听)
    │        │
    │        ├─ 创建 AbortController
    │        ├─ 检查 pendingCancels
    │        │
    │        ├─► emit SLASH_SUBAGENT_STARTED_EVENT
    │        │
    │        ├─ 调用 options.execute()
    │        │       │
    │        │       ├─ 进度回调 onUpdate()
    │        │       │       │
    │        │       │       └─► emit SLASH_SUBAGENT_UPDATE_EVENT
    │        │       │
    │        │       └─ 返回 result
    │        │
    │        └─► emit SLASH_SUBAGENT_RESPONSE_EVENT
    │
    ├─► emit SLASH_SUBAGENT_CANCEL_EVENT
    │        │
    │        ▼
    │   slash-bridge.ts (监听)
    │        │
    │        └─ 调用 controller.abort()
    │
    ▼
监听 SLASH_SUBAGENT_RESPONSE_EVENT
    返回结果给 Slash 命令
```

---

## 监听的事件

### 1. SLASH_SUBAGENT_REQUEST_EVENT

```typescript
subscribe(SLASH_SUBAGENT_REQUEST_EVENT, async (data) => {
    const { requestId, params } = data;
    
    // 创建 AbortController 支持取消
    const controller = new AbortController();
    controllers.set(requestId, controller);
    
    // 发出开始事件
    options.events.emit(SLASH_SUBAGENT_STARTED_EVENT, { requestId });
    
    // 调用内部执行器
    const result = await options.execute(
        requestId,
        params,
        controller.signal,  // 取消信号
        (update) => {
            // 进度回调
            const payload: SlashSubagentUpdate = {
                requestId,
                progress: update.details?.progress,
                currentTool: first?.currentTool,
                toolCount: first?.toolCount,
            };
            options.events.emit(SLASH_SUBAGENT_UPDATE_EVENT, payload);
        },
        ctx,
    );
    
    // 发出完成事件
    options.events.emit(SLASH_SUBAGENT_RESPONSE_EVENT, response);
});
```

### 2. SLASH_SUBAGENT_CANCEL_EVENT

```typescript
subscribe(SLASH_SUBAGENT_CANCEL_EVENT, (data) => {
    const { requestId } = data;
    
    // 获取控制器并取消
    const controller = controllers.get(requestId);
    if (controller) {
        controller.abort();
        return;
    }
    
    // 如果还没开始，加入待取消队列
    pendingCancels.add(requestId);
});
```

---

## 发出的事件

### 1. SLASH_SUBAGENT_STARTED_EVENT

```typescript
options.events.emit(SLASH_SUBAGENT_STARTED_EVENT, { requestId });
```

**触发时机**: 开始执行时

### 2. SLASH_SUBAGENT_UPDATE_EVENT

```typescript
options.events.emit(SLASH_SUBAGENT_UPDATE_EVENT, {
    requestId,
    progress,       // 进度详情
    currentTool,    // 当前工具
    toolCount,      // 工具计数
});
```

**触发时机**: 执行过程中，进度更新时

### 3. SLASH_SUBAGENT_RESPONSE_EVENT

```typescript
options.events.emit(SLASH_SUBAGENT_RESPONSE_EVENT, {
    requestId,
    result,         // 执行结果
    isError,        // 是否错误
    errorText,      // 错误信息
});
```

**触发时机**: 执行完成（成功或失败）

---

## 核心设计

| 设计点 | 说明 |
|--------|------|
| **AbortController** | 支持取消执行 |
| **pendingCancels** | 处理"取消早于开始"的竞态条件 |
| **进度回调** | 实时反馈执行进度 |
| **错误处理** | try-catch 捕获异常并返回 |

---

## 与内部执行器的关系

```
slash-bridge.ts
    │
    ├─ 监听外部 Slash 命令请求
    │
    ├─ 调用 options.execute()
    │       │
    │       ▼
    │   subagent-executor.execute()
    │       │
    │       ├─ runSync (同步)
    │       ├─ executeAsyncSingle (异步单任务)
    │       ├─ executeAsyncChain (异步链式)
    │       └─ ... (9种执行路径)
    │
    └─ 将结果 emit 回 Slash 命令
```

---

## 一句话总结

**`slash-bridge` 是"转发器"：监听 Slash 命令的 REQUEST/CANCEL，转发给内部 `execute()`，通过 STARTED/UPDATE/RESPONSE 事件返回执行状态和结果，实现跨扩展调用！**
