# 📘 Basics of JWT (JSON Web Token)

This document explains **JWT (JSON Web Token)** from fundamentals to practical usage, with examples suitable for backend developers (FastAPI / Node / Django) and modern web & GenAI systems.

---

## 📌 What is JWT?

**JWT (JSON Web Token)** is a compact, URL-safe token used to **securely transmit information** between two parties (client and server).

> In simple words:
> **JWT is a signed proof that the server trusts.**

It is most commonly used for:

* Authentication (login systems)
* Authorization (who can access what)
* Stateless APIs

---

## ❓ Why JWT Exists (The Problem It Solves)

After login, a server must answer:

> “How do I know this user is authenticated on the next request?”

### ❌ Traditional Session-Based Auth

* Server stores sessions (Redis / DB)
* Requires state
* Hard to scale
* Painful in microservices

### ✅ JWT-Based Auth

* No server-side session storage
* Token carries authentication proof
* Server only verifies token signature

---

## 🧠 Key Properties of JWT

| Property       | Description                    |
| -------------- | ------------------------------ |
| Stateless      | Server does not store sessions |
| Self-contained | Token contains user info       |
| Signed         | Cannot be tampered with        |
| Compact        | Safe for headers & URLs        |

---

## 🧩 JWT Structure

A JWT looks like this:

```
xxxxx.yyyyy.zzzzz
```

It has **3 parts**:

```
HEADER . PAYLOAD . SIGNATURE
```

---

## 1️⃣ Header

Describes **how the token is signed**.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

* `alg` → signing algorithm
* `typ` → token type

Base64 encoded.

---

## 2️⃣ Payload (Claims)

Contains **data about the user**.

```json
{
  "sub": "akash@gmail.com",
  "exp": 1700000000,
  "iat": 1699996400
}
```

### Common Claims

| Claim | Meaning                   |
| ----- | ------------------------- |
| `sub` | Subject (user id / email) |
| `exp` | Expiration time           |
| `iat` | Issued at                 |
| `nbf` | Not before                |
| `iss` | Issuer                    |

⚠️ Payload is **NOT encrypted** — only encoded.

---

## 3️⃣ Signature (Security Layer)

Signature ensures the token is **authentic and untampered**.

```
HMACSHA256(
  base64(header) + "." + base64(payload),
  SECRET_KEY
)
```

✔ Only the server knows the `SECRET_KEY`
✔ Client cannot forge tokens

---

## 🔐 JWT Authentication Flow

### 1️⃣ User Logs In

```
POST /login
{ email, password }
```

### 2️⃣ Server Verifies Credentials

* Password hash matches

### 3️⃣ Server Issues JWT

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

### 4️⃣ Client Stores Token

* LocalStorage / Memory / Cookie

### 5️⃣ Client Sends Token

```
Authorization: Bearer <JWT>
```

### 6️⃣ Server Verifies Token

* Signature
* Expiry
* Claims

---

## 🧪 JWT Example (Python)

### Create Token

```python
from jose import jwt
from datetime import datetime, timedelta

payload = {
    "sub": "user@email.com",
    "exp": datetime.utcnow() + timedelta(minutes=30)
}

token = jwt.encode(payload, "SECRET_KEY", algorithm="HS256")
```

### Decode Token

```python
jwt.decode(token, "SECRET_KEY", algorithms=["HS256"])
```

---

## ⏳ Token Expiration (`exp`)

JWT **must expire**.

Why?

* Prevents permanent access
* Limits damage if token leaks

Best practice:

* Access token: **5–30 minutes**
* Refresh token: **days / weeks**

---

## 🔄 Access Token vs Refresh Token

| Token         | Purpose              | Lifetime |
| ------------- | -------------------- | -------- |
| Access Token  | API calls            | Short    |
| Refresh Token | Get new access token | Long     |

---

## 🚪 Logout in JWT (Important Concept)

JWT is stateless → logout is **not automatic**.

### Common Logout Strategies

1. Client deletes token
2. Short token expiry
3. Token blacklist
4. Token versioning

---

## ⚠️ Common JWT Mistakes

❌ Storing passwords in JWT
❌ Long-lived access tokens
❌ No expiration
❌ Using weak secret keys
❌ Trusting payload without verifying signature

---

## ✅ JWT Best Practices

✔ Use strong `SECRET_KEY`
✔ Keep tokens short-lived
✔ Use HTTPS only
✔ Validate on every request
✔ Separate access & refresh tokens
✔ Store minimal data in payload

---

## 🧠 JWT in Modern Architectures

JWT is ideal for:

* FastAPI / Node.js APIs
* Microservices
* Mobile apps
* SPAs (React / Next.js)
* GenAI user isolation

---

## 🧾 JWT vs Sessions

| Feature      | JWT    | Session |
| ------------ | ------ | ------- |
| Server state | ❌      | ✅       |
| Scalability  | High   | Medium  |
| Storage      | Client | Server  |
| Revocation   | Hard   | Easy    |

---

## 🏁 Final Summary

> **JWT is a stateless, signed token that allows servers to authenticate users without storing sessions.**

Used correctly, JWT provides **scalable, secure authentication** for modern applications.

---

## 📚 Further Reading

* RFC 7519 (JWT Standard)
* OAuth 2.0
* OpenID Connect

---

✅ End of document
