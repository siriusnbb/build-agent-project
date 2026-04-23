# ADK Context & State

> 4 种 Context 层级、Context 包装模式、State 前缀、StateKeys 集中式常量、session.events、动态 instruction。
>
> **Last synced:** 2026-04-23 · **adk-python commit:** `62d7ee0` · **Upstream:** https://adk.dev/context/session/ · https://adk.dev/get-started/about/

---

## 4 种 Context 总览

ADK 按生命周期阶段提供 4 种 Context，权限和可见信息逐级扩大：

| Context | 用途 | 可做什么 | 源码 |
|---------|------|---------|------|
| `ReadonlyContext` | 动态 instruction | **只读** state，用来做条件 prompt | `agents/readonly_context.py` |
| `CallbackContext` | 6 类 callback | 读写 state；访问 agent_name / user_content；保存/加载 artifact | `agents/callback_context.py` |
| `Context`（= `ToolContext`） | Tool 内部 | 读写 state；save/load/list artifact；`actions`（如 `escalate`） | `agents/context.py`（别名 `tools.ToolContext`） |
| `InvocationContext` | `BaseAgent._run_async_impl` | 访问完整 invocation：session（含 events）、end_invocation、sub_agents | `agents/invocation_context.py` |

**继承关系**（简化，以源码为准）：
```
ReadonlyContext  <---  CallbackContext
                 \
                   <-- Context (ToolContext)
InvocationContext (独立，承载最完整的 runtime 信息)
```

**Docs**：详细 API 在 https://adk.dev/api-reference/python/（仅 API reference，无专题教程）

---

## 各 Context 详细

### `ReadonlyContext`

**用途**：`instruction` 作为 `Callable[[ReadonlyContext], str]` 时，允许根据 state 动态生成 prompt。**只读**。

**Import**：
```python
from google.adk.agents.readonly_context import ReadonlyContext
```

**可访问**：
- `ctx.state`（只读 dict-like）

**典型用法**：
```python
def dynamic_instruction(ctx: ReadonlyContext) -> str:
    if ctx.state.get("has_uploaded_file"):
        return PROMPT_WITH_FILE
    return PROMPT_WITHOUT_FILE

Agent(instruction=dynamic_instruction, ...)
```

---

### `CallbackContext`

**用途**：6 类 callback 的参数（see `callbacks-and-events.md`）。可读写 state、访问 agent 元信息、操作 artifact。

**Import**：
```python
from google.adk.agents.callback_context import CallbackContext
```

**可访问（常用）**：
- `ctx.state` — 可读写 dict
- `ctx.agent_name` — 当前 agent 名
- `ctx.user_content` — 本轮用户输入（`types.Content`）
- `ctx.invocation_id` — 本次 invocation 唯一 ID
- `await ctx.save_artifact(filename, part)` → 返回版本号
- `await ctx.load_artifact(filename, version=None)` → `types.Part`
- `await ctx.list_artifacts()` → 文件名列表

**典型用法**：在 `before_agent_callback` 里初始化 state：
```python
async def init_state(ctx: CallbackContext) -> None:
    defaults = {"company_name": "", "pipeline_completed_steps": []}
    for k, v in defaults.items():
        if k not in ctx.state:
            ctx.state[k] = v
```

---

### `Context`（= `ToolContext`）

**用途**：Tool 函数签名末尾 `tool_context: ToolContext` 时 ADK 注入的对象。也是 `BaseAgent._run_async_impl` 中手动包装 `InvocationContext` 得到的类型。

**Import**：
```python
from google.adk.tools import ToolContext       # 推荐，tool 内
from google.adk.agents import Context          # 等价，BaseAgent 包装时
```

（`ToolContext` 是 `Context` 的别名）

**可访问**（叠加在 `CallbackContext` 之上）：
- 所有 `CallbackContext` 的能力
- `ctx.actions` — `EventActions` 实例，可设置：
  - `ctx.actions.escalate = True` — 退出 LoopAgent
  - `ctx.actions.skip_summarization = True` — 跳过本次 tool 结果的 summarization
  - `ctx.actions.transfer_to_agent = "other_agent"` — 显式 handoff
  - `state_delta` / `artifact_delta`（通常由框架自动维护）

**典型用法**：
```python
def my_tool(param: str, tool_context: ToolContext) -> dict:
    tool_context.state["last_param"] = param
    tool_context.actions.skip_summarization = True
    return {"status": "success"}
```

---

### `InvocationContext`

**用途**：`BaseAgent._run_async_impl(ctx: InvocationContext)` 的参数。承载完整 runtime 信息。

**Import**：
```python
from google.adk.agents.invocation_context import InvocationContext
```

