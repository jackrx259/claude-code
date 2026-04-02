# Tool System — 工具体系

> 核心文件：`src/Tool.ts`，实现目录：`src/tools/`

## Tool 基础结构

每个工具本质上是一个对象，包含：

```typescript
interface Tool {
  name: string                    // 工具名称（发给 Claude 的 tool name）
  description: string             // 工具描述（Claude 根据此决定是否调用）
  inputSchema: ZodSchema          // 输入参数 schema（转成 JSON Schema 发给 API）
  outputSchema?: ZodSchema        // 输出 schema（可选）
  
  execute(input, context): Promise<ToolResult>  // 实际执行逻辑
  
  // 权限相关
  needsPermissions?: boolean
  isReadOnly?: boolean
}
```

## 工具注册 & 加载流程

```
build.ts 编译期
    → feature flags 决定哪些工具被编译进 bundle

src/main.tsx 启动期
    → 初始化工具列表（读取 src/tools.ts 导出的工具集合）
    → 注册到全局状态

QueryEngine.getTools()
    → 每次 API 调用前动态决定当前可用工具
    → （某些工具在特定模式下不可用，如 Plan Mode 限制写操作）
```

## 权限模型

工具执行前经过权限检查：

| 权限级别 | 行为 |
|---|---|
| 自动允许 | 只读操作（FileRead、Glob、Grep 等）|
| 需要确认 | 写操作（FileEdit、FileWrite）、Bash 命令 |
| 永久允许 | 用户可设置"总是允许"某工具 |
| 拒绝 | 用户选择 deny 后，当前会话不再询问 |

权限状态存储在 `src/state/` 中，通过 `src/hooks/` 中的权限 hooks 读取。

## Plan Mode 对工具的影响

进入 Plan Mode（`EnterPlanModeTool`）后：
- 写类工具（FileEdit、FileWrite、Bash 写操作）被屏蔽
- 只允许只读工具运行
- 设计目的：让 Claude "只规划不执行"

## MCP 工具 vs 内置工具

| 类型 | 来源 | 注册方式 |
|---|---|---|
| 内置工具 | `src/tools/` 目录 | 编译时确定 |
| MCP 工具 | 外部 MCP server | 运行时动态注册（`src/services/mcp/`）|

MCP 工具通过 `MCPTool` 适配器统一接入，对 QueryEngine 来说和内置工具没有区别。

## 关键问题待研究

- [ ] `src/Tool.ts` 中的基类/接口具体定义
- [ ] 工具输出如何被截断（超长输出处理）
- [ ] AgentTool 如何创建隔离的子 QueryEngine 实例
- [ ] BashTool 的沙箱/超时机制

## 相关文件

- 基础类型：[`src/Tool.ts`](../../src/Tool.ts)
- 工具聚合：[`src/tools.ts`](../../src/tools.ts)
- 工具目录：[`src/tools/`](../../src/tools/)（各子目录）
- 权限 hooks：[`src/hooks/`](../../src/hooks/)