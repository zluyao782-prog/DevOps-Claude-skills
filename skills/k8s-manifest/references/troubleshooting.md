# K8s Manifest Troubleshooting

## Diagnostic Commands

```bash
# Pod status and events
kubectl describe pod <pod-name> -n <namespace>

# Recent events sorted by time
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Pod logs (current and previous)
kubectl logs <pod-name> -n <namespace> --tail=100
kubectl logs <pod-name> -n <namespace> --previous

# Resource usage
kubectl top pod -n <namespace>
kubectl top node

# Check endpoints (verify Service → Pod mapping)
kubectl get endpoints -n <namespace>

# HPA status
kubectl describe hpa <hpa-name> -n <namespace>

# Certificate status
kubectl describe certificate -n <namespace>
kubectl describe certificaterequest -n <namespace>
```

## Common Patterns

### Pod won't start
1. `kubectl describe pod` → Check Events section
2. ImagePullBackOff? → Check image name, registry auth, imagePullSecrets
3. CrashLoopBackOff? → Check logs, liveness probe timing
4. Pending? → Check resource requests, node selectors, PV availability

### Service unreachable
1. `kubectl get endpoints <svc>` → Empty? Labels mismatch
2. `kubectl get networkpolicy` → Blocking traffic?
3. `kubectl describe svc <name>` → Port mapping correct?

### HPA not scaling
1. `kubectl get hpa` → Current utilization showing?
2. `kubectl top pod` → Returns data? (metrics-server needed)
3. `kubectl describe hpa` → Events show why not scaling
