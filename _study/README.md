# Claude Code 源码学习笔记

> 本目录是对 Claude Code 源码的系统性学习整理，不修改任何原始代码。

## 目录说明

| 目录 | 内容 |
|---|---|
| `architecture/` | 整体架构、数据流、feature flags 完整分析 |
| `deep-dives/` | 核心系统专题深入分析 |
| `files/` | 逐文件注解，镜像 src/ 目录结构 |

## 学习路线（推荐顺序）

```
1. architecture/overview.md        ← 先建立整体认知（四层架构、启动链、构建体系）
2. architecture/data-flow.md       ← 一次用户输入的完整生命周期
3. architecture/feature-flags.md   ← 90 个编译期 feature flag 完整分析
4. deep-dives/query-engine.md      ← 核心：多轮 API 编排（QueryEngine 类）
5. deep-dives/tool-system.md       ← 核心：49 个工具实现与权限模型
6. deep-dives/task-system.md       ← 任务状态机（7 种任务类型）
7. deep-dives/terminal-ui.md       ← Ink/React 终端渲染体系
8. deep-dives/mcp-system.md        ← MCP 协议集成
9. deep-dives/auth-oauth.md        ← 认证体系（OAuth、API Key、MDM、AWS）
10. deep-dives/command-system.md   ← Slash 命令体系
11. files/src/                     ← 按需查阅具体文件注解
```

## 项目关键数字（快速定位）

| 指标 | 数值 |
|---|---|
| 构建产物 | `dist/cli.js`，~22MB 单文件 ESM |
| 工具数量 | 49 个 Tool 实现 |
| 组件数量 | 146 个 React 组件 |
| Hooks 数量 | 87 个 React hooks |
| 工具文件 | 330+ 个 utils |
| 命令数量 | 105+ 个 slash commands，207+ 文件 |
| 功能标志 | ~90+ 个 build-time feature flags |

## 进度追踪

- [x] architecture/overview.md
- [x] architecture/data-flow.md
- [x] architecture/feature-flags.md
- [x] deep-dives/query-engine.md
- [x] deep-dives/tool-system.md
- [x] deep-dives/task-system.md
- [x] deep-dives/terminal-ui.md
- [x] deep-dives/auth-oauth.md
- [x] deep-dives/mcp-system.md
- [ ] deep-dives/command-system.md
- [ ] files/src/main.tsx.md
- [ ] files/src/QueryEngine.md
- [ ] files/src/Task.md
- [ ] files/src/Tool.md
- [ ] files/src/tools/
- [ ] files/src/services/
- [ ] files/src/components/
- [ ] files/build/build.ts.md