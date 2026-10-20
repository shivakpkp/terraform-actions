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
  uses: shivakpkp/terraform-actions/terraform-destroy@v1.0.0
  with:
    working_directory: environment/dev
    terraform_version: "1.5.7"
```

Replace `terraform-destroy` with your action's folder name if different.

Configure Azure authentication in the calling job using `ARM_CLIENT_ID`, `ARM_CLIENT_SECRET`, `ARM_TENANT_ID`, and `ARM_SUBSCRIPTION_ID`. The initialization action also expects `STATE_RESOURCE_GROUP_NAME`, `STATE_STORAGE_ACCOUNT`, `STATE_CONTAINER`, and `STATE_KEY` as environment variables for your Azure backend.

## Artifact

Download `tfplan-destroy` from the workflow run to access the saved destroy plan. Applying it requires a separate Terraform apply step.
