# 架构总览

> 对应源码版本：v2.1.88

## 整体分层

```
┌─────────────────────────────────────────────────────────────┐
│                      用户界面层                               │
│  src/ink/           自定义终端渲染器（Ink fork，96 文件）      │
│  src/components/    146 个 React UI 组件                     │
│  src/hooks/         87 个 React hooks                        │
│  src/keybindings/   快捷键注册系统                            │
│  src/vim/           Vim 模式                                 │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                      核心编排层                               │
│  src/entrypoints/cli.tsx    CLI 入口 & 快速路径分流           │
│  src/main.tsx               全局初始化（4,683 行）            │
│  src/replLauncher.tsx       REPL 启动                        │
│  src/components/REPL.tsx    主交互循环                        │
│  src/QueryEngine.ts         多轮 API 编排（class QueryEngine）│
│  src/query.ts               底层 query 函数（QueryEngine 调用）│
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    工具 & 任务 & 命令层                        │
│  src/Tool.ts + src/tools/     49 个工具实现                  │
│  src/Task.ts + src/tasks/     任务状态机                     │
│  src/commands.ts + src/commands/   105+ slash 命令           │
│  src/skills/                  Skills（预设工作流）            │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                   服务 & 基础设施层                            │
│  src/services/    39 子目录（API、MCP、认证、分析、LSP…）      │
│  src/utils/       330+ 工具文件（auth 65KB、git、settings…）  │
│  src/state/       全局 AppState（React Context + store）      │
│  src/bootstrap/   进程启动状态（state.js 存全局单例）          │
└─────────────────────────────────────────────────────────────┘
```

## 启动链（精确版）

```
src/entrypoints/cli.tsx
│
├── 顶层副作用（模块加载即执行）：
│   ├── process.env.COREPACK_ENABLE_AUTO_PIN = '0'   ← 禁止 corepack 自动 pin
│   └── CLAUDE_CODE_REMOTE=true 时设 --max-old-space-size=8192
│
├── 快速路径（零/极少模块加载，按 if/else 分流）：
│   ├── --version / -v / -V        → 打印版本直接退出（零 import）
│   ├── --dump-system-prompt       → (DUMP_SYSTEM_PROMPT flag，内部) 输出系统提示词退出
│   ├── --claude-in-chrome-mcp     → 启动 Chrome MCP server 模式
│   ├── --chrome-native-host       → 启动 Chrome Native Host 模式
│   ├── --computer-use-mcp         → (CHICAGO_MCP flag) 启动 Computer Use MCP server
│   ├── --daemon-worker            → (DAEMON flag，内部) 启动 daemon worker 子进程
│   └── remote-control/rc/bridge   → (BRIDGE_MODE flag，内部) 启动桥接环境
│
└── 完整路径（普通交互模式）→ 动态 import main.tsx
        │
        ▼
src/main.tsx（4,683 行）
│
├── 模块加载时立即触发的并行预取（副作用，在所有 import 之前）：
│   ├── profileCheckpoint('main_tsx_entry')   ← 启动性能打点
│   ├── startMdmRawRead()                     ← 并行触发 MDM 子进程（plutil/reg query）
│   └── startKeychainPrefetch()               ← 并行触发 macOS keychain 读取（~65ms）
│
├── 初始化序列（按顺序）：
│   ├── Commander CLI 参数解析
│   ├── config / settings 加载
│   ├── 认证 & OAuth 检查
│   ├── GrowthBook（feature gate）初始化
│   ├── MCP server 连接建立
│   ├── 工具列表构建（getTools()）
│   ├── 命令注册（getCommands()）
│   ├── Analytics / telemetry 初始化
│   ├── 数据迁移（migrations/）
│   └── LSP server 管理器初始化
│
└── → launchRepl() → src/replLauncher.tsx → src/components/REPL.tsx
```

## QueryEngine 类（核心）

