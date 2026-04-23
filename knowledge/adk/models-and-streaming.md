# ADK Models & Streaming

> Model providers（Gemini / Claude / LiteLlm / Gemma / Ollama / vLLM / Apigee）+ Live API / bidi streaming。
>
> **Last synced:** 2026-04-23 · **adk-python commit:** `62d7ee0` · **Upstream:** https://adk.dev/agents/models/ · https://adk.dev/streaming/

---

## Model Provider 总览

| Provider | Import | 源码 | 备注 |
|----------|--------|------|------|
| `Gemini` | `from google.adk.models import Gemini` | `models/google_llm.py` | 默认推荐，原生支持 tool-use / Live API / grounding |
| `Gemma` | `from google.adk.models import Gemma` | `models/gemma_llm.py` | 本地部署的开源 Gemma 系列 |
| `Claude` | `from google.adk.models import Claude` | `models/anthropic_llm.py` | 需 `google-adk[extensions]` + `anthropic` |
| `LiteLlm` | `from google.adk.models import LiteLlm` | `models/lite_llm.py` | 多 provider 统一抽象（OpenAI / Azure / Cohere / Bedrock / ...） |
| `Gemma3Ollama` | `from google.adk.models import Gemma3Ollama` | `models/lite_llm.py` | 本地 Ollama 上的 Gemma 3（via litellm） |
| `ApigeeLlm` | 仅源码：`from google.adk.models.apigee_llm import ApigeeLlm` | `models/apigee_llm.py` | 不在 `__all__`，仅源码可用 |
| `BaseLlm` | `from google.adk.models import BaseLlm` | `models/base_llm.py` | 自建 provider 时继承 |
| `LLMRegistry` | `from google.adk.models import LLMRegistry` | `models/registry.py` | 按名字查找已注册的 LLM |

---

## `Gemini`（默认）

**Import**：`from google.adk.models import Gemini`

```python
from google.adk.models import Gemini
from google.genai import types

model = Gemini(
    model="gemini-2.0-flash",   # 或其他 Gemini 模型名
    retry_options=types.HttpRetryOptions(attempts=3),
    # 低层参数：
    # system_instruction=...
    # safety_settings=[...]
)

Agent(model=model, ...)
```

**等价的字符串形式**（大多数情况够用）：
```python
Agent(model="gemini-2.0-flash", ...)
```

ADK 自动包成 `Gemini`。

**何时用显式 Gemini 实例**：
- 需要自定义 retry
- 需要传 safety settings
- 需要 `generate_content_config`

**模型名清单**：去 Vertex AI / AI Studio 官方文档查最新。本库不镜像（更新太频繁）。

**Docs**：https://adk.dev/agents/models/google-gemini/

---

## `Gemma`（本地）

**Import**：`from google.adk.models import Gemma`

**用途**：本地运行的 Gemma 系列（1B / 2B / 7B 等），无需联网。

**Docs**：https://adk.dev/agents/models/google-gemma/

---

## `Claude`

**Import**：`from google.adk.models import Claude`（需 `google-adk[extensions]`）

```python
from google.adk.models import Claude

model = Claude(model="claude-3-5-sonnet-20241022")
Agent(model=model, ...)
```

**依赖**：`anthropic>=0.43.0`（pyproject 已声明为可选）。

**Docs**：https://adk.dev/agents/models/anthropic/

---

## `LiteLlm`（多 provider 抽象）

**Import**：`from google.adk.models import LiteLlm`（需 `google-adk[extensions]`）

**用途**：统一接口调 OpenAI / Azure / Cohere / Bedrock / Ollama / vLLM / 100+ provider。

```python
from google.adk.models import LiteLlm

model = LiteLlm(model="openai/gpt-4o-mini")
# 或 "anthropic/claude-3-5-sonnet"
# 或 "ollama/llama3.1:8b"
# 或 "vertex_ai/gemini-2.0-flash"
Agent(model=model, ...)
```

