# Chapter 15 — Common Mistakes and Pitfalls

## Learning Objectives

By the end of this chapter, you will be able to:

- Identify the most common DevSecOps, application-security, and supply-chain mistakes by recognizing their symptoms
- Understand the misunderstanding or organizational pressure that leads to each mistake
- Apply the correct pattern from earlier chapters of this course immediately, without having to look it up
- Recognize these mistakes in a pipeline, manifest, or process review before they cause a real incident or a false sense of security

---

## How to Read This Chapter

Each mistake is presented with four parts:

1. **The wrong pattern** — a config, workflow, or organizational habit you will encounter in the wild
2. **Why it happens** — the misunderstanding or pressure that leads to it
3. **The correct fix** — a drop-in replacement, pointing back to the chapter that covers it in depth
4. **Impact / Prevention** — what breaks, and how to stop it happening again

This chapter is strictly about the **DevSecOps, application-security, and software-supply-chain domain** — SAST/SCA/DAST, secrets, container and artifact security, IaC security, CI/CD pipeline security, compliance, and incident response. If you're looking for app-deployment mistakes (missing probes, `:latest` tags, no resource limits), those are covered in Kubernetes Basics, Chapter 16. If you're looking for platform/cluster-admin mistakes (broad RBAC grants, unverified NetworkPolicies, premature multi-cluster adoption), those are covered in Advanced Kubernetes, Chapter 15. If you're looking for observability mistakes (cardinality explosions, alert fatigue, unstructured logging), those are covered in Monitoring & Logging, Chapter 15. This chapter assumes you already know and avoid all three of those lists, and covers what goes wrong specifically in how a team builds and operates security into its software supply chain.

---

## Mistake 1: Treating a Passing Scan as Proof of "Secure"

```text
# WRONG — the team's mental model:
"SAST passed, SCA passed, DAST passed. We're secure. Ship it."
```

**Why it happens:** A dashboard full of green checkmarks feels conclusive, and it's psychologically satisfying to treat "every scan we ran came back clean" as equivalent to "this application has no vulnerabilities." The language security tools use in their own UIs — "no issues found," "scan passed" — actively encourages this reading.

**The correct fix:** Read every scan result as "no *known* issues found by *this specific tool*, looking for *the specific class of issue it's designed to find*" — never as "secure," full stop (Chapters 4–8). A SAST tool finds the patterns it has rules for; it says nothing about a logic flaw or a vulnerability class outside its rule coverage. An SCA tool finds known CVEs in its vulnerability database; it says nothing about a zero-day or a vulnerability disclosed an hour after the scan ran. A DAST tool finds what it could reach and successfully exploit against the specific target configuration it was pointed at; it says nothing about a code path it never triggered.

```text
# RIGHT — the team's mental model:
"SAST, SCA, and DAST all passed against their respective known-issue
databases and rule sets. This reduces risk meaningfully and is
necessary. It is not proof of the absence of vulnerabilities outside
each tool's coverage — defense in depth (threat modeling, least
privilege, runtime detection) still matters precisely because of that."
```

**Impact:** Complacency compounds across every layer of the stack — a team that believes "scans passed" means "secure" stops threat modeling new features (Chapter 2), stops maintaining runtime detection as a serious priority (Chapter 10) because "we'd have caught it earlier in the pipeline," and is genuinely surprised when a real incident occurs despite a clean scan history. The incident, when it happens, is often in exactly the class of issue no deployed tool was designed to catch.

**Prevention:** Explicitly frame every scan result, in dashboards and in team communication, using the precise "no known issues found by this tool" language rather than "secure" or "clean." Maintain defense in depth (threat modeling, runtime detection, least privilege) as standing practice regardless of how clean recent scans look, because it is precisely the layer that exists for what scanning didn't catch.

---

## Mistake 2: Enabling Every Default SAST/Policy Rule at Once

```yaml
# WRONG — the full default ruleset, thousands of rules, turned on
# for a team's very first week using the tool
rules:
  - p/default    # every community rule, including highly stylistic,
                  # low-confidence, and framework-mismatched ones
```

**Why it happens:** More rules sounds like more coverage, and it feels like leaving security value on the table to enable anything less than the tool's full default rule set on day one. Nobody sets out to overwhelm the team — the instinct is simply "why would we turn off a security check."

**The correct fix:** Start with a narrow, high-confidence, security-focused rule subset, and expand deliberately over time as the team demonstrates it trusts and acts on the tool's output (Chapter 4, Chapter 14):

