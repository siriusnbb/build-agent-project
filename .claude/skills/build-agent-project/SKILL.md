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
- 使用 `requirements-analyst` agent
- 产出：`{target}/.build/phase-1-requirements/requirements.md`
- **Mode B 额外产出**：`{target}/.build/phase-1-requirements/existing-project-snapshot.md`（12 节项目现状普查，下游所有 phase 都会读）
- ⏸️ 暂停等待用户明确确认

### Phase 2: 系统设计（system-designer agent）
- 使用 `system-designer` agent
- 读取：Phase 1 产出
- 产出：`{target}/.build/phase-2-design/build-plan.md`
- ⏸️ 暂停等待用户明确确认

### Phase 3: 项目脚手架（scaffolder agent）
- 使用 `scaffolder` agent
- 读取：Phase 1 + Phase 2 产出
- 产出：项目骨架 + `{target}/.build/phase-3-scaffold/scaffold-report.md`
- 门控：`uv sync` 成功
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
- 读取：Phase 1 + Phase 4 + Phase 5 + Phase 6 产出（**Phase 1 用于验收标准映射**，**Phase 6 用于决定 load test 目标**）
- 产出：测试代码 + `{target}/.build/phase-7-testing/test-report.md`
- 门控：`make test` 通过
- ⏸️ 暂停等待用户明确确认

### Phase 8: 集成验证（integrator agent）
- 使用 `integrator` agent
- 读取：Phase 1-7 全部产出（**Phase 1 用于最终验收检查**）
- 产出：设计文档 + `{target}/.build/phase-8-integration/final-report.md`
- 门控：`make lint && make test` 通过
- ⏸️ 暂停等待用户最终确认

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
