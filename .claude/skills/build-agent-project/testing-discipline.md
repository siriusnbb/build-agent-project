# 测试纪律（testing-discipline）

> 这份文档是 implementer / test-engineer / integrator 写代码或写测试时的**自检清单**。
> 跟 acceptance-yaml-schema.md 互补：
> - acceptance-yaml-schema 解决「**验收**怎么不被 LLM 自由解读」
> - 本文档解决「**测试**怎么不被 happy-path 自我证明」
>
> 起源：早期 web_data_search v1 项目 144 unit test 全过、Phase 8 报「21/23 P0 PASS」，但外部 ChatGPT 5 轮独立 review 仍抓出 11 个真 bug。归根：测试只覆盖 happy path，对真实数据形态 / 边角 case / 层间对账 / 反向验证 / 文档语义 5 类系统性盲点没意识。本清单从这 5 类扩到通用 20 项。

---

## 怎么用

### implementer 写代码 + 写测试时

每写完一个函数 / 一组 tool 调用，**对照下表自检**：
- 这件事会不会踩 A1-D5 中的某项盲点？
- 我的测试覆盖了相关项吗？
- 没覆盖的，是「这项不适用」还是「我应该补」？

不强求全 20 项都有测试 — 但**没覆盖的项必须能说出「为什么不适用」**。沉默 = 漏掉。

### test-engineer 写测试时

把本清单作为**测试覆盖矩阵**：
- 每条 P0 关键测试，对照 20 项标注覆盖了哪几项
- test-report 必须列出「未覆盖项 + 原因」

### integrator 在 Phase 8 audit 时

抽样 5-10 个核心模块，按本清单逐项排查；漏掉的列入 KNOWN GAP。

### 对照 acceptance.yaml

- acceptance.yaml 的 auto_check 同样适用本清单（写 check 时考虑「我这条 check 防的是哪类盲点？」）
- 反向测试 (acceptance-yaml-schema §反向测试) = 本文档 D1

---

## A. 数据形态盲点（输入端）

### A1. 真实数据 vs mock 数据

**定义**：mock 数据来自开发者想象，跟真实生产数据形态可能差距巨大（编码 / 字符集 / 特殊字符 / 长度 / null vs ""）。

**反例**（v1）：FTS BM25 测试 mock 数据全 ASCII 关键词，从没传过 `トヨタ` 这种无空格 CJK 串。结果生产环境用户搜日文永远 0 命中。

**对策**：
- 解析器 / URL 处理 / 文件格式 类测试**禁止凭空 mock**，必须有 `tests/fixtures/` 真采样
- 真采样从用户提供 / 从生产抓样 / 从开源数据集
- 数据敏感的（含 PII / token）**必须脱敏**后存

### A2. 边界值

**定义**：空字符串 / null / undefined / 0 / 1 / 极小 / 极大 / 单字符 / 全 CJK / 仅空白 / 含 unicode 控制字符。

**反例**：列表 API 写了 `if not items: skip`，但 `items=[]` 和 `items=None` 行为不同；用户传 `?stock_code=` 空串绕过过滤。

**对策**：
- 每个字符串入参测试至少 4 case：正常 / 空 / 极长 / 特殊字符
- 数值入参至少 5 case：0 / 1 / -1 / 上界 / 下界
- 列表入参至少 3 case：空 / 单元素 / 多元素

### A3. 时间 / 时区

**定义**：UTC vs 本地 / DST 跳变（春秋两小时）/ 跨天 23:59 / leap second / 时间戳精度（秒 vs 毫秒 vs 纳秒）/ 字符串格式（ISO 8601 vs RFC 3339 vs 自定义）。

**反例**：调度器用 `datetime.now()` 没指定时区，dev 机 UTC / 部署机 JST，时间窗判断错位 9 小时。

**对策**：
- 内部 timestamp 全用 UTC + `tzinfo`，IO 边界（用户/UI）才转本地时区
- 测试至少覆盖 23:59 跨天 / 月底 / 闰年 / DST 切换日

### A4. 重复 / 幂等性

**定义**：同输入两次的行为：第二次应产生相同结果（幂等）或显式拒绝（去重）。不能默默二次写入造成数据重复 / orphan。

