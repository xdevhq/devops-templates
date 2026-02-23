# DevOps Templates

Reusable GitHub Actions templates for:

- container build + push to GHCR
- deploy to Azure Web App for Containers or Azure Container Apps
- NuGet package publish

The design is intentionally opinionated and proven across multiple .NET services and package repos.

## What this repo provides

- `.github/workflows/build.yml`: reusable container build + push workflow
- `.github/workflows/deploy.yml`: reusable Azure container deploy workflow (Web App or Container Apps)
- `.github/workflows/nuget.yml`: reusable NuGet pack + publish workflow
- `.github/workflows/cleanup-artifacts.yml`: reusable GitHub Actions artifact cleanup workflow
- `.github/workflows/cleanup-ghcr.yml`: reusable GHCR version cleanup workflow
- `.github/actions/*`: small composite actions used by the workflows
- `.github/containerfiles/*`: template Containerfiles (`node`, `dotnet`)

## Core behavior

- Build workflow requires consuming repos to set `template` explicitly.
- Build workflow supports optional `context`, `containerfile`, and `build_args`.
- Build workflow builds and pushes directly (no image tar artifact upload/download).
- Deploy workflow supports Azure Web App for Containers and Azure Container Apps (both with GHCR images).
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
      deploy_target: webapp
      image_name: ${{ github.event.repository.name }}
      image_tag: ${{ github.event.workflow_run.head_sha }}
```

Set `deploy_target` to:
- `webapp` (default): requires `AZURE_WEBAPP_NAME`
- `containerapp`: requires `AZURE_CONTAINERAPP_NAME`

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
- environment secret in consuming repo environment:
  - `AZURE_CREDENTIALS`
- repository-level or environment-level secrets (consuming repo):
  - `GHCR_USERNAME`
  - `GHCR_PASSWORD`
- environment variables in consuming repo environment:
  - `AZURE_WEBAPP_NAME` (for `deploy_target: webapp`)
  - `AZURE_CONTAINERAPP_NAME` (for `deploy_target: containerapp`)
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
- `deploy_target` (optional, default `webapp`; allowed values: `webapp`, `containerapp`)
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

Recommended baseline:

- scope RBAC at resource group level (or narrower) where possible
- create one `AZURE_CREDENTIALS` secret per GitHub Environment (target)

Create the App Registration / service principal once:

```bash
az ad app create \
  --display-name "sp-gha-deploy-<ENV_GROUP>" \
  --query appId \
  --output tsv
```

```bash
az ad sp create \
  --id "<APP_ID>"
```

```bash
az account show \
  --query tenantId \
  --output tsv
```

Use `appId` as `<APP_ID>`. Then assign RBAC per target subscription/resource group:

```bash
az role assignment create \
  --assignee "<APP_ID>" \
  --role "Contributor" \
  --scope "/subscriptions/<SUB_ID>/resourceGroups/<RG_NAME>"
```

Repeat the role assignment command for each target scope.

Create one `AZURE_CREDENTIALS` JSON value per deployment target by running this command once per target:

```bash
az ad app credential reset \
  --id "<APP_ID>" \
  --append \
  --display-name "github-<RG>" \
  --years 1 \
  --query "{clientId:'<APP_ID>',clientSecret:password,subscriptionId:'<SUB_ID>',tenantId:'<TENANT_ID>'}" \
  --output json
```

Store one `AZURE_CREDENTIALS` secret per GitHub Environment:

```json
{"clientId":"<APP_ID>","clientSecret":"<SECRET_FROM_COMMAND>","subscriptionId":"<SUB_ID>","tenantId":"<TENANT_ID>"}
```

For each target, set in that GitHub Environment:

- secret: `AZURE_CREDENTIALS`
- vars: `AZURE_RESOURCE_GROUP` and one of:
  - `AZURE_WEBAPP_NAME` (for Web App deploy target)
  - `AZURE_CONTAINERAPP_NAME` (for Container Apps deploy target)

RBAC note: all secrets on the same App Registration represent the same identity and permissions. Grant RBAC access for all target resources (Web Apps and/or Container Apps, or at resource group scope). If you need different permissions per app/environment, use separate service principals. If targets are across different tenants, use separate identities per tenant.

## Guiding principle

Keep workflows boring, explicit, and easy to extend.
