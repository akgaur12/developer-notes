# Chapter 09 — Networking Tools Deep Dive

## Learning Objectives

By the end of this chapter, you will:
- Master `curl` for API testing and HTTP debugging
- Use `ping`, `traceroute`, and `mtr` for connectivity testing
- Capture and analyze traffic with `tcpdump`
- Scan ports with `nmap`
- Use `ss` and `netstat` for socket inspection
- Combine tools into a systematic debugging workflow

## Prerequisites

- Chapter 08 — Nginx Basics

---

## 9.1 `curl` — Advanced Usage

```bash
# ─── Timing Analysis ──────────────────────────────────────────
curl -w "DNS: %{time_namelookup}s | TCP: %{time_connect}s | TLS: %{time_appconnect}s | TTFB: %{time_starttransfer}s | Total: %{time_total}s\n" \
     -o /dev/null -s https://github.com

# ─── Download ──────────────────────────────────────────────────
curl -L -o kubectl "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -C - -O https://example.com/bigfile.tar.gz    # resume download

# ─── Upload ────────────────────────────────────────────────────
curl -F "file=@/path/to/file.txt" https://upload.example.com/
curl -T /path/to/file ftp://ftp.example.com/

# ─── Authentication ────────────────────────────────────────────
curl -u user:pass https://api.example.com/
curl -H "Authorization: Bearer $TOKEN" https://api.example.com/
curl --netrc-file ~/.netrc https://api.example.com/   # credentials from file

# ─── Debugging TLS ─────────────────────────────────────────────
curl -v --tlsv1.2 https://example.com 2>&1 | grep -E "SSL|TLS|Cipher"
curl -k https://self-signed.example.com    # skip cert verification (dev only)
curl --cacert /path/to/ca.crt https://internal.example.com

# ─── Proxy ─────────────────────────────────────────────────────
curl -x http://proxy:3128 https://example.com   # use HTTP proxy
curl --socks5 proxy:1080 https://example.com    # use SOCKS5 proxy

# ─── Headers Control ───────────────────────────────────────────
curl -H "Content-Type: application/json" \
     -H "X-Request-ID: $(uuidgen)" \
     -X POST https://api.example.com/events \
     -d '{"type": "deploy", "version": "1.5"}'

# ─── Rate Testing ──────────────────────────────────────────────
# Send 10 sequential requests
for i in $(seq 1 10); do
    curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" https://api.example.com/
done

# ─── Conditional Requests ──────────────────────────────────────
curl -I https://example.com/file.js                    # get ETag
curl -H 'If-None-Match: "abc123"' https://example.com/file.js  # conditional GET
```

---

## 9.2 `ping` — ICMP Connectivity

```bash
ping google.com               # continuous (Ctrl+C to stop)
ping -c 4 google.com          # 4 packets only
ping -i 0.2 google.com        # every 0.2 seconds
ping -s 1400 google.com       # larger packet size (test MTU)
ping -q google.com -c 100     # quiet mode, 100 packets

# Broadcast ping (find all hosts on subnet)
ping -b 192.168.1.255

# IPv6 ping
ping6 ::1                     # loopback
ping6 google.com              # IPv6 to google
```

### Reading ping Output

```
64 bytes from 142.250.80.46: icmp_seq=1 ttl=55 time=12.3 ms
│                              │         │         │
│                              │         │         └── round-trip time
│                              │         └── Time To Live (decremented each hop)
│                              └── sequence number
└── bytes in response
```

**Packet loss >1%** in production = investigate routing, check for congestion.

---

## 9.3 `traceroute` / `mtr` — Path Analysis

```bash
# traceroute (may need: sudo apt install traceroute)
traceroute google.com
traceroute -n google.com       # no DNS lookups (faster)
traceroute -T google.com       # TCP instead of ICMP (bypasses some firewalls)
traceroute -p 80 google.com    # use port 80

# mtr — combines ping + traceroute (interactive)
mtr google.com
mtr --report google.com        # non-interactive report
mtr -n --report google.com     # no DNS, report mode
```

### Reading traceroute

```
1  192.168.1.1 (home router)     1.234 ms
2  10.20.0.1 (ISP gateway)      12.345 ms
3  72.14.196.74 (Google peering) 20.123 ms
4  * * *                         (no response — ICMP blocked)
5  142.250.80.46 (destination)   25.678 ms
```

`* * *` — hop doesn't respond to probes but traffic passes through. Normal.  
High latency on one hop — may indicate congestion or long physical distance.

---

## 9.4 `tcpdump` — Packet Capture

`tcpdump` captures raw network packets. Essential for deep debugging.

```bash
# Basic capture
sudo tcpdump                           # all interfaces, all traffic
sudo tcpdump -i eth0                   # specific interface
sudo tcpdump -i any                    # all interfaces

# Filtering (BPF syntax)
sudo tcpdump -i any host 8.8.8.8       # traffic to/from 8.8.8.8
sudo tcpdump -i any port 80            # HTTP traffic
sudo tcpdump -i any port 80 or port 443
sudo tcpdump -i any tcp                # TCP only
sudo tcpdump -i any udp port 53        # DNS queries

# Readable output
sudo tcpdump -i any -nn port 80        # -nn = no DNS/port name resolution (faster)
sudo tcpdump -i any -A port 80         # -A = ASCII payload (see HTTP headers!)
sudo tcpdump -i any -X port 80         # -X = hex + ASCII

# Save to file (for analysis in Wireshark)
sudo tcpdump -i any -w capture.pcap
sudo tcpdump -r capture.pcap           # read from file
sudo tcpdump -r capture.pcap -n port 80

# Limit capture size
sudo tcpdump -i any -c 100 port 80     # capture 100 packets
sudo tcpdump -i any -G 60 -w capture_%Y%m%d_%H%M%S.pcap  # rotate every 60s
```

