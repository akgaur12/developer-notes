# Chapter 13 — Common Networking Mistakes

## Learning Objectives

By the end of this chapter, you will:
- Recognize the most common networking pitfalls
- Debug connection issues systematically
- Avoid misconfigurations that cause production outages
- Understand "works on my machine" networking problems

## Prerequisites

- Chapter 12 — Networking Best Practices

---

## 13.1 Forgetting to Open Firewall Ports

**Symptom:** App works locally, but clients can't connect in production.

```bash
# Debugging workflow
# Step 1: verify the service is actually running
ss -tlnp | grep :8080    # is port 8080 listening?

# Step 2: verify it's listening on the right interface
ss -tlnp | grep :8080
# "127.0.0.1:8080" ← only localhost, NOT accessible from outside!
# "0.0.0.0:8080"   ← accessible from any interface ✓

# Step 3: check firewall
sudo ufw status verbose
sudo iptables -L INPUT -n -v

# Step 4: fix
sudo ufw allow 8080/tcp

# Common: app binds to 127.0.0.1 by default, needs to bind to 0.0.0.0
# Check your app config:
#   Python Flask: app.run(host='0.0.0.0', port=8080)
#   Node Express: app.listen(8080, '0.0.0.0')
#   Go: http.ListenAndServe("0.0.0.0:8080", nil)
```

---

## 13.2 Misunderstanding CIDR Ranges

**Mistake:** Assuming `/24` means 256 hosts.

```
10.0.0.0/24  — actually 254 usable hosts (256 - network addr - broadcast addr)
10.0.0.0     — network address (reserved, can't assign)
10.0.0.255   — broadcast address (reserved, can't assign)

/28 = 16 addresses = 14 usable hosts
/29 = 8 addresses = 6 usable hosts
/30 = 4 addresses = 2 usable hosts (point-to-point links)
/31 = 2 addresses = 2 usable hosts (RFC 3021 — no network/broadcast)
/32 = 1 address = single host (security group rules, routes)
```

```bash
# Calculate subnet info
ipcalc 10.0.0.0/24
ipcalc 192.168.1.0/28

# Or use Python
python3 -c "import ipaddress; n=ipaddress.ip_network('10.0.0.0/24'); print(f'Hosts: {n.num_addresses-2}')"
```

**Real-world mistake:** Creating a /28 subnet in AWS for an EKS node group that needs 30 nodes. The subnet runs out of IPs.

---

## 13.3 DNS Caching Issues

**Symptom:** After changing a DNS record, some clients still get the old IP.

```bash
# Problem: TTL controls how long records are cached
# If your A record has TTL=3600 (1 hour), changes take 1 hour to propagate

# Before migration: reduce TTL to 60 seconds
# Wait for old TTL to expire
# Make the change
# After migration is stable: increase TTL back to 3600

# Check current TTL
dig +noall +answer example.com
# example.com.   3600  IN  A  1.2.3.4
#                ^^^^  ← TTL in seconds

# Flush your local DNS cache
sudo systemd-resolve --flush-caches           # systemd-resolved
sudo service nscd restart                     # nscd
# macOS: sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder

# Check what a specific DNS server returns
dig @8.8.8.8 example.com        # Google
dig @1.1.1.1 example.com        # Cloudflare
dig @ns1.yourdns.com example.com  # authoritative
```

**Mistake:** Changing DNS without lowering TTL first, then wondering why rollback doesn't work immediately.

---

## 13.4 TLS Certificate Pitfalls

### Expired Certificates

```bash
# Check expiry
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -enddate

# Quick shell function for monitoring
check_cert() {
    domain=$1
    days=$(echo | openssl s_client -connect "$domain:443" 2>/dev/null \
           | openssl x509 -noout -enddate 2>/dev/null \
           | sed 's/notAfter=//' \
           | xargs -I{} sh -c 'echo $(( ($(date -d "{}" +%s) - $(date +%s)) / 86400 ))')
    echo "$domain: $days days remaining"
}
check_cert github.com
```

### Common TLS Errors

```bash
# SSL certificate problem: unable to get local issuer certificate
# Cause: missing intermediate cert in chain
# Fix: use fullchain.pem (contains cert + intermediates), not just cert.pem
nginx: ssl_certificate /etc/letsencrypt/live/domain/fullchain.pem   # ✓ correct
nginx: ssl_certificate /etc/letsencrypt/live/domain/cert.pem        # ✗ missing chain

# SSL: CERTIFICATE_VERIFY_FAILED
# Cause 1: self-signed cert, trust explicitly
curl --cacert /path/to/ca.crt https://internal.example.com

# Certificate name mismatch
# Cert is for example.com, accessing as www.example.com
# Fix: use SAN (Subject Alternative Names) cert covering both
openssl x509 -noout -ext subjectAltName -in cert.pem
```

---

## 13.5 Timeouts and Connection Resets

**Symptom:** Large file uploads or long-running requests fail with timeout errors.

```nginx
# nginx proxy timeouts (defaults are often too short for file uploads or slow backends)
proxy_connect_timeout  5s;    # time to establish connection to backend
proxy_send_timeout    60s;    # time between writing successive data to backend
proxy_read_timeout    60s;    # time between successive reads from backend
                               # NOT the total request time

# For large file uploads:
client_max_body_size 100m;    # default is 1m — causes 413 for large files
client_body_timeout 120s;     # time waiting for client to send body
```

