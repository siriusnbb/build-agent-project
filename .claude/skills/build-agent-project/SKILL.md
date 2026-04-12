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
2. 询问需求文档（用户直接输入文本或提供文件路径）
3. 创建项目目录 + `.build/` 目录（7 个 phase 文件夹）+ `.status/` 目录
4. 起始 Phase 默认为 1

**Mode B（继续开发）：**
1. 询问项目路径（已有项目的根目录）
2. 如果 `.status/` 存在，Read 最近 3-5 条记录，向用户展示项目状态摘要：
   > 「上次你在 {日期} 完成了 {内容}，建议接下来 {建议}」
3. 检查 `.build/` 是否存在：
   - **存在**：Read 已有的 phase 报告，展示当前进度
   - **不存在**：仅创建 `.build/` 空目录结构（7 个空 phase 文件夹），**不扫描代码、不补生成报告**
4. 询问本次需求（追加的需求 / 要改进的方面）
5. 询问从哪个 Phase 开始（结合 `.status/` 建议供参考，用户自行决定）

### Step 3: 询问结束阶段

> **本次开发计划执行到哪个阶段？**（1-7，默认 7 全部完成）

---

## 执行流程

按以下顺序，从用户指定的起始 Phase 到结束 Phase 依次执行：

### Phase 1: 需求分析（planner agent）
- 使用 `planner` agent
- 产出：`{target}/.build/phase-1-planning/build-plan.md`
- ⏸️ 暂停等待用户确认

### Phase 2: 项目脚手架（scaffolder agent）
- 使用 `scaffolder` agent
- 读取：Phase 1 产出
- 产出：项目骨架 + `{target}/.build/phase-2-scaffold/scaffold-report.md`
- 门控：`uv sync` 成功
- Mode A 时：`git init` + 首次 commit
- ⏸️ 暂停等待用户确认

### Phase 3: Agent 架构设计（agent-designer agent）
- 使用 `agent-designer` agent
- 读取：Phase 1 + Phase 2 产出
- 产出：agent 代码 + `{target}/.build/phase-3-agent-design/{agent-graph.md, tool-signatures.md}`
- 门控：`python -c "from app.agent import root_agent"` 成功
- ⏸️ 暂停等待用户确认

### Phase 4: 核心实现（implementer agent）
- 使用 `implementer` agent
- 读取：Phase 3 产出
- 产出：完整代码 + `{target}/.build/phase-4-implementation/impl-report.md`
- 门控：`make lint` 通过
- ⏸️ 暂停等待用户确认

### Phase 5: 基础设施（infra-engineer agent）
- 使用 `infra-engineer` agent
- 读取：Phase 1 + Phase 2 产出
- 产出：Terraform + CI/CD + `{target}/.build/phase-5-infra/infra-report.md`
- 门控：`terraform validate` 通过
- ⏸️ 暂停等待用户确认

### Phase 6: 测试（test-engineer agent）
- 使用 `test-engineer` agent
- 读取：Phase 3 + Phase 4 产出
- 产出：测试代码 + `{target}/.build/phase-6-testing/test-report.md`
- 门控：`make test` 通过
- ⏸️ 暂停等待用户确认

### Phase 7: 集成验证（integrator agent）
- 使用 `integrator` agent
- 读取：Phase 1-6 全部产出
- 产出：设计文档 + `{target}/.build/phase-7-integration/final-report.md`
- 门控：`make lint && make test` 通过
- ⏸️ 暂停等待用户最终确认

---

## 暂停和恢复规则

### 每个 Phase 结束后

1. 通知用户 Phase 完成，指示审阅 `.build/phase-N/` 中的产出
2. **强制暂停**，等待用户明确指令：
   - 「通过」/「下一步」→ 进入下一 Phase
   - 「这里要改...」→ 当前 agent 读取用户修改，继续迭代
   - 「停止」/「今天到这里」→ 保存进度，生成 `.status/` 记录

### 用户可在暂停期间

- 直接编辑 `.build/phase-N/` 中的文件
- 直接编辑项目代码文件
- 在对话中要求 agent 修改（可反复迭代，无次数限制）

### 门控失败时

agent 自动尝试修复 → 修复后再次暂停等待用户确认

---

## `.status/` 长期记忆

### 何时写入

| 触发时机 | 行为 |
|---------|------|
| 用户说「通过，下一步」 | 为当前 Phase 成果生成记录 |
| 用户说「停止」 | 立即生成记录，包含中断点和下一步建议 |
| Phase 内小目标完成 | 用户确认某轮迭代完成时生成 |
| 用户说「记录一下」 | 随时生成 |

### 文件格式

文件名：`{YYYY-MM-DD}_{HH-MM}_{简短描述}.md`

内容包含：
- 完成时间、所属功能、当前 Phase、执行模式
- 本次完成的内容（详细）
- 关键变更（文件 + 变更类型 + 说明）
- 架构/设计参考（引用 `.build/` 中的图）
- 当前项目状态
- 下一步建议

### 新 Session 读取

Mode B 启动时，自动读取 `.status/` 最近 3-5 条记录，展示摘要。

---

## Git 管理

- **Mode A**：Phase 2 完成后 `git init` + 首次 commit
- **每个 Phase 通过后**：建议用户 commit（不强制）
- `.build/` 和 `.status/` 加入 `.gitignore`

---

## Mode B 特殊规则

- `.build/` 只存放与**本次新增功能**相关的产出，不混入已有功能信息
- 如果 `.build/` 不存在，仅创建空目录结构，不扫描代码、不补生成报告
- agent 需要了解现有代码上下文时，直接 Read 项目源码
- 用户可手动将参考文件放入对应 phase 文件夹

---

## 关键原则

1. **下游 agent 以 `.build/` 文件为唯一输入依据**，不依赖对话历史
2. **用户修改优先**：用户手动修改的文件，agent 必须以修改版为准
3. **每个 Phase 至少暂停一次**，即使门控自动通过也必须等用户确认
4. **进度持久化**：所有进度保存在 `.build/` + `.status/` 中
5. Phase 7 的 integrator 发现问题时，可建议回退到特定 Phase 修复
