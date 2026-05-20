---
name: build-agent-project
description: "构建基于 Google ADK 的多 agent 项目。支持全新开发和基于已有项目继续开发。"
---

# Build Agent Project

构建或迭代基于 Google ADK 的生产级多 agent 项目。

## 启动流程

### Step 1: 询问开发模式

向用户提问：

> **请选择开发模式：**
> - **[A] 全新开发** — 从零开始构建一个新的 agent 项目
> - **[B] 继续开发** — 基于已有项目继续开发（追加需求 / 接手半成品）

### Step 2: 根据模式收集信息

**Mode A（全新开发）：**
1. 询问项目路径（默认 `~/projects/{name}/`）
2. **询问部署模式**（必问）：
   > 项目准备如何部署 / 运行？
   > - **[A] local-only-light** — 本地轻量原型（playground / 无 Docker / 无持久化 / 单用户）
   > - **[B] local-only-prod**  — 本地生产（Docker / SQLite 持久化 / start 脚本 / 备份）
   > - **[C] gcp-deploy**       — 云端（Agent Engine / Cloud Run / Terraform / CI/CD）
3. 询问需求文档（用户直接输入文本或提供文件路径）
4. 创建项目目录 + `.build/` 目录（**8 个 phase 文件夹**）+ `.status/` 目录
5. 起始 Phase 默认为 1

**Mode B（继续开发）：**
1. 询问项目路径（已有项目的根目录）
2. **不主动问部署模式** — 由 Phase 1 requirements-analyst 在普查中从现存项目推断；如推断不出再询问用户
3. 如果 `.status/` 存在，Read 最近 3-5 条记录，向用户展示项目状态摘要：
   > 「上次你在 {日期} 完成了 {内容}，建议接下来 {建议}」
4. 检查 `.build/` 是否存在：
   - **存在**：Read 已有的 phase 报告，展示当前进度
   - **不存在**：仅创建 `.build/` 空目录结构（**8 个空 phase 文件夹**），**不扫描代码、不补生成报告**（深度普查由 Phase 1 的 requirements-analyst 负责）
4. 询问本次需求（追加的需求 / 要改进的方面）
5. **必须从 Phase 1 开始**：requirements-analyst 进入后**第一动作是项目现状普查**，产出 `existing-project-snapshot.md`（12 节深度普查），与用户对账无误后才挖新需求。下游 phase 也都会读这份快照 + 各自对应的现存产物，保证增量改不冲突。**用户可以选择跳过哪些下游 phase，但 Phase 1 不可跳过**。

### Step 3: 询问结束阶段

> **本次开发计划执行到哪个阶段？**（1-8，默认 8 全部完成）

---

## 执行流程

按以下顺序，从用户指定的起始 Phase 到结束 Phase 依次执行。**每个 phase 内部都允许多轮交互**（详见下方"全程交互规则"）。

### Phase 1: 需求分析（requirements-analyst agent）

**主 Claude 启动 requirements-analyst 前必做**（memory feedback 注入）：

1. 尝试 Read `$HOME/.claude/projects/<encoded-build-agent-project-path>/memory/MEMORY.md` 和 `$HOME/.claude/projects/<encoded-target-project-path>/memory/MEMORY.md`
   - encoded path = 项目绝对路径，把 `/` 替换成 `-`，开头加 `-`（参考 Claude Code memory 路径规约）
   - 如：`/Users/sdl469/personal_projects/web_data_search` → `-Users-sdl469-personal-projects-web_data_search`
2. 从 MEMORY.md 中识别 type 为 `feedback` 的条目（按 frontmatter `type: feedback` 或文件名前缀 `feedback_` 判断）
3. 把这些 feedback 条目原文（含 trigger / 原因 / 应用场景）作为 sub-agent prompt 启动消息的一部分，标记 `<USER_FEEDBACK>...</USER_FEEDBACK>` 包裹
4. 如果两个 memory 文件都不存在（首次使用 / 用户没在 memory 里写 feedback），跳过这一步

