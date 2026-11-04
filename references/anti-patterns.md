# Common mistakes

🔴 breaks or deploys something unintended · 🟠 silently wrong · 🟡 tech debt

## Structure

| Sev. | Mistake | Symptom | Fix |
|---|---|---|---|
| 🔴 | The parent reads `module.x.id` while the child exposes `x_output` (an object) | `Unsupported attribute` | `module.x.x_output.id`, or add a flat output |
| 🔴 | Root output referencing an undeclared module | `Reference to undeclared module` | Remove it or declare the module |
| 🔴 | Required Resource variable not passed by the middle module | `Missing required argument` | Wire it, e.g. from a sibling Resource's output |
| 🔴 | `provider` blocks inside modules | modules cannot be used with `count`/`for_each`; provider removal errors | Providers only in the root; modules declare `required_providers` only |
| 🔴 | Renaming a module or resource without `moved {}` | plan destroys and recreates resources | Add a `moved` block in the same change |
| 🟠 | `try(module.x[0].attr, null)` with a misspelled `attr` | output is always `null`, no error | Check names; use `try` only for the `count = 0` case |
| 🟠 | The module receives a variable and does not forward it (e.g. `tags`) | the resource is created without it | Forward it, or remove it from the interface |
| 🟠 | Dependent flags not validated (`module.data[0]` with `enable_data=false`) | `Invalid index` | `validation` or `check` block |
| 🟠 | `depends_on` on whole modules | broad "known after apply", slow and noisy plans | Use output references; scope `depends_on` to resources |
| 🟠 | Data source reading a resource created in the same root without referencing it | `not found` on the first apply | Reference the resource, or drop the data source |
| 🟡 | Module per resource type ("module of a module") with no domain | deep trees and long variable plumbing | Group by domain and lifecycle |
| 🟡 | Providers declared but unused | slower `init` | Remove them |
| 🟡 | Variables without `type` / `description` | unexpected coercion, unclear interface | Always set them |
| 🟡 | Mixed naming styles (camelCase + snake_case) | wiring mistakes | snake_case everywhere |

## Naming

| Sev. | Mistake | Symptom | Fix |
|---|---|---|---|
| 🟠 | `replace(x, "[^a-z0-9-]", "")` without slashes | invalid characters reach the API | `"/[^a-z0-9-]/"` |
| 🟠 | Truncation computed but not used | Storage name > 24 characters | Pass the truncated name |
| 🟠 | Globally unique name without an instance suffix | `already exists`, or soft delete blocks reuse | Add an instance suffix or a deterministic hash |
| 🟠 | Names built inside modules | inconsistent names, hard to predict | Build them in root `locals.tf` only |
| 🟠 | Region not supported by a type | provider error at apply | A dedicated `location` variable for that type |

## Variables and state

| Sev. | Mistake | Symptom | Fix |
|---|---|---|---|
| 🔴 | Secrets in `terraform.tfvars` or in defaults | secrets in git history | `TF_VAR_*` from Key Vault, or generate them and store in Key Vault |
| 🔴 | Local state, or state committed to git | lost or conflicting state, secrets leaked | Remote backend with locking; ignore `*.tfstate*` |
| 🔴 | Backend created by the same root that uses it | chicken-and-egg problem; destroy deletes the state | Bootstrap the backend separately |
| 🟠 | The same variable set in several sources (tfvars + `TF_VAR_` + `-var`) | the value depends on precedence | One source per variable |
| 🟠 | Sensitive output without `sensitive = true` | secret printed in plan/apply logs | Mark it sensitive |
| 🟠 | `count`/`for_each` based on values known only after apply | `Invalid count argument` | Base them on variables or locals |
| 🟡 | `.terraform.lock.hcl` not committed | different provider versions per machine | Commit it; lock for every platform |

## Pipeline

| Sev. | Mistake | Symptom | Fix |
|---|---|---|---|
| 🔴 | `terraform apply -auto-approve` without a reviewed plan | unreviewed destructive changes | Plan artifact + approval + apply of that plan |
| 🟠 | Terraform version differs between CI and local | lock file churn, state upgrades | Pin `required_version` and the CI version |
| 🟠 | No drift detection | manual portal changes reverted unexpectedly | Scheduled `plan -detailed-exitcode` |
| 🟠 | Plan/outputs with secrets printed in logs | leaked secrets | Do not echo them; publish them as restricted artifacts |
| 🟡 | Terraform downloaded without a checksum | non-reproducible, supply-chain risk | Official task or SHA256SUMS verification |
| 🟡 | Using `-target` routinely to make applies work | hidden dependency bugs | Add the missing reference |

## Debugging checklist

1. Run `terraform init -upgrade=false` and `terraform validate` locally, with the CI Terraform version.
2. `Unsupported attribute` / `Missing required argument` → output and input contracts between tiers.
3. `Invalid index` → a flag with `count = 0` while something else uses `[0]`.
4. Type errors → tfvars syntax for lists/maps, or a missing `type`.
5. `already exists` / invalid name → regex, truncation, uniqueness, soft-deleted Key Vault, region.
6. Unexpected destroy/recreate → missing `moved {}`, a changed name, or a `ForceNew` attribute.
7. Plan differs between local and CI → a different var source, Terraform version or lock file.
8. State lock errors → a previous run was cancelled; check it, then `terraform force-unlock <id>` with care.
