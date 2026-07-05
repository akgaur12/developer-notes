# Chapter 4 — SAST and Static Code Analysis

## Learning Objectives

By the end of this chapter, you will be able to:

- Define Static Application Security Testing (SAST) and explain how it differs from Dynamic Application Security Testing (DAST)
- Describe, with concrete examples, the classes of vulnerabilities SAST is good at catching: injection flaws, hardcoded secrets, insecure deserialization, path traversal, and unsafe API usage
- Explain how SAST tools work conceptually — AST parsing, control/data-flow analysis, and taint tracking from source to sink
- Compare the real tools in this space: Semgrep, CodeQL, SonarQube, and lightweight language-specific linters
- Wire a SAST scan into a GitHub Actions pull request workflow so findings appear as inline review comments
- Diagnose and fix the single biggest reason SAST programs fail in practice: uncontrolled false positives and alert fatigue

## Prerequisites for This Chapter

- Chapter 3 (Secrets Management) — this chapter references SAST as one detection layer for hardcoded secrets, alongside `gitleaks`
- CI/CD Pipelines Chapter 6 (Testing Strategies) — the concept of a CI job that fails a pull request and posts inline annotations
- CI/CD Pipelines Chapters 3–4 — general GitHub Actions workflow syntax
- Monitoring & Logging Chapter 7/15 — the concept of alert fatigue, which reappears here in a security context

---

## 4.1 What SAST Actually Is (and What It Is Not)

Static Application Security Testing (SAST) means analyzing your source code — without ever running it — to find patterns that indicate security vulnerabilities. A SAST tool reads your `.py`, `.js`, `.java`, or `.go` files the way a very fast, very literal-minded senior engineer would during code review, except it never executes a single line.

This is the crucial distinction from **Dynamic Application Security Testing (DAST)**, which you will meet properly in Chapter 8. DAST tests a **running** application from the outside — it sends real HTTP requests to a live endpoint and observes the responses, the way an attacker actually would. SAST never starts your application at all; it only reads the text of your code.

