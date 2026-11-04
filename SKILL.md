---
name: iac-terraform
description: >
  General-purpose guide for Terraform Infrastructure-as-Code repositories that deploy a
  single environment (e.g. Dev). Code is organized in three tiers: atomic Resources →
  Middle Modules (domain building blocks) → one root module for the environment. Use it
  when creating or reviewing such a repo; adding a resource or a middle module; wiring
  modules together; designing naming, variables, tfvars, backend/state or providers;
  writing or debugging the plan/apply CI/CD pipeline; or troubleshooting a failing plan.
---

# IaC Terraform — single-environment repository

This skill defines how to structure and work in a Terraform repository that deploys **one
environment** (for example `dev`). The repo holds one root module, one state and one
pipeline. Inside the repo, code is composed in three tiers so it stays reusable and easy to review.

## When to use

- "Set up the Terraform structure for this repo"
- "Add a storage account / key vault / app service / database"
- "Create a middle module for <domain>"
- "Wire module X's output into module Y"
- "What should this resource be named?" / "Where does this variable come from?"
- "Create or review the plan/apply pipeline"
- "Why is `terraform plan` failing?"

## The three tiers

```
Resources ─────────► Middle Modules ─────────► Root (the environment)
one resource each    one domain each          backend, providers, naming,
no logic             composes Resources       flags, wiring, outputs
```

| Tier | Folder | Contains | Never contains |
|---|---|---|---|
| **Resources** | `terraform/resources/<provider>_<type>/` | One resource, its variables, **one object output** | backend, providers, naming logic |
| **Middle Modules** | `terraform/modules/<domain>/` | Several Resources of one domain (network, data, app…) and their internal wiring | backend, providers, environment values |
| **Root** | `terraform/env/` | `backend.tf`, `providers.tf`, `variables.tf`, `locals.tf`, `main.tf`, `outputs.tf`, `terraform.tfvars` | resource blocks for things a module already covers |

Details, dependency graph and ordering: [references/architecture.md](references/architecture.md).

## Critical rules

1. **One environment, one root, one state.** The root in `terraform/env/` is the only place
   with a backend and provider configuration. If the environment grows too big, split it
   into several roots with separate states, and connect them with `terraform_remote_state` or `data` sources.
2. **Dependencies flow one way: Root → Middle Modules → Resources.** A Resource never
   calls a module; a middle module never reads another middle module directly. The root
   passes outputs from one module as inputs to another.
3. **Order comes from references, not from file order.** Terraform builds the graph from
   the expressions you use. Prefer passing outputs (`module.network.subnet_id`) over
   `depends_on`, and use `depends_on` only for dependencies that have no data flow.
4. **Explicit output contract.** A Resource exposes `<type>_output = { id, name, … }`; a
   middle module reads `module.<x>.<type>_output.<attr>` and re-exports flat outputs; the
   root reads module outputs (`try(module.<m>[0].<out>, null)` when the module uses `count`).
   Every referenced attribute must exist in the child.
5. **Names are computed once, in `locals.tf`.** Deterministic, lowercase, sanitized with a regex
   between slashes, and truncated to each type's limit.
6. **Environment values live in the root**: `terraform.tfvars` holds non-secret configuration
   (versioned), CI injects secrets through `TF_VAR_*`, and `variables.tf` defaults hold constants.
   Modules get everything through variables and never hard-code environment values.
7. **One feature flag per optional middle module** (`enable_<domain>` + `count`). If one
   module needs another, validate that the flags are consistent.
8. **Always plan before apply**, and in CI apply exactly the reviewed plan file.

## Folder structure

