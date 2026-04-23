# ADK Runtime & Deploy

> App / Runner / RunConfig / ResumabilityConfig / CLI / 部署目标 / Auth / Telemetry。
>
> **Last synced:** 2026-04-23 · **adk-python commit:** `62d7ee0` · **Upstream:** https://adk.dev/runtime/ · https://adk.dev/deploy/ · https://adk.dev/observability/

---

## 启动链路总览

```
开发：  agent.py → App → adk web / adk run → 本地
生产：  agent.py → App → Runner(session/artifact/memory/plugins) → Agent Engine / Cloud Run / GKE
```

关键装配对象：
- `App`（app 描述）→ 最终打包的单元
- `Runner`（执行器）→ 装配所有 service 和 plugin
- `RunConfig`（单次运行参数）→ 流式、语音、函数调用上限等
- `ResumabilityConfig`（断点续传）→ 按 invocation_id 恢复

---

## `App`

**用途**：打包 root_agent + 元数据，部署的最小单元。

**Import**：`from google.adk.apps import App, ResumabilityConfig`

```python
from google.adk.apps import App

root_agent = Agent(name="root_agent", ...)

app = App(
    root_agent=root_agent,
    name="my_app",
)
```

导出约定（项目）：`app/__init__.py` → `from .agent import app`，`__all__ = ["app"]`。

**源码**：`src/google/adk/apps/__init__.py`

---

## `Runner` / `InMemoryRunner`

**用途**：执行 agent。装配 session / artifact / memory 三家 service + plugins。

**Import**：`from google.adk.runners import Runner, InMemoryRunner`

（注意：`runners` 是顶层模块文件，不是子目录。）

```python
from google.adk.runners import Runner
from google.adk.sessions import VertexAiSessionService
from google.adk.artifacts import GcsArtifactService
from google.adk.memory import VertexAiMemoryBankService
from google.adk.plugins import LoggingPlugin

runner = Runner(
    agent=root_agent,
    app_name="my_app",
    session_service=VertexAiSessionService(project=..., location=...),
    artifact_service=GcsArtifactService(bucket_name=...),
    memory_service=VertexAiMemoryBankService(...),
    plugins=[LoggingPlugin()],
)

# 运行
async for event in runner.run_async(
    user_id="user1",
    session_id=session.id,
    new_message=types.Content(parts=[types.Part(text="hello")]),
):
    print(event)
```

### `InMemoryRunner` — 开发速记

等价于 `Runner` + 所有 `InMemory*Service`，单测 / POC 用：

```python
from google.adk.runners import InMemoryRunner
runner = InMemoryRunner(agent=root_agent, app_name="my_app")
```

**源码**：`src/google/adk/runners.py`

---

## `RunConfig`

**用途**：控制单次运行的参数（流式、语音、函数调用上限、LLM 调用上限等）。

**Import**：`from google.adk.agents import RunConfig`

**典型字段**（以源码为准）：
- `response_modalities` — `["TEXT"]` / `["AUDIO"]` 等（Live API）
- `streaming_mode` — 流式模式开关
- `max_llm_calls` — 单次 invocation 最多调几次 LLM（防失控）
- `save_input_blobs_as_artifacts` — 是否自动把 user 上传的 binary 存成 artifact

```python
async for event in runner.run_async(
    ...,
    run_config=RunConfig(max_llm_calls=20),
):
    ...
```

**Docs**：https://adk.dev/runtime/runconfig/

---

## `ResumabilityConfig`（断点续传，1.14.0+）

**用途**：一次 invocation 中途中断后，用相同 `invocation_id` 恢复，跳过已完成步骤。

**Import**：`from google.adk.apps import ResumabilityConfig`

**典型用法**（形式，以上游为准）：

```python
app = App(
    root_agent=root_agent,
    name="my_app",
    resumability_config=ResumabilityConfig(
        enabled=True,
        ttl_hours=24,
    ),
)
```

恢复时用同一个 `invocation_id` 调 `runner.run_async`，ADK 会读取上次状态接续。

与自建的 `pipeline_completed_steps` 模式（见 `agents.md`）不冲突，可叠加。

**Docs**：https://adk.dev/runtime/resume/

---

## ADK CLI

所有命令都是 `adk <command>`。完整清单：https://adk.dev/api-reference/cli/

