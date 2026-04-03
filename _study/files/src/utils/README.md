# src/utils/ — 工具函数目录概览

> 目录：[`src/utils/`](../../../../../../src/utils/)（330 个文件，32 个子目录）  
> 角色：**跨切面工具层**，无业务逻辑的基础设施函数，被所有层（tools、services、components、screens）调用。

---

## 子目录速览（32 个）

| 子目录 | 职责 |
|---|---|
| `permissions/` | 权限规则解析、模式判断、分类器、bash 规则匹配 |
| `settings/` | settings.json 读写、验证、MDM 合并、schema |
| `model/` | 模型信息、能力查询、cost 计算、deprecation 管理 |
| `messages/` | 消息构造/转换辅助 |
| `hooks/` | session hooks、post-sampling hooks（`~/.claude/hooks/` 配置）|
| `git/` | git 操作封装（status、log、branch、worktree）|
| `github/` | GitHub API 辅助（PR 状态、链接）|
| `shell/` | shell 命令解析、历史、通配符展开 |
| `bash/` | Bash 特定工具（注入检测、危险命令等）|
| `swarm/` | 多 Agent swarm 辅助（team helpers、权限同步、leader bridge）|
| `teleport/` | Teleport 远程会话辅助 |
| `telemetry/` | OpenTelemetry 初始化、session tracing |
| `processUserInput/` | 用户输入处理（@引用解析、路径展开）|
| `suggestions/` | 自动补全建议（shell 历史、文件、命令）|
| `sandbox/` | macOS Seatbelt / Linux seccomp 沙箱 |
| `secureStorage/` | macOS Keychain / Linux libsecret 安全存储 |
| `memory/` | CLAUDE.md 记忆文件操作 |
| `task/` | 任务辅助（任务状态操作）|
| `todo/` | Todo 列表类型和操作 |
| `skills/` | Skills（skill 文件加载、执行）|
| `plugins/` | 插件加载、manifest 解析 |
| `mcp/` | MCP 工具辅助（schema 转换、transport）|
| `filePersistence/` | 文件持久化（原子写、备份）|
| `background/` | 后台任务文件管理（输出持久化）|
| `claudeInChrome/` | Chrome 扩展集成（MCP server、native host）|
| `computerUse/` | Computer Use MCP（截图、点击、键盘）|
| `deepLink/` | 深链接处理（claude:// URL scheme）|
| `dxt/` | DXT（Desktop Extension）包管理 |
| `nativeInstaller/` | 原生模块安装器 |
| `powershell/` | Windows PowerShell 支持 |
| `ultraplan/` | UltraPlan 功能（深度规划模式）|

---

## 关键顶层文件

### 认证

| 文件 | 职责 |
|---|---|
| `auth.ts` (~65KB) | 认证核心：OAuth token 管理、keychain 存取、多认证模式 |
| `authFileDescriptor.ts` | 文件描述符认证（传递给子进程）|
| `authPortable.ts` | 跨平台认证适配 |
| `aws.ts` + `awsAuthStatusManager.ts` | AWS Bedrock 认证（IAM、SSO）|

### 配置 & 设置

| 文件 | 职责 |
|---|---|
| `config.ts` | 全局配置（`~/.claude/` 目录、settings 加载）|
| `configConstants.ts` | 配置路径常量 |
| `claudemd.ts` | CLAUDE.md 文件发现和读取（项目记忆）|
| `managedEnv.ts` | MDM 管理的环境变量（企业策略注入）|

### Shell & 进程执行

| 文件 | 职责 |
|---|---|
| `Shell.ts` | `exec()` 核心封装（环境变量、cwd、超时、流式输出）|
| `ShellCommand.ts` | shell 命令结构化表示 |
| `execFileNoThrow.ts` | `execFile` 不抛异常版（返回 exit code）|
| `execSyncWrapper.ts` | `execSync` 封装 |
| `promptShellExecution.ts` | 提示型 shell 执行（需用户确认）|
| `subprocessEnv.ts` | 子进程环境变量注入（proxy、auth 等）|

### 消息处理

| 文件 | 职责 |
|---|---|
| `messages.ts` | 消息工厂函数（createUserMessage、createAssistantMessage 等）|
| `messagePredicates.ts` | 消息类型判断（isHumanTurn、isToolResult 等）|
| `messageQueueManager.ts` | 消息队列管理 |
| `handlePromptSubmit.ts` | 提交处理辅助（REPL.tsx `onSubmit` 的辅助层）|

### 会话管理

| 文件 | 职责 |
|---|---|
| `sessionStorage.ts` | 会话持久化（消息历史写磁盘）|
| `sessionState.ts` | 会话状态（session ID、起始时间）|
| `sessionStart.ts` | 会话启动初始化 |
| `sessionRestore.ts` | 会话恢复（从磁盘加载历史）|
| `sessionTitle.ts` | 会话标题生成（用 Claude API 自动生成）|
| `listSessionsImpl.ts` | 列举历史会话实现 |
| `concurrentSessions.ts` | 并发会话检测（防止同目录多实例冲突）|

### 权限系统（`permissions/` 子目录）

