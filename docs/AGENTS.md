# Docs Agent Guide
Agents must read this file before making changes within this directory scope.

## Scope
This file applies to everything under `docs/`.

The root `AGENTS.md` still applies. This guide adds documentation-specific rules for the DevOps templates repository.

## Docs Purpose
The `docs/` directory is for durable, consumer-facing guidance that does not belong in inline workflow comments or short README quick starts.

Use `docs/` for:
- reusable workflow behavior and contracts
- composite action conventions
- publishing, deployment, cleanup, and container build patterns
- security, permissions, secrets, and token guidance
- migration notes for meaningful template contract changes
- operational guidance for consumers of these templates

## Documentation Boundaries
- Treat workflows, composite actions, and documented inputs/outputs as public contracts.
- Keep the root `README.md` as the concise entry point and quick-start guide.
- Put deeper explanations in `docs/` only when they would make the README too large or repetitive.
- Do not document repository-internal implementation details unless they affect downstream consumers.
- Do not duplicate YAML reference content that is already clear from a workflow file unless consumers need context, examples, or warnings.
- Do not place consuming-repository secrets, credentials, environment values, or organization-specific assumptions in docs.

## Change Guidelines
- Before adding a new document, check whether the root `README.md` or an existing docs page should be updated instead.
- Prefer extending one canonical document over creating overlapping pages for the same workflow or pattern.
- Keep examples short, valid, and consumer-focused.
- Preserve backward compatibility language when documenting changed inputs, outputs, permissions, secrets, or defaults.
- Clearly call out breaking changes, migration steps, and affected workflows when behavior changes.
- Request approval before major docs taxonomy changes or broad documentation rewrites.

## Validation
- Verify documented workflow names, inputs, outputs, permissions, defaults, secrets, and variables against the actual files under `.github/workflows/` and `.github/actions/`.
- Check README examples when docs change reusable workflow behavior.
- Prefer official GitHub, Azure, Docker, NuGet, npm, or package registry documentation when adding new platform guidance.
- Remove or correct stale guidance when implementation and docs disagree.

## Style
- Keep documentation minimal, practical, and stable over time.
- Use direct headings and short sections.
- Prefer links to canonical docs over repeated explanations.
- Document the consumer impact and expected usage, not every implementation step.
- Keep Markdown portable and readable in GitHub.
