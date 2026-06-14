# Security Model

## Security Objectives

-   Zero long-lived cloud credentials
-   Least privilege access
-   Full auditability
-   Strong account isolation
-   Enterprise governance

## Authentication Model

### GitHub Actions

Authentication uses:

    GitHub OIDC
          ↓
    AWS IAM Role
          ↓
    Temporary Credentials

Benefits:

-   No static secrets
-   Short-lived credentials
-   Automatic rotation

## IAM Role Design

Each account contains:

    GitHubDeployRole-Dev
    GitHubDeployRole-Staging
    GitHubDeployRole-Prod

Trust policies restrict:

-   Repository
-   Branch
-   Environment

Example:

    main branch → Production Role
    develop branch → Dev Role

## Authorization Model

Least Privilege Principles:

-   Environment-specific roles
-   Scoped Terraform permissions
-   Separation of duties

Platform Engineers:

-   Module management
-   Shared services

DevOps Engineers:

-   Environment deployments

Software Engineers:

-   Self-service provisioning only

## Secret Management

### GitHub

Store only:

-   OIDC configuration
-   Non-sensitive metadata

Do NOT store:

-   Database passwords
-   API keys
-   Certificates

### AWS Secrets Manager

Primary secret store.

Examples:

-   Database credentials
-   Application secrets
-   Third-party API keys

### Parameter Store

Used for:

-   Configuration values
-   Feature flags
-   Environment settings

## Secret Injection

Runtime:

    Application
          ↓
    IAM Role
          ↓
    Secrets Manager

CI/CD:

    GitHub Action
          ↓
    OIDC Role
          ↓
    Secrets Manager

## Governance Controls

Mandatory:

-   Security Hub
-   GuardDuty
-   CloudTrail
-   Config Rules
-   Centralized Logging

## Compliance Controls

-   MFA enforced
-   SCP guardrails
-   IAM Access Analyzer
-   Encrypted state storage
-   KMS encryption
-   Production approval workflows

## Auditability

All changes are traceable through:

-   Pull Requests
-   GitHub Actions logs
-   CloudTrail
-   Terraform state history
