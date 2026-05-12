---
name: dockerfile-optimizer
description: >
  Use when the user wants to optimize, analyze, or create Dockerfiles. Triggers include:
  "Dockerfile", "docker build", "镜像优化", "镜像太大", "安全加固", "multi-stage build",
  "distroless", "containerize", "容器化", "docker 优化", "减小镜像体积". Also trigger when
  the user shares an existing Dockerfile and asks for improvements, or describes a service
  and wants to containerize it. Do NOT use for pure CI/CD pipeline generation — use cicd-pipeline instead.
---

# Dockerfile Optimizer

## Overview

This skill generates production-grade Dockerfiles and optimizes existing ones. It doesn't just produce
syntactically valid `Dockerfile`s — it encodes layer caching, multi-stage builds, non-root execution,
minimal base images, and security hardening that take engineers months to get right.

---

## Decision Flow

```dot
digraph dockerfile_flow {
    "User input received" [shape=doublecircle];
    "Has existing Dockerfile?" [shape=diamond];
    "Analyze existing" [shape=box];
    "Language known?" [shape=diamond];
    "Ask for language/framework" [shape=box];
    "Generate from template" [shape=box];
    "Apply security hardening" [shape=box];
    "Apply size optimization" [shape=box];
    "Apply build efficiency" [shape=box];
    "Deliver: optimized file + change log + comparison" [shape=doublecircle];

    "User input received" -> "Has existing Dockerfile?";
    "Has existing Dockerfile?" -> "Analyze existing" [label="yes"];
    "Has existing Dockerfile?" -> "Language known?" [label="no"];
    "Language known?" -> "Ask for language/framework" [label="no"];
    "Language known?" -> "Generate from template" [label="yes"];
    "Ask for language/framework" -> "Generate from template";
    "Analyze existing" -> "Apply security hardening";
    "Generate from template" -> "Apply security hardening";
    "Apply security hardening" -> "Apply size optimization";
    "Apply size optimization" -> "Apply build efficiency";
    "Apply build efficiency" -> "Deliver: optimized file + change log + comparison";
}
```

---

## Quick Reference

| Language | Base (Build) | Base (Runtime) | Package Manager | Key optimization |
|----------|-------------|----------------|-----------------|------------------|
| Go | `golang:1.23-alpine` | `gcr.io/distroless/static-debian12` | `go mod` | CGO_ENABLED=0, upx optional |
| Node.js | `node:22-alpine` | `node:22-alpine` | npm / yarn / pnpm | `npm ci --omit=dev`, pnpm deploy |
| Python | `python:3.12-slim` | `python:3.12-slim` | pip / poetry | venv copy, `--no-cache-dir` |
| Java/Maven | `maven:3.9-eclipse-temurin-21` | `eclipse-temurin:21-jre-alpine` | mvn | `mvn dependency:go-offline` first |
| Java/Gradle | `gradle:8-jdk21-alpine` | `eclipse-temurin:21-jre-alpine` | gradle | `gradle dependencies` first |
| Rust | `rust:1.78-alpine` | `alpine:3.20` | cargo | `cargo chef` for caching, musl target |

---

## Workflow: Optimizing an Existing Dockerfile

### Step 1 — Audit Categories

Scan the existing Dockerfile and classify every line by category:

| Category | Red flags to detect |
|----------|-------------------|
| **Security** | `USER root`, no USER directive, `latest` tag, ENV with secrets, port < 1024, `RUN wget \| sh` |
| **Size** | Single stage, `ubuntu`/`debian` base without slim, dev dependencies in final image, `apt-get install` without cleanup |
| **Build Speed** | No `.dockerignore`, no cache mount, COPY . too early, `apt-get update` not cached |
| **Maintainability** | Missing HEALTHCHECK, no OCI labels, `CMD` without `ENTRYPOINT`, hardcoded versions |

### Step 2 — Generate Optimized Version

Produce the corrected Dockerfile with inline comments explaining each change.

### Step 3 — Deliver Comparison

Output:
1. The optimized Dockerfile
2. A change log: what was changed and why
3. Estimated impact table (size reduction, build time, security posture)

---

## Workflow: Generating from Scratch

### Step 1 — Gather Context

**Required:**
- Language / framework (Go, Node.js, Python, Java/Maven, Java/Gradle, Rust)
- Build command (infer from language if not provided)
- Port the app listens on

**Infer from context if possible, ask only if ambiguous:**
- Package manager (npm vs yarn vs pnpm; pip vs poetry)
- Base image preference (alpine vs distroless vs slim)
- Need for shell/debug tools in final image

**Optional enhancements (offer proactively):**
- Non-root user setup
- HEALTHCHECK configuration
- docker-compose.yml for local dev with hot-reload
- .dockerignore generation

