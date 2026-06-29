# Chapter 12 — Networking Best Practices

## Learning Objectives

By the end of this chapter, you will:
- Apply defense-in-depth networking security
- Monitor network health with proper metrics
- Tune Linux networking performance
- Design resilient multi-zone architectures
- Use network policies in container environments
- Document and audit network topology

## Prerequisites

- Chapter 11 — Advanced Networking Concepts

---

## 12.1 Defense in Depth

Never rely on a single security control. Layer your defenses:

```
                    ┌─────────────────────────────────────────┐
                    │  Layer 1: Cloud Security Groups/NACLs   │
                    │  (block at cloud provider level)        │
                    │  ┌───────────────────────────────────┐  │
                    │  │  Layer 2: WAF / DDoS Protection   │  │
                    │  │  (Cloudflare, AWS WAF)             │  │
                    │  │  ┌─────────────────────────────┐  │  │
                    │  │  │  Layer 3: Host Firewall     │  │  │
                    │  │  │  (UFW / iptables)           │  │  │
                    │  │  │  ┌───────────────────────┐  │  │  │
                    │  │  │  │  Layer 4: App Auth    │  │  │  │
                    │  │  │  │  (TLS, JWT, API keys) │  │  │  │
                    │  │  │  └───────────────────────┘  │  │  │
                    │  │  └─────────────────────────────┘  │  │
                    │  └───────────────────────────────────┘  │
                    └─────────────────────────────────────────┘
```

### Principle of Least Privilege for Networks

```
✗ Wrong:
  Database port 5432 open to 0.0.0.0/0

✓ Right:
  Database port 5432 open only to application server subnet (10.0.1.0/24)
  Application port 8080 open only from load balancer security group
  Load balancer port 443 open to 0.0.0.0/0 (internet-facing)
  SSH port 22 open only from bastion/VPN IP
```

```bash
# UFW: restrict database to app subnet only
sudo ufw default deny incoming
sudo ufw allow from 10.0.1.0/24 to any port 5432    # PostgreSQL from app tier
sudo ufw allow from 10.0.2.0/24 to any port 22       # SSH from bastion subnet
sudo ufw deny 5432                                    # block all other DB access
sudo ufw enable
```

---

## 12.2 HTTPS Everywhere

```nginx
# Redirect all HTTP → HTTPS
server {
    listen 80 default_server;
    return 301 https://$host$request_uri;
}

# Strong TLS config
server {
    listen 443 ssl http2 default_server;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;    # client picks best cipher it supports
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384';

    # HSTS — tell browsers to always use HTTPS (max 2 years)
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # Certificate stapling (performance optimization)
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
}
```

```bash
# Test SSL configuration quality
# https://www.ssllabs.com/ssltest/ (external)

# Local test with nmap
nmap --script ssl-enum-ciphers -p 443 your-server.com

# Check for weak ciphers
openssl s_client -connect your-server.com:443 -cipher NULL 2>&1 | grep -i "no cipher"
```

### Certificate Management

```bash
# Monitor certificate expiry (run in cron)
#!/bin/bash
DOMAINS=(example.com api.example.com admin.example.com)
WARN_DAYS=30

for domain in "${DOMAINS[@]}"; do
    expiry=$(echo | openssl s_client -connect "$domain:443" -servername "$domain" 2>/dev/null \
             | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)
    expiry_epoch=$(date -d "$expiry" +%s 2>/dev/null)
    now_epoch=$(date +%s)
    days_left=$(( (expiry_epoch - now_epoch) / 86400 ))

    if [[ $days_left -lt $WARN_DAYS ]]; then
        echo "WARNING: $domain expires in $days_left days ($expiry)"
    else
        echo "OK: $domain expires in $days_left days"
    fi
done
```

---

## 12.3 Network Monitoring and Observability

### Key Metrics to Track

| Metric | Warning Threshold | Tool |
|--------|-----------------|------|
| Packet loss | >0.1% | ping, mtr |
| Latency (p99) | >100ms internal | Application metrics |
| Connection errors | >0 per minute | nginx logs, app logs |
| Open connections | Approaching `ulimit -n` | ss -s |
| DNS resolution time | >50ms | Application metrics |
| TCP retransmits | >0.1% of packets | `ss -s`, `/proc/net/snmp` |

```bash
# ─── Monitor connections ───────────────────────────────────────────────
# Count connections by state (run every 30s)
watch -n 30 'ss -tan | awk "NR>1{print \$1}" | sort | uniq -c | sort -rn'

# ─── Check TCP stats ───────────────────────────────────────────────────
cat /proc/net/snmp | grep -E "^Tcp:" | awk 'NR==1{for(i=1;i<=NF;i++) key[i]=$i} NR==2{for(i=1;i<=NF;i++) print key[i]": "$i}'

# ─── Retransmit rate ───────────────────────────────────────────────────
# /proc/net/snmp: RetransSegs / OutSegs = retransmit rate
awk '/^Tcp:/{if(NR==9){for(i=1;i<=NF;i++)k[i]=$i}else{for(i=1;i<=NF;i++)print k[i]": "$i}}' /proc/net/snmp | grep -E "RetransSegs|OutSegs"

# ─── Network interface stats ───────────────────────────────────────────
cat /proc/net/dev | awk 'NR>2{print $1, "rx_bytes:", $2, "tx_bytes:", $10}'

# ─── Watch interface errors ────────────────────────────────────────────
ip -s link show eth0
```

