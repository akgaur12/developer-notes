# Chapter 14 — CI/CD Integration & Atlantis

## Learning Objectives

By the end of this chapter you will be able to:

- Explain why automating Terraform through CI/CD is safer than running it manually
- Write a GitHub Actions workflow that plans on every PR and applies on merge
- Configure OIDC-based AWS authentication — no long-lived credentials stored in CI
- Deploy and configure Atlantis as a GitOps bot for Terraform
- Understand the Terraform Cloud run model as an alternative to self-hosted CI
- Set up scheduled drift detection to catch manual infrastructure changes
- Apply pre-commit hooks that enforce formatting and security scanning locally

---

## 14.1 Why Automate Terraform?

**Running Terraform manually creates hidden risks:**

- No record of who ran what, or when
- Depends on each engineer's local AWS credentials and Terraform version
- No team review of the plan before apply — one typo can delete production data
- No notifications when infrastructure changes
- "Works on my machine" — different provider plugin versions produce different behavior

**CI/CD for Terraform solves all of these:**

- Every change goes through Git and PR review — full audit trail
- Plan output is posted as a PR comment — reviewers see exactly what will change before approving
- Apply runs only after PR approval and merge to main
- Consistent tooling: same Terraform version, same provider pin, same environment variables
- Automatic notifications on success or failure

---

## 14.2 GitHub Actions for Terraform

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main]
    paths: ['infra/**']
  pull_request:
    branches: [main]
    paths: ['infra/**']

env:
  TF_VERSION: 1.9.0
  AWS_REGION: us-east-1

permissions:
  id-token: write       # for OIDC
  contents: read
  pull-requests: write  # to post plan as PR comment

jobs:
  terraform:
    name: Terraform Plan & Apply
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra/environments/prod

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-terraform-role
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        continue-on-error: false

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        continue-on-error: true

      - name: Post Plan to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 📋
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            *Pusher: @${{ github.actor }}, Working Directory: \`infra/environments/prod\`*`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

      - name: Terraform Plan Status
        if: steps.plan.outcome == 'failure'
        run: exit 1

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

Key design decisions in this workflow:

| Decision | Reason |
|---|---|
| `paths: ['infra/**']` | Skip the workflow on docs-only changes |
| `-out=tfplan` then `apply tfplan` | Apply uses the exact plan that was reviewed — no drift between plan and apply |
| `continue-on-error: true` on plan | Allows the plan output to be posted even when the plan fails |
| Apply only on `push` to `main` | PRs get plan-only; apply is gated behind merge |

---

## 14.3 IAM Role for GitHub Actions (OIDC)

Never store AWS access keys as GitHub secrets. Use OIDC: GitHub issues a short-lived JWT, and AWS exchanges it for temporary credentials. No secrets to rotate or leak.

```hcl
# Create OIDC provider for GitHub Actions
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

# Trust policy: only allow JWTs from your specific repo
data "aws_iam_policy_document" "github_terraform_assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:${var.github_org}/${var.github_repo}:*"]
    }
  }
}

resource "aws_iam_role" "github_terraform" {
  name               = "github-terraform-role"
  assume_role_policy = data.aws_iam_policy_document.github_terraform_assume_role.json
}

resource "aws_iam_role_policy_attachment" "github_terraform" {
  role       = aws_iam_role.github_terraform.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
  # In production: replace with a scoped policy listing only the services Terraform manages
}
```

The `sub` condition is the critical security control. Without it, any GitHub Actions workflow in any repo could assume your role. The `StringLike` pattern with `:*` allows both branch and tag JWT subjects from your repo.

---

## 14.4 Atlantis — GitOps for Terraform

Atlantis is a self-hosted service that listens to GitHub/GitLab webhooks and runs Terraform plan and apply in response to PR comments. It gives teams a controlled, auditable workflow without a full CI platform for Terraform.

**Workflow:**

