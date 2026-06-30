# Chapter 6 — Volumes & Storage

> **Docker Course — Chapter 6 of 17**

## Learning Objectives

By the end of this chapter you will be able to:

- Explain why containers need external storage mechanisms
- Distinguish between volumes, bind mounts, and tmpfs mounts
- Create, use, inspect, and delete named volumes
- Use bind mounts effectively during local development
- Configure storage in Docker Compose
- Back up and restore volume data
- Avoid the most common storage pitfalls

---

## 6.1 The Container Storage Problem

Containers are designed to be **ephemeral**: stateless, replaceable, and disposable. When you remove a container (`docker rm`), everything written to its filesystem is gone permanently. This is intentional — it is what makes containers easy to replace and scale.

But real applications need to write data that must survive:

- A PostgreSQL database storing user records
- An application uploading files to disk
- A log aggregation service writing log files
- A configuration file modified at runtime

Docker solves this with **mount points** — ways to attach external storage to a container's filesystem so that data lives outside the container lifecycle.

### The Container Filesystem Layers

A running container has a layered filesystem:

```
┌─────────────────────────────────────────┐
│   Writable container layer (ephemeral)  │  ← removed with `docker rm`
├─────────────────────────────────────────┤
│   Image layer N (read-only)             │
├─────────────────────────────────────────┤
│   Image layer ...                       │
├─────────────────────────────────────────┤
│   Image layer 1 (read-only)             │
└─────────────────────────────────────────┘
```

The writable layer uses a **copy-on-write** strategy — reading from lower layers is cheap; writing creates a new copy in the writable layer. Writing large amounts of data to this layer is slow and disappears when the container is removed.

---

## 6.2 Three Storage Options

Docker provides three types of mounts. Each has a specific purpose:

| Type | Location | Managed By | Use Case |
|------|----------|------------|----------|
| **Volume** | Docker-managed path on host | Docker | Databases, persistent app data — recommended for production |
| **Bind Mount** | Host path you specify | You | Development (live code reload), injecting config files |
| **tmpfs** | Memory only | OS kernel | Sensitive data that must not persist to disk |

```
Host filesystem
┌─────────────────────────────────────────────────────┐
│                                                     │
│  /var/lib/docker/volumes/mydata/_data   ← Volume   │
│  /home/user/myproject                  ← Bind Mount│
│  RAM                                   ← tmpfs      │
│                                                     │
└──────────────┬──────────────┬──────────────────────┘
               │              │
       ┌───────▼──────────────▼──────────┐
       │         Container               │
       │  /var/lib/postgresql/data       │  (volume)
       │  /app                           │  (bind mount)
       │  /run                           │  (tmpfs)
       └─────────────────────────────────┘
```

---

## 6.3 Volumes — Recommended for Production

**Named volumes** are the recommended way to persist data in production. Docker manages the storage location (typically under `/var/lib/docker/volumes/` on Linux). You reference them by name, not by path.

### Creating and Inspecting Volumes

```bash
# Create a named volume
docker volume create mydata

# List all volumes
docker volume ls

# Inspect a volume (shows mount point on host)
docker volume inspect mydata

# Output:
# [
#   {
#     "Name": "mydata",
#     "Driver": "local",
#     "Mountpoint": "/var/lib/docker/volumes/mydata/_data",
#     ...
#   }
# ]
```

### Using a Volume with a Container

```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -v mydata:/var/lib/postgresql/data \
  postgres:15
```

Syntax: `-v <volume-name>:<container-path>`

Docker creates the volume if it does not exist. On subsequent runs, the same volume is reattached.

### Data Survives Container Removal

