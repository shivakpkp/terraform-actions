# Terraform Destroy

Creates a Terraform destroy plan and uploads it as the `tfplan-destroy` artifact for later use. **This action does not delete resources or apply the plan.**

## What it does

- Checks out your repository and sets up Terraform with provider caching.
- Initializes Terraform and displays backend, workspace, and tracked resources.
- Runs `terraform plan -destroy` and uploads the saved plan.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `working_directory` | Yes | Directory containing your Terraform configuration. |
| `terraform_version` | Yes | Terraform CLI version to install. |

## Example

Add this step to your workflow job:

```yaml
- name: Create destroy plan
  id: destroy_plan
  uses: shivakpkp/terraform-actions/terraform-destroy@c2aff50610321fce2f47d31653652e46d23a64a3
  with:
    working_directory: environment/dev
    terraform_version: "1.5.7"
```

Replace `environment/dev` with your Terraform root directory. The example pins a reviewed commit.

Configure Azure authentication in the calling job using `ARM_CLIENT_ID`, `ARM_CLIENT_SECRET`, `ARM_TENANT_ID`, and `ARM_SUBSCRIPTION_ID`. The initialization action also expects `STATE_RESOURCE_GROUP_NAME`, `STATE_STORAGE_ACCOUNT`, `STATE_CONTAINER`, and `STATE_KEY` as environment variables for your Azure backend.

## Artifact

Download `tfplan-destroy` from the workflow run to access the saved destroy plan. Applying it requires a separate Terraform apply step.

## Outputs

| Output | Description |
| --- | --- |
| `has_changes` | Returns `'true'` when resources would change; otherwise `'false'`. |
| `archive_name` | Name of the uploaded artifact: `tfplan-destroy`. |

For a later apply job, expose the step outputs on the destroy-plan job:

```yaml
outputs:
  has_changes: ${{ steps.destroy_plan.outputs.has_changes }}
  artifact_name: ${{ steps.destroy_plan.outputs.archive_name }}
```

In the apply job, set `needs: destroy-plan`, use `if: needs.destroy-plan.outputs.has_changes == 'true'`, and pass `${{ needs.destroy-plan.outputs.artifact_name }}` to the apply action's `plan_artifact_name` input. Configure an approval environment on that job before applying a destroy plan.
