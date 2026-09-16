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
- `.github/containerfiles/*`: template Containerfiles for `node`, `dotnet`, and `python`

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
- Build workflow builds container images and can optionally push to GHCR.
- Consuming repos should keep a `.containerignore` in the build context used by the workflow.
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
  packages: read

jobs:
  build:
    uses: <owner>/<templates-repo>/.github/workflows/build.yml@main
    with:
      image_name: ${{ github.event.repository.name }}
      image_tag: ${{ github.sha }}
      template: dotnet
      build_args: --build-arg APP_PROJECT=src/<path-to-api>.csproj --build-arg APP_DLL=<app-name>.dll
      push_image: false
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Set `push_image: false` for validation-only builds that should not publish to GHCR. Use `packages: write` only when publishing images. See [Container builds](docs/container-builds.md) for template defaults, private feed restore, and build argument guidance.

### Python Container Build

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
      template: python
      build_args: --build-arg APP_PORT=8080
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

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
    secrets:
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
      GHCR_USERNAME: ${{ secrets.GHCR_USERNAME }}
      GHCR_PASSWORD: ${{ secrets.GHCR_PASSWORD }}
    with:
      deploy_env: prod
      deploy_target: webapp
      image_name: ${{ github.event.repository.name }}
      image_tag: ${{ github.event.workflow_run.head_sha }}
```

See [Deployments](docs/deployments.md) for target-specific variables, secrets, and port behavior.

### .NET Test

```yaml
name: Test

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: read

jobs:
  test:
    uses: <owner>/<templates-repo>/.github/workflows/dotnet-test.yml@main
    with:
      test_paths: |
        ./Platform.Example.slnx
```

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
      test_paths: |
        ./<solution-or-test-project>.slnx
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
