# Project Architecture Patterns Reference

基于生产项目提炼的架构惯例和代码模式参考。Planner 根据需求选择合适的架构模式，以下为各模式的默认实现。

---

## 架构实现通用约定

以下约定不因架构模式选择而变化，所有模式都适用：

- **Sub-agents 目录**：`app/sub_agents/{name}/`（多 agent 模式时）
- **Tools**：带 `ToolContext` 末参数的函数
- **Session state**：集中式 `StateKeys` 类于 `config.py`
- **AgentTool**：包装 sub-agent 为 tool（遵循 ADK 单父约束）

详细实现参见下面各章节。

---

## 1. 目录结构惯例

```
{project-name}/
├── app/                              # 主应用代码
│   ├── __init__.py                   # .env 加载、logging 初始化、export app
│   ├── agent.py                      # Root agent 定义 + root-level tools
│   ├── agent_engine_app.py           # Agent Engine 部署封装
│   ├── config.py                     # 环境变量 + StateKeys 常量类
│   ├── prompt.py                     # Root agent instruction
│   ├── shared_libraries/
│   │   ├── __init__.py
│   │   └── callbacks.py             # session state 初始化等回调
│   ├── app_utils/
│   │   ├── deploy.py                # 部署脚本
│   │   ├── telemetry.py             # OpenTelemetry 设置
│   │   ├── typing.py                # 公共 Pydantic 模型
│   │   └── .requirements.txt        # 部署用依赖（自动生成）
│   ├── tools/                        # 共享 tool 模块
│   │   ├── __init__.py
│   │   └── {tool_module}.py
│   ├── sub_agents/                   # 所有子 agent（多 agent 模式时使用）
│   │   ├── __init__.py               # 空
│   │   ├── pipeline/                 # 确定性编排 agent（Deterministic Pipeline 时使用）
│   │   ├── {agent_name}/            # 各功能 agent
│   │   └── ...
│   └── templates/                    # 模板文件（Excel 等）
│
├── tests/
│   ├── unit/                         # 单元测试（mock）
│   ├── integration/                  # 集成测试（Runner）
│   ├── eval/                         # Agent 评估
│   │   ├── eval_config.json
│   │   └── evalsets/
│   │       └── *.evalset.json
│   └── load_test/                    # 负载测试（Locust）
│       └── load_test.py
│
├── deployment/
│   └── terraform/                    # IaC
│       ├── *.tf
│       ├── dev/                      # Dev 环境
│       ├── sql/
│       └── vars/
│
├── .github/workflows/                # CI/CD
│   ├── pr_checks.yaml
│   └── deploy-to-dev.yaml
│
├── Makefile                          # 标准命令集
├── pyproject.toml                    # 项目配置
├── .env / .env.example               # 环境变量
├── .env.dev / .env.dev.example
└── README.md
```

---

## 2. Sub-Agent 目录惯例

每个 sub-agent 位于 `app/sub_agents/{name}/`，包含：

```
{agent_name}/
├── __init__.py        # 导出 create_{agent_name}_agent
├── agent.py           # Agent 工厂函数 + agent 专属 tools
├── prompt.py          # Instruction 字符串常量
└── [schema.py]        # （可选）Pydantic 数据模型
```

### __init__.py 模板

```python
from .agent import create_{name}_agent

__all__ = ["create_{name}_agent"]
```

### agent.py 模板

```python
from google.adk.agents import Agent
from google.adk.models import Gemini
from google.adk.tools import ToolContext
from google.genai import types

from app.config import DEFAULT_MODEL_NAME, DEFAULT_MODEL_RETRIES, StateKeys
from .prompt import {NAME}_INSTRUCTION


def create_{name}_agent() -> Agent:
    """Create and return a {Name} Agent instance."""
    return Agent(
        name="{name}_agent",
        model=Gemini(
            model=DEFAULT_MODEL_NAME,
            retry_options=types.HttpRetryOptions(attempts=DEFAULT_MODEL_RETRIES),
        ),
        description="...",
        instruction={NAME}_INSTRUCTION,
        tools=[tool_func_1, tool_func_2],
        output_key="{name}_result",          # 可选
    )


def tool_func_1(param: str, tool_context: ToolContext) -> dict:
    """Tool description."""
    tool_context.state[StateKeys.OUTPUT_KEY] = param
    return {"status": "success"}
```

### prompt.py 模板

