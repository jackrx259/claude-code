# build.ts — 构建脚本

> 文件：[`build.ts`](../../../build.ts)（项目根目录）

## 职责

这是整个项目的**唯一构建入口**，运行 `bun run build.ts` 产出 `dist/cli.js`。

## 关键输出

| 属性 | 值 |
|---|---|
| 产物路径 | `dist/cli.js` |
| 文件大小 | ~22MB |
| 格式 | 单文件 ESM bundle |
| 可执行性 | 有 shebang，chmod 755 |

## 核心机制：`bun:bundle` feature()

Bun 专有 API，允许在打包时注入布尔常量，编译器用这些常量做**死代码消除**：

```typescript
// build.ts 中
const result = await Bun.build({
  entrypoints: ['src/entrypoints/cli.tsx'],
  define: {
    'MACRO.VERSION': JSON.stringify(version),
    'MACRO.VOICE_MODE': 'false',
    'MACRO.TOKEN_BUDGET': 'true',
    // ...90+ 个 feature flags
  }
})
```

## MACRO.* 常量注入

| 常量 | 作用 |
|---|---|
| `MACRO.VERSION` | 版本号（从 package.json 读取）|
| `MACRO.BUILD_TIME` | 构建时间戳 |
| `MACRO.ISSUES_EXPLAINER` | 错误时显示的反馈链接文本 |

## Feature Flags 完整状态

详见 → [`../../architecture/feature-flags.md`](../../architecture/feature-flags.md)

## Stubs 处理

构建时将 `stubs/` 目录中的 stub 包映射到对应的私有包路径：
- `color-diff-napi` → stub（返回空实现）
- `modifiers-napi` → stub（返回空实现）
- `@anthropic-ai/mcpb` → stub（标准 MCP 替代）

## 关键问题待研究

- [ ] `bun:bundle` 的 feature() API 具体用法
- [ ] 构建时如何处理 TSX（React）— JSX 转换配置
- [ ] `vendor/` 目录如何被打包进来
- [ ] 是否有 source map 生成？
