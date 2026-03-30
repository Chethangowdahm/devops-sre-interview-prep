# Docker — Deep Dive for SRE/DevOps (6 YOE)

---

## What is Docker?

Docker is a **containerization platform** that packages an application and all its dependencies (libraries, runtime, config) into a single portable unit called a **container**.

> "It works on my machine" → Docker eliminates this by making the environment part of the artifact.

---

## Core Concepts

### Image vs Container

```
Dockerfile  ──build──►  Docker Image  ──run──►  Container
(recipe)                (template/snapshot)      (running process)
```

- **Image**: Read-only layered filesystem. Stored in registry (GCR, DockerHub).
- **Container**: Running instance of an image. Ephemeral by default.
- **Registry**: Storage for images (Google Container Registry, Artifact Registry, DockerHub).

### Layered Architecture

```
┌─────────────────────────────────────┐  ← App layer (your code)     WRITABLE
├─────────────────────────────────────┤  ← COPY dependencies          READ-ONLY
├─────────────────────────────────────┤  ← Install packages (RUN)     READ-ONLY
├─────────────────────────────────────┤  ← Base OS (ubuntu:22.04)     READ-ONLY
└─────────────────────────────────────┘
```

Each `RUN`, `COPY`, `ADD` instruction in a Dockerfile creates a new **layer**.
Layers are **cached** — unchanged layers reuse cache → faster builds.

---

## Dockerfile — Best Practices at 6 YOE

```dockerfile
# ── Stage 1: Build ──────────────────────────────────
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Copy dependency files FIRST (cache optimization)
COPY go.mod go.sum ./
RUN go mod download

# Then copy source
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/server

# ── Stage 2: Runtime (minimal image) ─────────────────
FROM gcr.io/distroless/static:nonroot

# Security: run as non-root
USER nonroot:nonroot

COPY --from=builder --chown=nonroot:nonroot /app/server /server

EXPOSE 8080

ENTRYPOINT ["/server"]
```

**Key best practices:**
- **Multi-stage builds**: Separate build env from runtime → smaller final image
- **Distroless/Alpine base**: Reduce attack surface, smaller size
- **Non-root user**: Security hardening
- **Layer ordering**: Copy dependencies before source code (cache benefit)
- **`.dockerignore`**: Exclude `.git`, `node_modules`, test files from build context

---

## Essential Docker Commands

```bash
# Build
docker build -t myapp:v1.0.0 .
docker build --no-cache -t myapp:latest .

# Run
docker run -d -p 8080:8080 --name myapp myapp:v1.0.0
docker run -it ubuntu:22.04 /bin/bash          # interactive
docker run --rm myapp:latest                   # auto-remove on exit

# Inspect
docker ps                                       # running containers
docker ps -a                                    # all containers
docker logs -f myapp                            # follow logs
docker exec -it myapp sh                        # shell into container
docker inspect myapp                            # full metadata (JSON)
docker stats                                    # live resource usage

# Image management
docker images
docker pull gcr.io/project/image:tag
docker push gcr.io/project/image:tag
docker tag myapp:latest gcr.io/my-project/myapp:v1.0.0
docker rmi myapp:old                            # remove image

# Cleanup
docker system prune -a                          # remove all unused resources
docker volume prune
```

---

## Docker Networking

```
┌──────────────────────────────────────────────────┐
│  Docker Host                                      │
│                                                   │
│  ┌─────────────┐    bridge0    ┌─────────────┐   │
│  │  container1 │◄─────────────►│  container2 │   │
│  │  172.17.0.2 │               │  172.17.0.3 │   │
│  └─────────────┘               └─────────────┘   │
│                                                   │
│  Host ports mapped via -p 8080:8080               │
└──────────────────────────────────────────────────┘
```

**Network types:**
- `bridge` (default): Isolated network, containers talk via bridge
- `host`: Container shares host network stack (no isolation)
- `none`: No networking
- `overlay`: Multi-host networking (Docker Swarm / not used with K8s)

> In Kubernetes, Docker networking is replaced by CNI plugins (Calico, Flannel, Cilium).

---

## Docker Volumes

```bash
# Named volume (persists after container removal)
docker run -v mydata:/var/lib/mysql mysql:8

# Bind mount (host dir → container)
docker run -v /host/path:/container/path myapp

# Read-only mount
docker run -v /config:/etc/app:ro myapp
```

---

## Docker Compose (local dev / testing)

```yaml
# docker-compose.yml
version: '3.9'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
      - DB_PORT=5432
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

```bash
docker-compose up -d          # start in background
docker-compose logs -f app    # follow app logs
docker-compose down -v        # stop and remove volumes
```

---

## Container Security (SRE critical)

| Risk | Mitigation |
|------|-----------|
| Root inside container | `USER nonroot` in Dockerfile |
| Vulnerable base image | Scan with `trivy`, `grype`, or `Snyk` |
| Secrets in image | Use K8s Secrets / GCP Secret Manager at runtime |
| Privileged containers | Avoid `--privileged`; use securityContext in K8s |
| Image from unknown source | Use trusted registries, pin digests `image@sha256:abc` |

```bash
# Scan image for CVEs
trivy image myapp:v1.0.0
```

---

## Docker in the CI/CD Pipeline

```
GitHub PR merged
       │
       ▼
Jenkins / GitHub Actions
       │
       ├─ docker build -t gcr.io/proj/app:${GIT_SHA} .
       ├─ trivy image gcr.io/proj/app:${GIT_SHA}      ← security scan
       ├─ docker push gcr.io/proj/app:${GIT_SHA}
       └─ kubectl set image deployment/app app=gcr.io/proj/app:${GIT_SHA}
```

---

## Interview Questions — Docker (6 YOE Level)

**Q: What's the difference between CMD and ENTRYPOINT?**
> `ENTRYPOINT` defines the main executable (not easily overridden). `CMD` provides default arguments to `ENTRYPOINT` or a default command. Best practice: use `ENTRYPOINT ["binary"]` + `CMD ["--default-flag"]`.

**Q: How do you reduce Docker image size?**
> Multi-stage builds, distroless/alpine base images, combine RUN commands with `&&` to reduce layers, use `.dockerignore`, clean up package manager caches in same RUN layer (`apt-get clean && rm -rf /var/lib/apt/lists/*`).

**Q: How do containers differ from VMs?**
> VMs virtualize hardware (each has its own OS kernel). Containers share the host OS kernel and are isolated via Linux namespaces (pid, net, mnt, ipc) and cgroups (resource limits). Containers start in milliseconds, VMs in minutes.

**Q: How do you handle secrets in Docker?**
> Never bake secrets into images. Use runtime injection: K8s Secrets (mounted as env or file), GCP Secret Manager, or Docker secrets (Swarm). Pass at runtime via environment variables injected by orchestrator.

**Q: What are cgroups and namespaces?**
> **Namespaces**: Isolate what a process can see (pid, network, filesystem, users). **cgroups**: Limit what a process can use (CPU, memory, I/O). Together they form container isolation.

**Q: How do you debug a crashing container?**
```bash
docker logs <container>             # check logs even after crash
docker inspect <container>          # check exit code, OOMKilled
docker run --entrypoint sh myapp    # override entrypoint to debug
docker events                       # real-time Docker events
```