```python
"""{Name} Agent instructions."""

{NAME}_INSTRUCTION = """あなたは {role description} です。

## タスク
...

## 使用可能なツール
...

## 制約
...
"""
```

---

## 3. StateKeys 集中管理模式

所有 session state key 在 `app/config.py` 中以类常量形式集中定义：

```python
class StateKeys:
    """Session state key constants."""
    # Input
    COMPANY_NAME = "company_name"
    HAS_ADDITIONAL_FILES = "has_additional_files"

    # Pipeline tracking
    PIPELINE_COMPLETED_STEPS = "pipeline_completed_steps"

    # Intermediate results
    RESEARCH_RESULT = "research_result"
    STRUCTURED_REPORT = "structured_report"

    # Output
    ANALYSIS_RESULT = "analysis_result"
```

### 命名惯例

- 全大写 + 下划线
- 按功能分组（Input / Pipeline / Intermediate / Output）
- 值为 snake_case 字符串

### 初始化

在 `shared_libraries/callbacks.py` 的 `initialize_session_state` 回调中为所有 key 设置默认值：

```python
defaults = {
    StateKeys.COMPANY_NAME: "",
    StateKeys.PIPELINE_COMPLETED_STEPS: [],
    StateKeys.STRUCTURED_REPORT: "",
}
for key, default in defaults.items():
    if key not in state:
        state[key] = default
```

---

## 4. 架构模式实现参考

Planner 在 Phase 1 选择架构模式后，后续 agent 按该模式实现。以下为各模式的参考模板。

### 4.1 Single Agent 模式

最简架构：一个 LLM agent + tools，无 sub-agent。适合单一任务、tool ≤ 5 的场景。

```python
# app/agent.py
from google.adk.agents import Agent
from google.adk.apps import App
from google.adk.models import Gemini
from google.adk.tools import ToolContext
from google.genai import types

from app.config import DEFAULT_MODEL_NAME, DEFAULT_MODEL_RETRIES, StateKeys
from app.prompt import ROOT_INSTRUCTION


def my_tool(param: str, tool_context: ToolContext) -> dict:
    """Tool 描述。"""
    tool_context.state[StateKeys.OUTPUT] = param
    return {"status": "success"}


root_agent = Agent(
    name="root_agent",
    model=Gemini(
        model=DEFAULT_MODEL_NAME,
        retry_options=types.HttpRetryOptions(attempts=DEFAULT_MODEL_RETRIES),
    ),
    instruction=ROOT_INSTRUCTION,
    tools=[my_tool],
)

app = App(root_agent=root_agent, name="app")
```

**目录结构**（无 `sub_agents/`）：

```
app/
├── __init__.py
├── agent.py          # root agent + tools
├── config.py         # StateKeys + 环境变量
└── prompt.py         # instruction
```

### 4.2 Sequential Pipeline 模式

使用 ADK 内置 SequentialAgent 按固定顺序执行 sub-agents。适合无条件分支的多步骤流程。

```python
# app/agent.py
from google.adk.agents import Agent, SequentialAgent
from google.adk.apps import App
from google.adk.models import Gemini
from google.genai import types

from app.config import DEFAULT_MODEL_NAME, DEFAULT_MODEL_RETRIES
from app.sub_agents.step_a import create_step_a_agent
from app.sub_agents.step_b import create_step_b_agent
from app.sub_agents.step_c import create_step_c_agent

root_agent = SequentialAgent(
    name="root_agent",
    description="按顺序执行 A → B → C",
    sub_agents=[
        create_step_a_agent(),
        create_step_b_agent(),
        create_step_c_agent(),
    ],
)

app = App(root_agent=root_agent, name="app")
```

### 4.3 Deterministic Pipeline 模式

BaseAgent 子类，支持条件分支、跳步、resume-from-failure。适合复杂确定性流程。

```python
class PipelineAgent(BaseAgent):
    @override
    async def _run_async_impl(self, ctx: InvocationContext):
        state = ctx.session.state
        completed = list(state.get(StateKeys.PIPELINE_COMPLETED_STEPS, []))

        steps = [
            ("step_name", self.sub_agents[IDX], condition_bool),
            ...
        ]

        for step_name, agent, should_run in steps:
            # 1. 已完成则跳过
            if step_name in completed:
                continue

            # 2. 输出已存在（幂等保护）
            if _has_output(state, step_name):
                completed.append(step_name)
                state[StateKeys.PIPELINE_COMPLETED_STEPS] = list(completed)
                continue

            # 3. 条件不满足则跳过
            if not should_run:
                completed.append(step_name)
                state[StateKeys.PIPELINE_COMPLETED_STEPS] = list(completed)
                continue

            # 4. 执行
            async for event in agent.run_async(ctx):
                yield event

            if ctx.end_invocation:
                return

            # 5. 记录完成
            completed.append(step_name)
            state[StateKeys.PIPELINE_COMPLETED_STEPS] = list(completed)
```

