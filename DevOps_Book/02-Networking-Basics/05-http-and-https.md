# Chapter 05 — HTTP & HTTPS

## Learning Objectives

By the end of this chapter, you will:
- Understand the HTTP request/response cycle in full detail
- Know all HTTP methods and when to use each
- Read and set HTTP headers confidently
- Understand HTTP status codes and what they mean
- Know how HTTPS/TLS works and why it matters
- Inspect real HTTP traffic with `curl -v`
- Understand HTTP/1.1 vs HTTP/2 differences

## Prerequisites

- Chapter 04 — DNS

---

## 5.1 What Is HTTP?

**HTTP (HyperText Transfer Protocol)** is the application-layer protocol for transferring data on the web. It is a **text-based, stateless, request-response** protocol running over TCP.

- **Text-based**: requests and responses are human-readable text (HTTP/1.1)
- **Stateless**: each request is independent; server doesn't remember previous requests
- **Request-response**: client sends request, server sends exactly one response

```
Client (browser, curl, app)          Server (nginx, Apache, your app)
          │                                       │
          │── GET /api/users HTTP/1.1 ───────────►│
          │   Host: api.example.com               │
          │   Accept: application/json            │
          │                                       │
          │◄── HTTP/1.1 200 OK ───────────────────│
          │    Content-Type: application/json     │
          │                                       │
          │    [{"id":1,"name":"Akash"}]          │
```

---

## 5.2 HTTP Request Structure

```
METHOD  PATH                    HTTP-VERSION
GET     /api/users?page=1       HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer mytoken123
User-Agent: curl/7.81.0
Content-Type: application/json
                                          ← blank line separates headers from body
{"query": "search term"}                  ← request body (only for POST/PUT/PATCH)
```

### Anatomy

```
Request Line:  METHOD  URL-PATH  HTTP/VERSION
Headers:       Key: Value  (one per line)
Blank line:    mandatory separator
Body:          optional (POST/PUT/PATCH)
```

---

## 5.3 HTTP Methods

| Method | Purpose | Has Body | Idempotent | Safe |
|--------|---------|----------|------------|------|
| `GET` | Retrieve data | No | Yes | Yes |
| `POST` | Create resource | Yes | No | No |
| `PUT` | Replace resource | Yes | Yes | No |
| `PATCH` | Partial update | Yes | No | No |
| `DELETE` | Remove resource | Optional | Yes | No |
| `HEAD` | GET without body | No | Yes | Yes |
| `OPTIONS` | List allowed methods | No | Yes | Yes |

**Idempotent**: calling multiple times has same effect as calling once  
**Safe**: doesn't modify data (only reads)

```bash
# GET — retrieve
curl https://api.example.com/users

# POST — create
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Akash", "email": "akash@example.com"}'

# PUT — replace entire resource
curl -X PUT https://api.example.com/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name": "Akash Gaur", "email": "akash@example.com"}'

# PATCH — update specific fields
curl -X PATCH https://api.example.com/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name": "Akash Gaur"}'

# DELETE — remove
curl -X DELETE https://api.example.com/users/1

# HEAD — get headers only (check if resource exists, get size)
curl -I https://example.com/largefile.zip
```

---

## 5.4 HTTP Response Structure

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 47
Cache-Control: max-age=300
X-Request-Id: abc123

{"id": 1, "name": "Akash", "email": "akash@example.com"}
```

### Response Structure

```
Status Line:  HTTP/VERSION  STATUS-CODE  REASON-PHRASE
Headers:      Key: Value
Blank line:   separator
Body:         response data (JSON, HTML, binary, etc.)
```

---

## 5.5 HTTP Status Codes

### 1xx — Informational

| Code | Name | Meaning |
|------|------|---------|
| 100 | Continue | Server received headers, send body |
| 101 | Switching Protocols | Upgrading to WebSocket |

### 2xx — Success

| Code | Name | Meaning |
|------|------|---------|
| 200 | OK | Standard success |
| 201 | Created | Resource created (POST response) |
| 202 | Accepted | Request accepted, processing async |
| 204 | No Content | Success, no body (DELETE response) |
| 206 | Partial Content | Range request fulfilled |

### 3xx — Redirection

| Code | Name | Meaning |
|------|------|---------|
| 301 | Moved Permanently | Resource moved forever (update bookmarks) |
| 302 | Found | Temporary redirect |
| 304 | Not Modified | Use cached version |
| 307 | Temporary Redirect | Temporary, keep method |
| 308 | Permanent Redirect | Permanent, keep method |

### 4xx — Client Errors

| Code | Name | Meaning |
|------|------|---------|
| 400 | Bad Request | Malformed request syntax |
| 401 | Unauthorized | Not authenticated (need to login) |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 405 | Method Not Allowed | Wrong HTTP method |
| 408 | Request Timeout | Client too slow |
| 409 | Conflict | State conflict (duplicate, version mismatch) |
| 410 | Gone | Resource permanently deleted |
| 422 | Unprocessable Entity | Validation failed |
| 429 | Too Many Requests | Rate limited |

### 5xx — Server Errors

| Code | Name | Meaning |
|------|------|---------|
| 500 | Internal Server Error | Generic server error (check logs!) |
| 501 | Not Implemented | Method not supported |
| 502 | Bad Gateway | Proxy got bad response from upstream |
| 503 | Service Unavailable | Server down or overloaded |
| 504 | Gateway Timeout | Proxy timeout waiting for upstream |

> **Quick memory:**
> - 4xx = YOUR fault (bad request)
> - 5xx = SERVER's fault (backend problem)
> - 502 = your proxy is fine, the backend is broken
> - 504 = your proxy is fine, the backend is too slow

```bash
# Check HTTP status code
curl -s -o /dev/null -w "%{http_code}\n" https://example.com