```bash
# Debug: is it client timeout or backend timeout?
# 502 Bad Gateway = backend down or refused connection
# 504 Gateway Timeout = backend responded too slowly
# 499 Client Closed Request = client disconnected (their timeout hit first)

# Check nginx error log
grep -E "504|502|connect.*failed|upstream" /var/log/nginx/error.log | tail -20

# Test backend directly (bypass nginx)
curl -v --max-time 120 http://backend:8080/slow-endpoint
```

---

## 13.6 Too Many Open Connections / TIME_WAIT

**Symptom:** "Address already in use" errors, new connections failing.

```bash
# Symptoms
ss -s   # see many TIME_WAIT connections
# TIME_WAIT: 50000

# Why TIME_WAIT happens:
# After connection close, port stays in TIME_WAIT for 2*MSL (~60-120s)
# Protects against late packets from old connections arriving on new connection

# Monitor
watch 'ss -tan | awk "NR>1{print \$1}" | sort | uniq -c | sort -rn'

# Fix 1: enable TCP TIME_WAIT reuse (for client-side connections)
echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse

# Fix 2: persistent connections (don't create new connections for every request)
# nginx: keepalive 32; in upstream block
# Your app: use connection pooling (pgbouncer for postgres, etc.)

# Fix 3: widen ephemeral port range
echo "1024 65535" > /proc/sys/net/ipv4/ip_local_port_range
```

---

## 13.7 Forgetting to Handle Proxied IPs

**Symptom:** All requests show nginx's IP (`127.0.0.1`) instead of real client IP.

```nginx
# ✗ Missing headers
location / {
    proxy_pass http://backend;
    # backend sees remote_addr as 127.0.0.1
}

# ✓ Forward client IP
location / {
    proxy_pass http://backend;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

```python
# Backend: read real IP from header (Python Flask example)
from flask import request

def get_client_ip():
    # Trust X-Forwarded-For only if behind trusted proxy
    forwarded_for = request.headers.get('X-Forwarded-For')
    if forwarded_for:
        return forwarded_for.split(',')[0].strip()
    return request.remote_addr
```

**Warning:** Never trust `X-Forwarded-For` unconditionally — clients can spoof it. Only trust it when the request actually comes from your trusted proxy.

---

## 13.8 Split-Brain DNS

**Symptom:** External users can reach `api.example.com`, but internal services calling `api.example.com` fail or get routed externally.

```
Problem:
  External DNS: api.example.com → 52.1.2.3 (public IP)
  Internal services call api.example.com → hair-pin NAT → public IP → back inside
  (this often fails, adds latency, and costs money)

Solution: Split-horizon DNS
  External DNS: api.example.com → 52.1.2.3 (public IP)
  Internal DNS: api.example.com → 10.0.1.5 (private IP, direct)
```

```bash
# In Docker Compose or Kubernetes, use internal service names:
# http://api-service:8080  (not http://api.example.com)

# In /etc/hosts on servers (manual internal DNS):
echo "10.0.1.5 api.example.com" >> /etc/hosts

# In dnsmasq (internal DNS server):
address=/api.example.com/10.0.1.5
```

---

## 13.9 MTU Mismatches

**Symptom:** Large packets fail, small packets work fine. SSH works but `git clone` hangs. Pings work but HTTPS fails.

```
Standard MTU:   1500 bytes (Ethernet)
VPN/Tunnel MTU: ~1420 bytes (VPN adds header overhead)
If you try to send a 1500-byte packet through a 1420-byte VPN: fragmentation or drop
```

```bash
# Test with ping (use -s to set payload size, -M do to set "Don't Fragment" flag)
ping -M do -s 1472 8.8.8.8   # 1472 + 28 header = 1500 total
ping -M do -s 1400 8.8.8.8   # smaller — try to find the MTU

# If 1472 fails but 1400 works: MTU mismatch
# Find the exact MTU
for size in 1450 1400 1350 1300; do
    if ping -M do -s $size -c 1 -W 1 8.8.8.8 &>/dev/null; then
        echo "MTU OK at $((size + 28))"
        break
    fi
done

# Fix: clamp MSS to path MTU in iptables (for VPN/tunnel)
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

---

## Quick Diagnostic Checklist

When "it doesn't work over the network":

```
□ Is the service running?          ss -tlnp | grep :<PORT>
□ Is it listening on 0.0.0.0?     (not 127.0.0.1)
□ Does firewall allow it?          sudo ufw status
□ Does DNS resolve?                dig example.com
□ Does TCP connect?                nc -zv host port
□ Is TLS cert valid?               curl -v https://host
□ Are proxy headers set?           check nginx config
□ Are timeouts appropriate?        check nginx/app timeout config
□ Is it a MTU issue?               ping -M do -s 1472 host
```

---

## Knowledge Check

1. An app works locally but not from outside. What's the first thing to check?
2. Why might DNS changes not take effect immediately even after you update the record?
3. What causes a flood of TIME_WAIT connections and how do you reduce them?
4. What's the difference between a 502 and 504 from nginx?
5. Why is it dangerous to trust `X-Forwarded-For` blindly?

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="12-best-practices.md">← Previous: Best Practices</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="14-projects.md">Next: Hands-On Projects →</a>
</div>
