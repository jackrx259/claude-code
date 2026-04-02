# Tool System — 工具体系深度分析

> 核心文件：`src/Tool.ts`（792 行）+ `src/tools/`（190 个文件，44 个工具目录）
> 版本：v2.1.88

---

## 一、Tool 类型定义（src/Tool.ts）

### 核心 Tool 接口

每个工具是一个满足 `Tool<Input, Output, P>` 类型的对象（泛型参数分别为输入 Schema、输出类型、进度类型）：

**必须实现的方法：**

| 方法/字段 | 说明 |
|---|---|
| `name: string` | 工具名，发给 Claude API 的 tool name |
| `inputSchema: Input` | Zod Schema（API 发送时转为 JSON Schema）|
| `call(args, context, canUseTool, parentMessage, onProgress)` | 实际执行逻辑，返回 `Promise<ToolResult<Output>>` |
| `description(input, options)` | 工具描述（Claude 据此决定是否调用）|
| `prompt(options)` | 注入系统提示词的说明文本 |
| `checkPermissions(input, context)` | 工具级权限检查（补充通用权限系统）|
| `isConcurrencySafe(input)` | 是否允许并发执行 |
| `isReadOnly(input)` | 是否只读（影响 Plan Mode 屏蔽逻辑）|
| `mapToolResultToToolResultBlockParam(content, toolUseID)` | 序列化结果为 API tool_result 格式 |
| `renderToolUseMessage(input, options)` | 渲染工具调用信息（React 组件）|
| `userFacingName(input)` | 用户界面显示的工具名 |
| `toAutoClassifierInput(input)` | Auto 模式安全分类器的输入格式 |
| `maxResultSizeChars: number` | 结果超限后持久化到磁盘（Infinity = 不持久化）|

**可选方法（有则覆盖默认行为）：**

| 方法 | 说明 |
|---|---|
| `aliases?: string[]` | 工具名别名（向后兼容重命名）|
| `searchHint?: string` | ToolSearch 关键词匹配提示（3-10 词）|
| `validateInput(input, context)` | 输入合法性检查，失败则告知模型原因 |
| `isDestructive(input)` | 是否不可逆操作（delete/overwrite/send）|
| `isSearchOrReadCommand(input)` | 返回 `{isSearch, isRead, isList}`，影响 UI 折叠显示 |
| `isOpenWorld(input)` | 是否开放世界（影响 context 计算）|
| `interruptBehavior()` | 用户中断时行为：`'cancel'` 或 `'block'`（默认 block）|
| `getPath(input)` | 操作的文件路径（权限 rule 匹配使用）|
| `preparePermissionMatcher(input)` | 为 hook `if` 条件准备 pattern 匹配器 |
| `backfillObservableInput(input)` | 序列化前填充遗留/派生字段（幂等，不改变原始 input）|
| `shouldDefer?: boolean` | 延迟加载（需 ToolSearch 才能调用）|
| `alwaysLoad?: boolean` | 永远不延迟，首次 prompt 就暴露 |
| `strict?: boolean` | API strict 模式 |
| `inputsEquivalent(a, b)` | 判断两次调用是否等价（用于去重/缓存）|
| `renderToolResultMessage(...)` | 渲染结果（React）|
| `renderToolUseProgressMessage(...)` | 渲染进度 UI |
| `renderToolUseRejectedMessage(...)` | 渲染拒绝 UI |
| `renderGroupedToolUse(...)` | 批量渲染多个并行调用 |
| `getToolUseSummary(input)` | 紧凑摘要（compact view）|
| `getActivityDescription(input)` | spinner 文字（如 "Reading src/foo.ts"）|
| `isTransparentWrapper()` | 透明包装器（REPL 工具，不自渲染）|

### buildTool() — 统一工厂函数

