# ADK Agents

> Agent 类型、工作流编排、多 agent 组合、BaseAgent 模式。
>
> **Last synced:** 2026-04-23 · **adk-python commit:** `62d7ee0` · **Upstream:** https://adk.dev/agents/

---

## Agent 类型总览

| 类型 | 用途 | Import | 源码 |
|------|------|--------|------|
| `LlmAgent` / `Agent` | LLM 驱动，用 instruction + tools + sub_agents 编排 | `from google.adk.agents import Agent, LlmAgent` | `src/google/adk/agents/llm_agent.py` |
| `BaseAgent` | 确定性自定义编排（继承实现 `_run_async_impl`） | `from google.adk.agents import BaseAgent` | `src/google/adk/agents/base_agent.py` |
| `SequentialAgent` | 子 agent 顺序执行 | `from google.adk.agents import SequentialAgent` | `src/google/adk/agents/sequential_agent.py` |
| `LoopAgent` | 循环执行子 agent，直到 `escalate=True` 或达到 `max_iterations` | `from google.adk.agents import LoopAgent` | `src/google/adk/agents/loop_agent.py` |
| `ParallelAgent` | 所有子 agent 并行，全部完成后继续 | `from google.adk.agents import ParallelAgent` | `src/google/adk/agents/parallel_agent.py` |
| `RemoteA2aAgent` | 跨进程调用远程 agent（A2A 协议） | `from google.adk.agents import RemoteA2aAgent` | `src/google/adk/agents/remote_a2a_agent.py` |
| `LanggraphAgent` | 包装 LangGraph 图为 agent | `from google.adk.agents.langgraph_agent import LanggraphAgent` | `src/google/adk/agents/langgraph_agent.py` |

`Agent` 是 `LlmAgent` 的别名，两者可互换。

---

## LlmAgent（= Agent）

**用途**：LLM 编排型 agent，根据 `instruction` 使用 `tools` 和 `sub_agents`。

**Import**：`from google.adk.agents import Agent` 或 `LlmAgent`

**关键构造参数（完整列表见上游 API）：**

```python
Agent(
    name="my_agent",                    # 唯一标识
    model=Gemini(model="gemini-2.0-flash"),  # 或字符串 "gemini-2.0-flash"
    instruction=INSTRUCTION_OR_CALLABLE,     # str | Callable[[ReadonlyContext], str]
    description="给 dispatcher 看的描述",
    tools=[tool_func, AgentTool(sub)],       # 函数 / Tool / AgentTool
    sub_agents=[child_a, child_b],           # 子 agent（建立层级）
    output_key="state_key",                  # LLM 最终文本自动写入 state[key]
    output_schema=PydanticModel,             # 约束 LLM 输出格式
    include_contents="none",                 # "default" | "none"（排除前序消息）
    before_agent_callback=cb,                # 生命周期 callback（见 callbacks 文件）
    after_agent_callback=cb,
    before_model_callback=cb,
    after_model_callback=cb,
    before_tool_callback=cb,
    after_tool_callback=cb,
    planner=planner,                         # BasePlanner 实例（见 advanced.md）
    code_executor=executor,                  # BaseCodeExecutor 实例
    generate_content_config=types.GenerateContentConfig(...),  # 低层生成参数
)
```

**关键语义**：

- `instruction` 既可以是字符串也可以是 `Callable[[ReadonlyContext], str]`。Callable 形式在每次调用时动态生成，可根据 state 做条件 prompt，见 `context-and-state.md`。
- `output_key` 和 `output_schema` 搭配 Pydantic 见 `tools.md` 的 "Pydantic schema 集成"。
- `include_contents="none"` 常用于 LoopAgent 内的子 agent，避免前序消息吞掉上下文。
- 6 类 callback 签名和模式见 `callbacks-and-events.md`。

**Docs**：https://adk.dev/agents/llm-agents/

---

## BaseAgent 子类化（确定性编排）

**用途**：用 Python 控制流自定义子 agent 的调度逻辑。零 LLM 成本，零延迟。

**Import**：`from google.adk.agents import BaseAgent`

**骨架**（继承并实现 `_run_async_impl`）：

```python
from typing import AsyncGenerator
from typing_extensions import override
from google.adk.agents import BaseAgent
from google.adk.agents.invocation_context import InvocationContext
from google.adk.events.event import Event

class MyPipeline(BaseAgent):
    @override
    async def _run_async_impl(
        self, ctx: InvocationContext
    ) -> AsyncGenerator[Event, None]:
        state = ctx.session.state
        for sub in self.sub_agents:
            async for event in sub.run_async(ctx):
                yield event
            if ctx.end_invocation:
                return
```

