# Task System — 任务体系

> 核心文件：`src/Task.ts`，实现目录：`src/tasks/`

## Task 类型

`Task.ts` 定义了一个抽象任务模型，支持多种任务类型：

| 类型 | 说明 |
|---|---|
| `LocalBash` | 本地 shell 命令任务 |
| `LocalAgent` | 本地子 Agent 任务 |
| `RemoteAgent` | 远程 Agent 任务（跨机器/进程）|
| `InProcessTeammate` | 进程内协作者（同进程并发 Agent）|
| `LocalWorkflow` | 本地工作流任务 |
| `MonitorMCP` | MCP 监控任务 |
| `Dream` | Dream 类型任务（推测为异步/后台规划）|

## 状态机

```
pending
   │
   ▼
running
   │
   ├──→ completed   (正常完成)
   ├──→ failed      (执行出错)
   └──→ killed      (用户中止)
```

## 输出持久化

任务输出持久化到磁盘，这意味着：
- 长时间运行的任务可以被查询历史输出
- 进程重启后可以恢复任务状态（查询，不一定能继续）
- `TaskOutput` 工具可以读取已完成任务的输出

## 与工具的关系

任务系统通过工具暴露给 Claude：

| 工具 | 操作 |
|---|---|
| `TaskCreateTool` | 创建并启动任务 |
| `TaskGet` | 查询任务状态 |
| `TaskList` | 列出所有任务 |
| `TaskOutput` | 获取任务输出 |
| `TaskStop` | 停止任务 |
| `TaskUpdate` | 更新任务状态 |
| `SendMessageTool` | 向运行中的 Agent 任务发消息 |

## Worktree 隔离

`EnterWorktreeTool` / `ExitWorktreeTool` 配合任务系统使用：
- 为任务创建独立的 git worktree
- Agent 在隔离副本上工作，不影响主分支
- 任务完成后 worktree 自动清理（无变更时）或保留（有变更时返回分支名）

## 关键问题待研究

- [ ] `Task.ts` 中任务的具体数据结构
- [ ] 输出持久化到哪个目录？格式？
- [ ] RemoteAgent 如何跨进程通信？
- [ ] InProcessTeammate 的并发模型（shared context？）
- [ ] Dream 类型的实际用途

## 相关文件

- 基础类型：[`src/Task.ts`](../../src/Task.ts)
- 任务聚合：[`src/tasks.ts`](../../src/tasks.ts)
- 任务目录：[`src/tasks/`](../../src/tasks/)
- 工具实现：[`src/tools/TaskCreateTool/`](../../src/tools/TaskCreateTool/)