这样 requirements-analyst 在 Phase 1 末尾能根据收集的需求 + 看到的 feedback 做分流。

**产出**：
- `{target}/.build/phase-1-requirements/requirements.md`（业务需求自然语言版）
- **`{target}/.build/phase-1-requirements/acceptance.yaml`**（P0 验收机械化定义；下游 runner 跑这个）
- **`{target}/.build/phase-1-requirements/applicable-feedback.md`**（用户 memory feedback 分流到下游 agent；详见 requirements-analyst.md §7）
- **Mode B 额外产出**：`{target}/.build/phase-1-requirements/existing-project-snapshot.md`（12 节项目现状普查，下游所有 phase 都会读）

⏸️ 暂停等待用户明确确认

### Phase 2: 系统设计（system-designer agent）
- 使用 `system-designer` agent
- 读取：Phase 1 产出
- 产出：`{target}/.build/phase-2-design/build-plan.md`
- ⏸️ 暂停等待用户明确确认

### Phase 3: 项目脚手架（scaffolder agent）
- 使用 `scaffolder` agent
- 读取：Phase 1 + Phase 2 产出
- 产出：项目骨架 + `{target}/.build/phase-3-scaffold/scaffold-report.md`
- **必产**：`{target}/scripts/run_acceptance.py`（acceptance.yaml runner，详见 `acceptance-yaml-schema.md`）+ `{target}/.build/phase-8-integration/acceptance-evidence/.gitkeep`（evidence 目录）+ `pyyaml` 加入 dev deps
- 门控：`uv sync` 成功 + `python scripts/run_acceptance.py --manual-only` 不报错（runner 自检）
- Mode A 时：`git init` + 首次 commit
- ⏸️ 暂停等待用户明确确认

### Phase 4: Agent 架构设计（agent-designer agent）
- 使用 `agent-designer` agent
- 读取：Phase 2 + Phase 3 产出
- 产出：agent 代码 + `{target}/.build/phase-4-agent-design/{agent-graph.md, tool-signatures.md}`
- 门控：`python -c "from app.agent import root_agent"` 成功
- ⏸️ 暂停等待用户明确确认

### Phase 5: 核心实现（implementer agent）
- 使用 `implementer` agent
- 读取：Phase 4 产出
- 产出：完整代码 + `{target}/.build/phase-5-implementation/impl-report.md`
- 门控：`make lint` 通过
- ⏸️ 暂停等待用户明确确认

### Phase 6: 部署准备（deployer agent）
- 使用 `deployer` agent
- 读取：Phase 1 (deployment_mode) + Phase 2 + Phase 3 产出
- 产出按 `deployment_mode` 分支：
  - `local-only-light` → `.env.example` + run 脚本 + README → `phase-6-deploy/local-light-report.md`
  - `local-only-prod`  → 上述 + Dockerfile + docker-compose + start/backup 脚本 → `phase-6-deploy/local-prod-report.md`
  - `gcp-deploy`       → Terraform + CI/CD workflows → `phase-6-deploy/cloud-report.md`
- 门控按 mode：
  - `local-only-light` → `.env.example` 不含真实密钥
  - `local-only-prod`  → `docker compose config` 通过
  - `gcp-deploy`       → `terraform validate` 通过
- ⏸️ 暂停等待用户明确确认

### Phase 7: 测试（test-engineer agent）
- 使用 `test-engineer` agent
- 读取：Phase 1 + Phase 4 + Phase 5 + Phase 6 产出（**Phase 1 含 requirements.md + acceptance.yaml**，**Phase 6 用于决定 load test 目标**）
- 产出：测试代码 + `{target}/.build/phase-7-testing/test-report.md`
- **必做**：跑一次 `python scripts/run_acceptance.py` 确认 acceptance.yaml 的 auto_checks 全 PASS（如有 manual_checks 需在 test-report 里说明哪些项要在 Phase 8 移交给用户）
- 门控：`make test` 通过 + `python scripts/run_acceptance.py` exit 0
- ⏸️ 暂停等待用户明确确认

