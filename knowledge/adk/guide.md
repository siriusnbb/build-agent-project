# ADK 知识库导航

> Google Agent Development Kit (ADK) 本地速查 + 决策表 + 子文件索引。
>
> **Last synced:** 2026-04-23 · **adk-python commit:** `62d7ee0` · **Upstream:** https://adk.dev/ · https://github.com/google/adk-python

---

## 这个知识库怎么用

**定位：导航地图 + cheat sheet。** 每个 ADK 概念都在本库有条目：名字、一句话用途、典型场景、import 路径、上游链接。**不复刻大段示例代码，但不漏任何概念。**

**使用流程（面向未来写代码的 agent）：**

1. 先看本 `guide.md` 的"决策表"和"主题到文件反向索引"定位去哪个详情文件
2. 去对应详情文件找条目，拿到 import、签名、最小骨架
3. 只在以下情况 WebFetch 上游：
   - 条目标注"仅源码，无文档页"→ 必须去 GitHub
   - 条目标注"见上游最新"→ 官方语义可能变化
   - 要找具体的参数默认值、枚举值清单
   - 要看完整示例（本库只放骨架）

**何时 WebFetch：** 见最后一节"联网补查清单"。

---

## 文件分工速查

| 文件 | 主题 |
|------|------|
| **guide.md**（本文件） | 入口索引、决策表、import 速查、State 前缀、反向主题索引 |
| **agents.md** | Agent 类型（LlmAgent / BaseAgent / Sequential / Loop / Parallel / RemoteA2a）、多 agent transfer、单父约束、__init__.py 导出惯例 |
| **tools.md** | FunctionTool、LongRunningFunctionTool、AgentTool、内置 tools、OpenAPI / MCP / APIHub Toolset、tool 内 auth、Pydantic schema 集成 |
| **context-and-state.md** | 4 种 Context（Readonly / Callback / Tool / Invocation）、Context(ctx) 包装、State 前缀、StateKeys 模式、session.events、动态 instruction |
| **sessions-artifacts-memory.md** | BaseSessionService / InMemory / Vertex / Database / Sqlite、Artifact 三个实现、Memory 三个实现、Artifact vs State 边界 |
| **callbacks-and-events.md** | 6 类 callback、Event、EventActions（escalate / transfer_to_agent / state_delta / artifact_delta）、Plugins |
| **runtime-and-deploy.md** | App、Runner、RunConfig、ResumabilityConfig、CLI（adk web/run/api_server/deploy/eval/create/optimize）、Agent Engine / Cloud Run / GKE、Auth、Telemetry |
| **models-and-streaming.md** | Gemini / Claude / LiteLlm / Gemma / Ollama / vLLM / Apigee、Live API、LiveRequest(Queue)、流式 tool |
| **advanced.md** | A2A、Planners、Code Executors、Evaluation、Grounding、Safety、Skills、Optimization、Integrations 目录 |

---

## 主题 → 文件反向索引

用任务描述反查去哪个文件：

| 我想做的事 | 去哪个文件 |
|-----------|-----------|
| 定义一个新 LlmAgent | agents.md |
| 自定义确定性 pipeline | agents.md（BaseAgent 模式） |
| LoopAgent 迭代 + exit_loop | agents.md + tools.md |
| 把 sub-agent 包成 tool | agents.md（AgentTool） + tools.md |
| 让 agent 动态路由到另一个 agent | agents.md（transfer）+ tools.md（TransferToAgentTool） |
| 写 FunctionTool | tools.md |
| 长运行 / 人机回合 tool | tools.md（LongRunningFunctionTool） |
| 接 OpenAPI 或 MCP 服务 | tools.md |
| Tool 里做 OAuth | tools.md（Tool auth） + runtime-and-deploy.md（AuthCredential） |
| 读/写 session state | context-and-state.md |
| 状态前缀 app:/user:/temp: | context-and-state.md |
| BaseAgent 里 save_artifact | context-and-state.md（Context 包装）+ sessions-artifacts-memory.md |
| 把 session 存到数据库 | sessions-artifacts-memory.md（DatabaseSessionService） |
| Artifact 存 GCS | sessions-artifacts-memory.md（GcsArtifactService） |
| 跨 session 检索记忆 | sessions-artifacts-memory.md（Memory services） |
| 初始化 state | callbacks-and-events.md（before_agent_callback） |
| 改 LLM prompt 或响应 | callbacks-and-events.md（before/after_model_callback） |
| 防御性检查 tool 参数 | callbacks-and-events.md（before_tool_callback） |
| 跨切面 logging | callbacks-and-events.md（Plugins） |
| 跑 agent（本地 dev） | runtime-and-deploy.md（adk web / Runner） |
| 部署到 Agent Engine | runtime-and-deploy.md + gcp-deployment-guide.md |
| 部署到 Cloud Run / GKE | runtime-and-deploy.md + gcp-deployment-guide.md |
| 断点续传（resumability） | runtime-and-deploy.md（RunConfig / ResumabilityConfig） |
| 换 Gemini 为 Claude | models-and-streaming.md |
| 本地 Ollama / Gemma | models-and-streaming.md |
| 流式吐字 / Live API | models-and-streaming.md |
| 远程调用另一个 agent（跨进程） | advanced.md（A2A） |
| 用 sandbox 执行 LLM 生成的代码 | advanced.md（Code Executors） |
| Agent 端到端 eval | advanced.md（Evaluation） |
| google_search grounding | advanced.md（Grounding） 或 tools.md（内置 tool） |
| 安全 / 内容过滤 | advanced.md（Safety） |
| adk optimize 自动优化 | advanced.md + runtime-and-deploy.md |

