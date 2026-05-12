# Dockerfile Production Readiness Checklist

Quick audit checklist for reviewing Dockerfiles. Ordered by priority.

## P0 — Security (must fix)

- [ ] Non-root user: `USER` directive present and not root
- [ ] Base image pinned: both version tag AND SHA256 digest
- [ ] No `:latest` tag
- [ ] No secrets in ENV or ARG (API keys, tokens, passwords)
- [ ] `COPY` preferred over `ADD`
- [ ] No `curl | sh` or `wget -O- | bash` patterns
- [ ] Sensitive files excluded via `.dockerignore` (.env, .git, credentials)

## P1 — Size (should fix)

- [ ] Multi-stage build: build tools separated from runtime
- [ ] Minimal runtime base (distroless > alpine > slim > full)
- [ ] Package manager caches cleaned in same RUN layer
- [ ] Only necessary artifacts copied between stages
- [ ] Debug symbols stripped (`-ldflags="-w -s"` for Go, `strip` for C/Rust)
- [ ] Dev/test dependencies excluded from final image

## P2 — Build Speed (good to have)

- [ ] `.dockerignore` present and covers node_modules, .git, dist, .env
- [ ] Dependency installation layer before source copy (layer ordering)
- [ ] BuildKit cache mounts used for package managers
- [ ] `--no-install-recommends` (apt) or `--no-cache` (apk) used
- [ ] Single apt-get/apk RUN layer (update + install + clean)

## P3 — Maintainability

- [ ] HEALTHCHECK configured for the app type (HTTP, gRPC, process)
- [ ] OCI labels present (title, version, source)
- [ ] ENTRYPOINT for binary, CMD for default arguments
- [ ] Port exposed via EXPOSE
- [ ] Shell/curl included in runtime image if debugging needed
