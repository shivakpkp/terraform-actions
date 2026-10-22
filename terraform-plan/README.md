# Terraform Plan Action

A composite GitHub Action that initializes Terraform, creates a plan, detects changes, and uploads the saved plan as an artifact.

## Usage

Add this step to your workflow job. The example pins a reviewed commit. Replace `infrastructure` with your Terraform root directory.

```yaml
- name: Terraform plan
  id: plan
  uses: shivakpkp/terraform-actions/terraform-plan@1e84ec37dda9edc4f44fcf21c81903ac98764de7
  with:
    working_directory: infrastructure
    terraform_version: "1.5.7"
    plan_artifact_name: plan-dev
```

Provide cloud credentials and backend settings through the calling job's environment, as required by the setup/init action.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `working_directory` | Yes | Path to the Terraform root directory. |
| `terraform_version` | Yes | Terraform CLI version to install. |
| `plan_artifact_name` | No | Uploaded artifact name. Defaults to `tfplan`. |

## Outputs

| Output | Description |
| --- | --- |
| `has-changes` | Returns `'true'` when the plan contains changes; otherwise `'false'`. |
| `artifact_name` | Selected artifact name, such as `plan-dev`; defaults to `tfplan`. |

Use the output in subsequent steps:

```yaml
- name: Changes detected
  if: steps.plan.outputs.has-changes == 'true'
  run: echo "Terraform changes are ready for review."
```

## Plan Artifact

The saved file is always `<working_directory>/tfplan`. Its artifact name is configurable and defaults to `tfplan`. Use unique names such as `plan-dev` and `plan-prod` for multiple planning jobs in one workflow run.

When applying the example above, pass `plan_artifact_name: plan-dev` and `plan_file_name: tfplan` to the apply action. The artifact name identifies the package; the filename identifies the plan inside it. Planning errors fail the action; no infrastructure changes are applied.