```yaml
rules:
  - p/security-audit   # curated, high-signal, low false-positive rate
  - p/secrets           # curated, high-signal
# expand later, once findings from the above are consistently
# triaged and fixed rather than accumulating unread
```

**Impact:** A flood of low-confidence findings in a team's first week — many stylistic, many false positives, many about frameworks the codebase doesn't even use — teaches developers within days that the tool's output isn't worth reading closely. This is a security-tool-specific instance of alert fatigue, distinct from Monitoring & Logging's observability alert fatigue: the failure mode here is developers learning to click "dismiss" on SAST findings in their PR without reading them, or a team disabling the check in CI entirely after enough complaints, rather than an on-call engineer ignoring a paging alert. Either way, the tool is effectively abandoned within weeks of being adopted with the best intentions.

**Prevention:** Treat rule-set curation as an ongoing responsibility of whoever owns the SAST/policy tool, not a one-time "turn it on" task. Review the finding-to-fix ratio periodically — a rule with a very high dismiss/ignore rate relative to genuine fixes is a strong candidate for tightening or disabling, the same way Monitoring Chapter 15 recommends pruning stale, unhelpful alerting rules.

---

## Mistake 3: Removing a Secret From the Latest Commit Without Rotating It

```bash
# WRONG — the team's reaction on discovering a committed secret
git rm .env
git commit -m "Remove accidentally committed secret"
git push
# considered the incident closed
```

**Why it happens:** The secret is no longer visible in the latest commit or in `git log` at a casual glance, and it *feels* resolved — the file is gone, the diff shows it removed, the PR that introduced it might even get force-pushed over or squashed. This creates a strong, entirely wrong intuition that the secret is no longer exposed.

**The correct fix:** Treat any secret that was ever committed — even briefly, even in a since-deleted or since-rewritten commit — as compromised and rotate it immediately (Chapter 3). Git history, once pushed to a shared remote, cannot be reliably un-exposed: the object may already be cached by a CI runner, cloned by a collaborator, indexed by a code-scanning bot, or retrievable via GitHub's own reflog/fork mechanisms even after a force-push:

```bash
# The secret is compromised the moment it's pushed, regardless of
# what happens to the commit afterward — rotation is mandatory
vault write database/rotate-root/production-db
# THEN, separately and only after rotation, worry about scrubbing
# history (git-filter-repo / BFG) as cleanup, not as the fix itself
```

**Impact:** A "cleaned up" repository with an un-rotated secret still buried somewhere in its object history, a fork, or a CI cache provides zero actual protection — anyone who cloned the repository before the cleanup, or who has access to a cached build artifact, still has the original valid credential. This is one of the most common and most dangerous misunderstandings in secrets management precisely because the visible symptom (the secret is gone from `git log`) so convincingly suggests the problem is fixed.

**Prevention:** Make secret rotation the mandatory, non-negotiable first response the instant a leak is detected — by `gitleaks` in CI (Chapter 3) or by manual discovery — before any git-history cleanup is even attempted. Treat "we force-pushed over it" and "the secret is no longer exposed" as two entirely unrelated claims that must never be conflated.

---

## Mistake 4: Not Pinning or Scoping Package Registries, Enabling Dependency Confusion

```json
// WRONG — an internal package name with no registry scoping,
// resolvable from either the internal registry or the public
// npm registry, whichever responds with a higher version number
{
  "dependencies": {
    "@company-internal/auth-utils": "^2.1.0"
  }
}
```

**Why it happens:** Internal package names often look like ordinary package names, and package managers resolve by name and version across whatever registries are configured, without inherently knowing "this specific package should only ever come from our private registry." Registry configuration that doesn't scope internal package names explicitly leaves this ambiguity open by default.

**The correct fix:** Explicitly scope internal package namespaces to your private registry, and pin registries per scope, so a public registry can never be an authoritative source for an internal package name (Chapter 7):

```
# .npmrc — explicit scope-to-registry pinning
@company-internal:registry=https://npm.internal.company.com/
# any package under @company-internal can ONLY resolve from this
# registry, regardless of what a public registry might also offer
# under a matching name and a higher version number
```

