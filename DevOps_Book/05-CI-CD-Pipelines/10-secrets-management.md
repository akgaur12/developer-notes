# Chapter 10 — Secrets Management in CI/CD

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand why secrets management is critical in CI/CD pipelines
- Store and use secrets safely in GitHub Actions and GitLab CI
- Integrate HashiCorp Vault and AWS Secrets Manager with pipelines
- Prevent accidental secret exposure in logs and version control
- Implement automated secret scanning as a CI gate

---

## 10.1 The Secrets Problem in CI/CD

Pipelines need credentials to do real work: deploy keys, cloud API tokens, registry passwords, database URLs. The trouble is that this power makes CI/CD systems a prime target for attackers — they hold the keys to everything.

**Bad practices are extremely common:**

- Secrets hardcoded in `Jenkinsfile` or `.gitlab-ci.yml`
- Committed in `.env` files that "accidentally" make it to the repo
- Printed in build logs via `echo` or debug output
- Shared between environments (dev and prod using the same API key)

**Real-world incidents:**

- **CodeCov breach (2021):** Attackers modified the CodeCov bash uploader script to exfiltrate all environment variables — including CI secrets — from thousands of pipelines.
- **CircleCI breach (2023):** A malware-infected engineer's laptop exposed customer secrets stored in CircleCI. All secrets had to be rotated immediately across all affected pipelines.

The lesson: treat secrets as first-class citizens. Never hardcode. Never log. Rotate regularly.

---

## 10.2 GitHub Actions Secrets

GitHub provides a built-in secrets store accessible from workflow YAML.

**Three scopes:**

| Scope | Where set | Who can access |
|---|---|---|
| Repository secrets | Repo → Settings → Secrets and variables → Actions | All workflows in that repo |
| Environment secrets | Repo → Settings → Environments | Only jobs targeting that environment |
| Organization secrets | Org → Settings → Secrets | Repos granted access by org admin |

```yaml
# Using secrets in workflows
jobs:
  deploy:
    environment: production     # environment secrets available here
    steps:
      - run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.API_KEY }}
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
          # GITHUB_TOKEN is always available automatically
          REGISTRY_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Important rules:**

- Secrets are **masked in logs**: if accidentally printed, GitHub replaces the value with `***`
- Secrets are **not passed to forked PRs** by default — this prevents secret theft via malicious PR workflows
- Secret names cannot start with `GITHUB_` (reserved prefix)
- Maximum 64 KB per secret; maximum 100 repository secrets
- Secret values **cannot be read back** after being set (write-only via UI) — store a copy elsewhere

---

## 10.3 Environment-Scoped Secrets and Protection Rules

Environments let you gate deployments with approval workflows and restrict which secrets are accessible to which jobs.

```yaml
# prod secrets only available in jobs targeting the production environment
jobs:
  deploy-prod:
    environment: production     # triggers protection rules check
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ secrets.PROD_DB_PASSWORD }}"  # only accessible here
```

**Configure environment protection rules:**

- **Required reviewers:** Named people or teams must approve before the job runs
- **Deployment branch whitelist:** Only `main` or release branches can deploy to production
- **Wait timer:** Enforce a minimum delay (e.g., 10 minutes) before deployment proceeds

This separation ensures staging and production credentials are never accessible in the same job, even if a workflow is misconfigured.

---

## 10.4 GitLab CI Secrets

GitLab's equivalent is **CI/CD Variables**, configured under Settings → CI/CD → Variables.

**Variable types:**

| Type | Behavior |
|---|---|
| Variable | Plaintext, accessible as `$VAR_NAME` in job script |
| File | Written to a temporary file; `$VAR_NAME` holds the file path |
| Masked | Value is redacted from job logs |
| Protected | Only available on protected branches and tags |

```yaml
deploy:
  script:
    - echo "$API_KEY"           # from CI/CD variable
    - cat "$KUBECONFIG"         # file-type variable: $KUBECONFIG is a path
  environment: production
