# MCP System — Model Context Protocol 体系深度分析

> 核心目录：`src/services/mcp/`（22 文件）
> 版本：v2.1.88

---

## 一、什么是 MCP

MCP（Model Context Protocol）是 Anthropic 开发的开放协议，允许外部服务通过标准接口向 Claude 暴露工具、资源和提示词模板。

Claude Code 作为 MCP **客户端**，可以连接任意 MCP server。

---

## 二、整体架构

```
MCP Server（外部进程/服务）
    │  JSON-RPC over stdio/SSE/HTTP/WebSocket
    ▼
src/services/mcp/client.ts  ← MCP 客户端（@modelcontextprotocol/sdk Client）
    │  连接管理、工具/资源/prompts 发现
    ▼
MCPTool / ListMcpResourcesTool / ReadMcpResourceTool / McpAuthTool
    │  适配为 Claude Code 内置工具
    ▼
QueryEngine → Claude API（tools 数组包含 MCP 工具）
```

---

## 三、传输层（Transport）— 6 种连接方式

```typescript
export type Transport = 'stdio' | 'sse' | 'sse-ide' | 'http' | 'ws' | 'sdk'
```

| 传输类型 | 场景 | 实现 |
|---|---|---|
| `stdio` | 本地进程（最常用）| `StdioClientTransport` |
| `sse` | 远程 SSE 服务器 | `SSEClientTransport` |
| `http` | Streamable HTTP | `StreamableHTTPClientTransport` |
| `ws` | WebSocket | 自定义 WS transport |
| `sse-ide` | IDE 扩展（内部）| `McpSSEIDEServerConfig` |
| `sdk` | SDK 内嵌 MCP | `McpSdkServerConfig` |

---

## 四、服务器配置类型

Zod Schema 验证，支持多种配置：

```typescript
// stdio（最常用）
McpStdioServerConfig: {
  type?: 'stdio'       // 可选（向后兼容）
  command: string      // 可执行文件路径
  args: string[]
  env?: Record<string, string>
}

// SSE（远程）
McpSSEServerConfig: {
  type: 'sse'
  url: string
  headers?: Record<string, string>
  headersHelper?: string     // 动态 header 生成脚本
  oauth?: McpOAuthConfig
}

// HTTP Streamable
McpHTTPServerConfig: {
  type: 'http'
  url: string
  headers?: Record<string, string>
  headersHelper?: string
  oauth?: McpOAuthConfig
}
```

**配置作用域（ConfigScope）：**

```typescript
type ConfigScope = 'local' | 'user' | 'project' | 'dynamic' | 'enterprise' | 'claudeai' | 'managed'
```

- `local`：项目级（`.claude/settings.local.json`）
- `user`：用户级（`~/.claude/settings.json`）
- `project`：项目级（`.claude/settings.json`，可提交到 git）
- `enterprise`：企业 MDM 下发
- `claudeai`：通过 claude.ai 代理连接的 MCP 服务器
- `managed`：插件提供的 MCP 服务器

---

## 五、服务器连接状态（MCPServerConnection）

```typescript
type MCPServerConnection =
  | ConnectedMCPServer     // 已连接，有 capabilities/tools
  | FailedMCPServer        // 连接失败，有 error 信息
  | NeedsAuthMCPServer     // 需要认证（OAuth）
  | PendingMCPServer       // 正在连接中，有 reconnectAttempt
  | DisabledMCPServer      // 已禁用
```

**ConnectedMCPServer 关键字段：**

```typescript
type ConnectedMCPServer = {
  client: Client              // @modelcontextprotocol/sdk 的 Client 实例
  name: string
  type: 'connected'
  capabilities: ServerCapabilities
  serverInfo?: { name: string; version: string }
  instructions?: string       // 服务器使用说明（注入系统提示词）
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>
}
```

---

## 六、client.ts — 核心客户端逻辑

`src/services/mcp/client.ts` 是 MCP 集成的核心，负责：

1. **连接建立**：根据 config.type 选择对应 Transport，调用 `client.connect()`
2. **工具发现**：`client.listTools()` → 转换为 `MCPTool` 实例
3. **资源发现**：`client.listResources()` → 转换为 `ServerResource[]`
4. **Prompts 发现**：`client.listPrompts()` → 转换为 Skills
5. **工具调用**：`client.callTool(name, args)` → 解析结果（文本/图片/二进制）
6. **OAuth 处理**：监听 `-32042` 错误码（`elicitationHandler.ts`）触发认证流程
7. **重连机制**：`PendingMCPServer.reconnectAttempt` 跟踪，有最大重试次数
8. **图片/大结果处理**：`maybeResizeAndDownsampleImageBuffer`、`persistBinaryContent`（大文件落盘）

### 工具名规范化

MCP 工具名经过规范化：

```typescript
// 标准模式（有前缀）
mcp__serverName__toolName

// 无前缀模式（CLAUDE_AGENT_SDK_MCP_NO_PREFIX 环境变量）
toolName
```

`normalization.ts` 处理名称冲突和特殊字符（`-`→`_` 等），`mcpInfo.originalToolName` 保存原始名。

---

## 七、三类 MCP 能力

