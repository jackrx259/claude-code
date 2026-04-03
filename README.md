# Claude Code 源码学习计划

> 从 `@anthropic-ai/claude-code` v2.1.88 npm 包的 source map 中完整还原的 TypeScript 源代码，已配置为可本地编译运行。

这是一个面向所有热爱学习 AI 工具工程实践的开发者的开源学习项目。Claude Code 是目前最复杂、最完整的 AI 编程助手之一，其源码涵盖了终端 UI 渲染、多轮 LLM 编排、工具系统设计、MCP 协议、OAuth 认证、任务调度等众多工程课题。读懂它，你会对 AI Native 应用的架构设计有全新的认知。

---

## 学习路径

建议按以下顺序进行，从整体到局部，从宏观到微观：

### 第一阶段：建立整体认知

| 步骤 | 资料 | 目标 |
|---|---|---|
| 1 | [`_study/architecture/overview.md`](./_study/architecture/overview.md) | 理解四层架构、启动链、构建体系 |
| 2 | [`_study/architecture/data-flow.md`](./_study/architecture/data-flow.md) | 跟踪一次用户输入的完整生命周期 |
| 3 | [`_study/architecture/feature-flags.md`](./_study/architecture/feature-flags.md) | 理解编译期 feature flag 剪枝机制 |

### 第二阶段：攻克核心系统

| 步骤 | 资料 | 对应源码 |
|---|---|---|
| 4 | [`_study/deep-dives/query-engine.md`](./_study/deep-dives/query-engine.md) | `src/QueryEngine.ts` |
| 5 | [`_study/deep-dives/tool-system.md`](./_study/deep-dives/tool-system.md) | `src/Tool.ts` + `src/tools/` |
| 6 | [`_study/deep-dives/task-system.md`](./_study/deep-dives/task-system.md) | `src/Task.ts` + `src/tasks/` |
| 7 | [`_study/deep-dives/terminal-ui.md`](./_study/deep-dives/terminal-ui.md) | `src/ink/` + `src/components/` |
| 8 | [`_study/deep-dives/mcp-system.md`](./_study/deep-dives/mcp-system.md) | `src/services/mcp/` |
| 9 | [`_study/deep-dives/auth-oauth.md`](./_study/deep-dives/auth-oauth.md) | `src/utils/`（auth 部分）|
| 10 | [`_study/deep-dives/command-system.md`](./_study/deep-dives/command-system.md) | `src/commands.ts` + `src/commands/` |

### 第三阶段：逐文件精读

进入 [`_study/files/`](./_study/files/) 目录，按模块查阅具体文件的逐行分析笔记。
这一阶段是长期积累的过程，建议结合具体问题驱动阅读。

| 文件 | 内容 |
|---|---|
| [`files/src/README.md`](./_study/files/src/README.md) | `src/` 目录整体导览 |
| [`files/src/main.tsx/README.md`](./_study/files/src/main.tsx/README.md) | 4683 行全局初始化逐段拆解 |
| [`files/src/QueryEngine/README.md`](./_study/files/src/QueryEngine/README.md) | QueryEngine 类字段与方法注解 |
| [`files/src/Task/README.md`](./_study/files/src/Task/README.md) | Task 类型定义与状态机注解 |
| [`files/src/Tool/README.md`](./_study/files/src/Tool/README.md) | Tool 接口与 buildTool 工厂注解 |
| [`files/src/tools/README.md`](./_study/files/src/tools/README.md) | 44 个工具实现目录（含 BashTool、AgentTool 深入分析）|
| [`files/src/components/README.md`](./_study/files/src/components/README.md) | 146 个 React 终端 UI 组件 |
| [`files/src/screens/README.md`](./_study/files/src/screens/README.md) | REPL.tsx 主交互循环（5005 行）|
| [`files/src/hooks/README.md`](./_study/files/src/hooks/README.md) | 87 个 React hooks |
| [`files/src/services/README.md`](./_study/files/src/services/README.md) | 39 个服务子目录（API、MCP、认证、分析）|
| [`files/src/state/README.md`](./_study/files/src/state/README.md) | AppState 形状、Store 实现、selectors |
| [`files/src/entrypoints/README.md`](./_study/files/src/entrypoints/README.md) | CLI 入口与快速路径分流 |
| [`files/src/utils/README.md`](./_study/files/src/utils/README.md) | 330+ 工具文件按子目录分类概览 |
| [`files/build/build.ts.md`](./_study/files/build/build.ts.md) | 构建脚本与 90+ feature flags 详解 |

