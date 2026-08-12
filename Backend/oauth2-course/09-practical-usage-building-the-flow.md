# Chapter 9: Practical Usage — Building the Flow

## Learning Objectives

- Run a complete, working Authorization Code + PKCE flow on your own machine
- See the exact HTTP requests/responses at every step, matching the theory from Chapters 3–5
- Watch an access token expire and get silently refreshed, live
- Confirm PKCE actually rejects a mismatched verifier

## Prerequisites for This Chapter

Chapters [4](./04-authorization-code-flow-with-pkce.md) and [5](./05-tokens-access-refresh-id.md). Python 3.9+ only — **no external dependencies** (uses only the standard library, so there's nothing to `pip install`).

## What You're About to Build

Two small scripts:

- **`server.py`** — plays both the Authorization Server and Resource Server roles (a single company's system often does both, as in Chapter 6).
- **`client.py`** — plays the Client role, exactly like Claude/Kimi's MCP client: opens a browser for login/consent, catches the redirect locally, exchanges the code for tokens, calls the protected API, then demonstrates the silent refresh.

This is deliberately minimal (in-memory storage, no HTTPS, no real user database) so you can read every line and see exactly where each concept from earlier chapters lives in real code. It is **not** production-ready — see [Chapter 12](./12-best-practices.md) for what's missing.

## `server.py` — Authorization Server + Resource Server

```python
"""Minimal OAuth 2.0 Authorization Server + Resource Server, stdlib only.
Demonstrates: /authorize (consent), /token (code exchange + refresh), PKCE verification.
Not production code -- no HTTPS, no real user DB, in-memory storage. For learning only.
"""
import base64
import hashlib
import http.server
import json
import secrets
import time
import urllib.parse

CLIENT_ID = "demo-client"
ALLOWED_REDIRECT_PREFIX = "http://localhost:"
ACCESS_TOKEN_TTL = 20   # seconds -- short on purpose so you can *watch* it expire
REFRESH_TOKEN_TTL = 3600

# in-memory "databases"
AUTH_CODES = {}      # code -> {client_id, redirect_uri, code_challenge, scope, expires_at}
ACCESS_TOKENS = {}   # token -> {expires_at, scope}
REFRESH_TOKENS = {}  # token -> {expires_at, scope}


def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode()


class Handler(http.server.BaseHTTPRequestHandler):
    def log_message(self, fmt, *args):
        pass  # keep demo output quiet

    def _send_json(self, obj, status=200):
        body = json.dumps(obj).encode()
        self.send_response(status)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def do_GET(self):
        parsed = urllib.parse.urlparse(self.path)
        qs = urllib.parse.parse_qs(parsed.query)

        if parsed.path == "/authorize":
            client_id = qs.get("client_id", [""])[0]
            redirect_uri = qs.get("redirect_uri", [""])[0]
            state = qs.get("state", [""])[0]
            scope = qs.get("scope", [""])[0]
            code_challenge = qs.get("code_challenge", [""])[0]

            if client_id != CLIENT_ID or not redirect_uri.startswith(ALLOWED_REDIRECT_PREFIX):
                self._send_json({"error": "invalid_client_or_redirect_uri"}, 400)
                return

            # Render a real (if ugly) consent screen -- this is the page you actually
            # see and click "Allow" on, hosted by the Authorization Server, not the client.
            approve_url = "/approve?" + urllib.parse.urlencode({
                "client_id": client_id, "redirect_uri": redirect_uri,
                "state": state, "scope": scope, "code_challenge": code_challenge,
            })
            html = f"""<html><body style="font-family:sans-serif;max-width:420px;margin:80px auto">
            <h2>Demo Auth Server</h2>
            <p><b>{client_id}</b> is requesting access to:</p>
            <p><code>{scope}</code></p>
            <form method="GET" action="{approve_url}">
              <button style="padding:8px 20px">Allow</button>
            </form>
            </body></html>"""
            body = html.encode()
            self.send_response(200)
            self.send_header("Content-Type", "text/html")
            self.send_header("Content-Length", str(len(body)))
            self.end_headers()
            self.wfile.write(body)
            return

        if parsed.path == "/approve":
            redirect_uri = qs["redirect_uri"][0]
            state = qs.get("state", [""])[0]
            scope = qs.get("scope", [""])[0]
            code_challenge = qs.get("code_challenge", [""])[0]

            code = secrets.token_urlsafe(24)
            AUTH_CODES[code] = {
                "client_id": qs["client_id"][0],
                "redirect_uri": redirect_uri,
                "code_challenge": code_challenge,
                "scope": scope,
                "expires_at": time.time() + 60,  # codes are single-use, very short-lived
            }
            target = redirect_uri + "?" + urllib.parse.urlencode({"code": code, "state": state})
            self.send_response(302)
            self.send_header("Location", target)
            self.end_headers()
            return

        if parsed.path == "/api/data":
            auth_header = self.headers.get("Authorization", "")
            token = auth_header.removeprefix("Bearer ").strip()
            record = ACCESS_TOKENS.get(token)
            if not record or record["expires_at"] < time.time():
                self._send_json({"error": "invalid_or_expired_token"}, 401)
                return
            self._send_json({"message": "here is your protected data", "scope": record["scope"]})
            return

        self._send_json({"error": "not_found"}, 404)

    def do_POST(self):
        if self.path != "/token":
            self._send_json({"error": "not_found"}, 404)
            return

        length = int(self.headers.get("Content-Length", 0))
        body = urllib.parse.parse_qs(self.rfile.read(length).decode())
        grant_type = body.get("grant_type", [""])[0]

        if grant_type == "authorization_code":
            code = body.get("code", [""])[0]
            verifier = body.get("code_verifier", [""])[0]
            record = AUTH_CODES.pop(code, None)  # single-use: pop, not get

            if not record or record["expires_at"] < time.time():
                self._send_json({"error": "invalid_grant"}, 400)
                return

            # --- THE PKCE CHECK ---
            expected_challenge = b64url(hashlib.sha256(verifier.encode()).digest())
            if expected_challenge != record["code_challenge"]:
                self._send_json({"error": "invalid_grant", "detail": "PKCE verification failed"}, 400)
                return

            self._issue_tokens(record["scope"])
            return

        if grant_type == "refresh_token":
            refresh_token = body.get("refresh_token", [""])[0]
            record = REFRESH_TOKENS.pop(refresh_token, None)  # rotate: old one dies now
            if not record or record["expires_at"] < time.time():
                self._send_json({"error": "invalid_grant"}, 400)
                return
            self._issue_tokens(record["scope"])
            return

        self._send_json({"error": "unsupported_grant_type"}, 400)

    def _issue_tokens(self, scope):
        access_token = secrets.token_urlsafe(32)
        refresh_token = secrets.token_urlsafe(32)
        ACCESS_TOKENS[access_token] = {"expires_at": time.time() + ACCESS_TOKEN_TTL, "scope": scope}
        REFRESH_TOKENS[refresh_token] = {"expires_at": time.time() + REFRESH_TOKEN_TTL, "scope": scope}
        self._send_json({
            "access_token": access_token,
            "refresh_token": refresh_token,
            "token_type": "Bearer",
            "expires_in": ACCESS_TOKEN_TTL,
            "scope": scope,
        })


if __name__ == "__main__":
    port = 9000
    print(f"Authorization Server + Resource Server listening on http://localhost:{port}")
    http.server.HTTPServer(("localhost", port), Handler).serve_forever()
```

## `client.py` — The Client (Plays Claude/Kimi's Role)

```python
"""Minimal OAuth 2.0 Client using Authorization Code + PKCE, stdlib only.
Run server.py first, then this file. It opens a browser (or prints a URL),
catches the redirect on a local port, exchanges the code for tokens, calls
the protected API, waits for the access token to expire, and shows the
silent refresh happening automatically -- no re-login.
"""
import base64
import hashlib
import http.server
import json
import secrets
import threading
import time
import urllib.parse
import urllib.request
import webbrowser

AS_BASE = "http://localhost:9000"
CLIENT_ID = "demo-client"
REDIRECT_PORT = 8765
REDIRECT_URI = f"http://localhost:{REDIRECT_PORT}/callback"

received = {}


class CallbackHandler(http.server.BaseHTTPRequestHandler):
    def log_message(self, fmt, *args):
        pass

    def do_GET(self):
        qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
        received["code"] = qs.get("code", [None])[0]
        received["state"] = qs.get("state", [None])[0]
        body = b"<html><body>Login complete -- you can close this tab.</body></html>"
        self.send_response(200)
        self.send_header("Content-Type", "text/html")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)


def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode()


def post_form(url, fields):
    data = urllib.parse.urlencode(fields).encode()
    req = urllib.request.Request(url, data=data, method="POST")
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())


def get_with_bearer(url, token):
    req = urllib.request.Request(url, headers={"Authorization": f"Bearer {token}"})
    try:
        with urllib.request.urlopen(req) as resp:
            return resp.status, json.loads(resp.read())
    except urllib.error.HTTPError as e:
        return e.code, json.loads(e.read())


def main():
    # 1-2: generate PKCE verifier + challenge
    verifier = b64url(secrets.token_bytes(32))
    challenge = b64url(hashlib.sha256(verifier.encode()).digest())
    state = secrets.token_urlsafe(8)

    # start local callback listener BEFORE opening the browser
    server = http.server.HTTPServer(("localhost", REDIRECT_PORT), CallbackHandler)
    threading.Thread(target=server.handle_request, daemon=True).start()

    # 3: send the user's browser to the authorization endpoint
    auth_url = AS_BASE + "/authorize?" + urllib.parse.urlencode({
        "response_type": "code", "client_id": CLIENT_ID, "redirect_uri": REDIRECT_URI,
        "scope": "read:data", "state": state,
        "code_challenge": challenge, "code_challenge_method": "S256",
    })
    print(f"\nOpen this URL and click Allow (opening browser automatically if possible):\n{auth_url}\n")
    webbrowser.open(auth_url)

    # 4-7: wait for the redirect to land on our local listener
    while "code" not in received:
        time.sleep(0.2)
    assert received["state"] == state, "state mismatch -- possible CSRF, aborting"
    print("Received authorization code, state verified.")

    # 9: exchange code + verifier for tokens (this call never touches the browser)
    tokens = post_form(AS_BASE + "/token", {
        "grant_type": "authorization_code", "code": received["code"],
        "redirect_uri": REDIRECT_URI, "client_id": CLIENT_ID, "code_verifier": verifier,
    })
    print("Token response:", json.dumps(tokens, indent=2))

    # 12-13: call the protected API
    status, data = get_with_bearer(AS_BASE + "/api/data", tokens["access_token"])
    print(f"\nAPI call with fresh access token -> {status}: {data}")

    # Wait past expiry to prove the access token really does die
    wait_s = tokens["expires_in"] + 2
    print(f"\nWaiting {wait_s}s for the access token to expire (TTL={tokens['expires_in']}s)...")
    time.sleep(wait_s)

    status, data = get_with_bearer(AS_BASE + "/api/data", tokens["access_token"])
    print(f"API call with EXPIRED access token -> {status}: {data}")

    # Silent refresh: no browser, no login prompt, just a background POST
    print("\nToken expired -- silently refreshing using the refresh_token...")
    new_tokens = post_form(AS_BASE + "/token", {
        "grant_type": "refresh_token", "refresh_token": tokens["refresh_token"],
    })
    print("New token response:", json.dumps(new_tokens, indent=2))

    status, data = get_with_bearer(AS_BASE + "/api/data", new_tokens["access_token"])
    print(f"\nAPI call with REFRESHED access token -> {status}: {data}")


if __name__ == "__main__":
    main()
```

## Running It

```bash
# terminal 1
python3 server.py

# terminal 2
python3 client.py
```

A browser tab opens showing the demo consent screen. Click **Allow**. Watch terminal 2 — you'll see, in order: the authorization code arriving, the token exchange response (access + refresh token, `expires_in: 20`), a successful API call, a ~20-second pause, a `401 invalid_or_expired_token` once the access token dies, and then a fresh token pulled via the refresh grant with **no browser interaction at all** — the exact "quietly re-authenticated" behavior you noticed with Claude/Kimi.

## Verified Output (What This Course's Author Actually Ran)

Skipping the browser click and driving the same endpoints directly (to confirm the server logic, independent of any browser):

```
authorize status 200
approve status 302 http://localhost:8765/callback?code=BPGnEKpdQ87z...&state=teststate
token response: {'access_token': 'Xb3qqf3aJiUL...', 'refresh_token': '9fEKSfDjSGZ_...',
                  'token_type': 'Bearer', 'expires_in': 20, 'scope': 'read:data'}
api response: {'message': 'here is your protected data', 'scope': 'read:data'}
refreshed token response: {'access_token': '1xChIJ6UTpg1...', 'refresh_token': 'zgNk-lI677c...',
                            'token_type': 'Bearer', 'expires_in': 20, 'scope': 'read:data'}
PKCE failure correctly rejected: 400 {'error': 'invalid_grant', 'detail': 'PKCE verification failed'}
```

Notice the last line: swapping in a **wrong** `code_verifier` gets a hard `400 invalid_grant` — exactly the protection described in Chapter 4. Try it yourself: edit `client.py`'s `code_verifier` field to a different string right before the token exchange and confirm you get rejected too.

## Mapping Code Back to Concepts

| Code | Concept | Chapter |
|---|---|---|
| `AUTH_CODES.pop(code, None)` | Authorization codes are single-use | 4 |
| `hashlib.sha256(verifier.encode())` comparison | The PKCE check | 4 |
| `ACCESS_TOKEN_TTL = 20` | Short-lived access tokens | 5 |
| `REFRESH_TOKENS.pop(refresh_token, None)` on refresh | Refresh token rotation | 5 |
| `assert received["state"] == state` | CSRF protection via `state` | 3, 11 |
| Server plays both AS and RS roles | Common when one company hosts both | 6 |

## Summary

- You ran a real, working Authorization Code + PKCE flow, end to end, with no external dependencies.
- You watched an access token expire and get silently replaced via the refresh token — with zero browser interaction.
- You confirmed PKCE actually blocks a mismatched verifier, not just in theory.

## Knowledge Check

1. Why does `do_POST`'s `/token` handler `pop()` the authorization code and refresh token instead of just reading them?
2. What would happen if you removed the `code_challenge` comparison in `/token` entirely? Which chapter's threat model would that reopen?
3. Why does `client.py` start its local callback listener *before* opening the browser, not after?

## Hands-On Exercise

Modify `server.py` to add a `/revoke` endpoint that deletes a given refresh token from `REFRESH_TOKENS`. Then modify `client.py` to call it after the first successful API call, and confirm the subsequent refresh attempt fails with `invalid_grant` — this is exactly what happens when you click "Disconnect" on a real MCP server's settings.

## Further Reading

- [Python `http.server` documentation](https://docs.python.org/3/library/http.server.html)
- Re-read [Chapter 4](./04-authorization-code-flow-with-pkce.md)'s sequence diagram side-by-side with this code

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./08-openid-connect-vs-oauth.md">← Previous: OpenID Connect vs. OAuth</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./10-mcp-and-oauth-in-practice.md">Next: MCP and OAuth in Practice →</a>
</div>
