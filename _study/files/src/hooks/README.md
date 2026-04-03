# src/hooks/ — React Hooks 目录概览

> 目录：[`src/hooks/`](../../../../../../src/hooks/)（85 个文件，含 2 个子目录）  
> 角色：**UI 逻辑层**，封装所有有状态的交互逻辑，供 `screens/REPL.tsx` 和各组件调用。

---

## 子目录

| 子目录 | 职责 |
|---|---|
| `toolPermission/` | 权限审批流程核心（PermissionContext + 三种模式 handler）|
| `notifs/` | 各类通知 hooks（模型迁移、插件更新、速率限制、npm 废弃等）|

---

## 按功能分类

### 输入处理

| Hook | 职责 |
|---|---|
| `useTextInput.ts` (529 行) | 主文本输入状态管理（光标、选区、行缓冲、组合输入）|
| `useInputBuffer.ts` | 输入缓冲（多字节、粘贴大块文本）|
| `useArrowKeyHistory.tsx` (228 行) | 上/下箭头浏览历史命令 |
| `useHistorySearch.ts` (303 行) | Ctrl+R 历史搜索（类 bash reverse-i-search）|
| `useSearchInput.ts` (364 行) | 消息列表内搜索输入 |
| `useVimInput.ts` | Vim 模式输入（hjkl、模式切换）|
| `usePasteHandler.ts` | 粘贴处理（大段文本、图片粘贴检测）|
| `useTypeahead.tsx` | 预输入补全（键入即提示）|
| `fileSuggestions.ts` | 文件路径 `@` 提及补全建议 |
| `unifiedSuggestions.ts` | 统一补全建议（合并文件/命令/agent 等来源）|

### 键盘绑定

| Hook | 职责 |
|---|---|
| `useGlobalKeybindings.tsx` (248 行) | 全局快捷键（Ctrl+C、Ctrl+D、Esc 等）|
| `useCommandKeybindings.tsx` (107 行) | 命令模式快捷键（/help、/clear 等触发）|
| `useExitOnCtrlCD.ts` | Ctrl+C/D 退出逻辑 |
| `useExitOnCtrlCDWithKeybindings.ts` | 带自定义键绑的退出逻辑 |
| `useDoublePress.ts` | 双击检测（双按 Esc 等）|
| `useCopyOnSelect.ts` | 选中自动复制（macOS 终端协议）|

### 滚动 & 虚拟列表

| Hook | 职责 |
|---|---|
| `useVirtualScroll.ts` (721 行) | 虚拟滚动核心，精细调参：|

`useVirtualScroll` 关键常量：

```typescript
DEFAULT_ESTIMATE = 3     // 未测量项的估算高度（行），故意偏低（宁多渲染不留白）
OVERSCAN_ROWS    = 80    // 视口外额外渲染的行数（防止 long tool results 露白）
COLD_START_COUNT = 30    // ScrollBox 布局前预渲染数量
SCROLL_QUANTUM   = 40    // scrollTop 量化步长（防止每次滚动触发全量 React commit）
PESSIMISTIC_HEIGHT = 1   // 计算覆盖范围时的保底高度
MAX_MOUNTED_ITEMS  = 300 // 最大挂载项数上限
SLIDE_STEP = 25          // 每次 commit 最多新挂载项数（避免 ~290ms 同步阻塞）
```

设计关键：`useSyncExternalStore` + SCROLL_QUANTUM 保证视觉滚动（Ink ScrollBox 直接读 DOM）与 React 状态更新解耦，只在需要移动挂载范围时才触发 React re-render。

### 权限系统（toolPermission/）

```
PermissionContext.ts (388 行)
    — 权限审批的核心状态机，独立于 React（PermissionQueueOps 接口解耦）
    — createResolveOnce<T>: 原子性 claim() 防止并发 resolve 竞争
    — PermissionApprovalSource: hook | user | classifier
    — PermissionRejectionSource: hook | user_abort | user_reject
    — 处理 hooks 自动审批、分类器审批、持久化权限写入

handlers/interactiveHandler.ts (536 行)
    — 交互模式下的权限处理（弹出 PermissionRequest 组件等待用户确认）

handlers/swarmWorkerHandler.ts (159 行)
    — Swarm worker 模式的权限处理（通过 IPC 转发给 coordinator 批准）

handlers/coordinatorHandler.ts (65 行)
    — Coordinator 模式的权限处理
```

### 工具 & 命令合并

| Hook | 职责 |
|---|---|
| `useMergedTools.ts` | 合并本地工具 + MCP 工具列表 |
| `useMergedCommands.ts` | 合并内置 slash 命令 + 插件命令 |
| `useMergedClients.ts` | 合并 MCP 客户端连接列表 |

