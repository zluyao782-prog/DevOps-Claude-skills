# AWS Terraform Reference

## Provider Configuration

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}

provider "aws" {
  region = var.region

  # Assume role in target account (multi-account setup)
  assume_role {
    role_arn     = "arn:aws:iam::${var.account_id}:role/TerraformExecution"
    session_name = "terraform-${var.environment}"
  }

  default_tags {
    tags = var.default_tags
  }
}
```

## Common Resource Patterns

### VPC

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true
}

resource "aws_subnet" "private" {
  count             = var.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.cidr_block, 4, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

### RDS

```hcl
resource "aws_db_instance" "main" {
  engine                = "postgres"
  engine_version        = "16.2"
  instance_class        = var.instance_class
  allocated_storage     = var.allocated_storage
  storage_encrypted     = true
  db_name               = var.db_name
  username              = var.db_username
  password              = random_password.db.result
  parameter_group_name  = aws_db_parameter_group.main.name
  db_subnet_group_name  = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  backup_retention_period = 30
  deletion_protection   = var.environment == "prod"
  skip_final_snapshot   = false
  apply_immediately     = var.environment != "prod"
}
```

### EKS IAM Roles

```hcl
# IRSA — IAM Roles for Service Accounts (OIDC)
resource "aws_iam_role" "service" {
  name = "${var.environment}-${var.service_name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = var.eks_oidc_provider_arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${var.eks_oidc_provider_url}:sub" = "system:serviceaccount:${var.namespace}:${var.service_account_name}"
        }
      }
    }]
  })
}
```

## Key AWS Services & Resource Names

| Service | Resource prefix | Key attributes |
|---------|----------------|----------------|
| VPC | `aws_vpc`, `aws_subnet` | CIDR, AZ count, NAT gateway |
| EKS | `aws_eks_cluster`, `aws_eks_node_group` | KMS encryption, private endpoint |
| RDS | `aws_db_instance` | `storage_encrypted`, `deletion_protection` |
| S3 | `aws_s3_bucket` | `aws_s3_bucket_public_access_block`, SSE |
| IAM | `aws_iam_role`, `aws_iam_policy` | Least privilege, conditions |
| Lambda | `aws_lambda_function` | VPC attach, env vars encrypted |
| ECS | `aws_ecs_service` | Fargate, capacity provider |
| ALB/NLB | `aws_lb` | HTTPS only, access logs to S3 |
| DynamoDB | `aws_dynamodb_table` | PITR, KMS, on-demand billing |
| ElastiCache | `aws_elasticache_cluster` | encryption at rest + transit |
