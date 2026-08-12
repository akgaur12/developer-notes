# Chapter 16: Interview Preparation

## Learning Objectives

- Answer conceptual, scenario-based, and system-design OAuth questions confidently
- Practice troubleshooting a broken OAuth integration from symptoms alone
- Know the "gotcha" questions that separate tutorial-level knowledge from working knowledge

## Prerequisites for This Chapter

The entire course.

## Frequently Asked Conceptual Questions

**Q: What's the difference between authentication and authorization, and where does OAuth fit?**
A: Authentication proves identity; authorization grants permission to act. OAuth 2.0 is fundamentally an authorization protocol — it doesn't itself prove who's logging in; OpenID Connect adds that on top via the ID token. (Chapters 1, 8)

**Q: Why does PKCE exist, and is it only for mobile apps?**
A: It protects the authorization code exchange from being hijacked when a client can't hold a secret safely. Originally aimed at public clients (mobile, SPA, desktop), OAuth 2.1 now mandates it for *all* clients, confidential or not, because it's a strict security improvement with no real downside. (Chapter 4)

**Q: What's the difference between an access token and a refresh token?**
A: Access tokens are short-lived and sent on every API call to a Resource Server; refresh tokens are long-lived, sent only to the Authorization Server, and used purely to mint new access tokens without re-prompting the user. (Chapter 5)

**Q: Explain refresh token rotation and why it matters.**
A: Every use of a refresh token invalidates it and issues a new one. This lets an Authorization Server detect theft: if an already-rotated-out token is presented again, that's a signal of compromise, since the legitimate client should already be holding the newer one. (Chapter 5)

**Q: What is the "confused deputy" problem?**
A: A service with legitimate delegated authority gets tricked (often via blind token forwarding) into misusing that authority for a purpose the resource owner never intended — e.g., a compromised tool replaying a token meant for a different, unrelated API. (Chapter 11)

**Q: Why would you use the Client Credentials grant instead of Authorization Code?**
A: When there's no human resource owner involved — pure machine-to-machine access, like a backend service or IoT gateway authenticating to an API on its own behalf. (Chapter 7)

**Q: What's the actual difference between OAuth and OpenID Connect?**
A: OAuth authorizes API access (access token, consumed by a Resource Server); OIDC adds a standardized login/identity layer on top (ID token, consumed by the client itself, plus a `/userinfo` endpoint). (Chapter 8)

## Scenario-Based Questions

**Scenario: "Our mobile app stores its OAuth `client_secret` hardcoded in the app binary. Is this a problem?"**
A: Yes — mobile apps are public clients; any embedded secret is extractable via decompilation. It provides no real security and should be removed in favor of PKCE, which doesn't rely on a secret being kept confidential. (Chapters 3, 13)

**Scenario: "A user reports being logged out unexpectedly every hour, even though they're actively using the app."**
A: This points to either (a) the access token TTL being too short relative to how the client refreshes, or more likely (b) the client isn't implementing the refresh flow at all and is instead forcing a full re-login on every 401 instead of silently exchanging the refresh token. Walk through: check if a refresh_token was even issued (was `offline_access`/appropriate scope requested?), then check the client's 401-handling logic. (Chapter 5, 9)

**Scenario: "We have one company SSO (Authorization Server) and five internal APIs. How do we stop a token issued for API A from working against API B?"**
A: Implement audience binding — request tokens scoped to a specific resource (RFC 8707-style `resource` parameter), and have every API validate the `aud` claim against its own identity before accepting a request, not just signature and expiry. (Chapters 6, 10, 11)

**Scenario: "Users complain the 'Allow' consent screen is scary — it lists a huge list of permissions for what should be a simple integration."**
A: Likely over-broad scope requests. Review what scopes are actually being requested vs. what the integration uses, and narrow them — this is both a security practice and a UX one. (Chapters 2, 12, 13)

## System Design Discussion

**"Design an OAuth-based authorization system for a platform where multiple third-party developers can build apps that access user data, similar to how Claude/Kimi connect to MCP servers."**