### Step 2 — Select Template

Use the matrix in Quick Reference to pick the base template.

### Step 3 — Generate with Best Practices Injected

Every generated Dockerfile MUST include:

**Security:**
- Non-root user (UID 1000:1000 or language default)
- NEVER `USER root` past the build stage
- Pin base image to SHA digest AND version tag
- NEVER use `:latest` — always specify exact version
- `COPY` not `ADD` unless remote URL needed
- No secrets in ENV — use build args with care

**Build efficiency:**
- `.dockerignore` generated alongside Dockerfile
- Dependencies copied/installed BEFORE source code (layer ordering)
- BuildKit cache mounts for package managers (`RUN --mount=type=cache,target=/root/.cache/...`)
- Multi-stage builds: build in stage 1, copy only artifacts to stage 2

**Size:**
- Minimal runtime base (distroless > alpine > slim > full OS)
- Clean package manager caches in same RUN layer (`rm -rf /var/lib/apt/lists/*`)
- No build tools in runtime image
- Copy only the binary/venv/dist — not the entire working tree

**Maintainability:**
- HEALTHCHECK instruction tuned to the framework (HTTP for web, TCP for gRPC, exec for workers)
- OCI labels: `org.opencontainers.image.*`
- `ENTRYPOINT` for the binary, `CMD` for default arguments
- EXPOSE the correct port

---

## Templates

### Go

```dockerfile
# ===== Stage 1: Build =====
FROM golang:1.23-alpine@sha256:abc123... AS builder

RUN apk add --no-cache git ca-certificates

WORKDIR /src

# Cache dependencies (layer reuse until go.sum changes)
COPY go.mod go.sum ./
RUN go mod download

# Copy source and build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-w -s" -o /app ./cmd/server

# ===== Stage 2: Runtime =====
FROM gcr.io/distroless/static-debian12@sha256:def456... AS runtime

# distroless is already non-root, but copy certs if needed
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app /app

EXPOSE 8080

# distroless has no shell — entrypoint must be the binary
ENTRYPOINT ["/app"]
```

### Node.js

```dockerfile
# ===== Stage 1: Build / Dependencies =====
FROM node:22-alpine@sha256:abc123... AS deps

# Create non-root user early
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001

WORKDIR /app

# Cache npm install (layer reuse until package-lock.json changes)
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

# Copy only production source
COPY src/ ./src/

# ===== Stage 2: Runtime =====
FROM node:22-alpine@sha256:def456... AS runtime

RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001

WORKDIR /app
COPY --from=deps --chown=nodejs:nodejs /app /app

USER nodejs
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

ENTRYPOINT ["node"]
CMD ["src/index.js"]
```

### Python

```dockerfile
# ===== Stage 1: Build =====
FROM python:3.12-slim@sha256:abc123... AS builder

RUN useradd --create-home --shell /bin/bash appuser

# Create venv
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install dependencies into venv
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt

# ===== Stage 2: Runtime =====
FROM python:3.12-slim@sha256:def456... AS runtime

RUN useradd --create-home --shell /bin/bash appuser

# Copy only the venv — no pip/build-essential in runtime
COPY --from=builder /opt/venv /opt/venv
COPY src/ /app/src/

ENV PATH="/opt/venv/bin:$PATH"
WORKDIR /app

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

ENTRYPOINT ["python"]
CMD ["-m", "src.server"]
```

### Java (Maven)

```dockerfile
# ===== Stage 1: Build =====
FROM maven:3.9-eclipse-temurin-21@sha256:abc123... AS builder

WORKDIR /src

# Cache Maven dependencies (layer reuse until pom.xml changes)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source and build
COPY src/ ./src/
RUN mvn package -DskipTests -B

# ===== Stage 2: Runtime =====
FROM eclipse-temurin:21-jre-alpine@sha256:def456... AS runtime

RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001 -G appgroup

WORKDIR /app
COPY --from=builder --chown=appuser:appgroup /src/target/*-runner.jar app.jar

USER appuser
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

### Java (Gradle)

```dockerfile
# ===== Stage 1: Build =====
FROM gradle:8-jdk21-alpine@sha256:abc123... AS builder

WORKDIR /src

# Cache Gradle dependencies
COPY build.gradle settings.gradle ./
COPY gradle/ gradle/
RUN gradle dependencies --no-daemon

# Copy source and build
COPY src/ ./src/
RUN gradle build --no-daemon -x test

# ===== Stage 2: Runtime =====
FROM eclipse-temurin:21-jre-alpine@sha256:def456... AS runtime

RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001 -G appgroup