### Phase 7.5: 对抗性独立审查（red-team-reviewer agent）
- 使用 `red-team-reviewer` agent
- 读取：Phase 1-7 全部产出 + 真实代码 + tests/fixtures
- 产出：`{target}/.build/phase-7-5-redteam-review/red-team-review.md`
- **核心纪律**：不写代码、不改代码、不修 bug — 只写 review 报告（列 bug + 复现 + 严重度 + 建议修法）
- 任务：
  1. 真跑 `python scripts/run_acceptance.py`（不信任 test-engineer 自陈）
  2. 静态语义对账（同一概念两处实现不一致 / docstring vs 行为 / API 文档 vs 响应）
  3. 动态对抗测试（真 fixture 端到端 / 边界输入 / 重复输入 / 第三方异常响应 / 冷启动 / silent failure）
  4. 反向测试 audit（抽样 5-10 条，故意改坏代码看测试能否抓出 — 抓不出的是死测试）
  5. testing-discipline 20 项 coverage audit
- 门控：
  - runner 真跑过
  - review 报告 ≥ 5 条对抗发现（含 critical / high / medium / low 任意）
  - 反向测试至少 5 条
  - 所有 bug 含复现命令 + 真 stdout
- ⏸️ 暂停等待用户决定：`通过` / `修后再 review` / `直接进 Phase 8`

### Phase 8: 集成验证（integrator agent）
- 使用 `integrator` agent
- 读取：Phase 1-7 全部产出（**Phase 1 含 acceptance.yaml — 验收依据是这个，不是 LLM 自由解读**）
- 产出：
  - 设计文档 + `{target}/.build/phase-8-integration/final-report.md`
  - **`{target}/.build/phase-8-integration/acceptance-evidence/`**（runner 真跑出的 stdout，git-track）
  - **`{target}/.build/phase-8-integration/manual_pending.md`**（一次性手工清单，交给用户做最后一轮验收）
- 门控：
  - `make lint && make test` 通过
  - **`python scripts/run_acceptance.py` exit 0**（auto_checks 全 PASS）
  - **final-report 里所有 PASS 判定必须引用 acceptance-evidence/ 中的具体文件**（不允许「机制实装完整」零证据陈述）
- ⏸️ 暂停等待用户最终确认 — **用户做完 manual_pending 后回来在 yaml 里把对应项 user_signed_off 改 true，再 `python scripts/run_acceptance.py --manual-only` 重新生成清单**

---

## 验收机械化（acceptance.yaml 机制）

> **背景（教训记录）**：早期版本由 integrator 在 Phase 8 凭 LLM 自由解读 §14 验收，自我打分「21/23 PASS」。外部独立 review 实际抓出 11 个真 bug（happy-path 测试通过 + mock 数据掩盖真实形态 + 自我打分循环）。本机制对症修复。

### 核心原则

- **验收必须可机器跑**：每条 P0 拆成原子 check（5 种 type 白名单：sql / api / python_call / file_check / bash），runner 自动跑、自动出证据、自动打分。
- **agent 不允许自我打分**：integrator 在 Phase 8 不再凭 LLM 解读 P0，必须跑 `scripts/run_acceptance.py` 拿 PASS/FAIL，引用 `acceptance-evidence/<check_id>.txt` 中的真实 stdout 作为证据。
- **真环境项必须显式列 manual**：做不了的（真账号 / 真 SMTP / 真双设备 / 1M 真数据）写 `manual_checks`，runner 自动汇总成 `manual_pending.md` 一次性清单交给用户做最后一轮。
- **反向测试是必做**：每条 auto_check 写完后 30 秒反向测一次（故意改坏 expect → 必须 FAIL → 改回）。防 silent-pass 死测试。

