# Docker Security Deep Dive

## Security Hierarchy

From least to most secure:

| Level | Image Type | CVEs | Use Case |
|-------|-----------|------|----------|
| 1 | Full OS (Ubuntu/Debian) | 100+ | Development only |
| 2 | Slim variants | 50-100 | When need apt |
| 3 | Alpine | 10-50 | General production |
| 4 | Distroless | 1-10 | Runtime-only apps |
| 5 | Scratch | 0 | Static binaries |

---

## Non-Root User Configuration

### Standard Pattern

```dockerfile
# Create user with explicit UID/GID
RUN groupadd -g 1001 -r appgroup && \
    useradd -r -u 1001 -g appgroup -s /sbin/nologin appuser

# Change ownership of app directory
WORKDIR /app
COPY --chown=appuser:appgroup . .

# Switch to non-root
USER 1001
```

### Alpine-Specific

```dockerfile
RUN addgroup -g 1001 -S app && \
    adduser -S -u 1001 -G app -s /sbin/nologin app

USER 1001
```

### Why Explicit UID/GID?

- Reproducible across builds
- Consistent permissions with mounted volumes
- Required for some orchestrators (OpenShift)
- Use `1001` as convention (above system range)

---

## Rootless Docker

### Enable Rootless Mode

```bash
# Install rootless prerequisites
sudo apt-get install -y uidmap dbus-user-session

# Install rootless Docker
dockerd-rootless-setuptool.sh install

# Configure environment
export PATH=/usr/bin:$PATH
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```

### Requirements

- `newuidmap` and `newgidmap` binaries
- At least 65,536 subordinate UIDs/GIDs per user
- `/etc/subuid` and `/etc/subgid` configured

### Limitations

- No `--privileged` containers
- No AppArmor support
- Some volume mount restrictions

---

## Secrets Management

### Build-Time Secrets (BuildKit)

```dockerfile
# syntax=docker/dockerfile:1

# Mount secret during build (NOT persisted in image)
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) \
    npm install

# Build command:
# docker build --secret id=npm_token,src=.npmrc .
```

### Runtime Secrets (Docker Compose)

```yaml
services:
  app:
    secrets:
      - db_password
      - api_key
    environment:
      # Reference file, not value
      DB_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true  # Pre-created secret
```

### Reading Secrets in Application

```python
# Python example
import os

def get_secret(name):
    """Read secret from file or environment."""
    # Try file first (Docker secrets)
    secret_file = os.environ.get(f'{name}_FILE')
    if secret_file and os.path.exists(secret_file):
        with open(secret_file) as f:
            return f.read().strip()
    # Fall back to environment variable
    return os.environ.get(name)

db_password = get_secret('DB_PASSWORD')
```

### What NOT to Do

```dockerfile
# ❌ WRONG - Secret visible in image history
ENV API_KEY=secret123

# ❌ WRONG - Secret persists in layer
ARG API_KEY
RUN echo $API_KEY > /app/config

# ❌ WRONG - Secret in build context
COPY .env /app/.env
```

---

## Runtime Security Options

### Docker Run Security Flags

```bash
docker run \
  # Non-root user
  --user 1001:1001 \
  
  # Read-only filesystem
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid \
  
  # Drop all capabilities, add only needed
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  
  # Prevent privilege escalation
  --security-opt=no-new-privileges:true \
  
  # Limit resources
  --memory=512m \
  --cpus=1 \
  
  # Disable inter-container communication
  --icc=false \
  
  myimage
```

### Compose Security Configuration

```yaml
services:
  app:
    user: "1001:1001"
    read_only: true
    tmpfs:
      - /tmp:size=100M,mode=1777
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
```

### Linux Capabilities Reference

| Capability | Purpose | Risk Level |
|------------|---------|------------|
| `NET_BIND_SERVICE` | Bind ports < 1024 | Low |
| `CHOWN` | Change file ownership | Medium |
| `SETUID`/`SETGID` | Change user/group | High |
| `SYS_ADMIN` | System administration | Critical |
| `NET_ADMIN` | Network administration | High |

---

## Image Scanning

