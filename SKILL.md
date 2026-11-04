---
name: iac-terraform
description: >
  Guide for structuring and working in Terraform Infrastructure-as-Code repositories
  organized in three tiers (atomic Resources → Middle Modules → Envs) and in ordered
  provisioning phases (Region → Plan → Client) for any environment (Develop, QA, UAT,
  PROD). Use it when creating or reviewing an IaC repo; adding a resource, a middle
  module, a phase or an environment; wiring phases together; designing naming, variable
  handling or backend/state; writing or debugging plan/apply CI/CD pipelines; or
  promoting infrastructure between environments.
---

# IaC Terraform — Middle Modules by Phase

This skill defines **how to structure** an IaC repository where *middle modules*
(domain-level intermediate modules) control each phase of cloud provisioning, and
**in what order** those phases run. It applies to any environment: the environment is a
parameter, not a code variant.

## When to use

- "Set up the IaC structure for this repo / new project"
- "Add a resource / a middle module / a phase / an environment"
- "Which phase does this resource belong to?" / "How does the Client phase consume what Region created?"
- "What should this resource be named?" / "Where does this variable come from?"
- "Create or review the plan/apply pipeline for <phase> in <environment>"
- "Promote the infrastructure from Develop to QA/UAT/PROD"
- "Why is `plan` failing?"

## Two-axis model

```
                 TIER (how code is composed)
                 Resources ──► Middle Modules ──► Envs/<Env>/<Phase>  (root module)

PHASE (provi-    1. Region  ─ regional shared foundation (network, RG, SQL Server, pools, observability)
sioning order)   2. Plan    ─ shared compute capacity by size (App Service Plan, node pools)
                 3. Client  ─ per-client/tenant workload (apps, databases, storage, secrets)
```

- **Resources** (`Terraform/Resources/<provider>_<type>/`): one resource, no business logic,
  **a single object output**.
- **Middle Modules** (`Terraform/Modules/<Phase>_IaC_<Domain>/`): compose Resources by
  domain (Common, Application, Data, Monitoring…). They are the unit each phase turns on or
  off. No backend or providers.
- **Envs** (`Terraform/Envs/<Env>/<Phase>/`): root modules. The only place with `backend.tf`,
  `providers.tf`, naming (`local.tf`) and feature flags. One per phase and environment.

Details and dependency tree: [references/architecture-and-phases.md](references/architecture-and-phases.md).

## Critical rules

1. **Phases run in order and only depend backwards**: Region → Plan → Client.
   A phase never creates or modifies another phase's resources; it only *reads* them
   (data sources by deterministic name, or `terraform_remote_state`).
2. **One state per phase and instance.** Client: one state per client; Region: per region;
   Plan: per plan. Never mix phases in one state.
3. **The environment is a parameter.** The same Resources and Middle Modules serve every
   environment; what changes is `Envs/<Env>/…`, the variable group, the service connection
   and the subscription. If you duplicate modules per environment, every `source` must point
   to the folder **of the same environment**, and you must verify it on every promotion.
4. **A pipeline must target the environment in its name**: working directory, variable
   groups, service connection and backend. This is the most expensive copy-paste mistake.
5. **Explicit output contract**: Resources expose `<type>_output = { id, name, … }`; middle
   modules read `module.<x>.<type>_output.<attr>`; Envs read middle-module outputs with
   `try(module.<m>[0].<out>, null)`. Every reference must exist in the child.
6. **Naming is deterministic and computed only in `Envs/*/local.tf`.** Later phases rebuild
   the names of earlier phases' resources from the same inputs.
7. **One feature flag per middle module** (`enable_<module>` + `count`). If one module indexes
   another (`module.common[0]`), document and validate the dependency between flags.
8. **`*.tfvars` files are not versioned**: in CI, values arrive via `-var`/`TF_VAR_*` and the
   defaults in `variables.tf`. A local plan with `terraform.tfvars` is not comparable to CI.

## Folder structure

