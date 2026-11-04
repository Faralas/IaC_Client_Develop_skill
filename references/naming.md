# Naming

## Patterns per phase

| Phase | Pattern | Example (`org=acme rc=eus count_r=01 size=s count_p=02 tenant=ctx count_c=0001`) |
|---|---|---|
| Region | `<abbr>-<org>-<rc>-<count_r>` | `rg-acme-eus-01`, `sql-acme-eus-01`, `sqlep-sql-acme-eus-01` |
| Plan | `<abbr>-<org>-<rc>-<size>-<count_p>` | `rg-acme-eus-s-02`, `plan-acme-eus-s-02` |
| Client | `<abbr>-<org>-<rc>-<tenant>-<count_c>` | `rg-acme-eus-ctx-0001`, `appi-acme-eus-ctx-0001` |
| Client (global apps) | `<abbr>-<org>-<segment>-<tenant>-<count_c>` | `app-acme-web-ctx-0001` |

Later phases **rebuild** the names of earlier phases from the same inputs (`rc`, `count_r`,
`size`, `count_p`) to perform lookups. If you change a pattern in Region, change it in the
`local.tf` of Plan and Client too, or move to `remote_state`.

## Name classes (Azure)

| Class | Charset | Types | How to build |
|---|---|---|---|
| STANDARD | `a-z0-9-` | RG, App Insights, LAW, SQL DB, Web/Function App, Plan | `replace(…, "/[^a-z0-9-]/", "")` |
| ALPHANUM | `a-z0-9` | Storage Account (3–24), ACR (5–50), Fabric Capacity (`^[a-z][a-z0-9]*$`) | `substr(replace(…, "/[^a-z0-9]/", ""), 0, N)` |
| SHORT | `a-z0-9-`, ≤ 24 | Key Vault (3–24, global) | `substr(…, 0, 24)` with no trailing hyphen |

## Global uniqueness

Types with a global DNS name (Storage, Key Vault, Web/Function App, SWA, ACR, SQL Server):
the name must include **everything** that distinguishes the instance (tenant **and** `count_c`,
or `rc`). Key Vault also keeps its name while soft-deleted; with purge protection on you
cannot reuse it during the retention period.

## Implementation rules

- Normalize every input with `lower()` before `format()`.
- **Terraform regex goes between slashes**: `replace(s, "/[^a-z0-9-]/", "")`. Without slashes
  (`"[^a-z0-9-]"`) it is a literal replacement and cleans nothing.
- Truncate **after** sanitizing, and apply the truncation to the name you actually pass to the module.
- Declare abbreviations as variables with defaults (so they can be adjusted), but **do not**
  override them per environment: names must be predictable across environments.
- Export the computed names as root outputs: it makes lookups easier to debug.
- If some region has restrictions (e.g. Static Web Apps only in some regions), use a
  dedicated `location` variable for that type.