---

## 决策表 1：Agent 类型选择

| 需求 | 选择 | 在哪定义 |
|------|------|---------|
| 单一 LLM 任务、tool ≤ 5 | `LlmAgent`（也叫 `Agent`） | agents.md |
| 固定顺序多步、无条件分支 | `SequentialAgent` | agents.md |
| 需要条件分支、resume-from-failure | 继承 `BaseAgent` | agents.md |
| 子任务独立、可并行 | `ParallelAgent` | agents.md |
| 需要质量迭代直到达标 | `LoopAgent` + `exit_loop` tool | agents.md + tools.md |
| LLM 动态决定调用哪个 sub-agent | `LlmAgent` + `AgentTool(sub)` | agents.md |
| 明确 handoff 给另一个 agent | `transfer_to_agent` / `TransferToAgentTool` | tools.md |
| 跨进程调用远程 agent | `RemoteA2aAgent` | advanced.md |
| 混合需求 | 上述组合（Sequential 套 Loop、Parallel 套 Sequential 等） | — |

**CLAUDE.md 提醒**：判断逻辑可确定（if/else）时首选 `BaseAgent`，零 LLM 成本零延迟。

---

## 决策表 2：Tool 类型选择

| 需求 | 选择 |
|------|------|
| 普通 Python 函数暴露给 LLM | `FunctionTool`（或直接把函数传入 `tools=[...]`） |
| 长运行 / 需要人类确认再继续 | `LongRunningFunctionTool` |
| Sub-agent 作为可调用能力 | `AgentTool(sub_agent)` |
| 动态转交对话给另一个 agent | `transfer_to_agent` / `TransferToAgentTool` |
| 从 OpenAPI 规范自动生成 tool | `OpenAPIToolset` |
| 接现有 MCP 服务 | `MCPToolset` |
| 接 Apigee API Hub 管理的 API | `APIHubToolset` |
| Google 搜索（Gemini 原生） | `google_search`（Gemini 2.0+） |
| Vertex AI Search（企业数据） | `enterprise_web_search` / `VertexAiSearchTool` |
| 注入 URL 内容到 prompt | `url_context` |
| 检索 Memory 中的历史 | `load_memory` / `preload_memory` |
| 加载历史 Artifact | `load_artifacts` |
| 让用户多选 | `get_user_choice` |
| 退出 LoopAgent | `exit_loop` |

---

## 决策表 3：Service 选型

三大 service 家族都有"内存 / 本地 / 云端"三档：

| 场景 | Session | Artifact | Memory |
|------|---------|----------|--------|
| 开发 / pytest | `InMemorySessionService` | `InMemoryArtifactService` | `InMemoryMemoryService` |
| 本地持久化 | `DatabaseSessionService`（SQLAlchemy）或 `SqliteSessionService` | `FileArtifactService` | （无本地版） |
| 生产 on GCP | `VertexAiSessionService`（Agent Engine 原生） | `GcsArtifactService` | `VertexAiMemoryBankService` 或 `VertexAiRagMemoryService` |

详见 `sessions-artifacts-memory.md`。

---

## 决策表 4：部署目标

