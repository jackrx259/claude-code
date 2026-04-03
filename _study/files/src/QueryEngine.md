# src/QueryEngine.ts — 多轮 API 编排引擎

> 文件：[`src/QueryEngine.ts`](../../../../src/QueryEngine.ts)（1295 行）  
> 角色：**SDK/无头模式的对话生命周期管理器**，将核心 `query()` 循环封装为可跨轮次保持状态的类。

## 职责总览

`QueryEngine` 是 SDK 调用路径（非交互模式）的核心类。它：

1. **管理跨轮次状态**：消息历史、文件缓存、token 用量、权限拒绝记录
2. **协调单轮 `submitMessage()`**：从用户输入到最终 `result` 消息的完整流程
3. **对外暴露 SDK 消息流**：yield 标准化的 `SDKMessage` 供调用者消费
4. **提供便捷包装**：`ask()` 函数，一次性使用 QueryEngine 的单轮调用

> 注：交互式 REPL 不走 QueryEngine，直接使用 `query.ts` 的 `query()` 函数。

---

## 类结构

```typescript
export class QueryEngine {
  private config: QueryEngineConfig     // 构造时传入，跨轮次不变
  private mutableMessages: Message[]    // 累积的对话历史（含助手、工具结果等）
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]  // 本次会话的拒绝记录
  private totalUsage: NonNullableUsage  // 累积 token 用量
  private readFileState: FileStateCache // LRU 文件读取缓存
  private discoveredSkillNames: Set<string>       // 本轮发现的 skill（每轮 clear）
  private loadedNestedMemoryPaths: Set<string>    // 已注入的 CLAUDE.md 路径（跨轮持久）
  
  constructor(config: QueryEngineConfig)
  async *submitMessage(prompt, options?): AsyncGenerator<SDKMessage>
  interrupt(): void
  getMessages(): readonly Message[]
  getReadFileState(): FileStateCache
  getSessionId(): string
  setModel(model: string): void
}
```

---

## `QueryEngineConfig` 配置项（130–173 行）

| 字段 | 说明 |
|---|---|
| `cwd` | 工作目录 |
| `tools` | 工具列表 |
| `commands` | Slash commands |
| `mcpClients` | MCP 服务器连接 |
| `agents` | Agent 定义列表 |
| `canUseTool` | 权限检查回调 |
| `getAppState / setAppState` | 应用状态访问器 |
| `initialMessages?` | 恢复会话时的历史消息 |
| `readFileCache` | 文件状态缓存（跨轮共享）|
| `customSystemPrompt?` | 替换默认系统提示 |
| `appendSystemPrompt?` | 追加到系统提示末尾 |
| `userSpecifiedModel?` | 用户指定的模型 |
| `maxTurns?` | 最大轮数限制 |
| `maxBudgetUsd?` | 最大费用限制（USD）|
| `taskBudget?` | 任务 token 预算 |
| `jsonSchema?` | 结构化输出 JSON Schema |
| `replayUserMessages?` | 是否向 SDK 重放用户消息 |
| `includePartialMessages?` | 是否 yield 流式 stream_event |
| `snipReplay?` | HISTORY_SNIP 功能回调（SDK-only 内存管理）|

---

## `submitMessage()` 完整流程（209–1156 行）

这是 QueryEngine 的核心方法，是一个 AsyncGenerator。完整流程分为 7 个阶段：

### 阶段 1：初始化（209–333 行）

```
权限拒绝追踪包装 wrappedCanUseTool
    ↓
获取模型（用户指定 or 默认主循环模型）
    ↓
fetchSystemPromptParts()  → defaultSystemPrompt + userContext + systemContext
    ↓
注入 COORDINATOR 上下文（COORDINATOR_MODE feature 启用时）
    ↓
注入内存机制提示（CLAUDE_COWORK_MEMORY_PATH_OVERRIDE 环境变量时）
    ↓
asSystemPrompt() 组装最终 system prompt
    ↓
注册结构化输出强制 hook（如果有 jsonSchema）
```

### 阶段 2：处理用户输入（335–486 行）

```
构建 processUserInputContext（第一版，含 setMessages 回调）
    ↓
handleOrphanedPermission()（孤儿权限处理，仅首次）
    ↓
processUserInput()  → 解析 slash commands、展开路径、处理附件
    ↓
mutableMessages.push(...messagesFromUserInput)
    ↓
recordTranscript()  → 写入会话持久化（用户消息先写，防止进程被 kill 后无法 resume）
    ↓
更新 toolPermissionContext（alwaysAllowRules 来自用户 /allow 命令）
```

### 阶段 3：加载 Skills/Plugins（529–551 行）

```
parallel:
  getSlashCommandToolSkills(cwd)    // 加载技能
  loadAllPluginsCacheOnly()         // 加载插件（仅缓存，不发网络）

yield buildSystemInitMessage(...)   // ★ 第一个 SDK 消息：系统初始化信息
```

### 阶段 4：短路路径（556–638 行）

若 `shouldQuery = false`（slash command 本地处理完毕，无需调 API）：

