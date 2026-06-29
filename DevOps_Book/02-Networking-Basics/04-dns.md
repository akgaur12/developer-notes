# Chapter 04 — DNS: Domain Name System

## Learning Objectives

By the end of this chapter, you will:
- Understand the complete DNS resolution process
- Know all important DNS record types and their uses
- Use `dig` and `nslookup` to query and debug DNS
- Understand TTL and caching in DNS
- Configure `/etc/hosts` and `/etc/resolv.conf`
- Diagnose and fix DNS failures in production

## Prerequisites

- Chapter 03 — TCP, UDP & Ports

---

## 4.1 What Is DNS and Why Does It Exist?

Computers communicate via IP addresses (`142.250.80.46`). Humans communicate via names (`google.com`). DNS is the translation layer.

```
You type:     https://github.com
Browser asks: "What is the IP for github.com?"
DNS answers:  "140.82.121.4"
Browser uses: https://140.82.121.4
```

Without DNS, you'd need to memorize IP addresses for every website. DNS is often called the "phone book of the internet."

---

## 4.2 DNS Hierarchy

DNS is a **distributed**, **hierarchical** database. No single server knows everything:

```
.                           ← Root zone (13 root server clusters)
├── com.                    ← Top-Level Domain (TLD) — .com TLD servers
│   ├── github.com.         ← Second-Level Domain — GitHub's nameservers
│   │   └── api.github.com. ← Subdomain — also GitHub's nameservers
│   └── google.com.
├── org.
├── net.
├── io.
└── in.                     ← Country TLD
    └── kloudspot.com.in.
```

---

## 4.3 The DNS Resolution Process (Step by Step)

When you visit `github.com` for the first time:

```
Browser                  OS          Resolver      Root NS    .com NS   GitHub NS
   │                      │              │             │          │          │
   │── "github.com IP?" ─►│              │             │          │          │
   │                      │─ check /etc/hosts ─►(miss)│          │          │
   │                      │─ check local cache ─►(miss)│         │          │
   │                      │── "github.com?" ──────────►│          │          │
   │                      │                             │─ "Ask   │          │
   │                      │                             │  .com   │          │
   │                      │                             │  NS" ──►│          │
   │                      │                             │         │─ "Ask    │
   │                      │                             │         │  GitHub  │
   │                      │                             │         │  NS" ────►│
   │                      │                             │         │          │─ "140.82.121.4"
   │                      │◄─────── "140.82.121.4" ─────────────────────────│
   │◄──── "140.82.121.4" ─│              │             │          │          │
```

### The Four Players

1. **DNS Resolver (Recursive Resolver)**: Your local DNS server (ISP-provided, or 8.8.8.8, 1.1.1.1). It does the heavy lifting — asks root, TLD, and authoritative servers on your behalf.

2. **Root Name Servers**: 13 clusters (a-m.root-servers.net) worldwide. Know where TLD servers are. Don't know individual domains.

3. **TLD Name Servers**: Know where authoritative servers for each domain in the TLD are.

4. **Authoritative Name Server**: The server that actually knows the IP for `github.com`. Operated by GitHub (or their DNS provider like Cloudflare, Route 53).

```bash
# Trace the full resolution
dig +trace github.com
# Shows each step: root → TLD → authoritative
```

---

## 4.4 DNS Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| `A` | IPv4 address | `github.com. → 140.82.121.4` |
| `AAAA` | IPv6 address | `github.com. → 2606:50c0::` |
| `CNAME` | Alias to another name | `www.example.com → example.com` |
| `MX` | Mail server | `example.com → mail.example.com (priority 10)` |
| `TXT` | Text data | SPF, DKIM, domain verification |
| `NS` | Authoritative nameservers | `github.com → ns1.p16.dynect.net` |
| `SOA` | Start of Authority | Zone info, serial, refresh TTL |
| `PTR` | Reverse DNS (IP → name) | `4.121.82.140.in-addr.arpa → github.com` |
| `SRV` | Service location | `_https._tcp.example.com → host:443` |
| `CAA` | Certificate Authority Authorization | Which CAs can issue certs |

### Practical Record Usage in DevOps

