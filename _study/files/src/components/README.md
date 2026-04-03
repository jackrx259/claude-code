# src/components/ — React 组件目录概览

> 目录：[`src/components/`](../../../../../src/components/)（144 个文件/目录）  
> 角色：**终端 UI 组件层**，基于自定义 Ink（React reconciler）实现所有可见 UI 元素。  
> 注：主交互循环组件在 `src/screens/REPL.tsx`（5005 行），不在此目录。

---

## 目录结构（按功能分类）

### 核心渲染

| 文件/目录 | 职责 |
|---|---|
| `App.tsx` | 应用根组件，路由顶层布局（REPL / Teleport / 向导等）|
| `Messages.tsx` | 消息列表容器，协调虚拟滚动 |
| `VirtualMessageList.tsx` | 虚拟化消息列表（只渲染可见区域，性能优化）|
| `Message.tsx` | 单条消息渲染分发（按 message.type 选择渲染方式）|
| `MessageRow.tsx` | 消息行布局（头像、时间戳、内容区）|
| `MessageResponse.tsx` | 助手回复渲染（text + thinking blocks）|
| `MessageModel.tsx` | 模型名称标签显示 |
| `MessageTimestamp.tsx` | 消息时间戳 |
| `MessageSelector.tsx` | 消息选择器（回溯历史、重新提交）|

### 输入组件

| 文件/目录 | 职责 |
|---|---|
| `PromptInput/` | 主输入框（多行、命令历史、自动补全）|
| `TextInput.tsx` | 基础文本输入组件 |
| `BaseTextInput.tsx` | 更底层的输入原语 |
| `VimTextInput.tsx` | Vim 模式输入（hjkl 导航）|

### 工具调用 UI

| 文件/目录 | 职责 |
|---|---|
| `ToolUseLoader.tsx` | 工具调用加载状态显示 |
| `FallbackToolUseErrorMessage.tsx` | 通用工具错误显示（工具未提供自定义渲染时）|
| `FallbackToolUseRejectedMessage.tsx` | 工具被拒绝时的通用显示 |
| `FileEditToolDiff.tsx` | 文件编辑差异（diff 视图）|
| `FileEditToolUpdatedMessage.tsx` | 文件编辑完成消息 |
| `FileEditToolUseRejectedMessage.tsx` | 文件编辑被拒绝消息 |
| `NotebookEditToolUseRejectedMessage.tsx` | Notebook 编辑被拒绝 |

### 权限交互

| 目录 | 职责 |
|---|---|
| `permissions/` | 权限请求对话框（`PermissionRequest.tsx`）、工作进程待批权限（`WorkerPendingPermission.tsx`）|

### 对话框 & 流程

| 文件/目录 | 职责 |
|---|---|
| `AutoUpdater.tsx` | 自动更新提示 |
| `BypassPermissionsModeDialog.tsx` | 绕过权限模式警告 |
| `CostThresholdDialog.tsx` | 费用阈值确认对话框 |
| `ExitFlow.tsx` | 退出确认流程 |
| `ExportDialog.tsx` | 对话导出对话框 |
| `TrustDialog/` | 项目信任确认（首次打开新项目时）|
| `ClaudeMdExternalIncludesDialog.tsx` | CLAUDE.md 外部引用确认 |
| `WorktreeExitDialog.tsx` | worktree 退出确认 |
| `WorkflowMultiselectDialog.tsx` | 工作流多选对话框 |

### 任务 & Agent

| 文件/目录 | 职责 |
|---|---|
| `tasks/` | 任务列表、任务状态显示 |
| `TaskListV2.tsx` | 任务列表 V2（新 UI）|
| `agents/` | 多 Agent 状态、进度显示 |
| `CoordinatorAgentStatus.tsx` | Coordinator Agent 状态面板 |
| `TeammateViewHeader.tsx` | 队友视图头部（多 Agent swarm 模式）|

### MCP

