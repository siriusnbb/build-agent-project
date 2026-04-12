# Google ADK Framework Reference Guide

本文档是 Google Agent Development Kit (ADK) 的核心用法参考。基于 uw-agents-dno 实际项目提炼。

---

## 1. Agent 类型总览

| 类型 | 用途 | import |
|------|------|--------|
| `Agent` | LLM 编排型 agent，根据 instruction 使用 tools 和 sub_agents | `from google.adk.agents import Agent` |
| `BaseAgent` | 确定性自定义 agent，用 Python 控制流编排子 agent | `from google.adk.agents import BaseAgent` |
| `SequentialAgent` | 按顺序依次执行所有 sub_agents | `from google.adk.agents import SequentialAgent` |
| `LoopAgent` | 循环执行 sub_agents，直到 `escalate=True` 或达到 max_iterations | `from google.adk.agents import LoopAgent` |
| `ParallelAgent` | 并行执行所有 sub_agents（项目中未使用但 ADK 支持） | `from google.adk.agents import ParallelAgent` |

---

## 2. Agent (LlmAgent) 定义方式

```python
from google.adk.agents import Agent
from google.adk.models import Gemini
from google.genai import types

agent = Agent(
    name="agent_name",                          # 唯一标识
    model=Gemini(
        model="gemini-2.0-flash",               # 模型名称
        retry_options=types.HttpRetryOptions(attempts=3),
    ),
    instruction=INSTRUCTION_STRING,             # str 或 Callable[[ReadonlyContext], str]
    description="Agent 描述文本",                # 用于 dispatcher 路由
    tools=[tool_func_1, tool_func_2, AgentTool(sub)],  # tool 列表
    sub_agents=[child_agent_1, child_agent_2],  # 子 agent 列表
    output_key="state_key_name",                # 可选：LLM 输出自动写入此 state key
    include_contents="none",                    # 可选："none" 排除前序消息
    before_agent_callback=callback_func,        # 可选：执行前回调
)
```

### 关键参数说明

- **instruction**: 可以是字符串或接受 `ReadonlyContext` 的 callable（动态 prompt）
- **output_key**: 设置后 LLM 的最终文本响应会自动写入 `state[output_key]`；当 tool 直接写 state 时应省略
- **include_contents**: 设为 `"none"` 可避免前序消息占满上下文（用于 LoopAgent 内的子 agent）

### 动态 Instruction 模式

```python
from google.adk.agents.readonly_context import ReadonlyContext

def dynamic_instruction(ctx: ReadonlyContext) -> str:
    state = ctx.state
    if state.get("condition"):
        return PROMPT_A
    return PROMPT_B

Agent(instruction=dynamic_instruction, ...)
```

---

## 3. BaseAgent 子类化

用于构建确定性编排逻辑（如 Pipeline）：

```python
from google.adk.agents import BaseAgent
from google.adk.agents.invocation_context import InvocationContext
from google.adk.events.event import Event
from typing import AsyncGenerator
from typing_extensions import override

class PipelineAgent(BaseAgent):
    """确定性流水线 agent。"""

    @override
    async def _run_async_impl(
        self, ctx: InvocationContext
    ) -> AsyncGenerator[Event, None]:
        state = ctx.session.state

        for agent in self.sub_agents:
            async for event in agent.run_async(ctx):
                yield event

            if ctx.end_invocation:
                return
```

### InvocationContext 关键属性

- `ctx.session.state` — 可读写的 session state 字典
- `ctx.end_invocation` — bool，是否提前终止
- `self.sub_agents` — 注册的子 agent 列表

### Resume-from-Failure 模式

```python
completed = list(state.get("pipeline_completed_steps", []))

for step_name, agent, should_run in steps:
    if step_name in completed:
        continue                    # 跳过已完成
    if not should_run:
        completed.append(step_name)
        state["pipeline_completed_steps"] = list(completed)
        continue                    # 条件不满足，标记跳过

    async for event in agent.run_async(ctx):
        yield event

    if ctx.end_invocation:
        return

    completed.append(step_name)
    state["pipeline_completed_steps"] = list(completed)
```

---

## 4. Tool 定义规范

### 函数签名

```python
from google.adk.tools import ToolContext

def my_tool(
    param1: str,
    param2: int,
    tool_context: ToolContext,       # 必须是最后一个参数
) -> dict:
    """Tool 的功能描述（此 docstring 即为 LLM 看到的 tool description）。

    Args:
        param1: 参数1说明
        param2: 参数2说明

    Returns:
        包含 status 和结果数据的字典。
    """
    # 读取 state
    value = tool_context.state.get("key_name")

    # 写入 state
    tool_context.state["output_key"] = result

    return {"status": "success", "message": "...", "data": result}
```

### 关键规则

1. `tool_context: ToolContext` 必须是最后一个参数（ADK 自动注入）
2. docstring 即为 LLM 的 tool description，要准确描述功能
3. 返回值通常为 dict，LLM 可看到完整返回内容
4. state 读写通过 `tool_context.state`

### 保存 Artifact

