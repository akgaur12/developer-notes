# Security (DevSecOps) — Complete Course Index

> **DevOps Learning Path — Topic 11 of 11**

---

## Course Overview

Security has been mentioned throughout this entire roadmap without ever getting its own course: Docker's non-root users and image scanning (Topic 4), CI/CD's secrets management (Topic 5), AWS IAM and security groups (Topic 6), Terraform's `tfsec`/Checkov mentions (Topic 7), Kubernetes RBAC and NetworkPolicies (Topic 9), audit logging (Topic 9), and alerting (Topic 10). Every one of those chapters deferred full depth to "Topic 11." This is that course — and the final one in the roadmap.

DevSecOps is the practice of moving security from a final gate before release ("shift right" — a security team reviews everything right before it ships, discovering problems too late and too expensively to fix cheaply) to something built into every stage of the software lifecycle ("shift left" — security checks run automatically in the same pipelines and workflows you already use, from the first commit onward). This course teaches that practice concretely: threat modeling, secrets management in depth, static and dynamic application security testing, software supply chain security (the domain that produced incidents like SolarWinds and the `xz` backdoor), securing the CI/CD pipeline itself, Infrastructure as Code security scanning, a security-focused synthesis of everything you learned about Kubernetes, compliance frameworks, and security incident response.

By the end, you will be able to build security into a DevOps pipeline the way modern engineering organizations actually do it — automated, continuous, and owned by the whole team rather than bolted on by a separate department at the end.

---

## What You'll Be Able to Do

- Explain shift-left security and why DevSecOps treats security as everyone's responsibility, not a separate team's gate
- Threat model a system using STRIDE and prioritize risks realistically
- Design a production-grade secrets management strategy with Vault or a cloud secrets manager, including rotation and leak detection
- Integrate Static Application Security Testing (SAST) into CI without drowning developers in false positives
- Perform Software Composition Analysis (SCA) to find and remediate vulnerable dependencies
- Scan and harden container images, and understand image signing and verification
- Explain software supply chain security: SBOMs, SLSA, provenance, and real-world supply chain attacks
- Run Dynamic Application Security Testing (DAST) against a running application
- Scan Infrastructure as Code for misconfigurations before it's ever applied
- Apply a security-focused lens to everything learned in the Kubernetes courses (RBAC, admission control, NetworkPolicies, audit logs)
- Secure a CI/CD pipeline itself against being the attack vector
- Map security work to compliance frameworks (SOC 2, PCI-DSS, CIS Benchmarks) without treating compliance as a checkbox exercise
- Design security monitoring and alerting, and run an effective incident response process
- Apply production-grade DevSecOps best practices and avoid the most common security mistakes in a DevOps pipeline
- Answer DevSecOps and security engineering interview questions, including incident scenarios

---

## Prerequisites

- **Docker (Topic 4)** — required, specifically Chapter 10 (Security), which this course builds on directly.
- **CI/CD Pipelines (Topic 5)** — required, specifically Chapter 10 (Secrets Management), which this course goes far beyond.
- **Cloud Fundamentals AWS (Topic 6)** — required, specifically Chapter 2 (IAM) and Chapter 10 (Security).
- **Kubernetes Basics and Advanced Kubernetes (Topics 8-9)** — required, specifically Advanced Kubernetes Chapters 2-4 (RBAC, Admission Control/Pod Security, NetworkPolicies) and Chapter 13 (Auditing).
- **Monitoring & Logging (Topic 10)** — recommended, specifically Chapter 7 (Alerting), which this course's security-monitoring chapter builds on.

---

## Estimated Learning Timeline

**4–5 weeks** at 1–2 hours/day.

---

## Learning Milestones

| Milestone | Chapters | Skills Unlocked |
|-----------|----------|-----------------|
| Foundations & Secrets | 01–03 | DevSecOps philosophy, threat modeling, production-grade secrets management |
| Application & Supply Chain Security | 04–08 | SAST, SCA, container security, software supply chain security, DAST |
| Infrastructure & Platform Security | 09–11 | IaC security scanning, Kubernetes security synthesis, CI/CD pipeline security |
| Governance & Operations | 12–13 | Compliance frameworks, security monitoring and incident response |
| Professional | 14–18 | Best practices, avoiding common mistakes, capstone projects, interview-ready |

---

## Full Chapter Index

