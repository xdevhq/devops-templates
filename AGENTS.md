# DevOps Templates Agent Guide
Agents must read this file before making changes within this directory scope.

## Scope
This file applies to the entire repository at the root level.

This repository is a shared DevOps template product. Workflows, composite actions, and their inputs/outputs must be treated as public contracts used by downstream repositories.

## Agent Operating Procedure
Before making changes, agents must follow this workflow:
1. Understand context: identify the template or workflow being changed, read applicable `AGENTS.md` files, and review relevant README documentation.
2. Search for existing patterns: find similar workflows, composite actions, and template usage before introducing new approaches.
3. Verify external best practices: for new workflow behavior, deployment changes, or infrastructure patterns, check official GitHub, Azure, Docker, or package publishing documentation.
4. Define a plan: identify affected workflows, actions, inputs, outputs, permissions, secrets, and validation steps before implementation.
5. Validate consumer impact: keep downstream repositories, versioned usage, backward compatibility, and migration impact in mind.
6. Implement minimal changes: prefer incremental updates and reusable building blocks over broad rewrites.
7. Verify correctness: ensure YAML validity and consistent execution behavior, and update README content when behavior, usage, inputs, outputs, or secrets change.

## Repository Structure
- `.github/workflows/`: Reusable GitHub Actions workflows for build, deploy, publish, and cleanup flows.
- `.github/actions/`: Composite actions shared by the reusable workflows.
- `.github/containerfiles/`: Reusable container build templates.
- `README.md`: Primary usage guidance for consuming repositories.

## Working Principles
- Treat this repository as a template product used by downstream repositories.
- Follow existing workflow and action patterns before introducing new structure.
- Prefer reusable, parameterized templates over repo-specific assumptions or hardcoded values.
- Keep workflows simple, readable, and production-safe.
- Apply strong standards for security, reliability, and maintainability.
- Prefer official and broadly adopted platform patterns over ad-hoc or experimental implementations unless explicitly requested.
- Avoid introducing new dependencies, actions, or tools unless clearly necessary.
- Keep secrets, credentials, and environment-specific values in consuming repositories, not in shared templates.

## Discovery Before Changes
- Always search for related workflows, composite actions, and README examples before adding a new pattern.
- Check nearest `AGENTS.md` files first and obey the most specific scope.
- Reuse existing naming conventions for inputs, outputs, secrets, variables, and workflow structure.

## Documentation Standards
- Keep `README.md` concise and practical for consuming repositories.
- When meaningful changes are introduced, update the relevant usage guidance in the same iteration.
- Document purpose, required inputs, optional inputs, secrets, outputs, and example usage when those change.
- Do not duplicate information that is already obvious from the repository layout.

## Change Planning
- Plan work before editing: define scope, affected templates, consumer impact, and validation steps.
- For significant or cross-cutting changes such as workflow contracts, deployment strategy, security posture, or publish behavior, present the plan and seek approval before implementation.
- Prefer incremental, reviewable changes over large rewrites.

## Safety and Quality Baseline
- Do not commit secrets, tokens, credentials, generated artifacts, or other sensitive material.
- Keep reusable workflow interfaces backward compatible when possible; document breaking changes clearly.
- Use least-privilege permissions and explicit `permissions:` blocks in workflows where possible.
- Validate impacted workflows, actions, and documentation before concluding work.

## Instruction Precedence
- When multiple `AGENTS.md` files apply, the most specific scope takes precedence over broader scopes.