**反例**（v1）：`INSERT OR IGNORE INTO news` 命中重复 URL 时静默 ignore，但代码继续用新生成的 `news_id` 写 `INSERT INTO attachments` → orphan attachment（关联到不存在的 news）。

**对策**：
- 写入前先 SELECT 确认是否已存在；存在则**复用 id** 或**显式拒绝**，不要盲生成新 id
- 测试每个写入端点至少 3 case：首次 / 重复 / 并发两次（concurrent）

### A5. 注入 / 路径穿越

**定义**：用户输入直接拼 SQL / `os.system` / 文件路径 / shell 命令 / regex pattern。`../` 在路径里、`'; DROP TABLE` 在 SQL 里、`$(cmd)` 在 bash 里。

**反例**：file 下载端点 `path = base / user_input`，没校验 `..`，用户传 `../../../../etc/passwd` 能读任意文件。

**对策**：
- SQL 全用参数化（`?` 占位）或 ORM
- 文件路径用 `pathlib.Path.resolve().is_relative_to(base)` 校验
- shell 命令禁用 `shell=True`；必须时用 `shlex.quote`
- 测试每个边界端点至少 1 个 negative case：`'..'` / `';--` / `${}`

---

## B. 行为盲点（处理过程）

### B1. 并发 / 竞态 / async race / 锁顺序

**定义**：多 worker / 多 async task 同时操作共享资源（DB / 文件 / 内存对象）。lock 嵌套顺序不一致导致死锁。SQL 隔离级别问题。

**反例**：daily_sync 跑时 worker 也在写同一行 `last_seen_at` → DuckDB 写锁竞争 → 一边超时；多 NewsTask 并发抓同一 URL → 两边都生成新 news_id 然后 UNIQUE 冲突。

**对策**：
- 共享状态写访问全用 `asyncio.Lock` / DB 层 transaction
- DB schema 加 UNIQUE 约束 + 应用层 try/except IntegrityError 兜底
- 测试至少 1 个 `asyncio.gather([same_op, same_op])` 并发 case

### B2. 资源耗尽

**定义**：内存（无界 cache / 大 JSON）/ 文件句柄（不关 file / DB 连接）/ 网络连接（连接池满）/ 进程数 / 磁盘空间。

**反例**：循环里 `requests.get(url)` 不用 session，连接堆积；`for row in db.fetch_all(...)` 一次拉百万行入内存。

**对策**：
- 长跑流式处理用 generator / iterator（`fetchmany` 分批）而非 `fetchall`
- 文件 / 连接全用 context manager（`with`）
- 测试至少 1 个 large-data case（10K+ 元素）观察内存

### B3. 状态机 / 不合法转换

**定义**：实体生命周期有状态（pending → running → completed → archived）。某些转换合法，某些不合法。代码必须显式拒非法转换 + 处理 unreachable 状态。

**反例**：task 表 status 字段：A `pending → completed` 直接跳过 running 没拦截；B 状态字符串拼写错 `compeleted` 永远没人匹配，task 永远卡。

**对策**：
- 状态用 Enum 不用 string
- 转换函数显式列允许的 from→to，其他 raise
- 测试每个状态机至少 1 个非法转换被拒 case

### B4. 业务规则边界

**定义**：「最多 N 次」「至少 X 元」「优先 A 后 B」之类规则，N / X 边界值的行为；优先级冲突时的 tie-breaker。

**反例**（v1 类似）：`MAX_RETRIES = 5` 实际看代码是 retry 5 次还是尝试 6 次（含首次）？业务说 5 次但实际跑 6 次 → 反爬触发。

**对策**：
- 边界值（N / N-1 / N+1 / 0）必须有测试断言
- tie-breaker 在文档明示（按 ID 升序 / 按时间降序），测试用相同优先级的元素验

### B5. 错误处理路径（异常吞噬 / silent failure）

**定义**：`except: pass` / `except Exception: continue` / 函数返 `None` 不抛 / 默认值掩盖错误。错误被吞 → 用户看似成功实则失败。

**反例**（v1）：smoke.sh `jq -e ... || echo FAIL` 拼成 `false\nFAIL` 字符串既不等 FAIL 也不等 false → 真失败被假通过。

**对策**：
- 收窄 except 到具体异常类型 + 加注释说明为什么吞
- 协议C（CLAUDE.md）：所有 skip / 默认值 / 容错路径必须显眼上报，禁止默默 skip
- 测试每个有 fallback 路径的端点都验「故意造错 → 必须显式失败」

