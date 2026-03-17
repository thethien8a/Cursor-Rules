# Docker CI/CD Integration

## GitHub Actions

### Basic Build and Push

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix=
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Multi-Platform Build

```yaml
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    platforms: linux/amd64,linux/arm64
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### With Security Scanning

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true
          tags: myapp:${{ github.sha }}

      - name: Scan for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Fail on critical
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          exit-code: '1'
          severity: 'CRITICAL'
```

### Build Matrix for Multiple Images

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - image: api
            context: ./services/api
          - image: worker
            context: ./services/worker
          - image: web
            context: ./services/web

    steps:
      - uses: actions/checkout@v4

      - name: Build ${{ matrix.image }}
        uses: docker/build-push-action@v5
        with:
          context: ${{ matrix.context }}
          push: true
          tags: ghcr.io/${{ github.repository }}/${{ matrix.image }}:${{ github.sha }}
          cache-from: type=gha,scope=${{ matrix.image }}
          cache-to: type=gha,mode=max,scope=${{ matrix.image }}
```

---

## Caching Strategies

### GitHub Actions Cache (Recommended)

```yaml
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**Cache modes**:
- `mode=min` - Only cache final result (smaller, faster)
- `mode=max` - Cache all intermediate layers (larger, better hit rate)

### Registry Cache

```yaml
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myregistry/myimage:latest
    cache-from: type=registry,ref=myregistry/myimage:buildcache
    cache-to: type=registry,ref=myregistry/myimage:buildcache,mode=max
```

### Local Cache (Self-hosted runners)

```yaml
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=local,src=/tmp/.buildx-cache
    cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max

# Prevent cache from growing indefinitely
- name: Move cache
  run: |
    rm -rf /tmp/.buildx-cache
    mv /tmp/.buildx-cache-new /tmp/.buildx-cache
```

---

## GitLab CI

### Basic Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - scan
  - push

variables:
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t $IMAGE_TAG .
    - docker save $IMAGE_TAG > image.tar
  artifacts:
    paths:
      - image.tar
    expire_in: 1 hour

test:
  stage: test
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker load < image.tar
    - docker run --rm $IMAGE_TAG npm test

scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity CRITICAL $IMAGE_TAG
  allow_failure: true

push:
  stage: push
  image: docker:24
  services:
    - docker:24-dind
  only:
    - main
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker load < image.tar
    - docker push $IMAGE_TAG
    - docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
```

### With BuildKit and Cache

```yaml
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_BUILDKIT: 1
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      docker build \
        --cache-from $CI_REGISTRY_IMAGE:buildcache \
        --build-arg BUILDKIT_INLINE_CACHE=1 \
        -t $IMAGE_TAG \
        -t $CI_REGISTRY_IMAGE:buildcache \
        .
    - docker push $IMAGE_TAG
    - docker push $CI_REGISTRY_IMAGE:buildcache
```

---

## Docker Bake (Complex Builds)

### Bake File

```hcl
# docker-bake.hcl
variable "TAG" {
  default = "latest"
}

variable "REGISTRY" {
  default = "ghcr.io/myorg"
}

group "default" {
  targets = ["api", "worker", "web"]
}

target "base" {
  context = "."
  dockerfile = "Dockerfile.base"
}

target "api" {
  inherits = ["base"]
  context = "./services/api"
  tags = ["${REGISTRY}/api:${TAG}"]
  cache-from = ["type=registry,ref=${REGISTRY}/api:buildcache"]
  cache-to = ["type=registry,ref=${REGISTRY}/api:buildcache,mode=max"]
}

target "worker" {
  inherits = ["base"]
  context = "./services/worker"
  tags = ["${REGISTRY}/worker:${TAG}"]
}

target "web" {
  context = "./services/web"
  dockerfile = "Dockerfile"
  tags = ["${REGISTRY}/web:${TAG}"]
  platforms = ["linux/amd64", "linux/arm64"]
}

# Matrix build
target "app" {
  name = "app-${tgt}"
  matrix = {
    tgt = ["api", "worker", "web"]
  }
  context = "./services/${tgt}"
  tags = ["${REGISTRY}/${tgt}:${TAG}"]
}
```

### Bake in CI

```yaml
# GitHub Actions
- name: Build all images
  uses: docker/bake-action@v4
  with:
    files: docker-bake.hcl
    push: true
  env:
    TAG: ${{ github.sha }}
```

---

## Best Practices

### Tagging Strategy

```yaml
tags: |
  # Branch name
  type=ref,event=branch
  
  # PR number
  type=ref,event=pr
  
  # Git SHA (short)
  type=sha,prefix=
  
  # Semantic version from tag
  type=semver,pattern={{version}}
  type=semver,pattern={{major}}.{{minor}}
  
  # Latest on main
  type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
```

### Build Arguments from Git

```yaml
- name: Build
  uses: docker/build-push-action@v5
  with:
    build-args: |
      GIT_SHA=${{ github.sha }}
      GIT_REF=${{ github.ref_name }}
      BUILD_DATE=${{ github.event.head_commit.timestamp }}
```

```dockerfile
ARG GIT_SHA
ARG BUILD_DATE
LABEL org.opencontainers.image.revision=$GIT_SHA
LABEL org.opencontainers.image.created=$BUILD_DATE
```

### Secrets in Build

```yaml
- name: Build with secrets
  uses: docker/build-push-action@v5
  with:
    context: .
    secrets: |
      npm_token=${{ secrets.NPM_TOKEN }}
      github_token=${{ secrets.GITHUB_TOKEN }}
```

```dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm ci
```

---

## Deployment Patterns

### Rolling Update with Docker Compose

```yaml
deploy:
  runs-on: ubuntu-latest
  steps:
    - name: Deploy to server
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.SSH_KEY }}
        script: |
          cd /app
          docker compose pull
          docker compose up -d --remove-orphans
          docker image prune -f
```

### Kubernetes Deployment

```yaml
- name: Deploy to Kubernetes
  uses: azure/k8s-deploy@v4
  with:
    namespace: production
    manifests: |
      k8s/deployment.yaml
      k8s/service.yaml
    images: |
      ghcr.io/${{ github.repository }}:${{ github.sha }}
```

### Watchtower (Auto-update)

```yaml
# docker-compose.yml on server
services:
  watchtower:
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - WATCHTOWER_POLL_INTERVAL=300
      - WATCHTOWER_CLEANUP=true
```
