# Example 2: Generate Dockerfile from Scratch

## Input

User says:
> "I have a Go REST API service that connects to Postgres, listens on port 8080, has a /healthz endpoint. Containerize it with best practices."

## Claude Output (with dockerfile-optimizer skill)

### Dockerfile

```dockerfile
# ===== Stage 1: Build =====
FROM golang:1.23-alpine@sha256:e8a2b... AS builder

RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /src

# Layer 1: Dependencies (cached until go.sum changes)
COPY go.mod go.sum ./
RUN go mod download

# Layer 2: Source & build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -o /app ./cmd/server

# ===== Stage 2: Runtime =====
FROM gcr.io/distroless/static-debian12@sha256:def456... AS runtime

# Copy certs (for HTTPS outbound) and timezone data
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /app /app

EXPOSE 8080

# distroless is non-root by default, has no shell
ENTRYPOINT ["/app"]
```

### .dockerignore

```dockerignore
.git/
.github/
*.md
LICENSE
.env*
docker-compose*.yml
Dockerfile*
examples/
docs/
```

### Generated Files Summary

| File | Purpose |
|------|---------|
| `Dockerfile` | Multi-stage, distroless runtime, static binary, non-root |
| `.dockerignore` | Excludes git, docs, env files, docker-compose |

### Key Decisions

| Decision | Why |
|----------|-----|
| `CGO_ENABLED=0` | Produce statically linked binary — no libc dependency |
| `distroless/static` runtime | ~3 MB, no shell, non-root, has CA certs for TLS |
| `-ldflags="-w -s"` | Strip debug symbols — smaller binary |
| No HEALTHCHECK? | distroless has no `wget` or `curl` — suggest adding `/healthz` endpoint; orchestrator (K8s) can probe via HTTP |

### Recommended Companion (K8s)

```yaml
# In your Deployment:
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 15
```