### 详细 schema 与例子

见 `.claude/skills/build-agent-project/acceptance-yaml-schema.md`。requirements-analyst / scaffolder / test-engineer / integrator 都必须先 Read 这份再开工。

### 配套：测试纪律（testing-discipline.md）

`.claude/skills/build-agent-project/testing-discipline.md` 是**测试**怎么不被 happy-path 自欺的姊妹文档（acceptance.yaml 是验收，testing-discipline 是测试 — 两层互补）：

- 4 大类 × 20 子项盲点清单（数据形态 / 行为 / 集成 / 测试自身的元）
- implementer / test-engineer 写代码 + 写测试时的自检清单
- 涵盖：fixture-first / contract test / negative test / encoding / 时间时区 / 并发 / 资源耗尽 / 状态机 / 第三方变更 / mock 失真 / secret 泄漏 等

implementer / test-engineer / integrator 都要先 Read。

### 各 phase 的职责分工

| Phase | Agent | 与 acceptance.yaml 的关系 |
|---|---|---|
| 1 | requirements-analyst | **创建** acceptance.yaml；每条 P0 必须有 ≥ 1 条 auto/manual check；空着不行 |
| 3 | scaffolder | **创建 runner**：`scripts/run_acceptance.py` + `acceptance-evidence/.gitkeep` + 加 pyyaml dev dep |
| 7 | test-engineer | **真跑 runner**：跑通所有 auto_checks；test-report 报告哪些项交给 Phase 8 manual |
| 7.5 | red-team-reviewer | **真跑 runner + 反向测试 + 静态对账 + 找 bug**：把以前需要外部 ChatGPT review 才抓到的 bug 内部抓出来；只写 review 报告不改代码 |
| 8 | integrator | **跑 runner + 写 evidence + 出 manual_pending + 处理 red-team bug**：final-report PASS 判定全部引用 evidence 文件，不允许零证据 |

### 用户视角

跑完 v1 / v1.1 / 任意迭代后：
1. 用户 cd 到项目，`python scripts/run_acceptance.py` 自己也能跑（验证 agent 没造假）
2. cat `manual_pending.md` 看自己要做什么
3. 做完一项 → 把 yaml 中那条 `user_signed_off: false` 改 `true` → `--manual-only` 重生成
4. 全 0 pending = full-PASS

---

## 全程交互规则（本项目核心设计原则）

**每个 phase 内部全程允许双向多轮交互**，不仅限于 phase 之间的暂停点。

### Agent → 用户主动提问

每个 agent 在执行过程中，遇到下列任一情况应**主动停下来问用户**（不强制，但鼓励）：
- 输入存在歧义或多种合理解读
- 关键技术 / 实现选型有 trade-off
- 涉及外部成本（云资源、付费 API、用户额度）
- 与 CLAUDE.md 铁律冲突的临时需求
- 上游 phase 产出与本阶段需求矛盾或遗漏

具体触发条件已写进每个 agent 的"多轮交互协议"段落，agent 自行判断。

### 用户 → Agent 随时插话

用户可在 agent 执行过程中**任意时刻**：
- 临时补充新约束（"必须用 us-central1"）
- 修正方向（"这里改成用 LoopAgent"）
- 提出疑问（"为什么选这个 service"）
- 要求 walkthrough 当前进度

agent 必须**停下当前动作**，重新评估，把新输入纳入后再继续。**不允许"等我做完再说"**。

### 双重确认机制

每个 agent 在交付时执行两层确认：

**子阶段定稿**：当 agent 自认某节产出（如 requirements.md 的某节、某个 sub-agent 实现、某个 Terraform 文件、某个测试套件）已成型时：
- 主动 walkthrough 该节内容要点
- 询问用户"还有要补充的吗 / 是否可定稿"
- **必须得到明确肯定回答**，才视为该节定稿

