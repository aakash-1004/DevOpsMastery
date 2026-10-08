---
title: Revision — Docker
type: cheatsheet
module: revision
status: ✅ Active Revision
interview-relevance: 🔴 Maximum — Full Stack DevOps Interview Prep
tags:
  - devops
  - revision
  - interview
  - masterreview
  - cheatsheet
order: 4
up: "[[Master-Revision-MOC]]"
prev: "[[Revision-03-Git-and-Version-Control]]"
next: "[[Revision-05-Kubernetes]]"
related:
  - "[[Docker-MOC]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — Docker

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · ← [[Revision-03-Git-and-Version-Control|Revision — Git & Version Control]] · → [[Revision-05-Kubernetes|Revision — Kubernetes]]
<!-- /nav -->

## Concept

Docker packages your application and all its dependencies into a portable unit called a container. A container is an isolated process running on the host kernel — it's NOT a VM. Containers share the host kernel but have isolated: filesystem (via Union FS layers), network, process space, and resource limits (via cgroups).

**The key insight:** "It works on my machine" dies because containers make your environment reproducible. The container IS the environment.

**Docker Architecture:**

```
Docker CLI  →  Docker Daemon (dockerd)  →  containerd  →  runc
     (client)        (API server)              (runtime)    (OCI runtime)
```

**Image Layers (Union FS):**

- Each `RUN`, `COPY`, `ADD` instruction creates a new layer
- Layers are cached — if layer N hasn't changed, Docker reuses it
- Build order matters: put frequently-changing layers (COPY . .) last
- Layers are read-only. A container adds a thin writable layer on top

---

## Core Commands

```bash
# Image management
docker build -t myapp:1.0 .                # Build image from Dockerfile in current dir
docker build -t myapp:1.0 -f Dockerfile.prod .  # Use specific Dockerfile
docker images                              # List local images
docker pull nginx:1.25                     # Pull specific version (always pin versions!)
docker push aakash0908/myapp:1.0          # Push to Docker Hub
docker rmi myapp:1.0                       # Remove image
docker image prune                         # Remove dangling images

# Container lifecycle
docker run -d -p 8080:5000 --name myapp myapp:1.0    # Detached, port mapping, named
docker run -it ubuntu:22.04 /bin/bash               # Interactive terminal
docker run --rm -e DB_URL=mongodb://... myapp:1.0   # Auto-remove, env var
docker ps                                            # Running containers
docker ps -a                                         # All containers including stopped
docker stop myapp && docker rm myapp                 # Stop then remove
docker exec -it myapp /bin/bash                      # Shell into running container
docker logs myapp -f                                 # Follow live logs
docker inspect myapp                                 # Full container metadata as JSON

# Networking
docker network ls                          # List networks
docker network create mynet               # Create custom bridge network
docker network inspect mynet             # Show connected containers and IPs
docker run --network mynet myapp:1.0     # Connect container to network

# Volumes
docker volume create mydata              # Create named volume
docker run -v mydata:/app/data myapp     # Mount named volume
docker run -v $(pwd):/app myapp          # Bind mount (dev use — live code reload)
docker volume ls                         # List volumes

# Cleanup
docker system prune -af                  # Remove ALL unused: images, containers, networks
docker volume prune                      # Remove unused volumes
```

---

## Dockerfile — Production Pattern

```dockerfile
# Stage 1: Build
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Runtime (smaller final image)
FROM python:3.11-slim
WORKDIR /app

# Create non-root user FIRST
RUN useradd -m -u 1001 appuser

# Copy only installed packages from builder
COPY --from=builder /root/.local /root/.local
COPY . .

# Set PATH, switch to non-root user
ENV PATH=/root/.local/bin:$PATH
USER appuser

EXPOSE 5000
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

**Why multi-stage?** Your final image doesn't contain build tools (gcc, pip cache, etc.). A Python app that's 800MB with build tools becomes 150MB runtime-only.

---

## Docker Compose

```yaml
version: "3.9"
services:
  web:
    build: .
    ports:
      - "8080:5000"
    environment:
      - MONGO_URL=mongodb://mongo:27017/taskdb
    depends_on:
      - mongo
    networks:
      - appnet
    restart: unless-stopped

  mongo:
    image: mongo:6
    volumes:
      - mongodata:/data/db
    networks:
      - appnet
    restart: unless-stopped

volumes:
  mongodata:

networks:
  appnet:
    driver: bridge
