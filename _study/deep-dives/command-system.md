# Command System — Slash 命令体系

> 核心文件：`src/commands.ts`，实现目录：`src/commands/`（105+ 子目录，207+ 文件）

## 什么是 Slash 命令

用户在输入框输入 `/` 开头的命令，如 `/commit`、`/review`、`/mcp`。
这类命令不走 QueryEngine，而是直接执行对应的命令逻辑
（或者构造特定 prompt 再走 QueryEngine）。

## 路由机制

```
用户输入 "/commit -m 'fix bug'"
    │
    ▼
REPL.tsx 检测到 "/" 前缀
    │
    ▼
commands.ts 路由匹配
    ├── 精确匹配命令名 "commit"
    └── 找到 src/commands/commit/ 目录中的实现
            │
            ▼
        执行命令逻辑
```

## 命令实现模式

每个命令目录包含：
- `index.ts` / `index.tsx` — 命令入口
- 可能有子命令（如 `/mcp add`、`/mcp remove`）
- 可以是纯逻辑（git 操作等），也可以触发 Claude 对话

## 重要命令列表（部分）

| 命令 | 功能 |
|---|---|
| `/commit` | 自动生成 git commit message 并提交 |
| `/review` | Code review |
| `/diff` | 查看 diff |
| `/mcp` | MCP server 管理（add/remove/list/status）|
| `/config` | 配置管理 |
| `/skills` | Skills 管理 |
| `/tasks` | 任务管理 |
| `/compact` | 手动触发对话压缩 |
| `/clear` | 清除对话历史 |
| `/help` | 帮助信息 |

## Skills vs 命令

Skills（`src/skills/`）是命令系统的扩展机制：
- 内置 Skills：编译进 bundle 的预设工作流
- MCP Skills：由外部 MCP server 提供（`MCP_SKILLS` flag）
- 用户通过 `/skill-name` 调用
- `SkillTool` 在 Claude 消息中调用 Skill

## 命令补全

输入 `/` 后，UI 会展示命令补全列表（`ContextSuggestions.tsx`），
列表来自注册的命令集合。

## 关键问题待研究

- [ ] `commands.ts` 中命令的注册机制
- [ ] 子命令的路由如何处理（`/mcp add` vs `/mcp remove`）
- [ ] 命令和工具的边界——何时用命令，何时用工具？

## 相关文件

- 命令路由：[`src/commands.ts`](../../src/commands.ts)
- 命令实现：[`src/commands/`](../../src/commands/)
- Skills：[`src/skills/`](../../src/skills/)
- 补全 UI：[`src/components/ContextSuggestions.tsx`](../../src/components/ContextSuggestions.tsx)
