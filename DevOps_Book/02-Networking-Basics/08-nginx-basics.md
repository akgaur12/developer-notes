# Chapter 08 — Nginx Basics

## Learning Objectives

By the end of this chapter, you will:
- Install and understand nginx's configuration structure
- Configure virtual hosts (server blocks)
- Set up nginx as a reverse proxy
- Configure SSL/HTTPS termination
- Serve static files and configure caching
- Read nginx logs and debug issues

## Prerequisites

- Chapter 07 — Load Balancers & Reverse Proxies

---

## 8.1 What Is Nginx?

Nginx (pronounced "engine-x") is a high-performance web server and reverse proxy. It handles:
- **Static file serving** (HTML, CSS, JS, images)
- **Reverse proxy** to application servers
- **Load balancing**
- **SSL/TLS termination**
- **HTTP caching**

Nginx uses an **event-driven, non-blocking** architecture — one worker process handles thousands of connections efficiently.

```bash
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx

# Verify
curl -s http://localhost | grep -o "<title>.*</title>"
# <title>Welcome to nginx!</title>
```

---

## 8.2 Configuration Structure

```
/etc/nginx/
├── nginx.conf              ← main config (global settings)
├── sites-available/        ← available virtual host configs
│   ├── default             ← default site
│   └── myapp               ← your app config
├── sites-enabled/          ← symlinks to active configs
│   └── myapp → ../sites-available/myapp
├── conf.d/                 ← extra config files (auto-loaded)
├── snippets/               ← reusable config fragments
│   └── fastcgi-php.conf
└── mime.types              ← file extension → content-type mapping
```

```bash
# Enable a site (create symlink)
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/myapp

# Test config before applying
sudo nginx -t

# Reload config (zero downtime)
sudo nginx -s reload
# Or:
sudo systemctl reload nginx
```

---

## 8.3 Main Config — `nginx.conf`

```nginx
# /etc/nginx/nginx.conf

user www-data;
worker_processes auto;           # one worker per CPU core
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 1024;     # max connections per worker
    use epoll;                   # Linux kernel event mechanism
    multi_accept on;             # accept multiple connections at once
}

http {
    ##
    # Basic Settings
    ##
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ##
    # Logging
    ##
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    ##
    # Gzip Compression
    ##
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 1000;

    ##
    # Virtual Host Configs
    ##
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

---

## 8.4 Virtual Hosts (Server Blocks)

Each `server {}` block is a virtual host — one nginx handles many domains:

```nginx
# /etc/nginx/sites-available/myapp

