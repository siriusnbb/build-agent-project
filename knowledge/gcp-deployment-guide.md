# GCP Deployment Reference Guide

本文档是基于 Google Cloud Platform 部署 ADK Agent 项目的完整参考。基于 uw-agents-dno 实际项目提炼。

---

## 1. Vertex AI Agent Engine 部署

### AgentEngineApp 封装

```python
# app/agent_engine_app.py
from google.adk.agents.vertex_ai_agent_engine import AdkApp

class AgentEngineApp(AdkApp):
    def set_up(self):
        import vertexai
        vertexai.init()
        # 初始化 telemetry、logging

    def register_operations(self):
        super().register_operations()
        return {"register_feedback": self.register_feedback}

    def register_feedback(self, feedback: dict) -> dict:
        # 自定义操作（如用户反馈记录）
        ...

agent_engine = AgentEngineApp(
    agent=root_agent,
    enable_telemetry=True,
)
```

### 部署命令（通过 deploy.py 模块）

```bash
uv run python -m app.app_utils.deploy \
    --project=PROJECT_ID \
    --location=asia-northeast1 \
    --display-name=my-agent \
    --source-packages=./app \
    --entrypoint-module=app.agent_engine_app \
    --entrypoint-object=agent_engine \
    --requirements-file=app/app_utils/.requirements.txt \
    --service-account=SA_EMAIL \
    --min-instances=1 \
    --max-instances=10 \
    --cpu=4 \
    --memory=8Gi \
    --container-concurrency=9 \
    --set-env-vars="KEY1=VAL1,KEY2=VAL2" \
    --set-secrets="ENV_VAR=SECRET_ID:latest"
```

### deploy.py 主要参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--project` | 自动检测 | GCP project ID |
| `--location` | asia-northeast1 | リージョン |
| `--display-name` | 项目名 | Agent Engine 表示名 |
| `--source-packages` | ./app | ソースパッケージ |
| `--entrypoint-module` | app.agent_engine_app | Python モジュールパス |
| `--entrypoint-object` | agent_engine | AgentEngineApp インスタンス名 |
| `--requirements-file` | app/app_utils/.requirements.txt | 依存ファイル |
| `--min-instances` | 1 | 最小インスタンス数 |
| `--max-instances` | 10 | 最大インスタンス数 |
| `--cpu` | 4 | CPU コア数 |
| `--memory` | 8Gi | メモリ |
| `--container-concurrency` | 9 | 同時リクエスト数 |

### 部署逻辑（create vs update）

```
1. 动态 import agent 模块
2. 从 register_operations() 生成 class_methods
3. 构建 AgentEngineConfig
4. 列出已有 agent → 检测是 create 还是 update
5. 创建或更新 Agent Engine
6. 写入 deployment_metadata.json
7. 输出 Console playground URL
```

---

## 2. Cloud Run 部署

### ADK CLI 部署

```bash
uv run adk deploy cloud_run \
    --project=PROJECT_ID \
    --region=REGION \
    --service_name=SERVICE_NAME \
    --with_ui \
    --session_service_uri=agentengine://AGENT_ENGINE_ID \
    --artifact_service_uri=gs://BUCKET_NAME \
    ./app \
    -- \
    --service-account=SA_EMAIL \
    --build-service-account=projects/PROJECT/serviceAccounts/BUILD_SA \
    --vpc-connector=VPC_CONNECTOR \
    --vpc-egress=all-traffic \
    --ingress=internal-and-cloud-load-balancing \
    --allow-unauthenticated \
    --set-env-vars="KEY=VALUE" \
    --set-secrets="ENV_VAR=SECRET_ID:VERSION"
```

### 关键配置

- `--with_ui` — 启用 Streamlit Web UI
- `--session_service_uri` — 与 Agent Engine 共享 session
- `--artifact_service_uri` — artifact 存储到 GCS
- VPC 配置确保安全网络访问

---

## 3. Artifact Registry

### IAM 配置

CI/CD runner SA 需要 `roles/artifactregistry.writer` 角色。

### 用途

- 存储 Docker 镜像（Cloud Run 部署时自动推送）
- Cloud Build 自动处理镜像构建和推送

---

## 4. Terraform 基础设施

### 目录结构