```bash
# A record — get IP
dig github.com A +short
dig github.com +short   # default is A

# AAAA — get IPv6
dig github.com AAAA +short

# CNAME — follow aliases
dig www.github.com CNAME +short
# Shows: github.com.  (www is just a CNAME to the apex)

# MX — find mail servers (useful for email config)
dig gmail.com MX +short
# 5 gmail-smtp-in.l.google.com.
# 10 alt1.gmail-smtp-in.l.google.com.

# TXT — read text records (SPF, DKIM, domain verification)
dig github.com TXT +short
dig _dmarc.github.com TXT +short

# NS — find authoritative nameservers
dig github.com NS +short

# Reverse DNS (PTR)
dig -x 8.8.8.8 +short
# dns.google.

# SOA — zone serial and TTL info
dig github.com SOA
```

---

## 4.5 TTL — Time to Live

Every DNS record has a **TTL (Time to Live)** — how long caches should keep the answer (in seconds).

```
github.com.    299    IN    A    140.82.121.4
                │
                └── TTL: 299 seconds (~5 minutes)
```

- **Short TTL (60–300s)**: Changes propagate fast. Higher DNS load. Use during migrations or when IP changes frequently.
- **Long TTL (3600–86400s)**: Propagates slowly. Lower DNS load. Use for stable records.

```bash
# See TTL in dig output
dig github.com A
# github.com.  60  IN  A  140.82.121.4
#               └── 60 seconds TTL

# Check how much TTL is left (caching resolver)
dig github.com @8.8.8.8   # query Google's resolver
# Keep querying — TTL decrements each time
```

### TTL Strategy for DevOps

```bash
# BEFORE a server migration:
# 1. Lower TTL to 60 seconds (wait for old TTL to expire)
# 2. Make the IP change
# 3. Verify propagation
# 4. Raise TTL back to normal

# Check propagation globally
# Use: https://dnschecker.org/
```

---

## 4.6 `dig` — The DNS Debugging Tool

```bash
# Basic lookup
dig example.com
dig example.com A
dig example.com AAAA

# Short output (just the answer)
dig +short example.com
dig +short example.com MX

# Specific DNS server
dig @8.8.8.8 example.com     # query Google DNS
dig @1.1.1.1 example.com     # query Cloudflare DNS
dig @9.9.9.9 example.com     # query Quad9 DNS

# Trace full resolution
dig +trace example.com

# No recursion (ask authoritative directly)
dig +norec example.com @ns1.example.com

# Multiple records at once
dig example.com ANY

# Reverse lookup
dig -x 8.8.8.8

# Full answer with all sections
dig example.com +all
```

### Reading dig Output

```
; <<>> DiG 9.18.0 <<>> github.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;github.com.                    IN      A

;; ANSWER SECTION:
github.com.             60      IN      A       140.82.121.4
│          │            │       │       │       │
│          │            │       │       │       └─ value (IP)
│          │            │       │       └─ record type
│          │            │       └─ class (IN = internet)
│          │            └─ TTL (60 seconds)
│          └─ trailing dot (FQDN)
└─ queried name

;; Query time: 12 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)   ← which resolver answered
;; WHEN: Mon Jun 23 10:00:00 2026
;; MSG SIZE  rcvd: 55
```

**Status codes:**
- `NOERROR` — success
- `NXDOMAIN` — name doesn't exist
- `SERVFAIL` — resolver couldn't get an answer
- `REFUSED` — server refuses to answer

---

## 4.7 `/etc/hosts` — Local DNS Override

`/etc/hosts` is checked **before** DNS. Use it to override or create local name-to-IP mappings:

```bash
cat /etc/hosts
# 127.0.0.1   localhost
# 127.0.1.1   myhostname
# ::1         localhost ip6-localhost

# Add custom entries
sudo vim /etc/hosts
# Add: 10.0.0.5  db.internal
# Add: 10.0.0.6  redis.internal
# Add: 127.0.0.1 myapp.local

# Test
ping db.internal    # uses /etc/hosts, no DNS query
```

### `/etc/hosts` DevOps Uses

```bash
# Development: point production domains to local/staging
echo "127.0.0.1  api.myapp.com" | sudo tee -a /etc/hosts

# Testing: bypass DNS during migration
echo "10.0.0.100  api.myapp.com" | sudo tee -a /etc/hosts

# Block sites (security)
echo "0.0.0.0  malicious-site.com" | sudo tee -a /etc/hosts
```

---

## 4.8 `/etc/resolv.conf` — DNS Resolver Configuration

```bash
cat /etc/resolv.conf

# nameserver 127.0.0.53       ← systemd-resolved (Ubuntu default)
# search internal.mycompany.com  ← auto-append to short names
# options ndots:5             ← threshold for trying search domains
```

