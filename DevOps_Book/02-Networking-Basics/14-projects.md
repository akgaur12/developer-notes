# Chapter 14 — Hands-On Projects

## Overview

Four projects that take you from beginner to capstone level, applying everything learned in this course. Each project builds on the previous one.

---

## Project 1 — Beginner: Nginx Static Site with HTTPS

**Goal:** Deploy a static website with nginx, configure HTTPS with a self-signed cert, set up proper security headers.

**Skills:** nginx config, TLS, firewall, curl debugging

### Requirements

- Serve static HTML/CSS from `/var/www/mysite`
- HTTPS with TLS 1.2+ (self-signed cert for local, Let's Encrypt for real domain)
- HTTP → HTTPS redirect
- Security headers (HSTS, X-Frame-Options, X-Content-Type-Options)
- UFW firewall (only 22, 80, 443 open)
- Access log in custom format

### Implementation

```bash
# 1. Create project directory
sudo mkdir -p /var/www/mysite
sudo tee /var/www/mysite/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>My DevOps Site</title></head>
<body>
  <h1>Hello from nginx!</h1>
  <p>Served over HTTPS with nginx.</p>
</body>
</html>
EOF

# 2. Generate self-signed certificate (local testing)
sudo mkdir -p /etc/ssl/mysite
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/mysite/private.key \
  -out /etc/ssl/mysite/cert.crt \
  -subj "/CN=localhost"

# 3. Create nginx config
sudo tee /etc/nginx/sites-available/mysite << 'EOF'
# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name localhost;
    return 301 https://$host$request_uri;
}

# HTTPS site
server {
    listen 443 ssl;
    server_name localhost;

    root /var/www/mysite;
    index index.html;

    ssl_certificate     /etc/ssl/mysite/cert.crt;
    ssl_certificate_key /etc/ssl/mysite/private.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    # Security headers
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Custom log format
    log_format myformat '$remote_addr [$time_local] "$request" $status $body_bytes_sent "$http_user_agent" ${request_time}s';
    access_log /var/log/nginx/mysite.log myformat;

    location / {
        try_files $uri $uri/ =404;
    }

    # Cache static assets
    location ~* \.(css|js|png|jpg|ico)$ {
        expires 7d;
        add_header Cache-Control "public";
    }
}
EOF

# 4. Enable and test
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# 5. Configure firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# 6. Test
curl -k https://localhost                          # -k skips cert verification
curl -v -k https://localhost 2>&1 | grep -i "HTTP\|ssl\|header"
curl -I http://localhost                           # should redirect to HTTPS
```

### Verification Checklist

```bash
□ curl http://localhost returns 301 redirect
□ curl -k https://localhost returns 200 with HTML
□ curl -kI https://localhost shows security headers
□ sudo ufw status shows only 22/80/443
□ tail /var/log/nginx/mysite.log shows custom format
```

---

## Project 2 — Intermediate: Reverse Proxy with Multiple Backends

**Goal:** Build a nginx reverse proxy routing to multiple backend services based on URL path.

**Skills:** nginx upstream, load balancing, health checks, Docker networking

### Architecture

```
Internet → nginx:80/443
  /api/v1/  → api-v1-server:8001
  /api/v2/  → api-v2-server:8002
  /         → web-frontend:3000
  /static/  → served directly from disk
```

### Implementation with Docker

```bash
# Create Docker network
docker network create proxy-demo

# Start multiple backends
docker run -d --name api-v1 --network proxy-demo \
  -e PORT=8001 \
  python:3.11-slim python3 -c "
import http.server, socketserver
class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-Type','application/json')
        self.end_headers()
        self.wfile.write(b'{\"version\":\"v1\",\"message\":\"API v1 response\"}')
    def log_message(self, *args): pass
with socketserver.TCPServer(('0.0.0.0', 8001), H) as s: s.serve_forever()
"

docker run -d --name api-v2 --network proxy-demo \
  -e PORT=8002 \
  python:3.11-slim python3 -c "
import http.server, socketserver
class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-Type','application/json')
        self.end_headers()
        self.wfile.write(b'{\"version\":\"v2\",\"message\":\"API v2 response\"}')
    def log_message(self, *args): pass
with socketserver.TCPServer(('0.0.0.0', 8002), H) as s: s.serve_forever()
"

# nginx config
sudo tee /etc/nginx/sites-available/proxy-demo << 'EOF'
upstream api_v1 {
    server api-v1:8001 max_fails=2 fail_timeout=10s;
}

upstream api_v2 {
    server api-v2:8002 max_fails=2 fail_timeout=10s;
}

server {
    listen 80;
    server_name localhost;

    # API v1
    location /api/v1/ {
        proxy_pass http://api_v1/;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }

    # API v2
    location /api/v2/ {
        proxy_pass http://api_v2/;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Static files (serve directly)
    location /static/ {
        alias /var/www/static/;
        expires 30d;
    }

    # Default: show a status page
    location = /status {
        return 200 '{"proxy":"nginx","status":"ok"}';
        add_header Content-Type application/json;
    }

    location / {
        return 404 '{"error":"not found"}';
        add_header Content-Type application/json;
    }
}
EOF

# Note: For nginx to resolve Docker container names, run nginx inside Docker too
# Or add Docker container IPs to /etc/hosts for testing

# Test
curl http://localhost/api/v1/users
curl http://localhost/api/v2/products
curl http://localhost/status
```

---

## Project 3 — Advanced: Load Balancer with Health Checks

**Goal:** Build a production-like load balancer setup with multiple nginx instances, health checks, and automatic failover. Observe failover behavior in real time.

**Skills:** upstream health checks, connection draining, observability, tcpdump

### docker-compose.yml

```yaml
version: '3.9'

services:
  lb:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./lb.conf:/etc/nginx/conf.d/default.conf
    networks:
      - webnet
    depends_on:
      - app1
      - app2
      - app3

  app1:
    image: nginx:alpine
    volumes:
      - ./app1/:/usr/share/nginx/html/
    networks:
      - webnet

  app2:
    image: nginx:alpine
    volumes:
      - ./app2/:/usr/share/nginx/html/
    networks:
      - webnet

  app3:
    image: nginx:alpine
    volumes:
      - ./app3/:/usr/share/nginx/html/
    networks:
      - webnet

networks:
  webnet:
```

```nginx
# lb.conf
upstream backend {
    least_conn;
    server app1:80 max_fails=2 fail_timeout=5s;
    server app2:80 max_fails=2 fail_timeout=5s;
    server app3:80 max_fails=2 fail_timeout=5s backup;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        add_header X-Upstream $upstream_addr;    # which backend served this
    }

    location /lb-status {
        return 200 "lb-ok";
        access_log off;
    }
}
```

```bash
# Setup app directories
for n in 1 2 3; do
    mkdir -p app$n
    echo "Response from app$n" > app$n/index.html
done

# Start stack
docker-compose up -d

# Test load balancing (see X-Upstream header rotating)
for i in $(seq 1 10); do
    curl -s -o /dev/null -D - http://localhost | grep "X-Upstream"
done

# Test failover: stop app1, observe traffic moves to app2/app3
docker-compose stop app1
for i in $(seq 1 10); do
    curl -s http://localhost/
done
# All responses come from app2/app3 only

# Restore app1
docker-compose start app1

# Cleanup
docker-compose down
```

---

## Project 4 — Capstone: Secure Multi-Service Architecture

**Goal:** Deploy a complete web stack with proper network segmentation, TLS, firewall rules, and monitoring.

### Architecture

```
Internet
    │
    ▼
[nginx: :443] ─── SSL termination, rate limiting
    │
    ├── /api/*  ──► [Python API: :8080] ─── internal only
    │                      │
    │                      ▼
    │               [PostgreSQL: :5432] ─── loopback only
    │
    └── /      ──► [Static files] ─── served from disk
```

### Implementation Plan

```bash
# 1. System setup
sudo apt update && sudo apt install -y nginx postgresql python3-pip

# 2. Database (loopback only)
sudo -u postgres psql -c "CREATE USER app PASSWORD 'strongpass';"
sudo -u postgres psql -c "CREATE DATABASE appdb OWNER app;"
# Verify PostgreSQL listens on localhost only:
ss -tlnp | grep 5432   # should show 127.0.0.1:5432

# 3. Firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 443/tcp   # HTTPS
sudo ufw deny 8080/tcp   # API not directly accessible
sudo ufw deny 5432/tcp   # DB not directly accessible
sudo ufw enable

# 4. TLS certificate
sudo apt install -y certbot python3-certbot-nginx
# For real domain: sudo certbot --nginx -d your-domain.com
# For testing: use self-signed (Project 1 method)

# 5. nginx configuration
sudo tee /etc/nginx/sites-available/capstone << 'EOF'
limit_req_zone $binary_remote_addr zone=api_rate:10m rate=20r/s;

server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name _;

    ssl_certificate     /etc/ssl/capstone/cert.crt;
    ssl_certificate_key /etc/ssl/capstone/private.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;

    # Rate limit API
    location /api/ {
        limit_req zone=api_rate burst=40 nodelay;
        limit_req_status 429;

        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_connect_timeout  5s;
        proxy_read_timeout    30s;
    }

    # Static files
    location / {
        root /var/www/capstone;
        try_files $uri $uri/ =404;
        expires 1d;
    }

    # Health check (no rate limit)
    location = /health {
        access_log off;
        proxy_pass http://127.0.0.1:8080/health;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/capstone /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# 6. Monitoring script
tee /usr/local/bin/check-stack.sh << 'SCRIPT'
#!/bin/bash
echo "=== Stack Health Check ==="

# nginx
curl -sk https://localhost/health | grep -q "ok" && \
  echo "✓ nginx + API: healthy" || echo "✗ nginx + API: FAILED"

# PostgreSQL
sudo -u postgres psql -c "SELECT 1" &>/dev/null && \
  echo "✓ PostgreSQL: healthy" || echo "✗ PostgreSQL: FAILED"

# Firewall
sudo ufw status | grep -q "Status: active" && \
  echo "✓ Firewall: active" || echo "✗ Firewall: INACTIVE"

# Open ports
echo "Open ports:"
ss -tlnp | awk 'NR>1{print "  "$4, $6}'

echo "==========================="
SCRIPT
chmod +x /usr/local/bin/check-stack.sh
```

### Extensions

- Add Prometheus node_exporter and scrape nginx metrics
- Implement log rotation for nginx access logs
- Set up automated certificate renewal monitoring
- Add fail2ban for SSH and HTTP brute-force protection
- Configure database connection pooling with pgbouncer

---

## Knowledge Check

1. In Project 2, why is routing by URL path a Layer 7 feature?
2. In Project 3, what does `max_fails=2 fail_timeout=5s` mean for health checks?
3. In Project 4, why is the API bound to `127.0.0.1` instead of `0.0.0.0`?
4. What does `limit_req burst=40 nodelay` mean in nginx rate limiting?
5. How would you monitor whether the rate limit is being hit?

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="13-common-mistakes.md">← Previous: Common Mistakes</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="15-interview-preparation.md">Next: Interview Preparation →</a>
</div>
