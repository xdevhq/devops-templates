# Cleanup

Use cleanup workflows to keep Actions artifacts and GHCR container versions bounded over time.

## Retention Strategy

- Do not store container image tar files in GitHub Actions artifacts.
- Treat GHCR as the source of truth for runtime images.
- Keep a rollback window in GHCR, for example 20 versions.
- Run scheduled cleanup for old GitHub Actions artifacts and old GHCR container versions.

## GitHub Actions Artifacts

Use `.github/workflows/cleanup-artifacts.yml`:

```yaml
name: Cleanup Artifacts

on:
  schedule:
    - cron: "0 3 * * *"
  workflow_dispatch:

jobs:
  cleanup:
    permissions:
      actions: write
      contents: read
    uses: <owner>/<templates-repo>/.github/workflows/cleanup-artifacts.yml@main
    with:
      keep_latest_count: 20
```

Set `dry_run: true` to log planned deletions without deleting artifacts.

## GHCR Container Versions

Use `.github/workflows/cleanup-ghcr.yml`:

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

Use `delete_only_untagged: true` when tagged image versions must be preserved.
