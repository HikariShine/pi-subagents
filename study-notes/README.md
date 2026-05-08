# pi-subagents 学习笔记

## 目录结构

```
study-notes/
├── README.md                 # 本文件
├── 00-template.md            # 详解文件模板
├── 01-core/                  # 核心基础设施
│   ├── package.json.md
│   ├── README.md
│   ├── types.md
│   ├── schemas.md
│   └── index.md
├── 02-execution/             # 执行层
│   ├── subagent-executor.md
│   ├── execution.md
│   └── subagent-runner.md
├── 03-agents/                # Agent 管理
│   ├── agents.md
│   ├── agent-management.md
│   └── agent-manager.md
├── 04-async/                 # 异步执行
│   ├── async-execution.md
│   └── async-job-tracker.md
├── 05-chain/                 # 链式执行
│   ├── chain-execution.md
│   └── chain-clarify.md
├── 06-ui/                    # UI 和交互
│   ├── render.md
│   ├── slash-commands.md
│   └── slash-bridge.md
├── 07-utils/                 # 工具函数
│   ├── utils.md
│   └── formatters.md
└── 08-docs/                  # 文档和配置
    └── agents-oracle.md
```

## 如何创建详解文件

1. 阅读原文件（如 `types.ts`）
2. 在对应目录下创建 `.md` 文件（如 `study-notes/01-core/types.md`）
3. 参考 `00-template.md` 格式填写内容
4. 更新根目录 `STUDY.md` 中的阅读状态和详解路径

## 命名规范

- 目录: `XX-模块名/`（两位数字前缀）
- 文件: `原始文件名（不含扩展名）.md`

## 内容规范

每个详解文件应包含：

1. **文件信息**: 大小、位置、模块
2. **文件作用**: 一句话描述
3. **核心内容**: 主要导出、类型、算法
4. **依赖关系**: 导入导出关系
5. **关键代码段**: 重要代码及注释
6. **阅读笔记**: 个人理解和疑问
