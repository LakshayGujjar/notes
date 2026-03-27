# 🐳 Docker Comprehensive Cheatsheet

> A complete reference for Docker — covering architecture, Dockerfiles, multi-stage builds, BuildKit, networking, volumes, Compose, Swarm, security, and real-world production patterns.

---

## 📚 Table of Contents

1. [Core Concepts & Architecture](#1-core-concepts--architecture)
2. [Installation & Configuration](#2-installation--configuration)
3. [Images](#3-images)
4. [Dockerfile — Complete Reference](#4-dockerfile--complete-reference)
5. [Multi-stage Builds](#5-multi-stage-builds)
6. [BuildKit & Advanced Building](#6-buildkit--advanced-building)
7. [Containers — Running & Managing](#7-containers--running--managing)
8. [Networking](#8-networking)
9. [Volumes & Storage](#9-volumes--storage)
10. [Docker Compose](#10-docker-compose)
11. [Docker Registry & Image Distribution](#11-docker-registry--image-distribution)
12. [Security](#12-security)
13. [Docker Swarm](#13-docker-swarm)
14. [Production Patterns & Best Practices](#14-production-patterns--best-practices)
15. [Observability & Debugging](#15-observability--debugging)
16. [Real-World Patterns & Recipes](#16-real-world-patterns--recipes)

---

## 1. Core Concepts & Architecture

### Containerisation vs Virtualisation

```
Virtual Machines:                    Containers:
┌──────────────────────────┐        ┌──────────────────────────┐
│  App A  │  App B         │        │  App A  │  App B         │
│─────────│────────────────│        │─────────│────────────────│
│  Guest  │  Guest OS      │        │  Libs   │  Libs          │
│  OS     │                │        │─────────────────────────-│
│─────────────────────────-│        │     Container Runtime    │
│     Hypervisor           │        │     (containerd/runc)    │
│─────────────────────────-│        │─────────────────────────-│
│     Host OS              │        │     Host OS (kernel)     │
│─────────────────────────-│        │─────────────────────────-│
│     Hardware             │        │     Hardware             │
└──────────────────────────┘        └──────────────────────────┘
  Full OS per VM — heavy               Shared kernel — lightweight
  Minutes to start                     Milliseconds to start
  GBs of RAM                           MBs of RAM
```

**Containers** share the host kernel but isolate processes using Linux namespaces and cgroups. They are not VMs — a container is just a process with restricted visibility and resource limits.

**OCI (Open Container Initiative)** — vendor-neutral standards for container image format (`image-spec`) and runtime (`runtime-spec`). Docker images are OCI-compliant and run on any OCI runtime (containerd, CRI-O, Podman).

---

### Docker Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Docker CLI (docker)                                        │
│  docker build / run / push / pull ...                       │
└──────────────────────┬──────────────────────────────────────┘
                       │ REST API (unix:///var/run/docker.sock)
┌──────────────────────▼──────────────────────────────────────┐
│  Docker Daemon (dockerd)                                    │
│  Image management, API server, build orchestration          │
└──────────────────────┬──────────────────────────────────────┘
                       │ gRPC
┌──────────────────────▼──────────────────────────────────────┐
│  containerd                                                 │
│  Container lifecycle, image storage, snapshots              │
└──────────────────────┬──────────────────────────────────────┘
                       │ OCI runtime spec
┌──────────────────────▼──────────────────────────────────────┐
│  runc                                                       │
│  Spawns containers using kernel namespaces + cgroups        │
└─────────────────────────────────────────────────────────────┘
```

| Component | Responsibility |
|-----------|---------------|
| `docker` CLI | User-facing commands; calls Docker daemon API |
| `dockerd` | Daemon managing images, containers, volumes, networks |
| `containerd` | Container lifecycle manager; image store |
| `runc` | Low-level OCI container runner; calls kernel APIs |
| Docker Desktop | GUI + VM layer for Mac/Windows; wraps Docker Engine |

---

### Images vs Containers

```
Image (read-only layers):          Container (image + writable layer):
┌────────────────────┐             ┌────────────────────┐
│  Layer 3: App code │             │  Writable layer ←──── your changes
│  Layer 2: npm deps │    run →    │  Layer 3: App code │  (copy-on-write)
│  Layer 1: node:20  │             │  Layer 2: npm deps │
│  Layer 0: scratch  │             │  Layer 1: node:20  │
└────────────────────┘             │  Layer 0: scratch  │
  Image ID: sha256:abc…            └────────────────────┘
  Immutable — shared between         Mutable — unique per container
  all containers using it            Deleted when container is removed
```

**Copy-on-write (CoW):** When a container modifies a file from the image, the file is copied to the writable layer first. The original image layer is never changed — multiple containers share the same read-only image layers.

---

### Container Lifecycle

```
           docker create
  ──────────────────────────→  CREATED
                                  │
                    docker start  │
                  ◄───────────────┘
                  │
               RUNNING ──────────────────────────→  PAUSED
                  │        docker pause             (SIGSTOP)
                  │                    docker unpause ◄─────┘
                  │  docker stop (SIGTERM → SIGKILL after timeout)
                  ▼
               STOPPED / EXITED
                  │
        docker rm │
                  ▼
               REMOVED (gone)
```

---

### Namespaces — Process Isolation

| Namespace | Isolates | Docker use |
|-----------|----------|-----------|
| **PID** | Process IDs | Container sees its own PID 1 (not host PIDs) |
| **NET** | Network interfaces, routes, ports | Each container gets its own `eth0` |
| **MNT** | Filesystem mount points | Container has its own `/` filesystem view |
| **UTS** | Hostname and NIS domain | Container has its own hostname |
| **IPC** | System V IPC, POSIX message queues | Isolated shared memory |
| **USER** | UIDs and GIDs | Map container root (UID 0) to unprivileged host UID |

---

### cgroups — Resource Enforcement

```bash
# cgroups (control groups) limit what a container CAN USE
# Linux kernel feature — Docker configures them per container

# View a running container's cgroup limits
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_quota_us

# Docker sets these automatically via --memory and --cpus flags
docker run --memory=512m --cpus=1.5 nginx
```

---

### Union Filesystems — OverlayFS

```
OverlayFS layer structure (docker overlay2):

upperdir (writable)  ← container writes go here
─────────────────────────────────────────────
lowerdir[2]          ← image layer 3 (app)
lowerdir[1]          ← image layer 2 (deps)
lowerdir[0]          ← image layer 1 (base OS)
─────────────────────────────────────────────
merged               ← what the container sees (unified view)

On READ:  kernel returns from highest layer containing the file
On WRITE: file is copied from lowerdir to upperdir first (copy-on-write)
On DELETE: a "whiteout" file is created in upperdir to hide lowerdir file
```

---

### Docker Context

```bash
# Context = a named connection to a Docker endpoint
docker context ls                           # list all contexts
docker context use my-remote-server         # switch active context
docker context create remote \              # create context for remote daemon
  --docker "host=ssh://user@server"
docker context inspect default             # show context details

# Current context affects ALL docker commands
docker ps                                  # lists containers on active context
```

---

## 2. Installation & Configuration

### Ubuntu/Debian Installation

```bash
# Remove old versions
sudo apt-get remove docker docker-engine docker.io containerd runc

# Install prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# Post-install: add your user to docker group (avoid sudo for docker)
sudo usermod -aG docker $USER
newgrp docker                              # apply group change without logout
```

> ⚠️ **Warning:** Adding a user to the `docker` group grants root-equivalent privileges on the host (the docker daemon runs as root). Treat `docker` group membership like `sudo` access.

---

### daemon.json — Daemon Configuration

```json
// /etc/docker/daemon.json — Docker daemon configuration
{
  "data-root": "/mnt/docker-data",        // custom Docker data directory (default: /var/lib/docker)
  "storage-driver": "overlay2",           // filesystem driver (overlay2 is default and recommended)

  "log-driver": "json-file",              // default log driver for all containers
  "log-opts": {
    "max-size": "10m",                    // rotate logs at 10MB
    "max-file": "3"                       // keep last 3 log files
  },

  "insecure-registries": [               // allow HTTP (not HTTPS) for these registries
    "my-registry.internal:5000"
  ],

  "registry-mirrors": [                  // pull-through cache / mirror
    "https://mirror.example.com"
  ],

  "default-address-pools": [             // IP ranges for new networks
    {"base": "172.17.0.0/12", "size": 24}
  ],

  "dns": ["8.8.8.8", "8.8.4.4"],        // custom DNS for all containers

  "live-restore": true,                  // keep containers running during daemon restart

  "experimental": false,                 // enable experimental features

  "builder": {
    "gc": {
      "enabled": true,
      "defaultKeepStorage": "20GB"        // BuildKit cache size limit
    }
  }
}
```

```bash
# Apply daemon.json changes
sudo systemctl reload docker    # or: sudo systemctl restart docker
```

---

### Rootless Docker

```bash
# Install rootless Docker (no daemon running as root)
curl -fsSL https://get.docker.com/rootless | sh

# Configure environment
export PATH=/usr/bin:$PATH
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock

# Start rootless daemon
systemctl --user start docker
systemctl --user enable docker    # start on login

# Limitations of rootless:
# ❌ Cannot bind ports < 1024 (use --net=host or sysctl)
# ❌ No overlay2 storage driver on some filesystems
# ❌ Some cgroup v2 features unavailable
# ✅ Container root cannot escape to host root
```

---

## 3. Images

### Image Naming

```
registry/repository:tag@digest
│          │          │   │
│          │          │   └── sha256:abc123... (content hash)
│          │          └────── v1.2.3, latest, main (mutable label)
│          └───────────────── nginx, library/nginx, myorg/myapp
└──────────────────────────── docker.io (default), ghcr.io, 123.dkr.ecr.us-east-1.amazonaws.com
```

```bash
# Pull by tag (mutable — may change!)
docker pull nginx:1.25                    # specific version tag
docker pull nginx:latest                  # latest — avoid in production!

# Pull by digest (immutable — always the same image)
docker pull nginx@sha256:a3e0f04...       # pinned to exact content hash

# Pull for specific platform
docker pull --platform linux/arm64 nginx:1.25

# Fully qualified pull (explicit registry)
docker pull registry.example.com/myapp:v1.2.3
```

> 🔴 **Anti-pattern:** Using `latest` tag in production. `latest` is a mutable pointer — the image it refers to changes silently when new versions are pushed. Always pin to a specific version tag or digest.

---

### Inspecting Images

```bash
# List local images
docker images                              # basic list
docker image ls --filter dangling=true    # show untagged (<none>) images
docker image ls --filter "label=version=1.0"
docker image ls --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Inspect image metadata
docker inspect nginx:1.25                  # full JSON metadata
docker inspect --format='{{.Config.Entrypoint}}' nginx:1.25
docker inspect --format='{{range .RootFS.Layers}}{{.}}\n{{end}}' nginx:1.25

# View layer history (shows commands that built each layer)
docker history nginx:1.25
docker history --no-trunc nginx:1.25      # show full commands (not truncated)
docker history --format "{{.Size}}\t{{.CreatedBy}}" nginx:1.25
```

---

### Image Scanning

```bash
# docker scout — built-in scanning (Docker Desktop / Hub)
docker scout cves nginx:1.25              # list CVEs in image
docker scout recommendations nginx:1.25  # suggest better base images
docker scout compare myapp:v1 myapp:v2   # compare vulnerability profiles

# trivy — standalone scanner
trivy image nginx:1.25                    # full vulnerability scan
trivy image --severity HIGH,CRITICAL nginx:1.25   # only critical issues
trivy image --format json nginx:1.25 > report.json

# Generate SBOM (Software Bill of Materials)
docker buildx build --sbom=true --output type=local,dest=. .
trivy image --format cyclonedx myapp:latest > sbom.json
```

---

### Saving, Loading, Exporting

```bash
# docker save — save IMAGE to tar (includes all layers and metadata)
docker save nginx:1.25 -o nginx.tar           # save single image
docker save nginx:1.25 redis:7 | gzip > images.tar.gz  # multiple images

# docker load — load image from tar (restores tags, layers)
docker load -i nginx.tar
docker load < images.tar.gz

# docker export — export CONTAINER filesystem to tar (flat, no layers/history)
docker export my-container -o container.tar   # just the merged filesystem

# docker import — create IMAGE from flat filesystem tar
docker import container.tar myapp:imported

# Key difference:
# save/load → preserves image layers, history, metadata → use for image transfer
# export/import → flat filesystem snapshot → smaller but loses history/layers
```

---

## 4. Dockerfile — Complete Reference

### Syntax and .dockerignore

```dockerfile
# syntax=docker/dockerfile:1          ← parser directive: must be FIRST line
# (enables latest BuildKit features)

# Comments start with #
# Line continuation with backslash:
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

```
# .dockerignore — files excluded from build context (like .gitignore)
.git/                    # never send git history (large, unnecessary)
.gitignore
node_modules/            # dependencies built in container, not copied from host
**/*.test.js             # test files not needed in production image
**/*.md                  # documentation
.env                     # ⚠️ never send .env files — contain secrets!
Dockerfile*              # Dockerfiles themselves
docker-compose*.yml
.DS_Store
coverage/
dist/                    # if built inside container
```

> 💡 **Tip:** Build context is the entire directory sent to the daemon. A missing or minimal `.dockerignore` makes builds slow and may leak secrets. Always create `.dockerignore` before your first `docker build`.

---

### FROM

```dockerfile
# Simple FROM
FROM ubuntu:22.04

# FROM with platform (for multi-arch builds)
FROM --platform=linux/amd64 ubuntu:22.04

# Named stage (for multi-stage builds)
FROM node:20-alpine AS builder

# Minimal base — no OS at all (for static binaries)
FROM scratch

# Reference another stage from the same Dockerfile
COPY --from=builder /app/dist /usr/share/nginx/html
```

---

### RUN

```dockerfile
# Shell form — runs via /bin/sh -c (has shell features but creates shell process)
RUN apt-get update && apt-get install -y curl

# Exec form — no shell, runs directly (preferred for clarity)
RUN ["apt-get", "install", "-y", "curl"]

# ✅ Best practice: combine commands to reduce layers
# Use set -eux: e=exit on error, u=unset var is error, x=print commands
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        curl \
        ca-certificates; \
    rm -rf /var/lib/apt/lists/*   # clean apt cache to reduce layer size

# BuildKit: cache mount (package cache persists between builds)
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y curl

# BuildKit: secret mount (inject secret without baking into layer)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install    # .npmrc with auth token never stored in image!

# BuildKit: SSH forwarding (access private repos during build)
RUN --mount=type=ssh \
    git clone git@github.com:org/private-repo.git

# BuildKit: network=none (verify offline installability)
RUN --network=none pip install --no-index --find-links /wheels -r requirements.txt
```

---

### COPY and ADD

```dockerfile
# COPY — preferred for copying local files
COPY src/ /app/src/               # copy directory
COPY package*.json /app/          # glob pattern (copies package.json AND package-lock.json)
COPY --chown=node:node . /app/    # set ownership on copy
COPY --chmod=755 entrypoint.sh /  # set permissions on copy

# BuildKit: --link flag (faster, better cache)
COPY --link package.json /app/

# Copy from another build stage
COPY --from=builder /app/dist /usr/share/nginx/html
COPY --from=golang:1.22 /usr/local/go /usr/local/go  # copy from external image

# ADD — use only for these specific cases:
ADD archive.tar.gz /app/          # auto-extracts archives (COPY does not!)
ADD https://example.com/file.txt /tmp/  # fetch from URL (but: use curl in RUN instead)

# 🔴 Anti-pattern: ADD for plain file copy
ADD config.json /app/config.json  # use COPY instead — ADD has surprising behaviours
```

---

### ENV and ARG

```dockerfile
# ENV — runtime environment variable (persists in container)
ENV NODE_ENV=production                    # single variable
ENV APP_PORT=3000 \                        # multiple variables
    APP_HOST=0.0.0.0

# ARG — build-time variable (NOT available at runtime unless set via ENV)
ARG VERSION=1.0.0                         # default value
ARG BUILD_DATE                            # no default — must be passed via --build-arg

# ARG before FROM is available only in FROM
ARG BASE_IMAGE=node:20-alpine
FROM ${BASE_IMAGE}

# ARG after FROM — available in RUN, COPY, etc.
ARG VERSION
RUN echo "Building version ${VERSION}"

# Promote ARG to ENV so it's available at runtime
ARG APP_VERSION
ENV APP_VERSION=${APP_VERSION}

# ⚠️ ARG values are visible in docker history — don't use for secrets!
ARG API_KEY=secret123   # visible to anyone with access to the image!
```

```bash
# Pass build args at build time
docker build --build-arg VERSION=2.0.0 --build-arg BUILD_DATE=$(date -I) .
```

---

### WORKDIR, EXPOSE, LABEL

```dockerfile
# WORKDIR — sets working directory for subsequent RUN/COPY/CMD/ENTRYPOINT
WORKDIR /app                    # created if doesn't exist
WORKDIR /app/src                # can stack — creates /app/src
# Avoids: RUN cd /app && ... (doesn't persist across layers)

# EXPOSE — documents the port (does NOT publish it!)
EXPOSE 3000                     # TCP by default
EXPOSE 53/udp                   # explicit UDP
EXPOSE 80 443                   # multiple ports

# LABEL — metadata (OCI standard annotations)
LABEL org.opencontainers.image.title="My App" \
      org.opencontainers.image.version="1.2.3" \
      org.opencontainers.image.source="https://github.com/org/repo" \
      org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      maintainer="team@example.com"
```

---

### ENTRYPOINT and CMD

```dockerfile
# ENTRYPOINT — the executable; not easily overridden
# CMD — default arguments to ENTRYPOINT (or default command if no ENTRYPOINT)

# The four combinations:
# ─────────────────────────────────────────────────────────────────
# 1. Only CMD (shell form) — runs via /bin/sh -c, PID 1 is shell
CMD echo "hello"                  # ❌ can't receive signals properly

# 2. Only CMD (exec form) — runs directly, IS PID 1
CMD ["node", "server.js"]         # ✅ PID 1 = node; receives SIGTERM

# 3. ENTRYPOINT + CMD (exec form) — best practice
ENTRYPOINT ["nginx", "-g"]        # fixed executable
CMD ["daemon off;"]               # default arg — easily overridden at runtime

# 4. ENTRYPOINT (shell form) — ❌ don't use
ENTRYPOINT nginx -g "daemon off;" # shell becomes PID 1 — signals lost!
# ─────────────────────────────────────────────────────────────────

# Override at runtime:
# docker run myimage --other-args          # replaces CMD, keeps ENTRYPOINT
# docker run --entrypoint sh myimage       # replaces ENTRYPOINT
```

```
Signal flow — why exec form matters:

Shell form:  docker run → /bin/sh -c → your_process
                                          ↑
             SIGTERM → /bin/sh (catches it, may not forward!)
             your_process may never receive SIGTERM → gets SIGKILL after timeout

Exec form:   docker run → your_process (PID 1)
                              ↑
             SIGTERM → your_process (receives directly, can graceful shutdown)
```

---

### USER and HEALTHCHECK

```dockerfile
# USER — run as non-root user
# Create the user first
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --shell /bin/sh --create-home appuser

USER appuser                      # all subsequent RUN, CMD, ENTRYPOINT run as this user

# Or use numeric UID directly
USER 1001:1001                    # uid:gid

# HEALTHCHECK — instruct Docker to test if container is healthy
HEALTHCHECK --interval=30s \      # check every 30 seconds
            --timeout=5s \        # timeout for the check command
            --start-period=10s \  # grace period before first check
            --retries=3 \         # fail after 3 consecutive failures
  CMD curl -f http://localhost:3000/healthz || exit 1

# Or use wget (smaller than curl for Alpine)
HEALTHCHECK CMD wget -qO- http://localhost:3000/healthz || exit 1

# Disable health check inherited from base image
HEALTHCHECK NONE
```

---

### HEREDOC Syntax (BuildKit)

```dockerfile
# syntax=docker/dockerfile:1

# Multi-line RUN with heredoc (cleaner than && \)
RUN <<EOF
set -eux
apt-get update
apt-get install -y curl git
rm -rf /var/lib/apt/lists/*
EOF

# COPY a file inline (no need for a file on disk)
COPY <<EOF /app/config.json
{
  "port": 3000,
  "env": "production"
}
EOF

# Multiple commands with different shells
RUN <<PYTHON python3
import os
print("Building with Python!")
PYTHON
```

---

## 5. Multi-stage Builds

### The Problem Multi-stage Solves

```
Without multi-stage:               With multi-stage:
┌──────────────────────┐           ┌──────────────────────┐
│  Final image         │           │  Stage 1: builder    │
│  ├── GCC compiler    │           │  ├── GCC compiler    │
│  ├── Build tools     │    ────►  │  ├── Build tools     │
│  ├── Source code     │           │  ├── Source code     │
│  └── Binary          │           │  └── Binary          │
│                      │           └──────────────────────┘
│  Size: 1.2 GB        │                     │ COPY --from=builder /binary
└──────────────────────┘           ┌──────────▼───────────┐
                                   │  Stage 2: runtime    │
                                   │  └── Binary only     │
                                   │                      │
                                   │  Size: 12 MB         │
                                   └──────────────────────┘
```

---

### Go — Static Binary to Scratch

```dockerfile
# syntax=docker/dockerfile:1

# ── Stage 1: build ──────────────────────────────────────────
FROM golang:1.22-alpine AS builder

WORKDIR /src
COPY go.mod go.sum ./           # copy dependency files first (cache optimization)
RUN go mod download             # download deps (cached if go.mod unchanged)
COPY . .                        # copy source code

RUN CGO_ENABLED=0 GOOS=linux \  # static binary (no libc dependency)
    go build -trimpath \         # remove build paths from binary
    -ldflags="-s -w" \           # strip debug info (smaller binary)
    -o /app/server ./cmd/server

# ── Stage 2: final (scratch = zero OS) ──────────────────────
FROM scratch                    # no OS, no shell, no nothing

COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/  # TLS certs
COPY --from=builder /app/server /server

USER 65534:65534                # nobody user (doesn't exist but uid is valid)
EXPOSE 8080
ENTRYPOINT ["/server"]
```

---

### Node.js — Multi-stage

```dockerfile
# syntax=docker/dockerfile:1

# ── Stage 1: dependencies ────────────────────────────────────
FROM node:20-alpine AS deps

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production       # install only production deps

# ── Stage 2: builder ─────────────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci                          # all deps including devDependencies
COPY . .
RUN npm run build                   # compile TypeScript / bundle

# ── Stage 3: runner (production) ─────────────────────────────
FROM node:20-alpine AS runner

RUN addgroup -S appgroup && adduser -S appuser -G appgroup   # non-root user
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules  # prod deps only
COPY --from=builder /app/dist ./dist               # compiled output
COPY --from=builder /app/package.json .

USER appuser
EXPOSE 3000
HEALTHCHECK CMD wget -qO- http://localhost:3000/healthz || exit 1
ENTRYPOINT ["node", "dist/server.js"]
```

---

### Java — Spring Boot with Layered JAR

```dockerfile
# syntax=docker/dockerfile:1

# ── Stage 1: build with Maven ────────────────────────────────
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /src
COPY pom.xml .                        # copy POM first for dep caching
RUN mvn dependency:go-offline -q      # download all deps (cached layer)
COPY src/ ./src/
RUN mvn package -DskipTests -q        # build JAR

# Extract Spring Boot layered JAR for optimal Docker caching
RUN java -Djarmode=layertools -jar target/*.jar extract --destination /layers

# ── Stage 2: production runtime ──────────────────────────────
FROM eclipse-temurin:21-jre-alpine    # JRE only — not JDK (smaller)

RUN addgroup -S spring && adduser -S spring -G spring
WORKDIR /app

# Copy layers in order of change frequency (least → most frequent)
COPY --from=builder /layers/dependencies ./          # rarely changes
COPY --from=builder /layers/spring-boot-loader ./    # rarely changes
COPY --from=builder /layers/snapshot-dependencies ./  # occasionally
COPY --from=builder /layers/application ./            # changes most often

USER spring
EXPOSE 8080

# JVM memory settings for containers
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]
```

---

### Python — Multi-stage with Poetry

```dockerfile
# syntax=docker/dockerfile:1

# ── Stage 1: export requirements ─────────────────────────────
FROM python:3.12-slim AS deps-export

RUN pip install poetry==1.8.2          # pin Poetry version
COPY pyproject.toml poetry.lock ./
RUN poetry export --without-hashes \   # convert to requirements.txt
    --format=requirements.txt \
    -o /requirements.txt

# ── Stage 2: build wheels ─────────────────────────────────────
FROM python:3.12-slim AS builder

COPY --from=deps-export /requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

# ── Stage 3: production runtime ───────────────────────────────
FROM python:3.12-slim AS runtime

RUN useradd --system --no-create-home appuser

WORKDIR /app
COPY --from=builder /wheels /wheels     # pre-built wheels
COPY --from=deps-export /requirements.txt .
RUN pip install --no-cache-dir --no-index \    # install offline from wheels
    --find-links /wheels \
    -r requirements.txt && \
    rm -rf /wheels                      # remove wheels after install

COPY src/ /app/src/

USER appuser
EXPOSE 8000
HEALTHCHECK CMD curl -f http://localhost:8000/health || exit 1
ENTRYPOINT ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

### Dev vs Prod — Same Dockerfile, Different Targets

```dockerfile
# syntax=docker/dockerfile:1

FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci

# ── Dev target: includes devDependencies and hot-reload ───────
FROM base AS development
ENV NODE_ENV=development
COPY . .                         # copy all source (will be volume-mounted anyway)
CMD ["npm", "run", "dev"]        # hot-reload server

# ── Test target: run tests then exit ──────────────────────────
FROM base AS test
COPY . .
RUN npm test                     # runs during build; fails build if tests fail

# ── Prod target: minimal runtime ──────────────────────────────
FROM base AS build
COPY . .
RUN npm run build                # compile

FROM node:20-alpine AS production
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=base /app/node_modules ./node_modules  # prod deps only
USER app
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

```bash
# Target specific stages
docker build --target development .    # dev image with hot-reload
docker build --target test .           # run tests
docker build .                         # default: production (last stage)
docker build --target production .     # explicit prod
```

---

## 6. BuildKit & Advanced Building

### Enabling BuildKit

```bash
# Method 1: environment variable (per-command)
DOCKER_BUILDKIT=1 docker build .

# Method 2: set in daemon.json (permanent, for all builds)
# "features": { "buildkit": true }

# Method 3: docker buildx (always uses BuildKit)
docker buildx build .                 # BuildKit is mandatory with buildx

# Check BuildKit version
docker buildx version
```

---

### docker buildx — Setup

```bash
# List available builders
docker buildx ls

# Create a new builder with a specific driver
docker buildx create \
  --name my-builder \               # name the builder
  --driver docker-container \       # use container-based builder (more features)
  --bootstrap \                     # start it immediately
  --use                             # make it the active builder

# Inspect builder capabilities
docker buildx inspect my-builder

# Remove a builder
docker buildx rm my-builder
```

---

### Build Cache Strategies

```bash
# ── Local cache (development) ────────────────────────────────
docker buildx build \
  --cache-to type=local,dest=/tmp/docker-cache \   # export cache to local dir
  --cache-from type=local,src=/tmp/docker-cache \  # import cache from local dir
  -t myapp:latest .

# ── Registry cache (CI/CD) ────────────────────────────────────
docker buildx build \
  --cache-to type=registry,ref=registry.example.com/myapp:cache,mode=max \
  --cache-from type=registry,ref=registry.example.com/myapp:cache \
  -t registry.example.com/myapp:latest \
  --push .

# mode=max: cache ALL layers (slower push, faster rebuild)
# mode=min: cache only final layer (smaller, less cache hit rate)

# ── Inline cache (simple, embedded in image) ──────────────────
docker buildx build \
  --cache-to type=inline \          # embed cache metadata in the image itself
  --cache-from myapp:latest \       # reuse cache from previously built image
  -t myapp:latest .
```

---

### RUN --mount Types

```dockerfile
# syntax=docker/dockerfile:1

# type=cache — persistent cache between builds (NOT in final image)
RUN --mount=type=cache,target=/root/.npm \     # npm cache persists
    npm ci

RUN --mount=type=cache,target=/root/.m2 \      # Maven .m2 cache persists
    mvn package -DskipTests

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y build-essential

# type=secret — inject secret without storing in image
# Build with: docker buildx build --secret id=npmrc,src=$HOME/.npmrc .
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install              # .npmrc used but NOT stored in any layer

# Build with: docker buildx build --secret id=aws,env=AWS_SECRET_ACCESS_KEY .
RUN --mount=type=secret,id=aws,env=AWS_SECRET_ACCESS_KEY \
    aws s3 cp s3://bucket/file .

# type=ssh — forward SSH agent
# Build with: docker buildx build --ssh default .
RUN --mount=type=ssh \
    git clone git@github.com:org/private-repo.git /app

# type=bind — temporary bind mount from build context (no COPY needed)
RUN --mount=type=bind,source=scripts/,target=/scripts \
    /scripts/configure.sh    # script used but not copied to image
```

---

### Multi-platform Builds

```bash
# QEMU emulation setup (one-time)
docker run --privileged --rm tonistiigi/binfmt --install all

# Build for multiple platforms simultaneously
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \  # all target platforms
  -t registry.example.com/myapp:latest \
  --push .                           # must push to registry (can't load multi-arch locally)

# Build for single alternate platform (can load locally)
docker buildx build \
  --platform linux/arm64 \
  -t myapp:arm64 \
  --load .                           # load into local docker images

# Inspect multi-arch manifest
docker buildx imagetools inspect registry.example.com/myapp:latest
```

---

### docker buildx bake

```hcl
# docker-bake.hcl — build multiple images with one command
group "default" {
  targets = ["app", "nginx"]          # what 'docker buildx bake' builds by default
}

variable "TAG" {
  default = "latest"                  # override: TAG=v1.2 docker buildx bake
}

variable "REGISTRY" {
  default = "registry.example.com"
}

target "app" {
  context    = "."
  dockerfile = "Dockerfile"
  tags       = ["${REGISTRY}/myapp:${TAG}", "${REGISTRY}/myapp:latest"]
  platforms  = ["linux/amd64", "linux/arm64"]
  cache-from = ["type=registry,ref=${REGISTRY}/myapp:cache"]
  cache-to   = ["type=registry,ref=${REGISTRY}/myapp:cache,mode=max"]
}

target "nginx" {
  context    = "nginx/"
  dockerfile = "Dockerfile.nginx"
  tags       = ["${REGISTRY}/nginx-custom:${TAG}"]
  platforms  = ["linux/amd64", "linux/arm64"]
}
```

```bash
# Build all targets
docker buildx bake --push

# Build specific target with override
TAG=v1.2.3 docker buildx bake app --push
```

---

### Hadolint — Dockerfile Linting

```bash
# Run hadolint
docker run --rm -i hadolint/hadolint < Dockerfile

# Common violations:
# DL3007: Use specific tags (not latest) — FROM node:latest
# DL3008: Pin package versions — apt-get install curl (not curl=7.x)
# DL3009: Delete apt lists — rm -rf /var/lib/apt/lists/*
# DL3025: Use exec form in CMD — CMD node server.js → CMD ["node", "server.js"]
# DL4006: Set SHELL to use pipefail — set -o pipefail not default in /bin/sh
# SC2086: Double-quote variables in shell scripts within RUN
```

## 7. Containers — Running & Managing

### docker run — Complete Flag Reference

```bash
# Core flags
docker run \
  --detach, -d \              # run in background (detached mode)
  --interactive, -i \         # keep stdin open
  --tty, -t \                 # allocate pseudo-TTY
  --rm \                      # remove container when it exits
  --name my-container \       # assign a name (else random)

# Image and command
  nginx:1.25 \                # image
  nginx -g "daemon off;" \    # override CMD (extra args after image)

# Port publishing
  -p 8080:80 \                # host:container (TCP)
  -p 127.0.0.1:8080:80 \      # bind to specific host IP
  -p 80 \                     # random host port → container port 80
  -P \                        # publish ALL EXPOSE'd ports to random host ports

# Volumes
  -v myvolume:/data \         # named volume
  -v /host/path:/app:ro \     # bind mount, read-only
  --mount type=tmpfs,target=/tmp,tmpfs-size=100m \   # tmpfs

# Environment
  -e NODE_ENV=production \    # single env var
  --env-file .env \           # env vars from file
  --env-file prod.env \       # multiple env files

# Networking
  --network my-network \      # connect to named network
  --network-alias myapp \     # add DNS alias on that network
  --hostname myhost \         # set container hostname

# Resource limits
  --memory 512m \             # memory limit (hard)
  --memory-reservation 256m \ # soft limit (best effort)
  --memory-swap 1g \          # total memory + swap (-1 = unlimited)
  --cpus 1.5 \                # CPU cores (fractional OK)
  --cpu-shares 512 \          # relative CPU weight (default 1024)
  --pids-limit 100 \          # max number of processes

# User and security
  --user 1001:1001 \          # run as uid:gid
  --workdir /app \            # override WORKDIR
  --read-only \               # read-only root filesystem
  --tmpfs /tmp \              # writable tmpfs for /tmp
  --cap-drop ALL \            # drop all capabilities
  --cap-add NET_BIND_SERVICE \ # add specific capability back
  --security-opt no-new-privileges \   # prevent privilege escalation
  --security-opt seccomp=my-profile.json \

# Lifecycle
  --restart unless-stopped \  # restart policy
  --platform linux/arm64 \    # specify platform for multi-arch images
  --pull always \             # always pull latest version of tag
  --entrypoint /bin/sh        # override ENTRYPOINT
```

---

### Container Lifecycle Commands

```bash
# Start, stop, restart
docker start my-container              # start stopped container
docker stop my-container               # send SIGTERM, wait, then SIGKILL
docker stop --time=30 my-container     # wait up to 30s for graceful stop (default 10s)
docker restart my-container            # stop + start
docker kill my-container               # send SIGKILL immediately
docker kill --signal SIGHUP my-container  # send specific signal

# Execute commands in running container
docker exec my-container ls /app        # run command
docker exec -it my-container bash       # interactive shell
docker exec -it -u root my-container sh # as root user
docker exec -e DEBUG=1 my-container cmd # with extra env var

# Get a shell when bash doesn't exist (Alpine)
docker exec -it my-alpine-container sh  # ash/sh on Alpine

# Logs
docker logs my-container               # all logs
docker logs -f my-container            # follow (stream)
docker logs --tail 100 my-container    # last 100 lines
docker logs --since 1h my-container    # last hour
docker logs --since "2024-01-15T10:00:00" my-container
docker logs --until "2024-01-15T11:00:00" my-container
docker logs --timestamps my-container  # include timestamps
```

---

### Inspecting Containers

```bash
# docker ps — list containers
docker ps                              # running only
docker ps -a                           # all (including stopped)
docker ps --filter status=exited       # only exited
docker ps --filter name=myapp          # filter by name
docker ps --filter ancestor=nginx:1.25 # containers using this image
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# docker inspect with Go templates
docker inspect my-container
docker inspect --format='{{.NetworkSettings.IPAddress}}' my-container
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-container
docker inspect --format='{{range .Mounts}}{{.Source}} → {{.Destination}}\n{{end}}' my-container
docker inspect --format='{{json .State}}' my-container | jq .
docker inspect --format='{{.State.Health.Status}}' my-container

# Resource usage
docker stats                           # live stats for all containers
docker stats my-container             # stats for one container
docker stats --no-stream              # snapshot (not live)
docker stats --format "{{.Name}}: CPU={{.CPUPerc}} MEM={{.MemUsage}}"

# Processes inside container
docker top my-container               # ps-style process list
docker top my-container aux           # with options

# File changes since container started
docker diff my-container              # A=added, C=changed, D=deleted
```

---

### Restart Policies

```bash
# no (default) — never restart
docker run --restart no nginx

# always — always restart; survives daemon restart
docker run --restart always nginx

# unless-stopped — restart unless manually stopped; survives daemon restart
docker run --restart unless-stopped nginx  # ✅ recommended for services

# on-failure — only restart on non-zero exit; optional max count
docker run --restart on-failure:3 my-job   # retry up to 3 times
```

| Policy | On crash | On daemon restart | On manual stop |
|--------|----------|-------------------|----------------|
| `no` | ❌ | ❌ | ❌ |
| `always` | ✅ | ✅ | ✅ (restarts again!) |
| `unless-stopped` | ✅ | ✅ | ❌ stays stopped |
| `on-failure` | ✅ | ✅ | ❌ stays stopped |

---

### Resource Constraints

```bash
# Memory limits
docker run --memory=512m nginx        # hard limit: OOM-killed if exceeded
docker run --memory=512m \
           --memory-swap=1g nginx     # swap = total RAM+swap, not just swap amount
docker run --memory=512m \
           --memory-swap=-1 nginx     # unlimited swap (⚠️ can cause host issues)
docker run --memory-reservation=256m nginx  # soft limit: preferred target

# CPU limits
docker run --cpus=1.5 nginx           # 1.5 CPU cores worth
docker run --cpu-shares=512 nginx     # relative weight (default 1024)
                                      # only matters when CPU is contested

# OOM behaviour
docker run --oom-kill-disable nginx   # ⚠️ dangerous: can OOM the host!
docker run --oom-score-adj=-500 nginx # prefer other processes for OOM kill (range -1000 to 1000)

# Process limits
docker run --pids-limit=100 nginx     # max 100 processes (fork bomb protection)
```

---

## 8. Networking

### Network Drivers

| Driver | Scope | Use case |
|--------|-------|----------|
| `bridge` | Single host | Default; isolated containers on one host |
| `host` | Single host | Container shares host network stack |
| `none` | Single host | No networking at all |
| `overlay` | Multi-host | Swarm/Kubernetes cross-host communication |
| `macvlan` | Single host | Container gets real MAC/IP on LAN |
| `ipvlan` | Single host | Like macvlan but shares MAC address |

---

### Bridge Networks

```
Default bridge (docker0) — AVOID for production:
┌────────────────────────────────────────────────┐
│  Host                                          │
│  ┌──────────┐  ┌──────────┐  docker0:          │
│  │Container1│  │Container2│  172.17.0.1         │
│  │172.17.0.2│  │172.17.0.3│                     │
│  └──────────┘  └──────────┘                     │
│  ❌ containers communicate only by IP (no DNS)  │
│  ❌ all containers on same network by default   │
└────────────────────────────────────────────────┘

User-defined bridge — USE THIS:
┌────────────────────────────────────────────────┐
│  Host                                          │
│  ┌──────────────────────────────────────────┐  │
│  │  my-network                              │  │
│  │  ┌──────────┐  ┌──────────┐             │  │
│  │  │  web     │  │  api     │             │  │
│  │  │(10.0.1.2)│  │(10.0.1.3)│             │  │
│  │  └──────────┘  └──────────┘             │  │
│  │  ✅ web can reach api by name "api"      │  │
│  │  ✅ isolated from other networks         │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

```bash
# Create user-defined bridge network
docker network create \
  --driver bridge \                     # bridge is default
  --subnet 10.0.1.0/24 \               # custom IP range
  --gateway 10.0.1.1 \                 # custom gateway
  --opt com.docker.network.bridge.name=br-myapp \  # host interface name
  my-network

# Run containers connected to the network
docker run -d --name api --network my-network myapi:latest
docker run -d --name web --network my-network \
  -e API_URL=http://api:8080 \          # resolve 'api' by container name!
  mynginx:latest

# Attach running container to additional network
docker network connect my-network existing-container

# Inspect network
docker network inspect my-network
docker network ls
docker network prune                    # remove all unused networks
```

---

### Port Publishing

```bash
# Publish port: host_ip:host_port:container_port/protocol
docker run -p 8080:80 nginx              # all IPs, host 8080 → container 80
docker run -p 127.0.0.1:8080:80 nginx   # localhost only (more secure)
docker run -p 80 nginx                   # random host port → container 80
docker run -P nginx                      # all EXPOSE'd ports → random host ports

# Find what port was assigned
docker port my-container                 # show all port mappings
docker port my-container 80             # show host port for container port 80
# or: docker inspect --format='{{.NetworkSettings.Ports}}' my-container
```

> 💡 **Tip:** Bind to `127.0.0.1` (`-p 127.0.0.1:8080:80`) for services that should only be accessible locally. Binding to `0.0.0.0` (default) exposes the port on all host interfaces including public ones — bypasses UFW/iptables firewall rules.

---

### Container DNS

```bash
# DNS inside containers
# - Built-in resolver: 127.0.0.11 (Docker's embedded DNS server)
# - Resolves: container names, service names, network aliases
# - Falls back to host DNS for external names

# Custom DNS for a container
docker run --dns 1.1.1.1 --dns 8.8.8.8 myapp   # custom DNS servers
docker run --dns-search example.com myapp         # default search domain
docker run --dns-opt ndots:1 myapp               # resolver options

# Network alias — extra DNS name for a container
docker run --network my-net --network-alias db --network-alias database mypostgres
# Both 'db' and 'database' resolve to this container on my-net
```

---

### Host and None Networking

```bash
# Host networking — container shares host network stack
# Pros: maximum performance, no NAT overhead
# Cons: no isolation, port conflicts, only works on Linux
docker run --network host nginx           # nginx listens on host's port 80 directly

# None — no networking
docker run --network none myapp           # completely isolated, no internet access
# Use case: data processing jobs that don't need network

# Check container's network mode
docker inspect --format='{{.HostConfig.NetworkMode}}' my-container
```

---

## 9. Volumes & Storage

### Storage Types — Decision Matrix

```
Which storage type to use?

Data must persist after container removed?
  NO  → tmpfs mount (in-memory, fast, ephemeral)
  YES → continue...

Data shared with host for development?
  YES → bind mount (exact host path in container)
  NO  → continue...

Managed by Docker, portable, works in Swarm?
  YES → named volume ✅ (recommended for production)

Need shared memory between processes?
  YES → tmpfs mount

Data needed by multiple containers?
  YES → named volume (share by name) or bind mount (same host path)
```

---

### Named Volumes

```bash
# Create and manage volumes
docker volume create mydata              # create named volume
docker volume ls                         # list volumes
docker volume inspect mydata            # details: mountpoint, driver, etc.
docker volume rm mydata                 # remove (fails if in use)
docker volume prune                     # remove all unused volumes
docker volume prune --filter "label!=keep"

# Use in docker run
docker run -v mydata:/app/data myapp    # name:path shorthand
docker run \
  --mount type=volume,source=mydata,target=/app/data \   # explicit syntax
  myapp

# Find volume's data on host (inspect the mountpoint)
docker volume inspect mydata --format='{{.Mountpoint}}'
# → /var/lib/docker/volumes/mydata/_data
```

---

### Bind Mounts

```bash
# -v shorthand (older style)
docker run -v /host/absolute/path:/container/path nginx
docker run -v /host/path:/container/path:ro nginx  # read-only

# --mount (preferred — explicit and clear)
docker run \
  --mount type=bind,source=/host/absolute/path,target=/container/path \
  nginx

docker run \
  --mount type=bind,source=$(pwd)/config,target=/etc/nginx/conf.d,readonly \
  nginx

# ⚠️ Bind mounts require ABSOLUTE paths
docker run -v ./config:/etc/nginx/conf.d nginx     # ❌ fails — relative path!
docker run -v "$(pwd)/config":/etc/nginx/conf.d nginx  # ✅ expand to absolute
```

---

### --mount vs -v

| Feature | `-v` | `--mount` |
|---------|------|-----------|
| Syntax | `name:path:options` | `type=X,source=X,target=X` |
| Readability | Compact | Explicit and clear |
| Error messages | Cryptic | Helpful |
| tmpfs options | Limited | Full support |
| Required for Swarm | ❌ | ✅ |

> 💡 **Tip:** Always use `--mount` in scripts and Dockerfiles for clarity. Use `-v` only for quick interactive commands where brevity matters.

---

### tmpfs Mounts

```bash
# In-memory filesystem — not persisted, lost when container stops
# Use for: sensitive data, temp files, high-write workloads with low persistence needs

docker run \
  --mount type=tmpfs,target=/tmp,tmpfs-size=100m,tmpfs-mode=1777 \
  myapp

# Or shorthand
docker run --tmpfs /tmp:rw,size=100m,exec myapp

# Combine read-only rootfs with tmpfs for writable areas
docker run \
  --read-only \                         # root filesystem is read-only
  --tmpfs /tmp \                        # writable temp
  --tmpfs /run \                        # writable run directory
  myapp
```

---

### Volume Backup and Restore

```bash
# Backup: start temp container to tar the volume
docker run --rm \
  -v mydata:/data \                     # mount volume to backup
  -v $(pwd):/backup \                   # mount current dir for output
  alpine \
  tar czf /backup/mydata-backup.tar.gz /data  # create archive

# Restore: extract archive back into volume
docker run --rm \
  -v mydata:/data \                     # target volume
  -v $(pwd):/backup \                   # source backup location
  alpine \
  tar xzf /backup/mydata-backup.tar.gz -C /

# Copy between volumes (clone)
docker run --rm \
  -v source-vol:/source:ro \
  -v dest-vol:/dest \
  alpine cp -a /source/. /dest/
```

---

### Volume Permissions — UID/GID Mismatch

```
Problem: container runs as non-root (uid 1001)
         but mounted volume is owned by root (uid 0)
         → permission denied!

Solutions:
```

```dockerfile
# Solution 1: match UIDs in Dockerfile
RUN useradd --uid 1001 appuser
USER 1001
VOLUME /data                        # volume created with uid 1001

# Solution 2: entrypoint chown (init container pattern)
# entrypoint.sh
#!/bin/sh
chown -R appuser:appgroup /data     # fix ownership at startup
exec su-exec appuser "$@"           # then drop to appuser and run CMD
```

```yaml
# Solution 3: Compose user mapping
services:
  app:
    image: myapp
    user: "1001:1001"
    volumes:
      - mydata:/data
```

---

## 10. Docker Compose

### compose.yaml Structure

```yaml
# compose.yaml (preferred name) or docker-compose.yml
# 'version' key is deprecated — omit it

services:          # define containers
  ...

networks:          # define networks
  ...

volumes:           # define named volumes
  ...

configs:           # define config files (Swarm)
  ...

secrets:           # define secrets (Swarm or Compose)
  ...
```

---

### Full Service Definition

```yaml
services:
  app:
    # Image and build
    image: registry.example.com/myapp:latest    # use this image
    build:                                        # OR build from Dockerfile
      context: .                                 # build context directory
      dockerfile: Dockerfile.prod                # Dockerfile path
      target: production                         # multi-stage target
      args:                                      # build-time ARG values
        NODE_ENV: production
        VERSION: "1.2.3"
      cache_from:                                # use these as cache sources
        - registry.example.com/myapp:cache
      secrets:                                   # secrets available during build
        - npmrc
      platforms:                                 # build for platforms
        - linux/amd64
        - linux/arm64
      tags:                                      # additional tags to apply
        - registry.example.com/myapp:${VERSION}

    # Container config
    container_name: myapp                        # fixed name (can't scale if set)
    hostname: myapp-host
    restart: unless-stopped                      # restart policy

    # Commands
    command: ["node", "server.js", "--port", "3000"]  # override CMD
    entrypoint: ["/docker-entrypoint.sh"]              # override ENTRYPOINT

    # Ports
    ports:
      - "3000:3000"                              # host:container
      - "127.0.0.1:9229:9229"                   # debug port, localhost only

    # Environment
    environment:
      NODE_ENV: production
      DB_HOST: postgres                          # service name = DNS hostname
      DB_PORT: 5432
    env_file:
      - .env                                     # load from .env file
      - production.env                           # override with production values

    # Volumes
    volumes:
      - app-data:/app/data                       # named volume
      - ./config:/app/config:ro                 # bind mount, read-only
      - type: tmpfs                              # tmpfs mount
        target: /tmp
        tmpfs:
          size: 100m

    # Networking
    networks:
      - frontend
      - backend
    network_mode: bridge                         # or: host, none

    # Dependencies
    depends_on:
      postgres:
        condition: service_healthy               # wait for postgres healthcheck
      redis:
        condition: service_started               # just wait for start
      migrations:
        condition: service_completed_successfully # one-time job must succeed

    # Health check (override Dockerfile HEALTHCHECK)
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/healthz"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

    # Resource limits (requires deploy section or --compatibility flag)
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
      replicas: 1

    # Labels
    labels:
      com.example.service: "app"
      traefik.enable: "true"
      traefik.http.routers.app.rule: "Host(`app.example.com`)"

    # User and security
    user: "1001:1001"
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE

    # Profiles (only start with --profile flag)
    profiles:
      - production

    # Logging
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

### Variable Substitution and .env

```yaml
# compose.yaml — using variables
services:
  app:
    image: myapp:${APP_VERSION:-latest}         # default: latest
    environment:
      DB_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD must be set}  # error if unset
      OPTIONAL_VAR: ${OPTIONAL_VAR:-}           # empty if unset (no error)
```

```bash
# .env file (auto-loaded by Compose — DO NOT commit to git!)
APP_VERSION=1.2.3
DB_PASSWORD=supersecret
POSTGRES_DATA=/data/postgres
```

```bash
# Override .env file location
docker compose --env-file staging.env up

# Shell env takes precedence over .env file
APP_VERSION=1.3.0 docker compose up    # uses 1.3.0 not .env value
```

---

### depends_on with Health Checks

```yaml
services:
  postgres:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 30s     # give postgres time to init

  app:
    image: myapp
    depends_on:
      postgres:
        condition: service_healthy   # app won't start until postgres is healthy
        restart: true                # restart app if postgres restarts (Compose 2.17+)
```

---

### docker compose watch (Live Reload)

```yaml
# compose.yaml — define watch rules (Compose 2.22+)
services:
  app:
    build: .
    develop:
      watch:
        - action: sync                           # copy changed files into container
          path: ./src
          target: /app/src
          ignore:
            - node_modules/

        - action: rebuild                        # rebuild image when these change
          path: package.json

        - action: sync+restart                  # sync files then restart container
          path: ./config
          target: /app/config
```

```bash
docker compose watch                            # start watching for changes
docker compose up --watch                      # up + watch in one command
```

---

### Override Files

```bash
# compose.yaml — base config
# compose.override.yml — auto-merged for local dev (add to .gitignore!)
# compose.prod.yml — production overrides

# Apply override files
docker compose -f compose.yaml -f compose.prod.yml up

# Merge strategy: override file wins for scalar values, lists are appended
```

```yaml
# compose.override.yml — local dev overrides (auto-loaded)
services:
  app:
    build:
      target: development          # use dev stage
    volumes:
      - .:/app                     # mount source for hot-reload
    environment:
      NODE_ENV: development
      DEBUG: "true"
    ports:
      - "9229:9229"               # expose debug port locally
```

---

### Compose in CI/CD

```bash
# Run tests in isolation
docker compose -f compose.test.yml run --rm \
  --exit-code-from tests \         # exit with tests container's exit code
  tests

# Clean up everything after CI run
docker compose down --volumes --remove-orphans --rmi local

# Build with specific tags for CI
docker compose build --build-arg VERSION=${CI_COMMIT_SHA}

# Push all service images
docker compose push
```

---

### Secrets in Compose

```yaml
services:
  app:
    image: myapp
    secrets:
      - db_password                          # mounted at /run/secrets/db_password
      - source: api_key
        target: /app/secrets/api_key         # custom mount path
        uid: "1001"
        mode: 0400                           # read-only by owner

secrets:
  db_password:
    file: ./secrets/db_password.txt         # file-based secret (dev)

  api_key:
    environment: "API_KEY"                   # from environment variable (Compose 2.23+)
```

---

## 11. Docker Registry & Image Distribution

### Docker Hub

```bash
# Login
docker login                              # prompts for Docker Hub credentials
docker login -u myuser                    # specify username

# Logout (removes stored credentials)
docker logout

# Pull from Docker Hub (implicit registry)
docker pull nginx                         # same as docker pull docker.io/library/nginx:latest
docker pull myorg/myapp:v1.2.3

# Push to Docker Hub
docker tag myapp:latest myuser/myapp:v1.2.3
docker push myuser/myapp:v1.2.3
docker push myuser/myapp:latest

# Rate limits: Docker Hub free tier = 100 pulls/6h for unauthenticated, 200 for authenticated
```

---

### AWS ECR

```bash
# Authenticate (requires AWS CLI configured)
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS \
    --password-stdin \
    123456789012.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE      # prevent overwriting existing tags

# Tag and push
docker tag myapp:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2.3
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2.3

# Pull
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2.3

# ECR lifecycle policy — auto-delete old images
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy '{
    "rules": [{
      "rulePriority": 1,
      "description": "Keep last 10 images",
      "selection": {"tagStatus": "any", "countType": "imageCountMoreThan", "countNumber": 10},
      "action": {"type": "expire"}
    }]
  }'
```

---

### GitHub Container Registry (GHCR)

```bash
# Authenticate with GitHub token
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Tag and push
docker tag myapp:latest ghcr.io/myorg/myapp:v1.2.3
docker push ghcr.io/myorg/myapp:v1.2.3

# In GitHub Actions (automated login)
- name: Log in to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

---

### Image Promotion Pattern

```bash
# Build once, promote through environments
# Never rebuild — same image, different tags

# CI: build and push with commit SHA
docker buildx build \
  --tag registry.example.com/myapp:${GIT_SHA} \
  --push .

# Promote to staging (add tag, no rebuild)
docker buildx imagetools create \
  --tag registry.example.com/myapp:staging \
  registry.example.com/myapp:${GIT_SHA}

# Promote to production (same image, new tag)
docker buildx imagetools create \
  --tag registry.example.com/myapp:production \
  --tag registry.example.com/myapp:v1.2.3 \
  registry.example.com/myapp:${GIT_SHA}

# Verify same digest
docker inspect --format='{{index .RepoDigests 0}}' registry.example.com/myapp:staging
docker inspect --format='{{index .RepoDigests 0}}' registry.example.com/myapp:production
# Both should print the same sha256 digest!
```

---

### Image Signing with cosign

```bash
# Install cosign: https://github.com/sigstore/cosign

# Generate key pair
cosign generate-key-pair             # creates cosign.key + cosign.pub

# Sign image (after pushing to registry)
cosign sign --key cosign.key registry.example.com/myapp:v1.2.3

# Keyless signing with Sigstore (uses OIDC identity)
COSIGN_EXPERIMENTAL=1 cosign sign registry.example.com/myapp:v1.2.3

# Verify signature
cosign verify \
  --key cosign.pub \
  registry.example.com/myapp:v1.2.3 \
  | jq .

# Verify in admission controller (Kubernetes policy enforcement)
# Use: Kyverno, OPA Gatekeeper, or Sigstore Policy Controller
```

---

### Running a Private Registry

```bash
# Start registry with TLS
docker run -d \
  --name registry \
  --restart unless-stopped \
  -p 5000:5000 \
  -v /etc/ssl/registry:/certs:ro \          # TLS certificates
  -v /data/registry:/var/lib/registry \     # persistent storage
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/server.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/server.key \
  registry:2

# Push to private registry
docker tag myapp:latest registry.example.com:5000/myapp:v1.2.3
docker push registry.example.com:5000/myapp:v1.2.3

# For self-signed cert (dev): configure insecure-registries in daemon.json
# "insecure-registries": ["my-registry.internal:5000"]
```

## 12. Security

### Container Threat Model

```
What Docker isolation PROVIDES:
  ✅ Process isolation (PID namespace) — containers can't see host processes
  ✅ Network isolation — containers have their own network stack by default
  ✅ Filesystem isolation — containers have their own root filesystem
  ✅ Resource limits — containers can be CPU/memory constrained

What Docker isolation does NOT provide:
  ❌ Kernel isolation — all containers share the HOST KERNEL
     → kernel exploits can escape containers
  ❌ Root isolation (without USER namespace) — container root = host root
  ❌ Hardware isolation — shared CPU, memory hardware
  ❌ Complete security — defence in depth is still required

Defence-in-depth strategy:
  1. Non-root user in container (USER directive)
  2. Read-only filesystem (--read-only)
  3. Drop all capabilities (--cap-drop ALL)
  4. No privilege escalation (--security-opt no-new-privileges)
  5. Seccomp profile (limit syscalls)
  6. Scan images for CVEs
  7. Never use --privileged
  8. Network segmentation (user-defined networks)
```

---

### Running as Non-Root

```dockerfile
# ✅ Create dedicated user in Dockerfile
FROM node:20-alpine

RUN addgroup -S -g 1001 appgroup && \
    adduser -S -u 1001 -G appgroup appuser

WORKDIR /app
COPY --chown=appuser:appgroup . .    # set ownership during copy
RUN npm ci --only=production

USER appuser                          # switch to non-root
EXPOSE 3000
CMD ["node", "server.js"]
```

```bash
# ✅ Override user at runtime
docker run --user 1001:1001 myapp

# Verify container is not running as root
docker exec myapp whoami              # should NOT print "root"
docker exec myapp id                  # uid should be non-zero
```

---

### Capabilities

```bash
# Drop ALL capabilities, add only what's needed
docker run \
  --cap-drop ALL \                    # drop every capability
  --cap-add NET_BIND_SERVICE \       # allow binding to ports < 1024
  --cap-add CHOWN \                  # allow chown (if needed)
  nginx

# Common capabilities and when to drop/add them
# CAP_NET_BIND_SERVICE — bind ports < 1024 (add for web servers if running as non-root)
# CAP_NET_RAW          — raw network access (drop: enables ping flood attacks)
# CAP_SYS_PTRACE       — strace, gdb (drop: allows debugging/hijacking processes)
# CAP_SYS_ADMIN        — broad admin (NEVER add: almost = root)
# CAP_DAC_OVERRIDE     — bypass file permissions (drop: hardening)

# List capabilities in a running container
docker inspect --format='{{.HostConfig.CapAdd}}' my-container
docker exec my-container cat /proc/self/status | grep Cap
```

---

### Seccomp Profiles

```bash
# Docker's default seccomp profile blocks ~44 syscalls
# Show current profile
docker inspect --format='{{.HostConfig.SecurityOpt}}' my-container

# Use custom seccomp profile
docker run --security-opt seccomp=my-profile.json myapp

# Disable seccomp (⚠️ dangerous)
docker run --security-opt seccomp=unconfined myapp
```

```json
// custom-seccomp.json — allow only needed syscalls
{
  "defaultAction": "SCMP_ACT_ERRNO",    // deny everything by default
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "stat", "exit", "mmap",
                "accept4", "socket", "connect", "sendto", "recvfrom"],
      "action": "SCMP_ACT_ALLOW"         // allow only these syscalls
    }
  ]
}
```

---

### The --privileged Flag

```bash
# --privileged: disables ALL security mechanisms
# Gives container FULL access to host kernel, all devices, all capabilities
# Container root = host root — complete container escape!

# 🔴 Anti-pattern: always avoid --privileged
docker run --privileged myapp         # equivalent to running as host root

# ✅ Alternatives for common --privileged use cases:

# Mount specific device (instead of all devices)
docker run --device /dev/net/tun myapp

# Load kernel module (instead of --privileged)
docker run --cap-add SYS_MODULE myapp

# Access cgroups (for monitoring tools)
docker run --cgroupns=host \
           --mount type=bind,src=/sys/fs/cgroup,dst=/sys/fs/cgroup,ro \
           prometheus-node-exporter
```

---

### Secrets Management — Never Bake Into Images

```bash
# 🔴 Anti-pattern: secret in Dockerfile ENV
ENV API_KEY=supersecret    # visible in docker inspect, docker history!

# 🔴 Anti-pattern: secret passed as ARG
docker build --build-arg API_KEY=supersecret .  # visible in docker history!

# ✅ BuildKit secret (not stored in any layer)
docker buildx build --secret id=api_key,env=API_KEY .
# In Dockerfile: RUN --mount=type=secret,id=api_key ...

# ✅ Runtime secret injection via environment variable
docker run -e API_KEY="${API_KEY}" myapp        # from host environment

# ✅ Docker secret (Swarm/Compose) — file mounted at /run/secrets/
docker secret create api_key ./api_key.txt
docker run --secret api_key myapp
# Access in container: cat /run/secrets/api_key

# ✅ External secret store (production best practice)
# HashiCorp Vault agent sidecar
# AWS Secrets Manager via env injection
# Kubernetes Secrets (encrypted etcd)
```

---

### Docker Socket Security

```bash
# /var/run/docker.sock — the daemon's Unix socket
# Mounting it into a container = giving that container ROOT on the host!
# Any process in the container can start privileged containers, mount host dirs

# 🔴 Anti-pattern (commonly seen in CI/CD)
docker run -v /var/run/docker.sock:/var/run/docker.sock docker:latest

# ✅ Alternatives for building images without socket access:

# Kaniko — build in Kubernetes without Docker daemon
docker run gcr.io/kaniko-project/executor:latest \
  --dockerfile=Dockerfile \
  --context=. \
  --destination=registry.example.com/myapp:latest

# Buildah — build OCI images without daemon
buildah bud -t myapp:latest .
buildah push myapp:latest registry.example.com/myapp:latest

# Docker-in-Docker with TLS (more secure than socket mount)
# Use 'docker:dind' service in CI, connect via TLS
```

---

### Read-only Containers

```bash
# Make root filesystem read-only
docker run \
  --read-only \                         # root filesystem = read-only
  --tmpfs /tmp \                        # writable /tmp (in-memory)
  --tmpfs /run \                        # writable /run
  -v app-data:/app/data \               # writable volume for app data
  myapp

# Combine all hardening flags
docker run \
  --read-only \                         # immutable filesystem
  --tmpfs /tmp:size=50m \              # writable temp
  --user 1001:1001 \                   # non-root
  --cap-drop ALL \                     # no capabilities
  --security-opt no-new-privileges \   # no escalation
  --security-opt seccomp=default \     # default seccomp
  --memory 256m \                      # memory limit
  --cpus 0.5 \                        # CPU limit
  --network my-internal-network \      # isolated network
  myapp
```

---

### docker-bench-security

```bash
# CIS Docker Benchmark automated check
docker run --rm \
  --net host \
  --pid host \
  --userns host \
  --cap-add audit_control \
  -v /etc:/etc:ro \
  -v /lib/systemd/system:/lib/systemd/system:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  docker/docker-bench-security
```

---

## 13. Docker Swarm

### Swarm Architecture

```
Swarm cluster:
┌─────────────────────────────────────────────────────────────┐
│                    MANAGER NODES (Raft consensus)           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Manager 1   │  │  Manager 2   │  │  Manager 3   │      │
│  │  (Leader)    │←─│  (Follower)  │←─│  (Follower)  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                                                   │
│    (schedules services onto workers)                        │
│         │                                                   │
│  ┌──────▼──────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Worker 1   │  │  Worker 2    │  │  Worker 3    │      │
│  │  task task  │  │  task task   │  │  task        │      │
│  └─────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
Use odd number of managers (1, 3, 5): tolerates (N-1)/2 failures
1 manager: no fault tolerance
3 managers: tolerates 1 failure
5 managers: tolerates 2 failures
```

---

### Swarm Initialisation

```bash
# Initialise swarm on first manager
docker swarm init \
  --advertise-addr 192.168.1.10 \    # IP other nodes reach this manager on
  --listen-addr 0.0.0.0:2377         # port to listen on (default 2377)

# Get join tokens
docker swarm join-token worker       # prints command for adding workers
docker swarm join-token manager      # prints command for adding managers

# Worker joins the swarm
docker swarm join \
  --token SWMTKN-1-xxxxx \
  192.168.1.10:2377                  # manager IP:port

# List nodes
docker node ls
```

---

### Services — Swarm's Deployment Unit

```bash
# Create a service (replicated containers managed by Swarm)
docker service create \
  --name webapp \
  --replicas 3 \                     # run 3 instances
  --publish published=80,target=8080 \  # expose via routing mesh
  --network my-overlay-net \         # overlay network
  --constraint 'node.role==worker' \ # only run on workers
  --env NODE_ENV=production \
  --secret my-secret \               # inject Docker secret
  --update-delay 30s \              # wait 30s between each replica update
  --update-parallelism 1 \          # update 1 replica at a time
  --update-failure-action rollback \ # rollback if health check fails
  --rollback-parallelism 2 \        # rollback 2 replicas at a time
  --health-cmd "curl -f http://localhost:8080/health" \
  registry.example.com/webapp:v1.2.3

# Update service (rolling update)
docker service update \
  --image registry.example.com/webapp:v1.3.0 \  # new image
  --update-delay 15s \
  --update-parallelism 1 \
  webapp

# Rollback to previous version
docker service rollback webapp

# Scale service
docker service scale webapp=5 redis=2   # scale multiple services at once

# List services and their tasks
docker service ls
docker service ps webapp               # tasks per service
docker service logs webapp             # aggregated logs from all replicas
docker service inspect webapp --pretty
```

---

### Swarm Secrets

```bash
# Create secrets
echo "supersecretpassword" | docker secret create db_password -
docker secret create ssl_cert /path/to/cert.pem

# List secrets (values are never shown)
docker secret ls

# Use in service (mounted at /run/secrets/<name>)
docker service create \
  --name myapp \
  --secret db_password \             # mounted at /run/secrets/db_password
  --secret source=ssl_cert,target=/etc/ssl/app.pem,mode=0400 \  # custom path
  myimage

# In application code:
# db_pass = open('/run/secrets/db_password').read().strip()

# Remove secret (must remove from all services first)
docker service update --secret-rm db_password myapp
docker secret rm db_password
```

---

### Stack Deploy (Compose for Swarm)

```yaml
# compose-swarm.yaml — Swarm stack file
services:
  webapp:
    image: registry.example.com/webapp:v1.2.3
    networks:
      - frontend
    secrets:
      - db_password
    deploy:                              # Swarm deployment config
      replicas: 3
      update_config:
        parallelism: 1                   # update 1 replica at a time
        delay: 30s
        failure_action: rollback         # auto-rollback on failure
        order: start-first               # start new before stopping old (zero-downtime)
      rollback_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      placement:
        constraints:
          - node.role == worker
          - node.labels.region == eu-west-1
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          memory: 256M
      labels:
        - "traefik.enable=true"

networks:
  frontend:
    driver: overlay
    attachable: true

secrets:
  db_password:
    external: true                       # created externally with 'docker secret create'
```

```bash
# Deploy stack
docker stack deploy -c compose-swarm.yaml mystack
docker stack ls
docker stack ps mystack                  # tasks in stack
docker stack services mystack            # services in stack
docker stack rm mystack                  # remove stack
```

---

## 14. Production Patterns & Best Practices

### Layer Ordering for Cache Efficiency

```dockerfile
# 🔴 Anti-pattern: copy everything first (cache busted on any file change)
FROM node:20-alpine
WORKDIR /app
COPY . .                           # any change → cache busted
RUN npm ci                         # re-downloads ALL deps every time!

# ✅ Right way: copy dependency files first, code last
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./   # only changes when deps change
RUN npm ci                               # only re-runs when deps change
COPY . .                                 # changes frequently — at the end!
```

```
Layer cache rule: put the LEAST frequently changing operations FIRST
                  put the MOST frequently changing operations LAST

Order:
  1. Base image (FROM)           ← changes: never / rarely
  2. System packages (RUN apt)   ← changes: occasionally
  3. Dependency files (COPY pkg) ← changes: when deps added
  4. Install dependencies (RUN)  ← changes: when deps change
  5. App source code (COPY .)    ← changes: every commit
  6. Build command (RUN build)   ← changes: every commit
```

---

### Graceful Shutdown and PID 1

```dockerfile
# The PID 1 problem:
# - In Linux, PID 1 must handle zombie process reaping
# - Shell-form ENTRYPOINT: /bin/sh becomes PID 1; SIGTERM not forwarded to app
# - Many apps don't handle SIGTERM properly → docker stop takes 10s then SIGKILL

# ✅ Solution 1: exec form ENTRYPOINT (app IS PID 1)
ENTRYPOINT ["node", "server.js"]     # node receives SIGTERM directly

# ✅ Solution 2: tini as PID 1 (init process, reaps zombies, forwards signals)
FROM node:20-alpine
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--", "node", "server.js"]

# ✅ Solution 3: use base image with init (some images have it)
FROM node:20-alpine
ENTRYPOINT ["docker-entrypoint.sh"]  # many official images handle this
```

```javascript
// In your application: handle SIGTERM gracefully
process.on('SIGTERM', () => {
    console.log('SIGTERM received, shutting down gracefully');
    server.close(() => {
        console.log('HTTP server closed');
        process.exit(0);
    });
    // Force exit after 10s if graceful shutdown hangs
    setTimeout(() => process.exit(1), 10000);
});
```

---

### .dockerignore Best Practices

```
# .dockerignore — keep build context small and secure

# Version control
.git/
.gitignore
.gitattributes

# Dependencies (always rebuilt inside container)
node_modules/
vendor/
__pycache__/
*.pyc
.venv/
venv/

# IDE and OS
.idea/
.vscode/
*.swp
.DS_Store
Thumbs.db

# Test artifacts
coverage/
.pytest_cache/
*.test.js
*.spec.js
test/
tests/
spec/

# Build artifacts (built inside container)
dist/
build/
target/
*.o
*.a

# Documentation
*.md
docs/
README*

# CI/CD config (not needed inside container)
.github/
.gitlab-ci.yml
Jenkinsfile
.travis.yml

# Security-sensitive (NEVER include)
.env
.env.*
*.pem
*.key
secrets/
credentials/
```

---

### Log Management

```json
// daemon.json — default log config for all containers
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",         // rotate at 10MB
    "max-file": "3",           // keep 3 rotated files = max 30MB per container
    "labels": "service,version",
    "env": "NODE_ENV"
  }
}
```

```bash
# Per-container log driver override
docker run \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp

# Log to syslog
docker run --log-driver syslog --log-opt syslog-address=udp://logserver:514 myapp

# Log to AWS CloudWatch
docker run \
  --log-driver awslogs \
  --log-opt awslogs-region=us-east-1 \
  --log-opt awslogs-group=/myapp/production \
  --log-opt awslogs-stream=instance-1 \
  myapp

# Forward all container logs to Fluentd/Loki
docker run --log-driver fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt tag=myapp.{{.Name}} \
  myapp
```

---

### 12-Factor App in Containers

| Factor | Docker implementation |
|--------|----------------------|
| Config via env | `docker run -e KEY=val` or `--env-file` |
| Logs to stdout | App writes to stdout/stderr; docker logs + drivers handle routing |
| Stateless processes | No local disk state; use volumes/external storage |
| Disposability | Fast startup, graceful shutdown on SIGTERM |
| Dev/prod parity | Same image across envs; only env vars differ |
| Port binding | `EXPOSE` + `-p`; app listens on configurable port |
| Concurrency | Scale via `--scale` in Compose or `--replicas` in Swarm |

---

### Image Size Optimisation

```bash
# Check image sizes
docker image ls --format "{{.Repository}}:{{.Tag}}\t{{.Size}}" | sort -k2 -h

# Analyse layer-by-layer with dive
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive myapp:latest

# CI integration: fail if wasted space > threshold
docker run --rm \
  -e CI=true \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive myapp:latest

# Base image size comparison:
# ubuntu:22.04     → 77 MB
# debian:bookworm-slim → 74 MB
# alpine:3.19      → 7 MB      (musl libc — test compatibility!)
# distroless/base  → 20 MB     (no shell, no package manager)
# scratch          → 0 MB      (static binaries only)
```

---

## 15. Observability & Debugging

### docker inspect — Go Templates

```bash
# Get container IP address
docker inspect --format='{{.NetworkSettings.IPAddress}}' my-container

# Get IP on a specific network
docker inspect --format='{{.NetworkSettings.Networks.my-network.IPAddress}}' my-container

# List all mounts
docker inspect --format='{{range .Mounts}}Source: {{.Source}} → Dest: {{.Destination}} ({{.Mode}})\n{{end}}' my-container

# Get all environment variables
docker inspect --format='{{range .Config.Env}}{{.}}\n{{end}}' my-container

# Get health status
docker inspect --format='{{.State.Health.Status}}' my-container
docker inspect --format='{{json .State.Health}}' my-container | jq .

# Check if container is running
docker inspect --format='{{.State.Running}}' my-container

# Get restart count
docker inspect --format='{{.RestartCount}}' my-container

# Get the entrypoint and command
docker inspect --format='Entrypoint: {{.Config.Entrypoint}} CMD: {{.Config.Cmd}}' my-container
```

---

### docker stats — Interpreting Metrics

```bash
# Live stats for all containers
docker stats

# One-shot snapshot (good for scripting)
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"

# Output explanation:
# CPU %     = % of ONE CPU core (200% means 2 cores fully used)
# MEM USAGE = current resident set size / hard memory limit
# NET I/O   = total bytes received/sent since container started
# BLOCK I/O = total bytes read/written to block devices
# PIDs      = number of processes in container
```

---

### docker events — Event Stream

```bash
# Stream all Docker events in real-time
docker events

# Filter by event type
docker events --filter type=container       # container events only
docker events --filter type=image           # image events (pull, push, delete)
docker events --filter type=network         # network events

# Filter by action
docker events --filter event=start          # container starts
docker events --filter event=die            # container exits
docker events --filter event=health_status  # health check changes

# Filter by container name
docker events --filter container=my-container

# Time range
docker events --since "2024-01-15T10:00:00" --until "2024-01-15T11:00:00"

# Use for automation
docker events --filter event=die --format '{{json .}}' \
  | jq '.Actor.Attributes.name' \
  | xargs -I{} echo "Container {} died at $(date)"
```

---

### Debugging Without exec

```bash
# docker cp — copy files out of container (even stopped containers)
docker cp my-container:/app/logs/error.log ./
docker cp my-container:/etc/nginx/nginx.conf ./debug-nginx.conf

# docker diff — see what changed since container started
docker diff my-container
# A /app/logs/error.log    ← Added
# C /etc/nginx/nginx.conf  ← Changed
# D /tmp/cache             ← Deleted

# Ephemeral debug container sharing namespaces
# (access container's processes and network without exec into it)
docker run --rm -it \
  --pid=container:my-container \     # share PID namespace
  --network=container:my-container \ # share network namespace
  --cap-add SYS_PTRACE \            # allow strace
  nicolaka/netshoot                  # image with debugging tools

# nsenter (host-level access to container namespaces)
PID=$(docker inspect --format '{{.State.Pid}}' my-container)
sudo nsenter --target $PID --mount --uts --ipc --net --pid -- bash
```

---

### docker system — Disk Management

```bash
# Show disk usage breakdown
docker system df
# TYPE           TOTAL  ACTIVE  SIZE     RECLAIMABLE
# Images         12     5       4.2GB    2.1GB (50%)
# Containers     3      2       100MB    0B
# Local Volumes  5      2       20GB     8GB (40%)
# Build Cache    -      -       500MB    500MB

# Verbose: show individual items
docker system df -v

# Prune — remove unused resources
docker system prune               # remove stopped containers, unused networks, dangling images
docker system prune --volumes     # also remove unused volumes (⚠️ careful!)
docker system prune --all         # also remove all unused images (not just dangling)
docker system prune --filter until=48h  # only items older than 48 hours

# Individual prune commands
docker container prune            # stopped containers
docker image prune               # dangling images (<none>:<none>)
docker image prune --all         # all unused images
docker volume prune              # unused volumes
docker network prune             # unused networks
docker builder prune             # BuildKit cache
docker builder prune --keep-storage=5GB   # keep 5GB, remove oldest
```

---

### Common Failure Patterns — Diagnosis

```bash
# OOM Kill diagnosis
docker inspect --format='{{.State.OOMKilled}}' my-container   # true if OOM killed
docker stats --no-stream my-container                         # check memory usage
# Fix: increase --memory limit or reduce app memory usage

# Port already in use
docker run -p 80:80 nginx
# Error: bind: address already in use
lsof -i :80                    # find what's using port 80
docker ps -a --filter publish=80   # find Docker containers using it

# Permission denied
# Usually: container running as non-root trying to write to root-owned bind mount
docker exec my-container id        # check uid
ls -la /host/path                  # check host path ownership
# Fix: match UIDs or use volumes instead of bind mounts

# DNS resolution failures
docker exec my-container nslookup google.com   # test DNS
docker exec my-container cat /etc/resolv.conf  # check DNS config
# Fix: check --dns settings, custom network DNS, daemon.json dns

# Image pull errors
docker pull registry.example.com/myapp
# Error: unauthorized → docker login registry.example.com
# Error: connection refused → check registry URL and network
# Error: no matching manifest → wrong platform, check --platform flag

# Container starts then immediately exits
docker logs my-container          # check application startup error
docker inspect --format='{{.State.ExitCode}}' my-container   # get exit code
# Common causes: missing env vars, missing files, app crash on startup
```

---

## 16. Real-World Patterns & Recipes

### Full Local Development Stack

```yaml
# compose.yaml — local dev with hot reload, database, cache, and proxy
services:
  # ── Reverse proxy ────────────────────────────────────────────
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro  # nginx config
    depends_on:
      app:
        condition: service_healthy
    networks:
      - frontend

  # ── Application ──────────────────────────────────────────────
  app:
    build:
      context: .
      target: development          # use dev stage with hot-reload
    environment:
      NODE_ENV: development
      DATABASE_URL: postgres://appuser:${DB_PASSWORD}@postgres:5432/appdb
      REDIS_URL: redis://redis:6379
    volumes:
      - .:/app                     # mount source for hot-reload
      - /app/node_modules          # anonymous volume: keep container's node_modules
    ports:
      - "3000:3000"
      - "9229:9229"               # Node.js debug port
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/healthz"]
      interval: 10s
      timeout: 5s
      retries: 3
    networks:
      - frontend
      - backend

  # ── Database ─────────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: appdb
    volumes:
      - postgres-data:/var/lib/postgresql/data  # named volume for persistence
      - ./db/init:/docker-entrypoint-initdb.d:ro  # init scripts run at first start
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 30s
    networks:
      - backend

  # ── Cache ─────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    networks:
      - backend

  # ── DB Admin (dev only) ────────────────────────────────────────
  adminer:
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    profiles:
      - tools                      # only starts with: docker compose --profile tools up
    networks:
      - backend

volumes:
  postgres-data:
  redis-data:

networks:
  frontend:
  backend:
    internal: true                 # no external access to backend network
```

---

### Python App Dockerfile (Production)

```dockerfile
# syntax=docker/dockerfile:1
# Dockerfile for Python FastAPI application

# ── Stage 1: export requirements ──────────────────────────────
FROM python:3.12-slim AS requirements

WORKDIR /tmp
RUN pip install poetry==1.8.2 --no-cache-dir   # pin exact Poetry version
COPY pyproject.toml poetry.lock ./
RUN poetry export \
    --without-hashes \             # don't include hashes (for pip compatibility)
    --without dev \                # exclude dev dependencies
    --format requirements.txt \
    -o requirements.txt

# ── Stage 2: production runtime ───────────────────────────────
FROM python:3.12-slim AS production

# Security: create non-root user
RUN groupadd --system --gid 1001 appgroup && \
    useradd --system --uid 1001 --gid appgroup --no-create-home appuser

# Install dependencies first (layer caches until requirements change)
COPY --from=requirements /tmp/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    rm requirements.txt

WORKDIR /app
COPY --chown=appuser:appgroup src/ /app/src/

# Labels (OCI standard)
LABEL org.opencontainers.image.title="myapi" \
      org.opencontainers.image.description="FastAPI application"

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 --start-period=10s \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

ENTRYPOINT ["python", "-m", "uvicorn", "src.main:app"]
CMD ["--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

---

### Node.js App Dockerfile (Production)

```dockerfile
# syntax=docker/dockerfile:1

# ── Stage 1: install dependencies ─────────────────────────────
FROM node:20-alpine AS deps

WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production               # prod deps with cache mount

# ── Stage 2: build TypeScript ─────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci                                 # all deps (includes devDependencies)
COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build                          # compile TypeScript → dist/

# ── Stage 3: production image ─────────────────────────────────
FROM node:20-alpine AS production

# tini: init process for proper signal handling and zombie reaping
RUN apk add --no-cache tini

# Non-root user
RUN addgroup -S -g 1001 nodejs && adduser -S -u 1001 -G nodejs nodeuser

WORKDIR /app

# Copy only what's needed to run
COPY --from=deps --chown=nodeuser:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodeuser:nodejs /app/dist ./dist
COPY --chown=nodeuser:nodejs package.json ./

USER nodeuser
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/healthz || exit 1

ENTRYPOINT ["/sbin/tini", "--"]           # tini as PID 1
CMD ["node", "dist/server.js"]
```

---

### Nginx Reverse Proxy with Let's Encrypt

```yaml
# compose.yaml — nginx + certbot for HTTPS
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro      # per-site configs
      - certbot-webroot:/var/www/certbot:ro       # ACME challenge files
      - certbot-certs:/etc/letsencrypt:ro         # TLS certificates
    depends_on:
      - app

  certbot:
    image: certbot/certbot
    volumes:
      - certbot-webroot:/var/www/certbot          # shared with nginx
      - certbot-certs:/etc/letsencrypt            # store certs
    entrypoint: >
      /bin/sh -c 'trap exit TERM;
        while :; do
          certbot renew;
          sleep 12h & wait $${!};
        done'                                     # auto-renew every 12h

  app:
    image: myapp:latest
    networks:
      - internal

volumes:
  certbot-webroot:
  certbot-certs:
```

```nginx
# nginx/conf.d/app.conf
server {
    listen 80;
    server_name app.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;         # Let's Encrypt challenge
    }

    location / {
        return 301 https://$host$request_uri;   # redirect all HTTP → HTTPS
    }
}

server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://app:3000;            # proxy to app service (DNS resolution)
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

### Docker Cleanup Automation

```bash
#!/bin/bash
# docker-cleanup.sh — safe automated cleanup
set -euo pipefail

echo "=== Docker Disk Usage Before ==="
docker system df

# Remove stopped containers older than 24h
docker container prune --force --filter "until=24h"

# Remove dangling images (untagged)
docker image prune --force

# Remove build cache older than 7 days (keep 5GB)
docker builder prune --force \
  --filter "until=168h" \           # 7 days
  --keep-storage=5GB

# Remove unused volumes (⚠️ check your setup before enabling)
# docker volume prune --force

# Remove unused networks
docker network prune --force

echo "=== Docker Disk Usage After ==="
docker system df
```

```yaml
# Add to compose.yaml as a scheduled service
services:
  cleanup:
    image: docker:cli
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command: >
      sh -c 'while true; do
        docker system prune --force --filter until=24h;
        sleep 86400;    # run daily
      done'
    profiles:
      - maintenance     # only when explicitly started
```

---

## Appendix: Quick Reference

### Most-Used Commands

```bash
# Images
docker pull image:tag                  # pull image
docker build -t name:tag .            # build from Dockerfile
docker push name:tag                  # push to registry
docker images                         # list local images
docker rmi image:tag                  # remove image
docker history image:tag              # show layers

# Containers
docker run -d --name x image          # start detached
docker run -it --rm image bash        # interactive, auto-remove
docker ps -a                          # list all containers
docker logs -f name                   # follow logs
docker exec -it name bash             # shell into container
docker stop name && docker rm name    # stop and remove
docker stats                          # live resource usage
docker inspect name                   # full metadata

# Volumes
docker volume create vol              # create volume
docker volume ls                      # list volumes
docker volume inspect vol             # inspect volume
docker run -v vol:/data image         # use volume

# Networks
docker network create net             # create network
docker network ls                     # list networks
docker run --network net image        # connect to network

# Compose
docker compose up -d --build          # start all services
docker compose down --volumes         # stop and remove volumes
docker compose logs -f service        # follow service logs
docker compose exec service bash      # shell into service
docker compose ps                     # list services

# Cleanup
docker system df                      # show disk usage
docker system prune --all             # remove everything unused
```

### Dockerfile Instruction Reference

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image |
| `RUN` | Execute command (creates layer) |
| `COPY` | Copy files from context |
| `ADD` | Copy + extract tar / fetch URL |
| `ENV` | Runtime environment variable |
| `ARG` | Build-time variable |
| `WORKDIR` | Set working directory |
| `EXPOSE` | Document listening port |
| `ENTRYPOINT` | Fixed executable |
| `CMD` | Default args / command |
| `USER` | Set runtime user |
| `LABEL` | Metadata key=value |
| `VOLUME` | Declare mount point |
| `HEALTHCHECK` | Container health probe |
| `STOPSIGNAL` | Signal for graceful stop |
| `SHELL` | Change default shell |

### Port Reference

| Port | Protocol | Purpose |
|------|----------|---------|
| `2375` | TCP | Docker daemon (no TLS) — ⚠️ never expose! |
| `2376` | TCP | Docker daemon (TLS) |
| `2377` | TCP | Swarm cluster management |
| `4789` | UDP | Overlay network (VXLAN) |
| `7946` | TCP/UDP | Swarm node communication |
| `50000` | TCP | JNLP agent (Jenkins) |

---

*Docker version: 25.x+ recommended. Check with `docker version`.*
*Reference: [Docker Documentation](https://docs.docker.com) | [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) | [Compose Specification](https://compose-spec.io)*