**整阶段结束**：所有子阶段定稿后：
- 完整 walkthrough 本 phase 全部产出
- 明确说明"我认为本阶段可以结束，请确认"
- **必须得到用户明确同意**，才能交棒下一 phase。不允许默认通过、超时通过、推断通过。

### 用户在暂停期间还可以

- 直接编辑 `.build/phase-N-{name}/` 中的文件
- 直接编辑项目代码文件
- 在对话中要求 agent 修改（可反复迭代，无次数限制）

### 门控失败时

agent 自动尝试修复 → 修复后再次执行双重确认流程，等用户确认。

---

## `.status/` 长期记忆

### 何时写入

| 触发时机 | 行为 |
|---------|------|
| 用户说「通过，下一步」 | 为当前 Phase 成果生成记录 |
| 用户说「停止」 | 立即生成记录，包含中断点和下一步建议 |
| Phase 内子阶段定稿 | 用户确认某轮迭代完成时生成（可选，按用户意愿） |
| 用户说「记录一下」 | 随时生成 |

### 文件格式

文件名：`{YYYY-MM-DD}_{HH-MM}_{简短描述}.md`

内容包含：
- 完成时间、所属功能、当前 Phase（1-8）、执行模式
- 本次完成的内容（详细）
- 关键变更（文件 + 变更类型 + 说明）
- 架构/设计参考（引用 `.build/` 中的图）
- 当前项目状态
- 下一步建议

### 新 Session 读取

Mode B 启动时，自动读取 `.status/` 最近 3-5 条记录，展示摘要。

---

## Git 管理

- **Mode A**：Phase 3 完成后 `git init` + 首次 commit
- **每个 Phase 通过后**：建议用户 commit（不强制）
- `.build/` 和 `.status/` 加入 `.gitignore`

---

## Mode B 特殊规则

- `.build/` 只存放与**本次新增功能**相关的产出，不混入已有功能信息
- 如果 `.build/` 不存在，仅创建空目录结构（8 个 phase 文件夹），不扫描代码、不补生成报告
- agent 需要了解现有代码上下文时，直接 Read 项目源码
- 用户可手动将参考文件放入对应 phase 文件夹
- **Phase 1 不可跳过**：任何新增功能都从需求分析开始

---

## 关键原则

1. **下游 agent 以 `.build/` 文件为唯一输入依据**，不依赖对话历史
2. **用户修改优先**：用户手动修改的文件，agent 必须以修改版为准
3. **双重确认强制**：子阶段定稿和整阶段结束都必须得到用户明确同意
4. **进度持久化**：所有进度保存在 `.build/` + `.status/` 中
5. Phase 8 的 integrator 发现问题时，可建议回退到特定 Phase 修复
6. **requirements.md 是最终验收依据**：test-engineer 用它推导 eval / load 目标，integrator 用它做最终对账

---

## 核心反失效机制（2026-05-05 反思后强化）

> 历史失败案例（来自 web_data_search v1）：
> - Phase 8 integrator 报告 "13 页面已实现 / B2 一致性 100%" → 实际 13 个 page 文件全是 9 行 TODO 占位
> - Pool A 端点找不到 stock 静默 `continue` 跳过 → 用户输入永远 "0 enqueued"
> - 142 个单元测试通过 → 但没一个模拟真实用户路径，端到端从入口到结果跑不通
> - "次日生效" / "v1 占位" 类语义没在 UI 显眼 → 用户期待立即生效结果发现没动静
>
> 共同模式：**「函数存在 / 测试通过 / lint 干净」≠「用户能跑通」**。

### 强制规则（所有 phase 必遵守）

#### 规则 1：「函数存在」≠「用户能用通」

任何 phase agent 报告「完成 X 功能」前必须证明：

> 用户从入口（浏览器 / CLI / API call）触发的真实路径，从入口到结果，**每一环都被实际跑过、每一个返回值都被实际看过**。

