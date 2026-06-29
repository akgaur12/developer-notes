# Chapter 08 — Networking Tools

## Learning Objectives

By the end of this chapter, you will:
- Test connectivity with `ping`, `traceroute`, and `mtr`
- Download files and test HTTP endpoints with `curl` and `wget`
- Inspect network connections with `ss` and `netstat`
- Look up DNS records with `dig` and `nslookup`
- Manage SSH connections and key-based authentication
- Configure basic network interfaces
- Debug common network issues in production

## Prerequisites

- Chapter 07 — Process Management

---

## 8.1 The Networking Toolkit Every DevOps Engineer Needs

Network debugging is a daily activity in DevOps:
- "Is the service reachable from outside?"
- "Is DNS resolving correctly?"
- "What port is this process listening on?"
- "Why is my HTTPS request failing?"
- "How do I connect to a remote server securely?"

These questions have specific tools as answers. Learn them cold.

---

## 8.2 `ping` — Basic Connectivity Test

`ping` sends ICMP echo request packets to test if a host is reachable.

```bash
ping google.com            # continuous ping (Ctrl+C to stop)
ping -c 4 google.com       # send exactly 4 packets
ping -c 4 8.8.8.8          # ping Google's DNS server
ping -i 0.5 google.com     # ping every 0.5 seconds
ping localhost             # ping yourself (test TCP/IP stack)
ping 127.0.0.1             # same: loopback address
```

### Reading ping Output

```
PING google.com (142.250.80.46) 56(84) bytes of data.
64 bytes from lga25s78-in-f14.1e100.net (142.250.80.46): icmp_seq=1 ttl=55 time=12.3 ms
64 bytes from ... : icmp_seq=2 ttl=55 time=11.8 ms

--- google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 11.8/12.0/12.3/0.2 ms
```

- **time=12.3ms** — round-trip time (lower is better)
- **0% packet loss** — no packets dropped (good)
- **TTL** — Time To Live, decremented each hop (helps detect routing issues)

> **Limitations:** Many firewalls block ICMP. A host not responding to ping doesn't necessarily mean it's down — it may just block ICMP.

---

## 8.3 `traceroute` / `tracepath` — Trace Network Path

Traces the route packets take from your machine to a destination, showing each hop.

```bash
traceroute google.com      # trace path (may need: sudo apt install traceroute)
tracepath google.com       # similar, built-in on Ubuntu
mtr google.com             # interactive, combines ping + traceroute (install: sudo apt install mtr)
```

```
traceroute to google.com (142.250.80.46), 30 hops max
 1  192.168.1.1 (router)        1.234 ms
 2  10.20.0.1 (ISP gateway)    12.345 ms
 3  72.14.196.74               20.123 ms
 4  * * *                      (firewall blocking)
 5  142.250.80.46              25.678 ms
```