**Impact:** A **dependency confusion** attack publishes a malicious package to the public registry using the exact same name as an internal package, with an artificially high version number — and an unscoped build resolves the attacker's public package instead of the legitimate internal one, because most package managers default to preferring the highest available version across all configured sources. The malicious package's install script then runs with the same privileges as a normal dependency install, inside your build environment — a real, publicly documented attack class that has compromised major organizations.

**Prevention:** Scope every internal package namespace to its private registry explicitly, as a mandatory part of registry configuration, and verify this scoping during onboarding for any new internal package namespace — never let it default to "resolvable from anywhere."

---

## Mistake 5: Signing Artifacts With Cosign but Never Enforcing Verification at Deploy Time

```yaml
# WRONG — every image is dutifully signed in CI...
- run: cosign sign --key cosign.key $IMAGE:$TAG
# ...but nothing in the cluster ever checks the signature before
# running the image. No admission policy verifies it at all.
```

**Why it happens:** Signing feels like the security win, and it's genuinely the more visible, more discussed half of the artifact-integrity story (Chapters 6–7). Wiring up admission-time verification is a separate piece of infrastructure (Kyverno's `verifyImages` or an equivalent policy engine), configured in a completely different system (the cluster) than where signing happens (the CI pipeline) — and it's easy for the project to be considered "done" once signing itself works.

**The correct fix:** Enforce signature verification at admission time, so an unsigned or invalidly-signed image is rejected before it ever runs, not just optionally checked by whoever remembers to run `cosign verify` manually (Chapter 6, Chapter 10):

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-cosign-signature
      match:
        any:
          - resources: { kinds: ["Pod"] }
      verifyImages:
        - imageReferences: ["registry.company.com/*"]
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      ...
                      -----END PUBLIC KEY-----
```

**Impact:** Unenforced signing provides **zero actual protection** against the exact threat it exists to prevent — a substituted or tampered image can still be deployed without issue, because nothing in the deployment path ever checks the signature. This is a false sense of supply-chain security stated as plainly as it can be: the organization can accurately say "we sign our images," and that statement is true, and it protects nothing, because a signature nobody verifies is equivalent to no signature at all from a runtime-security standpoint. This mirrors Chapter 12's "policy documented but not technically enforced" compliance failure — signing without verification is that same gap, specific to the supply chain.

**Prevention:** Treat "signing" and "enforced verification" as two separate, both-mandatory halves of one control, and don't consider artifact integrity "done" until an admission-time test confirms an unsigned or badly-signed image is actually, empirically rejected — not just that the policy YAML exists.

---

## Mistake 6: Running DAST Only Against a Pristine Staging Environment

```yaml
# WRONG — DAST always targets a staging environment configured
# with debug auth bypassed, feature flags all enabled for testing
# convenience, and none of production's actual hardening applied
- name: DAST scan
  run: zap-baseline.py -t https://staging.internal.company.com
```

**Why it happens:** Staging is the obvious, convenient, always-available target — it's meant to mirror production, it's safe to attack without customer impact, and pointing a scanner at it feels like the responsible default. The gap between "staging" and "production, as actually configured" is often invisible to whoever wires up the DAST job, especially if staging and production configuration have quietly drifted apart over time.

**The correct fix:** Periodically validate that staging's security-relevant configuration (authentication settings, feature flags, WAF rules, rate limiting) actually matches production, or run scans against a production-like environment built specifically to mirror production's real configuration (Chapter 8) — not whatever staging happens to look like today:

```text
Checklist before trusting a staging DAST result as representative
of production risk:
[ ] Authentication/authorization configuration matches production
    (no debug bypass, no test-only auth shortcut)
[ ] Feature flags match production's default state, not "all on
    for testing convenience"
[ ] Any WAF/rate-limiting/reverse-proxy layer in front of production
    is also present (or explicitly accounted for) in staging
