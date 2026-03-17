# Language-Specific Docker Patterns

## Python with uv (Recommended 2025)

### Production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim AS builder

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Set uv environment variables
ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never

WORKDIR /app

# Install dependencies (cached layer)
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-install-project --no-dev

# Copy application and sync
COPY . .
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

# Production stage
FROM python:3.13-slim AS production

# Create non-root user
RUN groupadd -g 1001 app && useradd -u 1001 -g app -s /bin/false app

WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder --chown=app:app /app/.venv /app/.venv
COPY --from=builder --chown=app:app /app /app

# Set PATH to use venv
ENV PATH="/app/.venv/bin:$PATH"

USER 1001
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Key uv Optimizations

| Variable | Purpose |
|----------|---------|
| `UV_COMPILE_BYTECODE=1` | Pre-compile .pyc for faster startup |
| `UV_LINK_MODE=copy` | Copy instead of hardlink (cross-filesystem safe) |
| `UV_PYTHON_DOWNLOADS=never` | Use system Python, don't download |

### Alternative: Poetry

```dockerfile
FROM python:3.13-slim AS builder

RUN pip install poetry
ENV POETRY_VIRTUALENVS_IN_PROJECT=true

WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN poetry install --no-interaction --no-ansi --only=main

FROM python:3.13-slim
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
CMD ["python", "-m", "app"]
```

---

## Node.js with pnpm (Recommended 2025)

### Production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@latest --activate
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"

# Dependencies stage
FROM base AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
    pnpm install --frozen-lockfile --prod

# Build stage
FROM base AS build
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
    pnpm install --frozen-lockfile
COPY . .
RUN pnpm run build

# Production stage
FROM node:22-alpine AS production
RUN addgroup -g 1001 -S app && adduser -S app -u 1001 -G app

WORKDIR /app

COPY --from=deps --chown=app:app /app/node_modules ./node_modules
COPY --from=build --chown=app:app /app/dist ./dist
COPY --from=build --chown=app:app /app/package.json ./

USER 1001
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

### Key pnpm Optimizations

- `corepack enable` - Use Node.js built-in package manager management
- `--frozen-lockfile` - Fail if lockfile needs update (reproducible)
- `--mount=type=cache,id=pnpm,target=/pnpm/store` - Cache pnpm store

### Alternative: npm

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build && npm prune --production

FROM node:22-alpine
RUN addgroup -g 1001 -S app && adduser -S app -u 1001
WORKDIR /app
COPY --from=deps --chown=app:app /app/node_modules ./node_modules
COPY --from=build --chown=app:app /app/dist ./dist
USER 1001
CMD ["node", "dist/index.js"]
```

---

## Go (Static Binary)

### Production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.23-alpine AS builder

# Add certificates for HTTPS
RUN apk add --no-cache ca-certificates

WORKDIR /app

# Download dependencies
COPY go.mod go.sum ./
RUN go mod download

# Build static binary
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags='-w -s -extldflags "-static"' \
    -o /app/server ./cmd/server

# Minimal production image
FROM scratch

# Copy certificates
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy binary
COPY --from=builder /app/server /server

# Non-root user (numeric for scratch)
USER 65534:65534

EXPOSE 8080
ENTRYPOINT ["/server"]
```

### Key Go Optimizations

- `CGO_ENABLED=0` - Pure Go, no C dependencies
- `-ldflags='-w -s'` - Strip debug info, reduce binary size
- `FROM scratch` - Smallest possible image (~10MB total)

---

## Rust (Static Binary)

### Production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM rust:1.83-alpine AS builder

RUN apk add --no-cache musl-dev

WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release && rm -rf src

COPY src ./src
RUN touch src/main.rs && cargo build --release

FROM scratch
COPY --from=builder /app/target/release/myapp /myapp
USER 65534:65534
ENTRYPOINT ["/myapp"]
```

---

## Java (Spring Boot)

### Production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /app
COPY gradlew build.gradle.kts settings.gradle.kts ./
COPY gradle ./gradle
RUN --mount=type=cache,target=/root/.gradle \
    ./gradlew dependencies --no-daemon

COPY src ./src
RUN --mount=type=cache,target=/root/.gradle \
    ./gradlew bootJar --no-daemon

# Extract layers for better caching
RUN java -Djarmode=layertools -jar build/libs/*.jar extract

FROM eclipse-temurin:21-jre-alpine
RUN addgroup -g 1001 -S app && adduser -S app -u 1001 -G app

WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

USER 1001
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

### Key Java Optimizations

- Use JRE (not JDK) for runtime
- Extract Spring Boot layers for better caching
- `--no-daemon` prevents Gradle daemon in container

---

## .NET

### Production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src

COPY *.csproj ./
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine
RUN addgroup -g 1001 -S app && adduser -S app -u 1001 -G app

WORKDIR /app
COPY --from=build --chown=app:app /app .

USER 1001
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

---

## Package Manager Cache Mounts Summary

| Language | Cache Mount Target |
|----------|-------------------|
| Python (uv) | `/root/.cache/uv` |
| Python (pip) | `/root/.cache/pip` |
| Python (poetry) | `/root/.cache/pypoetry` |
| Node.js (npm) | `/root/.npm` |
| Node.js (pnpm) | `/pnpm/store` |
| Node.js (yarn) | `/root/.yarn` |
| Go | `/go/pkg/mod` |
| Rust | `/usr/local/cargo/registry` |
| Java (Gradle) | `/root/.gradle` |
| Java (Maven) | `/root/.m2` |
| .NET | `/root/.nuget/packages` |
