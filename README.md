# iac-terraform

An AI agent skill ([Agent Skills](https://skills.sh) format) for structuring and working in
**Terraform Infrastructure-as-Code** repositories built on **Middle Modules** that control the
phases of cloud provisioning, executed in order (**Region → Plan → Client**), for any
environment (**Develop / QA / UAT / PROD**).

## Install

```bash
npx skills add Faralas/IaC_Client_Develop_skill
```

Or copy the folder into `~/.claude/skills/iac-terraform/` (or into the repo's `.claude/skills/`).

## Contents

| File | What it covers |
|---|---|
| [SKILL.md](SKILL.md) | Tier × phase model, critical rules, structure, flow, variables, naming, recipes and checklist |
| [references/architecture-and-phases.md](references/architecture-and-phases.md) | Resources → Middle Modules → Envs tiers, phases and their order, dependency tree, how phases connect |
| [references/folder-structure.md](references/folder-structure.md) | Folder tree and naming conventions for files, variables and outputs |
| [references/templates.md](references/templates.md) | HCL templates per tier (resource, middle module, root module) |
| [references/pipelines.md](references/pipelines.md) | Pipelines per phase × environment: parameters, variable groups, backend, plan/apply, outputs |
| [references/variables.md](references/variables.md) | Variable chain, classification, example mapping, typing pitfalls, flags |
| [references/naming.md](references/naming.md) | Patterns per phase, name classes, global uniqueness, regex |
| [references/anti-patterns.md](references/anti-patterns.md) | Common mistakes with symptom and fix, debugging checklist |

## Scope

- Terraform with Azure providers (`azurerm`, `azapi`); the model applies to other clouds.
- CI/CD with Azure DevOps Pipelines; the same pattern works for GitHub Actions or GitLab CI.
- Any environment: the environment is a parameter, not a code variant.