具体操作：
- API endpoint 改动 → `docker exec curl` 或 `httpie` 实际请求 + 贴响应 JSON
- UI 页面改动 → 抽样 1 个 page 文件读 30 行（不是 stub / 9 行 TODO 占位）+ `npm run build` 通过
- DB schema 改动 → 实际跑 migration + `SELECT *` 验证字段
- agent 实现 → 用 mock ToolContext 跑通 → 端到端 Runner 能产出非空 events

#### 规则 2：禁止盲信 sub-agent 自报

主调度（spawning agent 的角色）收到 sub-agent 完成报告时**不能直接信**：

- agent 报「N 个文件 / N 行代码 / 全过」时，**必须 `wc -l` 抽样 ≥ 3 文件**确认非占位
- agent 报「测试通过」时，**必须自己跑一遍** `make lint / make test` 不能光信
- agent 报告里出现 `placeholder / TODO / NotImplementedError / 占位` 字样**必须列入 KNOWN GAP** 显眼上报用户
- agent 用「应该 / 大概 / 推测 / 默认能 work / 理论上」类语言**必须实际验证再用**

特别提醒 integrator（Phase 8）：
- B2 build-plan ↔ impl 一致性检查不能光看 impl-report 自陈
- 必须**实际抽样阅读**关键代码文件（每个核心模块 ≥ 1 个文件读 30 行）
- 必须**实际执行**最关键的用户路径（不是 unit test，是真实启动 + 调 API）

#### 规则 3：静默失败禁令（贯穿 implementer / test-engineer）

implementer 写代码时：

1. **禁止 `if not found: continue` / `pass`** —— 改成至少之一：
   - `raise HTTPException(...)` / `return error`
   - 在 response 里返回 `skipped[] / failed[] / not_found[]` 列表
   - `log.warning(...)` + UI 显眼 banner
2. **placeholder / NotImplementedError 必须满足**：
   - 用户路径上的 placeholder **必须在 UI 显眼提示**「此功能 v1 占位，详情见 X」
   - 后端 placeholder 端点返回 501 + 明确 detail 引导替代方案
   - 不能只写注释 `# TODO: Phase X` 让用户自己发现
3. **API 返回空集合**（`enqueued: 0` / `items: []`）**必须配 explanation 字段**让 UI 解释原因
4. **所有 `try / except` 吞异常**必须 log 完整 traceback + response 含 `partial_failure: true`

test-engineer 写测试时：

1. **每个 P0 验收标准必须有一个端到端测试**（不是 unit test）：从用户输入到最终输出整条路径
2. **测试不能只检查 200 OK**，必须检查 response 内容含正确字段 / 数量
3. **静默失败检测测试**：故意输入未知 stock_code / 不存在的 ID / 空数据，验证返回的不是 silent success（应该是 explicit skipped 列表或 error）

#### 规则 4：requirements.md 的 UI 功能必须**抽样验证真实实现**

特别针对包含 UI / 前端 / 控制台的项目：

implementer 在 Phase 5 完工前：
- Read `ui/src/pages/*.tsx`（或对应 UI 入口）至少 3 个文件
- 确认每个文件 ≥ 30 行 + 含真实业务逻辑（import API hooks / 含 useState / 含真实组件树）
- **不能仅靠路由文件存在 / Sidebar 菜单项有标签 就报 "页面已实现"**
- `npm run build` / `pnpm build` 必须通过

test-engineer 在 Phase 7：
- **必须有 UI 测试覆盖**（最起码 Vitest component 或 Playwright smoke）
- test-report 应单列 UI 测试数量
- 没 UI 测试就明确写 KNOWN GAP

integrator 在 Phase 8 B2 一致性检查：
- 必须显式核对 build-plan 中提到的每个 UI 文件 / 路由 / 组件**确实存在且非占位**
- B2 报告不允许 "100% 一致" 这类总结，必须列出具体核对的文件 + 抽样行数

