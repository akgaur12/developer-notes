# Chapter 18 — Course Summary & Next Steps

## 18.1 What You've Learned

Congratulations on completing **Security (DevSecOps)** — Topic 11 of the DevOps Learning Path, and the final course in the entire roadmap. You started this course able to design, build, deploy, monitor, and operate production systems across the full stack — and you finish it able to secure every one of those stages, automatically, continuously, as a built-in part of the pipeline rather than a separate department's afterthought.

| Chapter | What You Can Do Now |
|---------|----------------------|
| 01 Introduction to DevSecOps | Explain shift-left security and why DevSecOps treats security as everyone's responsibility, and recognize where security was already lurking in every prior course |
| 02 Threat Modeling | Apply STRIDE systematically to a system's design and prioritize risks by realistic likelihood and impact, not gut feeling |
| 03 Secrets Management | Design a production-grade secrets strategy with Vault or a cloud secrets manager, dynamic credentials, rotation, and leak detection |
| 04 SAST and Static Code Analysis | Integrate Semgrep/CodeQL into CI with inline PR annotations, and triage findings without drowning developers in false positives |
| 05 Dependency Scanning and SCA | Find and remediate vulnerable dependencies (including transitive ones) with Dependabot/Renovate/Snyk, and reason about license compliance |
| 06 Container and Image Security | Scan images with Trivy/Grype, build minimal/distroless images, and sign and verify images before they run |
| 07 Software Supply Chain Security | Generate and query SBOMs, explain SLSA provenance levels, and recognize real-world supply chain attack patterns |
| 08 DAST and Runtime Security Testing | Run OWASP ZAP against a live application, test API security, and understand where fuzzing fits |
| 09 Infrastructure as Code Security | Scan Terraform with `tfsec`/Checkov and enforce custom policy against a plan with OPA/Conftest before `apply` |
| 10 Kubernetes Security Deep Dive | Apply RBAC, admission control, and NetworkPolicies through a security lens, benchmark a cluster with `kube-bench`, and detect runtime threats with Falco |
| 11 CI/CD Pipeline Security | Secure the pipeline itself with least-privilege, OIDC-based credentials, and artifact signing and provenance |
| 12 Compliance and Governance | Map security controls to SOC 2/PCI-DSS/CIS frameworks and maintain audit evidence continuously instead of scrambling annually |
| 13 Security Monitoring and Incident Response | Design SIEM-style centralized security visibility and run an effective, blameless incident response process |
| 14 Best Practices | Apply production-grade DevSecOps patterns across the entire pipeline |
| 15 Common Mistakes | Recognize and avoid the most frequent DevSecOps mistakes before they become incidents |
| 16 Projects | Secured a CI pipeline end to end, built a signed and admission-gated container supply chain, hardened a live cluster with `kube-bench`/Falco/OIDC, and designed a full production-grade DevSecOps platform with compliance evidence mapping |
| 17 Interview Preparation | Answer DevSecOps/security-engineer-level foundational, architectural, scenario, and system-design questions with confidence |

---

## 18.1.1 The Mental Model, in One Paragraph

Every prior course in this roadmap taught you to build, deploy, and observe systems; this course taught you to defend them — and the unifying idea across all eighteen chapters is that security, like observability before it, is not a single tool but a discipline of turning every stage of the software lifecycle into a checkpoint that produces evidence. A threat model tells you what could go wrong before you write a line of code; SAST and SCA catch what went wrong in the code and its dependencies before it merges; image scanning, signing, and SBOMs make sure what runs in production is provably the thing you actually built and scanned, not something silently substituted along the way; IaC scanning and policy-as-code catch misconfiguration before infrastructure ever exists; Kubernetes hardening, Falco, and pipeline-credential discipline defend the running system and the system that deploys it; and compliance mapping and incident response turn all of the above into something you can prove happened, and something you can recover from when, despite every control, something still gets through. None of these are separate disciplines bolted together — they are one continuous chain of "verify, don't assume" applied at every single stage from the first commit to the running Pod, which is exactly the shift-left philosophy Chapter 1 opened with and every subsequent chapter made concrete.

