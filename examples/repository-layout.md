# Repository Layout Examples

## terraform-modules (Shared Module Library)

```
terraform-modules/
├── README.md
├── CHANGELOG.md
├── .github/
│   ├── workflows/
│   │   ├── module-test.yml          # Test + lint on PR
│   │   └── module-release.yml       # Tag + GitHub Release on merge to main
│   └── CODEOWNERS                   # Platform team owns all modules
│
├── modules/
│   ├── networking/
│   │   ├── vpc/
│   │   │   ├── main.tf              # VPC, subnets, IGW, NAT, route tables
│   │   │   ├── variables.tf         # Documented inputs with descriptions
│   │   │   ├── outputs.tf           # vpc_id, subnet_ids, etc.
│   │   │   └── README.md            # Usage example + required inputs
│   │   └── transit-gateway/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── outputs.tf
│   │
│   ├── security/
│   │   ├── guardduty/
│   │   │   ├── main.tf              # GuardDuty + findings export
│   │   │   └── ...
│   │   ├── security-hub/
│   │   ├── iam-baseline/
│   │   │   ├── main.tf              # IAM password policy, support role, etc.
│   │   │   └── ...
│   │   └── cloudtrail/
│   │
│   ├── compute/
│   │   ├── ecs-service/
│   │   │   ├── main.tf              # ECS task def + service + ALB target
│   │   │   ├── variables.tf         # image, cpu, memory, port, desired_count
│   │   │   ├── outputs.tf           # service_name, alb_dns
│   │   │   └── README.md
│   │   ├── ecs-cluster/
│   │   └── lambda/
│   │       ├── main.tf              # Lambda + IAM role + log group
│   │       ├── variables.tf
│   │       └── outputs.tf
│   │
│   ├── data/
│   │   ├── rds/
│   │   │   ├── main.tf              # RDS instance + subnet group + SG
│   │   │   ├── variables.tf         # engine, instance_class, multi_az
│   │   │   └── outputs.tf           # endpoint, port, db_name
│   │   ├── dynamodb/
│   │   └── s3/
│   │       ├── main.tf              # S3 + encryption + lifecycle + access logs
│   │       └── ...
│   │
│   ├── messaging/
│   │   ├── sqs/
│   │   ├── sns/
│   │   └── api-gateway/
│   │
│   └── observability/
│       ├── cloudwatch-alarms/
│       └── log-groups/
│
└── examples/
    ├── complete-vpc/                 # Full usage example (used in tests)
    ├── ecs-with-rds/
    └── lambda-with-api-gateway/
```

---

## gha-workflows (Reusable GitHub Actions)

```
gha-workflows/
├── README.md
│
├── .github/
│   └── workflows/
│       ├── terraform-plan.yml       # Called on PR: plan + post comment
│       ├── terraform-apply-dev.yml  # Called on merge: apply to dev
│       ├── terraform-apply-stg.yml  # Apply to staging (after dev)
│       ├── terraform-apply-prd.yml  # Apply to prod (manual approval gate)
│       ├── drift-detect.yml         # Scheduled: plan in read-only mode
│       ├── module-publish.yml       # Tag + release terraform-modules
│       ├── account-onboard.yml      # New BU provisioning
│       └── security-scan.yml        # tfsec, checkov, terraform validate
│
└── scripts/
    ├── post-plan-comment.sh         # Parse plan output → format PR comment
    ├── drift-report.sh              # Aggregate drift across all components
    └── emergency-access.sh          # Break-glass role request script
```

### How a BU repo calls a shared workflow

```yaml
# infra-bu-a/.github/workflows/deploy.yml
name: Deploy Infrastructure

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  plan:
    if: github.event_name == 'pull_request'
    uses: xyz-org/gha-workflows/.github/workflows/terraform-plan.yml@v2
    with:
      working_directory: live/dev/vpc
      environment: dev
    secrets: inherit

  apply-dev:
    if: github.event_name == 'push'
    uses: xyz-org/gha-workflows/.github/workflows/terraform-apply-dev.yml@v2
    with:
      working_directory: live/dev/vpc
    secrets: inherit
```

---

## service-catalog (Self-Service Templates)

```
service-catalog/
├── README.md                         # How to use the catalog
├── CONTRIBUTING.md                   # How to add a new blueprint
│
├── templates/
│   ├── ecs-service/
│   │   ├── README.md                 # What this creates + required inputs
│   │   ├── terragrunt.hcl.tpl        # Parameterized template
│   │   ├── variables.yaml            # Schema with defaults + validation
│   │   └── example-values.yaml       # Filled-in example for copy-paste
│   │
│   ├── lambda-api/
│   │   ├── README.md
│   │   ├── terragrunt.hcl.tpl
│   │   └── variables.yaml
│   │
│   ├── rds-postgres/
│   │   ├── README.md
│   │   ├── terragrunt.hcl.tpl
│   │   └── variables.yaml
│   │
│   ├── s3-data-bucket/
│   └── sqs-queue/
│
└── scripts/
    └── scaffold.sh                   # CLI: scaffold.sh ecs-service my-api
```

