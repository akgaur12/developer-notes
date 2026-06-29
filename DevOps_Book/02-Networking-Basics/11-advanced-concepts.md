# Chapter 11 — Advanced Networking Concepts

## Learning Objectives

By the end of this chapter, you will:
- Understand HTTP/2 multiplexing and HTTP/3 / QUIC
- Work with WebSockets for real-time communication
- Understand gRPC and protobuf for service communication
- Implement mutual TLS (mTLS) between services
- Understand BGP at a conceptual level
- Know service mesh basics (Istio/Envoy pattern)

## Prerequisites

- Chapter 10 — Intermediate Networking Concepts

---

## 11.1 HTTP/2 in Depth

HTTP/2 solves the **head-of-line blocking** problem of HTTP/1.1 by multiplexing multiple requests over a single TCP connection.

### HTTP/1.1 vs HTTP/2

```
HTTP/1.1:
Connection 1:  [Request A ────────────────► Response A]
Connection 2:  [Request B ──────────► Response B]
Connection 3:  [Request C ───────────────────► Response C]
(browser opens 6 parallel connections per domain)

HTTP/2:
Connection 1:  Stream 1 [Request A ──► Response A]
               Stream 3 [Request B ──► Response B]
               Stream 5 [Request C ──► Response C]
(one connection, parallel streams)
```

### Key HTTP/2 Features

**Header Compression (HPACK):**
```
HTTP/1.1: headers repeated in every request (text)
  User-Agent: Mozilla/5.0...    (500+ bytes, same every time)
  Accept-Encoding: gzip...
  Cookie: session=abc...

HTTP/2: HPACK compresses repeated headers using a dictionary
  (80-90% compression ratio for headers)
```

**Server Push:**
```
Client: GET /index.html
Server: here's index.html...
        AND I'm pushing style.css (you'll need it)
        AND I'm pushing app.js (you'll need it)
```

```nginx
# Enable HTTP/2 in nginx
server {
    listen 443 ssl http2;    # ← just add "http2" here
    ...
}

# Server push
location = /index.html {
    http2_push /static/style.css;
    http2_push /static/app.js;
}
```

---

## 11.2 HTTP/3 and QUIC

HTTP/3 replaces TCP with **QUIC** (built on UDP):

```
HTTP/1.1:  [HTTP]     [TCP]     [IP]
HTTP/2:    [HTTP]     [TCP+TLS] [IP]
HTTP/3:    [HTTP]     [QUIC]    [IP]
                      (UDP-based, built-in TLS)
```

### Why QUIC?

```
TCP problem — head-of-line blocking at transport layer:
  Packet lost ───► TCP waits for retransmit ───► ALL streams stall
                                                  (even unrelated ones)

QUIC solution:
  Packet lost on stream 1 ───► only stream 1 stalls
                                streams 2, 3, 4 continue
```

Other benefits:
- **0-RTT connection** — can send data with first packet (revisited connections)
- **Connection migration** — switch IP (WiFi → 4G) without reconnecting
- **Built-in TLS 1.3**

```bash
# Check HTTP version curl uses
curl -vI --http3 https://cloudflare.com 2>&1 | grep "HTTP/"
```

---

## 11.3 WebSockets

**WebSockets** provide full-duplex communication over a single TCP connection — client and server can send messages to each other at any time.

### WebSocket Handshake

```
1. Client sends HTTP Upgrade request:
   GET /chat HTTP/1.1
   Host: example.com
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
   Sec-WebSocket-Version: 13

2. Server responds:
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

3. Both sides now send frames (not HTTP) bidirectionally
```

### Nginx WebSocket Proxy

```nginx
server {
    listen 80;
    server_name ws.example.com;

    location /ws/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;       # pass upgrade header
        proxy_set_header Connection "Upgrade";         # keep connection open
        proxy_set_header Host $host;
        proxy_read_timeout 3600s;                      # long timeout for WS
    }
}
```

### Testing WebSockets

```bash
# Install websocat
sudo apt install websocat -y
# Or: cargo install websocat

# Connect to WebSocket
websocat ws://localhost:8080/ws

# Test with curl (just handshake)
curl -v \
  --no-buffer \
  -H "Upgrade: websocket" \
  -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  http://localhost:8080/ws
```

---

## 11.4 gRPC

**gRPC** is a high-performance RPC framework using Protocol Buffers (protobuf) for serialization and HTTP/2 for transport.

### Why gRPC over REST?

| | REST/JSON | gRPC/protobuf |
|--|----------|--------------|
| **Serialization** | Text JSON (verbose) | Binary protobuf (compact) |
| **Performance** | Slower | 5-10x faster serialization |
| **Schema** | Optional (OpenAPI) | Required `.proto` files |
| **Streaming** | SSE or WebSocket | Native bidirectional streaming |
| **Language support** | Universal | Generated clients for 10+ languages |
| **Browser support** | Native | Needs grpc-web proxy |

### Service Definition

```protobuf
// user.proto
syntax = "proto3";

service UserService {
    rpc GetUser (GetUserRequest) returns (User);
    rpc ListUsers (ListUsersRequest) returns (stream User);  // server streaming
    rpc CreateUser (stream CreateUserRequest) returns (User); // client streaming
}

message GetUserRequest {
    string user_id = 1;
}

message User {
    string id = 1;
    string name = 2;
    string email = 3;
    int64 created_at = 4;
}
```