所有工具通过 `buildTool<D>(def: D): BuiltTool<D>` 创建，自动填入安全默认值：

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,  // 默认不并发（fail-closed）
  isReadOnly: () => false,          // 默认写操作（fail-closed）
  isDestructive: () => false,
  checkPermissions: () => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',  // 默认跳过分类器
  userFacingName: () => def.name,
}
```

`BuiltTool<D>` 类型通过 TypeScript 泛型在类型层面镜像 `{ ...TOOL_DEFAULTS, ...def }` 的运行时语义，确保 60+ 工具的类型安全。

---

## 二、ToolUseContext — 工具执行上下文

`ToolUseContext` 是工具执行时拿到的完整上下文，字段极其丰富：

```
ToolUseContext {
  options: {
    commands, tools, mainLoopModel, thinkingConfig,
    mcpClients, mcpResources, isNonInteractiveSession,
    agentDefinitions, maxBudgetUsd, customSystemPrompt, ...
  }
  abortController              ← 中断信号
  readFileState: FileStateCache  ← 文件读取 LRU 缓存
  getAppState() / setAppState()  ← React Context 状态
  setAppStateForTasks()        ← 专用于任务的状态写入（子 Agent 的 setAppState 是 no-op）
  handleElicitation?()         ← URL elicitation（MCP -32042 错误处理）
  setToolJSX?()                ← 更新工具的 JSX 渲染
  addNotification?()
  appendSystemMessage?()       ← 只写 UI-only 系统消息（不发给 API）
  nestedMemoryAttachmentTriggers  ← 触发 CLAUDE.md 嵌套加载
  loadedNestedMemoryPaths      ← 已加载路径去重
  discoveredSkillNames         ← skill 发现追踪（遥测）
  messages: Message[]          ← 当前完整消息历史
  fileReadingLimits / globLimits
  toolDecisions: Map           ← 工具决策缓存（source+decision+timestamp）
  queryTracking                ← 查询链追踪（chainId + depth）
  requestPrompt?()             ← 向用户发起交互提示（仅 REPL 模式）
  agentId?                     ← 子 Agent ID（仅子 Agent 有）
  agentType?                   ← 子 Agent 类型名
  contentReplacementState      ← 工具结果预算的内容替换状态
  renderedSystemPrompt?        ← 父 Agent 的系统提示词快照（fork 用）
  localDenialTracking?         ← 异步子 Agent 的本地拒绝追踪
}
```

---

## 三、权限模型

### 权限检查流程

```
query.ts 准备执行工具：
    │
    ▼
validateInput(input, context)     ← 1. 输入合法性（tool 自身）
    │失败 → 告知模型，不弹权限框
    ▼
canUseTool(tool, input, context)  ← 2. 通用权限系统（hooks/useCanUseTool.ts）
    │
    ├── tool.checkPermissions(input, context)  ← 3. 工具级权限（tool 自身）
    ├── alwaysAllowRules → allow
    ├── alwaysDenyRules  → deny
    ├── permissionMode === 'bypassPermissions' → allow（Auto 模式）
    ├── shouldAvoidPermissionPrompts → auto-deny（后台 Agent）
    └── 弹权限确认 UI（REPL 模式）