| 命令 | 用途 |
|------|------|
| `adk create <dir>` | 脚手架新 agent 项目 |
| `adk web` | 启本地 dev UI（浏览器聊天界面，默认 localhost:8000） |
| `adk run <path>` | 单次 CLI 交互（带会话保存/恢复/回放，默认 SQLite `.adk/session.db`） |
| `adk api_server <path>` | 启 FastAPI REST + SSE 服务（`/list-apps`, `/run`, `/run_sse`） |
| `adk deploy agent_engine <path>` | 部署到 Vertex AI Agent Engine |
| `adk deploy cloud_run <path>` | 部署到 Cloud Run |
| `adk deploy gke <path>` | 部署到 GKE |
| `adk eval <path>` | 跑单次评估 |
| `adk eval_set create/add_eval_case/generate_eval_cases` | 评估集管理 |
| `adk conformance record/test` | 一致性回归（录制 / 重放） |
| `adk optimize` | 用 GEPA 自动优化 instruction |
| `adk migrate` | 版本迁移工具 |

**项目 Make 入口**（CLAUDE.md）：
- `make install` / `make playground`（= `adk web`）/ `make test` / `make lint` / `make eval` / `make deploy`

**Docs**：
- CLI 总入口：https://adk.dev/api-reference/cli/
- Web UI：https://adk.dev/runtime/web-interface/
- Command line：https://adk.dev/runtime/command-line/
- API server：https://adk.dev/runtime/api-server/

---

## Deployment Targets

三种部署目标对比：

| 目标 | 适用 | 运维 | Session / Memory 原生支持 | CLI 命令 |
|------|------|------|--------------------------|---------|
| **Vertex AI Agent Engine** | 希望最少运维、最快上线 | GCP 全托管 | ✅ `VertexAiSessionService` + `VertexAiMemoryBankService` 开箱 | `adk deploy agent_engine` |
| **Cloud Run** | 需要 HTTP 接口、已有 Cloud Run 栈、按需伸缩 | 需自己配 session backend | 需 `DatabaseSessionService` 或 `VertexAiSessionService` | `adk deploy cloud_run` |
| **GKE** | 已有 K8s 栈、细粒度控制、多租户隔离 | 自运维 | 同 Cloud Run | `adk deploy gke` |

**详细流程**：见项目的 `../gcp-deployment-guide.md`。

---

### Vertex AI Agent Engine 入口（`AgentEngineApp`）

**项目约定**：部署入口文件 `app/agent_engine_app.py`。

`AgentEngineApp` 是 `AdkApp` 的子类，在 `set_up()` 里做 telemetry / logging 初始化：

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

**上游变化**：`AdkApp` 在新版本可能位置调整，部署前去上游确认：https://adk.dev/deploy/agent-runtime/

**Docs**：
- Agent Runtime 总览：https://adk.dev/deploy/agent-runtime/
- Standard deploy：https://adk.dev/deploy/agent-runtime/deploy/
- `agents-cli`（CI/CD + IaC）：https://adk.dev/deploy/agent-runtime/agents-cli/
- Test deployed agents：https://adk.dev/deploy/agent-runtime/test/

---

### Cloud Run 部署

`adk deploy cloud_run <path>` 生成 FastAPI 容器，自动打包部署。

**Docs**：https://adk.dev/deploy/cloud-run/

---

### GKE 部署

`adk deploy gke <path>` 生成 K8s manifest。或手动 gcloud + kubectl。

**Docs**：https://adk.dev/deploy/gke/

---

## Authentication

**两级**：
1. **服务层**：ADK 进程调 GCP API 的身份（ADC / Service Account）
2. **Tool 层**：Tool 调外部 API 时的 credential（传给 toolset）

**Import**：
```python
from google.adk.auth import (
    AuthCredential,
    AuthCredentialTypes,
    OAuth2Auth,
    OpenIdConnectWithConfig,
    AuthConfig,
    AuthHandler,
    AuthScheme,
    AuthSchemeType,
    BaseAuthProvider,
)
```

### AuthCredentialTypes 枚举

- `API_KEY`
- `HTTP`（Basic / Bearer）
- `OAUTH2`
- `OIDC`（OpenID Connect）
- `SERVICE_ACCOUNT`

### OAuth2 流程（概要）

1. Tool 声明需要 OAuth credential
2. 首次调用时 runner 发出 "需要授权" event（`EventActions.requested_auth_configs`）
3. UI 引导用户完成 OAuth 跳转
4. 拿到 token 后 runner 重放 tool 调用

详细流程见 https://adk.dev/tools-custom/authentication/。

### 服务层（ADC）

