# AGENTS.md

## Sources of authority

- Treat `docs/design/` as the authoritative published system design.
- Treat `.42p/cases/` as working, evidence, capture, and rationale material unless explicitly stated otherwise.
- Treat `.42p/standards/` as authoritative repository standards governing how relevant work is produced and reviewed.

When sources disagree, do not reconcile them silently. Identify the applicable authority before modifying the repository.

## Standards

Load the relevant standard before modifying its subject:

- Repository documentation: `.42p/standards/editorial.md`

Apply only the standards relevant to the task. Do not load unrelated standards.

## Working principle

Prefer the smallest change that completely satisfies the task.

Do not introduce unsupported scope, abstractions, compatibility guarantees, dependencies, process, or documentation.
