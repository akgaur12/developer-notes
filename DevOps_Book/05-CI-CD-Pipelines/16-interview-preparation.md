# Chapter 16 — Interview Preparation

## 16.1 Foundational Questions

**Q: What is the difference between CI and CD?**
> CI (Continuous Integration) is the practice of automating build and test runs on every code commit — it answers "does this code work?". CD stands for either Continuous Delivery (every passing build is deployable, but a human approves release) or Continuous Deployment (every passing build automatically deploys to production). CI is a prerequisite for CD. CI catches bugs in minutes; CD gets working code to users quickly.

**Q: Why is CI/CD important?**
> Without CI/CD: developers integrate code infrequently, causing painful merge conflicts ("integration hell"); deployments are manual, error-prone, and dreaded events. With CI/CD: small changes are tested continuously, bugs surface within minutes of introduction, and deployments are routine, automated, low-risk events. DORA research shows high-performing teams deploy 973x more frequently with 3x lower change failure rates.

**Q: What is a GitHub Actions workflow and how does it work?**
> A workflow is a YAML file in `.github/workflows/`. It defines triggers (events like push, PR, schedule), jobs (logical units of work that run on a runner), and steps (individual commands or action invocations within a job). When a trigger fires, GitHub queues the jobs, allocates runners, and executes steps sequentially within each job. Jobs run in parallel by default unless you use `needs:` to create dependencies.

**Q: How do you pass data between jobs in GitHub Actions?**
> Using job outputs: a step writes to `$GITHUB_OUTPUT` (e.g., `echo "sha=${GITHUB_SHA::8}" >> $GITHUB_OUTPUT`), the job declares it as an output, and downstream jobs access it with `${{ needs.job-name.outputs.output-name }}`. For files/artifacts, use `actions/upload-artifact` and `actions/download-artifact`.

**Q: What is OIDC in the context of CI/CD?**
> OpenID Connect (OIDC) allows CI/CD pipelines to authenticate to cloud providers (AWS, GCP, Azure) without storing long-lived credentials. GitHub generates a short-lived JWT token signed by GitHub's OIDC provider; the cloud provider is configured to trust it; the pipeline exchanges the token for short-lived cloud credentials. No secrets to rotate, no credentials to leak.

---

## 16.2 Architecture Questions

**Q: How would you design a CI/CD pipeline for a microservices monorepo?**
```
Key considerations:
1. Change detection: only build services that changed (path filters or tools like nx/turborepo)
2. Parallel builds: use matrix strategy — all changed services build simultaneously
3. Shared cache: base images and dependencies cached across service builds
4. Independent deployment: each service deploys independently, not as a monolith
5. Integration tests: run after all services deploy to staging
6. GitOps: update manifests per-service; ArgoCD syncs each service independently
```

**Q: Explain blue/green vs canary deployment. When would you use each?**
> Blue/green: two full environments, switch all traffic instantly. Use when: rollback must be instant (financial, healthcare), deployment is risky, you can afford double infrastructure. Canary: route a small traffic percentage to the new version, gradually increase. Use when: you want to validate on real traffic before full rollout, you have good monitoring to detect degradation, gradual rollout is acceptable. Blue/green is all-or-nothing; canary limits blast radius but requires better observability.

**Q: How do you handle database migrations in a CD pipeline?**
> The expand/contract pattern: Phase 1 (expand) — add new columns as nullable with defaults, deploy old app + new app work; Phase 2 (migrate) — backfill data; Phase 3 (contract) — remove old columns after all old app instances are gone. Migrations run as a separate pre-deploy job. Never run destructive migrations while old app versions are still running.

---

## 16.3 Scenario Questions

**Scenario: "Your pipeline was 5 minutes, now it's 45 minutes. How do you fix it?"**
```
1. Profile: look at each job's duration in the GitHub Actions timeline UI
2. Common culprits:
   - Dependency install: add caching (saves 60-90s)
   - Docker build: add BuildKit GHA cache (saves 3-5 min)
   - Tests growing: parallelize with matrix sharding
   - New integration tests that could be mocked
   - Flaky tests causing retries
3. Measure before/after each change
4. Set timeout: jobs that hang silently kill pipeline velocity
Target: < 10 min for PR CI, < 15 min full pipeline
```