```
1. Developer opens a PR modifying .tf files
2. Atlantis detects the PR via webhook and runs `terraform plan`
3. Plan output is posted as a PR comment
4. Reviewer approves the plan by commenting `atlantis apply`
5. Atlantis runs `terraform apply` and posts the result as a comment
6. PR is merged after successful apply
```

**Deploying Atlantis on ECS Fargate:**

```hcl
resource "aws_ecs_task_definition" "atlantis" {
  family                   = "atlantis"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 512
  memory                   = 1024
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.atlantis_task.arn

  container_definitions = jsonencode([{
    name  = "atlantis"
    image = "ghcr.io/runatlantis/atlantis:latest"
    portMappings = [{ containerPort = 4141 }]
    environment = [
      { name = "ATLANTIS_GH_USER",           value = var.github_user },
      { name = "ATLANTIS_GH_TOKEN",          value = var.github_token },
      { name = "ATLANTIS_GH_WEBHOOK_SECRET", value = var.webhook_secret },
      { name = "ATLANTIS_REPO_ALLOWLIST",    value = "github.com/myorg/*" },
    ]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"  = aws_cloudwatch_log_group.atlantis.name
        "awslogs-region" = var.region
      }
    }
  }])
}
```

**Per-repo configuration (`atlantis.yaml`):**

```yaml
version: 3
projects:
  - name: prod-app
    dir: infra/environments/prod
    workspace: default
    terraform_version: v1.9.0
    autoplan:
      when_modified: ["**/*.tf", "../modules/**/*.tf"]
    apply_requirements:
      - approved
      - mergeable
```

`apply_requirements` enforces that at least one reviewer has approved the PR and no blocking checks are failing before Atlantis will accept an `atlantis apply` comment.

---

## 14.5 Terraform Cloud Runs

Terraform Cloud (HCP Terraform) is HashiCorp's hosted platform. It replaces the need for S3 state, DynamoDB locking, and a self-hosted CI runner.

```hcl
terraform {
  cloud {
    organization = "myorg"
    workspaces {
      name = "prod-app"
    }
  }
}
```

**Run workflow in Terraform Cloud:**

1. Push a branch → **speculative plan** (shown as a VCS PR check — no apply)
2. Merge to main → **standard run** (plan + confirmation required before apply)
3. Team members review the plan in the Terraform Cloud UI and click Confirm or Discard
4. State is stored and locked inside Terraform Cloud — no S3 bucket needed
5. Policy checks (Sentinel) can enforce org-wide rules before any apply runs:
   - "No S3 buckets with `acl = public-read`"
   - "All EC2 instances must have a `CostCenter` tag"

Terraform Cloud is well suited for small-to-medium teams that want managed state and a UI without operating their own infrastructure. GitHub Actions or Atlantis is a better fit when you need deeper pipeline customisation or want to stay within existing CI tooling.

---

## 14.6 Drift Detection

Drift occurs when someone makes a manual change to infrastructure outside of Terraform — through the console, CLI, or another tool. Without detection, the Terraform state diverges from reality silently until the next apply, which can cause unexpected changes or failures.

```yaml
# GitHub Actions: daily drift detection
name: Terraform Drift Detection

on:
  schedule:
    - cron: "0 8 * * 1-5"   # weekdays at 8 am UTC

jobs:
  drift-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-terraform-role
          aws-region: us-east-1

      - run: terraform init
        working-directory: infra/environments/prod

      - name: Check for drift
        working-directory: infra/environments/prod
        run: |
          terraform plan -detailed-exitcode -refresh-only || \
          (echo "DRIFT DETECTED — infrastructure has changed outside Terraform" && exit 1)
```

`terraform plan -detailed-exitcode` exits with:
- `0` — no changes (no drift)
- `1` — error
- `2` — changes detected (drift present)

The `-refresh-only` flag updates the state to match real infrastructure without proposing any resource changes. Combining both flags produces a clean "drift or no drift" signal. When the job fails, send an alert to Slack or PagerDuty to prompt investigation before the next planned apply.

