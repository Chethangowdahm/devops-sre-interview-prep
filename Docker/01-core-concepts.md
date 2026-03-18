# Docker - Core Concepts
> Senior DevOps/SRE Level | 6+ Years Experience

---

## 1. What is Docker?
Docker is a platform for developing, shipping, and running applications in **containers**. Containers are lightweight, portable, isolated environments that package an application with all its dependencies.

**Key benefit**: "Works on my machine" problem solved — same container runs identically everywhere.

---

## 2. Docker Architecture

```
Docker Client (CLI)
    ↓  REST API
Docker Daemon (dockerd)
    ↓
Containerd (container runtime)
    ↓
runc (OCI-compliant low-level runtime)
    ↓
Linux Kernel (namespaces + cgroups)
```

### Components
- **Docker Daemon (dockerd)**: Background service managing containers, images, volumes, networks
- **Docker Client**: CLI tool that talks to daemon via REST API
- **Docker Registry**: Storage for images (Docker Hub, ECR, GCR, ACR)
- **containerd**: Industry-standard container runtime (also used by Kubernetes)

---

## 3. Core Concepts

### Images vs Containers
| Aspect | Image | Container |
|---|---|---|
| What | Read-only template | Running instance of an image |
| State | Static (layers) | Dynamic (ephemeral writable layer) |
| Storage | Registry | Host filesystem |
| Lifecycle | Build once, use many times | Start, run, stop, delete |

### Layers
Docker images are built in layers. Each Dockerfile instruction creates a layer. Layers are **cached** and **shared** between images.

```
FROM ubuntu:20.04      → Layer 1: Base OS
RUN apt-get update    → Layer 2: Package updates
RUN apt-get install   → Layer 3: Installed packages
COPY app/ /app        → Layer 4: Application code
```

### Union File System (OverlayFS)
- **Lower layers**: Read-only image layers
- **Upper layer**: Writable container layer (changes deleted when container stops unless volume used)
- Files appear as single unified filesystem

---

## 4. Dockerfile Best Practices

```dockerfile
# Production-ready multi-stage Dockerfile
# Stage 1: Build
FROM golang:1.21-alpine AS builder
WORKDIR /app

# Copy dependencies first (cache optimization)
COPY go.mod go.sum ./
RUN go mod download

# Copy source and build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

# Stage 2: Final image (minimal)
FROM alpine:3.18
RUN apk --no-cache add ca-certificates tzdata

# Run as non-root user (security)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

WORKDIR /app
COPY --from=builder /app/server .

# Document exposed port
EXPOSE 8080

# Use exec form for signals
CMD ["/app/server"]
```

### Best Practices Summary
1. **Multi-stage builds**: Separate build and runtime images. Final image is minimal.
2. **Non-root user**: Security — never run as root in production
3. **Minimize layers**: Combine RUN commands with &&
4. **.dockerignore**: Exclude build artifacts, git history, node_modules
5. **Specific tags**: Never use `latest` in production (`nginx:1.25.3` not `nginx:latest`)
6. **Cache ordering**: Put rarely-changing instructions first (FROM, RUN apt-get) and frequently-changing last (COPY source)
7. **No secrets in Dockerfile**: Use BuildKit secrets or runtime env vars

---

## 5. Docker Networking

### Network Types
| Type | Description | Use Case |
|---|---|---|
| **bridge** (default) | Private internal network on host | Container-to-container on same host |
| **host** | Shares host network namespace | High performance, no isolation |
| **none** | No networking | Completely isolated containers |
| **overlay** | Multi-host networking (Docker Swarm) | Distributed apps |
| **macvlan** | Container gets its own MAC/IP | Network appliances |

```bash
# Create custom bridge network (better than default)
docker network create myapp-network

# Containers on same network can communicate by container name (DNS)
docker run --name db --network myapp-network postgres:13
docker run --name app --network myapp-network -e DB_HOST=db myapp:v1
```

---

## 6. Docker Storage/Volumes