Think of it as the difference between a building inspector reviewing architectural blueprints (SAST — can find a structural flaw before a single brick is laid, but can't tell you the front door actually gets stuck in humid weather) and a building inspector walking through the finished building testing every door and window (DAST — only catches problems in the parts of the building it actually walks through, but every problem it finds is real and reproducible).

| | SAST | DAST |
|---|---|---|
| **What it examines** | Source code (never executed) | A running application (black-box, from outside) |
| **When it runs** | At commit/PR time, before the app even builds | After deployment to a running test environment |
| **Can find** | Vulnerable code patterns anywhere in the codebase, including unreachable/rarely-exercised paths | Only vulnerabilities in code paths actually exercised during the scan |
| **Can't find** | Runtime configuration issues, environment-specific bugs, auth flaws that only manifest at runtime | Vulnerabilities in code paths the scanner never triggers |
| **Speed** | Fast — seconds to a few minutes per PR | Slow — minutes to hours; requires a live environment |
| **False positive tendency** | High — reasons about code without knowing runtime context | Low — a finding usually means the tool actually triggered the behavior |
| **Language/framework awareness** | Deep, requires understanding of source language syntax and semantics | Framework-agnostic; only cares about HTTP in/out |

Neither replaces the other. SAST and DAST are the two halves of automated application security testing — SAST shifts left as far as possible (catch it before it's even built), DAST validates what actually happens when the thing runs. This course covers both; this chapter is entirely about the SAST half.

---

## 4.2 What SAST Actually Catches

SAST tools are not magic — they are pattern matchers with varying degrees of sophistication. Here are the vulnerability classes where they earn their keep, with concrete before/after examples.

### SQL Injection

The classic case. String-concatenating untrusted input directly into a SQL query lets an attacker inject their own SQL.

```python
# WRONG — string concatenation, vulnerable to SQL injection
def get_user(username):
    query = "SELECT * FROM users WHERE username = '" + username + "'"
    return db.execute(query)
# An attacker submitting username = "' OR '1'='1" returns every row in the table.

# RIGHT — parameterized query, the database driver handles escaping
def get_user(username):
    query = "SELECT * FROM users WHERE username = %s"
    return db.execute(query, (username,))
```

A SAST tool flags the first pattern reliably: string concatenation (or f-string/template interpolation) feeding directly into a function known to execute SQL is one of the highest-confidence, lowest-false-positive rules in the SAST world — which is exactly why it's one of the first rules any serious SAST rollout enables.

### Hardcoded Secrets in Source

You already covered secrets management in depth in Chapter 3 — including tools like `gitleaks` that scan git history and commits specifically for credential-shaped strings. SAST tools overlap with this detection surface: because they are already parsing your source code, most SAST rule sets include patterns for API keys, private key headers (`-----BEGIN RSA PRIVATE KEY-----`), and hardcoded database connection strings with embedded passwords.

```javascript
// Flagged by SAST: hardcoded credential pattern
const client = new StripeClient("sk_live_51H8x...");
```

Think of SAST and dedicated secret scanners as overlapping layers, not competitors — `gitleaks` (and GitHub's native secret scanning) are purpose-built and typically faster/more precise for this one problem; SAST tools catch it as a byproduct of already reading every line of code for other reasons.

### Insecure Deserialization

Deserializing untrusted data using a format that can execute arbitrary code during the deserialization process itself (not just after) is a well-known and severe vulnerability class.

```python
# WRONG — pickle can execute arbitrary code during deserialization
import pickle
user_data = pickle.loads(request.body)

# RIGHT — JSON has no code-execution capability
import json
user_data = json.loads(request.body)
```

### Path Traversal

Building a filesystem path from unsanitized user input lets an attacker escape the intended directory using `../` sequences.

```python
# WRONG — attacker can request filename="../../etc/passwd"
def get_file(filename):
    return open(f"/app/uploads/{filename}").read()

# RIGHT — resolve and verify the path stays within the intended directory
import os
def get_file(filename):
    base = os.path.realpath("/app/uploads")
    target = os.path.realpath(os.path.join(base, filename))
    if not target.startswith(base + os.sep):
        raise ValueError("invalid path")
    return open(target).read()
```

### Use of Known-Insecure Functions/APIs

Many languages ship functions that are almost never the right choice in security-sensitive code: `eval()`, `exec()`, PHP's `system()` with unsanitized input, Java's `Runtime.exec()` built from string concatenation, weak cryptographic primitives (`MD5`, `SHA1` for password hashing, `Math.random()` for tokens). SAST rule sets flag calls to these functions by name — a simple, high-confidence detection that doesn't even require taint tracking.

---

## 4.3 How SAST Tools Actually Work

At a conceptual level, a SAST tool does roughly three things:

1. **Parse the source code into an Abstract Syntax Tree (AST).** Instead of treating your code as plain text, the tool builds a structured tree representation of the code's actual grammar — this function call, with these arguments, inside this conditional, inside this function definition. This is exactly the same first step a compiler takes.
2. **Build a control-flow graph and/or data-flow graph.** The control-flow graph maps every possible path execution could take through the code (branches, loops, function calls). The data-flow graph tracks how a given value moves and transforms as it flows through the program.
3. **Pattern-match or trace taint.** Simple rules look for a specific dangerous AST shape (e.g., "a call to `pickle.loads` anywhere"). More powerful analysis performs **taint tracking**: following untrusted data from where it enters the program (a **source**) through every transformation to see if it ever reaches a dangerous operation (a **sink**) without being sanitized along the way.

Taint tracking is the technique that makes SAST capable of catching injection-style vulnerabilities specifically, because the danger in an injection flaw isn't the SQL query itself — it's the fact that *attacker-controlled data* reaches that query unsanitized. A tool that only pattern-matches "string concatenation near a SQL call" without tracing where the string actually came from would flag plenty of perfectly safe code (concatenating two hardcoded constants, for instance) and miss cases where tainted data arrives indirectly through several function calls.

```python
# Annotated taint-tracking example

@app.route("/search")
def search():
    query_param = request.args.get("q")     # ① SOURCE: untrusted, attacker-controlled
    results = run_search(query_param)
    return jsonify(results)

def run_search(term):
    cleaned = term.strip()                  # ② passthrough — taint survives .strip()
    return execute_query(cleaned)

def execute_query(value):
    sql = f"SELECT * FROM items WHERE name = '{value}'"
    return db.execute(sql)                  # ③ SINK: raw SQL execution
```

A taint-tracking SAST tool follows `query_param` (①) through `run_search` and `execute_query`, across two function boundaries, and flags line ③ because tainted data reaches a SQL sink with no sanitization or parameterization in between. This is meaningfully more powerful than a simple text pattern match, and it's the core technique behind tools like CodeQL and, to varying degrees, Semgrep's more advanced rule types.

```
Taint flow:

  request.args.get("q")  ──▶  .strip()  ──▶  f-string SQL  ──▶  db.execute()
  [SOURCE: untrusted]         [passthrough]   [no sanitization]  [SINK: dangerous]

  Result: FLAGGED — tainted data reaches a dangerous sink unsanitized
```

---

## 4.4 The Tools Landscape

| Tool | Approach | Strengths | Best fit |
|---|---|---|---|
| **Semgrep** | Fast, rule-based pattern matching on AST; rules are readable YAML you can write yourself | Very fast, low setup friction, huge open rule registry, easy to add custom org-specific rules | CI on every PR — speed makes it the default choice for fast feedback |
| **CodeQL** | Full semantic code analysis with deep, cross-file taint tracking; treats code as a queryable database | Very powerful, catches complex multi-hop vulnerabilities, same engine behind GitHub's Dependabot code scanning | Deep security-critical analysis, especially in GitHub-native workflows |
| **SonarQube** | Broad code quality + security platform; combines maintainability metrics with a security rule set | Often already deployed for code quality/tech-debt tracking, so security becomes an incremental add-on rather than a new tool to introduce | Orgs that already use it for quality gates and want to consolidate |
| **Bandit** (Python) | Lightweight, Python-specific AST scanner | Narrow scope, very fast, zero configuration | Python-only projects wanting a quick first step |
| **ESLint security plugins** (JS/TS) | Lint rules layered onto the existing ESLint pipeline you already run | Zero new tooling — rides on infrastructure you already have | JS/TS projects wanting security checks without adding a new tool to the pipeline |

Semgrep and CodeQL are the two you'll see most often in modern CI pipelines. Semgrep wins on speed and simplicity of writing custom rules; CodeQL wins on analytical depth for the vulnerabilities that matter most (its taint tracking crosses file and function boundaries more thoroughly). Many mature security programs run both: Semgrep for fast, broad, cheap coverage on every PR, and CodeQL for deeper periodic or release-gating scans. SonarQube is common where it's already installed for code quality reasons — enabling its security rules is a much lower-friction path than introducing a brand-new tool into the pipeline.

---

## 4.5 Integrating SAST into CI

The pattern should feel familiar from CI/CD Chapter 6 (Testing Strategies): SAST is a job in your pipeline, and its findings need to be surfaced with the same review-friction-minimizing pattern already used for test failures and lint errors — inline on the exact line in the pull request, not buried in a report artifact nobody opens.

**GitHub Actions running Semgrep on every pull request:**

```yaml
name: SAST

on:
  pull_request:
    branches: [main]

jobs:
  semgrep:
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep
        run: semgrep ci --sarif --output=semgrep-results.sarif
        env:
          SEMGREP_RULES: p/security-audit p/secrets p/sql-injection

      - name: Upload SARIF to GitHub code scanning
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: semgrep-results.sarif
```

**GitHub Actions running CodeQL** (GitHub's native code scanning integration):

```yaml
name: CodeQL

on:
  pull_request:
    branches: [main]

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript, python

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
```

The crucial workflow detail in both examples is the **SARIF upload**. SARIF (Static Analysis Results Interchange Format) is the standard format that lets GitHub render findings as inline annotations directly on the diff in a pull request — the exact same low-friction, can't-miss-it pattern you already rely on for failing tests in CI/CD Chapter 6. A finding that shows up as a comment on line 47 of the file you're already reviewing gets fixed. A finding buried in a downloadable JSON report artifact gets ignored. This one integration detail is often the difference between a SAST rollout that developers actually engage with and one they route around.

```mermaid
flowchart LR
    A[Developer opens PR] --> B[CI triggers SAST job]
    B --> C[Semgrep / CodeQL scans source]
    C --> D[Results exported as SARIF]
    D --> E[Uploaded to GitHub code scanning]
    E --> F[Findings appear as inline PR annotations]
    F --> G{High-confidence finding?}
    G -->|Yes| H[Developer fixes before merge]
    G -->|No / false positive| I[Triaged and suppressed with justification]
```

---

## 4.6 The False Positive Problem

This is the single biggest reason SAST programs fail in practice, and it deserves to be treated seriously rather than as a footnote.

SAST tools are inherently imprecise because they reason about code **without ever running it**. A tool sees a seemingly dangerous pattern — say, string interpolation feeding into a shell command — but it often cannot tell whether that code path is actually reachable from untrusted input in practice, whether a sanitization function three layers up the call stack already neutralizes the danger, or whether the "vulnerable" function is only ever called with hardcoded, developer-controlled arguments. Without actually executing the program, the tool is making an educated guess about reachability and exploitability, and educated guesses produce noise.

This is precisely the same underlying failure pattern as **alert fatigue**, which you already encountered in Monitoring & Logging (Chapters 7 and 15) in the context of paging and dashboards: when a system produces too many low-value signals, humans don't get more careful — they get numb. They start ignoring the tool wholesale, including the alerts that were real. A monitoring system that pages you 200 times a night trains you to silence your phone. A SAST tool that produces 4,000 findings on day one trains developers to close the PR check and move on without reading it. The mechanism is identical; only the domain changed from uptime to security.

**The triage discipline that separates successful SAST adoption from failed adoption:**

1. **Start with a curated, high-confidence rule set — not everything the tool ships with.** Enable perhaps 10–20 rules covering the highest-severity, lowest-false-positive vulnerability classes (SQL injection, command injection, hardcoded secrets, insecure deserialization). Resist the temptation to flip on the full default rule pack "to be thorough" — thoroughness that nobody reads is not thoroughness, it's noise.
2. **Triage and suppress false positives with a recorded justification, not by silently ignoring them.** Most SAST tools support inline suppression comments (`// nosemgrep: sql-injection — value is a hardcoded enum, not user input`) that document *why* a finding was dismissed. This creates an auditable trail and prevents the same false positive from being re-litigated on every future scan.
3. **Track false-positive rate as a metric you actively work to improve**, the same way you'd track a monitoring system's page-to-incident ratio. If 90% of findings from a given rule are dismissed as false positives, that rule needs tuning or removal — not developers who've learned to click "dismiss" without reading.
4. **Expand rule coverage gradually, only after the current tier is trusted.** Each new rule tier should earn its place by demonstrating an acceptable signal-to-noise ratio on your specific codebase before the next tier is added.

---

## Real-World Scenario

A mid-sized engineering org rolls out a SAST tool with its full default rule set — every rule enabled — across a decade-old legacy codebase. Day one: 4,000 findings appear across the repository. There's no PR review process for the backlog; someone just enables the CI job and lets it scan on the next push. Within a week, developers have learned to click past the SAST check without reading it — a few real, exploitable SQL injection findings are sitting in the noise, indistinguishable from thousands of low-confidence style-adjacent flags, and nobody notices them for months.

The security team steps back and re-scopes. Instead of the full default rule pack, they configure a curated set of roughly 15 rules covering only SQL injection, hardcoded secrets, and command injection — the three highest-severity, highest-confidence categories. They triage the (now much smaller) finding set, fix the handful of genuinely exploitable issues, and document justified suppressions for the rest. Developers start seeing a SAST check that's almost always right when it fires.

Six months later, the same tool that was universally ignored on day one is actively trusted — developers fix findings without being told to, and the team has gradually expanded coverage to roughly 40 rules, each added only after proving itself on a pilot subset of the codebase. Nothing about the underlying tool changed. What changed was the discipline around rollout scope and false-positive triage.

---

## Best Practices

- Start with a small, curated, high-confidence rule set; expand deliberately, never all-at-once
- Route findings into the PR as inline SARIF annotations, not a separate report nobody opens
- Require a documented justification for every suppressed finding, not a silent dismiss
- Track false-positive rate per rule and prune or tune rules that consistently misfire
- Run fast, cheap tools (Semgrep) on every PR; reserve deeper, slower analysis (CodeQL) for scheduled or release-gating scans
- Treat SAST as one layer among several (alongside dedicated secret scanners and SCA, covered next chapter) — no single tool catches everything

## Common Mistakes

- Enabling every default rule on day one and drowning developers in low-confidence noise
- Treating SAST findings as a compliance checkbox — running the scan but never triaging or acting on results
- Silently ignoring or disabling findings instead of documenting why they're false positives
- Running SAST only on a schedule instead of on every pull request, so feedback arrives long after the code was written
- Never revisiting or expanding rule coverage once the initial rule set is chosen

---

## Summary

SAST analyzes source code without running it, using AST parsing and taint tracking to catch injection flaws, hardcoded secrets, insecure deserialization, path traversal, and dangerous API usage — complementing DAST, which tests a running application from the outside (Chapter 8). Semgrep, CodeQL, and SonarQube are the major tools, each with a different speed/depth trade-off. The technical mechanism (taint tracking from source to sink) matters less for adoption success than the rollout discipline: a curated high-confidence rule set with inline PR feedback and disciplined false-positive triage is what separates a SAST program developers trust from one they've learned to ignore — the exact same alert-fatigue failure mode you already saw in monitoring, now applied to security findings.

---

## Knowledge Check

1. A colleague says "DAST is just a slower version of SAST." Explain what's wrong with that statement using the source/sink and running-application distinction.
2. What is taint tracking, and why is it specifically useful for catching injection vulnerabilities rather than, say, hardcoded secret detection?
3. Why does SAST have a structurally higher false-positive tendency than DAST? Answer in terms of what each tool actually observes.
4. A team enables a SAST tool's full default rule set and gets thousands of findings on day one. What two concrete rollout changes would you recommend, and why?
5. Explain the parallel between "alert fatigue" in a monitoring context (Monitoring & Logging Ch. 7/15) and the SAST false-positive problem. What's the shared underlying mechanism?
6. Why does uploading SARIF results to GitHub's code scanning matter more than just generating a report file?

---

## Hands-On Exercise

1. Install Semgrep locally: `pip install semgrep` (or use the `semgrep/semgrep` Docker image).
2. Write a small vulnerable Python or JavaScript file containing at least: one SQL injection pattern (string concatenation into a query), one hardcoded API key, and one use of `eval()`/`pickle.loads()`.
3. Run `semgrep --config=p/security-audit .` against it and review the findings.
4. Fix each finding using the safe pattern shown in this chapter (parameterized query, environment variable / secrets manager reference, safe deserialization) and re-run the scan to confirm a clean result.
5. Add a GitHub Actions workflow (using the example in section 4.5) that runs Semgrep on every pull request and uploads SARIF results. Open a test PR reintroducing one of the vulnerable patterns and confirm it appears as an inline annotation on the diff.
6. Add a `# nosemgrep: <rule-id>` suppression comment to one intentionally-safe pattern that still trips a rule, with a one-line justification comment, and confirm the finding disappears from the next scan.

---

## Further Reading

- [Semgrep Registry — community rule sets](https://semgrep.dev/explore)
- [CodeQL documentation — GitHub](https://codeql.github.com/docs/)
- [OWASP Source Code Analysis Tools list](https://owasp.org/www-community/Source_Code_Analysis_Tools)
- [SARIF specification overview](https://sarifweb.azurewebsites.net/)
- [SonarQube security rules documentation](https://docs.sonarsource.com/sonarqube/)

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./03-secrets-management.md">← Previous: Secrets Management</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./05-dependency-scanning-and-sca.md">Next: Dependency Scanning and SCA →</a>
</div>
