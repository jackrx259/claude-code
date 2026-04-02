# Auth & OAuth — 认证体系深度分析

> 核心文件：`src/utils/auth.ts`（~2000 行）+ 相关 auth 工具文件
> 版本：v2.1.88

---

## 一、认证方式总览

Claude Code 支持 5 种认证方式，优先级从高到低：

| 优先级 | 方式 | 来源 | 场景 |
|---|---|---|---|
| 1 | `ANTHROPIC_AUTH_TOKEN` 环境变量 | 外部注入 | 代理/网关 token |
| 2 | `CLAUDE_CODE_OAUTH_TOKEN` 环境变量 | Claude Desktop/CCR 注入 | 托管会话 |
| 3 | `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | FD 管道 | CCR 子进程 |
| 4 | `apiKeyHelper`（脚本） | 用户配置 | 动态 key 获取 |
| 5 | Claude.ai OAuth tokens | macOS Keychain 或本地文件 | 普通用户（/login 流程）|
| 备选 | `ANTHROPIC_API_KEY` | 环境变量 | 开发者直接使用 |
| 备选 | AWS Bedrock / GCP Vertex | SDK 标准认证链 | 企业 3P 服务 |

---

## 二、认证模式判断（isAnthropicAuthEnabled）

```typescript
export function isAnthropicAuthEnabled(): boolean {
  // bare 模式 → 仅 API key，不走 OAuth
  if (isBareMode()) return false

  // SSH 远程 → 由代理注入，只看 CLAUDE_CODE_OAUTH_TOKEN
  if (process.env.ANTHROPIC_UNIX_SOCKET) return !!process.env.CLAUDE_CODE_OAUTH_TOKEN

  // 3P 服务（Bedrock/Vertex/Foundry）→ 禁用 Anthropic auth
  const is3P = isEnvTruthy(CLAUDE_CODE_USE_BEDROCK) || isEnvTruthy(CLAUDE_CODE_USE_VERTEX) || ...

  // 外部 API key / auth token（且非托管上下文）→ 禁用 OAuth
  const hasExternalAuth = ANTHROPIC_AUTH_TOKEN || apiKeyHelper || CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR
  const hasExternalApiKey = apiKeySource === 'ANTHROPIC_API_KEY' || 'apiKeyHelper'

  return !(is3P || hasExternalAuth || hasExternalApiKey)
}
```

**托管上下文（isManagedOAuthContext）：** `CLAUDE_CODE_REMOTE=true` 或 `CLAUDE_CODE_ENTRYPOINT=claude-desktop` 时为托管会话，不受外部 API key 影响（避免用户自己的 key 污染 CCR/Desktop 会话）。

---

## 三、Token 来源优先级（getAuthTokenSource）

```
getAuthTokenSource() 返回 { source, hasToken }

优先级顺序：
1. bare 模式 → apiKeyHelper only
2. ANTHROPIC_AUTH_TOKEN（非托管上下文）
3. CLAUDE_CODE_OAUTH_TOKEN（环境变量）
4. CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR（FD，CCR 管道）
5. CCR_OAUTH_TOKEN_FILE（CCR 子进程磁盘回退，无 FD）
6. apiKeyHelper（非托管上下文）
7. claude.ai OAuth tokens（shouldUseClaudeAIAuth + accessToken 有效）
8. none
```

---

## 四、API Key 管理

### 来源（getAnthropicApiKeyWithSource）

```
ApiKeySource: 'ANTHROPIC_API_KEY' | 'apiKeyHelper' | '/login managed key' | 'none'

优先级：
1. bare 模式：ANTHROPIC_API_KEY env → apiKeyHelper → none
2. CI/test 模式：CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR → ANTHROPIC_API_KEY → none
3. 普通模式：
   a. prefer3P 时检查 ANTHROPIC_API_KEY
   b. ANTHROPIC_API_KEY env（非 Homespace）
   c. apiKeyHelper（脚本动态获取）
   d. /login 管理的 key（macOS Keychain 或本地文件）
```

### apiKeyHelper — 动态 API Key 脚本

```typescript
// 用户在 settings.json 中配置
{ "apiKeyHelper": "/path/to/my-key-script.sh" }