```

**Impact:** A vulnerability that only exists because of a production-specific configuration difference — a feature flag enabled in prod but not staging, an authentication path that behaves differently once a production-only third-party integration is live — is invisible to a DAST scan run exclusively against a staging environment that quietly diverged from production months ago. The scan reports clean, the team trusts that result, and the actual production attack surface goes untested.

**Prevention:** Audit staging-vs-production configuration drift on a recurring schedule specifically for the settings that affect security posture (auth, flags, network-layer protections), and treat any confirmed drift as a defect to fix, not an acceptable cost of having a staging environment at all.

---

## Mistake 7: Granting CI/CD Pipeline Credentials Broad, Long-Lived Cloud Access "to Avoid Headaches"

```yaml
# WRONG — a single, broad, long-lived cloud access key stored as a
# CI secret, used by every pipeline for every job, "so nobody has
# to deal with permission errors"
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  # this key has near-account-wide permissions and never expires
```

**Why it happens:** Debugging a scoped credential's permission errors mid-pipeline is genuinely annoying, especially under deadline pressure, and a broad, long-lived key makes every future permission error simply disappear — the same underlying temptation, in a different system, as granting `cluster-admin` to unblock a Kubernetes deploy. It's meant to be temporary and "tightened up properly later," and it never is.

**The correct fix:** Adopt OIDC federation so the pipeline receives a short-lived, narrowly-scoped credential minted for the duration of a single job run, with no long-lived secret stored anywhere (Chapter 11):

```yaml
permissions:
  id-token: write   # required for OIDC — no stored credential at all
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/ci-deploy-role-scoped
      aws-region: us-east-1
      # the assumed role is scoped to exactly what THIS pipeline needs,
      # and the resulting credential expires when the job ends
```

**Impact:** This recreates, in the cloud-credential domain, precisely the risk OIDC and short-lived credentials exist to eliminate: a compromised CI runner, a leaked secret from a misconfigured log output, or a malicious pull request that manages to exfiltrate the stored key gives an attacker standing, long-lived, broad cloud access — not access limited to one job's duration and one job's actual needs. This is a distinct failure from Advanced Kubernetes' "cluster-admin to a CI ServiceAccount" mistake: that one concerns Kubernetes RBAC inside a cluster; this one concerns cloud provider IAM and container registry credentials reachable from outside the cluster entirely, and it's the specific gap Chapter 11's OIDC pattern was built to close.

**Prevention:** Treat any long-lived cloud access key stored as a CI secret as a finding requiring remediation, the same way Advanced Kubernetes treats a `cluster-admin` binding. Migrate to OIDC federation as the default pattern for any new pipeline, and set a deadline for migrating existing pipelines off stored long-lived keys rather than letting "the old pipelines still use keys" persist indefinitely as an exception nobody circles back to.

---

## Mistake 8: Generating an SBOM Once at Release, Never Regenerated or Diffed

```text
# WRONG — an SBOM is generated once, attached to the v2.3.0 release
# artifact, and filed away as a compliance checkbox — never looked
# at again, never regenerated for v2.3.1, v2.4.0, or beyond
release-v2.3.0/
  └── sbom.spdx.json   # generated once, six months ago, never revisited
```

**Why it happens:** An SBOM is often adopted specifically to satisfy a compliance requirement or a customer contractual ask ("provide an SBOM for your product"), and once that specific requirement is satisfied for a specific release, the task feels complete — the checkbox is checked, the audit or customer request is closed.

**The correct fix:** Regenerate the SBOM on every build, and continuously diff its contents against new vulnerability disclosures, not just at release time (Chapter 7):

```yaml
# Every build, not just tagged releases — and diffed against
# newly disclosed CVEs on a recurring schedule, not just at
# generation time
- run: syft . -o spdx-json > sbom.spdx.json
- run: grype sbom:sbom.spdx.json --fail-on high
# a nightly job re-checks EVERY release's stored SBOM against
# today's vulnerability database, catching newly disclosed CVEs
# against dependencies that haven't changed at all
```

**Impact:** A vulnerability disclosed against a dependency your application has depended on, unchanged, for the past year is invisible to an SBOM generated once at last year's release — the SBOM accurately reflects what was shipped, but nobody is re-checking that static snapshot against today's vulnerability landscape. The organization has produced a compliance artifact without the actual operational capability (continuous vulnerability awareness) the SBOM was meant to enable in the first place.

**Prevention:** Wire SBOM generation into every build as routine pipeline output (Chapter 14), and run a separate, recurring job that diffs previously-generated SBOMs against the current vulnerability database — treating "a new CVE was disclosed against something we ship" as an event that should surface automatically, not something discovered only the next time someone happens to regenerate an SBOM by hand.

---

## Mistake 9: An Incident Response Runbook That Has Never Been Rehearsed

```text
# WRONG — a detailed, well-written incident response runbook exists,
# has never been walked through in a tabletop exercise, and is
# opened for the very first time during an actual live incident
```

**Why it happens:** Writing the runbook feels like the completed deliverable — it exists, it's detailed, it's been reviewed and approved. Scheduling time for a tabletop exercise to actually rehearse it competes with every other item on a busy team's roadmap, and unlike the runbook itself, a rehearsal produces no visible artifact to point to as "done."

**The correct fix:** Rehearse every incident response runbook via a tabletop exercise on a recurring schedule, before it's ever needed for real (Chapter 13):

```text
Quarterly tabletop cadence, per major runbook:
[ ] Walk through the exact scenario the runbook covers
[ ] Confirm every named on-call contact still works there and still
    has the access the runbook assumes