```
yield 本地命令输出（作为 SDKUserMessageReplay 或 SDKAssistantMessage）
yield compact_boundary（如有）
yield result{subtype: 'success'}
return  ← 提前结束
```

### 阶段 5：主 query 循环（675–1048 行）

```typescript
for await (const message of query({
  messages, systemPrompt, userContext, systemContext,
  canUseTool: wrappedCanUseTool,
  toolUseContext: processUserInputContext,
  fallbackModel, querySource: 'sdk', maxTurns, taskBudget,
})) {
  // 按 message.type 分发处理：
}
```

各消息类型处理：

| `message.type` | 处理逻辑 |
|---|---|
| `assistant` | push 到 mutableMessages，记录 stop_reason，yield* normalizeMessage() |
| `user` | push 到 mutableMessages，turnCount++，yield* normalizeMessage() |
| `progress` | push，recordTranscript（inline），yield* normalizeMessage() |
| `stream_event` | 累积 token 用量（message_start/delta/stop），按需 yield（includePartialMessages）|
| `attachment` | 处理 structured_output / max_turns_reached / queued_command |
| `system` | 处理 compact_boundary（GC 历史消息）/ api_error（yield api_retry）/ snip |
| `tool_use_summary` | yield tool_use_summary SDK 消息 |
| `tombstone` | 忽略（控制信号）|

**中止条件检查**（每个消息处理后）：
- `maxBudgetUsd` 超限 → yield `error_max_budget_usd`，return
- 结构化输出重试次数超限 → yield `error_max_structured_output_retries`，return

### 阶段 6：结果提取（1058–1155 行）

```
找最后一条 assistant 或 user 消息
    ↓
isResultSuccessful() 检查：
  - assistant 消息末尾是 text/thinking block
  - 或 user 消息全是 tool_result block
  - 且 stop_reason 是 end_turn（或允许的其他值）
    ↓
成功 → yield result{subtype: 'success', result: textResult}
失败 → yield result{subtype: 'error_during_execution', errors: [...]}
```

### 阶段 7：Compact（GC）处理

在 compact_boundary 消息时：
```
记录 preservedSegment tail 到 transcript（防 resume 失败）
    ↓
splice mutableMessages（释放 boundary 前消息，GC）
    ↓
yield compact_boundary SDK 消息
```

---

## SDK 消息类型对照

| 内部消息 | SDK 消息 | 触发时机 |
|---|---|---|
| — | `system_init` | submitMessage 开始时 |
| `assistant` | `assistant` | Claude 返回文本 |
| `user`（含 tool_result）| `user` | 工具结果注入 |
| `progress` | `tool_result_delta` 等 | 工具执行中 |
| `stream_event` | `stream_event` | 仅 includePartialMessages |
| `system.compact_boundary` | `system.compact_boundary` | 对话压缩 |
| `system.api_error` | `system.api_retry` | API 重试通知 |
| `tool_use_summary` | `tool_use_summary` | 并发工具汇总 |
| — | `result.success` | 正常结束 |
| — | `result.error_max_turns` | 超最大轮数 |
| — | `result.error_max_budget_usd` | 超费用上限 |
| — | `result.error_during_execution` | 异常终止 |

---

## `ask()` 便捷函数（1186–1295 行）

```typescript
export async function* ask({prompt, cwd, tools, ...}) {
  const engine = new QueryEngine({...})
  yield* engine.submitMessage(prompt)
}
```

`ask()` 是对 QueryEngine 的一次性封装。适合单轮调用场景（如 `-p` 模式），多轮对话（如 SDK 会话管理）应直接使用 `QueryEngine` 类。

---

## 与 REPL 模式的对比

| 维度 | QueryEngine（SDK 模式）| REPL 模式（交互）|
|---|---|---|
| 入口 | `ask()` / `QueryEngine.submitMessage()` | `query()` in `query.ts` |
| 消息历史 | 自管理（mutableMessages）| AppState（React 状态）|
| 权限 UI | 无（programmatic deny）| 弹出交互对话框 |
| Compact 后 | GC 历史消息（节省内存）| 保留（UI 可滚动回看）|
| 会话持久化 | recordTranscript（含 flush）| useLogMessages.ts fire-and-forget |
| 中止 | `engine.interrupt()` | SIGINT / Stop 按钮 |

---

## 关键设计决策

1. **用户消息先写 transcript**（449 行注释）：在进入 API 循环前就写入，确保进程被 kill 后仍可 resume。代价是 ~4ms I/O 延迟（bare 模式下 fire-and-forget）。

2. **processUserInputContext 构建两次**（335, 492 行）：第一次含 setMessages（slash command 可能修改历史），第二次 setMessages 置 no-op（之后不再需要修改）。

3. **`wrappedCanUseTool`（244 行）**：包装原始 canUseTool 只是为了记录 denials，不改变权限决策。最终 `permission_denials` 附在 result 消息中给 SDK 消费者。

4. **`snipReplay` 注入（168 行）**：HISTORY_SNIP 功能的 feature-gated 字符串保留在 snipCompact.js 中，不污染 QueryEngine 源码（让 excluded-strings 检查通过）。