| 目录 | 职责 |
|---|---|
| `mcp/` | MCP 服务器相关 UI（ElicitationDialog、权限等）|
| `MCPServerApprovalDialog.tsx` | MCP 服务器接入审批 |

### 进度 & 状态

| 文件/目录 | 职责 |
|---|---|
| `BashModeProgress.tsx` | Bash 命令执行进度 |
| `CompactSummary.tsx` | 对话压缩后的摘要显示 |
| `ContextVisualization.tsx` | Context window 使用可视化 |
| `EffortCallout.tsx` / `EffortIndicator.ts` | 任务努力程度指示 |
| `StatusNotices.tsx` | 状态栏通知 |
| `TokenWarning.tsx` | Token 用量警告 |
| `DiagnosticsDisplay.tsx` | 诊断信息显示 |

### 传送（Teleport）

| 文件 | 职责 |
|---|---|
| `TeleportProgress.tsx` | 传送进度显示 |
| `TeleportError.tsx` | 传送失败处理 |
| `TeleportResumeWrapper.tsx` | 传送会话恢复包装器 |
| `TeleportStash.tsx` | 传送前 git stash 处理 |
| `TeleportRepoMismatchDialog.tsx` | 仓库不匹配警告 |

### Skills & 配置

| 文件/目录 | 职责 |
|---|---|
| `skills/` | Skills 相关 UI |
| `ThemePicker.tsx` | 主题选择器 |
| `ThinkingToggle.tsx` | 思考模式开关 |
| `ConfigurableShortcutHint.tsx` | 可配置快捷键提示 |

### 差异显示

| 目录 | 职责 |
|---|---|
| `diff/` | 差异渲染（unified diff、并排对比）|
| `StructuredDiff.tsx` / `StructuredDiffList.tsx` | 结构化差异列表 |

### 其他 UI

| 文件/目录 | 职责 |
|---|---|
| `design-system/` | 设计系统基础组件（按钮、颜色、间距）|
| `ui/` | 通用 UI 组件（Select、Modal 等）|
| `CustomSelect/` | 自定义下拉选择组件 |
| `ContextSuggestions.tsx` | 上下文建议（@ 提及文件等）|
| `ClickableImageRef.tsx` | 可点击图片引用 |
| `DevBar.tsx` | 开发者工具栏（调试模式）|
| `Feedback.tsx` / `FeedbackSurvey/` | 反馈收集 |
| `memory/` | Memory 相关 UI（CLAUDE.md 内容展示）|
| `messages/` | 消息子组件（工具结果、代码块等）|
| `shell/` | Shell 输出相关组件 |
| `teams/` | 多 Agent 团队视图 |
| `wizard/` | 新用户引导向导 |
| `grove/` | Grove 特有 UI（内部功能）|

---

## 主交互循环（screens/REPL.tsx）

> ⚠ REPL 主组件在 `src/screens/REPL.tsx`（5005 行），不在 `components/` 目录。

`REPL.tsx` 是整个 UI 的核心协调者：

```
REPL.tsx
├── 处理键盘输入（useInput, useSearchInput）
├── 管理 query 执行（QueryGuard，防止并发提交）
├── 处理权限请求弹出（PermissionRequest）
├── 协调消息滚动（VirtualMessageList + JumpHandle）
├── 管理后台任务通知
└── 渲染：
    ├── <Messages /> — 消息历史
    ├── <PromptInput /> — 输入框
    ├── <TaskListV2 /> — 任务面板
    └── 各种 Dialog（费用警告、导出等）
```

---

## 渲染机制

组件使用自定义 `src/ink/` 渲染器（Ink fork），通过 Yoga 布局引擎将 React Flexbox 转换为终端 ANSI 输出。

**工具自渲染原则**：工具的结果显示（`renderToolResultMessage`、`renderToolUseMessage` 等）由工具自身实现，组件层只调用这些方法，不内嵌工具特定逻辑。
