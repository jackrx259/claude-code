# 认证体系 — Auth & OAuth

> 核心文件：`src/utils/`（auth/OAuth 相关，~65KB）

## 认证方式

Claude Code 支持多种认证方式：

| 方式 | 场景 |
|---|---|
| OAuth（claude.ai 账号） | 普通用户，通过浏览器授权 |
| API Key | 开发者直接使用 Anthropic API Key |
| MDM 配置 | 企业管理设备，IT 预配置 |
| AWS（Bedrock）| 企业通过 AWS Bedrock 访问 Claude |

## OAuth 流程

```
用户首次运行 claude
    │
    ▼
检查本地 keychain 有无 token
    │
    ├── 有且有效 → 直接使用
    └── 无/过期 ↓
            │
            ▼
        生成授权 URL
            │
            ▼
        打开浏览器（或显示 URL 让用户手动打开）
            │
            ▼
        用户在 claude.ai 授权
            │
            ▼
        回调到本地端口（或用户粘贴 code）
            │
            ▼
        换取 access_token + refresh_token
            │
            ▼
        存储到系统 keychain
```

## 控制台 OAuth 流程

`ConsoleOAuthFlow.tsx` 组件处理在终端内完成的 OAuth 交互（无浏览器场景）：
- 显示授权 URL
- 等待用户粘贴 authorization code
- 完成 token 交换

## Keychain 存储

认证 token 存储在系统 keychain（macOS Keychain、Windows Credential Store、Linux Secret Service），
不写入普通文件，提高安全性。

## MDM 支持

企业场景下，IT 通过 MDM（Mobile Device Management）预配置：
- API Key 或 OAuth 配置
- 策略约束（允许/禁止的功能）
- `src/utils/` 中的 MDM 工具解析这些配置

## AWS Bedrock 认证

`src/components/AwsAuthStatusBox.tsx` 提供 AWS 认证状态展示，
认证逻辑依赖 AWS SDK 的标准认证链（环境变量、~/.aws/credentials 等）。

## 关键问题待研究

- [ ] token refresh 的自动刷新机制
- [ ] 多账号切换支持？
- [ ] API Key 的存储方式（也是 keychain？还是 env var？）
- [ ] `ApproveApiKey.tsx` — API Key 确认流程

## 相关文件

- OAuth 工具（~65KB）：[`src/utils/`](../../src/utils/)（搜索 auth/oauth 文件名）
- OAuth UI：[`src/components/ConsoleOAuthFlow.tsx`](../../src/components/ConsoleOAuthFlow.tsx)
- AWS 状态：[`src/components/AwsAuthStatusBox.tsx`](../../src/components/AwsAuthStatusBox.tsx)
- API Key 确认：[`src/components/ApproveApiKey.tsx`](../../src/components/ApproveApiKey.tsx)
