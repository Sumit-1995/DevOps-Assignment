# AWS Infrastructure Provisioning Platform Architecture

## Overview

This platform enables Business Units (BUs) to provision AWS infrastructure through a standardized self-service model using:

- AWS Organizations
- Terraform Modules
- Terragrunt
- GitHub Actions
- AWS OIDC Federation
- AWS Secrets Manager
- Central Governance Controls

The platform is designed for:

- Multi-account AWS environments
- Shared-nothing isolation between Business Units
- Enterprise governance
- Self-service infrastructure provisioning
- Scalable onboarding of 50+ Business Units

------------------------------------------------------------------------

## High-Level Architecture

    +------------------------------------------------+
    |                  GitHub Enterprise             |
    +------------------------------------------------+
    |                                                |
    |  terraform-modules                             |
    |  platform-workflows                            |
    |  infrastructure-live                           |
    |                                                |
    +-------------------+----------------------------+
                        |
                        | OIDC Federation
                        v

    +------------------------------------------------+
    |                AWS Organization                |
    +------------------------------------------------+
    |                                                |
    | Shared Services Account                        |
    |  - Terraform State S3                          |
    |  - DynamoDB Lock Table                         |
    |  - Audit Logging                               |
    |  - Security Tooling                            |
    |                                                |
    +------------------------------------------------+

          Business Unit A
          ├── Dev Account
          ├── Staging Account
          └── Prod Account

          Business Unit B
          ├── Dev Account
          ├── Staging Account
          └── Prod Account

          Business Unit C
          ├── Dev Account
          ├── Staging Account
          └── Prod Account

------------------------------------------------------------------------

## Repository Strategy

Recommended approach: Hybrid Repository Model

### Platform Repositories

#### terraform-modules

Contains reusable Terraform modules.

Examples:

- VPC
- ECS
- Lambda
- RDS
- S3
- Security Hub
- GuardDuty

Ownership:

- Platform Engineering

### infrastructure-live

Contains environment-specific Terragrunt configuration.

Ownership:

- Platform Engineering
- DevOps Teams

### platform-workflows

Reusable GitHub Actions workflows.

Ownership:

- Platform Engineering

------------------------------------------------------------------------

## Terragrunt Hierarchy

    live/

    ├── _global
    │   ├── networking.hcl
    │   ├── security.hcl
    │   └── monitoring.hcl

    ├── bu-a
    │   ├── dev
    │   ├── staging
    │   └── prod

    ├── bu-b
    │   ├── dev
    │   ├── staging
    │   └── prod

Benefits:

- Maximum DRY
- Environment inheritance
- Central governance
- Easy scaling

------------------------------------------------------------------------

## Module Strategy

### Core Platform Modules

- VPC
- IAM Baseline
- CloudWatch
- Route53
- Security Hub
- GuardDuty
- Logging

### Application Modules

- ECS Service
- Lambda Function
- API Gateway
- RDS
- DynamoDB
- SQS
- SNS
- S3

### Design Principles

- Wrapper modules around community modules
- Centralized standards
- Semantic versioning
- Backward compatibility

------------------------------------------------------------------------

## State Management

Backend:

- S3 Remote State
- DynamoDB State Locking

State Isolation:

    state/
     ├── bu-a/dev/networking.tfstate
     ├── bu-a/dev/ecs.tfstate
     ├── bu-a/prod/networking.tfstate

Principles:

- One state per component
- One environment per account
- No state sharing across Business Units

------------------------------------------------------------------------

## Scalability Strategy

The architecture supports:

- 50+ Business Units
- Hundreds of AWS Accounts
- Thousands of Terraform Deployments

by leveraging:

- Terragrunt inheritance
- Reusable workflows
- Standardized modules
- AWS Organizations governance
