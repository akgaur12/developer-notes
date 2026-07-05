# Chapter 17 — Interview Preparation

**Learning Objectives**

By the end of this chapter you will be able to confidently answer DevSecOps/security-engineer-level questions spanning foundational concepts, architecture and internals, incident scenarios, and system design — and articulate your own DevSecOps experience in a structured way.

---

## 17.1 Foundational Questions

**Q: What is shift-left security and why does it matter?**
> Shift-left security means moving security checks as early as possible in the software lifecycle — into the IDE, the pre-commit hook, and the pull request — instead of running them only as a final gate right before release. It matters because the cost of fixing a vulnerability grows the later it's found: a SQL-injection pattern caught by SAST during code review costs a few minutes to fix before merge; the same pattern found by a penetration test after release costs an incident, a hotfix, and possibly a breach disclosure. Chapter 1 frames this as the core philosophy of the entire course — not a single tool, but the practice of running the same checks a dedicated security team would eventually run, automatically, in the same pipelines developers already use, so security becomes everyone's responsibility rather than one team's gate.

**Q: What's the difference between SAST and DAST?**
> SAST (Static Application Security Testing) analyzes source code, bytecode, or binaries without executing them, looking for known-vulnerable patterns like string-concatenated SQL queries, hardcoded secrets, or unsafe deserialization — it runs early (on every PR) and can point to an exact line of code, but it cannot see how the application actually behaves at runtime and tends to produce false positives on patterns that are technically flagged but not actually exploitable in context. DAST (Dynamic Application Security Testing) attacks a running instance of the application from the outside, the way a real attacker would — sending malformed requests, fuzzing inputs, probing authentication — and it catches issues SAST structurally cannot see, like a misconfigured server header, a broken authentication flow, or a business-logic flaw, at the cost of running later in the pipeline (it needs a deployed, running target) and not being able to point to a specific line of source code. Chapters 4 and 8 treat them as complementary, not competing — a mature pipeline runs both.

**Q: What is SCA and why does it matter for transitive dependencies?**
> SCA (Software Composition Analysis) scans a project's declared and resolved dependencies against known-vulnerability databases to find CVEs in third-party code the team didn't write itself. It matters specifically for transitive dependencies — the dependencies of your dependencies, often several layers deep and never directly declared in your own manifest — because most real-world vulnerable-dependency incidents (including large-scale ones like Log4Shell) were transitive: the vulnerable library wasn't something any engineer on the team consciously chose to add, it was pulled in silently by something else they did choose. Chapter 5's core point is that an SCA tool has to resolve the full dependency graph, not just the top-level manifest, or it will miss exactly the class of vulnerability most likely to actually bite a real organization.

**Q: What is an SBOM and what problem does it solve?**
> An SBOM (Software Bill of Materials) is a structured, machine-readable inventory of every component — direct and transitive dependency, base image layer, and often build tool — that went into producing a specific software artifact, typically in a standard format like CycloneDX or SPDX. It solves the "which of our 200 services are affected by this new CVE" problem: without an SBOM, answering that question means re-scanning every service from scratch and hoping the scan is current; with an SBOM generated and stored at build time for every artifact, the same question becomes a query against existing, static data — "which SBOMs list this exact vulnerable package version" — answerable in minutes instead of days. Chapter 7 frames the SBOM as the artifact that makes the difference between reactive scrambling and confident, fast incident response when a new CVE drops.

**Q: Explain the difference between authentication via a static API key vs. OIDC-based short-lived tokens for CI/CD.**
> A static API key or long-lived cloud credential (e.g., an `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` pair) is a permanent secret stored somewhere — a CI secret store, a config file — that remains valid indefinitely until someone manually rotates or revokes it; if it leaks, an attacker has standing access until that manual rotation happens, and in practice static credentials are rotated rarely because rotation is manual, disruptive work. OIDC-based authentication lets the CI system (e.g., GitHub Actions) mint a short-lived, cryptographically signed identity token scoped to the exact workflow, repository, and branch that's running, which the cloud provider's IAM trusts via a federated role and exchanges for temporary credentials that expire automatically, usually within an hour. There is no long-lived secret to leak in the first place — Chapter 11's core argument is that this isn't just "more secure," it eliminates an entire category of incident (a leaked long-lived cloud credential) rather than merely making it harder to exploit.

