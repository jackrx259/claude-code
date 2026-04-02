# 数据流：一次用户输入的完整生命周期

> 对应源码版本：v2.1.88

## 主流程总览

```
用户在终端输入文字
        │
        ▼
src/screens/REPL.tsx          ← 主交互循环（注意：在 screens/ 而非 components/）
    ├── 识别 slash 命令（/ 前缀）→ 走命令流（见下）
    └── 普通消息 → 触发 QueryEngine
                │
                ▼
QueryEngine.submitMessage(prompt)   ← AsyncGenerator，yield SDKMessage
    ├── 1. 组装系统提示词（fetchSystemPromptParts）
    ├── 2. processUserInput()   ← 处理用户输入（路径展开、附件解析等）
    ├── 3. 调用 query()         ← 底层 API 循环（src/query.ts）
    │       │
    │       └── 多轮循环（直到 end_turn）：
    │               ├── 发 Claude API 请求
    │               ├── 流式接收响应（yield StreamEvent）
    │               ├── 若返回 tool_use → 执行工具 → 注入 tool_result
    │               ├── 若 context 接近上限 → 触发 compact
    │               └── 若 end_turn → 退出循环
    │
    ├── 4. 持久化：recordTranscript() / flushSessionStorage()
    └── 5. yield 最终结果给 REPL 渲染
```

## 启动时的渲染树

```
src/main.tsx
    └── launchRepl(root, appProps, replProps, renderAndRun)
            │  (src/replLauncher.tsx)
            ├── 动态 import('./components/App.js')
            ├── 动态 import('./screens/REPL.js')
            └── renderAndRun(root,
                  <App {...appProps}>
                    <REPL {...replProps} />
                  </App>
                )
```

> `replLauncher.tsx` 用动态 import 延迟加载 App 和 REPL，避免在快速路径（--version 等）时加载 React。

## QueryEngine.submitMessage() 详细步骤

```typescript
// 签名
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean }
): AsyncGenerator<SDKMessage, void, unknown>
```

**每次调用的执行过程：**

```
1. this.discoveredSkillNames.clear()   ← 清除上轮技能发现记录

2. 组装系统提示词
   fetchSystemPromptParts({ tools, mainLoopModel, mcpClients, ... })
   → defaultSystemPrompt（默认）或 customSystemPrompt（SDK 调用时）
   → + memoryMechanicsPrompt（若设置了 CLAUDE_COWORK_MEMORY_PATH_OVERRIDE）
   → + appendSystemPrompt（可选追加）

3. 构建 processUserInputContext（包含 messages、工具列表、权限回调等）

4. processUserInput()
   → 解析 slash 命令（若有）
   → 路径展开、@文件引用解析
   → 构造 UserMessage 追加到 this.mutableMessages

5. query(context)   ← 进入 src/query.ts 的核心循环
   （见下方 query.ts 说明）

6. 记录 transcript、更新 session 存储
7. yield 各类 SDKMessage 给调用者
```

## query.ts 核心循环

`src/query.ts` 是真正执行 API 调用的地方，功能包括：

```
while (true):
    normalizeMessagesForAPI(messages)   ← 去除 UI-only 消息，适配 API 格式
    Claude API 请求（流式）
        │
        ├── text delta        → yield StreamEvent（逐字渲染）
        ├── thinking block    → yield StreamEvent（可折叠的思考过程）
        └── tool_use          → 执行工具
                │
                ├── canUseTool()    ← 权限检查（自动允许/弹窗确认/拒绝）
                ├── tool.execute()  ← 实际执行（BashTool/FileEditTool/AgentTool...）
                └── 注入 tool_result 到 messages，继续循环

    stop_reason === 'end_turn'  → break

    Token 监控：
    ├── calculateTokenWarningState()  → 接近上限时给 UI 警告
    └── isAutoCompactEnabled() + 超阈值 → 触发 compact（buildPostCompactMessages）
```

**compaction（对话压缩）触发条件：**
- `isAutoCompactEnabled()` 为 true（用户未禁用）
- token 使用量超过阈值（`calculateTokenWarningState` 判断）
- 调用 `src/services/compact/compact.js` 的 `buildPostCompactMessages()`
- 用摘要替换历史，释放 context window 空间

## Slash 命令流

```
用户输入 "/commit"
    │
    ▼
REPL.tsx 检测到 "/" 前缀
    │
    ▼
src/commands.ts 路由匹配
    └── getCommands() 中注册的命令列表
            │
            ▼
        src/commands/commit/   ← 对应命令实现目录
            ├── 直接执行逻辑（git 操作等）
            └── 或构造 prompt 再走 QueryEngine
```

## 工具权限检查流

```
query.ts 准备执行工具时：
    │
    ▼
QueryEngine.wrappedCanUseTool()
    │  （对 config.canUseTool 的包装，额外追踪 permissionDenials）
    ▼
canUseTool(tool, input, toolUseContext, ...)
    │
    ├── 检查 alwaysAllowRules → 直接 allow
    ├── 检查 alwaysDenyRules  → 直接 deny
    ├── permissionMode === 'bypassPermissions' → allow（Auto 模式）
    └── 弹出权限确认 UI（REPL 模式）或自动拒绝（headless 模式）

结果：
    ├── allow    → tool.execute()
    └── deny/reject → 向 messages 注入拒绝提示，记录到 permissionDenials
```

## 状态更新路径

```
工具执行 / API 响应
    │
    ├── setAppState(fn)        ← 写入 React Context AppState
    │       │
    │       └── 触发订阅该状态的组件重渲染（ink/React 协调）
    │
    └── setAppStateForTasks(fn) ← 仅用于任务/后台 Agent，
                                   直达根 store（子 Agent 的 setAppState 是 no-op）
```

## 关键路径上的文件

| 步骤 | 文件 |
|---|---|
| 用户输入 → REPL | `src/screens/REPL.tsx` |
| REPL 启动 | `src/replLauncher.tsx` |
| QueryEngine 编排 | `src/QueryEngine.ts` |
| 底层 API 循环 | `src/query.ts` |
| 用户输入预处理 | `src/utils/processUserInput/processUserInput.ts` |
| 系统提示词构建 | `src/utils/queryContext.ts`（fetchSystemPromptParts）|
| 消息规范化 | `src/utils/messages.ts`（normalizeMessagesForAPI）|
| compaction | `src/services/compact/compact.ts`、`autoCompact.ts` |
| 权限检查 | `src/hooks/useCanUseTool.ts` |
| Session 持久化 | `src/utils/sessionStorage.ts` |
