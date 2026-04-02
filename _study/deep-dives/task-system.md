# Task System — 任务体系深度分析

> 核心文件：`src/Task.ts`（125 行）+ `src/tasks/`（8 个任务类型目录）
> 版本：v2.1.88

---

## 一、任务系统概述

Task System 是 Claude Code 的后台异步执行框架，支持在主对话之外并行运行多种任务类型。用户可以通过 `TaskCreateTool`/`TaskGetTool`/`TaskListTool`/`TaskOutputTool`/`TaskStopTool` 等工具管理任务，Claude 也可以直接操作。

---

## 二、核心类型定义（src/Task.ts）

### TaskType — 7 种任务类型

```typescript
export type TaskType =
  | 'local_bash'         // 本地 shell 命令执行
  | 'local_agent'        // 本地子 Agent（隔离的 QueryEngine）
  | 'remote_agent'       // 远程 Agent（CCR 等云端运行）
  | 'in_process_teammate'// 进程内协作 Agent（共享内存）
  | 'local_workflow'     // 本地工作流脚本
  | 'monitor_mcp'        // MCP 监控任务
  | 'dream'              // Dream 异步规划（内部）
```

### TaskStatus — 5 种状态

```typescript
export type TaskStatus =
  | 'pending'    // 已创建，等待启动
  | 'running'    // 正在执行
  | 'completed'  // 成功完成（终态）
  | 'failed'     // 执行失败（终态）
  | 'killed'     // 被用户/系统中止（终态）
```

`isTerminalTaskStatus(status)` 判断终态（completed/failed/killed），用于防止向已完成的 teammate 注入消息、清理 AppState 等。

### 状态机转换

```
pending
  │
  ▼
running ──────────────────────────────────────────┐
  │                    │                          │
  ▼                    ▼                          ▼
completed           failed                     killed
（end_turn）     （error/exception）         （interrupt/stop）
```

### TaskStateBase — 所有任务共享基础字段

```typescript
export type TaskStateBase = {
  id: string           // 格式：{前缀}{8位随机字符}，如 'a3f7k2x9'
  type: TaskType
  status: TaskStatus
  description: string  // 用户可见的任务描述
  toolUseId?: string   // 关联的 tool_use block ID（可选）
  startTime: number    // Unix timestamp（ms）
  endTime?: number
  totalPausedMs?: number
  outputFile: string   // 磁盘输出文件路径（getTaskOutputPath(id)）
  outputOffset: number // 当前读取偏移（流式读取）
  notified: boolean    // 是否已发出 OS 通知
}
```

### Task ID 生成机制

```typescript
const TASK_ID_PREFIXES = {
  local_bash: 'b',           // 'b' 保持向后兼容
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}

// 36^8 ≈ 2.8 万亿种组合，能抵抗暴力符号链接攻击
generateTaskId(type) → '{prefix}{8位小写字母数字}'
```

### Task 接口（用于多态 kill）

```typescript
export type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

注释说明：`spawn/render` 方法曾经是多态的，已在 #22546 中移除。当前仅 `kill` 方法需要多态调度。

---

## 三、7 种任务实现详解

### 1. LocalShellTask（`local_bash`）

**文件：** `src/tasks/LocalShellTask/`

本地 shell 命令异步执行，对应 BashTool 的后台模式。

```typescript
type LocalShellTaskState = TaskStateBase & {
  type: 'local_bash'
  command: string
  result?: { code: number; interrupted: boolean }
  completionStatusSentInAttachment: boolean
  shellCommand: ShellCommand | null    // 正在运行的 shell 进程引用
  unregisterCleanup?: () => void
  cleanupTimeoutId?: NodeJS.Timeout
  lastReportedTotalLines: number       // 增量 delta 跟踪
  isBackgrounded: boolean              // false=前台运行，true=已后台化
  agentId?: AgentId                    // 孤儿任务清理用（Agent 退出时）
  kind?: 'bash' | 'monitor'           // UI 变体：bash显示命令，monitor显示描述
}
```

关键机制：
- 输出流式写入磁盘文件（`outputFile`），`TaskOutputTool` 按 `outputOffset` 增量读取
- `kind='monitor'` 时显示描述而非命令（UI 差异）
- `agentId` 用于 Agent 退出时自动清理其孤儿 bash 任务

### 2. LocalAgentTask（`local_agent`）

**文件：** `src/tasks/LocalAgentTask/LocalAgentTask.tsx`

创建一个完全隔离的子 Agent（独立的 QueryEngine 实例），在独立的 worktree 中运行。

```typescript
type AgentProgress = {
  toolUseCount: number
  tokenCount: number           // latestInputTokens + cumulativeOutputTokens
  lastActivity?: ToolActivity
  recentActivities?: ToolActivity[]  // 最近 5 个工具调用
  summary?: string
}

