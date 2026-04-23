# ADK Callbacks & Events & Plugins

> 6 类 callback / Event 结构 / EventActions（escalate / transfer_to_agent / state_delta / artifact_delta）/ Plugins（跨切面机制）。
>
> **Last synced:** 2026-04-23 · **adk-python commit:** `62d7ee0` · **Upstream:** https://adk.dev/callbacks/ · https://adk.dev/events/ · https://adk.dev/plugins/

---

## Callback 6 类总览

| 名字 | 触发时机 | Context 类型 | 返回值语义 |
|------|---------|-------------|-----------|
| `before_agent_callback` | 进入 agent 时 | `CallbackContext` | 返回 `types.Content` 可跳过 agent 直接响应；返回 `None` 正常执行 |
| `after_agent_callback` | 退出 agent 时 | `CallbackContext` | 返回 `types.Content` 可覆盖最终响应 |
| `before_model_callback` | 调 LLM 前 | `CallbackContext` | 返回 `LlmResponse` 可跳过 LLM（缓存命中、测试 stub）；可修改 request |
| `after_model_callback` | LLM 返回后 | `CallbackContext` | 返回 `LlmResponse` 可覆盖响应（内容过滤、摘要） |
| `before_tool_callback` | 调 tool 前 | `ToolContext` | 返回 `dict` 可跳过 tool 直接用该 dict 作结果（权限拒绝、mock） |
| `after_tool_callback` | tool 返回后 | `ToolContext` | 返回 `dict` 可覆盖 tool 结果 |

**挂载方式**：都作为 `Agent(...)` 的构造参数。单个 callback 或 list 都可以。

**Docs**：https://adk.dev/callbacks/

---

## 各 Callback 详细

### `before_agent_callback` — 状态初始化最佳位置

**签名**：
```python
async def cb(callback_context: CallbackContext) -> Optional[types.Content]:
    ...
```

**典型用途**：
- 初始化 session state 默认值
- 预热 memory（调 `preload_memory`）
- 根据 user 权限决定是否允许执行（返回 Content 来拒绝）

**示例**（初始化 state）：
```python
from google.adk.agents.callback_context import CallbackContext

async def init_state(cc: CallbackContext) -> None:
    defaults = {
        StateKeys.COMPANY_NAME: "",
        StateKeys.HAS_ADDITIONAL_FILES: False,
        StateKeys.PIPELINE_COMPLETED_STEPS: [],
    }
    for k, v in defaults.items():
        if k not in cc.state:
            cc.state[k] = v

root_agent = Agent(before_agent_callback=init_state, ...)
```

**CLAUDE.md**：Root callback 是确定性设置 state 的最佳位置（在 LLM 执行前就已确定）。

---

### `after_agent_callback` — 清理与结果校验

**签名**：
```python
async def cb(callback_context: CallbackContext) -> Optional[types.Content]:
    ...
```

**典型用途**：
- Post-hoc 结果 validation
- 清理 temp: 前缀的 state
- 触发异步清理（`asyncio.ensure_future` 清理失败的 upload 等）

---

### `before_model_callback` — 改 prompt / 缓存 / stub

**签名**：
```python
from google.adk.models import LlmRequest, LlmResponse

async def cb(callback_context: CallbackContext, request: LlmRequest) -> Optional[LlmResponse]:
    # 可修改 request（如追加 system message）
    # 返回 LlmResponse 跳过真实 LLM（缓存命中、测试 stub）
    ...
```

**典型用途**：
- 敏感词过滤 / PII 检查 request
- 缓存层（相同 prompt 返回上次响应）
- 测试时 stub LLM

---

### `after_model_callback` — 改 response / 过滤 / 日志

**签名**：
```python
async def cb(callback_context: CallbackContext, response: LlmResponse) -> Optional[LlmResponse]:
    ...
```

**典型用途**：
- 内容安全过滤
- 统一的 LLM 响应日志（INFO 级别，带 token 数和 tool_calls 名字）
- 响应摘要 / 提取字段

---