```
permissions/
├── PermissionMode.ts     — 权限模式枚举（default/plan/auto/bypassPermissions）
├── PermissionRule.ts     — 规则数据结构
├── PermissionResult.ts   — 决策结果（allow/deny/ask）
├── PermissionUpdate.ts   — 更新操作（applyPermissionUpdate、persistPermissionUpdate）
├── permissions.ts        — 顶层入口（hasPermissionsToUseTool）
├── permissionsLoader.ts  — 从 settings 加载规则
├── permissionRuleParser.ts  — 规则字符串解析（路径前缀、通配符）
├── shellRuleMatching.ts  — shell 命令规则匹配
├── bashClassifier.ts     — Bash 命令 AI 分类（auto-approve/deny）
├── yoloClassifier.ts     — YOLO 模式（auto-approve all）
├── autoModeState.ts      — Auto 模式状态追踪
├── dangerousPatterns.ts  — 危险命令模式列表
└── filesystem.ts         — 文件系统权限检查
```

### 模型管理（`model/` 子目录）

```
model/
├── model.ts         — 主入口：getMainLoopModel()、模型选择逻辑
├── modelCapabilities.ts  — 能力查询（支持 thinking？支持 extended output？）
├── modelCost.ts     — token 单价表
├── antModels.ts     — Anthropic 内部模型列表
├── bedrock.ts       — Bedrock 模型 ID 映射
├── deprecation.ts   — 旧模型废弃提示
└── validateModel.ts — 模型 ID 验证
```

### 性能 & 调试

| 文件 | 职责 |
|---|---|
| `startupProfiler.ts` | 启动性能分析（`profileCheckpoint()`，记录各阶段耗时）|
| `queryProfiler.ts` | Query 执行性能分析 |
| `headlessProfiler.ts` | 无 UI 模式的 profiler |
| `QueryGuard.ts` | 防并发 query 的状态机（REPL.tsx 核心依赖）|
| `debug.ts` | `logForDebugging()`（写到 `~/.claude/logs/`）|
| `diagLogs.ts` | 诊断日志（无 PII）|

### 文件操作

| 文件 | 职责 |
|---|---|
| `file.ts` | 文件读写基础操作 |
| `fileRead.ts` | 文件读取（带 size limit、encoding 处理）|
| `fileStateCache.ts` | 文件状态 LRU 缓存（最近修改时间、hash）|
| `fileHistory.ts` | 文件历史快照（每次编辑前记录）|
| `diff.ts` | unified diff 生成 |
| `fsOperations.ts` | 原子文件操作（rename、mkdir -p）|
| `lockfile.ts` | 文件锁（防多进程竞争）|

### 格式 & 显示

| 文件 | 职责 |
|---|---|
| `format.ts` | 数字/字节/时间格式化（`formatTokens`、`truncateToWidth`）|
| `markdown.ts` | Markdown 渲染辅助 |
| `cliHighlight.ts` | 语法高亮（终端输出）|
| `ansiToPng.ts` + `ansiToSvg.ts` | ANSI 输出转图片 |
| `truncate.ts` | 字符串截断（带 ANSI 支持）|
| `sliceAnsi.ts` | ANSI 字符串切片 |
| `hyperlink.ts` | 终端超链接（OSC 8）|

### 网络 & API

| 文件 | 职责 |
|---|---|
| `http.ts` | HTTP 请求封装（fetch 配置）|
| `proxy.ts` | HTTP 代理配置（读 `HTTP_PROXY` 等）|
| `mtls.ts` | mTLS 客户端证书配置 |
| `api.ts` | API 相关辅助（endpoint、headers）|
| `apiPreconnect.ts` | Anthropic API TCP+TLS 预热 |

### Git 操作（`git/` 子目录）

```
git/
├── status.ts    — git status 解析
├── log.ts       — git log 解析
├── branch.ts    — 分支操作
├── worktree.ts  — worktree 管理
└── ...
```
顶层还有 `git.ts`（主入口）、`gitDiff.ts`（diff 生成）、`gitSettings.ts`（git 配置读取）。

### 其他亮点

| 文件 | 职责 |
|---|---|
| `CircularBuffer.ts` | 固定大小环形缓冲区（日志、输出截断）|
| `Cursor.ts` | 终端光标位置管理 |
| `lazySchema.ts` | Zod schema 懒加载（feature flag 动态增删字段）|
| `zodToJsonSchema.ts` | Zod → JSON Schema 转换（工具定义）|
| `handlePromptSubmit.ts` | 用户提交处理（REPL.tsx `onSubmit` 辅助）|
| `processUserInput/` | 输入预处理（@ 引用解析、paste 引用展开）|
| `autoUpdater.ts` | 自动更新检查（npm 版本对比）|
| `worktree.ts` | git worktree 隔离模式操作 |
| `cron.ts` + `cronScheduler.ts` | 定时任务调度 |
| `thinking.ts` | thinking mode 配置（extended thinking）|
| `tokenBudget.ts` | token 预算管理（防超 context window）|

---

## 依赖关系

utils/ 是最底层，不应反向依赖 services/、tools/、components/。实际上：
- `auth.ts`、`config.ts`、`Shell.ts` 几乎被所有模块引用
- `messages.ts`、`messagePredicates.ts` 被 QueryEngine、REPL、tools 大量使用
- `permissions/` 被 Tool.ts、BashTool、REPL.tsx 使用
- `model/` 被 QueryEngine、main.tsx、entrypoints 使用