```

### ToolPermissionContext — 权限上下文

```typescript
type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode             // 'default' | 'bypassPermissions' | 'acceptEdits' | etc.
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource  // 按来源分组的永久允许规则
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean      // Auto 模式是否可用
  shouldAvoidPermissionPrompts?: boolean         // 后台 Agent 自动拒绝
  awaitAutomatedChecksBeforeDialog?: boolean     // 展示 UI 前等自动检查
  prePlanMode?: PermissionMode                  // Plan Mode 前的模式备份
}>
```

规则来源（`ToolPermissionRulesBySource`）：包括 `command`（slash 命令设置）、`settings`（配置文件）、`hook`（hooks 脚本），分源管理便于撤销特定来源的规则。

---

## 四、44 个工具实现分类

### 文件操作（核心）

| 工具 | 目录 | 关键能力 |
|---|---|---|
| `FileReadTool` | `FileReadTool/` | 读文件，支持行范围，`maxResultSizeChars=Infinity`（不持久化）|
| `FileEditTool` | `FileEditTool/` | 字符串替换编辑（old_string → new_string），LSP 集成 |
| `FileWriteTool` | `FileWriteTool/` | 新建/覆写文件 |
| `GlobTool` | `GlobTool/` | 文件名 glob 匹配，`isConcurrencySafe=true`, `isReadOnly=true` |
| `GrepTool` | `GrepTool/` | 内容 regex 搜索（基于 ripgrep），`isReadOnly=true` |
| `NotebookEditTool` | `NotebookEditTool/` | Jupyter notebook 编辑 |

### Shell 执行

| 工具 | 说明 |
|---|---|
| `BashTool` | 核心：执行 shell 命令，有超时、沙箱、sedEdit 解析、破坏性命令检测 |
| `PowerShellTool` | Windows PowerShell（`POWERSHELL_AUTO_MODE` flag）|
| `REPLTool` | 持久化 REPL 会话（Python/JS 等），透明包装器 |

### Agent & Task 编排

| 工具 | 说明 |
|---|---|
| `AgentTool` | 创建子 Agent（隔离的 QueryEngine 实例），支持 LocalAgent/RemoteAgent |
| `TaskCreateTool` | 创建后台任务 |
| `TaskGetTool` | 查询任务状态 |
| `TaskListTool` | 列出任务列表 |
| `TaskOutputTool` | 获取任务输出 |
| `TaskStopTool` | 终止任务 |
| `TaskUpdateTool` | 更新任务状态 |
| `SendMessageTool` | 向 Agent 发送消息 |
| `RemoteTriggerTool` | 触发远程 Agent |

### Plan Mode 控制

| 工具 | 说明 |
|---|---|
| `EnterPlanModeTool` | 进入 Plan Mode（模型发起，禁用写工具）|
| `ExitPlanModeTool` | 退出 Plan Mode |
| `VerifyPlanExecutionTool` | 验证计划执行（Ant-only）|
| `EnterWorktreeTool` | 进入 git worktree 隔离环境 |
| `ExitWorktreeTool` | 退出 worktree |

### MCP & 外部集成

| 工具 | 说明 |
|---|---|
| `MCPTool` | MCP 工具适配器（isMcp=true，动态从 MCP server 加载）|
| `ListMcpResourcesTool` | 列出 MCP 资源 |
| `ReadMcpResourceTool` | 读取 MCP 资源 |
| `McpAuthTool` | MCP 认证 |

### 搜索 & 网络

| 工具 | 说明 |
|---|---|
| `WebFetchTool` | 抓取网页内容 |
| `WebSearchTool` | 网络搜索 |
| `ToolSearchTool` | 搜索延迟加载的工具（`shouldDefer` 工具需先 ToolSearch）|

### 用户交互

| 工具 | 说明 |
|---|---|
| `AskUserQuestionTool` | 向用户提问（`requiresUserInteraction=true`）|
| `ConfigTool` | 读写配置 |
| `TodoWriteTool` | 管理 todo 列表（结果不渲染在 transcript，直接更新 todo panel）|

### LSP & 代码智能

| 工具 | 说明 |
|---|---|
| `LSPTool` | Language Server Protocol 诊断 |

### 其他

| 工具 | 说明 |
|---|---|
| `SkillTool` | 执行预设 Skill（`MCP_SKILLS` flag 启用）|
| `WorkflowTool` | 工作流执行（`WORKFLOW_SCRIPTS` flag）|
| `SleepTool` | 等待/延迟（特殊：`SLEEP_TOOL_NAME` 常量）|
| `SyntheticOutputTool` | 结构化输出收集（jsonSchema 约束时注入）|
| `BriefTool` | 生成简报（Ant-only）|
| `TungstenTool` | 未知内部工具 |
| `SuggestBackgroundPRTool` | 建议创建后台 PR |
| `ScheduleCronTool` | 定时任务调度 |
| `TeamCreateTool/TeamDeleteTool` | 团队协作（TEAMMEM flag）|

---

## 五、工具执行并发机制

工具调用由 `src/services/tools/toolOrchestration.ts` 的 `runTools()` 编排：

```
Claude 返回多个 tool_use blocks：
    │
    ├── isConcurrencySafe(input) === false（任意一个）
    │       → 串行逐一执行
    │
    └── 全部 isConcurrencySafe(input) === true
            → StreamingToolExecutor 并发执行
