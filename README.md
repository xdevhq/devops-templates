# Minimal Release Pipeline (v0.1)

This repo provides a small, componentized GitHub Actions release pipeline design for containerized apps deployed to Azure Web App for Containers (Linux). It is intentionally simple and language-agnostic.

## How it composes

Entry points → stages → jobs → steps

- `.github/workflows/release.yml`: release entry point
- `.github/workflows/stages/*`: stage composition
- `.github/workflows/jobs/*`: job definitions
- `.github/workflows/steps/*`: small, single-purpose steps (composite actions)

## Required secrets

- `GITHUB_TOKEN`: for GHCR push (provided by GitHub)
- `AZURE_CREDENTIALS`: Azure service principal JSON for `azure/login` (environment secret)

## Consume from another repo

```yaml
permissions:
  contents: read
  packages: write

jobs:
  release:
    uses: alexhovy/devops-templates/.github/workflows/release.yml@main
    secrets: inherit
```

## Environment config in the consuming repo

- Create a GitHub Environment named `prod`.
- Set `AZURE_WEBAPP_NAME` as an environment variable (not secret).
- Set `AZURE_CREDENTIALS` as an environment secret.

## Get AZURE_CREDENTIALS (service principal JSON)

Use Azure CLI to create a service principal scoped to the target Web App (least privilege).

If you need to sign in with a specific tenant:

```bash
az login --tenant <TENANT_ID> --use-device-code
```

Generate the Azure Credentials:

```bash
az ad sp create-for-rbac --name "gh-actions-webapp-deploy" --role "Contributor" --scopes "/subscriptions/<SUB_ID>/resourceGroups/<RG_NAME>/providers/Microsoft.Web/sites/<WEBAPP_NAME>" --query "{clientId:appId, clientSecret:password, tenantId:tenant, subscriptionId:'<SUB_ID>'}" --output json > azure-credentials.json
```

Save the JSON as the `AZURE_CREDENTIALS` secret in the `prod` environment.

## GITHUB_TOKEN (GHCR push)

`GITHUB_TOKEN` is automatically provided by GitHub Actions. Ensure the consuming repo’s workflow has permissions to write packages:

```yaml
permissions:
  contents: read
  packages: write
```

Note: avoid defining custom variables or secrets that start with `GITHUB_`, as GitHub reserves that prefix.


## Notes

- This is a minimal, design-focused layout. GitHub Actions only loads workflows from `.github/workflows`, and composite actions require an `action.yml` inside a directory.
- Podman is used for build/tag/push. No Docker.
- Azure logic is isolated to deploy components.

## Guiding sentence

This pipeline should feel boring, obvious, and easy to extend.
