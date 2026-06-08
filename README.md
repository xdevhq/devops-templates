# DevOps Templates

Reusable GitHub Actions templates for:

- container build and push to GHCR
- deploy to Azure Web App for Containers or Azure Container Apps
- NuGet package publish
- npm package publish with GitHub Packages
- GitHub Actions artifact and GHCR package cleanup

The design is intentionally opinionated and proven across multiple .NET services and package repositories.

## What This Repo Provides

- `.github/workflows/build.yml`: reusable container build and push workflow
- `.github/workflows/deploy.yml`: reusable Azure container deploy workflow
- `.github/workflows/nuget.yml`: reusable NuGet pack and publish workflow
- `.github/workflows/npm.yml`: reusable npm package publish workflow
- `.github/workflows/cleanup-artifacts.yml`: reusable GitHub Actions artifact cleanup workflow
- `.github/workflows/cleanup-ghcr.yml`: reusable GHCR version cleanup workflow
- `.github/actions/*`: composite actions used by the workflows
- `.github/containerfiles/*`: template Containerfiles for `node` and `dotnet`

## Documentation

- [Workflow contracts](docs/workflows.md): inputs, outputs, secrets, permissions, defaults, and allowed values.
- [Required secrets and variables](docs/secrets-and-variables.md): consuming-repository secrets and environment variables.
- [Container builds](docs/container-builds.md): build behavior, template defaults, GHCR push behavior, and build arguments.
- [Deployments](docs/deployments.md): Azure Web App and Azure Container Apps deployment guidance.
- [Package publishing](docs/package-publishing.md): NuGet and npm publishing guidance.
- [Cleanup](docs/cleanup.md): artifact cleanup, GHCR cleanup, and retention strategy.
- [Repository access](docs/repository-access.md): access setup when this templates repository is private.

## Core Behavior

- Build workflow requires consuming repos to set `template` explicitly.
- Build workflow supports optional `context`, `containerfile`, and `build_args`.
- Build workflow builds and pushes directly to GHCR.
- Deploy workflow supports Azure Web App for Containers and Azure Container Apps.
- NuGet workflow uses the consuming repo's `NuGet.Config` for restore sources.
- npm workflow uses the consuming repo's `.npmrc` for registry and auth policy.
- Cleanup workflows keep Actions artifacts and GHCR package history bounded.

## Quick Starts

### Node Container Build

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

### .NET Container Build

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

See [Container builds](docs/container-builds.md) for template defaults, private feed restore, and build argument guidance.

### Deploy

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

See [Deployments](docs/deployments.md) for target-specific variables, secrets, and port behavior.

### NuGet Publish

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

### npm Publish

```yaml
name: Publish NPM

on:
  push:
    branches: [main]
    tags: ["v*"]

jobs:
  publish:
    permissions:
      contents: read
      packages: write
    uses: <owner>/<templates-repo>/.github/workflows/npm.yml@main
    with:
      package_path: .
      package_manager: pnpm
```

See [Package publishing](docs/package-publishing.md) for registry, token, and package manager guidance.

### Cleanup

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

See [Cleanup](docs/cleanup.md) for Actions artifact cleanup, GHCR cleanup, and retention guidance.

## Guiding Principle

Keep workflows boring, explicit, and easy to extend.
