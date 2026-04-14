# Docker for CI/CD

**Automating Build, Test, and Deploy Pipelines with Containers**

> Reproducible environments from commit to production -- eliminating "works on my machine" from every stage of your pipeline.

`Code` -> `Build` -> `Test` -> `Scan` -> `Push` -> `Deploy`

CI/CD - Pipelines - Automation - DevOps

---

## Table of Contents

1. [Why Docker in CI/CD Pipelines](#why-docker-in-cicd-pipelines)
2. [Docker-in-Docker vs Socket Binding](#docker-in-docker-vs-socket-binding)
3. [GitHub Actions with Docker](#github-actions-with-docker)
4. [GitLab CI with Docker](#gitlab-ci-with-docker)
5. [Jenkins with Docker Agents](#jenkins-with-docker-agents)
6. [Building Images in CI](#building-images-in-ci)
7. [Image Tagging Strategies](#image-tagging-strategies)
8. [Layer Caching in CI](#layer-caching-in-ci)
9. [Running Tests in Containers](#running-tests-in-containers)
10. [Docker Compose for Integration Testing](#docker-compose-for-integration-testing)
11. [Container Registries in CI](#container-registries-in-ci)
12. [Automated Security Scanning in Pipelines](#automated-security-scanning-in-pipelines)
13. [Deployment Strategies](#deployment-strategies)
14. [GitOps with Docker Images](#gitops-with-docker-images)
15. [CI/CD Pipeline Example Walkthrough](#cicd-pipeline-example-walkthrough)
16. [Performance Optimization Tips](#performance-optimization-tips)
17. [Summary & Further Reading](#summary--further-reading)

---

## Why Docker in CI/CD Pipelines

Containers solve the core CI/CD challenge: **consistent, reproducible environments** at every stage.

### Without Docker

- Snowflake build servers with drift
- "Works on my machine" failures in CI
- Dependency conflicts between projects
- Hours wasted debugging environment issues

### With Docker

- Identical environments: dev, CI, staging, prod
- Pinned dependencies via image layers
- Parallel jobs with isolated containers
- Immutable artifacts -- what you test is what you deploy

Every major CI platform -- GitHub Actions, GitLab CI, Jenkins, CircleCI -- has first-class Docker support.

---

## Docker-in-Docker vs Socket Binding

Two approaches to running Docker commands inside a CI container:

### Docker-in-Docker (DinD)

- Runs a full Docker daemon inside a container
- Uses `docker:dind` service image
- Complete isolation from host
- Slower startup, higher resource usage
- Layer cache lost between jobs

```yaml
# GitLab CI example
services:
  - docker:dind
variables:
  DOCKER_HOST: tcp://docker:2376
```

### Socket Binding

- Mounts host's `/var/run/docker.sock`
- Uses the host Docker daemon directly
- Faster -- shared layer cache
- **Security risk**: container has host-level access
- Simpler setup, better performance

```bash
# Docker run with socket mount
docker run -v /var/run/docker.sock:/var/run/docker.sock mybuilder
```

**Rule of thumb:** use DinD for untrusted workloads, socket binding for trusted internal CI.

---

## GitHub Actions with Docker

GitHub Actions runners come with Docker pre-installed. Build, test, and push in a single workflow.

```yaml
# .github/workflows/docker.yml
name: Build and Push
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Key actions:** `docker/login-action`, `docker/build-push-action`, `docker/setup-buildx-action`

---

## GitLab CI with Docker

GitLab CI has deep Docker integration via the `docker` executor and DinD services.

```yaml
# .gitlab-ci.yml
stages: [build, test, deploy]

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

test:
  stage: test
  image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  script:
    - npm test
```

**Built-in variables:** `$CI_REGISTRY_IMAGE`, `$CI_COMMIT_SHA`, `$CI_REGISTRY` are provided automatically.

---

## Jenkins with Docker Agents

Jenkins can spin up ephemeral Docker containers as build agents -- no permanent agent infrastructure needed.

```groovy
// Jenkinsfile (Declarative)
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-v $HOME/.npm:/root/.npm'  // cache mount
        }
    }
    stages {
        stage('Install') { steps { sh 'npm ci' } }
        stage('Test')    { steps { sh 'npm test' } }
        stage('Build Image') {
            agent any
            steps {
                script {
                    def img = docker.build("myapp:${env.BUILD_NUMBER}")
                    docker.withRegistry('https://registry.example.com', 'creds') {
                        img.push()
                        img.push('latest')
                    }
                }
            }
        }
    }
}
```

The **Docker Pipeline plugin** provides the `docker.build()` and `docker.withRegistry()` DSL.

---

## Building Images in CI

Best practices for reliable, fast image builds in CI pipelines:

### Multi-Stage Builds

- Separate build and runtime stages
- Smaller final images
- No build tools in production

### BuildKit

- Set `DOCKER_BUILDKIT=1`
- Parallel stage execution
- Better caching and secrets

### .dockerignore

- Exclude `.git/`, `node_modules/`
- Reduces build context size
- Faster uploads to daemon

```dockerfile
# Multi-stage Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --production=false
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

---

## Image Tagging Strategies

Good tagging makes images traceable, rollbackable, and auditable.

| Strategy | Format | When to Use | Example |
|----------|--------|-------------|---------|
| **Git SHA** | `:abc1234` | Every CI build -- unique, traceable | `myapp:a1b2c3d` |
| **Semver** | `:v1.2.3` | Releases -- human-readable versions | `myapp:v2.1.0` |
| **Branch** | `:main` | Mutable tag for latest on branch | `myapp:develop` |
| **latest** | `:latest` | Convenience -- **avoid in prod** | `myapp:latest` |
| **Date+SHA** | `:20260414-abc123` | Sortable + traceable | `myapp:20260414-a1b2c3d` |

```bash
# Multi-tag in CI
SHA=$(git rev-parse --short HEAD)
VERSION=$(cat VERSION)
docker build -t myapp:$SHA -t myapp:v$VERSION -t myapp:latest .
```

**Never deploy `:latest` to production.** Always use an immutable tag for traceability.

---

## Layer Caching in CI

CI runners are ephemeral -- without caching, every build starts from scratch. **Caching slashes build times by 50-80%.**

### GitHub Actions Cache

```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

Uses GitHub's native cache backend. Up to 10 GB per repo.

### Registry Cache

```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=registry,ref=ghcr.io/org/app:cache
    cache-to: type=registry,ref=ghcr.io/org/app:cache
```

Cache stored in registry. Shared across runners.

### BuildKit Inline Cache

```bash
docker build --build-arg BUILDKIT_INLINE_CACHE=1 \
  --cache-from myapp:latest -t myapp:new .
```

### Dockerfile Tips

- Copy `package.json` before source code
- Order layers: least-changing first
- Pin base image digests for reproducibility

---

## Running Tests in Containers

Run your test suite inside the same image you'll deploy -- guaranteeing environment parity.

### Unit Tests in Build Stage

```dockerfile
# Test stage in multi-stage build
FROM node:20-alpine AS test
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm test

FROM node:20-alpine AS production
COPY --from=test /app/dist ./dist
```

Build fails if tests fail -- image never gets created.

### Tests Against Running Container

```yaml
# GitHub Actions
steps:
  - run: docker build -t myapp .
  - run: |
      docker run -d --name app -p 3000:3000 myapp
      sleep 5
      curl -f http://localhost:3000/health
      docker run --network host postman/newman run tests.json
```

Smoke tests and API tests against the built image.

**Extract test results:** `docker cp container:/app/coverage ./coverage` to upload artifacts.

---

## Docker Compose for Integration Testing

Spin up your entire stack -- app, database, cache, message queue -- for true integration tests.

```yaml
# docker-compose.test.yml
services:
  app:
    build: .
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_started }
    environment:
      DATABASE_URL: postgres://test:test@db:5432/testdb
      REDIS_URL: redis://redis:6379

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 2s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine
```

```bash
# In CI pipeline
docker compose -f docker-compose.test.yml up --build --abort-on-container-exit --exit-code-from app
```

`--exit-code-from app` returns the test container's exit code so CI fails on test failure.

---

## Container Registries in CI

Where your images live between build and deploy:

| Registry | Provider | Auth Method | Best For |
|----------|----------|-------------|----------|
| **GHCR** | GitHub | `GITHUB_TOKEN` | GitHub-native workflows |
| **ECR** | AWS | IAM / OIDC | AWS deployments (ECS, EKS) |
| **ACR** | Azure | Service principal | Azure deployments (AKS, ACI) |
| **GAR** | Google Cloud | Workload identity | GCP deployments (GKE, Cloud Run) |
| **Docker Hub** | Docker Inc | Access token | Open-source, public images |

```bash
# ECR login (AWS)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# GHCR login
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
```

**Use OIDC federation** where possible -- no long-lived secrets to rotate.

---

## Automated Security Scanning in Pipelines

Shift security left: scan images **before** they reach production.

### Trivy

- OS and library CVEs
- Misconfigurations
- Secrets detection
- Free and open-source

### Snyk Container

- Deep dependency analysis
- Fix recommendations
- IDE and CI integration
- Free tier available

### Docker Scout

- Built into Docker Desktop
- SBOM generation
- Policy evaluation
- Integrated with Docker Hub

```yaml
# Trivy in GitHub Actions
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'        # Fail pipeline on findings
```

**Gate deployments:** block images with CRITICAL vulnerabilities from reaching production.

---

## Deployment Strategies

How you roll out new container versions determines risk and downtime:

### Rolling Update

- Gradually replace old pods/tasks
- Zero downtime
- Both versions run temporarily
- Kubernetes default strategy
- **Risk: Low**

### Blue-Green

- Two identical environments
- Switch traffic at load balancer
- Instant rollback
- Double the infrastructure cost
- **Risk: Very Low**

### Canary

- Route small % of traffic to new version
- Monitor metrics and errors
- Gradually increase traffic
- Rollback if anomalies detected
- **Risk: Lowest**

Pipeline flow: `Build` -> `Test` -> `Scan` -> `Deploy 5%` -> `Monitor` -> `Deploy 100%`

---

## GitOps with Docker Images

Git as the single source of truth: **declare desired state in a repo, let controllers reconcile.**

### How It Works

- CI builds and pushes image with SHA tag
- CI updates image tag in deployment manifests
- Git commit triggers reconciliation
- Controller (ArgoCD/Flux) applies changes
- Rollback = `git revert`

### Tools

- **ArgoCD** -- K8s-native, UI dashboard
- **Flux** -- lightweight, CNCF project
- **Kustomize** -- overlay-based config
- **Helm** -- chart-based templating

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/org/k8s-manifests
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## CI/CD Pipeline Example Walkthrough

A complete pipeline from commit to production:

`Checkout` -> `Lint` -> `Build` -> `Test` -> `Scan` -> `Push` -> `Deploy`

```yaml
# Complete GitHub Actions Pipeline
name: CI/CD
on:
  push:
    branches: [main]
  pull_request: {}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker run --rm -v $PWD:/app -w /app golangci/golangci-lint golangci-lint run

  build-and-test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v5
        with:
          load: true
          tags: "myapp:test"
          cache-from: "type=gha"
          cache-to: "type=gha,mode=max"
      - run: docker compose -f docker-compose.test.yml up --exit-code-from app

  scan:
    needs: build-and-test
    runs-on: ubuntu-latest
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: "myapp:test"
          severity: "CRITICAL"
          exit-code: "1"

  push:
    if: github.ref == 'refs/heads/main'
    needs: scan
    runs-on: ubuntu-latest
    steps:
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

---

## Performance Optimization Tips

Fast pipelines keep developers productive. Target: **< 10 minutes** from push to deploy-ready.

### Build Speed

- Use BuildKit (`DOCKER_BUILDKIT=1`)
- Enable layer caching (GHA, registry, local)
- Multi-stage builds to parallelize stages
- Use `--mount=type=cache` for package managers
- Pin base images by digest, not tag

### Image Size

- Use `-alpine` or `-slim` base images
- Distroless images for production
- Merge RUN layers to reduce intermediate files
- Remove package manager caches
- Use `docker image inspect --format` to audit

```dockerfile
# BuildKit cache mount for npm
RUN --mount=type=cache,target=/root/.npm \
    npm ci --production

# BuildKit cache mount for apt
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl
```

### CI Runner Tips

- Use larger runners for big builds
- Parallelize independent jobs
- Use matrix strategies for multi-platform

### Registry Tips

- Use same-region registry as runners
- Enable image manifest caching
- Clean up old tags with lifecycle policies

---

## Summary & Further Reading

Docker transforms CI/CD from fragile scripts into **reproducible, portable pipelines**.

### Key Takeaways

- Containers ensure identical environments across all stages
- Use multi-stage builds and BuildKit for fast, small images
- Tag images with git SHA for traceability
- Cache aggressively -- GHA cache, registry cache, mount cache
- Scan images before they reach production
- GitOps makes deployments auditable and reversible

### Further Reading

- [Docker CI/CD Documentation](https://docs.docker.com/build/ci/)
- [GitHub Actions Docker Guide](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images)
- [GitLab CI Docker Builds](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Trivy Security Scanner](https://aquasecurity.github.io/trivy/)
- [Docker Build Cache Guide](https://docs.docker.com/build/cache/)