```

**并发安全工具**（`isConcurrencySafe=true`）：GlobTool、GrepTool、FileReadTool 等只读工具。  
**非并发安全工具**：BashTool、FileEditTool、AgentTool 等（默认 `false`）。

---

## 六、工具结果大小管理

`maxResultSizeChars` 控制单条工具结果的大小：

```
工具执行完毕：
    │
    ├── result.length <= maxResultSizeChars → 直接注入 messages
    └── result.length > maxResultSizeChars
            → 写入临时文件
            → Claude 收到文件路径 + 预览（不是完整内容）
            → contentReplacementState 记录替换信息（用于 resume）
```

`maxResultSizeChars = Infinity` 的工具（如 FileReadTool）：不持久化，因为持久化会导致 Read→file→Read 循环，且工具本身已有长度限制。

---

## 七、延迟加载（shouldDefer）

当 ToolSearch 功能激活时，`shouldDefer=true` 的工具不会出现在初始 prompt 中：

```
初始 prompt：只包含 alwaysLoad=true 的工具 + ToolSearchTool
    │
    Claude 调用 ToolSearchTool("keyword")
    │
    ▼
ToolSearchTool 返回匹配工具的完整 schema
    │
    Claude 可以调用该工具
```

目的：减少初始 prompt 的 token 用量，让不常用工具按需加载。

---

## 八、MCP 工具集成

MCP 工具通过 `MCPTool` 适配器统一接入，对 QueryEngine 透明：

```typescript
// MCPTool 的 isMcp=true 标志
// 工具名格式：mcp__serverName__toolName
// 或 unprefixed 模式（CLAUDE_AGENT_SDK_MCP_NO_PREFIX）：直接使用 toolName
```

`mcpInfo?: { serverName: string; toolName: string }` 保存原始名（未规范化），无论是否有前缀都存在。

---

## 九、设计亮点

### 1. fail-closed 默认值
`isConcurrencySafe=false`（默认串行）、`isReadOnly=false`（默认写操作），确保安全前提下的最保守行为。

### 2. `backfillObservableInput` — 向后兼容
工具输入在序列化前（发给 SDK stream、transcript、hooks）可通过此方法填入遗留/派生字段。原始 API bound input 永不修改，保护 prompt cache。幂等要求确保多次调用安全。

### 3. 双渲染路径
每个工具同时实现 `renderToolUseMessage`（调用时）和 `renderToolResultMessage`（结果时），分别对应 UI 的 "正在执行" 和 "已完成" 两个状态。

### 4. `contextModifier`（ToolResult 字段）
工具可以返回 `contextModifier: (ctx) => ctx`，修改后续工具的执行上下文（仅非并发安全工具生效）。例如 FileEditTool 触发后刷新 LSP 状态。

---

## 十、相关文件

| 文件 | 说明 |
|---|---|
| `src/Tool.ts` | Tool 接口、ToolUseContext、ToolPermissionContext、buildTool |
| `src/tools/` | 各工具实现目录（44 个工具）|
| `src/services/tools/toolOrchestration.ts` | `runTools()` 工具并发编排 |
| `src/services/tools/StreamingToolExecutor.ts` | 并发执行引擎 |
| `src/hooks/useCanUseTool.ts` | 权限检查主逻辑 |
| `src/utils/permissions/filesystem.ts` | 文件系统权限（checkWritePermissionForTool）|
| `src/utils/toolResultStorage.ts` | 结果大小管理（applyToolResultBudget）|
