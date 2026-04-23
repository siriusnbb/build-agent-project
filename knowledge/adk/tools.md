# ADK Tools

> FunctionTool / LongRunningFunctionTool / AgentTool / 内置 tools / OpenAPI / MCP / APIHub / Tool 内 auth / Pydantic schema 集成。
>
> **Last synced:** 2026-04-23 · **adk-python commit:** `62d7ee0` · **Upstream:** https://adk.dev/tools-custom/

---

## Tool 类型总览

| 类型 | Import | 源码 | 何时用 |
|------|--------|------|--------|
| `FunctionTool` | `from google.adk.tools import FunctionTool` | `tools/function_tool.py` | 普通 Python 函数暴露给 LLM |
| 函数直接作 tool | `Agent(tools=[my_func])` | — | 传函数时 ADK 自动包 `FunctionTool` |
| `LongRunningFunctionTool` | `from google.adk.tools import LongRunningFunctionTool` | `tools/long_running_tool.py` | LRO、后台任务、需要人类回合 |
| `AgentTool` | `from google.adk.tools import AgentTool` | `tools/agent_tool.py` | 把 sub-agent 当 tool 调用 |
| `BaseTool` | `from google.adk.tools import BaseTool` | `tools/base_tool.py` | 所有 tool 的基类，自定义时继承 |
| `TransferToAgentTool` / `transfer_to_agent` | `from google.adk.tools import ...` | — | 对话转交给另一个 agent |
| `ExampleTool` | `from google.adk.tools import ExampleTool` | `tools/example_tool.py` | Few-shot 示例注入 |

**Docs 总览**：https://adk.dev/tools-custom/function-tools/

---

## FunctionTool / 函数作 tool

**用途**：把普通 Python 函数暴露给 LLM。ADK 从函数签名 + docstring 自动生成 tool schema。

### 最简用法（直接传函数）

```python
from google.adk.tools import ToolContext  # = Context，LLM 注入

def get_weather(city: str, tool_context: ToolContext) -> dict:
    """Return the weather of a city.

    Args:
        city: City name, e.g. "Tokyo".

    Returns:
        Dict with status and data.
    """
    # 读 state
    prev = tool_context.state.get("last_query")
    # 写 state
    tool_context.state["last_query"] = city
    return {"status": "success", "temperature": 22}

Agent(tools=[get_weather])   # ADK 自动包成 FunctionTool
```

### 规则（必记）

1. **`tool_context: ToolContext` 必须是最后一个参数**，ADK 自动注入，不在 schema 中暴露给 LLM
2. **docstring 即为 LLM 看到的 tool description**，要准确描述功能和参数
3. 参数要有类型标注（Pydantic 风格），ADK 据此生成 JSON schema
4. 返回值通常是 `dict`，LLM 看到完整内容（避免塞大对象）
5. 不要 return 二进制 / 大 dict；binary 走 `save_artifact`，见 `sessions-artifacts-memory.md`

### 异步 tool

ADK 支持 `async def`：

```python
async def fetch_user(user_id: str, tool_context: ToolContext) -> dict:
    """..."""
    result = await async_api(user_id)
    return result
```

**CLAUDE.md 约束**：异步必须用 `asyncio`，禁止 `threading`。

---

## LongRunningFunctionTool

**用途**：
- Long Running Operation（LRO）：外部 API 返回 operation name，需要轮询
- Human-in-the-loop：要人工确认后再继续
- 后台任务：不阻塞主 agent 响应

**Import**：`from google.adk.tools import LongRunningFunctionTool`

**典型形态**（概念，具体 API 见上游）：

```python
from google.adk.tools import LongRunningFunctionTool, ToolContext

async def kick_off_job(task_id: str, tool_context: ToolContext):
    """Start a long-running job and yield progress."""
    # 启动后台任务
    yield {"status": "started", "task_id": task_id}
    # 期间 agent 可以响应用户，轮询完成后 runner 会把最终结果发回
    # ...

tool = LongRunningFunctionTool(func=kick_off_job)
```

完整用法参见上游：https://adk.dev/tools-custom/function-tools/ 的 "Long-Running" 段落。

**Docs**：https://adk.dev/tools-custom/function-tools/

---

## AgentTool（sub-agent 作为 tool）

**用途**：把 sub-agent 包装成 tool，父 agent 像调用 tool 一样调用子 agent。

**Import**：`from google.adk.tools import AgentTool`

```python
from google.adk.tools import AgentTool

helper = Agent(name="helper", ...)

parent = Agent(
    name="parent",
    tools=[
        my_func_tool,
        AgentTool(helper),   # 包装
    ],
)
```

**语义**：
- 父 agent 调用 AgentTool → 子 agent 以新 invocation 执行 → 返回结果给父 agent → 父 agent 继续自己的回合
- 与 `sub_agents=[...]` + transfer 的区别见 `agents.md` 的 "Multi-Agent 系统" 章节

