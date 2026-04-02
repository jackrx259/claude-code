# Command System — Slash 命令体系深度分析

> 核心文件：`src/commands.ts`（754 行）+ `src/commands/`（103 个子目录，207+ 文件）
> 类型定义：`src/types/command.ts`
> 版本：v2.1.88

---

## 一、什么是 Slash 命令

用户在输入框输入 `/` 开头的命令，如 `/commit`、`/mcp`、`/config`。

命令有两种执行路径：
1. **本地执行**（`local`/`local-jsx`）：直接执行逻辑，不走 Claude API
2. **Prompt 注入**（`prompt`）：将命令内容作为 prompt 注入对话，走 QueryEngine

---

## 二、Command 类型系统

### 三种命令类型（`src/types/command.ts`）

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

#### PromptCommand（Prompt 注入型）

```typescript
type PromptCommand = {
  type: 'prompt'
  progressMessage: string              // 执行期间显示的进度文字
  contentLength: number                // 内容字符数（用于 token 估算）
  argNames?: string[]                  // 参数名列表
  allowedTools?: string[]              // 限制可用工具（白名单）
  model?: string                       // 覆盖模型
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  disableNonInteractive?: boolean
  hooks?: HooksSettings                // 调用时注册的 hooks
  skillRoot?: string                   // Skill 资源根目录（CLAUDE_PLUGIN_ROOT 环境变量）
  context?: 'inline' | 'fork'         // inline=注入当前对话, fork=独立子 Agent
  agent?: string                       // fork 时使用的 Agent 类型
  effort?: EffortValue                 // 努力程度设置
  paths?: string[]                     // 文件路径 glob（仅当模型接触匹配文件时可见）
  getPromptForCommand(args, context): Promise<ContentBlockParam[]>  // 获取注入 prompt
}
```

#### LocalCommand（本地执行型，懒加载）

```typescript
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<LocalCommandModule>  // 懒加载（避免大模块提前加载）
}

type LocalCommandModule = {
  call: (args: string, context: LocalJSXCommandContext) => Promise<LocalCommandResult>
}

// 返回结果类型
type LocalCommandResult =
  | { type: 'text'; value: string }
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
  | { type: 'skip' }
```

#### LocalJSXCommand（React UI 型，懒加载）

```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>  // 懒加载
}

type LocalJSXCommandCall = (
  onDone: LocalJSXCommandOnDone,
  context: ToolUseContext & LocalJSXCommandContext,
  args: string,
) => Promise<React.ReactNode>

// onDone 回调
type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: 'skip' | 'system' | 'user'  // 结果显示方式
    shouldQuery?: boolean      // 完成后是否触发 API 调用
    metaMessages?: string[]    // 插入 isMeta 消息（模型可见但 UI 不显示）
    nextInput?: string         // 自动填充下一条输入
    submitNextInput?: boolean  // 自动提交
  },
) => void
```

### CommandBase — 所有命令共享基础字段

```typescript
type CommandBase = {
  name: string
  aliases?: string[]
  description: string
  hasUserSpecifiedDescription?: boolean
  isEnabled?: () => boolean      // 运行时检查（feature flags、env）
  isHidden?: boolean             // 隐藏于 typeahead/help
  availability?: CommandAvailability[]  // 认证/provider 限制
  argumentHint?: string          // 参数提示（灰色显示）
  whenToUse?: string             // Skill 专用：详细使用场景
  version?: string
  disableModelInvocation?: boolean  // 禁止模型调用（只允许用户手动调用）
  userInvocable?: boolean           // 用户可输入 /name 调用
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  kind?: 'workflow'              // workflow 类型（autocomplete 显示徽章）
  immediate?: boolean            // 立即执行（不等队列停止点）
  isSensitive?: boolean          // 参数从对话历史中隐藏
  isMcp?: boolean
  userFacingName?: () => string  // 覆盖显示名（如插件前缀剥离）
}
```

---

## 三、命令注册机制

### 内置命令（COMMANDS — memoize 函数）

`commands.ts` 中所有内置命令通过静态 import 加载，组合到 `COMMANDS()` 数组。memoize 确保只加载一次（配置读取发生在 `COMMANDS()` 首次调用时，而非模块加载时）：

```typescript
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, btw, chrome, clear, color, compact, config,
  copy, desktop, context, cost, diff, doctor, effort, exit, fast, files,
  heapDump, help, ide, init, keybindings, mcp, memory, mobile, model, ...
  // feature flag 条件命令
  ...(workflowsCmd ? [workflowsCmd] : []),    // WORKFLOW_SCRIPTS
  ...(voiceCommand ? [voiceCommand] : []),     // VOICE_MODE
  ...(proactive ? [proactive] : []),           // PROACTIVE
  ...(bridge ? [bridge] : []),                 // BRIDGE_MODE
  ...(ultraplan ? [ultraplan] : []),           // ULTRAPLAN
  // 仅 ant 内部
  ...(process.env.USER_TYPE === 'ant' ? INTERNAL_ONLY_COMMANDS : []),
])
```