| 目标 | 何时选 | 细节文件 |
|------|--------|---------|
| **Vertex AI Agent Engine** | 全托管、原生支持 session/memory、最少运维 | runtime-and-deploy.md + gcp-deployment-guide.md |
| **Cloud Run** | 需要 HTTP 接口、按需伸缩、已有 Cloud Run 栈 | runtime-and-deploy.md + gcp-deployment-guide.md |
| **GKE** | 已有 K8s 栈、需要更细粒度控制 | runtime-and-deploy.md + gcp-deployment-guide.md |

---

## 决策表 5：确定性 vs LLM 判断（项目规范）

参见 CLAUDE.md 的"Agent 类型选择"和"确定性判断 vs LLM 判断"段落。核心规则：

| 判断来源 | 用什么 |
|---------|-------|
| 条件可以从代码获取（state key 值、文件存在性、队列长度） | 代码 if/else，在 `BaseAgent._run_async_impl` 或 callback 内 |
| 需要自然语言理解 / 动态查询构建 / 内容生成 / 用户对话 | `LlmAgent` |
| Root callback 里可以确定的 state | `before_agent_callback`（在 LLM 执行前就定下） |

---

## Import 速查（~40 条常用 import）

按模块分组，全部基于 `google.adk.*`：

```python
# —— agents ——
from google.adk.agents import Agent, LlmAgent, BaseAgent
from google.adk.agents import SequentialAgent, LoopAgent, ParallelAgent
from google.adk.agents import Context, InvocationContext
from google.adk.agents import LiveRequest, LiveRequestQueue, RunConfig
from google.adk.agents.readonly_context import ReadonlyContext
from google.adk.agents.callback_context import CallbackContext

# —— tools ——
from google.adk.tools import ToolContext  # alias of Context
from google.adk.tools import FunctionTool, LongRunningFunctionTool, AgentTool
from google.adk.tools import BaseTool, TransferToAgentTool
from google.adk.tools import google_search, enterprise_web_search, url_context
from google.adk.tools import load_memory, preload_memory, load_artifacts
from google.adk.tools import get_user_choice, exit_loop, transfer_to_agent
from google.adk.tools import VertexAiSearchTool, google_maps_grounding
from google.adk.tools import APIHubToolset, MCPToolset  # MCPToolset == McpToolset

# —— models ——
from google.adk.models import BaseLlm, Gemini, Gemma, LLMRegistry
# 以下需要 google-adk[extensions]：
# from google.adk.models import Claude, LiteLlm, Gemma3Ollama

# —— sessions / artifacts / memory ——
from google.adk.sessions import BaseSessionService, Session, State
from google.adk.sessions import InMemorySessionService, DatabaseSessionService, VertexAiSessionService
from google.adk.artifacts import BaseArtifactService
from google.adk.artifacts import InMemoryArtifactService, FileArtifactService, GcsArtifactService
from google.adk.memory import BaseMemoryService, InMemoryMemoryService
from google.adk.memory import VertexAiMemoryBankService, VertexAiRagMemoryService

# —— runtime ——
from google.adk.apps import App, ResumabilityConfig
from google.adk.runners import Runner, InMemoryRunner

# —— events ——
from google.adk.events import Event, EventActions

# —— auth ——
from google.adk.auth import AuthCredential, AuthCredentialTypes
from google.adk.auth import OAuth2Auth, OpenIdConnectWithConfig, AuthConfig

# —— code executors / planners / evaluation ——
from google.adk.code_executors import BuiltInCodeExecutor, UnsafeLocalCodeExecutor
# Extensions: VertexAiCodeExecutor, ContainerCodeExecutor, GkeCodeExecutor, AgentEngineSandboxCodeExecutor
from google.adk.planners import BasePlanner, BuiltInPlanner, PlanReActPlanner
from google.adk.evaluation import AgentEvaluator  # 需要 google-cloud-aiplatform[evaluation]

# —— plugins ——
from google.adk.plugins import BasePlugin, LoggingPlugin, DebugLoggingPlugin
from google.adk.plugins import PluginManager, ReflectAndRetryToolPlugin
```

**注意：**
- `Agent` 是 `LlmAgent` 的别名。
- `ToolContext` 是 `Context` 的别名（Context 定义在 `agents/context.py`）。
- `Claude` / `LiteLlm` / `Gemma3Ollama` 需要安装 `google-adk[extensions]` 附加包。
- `ApigeeLlm` 在源码中注册但不在 `__all__` 里，只能从 `google.adk.models.apigee_llm` 直接引入（见 models-and-streaming.md）。
- MCP 相关：`MCPToolset` 与 `McpToolset` 是同一类的两种拼写。
- `Runner` / `InMemoryRunner` 位于顶层模块 `google.adk.runners`，不是子目录。