**依赖**：`litellm>=1.75.5,<=1.82.6`（pyproject 已声明）。

**Docs**：https://adk.dev/agents/models/litellm/

---

## `Gemma3Ollama`

**Import**：`from google.adk.models import Gemma3Ollama`（需 `google-adk[extensions]`）

**用途**：专门封装 Ollama 上的 Gemma 3，继承 `LiteLlm`。

**Docs**：
- Ollama：https://adk.dev/agents/models/ollama/
- Gemma 本地：https://adk.dev/agents/models/google-gemma/

---

## `ApigeeLlm`（仅源码）

**Import**：`from google.adk.models.apigee_llm import ApigeeLlm`（不在 `__all__`）

**用途**：通过 Apigee AI Gateway 访问企业级模型（企业网关场景）。

**Docs**：https://adk.dev/agents/models/apigee/（页面结构可能简略，以源码为准）

---

## `BaseLlm`（自建 provider）

**Import**：`from google.adk.models import BaseLlm`

继承并实现 `generate_content_async` / `connect`（for live） 等方法。注册到 `LLMRegistry` 后即可 `Agent(model="my_provider/...")`。

源码：`src/google/adk/models/base_llm.py`

---

## 模型 Switching 实操

**换 provider 只需改一行**：

```python
# 1. Gemini（默认）
Agent(model="gemini-2.0-flash", ...)

# 2. Claude
from google.adk.models import Claude
Agent(model=Claude(model="claude-3-5-sonnet-20241022"), ...)

# 3. OpenAI via LiteLlm
from google.adk.models import LiteLlm
Agent(model=LiteLlm(model="openai/gpt-4o-mini"), ...)

# 4. 本地 Ollama
Agent(model=LiteLlm(model="ollama/llama3.1:8b"), ...)
```

**注意**：不同 provider 对 tool-use、Live API、grounding 等的支持程度不同。Gemini 原生支持度最高。

---

## vLLM / LiteRT-LM（特殊）

- **vLLM**：高性能推理引擎，通常通过 `LiteLlm(model="openai/...")` 指向本地 vLLM OpenAI 兼容端点
  - Docs：https://adk.dev/agents/models/vllm/
- **LiteRT-LM**：边缘设备（如 mobile）模型
  - Docs：https://adk.dev/agents/models/litert-lm/

---

## Agent Platform hosted Models

**用途**：通过 Agent Engine 托管的模型访问多家 provider 模型。

**Docs**：https://adk.dev/agents/models/agent-platform/

---

## Streaming / Live API 总览

**Import**：
```python
from google.adk.agents import LiveRequest, LiveRequestQueue, RunConfig
```

源码：`src/google/adk/agents/live_request_queue.py`

**三种"实时"模式：**

| 模式 | 何时用 | 配置 |
|------|-------|------|
| 普通流式文本 | 聊天逐字显示 | `RunConfig(streaming_mode=STREAMING)` 或 runner 自动 |
| Live API（bidi 音视频） | 语音对话 / 视频感知 | `RunConfig(response_modalities=["AUDIO"])` + Gemini Live API |
| Streaming tools | tool 执行期间产出中间结果 | tool 函数用 `async generator` |

**Docs**：
- 总览：https://adk.dev/streaming/
- 开发指南 5 部分：https://adk.dev/streaming/dev-guide/
- Quickstart Python：https://adk.dev/get-started/streaming/quickstart-streaming/

---

## 普通流式文本

**默认行为**：Runner 的 `run_async` 本来就是 `AsyncGenerator[Event]`，`event.partial=True` 是流式中间帧。

**在 Web UI 中**：`adk web` 默认支持流式显示。

---

## Live API（双向音视频）

**用途**：Gemini Live API 支持实时双向音频 / 视频流，用户可打断 agent、agent 可感知环境。