#### 规则 5：用户报「不能用」永远当真

任何 phase agent 收到用户反馈「跑不起来 / 没动静 / 报错」时：
- **不假定用户操作错**
- 立即查 docker logs / DB state / UI bundle 的实际状态
- 不能只说"应该是 X 原因"——必须实际查证再回答
- 修复前后必须各跑一次端到端 smoke 验证

#### 规则 6：同类问题一次性 audit，禁止 patch-and-fail loop（2026-05-19 追加）

> 历史教训：某 PR 在 CI 上连续撞 3 个同类问题 (env var 在 record-time / replay-time 不对齐)，每次只修被点名的那一个、push、等 CI、撞下一个、再修。3 round 后用户截断："你在看看还有没有别的会影响的环境变量一次修改了、不要这样一遍遍的 debug 了"。共同 root cause 是「`.env` 加载差异」一整类问题，第一次撞时就该 audit 全部同类。

任何 phase agent 发现 bug root cause 属于"一类问题" (e.g. "所有 env var 在 X 环境读不到"、"所有 sub-agent 都漏某个 callback"、"所有 API endpoint 都缺 schema 校验") 时：

- **不能只修被报错点名的那一个**，必须 grep 整个 codebase 找全部同类、一次性对齐
- 不修的同类也要**每个带一行理由**记录（防御下次又怀疑、做无用调查）
- audit 结果用一张 table 列出，"影响 / 不影响 + 理由"，一个 commit 修 N 处而不是 N 个 commit 各修一处
- 反模式：「他にも踩む可能性あるけど、撞いたら修す」(技术债 + reviewer 信頼 erosion + CI round count 爆发)

判断"是一类问题"的信号：
- 第二次同样 root cause 类型撞到时 → 一定 stop patch loop、改去 audit
- Bug 的最简描述里含"X 类的"、"X 全部的" → 强类信号

#### 规则 7：Debug instrumentation 必须 temporary 标签 + 一次性 cleanup（2026-05-19 追加）

> 历史教训：debug 阶段连续加 5 个 instrumentation commit (CI workflow debug step / cassette miss diagnostic 函数 180 行 / hit count map / framework event field print / SDK version print)。Root cause fix 落地后忘了 cleanup、merge 进 main。新来的 reviewer 读代码看到 180 行的诊断函数、读半天不知道它是 debug 残骸还是 production behavior、最终发现没人在用。每个 reviewer 重复这个 cognitive cost。

任何 phase agent 在 debug 中临时加 logging / print / diagnostic helper / CI workflow debug step 时：

- **加的时候**: commit message 用 `debug(scope): ...` 前缀 + inline comment 第一行写 `DEBUG INSTRUMENTATION (temporary)` 标签
- **Root cause fix 落地后**: 专用 cleanup commit (`chore(scope): remove debug instrumentation`)，按 grep 标签一次性删除全部 instrumentation。不要 commit-by-commit 渐进 unwind
- **不留**: commented-out 代码 / "万一以后还要用" 的 dead branch / `logger.debug(...)` 残骸
- **Sanity test 跟 debug instrumentation 严格区分生命周期**: sanity test 是永久 invariant 守护、debug instrumentation 是临时诊断、不混用。如果某条 instrumentation 真的有长期价值、重新分类成 sanity test、单独 commit + 配 doc 解释为什么 keep

Cleanup commit 的 net 行数 (deletions − insertions) 大于 0 是基本要求 —— 如果加了 200 行 instrumentation、cleanup commit 应该 -200 左右。

#### 规则 8：Agent 输出可信度强化机制（2026-05-20 追加）