[ ] Confirm every command/YAML snippet in the runbook actually still
    applies to the current infrastructure
[ ] Update the runbook immediately with anything the exercise found stale
```

**Impact:** A runbook discovers its own gaps at the single worst possible moment — during a real incident — rather than during a scheduled, low-stakes drill. A named on-call contact who left the company eight months ago, a `kubectl` command referencing a NetworkPolicy naming convention the platform team changed last quarter, or an assumed access level that was revoked in a routine RBAC audit are all exactly the kind of small, easily-fixed staleness that a tabletop exercise catches cheaply and a real incident discovers expensively, under pressure, while the clock on actual damage is running.

**Prevention:** Put a tabletop exercise for every major incident response runbook on the same recurring operational calendar as DR game days (Advanced Kubernetes Chapter 10) — quarterly, at minimum — and treat a runbook that hasn't been rehearsed within that window as stale and unverified, regardless of how well-written it reads on paper.

---

## Mistake 10: Blame-Focused Postmortems That Discourage Fast Reporting

```text
# WRONG — an engineer who accidentally committed a secret is called
# out by name in a wide postmortem distribution, or faces a formal
# performance conversation about the mistake, creating an organizational
# incentive to hide or quietly self-correct future mistakes instead
# of reporting them immediately
```

**Why it happens:** Holding someone accountable feels, intuitively, like the responsible organizational response to a mistake — especially a mistake with real consequences. It's easy to conflate "taking the incident seriously" with "identifying and addressing the person who made the error," without noticing the second framing actively undermines the organization's ability to respond quickly to the *next* mistake.

**The correct fix:** Run every postmortem — especially security postmortems — blamelessly, focused entirely on the systemic and process gap that allowed the mistake to have impact, never on the individual (Chapter 13):

```text
# RIGHT — the postmortem's central questions
"What made this mistake easy to make, and how do we make it
harder or lower-impact next time?" (e.g., "we didn't have a
pre-commit secret scanner" or "our rotation runbook wasn't
clear about who has authority to rotate a production credential
on a weekend")

