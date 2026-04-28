# ADK Advanced

> A2A / Planners / Code Executors / Evaluation / Grounding / Safety / Skills / Optimization / Integrations。
>
> **Last synced:** 2026-04-23 · **adk-python commit:** `62d7ee0` · **Upstream:** https://adk.dev/

---

## A2A Protocol（Agent-to-Agent）

**用途**：agent 跨进程互相调用。一个 agent 可以"暴露"为 A2A 服务被他人调用，也可以"消费"远程 A2A agent 当成本地 sub-agent 用。

### 消费远程 agent → `RemoteA2aAgent`

**Import**：`from google.adk.agents import RemoteA2aAgent`

**典型形式**（以上游为准）：
```python
from google.adk.agents import RemoteA2aAgent

remote = RemoteA2aAgent(
    name="billing_service",
    endpoint="https://billing-agent.example.com/a2a",
    # auth_credential=...
)

parent = Agent(
    name="parent",
    sub_agents=[remote],
    ...
)
```

### 暴露为 A2A 服务

通过 `adk api_server` 启动的 REST 服务本身就是 A2A 端点。其他 agent 可以用 `RemoteA2aAgent` 指向这个地址消费。

### A2A vs AgentTool 选型

| A2A（RemoteA2aAgent） | AgentTool |
|----------------------|-----------|
| 跨进程 / 跨机器 | 同进程 |
| 需网络、有认证 | 直接函数调用 |
| 独立部署、独立扩容 | 共享资源 |
| 不同语言团队协作 | Python 内部组合 |

**源码**：
- `src/google/adk/a2a/`（客户端 + server converter + executor）
- `src/google/adk/agents/remote_a2a_agent.py`

**Docs**：无独立文档页，细节见源码和 adk-samples。

---

## Planners

**用途**：在 LLM 实际调 tool 前，先让模型"规划"要做什么。

**Import**：
```python
from google.adk.planners import BasePlanner, BuiltInPlanner, PlanReActPlanner
```

| Planner | 用途 |
|---------|------|
| `BasePlanner` | 抽象基类，自建规划器时继承 |
| `BuiltInPlanner` | Gemini 2.0+ 原生思考模式（thinking） |
| `PlanReActPlanner` | ReAct 风格：规划 → 行动 → 观察 → 再规划 |

**挂载**：`Agent(planner=PlanReActPlanner())`

**Docs**：仅在 LLM Agents 文档中提及，细节见源码 `src/google/adk/planners/`。

---

## Code Executors

**用途**：让 LLM 生成的代码在隔离环境中执行。

**Import**：
```python
from google.adk.code_executors import (
    BaseCodeExecutor,
    BuiltInCodeExecutor,
    CodeExecutorContext,
    UnsafeLocalCodeExecutor,
)
# 以下需要 google-adk[extensions]：
# from google.adk.code_executors import VertexAiCodeExecutor
# from google.adk.code_executors import ContainerCodeExecutor
# from google.adk.code_executors import GkeCodeExecutor
# from google.adk.code_executors import AgentEngineSandboxCodeExecutor
```

| Executor | 隔离级别 | 何时用 |
|----------|---------|-------|
| `BuiltInCodeExecutor` | Gemini 内置 | 默认，最简单，Gemini 原生支持 |
| `UnsafeLocalCodeExecutor` | ❌ 无（执行在本进程） | **仅开发 / 实验**，生产禁用 |
| `VertexAiCodeExecutor` | Vertex 沙箱 | 生产，中度隔离 |
| `ContainerCodeExecutor` | 容器（Docker） | 需本地 Docker |
| `GkeCodeExecutor` | GKE Pod | K8s 环境 |
| `AgentEngineSandboxCodeExecutor` | Agent Engine 托管 | Agent Engine 部署 |

**挂载**：`Agent(code_executor=BuiltInCodeExecutor())`

**Docs**：仅源码，无独立文档页。源码：`src/google/adk/code_executors/`

---

## Evaluation（Agent 评估）

**用途**：对 agent 的响应质量、tool 调用轨迹、端到端行为做量化评估。

**Import**：
```python
from google.adk.evaluation import AgentEvaluator   # 需要 google-cloud-aiplatform[evaluation]
```

### 核心概念

| 概念 | 说明 |
|------|------|
| Eval Set | 测试用例集合（JSON / YAML），每条含用户输入 + 预期行为 |
| Criteria | 评估标准（tool_trajectory、response_match、custom） |
| User Simulation | 动态生成用户提示（模拟多轮对话） |
| Environment Simulation | 模拟外部环境（API 返回、文件系统） |
| Custom Metrics | 项目自定义指标（regex 匹配、LLM-judge 等） |

### CLI

```bash
adk eval <path>                               # 跑评估
adk eval_set create                           # 建评估集
adk eval_set add_eval_case                    # 加 case
adk eval_set generate_eval_cases              # LLM 自动生成 case
adk conformance record / test                 # 一致性回归（录制 → 重放）
```