### Developer self-service example (ECS Service)

**Developer fills in `example-values.yaml`:**

```yaml
# service-catalog/templates/ecs-service/example-values.yaml
service_name: payment-processor
image:        "123456789.dkr.ecr.us-east-1.amazonaws.com/payment-processor:latest"
cpu:          512
memory:       1024
port:         8080
desired_count: 2
health_check_path: /health
environment_vars:
  DB_HOST: "${dependency.rds.outputs.endpoint}"
  LOG_LEVEL: "INFO"
```

**Then runs:**

```bash
# Scaffold script generates terragrunt.hcl from template
./scripts/scaffold.sh ecs-service payment-processor \
  --values example-values.yaml \
  --output live/dev/ecs-payment-processor/terragrunt.hcl
```

**Generated `terragrunt.hcl`:**

```hcl
include "root" { path = find_in_parent_folders() }

terraform {
  source = "git::https://github.com/xyz-org/terraform-modules.git//modules/compute/ecs-service?ref=v2.1.0"
}

dependency "vpc"     { config_path = "../vpc" }
dependency "cluster" { config_path = "../ecs-cluster" }

inputs = {
  service_name       = "payment-processor"
  cluster_id         = dependency.cluster.outputs.cluster_id
  vpc_id             = dependency.vpc.outputs.vpc_id
  subnet_ids         = dependency.vpc.outputs.private_subnets
  container_image    = "123456789.dkr.ecr.us-east-1.amazonaws.com/payment-processor:latest"
  cpu                = 512
  memory             = 1024
  container_port     = 8080
  desired_count      = 2
  health_check_path  = "/health"
  environment_vars   = {
    LOG_LEVEL = "INFO"
  }
}
```

Developer opens a PR with this file. CI runs plan. Platform team or BU lead reviews. Merge → auto-deploy to dev.

---

## AWS Authentication: OIDC Flow (Pseudocode)

```
FUNCTION authenticate_github_to_aws(repo, environment, account_id):

  # GitHub generates a short-lived OIDC token for the job
  oidc_token = github.get_id_token(
    audience = "sts.amazonaws.com"
  )

  # Token contains claims:
  # - sub: "repo:xyz-org/infra-bu-a:environment:dev"
  # - aud: "sts.amazonaws.com"

  # AWS STS validates the token against the OIDC provider
  credentials = aws.sts.assume_role_with_web_identity(
    role_arn           = "arn:aws:iam::{account_id}:role/{environment}-TerraformDeployRole",
    role_session_name  = "github-actions-{repo}-{run_id}",
    web_identity_token = oidc_token,
    duration_seconds   = 3600   # 1-hour max, expires automatically
  )

  # Trust policy on the IAM role only allows this exact repo + environment
  # No static credentials stored anywhere
  RETURN credentials

END FUNCTION

IAM Role Trust Policy (per account, per env):
{
  "Principal": { "Federated": "arn:aws:iam::{account_id}:oidc-provider/token.actions.githubusercontent.com" },
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub":
        "repo:xyz-org/infra-bu-a:environment:prod"   ← scoped to exact repo + env
    }
  }
}
```

---

## Drift Detection (Pseudocode)

```
SCHEDULE: every 6 hours (GitHub Actions cron)

FUNCTION detect_drift():
  components = list_all_terragrunt_components("platform-config/live/")

  drift_report = []

  FOR EACH component IN components:
    result = run("terragrunt plan -detailed-exitcode --no-color", component.path)

    IF result.exit_code == 2:   # Changes detected = drift
      drift_report.append({
        component: component.path,
        plan_output: result.stdout,
        detected_at: now()
      })
    ELSE IF result.exit_code == 1:  # Error
      alert_slack(f"Plan error in {component.path}")

  IF drift_report is NOT empty:
    post_to_slack(drift_report)
    create_github_issue(drift_report)
    update_security_hub_finding(drift_report)

  RETURN drift_report

REMEDIATION OPTIONS:
  Option A (preferred): Engineer opens PR to codify the manual change
  Option B (if unintentional): Engineer opens PR to revert the drift
  Option C (urgent): Apply terraform plan directly to restore desired state
```

---

## State Management Design

```
State bucket naming convention:
  xyz-tfstate-{bu_name}-{env_name}
  e.g. xyz-tfstate-bu-a-prod

State key structure (set automatically by root terragrunt.hcl):
  {component_path}/terraform.tfstate
  e.g. bu-a/prod/vpc/terraform.tfstate
       bu-a/prod/ecs-cluster/terraform.tfstate
       bu-a/prod/rds-main/terraform.tfstate

Benefits of per-component state:
  - Blast radius containment: VPC plan doesn't lock ECS state
  - Parallel applies across components
  - Independent rollback per component
  - Cleaner dependency graph via terragrunt dependency blocks

DynamoDB lock table:
  xyz-tfstate-lock-{bu_name}-{env_name}
  Primary key: LockID (string) = state key
  Auto-released after apply completes or on force-unlock
```
