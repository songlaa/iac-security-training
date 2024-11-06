---
title: "Azure Keyvault"
weight: 57
sectionnumber: 5.7
---


In a previous module in the CAS you created a (free Azure Account)[[text](https://azure.microsoft.com/en-us/pricing/purchase-options/azure-account)].
If you have still some free credit on it we can use that to create an Azure Key Vault. The pricing for it you find [here](https://azure.microsoft.com/en-us/pricing/details/key-vault/):

## Task {{% param sectionnumber %}}.1: Create files to provision a Key Vault

Create a new directory

```bash
mkdir $LAB_ROOT/advanced/keyvault
cd $LAB_ROOT/advanced/keyvault
```

Create a file named `providers.tf` and insert the following code:

```terraform
terraform {
  required_version = ">=1.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>3.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~>3.0"
    }
  }
}
provider "azurerm" {
  features {}
}
```

Create a file named `main.tf` and insert the following code:

```terraform
resource "random_pet" "rg_name" {
  prefix = var.resource_group_name_prefix
}

resource "azurerm_resource_group" "rg" {
  name     = random_pet.rg_name.id
  location = var.resource_group_location
}

data "azurerm_client_config" "current" {}

resource "random_string" "azurerm_key_vault_name" {
  length  = 13
  lower   = true
  numeric = false
  special = false
  upper   = false
}

locals {
  current_user_id = coalesce(var.msi_id, data.azurerm_client_config.current.object_id)
}

resource "azurerm_key_vault" "vault" {
  name                       = coalesce(var.vault_name, "vault-${random_string.azurerm_key_vault_name.result}")
  location                   = azurerm_resource_group.rg.location
  resource_group_name        = azurerm_resource_group.rg.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = var.sku_name
  soft_delete_retention_days = 7

  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = local.current_user_id

    key_permissions    = var.key_permissions
    secret_permissions = var.secret_permissions
  }
}

resource "random_string" "azurerm_key_vault_key_name" {
  length  = 13
  lower   = true
  numeric = false
  special = false
  upper   = false
}

resource "azurerm_key_vault_key" "key" {
  name = coalesce(var.key_name, "key-${random_string.azurerm_key_vault_key_name.result}")

  key_vault_id = azurerm_key_vault.vault.id
  key_type     = var.key_type
  key_size     = var.key_size
  key_opts     = var.key_ops

  rotation_policy {
    automatic {
      time_before_expiry = "P30D"
    }

    expire_after         = "P90D"
    notify_before_expiry = "P29D"
  }
}
```

Create a file named `variables.tf` and insert the following code:

```terraform
variable "resource_group_location" {
  type        = string
  description = "Location for all resources."
  default     = "eastus"
}

variable "resource_group_name_prefix" {
  type        = string
  description = "Prefix of the resource group name that's combined with a random ID so name is unique in your Azure subscription."
  default     = "rg"
}

variable "vault_name" {
  type        = string
  description = "The name of the key vault to be created. The value will be randomly generated if blank."
  default     = ""
}

variable "key_name" {
  type        = string
  description = "The name of the key to be created. The value will be randomly generated if blank."
  default     = ""
}

variable "sku_name" {
  type        = string
  description = "The SKU of the vault to be created."
  default     = "standard"
  validation {
    condition     = contains(["standard", "premium"], var.sku_name)
    error_message = "The sku_name must be one of the following: standard, premium."
  }
}

variable "key_permissions" {
  type        = list(string)
  description = "List of key permissions."
  default     = ["List", "Create", "Delete", "Get", "Purge", "Recover", "Update", "GetRotationPolicy", "SetRotationPolicy"]
}

variable "secret_permissions" {
  type        = list(string)
  description = "List of secret permissions."
  default     = ["Set"]
}

variable "key_type" {
  description = "The JsonWebKeyType of the key to be created."
  default     = "RSA"
  type        = string
  validation {
    condition     = contains(["EC", "EC-HSM", "RSA", "RSA-HSM"], var.key_type)
    error_message = "The key_type must be one of the following: EC, EC-HSM, RSA, RSA-HSM."
  }
}

variable "key_ops" {
  type        = list(string)
  description = "The permitted JSON web key operations of the key to be created."
  default     = ["decrypt", "encrypt", "sign", "unwrapKey", "verify", "wrapKey"]
}

variable "key_size" {
  type        = number
  description = "The size in bits of the key to be created."
  default     = 2048
}

variable "msi_id" {
  type        = string
  description = "The Managed Service Identity ID. If this value isn't null (the default), 'data.azurerm_client_config.current.object_id' will be set to this value."
  default     = null
}
```

Finally, write the `outputs.tf` file:

```terraform
output "resource_group_name" {
  value = azurerm_resource_group.rg.name
}

output "azurerm_key_vault_name" {
  value = azurerm_key_vault.vault.name
}

output "azurerm_key_vault_id" {
  value = azurerm_key_vault.vault.id
}
```

Please read the files carefully and try to understand them.

## Task {{% param sectionnumber %}}.1: Terraform plan and apply

Run terraform init to initialize the Terraform deployment. This command downloads the Azure provider required to manage your Azure resources.

```bash
terraform init -upgrade
```

The -upgrade parameter upgrades the necessary provider plugins to the newest version that complies with the configuration's version constraints.
Create a Terraform execution plan

Run terraform plan to create an execution plan.

```bash
terraform plan -out main.tfplan
```

You will see that you need to login with your Azure Account, please do so and be carefull to use the right subscription

Run terraform apply to apply the execution plan to your cloud infrastructure.

```bash
terraform apply main.tfplan
```

Then verify the results:

```bash
azurerm_key_vault_name=$(terraform output -raw azurerm_key_vault_name)
```

and

```bash
az keyvault key list --vault-name $azurerm_key_vault_name
```

## Task {{% param sectionnumber %}}.1: Cleanup

```bash
terraform plan -destroy -out main.destroy.tfplan
```

We saw how to create a real resource in the public cloud. This Keyvaul could be used to store secrets for a Deployment or with a Pipeline.