---

## Agent 开发工程模式

[`_study/agent-design-guide.md`](./_study/agent-design-guide.md) 从源码中提炼了构建生产级 agent 的核心工程模式，每个模式附 Python 示例。以下是主要内容索引：

### 会话与循环

**会话状态封装**：一个对话引擎实例对应一段对话的完整生命周期，跨多轮调用持续持有消息历史、token 累计用量和文件读取缓存。每轮共享同一实例，而非每轮重建。

**消息规范化**：每次 API 调用前对消息列表做规范化（去除 UI 专用字段、合并相邻同角色消息），而非只在首次调用时处理。原因：对话压缩等操作会在循环中途修改消息列表。

**循环终止条件**：除 `end_turn` 外，需要明确处理最大轮次、Token 预算上限、用户中断三种终止路径。

→ 详见：[`deep-dives/query-engine.md`](./_study/deep-dives/query-engine.md)

### 工具系统

**工具定义协议**：工具不只是可调用对象，还携带 `is_read_only`、`is_destructive`、`is_concurrency_safe`、`max_result_chars`、`system_prompt_fragment` 等元数据，供权限系统、调度器、系统提示词构建器分别使用。

**fail-closed 默认值**：所有安全相关布尔属性默认取保守一侧（`is_concurrency_safe=False`、`is_read_only=False`）。工具必须显式声明才能获得宽松权限。

**工具并发调度**：同批次工具调用中，任意一个声明不可并发则全部串行。全部声明安全时才并行执行。

**结果大小管理**：结果超过 `max_result_chars` 时写入磁盘临时文件，消息中只传路径和预览，防止单条结果消耗过多 context。

**工具延迟加载**：不常用工具不出现在初始 prompt 中，通过 ToolSearch 工具按需加载 schema，减少初始 token 消耗。

→ 详见：[`deep-dives/tool-system.md`](./_study/deep-dives/tool-system.md)

### Context Window 管理

**三级应对策略**：
1. Token 监控分级（ok / warning / danger / critical）
2. 对话压缩：超阈值时调用模型生成语义摘要，替换原始历史，压缩边界前的消息立即从内存释放
3. 工具结果预算：注入前截断单条过大结果

→ 详见：[`architecture/data-flow.md`](./_study/architecture/data-flow.md)

### 权限模型

**三层检查**：输入验证（工具自身）→ 通用权限检查（永久规则 + 模式决策）→ 工具级权限覆盖。

**权限模式**：`interactive`（危险操作需确认）/ `bypass`（无监督自动化）/ `accept_edits`（文件修改静默放行）/ `plan_only`（禁用所有写工具）。

**规则来源分离**：权限规则按来源（配置文件、命令、hooks）分组存储，支持按来源整体撤销，不影响其他来源。

→ 详见：[`deep-dives/tool-system.md`](./_study/deep-dives/tool-system.md)

### 多 Agent 与任务调度

**子 Agent 隔离**：子 Agent 持有独立的会话引擎实例，不能直接写入父 Agent 的全局状态（`setAppState` 为 no-op），只能通过返回值传递结果。

**任务状态机**：`pending → running → completed | failed | killed`，终态不可逆转。任务输出流式写入磁盘，调用方通过偏移量增量读取，实现执行与展示解耦。

**孤儿清理**：任务记录创建它的 Agent ID，父 Agent 退出时统一清理其遗留任务。

→ 详见：[`deep-dives/task-system.md`](./_study/deep-dives/task-system.md)

### 系统提示词与持久化

