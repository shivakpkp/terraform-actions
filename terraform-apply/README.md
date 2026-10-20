# Terraform Apply

A composite GitHub Action that sets up Terraform, initializes the backend, restores the provider cache, downloads a saved plan, and applies it without an interactive prompt.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `working_directory` | Yes | Directory containing the Terraform configuration. |
| `terraform_version` | Yes | Terraform CLI version. Use the same version as the plan job. |
| `plan_artifact_name` | Yes | Artifact name and the saved plan filename inside it. |

## Requirements

- A previous job in the same workflow run must upload the saved plan artifact.
- The artifact name and plan filename must match. For example, upload a file named `tfplan` as an artifact named `tfplan`.
- Provide cloud credentials and backend environment variables required by the setup action.
- Use the same Terraform configuration, provider lock file, and backend as the plan job.

## Example

Add this job to a workflow with an existing `plan` job that uploads `tfplan`:

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
        uses: shivakpkp/terraform-actions/terraform-apply@v1.0.0
        with:
          working_directory: environment/dev
          terraform_version: '1.5.7'
          plan_artifact_name: tfplan
```

Adjust the action folder, Terraform directory, and version to match your repository. If your secrets and variables belong to a GitHub Environment, add `environment: <your-environment>` to the job.

## Outputs

This action does not define any outputs. Terraform prints the apply result in the workflow logs; the step fails if the apply command fails.
