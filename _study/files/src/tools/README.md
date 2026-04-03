# src/tools/ — 工具实现目录

每个子目录对应一个 Tool 实现。工具是 Claude 可调用的能力单元。

## 工具注册机制

工具在 `src/tools.ts`（入口聚合文件）中统一导出，
在 `src/main.tsx` 初始化阶段注册到系统中，
在 `QueryEngine` 的每次 API 调用前通过 `getTools()` 加载。

每个工具实现需要：
1. 继承/实现 `Tool`（定义于 `src/Tool.ts`）
2. 声明 `inputSchema`（Zod schema，发给 Claude 作为 tool definition）
3. 声明 `outputSchema`（Zod schema）
4. 实现 `execute()` 方法
5. 声明权限需求（read-only / write / network 等）

## 工具清单

### 文件系统类

| 工具目录 | 功能 |
|---|---|
| `FileReadTool/` | 读文件内容（支持 PDF、图片、Notebook）|
| `FileEditTool/` | 精确字符串替换编辑文件 |
| `FileWriteTool/` | 写入/创建文件 |
| `GlobTool/` | 文件路径模式匹配（类 find）|
| `GrepTool/` | 基于 ripgrep 的内容搜索 |

### 执行类

| 工具目录 | 功能 |
|---|---|
| `BashTool/` | 执行 shell 命令 |
| `PowerShellTool/` | Windows PowerShell 命令执行 |
| `REPLTool/` | 持久化 REPL 会话（保持 shell 状态）|

### Agent & 任务类

| 工具目录 | 功能 |
|---|---|
| `AgentTool/` | 启动子 Agent（general-purpose / Explore / Plan 等）|
| `TaskCreateTool/` | 创建任务 |
| `SendMessageTool/` | 向运行中的 Agent 发送消息 |
| `RemoteTriggerTool/` | 触发远程 Agent |
| `ScheduleCronTool/` | 创建定时任务 |
| `SleepTool/` | 等待（用于轮询场景）|

### 交互类

| 工具目录 | 功能 |
|---|---|
| `AskUserQuestionTool/` | 向用户提问（打断流程等待输入）|
| `SkillTool/` | 调用 Skills（slash command 扩展）|
| `EnterPlanModeTool/` | 进入 Plan 模式 |
| `ExitPlanModeTool/` | 退出 Plan 模式 |
| `EnterWorktreeTool/` | 进入 git worktree 隔离环境 |
| `ExitWorktreeTool/` | 退出 git worktree |

### MCP 类

| 工具目录 | 功能 |
|---|---|
| `MCPTool/` | 调用 MCP server 暴露的工具 |
| `McpAuthTool/` | MCP 认证 |
| `ListMcpResourcesTool/` | 列出 MCP 资源 |
| `ReadMcpResourceTool/` | 读取 MCP 资源 |

### 其他

| 工具目录 | 功能 |
|---|---|
| `LSPTool/` | LSP（语言服务器）集成，获取诊断信息 |
| `NotebookEditTool/` | Jupyter Notebook 编辑 |
| `BriefTool/` | 生成简报 |
| `ConfigTool/` | 配置读写 |
| `SuggestBackgroundPRTool/` | 建议创建后台 PR |
| `SyntheticOutputTool/` | 合成输出（测试/调试用）|

---

## BashTool — Shell 命令执行工具深入

> 目录：[`src/tools/BashTool/`](../../../../../src/tools/BashTool/)（18 个文件，核心文件 BashTool.tsx 1143 行）  
> 角色：执行任意 shell 命令，是 Claude 最常用、最复杂、安全机制最多的工具。

### 目录结构

```
BashTool/
├── BashTool.tsx           ← 主实现（call、权限检查、UI 渲染）
├── BashToolResultMessage.tsx  ← 结果消息渲染组件
├── UI.tsx                 ← 终端 UI（进度、错误、队列提示）
├── bashPermissions.ts     ← 权限规则匹配（路径前缀、通配符）
├── bashSecurity.ts        ← 安全检查（注入检测、危险命令过滤）
├── commandSemantics.ts    ← 命令语义分析（exit code 含义）
├── commentLabel.ts        ← 从命令提取 UI 标签
├── destructiveCommandWarning.ts  ← 不可逆命令警告
├── modeValidation.ts      ← 读写模式约束校验
├── pathValidation.ts      ← 工作目录校验
├── prompt.ts              ← 工具的 system prompt + 超时设置
├── readOnlyValidation.ts  ← 只读约束检查
├── sedEditParser.ts       ← sed 命令语法解析（解析为结构化编辑）
├── sedValidation.ts       ← sed 命令校验
├── shouldUseSandbox.ts    ← 是否启用沙箱判断
├── toolName.ts            ← 工具名常量 BASH_TOOL_NAME = 'Bash'
└── utils.ts               ← 辅助（图片输出、行截断、路径重置等）
```

