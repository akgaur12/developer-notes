# Chapter 12: Best Practices

## Learning Objectives

- Apply a concrete checklist for both OAuth clients and Authorization Servers
- Know what the Chapter 9 demo deliberately skipped, and why each omission matters in production
- Understand token storage, HTTPS, and logging practices that separate a toy implementation from a production one

## Prerequisites for This Chapter

Chapters [9](./09-practical-usage-building-the-flow.md) and [11](./11-security-and-risks.md).

## What Chapter 9's Demo Deliberately Skipped

Being explicit about this matters — treat this table as your production gap list if you ever adapt that demo into something real:

| Skipped in the demo | Why it's needed in production |
|---|---|
| **HTTPS everywhere** | Plain HTTP exposes codes and tokens to network eavesdroppers. Every real Authorization Server endpoint must be TLS-only. |
| **Real user authentication** | The demo auto-trusts anyone who clicks "Allow" — a real AS must verify identity (password, SSO, MFA) before showing consent. |
| **Persistent storage** | In-memory dicts vanish on restart and don't scale beyond one process. Real systems use a database or distributed cache for codes/tokens. |
| **Rate limiting** | Nothing stops brute-forcing the `/token` endpoint in the demo. Real Authorization Servers rate-limit and monitor for abuse. |
| **Client authentication for confidential clients** | The demo's client is public (no secret). A confidential client's requests to `/token` should also verify a `client_secret` or equivalent. |
| **Audience/`resource` binding** | The demo issues one generic token type. A production system fronting multiple APIs must bind tokens to a specific audience (Chapter 6, Chapter 10). |
| **Signed, verifiable tokens (JWTs) with a real key** | The demo uses random opaque strings looked up in a dict. Production systems typically issue signed JWTs validated cryptographically (Chapter 5). |

## Client-Side Best Practices

1. **Always use PKCE**, even if the Authorization Server doesn't strictly require it for your client type. There's no downside, and OAuth 2.1 makes it mandatory anyway.
2. **Generate a fresh, high-entropy `state` and `code_verifier` per flow.** Never reuse them across requests.
3. **Store tokens in the most secure storage your platform offers**: OS keychain for desktop apps, httpOnly secure cookies or server-side sessions for web apps — never in `localStorage` or plain files.
4. **Validate the redirect actually came from where you expect** — check `state`, and if using OIDC, validate the ID token's `iss` and `aud` claims (Chapter 8).
5. **Handle token expiry proactively**: attempt a refresh slightly before expiry when possible, rather than waiting for a `401` and retrying — this avoids user-visible latency spikes on the request that happens to catch the token expired.
6. **Request the minimum scope needed.** Don't ask for `write` access "in case you need it later" — request it when you actually add that feature.
7. **Treat refresh tokens as more sensitive than access tokens** in your storage and logging decisions.

## Authorization-Server / Resource-Server Best Practices

1. **Exact-match redirect URI validation** — no substring or prefix matching (Chapter 11).
2. **Short access token lifetimes** (minutes, not days) paired with refresh token rotation (Chapter 5).
3. **Sign tokens with a real asymmetric key (RS256/ES256)** and expose a JWKS (JSON Web Key Set) endpoint so Resource Servers can verify signatures without a shared secret.
4. **Validate audience/`aud` on every request** at the Resource Server, not just signature and expiry (Chapter 6, Chapter 10, and the confused-deputy discussion in Chapter 11).
5. **Support token revocation** ([RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009)) so a "Disconnect" button in your product actually invalidates a token immediately, not just lets it expire naturally.
6. **Rate-limit and monitor the `/token` endpoint** — a spike in failed grant exchanges is a strong signal of a credential-stuffing or code-guessing attempt.
7. **Never log full tokens.** Log a hash or the first/last few characters if you need to correlate requests, never the raw token value — treat it the same as you'd treat a password in logs.
8. **Make authorization codes single-use and short-lived** (30–60 seconds is typical) — the Chapter 9 demo already does this via `AUTH_CODES.pop()`, which is correct.

## Observability

Production OAuth systems should emit metrics/logs for:
- Token issuance rate (by client, by grant type)
- Token validation failures, broken out by *reason* (expired vs. bad signature vs. audience mismatch) — an audience-mismatch spike specifically indicates the confused-deputy scenario from Chapter 11
- Refresh token reuse detection events (a rotated-out token being presented again)
- Consent screen abandonment rate (users starting but not completing the flow — often signals UX friction or a broken redirect URI)

## A Minimal Production Checklist

Before shipping any OAuth integration (client or server), confirm:

- [ ] All endpoints are HTTPS-only
- [ ] PKCE is implemented and enforced
- [ ] `state` is generated fresh and validated on every callback
- [ ] Tokens are stored in platform-appropriate secure storage, never client-side JS-accessible storage
- [ ] Access tokens are short-lived; refresh tokens rotate on use
- [ ] Redirect URIs are exact-match validated
- [ ] Token audience is validated at the Resource Server
- [ ] Revocation is supported and wired to any "Disconnect"/"Remove access" UI
- [ ] No raw tokens appear in logs, error messages, or crash reports

## Summary

- The Chapter 9 demo is a correct teaching tool but skips HTTPS, real auth, persistence, and audience binding — know the gap before reusing any of that code.
- Client-side discipline (PKCE, minimal scope, secure storage) matters as much as server-side discipline (exact redirect matching, short-lived tokens, audience validation).
- Observability on token validation *failures*, specifically broken out by cause, is what turns a security incident into a detected one instead of a discovered-after-the-fact one.

## Knowledge Check

1. Name three things the Chapter 9 demo would need before it could be considered production-ready, and explain the risk each one mitigates.
2. Why should refresh tokens be treated with more caution in logging/storage decisions than access tokens?
3. Why is "audience-mismatch failure rate" specifically a more useful metric to monitor than a general "token validation failures" count?

## Hands-On Exercise

Take the Chapter 9 demo and add exactly one production practice from this chapter's checklist — e.g., swap the opaque random access tokens for signed JWTs (using `PyJWT` with an HMAC secret for simplicity), and update `/api/data`'s validation logic to verify the signature and `exp` claim from the JWT instead of a dictionary lookup.

## Further Reading

- [RFC 7009 — OAuth 2.0 Token Revocation](https://datatracker.ietf.org/doc/html/rfc7009)
- [RFC 7517 — JSON Web Key (JWK)](https://datatracker.ietf.org/doc/html/rfc7517)
- [OWASP OAuth 2.0 Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html)

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./11-security-and-risks.md">← Previous: Security & Risks</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./13-common-mistakes-and-pitfalls.md">Next: Common Mistakes & Pitfalls →</a>
</div>
