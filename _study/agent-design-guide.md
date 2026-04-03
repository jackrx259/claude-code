# Agent 设计精要：从 Claude Code 源码提炼的工程模式

> 本文从 Claude Code（v2.1.88）源码中提炼可复用的架构决策与工程模式，  
> 附 Python 示例说明每项模式的实现方法。

---

## 一、会话状态管理：对话引擎的封装边界

模型 API 是无状态的：每次请求必须携带完整的消息历史。因此，"状态"完全由调用方维护。

Claude Code 的做法是将一段对话的全部可变状态封装在 `QueryEngine` 类中，跨多次 `submitMessage()` 调用保持存活：
- 完整消息历史（`mutableMessages`）
- 累计 token 用量
- 权限拒绝记录
- 文件读取 LRU 缓存
- 已加载的 CLAUDE.md 路径（去重用）

一个实例对应一段对话，不同对话不共享实例。`ask()` 是对它的一次性包装，供单轮无状态场景使用。

```python
import anthropic
import json
from dataclasses import dataclass, field
from typing import Any

@dataclass
class ConversationEngine:
    """封装一段对话的全部可变状态。一个实例对应一段对话生命周期。"""
    system_prompt: str
    tools: list[dict]
    messages: list[dict] = field(default_factory=list)
    total_input_tokens: int = 0
    total_output_tokens: int = 0

    def submit(self, user_content: str) -> str:
        """提交用户消息，执行完整的工具调用循环，返回最终文字回复。"""
        self.messages.append({"role": "user", "content": user_content})
        result = _run_loop(self)
        return result

def _run_loop(engine: ConversationEngine) -> str:
    client = anthropic.Anthropic()
    while True:
        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=4096,
            system=engine.system_prompt,
            tools=engine.tools,
            messages=engine.messages,
        )
        engine.total_input_tokens += response.usage.input_tokens
        engine.total_output_tokens += response.usage.output_tokens

        engine.messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason == "end_turn":
            # 提取最终文字回复
            for block in response.content:
                if block.type == "text":
                    return block.text
            return ""

        # 处理工具调用，注入结果后继续循环
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = dispatch_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result,
                })
        engine.messages.append({"role": "user", "content": tool_results})


def dispatch_tool(name: str, inputs: dict) -> str:
    # 实际项目中替换为真实的工具路由
    return f"[{name} 执行结果]"
```

---

## 二、主循环：消息规范化是每轮的第一步

每次进入 API 调用前，Claude Code 都会重新规范化消息列表（`normalizeMessagesForAPI`）。原因：消息列表可能被外部操作修改（如对话压缩替换了历史），直接使用未经规范化的列表可能导致 API 请求格式非法。

规范化的职责包括：
- 移除仅用于 UI 渲染的字段（不发给 API）
- 合并相邻的同角色消息（部分模型要求）
- 确保 `tool_result` 消息紧跟 `tool_use` 消息

```python
def normalize_messages_for_api(messages: list[dict]) -> list[dict]:
    """
    在每次 API 调用前对消息列表做规范化处理。
    去除 UI 专用字段，合并相邻同角色消息。
    """
    cleaned = []
    for msg in messages:
        # 去除不应发送给 API 的内部字段
        api_msg = {k: v for k, v in msg.items() if k not in ("_ui_only", "_source")}
        cleaned.append(api_msg)

    # 合并相邻的同角色消息（避免某些模型的格式错误）
    merged = []
    for msg in cleaned:
        if merged and merged[-1]["role"] == msg["role"]:
            # 将 content 合并为列表形式
            prev = merged[-1]
            prev_content = prev["content"] if isinstance(prev["content"], list) else [{"type": "text", "text": prev["content"]}]
            curr_content = msg["content"] if isinstance(msg["content"], list) else [{"type": "text", "text": msg["content"]}]
            merged[-1] = {"role": msg["role"], "content": prev_content + curr_content}
        else:
            merged.append(msg)
    return merged


def run_loop_with_normalization(engine: ConversationEngine) -> str:
    client = anthropic.Anthropic()
    while True:
        # 每轮调用前规范化，而不是只在第一次
        normalized = normalize_messages_for_api(engine.messages)

        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=4096,
            system=engine.system_prompt,
            tools=engine.tools,
            messages=normalized,
        )
        # ... 后续处理同上
```