**ADK 单父约束**：一个 agent 实例只能有一个父，同一个 helper 要给两个父用必须创建两份实例。参见 `agents.md`。

**Docs**：仅源码，无独立文档页 — 在 Multi-Agent 文档中提及。源码：`src/google/adk/tools/agent_tool.py`

---

## TransferToAgentTool / transfer_to_agent

**用途**：LLM 显式把对话转交给另一个 agent（handoff 模式，不是返回值式）。

**Import**：
- `from google.adk.tools import transfer_to_agent`（函数形式）
- `from google.adk.tools import TransferToAgentTool`（类形式）

**默认行为**：当 agent 有 `sub_agents` 时，`transfer_to_agent` 自动可用，LLM 可以调：
```
transfer_to_agent(agent_name="billing_agent")
```

**与 AgentTool 的区别**：
| AgentTool | TransferToAgentTool |
|-----------|---------------------|
| 父得到返回值后继续 | 对话完全交给新 agent |
| 工具化 sub-ability | 专家分流 / handoff |

**EventActions 关联**：transfer 会发出 `EventActions.transfer_to_agent`，见 `callbacks-and-events.md`。

---

## 内置 Tools

所有都 `from google.adk.tools import ...`：

| 名字 | 用途 | 备注 |
|------|------|------|
| `google_search` | Google 搜索（Gemini 2.0+ 原生 grounding） | Gemini 专属 |
| `enterprise_web_search` | 企业版 Web 搜索 | Gemini 专属 |
| `VertexAiSearchTool` | Vertex AI Search（私有数据索引） | 类形式 |
| `url_context` | 把 URL 内容注入到 prompt | Gemini 2.0+ |
| `load_memory` | 从 Memory service 检索 | 需配套 memory service |
| `preload_memory` | 预热 memory（入口 callback 用） | 同上 |
| `load_artifacts` | 加载历史 artifact | 需配套 artifact service |
| `get_user_choice` | 让用户从多个选项中选 | Human-in-the-loop |
| `exit_loop` | 退出当前 LoopAgent | 见 `agents.md` |
| `transfer_to_agent` | 转交对话 | 见上文 |
| `google_maps_grounding` | Google Maps 地理信息 grounding | Gemini 2.0+ |
| `DiscoveryEngineSearchTool` / `SearchResultMode` | Discovery Engine 搜索（含项目已用） | 详见上游 |
| `ExampleTool` | Few-shot 示例 | — |

**Docs 总览**：https://adk.dev/tools-custom/ 各子页面

---

## OpenAPIToolset（REST API 自动 tool 化）

**用途**：从 OpenAPI 3.0 规范自动生成 tool。给定 OpenAPI spec，每个 operation 变成一个 tool。

**Import**：`from google.adk.tools import OpenAPIToolset`（实际类名以当前源码为准）

**典型用法**（形式，具体 API 见上游）：

```python
toolset = OpenAPIToolset(
    spec_path_or_url="https://api.example.com/openapi.json",
    auth_credential=AuthCredential(...),  # 可选，见 tool auth 章节
)
Agent(tools=toolset.get_tools())
```

**Docs**：https://adk.dev/tools-custom/openapi-tools/

---

## MCPToolset（Model Context Protocol）

**用途**：接入符合 MCP 规范的外部工具服务器（如 Claude Desktop MCP 生态）。ADK 可以作 MCP client 消费工具，也可作 server 暴露工具。

**Import**：`from google.adk.tools import MCPToolset`（别名 `McpToolset`）

**Docs**：https://adk.dev/tools-custom/mcp-tools/

**MCP 规范**：https://modelcontextprotocol.io/specification/

---

## APIHubToolset（Apigee API Hub）

**用途**：接入 Google Cloud Apigee API Hub 管理的 API。

**Import**：`from google.adk.tools import APIHubToolset`

**Docs**：仅在 tools-custom 文档中简述，细节见源码 `src/google/adk/tools/apihub_tool.py`

---

## Tool 内 Auth（AuthCredential）

**用途**：tool 里调用第三方 API 时，用统一的 credential 对象管理 API key / OAuth / OIDC / Service Account。

**支持类型**（`AuthCredentialTypes` 枚举）：
- API_KEY
- HTTP（Basic / Bearer）
- OAUTH2
- OIDC（OpenID Connect）
- SERVICE_ACCOUNT

**Import**：
```python
from google.adk.auth import AuthCredential, AuthCredentialTypes
from google.adk.auth import OAuth2Auth, OpenIdConnectWithConfig, AuthConfig
```

**典型用法**（形式）：

