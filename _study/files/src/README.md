# src/ 目录结构说明

## 顶层文件（核心）

| 文件 | 说明 | 重要度 |
|---|---|---|
| `entrypoints/cli.tsx` | CLI 入口，快速路径处理 | ⭐⭐⭐⭐⭐ |
| `main.tsx` | 全局初始化（4600+ 行），所有子系统在此启动 | ⭐⭐⭐⭐⭐ |
| `QueryEngine.ts` | 多轮 API 编排核心 | ⭐⭐⭐⭐⭐ |
| `Task.ts` | 任务抽象基础定义 | ⭐⭐⭐⭐ |
| `Tool.ts` | 工具抽象基础定义 | ⭐⭐⭐⭐ |
| `commands.ts` | Slash 命令路由 | ⭐⭐⭐⭐ |
| `replLauncher.tsx` | REPL 启动逻辑 | ⭐⭐⭐ |
| `query.ts` | 查询工具函数 | ⭐⭐⭐ |
| `context.ts` | 全局 Context 入口 | ⭐⭐⭐ |
| `cost-tracker.ts` | Token 费用追踪 | ⭐⭐ |
| `history.ts` | 会话历史管理 | ⭐⭐ |
| `tasks.ts` | 任务聚合导出 | ⭐⭐ |
| `tools.ts` | 工具聚合导出 | ⭐⭐ |

## 子目录概览

| 目录 | 文件数 | 功能 |
|---|---|---|
| `tools/` | 30+ 子目录 | 49 个工具实现 |
| `components/` | 146 文件 | React UI 组件 |
| `hooks/` | 87 文件 | React hooks |
| `ink/` | 96 文件 | 自定义终端渲染器（Ink fork）|
| `services/` | 39 子目录 | 各类后台服务 |
| `commands/` | 105+ 子目录 | Slash 命令实现 |
| `utils/` | 330+ 文件 | 工具函数 |
| `state/` | - | 全局 AppState 定义 |
| `context/` | - | React Context 定义 |
| `types/` | - | 共享 TypeScript 类型 |
| `schemas/` | - | Zod schema 定义 |

## 内部专有功能目录（了解即可）

| 目录 | 推测功能 |
|---|---|
| `bridge/` | Bridge 模式（外部版禁用）|
| `buddy/` | Buddy 功能（可能是某个协作特性）|
| `coordinator/` | Coordinator 模式（外部版禁用）|
| `voice/` | 语音输入（外部版禁用）|
| `remote/` | 远程会话支持 |
| `server/` | 内置服务器（可能是 daemon 模式）|
| `memdir/` | 记忆目录（持久化记忆功能）|
| `moreright/` | 未知，命名奇特 |
| `upstreamproxy/` | 上游代理支持 |

## 学习笔记索引

- 工具系统 → [`tools/README.md`](tools/README.md)
- 详细深入分析 → [`../../deep-dives/`](../../deep-dives/)
- 架构总览 → [`../../architecture/overview.md`](../../architecture/overview.md)
