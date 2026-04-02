# QueryEngine — 核心编排引擎深度分析

> 文件：`src/QueryEngine.ts`（1,295 行）+ `src/query.ts`（500+ 行）
> 版本：v2.1.88

---

## 一、职责定位

`QueryEngine` 是 Claude Code 的"大脑调度中心"，处于架构的核心编排层。  
一个 QueryEngine 实例 = 一段对话，`submitMessage()` 在同一实例上多次调用维持状态。

主要职责：

| 职责 | 说明 |
|---|---|
| 多轮 API 调用循环 | 发请求 → 处理工具调用 → 注入结果 → 再发请求，直到 `end_turn` |
| 消息历史管理 | 维护完整的 `mutableMessages: Message[]` |
| Context Window 管理 | 监控 token 用量，触发对话压缩（compaction） |
| Token Budget 控制 | 限制单次任务的 USD 预算和最大轮次 |
| Thinking Mode | 支持 Claude 的 extended thinking 模式（adaptive 或 disabled）|
| 权限追踪 | 包装 `canUseTool`，收集所有权限拒绝记录到 `permissionDenials` |
| 结构化输出 | 支持 JSON Schema 约束的结构化输出，最多重试 5 次 |
| Session 持久化 | 每轮调用前后异步写入 transcript |

---

## 二、类结构

```typescript
export class QueryEngine {
  // 构造时注入的配置（不可变）
  private config: QueryEngineConfig

  // 跨轮次持久状态
  private mutableMessages: Message[]          // 完整对话历史
  private abortController: AbortController   // 中断控制（可外部注入）
  private permissionDenials: SDKPermissionDenial[]  // 累积的拒绝记录
  private totalUsage: NonNullableUsage        // 累计 token 用量（message_stop 时累加）
  private hasHandledOrphanedPermission = false  // 孤立权限只处理一次
  private readFileState: FileStateCache       // 文件读取 LRU 缓存
  private discoveredSkillNames = new Set<string>()  // 每轮开始时清空
  private loadedNestedMemoryPaths = new Set<string>()  // 跨轮去重

  // 方法
  constructor(config: QueryEngineConfig)
  async *submitMessage(prompt, options?): AsyncGenerator<SDKMessage>
  interrupt(): void                           // 触发 AbortController.abort()
  getMessages(): readonly Message[]
  getReadFileState(): FileStateCache
  getSessionId(): string
  setModel(model: string): void               // 运行时切换模型
}
```

### QueryEngineConfig 关键字段

```typescript
type QueryEngineConfig = {
  cwd: string                    // 工作目录
  tools: Tools                   // 工具列表
  commands: Command[]            // slash 命令列表
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn       // 权限检查回调（注入自外部）
  getAppState: () => AppState
  setAppState: (f) => void       // React Context 状态写入
  initialMessages?: Message[]    // 可传入历史消息（--resume）
  readFileCache: FileStateCache
  customSystemPrompt?: string    // SDK 调用者可自定义
  appendSystemPrompt?: string    // 追加到系统提示词末尾
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig  // { type: 'adaptive' | 'disabled' | 'enabled' }
  maxTurns?: number
  maxBudgetUsd?: number          // USD 预算上限
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>  // 结构化输出 schema
  replayUserMessages?: boolean
  includePartialMessages?: boolean  // 是否 yield stream_event 给调用者
  snipReplay?: (msg, store) => ...  // HISTORY_SNIP flag 注入的回调
}
```

---

## 三、submitMessage() 完整执行流程

