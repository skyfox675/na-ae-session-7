# CI Pipeline

This repository uses a reusable GitHub Actions workflow to provide a consistent validation and deployment path for Node.js services with Terraform infrastructure.

## Workflow Overview

`.github/workflows/golden-path-ci.yml` is reusable and runs through `workflow_call`. It accepts these inputs:

| Input | Type | Default | Purpose |
| --- | --- | --- | --- |
| `node_version` | string | `20` | Node.js version used by lint and test jobs |
| `terraform_version` | string | `1.7.0` | Terraform version used by infrastructure jobs |
| `run_terraform_plan` | boolean | `false` | Enables `security-scan` and `terraform-plan` |
| `run_terraform_apply` | boolean | `false` | Enables Terraform apply after a successful plan |
| `build_and_push` | boolean | `false` | Enables Docker image publishing and ECS deployment |

The workflow has repository-level read permissions and pull request write permission. Jobs that use AWS request `id-token: write` and use GitHub OIDC to assume the configured AWS role.

### Jobs

- **`lint`** checks the backend and frontend with ESLint. It catches syntax errors and code-quality violations before tests or images run.
- **`test`** installs workspace dependencies and runs the backend Jest suite with coverage. Jest enforces the backend's configured global coverage thresholds, and the job appends line, branch, function, and statement coverage to the workflow summary.
- **`security-scan`** runs Checkov against `infra/` and fails on high-severity findings. It runs only when `run_terraform_plan` is enabled.
- **`terraform-plan`** installs the requested Terraform version, initializes the dev stack, creates mock networking values when an apply is not requested, and produces a saved plan. The plan summary is added to the workflow summary and the plan is uploaded as an artifact. It runs only when `run_terraform_plan` is enabled.
- **`docker-build`** builds the backend and frontend Dockerfiles without pushing images or using AWS. It runs on pull requests after both `lint` and `test` succeed.
- **`terraform-apply`** downloads the saved plan and applies it to the dev stack, then publishes the load balancer URL in the workflow summary. It runs only when `run_terraform_apply` is enabled and after `terraform-plan`.
- **`build-and-push`** resolves the ECR repositories, builds and pushes immutable SHA tags and `latest` tags, and forces a new ECS deployment. It runs only when `build_and_push` is enabled and after `terraform-apply`.

## Adopting the Golden Path

A service team adds a caller workflow at `.github/workflows/todo-service-ci.yml`. The minimum caller validates pull requests and pushes to `main`, and enables the reusable workflow's plan checks:

```yaml
name: Todo Service CI

on:
  push:
    branches:
      - main
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
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

The caller must request `id-token: write` at its own top-level. The reusable workflow cannot grant that permission on behalf of the caller. Teams can also set `run_terraform_apply` and `build_and_push` to true for a protected deployment path, normally only for pushes to the deployment branch.

## Required Checks

The standard required checks are:

| Check | Validates | Why it is required |
| --- | --- | --- |
| `lint` | ESLint for backend and frontend workspaces | Prevents style, syntax, and static code-quality issues from reaching later stages |
| `test` | Backend Jest tests and configured coverage thresholds | Protects API behavior and prevents changes that reduce required test coverage |
| `security-scan` | Checkov findings in Terraform configuration | Detects infrastructure security problems before provisioning |
| `terraform-plan` | Terraform initialization and the dev-stack execution plan | Confirms the infrastructure configuration can be evaluated and makes intended changes reviewable |

On pull requests, `docker-build` additionally validates that both service Dockerfiles build successfully. It does not push images.

## AWS OIDC Secret Configuration

The Terraform jobs use the reusable workflow secret `aws_role_arn`:

```yaml
secrets:
  aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

Configure it as follows:

1. Create an AWS IAM role for GitHub Actions with the permissions needed to plan or apply this service's infrastructure.
2. Configure the role trust policy for GitHub's OIDC provider (`token.actions.githubusercontent.com`). Restrict the `sub` claim to the repository and the permitted branch or environment.
3. In the repository, open **Settings > Secrets and variables > Actions**.
4. Add a repository secret named `AWS_ROLE_ARN` containing the full IAM role ARN, for example `arn:aws:iam::123456789012:role/todo-service-github-actions`.
5. Keep the caller mapping exact: the caller exposes the uppercase repository secret as the reusable workflow's lowercase `aws_role_arn` secret.

The reusable workflow passes this value to `aws-actions/configure-aws-credentials`, which exchanges the GitHub OIDC token for short-lived AWS credentials. No long-lived AWS access key or secret key is stored in GitHub.
