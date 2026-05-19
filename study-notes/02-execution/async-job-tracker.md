# async-job-tracker.ts - 异步作业追踪器详解

## 文件信息

- **文件名**: `async-job-tracker.ts`
- **大小**: 7.4K
- **位置**: 项目根目录
- **所属模块**: Execution / 异步监控层

---

## 文件作用

`async-job-tracker.ts` 是 **pi-subagents** 的异步作业监控中心，负责：

1. **追踪异步作业状态** - 维护所有后台运行的子代理作业
2. **轮询状态更新** - 每秒读取 `status.json` 获取最新状态
3. **事件转发** - 将子进程的控制事件转发给父会话
4. **UI 更新** - 驱动侧边栏 Widget 显示作业进度

---

## 核心架构

```
┌─────────────────────────────────────────┐
│     createAsyncJobTracker()             │
│     返回: {                               │
│       ensurePoller,    ← 启动轮询器     │
│       handleStarted,   ← 处理启动事件   │
│       handleComplete,  ← 处理完成事件   │
│       resetJobs        ← 重置所有作业   │
│     }                                    │
└─────────────────────────────────────────┘
                    │
    ┌───────────────┼───────────────┐
    │               │               │
handleStarted    ensurePoller    handleComplete
    │               │               │
    ▼               ▼               ▼
 注册作业         轮询检查        标记完成
 status.json     events.jsonl    触发清理
```

---

## 返回值详解

### 1. handleStarted - 处理启动事件

**触发**: `SUBAGENT_ASYNC_STARTED_EVENT`

**调用位置**: `executeAsyncChain` / `executeAsyncSingle`

```typescript
// 子进程启动后发出事件
ctx.pi.events.emit(SUBAGENT_ASYNC_STARTED_EVENT, {
    id,              // 作业ID
    asyncDir,        // 异步目录
    agent,           // Agent名称
    chain            // Chain步骤（可选）
});
```

**处理逻辑**:
```typescript
const handleStarted = (data: unknown) => {
    const info = data as { id, asyncDir, agent, chain };
    if (!info.id) return;
    
    const now = Date.now();
    const asyncDir = info.asyncDir ?? path.join(asyncDirRoot, info.id);
    const agents = info.chain && info.chain.length > 0 
        ? info.chain 
        : info.agent ? [info.agent] : undefined;
    
    // 创建作业记录
    state.asyncJobs.set(info.id, {
        asyncId: info.id,
        asyncDir,
        status: "queued",              // 初始状态：排队中
        mode: info.chain ? "chain" : "single",
        agents,
        stepsTotal: agents?.length,
        startedAt: now,
        updatedAt: now,
    });
    
    ensurePoller();  // 确保轮询器运行
    rerenderWidget(state.lastUiContext);  // 更新UI
};
```

---

### 2. ensurePoller - 轮询器核心

**作用**: 启动定时器，每秒检查所有作业状态

```typescript
const ensurePoller = () => {
    if (state.poller) return;  // 已有轮询器，不重复创建
    
    state.poller = setInterval(() => {
        // 没有作业时停止轮询
        if (state.asyncJobs.size === 0) {
            clearInterval(state.poller);
            state.poller = null;
            return;
        }
        
        // 遍历所有作业
        for (const job of state.asyncJobs.values()) {
            // 1. 读取控制事件（events.jsonl）
            emitNewControlEvents(job);
            
            // 2. 读取状态（status.json）
            const status = readStatus(job.asyncDir);
            if (status) {
                // 更新作业状态
                job.status = status.state;           // running/complete/failed/paused
                job.activityState = status.activityState;
                job.currentStep = status.currentStep;
                job.stepsTotal = status.steps?.length;
                job.totalTokens = status.totalTokens;
                job.lastActivityAt = status.lastActivityAt ?? job.lastActivityAt;
                job.currentTool = status.currentTool ?? job.currentTool;
                job.updatedAt = status.lastUpdate ?? Date.now();
                
                // 完成后安排清理
                if (isCompleteOrFailed && statusChanged) {
                    scheduleCleanup(job.asyncId);
                }
            }
        }
        
        // 更新UI显示
        rerenderWidget(state.lastUiContext);
        
    }, pollIntervalMs);  // 默认 1000ms（1秒）
    
    state.poller.unref?.();  // 允许Node.js退出
};
```

**轮询数据来源**:

| 文件 | 作用 | 更新方 |
|------|------|--------|
| `status.json` | 作业状态、进度、当前步骤 | `subagent-runner.ts` |
| `events.jsonl` | 控制事件（needs_attention等） | `subagent-runner.ts` |
| `result.json` | 最终结果 | `subagent-runner.ts` |

---

### 3. emitNewControlEvents - 跨进程事件桥接

**核心机制**: 通过文件系统实现跨进程通信

```typescript
const emitNewControlEvents = (job: AsyncJobState) => {
    const eventsPath = path.join(job.asyncDir, "events.jsonl");
    
    // 打开文件（只读模式）
    const fd = fs.openSync(eventsPath, "r");
    
    try {
        const stat = fs.fstatSync(fd);
        const cursor = job.controlEventCursor ?? 0;  // 上次读取位置
        
        // 只读取新内容
        if (stat.size <= cursor) return;
        const buffer = Buffer.alloc(stat.size - cursor);
        fs.readSync(fd, buffer, 0, buffer.length, cursor);
        
        // 更新读取位置
        const lastNewline = buffer.lastIndexOf(0x0a);  // 找到最后一个换行
        job.controlEventCursor = cursor + lastNewline + 1;
        
        // 解析每一行JSON
        for (const line of buffer.subarray(0, lastNewline).toString().split("\n")) {
            if (!line.trim()) continue;
            
            const parsed = JSON.parse(line);
            if (parsed.type !== "subagent.control") continue;
            
            const record = parsed as {
                event?: ControlEvent;
                channels?: string[];
                childIntercomTarget?: string;
                intercom?: { to?: string; message?: string };
            };
            
            const payload = {
                event: record.event,
                source: "async",
                asyncDir: job.asyncDir,
                childIntercomTarget: record.childIntercomTarget,
                noticeText: formatControlNoticeMessage(record.event, record.childIntercomTarget),
            };
            
            // 转发到Pi核心
            if (record.channels.includes("event")) {
                pi.events.emit(SUBAGENT_CONTROL_EVENT, payload);
            }
            if (record.channels.includes("intercom") && record.intercom?.to) {
                pi.events.emit(SUBAGENT_CONTROL_INTERCOM_EVENT, {
                    ...payload,
                    to: record.intercom.to,
                    message: record.intercom.message,
                });
            }
        }
    } finally {
        fs.closeSync(fd);
    }
};
```

**事件类型**:

| Channel | Pi事件 | 用途 |
|---------|--------|------|
| `event` | `SUBAGENT_CONTROL_EVENT` | 状态更新、控制通知 |
| `intercom` | `SUBAGENT_CONTROL_INTERCOM_EVENT` | 需要人工决策、阻塞等待 |

---

### 4. handleComplete - 处理完成事件

**触发**: `SUBAGENT_ASYNC_COMPLETE_EVENT`

**调用位置**: `result-watcher.ts` 检测到 `result.json`

```typescript
const handleComplete = (data: unknown) => {
    const result = data as { id, success, asyncDir };
    const asyncId = result.id;
    if (!asyncId) return;
    
    const job = state.asyncJobs.get(asyncId);
    if (job) {
        job.status = result.success ? "complete" : "failed";
        job.updatedAt = Date.now();
    }
    
    rerenderWidget(state.lastUiContext);  // 更新UI
    scheduleCleanup(asyncId);            // 安排清理
};
```

---

### 5. scheduleCleanup - 清理机制

**作用**: 完成后保留一段时间，然后清理内存

```typescript
const scheduleCleanup = (asyncId: string) => {
    // 默认10秒后清理（completionRetentionMs = 10000）
    const timer = setTimeout(() => {
        state.cleanupTimers.delete(asyncId);
        state.asyncJobs.delete(asyncId);      // 从内存移除
        rerenderWidget(state.lastUiContext);   // 更新UI
    }, completionRetentionMs);
    
    state.cleanupTimers.set(asyncId, timer);
};
```

---

## 使用位置（index.ts）

```typescript
// 创建tracker
const tracker = createAsyncJobTracker(pi, state, asyncDirRoot, {
    completionRetentionMs: 10000,  // 完成后保留10秒
    pollIntervalMs: POLL_INTERVAL_MS,  // 默认1000ms
});

// 1. 会话开始时重置
pi.events.on("session_start", () => {
    tracker.resetJobs();
});

// 2. 有异步作业时确保轮询
pi.events.on("tool_result", (result) => {
    if (result.toolName === "subagent"