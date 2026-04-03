# src/Task.ts — 任务系统类型定义层

> 文件：[`src/Task.ts`](../../../../src/Task.ts)（125 行）  
> 角色：**任务系统的类型契约层**，定义任务类型、状态机枚举、ID 生成规则，以及通用 `Task` 接口。不含任何业务逻辑。

## 职责总览

`Task.ts` 只做三件事：

1. **定义枚举类型**：`TaskType`（7 种任务类型）、`TaskStatus`（5 种状态）
2. **定义数据结构**：`TaskStateBase`（所有任务的基础字段集）、`Task`（多态接口）
3. **提供工具函数**：`generateTaskId()`、`createTaskStateBase()`、`isTerminalTaskStatus()`

---

## 核心类型

### `TaskType` — 7 种任务类型（6–13 行）

```typescript
type TaskType =
  | 'local_bash'          // 本地 shell 命令（BashTool 后台化时创建）
  | 'local_agent'         // 本地子 Agent（AgentTool 启动，在当前进程内运行）
  | 'remote_agent'        // 远程 Agent（在 CCR 云端运行）
  | 'in_process_teammate' // 进程内队友（多 Agent 协作/swarm 场景）
  | 'local_workflow'      // 本地工作流（跨步骤编排）
  | 'monitor_mcp'         // MCP 监控任务
  | 'dream'               // Dream 模式（实验性）
```

### `TaskStatus` — 状态机（15–21 行）

```
pending → running → completed
                 → failed
                 → killed
```

`isTerminalTaskStatus()` 判断是否处于终态（`completed | failed | killed`），用于：
- 防止向已完成的 teammate 注入消息
- 从 AppState 中驱逐已结束的任务
- 孤儿任务清理路径

### `Task` — 多态接口（72–76 行）

```typescript
type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

这是任务系统的多态接口。注意：只有 `kill` 方法是多态的——`spawn` 和 `render` 方法已在历史重构中移除（6 种任务实现的 `kill` 都只需要 `setAppState`，所以 `getAppState/abortController` 已删除）。

### `TaskStateBase` — 所有任务的基础字段（45–57 行）

```typescript
type TaskStateBase = {
  id: string              // 任务 ID（带类型前缀的 base-36 字符串）
  type: TaskType
  status: TaskStatus
  description: string     // 用户可见的任务描述
  toolUseId?: string      // 关联的工具调用 ID（用于结果回注）
  startTime: number       // Unix 时间戳（毫秒）
  endTime?: number
  totalPausedMs?: number  // 暂停的总时长（用于计算实际运行时间）
  outputFile: string      // 任务输出持久化到磁盘的路径
  outputOffset: number    // 已读取的输出字节偏移量
  notified: boolean       // 是否已发送完成通知
}
```

### `TaskContext` — 任务执行上下文（38–42 行）

```typescript
type TaskContext = {
  abortController: AbortController
  getAppState: () => AppState
  setAppState: SetAppState
}
```

任务执行时能访问的最小上下文，刻意比 `ToolUseContext` 精简。

---

## 工具函数

### `generateTaskId(type)` — ID 生成（98–106 行）

```
前缀字母 + 8 位 base-36 随机字符
示例：'a' + 'x7k2m9pq' → 'ax7k2m9pq'（local_agent 任务）
```

**ID 前缀对照表**：

| TaskType | 前缀 |
|---|---|
| `local_bash` | `b`（向后兼容保留）|
| `local_agent` | `a` |
| `remote_agent` | `r` |
| `in_process_teammate` | `t` |
| `local_workflow` | `w` |
| `monitor_mcp` | `m` |
| `dream` | `d` |

**安全性**：36^8 ≈ 2.8 万亿组合，足以抵抗暴力符号链接攻击（针对 `outputFile` 路径）。

### `createTaskStateBase(id, type, description, toolUseId?)` — 创建基础状态（108–125 行）

初始化所有任务共有字段。`outputFile` 路径由 `getTaskOutputPath(id)` 生成（写入磁盘的固定位置）。

---

## 与其他模块的关系

```
Task.ts（类型定义）
    ← tasks/LocalBashTask/   ← BashTool 后台化时创建
    ← tasks/LocalAgentTask/  ← AgentTool 创建子 Agent
    ← tasks/RemoteAgentTask/ ← AgentTool remote 模式
    ← tasks/InProcessTeammateTask/ ← swarm 队友
    ← AppState              ← 任务列表存储在全局状态
    ← TaskCreateTool / TaskListTool / TaskGetTool / ... ← 任务管理工具
```

## 设计要点

1. **类型层与实现层分离**：`Task.ts` 只定义类型，具体的 `spawn/kill` 实现在 `tasks/` 各子目录中，通过 `getTaskByType()` 多态分发。

2. **输出持久化到磁盘**：每个任务有专属的 `outputFile`，输出以追加方式写入，`outputOffset` 跟踪已读位置——这使得主进程可以在任务运行中增量读取输出，也能在进程重启后恢复。

3. **`TaskStatus` 是状态机而非标志位**：只允许单向流转（pending → running → terminal），`isTerminalTaskStatus()` 是所有防重入检查的统一入口。