**Scenario: "A deploy to production broke the site. Walk me through your response."**
```
1. Rollback immediately (< 5 min): kubectl rollout undo / docker service rollback
2. Verify rollback: smoke test the previous version
3. Communicate: post in incident channel, update status page
4. Root cause: compare what changed (git diff vs previous deploy)
5. Fix in branch: don't push directly to main
6. Add regression test: ensure the bug can't be redeployed silently
7. Post-mortem: document timeline, root cause, prevention measures
Key: rollback first, investigate second. Never investigate while site is down.
```

**Scenario: "How would you secure a CI/CD pipeline?"**
```
1. Secrets: OIDC over long-lived credentials; Vault/SSM for complex secrets
2. Permissions: least privilege on each job (read vs write scopes)
3. Actions: pin to SHA, not tags; audit third-party actions
4. Branches: protected main branch; require PR review before merge
5. Environments: production requires approval + deployment branch restrictions
6. Scanning: SAST (CodeQL), dependency audit (Snyk), secret scanning (TruffleHog)
7. Image signing: Cosign; only deploy images your pipeline built and signed
8. Audit log: all deploys traceable to a commit, PR, and approver
```

---

## 16.4 Quick-Fire Questions

| Question | Answer |
|----------|--------|
| Default trigger for GitHub Actions? | Events defined in `on:` block |
| What is `actions/checkout` for? | Clones the repository onto the runner |
| Difference between `run` and `uses`? | `run` executes a shell command; `uses` calls a pre-built action |
| What does `needs:` do? | Creates a dependency — job waits for the listed jobs to succeed |
| What is a GitHub Environment? | A deployment target with protection rules and scoped secrets |
| What is a runner? | The machine/container that executes workflow jobs |
| How to cancel a running workflow? | `gh run cancel <run-id>` or via GitHub UI |
| What is a reusable workflow? | A workflow callable by other workflows via `workflow_call` |
| What does `if: always()` do? | Step runs regardless of previous step failures |
| What is a matrix build? | Running a job across multiple configurations in parallel |
| What is DORA? | Four key metrics: deployment frequency, lead time, failure rate, MTTR |
| What is a Jenkinsfile? | Groovy-based pipeline-as-code for Jenkins |

---

## 16.5 "Tell Me About a Pipeline You Built"

STAR format:
```
Situation: Startup with 8 developers, no CI/CD. Deployments were manual
           SSH + git pull on a production server. 2-3 hour "deploy days" monthly.

Task: Build a reliable pipeline that let us deploy safely multiple times per day.

Action:
1. Added GitHub Actions CI: lint + unit tests on every PR (<5 min)
2. Added Docker build pipeline with GHCR push on main
3. Set up branch protection: PRs required, CI must pass
4. Added Trivy scanning — caught a critical CVE in our base image immediately
5. Implemented blue/green deploy to our single AWS server with nginx upstream swap
6. Added Slack notifications for deploy start/success/failure
7. Wrote smoke test script checking 5 key endpoints post-deploy
8. Added automatic rollback: if smoke tests fail, flip nginx upstream back

Result: Deployment time went from 2 hours (manual) to 8 minutes (automated).
Deployment frequency went from monthly to 3–5 times per day.
Zero production incidents caused by bad deployments in 6 months post-implementation.
```

**Knowledge Check:**

1. A recruiter asks "What is the difference between Continuous Delivery and Continuous Deployment?" Write a one-paragraph answer in your own words.
2. What two things should you do immediately when a production deploy causes an outage?
3. Name three specific controls you would add to make a pipeline more secure.
4. Why is OIDC authentication better than storing a static AWS access key as a GitHub secret?
5. Your PR pipeline takes 22 minutes. List three specific things you would investigate first and why.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="15-projects.md">← Previous: Hands-On Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="17-course-summary.md">Next: Course Summary →</a>
</div>