type ProgressTracker = {
  toolUseCount: number
  latestInputTokens: number    // API 返回值是累积的，只保留最新
  cumulativeOutputTokens: number  // 每轮累加
  recentActivities: ToolActivity[]
}
```

`updateProgressFromMessage()` 从 assistant message 中提取工具调用信息，构建进度摘要：
- Token 计数：输入 token 取最新值（API 是累积的），输出 token 逐轮累加
- 活动追踪：最多保留 5 个最近活动（`MAX_RECENT_ACTIVITIES = 5`）

任务框架工具（`src/utils/task/framework.ts`）：
- `registerTask()` — 注册任务到 AppState
- `updateTaskState<T>()` — 类型安全地更新任务状态
- `POLL_INTERVAL_MS = 1000` — 标准轮询间隔
- `STOPPED_DISPLAY_MS = 3000` — killed 任务展示时长后驱逐
- `PANEL_GRACE_MS = 30000` — 协调者面板中终态任务的宽限期

### 3. RemoteAgentTask（`remote_agent`）

**文件：** `src/tasks/RemoteAgentTask/`

在远程环境（CCR/云端）运行的 Agent，通过 CCR 基础设施管理。依赖 `CCR_AUTO_CONNECT`、`CCR_REMOTE_SETUP` 等 feature flags（外部版禁用）。

### 4. InProcessTeammateTask（`in_process_teammate`）

**文件：** `src/tasks/InProcessTeammateTask/`

进程内协作者任务，与主 Agent 共享内存（`preserveToolUseResults=true`），转录对用户可见。用于需要实时通信的多 Agent 协作场景。

`setAppState` 不是 no-op（与其他子 Agent 不同），确保状态变更能回到根 store。

### 5. LocalWorkflowTask（`local_workflow`）

**文件：** `src/tasks/LocalWorkflowTask/`（依赖 `WORKFLOW_SCRIPTS` feature flag）

执行预定义的工作流脚本，通过 `WorkflowTool` 触发。

### 6. MonitorMcpTask（`monitor_mcp`）

**文件：** `src/tasks/MonitorMcpTask/`

监控 MCP 服务器的任务，用于长期运行的 MCP 会话监控（`MONITOR_TOOL` flag 相关）。

### 7. DreamTask（`dream`）

**文件：** `src/tasks/DreamTask/`

异步规划任务（内部功能，`KAIROS_DREAM` flag 控制）。用于在用户不活跃时执行后台规划。

---

## 四、任务输出流（磁盘 I/O）

```
任务执行
    │
    ├── 输出写入磁盘：getTaskOutputPath(taskId)
    │       路径：~/.claude/tasks/{taskId}.log
    │
    └── TaskOutputTool 读取：
            getTaskOutputDelta(taskId, offset)
            → 从 outputOffset 开始读取新内容（流式增量）
            → 返回 { delta: string, newOffset: number }
```

输出文件的作用：
1. 解耦执行与展示（任务可后台运行，用户随时查询）
2. 进程崩溃后可恢复（磁盘持久化）
3. LocalAgentTask 的输出文件是 symlink，指向 agent 的 transcript 文件

---

## 五、TaskAttachment — 进度通知机制

任务状态变化通过 `TaskAttachment` 注入到消息流：

```typescript
type TaskAttachment = {
  type: 'task_status'
  taskId: string
  toolUseId?: string
  taskType: TaskType
  status: TaskStatus
  description: string
  deltaSummary: string | null  // 自上次 attachment 以来的新输出
}
```

框架标准化了通知逻辑：
- `enqueuePendingNotification()` — 排队发给 Claude 的通知
- `enqueueSdkEvent()` — 向 SDK 消费者广播

---

## 六、后台任务 UI 判断

```typescript
export function isBackgroundTask(task: TaskState): boolean {
  if (task.status !== 'running' && task.status !== 'pending') return false
  if ('isBackgrounded' in task && task.isBackgrounded === false) return false
  return true
}
```

`isBackgrounded=false` 的任务（前台执行中）不计入后台任务指示器，避免 UI 混淆。

---

## 七、AppState 中的任务存储

所有任务状态存储在 `AppState.tasks: TaskState[]`（React Context）：

```
AppState.tasks
    ├── TaskState[] (union type of all 7 task types)
    ├── updateTaskState<T>(taskId, setAppState, updater)  ← 类型安全更新
    └── 终态任务定期清理（STOPPED_DISPLAY_MS 后从 UI 移除）
```

子 Agent 的 `setAppState` 是 no-op（防止子 Agent 污染主线程状态），但 `setAppStateForTasks` 直达根 store（用于任务注册/清理等基础设施操作）。

---

## 八、任务与 Agent 的关系

```
用户调用 AgentTool / TaskCreateTool
    │
    ├── 创建 TaskState（pending）→ 存入 AppState.tasks
    ├── 分配 taskId（带前缀的随机 ID）
    ├── 创建 outputFile（磁盘）
    │
    └── 启动任务：
            LocalAgent → new QueryEngine(subagentConfig)
            LocalShell → spawn ShellCommand
            RemoteAgent → CCR API
```

孤儿清理机制：
- `killShellTasksForAgent(agentId, setAppState)` — Agent 退出时清理其所有 bash 任务
- `cleanupRegistry` — 进程退出时统一清理（`registerCleanup`）

---

## 九、相关文件

| 文件 | 说明 |
|---|---|
| `src/Task.ts` | 类型定义、TaskStateBase、generateTaskId、isTerminalTaskStatus |
| `src/tasks/types.ts` | TaskState 联合类型、isBackgroundTask |
| `src/tasks/LocalShellTask/guards.ts` | LocalShellTaskState 类型 + type guard |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | 子 Agent 创建、进度追踪 |
| `src/utils/task/framework.ts` | registerTask、updateTaskState、常量 |
| `src/utils/task/diskOutput.ts` | getTaskOutputPath、getTaskOutputDelta |
| `src/tools/TaskCreateTool/` | 创建任务的 Tool 实现 |
| `src/tools/TaskOutputTool/` | 读取任务输出（增量）|
| `src/tools/TaskStopTool/` | 终止任务 |
| `src/utils/task/sdkProgress.ts` | emitTaskProgress（向 SDK 广播）|
