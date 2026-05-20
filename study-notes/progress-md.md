# progress.md 机制详解

## 概述

`progress.md` 是 pi-subagents 链式执行中的**状态跟踪文件**，用于在多个 Agent 之间共享执行进度。

---

## 初始内容模板

```typescript
// settings.ts:11
const INITIAL_PROGRESS_CONTENT = `# Progress

## Status
In Progress

## Tasks

## Files Changed

## Notes
`;
```

---

## 启用方式

通过 `progress: true` 参数启用：

```typescript
// schemas.ts
progress: Type.Boolean({ description: "Enable progress.md tracking in {chain_dir}" })
```

---

## 创建逻辑

### 第一步创建

当检测到链中有步骤启用 `progress: true` 时，在**第一步**自动创建：

```typescript
// chain-execution.ts:134-143
function ensureProgressFile(
    chainDir: string,
    parallelBehaviors: ResolvedStepBehavior[],
    progressCreated: boolean,
): boolean {
    // 如果已创建，或没有启用 progress，直接返回
    if (progressCreated || !parallelBehaviors.some((b) => b.progress)) {
        return progressCreated;
    }
    
    // 【第一步】创建 progress.md
    writeInitialProgressFile(chainDir);
    return true;
}
```

### 调用位置

| 文件 | 场景 | 条件 |
|------|------|------|
| `chain-execution.ts:141` | 同步链式并行步骤 | `progress: true` |
| `async-execution.ts:295` | 异步链式并行步骤 | `progressPrecreated` |
| `subagent-executor.ts:1170` | 子代理执行器并行 | `parallelProgressPrecreated` |
| `subagent-runner.ts:824` | 并行任务组 | `ensureParallelProgressFile` |

---

## 指令注入

### settings.ts:254-260

```typescript
if (behavior.progress) {
    const progressPath = path.join(chainDir, "progress.md");
    
    if (isFirstProgressAgent) {
        // 第一步：创建并维护
        suffixParts.push(`Create and maintain progress at: ${progressPath}`);
    } else {
        // 后续步骤：读取更新
        suffixParts.push(`Update progress at: ${progressPath}`);
    }
}
```

### 指令变化

| 步骤 | 指令 | 作用 |
|------|------|------|
| 第一步 | `Create and maintain progress at: ...` | 创建文件并维护 |
| 后续 | `Update progress at: ...` | 读取并更新 |

---

## UI 显示

### chain-clarify.ts:1331-1338

```typescript
if (progressEnabled) {
    const isFirstStep = i === 0;
    const progressAction = isFirstStep 
        ? "writes progress.md"   // 第一步创建
        : "reads progress.md"; // 后续读取
    lines.push(`progress: ${progressAction}`);
}
```

---

## 完整流程

```
链式执行开始
    │
    ▼
检查 parallelBehaviors
    │
    ├─ 有 progress: true?
    │      │
    │      ▼
    │   writeInitialProgressFile(chainDir)
    │      │
    │      ▼
    │   写入 INITIAL_PROGRESS_CONTENT
    │      │
    │      ▼
    │   任务指令："Create and maintain progress at: ..."
    │
    ▼
Agent 1 执行（创建 progress.md）
    │
    ▼
Agent 2 执行
    │
    └─ 任务指令："Update progress at: ..."（读取更新）
    │
    ▼
Agent 3 执行
    │
    └─ 任务指令："Update progress at: ..."
```

---

## 一句话总结

**`progress.md` 是链式执行的状态跟踪文件：通过 `progress: true` 启用，`writeInitialProgressFile` 在第一步创建，注入 `Create and maintain` 指令给第一步 Agent，`Update` 指令给后续 Agent，实现多 Agent 协作的状态共享！**
