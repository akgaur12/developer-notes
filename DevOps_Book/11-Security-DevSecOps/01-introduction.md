# Chapter 1 — Introduction to DevSecOps

## Learning Objectives

By the end of this chapter you will be able to:

- Explain precisely what DevSecOps is and how it differs from the traditional, siloed security model
- Name the specific chapters across Topics 4, 5, 6, and 9 that deferred security depth to this course, and what each one covered instead
- Explain the "shift-left" metaphor precisely, including why a vulnerability's cost to fix grows the later it's caught
- Describe why the attack surface of a modern DevOps pipeline is larger than that of a traditional, manually-deployed application
- Explain what "everyone owns security" changes about a developer's day-to-day work, concretely
- Describe the five milestone groups this course is organized into and what each unlocks

---

## Prerequisites for This Chapter

- **Docker (Topic 4)**, specifically **Chapter 10 (Security)** — non-root containers, capability dropping, seccomp, image scanning, and secrets basics. This course does not re-teach any of it; it is referenced by chapter number when relevant.
- **CI/CD Pipelines (Topic 5)**, specifically **Chapter 10 (Secrets Management)** — not hardcoding secrets, using a CI system's encrypted secrets store.
- **Cloud Fundamentals AWS (Topic 6)**, specifically **Chapter 2 (IAM)** and **Chapter 10 (Security)** — least-privilege IAM policies, security groups, and basic cloud hardening.
- **Advanced Kubernetes (Topic 9)**, specifically **Chapters 2–4 (RBAC, Admission Control/Pod Security, NetworkPolicies)** and **Chapter 13 (Auditing)**.
- **Monitoring & Logging (Topic 10)**, specifically **Chapter 7 (Alerting)** — this course's later security-monitoring chapter builds directly on it.

If any of those feel shaky, this is the moment to go back — everything in this course assumes fluent recall of them, not a re-introduction.

---

## 1.1 A Promise Made Six Times, Kept Once

If you have worked through this roadmap in order, you have already been told — more than once — that a topic was being deliberately left shallow, with the promise that "Topic 11" would go deeper. It's worth naming every one of those promises explicitly, because this course exists to keep all of them at once, and seeing them side by side is the fastest way to understand why security finally gets its own dedicated course instead of staying scattered as a paragraph here and a paragraph there.

| Where the promise was made | What it covered (shallow) | What was deferred |
|---|---|---|
| **Docker, Chapter 10 (Security)** | Non-root users, read-only filesystems, dropped Linux capabilities, seccomp/AppArmor, `trivy`/Docker Scout image scanning, not hardcoding secrets in images | Software supply chain security, image signing/verification in a production pipeline, integrating scanning as a *gate* rather than a manual step |
| **CI/CD Pipelines, Chapter 10 (Secrets Management)** | Never hardcoding secrets; using your CI system's built-in encrypted secret store (e.g., GitHub Actions secrets) | Dynamic, short-lived secrets; centralized secret management across many systems; automatic rotation; leak detection across git history |
| **Cloud Fundamentals AWS, Chapter 2 (IAM)** | IAM users, roles, policies, least-privilege basics | Threat modeling AWS-specific attack paths; tying IAM into a full DevSecOps program (compliance mapping, CSPM) |
| **Cloud Fundamentals AWS, Chapter 10 (Security)** | Security groups, basic hardening checklist | A structured threat model and ongoing security monitoring/incident response practice |
| **Advanced Kubernetes, Chapters 2–4 (RBAC, Admission Control/Pod Security, NetworkPolicies)** | How to configure each mechanism | Applying all three together through a *security-first lens*, CIS Benchmarks (`kube-bench`), runtime threat detection (Falco) |
| **Advanced Kubernetes, Chapter 13 (Auditing)** | Kubernetes audit logs as one log source | Using audit logs as a countermeasure to Repudiation in a formal threat model; feeding them into a broader security monitoring practice |

