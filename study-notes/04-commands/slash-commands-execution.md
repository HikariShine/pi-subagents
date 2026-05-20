# slash-commands.ts 执行核心详解

## 概述

`slash-commands.ts` 中的执行核心是 Slash 命令与 `slash-bridge` 协作的完整链路。

---

## 执行流程图

```
用户输入 /run researcher "分析代码"
    │
    ▼
registerCommand("run") handler
    │
    ├─ extractExecutionFlags()     提取 --bg/--fork
    ├─ parseAgentToken()           解析 agent[output=...]
    ├─ discoverAgents()            查找 Agent
    │
    ▼
runSlashSubagent(pi, ctx, params)
    │
    ├─ 1. buildSlashInitialResult()      构建初始结果
    ├─ 2. extractSlashMessageText()    提取初始文本
    ├─ 3. pi.sendMessage()               发送"Running..."消息
    ├─ 4. persistSlashSessionSnapshot()  持久化会话快照
    │
    ▼
    ├─ 5. requestSlashRun() 【核心】
    │       │
    │       ├─ 设置 15秒启动超时
    │       │
    │       ├─ 订阅事件
    │       │   ├─ SLASH_SUBAGENT_STARTED_EVENT → onStarted()
    │       │   ├─ SLASH_SUBAGENT_UPDATE_EVENT  → onUpdate()
    │       │   └─ SLASH_SUBAGENT_RESPONSE_EVENT → onResponse()
    │       │
    │       ├─ emit SLASH_SUBAGENT_REQUEST_EVENT
    │       │       │
    │       │       ▼
    │       │   slash-bridge 监听
    │       │       │
    │       │       ├─ 调用 options.execute()
    │       │       ├─ emit STARTED
    │       │       ├─ emit UPDATE (多次)
    │       │       └─ emit RESPONSE
    │       │
    │       └─ await Promise → 返回 response
    │
    ▼
    ├─ 6. finalizeSlashResult(response)  构建最终结果
    ├─ 7. pi.sendMessage()               发送完成消息
    ├─ 8. persistSlashSessionSnapshot()  再次持久化
    └─ 9. UI 状态清理 / 错误通知
```

---

## 核心方法详解

### runSlashSubagent - 执行主函数

```typescript
async function runSlashSubagent(
    pi: ExtensionAPI,
    ctx: ExtensionContext,
    params: SubagentParamsLike
): Promise<void> {
    const requestId = randomUUID();
    
    // 1. 构建初始结果
    const initialDetails = buildSlashInitialResult(requestId, params);
    
    // 2. 提取初始显示文本
    const initialText = extractSlashMessageText(initialDetails.result.content) 
        || "Running subagent...";
    
    // 3. 发送初始消息（执行前）
    pi.sendMessage({
        customType: SLASH_RESULT_TYPE,
        content: initialText,  // "Running researcher..."
        display: true,
        details: initialDetails,
    });
    
    // 4. 持久化会话快照
    persistSlashSessionSnapshot(ctx);
    
    try {
        // 5. 【核心】发出请求并等待执行
        const response = await requestSlashRun(pi, ctx, requestId, params);
        
        // 6. 构建最终结果
        const finalDetails = finalizeSlashResult(response);
        
        // 7. 发送完成消息
        pi.sendMessage({
            customType: SLASH_RESULT_TYPE,
            content: buildSlashExportText(response),  // "Analysis complete..."
            display: true,
            details: finalDetails,
        });
        
        // 8. 再次持久化
        persistSlashSessionSnapshot(ctx);
        
        // 9. UI 清理
        if (ctx.hasUI) {
            ctx.ui.setStatus("subagent-slash", undefined);
        }
        
        // 10. 错误通知
        if (response.isError && ctx.hasUI) {
            ctx.ui.notify(response.errorText || "Subagent failed", "error");
        }
        
    } catch (error) {
        // 错误处理...
    }
}
```

---

### requestSlashRun - 核心执行引擎

```typescript
async function requestSlashRun(
    pi: ExtensionAPI,
    ctx: ExtensionContext,
    requestId: string,
    params: SubagentParamsLike
): Promise<SlashSubagentResponse> {
    return new Promise((resolve, reject) => {
        let done = false;
        let started = false;

        // 1. 15秒启动超时
        const startTimeout = setTimeout(() => {
            reject(new Error("Slash subagent bridge did not start within 15s"));
        }, 15000);

        // 2. 监听开始事件
        const onStarted = (data: unknown) => {
            if (done || !data || typeof data !== "object") return;
            if ((data as { requestId?: unknown }).requestId !== requestId) return;
            
            started = true;
            clearTimeout(startTimeout);
            if (ctx.hasUI) ctx.ui.setStatus("subagent-slash", "running...");
        };

        // 3. 监听响应事件（完成）
        const onResponse = (data: unknown) => {
            if (done || !data || typeof data !== "object") return;
            const response = data as Partial<SlashSubagentResponse>;
            if (response.requestId !== requestId) return;
            
            clearTimeout(startTimeout);
            finish(() => resolve(response as SlashSubagentResponse));
        };

        // 4. 监听更新事件（进度）
        const onUpdate = (data: unknown) => {
            if (done || !data || typeof data !== "object") return;
            const update = data as SlashSubagentUpdate;
            if (update.requestId !== requestId) return;
            
            // 更新实时快照
            applySlashUpdate(requestId, update);
            
            // 更新 UI 状态
            if (!ctx.hasUI) return;
            const tool = update.currentTool ? ` ${update.currentTool}` : "";
            const count = update.toolCount ?? 0;
            ctx.ui.setStatus("subagent-slash", `${count}${tool}`);
        };

        // 5. 订阅事件
        const unsubStarted = pi.events.on(SLASH_SUBAGENT_STARTED_EVENT, onStarted);
        const unsubResponse = pi.events.on(SLASH_SUBAGENT_RESPONSE_EVENT, onResponse);
        const unsubUpdate = pi.events.on(SLASH_SUBAGENT_UPDATE_EVENT, onUpdate);

        // 6. 发出执行请求 → 触发 slash-bridge
        pi.events.emit(SLASH_SUBAGENT_REQUEST_EVENT, { requestId, params });

        // 清理函数
        function finish(fn: () => void): void {
            if (done) return;
            done = true;
            unsubStarted();
            unsubResponse();
            unsubUpdate();
            fn();
        }
    });
}
```

---

### 三个事件职责

| 事件 | 来源 | 处理 | 作用 |
|------|------|------|------|
| `SLASH_SUBAGENT_STARTED_EVENT` | bridge → onStarted() | `ctx.ui.setStatus(