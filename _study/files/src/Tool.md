# src/Tool.ts — 工具抽象基础定义

> 文件：[`src/Tool.ts`](../../../../src/Tool.ts)（793 行）  
> 角色：**工具接口契约层**，定义了所有 49 个工具必须遵守的类型与行为规范。

## 职责总览

`Tool.ts` 不包含任何业务逻辑，它只做三件事：

1. **定义 `Tool<Input, Output, P>` 接口**：所有工具必须实现的完整契约
2. **定义 `ToolUseContext`**：工具执行时能访问的完整上下文（180+ 字段）
3. **导出 `buildTool()` 工厂函数**：用安全默认值补全工具定义中的可选字段

---

## 核心类型结构

### `Tool<Input, Output, P>` 接口（362–695 行）

工具接口是整个工具系统的核心契约。字段按功能分组：

**身份与元数据**
```typescript
name: string                    // 主名称（唯一），发给 Claude API
aliases?: string[]              // 兼容旧名称（工具重命名时保持向后兼容）
searchHint?: string             // ToolSearch 关键词匹配用（3-10词）
isMcp?: boolean                 // 是否来自 MCP server
isLsp?: boolean                 // 是否是 LSP 工具
maxResultSizeChars: number      // 结果超此大小则持久化到磁盘
strict?: boolean                // 是否启用严格模式（API 侧）
shouldDefer?: boolean           // 是否延迟加载（需 ToolSearch 先找到才能调用）
alwaysLoad?: boolean            // 是否永远不延迟（首轮就出现在 schema 中）
mcpInfo?: { serverName, toolName } // MCP 工具的原始服务器/工具名
```

**Schema 定义**
```typescript
readonly inputSchema: Input      // Zod schema → 自动转为 JSON Schema 发给 Claude
readonly inputJSONSchema?: ToolInputJSONSchema  // MCP 工具可直接提供 JSON Schema
outputSchema?: z.ZodType<unknown>
```

**核心行为方法**
```typescript
call(args, context, canUseTool, parentMessage, onProgress?)
  → Promise<ToolResult<Output>>           // 实际执行工具

description(input, options)
  → Promise<string>                       // 生成工具描述（可动态）

prompt(options)
  → Promise<string>                       // 生成工具的 system prompt 片段
```

**权限与安全**
```typescript
checkPermissions(input, context) → Promise<PermissionResult>
validateInput(input, context) → Promise<ValidationResult>
preparePermissionMatcher?(input) → Promise<(pattern: string) => boolean>
isConcurrencySafe(input): boolean         // false = 不能并发执行
isReadOnly(input): boolean                // false = 会写磁盘
isDestructive?(input): boolean            // true = 不可逆操作（删除、发送等）
```

**UI 渲染方法**（终端显示用）
```typescript
renderToolUseMessage(input, options)           // 工具调用时显示（流式，partial input）
renderToolResultMessage(content, progress, options)  // 结果显示
renderToolUseProgressMessage(progress, options)      // 执行中进度显示
renderToolUseRejectedMessage(input, options)         // 被拒绝时显示
renderToolUseErrorMessage(result, options)           // 出错时显示
renderGroupedToolUse(toolUses, options)              // 多个并发工具分组显示
renderToolUseTag?(input)                             // 工具调用后的标签（如超时时间）
renderToolUseQueuedMessage?()                        // 排队等待时显示
```

**其他辅助方法**
```typescript
userFacingName(input): string             // 用户可见的名称（默认 = name）
getPath?(input): string                   // 操作的文件路径（权限检查用）
backfillObservableInput?(input): void     // 注入遗留字段（幂等，用于 hooks/SDK）
isSearchOrReadCommand?(input): {...}      // 判断是否是搜索/读取（UI 折叠用）
interruptBehavior?(): 'cancel' | 'block'  // 用户中断时的行为
isResultTruncated?(output): boolean       // 结果是否被截断（展开提示）
extractSearchText?(out): string           // 用于转录搜索索引的文本
toAutoClassifierInput(input): unknown     // 供自动模式安全分类器使用
mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
```

---

### `ToolUseContext` 类型（158–300 行）

工具执行时传入的完整上下文，是工具访问系统能力的唯一入口：

**配置项（`options` 字段）**
```typescript
options: {
  commands: Command[]          // 当前可用的 slash commands
  tools: Tools                 // 当前所有工具
  mainLoopModel: string        // 当前主循环模型
  mcpClients: MCPServerConnection[]
  mcpResources: Record<...>
  thinkingConfig: ThinkingConfig
  verbose: boolean
  isNonInteractiveSession: boolean
  agentDefinitions: AgentDefinitionsResult
  maxBudgetUsd?: number
  customSystemPrompt?: string
  appendSystemPrompt?: string
  refreshTools?: () => Tools   // 动态刷新工具列表（MCP 热连接场景）
}
```

