# Container Builds

The build workflow builds container images with Podman and pushes them directly to GHCR. GHCR is the source of truth for runtime images; the workflow does not upload image tar files as GitHub Actions artifacts.

## Build Workflow

Use `.github/workflows/build.yml` from a consuming repository:

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

Set `template` explicitly. The template identifies the reusable Containerfile family under `.github/containerfiles/`.

## .NET Builds

Use the `dotnet` template for .NET services:

```yaml
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

Set `APP_PROJECT` to the project file to containerize. Set `APP_DLL` to the published app assembly name, for example `Example.Gateway.Api.dll`.

If private GitHub Packages restore is needed during image build, keep restore policy in the consuming repo `NuGet.Config`. The build workflow forwards these auth build args from `GHCR_TOKEN` to the container build:

- `--build-arg GITHUB_USERNAME=${{ github.actor }}`
- `--build-arg GITHUB_PACKAGES_TOKEN=${{ secrets.GHCR_TOKEN }}`

## Python Builds

Use the `python` template for Python services:

```yaml
jobs:
  build:
    uses: <owner>/<templates-repo>/.github/workflows/build.yml@main
    with:
      image_name: ${{ github.event.repository.name }}
      image_tag: ${{ github.sha }}
      template: python
      build_args: >-
        --build-arg PYTHON_VERSION=3.12
        --build-arg APP_PORT=8080
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The Python template supports `uv.lock`, `pyproject.toml`, and
`requirements.txt` projects. It uses `uv` during the image build, runs as the
non-root `appuser`, exposes `APP_PORT`, and defaults to
`APP_COMMAND="python main.py"` at runtime. Override `APP_COMMAND` in the
consuming runtime environment when a service starts differently, for example
`APP_COMMAND="gunicorn app:app --bind 0.0.0.0:8080"`.

Optional build arguments:

- `PYTHON_VERSION`: Python slim image version. Defaults to `3.12`.
- `APP_PORT`: documented container port. Defaults to `8080`.
- `APT_BUILD_PACKAGES`: Debian packages needed only while building Python
  dependencies, for example compiler or header packages.
- `APT_RUNTIME_PACKAGES`: Debian packages needed by the running application.

## Defaults

- `.github/containerfiles/node/Containerfile` runs as non-root `appuser` and uses port `8080`.
- `.github/containerfiles/dotnet/Containerfile` runs as non-root `appuser` and uses port `8080`.
- `.github/containerfiles/python/Containerfile` runs as non-root `appuser` and uses port `8080`.

Override `context`, `containerfile`, or `build_args` only when the consuming repository layout requires it.