### 18.1.2 Professional Capabilities Unlocked

Framed the way a resume or an interview conversation would frame it, completing this course means you can now credibly say you can:

- Threat model a new system before it's built, rather than discovering its weaknesses after an incident
- Stand up a secrets management architecture that eliminates hardcoded credentials organization-wide, with rotation and leak detection as defaults, not afterthoughts
- Add automated SAST, SCA, and secret scanning to any CI pipeline in under a day, with a false-positive triage process the team will actually trust
- Build a container supply chain where every deployed image is scanned, SBOM'd, signed, and cryptographically verifiable back to the exact CI run that produced it
- Scan and gate Infrastructure as Code changes with both off-the-shelf and custom policy before they're ever applied
- Harden a Kubernetes cluster against a recognized industry benchmark (CIS) and detect live intrusions at the kernel level with Falco
- Eliminate long-lived cloud credentials from a CI/CD pipeline entirely in favor of short-lived, workflow-scoped OIDC authentication
- Map a set of engineering controls to the language auditors actually use (SOC 2, PCI-DSS, CIS) and produce continuous evidence instead of a once-a-year scramble
- Run an incident response process — for a leaked secret, a runtime intrusion, or a disclosed CVE — calmly, in the right order, with a rehearsed runbook rather than improvisation under pressure
- Speak fluently, in an interview, about the DevSecOps discipline end to end — not just define the terms, but reason through scenarios and defend design decisions under follow-up questions

---

## 18.2 Completion Checklist

```
Secrets & Threat Modeling:
  [ ] Can apply STRIDE to a real system design and prioritize the resulting risks
  [ ] Can design a secrets management architecture using Vault or a cloud secrets manager
  [ ] Can explain dynamic/short-lived credentials and why they beat static long-lived secrets
  [ ] Can set up leak detection (pre-commit + CI) and know the rotation-first response

Application Security Testing:
  [ ] Can integrate a SAST tool into CI with inline PR annotations
  [ ] Can triage a SAST finding and distinguish a real vulnerability from a false positive
  [ ] Can run SCA and explain why transitive dependencies matter more than direct ones
  [ ] Can run a DAST scan against a live application and interpret the results
  [ ] Can explain the difference between SAST, SCA, and DAST, and when each is used

Supply Chain Security:
  [ ] Can scan a container image and fail a build on Critical/High CVEs
  [ ] Can generate an SBOM and explain what problem it solves
  [ ] Can sign a container image keylessly with cosign, tied to a CI workflow's OIDC identity
  [ ] Can explain the SLSA framework's core idea and how it differs from an SBOM

Infrastructure & Kubernetes Security:
  [ ] Can scan Terraform with tfsec/Checkov and enforce custom policy with OPA/Conftest against a plan
  [ ] Can run kube-bench against a cluster and remediate real findings
  [ ] Can write and deploy a Kyverno/Gatekeeper admission policy enforcing image signature verification
  [ ] Can deploy Falco, write a custom detection rule, and route its alerts to a real notification channel

Pipeline Security & Compliance:
  [ ] Can explain least-privilege CI credentials and migrate a pipeline from static keys to OIDC
  [ ] Can map a set of pipeline controls to real compliance framework requirements (SOC 2/PCI-DSS/CIS)
  [ ] Can explain the difference between a clean vulnerability scan and an actually-secure application

Incident Response:
  [ ] Can run the rotation-first response for a leaked secret without hesitating on order of operations
  [ ] Can run a forensics-before-remediation response for a runtime intrusion
  [ ] Can write and actually rehearse (not just draft) an incident response runbook
  [ ] Can explain blameless postmortems and why they matter specifically for security

Projects Completed:
  [ ] Project 1: CI pipeline secured with SAST, dependency updates, and secret-leak detection, proven against deliberate vulnerabilities
  [ ] Project 2: Container image scanned, SBOM'd, signed, and enforced via a Kubernetes admission policy
  [ ] Project 3: Cluster CIS-remediated, Falco detecting and alerting on intrusions, CI migrated to OIDC
  [ ] Project 4: Full production-grade DevSecOps platform with PR-to-runtime gating and compliance evidence mapping
```