None of those chapters were wrong to stay shallow — teaching production-grade Vault architecture in the middle of a Docker course, or STRIDE threat modeling in the middle of an IAM lesson, would have buried the point each of those chapters was actually trying to make. But the debt was real, and it's due now. This course is not eighteen chapters of brand-new, disconnected material — a good third of it is picking up threads you've already been holding and finally tying them together into one coherent discipline: **DevSecOps**.

---

## 1.2 What DevSecOps Actually Is

**DevSecOps** is the practice of integrating security into every stage of the software delivery lifecycle — plan, code, build, test, release, deploy, and operate — using the same automation-first, "everyone owns it" philosophy that DevOps originally applied to the relationship between development and operations.

That definition is dense, so unpack the two halves separately, because each one is doing real work.

**"Every stage of the lifecycle"** means security is not a single checkpoint. It is a set of automated checks distributed across planning (threat modeling, Chapter 2), coding (SAST, Chapter 4), building (dependency scanning and container scanning, Chapters 5–6), testing (DAST, Chapter 8), releasing (supply chain provenance, Chapter 7), deploying (IaC scanning, Chapter 9; Kubernetes security, Chapter 10), and operating (monitoring and incident response, Chapter 13). No single stage is "the security stage" — every stage has a security stage *within* it.

**"Everyone owns it"** is the cultural half, and it is arguably the more important one, because tooling without ownership just becomes noise nobody acts on. DevOps's original insight was that "throw code over the wall to Ops" produced slow, blame-driven, unreliable software — the fix wasn't a better wall, it was making developers and operators share the same goals and the same on-call rotation. DevSecOps applies the identical insight to security: "throw code over the wall to a separate Security team for review right before release" produces slow, blame-driven, insecure software for exactly the same structural reasons. The fix is the same, too — security becomes something the whole delivery team is responsible for continuously, not a department that shows up once per release cycle with a clipboard.

### Contrast with the traditional model

Picture the traditional, pre-DevSecOps model concretely, because its failure mode is what motivates everything else in this course:

A development team spends six weeks building a feature. Three days before the scheduled release, they hand the build to a separate security team, who run a manual penetration test and a checklist review. The security team finds a dozen issues — some serious, some cosmetic — and produces a report. Now the delivery team faces an impossible choice: delay a release that's already been promised to stakeholders while they fix a dozen issues discovered at the last possible moment, or ship anyway and "fix it in the next release," which is corporate-speak for "we accepted this risk without ever explicitly deciding to."

This model fails for structural reasons, not because the security team is bad at its job:

- **It's late.** By the time security reviews the build, every architectural decision has already been made and is expensive to unwind. A vulnerability that would have taken ten minutes to avoid at the design stage now requires reworking code that's already been tested, reviewed, and scheduled for release.
- **It's a bottleneck.** One security team reviewing every release from every delivery team in the organization does not scale. As the number of teams and release frequency grows, the security team becomes the queue everyone waits behind.
- **It creates adversarial incentives.** When security is a gate that can block a release, the delivery team's incentive is to get past the gate, not to actually be secure — findings get argued down, deadlines create pressure to accept risk quietly, and the whole relationship becomes friction instead of collaboration.
- **It gets bypassed under deadline pressure.** The predictable end state of a late, manual, adversarial gate is that it gets skipped — quietly, "just this once," when a release absolutely has to ship on a Friday. That "just this once" is how most of the ugliest production security incidents actually happen.

DevSecOps isn't a rebrand of this same process with more automation sprinkled on. It is a structural rejection of the "gate near the end" model in favor of continuous, distributed, largely automated checks that run in the same pull requests and pipelines developers already use — which is exactly what "shift-left" means, covered next.

---

## 1.3 Shift-Left Security

### The metaphor, precisely

Picture the software delivery lifecycle as a single timeline running left to right: **plan → code → build → test → release → deploy → operate**. In the traditional model described above, the one meaningful security check sits far to the right, right before deploy — a security review gate squeezed in at the last responsible moment. "Shifting left" means moving security activity as far left on that timeline as it can meaningfully go: catching problems during planning and coding, not during a pre-release scramble.

