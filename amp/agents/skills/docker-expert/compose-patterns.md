# Docker Compose Advanced Patterns

## Complete Production Stack

```yaml
# docker-compose.yml
name: myapp

services:
  # ======================
  # Application Services
  # ======================
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
      args:
        - BUILDKIT_INLINE_CACHE=1
    image: myapp:${TAG:-latest}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://postgres:${DB_PASSWORD}@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    ports:
      - "3000:3000"
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      start_period: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  # ======================
  # Database Services
  # ======================
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: myapp
    secrets:
      - db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 1G

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ======================
  # Utility Services (profiles)
  # ======================
  adminer:
    profiles: [dev, debug]
    image: adminer:latest
    ports:
      - "8080:8080"
    networks:
      - backend
    depends_on:
      - db

  redis-commander:
    profiles: [dev, debug]
    image: rediscommander/redis-commander:latest
    environment:
      - REDIS_HOSTS=local:redis:6379
    ports:
      - "8081:8081"
    networks:
      - backend
    depends_on:
      - redis

# ======================
# Networks
# ======================
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access

# ======================
# Volumes
# ======================
volumes:
  postgres_data:
  redis_data:

# ======================
# Secrets
# ======================
secrets:
  db_password:
    file: ./secrets/db_password.txt
```

---

## Development Override

```yaml
# docker-compose.override.yml (auto-loaded)
services:
  app:
    build:
      target: development
    volumes:
      - .:/app
      - /app/node_modules  # Preserve container's node_modules
    environment:
      - NODE_ENV=development
      - DEBUG=app:*
    ports:
      - "3000:3000"
      - "9229:9229"  # Debug port
    command: npm run dev

  db:
    ports:
      - "5432:5432"  # Expose for local tools

  redis:
    ports:
      - "6379:6379"
```

---

## Profile-Based Configuration

### Usage Patterns

```bash
# Default services only
docker compose up

# With development tools
docker compose --profile dev up

# With monitoring stack
docker compose --profile monitoring up

# Multiple profiles
docker compose --profile dev --profile monitoring up

# Using environment variable
COMPOSE_PROFILES=dev,monitoring docker compose up
```

### Profile Definition Example

```yaml
services:
  # Always runs (no profile)
  app:
    build: .

  # Development only
  app-debug:
    profiles: [dev]
    build:
      target: debug

  # Monitoring stack
  prometheus:
    profiles: [monitoring]
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"

  grafana:
    profiles: [monitoring]
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    depends_on:
      - prometheus

  # Testing
  test-runner:
    profiles: [test]
    build:
      target: test
    command: npm test
```

---

## Multi-Environment Configuration

### Directory Structure

```
project/
├── docker-compose.yml          # Base configuration
├── docker-compose.override.yml # Local dev (auto-loaded)
├── docker-compose.prod.yml     # Production overrides
├── docker-compose.test.yml     # Test environment
└── .env                        # Environment variables
```

### Production Compose

```yaml
# docker-compose.prod.yml
services:
  app:
    image: registry.example.com/myapp:${TAG}
    build: !reset null  # Don't build in prod
    volumes: !reset []  # No volume mounts
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      rollback_config:
        parallelism: 1
        delay: 10s

  db:
    volumes:
      - /mnt/data/postgres:/var/lib/postgresql/data
```

### Commands

```bash
# Development (uses override)
docker compose up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Testing
docker compose -f docker-compose.yml -f docker-compose.test.yml run --rm test-runner
```

---

## Health Check Patterns

### HTTP Service

```yaml
healthcheck:
  test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
  interval: 30s
  timeout: 10s
  start_period: 30s
  retries: 3
```

### Database Services

```yaml
# PostgreSQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
  interval: 10s
  timeout: 5s
  retries: 5

# MySQL
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
  interval: 10s
  timeout: 5s
  retries: 5

# MongoDB
healthcheck:
  test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
  interval: 10s
  timeout: 5s
  retries: 5
```

### Message Queues

```yaml
# Redis
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 10s
  timeout: 5s
  retries: 5

# RabbitMQ
healthcheck:
  test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
  interval: 30s
  timeout: 10s
  retries: 5

# Kafka
healthcheck:
  test: ["CMD-SHELL", "kafka-broker-api-versions --bootstrap-server localhost:9092"]
  interval: 30s
  timeout: 10s
  retries: 5
```

---

## Networking Patterns

### Internal Network (Backend Isolation)

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No internet access

services:
  nginx:
    networks:
      - frontend
      - backend

  app:
    networks:
      - backend

  db:
    networks:
      - backend
```

### Custom Subnet

```yaml
networks:
  custom:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
          gateway: 172.28.0.1

services:
  app:
    networks:
      custom:
        ipv4_address: 172.28.0.10
```

---

## Volume Patterns

### Named Volumes with Driver Options

```yaml
volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/data/postgres

  # NFS mount
  shared_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/path/to/share"
```

### Bind Mount Best Practices

```yaml
services:
  app:
    volumes:
      # Read-only config
      - ./config:/app/config:ro
      
      # Cached for better performance on macOS
      - .:/app:cached
      
      # Delegated for write-heavy
      - ./logs:/app/logs:delegated
      
      # Anonymous volume to preserve container data
      - /app/node_modules
```

---

## Logging Configuration

### JSON File with Rotation

```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"
        compress: "true"
```

### Fluentd

```yaml
services:
  app:
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: myapp.{{.Name}}

  fluentd:
    image: fluent/fluentd:v1.16-1
    volumes:
      - ./fluentd/conf:/fluentd/etc
    ports:
      - "24224:24224"
```

---

## Docker Compose Watch (Development)

```yaml
# Compose Watch for hot reload (Compose 2.22+)
services:
  app:
    build: .
    develop:
      watch:
        # Rebuild on package changes
        - action: rebuild
          path: package.json
        
        # Sync source code
        - action: sync
          path: ./src
          target: /app/src
        
        # Sync and restart on config change
        - action: sync+restart
          path: ./config
          target: /app/config
```

```bash
# Start with watch mode
docker compose watch
```
