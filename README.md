# Build Agent Project

用于构建基于 Google ADK 的多 agent 系统。通过 **8 阶段流水线 + 8 个专职 sub-agent**，从零开发或在现有项目上追加功能，支持本地与云端三种部署模式。

## 8 个 Phase

| Phase | Agent | 职责 | 产出 |
|-------|-------|-----|------|
| 1 | requirements-analyst | 深挖需求 + Mode B 项目现状普查 | `phase-1-requirements/requirements.md`（+ Mode B 时 `existing-project-snapshot.md`）|
| 2 | system-designer | 架构选型与 agent 图谱 | `phase-2-design/build-plan.md` |
| 3 | scaffolder | 项目骨架与配置 | `phase-3-scaffold/scaffold-report.md` |
| 4 | agent-designer | Agent 拓扑与 tool stub | `phase-4-agent-design/{agent-graph.md, tool-signatures.md}` |
| 5 | implementer | 填充 tool 实现 | `phase-5-implementation/impl-report.md` |
| 6 | deployer | **部署准备**（按 mode 分支） | `phase-6-deploy/{local-light,local-prod,cloud}-report.md` |
| 7 | test-engineer | 单元 / 集成 / eval / 负载测试 | `phase-7-testing/test-report.md` |
| 8 | integrator | 全量验证 + 设计文档 | `phase-8-integration/final-report.md` |

每个 phase 都内置**多轮交互协议**：agent 主动提问、接受用户随时插话、子阶段定稿与整阶段结束的双重确认。

## 启动方式

```bash
cd /Users/sdl469/personal_projects/build-agent-project && claude
```

然后输入 `/build-agent-project` 触发 Skill。

## 选择维度

启动时会问两个独立维度：

**开发模式**：
- **Mode A**（全新开发）— 从零开始
- **Mode B**（继续开发）— 已有项目追加需求；Phase 1 会先做 12 节深度项目现状普查，下游 phase 都基于快照增量改

**部署模式**（仅 Mode A 主动问，Mode B 由普查推断）：
- `local-only-light` — 轻量原型（playground / 无 Docker / 无持久化）
- `local-only-prod`  — 本地生产（Docker / SQLite / start 脚本 / 备份）
- `gcp-deploy`        — 云端（Agent Engine / Cloud Run / Terraform / CI/CD）

每个 phase 的产出与门控都按 deployment_mode 自动分支。

## 项目结构

```
build-agent-project/
├── CLAUDE.md                            # 项目铁律 + 架构模式表 + knowledge 索引
├── README.md                            # 本文件
├── .claude/
│   ├── skills/
│   │   └── build-agent-project/
│   │       └── SKILL.md                 # Skill 入口（流程编排）
│   └── agents/                          # 8 个 sub-agent 的 prompt
│       ├── requirements-analyst.md
│       ├── system-designer.md
│       ├── scaffolder.md
│       ├── agent-designer.md
│       ├── implementer.md
│       ├── deployer.md
│       ├── test-engineer.md
│       └── integrator.md
└── knowledge/                           # 共享参考资料（agent 按需 Read）
    ├── adk/                             # 9 个 ADK 知识文件（curated guide + cheat sheet）
    │   ├── guide.md                     # 索引 + 决策表 + 主题反向索引
    │   ├── agents.md
    │   ├── tools.md
    │   ├── context-and-state.md
    │   ├── sessions-artifacts-memory.md
    │   ├── callbacks-and-events.md
    │   ├── runtime-and-deploy.md
    │   ├── models-and-streaming.md
    │   └── advanced.md
    ├── gcp-deployment-guide.md          # GCP 部署全流程（仅 cloud 模式用）
    ├── project-patterns.md              # 项目架构惯例
    └── coding-standards.md              # 编码规范、错误处理、日志、测试
```

## 目标项目运行时产出

被构建的目标项目里会生成两个目录：

- `.build/` — 8 个 phase 的产出报告（每次新功能开发时被 agent 写入）
- `.status/` — 跨 session 长期记忆（每个 phase 通过时写入一条记录）

两者都加入 `.gitignore`，不入版本控制。

## 设计原则

1. **下游 agent 以 `.build/` 文件为唯一输入依据**，不依赖对话历史
2. **用户修改优先**：用户手动改的文件 agent 必须以修改版为准
3. **双重确认强制**：子阶段定稿与整阶段结束都需要用户明确同意
4. **Phase 1 不可跳过**：任何新功能（Mode A 或 Mode B）都从需求分析开始
5. **deployment_mode 贯穿全流程**：从 Phase 1 写入 requirements，下游 phase 都按它分支决策
6. **Mode B 不冲突**：所有 phase 在 Mode B 下增量改不重写，关键修改前与用户确认

## 知识库同步策略

`knowledge/adk/` 是 ADK 的 curated 速查 + 索引，**不追求镜像 adk.dev 全部内容**。每个文件顶部标注 `Last synced` 日期和 upstream URL，agent 在写代码遇到细节时会主动 WebFetch 补查。

建议每季度同步一次，或在 ADK 出现主要版本变更时同步。