```bash
# Connect to postgres, create a database and table
docker exec -it postgres psql -U postgres -c "CREATE DATABASE myapp;"
docker exec -it postgres psql -U postgres -d myapp -c "CREATE TABLE users (id serial PRIMARY KEY, name text);"
docker exec -it postgres psql -U postgres -d myapp -c "INSERT INTO users (name) VALUES ('Alice');"

# Remove the container — data is in the volume, not the container
docker rm -f postgres

# Start a brand-new container with the same volume
docker run -d \
  --name postgres2 \
  -e POSTGRES_PASSWORD=secret \
  -v mydata:/var/lib/postgresql/data \
  postgres:15

# Data is still there
docker exec -it postgres2 psql -U postgres -d myapp -c "SELECT * FROM users;"
#  id | name
# ----+-------
#   1 | Alice
```

### Removing Volumes

```bash
# Remove a specific volume (fails if a container is using it)
docker volume rm mydata

# Remove all volumes not used by any container
docker volume prune

# Remove all volumes including anonymous ones
docker volume prune -a
```

**Warning:** `docker system prune` does NOT remove named volumes by default. You must pass `--volumes` to include them.

---

## 6.4 Bind Mounts — For Development

A **bind mount** mounts a specific path from your host filesystem into the container. You control exactly which host path is used.

```bash
# Mount the current directory into the container
docker run -d \
  -v $(pwd):/app \
  -w /app \
  -p 3000:3000 \
  node:20-alpine \
  node server.js

# With explicit host and container paths
docker run -d \
  -v /home/user/myproject:/app \
  -p 3000:3000 \
  node:20-alpine \
  node server.js
```

**Why bind mounts are great for development:**

- Changes to files on your host are immediately visible inside the container — no rebuild needed
- Your editor, linter, and version control tools all work on the host
- Hot-reload tools (nodemon, uvicorn `--reload`, webpack-dev-server) detect changes instantly

**Why bind mounts are problematic for production:**

- The host path must exist — breaks portability across machines and CI environments
- File permission issues: the container user (UID 1000 inside the container) may not match your host user UID, leading to "permission denied" errors on Linux
- Bind mounts bypass Docker's volume management — no easy backup, no volume drivers

**Read-only bind mount** (inject config without allowing writes):

```bash
docker run -d \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx:alpine
```

The `:ro` suffix makes the mount read-only inside the container.

---

## 6.5 tmpfs Mounts

A **tmpfs mount** stores data in the host's memory. It is never written to disk and is cleared when the container stops.

```bash
docker run -d \
  --tmpfs /run:rw,noexec,nosuid,size=100m \
  nginx
```

Options:
- `rw` — read-write (default)
- `noexec` — no executable files (security)
- `nosuid` — disallow setuid binaries (security)
- `size=100m` — maximum size in memory

**Use cases:**

- Session tokens or one-time passwords that should never touch disk
- Temporary scratch space for sensitive computations
- High-speed temporary storage where durability is not needed

**Note:** tmpfs mounts cannot be shared between containers. Each container gets its own isolated in-memory mount.

---

## 6.6 Volume Drivers

The default volume driver is `local` — it stores data on the host filesystem. For production clusters you often need storage that is shared across multiple hosts or backed by network/cloud storage.

### NFS Volume

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=nfs-server.example.com,rw \
  --opt device=:/path/to/share \
  nfs-vol

# Use like any named volume
docker run -d \
  -v nfs-vol:/data \
  myapp
```

### Cloud Storage Drivers

| Provider | Plugin | Storage Backend |
|----------|--------|-----------------|
| AWS | `rexray/ebs` or EFS mount | EBS or EFS |
| Azure | `azurefile` | Azure Files |
| GCP | GCE persistent disk | GCS |
| General | `convoy`, `portworx` | Various |

In Kubernetes (covered in the Kubernetes course), persistent storage is handled by `PersistentVolumes` and `StorageClasses`, which abstract away the driver details.

---

## 6.7 Backing Up and Restoring Volumes

Docker volumes do not have a built-in backup mechanism. The standard pattern is to run a temporary container that accesses the volume and creates a tar archive.

### Backup a Volume

```bash
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/mydata-backup.tar.gz -C /data .
```

**What this does:**

1. Runs a temporary Alpine container (`--rm` removes it when done)
2. Mounts the volume at `/data` inside the container
3. Mounts the current host directory at `/backup`
4. Creates a compressed tarball of the volume contents into `/backup/mydata-backup.tar.gz` on your host

### Restore a Volume

```bash
# Create the volume if it doesn't exist
docker volume create mydata