### 命令执行流程

```
call(input) →
  1. 权限检查（bashToolHasPermission）
  2. 只读约束校验（checkReadOnlyConstraints）
  3. 沙箱判断（shouldUseSandbox）
  4. exec() / sandboxed exec
  5. 输出截断（EndTruncatingAccumulator，TOOL_SUMMARY_MAX_LENGTH）
  6. 图片输出检测（isImageOutput → base64 回注）
  7. 后台化判断（PROGRESS_THRESHOLD_MS = 2000ms → spawnShellTask）
```

### 后台化机制

命令执行超过 2 秒时显示进度提示；在 assistant 模式下，超过 15 秒（`ASSISTANT_BLOCKING_BUDGET_MS`）自动后台化（变为 `local_bash` Task），不阻塞主循环。

### 命令折叠分类（用于 UI 折叠显示）

```typescript
BASH_SEARCH_COMMANDS = ['find', 'grep', 'rg', 'ag', 'ack', 'locate', 'which', 'whereis']
BASH_READ_COMMANDS   = ['cat', 'head', 'tail', 'less', 'more', 'wc', 'stat', 'jq', 'awk', ...]
BASH_LIST_COMMANDS   = ['ls', 'tree', 'du']
BASH_SEMANTIC_NEUTRAL_COMMANDS = ['echo', 'printf', 'true', 'false', ':']
BASH_SILENT_COMMANDS = ['mv', 'cp', 'rm', 'mkdir', 'chmod', 'touch', 'ln', ...]
```

`isSearchOrReadBashCommand()` 分析 pipeline 中每个部分：所有非中性命令都必须是 search/read/list 类，整体才可折叠。

### 安全机制层次

| 层次 | 文件 | 机制 |
|---|---|---|
| 1. 权限规则 | `bashPermissions.ts` | 路径前缀匹配 + 通配符，`/allow` 命令写入 |
| 2. 只读约束 | `readOnlyValidation.ts` | read-only 模式下禁止写操作 |
| 3. 安全检查 | `bashSecurity.ts` | 命令注入检测（AST 解析）、危险命令过滤 |
| 4. 沙箱 | `shouldUseSandbox.ts` + `SandboxManager` | macOS Seatbelt / Linux seccomp |
| 5. 不可逆警告 | `destructiveCommandWarning.ts` | `rm -rf`、`git push --force` 等提示确认 |
| 6. sed 解析 | `sedEditParser.ts` | 将 sed 命令转化为结构化编辑，避免误用 |

### 关键设计点

1. **`sed` 命令特殊处理**：Claude 常用 `sed -i` 编辑文件，但正则替换容易出错。BashTool 检测到 sed 命令后会解析为结构化替换操作，并在失败时给出友好错误。

2. **权限检查 vs 沙箱**：权限规则在调用前检查（`checkPermissions`），沙箱在执行时限制系统调用。两者独立运作，都可能拦截命令。

3. **`commandHasAnyCd`**：若命令含 `cd`，则权限匹配时不能只看命令前缀，需考虑工作目录变化。`bashPermissions.ts` 专门处理此边界情况。

4. **图片输出**：命令输出如为图片（如 `imgcat`、iTerm2 inline image），BashTool 检测并以 base64 附件形式回注，Claude 可直接看到图片内容。

---

## AgentTool — 子 Agent 启动工具深入

> 目录：[`src/tools/AgentTool/`](../../../../../src/tools/AgentTool/)（核心文件 AgentTool.tsx 1397 行）  
> 角色：启动子 Agent（`local_agent`、`remote_agent`、worktree 隔离、in-process teammate），是多 Agent 协作体系的核心入口。

### 目录结构

```
AgentTool/
├── AgentTool.tsx          ← 主实现（schema、call、路由逻辑）
├── agentToolUtils.ts      ← 生命周期辅助（progress tracking、result finalize）
├── agentColorManager.ts   ← 多 Agent 颜色分配（UI 区分）
├── built-in/              ← 内置 Agent 定义（general-purpose、Explore、Plan 等）
├── constants.ts           ← AGENT_TOOL_NAME、ONE_SHOT_BUILTIN_AGENT_TYPES 等
├── forkSubagent.ts        ← worktree fork 机制（隔离副本）
├── loadAgentsDir.ts       ← 从目录加载 Agent 定义文件（支持用户自定义）
├── prompt.ts              ← 工具的 system prompt 片段
├── runAgent.ts            ← 子 Agent 的 query 循环封装
└── UI.tsx                 ← 进度、结果、分组工具调用渲染
```

