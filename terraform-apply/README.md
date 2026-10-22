# Terraform Apply

A composite GitHub Action that sets up Terraform, initializes the backend, restores the provider cache, downloads a saved plan, and applies it without an interactive prompt.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `working_directory` | Yes | Directory containing the Terraform configuration. |
| `terraform_version` | Yes | Terraform CLI version. Use the same version as the plan job. |
| `plan_artifact_name` | Yes | Name of the artifact to download, such as `plan-dev`. |
| `plan_file_name` | No | Saved plan filename relative to `working_directory`, such as `tfplan`. If omitted or empty, uses `plan_artifact_name` for compatibility. |

## Requirements

- A previous job in the same workflow run must upload the saved plan artifact.
- Set `plan_file_name` when the filename differs from the artifact name. The plan action saves a file named `tfplan` regardless of its artifact name.
- Provide cloud credentials and backend environment variables required by the setup action.
- Use the same Terraform configuration, provider lock file, and backend as the plan job.

## Example

Add this job to a workflow with an existing `plan` job that uploads an artifact named `plan-dev` containing `tfplan`:

```yaml
jobs:
  # Your existing plan job goes here.

  apply:
    needs: plan
    runs-on: ubuntu-24.04
    permissions:
      contents: read
    env:
      ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
      STATE_RESOURCE_GROUP_NAME: ${{ vars.STATE_RESOURCE_GROUP_NAME }}
      STATE_STORAGE_ACCOUNT: ${{ vars.STATE_STORAGE_ACCOUNT }}
      STATE_CONTAINER: ${{ vars.STATE_CONTAINER }}
      STATE_KEY: ${{ vars.STATE_KEY }}
    steps:
      - name: Apply Terraform plan
        uses: shivakpkp/terraform-actions/terraform-apply@1e84ec37dda9edc4f44fcf21c81903ac98764de7
        with:
          working_directory: environment/dev
          terraform_version: '1.5.7'
          plan_artifact_name: plan-dev
          plan_file_name: tfplan
```

Adjust the action folder, Terraform directory, and version to match your repository. If your secrets and variables belong to a GitHub Environment, add `environment: <your-environment>` to the job.

## Outputs

This action does not define any outputs. Terraform prints the apply result in the workflow logs; the step fails if the apply command fails.

## Compatibility

Existing callers using only `plan_artifact_name: tfplan` continue to apply `tfplan`. Callers using `plan_artifact_name: tfplan-destroy` continue to apply `tfplan-destroy`. An explicit `plan_file_name` overrides this fallback.

The action checks that the selected file exists before invoking Terraform and passes the filename as a quoted shell argument.
