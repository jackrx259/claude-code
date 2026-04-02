# src/tools/ — 工具实现目录

每个子目录对应一个 Tool 实现。工具是 Claude 可调用的能力单元。

## 工具注册机制

工具在 `src/tools.ts`（入口聚合文件）中统一导出，
在 `src/main.tsx` 初始化阶段注册到系统中，
在 `QueryEngine` 的每次 API 调用前通过 `getTools()` 加载。

每个工具实现需要：
1. 继承/实现 `Tool`（定义于 `src/Tool.ts`）
2. 声明 `inputSchema`（Zod schema，发给 Claude 作为 tool definition）
3. 声明 `outputSchema`（Zod schema）
4. 实现 `execute()` 方法
5. 声明权限需求（read-only / write / network 等）

## 工具清单

### 文件系统类

| 工具目录 | 功能 |
|---|---|
| `FileReadTool/` | 读文件内容（支持 PDF、图片、Notebook）|
| `FileEditTool/` | 精确字符串替换编辑文件 |
| `FileWriteTool/` | 写入/创建文件 |
| `GlobTool/` | 文件路径模式匹配（类 find）|
| `GrepTool/` | 基于 ripgrep 的内容搜索 |

### 执行类

| 工具目录 | 功能 |
|---|---|
| `BashTool/` | 执行 shell 命令 |
| `PowerShellTool/` | Windows PowerShell 命令执行 |
| `REPLTool/` | 持久化 REPL 会话（保持 shell 状态）|

### Agent & 任务类

| 工具目录 | 功能 |
|---|---|
| `AgentTool/` | 启动子 Agent（general-purpose / Explore / Plan 等）|
| `TaskCreateTool/` | 创建任务 |
| `SendMessageTool/` | 向运行中的 Agent 发送消息 |
| `RemoteTriggerTool/` | 触发远程 Agent |
| `ScheduleCronTool/` | 创建定时任务 |
| `SleepTool/` | 等待（用于轮询场景）|

### 交互类

| 工具目录 | 功能 |
|---|---|
| `AskUserQuestionTool/` | 向用户提问（打断流程等待输入）|
| `SkillTool/` | 调用 Skills（slash command 扩展）|
| `EnterPlanModeTool/` | 进入 Plan 模式 |
| `ExitPlanModeTool/` | 退出 Plan 模式 |
| `EnterWorktreeTool/` | 进入 git worktree 隔离环境 |
| `ExitWorktreeTool/` | 退出 git worktree |

### MCP 类

| 工具目录 | 功能 |
|---|---|
| `MCPTool/` | 调用 MCP server 暴露的工具 |
| `McpAuthTool/` | MCP 认证 |
| `ListMcpResourcesTool/` | 列出 MCP 资源 |
| `ReadMcpResourceTool/` | 读取 MCP 资源 |

### 其他

| 工具目录 | 功能 |
|---|---|
| `LSPTool/` | LSP（语言服务器）集成，获取诊断信息 |
| `NotebookEditTool/` | Jupyter Notebook 编辑 |
| `BriefTool/` | 生成简报 |
| `ConfigTool/` | 配置读写 |
| `SuggestBackgroundPRTool/` | 建议创建后台 PR |
| `SyntheticOutputTool/` | 合成输出（测试/调试用）|

## 待深入

- [ ] BashTool：沙箱机制、超时控制、输出截断
- [ ] AgentTool：子 Agent 的 context 隔离、worktree 机制
- [ ] MCPTool：协议适配层、权限传递
- [ ] FileEditTool：`old_string` 唯一性校验逻辑