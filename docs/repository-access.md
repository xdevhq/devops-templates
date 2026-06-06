# Repository Access

If this templates repo is private, allow other repositories to call its reusable workflows.

In the templates repository:

- Open Settings -> Actions -> General -> Access.
- Enable access from the repositories that will consume these templates.

Consuming repositories must reference workflows using the repository path and ref, for example:

```yaml
jobs:
  build:
    uses: <owner>/<templates-repo>/.github/workflows/build.yml@main
```