### 输入 Schema

```typescript
{
  description: string         // 3-5 词任务摘要（必填）
  prompt: string              // 详细任务描述（必填）
  subagent_type?: string      // Agent 类型（"general-purpose" / "Explore" / "Plan" / 用户自定义）
  model?: 'sonnet' | 'opus' | 'haiku'  // 模型覆盖
  run_in_background?: boolean // 后台运行（不阻塞主循环）

  // 多 Agent 模式（ENABLE_AGENT_SWARMS 时）
  name?: string               // 赋名后可通过 SendMessage 发消息
  team_name?: string          // 所属团队
  mode?: PermissionMode       // 权限模式（如 "plan" 强制 Plan 审批）

  // 隔离模式
  isolation?: 'worktree' | 'remote'   // worktree = git 工作树副本；remote = 云端 CCR
  cwd?: string                // 覆盖工作目录（KAIROS 功能）
}
```

Schema 字段通过 feature flag 动态增减（`run_in_background` 在 `DISABLE_BACKGROUND_TASKS` 时隐藏，`cwd` 在非 KAIROS 时隐藏），确保 Claude 不会看到不可用的参数。

### 路由逻辑（call 方法）

```
call(input) →
  ┌─ isolation === 'remote'     → registerRemoteAgentTask()（CCR 云端）
  ├─ isolation === 'worktree'   → createAgentWorktree() + runAsyncAgentLifecycle()
  ├─ run_in_background === true → runAsyncAgentLifecycle()（本地后台）
  ├─ isForkSubagentEnabled()    → forkSubagent()（worktree fork 模式）
  ├─ isAgentSwarmsEnabled()     → spawnTeammate()（in-process teammate）
  └─ 否则                       → runAgent()（同步子 Agent，阻塞直到完成）
```

### 输出类型

```typescript
// 同步完成
{ status: 'completed', result: string, prompt: string, ... }

// 后台启动
{ status: 'async_launched', agentId: string, outputFile: string, ... }

// 进程内队友（swarm 模式，内部类型不暴露给 API）
{ status: 'teammate_spawned', teammate_id: string, tmux_session_name: string, ... }
```

### 内置 Agent 类型

| Agent | 描述 |
|---|---|
| `general-purpose` | 通用 Agent（默认）|
| `Explore` | 代码库探索专用，禁止编辑文件 |
| `Plan` | 架构设计规划，只输出计划不执行 |
| `claude-code-guide` | Claude Code 文档问答 |
| `statusline-setup` | 状态栏配置 |

内置 Agent 定义在 `built-in/` 目录，用户可在 `~/.claude/agents/` 添加自定义定义文件（YAML 格式，支持 `name`、`description`、`model`、`systemPrompt` 字段）。

### Context 隔离

**worktree 模式**：`createAgentWorktree()` 为子 Agent 创建独立的 git worktree——子 Agent 的文件修改不会污染主工作区，任务完成后如无修改自动清理，有修改则保留并返回路径，主 Agent 可选择是否合并。

**消息 fork（`buildForkedMessages`）**：在 worktree/fork 模式下，子 Agent 继承父 Agent 的消息历史副本，但通过 `renderedSystemPrompt` 共享 prompt cache，减少 token 消耗。

### 关键设计点

1. **`lazySchema`**：输入 schema 使用懒加载，避免 feature flag 在模块加载时未初始化的问题。`.omit()` 比条件 spread 更稳定，不会丢失 Zod 字面量类型。

2. **`getAutoBackgroundMs()`**：通过 env var 或 GrowthBook feature gate 控制自动后台化阈值（默认 120s），在函数调用时懒求值（而非模块级），因为 GrowthBook 在模块加载时可能未就绪。

3. **颜色分配**：`agentColorManager` 为每个子 Agent 分配唯一颜色，多 Agent 并发时 UI 可清晰区分各自输出。

4. **`ONE_SHOT_BUILTIN_AGENT_TYPES`**：特定内置 Agent（如 `Explore`、`Plan`）是"一次性"的——主循环在它们完成后不继续多轮对话，直接返回结果。