```
Traditional model — security concentrated at the right edge:

 plan ─── code ─── build ─── test ─── release ─────[ SECURITY REVIEW ]──── deploy ─── operate
                                                      (manual, late, one gate)


DevSecOps — security distributed across the whole timeline:

 plan ──── code ──── build ──── test ──── release ──── deploy ──── operate
  │         │          │          │          │           │           │
threat    SAST on   dependency  DAST on   SBOM /      IaC scan /  monitoring,
modeling  every PR   & image    staging   provenance  policy as   audit logs,
(Ch 2)    (Ch 4)     scanning   (Ch 8)    (Ch 7)       code (Ch 9,  alerting,
                     (Ch 5-6)                          Ch 10-11)   IR (Ch 13)
```

The shift isn't just "do the same review earlier." It's "replace one big manual review with many small automated ones, each running exactly where the relevant decision is being made."

### Why "earlier" is dramatically cheaper, quantified

The reason shift-left is worth an entire architectural philosophy, rather than just a nice idea, is that the cost of fixing a vulnerability grows enormously the later it's discovered — and the growth isn't linear, it's closer to exponential.

- **Caught by a SAST scan on a pull request (Chapter 4):** the developer who wrote the vulnerable code sees an inline annotation on their own PR, still has full context on why they wrote it that way, and fixes it in a few minutes before it's ever merged. Cost: minutes of one engineer's time.
- **Caught by a dependency scanner before a release ships (Chapter 5):** an automated PR bumps a vulnerable library version; a maintainer reviews and merges it. Cost: minutes to hours, mostly automated.
- **Caught in production, after an attacker finds it first:** now the cost includes incident response (multiple engineers, often paged outside business hours), forensic investigation (what data was actually accessed?), customer notification obligations, reputational damage, and — depending on the data involved — regulatory fines under frameworks like GDPR or PCI-DSS (previewed further in Chapter 12). Cost: potentially weeks of engineering time, direct financial penalties, and damage to customer trust that doesn't show up on any invoice but shows up in churn.

The same underlying bug — say, a SQL injection flaw — might cost ten minutes to fix as a PR comment and six figures to remediate after a breach. Shift-left isn't a philosophy adopted out of idealism; it's adopted because the economics are this lopsided.

---

## 1.4 Why the Modern Attack Surface Is Larger Than It Used to Be

This isn't fearmongering for its own sake — it's the specific motivation for why this course's chapters exist in the order they do. Every practice you've adopted across this roadmap to move faster has also, as a side effect, given attackers more places to try.

- **More dependencies.** A modern application might pull in hundreds or thousands of open-source packages, each with its own maintainers, its own release cadence, and its own potential vulnerabilities — and increasingly, its own potential for outright malicious compromise (the domain of **Chapter 7 — Software Supply Chain Security**, which covers incidents like SolarWinds and the `xz` backdoor). You don't just trust your own code anymore; you implicitly trust everyone who maintains anything you depend on, transitively.
- **More infrastructure defined as code.** Terraform (Topic 7) means your cloud infrastructure — networks, IAM policies, storage buckets, security groups — is now just text files that can be reviewed, but also just text files that can contain a misconfiguration nobody notices until `terraform apply` has already created a publicly-readable S3 bucket. **Chapter 9 — Infrastructure as Code Security** covers scanning that IaC before it's ever applied.
- **More automated pipelines with broad permissions.** Your CI/CD pipeline (Topic 5) almost certainly holds credentials to deploy to production, pull private packages, and push container images — which makes the pipeline itself an extremely high-value attack target. If an attacker can get malicious code to run inside your CI job, they inherit whatever that job is trusted with. **Chapter 11 — CI/CD Pipeline Security** covers securing the pipeline as an asset in its own right, not just a means of shipping other assets.
- **More services communicating over networks.** Kubernetes (Topics 8–9) turned one monolithic process into a dozen or a hundred services, each talking to the others over the network, each a potential point of compromise, each requiring its own access control. **Chapter 10 — Kubernetes Security Deep Dive** synthesizes everything Advanced Kubernetes taught about RBAC, admission control, and NetworkPolicies specifically through this lens.

