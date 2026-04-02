# MCP System — Model Context Protocol 体系

> 核心目录：`src/services/mcp/`

## 什么是 MCP

MCP（Model Context Protocol）是 Anthropic 开发的开放协议，
允许外部服务通过标准接口向 Claude 暴露工具、资源和提示词模板。

Claude Code 作为 MCP **客户端**，可以连接任意 MCP server。

## 架构角色

```
MCP Server（外部进程/服务）
    │  JSON-RPC over stdio/SSE/HTTP
    ▼
src/services/mcp/       ← MCP 客户端管理层
    │  发现工具、资源、prompts
    ▼
MCPTool / ListMcpResourcesTool / ReadMcpResourceTool
    │  适配为 Claude Code 工具
    ▼
QueryEngine → Claude API
```

## 三类 MCP 能力

| 类型 | 说明 | 对应工具 |
|---|---|---|
| Tools | MCP server 暴露的工具 | `MCPTool` |
| Resources | MCP server 暴露的资源（文件、数据） | `ListMcpResourcesTool` / `ReadMcpResourceTool` |
| Prompts | MCP server 暴露的提示词模板 | （通过 Skills 系统集成）|

## MCP Skills（`MCP_SKILLS` flag 启用）

外部版启用了 `MCP_SKILLS`，这意味着：
- MCP server 可以暴露 Skills（可调用的预定义工作流）
- 用户可以通过 slash command（`/skill-name`）调用
- `SkillTool` 负责执行

## MCP 认证

`McpAuthTool` 处理需要认证的 MCP server（OAuth 等），
认证流程通过 `src/services/mcp/` 中的认证服务处理。

## MCP Server 管理

- MCP server 配置存储在用户设置中
- Claude Code 启动时（`src/main.tsx`）初始化所有配置的 MCP server 连接
- `/mcp` 命令用于管理 MCP server（列出、添加、删除、查看状态）

## 内部 MCP（`@anthropic-ai/mcpb`）

`stubs/` 中存在 `@anthropic-ai/mcpb` 的 stub，说明：
- Anthropic 内部有一个私有的 MCP 实现 (`mcpb`)
- 外部版用 stub 替代，使用标准 MCP 协议

## 关键问题待研究

- [ ] MCP server 连接的生命周期管理（断线重连？）
- [ ] MCP 工具的权限如何与内置工具权限体系统一？
- [ ] `mcpServerApproval.tsx` — 首次使用 MCP server 的确认流程

## 相关文件

- MCP 服务：[`src/services/mcp/`](../../src/services/mcp/)
- MCP 工具：[`src/tools/MCPTool/`](../../src/tools/MCPTool/)
- MCP 认证：[`src/tools/McpAuthTool/`](../../src/tools/McpAuthTool/)
- MCP 资源：[`src/tools/ListMcpResourcesTool/`](../../src/tools/ListMcpResourcesTool/)
- MCP 审批 UI：[`src/services/mcpServerApproval.tsx`](../../src/services/mcpServerApproval.tsx)
- Stub：[`stubs/@anthropic-ai/mcpb/`](../../stubs/@anthropic-ai/mcpb/)
