# Chapter 15 — Interview Preparation

## Learning Objectives

- Answer networking interview questions confidently
- Solve scenario-based troubleshooting problems
- Discuss system design with networking components
- Handle follow-up questions about trade-offs

---

## 15.1 Core Concepts Questions

### "What happens when you type a URL into a browser?"

This is the most common networking interview question. Answer layer by layer:

```
1. URL parsing
   Browser parses: https://api.example.com/users?page=1
   Scheme: https | Host: api.example.com | Path: /users | Query: page=1

2. DNS resolution
   a. Check browser DNS cache (TTL-based)
   b. Check OS cache (/etc/hosts, nscd)
   c. Query recursive resolver (e.g., 8.8.8.8 from /etc/resolv.conf)
   d. Resolver queries root → .com TLD → authoritative → A record
   e. Returns: api.example.com → 203.0.113.5

3. TCP connection (3-way handshake)
   Client ──SYN──► Server
   Client ◄──SYN-ACK── Server
   Client ──ACK──► Server
   Connection established on port 443

4. TLS handshake
   ClientHello (cipher suites, random)
   ServerHello (chosen cipher, certificate)
   Client verifies certificate (CA chain, domain match, expiry)
   Key exchange → session keys derived
   Both send Finished → encrypted channel established

5. HTTP request
   GET /users?page=1 HTTP/2
   Host: api.example.com
   Authorization: Bearer ...

6. Server processes request
   nginx receives → proxies to app server
   App queries database → builds JSON response

7. HTTP response
   HTTP/2 200 OK
   Content-Type: application/json
   [JSON body]

8. Browser renders
   Parses JSON, updates DOM
```

---

### Common Concept Q&A

**Q: What is the difference between TCP and UDP?**

> TCP: reliable, ordered, connection-based. 3-way handshake, ACKs, retransmission. For HTTP, SSH, databases.
> UDP: unreliable, unordered, connectionless. No handshake. For DNS, DHCP, VoIP, gaming, video streaming. Faster because no overhead.

**Q: What is a DNS A record vs CNAME record?**

> A record: maps name directly to IPv4 address (`example.com → 1.2.3.4`).
> CNAME: alias to another name (`www.example.com → example.com`). Can't be used at zone apex (root domain).

**Q: Explain HTTP 502 vs 504.**

> Both come from a proxy (like nginx) that couldn't get a valid response from the backend.
> 502 Bad Gateway: backend returned an invalid/error response, or actively refused connection.
> 504 Gateway Timeout: backend didn't respond within the timeout period.
> 502 = backend is broken or down. 504 = backend is too slow.

**Q: What is NAT and why is it needed?**

> NAT (Network Address Translation) translates private IP addresses to a public IP. It exists because IPv4 has only ~4.3 billion addresses — not enough for every device. NAT allows many devices on a private network to share one public IP. The router maintains a mapping table tracking which internal port belongs to which internal host.

**Q: What is the difference between a reverse proxy and a load balancer?**

> A reverse proxy sits in front of servers, forwarding client requests to backends. A load balancer is a type of reverse proxy that distributes traffic across multiple backend servers. All load balancers are reverse proxies, but not all reverse proxies do load balancing. nginx does both.

**Q: What does SSL termination mean?**

> SSL termination means the reverse proxy (nginx) handles the TLS encryption/decryption. The backend servers communicate with nginx over plain HTTP on the internal network. Benefits: certificate management in one place, backends don't need TLS code, nginx can inspect/modify HTTP headers.

---

## 15.2 Scenario-Based Troubleshooting

### Scenario 1: "Users report the site is down but ping works"

```
Your approach:
1. ping works → Layer 3 (IP) is fine
2. Check DNS: dig example.com → resolves?
3. Check TCP: nc -zv example.com 443 → port open?
4. Check HTTP: curl -v https://example.com → what response?
5. Check backend: curl http://localhost:8080/health → is app responding?
6. Check nginx logs: tail /var/log/nginx/error.log

Likely causes:
- Nginx crashed (systemctl status nginx)
- Backend app crashed (systemctl status myapp)
- Firewall blocking port 443 (sudo ufw status)
- Certificate expired (curl -v will show SSL error)
- App responding but with 5xx errors (nginx access log shows 500/502/504)
```

### Scenario 2: "Site responds slowly only for some users"

```
Your approach:
1. Is it geographic? (users far from server)
   → CDN might help, or multi-region deployment

2. Check response times from different locations
   → curl -w "%{time_total}" from various servers

3. Check DNS TTL and propagation
   → dig from different DNS servers

4. Check server resource usage during slow periods
   → top, iostat, ss -s (too many connections?)

5. Check nginx upstream response times
   → awk '{print $NF}' /var/log/nginx/access.log | sort -n | tail

6. tcpdump during slow request
   → sudo tcpdump -i any -nn host client-ip port 443
```

### Scenario 3: "API calls work from Postman but fail from browser (CORS)"

```
Cause: Browser same-origin policy blocks cross-origin requests
Postman doesn't enforce CORS (it's not a browser)

Symptoms:
  Browser console: "CORS policy: No 'Access-Control-Allow-Origin' header"

Fix in nginx:
  location /api/ {
      add_header Access-Control-Allow-Origin "https://app.example.com";
      add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
      add_header Access-Control-Allow-Headers "Authorization, Content-Type";

      if ($request_method = OPTIONS) {
          return 204;    # pre-flight request
      }

      proxy_pass http://backend;
  }
```

