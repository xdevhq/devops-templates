# DevOps Templates Docs

This directory contains detailed consumer guidance for the reusable workflows, composite actions, and container templates in this repository.

Start with the root [`README.md`](../README.md) for quick-start examples. Use these docs for contract details and operational guidance:

- [Workflow contracts](workflows.md): reusable workflow inputs, outputs, secrets, permissions, defaults, and allowed values.
- [Required secrets and variables](secrets-and-variables.md): required consuming-repository secrets and environment variables.
- [Container builds](container-builds.md): GHCR image builds, Podman behavior, Containerfile templates, and build arguments.
- [Deployments](deployments.md): Azure Web App and Azure Container Apps deployment behavior.
- [Package publishing](package-publishing.md): NuGet and npm publishing behavior.
- [Cleanup](cleanup.md): GitHub Actions artifact cleanup, GHCR cleanup, and retention strategy.
- [Repository access](repository-access.md): access setup when this templates repository is private.

Keep consumer-specific environment values, secrets, and registry policy in consuming repositories.
