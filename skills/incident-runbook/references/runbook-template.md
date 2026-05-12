# Runbook Template (Minimal)

Copy this for each alert/incident type.

```markdown
## <Alert/Incident Name>

**Severity:** P1 / P2 / P3
**Alert source:** <Prometheus/Datadog/CloudWatch alert name>

### Triage (2 min)
- [ ] Recent deploy?
- [ ] Cloud provider incident?
- [ ] Dependencies healthy?

### Diagnosis (5 min)
- Check pods: `kubectl get pods -l app=<name>`
- Check logs: `kubectl logs -l app=<name> --tail=200 --since=10m`
- Check resources: `kubectl top pod -l app=<name>`
- Check dependencies: <dependency dashboards>

### Mitigation
1. First action: <command>
2. If that fails: <command>
3. Last resort: <command>

### Verification
- [ ] Error rate back to baseline
- [ ] Latency back to baseline
- [ ] All health checks passing

### Escalation
- 30 min unresolved → @<lead>
- 60 min unresolved → @<manager>
```
