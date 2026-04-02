# 架构总览

## 整体分层

```
┌─────────────────────────────────────────────────────┐
│                   用户界面层                          │
│  src/ink/          自定义 React 终端渲染器 (96 文件)  │
│  src/components/   146 个 React UI 组件              │
│  src/hooks/        87 个 React hooks                 │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│                   核心编排层                          │
│  src/entrypoints/cli.tsx   CLI 入口 & 快速路径        │
│  src/main.tsx              全局初始化 (4600+ 行)      │
│  src/replLauncher.tsx      启动交互式 REPL            │
│  src/components/REPL.tsx   主交互循环                 │
│  src/QueryEngine.ts        多轮 API 编排引擎           │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│                   工具 & 任务层                        │
│  src/Tool.ts + src/tools/   49 个工具实现             │
│  src/Task.ts + src/tasks/   任务状态机                │
│  src/commands.ts + src/commands/  105+ slash 命令    │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│                   服务 & 基础设施层                    │
│  src/services/   39 个子目录（API、MCP、认证、分析…）  │
│  src/utils/      330+ 工具文件                        │
│  src/state/      全局 AppState (React Context)       │
└─────────────────────────────────────────────────────┘
```

## 启动链

```
src/entrypoints/cli.tsx
    ├── 快速路径：--version / --daemon-worker / --bridge (直接返回)
    └── 完整路径 ↓
src/main.tsx                    (config, auth, MCP, analytics, tools, commands 全部在这里初始化)
    └── src/replLauncher.tsx    (启动 REPL 进程)
            └── src/components/REPL.tsx   (主循环：读输入 → 调 QueryEngine → 渲染输出)
```

## 构建体系

| 项目 | 说明 |
|---|---|
| Bundler | Bun（非 webpack/esbuild），用 `bun:bundle` 实现编译期 feature flag 剪枝 |
| 包管理 | pnpm |
| 产物 | `dist/cli.js`，单文件 ESM，shebang + chmod 755 |
| Feature flags | ~90+ 布尔常量，通过 `MACRO.*` 在编译期注入，控制死代码消除 |

### 关键 feature flags 状态

| Flag | 外部版状态 |
|---|---|
| `BUILTIN_EXPLORE_PLAN_AGENTS` | ✅ 启用 |
| `TOKEN_BUDGET` | ✅ 启用 |
| `MCP_SKILLS` | ✅ 启用 |
| `BRIDGE_MODE` | ❌ 禁用 |
| `KAIROS` | ❌ 禁用 |
| `DAEMON` | ❌ 禁用 |
| `VOICE_MODE` | ❌ 禁用 |
| `COORDINATOR_MODE` | ❌ 禁用 |

## 私有/Native 包处理

`stubs/` 目录存放了 Anthropic 内部私有包的 stub 实现：
- `color-diff-napi` — 颜色差异计算（原生模块）
- `modifiers-napi` — 修饰键处理（原生模块）
- `@anthropic-ai/mcpb` — MCP 内部实现

这是外部可编译版本的关键：用 stub 替换掉了依赖 native binding 的私有包。

## 相关文件

- 启动入口：[`src/entrypoints/cli.tsx`](../../src/entrypoints/cli.tsx)
- 核心初始化：[`src/main.tsx`](../../src/main.tsx)
- 构建脚本：[`build.ts`](../../build.ts)
- 更多细节 → [`data-flow.md`](./data-flow.md)