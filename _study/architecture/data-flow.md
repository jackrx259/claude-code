# 数据流：一次用户输入的完整生命周期

## 主流程

```
用户在终端输入文字
        │
        ▼
REPL.tsx::processUserInput()
    ├── 解析输入（是 slash 命令？普通消息？）
    ├── 路径展开（~/path → 绝对路径）
    └── 触发 slash 命令 OR 进入 QueryEngine
        │
        ▼ (普通消息)
QueryEngine.query()
    ├── 1. getTools()           — 收集当前可用工具列表
    ├── 2. 构造 messages 数组   — 历史消息 + 新消息
    ├── 3. Claude API 调用      — 发送 messages + tool definitions
    │       │
    │       ├── 返回 text      → 直接渲染
    │       └── 返回 tool_use  → 执行工具
    │               │
    │               ▼
    │           Tool.execute()
    │               ├── BashTool   — 执行 shell 命令
    │               ├── FileReadTool — 读文件
    │               ├── AgentTool  — 启动子 Agent
    │               ├── MCPTool    — 调用 MCP server
    │               └── ...其他工具
    │               │
    │               ▼
    │           tool_result 注入回 messages
    │               │
    │               └── 循环回到第 3 步（直到 end_turn）
    │
    ▼
Session 持久化 + Analytics 上报
        │
        ▼
终端渲染（ink/React 组件树更新）
```

## Context Window 管理

QueryEngine 内部维护 token budget，当接近上限时触发**对话压缩**（compaction）：
- 调用 `src/services/compact/` 中的逻辑
- 将历史消息摘要化，保留关键上下文
- 详见 → [`deep-dives/query-engine.md`](../deep-dives/query-engine.md)

## Slash 命令流

```
用户输入 /commit
        │
        ▼
commands.ts 路由匹配
        │
        ▼
src/commands/commit/    (对应命令的实现目录)
        │
        ├── 可以直接执行逻辑（不经过 QueryEngine）
        └── 也可以构造 prompt 再走 QueryEngine
```

## Tool 权限检查流

```
Tool.execute() 调用前
        │
        ▼
permission check (src/hooks/ 中的权限 hooks)
        │
        ├── 自动允许（用户预设 / 低风险）→ 直接执行
        └── 需要确认 → 弹出权限对话框 → 用户 approve/deny
```

## 状态更新

所有消息、工具调用、任务状态变更都通过 React Context 的 `AppState` 广播，
触发对应组件重渲染。

- 状态定义：`src/state/`
- Context 定义：`src/context/`
- 全局 context 入口：`src/context.ts`