幂等证据映射：

```python
_STEP_OUTPUT_EVIDENCE = {
    "step_a": [StateKeys.STEP_A_RESULT],
    "step_b": [StateKeys.STEP_B_RESULT],
}
```

### 4.4 Hierarchical 模式

Root Agent 通过 AgentTool 将 sub-agent 包装为 tool，由 LLM 动态决定调用哪个。适合多领域、无固定流程的场景。

```python
# app/agent.py
from google.adk.agents import Agent
from google.adk.apps import App
from google.adk.models import Gemini
from google.adk.tools import AgentTool
from google.genai import types

from app.config import DEFAULT_MODEL_NAME, DEFAULT_MODEL_RETRIES
from app.prompt import ROOT_INSTRUCTION
from app.sub_agents.researcher import create_researcher_agent
from app.sub_agents.writer import create_writer_agent
from app.sub_agents.reviewer import create_reviewer_agent

root_agent = Agent(
    name="root_agent",
    model=Gemini(
        model=DEFAULT_MODEL_NAME,
        retry_options=types.HttpRetryOptions(attempts=DEFAULT_MODEL_RETRIES),
    ),
    instruction=ROOT_INSTRUCTION,
    tools=[
        AgentTool(create_researcher_agent()),
        AgentTool(create_writer_agent()),
        AgentTool(create_reviewer_agent()),
    ],
)

app = App(root_agent=root_agent, name="app")
```

> 注意 ADK 单父约束：每个 agent 实例只能有一个父。需要在多处使用时，调用工厂函数创建不同实例。

### 4.5 Loop Refinement 模式

LoopAgent 迭代执行直到质量达标。适合需要验证-改进循环的场景。

```python
# app/agent.py
from google.adk.agents import Agent, LoopAgent
from google.adk.apps import App
from google.adk.tools import ToolContext

from app.sub_agents.generator import create_generator_agent
from app.sub_agents.evaluator import create_evaluator_agent


# evaluator agent 的 tool 中退出循环
def approve_or_refine(quality_score: float, tool_context: ToolContext) -> dict:
    """评估质量，达标则退出循环。"""
    if quality_score >= 0.8:
        tool_context.actions.escalate = True  # 退出 LoopAgent
        return {"status": "approved"}
    return {"status": "needs_refinement", "feedback": "..."}


root_agent = LoopAgent(
    name="root_agent",
    description="迭代生成和评估，直到质量达标",
    sub_agents=[
        create_generator_agent(),
        create_evaluator_agent(),  # 内含 approve_or_refine tool
    ],
    max_iterations=5,  # 防止无限循环
)

app = App(root_agent=root_agent, name="app")
```

### 4.6 Parallel Fan-out 模式

ParallelAgent 并行执行独立子任务。适合多个无依赖关系的子任务需要同时处理的场景。

```python
# app/agent.py
from google.adk.agents import Agent, ParallelAgent, SequentialAgent
from google.adk.apps import App

from app.sub_agents.analyzer_a import create_analyzer_a_agent
from app.sub_agents.analyzer_b import create_analyzer_b_agent
from app.sub_agents.merger import create_merger_agent

# 并行执行分析，然后合并结果
parallel_step = ParallelAgent(
    name="parallel_analysis",
    description="并行执行多路分析",
    sub_agents=[
        create_analyzer_a_agent(),
        create_analyzer_b_agent(),
    ],
)

root_agent = SequentialAgent(
    name="root_agent",
    description="并行分析后合并结果",
    sub_agents=[
        parallel_step,
        create_merger_agent(),
    ],
)

app = App(root_agent=root_agent, name="app")
```

### 4.7 Hybrid 模式

组合以上模式构建复杂系统。典型组合：

- **LLM Root + Deterministic Pipeline**: Root Agent 收集输入，Pipeline 确定性执行
- **Sequential + Loop**: 顺序流程中嵌套迭代验证环节
- **Parallel + Sequential**: 并行收集数据后顺序处理