server {
    listen 80;
    server_name myapp.example.com www.myapp.example.com;

    root /var/www/myapp;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

```bash
# Activate it
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 8.5 Nginx as a Reverse Proxy

```nginx
# /etc/nginx/sites-available/myapp

upstream app_backend {
    server 127.0.0.1:8080;    # your app running on port 8080
}

server {
    listen 80;
    server_name myapp.example.com;

    location / {
        proxy_pass http://app_backend;

        # Forward client info to backend
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout  5s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        # Buffer settings
        proxy_buffering on;
        proxy_buffer_size 8k;
        proxy_buffers 8 8k;
    }
}
```

### Why Proxy Headers Matter

```bash
# Without X-Forwarded-For: your app sees nginx's IP (127.0.0.1)
# With X-Forwarded-For: your app sees the real client IP

# In your app code:
# Python Flask: request.headers.get('X-Forwarded-For')
# Node Express: req.headers['x-forwarded-for']
# Java: request.getHeader("X-Forwarded-For")
```

---

## 8.6 HTTPS / SSL Termination

```nginx
server {
    listen 80;
    server_name myapp.example.com;

    # Redirect all HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.example.com;

    # Certificate files (from Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/myapp.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.example.com/privkey.pem;

    # Modern SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;
    add_header X-XSS-Protection "1; mode=block";

    location / {
        proxy_pass http://app_backend;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

```bash
# Get certificate with certbot
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d myapp.example.com

# Auto-renewal (already set up by certbot, verify with:)
sudo systemctl list-timers certbot*
```

---

## 8.7 Static File Serving

```nginx
server {
    listen 80;
    server_name static.example.com;

    root /var/www/static;

    # Cache static files aggressively
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    location / {
        try_files $uri $uri/ =404;
    }

    # Enable gzip for text files
    gzip on;
    gzip_types text/css application/javascript image/svg+xml;
}
```

---

## 8.8 URL Routing and Location Blocks

```nginx
server {
    listen 80;
    server_name api.example.com;

    # Exact match (fastest)
    location = /health {
        return 200 "OK\n";
        access_log off;
    }

    # Prefix match
    location /api/v1/ {
        proxy_pass http://api_v1_backend/;
    }

    location /api/v2/ {
        proxy_pass http://api_v2_backend/;
    }

    # Regex match (slower, use sparingly)
    location ~* \.(php|asp|aspx)$ {
        return 403;    # block PHP/ASP execution attempts
    }

    # Default
    location / {
        proxy_pass http://default_backend;
    }
}
```

### Location Priority

```
= exact match           highest priority
^~ prefix match         blocks regex
~  regex (case-sensitive)
~* regex (case-insensitive)
/  prefix match         lowest priority
```

---

## 8.9 Nginx Logs

```bash
# Access log — every request
tail -f /var/log/nginx/access.log
# 192.168.1.1 - - [23/Jun/2026:10:00:01 +0530] "GET / HTTP/2.0" 200 1234 "-" "curl/7.81.0"

# Error log — problems
tail -f /var/log/nginx/error.log

# Custom log format
log_format detailed '$remote_addr [$time_local] "$request" '
                    '$status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    '$request_time $upstream_response_time';
access_log /var/log/nginx/access.log detailed;

# Analyze access log
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn  # status codes
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head  # top IPs
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head  # top URLs
```

---

## 8.10 Rate Limiting

```nginx
http {
    # Define rate limit zone: 10 requests/second per IP
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    server {
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            limit_req_status 429;

            proxy_pass http://app_backend;
        }
    }
}
```

---

## Summary

```
nginx config hierarchy:
  nginx.conf → sites-enabled/*.conf → actual server blocks

Virtual host: server { listen 80; server_name ...; }
Reverse proxy: proxy_pass http://upstream;
SSL: listen 443 ssl; ssl_certificate; ssl_certificate_key;
Static files: root /path/; try_files $uri =404;
Test config: sudo nginx -t
Reload: sudo nginx -s reload
```

---

## Knowledge Check

1. What is the difference between `sites-available` and `sites-enabled`?
2. What does `sudo nginx -t` do?
3. Why do you need to forward `X-Real-IP` and `X-Forwarded-For` headers?
4. What does `try_files $uri $uri/ =404` do?
5. How do you reload nginx config without dropping connections?

---

## Hands-On Exercise

```bash
# 1. Install and start nginx
sudo apt install nginx -y
sudo systemctl start nginx
curl http://localhost    # see default page

# 2. Create a simple virtual host
sudo mkdir -p /var/www/mysite
echo "<h1>My Site Works!</h1>" | sudo tee /var/www/mysite/index.html

sudo tee /etc/nginx/sites-available/mysite << 'EOF'
server {
    listen 8081;
    server_name localhost;
    root /var/www/mysite;
    index index.html;
    location / {
        try_files $uri $uri/ =404;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo nginx -t && sudo nginx -s reload
curl http://localhost:8081    # should see "My Site Works!"

# 3. Configure as reverse proxy (proxy to any service)
# Start a simple Python HTTP server on port 8082
python3 -m http.server 8082 &
PY_PID=$!

sudo tee /etc/nginx/sites-available/proxy-demo << 'EOF'
server {
    listen 8083;
    location / {
        proxy_pass http://localhost:8082;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/proxy-demo /etc/nginx/sites-enabled/
sudo nginx -t && sudo nginx -s reload
curl http://localhost:8083    # proxied through nginx to python server

# Cleanup
kill $PY_PID
sudo rm /etc/nginx/sites-enabled/proxy-demo
sudo nginx -s reload

# 4. Check nginx logs
tail -5 /var/log/nginx/access.log
tail -5 /var/log/nginx/error.log
```

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="07-load-balancers-and-reverse-proxies.md">← Previous: Load Balancers & Reverse Proxies</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="09-networking-tools.md">Next: Networking Tools Deep Dive →</a>
</div>