---

## 三、工具定义协议：工具不只是一个可调用对象

Claude Code 的工具不是简单的函数，而是一个携带完整元数据的对象，回答以下问题：

| 字段 | 类型 | 用途 |
|---|---|---|
| `name` | `str` | 模型调用时使用的标识符 |
| `description` | `str` | 模型据此判断是否调用此工具 |
| `input_schema` | JSON Schema | 模型据此生成调用参数 |
| `is_read_only` | `bool` | 权限系统判断是否允许在只读模式执行 |
| `is_destructive` | `bool` | 是否不可逆（删除、覆写、发送等） |
| `is_concurrency_safe` | `bool` | 同批次是否可并行执行 |
| `max_result_chars` | `int` | 结果超限时的处理阈值 |
| `execute(args, ctx)` | callable | 实际执行逻辑 |
| `system_prompt_fragment` | `str` | 该工具希望注入系统提示词的说明文本 |

**fail-closed 原则**：所有布尔属性的默认值取保守一侧。`is_concurrency_safe` 默认 `False`（串行），`is_read_only` 默认 `False`（视为写操作）。工具必须显式声明才能获得宽松权限，而非显式声明才受限制。

```python
from dataclasses import dataclass, field
from typing import Callable, Any
import subprocess

@dataclass
class Tool:
    name: str
    description: str
    input_schema: dict
    execute: Callable[[dict, dict], str]
    system_prompt_fragment: str = ""
    is_read_only: bool = False        # fail-closed: 默认视为写操作
    is_destructive: bool = False
    is_concurrency_safe: bool = False  # fail-closed: 默认串行
    max_result_chars: int = 100_000

    def to_api_schema(self) -> dict:
        """转换为 Anthropic API 要求的工具定义格式。"""
        return {
            "name": self.name,
            "description": self.description,
            "input_schema": self.input_schema,
        }


# 只读工具：显式声明并发安全
read_file_tool = Tool(
    name="read_file",
    description="读取指定路径的文件内容",
    input_schema={
        "type": "object",
        "properties": {
            "path": {"type": "string", "description": "文件绝对路径"},
            "start_line": {"type": "integer"},
            "end_line": {"type": "integer"},
        },
        "required": ["path"],
    },
    is_read_only=True,
    is_concurrency_safe=True,   # 读操作并发安全
    max_result_chars=200_000,   # 文件读取不做磁盘持久化，本身有行范围限制
    execute=lambda args, ctx: open(args["path"]).read(),
    system_prompt_fragment="使用 read_file 读取文件内容，支持通过 start_line/end_line 指定行范围。",
)

# 写操作工具：保持 fail-closed 默认值
write_file_tool = Tool(
    name="write_file",
    description="将内容写入指定路径的文件（覆写）",
    input_schema={
        "type": "object",
        "properties": {
            "path": {"type": "string"},
            "content": {"type": "string"},
        },
        "required": ["path", "content"],
    },
    is_destructive=True,  # 覆写是不可逆操作
    # is_read_only 和 is_concurrency_safe 保持默认 False
    execute=lambda args, ctx: (open(args["path"], "w").write(args["content"]), "OK")[1],
)
```

---

## 四、工具并发执行：保守的调度策略

当模型在单次回复中返回多个 `tool_use` 块时，调度策略决定是串行还是并行执行。

Claude Code 的规则：**只要有任意一个工具声明 `is_concurrency_safe=False`，本批次全部串行执行**。全部声明安全时才并行。

这是一个保守策略，可能损失一定并行性能，但完全规避了并发执行带来的竞态条件风险（如两个写操作同时修改同一文件）。

```python
import asyncio
from typing import Callable

async def execute_tool_calls(
    tool_calls: list[dict],
    tool_registry: dict[str, Tool],
    context: dict,
) -> list[dict]:
    """
    根据工具的并发安全声明决定执行策略。
    任意一个工具不安全 → 全部串行。
    """
    tools = [tool_registry[call["name"]] for call in tool_calls]
    all_safe = all(t.is_concurrency_safe for t in tools)

    results = []
    if all_safe:
        # 全部并行执行
        tasks = [
            asyncio.to_thread(t.execute, call["input"], context)
            for t, call in zip(tools, tool_calls)
        ]
        outputs = await asyncio.gather(*tasks)
        for call, output in zip(tool_calls, outputs):
            results.append({
                "type": "tool_result",
                "tool_use_id": call["id"],
                "content": output,
            })
    else:
        # 串行执行
        for t, call in zip(tools, tool_calls):
            output = t.execute(call["input"], context)
            results.append({
                "type": "tool_result",
                "tool_use_id": call["id"],
                "content": output,
            })
    return results
```