`* * *` means no response from that hop (doesn't mean failure).

---

## 8.4 `curl` — The HTTP Swiss Army Knife

`curl` transfers data using URLs — HTTP, HTTPS, FTP, and more. It's the primary tool for testing APIs and web services.

### Basic Usage

```bash
curl https://google.com                    # GET request, print body
curl -o output.html https://google.com     # save to file
curl -O https://example.com/file.tar.gz   # save with remote filename
curl -L https://example.com               # follow redirects
curl -I https://google.com                # headers only (HEAD request)
curl -s https://google.com                # silent (no progress bar)
curl -v https://google.com                # verbose (see all headers)
```

### HTTP Methods

```bash
# GET (default)
curl https://api.example.com/users

# POST with JSON body
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Akash", "email": "akash@example.com"}'

# PUT
curl -X PUT https://api.example.com/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated Name"}'

# DELETE
curl -X DELETE https://api.example.com/users/1

# POST with form data
curl -X POST https://example.com/login \
  -d "username=admin&password=secret"
```

### Authentication

```bash
# Basic auth
curl -u username:password https://api.example.com/secure

# Bearer token
curl -H "Authorization: Bearer mytoken123" https://api.example.com/data

# API key in header
curl -H "X-API-Key: mykey" https://api.example.com/data
```

### Essential curl Flags

| Flag | Meaning |
|------|---------|
| `-X METHOD` | HTTP method (GET, POST, PUT, DELETE) |
| `-H "Header: Value"` | Add request header |
| `-d "data"` | Request body data |
| `-o file` | Save output to file |
| `-O` | Save with remote filename |
| `-L` | Follow redirects |
| `-s` | Silent (no progress) |
| `-v` | Verbose output |
| `-I` | HEAD request (headers only) |
| `-k` | Insecure (skip TLS verification) |
| `-w "%{http_code}"` | Print HTTP status code |
| `--compressed` | Accept compressed response |
| `-u user:pass` | Basic authentication |
| `--connect-timeout N` | Connection timeout in seconds |
| `--max-time N` | Maximum total time |

### Real DevOps curl Examples

```bash
# Test if a service is up and get HTTP status code
curl -s -o /dev/null -w "%{http_code}" https://myapp.example.com

# Health check endpoint
curl -f https://api.example.com/health || echo "Service down!"
# -f = fail silently with exit code 22 on HTTP errors (4xx/5xx)

# Download a binary (kubectl, helm, etc.)
curl -LO "https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Test an API endpoint with pretty-printed JSON
curl -s https://api.example.com/data | python3 -m json.tool
curl -s https://api.example.com/data | jq .   # requires: sudo apt install jq

# Measure request timing
curl -s -o /dev/null -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTotal: %{time_total}s\n" https://example.com

# Send file as POST body
curl -X POST https://api.example.com/upload \
  -F "file=@/path/to/file.txt"

# Test with specific TLS version
curl --tlsv1.2 https://example.com
```

---

## 8.5 `wget` — File Downloader

`wget` is primarily a download tool, better than curl for recursive downloads and resuming.

```bash
wget https://example.com/file.tar.gz           # download file
wget -O output.tar.gz https://example.com/f.gz # save as specific name
wget -c https://example.com/bigfile.tar.gz     # resume interrupted download
wget -q https://example.com/file               # quiet mode
wget --no-check-certificate https://...        # skip TLS verification
wget -r -l2 https://example.com/               # recursive download, 2 levels deep
wget -b https://example.com/bigfile.tar.gz     # download in background
```

### curl vs wget

| Feature | curl | wget |
|---------|------|------|
| Protocols | HTTP, HTTPS, FTP, more | HTTP, HTTPS, FTP |
| API testing | Excellent (methods, headers, auth) | Basic |
| File download | Good | Excellent (resume, recursive) |
| Piping output | Natural (`curl url | command`) | Less natural |
| Available | Usually pre-installed | Sometimes needs install |

> **Use curl** for API testing and DevOps automation. **Use wget** for straightforward file downloads.

---

## 8.6 `ss` — Socket Statistics (Modern netstat)

`ss` shows network socket information. It's the modern replacement for `netstat`.

```bash
ss -tlnp           # TCP listening ports with process names
ss -ulnp           # UDP listening ports
ss -tlnp4          # TCP IPv4 only
ss -tlnp6          # TCP IPv6 only
ss -a              # all sockets (listening + established)
ss -s              # summary statistics
ss -o              # show timer information
```

### Decoding `ss -tlnp` Output

```
State    Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
LISTEN   0       128     0.0.0.0:80          0.0.0.0:*          users:(("nginx",pid=1234,fd=6))
LISTEN   0       128     0.0.0.0:443         0.0.0.0:*          users:(("nginx",pid=1234,fd=7))
LISTEN   0       128     127.0.0.1:5432      0.0.0.0:*          users:(("postgres",pid=5678,fd=5))
```

- `0.0.0.0:80` — listening on ALL interfaces on port 80 (public)
- `127.0.0.1:5432` — listening only on localhost (private, not externally accessible)
- `:::80` — listening on all IPv6 interfaces

```bash
# Find what's using port 80
ss -tlnp | grep :80

# Find all established connections
ss -tn state established

# Find connections to a specific remote port
ss -tn dst :443

# Count established connections
ss -tn state established | wc -l
```

---

## 8.7 `netstat` — Network Statistics (Legacy)

`netstat` is older but still widely available and used in scripts:

```bash
netstat -tlnp          # same as ss -tlnp
netstat -an            # all connections numeric
netstat -rn            # routing table
netstat -i             # interface statistics
```

> `ss` is faster and more powerful. Use `ss` in new work, but know `netstat` for older systems.

---

## 8.8 DNS Tools — `dig`, `nslookup`, `host`

### `dig` — DNS Lookup (Professional)

```bash
dig google.com                    # A record (IPv4 address)
dig google.com A                  # explicit A record
dig google.com AAAA               # IPv6 address
dig google.com MX                 # mail servers
dig google.com TXT                # text records (SPF, DKIM)
dig google.com NS                 # nameservers
dig google.com CNAME              # canonical name

# Use a specific DNS server
dig @8.8.8.8 google.com           # query Google's DNS
dig @1.1.1.1 google.com           # query Cloudflare's DNS

# Reverse lookup
dig -x 8.8.8.8                    # IP to hostname

# Short output
dig +short google.com
dig +short google.com MX

# Trace the full DNS resolution
dig +trace google.com
```

### Reading dig Output

```
;; ANSWER SECTION:
google.com.    299    IN    A    142.250.80.46
     │          │     │    │         │
     domain   TTL(s) class type    IP address
```

### `nslookup` — Simpler DNS Lookup

```bash
nslookup google.com
nslookup google.com 8.8.8.8     # use specific DNS server
nslookup -type=MX google.com    # query MX records
```

### `host` — Quickest DNS Lookup

```bash
host google.com
host 8.8.8.8            # reverse lookup
host -t MX google.com
```

---

## 8.9 `ip` — Network Interface Management

`ip` is the modern replacement for `ifconfig`:

```bash
ip addr                    # show all interfaces and IPs
ip addr show eth0          # show specific interface
ip link                    # show interface state
ip route                   # routing table
ip route show default      # default gateway

# Bring interface up/down (requires sudo)
sudo ip link set eth0 up
sudo ip link set eth0 down

# Add/remove IP address (temporary, lost on reboot)
sudo ip addr add 192.168.1.100/24 dev eth0
sudo ip addr del 192.168.1.100/24 dev eth0
```

```bash
# Quick: show your IP addresses
ip addr | grep "inet " | awk '{print $2, $NF}'

# Show default gateway
ip route | grep default
```

### `ifconfig` — Legacy Interface Tool

```bash
ifconfig               # show all interfaces
ifconfig eth0          # show specific interface
# Available but deprecated — use 'ip' instead
```

---

## 8.10 SSH — Secure Shell

SSH is how you connect to remote servers. It's a critical DevOps skill.

### Basic SSH

```bash
ssh user@hostname               # connect to remote server
ssh akash@192.168.1.100         # connect by IP
ssh akash@server.example.com    # connect by hostname
ssh -p 2222 akash@server        # custom port (default is 22)
ssh -v akash@server             # verbose (debugging)
```

### SSH Key-Based Authentication

Using SSH keys is more secure than passwords and required for automation.

```bash
# 1. Generate SSH key pair (on your LOCAL machine)
ssh-keygen -t ed25519 -C "akash@work"
# This creates:
#   ~/.ssh/id_ed25519       (private key — NEVER share this)
#   ~/.ssh/id_ed25519.pub   (public key — share this)

# 2. Copy public key to remote server
ssh-copy-id user@remote-server
# Or manually:
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# 3. Set correct permissions on remote server
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# 4. Now login without password
ssh user@remote-server
```

### SSH Config File

Create `~/.ssh/config` for shortcuts:

```
Host myserver
    HostName 192.168.1.100
    User akash
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host production
    HostName prod.example.com
    User deploy
    Port 2222
    IdentityFile ~/.ssh/prod_key
```

Now you can just: `ssh myserver` or `ssh production`

### SSH Port Forwarding (Tunneling)

```bash
# Local port forwarding: access remote service locally
# Forward local port 8080 to remote port 80 (useful for databases, internal services)
ssh -L 8080:localhost:80 user@remote-server
# Now: curl http://localhost:8080 → hits remote server's port 80

# Access remote database from local machine
ssh -L 5432:localhost:5432 user@db-server
# Now connect to localhost:5432 to reach remote PostgreSQL

# Dynamic (SOCKS) proxy
ssh -D 1080 user@remote-server
```

### `scp` — Secure Copy

```bash
scp file.txt user@server:/remote/path/      # local to remote
scp user@server:/remote/file.txt /local/    # remote to local
scp -r directory/ user@server:/remote/      # recursive (directory)
scp -P 2222 file.txt user@server:/path/     # custom port
```

### `rsync` — Efficient Sync

```bash
rsync -avz /local/dir/ user@server:/remote/dir/    # sync to remote
rsync -avz --delete /local/ user@server:/remote/   # sync + delete extras
rsync -avz --progress large-file user@server:~/    # show progress
rsync -avz user@server:/remote/ /local/            # sync FROM remote
```

rsync only transfers **changed files** — much faster than scp for large dirs.

---

## 8.11 `firewall-cmd` and `ufw` — Firewall Management

### UFW (Ubuntu Firewall — Simple)

```bash
sudo ufw status                    # check firewall status
sudo ufw enable                    # enable firewall
sudo ufw disable                   # disable

sudo ufw allow 22                  # allow SSH
sudo ufw allow 80                  # allow HTTP
sudo ufw allow 443                 # allow HTTPS
sudo ufw allow 8080/tcp            # specific port/protocol

sudo ufw deny 3306                 # block MySQL from outside
sudo ufw delete allow 8080         # remove a rule

sudo ufw allow from 192.168.1.0/24 to any port 5432  # allow subnet to access postgres
```

### `iptables` (Advanced, but know it exists)

```bash
sudo iptables -L                   # list all rules
sudo iptables -L -n -v             # verbose with packet counts
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT   # allow port 80
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT   # allow SSH
sudo iptables -A INPUT -j DROP     # drop everything else (careful!)
```

---

## 8.12 Common Network Debugging Scenarios

### Scenario 1: "Cannot connect to service"

```bash
# Step 1: Is the service running?
systemctl status nginx

# Step 2: Is it listening on the right port?
ss -tlnp | grep :80

# Step 3: Is the firewall blocking it?
sudo ufw status
sudo iptables -L

# Step 4: Can you reach it locally?
curl http://localhost:80

# Step 5: Can you reach it from outside?
curl http://YOUR_PUBLIC_IP:80
```

### Scenario 2: "DNS resolution failing"

```bash
# Check if DNS works
dig google.com +short
nslookup google.com

# Check DNS server config
cat /etc/resolv.conf

# Try a different DNS server
dig @8.8.8.8 google.com

# Check /etc/hosts for overrides
cat /etc/hosts
```

### Scenario 3: "SSL/TLS certificate issue"

```bash
# Check certificate
openssl s_client -connect example.com:443 </dev/null 2>/dev/null | openssl x509 -text | grep -E "Subject:|Issuer:|Not"

# Quick cert expiry check
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

---

## Summary

| Tool | Use Case |
|------|----------|
| `ping host` | Basic connectivity test |
| `traceroute host` | Trace network path |
| `curl -I URL` | Test HTTP endpoint |
| `curl -s URL \| jq .` | API testing |
| `wget URL` | Download files |
| `ss -tlnp` | What ports are listening |
| `dig domain` | DNS lookup |
| `ssh user@host` | Connect to remote server |
| `scp file user@host:path` | Secure file copy |

---

## Knowledge Check

1. What's the difference between `curl` and `wget`?
2. How do you check which process is listening on port 3000?
3. What is the purpose of SSH key-based authentication?
4. How do you follow DNS resolution step by step?
5. What does `curl -f` do, and why is it useful in scripts?

---

## Hands-On Exercise

```bash
# 1. Test connectivity
ping -c 4 8.8.8.8
ping -c 4 google.com

# 2. Check what ports are listening
ss -tlnp
ss -tlnp | grep :22    # SSH port
ss -tlnp | grep :80    # HTTP

# 3. Test HTTP with curl
curl -I https://google.com
curl -s -o /dev/null -w "HTTP Status: %{http_code}\nTotal Time: %{time_total}s\n" https://google.com

# 4. DNS lookups
dig google.com +short
dig @8.8.8.8 google.com +short
dig google.com MX +short

# 5. Check your network interfaces
ip addr
ip route

# 6. Test a local service
sudo apt install nginx -y 2>/dev/null
sudo systemctl start nginx
curl -s http://localhost | head -5    # should see nginx HTML
ss -tlnp | grep :80                   # see nginx listening
sudo systemctl stop nginx

# 7. Generate SSH keys (if not already done)
ls ~/.ssh/ 2>/dev/null || echo "No .ssh directory"
# Only run if no keys exist:
# ssh-keygen -t ed25519 -C "practice" -f ~/.ssh/practice_key -N ""
# ls -la ~/.ssh/
```

**Challenge:** Use curl to get your public IP address (hint: there are free services for this like `ifconfig.me` or `api.ipify.org`), then run a `dig -x` reverse lookup on that IP.

---

## Further Reading

- `man curl` — exhaustive curl reference (use `curl --manual | less`)
- `man ssh_config` — SSH configuration options
- `man dig` — DNS lookup reference
- [Linux Networking Tools](https://www.redhat.com/sysadmin/beginners-guide-network-troubleshooting-linux)

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="07-process-management.md">← Previous: Process Management</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="09-shell-scripting.md">Next: Shell Scripting →</a>
</div>
