# QueryEngine — 核心编排引擎

> 文件：`src/QueryEngine.ts`

## 职责

QueryEngine 是 Claude Code 的"大脑调度中心"，负责：

1. **多轮 API 调用循环**：发请求 → 处理工具调用 → 注入结果 → 再发请求，直到 `end_turn`
2. **消息历史管理**：维护完整的 `messages` 数组
3. **Context Window 管理**：监控 token 用量，触发对话压缩（compaction）
4. **Token Budget 控制**：（`TOKEN_BUDGET` flag 启用时）限制单次任务的 token 消耗
5. **Thinking Mode**：支持 Claude 的 extended thinking 模式
6. **工具执行调度**：串行/并行分发工具调用，收集结果

## 核心循环伪代码

```typescript
async function* query(userMessage, options) {
  messages.push({ role: 'user', content: userMessage })
  
  while (true) {
    const response = await claudeAPI.messages.create({
      model,
      messages,
      tools: getTools(),
      // ...
    })
    
    yield response  // 流式吐出给 REPL 渲染
    
    if (response.stop_reason === 'end_turn') break
    
    if (response.stop_reason === 'tool_use') {
      const toolResults = await executeTools(response.content)
      messages.push({ role: 'assistant', content: response.content })
      messages.push({ role: 'user', content: toolResults })
      // 继续循环
    }
    
    if (tokenBudgetExceeded()) compact()
  }
}
```

## 对话压缩（Compaction）

当 context window 接近上限时，QueryEngine 会：
1. 调用 `src/services/compact/` 的压缩逻辑
2. 用 Claude 自己生成一段"当前对话摘要"
3. 将历史消息替换为摘要，释放 token 空间
4. 继续后续对话

这是让 Claude Code 能处理超长任务而不中断的关键机制。

## Thinking Mode

当启用 extended thinking 时，response 中会包含 `thinking` 类型的 block，
QueryEngine 会将其传递给 UI 渲染（显示为可折叠的思考过程）。

## Token Budget（`TOKEN_BUDGET` flag）

- 在 `build.ts` 中为 `true`（外部版启用）
- 对每个 Agent 任务设定 token 上限
- 超限时中断任务并通知用户

## 关键问题待研究

- [ ] 工具调用是串行还是并行？（看 `executeTools` 实现）
- [ ] compaction 触发阈值是多少？（看 `src/services/compact/`）
- [ ] thinking mode 的 token 计费方式
- [ ] 多个子 Agent 的 context 是否共享？

## 相关文件

- 实现：[`src/QueryEngine.ts`](../../src/QueryEngine.ts)
- 压缩服务：[`src/services/compact/`](../../src/services/compact/)
- query 工具函数：[`src/query.ts`](../../src/query.ts) / [`src/query/`](../../src/query/)
- Token 计算：[`src/cost-tracker.ts`](../../src/cost-tracker.ts)