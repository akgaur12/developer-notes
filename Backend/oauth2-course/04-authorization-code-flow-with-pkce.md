# Chapter 4: The Authorization Code Flow with PKCE

## Learning Objectives

- Walk through every step of the flow you saw in Claude/Kimi, in order
- Explain what PKCE is, why it exists, and how the code verifier/challenge pair works
- Trace a concrete example end-to-end with real (illustrative) values

## Prerequisites for This Chapter

Chapters [2](./02-core-concepts-and-terminology.md) and [3](./03-fundamental-components.md) — you need the 4 actors, the two endpoints, and the concept of public vs. confidential clients before this will click.

## Why PKCE Exists

Recall from Chapter 3: Claude/Kimi's MCP client is a **public client** — it cannot hold a secret. Without a secret, here's the vulnerability in the plain Authorization Code flow:

1. Claude sends the user to the Authorization Server to log in.
2. The AS redirects back to Claude's redirect URI with an authorization code.
3. **If a malicious app on the same device can intercept that redirect** (a real risk on mobile OSes with shared URI schemes, or via a network intermediary), it steals the code.
4. That malicious app exchanges the stolen code for an access token at the Token Endpoint — and since there's no client secret to also check, nothing stops it.

**PKCE (Proof Key for Code Exchange, [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636))** closes this gap by having the *client itself* generate a one-time secret for each individual flow, so even a stolen code is useless without it.

Analogy: instead of a static password (which, once stolen, works for anyone), the client writes a random secret word on a piece of paper, folds it up and gives a *sealed hash* of it to the Authorization Server upfront ("code challenge"), then later proves it holds the original unfolded paper ("code verifier") when redeeming the code. An attacker who steals just the folded code from the redirect never sees the paper.

## The Two PKCE Values

| Term | What it is | Analogy |
|---|---|---|
| **Code Verifier** | A random, high-entropy string generated fresh by the client for this one flow (43–128 characters) | The paper with the secret word |
| **Code Challenge** | `BASE64URL(SHA256(code_verifier))` — sent to the AS *before* login | The sealed hash of that paper |

The client keeps the verifier locally (never transmitted until the final step) and sends only the challenge (a hash) in the initial browser redirect.

## Step-by-Step Flow

```mermaid
sequenceDiagram
    participant U as You (Resource Owner)
    participant C as Claude (Client)
    participant AS as Authorization Server
    participant RS as Resource Server (API)

    C->>C: 1. Generate code_verifier (random secret)
    C->>C: 2. Compute code_challenge = SHA256(code_verifier)
    C->>U: 3. Open browser to /authorize?...&code_challenge=...&state=...
    U->>AS: 4. Browser loads Authorization Server's login page
    U->>AS: 5. Log in (password or SSO) + click "Allow"
    AS->>U: 6. Redirect browser to redirect_uri?code=XYZ&state=...
    U->>C: 7. Browser hits Claude's local callback listener
    C->>C: 8. Verify state matches what was sent in step 3
    C->>AS: 9. POST /token with code=XYZ + code_verifier (the original secret)
    AS->>AS: 10. Recompute SHA256(code_verifier), compare to stored code_challenge
    AS->>C: 11. If match: return access_token + refresh_token (JSON)
    C->>RS: 12. Call API with "Authorization: Bearer <access_token>"
    RS->>C: 13. Return protected data
```

## Concrete Walkthrough With Illustrative Values

**Step 1–2 (Client, before opening browser):**
```
code_verifier  = "dGhpc19pc19hX3JhbmRvbV9zZWNyZXRfZm9yX3RoaXNfZmxvdw"
code_challenge = BASE64URL(SHA256(code_verifier))
              = "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"
```

**Step 3 (Client opens browser to):**
```
https://notion.so/oauth/authorize?
  response_type=code&client_id=abc123claude
  &redirect_uri=http://localhost:52847/callback
  &scope=read:pages%20offline_access
  &state=x9f2kzp1
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
```

**Steps 4–6 (You interact with Notion's page, not Claude's):** you log in, see "Allow Claude to read your pages?", click Allow.

**Step 6 (Notion redirects browser to):**
```
http://localhost:52847/callback?code=AUTH_CODE_ABCDEF&state=x9f2kzp1
```

**Step 7–8:** Claude's tiny local listener on port 52847 catches this, checks `state` matches, and closes the browser tab — this is the moment you saw the window disappear.

**Step 9 (Claude calls the Token Endpoint directly, invisibly, no browser involved):**
```bash
curl -X POST https://notion.so/oauth/token \
  -d grant_type=authorization_code \
  -d code=AUTH_CODE_ABCDEF \
  -d redirect_uri=http://localhost:52847/callback \
  -d client_id=abc123claude \
  -d code_verifier=dGhpc19pc19hX3JhbmRvbV9zZWNyZXRfZm9yX3RoaXNfZmxvdw
```

**Step 10–11 (Notion's response):**
```json
{
  "access_token": "ntn_at_9f8e7d6c5b4a...",
  "refresh_token": "ntn_rt_1a2b3c4d5e...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "read:pages offline_access"
}
```

**Step 12–13:** From now on, Claude calls Notion's API with `Authorization: Bearer ntn_at_9f8e7d6c5b4a...` and gets your pages back.

## What If the Code Verifier Doesn't Match?

The Authorization Server rejects the token exchange outright — even with a valid, unexpired authorization code. This is the entire security value of PKCE: possession of the code alone is insufficient.

## Summary

- PKCE adds a client-generated, per-flow secret (verifier/challenge pair) to protect public clients that can't hold a static client secret.
- The challenge (a hash) goes out with the initial browser redirect; the verifier (the original secret) is only revealed at the final, direct-to-server token exchange.
- This is the exact flow behind every "Authenticate" button you've clicked on an MCP server.

## Knowledge Check

1. Why would PKCE be unnecessary for a confidential, server-side-only client?
2. What specifically would an attacker be missing if they intercepted only the authorization code from the redirect?
3. At which step does `state` get checked, and what would happen if it didn't match?
4. Why is the code_challenge sent as a hash rather than the raw verifier?

## Hands-On Exercise

Using Python's `hashlib` and `base64`, generate your own code_verifier/code_challenge pair:
```python
import hashlib, base64, secrets

verifier = base64.urlsafe_b64encode(secrets.token_bytes(32)).rstrip(b"=").decode()
challenge = base64.urlsafe_b64encode(
    hashlib.sha256(verifier.encode()).digest()
).rstrip(b"=").decode()

print("code_verifier: ", verifier)
print("code_challenge:", challenge)
```
Confirm that re-hashing the verifier always reproduces the same challenge, and that changing even one character of the verifier produces a completely different challenge.

## Further Reading

- [RFC 7636 — Proof Key for Code Exchange](https://datatracker.ietf.org/doc/html/rfc7636)
- [oauth.com — PKCE](https://www.oauth.com/oauth2-servers/pkce/)

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./03-fundamental-components.md">← Previous: Fundamental Components</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./05-tokens-access-refresh-id.md">Next: Tokens: Access, Refresh, and ID Tokens →</a>
</div>