If any row above still feels shaky, that's a signal, not a failure — go back to the relevant chapter and redo the hands-on exercise. Security skills are trusted in an interview, and in production, only once they're muscle memory rather than something you've read about once — and unlike most other domains in this roadmap, a security gap you don't notice is a gap an attacker eventually will.

---

## 18.3 DevSecOps Command Quick Reference

```bash
# ── Secrets (Chapter 03) ────────────────────────────────────────────────
vault kv put secret/app/db password="..."          # write a static secret
vault read database/creds/app-role                  # request a dynamic, short-lived DB credential
vault kv metadata get -mount=secret app/db           # inspect a secret's version/rotation metadata

# ── SAST / SCA (Chapters 04-05) ─────────────────────────────────────────
semgrep --config p/owasp-top-ten .                   # run a curated SAST rule pack locally
semgrep --config p/owasp-top-ten --sarif -o out.sarif .   # emit SARIF for code-scanning upload
gitleaks detect --source . -v                        # scan the working tree and history for secrets
gitleaks protect --staged                            # pre-commit hook mode — scan only staged changes

# ── Container / Image Security (Chapters 06-07) ─────────────────────────
trivy image --severity CRITICAL,HIGH --exit-code 1 myapp:1.0     # build-blocking image scan
trivy fs --severity CRITICAL,HIGH .                              # scan a filesystem/repo directly
syft myapp:1.0 -o cyclonedx-json > sbom.cdx.json                  # generate an SBOM
grype sbom:sbom.cdx.json                                           # scan an existing SBOM for CVEs
cosign sign --yes myregistry/myapp@sha256:...                      # keyless sign an image digest
cosign verify --certificate-identity-regexp ".*" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  myregistry/myapp@sha256:...                                       # verify a keyless signature
cosign attest --predicate sbom.cdx.json --type cyclonedx myapp@sha256:...   # attach an SBOM attestation

# ── DAST (Chapter 08) ────────────────────────────────────────────────────
docker run -t owasp/zap2docker-stable zap-baseline.py -t https://staging.example.com

# ── IaC Security (Chapter 09) ───────────────────────────────────────────
tfsec .                                              # scan Terraform source for misconfigurations
checkov -d infra/                                    # broader policy scan across IaC frameworks
terraform show -json tfplan.binary > tfplan.json     # convert a plan to JSON for policy evaluation
conftest test tfplan.json --policy policy/           # enforce custom Rego policy against the plan

# ── Kubernetes Security (Chapter 10) ────────────────────────────────────
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench                          # view the CIS benchmark report
kubectl get clusterpolicy                            # list active Kyverno policies
falco --list                                          # list all loaded Falco rules
kubectl logs -n falco -l app.kubernetes.io/name=falco -f   # tail live Falco detections

# ── CI/CD Pipeline Security (Chapter 11) ────────────────────────────────
aws sts get-caller-identity                          # confirm which identity a CI job is actually running as
gh api /repos/OWNER/REPO/actions/oidc/customization/sub   # inspect a repo's OIDC subject claim customization
```

---

## 18.4 DevSecOps Toolchain Quick Reference

| Tool | Purpose | This Course's Chapter |
|------|---------|-------------------------|
| HashiCorp Vault / cloud secrets manager | Centralized secrets storage, dynamic credentials, rotation | 03 |
| External Secrets Operator | Syncs secrets from Vault/cloud secrets managers into Kubernetes without apps touching the secrets engine directly | 03 |
| Semgrep / CodeQL | SAST — static analysis of source code for known-vulnerable patterns | 04 |
| Dependabot / Renovate / Snyk | SCA — automated detection and remediation PRs for vulnerable dependencies | 05 |
| Trivy / Grype | Container image and filesystem vulnerability scanning | 06 |
| cosign / Sigstore | Keyless container image signing and verification, backed by Fulcio and Rekor | 06, 07, 11 |
| syft / SBOM tooling | Generates a Software Bill of Materials (CycloneDX/SPDX) for a build artifact | 07 |
| OWASP ZAP | DAST — attacks a running application to find exploitable runtime vulnerabilities | 08 |
| tfsec / Checkov | Static scanning of Terraform source for cloud misconfigurations | 09 |
| Conftest / OPA | General-purpose policy-as-code engine, commonly used to enforce custom policy against a Terraform plan | 09 |
| kube-bench | Runs the CIS Kubernetes Benchmark against a live cluster | 10 |
| Falco | Kernel-level (eBPF/kernel module) runtime threat detection inside containers | 10 |
| Kyverno / OPA Gatekeeper | Kubernetes admission control — enforces policy (e.g., image signature verification) before a resource is ever created | 10 |

