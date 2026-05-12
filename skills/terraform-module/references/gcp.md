# GCP Terraform Reference

## Provider Configuration

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.20"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# Backend: GCS
terraform {
  backend "gcs" {
    bucket = "terraform-state-${var.project_id}"
    prefix = "env:/${var.environment}/${var.component}"
  }
}
```

## Common Resource Patterns

### GKE Cluster

```hcl
resource "google_container_cluster" "main" {
  name     = "${var.environment}-${var.cluster_name}"
  location = var.region

  # Private cluster (no public endpoint)
  private_cluster_config {
    enable_private_endpoint = false
    enable_private_nodes    = true
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  # Workload Identity (OIDC for K8s → GCP auth)
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Secret encryption at application layer
  database_encryption {
    state    = "ENCRYPTED"
    key_name = var.kms_key_name
  }

  node_config {
    service_account = google_service_account.gke_nodes.email
    machine_type    = var.node_machine_type
    disk_size_gb    = 100
    disk_type       = "pd-ssd"
  }
}
```

### Cloud SQL

```hcl
resource "google_sql_database_instance" "main" {
  name             = "${var.environment}-${var.db_name}"
  database_version = "POSTGRES_16"
  region           = var.region

  settings {
    tier              = var.db_tier
    availability_type = var.environment == "prod" ? "REGIONAL" : "ZONAL"
    disk_size         = var.db_disk_size
    disk_type         = "PD_SSD"

    ip_configuration {
      ipv4_enabled    = false
      private_network = var.vpc_network_id
      require_ssl     = true
    }

    backup_configuration {
      enabled                        = true
      start_time                     = "03:00"
      point_in_time_recovery_enabled = true
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 30
      }
    }
  }

  deletion_protection = var.environment == "prod"
}
```

## Key GCP Services & Resource Names

| Service | Resource prefix | Key attributes |
|---------|----------------|----------------|
| GKE | `google_container_cluster` | Private cluster, Workload Identity, KMS encryption |
| Cloud SQL | `google_sql_database_instance` | Private IP, SSL required, PITR |
| Cloud Run | `google_cloud_run_v2_service` | Ingress restriction, Secret Manager refs |
| GCS | `google_storage_bucket` | Uniform bucket-level access, KMS |
| VPC | `google_compute_network` | Subnet flow logs, private Google access |
| IAM | `google_project_iam_member` | Workload Identity binding, conditions |
| Secret Manager | `google_secret_manager_secret` | Automatic replication, IAM per secret |
