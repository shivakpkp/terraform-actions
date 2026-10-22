# Terraform Destroy

Creates a Terraform destroy plan and uploads it under a configurable artifact name (default: `tfplan-destroy`) for later use. **This action does not delete resources or apply the plan.**

## What it does

- Checks out your repository and sets up Terraform with provider caching.
- Initializes Terraform and displays backend, workspace, and tracked resources.
- Runs `terraform plan -destroy` and uploads the saved plan.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `working_directory` | Yes | Directory containing your Terraform configuration. |
| `terraform_version` | Yes | Terraform CLI version to install. |
| `plan_artifact_name` | No | Artifact name. Defaults to `tfplan-destroy`. |

## Example

Add this step to your workflow job:

```yaml
- name: Create destroy plan
  id: destroy_plan
  uses: shivakpkp/terraform-actions/terraform-destroy@88d4f8315b18253db89e1381d9b7d8f5178abfb5
  with:
    working_directory: environment/dev
    terraform_version: "1.5.7"
    plan_artifact_name: destroy-dev
```

Replace `environment/dev` with your Terraform root directory. The example pins a reviewed commit.

Configure Azure authentication in the calling job using `ARM_CLIENT_ID`, `ARM_CLIENT_SECRET`, `ARM_TENANT_ID`, and `ARM_SUBSCRIPTION_ID`. The initialization action also expects `STATE_RESOURCE_GROUP_NAME`, `STATE_STORAGE_ACCOUNT`, `STATE_CONTAINER`, and `STATE_KEY` as environment variables for your Azure backend.

## Artifact

The saved file is always `tfplan-destroy`. The example uploads it as the `destroy-dev` artifact. Use unique artifact names for multiple environments in the same run. Applying the saved plan requires a separate Terraform apply step.

## Outputs

| Output | Description |
| --- | --- |
| `has_changes` | Returns `'true'` when resources would change; otherwise `'false'`. |
| `archive_name` | Selected artifact name, such as `destroy-dev`; defaults to `tfplan-destroy`. |

## Pass the plan to an apply job

The following jobs belong in a manually triggered workflow. Set the Azure credentials and backend variables described above for both jobs. Use the same Terraform configuration, version, provider lock file, and backend for planning and applying.

```yaml
jobs:
  destroy-plan:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
    outputs:
      has_changes: ${{ steps.destroy_plan.outputs.has_changes }}
      artifact_name: ${{ steps.destroy_plan.outputs.archive_name }}
    steps:
      - name: Create destroy plan
        id: destroy_plan
        uses: shivakpkp/terraform-actions/terraform-destroy@88d4f8315b18253db89e1381d9b7d8f5178abfb5
        with:
          working_directory: environment/dev
          terraform_version: "1.5.7"
          plan_artifact_name: destroy-dev

  destroy-apply:
    needs: destroy-plan
    if: needs.destroy-plan.outputs.has_changes == 'true'
    runs-on: ubuntu-24.04
    environment: dev
    permissions:
      contents: read
    steps:
      - name: Apply saved destroy plan
        uses: shivakpkp/terraform-actions/terraform-apply@88d4f8315b18253db89e1381d9b7d8f5178abfb5
        with:
          working_directory: environment/dev
          terraform_version: "1.5.7"
          plan_artifact_name: ${{ needs.destroy-plan.outputs.artifact_name }}
          plan_file_name: tfplan-destroy
```

Configure required reviewers on the `dev` environment to require approval before deletion; the environment name alone does not enable approval. If credentials are environment-scoped, configure the appropriate environment on each job that needs them. Use workflow concurrency shared with deployment workflows targeting the same state.

## Compatibility

Existing callers may omit `plan_artifact_name` and continue to receive the `tfplan-destroy` artifact. The existing outputs `has_changes` and `archive_name` are preserved. When using a custom artifact name, pass `plan_file_name: tfplan-destroy` to the apply action.

Use a commit or release containing these changes for the new input. Older release tags do not gain new inputs automatically.