# Restore from backup
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  alpine \
  tar xzf /backup/mydata-backup.tar.gz -C /data
```

### Migrate a Volume Between Hosts

```bash
# Host A: backup
docker run --rm -v mydata:/data -v $(pwd):/backup alpine tar czf /backup/mydata.tar.gz -C /data .

# Transfer the file
scp mydata.tar.gz user@host-b:/tmp/

# Host B: restore
docker volume create mydata
docker run --rm -v mydata:/data -v /tmp:/backup alpine tar xzf /backup/mydata.tar.gz -C /data
```

---

## 6.8 Storage in Docker Compose

Docker Compose makes volume management declarative. Volumes and bind mounts are configured under the `volumes` key of each service, and named volumes are declared at the top-level `volumes` section.

```yaml
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      # Named volume for database data
      - pgdata:/var/lib/postgresql/data
      # Bind mount for initialization scripts
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  app:
    image: myapp:latest
    volumes:
      # Read-only bind mount for config
      - ./config:/app/config:ro

  dev-app:
    image: node:20-alpine
    working_dir: /app
    volumes:
      # Bind mount for live development
      - .:/app
      # Anonymous volume to prevent bind mount from overwriting node_modules
      - /app/node_modules
    ports:
      - "3000:3000"
    command: ["node", "--watch", "server.js"]

# Declare named volumes here — required for volumes used by services above
volumes:
  pgdata:

  # Optional: reference an externally created volume
  shared-data:
    external: true
```

**The `node_modules` anonymous volume trick** (line `- /app/node_modules`): when you bind-mount your source code, the host's `node_modules` directory would overwrite the container's `node_modules`. The anonymous volume at `/app/node_modules` takes precedence, preserving the container's installed modules.

### Common Compose Volume Patterns

```yaml
volumes:
  # Default — Docker manages location
  pgdata:

  # Explicit driver options
  nfs-volume:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=nfs.example.com,rw"
      device: ":/shared"

  # External volume (must exist before compose up)
  existing-data:
    external: true
    name: my-existing-volume
```

---

## 6.9 Common Volume Mistakes

### 1. Not Using Volumes for Database Containers

```bash
# Bad — database data lives in the writable container layer
docker run -d postgres:15

# Good — named volume ensures data persists
docker run -d -v pgdata:/var/lib/postgresql/data postgres:15
```

If you run `docker-compose down` without `-v`, named volumes are preserved. But if you run `docker rm` on the container without having used a volume, all your database data is gone.

### 2. Using Bind Mounts in Production

Bind mounts tie your container to a specific host path. If the host path does not exist or the path differs between deployment environments, the container fails to start.

### 3. Running `docker system prune` Without Knowing What It Removes

`docker system prune` removes stopped containers, dangling images, unused networks, and — if you pass `--volumes` — unused volumes. Always check what volumes exist before running prune in production:

```bash
# Check first
docker volume ls

# Then prune only unused volumes
docker volume prune
```

### 4. UID Mismatch with Bind Mounts on Linux

If your container runs as UID 1000 (a non-root user you created) but your host files are owned by UID 1001, writes to the bind mount will fail with "permission denied".

**Solutions:**

```bash
# Option 1: Match the container UID to your host UID
docker run -u $(id -u):$(id -g) -v $(pwd):/app myimage

# Option 2: In Dockerfile, create user with specific UID
RUN adduser --uid 1001 --disabled-password appuser

