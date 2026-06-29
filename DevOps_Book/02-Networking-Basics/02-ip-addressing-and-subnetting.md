# Chapter 02 — IP Addressing & Subnetting

## Learning Objectives

By the end of this chapter, you will:
- Understand IPv4 address structure and classes
- Read and calculate CIDR notation (`/24`, `/16`, etc.)
- Identify private vs public IP ranges
- Understand subnetting and why it exists
- Know IPv6 basics and why it matters
- Recognize DevOps-relevant IP concepts (VPC CIDR, pod CIDR)

## Prerequisites

- Chapter 01 — Introduction to Networking

---

## 2.1 What Is an IP Address?

An **IP address** is a unique numerical label assigned to every device on a network. Think of it like a postal address — it tells the network where to deliver packets.

**IPv4** (the current standard): 32-bit number, written as 4 octets (0–255) separated by dots:
```
192.168.1.100
│   │   │ │
│   │   │ └── last octet (host part in /24 network)
│   │   └──── third octet
│   └──────── second octet
└──────────── first octet
```

32 bits = 2³² = ~4.3 billion possible addresses.

---

## 2.2 Binary and IP Addresses

Every IP address is just a 32-bit binary number:

```
192       .168       .1         .100
11000000   10101000   00000001   01100100
```

Understanding binary helps you understand subnetting.

```bash
# Convert IP to binary manually:
# 192 = 128+64 = 11000000
# 168 = 128+32+8 = 10101000
# 1   = 00000001
# 100 = 64+32+4 = 01100100

# Quick conversion in bash:
printf '%d.%d.%d.%d\n' 0xC0 0xA8 0x01 0x64   # hex → decimal
```

---

## 2.3 Network Part vs Host Part

An IP address has two parts:
- **Network part** — identifies the network (like a street name)
- **Host part** — identifies the device on that network (like a house number)

The **subnet mask** (or prefix length) defines how many bits are the network part:

```
IP:      192.168.1.100   →  11000000.10101000.00000001.01100100
Mask:    255.255.255.0   →  11111111.11111111.11111111.00000000
                                 ↑ network part ↑    ↑ host ↑

Network: 192.168.1.0      (all host bits = 0)
Broadcast: 192.168.1.255  (all host bits = 1)
Hosts:   192.168.1.1 – 192.168.1.254 (254 usable addresses)
```

---

## 2.4 CIDR Notation

**CIDR (Classless Inter-Domain Routing)** notation is `IP/prefix_length`:

```
192.168.1.0/24    → 24 bits for network, 8 bits for hosts → 254 hosts
192.168.1.0/16    → 16 bits for network, 16 bits for hosts → 65,534 hosts
192.168.1.0/8     → 8 bits for network, 24 bits for hosts → 16,777,214 hosts
192.168.1.0/30    → 30 bits for network, 2 bits for hosts → 2 hosts (point-to-point)
192.168.1.0/32    → 32 bits for network, 0 bits for hosts → 1 host (single IP)
```

### CIDR Quick Reference

| CIDR | Subnet Mask | Total IPs | Usable Hosts |
|------|------------|-----------|--------------|
| /8 | 255.0.0.0 | 16,777,216 | 16,777,214 |
| /16 | 255.255.0.0 | 65,536 | 65,534 |
| /24 | 255.255.255.0 | 256 | 254 |
| /25 | 255.255.255.128 | 128 | 126 |
| /26 | 255.255.255.192 | 64 | 62 |
| /27 | 255.255.255.224 | 32 | 30 |
| /28 | 255.255.255.240 | 16 | 14 |
| /29 | 255.255.255.248 | 8 | 6 |
| /30 | 255.255.255.252 | 4 | 2 |
| /32 | 255.255.255.255 | 1 | 1 (single host) |

> **Rule:** Each step down in prefix (e.g. /24 → /23) doubles the number of hosts.

### Calculating Network Address

```bash
# Given: IP 192.168.10.50/26
# 1. /26 means 26 network bits, 6 host bits
# 2. 64 addresses per block (2^6)
# 3. Block containing .50: 50 / 64 = 0 remainder 50 → block starts at .0
# Network: 192.168.10.0/26
# Broadcast: 192.168.10.63
# Hosts: 192.168.10.1 – 192.168.10.62

# Use ipcalc for quick calculation:
sudo apt install ipcalc
ipcalc 192.168.10.50/26
```

---

## 2.5 Private IP Ranges

These IP ranges are reserved for **private networks** (home, office, cloud VPCs). They are **not routed on the public internet**.

| Range | CIDR | Addresses | Common Use |
|-------|------|-----------|------------|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | ~16.7M | Corporate, AWS VPC |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | ~1M | Docker default network |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | ~65K | Home networks |

```bash
# You'll see these everywhere in DevOps:
ip addr   # your machine's IP (likely 10.x.x.x or 192.168.x.x)

# Docker uses 172.17.0.0/16 by default
docker network inspect bridge | grep Subnet

# Kubernetes pod CIDR is often 10.244.0.0/16 (flannel) or 192.168.0.0/16 (calico)
kubectl get nodes -o wide   # shows node IPs
```

### Special Addresses

| Address/Range | Purpose |
|---------------|---------|
| `127.0.0.1` / `127.0.0.0/8` | Loopback — packets go to yourself |
| `0.0.0.0` | "All interfaces" or "unspecified" |
| `255.255.255.255` | Limited broadcast |
| `169.254.0.0/16` | Link-local (APIPA — assigned when DHCP fails) |

---

## 2.6 Public IP Addresses

Everything not in the private ranges is a **public IP**, routable on the internet. Your cloud instances get public IPs; your home router has one public IP (shared by all devices via NAT).

