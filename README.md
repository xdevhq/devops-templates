# Minimal Build + Deploy Pipeline (v0.1)

This repo provides a small, componentized GitHub Actions build + deploy design for containerized apps deployed to Azure Web App for Containers (Linux). It is intentionally simple and language-agnostic.

## How it composes

Entry points → jobs → steps

- `.github/workflows/build.yml`: build + push entry point
- `.github/workflows/deploy.yml`: deploy entry point
- `.github/workflows/nuget.yml`: NuGet pack + publish entry point
- `.github/actions/*`: small, single-purpose steps (composite actions)
- `.github/containerfiles/*`: reusable Containerfile templates used by build steps

Build step expects a template name (e.g., `node`) to select a Containerfile from `.github/containerfiles/<template>/Containerfile`.

## Required secrets

- `GHCR_TOKEN`: for GHCR push (use `${{ secrets.GITHUB_TOKEN }}` from the consuming repo)
- `AZURE_CREDENTIALS`: Azure service principal JSON for `azure/login` (repo secret)
- `GHCR_USERNAME`: GitHub username for GHCR pull (repo secret)
- `GHCR_PASSWORD`: GitHub PAT with `read:packages` for GHCR pull (repo secret)
- `NUGET_API_KEY` (optional): API key override when publishing outside GitHub Packages

## Consume from another repo

```yaml
permissions:
  contents: read
  packages: write

jobs:
  build:
    uses: <owner>/<templates-repo>/.github/workflows/build.yml@main
    with:
      image_name: ${{ github.event.repository.name }}
      image_tag: ${{ github.sha }}
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}

```

NuGet package entry point:

```yaml
name: Publish NuGet

on:
  push:
    branches: [main]
    tags: ["v*"]

jobs:
  publish:
    permissions:
      contents: read
      packages: write
    uses: <owner>/<templates-repo>/.github/workflows/nuget.yml@main
    with:
      project_path: ./src/<path-to-project>.csproj
```

Set `project_path` to your repository's `.csproj` path.

Restore source configuration should live in the consuming repo's `NuGet.Config`.

Recommended pattern (build once, deploy many) uses `workflow_run`:

```yaml
on:
  push:
    branches: [main]

jobs:
  build:
    uses: <owner>/<templates-repo>/.github/workflows/build.yml@main
    with:
      image_name: ${{ github.event.repository.name }}
      image_tag: ${{ github.sha }}
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

```yaml
name: Deploy Prod

on:
  workflow_run:
    workflows: ["Build"]
    types: [completed]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: <owner>/<templates-repo>/.github/workflows/deploy.yml@main
    with:
      deploy_env: prod
      image_name: ${{ github.event.repository.name }}
      image_tag: ${{ github.event.workflow_run.head_sha }}
    secrets:
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS_PROD }}
      GHCR_USERNAME: ${{ secrets.GHCR_USERNAME }}
      GHCR_PASSWORD: ${{ secrets.GHCR_PASSWORD }}
```

## Environment config in the consuming repo

- Create GitHub Environments as needed (e.g., `dev`, `test`, `prod`).
- Set `AZURE_WEBAPP_NAME` as an environment variable (not secret).
- Set `AZURE_RESOURCE_GROUP` as an environment variable (not secret).
- Set `AZURE_CREDENTIALS` as a repo secret (reusable workflows require explicit secrets).
- To gate production, add required reviewers on the `prod` environment in GitHub.

## Template repo access

If this repo is private, allow other repos to use its reusable workflows:

- Settings → Actions → General → Access → "Accessible from repositories owned by the user 'username'"

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

Save the JSON as the `AZURE_CREDENTIALS` repo secret.

## GITHUB_TOKEN (GHCR push)

`GITHUB_TOKEN` is automatically provided by GitHub Actions. Ensure the consuming repo’s workflow has permissions to write packages:

```yaml
permissions:
  contents: read
  packages: write
```

Note: avoid defining custom variables or secrets that start with `GITHUB_`, as GitHub reserves that prefix.

## NuGet publish options

Defaults in `.github/workflows/nuget.yml`:

- `dotnet_version`: `8.0.x`
- `configuration`: `Release`
- restore sources: from consuming repo `NuGet.Config`
- `source_url`: publish target; defaults to GitHub Packages for the current repository owner (`https://nuget.pkg.github.com/<owner>/index.json`)
- publish token: `${{ github.token }}`

To publish to nuget.org instead of GitHub Packages:

```yaml
with:
  project_path: ./src/<path-to-project>.csproj
  source_url: https://api.nuget.org/v3/index.json
secrets:
  NUGET_API_KEY: ${{ secrets.NUGET_API_KEY }}
```

To publish to a custom private feed:

```yaml
with:
  project_path: ./src/<path-to-project>.csproj
  source_url: https://<feed-url>/v3/index.json
secrets:
  NUGET_API_KEY: ${{ secrets.<feed-api-key> }}
```


## Notes

- This is a minimal, design-focused layout. GitHub Actions only loads workflows from `.github/workflows`, and composite actions require an `action.yml` inside a directory.
- Podman is used for build/tag/push. No Docker.
- Azure logic is isolated to deploy components.

## Guiding sentence

This pipeline should feel boring, obvious, and easy to extend.