```python
# 示例：LLM Root → Deterministic Pipeline（含 Loop 子步骤）
root_agent = Agent(
    name="root_agent",
    model=Gemini(model=DEFAULT_MODEL_NAME, ...),
    instruction=ROOT_INSTRUCTION,
    tools=[collect_input, AgentTool(pipeline_agent)],
)

# pipeline_agent 是 BaseAgent 子类，其中某步骤使用 LoopAgent
# 具体实现参考 4.3 + 4.5 组合
```

选择组合时的原则：
- 优先使用最简单的能满足需求的模式
- 组合不超过 2-3 层嵌套，避免过度复杂
- 每种组合都应在 build-plan.md 中明确说明理由

---

## 5. PipelineAgent Resume-from-Failure 详解

> 此节是 4.3 Deterministic Pipeline 模式的详细实现参考。

确定性编排 agent（BaseAgent 子类），支持断点续传：

```python
class PipelineAgent(BaseAgent):
    @override
    async def _run_async_impl(self, ctx: InvocationContext):
        state = ctx.session.state
        completed = list(state.get(StateKeys.PIPELINE_COMPLETED_STEPS, []))

        steps = [
            ("step_name", self.sub_agents[IDX], condition_bool),
            ...
        ]

        for step_name, agent, should_run in steps:
            # 1. 已完成则跳过
            if step_name in completed:
                continue

            # 2. 输出已存在（幂等保护）
            if _has_output(state, step_name):
                completed.append(step_name)
                state[StateKeys.PIPELINE_COMPLETED_STEPS] = list(completed)
                continue

            # 3. 条件不满足则跳过
            if not should_run:
                completed.append(step_name)
                state[StateKeys.PIPELINE_COMPLETED_STEPS] = list(completed)
                continue

            # 4. 执行
            async for event in agent.run_async(ctx):
                yield event

            if ctx.end_invocation:
                return

            # 5. 记录完成
            completed.append(step_name)
            state[StateKeys.PIPELINE_COMPLETED_STEPS] = list(completed)
```

### 幂等证据映射

```python
_STEP_OUTPUT_EVIDENCE = {
    "step_a": [StateKeys.STEP_A_RESULT],
    "step_b": [StateKeys.STEP_B_RESULT],
}
```

---

## 6. Pydantic Schema 验证模式

### 结构化输出验证

```python
from pydantic import BaseModel, Field, field_validator, model_validator

class OutputSchema(BaseModel):
    field_a: str = Field(description="字段 A")
    field_b: list[float] = Field(default_factory=list)

    @field_validator("field_b", mode="before")
    @classmethod
    def normalize(cls, v):
        # 输入标准化逻辑
        return v

    @model_validator(mode="before")
    @classmethod
    def coerce_defaults(cls, v):
        # None → 默认值
        if isinstance(v, dict):
            for k in ("field_a",):
                if v.get(k) is None:
                    v[k] = ""
        return v
```

### 在 Tool 中使用

```python
def format_output(data: dict, tool_context: ToolContext) -> dict:
    validated = OutputSchema.model_validate(data)
    tool_context.state[StateKeys.OUTPUT] = validated.model_dump()
    return {"status": "success"}
```

---

## 7. 测试模式

### 单元测试

```python
# tests/unit/test_{agent_name}.py
from unittest.mock import MagicMock
from google.adk.tools import ToolContext
from app.config import StateKeys

def test_tool_function():
    tool_context = MagicMock(spec=ToolContext)
    tool_context.state = {StateKeys.INPUT_KEY: "value"}

    result = my_tool("param", tool_context=tool_context)

    assert result["status"] == "success"
    assert tool_context.state[StateKeys.OUTPUT_KEY] == expected

def test_agent_creation():
    agent = create_my_agent()
    assert agent.name == "my_agent"
    assert len(agent.tools) >= 1
```

### 集成测试

```python
# tests/integration/test_agent.py
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types
from app.agent import root_agent

def test_agent_stream():
    session_service = InMemorySessionService()
    session = session_service.create_session_sync(
        user_id="test", app_name="test"
    )
    runner = Runner(
        agent=root_agent,
        session_service=session_service,
        app_name="test",
    )
    message = types.Content(
        role="user",
        parts=[types.Part.from_text(text="Hello")],
    )
    events = list(runner.run(
        new_message=message,
        user_id="test",
        session_id=session.id,
    ))
    assert len(events) > 0
```

