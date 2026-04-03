# src/screens/ — 屏幕级组件

> 目录：[`src/screens/`](../../../../../../src/screens/)（3 个文件）  
> 角色：**顶层页面组件**，每个文件对应一个完整的交互屏幕。`REPL.tsx` 是整个 UI 的核心——5005 行，几乎所有交互逻辑汇聚于此。

---

## 文件速览

| 文件 | 行数 | 职责 |
|---|---|---|
| `REPL.tsx` | 5005 | 主交互循环，用户输入 → query → 渲染的完整协调者 |
| `Doctor.tsx` | 574 | `/doctor` 诊断页面（检查 API key、MCP、权限等）|
| `ResumeConversation.tsx` | 398 | 会话恢复选择页面（列出历史会话供选择）|

---

## REPL.tsx — 主交互循环

### Props 接口

```typescript
type Props = {
  commands: Command[]
  debug: boolean
  initialTools: Tool[]
  initialMessages?: MessageType[]
  pendingHookMessages?: Promise<HookResultMessage[]>  // 首次渲染即显示，hook 消息异步注入
  initialFileHistorySnapshots?: FileHistorySnapshot[]
  initialContentReplacements?: ContentReplacementRecord[]
  initialAgentName?: string
  initialAgentColor?: AgentColorName
  mcpClients?: MCPServerConnection[]
  dynamicMcpConfig?: Record<string, ScopedMcpServerConfig>
  autoConnectIdeFlag?: boolean
  strictMcpConfig?: boolean
  systemPrompt?: string
  appendSystemPrompt?: string
  onBeforeQuery?: (input, newMessages) => Promise<boolean>  // 返回 false 可取消
  onTurnComplete?: (messages) => void | Promise<void>
  disabled?: boolean
  mainThreadAgentDefinition?: AgentDefinition
  disableSlashCommands?: boolean
  taskListId?: string              // 任务队列模式（自动处理 task list）
  remoteSessionConfig?: RemoteSessionConfig
  directConnectConfig?: DirectConnectConfig
  sshSession?: SSHSession
  thinkingConfig: ThinkingConfig
}
```

---

### 并发控制：QueryGuard

```typescript
const queryGuard = React.useRef(new QueryGuard()).current
const isQueryActive = useSyncExternalStore(queryGuard.subscribe, queryGuard.getSnapshot)
```

`QueryGuard` 解决的问题：React 状态更新是批处理异步的，而 `isQueryRunning` 需要在同步路径中精确判断是否有 query 在运行。

```
queryGuard.reserve()      // 用户提交后立即预占（同步），防止二次提交
queryGuard.tryStart()     // query 真正开始时原子获取 generation ID
queryGuard.end(gen)       // query 结束时按 generation 原子检查并释放
queryGuard.forceEnd()     // 强制结束（取消时）
queryGuard.cancelReservation()  // 取消预占（processUserInput 返回前中止）
```

`reserve()` 在 `executeUserInput` 中调用，比 `processUserInput` 的异步操作更早执行，消除竞态窗口。

---

### 核心数据流

```
用户输入
    → onSubmit(input, helpers)
        → processUserInput()         — 解析 @文件引用、扩展路径
        → queryGuard.reserve()       — 预占，阻止重复提交
        → executeUserInput()
            → onQuery()
                → queryGuard.tryStart()
                → query(messages, tools, ...)  — src/query.ts
                    → QueryEngine.query()      — 多轮 API 调用
                → queryGuard.end(gen)
```

---

### onSubmit 的主要职责（~400 行）

1. **slash 命令路由**：识别 `/command` 前缀，分发到对应 handler
   - `immediate` 命令在 query 进行中也可执行（如 `/cancel`）
   - `local-jsx` 命令直接返回 JSX，不走 API
   
2. **消息构造**：将输入转为 `UserMessage`，加入 `messages` 数组

3. **投机执行检查**（Speculation）：检查预测的响应是否匹配，若匹配则跳过 API 调用直接采用

4. **agent 路由**：`getActiveAgentForInput()` 决定输入发往 leader 还是某个 teammate

5. **hooks 触发**：`UserPromptSubmit` hook 在提交前触发（deferred hook messages 处理）

---

### 渲染结构（简化）

```
<KeybindingSetup>
  <GlobalKeybindingHandlers>   // Ctrl+C、Ctrl+D、Esc 等
  <CommandKeybindingHandlers>  // /commit、/clear 等命令快捷键
  <CancelRequestHandler>       // 取消运行中的 query

  <Messages>                   // 消息历史（VirtualMessageList）
  <PermissionRequest>          // 权限审批弹出框
  <ElicitationDialog>          // MCP elicitation 弹出
  <PromptInput>                // 用户输入框
  <TaskListV2>                 // 任务面板（可展开/折叠）
  
  // 各种 Dialog：
  <CostThresholdDialog>        // 费用超限确认
  <BypassPermissionsModeDialog>
  <ExportDialog>
  <WorktreeExitDialog>
  ...
</KeybindingSetup>
```

---

### 屏幕模式

```typescript
type Screen = 'prompt' | 'transcript'
```

- `prompt`：正常输入模式（默认）
- `transcript`：浏览历史记录模式（类 less，支持 `/` 搜索、`n/N` 跳转）
  - 进入：Ctrl+R 或 `<` 键
  - 退出：`q` / `Esc`
  - 转录模式下冻结消息快照（`frozenTranscriptState`），不受新消息影响

---

### 关键状态

| 状态 | 说明 |
|---|---|
| `messages` | 完整消息历史（包含 tool use、tool result、system 消息）|
| `streamingToolUses` | 正在流式输出中的 tool use（实时更新）|
| `isLoading` | 由 `queryGuard` 派生（useSyncExternalStore），而非独立 useState |
| `toolPermissionContext` | 当前权限模式和已授权规则 |
| `screen` | 'prompt' \| 'transcript' |
| `speculationState` | 投机执行状态（预测下一步响应）|

---

### 辅助内部组件

```
TranscriptModeFooter    — transcript 模式的底部状态栏
TranscriptSearchBar     — transcript 模式搜索框（/ 键触发）
AnimatedTerminalTitle   — 动态终端标题（显示当前任务描述）
```

---

### 条件加载（feature-gated）

```typescript
// 语音模式：VOICE_MODE 为 false 时 tree-shake 掉
const useVoiceIntegration = feature('VOICE_MODE')
  ? require('../hooks/useVoiceIntegration')
  : () => ({ ... noop ... })

// 挫败感检测：仅内部构建（'ant'）
const useFrustrationDetection = "external" === 'ant'
  ? require('...')
  : () => ({ state: 'closed', ... })

// Coordinator 模式：COORDINATOR_MODE 为 false 时剔除
const getCoordinatorUserContext = feature('COORDINATOR_MODE')
  ? require('../coordinator/...')
  : () => ({})
```

---

## Doctor.tsx — 诊断页面

`/doctor` 命令触发，显示系统诊断信息：
- API key 有效性检查
- MCP 服务器连接状态
- 权限模式和规则
- 当前模型、版本
- 环境变量检查（代理、CA 证书等）

## ResumeConversation.tsx — 会话恢复

`claude --resume` 或启动时发现可恢复会话时触发，显示历史会话列表供选择，支持预览对话摘要。
