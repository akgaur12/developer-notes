# Chapter 10 — Intermediate Networking Concepts

## Learning Objectives

By the end of this chapter, you will:
- Understand NAT and why it exists
- Know VPN types and use cases in DevOps
- Understand CDN architecture
- Know Docker networking (bridge, host, overlay)
- Understand Linux network namespaces
- Configure container networking in practice

## Prerequisites

- Chapter 09 — Networking Tools Deep Dive

---

## 10.1 NAT — Network Address Translation

**NAT** translates private IP addresses to public IP addresses, allowing many devices to share one public IP.

```
Private Network                   Internet
192.168.1.10 ──┐
192.168.1.11 ──┤── [NAT Router: 203.0.113.5] ──► google.com
192.168.1.12 ──┘
```

### How NAT Works

```
Outgoing packet:
  src: 192.168.1.10:50234  dst: 8.8.8.8:53
  ─── NAT translates ────►
  src: 203.0.113.5:50234   dst: 8.8.8.8:53

Response packet:
  src: 8.8.8.8:53          dst: 203.0.113.5:50234
  ─── NAT translates ────►
  src: 8.8.8.8:53          dst: 192.168.1.10:50234
```

### NAT Types

| Type | Description | Example |
|------|-------------|---------|
| **SNAT** (Source NAT) | Replace source IP on outgoing packets | Home router → ISP |
| **DNAT** (Destination NAT) | Replace destination IP on incoming packets | Port forwarding |
| **Masquerade** | Dynamic SNAT (DHCP-friendly) | Linux `iptables -t nat -A POSTROUTING -j MASQUERADE` |
| **PAT** (Port Address Translation) | NAT + port tracking (most common) | All home routers |

```bash
# Configure Linux as NAT router
echo 1 > /proc/sys/net/ipv4/ip_forward   # enable packet forwarding
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE  # masquerade on external interface
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT          # allow forwarding from internal to external
iptables -A FORWARD -i eth0 -o eth1 -m state --state ESTABLISHED,RELATED -j ACCEPT

# Port forwarding (DNAT): route external port 8080 to internal 192.168.1.10:80
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.10:80
iptables -A FORWARD -p tcp -d 192.168.1.10 --dport 80 -j ACCEPT
```

---

## 10.2 VPN — Virtual Private Network

A **VPN** creates an encrypted tunnel across a public network, making remote devices appear to be on the same private network.

```
Office Network (10.0.0.0/24)          Remote Developer
  app-server: 10.0.0.5                 Laptop: 85.1.2.3
       │                                     │
       └──────── encrypted VPN tunnel ───────┘
                 (developer appears as 10.0.0.100)
```

### VPN Types in DevOps

| Type | Protocol | Use Case |
|------|----------|---------|
| **Site-to-site** | IPsec, WireGuard | Connect two office networks |
| **Remote access** | WireGuard, OpenVPN | Developer connecting to infra |
| **Cloud VPN** | AWS VPN, GCP Cloud VPN | Connect on-prem to cloud VPC |

### WireGuard — Modern VPN

```bash
# Install WireGuard
sudo apt install wireguard -y

# Generate key pair
wg genkey | tee server_private.key | wg pubkey > server_public.key

# Server config: /etc/wireguard/wg0.conf
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <server_private_key>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.8.0.2/32    # only this IP from this client

# Start WireGuard
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0

# Status
sudo wg show
```

---

## 10.3 CDN — Content Delivery Network

A **CDN** distributes content across geographically distributed servers (PoPs — Points of Presence), so users download from the nearest location.

```
Without CDN:
  User in Mumbai ──────────────────► Server in New York (~200ms)

With CDN:
  User in Mumbai ──► CDN PoP (Mumbai) ──► Server in New York (first request only)
                     ↑
                  Cached here (~10ms for subsequent requests)
```

### How CDN Caching Works

```
Request 1 (cache miss):
  Browser → CDN PoP → Origin Server (get content) → CDN PoP (cache) → Browser

Request 2+ (cache hit):
  Browser → CDN PoP (serve from cache) → Browser
```

### CDN Cache Control

Your origin server controls what CDN caches:

```nginx
# nginx on origin server
location ~* \.(jpg|png|css|js|woff2)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";    # CDN can cache this
}

location /api/ {
    add_header Cache-Control "private, no-cache";    # don't cache API responses
}

location /user-data/ {
    add_header Cache-Control "no-store";             # never cache sensitive data
}
```

### CDN Concepts for DevOps

```bash
# Invalidate CDN cache (AWS CloudFront)
aws cloudfront create-invalidation \
  --distribution-id E1234567890 \
  --paths "/*"    # all paths

# Invalidate specific paths
aws cloudfront create-invalidation \
  --distribution-id E1234567890 \
  --paths "/static/app.js" "/static/app.css"

# Check if response came from CDN cache
curl -I https://cdn.example.com/logo.png | grep -i "x-cache\|cf-cache"
# X-Cache: Hit from cloudfront   ← served from cache
# X-Cache: Miss from cloudfront  ← went to origin
```