```

**Best practice:** Combine **Masked** + **Protected** for production credentials. Use **File** type for multi-line secrets like kubeconfig files, certificates, and SSH keys.

---

## 10.5 HashiCorp Vault Integration

HashiCorp Vault is the industry standard for secrets management at scale. Vault provides dynamic secrets, fine-grained access policies, and a full audit log.

**OIDC authentication — no stored credentials needed:**

```yaml
# GitHub Actions + Vault via OIDC (no long-lived credentials stored)
- name: Import Secrets from Vault
  uses: hashicorp/vault-action@v3
  with:
    url: https://vault.mycompany.com
    method: jwt
    jwtGithubAudience: https://github.com/myorg
    role: github-ci
    secrets: |
      secret/data/myapp/prod db_password | DB_PASSWORD ;
      secret/data/myapp/prod api_key | API_KEY ;
      secret/data/aws/creds access_key | AWS_ACCESS_KEY_ID ;
      secret/data/aws/creds secret_key | AWS_SECRET_ACCESS_KEY

- name: Deploy
  run: ./deploy.sh
  env:
    DB_PASSWORD: ${{ env.DB_PASSWORD }}
```

**How OIDC works:**

1. GitHub generates a short-lived JWT for the workflow run
2. Vault verifies the JWT against GitHub's OIDC endpoint
3. If the JWT matches the role's bound claims (repo, branch, environment), Vault issues a token
4. The token fetches the requested secrets — no stored credentials anywhere

This is the gold standard: even if the workflow YAML is leaked, there are no long-lived credentials to steal.

---

## 10.6 AWS Secrets Manager

For AWS-native deployments, combining OIDC with Secrets Manager eliminates the need for stored AWS credentials entirely.

```yaml
# OIDC → AWS → Secrets Manager (no stored AWS credentials)
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions
    aws-region: us-east-1

- name: Get secrets
  run: |
    DB_PASSWORD=$(aws secretsmanager get-secret-value \
      --secret-id myapp/prod/db-password \
      --query SecretString --output text)
    echo "DB_PASSWORD=$DB_PASSWORD" >> $GITHUB_ENV
    echo "::add-mask::$DB_PASSWORD"   # mask in logs before using
```

The `::add-mask::` workflow command tells the runner to redact that value from all subsequent log output — critical when fetching secrets dynamically at runtime.

---

## 10.7 Preventing Accidental Secret Exposure

**Never echo secrets directly:**

```yaml
# BAD — leaks value in job logs
- run: echo "${{ secrets.API_KEY }}"

# GOOD — use the secret without printing it
- run: ./deploy.sh
  env:
    API_KEY: ${{ secrets.API_KEY }}
```

**Mask dynamically fetched secrets immediately:**

```yaml
- run: |
    SECRET=$(fetch-from-vault)
    echo "::add-mask::$SECRET"         # mask before any use
    echo "SECRET=$SECRET" >> $GITHUB_ENV
```

**Avoid secrets in URLs (they appear in curl verbose logs and error messages):**

```yaml
# BAD — password visible in logs
- run: curl https://user:$PASSWORD@api.example.com/endpoint

# GOOD — use headers instead
- run: curl -H "Authorization: Bearer $API_KEY" https://api.example.com/endpoint
```

**Rotation after exposure:**

Treat any leaked secret as compromised immediately. Do not wait to confirm misuse. The correct response is:

1. Revoke the exposed credential
2. Issue a new one
3. Update all pipelines using the old value
4. Audit access logs for the exposure window

---

## 10.8 Secret Detection in CI

Scanning for accidentally committed secrets should be a mandatory CI gate — it catches mistakes before they reach the default branch.

**TruffleHog:**

```yaml
- name: Scan for secrets
  uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: ${{ github.event.repository.default_branch }}
    head: HEAD
    extra_args: --only-verified
```

**GitLeaks:**

```yaml
- name: GitLeaks scan
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Both tools scan the git diff between the PR branch and the base, flagging high-entropy strings and known secret patterns (AWS keys, GitHub tokens, Stripe keys, etc.). The `--only-verified` flag on TruffleHog reduces false positives by only reporting secrets it can verify are active.

