# CI Pipeline Guide

This document explains how the reusable CI workflow works, why each job exists, how teams adopt it, and how to configure AWS OIDC access for Terraform planning.

## What the reusable workflow does

The reusable workflow is defined in [.github/workflows/golden-path-ci.yml](../.github/workflows/golden-path-ci.yml). It centralizes platform guardrails so service teams can consume one shared CI contract instead of rebuilding pipeline logic per repository.

### Jobs and purpose

1. lint
- Runs ESLint for backend and frontend workspaces.
- Prevents style and correctness regressions from merging.

2. test
- Runs backend Jest tests with coverage reporting.
- Appends coverage metrics to the GitHub job summary for fast review.

3. security-scan
- Runs Checkov against infrastructure code when Terraform planning is enabled.
- Fails on HIGH findings to block risky IaC changes.

4. terraform-plan
- Authenticates to AWS with OIDC, initializes Terraform state, and generates a plan artifact.
- Publishes a plan summary so reviewers can inspect intended infrastructure changes in PRs.

5. docker-build
- Builds backend and frontend images on pull requests.
- Catches Dockerfile and build-context issues before merge.

6. terraform-apply
- Applies the previously generated plan when apply is enabled.
- Emits deployment output summary, including service URL.

7. build-and-push
- Resolves ECR repository outputs, builds versioned plus latest images, pushes to ECR, then triggers ECS redeploy.
- Enables push-button delivery after infrastructure is in place.

## Minimum caller workflow for a new service team

Create a caller workflow like [.github/workflows/todo-service-ci.yml](../.github/workflows/todo-service-ci.yml):

```yaml
name: Service CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
      run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

Why this is the minimum:
- It enables PR validation and main-branch delivery from the same workflow entrypoint.
- It declares caller-level OIDC permission, which is required for reusable workflows to receive an identity token.
- It passes the AWS role through the reusable-workflow secret input contract.

## Required checks and why they are required

The required checks are implemented in the reusable workflow and should be enforced as branch protection checks.

1. lint
- Validates code quality consistency across both apps.
- Required to reduce review noise and catch obvious issues early.

2. test
- Validates application behavior and enforces Jest coverage expectations.
- Required to prevent regressions and maintain confidence in changes.

3. security-scan
- Validates IaC security posture with Checkov and fails on HIGH severity.
- Required to prevent insecure infrastructure from being promoted.

4. terraform-plan
- Validates Terraform configuration and exposes intended infrastructure deltas.
- Required to make infrastructure changes reviewable and auditable before apply.

## OIDC secret configuration for terraform-plan

The terraform-plan job uses aws-actions/configure-aws-credentials with the reusable workflow secret input aws_role_arn. Configure it as follows:

1. Create or identify an AWS IAM role trusted for GitHub OIDC.
- Trust policy should allow your repository (or org policy scope) to assume the role via token.actions.githubusercontent.com.
- Grant least-privilege permissions needed for Terraform plan in your target account.

2. Add repository secret in GitHub.
- Go to Settings -> Secrets and variables -> Actions.
- Add secret name AWS_ROLE_ARN.
- Set value to the full IAM role ARN, for example arn:aws:iam::123456789012:role/github-oidc-terraform.

3. Ensure caller workflow requests id-token permission.
- In caller permissions, include id-token: write.
- Without this, OIDC token minting fails even if the reusable workflow requests it.

4. Ensure caller maps the secret to reusable input.
- In the caller job, set:
  - secrets:
    - aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
- Do not reference secrets.AWS_ROLE_ARN directly inside the reusable workflow.

## File map

- Reusable workflow: [.github/workflows/golden-path-ci.yml](../.github/workflows/golden-path-ci.yml)
- Caller workflow: [.github/workflows/todo-service-ci.yml](../.github/workflows/todo-service-ci.yml)
