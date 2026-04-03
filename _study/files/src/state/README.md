# src/state/ + src/context/ — 全局状态管理

> 目录：[`src/state/`](../../../../../../src/state/)（6 文件）+ [`src/context/`](../../../../../../src/context/)（9 文件）  
> 角色：**全局状态层**，state/ 是唯一真实数据源（AppState），context/ 提供 React Context 封装的辅助状态。

---

## state/ 目录

### 文件速览

| 文件 | 行数 | 职责 |
|---|---|---|
| `AppStateStore.ts` | 569 | `AppState` 类型定义 + 默认值工厂 + Store 实例 |
| `AppState.tsx` | 199 | React Context 封装（Provider、hooks）|
| `store.ts` | 34 | 通用 `Store<T>` 实现（发布-订阅，无框架依赖）|
| `selectors.ts` | 76 | 纯函数 selectors（派生状态）|
| `onChangeAppState.ts` | 171 | AppState 变更副作用（持久化、analytics 上报）|
| `teammateViewHelpers.ts` | 141 | Swarm 队友视图辅助函数 |

---

### store.ts — 极简状态容器

```typescript
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: () => void) => () => void
}
```

- `setState` 接收 updater 函数（immutable 更新），`Object.is` 判断是否真正变更，无变化不触发监听器
- 发布-订阅模式（`Set<Listener>`），无 Redux/Zustand 等第三方依赖
- `onChange` 回调：AppState 在创建时传入 `onChangeAppState`，用于触发持久化和日志

---

### AppStateStore.ts — AppState 类型

`AppState` 是整个应用的单一状态树，用 `DeepImmutable<{...}>` 包裹（绝大部分字段不可直接修改）。

#### 核心字段分类

**UI 显示**
```typescript
settings: SettingsJson           // 用户设置（主题、权限模式等）
verbose: boolean
mainLoopModel: ModelSetting      // 当前使用的模型
statusLineText: string | undefined
expandedView: 'none' | 'tasks' | 'teammates'
footerSelection: FooterItem | null  // 底部导航焦点
spinnerTip?: string
```

**任务 & 多 Agent**
```typescript
tasks: { [taskId: string]: TaskState }   // 所有任务（排除在 DeepImmutable 外，因含函数）
agentNameRegistry: Map<string, AgentId>  // name → AgentId（SendMessage 路由用）
foregroundedTaskId?: string
viewingAgentTaskId?: string              // 当前正在查看哪个 agent 的 transcript
coordinatorTaskIndex: number             // Coordinator 面板选中行
```

**MCP & 插件**
```typescript
mcp: {
  clients: MCPServerConnection[]
  tools: Tool[]
  commands: Command[]
  resources: Record<string, ServerResource[]>
  pluginReconnectKey: number             // /reload-plugins 自增，触发 effects 重跑
}
plugins: {
  enabled: LoadedPlugin[]
  disabled: LoadedPlugin[]
  commands: Command[]
  errors: PluginError[]
  installationStatus: { ... }
  needsRefresh: boolean
}
```

**Bridge（远程控制）**
```typescript
replBridgeEnabled: boolean
replBridgeConnected: boolean         // "Ready"：session 已创建
replBridgeSessionActive: boolean     // "Connected"：用户在 claude.ai
replBridgeReconnecting: boolean
replBridgeConnectUrl: string | undefined
replBridgeSessionUrl: string | undefined
```

**推测执行（Speculation）**
```typescript
SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }   // 可变 ref，避免每条消息 spread
      writtenPathsRef: { current: Set<string> }
      boundary: CompletionBoundary | null
      ...
    }
```

**其他子系统**
```typescript
thinkingEnabled: boolean | undefined
promptSuggestionEnabled: boolean
notifications: { current: Notification | null; queue: Notification[] }
elicitation: { queue: ElicitationRequestEvent[] }
todos: { [agentId: string]: TodoList }
fileHistory: FileHistoryState
attribution: AttributionState
sessionHooks: SessionHooksState
replContext?: { vmContext, registeredTools, console }  // REPL 工具 VM 沙箱
tungstenActiveSession?: { ... }  // Tmux 集成（codename tungsten）
bagelActive?: boolean            // WebBrowser 工具（codename bagel）
computerUseMcpState?: { ... }    // Computer Use MCP（codename chicago）
```

---

### AppState.tsx — React Context 封装

```typescript
// Provider（在 main.tsx / REPL.tsx 顶层挂载）
<AppStateProvider initialState={...}>

// Hooks（消费方）
useAppState()        → AppState（只读快照，触发 re-render）
useAppStateStore()   → Store<AppState>（直接访问 store，可 getState + subscribe）
useSetAppState()     → (updater) => void（触发更新）
```

内部通过 `useSyncExternalStore` 将非 React 的 `Store<T>` 接入 React 渲染周期，保证 concurrent mode 下的安全读取。

---

### selectors.ts — 纯函数状态派生

```typescript
getViewedTeammateTask(appState)
    → InProcessTeammateTaskState | undefined
    // 当前查看的队友 task（viewingAgentTaskId → tasks 查找）

getActiveAgentForInput(appState)
    → { type: 'leader' }
    | { type: 'viewed', task: InProcessTeammateTaskState }
    | { type: 'named_agent', task: LocalAgentTaskState }
    // 决定用户输入应路由到哪个 agent
```

---

## context/ 目录

辅助 React Context，覆盖 state/ 中没有（或不适合放入）的 UI 级状态。

| 文件 | 职责 |
|---|---|
| `notifications.tsx` (239 行) | 通知队列（优先级排序、fold 合并、超时自动消失）|
| `stats.tsx` (219 行) | 性能/用量统计（token 计数、API 耗时、渲染帧率）|
| `overlayContext.tsx` (150 行) | 全屏 overlay 层（ThemePicker、ExportDialog 等弹层）|
| `promptOverlayContext.tsx` (124 行) | 提示框 overlay（输入框上方的提示覆盖层）|
| `QueuedMessageContext.tsx` (62 行) | 排队消息上下文（消息缓冲发送）|
| `modalContext.tsx` (57 行) | 模态框状态（全局唯一弹出层管理）|
| `voice.tsx` (87 行) | 语音上下文（录音状态、音量、转录）|
| `mailbox.tsx` (37 行) | Mailbox 消息通道（agent 间消息传递）|
| `fpsMetrics.tsx` (29 行) | 帧率指标（开发调试用）|

### 通知系统（notifications.tsx）

```typescript
type Notification = TextNotification | JSXNotification

type BaseNotification = {
  key: string
  priority: 'low' | 'medium' | 'high' | 'immediate'
  timeoutMs?: number      // 默认 8000ms
  invalidates?: string[]  // 此通知会使哪些 key 的通知失效
  fold?: (accumulator, incoming) => Notification  // 相同 key 的通知合并策略
}
```

通知从 `AppState.notifications.queue` 中按优先级取出，`immediate` 优先级会清除当前正在显示的通知。

---

## 依赖关系

```
store.ts
    ← AppStateStore.ts（createStore(initialAppState, onChangeAppState)）
        ← AppState.tsx（React Context 封装，useSyncExternalStore）
            ← REPL.tsx、main.tsx（Provider 挂载）
            ← 所有消费组件（useAppState / useSetAppState）
```

`context/` 中的 Context 独立于 `state/`，在各自的 Provider 中管理，通过 hook 暴露给子组件。