---

## 五、工具结果大小管理：防止单条结果撑爆上下文

工具结果直接注入消息列表时，过大的结果会迅速消耗 context window。Claude Code 的处理方式：

- 结果大小 ≤ `max_result_chars`：直接注入
- 结果大小 > `max_result_chars`：写入磁盘临时文件，消息中只传文件路径和前 N 个字符的预览

模型收到路径后，可以使用文件读取工具按需读取，而不是被动接收全量内容。

```python
import os
import tempfile

MAX_INLINE_CHARS = 50_000  # 超过此大小的结果写入磁盘

def apply_result_budget(
    tool_name: str,
    tool_use_id: str,
    raw_result: str,
    max_chars: int = MAX_INLINE_CHARS,
) -> dict:
    """
    若结果超过大小预算，持久化到磁盘并返回文件路径。
    否则直接内联返回。
    """
    if len(raw_result) <= max_chars:
        return {
            "type": "tool_result",
            "tool_use_id": tool_use_id,
            "content": raw_result,
        }

    # 写入临时文件
    tmp = tempfile.NamedTemporaryFile(
        mode="w",
        suffix=f"_{tool_name}.txt",
        delete=False,
        encoding="utf-8",
    )
    tmp.write(raw_result)
    tmp.close()

    preview = raw_result[:500]
    message = (
        f"结果过大（{len(raw_result):,} 字符），已写入文件：{tmp.name}\n"
        f"前 500 字符预览：\n{preview}\n...\n"
        f"请使用 read_file 工具读取完整内容。"
    )
    return {
        "type": "tool_result",
        "tool_use_id": tool_use_id,
        "content": message,
    }
```

---

## 六、工具延迟加载：控制初始 Prompt 的 Token 消耗

工具越多，发给模型的工具定义占用的 token 越多，即使这些工具本轮根本不会被调用。

Claude Code 的解决方案：引入 `ToolSearch` 工具作为索引层。不常用的工具标记为 `should_defer=True`，不出现在初始请求中。模型需要时先调用 `ToolSearch` 搜索工具，得到完整 schema 后才能调用。

```python
import json

# 工具注册表：分为"始终加载"和"延迟加载"两类
ALWAYS_LOAD_TOOLS: list[Tool] = []   # 每次请求都携带
DEFERRED_TOOLS: list[Tool] = []      # 仅在 ToolSearch 后才暴露

def build_tool_search_tool(deferred: list[Tool]) -> dict:
    """构造 ToolSearch 工具的 API 定义。"""
    return {
        "name": "tool_search",
        "description": "搜索可用的工具。当你需要某个功能但不确定工具名时，先搜索再调用。",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "搜索关键词"},
            },
            "required": ["query"],
        },
    }

def handle_tool_search(query: str, deferred: list[Tool]) -> str:
    """执行工具搜索，返回匹配工具的完整 schema。"""
    query_lower = query.lower()
    matches = [
        t for t in deferred
        if query_lower in t.name.lower() or query_lower in t.description.lower()
    ]
    if not matches:
        return "未找到匹配的工具。"
    return json.dumps(
        [t.to_api_schema() for t in matches],
        ensure_ascii=False,
        indent=2,
    )

def get_initial_tools(session_tools: list[Tool]) -> list[dict]:
    """
    构建初始请求的工具列表：始终加载的工具 + ToolSearch（如有延迟加载工具）。
    """
    schemas = [t.to_api_schema() for t in session_tools if not getattr(t, "should_defer", False)]
    deferred = [t for t in session_tools if getattr(t, "should_defer", False)]
    if deferred:
        schemas.append(build_tool_search_tool(deferred))
    return schemas
```

---

## 七、Context Window 管理：三级应对策略

随对话轮次增加，消息历史长度持续增长，最终逼近模型的 context window 上限。Claude Code 采用三级策略：

### 第一级：Token 监控与分级状态

每次 API 响应后，根据 token 使用量计算警告等级：