### Log-Based Monitoring

```bash
# Nginx access log → extract metrics
# Error rate (5xx responses as % of total)
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | \
  awk 'BEGIN{total=0;errors=0} {total+=$1; if($2~/^5/)errors+=$1} END{printf "Error rate: %.2f%%\n", errors/total*100}'

# Top 10 slowest URLs
awk '{print $NF, $7}' /var/log/nginx/access.log | sort -rn | head -10

# Alert on high error rate (cron job)
ERROR_RATE=$(awk '{print $9}' /var/log/nginx/access.log | \
  awk 'BEGIN{t=0;e=0} {t++; if(/^5/)e++} END{printf "%.0f", e/t*100}')
[[ $ERROR_RATE -gt 5 ]] && echo "ALERT: ${ERROR_RATE}% error rate"
```

---

## 12.4 Performance Tuning

### Linux Kernel TCP Tuning

```bash
# /etc/sysctl.d/99-network-tuning.conf

# ─── Socket buffer sizes ───────────────────────────────────────────────
net.core.rmem_default = 1048576       # 1MB default receive buffer
net.core.rmem_max = 16777216          # 16MB max receive buffer
net.core.wmem_default = 1048576       # 1MB default send buffer
net.core.wmem_max = 16777216          # 16MB max send buffer

# ─── TCP buffer sizes ──────────────────────────────────────────────────
net.ipv4.tcp_rmem = 4096 1048576 16777216
net.ipv4.tcp_wmem = 4096 1048576 16777216

# ─── Connection queue ──────────────────────────────────────────────────
net.core.somaxconn = 65535           # max listen backlog (default: 128)
net.ipv4.tcp_max_syn_backlog = 65535

# ─── TIME_WAIT optimization ────────────────────────────────────────────
net.ipv4.tcp_tw_reuse = 1            # reuse TIME_WAIT sockets
net.ipv4.tcp_fin_timeout = 15        # reduce TIME_WAIT timeout

# ─── Keep-alive ────────────────────────────────────────────────────────
net.ipv4.tcp_keepalive_time = 60     # seconds before keep-alive probe
net.ipv4.tcp_keepalive_intvl = 10    # interval between probes
net.ipv4.tcp_keepalive_probes = 5    # number of probes before declaring dead

# ─── Port range ────────────────────────────────────────────────────────
net.ipv4.ip_local_port_range = 1024 65535

# Apply
sudo sysctl -p /etc/sysctl.d/99-network-tuning.conf
```

### File Descriptor Limits

Each connection uses a file descriptor:

```bash
# Check current limit
ulimit -n                              # per-process limit (typically 1024)
cat /proc/sys/fs/file-max             # system-wide limit

# Increase for production (add to /etc/security/limits.conf)
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf

# Or system-wide
echo "fs.file-max = 1048576" >> /etc/sysctl.conf
sysctl -p

# For nginx (in nginx.conf)
worker_rlimit_nofile 65535;
events {
    worker_connections 65535;
}
```

---

## 12.5 Network Architecture Best Practices

### Multi-AZ Design

```
VPC: 10.0.0.0/16
├── AZ-a
│   ├── Public subnet:  10.0.0.0/24  (load balancer, NAT gateway)
│   └── Private subnet: 10.0.10.0/24 (app servers, databases)
├── AZ-b
│   ├── Public subnet:  10.0.1.0/24
│   └── Private subnet: 10.0.11.0/24
└── AZ-c
    ├── Public subnet:  10.0.2.0/24
    └── Private subnet: 10.0.12.0/24
```

Rules:
- **Databases**: private subnets only, never public
- **App servers**: private subnets, only reachable from load balancer
- **Load balancers**: public subnets, only ports 80/443 open
- **Bastion/jump host**: one per VPC, port 22 from specific IPs only
- **NAT Gateway**: one per AZ (avoid cross-AZ NAT traffic charges)

---

## 12.6 Container Network Policies

In Kubernetes, use **NetworkPolicy** to restrict pod-to-pod communication:

```yaml
# Only allow the web tier to talk to the api tier
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-web-only
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: web
      ports:
        - protocol: TCP
          port: 8080
```

```yaml
# Default deny all in a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}    # matches all pods
  policyTypes:
    - Ingress
    - Egress
```

---

## Summary

| Practice | Why |
|----------|-----|
| Least privilege firewall rules | Limits blast radius if one service is compromised |
| HTTPS everywhere with strong TLS | Encrypts all traffic, prevents MITM |
| Monitor packet loss + latency | Catches network degradation before users notice |
| Tune TCP buffers for high traffic | Prevents buffer exhaustion under load |
| Multi-AZ architecture | Single AZ failure doesn't cause outage |
| Private subnets for databases | Databases never directly internet-reachable |
| Kubernetes NetworkPolicy | Zero-trust networking between pods |

---

## Knowledge Check

1. What is "defense in depth" in a networking context?
2. Why should databases be in private subnets?
3. What does `net.core.somaxconn` control and why does it matter for web servers?
4. What does `ulimit -n` control and what happens if you hit the limit?
5. What does a Kubernetes NetworkPolicy with empty `podSelector: {}` do?

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="11-advanced-concepts.md">← Previous: Advanced Concepts</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="13-common-mistakes.md">Next: Common Mistakes →</a>
</div>