```bash
# Generate code from proto
protoc --go_out=. --go-grpc_out=. user.proto
protoc --python_out=. --grpc_python_out=. user.proto

# Test gRPC with grpcurl
grpcurl -plaintext localhost:50051 list
grpcurl -plaintext localhost:50051 UserService/GetUser
grpcurl -plaintext -d '{"user_id": "123"}' localhost:50051 UserService.GetUser
```

### gRPC Nginx Proxy

```nginx
# gRPC requires HTTP/2
upstream grpc_backend {
    server localhost:50051;
}

server {
    listen 443 ssl http2;
    server_name grpc.example.com;

    ssl_certificate ...;
    ssl_certificate_key ...;

    location / {
        grpc_pass grpc://grpc_backend;    # use grpc_pass not proxy_pass
        grpc_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 11.5 Mutual TLS (mTLS)

Standard TLS: **server** presents certificate → client verifies server identity.  
**mTLS**: both server AND client present certificates → mutual verification.

```
Standard TLS:              mTLS:
Client ──► Server cert     Client cert ──► Server
           Verify                         Verify
                           Client ──► Server cert
                                     Verify
```

### Use Case

Microservices authentication — each service has a certificate. Service A cannot talk to Service B without a valid cert. This replaces API keys/tokens for service-to-service auth.

```bash
# Generate CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 1825 -key ca.key -out ca.crt -subj "/CN=My-CA"

# Generate server cert signed by CA
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -subj "/CN=server.example.com"
openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt

# Generate client cert signed by same CA
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr -subj "/CN=client-service"
openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt
```

```nginx
# nginx — require client certificate
server {
    listen 443 ssl;

    ssl_certificate server.crt;
    ssl_certificate_key server.key;

    # Require client cert signed by our CA
    ssl_client_certificate ca.crt;
    ssl_verify_client on;

    location / {
        # Access client cert info in headers
        proxy_set_header X-Client-Cert-CN $ssl_client_s_dn_cn;
        proxy_pass http://backend;
    }
}
```

```bash
# Connect with client certificate
curl --cert client.crt --key client.key --cacert ca.crt https://server.example.com
```

---

## 11.6 BGP — Border Gateway Protocol (Conceptual)

**BGP** is how the internet's routing table is maintained — it's the protocol that makes the internet work.

```
Internet = ~70,000 Autonomous Systems (AS), each with an AS number
  AS 15169 = Google
  AS 16509 = Amazon (AWS)
  AS 13335 = Cloudflare

BGP tells routers: "To reach 8.8.8.8, go through AS 15169"
```

### Why DevOps Engineers Need to Know BGP

- **Anycast** (how CDNs and DNS work) — same IP announced from multiple locations
- **BGP hijacking** — misconfigured BGP can cause outages (Pakistan Telecom hijacking YouTube, 2008)
- **Cloud Direct Connect** — AWS Direct Connect and GCP Cloud Interconnect use BGP to advertise routes
- **MetalLB** — Kubernetes load balancer that uses BGP to advertise service IPs to routers

```bash
# Check BGP route for an IP (using public looking glass)
# bgp.he.net shows real BGP routing info

# From a server with BGP tools:
# whois 8.8.8.8 | grep -E "origin|route"
# route:  8.8.8.0/24
# origin: AS15169
```

---

## 11.7 Service Mesh (Pattern)

As microservices scale, managing networking between them becomes complex. A **service mesh** adds a sidecar proxy to every service pod:

```
Without mesh:
  Service A ──► Service B  (no auth, no mTLS, no tracing)

With mesh (Istio + Envoy):
  Service A ──► [Envoy Proxy] ──► [Envoy Proxy] ──► Service B
                mTLS, tracing,     mTLS, tracing,
                circuit breaking   rate limiting
```

Service meshes like **Istio** and **Linkerd** handle:
- **mTLS** between all services automatically
- **Distributed tracing** (Jaeger, Zipkin integration)
- **Traffic management** (canary deployments, A/B testing)
- **Circuit breaking** (stop calls to failing services)
- **Rate limiting** and **retry policies**

---

## Summary

```
HTTP/2:    Multiplexed streams, header compression, server push
HTTP/3:    QUIC (UDP-based), no TCP head-of-line blocking, 0-RTT
WebSocket: Full-duplex over HTTP upgrade; nginx: proxy_set_header Upgrade
gRPC:      HTTP/2 + protobuf; faster than REST; strong schema
mTLS:      Both sides present certs; service-to-service auth
BGP:       Internet routing protocol; anycast; cloud direct connect
Mesh:      Sidecar proxies (Envoy) for all service networking concerns
```

---

## Knowledge Check

1. What problem does HTTP/2 multiplexing solve that HTTP/1.1 has?
2. What does `101 Switching Protocols` mean in a WebSocket context?
3. Why would you use gRPC over REST for internal service communication?
4. What is the difference between standard TLS and mTLS?
5. What is anycast and which protocol enables it?

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="10-intermediate-concepts.md">← Previous: Intermediate Concepts</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="12-best-practices.md">Next: Best Practices →</a>
</div>