```
<iac-repo>/
├─ Pipelines/
│  ├─ Templates/                         reusable templates (init/plan/apply, outputs)
│  └─ <Env>/                             Develop | QA | UAT | PROD
│     ├─ regionIac<Env>.yml              phase 1
│     ├─ planIac<Env>.yml                phase 2
│     └─ clientIac<Env>.yml              phase 3
└─ Terraform/
   ├─ Resources/<provider>_<type>/       main|<type>.tf · variables.tf · output.tf
   ├─ Modules/<Phase>_IaC_<Domain>/      main.tf · variables.tf · output.tf
   └─ Envs/<Env>/<Phase>/                backend.tf · providers.tf · variables.tf
                                         local.tf · main.tf · output.tf
```

Details and conventions: [references/folder-structure.md](references/folder-structure.md).
File templates: [references/templates.md](references/templates.md).

## Flow of a phase deployment

1. The pipeline receives instance parameters (region, codes, size, client…).
2. It authenticates with the environment's service connection and exports `ARM_*`.
3. It ensures the instance's state container exists and runs `terraform init` with `-backend-config`.
4. It computes derived flags (e.g. enable a module depending on the client type).
5. `terraform plan -out=tfplan` → artifact.
6. Approval (`environment`) → `terraform apply tfplan`.
7. It exports `terraform output -json` next to the state for later phases or pipelines.
8. Optional: it triggers the next phase or the post-deployment pipeline.

Details: [references/pipelines.md](references/pipelines.md).

## Variables

| Source | Purpose | Syntax (Azure DevOps) |
|---|---|---|
| Run parameters | instance identity (client, region, codes, size) | `${{ parameters.x }}` → `env:` → `${X}` |
| Per-environment variable groups | SKUs, flags, retention, control backend | `$(x)` |
| Computed in the pipeline | state container name, derived flags | bash |
| `variables.tf` defaults | constants (org, abbreviations) | — |
| `locals` | names and wiring | — |

Full matrix and typing pitfalls: [references/variables.md](references/variables.md).

## Naming

Base pattern: `<abbr>-<org>-<regionCode>-<tenantAbbr>-<instance>`, computed in `local.tf`
with `lower()` + `format()` and sanitized with a regex **between slashes**
(`replace(x, "/[^a-z0-9-]/", "")`). Alphanumeric types (Storage, some capacities) have no
hyphens and are truncated to their limit.
Per-type rules and cross-phase lookup names: [references/naming.md](references/naming.md).

## Recipes

**Add a resource to a phase**
1. Create or reuse `Resources/<provider>_<type>` with a single object output.
2. Add it to the domain's middle module and expose flat outputs read from the object.
3. Compute its name in `Envs/<Env>/<Phase>/local.tf`.
4. Declare variables with defaults in the root; pass them to the module in `main.tf`.
5. If they vary per environment: add a `-var` in the pipeline and the variable in the group of **every** environment.
6. Expose the output in `Envs/.../output.tf` with `try()`.

**Add a middle module**: `Modules/<Phase>_IaC_<Domain>/` + an `enable_<domain>` flag +
a `module` block with `count` and `depends_on` in the root + an aggregated output.

**Add a phase**: decide which phases it reads (earlier ones only), its state granularity,
its naming and its pipeline; document the order.

**Add or promote an environment**: copy `Envs/<SourceEnv>` → `Envs/<TargetEnv>`, adjust
`source` if modules are per environment, create the variable groups and service connection,
and generate the pipelines from the template. Deploy the phases **in order**.

## Checklist before accepting a change

- [ ] The pipeline uses the working directory, variable groups, service connection and backend of the right environment.
- [ ] No `source` crosses environments by mistake.
- [ ] The phase only reads resources from earlier phases, and those names match the ones created there.
- [ ] Every `module.X.<attr>` exists in the child, and every required variable receives a value.
- [ ] Every pipeline `-var` is declared, and every variable without a default is passed.
- [ ] Names respect their type's length limit and charset, and global uniqueness where it applies.
- [ ] There is a reviewed plan before apply, and an approval gate in higher environments.
- [ ] `terraform fmt -check` and `terraform validate` pass on every root you touched.

Common mistakes with symptom and fix: [references/anti-patterns.md](references/anti-patterns.md).