**动态组装**：系统提示词在每次请求前由多个来源合并：基础提示词 + 各工具的说明片段 + 按目录层级加载的配置文件 + MCP server 说明 + 调用方追加内容。

**预写日志**：用户消息在进入 API 循环**之前**写入磁盘，确保进程崩溃后仍可从 transcript 文件恢复会话。

**启动并行预取**：凭据读取、MCP 连接建立、配置加载等慢操作在主流程开始时立即并行触发，在真正需要时才 await，消除串行等待的启动延迟。

→ 详见：[`deep-dives/auth-oauth.md`](./_study/deep-dives/auth-oauth.md)、[`deep-dives/mcp-system.md`](./_study/deep-dives/mcp-system.md)

### 完整 Python 示例

[**→ Agent 设计精要（含 Python 示例）**](./_study/agent-design-guide.md)

---

## 未来蓝图

这个学习计划正在持续建设中，欢迎任何人提交 PR 补充笔记、纠正错误、增加图示。

计划中的内容包括：

- **流程图**：用 Mermaid 绘制 QueryEngine 多轮循环、Tool 权限检查、OAuth 流程等关键流程
- **对比分析**：Claude Code 与 Cursor、Aider、Continue 等工具的架构差异
- **专题研究**：
  - `src/ink/`：Anthropic 对 Ink 做了哪些深度定制？为什么要 fork？
  - `src/main.tsx`：4600 行的超级初始化文件，逐段拆解
  - Feature flags：90+ 个编译开关背后，哪些内部功能被隐藏了？
  - Context compaction：超长任务不中断的秘密
  - Worktree 隔离：Agent 如何在沙箱中安全执行写操作
- **实验记录**：修改 feature flags 后重新构建，观察行为变化
- **问题清单**：整理所有「待研究」问题，作为社区探索的方向

**如果你在阅读中发现了有价值的东西，欢迎在 Issues 中分享，或直接在 `_study/` 目录下提交你的笔记。**

---

## 快速开始（本地构建）

```bash
# 1. 安装 Bun（macOS arm64）
curl -LO https://github.com/oven-sh/bun/releases/latest/download/bun-darwin-aarch64.zip
unzip bun-darwin-aarch64.zip -d /tmp/bun && sudo cp /tmp/bun/bun-darwin-aarch64/bun /usr/local/bin/bun

# 2. 安装依赖（node_modules 已包含，此步可选）
pnpm install --registry https://registry.npmjs.org

# 3. 构建
bun run build.ts

# 4. 验证
bun dist/cli.js --version
```

**验证输出：**
```
$ bun dist/cli.js --version
2.1.88 (Claude Code)

$ bun dist/cli.js --help
Usage: claude [options] [command] [prompt]
Claude Code - starts an interactive session by default...
```

---

## 源码是如何还原的

npm 包 `@anthropic-ai/claude-code` 发布时附带了完整的 `cli.js.map` source map 文件，其中包含所有原始 TypeScript/TSX 的 `sourcesContent`。通过解析这个 source map，可以无损还原全部 4756 个源文件。

```bash
# 1. 下载 npm 包
npm pack @anthropic-ai/claude-code --registry https://registry.npmjs.org

# 2. 解压
tar xzf anthropic-ai-claude-code-2.1.88.tgz

# 3. 从 source map 写出所有源文件
node -e "
const fs = require('fs'), path = require('path');
const map = JSON.parse(fs.readFileSync('package/cli.js.map', 'utf8'));
const outDir = './claude-code-source';
for (let i = 0; i < map.sources.length; i++) {
  const content = map.sourcesContent[i];
  if (!content) continue;
  let relPath = map.sources[i];
  while (relPath.startsWith('../')) relPath = relPath.slice(3);
  const outPath = path.join(outDir, relPath);
  fs.mkdirSync(path.dirname(outPath), { recursive: true });
  fs.writeFileSync(outPath, content);
}
"
```

---

## 构建细节

### 依赖环境

