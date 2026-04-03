# src/main.tsx — 全局初始化与 CLI 命令注册

> 文件：[`src/main.tsx`](../../../../src/main.tsx)（4683 行）  
> 角色：**系统的中央接线员**，负责所有子系统的初始化、CLI 命令定义、交互/无头路径的分流。

## 职责总览

`main.tsx` 是整个应用最复杂的文件。它：

1. **预取优化**：模块评估阶段并行启动 MDM 读取、keychain 预取（节省 ~65ms 启动时间）
2. **定义 `main()` 函数**：由 `cli.tsx` 入口调用
3. **定义 `run()` 函数**：Commander.js 命令树，105+ 子命令注册
4. **处理启动分流**：交互 REPL vs `-p` 无头模式 vs SDK 打印模式
5. **迁移管理**：11 个配置迁移函数，升级用户设置
6. **遥测**：启动事件、managed settings、skills/plugins 统计

---

## 启动时序（模块加载阶段）

```typescript
// 模块加载时（同步，import 阶段）立即执行：
profileCheckpoint('main_tsx_entry')     // 1. 记录入口时间点
startMdmRawRead()                        // 2. 并行启动 MDM subprocess（plutil/reg query）
startKeychainPrefetch()                  // 3. 并行启动两个 macOS keychain 读取

// ... ~135ms 的其余 import 加载 ...

profileCheckpoint('main_tsx_imports_loaded')  // 导入完成
```

**为什么这么早启动**：MDM 和 keychain 读取是同步 spawn，必须在模块链评估期间并行运行，否则会在 init() 中串行执行（+65ms 启动延迟）。

---

## 顶层函数总览

| 函数 | 行数 | 说明 |
|---|---|---|
| `main()` | 585–856 | 入口，feature flag 快速路径 + `run()` |
| `run()` | 884–4513 | Commander.js 命令树，默认命令是 REPL |
| `logStartupTelemetry()` | 307–321 | 异步上报启动事件 |
| `logSessionTelemetry()` | 279–290 | 上报 skills/plugins 加载情况 |
| `runMigrations()` | 326–358 | 运行所有配置迁移 |
| `prefetchSystemContextIfSafe()` | 360–387 | 预取 system context（如 git 信息）|
| `startDeferredPrefetches()` | 388–431 | 非阻塞预取（GrowthBook、pass eligibility 等）|
| `eagerLoadSettings()` | 502–516 | 同步加载设置（model strings 初始化）|
| `initializeEntrypoint()` | 517–584 | 入口公共初始化（cwd、日志、session ID）|
| `logTenguInit()` | 4514–4610 | 详细的 tengu_init 遥测事件 |

---

## `main()` 函数结构（585–856 行）

```
Windows PATH 劫持防御（NoDefaultCurrentDirectoryInExePath）
    ↓
SIGINT 处理注册（-p 模式跳过，交由 print.ts 的 handler）
    ↓
DIRECT_CONNECT feature: cc:// URL 重写（cc:// → internal `open` 子命令）
    ↓
LODESTONE feature: deep link URI 处理（--handle-uri）
    ↓
KAIROS feature: assistant 模式检查
    ↓
await run()  ← 进入 Commander.js 命令树
```

---

## `run()` 函数结构（884–4513 行）

这是整个 CLI 的命令定义，基于 `@commander-js/extra-typings`：

```
new CommanderCommand()
    .hook('preAction', async () => {
        ensureMdmSettingsLoaded() + ensureKeychainPrefetchCompleted()
        await init()           // auth + config 初始化
        initSinks()            // 日志 sink 注册
        runMigrations()        // 配置迁移
        loadRemoteManagedSettings()
        loadPolicyLimits()
    })
    ↓
// 子命令注册（按功能分组）：
program
  .command('doctor')           // 诊断工具
  .command('mcp')              // MCP 服务器管理
  .command('plugin')           // 插件管理
  .command('auth')             // 认证管理
  .command('config')           // 配置管理
  .command('migrate')          // 手动迁移
  .command('open')             // 打开 cc:// URL（DIRECT_CONNECT）
  .command('snapshot')         // 快照管理（LODESTONE）
  // ... 更多子命令 ...
    ↓
// 默认动作（无子命令时）= 启动 REPL 或 -p 无头模式
program.action(async (options) => {
    // 处理 --print(-p) 标志 → runHeadless()
    // 处理 --resume → 恢复会话
    // 处理 --teleport → teleport 模式
    // 否则 → launchRepl()
})
```