The pattern across all four: practices that made teams faster and more independent — dependency reuse, IaC, CI/CD automation, microservices — each expanded the number of places where something can go wrong. DevSecOps doesn't ask you to give any of that up; it gives you the automated checks that let you keep moving fast without moving blind.

```
A decade ago                              Today

┌─────────────────────┐                   ┌─────────────────────────────────────┐
│   One monolith       │                   │  Dozens of microservices (K8s)       │
│   A handful of        │                  │  Hundreds–thousands of dependencies   │
│    dependencies      │        ──▶        │  Infrastructure defined as code       │
│   Manual deploys      │                  │  Automated CI/CD with broad creds     │
│   One data center    │                   │  Multi-cloud, dynamic, autoscaled     │
└─────────────────────┘                   └─────────────────────────────────────┘
   Small, enumerable                          Large, combinatorial
   attack surface                             attack surface
```

None of the changes on the right are mistakes — they are exactly the practices this entire roadmap taught you to adopt, because they make teams faster and systems more resilient to ordinary operational failure. The point of naming this growth explicitly is narrower: every one of this course's later chapters exists to put a specific, automated guardrail around one of these specific expansions, so that "faster" doesn't have to mean "blinder."

---

## 1.5 "Everyone Owns Security": What Actually Changes Day to Day

It's easy to nod along with "security is everyone's responsibility" as an abstract value statement and have nothing about your actual workflow change. So make it concrete: here is what changes for a single developer, on a single afternoon, in an organization that has actually adopted DevSecOps versus one that has merely said the words.

**Without DevSecOps:** A developer opens a pull request, gets it approved by a teammate, and merges it. Three weeks later, during a scheduled quarterly security review, the security team runs a scan across the whole codebase and emails a spreadsheet listing 40 findings to the engineering manager. Nobody remembers the context of the code by then. The findings get triaged into a backlog, most of which never gets prioritized against feature work, because there's no mechanism connecting "this is a security finding" to "this is urgent."

**With DevSecOps:** The same developer opens the same pull request. A SAST tool (Chapter 4) has already run automatically as a CI check and posted an inline comment directly on the line of code that introduced a potential SQL injection, with a suggested fix. The developer — who still has full context, having written the code minutes ago — fixes it before requesting review, the same way they'd fix a failing unit test. No spreadsheet, no meeting, no three-week delay, and no separate team involved at all for this particular finding.

That's the entire cultural shift in miniature: security feedback arrives in the same place, at the same speed, and to the same person as every other kind of engineering feedback (a failing test, a linter warning, a code review comment) — rather than arriving later, to a different person, through a different process. This is precisely why "shift-left" and "everyone owns it" are two descriptions of the same underlying change, not two separate initiatives: moving the check earlier is *what makes* ownership by the original developer possible in the first place.

### If everyone owns it, what does a security team even do?

A fair question, since "everyone owns security" can sound like it's arguing a dedicated security team becomes unnecessary. The opposite is true — the role changes, rather than disappearing, in a way that should feel familiar because it's the exact same shift Ops went through when DevOps first displaced the "throw it over the wall to Ops" model.

| | Traditional Model | DevSecOps Model |
|---|---|---|
| Security team's main activity | Manually reviewing/testing releases one at a time, near the end | Building and maintaining the automated tooling, policies, and guardrails every team uses continuously |
| Who fixes a given finding | The security team files a ticket; someone else (eventually) fixes it | The developer who introduced it, immediately, in their own PR |
| How security expertise scales | Doesn't — one team's headcount is a hard ceiling on how many releases can be reviewed | Scales with the organization — the tooling reviews every PR, every build, every deploy, automatically |
| What the security team is expert in | Manual review technique for a given release | Platform engineering: building the SAST/SCA/IaC-scanning/secrets pipeline that every team plugs into |

Many organizations formalize the "everyone owns it" model with a **Security Champion** — a volunteer or designated engineer embedded within each product team who has slightly deeper security training than their teammates, acts as the first point of contact for security questions within that team, and represents that team's context back to the central security/platform team. This is a scaling mechanism, not a loophole: it means the (necessarily small) central security team's expertise reaches every team without that central team needing to review every single change personally — a structural solution to the exact bottleneck problem described in section 1.2.