```python
credential = AuthCredential(
    auth_type=AuthCredentialTypes.OAUTH2,
    oauth2=OAuth2Auth(
        client_id="...",
        client_secret="...",
        authorization_url="...",
        token_url="...",
        scopes=["scope1", "scope2"],
    ),
)
```

**与 tool 的集成**：大多数 toolset（OpenAPIToolset, APIHubToolset, MCPToolset）构造时接受 `auth_credential` 或 `auth_config` 参数。完整流程（含 user 交互式授权回合）参见 `runtime-and-deploy.md` 的 Auth 章节。

**Docs**：https://adk.dev/tools-custom/authentication/

---

## Action Confirmations（Tool 确认）

**用途**：在敏感 tool 执行前要求用户显式确认。

**Docs**：https://adk.dev/tools-custom/confirmation/

---

## Tool 里写 / 读 Session State

```python
def my_tool(key: str, tool_context: ToolContext) -> dict:
    # 读
    value = tool_context.state.get(key)
    # 写（带前缀决定作用域，见 guide.md）
    tool_context.state["key_name"] = "..."
    tool_context.state["user:preference"] = "..."      # 跨 session
    tool_context.state["app:global_flag"] = True       # 跨 user
    tool_context.state["temp:scratch"] = {...}         # 不持久化
    return {"status": "success"}
```

State 前缀规则与 StateKeys 集中式常量模式见 `context-and-state.md`。

---

## Tool 里保存 Artifact

Binary 数据（文件内容、JSON dump）走 Artifact，不塞 state：

```python
from google.genai import types

async def save_report(data: dict, tool_context: ToolContext) -> dict:
    payload = json.dumps(data, ensure_ascii=False).encode("utf-8")
    part = types.Part.from_bytes(data=payload, mime_type="application/json")
    version = await tool_context.save_artifact("report.json", part)
    return {"status": "success", "artifact_version": version}
```

Artifact 的存储后端（InMemory / File / GCS）由 runner 装配，tool 里不关心。详见 `sessions-artifacts-memory.md`。

**CLAUDE.md 规则**：文件 binary 必须走 save_artifact，不塞 session state。

---

## 退出 LoopAgent（tool_context.actions）

```python
def exit_loop(tool_context: ToolContext) -> dict:
    """Exit the surrounding LoopAgent."""
    tool_context.actions.escalate = True
    return {}
```

`tool_context.actions` 还支持：
- `skip_summarization = True` — 本次工具结果不进入 LLM summarization
- 其它 action 见 `callbacks-and-events.md` 的 EventActions

内置 `exit_loop` tool 与此等价，推荐直接用：
```python
from google.adk.tools import exit_loop
Agent(tools=[exit_loop])
```

---

## Pydantic Schema 集成

**用途一：约束 Tool 参数**

Tool 参数用 Pydantic 模型时 ADK 自动生成对应 JSON schema，LLM 输出被约束到合法结构：

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

def format_report(report: dict, tool_context: ToolContext) -> dict:
    validated = ReportSchema.model_validate(report)
    tool_context.state["report"] = validated.model_dump()
    return {"status": "success"}
```

**用途二：约束 LLM 输出（`output_schema`）**

Agent 构造时传 `output_schema=PydanticModel`，LLM 必须产出合法 JSON：

```python
Agent(
    name="structurer",
    output_schema=ReportSchema,
    output_key="structured_report",
    instruction="...",
)
```

这种模式下 agent 不能同时有 tools（二选一）。

---

## CLAUDE.md 相关约束

- **日志**：tool 调用前 log `INFO "tool {name} called"`（带入参摘要）；调用后 log 完成或失败 + 耗时（见 CLAUDE.md "Tool 调用必须 log"）
- **循环处理多项目**：try/except 包循环体，失败继续，返回 `{total, success, failed, failed_items}` 汇总
- **清理失败**：`logger.exception()` 保留 stack trace，不要 `str(e)`
- **不要为测试改 tool 签名**：测试用 `unittest.mock.patch` mock 外部依赖，不要给 tool 加 `sleep_fn` / `get_document` 注入参数
- **敏感数据脱敏**：LLM prompt 和响应仅 DEBUG 级别 log 全文；INFO 级别只 log 长度和 tool_calls 名称

---

## 上游参考

- **Tools 总览**：https://adk.dev/tools-custom/
- **Function Tools**：https://adk.dev/tools-custom/function-tools/
- **OpenAPI Tools**：https://adk.dev/tools-custom/openapi-tools/
- **MCP Tools**：https://adk.dev/tools-custom/mcp-tools/
- **Authentication**：https://adk.dev/tools-custom/authentication/
- **Action Confirmations**：https://adk.dev/tools-custom/confirmation/
- **Performance**：https://adk.dev/tools-custom/performance/
- **MCP 规范**：https://modelcontextprotocol.io/specification/
- **源码根目录**：https://github.com/google/adk-python/tree/main/src/google/adk/tools
