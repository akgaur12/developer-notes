# Chapter 11: Security & Risks

## Learning Objectives

- Identify the major OAuth attack classes: CSRF, authorization code interception, open redirect, token leakage, confused deputy
- Explain the "confused deputy" problem specifically, since it's the one most unique to OAuth
- Recognize why token storage location matters as much as the protocol itself

## Prerequisites for This Chapter

Chapters [3](./03-fundamental-components.md)–[6](./06-architecture-and-internals.md).

## 1. CSRF via a Missing or Unchecked `state`

**The attack:** an attacker starts their own OAuth flow with the Authorization Server, gets a valid authorization code intended for *their own* attacker-controlled account, then tricks your browser (via a crafted link) into completing the redirect step to your client's callback URL with the attacker's code. If your client doesn't check `state`, it will happily exchange that code and end up associated with the attacker's account.

**The defense:** always generate a fresh, random `state` per flow and reject any callback where it doesn't match (Chapter 3). This is why `client.py` in Chapter 9 asserts `received["state"] == state` before proceeding — that single line is the entire CSRF defense.

## 2. Authorization Code Interception

**The attack:** on a shared device or through a malicious app registered with an overlapping URI scheme, another process intercepts the redirect containing the authorization code before the legitimate client does, then races to exchange it for a token itself.

**The defense:** PKCE (Chapter 4). Even with a stolen code, the attacker lacks the `code_verifier`, so the token exchange fails. This is precisely why PKCE became mandatory for all clients (not just public ones) in the OAuth 2.1 consolidation.

## 3. Open Redirect / Redirect URI Manipulation

**The attack:** if an Authorization Server validates redirect URIs loosely (e.g., accepting *any* URI on a registered domain, or doing substring matching instead of exact matching), an attacker registers a lookalike or crafts a URI that passes validation but actually points at attacker-controlled infrastructure, capturing codes meant for the real client.

**The defense:** Authorization Servers must do **exact-match** validation of redirect URIs against the pre-registered value — no wildcards, no "starts with," no query-string trickery. If you ever implement an Authorization Server, this is a "get it exactly right or the rest of your security model doesn't matter" detail.

## 4. Confused Deputy

This is the attack class most unique to delegated-authorization systems like OAuth, and worth understanding precisely because it's easy to state vaguely and miss the actual mechanism.

**The setup:** a service (the "deputy") holds legitimate authority to act on a resource, granted for one specific purpose. **The attack:** a malicious party tricks the deputy into misusing that authority for a purpose the resource owner never intended — the deputy is "confused" about which action it's actually authorizing.

**Concrete OAuth example:** imagine an MCP server that, whenever it receives *any* validly-signed bearer token from your company's shared identity provider, blindly forwards that exact token to a downstream internal API — without checking which specific resource/audience the token was actually minted for (this is exactly the **token passthrough** anti-pattern flagged in this repo's [`mcp-course` Chapter 13](../../AI-ML/mcp-course/13-authentication-and-authorization.md#10-token-passthrough--the-one-explicit-must-not)). A token you obtained to let one, low-privilege tool read your calendar could then be replayed by a compromised or malicious *other* tool against a completely different, more sensitive API — because nothing checked "was this token actually meant for that API?"

**The defense:** never forward tokens blindly between services; validate the audience (`aud`) claim strictly at every hop; each service should mint or request its own appropriately-scoped credential rather than reusing whatever token happened to arrive.

## 5. Token Leakage

Access and refresh tokens are just strings — anywhere a string can end up unintentionally, a token can leak:

- **Browser history / server logs**: tokens in URL query strings get logged by proxies, CDNs, and browser history. This is one more reason the Authorization Code flow moves the code (not the final token) through the URL, and the actual token exchange happens via a POST body instead.
- **Referrer headers**: a page that includes a token in its URL can leak it to any third-party resource that page loads, via the `Referer` header.
- **Insecure storage**: tokens stored in browser `localStorage` are readable by any JavaScript running on the page, including injected malicious scripts (XSS) — this is a major reason OAuth 2.1 pushes toward the Authorization Code + PKCE flow with tokens held server-side or in OS-level secure storage rather than client-side JavaScript-accessible storage.
- **Overly verbose error messages or debug logs**: developers accidentally logging full request/response bodies during debugging, tokens included.

## 6. Refresh Token Theft

Because refresh tokens are long-lived, stealing one is far more valuable to an attacker than stealing a short-lived access token. Mitigations covered so far: **refresh token rotation** (Chapter 5) so a stolen-then-reused old token is detectable, and storing refresh tokens only in protected locations (never browser-accessible storage, never sent to the Resource Server).

## 7. Scope Creep / Over-Permissioning

Not an "attack" from an outside party, but a self-inflicted risk: requesting broader scopes than an integration actually needs "just in case." If that client is later compromised, the attacker inherits everything the over-broad scope allowed. The mitigation is discipline, not a protocol feature: request the narrowest scope that accomplishes the task (Chapter 2), matching your IoT-systems instinct of least-privilege device credentials.

## A Note for Your IoT Background

Several of these map directly onto patterns you likely already apply to device credentials: short-lived tokens are the OAuth analog of rotating device certificates; scope minimization is the same instinct as giving a sensor read-only telemetry access rather than admin control; audience binding is conceptually similar to pinning a device credential to one specific gateway rather than letting it authenticate to any gateway on the network.

## Summary

- `state` defeats CSRF; PKCE defeats authorization-code interception; exact-match redirect URI validation defeats open-redirect hijacking.
- Confused deputy is OAuth's signature attack class: a service with legitimate authority gets tricked into misusing it, typically via unchecked token forwarding.
- Token leakage happens through mundane channels (logs, browser history, Referer headers, insecure client-side storage) more often than through cryptographic breaks.
- Refresh token rotation and narrow scoping limit the damage of any single leak.

## Knowledge Check

1. Walk through, step by step, how an attacker without `state` checking could hijack a login flow.
2. Why does PKCE specifically defeat authorization code interception but not, say, a stolen refresh token?
3. In your own words, define "confused deputy" using an example different from the one in this chapter.
4. Why is storing an access token in browser `localStorage` riskier than storing it server-side?

## Hands-On Exercise

Using the `client.py`/`server.py` demo from Chapter 9, deliberately remove the `state` check (comment out the `assert`) and simulate a CSRF-style attack: manually craft a request to `/approve` with a `state` value your client never generated, and confirm the client would have accepted it without the check. Then restore the check and confirm the modified request no longer succeeds.

## Further Reading

- [RFC 6749, Section 10 — Security Considerations](https://datatracker.ietf.org/doc/html/rfc6749#section-10)
- [OAuth 2.0 Security Best Current Practice (draft)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- This repo's [`mcp-course`, Chapter 14 — MCP Security](../../AI-ML/mcp-course/14-mcp-security.md) for the confused-deputy and token-passthrough problem in an MCP-specific setting

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./10-mcp-and-oauth-in-practice.md">← Previous: MCP and OAuth in Practice</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./12-best-practices.md">Next: Best Practices →</a>
</div>
