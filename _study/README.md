# Claude Code 源码学习笔记

> 本目录是对 Claude Code 源码的系统性学习整理，不修改任何原始代码。

## 目录说明

| 目录 | 内容 |
|---|---|
| `architecture/` | 整体架构图、数据流、系统设计 |
| `deep-dives/` | 按核心系统专题深入分析 |
| `files/` | 镜像项目结构，逐文件/目录注解 |

## 学习路线（推荐顺序）

```
1. architecture/overview.md        ← 先建立整体认知
2. architecture/data-flow.md       ← 理解请求如何流转
3. deep-dives/query-engine.md      ← 核心：多轮 API 编排
4. deep-dives/tool-system.md       ← 核心：49 个工具实现
5. deep-dives/task-system.md       ← 任务状态机
6. deep-dives/terminal-ui.md       ← Ink/React 终端渲染
7. deep-dives/auth-oauth.md        ← 认证体系
8. files/src/                      ← 按需查阅具体文件
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

- [ ] architecture/overview.md
- [ ] architecture/data-flow.md
- [ ] architecture/feature-flags.md
- [ ] deep-dives/query-engine.md
- [ ] deep-dives/tool-system.md
- [ ] deep-dives/task-system.md
- [ ] deep-dives/terminal-ui.md
- [ ] deep-dives/auth-oauth.md
- [ ] deep-dives/mcp-system.md
- [ ] deep-dives/command-system.md
- [ ] files/src/main.tsx.md
- [ ] files/src/QueryEngine.md
- [ ] files/src/Task.md
- [ ] files/src/Tool.md
- [ ] files/src/tools/
- [ ] files/src/services/
- [ ] files/src/components/
- [ ] files/build/build.ts.md