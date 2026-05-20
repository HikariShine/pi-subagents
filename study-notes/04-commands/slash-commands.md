# slash-commands.ts - Slash 命令注册详解

## 文件信息

- **文件名**: `slash-commands.ts`
- **大小**: 21K
- **位置**: 项目根目录
- **所属模块**: Commands / Slash 命令
- **九大组件**: #4 Slash 命令（批量注册）

---

## 文件作用

注册 `/run`、`/chain`、`/parallel`、`/run-chain`、`/agents`、`/subagents-status`、`/subagents-doctor` 等 Slash 命令。

让用户通过命令行快速触发子代理执行。

---

## 命令分类

### 执行类命令（5个）

| 命令 | 说明 | 执行路径 |
|------|------|---------|
| `/run` | 直接运行子代理 | 解析参数 → `runSlashSubagent()` |
| `/chain` | 链式执行 | 解析参数 → `runSlashSubagent()` |
| `/run-chain` | 运行保存的 Chain | 解析参数 → `runSlashSubagent()` |
| `/parallel` | 并行执行 | 解析参数 → `runSlashSubagent()` |
| `/subagents-doctor` | 诊断 | `runSlashSubagent({ action: "doctor" })` |

### UI 类命令（2个）

| 命令 | 说明 | 执行方式 |
|------|------|---------|
| `/agents` | 打开 Agent 管理器 | `openAgentManager()` UI 交互 |
| `/subagents-status` | 打开状态管理器 | `SubagentsStatusComponent` UI |

---

## 核心执行流程

### 执行类命令

```
用户输入 /run researcher[output=result.md] "分析代码" --bg
    │
    ▼
pi.registerCommand("run") handler
    │
    ├─ extractExecutionFlags()     提取 --bg/--fork
    ├─ parseAgentToken()           解析 researcher[output=result.md]
    ├─ discoverAgents()            查找 Agent
    │
    ▼
构建 SubagentParamsLike
    │
    ▼
runSlashSubagent(pi, ctx, params)
    │
    ├─ emit SLASH_SUBAGENT_REQUEST_EVENT
    │
    ▼
slash-bridge 接收请求 → 调用 execute()
    │
    ▼
监听 SLASH_SUBAGENT_RESPONSE_EVENT
    │
    ▼
pi.sendMessage() 显示结果
```

---

## 关键函数

### registerSlashCommands

```typescript
export function registerSlashCommands(pi: ExtensionAPI, state: SubagentState): void {
    // UI 类
    pi.registerCommand("agents", { ... });           // Agent 管理器
    pi.registerCommand("subagents-status", { ... }); // 状态管理器
    
    // 执行类
    pi.registerCommand("run", { ... });              // 单任务
    pi.registerCommand("chain", { ... });           // 链式
    pi.registerCommand("run-chain", { ... });       // 保存的 Chain
    pi.registerCommand("parallel", { ... });         // 并行
    pi.registerCommand("subagents-doctor", { ... }); // 诊断
}
```

### runSlashSubagent

```typescript
async function runSlashSubagent(pi, ctx, params): Promise<void> {
    const requestId = randomUUID();
    
    // 发送初始消息
    pi.sendMessage({ customType: SLASH_RESULT_TYPE, ... });
    
    try {
        // 发出请求并等待响应
        const response = await requestSlashRun(pi, ctx, requestId, params);
        
        // 显示最终结果
        pi.sendMessage({ customType: SLASH_RESULT_TYPE, ... });
    } catch (error) {
        // 错误处理
        pi.sendMessage({ ... });
    }
}
```

### requestSlashRun

```typescript
async function requestSlashRun(pi, ctx, requestId, params): Promise<SlashSubagentResponse> {
    return new Promise((resolve, reject) => {
        // 15秒启动超时
        const startTimeout = setTimeout(() => {
            reject(new Error("Slash subagent bridge did not start within 15s"));
        }, 15000);
        
        // 监听 bridge 事件
        const onStarted = (data) => {
            if (data.requestId !== requestId) return;
            started = true;
            clearTimeout(startTimeout);
        };
        
        const onResponse = (data) => {
            if (data.requestId !== requestId) return;
            clearTimeout(startTimeout);
            resolve(data);
        };
        
        const onUpdate = (data) => {
            if (data.requestId !== requestId) return;
            applySlashUpdate(requestId, data);
        };
        
        // 订阅事件
        pi.events.on(SLASH_SUBAGENT_STARTED_EVENT, onStarted);
        pi.events.on(SLASH_SUBAGENT_RESPONSE_EVENT, onResponse);
        pi.events.on(SLASH_SUBAGENT_UPDATE_EVENT, onUpdate);
        
        // 发出执行请求
        pi.events.emit(SLASH_SUBAGENT_REQUEST_EVENT, { requestId, params });
    });
}
```

---

## 参数解析

### 内联配置解析

```typescript
// /run researcher[output=result.md,model=gpt-4,progress]
const { name: agentName, config: inline } = parseAgentToken("researcher[output=result.md,model=gpt-4]");

// 结果
{
    name: "researcher",
    config: {
        output: "result.md",
        model: "gpt-4",
        progress: true
    }
}
```

### 执行标志提取

```typescript
// task --bg --fork
const { args: cleanedArgs, bg, fork } = extractExecutionFlags("task --bg --fork");

// 结果
{
    args: "task",
    bg: true,      // 后台执行
    fork: true     // fork 模式
}
```

---

## 与 slash-bridge 的关系

```
slash-commands.ts                    slash-bridge.ts
    │                                    │
    ├─ emit SLASH_SUBAGENT_REQUEST_EVENT ─┤
    │                                    │
    │                                    ▼
    │                              监听 REQUEST
    │                                    │
    │                                    ├─ 调用 options.execute()
    │                                    │
    │                                    ├─ emit STARTED
    │                                    ├─ emit UPDATE (进度)
    │                                    └─ emit RESPONSE (完成)
    │                                    │
    ├─ 监听 STARTED ◄────────────────────┤
    ├─ 监听 UPDATE ◄─────────────────────┤
    ├─ 监听 RESPONSE ◄───────────────────┤
    │                                    │
    ▼                                    ▼
显示结果                          内部执行
```

---

## 快捷键

```typescript
pi.registerShortcut("ctrl+shift+a", {
    handler: async (ctx) => {
        await openAgentManager(pi, ctx);
    },
});
```

**Ctrl+Shift+A**: 快速打开 Agent 管理器

---

## 一句话总结

**`slash-commands.ts` 是"命令入口层"：注册 `/run`、`/chain`、`/parallel` 等 Slash 命令，解析参数和内联配置（`--bg`、`--fork`），通过 `SLASH_SUBAGENT_REQUEST_EVENT` 触发 `slash-bridge`，监听 `STARTED/UPDATE/RESPONSE` 事件，实时显示进度和结果，实现用户命令行快速触发子代理执行！**
