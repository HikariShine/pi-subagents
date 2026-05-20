# index.ts - 事件订阅（Event Bus）详解

## 文件信息

- **文件名**: `index.ts`
- **所属模块**: Core / 事件系统
- **相关组件**: 九大组件之六 - 事件订阅

---

## 核心作用

`pi.events` 是 Pi 扩展系统的**全局事件总线**，实现：

1. **跨组件通信** - 解耦扩展内部模块
2. **跨进程通信** - 父子进程事件转发
3. **生命周期管理** - 会话开始/关闭事件
4. **工具集成** - 工具执行完成事件

---

## 事件类型分类

| 类型 | 事件名 | 来源 | 作用 |
|------|--------|------|------|
| **生命周期** | `session_start` | Pi 核心 | 新会话开始，重置状态 |
| **生命周期** | `session_shutdown` | Pi 核心 | 会话关闭，清理资源 |
| **工具** | `tool_result` | Pi 核心 | 工具执行完成，更新UI |
| **自定义** | `SUBAGENT_ASYNC_STARTED_EVENT` | pi-subagents | 异步作业启动 |
| **自定义** | `SUBAGENT_ASYNC_COMPLETE_EVENT` | pi-subagents | 异步作业完成 |
| **自定义** | `SUBAGENT_CONTROL_EVENT` | pi-subagents | 控制事件（needs_attention等）|

---

## 事件订阅详解

### 1. 生命周期事件

```typescript
// 会话开始 - 重置状态
pi.on("session_start", (_event, ctx) => {
    resetSessionState(ctx);
});

// 会话关闭 - 清理资源
pi.on("session_shutdown", () => {
    // 取消事件订阅
    for (const unsubscribe of eventUnsubscribes) {
        unsubscribe();
    }
    // 停止监视器
    stopResultWatcher();
    stopWidgetAnimation();
    // 清理计时器
    state.cleanupTimers.forEach(clearTimeout);
    state.cleanupTimers.clear();
    state.asyncJobs.clear();
});
```

---

### 2. 工具执行事件

```typescript
pi.on("tool_result", (event, ctx) => {
    if (event.toolName !== "subagent") return;
    if (!ctx.hasUI) return;
    
    // 更新UI状态显示
    state.lastUiContext = ctx;
    if (state.asyncJobs.size > 0) {
        renderWidget(ctx, Array.from(state.asyncJobs.values()));
        ensurePoller();  // 确保轮询器运行
    }
});
```

**作用**: 每次子代理工具执行后，更新侧边栏状态Widget。

---

### 3. 自定义事件订阅

```typescript
// 创建订阅列表（用于批量取消）
const eventUnsubscribes = [
    // 异步作业启动
    pi.events.on(SUBAGENT_ASYNC_STARTED_EVENT, handleStarted),
    
    // 异步作业完成
    pi.events.on(SUBAGENT_ASYNC_COMPLETE_EVENT, handleComplete),
    
    // 控制事件（子进程needs_attention等）
    pi.events.on(SUBAGENT_CONTROL_EVENT, controlEventHandler),
];

// 存储到全局，会话关闭时统一取消
globalStore[eventUnsubscribeStoreKey] = eventUnsubscribes;
```

**事件流向**:

```
subagent-runner.ts (子进程)
    │
    ├─ 写入 events.jsonl
    │   {"type": "subagent.control", "event": {...}}
    │
    └─ pi.events.emit("subagent.control", ...)
           │
           ▼
    async-job-tracker.emitNewControlEvents()
           │
           ▼
    pi.events.emit(SUBAGENT_CONTROL_EVENT, {...})
           │
           ▼
    index.ts controlEventHandler(data)
           │
           ├─ 发送消息到UI
           └─ 可选: intercom通知
```

---

## 事件订阅 vs 取消订阅

```typescript
// 订阅 - 返回取消函数
const unsubscribe = pi.events.on("event_name", handler);

// 取消订阅
unsubscribe();

// 批量取消
const eventUnsubscribes = [
    pi.events.on(EVENT_A, handlerA),
    pi.events.on(EVENT_B, handlerB),
];
eventUnsubscribes.forEach(unsub => unsub());
```

---

## 跨进程事件通信

### 子进程 → 父进程事件流

```
subagent-runner.ts (子进程)
    │
    ├─ 需要发送控制事件
    │       │
    │       ▼
    │   events.jsonl
    │   {"type": "subagent.control", ...}
    │       │
    │       ▼
    │   pi.events.emit("subagent.control", ...)
    │
    └─ 实际写入文件系统
            │
            ▼
    async-job-tracker.ts (父进程)
            │
            ├─ 轮询读取 events.jsonl
            ├─ 解析JSON
            ├─ pi.events.emit(SUBAGENT_CONTROL_EVENT, ...)
            │
            ▼
    index.ts controlEventHandler()
            │
            ├─ 显示控制通知
            └─ 发送intercom消息
```

---

## 核心设计

| 设计点 | 说明 |
|--------|------|
| **全局总线** | `pi.events` 全局共享，所有扩展可访问 |
| **发布订阅** | `emit` 发布，`on` 订阅，完全解耦 |
| **自动清理** | 会话关闭时取消订阅，防止内存泄漏 |
| **跨进程** | 通过 `events.jsonl` 文件实现子进程→父进程转发 |
| **类型安全** | TypeScript 类型定义，编译时检查 |

---

## 一句话总结

**`pi.events` 是 Pi 扩展系统的"神经系统"：全局事件总线实现跨组件解耦通信，生命周期事件管理会话状态，自定义事件支持跨进程转发，自动清理防止内存泄漏！**