```python
from enum import Enum

class TokenWarningState(Enum):
    OK = "ok"
    WARNING = "warning"   # 提示用户
    DANGER = "danger"     # 明确警告
    CRITICAL = "critical" # 触发自动压缩

def calculate_token_warning_state(
    input_tokens: int,
    context_window: int = 200_000,
) -> TokenWarningState:
    ratio = input_tokens / context_window
    if ratio < 0.7:
        return TokenWarningState.OK
    elif ratio < 0.85:
        return TokenWarningState.WARNING
    elif ratio < 0.95:
        return TokenWarningState.DANGER
    else:
        return TokenWarningState.CRITICAL
```

### 第二级：对话压缩（Compaction）

当 token 使用量超过阈值时，用一次模型调用生成历史摘要，替换原始消息列表：

```python
def compact_conversation(
    engine: ConversationEngine,
    keep_last_n: int = 4,
) -> None:
    """
    调用模型将当前消息历史压缩为摘要。
    压缩后的消息列表由 [摘要消息 + 最近 N 轮] 构成。
    压缩边界前的消息从内存中移除，允许 GC 回收。
    """
    client = anthropic.Anthropic()

    # 构造压缩请求：请求模型总结当前对话
    summary_response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2048,
        system="请将以下对话历史压缩为结构化摘要，保留所有关键决策、工具执行结果和上下文信息。",
        messages=engine.messages,
    )
    summary_text = summary_response.content[0].text

    # 保留最近 N 轮消息（防止压缩掉正在进行的工具调用链）
    recent = engine.messages[-keep_last_n:] if len(engine.messages) > keep_last_n else []

    # 用摘要替换历史，就地修改（允许 GC 回收旧列表）
    engine.messages.clear()
    engine.messages.append({
        "role": "user",
        "content": f"[对话历史摘要]\n{summary_text}",
    })
    engine.messages.append({
        "role": "assistant",
        "content": "已理解历史上下文，继续执行。",
    })
    engine.messages.extend(recent)


def run_loop_with_compaction(engine: ConversationEngine) -> str:
    """在主循环中集成自动压缩。"""
    client = anthropic.Anthropic()
    while True:
        normalized = normalize_messages_for_api(engine.messages)
        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=4096,
            system=engine.system_prompt,
            tools=[t.to_api_schema() for t in engine.tools] if hasattr(engine, 'tools') else [],
            messages=normalized,
        )
        engine.total_input_tokens += response.usage.input_tokens

        # 每轮结束后检查 token 状态
        state = calculate_token_warning_state(response.usage.input_tokens)
        if state == TokenWarningState.CRITICAL:
            compact_conversation(engine)

        engine.messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason == "end_turn":
            for block in response.content:
                if hasattr(block, "text"):
                    return block.text
            return ""

        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                raw = dispatch_tool(block.name, block.input)
                tool_result = apply_result_budget(block.name, block.id, raw)
                tool_results.append(tool_result)
        engine.messages.append({"role": "user", "content": tool_results})
```

### 第三级：工具结果预算

见第五节，在结果注入前截断单条过大结果，避免单次工具调用消耗过多 token。

---

## 八、权限模型：三层检查与多种执行模式

### 三层检查

```
准备执行工具
    ↓
第一层：输入验证（工具自身）
    — 参数类型、格式、业务约束是否满足
    — 验证失败直接告知模型，不进入权限判断
    ↓
第二层：通用权限检查
    — 查询永久允许规则列表 → 命中则放行
    — 查询永久拒绝规则列表 → 命中则拒绝
    — 根据当前权限模式决策（见下表）
    ↓
第三层：工具级权限（工具自身）
    — 工具有特殊约束时在此层覆盖通用决策
```

### 权限模式

| 模式 | 行为 |
|---|---|
| `interactive` | 写操作和破坏性操作需用户确认 |
| `bypass` | 所有操作静默放行（无监督自动化场景） |
| `accept_edits` | 文件修改操作静默放行，其余仍需确认 |
| `plan_only` | 拒绝所有 `is_read_only=False` 的工具 |