```
<iac-repo>/
├─ .gitignore                 .terraform/, *.tfstate*, *.tfplan, crash.log, override.tf, *.auto.tfvars (local)
├─ README.md                  what the environment contains and how to deploy it
├─ pipelines/
│  ├─ azure-pipelines.yml     PR validation + plan + approval + apply
│  └─ templates/              terraform-plan.yml, terraform-apply.yml
└─ terraform/
   ├─ resources/<provider>_<type>/   main.tf · variables.tf · outputs.tf
   ├─ modules/<domain>/              main.tf · variables.tf · outputs.tf · README.md
   └─ env/                           backend.tf · providers.tf · versions.tf
                                     variables.tf · locals.tf · main.tf · outputs.tf
                                     terraform.tfvars · .terraform.lock.hcl
```

Details and conventions: [references/folder-structure.md](references/folder-structure.md).
File templates: [references/templates.md](references/templates.md).

## Deployment flow

1. A PR runs `fmt -check`, `init -backend=false`, `validate` and `tflint`.
2. On merge (or a manual run) the pipeline authenticates, runs `init` against the remote
   backend, and runs `plan -out=tfplan`, which it publishes as an artifact.
3. An approval gate (pipeline `environment`) shows the plan to reviewers.
4. Apply runs `terraform apply tfplan`, using the same plan and the same Terraform and provider versions.
5. `terraform output -json` is exported as an artifact for downstream consumers.

Details: [references/pipelines.md](references/pipelines.md).

## Variables

| Source | Purpose | Example |
|---|---|---|
| `terraform.tfvars` (versioned) | non-secret environment configuration | `location`, SKUs, flags, address spaces |
| `TF_VAR_*` / pipeline secrets | secrets and identity-dependent values | admin passwords, object IDs |
| `variables.tf` defaults | constants and safe defaults | abbreviations, retention |
| `locals.tf` | computed names, tags, wiring maps | `local.names.kv` |
| `data` sources | existing resources not managed here | client config, shared DNS zone |

Full chain and pitfalls: [references/variables.md](references/variables.md).

## Naming

Base pattern: `<abbr>-<workload>-<env>-<regionCode>[-<instance>]`, e.g. `kv-shop-dev-eus-01`.
It is computed in `locals.tf` with `lower()` + `format()` and sanitized with
`replace(x, "/[^a-z0-9-]/", "")`. Types that only allow alphanumerics (Storage, ACR) drop the
hyphens and are truncated to their limit.
Rules per type: [references/naming.md](references/naming.md).

## Recipes

**Add a resource**
1. Reuse or create `terraform/resources/<provider>_<type>/` with a single object output.
2. Add it to the middle module of its domain, and re-export flat outputs (`<x>_id`, `<x>_name`).
3. Add its name to `locals.tf`.
4. Add any new variables to the module and the root (with a default if it is safe), and set
   environment values in `terraform.tfvars`.
5. Pass the inputs in the root `main.tf`, wiring other modules' outputs instead of hard-coded IDs.
6. Expose what consumers need in the root `outputs.tf`.
7. `terraform fmt`, `validate`, `plan`, then review the plan.

**Add a middle module**: create `terraform/modules/<domain>/`, compose the Resources, define
variables and outputs, add an `enable_<domain>` flag if it is optional, call it from the
root, and add an aggregated root output.

**Use an existing resource you do not manage**: use a `data` source in the root, and pass
its attributes to the module. Never `import` something another team owns.

**Rename or move a resource without recreating it**: add a `moved {}` block in the root
(or in the module) in the same change.

## Checklist before accepting a change

- [ ] Only the root has backend and provider configuration; modules declare `required_providers` only.
- [ ] Every `module.X.<attr>` exists in the child, and every required variable receives a value.
- [ ] Dependencies are expressed through references; `depends_on` only where unavoidable.
- [ ] Names respect each type's length limit, charset and global uniqueness.
- [ ] No secrets in `terraform.tfvars`, in defaults or in outputs without `sensitive = true`.
- [ ] Renames and moves have `moved {}` blocks; the plan shows no unexpected destroy.
- [ ] `terraform fmt -check`, `terraform validate` and `tflint` are clean, and the plan was reviewed.

Common mistakes with symptom and fix: [references/anti-patterns.md](references/anti-patterns.md).
