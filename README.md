# DevOps Templates

Minimal, pipeline-agnostic templates for containerized application releases to Azure Web App for Containers (Linux).

## Structure
```
pipelines/
  templates/
    entrypoints/
      release.yml
    stages/
      build-stage.yml
      deploy-stage.yml
    jobs/
      build-job.yml
      deploy-job.yml
    steps/
      checkout.yml
      app-build.yml
      container-build.yml
      container-tag.yml
      container-push.yml
      select-environment.yml
      azure-auth.yml
      azure-webapp-deploy.yml
      deploy-verify.yml
```

## Design principles
- Pipeline-agnostic: templates can be adapted to GitHub Actions, Azure DevOps, or similar.
- Multi-stage: build then deploy.
- Modular: each template does one thing and accepts parameters.
- Minimal: no approvals, slot swaps, health checks, or rollbacks.

## Entry point
`pipelines/templates/entrypoints/release.yml` orchestrates build -> deploy and defines all parameters used by child templates.

## Parameters (summary)
- Build: `build_command`, `container_build_command`, `container_tag_command`, `container_push_command`
- Image: `image_name`, `image_tag`, `registry_url`, `registry_repository`
- Deploy: `environment_name`, `azure_auth_command`, `azure_deploy_command`, `deploy_verify_command`
- App: `app_name`

## Adapting to a CI system
These templates use a generic `template:` include style. Map that to your CI system’s include/uses mechanism.
- GitHub Actions: translate to `uses:` with `workflow_call` templates or composite actions.
- Azure DevOps: map directly to YAML templates (`template:` is already native).

## Example parameter values (non-binding)
```
app_name: my-webapp
image_name: my-app
image_tag: 1.0.0
registry_url: myregistry.azurecr.io
registry_repository: apps
build_command: make build
container_build_command: docker build -t myregistry.azurecr.io/apps/my-app:1.0.0 .
container_tag_command: docker tag myregistry.azurecr.io/apps/my-app:1.0.0 myregistry.azurecr.io/apps/my-app:latest
container_push_command: docker push myregistry.azurecr.io/apps/my-app:1.0.0
environment_name: dev
azure_auth_command: az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant $AZURE_TENANT_ID
azure_deploy_command: az webapp config container set --name my-webapp --resource-group my-rg --docker-custom-image-name myregistry.azurecr.io/apps/my-app:1.0.0
deploy_verify_command: az webapp show --name my-webapp --resource-group my-rg
```

## Extending later
Designed to be extended with:
- environment-specific parameters
- approvals
- slot swaps
- health checks
- rollbacks
