# result-watcher.ts - 结果文件监视器详解

## 文件信息

- **文件名**: `result-watcher.ts`
- **大小**: 3.2K
- **位置**: 项目根目录
- **所属模块**: Execution / 异步监控层

---

## 文件作用

`result-watcher.ts` 是 **pi-subagents** 的异步作业完成检测器：

1. **监视结果目录** - 通过 `fs.watch` 监听 `RESULTS_DIR`
2. **检测新结果文件** - 子进程完成时写入的 `result.json`
3. **发出完成事件** - 触发 `SUBAGENT_ASYNC_COMPLETE_EVENT`
4. **清理结果文件** - 处理后立即删除

---

## 返回值详解

### 1. startResultWatcher - 启动监视器

```typescript
const startResultWatcher = () => {
    state.watcher = fs.watch(resultsDir, (ev, file) => {
        if (ev !== "rename" || !file) return;  // 只处理 rename 事件
        if (!file.endsWith(".json")) return;  // 只处理 JSON
        state.resultFileCoalescer.schedule(file);  // 防抖处理
    });
    state.watcher.on("error", scheduleRestart);  // 错误时自动重启
};
```

**为什么是 `rename`？** Node.js `fs.watch` 跨平台封装，文件创建统一触发 `rename`。

---

### 2. primeExistingResults - 预热处理

```typescript
const primeExistingResults = () => {
    fs.readdirSync(resultsDir)
        .filter(f => f.endsWith(".json"))
        .forEach(file => state.resultFileCoalescer.schedule(file, 0));  // delay=0
};
```

**场景**: 插件热重载后恢复未处理的结果。

---

### 3. stopResultWatcher - 停止监视

```typescript
const stopResultWatcher = () => {
    state.watcher?.close();
    state.watcher = null;
    state.resultFileCoalescer.clear();
};
```

---

## 核心处理: handleResult

```typescript
const handleResult = (file: string) => {
    const data = JSON.parse(fs.readFileSync(resultPath, "utf-8"));
    
    // 1. 会话过滤
    if (data.sessionId && data.sessionId !== state.currentSessionId) return;
    if (!data.sessionId && data.cwd && data.cwd !== state.baseCwd) return;
    
    // 2. 去重检查（10秒TTL）
    const completionKey = buildCompletionKey(data, `result:${file}`);
    if (markSeenWithTtl(state.completionSeen, completionKey, now, ttlMs)) {
        fs.unlinkSync(resultPath);  // 已处理过，直接删除
        return;
    }
    
    // 3. 发出完成事件！
    pi.events.emit(SUBAGENT_ASYNC_COMPLETE_EVENT, data);
    
    // 4. 删除结果文件
    fs.unlinkSync(resultPath);
};
```

---

## 双重防抖机制

| 层级 | 机制 | 延迟 | 作用 |
|------|------|------|------|
| 第一层 | `FileCoalescer` | 50ms | 合并重复文件事件 |
| 第二层 | `markSeenWithTtl` | 10秒 | 防止重复处理同一结果 |

---

## 与 async-job-tracker 的关系

```
result-watcher (事件驱动)
    │
    ├─ fs.watch 检测 result.json
    ├─ 发出 SUBAGENT_ASYNC_COMPLETE_EVENT
    │
    ▼
async-job-tracker.handleComplete()
    ├─ 更新 job.status
    ├─ rerenderWidget()
    └─ scheduleCleanup()
```

| 组件 | 触发 | 数据源 | 职责 |
|------|------|--------|------|
| `result-watcher` | 文件事件（即时） | `result.json` | 最终完成通知 |
| `async-job-tracker` | 定时轮询（1秒） | `status.json` | 实时状态更新 |

---

## 使用位置 (index.ts)

```typescript
const watcher = createResultWatcher(pi, state, RESULTS_DIR, ttlMs);

// 插件激活时
watcher.startResultWatcher();
watcher.primeExistingResults();

// 插件停用时
watcher.stopResultWatcher();
```

---

## 一句话总结

**`result-watcher` 是"文件系统哨兵"：用 `fs.watch` 监听 `rename` 事件检测新结果文件，双重防抖（50ms合并+10秒去重），过滤当前会话，发出完成事件，处理后立即清理，与 `async-job-tracker` 形成"事件驱动+轮询更新"的互补监控体系！**