---

## 1.6 Course Map

This course is organized into five milestone groups, mirroring the way Advanced Kubernetes and Monitoring & Logging structured their own courses.

```
                         SECURITY (DevSecOps) — Topic 11
                                      │
     ┌───────────────┬───────────────┼────────────────┬──────────────────┐
     ▼                ▼               ▼                ▼                  ▼
┌───────────┐  ┌──────────────┐ ┌───────────────┐ ┌─────────────┐  ┌──────────────┐
│FOUNDATIONS │  │ APPLICATION & │ │INFRASTRUCTURE │ │ GOVERNANCE & │  │ PROFESSIONAL │
│ & SECRETS  │  │ SUPPLY CHAIN  │ │  & PLATFORM   │ │  OPERATIONS  │  │              │
│ Ch 01–03   │  │  Ch 04–08     │ │  Ch 09–11     │ │  Ch 12–13    │  │  Ch 14–18    │
├───────────┤  ├──────────────┤ ├───────────────┤ ├─────────────┤  ├──────────────┤
│DevSecOps   │  │ SAST          │ │ IaC scanning  │ │ Compliance   │  │ Best         │
│philosophy  │  │ SCA / deps    │ │ K8s security  │ │  frameworks  │  │  practices   │
│Threat      │  │ Container/    │ │  synthesis    │ │ Security     │  │ Common       │
│ modeling   │  │  image sec.   │ │ CI/CD pipeline│ │  monitoring  │  │  mistakes    │
│Secrets mgmt│  │ Supply chain  │ │  security     │ │ Incident     │  │ Projects,    │
│            │  │ DAST          │ │               │ │  response    │  │  interview   │
└───────────┘  └──────────────┘ └───────────────┘ └─────────────┘  └──────────────┘
"What is       "Find and fix    "Secure the       "Prove it and    "Package it all
 DevSecOps,     what's wrong     platform your     respond when     up: patterns,
 and how do     with the code    code runs on"     it fails"        pitfalls, and
 we manage      and what it                                         proof of skill"
 our secrets?"  depends on"
```

Each group is a prerequisite for the next in spirit, if not always in strict technical dependency: threat modeling (Chapter 2) gives you the vocabulary and mental model that every later chapter's "why does this matter" section leans on, and secrets management (Chapter 3) is foundational because nearly every later chapter — CI/CD security, Kubernetes security, IaC security — assumes credentials are already being handled correctly.

---

## 1.7 Real-World Scenario: Two Companies, One CVE

A critical remote-code-execution vulnerability is publicly disclosed in a popular open-source logging library — the kind of library that quietly ends up as a transitive dependency of a huge fraction of backend services, the exact category of incident that motivated **Chapter 7 (Software Supply Chain Security)** to exist as its own chapter. Both companies below run around 200 backend services. Neither wrote the vulnerable library. Both are affected. Their responses diverge completely.

**Company A — no DevSecOps practices.** Nobody has a reliable inventory of which of the 200 services use the vulnerable library, directly or transitively — dependency lists live wherever each team happens to keep them, if they're tracked at all. Security sends an urgent email asking every team to check their own services "as soon as possible." Some teams respond within hours; others don't see the email for days. A manual, service-by-service audit, coordinated over Slack and spreadsheets, takes three weeks to reach reasonable confidence that every affected service has been found — and even then, nobody is fully sure the list is complete, because there's no systematic way to check. During those three weeks, an attacker who has been scanning the internet for the same vulnerability (a matter of public record, disclosed to everyone simultaneously) finds and exploits one of the unpatched services, resulting in a breach.