---

## 10.4 Linux Network Namespaces

Network namespaces are the kernel feature that makes container networking work. Each namespace has its own network interfaces, routing table, and firewall rules.

```bash
# List namespaces
ip netns list

# Create a namespace
sudo ip netns add myns

# Run command in namespace
sudo ip netns exec myns ip addr     # sees only loopback
sudo ip netns exec myns ping 8.8.8.8   # fails — no connectivity yet

# Create a veth pair (virtual ethernet — like a cable with two ends)
sudo ip link add veth0 type veth peer name veth1

# Move one end into namespace
sudo ip link set veth1 netns myns

# Configure both ends
sudo ip addr add 10.200.0.1/24 dev veth0
sudo ip netns exec myns ip addr add 10.200.0.2/24 dev veth1

# Bring interfaces up
sudo ip link set veth0 up
sudo ip netns exec myns ip link set veth1 up
sudo ip netns exec myns ip link set lo up

# Now they can communicate
ping 10.200.0.2        # from host to namespace
sudo ip netns exec myns ping 10.200.0.1   # from namespace to host

# Delete namespace
sudo ip netns del myns
```

This is exactly what Docker does — each container gets a namespace connected to a bridge via a veth pair.

---

## 10.5 Docker Networking

Docker has several network drivers:

```
Bridge (default):        Host:                    None:
Container ─► docker0 ─► eth0    Container ─► eth0    Container (isolated)
```

### Bridge Networking (Default)

```bash
# See Docker networks
docker network ls
docker network inspect bridge

# Default bridge — containers can reach internet but not each other by name
docker run -d --name web nginx

# User-defined bridge — containers can reach each other by name
docker network create mynet
docker run -d --name web --network mynet nginx
docker run -d --name app --network mynet myapp

# Now "app" container can reach "web" container by name:
# curl http://web:80   (Docker DNS resolves "web" to container IP)
```

### Host Networking

Container shares host's network stack — no isolation, no NAT:

```bash
docker run --network host nginx    # nginx listens on host's port 80
```

Use for performance-critical apps where port mapping overhead matters, or containers that need to bind arbitrary ports.

### Port Mapping

```bash
docker run -p 8080:80 nginx        # host:8080 → container:80
docker run -p 127.0.0.1:8080:80 nginx  # bind only on localhost (safer)

# See port mappings
docker port container_name
```

### Container-to-Container Networking

```bash
# docker-compose.yml
services:
  web:
    image: nginx
    ports:
      - "80:80"
    networks:
      - frontend

  app:
    image: myapp
    networks:
      - frontend
      - backend

  db:
    image: postgres
    networks:
      - backend    # db not reachable from web directly

networks:
  frontend:
  backend:
```

### Overlay Networks (Swarm/Kubernetes)

```bash
# Docker Swarm — overlay spans multiple hosts
docker network create --driver overlay --attachable myoverlay

# Containers on different hosts can communicate by name
# Docker uses VXLAN to tunnel traffic across hosts
```

---

## Summary

| Concept | What It Does |
|---------|--------------|
| NAT | Translates private IPs to public; enables internet access from private networks |
| VPN | Encrypted tunnel; remote devices join private network |
| CDN | Distributed caching; serves content from nearest PoP |
| Net Namespace | Kernel isolation; foundation for container networking |
| Docker Bridge | Default container networking; user-defined bridges enable DNS resolution |
| Docker Host | Container shares host network stack; no isolation |
| Overlay | Multi-host container networking; VXLAN-based |

---

## Knowledge Check

1. What is the difference between SNAT and DNAT?
2. Why can containers on a user-defined Docker bridge resolve each other by name, but containers on the default bridge cannot?
3. What is a veth pair and how is it used in container networking?
4. How do you tell a CDN not to cache an API response?
5. What kernel feature do containers use for network isolation?

---

## Hands-On Exercise

```bash
# 1. Explore Docker networking
docker network ls
docker network inspect bridge

# 2. Create a user-defined network and test DNS resolution
docker network create testnet

# Start two containers on the same network
docker run -d --name web --network testnet nginx
docker run -it --network testnet alpine sh

# Inside alpine, resolve and reach nginx by name:
# ping web
# wget -O- http://web

# 3. Test port mapping
docker run -d -p 8088:80 --name nginx-test nginx
curl http://localhost:8088

# Cleanup
docker rm -f web nginx-test
docker network rm testnet
```

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="09-networking-tools.md">← Previous: Networking Tools Deep Dive</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="11-advanced-concepts.md">Next: Advanced Concepts →</a>
</div>
