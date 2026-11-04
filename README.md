# iac-terraform

An AI agent skill ([Agent Skills](https://skills.sh) format) for structuring and working in
**Terraform Infrastructure-as-Code repositories that deploy a single environment** (for
example `dev`). Code is organized in three tiers: atomic **Resources** → **Middle Modules**
(domain building blocks) → one **root module** with the environment's backend, providers,
naming and wiring.

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
| [SKILL.md](SKILL.md) | Three-tier model, critical rules, folder structure, deployment flow, variables, naming, recipes, checklist |
| [references/architecture.md](references/architecture.md) | Tiers, typical middle modules, dependency graph, ordering via references, when to split the root, providers |
| [references/folder-structure.md](references/folder-structure.md) | Folder tree, `.gitignore`, naming conventions for folders, variables and outputs |
| [references/templates.md](references/templates.md) | HCL templates for a Resource, a middle module and the root |
| [references/pipelines.md](references/pipelines.md) | Validate → plan → approval → apply pipeline, backend and identity permissions |
| [references/variables.md](references/variables.md) | Variable chain and precedence, where each value goes, variable design, flags, tags |
| [references/naming.md](references/naming.md) | Naming pattern, name classes, global uniqueness, implementation rules |
| [references/anti-patterns.md](references/anti-patterns.md) | Common mistakes with symptom and fix, debugging checklist |

## Scope

- Terraform with Azure providers (`azurerm`, `azapi`); the structure applies to other clouds.
- CI/CD examples use Azure DevOps Pipelines; the same stages work in GitHub Actions or GitLab CI.
- One environment per repository (one root, one state). For larger environments, see
  "When to split the root" in [references/architecture.md](references/architecture.md).
