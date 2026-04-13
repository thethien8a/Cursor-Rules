# Docker Troubleshooting Guide

## Build Issues

### Cache Not Working

**Symptoms**: Build re-runs all steps despite no changes

**Causes & Solutions**:

1. **Layer order wrong**
   ```dockerfile
   # ❌ Source before deps - cache invalidates on any code change
   COPY . .
   RUN npm install
   
   # ✓ Deps before source - cache preserved for deps
   COPY package*.json ./
   RUN npm install
   COPY . .
   ```

2. **Build context too large**
   ```bash
   # Check context size
   docker build --no-cache . 2>&1 | grep "Sending build context"
   
   # Solution: Add to .dockerignore
   node_modules
   .git
   *.log
   ```

3. **Using ADD instead of COPY**
   - ADD always invalidates cache for remote URLs
   - Use COPY for local files

4. **Different build args**
   ```bash
   # Args affect cache - use consistent values
   docker build --build-arg VERSION=1.0 .
   ```

### Build Fails: "No space left on device"

```bash
# Check Docker disk usage
docker system df

# Clean up
docker system prune -a --volumes

# Clean build cache specifically
docker builder prune -a

# Set build cache limit
# In daemon.json:
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB"
    }
  }
}
```

### Build Fails: Permission Denied

```dockerfile
# Problem: COPY fails on mounted volumes
COPY . /app  # Fails if volume has different permissions

# Solutions:
# 1. Use --chown
COPY --chown=1001:1001 . /app

# 2. Fix permissions in RUN
COPY . /app
RUN chown -R 1001:1001 /app
USER 1001
```

### Multi-Stage Build: File Not Found

```dockerfile
# ❌ Wrong: copying from wrong path
FROM node AS build
WORKDIR /app
RUN npm run build  # Creates /app/dist

FROM nginx
COPY --from=build /dist /usr/share/nginx/html  # Wrong path!

# ✓ Correct: use full path
COPY --from=build /app/dist /usr/share/nginx/html
```

---

## Runtime Issues

### Container Exits Immediately

**Debug steps**:

```bash
# Check exit code
docker ps -a --format "{{.Names}} {{.Status}}"

# View logs
docker logs <container>

# Run interactively to debug
docker run -it --entrypoint /bin/sh myimage

# Common causes:
# 1. CMD fails - check command syntax
# 2. Missing dependencies - check error messages
# 3. Permission issues - check USER directive
```

**Exit codes**:
| Code | Meaning |
|------|---------|
| 0 | Success (normal exit) |
| 1 | Application error |
| 126 | Command not executable |
| 127 | Command not found |
| 137 | SIGKILL (OOM or forced stop) |
| 139 | SIGSEGV (segfault) |
| 143 | SIGTERM (graceful stop) |

### Container Ignores SIGTERM (Ctrl+C)

**Cause**: Shell form CMD doesn't receive signals

```dockerfile
# ❌ Shell form - process is child of shell
CMD npm start

# ✓ Exec form - process is PID 1
CMD ["npm", "start"]

# ✓ Or use init process
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["npm", "start"]
```

### OOM Killed (Exit Code 137)

```bash
# Check if OOM killed
docker inspect <container> | grep OOMKilled

# View memory stats
docker stats <container>

# Solutions:
# 1. Increase memory limit
docker run --memory=1g myimage

# 2. Optimize application memory usage

# 3. Add swap (not recommended for production)
docker run --memory=512m --memory-swap=1g myimage
```

### Permission Denied on Mounted Volume

```bash
# Check container user
docker exec <container> id
# uid=1001 gid=1001

# Check volume permissions
ls -la /path/to/volume

# Solutions:
# 1. Match UID/GID
docker run -u $(id -u):$(id -g) -v /path:/data myimage

# 2. Fix permissions on host
sudo chown -R 1001:1001 /path/to/volume

# 3. Use named volume instead of bind mount
docker volume create mydata
docker run -v mydata:/data myimage
```

---

## Networking Issues

### Container Can't Reach Internet

```bash
# Test connectivity
docker run --rm alpine ping -c 3 8.8.8.8

# Check DNS
docker run --rm alpine nslookup google.com

# Solutions:
# 1. Check Docker daemon DNS settings
cat /etc/docker/daemon.json
# Add: {"dns": ["8.8.8.8", "8.8.4.4"]}

# 2. Use host network for testing
docker run --network host myimage

# 3. Check firewall/iptables
sudo iptables -L DOCKER
```

### Containers Can't Communicate

```bash
# Check they're on same network
docker network inspect bridge

# Solutions:
# 1. Create custom network (enables DNS)
docker network create mynet
docker run --network mynet --name app1 image1
docker run --network mynet --name app2 image2
# Now app2 can reach app1 by name

# 2. Use docker-compose (auto-creates network)
```

### Port Already in Use

