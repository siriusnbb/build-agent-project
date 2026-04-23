# Agent 项目编码规范

本文档定义 Agent 项目的实现层规范：编码风格、架构约束、state 设计、
日志与可观测性、错误处理、测试模式等。由 Phase 3–7 的 agent 按需加载，
不在 CLAUDE.md 全局注入，避免 context 膨胀。

## 编码风格 (Coding Standards)
所有新函数必须包含 Google 风格的 Docstrings。异步函数必须使用 asyncio，禁止使用 threading
代码要保证简洁，模仿人类编程的流程，功能从少到多，尽量保证实现某个功能的逻辑单一易懂，如果有和其他功能交互的需求，明确边界条件和变量

## 架构约束 (Architecture Rules)
在修改 Agent 逻辑时，始终优先检查 ctx.session.state。不要创建全局变量来存储用户信息

## Agent 类型选择
- 创建 Agent 前必须先问：这个逻辑需要 LLM 判断吗？
- 如果逻辑是确定性的（if/else 可以决定），使用确定性判断方法，比如BaseAgent，不使用 LlmAgent（Agent）
- BaseAgent 的优势：零 LLM 成本、零延迟、无不确定性、行为完全可预测
- 典型的 BaseAgent 场景：文件上传处理、索引验证、数据格式转换、条件分支路由
- 典型的 LlmAgent 场景：自然语言理解、动态查询构建、内容生成、用户对话

## State 设计原则
- 每个 state key 必须有唯一明确的职责，不允许一个 key 同时承担多个含义
- 不要用 state value 的前缀/格式来编码状态（如用 "gcs:xxx" 表示失败）——用独立的 boolean flag
- 如果一个值来自环境变量且不会变，不需要存入 state——直接从 config 读取
- ADK session 自带唯一 ID（session.id），除非有特殊需求， 不要自行生成 UUID 作为 session 标识
- State 只存轻量数据（字符串、数字、小型 dict）。二进制数据和大数据用 ADK Artifact 机制或其他合适的方式存储
- State key 命名要让不了解上下文的人也能理解。避免缩写和内部术语

## ADK Artifact 使用规范
- 文件的 binary 数据必须通过 save_artifact / load_artifact 存储，不要存在 session state 中
- Artifact 的存储后端（GCS / 本地文件 / 内存）由 artifact_service_builder 在应用初始化时配置，代码中不需要关心
- CallbackContext = Context，callback 中可以直接使用 save_artifact
- BaseAgent 的 _run_async_impl(ctx: InvocationContext) 中，用 Context(ctx) 构建 Context 来访问 artifact
  
## 外部 API 集成前必须确认的事项
- API 的并发限制（例：Discovery Engine 不允许同一 branch 并行 import）
- 命名规则和字符限制（例：Document ID 只允许 [a-zA-Z0-9_-]）
- Schema 变更限制（例：Discovery Engine 不允许通过 update_schema 删除字段）
- 操作是同步还是异步（例：import_documents 返回 LRO，可以不等待完成）
- 批量操作 vs 逐个操作的限制和最佳实践

## 文件上传和 Data Store 导入模式
- GCS 上传：逐个文件上传，GCS 支持日语等 Unicode 文件名
- Data Store 导入：所有文件上传完毕后，构建一个 JSONL 清单文件一次性批量导入
- Document ID：在文件名有可能出现不支持字符时，构建专门的命名逻辑，保证能够追踪到文件个体。举例：比如用 SHA-256 hash 生成确定性且唯一的 ID，避免非 ASCII 字符导致的碰撞，但是保证和文件原名和GCS保存uri的联系
- 文件名对照表：在 upload record 中同时保存 original_name（原始名）、file_name（GCS 用名）、doc_id（Data Store 用 ID），便于下游反查来源
- 索引验证：导入后不阻塞等待，在合适的地方（一般是使用查询前一步）中验证索引完成

## 确定性判断 vs LLM 判断
- 如果判断条件可以通过代码获取（如文件是否存在、队列是否为空），必须用代码确定性判断
- 不要把可以确定的判断交给 LLM（如让 LLM 判断用户是否上传了文件）
- Root callback 是确定性设置 state 的最佳位置（在 LLM 执行前就已确定）

## Prompt 管理
- 不要复制粘贴 prompt。共通部分定义为一个基础 prompt，差异部分用条件追加
- Prompt 中的条件选择逻辑放在 prompt 文件中（不要放在 agent 文件中），保持关注点分离
- Prompt 中不要引用已删除的工具或已变更的 state key 名

## 错误处理模式
- 循环处理多个项目（文件上传、索引验证等）时，用 try/except 包住循环体，失败时记录并继续处理剩余项目
- 返回结果要包含：总数、成功数、失败数、失败项目的具体名称
- 异步清理（如删除未索引文件）用 asyncio.ensure_future + asyncio.to_thread，不阻塞主流程
- 清理失败只 log warning，不影响 pipeline 继续
- 捕获异常时必须用 `logger.exception()`（或 `logger.error(..., exc_info=True)`）保留 stack trace，不要只记录 `str(e)`
- 不允许 `except Exception: pass` 静默吞异常；即使选择忽略也要 log warning 并说明原因

