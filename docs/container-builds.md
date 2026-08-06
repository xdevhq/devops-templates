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

The build workflow forwards GitHub Packages auth to container builds:

- `GITHUB_USERNAME`: `${{ github.actor }}`
- `GITHUB_PACKAGES_OWNER`: `${{ github.repository_owner }}`
- `github_packages_token`: a Podman build secret sourced from `GHCR_TOKEN`

The Node and .NET templates use these values only during dependency install or
restore. Do not commit real package credentials to consuming repositories.

## Build Context Ignore Files

Add a `.containerignore` file to the root of the consuming repository build
context. The reusable templates copy the selected Containerfile into that
context and build with `COPY . .`, so ignored files must be controlled by the
consuming repository.

At minimum, exclude local dependencies, build outputs, test reports, caches,
logs, editor files, and local environment files that are not needed during the
image build. Use the `context` workflow input when a service builds from a
subdirectory, and place `.containerignore` in that same context directory.

Podman also supports `.dockerignore`, but `.containerignore` takes precedence
when both files exist. Prefer `.containerignore` for repositories using this
workflow because the build runs with Podman. Use `.dockerignore` only when the
same build context must also be built directly with Docker tooling.

## Node Builds

Use the `node` template for Node services:

```yaml
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

For private GitHub Packages dependencies, commit registry policy in the
consuming repo `.npmrc` and use the `NODE_AUTH_TOKEN` placeholder:

```ini
@<scope>:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
always-auth=true
```

The Node template reads the `github_packages_token` build secret and exposes it
as `NODE_AUTH_TOKEN` while it runs `pnpm install`, `yarn install`, `npm ci`, or
`npm install`. If `.npmrc` is absent and the build secret is available, the
template creates a temporary GitHub Packages `.npmrc` for
`GITHUB_PACKAGES_OWNER`. The template removes `.npmrc` after dependency install
and production pruning so registry auth is not copied into the runtime image.

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

For private GitHub Packages dependencies, commit restore source policy in the
consuming repo `NuGet.Config` and use `github` as the package source key:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="github" value="https://nuget.pkg.github.com/<owner>/index.json" />
  </packageSources>
</configuration>
```

The .NET template supplies credentials for the `github` source during
`dotnet restore` through NuGet's
`NuGetPackageSourceCredentials_github` environment variable, sourced from the
`github_packages_token` build secret. If `NuGet.Config` or `nuget.config` is
absent and the build secret is available, the
template creates a temporary `NuGet.Config` with `nuget.org` and
`https://nuget.pkg.github.com/${GITHUB_PACKAGES_OWNER}/index.json`, runs
restore, and removes the generated config.

The build workflow forwards these metadata build args to the container build:

- `--build-arg GITHUB_USERNAME=${{ github.actor }}`
- `--build-arg GITHUB_PACKAGES_OWNER=${{ github.repository_owner }}`

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