---

## 18.5 Security Quick Reference — The Full Defense-in-Depth Stack

```
Threat model (STRIDE)
        │  what could go wrong, before a line of code exists
        ▼
Secrets management (Vault / cloud secrets manager, dynamic creds, rotation)
        │  nothing sensitive is ever hardcoded or long-lived
        ▼
SAST + SCA on every pull request (Semgrep/CodeQL + Dependabot/Snyk)
        │  known-vulnerable code patterns and dependencies caught before merge
        ▼
Signed, scanned, SBOM'd artifacts (Trivy + syft + cosign)
        │  what's built is provably what's scanned, and provably unmodified since
        ▼
IaC policy checks before apply (tfsec/Checkov + Conftest/OPA)
        │  infrastructure misconfiguration caught before it ever exists
        ▼
Hardened Kubernetes runtime (RBAC, NetworkPolicies, admission control, Falco)
        │  least privilege, segmented traffic, signature-gated deploys,
        │  kernel-level detection of anything that still gets through
        ▼
Secured pipeline (OIDC-based short-lived credentials, least-privilege CI)
        │  the system that deploys everything above is itself not a soft target
        ▼
Compliance evidence (continuously generated, mapped to SOC 2/PCI-DSS/CIS)
        │  every layer above produces auditable proof, not just a good feeling
        ▼
Monitoring & incident response (centralized logs, Falco→Alertmanager, runbooks)
        │  the layer that assumes every layer above will eventually be tested
        │  by something real, and makes sure the response is fast and calm
```

This is the same "defense in depth" idea Chapter 10 introduced for Kubernetes specifically, now shown at the scale of the entire course — no single layer is assumed to be perfect, and the layers are ordered so that a failure at one is very likely to be caught by the next. An attacker who gets past code review still has to get past dependency scanning; an attacker who gets past that still has to get past image scanning and signing; an attacker who somehow gets a workload running still has to get past RBAC, NetworkPolicies, and Falco; and if all of that somehow fails, the incident response process and the audit trail it depends on are what determine whether the story ends in minutes or months.

---

## 18.6 You've Completed the Full DevOps Roadmap

This is not the end of one course — it's the end of an eleven-topic, roughly two-hundred-chapter journey that started with `ls`, `cd`, and file permissions in Linux Fundamentals. It's worth pausing on the full shape of what that journey actually built, because no single course along the way could show you the whole picture:

