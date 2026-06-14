# Terragrunt Layout — DRY Configuration Hierarchy

## Philosophy

Terragrunt provides a thin wrapper over Terraform enabling:
- A single `source` reference per module — no copy-paste
- Hierarchical variable inheritance (org → BU → env → component)
- Automatic remote state backend generation
- `dependency` blocks to wire outputs between components

---

## Folder Structure

```
platform-config/
├── terragrunt.hcl                     ← ROOT: org-wide defaults
├── _common/
│   ├── backend.hcl                    ← S3 + DynamoDB backend template
│   ├── provider.hcl                   ← AWS provider defaults
│   └── tags.hcl                       ← Mandatory tagging policy
│
├── bu-a/
│   ├── account.hcl                    ← BU-A account IDs + region
│   ├── dev/
│   │   ├── env.hcl                    ← Dev-specific vars (size, flags)
│   │   ├── vpc/
│   │   │   └── terragrunt.hcl         ← Points to vpc module v1.3.0
│   │   ├── ecs-cluster/
│   │   │   └── terragrunt.hcl
│   │   ├── rds-main/
│   │   │   └── terragrunt.hcl
│   │   └── s3-assets/
│   │       └── terragrunt.hcl
│   ├── staging/
│   │   ├── env.hcl                    ← Staging-specific vars
│   │   ├── vpc/
│   │   │   └── terragrunt.hcl         ← Same source, different vars
│   │   └── ...
│   └── prod/
│       ├── env.hcl                    ← Prod-specific vars (larger, HA)
│       ├── vpc/
│       │   └── terragrunt.hcl
│       └── ...
│
├── bu-b/
│   ├── account.hcl
│   └── dev/ staging/ prod/            ← Identical pattern
│
└── shared-services/
    ├── tgw/
    │   └── terragrunt.hcl
    ├── route53/
    │   └── terragrunt.hcl
    └── logging/
        └── terragrunt.hcl
```

---

## Root terragrunt.hcl (org-wide defaults)

```hcl
# platform-config/terragrunt.hcl
locals {
  # Parse the account.hcl and env.hcl from the calling directory tree
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  env_vars     = read_terragrunt_config(find_in_parent_folders("env.hcl"))

  aws_region   = local.account_vars.locals.aws_region
  account_id   = local.account_vars.locals.account_id
  bu_name      = local.account_vars.locals.bu_name
  env_name     = local.env_vars.locals.env_name
}

# Auto-generate S3 backend for every component — no copy-paste
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "xyz-tfstate-${local.bu_name}-${local.env_name}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = local.aws_region
    encrypt        = true
    kms_key_id     = "alias/terraform-state-key"
    dynamodb_table = "xyz-tfstate-lock-${local.bu_name}-${local.env_name}"
  }
}

# Auto-generate provider config
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<-EOF
    provider "aws" {
      region = "${local.aws_region}"
      assume_role {
        role_arn = "arn:aws:iam::${local.account_id}:role/TerraformDeployRole"
      }
      default_tags {
        tags = {
          ManagedBy   = "terraform"
          BU          = "${local.bu_name}"
          Environment = "${local.env_name}"
          Repository  = "platform-config"
        }
      }
    }
  EOF
}
```

---

## BU-A account.hcl

```hcl
# platform-config/bu-a/account.hcl
locals {
  bu_name    = "bu-a"
  aws_region = "us-east-1"

  # Per-environment account IDs
  account_ids = {
    dev     = "111111111111"
    staging = "222222222222"
    prod    = "333333333333"
  }

  # The account_id is inferred based on env_name at plan time
  account_id = local.account_ids[read_terragrunt_config(
    find_in_parent_folders("env.hcl")).locals.env_name]
}
```

---

## env.hcl — Dev example

```hcl
# platform-config/bu-a/dev/env.hcl
locals {
  env_name         = "dev"
  vpc_cidr         = "10.10.0.0/16"
  az_count         = 2            # Fewer AZs in dev to save cost
  enable_nat_gw    = true
  single_nat       = true         # Single NAT in dev (cost saving)
  rds_instance     = "db.t3.small"
  rds_multi_az     = false        # No Multi-AZ in dev
  ecs_desired      = 1
  deletion_protect = false        # Allow destroy in dev
}
```

---

## env.hcl — Prod example

```hcl
# platform-config/bu-a/prod/env.hcl
locals {
  env_name         = "prod"
  vpc_cidr         = "10.12.0.0/16"
  az_count         = 3            # Full AZ coverage
  enable_nat_gw    = true
  single_nat       = false        # One NAT per AZ for HA
  rds_instance     = "db.r6g.large"
  rds_multi_az     = true         # Multi-AZ in prod
  ecs_desired      = 3
  deletion_protect = true         # Prevent accidental destroy
}
```

---

## Component: VPC terragrunt.hcl

```hcl
# platform-config/bu-a/dev/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()   # Inherits backend + provider from root
}

locals {
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  env      = local.env_vars.locals
}

# Pin to a specific module version
terraform {
  source = "git::https://github.com/xyz-org/terraform-modules.git//modules/networking/vpc?ref=v1.3.2"
}

inputs = {
  vpc_cidr   = local.env.vpc_cidr
  az_count   = local.env.az_count
  single_nat = local.env.single_nat
  tags = {
    Component = "vpc"
  }
}
```

---

## Component with dependency: ECS Cluster

```hcl
# platform-config/bu-a/dev/ecs-cluster/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

locals {
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  env      = local.env_vars.locals
}

terraform {
  source = "git::https://github.com/xyz-org/terraform-modules.git//modules/compute/ecs-cluster?ref=v2.1.0"
}

# Wire VPC outputs into this component — no hardcoding
dependency "vpc" {
  config_path = "../vpc"
  mock_outputs = {
    vpc_id          = "vpc-mock"
    private_subnets = ["subnet-mock-a", "subnet-mock-b"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  vpc_id          = dependency.vpc.outputs.vpc_id
  subnet_ids      = dependency.vpc.outputs.private_subnets
  desired_count   = local.env.ecs_desired
}
```

---

## DRY Summary

| Level | File | What it controls |
|---|---|---|
| Org-wide | `terragrunt.hcl` (root) | Backend template, provider, tags |
| BU-wide | `account.hcl` | Account IDs, region, BU name |
| Env-wide | `env.hcl` | Sizing, flags, CIDR, feature toggles |
| Component | `terragrunt.hcl` | Module source + version + inputs |

Zero Terraform code is duplicated. Adding a new environment = create a new `env.hcl` + copy component `.hcl` files (which are <20 lines each).

---

## Module Versioning Strategy

```
Semantic versioning via Git tags:
  v1.0.0  — initial release
  v1.1.0  — backward-compatible feature add
  v1.1.1  — bug fix
  v2.0.0  — breaking change (input rename, output removal)

In terragrunt.hcl:
  source = "git::...?ref=v1.3.2"   ← pinned, reproducible

CHANGELOG.md in terraform-modules repo:
  ## v2.0.0 (BREAKING)
  - Renamed input `vpc_id` to `network_vpc_id`
  - Migration guide: s/vpc_id/network_vpc_id/ in all terragrunt.hcl

BU upgrade process:
  1. Platform team cuts v2.0.0 tag with migration guide
  2. BU opens PR updating ref=v2.0.0 + fixing renamed inputs
  3. Plan shows expected diff only (no surprises)
  4. BU merges at their own schedule (no forced upgrades)
```
