# Terminal UI — 终端渲染体系

> 核心目录：`src/ink/`（96 文件）、`src/components/`（146 组件）、`src/hooks/`（87 hooks）

## 技术选型

Claude Code 使用 **React** 渲染终端 UI，基于 **Ink**（一个将 React 渲染到终端的库）。

但重要的是：`src/ink/` 是 **Ink 的深度修改版（fork）**，而不是直接用 npm 包。
这说明 Anthropic 对终端渲染层有大量定制需求，超出了原版 Ink 的能力范围。

## 渲染原理

```
React 组件树
    │
    ▼
自定义 Ink Reconciler（src/ink/）
    │  将 React 虚拟 DOM 转换为终端控制序列
    ▼
终端 (ANSI escape codes)
    │
    ▼
用户看到的 TUI 界面
```

## 组件架构（src/components/）

146 个组件，关键组件：

| 组件 | 职责 |
|---|---|
| `REPL.tsx` | 主交互循环，最核心的组件 |
| `App.tsx` | 应用根组件 |
| `AgentProgressLine.tsx` | Agent 执行进度展示 |
| `CompactSummary.tsx` | 对话压缩后的摘要展示 |
| `BaseTextInput.tsx` | 文本输入框基础组件 |
| `ContextSuggestions.tsx` | 上下文补全建议 |
| `BashModeProgress.tsx` | Bash 执行进度 |
| `AutoUpdater.tsx` | 自动更新 UI |

命名模式中的 `Dialog` 后缀表示弹出对话框类组件（权限确认、配置等）。

## Hooks 体系（src/hooks/）

87 个 hooks，按功能分类：

| 类型 | 示例 |
|---|---|
| 权限管理 | 工具调用权限确认 hooks |
| 键绑定 | 快捷键注册 & 响应 |
| 异步操作 | 包装 API 调用的 loading/error 状态 |
| 状态同步 | 订阅全局 AppState 变更 |

## 关键设计：流式渲染

Claude 的响应是流式（streaming）的，UI 必须处理增量更新：
- text delta → 逐字追加
- tool_use 开始 → 显示工具调用进度
- tool_result → 折叠/展示结果
- thinking block → 可折叠的思考过程面板

## 关键问题待研究

- [ ] `src/ink/` 相比原版 Ink 做了哪些修改？（性能优化？新原语？）
- [ ] 流式输出如何触发 React re-render（不能每个字都 setState）
- [ ] 键绑定系统的实现（`src/keybindings/`）
- [ ] vim 模式的实现（`src/vim/`）

## 相关文件

- 自定义 Ink：[`src/ink/`](../../src/ink/)
- Ink 入口：[`src/ink.ts`](../../src/ink.ts)
- 所有 UI 组件：[`src/components/`](../../src/components/)
- 所有 hooks：[`src/hooks/`](../../src/hooks/)
- 键绑定：[`src/keybindings/`](../../src/keybindings/)
- Vim 模式：[`src/vim/`](../../src/vim/)