### Scenario 4: "New deploy broke networking, need to rollback fast"

```
Pre-deployment checklist:
□ Test in staging first
□ Keep old config as backup: cp nginx.conf nginx.conf.bak
□ Test new config before applying: nginx -t
□ Have rollback script ready

Emergency rollback:
  # nginx config rollback
  sudo cp /etc/nginx/nginx.conf.bak /etc/nginx/nginx.conf
  sudo nginx -t && sudo nginx -s reload

  # Docker rollback
  docker service update --rollback myservice

  # Kubernetes rollback
  kubectl rollout undo deployment/myapp
```

---

## 15.3 System Design Questions

### "Design a highly available web API"

Key networking components to mention:

```
                         ┌─────────────────────┐
                         │     DNS + CDN        │
                         │  (Cloudflare/Route53)│
                         └──────────┬──────────┘
                                    │
                         ┌──────────▼──────────┐
                         │   Load Balancer      │
                         │   (nginx/ALB)         │
                         └──────┬──────┬────────┘
                                │      │
                     ┌──────────▼──┐ ┌─▼──────────┐
                     │  App Zone A │ │  App Zone B │
                     │  10.0.1.0/24│ │ 10.0.2.0/24 │
                     └──────┬──────┘ └─────┬───────┘
                            │              │
                     ┌──────▼──────────────▼──────┐
                     │     Database (primary/replica) │
                     │     Private subnets only       │
                     └────────────────────────────────┘

Design decisions to discuss:
- Multi-AZ for availability (single AZ = single point of failure)
- Load balancer health checks (remove failed instances)
- CDN for static assets (reduces origin load, global speed)
- Private subnets for DB (security)
- VPC with security groups (least privilege)
- Auto-scaling based on connections/CPU
- TLS everywhere (even internal with mTLS for sensitive services)
```

### "How would you design networking for microservices?"

```
Key concerns:
1. Service discovery: how does Service A find Service B?
   → Kubernetes DNS: http://service-b.namespace.svc.cluster.local
   → Consul service registry

2. Security between services
   → mTLS (mutual TLS) — each service has a certificate
   → Service mesh (Istio) handles this automatically

3. Traffic management
   → Circuit breaking (stop calls to failing services)
   → Retry with backoff
   → Canary deployments (route 5% of traffic to v2)

4. Observability
   → Distributed tracing (each request gets a trace ID)
   → Service mesh collects metrics per service pair

5. Network policies
   → Only allow Service A to call Service B, not Service C
   → Kubernetes NetworkPolicy or Istio AuthorizationPolicy
```

---

## 15.4 Quick-Fire Questions

| Question | Answer |
|---------|--------|
| Default HTTP port? | 80 |
| Default HTTPS port? | 443 |
| Default SSH port? | 22 |
| Default DNS port? | 53 (UDP+TCP) |
| Default PostgreSQL port? | 5432 |
| Default Redis port? | 6379 |
| What is a subnet mask? | Identifies which part of an IP is network vs host |
| What is `/24`? | 256 addresses, 254 usable hosts |
| What is RFC 1918? | Defines private IP ranges (10.x, 172.16.x, 192.168.x) |
| OSI Layer 4? | Transport (TCP/UDP) |
| OSI Layer 7? | Application (HTTP, DNS, SSH) |
| What is ICMP? | Internet Control Message Protocol — used by ping, traceroute |
| What is ARP? | Address Resolution Protocol — resolves IP → MAC address |
| What is BGP? | Border Gateway Protocol — internet routing between ASes |
| What is DHCP? | Assigns IP addresses automatically |
| What is a SYN flood? | DDoS attack sending many SYN packets without completing handshake |

---

## 15.5 "Tell Me About a Hard Networking Problem You Solved"

STAR format for DevOps networking stories:

```
Situation: We had intermittent connection failures between microservices in staging

Task: Needed to diagnose why Service A would randomly fail to reach Service B (5% of calls)

Action:
1. Reproduced by running a load test
2. Used tcpdump to capture traffic during failures: sudo tcpdump -i any -nn host service-b
3. Noticed TCP RST packets during failures
4. Found TIME_WAIT connections exhausting ephemeral ports (ss -s showed 50k TIME_WAIT)
5. Each API call opened a new TCP connection (no connection pooling)
6. Fix: enabled connection pooling in the HTTP client, added keepalive to nginx upstream

Result: Connection failure rate dropped from 5% to 0%. P99 latency improved 40ms.
```

---

## Knowledge Check

1. Walk through what happens at the network level when you run `curl https://github.com`
2. A user reports "connection refused" on port 8080. What are the 3 most likely causes?
3. Nginx returns 502 for all requests. Walk through your debugging steps.
4. What is the difference between a CDN cache hit and a cache miss from a networking perspective?
5. Explain how mTLS adds security that regular TLS doesn't provide.

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="14-projects.md">← Previous: Hands-On Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="16-course-summary.md">Next: Course Summary →</a>
</div>
