# Chapter 16 — Hands-On Projects

## Learning Objectives

By the end of this chapter you will have:

- Added SAST, dependency scanning, and secret-leak detection to a real CI pipeline, and proven each one catches a deliberately introduced vulnerability
- Built a signed, scanned, SBOM-attested container image and enforced that signature at deploy time with a Kubernetes admission policy
- Hardened a running cluster against a CIS benchmark, deployed runtime threat detection with Falco, and replaced static cloud credentials in CI with short-lived OIDC tokens
- Designed a complete, end-to-end DevSecOps platform spanning code, build, deploy, and runtime — with compliance evidence mapped to real audit requirements — suitable as the centerpiece of a DevOps/platform/security engineering portfolio

---

## Project 1 — Secure a CI Pipeline (Beginner)

**Goal:** Take an existing CI/CD pipeline (the GitHub Actions pipeline from CI/CD Pipelines Chapter 6, or a simple one built for this project) and add three independent layers of shift-left defense: SAST on every pull request, automated dependency updates, and secret-leak detection both locally and in CI. Each layer is proven with a deliberately introduced vulnerability.

**Architecture:**

```
Developer laptop                     GitHub
  ├── pre-commit hook (gitleaks)  →   Pull Request opened
                                        ├── Job: sast-scan (Semgrep)       — annotates PR inline
                                        ├── Job: dependency-review          — Dependabot config
                                        └── Job: secret-scan (gitleaks CI) — fails build on match
```

**Step 1 — Add a Semgrep SAST scan that annotates the PR** (Chapter 4). Semgrep's GitHub Action posts findings as inline PR review comments, which is what makes SAST usable by developers instead of a wall of text in a separate tab:

```yaml
# .github/workflows/sast.yml
name: SAST
on:
  pull_request:
    branches: [main]

jobs:
  semgrep:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - name: Run Semgrep
        uses: semgrep/semgrep-action@v1
        with:
          config: p/owasp-top-ten
          generateSarif: true
      - name: Upload SARIF to code scanning
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: semgrep.sarif
```

**Step 2 — Introduce a deliberate SQL-injection pattern** to prove the scan actually catches something real, not just runs green:

```python
# vulnerable.py — deliberately introduced for this project
def get_user(conn, username):
    query = "SELECT * FROM users WHERE username = '" + username + "'"
    return conn.execute(query)   # string-concatenated SQL — classic injection
```

Open a PR containing this file. `p/owasp-top-ten` includes a rule matching exactly this pattern (`tainted-sql-string` / string concatenation into a query), and Semgrep should post an inline comment on the offending line within a minute or two of the PR opening.

**Step 3 — Add automated dependency updates** (Chapter 5) so vulnerable dependencies get PRs opened against them automatically instead of silently aging:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    labels: ["dependencies", "security"]
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

Commit a `requirements.txt` pinning a package to a version with a known published CVE (check the GitHub Advisory Database for a currently-archived, non-exploitable example version to use safely for this exercise) and confirm Dependabot opens a PR bumping it within its next scheduled run — or trigger it manually from the repo's **Insights → Dependency graph → Dependabot** tab rather than waiting a full week.

