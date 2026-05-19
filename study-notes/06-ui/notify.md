# notify.ts - 子代理完成通知

## 文件信息

- **文件名**: `notify.ts`
- **大小**: 2.9K
- **位置**: 项目根目录
- **所属模块**: UI / 通知系统
- **九大组件**: #5 通知系统

---

## 文件作用

异步作业完成后，发送**桌面通知/消息提醒**给用户。

与 `async-job-tracker` 形成**并行监听**模式，各司其职。

---

## 多监听器模式

```
SUBAGENT_ASYNC_COMPLETE_EVENT 发出
    │
    ├─► async-job-tracker.handleComplete()
    │      ├─ 更新 state.asyncJobs[id].status
    │      ├─ rerenderWidget() 更新侧边栏
    │      └─ scheduleCleanup() 安排清理
    │
    ├─► notify.ts handleComplete()  ◄── 这里！
    │      ├─ 去重检查（防止重复通知）
    │      ├─ 构建通知内容
    │      └─ pi.sendMessage() 发送桌面通知
    │
    └─► 可能还有其他监听器...
```

---

## 核心代码

```typescript
export default function registerSubagentNotify(pi: ExtensionAPI): void {
    // 获取去重Map（10分钟TTL）
    const seen = getGlobalSeenMap("__pi_subagents_notify_seen__");
    const ttlMs = 10 * 60 * 1000;

    const handleComplete = (data: unknown) => {
        const result = data as SubagentResult;
        const now = Date.now();
        
        // 1. 去重检查
        const key = buildCompletionKey(result, "notify");
        if (markSeenWithTtl(seen, key, now, ttlMs)) return;

        // 2. 判断状态
        const paused = !result.success && (
            result.exitCode === 0
            || result.state === "paused"
            || summary.startsWith("Paused after interrupt.")
        );
        const status = paused ? "paused" 
                      : result.success ? "completed" 
                      : "failed";

        // 3. 构建任务信息
        const taskInfo = (result.taskIndex !== undefined && result.totalTasks !== undefined)
            ? ` (${result.taskIndex + 1}/${result.totalTasks})`
            : "";

        // 4. 构建分享链接行
        const sessionLine = result.shareUrl
            ? `Session: ${result.shareUrl}`
            : result.shareError
                ? `Session share error: ${result.shareError}`
                : result.sessionFile
                    ? `Session file: ${result.sessionFile}`
                    : undefined;

        // 5. 构建通知内容
        const content = [
            `Background task ${status}: **${agent}**${taskInfo}`,
            "",
            displaySummary,
            sessionLine ? "" : undefined,
            sessionLine,
        ]
            .filter((line) => line !== undefined)
            .join("\n");

        // 6. 发送通知消息
        pi.sendMessage(
            {
                customType: "subagent-notify",
                content,
                display: true,
            },
            { triggerTurn: true },
        );
    };

    // 7. 订阅完成事件
    pi.events.on(SUBAGENT_ASYNC_COMPLETE_EVENT, handleComplete);
}
```

---

## 通知内容示例

### 单任务完成

```
Background task completed: **researcher**

Analysis complete. Found 3 relevant sources.

Session: https://share.example.com/abc123
```

### 并行任务完成

```
Background task completed: **analyzer** (2/3)

Module B analysis finished.

Session: https://share.example.com/abc123
```

### 任务失败

```
Background task failed: **worker**

Error: API timeout after 30s.

Session file: /path/to/session.jsonl
```

### 任务中断

```
Background task paused: **researcher**

Paused after interrupt at step 2.

Session: https://share.example.com/abc123
```

---

## 关键特性

| 特性 | 说明 |
|------|------|
| **去重** | 10分钟TTL，防止重复通知 |
| **状态判断** | completed / failed / paused（中断） |
| **任务计数** | 并行任务显示 `(1/3)` |
| **分享链接** | 自动附加 session share URL |
| **热重载安全** | 清理旧订阅，防止重复注册 |

---

## 与 result-watcher 的关系

```
result-watcher 检测到 result.json
    │
    └─ emit SUBAGENT_ASYNC_COMPLETE_EVENT
           │
           ├─► async-job-tracker.handleComplete()  [更新UI]
           │
           └─► notify.ts handleComplete()           [发送通知]
                  │
                  └─ pi.sendMessage() 发送完成通知
```

---

## 一句话总结

**`notify.ts` 是"完成通知器"：监听 `SUBAGENT_ASYNC_COMPLETE_EVENT`，10分钟去重，发送桌面通知显示任务状态、结果摘要和分享链接，与 `async-job-tracker` 并行工作，各司其职！**
