# ADR-001 — Repository Topology

## Document Header

| Field        | Value                                                                 |
|--------------|-----------------------------------------------------------------------|
| Document ID  | ADR-001                                                               |
| Version      | 1.0.0                                                                 |
| Status       | Accepted                                                              |
| Author       | StagePass Engineering                                                 |
| Created      | 2025-05-15                                                            |
| Last Updated | 2025-05-15                                                            |
| Repo         | stagepass-docs                                                        |
| Path         | /docs/adr/ADR-001-repository-topology.md                             |
| Phase        | Phase 1 — Requirements & Design                                       |
| Traces To    | PRD.md §4 (Roadmap), PRD.md §7 (GitHub Organisation Structure),      |
|              | PRD.md Constraints C-01 C-05 C-06, NFR.md §3 (Service Tier Map)     |

### Change Log

| Version | Date       | Author                | Summary                  |
|---------|------------|-----------------------|--------------------------|
| 1.0.0   | 2025-05-15 | StagePass Engineering | Initial accepted version |

---

## Table of Contents

1. [Status](#1-status)
2. [Context](#2-context)
3. [Decision](#3-decision)
4. [Consequences](#4-consequences)
5. [Alternatives Considered](#5-alternatives-considered)
6. [References](#6-references)
7. [Quick Self-Check](#7-quick-self-check)

---

## 1. Status

**Accepted.**

This ADR is the authoritative record of the repository topology decision for
StagePass. All subsequent service scaffolding, CI pipeline setup, Helm chart
organisation, and documentation placement must conform to the structure defined
here. Deviations require a superseding ADR.

---

## 2. Context

### 2.1 What We Are Deciding

StagePass is a platform of 17 backend services, 1 API gateway, 1 frontend
application (containing 4 distinct surfaces), 1 shared contracts library, and
3 cross-cutting concern repositories (documentation, infrastructure,
shared contracts). Before a single line of code is written, the repository
topology must be locked — every other Phase 2–10 decision builds on top of it.

The topology determines:

- Where each artifact lives and who owns it
- How service isolation is enforced (dependency boundaries, deployment
  independence, CI scope)
- What tooling overhead the solo developer must sustain
- How the learning goal of polyglot service development is supported
- Whether the structure can scale from Phase 2 (2 active services) to
  Phase 10 (20+ active repos)

### 2.2 Forces at Play

**Solo development (PRD C-01).**
One developer, 9–12 months. Every tool, workflow, and repo that exists must
earn its place. Cognitive overhead compounds directly into missed learning time.
The topology must be simple enough to operate daily without friction, yet
disciplined enough to teach real-team habits.

**Polyglot by design (PRD C-06).**
Services are deliberately built in Java (Spring Boot, Quarkus), TypeScript
(NestJS, Fastify, Node.js), and Python (FastAPI, Flask). This is a learning
goal, not an accident. Each language brings its own build tool (Maven/Gradle,
npm/pnpm, pip/Poetry), its own linter, its own test runner, and its own CI
configuration. The topology must contain this diversity without letting it bleed
across service boundaries.

**Stack is locked (PRD C-05).**
The technology choices per service are fixed. The topology must accommodate
them exactly as specified — no reshaping of the stack to fit a different repo
structure is acceptable.

**Sequential phases and just-in-time service creation (PRD §4).**
Repos are not created upfront. Service repos are created at the start of the
phase that implements that service. The topology must be coherent from Phase 0
(3 Tier 1 repos only) through Phase 10 (21 repos). Early decisions must not
force rework when later repos are added.

**Deployment independence is a core learning objective.**
A key goal is to demonstrate that services can be deployed, updated, and rolled
back independently (PRD §2 — Learning Goals, Distributed Systems). This
property is architectural, not just operational. It requires that no service
repo has a runtime dependency on another service repo's source code.

**Service criticality is not uniform (NFR §3).**
T1 services (Auth, Seat Inventory, Booking, Payment, Disbursement) have a
99.9% SLO and 43-minute monthly error budget. T3 services (Recommendation,
Chatbot, Fraud Detection, Analytics, Waitlist) have a 99.0% SLO and 7-hour
budget. The topology must support independent CI gating, deployment promotion,
and versioning per service — a T3 change must not block a T1 deployment.

**GitHub Free plan (PRD §7).**
The organisation runs on the free tier. There are no private npm/Maven
packages. No GitHub Packages registry for internal libraries. Shared
contracts (Protobuf definitions, Kafka schemas) must be distributed as
source — referenced by Git tag, not as compiled packages.

**Shared contracts are a first-class concern.**
Two pairs of services communicate via gRPC (Booking→Seat Inventory,
Check-in→Ticket). All services communicate via Kafka with typed event
schemas. The `.proto` files and JSON Schemas that define these contracts
must live in a single, versioned, authoritative location. No service may
own a contract that another service depends on.

### 2.3 What This Decision Is Not About

This ADR decides repository topology only. It does not decide:

- Service communication patterns (→ ADR-003)
- Tech stack per service (→ ADR-002)
- Money type representation (→ ADR-004)
- Booking saga pattern (→ ADR-005)
- Seat inventory concurrency (→ ADR-006)
- Flash sale queue pattern (→ ADR-007)
- Revenue split and disbursement model (→ ADR-008)

---

## 3. Decision

**We adopt a three-tier polyrepo topology.**

All repositories live inside the `stagepass-aiman` GitHub organisation.
The topology consists of 21 repositories organised into three tiers,
created in sequence as the project progresses.

### 3.1 Topology Overview

```
stagepass-aiman (GitHub Organisation — free plan)
│
├── TIER 1 — Cross-cutting (created Phase 0, always exist)
│   ├── stagepass-docs
│   ├── stagepass-shared-contracts
│   └── stagepass-infrastructure
│
├── TIER 2 — Service repos (created just-in-time per phase)
│   ├── stagepass-api-gateway            [Phase 3]
│   ├── stagepass-auth-service           [Phase 3]
│   ├── stagepass-event-service          [Phase 3]
│   ├── stagepass-venue-service          [Phase 4]
│   ├── stagepass-seat-inventory-service [Phase 4]
│   ├── stagepass-booking-service        [Phase 4]
│   ├── stagepass-payment-service        [Phase 4]
│   ├── stagepass-disbursement-service   [Phase 4]
│   ├── stagepass-ticket-service         [Phase 4]
│   ├── stagepass-check-in-service       [Phase 4]
│   ├── stagepass-notification-service   [Phase 4]
│   ├── stagepass-waitlist-service       [Phase 4]
│   ├── stagepass-search-service         [Phase 5]
│   ├── stagepass-recommendation-service [Phase 5]
│   ├── stagepass-chatbot-service        [Phase 5]
│   ├── stagepass-fraud-detection-service[Phase 5]
│   └── stagepass-analytics-service      [Phase 5]
│
└── TIER 3 — Frontend (created Phase 6)
    └── stagepass-web
```

### 3.2 Tier 1 Repositories — Detailed Specification

#### 3.2.1 `stagepass-docs`

**Purpose:** Single source of truth for all design artifacts. No code.
No configuration. No runnable content.

**Owns:**

| Artifact | Path |
|---|---|
| PRD | `/PRD.md` |
| NFR | `/NFR.md` |
| ADR-001 through ADR-008 (and any future ADRs) | `/docs/adr/` |
| C4 HLD diagrams (Mermaid) | `/docs/architecture/HLD.md` |
| Sequence diagrams | `/docs/architecture/sequences/` |
| OpenAPI specs (contract-first, one per service) | `/docs/api/<service>.yaml` |
| AsyncAPI event schemas (one per Kafka topic) | `/docs/async-api/<topic>.yaml` |
| ER diagrams (one per service) | `/docs/er-diagrams/<service>.md` |
| Threat model (STRIDE) | `/docs/threat-model/STRIDE.md` |
| Runbooks (one per Alertmanager alert) | `/docs/runbooks/<alert-name>.md` |

**Access pattern:** Read by all services. Written only by design
and documentation work. Changes require a `docs/*` branch, PR, and
CI validation (link checking, Mermaid rendering, OpenAPI schema lint).

**Why it exists separately:** Design artifacts have a different
change cadence from code. They are reviewed differently. They are
consumed by humans, not build tools. Separating them prevents design
documents from being buried inside service repositories where they
are not easily discoverable. Every service's OpenAPI spec lives
here, not in the service repo, because the spec is the contract —
it is consumed by multiple parties (API Gateway, Pact tests,
Postman collections) and must be authoritative independent of any
one service implementation.

#### 3.2.2 `stagepass-shared-contracts`

**Purpose:** Authoritative definitions for all cross-service
communication contracts. This is the only permitted source of truth
for Protobuf files and Kafka message schemas.

**Owns:**

| Artifact | Path |
|---|---|
| Protobuf definition: Seat Inventory Service | `/proto/stagepass/seat_inventory/v1/seat_inventory.proto` |
| Protobuf definition: Ticket Service | `/proto/stagepass/ticket/v1/ticket.proto` |
| Kafka event schemas (one directory per domain) | `/schemas/events/<domain>/` |

**Kafka event schema directories:**

```
/schemas/events/
  booking/
  payment/
  seat/
  notification/
  fraud/
  disbursement/
  waitlist/
  analytics/
```

**Access pattern:** Services reference this repo by pinning a specific
Git tag. Because the GitHub Free plan has no private package registry,
there is no compiled artefact to consume. Each service generates its
own gRPC stubs from the `.proto` files at build time using the
appropriate protoc plugin for its language. Kafka schemas are
referenced as JSON Schema files, validated at consumer startup.

**Schema evolution contract:**
- Add optional fields freely
- Never remove an existing field
- Never change the type of an existing field
- Never change a Protobuf field number
- Breaking changes require a new version directory
  (`v2/`) and a migration ADR

**Why it exists separately:** Shared contracts are the API surface
between services. Their ownership cannot belong to any one service —
if the Booking Service owned the Protobuf that Seat Inventory uses,
Seat Inventory's release process would be hostage to Booking's
repository. A standalone repo with independent versioning (Git tags)
makes this boundary explicit and enforces the rule that a producer
cannot silently break a consumer.

#### 3.2.3 `stagepass-infrastructure`

**Purpose:** All runnable infrastructure configuration.
No service application code.

**Owns:**

| Artifact | Path |
|---|---|
| Docker Compose (full local stack) | `/docker/compose/docker-compose.yml` |
| Helm chart per service | `/helm/charts/<service>/` |
| Kubernetes namespaces and manifests | `/k8s/namespaces/`, `/k8s/configmaps/` |
| Terraform modules (Phase 9) | `/terraform/modules/`, `/terraform/environments/` |
| Prometheus alert rules | `/observability/prometheus/rules/` |
| Grafana dashboards | `/observability/grafana/dashboards/` |
| Loki configuration | `/observability/loki/` |
| Jaeger configuration | `/observability/jaeger/` |
| Utility scripts | `/scripts/` |

**Key scripts:**

```
/scripts/
  check-memory.sh       # Assert docker stack < 12 GB (NFR-PERF-043)
  seed-data.sh          # Seed development database fixtures
  smoke-test.sh         # Verify all service health endpoints
  replay-dlq.sh         # DLQ message replay tooling (NFR-REL-007)
```

**Helm chart structure per service:**

```
/helm/charts/<service-name>/
  Chart.yaml
  values.yaml           # Shared defaults
  values-dev.yaml       # Local Minikube overrides
  values-staging.yaml   # Staging environment overrides
  values-prod.yaml      # Production environment overrides
  templates/
    deployment.yaml
    service.yaml
    configmap.yaml
    hpa.yaml
    serviceaccount.yaml
```

**Why it exists separately:** Infrastructure configuration changes
independently from application code. A Prometheus alert rule fix
must not trigger a rebuild of the Booking Service Docker image.
Helm charts have their own release cadence. Terraform modules are
consumed by all cloud environments. Centralising infrastructure
also gives a single place to verify that `docker compose up` brings
up the entire stack correctly — the runnable invariant (PRD §6)
is owned by this repository.

### 3.3 Tier 2 Repositories — Service Repos

**18 service repos total** (including API Gateway):

| Repository | NFR Tier | Language / Framework | Phase |
|---|---|---|---|
| `stagepass-api-gateway` | T2 | Java / Spring Cloud Gateway | Phase 3 |
| `stagepass-auth-service` | T1 | Java / Spring Boot 3 | Phase 3 |
| `stagepass-event-service` | T2 | TypeScript / NestJS | Phase 3 |
| `stagepass-venue-service` | T2 | TypeScript / NestJS | Phase 4 |
| `stagepass-seat-inventory-service` | T1 | Java / Spring Boot 3 | Phase 4 |
| `stagepass-booking-service` | T1 | Java / Spring Boot 3 | Phase 4 |
| `stagepass-payment-service` | T1 | Java / Quarkus | Phase 4 |
| `stagepass-disbursement-service` | T1 | Java / Spring Boot 3 | Phase 4 |
| `stagepass-ticket-service` | T2 | TypeScript / Fastify | Phase 4 |
| `stagepass-check-in-service` | T2 | TypeScript / Fastify | Phase 4 |
| `stagepass-notification-service` | T2 | JavaScript / Node.js + Socket.IO | Phase 4 |
| `stagepass-waitlist-service` | T3 | TypeScript / NestJS | Phase 4 |
| `stagepass-search-service` | T2 | Python / FastAPI | Phase 5 |
| `stagepass-recommendation-service` | T3 | Python / FastAPI | Phase 5 |
| `stagepass-chatbot-service` | T3 | Python / FastAPI | Phase 5 |
| `stagepass-fraud-detection-service` | T3 | Python / FastAPI | Phase 5 |
| `stagepass-analytics-service` | T3 | Python / Flask | Phase 5 |

**NFR tier classification rationale:**

T1 services are those where unavailability causes immediate revenue
impact or correctness violations with no acceptable fallback:

- **Auth** — every authenticated request depends on key rotation and
  revocation. Without it, the security model collapses.
- **Seat Inventory** — holds are atomic; unavailability means checkout
  is impossible with no degradation path.
- **Booking** — saga coordinator; without it no booking can be created
  or compensated.
- **Payment** — no payment confirmation means no completed booking.
- **Disbursement** — ledger integrity; a corrupt ledger is a financial
  compliance failure.

T2 and T3 services have documented fail-open fallbacks (PRD §8.7,
NFR §8) that keep the core booking flow available when they are
down. Their tier assignment reflects this: degradation is accepted,
but core correctness is not compromised.

**Standard service repo initialisation structure:**

Every service repo is initialised with this exact structure before
any implementation begins:

```
stagepass-<name>-service/
├── .github/
│   ├── workflows/
│   │   └── ci.yml              # Lint → test → SonarQube → build
│   │                           # → Trivy scan → push
│   └── ISSUE_TEMPLATE/
│       ├── feature.yml
│       └── bug.yml
├── src/                        # Application source code
│   └── test/
│       └── integration/        # Testcontainers integration tests
├── pact/                       # Pact consumer/provider contracts
├── Dockerfile                  # Multi-stage, distroless/slim,
│                               # non-root user
├── docker-compose.yml          # Standalone local dev only
│                               # (does NOT depend on infra repo)
├── README.md                   # What it does, how to run (<30 min)
├── CHANGELOG.md                # Semantic versioning entries
└── .gitignore
```

**Service repo README mandatory sections:**

1. What this service does (one paragraph, plain English)
2. NFR tier (T1/T2/T3) and SLO target
3. How to run locally (exact commands, runnable from clean clone
   in under 30 minutes — NFR-MAINT-004)
4. Dependencies (which services it calls; which call it)
5. Environment variables (names only; values come from Vault —
   NFR-SEC-008)
6. Health check endpoints (`/health/live`, `/health/ready`)
7. Link to OpenAPI or AsyncAPI spec in `stagepass-docs`

### 3.4 Tier 3 Repository — Frontend

#### `stagepass-web`

**Purpose:** All frontend application surfaces. Created at Phase 6.

**Internal structure:**

```
stagepass-web/
├── apps/
│   ├── storefront/             # Customer: discovery, booking,
│   │                           # seat map, tickets, waitlist
│   ├── organiser-portal/       # Organiser: event management,
│   │                           # live dashboard, disbursement
│   ├── venue-portal/           # Venue: booking calendar,
│   │                           # layout management, analytics
│   └── admin-dashboard/        # Admin: KYC, fraud review,
│                               # platform health, config
├── packages/
│   ├── ui/                     # Shared Radix UI / shadcn-ui
│   │                           # component library
│   ├── types/                  # Shared TypeScript types
│   └── hooks/                  # Shared React hooks
├── .github/
│   └── workflows/
│       └── ci.yml
└── turrepo.json / pnpm-workspace.yaml
```

**Tooling:** pnpm workspaces + Turborepo for the internal monorepo
within this single repo. This is not a polyrepo within the frontend —
all four surfaces share a build system and component library because
they are all React + TypeScript and have genuine code-sharing needs
(design tokens, auth hooks, API client types). The decision to keep
them in one repo is therefore justified: they share a language, a
build tool, and a component library. This is precisely the condition
under which a monorepo is appropriate.

**Why the frontend is a monorepo-within-polyrepo:** The four surfaces
share Radix UI components, TypeScript types generated from OpenAPI
specs, auth hooks, and design tokens. Splitting them into four
separate repos would require either a private npm package registry
(unavailable on GitHub Free) or copy-pasting shared code. The pnpm
workspace approach gives shared packages without publishing, at the
cost of a single CI pipeline that must be configured per-app.

### 3.5 Authoritative Artifact-to-Repo Mapping

The following table is the definitive reference. Every artifact has
exactly one home. When in doubt, this table is correct.

| Artifact | Repository | Path |
|---|---|---|
| PRD.md | `stagepass-docs` | `/PRD.md` |
| NFR.md | `stagepass-docs` | `/NFR.md` |
| ADR-NNN-*.md | `stagepass-docs` | `/docs/adr/` |
| HLD C4 diagrams | `stagepass-docs` | `/docs/architecture/HLD.md` |
| Sequence diagrams | `stagepass-docs` | `/docs/architecture/sequences/` |
| OpenAPI spec (per service) | `stagepass-docs` | `/docs/api/<service>.yaml` |
| AsyncAPI schema (per topic) | `stagepass-docs` | `/docs/async-api/<topic>.yaml` |
| ER diagram (per service) | `stagepass-docs` | `/docs/er-diagrams/<service>.md` |
| Threat model | `stagepass-docs` | `/docs/threat-model/STRIDE.md` |
| Runbooks (per alert) | `stagepass-docs` | `/docs/runbooks/<alert-name>.md` |
| Protobuf definitions | `stagepass-shared-contracts` | `/proto/stagepass/<service>/v1/` |
| Kafka event schemas | `stagepass-shared-contracts` | `/schemas/events/<domain>/` |
| Docker Compose (full stack) | `stagepass-infrastructure` | `/docker/compose/` |
| Helm chart (per service) | `stagepass-infrastructure` | `/helm/charts/<service>/` |
| Kubernetes manifests | `stagepass-infrastructure` | `/k8s/` |
| Terraform modules | `stagepass-infrastructure` | `/terraform/` |
| Prometheus alert rules | `stagepass-infrastructure` | `/observability/prometheus/rules/` |
| Grafana dashboards | `stagepass-infrastructure` | `/observability/grafana/dashboards/` |
| Utility scripts | `stagepass-infrastructure` | `/scripts/` |
| Service source code | `stagepass-<name>-service` | `/src/` |
| Service Dockerfile | `stagepass-<name>-service` | `/Dockerfile` |
| Service CI pipeline | `stagepass-<name>-service` | `/.github/workflows/ci.yml` |
| Service Pact contracts | `stagepass-<name>-service` | `/pact/` |
| Service unit tests | `stagepass-<name>-service` | `/src/test/` |
| Service integration tests | `stagepass-<name>-service` | `/src/test/integration/` |
| Frontend source (per surface) | `stagepass-web` | `/apps/<surface>/` |
| Frontend CI pipeline | `stagepass-web` | `/.github/workflows/ci.yml` |

### 3.6 Branch Strategy

**Applies to every repository without exception.**

```
main
  Always deployable. Protected by branch rule.
  No direct pushes from any account, including the owner.
  Merges only via PR with CI green.

feature/*     New features and behaviour
  Format: feature/<issue-number>-<short-description>
  Example: feature/23-booking-saga-seat-hold

fix/*         Bug fixes
  Format: fix/<issue-number>-<short-description>

chore/*       Tooling, config, cleanup — no production code change
  Format: chore/<issue-number>-<short-description>

docs/*        Design artifacts (used primarily in stagepass-docs)
  Format: docs/<issue-number>-<short-description>
  Example: docs/5-adr-001-repository-topology

release/*     Release preparation
  Format: release/v<major>.<minor>.<patch>
```

**Merge strategy:** Squash merge only. One commit on main per logical
unit of work. This keeps `git log` on main readable and makes
bisecting failures straightforward.

**Branch protection settings (applied to `main` in every repo):**

```bash
gh api repos/stagepass-aiman/<REPO>/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["ci"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews='{"required_approving_review_count":0}' \
  --field restrictions=null
```

Note: `required_approving_review_count: 0` is the solo-developer
accommodation — PRs still exist and CI still gates, but self-approval
is permitted. The discipline of the PR workflow is the learning
objective, not the approval gate itself.

### 3.7 Tag Formats

**Design documents (`stagepass-docs`):**

```
phase-<N>-<artifact-id>-v<major>.<minor>.<patch>

Examples:
  phase-1-prd-v1.0.0
  phase-1-adr-001-v1.0.0
  phase-1-nfr-v1.0.0
  phase-1-hld-v1.0.0
```

**Shared contracts (`stagepass-shared-contracts`):**

```
contracts-v<major>.<minor>.<patch>

Examples:
  contracts-v1.0.0    (initial: seat_inventory.proto + ticket.proto)
  contracts-v1.1.0    (add booking event schema)
  contracts-v2.0.0    (breaking: new v2 proto directory, migration ADR required)
```

**Service repos:**

```
v<major>.<minor>.<patch>

Examples:
  v0.1.0    (first runnable service stub)
  v1.0.0    (feature-complete for its phase)
  v1.1.0    (post-phase enhancement or fix)
```

**Phase milestone tags (on `stagepass-infrastructure`):**

```
phase-<N>-complete

Examples:
  phase-0-complete
  phase-2-complete
```

### 3.8 Repo Creation Sequence

Repos are created in this exact sequence. No repo is created before
its phase begins.

| Phase | Repos Created |
|---|---|
| Phase 0 | `stagepass-docs`, `stagepass-shared-contracts`, `stagepass-infrastructure` |
| Phase 3 | `stagepass-api-gateway`, `stagepass-auth-service`, `stagepass-event-service` |
| Phase 4 | `stagepass-venue-service`, `stagepass-seat-inventory-service`, `stagepass-booking-service`, `stagepass-payment-service`, `stagepass-disbursement-service`, `stagepass-ticket-service`, `stagepass-check-in-service`, `stagepass-notification-service`, `stagepass-waitlist-service` |
| Phase 5 | `stagepass-search-service`, `stagepass-recommendation-service`, `stagepass-chatbot-service`, `stagepass-fraud-detection-service`, `stagepass-analytics-service` |
| Phase 6 | `stagepass-web` |

The `stagepass-infrastructure` repo is created in Phase 0 but grows
throughout all phases — Helm charts and Kubernetes manifests for
Phase 3 services are added during Phase 3, Phase 4 services during
Phase 4, and so on.

---

## 4. Consequences

### 4.1 Positive Consequences

**True deployment independence.**
Each service repo has its own CI pipeline, its own Docker image, its
own Helm chart, and its own version tag. A change to the Chatbot
Service (T3) produces no CI run, no Docker build, and no deployment
for the Booking Service (T1). This is the architectural property that
makes independent release cadence possible — and it is something that
must be practiced, not just declared.

**Crisp ownership boundaries.**
The question "where does this artifact live?" always has a single
correct answer, given by the mapping table in §3.5. There is no
ambiguity about whether a Helm chart belongs in the service repo or
the infrastructure repo (answer: infrastructure repo, always).
Enforcing this discipline consistently is how the solo developer
avoids the drift that turns a clean codebase into a tangled one.

**Learning is isolated to its domain.**
When implementing the Booking Service Saga pattern, the only repo
touched is `stagepass-booking-service`. The Event Service CI does not
run. The Venue Service tests do not break. Failures are immediately
localised to their source. This also matches how a real team operates:
a backend engineer breaking the Booking Service does not block the
Fraud Detection team's work.

**Polyglot build tool diversity is contained.**
Maven is a Booking Service concern. npm is a Notification Service
concern. Poetry is a Fraud Detection Service concern. These build
tools never appear in the same `pom.xml`, `package.json`, or
`pyproject.toml`. The cognitive model for each service is entirely
its own language's conventions.

**Contracts are strictly versioned and independently auditable.**
The `stagepass-shared-contracts` repo gives a complete history of
every schema change across the platform. When a consumer fails because
of a schema evolution, the blame is precisely attributable to a commit
in `stagepass-shared-contracts`. This is the correct model for schema
governance.

**Observability stack is centrally managed.**
Prometheus alert rules, Grafana dashboards, and Loki configuration
live in `stagepass-infrastructure` — not scattered across service
repos. A new alert for the Booking Service DLQ can be added, reviewed,
and deployed without touching the Booking Service repo or redeploying
the service.

### 4.2 Negative Consequences and Mitigations

**Context switching overhead.**
Working across 21 repos requires `cd` commands, repo clones, branch
management, and multiple terminal sessions. Mitigation: a consistent
directory structure (`~/stagepass/<repo-name>`) and shell aliases
reduce the friction to a few keystrokes.

**Cross-repo changes require coordination.**
A gRPC contract change requires PRs in `stagepass-shared-contracts`,
`stagepass-booking-service`, and `stagepass-seat-inventory-service` —
three PRs, three CI runs, three merges. Mitigation: the shared
contracts repo is versioned with Git tags; services pin a specific
tag and upgrade explicitly. Cross-repo changes are a feature, not a
bug — they make the impact of schema changes visible.

**No private package registry on GitHub Free.**
Services cannot consume shared contracts as compiled npm packages or
Maven artifacts. They must reference source via Git tag and generate
stubs at build time. Mitigation: each service CI pipeline generates
gRPC stubs from the pinned `.proto` files at build time using the
appropriate `protoc` plugin. The build step is documented in each
service README. This is a real-world constraint that many teams face;
experiencing it is part of the learning.

**Solo developer must maintain 21 repos.**
Each repo needs its own CI pipeline, README, CHANGELOG, branch
protection, and labels. Mitigation: a `scripts/init-repo.sh` script
in `stagepass-infrastructure` handles repetitive initialisation
(branch protection, labels, issue templates, PR template) via the
GitHub CLI. The script is idempotent and accepts a repo name argument.

**Claude Project context must span repos.**
Because artifacts are spread across repos, the Claude Project
knowledge base must be kept current. Mitigation: the step completion
protocol requires uploading every document-class artifact to the
knowledge base after merge.

### 4.3 Solo Developer Cost Summary

| Cost | Frequency | Management |
|---|---|---|
| Clone a new service repo | Once per service, at phase start | `gh repo clone` — 10 seconds |
| Create branch + issue + PR | Every work session | gh CLI + step completion protocol — < 2 min |
| Update Helm chart for new service | Once per service | Templated from a base chart — ~30 min |
| Add Prometheus scrape config | Once per service | Copy-paste pattern — ~5 min |
| Upload artifact to Claude Projects | After every doc merge | Manual — ~1 min per artifact |

After Phase 3 the patterns are fully internalised and the overhead
drops to near-zero per step.

---

## 5. Alternatives Considered

### 5.1 Monorepo with Nx or Turborepo

**Structure:** All 17 service repos and the frontend in a single
repository managed with Nx (polyglot support) or Turborepo
(TypeScript-focused).

**Arguments in favour:**
- Single `git clone` and `git log`
- Atomic cross-service commits
- Nx affected analysis runs CI only for changed services
- Easier refactoring when boundaries change

**Why rejected:**

*Polyglot tool conflict.* Nx and Turborepo are Node.js-based build
orchestrators. Integrating Maven (Java), pip (Python), and npm (Node)
into a single Nx workspace requires custom executors and plugin
development. This is significant tooling work that contributes nothing
to learning distributed systems. It would front-load weeks of build
tooling before a single service is written.

*Deployment isolation is harder to teach.* A monorepo with Nx can
achieve independent deployment via `nx affected`, but the dependency
analysis is implicit — a developer trusts the tool to determine what
changed. In a polyrepo, the isolation is explicit and structural. A CI
run on `stagepass-booking-service` cannot accidentally include
`stagepass-notification-service` because they are in different repos.
The structural enforcement is the learning.

*False sense of atomicity.* Atomic cross-service commits in a monorepo
feel clean but obscure the distributed systems reality: in production,
you cannot atomically deploy two services. The discipline of separate
PRs, separate merges, and separate deployments is the correct mental
model for operating distributed systems. Teaching it from day one
prevents the tight-coupling reflex.

*GitHub Free plan limitations.* Nx Cloud (for distributed CI caching)
requires a paid plan. Without it, Nx in a large polyglot monorepo means
every CI run rebuilds everything.

**Verdict: Rejected.**

### 5.2 Hybrid — Shared Contracts Monorepo + Service Polyrepo

**Structure:** `stagepass-shared-contracts` contains both `.proto`
files and all service source code, with infrastructure and frontend
in separate repos.

**Arguments in favour:**
- Reduces repo count from 21 to 4–5
- Shared utility code can live alongside contracts without a publish step
- Proto files and their consuming service code can be updated atomically

**Why rejected:**

*Mixes contract ownership with implementation.* If Booking Service
source code lives in the same repo as the `.proto` file it produces
for Seat Inventory, then Seat Inventory's build process is entangled
with Booking's source tree. A bad commit in Booking breaks Seat
Inventory's CI even though no contract changed.

*Shared utility code is a coupling trap.* Once services can import from
each other's directories — even "utility" code — the coupling grows
organically. In a real codebase, once this path exists it is used. The
whole point of service separation is that cross-service imports must be
impossible at the build-tool level.

*Contract evolution and service code evolution have different cadences.*
Contracts should change slowly and deliberately, with backward
compatibility as the primary constraint. Service implementation code
changes frequently. Mixing them means every service commit goes through
schema governance and every schema change goes through the faster
service CI. The incentives are misaligned.

**Verdict: Rejected.**

### 5.3 Single-Repo-Per-Phase

**Structure:** One repo per project phase — e.g., `stagepass-phase-3`,
`stagepass-phase-4` — each containing all services built in that phase.

**Arguments in favour:**
- Dramatically fewer repos (10 instead of 21)
- Phase-scoped CI and deployment are simpler
- All services within a phase can share code if needed

**Why rejected:**

*Services outlive phases.* Auth Service (Phase 3) is still deployed
and maintained in Phase 7, 8, and 9. Every post-Phase-3 change to Auth
would require a PR in a repo named after a completed phase. The naming
is semantically wrong and the CI context is polluted with unrelated
services.

*Cross-phase dependencies create a monorepo by accident.* In Phase 4,
the Booking Service calls the Seat Inventory Service via gRPC. If both
are in `stagepass-phase-4`, the temptation to import shared code
directly (rather than via the contracts repo) is irresistible.

*Deployment granularity is lost.* To deploy only the Disbursement
Service fix, a CI pipeline must target a single service within the
phase repo — reintroducing Nx/Turborepo-style tooling with none of
its benefits.

**Verdict: Rejected.**

---

## 6. References

| Source | Relevance |
|---|---|
| PRD.md §4 — Phase Roadmap | Defines which services are built in which phase; drives just-in-time repo creation schedule |
| PRD.md §7 — GitHub Organisation Structure | Primary source this ADR formalises; all repo names, paths, and mapping tables originate here |
| PRD.md Constraint C-01 | Solo developer constraint; drives mitigation strategies in §4.2 |
| PRD.md Constraint C-05 | Stack is locked; topology must accommodate Java + TypeScript + Python in isolation |
| PRD.md Constraint C-06 | Polyglot by design; polyrepo provides the build-tool isolation that makes this workable |
| NFR.md §3 — Service Tier Map | T1/T2/T3 assignments drive isolation rationale; T1 services must have fully independent CI and deployment |
| NFR-MAINT-004 | 30-minute local setup target; drives README requirements for service repos |
| NFR-MAINT-005 | ADR-before-implementation rule; this ADR satisfies it for the topology decision |
| NFR-SEC-008 | Secrets in Vault only; service READMEs document env var names, never values |
| NFR-REL-007 | DLQ tooling; `replay-dlq.sh` lives in infrastructure repo scripts |
| NFR-PERF-043 | Full stack < 12 GB; enforced by `check-memory.sh` in infrastructure repo |
| Newman — Building Microservices (2nd ed.), Ch. 3 | Service ownership and repository boundaries in a microservices organisation |
| Kleppmann — DDIA, Ch. 1 | Reliability, scalability, maintainability as first-class concerns; isolation per service supports independent reliability management |
| Martin Fowler — "MonolithFirst" (martinfowler.com) | Context for when monorepo-to-polyrepo migration is appropriate; we begin polyrepo because service boundaries are already known |

---

## 7. Quick Self-Check

Answer these three questions before creating the first repository.
If you cannot answer them clearly, re-read this ADR and the PRD
sections it references before proceeding.

---

**Q1: A new requirement means adding a shared utility function that
both the Booking Service and the Disbursement Service need. Where
does it go, and why does your answer rule out placing it in
`stagepass-shared-contracts`?**

`stagepass-shared-contracts` owns interface contracts — the shape of
communication between services (Protobuf definitions, Kafka schemas).
A shared utility function is application logic, not a communication
contract. The correct answer is that each service duplicates the small
utility, or the function is documented as a coding convention in
`stagepass-docs`. The polyrepo model intentionally accepts some
duplication to preserve isolation. If the function is complex enough
to justify sharing across languages, that is a signal to reconsider
service boundaries — not to break the contract-repo isolation.

---

**Q2: The Seat Inventory Service needs a breaking change to its gRPC
Protobuf definition. Walk through every repo and file that needs to
change, and identify which change must be made first.**

1. **First** — open a PR in `stagepass-shared-contracts` adding a
   `/proto/stagepass/seat_inventory/v2/` directory with the new
   `.proto` file. Tag `contracts-v2.0.0`. Write or reference a
   migration ADR justifying the break.
2. Update `stagepass-booking-service` to generate stubs from
   `seat_inventory/v2` and update its gRPC client code. This service
   is the consumer of the contract.
3. Update `stagepass-seat-inventory-service` to implement the v2
   service definition.
4. Both service repos pin `contracts-v2.0.0` in their build
   configuration.
5. Update Helm charts in `stagepass-infrastructure` if the service's
   routing rules or environment configuration changes.

The shared-contracts repo change comes first because services are
consumers of a contract they do not own.

---

**Q3: The just-in-time repo creation rule says not to create service
repos before their phase. A colleague suggests creating all 21 repos
upfront so the CI pipelines are ready. What is the concrete risk, and
what is the rule to follow instead?**

Creating 21 repos upfront with empty or stub CI pipelines creates
repositories with failing or trivially-passing pipelines because there
is nothing to test. Branch protection rules on repos with no code can
create false confidence. When the real service code arrives, the CI
pipeline may need significant rework to match the actual build tool and
test framework. The correct rule: create each service repo at the start
of the phase that implements it, initialise it with the standard
structure (§3.3), write the actual CI pipeline immediately, and run it
against the actual service from the first commit. The standard structure
and this ADR give everything needed to initialise correctly at the right
moment.