```
submitMessage(prompt, options?)
│
├── 0. 每轮初始化
│   ├── discoveredSkillNames.clear()          ← 清空上轮 skill 发现记录
│   ├── setCwd(cwd)
│   └── persistSession = !isSessionPersistenceDisabled()
│
├── 1. 包装 canUseTool（权限拒绝追踪）
│   └── wrappedCanUseTool: 调用原 canUseTool，若结果 !== 'allow' 则追加到 permissionDenials
│
├── 2. 系统提示词构建
│   ├── fetchSystemPromptParts({ tools, mainLoopModel, mcpClients, ... })
│   │   → defaultSystemPrompt 或 customSystemPrompt
│   ├── + memoryMechanicsPrompt（若设了 CLAUDE_COWORK_MEMORY_PATH_OVERRIDE）
│   ├── + appendSystemPrompt
│   └── asSystemPrompt([...])  ← 组合成最终 SystemPrompt
│
├── 3. 构建 processUserInputContext（包含 messages、工具列表、各类回调）
│   ├── 第一次构建：setMessages 会写回 mutableMessages（slash 命令可能修改消息）
│   └── 第二次构建（处理 prompt 之后）：setMessages 为 no-op
│
├── 4. 处理孤立权限（orphanedPermission，每生命周期仅一次）
│
├── 5. processUserInput()
│   ├── 解析 slash 命令（/ 前缀）
│   ├── 路径展开、@文件引用解析
│   ├── 返回 { messages, shouldQuery, allowedTools, model, resultText }
│   └── 将新消息 push 到 mutableMessages
│
├── 6. 持久化用户消息到 transcript（在进入 API 循环前）
│   ├── 非 bare 模式：await recordTranscript()
│   └── bare 模式：fire-and-forget（不阻塞）
│
├── 7. 并行加载 skills 和 plugins
│   ├── getSlashCommandToolSkills(cwd)
│   └── loadAllPluginsCacheOnly()
│
├── 8. yield buildSystemInitMessage(...)    ← SDK 调用者的初始化元信息
│
├── 9. shouldQuery 分支
│   ├── false（slash 命令本地执行）→ yield 命令结果 → yield result → return
│   └── true → 进入 query() 循环（见下）
│
├── 10. for await (const message of query(...))
│   │   ← 迭代 query.ts 的 AsyncGenerator
│   │
│   ├── 消息类型处理：
│   │   ├── 'assistant'      → push mutableMessages, yield
│   │   ├── 'user'           → push mutableMessages, turnCount++, yield
│   │   ├── 'progress'       → push + 持久化 + yield
│   │   ├── 'attachment'
│   │   │   ├── 'structured_output' → 保存到 structuredOutputFromTool
│   │   │   ├── 'max_turns_reached' → yield error_max_turns → return
│   │   │   └── 'queued_command'    → yield user replay（若 replayUserMessages）
│   │   ├── 'stream_event'
│   │   │   ├── message_start  → 重置 currentMessageUsage
│   │   │   ├── message_delta  → 更新 usage, 捕获 stop_reason
│   │   │   └── message_stop   → accumulateUsage 到 totalUsage
│   │   ├── 'system'
│   │   │   ├── compact_boundary → 清理前序消息（GC），yield 给 SDK
│   │   │   └── api_error        → yield api_retry 给 SDK
│   │   └── 'tool_use_summary' → yield 给 SDK
│   │
│   └── 每条消息后检查预算：
│       ├── maxBudgetUsd 超限 → yield error_max_budget_usd → return
│       └── jsonSchema 结构化输出重试超限 → yield error_max_structured_output_retries → return
│
└── 11. 最终结果
    ├── isResultSuccessful() 检查 → 否则 yield error_during_execution
    └── yield result (subtype: 'success', 含 textResult, usage, cost, permissionDenials 等)
```

---

## 四、query.ts — 底层 API 循环

`src/query.ts` 是真正执行 Claude API 调用的地方，被 `submitMessage` 通过 `for await` 消费。

### 入口签名

```typescript
export async function* query(params: QueryParams): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal
>
```

`query()` 本身只是 `queryLoop()` 的包装器，负责在正常退出时通知命令生命周期（`notifyCommandLifecycle`）。

### queryLoop() 关键结构

```
queryLoop():
│
├── 初始化一次性状态
│   ├── budgetTracker（TOKEN_BUDGET flag 启用时）
│   ├── buildQueryConfig()（快照 env/statsig/session 状态）
│   └── startRelevantMemoryPrefetch()（using 关键字，自动析构）
│
└── while (true):
    ├── 取出最新 messages（getMessagesAfterCompactBoundary）
    ├── applyToolResultBudget()  ← 限制单条消息的 tool result 大小
    ├── HISTORY_SNIP（若启用）  ← 裁剪历史片段
    ├── microcompact（缓存微压缩）
    ├── autocompact 检查（calculateTokenWarningState + 阈值判断）
    │   └── 若超限 → buildPostCompactMessages() → 替换历史为摘要
    ├── 发 Claude API 请求（流式）
    │   ├── text delta        → yield StreamEvent
    │   ├── thinking block    → yield StreamEvent
    │   └── tool_use block    → 执行工具
    │           ├── canUseTool() 权限检查
    │           ├── tool.execute()
    │           └── 注入 tool_result 到 messages
    ├── executePostSamplingHooks()（stop hooks）
    └── stop_reason === 'end_turn' → return Terminal
```

### 工具执行并发机制

工具执行由 `src/services/tools/toolOrchestration.ts` 的 `runTools()` 负责：
- **非并行工具**：串行逐一执行
- **可并行工具**：通过 `StreamingToolExecutor` 并发执行
- 每个工具有 `maxResultSizeChars` 限制（超限会被 `applyToolResultBudget` 截断）

---

## 五、Token 与 Budget 管理

### token 监控流