> 历史教训：让 LLM 自由输出文本然后 parse / 让 LLM 一次性填整个 schema / 让 LLM 输出 URL 和 source 引用——这三种是 production agent 系统里最频繁的失稳源。LLM 一致性差、会幻觉 URL、会编造数据、字段间会互相影响 (fluent reasoning bias)。Production agent 必须在多 layer 锁住 LLM 输出。
>
> D&O Underwriting Agent 在这块下了重金沉淀、详细经验包: `/Users/sdl469/Documents/llm_output_stabilization_techniques.md`

implementer (Phase 5) 实装任何 LLM agent 时、对每个 LLM call 必检以下 7 项：

1. **Schema 强制**：LLM 输出要被下游程序消费时，必须用 Pydantic `output_schema` (Vertex / ADK) 或 tool function call、**不要让它返回自由文本然后正则 parse**。Validation 失败配 retry 2-3 次、不要无限。

2. **职责单一 (一个 LLM call 一个职责)**：不要让一个 LLM 一次填 50 个字段。拆成多 mini LLM call、每个一个小 schema、最后 Python merge。字段之间的 reasoning bias 会让多字段一次性填出问题。

3. **URL 永远 Python 算**：LLM 是 hallucination URL 重灾区。技术: 让 LLM 提 verbatim span (字面文本片段)、Python 在已知 source list (e.g. grounding segments / search results) 里 lookup resolve URL。**LLM 永远不应该输出 URL 字符串**。

4. **静态数据优先 (Tier 0 决定论)**：任何能 Python 算的字段（解析、计算、查表、enum 分类、日期算术）都不让 LLM 碰。LLM 留给真正的"语义合成"。Skeleton agent + overlay callback 模式：Python 算骨架、LLM 只填 narrative、merge 顺序 Python 控制。

5. **不确定性 surface**：每个 LLM 产 fact 字段必须有 `confidence: Literal["high", "medium", "low"]` + `sources: list[SourceLink]` 槽位、允许 LLM 标"不确定"。否则 LLM 默认 confident、会硬编。

6. **多源 cross-check (关键字段)**：业务 critical 字段从两个独立 source 各提一次（e.g. EDINET prose + Deep Research）、第三个 LLM (judge) 读两个 summary、一致选其一 / 矛盾标 `discrepancy_note`。不要 squash 矛盾、要 surface 给 reviewer。

7. **迭代修正配 max_iterations**：复杂任务 (validate / refine / gap-fill) 用 LoopAgent (critic + refiner) 但 **必须有 max_iterations 上限**（推荐 3）+ "no-progress exit" (本轮没改善就 stop)、避免无限循环。

test-engineer (Phase 7) 测试 LLM agent 时：

- LLM call 必须用 cassette (record-replay) 或 mock、CI 不打真 API
- 测试 `confidence=low` / `sources=[]` 的 edge case (LLM 标"不确定"时下游怎么处理)
- 测试 `discrepancy_note` 非空的 case (双源矛盾时报告怎么 render)
- 测试 LoopAgent 达 max_iterations 时的退出路径 (不应该是 fatal error)

system-designer (Phase 2) 设计 agent 拓扑时、对每个字段评分 Tier：
- **Tier 0**: 纯 Python parser / formula / lookup → 不进 LLM
- **Tier 1**: LLM + schema 约束 → 单 LLM call
- **Tier 2**: 多源 + judge → 关键字段
- **Tier 3**: 迭代修正 loop → 一次跑不好的复杂任务
- 同一 schema 不同字段可以不同 tier、不要全字段一刀切

### 项目内 protocols.md（推荐机制）

每个 build-agent-project 项目应在 `.build/protocols.md` 写明：
- 项目特定的 acceptance criteria 模板
- 端到端 smoke checklist（关键用户路径列表）
- 用户反制工具箱（"Show me" / "按协议 A 走" / "你确定吗" 等口令）

参考实例：`web_data_search/.build/protocols.md`（首例）。

未来项目在 Phase 1 完工时由 requirements-analyst 一并产出此文件骨架，Phase 5/7/8 都基于它做实证验证。