NOT: "Who committed the secret, and what's the consequence for them?"
```

**Impact:** A blame-focused culture creates a direct, rational incentive for an engineer who makes a mistake — accidentally committing a secret, for example — to try to quietly fix it themselves first rather than immediately reporting it to whoever can rotate the affected credential fastest. That delay is precisely the window during which a leaked secret does real damage (Chapter 3's "rotate immediately" principle only works if the report happens immediately). The organizational cost of a blame-focused culture isn't just morale — it's measurably slower incident response, specifically for the class of incidents that start with an individual's honest mistake.

**Prevention:** Establish and communicate blameless postmortem norms explicitly, before an incident forces the question, and hold leadership accountable to actually practicing them (a blameless postmortem policy that leadership quietly ignores the first time a mistake is expensive enough is no policy at all). Track postmortem action items as process/tooling fixes, never as notes in an individual's performance record.

---

## Mistake 11: Compliance as a Checkbox Disconnected From Real Technical Controls

```text
# WRONG — a written policy document states: "Access to production
# systems requires quarterly review and approval." No automated
# access review actually runs. No technical enforcement exists.
# The document alone is presented to an auditor as evidence.
```

**Why it happens:** Writing a policy document is fast, produces an artifact auditors visibly ask for, and can be completed without touching any actual system. Building the real technical control the document describes is slower, requires engineering time, and doesn't feel like "compliance work" to whoever owns the audit relationship — so the document sometimes gets written and treated as sufficient on its own, especially under time pressure ahead of an audit deadline.

**The correct fix:** Ensure every compliance control documented in policy has a corresponding, actually-enforced technical mechanism behind it, and generate audit evidence directly from that mechanism running (Chapter 12, Chapter 14):

```bash
# RIGHT — the policy is backed by an actual automated process that
# produces its own evidence, rather than existing only on paper
# A scheduled job that actually performs, and logs, the quarterly
# access review the policy document describes
0 0 1 */3 * /opt/scripts/quarterly-access-review.sh >> /var/log/access-review.log
# audit evidence = the log itself, not a manually-assembled claim
```

**Impact:** This fails in one of two ways, both bad. It can fail the very first time an auditor asks for supporting evidence of the described control actually operating — a policy document with no corresponding log, ticket, or system record behind it is not evidence, it's an assertion. Or, worse, the audit *passes* anyway (a checklist-style audit sometimes accepts the policy document at face value) while providing zero real security benefit, because the control it describes was never technically enforced at all — the organization is now compliant on paper and exposed in reality, which is the most dangerous version of this mistake because nothing prompts anyone to notice.

**Prevention:** Require every compliance control in a policy document to name the specific technical mechanism that enforces it and the specific automated evidence it produces, as a condition of the policy being considered complete — a policy line item with no corresponding system behind it is a gap to close, not a document to file away.

---

## Mistake 12: Running IaC Security Scanning Only Locally, Never as an Enforced CI Gate

```bash
# WRONG — tfsec/Checkov exist only as a `make security-check` target
# a developer can run manually, entirely optional, never wired into
# the CI pipeline that actually gates merges
make security-check   # nobody is required to run this before merging
```

**Why it happens:** Adding a `make` target is a fast first step and demonstrates the tool works — it feels like real progress. Wiring it into CI as a blocking gate is a slightly bigger step (deciding on failure thresholds, handling existing violations in the codebase, getting team buy-in for something that can now block a merge), and it's easy for the project to stall at "the tool exists and works" without ever reaching "the tool is actually enforced."

**The correct fix:** Wire IaC scanning into the CI pipeline as a required, blocking check on every pull request touching infrastructure code, not an optional local target (Chapter 9):

```yaml
# CI pipeline — blocking, not optional
- name: IaC security scan
  run: tfsec . --minimum-severity HIGH
  # a non-zero exit code here fails the CI job and blocks the merge,
  # exactly like a failing test suite would
```

**Impact:** A manual, optional check is, in practice, a check that gets skipped precisely when it matters most — under deadline pressure, during an urgent hotfix, or simply because a developer working on infrastructure code that day never knew the `make` target existed. Misconfigurations that `tfsec`/Checkov would have caught (a publicly readable S3 bucket, an overly permissive security group) reach production not because the tooling failed, but because nothing ever required it to run.

**Prevention:** Treat "wired into CI as a required, blocking gate" as the actual definition of "IaC scanning is in place" — a locally-runnable tool that isn't enforced anywhere doesn't meet that bar, regardless of how well it works when someone remembers to invoke it.

---

## Mistake 13: Fixing a Vulnerable Direct Dependency Without Checking Transitive Paths

```text
# WRONG — package.json is updated to bump a directly-declared
# dependency past its vulnerable version, and the fix is
# considered complete
"dependencies": {
  "lodash": "^4.17.21"   # bumped past the vulnerable version — done, right?
}
```

**Why it happens:** SCA tooling (Chapter 5) typically flags the vulnerable package by name and version, and the most visible, most obvious fix is bumping wherever that package name appears in your own direct dependency list. It's easy to stop there, because the direct fix appears to resolve the specific finding the scanner reported.

**The correct fix:** Check the full dependency tree, not just direct dependencies, for every path that still pulls in the vulnerable version transitively through a different package (Chapter 5):

```bash
# Confirm no OTHER path in the dependency tree still resolves to
# the vulnerable version, before considering this fixed
npm ls lodash
# lodash@4.17.21           <- your direct fix, good
# └─┬ some-other-package@1.2.0
#   └── lodash@4.17.15     <- still vulnerable, pulled in transitively
#       through a completely different package
```

**Impact:** The application remains exposed to the exact same CVE through the second, unfixed path — the vulnerable code is still present and still reachable in the built artifact, even though the direct dependency declaration looks fixed. An SCA re-scan may even still flag the finding, and a team that only checked the direct dependency is genuinely confused about why "the fix" didn't resolve the scanner's alert.

**Prevention:** Always run a full dependency-tree query (`npm ls`, `pip show --files`, `mvn dependency:tree`, or your ecosystem's equivalent) for the vulnerable package name after applying a fix, confirming zero remaining resolved paths to the vulnerable version — not just that your own direct declaration was updated. Where a transitive fix isn't directly controllable, use your package manager's override/resolution mechanism to force the entire tree onto a patched version.

---

## Mistake 14: No Runtime Detection at All — Betting Everything on Preventive Controls

```text
# WRONG — the organization's entire security posture is RBAC,
# admission control, and scanning (all preventive, all pre-deploy).
# Nothing watches what's actually happening inside a running
# container or Pod. If an attacker gets past every preventive
# layer, nothing in the stack is positioned to notice.
```

**Why it happens:** Preventive controls are where most of this course's earlier chapters focus, and for good reason — they're cheaper to reason about, they stop problems before they exist, and each one produces a clear pass/fail signal a team can point to. Runtime detection (Chapter 10's Falco) requires committing to operate an entirely different kind of system — one that watches live behavior continuously rather than gating a one-time check — and it's easy for a security roadmap to run out of time or budget before reaching that step, especially once the preventive controls all look solid.

**The correct fix:** Deploy runtime detection (Falco or an equivalent) as a mandatory, not optional, layer of the security stack — explicitly for the case where every preventive control has already failed (Chapter 10):

```yaml
# A default Falco rule that requires no custom tuning to provide
# real, immediate value against exactly this gap
- rule: Terminal shell in container
  desc: A shell was spawned inside a container — often the first
        observable sign of a successful exploit
  condition: spawned_process and container and shell_procs
  output: "Shell spawned in container (user=%user.name container=%container.name)"
  priority: WARNING