**关键属性**：

- `ctx.session.state` — 可读写的 session state（dict）
- `ctx.session.events` — 历史事件列表
- `ctx.session.id` — 当前 session ID（不要自建 UUID）
- `ctx.end_invocation` — bool，是否提前终止本次 invocation
- `self.sub_agents` — 构造时注册的子 agent 列表

**访问 artifact / save_artifact / load_artifact**：`InvocationContext` 不直接暴露 artifact 方法，需要用 `Context(ctx)` 包装成 `Context`（= `ToolContext`）。见 `context-and-state.md` 的 "Context 包装模式"。

**Docs**：https://adk.dev/agents/custom-agents/

---

## Resume-from-Failure 模式（项目模式）

BaseAgent 中记录已完成步骤，失败后从断点续跑：

```python
completed = list(state.get("pipeline_completed_steps", []))

for step_name, agent, should_run in steps:
    if step_name in completed:
        continue                    # 已完成，跳过
    if not should_run:
        completed.append(step_name)
        state["pipeline_completed_steps"] = list(completed)
        continue                    # 条件不满足，标记跳过

    async for event in agent.run_async(ctx):
        yield event

    if ctx.end_invocation:
        return

    completed.append(step_name)
    state["pipeline_completed_steps"] = list(completed)  # 持久化进度
```

**Why**：一次 invocation 中途失败（比如外部 API 超时）重跑时，不重复已完成的昂贵步骤。

**ADK 原生能力**：ADK Python 1.14.0+ 提供了 `ResumabilityConfig`（见 `runtime-and-deploy.md`），基于 `invocation_id` 做真正的断点续传。自建的 `pipeline_completed_steps` 模式粒度更粗但更容易控制；两者可并存。

---

## SequentialAgent

**用途**：按顺序依次执行所有 `sub_agents`。无条件分支。

**Import**：`from google.adk.agents import SequentialAgent`

```python
SequentialAgent(
    name="research_pipeline",
    description="...",
    sub_agents=[planner, validator, executor],
)
```

子 agent 通过 `session.state` 传递数据，上一个 agent 用 `output_key` 写入，下一个从 state 读。

**Docs**：https://adk.dev/agents/workflow-agents/sequential-agents/

---

## LoopAgent

**用途**：循环执行子 agent 序列，直到触发退出或达到 `max_iterations`。

**Import**：`from google.adk.agents import LoopAgent`

```python
LoopAgent(
    name="validation_loop",
    sub_agents=[validator, refiner],
    max_iterations=3,              # 硬上限，防死循环
)
```

**退出机制**：子 agent 的 tool 里调用 `tool_context.actions.escalate = True` 触发。

```python
from google.adk.tools import ToolContext

def exit_loop(tool_context: ToolContext) -> dict:
    tool_context.actions.escalate = True
    return {}
```

ADK 内置了同名的 `exit_loop` tool（见 `tools.md`），也可以直接在 agent 的 `tools=[exit_loop]` 里用。

**Docs**：https://adk.dev/agents/workflow-agents/loop-agents/

---

## ParallelAgent

**用途**：所有 sub_agents 并行执行，全部完成后才继续。

**Import**：`from google.adk.agents import ParallelAgent`

```python
ParallelAgent(
    name="parallel_analysis",
    sub_agents=[analyzer_a, analyzer_b],
)
```

**典型组合**：ParallelAgent 并行收集 → SequentialAgent 合并：

```python
root = SequentialAgent(
    name="root",
    sub_agents=[
        ParallelAgent(name="fanout", sub_agents=[a, b, c]),
        merger_agent,
    ],
)
```

**注意**：并行 agent 共享同一个 `session.state`，写同一个 key 时后写覆盖前写（非确定性）。让它们写不同 key 或最后由 merger 归并。

**Docs**：https://adk.dev/agents/workflow-agents/parallel-agents/

---

## Multi-Agent 系统（层级 + transfer）

**用途**：LLM 动态决定调用哪个 sub-agent。

**两种组合方式：**

### 方式一：`sub_agents=[...]` + LLM 自动 transfer

父 agent 把子 agent 列在 `sub_agents`，LLM 基于子 agent 的 `description` 决定 transfer 给谁：

```python
dispatcher = Agent(
    name="dispatcher",
    instruction="根据用户意图转交给合适的专家。",
    sub_agents=[billing_agent, support_agent, tech_agent],
)
```

LLM 可以通过内置的 `transfer_to_agent(agent_name)` tool（自动可用）路由。

### 方式二：`AgentTool` 包装

