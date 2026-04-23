# Project Completion Agent System

## Purpose

用于构建基于 Google ADK 的生产级多 agent 系统。通过 7 阶段流水线（需求分析 → 脚手架 → Agent 设计 → 实现 → 基础设施 → 测试 → 集成验证）从零构建或在现有项目上追加功能。

## 项目约定

- 栈: Python 3.10+ · uv · Google ADK · GCP（Vertex AI / GCS / Discovery Engine）
- 工具细节（Terraform / pytest / Pydantic v2 / ruff / hatchling）见各 knowledge 文件

## 核心铁律

- **State 是唯一数据通道**：修改 Agent 逻辑优先检查 `ctx.session.state`，不用全局变量存储用户/会话信息
- **异步用 asyncio，禁止 threading**
- **Secrets 不入 log**：API key / token / 用户 PII / 文件完整内容 不得出现在 log、错误消息、异常 trace、commit message 中（详细分级见 `knowledge/coding-standards.md` 敏感数据脱敏）

## 架构模式

Planner（Phase 1）根据需求从以下模式中选择最合适的拓扑，可组合使用。

| 模式 | ADK 类型 | 适用场景 | 复杂度 |
|------|---------|---------|--------|
| Single Agent | Agent | 单一任务、tool ≤ 5 | 低 |
| Sequential Pipeline | SequentialAgent | 固定顺序多步骤、无条件分支 | 中 |
| Deterministic Pipeline | BaseAgent 子类 | 需条件分支、跳步、resume-from-failure | 高 |
| Hierarchical | Agent → AgentTool(sub) | LLM 动态决定调用哪个 sub-agent | 中 |
| Loop Refinement | LoopAgent | 迭代改进直到质量达标 | 中 |
| Parallel Fan-out | ParallelAgent | 独立子任务可并行 | 中 |
| Hybrid | 上述组合 | 复杂业务流程 | 高 |

**选择指南**:
- 流程固定？→ Sequential / Deterministic Pipeline
- 需要断点续传？→ Deterministic Pipeline（BaseAgent + resume-from-failure）
- 需要质量迭代？→ LoopAgent
- 子任务独立？→ ParallelAgent
- 动态路由？→ Hierarchical（Agent + AgentTool）
- 单 LLM + 少量 tool 就够？→ Single Agent

实现层通用约定（目录结构、Tool 签名、StateKeys、AgentTool 单父约束）见 `knowledge/project-patterns.md`。

## 关键命令（目标项目的标准命令集）

- `make install` / `make playground` / `make test` / `make lint` / `make eval` / `make deploy`

## Knowledge 参考资料

Agent 在工作前应先读取相关 knowledge 文件：
- `knowledge/google-adk-guide.md` — ADK 框架用法
- `knowledge/gcp-deployment-guide.md` — GCP 部署全流程
- `knowledge/project-patterns.md` — 项目架构惯例
- `knowledge/coding-standards.md` — Agent 项目编码规范（架构、state、日志、测试、错误处理）

## 构建过程产出

- `.build/` — 各阶段设计文档和产出报告（在目标项目目录中）
- `.status/` — 跨 session 长期记忆（在目标项目目录中）