```

```bash
docker compose up -d         # Start all services detached
docker compose down          # Stop and remove containers
docker compose down -v       # Also remove volumes (wipes data)
docker compose logs -f web   # Follow logs for specific service
docker compose exec web sh   # Shell into running service
docker compose ps            # Status of all services
```

---

## Senior Interview Q&A

**Q: How do Docker containers achieve isolation?**

> "Containers use two kernel features: namespaces and cgroups.
>
> Namespaces provide isolation of system resources. There are six: PID namespace (container processes have their own PID tree, PID 1 inside is not PID 1 on host), network namespace (own network interfaces, routing table, iptables), mount namespace (own filesystem view), UTS namespace (own hostname), IPC namespace (own inter-process communication), user namespace (map container root to unprivileged host user).
>
> cgroups (control groups) provide resource limits — you can cap a container at 512MB RAM and 0.5 CPU. Without cgroups, a runaway container could OOM the host.
>
> Together: namespaces make each container think it's alone on the system, cgroups ensure it can't consume more than its share.
>
> This is fundamentally different from VMs: VMs virtualize hardware, containers virtualize the OS. That's why containers start in milliseconds (no OS boot) but VMs take minutes."

---

**Q: Container keeps restarting. How do you debug it?**

> "Classic scenario. Four things to check in order:
>
> `docker ps` — shows restart count and status. If status is 'Restarting', it's crashing and Docker is restarting it per the restart policy.
>
> `docker logs <container>` — look at the last few lines before the crash. Application errors, missing config, permission denied, port already in use — 90% of crash reasons show up here.
>
> `docker inspect <container>` — look at the ExitCode in State. Code 1 is generic error, code 137 is OOM kill (cgroup killed it for exceeding memory limit), code 139 is segfault, code 127 is command not found.
>
> If the container crashes before you can exec into it: override the entrypoint to get a shell: `docker run -it --entrypoint /bin/sh myapp:1.0`. This lets you inspect the filesystem, check if env vars are set, check if config files exist.
>
> In production on Kubernetes: `kubectl describe pod` shows Events (OOMKilled, CrashLoopBackOff reasons), `kubectl logs <pod> --previous` shows logs from the crashed instance."

---

**Q: What's the difference between CMD and ENTRYPOINT?**

> "Both define what runs when a container starts, but they interact differently with arguments.
>
> ENTRYPOINT sets the executable — it's the main process, and it can't be easily overridden at runtime (you need `--entrypoint` flag). CMD provides default arguments to ENTRYPOINT, or if ENTRYPOINT isn't set, CMD is the command to run.
>
> The common pattern I use: ENTRYPOINT is the executable, CMD is the default arguments. This means someone can do `docker run myapp --debug` and it appends `--debug` to the entrypoint command without replacing it.
>
> If you only use CMD: `docker run myapp /bin/sh` replaces CMD entirely, which is sometimes what you want for debugging.
>
> In production: always use the exec form (`["executable", "arg"]`) not shell form (`executable arg`). Shell form wraps the command in `/bin/sh -c`, which means your process is not PID 1 — it's the shell. That means Docker stop signals don't reach your app, and you get ugly shutdown behavior."

---

**Q: How do you reduce Docker image size?**

> "Six strategies I apply to every production image:
>
> 1. Multi-stage builds — build in one stage, copy only artifacts to runtime stage. No build tools in final image.
> 2. Use slim or alpine base images — `python:3.11-slim` vs `python:3.11`. 150MB vs 900MB.
> 3. Clean up in the same RUN layer — `apt-get install -y curl && apt-get clean && rm -rf /var/lib/apt/lists/*`. If you clean in a separate RUN, the install layer is still in the image.
> 4. Use `.dockerignore` — exclude `.git`, `node_modules`, `__pycache__`, test files.
> 5. Combine RUN commands — every RUN creates a layer. Chain with && to keep it one layer.
> 6. Don't copy unnecessary files — only COPY what the runtime needs."

---

**Q: Docker networking — how does a container reach another container?**

> "Docker has four network modes:
>
> **bridge** (default): Docker creates a virtual bridge (`docker0`). Each container gets a virtual ethernet on that bridge. Containers on the same bridge can reach each other by IP. For DNS resolution by name, you need a user-defined bridge network — not the default bridge. That's why Compose creates its own network: so services can reach each other by service name.
>
> **host**: Container shares the host's network namespace. No port mapping needed — a container listening on port 5000 IS listening on host port 5000. Best performance, worst isolation. Use for high-throughput networking where NAT overhead matters.
>
> **none**: No network. Complete isolation. Used for batch jobs or security-sensitive workloads.
>
> **overlay**: For Docker Swarm — spans multiple hosts. Built on VXLAN. In Kubernetes, this is replaced by CNI plugins (Calico, Flannel, Cilium).
>
> User-defined bridge is what I use in Compose: containers get DNS resolution by service name, isolated network per app, no cross-contamination."

<!-- related -->
---
**Related:** [[Docker-MOC|Docker]]
<!-- /related -->
