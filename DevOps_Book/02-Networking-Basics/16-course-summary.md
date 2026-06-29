# Chapter 16 — Course Summary

## You've Completed Networking Basics!

This chapter reviews everything covered, gives you a completion checklist, and points you toward the next topic in the DevOps roadmap.

---

## What You Learned

### Foundations (Chapters 01–03)
- **OSI Model**: 7 layers — Physical, Data Link, Network, Transport, Session, Presentation, Application
- **TCP/IP Model**: 4 layers — Network Access, Internet, Transport, Application
- **IP Addressing**: Binary representation, CIDR notation, private ranges (RFC 1918)
- **Subnetting**: Calculate usable hosts, design subnet layouts for cloud VPCs
- **TCP**: 3-way handshake (SYN/SYN-ACK/ACK), 4-way teardown, TCP states
- **UDP**: Connectionless, use cases (DNS, DHCP, VoIP, streaming)
- **Ports**: Well-known ports (22, 80, 443, 5432, 6379), `ss -tlnp` to inspect

### Core Services (Chapters 04–06)
- **DNS**: Resolution hierarchy (resolver → root → TLD → authoritative), record types (A, AAAA, CNAME, MX, TXT), TTL, `dig` for debugging
- **HTTP/HTTPS**: Request/response structure, methods (GET/POST/PUT/PATCH/DELETE), status codes (2xx success, 3xx redirect, 4xx client error, 5xx server error), headers
- **TLS**: Handshake sequence, certificates, Let's Encrypt, `openssl s_client` for inspection
- **Firewalls**: UFW (default deny, explicit allow), iptables chains and tables, cloud security groups (stateful vs stateless)

### Infrastructure (Chapters 07–09)
- **Load Balancers**: Round robin, weighted, least_conn, ip_hash algorithms; health checks; Layer 4 vs Layer 7
- **Reverse Proxies**: SSL termination, proxy headers (X-Real-IP, X-Forwarded-For), connection draining
- **Nginx**: Config structure (sites-available/sites-enabled), virtual hosts, reverse proxy config, location block priority, rate limiting, log analysis
- **Networking Tools**: `curl` (timing, API testing), `ping`, `traceroute`, `mtr`, `tcpdump`, `nmap`, `ss`

### Advanced (Chapters 10–11)
- **NAT**: SNAT, DNAT, masquerade; iptables NAT rules
- **VPN**: WireGuard configuration, site-to-site vs remote access
- **CDN**: Cache hit/miss, Cache-Control headers, CloudFront invalidation
- **Docker Networking**: Bridge, host, overlay drivers; user-defined networks; DNS resolution between containers
- **HTTP/2**: Multiplexing, header compression (HPACK), server push
- **HTTP/3 / QUIC**: UDP-based, 0-RTT, no TCP head-of-line blocking
- **WebSockets**: HTTP upgrade, nginx WebSocket proxy config
- **gRPC**: Protobuf serialization, HTTP/2 transport, service definitions
- **mTLS**: Client + server certificates; service-to-service auth
- **BGP**: Autonomous systems, anycast, cloud direct connect

---

## Completion Checklist

### Beginner (Chapters 01–05)
```
□ Explain the 7 OSI layers and what each does
□ Calculate the usable hosts in a /24, /28, /30 subnet
□ Explain the TCP 3-way handshake
□ Describe the DNS resolution process step by step
□ Write a curl command that shows timing breakdown
□ Identify a TLS certificate's expiry date with openssl
□ Explain the difference between 401 and 403, 502 and 504
```

### Intermediate (Chapters 06–09)
```
□ Configure UFW for a web server (allow 22/80/443, deny rest)
□ Write an nginx server block with SSL termination
□ Configure nginx as a reverse proxy with all proxy headers
□ Debug a "connection refused" using ss, nc, and curl
□ Use tcpdump to capture DNS queries
□ Explain round-robin vs least-conn load balancing
□ Read and analyze nginx access logs
```

### Advanced (Chapters 10–11)
```
□ Explain how Docker container networking works at the kernel level
□ Configure a WireGuard VPN
□ Describe the difference between HTTP/2 and HTTP/3
□ Explain what mTLS adds over standard TLS
□ Set up nginx to proxy WebSocket connections
□ Use tcpdump to diagnose a connection reset
□ Design a subnet layout for a 3-tier app in AWS
```

---

## Key Commands Reference

```bash
# ─── IP and Interfaces ─────────────────────────────────────────
ip addr show                          # show all interfaces
ip route show                         # routing table

# ─── Connections ───────────────────────────────────────────────
ss -tlnp                              # listening TCP ports
ss -tan state established             # established connections
ss -s                                 # summary statistics

# ─── DNS ───────────────────────────────────────────────────────
dig example.com                       # A record
dig +short example.com                # just the IP
dig @8.8.8.8 example.com             # specific nameserver
dig -x 1.2.3.4                        # reverse lookup

# ─── Firewall ──────────────────────────────────────────────────
sudo ufw status verbose               # current rules
sudo ufw allow 8080/tcp               # open port
sudo iptables -L -n -v               # raw iptables rules

# ─── HTTP / TLS ────────────────────────────────────────────────
curl -v https://example.com          # verbose HTTP request
curl -w "Total: %{time_total}s\n" -o /dev/null -s https://example.com
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# ─── Packet Analysis ───────────────────────────────────────────
sudo tcpdump -i any -nn port 80      # HTTP traffic
sudo tcpdump -i any -nn udp port 53  # DNS queries

# ─── Port Testing ──────────────────────────────────────────────
nc -zv example.com 443               # TCP port check
nmap -p 80,443 example.com           # port scan

# ─── Nginx ─────────────────────────────────────────────────────
sudo nginx -t                         # test config
sudo nginx -s reload                  # reload (zero downtime)
sudo systemctl status nginx           # service status
```

---

## Mental Models to Keep

**"Connection refused" vs "Connection timed out":**
- Refused → service not listening on that port (or firewall sending RST)
- Timed out → firewall silently dropping packets (no RST)

**Debugging ladder:**
```
DNS → TCP → TLS → HTTP → Application
```
Start at DNS, work up. Each layer depends on the one below.

**Firewall philosophy:**
> Default DENY, explicit ALLOW. If you didn't open it, it's closed.

**NAT in a sentence:**
> Many private IPs share one public IP; router tracks which port belongs to whom.

---

## What's Next

You've completed Topic 2: Networking Basics. The next topic in the DevOps roadmap is:

**Topic 3: Git & Version Control**
- Git internals (objects, refs, the DAG)
- Branching strategies (GitFlow, trunk-based development)
- Merge vs rebase
- Working with remotes (push, pull, fetch)
- Resolving conflicts
- Git hooks and automation
- Monorepos

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="15-interview-preparation.md">← Previous: Interview Preparation</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="../03-Git-Version-Control/00-index.md">Next: Git & Version Control →</a>
</div>
