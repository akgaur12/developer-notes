# Chapter 07 — Load Balancers & Reverse Proxies

## Learning Objectives

By the end of this chapter, you will:
- Understand what a reverse proxy does and why you need one
- Know the difference between a forward proxy and a reverse proxy
- Understand all load balancing algorithms and when to use each
- Know health checks and connection draining
- Understand Layer 4 vs Layer 7 load balancing
- Configure nginx as a reverse proxy and load balancer

## Prerequisites

- Chapter 06 — Ports & Firewalls

---

## 7.1 What Is a Reverse Proxy?

A **forward proxy** sits in front of **clients** — clients send requests through it (used for privacy, filtering, caching).

A **reverse proxy** sits in front of **servers** — all requests from clients hit the proxy, which forwards them to backend servers.

```
Forward Proxy:                          Reverse Proxy:
Client → [Proxy] → Internet             Internet → [Proxy] → Server

User → [VPN/Squid] → google.com         Browser → [nginx] → app:8080
```

### Why Use a Reverse Proxy?

```
                    ┌─── App Server 1 (10.0.1.10:8080)
Internet ──► nginx ─┼─── App Server 2 (10.0.1.11:8080)
                    └─── App Server 3 (10.0.1.12:8080)
```

| Benefit | How |
|---------|-----|
| **SSL Termination** | Reverse proxy handles HTTPS; backends use HTTP |
| **Load Balancing** | Distribute traffic across multiple backends |
| **Caching** | Cache static assets closer to clients |
| **Compression** | Compress responses before sending |
| **Security** | Hide backend topology; WAF capabilities |
| **Single entry point** | One IP/port for many backend services |
| **Header manipulation** | Add/remove/rewrite headers |

---

## 7.2 Load Balancing Algorithms

### Round Robin (Default)

Requests distributed evenly, cycling through servers:

```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1   (cycle repeats)
```

```nginx
upstream myapp {
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
    server 10.0.1.12:8080;
}
```

**Use when:** Servers have equal capacity and requests have similar cost.

### Weighted Round Robin

Send more requests to more powerful servers:

```nginx
upstream myapp {
    server 10.0.1.10:8080 weight=3;  # gets 3 requests
    server 10.0.1.11:8080 weight=1;  # gets 1 request
    server 10.0.1.12:8080 weight=1;  # gets 1 request
}
```

**Use when:** Servers have different capacities.

### Least Connections

New request goes to server with fewest active connections:

```nginx
upstream myapp {
    least_conn;
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
    server 10.0.1.12:8080;
}
```

**Use when:** Request processing time varies significantly (some requests fast, some slow). Prevents hot servers from getting overloaded.

### IP Hash (Session Persistence / Sticky Sessions)

Same client IP always routes to same server:

```nginx
upstream myapp {
    ip_hash;
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
}
```

**Use when:** Application state is stored on the server (sessions not in Redis/DB). Not ideal for stateless apps.

### Random

Randomly selects a server:

```nginx
upstream myapp {
    random;
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
}
```

---

## 7.3 Health Checks

A load balancer should stop sending traffic to unhealthy backends:

```nginx
upstream myapp {
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;

    # Passive health check (nginx OSS)
    # Remove server from rotation if 3 failures in 30 seconds
    server 10.0.1.10:8080 max_fails=3 fail_timeout=30s;
}
```

```nginx
# Active health checks (nginx Plus or use a separate tool)
# Check /health endpoint every 5 seconds
health_check interval=5s fails=3 passes=2 uri=/health;
```

### Health Check Endpoint Best Practices

```python
# Your app should expose a /health endpoint
@app.route('/health')
def health():
    # Check dependencies
    try:
        db.execute("SELECT 1")  # test database
        redis.ping()             # test cache
        return {"status": "ok"}, 200
    except Exception as e:
        return {"status": "error", "detail": str(e)}, 503
```

```bash
# Test health endpoint
curl -s http://localhost:8080/health
# {"status": "ok"}
```

---

## 7.4 Layer 4 vs Layer 7 Load Balancing

### Layer 4 (Transport Layer)

- Operates on TCP/UDP connections
- Routes based on IP:port only — can't see HTTP content
- Faster (no HTTP parsing)
- Cannot route based on URL path or hostname

