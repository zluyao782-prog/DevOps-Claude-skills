# K8s Production Best Practices Quick Reference

## Pod Security (P0)

- [ ] `runAsNonRoot: true`
- [ ] `runAsUser: 1000` (or equivalent non-zero)
- [ ] `seccompProfile.type: RuntimeDefault`
- [ ] `allowPrivilegeEscalation: false`
- [ ] `capabilities.drop: [ALL]`
- [ ] `readOnlyRootFilesystem: true` (with emptyDir for /tmp if needed)
- [ ] `automountServiceAccountToken: false` (unless API access needed)
- [ ] Dedicated `serviceAccountName` (not default)

## Probes (P0)

- [ ] `livenessProbe` configured (different from readiness for web apps)
- [ ] `readinessProbe` configured (fail fast if dependency down)
- [ ] `startupProbe` for slow-starting apps (Java, ML models)

## Resources (P1)

- [ ] `resources.requests` set for CPU and memory
- [ ] `resources.limits` set for memory (at minimum)
- [ ] CPU requests = CPU limits for Guaranteed QoS in production

## Availability (P1)

- [ ] `replicas >= 2` for production
- [ ] `PodDisruptionBudget` for HA services
- [ ] `topologySpreadConstraints` across zones
- [ ] `podAntiAffinity` for critical services (soft preferred)
- [ ] `terminationGracePeriodSeconds >= 30`

## Networking (P1)

- [ ] `NetworkPolicy` with default-deny ingress in namespace
- [ ] Named ports on container and service
- [ ] TLS on Ingress (cert-manager for auto-issue)

## Observability (P2)

- [ ] `app.kubernetes.io/name` and `app.kubernetes.io/version` labels
- [ ] Prometheus metrics port exposed (if applicable)
- [ ] Structured logging to stdout (not files)