// getApiKeyFromApiKeyHelper() 执行脚本，缓存结果
const DEFAULT_API_KEY_HELPER_TTL = 5 * 60 * 1000  // 5 分钟

// prefetchApiKeyFromApiKeyHelperIfSafe() 在启动时预取（不阻塞）
```

### Keychain 存储

- **macOS**：Keychain Services（`getMacOsKeychainStorageServiceName`）
- **通用**：`getSecureStorage()` → 跨平台安全存储
- API key 通过 `saveApiKey()` 存储，`getApiKeyFromConfigOrMacOSKeychain()` 读取（memoize 缓存）

---

## 五、OAuth 2.0 流程（Claude.ai 认证）

### 完整流程

```
用户首次运行 claude（或 /login）
    │
    ├── isAnthropicAuthEnabled() === true
    │
    ▼
src/services/oauth/client.ts
    │
    ├── 生成 OAuth 授权 URL（PKCE: code_challenge + code_verifier）
    ├── 打开浏览器（或显示 URL 让用户手动访问）
    │
    │   用户在 claude.ai 完成授权
    │
    ├── 监听本地回调端口 OR 用户粘贴 auth code
    ├── 用 code 换取 { accessToken, refreshToken, expiresAt }
    └── saveOAuthTokensIfNeeded(tokens) → 存入 Keychain
```

### ConsoleOAuthFlow（无浏览器场景）

`src/components/ConsoleOAuthFlow.tsx` 处理在终端内完成的 OAuth 交互：
1. 显示授权 URL（并尝试自动打开浏览器）
2. 等待用户粘贴 authorization code 或自动监听回调
3. 完成 token 交换

### Token 结构（OAuthTokens）

```typescript
type OAuthTokens = {
  accessToken: string
  refreshToken?: string
  expiresAt?: number      // Unix timestamp (ms)
  scopes?: string[]       // 如 ['claude.ai:profile', ...]
}
```

### Token 管理

```
getClaudeAIOAuthTokens()   ← memoize 缓存，读 Keychain/文件
    │
    ├── isOAuthTokenExpired(tokens)  ← 检查 expiresAt
    └── refreshOAuthToken(refreshToken)  ← 使用 refresh token 续期

checkAndRefreshOAuthTokenIfNeeded()
    ├── 若未过期 → 直接返回 accessToken
    └── 若过期 → 调用 refreshOAuthToken → 更新 Keychain

handleOAuth401Error()
    └── 遇到 401 → 尝试刷新 token → 若失败 → 触发重新登录提示
```

### 作用域（CLAUDE_AI_PROFILE_SCOPE）

```typescript
hasProfileScope()   ← 是否有 profile 作用域（用于获取用户信息）
isClaudeAISubscriber()  ← 是否为 Claude.ai 订阅用户
shouldUseClaudeAIAuth(scopes)  ← 是否应使用 Claude.ai 认证
```

---

## 六、订阅类型体系

认证后可通过 OAuth scopes/响应判断订阅类型：

```typescript
getSubscriptionType(): SubscriptionType | null
  // 'free' | 'pro' | 'pro_plus' | 'team' | 'team_premium' | 'enterprise' | 'max' | ...

isMaxSubscriber()         ← Max 计划
isTeamSubscriber()        ← Team 计划
isTeamPremiumSubscriber() ← Team Premium 计划
isEnterpriseSubscriber()  ← 企业计划
isProSubscriber()         ← Pro 计划
getRateLimitTier()        ← 速率限制等级
hasOpusAccess()           ← 是否可访问 Opus 模型
isOverageProvisioningAllowed()  ← 是否允许超额
```

---

## 七、AWS Bedrock 认证

```
CLAUDE_CODE_USE_BEDROCK=true
    │
    ▼
isUsing3PServices() === true → isAnthropicAuthEnabled() === false

refreshAndGetAwsCredentials()  ← memoize with TTL
    ├── isAwsAuthRefreshFromProjectSettings()
    │       → awsAuthRefresh 脚本（项目级配置）
    └── AWS SDK 标准认证链（环境变量、~/.aws/credentials、IAM role）

