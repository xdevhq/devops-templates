# Required Secrets And Variables

Keep secrets, credentials, and environment-specific values in consuming repositories, not in this templates repository.

## Build And Push

Secrets:
- `GHCR_TOKEN`: usually `${{ secrets.GITHUB_TOKEN }}` in the consuming repository. The build workflow uses it for GHCR push authentication and as the GitHub Packages build secret for Node and .NET dependency install or restore.

## Deploy

Caller requirements:
- The caller workflow must include `secrets: inherit` when invoking the reusable deploy workflow.

Environment secret in the consuming repository environment:
- `AZURE_CREDENTIALS`

Repository-level or environment-level secrets in the consuming repository:
- `GHCR_USERNAME`
- `GHCR_PASSWORD`

Environment variables in the consuming repository environment:
- `AZURE_WEBAPP_NAME` for `deploy_target: webapp`
- `AZURE_CONTAINERAPP_NAME` for `deploy_target: containerapp`
- `AZURE_RESOURCE_GROUP`

## NuGet

Secrets:
- `NUGET_API_KEY`: optional, only for non-default feeds.

## npm

Secrets:
- `NPM_TOKEN`: optional; defaults to `GITHUB_TOKEN` for GitHub Packages.

## Cleanup

No additional secrets are required when using `GITHUB_TOKEN`.
