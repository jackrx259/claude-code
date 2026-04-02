# Feature Flags 体系

## 机制原理

Claude Code 使用 Bun 专有的 `bun:bundle` feature() API 在**编译期**注入 feature flags。
这意味着被禁用的功能代码会被**完全从 bundle 中剪掉**（dead code elimination），
而不是运行时判断。

```typescript
// build.ts 中定义
const flags = {
  VOICE_MODE: false,
  BRIDGE_MODE: false,
  TOKEN_BUDGET: true,
  // ...90+ 个
}

// 源码中使用（编译后被替换为字面量 true/false）
if (MACRO.VOICE_MODE) {
  // 这整块在外部版 bundle 中不存在
}
```

## 构建时常量（MACRO.*）

除了 feature flags，以下常量也在编译期注入：

| 常量 | 用途 |
|---|---|
| `MACRO.VERSION` | 版本号 |
| `MACRO.BUILD_TIME` | 构建时间戳 |
| `MACRO.ISSUES_EXPLAINER` | 错误报告提示文本 |

## 外部版启用的功能

| Flag | 功能说明 |
|---|---|
| `BUILTIN_EXPLORE_PLAN_AGENTS` | 内置 Explore / Plan 子 Agent |
| `TOKEN_BUDGET` | Token 预算管理 & 上下文窗口监控 |
| `MCP_SKILLS` | MCP Skills 支持 |

## 外部版禁用的功能（Anthropic 内部专有）

| Flag | 功能说明 | 推测原因 |
|---|---|---|
| `BRIDGE_MODE` | 桥接模式 | 内部 infra 依赖 |
| `KAIROS` | Kairos（推测是某调度系统） | 内部系统 |
| `DAEMON` | 后台守护进程模式 | 内部服务依赖 |
| `VOICE_MODE` | 语音输入 | 内部语音服务依赖 |
| `COORDINATOR_MODE` | 协调者模式 | 多 Agent 内部编排 |

## 关联文件

- Flag 定义：[`build.ts`](../../build.ts)（搜索 `feature(`）
- Stub 实现：[`stubs/`](../../stubs/)（被禁用功能依赖的私有包替代品）