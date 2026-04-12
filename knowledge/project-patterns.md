# Project Architecture Patterns Reference

本文档是基于 uw-agents-dno 项目提炼的架构惯例和代码模式参考。所有新项目应遵循这些模式。

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
│   ├── sub_agents/                   # 所有子 agent
│   │   ├── __init__.py               # 空
│   │   ├── pipeline/                 # 确定性编排 agent
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

## 4. PipelineAgent Resume-from-Failure 模式

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

## 5. Pydantic Schema 验证模式

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

## 6. 测试模式

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

## 7. Makefile 标准命令集

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

## 8. pyproject.toml 结构

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

## 9. app/__init__.py 惯例

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

## 10. .env 文件惯例

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