### INTERNAL_ONLY_COMMANDS — Ant 内部命令

```typescript
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, initVerifiers, mockLimits,
  bridgeKick, version, resetLimits, onboarding, share, summary,
  teleport, antTrace, perfIssue, env, oauthRefresh, debugToolCall,
  agentsPlatform, autofixPr,
].filter(Boolean)
```

这些命令只在 `USER_TYPE=ant` 且非 demo 环境时才加入（外部用户看不到）。

---

## 四、命令加载流程

```
getCommands(cwd)
    │
    ├── loadAllCommands(cwd)  [memoize by cwd]
    │   ├── getSkills(cwd)  [并行]
    │   │   ├── getSkillDirCommands(cwd)  ← .claude/skills/ 目录下的文件
    │   │   ├── getPluginSkills()         ← 插件提供的 skills
    │   │   ├── getBundledSkills()        ← 编译进 bundle 的内置 skills
    │   │   └── getBuiltinPluginSkillCommands()  ← 内置插件的 skills
    │   ├── getPluginCommands()           ← 插件提供的命令
    │   └── getWorkflowCommands(cwd)     ← 工作流命令（WORKFLOW_SCRIPTS flag）
    │
    ├── getDynamicSkills()  ← 模型文件操作期间动态发现的 skills
    │
    ├── 过滤：meetsAvailabilityRequirement() + isCommandEnabled()
    │
    └── 动态 skills 去重 + 插入正确位置
```

### 加载顺序（优先级从低到高）

```
COMMANDS()（内置）< pluginSkills < bundledSkills < builtinPluginSkills < skillDirCommands < workflowCommands < pluginCommands
```

动态 skills 插入在 pluginSkills 之后、内置命令之前。

---

## 五、可用性控制（availability + isEnabled）

两层独立过滤：

```
availability: ['claude-ai', 'console']   ← 静态：认证/provider 要求
    ├── 'claude-ai'  → isClaudeAISubscriber()（OAuth 订阅用户）
    └── 'console'    → !isClaudeAISubscriber() && !is3P && isFirstPartyAnthropicBaseUrl()

isEnabled(): boolean                     ← 动态：feature flags、env、GrowthBook
    └── 默认 true（未定义时）
```

注意：`availability` 在 `meetsAvailabilityRequirement()` 中独立检查，不是 `isEnabled()` 的一部分，因为 auth 状态可能在会话中变化（如 `/login` 后立即生效）。

---

## 六、Skills 体系

Skills 是 `prompt` 类型命令的子集，专为模型调用设计：

### Skills 来源

| 来源 | loadedFrom | 目录/位置 |
|---|---|---|
| 用户 Skill 目录 | `'skills'` | `.claude/skills/` 或自定义 Skill 目录 |
| 插件提供 | `'plugin'` | 插件 manifest 中声明 |
| 内置打包 | `'bundled'` | `src/skills/bundledSkills.ts`（编译进 bundle）|
| 内置插件 | `'plugin'`（builtin）| `src/plugins/builtinPlugins.ts` |
| MCP Skills | `'mcp'` | MCP server prompts（`MCP_SKILLS` flag）|
| 工作流 | `kind='workflow'` | `WORKFLOW_SCRIPTS` flag |

### Skills 过滤（getSlashCommandToolSkills）

```typescript
// 仅 SkillTool 可调用（非 builtin source，有描述/whenToUse）
allCommands.filter(
  cmd =>
    cmd.type === 'prompt' &&
    cmd.source !== 'builtin' &&
    (cmd.hasUserSpecifiedDescription || cmd.whenToUse) &&
    (loadedFrom === 'skills' || 'plugin' || 'bundled' || cmd.disableModelInvocation),
)
```

### 动态 Skill 发现（paths 字段）

```typescript
// Skill 声明 paths: ['src/**/*.tsx']
// 只有当模型读取了匹配路径的文件后，才会动态加入命令列表
```

`getDynamicSkills()` 返回当前会话中已触发的动态 skills（通过 `dynamicSkillDirTriggers` set 跟踪）。

---

## 七、命令执行流程