**Step 4 — Add a `gitleaks` pre-commit hook** so a secret never leaves the developer's machine in the first place:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
```

```bash
pip install pre-commit
pre-commit install
```

**Step 5 — Add a CI-level secondary scan** (Chapter 3), because a pre-commit hook can be skipped with `git commit --no-verify` or simply isn't installed on every contributor's machine — CI is the backstop that cannot be bypassed by a local flag:

```yaml
# .github/workflows/secret-scan.yml
name: Secret Scan
on: [pull_request]

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # gitleaks needs full history to scan prior commits, not just the diff
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITLEAKS_LICENSE: ""   # not required for the open-source scan mode
```

**Step 6 — Prove the secret-detection layer with a deliberate fake key**, then confirm both layers catch it independently:

```bash
echo 'AWS_SECRET_ACCESS_KEY = "AKIAFAKEEXAMPLEKEY1234"' >> config.py
git add config.py
git commit -m "test: add config"   # the pre-commit hook should block this locally
```

Then bypass the hook on purpose to prove the CI backstop works too:

```bash
git commit --no-verify -m "test: add config (bypassing hook)"
git push origin test-secret-detection
```

Open a PR and confirm the `Secret Scan` job fails the build with the fake key flagged, even though the local hook was skipped.

**Success criteria:** The deliberately introduced SQL-injection pattern is flagged by Semgrep with an inline PR annotation on the exact line. The Dependabot config opens a real PR against an outdated dependency. The fake API key is blocked by the local `gitleaks` pre-commit hook when committed normally, and separately caught and failed by the CI-level `gitleaks` scan when the hook is deliberately bypassed with `--no-verify`.

---

## Project 2 — Container and Supply Chain Security (Intermediate)

**Goal:** Take an application's Dockerfile (from Docker Chapter 10 or Kubernetes Basics) and build a complete signed-and-verified supply chain: a Trivy scan that fails the build on Critical/High CVEs, an SBOM generated with `syft`, keyless image signing with `cosign` tied to the CI workflow's own OIDC identity, and a Kyverno admission policy in Kubernetes that rejects any Pod running an unsigned image.

**Architecture:**

```
git push → CI workflow (OIDC identity: repo:org/app:ref:refs/heads/main)
  ├── docker build
  ├── trivy scan   → fail build on CRITICAL/HIGH
  ├── syft sbom     → attach SBOM as an attestation
  ├── cosign sign   → keyless, signed by the workflow's OIDC token, no private key stored anywhere
  └── docker push  →  registry

kind cluster
  └── Kyverno ClusterPolicy: verifyImages
        └── rejects any Pod whose image has no valid cosign signature
```

**Step 1 — Add a Trivy scan that fails the build** (Chapter 6):

```yaml
# .github/workflows/build-and-scan.yml (excerpt)
- name: Build image
  run: docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .

- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@0.24.0
  with:
    image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
    severity: CRITICAL,HIGH
    exit-code: "1"          # non-zero exit fails the workflow step
    ignore-unfixed: true    # don't fail the build on CVEs with no available fix yet
```

**Step 2 — Generate an SBOM with `syft`** (Chapter 7), so the exact contents of the image are recorded as a queryable artifact rather than only being knowable by re-scanning the image later:

```yaml
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: ghcr.io/${{ github.repository }}:${{ github.sha }}
    format: cyclonedx-json
    output-file: sbom.cdx.json

- name: Upload SBOM as a build artifact
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: sbom.cdx.json
```

**Step 3 — Sign the image keylessly with `cosign`** (Chapter 11), using the CI workflow's own OIDC identity instead of a long-lived private key stored as a secret:

```yaml
permissions:
  contents: read
  packages: write
  id-token: write   # required — this is what lets the workflow request an OIDC token from GitHub's issuer

steps:
  - name: Push image
    run: docker push ghcr.io/${{ github.repository }}:${{ github.sha }}

  - name: Install cosign
    uses: sigstore/cosign-installer@v3

  - name: Sign image (keyless)
    run: cosign sign --yes ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
    env:
      COSIGN_EXPERIMENTAL: "1"

  - name: Attach SBOM attestation
    run: cosign attest --yes --predicate sbom.cdx.json --type cyclonedx \
      ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
```

Keyless signing works because GitHub Actions can mint a short-lived OIDC token identifying the exact workflow, repo, and ref that's running; `cosign` exchanges that token with Sigstore's Fulcio for a short-lived signing certificate, signs with a throwaway key that's discarded immediately after, and records the signature in Sigstore's public transparency log (Rekor) — there is no long-lived private key to leak, rotate, or steal.

**Step 4 — Verify the signature manually** before trusting the cluster-side enforcement:

```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/your-org/your-app/.github/workflows/build-and-scan.yml@refs/heads/main" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/your-org/your-app@sha256:...
```

**Step 5 — Install Kyverno and write the admission policy** rejecting unsigned images cluster-wide:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

```yaml
# kyverno-verify-images.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  webhookTimeoutSeconds: 30
  rules:
    - name: verify-cosign-signature
      match:
        any:
          - resources:
              kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "ghcr.io/your-org/*"
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/your-org/your-app/.github/workflows/build-and-scan.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: https://rekor.sigstore.dev
```

**Step 6 — Prove both directions.** Deploy an unrelated, unsigned public image and confirm the admission webhook rejects it; then deploy the properly built, scanned, and signed image and confirm it's admitted:

```bash
kubectl run unsigned-test --image=nginx:latest
# Expected: Error from server: admission webhook "validate.kyverno.svc-fail" denied the request

