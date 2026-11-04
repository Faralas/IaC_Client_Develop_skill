# File templates

## Resources/<provider>_<type>/

```hcl
# keyVault.tf
resource "azurerm_key_vault" "this" {
  name                       = var.azure_keyVault_name
  location                   = var.azure_keyVault_location
  resource_group_name        = var.azure_keyVault_resourceGroupName
  tenant_id                  = var.azure_keyVault_tenantId
  sku_name                   = var.azure_keyVault_skuName
  soft_delete_retention_days = var.azure_keyVault_retentionDays
  purge_protection_enabled   = var.azure_keyVault_purgeProtection
}
```

```hcl
# output.tf — a single object
output "azure_key_vault_output" {
  description = "Key Vault object"
  value = {
    id        = azurerm_key_vault.this.id
    name      = azurerm_key_vault.this.name
    vault_uri = azurerm_key_vault.this.vault_uri
  }
}
```

## Modules/<Phase>_IaC_<Domain>/

```hcl
# main.tf
module "azure_log_analytics" {
  source                                        = "../../Resources/azure_log_analytics"
  azure_logAnalyticsWorkspace_name              = var.azure_logAnalyticsWorkspace_name
  azure_logAnalyticsWorkspace_location          = var.location
  azure_logAnalyticsWorkspace_resourceGroupName = var.resource_group_name
  azure_logAnalyticsWorkspace_sku               = var.azure_logAnalyticsWorkspace_sku
  azure_logAnalyticsWorkspace_retentionInDays   = var.azure_logAnalyticsWorkspace_retentionInDays
}

module "azure_app_insights" {
  source                                      = "../../Resources/azure_app_insights"
  azure_applicationInsights_name              = var.azure_applicationInsights_name
  azure_applicationInsights_location          = var.location
  azure_applicationInsights_resourceGroupName = var.resource_group_name
  azure_applicationInsights_applicationType   = var.azure_applicationInsights_applicationType
  azure_applicationInsights_workspaceId       = module.azure_log_analytics.azure_log_analytics_output.id
}
```

```hcl
# output.tf — object + flat outputs, always read from the resource's object
output "log_analytics"    { value = module.azure_log_analytics.azure_log_analytics_output }
output "log_analytics_id" { value = module.azure_log_analytics.azure_log_analytics_output.id }
output "app_insights_connection_string" {
  value     = module.azure_app_insights.azure_app_insights_output.connection_string
  sensitive = true
}
```

## Envs/<Env>/<Phase>/

```hcl
# backend.tf — partial configuration; values arrive via -backend-config
terraform {
  backend "azurerm" {}
}
```

```hcl
# providers.tf
terraform {
  required_version = ">= 1.8, < 2.0"
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 4.39" }
    azapi   = { source = "Azure/azapi", version = "~> 2.0" }
  }
}
provider "azurerm" { features {} }
provider "azapi" {}
```

```hcl
# local.tf
locals {
  _org    = lower(var.naming_org)
  _rc     = lower(var.region_code)
  _tenant = lower(var.tenant_abbr)
  _inst   = lower(var.instance_count)

  _clean = "/[^a-z0-9-]/"
  _alnum = "/[^a-z0-9]/"

  name_rg  = replace(format("rg-%s-%s-%s-%s", local._org, local._rc, local._tenant, local._inst), local._clean, "")
  name_kv  = substr(replace(format("kv-%s-%s-%s", local._org, local._tenant, local._inst), local._clean, ""), 0, 24)
  name_stg = substr(replace(format("st%s%s%s%s", local._org, local._rc, local._tenant, local._inst), local._alnum, ""), 0, 24)

  # names from earlier phases (lookups)
  name_region_rg  = format("rg-%s-%s-%s", local._org, local._rc, lower(var.region_count))
  name_region_sql = format("sql-%s-%s-%s", local._org, local._rc, lower(var.region_count))
  name_plan_rg    = format("rg-%s-%s-%s-%s", local._org, local._rc, lower(var.plan_size), lower(var.plan_count))
  name_plan_plan  = format("plan-%s-%s-%s-%s", local._org, local._rc, lower(var.plan_size), lower(var.plan_count))
}
```

```hcl
# main.tf (Client phase)
module "resource_group" {
  source                       = "../../../Resources/azure_resource_group"
  azure_resourceGroup_name     = local.name_rg
  azure_resourceGroup_location = var.location
}

data "azurerm_service_plan" "plan" {
  name                = local.name_plan_plan
  resource_group_name = local.name_plan_rg
}

module "common" {
  count      = var.enable_common ? 1 : 0
  source     = "../../../Modules/Client_IaC_Common"
  # …
  depends_on = [module.resource_group]
}

module "application" {
  count  = var.enable_application ? 1 : 0
  source = "../../../Modules/Client_IaC_Application"
  azure_functionWindows_storageAccountName = module.common[0].storage_account_name
  # …
  depends_on = [module.common]
}

# Dependency between flags. Module blocks do not support lifecycle/precondition;
# a check block warns at plan time, and a validation (TF >= 1.9 can reference
# other variables) turns it into an error.
variable "enable_application" {
  type    = bool
  default = true
  validation {
    condition     = !var.enable_application || var.enable_common
    error_message = "enable_application requires enable_common = true."
  }
}
```

```hcl
# output.tf
output "common" {
  value = length(module.common) > 0 ? {
    key_vault_id = try(module.common[0].key_vault_id, null)
    storage_id   = try(module.common[0].storage_account_id, null)
  } : null
}
```