### `before_tool_callback` — tool 权限 / mock / 参数预处理

**签名**：
```python
from google.adk.tools import ToolContext, BaseTool

async def cb(tool: BaseTool, args: dict, tool_context: ToolContext) -> Optional[dict]:
    # 返回 dict 跳过 tool 直接用该 dict 作结果
    ...
```

**典型用途**：
- 敏感 tool（删除、支付）要求用户确认（配合 Action Confirmations）
- 参数预处理（如强制把 user_id 注入）
- 单测 mock tool 结果

---

### `after_tool_callback` — tool 结果修改 / 记录

**签名**：
```python
async def cb(tool: BaseTool, args: dict, tool_context: ToolContext, result: dict) -> Optional[dict]:
    ...
```

**典型用途**：
- 脱敏 tool 结果再进 LLM
- 把 tool 结果写入额外的 state key
- 收集 metrics

---

## Callback 挂载多个

`Agent` 构造参数既接受单个 callback 也接受 list：

```python
Agent(
    before_agent_callback=[init_state, preload_memory_cb],
    after_model_callback=[content_filter, token_logger],
    ...
)
```

按注册顺序链式调用。前一个返回非 None 会短路后面的。

---

## Event 结构

**Import**：`from google.adk.events import Event, EventActions`

**Event 核心字段**（以源码为准）：

| 字段 | 说明 |
|------|------|
| `id` | Event 唯一 ID |
| `invocation_id` | 所属 invocation |
| `author` | 产生者：`"user"` / agent name / `"system"` |
| `content` | `types.Content`（文本 / tool_call / tool_response / function_call 等） |
| `actions` | `EventActions` 实例，见下 |
| `timestamp` | 时间戳 |
| `error_code` / `error_message` | 错误信息（如果有） |
| `partial` | 是否为流式输出的中间片段 |

---

## EventActions

**Import**：`from google.adk.events import EventActions`

**关键字段**（以源码为准）：

| 字段 | 类型 | 作用 |
|------|------|------|
| `state_delta` | `dict[str, Any]` | 本 event 对 state 的变更（框架自动维护，append_event 时应用） |
| `artifact_delta` | `dict[str, int]` | 本 event 新保存的 artifact 及版本号 |
| `transfer_to_agent` | `str \| None` | 把对话转交给指定 agent |
| `escalate` | `bool` | 触发 LoopAgent 退出 |
| `skip_summarization` | `bool` | 跳过本次 tool 结果的 summarization |
| `requested_auth_configs` | `dict` | 标记需要用户授权（OAuth 交互回合） |

**在 Tool 中设置**：
```python
def my_tool(tool_context: ToolContext) -> dict:
    tool_context.actions.escalate = True
    tool_context.actions.skip_summarization = True
    tool_context.actions.transfer_to_agent = "other_agent"
    return {"status": "done"}
```

---

## 在 BaseAgent 中 yield Event

`BaseAgent._run_async_impl` 需要 yield `Event` 流转给 runner。通常只需把 sub_agents 的事件流 yield 出去：

```python
class MyPipeline(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext):
        for sub in self.sub_agents:
            async for event in sub.run_async(ctx):
                yield event     # 透传
            if ctx.end_invocation:
                return
```

需要主动产出 event（如自定义进度消息）时：

```python
from google.adk.events import Event
from google.genai import types

yield Event(
    invocation_id=ctx.invocation_id,
    author=self.name,
    content=types.Content(parts=[types.Part(text="进度: 50%")]),
)
```

**Docs**：https://adk.dev/events/

---

## Plugins（跨切面机制）

**用途**：类似 AOP，把横切关注点（logging、retry、guardrails）从业务 agent 抽出，全局挂载到 runner 上。

**Import**：
```python
from google.adk.plugins import (
    BasePlugin,
    LoggingPlugin,
    DebugLoggingPlugin,
    PluginManager,
    ReflectAndRetryToolPlugin,
)
```

---

### 7 个生命周期钩子

`BasePlugin` 提供以下钩子（以源码为准）：

