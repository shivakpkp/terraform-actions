# Terraform Init Cache Setup

A reusable GitHub composite action that installs Terraform, caches provider binaries, and initializes a Terraform configuration with an Azure Storage backend.

## Features

- Installs the requested Terraform version with the Terraform wrapper disabled.
- Creates a provider cache directory.
- Restores and saves provider binaries using GitHub Actions caching.
- Checks that the required Azure backend configuration variables are available.
- Runs `terraform init` in the specified working directory.

This action does not run `terraform plan` or `terraform apply`.

## Prerequisites

Before calling this action:

1. Check out the repository containing your Terraform configuration.
2. Configure an `azurerm` backend in your Terraform root module.
3. Create the Azure Storage account and blob container used for remote state.
4. Configure Azure authentication and grant the identity access to the backend.
5. Commit `.terraform.lock.hcl` for consistent provider selection and cache keys.

The calling repository must be allowed to use this action and its dependencies.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `terraform_version` | Yes | None | Terraform version to install. |
| `working_directory` | Yes | None | Terraform root directory, relative to the checked-out repository root. |
| `TF_PLUGIN_CACHE_DIR` | No | `${{ github.workspace }}/.terraform.d/plugin-cache` | Directory used to store cached provider binaries. Use an absolute path when overriding it. |

## Backend Configuration

Set the following environment variables in the calling workflow:

| Variable | Description |
|---|---|
| `STATE_RESOURCE_GROUP_NAME` | Resource group containing the state storage account. |
| `STATE_STORAGE_ACCOUNT` | Azure Storage account name. |
| `STATE_CONTAINER` | Blob container used for Terraform state. |
| `STATE_KEY` | State blob name, such as `dev/terraform.tfstate`. |

The action fails if any of these variables are missing or empty.

These checks validate configuration presence, not Azure credentials or permissions.

## Azure Authentication

Authentication must be configured by the calling workflow.

For service principal authentication using a client secret, provide:

| Environment variable | Description |
|---|---|
| `ARM_CLIENT_ID` | Service principal application ID. |
| `ARM_CLIENT_SECRET` | Service principal client secret. |
| `ARM_TENANT_ID` | Microsoft Entra tenant ID. |
| `ARM_SUBSCRIPTION_ID` | Azure subscription ID. |

Store credentials in GitHub secrets.

For Microsoft Entra authentication to Azure Storage, configure the backend as follows:

```hcl
terraform {
  backend "azurerm" {
    use_azuread_auth = true
  }
}
```

The identity needs blob data access to the state container, typically through
the `Storage Blob Data Contributor` role scoped to that container.

Additional permissions depend on the resources managed by subsequent Terraform commands.

The action does not run `az login` or configure OIDC automatically.

## Usage

Create a workflow in the repository containing your Terraform configuration:

```yaml
name: Terraform Init and Validate

on:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest

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

    steps:
      - name: Checkout Terraform configuration
        uses: actions/checkout@v4

      - name: Setup Terraform and initialize backend
        uses: YOUR_GITHUB_USERNAME/terraform-actions/terraform-init-cache-setup@v1
        with:
          terraform_version: "1.5.7"
          working_directory: environment/POC

      - name: Terraform validate
        working-directory: environment/POC
        run: terraform validate
```

Replace:

- `YOUR_GITHUB_USERNAME` with the owner of the action repository.
- `@v1` with an existing tag or commit SHA in that repository.
- `terraform_version` with the version required by your project.
- `working_directory` with your Terraform root directory.

Create the `v1` tag before using the example as written. For immutable references,
pin the action to a full commit SHA.

If your secrets and variables are stored in a GitHub Environment, add
`environment: YOUR_ENVIRONMENT` to the job.

### Custom Cache Directory

```yaml
- name: Setup Terraform and initialize backend
  uses: YOUR_GITHUB_USERNAME/terraform-actions/terraform-init-cache-setup@v1
  with:
    terraform_version: "1.5.7"
    working_directory: environment/POC
    TF_PLUGIN_CACHE_DIR: ${{ github.workspace }}/terraform-provider-cache
```

## How Caching Works

The cache key includes:

- Runner operating system.
- Runner architecture.
- Hash of `.terraform.lock.hcl` in the working directory.

On a cache miss, Terraform downloads the required providers. GitHub saves the
provider cache after a successful job.

On a matching cache hit, GitHub restores the provider binaries so Terraform
can reuse them during initialization.

Changing `.terraform.lock.hcl` changes the cache key. Cache availability also
depends on GitHub's cache scope and retention rules.

Only provider binaries are cached. Terraform state, downloaded modules, and the
working directory's `.terraform` folder are not cached by this action.

Caching does not skip `terraform init` or eliminate all network requests.

## Outputs

This action does not define custom outputs.

After successful execution, Terraform is installed and the specified working
directory is initialized for subsequent Terraform commands in the same job.

## Troubleshooting

| Issue | Check |
|---|---|
| Required backend variable is missing | Define all four `STATE_*` variables in the calling job or action step. |
| Azure authentication fails | Verify credentials, tenant, subscription, and backend access permissions. |
| Working directory does not exist | Check out the calling repository first and verify the directory path. |
| Providers are downloaded again | Check the cache restore result and whether the lock file or runner platform changed. |
| Action cannot be found | Verify the repository owner, action folder, referenced tag or SHA, and repository access settings. |