# Folder structure and conventions

```
<iac-repo>/
├─ .gitignore              *.tfvars, *.tfvars.json, backend.local.hcl, .terraform/, *.tfstate*, *.out, override.tf
├─ README.md               phases, execution order, environments
├─ Pipelines/
│  ├─ Templates/
│  │  ├─ terraform-plan.yml       init + plan + publish artifact
│  │  ├─ terraform-apply.yml      download artifact + apply + export outputs
│  │  └─ trigger-next-phase.yml
│  └─ <Env>/                      Develop | QA | UAT | PROD
│     ├─ regionIac<Env>.yml
│     ├─ planIac<Env>.yml
│     └─ clientIac<Env>.yml
└─ Terraform/
   ├─ Resources/
   │  └─ <provider>_<type>/       azure_key_vault, azure_storage_account, azure_sql_db, …
   │     ├─ <type>.tf | main.tf
   │     ├─ variables.tf
   │     └─ output.tf
   ├─ Modules/
   │  └─ <Phase>_IaC_<Domain>/    Client_IaC_Common, Plan_IaC_Application, Region_IaC_Monitoring, …
   │     ├─ main.tf
   │     ├─ variables.tf
   │     └─ output.tf
   └─ Envs/
      └─ <Env>/
         ├─ Region/   backend.tf providers.tf variables.tf local.tf main.tf output.tf
         ├─ Plan/     (same)
         └─ Client/   (same)
```

## Variant: per-environment modules

Some repos duplicate `Modules/<Env>/<Phase>_IaC_<Domain>/` so each environment can diverge.
That is valid, but it requires:

- every `source` in `Envs/<Env>/*` to point to `Modules/<Env>/…` of the **same** environment;
- comparing the modules with `diff -r` when promoting;
- in the medium term, preferring single versioned modules (git tags or a registry) with the
  version pinned per environment.

## Conventions

| Element | Convention | Example |
|---|---|---|
| Resource folder | `<provider>_<type_snake>` | `azure_function_app_windows` |
| Middle module folder | `<Phase>_IaC_<Domain>` | `Client_IaC_Application` |
| Resource variables | `<provider>_<resourceCamel>_<attribute>` | `azure_keyVault_skuName` |
| Domain suffix in modules | `_<domain>` when a type repeats | `azure_windowsWebApp_name_transporter` |
| Resource output | `<type>_output` (object) | `azure_key_vault_output` |
| Module outputs | object + flat `<resource>_id`, `<resource>_name` | `key_vault_id` |
| Flags | `enable_<module>` | `enable_common` |
| Files | singular | `local.tf`, `output.tf` |
| Pipelines | `<phase>Iac<Env>.yml` | `clientIacQA.yml` |

Be consistent: if a resource uses `location` without a prefix, document it as an exception or normalize it.
