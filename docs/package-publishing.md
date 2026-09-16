# Package Publishing

This repository provides reusable workflows for NuGet and npm package publishing. Registry and restore policy should stay in the consuming repository.

## NuGet

Use `.github/workflows/nuget.yml` to restore, optionally test, pack, and publish a NuGet package:

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

Guidance:
- Restore source policy should live in the consuming repo `NuGet.Config`.
- Provide `test_paths` when the package should run validation before packing. Each non-empty line is passed to `dotnet test`.
- Use `github` as the GitHub Packages source key. The workflow supplies
  credentials for that source during restore.
- The default publish target is the GitHub Packages owner feed.
- Override the publish target with `source_url`.
- Use `NUGET_API_KEY` only when publishing to feeds that require a non-GitHub token, such as nuget.org.

Example `NuGet.Config` for consuming repositories:

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

## npm

Use `.github/workflows/npm.yml` to install, validate, build, and publish an npm package:

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

Guidance:
- Keep registry and auth policy in the consuming repo `.npmrc`.
- The default publish target is GitHub Packages: `https://npm.pkg.github.com`.
- Version is read from the consuming repo `package.json`.
- `NPM_TOKEN` is optional for GitHub Packages because the workflow falls back to `GITHUB_TOKEN`.

Example `.npmrc` for consuming repositories:

```ini
@<scope>:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
always-auth=true
```

Use `publish: false` to run install and quality gates without publishing. Use `dry_run: true` to exercise publish behavior without publishing a package.

Package manager versions:
- For `pnpm` and `yarn`, the installer uses the consuming package's `packageManager` field when present.
- If `pnpm` or `yarn` is selected and no matching `packageManager` field is present, the installer activates the latest available version through Corepack.