### Docker Scout

```bash
# Quick overview
docker scout quickview myimage

# Detailed CVE report
docker scout cves myimage

# Recommendations
docker scout recommendations myimage

# Compare images
docker scout compare myimage:v1 myimage:v2
```

### Trivy (Recommended)

```bash
# Install
brew install trivy  # macOS
sudo apt-get install trivy  # Debian/Ubuntu

# Scan image
trivy image myimage

# Scan with severity filter
trivy image --severity HIGH,CRITICAL myimage

# Scan Dockerfile
trivy config Dockerfile

# JSON output for CI
trivy image --format json --output results.json myimage

# Fail CI on critical
trivy image --exit-code 1 --severity CRITICAL myimage
```

### Snyk

```bash
# Authenticate
snyk auth

# Scan image
snyk container test myimage

# Monitor for new vulnerabilities
snyk container monitor myimage
```

### CI Integration Example

```yaml
# GitHub Actions
- name: Scan for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myimage:${{ github.sha }}'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'

- name: Upload Trivy results
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: 'trivy-results.sarif'
```

---

## Supply Chain Security

### Pin Base Images by Digest

```dockerfile
# Instead of mutable tag
FROM python:3.13-slim

# Use immutable digest
FROM python:3.13-slim@sha256:abc123...
```

### Verify Image Signatures (Cosign)

```bash
# Sign image
cosign sign myregistry/myimage:v1

# Verify signature
cosign verify myregistry/myimage:v1

# In Kubernetes, use policy enforcement
```

### SBOM Generation

```bash
# Generate SBOM with Trivy
trivy image --format spdx-json myimage > sbom.json

# Generate with Docker Scout
docker scout sbom myimage

# Syft
syft myimage -o spdx-json > sbom.json
```

---

## Network Security

### Isolate Backend Services

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No internet access

services:
  nginx:
    networks: [frontend, backend]
  
  app:
    networks: [backend]
  
  db:
    networks: [backend]
```

### Limit Container Communication

```bash
# Disable inter-container communication globally
dockerd --icc=false

# Per-network
docker network create --driver bridge -o com.docker.network.bridge.enable_icc=false isolated
```

---

## Hardened Base Images

### Google Distroless

```dockerfile
# Node.js
FROM gcr.io/distroless/nodejs22-debian12

# Python
FROM gcr.io/distroless/python3-debian12

# Java
FROM gcr.io/distroless/java21-debian12

# Static (Go/Rust)
FROM gcr.io/distroless/static-debian12
```

### Chainguard Images

```dockerfile
# More minimal than distroless
FROM cgr.dev/chainguard/python:latest
FROM cgr.dev/chainguard/node:latest
FROM cgr.dev/chainguard/go:latest
```

### Docker Hardened Images (DHI)

```dockerfile
# Enterprise hardened images
FROM docker.io/library/python:3.13-slim-bookworm-hardened
```

---

## Security Checklist

### Dockerfile
- [ ] Non-root USER with explicit UID/GID
- [ ] Base image pinned (tag + digest)
- [ ] No secrets in ENV or ARG
- [ ] COPY preferred over ADD
- [ ] Minimal packages installed
- [ ] Package cache cleaned

### Runtime
- [ ] `--read-only` filesystem where possible
- [ ] Capabilities dropped (`--cap-drop=ALL`)
- [ ] `--security-opt=no-new-privileges`
- [ ] Resource limits set
- [ ] Health checks implemented

### Network
- [ ] Internal networks for backend
- [ ] Minimal port exposure
- [ ] TLS for external communication

### Monitoring
- [ ] Regular vulnerability scanning
- [ ] Base image update automation
- [ ] Runtime anomaly detection
- [ ] Audit logging enabled

---

## Security Scanning in Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.13-slim AS base
# ... build steps ...

FROM base AS security-scan
# Copy Trivy for in-build scanning
COPY --from=aquasec/trivy:latest /usr/local/bin/trivy /usr/local/bin/trivy
RUN trivy filesystem --exit-code 1 --severity HIGH,CRITICAL /app

FROM base AS production
# ... final stage ...
```