```
TCP connection arrives on :80
Layer 4 LB picks a backend and forwards the raw TCP stream
```

Tools: HAProxy (TCP mode), AWS NLB, iptables with DNAT

### Layer 7 (Application Layer)

- Operates on HTTP/HTTPS content
- Can route based on URL, hostname, headers, cookies
- Enables URL-based routing, A/B testing, canary deployments
- Slightly higher overhead (parses HTTP)

```
GET /api/users → route to API servers
GET /static/   → route to static file servers
Host: admin.example.com → route to admin backend
```

Tools: nginx, HAProxy (HTTP mode), AWS ALB, Traefik

---

## 7.5 Content-Based Routing (URL Routing)

```nginx
server {
    listen 80;
    server_name api.example.com;

    # Route /api/ to API backend
    location /api/ {
        proxy_pass http://api_upstream/;
    }

    # Route /static/ to static file server
    location /static/ {
        proxy_pass http://static_upstream/;
    }

    # Route /admin/ to admin backend (restricted)
    location /admin/ {
        allow 10.0.0.0/8;
        deny all;
        proxy_pass http://admin_upstream/;
    }
}
```

---

## 7.6 Connection Draining (Graceful Removal)

When removing a server from rotation (deployment, maintenance), **drain** existing connections first:

```
1. Mark server as "draining" (no new connections)
2. Existing connections finish naturally
3. Once all connections done, server removed
4. Deploy/restart server
5. Add back to rotation
```

```nginx
# Manually take server out of rotation
upstream myapp {
    server 10.0.1.10:8080;
    server 10.0.1.11:8080 down;   # "down" removes from rotation
}
```

In Kubernetes: graceful rollouts drain pods automatically.

---

## 7.7 SSL Termination vs SSL Passthrough

### SSL Termination (Most Common)

Reverse proxy decrypts HTTPS, communicates with backends over HTTP:

```
Client ──[HTTPS]──► nginx ──[HTTP]──► App Server
         encrypted        unencrypted (internal network)
```

Benefits: Single certificate to manage, backend code is simpler, can inspect/modify HTTP headers.

### SSL Passthrough

Reverse proxy forwards encrypted traffic without decrypting:

```
Client ──[HTTPS]──► nginx ──[HTTPS]──► App Server (handles TLS)
         encrypted          encrypted
```

Benefits: End-to-end encryption, backend handles certificates.

```nginx
# SSL passthrough (stream module, Layer 4)
stream {
    server {
        listen 443;
        proxy_pass backend:443;  # raw TCP forwarding
    }
}
```

---

## 7.8 Practical Load Balancer Setup

```nginx
# /etc/nginx/sites-available/load-balancer

upstream app_servers {
    least_conn;                          # algorithm
    server 10.0.1.10:8080 weight=2;     # weight
    server 10.0.1.11:8080 weight=1;
    server 10.0.1.12:8080 backup;       # only used when others fail

    keepalive 32;                        # keep 32 connections to backends
}

server {
    listen 80;
    server_name app.example.com;
    return 301 https://$server_name$request_uri;  # redirect to HTTPS
}

server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    location / {
        proxy_pass http://app_servers;

        # Essential proxy headers
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout  5s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;
    }

    location /health {
        access_log off;
        return 200 "healthy\n";
    }
}
```

---

## Summary

```
Reverse Proxy: sits in front of servers
  Benefits: SSL termination, load balancing, security, caching

Load Balancing Algorithms:
  Round Robin     — equal distribution (default)
  Weighted        — proportional to server capacity
  Least Conn      — least active connections (variable request times)
  IP Hash         — same client → same server (sticky sessions)

L4 LB: TCP/UDP — fast, no HTTP awareness
L7 LB: HTTP    — URL routing, header inspection, full HTTP features

Health checks: removes failed backends automatically
SSL termination: decrypt at proxy, HTTP to backends
```

---

## Knowledge Check

1. What is the difference between a forward proxy and a reverse proxy?
2. When would you use least-connections over round-robin?
3. What are sticky sessions and why might they be a problem?
4. What is SSL termination and what are its benefits?
5. What is a health check and why is it important?

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="06-ports-and-firewalls.md">← Previous: Ports & Firewalls</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="08-nginx-basics.md">Next: Nginx Basics →</a>
</div>
