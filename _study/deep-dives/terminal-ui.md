# Terminal UI — 终端渲染体系深度分析

> 核心目录：`src/ink/`（96 文件）、`src/components/`（146 组件）、`src/hooks/`（87 hooks）
> 版本：v2.1.88

---

## 一、整体架构

Claude Code 的 UI 层基于 React，通过自定义的终端 Reconciler 渲染到终端：

```
用户输入 / API 响应
        │
        ▼
React 组件树（src/components/）
        │  setState / useReducer
        ▼
自定义 Ink Reconciler（src/ink/reconciler.ts）
        │  React Fiber → 虚拟 DOM → Yoga 布局
        ▼
虚拟 Screen Buffer（src/ink/screen.ts）
        │  diff 算法，只更新变化的单元格
        ▼
终端 ANSI 控制序列（writeDiffToTerminal）
        │
        ▼
用户看到的 TUI 界面
```

---

## 二、src/ink/ — 深度定制的 Ink

### 为什么 fork 而不是用 npm 包？

原版 [Ink](https://github.com/vadimdemedes/ink) 是功能完整的终端 React 渲染器，但 Claude Code 对它做了大量深度修改，无法通过上游 PR 满足需求：

1. **逐单元格 Screen Buffer 渲染**：`screen.ts` 实现了像素级的 `CharPool`、`StylePool`、`HyperlinkPool` 对象池，支持 diff 更新
2. **帧率控制**：`FRAME_INTERVAL_MS = 16`（约 60fps），通过 `throttle` 限流，避免流式响应时频繁重绘
3. **Alt Screen 支持**：切换到终端备用屏幕，完整 fullscreen TUI 体验
4. **选中/复制**：`selection.ts`（~800 行）实现了鼠标/键盘文本选择和复制（`getSelectedText`、`captureScrolledRows`）
5. **搜索高亮**：`searchHighlight.ts`、`scanPositions` 在 screen buffer 上叠加高亮层
6. **双向文字（BiDi）**：`bidi.ts` 处理阿拉伯语/希伯来语等 RTL 文本
7. **Scrollbox**：`components/ScrollBox.tsx` 实现可滚动的内容区域
8. **鼠标事件**：`events/click-event.ts`、`hit-test.ts` 实现点击和悬停
9. **终端 focus 状态**：`terminal-focus-state.ts` 追踪终端窗口是否 focused
10. **React Devtools 支持**：`devtools.ts`（development 模式下）

### 关键文件功能

| 文件 | 说明 |
|---|---|
| `ink.tsx` | Ink 实例（`class Ink`），持有 FiberRoot、LogUpdate、Screen 等引用 |
| `reconciler.ts` | React Reconciler（`createReconciler`），将 React 操作映射到虚拟 DOM |
| `dom.ts` | 虚拟 DOM 节点定义（DOMElement、TextNode），Yoga 节点管理 |
| `renderer.ts` | 帧渲染循环，将 DOM 树渲染为 Frame |
| `screen.ts` | 逐单元格 screen buffer，CharPool/StylePool/HyperlinkPool 对象池 |
| `render-node-to-output.ts` | DOM 节点 → 字符串输出，滚动跟踪 |
| `render-to-screen.ts` | 将输出写入 screen buffer，支持搜索高亮叠加 |
| `terminal.ts` | 终端抽象，`writeDiffToTerminal`（只写变化的部分）|
| `optimizer.ts` | Frame diff 优化，减少终端写入量 |
| `layout/engine.ts` + `yoga.ts` | Yoga（Facebook CSS 布局引擎）适配层 |
| `selection.ts` | 文本选择状态机（鼠标/键盘，~800 行）|
| `parse-keypress.ts` | 原始键盘输入解析（ANSI escape codes）|
| `log-update.ts` | 行内更新（不切换 alt screen 时）|
| `frame.ts` | Frame 数据结构（渲染结果快照）|
| `instances.ts` | 多 Ink 实例管理（全局单例集合）|

### 渲染循环详解

```
ink.tsx → Ink 实例
    │
    ├── throttle(render, FRAME_INTERVAL_MS=16)  ← 帧率限制
    │
    └── render() 每帧执行：
            1. reconciler 提交 React 更新（Fiber commit phase）
            2. renderer 将 DOM 树转为 Frame（layout via Yoga）
            3. optimizer.optimize(diff) 最小化终端写入
            4. writeDiffToTerminal()  ← 只写变化的单元格
```

### 布局引擎：Yoga

`src/ink/layout/yoga.ts` 将 Yoga（Facebook 的 C 实现，TypeScript 包装）适配到自定义 `LayoutNode` 接口：

- Flexbox 布局（flex-direction、justify-content、align-items 等）
- 自定义测量函数（`LayoutMeasureFunc`）用于文字宽度
- 每次 React commit 后触发 Yoga 重新计算布局

---

## 三、src/ink/components/ — Ink 原语组件

这些是 Ink 内置的底层 React 组件（不是业务组件）：

| 组件 | 说明 |
|---|---|
| `Box.tsx` | 基础布局容器（类 `<div>`，Flexbox）|
| `Text.tsx` | 文本节点（支持颜色、加粗、下划线等）|
| `Newline.tsx` | 换行 |
| `Spacer.tsx` | 弹性空白 |
| `App.tsx` | 根节点（错误边界 + StdinContext）|
| `ScrollBox.tsx` | 可滚动区域（行级滚动）|
| `AlternateScreen.tsx` | Alt screen 模式切换 |
| `Button.tsx` | 可聚焦按钮 |
| `Link.tsx` | 超链接（OSC 8，终端支持时）|
| `RawAnsi.tsx` | 直接输出 ANSI 序列（转义 Ink 渲染管线）|
| `NoSelect.tsx` | 禁止选中区域 |
| `ErrorOverview.tsx` | Ink 级别错误展示 |

---

## 四、src/ink/hooks/ — Ink 内置 Hooks

| Hook | 说明 |
|---|---|
| `use-input.ts` | 键盘输入监听（最核心的输入 hook）|
| `use-stdin.ts` | 原始 stdin 流读取 |
| `use-app.ts` | 访问 Ink 应用实例（quit 等）|
| `use-terminal-focus.ts` | 监听终端窗口 focus/blur |
| `use-terminal-viewport.ts` | 获取终端尺寸（columns×rows）|
| `use-selection.ts` | 文本选中状态 |
| `use-search-highlight.ts` | 搜索高亮控制 |
| `use-animation-frame.ts` | 按帧动画（`FRAME_INTERVAL_MS` 对齐）|
| `use-interval.ts` | setInterval 封装 |
| `use-declared-cursor.ts` | 游标声明 |
| `use-terminal-title.ts` | 设置终端标题（OSC 0）|
| `use-tab-status.ts` | 标签页状态（OSC，如 iTerm2 徽章）|

---

## 五、src/components/ — 业务 UI 组件

146 个组件，按功能分类：

### 核心交互

| 组件 | 说明 |
|---|---|
| `REPL.tsx` | **注意：实际在 `src/screens/REPL.tsx`**，主交互循环 |
| `App.tsx` | 应用根组件（AppState Provider）|
| `BaseTextInput.tsx` | 文本输入框基础组件（vim 模式支持）|
| `PromptInput.tsx` | 主提示词输入框 |

### 消息渲染

| 组件 | 说明 |
|---|---|
| `AssistantMessage.tsx` | Claude 响应渲染（文字/思考/工具）|
| `UserMessage.tsx` | 用户消息渲染 |
| `CompactSummary.tsx` | 对话压缩后的摘要展示 |
| `AgentProgressLine.tsx` | 子 Agent 执行进度（工具数、token 数）|
| `BashModeProgress.tsx` | Bash 执行进度 |

### 权限 & 确认对话框（Dialog 后缀）

| 组件 | 说明 |
|---|---|
| `BypassPermissionsModeDialog.tsx` | Auto 模式确认 |
| `AutoModeOptInDialog.tsx` | Auto 模式首次引导 |
| `ClaudeMdExternalIncludesDialog.tsx` | CLAUDE.md 外部引用确认 |
| `BridgeDialog.tsx` | IDE 桥接确认 |
| `ChannelDowngradeDialog.tsx` | 频道降级确认 |

### 配置 & 认证

| 组件 | 说明 |
|---|---|
| `ApproveApiKey.tsx` | API key 确认 |
| `ConsoleOAuthFlow.tsx` | 控制台 OAuth 流程 |
| `AwsAuthStatusBox.tsx` | AWS 认证状态 |

### 工具相关

| 组件 | 说明 |
|---|---|
| `ContextSuggestions.tsx` | 上下文补全建议（@ 文件引用）|
| `ClickableImageRef.tsx` | 可点击图片引用 |
| `ClaudeCodeHint/` | 快捷键提示组件目录 |

### 系统

| 组件 | 说明 |
|---|---|
| `AutoUpdater.tsx` | 自动更新提示 |
| `AutoUpdaterWrapper.tsx` | 更新器包装 |
| `Spinner.tsx` | 加载动画（多种 SpinnerMode）|
| `MessageSelector.tsx` | 消息过滤器（懒加载，避免 React/ink 早期 import）|

---

## 六、src/hooks/ — 业务逻辑 Hooks（87 个）

按功能分类：

### 权限管理

| Hook | 说明 |
|---|---|
| `useCanUseTool.ts` | 工具权限检查（最核心的权限 hook）|
| `useToolPermissionContext.ts` | 获取/更新权限上下文 |

### 键绑定

| Hook | 说明 |
|---|---|
| `useKeymap.ts` | 按键映射注册 |
| `useAutoMode.ts` | Auto 模式状态 |

### 状态同步

| Hook | 说明 |
|---|---|
| `useLogMessages.ts` | 消息日志（fire-and-forget transcript 写入）|
| `useSessionHistory.ts` | 历史会话 |

### 异步操作

| Hook | 说明 |
|---|---|
| `useApiStatus.ts` | API 连接状态 |
| `useIsMounted.ts` | 组件挂载状态（防止 unmount 后 setState）|

---

## 七、src/keybindings/ — 快捷键系统

独立的快捷键系统，支持用户自定义：

```
KeybindingContext.tsx      ← React Context，持有快捷键注册表
KeybindingProviderSetup.tsx ← 初始化默认 + 用户自定义绑定
defaultBindings.ts         ← 内置默认快捷键
loadUserBindings.ts        ← 从配置文件加载用户绑定
schema.ts                  ← 快捷键 schema 定义（Zod）
validate.ts                ← 验证用户绑定合法性
resolver.ts                ← 快捷键解析（冲突检测）
template.ts                ← 快捷键模板
useKeybinding.ts           ← 组件级快捷键注册 hook
reservedShortcuts.ts       ← 系统保留快捷键（不可自定义）
```

---

## 八、src/vim/ — Vim 模式

为 `BaseTextInput` 提供 Vim 键位：

```
motions.ts     ← 光标移动（w/b/e/0/$等）
operators.ts   ← 操作符（d/c/y等）
textObjects.ts ← 文本对象（iw/aw/i"/a"等）
transitions.ts ← 模式切换（Normal/Insert/Visual）
types.ts       ← Vim 模式类型
```

---

## 九、流式响应的渲染处理

Claude 的 token 流通过帧率限制转化为平滑 UI 更新：

```
API streaming → text delta 事件
    │
    ├── 更新 React state（消息数组追加字符）
    │
    └── Ink reconciler：
            ├── React commit phase（同步）
            └── throttle(render, 16ms)  ← 最多 60fps 重绘
                    → screen buffer diff
                    → writeDiffToTerminal（只写变化的单元格）
```

不是每个 token 都触发重绘——`throttle` 确保帧率上限，即使高速 streaming 也不会超过 60fps。

---

## 十、相关文件

| 文件/目录 | 说明 |
|---|---|
| `src/ink/` | 自定义终端 React Reconciler（96 文件）|
| `src/components/` | 146 个业务 UI 组件 |
| `src/hooks/` | 87 个业务逻辑 Hooks |
| `src/keybindings/` | 快捷键系统（13 文件）|
| `src/vim/` | Vim 模式实现（5 文件）|
| `src/screens/REPL.tsx` | 主交互循环（不在 components/ 中！）|
| `src/replLauncher.tsx` | 动态 import App + screens/REPL |
| `src/native-ts/yoga-layout/` | TypeScript Yoga 布局引擎 |
