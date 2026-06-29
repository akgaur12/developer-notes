# Networking Basics — Complete Course Index

> **DevOps Learning Path — Topic 2 of 11**
> Phase 0: Core Foundations | Estimated Duration: 2–3 weeks

---

## Course Overview

Networking is the invisible foundation every application runs on. When your container can't reach the database, when DNS fails in production, when HTTPS suddenly stops working — you need to diagnose and fix it fast. This course gives you that power.

Everything in DevOps moves over a network: Docker containers communicate via virtual networks, Kubernetes pods have their own IPs, CI/CD pipelines pull from Git over HTTPS, microservices call each other via HTTP. **Understanding networking is not optional.**

**What you will be able to do after this course:**

- Explain how data travels from your browser to a server and back
- Diagnose DNS failures and fix them in under 5 minutes
- Understand TCP vs UDP and when each is used
- Debug HTTP/HTTPS issues using `curl` and browser tools
- Configure nginx as a reverse proxy and load balancer
- Set up and manage firewall rules with UFW and iptables
- Troubleshoot "cannot connect" issues systematically
- Answer networking questions confidently in DevOps interviews

---

## Prerequisites

- Chapter 01–08 of Linux Fundamentals (especially the networking tools chapter)
- Comfort with the Linux terminal
- Basic understanding of files and processes

**Tools to have ready:**
```bash
sudo apt install curl wget traceroute dnsutils netcat nmap nginx -y
```

---

## Estimated Learning Timeline

| Week | Focus Areas | Chapters |
|------|-------------|----------|
| Week 1 | OSI/TCP-IP model, IP addressing, TCP/UDP, DNS | 01–04 |
| Week 2 | HTTP/HTTPS, ports, firewalls, load balancers | 05–07 |
| Week 3 | Nginx, tools, advanced topics, projects | 08–12 |
| Week 4 | Best practices, mistakes, interview prep | 13–16 |

> **Total: 2–3 weeks** (2–3 hrs/day)

---

## Learning Roadmap

```
BEGINNER                    INTERMEDIATE                ADVANCED
──────────────────────      ──────────────────────      ──────────────────────
✓ OSI model overview        ✓ HTTP/HTTPS deep dive      ✓ Load balancing algos
✓ IP addresses & subnets    ✓ TLS/SSL mechanics         ✓ mTLS & service mesh
✓ TCP 3-way handshake       ✓ Firewall rules (UFW)      ✓ HTTP/2 & HTTP/3
✓ UDP and use cases         ✓ Nginx reverse proxy       ✓ Network namespaces
✓ DNS resolution            ✓ Load balancing basics     ✓ BGP & routing basics
✓ Ports and sockets         ✓ CDN concepts              ✓ Performance tuning
```

---

## Milestones

### Milestone 1 — Packets & Addresses (End of Week 1)
- [ ] Explain all 7 OSI layers and what happens at each
- [ ] Calculate subnets from CIDR notation
- [ ] Trace the complete lifecycle of a TCP connection
- [ ] Perform a full DNS lookup manually with `dig`

### Milestone 2 — Application Layer (End of Week 2)
- [ ] Explain HTTP request/response cycle in detail
- [ ] Understand TLS handshake and certificate chain
- [ ] Configure UFW firewall rules for a web server
- [ ] Explain round-robin vs least-connection load balancing

### Milestone 3 — Configuration & Tools (End of Week 3)
- [ ] Configure nginx as a reverse proxy with SSL termination
- [ ] Use `tcpdump` to capture and inspect network traffic
- [ ] Diagnose a broken network connection in under 5 minutes
- [ ] Set up a basic load balancer with nginx upstream blocks

### Milestone 4 — Production Ready (End of Week 4)
- [ ] Complete all three projects
- [ ] Pass all knowledge check questions
- [ ] Answer networking interview questions confidently

---

## Complete Chapter Index

| # | Chapter | Topics | Level |
|---|---------|--------|-------|
| [01](01-introduction-to-networking.md) | Introduction to Networking | OSI model, TCP/IP stack, how the internet works | Beginner |
| [02](02-ip-addressing-and-subnetting.md) | IP Addressing & Subnetting | IPv4, IPv6, CIDR, private ranges, subnetting | Beginner |
| [03](03-tcp-udp-and-ports.md) | TCP, UDP & Ports | 3-way handshake, UDP, sockets, well-known ports | Beginner |
| [04](04-dns.md) | DNS — Domain Name System | Resolution process, record types, dig, troubleshooting | Beginner |
| [05](05-http-and-https.md) | HTTP & HTTPS | Methods, status codes, headers, TLS/SSL, certificates | Intermediate |
| [06](06-ports-and-firewalls.md) | Ports & Firewalls | Port ranges, UFW, iptables, security groups | Intermediate |
| [07](07-load-balancers-and-reverse-proxies.md) | Load Balancers & Reverse Proxies | Algorithms, nginx upstream, health checks | Intermediate |
| [08](08-nginx-basics.md) | Nginx Configuration | Install, virtual hosts, reverse proxy, SSL termination | Intermediate |
| [09](09-networking-tools.md) | Networking Tools Deep Dive | curl, ping, traceroute, tcpdump, nmap, ss | Intermediate |
| [10](10-intermediate-concepts.md) | Intermediate Concepts | NAT, VPN, CDN, network namespaces, Docker networking | Intermediate |
| [11](11-advanced-concepts.md) | Advanced Concepts | HTTP/2, HTTP/3, WebSockets, gRPC, mTLS, BGP intro | Advanced |
| [12](12-best-practices.md) | Networking Best Practices | Security, monitoring, performance tuning | Advanced |
| [13](13-common-mistakes.md) | Common Mistakes & Pitfalls | Diagnosis checklist, real-world failure scenarios | All Levels |
| [14](14-projects.md) | Hands-On Projects | 4 projects: DNS server → full reverse proxy stack | All Levels |
| [15](15-interview-preparation.md) | Interview Preparation | Q&A, scenarios, system design discussions | Advanced |
| [16](16-course-summary.md) | Course Summary & Next Steps | Review, checklist, path to Docker | All Levels |

---

## DevOps Roadmap — Course Series

| # | Course | Status |
|---|--------|--------|
| 1 | [Linux Fundamentals](../01-Linux-Fundamentals/00-index.md) | ✅ Complete |
| 2 | [Networking Basics](../02-Networking-Basics/00-index.md) | ✅ You are here |
| 3 | Git & Version Control | Coming soon |
| 4 | Docker | Coming soon |
| 5 | CI/CD Pipelines | Coming soon |

---

## Recommended Resources

### Books
- *Computer Networks* by Andrew Tanenbaum — the definitive textbook
- *TCP/IP Illustrated* by W. Richard Stevens — deep protocol internals
- *High Performance Browser Networking* by Ilya Grigorik (free at hpbn.co)

### Interactive Tools
- [Wireshark](https://www.wireshark.org/) — GUI packet analyzer
- [curl.trillworks.com](https://curlconverter.com/) — curl to code converter
- [DNS Checker](https://dnschecker.org/) — check DNS propagation globally

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <span></span>
  <a href="./00-index.md">🏠 Index</a>
  <a href="01-introduction-to-networking.md">Next: Introduction to Networking →</a>
</div>