### 与 pytest / Make 集成

项目约定：`make eval` 执行 ADK 评估。通常在 CI 中：
```bash
pytest tests/integration/ && make eval
```

### Docs

- **Evaluation 总览**：https://adk.dev/evaluate/
- **Criteria**：https://adk.dev/evaluate/criteria/
- **User Simulation**：https://adk.dev/evaluate/user-sim/
- **Environment Simulation**：https://adk.dev/evaluate/environment_simulation/
- **Custom Metrics**：https://adk.dev/evaluate/custom_metrics/

**源码**：`src/google/adk/evaluation/`

---

## Grounding

**用途**：把 LLM 响应"锚定"到外部事实源（搜索结果、私有数据、地图等），减少幻觉。

### 三家内置 grounding

| Grounding | Import | 何时用 |
|-----------|--------|-------|
| Google Search | `from google.adk.tools import google_search` | 公开网络实时信息（Gemini 2.0+） |
| Vertex AI Search | `from google.adk.tools import VertexAiSearchTool` | 私有数据 / 企业文档索引 |
| Google Maps | `from google.adk.tools import google_maps_grounding` | 地理位置、地图信息 |

**使用方式**：作为 tool 放进 `Agent(tools=[...])`，见 `tools.md`。

### Docs

- **Google Search Grounding**：https://adk.dev/grounding/google_search_grounding/
- **Grounding with Search (Enterprise)**：https://adk.dev/grounding/grounding_with_search/
- **Agentic RAG 示例**：
  - Deep Search Agent：https://github.com/google/adk-samples/tree/main/python/agents/deep-search
  - RAG Agent：https://github.com/google/adk-samples/tree/main/python/agents/RAG
  - Vector Search 2.0 Travel Agent：https://github.com/google/adk-samples/blob/main/python/notebooks/grounding/vectorsearch2_travel_agent.ipynb

---

## Safety & Security

**用途**：防止恶意输入、有害内容、tool 误用。

### 三层风险（按上游分类）

1. **歧义指令**：用户提示语义不清可能引导不当行为
2. **Prompt 注入**：恶意输入嵌入指令劫持 agent
3. **Tool 链接注入**：通过 tool 返回值注入后续指令

### 防御机制

| 机制 | 何时用 |
|------|-------|
| **Agent-Auth vs User-Auth 分离** | tool 调用时区分 agent 权限和用户权限 |
| **Gemini 安全设置** | `safety_settings` 控制 harm 类别阈值 |
| **Callback / Plugin 防护** | `before_model_callback` / `before_tool_callback` 做规则过滤 |
| **代码沙箱** | 见 Code Executors |
| **VPC-SC** | GCP VPC Service Controls 网络隔离 |
| **HTML 转义** | LLM 生成 HTML 时自动转义 |
| **Content filters** | 配置 harm 类别阈值 |
| **Action Confirmations** | 敏感 tool 执行前用户确认，见 `tools.md` |

**Docs**：https://adk.dev/safety/

---

## Skills（实验性）

> ⚠️ ADK 1.14+ 实验性，API 还在变。处于 active development。

**用途**：模块化的功能单元，遵循 Agent Skills 规范。可动态加载到 agent，复用 prompt + 资源 + 脚本。

**Import**：
```python
from google.adk.skills import (
    Skill,
    Frontmatter,
    Resources,
    Script,
    load_skill_from_dir,
    list_skills_in_dir,
    load_skill_from_gcs_dir,
    list_skills_in_gcs_dir,
    DEFAULT_SKILL_SYSTEM_INSTRUCTION,
)
```

### Skill 目录结构

一个 skill 是磁盘上的一个目录，约定结构：

```
my_skill/
├── SKILL.md              # 必有：Frontmatter（YAML） + Instructions（Markdown 正文）
├── references/           # 可选：长文/规范/参考资料（被 Resources 引用）
├── assets/               # 可选：图片、模板等静态资产
└── scripts/              # 可选：辅助脚本（被 Script 引用）
```

**Frontmatter**（位于 `SKILL.md` 顶部 YAML）：装载 skill 的元数据 —— `name` / `description` / `capabilities` 等。
**Resources**：引用 `references/` 和 `assets/` 下的内容，让 LLM 在需要时按需加载。
**Script**：引用 `scripts/` 下的可执行脚本（沙箱执行）。

### 三层模型

| 层 | 内容 | 对应数据类 |
|----|------|----------|
| L1 元数据 | name / description / capabilities | `Frontmatter` |
| L2 Instructions | SKILL.md 的 Markdown 正文（skill 自己的 prompt 片段） | `Skill.instructions` |
| L3 资源 | references / assets / scripts | `Resources` / `Script` |

### 从本地目录加载

```python
from google.adk.skills import load_skill_from_dir, list_skills_in_dir

# 加载单个 skill
skill = load_skill_from_dir("/path/to/my_skill")

# 列出某目录下所有 skill（每个子目录一个 skill）
skills = list_skills_in_dir("/path/to/skills_root")
```