### Resolution Order: `/etc/nsswitch.conf`

```bash
cat /etc/nsswitch.conf | grep hosts
# hosts: files dns myhostname
#         │     │
#         │     └── then DNS
#         └──────── first: /etc/hosts
```

---

## 4.9 DNS in Docker and Kubernetes

### Docker DNS

```bash
# Each container gets DNS from Docker's embedded resolver (127.0.0.11)
docker run --rm busybox cat /etc/resolv.conf
# nameserver 127.0.0.11
# options ndots:0

# Containers can reach each other by name (same network)
docker network create mynet
docker run -d --name db --network mynet postgres
docker run --rm --network mynet busybox ping db   # works!
```

### Kubernetes DNS (CoreDNS)

```bash
# Every pod gets DNS automatically
kubectl exec -it mypod -- cat /etc/resolv.conf
# nameserver 10.96.0.10        ← CoreDNS ClusterIP
# search default.svc.cluster.local svc.cluster.local cluster.local

# Service DNS patterns:
# <service>                        → same namespace
# <service>.<namespace>            → across namespaces
# <service>.<namespace>.svc.cluster.local  → FQDN

# Pods can reach services by name:
curl http://my-service              # same namespace
curl http://my-service.staging      # staging namespace
```

---

## 4.10 DNS Troubleshooting Playbook

```bash
# Step 1: Can you resolve anything?
dig google.com +short
# If no: DNS is completely broken → check /etc/resolv.conf

# Step 2: Is it the specific domain?
dig @8.8.8.8 yourapp.com +short
# If works with 8.8.8.8 but not default: your resolver is broken

# Step 3: Does the record exist?
dig yourapp.com A +short
# If NXDOMAIN: record doesn't exist → check DNS provider

# Step 4: Is TTL cached stale value?
dig yourapp.com @ns1.yourprovider.com +short   # query authoritative
# Compare with what resolver returns

# Step 5: Check /etc/hosts override
grep "yourapp.com" /etc/hosts

# Step 6: Check search domains
cat /etc/resolv.conf
# If short name fails: try FQDN (with trailing dot)
dig yourapp.com.   # FQDN
```

---

## Summary

```
DNS Resolution: browser → resolver → root → TLD → authoritative
Record Types:
  A     = IPv4 address
  AAAA  = IPv6 address
  CNAME = alias
  MX    = mail server
  TXT   = text (SPF, DKIM, verification)
  NS    = nameservers

Tools:
  dig domain        = query DNS
  dig +trace domain = trace full resolution
  dig @8.8.8.8      = use specific resolver
  /etc/hosts        = local override (checked first)
  /etc/resolv.conf  = resolver config
```

---

## Knowledge Check

1. What are the four players in DNS resolution?
2. What record type do you look up to find a domain's mail servers?
3. Why would you set a low TTL before a server migration?
4. What does `NXDOMAIN` mean in a DNS response?
5. What file is checked before DNS queries are sent?

---

## Hands-On Exercise

```bash
# 1. Full DNS lookup chain
dig +trace github.com 2>/dev/null | head -30

# 2. Query all record types
for type in A AAAA MX TXT NS; do
    echo "=== $type ==="
    dig github.com $type +short
done

# 3. Compare local resolver vs authoritative
echo "Via local resolver:"
dig github.com A +short
echo "Via Google DNS:"
dig @8.8.8.8 github.com A +short

# 4. Reverse DNS lookup
dig -x 8.8.8.8 +short
dig -x 1.1.1.1 +short

# 5. Check DNS resolution time
time dig github.com > /dev/null    # first query (may hit upstream)
time dig github.com > /dev/null    # second query (cached, should be faster)

# 6. Test /etc/hosts override
echo "127.0.0.1 test.local.example" | sudo tee -a /etc/hosts
ping -c 1 test.local.example       # should resolve to 127.0.0.1
# Cleanup:
sudo sed -i '/test.local.example/d' /etc/hosts
```

**Challenge:** Use `dig` to find the TTL on `google.com`'s A record. Then wait 30 seconds and query again — does the TTL decrease? (Yes, it should, showing the cache decrementing.)

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="03-tcp-udp-and-ports.md">← Previous: TCP, UDP & Ports</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="05-http-and-https.md">Next: HTTP & HTTPS →</a>
</div>