- **Foundations (Topics 1-3):** Linux Fundamentals, Networking Basics, and Git & Version Control gave you the bedrock every other tool in this roadmap silently assumes — a shell you're fluent in, an understanding of how machines actually talk to each other, and a version control discipline every CI/CD pipeline in every later course was built on top of.
- **Containers (Topic 4):** Docker taught you to package an application and its dependencies into a single, portable, reproducible unit — the atomic building block that Kubernetes, CI/CD, and this security course's image scanning and signing all operate on.
- **Automation (Topic 5):** CI/CD Pipelines taught you to turn manual, error-prone deployment steps into a repeatable, automated pipeline — the exact pipeline this course spent Chapters 4, 5, 9, and 11 teaching you to secure.
- **Cloud (Topic 6):** Cloud Fundamentals (AWS) taught you to provision and operate real infrastructure at scale, including the IAM model this course's OIDC-based credential work builds directly on.
- **Infrastructure as Code (Topic 7):** Terraform taught you to define infrastructure declaratively and reproducibly — the exact discipline this course's Chapter 9 taught you to scan and gate with policy before every `apply`.
- **Orchestration (Topics 8-9):** Kubernetes Basics and Advanced Kubernetes taught you to run containerized applications reliably at scale, with the RBAC, admission control, and NetworkPolicy mechanisms this course's Chapter 10 revisited entirely through a security lens.
- **Operations (Topic 10):** Monitoring & Logging taught you to see what your systems are actually doing — the same alerting, correlation, and centralized-logging instincts this course's Chapters 10 and 13 pointed at a different category of signal: security events instead of reliability events.
- **Security (Topic 11 — this course):** DevSecOps taught you to defend everything the previous ten topics taught you to build, deploy, and observe — closing the loop on a roadmap that mentioned security in nearly every earlier course without ever giving it full depth, until now.

What this specific combination of skills qualifies you for is exactly what the README's own milestone table names as the final rung: **Platform Engineer** — someone whose competency spans full-stack infrastructure, including security, observability, and compliance, not as separate specialties bolted together but as one coherent practice. A engineer who has genuinely completed all eleven topics can take a business requirement, provision the infrastructure for it, containerize and orchestrate the application, build and secure the pipeline that ships it, monitor whether it's healthy, and defend it against the threats it will actually face in production — the complete loop, not a slice of it.

### 18.6.1 How Every Earlier Course Fed Into This One

This course never stood alone — nearly every chapter in it took a foundation laid by an earlier topic and asked "now how do we make sure this can't be turned against us?" Seeing that dependency chain laid out explicitly is a useful way to appreciate just how cumulative this roadmap actually was:

| Earlier Topic & Chapter | What It Gave You | This Course's Chapter That Built On It |
|---|---|---|
| Docker Chapter 10 (Security) | Non-root users, minimal base images, basic image scanning awareness | Chapter 06 (Container and Image Security) — taken from awareness to enforced, signed, admission-gated practice |
| CI/CD Pipelines Chapter 10 (Secrets Management) | Storing a secret as a CI variable instead of hardcoding it | Chapter 03 (Secrets Management) — taken from "store it somewhere safer" to dynamic, rotated, leak-detected secrets management |
| Cloud Fundamentals AWS Chapter 2 (IAM) | Users, roles, and policies as the access-control primitive | Chapter 11 (CI/CD Pipeline Security) — taken to OIDC-federated, credential-free CI-to-cloud authentication |
| Cloud Fundamentals AWS Chapter 10 (Security) | Security groups, encryption at rest, basic AWS hardening | Chapter 09 (IaC Security) and Chapter 12 (Compliance) — taken to policy-as-code enforcement and audit-ready evidence |
| Terraform Chapter 14 (mentioned `tfsec`) | Awareness that IaC itself can be scanned for misconfiguration | Chapter 09 (Infrastructure as Code Security) — taken to full `tfsec`/Checkov/Conftest pipeline gating before `apply` |
| Advanced Kubernetes Chapters 2-4 (RBAC, Admission Control, NetworkPolicies) | The mechanisms themselves — how to write a Role, a NetworkPolicy, an admission webhook | Chapter 10 (Kubernetes Security Deep Dive) — taken from "how it works" to "how to use it to actively stop an attack" |
| Advanced Kubernetes Chapter 13 (Auditing) | Kubernetes audit logs as a debugging tool for cluster incidents | Chapter 13 (Security Monitoring and Incident Response) — the same logs, now the primary forensic record for a security incident |
| Monitoring & Logging Chapter 7 (Alerting) | Grouping, routing, and inhibition discipline to avoid alert fatigue | Chapter 10 and Chapter 13 — the identical Alertmanager discipline, now carrying Falco and security-event signal instead of latency/error-rate signal |
| Monitoring & Logging Chapters 8-10 (Logging) | Centralized, structured, correlated logging across a distributed system | Chapter 13 — the same centralized logging pipeline, extended into a basic SIEM-style security view |

