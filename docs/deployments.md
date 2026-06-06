# Deployments

The deploy workflow deploys a GHCR-hosted container image to Azure Web App for Containers or Azure Container Apps.

## Build Once, Deploy On Success

Use a build workflow first, then trigger deployment from the successful build:

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

## Targets

Set `deploy_target` to:
- `webapp` (default): deploys to Azure Web App for Containers and requires `AZURE_WEBAPP_NAME`.
- `containerapp`: deploys to Azure Container Apps and requires `AZURE_CONTAINERAPP_NAME`.

Both targets require:
- `AZURE_RESOURCE_GROUP`
- `AZURE_CREDENTIALS`
- `GHCR_USERNAME`
- `GHCR_PASSWORD`

The deploy job uses `deploy_env` as the GitHub environment, so environment-specific variables and secrets can live on the selected environment in the consuming repository.

## Container Port

`container_port` defaults to `8080`.

For Web App deployments, the value is applied as `WEBSITES_PORT`. For Container Apps deployments, the value is applied as the ingress `targetPort`.

## Azure Credentials Helper

Recommended baseline:

- Scope RBAC at resource group level or narrower where possible.
- Create one `AZURE_CREDENTIALS` secret per GitHub Environment target.

Create the App Registration / service principal once:

```bash
az ad app create \
  --display-name "sp-gha-deploy-<ENV_GROUP>" \
  --query appId \
  --output tsv
```

```bash
az ad sp create \
  --id "<APP_ID>"
```

```bash
az account show \
  --query tenantId \
  --output tsv
```

Use `appId` as `<APP_ID>`. Then assign RBAC per target subscription/resource group:

```bash
az role assignment create \
  --assignee "<APP_ID>" \
  --role "Contributor" \
  --scope "/subscriptions/<SUB_ID>/resourceGroups/<RG_NAME>"
```

Repeat the role assignment command for each target scope.

Create one `AZURE_CREDENTIALS` JSON value per deployment target by running this command once per target:

```bash
az ad app credential reset \
  --id "<APP_ID>" \
  --append \
  --display-name "github-<RG>" \
  --years 1 \
  --query "{clientId:'<APP_ID>',clientSecret:password,subscriptionId:'<SUB_ID>',tenantId:'<TENANT_ID>'}" \
  --output json
```

Store one `AZURE_CREDENTIALS` secret per GitHub Environment:

```json
{"clientId":"<APP_ID>","clientSecret":"<SECRET_FROM_COMMAND>","subscriptionId":"<SUB_ID>","tenantId":"<TENANT_ID>"}
```

For each target, set in that GitHub Environment:

- secret: `AZURE_CREDENTIALS`
- vars: `AZURE_RESOURCE_GROUP` and one of:
  - `AZURE_WEBAPP_NAME` for Web App deploy target
  - `AZURE_CONTAINERAPP_NAME` for Container Apps deploy target

RBAC note: all secrets on the same App Registration represent the same identity and permissions. Grant RBAC access for all target resources, such as Web Apps and/or Container Apps, or at resource group scope. If you need different permissions per app/environment, use separate service principals. If targets are across different tenants, use separate identities per tenant.