把 sub-agent 包装成 tool，父 agent 像调 tool 一样调子 agent（返回结果后继续自己的对话）：

```python
from google.adk.tools import AgentTool

parent = Agent(
    name="parent",
    tools=[
        my_func_tool,
        AgentTool(helper_agent),
    ],
)
```

**两者的区别：**

| 特性 | `sub_agents=[...]` + transfer | `AgentTool(sub)` |
|------|-------------------------------|------------------|
| 对话控制权 | 交给被 transfer 的 agent | 父 agent 拿到返回值后继续 |
| 响应对象 | 子 agent 直接响应用户 | 子 agent 响应父 agent |
| 典型用途 | 专家 handoff（客服分流） | 子能力查询（工具化） |

**Docs**：https://adk.dev/agents/multi-agents/

---

## Transfer 机制

**`transfer_to_agent`** — 让 LLM 显式指定转交目标：

```python
from google.adk.tools import transfer_to_agent, TransferToAgentTool
```

- `transfer_to_agent`：可直接放进 `tools=[...]` 的函数 tool
- `TransferToAgentTool`：类形式，可自定义

LLM 调 `transfer_to_agent(agent_name="billing_agent")` 后，runner 将对话转交给该 agent。

**触发条件**：在 `sub_agents` 列表中，或当前 agent 的 ancestor 链路中可找到目标。

参见 `tools.md` 的 "TransferToAgentTool" 条目。

---

## ADK 单父约束

**规则**：每个 agent 实例只能有一个父 agent。同一个 agent 要在多个父中使用，必须创建多个实例。

```python
# ❌ 错：helper 被两个父引用
helper = create_helper()
parent_a = Agent(tools=[AgentTool(helper)])
parent_b = Agent(tools=[AgentTool(helper)])  # 运行时报错

# ✅ 对：为每个父单独造实例
parent_a = Agent(tools=[AgentTool(create_helper())])
parent_b = Agent(tools=[AgentTool(create_helper())])
```

**建议**：把 agent 构造包在工厂函数里（如 `create_xxx_agent()`），避免模块级共享实例。

---

## RemoteA2aAgent（跨进程远程 agent）

**用途**：把另一个进程里暴露的 A2A agent 当成本地 sub-agent 使用。

**Import**：`from google.adk.agents import RemoteA2aAgent`

细节见 `advanced.md` 的 A2A 章节。

---

## __init__.py 导出惯例

项目约定：

### Sub-agent 包

```python
# app/sub_agents/{name}/__init__.py
from .agent import create_{name}_agent
__all__ = ["create_{name}_agent"]
```

Sub-agent 目录结构：
```
app/sub_agents/{name}/
├── __init__.py
├── agent.py          # create_{name}_agent() 工厂
├── prompt.py         # INSTRUCTION 常量
└── tools.py          # 本 agent 专属 tool（如果跨 agent 共享则放 shared_libraries/）
```

### App 入口

```python
# app/__init__.py
from .agent import app
__all__ = ["app"]
```

### Tools 包

```python
# app/tools/__init__.py
from .module_name import tool_func_1, tool_func_2
__all__ = ["tool_func_1", "tool_func_2"]
```

---

## Agent Configuration（YAML，实验性）

ADK 还支持 YAML 声明式配置 agent（仅 Gemini，实验性）：

- **用途**：不写 Python，用 YAML 描述 agent 结构
- **Docs**：https://adk.dev/agents/config/
- **API 参考**：https://adk.dev/api-reference/agentconfig/

本项目目前未采用此路线，留作参考。

---

## CLAUDE.md 相关约束

- 判断逻辑确定时（if/else 能决策）优先 `BaseAgent`，不用 `LlmAgent`
- `BaseAgent._run_async_impl(ctx: InvocationContext)` 中用 `Context(ctx)` 获取 artifact 访问
- 入口 callback `before_agent_callback` 是确定性设置 state 的最佳位置
- 每个 BaseAgent 入/出必须 log `INFO`（agent name + 耗时）
- 关键决策分支必须 log 决策依据（哪个 state key 值为真导致走哪条路径）

---

## 上游参考

- **Agents 总览**：https://adk.dev/agents/
- **LLM Agents**：https://adk.dev/agents/llm-agents/
- **Workflow Agents**：https://adk.dev/agents/workflow-agents/
- **Custom Agents**：https://adk.dev/agents/custom-agents/
- **Multi-Agent Systems**：https://adk.dev/agents/multi-agents/
- **Agent Configuration (YAML)**：https://adk.dev/agents/config/
- **源码根目录**：https://github.com/google/adk-python/tree/main/src/google/adk/agents
