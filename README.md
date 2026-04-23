# Project Completion Agent System

用于构建基于 Google ADK 的生产级多 agent 系统。通过 7 阶段流水线
（需求分析 → 脚手架 → Agent 设计 → 实现 → 基础设施 → 测试 → 集成验证）
从零构建或在现有项目上追加功能。

## 启动方式

1. `cd /Users/sdl469/projects/build-agent-project && claude`
2. 输入 `/build-agent-project` 或自然语言触发
3. 按提示选择 Mode A（全新开发）或 Mode B（继续开发）

## 设计文档与规范

- `CLAUDE.md` — 项目身份、核心铁律、架构模式、knowledge 索引
- `knowledge/` — 深度参考（ADK 用法、GCP 部署、项目模式、编码规范）
- `.claude/agents/` — 7 阶段 subagent 的 prompt 定义