| 工具 | 版本 | 用途 |
|---|---|---|
| [Bun](https://bun.sh) | v1.3.11 | 构建工具（必须，使用了 `bun:bundle` 专有特性）|
| [pnpm](https://pnpm.io) | v10+ | 包管理 |
| Node.js | v18+ | 运行时 |

### 为什么必须用 Bun 构建？

源码使用了 Bun bundler 专有的 `feature()` API，实现**编译期死代码消除**：

```typescript
import { feature } from 'bun:bundle'

// 编译时替换为 true/false，并消除对应分支
const coordinatorModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null
```

这等价于 webpack 的 `DefinePlugin`，但与 `build.ts` 中 90+ 个特性开关深度绑定，无法用其他打包器替代。

### MACRO 常量注入

```typescript
// 源码中
console.log(`${MACRO.VERSION} (Claude Code)`)

// build.ts 中注入
define: {
  'MACRO.VERSION': JSON.stringify('2.1.88'),
  'MACRO.BUILD_TIME': JSON.stringify(new Date().toISOString()),
  'MACRO.ISSUES_EXPLAINER': JSON.stringify('...'),
}
```

### 私有包存根（stubs/）

以下 Anthropic 内部包不在公开 npm 中，已在 `stubs/` 目录创建功能存根以使构建通过：

| 包名 | 存根策略 |
|---|---|
| `color-diff-napi` | 禁用颜色差异高亮（返回空实现）|
| `modifiers-napi` | 禁用 macOS 按键修饰符检测（返回空）|
| `@ant/claude-for-chrome-mcp` | Chrome 扩展 MCP 服务器（空实现）|
| `@anthropic-ai/mcpb` | MCP bundle 处理器（用标准 MCP 替代）|
| `@anthropic-ai/sandbox-runtime` | 沙盒运行时（空实现）|

### commander 补丁

源码使用 `-d2e` 形式的多字符短选项，而 commander v14 只允许单字符短选项。`patches/` 目录中有一个最小化补丁，将 option 解析正则从 `/^-[^-]$/` 改为 `/^-[^-]+$/`。

---

## 目录结构

```
.
├── _study/               # 学习笔记（本项目核心产出）
│   ├── README.md         # 学习地图 & 进度追踪
│   ├── architecture/     # 整体架构、数据流、feature flags 完整分析
│   ├── deep-dives/       # 核心系统专题深入分析
│   └── files/            # 逐文件注解，镜像 src/ 目录结构
├── src/                  # 还原的核心源码（1,908 个文件）
│   ├── entrypoints/      # CLI 入口（12 文件）
│   ├── main.tsx          # 全局初始化（4,683 行）
│   ├── QueryEngine.ts    # 多轮 API 编排引擎
│   ├── Tool.ts           # 工具系统基础定义
│   ├── Task.ts           # 任务系统基础定义
│   ├── commands/         # Slash 命令实现（191 文件）
│   ├── components/       # 终端 UI 组件（390 文件）
│   ├── hooks/            # React hooks（104 文件）
│   ├── ink/              # 自研终端渲染引擎（98 文件）
│   ├── services/         # 核心服务（133 文件）
│   ├── tools/            # 工具实现（190 文件）
│   └── utils/            # 工具函数（566 文件）
├── stubs/                # 私有包存根
├── vendor/               # 内部 vendor 代码（4 文件）
├── patches/              # pnpm 补丁（commander 多字符短选项）
├── build.ts              # Bun 构建脚本（含 90+ feature flags）
├── dist/cli.js           # 构建产出（22MB 单文件 ESM）
└── package.json
```

---

## 关键统计

| 指标 | 数值 |
|---|---|
| 包版本 | 2.1.88 |
| 源文件总数 | 4,756 |
| 核心源码（src/ + vendor/） | 1,912 文件 |
| Source Map 大小 | 57 MB |
| 构建产物大小 | 22 MB |
| Feature flags 数量 | 90+ |
| 工具实现数量 | 49 |
| React UI 组件数量 | 146 |
| Slash 命令数量 | 105+ |