If you completed all ten prior courses before starting this one, none of this course's chapters were really teaching you a brand-new discipline from a standing start — every one of them was showing you how to point a skill you already had at a security-shaped version of the same problem.

---

## 18.7 The Full Picture

```
Topic 1:      Linux Fundamentals                    ✅ Complete
Topic 2:      Networking Basics                     ✅ Complete
Topic 3:      Git & Version Control                 ✅ Complete
Topic 4:      Docker                                ✅ Complete
Topic 5:      CI/CD Pipelines                       ✅ Complete
Topic 6:      Cloud Fundamentals (AWS)              ✅ Complete
Topic 7:      Infrastructure as Code (Terraform)    ✅ Complete
Topic 8:      Kubernetes Basics                     ✅ Complete
Topic 9:      Advanced Kubernetes                   ✅ Complete
Topic 10:     Monitoring & Logging                  ✅ Complete
Topic 11:     Security (DevSecOps)                  ✅ Complete  ← YOU ARE HERE

                    ┌─────────────────────────────────────────┐
                    │   THE ROADMAP IS COMPLETE.                │
                    │   11 topics. ~200 chapters. One engineer   │
                    │   who can build, ship, run, watch, and     │
                    │   defend a production system end to end.  │
                    └─────────────────────────────────────────┘
```

Every earlier course's diagram in this series ended with at least one "coming soon" box. This is the one course-summary in the entire roadmap where that's no longer true — there is nothing left on the list.

---

## 18.8 Where to Go From Here

There's no Topic 12 to point you to, but that doesn't mean the learning stops — it means it changes shape, from following a curriculum to directing your own growth. A few genuinely useful directions:

- **Pursue relevant certifications.** The CKA (Certified Kubernetes Administrator) and CKAD (Certified Kubernetes Application Developer), recapped back in Advanced Kubernetes Chapter 1, validate the orchestration skills from Topics 8-9; the CKS (Certified Kubernetes Security Specialist) validates almost exactly what this course's Kubernetes-security chapter covers, and is a natural next target given how directly it overlaps with Chapter 10's material. On the cloud side, an AWS certification (Solutions Architect or the security-specialty track) validates Topic 6's material at a more formal, third-party-verified level. More broadly, a foundational security certification like (ISC)²'s Certified in Cybersecurity or CompTIA Security+ can round out the security fundamentals this course covered from a DevOps-practitioner angle rather than a traditional security-analyst one.
- **Build and open-source the Chapter 16 Project 4 capstone.** A working, documented, public repository containing a real PR-to-runtime DevSecOps pipeline — SAST, SCA, IaC scanning, signed and scanned images, admission-gated deploys, Falco, and a compliance evidence table — is a far stronger portfolio piece than a resume bullet point. Interviewers can clone it, read the runbooks, and ask you to walk through a design decision live.
- **Contribute to open-source security tooling.** Projects like Semgrep, Trivy, Falco, Kyverno, cosign/Sigstore, and OPA are all actively developed in the open, and contributing — even something as small as a documentation fix, a new detection rule, or a bug report with a good reproduction — is one of the fastest ways to deepen real understanding of how these tools work internally, not just how to configure them.
- **Join security and DevOps communities.** The CNCF's Slack and working groups, security-focused subreddits and Discord communities, and local DevOps/SRE meetups are where the practical, unglamorous lessons ("here's what actually broke when we rolled this out") get shared well before they show up in official documentation. Treat this course as your entry ticket into those conversations, not a substitute for having them.
- **Keep tracking the moving landscape.** DevSecOps does not hold still — new CNCF security projects graduate regularly, compliance requirements evolve (new frameworks, new interpretations of existing ones), and the supply chain attack techniques this course covered as "current" will be superseded by new ones. The specific tools in this course's toolchain reference may not all still be the industry default in five years; the underlying discipline — verify, don't assume, at every stage — will still be exactly right.
A quick reference for the certifications mentioned above, since "which one, and issued by whom" is often the first practical question:

| Certification | Issuing Body | Most Directly Validates |
|---|---|---|
| CKA — Certified Kubernetes Administrator | CNCF/Linux Foundation | Topics 8-9 (cluster operations, troubleshooting) |
| CKAD — Certified Kubernetes Application Developer | CNCF/Linux Foundation | Topics 8-9 (application deployment and configuration) |
| CKS — Certified Kubernetes Security Specialist | CNCF/Linux Foundation | This course's Chapter 10 material, almost directly |
| AWS Certified Solutions Architect | Amazon Web Services | Topic 6 (cloud infrastructure design) |
| AWS Certified Security – Specialty | Amazon Web Services | Topic 6's security material and this course's cloud-adjacent chapters |
| Certified in Cybersecurity (CC) | (ISC)² | Foundational security concepts underpinning this entire course |
| Security+ | CompTIA | General security fundamentals, a common entry point outside the Kubernetes-specific certs above |

The CKS is worth calling out specifically: its exam objectives overlap this course's Chapter 10 almost line for line (cluster hardening, supply chain security, runtime detection, admission control), which makes it the single most direct, verifiable proof-of-completion available for the Kubernetes-security material in this course, for anyone who wants a third-party credential rather than only a self-assessed checklist.

- **Find a real system to keep practicing on.** Every course in this roadmap emphasized hands-on repetition over passive reading, and that doesn't stop just because the curriculum did — a home lab, a personal project, or a low-stakes work system you're allowed to experiment on is where this course's skills either solidify into instinct or quietly fade. Volunteer to lead a threat-modeling session, propose adding a SAST gate to a real pipeline at work, or run `kube-bench` against a cluster you actually operate — applied practice against a system with real stakes teaches things no tutorial-scale project ever will.
- **Teach what you've learned.** Explaining STRIDE, or why keyless signing beats a stored private key, or how an SLO's error budget works, to someone else is one of the fastest ways to discover the parts of your own understanding that were shakier than they felt while reading — consider writing up a project, presenting at a local meetup, or mentoring someone earlier in this exact roadmap.

---

## 18.9 Recommended Resources

- **OWASP Top 10** (owasp.org/Top10) — the industry-standard reference for the most critical web application security risks; essential ongoing reading beyond what Chapters 4 and 8 had room to cover in full
- **OWASP Application Security Verification Standard (ASVS)** (owasp.org/ASVS) — a far more detailed, checklist-style standard for verifying application security controls than the Top 10 alone provides
- **"The Phoenix Project"** by Gene Kim, Kevin Behr, and George Spafford (IT Revolution Press) — a novel-format introduction to DevOps culture and flow that, read after finishing this entire roadmap, reads less like theory and more like a description of problems you've now actually seen
- **"The DevOps Handbook"** by Gene Kim, Jez Humble, Patrick Debois, and John Willis (IT Revolution Press) — the non-fiction companion to "The Phoenix Project," and the foundational text tying together everything from CI/CD (Topic 5) to the security-integrated culture this course argued for from Chapter 1 onward
- **NIST Secure Software Development Framework (SSDF)** (csrc.nist.gov) — a framework-level view of secure software development practices that complements this course's hands-on, tool-specific approach with a more standards-oriented lens, useful for compliance-mapping work like Chapter 12's
- **The SLSA framework** (slsa.dev) — the canonical reference for supply chain provenance levels introduced in Chapter 7, worth revisiting directly now that you've implemented signing and SBOM generation hands-on in Chapter 16's projects
- **Sigstore documentation** (docs.sigstore.dev) — the authoritative reference for `cosign`, Fulcio, and Rekor internals, going well past this course's introductory keyless-signing coverage
- **CNCF Cloud Native Security Landscape** (landscape.cncf.io, security and compliance category) — now that you've used real projects from it (Falco, OPA/Gatekeeper, and the broader signing/SBOM ecosystem), it's worth revisiting to see the surrounding tooling and what else exists for specialized needs
- **"Site Reliability Engineering" and "The Site Reliability Workbook"** by Google (both free online at sre.google) — already recommended at the end of Monitoring & Logging, and worth a second look now specifically for how both books treat security as one input into an SRE team's broader risk and error-budget thinking, rather than a separate track
- **CIS Benchmarks** (cisecurity.org) — the authoritative source behind `kube-bench` and behind most "hardened baseline" guidance referenced throughout Chapter 10; worth bookmarking directly rather than only encountering secondhand through a scanning tool's output