### 任务 & 后台

| Hook | 职责 |
|---|---|
| `useTasksV2.ts` | 任务列表状态（TaskListV2 面板数据源）|
| `useTaskListWatcher.ts` | 监听任务状态变更（轮询/订阅）|
| `useBackgroundTaskNavigation.ts` | 后台任务切换导航 |
| `useQueueProcessor.ts` (68 行) | 通用队列处理器（顺序消费异步队列）|
| `useScheduledTasks.ts` | 定时任务（cron-style）|
| `useSessionBackgrounding.ts` | 会话后台化（窗口失焦时）|
| `useCommandQueue.ts` | slash 命令执行队列 |

### IDE 集成

| Hook | 职责 |
|---|---|
| `useIDEIntegration.tsx` | IDE 集成核心（VS Code / JetBrains 连接）|
| `useIdeConnectionStatus.ts` | IDE 连接状态监控 |
| `useIdeSelection.ts` | 获取 IDE 当前选中代码 |
| `useIdeAtMentioned.ts` | IDE 中 @ 提及文件的处理 |
| `useIdeLogging.ts` | IDE 操作日志 |
| `useDiffInIDE.ts` | 在 IDE 中展示 diff |

### 多 Agent（Swarm）

| Hook | 职责 |
|---|---|
| `useSwarmInitialization.ts` | 初始化 Swarm 多 Agent 环境 |
| `useSwarmPermissionPoller.ts` | 轮询 Swarm worker 的待批权限 |
| `useTeammateViewAutoExit.ts` | 队友视图自动退出检测 |

### 会话 & 历史

| Hook | 职责 |
|---|---|
| `useAssistantHistory.ts` | 助手回复历史记录 |
| `useFileHistorySnapshotInit.ts` | 文件历史快照初始化 |
| `useTurnDiffs.ts` | 每轮对话的文件 diff 汇总 |
| `useAwaySummary.ts` | 离开后返回时的摘要显示 |

### 语音

| Hook | 职责 |
|---|---|
| `useVoice.ts` | 语音输入核心 |
| `useVoiceEnabled.ts` | 语音功能开关状态 |
| `useVoiceIntegration.tsx` | 语音与文本输入集成 |

### 通知（notifs/）

| Hook | 触发场景 |
|---|---|
| `useRateLimitWarningNotification.tsx` | API 速率限制警告 |
| `useModelMigrationNotifications.tsx` | 模型版本迁移提示 |
| `useNpmDeprecationNotification.tsx` | npm 包废弃通知 |
| `usePluginAutoupdateNotification.tsx` | 插件自动更新通知 |
| `useFastModeNotification.tsx` | Fast mode 切换提示 |
| `useMcpConnectivityStatus.tsx` | MCP 服务器连接状态 |
| `useStartupNotification.ts` | 启动时通知 |
| `useLspInitializationNotification.tsx` | LSP 初始化通知 |

### 杂项

| Hook | 职责 |
|---|---|
| `useSettings.ts` | 读取/监听设置变更 |
| `useSettingsChange.ts` | 设置变更回调 |
| `useDynamicConfig.ts` | 运行时动态配置 |
| `useMemoryUsage.ts` | 内存用量监控 |
| `useTerminalSize.ts` | 终端尺寸（宽/高）变化监听 |
| `useElapsedTime.ts` | 计时器（任务耗时显示）|
| `useMinDisplayTime.ts` | 最小显示时间（防止 UI 闪烁）|
| `useBlink.ts` | 光标闪烁动画 |
| `useAfterFirstRender.ts` | 首次渲染后回调 |
| `useTimeout.ts` | setTimeout 的 hook 封装 |
| `useRemoteSession.ts` | 远程会话状态 |
| `renderPlaceholder.ts` | 输入框占位符渲染 |

---

## 关键设计点

1. **权限系统解耦**：`toolPermission/PermissionContext.ts` 通过 `PermissionQueueOps` 接口与 React 状态解耦，可在非 React 上下文（如单测）中使用。`createResolveOnce` 的 `claim()` 方法在异步 callback 中原子性抢占，防止 hook 审批和用户手动审批同时 resolve。

2. **虚拟滚动量化**：`useVirtualScroll` 使用 `SCROLL_QUANTUM=40` 量化 scrollTop，使 React 只在挂载范围需要移动时才重新渲染，而视觉滚动由 Ink 的 ScrollBox 独立处理。两条渲染路径解耦是高性能的关键。

3. **输入历史**：`useArrowKeyHistory` 和 `useHistorySearch` 各自独立处理两种历史导航模式，通过统一的历史存储交互，避免状态耦合。
