# Naming

## Pattern

`<abbr>-<workload>-<env>-<regionCode>[-<instance>]`, using Microsoft CAF abbreviations.

Example with `workload=shop env=dev region_code=eus instance=01`:

| Resource | Abbr | Name |
|---|---|---|
| Resource group | `rg` | `rg-shop-dev-eus` |
| Log Analytics | `log` | `log-shop-dev-eus` |
| Application Insights | `appi` | `appi-shop-dev-eus` |
| Virtual network | `vnet` | `vnet-shop-dev-eus` |
| Subnet | `snet` | `snet-app-shop-dev-eus` |
| Key Vault | `kv` | `kv-shop-dev-eus-01` |
| Storage account | `st` | `stshopdeveus01` |
| App Service Plan | `asp` | `asp-shop-dev-eus` |
| Web App | `app` | `app-shop-dev-eus-01` |
| Function App | `func` | `func-shop-dev-eus-01` |
| SQL Server / DB | `sql` / `sqldb` | `sql-shop-dev-eus-01` / `sqldb-shop-dev-eus` |
| Container registry | `cr` | `crshopdeveus01` |
| Managed identity | `id` | `id-shop-dev-eus` |

## Name classes

| Class | Charset | Types | How to build |
|---|---|---|---|
| STANDARD | `a-z0-9-` | RG, LAW, App Insights, VNet, Plan, SQL DB, Web/Function App | `replace(s, "/[^a-z0-9-]/", "")` |
| ALPHANUM | `a-z0-9` | Storage account (3–24), Container registry (5–50) | `substr(replace(s, "/[^a-z0-9]/", ""), 0, N)` |
| SHORT | `a-z0-9-`, ≤ 24, starts with a letter, no trailing hyphen | Key Vault (3–24) | `substr(s, 0, 24)` + `trimsuffix(…, "-")` |

## Global uniqueness

Storage accounts, Key Vaults, Web/Function Apps, Static Web Apps, container registries and SQL
servers have globally unique DNS names. Add the `instance` suffix, or a short deterministic
hash (`substr(sha1("${var.subscription_id}${var.workload}"), 0, 4)`), to them. Do not use
`random_string` for names unless you accept that the name is unknown until apply.

Key Vault keeps its name while soft-deleted; with purge protection on, you cannot reuse it
until the retention period ends. Plan names so a destroy/recreate in dev does not collide.

## Implementation rules

- Compute every name in `locals.tf` as a map (`local.names.<key>`) and pass it to the modules;
  modules never build names.
- Normalize inputs with `lower()` before `format()`.
- **Regex in Terraform goes between slashes**: `replace(s, "/[^a-z0-9-]/", "")`. Without slashes
  (`"[^a-z0-9-]"`) it is a literal replacement and cleans nothing.
- Sanitize first, truncate second, and pass the truncated value: the one you actually use.
- Add `validation` blocks on module name variables with the type's regex, so a bad name fails
  at plan time instead of at the API.
- Export the name map as a root output: it makes debugging and documentation easier.
- Some types are limited to certain regions (e.g. Static Web Apps). Give them their own `location` variable.