kubectl run signed-test --image=ghcr.io/your-org/your-app@sha256:...
# Expected: pod/signed-test created
```

**Success criteria:** `docker build` fails outright when the base image or a dependency introduces a Critical/High CVE with an available fix. The SBOM is generated and attached as an attestation on the image digest. `cosign verify` succeeds against the pushed image using only the workflow's OIDC identity, with no private key ever present in any secret store. The Kyverno policy rejects the unsigned `nginx:latest` deployment attempt and admits the properly signed image without modification.

---

## Project 3 — Kubernetes and Pipeline Hardening (Advanced)

**Goal:** Run a CIS benchmark against a real cluster and fix real findings, deploy runtime threat detection that can catch what static scanning cannot (a live process spawning inside a container), route those alerts to a human within seconds, and eliminate long-lived cloud credentials from a CI pipeline entirely.

**Architecture:**

```
kind cluster
  ├── kube-bench Job          → CIS Kubernetes Benchmark report
  ├── Falco DaemonSet          → kernel-level runtime detection
  │     └── custom rule: shell spawned in namespace "payments"
  ├── Falcosidekick             → routes Falco alerts to Alertmanager
  └── Alertmanager              → routes to #security-alerts Slack channel

CI workflow
  └── OIDC federation → cloud provider's IAM role (no static AWS_ACCESS_KEY_ID/SECRET stored anywhere)
```

**Step 1 — Run `kube-bench` against the `kind` cluster** (Chapter 10):

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench
```

**Step 2 — Remediate at least three findings.** Typical `kind`-cluster findings and their fixes:

```
[FAIL] 1.2.20 Ensure that the --profiling argument is set to false
  Fix: add --profiling=false to the kube-apiserver static pod manifest

[FAIL] 4.2.6 Ensure that the --protect-kernel-defaults argument is set to true
  Fix: add protectKernelDefaults: true to the kubelet config, restart kubelet

[FAIL] 5.1.5 Ensure that default service accounts are not actively used
  Fix: set automountServiceAccountToken: false on the default ServiceAccount
       in every namespace that doesn't explicitly need it:
       kubectl patch serviceaccount default -n payments \
         -p '{"automountServiceAccountToken": false}'
```

Re-run `kube-bench` after each fix and confirm the specific check flips from `FAIL` to `PASS` — remediating blind without re-verification is how a "fix" silently regresses in the next cluster rebuild.

**Step 3 — Deploy Falco** (Chapter 10):

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco --namespace falco --create-namespace \
  --set falcosidekick.enabled=true
```

**Step 4 — Write a custom rule** detecting a shell spawned inside a specific namespace's Pods — the `payments` namespace, treated as higher-sensitivity than the rest of the cluster:

```yaml
# falco-custom-rules.yaml
- rule: Shell Spawned in Payments Namespace
  desc: A shell was spawned inside a Pod in the payments namespace — this namespace should never see interactive shell activity in normal operation
  condition: >
    spawned_process and
    proc.name in (bash, sh, zsh) and
    k8s.ns.name = "payments"
  output: >
    Shell spawned in payments namespace (user=%user.name command=%proc.cmdline
    pod=%k8s.pod.name container=%container.name)
  priority: CRITICAL
  tags: [shell, payments, mitre_execution]
```

```bash
kubectl create configmap falco-custom-rules --from-file=falco-custom-rules.yaml -n falco
helm upgrade falco falcosecurity/falco -n falco \
  --set customRules."custom-rules\.yaml"="$(cat falco-custom-rules.yaml)"