**可访问（常用）**：
- `ctx.session` — 当前 `Session` 对象
  - `ctx.session.id` — session ID（不要自建 UUID）
  - `ctx.session.state` — 可读写 state dict
  - `ctx.session.events` — 历史事件列表（`list[Event]`）
  - `ctx.session.user_id` / `ctx.session.app_name`
- `ctx.invocation_id` — 本次 invocation ID（跨 event 追踪用）
- `ctx.end_invocation` — bool，是否提前终止
- `ctx.user_content` — 本轮用户输入
- `ctx.agent` — 当前 agent 实例（可取 `self.sub_agents` 等）

**典型用法**（BaseAgent pipeline）：
```python
class MyPipeline(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext):
        for sub in self.sub_agents:
            async for event in sub.run_async(ctx):
                yield event
            if ctx.end_invocation:
                return
```

---

## Context(ctx) 包装模式（BaseAgent 里访问 artifact）

**问题**：`InvocationContext` 本身不暴露 `save_artifact` / `load_artifact`。

**解法**：用 `Context(ctx)` 包装得到 `Context`（= `ToolContext`）后操作：

```python
from google.adk.agents import Context

class MyPipeline(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext):
        context = Context(ctx)     # 包装
        await context.save_artifact("result.json", part)
        # ... 继续调度 sub_agents
        yield from ...  # 正常流程
```

**CLAUDE.md 规则**：`BaseAgent._run_async_impl(ctx: InvocationContext)` 中用 `Context(ctx)` 构建 Context 来访问 artifact（见 "ADK Artifact 使用规范" 章节）。

---

## Context 跨场景访问对照表

| 场景 | state 读写 | artifact | actions (escalate 等) | 元信息 |
|------|-----------|----------|----------------------|-------|
| 动态 instruction | `ctx.state.get(...)` 只读 | ❌ | ❌ | ❌ |
| before/after agent callback | `ctx.state[k] = v` | ✅ | ❌ | `ctx.agent_name` |
| before/after model callback | `ctx.state[k] = v` | ✅ | ❌ | 可改 request/response |
| before/after tool callback | `ctx.state[k] = v` | ✅ | ❌ | 可改 args / result |
| Tool 函数体 | `tc.state[k] = v` | ✅ | ✅ `tc.actions.escalate = True` | `tc.agent_name` |
| BaseAgent `_run_async_impl` | `ctx.session.state[k] = v` | 需 `Context(ctx)` | 通过 yield Event 控制 | `ctx.session.id` / `ctx.invocation_id` |

---

## Session State 概述

**`Session.state`** 是一个可读写的 dict-like 结构，按 key 前缀区分作用域：

| 前缀 | 作用域 | 持久化 |
|------|-------|--------|
| （无） | 当前 session | 随 session 持久化 |
| `user:` | 当前 user 所有 session | 跨 session |
| `app:` | 整个 app 所有 user | 全局共享 |
| `temp:` | 当前 invocation | 不持久化 |

**使用规则（项目约定）：**

1. 每个 key 职责唯一（不一个 key 编码多个含义）
2. 不用 value 前缀（如 `"gcs:xxx"`）表示状态——用独立 boolean flag
3. 环境变量值直接从 config 读，不写 state
4. 只存轻量数据（字符串、数字、小 dict），binary 用 artifact
5. 不要自建 UUID，用 `session.id`

**Why**（来自 CLAUDE.md "State 设计原则"）：避免混乱的状态语义、减少不必要的序列化负担、让 debug 时状态含义一目了然。

---

## StateKeys 集中式常量模式

**问题**：state key 散落在多个 agent / tool 文件里，改名字时漏掉一个就引发沉默 bug。

**解法**：在 `app/config.py` 下用一个类统一收集：

```python
# app/config.py
class StateKeys:
    COMPANY_NAME = "company_name"
    HAS_ADDITIONAL_FILES = "has_additional_files"
    PIPELINE_COMPLETED_STEPS = "pipeline_completed_steps"
    STRUCTURED_REPORT = "structured_report"
    UPLOAD_RESULT = "upload_result"
    INDEX_READY = "index_ready"
    # ... 所有 key 集中定义
```

**各处使用**：

```python
# Tool 中
tool_context.state[StateKeys.COMPANY_NAME] = "..."

# BaseAgent 中
ctx.session.state[StateKeys.PIPELINE_COMPLETED_STEPS] = [...]

# Callback 中
callback_context.state[StateKeys.INDEX_READY] = True

# 动态 instruction 中（只读）
value = ctx.state.get(StateKeys.COMPANY_NAME)
```

