# Azure Terraform Reference

## Provider Configuration

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.100"
    }
  }
}

provider "azurerm" {
  features {}

  # Don't configure subscription_id/client_id here —
  # use az CLI login or service principal env vars
  # ARM_SUBSCRIPTION_ID, ARM_CLIENT_ID, ARM_CLIENT_SECRET, ARM_TENANT_ID
}

# Backend: Azure Storage
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstate${var.environment}"
    container_name       = "tfstate"
    key                  = "env:/${var.environment}/${var.component}.terraform.tfstate"
  }
}
```

## Common Resource Patterns

### AKS Cluster

```hcl
resource "azurerm_kubernetes_cluster" "main" {
  name                = "${var.environment}-${var.cluster_name}"
  location            = var.location
  resource_group_name = var.resource_group_name
  kubernetes_version  = var.kubernetes_version

  azure_active_directory_role_based_access_control {
    managed            = true
    azure_rbac_enabled = true
  }

  key_vault_secrets_provider {
    secret_rotation_enabled  = true
    secret_rotation_interval = "2m"
  }

  network_profile {
    network_plugin = "azure"
    network_policy = "calico"
  }

  default_node_pool {
    name                = "default"
    vm_size             = var.node_vm_size
    enable_auto_scaling = true
    min_count           = var.min_nodes
    max_count           = var.max_nodes
    vnet_subnet_id      = var.aks_subnet_id
  }

  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.aks.id]
  }
}
```

### PostgreSQL Flexible Server

```hcl
resource "azurerm_postgresql_flexible_server" "main" {
  name                = "${var.environment}-${var.db_name}"
  resource_group_name = var.resource_group_name
  location            = var.location
  version             = "16"

  delegated_zone_id = var.private_dns_zone_id
  private_dns_zone_id = var.private_dns_zone_id

  storage_mb   = var.db_storage_mb
  storage_tier = var.environment == "prod" ? "P30" : "P4"

  sku_name = var.db_sku_name

  authentication {
    active_directory_auth_enabled = true
  }
}
```

## Key Azure Services & Resource Names

| Service | Resource prefix | Key attributes |
|---------|----------------|----------------|
| AKS | `azurerm_kubernetes_cluster` | Azure AD RBAC, Key Vault CSI, Calico netpol |
| PostgreSQL | `azurerm_postgresql_flexible_server` | Private DNS, AD auth, HA for prod |
| Storage | `azurerm_storage_account` | HTTPS only, min TLS 1.2, network rules |
| Key Vault | `azurerm_key_vault` | RBAC, purge protection, network ACL |
| VNet | `azurerm_virtual_network` | Service endpoints, private link |
| Container Apps | `azurerm_container_app` | Managed identity, Dapr |
| ACR | `azurerm_container_registry` | Admin disabled, private link |
| Log Analytics | `azurerm_log_analytics_workspace` | OMS agent for AKS monitoring |