# Option 3: Grant write permissions on the host (less secure)
chmod 777 $(pwd)/data
```

### 5. Forgetting to Back Up Volumes Before `docker compose down -v`

The `-v` flag on `docker compose down` removes all named volumes defined in the compose file. This is destructive and irreversible. Always back up before using this flag.

---

## Summary

- Containers are ephemeral — data written to the container layer is lost when the container is removed
- **Named volumes** (managed by Docker) are the recommended solution for production data persistence
- **Bind mounts** (host path you specify) are ideal for development — live code reloading, config injection
- **tmpfs mounts** live only in memory — use for sensitive data that must never touch disk
- Volumes survive container removal and can be shared between containers
- Backing up volumes requires running a temporary container that tars the volume content
- In Docker Compose, declare named volumes at the top level and reference them in service `volumes` keys
- Avoid `docker system prune --volumes` in production without checking what you are about to delete
- On Linux, watch for UID mismatches between the container user and host file ownership when using bind mounts

---

## Knowledge Check

1. A developer runs `docker rm -f mydb` on their PostgreSQL container and loses all their data. What should they have done differently?
2. What is the syntax difference between a named volume and a bind mount in `docker run -v`?
3. When would you use a tmpfs mount instead of a volume?
4. You are developing a Node.js app and want code changes on your host to be immediately visible inside the container. Which mount type should you use, and what is the command?
5. In Docker Compose, you add a service that uses a named volume `pgdata`. Where must you also declare `pgdata`, and why?

---

## Hands-on Exercise

**Goal:** Prove that named volumes persist data across container lifecycle; practice backup and restore.

**Part 1 — Volume Persistence**

1. Create a named volume: `docker volume create pgdata`
2. Start a PostgreSQL container using the volume:
   ```bash
   docker run -d --name pg1 \
     -e POSTGRES_PASSWORD=secret \
     -v pgdata:/var/lib/postgresql/data \
     postgres:15
   ```
3. Create a database and insert a row:
   ```bash
   docker exec -it pg1 psql -U postgres -c "CREATE DATABASE demo;"
   docker exec -it pg1 psql -U postgres -d demo -c "CREATE TABLE notes (id serial PRIMARY KEY, note text);"
   docker exec -it pg1 psql -U postgres -d demo -c "INSERT INTO notes (note) VALUES ('this survives the container');"
   ```
4. Remove the container completely: `docker rm -f pg1`
5. Start a new container with the same volume:
   ```bash
   docker run -d --name pg2 \
     -e POSTGRES_PASSWORD=secret \
     -v pgdata:/var/lib/postgresql/data \
     postgres:15
   ```
6. Verify the data is still there:
   ```bash
   docker exec -it pg2 psql -U postgres -d demo -c "SELECT * FROM notes;"
   ```

**Part 2 — Backup and Restore**

1. Back up the `pgdata` volume to a tar file:
   ```bash
   docker run --rm \
     -v pgdata:/data \
     -v $(pwd):/backup \
     alpine \
     tar czf /backup/pgdata-backup.tar.gz -C /data .
   ```
2. Remove the volume: `docker rm -f pg2 && docker volume rm pgdata`
3. Create a new volume and restore: 
   ```bash
   docker volume create pgdata
   docker run --rm \
     -v pgdata:/data \
     -v $(pwd):/backup \
     alpine \
     tar xzf /backup/pgdata-backup.tar.gz -C /data
   ```
4. Start postgres again with the restored volume and verify data is intact

**Part 3 — Bind Mount for Development**

1. Create a simple `index.html` file in your current directory
2. Run nginx with a bind mount:
   ```bash
   docker run -d -p 8080:80 \
     -v $(pwd):/usr/share/nginx/html:ro \
     nginx:alpine
   ```
3. Open `http://localhost:8080` in your browser
4. Edit `index.html` on your host and refresh — observe changes appear immediately without rebuilding

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="05-multi-stage-builds.md">← Previous: Multi-stage Builds</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="07-networking.md">Next: Docker Networking →</a>
</div>