**命名规范**：让不了解上下文的人也能理解。避免缩写和内部术语。

---

## 动态 Instruction（Callable[[ReadonlyContext], str]）

**用途**：根据当前 state 动态生成 prompt。

```python
from google.adk.agents.readonly_context import ReadonlyContext

def dynamic_instruction(ctx: ReadonlyContext) -> str:
    base = BASE_PROMPT
    if ctx.state.get(StateKeys.HAS_ADDITIONAL_FILES):
        return base + ADDITIONAL_FILES_APPENDIX
    return base

Agent(instruction=dynamic_instruction, ...)
```

**CLAUDE.md 规则**：Prompt 中的条件选择逻辑放在 prompt 文件中（不要放在 agent 文件中），保持关注点分离。建议把 `dynamic_instruction` 放到 `app/sub_agents/{name}/prompt.py`，而不是 `agent.py`。

---

## 访问 session.events（历史事件）

**在 BaseAgent 中：**

```python
async def _run_async_impl(self, ctx: InvocationContext):
    for event in ctx.session.events:
        if event.author == "user":
            ...
        if event.actions and event.actions.escalate:
            ...
```

**事件来源**：每个 LLM 调用、tool 调用、callback、agent transfer 都会产生一个或多个 `Event`。详细结构见 `callbacks-and-events.md`。

---

## invocation_id vs session.id

| ID | 作用域 | 典型用途 |
|----|-------|---------|
| `session.id` | 一次会话 | session 持久化的主键、跨 invocation 的稳定标识 |
| `invocation_id` | 一次用户消息触发的执行 | 链路追踪、断点续传（`ResumabilityConfig`）、log 关联 |

**一次 session 里可能有多次 invocation**（用户发 N 条消息 = N 次 invocation，共享 session.state）。

**CLAUDE.md 日志要求**：所有 log 必须带 `session.id` 和 `invocation_id`（通过 `extra={}` 注入或 LoggerAdapter），支持按单次调用链路追查。

---

## Session 对象

**Import**：`from google.adk.sessions import Session, State`

常用属性：
- `session.id` — 主键
- `session.app_name` — app 标识
- `session.user_id` — 用户 ID
- `session.state` — state dict（实际类型是 `State`）
- `session.events` — `list[Event]`
- `session.last_update_time` — 更新时间戳

**Session 的生命周期**由 SessionService 管理（create / get / append_event / delete），见 `sessions-artifacts-memory.md`。

---

## Context Caching（模型层优化）

**用途**：Gemini 2.0+ 支持 context caching，复用长 instruction 或大型数据集，减少 token 消耗。

**Docs**：https://adk.dev/context/caching/

**与本文件的区别**：此处的 "context" 指 LLM 上下文（prompt + history），不是 ADK 的 Context 类。

---

## Context Compaction（事件历史压缩）

**用途**：session 变长时，把老的 events 摘要化，避免 LLM 上下文爆炸。

**Docs**：https://adk.dev/context/compaction/

---

## Sessions Rewind / Migrate（高级）

- **Rewind**：回退 session 到某个事件点（实验性）
- **Migrate**：跨 session service 迁移数据（实验性）

上游文档：
- https://adk.dev/context/session/rewind/（如存在）
- https://adk.dev/context/session/migrate/（如存在）

---

## CLAUDE.md 相关约束

- `BaseAgent._run_async_impl(ctx)` 中用 `Context(ctx)` 包装访问 artifact
- `Root` callback 是确定性设置 state 的最佳位置（LLM 执行前就能确定的事）
- State key 命名让不了解上下文的人也能理解；避免缩写和内部术语
- 业务关键 state key 写入时 log 变更前后的值（DEBUG 级别）
- 状态变更分支必须 log 决策依据（"skip index verify: state.index_ready=true"）

---

## 上游参考

- **Context 总览**：https://adk.dev/get-started/about/
- **Sessions**：https://adk.dev/context/session/
- **Memory**：https://adk.dev/context/memory/
- **Context Caching**：https://adk.dev/context/caching/
- **Context Compaction**：https://adk.dev/context/compaction/
- **Session Rewind**：https://adk.dev/context/session/rewind/（如存在）
- **Session Migration**：https://adk.dev/context/session/migrate/（如存在）
- **Python API Reference**：https://adk.dev/api-reference/python/
- **源码**：
  - `src/google/adk/agents/readonly_context.py`
  - `src/google/adk/agents/callback_context.py`
  - `src/google/adk/agents/context.py`
  - `src/google/adk/agents/invocation_context.py`
  - `src/google/adk/sessions/session.py`
  - `src/google/adk/sessions/state.py`