Key points an interviewer wants to hear:
1. Distinguish the four roles clearly: your platform's users (Resource Owners), third-party apps (Clients), your own identity system (Authorization Server), and your APIs (Resource Servers).
2. Use Authorization Code + PKCE for all clients, since you can't assume any given third-party app is confidential.
3. Scope design: granular, action-based scopes shown clearly on a consent screen, not one blanket "full access" grant.
4. Token architecture: short-lived signed JWT access tokens (so your many Resource Servers can validate independently without hitting the AS on every request) + rotating refresh tokens.
5. Audience binding if you have multiple distinct APIs, to prevent one compromised third-party integration from replaying tokens against unrelated APIs.
6. A developer-facing dashboard for revoking a specific app's access (maps to real token revocation support).
7. Observability: monitor token issuance and validation-failure rates broken out by cause.
8. Mention Dynamic Client Registration only if third-party apps need to self-register without your team manually provisioning each `client_id` — and note (correctly) that it's a convenience, not a requirement, if asked about MCP specifically.

## Practical Troubleshooting Exercises

**Exercise 1:** A client reports `invalid_grant` errors intermittently on token exchange. List at least three distinct root causes consistent with this single error code, and how you'd distinguish between them via logging.
*(Expected reasoning: expired/already-used authorization code, PKCE verifier mismatch, or a rotated-out refresh token being reused — each needs different logging to tell apart: code age vs. verifier hash comparison vs. refresh-token generation tracking.)*

**Exercise 2:** Users on a corporate network report the OAuth login redirect fails with a browser error, while users elsewhere are unaffected. What would you check first?
*(Expected reasoning: likely a corporate proxy/firewall blocking the redirect_uri's domain or port, especially if it's a `localhost` callback listener as in desktop clients like Claude's MCP integration — check firewall/proxy rules before assuming an application bug.)*

**Exercise 3:** A security review flags that your Resource Server accepts any validly-signed JWT from your company's identity provider, regardless of which specific internal API it was minted for. What's the fix, and which RFC formalizes the right mechanism?
*(Expected answer: implement audience validation using Resource Indicators, RFC 8707 — Chapters 6, 10, 11, 14.)*

## Real-World Production Cases to Know

- **The historical "Facebook Connect" login-spoofing class of bugs**, where apps used raw API-call success as a login check instead of a proper identity token — the reason OIDC's ID token exists as a distinct artifact from the access token. (Chapter 8)
- **Refresh token reuse detection in production identity providers** (Okta, Auth0, Google all implement rotation-based theft detection) — know that this is now industry-standard, not experimental. (Chapter 5)
- **The MCP ecosystem's own evolving auth requirements** (Protected Resource Metadata becoming mandatory, Dynamic Client Registration downgraded from SHOULD to MAY across spec revisions) — a good example of a live standard tightening its security posture over time, worth mentioning if asked about how specs evolve. (Chapter 10)

## Summary

- Interview questions on OAuth cluster around: the 4 roles, why PKCE/state/audience-binding each exist, access-vs-refresh-token design, and OAuth-vs-OIDC.
- System design questions want you to name the actual RFC-backed mechanisms (Resource Indicators, Token Exchange, rotation) rather than vague "add security" answers.
- Troubleshooting questions reward reasoning through multiple plausible root causes for one symptom, not jumping to a single guess.

## Knowledge Check

1. Prepare a 60-second answer to "explain OAuth to me like I'm a senior engineer who's never touched it."
2. Prepare a 60-second answer to "explain OAuth to me like I'm a non-technical product manager."
3. Pick one scenario question above and write out your answer without looking at the model answer first.

## Hands-On Exercise

Pair up (or role-play both sides yourself) and run a mock interview using the three scenario questions above, timing yourself to answer each in under 3 minutes, referencing specific mechanisms (PKCE, audience validation, rotation) by name rather than describing them vaguely.

## Further Reading

- Revisit [Chapter 11 (Security & Risks)](./11-security-and-risks.md) and [Chapter 12 (Best Practices)](./12-best-practices.md) — nearly every interview question in this chapter traces back to one of those two chapters
- [OWASP OAuth 2.0 Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html) — good for a final pre-interview skim

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./15-capstone-projects.md">← Previous: Capstone Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <span></span>
</div>
