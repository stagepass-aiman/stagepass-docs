# Contributing to stagepass-docs

## Branch Strategy

All changes go through a PR. No direct commits to `main`.

| Branch prefix | Purpose |
|---|---|
| `docs/<issue>-<description>` | New or revised design artifact |
| `chore/<issue>-<description>` | Template, tooling, or structure changes |

## Commit Convention

Format: `<type>(<scope>): <imperative description>`

docs(adr): add ADR-006 seat inventory concurrency decision
chore(templates): update issue template for new artifact type

Body explains WHY, not WHAT. Reference the issue number: `Refs #N`.

## Document Standards

Every document must include a header block:

```markdown
---
id: <ARTIFACT-ID>          # e.g. ADR-006, PRD, NFR
title: <Full Title>
status: Draft | Proposed | Accepted | Deprecated | Superseded
version: 0.1.0
created: YYYY-MM-DD
updated: YYYY-MM-DD
authors: [your-github-handle]
traces-to: [list of related ADR IDs or NFR IDs]
---
```

## Coding Standards by Language

These apply to all service repos. They live here as the single source of truth.

### Java (Java 21)
- Records, sealed classes, pattern matching, virtual threads — use them.
- No Lombok. No field injection. Constructor injection only.
- Explicit over magic.
- Formatter: Google Java Format (enforced in CI via `google-java-format`).
- Linter: Checkstyle with Google ruleset + custom StagePass additions.
- No raw types. No unchecked casts without a comment explaining why.

### TypeScript / JavaScript
- Strict TypeScript everywhere. `"strict": true` in every `tsconfig.json`.
- No `any`. No `as` casts without an explanatory comment.
- ESLint: `@typescript-eslint/recommended` + `import` plugin.
- Formatter: Prettier, config shared via `@stagepass/prettier-config`.
- No default exports in library code (named exports only for tree-shaking).

### Python
- Type hints throughout. Pydantic at all service boundaries.
- Formatter: Black, line length 88.
- Linter: Ruff (replaces flake8 + isort + pyupgrade).
- No bare `except` clauses. Always catch a specific exception type.
- No mutable default arguments.

### Proto / gRPC
- Package naming: `stagepass.<service>.v1`
- All fields must have comments.
- Backward-compatible evolution only: add optional fields, never remove,
  never change field numbers.

### SQL
- Migrations via Flyway (Java services) or Alembic (Python services).
- Migration files are immutable once merged.
- Every migration must be reversible (include `V{n}__undo` where feasible).
- Column names: `snake_case`. Table names: `snake_case`, plural.
- Every table has `created_at` and `updated_at` with UTC timestamps.

## ADR Process

1. Open an issue with label `artifact:adr` and template `adr.yml`.
2. Branch: `docs/<issue>-adr-<NNN>-<short-title>`
3. Copy `docs/adr/ADR-000-template.md`, rename to `ADR-<NNN>-<title>.md`.
4. Fill in every section. Do not leave sections blank.
5. Status starts as `Proposed`. Changes to `Accepted` only after review.
6. PR must link to the issue. Checklist must be complete before merge.
7. After merge: upload to Claude Project knowledge base.

## PR Checklist

Every PR to this repo must confirm:
- [ ] Document header block is complete and accurate
- [ ] All traces-to references are valid (point to real artifact IDs)
- [ ] No contradictions with existing accepted ADRs
- [ ] Changelog entry added if this is a revision
- [ ] Artifact uploaded to Claude Project knowledge base after merge