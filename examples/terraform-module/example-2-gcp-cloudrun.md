# Example 2: GCP Cloud Run Service

## Input

User says:
> "I need Terraform for a Cloud Run service on GCP. It's a Go API, needs to connect to Cloud SQL, secrets from Secret Manager, auto-scale 0-10 instances."

## Claude Output (with terraform-module skill)

Generates a Terraform root module for a Cloud Run service with:
- Cloud Run v2 service with internal LB ingress
- Cloud SQL Postgres instance (private IP, SSL required)
- Secret Manager secrets (DB password) referenced as env vars
- Workload Identity for Cloud Run → Cloud SQL auth (no passwords in code)
- Auto-scaling config (min 0 for scale-to-zero in dev, min 1 for prod)
- VPC connector for private Cloud SQL access