**Company B — DevSecOps practices in place.** Every service in the company already produces a **Software Bill of Materials (SBOM)** as part of its build pipeline, and a **Software Composition Analysis (SCA)** system (both covered in Chapters 5 and 7) continuously cross-references every service's dependency list against newly disclosed vulnerabilities. Within minutes of the CVE's public disclosure, the SCA system flags every one of the — say — 14 services that actually depend on the vulnerable library, directly or transitively, with zero manual auditing required. Automated dependency-update tooling (Dependabot/Renovate, Chapter 5) opens pull requests bumping the library to the patched version in each affected service. Most of those PRs pass CI automatically and are merged within hours; a couple require a human to confirm a minor API change and are merged by the end of the day. The organization is fully patched before most companies have even finished manually figuring out whether they're affected at all.

Same vulnerability, same disclosure, same starting technology. The difference in outcome — hours versus weeks, and a near-miss versus an actual breach — is entirely explained by whether an SBOM/SCA system existed *before* the vulnerability was disclosed. You cannot build that system reactively, in the three weeks after a CVE drops; it only works if it was already running continuously, which is precisely the "everyone owns it, continuously, automated" DevSecOps philosophy in action.

| | Company A (no DevSecOps) | Company B (DevSecOps in place) |
|---|---|---|
| Knowing which services are affected | Manual, service-by-service audit via Slack and spreadsheets | SCA system flags every affected service automatically, in minutes |
| Time to identify full blast radius | ~3 weeks, with lingering uncertainty about completeness | Minutes |
| Who does the patching work | Every team, manually, in whatever order they get to it | Automated PRs open themselves; most merge with no human intervention |
| Time to fully patched | Unknown — a breach occurred before the audit even finished | Hours to one business day |
| What the CVE disclosure triggers | A scramble and an all-hands email | A routine, mostly-automated remediation workflow |

Notice that Company B's advantage isn't a smarter security team — it's that the *infrastructure for answering the question* was built in advance, continuously, as a byproduct of ordinary DevSecOps practice, rather than assembled from scratch under pressure once the question suddenly mattered.

---

## 1.8 A Maturity Model: Where an Organization Typically Starts

It's worth setting expectations honestly: almost no organization adopts every practice in this course simultaneously, and this course doesn't ask you to either. DevSecOps adoption tends to follow a recognizable crawl-walk-run progression, and it's useful to know roughly where a given organization — including, possibly, your own — sits on it, because the next reasonable investment differs a lot depending on the starting point.

| Stage | What's True | Typical Gaps |
|---|---|---|
| **Ad hoc** | Security is whatever individual engineers happen to remember to do; no automated checks exist in the pipeline at all | No SAST/SCA, no threat modeling, secrets often handled inconsistently |
| **Foundational** | Basic automated gates exist — a SAST scanner runs in CI, secrets live in a CI secrets store, container images get scanned before push | Findings often generate noise nobody triages; no centralized secrets management; no formal threat modeling |
| **Integrated** | Security checks are a normal part of the developer's own feedback loop (inline PR comments, not a separate report); secrets are centralized and partially dynamic; IaC is scanned before apply | Governance and compliance mapping still manual; incident response is reactive rather than rehearsed |
| **Optimized** | Security telemetry feeds directly into monitoring and alerting; compliance evidence is generated automatically as a byproduct of normal operations; incident response is rehearsed and continuously improved | Mostly a matter of refining thresholds, reducing false positives, and expanding coverage to new services |

This course's eighteen chapters roughly trace that same path — Chapters 1–3 establish the philosophy and the two most foundational practices, Chapters 4–11 build out the "Integrated" stage's automated gates across the whole lifecycle, and Chapters 12–13 build the "Optimized" stage's governance and response capabilities. Very few readers will find their own organization already at "Optimized" — that's normal, not a sign of failure, and the maturity model exists to help you have an honest, specific conversation about what to invest in next, rather than trying to do everything in this course at once on day one.

---

## 1.9 Vocabulary You'll Hear Throughout This Course

Before moving on, it's worth anchoring a handful of terms that will recur constantly for the rest of this course — not to master them yet, but so they don't feel like unexplained jargon when a later chapter uses them.

