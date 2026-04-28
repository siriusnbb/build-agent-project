# ADK Sessions / Artifacts / Memory Services

> 三大 service 家族：会话存储、工件存储、长期记忆。每家都有 InMemory / 本地 / 云端三档。
>
> **Last synced:** 2026-04-23 · **adk-python commit:** `62d7ee0` · **Upstream:** https://adk.dev/context/session/ · https://adk.dev/artifacts/ · https://adk.dev/context/memory/

---

## 三家 Service 定位速查

| 职责 | 存什么 | 接口基类 | 装配位置 |
|------|-------|---------|---------|
| **Session** | 会话本身（state + events + 元数据） | `BaseSessionService` | `Runner(session_service=...)` |
| **Artifact** | 二进制数据（文件、PDF、JSON dump 等） | `BaseArtifactService` | `Runner(artifact_service=...)` |
| **Memory** | 跨 session 的长期语义记忆 | `BaseMemoryService` | `Runner(memory_service=...)` |

三者都由 `Runner` 在启动时注入。Tool/Callback 通过 `ctx` 间接使用，不直接接触 service 实现。

---

## Session Services

**Import 总览**：

```python
from google.adk.sessions import (
    BaseSessionService,
    InMemorySessionService,
    DatabaseSessionService,     # 需要 sqlalchemy>=2.0
    VertexAiSessionService,
    Session, State,
)
```

还有一个源码里存在、但不在 `__all__` 中的：

```python
# 仅从源码引入：
# from google.adk.sessions.sqlite_session_service import SqliteSessionService
```

---

### `BaseSessionService`（抽象基类）

**用途**：自建 session service 时继承。定义 CRUD + append_event 接口。

**源码**：`src/google/adk/sessions/base_session_service.py`

**关键方法**（自定义实现时需覆盖）：
- `async def create_session(app_name, user_id, state=None, session_id=None) -> Session`
- `async def get_session(app_name, user_id, session_id) -> Session | None`
- `async def list_sessions(app_name, user_id) -> list[Session]`
- `async def delete_session(app_name, user_id, session_id) -> None`
- `async def append_event(session, event) -> Event`

**Docs**：仅在 API reference 中，https://adk.dev/api-reference/python/

---

### `InMemorySessionService`

**用途**：开发 / pytest 用。进程退出数据就没。

**Import**：`from google.adk.sessions import InMemorySessionService`

```python
session_service = InMemorySessionService()
```

**何时用**：CLI dev、单测、POC。

---

### `DatabaseSessionService`

**用途**：用 SQLAlchemy 支持的数据库（SQLite / PostgreSQL / MySQL / ...）持久化 session。

**Import**：`from google.adk.sessions import DatabaseSessionService`

```python
session_service = DatabaseSessionService(
    db_url="postgresql+psycopg://user:pass@host:5432/adk",
    # 或 sqlite:////path/to/db.sqlite
)
```

**依赖**：`sqlalchemy>=2.0`（ADK pyproject 已声明）。

**何时用**：自建基础设施、私有部署、已有 RDBMS 栈。

**源码**：`src/google/adk/sessions/database_session_service.py`

---

### `VertexAiSessionService`

**用途**：Vertex AI Agent Engine 原生 session 存储。部署到 Agent Engine 时默认就是它。

**Import**：`from google.adk.sessions import VertexAiSessionService`

```python
session_service = VertexAiSessionService(
    project="my-gcp-project",
    location="us-central1",
)
```

**何时用**：生产部署在 GCP Agent Engine；跨区域多实例；最少运维。

**源码**：`src/google/adk/sessions/vertex_ai_session_service.py`

---

### `SqliteSessionService`（源码存在，未导出）

**用途**：SQLite 专用实现（`DatabaseSessionService` 的轻量替代）。

**Import**：
```python
from google.adk.sessions.sqlite_session_service import SqliteSessionService
```

**何时用**：ADK CLI 的 `adk run` 默认用 SQLite 持久化 session（`.adk/session.db`）。应用代码中一般用 `DatabaseSessionService` 并传 `sqlite:///...` URL 更一致。

---

### Session 对象结构

```python
session.id                  # 主键
session.app_name            # app 标识
session.user_id             # 用户 ID
session.state               # State（dict-like，带前缀规则）
session.events              # list[Event]
session.last_update_time    # 更新时间戳
```

`State` 类型详见 `context-and-state.md` 的 "Session State 概述"。

---

## Artifact Services

**Import 总览**：

```python
from google.adk.artifacts import (
    BaseArtifactService,
    InMemoryArtifactService,
    FileArtifactService,
    GcsArtifactService,
)
```

