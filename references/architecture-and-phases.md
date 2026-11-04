# Architecture: tiers and phases

## 1. Code tiers

| Tier | Folder | Responsibility | Has backend/providers | Output |
|---|---|---|---|---|
| Resources | `Terraform/Resources/<provider>_<type>/` | One parameterized provider resource | No (except `required_providers` for a non-default provider, e.g. `azapi`) | A single object `<type>_output` |
| Middle Modules | `Terraform/Modules/<Phase>_IaC_<Domain>/` | Compose the Resources of one domain within a phase | No | One object per resource + flat convenience outputs |
| Envs | `Terraform/Envs/<Env>/<Phase>/` | Root module: naming, flags, lookups, wiring, state | Yes | Aggregates with `try()` |

Relative paths: Envs → `../../../Modules|Resources/...`; Modules → `../../Resources/...`
(adjust the depth if modules are per environment: `Modules/<Env>/<...>` → `../../../`).

## 2. Provisioning phases

| # | Phase | Granularity (1 state per…) | Creates (Azure examples) | Reads from earlier phases |
|---|---|---|---|---|
| 1 | **Region** | region + instance (`count_r`) | Regional RG, VNet/subnets, SQL Server + Elastic Pool, Log Analytics, regional Key Vault, ACR/AKS, shared storage | — |
| 2 | **Plan** | region + size + instance (`plansize`, `count_p`) | Plan RG, App Service Plan (Windows/Linux), shared compute capacity | Region (network, observability) |
| 3 | **Client** | client + instance (`count_c`) | Client RG, App Insights, Key Vault, Storage, SQL DB in the pool, Web/Function/Static Apps, dedicated capacities | Region (SQL Server, Elastic Pool), Plan (App Service Plan) |

You can add phases (e.g. `Network` before Region, or `PostIaC` after Client) as long as they
follow the rule: **a phase only reads earlier phases**.

### Typical middle modules per phase

```
Region_IaC_Common             RG, LAW, App Insights, Key Vault, Storage
Region_IaC_Application        Regional App Service, shared APIs
Region_IaC_DataOrchestration  SQL Server, Elastic Pool, PostgreSQL, Redis
Region_IaC_MLModel            Databricks, ML workspaces
Region_IaC_Monitoring         Alerts, diagnostics
Plan_IaC_Application          App Service Plan
Plan_IaC_Data                 Data resources tied to the plan
Plan_IaC_Monitoring           Plan diagnostics
Client_IaC_Common             LAW, App Insights, Key Vault, Storage, SQL DB
Client_IaC_Application        Web App, Function App, Static Web App
Client_IaC_Fabric             Dedicated capacity (e.g. Microsoft Fabric via azapi)
Client_IaC_Transporter        Optional integration API
```

## 3. Reference dependency tree (Client phase)

Notation: `──►` invokes (`source`), `··►` consumes a value, `[dep]` = `depends_on`.

```
[Phase 1 Region]  rg-<org>-<rc>-<count_r>
                   ├─ sql-<org>-<rc>-<count_r>
                   └─ sqlep-sql-<org>-<rc>-<count_r>
[Phase 2 Plan]    rg-<org>-<rc>-<size>-<count_p>
                   └─ plan-<org>-<rc>-<size>-<count_p>
[Control]         <controlStorageAccount>/<tenant>-<count_c>/terraform.tfstate

Envs/<Env>/Client
├─ data.azurerm_client_config / data.azurerm_subscription
├─ data.azurerm_mssql_server        (Region) ─┐
├─ data.azurerm_mssql_elasticpool   (Region) ─┤
├─ data.azurerm_service_plan        (Plan)   ─┤
├─ module.resource_group ──► Resources/azure_resource_group
│
├─[dep]─ module.common[0] ──► Client_IaC_Common
│         ├─► azure_log_analytics
│         ├─► azure_app_insights   ··► log_analytics.id (workspaceId)
│         ├─► azure_key_vault      ··► client_config.object_id, tenant_id
│         ├─► azure_storage_account
│         └─► azure_sql_db         ··► sql_server.id, elasticpool.id
│
├─[dep common]─ module.application[0] ──► Client_IaC_Application
│         ··► service_plan.id, common[0].storage_account_{name,key}
│         ├─► azure_web_app_windows
│         ├─► azure_function_app_windows
│         └─► azure_static_web_app
│
├─[dep]─ module.dedicated_capacity[0] ──► Client_IaC_Fabric
│         ··► resource_group.id (parent_id)
│         └─► azure_fabric (azapi_resource)
│
└─[dep]─ module.integration_api[0] ──► Client_IaC_Transporter
          ··► service_plan.id
          └─► azure_web_app_windows
```

**Effective order inside the phase**: `data.*` (these fail at *plan* time if the earlier phase
does not exist) → resource group → independent modules in parallel → modules that depend on others.

## 4. How phases connect

| Mechanism | When to use it | Pros | Cons |
|---|---|---|---|
| `data` source by deterministic name | stable, shared naming | no coupling between states; simple | if naming changes in one phase, the next one breaks at *plan* |
| `terraform_remote_state` | you need IDs/attributes that cannot be rebuilt | explicit contract | needs read access to the earlier state |
| Outputs file (`values.json`) next to the state | non-Terraform consumers (scripts, post-IaC) | decoupled | the schema must be maintained |

Recommendation: `data` by name for Region/Plan resources, and `values.json` for post-IaC.
Move to `remote_state` when naming is no longer enough.

## 5. Providers

| Provider | Indicative version | Use |
|---|---|---|
| `hashicorp/azurerm` | `~> 4.x` | ARM resources; v4 requires `subscription_id` (`ARM_SUBSCRIPTION_ID`) |
| `Azure/azapi` | `~> 2.0` | types not yet supported by azurerm |
| others (`microsoft/fabric`…) | pin the version | declare them only if some resource uses them |

Use a bounded `required_version` (`>= 1.8, < 2.0`) and the same Terraform version in CI and locally.