| 钩子 | 触发 |
|------|------|
| `on_user_message_callback` | 用户消息到达 |
| `on_start_invocation_callback` | Runner 启动 invocation |
| `before_agent_callback` / `after_agent_callback` | Agent 执行前后 |
| `before_model_callback` / `after_model_callback` | LLM 调用前后 |
| `before_tool_callback` / `after_tool_callback` | Tool 执行前后 |
| `on_event_callback` | 任意 event 流出时 |
| `on_end_invocation_callback` | Runner 结束 invocation |

（具体钩子名和签名以 `src/google/adk/plugins/base_plugin.py` 为准。）

---

### 预置 Plugin

| Plugin | 用途 |
|--------|------|
| `LoggingPlugin` | 标准 INFO 级日志（agent / tool / model 调用） |
| `DebugLoggingPlugin` | 详细 DEBUG 日志（state 变更、full prompt） |
| `ReflectAndRetryToolPlugin` | Tool 失败时让 LLM reflect 并重试 |
| （社区 / integrations 中）BigQuery Analytics、Context Filter、Global Instruction、Save Files as Artifacts | 见 Integrations 目录 |

---

### 挂载 Plugin

```python
from google.adk.runners import Runner
from google.adk.plugins import LoggingPlugin

runner = Runner(
    agent=root_agent,
    app_name="my_app",
    plugins=[LoggingPlugin(), my_custom_plugin],
)
```

**Docs**：https://adk.dev/plugins/

---

## Callback vs Plugin：怎么选

| 需求 | 用 Callback | 用 Plugin |
|------|-------------|-----------|
| 只影响某一个 agent | ✅ | ❌（全局） |
| 影响所有 agent | ❌（重复挂载） | ✅ |
| 需要访问当前 agent 的业务 state | ✅ | 部分可行 |
| 横切日志、重试、监控 | 可以但重复 | ✅ 推荐 |

---

## CLAUDE.md 相关约束

- 每个 BaseAgent 的 `_run_async_impl` 入口 log `INFO "agent {name} start"`，出口 log `INFO "agent {name} done"`（含耗时）
- LlmAgent 的 before/after callback 必须 log 触发事件
- 关键决策分支（if/else 路由）必须 log 决策依据
- Tool 调用前后必须 log（名字、入参摘要、耗时）
- 异常必须用 `logger.exception()` 保留 stack trace，不要只记录 `str(e)`
- 不允许 `except Exception: pass`；静默吞异常要 log warning 并说明原因
- LLM prompt 和响应生产环境仅 DEBUG 级别 log 全文；INFO 级别只 log 长度和 tool_calls 名字

---

## 常见模式

### 模式：根 agent 统一 log

```python
async def log_in(cc: CallbackContext) -> None:
    logger.info("agent %s start", cc.agent_name,
                extra={"session_id": cc.state.get("_session_id"),
                       "invocation_id": cc.invocation_id})

async def log_out(cc: CallbackContext) -> None:
    logger.info("agent %s done", cc.agent_name, extra={...})

root_agent = Agent(
    before_agent_callback=[init_state, log_in],
    after_agent_callback=[log_out],
    ...
)
```

（全局 logging 更建议用 `LoggingPlugin`。）

### 模式：Callback 用 State Flag 控制分支

```python
async def route_decision(cc: CallbackContext) -> Optional[types.Content]:
    if not cc.state.get(StateKeys.AUTH_CONFIRMED):
        # 跳过 agent 执行，直接响应请求授权
        return types.Content(parts=[types.Part(text="请先完成授权")])
    return None  # 正常执行
```

---

## 上游参考

- **Callbacks**：https://adk.dev/callbacks/
- **Events**：https://adk.dev/events/
- **Plugins**：https://adk.dev/plugins/
- **Python API Reference**：https://adk.dev/api-reference/python/
- **源码**：
  - `src/google/adk/agents/callback_context.py`
  - `src/google/adk/events/event.py`
  - `src/google/adk/events/event_actions.py`
  - `src/google/adk/plugins/`