| Term | Plain-Language Meaning | Where It's Covered in Full |
|---|---|---|
| **SAST** | Static Application Security Testing — scanning source code itself for vulnerable patterns, without running it | Chapter 4 |
| **SCA** | Software Composition Analysis — scanning your dependency list for known-vulnerable versions | Chapter 5 |
| **SBOM** | Software Bill of Materials — a complete, machine-readable inventory of everything a piece of software depends on | Chapters 5, 7 |
| **DAST** | Dynamic Application Security Testing — attacking a *running* application from the outside, the way a real attacker would | Chapter 8 |
| **SLSA** | Supply-chain Levels for Software Artifacts — a framework for proving how and where an artifact was actually built | Chapter 7 |
| **CSPM** | Cloud Security Posture Management — continuously checking live cloud infrastructure for misconfigurations | Chapter 9 |
| **CIS Benchmark** | An industry-standard, prescriptive hardening checklist for a given technology (e.g., Kubernetes) | Chapter 10 |
| **SIEM** | Security Information and Event Management — a system that centralizes and correlates security-relevant logs across an organization | Chapter 13 |

You do not need to memorize this table now — treat it as a map to return to whenever a later chapter uses a term you've half-forgotten.

---

## 1.10 Self-Assessment: Where Does Your Organization Actually Stand?

Before moving into Chapter 2, it's worth turning the maturity model from section 1.8 into a concrete, honest checklist about a real system you're responsible for (or have worked on), rather than leaving it abstract:

1. **If a critical CVE were disclosed in a library you depend on today, how long would it take you to know whether — and where — you're affected?** Minutes (an SCA/SBOM system already answers this) or days/weeks (a manual audit would be required)?
2. **When a security scanner finds something, who sees the result, and how fast?** The engineer who wrote the code, within the same pull request — or a separate team, weeks later, via a report?
3. **Are your secrets static and long-lived, or dynamic and short-lived — and do you know, right now, how many systems would need to change if a single shared credential leaked today?**
4. **Has anyone on your team ever deliberately threat-modeled the system you work on, or has security review only ever happened reactively, after something went wrong?**
5. **If asked tomorrow to prove compliance with a framework relevant to your industry (SOC 2, PCI-DSS), could you produce the evidence quickly, or would it require weeks of manual log-gathering?**

Most real organizations will answer "the slower, more manual option" to at least a few of these — that's the normal starting point, not a special failure, and it's exactly why this course exists. Keep your honest answers in mind; Chapters 2 and 3 begin closing the gap on the two most foundational ones (threat modeling and secrets), and the rest of the course closes the others in turn.

---

## Best Practices

- Treat this course's chapter order as a deliberate progression: threat modeling and secrets management (Chapters 2–3) are the foundation the application and infrastructure security chapters lean on — don't skip ahead expecting them to stand alone.
- Whenever a later chapter introduces a scanning tool or gate, ask "where on the shift-left timeline does this run, and who sees the result first?" — the earliest, most direct feedback loop is almost always the right one to build first.
- Revisit the specific prior-course chapters named in section 1.1 as needed; this course references them by number precisely so you can jump back rather than relying on memory.
- Read each chapter's Real-World Scenario as motivation, not decoration — DevSecOps tooling only makes sense in light of the specific failure mode it's designed to prevent.

## Common Mistakes

- Treating "DevSecOps" as a rebrand of the old "security team reviews before release" process with a new name and a scanner bolted on, rather than a structural change in *when* and *by whom* security is handled.
- Assuming shift-left means moving the exact same heavyweight review earlier, rather than replacing one big manual gate with many small automated checks distributed across the pipeline.
- Believing "everyone owns security" requires every developer to become a security expert, rather than requiring tooling that surfaces expert-level findings directly and immediately to the person best positioned to fix them.
- Underestimating how much of the modern attack surface is a direct side effect of practices that also make teams faster (dependency reuse, IaC, CI/CD automation) — and concluding the fix is to slow down, rather than to add the automated guardrails this course covers.

---

## Summary

