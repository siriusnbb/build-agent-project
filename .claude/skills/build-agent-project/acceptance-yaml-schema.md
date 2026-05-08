# acceptance.yaml — 验收机械化 schema 参考

> 这份文档是 requirements-analyst / scaffolder / test-engineer / integrator 写或跑 `acceptance.yaml` 的唯一来源。
> 不要改 schema，要改先和用户讨论。

---

## 为什么有这个机制

**问题**：自然语言验收标准让 agent 在 Phase 8 凭 LLM 自由解读「机制实装完整 PASS」自我打分。结果上线后用户/外部 reviewer 一查就发现「函数存在但不真过 / mock 测试不能反映真实数据 / 层间识别不一致」一堆 bug。

**对策**：把 P0 验收拆成**机器可执行的原子 check**，agent 必须真跑 + 真出 stdout，且做不到的项必须显式列入 manual 清单（不允许藏）。

---

## 文件位置

- `{project}/.build/phase-1-requirements/acceptance.yaml` — 验收定义（requirements-analyst 在 Phase 1 输出）
- `{project}/scripts/run_acceptance.py` — runner（scaffolder 在 Phase 3 生成；详见 §runner 段）
- `{project}/.build/phase-8-integration/acceptance-evidence/` — 跑 runner 后产出（git-track）
  - `<check_id>.txt` — 每条 check 的 command + stdout + 时间戳 + PASS/FAIL
  - `_summary.txt` — 全量打分汇总
- `{project}/.build/phase-8-integration/manual_pending.md` — 手工清单（runner 自动生成）

---

## 顶层结构

acceptance.yaml 是 list，每个 item 对应 requirements.md 中一条 P0：

```yaml
- id: "<P0 id, 如 14.1.3>"
  description: "<人类可读描述>"
  source_requirement: "<引用 requirements.md 章节路径>"
  auto_checks: [...]      # 机器跑（必）
  manual_checks: [...]    # 真环境用户手工（按需）
```

**铁律**：任何 P0 都必须有 ≥ 1 条 auto_check **或** ≥ 1 条 manual_check。空着不行。

如果一条 P0 完全无法验证（既不能机器跑也不能用户手测），那它就不该是 P0 — 回 Phase 1 找 requirements-analyst 跟用户对齐。

---

## auto_checks（机器跑）

### check type 白名单

仅以下 5 种 type 合法。runner 见到未知 type 会抛错（这就是不让 agent 自由发挥的核心）：

#### `sql`
跑 SQL 语句（项目 DB 用 DuckDB 时）：
```yaml
- id: "14.1.3.fts_index_exists"
  description: "FTS 索引可调用"
  type: sql
  query: "SELECT fts_main_news.match_bm25(id, 'test') FROM news WHERE 1=0"
  expect: no_error          # 或: {row_count: N} | {value: <expected>}
```

#### `api`
HTTP 请求 + jq 表达式断言 body：
```yaml
- id: "14.1.3.cjk_fallback"
  description: "CJK 关键词应回退 LIKE"
  type: api
  method: GET                # 或 POST/PUT/DELETE
  url: "/api/news?query=トヨタ&search_mode=auto"
  expect_status: 200
  expect_jq: '.search_mode | contains("like")'
```

`url` 相对路径会拼 `BASE_URL`（runner 默认 `http://127.0.0.1:8000`，可用环境变量覆盖）。

#### `python_call`
import 模块 + 调用函数 + 比 expect：
```yaml
- id: "14.1.3.is_pdf_url"
  description: "is_pdf_url 识别真实 disclosure URL"
  type: python_call
  module: "app.adapters.kabutan"
  func: "is_pdf_url"
  arg: "https://kabutan.jp/disclosures/pdf/x/"   # 或: args: [...]（多参数）
  expect: true
```

#### `file_check`
文件系统检查：
```yaml
- id: "14.2.1.compose_volume"
  description: "compose 把 ./browser_context 挂到容器"
  type: file_check
  path: "docker-compose.yml"
  expect:
    contains: "./browser_context:/app/browser_context"
    # 或: exists           （只检存在）
    # 或: {size_gt: 1024}   （文件大小）
```

#### `bash`
兜底 shell 命令（少用，能用上面 4 种就别用这个）：
```yaml
- id: "14.3.2.no_cloud_sdk"
  description: "pyproject.toml 不含云依赖"
  type: bash
  cmd: "! grep -E '(google-cloud|boto3|azure-)' pyproject.toml"
  expect_exit: 0
  expect_stdout_contains: "..."   # 可选
```

### 写 auto_check 的原则

1. **单命令可重跑** — 任何人（你、用户、ChatGPT）都能从干净环境一键复跑。不允许「先做 X 再做 Y 才能跑」
2. **断言明确** — 不要写 `expect: no_error` 兜底；优先写具体值/字段/jq 表达式
3. **只跟一件事** — 一条 check 验一件事，不要 OR 多个条件
4. **时间稳定** — 不依赖时间敏感数据（昨天的新闻数）、不依赖外网（kabutan 服务器）
5. **反向测试可行** — 写完后能简单地改坏代码让它 FAIL（详见 §反向测试）

