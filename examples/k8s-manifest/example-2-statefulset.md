# Example 2: StatefulSet for Database

## Input

User says:
> "I need to run PostgreSQL on K8s for staging. 2 replicas, 50GB storage each, needs to survive node restarts."

## Claude Output (with k8s-manifest skill)

Generates: StatefulSet with headless Service, volumeClaimTemplates for persistent storage, ConfigMap for PG config, NetworkPolicy limiting access to the backend namespace only. Includes anti-affinity to ensure replicas land on different nodes.
