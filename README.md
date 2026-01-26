# Minimal Release Pipeline (v0.1)

This repo provides a small, componentized GitHub Actions release pipeline design for containerized apps deployed to Azure Web App for Containers (Linux). It is intentionally simple and language-agnostic.

## How it composes

Entry points → stages → jobs → steps

- `.github/workflows/release.yml`: release entry point
- `.github/workflows/stages/*`: stage composition
- `.github/workflows/jobs/*`: job definitions
- `.github/workflows/steps/*`: small, single-purpose steps (composite actions)

## Required secrets

- `GITHUB_TOKEN`: for GHCR push (provided by GitHub)
- `AZURE_CREDENTIALS`: Azure service principal JSON for `azure/login` (environment secret)

## Consume from another repo

```yaml
jobs:
  release:
    uses: alexhovy/devops-templates/.github/workflows/release.yml@main
    secrets: inherit
```

## Environment config in the consuming repo

- Create a GitHub Environment named `prod`.
- Set `AZURE_WEBAPP_NAME` as an environment variable (not secret).
- Set `AZURE_CREDENTIALS` as an environment secret.


## Notes

- This is a minimal, design-focused layout. GitHub Actions only loads workflows from `.github/workflows`, and composite actions require an `action.yml` inside a directory.
- Podman is used for build/tag/push. No Docker.
- Azure logic is isolated to deploy components.

## Guiding sentence

This pipeline should feel boring, obvious, and easy to extend.
