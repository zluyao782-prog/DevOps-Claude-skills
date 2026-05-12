# Example 1: Standard Web Service

## Input

User says:
> "Deploy my payment-service (3 replicas, needs 512Mi RAM, connects to Postgres, exposes /api on port 8080). Scale between 2-10 pods based on CPU. Production ready."

## Claude Output (with k8s-manifest skill)

Generates: Deployment, Service, Ingress, HPA, PDB, NetworkPolicy — all in a single apply-ready YAML with security context, probes, resource management, zone distribution, and zero-trust networking. Approximately 250 lines of manifest with inline security review and apply instructions.