# Loop until service returns 200
until [[ $(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health) == "200" ]]; do
    echo "Waiting for service..."
    sleep 2
done
echo "Service is up!"
```

---

## 5.6 HTTP Headers

Headers are key-value metadata for request and response.

### Common Request Headers

| Header | Purpose | Example |
|--------|---------|---------|
| `Host` | Target server (required in HTTP/1.1) | `api.example.com` |
| `Accept` | Acceptable response formats | `application/json, text/html` |
| `Content-Type` | Format of request body | `application/json` |
| `Content-Length` | Size of request body | `47` |
| `Authorization` | Auth credentials | `Bearer token123` |
| `Cookie` | Session cookies | `session=abc123` |
| `User-Agent` | Client identifier | `Mozilla/5.0 ...` |
| `Referer` | Page that linked here | `https://google.com` |
| `Origin` | CORS: request origin | `https://myapp.com` |
| `X-Request-Id` | Distributed tracing ID | `abc-123-xyz` |
| `Cache-Control` | Caching directives | `no-cache` |

### Common Response Headers

| Header | Purpose | Example |
|--------|---------|---------|
| `Content-Type` | Response body format | `application/json; charset=utf-8` |
| `Content-Length` | Response body size | `1234` |
| `Set-Cookie` | Set a cookie | `session=abc; HttpOnly; Secure` |
| `Location` | Redirect target (3xx) | `https://example.com/new-path` |
| `Cache-Control` | How to cache | `max-age=3600, public` |
| `ETag` | Resource version for caching | `"abc123"` |
| `X-Frame-Options` | Prevent clickjacking | `DENY` |
| `Strict-Transport-Security` | Force HTTPS | `max-age=31536000; includeSubDomains` |
| `Access-Control-Allow-Origin` | CORS policy | `*` or `https://myapp.com` |

```bash
# See all request and response headers
curl -v https://github.com 2>&1 | grep -E "^[<>]"
# Lines starting with > = sent headers
# Lines starting with < = received headers

# Just response headers
curl -I https://github.com

# Send custom header
curl -H "X-Custom-Header: value" https://api.example.com
curl -H "Accept: application/json" https://api.example.com/data
```

---

## 5.7 HTTPS and TLS

**HTTPS = HTTP over TLS (Transport Layer Security)**. Everything is encrypted.

### Why HTTPS?

Without HTTPS (plain HTTP):
- Anyone on your network can read your traffic
- Passwords, tokens, personal data all visible
- Attackers can modify content in transit (MITM)

With HTTPS:
- All traffic is encrypted
- Server identity is verified (certificate)
- Content cannot be tampered with

### The TLS Handshake

```
Client                                      Server
  │                                            │
  │─── ClientHello ───────────────────────────►│
  │    (TLS version, cipher suites, random)    │
  │                                            │
  │◄── ServerHello ───────────────────────────│
  │    (chosen cipher, server random,          │
  │     server certificate)                    │
  │                                            │
  │    [Client verifies certificate:]          │
  │    Is it signed by a trusted CA?           │
  │    Is the domain name correct?             │
  │    Is it expired?                          │
  │                                            │
  │─── ClientKeyExchange ─────────────────────►│
  │    (pre-master secret, encrypted           │
  │     with server's public key)              │
  │                                            │
  │    [Both derive session keys from          │
  │     randoms + pre-master secret]           │
  │                                            │
  │─── Finished (encrypted) ──────────────────►│
  │◄── Finished (encrypted) ───────────────────│
  │                                            │
  │══════════ Encrypted HTTP starts ═══════════│
```

### TLS Certificates

