# Docker - Interview Questions & Answers
> Senior DevOps/SRE Level | 6+ Years Experience

---

## Q1: What is the difference between a Docker image and a container?

**Answer:**
- **Image**: Read-only template with layers. Immutable. Stored in registry. (Like a class in OOP)
- **Container**: A running instance of an image. Has a writable layer on top of image layers. Ephemeral. (Like an object)

A single image can spawn many containers. The writable layer is lost when container is deleted (use volumes for persistence).

---

## Q2: Explain Docker layered filesystem and why it matters

Docker uses OverlayFS (overlay2 driver). Each Dockerfile instruction creates a layer.

**Why it matters**:
1. **Caching**: Unchanged layers are cached. Rebuild only from changed instruction onward.
2. **Sharing**: Multiple containers share same image layers — saves disk space.
3. **Speed**: Only pull/push changed layers when updating images.

**Implication**: Order instructions from least-changing (base image, dependencies) to most-changing (app code).

---

## Q3: What is a multi-stage build? Why use it?

Multi-stage builds use multiple FROM statements. Earlier stages build; final stage creates minimal production image.

```dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production (tiny image)
FROM nginx:1.25-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Benefits**: Node.js app from 1.2GB to 23MB. No build tools/source code in production image. Reduced attack surface.

---

## Q4: How do you handle secrets in Docker?

**WRONG** - secrets baked into image layers (visible in docker history forever):
```dockerfile
ENV AWS_SECRET_KEY=mysecretkey123  # NEVER DO THIS
```

**RIGHT approaches**:

1. **Runtime environment variables**:
```bash
docker run -e DB_PASSWORD=$(vault kv get -field=password secret/db) myapp
```

2. **Docker BuildKit secrets** (not persisted in layers):
```dockerfile
RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret
```
```bash
DOCKER_BUILDKIT=1 docker build --secret id=mysecret,src=./secret.txt .
```

3. **Kubernetes Secrets + External Secrets Operator** (preferred in production)

---

## Q5: COPY vs ADD in Dockerfile?

| Aspect | COPY | ADD |
|---|---|---|
| Basic file copy | Yes | Yes |
| URL download | No | Yes |
| Auto-extract tar.gz | No | Yes |
| Recommendation | **Always prefer** | Only when needed |

Use COPY by default. It's explicit and predictable.

---

## Q6: CMD vs ENTRYPOINT?

```dockerfile
ENTRYPOINT ["nginx"]        # Fixed executable (hard to override)
CMD ["-g", "daemon off;"]   # Default args (easily overridden)

# docker run myimage → nginx -g 'daemon off;'
# docker run myimage -v → nginx -v
```

**Best practice**: ENTRYPOINT for main executable, CMD for defaults. Use exec form (JSON array) for proper signal handling.

---

## Q7: How do you optimize Docker image size?

1. **Minimal base**: alpine (5MB) vs ubuntu (77MB), distroless, scratch
2. **Multi-stage builds**: Only copy artifacts to final image
3. **Combine RUN commands** (reduce layers, clean in same layer):
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```
4. **.dockerignore**: Exclude .git, node_modules, tests, docs
5. **No unnecessary packages**: --no-install-recommends

---

## Q8: Docker networking modes

| Mode | Description | Use Case |
|---|---|---|
| bridge (default) | Private network, port mapping needed | Typical container apps |
| host | Shares host network | High-performance, no isolation |
| none | No networking | Complete isolation |
| Custom bridge | Built-in DNS by container name | Multi-container apps |

**Always use custom bridge networks** over default bridge — provides DNS resolution by container name:
```bash
docker network create myapp-net
docker run --network myapp-net --name db postgres
docker run --network myapp-net --name app myapp  # app can connect to 'db' by name
```

---

## Q9: How do you debug a running container?

```bash
# Logs
docker logs mycontainer --tail=100 -f

# Execute into container
docker exec -it mycontainer /bin/sh

# Resource usage
docker stats mycontainer
docker top mycontainer

# Inspect metadata
docker inspect mycontainer
docker inspect --format='{{.State.Status}}' mycontainer

# If container crashes immediately - override entrypoint
docker run --entrypoint /bin/sh mycontainer:broken

# Copy files out
docker cp mycontainer:/app/logs/error.log ./

# View filesystem changes
docker diff mycontainer
```

---

## Q10: What is Docker BuildKit?

BuildKit is Docker's next-gen build engine (default since Docker 23.0).

**Advantages**:
- Parallel stage execution (faster builds)
- Better cache management
- Build secrets support (--mount=type=secret)
- SSH forwarding for private repos
- Multi-platform builds (linux/amd64, linux/arm64)
- 30-70% faster than legacy builder

```bash
# Multi-platform build
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:v1 .

# Cache from registry (CI/CD optimization)
docker buildx build \
  --cache-from type=registry,ref=myregistry/myapp:cache \
  --cache-to type=registry,ref=myregistry/myapp:cache,mode=max \
  -t myapp:v1 .
```

---

## Q11: Containers vs VMs

| Aspect | Container | VM |
|---|---|---|
| Isolation | Process-level (kernel namespaces) | Full hardware virtualization |
| OS | Shares host kernel | Has own kernel |
| Size | MBs | GBs |
| Startup | Milliseconds | Minutes |
| Performance | Near-zero overhead | 5-20% overhead |
| Security | Weaker (shared kernel) | Stronger |

Containers use Linux **namespaces** (pid, net, mnt, uts, ipc, user) and **cgroups** (CPU/memory limits). No hypervisor.

---

## Q12: Container image security scanning

```bash
# Trivy (most popular)
trivy image myapp:v1
trivy image --severity HIGH,CRITICAL myapp:v1

# Docker Scout (built-in)
docker scout cves myapp:v1
docker scout recommendations myapp:v1

# In CI pipeline (fail build on critical)
trivy image --exit-code 1 --severity CRITICAL myapp:v1
```

**Security hardening**:
- Run as non-root user
- Use read-only filesystem where possible
- Drop all capabilities, add only needed
- Use distroless images (no shell = smaller attack surface)
- Scan images in CI pipeline and registry
- Sign images with Cosign/Notary
