# Example 1: API Service Runbook

## Input

User says:
> "My payment-service is a Go REST API on EKS, connects to RDS Postgres and ElastiCache Redis. It's called by the API Gateway. Create a runbook for the on-call team."

## Claude Output (with incident-runbook skill)

Generates a complete 8-section runbook including:
1. Service dependency graph (EKS → RDS, ElastiCache; API Gateway → payment-service)
2. Alert-to-runbook mapping (HighErrorRate, HighLatency, OOMKilled, InstanceDown, DBConnectionExhausted)
3. Per-incident diagnosis steps with actual `kubectl` and `psql` commands
4. Mitigation procedures (rollback, scale, circuit-break)
5. Escalation paths (L1 → L2 → L3 → Infra)
6. Slack communication templates
7. Postmortem template
8. Dashboard links placeholder
