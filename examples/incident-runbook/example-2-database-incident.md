# Example 2: Database Incident Runbook

## Input

User says:
> "We had a Postgres outage last week and nobody knew what to check. Generate a runbook for RDS Postgres incidents."

## Claude Output (with incident-runbook skill)

Generates a Postgres-specific runbook covering:
- Connection exhaustion diagnosis (`pg_stat_activity`, pool config)
- Slow query identification (`pg_stat_statements`, `EXPLAIN ANALYZE`)
- Replication lag monitoring and recovery
- Disk full mitigation (log rotation, WAL cleanup)
- Failover procedure (Multi-AZ forced failover)
- Point-in-time recovery steps
