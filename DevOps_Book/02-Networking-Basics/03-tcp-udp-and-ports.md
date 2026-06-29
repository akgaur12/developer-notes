# Chapter 03 — TCP, UDP & Ports

## Learning Objectives

By the end of this chapter, you will:
- Understand the TCP 3-way handshake and 4-way teardown
- Know when to use TCP vs UDP
- Understand ports and sockets
- Know the well-known port numbers used in DevOps
- Use `ss`, `netstat`, and `nc` to inspect socket state

## Prerequisites

- Chapter 02 — IP Addressing & Subnetting

---

## 3.1 What Are Transport Layer Protocols?

IP (Layer 3) delivers packets to a machine. But a machine runs many processes — a web server, a database, an SSH daemon. How does the OS know which process should receive each packet?

**Ports** and **transport protocols** (TCP/UDP) solve this problem.

```
Incoming packet → OS → Which process should get this?
                        Answer: look at the port number
192.168.1.5:52301 → 10.0.0.1:443  → nginx (HTTPS server)
192.168.1.5:52302 → 10.0.0.1:5432 → postgresql
192.168.1.5:52303 → 10.0.0.1:22   → sshd
```

---

## 3.2 Ports

A **port** is a 16-bit number (0–65535) identifying a specific process on a host.

```
Full connection identifier (5-tuple):
  Protocol  Source IP      Source Port  Dest IP       Dest Port
  TCP       192.168.1.100  52301        10.0.0.1      80
```

This 5-tuple uniquely identifies every connection on the internet.

### Port Ranges

| Range | Name | Description |
|-------|------|-------------|
| 0–1023 | Well-known ports | Reserved for standard services, require root |
| 1024–49151 | Registered ports | Assigned to applications |
| 49152–65535 | Ephemeral ports | Temporary client-side ports |

### Critical Well-Known Ports (Memorize These)

| Port | Protocol | Service |
|------|----------|---------|
| 20, 21 | TCP | FTP (data, control) |
| 22 | TCP | SSH |
| 23 | TCP | Telnet (insecure, avoid) |
| 25 | TCP | SMTP (email sending) |
| 53 | TCP/UDP | DNS |
| 67, 68 | UDP | DHCP (server, client) |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 (email retrieval) |
| 143 | TCP | IMAP (email retrieval) |
| 443 | TCP | HTTPS |
| 465/587 | TCP | SMTP with TLS |
| 993 | TCP | IMAP over TLS |
| 3306 | TCP | MySQL / MariaDB |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080 | TCP | HTTP alternate / dev servers |
| 8443 | TCP | HTTPS alternate |
| 27017 | TCP | MongoDB |

```bash
# See all well-known ports
cat /etc/services | grep -E "ssh|http|mysql|postgres" | head -20
```

---

## 3.3 TCP — Transmission Control Protocol

TCP provides **reliable, ordered, error-checked** delivery. Use it when you can't afford to lose data.

### The 3-Way Handshake — Connection Establishment

Before any data is exchanged, TCP establishes a connection:

```
Client                              Server
  │                                   │
  │─── SYN (seq=100) ────────────────►│   Client: "Hello, I want to connect"
  │                                   │   (seq=100 is client's starting sequence)
  │◄── SYN-ACK (seq=200, ack=101) ────│   Server: "Hello back! And I want to connect"
  │                                   │   (seq=200 is server's starting sequence)
  │─── ACK (ack=201) ────────────────►│   Client: "Acknowledged"
  │                                   │
  │═══════ Data transfer begins ══════│
```

**Why 3 steps?**
- SYN: client picks a starting sequence number
- SYN-ACK: server acknowledges client's seq AND sends its own
- ACK: client acknowledges server's seq

Both sides now know each other's starting sequence numbers — critical for ordering and reassembly.

### TCP Flags

| Flag | Meaning |
|------|---------|
| `SYN` | Synchronize — start connection |
| `ACK` | Acknowledge — confirm receipt |
| `FIN` | Finish — gracefully close |
| `RST` | Reset — abruptly close (error) |
| `PSH` | Push — deliver data immediately |
| `URG` | Urgent — out-of-band data |

