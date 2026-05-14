# stagepass-docs

Design artifact repository for the StagePass platform.

## What lives here

| Path | Contents |
|------|----------|
| `/PRD.md` | Product Requirements Document |
| `/NFR.md` | Non-Functional Requirements with measurable SLOs |
| `/CONTRIBUTING.md` | How to make changes correctly |
| `/docs/adr/` | Architecture Decision Records (ADR-000 template, ADR-001 onwards) |
| `/docs/api/` | OpenAPI specs (one YAML per service, contract-first) |
| `/docs/async-api/` | AsyncAPI schemas (one YAML per Kafka topic) |
| `/docs/architecture/` | HLD C4 diagrams (L1 + L2), sequence diagrams |
| `/docs/threat-model/` | STRIDE threat model per service |
| `/docs/runbooks/` | One runbook per Prometheus alert |
| `/docs/er-diagrams/` | ER diagrams per service |
| `/docs/standards/` | Coding standards and conventions per language |

## Rules

- No code. No config. Documents only.
- Every document must stand alone: header, version, status, traces-to references, changelog.
- Every merged document must be uploaded to the Claude Project knowledge base immediately.
- Branch prefix: `docs/<issue-number>-<short-description>`