WORKDIR /app
COPY --from=builder --chown=appuser:appgroup /src/build/libs/*-all.jar app.jar

USER appuser
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

### Rust

```dockerfile
# ===== Stage 1: Build =====
FROM rust:1.78-alpine@sha256:abc123... AS builder

RUN apk add --no-cache musl-dev

WORKDIR /src

# Use cargo-chef for dependency caching
RUN cargo install cargo-chef
COPY Cargo.toml Cargo.lock ./
RUN cargo chef prepare --recipe-path recipe.json
RUN cargo chef cook --release --target x86_64-unknown-linux-musl --recipe-path recipe.json

# Copy source and build
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

# ===== Stage 2: Runtime =====
FROM alpine:3.20@sha256:def456... AS runtime

RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001 -G appgroup

COPY --from=builder /src/target/x86_64-unknown-linux-musl/release/app /app

USER appuser
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

ENTRYPOINT ["/app"]
```

---

## Security Rules (Enforced)

### 1. NEVER Run as Root

```dockerfile
# ❌ BAD
FROM node:22-alpine

# ✅ GOOD
FROM node:22-alpine
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
USER nodejs
```

### 2. Pin Base Images

```dockerfile
# ❌ BAD
FROM golang:latest
FROM node:22

# ✅ GOOD (tag + digest for supply-chain security)
FROM golang:1.23-alpine@sha256:e8a2b...
FROM node:22-alpine@sha256:f1c3d...
```

### 3. Never Leak Secrets

```dockerfile
# ❌ BAD
ENV DATABASE_URL=postgres://user:pass@host/db
ARG TOKEN=abc123

# ✅ GOOD
# Secrets injected at runtime via orchestrator (K8s Secret, Docker Swarm secret, etc.)
ENV DATABASE_URL=""
# Use build args only for non-sensitive values:
ARG NODE_VERSION=22
```

### 4. Use COPY, not ADD

```dockerfile
# ❌ BAD (ADD unpacks tarballs, fetches URLs — surprising behavior)
ADD https://example.com/script.sh /app/

# ✅ GOOD
COPY ./script.sh /app/
# If remote fetch is unavoidable, use RUN wget in the same layer
```

### 5. No sudo / wget | sh Patterns

```dockerfile
# ❌ BAD — arbitrary code execution from network
RUN curl -sSL https://install.example.com | sh

# ✅ GOOD — verify checksum
RUN curl -sSL https://install.example.com/installer -o installer && \
    echo "expected_sha256  installer" | sha256sum -c && \
    chmod +x installer && \
    ./installer && \
    rm installer
```

---

## Build Efficiency Rules

### 1. .dockerignore Generated Automatically

When generating a Dockerfile, also generate a `.dockerignore`:

```dockerignore
# Git
.git
.gitignore
.gitattributes

# Dependencies
node_modules/
.pnpm-store/

# Build artifacts
dist/
target/
build/
*.exe
*.dll

# IDE
.vscode/
.idea/
*.swp
*.swo

# Docs & Tests
*.md
LICENSE
tests/
__tests__/
**/*.test.*
**/*.spec.*

# Misc
.env
.env.*
docker-compose*.yml
Dockerfile*
.dockerignore

# CI
.github/
.gitlab-ci.yml
Jenkinsfile
```

### 2. Layer Ordering

```dockerfile
# ❌ BAD — source changes invalidate dependency cache every build
COPY . .
RUN npm ci

# ✅ GOOD — dependencies cached until lockfile changes
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

### 3. BuildKit Cache Mounts

```dockerfile
# ❌ BAD — every build downloads fresh
RUN pip install -r requirements.txt

# ✅ GOOD — pip cache survives between builds
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

| Package manager | Cache mount target |
|----------------|-------------------|
| npm | `/root/.npm` |
| yarn | `/root/.yarn` |
| pnpm | `/root/.local/share/pnpm/store` |
| pip | `/root/.cache/pip` |
| go mod | GOMODCACHE=`/go/pkg/mod`; GOCACHE=`/root/.cache/go-build` |
| maven | `/root/.m2` |
| gradle | `/root/.gradle/caches` |
| cargo | `/usr/local/cargo/registry` |

### 4. Single apt-get Layer

```dockerfile
# ❌ BAD — each RUN creates a layer
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# ✅ GOOD — single layer, clean cache in same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

---

## Size Optimization

### Base Image Selection Guide

```
┌──────────────────────────────────────────────────────┐
│  Base image size hierarchy (smallest → largest)       │
│                                                       │
│  scratch  <  distroless  <  alpine  <  slim  <  full  │
│  0 MB          ~3 MB        ~7 MB    ~25 MB   ~100 MB │
│                                                       │
│  Trade-offs:                                          │
│  scratch:   no shell, no CA certs, no libc            │
│  distroless: no shell, has CA certs, glibc, non-root  │
│  alpine:    has shell, musl (not glibc), apk          │
│  slim:      has shell, glibc, apt, bigger             │
└──────────────────────────────────────────────────────┘
```

| Language | Recommended runtime base | Why |
|----------|-------------------------|-----|
| Go (statically linked) | `distroless/static` or `scratch` | No libc dependency with CGO_ENABLED=0 |
| Go (cgo needed) | `alpine` | musl libc |
| Node.js | `node:XX-alpine` | Needs Node runtime |
| Python | `python:XX-slim` | Needs Python runtime; alpine musl can cause wheel issues |
| Java/JRE | `eclipse-temurin:XX-jre-alpine` | JRE-only, smaller than JDK |
| Rust (statically linked) | `alpine` after musl build | Small, has certs for TLS |

### Reduce Image Size Checklist

- [ ] Use multi-stage build (build tools never reach final image)
- [ ] Copy only necessary artifacts between stages (not `COPY --from=builder /src /src`)
- [ ] Clean package manager caches in same RUN layer
- [ ] Remove temp files, build caches, test files before final stage
- [ ] Use `--no-install-recommends` (apt) or `--no-cache` (apk)
- [ ] Strip debug symbols: `-ldflags="-w -s"` (Go), `strip` (C/Rust binaries)

---

## Maintainability

### HEALTHCHECK

| App type | HEALTHCHECK pattern |
|----------|-------------------|
| HTTP API | `wget --spider http://localhost:PORT/health` |
| gRPC | `grpc_health_probe -addr=localhost:PORT` |
| Worker (no port) | `pgrep app || exit 1` |

Default intervals: `--interval=30s --timeout=3s --start-period=5s --retries=3`

### OCI Labels

```dockerfile
LABEL org.opencontainers.image.title="my-service"
LABEL org.opencontainers.image.description="Handles payment processing"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.source="https://github.com/org/repo"
LABEL org.opencontainers.image.authors="team@example.com"
```

### ENTRYPOINT vs CMD

```dockerfile
# ✅ Correct pattern: ENTRYPOINT = binary, CMD = default args (overridable)
ENTRYPOINT ["/app"]
CMD ["--config", "/etc/app/config.yaml"]

# Container run: docker run my-img → /app --config /etc/app/config.yaml
# Container run: docker run my-img --debug → /app --debug
```

---

## Common Mistakes & Troubleshooting

**"Why is my image 1.2 GB?"**
→ Probably using a full OS base (`ubuntu`, `debian`) without multi-stage. Check: are build tools (gcc, maven, pip) in the final image? Switch to multi-stage + distroless/alpine.

**"Layer cache never hits"**
→ Check COPY order. If `COPY . .` comes before `RUN npm ci`, every source change invalidates the dependency layer. Copy lockfiles first, install deps, then copy source.

**"Permission denied (running as non-root)"**
→ Port < 1024 requires root. Bind to port >= 1024 (8080, 3000, etc.) or use `setcap`. Ensure all copied files are owned by the non-root user: `COPY --chown=appuser:appgroup`.

**"Container works locally but fails on EKS/GKE"**
→ Missing CA certificates in scratch/distroless. Copy certs from build stage: `COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/`.

**"Python alpine can't install my package"**
→ Some Python wheels require glibc and fail on alpine's musl. Use `python:XX-slim` instead.

**"BuildKit cache mount says 'not supported'"**
→ Requires BuildKit enabled: `DOCKER_BUILDKIT=1 docker build ...` or set `{ "features": { "buildkit": true } }` in Docker daemon config.

---

## Output Format

When delivering, always provide:

### 1. Optimized Dockerfile
The full file with inline comments on non-obvious decisions.

### 2. Change Log

| Line | Issue | Fix | Impact |
|------|-------|-----|--------|
| 1 | `FROM golang:latest` | `FROM golang:1.23-alpine@sha256:...` | Deterministic builds, smaller base |
| 5 | `COPY . .` before deps | Reordered: lockfile → install → source | Cache hits on deps layer |
| 8 | Missing USER | `USER 1000:1000` | Non-root execution |
| 12 | No HEALTHCHECK | Added HTTP health check | Orchestrator can detect stalls |

### 3. Estimated Impact

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Image size | ~900 MB | ~25 MB | 97% reduction |
| Build time (cached) | ~180s | ~15s | 92% faster |
| Security (Trivy) | 12 HIGH, 3 CRIT | 0 HIGH, 0 CRIT | Clean scan |
| Root user | Yes | No | Hardened |

---

## Required Files Generated Alongside

When generating a Dockerfile from scratch, always offer to create:

1. `Dockerfile` — the optimized build file
2. `.dockerignore` — prevent context bloat
3. `docker-compose.yml` (optional) — for local dev with hot-reload
