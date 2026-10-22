# Terraform Actions

Reusable GitHub composite actions for Terraform with an Azure Storage backend.

## Actions

| Action | Purpose | Outputs |
| --- | --- | --- |
| [Setup, init, and cache](terraform-init-cache-setup/ReadMe.md) | Install Terraform, cache providers, and initialize the Azure backend. | None |
| [Plan](terraform-plan/README.md) | Create and upload a saved plan. | `has-changes`, `artifact_name` |
| [Apply](terraform-apply/README.md) | Download and apply a saved plan. | None |
| [Destroy plan](terraform-destroy/README.md) | Create and upload a destroy plan for a separate apply job. | `has_changes`, `archive_name` |

## Requirements

- A Terraform root module with an `azurerm` backend and a committed `.terraform.lock.hcl`.
- An existing Azure Storage account and container for remote state.
- Azure credentials with access to the state container and the resources being managed.
- A Linux runner with Bash. The example uses `ubuntu-24.04`.

See the [setup documentation](terraform-init-cache-setup/ReadMe.md) for authentication and backend details.

## Example: plan and apply

Save this workflow in your infrastructure repository. Set the four `ARM_*` repository secrets and four `STATE_*` repository variables below. Replace `environment/dev` and the Terraform version with your project's values.

The example pins these actions to a reviewed commit. Use the same action revision, Terraform version, configuration, lock file, and backend for both jobs.

```yaml
name: Terraform Plan and Apply

on:
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: terraform-dev
  cancel-in-progress: false

env:
  TF_IN_AUTOMATION: "true"
  TF_INPUT: "false"
  ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
  ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
  ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
  STATE_RESOURCE_GROUP_NAME: ${{ vars.STATE_RESOURCE_GROUP_NAME }}
  STATE_STORAGE_ACCOUNT: ${{ vars.STATE_STORAGE_ACCOUNT }}
  STATE_CONTAINER: ${{ vars.STATE_CONTAINER }}
  STATE_KEY: ${{ vars.STATE_KEY }}

jobs:
  plan:
    runs-on: ubuntu-24.04
    outputs:
      has_changes: ${{ steps.plan.outputs['has-changes'] }}
      artifact_name: ${{ steps.plan.outputs.artifact_name }}
    steps:
      - name: Create plan
        id: plan
        uses: shivakpkp/terraform-actions/terraform-plan@1e84ec37dda9edc4f44fcf21c81903ac98764de7
        with:
          working_directory: environment/dev
          terraform_version: "1.5.7"
          plan_artifact_name: plan-dev

  apply:
    needs: plan
    if: needs.plan.outputs.has_changes == 'true'
    runs-on: ubuntu-24.04
    environment: dev
    steps:
      - name: Apply saved plan
        uses: shivakpkp/terraform-actions/terraform-apply@1e84ec37dda9edc4f44fcf21c81903ac98764de7
        with:
          working_directory: environment/dev
          terraform_version: "1.5.7"
          plan_artifact_name: ${{ needs.plan.outputs.artifact_name }}
          plan_file_name: tfplan
```

Create the `dev` GitHub Environment before running this example. Configure required reviewers if you want an approval before apply; an environment name alone does not enforce approval. If credentials are environment-scoped, configure the appropriate environment on each job that needs them.

Use a concurrency group shared by workflows targeting the same state. Plan and apply actions include checkout; the standalone setup action requires the caller to check out first.

## Saved plans

- Deployment planning saves `tfplan` and uploads it under `plan_artifact_name` (default: `tfplan`). Use unique artifact names for multiple environment jobs in the same run.
- Apply accepts a separate `plan_file_name`. When omitted, it uses the artifact name to preserve existing callers.
- Destroy planning still uses the fixed artifact and filename `tfplan-destroy`; its interface is unchanged.
- Saved plans may contain sensitive values. Restrict access to workflow artifacts and select an appropriate repository retention policy.
- Approval gates and workflow permissions are configured by the calling workflow.