```
deployment/terraform/
├── providers.tf          # Provider 配置（google, github, random）
├── variables.tf          # 输入变量定义
├── locals.tf             # 本地变量（服务列表、项目 ID 映射）
├── apis.tf               # GCP API 启用
├── service_accounts.tf   # Service Account 创建
├── iam.tf                # IAM 角色绑定
├── wif.tf                # Workload Identity Federation
├── storage.tf            # GCS Bucket
├── service.tf            # Agent Engine 资源（google_vertex_ai_reasoning_engine）
├── telemetry.tf          # BigQuery + Cloud Logging + 遥测
├── github.tf             # GitHub Actions 变量和 Secret
├── sql/                  # SQL 模板
│   └── completions.sql
├── vars/
│   └── env.tfvars        # 变量值文件
└── dev/                  # Dev 环境独立配置
    ├── providers.tf
    ├── variables.tf
    ├── apis.tf
    ├── iam.tf
    ├── storage.tf
    ├── service.tf
    └── telemetry.tf
```

### Provider 配置模板

```hcl
terraform {
  required_version = ">= 1.0.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 7.13.0"
    }
    github = {
      source  = "integrations/github"
      version = "~> 6.5.0"
    }
  }
}
```

### 核心 Variables

```hcl
variable "project_name"        { default = "my-agent" }
variable "prod_project_id"     {}
variable "staging_project_id"  {}
variable "cicd_runner_project_id" {}
variable "region"              { default = "asia-northeast1" }
variable "repository_name"     {}
variable "repository_owner"    {}
```

### Agent Engine 资源定义（service.tf）

```hcl
resource "google_vertex_ai_reasoning_engine" "app" {
  project      = var.prod_project_id  # or staging
  location     = var.region
  display_name = var.project_name

  spec {
    agent_framework = "google-adk"
    deployment_spec {
      min_replica_count     = 1
      max_replica_count     = 10
      container_concurrency = 9
      dedicated_resources {
        machine_spec {
          cpu_count = 4
          memory    = "8Gi"
        }
      }
    }
    class_methods_spec { ... }
    source_code_spec {
      entrypoint_spec {
        module = "app.agent_engine_app"
        object = "agent_engine"
      }
      requirements_filename = "app/app_utils/.requirements.txt"
      python_version        = "3.12"
    }
  }

  lifecycle {
    ignore_changes = [spec[0].source_code_spec]  # CI/CD 管理
  }
}
```

### 环境分离模式

- **CICD 项目**：CI/CD runner SA、WIF pool、Artifact Registry
- **Staging 项目**：staging 用 Agent Engine + App SA
- **Production 项目**：prod 用 Agent Engine + App SA
- **Dev 项目**（独立 Terraform）：开发环境独立配置

---

## 5. IAM 与 Service Account

### Service Account 体系

| SA | 项目 | 用途 |
|----|------|------|
| `{name}-cb` | CICD | CI/CD pipeline 执行 |
| `{name}-app` | prod/staging 各一个 | Agent 运行时身份 |

### CICD Runner SA 角色

```
roles/storage.admin
roles/aiplatform.user
roles/discoveryengine.editor
roles/logging.logWriter
roles/cloudtrace.agent
roles/artifactregistry.writer
roles/cloudbuild.builds.builder
roles/iam.serviceAccountTokenCreator（自身）
roles/iam.serviceAccountUser（自身）
```

### Application SA 角色

```
roles/aiplatform.user
roles/discoveryengine.editor
roles/logging.logWriter
roles/cloudtrace.agent
roles/storage.admin
roles/serviceusage.serviceUsageConsumer
```

---

## 6. Workload Identity Federation (WIF)

### 架构

```
GitHub Actions → OIDC Token → WIF Pool → SA Impersonation → GCP Resources
```

### Terraform 配置

```hcl
# 1. Pool
resource "google_iam_workload_identity_pool" "github_pool" {
  workload_identity_pool_id = "${var.project_name}-pool"
  display_name              = "GitHub Actions Pool"
  project                   = var.cicd_runner_project_id
}

# 2. OIDC Provider
resource "google_iam_workload_identity_pool_provider" "github_provider" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github_pool.workload_identity_pool_id
  workload_identity_pool_provider_id = "${var.project_name}-oidc"

  oidc { issuer_uri = "https://token.actions.githubusercontent.com" }

  attribute_mapping = {
    "google.subject"             = "assertion.sub"
    "attribute.repository"       = "assertion.repository"
    "attribute.repository_owner" = "assertion.repository_owner"
  }

  attribute_condition = "attribute.repository == '${var.repository_owner}/${var.repository_name}'"
}

# 3. IAM Bindings
resource "google_service_account_iam_member" "github_oidc_access" {
  service_account_id = google_service_account.cicd_runner_sa.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${pool_name}/attribute.repository/${owner}/${repo}"
}
```

### GitHub Actions 使用