### tcpdump for Common Tasks

```bash
# Watch DNS queries in real time
sudo tcpdump -i any -nn udp port 53

# See HTTP requests and responses (plain HTTP)
sudo tcpdump -i any -A 'tcp port 80 and (tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354)'
# Simpler approach:
sudo tcpdump -i any -A port 80 2>/dev/null | grep -E "GET |POST |HTTP/"

# Watch TCP handshake
sudo tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0' port 443

# Find connection resets (potential issues)
sudo tcpdump -i any 'tcp[tcpflags] & tcp-rst != 0'

# Capture only first 100 bytes of each packet (enough for headers)
sudo tcpdump -i any -s 100 port 80
```

---

## 9.5 `nmap` — Network Scanner

`nmap` discovers hosts and scans ports.

```bash
sudo apt install nmap -y

# Port scan
nmap 192.168.1.1                   # default: top 1000 TCP ports
nmap -p 80,443,8080 192.168.1.1    # specific ports
nmap -p 1-1024 192.168.1.1         # port range
nmap -p- 192.168.1.1               # all 65535 ports (slow)

# Scan multiple hosts
nmap 192.168.1.1 192.168.1.2
nmap 192.168.1.0/24                # entire subnet
nmap 192.168.1.1-50                # range

# Service and version detection
nmap -sV 192.168.1.1               # detect service versions
nmap -sV -p 22,80,443 192.168.1.1

# OS detection (requires root)
sudo nmap -O 192.168.1.1

# Fast scan
nmap -F 192.168.1.1                # top 100 ports

# UDP scan
sudo nmap -sU 192.168.1.1          # UDP (slower)
sudo nmap -sU -p 53,67,123 192.168.1.1

# Ping scan (find live hosts)
nmap -sn 192.168.1.0/24            # host discovery only

# Verbose
nmap -v -p 80,443 192.168.1.1
```

> **Important:** Only scan networks you own or have explicit permission to scan. Unauthorized scanning is illegal in many jurisdictions.

---

## 9.6 `ss` — Socket Statistics

```bash
# Listening TCP ports
ss -tlnp
# -t = TCP, -l = listening, -n = numeric, -p = process

# Established connections
ss -tn state established

# All TCP connections
ss -tan

# Filter by port
ss -tn 'sport = :80'    # source port 80
ss -tn 'dport = :443'   # destination port 443

# Filter by state
ss -tn state time-wait
ss -tn state close-wait
ss -tn state syn-sent

# Show timers
ss -to

# Show memory usage per socket
ss -tm

# Count connections per state
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# Unix domain sockets
ss -x

# Statistics summary
ss -s
```

---

## 9.7 Systematic Network Debugging

Use this methodology when something "can't connect":

```bash
#!/bin/bash
# debug-connection.sh — systematic network debugging
TARGET_HOST="${1:-google.com}"
TARGET_PORT="${2:-443}"

echo "=== Network Debug: $TARGET_HOST:$TARGET_PORT ==="
echo ""

# 1. DNS Resolution
echo "1. DNS Resolution"
IP=$(dig +short "$TARGET_HOST" 2>/dev/null | head -1)
if [[ -n "$IP" ]]; then
    echo "   ✓ $TARGET_HOST → $IP"
else
    echo "   ✗ DNS FAILED — cannot resolve $TARGET_HOST"
    echo "   Check: cat /etc/resolv.conf"
    echo "   Try:   dig @8.8.8.8 $TARGET_HOST"
    exit 1
fi

# 2. ICMP Ping
echo ""
echo "2. ICMP Ping"
if ping -c 2 -W 2 "$TARGET_HOST" &>/dev/null; then
    echo "   ✓ Host responds to ping"
else
    echo "   ✗ No ping response (may be firewall blocking ICMP, not necessarily down)"
fi

# 3. TCP Port Check
echo ""
echo "3. TCP Port $TARGET_PORT"
if nc -zw3 "$TARGET_HOST" "$TARGET_PORT" 2>/dev/null; then
    echo "   ✓ Port $TARGET_PORT is OPEN"
else
    echo "   ✗ Port $TARGET_PORT is CLOSED or FILTERED"
    echo "   Check: sudo ufw status"
    echo "   Check: ss -tlnp | grep :$TARGET_PORT"
    exit 1
fi

# 4. HTTP Response (if port 80 or 443)
if [[ "$TARGET_PORT" == "80" || "$TARGET_PORT" == "443" ]]; then
    echo ""
    echo "4. HTTP Response"
    PROTO="http"
    [[ "$TARGET_PORT" == "443" ]] && PROTO="https"
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 "$PROTO://$TARGET_HOST" 2>/dev/null)
    echo "   HTTP Status: $STATUS"
fi

echo ""
echo "=== Debug Complete ==="
```

---

## Summary

| Tool | Primary Use |
|------|-------------|
| `curl -v URL` | HTTP request debugging |
| `curl -w "..." URL` | HTTP timing analysis |
| `ping host` | ICMP connectivity |
| `traceroute host` | Path tracing |
| `mtr host` | Continuous path analysis |
| `tcpdump -i any port N` | Packet capture |
| `nmap -p PORT host` | Port scanning |
| `ss -tlnp` | Listening socket inspection |
| `nc -zv host port` | Quick port test |

---

## Knowledge Check

1. What information does `traceroute` give you that `ping` doesn't?
2. How do you capture only HTTP traffic with tcpdump?
3. What does `ss -tlnp` show?
4. What does `* * *` in traceroute output mean?
5. When would you use `nmap` vs `nc -zv`?

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="08-nginx-basics.md">← Previous: Nginx Basics</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="10-intermediate-concepts.md">Next: Intermediate Concepts →</a>
</div>