```python
from enum import Enum
from typing import Optional

class PermissionMode(Enum):
    INTERACTIVE = "interactive"
    BYPASS = "bypass"
    ACCEPT_EDITS = "accept_edits"
    PLAN_ONLY = "plan_only"

@dataclass
class PermissionContext:
    mode: PermissionMode
    always_allow: list[str] = field(default_factory=list)  # 工具名白名单
    always_deny: list[str] = field(default_factory=list)   # 工具名黑名单

def check_permission(
    tool: Tool,
    args: dict,
    perm_ctx: PermissionContext,
    ask_user: Optional[Callable[[str], bool]] = None,
) -> bool:
    """
    按三层顺序检查工具执行权限。
    返回 True 表示允许执行，False 表示拒绝。
    """
    # 第一层：永久规则（优先级最高）
    if tool.name in perm_ctx.always_deny:
        return False
    if tool.name in perm_ctx.always_allow:
        return True

    # 第二层：模式决策
    if perm_ctx.mode == PermissionMode.BYPASS:
        return True

    if perm_ctx.mode == PermissionMode.PLAN_ONLY:
        return tool.is_read_only  # 只读工具才允许

    if perm_ctx.mode == PermissionMode.ACCEPT_EDITS:
        # 文件写操作自动允许，破坏性操作仍需确认
        if not tool.is_destructive:
            return True

    # 交互模式：破坏性操作询问用户
    if tool.is_destructive and ask_user:
        return ask_user(f"工具 '{tool.name}' 将执行不可逆操作，是否允许？")

    return True
```

### 规则来源分离

Claude Code 将权限规则按**来源**分组存储，而非混合在一个列表中。好处：撤销某个来源（如退出某个模式）时，只删除该来源的规则，不影响其他来源。

```python
from collections import defaultdict

class PermissionRuleStore:
    """按来源分组存储权限规则，支持按来源撤销。"""

    def __init__(self):
        # source -> list of tool names
        self._allow: dict[str, list[str]] = defaultdict(list)
        self._deny: dict[str, list[str]] = defaultdict(list)

    def add_allow(self, source: str, tool_name: str):
        self._allow[source].append(tool_name)

    def add_deny(self, source: str, tool_name: str):
        self._deny[source].append(tool_name)

    def revoke_source(self, source: str):
        """撤销某个来源的所有规则（如退出某个命令模式时调用）。"""
        self._allow.pop(source, None)
        self._deny.pop(source, None)

    def is_always_allowed(self, tool_name: str) -> bool:
        return any(tool_name in rules for rules in self._allow.values())

    def is_always_denied(self, tool_name: str) -> bool:
        return any(tool_name in rules for rules in self._deny.values())
```

---

## 九、多 Agent 模式：子 Agent 的隔离边界

当主 Agent 需要并行处理多个独立子任务时，为每个子任务创建独立的会话引擎实例。隔离的关键在于：

1. **状态不共享**：子 Agent 持有独立的消息历史，主 Agent 不能直接读取
2. **状态写入受限**：子 Agent 不能修改主 Agent 的全局状态（防止竞态污染）
3. **工作目录隔离**：子 Agent 在独立的文件系统路径（git worktree）中操作

```python
import asyncio
import uuid
from dataclasses import dataclass, field
from typing import Any

@dataclass
class SubAgentResult:
    agent_id: str
    status: str   # "completed" | "failed" | "killed"
    output: str
    error: str | None = None

async def run_sub_agent(
    task_prompt: str,
    parent_tools: list[Tool],
    parent_system_prompt: str,
    working_dir: str,
) -> SubAgentResult:
    """
    创建并运行一个隔离的子 Agent。
    子 Agent 持有独立的 ConversationEngine 实例。
    结果通过返回值传递给父 Agent，而非共享状态。
    """
    agent_id = str(uuid.uuid4())[:8]

    # 子 Agent 使用独立的引擎实例——不是父引擎的引用
    sub_engine = ConversationEngine(
        system_prompt=parent_system_prompt,
        tools=parent_tools,
    )

    try:
        output = sub_engine.submit(task_prompt)
        return SubAgentResult(agent_id=agent_id, status="completed", output=output)
    except Exception as e:
        return SubAgentResult(agent_id=agent_id, status="failed", output="", error=str(e))


async def run_parallel_sub_agents(subtasks: list[str], tools: list[Tool], system_prompt: str) -> list[SubAgentResult]:
    """并行创建多个子 Agent，等待全部完成。"""
    tasks = [
        run_sub_agent(prompt, tools, system_prompt, working_dir=f"/tmp/worktree_{i}")
        for i, prompt in enumerate(subtasks)
    ]
    return await asyncio.gather(*tasks)
```

