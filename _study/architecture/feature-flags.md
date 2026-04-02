# Feature Flags 体系

> 对应源码版本：v2.1.88，flag 数据来自 `build.ts`

## 机制原理

源码中通过 `import { feature } from 'bun:bundle'` 使用 feature flags：

```typescript
// 源码使用方式
const coordinatorModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null
```

**在还原版的 build.ts 中，`bun:bundle` 被一个自定义 Bun 插件拦截**，生成一个字典查找函数：

```typescript
// build.ts 中的 bun-bundle-feature-shim 插件实际生成的代码：
const features = {
  'COORDINATOR_MODE': false,
  'TOKEN_BUDGET': true,
  // ...所有 flag
};
export function feature(name) {
  if (name in features) return features[name];
  return false;
}
```

Bun bundler 在打包时会内联这些 `feature()` 调用并消除死分支，效果等同于编译期常量替换。

## 完整 Flag 列表（90 个）

### 外部版已启用（true）

| Flag | 功能说明 |
|---|---|
| `BUILTIN_EXPLORE_PLAN_AGENTS` | 内置 Explore / Plan 子 Agent |
| `COMPACTION_REMINDERS` | 对话压缩提醒 |
| `MCP_SKILLS` | MCP Skills 支持 |
| `TOKEN_BUDGET` | Token 预算管理 & 上下文窗口监控 |

### 外部版禁用（false）—— 分类说明

#### Anthropic 内部基础设施（不对外开放）

| Flag | 功能推测 |
|---|---|
| `BRIDGE_MODE` | IDE 桥接模式（连接 Cursor/VSCode 等 IDE 进程）|
| `COORDINATOR_MODE` | 多 Agent 协调者模式 |
| `DAEMON` | 后台守护进程模式 |
| `KAIROS` | 助手模式（Kairos 是内部代号）|
| `KAIROS_BRIEF` | Kairos 简报功能 |
| `KAIROS_CHANNELS` | Kairos 频道功能 |
| `KAIROS_DREAM` | Kairos Dream（异步规划）|
| `KAIROS_GITHUB_WEBHOOKS` | Kairos GitHub Webhook 集成 |
| `KAIROS_PUSH_NOTIFICATION` | Kairos 推送通知 |
| `LODESTONE` | 未知（内部系统代号）|
| `CHICAGO_MCP` | Computer Use MCP server（内部代号 Chicago）|
| `DIRECT_CONNECT` | 直连服务器模式 |
| `SSH_REMOTE` | SSH 远程执行 |
| `UDS_INBOX` | Unix Domain Socket 收件箱 |
| `CCR_AUTO_CONNECT` | CCR（Cloud Code Runner）自动连接 |
| `CCR_MIRROR` | CCR 镜像 |
| `CCR_REMOTE_SETUP` | CCR 远程配置 |
| `BYOC_ENVIRONMENT_RUNNER` | BYOC（Bring Your Own Cloud）环境运行器 |
| `SELF_HOSTED_RUNNER` | 自托管运行器 |

#### 实验性功能（未发布）

| Flag | 功能推测 |
|---|---|
| `VOICE_MODE` | 语音输入 |
| `WEB_BROWSER_TOOL` | 网页浏览工具 |
| `ULTRAPLAN` | 超级规划模式 |
| `ULTRATHINK` | 超级思考模式 |
| `PROACTIVE` | 主动建议功能 |
| `TEMPLATES` | 模板系统 |
| `QUICK_SEARCH` | 快速搜索 |
| `MESSAGE_ACTIONS` | 消息操作（右键菜单等）|
| `HISTORY_PICKER` | 历史对话选择器 |
| `TERMINAL_PANEL` | 终端面板（IDE 集成）|
| `MONITOR_TOOL` | 监控工具 |
| `NATIVE_CLIPBOARD_IMAGE` | 原生剪贴板图像支持 |
| `WORKFLOW_SCRIPTS` | 工作流脚本 |
| `REVIEW_ARTIFACT` | 制品审查 |
| `BUILDING_CLAUDE_APPS` | 构建 Claude 应用引导 |

#### 分析 & 遥测（内部数据收集）

| Flag | 功能推测 |
|---|---|
| `ENHANCED_TELEMETRY_BETA` | 增强遥测（Beta）|
| `MEMORY_SHAPE_TELEMETRY` | 记忆形态遥测 |
| `COWORKER_TYPE_TELEMETRY` | 协作者类型遥测 |
| `SHOT_STATS` | 请求统计 |
| `SLOW_OPERATION_LOGGING` | 慢操作日志 |
| `PERFETTO_TRACING` | Perfetto 性能追踪 |
| `TRANSCRIPT_CLASSIFIER` | 对话分类器 |
| `COMMIT_ATTRIBUTION` | 提交归因追踪 |
| `ANTI_DISTILLATION_CC` | 反蒸馏（防止数据被用于训练竞品）|

#### 实验性 Agent / 内存功能