**Q: What is the STRIDE threat modeling framework?**
> STRIDE is a mnemonic for six categories of threat to systematically consider against a system's design: **S**poofing (pretending to be something/someone you're not), **T**ampering (modifying data or code without authorization), **R**epudiation (denying having performed an action, when there's no way to prove otherwise), **I**nformation disclosure (exposing data to someone unauthorized to see it), **D**enial of service (degrading or denying legitimate use of a system), and **E**levation of privilege (gaining capabilities beyond what was authorized). Chapter 2 uses it as a structured checklist applied to each element of a data-flow diagram — for every process, data store, and trust boundary, ask which of the six categories could apply — specifically because an unstructured "what could go wrong" brainstorm reliably misses entire categories of risk that a systematic pass catches.

**Q: What's the difference between a vulnerability scan passing and an application actually being secure?**
> A vulnerability scan (SAST, SCA, container scan, DAST) checks for a specific, known, and generally pattern-matchable set of issues — it can tell you "no known-CVE dependency, no matched insecure-code pattern, no default credential detected" — but it cannot reason about business logic, cannot catch a novel vulnerability class it has no signature for, and cannot verify the countless configuration and process decisions (who has access to what, whether incident response actually works, whether the team would even notice an intrusion) that determine real-world security posture. A clean scan is necessary but nowhere near sufficient — it's evidence about the presence of *known* problems, not proof of the absence of *all* problems. Chapters 4, 8, and 14 all return to this point because it's the single most common source of false confidence in a DevSecOps pipeline: teams start treating "the pipeline is green" as equivalent to "we are secure."

**Q: What is a blameless postmortem and why does it matter for security specifically?**
> A blameless postmortem investigates an incident by focusing on the systems, processes, and gaps that allowed it to happen, rather than which individual made a mistake — the working assumption is that any reasonable person in the same situation, with the same information and the same tooling, could have made the same choice, so the fix belongs at the systems level, not the personal-accountability level. It matters especially for security because the alternative — a blame-oriented culture — actively suppresses the exact information a security team needs most: if reporting "I think I just committed a secret" or "I clicked a link I shouldn't have" is punished, people stop reporting it, and the organization loses early-warning signal on the incidents that are cheapest to contain quickly. Chapter 13 treats blameless postmortems as a security control in their own right, not just a nice cultural practice — a security incident response process that punishes disclosure produces worse security outcomes than one that doesn't, measurably, because time-to-containment depends entirely on someone speaking up early.

**Q: Why is "trust but verify" central to supply chain security, and what does verification actually look like in practice?**
> Modern software is built almost entirely from components an organization didn't write and often can't fully audit — base images, open-source libraries, CI runner images, build tools — so supply chain security accepts that trust in those components is unavoidable, but insists that trust be verifiable rather than assumed. In practice this means: signing artifacts (so you can verify *what* was built and *by which* pipeline, via `cosign`/Sigstore), generating SBOMs (so you can verify *what's inside* an artifact), and checking provenance attestations against a framework like SLSA (so you can verify *how* an artifact was built — from what source, by what CI system, with what inputs). Chapter 7's framing is that supply chain attacks like SolarWinds and the `xz` backdoor succeeded precisely because verification was absent at exactly these checkpoints — the industry trusted artifacts by default rather than by evidence.

---

## 17.2 Architecture and Internals Questions

**Q: Explain how taint tracking works in a SAST tool.**
> Taint tracking models the flow of untrusted data through a program by marking any value originating from an external, attacker-influenceable source (a "source" — HTTP request parameters, form input, a file read) as "tainted," and then propagating that taint through assignments, function calls, and transformations as the analyzer builds a data-flow graph. If a tainted value reaches a dangerous operation (a "sink" — a SQL query execution, a shell command, an HTML response write) without first passing through a recognized sanitization step that would "cleanse" the taint (parameterized query construction, output encoding, an allow-list validation), the tool flags the path from source to sink as a vulnerability. This is exactly why SAST tools produce false positives on code that's actually safe — if the tool's model of "what counts as sanitization" doesn't recognize a particular library's escaping function, it will keep reporting taint that a human can see is actually handled, which is the root cause of the false-positive triage workload Chapter 4 spends significant time on.

**Q: Explain how `cosign` keyless signing works and why it's preferred over long-lived key pairs.**
> With a traditional key pair, someone generates a private/public signing key, stores the private key somewhere (ideally an HSM, often just a CI secret in practice), and every signature is only as trustworthy as that storage location's security and as the process for knowing whose key it even is. Keyless signing instead has the CI workflow request a short-lived OIDC identity token from its platform's issuer (e.g., GitHub Actions' built-in OIDC provider), presents that token to Sigstore's Fulcio certificate authority, which issues a short-lived (minutes-long) signing certificate binding the specific workflow/repo/ref identity to a throwaway keypair generated on the spot; `cosign` signs the artifact with that ephemeral key, and the signature plus certificate are recorded in Sigstore's public, append-only transparency log (Rekor) before the key is discarded. It's preferred because there is no long-lived private key existing anywhere to leak, and the resulting signature is verifiable against an auditable, publicly-inspectable identity ("this exact workflow, on this exact ref, at this exact time") rather than an opaque key ID that requires separate, manually-maintained infrastructure to know who it belongs to.

**Q: Explain how Falco detects suspicious behavior at the kernel level.**
> Falco uses either a kernel module or, more commonly today, eBPF probes attached to kernel system call entry/exit points to observe every process's syscalls in real time — file opens, process execs, network connections — as they actually happen at the OS level, correlated with container/Kubernetes metadata (namespace, Pod, image) pulled from the container runtime and the Kubernetes API. Falco's rule engine evaluates each observed event stream against a set of declarative conditions (Sysdig's filter-expression syntax, as used in Chapter 10's custom rule examples) and fires an alert the moment a matching pattern occurs — e.g., a shell process being exec'd inside a container, a write to a sensitive system path, an outbound connection to an unexpected destination. The key architectural point is that this operates below the application layer entirely — Falco doesn't need the application to be instrumented, log anything, or even be aware it's being watched, because it's observing the kernel's own record of what actually executed, which is why it can catch a live intrusion that no static scan of the image could ever have found (the malicious behavior didn't exist until the container was actually running and compromised).

**Q: Explain the SLSA framework's core idea.**
> SLSA (Supply-chain Levels for Software Artifacts) defines a set of progressively stricter requirements for how software is built, aimed at making the build process itself verifiable and tamper-resistant rather than just checking the final artifact's contents. Its core idea is provenance: at increasing SLSA levels, the framework requires the build to run on a hosted, non-user-controlled build platform (so a developer's laptop can't quietly inject something), to generate signed provenance metadata describing exactly what source, what build steps, and what dependencies produced a given artifact, and eventually to make the build process itself hermetic and reproducible enough that the same inputs provably always produce the same output. Chapter 7 frames SLSA as answering a different question than an SBOM does — an SBOM says *what's inside* an artifact, while SLSA provenance says *how and by whom* it was built — and the two are complementary rather than substitutes for each other.

**Q: Explain how OPA/Conftest can enforce policy against a Terraform plan before apply.**
> OPA (Open Policy Agent) evaluates structured data against declarative policies written in Rego; Conftest is a thin CLI wrapper that feeds OPA a JSON or YAML input file and a directory of Rego policies and reports which policies pass or fail. The specific pattern for Terraform is: run `terraform plan -out=tfplan.binary`, convert it to JSON with `terraform show -json tfplan.binary`, and run `conftest test` against that JSON with policies written against the plan's `resource_changes` structure — this means the policy check evaluates the *exact* set of changes Terraform is about to make (including the final, fully-interpolated values), not just the static `.tf` source files the way `tfsec`/Checkov typically do. This distinction matters because a value that looks safe in source (a variable default) can resolve to something dangerous at plan time depending on the actual `.tfvars` used for a given environment — Conftest against the plan JSON catches that; scanning source alone would not.

**Q: What's the actual difference between `tfsec`/Checkov and Conftest/OPA for IaC security?**
> `tfsec` and Checkov are purpose-built IaC scanners with a large library of pre-written rules covering common cloud misconfigurations (public S3 buckets, overly permissive security groups, unencrypted storage) — they work directly against Terraform source files, require little to no custom rule-writing to get immediate value, but are limited to whatever misconfiguration patterns their maintainers have already encoded. Conftest/OPA is a general-purpose policy engine with no built-in security knowledge at all — every rule is one your organization writes in Rego — which means far more setup work, but the ability to enforce organization-specific policy that no off-the-shelf scanner could know about (e.g., "every resource must carry a `cost-center` tag," or "only these three approved instance types may be provisioned"), and critically, the ability to evaluate the actual resolved plan rather than only static source. Chapter 9's practical guidance is to run both: `tfsec`/Checkov as a fast, low-effort baseline for well-known cloud misconfigurations, and Conftest/OPA for the custom organizational policy no generic scanner would ever encode.

**Q: Why does Kyverno (or OPA Gatekeeper) need to be an admission webhook rather than a periodic scan of the cluster?**
> An admission webhook intercepts the Kubernetes API request itself, before the object is ever persisted to etcd — it can reject a non-compliant Pod at creation time, meaning the cluster's actual state never diverges from policy even for a moment. A periodic scan (checking existing objects on a schedule and flagging or even deleting non-compliant ones after the fact) means there's always a window — however small — during which a policy-violating resource is live and doing whatever a live resource does, including, in the unsigned-image case from this course's Project 2, actually running attacker-controlled code. Chapter 10's synthesis point is that "detect and remediate after the fact" and "prevent before it ever happens" are different security postures with meaningfully different risk profiles, and admission control is what makes prevention possible for anything created through the normal Kubernetes API.

---

## 17.3 Scenario-Based Questions

**Scenario 1: "A critical CVE is disclosed in a library your organization uses somewhere across 200 microservices — how do you find out what's affected?"**
```
1. Do NOT start by re-scanning 200 repositories from scratch — that's slow
   and exactly the scramble a good supply-chain program avoids. Start by
   querying the SBOM store first (Chapter 7):
   grep -l "log4j-core.*2\.1[4-6]" sboms/*.cdx.json
   # or, with a queryable SBOM database (Dependency-Track, GUAC, etc.):
   curl -s "https://deptrack.internal/api/v1/vulnerability/component/log4j-core" | jq

2. If SBOMs aren't centrally queryable yet, fall back to the SCA tool's own
   organization-wide dashboard (Dependabot/Snyk usually aggregate findings
   across every connected repo) rather than manually checking repo by repo

3. Cross-reference which of the affected services are actually deployed and
   internet-facing vs. internal-only or already deprecated — prioritize by
   real exploitability and blast radius, not alphabetically by repo name

4. For confirmed-affected services, check whether the vulnerable code path
   is even reachable (a vulnerable function that's present but never called
   is lower urgency than one on a live request path) — this determines
   whether it's an emergency patch tonight or a scheduled fix this sprint

5. Patch the highest-risk, actually-exploitable services first via the
   existing Dependabot/Renovate PR flow, backed by the Trivy/SCA build gate
   to confirm the fix actually resolves the flagged CVE before merge

6. Retrospective: if this took hours instead of minutes to even scope, that
   itself is the finding — invest in a centrally queryable SBOM store
   (Chapter 7) so the NEXT critical CVE disclosure is a five-minute query
```

**Scenario 2: "A secret was just discovered committed to a public GitHub repo — what do you do in the next 10 minutes?"**
```
1. Rotate the credential at its source system FIRST, before anything else —
   investigation, git history cleanup, and notification all come after,
   because every minute the old credential remains valid is live exposure
   to anyone who already cloned or indexed the public repo

2. Revoke the specific leaked value if the provider distinguishes rotation
   from revocation (e.g., immediately invalidate a specific GitHub PAT or
   AWS access key ID rather than just issuing a new one alongside it)

3. Check the credential provider's own access/audit logs for any usage
   during the exposure window — this tells you whether this is "exposed,
   no evidence of misuse" or "exposed AND used," which changes the severity
   and required response dramatically

4. Do NOT assume rewriting git history (BFG Repo-Cleaner, git filter-repo)
   removes the exposure — a public repo may already be cloned, forked, or
   cached by GitHub's own systems and third-party scrapers; treat the
   credential as permanently burned regardless of history cleanup

5. Once the credential is rotated and revoked, THEN investigate root cause:
   why didn't the pre-commit gitleaks hook or CI secret scan (Chapter 3,
   Project 1) catch this before it was pushed?

6. Blameless postmortem and a concrete fix — add the specific pattern to
   the secret-scanner's custom rules if it was a bespoke format, and check
   whether the CI-level scan is actually required-to-pass on this repo
```

**Scenario 3: "Your SAST tool just went from 10 findings to 4,000 overnight after a config change — what happened and how do you respond?"**
```
1. Don't panic-triage all 4,000 individually — first confirm what actually
   changed in the config, since this is very rarely 4,000 new real
   vulnerabilities appearing overnight:
   git log -p -- .semgrep.yml  # or whichever SAST config file changed

2. The most common cause is a broadened rule set — a config change from a
   narrow, curated rule pack (e.g., p/owasp-top-ten) to an extremely broad
   or experimental rule pack, or accidentally removing a path-ignore list
   that used to exclude generated code, vendored dependencies, or test
   fixtures from the scan entirely

3. Sample a handful of the new findings across different rules to check
   whether they're genuine (previously-real issues the old config simply
   never looked for) or noise (a rule matching a pattern that isn't
   actually exploitable in this codebase's context)

4. If it's noise from an overly broad rule, don't just mass-suppress —
   identify the specific rule ID(s) responsible for the bulk of the volume
   (a handful of rules are almost always responsible for the vast majority
   of a spike like this) and either tune or disable those specific rules
   with a documented justification, rather than developers individually
   dismissing findings one by one

5. If path-ignore was accidentally dropped, restore it — scanning
   third-party vendored code or generated files for YOUR organization's
   coding-pattern issues is pure noise and erodes trust in the tool fast

6. Fix going forward: config changes to security tooling should go through
   the same PR review as any other pipeline change, with a required step
   of running the new config against the existing codebase and comparing
   finding counts BEFORE merging the config change to main
```

**Scenario 4: "Falco just fired an alert for a shell spawned in a production Pod — walk through your incident response."**
```
1. Do not immediately kill the Pod — capture forensic state first, since
   deleting it destroys process trees, open file descriptors, and network
   connection state that explain what happened:
   kubectl cp <namespace>/<pod>:/proc -o - > /tmp/forensic-proc-snapshot.tar

2. Isolate the Pod at the network layer without deleting it — apply a
   deny-all NetworkPolicy scoped to just this Pod (Advanced Kubernetes
   Chapter 4) so it can't communicate further while still being inspectable

3. Pull the exact Falco event details — user, command line, parent process,
   and correlate the timestamp against centralized audit logs and
   application logs (this course's Project 4 SIEM-style view) to
   reconstruct what happened immediately before the shell spawn

4. Determine the entry vector: was this an application-layer RCE, a
   compromised dependency executing unexpectedly, or a legitimate but
   unauthorized `kubectl exec` by a human using over-broad RBAC access?
   kubectl get events -n <namespace> --field-selector reason=Exec

5. Check blast radius — is the same image running elsewhere in the
   cluster, and does the same suspicious process pattern appear on other
   nodes via Falco's own historical alert log?

6. Once forensics are captured, terminate the compromised Pod and force
   redeploy from a known-good, freshly scanned and signed image (this
   course's Project 2 signing/scanning pipeline is what makes "known-good"
   a verifiable statement rather than a hope)

7. Rotate any credentials the compromised Pod's ServiceAccount had access
   to, since they may have been read from inside the container

8. Blameless postmortem: which earlier control (SAST, SCA, image scanning,
   RBAC scoping, NetworkPolicy) should have prevented or at least slowed
   this entry vector, and is that gap closed now or still open elsewhere
```

**Scenario 5: "A pentest found a Kubernetes NetworkPolicy that isn't actually blocking traffic as expected — how do you debug it?"**
```
1. Confirm the CNI plugin actually enforces NetworkPolicy at all — this is
   the single most common root cause and is easy to overlook: some CNI
   plugins (certain basic/legacy configurations of Flannel, for instance)
   don't enforce NetworkPolicy objects at all, silently accepting the
   object into the API without ever restricting real traffic
   kubectl get pods -n kube-system | grep -iE "calico|cilium|weave|flannel"

2. If the CNI does enforce policy, check the NetworkPolicy's selector logic
   for a scoping mistake — an empty podSelector matches ALL pods in the
   namespace, and a missing policyTypes field can silently mean the policy
   only governs Ingress when Egress restriction was also intended:
   kubectl get networkpolicy <name> -n <namespace> -o yaml

3. Check for a DIFFERENT, broader NetworkPolicy in the same namespace that
   allows the traffic the pentest exploited — NetworkPolicies are additive
   (any policy that allows a connection permits it, regardless of how many
   other policies would deny it), so a second, overly permissive policy
   silently defeats an otherwise-correct restrictive one

4. Verify the actual live behavior directly rather than trusting the
   manifest alone — exec into a test Pod and attempt the exact connection
   the policy is supposed to block:
   kubectl exec -it test-pod -n <namespace> -- nc -zv <target-service> <port>

5. Check whether the traffic path being tested even goes through the CNI's
   enforcement point — traffic to a Pod via a Service's ClusterIP, a
   NodePort, or a hostNetwork Pod can behave differently under some CNI
   implementations than pod-to-pod traffic the policy was actually tested
   against originally

6. Fix and verify: correct the policy (explicit podSelector, correct
   policyTypes, removal of the conflicting permissive policy), then re-run
   the exact same live connection test from Step 4 and confirm it's now
   blocked — a corrected YAML file with no live re-verification is not
   actually a confirmed fix
```

**Scenario 6: "You need to prepare for a SOC 2 audit in 60 days with minimal existing documentation — where do you start?"**
```
1. Start from the controls you can already prove via existing pipeline
   automation, not from a blank documentation template — this course's
   Chapter 12 point is that a DevSecOps pipeline that already runs SAST,
   SCA, image scanning, and signing is already GENERATING most of the
   evidence an auditor wants; the gap is usually collection and mapping,
   not the underlying practice

2. Build the compliance evidence mapping table first (this course's
   Project 4 pattern) — for each Trust Services Criterion in scope, list
   which existing pipeline artifact serves as evidence (CI logs, Trivy scan
   reports, Kyverno admission denials, Falco alert history, PR approval
   records) and where it's currently retained

3. Identify genuine gaps — controls with no automated evidence at all —
   and prioritize closing the highest-risk gaps first: access review
   cadence, incident response runbook existence and rehearsal history, and
   a documented, enforced change-management process are almost always the
   first things auditors ask for that a security-focused engineering team
   hasn't yet formalized in writing

4. Ensure retention windows for the evidence actually cover the audit
   period — a scan history that's only kept for 30 days is useless for
   proving a control operated correctly across a 6-12 month audit window;
   fix retention configuration immediately, since evidence can't be
   retroactively generated for a period that's already passed

5. Write (and actually rehearse, not just draft) the incident response
   runbooks if they don't exist yet — auditors typically want evidence of
   both the documented process AND that it's been exercised at least once,
   which a tabletop exercise satisfies even without a real incident

6. Engage a SOC 2 readiness tool or auditor early rather than waiting until
   documentation feels "complete" — a preliminary gap assessment from
   someone who's done this before will find missing evidence you don't
   know to look for, and 60 days is tight enough that late discovery of a
   gap is a real risk to the timeline
```

---

## 17.4 System Design Questions

**"Design a complete DevSecOps pipeline for a new microservices platform from scratch."**
```
1. PR-time gates: SAST (Semgrep/CodeQL) and SCA (Dependabot/Snyk) on every
   pull request with inline annotations, plus IaC scanning (tfsec/Checkov)
   and OPA/Conftest policy checks against any infrastructure change,
   required-to-pass before merge is allowed

2. Build-time: container image build → Trivy scan failing on Critical/High
   with available fixes → syft SBOM generation → cosign keyless signing
   tied to the CI workflow's OIDC identity, all per-service

3. Deploy-time: Kyverno (or OPA Gatekeeper) admission policy verifying both
   image signature and scan-attestation status before any Pod using that
   image is admitted to the cluster — this is the single control point
   that makes every earlier stage's check actually enforced, not just advisory

4. Runtime: Falco for kernel-level anomaly detection routed through the
   same Alertmanager already used for reliability alerting, `kube-bench`
   run on a recurring schedule for CIS drift detection, and NetworkPolicies
   plus RBAC scoped per-namespace as defense in depth beneath the admission
   layer

5. Credentials: OIDC-based short-lived authentication for every CI-to-cloud
   and CI-to-cluster interaction, with zero long-lived static credentials
   stored anywhere in the pipeline

6. Observability and compliance: centralize audit logs, Falco events, and
   pipeline scan results into the existing Loki/ELK stack under a
   dedicated security category, with a compliance evidence mapping
   maintained continuously rather than assembled once a year before an audit

7. Incident response: documented, tabletop-rehearsed runbooks for the most
   likely scenarios (leaked secret, runtime intrusion, critical CVE
   disclosure) written and exercised before they're needed for real
```

**"Design a secrets management strategy for a multi-cloud, multi-team organization."**
```
1. Centralize on one secrets engine (HashiCorp Vault, or a per-cloud
   managed secrets manager federated through a common access pattern)
   rather than letting each team invent its own convention — consistency
   is what makes rotation, auditing, and leak response tractable at
   multi-team scale

2. Use dynamic, short-lived credentials wherever the backing system
   supports them (database credentials, cloud IAM roles) rather than
   long-lived static secrets checked into a vault once and never rotated
   — Vault's dynamic secrets engines generate a fresh, narrowly-scoped
   credential per request with a built-in lease and automatic expiry

3. Scope access by least privilege per team/service via Vault policies or
   equivalent IAM scoping, with every access authenticated through a
   workload identity (Kubernetes ServiceAccount token, cloud instance
   identity, or OIDC) rather than a shared static token multiple teams use

4. Deploy the External Secrets Operator (or each cloud's native secrets-
   injection mechanism) so application teams reference a secret by name in
   their own manifests without ever handling the actual secret value or
   needing direct Vault access themselves

5. Automate rotation on a defined cadence for every secret class, and treat
   "this secret has no rotation policy" as a finding to remediate, not an
   acceptable long-term state

6. Layer leak detection (gitleaks in pre-commit and CI, secret-scanning on
   the git host itself) across every team's repositories uniformly, backed
   by a single, well-rehearsed rotation-first incident response runbook so
   every team responds to a leak the same, fast way regardless of which
   cloud or system the leaked secret belonged to

7. Audit every secret access centrally (Vault's audit log, or each cloud's
   equivalent) feeding the same centralized logging pipeline used for
   everything else, so "who accessed this credential, and when" is always
   answerable during an investigation
```

**"Design a security incident response process for a 24/7 SaaS platform."**
```
1. Define severity tiers up front with concrete, unambiguous criteria for
   each (e.g., confirmed data exposure vs. suspicious-but-unconfirmed
   activity vs. a policy violation with no evidence of harm) so responders
   aren't debating severity while an incident is actively unfolding

2. Establish a dedicated security on-call rotation, separate from general
   reliability on-call, with its own escalation path and its own paging
   channel — routed through the same Alertmanager infrastructure as
   reliability alerts (reusing the grouping/routing discipline from
   Monitoring Topic 10) but to a distinct receiver and rotation

3. Pre-stage runbooks for the highest-likelihood scenarios specific to a
   24/7 SaaS platform: a leaked credential, a Falco-detected runtime
   intrusion, a critical CVE in a widely-used dependency, and a suspected
   customer-data exposure — each with clear first-10-minutes actions,
   since a 24/7 platform can't wait for business hours to begin containment

4. Build the forensic-capture-before-remediation habit into every runbook
   (capture state, isolate, THEN remediate) so evidence isn't destroyed by
   an understandably urgent instinct to just shut the problem down immediately

5. Establish a clear internal and external communication plan in advance —
   who needs to be notified, on what timeline, and who's authorized to
   communicate externally — since figuring this out live, during an
   incident, at 3 a.m., is exactly when mistakes and delays happen

6. Require a blameless postmortem after every incident above a defined
   severity threshold, with concrete follow-up actions tracked to
   completion, not just documented and forgotten

7. Regularly tabletop-rehearse the runbooks with the actual on-call
   rotation, on a schedule, so the first time anyone reads through the
   leaked-secret runbook isn't during a real 3 a.m. leaked-secret incident
```

---

## 17.5 Quick-Fire Questions

| Question | Answer |
|----------|--------|
| Security testing that analyzes code without running it? | SAST |
| Security testing that attacks a running application? | DAST |
| Tool category that finds vulnerable dependencies? | SCA (Software Composition Analysis) |
| Machine-readable inventory of everything inside a build artifact? | SBOM |
| Framework defining threat categories: Spoofing, Tampering, Repudiation, Information disclosure, DoS, Elevation of privilege? | STRIDE |
| Framework for build provenance and supply chain integrity levels? | SLSA |
| Tool for keyless container image signing? | cosign (Sigstore) |
| Tool commonly used to generate an SBOM? | syft |
| Tool that scans container images for known CVEs? | Trivy (or Grype) |
| Kubernetes admission controller commonly used for image-signature verification policy? | Kyverno (or OPA Gatekeeper) |
| Tool that runs the CIS Kubernetes Benchmark against a cluster? | kube-bench |
| Runtime security tool that detects suspicious behavior via kernel-level syscall monitoring? | Falco |
| Authentication approach replacing long-lived CI cloud credentials? | OIDC-based short-lived federated credentials |
| Policy-as-code engine commonly paired with Terraform plans via Conftest? | OPA (Open Policy Agent) |
| Postmortem style that focuses on systems, not individual blame? | Blameless postmortem |
| Public, append-only log recording Sigstore signatures? | Rekor |

---

## 17.6 "Walk Me Through Your DevSecOps Experience"

STAR format example:

```
Situation: Our platform shipped about a dozen services with security
handled almost entirely as a manual review step right before a release —
a security engineer would look over a diff a day or two before shipping,
dependencies were updated only when someone happened to notice a CVE
mentioned somewhere, and container images were built from whatever base
image a Dockerfile happened to specify years earlier with no ongoing
scanning. A postmortem after a minor incident (a vulnerable dependency
that had been flagged in an advisory for months) found the real root
cause wasn't the vulnerability itself — it was that nobody had a
reliable, automated way to even know it existed in our stack.

Task: Build shift-left security into the existing CI/CD pipeline so
vulnerabilities were caught automatically, as early as possible, without
turning every pull request into a security-review bottleneck the team
would learn to route around.

Action:
1. Added Semgrep SAST scanning to every pull request with inline PR
   annotations, scoped initially to the OWASP Top Ten rule set to avoid
   an overwhelming first-week flood of findings, then expanded coverage
   incrementally as the team built trust in the tool's signal quality
2. Configured Dependabot across every repository with a defined SLA for
   Critical/High findings, replacing "someone happens to notice an
   advisory" with an automated PR appearing the same week a CVE was disclosed
3. Added a Trivy build-blocking scan and `syft` SBOM generation to the
   container build pipeline, and rolled out `cosign` keyless image signing
   so every deployed image had a verifiable, auditable build identity
4. Deployed a Kyverno admission policy requiring signature verification
   before any image could run in the cluster, closing the gap between
   "we scan images" and "an unscanned image can still actually run"
5. Migrated CI's cloud deployment credentials from a long-lived static
   AWS key stored as a GitHub secret to OIDC-based federated
   authentication, eliminating that entire class of leaked-credential risk
6. Wrote and tabletop-rehearsed incident response runbooks for a leaked
   secret and a runtime intrusion, after realizing during a fire-drill
   exercise that the team's actual first move under pressure (investigate
   first, rotate later) was backwards
7. Built a compliance evidence mapping table tying each of these
   automated controls to the specific SOC 2 criteria our upcoming audit
   would need evidence for, turning a dreaded annual scramble into a
   continuously-current artifact

Result: The vulnerable-dependency class of incident that triggered this
work became structurally much harder to repeat — Dependabot now surfaces
CVEs within days of disclosure instead of months, backed by an SLA. The
SOC 2 audit that followed went from an anticipated multi-week documentation
scramble to a few days of walking the auditor through already-existing
pipeline evidence. And when a genuine Falco alert fired on a since-patched
staging vulnerability, the on-call engineer followed the rehearsed runbook
almost verbatim, containing the issue in minutes rather than improvising
a response for the first time during a real incident.
```

**Self-Check Before Your Interview**

- Can you explain, without notes, the actual mechanics of `cosign` keyless signing — what Fulcio and Rekor each do — rather than just "it signs images"?
- Can you calculate the practical difference SAST, SCA, and DAST each catch, with a concrete example vulnerability type for each?
- Can you walk through the leaked-secret response order (rotate first, investigate second) and explain why that order matters, under a follow-up "why not investigate first" question?
- Can you describe a real (or project-based) Falco/runtime-intrusion response using the forensics-before-remediation flow from section 17.3, narrating your reasoning rather than jumping straight to "I'd just delete the Pod"?
- Can you defend a compliance evidence mapping table under a follow-up question like "how do you know this evidence would actually satisfy an auditor"?

No separate hands-on exercise for this chapter — working through the scenarios and system design questions above out loud, from memory, and defending your reasoning under follow-up questions, is the exercise.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./16-projects.md">← Previous: Hands-On Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./18-course-summary.md">Next: Course Summary →</a>
</div>