---

## 十、任务状态机与磁盘输出

子任务（尤其是长期运行的后台任务）需要一个明确的状态机，以及将输出持久化到磁盘的机制。

**状态机**：`pending → running → completed | failed | killed`

`completed`、`failed`、`killed` 为终态，进入后不再接受状态转移。

**磁盘输出设计**：任务输出流式写入文件，调用方通过记录已读偏移量做增量读取，实现进度查询与执行解耦。进程崩溃后输出不丢失。

```python
import os
import time
from dataclasses import dataclass
from typing import Iterator

TERMINAL_STATUSES = {"completed", "failed", "killed"}

@dataclass
class TaskState:
    task_id: str
    status: str = "pending"
    output_file: str = ""
    output_offset: int = 0   # 已读取的字节偏移量
    start_time: float = field(default_factory=time.time)
    end_time: float | None = None

    def __post_init__(self):
        if not self.output_file:
            self.output_file = f"/tmp/tasks/{self.task_id}.log"
            os.makedirs(os.path.dirname(self.output_file), exist_ok=True)

    def transition(self, new_status: str) -> None:
        """状态转移。终态不允许再转移。"""
        if self.status in TERMINAL_STATUSES:
            raise ValueError(f"任务 {self.task_id} 已处于终态 {self.status}，无法转移到 {new_status}")
        self.status = new_status
        if new_status in TERMINAL_STATUSES:
            self.end_time = time.time()

    def write_output(self, chunk: str) -> None:
        with open(self.output_file, "a", encoding="utf-8") as f:
            f.write(chunk)

    def read_output_delta(self) -> tuple[str, int]:
        """从上次读取位置开始读取新增输出，返回 (增量内容, 新偏移量)。"""
        if not os.path.exists(self.output_file):
            return "", self.output_offset
        with open(self.output_file, "r", encoding="utf-8") as f:
            f.seek(self.output_offset)
            delta = f.read()
        new_offset = self.output_offset + len(delta.encode("utf-8"))
        return delta, new_offset


def poll_task_output(task: TaskState, interval: float = 1.0) -> Iterator[str]:
    """轮询任务输出，直到任务进入终态。"""
    while task.status not in TERMINAL_STATUSES:
        delta, new_offset = task.read_output_delta()
        if delta:
            task.output_offset = new_offset
            yield delta
        time.sleep(interval)
    # 读取最后一批输出
    delta, new_offset = task.read_output_delta()
    if delta:
        task.output_offset = new_offset
        yield delta
```

---

## 十一、系统提示词动态组装

系统提示词在每次请求前动态构建，由多个来源合并：

```python
@dataclass
class SystemPromptBuilder:
    base_prompt: str
    tool_fragments: list[str] = field(default_factory=list)      # 来自各工具的 system_prompt_fragment
    memory_fragments: list[str] = field(default_factory=list)    # 来自 CLAUDE.md 等配置文件
    mcp_instructions: list[str] = field(default_factory=list)    # 来自 MCP server 的使用说明
    append_prompt: str = ""                                        # 调用方追加内容

    def build(self) -> str:
        parts = [self.base_prompt]
        if self.tool_fragments:
            parts.append("\n## 工具使用说明\n" + "\n".join(self.tool_fragments))
        if self.memory_fragments:
            parts.append("\n## 项目上下文\n" + "\n".join(self.memory_fragments))
        if self.mcp_instructions:
            parts.append("\n## 外部服务说明\n" + "\n".join(self.mcp_instructions))
        if self.append_prompt:
            parts.append(self.append_prompt)
        return "\n\n".join(parts)


def load_memory_fragments(cwd: str, loaded_paths: set[str]) -> list[str]:
    """
    按目录层级加载 CLAUDE.md 风格的配置文件。
    已加载的路径通过 loaded_paths 去重，跨轮次持久，防止重复注入。
    """
    fragments = []
    path = cwd
    while True:
        candidate = os.path.join(path, "AGENT.md")
        if candidate not in loaded_paths and os.path.exists(candidate):
            loaded_paths.add(candidate)
            fragments.append(open(candidate).read())
        parent = os.path.dirname(path)
        if parent == path:
            break
        path = parent
    return list(reversed(fragments))  # 越靠近根目录的优先级越低，反转后追加
```

