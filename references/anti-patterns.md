# Common mistakes

🔴 breaks or deploys to the wrong place · 🟠 silently wrong · 🟡 tech debt

## Pipeline

| Sev. | Mistake | Symptom | Fix |
|---|---|---|---|
| 🔴 | Pipeline copied from another environment without adapting it (groups, service connection, working dir) | "Develop" deploys to another environment with another environment's code | Parameterize by `env` from a template; check those 4 points in every PR |
| 🔴 | Phases run out of order | `plan` fails on `data.*` ("not found") | Region → Plan → Client; chain them with stages or triggers |
| 🟠 | State container name with uppercase letters | `az storage fs create` fails | `tr '[:upper:]' '[:lower:]'` |
| 🟠 | Plan + apply `-auto-approve` in the same step | destructive changes without review | Separate stages + an `environment` with approval |
| 🟠 | Variable missing from the group | literal `$(x)` reaches Terraform | Pre-check; single quotes |
| 🟡 | Declared parameters that are never used | noise, confusion | Use them (metadata/outputs) or remove them |
| 🟡 | Terraform downloaded without a checksum or a pinned version | non-reproducible builds | Official task or SHA256SUMS verification |
| 🟡 | Empty templates file and YAML duplicated per environment | environments drift apart | Extract templates |

## Terraform: structure

| Sev. | Mistake | Symptom | Fix |
|---|---|---|---|
| 🔴 | `source` in `Envs/<A>` points to `Modules/<B>` | changes in one environment affect another / are not applied | Same environment, or single versioned modules |
| 🔴 | The parent reads `module.x.id` while the child exposes `x_output` (object) | `Unsupported attribute` | `module.x.x_output.id` |
| 🔴 | Root output referencing an undeclared module | `Reference to undeclared module` | Remove it or declare the module |
| 🔴 | Required resource variable not passed by the module (e.g. `workspaceId`) | `Missing required argument` | Wire it from the sibling resource |
| 🔴 | A phase creates another phase's resources | duplicated state, cross-phase destroys | Move it to its phase; read it with `data` |
| 🟠 | `try(module.x[0].attr, null)` with a misspelled `attr` | output always `null`, no error | Check names; use `try` only for the `count = 0` case |
| 🟠 | The module receives a variable and does not forward it (e.g. tags) | the resource is created without it | Forward it or remove it from the interface |
| 🟠 | Dependent flag not validated (`module.common[0]` with `enable_common=false`) | `Invalid index` | `validation` / `check` |
| 🟡 | Providers declared but unused | slower `init`, larger surface | Remove them |
| 🟡 | Variables without `type` | unexpected coercions | Always set a type |
| 🟡 | Defaults from another environment (tags, SKUs) | mislabeled resources | Neutral defaults |
| 🟡 | Unused or buggy "wiring" locals | confusing maintenance | Remove or use them |

## Terraform: naming

| Sev. | Mistake | Symptom | Fix |
|---|---|---|---|
| 🟠 | `replace(x, "[^a-z0-9-]", "")` without slashes | invalid characters reach Azure | `"/[^a-z0-9-]/"` |
| 🟠 | Truncation computed but not used | Storage name > 24 characters | Pass the truncated name |
| 🟠 | Global name without the instance identifier (e.g. KV without `count_c`) | `already exists` / soft delete blocks it | Include every discriminator |
| 🟠 | Lookup pattern differs from the one in the phase that creates the resource | `data` not found | One documented pattern per phase |
| 🟠 | Region not supported for a type | provider error at apply | A dedicated `location` for that type |

## Variables and local environment

| Sev. | Mistake | Symptom | Fix |
|---|---|---|---|
| 🟠 | Local `terraform.tfvars` with values that differ from CI (abbreviations, SKUs) | local plan ≠ CI plan | Versioned `.example`; do not override naming |
| 🟠 | SKU incompatible with the configuration (e.g. dedicated SKU + elastic pool) | error at apply | Variable validation |
| 🟡 | Pipeline `-var` for undeclared variables | `undeclared variable` | Keep the list in sync (or use a generated tfvars.json) |

## Debugging checklist

1. Which working directory, groups and service connection did the run actually use?
2. Do the earlier phases' resources exist, with the same names?
3. Does the instance's state container exist, and is it lowercase?
4. `Unsupported attribute` / `Missing required argument` → output/input contracts between tiers.
5. Type error on a `-var` → missing variable, or a list/map not in HCL syntax.
6. `already exists` / invalid name → naming: regex, truncation, uniqueness, region.
7. Local vs CI differences → local tfvars, Terraform version, lock file.