---

## C. 集成盲点（跨模块/跨服务）

### C1. 层间数据流对账

**定义**：跨模块 / 跨进程 / 跨服务的数据流，A 模块输出 ⇔ B 模块输入 必须语义一致（字段名 / 类型 / 约束 / 业务概念）。

**反例**（v1）：`parse_news_list_html` 识别 3 种 PDF 模式标 `has_pdf=True`；`fetch_news_detail` 只识别 1 种 → 真实 disclosure URL（`/disclosures/pdf/.../`）list 标了 has_pdf 但 detail 不当 PDF 处理 → 永远抓不到。

**对策**：
- 跨模块共享的"概念"抽 module-level helper（如 `is_pdf_url(url)`），所有调用方共用
- 写 contract test：跑 list 真出一批数据，喂给 detail，断言行为一致
- 测试至少 1 个端到端 case（真 fixture → list → detail → DB）

### C2. 容器 vs host 路径 / 环境隔离

**定义**：Docker volume 挂载点 / Windows POSIX 路径分隔符 / dev vs prod 配置 / 容器内 vs 宿主 fs 视图差异。

**反例**（v1）：`docker-compose.yml` 挂 `./browser_context:/app/browser_context`；代码写 `browser_context_<os>/<id>` → 容器内路径错位 → cookie 重启丢。

**对策**：
- 路径常量集中在一处（不要散在多个文件）
- 测试至少 1 个 docker-compose path validation（路径应能 cd 到，文件应在 mount 点下）
- Windows 兼容用 `pathlib` 不用字符串拼

### C3. 第三方服务变更

**定义**：上游 API 改字段 / 改 endpoint / 加限流 / 临时离线 / 返 200 但 body 是错误页 / HTML 而非预期 JSON。

**反例**（v1）：kabutan PDF URL 返 200 但 content-type=text/html（被踢回登录页），代码看 status==200 就 `write_bytes` → HTML 被当 PDF 存盘 → 用户「看到附件但打不开」。

**对策**：
- 第三方响应必须验**content-type + body magic bytes 双重**（不只 status）
- 解析失败 / 字段不存在 → raise 而非默认值
- 测试至少模拟 1 个 200-but-wrong-body case

### C4. 冷启动 vs 热路径

**定义**：第一次跑 vs 第 N 次跑行为可能不同（缓存填充前后 / 索引建立前后 / 数据从 0 到有 / 配置重载）。

**反例**：FTS 索引在 db.init() 后才 PRAGMA create，第一次 query 时索引未就绪 → 报 BinderException → 用户初装时搜索全空。

**对策**：
- 关键资源（索引 / 缓存 / 后台 worker）启动检查 + 自愈
- 测试至少 1 个 fresh DB / 0 row case（不要永远基于「已有数据」假设）

### C5. 文档/语义对账

**定义**：docstring / API 描述 / UI 文案 vs 实际行为 一致性。文档说做 X，实际是否真做 X？还是仅"曾经做过"或"应该做"？

**反例**（v1）：`/api/watchlist/sync` API 文档声明 "每日同步检测三事件"，但实际 `daily_sync()` 只刷 `last_seen_at` 不做 diff；UI 按钮叫"立即 daily sync"反馈"新增 X / 退市 Y"全 0，用户误以为没事发生。

**对策**：
- 改实现时同步改 docstring / API description / UI 文案；不允许"将来再说"
- 复杂行为函数加 example 注释（输入 / 输出 / 副作用）
- code review 时主动查 docstring vs 行为是否一致

---

## D. 测试自身的元盲点

### D1. negative test（反向测试）

**定义**：写完测试断言后**故意改坏代码或 expect**，确认测试能报 FAIL。如果改坏后还 PASS → 测试是死的（永远过的废测试）。

**反例**（v1）：smoke.sh 表面 15/15 pass，故意改坏一个 has() 字段还 pass → 暴露 jq silent-pass bug。

**对策**：
- 关键 P0 测试写完做 30 秒反向：改 expect → 必须 FAIL → 改回
- 反向测试结果存档（acceptance-evidence/<id>_NEGATIVE.txt 或 commit message 备注）

### D2. mock 失真