---

## 十二、会话持久化：预写日志与恢复

**预写日志（Write-Ahead）**：用户消息在进入 API 调用循环**之前**写入磁盘。目的：若进程在 API 响应返回前崩溃，恢复时仍能找到该消息并继续，而非要求用户重新输入。

```python
import json

TRANSCRIPT_DIR = os.path.expanduser("~/.agent/transcripts")

def write_ahead_user_message(session_id: str, message: dict) -> None:
    """在发起 API 调用前将用户消息写入 transcript 文件。"""
    os.makedirs(TRANSCRIPT_DIR, exist_ok=True)
    path = os.path.join(TRANSCRIPT_DIR, f"{session_id}.jsonl")
    with open(path, "a", encoding="utf-8") as f:
        f.write(json.dumps({"type": "user", "data": message}, ensure_ascii=False) + "\n")

def append_assistant_message(session_id: str, message: dict) -> None:
    path = os.path.join(TRANSCRIPT_DIR, f"{session_id}.jsonl")
    with open(path, "a", encoding="utf-8") as f:
        f.write(json.dumps({"type": "assistant", "data": message}, ensure_ascii=False) + "\n")

def load_session(session_id: str) -> list[dict]:
    """从磁盘恢复会话消息历史。"""
    path = os.path.join(TRANSCRIPT_DIR, f"{session_id}.jsonl")
    if not os.path.exists(path):
        return []
    messages = []
    with open(path, encoding="utf-8") as f:
        for line in f:
            entry = json.loads(line)
            messages.append(entry["data"])
    return messages


def submit_with_persistence(engine: ConversationEngine, session_id: str, user_content: str) -> str:
    """在主循环外先持久化用户消息，再进入 API 循环。"""
    user_message = {"role": "user", "content": user_content}

    # 预写：在 API 调用前落盘
    write_ahead_user_message(session_id, user_message)
    engine.messages.append(user_message)

    # 进入 API 循环
    result = _run_loop(engine)

    # 持久化最终回复
    if engine.messages and engine.messages[-1]["role"] == "assistant":
        append_assistant_message(session_id, engine.messages[-1])

    return result
```

---

## 十三、启动性能：并行预取慢操作

Agent 初始化时通常有多个耗时的 I/O 操作（读取凭据、建立 MCP 连接、加载配置等）。串行执行会导致启动延迟叠加。

**做法**：在主初始化流程开始时，立即并行触发所有慢操作，不等结果。主流程继续执行同步初始化。在真正需要某个结果的节点再等待（此时大概率已完成）。

```python
import asyncio
from typing import Any

async def prefetch_credentials() -> str:
    await asyncio.sleep(0.065)  # 模拟 keychain 读取 ~65ms
    return "sk-ant-..."

async def prefetch_mcp_connections() -> list[dict]:
    await asyncio.sleep(0.3)   # 模拟 MCP 握手
    return [{"name": "filesystem", "status": "connected"}]

async def prefetch_config() -> dict:
    await asyncio.sleep(0.02)
    return {"model": "claude-opus-4-6"}

async def initialize_agent() -> ConversationEngine:
    """
    并行触发所有慢速初始化操作。
    主流程继续执行，在需要时才 await 结果。
    """
    # 立即并行触发，不等待
    cred_task = asyncio.create_task(prefetch_credentials())
    mcp_task = asyncio.create_task(prefetch_mcp_connections())
    config_task = asyncio.create_task(prefetch_config())

    # 执行不依赖上述结果的同步初始化
    system_prompt = "你是一个代码助手。"
    tools: list[Tool] = []

    # 在需要时 await（此时大概率已完成，等待时间接近零）
    api_key = await cred_task
    mcp_connections = await mcp_task
    config = await config_task

    return ConversationEngine(
        system_prompt=system_prompt,
        tools=tools,
    )
```

---

## 十四、命令路由：三种执行路径

面向用户的命令（如 `/commit`、`/review`）按执行方式分为三类，共享相同的路由入口：