---

### `BaseArtifactService`（抽象基类）

**关键方法**（自定义时覆盖）：
- `async def save_artifact(app_name, user_id, session_id, filename, artifact) -> int`（返回版本号）
- `async def load_artifact(app_name, user_id, session_id, filename, version=None) -> Part`
- `async def list_artifact_keys(app_name, user_id, session_id) -> list[str]`
- `async def delete_artifact(app_name, user_id, session_id, filename) -> None`
- `async def list_versions(app_name, user_id, session_id, filename) -> list[int]`

**源码**：`src/google/adk/artifacts/base_artifact_service.py`

---

### `InMemoryArtifactService`

**Import**：`from google.adk.artifacts import InMemoryArtifactService`

```python
artifact_service = InMemoryArtifactService()
```

开发 / pytest 用。

---

### `FileArtifactService`

**用途**：本地文件系统存储。

**Import**：`from google.adk.artifacts import FileArtifactService`

```python
artifact_service = FileArtifactService(base_dir="/path/to/artifacts")
```

**何时用**：本地长期运行、单机部署、需要查看文件。

**源码**：`src/google/adk/artifacts/file_artifact_service.py`

---

### `GcsArtifactService`

**用途**：Google Cloud Storage 存储。

**Import**：`from google.adk.artifacts import GcsArtifactService`

```python
artifact_service = GcsArtifactService(bucket_name="my-adk-artifacts")
```

**何时用**：生产部署、多实例共享、大文件、需要持久归档。

**源码**：`src/google/adk/artifacts/gcs_artifact_service.py`

---

### Artifact 的 user-scope 命名

Artifact filename 支持前缀区分作用域：

- `"report.pdf"` → session 作用域，仅本 session 可见
- `"user:profile.png"` → user 作用域，跨 session 可见

（具体前缀规则以 API reference 为准，类比 state 前缀。）

---

### 保存 / 加载 / 列出 Artifact

所有操作通过 `Context`（= `ToolContext` 或 `CallbackContext`）进行，不直接调 service：

```python
from google.genai import types

# 保存
payload = json.dumps(data, ensure_ascii=False).encode("utf-8")
part = types.Part.from_bytes(data=payload, mime_type="application/json")
version = await tool_context.save_artifact("report.json", part)

# 加载（version=None 取最新）
part = await tool_context.load_artifact("report.json", version=None)
data = json.loads(part.inline_data.data.decode("utf-8"))

# 列出
filenames = await tool_context.list_artifacts()
```

**在 BaseAgent 中**：`InvocationContext` 不直接暴露这些方法，用 `Context(ctx)` 包装，见 `context-and-state.md`。

---

### Artifact vs State 边界（何时用哪个）

| 数据类型 | 放哪 |
|---------|------|
| 字符串、数字、小 dict（业务状态、元数据） | `session.state` |
| 文件 binary（用户上传、PDF、图片、大 JSON） | `artifact` |
| 需要版本化保留的结构化输出 | `artifact`（JSON 序列化后存） |
| 跨 session 的用户偏好 | `state` with `user:` 前缀 |
| 跨 session 的用户文件 | `artifact` with `user:` 前缀 |

**CLAUDE.md 规则**：文件 binary 必须通过 save/load_artifact 存储，不要塞 state。

---

### Artifact 的 server-side copy（同 backend 优化）

**反模式**：用 `load_artifact()` 把文件 bytes 读到内存，再 `upload_from_string()` 上传到业务 GCS bucket——bytes 完全在应用内存里走一圈，对大 PDF 尤其浪费。

**正确模式**：当 artifact backend 是 GCS（生产 Vertex AI 默认）时，artifact 已经是 GCS object，可以走 GCS-to-GCS 的 server-side copy：

```python
# 1. 取 artifact 的 canonical URI（gs:// 形式）
artifact_version = await artifact_service.get_artifact_version(
    app_name=..., user_id=..., session_id=..., filename=fn, version=version,
)
artifact_uri = getattr(artifact_version, "canonical_uri", None)

# 2. 如果是 gs:// 用 server-side copy
if artifact_uri and artifact_uri.startswith("gs://"):
    _, _, bucket_and_path = artifact_uri.partition("gs://")
    source_bucket_name, _, source_blob_name = bucket_and_path.partition("/")
    source_bucket = client.bucket(source_bucket_name)
    source_blob = source_bucket.blob(source_blob_name)
    source_bucket.copy_blob(source_blob, target_bucket, target_path)
else:
    # 3. fallback: load_artifact + upload_from_string
    artifact = await context.load_artifact(filename=fn)
    bytes_data = artifact.inline_data.data
    target_bucket.blob(target_path).upload_from_string(bytes_data)
```

