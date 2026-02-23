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
- `.github/workflows/cleanup-artifacts.yml`: reusable GitHub Actions artifact cleanup workflow
- `.github/workflows/cleanup-ghcr.yml`: reusable GHCR version cleanup workflow
- `.github/actions/*`: small composite actions used by the workflows
- `.github/containerfiles/*`: template Containerfiles (`node`, `dotnet`)

## Core behavior

- Build workflow requires consuming repos to set `template` explicitly.
- Build workflow supports optional `context`, `containerfile`, and `build_args`.
- Build workflow builds and pushes directly (no image tar artifact upload/download).
- Deploy workflow is Azure Web App + GHCR focused.
- NuGet workflow uses the consuming repo's `NuGet.Config` for restore sources.
- NuGet workflow defaults publish target to GitHub Packages for current owner.

## Storage and retention strategy

- Do not store container image tar files in GitHub Actions artifacts.
- Treat GHCR as the source of truth for runtime images.
- Keep a rollback window in GHCR (for example 20 versions), not infinite history.
- Run scheduled cleanup for:
  - old GitHub Actions artifacts
  - old GHCR container versions

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
      build_args: --build-arg APP_PROJECT=src/<path-to-api>.csproj --build-arg APP_DLL=<app-name>.dll
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Notes:
- Set `APP_PROJECT` to the .csproj you want containerized.
- Set `APP_DLL` to the published app assembly name (for example `Platform.Gateway.Api.dll`).
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
    secrets: inherit
    with:
      deploy_env: prod
      image_name: ${{ github.event.repository.name }}
      image_tag: ${{ github.event.workflow_run.head_sha }}
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

## Quick start: Cleanup old GitHub Actions artifacts

```yaml
name: Cleanup Artifacts

on:
  schedule:
    - cron: "0 3 * * *"
  workflow_dispatch:

jobs:
  cleanup:
    permissions:
      actions: write
      contents: read
    uses: <owner>/<templates-repo>/.github/workflows/cleanup-artifacts.yml@main
    with:
      keep_latest_count: 20
```

## Quick start: Cleanup old GHCR versions

```yaml
name: Cleanup GHCR

on:
  schedule:
    - cron: "30 3 * * *"
  workflow_dispatch:

jobs:
  cleanup:
    permissions:
      contents: read
      packages: write
    uses: <owner>/<templates-repo>/.github/workflows/cleanup-ghcr.yml@main
    with:
      package_name: ${{ github.event.repository.name }}
      min_versions_to_keep: 20
      delete_only_untagged: false
```

## Required secrets and variables

Build/push:
- `GHCR_TOKEN` (usually `${{ secrets.GITHUB_TOKEN }}` in consuming repo)

Deploy:
- Caller workflow must include `secrets: inherit` when invoking the reusable deploy workflow
- environment secrets in consuming repo environment:
  - `AZURE_CREDENTIALS`
  - `GHCR_USERNAME`
  - `GHCR_PASSWORD`
- environment variables in consuming repo environment:
  - `AZURE_WEBAPP_NAME`
  - `AZURE_RESOURCE_GROUP`

NuGet:
- `NUGET_API_KEY` (optional, only for non-default feeds)

Cleanup:
- no additional secrets required when using `GITHUB_TOKEN`

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
- none in `workflow_call` contract (caller should use `secrets: inherit`)

### `.github/workflows/nuget.yml`

Inputs:
- `project_path` (required)
- `dotnet_version` (optional, default `8.0.x`)
- `configuration` (optional, default `Release`)
- `package_version` (optional)
- `source_url` (optional publish target override)

Secrets:
- `NUGET_API_KEY` (optional)

### `.github/workflows/cleanup-artifacts.yml`

Inputs:
- `keep_latest_count` (optional, default `20`)
- `dry_run` (optional, default `false`)

Secrets:
- none

### `.github/workflows/cleanup-ghcr.yml`

Inputs:
- `package_name` (required)
- `min_versions_to_keep` (optional, default `20`)
- `delete_only_untagged` (optional, default `false`)

Secrets:
- none

## Repository access setup

If this templates repo is private, allow other repos to call its reusable workflows:

- Settings -> Actions -> General -> Access
- Enable access from the repositories that will consume these templates

## Azure credentials helper

Create a service principal scoped to a Web App:

```bash
az ad sp create-for-rbac --name "gh-actions-webapp-deploy" --role "Contributor" --scopes "/subscriptions/<SUB_ID>/resourceGroups/<RG_NAME>/providers/Microsoft.Web/sites/<WEBAPP_NAME>" --query "{clientId:appId, clientSecret:password, tenantId:tenant, subscriptionId:'<SUB_ID>'}" --output json > azure-credentials.json
```

Store the JSON as `AZURE_CREDENTIALS` secret in each consuming GitHub Environment used for deploys.

## Guiding principle

Keep workflows boring, explicit, and easy to extend.
