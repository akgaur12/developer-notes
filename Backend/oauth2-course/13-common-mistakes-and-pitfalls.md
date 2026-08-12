# Chapter 13: Common Mistakes & Pitfalls

## Learning Objectives

- Recognize the mistakes that recur most often in real OAuth implementations and code reviews
- Distinguish a genuine security bug from a merely awkward-but-safe implementation
- Know the specific fix for each pitfall, not just that it's "wrong"

## Prerequisites for This Chapter

Chapters [4](./04-authorization-code-flow-with-pkce.md), [5](./05-tokens-access-refresh-id.md), [11](./11-security-and-risks.md).

## Pitfall 1: Treating the Access Token as Proof of Login

**The mistake:** using "the API call with this access token succeeded" as your app's login check.

**Why it's wrong:** an access token proves API authorization, not "this specific human is present right now." Chapter 8 covers this exact confusion — use an OIDC ID token for login, not an OAuth access token.

**The fix:** if you need "who is this user," request the `openid` scope and verify the ID token's signature and claims (`iss`, `aud`, `exp`) — don't infer identity from API success.

## Pitfall 2: Skipping `state` Validation "Because It's Extra Code"

**The mistake:** generating a `state` value (because a tutorial said to) but never actually checking it matches on the callback.

**Why it's wrong:** this makes the CSRF protection entirely theatrical — present in the code, absent in effect. Chapter 11's attack #1 walks through exactly how this gets exploited.

**The fix:** the check must be an actual comparison that aborts the flow on mismatch, as in Chapter 9's `assert received["state"] == state`.

## Pitfall 3: Storing Tokens in Browser `localStorage`

**The mistake:** for convenience, storing access/refresh tokens in `localStorage` or `sessionStorage` in a single-page app.

**Why it's wrong:** any JavaScript that runs on the page — including injected via an XSS vulnerability elsewhere on the same site — can read `localStorage` freely. A refresh token there is a standing, long-lived compromise waiting for any unrelated XSS bug.

**The fix:** prefer a backend-for-frontend pattern where tokens live server-side, and the browser only holds an httpOnly session cookie the JavaScript can't read directly.

## Pitfall 4: Confusing Client ID with Client Secret Security

**The mistake:** treating the `client_id` as sensitive and hiding it, while being lax about `client_secret` exposure, or the reverse — assuming a public client's hardcoded "secret" (visible in decompiled app code) actually provides security.

**Why it's wrong:** `client_id` is meant to be public — it identifies the app, not a credential. A `client_secret` embedded in a distributed binary (mobile app, desktop app, browser JS) is not actually secret; anyone can extract it. Relying on it as a security boundary for a public client is a false sense of protection.

**The fix:** public clients should have no secret at all and rely on PKCE (Chapter 4) instead — this is precisely why PKCE exists.

## Pitfall 5: Long-Lived or Non-Expiring Access Tokens

**The mistake:** issuing access tokens with multi-day or non-expiring lifetimes "to reduce refresh overhead."

**Why it's wrong:** it defeats the entire "blast radius" design in Chapter 5 — a leaked token now stays dangerous for as long as it's valid, potentially indefinitely.

**The fix:** keep access tokens short (minutes to a couple hours) and let the refresh mechanism absorb the renewal cost — that's what it's for.

## Pitfall 6: Not Rotating Refresh Tokens

**The mistake:** issuing the same refresh token forever, never replacing it on use.

**Why it's wrong:** it removes the theft-detection property described in Chapter 5 — there's no way to distinguish "the legitimate client refreshed" from "an attacker who stole the refresh token weeks ago refreshed."

**The fix:** implement rotation: every refresh call invalidates the presented refresh token and issues a new one; treat reuse of an already-rotated-out token as a signal to revoke the whole token family.

## Pitfall 7: Validating Signature and Expiry but Not Audience

**The mistake:** a Resource Server checks that a token is validly signed and not expired, and stops there.

**Why it's wrong:** this is exactly the gap that enables the confused-deputy/cross-server replay scenario from Chapter 11 — a token minted for a *different* API can still pass signature and expiry checks.

**The fix:** always validate the `aud` (audience) claim against your own service's identity before accepting a token, as covered in Chapter 6 and Chapter 10.

## Pitfall 8: Manually Parsing/Building OAuth URLs and JWTs From Scratch in Production

**The mistake:** hand-rolling URL construction, signature verification, or JWT parsing in a production system (fine for a learning exercise like Chapter 9 — wrong for anything real).

**Why it's wrong:** subtle bugs in signature verification (e.g., not checking `alg` correctly, one of the most infamous JWT vulnerability classes — accepting `alg: none` or letting an attacker switch algorithms) are easy to introduce and hard to catch in review.

**The fix:** use a well-maintained OAuth/OIDC client library and a well-maintained JWT library for your language, rather than reimplementing the cryptographic and parsing logic.

## Pitfall 9: Overly Broad Scope Requests

**The mistake:** requesting `scope=read write admin` for an integration that only ever reads data, because "we might need it later."

**Why it's wrong:** it maximizes the damage of any future compromise of that specific integration, and often triggers unnecessarily scary consent screens that erode user trust.

**The fix:** request only what's needed now; add scopes incrementally as features that need them ship.

## Pitfall 10: Assuming HTTPS Is Optional for "Internal" or "Development" OAuth Flows

**The mistake:** running the Authorization Server or token endpoint over plain HTTP because "it's just internal" or "it's just for local dev."

**Why it's wrong:** codes and tokens travel in the clear, and internal networks are not immune to sniffing (a compromised machine on the same network, a misconfigured proxy). Habits formed in "temporary" internal setups have a well-documented tendency to leak into production.

**The fix:** use HTTPS even for internal and development environments (self-signed certs are fine for local dev) so the habit and the tooling are already correct before anything goes to production.

## Summary

- Most real OAuth bugs are omissions of a check that exists specifically to stop a named attack (state, PKCE, audience) — not exotic cryptographic failures.
- Public clients should never rely on secrets; confidential clients should never expose theirs client-side.
- Token lifetime and rotation policy is a security control, not just an efficiency knob.

## Knowledge Check

1. Which two pitfalls in this chapter both stem from "a check exists in the code but isn't actually enforced correctly"?
2. Why is a `client_secret` embedded in a mobile app not actually providing the security its name implies?
3. What specific vulnerability class does "hand-rolling JWT verification" risk introducing, beyond ordinary bugs?

## Hands-On Exercise

Review the Chapter 9 demo code once more with this chapter's list in hand. Identify which pitfalls it deliberately avoids (it should be several) and which ones it explicitly leaves as an exercise (persistent storage, HTTPS, real user auth). Write one sentence per pitfall explaining which category it falls into for that codebase.

## Further Reading

- [Auth0 Blog — Critical Vulnerabilities in JSON Web Token Libraries](https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/)
- [OWASP OAuth 2.0 Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html)

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./12-best-practices.md">← Previous: Best Practices</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./14-advanced-concepts.md">Next: Advanced Concepts →</a>
</div>
