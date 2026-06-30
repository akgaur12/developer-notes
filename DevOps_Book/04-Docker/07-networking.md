# Chapter 7 — Docker Networking

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain Docker's network isolation model and why it exists
- Describe the five built-in network drivers and when to use each
- Create user-defined bridge networks and leverage automatic DNS resolution
- Publish container ports and control which host interfaces they bind to
- Use host networking mode and understand its trade-offs
- Diagnose common network connectivity issues between containers

---

## 7.1 The Networking Challenge

Containers are isolated by default — each container runs in its own **network namespace**, meaning it has its own network interfaces, routing table, and iptables rules, completely separate from the host and from other containers.

This isolation is a security feature, but it creates several communication requirements you must solve:

| Need | Example |
|------|---------|
| Container-to-container | API container talking to database container |
| Container-to-host | Container accessing a service running on the host |
| Container-to-internet | Container downloading packages or calling external APIs |
| External-to-container | Browser reaching a web app running in a container |

Docker networking bridges all of these scenarios through its pluggable **network driver** system.

---

## 7.2 Default Network Drivers

| Driver | Description | Use Case |
|--------|-------------|----------|
| `bridge` | Default; containers on the same bridge can communicate | Single-host container communication |
| `host` | Container shares the host network stack | High performance; no port mapping needed |
| `overlay` | Multi-host networking (Swarm/Kubernetes) | Distributed apps across multiple Docker hosts |
| `macvlan` | Container gets its own MAC/IP on the LAN | Legacy apps needing direct LAN access |
| `none` | No networking at all | Maximum isolation |

The `bridge` driver is what you will use most often for local development and single-host deployments.

---

## 7.3 Bridge Networks Deep Dive

When Docker is installed, it creates a default bridge network called `bridge` (backed by the `docker0` interface on Linux).

```bash
# List all networks
docker network ls

# Inspect the default bridge network
docker network inspect bridge

# Run two containers on the default bridge
docker run -d --name c1 alpine sleep 999
docker run -d --name c2 alpine sleep 999

# On the default bridge, containers can only reach each other by IP, not by name
docker inspect c1 | grep IPAddress    # e.g., 172.17.0.2
docker exec c2 ping 172.17.0.2        # works
docker exec c2 ping c1                # FAILS on default bridge
```

This DNS limitation on the default bridge is a key reason to always create **user-defined networks** instead.

---

## 7.4 User-Defined Bridge Networks (Recommended)

User-defined bridge networks add automatic DNS resolution — containers can reach each other by name, not just IP address.

```bash
# Create a custom network
docker network create myapp-net

# Run containers on the custom network
docker run -d --name db --network myapp-net \
  -e POSTGRES_PASSWORD=secret postgres:15

docker run -d --name api --network myapp-net myapi:latest

# DNS resolution works by container name!
docker exec api ping db               # works!
docker exec api curl http://db:5432   # works!

# Connect an already-running container to the network
docker network connect myapp-net existing-container

# Disconnect a container from the network
docker network disconnect myapp-net existing-container
```

**Network topology:**

```
[myapp-net bridge: 172.20.0.0/16]
         │
   ┌─────┴──────┐
   ▼            ▼
[db:172.20.0.2] [api:172.20.0.3]
```

Both containers share the `myapp-net` bridge. Docker's embedded DNS server handles name resolution, so `api` can always reach `db` by hostname — even if the IP address changes when the container restarts.

---

## 7.5 Port Publishing

Containers are not reachable from the host by default. You must explicitly **publish** a port to expose it.

```bash
# -p host_port:container_port
docker run -d -p 8080:80 nginx                    # bind on all host interfaces
docker run -d -p 127.0.0.1:8080:80 nginx          # localhost only (more secure)
docker run -d -p 80 nginx                         # Docker picks a random host port
docker run -d -P nginx                            # publish all EXPOSE'd ports automatically

# Inspect port mappings
docker port my-nginx
docker ps   # shows the Ports column
```

Binding to `127.0.0.1` rather than all interfaces is a simple hardening step — it means the port is not reachable from external machines without an explicit reverse proxy in front.