```

**Impact:** Preventive controls are necessarily imperfect — a zero-day vulnerability, a novel exploitation technique, or a simple misconfiguration that slipped past every scan and every admission policy will, eventually, get past all of them for some organization, at some point. With zero runtime detection, an attacker who succeeds at that has essentially unlimited time inside the environment before discovery — which typically arrives, in this failure mode, only through unrelated means: a customer noticing a data leak, a cloud bill anomaly, or an entirely separate audit months later. The gap between compromise and discovery, in an environment with no runtime detection, is the single biggest driver of how much damage a successful breach ultimately causes.

**Prevention:** Treat runtime detection as a required layer of the security stack from the start of any DevSecOps program, not a "nice to have" reached only after every preventive control is considered complete — the entire justification for defense in depth (Mistake 1's framing) is that preventive controls will eventually fail, and something has to be watching for that specific moment.

---

## Summary

| # | Mistake | Key Fix |
|---|---------|---------|
| 1 | Treating a passing scan as proof of "secure" | Read results as "no known issues found by this tool" — maintain defense in depth |
| 2 | Enabling every default SAST/policy rule at once | Start with a narrow, high-confidence rule set; expand deliberately |
| 3 | Removing a secret from the latest commit, not rotating it | Rotate immediately — git history cannot be reliably un-exposed |
| 4 | Unscoped package registries enabling dependency confusion | Pin internal package namespaces explicitly to your private registry |
| 5 | Signing artifacts but never enforcing verification | Enforce signature verification at admission time — unenforced signing protects nothing |
| 6 | DAST run only against a drifted staging environment | Audit staging-vs-production security config drift regularly |
| 7 | Broad, long-lived cloud credentials for CI "to avoid headaches" | Adopt OIDC federation for short-lived, scoped pipeline credentials |
| 8 | SBOM generated once at release, never revisited | Regenerate every build; continuously diff against new CVE disclosures |
| 9 | An IR runbook never rehearsed in a tabletop exercise | Schedule recurring tabletop exercises for every major runbook |
| 10 | Blame-focused postmortems discouraging fast reporting | Run every postmortem blamelessly, focused on systemic fixes |
| 11 | Compliance documented but not technically enforced | Every policy line item needs a real, evidence-producing mechanism |
| 12 | IaC scanning only run manually, never an enforced CI gate | Wire scanning into CI as a required, blocking check |
| 13 | Fixing a direct dependency, ignoring transitive paths | Check the full dependency tree for every path to the vulnerable version |
| 14 | No runtime detection at all | Deploy Falco (or equivalent) as a mandatory defense-in-depth layer |

---

## Knowledge Check

1. Why is "SAST/SCA/DAST all passed" not equivalent to "this application is secure," and what should a team maintain regardless of how clean recent scans look?
2. Explain the parallel and the difference between this chapter's SAST rule-curation mistake (Mistake 2) and Monitoring & Logging's observability alert-fatigue mistakes.
3. A team force-pushes to remove a commit containing a leaked API key and considers the incident closed. What is wrong with this response, and what should have happened instead?
4. Why does signing an artifact with cosign provide zero real protection if nothing verifies that signature at deploy time?
5. What is the mechanical difference between Mistake 7 (broad long-lived CI cloud credentials) and Advanced Kubernetes' "cluster-admin to a CI ServiceAccount" mistake?
6. Why is a compliance policy document with no corresponding enforced technical control worse than having no written policy at all, in the worst case?

---

## Hands-On Exercise

**Find and Fix the Broken DevSecOps Pipeline**

The configuration and process set below contains at least 5 of the mistakes covered in this chapter. Find every one of them, explain why each is harmful, and rewrite the configuration correctly using the patterns from this course.

```yaml
# 1) CI/CD pipeline credentials
name: Deploy to Production
on: [push]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          # this key has near-account-wide S3, ECR, and EC2 permissions
          # and has not been rotated since it was created two years ago
        run: aws configure set ...
      - name: Build and push image
        run: |
          docker build -t registry.company.com/checkout:${{ github.sha }} .
          docker push registry.company.com/checkout:${{ github.sha }}
      - name: Sign image
        run: cosign sign --key cosign.key registry.company.com/checkout:${{ github.sha }}
          # signed here — but no admission policy anywhere in the
          # target cluster verifies this signature before running the image
      - name: Deploy
        run: kubectl set image deployment/checkout checkout=registry.company.com/checkout:${{ github.sha }}
