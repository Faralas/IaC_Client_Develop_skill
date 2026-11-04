# Architecture

## 1. Tiers

| Tier | Folder | Responsibility | Backend / providers | Outputs |
|---|---|---|---|---|
| Resources | `terraform/resources/<provider>_<type>/` | One parameterized provider resource (plus inseparable children, e.g. a diagnostic setting) | `required_providers` only when a non-default provider is used (e.g. `azapi`) | One object `<type>_output` |
| Middle Modules | `terraform/modules/<domain>/` | Compose the Resources of one domain and wire them internally | `required_providers` only | One object per resource + flat convenience outputs |
| Root | `terraform/env/` | Backend, providers, naming, tags, flags, wiring between modules | Yes | What consumers of the environment need |

Relative paths: root → `../modules/<domain>`; module → `../../resources/<provider>_<type>`.

## 2. Typical middle modules

| Module | Composes (Azure examples) | Typical inputs from other modules |
|---|---|---|
| `foundation` | Resource group(s), tags, Log Analytics | — |
| `network` | VNet, subnets, NSGs, private DNS zones | `foundation` RG |
| `security` | Key Vault, managed identities, role assignments | `foundation` RG, `network` subnets (private endpoints) |
| `data` | Storage account, SQL Server/DB, PostgreSQL, Redis | RG, subnets, Key Vault (secrets), LAW (diagnostics) |
| `compute` | App Service Plan, AKS, VM scale sets, Container Apps environment | RG, subnets, LAW |
| `application` | Web Apps, Function Apps, Static Web Apps, API Management | plan / environment, storage, Key Vault, App Insights |
| `monitoring` | App Insights, alerts, action groups, diagnostic settings | LAW, IDs of monitored resources |

Split modules by **domain and lifecycle**, not by resource type. Resources that are created,
changed and destroyed together belong in the same module.

## 3. Dependency graph (reference)

`──►` calls (`source`), `··►` consumes an output.

```
terraform/env (root)
├─ data.azurerm_client_config.current
├─ module.foundation ──► resources/azure_resource_group, azure_log_analytics
├─ module.network    ──► azure_virtual_network, azure_subnet, azure_network_security_group
│     ··► foundation.resource_group_name
├─ module.security   ──► azure_key_vault, azure_user_assigned_identity
│     ··► foundation.resource_group_name, current.tenant_id, current.object_id
├─ module.data[0]    ──► azure_storage_account, azure_sql_server, azure_sql_db
│     ··► foundation.*, network.subnet_ids["data"], security.key_vault_id
├─ module.compute    ──► azure_service_plan
│     ··► foundation.*
├─ module.application[0] ──► azure_web_app_linux, azure_function_app_linux
│     ··► compute.service_plan_id, data.storage_account_name, security.identity_id
└─ module.monitoring ──► azure_app_insights, azure_monitor_diagnostic_setting
      ··► foundation.log_analytics_id, application.web_app_id
```

## 4. Ordering

Terraform derives the execution order from references. The graph above produces this
logical order, but you never write the order down explicitly:

```
foundation → network → security → data / compute (in parallel) → application → monitoring
```

Rules:
- Express the order with **references to outputs**. That gives correct ordering and also passes the value.
- Use `depends_on` only for dependencies that have no data flow (e.g. a role assignment
  that must exist before an app starts). Put it on the smallest scope possible: `depends_on`
  on a whole module delays every resource in it and makes plans show more "known after apply" values.
- Never rely on running `apply` twice, or on `-target`, to get the order right. If you need
  that, there is a missing reference.
- Data sources that read resources created in the same root must reference them (or
  depend on them); otherwise they run too early and fail.

## 5. When to split the root

Keep a single root while the plan stays readable and fast. Split it into several roots
(each with its own state) under `terraform/env/<stack>/` when:

- parts have very different change frequencies or owners (e.g. network vs apps);
- the plan takes too long or touches too many resources per change;
- the blast radius of a mistake has to be limited.

Connect split roots with `terraform_remote_state` (explicit outputs) or with `data` sources
by name, and document the order in which they must be applied.

## 6. Providers and versions

| Item | Recommendation |
|---|---|
| `required_version` | bounded range, e.g. `>= 1.8, < 2.0`; same version in CI and locally |
| `hashicorp/azurerm` | `~> 4.x`; v4 requires `subscription_id` (or `ARM_SUBSCRIPTION_ID`) |
| `Azure/azapi` | `~> 2.0` for types azurerm does not support yet |
| Lock file | commit `.terraform.lock.hcl` of the root; run `terraform providers lock` for every CI platform |
| Modules | declare `required_providers` (source + minimum version), never `provider` blocks |