| # | Chapter | Topics |
|---|---------|--------|
| 01 | [Introduction to DevSecOps](./01-introduction.md) | Shift-left security, the modern threat landscape, recap of security topics scattered across Topics 1-10 |
| 02 | [Threat Modeling](./02-threat-modeling.md) | STRIDE, attack surface analysis, risk prioritization |
| 03 | [Secrets Management](./03-secrets-management.md) | HashiCorp Vault architecture, cloud secrets managers, the External Secrets Operator, rotation, leak detection |
| 04 | [SAST and Static Code Analysis](./04-sast-and-static-analysis.md) | Static Application Security Testing, Semgrep/CodeQL/SonarQube, CI integration, triaging false positives |
| 05 | [Dependency Scanning and SCA](./05-dependency-scanning-and-sca.md) | Software Composition Analysis, vulnerable dependencies, Dependabot/Renovate/Snyk, license compliance |
| 06 | [Container and Image Security](./06-container-and-image-security.md) | Image scanning (Trivy/Grype), minimal/distroless images, image signing and verification |
| 07 | [Software Supply Chain Security](./07-software-supply-chain-security.md) | SBOMs, the SLSA framework, provenance, real-world supply chain attacks, dependency confusion |
| 08 | [DAST and Runtime Security Testing](./08-dast-and-runtime-security-testing.md) | Dynamic Application Security Testing, OWASP ZAP, API security testing, fuzzing |
| 09 | [Infrastructure as Code Security](./09-infrastructure-as-code-security.md) | `tfsec`/Checkov, policy as code (OPA/Conftest), Cloud Security Posture Management |
| 10 | [Kubernetes Security Deep Dive](./10-kubernetes-security-deep-dive.md) | RBAC/admission control/NetworkPolicies through a security lens, CIS Benchmarks (`kube-bench`), runtime security (Falco) |
| 11 | [CI/CD Pipeline Security](./11-cicd-pipeline-security.md) | Securing the pipeline itself, least-privilege CI credentials and OIDC, artifact signing and provenance |
| 12 | [Compliance and Governance](./12-compliance-and-governance.md) | SOC 2/PCI-DSS/CIS frameworks, policy as code for compliance, audit trails |
| 13 | [Security Monitoring and Incident Response](./13-security-monitoring-and-incident-response.md) | SIEM concepts, security alerting, the incident response lifecycle, blameless postmortems |
| 14 | [Best Practices](./14-best-practices.md) | Production-grade DevSecOps patterns |
| 15 | [Common Mistakes and Pitfalls](./15-common-mistakes.md) | The most frequent DevSecOps mistakes, why they happen, how to fix them |
| 16 | [Hands-On Projects](./16-projects.md) | Beginner → intermediate → advanced → production-grade capstone projects |
| 17 | [Interview Preparation](./17-interview-preparation.md) | DevSecOps and security engineering interview questions, incident scenarios |
| 18 | [Course Summary](./18-course-summary.md) | Recap, checklist, quick reference, and roadmap completion |

---

## DevOps Roadmap Series

| # | Topic | Status |
|---|-------|--------|
| 1 | [Linux Fundamentals](../01-Linux-Fundamentals/00-index.md) | ✅ Complete |
| 2 | [Networking Basics](../02-Networking-Basics/00-index.md) | ✅ Complete |
| 3 | [Git & Version Control](../03-Git-Version-Control/00-index.md) | ✅ Complete |
| 4 | [Docker](../04-Docker/00-index.md) | ✅ Complete |
| 5 | [CI/CD Pipelines](../05-CI-CD-Pipelines/00-index.md) | ✅ Complete |
| 6 | [Cloud Fundamentals AWS](../06-Cloud-Fundamentals-AWS/00-index.md) | ✅ Complete |
| 7 | [Infrastructure as Code (Terraform)](../07-Infrastructure-as-Code-Terraform/00-index.md) | ✅ Complete |
| 8 | [Kubernetes Basics](../08-Kubernetes-Basics/00-index.md) | ✅ Complete |
| 9 | [Advanced Kubernetes](../09-Advanced-Kubernetes/00-index.md) | ✅ Complete |
| 10 | [Monitoring & Logging](../10-Monitoring-and-Logging/00-index.md) | ✅ Complete |
| 11 | **Security (DevSecOps)** | 📍 You are here — final course |

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <span></span>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./01-introduction.md">Next: Introduction to DevSecOps →</a>
</div>