```

```text
# 2) Incident response documentation
security-runbooks/leaked-credential-response.md
  Last updated: 14 months ago
  Never rehearsed in a tabletop exercise
  Lists "Priya (on-call security lead)" as the primary contact —
  Priya left the company 9 months ago
```

```text
# 3) Git history
$ git log --oneline -5
a1b2c3d Remove accidentally committed AWS key (force-pushed over previous commit)
# the exposed AWS key was never rotated — the team considers this
# resolved because `git log` no longer shows it in the latest commits
```

```makefile
# 4) IaC security scanning
security-check:
	tfsec ./terraform
	checkov -d ./terraform
# this Makefile target exists and works correctly when run manually;
# it is not referenced anywhere in the CI pipeline configuration
```

```text
# 5) Compliance policy document (SOC 2 evidence package)
"Control CC6.1: Production access is reviewed quarterly by the
security team, and access is revoked for any account without a
current business justification."
# No automated review process exists. No ticket, log, or system
# record of any quarterly review has ever been produced. The
# document itself was presented to the auditor as the sole evidence.
```

Steps:

1. List every mistake you can find across all five artifacts (aim for at least 5). Hint: look closely at the credential type and lifetime in item 1, cross-reference the cosign step against what actually happens at deploy time, check whether "removed from git log" and "rotated" are the same thing in item 3, and check whether item 4's Makefile target is reachable from anywhere that actually blocks a merge.
2. Rewrite item 1's credential handling using OIDC federation, and describe the additional Kyverno (or equivalent) policy needed to make the `cosign sign` step actually mean something at deploy time.
3. Describe the tabletop exercise you'd schedule to catch item 2's problems before a real incident does, and what specifically it should verify.
4. For item 3, write the specific remediation step that should have happened immediately upon discovering the leaked key, before any git-history cleanup was even considered.
5. Rewrite item 4's CI configuration so the `tfsec`/Checkov checks are a required, blocking gate on every pull request touching Terraform code.
6. For item 5, describe the automated mechanism and the resulting evidence artifact that would make this control's compliance claim actually true, not just documented.

---

## Further Reading

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Dependency Confusion Advisory](https://owasp.org/www-community/Dependency_Confusion)
- [Sigstore / Cosign Documentation — Verifying Signatures](https://docs.sigstore.dev/cosign/verifying/)
- [CISA — Software Bill of Materials (SBOM) Resources](https://www.cisa.gov/sbom)
- [Google SRE Workbook — Postmortem Culture](https://sre.google/sre-book/postmortem-culture/)
- [NIST SP 800-61 Rev. 2 — Computer Security Incident Handling Guide](https://csrc.nist.gov/pubs/sp/800/61/r2/final)
- [Falco Documentation](https://falco.org/docs/)

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./14-best-practices.md">← Previous: Best Practices</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./16-projects.md">Next: Hands-On Projects →</a>
</div>