`src/QueryEngine.ts` 定义了 `class QueryEngine`，是整个系统最核心的类：

```typescript
class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]          // 完整对话历史
  private abortController: AbortController   // 中断控制
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage        // 累计 token 用量
  private readFileState: FileStateCache       // 文件读取缓存（LRU）
  private discoveredSkillNames: Set<string>  // 本轮发现的 Skill 名（遥测用）
  private loadedNestedMemoryPaths: Set<string>  // 已加载的 CLAUDE.md 路径（去重）

  submitMessage(...)  // 每次用户输入触发一个新"turn"
}
```

**一个 QueryEngine 实例对应一段对话**，`submitMessage()` 在同一实例上多次调用维持状态。

## 构建体系

| 项目 | 说明 |
|---|---|
| Bundler | Bun，通过自定义插件 shim `bun:bundle` 的 `feature()` API |
| 包管理 | pnpm |
| 产物 | `dist/cli.js`，单文件 ESM，带 shebang，chmod 755 |
| Feature flags | 90 个布尔常量（见下表），通过 `define` 在编译期注入 |
| `.md` / `.txt` 文件 | 通过 `text-file-loader` 插件作为字符串导入 |
| `.d.ts` 文件 | 编译为空模块（`export {}`）|
| 外部依赖 | `*.node`、`sharp`、`@img/*` 不打包（external）|

### `bun:bundle` feature() 的实际实现

这个版本的 `build.ts` **并非真正使用 Bun 原生的 feature() API**，而是通过一个自定义 Bun 插件 shim：

```typescript
// build.ts 中的插件
{
  name: 'bun-bundle-feature-shim',
  setup(build) {
    build.onResolve({ filter: /^bun:bundle$/ }, () => ({
      path: 'bun:bundle',
      namespace: 'bun-bundle-shim',
    }))
    build.onLoad({ filter: /.*/, namespace: 'bun-bundle-shim' }, () => {
      // 将所有 featureFlags 对象生成为 JS 代码
      // feature('FLAG_NAME') 返回对应的 true/false
      const lines = Object.entries(featureFlags)
        .map(([k, v]) => `  '${k}': ${v},`)
        .join('\n')
      return { contents: `...feature()实现...`, loader: 'js' }
    })
  }
}
```

这意味着 `feature()` 在打包后是一个字典查找，**并非编译期常量替换**。但由于 Bun 的 bundler 会内联常量，实际效果相同。

## MACRO.* 常量（build.ts 注入）

| 常量 | 值/来源 |
|---|---|
| `MACRO.VERSION` | `'2.1.88'` |
| `MACRO.BUILD_TIME` | `new Date().toISOString()` |
| `MACRO.ISSUES_EXPLAINER` | GitHub Issues URL 文本 |
| `MACRO.FEEDBACK_CHANNEL` | GitHub Issues URL |
| `MACRO.PACKAGE_URL` | npmjs.com 包页面 URL |
| `MACRO.NATIVE_PACKAGE_URL` | npmjs.com 包页面 URL |
| `MACRO.VERSION_CHANGELOG` | `''`（空）|

## 关键文件尺寸参考

| 文件 | 行数 | 说明 |
|---|---|---|
| `src/main.tsx` | 4,683 | 最大单文件，全局初始化 |
| `src/QueryEngine.ts` | 1,295 | 核心编排类 |
| `src/Tool.ts` | 792 | 工具类型系统（纯类型，无实现）|
| `src/Task.ts` | 125 | 任务抽象定义 |
| `src/entrypoints/cli.tsx` | 302 | CLI 入口 |
| `build.ts` | 194 | 构建脚本 |

## 相关笔记

- 数据流：[`data-flow.md`](./data-flow.md)
- Feature flags 完整表：[`feature-flags.md`](./feature-flags.md)
- QueryEngine 深入：[`../deep-dives/query-engine.md`](../deep-dives/query-engine.md)
