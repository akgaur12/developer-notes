# Chapter 5: Tokens — Access, Refresh, and ID Tokens

## Learning Objectives

- Explain the purpose, lifetime, and handling rules for access tokens vs. refresh tokens
- Decode a JWT and identify its three parts
- Trace exactly how "silent re-authentication" works without you ever seeing a login prompt
- Understand refresh token rotation and why it matters

## Prerequisites for This Chapter

[Chapter 4](./04-authorization-code-flow-with-pkce.md) — you should have already seen an access_token/refresh_token pair returned from a token exchange.

## The Two Tokens, and Why There Are Two

A single, long-lived token would be simpler to implement — so why does OAuth deliberately issue two?

| Token | Lifetime | Used for | Sent to |
|---|---|---|---|
| **Access Token** | Short — typically 5 minutes to 1 hour | Calling the actual API (Resource Server) on every request | Resource Server |
| **Refresh Token** | Long — days, weeks, or until revoked | Getting a *new* access token when the old one expires | Authorization Server **only** |

This split exists to limit the blast radius of a leak. Access tokens are attached to every single API call, so they're the ones most likely to leak — into logs, browser dev tools, a misconfigured proxy, a crash report. By making them short-lived, a leaked access token is only dangerous for a small window. The refresh token, by contrast, is used rarely (only against the trusted Authorization Server, never sent to arbitrary APIs), so it's held to a higher standard of protection and can safely live much longer.

Analogy: the access token is like a **day pass** you show at every door in the building — if you drop it, it's void by tomorrow anyway. The refresh token is like the **membership card** you keep in your wallet and only show at the front desk to get a fresh day pass — you protect it more carefully because it's valid for months.

## What's Actually Inside an Access Token: JWT Structure

Many (not all — some are opaque random strings the AS looks up in a database) access tokens are **JWTs (JSON Web Tokens, RFC 7519)** — self-contained, digitally signed strings. A JWT has three dot-separated parts, each Base64URL-encoded:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzQ0MiIsInNjb3BlIjoicmVhZDpwYWdlcyIsImV4cCI6MTc1NTAwMDAwMH0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
└──────────── header ────────────┘└──────────────────── payload ────────────────────┘└──────────── signature ────────────┘
```

| Part | Contents (decoded) | Purpose |
|---|---|---|
| **Header** | `{"alg":"RS256","typ":"JWT"}` | Which signing algorithm was used |
| **Payload** (claims) | `{"sub":"user_442","scope":"read:pages","exp":1755000000}` | The actual data: who, what scope, when it expires |
| **Signature** | Cryptographic signature over header+payload | Proves the Authorization Server issued it, unmodified |

Key claims you'll see constantly:
- `sub` (subject) — the user/account this token represents
- `exp` (expiration) — Unix timestamp after which the token is invalid
- `iat` (issued at) — when it was created
- `aud` (audience) — which Resource Server this token is meant for
- `scope` — the granted permissions

**Critical point:** anyone can *decode* (read) a JWT — it's just Base64, not encrypted. What they can't do is *forge* one, because they don't have the Authorization Server's private signing key. The Resource Server verifies the signature on every request; it never has to call back to the AS to check validity, which is why JWTs are fast. Try pasting a real JWT into [jwt.io](https://jwt.io) — you'll see the header and payload instantly, in plain JSON.

## How "Silent Re-Authentication" Actually Works

This is the part you specifically asked about. Here's exactly what happens when your access token expires while you're not looking:

```mermaid
sequenceDiagram
    participant C as Claude (Client)
    participant AS as Authorization Server
    participant RS as Resource Server

    Note over C: access_token has expired (exp claim in the past)
    C->>RS: API call with expired access_token
    RS->>C: 401 Unauthorized
    C->>AS: POST /token grant_type=refresh_token&refresh_token=ntn_rt_1a2b3c...
    AS->>AS: Validate refresh token, check it hasn't been revoked
    AS->>C: New access_token (+ possibly new refresh_token)
    C->>RS: Retry API call with new access_token
    RS->>C: 200 OK, data returned
```

No browser opens. No consent screen. No password prompt. The entire exchange is a background POST request, invisible to you — which is exactly the behavior you noticed and asked about.

```bash
curl -X POST https://notion.so/oauth/token \
  -d grant_type=refresh_token \
  -d refresh_token=ntn_rt_1a2b3c4d5e... \
  -d client_id=abc123claude
```

Response:
```json
{
  "access_token": "ntn_at_new_token_here...",
  "refresh_token": "ntn_rt_new_rotated_value...",
  "expires_in": 3600
}
```

## Refresh Token Rotation

Notice the response above includes a **new** refresh token, not the same one. This is **refresh token rotation**: every time a refresh token is used, the Authorization Server issues a new one and invalidates the old. This limits how long a stolen refresh token stays useful, and — critically — lets the AS detect theft: if the *old*, already-rotated-out refresh token is ever presented again, that's a strong signal it was stolen (a legitimate client would already be using the newer one), and the AS can revoke the entire token family in response.

## ID Tokens — A Preview

You'll also sometimes see a third token, the **ID Token**. It is *not* part of OAuth 2.0 itself — it's introduced by **OpenID Connect** (OIDC), covered fully in Chapter 8. Briefly: it's a JWT that represents proof of *authentication* ("this user logged in, here's who they are"), as opposed to the access token, which represents *authorization* ("this app can call this API"). Don't use an access token as proof of login identity — that confusion is common enough that Chapter 8 exists specifically to untangle it.

## Where Tokens Get Stored

- **Server-side clients**: in a database or encrypted session store, never sent to the browser.
- **Desktop apps like Claude/Kimi**: typically in the OS-level credential store (macOS Keychain, Windows Credential Manager, or an encrypted local file) — not plain text, and not in browser localStorage (which has no such protection).
- **Never**: in URL query strings that might get logged, or in client-side JavaScript variables accessible to any injected script.

## Summary

- Access tokens are short-lived and used against the API; refresh tokens are long-lived and used only against the Authorization Server.
- JWTs are signed, not encrypted — readable by anyone, forgeable by no one without the private key.
- Silent renewal is a plain background HTTP call using the refresh token — no user interaction.
- Refresh token rotation issues a new refresh token on every use, enabling theft detection.

## Knowledge Check

1. Why is it safe for an access token to be readable (unencrypted) by anyone who intercepts it, as long as it's short-lived?
2. What specific HTTP status code typically triggers a client to attempt a token refresh?
3. What does refresh token rotation let an Authorization Server detect that a static refresh token wouldn't?
4. Why shouldn't an access token be trusted as proof of "this is genuinely the logged-in user talking to me right now"?

## Hands-On Exercise

Take any JWT (you can generate one for free using [jwt.io](https://jwt.io)'s debugger, or capture one from a real login flow using browser dev tools' Network tab). Decode the payload by hand: split the JWT on `.`, take the middle section, and Base64URL-decode it. Verify you can see `exp` and calculate whether it's expired right now.

## Further Reading

- [RFC 7519 — JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 6749, Section 6 — Refreshing an Access Token](https://datatracker.ietf.org/doc/html/rfc6749#section-6)
- [OAuth 2.0 Security Best Current Practice — Refresh Token Rotation](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./04-authorization-code-flow-with-pkce.md">← Previous: The Authorization Code Flow with PKCE</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./06-architecture-and-internals.md">Next: Architecture & Internals →</a>
</div>