```

**Step 5 — Route Falco alerts through Alertmanager to Slack** (Monitoring Chapter 7), reusing the exact same grouping/routing pattern already trusted for reliability alerts, now carrying security signal instead:

```yaml
# falcosidekick values — point at Alertmanager rather than directly at Slack,
# so security alerts inherit the same grouping/inhibition discipline as every other alert
falcosidekick:
  config:
    alertmanager:
      hostport: "http://alertmanager.monitoring.svc:9093"
      minimumpriority: "warning"
```

```yaml
# alertmanager route addition
route:
  routes:
    - matchers: [source="falco"]
      receiver: slack-security-alerts
      group_wait: 5s      # security alerts should not wait as long as routine reliability alerts
      repeat_interval: 15m
receivers:
  - name: slack-security-alerts
    slack_configs:
      - channel: "#security-alerts"
        title: ":rotating_light: Falco: {{ .CommonLabels.rule }}"
        text: "{{ .CommonAnnotations.description }}"
```

**Step 6 — Prove it end to end:**

```bash
kubectl exec -it -n payments <some-pod> -- /bin/sh
```

Watch `#security-alerts` in Slack and confirm the message arrives within seconds of the `exec`, not minutes.

**Step 7 — Migrate CI's cloud credentials to OIDC** (Chapter 11, tying back to Terraform Chapter 14's remote-backend authentication). Replace a static `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` pair stored as a GitHub Actions secret with a federated IAM role trust relationship:

```json
// AWS IAM role trust policy — trusts GitHub's OIDC issuer, scoped to one repo/branch
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
        "StringLike": { "token.actions.githubusercontent.com:sub": "repo:your-org/your-app:ref:refs/heads/main" }
      }
    }
  ]
}
```

```yaml
# .github/workflows/deploy.yml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
      aws-region: us-east-1
      # no access-key-id / secret-access-key anywhere in this file or in secrets
  - run: terraform apply -auto-approve
```

**Success criteria:** `kube-bench` shows at least three findings flip from `FAIL` to `PASS` after remediation, re-verified by re-running the benchmark. Exec'ing a shell into a test Pod in the `payments` namespace triggers a Falco alert that arrives in the `#security-alerts` Slack channel within seconds. The CI pipeline deploys successfully to AWS using only an OIDC-federated role, with `git grep -r "AWS_SECRET_ACCESS_KEY"` returning no matches anywhere in the repository or its configured secrets.

---

## Project 4 — Full DevSecOps Platform (Production-Grade Capstone)

**Goal:** Design a complete, end-to-end DevSecOps pipeline for a multi-service application — real manifests and configs throughout, not a slide diagram — that gates every stage of the software lifecycle with an automated control, monitors the running system for both reliability and security signal, and produces the evidence an auditor would actually ask for. This is explicitly the single most comprehensive project in the entire 11-course roadmap: it draws on Docker (Topic 4), CI/CD (Topic 5), AWS (Topic 6), Terraform (Topic 7), both Kubernetes courses (Topics 8-9), Monitoring & Logging (Topic 10), and this entire Security course all at once.

**Architecture:**

```
┌─────────────────────────────────────────────────────────────────────┐
│ Pull Request                                                          │
│  ├── SAST (Semgrep)             — inline annotations                  │
│  ├── SCA (Dependabot/Snyk)      — dependency vulnerabilities           │
│  ├── IaC scan (tfsec + Checkov) — Terraform misconfigurations          │
│  └── Policy check (Conftest)    — OPA policy against `terraform plan`  │
├─────────────────────────────────────────────────────────────────────┤
│ Merge to main                                                          │
│  ├── docker build → trivy scan → syft SBOM → cosign sign (OIDC)        │
│  └── push to registry, digest-pinned                                   │
├─────────────────────────────────────────────────────────────────────┤
│ Deploy (OIDC-authenticated, no static cloud credentials)               │
│  └── Kyverno admission gate: signature valid? scan passed? → allow/deny│
├─────────────────────────────────────────────────────────────────────┤
│ Running cluster                                                        │
│  ├── kube-bench (scheduled)      — CIS benchmark drift detection       │
│  ├── Falco + Falcosidekick        — runtime threat detection            │
│  ├── Alertmanager                  — routes to #security-oncall         │
│  └── Loki/ELK (extended)             — audit logs + security events →   │
│                                         basic SIEM-style unified view    │
├─────────────────────────────────────────────────────────────────────┤
│ Governance                                                             │
│  ├── Incident response runbooks (leaked secret, Falco intrusion)        │
│  └── Compliance evidence map (SOC 2 / PCI-DSS ↔ pipeline controls)      │
└─────────────────────────────────────────────────────────────────────┘
```