**Pre-commit hooks for local detection:**

```bash
# Install detect-secrets pre-commit hook
pip install detect-secrets
detect-secrets scan > .secrets.baseline
pre-commit install
```

This catches secrets before they ever leave the developer's machine.

---

## 10.9 SSH Keys and Deploy Keys

Deploy keys grant read-only (or read-write) access to a single repository, making them safer than personal access tokens.

```yaml
# Repository deploy key setup:
# 1. Generate: ssh-keygen -t ed25519 -C "ci-deploy-key"
# 2. Add public key to repo: Settings → Deploy keys → Add deploy key
# 3. Store private key as a repository secret

- name: Setup SSH
  uses: webfactory/ssh-agent@v0.9.0
  with:
    ssh-private-key: ${{ secrets.DEPLOY_KEY }}

- name: Deploy
  run: ssh deploy@server.mycompany.com "cd /app && git pull && restart-app"
```

For accessing multiple private repositories in the same workflow, add each private key separately and configure SSH to route by hostname:

```yaml
- uses: webfactory/ssh-agent@v0.9.0
  with:
    ssh-private-key: |
      ${{ secrets.DEPLOY_KEY_APP }}
      ${{ secrets.DEPLOY_KEY_SHARED_LIB }}
```

---

## 10.10 Secrets Rotation

Long-lived credentials are a liability. Rotation limits the blast radius of any single exposure.

**Preference hierarchy (most to least preferred):**

1. **OIDC / short-lived tokens** — expires in minutes, no rotation needed
2. **Auto-rotated credentials** — AWS Secrets Manager can rotate RDS passwords automatically
3. **Manually rotated on schedule** — rotate monthly or quarterly with calendar reminders
4. **Never rotated** — this is the dangerous state most teams find themselves in

**Rotation checklist:**

- Rotate all long-lived secrets on a defined schedule
- Never share secrets between environments — dev/staging/prod each have their own set
- Audit secret access logs regularly (who accessed what, when)
- Automate rotation where the service supports it
- Document where each secret is used so rotation doesn't break pipelines silently

---

## Summary

| Topic | Key Takeaway |
|---|---|
| Secrets in CI/CD | Never hardcode; never log; treat as compromised if leaked |
| GitHub Actions | Use environment-scoped secrets with protection rules for production |
| GitLab CI | Combine Masked + Protected variable flags for production credentials |
| Vault / OIDC | OIDC eliminates stored credentials entirely — prefer it at scale |
| Accidental exposure | Mask dynamic secrets; avoid secrets in URLs; rotate immediately on leak |
| Secret scanning | TruffleHog or GitLeaks as a mandatory CI gate on every PR |
| Rotation | Prefer short-lived OIDC tokens; rotate all long-lived keys on schedule |

---

## Knowledge Check

1. Why are secrets not passed to forked PR workflows in GitHub Actions by default?
2. What is the difference between a Masked variable and a Protected variable in GitLab CI?
3. Explain how OIDC authentication to HashiCorp Vault eliminates the need for stored credentials.
4. What does `echo "::add-mask::$SECRET"` do, and when should you use it?
5. A developer accidentally commits an AWS access key to a public repository. What are the immediate steps?

---

## Hands-on Exercise

**Goal:** Implement environment-scoped secrets with protection rules and add automated secret scanning to a workflow.

**Steps:**

1. Create two environments in your GitHub repository: `staging` and `production`
2. Add a different `DB_CONNECTION_STRING` secret to each environment
3. Write a workflow that deploys to staging on push to `main`, and to production only after a required reviewer approves
4. Add TruffleHog scanning as the first job in the workflow — the deploy jobs should only run if scanning passes
5. Test the masking: add a step that tries to `echo` a secret value, confirm it appears as `***` in the logs
6. Verify that a PR from a fork cannot access repository secrets

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="09-jenkins.md">← Previous: Jenkins</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="11-intermediate-concepts.md">Next: Intermediate Concepts →</a>
</div>