DevSecOps integrates security into every stage of the software delivery lifecycle — plan, code, build, test, release, deploy, operate — using the same automation-first, shared-ownership philosophy DevOps applied to development and operations. It replaces the traditional model of a single, late, manual security gate (slow, adversarial, and prone to being bypassed under deadline pressure) with continuous, automated checks distributed across the pipeline, a philosophy captured in the term "shift-left": catching a vulnerability on a pull request costs minutes, while catching the same vulnerability in production after a breach costs incident response time, customer trust, and potentially regulatory fines. The modern attack surface has grown specifically because the practices that make teams faster — more open-source dependencies, infrastructure as code, automated CI/CD pipelines with broad permissions, and networked microservices — each introduce new places for something to go wrong, which is exactly why this course's later chapters exist. Six prior courses in this roadmap each deferred security depth to "Topic 11" — Docker's container hardening, CI/CD's basic secrets handling, AWS's IAM and security groups, and Advanced Kubernetes's RBAC, admission control, NetworkPolicies, and auditing — and this course is where all of those threads are picked up, deepened, and tied together into one coherent discipline, organized into five milestone groups: Foundations & Secrets, Application & Supply Chain Security, Infrastructure & Platform Security, Governance & Operations, and Professional.

---

## Knowledge Check

1. Name three specific chapters from prior courses (Topic 4, 5, 6, or 9) that explicitly deferred security depth to this course, and state what each covered instead.
2. In your own words, explain the "shift-left" metaphor, and give a concrete example of the cost difference between catching a vulnerability early versus late.
3. Why does the traditional model of "a separate security team reviews right before release" tend to get bypassed under deadline pressure? Identify the structural reason, not just "people get lazy."
4. Name two DevOps-adjacent practices (from earlier topics in this roadmap) that have each expanded the modern attack surface, and explain how.
5. Describe, concretely, what changes for an individual developer's day-to-day workflow under "everyone owns security" — use a specific example, not an abstract statement.
6. Name the five milestone groups this course is organized into and one skill unlocked by each.
7. Using the maturity model in section 1.8, honestly place a real or hypothetical organization you know at one of the four stages, and identify the single next investment that would move it forward.
8. What does a security team's role become in a mature DevSecOps organization, if it's no longer "review every release manually"? What is a Security Champion, and what structural problem does that role solve?

---

## Hands-On Exercise

No tooling installation is required for this introductory chapter — it is a research and self-assessment exercise to orient you before Chapter 2.

1. Pick a real or hypothetical application you're familiar with. Write down, honestly, where on the shift-left timeline (plan/code/build/test/release/deploy/operate) security is currently checked, if at all. Is it concentrated near the right edge (a late manual review, or nothing at all) or distributed across the timeline?
2. Search for a public postmortem or incident writeup involving a vulnerable open-source dependency (search `"[library name] CVE" postmortem` or check the GitHub Security Advisories database for a well-known incident). Identify how long it took the affected organization to find and patch every instance of the vulnerable dependency, and whether an SBOM/SCA system (as described in section 1.7) would have changed that timeline.
3. Revisit Docker Chapter 10, CI/CD Chapter 10, AWS Chapters 2 and 10, and Advanced Kubernetes Chapters 2–4 and 13 for five minutes each. For each, write one sentence connecting what it taught to the chapter in this course that will go deeper on the same topic (use the table in section 1.1 as a guide, but write it in your own words).
4. Complete the five-question self-assessment in section 1.10 for a real system you have access to (work, personal, or open-source). Write down your honest answers, and identify which single answer represents the biggest immediate risk.

---

## Further Reading

- owasp.org/www-project-devsecops-guideline/ — the OWASP DevSecOps Guideline, a practical, vendor-neutral reference for the practices this course covers
- dora.dev — the DORA (DevOps Research and Assessment) research program's findings on how security integrates with high-performing software delivery
- nist.gov/publications/secure-software-development-framework-ssdf — NIST's Secure Software Development Framework, a widely referenced standard for shift-left practices
- cncf.io/reports/devsecops/ — CNCF's cloud-native DevSecOps whitepaper, focused on the cloud-native and Kubernetes ecosystem this course builds on
- sonarsource.com/learn/devsecops/ — an accessible industry primer that complements this chapter's shift-left cost framing with additional data points

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./00-index.md">← Previous: Index</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./02-threat-modeling.md">Next: Threat Modeling →</a>
</div>