**Step 1 — PR-time gates.** Combine Project 1's SAST/SCA with IaC scanning (Chapter 9) and policy-as-code enforcement against a Terraform plan, so infrastructure changes are checked before `apply`, not after:

```yaml
# .github/workflows/pr-gates.yml (excerpt)
jobs:
  iac-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: tfsec
        uses: aquasecurity/tfsec-action@v1.0.3
      - name: Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: infra/

  policy-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: terraform -chdir=infra plan -out=tfplan.binary
      - run: terraform -chdir=infra show -json tfplan.binary > tfplan.json
      - name: Conftest against the plan
        run: conftest test tfplan.json --policy policy/
```

```rego
# policy/no_public_s3.rego — OPA policy blocking a public S3 bucket before apply
package main

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket_acl"
  resource.change.after.acl == "public-read"
  msg := sprintf("S3 bucket %v must not use public-read ACL", [resource.address])
}
```

**Step 2 — Merge-time build, scan, sign, SBOM.** Directly reuses Project 2's pipeline for every service in the multi-service application, run as a matrix job so each service is built and attested independently:

```yaml
strategy:
  matrix:
    service: [api-gateway, payments-service, checkout-service, search-service]
```

**Step 3 — Deploy-time admission gate.** Extend Project 2's Kyverno policy to check both signature validity and that the image passed its scan, verified via an attached Trivy scan-result attestation rather than signature alone:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-signed-and-scanned
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature-and-scan-attestation
      match:
        any: [{ resources: { kinds: ["Pod"] } }]
      verifyImages:
        - imageReferences: ["ghcr.io/your-org/*"]
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/your-org/*/.github/workflows/*.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
          attestations:
            - predicateType: https://in-toto.io/attestation/scan/v0.1
              conditions:
                - all:
                    - key: "{{ result.critical }}"
                      operator: Equals
                      value: 0
```

**Step 4 — Runtime hardening and monitoring.** Combine Project 3's `kube-bench` (scheduled as a recurring CronJob rather than a one-off run, so CIS drift is caught continuously) and Falco, routed to a dedicated security on-call rotation rather than a general channel:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kube-bench-scheduled
  namespace: security
spec:
  schedule: "0 6 * * 1"   # weekly CIS drift check
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: kube-bench
              image: aquasec/kube-bench:latest
              args: ["--outputfile", "/results/kube-bench-$(date +%F).json", "--json"]
          restartPolicy: Never
```

```yaml
route:
  routes:
    - matchers: [source="falco", priority=~"Critical|Emergency"]
      receiver: pagerduty-security-oncall
      group_wait: 0s
receivers:
  - name: pagerduty-security-oncall
    pagerduty_configs:
      - routing_key: "<security-oncall-integration-key>"
```

**Step 5 — Centralize audit logs and security events into a basic SIEM-style view**, extending the Loki/ELK stack from Monitoring Topic 10 rather than standing up a separate tool:

```yaml
# promtail scrape config addition — ship Kubernetes audit logs and Falco events
# into the same Loki instance already used for application logs, with a
# dedicated "security" label so they can be queried and retained separately
scrape_configs:
  - job_name: k8s-audit-log
    static_configs:
      - targets: [localhost]
        labels:
          job: k8s-audit
          category: security
          __path__: /var/log/kubernetes/audit.log
  - job_name: falco-events
    static_configs:
      - targets: [localhost]
        labels:
          job: falco
          category: security
          __path__: /var/log/falco/events.log
```

```logql
# LogQL: every security-relevant event across audit logs and Falco, last 24h
{category="security"} | json | line_format "{{.job}}: {{.message}}"
```

This does not need to be a full commercial SIEM to be useful — a dedicated Grafana dashboard querying `{category="security"}` alongside a saved set of LogQL queries for common investigation questions ("all `exec` audit events into the `payments` namespace in the last hour") already delivers most of the practical value for a team of this scale.

**Step 6 — Write and tabletop-rehearse two incident response runbooks** (Chapter 13):

```
## Runbook: Leaked Secret Discovered

1. Rotate the credential immediately at the source system (cloud IAM key,
   database password, third-party API key) — rotation always comes before
   investigation, since every minute the old credential remains valid is
   continued exposure
2. Revoke/invalidate the specific leaked value if the provider supports
   explicit revocation independent of rotation (e.g., GitHub PAT revocation)
3. Search version control history for the credential's full blast radius —
   was it in one commit or does it appear in older history/forks/CI logs?
   git log -p --all -S "<leaked-value-fragment>"
4. Check the audit trail of the compromised credential's provider for any
   usage during the exposure window — was it actually used by an attacker,
   or only exposed with no evidence of misuse?
5. Notify affected stakeholders per the incident severity, and file the
   blameless postmortem (Chapter 13) focused on why the secret reached
   version control (missing pre-commit hook? no CI backstop?) rather than
   who committed it
6. Add the specific pattern to gitleaks' custom rules if it was a bespoke
   internal credential format not covered by the default rule set
```

```
## Runbook: Falco-Detected Intrusion (Shell Spawned in Production Pod)

1. Do not immediately kill the Pod — first capture forensic state, since
   deleting it destroys the evidence needed to understand what happened:
   kubectl cp <namespace>/<pod>:/proc/1/root /tmp/forensic-capture
2. Isolate the Pod at the network layer without deleting it, using a
   deny-all NetworkPolicy scoped to just that Pod (Advanced Kubernetes
   Chapter 4) so it can't communicate further while still being inspectable
3. Correlate the Falco event's timestamp against audit logs and application
   logs in the centralized SIEM-style view (Step 5) to reconstruct what
   preceded the shell spawn — was it a compromised dependency, a leaked
   credential enabling `kubectl exec`, or an application-layer RCE?
4. Check whether the same image is running elsewhere in the cluster and
   whether the same suspicious process pattern appears on other nodes
5. Once forensics are captured, terminate the compromised Pod and force a
   redeploy from a known-good, freshly scanned and signed image
6. Rotate any credentials the compromised Pod's ServiceAccount had access
   to, on the assumption they may have been read from within the container
7. Blameless postmortem: how did an attacker reach code-execution in this
   Pod in the first place, and which earlier control (SAST, SCA, image
   scanning, NetworkPolicy) should have caught the entry vector
```

**Step 7 — Build the compliance evidence mapping table** (Chapter 12), showing which of this pipeline's automated controls satisfy which example SOC 2 / PCI-DSS requirements — turning the pipeline itself into continuously-generated audit evidence rather than a once-a-year scramble:

| Pipeline Control | SOC 2 Criterion (example) | PCI-DSS Requirement (example) |
|---|---|---|
| SAST on every PR (Semgrep) | CC7.1 — detection of security events | Req. 6.2 — identify and address common vulnerabilities |
| SCA / Dependabot | CC7.1, CC6.8 — malicious/vulnerable software detection | Req. 6.3 — vulnerable components addressed |
| Trivy image scan (build-blocking) | CC6.8 | Req. 6.3, Req. 5 (malware/vuln prevention) |
| cosign image signing + SBOM | CC8.1 — change management integrity | Req. 6.5 — secure software development |
| Kyverno admission policy | CC6.1, CC6.6 — logical access/deployment control | Req. 2.2 — secure configuration enforcement |
| `kube-bench` CIS scheduled scan | CC6.1 — configuration management | Req. 2.2 — CIS-based hardening standards |
| Falco runtime detection + alert routing | CC7.2 — monitoring for anomalies | Req. 10.6 — daily log/alert review |
| Centralized audit logs (Loki/ELK) | CC7.2, CC7.3 — incident evaluation | Req. 10 — logging and monitoring all access |
| Incident response runbooks | CC7.4 — incident response | Req. 12.10 — incident response plan |
| OIDC-based CI credentials | CC6.1 — least-privilege access | Req. 7, Req. 8.3 — restrict and authenticate access |

**Implementation steps:**

1. Stand up a multi-service demo application (three to four small services is enough to make the matrix build meaningful) with its infrastructure defined in Terraform.
2. Wire the full PR-gate workflow: SAST, SCA, `tfsec`/Checkov, and Conftest against the Terraform plan.
3. Wire the full merge-time pipeline: build, Trivy scan, `syft` SBOM, `cosign` sign, all per-service via a build matrix.
4. Deploy the Kyverno admission policy checking both signature and scan-result attestation, and confirm an unsigned or unscanned image is rejected.
5. Deploy `kube-bench` as a recurring CronJob and Falco with Falcosidekick routed to a dedicated security on-call receiver.
6. Extend the existing Loki/ELK stack with audit-log and Falco-event ingestion under a shared `category="security"` label, and build one Grafana dashboard for it.
7. Write both incident response runbooks and actually tabletop-rehearse them — walk through each step out loud against the running system rather than only writing the document.
8. Build the compliance evidence mapping table against your own pipeline's actual controls, and commit the entire platform (manifests, policies, runbooks, and the evidence table) to a public repository with a README describing the architecture end to end.

**Success criteria:** Every PR against the demo application shows SAST, SCA, IaC scan, and policy-check results inline. Every merge produces a signed, scanned, SBOM-attested image per service. The admission policy provably rejects an unsigned or failed-scan image and admits a properly built one. `kube-bench` runs on a schedule with results retained over time. A deliberate shell-spawn test triggers a Falco alert that reaches the security on-call channel. The centralized security log view can answer a real investigative query end to end. Both runbooks exist, have been rehearsed at least once against the real system, and the compliance evidence table accurately reflects the pipeline that was actually built — not an aspirational one.

---

## Summary

These four projects escalate deliberately: Project 1 proves the cheapest, highest-leverage shift-left controls (SAST, SCA, secret detection) work on a real pipeline; Project 2 extends that trust from source code to the container image itself, provable end to end with a real signature verification; Project 3 extends it further into the running cluster and the pipeline's own credentials, closing the gap between "the image is trustworthy" and "the runtime environment executing it is trustworthy too"; and Project 4 assembles every one of those controls, plus the compliance and incident-response work from later chapters, into one coherent, evidence-producing platform.

| Project | Level | Approx Time | Key Skills |
|---------|-------|-------------|------------|
| 1 — Secure a CI Pipeline | Beginner | 3–4 hours | Semgrep SAST with PR annotations, Dependabot/Renovate config, `gitleaks` pre-commit hook + CI backstop |
| 2 — Container and Supply Chain Security | Intermediate | 5–7 hours | Trivy build-blocking scan, `syft` SBOM generation, `cosign` keyless signing, Kyverno image-verification admission policy |
| 3 — Kubernetes and Pipeline Hardening | Advanced | 8–10 hours | `kube-bench` CIS remediation, Falco custom rules, Alertmanager-routed security alerts, OIDC-based CI cloud authentication |
| 4 — Full DevSecOps Platform | Advanced/Capstone | 16–20 hours | End-to-end PR-to-runtime pipeline gating, IaC policy-as-code, SIEM-style log centralization, incident response runbooks, compliance evidence mapping |

Project 4 is the natural "final exam" not just for this course but for the entire 11-course roadmap — it is the one project that cannot be completed without genuine, working knowledge of Docker, CI/CD, AWS, Terraform, both Kubernetes courses, Monitoring & Logging, and everything in this Security course at once, and it is exactly the kind of artifact that turns a resume line into a concrete, defensible portfolio piece in an interview.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./15-common-mistakes.md">← Previous: Common Mistakes and Pitfalls</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./17-interview-preparation.md">Next: Interview Preparation →</a>
</div>
