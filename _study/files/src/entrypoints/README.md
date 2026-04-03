# src/entrypoints/ — 程序入口层

> 目录：[`src/entrypoints/`](../../../../../../src/entrypoints/)（6 个文件 + `sdk/` 子目录）  
> 角色：**进程启动分发层**，负责在加载完整 CLI 之前尽快识别运行模式，将控制权交给对应处理器。

---

## 文件速览

| 文件 | 行数 | 职责 |
|---|---|---|
| `cli.tsx` | 302 | CLI 主入口，fast-path 分发，所有 import 动态加载 |
| `init.ts` | 340 | 完整初始化序列（配置、网络、遥测、OAuth）|
| `mcp.ts` | 196 | 将 Claude Code 工具暴露为 MCP 服务端 |
| `agentSdkTypes.ts` | 443 | Agent SDK 对外类型定义 |
| `sdk/` | — | SDK 辅助类型（controlSchemas, coreTypes, toolTypes 等）|

---

## cli.tsx — 启动分发逻辑

### 核心原则

> "All imports are dynamic to minimize module evaluation for fast paths."

`main()` 函数在任何模块加载之前先检查命令行参数，发现特殊模式后立即分发，避免加载不必要的模块。

### 全局环境预处理（模块顶层，最先执行）

```typescript
process.env.COREPACK_ENABLE_AUTO_PIN = '0'  // 禁止 corepack 污染 package.json

// CCR 容器环境：提升 V8 堆上限
if (process.env.CLAUDE_CODE_REMOTE === 'true')
  process.env.NODE_OPTIONS = '--max-old-space-size=8192'

// ABLATION_BASELINE 实验：关闭思考、compact、后台任务等，模拟 baseline 性能
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE)
  // 批量 set ??= '1'：DISABLE_COMPACT, DISABLE_INTERLEAVED_THINKING, 等
```

> 注：`ABLATION_BASELINE` 必须在顶层处理，因为 BashTool/AgentTool 在模块加载时就把 `DISABLE_BACKGROUND_TASKS` 捕获到了常量里，`init()` 调用太晚。

### Fast-Path 分发表

```
--version / -v / -V
    → console.log(MACRO.VERSION)（零模块加载）

--dump-system-prompt                      [feature: DUMP_SYSTEM_PROMPT, 仅内部]
    → getSystemPrompt() 输出后 exit

--claude-in-chrome-mcp
    → runClaudeInChromeMcpServer()

--chrome-native-host
    → runChromeNativeHost()

--computer-use-mcp                        [feature: CHICAGO_MCP]
    → runComputerUseMcpServer()

--daemon-worker=<kind>                    [feature: DAEMON]
    → runDaemonWorker()（supervisor 为每 worker 调用，性能敏感，不加载 configs）

remote-control / rc / remote / bridge     [feature: BRIDGE_MODE]
    → 检查 auth → 检查 GrowthBook gate → 检查 policy → bridgeMain()

daemon [subcommand]                       [feature: DAEMON]
    → daemonMain()

ps / logs / attach / kill / --bg          [feature: BG_SESSIONS]
    → bg.psHandler / logsHandler / attachHandler / killHandler

└─ 以上都不匹配 → 正常 CLI 路径（加载完整 main.tsx）
```

每个 fast-path 后都跟 `return`，保证互斥。

---

## init.ts — 完整初始化序列

`init()` 函数使用 `memoize` 包装，保证全生命周期只执行一次。

### 初始化顺序

```
1. enableConfigs()                    — 加载/验证 settings.json，解析错误时展示 InvalidConfigDialog
2. applySafeConfigEnvironmentVariables()  — 仅应用「安全」env vars（trust dialog 前）
3. applyExtraCACertsFromConfig()      — 注入自定义 CA 证书（TLS 首次握手前必须完成）
4. setupGracefulShutdown()            — 注册进程退出清理钩子
5. initialize1PEventLogging()         — 延迟加载 OpenTelemetry sdk-logs（已在模块缓存，无额外开销）
6. populateOAuthAccountInfoIfNeeded() — 补全 OAuth 账号信息（VSCode 扩展登录时可能缺失）
7. initJetBrainsDetection()           — JetBrains IDE 检测（异步，填充缓存）
8. detectCurrentRepository()          — GitHub 仓库检测（异步，用于 gitDiff PR 链接）
9. initializeRemoteManagedSettingsLoadingPromise()  — MDM 远程设置预加载
10. initializePolicyLimitsLoadingPromise()          — 策略限制预加载
11. recordFirstStartTime()
12. configureGlobalMTLS()             — mTLS 全局配置
13. configureGlobalAgents()           — HTTP 代理配置
14. preconnectAnthropicApi()          — TCP+TLS 预热（与后续 ~100ms 操作并发）
15. initUpstreamProxy()               — CCR 上游代理（仅 CLAUDE_CODE_REMOTE=true）
16. setShellIfWindows()
17. registerCleanup(shutdownLspServerManager)
18. registerCleanup(cleanupSessionTeams)
19. ensureScratchpadDir()             — scratchpad 目录创建（如启用）
```

### initializeTelemetryAfterTrust()

遥测初始化刻意延迟到用户授权（trust dialog）之后调用：
- 若需要 remote managed settings：等 settings 加载完毕 → `applyConfigEnvironmentVariables()` → 初始化
- 否则：直接初始化
- OpenTelemetry 约 400KB，gRPC 导出器约 700KB，全部懒加载避免影响启动速度

---

## mcp.ts — Claude Code 作为 MCP 服务端

`startMCPServer(cwd, debug, verbose)` 将 Claude Code 自身的所有工具通过 MCP 协议暴露给外部。

```
Server('claude/tengu', version)
├── ListTools:  getTools() → 转换 Zod schema → JSON Schema
│              outputSchema 只包含 type:"object" 根节点（跳过 union 类型）
└── CallTool:   findToolByName() → tool.validateInput() → tool.call()
               isNonInteractiveSession: true（无 UI 渲染）
               tools 包含所有 49 个 Claude Code 工具
```

额外暴露：`/review` slash 命令（`MCP_COMMANDS = [review]`）

传输层：`StdioServerTransport`（标准输入/输出，适合作为 subprocess 运行）

---

## agentSdkTypes.ts / sdk/

Agent SDK 的对外类型定义层，供第三方集成使用：
- `controlSchemas.ts` — 控制消息 schema
- `coreSchemas.ts` — 核心数据 schema  
- `coreTypes.ts` / `coreTypes.generated.ts` — 核心类型（部分自动生成）
- `runtimeTypes.ts` — 运行时类型
- `toolTypes.ts` — 工具相关类型
- `types.ts` — 通用类型

---

## 关键设计点

1. **动态 import 策略**：`cli.tsx` 中所有 `import` 都在函数体内动态执行，避免模块图在启动时全量加载。`--version` 是极端案例：零 import，直接打印。

2. **`feature()` DCE**：每个 fast-path 用 `feature('FEATURE_NAME')` 包裹，build 时对应 flag 为 false 则整块代码被 tree-shake 掉，不进入发布产物。

3. **CA 证书时序**：`applyExtraCACertsFromConfig()` 必须在 TLS 连接建立之前调用，因为 Bun 的 BoringSSL 在启动时缓存证书库。

4. **mTLS/代理先于预连接**：`preconnectAnthropicApi()` 在代理和 mTLS 配置完成后才调用，确保预热连接走正确的传输。