### Types
- **Volume**: Managed by Docker, stored in /var/lib/docker/volumes. Best for persistent data.
- **Bind mount**: Maps host directory into container. Good for development.
- **tmpfs mount**: In-memory only. Good for sensitive data.

```bash
# Named volume (preferred)
docker run -v mydata:/var/lib/mysql mysql:8

# Bind mount (dev)
docker run -v $(pwd)/src:/app/src myapp:dev

# Volume in Compose
volumes:
  mydata:
    driver: local
```

---

## 7. Docker Compose

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
    - "8080:8080"
    environment:
    - DB_HOST=db
    - DB_PORT=5432
    depends_on:
      db:
        condition: service_healthy
    volumes:
    - ./logs:/app/logs
    networks:
    - app-network
    restart: unless-stopped

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
    - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
    - app-network

  nginx:
    image: nginx:1.25
    ports:
    - "80:80"
    - "443:443"
    volumes:
    - ./nginx.conf:/etc/nginx/nginx.conf:ro
    - ./certs:/etc/nginx/certs:ro
    depends_on:
    - app
    networks:
    - app-network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

---

## 8. Container Security

### Key Security Practices
```dockerfile
# 1. Non-root user
RUN useradd -m appuser
USER appuser

# 2. Read-only filesystem
docker run --read-only myapp

# 3. Drop capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp

# 4. No privileged containers
# NEVER: docker run --privileged myapp (gives root access to host)

# 5. Resource limits
docker run --memory="512m" --cpus="1.0" myapp

# 6. Scan images for vulnerabilities
docker scout cves myimage:tag
trivy image myimage:tag
```

### Secrets Management
```bash
# BAD: Never put secrets in environment variables in Dockerfile
# BAD: Never bake secrets into image layers

# GOOD: Runtime injection
docker run -e DB_PASSWORD=$(vault read secret/db-password) myapp

# GOOD: Docker secrets (Swarm)
echo "mysecret" | docker secret create db_password -
docker service create --secret db_password myapp
```

---

## 9. Container Resource Management

```bash
# CPU and memory limits
docker run \
  --memory="512m" \          # Memory limit (OOM killed if exceeded)
  --memory-reservation="256m" \  # Soft limit
  --cpus="1.5" \             # Equivalent to 1.5 CPU cores
  --cpu-shares=512 \         # Relative weight (default 1024)
  myapp

# Check resource usage
docker stats
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

---

## 10. Docker Registry

### Common Registries
- **Docker Hub**: Public registry, limited free pulls
- **AWS ECR**: Private registry, integrates with EKS
- **GCP Artifact Registry**: Private registry, integrates with GKE
- **Harbor**: Self-hosted, enterprise features

```bash
# AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Tag and push
docker tag myapp:v1 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1

# GCP Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev
docker tag myapp:v1 us-central1-docker.pkg.dev/myproject/myrepo/myapp:v1
docker push us-central1-docker.pkg.dev/myproject/myrepo/myapp:v1
```

---

## 11. Key Docker Commands

```bash
# Images
docker build -t myapp:v1 .
docker build --no-cache -t myapp:v1 .
docker images
docker rmi myapp:v1
docker pull nginx:1.25

# Containers
docker run -d --name myapp -p 8080:8080 myapp:v1
docker ps
docker ps -a
docker stop myapp
docker rm myapp
docker logs myapp --tail=50 -f
docker exec -it myapp /bin/sh

# Inspect
docker inspect myapp
docker inspect --format='{{.NetworkSettings.IPAddress}}' myapp

# Cleanup
docker system prune -a          # Remove all unused resources
docker image prune -a           # Remove unused images
docker volume prune             # Remove unused volumes

# Stats
docker stats
docker top myapp

# Copy files
docker cp myapp:/app/config.yaml ./config.yaml
docker cp ./newconfig.yaml myapp:/app/config.yaml

# Build with BuildKit
DOCKER_BUILDKIT=1 docker build -t myapp:v1 .
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:v1 .
```
