# iac-terraform

An AI agent skill ([Agent Skills](https://skills.sh) format) for structuring and working in
**Terraform Infrastructure-as-Code** repositories built on **Middle Modules** that control the
phases of cloud provisioning, executed in order (**Region → Plan → Client**), for any
environment (**Develop / QA / UAT / PROD**).

## Install

Global (available in every project):

```bash
npx skills add Faralas/iac-terraform-skill -g -a claude-code
```

Current project only (installs into the project's skills folder):

```bash
npx skills add Faralas/iac-terraform-skill -a claude-code
```

Manual alternative: clone into `~/.claude/skills/iac-terraform/` (or into the repo's `.claude/skills/iac-terraform/`).

```bash
git clone https://github.com/Faralas/iac-terraform-skill.git ~/.claude/skills/iac-terraform
```

## Update

```bash
npx skills check                  # list skills with available updates
npx skills update iac-terraform   # update this skill
```

If you installed it with `git clone`:

```bash
git -C ~/.claude/skills/iac-terraform pull
```

## Uninstall

```bash
npx skills remove iac-terraform
```

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
