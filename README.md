# DevOps Templates

Reusable GitHub Actions templates for:

- container build + push to GHCR
- deploy to Azure Web App for Containers
- NuGet package publish

The design is intentionally opinionated and proven across multiple .NET services and package repos.

## What this repo provides

- `.github/workflows/build.yml`: reusable container build + push workflow
- `.github/workflows/deploy.yml`: reusable Azure Web App deploy workflow
- `.github/workflows/nuget.yml`: reusable NuGet pack + publish workflow
- `.github/actions/*`: small composite actions used by the workflows
- `.github/containerfiles/*`: template Containerfiles (`node`, `dotnet`)

## Core behavior

- Build workflow requires consuming repos to set `template` explicitly.
- Build workflow supports optional `context`, `containerfile`, and `build_args`.
- Deploy workflow is Azure Web App + GHCR focused.
- NuGet workflow uses the consuming repo's `NuGet.Config` for restore sources.
- NuGet workflow defaults publish target to GitHub Packages for current owner.

## Quick start: Node container build

```yaml
name: Build

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write

jobs:
  build:
    uses: <owner>/<templates-repo>/.github/workflows/build.yml@main
    with:
      image_name: ${{ github.event.repository.name }}
      image_tag: ${{ github.sha }}
      template: node
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Quick start: .NET container build

```yaml
name: Build

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write

jobs:
  build:
    uses: <owner>/<templates-repo>/.github/workflows/build.yml@main
    with:
      image_name: ${{ github.event.repository.name }}
      image_tag: ${{ github.sha }}
      template: dotnet
      build_args: --build-arg APP_PROJECT=src/<path-to-api>.csproj
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Notes:
- Set `APP_PROJECT` to the .csproj you want containerized.
- If private feed restore is needed during image build, pass auth build args:
  - `--build-arg GITHUB_USERNAME=${{ github.actor }}`
  - `--build-arg GITHUB_PACKAGES_TOKEN=${{ secrets.GITHUB_TOKEN }}`

## Quick start: Deploy (build once, deploy on success)

Use one of the build quick-start workflows above, then add this deploy workflow:

```yaml
name: Deploy

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

## Quick start: NuGet publish

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

NuGet notes:
- Restore source policy should live in the consuming repo `NuGet.Config`.
- Default publish target is GitHub Packages owner feed.
- Override publish target via `source_url`.
- Use `NUGET_API_KEY` only when publishing to feeds that require a non-GitHub token (for example nuget.org).

## Required secrets and variables

Build/push:
- `GHCR_TOKEN` (usually `${{ secrets.GITHUB_TOKEN }}` in consuming repo)

Deploy:
- `AZURE_CREDENTIALS`
- `GHCR_USERNAME`
- `GHCR_PASSWORD`
- environment variables in consuming repo environment:
  - `AZURE_WEBAPP_NAME`
  - `AZURE_RESOURCE_GROUP`

NuGet:
- `NUGET_API_KEY` (optional, only for non-default feeds)

## Reusable workflow contracts

### `.github/workflows/build.yml`

Inputs:
- `image_name` (required)
- `image_tag` (required)
- `template` (required)
- `context` (optional, default `.`)
- `containerfile` (optional, default `Containerfile`)
- `build_args` (optional, default empty)

Secrets:
- `GHCR_TOKEN` (required)

### `.github/workflows/deploy.yml`

Inputs:
- `deploy_env` (required)
- `image_name` (required)
- `image_tag` (required)

Secrets:
- `AZURE_CREDENTIALS` (required)
- `GHCR_USERNAME` (required)
- `GHCR_PASSWORD` (required)

### `.github/workflows/nuget.yml`

Inputs:
- `project_path` (required)
- `dotnet_version` (optional, default `8.0.x`)
- `configuration` (optional, default `Release`)
- `package_version` (optional)
- `source_url` (optional publish target override)

Secrets:
- `NUGET_API_KEY` (optional)

## Repository access setup

If this templates repo is private, allow other repos to call its reusable workflows:

- Settings -> Actions -> General -> Access
- Enable access from the repositories that will consume these templates

## Azure credentials helper

Create a service principal scoped to a Web App:

```bash
az ad sp create-for-rbac --name "gh-actions-webapp-deploy" --role "Contributor" --scopes "/subscriptions/<SUB_ID>/resourceGroups/<RG_NAME>/providers/Microsoft.Web/sites/<WEBAPP_NAME>" --query "{clientId:appId, clientSecret:password, tenantId:tenant, subscriptionId:'<SUB_ID>'}" --output json > azure-credentials.json
```

Store the JSON as `AZURE_CREDENTIALS` secret in the consuming repo.

## Guiding principle

Keep workflows boring, explicit, and easy to extend.