**定义**：mock 行为跟真实依赖不一致（mock 返 `None` 但真依赖 raise；mock 返列表但真依赖返 generator；mock 不验参数）。

**反例**：mock playwright `page.goto` 直接返 None，但真 patchright 在网络错时 raise `TimeoutError` → 测试不覆盖错误路径。

**对策**：
- mock 的接口跟真依赖签名严格对齐（用 `spec=`）
- mock 异常路径至少 1 case（`side_effect=Exception`）
- 关键依赖至少 1 个集成测试用真依赖 / 真 fixture

### D3. 测试 flaky / 依赖跑顺序 / 假阳性

**定义**：测试有时过有时不过（flaky）；测试 A 必须在 B 后跑（依赖顺序）；测试看似过实则没真断言（assertion 永远 truthy）。

**反例**：`assert response` 总是 truthy（response 对象非空就过），实际应 `assert response.status == 200`。

**对策**：
- 每个 assert 必须断言**具体值 / 具体字段**，不是「非空 / truthy」
- 测试用 fixture 隔离（每个 test 独立 DB / 独立 tmp_path）
- pytest 跑 `--randomly-seed=...` 检测顺序依赖

### D4. 覆盖率盲区（行覆盖 vs 场景覆盖）

**定义**：行覆盖率 / 分支覆盖率高（90%+）≠ 真覆盖。测试可能全是 getter / setter / 简单 happy path；关键边界 / 错误分支没覆盖。

**反例**：`workers.py` 行覆盖 95%，但「PDF download 收到 HTML」分支从来没测过 → v1.4 还是踩了。

**对策**：
- 评估测试用「场景覆盖」清单（本文档 20 项有几项有测试）而非纯行覆盖率
- 每个 P0 至少 1 个端到端真路径测试（不全 mock）
- 测试用 `# Arrange / Act / Assert` 三段式，让测的"场景"清晰可数

### D5. secret 泄漏 / PII

**定义**：fixture 文件里有真 token / 真账号；log 输出含 password / API key；commit message 含 .env 内容；error stack 直接 expose 给用户。

**反例**：`tests/fixtures/login_response.json` 含真 cookie 值；`logger.info(f"login with {password}")`；`commit -m "fix login with key=sk-xxx"`。

**对策**：
- fixture 入仓前用脚本脱敏（`tools/redact_fixtures.py`）
- log 必须 mask 凭证（`***` / 头尾留 4 字符）
- pre-commit hook 跑 `detect-secrets` / `gitleaks`
- 错误信息分两层：用户层（脱敏）/ 内部 log 层（详）

---

## 自检清单格式（implementer / test-engineer 提交时）

写完代码 / 测试后，在 commit message 或 phase report 里贴：

```text
20 项测试纪律自检：

已覆盖：
  A1 真实数据 ✓ (fixtures/news_list_7203.html 真采样)
  A4 重复幂等 ✓ (test_orphan_attachment)
  C1 层间对账 ✓ (test_parse_list_disclosure_url + test_fetch_detail_disclosure_url)
  C3 第三方变更 ✓ (test_download_pdf_rejects_html)
  D1 negative ✓ (commit message 末尾贴反向测试结果)

不适用（说明理由）：
  A3 时间时区 — 本函数不操作时间
  A5 注入 — 本函数不接用户输入
  B1 并发 — 单线程同步函数

未覆盖（KNOWN GAP）：
  B2 资源耗尽 — 长跑场景，留 Phase 8 load test
  D5 secret 泄漏 — 待加 pre-commit hook（Issue #X）

合计：5 ✓ / 3 N/A / 2 GAP / 10 已被其他测试间接覆盖
```

---

## 附录：本文档跟其他文档的关系

| 文档 | 解决什么 | 怎么用 |
|---|---|---|
| `acceptance-yaml-schema.md` | 验收怎么不被 LLM 自由解读 | requirements-analyst Phase 1 / integrator Phase 8 跑 |
| `testing-discipline.md`（本文）| 测试怎么不被 happy-path 自我证明 | implementer / test-engineer 写代码写测试时自检 |
| `coding-standards.md`（项目级）| 代码风格 / state 管理 / log 规范 | 全 phase 通用 |
| `CLAUDE.md`（项目级）| 5 条核心铁律（state / async / secrets / 异常 / 客户端） | 全 phase 通用 |
