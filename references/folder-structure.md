# Folder structure and conventions

```
<iac-repo>/
├─ .gitignore
├─ .tflint.hcl                 tflint rules (azurerm ruleset)
├─ README.md                   environment overview, how to plan/apply, owners
├─ pipelines/
│  ├─ azure-pipelines.yml      validate → plan → approval → apply
│  └─ templates/
│     ├─ terraform-validate.yml
│     ├─ terraform-plan.yml
│     └─ terraform-apply.yml
└─ terraform/
   ├─ resources/
   │  └─ <provider>_<type>/    azure_key_vault, azure_storage_account, azure_web_app_linux, …
   │     ├─ main.tf
   │     ├─ variables.tf
   │     └─ outputs.tf
   ├─ modules/
   │  └─ <domain>/             foundation, network, security, data, compute, application, monitoring
   │     ├─ main.tf
   │     ├─ variables.tf
   │     ├─ outputs.tf
   │     ├─ versions.tf        required_providers only
   │     └─ README.md          inputs, outputs, example
   └─ env/                     THE environment (e.g. dev)
      ├─ backend.tf
      ├─ versions.tf           required_version + required_providers
      ├─ providers.tf          provider blocks
      ├─ variables.tf
      ├─ locals.tf             names, tags
      ├─ main.tf               module calls and wiring
      ├─ outputs.tf
      ├─ moved.tf              moved {} blocks (optional)
      ├─ terraform.tfvars      non-secret environment values (versioned)
      └─ .terraform.lock.hcl   versioned
```

## .gitignore

```
**/.terraform/*
*.tfstate
*.tfstate.*
*.tfplan
tfplan
crash.log
crash.*.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json
*.auto.tfvars          # local-only overrides
.terraformrc
terraform.rc
```

`terraform.tfvars` **is** versioned because it holds non-secret configuration. If your
organization forbids that, version a `terraform.tfvars.example` and generate the real file in CI.

## Conventions

| Element | Convention | Example |
|---|---|---|
| Resource folder | `<provider>_<type_snake>` | `azure_function_app_linux` |
| Middle module folder | `<domain>` (lowercase, singular noun) | `network`, `application` |
| Resource block name | `this` when there is one per Resource | `azurerm_key_vault.this` |
| Resource variables | `<snake_attribute>` with the prefix the tier needs | `sku_name`, `retention_days` |
| Module variables | domain-scoped, typed, with description | `key_vault_sku_name` |
| Resource output | `<type>_output` (object) | `key_vault_output` |
| Module outputs | object + flat `<resource>_id` / `<resource>_name` | `key_vault_id` |
| Flags | `enable_<domain>` | `enable_data` |
| Files | `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, `locals.tf` | — |

Pick one variable style (snake_case is the Terraform standard) and apply it everywhere.
Mixing camelCase and snake_case across tiers makes wiring error-prone.
