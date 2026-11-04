# Pipelines per phase and environment (Azure DevOps)

One pipeline per **phase × environment**, generated from shared templates. Run manually or
chained (Region → Plan → Client). The example uses Azure DevOps; the same pattern applies to
GitHub Actions or GitLab CI.

## What changes per environment

| Element | Develop | QA | UAT | PROD |
|---|---|---|---|---|
| Control variable group (backend) | `library-control-tf-develop` | `…-qa` | `…-uat` | `…-prod` |
| Deployment variable group (SKUs, flags) | `library-deployment-tf-develop` | `…-qa` | `…-uat` | `…-prod` |
| Service connection | Develop subscription SC | QA | UAT | PROD |
| Working directory | `Terraform/Envs/Develop/<Phase>` | `…/QA/<Phase>` | `…/UAT/<Phase>` | `…/PROD/<Phase>` |
| Approval before apply | optional | recommended | required | required + reviewers |

Nothing else should change. If something else does, it is in the wrong place.

## Instance parameters per phase

| Phase | Parameters |
|---|---|
| Region | `region`, `regioncode`, `count_r`, `env` |
| Plan | Region's + `plansize`, `count_p` |
| Client | Plan's + `tenant_code`, `count_c`, business flags (e.g. `is_internal_tenant`) |

## Phase pipeline (reference)

```yaml
trigger: none

parameters:
- { name: env,         type: string, values: [develop, qa, uat, prod] }
- { name: region,      type: string }
- { name: regioncode,  type: string }
- { name: count_r,     type: string }
- { name: plansize,    type: string }
- { name: count_p,     type: string }
- { name: tenant_code, type: string }
- { name: count_c,     type: string }
- { name: is_internal_tenant, type: boolean, default: false }

variables:
- group: library-control-tf-${{ parameters.env }}
- group: library-deployment-tf-${{ parameters.env }}
- name: terraformWorkingDirectory
  value: Terraform/Envs/${{ parameters.env }}/Client     # match the real folder's casing
- name: terraformVersion
  value: 1.14.0

pool: { vmImage: ubuntu-latest }

stages:
- stage: Plan
  jobs:
  - job: Plan
    steps:
    - template: ../Templates/terraform-plan.yml
      parameters: { serviceConnection: $(serviceConnection), … }

- stage: Apply
  dependsOn: Plan
  jobs:
  - deployment: Apply
    environment: iac-${{ parameters.env }}     # approvals configured on the environment
    strategy:
      runOnce:
        deploy:
          steps:
          - template: ../Templates/terraform-apply.yml

- stage: Next
  dependsOn: Apply
  condition: and(succeeded(), eq(variables['triggerNext'], 'true'))
  jobs:
  - job: Trigger
    steps:
    - checkout: none
    # trigger the next phase or post-IaC with the same instance parameters
```

## `terraform-plan.yml` script (inside `AzureCLI@2`, `addSpnToEnvironment: true`)

```bash
set -euo pipefail

export ARM_CLIENT_ID="$servicePrincipalId"
export ARM_CLIENT_SECRET="$servicePrincipalKey"     # or federated: ARM_USE_OIDC=true
export ARM_TENANT_ID="$tenantId"
export ARM_SUBSCRIPTION_ID="$(az account show --query id -o tsv)"

# 1 state container per instance; container names must be lowercase
CONTAINER_NAME="$(echo "${TENANT_CODE}-${COUNT_C}" | tr '[:upper:]' '[:lower:]')"
if [ "$(az storage fs exists --account-name "$(controlStorageAccount)" \
        --name "$CONTAINER_NAME" --auth-mode login --query exists -o tsv)" != "true" ]; then
  az storage fs create --account-name "$(controlStorageAccount)" \
     --name "$CONTAINER_NAME" --auth-mode login 1>/dev/null
fi

cd "$(terraformWorkingDirectory)"
terraform init \
  -backend-config="resource_group_name=$(controlResourceGroup)" \
  -backend-config="storage_account_name=$(controlStorageAccount)" \
  -backend-config="container_name=${CONTAINER_NAME}" \
  -backend-config="key=terraform.tfstate"

# business-derived flags
[ "$(echo "${IS_INTERNAL_TENANT}" | tr '[:upper:]' '[:lower:]')" = "true" ] \
  && ENABLE_INTEGRATION_API=false || ENABLE_INTEGRATION_API=true

terraform plan -input=false -out=tfplan \
  -var "location=${REGION}" \
  -var "tenant_abbr=${TENANT_CODE}" \
  -var "instance_count=${COUNT_C}" \
  -var "enable_integration_api=${ENABLE_INTEGRATION_API}" \
  -var 'enable_common=$(enable_common)' \
  -var 'azure_keyVault_skuName=$(azure_keyVault_skuName)'
  # … remaining -var from the variable group
```

Publish `tfplan` (and `.terraform.lock.hcl`) as an artifact. The apply stage runs `init` with
the same backend, downloads the artifact and runs `terraform apply -input=false tfplan`.

## Export outputs for the next phase / post-IaC

```bash
terraform output -json > tfoutputs.json
jq -n --arg env "$ENV" --arg tenant "$TENANT_CODE" --arg count_c "$COUNT_C" \
      --arg build "$BUILD_BUILDNUMBER" --slurpfile tf tfoutputs.json \
      '{metadata:{build:$build}, parameters:{env:$env,tenant:$tenant,count_c:$count_c}, terraform_outputs:$tf[0]}' \
  > values.json
az storage fs file upload --account-name "$(controlStorageAccount)" --auth-mode login \
  --file-system "$CONTAINER_NAME" --source values.json --path values.json --overwrite true
```

Do not print `values.json` to the logs: it may contain sensitive outputs.

## Pipeline identity permissions

- Contributor (or a custom role) on the subscription or on the phase's RGs.
- Storage Blob Data Contributor on the control account (`--auth-mode login`).
- Read access to earlier phases' resources (for the `data` sources).
- Provider-specific permissions (e.g. to create dedicated capacities).
- If it creates Key Vaults with an access policy for itself, it becomes a secrets administrator: decide whether that is what you want.

## Good practices

- Install Terraform with an official task or verify the checksum; pin the version.
- Always use `-input=false` in CI.
- Separate plan and apply, with approval in higher environments.
- One template per step; environment YAML files only declare parameters and variables.
- Validate in PRs: `terraform fmt -check`, `terraform validate` and `tflint` for each root.