**状态管理**
```typescript
getAppState(): AppState
setAppState(f): void
setAppStateForTasks?: (f) => void   // 跨 Agent 层级的基础设施状态
abortController: AbortController
readFileState: FileStateCache        // LRU 文件内容缓存
```

**UI 回调**
```typescript
setToolJSX?: SetToolJSXFn            // 渲染工具自定义 UI 面板
addNotification?: (notif) => void
appendSystemMessage?: (msg) => void  // 追加 UI-only 系统消息（不发给 API）
sendOSNotification?: (opts) => void  // 系统级通知（iTerm2/Kitty 等）
setStreamMode?: (mode) => void
openMessageSelector?: () => void
```

**会话跟踪**
```typescript
messages: Message[]                    // 当前对话消息列表
agentId?: AgentId                     // 子 Agent ID（主线程无此字段）
agentType?: string                    // Agent 类型名
queryTracking?: QueryChainTracking    // 链路跟踪（chainId + depth）
discoveredSkillNames?: Set<string>    // 本轮发现的 skill 名（遥测用）
loadedNestedMemoryPaths?: Set<string> // 已注入的 CLAUDE.md 路径（去重用）
```

**权限相关**
```typescript
handleElicitation?: (serverName, params, signal) => Promise<ElicitResult>
requestPrompt?: (sourceName, summary?) => (request) => Promise<PromptResponse>
requireCanUseTool?: boolean            // 推测模式强制调用 canUseTool
localDenialTracking?: DenialTrackingState  // 异步子 Agent 的本地拒绝计数
toolDecisions?: Map<string, {...}>     // 工具决策记录
```

**文件操作限制**
```typescript
fileReadingLimits?: { maxTokens?, maxSizeBytes? }
globLimits?: { maxResults? }
contentReplacementState?: ContentReplacementState  // 工具结果 budget
renderedSystemPrompt?: SystemPrompt               // fork 子 Agent 共享 prompt cache
```

---

### `ToolResult<T>` 类型（321–336 行）

工具调用的返回值结构：

```typescript
type ToolResult<T> = {
  data: T                          // 工具输出数据
  newMessages?: Message[]          // 需要注入到对话的额外消息
  contextModifier?: (ctx) => ctx   // 修改 context（仅非并发安全工具可用）
  mcpMeta?: { _meta?, structuredContent? }  // MCP 协议元数据透传
}
```

---

## `buildTool()` 工厂函数（783–792 行）

所有工具都通过 `buildTool()` 导出，而不是直接实现 `Tool` 接口：

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, userFacingName: () => def.name, ...def } as BuiltTool<D>
}
```

**默认值（TOOL_DEFAULTS）**：
| 方法 | 默认行为 | 安全原则 |
|---|---|---|
| `isEnabled` | `() => true` | 默认启用 |
| `isConcurrencySafe` | `() => false` | fail-closed：默认不并发 |
| `isReadOnly` | `() => false` | fail-closed：默认有写操作 |
| `isDestructive` | `() => false` | 默认非破坏性 |
| `checkPermissions` | allow（defer to 通用权限系统）| 无工具特定逻辑 |
| `toAutoClassifierInput` | `() => ''` | 默认跳过分类器 |
| `userFacingName` | `() => def.name` | 名称即显示名 |

**类型魔法**：`BuiltTool<D>` 是编译期的 spread 模拟——若 `D` 有某字段则保留 `D` 的类型，否则用默认类型补全。这确保 `buildTool` 的返回类型精确反映具体工具的类型（不丢失字面量类型）。

---

## 辅助函数

```typescript
toolMatchesName(tool, name): boolean    // 匹配主名称或别名
findToolByName(tools, name): Tool | undefined
filterToolProgressMessages(msgs): ProgressMessage<ToolProgressData>[]
getEmptyToolPermissionContext(): ToolPermissionContext  // 空权限上下文（测试/初始化用）
```

---

## 关键设计理念

1. **纯接口，无继承**：`Tool` 是 TypeScript 类型接口，工具用对象字面量 + `buildTool()` 实现，不是 class 继承。这让工具定义极为简洁。

2. **UI 与逻辑同处一文件**：每个工具目录同时包含 `call()` 逻辑和 `renderToolResultMessage()` UI 渲染——工具是"自渲染"的。

3. **延迟加载（Deferred Tools）**：`shouldDefer: true` 的工具不在首轮 API 调用中发送 schema，必须先通过 `ToolSearch` 找到才能调用。这优化了 token 消耗（schema 很大）。

4. **权限层次**：`checkPermissions()` → `validateInput()` → `canUseTool()` 三层过滤，工具特定逻辑在前两层，通用规则引擎在第三层。

---

## 与其他模块的关系

```
Tool.ts（接口定义）
    ← tools/*/index.ts（具体实现，用 buildTool() 包装）
    ← tools.ts（聚合导出 getTools()）
    ← QueryEngine.ts（执行 tool.call()）
    ← query.ts（权限检查、并发控制）
    ← components/（用 render*() 方法渲染 UI）
```