None of these replace hands-on repetition. Keep the Chapter 16 Project 4 platform running (or rebuild a version of it against a real side project), deliberately try to break past its controls, and see what actually holds and what doesn't — the fastest way to retain a security discipline is to keep testing it against something real, the same way an actual attacker eventually will.

---

## 18.10 Final Self-Assessment

Before you consider the roadmap truly finished, sit with these honestly — not as a pass/fail gate, but as a genuine check on whether the last eighteen chapters (and, in a real sense, the last two hundred) are load-bearing knowledge or things you've merely read once:

- Could you threat-model a system you've never seen before, out loud, in a design review, using STRIDE, without notes?
- Could you explain to a skeptical engineering manager why signing and SBOMs matter, in terms of a real incident class they'd recognize, rather than in abstract security jargon?
- If a critical CVE dropped in a library right now, do you know — for real, not hypothetically — how you'd find out what in your own environment is affected?
- Could you walk into a cluster you've never operated before and run a meaningful first-pass security assessment (RBAC review, NetworkPolicy coverage, admission control status, `kube-bench` run) within an hour?
- If you got paged at 3 a.m. for a Falco alert, would you know your first three actions without having to think, or would you be improvising?
- Looking back across all eleven topics: could you take a bare cloud account and, alone, provision the infrastructure, containerize and deploy an application, build and secure its pipeline, monitor it, and defend it — the complete loop this roadmap set out to teach on page one of Linux Fundamentals?

If most of these feel solid, you're genuinely done — not just with this course, but with the roadmap. If a few feel shaky, that's useful information, not a failure: go back to the specific chapter, redo the hands-on exercise, and come back to this list. The goal was never to read two hundred chapters. It was to come out the other end able to do the job.

---

## 18.11 Progress Tracker

| # | Course | Status |
|---|--------|--------|
| 1 | [Linux Fundamentals](../01-Linux-Fundamentals/00-index.md) | Complete |
| 2 | [Networking Basics](../02-Networking-Basics/00-index.md) | Complete |
| 3 | [Git & Version Control](../03-Git-Version-Control/00-index.md) | Complete |
| 4 | [Docker](../04-Docker/00-index.md) | Complete |
| 5 | [CI/CD Pipelines](../05-CI-CD-Pipelines/00-index.md) | Complete |
| 6 | [Cloud Fundamentals (AWS)](../06-Cloud-Fundamentals-AWS/00-index.md) | Complete |
| 7 | [Infrastructure as Code (Terraform)](../07-Infrastructure-as-Code-Terraform/00-index.md) | Complete |
| 8 | [Kubernetes Basics](../08-Kubernetes-Basics/00-index.md) | Complete |
| 9 | [Advanced Kubernetes](../09-Advanced-Kubernetes/00-index.md) | Complete |
| 10 | [Monitoring & Logging](../10-Monitoring-and-Logging/00-index.md) | Complete |
| 11 | Security (DevSecOps) | Complete — you are here — roadmap finished |

---

You started this roadmap able to type `ls` and `cd`. You finish it able to design a threat model, provision cloud infrastructure declaratively, containerize and orchestrate an application, build and secure the pipeline that ships it, watch it with real metrics and logs, and defend it against a genuine intrusion attempt — and prove, with evidence, that you did all of it correctly. That is roughly two hundred chapters and eleven complete courses of real, cumulative skill, not eleven disconnected certificates. It took real time and real repetition to get here, and it should have — this is a legitimate, complete platform engineering education, not a shortcut. Whatever you build next, you're no longer building it from tutorials. You're building it from a foundation you've now proven you actually own.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./17-interview-preparation.md">← Previous: Interview Preparation</a>
  <a href="./00-index.md">🏠 Index</a>
  <span></span>
</div>
