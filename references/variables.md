# Variable handling

## Chain

```
terraform.tfvars (versioned) ─┐
TF_VAR_* (CI secrets) ────────┼─► root variable ─► locals (names, tags) ─► module input ─► resource input ─► attribute
variables.tf defaults ────────┘                         ▲
data sources (existing infra) ──────────────────────────┘
module outputs ───────────────────────────────────────────► other modules' inputs (wiring in the root)
```

Precedence, lowest to highest: `variables.tf` default → `TF_VAR_*` environment variables →
`terraform.tfvars` → `terraform.tfvars.json` → `*.auto.tfvars` (alphabetical order) →
`-var` / `-var-file` on the command line (in the order given). A `TF_VAR_*` is therefore
overridden by any tfvars entry for the same variable. Keep **one** source per variable so
precedence never decides the outcome.

## Where each kind of value goes

| Kind | Location | Examples | Rule |
|---|---|---|---|
| Environment configuration | `terraform.tfvars` | `location`, SKUs, sizes, flags, CIDRs | Versioned, reviewed in PRs, no secrets |
| Secrets | `TF_VAR_*` from a Key Vault-linked variable group, or read at runtime from Key Vault | DB admin password | Mark `sensitive = true`; better still, generate them (`random_password`) and store them in Key Vault |
| Identity / tenant | `data.azurerm_client_config`, `TF_VAR_subscription_id` | tenant ID, object ID | Never hard-coded |
| Constants | `variables.tf` default or `locals` | abbreviations, standard tags | Not overridden per run |
| Derived values | `locals` | names, merged tags, maps per subnet | A single place |
| Values from other modules | module outputs wired in the root | `module.network.subnet_ids` | Never copied by hand into tfvars |
| Existing external resources | `data` sources | shared DNS zone, hub VNet | Read-only |

## Variable design

- Always set `type` and `description`; use `validation` for allowed values and formats.
- Prefer objects and maps over many scalar variables for structured configuration:

```hcl
variable "subnets" {
  description = "Subnets by key."
  type = map(object({
    address_prefixes  = list(string)
    service_endpoints = optional(list(string), [])
  }))
}
```

- Use `optional()` with defaults in object types instead of `lookup()` chains.
- Use `nullable = false` for variables where `null` makes no sense.
- Mark secrets `sensitive = true` in the variable, in every output that exposes them, and in module variables.
- Do not declare variables nobody uses, and do not set tfvars entries for undeclared variables
  (Terraform warns, and the value is silently ignored).

## Feature flags

```hcl
variable "enable_data" {
  type    = bool
  default = true
}

variable "enable_application" {
  type    = bool
  default = true
  validation {   # TF >= 1.9 can reference other variables
    condition     = !var.enable_application || var.enable_data
    error_message = "enable_application requires enable_data = true."
  }
}
```

- Use `count = var.enable_x ? 1 : 0` and read outputs with `try(module.x[0].y, null)`, or
  `one(module.x[*].y)`.
- Switching a flag from `true` to `false` **destroys** the module's resources: review the plan.

## Tags

Build them once in `locals` (`merge(var.tags, { environment, workload, managed_by })`) and
pass them to every module. The azurerm provider has no provider-level default tags, so every
Resource must expose a `tags` variable and every module must forward it. If Azure Policy
also adds or modifies tags, exclude those keys in `lifecycle { ignore_changes = [tags["<key>"]] }`
to avoid permanent drift.

## Pitfalls

- A tfvars value for a `list`/`map` variable must use HCL syntax (`["a"]`, `{ k = "v" }`).
- Strings that look like numbers (`"01"`) stay strings only if the variable type is `string`.
- `sensitive` values cannot be used in `for_each` keys; restructure so the keys are non-sensitive.
- Values unknown until apply cannot drive `count`/`for_each`: base them on variables or locals.