refreshAwsAuth(awsAuthRefresh)  ← 执行刷新脚本
clearAwsCredentialsCache()      ← 清除缓存（认证过期时）
prefetchAwsCredentialsAndBedRockInfoIfSafe()  ← 启动时预取
```

**AwsAuthStatusManager**（`awsAuthStatusManager.ts`）：

管理 AWS 认证状态（valid/invalid/refreshing），供 `AwsAuthStatusBox.tsx` UI 组件读取。

---

## 八、GCP Vertex 认证

```
CLAUDE_CODE_USE_VERTEX=true
    │
refreshGcpAuth(gcpAuthRefresh)     ← 执行刷新脚本
refreshGcpCredentialsIfNeeded()    ← memoize with TTL
checkGcpCredentialsValid()         ← 验证 GCP 凭据
prefetchGcpCredentialsIfSafe()     ← 启动时预取
```

---

## 九、MDM（企业移动设备管理）

`src/main.tsx` 启动时触发并行预取：

```typescript
startMdmRawRead()  // 在模块加载时立即启动，不阻塞
```

MDM 在不同平台的实现：
- **macOS**：`plutil -convert json /Library/Preferences/com.anthropic.claudecode.plist`
- **Windows**：`reg query HKLM\SOFTWARE\Anthropic\ClaudeCode`

`IS_LIBC_GLIBC` / `IS_LIBC_MUSL` feature flags 用于区分 Linux 平台，影响 MDM 读取方式。

MDM 可配置：
- API Key 或 OAuth 凭据
- 功能开关（禁止某些工具/命令）
- 企业代理设置

---

## 十、SSH 远程认证（`ANTHROPIC_UNIX_SOCKET`）

`claude ssh` 远程模式通过 Unix Socket 代理 API 调用：

```
远程 claude 进程
    │  ANTHROPIC_UNIX_SOCKET = /path/to/socket
    ▼
本地 sshAuthProxy（src/ssh/sshAuthProxy.ts）
    │  注入 Authorization header（本地用户的 OAuth token）
    ▼
Anthropic API
```

远程进程设置 `CLAUDE_CODE_OAUTH_TOKEN` 作为占位符（若本地是订阅者），让 API 包含 `oauth-2025` beta header 与代理匹配。

---

## 十一、安全机制

### ApproveApiKey（自定义 API Key 审批）

```typescript
isCustomApiKeyApproved(apiKey: string): boolean
    ← 检查 API key 是否在已批准列表中
    ← 首次使用新 key 时弹 ApproveApiKey.tsx 确认对话框
```

### normalizeApiKeyForConfig（key 规范化）

`authPortable.ts`：提取 API key 的有效部分，用于存储到 settings.json（非 Keychain）。

### lockfile

API key 保存时使用 `lockfile` 防止并发写入冲突。

---

## 十二、相关文件

| 文件 | 说明 |
|---|---|
| `src/utils/auth.ts` | 认证体系主文件（~2000 行，核心逻辑）|
| `src/utils/authFileDescriptor.ts` | FD（文件描述符）传递 API key/OAuth token |
| `src/utils/authPortable.ts` | API key 规范化、macOS Keychain 操作 |
| `src/utils/awsAuthStatusManager.ts` | AWS 认证状态管理 |
| `src/utils/sessionIngressAuth.ts` | 会话入口认证 |
| `src/services/oauth/client.ts` | OAuth 2.0 客户端（PKCE、token 刷新）|
| `src/services/oauth/types.ts` | OAuthTokens、SubscriptionType 类型 |
| `src/services/oauth/getOauthProfile.ts` | 获取 OAuth 用户 profile |
| `src/utils/secureStorage/` | 跨平台安全存储（Keychain/Secret Service）|
| `src/utils/secureStorage/keychainPrefetch.ts` | macOS Keychain 预取（启动优化）|
| `src/components/ConsoleOAuthFlow.tsx` | OAuth 终端交互 UI |
| `src/components/ApproveApiKey.tsx` | 自定义 API key 确认 |
| `src/components/AwsAuthStatusBox.tsx` | AWS 认证状态显示 |