| Flag | 功能推测 |
|---|---|
| `AGENT_MEMORY_SNAPSHOT` | Agent 记忆快照 |
| `AGENT_TRIGGERS` | Agent 触发器 |
| `AGENT_TRIGGERS_REMOTE` | 远程 Agent 触发器 |
| `EXTRACT_MEMORIES` | 自动提取记忆 |
| `TEAMMEM` | 团队记忆共享 |
| `FILE_PERSISTENCE` | 文件持久化 |
| `FORK_SUBAGENT` | Fork 子 Agent |
| `BUDDY` | Buddy 子 Agent 系统 |
| `VERIFICATION_AGENT` | 验证 Agent |
| `ABLATION_BASELINE` | 消融实验基线（Harness 科学测试）|

#### 模型 & 压缩相关

| Flag | 功能推测 |
|---|---|
| `REACTIVE_COMPACT` | 响应式压缩（上下文动态压缩）|
| `CACHED_MICROCOMPACT` | 缓存微压缩 |
| `CONTEXT_COLLAPSE` | 上下文折叠 |
| `HISTORY_SNIP` | 历史片段裁剪 |
| `BASH_CLASSIFIER` | Bash 命令分类器 |
| `TREE_SITTER_BASH` | Tree-sitter Bash 解析 |
| `TREE_SITTER_BASH_SHADOW` | Tree-sitter Bash 影子模式 |
| `PROMPT_CACHE_BREAK_DETECTION` | Prompt Cache 断点检测 |
| `DUMP_SYSTEM_PROMPT` | 输出系统提示词（内部评估工具）|

#### 设置 & 配置同步

| Flag | 功能推测 |
|---|---|
| `DOWNLOAD_USER_SETTINGS` | 从云端下载用户设置 |
| `UPLOAD_USER_SETTINGS` | 上传用户设置到云端 |
| `MDM` 相关（通过 `IS_LIBC_GLIBC`/`IS_LIBC_MUSL`）| libc 平台检测 |
| `BREAK_CACHE_COMMAND` | 手动清除 prompt cache 命令 |

#### 其他

| Flag | 功能推测 |
|---|---|
| `AUTO_THEME` | 自动主题（跟随系统）|
| `AWAY_SUMMARY` | 离开时的摘要 |
| `BG_SESSIONS` | 后台会话 |
| `CONNECTOR_TEXT` | 连接器文本 |
| `EXPERIMENTAL_SKILL_SEARCH` | 实验性 Skill 搜索 |
| `HARD_FAIL` | 硬错误模式（不重试）|
| `HOOK_PROMPTS` | Hook 提示词 |
| `MCP_RICH_OUTPUT` | MCP 富文本输出 |
| `NATIVE_CLIENT_ATTESTATION` | 原生客户端证明 |
| `NEW_INIT` | 新初始化流程 |
| `OVERFLOW_TEST_TOOL` | 溢出测试工具 |
| `POWERSHELL_AUTO_MODE` | PowerShell 自动模式 |
| `RUN_SKILL_GENERATOR` | Skill 生成器 |
| `SKILL_IMPROVEMENT` | Skill 改进建议 |
| `STREAMLINED_OUTPUT` | 精简输出模式 |
| `TORCH` | 未知（内部代号）|
| `UNATTENDED_RETRY` | 无人值守重试 |
| `ALLOW_TEST_VERSIONS` | 允许测试版本 |
| `ABLATION_BASELINE` | 消融实验基线 |

## 如何修改 Flag 进行实验

修改 `build.ts` 中 `featureFlags` 对象，改变对应值后重新构建：

```bash
# 修改 build.ts 中某个 flag，例如启用 VOICE_MODE
# VOICE_MODE: true,   ← 改为 true

bun run build.ts
bun dist/cli.js
```

**注意**：某些 flag 启用后可能因依赖 Anthropic 内部服务而无法正常工作。

## 与 GrowthBook 的关系

部分功能除了 build-time flag 外，还有 **运行时 GrowthBook feature gate**（A/B 测试平台）：

```typescript
// src/services/analytics/growthbook.ts
const isFastModeEnabled = await getFeatureValue_CACHED_MAY_BE_STALE('fast_mode')
```

GrowthBook gate 独立于 `feature()` flag：
- `feature()` flag 控制**代码是否编译进 bundle**（死代码消除）
- GrowthBook 控制**已编译的功能是否对当前用户开启**（运行时开关）

两者可以组合使用：flag 为 true（代码存在）+ GrowthBook 为 false（当前用户不可见）。

## 相关文件

- Flag 定义：[`build.ts`](../../build.ts)（`featureFlags` 对象，第 10-100 行）
- Shim 插件：[`build.ts`](../../build.ts)（`bun-bundle-feature-shim` 插件，第 128-151 行）
- GrowthBook：[`src/services/analytics/growthbook.ts`](../../src/services/analytics/growthbook.ts)
- 运行时 flag 使用示例：`src/main.tsx`（搜索 `feature('`）
