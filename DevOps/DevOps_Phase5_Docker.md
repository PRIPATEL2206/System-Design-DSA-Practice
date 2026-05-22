# DevOps Phase 5 — Docker

> **Learning Goal:** Master containerization from the ground up — understand what containers really are at the OS level, build production-grade images, manage data, network containers together, and prepare for orchestration with Kubernetes.

---

## Table of Contents

1. [The Problem Docker Solves](#1-the-problem-docker-solves)
2. [Containers vs Virtual Machines](#2-containers-vs-virtual-machines)
3. [Docker Architecture — How It Really Works](#3-docker-architecture--how-it-really-works)
4. [Installing Docker](#4-installing-docker)
5. [Docker Images — Deep Dive](#5-docker-images--deep-dive)
6. [Dockerfile — Writing Production-Grade Images](#6-dockerfile--writing-production-grade-images)
7. [Multi-Stage Builds](#7-multi-stage-builds)
8. [Docker Volumes — Persistent Data](#8-docker-volumes--persistent-data)
9. [Docker Networking](#9-docker-networking)
10. [Docker Compose — Multi-Container Applications](#10-docker-compose--multi-container-applications)
11. [Docker Registry & Image Management](#11-docker-registry--image-management)
12. [Container Lifecycle & Management](#12-container-lifecycle--management)
13. [Docker Security Best Practices](#13-docker-security-best-practices)
14. [Docker in Production — Real Patterns](#14-docker-in-production--real-patterns)
15. [Debugging Containers](#15-debugging-containers)
16. [Interview Mastery](#16-interview-mastery)

---

## 1. The Problem Docker Solves

### Beginner Explanation

Imagine you built an app on your laptop. It works perfectly. You send it to your teammate — it crashes. You deploy it to the server — it crashes differently. Why?

Because your laptop has Python 3.11, your teammate has 3.9, and the server has 3.8. Your laptop has a library installed globally that the server doesn't have. The server uses a different OS version. The database driver expects a specific version of OpenSSL.

**Docker packages your application AND its entire environment** into a single unit (a "container") that runs identically everywhere — your laptop, your colleague's machine, staging, production, the cloud.

### Real-World Analogy

Think of shipping containers (the metal boxes on cargo ships). Before standardized containers, loading a ship meant handling thousands of differently-shaped items — barrels, crates, bags. Each needed special handling.

Standardized containers changed everything: whatever is inside (electronics, food, machinery), the outside is always the same shape. Any crane can lift it, any truck can carry it, any ship can hold it.

Docker containers do the same for software: whatever's inside (Node.js app, Python ML model, Java microservice), the outside interface is always the same. Any server can run it.

### The Technical Problem

```
WITHOUT CONTAINERS:

Developer's Laptop          Production Server
─────────────────          ─────────────────
macOS 14                   Ubuntu 22.04
Python 3.11.5              Python 3.9.2
OpenSSL 3.1                OpenSSL 1.1.1
libpq 15.4                 libpq 12.0
Node 20                    Node 18
npm packages (global)      Different npm packages
~/.bashrc env vars         Different env vars

"Works on my machine" ≠ "Works in production"

WITH CONTAINERS:

Developer's Laptop          Production Server
─────────────────          ─────────────────
Docker Engine              Docker Engine
    │                          │
    ▼                          ▼
┌───────────────┐          ┌───────────────┐
│  Container    │          │  Container    │
│  Python 3.11  │   ═══    │  Python 3.11  │
│  OpenSSL 3.1  │ IDENTICAL│  OpenSSL 3.1  │
│  All deps     │          │  All deps     │
│  App code     │          │  App code     │
└───────────────┘          └───────────────┘

Same image → Same behavior → Everywhere
```

---

## 2. Containers vs Virtual Machines

### The Architecture Difference

```
VIRTUAL MACHINES:                        CONTAINERS:

┌─────────┐ ┌─────────┐ ┌─────────┐    ┌─────────┐ ┌─────────┐ ┌─────────┐
│  App A  │ │  App B  │ │  App C  │    │  App A  │ │  App B  │ │  App C  │
├─────────┤ ├─────────┤ ├─────────┤    ├─────────┤ ├─────────┤ ├─────────┤
│  Libs A │ │  Libs B │ │  Libs C │    │  Libs A │ │  Libs B │ │  Libs C │
├─────────┤ ├─────────┤ ├─────────┤    └────┬────┘ └────┬────┘ └────┬────┘
│Guest OS │ │Guest OS │ │Guest OS │         │           │           │
│(Ubuntu) │ │(CentOS) │ │(Debian) │    ┌───┴───────────┴───────────┴───┐
└────┬────┘ └────┬────┘ └────┬────┘    │        Container Runtime       │
     │           │           │          │         (Docker Engine)        │
┌────┴───────────┴───────────┴────┐    ├────────────────────────────────┤
│          HYPERVISOR              │    │         Host OS (Linux)        │
│    (VMware, KVM, Hyper-V)       │    ├────────────────────────────────┤
├─────────────────────────────────┤    │         Hardware               │
│         Host OS                  │    └────────────────────────────────┘
├─────────────────────────────────┤
│         Hardware                 │
└─────────────────────────────────┘

EACH VM: Full OS (1-20 GB each)         EACH CONTAINER: Only app + libs (10-500 MB)
BOOT TIME: 30 seconds - 5 minutes       BOOT TIME: < 1 second
OVERHEAD: Heavy (each has full kernel)   OVERHEAD: Minimal (shared kernel)
ISOLATION: Strong (hardware-level)       ISOLATION: Process-level (via kernel)
```

### Detailed Comparison

| Feature | Virtual Machine | Container |
|---------|----------------|-----------|
| Size | 1–20 GB | 10–500 MB |
| Boot time | 30s – 5 min | Milliseconds – seconds |
| Memory overhead | GBs per VM | MBs per container |
| CPU overhead | 5–15% hypervisor tax | Near-native performance |
| Isolation | Full OS + hardware virtualization | Kernel namespaces + cgroups |
| Security boundary | Strong (hardware) | Process-level (weaker) |
| Density | 5–20 VMs per host | 100–1000 containers per host |
| Portability | VM image format varies | OCI standard, runs anywhere |
| Startup speed | Minutes | Sub-second |

### When to Use VMs vs Containers

| Use Case | VMs | Containers |
|----------|-----|------------|
| Multi-tenant with untrusted workloads | ✅ | ❌ (use gVisor/Kata for stronger isolation) |
| Running different OS kernels | ✅ (Linux + Windows on same host) | ❌ (containers share host kernel) |
| Maximum resource efficiency | ❌ | ✅ |
| Microservices architecture | ❌ (too heavy) | ✅ |
| Development environments | Slow to spin up | ✅ Instant |
| Legacy applications | ✅ (need full OS) | Sometimes ✅ |
| CI/CD build environments | Slow | ✅ |

### How Containers Actually Work (Linux Internals)

Containers are NOT lightweight VMs. They are **regular Linux processes** with two kernel features applied:

**1. Namespaces** — Isolation (what a container can see):

| Namespace | Isolates |
|-----------|---------|
| PID | Process IDs — container sees only its own processes (PID 1 is the app) |
| NET | Network stack — container gets its own IP, ports, routing table |
| MNT | Filesystem — container has its own root filesystem |
| UTS | Hostname — container has its own hostname |
| IPC | Inter-process communication — semaphores, shared memory |
| USER | User IDs — root inside container ≠ root on host |

**2. cgroups** — Resource limits (what a container can use):

| cgroup | Controls |
|--------|---------|
| cpu | CPU time allocation |
| memory | RAM limit (OOM kill if exceeded) |
| blkio | Disk I/O bandwidth |
| pids | Maximum number of processes |

```bash
# A container is just a process with namespaces and cgroups applied:
# This is basically what Docker does internally:
unshare --pid --net --mount --uts --ipc --fork /bin/bash
# Now you're in a namespace — isolated from the host
```

---

## 3. Docker Architecture — How It Really Works

### Component Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Docker Client (CLI)                       │
│                  docker build, run, pull, push                   │
└───────────────────────────┬─────────────────────────────────────┘
                            │ REST API (Unix socket /var/run/docker.sock)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Docker Daemon (dockerd)                     │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │   Image     │  │  Container   │  │     Network          │   │
│  │  Management │  │  Management  │  │    Management        │   │
│  └──────┬──────┘  └──────┬───────┘  └──────────────────────┘   │
│         │                │                                      │
└─────────┼────────────────┼──────────────────────────────────────┘
          │                │
          │                ▼
          │    ┌───────────────────────┐
          │    │    containerd         │   ← High-level container runtime
          │    │  (manages lifecycle)  │
          │    └──────────┬────────────┘
          │               │
          │               ▼
          │    ┌───────────────────────┐
          │    │      runc             │   ← Low-level OCI runtime
          │    │ (creates the process  │     (actually makes the container)
          │    │  with namespaces +    │
          │    │  cgroups)             │
          │    └───────────────────────┘
          │
          ▼
┌───────────────────────┐
│   Image Registry      │
│  (Docker Hub, ECR,    │
│   GCR, GHCR)         │
└───────────────────────┘
```

### The Flow: `docker run nginx`

```
Step 1: CLI sends request to Docker daemon
Step 2: Daemon checks if "nginx" image exists locally
Step 3: If not → pulls from Docker Hub (registry)
Step 4: Daemon tells containerd to create a container
Step 5: containerd tells runc to create the isolated process
Step 6: runc sets up namespaces + cgroups + filesystem
Step 7: runc starts the process (nginx)
Step 8: Container is running — has its own PID 1, network, filesystem
```

### Key Components

| Component | Role |
|-----------|------|
| **Docker CLI** | Command-line interface you type commands into |
| **Docker Daemon (dockerd)** | Background service that manages everything |
| **containerd** | Industry-standard container runtime (manages lifecycle) |
| **runc** | OCI-compliant runtime that actually creates the container |
| **Docker Registry** | Remote storage for images (Docker Hub, ECR, etc.) |
| **Docker Socket** | `/var/run/docker.sock` — Unix socket the CLI talks to daemon through |

---

## 4. Installing Docker

### Ubuntu/Debian

```bash
# Remove old versions
sudo apt-get remove docker docker-engine docker.io containerd runc

# Set up the repository
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Set up the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify installation
docker --version
docker run hello-world

# Run docker without sudo (add user to docker group)
sudo usermod -aG docker $USER
# Log out and back in for this to take effect
```

### macOS / Windows

```bash
# macOS: Install Docker Desktop
# Download from https://www.docker.com/products/docker-desktop/
# Docker Desktop includes: Docker Engine + Docker CLI + Docker Compose + Kubernetes

# Windows: Install Docker Desktop
# Requires WSL2 backend (Windows Subsystem for Linux)
# Download and install Docker Desktop, enable WSL2 integration
```

### Verify Installation

```bash
docker version           # Client + server version info
docker info              # System-wide information
docker run hello-world   # Pull and run a test container
```

---

## 5. Docker Images — Deep Dive

### What is a Docker Image?

A Docker image is a **read-only template** containing:
- A filesystem (the base OS + your app + dependencies)
- Metadata (default command to run, environment variables, exposed ports)

An image is built from a **Dockerfile** and produces a **container** when you run it.

### Image Layers (Union Filesystem)

```
Docker Image = Stack of read-only layers

┌────────────────────────────────────────┐  Layer 5: COPY app code (2 MB)
├────────────────────────────────────────┤  Layer 4: RUN npm ci (150 MB)
├────────────────────────────────────────┤  Layer 3: COPY package*.json (1 KB)
├────────────────────────────────────────┤  Layer 2: RUN apt-get install (50 MB)
├────────────────────────────────────────┤  Layer 1: Base image (Node:20, 350 MB)
└────────────────────────────────────────┘

Each Dockerfile instruction creates a new layer.
Layers are cached — only changed layers are rebuilt.
Multiple images can SHARE layers (saves disk space).
```

**Why layers matter for performance:**

```dockerfile
# ❌ BAD — any code change rebuilds ALL dependencies (slow)
COPY . .
RUN npm ci

# ✅ GOOD — dependencies cached unless package.json changes (fast)
COPY package.json package-lock.json ./
RUN npm ci
COPY . .

# With the good approach:
# Change app code → only Layer 5 rebuilds (2 seconds)
# Change package.json → Layers 4+5 rebuild (60 seconds)
# Change base image → everything rebuilds (rare)
```

### Essential Image Commands

```bash
# Pull an image from registry
docker pull nginx:1.25
docker pull python:3.11-slim
docker pull ubuntu:22.04

# List local images
docker images
docker image ls

# Image details (layers, size, config)
docker image inspect nginx:1.25

# See layer history of an image
docker history nginx:1.25

# Remove images
docker rmi nginx:1.25
docker image prune          # Remove dangling (untagged) images
docker image prune -a       # Remove all unused images

# Search Docker Hub
docker search redis

# Tag an image (create new reference)
docker tag myapp:latest registry.example.com/myapp:v1.2.0
```

### Image Naming Convention

```
registry.example.com/organization/image-name:tag
│                    │              │          │
│                    │              │          └── Version (default: latest)
│                    │              └── Image name
│                    └── Namespace/org
└── Registry (default: docker.io)

Examples:
  nginx                          = docker.io/library/nginx:latest
  nginx:1.25                     = docker.io/library/nginx:1.25
  mycompany/api:v2.1.0           = docker.io/mycompany/api:v2.1.0
  ghcr.io/org/app:sha-abc123    = GitHub Container Registry
  123456.dkr.ecr.us-east-1.amazonaws.com/myapp:latest = AWS ECR
```

### Choosing Base Images

| Base Image | Size | Use Case |
|-----------|------|----------|
| `ubuntu:22.04` | 77 MB | Full-featured, debugging tools available |
| `debian:bookworm-slim` | 74 MB | Minimal Debian, good balance |
| `alpine:3.19` | 7 MB | Tiny! But uses musl libc (compatibility issues possible) |
| `python:3.11` | 1 GB | Full Python + Debian + build tools |
| `python:3.11-slim` | 130 MB | Python + minimal Debian (recommended) |
| `python:3.11-alpine` | 50 MB | Python + Alpine (small but compilation issues) |
| `node:20` | 1.1 GB | Full Node + Debian + build tools |
| `node:20-slim` | 200 MB | Node + minimal Debian |
| `node:20-alpine` | 130 MB | Node + Alpine |
| `gcr.io/distroless/base` | 20 MB | No shell, no package manager — maximum security |
| `scratch` | 0 MB | Empty — for statically compiled Go/Rust binaries |

**Production recommendation:** Use `-slim` variants. They're small enough without the compatibility risks of Alpine. Use `distroless` for maximum security when you don't need shell access.

---

## 6. Dockerfile — Writing Production-Grade Images

### Dockerfile Basics

```dockerfile
# Every Dockerfile starts with FROM
# This is the base layer your image builds on
FROM node:20-slim

# Set the working directory inside the container
# All subsequent commands run relative to this path
WORKDIR /app

# Copy files from your local machine into the image
COPY package.json package-lock.json ./

# Run a command during the BUILD (creates a new layer)
RUN npm ci --only=production

# Copy the rest of the application code
COPY . .

# Document which port the app listens on
# This is metadata only — does NOT actually expose the port
EXPOSE 3000

# Set environment variables
ENV NODE_ENV=production

# The command to run when a container starts from this image
CMD ["node", "server.js"]
```

### All Dockerfile Instructions

| Instruction | Purpose | Builds Layer? |
|-------------|---------|--------------|
| `FROM` | Base image | Yes |
| `RUN` | Execute command during build | Yes |
| `COPY` | Copy files from host to image | Yes |
| `ADD` | Like COPY + can extract archives + fetch URLs | Yes |
| `WORKDIR` | Set working directory | Yes (metadata) |
| `ENV` | Set environment variable | Yes (metadata) |
| `ARG` | Build-time variable (not in final image) | No |
| `EXPOSE` | Document port (metadata only) | No |
| `CMD` | Default command when container runs | No |
| `ENTRYPOINT` | Fixed command (CMD becomes arguments to it) | No |
| `USER` | Switch to non-root user | Yes (metadata) |
| `VOLUME` | Declare a mount point | Yes (metadata) |
| `LABEL` | Add metadata (author, version) | Yes (metadata) |
| `HEALTHCHECK` | Define health check command | No |
| `STOPSIGNAL` | Signal to send for graceful shutdown | No |

### CMD vs ENTRYPOINT

```dockerfile
# CMD — the default command (can be overridden by docker run)
CMD ["node", "server.js"]

# Running: docker run myapp               → executes: node server.js
# Running: docker run myapp bash           → executes: bash (overrides CMD)

# ENTRYPOINT — fixed command (cannot be easily overridden)
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]

# Running: docker run myapp               → executes: python app.py --port 8080
# Running: docker run myapp --port 9090   → executes: python app.py --port 9090
# (CMD becomes the default arguments to ENTRYPOINT)

# ENTRYPOINT + CMD pattern is ideal for:
# - CLI tools where the base command is fixed but flags vary
# - Applications with configurable arguments
```

### Shell Form vs Exec Form

```dockerfile
# EXEC FORM (recommended) — runs directly, no shell
CMD ["node", "server.js"]
# PID 1 = node (receives SIGTERM for graceful shutdown)

# SHELL FORM — runs via /bin/sh -c
CMD node server.js
# PID 1 = /bin/sh (node is a child process, may not get SIGTERM!)

# ALWAYS use exec form for CMD and ENTRYPOINT
# Shell form wraps your process in sh, which:
# 1. Doesn't forward signals (graceful shutdown broken)
# 2. Can't receive SIGTERM directly
# 3. Creates an unnecessary parent process
```

### Production-Grade Dockerfile — Node.js

```dockerfile
# ─── BUILD STAGE ────────────────────────────────────────────
FROM node:20-slim AS builder

WORKDIR /app

# Install dependencies (cached unless package files change)
COPY package.json package-lock.json ./
RUN npm ci

# Copy source and build
COPY . .
RUN npm run build

# ─── PRODUCTION STAGE ───────────────────────────────────────
FROM node:20-slim AS production

# Security: don't run as root
RUN groupadd -r appuser && useradd -r -g appuser -d /app appuser

WORKDIR /app

# Install only production dependencies
COPY package.json package-lock.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy built assets from builder stage
COPY --from=builder /app/dist ./dist

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

EXPOSE 3000

ENV NODE_ENV=production

CMD ["node", "dist/server.js"]
```

### Production-Grade Dockerfile — Python

```dockerfile
FROM python:3.11-slim AS builder

WORKDIR /app

# Install system dependencies needed for building
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Create virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ─── PRODUCTION STAGE ───────────────────────────────────────
FROM python:3.11-slim

# Install only runtime system deps (no gcc, no build tools)
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq5 curl && \
    rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN useradd -r -s /bin/false appuser

WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copy application code
COPY app/ ./app/

USER appuser

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Production-Grade Dockerfile — Go (Distroless)

```dockerfile
# ─── BUILD ──────────────────────────────────────────────────
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Cache dependencies
COPY go.mod go.sum ./
RUN go mod download

# Build static binary (no C dependencies)
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/main.go

# ─── PRODUCTION ─────────────────────────────────────────────
FROM gcr.io/distroless/static-debian12

# Copy only the binary — no shell, no package manager, nothing else
COPY --from=builder /app/server /server

EXPOSE 8080

USER nonroot:nonroot

ENTRYPOINT ["/server"]
```

**Result:** Final image is ~10 MB with no attack surface (no shell to exec into, no package manager to install exploits with).

### Dockerfile Best Practices Checklist

```
✅ Use specific base image tags (node:20.11-slim, NOT node:latest)
✅ Use multi-stage builds (build deps NOT in final image)
✅ Run as non-root user (USER appuser)
✅ Order layers: rarely-changing first, frequently-changing last
✅ Combine RUN commands to reduce layers (apt-get update && apt-get install && rm cache)
✅ Use .dockerignore to exclude unnecessary files
✅ Add HEALTHCHECK instruction
✅ Use exec form for CMD/ENTRYPOINT
✅ Pin dependency versions in requirements.txt / package-lock.json
✅ Clean package manager cache in the same layer it was created

❌ Don't use root user
❌ Don't use :latest tag in production
❌ Don't COPY unnecessary files (node_modules, .git, tests)
❌ Don't store secrets in the image (use runtime env vars or secrets)
❌ Don't install development dependencies in production image
❌ Don't use ADD when COPY works (ADD has implicit decompression/URL behavior)
```

### .dockerignore

```
# .dockerignore — files/dirs excluded from COPY commands
node_modules
npm-debug.log
.git
.gitignore
.env
.env.*
*.md
Dockerfile
docker-compose*.yml
.dockerignore
tests/
coverage/
.nyc_output/
.vscode/
__pycache__/
*.pyc
.pytest_cache/
```

---

## 7. Multi-Stage Builds

### Why Multi-Stage?

Without multi-stage, your production image contains:
- Build tools (gcc, make, webpack, tsc)
- Source code (not needed at runtime)
- Development dependencies (test frameworks, linters)
- Intermediate build files

This bloats image size (security risk + slow pulls) and increases attack surface.

Multi-stage builds let you use one stage to build, then copy ONLY the output to a clean final stage.

### Visualization

```
SINGLE-STAGE (Bad):
┌──────────────────────────────────────┐
│  Final Image: 1.2 GB                 │
│                                      │
│  - Node.js runtime         (200 MB)  │
│  - Build tools (gcc, g++)  (400 MB)  │
│  - node_modules (all)      (300 MB)  │
│  - Source TypeScript        (20 MB)  │
│  - Compiled JavaScript      (10 MB)  │
│  - Test files               (50 MB)  │
│  - .git directory          (200 MB)  │
└──────────────────────────────────────┘

MULTI-STAGE (Good):
┌──────────────────────────────────────┐    ┌───────────────────────────────┐
│  Builder Stage (thrown away)         │    │  Final Image: 180 MB          │
│                                      │    │                               │
│  - Node.js runtime                   │    │  - Node.js runtime   (150 MB) │
│  - Build tools                       │──► │  - Compiled JS        (10 MB) │
│  - ALL node_modules                  │    │  - Prod node_modules  (20 MB) │
│  - Source TypeScript                 │    │                               │
│  - Compiled JavaScript ──────────────│──► │  That's it. Nothing else.     │
└──────────────────────────────────────┘    └───────────────────────────────┘
```

### Multi-Stage Pattern

```dockerfile
# Stage 1: "builder" — has all build tools
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build    # Compiles TypeScript to JavaScript

# Stage 2: "production" — clean, minimal
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production   # Only production deps
COPY --from=builder /app/dist ./dist
# ^ Takes ONLY the compiled output from the builder stage

USER node
CMD ["node", "dist/index.js"]
```

### Advanced: Multiple Build Stages

```dockerfile
# Stage 1: dependencies
FROM node:20 AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# Stage 2: build (needs deps)
FROM deps AS builder
COPY . .
RUN npm run build

# Stage 3: testing (can run tests without polluting final image)
FROM builder AS tester
RUN npm test

# Stage 4: production (only if tests pass)
FROM node:20-slim AS production
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package*.json ./
USER node
CMD ["node", "dist/index.js"]
```

```bash
# Build only up to a specific stage
docker build --target builder -t myapp:dev .
docker build --target tester -t myapp:test .
docker build --target production -t myapp:prod .
```

---

## 8. Docker Volumes — Persistent Data

### The Problem

Containers are **ephemeral** — when you delete a container, all data inside it is lost. But databases, uploads, and logs need to persist.

```
Container lifecycle:
  docker run mysql    →  Container created, data stored inside
  docker stop mysql   →  Container stopped, data still exists (in stopped container)
  docker rm mysql     →  Container DELETED, data GONE FOREVER

Solution: Volumes store data OUTSIDE the container lifecycle
```

### Three Types of Data Storage

```
┌─────────────────────────────────────────────────────────┐
│                    HOST MACHINE                          │
│                                                         │
│  ┌─────────────────────────────────────────────┐        │
│  │             Docker Engine                   │        │
│  │                                             │        │
│  │  ┌───────────┐                              │        │
│  │  │ Container │                              │        │
│  │  │           │                              │        │
│  │  │ /app/data ◄───── 1. VOLUME              │        │
│  │  │           │      (managed by Docker)     │        │
│  │  │ /app/code ◄───── 2. BIND MOUNT          │───────►│ /home/user/project/
│  │  │           │      (host directory)        │        │
│  │  │ /tmp/     ◄───── 3. TMPFS               │        │
│  │  │           │      (RAM only, no disk)     │        │
│  │  └───────────┘                              │        │
│  │                                             │        │
│  └─────────────────────────────────────────────┘        │
│                                                         │
│  /var/lib/docker/volumes/mydata/_data  ← Where Docker   │
│                                          stores volumes  │
└─────────────────────────────────────────────────────────┘
```

### 1. Named Volumes (Recommended for Data Persistence)

```bash
# Create a named volume
docker volume create postgres_data

# Run container with volume mounted
docker run -d \
    --name postgres \
    -v postgres_data:/var/lib/postgresql/data \
    -e POSTGRES_PASSWORD=secret \
    postgres:16

# Data persists even if container is deleted:
docker rm -f postgres
# postgres_data volume still exists with all data

# Start new container with same volume:
docker run -d \
    --name postgres-new \
    -v postgres_data:/var/lib/postgresql/data \
    -e POSTGRES_PASSWORD=secret \
    postgres:16
# All data is still there!

# Volume management commands
docker volume ls                    # List all volumes
docker volume inspect postgres_data # Show volume details (mountpoint, etc.)
docker volume rm postgres_data      # Delete volume
docker volume prune                 # Remove ALL unused volumes (dangerous!)
```

### 2. Bind Mounts (For Development — Mount Host Directories)

```bash
# Mount current directory into container (for development)
docker run -d \
    --name dev-server \
    -v $(pwd):/app \
    -v /app/node_modules \
    -p 3000:3000 \
    node:20 \
    sh -c "cd /app && npm install && npm run dev"

# $(pwd):/app     → Your local source code is LIVE inside the container
# Changes on host → immediately visible in container (hot reload!)
# /app/node_modules → Anonymous volume to prevent host node_modules from overriding container's

# Windows (PowerShell):
docker run -v ${PWD}:/app ...

# Windows (CMD):
docker run -v %cd%:/app ...
```

### 3. tmpfs Mounts (In-Memory Only)

```bash
# Data stored in RAM — ultra fast, disappears when container stops
docker run -d \
    --name redis-cache \
    --tmpfs /data:rw,size=100m \
    redis:7

# Use cases:
# - Temporary caches that don't need persistence
# - Sensitive data you never want to hit disk
# - High-speed scratch space
```

### Volume Use Cases

| Scenario | Storage Type | Example |
|----------|-------------|---------|
| Database files | Named volume | PostgreSQL `/var/lib/postgresql/data` |
| File uploads | Named volume | User uploads at `/app/uploads` |
| Development code | Bind mount | Source code for hot-reload |
| Config files | Bind mount (read-only) | `nginx.conf:/etc/nginx/nginx.conf:ro` |
| Temporary cache | tmpfs | Session data in Redis |
| Secrets | tmpfs | Certificates loaded at runtime |
| Shared between containers | Named volume | Two containers reading same data |

---

## 9. Docker Networking

### Default Networks

When Docker is installed, it creates three networks automatically:

```bash
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
a1b2c3d4e5f6   bridge    bridge    local    ← Default for containers
f6e5d4c3b2a1   host      host      local    ← Container shares host network
0000000000     none      null      local    ← No networking (isolated)
```

### Network Drivers Explained

```
1. BRIDGE (default) — Containers on same bridge can talk to each other
   
   ┌────────────────────────────────────────────────┐
   │              HOST                               │
   │                                                 │
   │  ┌──────────────────────────────────────┐       │
   │  │       docker0 bridge (172.17.0.1)    │       │
   │  │                                      │       │
   │  │   Container A ◄─────► Container B    │       │
   │  │   172.17.0.2          172.17.0.3     │       │
   │  └──────────────────────────────────────┘       │
   │                                                 │
   │  Port mapping: -p 8080:80                       │
   │  Host:8080 → Container A:80                     │
   └────────────────────────────────────────────────┘

2. HOST — Container uses host's network directly (no isolation)
   Container shares host's IP and ports
   No port mapping needed (-p flag has no effect)
   Best performance, least isolation
   
3. NONE — Container has no network at all
   Completely isolated from all networking
   Use case: batch processing that needs no network

4. OVERLAY — Spans multiple Docker hosts (Docker Swarm)
   Containers on different machines communicate as if on same network
   Used in Docker Swarm / multi-host deployments
```

### User-Defined Bridge Networks (Best Practice)

```bash
# Create a custom network
docker network create myapp-network

# Run containers on the custom network
docker run -d --name api --network myapp-network myapi:latest
docker run -d --name db --network myapp-network postgres:16

# KEY FEATURE: Automatic DNS resolution by container name!
# From inside "api" container:
ping db              # Resolves to the db container's IP automatically
curl http://db:5432  # Can connect using container NAME as hostname

# The default bridge network does NOT have DNS resolution
# Custom bridge networks DO — this is why you always use them
```

### DNS Resolution in Docker Networks

```
Custom bridge network: "myapp-network"

┌───────────────────────────────────────────────────────┐
│                                                       │
│  Container: "api"              Container: "db"        │
│  IP: 172.18.0.2              IP: 172.18.0.3          │
│                                                       │
│  # Inside "api" container:                            │
│  ping db              → 172.18.0.3 ✅                │
│  curl http://redis    → 172.18.0.4 ✅                │
│                                                       │
│  Docker's embedded DNS server resolves                │
│  container names to their IPs automatically           │
│                                                       │
└───────────────────────────────────────────────────────┘

This is how containers discover each other — by NAME, not IP.
In your app config: DATABASE_HOST=db (not 172.18.0.3)
```

### Port Mapping

```bash
# Syntax: -p HOST_PORT:CONTAINER_PORT
docker run -p 8080:80 nginx

#   Host port 8080 → forwarded to → Container port 80
#   Access at: http://localhost:8080

# Multiple port mappings
docker run -p 8080:80 -p 443:443 nginx

# Bind to specific interface only
docker run -p 127.0.0.1:8080:80 nginx     # Only accessible from localhost
docker run -p 0.0.0.0:8080:80 nginx        # Accessible from any interface (default)

# Random host port (Docker picks one)
docker run -p 80 nginx
docker port <container-id>   # See which host port was assigned
```

### Network Commands

```bash
# List networks
docker network ls

# Inspect a network (see connected containers and their IPs)
docker network inspect myapp-network

# Connect a running container to a network
docker network connect myapp-network existing-container

# Disconnect a container
docker network disconnect myapp-network container-name

# Remove a network
docker network rm myapp-network

# Remove all unused networks
docker network prune
```

---

## 10. Docker Compose — Multi-Container Applications

### What is Docker Compose?

Real applications aren't a single container. A typical app needs:
- Application server
- Database
- Cache (Redis)
- Message queue (RabbitMQ)
- Reverse proxy (Nginx)

Docker Compose lets you define and run multi-container applications with a single YAML file and a single command.

### Complete Docker Compose Example

```yaml
# docker-compose.yml (or compose.yml for v2)

services:
  # ─── APPLICATION ────────────────────────────────────────────
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: production      # Multi-stage: build only production stage
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://app:secret@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy    # Wait for DB to be actually ready
      redis:
        condition: service_started
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"

  # ─── DATABASE ──────────────────────────────────────────────
  db:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro   # Init script
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"    # Expose for local development tools
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # ─── CACHE ─────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - backend
    restart: unless-stopped

  # ─── REVERSE PROXY ─────────────────────────────────────────
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      - api
    networks:
      - backend
    restart: unless-stopped

  # ─── MONITORING ─────────────────────────────────────────────
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - backend

# ─── VOLUMES (Persistent data) ────────────────────────────────
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  prometheus_data:
    driver: local

# ─── NETWORKS ─────────────────────────────────────────────────
networks:
  backend:
    driver: bridge
```

### Docker Compose Commands

```bash
# Start all services (detached mode)
docker compose up -d

# Start and rebuild images
docker compose up -d --build

# Stop all services
docker compose down

# Stop and remove volumes (destroys data!)
docker compose down -v

# View logs
docker compose logs -f              # All services, following
docker compose logs -f api          # Single service

# List running services
docker compose ps

# Execute command in running service
docker compose exec api sh          # Shell into api container
docker compose exec db psql -U app myapp   # Connect to DB

# Run a one-off command (creates temporary container)
docker compose run --rm api npm test

# Scale a service (run multiple instances)
docker compose up -d --scale api=3

# Restart a single service
docker compose restart api

# Pull latest images
docker compose pull

# View resource usage
docker compose top
```

### Docker Compose for Development vs Production

```yaml
# docker-compose.yml — base (shared)
services:
  api:
    image: myapp:latest
    networks:
      - backend

# docker-compose.override.yml — auto-loaded for development
services:
  api:
    build: .
    volumes:
      - .:/app                    # Hot-reload source code
      - /app/node_modules         # Preserve container's node_modules
    ports:
      - "3000:3000"
      - "9229:9229"              # Node.js debugger port
    environment:
      - NODE_ENV=development
    command: npm run dev          # Use nodemon/ts-node-dev

# docker-compose.prod.yml — production overrides
services:
  api:
    image: registry.example.com/myapp:${GIT_SHA}
    restart: always
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 1G
    environment:
      - NODE_ENV=production
```

```bash
# Development (auto-loads override):
docker compose up

# Production:
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### depends_on and Startup Order

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy      # Wait for healthcheck to pass
      redis:
        condition: service_started      # Just wait for container to start
      migrations:
        condition: service_completed_successfully  # Wait for it to finish and exit 0

  migrations:
    build: .
    command: npm run migrate
    depends_on:
      db:
        condition: service_healthy
```

---

## 11. Docker Registry & Image Management

### What is a Registry?

A Docker registry is a **storage and distribution system** for Docker images. It's like npm for Node packages or Maven Central for JARs — but for container images.

### Registry Options

| Registry | Type | Best For |
|----------|------|----------|
| Docker Hub | Public SaaS | Open source projects, public images |
| Amazon ECR | AWS native | AWS workloads |
| Google GCR / Artifact Registry | GCP native | GCP workloads |
| Azure ACR | Azure native | Azure workloads |
| GitHub Container Registry (ghcr.io) | GitHub native | GitHub Actions CI/CD |
| Harbor | Self-hosted | Private registry with vulnerability scanning |
| JFrog Artifactory | Enterprise | Multi-format (Docker + Maven + npm) |

### Working with Registries

```bash
# ─── Docker Hub ────────────────────────────────────────────
docker login
docker push myusername/myapp:v1.0.0
docker pull myusername/myapp:v1.0.0

# ─── AWS ECR ──────────────────────────────────────────────
# Login (token valid for 12 hours)
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name myapp

# Push
docker tag myapp:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# ─── GitHub Container Registry ─────────────────────────────
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
docker push ghcr.io/org/myapp:sha-abc123
```

### Image Tagging Strategy

```bash
# Tag with git SHA (traceability)
docker tag myapp:latest registry.example.com/myapp:sha-abc1234

# Tag with semantic version
docker tag myapp:latest registry.example.com/myapp:1.2.3

# Tag with branch name (mutable — for development)
docker tag myapp:latest registry.example.com/myapp:develop

# Multi-tag on build:
docker build \
    -t registry.example.com/myapp:sha-abc1234 \
    -t registry.example.com/myapp:1.2.3 \
    -t registry.example.com/myapp:latest \
    .
```

---

## 12. Container Lifecycle & Management

### Container States

```
                    docker create
         ┌────────────────────────────┐
         │                            ▼
     ┌───────┐    docker run     ┌─────────┐
     │       │ ─────────────────►│         │
     │ Image │                   │ Created │
     │       │                   │         │
     └───────┘                   └────┬────┘
                                      │ docker start
                                      ▼
                                 ┌─────────┐
                                 │         │
                      ┌─────────►│ Running │◄──────── docker restart
                      │          │         │
                      │          └────┬────┘
                      │               │
                      │        docker stop / kill
                      │        (or process exits)
                      │               │
                      │               ▼
                      │          ┌─────────┐
                      │          │         │
                      └──────────│ Stopped │
                    docker start │(Exited) │
                                 └────┬────┘
                                      │ docker rm
                                      ▼
                                 ┌─────────┐
                                 │ Removed │
                                 │ (Gone)  │
                                 └─────────┘
```

### Essential Container Commands

```bash
# ─── RUN ─────────────────────────────────────────────────────
docker run -d --name myapp -p 8080:80 nginx:1.25
#  -d           = detached (background)
#  --name       = give it a name (otherwise random name)
#  -p 8080:80   = port mapping
#  nginx:1.25   = image:tag

# Run interactively (for debugging / one-off tasks)
docker run -it --rm ubuntu:22.04 bash
#  -it    = interactive + tty (gives you a shell)
#  --rm   = auto-remove container on exit

# Run with environment variables
docker run -d --name api \
    -e DATABASE_URL=postgresql://user:pass@db:5432/app \
    -e SECRET_KEY=abc123 \
    myapp:latest

# Run with resource limits
docker run -d --name api \
    --memory=512m \
    --cpus=1.5 \
    myapp:latest

# ─── INSPECT ─────────────────────────────────────────────────
docker ps                    # Running containers
docker ps -a                 # ALL containers (including stopped)
docker logs myapp            # View stdout/stderr logs
docker logs -f myapp         # Follow logs (like tail -f)
docker logs --tail 100 myapp # Last 100 lines
docker inspect myapp         # Full JSON details (IP, mounts, config)
docker stats                 # Live resource usage (CPU, memory, I/O)
docker top myapp             # Processes inside container

# ─── INTERACT ────────────────────────────────────────────────
docker exec -it myapp bash   # Open shell INSIDE running container
docker exec myapp ls /app    # Run single command inside container
docker cp myapp:/app/log.txt ./log.txt  # Copy file OUT of container
docker cp ./config.json myapp:/app/     # Copy file INTO container

# ─── LIFECYCLE ────────────────────────────────────────────────
docker stop myapp            # Graceful shutdown (SIGTERM, then SIGKILL after 10s)
docker kill myapp            # Immediate kill (SIGKILL)
docker start myapp           # Start a stopped container
docker restart myapp         # Stop + Start
docker rm myapp              # Remove stopped container
docker rm -f myapp           # Force remove (even if running)

# ─── CLEANUP ─────────────────────────────────────────────────
docker container prune       # Remove all stopped containers
docker system prune          # Remove stopped containers + unused images + unused networks
docker system prune -a       # + remove ALL unused images (not just dangling)
docker system df             # Show disk usage
```

### Resource Limits

```bash
# Memory limit — container gets OOM killed if it exceeds this
docker run -d --memory=512m --memory-swap=1g myapp

# CPU limit — container gets 1.5 CPU cores max
docker run -d --cpus=1.5 myapp

# CPU shares (relative weight — used when competing for CPU)
docker run -d --cpu-shares=512 myapp   # Default is 1024

# Combined limits (production-ready)
docker run -d \
    --memory=1g \
    --memory-swap=1g \
    --cpus=2 \
    --pids-limit=100 \
    --restart=unless-stopped \
    myapp:latest
```

---

## 13. Docker Security Best Practices

### 1. Never Run as Root

```dockerfile
# ✅ Create and switch to non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

# ❌ Running as root (default) — container compromise = root access
```

### 2. Use Read-Only Filesystems

```bash
# Container's filesystem is read-only — can't write malicious files
docker run --read-only --tmpfs /tmp --tmpfs /var/run myapp
```

### 3. Drop Capabilities

```bash
# Drop ALL Linux capabilities, add back only what's needed
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp

# Never use --privileged (gives ALL capabilities + device access)
# docker run --privileged   ← NEVER DO THIS unless you truly need it
```

### 4. Scan Images for Vulnerabilities

```bash
# Trivy (recommended — fast, accurate)
trivy image myapp:latest

# Docker Scout (built into Docker Desktop)
docker scout cves myapp:latest

# Grype (by Anchore)
grype myapp:latest
```

### 5. Use Minimal Base Images

```
Attack surface comparison:
  ubuntu:22.04    → 117 packages, 22 known CVEs
  debian:slim     → 80 packages, 14 known CVEs
  alpine:3.19     → 14 packages, 2 known CVEs
  distroless      → 0 shell, 0 package manager, 0 CVEs typical
  scratch         → literally empty
```

### 6. Don't Store Secrets in Images

```dockerfile
# ❌ WRONG — secret is baked into a layer (even if you delete it later)
COPY .env /app/.env
RUN source .env && do-something
RUN rm .env    # Too late! The secret is still in a previous layer!

# ✅ CORRECT — use build-time secrets (Docker BuildKit)
RUN --mount=type=secret,id=my_secret cat /run/secrets/my_secret

# ✅ CORRECT — pass secrets at RUNTIME via environment
docker run -e API_KEY=abc123 myapp
# Or better: use Docker secrets / Vault / cloud secrets manager
```

### 7. Pin Image Digests for Supply Chain Security

```dockerfile
# Tag can be moved to point to a different image (mutable)
FROM node:20-slim

# Digest is immutable — cryptographic hash of the exact image
FROM node:20-slim@sha256:a1b2c3d4e5f6...

# This protects against:
# - Compromised registry pushing malicious :latest
# - Typosquatting attacks
# - Tag mutability
```

---

## 14. Docker in Production — Real Patterns

### Pattern 1: Container Log Management

```bash
# Docker captures stdout/stderr from the container process
# In production, forward logs to a centralized system:

# Option 1: Docker logging driver
docker run --log-driver=fluentd --log-opt fluentd-address=fluentd:24224 myapp

# Option 2: Sidecar pattern (common in Kubernetes)
# A log-shipper container reads logs from a shared volume

# Option 3: Application-level logging to stdout (12-Factor App)
# App logs to stdout → Docker captures → log driver forwards to ELK/Loki/CloudWatch
```

### Pattern 2: Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1
```

```bash
# Docker marks container as "unhealthy" if healthcheck fails
# Orchestrators (Swarm, Kubernetes) use this to restart unhealthy containers
docker inspect --format='{{.State.Health.Status}}' myapp
# Output: healthy / unhealthy / starting
```

### Pattern 3: Graceful Shutdown

```javascript
// Node.js — handle SIGTERM for graceful shutdown
process.on('SIGTERM', async () => {
    console.log('SIGTERM received, shutting down gracefully');
    // Stop accepting new connections
    server.close();
    // Finish in-flight requests (with timeout)
    await drainConnections(30000);
    // Close database connections
    await db.close();
    process.exit(0);
});
```

```dockerfile
# Docker sends SIGTERM, waits 10 seconds, then SIGKILL
# Extend the grace period if your app needs more time:
STOPSIGNAL SIGTERM
# docker stop --time=30 myapp   ← give 30 seconds before SIGKILL
```

### Pattern 4: Container Orchestration Readiness

```yaml
# Kubernetes-ready Dockerfile patterns:
# 1. Single process per container (PID 1 = your app)
# 2. Log to stdout/stderr
# 3. Handle SIGTERM
# 4. Include health/readiness endpoints
# 5. Externalize all config via environment variables
# 6. Stateless (store state in external DB/cache)
# 7. Fast startup time
```

---

## 15. Debugging Containers

### Common Debugging Scenarios

```bash
# Container won't start — check logs
docker logs myapp
docker logs --tail 50 myapp

# Container starts but crashes — check exit code
docker inspect myapp --format='{{.State.ExitCode}}'
# Exit 0 = normal, 1 = app error, 137 = OOM killed, 139 = segfault

# OOM killed? Check memory usage
docker stats myapp
docker inspect myapp --format='{{.State.OOMKilled}}'

# Shell into a RUNNING container
docker exec -it myapp /bin/bash
docker exec -it myapp /bin/sh      # Alpine images don't have bash

# Shell into a STOPPED/CRASHED container (can't exec — it's not running)
# Workaround: start a new container from the same image
docker run -it --rm myapp:latest /bin/bash

# See what changed in the container filesystem
docker diff myapp
# A = added, C = changed, D = deleted

# Export container filesystem for inspection
docker export myapp > myapp-filesystem.tar

# Inspect networking
docker exec myapp cat /etc/hosts
docker exec myapp cat /etc/resolv.conf
docker exec myapp netstat -tlnp

# Check what's using disk space inside an image
docker history myapp:latest --human --format "{{.Size}}\t{{.CreatedBy}}"
```

### Debug with a Sidecar Container

```bash
# If your production image is distroless (no shell), debug with a sidecar:
# 1. Start a debug container sharing the network namespace:
docker run -it --rm --network container:myapp nicolaka/netshoot bash

# Now you can run curl, nslookup, tcpdump, etc. against "myapp" container
# without installing tools inside the production image
```

### Build Debugging

```bash
# See build output (not cached)
docker build --no-cache --progress=plain .

# Build up to a specific stage and inspect
docker build --target builder -t debug:latest .
docker run -it --rm debug:latest /bin/bash

# Show full build history of an image
docker history --no-trunc myapp:latest
```

---

## 16. Interview Mastery

---

### Beginner Interview Questions

---

**Q1: What is Docker and why do we use it?**

**Perfect Answer:**

"Docker is a platform that packages applications and their entire runtime environment into standardized units called containers. A container includes the application code, runtime, system libraries, and settings — everything needed to run.

We use Docker because it solves the 'works on my machine' problem. Without Docker, an application might work on a developer's laptop but fail in production because of different OS versions, library versions, or configurations. With Docker, the container is identical everywhere — development, testing, staging, production.

The key benefits are:
1. **Consistency** — same image runs everywhere
2. **Isolation** — containers don't interfere with each other
3. **Efficiency** — containers share the host OS kernel, much lighter than VMs
4. **Speed** — containers start in milliseconds, VMs take minutes
5. **Portability** — run on any machine with Docker installed
6. **Reproducibility** — the Dockerfile is version-controlled documentation of the environment"

---

**Q2: What is the difference between a Docker image and a container?**

**Perfect Answer:**

"An image is a read-only template — it's like a class in object-oriented programming. A container is a running instance of that image — it's like an object created from that class.

Specifically:
- **Image:** Immutable. Built from a Dockerfile. Contains the filesystem, libraries, and metadata. Can be stored in a registry and shared.
- **Container:** Mutable. Created from an image. Has its own writable layer on top. Has a lifecycle (created, running, stopped, removed).

You can create multiple containers from the same image — just like you can create multiple objects from the same class. Each container has its own state, filesystem changes, network, and process space."

---

**Q3: What is a Dockerfile? Explain the key instructions.**

**Perfect Answer:**

"A Dockerfile is a text file containing instructions to build a Docker image. It's essentially a recipe — each instruction adds a layer to the image.

Key instructions:
- **FROM** — the base image to start from (every Dockerfile must start with this)
- **WORKDIR** — sets the working directory for subsequent instructions
- **COPY** — copies files from the build context into the image
- **RUN** — executes a command during build (installs packages, compiles code)
- **ENV** — sets environment variables
- **EXPOSE** — documents which port the app listens on (metadata only)
- **CMD** — the default command to run when a container starts
- **ENTRYPOINT** — the fixed command (CMD becomes its arguments)
- **USER** — switches to a non-root user (security best practice)

The order matters: Docker caches layers. Put rarely-changing instructions first (install dependencies) and frequently-changing ones last (copy source code). This maximizes cache hits and speeds up builds."

---

### Intermediate Interview Questions

---

**Q4: Explain Docker layers and the union filesystem. How do they affect build performance?**

**Perfect Answer:**

"Each instruction in a Dockerfile creates a new read-only layer. These layers are stacked using a union filesystem (overlay2 on modern Linux). When you read a file, Docker looks from the top layer down until it finds the file.

This architecture has major performance implications:

**Caching:** Docker caches each layer. If a layer and all layers before it haven't changed, Docker reuses the cached version. This is why we order Dockerfiles with rarely-changing instructions first:
```
COPY package.json .     ← Only changes when deps change
RUN npm install         ← Cached unless package.json changed
COPY . .                ← Changes on every code change
```

**Sharing:** Multiple images that share the same base image share those layers on disk. Ten containers using `node:20` don't store ten copies of Node.js — they share one set of base layers.

**Layer size:** Each RUN command creates a layer. If you install a build tool and then remove it in a SEPARATE RUN command, both layers exist — the image is still large. You must install and clean up in the SAME RUN command to keep the final layer small:
```
RUN apt-get install gcc && make && apt-get purge gcc && rm -rf /var/lib/apt/lists/*
```

**Build performance optimization:** I structure Dockerfiles so that dependency installation (slow, rarely changes) is early, and source code copy (fast, changes often) is late. A code change only rebuilds the last few layers."

---

**Q5: What is a multi-stage build and when would you use it?**

**Perfect Answer:**

"A multi-stage build uses multiple FROM instructions in a single Dockerfile. Each FROM starts a new build stage. You can COPY artifacts from one stage to another, but only the final stage becomes the shipped image.

Use case: separate build-time dependencies from runtime:
```dockerfile
FROM node:20 AS builder      # Stage 1: full Node with build tools
COPY . .
RUN npm ci && npm run build  # Compile TypeScript, bundle, etc.

FROM node:20-slim            # Stage 2: minimal runtime
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ['node', 'dist/index.js']
```

The final image contains only the compiled output and production dependencies — no TypeScript compiler, no source maps, no devDependencies. Image goes from 1.2 GB to 180 MB.

I use multi-stage builds for:
- Compiled languages (Go, Rust, Java) — build in one stage, copy the binary to `scratch` or `distroless`
- Frontend apps — build with webpack in one stage, serve static files with nginx in the final stage
- Security — the final image has minimal attack surface (no build tools to exploit)

Multi-stage also enables running tests as a build stage — if tests fail, the build fails."

---

**Q6: How does Docker networking work? How do containers communicate?**

**Perfect Answer:**

"Docker creates a virtual network infrastructure using Linux network namespaces and virtual bridges.

**Default bridge network:** When Docker installs, it creates a `docker0` bridge. Containers connect to this bridge with a virtual ethernet pair (veth). Each container gets its own IP (e.g., 172.17.0.x). Containers on the same bridge can communicate via IP, but NOT by name.

**User-defined bridge (recommended):** When you create a custom network (`docker network create mynet`), Docker provides automatic DNS resolution. Containers can reach each other by name: `curl http://api:3000` from one container reaches the `api` container. This is the standard pattern.

**Port mapping:** Containers are isolated by default. To expose a container's port to the host, use `-p 8080:80`. Docker sets up iptables rules to NAT traffic from host:8080 to container:80.

**Network types:**
- `bridge` — default, containers share a virtual LAN
- `host` — container uses host's network directly (no isolation, maximum performance)
- `none` — no networking at all
- `overlay` — multi-host networking (Docker Swarm)

**In production with Docker Compose:** I always create a named network so services can resolve each other by service name. The database connection string uses the service name, not an IP: `DATABASE_HOST=db`."

---

### Advanced Interview Questions

---

**Q7: How do you optimize Docker image size and build time for a production application?**

**Perfect Answer:**

"I approach optimization on two axes — final image size and build speed:

**Image Size Optimization:**

1. **Multi-stage builds** — keep build tools out of the final image. This alone typically reduces size 5–10x.

2. **Minimal base images** — Use `-slim` variants (removes man pages, docs, unused locales). For Go/Rust, use `scratch` or `distroless` for a ~10 MB final image.

3. **Reduce layers** — Combine related RUN commands. Most importantly, clean up in the same layer:
```
RUN apt-get update && apt-get install -y gcc && \
    make && apt-get purge -y gcc && rm -rf /var/lib/apt/lists/*
```

4. **`.dockerignore`** — Exclude `.git`, `node_modules`, tests, docs from build context. Reduces context transfer time AND prevents accidentally COPY-ing them.

5. **Production-only dependencies** — `npm ci --only=production`, pip install without dev extras.

**Build Speed Optimization:**

1. **Layer ordering** — Put rarely-changing layers first (FROM, install system deps, install app deps), frequently-changing last (COPY source code).

2. **BuildKit cache mounts** — Cache package manager downloads across builds:
```
RUN --mount=type=cache,target=/root/.npm npm ci
```

3. **Registry-based layer caching** — In CI/CD, push layers to registry as cache:
```
docker build --cache-from registry/myapp:cache --cache-to registry/myapp:cache .
```

4. **Parallel builds** — Docker BuildKit builds independent stages in parallel.

In practice, I've reduced a 2.1 GB image to 85 MB (Go binary in distroless) and reduced build time from 8 minutes to 45 seconds (layer caching + BuildKit)."

---

**Q8: Explain the security implications of running Docker in production. What are the attack vectors and how do you mitigate them?**

**Perfect Answer:**

"Docker's security model has several layers, each with specific threats and mitigations:

**1. Kernel sharing:** All containers share the host kernel. A kernel exploit in one container compromises the host and all containers.
- Mitigation: Keep host kernel patched. Use gVisor or Kata Containers for stronger isolation. Run Docker on a minimal host OS (CoreOS, Bottlerocket).

**2. Root inside container:** By default, container processes run as root. If an attacker escapes the container, they have root on the host.
- Mitigation: Always use `USER nonroot` in Dockerfile. Enable user namespace remapping (`userns-remap`). Drop all capabilities (`--cap-drop=ALL`).

**3. Docker socket exposure:** `/var/run/docker.sock` gives full control over Docker. Mounting it into a container (common in CI/CD) means that container can create privileged containers.
- Mitigation: Never mount the socket unless absolutely necessary. Use rootless Docker or limited-access proxies.

**4. Supply chain attacks:** Base images or third-party images could contain malware.
- Mitigation: Pin image digests (not just tags). Scan images with Trivy/Snyk. Use signed images (Docker Content Trust/Notary). Build from official base images only.

**5. Secrets in images:** Secrets in environment variables or layers can be extracted.
- Mitigation: Use BuildKit secrets (`--mount=type=secret`). Never store secrets in Dockerfiles. Use runtime secrets management (Vault, AWS Secrets Manager).

**6. Container escape:** Misconfigured containers (`--privileged`, device mounts) enable escape.
- Mitigation: Never use `--privileged`. Use seccomp profiles, AppArmor/SELinux. Read-only root filesystem. Limit capabilities.

In production, I follow the principle of least privilege: minimal base image, non-root user, dropped capabilities, read-only filesystem, resource limits, and regular vulnerability scanning in CI/CD."

---

**Q9: A container is consuming too much memory and getting OOM killed. How do you debug and resolve this?**

**Perfect Answer:**

"OOM kill means the container exceeded its memory limit. Here's my systematic approach:

**1. Confirm the OOM kill:**
```bash
docker inspect container_id --format='{{.State.OOMKilled}}'
# true = OOM killed

# Or check host system logs
dmesg | grep -i 'oom\|killed'
```

**2. Determine current memory usage:**
```bash
docker stats container_id   # Live CPU/memory usage
# If it's steadily growing → likely a memory leak
# If it spikes → could be load-related or a specific code path
```

**3. Profile inside the container:**
```bash
# Node.js: use --max-old-space-size and heap snapshots
docker exec container_id node --v8-options | grep heap

# Java: check heap settings
docker exec container_id java -XX:+PrintFlagsFinal | grep -i heap

# General: check /proc/meminfo inside container
docker exec container_id cat /proc/meminfo
```

**4. Common causes and fixes:**

- **Memory leak in application code:** Take heap snapshots at intervals, compare them, find growing objects. Fix the leak.

- **Memory limit too low:** The app legitimately needs more memory. Increase the limit: `--memory=2g`

- **JVM/Node not respecting container limits:** JVM sees host memory by default, not container limit. Fix: `java -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0`

- **Off-heap memory (file caches, mmapped files):** These don't show in application memory profiling. Use `--memory-swap` equal to `--memory` to prevent swap and identify the true memory consumer.

**5. Prevention:**
- Set memory limits in production (always)
- Monitor memory over time (Prometheus + Grafana)
- Alert at 80% of limit, not 100%
- Run stress tests to determine actual memory requirements before setting limits"

---

### Scenario-Based Questions

---

**Q10: Your Docker build takes 15 minutes in CI/CD. How do you reduce it to under 2 minutes?**

**Perfect Answer:**

"Fifteen-minute builds are almost always caused by cache invalidation. Here's my approach:

**1. Profile the build (identify the bottleneck):**
```bash
docker build --progress=plain . 2>&1 | grep -E 'CACHED|DONE'
```
Find which steps are rebuilding vs using cache.

**2. Fix layer ordering (biggest impact):**
If COPY . . comes before RUN npm install, every code change invalidates the npm install cache.
```dockerfile
# Before (15 min — rebuilds deps on every code change):
COPY . .
RUN npm ci

# After (45 sec — deps cached unless package.json changes):
COPY package*.json ./
RUN npm ci
COPY . .
```

**3. Use BuildKit with registry-based caching (for CI):**
CI environments don't have local layer cache between runs. Push cache to the registry:
```bash
docker build \
    --cache-from registry.example.com/myapp:cache \
    --cache-to type=registry,ref=registry.example.com/myapp:cache,mode=max \
    .
```

**4. Use BuildKit cache mounts (for package managers):**
```dockerfile
RUN --mount=type=cache,target=/root/.npm npm ci
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
```
Downloads aren't re-fetched even when the lockfile changes.

**5. Reduce build context:**
A large `.git` directory or `node_modules` in the build context adds minutes of transfer time. Add to `.dockerignore`.

**6. Parallelize with multi-stage:**
BuildKit builds independent stages in parallel. Split tests and build into separate stages.

In my experience, items 2 and 3 together typically reduce a 15-minute build to 1–2 minutes. The most common mistake is COPY . . too early in the Dockerfile."

---

**Q11: Explain how you would containerize a legacy monolithic application that currently runs on a VM.**

**Perfect Answer:**

"Containerizing a legacy monolith is a phased migration — you don't rewrite it, you lift and shift first, then optimize.

**Phase 1: Understand the current state**
- What does the app depend on? (OS, runtime version, libraries, config files, cron jobs, file storage)
- How does it handle state? (local filesystem, session state in memory)
- What ports does it listen on? What services does it talk to?
- How is it started? (systemd, init script, manually)

**Phase 2: Create the initial Dockerfile (lift and shift)**
```dockerfile
FROM ubuntu:22.04   # Match the VM's OS
# Install the same packages as the VM
RUN apt-get update && apt-get install -y java-17-jdk tomcat9 ...
# Copy the application
COPY app.war /opt/tomcat/webapps/
# Copy the config that was in /etc/
COPY config/ /etc/myapp/
EXPOSE 8080
CMD ['catalina.sh', 'run']
```

This won't be elegant or small, but it works. The goal is to prove the app runs in a container.

**Phase 3: Externalize state and configuration**
- Move file storage to object storage (S3) or a volume mount
- Move session state to Redis
- Move config from files to environment variables
- Move cron jobs to a separate container or scheduler

**Phase 4: Optimize**
- Switch to a smaller base image
- Add health checks
- Implement graceful shutdown
- Add proper logging to stdout
- Set resource limits

**Phase 5: Consider decomposition (only if needed)**
- Identify bounded contexts
- Extract the most problematic/frequently-deployed component as a separate service
- Keep the monolith running for everything else

The key insight: you don't need to rewrite the whole thing. A containerized monolith is still better than a VM — it's portable, reproducible, and can run in Kubernetes. Microservices can come later if the team and architecture warrant it."

---

**Q12: FAANG-Style: Design a container platform that serves 10,000 containers across 500 hosts with automated health management.**

**Perfect Answer:**

"At this scale, we need an orchestration platform. I'd design this around Kubernetes, but let me explain the key subsystems:

**1. Scheduling & Placement:**
A central scheduler decides which host runs each container based on:
- Resource availability (CPU, memory, GPU)
- Affinity/anti-affinity rules (spread replicas across failure domains)
- Topology (keep latency-sensitive services in the same zone)
- Node taints/tolerations (GPU nodes only get GPU workloads)

This is what the Kubernetes scheduler does — bin-packing with constraints.

**2. Service Discovery & Load Balancing:**
Containers are ephemeral — IPs change. We need:
- Internal DNS that updates as containers start/stop (CoreDNS in Kubernetes)
- Service abstraction that load-balances across healthy pods
- Ingress layer for external traffic (NGINX Ingress, Envoy)

**3. Health Management:**
- **Liveness probes:** checks if the container is alive. If it fails, container is restarted automatically.
- **Readiness probes:** checks if the container can serve traffic. If it fails, container is removed from load balancer (but not killed).
- **Startup probes:** for slow-starting containers. Prevents premature liveness failures.
- **Self-healing:** If a container dies, the desired-state controller (ReplicaSet) notices and creates a replacement. If a host dies, all its containers are rescheduled to other hosts.

**4. Networking:**
- Pod-to-pod networking (CNI plugin — Calico, Cilium)
- Network policies (microsegmentation — restrict which services can talk)
- Service mesh (Istio/Linkerd) for mTLS, traffic splitting, observability

**5. Observability:**
- Centralized logging (all 10,000 containers to ELK/Loki)
- Metrics (Prometheus scraping each container's /metrics endpoint)
- Distributed tracing (Jaeger/Tempo for request paths across services)
- Alerting at the platform level (node failures, resource exhaustion)

**6. Multi-tenancy & Resource Isolation:**
- Namespaces for team isolation
- ResourceQuotas to prevent a single team from consuming all resources
- LimitRanges to enforce per-container resource bounds
- Network policies for network-level isolation

**7. Deployment Safety:**
- Rolling updates with configurable surge and unavailability
- Canary deployments via service mesh traffic splitting
- Automated rollback on health check failures
- Pod Disruption Budgets to prevent too many pods going down at once

This is essentially Kubernetes + a service mesh + observability stack — the same architecture used by Google (Borg), Netflix, and most companies at this scale."

---

### Production Debugging Questions

---

**Q13: A container works locally but fails in production with 'connection refused' when trying to reach another service. How do you debug?**

**Perfect Answer:**

"'Connection refused' means either the target service isn't running, isn't listening on the expected port, or the network path is blocked. Here's my investigation:

**1. Verify DNS resolution:**
```bash
docker exec myapp nslookup target-service
# If this fails: the containers aren't on the same network,
# or the target container name is wrong
```

**2. Check if the target is actually running:**
```bash
docker ps | grep target-service
# Is it up? Is it healthy?
docker logs target-service  # Any startup errors?
```

**3. Verify the target is listening:**
```bash
docker exec target-service netstat -tlnp
# Is it listening on the expected port?
# Is it listening on 0.0.0.0 (all interfaces) or 127.0.0.1 (localhost only)?
```

**Common root cause:** The target app is binding to `127.0.0.1` instead of `0.0.0.0`. When you bind to localhost inside a container, it's only accessible from within THAT container. In Docker networking, other containers connect via the container's virtual interface — which requires `0.0.0.0`.

**4. Check network connectivity:**
```bash
docker exec myapp ping target-service
docker exec myapp curl -v http://target-service:8080/
```

**5. Check if they're on the same network:**
```bash
docker network inspect mynetwork
# Are both containers listed? Do they have IPs on the same subnet?
```

**6. Check for port mismatches:**
The application config says port 8080, but the container is listening on 3000. Or the Dockerfile says EXPOSE 8080 (metadata only — doesn't actually expose anything).

**Resolution priority:**
1. Target binds to `0.0.0.0`, not `127.0.0.1` (most common)
2. Both containers on the same Docker network
3. Correct port in connection string
4. Target container is healthy and actually started successfully"

---

### Key Interview Terms

| Term | When to Use |
|------|------------|
| **OCI (Open Container Initiative)** | When discussing container standards and portability |
| **Union filesystem / overlay2** | When explaining layers and image efficiency |
| **Namespaces + cgroups** | When explaining how containers work at the kernel level |
| **Build cache invalidation** | When discussing build performance |
| **Distroless** | When discussing minimal/secure images |
| **12-Factor App** | When discussing containerization best practices |
| **Ephemeral** | When discussing stateless containers |
| **Image digest** | When discussing supply chain security |
| **Layer squashing** | When discussing image optimization |
| **Pod (Kubernetes context)** | Show you know containers are just the building block |

---

[⬇️ Download This File](#)

---

*Phase 5 Complete. Ready for Phase 6 — Kubernetes.*
