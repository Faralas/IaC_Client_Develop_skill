# File templates

## Resource — `terraform/resources/azure_key_vault/`

```hcl
# main.tf
resource "azurerm_key_vault" "this" {
  name                       = var.name
  location                   = var.location
  resource_group_name        = var.resource_group_name
  tenant_id                  = var.tenant_id
  sku_name                   = var.sku_name
  soft_delete_retention_days = var.retention_days
  purge_protection_enabled   = var.purge_protection
  tags                       = var.tags
  # Enable RBAC authorization with the argument your azurerm version supports
  # (enable_rbac_authorization, renamed rbac_authorization_enabled in later 4.x releases).
}
```

```hcl
# variables.tf
variable "name"                { type = string }
variable "location"            { type = string }
variable "resource_group_name" { type = string }
variable "tenant_id"           { type = string }

variable "sku_name" {
  type    = string
  default = "standard"
  validation {
    condition     = contains(["standard", "premium"], var.sku_name)
    error_message = "sku_name must be standard or premium."
  }
}

variable "retention_days" {
  type    = number
  default = 7
}

variable "purge_protection" {
  type    = bool
  default = true
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# outputs.tf — a single object
output "key_vault_output" {
  description = "Key Vault attributes."
  value = {
    id        = azurerm_key_vault.this.id
    name      = azurerm_key_vault.this.name
    vault_uri = azurerm_key_vault.this.vault_uri
  }
}
```

## Middle module — `terraform/modules/security/`

```hcl
# main.tf
module "key_vault" {
  source              = "../../resources/azure_key_vault"
  name                = var.key_vault_name
  location            = var.location
  resource_group_name = var.resource_group_name
  tenant_id           = var.tenant_id
  sku_name            = var.key_vault_sku_name
  tags                = var.tags
}

module "identity" {
  source              = "../../resources/azure_user_assigned_identity"
  name                = var.identity_name
  location            = var.location
  resource_group_name = var.resource_group_name
  tags                = var.tags
}

resource "azurerm_role_assignment" "identity_kv_secrets" {
  scope                = module.key_vault.key_vault_output.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = module.identity.user_assigned_identity_output.principal_id
}
```

```hcl
# outputs.tf — object + flat outputs, always read from the Resource's object
output "key_vault"    { value = module.key_vault.key_vault_output }
output "key_vault_id" { value = module.key_vault.key_vault_output.id }
output "identity_id"  { value = module.identity.user_assigned_identity_output.id }
```

```hcl
# versions.tf
terraform {
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = ">= 4.0" }
  }
}
```

## Root — `terraform/env/`

```hcl
# backend.tf — partial configuration; values come from -backend-config in CI
terraform {
  backend "azurerm" {
    key = "dev.terraform.tfstate"
  }
}
```

```hcl
# versions.tf
terraform {
  required_version = ">= 1.8, < 2.0"
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 4.39" }
  }
}
```

```hcl
# providers.tf
provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}
```

```hcl
# locals.tf
locals {
  _base  = format("%s-%s-%s", lower(var.workload), lower(var.environment), lower(var.region_code))
  _alnum = replace(local._base, "/[^a-z0-9]/", "")

  names = {
    rg      = "rg-${local._base}"
    law     = "log-${local._base}"
    kv      = substr("kv-${local._base}-${var.instance}", 0, 24)
    st      = substr("st${local._alnum}${var.instance}", 0, 24)
    plan    = "asp-${local._base}"
    web     = "app-${local._base}-${var.instance}"
    appi    = "appi-${local._base}"
    id      = "id-${local._base}"
  }

  tags = merge(var.tags, {
    environment = var.environment
    workload    = var.workload
    managed_by  = "terraform"
  })
}
```

```hcl
# main.tf
data "azurerm_client_config" "current" {}

module "foundation" {
  source              = "../modules/foundation"
  resource_group_name = local.names.rg
  log_analytics_name  = local.names.law
  location            = var.location
  tags                = local.tags
}

module "security" {
  source              = "../modules/security"
  key_vault_name      = local.names.kv
  identity_name       = local.names.id
  location            = var.location
  resource_group_name = module.foundation.resource_group_name   # reference = order
  tenant_id           = data.azurerm_client_config.current.tenant_id
  key_vault_sku_name  = var.key_vault_sku_name
  tags                = local.tags
}

module "application" {
  count               = var.enable_application ? 1 : 0
  source              = "../modules/application"
  web_app_name        = local.names.web
  service_plan_name   = local.names.plan
  location            = var.location
  resource_group_name = module.foundation.resource_group_name
  identity_id         = module.security.identity_id
  tags                = local.tags
}
```

```hcl
# variables.tf (excerpt)
variable "subscription_id" { type = string }
variable "workload"        { type = string }
variable "environment"     { type = string }   # e.g. "dev"
variable "region_code"     { type = string }   # e.g. "eus"
variable "location"        { type = string }   # e.g. "eastus"

variable "instance" {
  type    = string
  default = "01"
}

variable "enable_application" {
  type    = bool
  default = true
}

variable "key_vault_sku_name" {
  type    = string
  default = "standard"
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# terraform.tfvars (versioned, no secrets)
workload           = "shop"
environment        = "dev"
region_code        = "eus"
location           = "eastus"
enable_application = true
key_vault_sku_name = "standard"
tags               = { cost_center = "it-dev" }
```

```hcl
# outputs.tf
output "resource_group_name" { value = module.foundation.resource_group_name }
output "key_vault_uri"       { value = module.security.key_vault.vault_uri }
output "web_app_hostname" {
  value = try(module.application[0].web_app_default_hostname, null)
}
```

```hcl
# moved.tf — rename without destroy/recreate
moved {
  from = module.app
  to   = module.application[0]
}
```