```bash
# Find what's using the port
sudo lsof -i :3000
# or
sudo netstat -tlnp | grep 3000

# Solutions:
# 1. Stop conflicting process
# 2. Use different port mapping
docker run -p 3001:3000 myimage
# 3. Check for zombie containers
docker ps -a | grep 3000
docker rm -f <container>
```

---

## Performance Issues

### Slow Builds

**Diagnose**:
```bash
# Time each step
DOCKER_BUILDKIT=1 docker build --progress=plain . 2>&1 | tee build.log

# Check context size
du -sh . --exclude=node_modules
```

**Solutions**:

1. **Optimize .dockerignore**
2. **Use BuildKit cache mounts**
   ```dockerfile
   RUN --mount=type=cache,target=/root/.npm npm ci
   ```
3. **Parallelize with multi-stage**
   ```dockerfile
   FROM base AS deps
   RUN npm ci
   
   FROM base AS lint
   RUN npm run lint
   
   FROM deps AS build
   RUN npm run build
   ```

### Slow Container Startup

**Diagnose**:
```bash
# Time startup
time docker run --rm myimage echo "started"
```

**Solutions**:

1. **Pre-compile bytecode**
   ```dockerfile
   # Python
   ENV UV_COMPILE_BYTECODE=1
   
   # Java - use CDS
   RUN java -Xshare:dump
   ```

2. **Reduce image layers**
3. **Use smaller base image**
4. **Lazy load heavy dependencies**

### High Memory Usage

```bash
# Monitor in real-time
docker stats

# Detailed memory breakdown
docker exec <container> cat /proc/meminfo

# Solutions:
# 1. Set memory limits
docker run --memory=512m myimage

# 2. Optimize application
# 3. Use multi-process manager (gunicorn workers)
```

---

## Image Issues

### Image Too Large

**Analyze**:
```bash
# Check layer sizes
docker history myimage

# Detailed analysis with dive
dive myimage
```

**Solutions**:

1. **Multi-stage builds** - Only copy artifacts
2. **Smaller base image** - Alpine, distroless, scratch
3. **Clean in same layer**
   ```dockerfile
   RUN apt-get update && \
       apt-get install -y pkg && \
       rm -rf /var/lib/apt/lists/*
   ```
4. **Remove unnecessary files**
   ```dockerfile
   RUN npm ci --only=production && npm cache clean --force
   ```

### Can't Pull Image

```bash
# Authentication issue
docker login registry.example.com

# Check image exists
docker manifest inspect registry/image:tag

# Rate limit (Docker Hub)
# Solution: Authenticate or use mirror
docker login
```

---

## Compose Issues

### Service Won't Start: "depends_on"

**Problem**: Service starts but dependency not ready

```yaml
# ❌ Only waits for container to start
depends_on:
  - db

# ✓ Wait for health check
depends_on:
  db:
    condition: service_healthy
```

### Volume Data Not Persisting

```bash
# Check volume exists
docker volume ls

# Inspect volume
docker volume inspect myvolume

# Common issues:
# 1. Anonymous volume recreated on restart
#    Solution: Use named volumes
# 2. Wrong path in container
#    Solution: Check WORKDIR and paths
```

### Environment Variables Not Set

```yaml
# Check precedence (highest to lowest):
# 1. docker-compose run -e VAR=value
# 2. environment: in compose file
# 3. env_file: in compose file
# 4. .env file in project root

# Debug
docker compose config  # Shows resolved config
docker compose exec app env  # Shows runtime env
```

---

## Debug Commands Reference

```bash
# Container debugging
docker logs -f --tail 100 <container>     # Stream logs
docker exec -it <container> /bin/sh       # Interactive shell
docker inspect <container>                 # Full config
docker top <container>                     # Running processes
docker stats <container>                   # Resource usage

# Image debugging
docker history <image>                     # Layer history
docker inspect <image>                     # Image metadata
dive <image>                              # Interactive layer analysis

# Network debugging
docker network ls                          # List networks
docker network inspect <network>           # Network details
docker exec <container> netstat -tlnp      # Listening ports

# System debugging
docker system df                           # Disk usage
docker system events                       # Real-time events
docker info                               # System-wide info

# Clean up
docker system prune -a --volumes          # Remove everything unused
docker builder prune -a                   # Clear build cache
```

---

## Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| `COPY failed: file not found` | File not in build context | Check .dockerignore, path |
| `Cannot connect to Docker daemon` | Docker not running | `sudo systemctl start docker` |
| `unauthorized: authentication required` | Not logged in | `docker login` |
| `manifest unknown` | Image/tag doesn't exist | Check spelling, tag |
| `network not found` | Network removed/not created | `docker network create` |
| `port is already allocated` | Port conflict | Change port or stop conflict |
| `no space left on device` | Disk full | `docker system prune -a` |
| `OCI runtime error` | Container config issue | Check capabilities, seccomp |