```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: "projects/${{ vars.GCP_PROJECT_NUMBER }}/locations/global/workloadIdentityPools/${{ secrets.WIF_POOL_ID }}/providers/${{ secrets.WIF_PROVIDER_ID }}"
    service_account: "${{ secrets.GCP_SERVICE_ACCOUNT }}"
    create_credentials_file: true
    project_id: "${{ vars.CICD_PROJECT_ID }}"
```

---

## 7. CI/CD (GitHub Actions)

### PR Checks（pr_checks.yaml）

```yaml
on:
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - uses: astral-sh/setup-uv@v6
      - run: uv sync --locked
      - run: uv run codespell

  unit-test:
    runs-on: ubuntu-2-core-static-ip
    environment: DEV
    permissions: { contents: read, id-token: write }
    steps:
      - # checkout, python, uv, GCP auth (WIF)
      - run: uv run pytest tests/unit

  integration-test:
    runs-on: ubuntu-2-core-static-ip
    environment: DEV
    permissions: { contents: read, id-token: write }
    steps:
      - # checkout, python, uv, GCP auth (WIF)
      - run: uv run pytest tests/integration
```

### Deploy to Dev（deploy-to-dev.yaml）

```yaml
on:
  push:
    branches: [develop]
    paths: ["dno-research-agent/**"]

jobs:
  deploy_and_test_dev:
    runs-on: ubuntu-2-core-static-ip
    environment: DEV
    steps:
      - # checkout, python, uv, GCP auth
      - name: Export requirements
        run: uv export --no-hashes --no-sources ... > app/app_utils/.requirements.txt
      - name: Deploy
        run: uv run python -m app.app_utils.deploy --project=... --set-env-vars=... --set-secrets=...
      - name: Load Test
        run: uv run locust -f tests/load_test/load_test.py --headless -t 30s -u 2 -r 0.5
      - name: Upload Results to GCS
        run: gsutil cp results.csv gs://BUCKET/load-test-results/
```

---

## 8. 环境变量与 Secret 管理

### .env 文件体系

```
.env              # Production（gitignored）
.env.dev          # Development（gitignored）
.env.example      # Public template（tracked）
.env.dev.example  # Dev template（tracked）
```

### Makefile 环境加载

```makefile
ENV ?=
ENV_FILE := $(if $(ENV),.env.$(ENV),.env)
ifneq (,$(wildcard $(ENV_FILE)))
    include $(ENV_FILE)
    export
endif
```

使用：`make deploy`（加载 .env）/ `make deploy ENV=dev`（加载 .env.dev）

### Secret Manager 集成

部署时通过 `--set-secrets` 参数注入：

```
--set-secrets="GE_OAUTH_CLIENT_SECRET=SECRET_ID:latest"
```

格式：`ENV_VAR_NAME=SECRET_ID[:VERSION]`

### 关键环境变量

| 变量 | 说明 |
|------|------|
| `GOOGLE_CLOUD_PROJECT` | GCP 项目 ID |
| `GOOGLE_CLOUD_LOCATION` | リージョン |
| `LOGS_BUCKET_NAME` | 日志 GCS Bucket |
| `STORAGE_MODE` | "local" or "gcs" |
| `GCS_BUCKET_NAME` | 数据存储 Bucket |
| `SERVICE_ACCOUNT` | App SA email |
| `AGENT_ENGINE_ID` | Agent Engine ID |
| `LOG_LEVEL` | 日志级别（默认 INFO） |

---

## 9. Telemetry 基础设施

### 组件

1. **BigQuery Dataset** — 遥测数据存储
2. **Cloud Logging Bucket** — 10 年保留、analytics enabled
3. **Log Sinks** — GenAI 遥测 + Feedback 两条
4. **External Table** — 指向 GCS completions 数据
5. **Linked Dataset** — Cloud Logging → BigQuery 链接
6. **View** — SQL 连接 Logging 和 GCS 数据

### OpenTelemetry 配置

```
OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true
GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY=true
```

依赖：`opentelemetry-instrumentation-google-genai`

---

## 10. Makefile 部署命令速查

```bash
make install              # 安装依赖
make playground           # 本地开发 UI
make deploy               # 部署到 Agent Engine（prod）
make deploy-dev           # 部署到 Agent Engine（dev）
make deploy-dev-ui        # 部署 Cloud Run UI
make setup-dev-env        # Terraform 初始化 dev 环境
make test                 # 运行全部测试
make lint                 # 代码检查
make eval                 # Agent 评估
make register-gemini-enterprise  # 注册到 Gemini Enterprise
```
