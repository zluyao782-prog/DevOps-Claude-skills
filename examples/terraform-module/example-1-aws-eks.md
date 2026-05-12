# Example 1: AWS EKS Platform

## Input

User says:
> "Provision an EKS cluster on AWS for production. Private API endpoint, KMS-encrypted secrets, managed node group with auto-scaling, IRSA for workload identity. VPC with 3 AZs, public and private subnets."

## Claude Output (with terraform-module skill)

Generates a complete Terraform root module calling child modules for VPC and EKS:

```
terraform/
├── modules/
│   ├── vpc/           # Reusable VPC module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── eks/           # Reusable EKS module
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    └── prod/
        ├── main.tf    # Calls modules with prod-specific values
        ├── variables.tf
        ├── outputs.tf
        └── terraform.tfvars
```

Key decisions:
- S3 + DynamoDB remote backend with state encryption
- KMS key for EKS secrets, CloudWatch logs, EBS volumes
- OIDC provider for IRSA (no long-lived IAM user keys)
- Private API endpoint (no public exposure)
- Node groups in private subnets
- Cluster autoscaler tags on node groups