### The 4-Way Teardown — Connection Closing

```
Client                              Server
  │                                   │
  │─── FIN ──────────────────────────►│   Client: "I'm done sending"
  │◄── ACK ───────────────────────────│   Server: "OK, got it"
  │◄── FIN ───────────────────────────│   Server: "I'm also done sending"
  │─── ACK ──────────────────────────►│   Client: "OK, got it"
  │                                   │
  │═════════ Connection closed ═══════│
```

### TCP Reliability Mechanisms

**Sequence Numbers** — each byte is numbered; receiver can reassemble out-of-order packets

**Acknowledgments** — receiver sends ACK for data received; sender retransmits if no ACK

**Flow Control** — receiver tells sender how much buffer space it has (receive window)

**Congestion Control** — TCP detects network congestion and slows down (exponential backoff)

```bash
# Watch TCP sequence numbers and flags:
sudo tcpdump -i any -nn -S 'host google.com and tcp' &
curl -s https://google.com > /dev/null
kill %1
```

### TCP Connection States

```bash
ss -tn    # see connection states

# Common states:
# ESTABLISHED  — active connection
# TIME_WAIT    — connection closing, waiting for stray packets
# CLOSE_WAIT   — remote closed, local hasn't yet
# LISTEN       — server waiting for connections
# SYN_SENT     — client sent SYN, waiting for SYN-ACK
# SYN_RECEIVED — server got SYN, sent SYN-ACK, waiting for ACK
```

**TIME_WAIT** accumulation is a common production issue:
```bash
# Count TIME_WAIT connections
ss -tn | grep TIME-WAIT | wc -l

# If too many: adjust kernel parameters
sudo sysctl -w net.ipv4.tcp_tw_reuse=1
echo "net.ipv4.tcp_tw_reuse = 1" | sudo tee -a /etc/sysctl.d/99-tcp.conf
```

---

## 3.4 UDP — User Datagram Protocol

UDP provides **fast, connectionless, unreliable** delivery. No handshake, no acknowledgments, no ordering.

```
Client                              Server
  │                                   │
  │─── data ─────────────────────────►│   Just sends. No handshake.
  │─── data ─────────────────────────►│   Server may or may not receive.
  │─── data ─────────────────────────►│   No ACK.
```

### When to Use UDP

| Use Case | Why UDP |
|----------|---------|
| **DNS queries** | Small request/response, fast more important than reliability |
| **DHCP** | Broadcast-based, can't use TCP before IP address assigned |
| **Video streaming** | Old data is useless, better to drop than retry |
| **VoIP/gaming** | Latency is critical; retransmission would make it worse |
| **NTP (time sync)** | Small packets, best-effort acceptable |
| **QUIC (HTTP/3)** | Uses UDP but implements its own reliability |

```bash
# DNS uses UDP port 53 (TCP for large responses)
sudo tcpdump -i any -nn 'udp port 53' &
dig google.com
kill %1
```

### TCP vs UDP Comparison

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Required (3-way handshake) | None |
| Reliability | Guaranteed delivery | Best-effort |
| Ordering | Maintained | Not maintained |
| Error checking | Yes (retransmit) | Yes (checksum only) |
| Flow control | Yes | No |
| Speed | Slower (overhead) | Faster |
| Use cases | HTTP, SSH, email, databases | DNS, video, gaming, VoIP |

---

## 3.5 Sockets

A **socket** is an endpoint for communication — the combination of an IP address and a port.

```
Socket = IP address : Port
192.168.1.100:52301  (client socket)
10.0.0.1:443         (server socket)

Connection = client socket ↔ server socket
```

```bash
# View sockets in use
ss -tlnp        # listening TCP sockets (servers)
ss -tn          # established TCP connections (active)
ss -un          # UDP sockets
ss -x           # Unix domain sockets (local IPC)

# What's on port 80?
ss -tlnp | grep ':80'
lsof -i :80

# All connections to/from your machine
ss -tan

# Count connections by state
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn
```