---

## manual_checks（真环境，用户手工）

```yaml
manual_checks:
  - id: "14.1.3.real_kabutan_pdf"
    description: "真账号下从 disclosure URL 真下载 PDF"
    reason: "需真 kabutan 凭证 + Premium 订阅，runner 无凭证无法跑"
    how_to_verify: |
      1. ./scripts/start.sh
      2. /accounts 添加真账号
      3. /scraping 切 test_incremental
      4. /news 找一条 disclosure，详情页应能看到 PDF 附件
    user_signed_off: false        # 用户做完改 true
```

### 何时该列 manual

- 需要真凭证（真账号、真 SMTP、真 OAuth）
- 需要真硬件 / 多设备（双机数据同步）
- 需要真生产数据量（1M 行压测）
- 需要真用户互动（UI 视觉验证 / 用户体验流畅度）
- 需要真长时间（24h 稳定性）

### 何时不该列 manual

- 「难写 auto」≠「该列 manual」 — 想清楚是 auto 难还是真的没法 auto
- 偷懒别用 manual 当出口

### 用户签字流程

1. runner 跑完会自动生成 `manual_pending.md`，列出所有 `user_signed_off: false` 的项
2. 用户按 `how_to_verify` 步骤跑过且通过后，把 yaml 里那条改成 `user_signed_off: true`
3. 重跑 `python scripts/run_acceptance.py --manual-only` 重新生成清单（数字会减少）

---

## runner 行为

`scripts/run_acceptance.py`（scaffolder 在 Phase 3 创建）做这些事：

1. 读 acceptance.yaml
2. 跑每条 auto_check（按 type 分发到 5 个 handler）
3. 每条 stdout 写入 `acceptance-evidence/<check_id>.txt`
4. 写 `_summary.txt` 总览
5. 写/更新 `manual_pending.md`
6. exit 0（全过）/ exit 1（有 fail）

### 命令行

```bash
# 全量跑（要求服务在 BASE_URL，默认 http://127.0.0.1:8000）
BASE_URL=http://127.0.0.1:8000 python scripts/run_acceptance.py

# 只跑某条 check
BASE_URL=http://127.0.0.1:8000 python scripts/run_acceptance.py --check 14.1.3.cjk_fallback

# 只重新生成 manual_pending.md（不跑 auto_check，不需要 server）
python scripts/run_acceptance.py --manual-only
```

### 依赖

- PyYAML（dev dep；scaffolder 加）
- DuckDB（已是项目依赖）
- jq（系统二进制；runner 自检 + 报错；用户 `brew install jq` / `apt-get install jq`）
- Python stdlib（urllib / subprocess / importlib / pathlib）

---

## 反向测试（防死测试）

每条 auto_check 写完后做 30 秒反向测试，证明 check 是活的（不是永远 pass 的废测试）：

1. 故意把 yaml 的 `expect` 改成必然失败的值（如 `expect: false` 改成 `true`）
2. 重跑 runner，**check 必须报 FAIL**
3. 改回 yaml，再跑一次，恢复 PASS
4. 把这一来一回的结果作为开发笔记写进 `<check_id>_NEGATIVE.txt`（可选）

**反向测试 FAIL 的情况**：
- 改坏后仍 PASS → check 写错了，先修 check 再继续
- 改坏后报 FAIL，但理由跟你期望的不一样（如 mismatch 但 message 含糊） → check 可读性差，调整

历史教训：v1 的 smoke.sh 表面 15/15 pass，但 jq 表达式假阳性导致 false 永远被吞。反向测试是抓这种 silent-pass 的唯一方法。

---

## requirements.md vs acceptance.yaml 关系

- **requirements.md** 仍是 source of truth — 业务目标、用户故事、范围、假设、限制都在 md 里写
- **acceptance.yaml** 只是 P0 章节的**机器可验证补充**
- requirements.md 里 P0 列表必须每条对应 acceptance.yaml 里的一个 id；不允许 yaml 比 md 多或少（integrator 在 Phase 8 校对）

---

## 失败常见原因 + 修法

| 现象 | 原因 | 修 |
|---|---|---|
| `unknown type 'xxx'` | 写了不在白名单的 type | 用 5 种合法 type 之一；真不能表达就拆成几条 |
| `jq not installed` | 系统没装 jq | `brew install jq` / `apt-get install jq` |
| `manual_pending 不更新` | 改 yaml 后没重跑 | `python scripts/run_acceptance.py --manual-only` |
| auto_check 跑了但每次结果不一样 | check 不稳定 | 移除时间敏感 / 外网依赖 |
| FAIL 但 stdout 看起来正常 | jq 表达式写错了 | 用 `jq -e '<expr>' < body.json` 本地调通再写进 yaml |

---

## 例子（完整一条 P0）

参考 `web_data_search` 项目的 `.build/phase-1-requirements/acceptance.yaml`（5 条 demo P0）和 `.build/phase-8-integration/acceptance-mechanism-design.md`（决策记录）。
