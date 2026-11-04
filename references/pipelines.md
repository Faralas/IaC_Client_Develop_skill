# CI/CD pipeline (Azure DevOps)

One pipeline for the environment, with three stages: **Validate → Plan → Apply**. The approval
gate is the Apply stage's `environment`, so apply waits for a reviewer after the plan is published.
The example uses Azure DevOps; the same structure applies to GitHub Actions or GitLab CI.

## Pipeline

```yaml
trigger:
  branches: { include: [main] }
  paths:    { include: [terraform/*, pipelines/*] }

pr:
  branches: { include: [main] }

variables:
- group: tf-backend-dev           # backendResourceGroup, backendStorageAccount, backendContainer
- group: tf-secrets-dev           # Key Vault-linked: secrets exposed as TF_VAR_*
- name: workingDirectory
  value: terraform/env
- name: terraformVersion
  value: 1.14.0
- name: serviceConnection
  value: sc-azure-dev

pool: { vmImage: ubuntu-latest }

stages:
- stage: Validate
  jobs:
  - job: Validate
    steps:
    - template: templates/terraform-validate.yml

- stage: Plan
  dependsOn: Validate
  condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
  jobs:
  - job: Plan
    steps:
    - template: templates/terraform-plan.yml

- stage: Apply
  dependsOn: Plan
  condition: and(succeeded(), eq(dependencies.Plan.outputs['Plan.plan.hasChanges'], 'true'))
  jobs:
  - deployment: Apply
    environment: iac-dev              # approvals and checks configured here
    strategy:
      runOnce:
        deploy:
          steps:
          - checkout: self
          - template: templates/terraform-apply.yml
```

For PRs it is also useful to run the plan and post its summary as a PR comment, without applying.

## `terraform-validate.yml`

```bash
terraform fmt -check -recursive terraform
cd terraform/env
terraform init -backend=false -input=false
terraform validate
tflint --init && tflint --recursive
```

## `terraform-plan.yml` (inside `AzureCLI@2`, `addSpnToEnvironment: true`)

```bash
set -euo pipefail
export ARM_CLIENT_ID="$servicePrincipalId"
export ARM_TENANT_ID="$tenantId"
export ARM_SUBSCRIPTION_ID="$(az account show --query id -o tsv)"
export ARM_USE_OIDC=true                       # workload identity federation (preferred)
export ARM_OIDC_TOKEN="$idToken"               # or ARM_CLIENT_SECRET="$servicePrincipalKey"
export TF_IN_AUTOMATION=true
export TF_VAR_subscription_id="$ARM_SUBSCRIPTION_ID"

cd "$(workingDirectory)"
terraform init -input=false \
  -backend-config="resource_group_name=$(backendResourceGroup)" \
  -backend-config="storage_account_name=$(backendStorageAccount)" \
  -backend-config="container_name=$(backendContainer)" \
  -backend-config="use_azuread_auth=true"

set +e
terraform plan -input=false -lock-timeout=5m -detailed-exitcode -out=tfplan
rc=$?
set -e
[ $rc -eq 1 ] && exit 1
echo "##vso[task.setvariable variable=hasChanges;isOutput=true]$([ $rc -eq 2 ] && echo true || echo false)"
terraform show -no-color tfplan > plan.txt
```

Name the step `plan` so that `Plan.plan.hasChanges` resolves. Publish `tfplan`, `plan.txt`
and `.terraform.lock.hcl` as the pipeline artifact.

## `terraform-apply.yml`

```bash
set -euo pipefail
# same ARM_* exports and terraform init as in the plan
terraform apply -input=false -lock-timeout=5m tfplan
terraform output -json > outputs.json      # publish as artifact; do not print it
```

Apply **the downloaded plan file**, never a fresh `terraform apply`. If the state changed
since the plan, Terraform refuses the stale plan: run the pipeline again.

## Backend

- One storage account + container for the environment's state; key `dev.terraform.tfstate`.
- Enable blob versioning and soft delete on the storage account, and restrict network access.
- Authenticate with Entra ID (`use_azuread_auth = true`); the pipeline identity needs
  **Storage Blob Data Contributor** on the container.
- Create the backend outside this root (a bootstrap script or a separate root), never from the
  same state it stores.

## Pipeline identity permissions

- Contributor on the target subscription or resource groups (or a custom role).
- User Access Administrator / RBAC Administrator, scoped down, **only** if the code creates role assignments.
- Storage Blob Data Contributor on the state container.
- Read access to any existing resource used through `data` sources.

## Good practices

- Pin the Terraform version and install it with an official task or a verified checksum.
- `-input=false` and `TF_IN_AUTOMATION=true` always.
- Never use `-auto-approve` on a fresh plan; always apply the reviewed artifact.
- Do not print outputs or plans that contain sensitive values to the logs.
- Run a scheduled `terraform plan -detailed-exitcode` to detect drift.