## 日志与可观测性 (Logging & Observability)
- **统一 logger**：用 Python 标准 `logging` 模块，每个模块 `logger = logging.getLogger(__name__)`，禁止使用 `print()` 输出运行时信息
- **结构化输出**：生产环境配置 JSON formatter（字段含 timestamp / level / logger / message / session_id / invocation_id / agent_name），方便 Cloud Logging 过滤
- **Log level 使用规则**：
  - `DEBUG`：state 快照、tool 入参出参详细内容、循环内部每项状态
  - `INFO`：agent 入口/出口、关键业务步骤完成、外部 API 调用成功、批量操作汇总
  - `WARNING`：重试、降级路径、部分失败、清理失败
  - `ERROR`：异常、pipeline 中断、外部 API 不可恢复失败
- **Agent 生命周期必须 log**：
  - 每个 BaseAgent 的 `_run_async_impl` 入口 log `INFO` "agent {name} start"，出口 log `INFO` "agent {name} done"（含耗时）
  - LlmAgent 的 before/after callback 必须 log 触发事件
  - 关键决策分支（if/else 路由）必须 log 决策依据（哪个条件为真、对应的 state key 值）
- **Tool 调用必须 log**：
  - 调用前 log `INFO` "tool {name} called"（带关键入参摘要，不是全量）
  - 调用后 log `INFO` "tool {name} done" 或 `ERROR` "tool {name} failed"（带耗时）
- **外部 API 调用必须 log**：endpoint、耗时（毫秒）、返回状态、重试次数；失败时额外 log request payload 摘要（脱敏后）
- **循环进度 log**：处理 N 个项目时，至少记录 "processing {i}/{N}: {item_name}"，便于定位卡点
- **关联 ID**：所有 log 必须带 `session.id` 和 `invocation_id`（通过 `extra={}` 注入或 LoggerAdapter），支持按单次调用链路追查
- **耗时记录**：任何预期 > 500ms 的操作（LLM 调用、外部 API、文件上传）必须 log 开始和结束时间或 duration_ms

## 调试上下文 (Debugging Context)
- **失败路径 context**：抛异常或进入 error 分支前，必须 log 足够重建现场的信息——关键 state key 当前值、刚完成的步骤、预期下一步
- **State 变更**：对业务关键的 state key 写入（如 upload_result、index_status），log 变更前后的值（DEBUG 级别）
- **跳过/早返回**：BaseAgent 判断后跳过某步或直接返回时，必须 log 原因（"skip index verify: state.index_ready=true"）
- **长耗时操作**：LLM / Discovery Engine / GCS 等调用若 > 1s，log 包含操作名和 duration_ms，用于性能回溯
- **测试友好**：单元测试可用 `caplog` fixture 验证关键 log 是否输出，但 log 内容本身不作为业务断言依据

## 敏感数据脱敏 (Log Redaction)
- **绝对禁止出现在 log 中**：API key、access token、service account JSON、用户密码、完整个人信息（邮箱/电话/身份证号的完整串）
- **文件内容**：用户上传的文件 binary / 文本内容不得整体打印，只 log 元信息（文件名、大小、MIME、doc_id）
- **LLM prompt 和响应**：生产环境仅在 DEBUG 级别 log 全文；INFO 级别只 log 长度和 tool_calls 名称
- **大对象摘要**：dict / list 长度超过阈值（如 >10 项或 >1KB）时，log 摘要（长度、前几个 key），不 log 全量
- **用统一脱敏 helper**：在 `shared_libraries/` 下提供 `redact_sensitive(data)` 工具函数，新代码复用而不是各自实现

## 测试规范
- 生产代码不要为了测试而修改函数签名（不要加 sleep_fn、get_document 等注入参数）。用 unittest.mock.patch mock 外部依赖
- 单元测试和集成测试严格分开：单元测试 mock 所有外部依赖，集成测试验证模块间交互
- 必须用真实场景数据做边界测试（日语文件名、长文件名、同名文件等）
- 测试代码要和实际代码保持一致——如果功能被禁用或删除，对应的测试也要更新或删除

## 模块依赖规则
- shared_libraries/ 不得 import sub_agents/ 下的模块（避免循环依赖）
- 工具函数（utility function）放在 shared_libraries/ 或 app_utils/ 下
- 只被一个 agent 使用的逻辑放在该 agent 的目录下；被多个模块使用的放在共享目录

## 重试模式
- 重试循环中，sleep 放在状态检查之后（检查 → 失败 → sleep → 再检查），不是之前
- 最后一次失败后不 sleep，直接返回结果
- 部分成功模式：如果部分项目成功，使用已成功的继续推进，异步清理失败的