### Eval 模式

**evalset.json 格式：**

```json
{
  "eval_set_id": "basic_eval",
  "name": "Basic Evaluation",
  "eval_cases": [
    {
      "eval_id": "test_case_1",
      "conversation": [
        {
          "user_content": {
            "parts": [{"text": "ユーザー入力テキスト"}]
          }
        }
      ],
      "session_input": {
        "app_name": "app",
        "user_id": "eval_user",
        "state": {}
      }
    }
  ]
}
```

**eval_config.json 格式：**

```json
{
  "criteria": {
    "rubric_based_final_response_quality_v1": {
      "threshold": 0.6,
      "rubrics": [
        {
          "rubricId": "relevance",
          "rubricContent": {
            "textProperty": "The response directly addresses the user's query..."
          }
        }
      ]
    }
  }
}
```

### 负载测试（Locust）

```python
# tests/load_test/load_test.py
from locust import HttpUser, between, task

class ChatUser(HttpUser):
    wait_time = between(1, 3)

    @task
    def chat(self):
        headers = {"Authorization": f"Bearer {os.environ['_AUTH_TOKEN']}"}
        data = {"class_method": "async_stream_query", "input": {...}}
        with self.client.post(url, headers=headers, json=data, stream=True) as resp:
            for line in resp.iter_lines():
                pass
```

---

## 8. Makefile 标准命令集

```makefile
ENV ?=
ENV_FILE := $(if $(ENV),.env.$(ENV),.env)
ifneq (,$(wildcard $(ENV_FILE)))
    include $(ENV_FILE)
    export
endif

install:
	uv sync

playground:
	uv run adk web --port 8501 ./app

test:
	uv run pytest tests/unit tests/integration

lint:
	uv run codespell
	uv run ruff format --check .
	uv run ruff check .
	uv run ty check

eval:
	uv run adk eval --eval_set_file=$(EVALSET) --config=$(EVAL_CONFIG) ./app

deploy:
	uv export --no-hashes ... > app/app_utils/.requirements.txt
	uv run python -m app.app_utils.deploy --project=... --location=...

setup-dev-env:
	cd deployment/terraform/dev && terraform init && terraform apply -var-file=../vars/env.tfvars
```

---

## 9. pyproject.toml 结构

```toml
[project]
name = "project-name"
version = "0.1.0"
requires-python = ">=3.10,<3.14"
dependencies = [
    "google-adk>=1.27.0,<2.0.0",
    # ... 核心依赖
]

[dependency-groups]
dev = ["pytest", "pytest-asyncio", "nest-asyncio"]

[project.optional-dependencies]
eval = ["google-adk[eval]"]
lint = ["ruff", "ty", "codespell"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["app"]

[tool.pytest.ini_options]
pythonpath = "."
asyncio_default_fixture_loop_scope = "function"

[tool.ruff]
line-length = 88
target-version = "py310"

[tool.ruff.lint]
select = ["E", "F", "W", "I", "C", "B", "UP", "RUF"]
ignore = ["E501", "C901"]

[tool.ruff.lint.isort]
known-first-party = ["app"]

[tool.ty]
[tool.ty.environment]
python-version = "3.10"
[tool.ty.rules]
unresolved-import = "ignore"
```

---

## 10. app/__init__.py 惯例

```python
import logging
import os
from pathlib import Path
from dotenv import load_dotenv

# 加载 .env
_env_path = Path(__file__).resolve().parent.parent / ".env"
load_dotenv(_env_path)

# 配置 logging
_log_level = getattr(logging, (os.environ.get("LOG_LEVEL") or "INFO").upper())
logging.basicConfig(level=_log_level, force=True)

from .agent import app
__all__ = ["app"]
```

---

## 11. .env 文件惯例

### .env.example（tracked，值为空）

```
GOOGLE_CLOUD_PROJECT=
GOOGLE_CLOUD_LOCATION=
STORAGE_MODE=local
GCS_BUCKET_NAME=
LOGS_BUCKET_NAME=
SERVICE_ACCOUNT=
LOG_LEVEL=INFO
```

### .env（gitignored，实际值）

```
GOOGLE_CLOUD_PROJECT=my-project-id
GOOGLE_CLOUD_LOCATION=asia-northeast1
STORAGE_MODE=gcs
GCS_BUCKET_NAME=my-bucket
...
```

### .gitignore 相关条目

```
.env
.env.dev
!.env.example
!.env.dev.example
```