```python
from google.genai import types

async def save_artifact(tool_context: ToolContext) -> dict:
    data_bytes = json.dumps(data, ensure_ascii=False).encode("utf-8")
    part = types.Part.from_bytes(data=data_bytes, mime_type="application/json")
    version = await tool_context.save_artifact("filename.json", part)
    return {"status": "success", "version": version}
```

---

## 5. AgentTool 模式

将 sub-agent 包装为 tool，使父 agent 可以像调用 tool 一样调用子 agent：

```python
from google.adk.tools import AgentTool

helper_agent = Agent(name="helper", ...)

parent_agent = Agent(
    name="parent",
    tools=[
        my_function_tool,
        AgentTool(helper_agent),    # 包装为 tool
    ],
)
```

### ADK 单父约束

每个 agent 只能有一个父 agent。如果需要在多个父 agent 中使用同一个 agent，必须创建多个实例：

```python
# 正确：为不同父 agent 创建不同实例
helper_for_parent_a = create_helper_agent()
helper_for_parent_b = create_helper_agent()

parent_a = Agent(tools=[AgentTool(helper_for_parent_a)])
parent_b = Agent(tools=[AgentTool(helper_for_parent_b)])
```

---

## 6. Callback 机制

### before_agent_callback — 初始化 session state

```python
from google.adk.agents.callback_context import CallbackContext

async def initialize_session_state(callback_context: CallbackContext) -> None:
    state = callback_context.state
    defaults = {
        "company_name": "",
        "has_additional_files": False,
        "pipeline_completed_steps": [],
    }
    for key, default in defaults.items():
        if key not in state:
            state[key] = default

# 附加到 root agent
root_agent = Agent(before_agent_callback=initialize_session_state, ...)
```

### CallbackContext 属性

- `callback_context.state` — 可读写的 session state
- `callback_context.agent_name` — 当前 agent 名称
- `callback_context.user_content` — 用户输入内容

### 可用回调类型

| 回调 | 时机 | 典型用途 |
|------|------|---------|
| `before_agent_callback` | agent 执行前 | state 初始化、日志 |
| `after_agent_callback` | agent 执行后 | 清理、结果校验 |
| `before_tool_callback` | tool 执行前 | 权限检查、参数预处理 |

---

## 7. Session State 管理

### 集中式 StateKeys 常量

```python
# app/config.py
class StateKeys:
    COMPANY_NAME = "company_name"
    PIPELINE_COMPLETED_STEPS = "pipeline_completed_steps"
    STRUCTURED_REPORT = "structured_report"
    # ... 所有 key 集中定义
```

### 各处访问方式

| 场景 | 读写方式 |
|------|---------|
| Tool 中 | `tool_context.state[StateKeys.KEY]` |
| BaseAgent._run_async_impl 中 | `ctx.session.state[StateKeys.KEY]` |
| Callback 中 | `callback_context.state[StateKeys.KEY]` |
| 动态 instruction 中 | `ctx.state[StateKeys.KEY]`（只读） |

---

## 8. SequentialAgent + LoopAgent 组合

### SequentialAgent — 顺序执行

```python
SequentialAgent(
    name="research_pipeline",
    description="...",
    sub_agents=[
        planner_agent,
        validation_loop,        # 可嵌套 LoopAgent
        executor_agent,
    ],
)
```

### LoopAgent — 迭代验证

```python
LoopAgent(
    name="validation_loop",
    sub_agents=[
        validator_agent,        # 评估质量
        refiner_agent,          # 改进或退出
    ],
    max_iterations=3,           # 防止无限循环
)
```

### 退出 LoopAgent

在 refiner_agent 的 tool 中调用：

```python
def exit_loop(tool_context: ToolContext) -> dict:
    tool_context.actions.escalate = True    # 触发 LoopAgent 退出
    return {}
```

---

## 9. __init__.py 导出惯例

### Sub-agent 包

```python
# app/sub_agents/{name}/__init__.py
from .agent import create_{name}_agent

__all__ = ["create_{name}_agent"]
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

## 10. App 包装与入口

```python
from google.adk.apps import App

root_agent = Agent(name="root_agent", ...)

app = App(
    root_agent=root_agent,
    name="app",
)
```

部署入口（Agent Engine）：

```python
# app/agent_engine_app.py
from google.adk.agents.vertex_ai_agent_engine import AdkApp

class AgentEngineApp(AdkApp):
    def set_up(self):
        import vertexai
        vertexai.init()
        # 初始化 telemetry、logging 等

agent_engine = AgentEngineApp(agent=root_agent, enable_telemetry=True)
```

---

## 11. Pydantic Schema 与 Agent 集成

Agent 的 tool 可以直接使用 Pydantic 模型作为参数类型，ADK 会自动约束 LLM 输出格式：

```python
from pydantic import BaseModel, Field, field_validator

class ReportSchema(BaseModel):
    title: str = Field(description="报告标题")
    sections: list[str] = Field(default_factory=list)

    @field_validator("sections", mode="before")
    @classmethod
    def ensure_list(cls, v):
        if isinstance(v, str):
            return [v]
        return v

# 在 tool 中使用
def format_report(report: dict, tool_context: ToolContext) -> dict:
    validated = ReportSchema.model_validate(report)
    tool_context.state["report"] = validated.model_dump()
    return {"status": "success"}
```