---

## 7.6 Host Network Mode

```bash
docker run -d --network host nginx
# nginx binds directly to host port 80 — no port mapping needed or possible
```

Advantages:
- No NAT overhead — marginally better throughput
- Useful when a container needs to bind to many ports dynamically

Disadvantages:
- No network isolation — the container shares the host's full network stack
- Port conflicts are possible
- Not supported on Docker Desktop for Mac/Windows (Linux-only feature)

Use host networking only when you have a specific performance or compatibility requirement, not as a default.

---

## 7.7 Container DNS

Docker injects an embedded DNS server at `127.0.0.11` into every container on a user-defined network. This server:

- Resolves container names to their current IPs within the same network
- Forwards all other queries to the host's resolver (from `/etc/resolv.conf`)

```bash
# Override DNS servers for a container
docker run -d --dns 8.8.8.8 --dns 1.1.1.1 myapp

# Inspect DNS config inside the container
docker exec myapp cat /etc/resolv.conf

# Test resolution
docker exec myapp nslookup google.com
docker exec myapp nslookup db          # resolves sibling containers on same network
```

---

## 7.8 Network Troubleshooting

```bash
# Inspect full network configuration
docker network inspect myapp-net

# Test connectivity between containers
docker exec api ping db
docker exec api curl -v http://db:5432
docker exec api nc -zv db 5432         # netcat port check

# View container's network interfaces and routes
docker exec myapp ip addr
docker exec myapp ip route

# Verify the container can reach the internet
docker run --rm alpine ping -c 3 8.8.8.8

# Verify port mapping on the host
docker port my-container
ss -tlnp | grep 8080
```

**Common issues and fixes:**

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `ping c1` fails from `c2` | Using default bridge network | Switch to user-defined network |
| Container can't reach internet | Host firewall or IP forwarding disabled | Check `sysctl net.ipv4.ip_forward` |
| Can't connect to container from host | Port not published | Add `-p host:container` |
| Port already in use | Another process owns the host port | Change host port or stop conflicting process |

---

## Summary

- Docker containers are network-isolated by default via Linux network namespaces.
- Five built-in drivers cover most scenarios: `bridge`, `host`, `overlay`, `macvlan`, and `none`.
- The **default bridge** network lacks DNS resolution by container name — always prefer user-defined bridge networks.
- User-defined networks provide automatic DNS, better isolation, and dynamic connect/disconnect.
- Port publishing (`-p`) bridges the container's network namespace to the host, with optional interface binding for security.
- Docker's embedded DNS server at `127.0.0.11` handles name resolution within a network.

---

## Knowledge Check

1. Why does `ping c1` fail from another container on the **default** bridge network, but succeed on a user-defined bridge network?
2. What is the difference between `-p 8080:80` and `-p 127.0.0.1:8080:80`? When would you prefer the second form?
3. A container on `myapp-net` tries to reach `http://db:5432` but gets "host not found." What are the two most likely causes?
4. When would you use `--network host`? What do you lose by doing so?
5. What IP address does Docker's embedded DNS server run on, and which networks support it?

---

## Hands-on Exercise

**Goal:** Experience user-defined networking and DNS resolution firsthand.

1. Create a user-defined bridge network called `lab-net`.
2. Start a PostgreSQL container (`postgres:15`) named `db` on `lab-net` (set `POSTGRES_PASSWORD`).
3. Start an `alpine` container named `client` on `lab-net`.
4. From inside `client`, run `ping db` and `nslookup db` — confirm DNS resolution works.
5. Start a second `alpine` container named `client2` on the **default** bridge network (no `--network` flag).
6. From inside `client2`, attempt `ping db` — confirm it fails and explain why.
7. Run `nginx` with `-p 127.0.0.1:8080:80` and verify you can reach it from the host with `curl http://localhost:8080` but not from another machine on your LAN.
8. Clean up: `docker rm -f db client client2` and `docker network rm lab-net`.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="06-volumes-and-storage.md">← Previous: Volumes & Storage</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="08-docker-compose.md">Next: Docker Compose →</a>
</div>