本地开发：`gcloud auth application-default login`
部署到 GCP：使用绑定的 Service Account（Agent Engine / Cloud Run / GKE 都支持）

---

## Telemetry & Observability

**Docs**：https://adk.dev/observability/ · https://adk.dev/observability/logging/

### 自带 OpenTelemetry tracing

ADK 集成 OpenTelemetry，在 `AdkApp` / `AgentEngineApp` 中：

```python
AgentEngineApp(agent=root_agent, enable_telemetry=True)
```

- 导出到 Cloud Trace（部署到 GCP 时自动）
- Spans 覆盖 agent / tool / model 调用

### Logging 约定（项目）

- 标准 `logging` 模块，每个模块 `logger = logging.getLogger(__name__)`
- 生产 JSON formatter（timestamp / level / logger / message / session_id / invocation_id / agent_name）
- `INFO` 记关键业务步骤；`DEBUG` 记详细 state 和 tool args；`WARNING` 重试 / 降级；`ERROR` 异常 / 中断
- 所有 log 带 `session.id` 和 `invocation_id`（用 `extra={}` 或 LoggerAdapter）
- 异常用 `logger.exception()` 保留 stack trace

### 第三方 observability（Integrations）

- AgentOps
- Arize AX
- MLflow
- Phoenix
- W&B Weave

见 https://adk.dev/integrations/（按 Observability 筛选）。

---

## 断点续传 / Resume（再提一次）

**两种粒度：**

| 粒度 | 机制 | 何时选 |
|------|------|-------|
| **Invocation 级** | `ResumabilityConfig`（ADK 原生） | 希望 ADK 自动管理 |
| **步骤级** | 自建 `pipeline_completed_steps` state（见 `agents.md`） | 需要更粗的控制或需要自定义跳过逻辑 |

两者可叠加。

**Docs**：https://adk.dev/runtime/resume/

---

## Event Loop（运行时执行模型）

**Docs**：https://adk.dev/runtime/event-loop/

Runner 是一个异步事件循环：
1. 收到 user message → yield user event
2. 调 agent → agent 产生 LLM event
3. LLM 请 tool call → yield tool call event
4. 执行 tool → yield tool response event
5. 继续到 LLM 产生最终响应 → yield final event
6. Runner 把所有 event 追加到 session.events，持久化 state_delta / artifact_delta

**流式**：`response_modalities=["TEXT"]` 时，LLM partial events 会以流式流出（`event.partial=True`）。

---

## Ambient Agents（后台运行）

**Docs**：https://adk.dev/runtime/ambient-agents/

主动触发的后台 agent 模式（非用户 message 驱动，如定时任务、外部事件触发）。细节见上游。

---

## CLAUDE.md 相关约束

- 部署入口：`app/agent_engine_app.py`，`AgentEngineApp(AdkApp)` 并开 telemetry
- 生产日志：JSON formatter + 标准字段；log 所有 agent / tool / 外部 API 调用
- 所有 log 带 `session.id` + `invocation_id`
- 异常保留 stack trace（`logger.exception()`）
- 关联 ID：支持按单次调用链路追查
- 耗时记录：预期 > 500ms 的操作必须 log duration_ms
- 敏感数据脱敏：API key / token / SA JSON / 完整 PII 绝不入 log

---

## 上游参考

- **Runtime 总览**：https://adk.dev/runtime/
- **Web Interface**：https://adk.dev/runtime/web-interface/
- **Command Line**：https://adk.dev/runtime/command-line/
- **API Server**：https://adk.dev/runtime/api-server/
- **RunConfig**：https://adk.dev/runtime/runconfig/
- **Resume Agents**：https://adk.dev/runtime/resume/
- **Event Loop**：https://adk.dev/runtime/event-loop/
- **Ambient Agents**：https://adk.dev/runtime/ambient-agents/
- **Deploy 总览**：https://adk.dev/deploy/
- **Agent Engine**：https://adk.dev/deploy/agent-runtime/
- **Cloud Run**：https://adk.dev/deploy/cloud-run/
- **GKE**：https://adk.dev/deploy/gke/
- **CLI**：https://adk.dev/api-reference/cli/
- **Authentication**：https://adk.dev/tools-custom/authentication/
- **Observability**：https://adk.dev/observability/
- **Logging**：https://adk.dev/observability/logging/
- **源码**：
  - `src/google/adk/apps/`
  - `src/google/adk/runners.py`
  - `src/google/adk/auth/`
  - `src/google/adk/cli/`
  - `src/google/adk/telemetry/`