| 类型 | 行为 | 适用场景 |
|---|---|---|
| `local` | 直接执行本地逻辑，不调用模型 | git 操作、配置管理 |
| `interactive` | 渲染交互 UI，完成后可选触发模型 | 配置向导、确认对话框 |
| `prompt` | 将命令内容转为用户消息，进入完整 agent 循环 | 需要模型参与的任务 |

`prompt` 类型可以附加约束：`allowed_tools`（本次循环只允许使用的工具白名单）和 `fork`（在独立子 Agent 中执行，不污染当前会话的消息历史）。

```python
from typing import Callable

@dataclass
class Command:
    name: str
    description: str
    handler_type: str  # "local" | "prompt"
    # local 类型
    local_fn: Callable[[str], str] | None = None
    # prompt 类型
    get_prompt: Callable[[str], str] | None = None
    allowed_tools: list[str] | None = None   # None 表示不限制
    fork: bool = False                        # True 则在子 Agent 中执行

COMMANDS: dict[str, Command] = {}

def register_command(cmd: Command):
    COMMANDS[cmd.name] = cmd

def route_command(
    user_input: str,
    engine: ConversationEngine,
    session_id: str,
) -> str | None:
    """
    若输入以 / 开头，匹配命令并路由；否则返回 None（由调用方走普通 agent 循环）。
    """
    if not user_input.startswith("/"):
        return None

    parts = user_input[1:].split(" ", 1)
    name, args = parts[0], parts[1] if len(parts) > 1 else ""

    cmd = COMMANDS.get(name)
    if not cmd:
        return f"未知命令：/{name}"

    if cmd.handler_type == "local":
        return cmd.local_fn(args)

    if cmd.handler_type == "prompt":
        prompt = cmd.get_prompt(args)
        if cmd.fork:
            # 在子 Agent 中执行，不修改当前引擎的消息历史
            sub = ConversationEngine(
                system_prompt=engine.system_prompt,
                tools=engine.tools,
            )
            return sub.submit(prompt)
        else:
            return submit_with_persistence(engine, session_id, prompt)

    return None


# 注册示例
register_command(Command(
    name="status",
    description="显示当前会话状态",
    handler_type="local",
    local_fn=lambda _: f"消息数：{0}，Token 用量：{0}",
))

register_command(Command(
    name="review",
    description="对当前代码变更进行 code review",
    handler_type="prompt",
    get_prompt=lambda args: f"请对以下变更进行 code review：{args}",
    allowed_tools=["read_file", "glob"],  # review 只需读工具
))
```

---

## 十五、设计决策清单

构建 agent 系统时，以下问题需在设计阶段明确：

**会话管理**
- [ ] 会话状态封装在哪个对象中？该对象的生命周期与对话生命周期是否一致？
- [ ] 多轮调用共享同一个引擎实例，还是每轮重建？

**主循环**
- [ ] 是否在每次 API 调用前对消息列表做规范化处理？
- [ ] 循环的终止条件有哪些？（`end_turn`、最大轮次、Token 预算、用户中断）
- [ ] 是否支持流式输出？

**工具系统**
- [ ] 工具元数据是否包含 `is_read_only`、`is_destructive`、`is_concurrency_safe`？
- [ ] 工具结果是否有大小预算？超限时的处理策略是什么？
- [ ] 工具是全部初始加载，还是按需延迟加载？

**Context 管理**
- [ ] 是否监控 token 使用量？分级策略是什么？
- [ ] 对话压缩由模型生成摘要，还是简单截断历史？
- [ ] 压缩后旧消息是否从内存中释放？

**权限**
- [ ] 是否区分 `is_read_only` 和 `is_destructive` 两个维度？
- [ ] 权限规则是否按来源分组，支持按来源撤销？
- [ ] 是否有无监督自动化模式（跳过所有确认）？

**多 Agent**
- [ ] 子 Agent 的消息历史与主 Agent 是否完全隔离？
- [ ] 子 Agent 是否禁止直接修改主 Agent 的全局状态？
- [ ] 任务输出是否持久化到磁盘（崩溃恢复）？
- [ ] 父 Agent 退出时如何清理其创建的子任务？

**持久化**
- [ ] 用户消息是否在 API 调用前预写入磁盘？
- [ ] 是否支持从历史 transcript 恢复会话？

**扩展性**
- [ ] 系统提示词是否支持动态组装（工具注入、记忆注入）？
- [ ] 工具是否支持外部动态注册（类 MCP 协议）？
