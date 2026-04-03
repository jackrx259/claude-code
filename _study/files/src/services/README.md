# src/services/ — 服务层目录概览

> 目录：[`src/services/`](../../../../../src/services/)（39 个子目录 + 若干顶层文件）  
> 角色：**跨切面基础设施层**，提供 API 通信、认证、分析、MCP、会话管理、语音等能力，供工具、组件、QueryEngine 调用。

---

## 子目录速览

### API 通信

| 子目录/文件 | 职责 |
|---|---|
| `api/claude.ts` | Claude API 核心客户端，发 `messages.stream()` 请求 |
| `api/client.ts` | Anthropic SDK 客户端实例化（含 retry、timeout 配置）|
| `api/withRetry.ts` | 指数退避重试逻辑（速率限制、网络错误）|
| `api/errorUtils.ts` | API 错误分类（rate limit / overload / auth error）|
| `api/filesApi.ts` | Files API（上传文件供 Claude 使用）|
| `api/usage.ts` | token 用量统计与聚合 |
| `api/grove.ts` | Grove（Anthropic 内部 LLM 网关）客户端（feature-gated）|
| `api/logging.ts` | API 请求/响应日志（调试用）|

### 会话压缩（Compact）

| 子目录/文件 | 职责 |
|---|---|
| `compact/compact.ts` | 触发对话压缩的主入口（判断何时 compact）|
| `compact/autoCompact.ts` | 基于 token 阈值自动触发 compact |
| `compact/microCompact.ts` | 微型 compact（保留最近 N 条消息）|
| `compact/prompt.ts` | Compact 用的 system prompt（指导 Claude 摘要）|
| `compact/grouping.ts` | 消息分组（将 tool_use + tool_result 配对保留）|
| `compact/snipCompact.ts` | HISTORY_SNIP 功能（feature-gated）|
| `compact/sessionMemoryCompact.ts` | 会话记忆压缩 |

### 认证（OAuth）

| 子目录/文件 | 职责 |
|---|---|
| `oauth/` | OAuth 2.0 PKCE 流程（见 deep-dives/auth-oauth.md）|

### MCP

| 子目录/文件 | 职责 |
|---|---|
| `mcp/` | MCP 服务器管理（见 deep-dives/mcp-system.md）|

### 分析与遥测

| 子目录/文件 | 职责 |
|---|---|
| `analytics/` | 事件上报（Statsig / GrowthBook）、feature flag 查询 |
| `diagnosticTracking.ts` | 诊断信息收集 |
| `internalLogging.ts` | 内部日志（不发给用户）|

### 会话与记忆

| 子目录/文件 | 职责 |
|---|---|
| `SessionMemory/` | 会话记忆持久化（跨对话保留上下文）|
| `AgentSummary/` | 子 Agent 完成后生成摘要 |
| `extractMemories/` | 从对话中提取记忆片段 |
| `teamMemorySync/` | 多 Agent 团队记忆同步 |
| `settingsSync/` | 设置跨设备同步 |

### 语音输入

| 子目录/文件 | 职责 |
|---|---|
| `voice.ts` | 语音录制入口（feature-gated: VOICE_MODE）|
| `voiceStreamSTT.ts` | 流式语音转文字 |
| `voiceKeyterms.ts` | 语音关键词检测（唤醒词）|

### 其他基础设施

| 子目录/文件 | 职责 |
|---|---|
| `plugins/` | 插件系统（加载用户自定义插件）|
| `lsp/` | LSP（Language Server Protocol）集成 |
| `tips/` | 使用技巧提示（首次启动等场景）|
| `notifier.ts` | OS 通知（macOS 通知中心、iTerm2 等）|
| `preventSleep.ts` | 防止系统休眠（长任务运行时）|
| `rateLimitMessages.ts` | 速率限制错误信息格式化 |
| `tokenEstimation.ts` | token 数量估算（不调 API，本地计算）|
| `toolUseSummary/` | 工具调用汇总显示逻辑 |
| `vcr.ts` | VCR 录制/重放（API 请求记录，测试用）|
| `remoteManagedSettings/` | 远程管理的设置（MDM 场景）|
| `policyLimits/` | 策略限制（企业合规）|
| `claudeAiLimits.ts` | claude.ai 平台限制检查 |
| `contextCollapse/` | 上下文折叠（长对话 UI 优化）|
| `autoDream/` | Dream 自动模式（实验性）|
| `awaySummary.ts` | 离开后的摘要生成 |
| `PromptSuggestion/` | 输入框提示建议 |
| `MagicDocs/` | 文档生成（实验性）|
| `mcpServerApproval.tsx` | MCP 服务器接入审批 UI |

---

## 关键服务详解

### `services/api/claude.ts` — API 核心

这是所有 Claude API 调用的唯一出口。主要函数：

```typescript
streamClaude(params) → AsyncIterator<RawMessageStreamEvent>
```

包含：prompt cache 处理、thinking mode 配置、token budget 注入、extended thinking headers。

### `services/compact/` — 对话压缩体系

当对话历史超过 context window 阈值时触发：

```
autoCompact.ts 监测 token 用量
    → compact.ts 决定压缩策略
        → prompt.ts 构造摘要指令
        → grouping.ts 保留必要消息段
        → Claude API 生成摘要
    → 替换历史消息，释放 context 空间
```

### `services/analytics/` — 分析体系

- **Statsig**：A/B 测试、feature gate
- **GrowthBook**：feature flag 查询（`getFeatureValue_CACHED_MAY_BE_STALE()`，有意允许缓存过期以避免延迟）
- **事件上报**：通过 `logEvent()` 记录工具调用、命令执行、错误等，匿名化后发送

---

## 依赖关系

```
services/
    ← QueryEngine.ts / query.ts（使用 API、compact 服务）
    ← tools/（工具调用 analytics、notifier、lsp）
    ← components/（使用 compact 状态、rate limit 消息）
    ← main.tsx（初始化 analytics、MCP、plugins、auth）
```