```
每次 API 响应后：
  calculateTokenWarningState(usage) 
    → 'ok' / 'warning' / 'danger' / 'critical'
  
  isAutoCompactEnabled() && 超阈值
    → buildPostCompactMessages()  ← 调用 Claude 生成摘要
    → 替换 messages 为 [compact_boundary + summary + 尾部保留]
```

### USD 预算

- `maxBudgetUsd`（来自 `--max-budget` 或 SDK 配置）
- 每条消息后检查 `getTotalCost() >= maxBudgetUsd`
- 超限立即 yield `error_max_budget_usd` 并 return

### maxTurns

- 默认无限制，可通过 `maxTurns` 设置
- query.ts 内部的 `queryLoop` 跟踪 `turnCount`，超限 yield attachment `max_turns_reached`

---

## 六、SDKMessage 类型体系

`submitMessage()` 是 `AsyncGenerator<SDKMessage>`，yield 的所有消息类型：

| 类型 | subtype | 说明 |
|---|---|---|
| `system` | — | 初始化元信息（tools、model、permissions 等）|
| `assistant` | — | Claude 的文字/思考/工具调用响应 |
| `user` | — | 用户消息回放（`isReplay: true`）|
| `user` | — | tool_result 注入 |
| `stream_event` | — | 原始流事件（includePartialMessages=true 时）|
| `tool_use_summary` | — | 工具调用摘要 |
| `system` | `compact_boundary` | 对话压缩边界 |
| `system` | `api_retry` | API 重试通知 |
| `result` | `success` | 正常结束，含 textResult、usage、cost 等 |
| `result` | `error_max_turns` | 超过最大轮次 |
| `result` | `error_max_budget_usd` | 超过 USD 预算 |
| `result` | `error_during_execution` | 执行中出错，含 errors 诊断数组 |
| `result` | `error_max_structured_output_retries` | 结构化输出重试超限 |

---

## 七、ask() — 便捷包装器

`ask()` 是对 `QueryEngine` 的一次性封装（One-shot），供不需要多轮交互的场景使用：

```typescript
export async function* ask(params): AsyncGenerator<SDKMessage> {
  const engine = new QueryEngine({ ...params })
  try {
    yield* engine.submitMessage(prompt, { uuid: promptUuid, isMeta })
  } finally {
    setReadFileCache(engine.getReadFileState())  // 写回文件缓存
  }
}
```

`HISTORY_SNIP` flag 启用时，`ask()` 负责注入 `snipReplay` 回调到 `QueryEngineConfig`，使历史片段裁剪功能对 QueryEngine 透明（feature-gated 字符串不出现在 QueryEngine.ts 中）。

---

## 八、设计亮点

### 1. 状态跨轮次持久
`mutableMessages`、`permissionDenials`、`totalUsage`、`readFileState`、`loadedNestedMemoryPaths` 全部跨 `submitMessage()` 调用保持，支持多轮对话。

### 2. 孤立权限处理（orphanedPermission）
当上一轮的工具调用权限请求在新轮次才被处理（通常发生在 IDE 集成中），通过 `hasHandledOrphanedPermission` 标志保证每个 engine 生命周期只处理一次。

### 3. Transcript 预写
用户消息在进入 API 循环前就写入 transcript，即使进程在 API 响应前被杀死，`--resume` 也能找到消息并继续。

### 4. 内存管理
compact_boundary 触发后立即 splice 掉前序消息（`this.mutableMessages.splice(0, mutableBoundaryIdx)`），允许 GC 回收，防止长会话内存泄漏。

### 5. errorLogWatermark（ede 诊断）
`error_during_execution` 的 `errors[]` 是轮次范围的（通过 watermark 截取），避免把整个进程历史的错误日志全部包含进去。

---

## 九、相关文件

| 文件 | 说明 |
|---|---|
| `src/QueryEngine.ts` | 本文分析的主文件（1,295 行）|
| `src/query.ts` | 底层 API 调用循环（`queryLoop`）|
| `src/services/tools/toolOrchestration.ts` | `runTools()` 工具并发执行 |
| `src/services/compact/autoCompact.ts` | `calculateTokenWarningState`, `isAutoCompactEnabled` |
| `src/services/compact/compact.ts` | `buildPostCompactMessages()` 摘要生成 |
| `src/utils/processUserInput/processUserInput.ts` | 用户输入预处理 |
| `src/utils/queryContext.ts` | `fetchSystemPromptParts()` 系统提示词构建 |
| `src/utils/sessionStorage.ts` | `recordTranscript`, `flushSessionStorage` |
| `src/entrypoints/agentSdkTypes.ts` | `SDKMessage` 完整类型定义 |
| `src/cost-tracker.ts` | `getTotalCost`, `getTotalAPIDuration` |
