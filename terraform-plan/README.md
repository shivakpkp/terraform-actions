# Terraform Plan Action

A composite GitHub Action that initializes Terraform, creates a plan, detects changes, and uploads the saved plan as an artifact.

## Usage

Add this step to your workflow job. The example pins a reviewed commit. Replace `infrastructure` with your Terraform root directory.

```yaml
- name: Terraform plan
  id: plan
  uses: shivakpkp/terraform-actions/terraform-plan@c2aff50610321fce2f47d31653652e46d23a64a3
  with:
    working_directory: infrastructure
    terraform_version: "1.5.7"
```

Provide cloud credentials and backend settings through the calling job's environment, as required by the setup/init action.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `working_directory` | Yes | Path to the Terraform root directory. |
| `terraform_version` | Yes | Terraform CLI version to install. |

## Outputs

| Output | Description |
| --- | --- |
| `has-changes` | Returns `'true'` when the plan contains changes; otherwise `'false'`. |
| `artifact_name` | Name of the uploaded artifact: `tfplan`. |

Use the output in subsequent steps:

```yaml
- name: Changes detected
  if: steps.plan.outputs.has-changes == 'true'
  run: echo "Terraform changes are ready for review."
```

## Plan Artifact

The plan is saved to `<working_directory>/tfplan` and uploaded as the `tfplan` artifact for use in a later apply job. Planning errors fail the action; no infrastructure changes are applied.