---

## 关键初始化路径

### 交互 REPL 路径

```
program.action()
    → initializeEntrypoint(isNonInteractive=false)
    → showSetupScreens()           // 首次使用向导
    → await launchRepl({           // src/replLauncher.tsx
        tools, commands, mcpClients,
        initialMessages, ...
      })
```

### 无头（-p）路径

```
program.action() → 检测到 -p/--print
    → initializeEntrypoint(isNonInteractive=true)
    → getInputPrompt()             // stdin 或命令行参数
    → runHeadless() / print.ts
        → new QueryEngine(config)
        → engine.submitMessage(prompt)
        → 输出到 stdout
```

### SDK 路径（程序化调用）

不经过 main.tsx，直接使用：
```
import { ask } from 'src/QueryEngine.js'
// 或
import { QueryEngine } from 'src/QueryEngine.js'
```

---

## 迁移系统（326–358 行）

```typescript
const CURRENT_MIGRATION_VERSION = 11

function runMigrations(): void {
  if (config.migrationVersion !== CURRENT_MIGRATION_VERSION) {
    migrateAutoUpdatesToSettings()           // 迁移自动更新设置
    migrateBypassPermissionsAcceptedToSettings()
    migrateEnableAllProjectMcpServersToSettings()
    resetProToOpusDefault()                  // 模型默认值重置
    migrateSonnet1mToSonnet45()             // 模型字符串迁移
    migrateLegacyOpusToCurrent()
    migrateSonnet45ToSonnet46()
    migrateOpusToOpus1m()
    migrateReplBridgeEnabledToRemoteControlAtStartup()
    // ... feature-gated 迁移 ...
    saveGlobalConfig(prev => ({ ...prev, migrationVersion: 11 }))
  }
  migrateChangelogFromConfig()  // 异步迁移（fire-and-forget）
}
```

**规则**：新模型发布时必须检查是否需要新增迁移（见 `@[MODEL LAUNCH]` 注释）。

---

## 预取优化（388–431 行）

`startDeferredPrefetches()` 在 REPL 启动后异步触发：

```typescript
export function startDeferredPrefetches(): void {
  void initializeGrowthBook()              // feature flag 系统
  void prefetchPassesEligibility()         // referral 资格检查
  void prefetchFastModeStatus()            // fast mode 状态
  void prefetchOfficialMcpUrls()           // 官方 MCP URL 列表
  void prefetchAwsCredentialsAndBedRockInfoIfSafe()
  void prefetchGcpCredentialsIfSafe()
  // ...
}
```

这些都是 fire-and-forget，不阻塞 REPL 启动。

---

## 关键模块依赖

```
main.tsx
    → src/entrypoints/init.js   (auth/config 核心初始化)
    → src/replLauncher.tsx      (REPL 启动)
    → src/QueryEngine.ts        (无头模式)
    → src/tools.ts              (getTools() 工具加载)
    → src/commands.ts           (getCommands() 命令加载)
    → src/services/mcp/         (getMcpToolsCommandsAndResources)
    → src/utils/auth.ts         (认证状态)
    → src/utils/config.ts       (全局配置)
    → src/services/analytics/   (遥测)
    → src/migrations/           (11 个迁移函数)
    → src/state/AppStateStore.ts (getDefaultAppState)
```

---

## 文件体积解释

4683 行的原因：
1. Commander.js 每个子命令都有 `.option()` 声明（类型安全要求逐一列举）
2. 默认命令的 `program.action()` 包含完整的路径分流逻辑（teleport/resume/worktree/teammate 等特殊模式）
3. `logTenguInit()` 单函数 96 行（大量遥测字段）
4. 大量 feature-gated 条件分支（`feature('COORDINATOR_MODE')` 等）

**重要**：`main.tsx` 是只读参考文件，修改任何子系统时不应从这里开始——找对应子系统的入口文件。