# Project Completion Agent System

## Purpose

用于构建基于 Google ADK 的生产级多 agent 系统。通过 7 阶段流水线（需求分析 → 脚手架 → Agent 设计 → 实现 → 基础设施 → 测试 → 集成验证）从零构建或在现有项目上追加功能。

## 使用方式

1. `cd /Users/sdl469/projects/build-agent-project && claude`
2. 输入 `/build-agent-project` 或自然语言触发
3. 按提示选择 Mode A（全新开发）或 Mode B（继续开发）

## 项目约定（目标项目应遵循）

- 语言: Python 3.10+，带类型注解
- 包管理器: uv（非 pip）
- 框架: Google ADK (google-adk)
- 云平台: GCP (Vertex AI Agent Engine, GCS, Discovery Engine)
- IaC: Terraform (deployment/terraform/)
- CI/CD: GitHub Actions (.github/workflows/)
- 测试: pytest (unit/integration), ADK eval, Locust load test
- Linting: ruff + ty + codespell
- 数据验证: Pydantic v2
- 构建系统: hatchling

## 架构模式

- Root Agent: LLM 编排器（Agent 类）
- Pipeline Agent: 确定性 BaseAgent 子类，含 resume-from-failure
- Sub-agents: 每个位于 app/sub_agents/{name}/，含 __init__.py、agent.py、prompt.py
- Tools: 带 ToolContext 参数的函数，用于访问 session state
- Session state: 集中式 StateKeys 类位于 config.py
- AgentTool: 将 sub-agent 包装为 tool（遵循 ADK 单父约束）

## 关键命令（目标项目的标准命令集）

- `make install` / `make playground` / `make test` / `make lint` / `make eval` / `make deploy`

## Knowledge 参考资料

Agent 在工作前应先读取相关 knowledge 文件：
- `knowledge/google-adk-guide.md` — ADK 框架用法
- `knowledge/gcp-deployment-guide.md` — GCP 部署全流程
- `knowledge/project-patterns.md` — 项目架构惯例

## 构建过程产出

- `.build/` — 各阶段设计文档和产出报告（在目标项目目录中）
- `.status/` — 跨 session 长期记忆（在目标项目目录中）