A **certificate** proves the server's identity. It contains:
- Domain name (e.g., `github.com`)
- Public key
- Validity period (not before, not after)
- Issuer (Certificate Authority)
- Digital signature from CA

```bash
# View a certificate
echo | openssl s_client -connect github.com:443 2>/dev/null | openssl x509 -text | head -40

# Check certificate expiry
echo | openssl s_client -connect github.com:443 2>/dev/null | openssl x509 -noout -dates

# Quick expiry check for multiple domains
for domain in github.com google.com; do
    expiry=$(echo | openssl s_client -connect "$domain:443" 2>/dev/null | openssl x509 -noout -enddate 2>/dev/null)
    echo "$domain: $expiry"
done
```

### Let's Encrypt — Free TLS Certificates

```bash
# Get free certificate with certbot
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Auto-renew (certbot creates a cron/systemd timer)
sudo certbot renew --dry-run

# Check certificate status
sudo certbot certificates
```

---

## 5.8 HTTP/1.1 vs HTTP/2 vs HTTP/3

### HTTP/1.1
- One request per TCP connection (or keep-alive for reuse)
- Text-based headers (verbose)
- Head-of-line blocking: one slow request blocks others

### HTTP/2
- **Multiplexing**: multiple requests over one TCP connection simultaneously
- **Header compression**: binary headers (HPACK)
- **Server push**: server can proactively send resources
- Same TLS/TCP foundation as HTTP/1.1

### HTTP/3
- Based on **QUIC** (over UDP, not TCP)
- Eliminates TCP head-of-line blocking
- Faster connection setup (0-RTT)
- Better on lossy networks (mobile)

```bash
# Check which HTTP version is used
curl -v --http2 https://github.com 2>&1 | grep "< HTTP/"
curl -v --http1.1 https://github.com 2>&1 | grep "< HTTP/"

# Check if server supports HTTP/2
curl -sI --http2 https://github.com | head -1
# HTTP/2 200   ← uses HTTP/2

curl -sI https://github.com | head -1
# HTTP/2 200   ← curl auto-negotiates
```

---

## 5.9 Cookies and Sessions

HTTP is stateless. **Cookies** let servers remember state across requests:

```
First request (login):
  POST /login → {username, password}
  ← 200 OK
    Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict

Subsequent requests:
  GET /dashboard
  Cookie: session=abc123
  ← 200 OK (server looks up session, knows who you are)
```

```bash
# Follow and use cookies with curl
curl -c cookies.txt -b cookies.txt https://example.com/login \
  -d "username=admin&password=secret"

# Send specific cookie
curl -b "session=abc123" https://example.com/dashboard
```

---

## Summary

```
HTTP Request:   METHOD PATH HTTP/VERSION\r\nHeaders\r\n\r\nBody
HTTP Response:  HTTP/VERSION STATUS REASON\r\nHeaders\r\n\r\nBody

Methods: GET(read) POST(create) PUT(replace) PATCH(update) DELETE(remove)

Status codes:
  2xx = success    3xx = redirect
  4xx = client err  5xx = server err

HTTPS = HTTP + TLS (encryption + authentication)
  - Certificate proves server identity
  - Issued by CA, verified by browser
  - Let's Encrypt = free certificates

HTTP/2: multiplexed, binary, compressed headers
HTTP/3: UDP-based (QUIC), faster on mobile
```

---

## Knowledge Check

1. What is the difference between 401 and 403?
2. What does a 502 Bad Gateway tell you about where the failure is?
3. What is the difference between `PUT` and `PATCH`?
4. What does TLS provide that plain HTTP doesn't?
5. What does the `Host` header do in an HTTP request?

---

## Hands-On Exercise

```bash
# 1. Inspect a real HTTP conversation
curl -v https://httpbin.org/get 2>&1

# 2. Test all common HTTP methods
BASE="https://httpbin.org"
curl -s $BASE/get | python3 -m json.tool | head -10        # GET
curl -s -X POST $BASE/post -d "key=value" | python3 -m json.tool | head -10  # POST
curl -s -I $BASE/get | head -5                             # HEAD

# 3. HTTP status code exploration
for code in 200 301 404 500; do
    echo "=== $code ==="
    curl -s -o /dev/null -w "%{http_code}" "https://httpbin.org/status/$code"
    echo
done

# 4. View certificate details
echo | openssl s_client -connect github.com:443 -servername github.com 2>/dev/null \
  | openssl x509 -noout -subject -dates -issuer

# 5. Test response timing
curl -w "DNS: %{time_namelookup}s\nTCP: %{time_connect}s\nTLS: %{time_appconnect}s\nFirst byte: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
     -o /dev/null -s https://github.com
```

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="04-dns.md">← Previous: DNS</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="06-ports-and-firewalls.md">Next: Ports & Firewalls →</a>
</div>