### 从 GCS 加载（生产部署用）

```python
from google.adk.skills import load_skill_from_gcs_dir, list_skills_in_gcs_dir

# 单个 skill：gs://my-bucket/skills/my_skill/
skill = load_skill_from_gcs_dir("gs://my-bucket/skills/my_skill")

# 批量列出
skills = list_skills_in_gcs_dir("gs://my-bucket/skills")
```

GCS 变体让 skill 可以独立于 agent 镜像部署，无需重新打包就能更新 skill 内容。

### 编程式构造 Skill

```python
from google.adk.skills import Skill, Frontmatter

skill = Skill(
    frontmatter=Frontmatter(
        name="my_skill",
        description="...",
    ),
    instructions="...",     # Markdown 正文
    # resources=..., scripts=... 见源码
)
```

### 挂到 agent 上

ADK 提供 `DEFAULT_SKILL_SYSTEM_INSTRUCTION`：用于在 agent 的 instruction 里告诉 LLM "你有这些 skill 可用"。具体挂载方式以源码 `src/google/adk/skills/` 和 `contributing/samples/skills_agent` 为准。

**Docs**：https://adk.dev/skills/
**Source**：https://github.com/google/adk-python/tree/main/src/google/adk/skills
**示例**：https://github.com/google/adk-python/tree/main/contributing/samples/skills_agent

**与 Claude Code Skills 的区别**：ADK Skills 是 ADK agent 在运行时加载的能力包；Claude Code Skills 是 Claude Code CLI 自身的扩展机制。两者不通用、不互操作，不要混淆。

---

## Optimization（自动优化 instruction）

**用途**：用 GEPA（Gradient-based Evolutionary Prompt Algorithm）自动优化 agent 的 instruction 文本。

### CLI

```bash
adk optimize <agent_path> --eval_set <eval_set_path>
```

### 典型流程

1. 写好 eval set
2. `adk optimize` 迭代改写 instruction → 跑 eval → 保留更优版本
3. 人工审阅最终 prompt 后提交

**Docs**：https://adk.dev/optimize/

**源码**：`src/google/adk/optimization/`

---

## Integrations 目录（70+）

ADK 社区 / 官方维护的集成，按类别：

### Code & Execution
- Code Execution Tool
- GKE Code Executor
- Daytona
- Computer Use

### Data & Connectors
- BigQuery
- MongoDB
- Pinecone
- MCP Toolbox（30+ 数据源一站式）
- Spanner
- Firestore

### Observability
- AgentOps
- Arize AX
- MLflow
- Phoenix
- W&B Weave

### Search & Knowledge
- Google Search
- Agent Search
- Google Developer Knowledge

### Enterprise
- Atlassian（Jira / Confluence）
- Asana
- Stripe
- GitHub
- Application Integration

### Memory & State
- Database Memory Service
- Firestore Session
- GoodMem
- VertexAI RAG

**浏览总目录**：https://adk.dev/integrations/

**使用方式**：每个集成的文档给 import 和配置示例。不在本库镜像（更新太频繁）。

---

## Workflow 高级模式

### Graph routes / Dynamic workflows

通过 `BaseAgent` 自定义任意图结构的编排（条件分支、回退、并发融合）。

### Human-in-the-loop

- `get_user_choice` tool 让用户选项
- Action Confirmations（见 `tools.md`）
- `LongRunningFunctionTool` 等待人类回合

### 进度可视化

`adk web` 自带 trace view。也可通过 OpenTelemetry 导出到第三方工具（Phoenix / Arize / Weave）。

---

## CLAUDE.md 相关约束

- 外部 API 集成前必须确认：并发限制、命名规则、Schema 变更限制、同步 vs 异步、批量 vs 逐个
- 敏感 tool（代码执行、支付、删除）用 Action Confirmations
- Eval 集成到 `make eval`；项目测试规范 CLAUDE.md "测试规范" 段落
- 确定性判断 vs LLM 判断：能代码决策的用 `BaseAgent` / callback，零 LLM 成本零延迟

---

## 上游参考

- **Safety**：https://adk.dev/safety/
- **Skills**：https://adk.dev/skills/
- **Optimization**：https://adk.dev/optimize/
- **Integrations**：https://adk.dev/integrations/
- **Evaluation 总览**：https://adk.dev/evaluate/
- **Grounding**：
  - Google Search：https://adk.dev/grounding/google_search_grounding/
  - Enterprise Search：https://adk.dev/grounding/grounding_with_search/
- **MCP**：https://adk.dev/mcp/
- **ADK Samples**：https://github.com/google/adk-samples
- **ADK 2.0 预告**：https://adk.dev/2.0/
- **源码**：
  - `src/google/adk/a2a/`
  - `src/google/adk/planners/`
  - `src/google/adk/code_executors/`
  - `src/google/adk/evaluation/`
  - `src/google/adk/skills/`
  - `src/google/adk/optimization/`
  - `src/google/adk/integrations/`