---

## 3.6 `netcat` — The TCP/UDP Swiss Army Knife

`netcat` (nc) creates raw TCP or UDP connections — perfect for testing:

```bash
# Test if a port is open
nc -zv hostname 80      # -z = scan only, -v = verbose
nc -zv 10.0.0.5 22      # is SSH open?
nc -zv -w 3 db.internal 5432   # -w 3 = 3 second timeout

# Scan multiple ports
nc -zv hostname 80 443 8080

# Simple HTTP request manually
echo -e "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n" | nc example.com 80

# Create a simple TCP server (useful for testing)
nc -l 8888                          # listen on port 8888
nc -l 8888 < /etc/hostname         # serve a file

# Connect to the server from another terminal
nc localhost 8888

# Send data between machines
# On receiver:
nc -l 9999 > received_file.txt
# On sender:
nc receiver_ip 9999 < file_to_send.txt
```

---

## 3.7 Practical DevOps Examples

### Check if a service is accessible before connecting

```bash
#!/bin/bash
wait_for_port() {
    local host="$1"
    local port="$2"
    local timeout="${3:-30}"
    local elapsed=0

    echo "Waiting for $host:$port..."
    while ! nc -z "$host" "$port" 2>/dev/null; do
        if [[ $elapsed -ge $timeout ]]; then
            echo "Timeout waiting for $host:$port"
            return 1
        fi
        sleep 1
        ((elapsed++))
    done
    echo "$host:$port is ready (${elapsed}s)"
}

wait_for_port db.internal 5432 60     # wait up to 60s for DB
wait_for_port redis.internal 6379 30  # wait up to 30s for Redis
```

### Check all service ports are open after deployment

```bash
services=(
    "localhost:80:HTTP"
    "localhost:443:HTTPS"
    "localhost:8080:App"
    "db.internal:5432:PostgreSQL"
)

for svc in "${services[@]}"; do
    IFS=: read -r host port name <<< "$svc"
    if nc -zw2 "$host" "$port" 2>/dev/null; then
        echo "✓ $name ($host:$port) — open"
    else
        echo "✗ $name ($host:$port) — CLOSED"
    fi
done
```

---

## Summary

```
TCP: reliable, ordered, connected
  SYN → SYN-ACK → ACK (connect)
  FIN → ACK → FIN → ACK (close)

UDP: fast, connectionless, unreliable
  Just sends datagrams, no handshake

Port = process identifier (0–65535)
  Well-known: 0–1023 (HTTP=80, HTTPS=443, SSH=22)
  Ephemeral: 49152–65535 (client-side)

Socket = IP:Port
Connection = client socket ↔ server socket
```

---

## Knowledge Check

1. What three packets are exchanged in a TCP handshake?
2. Why does DNS use UDP instead of TCP?
3. What does the `RST` flag in TCP mean?
4. What is an ephemeral port?
5. How do you check what process is listening on port 8080?

---

## Hands-On Exercise

```bash
# 1. Watch a 3-way handshake in real time
sudo tcpdump -i any -nn 'host example.com and tcp[tcpflags] & (tcp-syn|tcp-ack) != 0' &
curl -s http://example.com > /dev/null
kill %1

# 2. Check all listening ports on your machine
ss -tlnp

# 3. Test port connectivity with nc
nc -zv google.com 80 && echo "Port 80 OPEN"
nc -zv google.com 443 && echo "Port 443 OPEN"
nc -zv localhost 9999 2>/dev/null || echo "Port 9999 CLOSED (expected)"

# 4. Create a quick server and connect to it
nc -l 9999 &          # start listener in background
nc localhost 9999 <<< "Hello from client"
wait

# 5. Count connections by state
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c
```

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="02-ip-addressing-and-subnetting.md">← Previous: IP Addressing & Subnetting</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="04-dns.md">Next: DNS →</a>
</div>