```
用户输入 "/commit -m 'fix bug'"
    │
    ▼
REPL.tsx / processUserInput() 检测 "/" 前缀
    │
    ▼
在 getCommands(cwd) 列表中匹配命令名
    │
    ├── type === 'local'
    │       → cmd.load() 懒加载 → call(args, context)
    │       → 返回 LocalCommandResult
    │
    ├── type === 'local-jsx'
    │       → cmd.load() 懒加载 → call(onDone, context, args)
    │       → 返回 React.ReactNode（渲染为 UI）
    │       → onDone() 触发后续操作
    │
    └── type === 'prompt'
            → cmd.getPromptForCommand(args, context)
            → 返回 ContentBlockParam[]（注入为用户消息）
            → shouldQuery = true → 进入 QueryEngine 循环
```

---

## 八、重要命令一览

### 外部版公开命令（部分）

| 命令 | 类型 | 功能 |
|---|---|---|
| `/add-dir` | local | 添加额外工作目录 |
| `/clear` | local | 清除对话历史 |
| `/compact` | local-jsx | 手动触发对话压缩 |
| `/config` | local-jsx | 配置管理（UI 界面）|
| `/context` | local/prompt | 查看/设置 context |
| `/cost` | local | 显示当前会话费用 |
| `/diff` | prompt | 查看 git diff |
| `/doctor` | local-jsx | 诊断工具（检查环境）|
| `/fast` | local | 切换 Fast Mode |
| `/help` | local-jsx | 帮助信息 |
| `/hooks` | local-jsx | 管理 Hooks 配置 |
| `/ide` | local-jsx | IDE 扩展管理 |
| `/init` | prompt | 初始化项目（生成 CLAUDE.md）|
| `/login` | local-jsx | OAuth 登录 |
| `/logout` | local | 登出 |
| `/mcp` | local-jsx | MCP 服务器管理 |
| `/memory` | local-jsx | 内存管理（CLAUDE.md）|
| `/model` | local-jsx | 切换模型 |
| `/permissions` | local-jsx | 权限规则管理 |
| `/plan` | local | 切换 Plan Mode |
| `/plugin` | local-jsx | 插件管理 |
| `/pr_comments` | prompt | 查看 PR 评论 |
| `/resume` | local-jsx | 恢复历史会话 |
| `/review` | prompt | Code review |
| `/session` | local-jsx | 会话管理 |
| `/skills` | local-jsx | Skills 管理 |
| `/status` | local | 状态概览 |
| `/tasks` | local-jsx | 任务管理 |
| `/theme` | local-jsx | 主题设置 |
| `/usage` | local | 使用量统计 |
| `/vim` | local | 切换 Vim 模式 |

### 仅 Ant 内部命令（INTERNAL_ONLY_COMMANDS）

`commit`、`commit-push-pr`、`bughunter`、`autofix-pr`、`backfill-sessions`、`ctx_viz`、`debug-tool-call` 等 20+ 个内部工具，仅在 `USER_TYPE=ant` 时加载。

---

## 九、缓存管理

```typescript
clearCommandMemoizationCaches()  ← 只清 memoize 缓存（动态 skill 添加时）
clearCommandsCache()              ← 完整清除（包括插件、skills）

// 注意：getSkillIndex（技能搜索索引）是独立的 memoize 层，
// 必须用 clearSkillIndexCache() 单独清除
```

---

## 十、命令与 Skills 的关系

```
所有 prompt 类 Command
    │
    ├── source === 'builtin'     → 只供用户调用，模型不可调用
    │                              (getSkillToolCommands 过滤)
    │
    └── source !== 'builtin'    → 可被 SkillTool 调用（模型发起）
        ├── disableModelInvocation=true  → 用户 /name 专用
        └── hasUserSpecifiedDescription or whenToUse  → 出现在 SkillTool 列表
```

`insights`（使用报告）命令是懒加载的特殊案例：模块 113KB（3200 行），通过 shim 对象推迟 `import('./commands/insights.js')` 到实际调用时。

---

## 十一、相关文件

| 文件 | 说明 |
|---|---|
| `src/commands.ts` | 命令注册、getCommands、getSlashCommandToolSkills |
| `src/types/command.ts` | Command/PromptCommand/LocalCommand/LocalJSXCommand 类型 |
| `src/commands/` | 103 个命令实现目录 |
| `src/skills/bundledSkills.ts` | 编译进 bundle 的内置 skills |
| `src/skills/loadSkillsDir.ts` | 从 .claude/skills/ 加载 skill 文件 |
| `src/utils/plugins/loadPluginCommands.ts` | 插件命令加载 |
| `src/plugins/builtinPlugins.ts` | 内置插件 skills |
| `src/tools/SkillTool/` | 模型调用 skill 的工具实现 |