---

## State key 命名与前缀速查

ADK `Session.state` 支持 4 种作用域，以 key 前缀区分：

| 前缀 | 作用域 | 持久化 | 典型用法 |
|------|-------|--------|---------|
| （无前缀） | 当前 session | 随 session 持久化 | 单次对话中的业务状态 |
| `user:` | 当前 user 的所有 session | 跨 session | 用户偏好、累积历史 |
| `app:` | 整个 app 所有用户 | 跨 user | 全局配置、共享白名单 |
| `temp:` | 当前 invocation | 不持久化 | 本轮工具调用间临时数据 |

**CLAUDE.md 规则：**
- 每个 state key 职责唯一，不用同一个 key 编码多种状态
- 不用 value 前缀（如 `"gcs:xxx"`）表达状态——用独立的 boolean flag
- 环境变量类值直接从 config 读，不写入 state
- ADK session 自带 `session.id`，一般不需要自建 UUID
- State 只存轻量数据；二进制走 Artifact

**StateKeys 集中式模式**（项目约定）：在 `app/config.py` 下定义 `class StateKeys` 收集所有 key 常量，详见 `context-and-state.md`。

---

## 项目约定回顾

已在 CLAUDE.md 固化的开发规范中直接相关的部分：

- **编码风格**：Google 风格 docstring；async 用 asyncio，禁用 threading；追求简洁单一的流程
- **Agent 类型选择**：能代码判断的用 `BaseAgent`
- **State 设计**：key 唯一职责；不用 value 编码状态；不造 UUID；只存轻量数据
- **Artifact**：binary 走 save/load_artifact，不塞 state
- **Prompt 管理**：基础 prompt + 条件追加；条件选择在 prompt 文件里，不在 agent 文件里
- **错误处理**：循环内 try/except 包单项；返回含总数/成功/失败数；异常必须带 stack trace
- **日志**：统一用标准 logging；生产 JSON；每个 BaseAgent 入/出 log；tool 前后 log；外部 API 必 log 耗时
- **脱敏**：API key、token、SA JSON、用户完整 PII 绝不出现在 log

**相关文件：**
- `../CLAUDE.md`（项目总规范）
- `../project-patterns.md`（项目架构惯例）
- `../coding-standards.md`（编码规范）
- `../gcp-deployment-guide.md`（GCP 部署）

---

## 联网补查清单：何时该 WebFetch 上游

**本库覆盖 80% 日常场景。以下情况必须去官方：**

1. **参数默认值、完整枚举值**：本库只标类型和名字，具体值去上游
2. **最新变更**：条目注明 `Last synced` 之后的版本变化，特别是：
   - `Gemini` 模型名列表（每月新增）
   - `adk deploy` 新目标或 flag
   - 新增的内置 tool
3. **完整示例**：本库给骨架，上游有端到端 demo
4. **注明"仅源码"的条目**：
   - `ApigeeLlm`
   - `AgentTool` 详细文档
   - State 前缀完整规则（目前分散在各页面）
   - 4 种 Context 的详细 API（仅在 api-reference 中）
   - Planners 详细文档
   - Code Executors 所有变种
5. **Agent Samples 查示例**：https://github.com/google/adk-samples

**推荐 WebFetch 入口：**

| 目标 | URL |
|------|-----|
| 官方总入口 | https://adk.dev/ |
| Python API 参考 | https://adk.dev/api-reference/python/ |
| 源码浏览 | https://github.com/google/adk-python/tree/main/src/google/adk |
| 官方示例仓库 | https://github.com/google/adk-samples |
| 集成目录 | https://adk.dev/integrations/ |
| CLI 命令全集 | https://adk.dev/api-reference/cli/ |
| MCP 规范 | https://modelcontextprotocol.io/specification/ |

---

## 版本与同步信息

- **本库 Last synced**：2026-04-23
- **adk-python snapshot**：commit `62d7ee0`
- **包名**：`google-adk`
- **Python 要求**：>= 3.10
- **主要上游**：
  - 文档：https://adk.dev/
  - 镜像：google.github.io/adk-docs → 301 重定向到 adk.dev
  - 源码：https://github.com/google/adk-python
  - 示例：https://github.com/google/adk-samples
- **版本策略**：按季度同步一次。每次同步更新各文件顶部的 `Last synced` 和 commit hash。