---

## 14.7 Pre-commit Hooks for Terraform

Catch issues before they ever reach CI by running checks at commit time:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.0
    hooks:
      - id: terraform_fmt           # format all .tf files
      - id: terraform_validate      # validate syntax and schema
      - id: terraform_docs          # auto-generate README from variables/outputs
      - id: terraform_tflint        # lint rules (missing tags, deprecated syntax)
      - id: checkov                 # security scanning — catches misconfigs
```

```bash
pip install pre-commit
pre-commit install
# git commit now triggers all checks; commit is blocked if any check fails
```

`terraform_docs` keeps your module README in sync automatically — the hook rewrites the variables and outputs tables every time you commit. `checkov` catches common AWS security misconfigurations (unencrypted volumes, public S3 buckets, missing CloudTrail) before they reach a PR.

---

## 14.8 CI/CD Best Practices for Terraform

```
Plan on every PR               — never apply without a reviewed plan
Apply only on merge to main    — not on every push
Separate pipelines per env     — prod requires manual approval; dev can auto-apply
Use OIDC for AWS auth in CI    — no long-lived access keys stored as secrets
Pin Terraform and provider versions — reproducible runs; no surprise provider changes
Run fmt and validate in CI     — reject unformatted or invalid configs at PR time
Run tfsec/checkov for security — catch misconfigs before they reach AWS
Store plan as artifact         — apply uses the exact plan that was reviewed
Drift detection on schedule    — catch manual changes before the next planned apply
Notifications on apply         — Slack or PagerDuty on success and failure
```

---

## Summary

| Approach | Best For | State Storage | Apply Gate |
|---|---|---|---|
| GitHub Actions | Teams already on GitHub CI | S3 + DynamoDB | Merge to main |
| Atlantis | GitOps via PR comments | S3 + DynamoDB | `atlantis apply` comment + approval |
| Terraform Cloud | Managed, hosted workflow | HCP Terraform | UI confirmation or auto |
| Local (not recommended for teams) | Solo learning | Local or S3 | Manual |

---

## Knowledge Check

1. Why is `-out=tfplan` important in a CI workflow? What problem does it solve?
2. What is OIDC and why is it preferred over storing AWS access keys in GitHub secrets?
3. In an Atlantis workflow, what must happen before `atlantis apply` will succeed when `apply_requirements` includes `approved`?
4. `terraform plan -detailed-exitcode` exits with code `2`. What does this mean in the context of drift detection?
5. You run `terraform fmt -check` in CI and it fails. What is the correct fix — in CI or locally?

---

## Hands-on Exercise

**Goal:** Build a complete GitHub Actions Terraform pipeline with OIDC authentication.

**Tasks:**

1. **Workflow file** — create `.github/workflows/terraform.yml` that:
   - Triggers on PRs and pushes to `main` for changes under `infra/`
   - Runs `terraform fmt -check`, `terraform validate`, and `terraform plan` on every PR
   - Posts the plan output as a PR comment using `actions/github-script`
   - Runs `terraform apply` automatically on merge to `main`

2. **OIDC role** — write Terraform HCL (in a bootstrap directory) that creates:
   - The GitHub OIDC provider in AWS
   - An IAM role with a trust policy scoped to your specific GitHub repo
   - An IAM policy attachment granting only the permissions your Terraform code needs

3. **Drift detection** — add a second workflow `drift-detection.yml` that runs on a weekday cron, runs `terraform plan -detailed-exitcode -refresh-only`, and exits non-zero if drift is found.

4. **Pre-commit** — add a `.pre-commit-config.yaml` with `terraform_fmt`, `terraform_validate`, and `checkov`. Run `pre-commit run --all-files` and fix any findings.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./13-advanced-hcl.md">← Previous: Advanced HCL: Loops, Conditionals & Dynamic Blocks</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./15-best-practices.md">Next: Best Practices & Patterns →</a>
</div>
