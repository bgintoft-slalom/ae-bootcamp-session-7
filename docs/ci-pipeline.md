# CI Pipeline

The golden path provides a reusable GitHub Actions workflow for validating and
deploying the todo-service. A service team adds a small caller workflow that
selects the checks and deployment stages it needs; the implementation stays in
`.github/workflows/golden-path-ci.yml`.

## Pipeline Jobs

### `lint`

Installs Node.js dependencies and runs ESLint for the backend and frontend.
This catches syntax, style, and maintainability problems before code is merged.

### `test`

Runs the backend Jest suite with coverage enabled. Lines, branches, functions,
and statements must each meet the 80% threshold. The coverage summary is added
to the GitHub Actions job summary so reviewers can see the result without
opening an artifact.

### `security-scan`

Runs Checkov against `infra/` and fails on HIGH severity findings. This protects
the shared infrastructure path from introducing avoidable security or policy
violations. It runs when `run_terraform_plan` is enabled.

### `terraform-plan`

Uses the supplied AWS IAM role through GitHub OIDC, installs the requested
Terraform version, initializes the dev stack, and creates a saved plan. The
plan is summarized in the job output and uploaded as an artifact for the apply
job. It runs when `run_terraform_plan` is enabled.

### `docker-build`

Builds the backend and frontend Dockerfiles on pull requests after lint and
tests pass. This validates that both deployable images can be built without
requiring AWS credentials or pushing images.

### `terraform-apply`

Runs after `terraform-plan` when `run_terraform_apply` is enabled. It uses OIDC
credentials, initializes the S3 state backend, downloads the saved plan, and
applies it to the dev environment. It publishes the resulting load balancer URL
to the job summary.

### `build-and-push`

Runs after the infrastructure is applied when `build_and_push` is enabled. It
looks up the ECR repositories, builds and pushes backend and frontend images
with both commit and `latest` tags, then forces a new ECS deployment.

## Adopt the Pipeline

Create `.github/workflows/todo-service-ci.yml` in the service repository. This
is the minimum caller for pull-request validation and `main` deployment:

```yaml
name: Todo Service CI

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

For a separate central repository, replace the local `uses` path with the
reusable workflow's repository and ref.

## Required Checks

- **Lint** validates backend and frontend JavaScript with ESLint so basic code
  quality issues are caught early.
- **Test** runs Jest and enforces 80% coverage across all coverage categories,
  protecting behavior as the service changes.
- **Security scan** runs Checkov against the infrastructure and blocks HIGH
  severity policy findings.
- **Terraform plan** validates the dev infrastructure and produces the exact
  saved plan that a later apply job consumes.
- **Docker build** confirms both deployment images build successfully on pull
  requests before they can reach `main`.

The caller enables the Terraform-related checks with
`run_terraform_plan: true`. Apply and image publishing remain restricted to
pushes to `main` by the caller's expressions.

## Configure AWS OIDC

1. Create or identify an AWS IAM role trusted by GitHub's OIDC provider. Scope
   its trust policy to this repository and the required branch or environment.
2. Grant the role only the permissions needed to plan or deploy this service.
3. In the repository, open **Settings > Secrets and variables > Actions** and
   create an Actions repository secret named `AWS_ROLE_ARN`.
4. Set its value to the full IAM role ARN, for example:
   `arn:aws:iam::123456789012:role/github-actions-todo-service`.

The caller maps that repository secret to the reusable workflow's
`aws_role_arn` secret. The `terraform-plan`, `terraform-apply`, and
`build-and-push` jobs request `id-token: write` and pass the secret to
`aws-actions/configure-aws-credentials`, so no long-lived AWS access keys are
stored in GitHub.