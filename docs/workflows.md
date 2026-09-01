# Workflow Contracts

Reusable workflow inputs, outputs, secrets, and permissions are public contracts for downstream repositories. Keep this page aligned with `.github/workflows/`.

## Build

Workflow: `.github/workflows/build.yml`

Purpose: build a container image from a selected template and push it to GHCR.

Inputs:
- `image_name` (required): GHCR package/image name.
- `image_tag` (required): image tag to build and push.
- `template` (required): container template folder under `.github/containerfiles/`.
- `context` (optional, default `.`): build context path in the consuming repository.
- `containerfile` (optional, default `Containerfile`): Containerfile name or path resolved by the build action.
- `build_args` (optional, default empty): extra Podman build arguments.

Secrets:
- `GHCR_TOKEN` (required): token used to authenticate to GHCR and passed as a build secret for GitHub Packages dependency install or restore. Usually `${{ secrets.GITHUB_TOKEN }}`.

Outputs:
- `image_name`
- `image_tag`

Caller permissions:
- `contents: read`
- `packages: write`

## Deploy

Workflow: `.github/workflows/deploy.yml`

Purpose: deploy a GHCR-hosted container image to Azure Web App for Containers or Azure Container Apps.

Inputs:
- `deploy_env` (required): GitHub environment name used by the deploy job.
- `deploy_target` (optional, default `webapp`): allowed values are `webapp` and `containerapp`.
- `image_name` (required): GHCR package/image name.
- `image_tag` (required): image tag to deploy.
- `container_port` (optional, default `8080`): application port exposed by the container.

Secrets:
- `AZURE_CREDENTIALS` (required)
- `GHCR_USERNAME` (required)
- `GHCR_PASSWORD` (required)
- Callers in the same organization or enterprise may use `secrets: inherit`.
- Callers outside that boundary should pass named repository or organization secrets explicitly.

Variables:
- `AZURE_RESOURCE_GROUP` is required for both targets.
- `AZURE_WEBAPP_NAME` is required for `deploy_target: webapp`.
- `AZURE_CONTAINERAPP_NAME` is required for `deploy_target: containerapp`.

Caller permissions:
- Use the minimum permissions required by the caller workflow. The reusable deploy workflow authenticates to Azure through inherited secrets.

## NuGet Package

Workflow: `.github/workflows/nuget.yml`

Purpose: restore, pack, and publish a NuGet package.

Inputs:
- `project_path` (required): project file to package.
- `dotnet_version` (optional, default `8.0.x`): .NET SDK version.
- `configuration` (optional, default `Release`): build configuration used for packing.
- `package_version` (optional, default empty): package version override.
- `source_url` (optional, default empty): publish target override. Empty defaults to `https://nuget.pkg.github.com/<owner>/index.json`.

Secrets:
- `NUGET_API_KEY` (optional): required only when publishing to feeds that do not accept `GITHUB_TOKEN`.

Job permissions:
- `contents: read`
- `packages: write`

## NPM Package

Workflow: `.github/workflows/npm.yml`

Purpose: install dependencies, optionally run quality gates, and optionally publish an npm package.

Inputs:
- `package_path` (required): package directory.
- `node_version` (optional, default `20`): Node.js version.
- `package_manager` (optional, default `auto`): allowed values are `auto`, `pnpm`, `yarn`, and `npm`.
- `registry_url` (optional, default `https://npm.pkg.github.com`): publish registry URL.
- `run_lint` (optional, default `true`): run the package `lint` script.
- `run_test` (optional, default `true`): run the package `test` script.
- `run_typecheck` (optional, default `true`): run the package `typecheck` script.
- `run_build` (optional, default `true`): run the package `build` script.
- `publish` (optional, default `true`): publish the package.
- `dry_run` (optional, default `false`): run publish in dry-run mode.

Secrets:
- `NPM_TOKEN` (optional): publish token. The workflow falls back to `GITHUB_TOKEN` for GitHub Packages.

Outputs:
- `package_name`
- `package_version`
- `package_manager`

Job permissions:
- `contents: read`
- `packages: write`

Package manager versions:
- For `pnpm` and `yarn`, the installer uses the consuming package's `packageManager` field when present.
- If `pnpm` or `yarn` is selected and no matching `packageManager` field is present, the installer activates the latest available version through Corepack.

## Cleanup Artifacts

Workflow: `.github/workflows/cleanup-artifacts.yml`

Purpose: delete older GitHub Actions artifacts while keeping the newest artifacts.

Inputs:
- `keep_latest_count` (optional, default `20`): number of newest artifacts to keep.
- `dry_run` (optional, default `false`): log intended deletions without deleting artifacts.

Secrets:
- No additional secrets are required when using `GITHUB_TOKEN`.

Job permissions:
- `actions: write`
- `contents: read`

## Cleanup GHCR

Workflow: `.github/workflows/cleanup-ghcr.yml`

Purpose: delete older GHCR container package versions.

Inputs:
- `package_name` (required): GHCR container package name.
- `min_versions_to_keep` (optional, default `20`): minimum number of package versions to retain.
- `delete_only_untagged` (optional, default `false`): restrict deletion to untagged versions.

Secrets:
- No additional secrets are required when using `GITHUB_TOKEN`.

Job permissions:
- `packages: write`
- `contents: read`