**适用范围**：S3-to-S3 / GCS-to-GCS / 同库 DB-to-DB 的场景永远优先 server-side copy。fallback 路径处理 backend 不是 GCS（如 `InMemoryArtifactService`、`FileArtifactService`）的情况。

---

## Memory Services

**Import 总览**：

```python
from google.adk.memory import (
    BaseMemoryService,
    InMemoryMemoryService,
    VertexAiMemoryBankService,
    VertexAiRagMemoryService,   # 需要 Vertex SDK
)
```

---

### `BaseMemoryService`（抽象基类）

**关键方法**（自定义时覆盖）：
- `async def add_session_to_memory(session) -> None` — 把 session 加入长期记忆
- `async def search_memory(app_name, user_id, query) -> SearchMemoryResponse` — 语义检索

**源码**：`src/google/adk/memory/base_memory_service.py`

---

### `InMemoryMemoryService`

开发 / pytest 用，进程级存储。

```python
memory_service = InMemoryMemoryService()
```

---

### `VertexAiMemoryBankService`

**用途**：Vertex AI 原生 Memory Bank，语义检索能力。

**Import**：`from google.adk.memory import VertexAiMemoryBankService`

```python
memory_service = VertexAiMemoryBankService(
    project="my-gcp-project",
    location="us-central1",
    agent_engine_id="...",
)
```

**何时用**：Agent Engine 部署、需要跨 session 的语义记忆。

---

### `VertexAiRagMemoryService`

**用途**：基于 Vertex AI RAG Engine 的记忆服务。适合把业务文档库当长期记忆。

**Import**：`from google.adk.memory import VertexAiRagMemoryService`

**依赖**：`google-adk[extensions]` 或 Vertex SDK。

---

### 通过 Tool 访问 Memory

在 agent 的 tools 里放 `load_memory` / `preload_memory`：

```python
from google.adk.tools import load_memory, preload_memory

Agent(
    tools=[load_memory],    # LLM 可决定何时查记忆
    ...
)
```

- `load_memory`：LLM 主动调，基于当前对话查相似记忆
- `preload_memory`：入口自动预热，常放在 `before_agent_callback` 触发

---

## 三家 Service 的 Runner 装配

```python
from google.adk.runners import Runner
from google.adk.sessions import VertexAiSessionService
from google.adk.artifacts import GcsArtifactService
from google.adk.memory import VertexAiMemoryBankService

runner = Runner(
    agent=root_agent,
    app_name="my_app",
    session_service=VertexAiSessionService(project=..., location=...),
    artifact_service=GcsArtifactService(bucket_name=...),
    memory_service=VertexAiMemoryBankService(...),
)
```

详细用法见 `runtime-and-deploy.md` 的 "Runner" 章节。

---

## Service 选型决策（再次）

| 场景 | Session | Artifact | Memory |
|------|---------|----------|--------|
| 开发 / pytest | `InMemorySessionService` | `InMemoryArtifactService` | `InMemoryMemoryService` |
| 本地持久化 | `DatabaseSessionService(sqlite:///...)` | `FileArtifactService` | （无本地版，可用 `InMemory`） |
| GCP 生产（Agent Engine） | `VertexAiSessionService` | `GcsArtifactService` | `VertexAiMemoryBankService` |
| GCP 生产（Cloud Run / GKE，自建） | `DatabaseSessionService(postgres://...)` | `GcsArtifactService` | 按需 |

---

## CLAUDE.md 相关约束

- 文件 binary 必须走 save/load_artifact，不塞 session state
- Artifact 后端由 `artifact_service` 装配，代码中不关心
- ADK session 自带 `session.id`，不要自建 UUID 作为 session 标识
- State 只存轻量数据；大数据用 Artifact 或其他合适机制
- 批量操作错误处理：用 try/except 包循环体，失败继续，返回 {total, success, failed, failed_items}
- 异步清理：`asyncio.ensure_future + asyncio.to_thread`，失败只 log warning

---

## 上游参考

- **Sessions**：https://adk.dev/context/session/
- **Memory**：https://adk.dev/context/memory/
- **Artifacts**：https://adk.dev/artifacts/
- **Python API Reference**：https://adk.dev/api-reference/python/
- **Integrations（第三方 backends）**：
  - Firestore Session：https://adk.dev/integrations/
  - GoodMem：https://adk.dev/integrations/
  - Database Memory Service：https://adk.dev/integrations/
- **源码**：
  - `src/google/adk/sessions/`
  - `src/google/adk/artifacts/`
  - `src/google/adk/memory/`
