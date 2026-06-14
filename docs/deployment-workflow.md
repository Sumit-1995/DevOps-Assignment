# Deployment Workflow

## Deployment Principles

- GitOps driven
- Pull-request based
- Immutable deployments
- Environment promotion
- Automated validation

------------------------------------------------------------------------

## Pull Request Workflow

### Step 1

Developer creates PR.

Example:

    feature/add-rds-cluster

### Step 2

GitHub Actions executes:

- terraform fmt
- terraform validate
- tfsec
- checkov
- terragrunt validate

### Step 3

Generate Terraform Plan

    terragrunt run-all plan

Plan output is attached to PR.

------------------------------------------------------------------------

## Review Process

### Dev

Approval Required:

- 1 reviewer

### Staging

Approval Required:

- Platform Team
- BU DevOps Team

### Production

Approval Required:

- Platform Team
- BU Owner
- Change Management Approval

------------------------------------------------------------------------

## Promotion Model

Recommended:

    Dev
     ↓
    Staging
     ↓
    Production

Promotion occurs through Git merge.

No direct changes between environments.

------------------------------------------------------------------------

## GitHub Actions Structure

### Reusable Workflow Repository

    platform-workflows/

    terraform-plan.yml
    terraform-apply.yml
    security-scan.yml
    drift-detection.yml

### Consumer Repository

    uses: company/platform-workflows/.github/workflows/terraform-plan.yml@v1

------------------------------------------------------------------------

## Deployment Authentication

GitHub Actions authenticates using:

- GitHub OIDC
- AWS IAM Roles

No long-lived AWS credentials are stored in GitHub.

------------------------------------------------------------------------

## Production Deployment Controls

Protected GitHub Environments:

- staging
- production

Features:

- Manual approval gates
- Restricted approvers
- Audit trail

------------------------------------------------------------------------

## New Business Unit Deployment Flow

1.  Create AWS accounts.
2.  Bootstrap baseline infrastructure.
3.  Create Terragrunt folders.
4.  Enable CI/CD workflows.
5.  Assign IAM roles.
6.  Deploy platform baseline.

Onboarding target:

\< 1 hour fully automated.