```bash
# Find your public IP
curl -s https://api.ipify.org
curl -s ifconfig.me
dig +short myip.opendns.com @resolver1.opendns.com

# Find your local/private IPs
ip addr show | grep "inet " | grep -v "127.0.0.1"
hostname -I
```

---

## 2.7 Subnetting — Dividing Networks

**Subnetting** divides a large network into smaller ones. Why?

1. **Security** — isolate dev from prod, frontend from database
2. **Performance** — reduce broadcast domain size
3. **Organization** — logical grouping of services

### Real DevOps Example: AWS VPC Design

```
VPC: 10.0.0.0/16 (65,534 hosts total)
  ├── Public Subnet:  10.0.1.0/24  (254 hosts) — load balancers, bastion
  ├── Public Subnet:  10.0.2.0/24  (254 hosts) — load balancers (AZ2)
  ├── Private Subnet: 10.0.10.0/24 (254 hosts) — application servers
  ├── Private Subnet: 10.0.11.0/24 (254 hosts) — application servers (AZ2)
  ├── Private Subnet: 10.0.20.0/24 (254 hosts) — databases
  └── Private Subnet: 10.0.21.0/24 (254 hosts) — databases (AZ2)
```

```
Internet → Internet Gateway → Load Balancer (Public Subnet)
                                        → App Servers (Private Subnet)
                                                    → Database (Private Subnet)
```

### Subnetting Exercise

Split `192.168.0.0/24` into 4 equal subnets:
- Need 4 subnets → need 2 bits for subnet → /26 (24 + 2)
- Each subnet: 2^6 = 64 addresses, 62 usable

```
Subnet 1: 192.168.0.0/26   (hosts: .1–.62,   broadcast: .63)
Subnet 2: 192.168.0.64/26  (hosts: .65–.126, broadcast: .127)
Subnet 3: 192.168.0.128/26 (hosts: .129–.190, broadcast: .191)
Subnet 4: 192.168.0.192/26 (hosts: .193–.254, broadcast: .255)
```

---

## 2.8 IPv6 — The Future

IPv4 has ~4.3 billion addresses. The internet has run out. **IPv6** uses 128 bits:

```
IPv4:  192.168.1.1         (32 bits)
IPv6:  2001:0db8:85a3:0000:0000:8a2e:0370:7334   (128 bits)
       2001:db8:85a3::8a2e:370:7334   (abbreviated, :: = consecutive zeros)
```

IPv6 provides 2¹²⁸ ≈ 340 undecillion addresses — enough for every atom on Earth to have an address.

### IPv6 Key Points for DevOps

```bash
# Check IPv6 address
ip -6 addr

# Common IPv6 addresses:
::1                 # loopback (same as 127.0.0.1)
fe80::/10           # link-local (same as 169.254.x.x)
fc00::/7            # unique local (same as RFC1918 private)

# AWS, GCP, Azure all support IPv6
# Docker supports IPv6 networks

# Listen on all IPv6 interfaces:
# [::]:80    (nginx config)
# 0.0.0.0:80 = IPv4 only, [::]:80 = IPv6 (and often IPv4 via dual-stack)
```

---

## 2.9 NAT — Network Address Translation

Since private IPs can't be routed on the internet, **NAT** translates private IPs to public IPs at the boundary (your home router, cloud NAT gateway):

```
Your laptop (192.168.1.5:54321)
        ↓ NAT
Router's public IP (203.0.113.1:54321)
        ↓ internet
github.com

# Return traffic:
github.com → 203.0.113.1:54321
        ↓ NAT table lookup
Router → 192.168.1.5:54321
```

This is why 1 public IP can serve hundreds of devices.

### NAT in DevOps

```
Docker containers: 172.17.0.2:80 → host NAT → host's external IP
Kubernetes pods: 10.244.0.5:8080 → NodePort NAT → node's IP:30080
AWS Lambda: runs in AWS VPC, NAT Gateway provides internet access
```

---

## Summary

```
IP Address: 4 octets (0–255), 32 bits
CIDR /24:   256 total IPs, 254 usable
Private:    10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
Loopback:   127.0.0.1 (goes to yourself)
Public:     Everything else (routable on internet)
NAT:        Translates private ↔ public at network boundary
```

---

## Knowledge Check

1. How many usable hosts does a `/26` subnet provide?
2. Is `172.20.0.1` a public or private IP?
3. What is the broadcast address for `10.0.1.0/24`?
4. Why can't `192.168.1.1` be reached from the public internet directly?
5. What problem does IPv6 solve?

---

## Hands-On Exercise

```bash
# 1. Find your machine's IP and subnet
ip addr show | grep "inet "

# 2. Calculate subnet info
sudo apt install ipcalc -y
ipcalc $(ip addr show | grep "inet " | grep -v "127.0.0.1" | awk '{print $2}' | head -1)

# 3. Find your public IP
curl -s https://api.ipify.org && echo

# 4. Identify private vs public
for ip in 10.0.0.1 172.16.5.1 192.168.1.1 8.8.8.8 203.0.113.5; do
    if [[ "$ip" =~ ^10\. ]] || [[ "$ip" =~ ^172\.(1[6-9]|2[0-9]|3[01])\. ]] || [[ "$ip" =~ ^192\.168\. ]]; then
        echo "$ip → PRIVATE"
    else
        echo "$ip → PUBLIC"
    fi
done

# 5. Check if Docker is using private range
docker network ls 2>/dev/null && docker network inspect bridge 2>/dev/null | grep -A2 '"Subnet"' || echo "Docker not running"
```

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="01-introduction-to-networking.md">← Previous: Introduction to Networking</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="03-tcp-udp-and-ports.md">Next: TCP, UDP & Ports →</a>
</div>