**核心对象**：
- `LiveRequestQueue`：向 agent 推送音/视频/文本帧
- `LiveRequest`：单个输入帧（audio / video / text）

**典型架构**（概要，完整代码去上游）：

```python
from google.adk.agents import LiveRequestQueue, LiveRequest, RunConfig

queue = LiveRequestQueue()

async def input_task():
    # 从麦克风 / 摄像头读帧
    while True:
        frame = read_frame()
        await queue.put(LiveRequest(audio_data=frame))

async def output_task():
    async for event in runner.run_live(
        queue=queue,
        run_config=RunConfig(response_modalities=["AUDIO", "TEXT"]),
    ):
        if event.content:
            play_audio(event.content)

await asyncio.gather(input_task(), output_task())
```

（具体 API 以 `src/google/adk/agents/live_request_queue.py` 为准。）

**Docs 开发指南 5 部分（必读）：**
1. Part 1：基础、架构、FastAPI 示例
2. Part 2：消息流、多模态内容
3. Part 3：事件处理、工具执行
4. Part 4：RunConfig、会话管理、上下文优化
5. Part 5：音频规范、转录、语音活动检测

URL：https://adk.dev/streaming/dev-guide/

---

## Streaming Tools（工具流式中间结果）

**用途**：tool 执行耗时长，希望逐步返回进度或中间结果。

**形式**：tool 函数声明为 `async generator`（`yield` 多次），ADK 把中间帧作为 partial event 流出。

```python
async def long_job(task_id: str, tool_context: ToolContext):
    """Run a long job and stream progress."""
    for i in range(10):
        await asyncio.sleep(1)
        yield {"progress": f"{i*10}%"}
    yield {"status": "done"}
```

**Docs**：https://adk.dev/streaming/streaming-tools/

---

## 何时选 Streaming vs 常规 invocation

| 场景 | 选择 |
|------|------|
| 简单 Q&A / 批处理 | 常规 `run_async`，读完整 event 再用 |
| UI 要逐字显示 | `run_async`（已自带流式 partial events） |
| 语音对话 / 视频 | Live API（`run_live` + `LiveRequestQueue`） |
| Tool 耗时 > 几秒需要进度条 | Streaming tools（async generator） |

---

## CLAUDE.md 相关约束

- 异步强制用 `asyncio`，禁用 threading
- 外部 API 调用必须 log endpoint / 耗时 / 状态 / 重试次数
- LLM 调用 > 1s 必须 log duration_ms
- LLM prompt/response 生产仅 DEBUG 级别 log 全文，INFO 只 log 长度 + tool_calls 名字
- Retry 循环中 sleep 放状态检查之后（检查 → 失败 → sleep → 重查）

---

## 上游参考

- **Models 总览**：https://adk.dev/agents/models/
- **Google Gemini**：https://adk.dev/agents/models/google-gemini/
- **Google Gemma**：https://adk.dev/agents/models/google-gemma/
- **Anthropic Claude**：https://adk.dev/agents/models/anthropic/
- **Ollama**：https://adk.dev/agents/models/ollama/
- **vLLM**：https://adk.dev/agents/models/vllm/
- **LiteLLM**：https://adk.dev/agents/models/litellm/
- **LiteRT-LM**：https://adk.dev/agents/models/litert-lm/
- **Agent Platform hosted**：https://adk.dev/agents/models/agent-platform/
- **Apigee**：https://adk.dev/agents/models/apigee/
- **Streaming 总览**：https://adk.dev/streaming/
- **Streaming Dev Guide (5 parts)**：https://adk.dev/streaming/dev-guide/
- **Streaming Quickstart Python**：https://adk.dev/get-started/streaming/quickstart-streaming/
- **Streaming Tools**：https://adk.dev/streaming/streaming-tools/
- **Get Started Streaming**：https://adk.dev/get-started/streaming/
- **源码**：
  - `src/google/adk/models/`
  - `src/google/adk/agents/live_request_queue.py`