| 能力 | 协议方法 | Claude Code 适配 |
|---|---|---|
| **Tools** | `tools/list` + `tools/call` | `MCPTool`（isMcp=true）|
| **Resources** | `resources/list` + `resources/read` | `ListMcpResourcesTool` + `ReadMcpResourceTool` |
| **Prompts** | `prompts/list` + `prompts/get` | Skills（`MCP_SKILLS` flag 启用时）|

---

## 八、认证体系（OAuth）

```
MCP server 调用需要 auth
    │  返回 -32042 (ElicitRequest with URL params)
    ▼
elicitationHandler.ts
    │
    ├── REPL 模式 → 弹出 McpAuthTool UI，用户在浏览器完成 OAuth
    └── SDK 模式 → handleElicitation 回调（ToolUseContext.handleElicitation）

auth.ts：
    ├── checkAndRefreshOAuthTokenIfNeeded()  ← 检查并刷新已有 token
    ├── getClaudeAIOAuthTokens()             ← 获取 Claude.ai OAuth tokens
    └── handleOAuth401Error()               ← 401 时触发重新认证

XAA（Cross-App Access / SEP-990）：
    ├── xaa.ts             ← 跨应用访问协议
    └── xaaIdpLogin.ts     ← IdP 登录流程
```

**OAuth 配置（McpOAuthConfig）：**

```typescript
{
  clientId?: string
  callbackPort?: number            // OAuth 回调端口
  authServerMetadataUrl?: string   // 必须 https://
  xaa?: boolean                    // 启用 Cross-App Access
}
```

---

## 九、MCP Skills（`MCP_SKILLS` flag 启用）

外部版启用了 `MCP_SKILLS`，MCP Prompts 被映射为 Skills：

```
MCP server 暴露 prompts/list
    │  
    ▼
getSlashCommandToolSkills(cwd)
    │
    ├── 用户通过 /skill-name 调用
    └── SkillTool 执行（获取 prompt template + 注入系统提示词）
```

---

## 十、MCPConnectionManager

`MCPConnectionManager.tsx`（React 组件）负责生命周期管理：

- 启动时（`src/main.tsx`）初始化所有配置的 MCP 连接
- 监听连接状态变化，更新 AppState
- 处理断线重连（`PendingMCPServer.reconnectAttempt`）
- `useManageMCPConnections.ts` Hook 封装连接管理逻辑

---

## 十一、特殊 MCP 集成

### VSCode SDK MCP（`vscodeSdkMcp.ts`）

IDE 扩展通过 SSE-IDE 传输类型连接，用于：
- 通知 IDE 文件已更新（`notifyVscodeFileUpdated`）
- IDE 安装状态检测（`ideInstallationStatus`）

### Claude.ai 代理（`claudeai.ts`）

通过 claude.ai 平台代理连接的 MCP 服务器，scope = `claudeai`，使用 `McpClaudeAIProxyServerConfig`。

### InProcessTransport（`InProcessTransport.ts`）

进程内 MCP 传输，用于内置的 MCP server（无需外部进程）。例如 `@ant/claude-for-chrome-mcp`（Chrome 扩展集成，外部版 stub）。

### 官方 Registry（`officialRegistry.ts`）

从 Anthropic 官方 MCP Registry 获取推荐的 MCP 服务器列表。

### Channel 相关

- `channelAllowlist.ts` — MCP channel 白名单（Anthropic 内部 channel 管理）
- `channelPermissions.ts` — channel 权限检查
- `channelNotification.ts` — channel 通知

---

## 十二、MCPCliState — /mcp 命令状态

`/mcp` 命令（`src/commands/mcp/`）通过 `MCPCliState` 展示当前 MCP 状态：

```typescript
interface MCPCliState {
  clients: SerializedClient[]          // 各服务器连接状态
  configs: Record<string, ScopedMcpServerConfig>
  tools: SerializedTool[]              // 所有可用 MCP 工具
  resources: Record<string, ServerResource[]>
  normalizedNames?: Record<string, string>  // 规范化名称映射
}
```

---

## 十三、环境变量扩展（`envExpansion.ts`）

MCP 服务器配置中的 env 字段支持变量扩展：
- `${PATH}` → 当前 PATH 环境变量
- 防止路径注入（安全检查）

---

## 十四、相关文件

| 文件 | 说明 |
|---|---|
| `src/services/mcp/types.ts` | 全部类型定义（Transport、Config、Connection）|
| `src/services/mcp/client.ts` | 核心客户端逻辑（连接、工具调用、认证）|
| `src/services/mcp/MCPConnectionManager.tsx` | React 生命周期管理 |
| `src/services/mcp/auth.ts` | OAuth token 管理 |
| `src/services/mcp/elicitationHandler.ts` | -32042 OAuth 触发处理 |
| `src/services/mcp/normalization.ts` | 工具名规范化 |
| `src/services/mcp/envExpansion.ts` | 配置中的环境变量扩展 |
| `src/tools/MCPTool/MCPTool.ts` | MCP 工具适配器 |
| `src/tools/McpAuthTool/McpAuthTool.ts` | 认证交互工具 |
| `src/tools/ListMcpResourcesTool/` | 资源列表工具 |
| `src/tools/ReadMcpResourceTool/` | 资源读取工具 |
| `stubs/@anthropic-ai/mcpb/` | 私有 MCP 实现 stub（外部版用标准 MCP 替代）|
