# Variable handling

## Full chain

```
run parameter ─► task env: ─► ${VAR} bash ─► -var "x=${VAR}" ──────┐
variable group ───────────► $(x) ADO macro ─► -var 'x=$(x)' ───────┤
computed in bash ─────────────────────────────► -var "x=${X}" ─────┼─► variable "x" (Envs)
variables.tf default ──────────────────────────────────────────────┘        │
                                                              local.* (naming, wiring)
                                                                            │
                                               middle module input ─► resource input ─► attribute
```

## Classification

| Kind | Where it lives | Examples | Rule |
|---|---|---|---|
| Instance identity | run parameters | `region`, `regioncode`, `tenant_code`, `count_c`, `count_r`, `count_p`, `plansize`, `env` | Required, no default in `variables.tf` |
| Per-environment configuration | the environment's variable group | SKUs, retention, replicas, flags, admins | Sensible default in `variables.tf`; the group overrides it |
| Repo constants | `variables.tf` default | `naming_org`, resource abbreviations | Not passed via `-var` |
| Derived | pipeline bash or `locals` | state container name, business flags, names | A single source of truth |
| Data from earlier phases | `data` / `remote_state` | SQL Server and App Service Plan IDs | Never as a hand-copied `-var` |
| Secrets | Key Vault linked to the variable group / identity | passwords, keys | Never in tfvars or in `-var` visible in logs; mark `sensitive = true` |

## Example mapping (Client phase)

| Source | `-var` | TF variable | Local | Ends up in |
|---|---|---|---|---|
| P `region` | `location` | `location` | — | `location` of every resource |
| P `tenant_code` | `tenant_abbr` | `tenant_abbr` | `_tenant` | every name in the phase |
| P `count_c` | `instance_count` | `instance_count` | `_inst` | names + state container |
| P `regioncode` | `region_code` | `region_code` | `_rc` | names + lookups |
| P `count_r` | `region_count` | `region_count` | — | Region lookups |
| P `plansize`, `count_p` | `plan_size`, `plan_count` | same | — | Plan lookup |
| P `env` | `environment` | `environment` | `_env` | tags and names that include it |
| G `enable_common` | `enable_common` | `enable_common` (bool) | — | module `count` |
| B `!is_internal_tenant` | `enable_integration_api` | same | — | module `count` |
| G `azure_keyVault_skuName` | same | same | — | Common module → `azurerm_key_vault.sku_name` |
| G `capacity_admins` | same | `list(string)` | — | `body.properties.administration.members` |
| D `naming_org` | — | `naming_org` | `_org` | every name |
| `data.azurerm_client_config` | — | — | — | `object_id` for access policies |

P = run parameter, G = variable group, B = computed in bash, D = `variables.tf` default.

## Typing and substitution pitfalls (Azure DevOps)

- `$(x)` is a macro that is replaced **before** bash runs. If the variable does not exist in
  the group, the literal `$(x)` remains:
  - in **single quotes** (`-var 'x=$(x)'`) Terraform receives `$(x)`: it fails type conversion
    for bool/number/list and, for a string, **creates resources with that literal value**;
  - in double quotes, bash runs it as a subshell (`x: command not found`).
  Recommendation: single quotes, plus a pre-check that fails if any `$(` remains.
- Lists and maps go into the group **in HCL syntax**: `["id1","id2"]`, `{env="qa"}`.
- Booleans from `parameters` arrive as `True`/`False`: normalize them with `tr '[:upper:]' '[:lower:]'`.
- A more robust alternative to 30 `-var` flags: generate a `ci.auto.tfvars.json` with `jq`
  from the environment variables, or use `TF_VAR_<name>`.

## Flags

```hcl
variable "enable_common" {
  type    = bool
  default = true
}

variable "enable_application" {   # requires enable_common
  type    = bool
  default = true
}
```

- Document which flags depend on others and validate it (`validation` or `check`).
- Compute business-derived flags in one place only (pipeline **or** locals, not both).
- Switching a flag from `true` to `false` makes the plan **destroy** the module: review the plan.

## Defaults and tfvars

- `variables.tf` holds defaults that are valid in **every** environment; anything
  environment-specific goes in the group.
- Do not leave defaults from another environment (e.g. tags `env = "sandbox"` in PROD).
- `terraform.tfvars` is for local use only and ignored by git. Keep a versioned
  `terraform.tfvars.example` with dummy values.
- Do not declare variables nobody consumes, and do not pass `-var` for variables that do not
  exist (Terraform fails with `undeclared variable`).
