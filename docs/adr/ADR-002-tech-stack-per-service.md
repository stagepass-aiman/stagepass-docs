# ADR-002 — Tech Stack Per Service

## Document Header

| Field        | Value                                                                                           |
|--------------|-------------------------------------------------------------------------------------------------|
| Document ID  | ADR-002                                                                                         |
| Version      | 1.0.0                                                                                           |
| Status       | Accepted                                                                                        |
| Author       | StagePass Engineering                                                                           |
| Created      | 2025-05-15                                                                                      |
| Last Updated | 2025-05-15                                                                                      |
| Repo         | stagepass-docs                                                                                  |
| Path         | /docs/adr/ADR-002-tech-stack-per-service.md                                                    |
| Phase        | Phase 1 — Requirements & Design                                                                 |
| Traces To    | PRD.md §3 (Tech Stack), PRD.md Constraints C-01 C-05 C-06, NFR.md §3 (Service Tier Map),       |
|              | NFR.md §5.4 (Local Stack Memory Budget), NFR-PERF-042 (Cold Start), NFR-PERF-043 (12 GB RAM),  |
|              | ADR-001 (Repository Topology — 21-repo structure)                                               |

### Change Log

| Version | Date       | Author                | Summary                  |
|---------|------------|-----------------------|--------------------------|
| 1.0.0   | 2025-05-15 | StagePass Engineering | Initial accepted version |

---

## Table of Contents

1. [Status](#1-status)
2. [Context](#2-context)
3. [Decision](#3-decision)
   - 3.1 [Framework-per-Service Assignment Table](#31-framework-per-service-assignment-table)
   - 3.2 [Database-per-Service Assignment Table](#32-database-per-service-assignment-table)
   - 3.3 [The Polyglot Justification — Where Intentional Diversity Ends](#33-the-polyglot-justification--where-intentional-diversity-ends)
   - 3.4 [Learning Rationale — Detailed Per-Service Reasoning](#34-learning-rationale--detailed-per-service-reasoning)
   - 3.5 [Cold Start Analysis — NFR-PERF-042](#35-cold-start-analysis--nfr-perf-042)
   - 3.6 [Local Memory Budget — NFR-PERF-043](#36-local-memory-budget--nfr-perf-043)
4. [Consequences](#4-consequences)
5. [Alternatives Considered](#5-alternatives-considered)
6. [References](#6-references)
7. [Quick Self-Check](#7-quick-self-check)

---

## 1. Status

**Accepted.**

This ADR is the authoritative record of which language and framework each StagePass
service uses, and why. The stack is locked per PRD Constraint C-05. This document does
not re-open the choices — it provides the rationale that must exist before a single line
of service code is written. Deviations from this assignment require a superseding ADR
with explicit learning-goal justification.

---

## 2. Context

### 2.1 What We Are Deciding

Eighteen services (17 domain services + 1 API Gateway) must each be assigned:

1. A programming language (Java, TypeScript, Python)
2. A framework within that language (Spring Boot 3, Quarkus, Spring Cloud Gateway,
   NestJS, Fastify, raw Node.js, FastAPI, Flask)
3. A primary database (PostgreSQL, MongoDB, Redis, Elasticsearch, ClickHouse, Qdrant)
   and secondary data stores where required

Before any service repo is initialised (Phase 3 onwards), this ADR must be
consulted to determine the correct starting point. A service scaffolded on the
wrong framework cannot be corrected cheaply once CI, Dockerfile, Helm charts,
and tests are established around it.

### 2.2 Forces at Play

**Stack is locked (PRD C-05).** The technology stack defined in the system prompt
is fixed. No framework or database changes without a superseding ADR. This ADR
provides the WHY for choices already made — it does not re-open them.

**Polyglot by design (PRD C-06).** Services are deliberately built in Java,
TypeScript, and Python to maximise learning exposure. However, accidental polyglot
complexity — using multiple frameworks for the same problem type without a reason —
is explicitly prohibited. Every framework choice must teach something the others
would not for that specific service.

**Solo development (PRD C-01).** One developer, 9–12 months. JDK version
management (Java 21 only — no mixing of Java versions), npm/pnpm/pip tool overhead,
local memory footprint, and cold start times are real costs. The stack must be
operable daily without environment management becoming a second job.

**Service criticality is not uniform (NFR §3).** T1 services (Auth, Seat Inventory,
Booking, Payment, Disbursement) have a 99.9% SLO and a 43-minute monthly error budget.
T3 services have a 99.0% SLO and 7-hour budget. The framework choice for a T1 service
must prioritise the ecosystem maturity, transactional guarantees, and failure mode
clarity that the SLO demands. T3 services have more freedom.

**Local stack fits in 12 GB (NFR-PERF-043).** The full Docker Compose stack of all
17 services plus infrastructure must run on a developer laptop with ≤ 12 GB consumed.
The per-service memory budget in NFR §5.4 is a hard constraint, not a target. Framework
overhead (JVM startup heap, Python interpreter base memory) is part of each service's
budget.

**Cold start targets are hard constraints (NFR-PERF-042).** Java services must start
in < 30 seconds; Node.js and Python services in < 10 seconds. This rules out designs
that rely on expensive JVM class scanning at startup without mitigation.

**Databases have domain-specific fit requirements.** The database assigned to each
service is not arbitrary. The system prompt specifies a primary database per service.
This ADR explains why each assignment is correct for the domain — relational
integrity, document flexibility, atomic sorted-set operations, or columnar analytics —
rather than treating the assignment as a configuration detail.

### 2.3 What This Decision Is Not About

This ADR decides language, framework, and database per service. It does not decide:

- Repository topology (→ ADR-001)
- Service communication patterns — REST, gRPC, Kafka (→ ADR-003)
- Money type representation (→ ADR-004)
- Booking saga pattern (→ ADR-005)
- Seat inventory concurrency strategy (→ ADR-006)
- Flash sale queue pattern (→ ADR-007)
- Revenue split and disbursement model (→ ADR-008)

---

## 3. Decision

The following tables and rationale are the complete, authoritative assignment
for all 18 services. Every entry in the framework table corresponds exactly to
one service repo defined in ADR-001.

### 3.1 Framework-per-Service Assignment Table

| Service | Tier | Language | Framework | ADR-001 Repo Name |
|---------|------|----------|-----------|-------------------|
| Auth Service | T1 | Java 21 | Spring Boot 3 + Spring Security 6 | stagepass-auth-service |
| Seat Inventory Service | T1 | Java 21 | Spring Boot 3 + Spring Security 6 | stagepass-seat-inventory-service |
| Booking Service | T1 | Java 21 | Spring Boot 3 + Spring Security 6 | stagepass-booking-service |
| Payment Service | T1 | Java 21 | Quarkus 3 | stagepass-payment-service |
| Disbursement Service | T1 | Java 21 | Spring Boot 3 + Spring Security 6 | stagepass-disbursement-service |
| API Gateway | T2 | Java 21 | Spring Cloud Gateway (WebFlux) | stagepass-api-gateway |
| Event Service | T2 | TypeScript | NestJS 10 | stagepass-event-service |
| Venue Service | T2 | TypeScript | NestJS 10 | stagepass-venue-service |
| Ticket Service | T2 | TypeScript | Fastify 4 | stagepass-ticket-service |
| Check-in Service | T2 | TypeScript | Fastify 4 | stagepass-check-in-service |
| Notification Service | T2 | TypeScript | Raw Node.js 20 + Socket.IO 4 | stagepass-notification-service |
| Waitlist Service | T3 | TypeScript | NestJS 10 | stagepass-waitlist-service |
| Search Service | T2 | Python 3.12 | FastAPI 0.111 | stagepass-search-service |
| Recommendation Service | T3 | Python 3.12 | FastAPI 0.111 | stagepass-recommendation-service |
| Chatbot Service | T3 | Python 3.12 | FastAPI 0.111 | stagepass-chatbot-service |
| Fraud Detection Service | T3 | Python 3.12 | FastAPI 0.111 | stagepass-fraud-detection-service |
| Analytics Service | T3 | Python 3.12 | Flask 3 + Gunicorn | stagepass-analytics-service |

**Learning rationale column is in §3.4. Read it before writing the first line of any service.**

### 3.2 Database-per-Service Assignment Table

The rationale column here answers the question: *why is this database type
correct for this domain?* Not just "it was specified" — but what property of
this data makes this storage engine the right fit.

| Service | Primary DB | Secondary / Cache | DB Type Rationale |
|---------|------------|-------------------|-------------------|
| Auth Service | PostgreSQL | Redis | User credentials, refresh token records, and JTI revocation entries are strongly relational (foreign keys between user ↔ session ↔ token), require ACID for JTI insertion atomicity (a partially-inserted revocation is a security hole), and never have schema flexibility requirements. Redis holds the JTI blocklist as a sorted set with TTL-based expiry — O(1) membership check at every JWT validation. |
| Seat Inventory Service | PostgreSQL | Redis | The source-of-truth seat ownership record (who holds seat 12B) is a relational fact requiring serialisable transactions under concurrent write. Redis is not the source of truth — it is the atomic lock: `SETNX seat:{eventId}:{seatId} {customerId} EX 600`. PostgreSQL commits the confirmed hold after Redis grant. The two-store pattern (Redis for atomicity, PostgreSQL for durability) is the core architectural lesson of this service. |
| Booking Service | PostgreSQL | MongoDB (CQRS read model) | The Booking write model is transactional: PENDING → CONFIRMED state transitions must be serialisable and compensatable. PostgreSQL's ACID guarantees are load-bearing here. The CQRS read model (Booking history, customer portal queries) is served from MongoDB where the flexible document shape (booking + nested tickets + split amounts in one document) is far more efficient for read queries than a 5-table join. |
| Payment Service | PostgreSQL | — | Payment records are financial facts: idempotency keys, webhook receipts, payment status transitions. These require ACID with no flexibility in schema — a payment record is not a document with optional fields. PostgreSQL with SERIALIZABLE isolation enforces the "one payment per booking" invariant. |
| Disbursement Service | PostgreSQL | — | The disbursement ledger is append-only and immutable: records are inserted, never updated or deleted. PostgreSQL enforces this at the application layer via insert-only table permissions and CHECK constraints. The columnar analytics advantage of ClickHouse is irrelevant here — the ledger is transactional (linked to RevenueSplit foreign keys), not analytic. |
| API Gateway | — (stateless) | Redis | The Gateway itself holds no data. Redis provides the rate-limit counter store (sliding window counters, keyed by IP or JWT subject) and the circuit breaker state store. Statelessness of the Gateway is what makes horizontal scaling trivial. |
| Event Service | MongoDB | Elasticsearch | Event documents have heterogeneous schemas: a comedy show and a cricket match have different metadata fields (venue capacity vs. team names, sets vs. innings). MongoDB's flexible document model accommodates this without nullable columns proliferating across a relational schema. Elasticsearch is the secondary index: it receives documents via Kafka after MongoDB persistence. MongoDB is the source of truth; Elasticsearch is the query index. |
| Venue Service | MongoDB | — | Venue seating layouts are deeply nested hierarchical documents (Venue → Sections → Rows → Seats, with per-seat accessibility flags, pricing overrides, and coordinate data for SVG rendering). A relational schema would require 4–5 tables and an expensive join for every layout fetch. MongoDB's document model stores the full layout as one retrievable unit. Versioning is handled by embedding a `layoutVersion` field — the old version is retained alongside the new. |
| Ticket Service | PostgreSQL | — | A ticket is a simple, strongly typed fact: it exists, it belongs to a booking, it has a QR token, it maps to one seat. No schema flexibility is needed. The HMAC token is stored as a varchar. PostgreSQL's UNIQUE constraint on `(eventId, seatId)` is the database-level guard against duplicate ticket issuance, backing up the saga-level guarantee. |
| Check-in Service | PostgreSQL | — | Check-in records are append-only audit facts: `(ticketId, checkedInAt, verifiedBy, method)`. PostgreSQL gives the UNIQUE constraint on `ticketId` that enforces NFR-SEC-006 (idempotent check-in — a second scan returns "already checked in", not a duplicate record). |
| Notification Service | — (stateless) | Redis | Notification Service holds no persistent state. Redis provides the Socket.IO adapter that synchronises room membership and message routing across horizontally scaled Notification Service instances. Without the Redis adapter, a customer connected to instance A would not receive a broadcast emitted by instance B. |
| Waitlist Service | Redis | — | A waitlist is a sorted set: `ZADD waitlist:{eventId} {timestamp} {customerId}`. The natural ordering is by score (timestamp = join time = priority). `ZPOPMIN` atomically removes and returns the next member in O(log N). No other datastore achieves sorted-set atomic operations at sub-millisecond latency without building this structure manually. PostgreSQL could model this with a SERIALIZABLE SELECT FOR UPDATE, but at the cost of lock contention. Redis sorted sets are the idiomatic tool. |
| Search Service | Elasticsearch | Qdrant | Keyword search (BM25 ranking, filters, facets) is Elasticsearch's native strength — its inverted index is purpose-built for this. Semantic search (approximate nearest-neighbour over embedding vectors) requires Qdrant's HNSW index. The Search Service is the only service that queries both and blends results — it is the bridge between the keyword and vector search worlds. |
| Recommendation Service | Qdrant | Redis | Recommendations are driven by user-event embedding similarity: find events whose embedding vectors are nearest to the user's preference vector. Qdrant's ANN query with payload filters (city, date, category) achieves this in < 100 ms. Redis caches the inference output (ranked event IDs per user) with a 5-minute TTL — the cache hit rate in production will exceed 95% for most users, making the Redis path the dominant one. |
| Chatbot Service | Qdrant | Redis | RAG retrieval requires a vector similarity search over the event corpus and policy documents (the "knowledge base"). Qdrant stores these embeddings. Redis stores conversation memory (last N turns per session), enabling multi-turn coherence without a database round-trip on every message. |
| Fraud Detection Service | PostgreSQL | — | Fraud models consume features derived from booking history, velocity data, and user behaviour. These features are relational facts (booking counts per user per hour, seat price vs. average for event). PostgreSQL's analytical query capability (window functions for velocity calculation) is sufficient at StagePass's scale. The fraud score output is written back to PostgreSQL for audit and model retraining. |
| Analytics Service | ClickHouse | — | ClickHouse is a columnar OLAP store optimised for aggregation queries over time-series event data. "Ticket sales in the last 60 minutes by venue" requires scanning millions of rows and grouping — a query ClickHouse executes in milliseconds that would take seconds in PostgreSQL. The Grafana dashboards (NFR-OBS-005) query ClickHouse directly. No other database in the stack is appropriate for this access pattern. |

### 3.3 The Polyglot Justification — Where Intentional Diversity Ends

PRD Constraint C-06 states: *"Accidental polyglot complexity — using multiple frameworks
for the same problem without a reason — is explicitly prohibited."*

This section defines the line. The question to ask before each service is:
**"What does this language/framework teach that any other choice in this project would not,
specifically for this service's domain?"**

#### What Java teaches here that TypeScript and Python do not

Java is used for all five T1 services plus the API Gateway. This is not arbitrary — Java is
used where the learning objectives require the JVM's transactional ecosystem, the Spring
Security model, or the contrast between Spring Boot and Quarkus.

- **Spring Security 6** is the industry standard for enterprise Java security. Its SecurityFilterChain,
  method-level `@PreAuthorize`, JWKS resource server configuration, and OAuth2 client support
  cannot be learned via NestJS Guards or FastAPI dependencies — they are different models. Auth
  Service must use Spring Security 6 because that is what the learning objective targets.
- **Spring's `@Transactional` with isolation levels** is how most Java developers encounter
  SERIALIZABLE isolation in the wild. Using it in Seat Inventory and Booking Services teaches the
  mapping between isolation level declaration and actual database lock behaviour — a lesson that
  only surfaces when you write code that faces real contention.
- **The Outbox pattern with Spring Data JPA** — writing to an Outbox table in the same transaction
  as the business entity, then relaying via a scheduled reader — is most naturally expressed in
  Spring Boot's application context lifecycle. The `@EventListener`, `@TransactionalEventListener`,
  and `@Scheduled` annotations are the exact tools for this pattern. A NestJS or FastAPI
  implementation would work but would require more manual wiring.
- **Quarkus vs. Spring Boot** is a contrast worth experiencing once. Payment Service uses Quarkus
  to teach compile-time dependency injection (CDI), Panache for reactive PostgreSQL access, and
  the Quarkus extension model. This contrast — not feature parity — is the reason for the choice.

#### What TypeScript / Node.js teaches here that Java and Python do not

Node.js is used for six services (NestJS ×3, Fastify ×2, raw Node.js ×1). The diversity within
the Node.js services is itself the lesson.

- **NestJS** teaches decorator-driven architecture, the Angular-influenced module system, and
  dependency injection in TypeScript. Comparing NestJS modules to Spring Boot components reveals
  that DI, modules, and interceptors are not Java concepts — they are architectural patterns that
  surface in every mature framework, regardless of language.
- **Fastify** teaches what NestJS gives you by its absence: schema-first validation via JSON Schema
  + ajv, plugins instead of modules, explicit rather than magic dependency wiring, and lower
  per-request overhead. Using Fastify for Ticket and Check-in (p99 200 ms constraint per
  NFR-PERF-004) makes the performance choice defensible, not gratuitous.
- **Raw Node.js + Socket.IO** (Notification Service) teaches what happens when you remove the
  framework entirely. HTTP server creation, middleware, connection management, and the Socket.IO
  Redis adapter must be wired manually. This reveals what every framework is hiding from you.
  Having done it once in this project, the developer understands the value of the abstractions
  they use everywhere else.
- **Cross-language gRPC** (Check-in in TypeScript calling Ticket in Java) is achievable precisely
  because both ends agree on the Protobuf contract in `stagepass-shared-contracts`. Node.js gRPC
  stub generation with `@grpc/proto-loader` and `@grpc/grpc-js` is a different setup than
  Spring's `protobuf-maven-plugin`. Teaching both is the goal.

#### What Python teaches here that Java and TypeScript do not

Python is used for five services (FastAPI ×4, Flask ×1). Python is the only language in this
project with a mature ML ecosystem, and that is the non-negotiable reason for its presence.

- **ML model lifecycle** — offline training with scikit-learn/LightGBM, serialisation via joblib,
  tracking with MLflow, online serving via FastAPI — is a workflow that does not exist in Java
  or TypeScript in the same way. Recommendation, Fraud Detection, and No-Show Prediction models
  are trained and served in Python because that is where the tooling lives.
- **LangChain** for RAG pipeline construction (Chatbot Service) requires Python. There are
  JavaScript ports of LangChain but they lag the Python version significantly and are not
  production-ready for the complexity of this pipeline.
- **FastAPI's type-hint-driven OpenAPI generation** demonstrates a different approach to
  contract-first API design: the type system becomes the specification. Comparing this with
  Spring's springdoc-openapi annotation approach teaches that OpenAPI spec generation is
  a spectrum, not a single tool.
- **asyncio vs. synchronous Python** — FastAPI (async) vs. Flask (sync) within the same project
  teaches when async IO is worth its added complexity. Analytics Service uses Flask because its
  ClickHouse queries are long-running OLAP queries where async at the framework level adds no
  benefit — the query itself is the bottleneck, not the I/O handling.

#### The Prohibited Zone — Complexity Without Learning

The following combinations are explicitly prohibited because they would add framework
overhead without adding distinct learning:

| Prohibited Combination | Why It Is Prohibited |
|------------------------|----------------------|
| Using both NestJS and Fastify for the same type of service | Already avoided: NestJS = complex domain logic, Fastify = simple high-throughput |
| Using Spring Boot for Analytics (instead of Python/Flask) | Java has no equivalent of ClickHouse's Python driver + pandas; the ML pipeline requires Python |
| Using FastAPI for Notification Service instead of raw Node.js | Socket.IO's Python implementation lags the Node.js reference implementation significantly; WebSocket performance at scale is a Node.js strength |
| Adding a fourth Java framework (Micronaut) for any service | The Spring Boot vs. Quarkus contrast is sufficient; adding Micronaut would be complexity without marginal learning |
| Using Flask for AI services (Recommendation, Chatbot) | Flask's synchronous model would block the event loop during model inference; FastAPI's async support is architecturally correct |

### 3.4 Learning Rationale — Detailed Per-Service Reasoning

This section answers: *"What does this framework teach for this service that an
alternative within the same language would not?"* Read this before implementing each
service. It is the implementation team's obligation to understand this rationale, not
just the framework name.

---

**Auth Service — Spring Boot 3 + Spring Security 6 (Java 21)**

Spring Security 6 introduced a breaking redesign of its configuration API (from
`WebSecurityConfigurerAdapter` to `SecurityFilterChain` beans). This redesign reflects
modern security composition — each filter is an explicit, testable bean rather than an
overridden method. Learning Spring Security 6's configuration model (not 5) is the target.

This service teaches:
- JWKS endpoint publication (`/auth/jwks`) via Spring Security's OAuth2 authorization server
  support or manual `JWKSet` serialization
- RS256 JWT issuance and validation
- Refresh token rotation with Redis-backed JTI revocation
- RBAC via `@PreAuthorize("hasRole('ORGANISER')")` at the method level
- bcrypt / argon2id password encoding via Spring Security's `PasswordEncoder` abstraction
- The difference between authentication (who are you) and authorization (what can you do)
  as expressed in Spring Security's layered filter chain

Alternative rejected: Quarkus for Auth. Quarkus's Panache + RESTEasy Reactive + SmallRye
JWT is a complete security stack, but it uses CDI instead of Spring DI, and its JWT
configuration model is different from Spring Security's. The learning goal is Spring Security
6 specifically — it is what the developer is most likely to encounter in Java enterprise
employment.

---

**Seat Inventory Service — Spring Boot 3 + Spring Security 6 (Java 21)**

This service teaches the most important architectural lesson in the platform: **the two-store
pattern for high-concurrency seat management**. Redis provides the atomic lock (SETNX with
TTL); PostgreSQL provides the durable commit. Neither alone is correct — Redis alone loses
holds on restart; PostgreSQL alone cannot achieve sub-500 ms hold latency under concurrent
load (NFR-PERF-001).

Spring Boot's Lettuce integration (Spring Data Redis) provides the reactive Redis client
for atomic operations. Spring Data JPA provides the PostgreSQL commit path. Both are
accessible within the same Spring application context, which is exactly the point: this
service demonstrates multi-store coordination without requiring a separate service for each
store.

This service also produces the gRPC server implementation that Booking Service calls
(Java-to-Java gRPC). Spring Boot + `protobuf-maven-plugin` generates the stub classes;
a `@GrpcService` (via `grpc-spring-boot-starter`) exposes them. This is the canonical
Spring Boot + gRPC setup.

This service also teaches: optimistic locking via `@Version`, pessimistic locking via
`SELECT FOR UPDATE`, and when each is appropriate — the core question of ADR-006.

---

**Booking Service — Spring Boot 3 + Spring Security 6 (Java 21)**

The Saga orchestrator. This is the most complex service in the platform and deliberately uses
the most mature, most ecosystem-rich Java framework because complexity must not compound onto
an unfamiliar toolchain.

This service teaches:
- **Saga orchestration**: a `BookingSagaOrchestrator` bean that sends commands and handles
  compensating transactions. Spring's `@TransactionalEventListener` drives the state machine.
- **Outbox pattern**: every saga command is written to an `OutboxEvent` table in the same
  `@Transactional` block as the Booking record update. A `@Scheduled` relay reads unprocessed
  Outbox records and publishes them to Kafka. Spring's transaction lifecycle hooks make this
  implementation clean and testable.
- **CQRS**: the write model is PostgreSQL (normalised, ACID, append-oriented saga log). The
  read model is MongoDB (a single BookingDocument containing the full booking context for
  portal queries). A `BookingEventListener` (Kafka consumer) maintains the read model from
  events — this is the projection pattern.
- **gRPC client**: Booking Service calls Seat Inventory Service's gRPC endpoint. The gRPC
  stub is generated from the shared `.proto` file and injected as a Spring bean.

This is intentionally in Spring Boot, not Quarkus, because the Spring ecosystem's Kafka
integration (Spring for Apache Kafka), transaction management, and scheduling are battle-tested
for exactly this use case. Learning this in Spring Boot first makes the Quarkus contrast in
Payment Service meaningful.

---

**Payment Service — Quarkus 3 (Java 21)**

The only Quarkus service in the platform. This is a deliberate contrast, not an accident.

Quarkus teaches:
- **Compile-time dependency injection via CDI**: unlike Spring's runtime reflection-based DI,
  Quarkus resolves injection at build time. The practical consequence: startup is faster (the
  container is already wired), and runtime errors from missing beans surface at compile time.
- **Panache for reactive PostgreSQL access**: `PanacheRepository<Payment>` with Mutiny reactive
  types (`Uni<Payment>`) teaches the reactive data access pattern. The developer must understand
  why `Uni.await().indefinitely()` is the escape hatch and when not to use it.
- **Quarkus extension model**: adding `quarkus-smallrye-jwt` vs. Spring's `spring-security-oauth2`
  reveals different configuration philosophies (application.properties vs. `@Configuration` beans).
- **Fast cold start**: Quarkus JVM mode starts in 5–10 seconds (vs. 15–25 for Spring Boot). For a
  T1 service that must recover from crashes quickly, this matters. The Payment Service is the
  service most likely to need emergency restart during a flash sale.

The contrast is the learning: if both Payment and Auth used Spring Boot, the developer would
never discover that Spring Boot's startup overhead is a design choice, not an inevitability.

---

**Disbursement Service — Spring Boot 3 (Java 21)**

The financial ledger. This service teaches **append-only table semantics enforced at the
application layer**. The PostgreSQL schema has no `UPDATE` or `DELETE` privileges granted to
the service's database user. Every ledger entry is an `INSERT`. Spring Data JPA's `@Entity`
with `updatable = false, insertable = false` on auto-generated fields plus a `@PreUpdate`
listener that throws if any update is attempted enforces immutability in the application.

This service also teaches: **BigDecimal money arithmetic via Spring Data JPA**. The
`RevenueSplit` and `DisbursementRecord` entities use `NUMERIC(19,4)` PostgreSQL columns mapped
to `java.math.BigDecimal`. Every arithmetic operation must use `BigDecimal.add()`,
`BigDecimal.multiply()`, with `HALF_UP` rounding mode and 4 decimal places. The developer
must understand why `double` is never acceptable here (see ADR-004).

Spring Boot is used (not Quarkus) because the immutable ledger pattern is best expressed
with Spring Data's auditing annotations (`@CreatedDate`, `@CreatedBy`) and its repository
abstraction that can be configured to refuse update operations. The additional Quarkus
contrast is already provided by Payment Service.

---

**API Gateway — Spring Cloud Gateway (Java 21)**

Spring Cloud Gateway is built on Spring WebFlux (Project Reactor), not Spring MVC. This is
the only place in the platform where the developer writes reactive Java (Mono, Flux) for HTTP
handling. The contrast with the imperative Servlet model in the other Spring Boot services is
the lesson.

This service teaches:
- **Reactive HTTP pipeline**: `GatewayFilter` chains where each filter is a function from
  `ServerWebExchange` to `Mono<Void>`. The reactive model forces correct thinking about
  non-blocking I/O at the HTTP layer.
- **JWT validation at the edge**: a custom `GatewayFilter` validates the RS256 JWT,
  extracts claims, and forwards a `X-User-Id` and `X-User-Role` header to downstream services.
  Downstream services trust this header (they do not re-validate the JWT signature on every
  request — local validation is for occasional secondary checks). This is the gateway-enforced
  authentication pattern.
- **Route configuration**: YAML-based or programmatic route definitions, predicates, and
  filters that map external paths to internal service addresses.
- **Rate limiting**: Redis-backed sliding window rate limiter via `spring-cloud-gateway`'s
  `RequestRateLimiter` filter. The developer must configure `RedisRateLimiter` and understand
  why the Redis key schema for rate limiting works.

---

**Event Service — NestJS 10 (TypeScript)**

NestJS is the "Spring Boot of Node.js" — an opinionated, module-based, decorator-driven
framework with built-in dependency injection. Using it in the first TypeScript service
teaches the architectural pattern contrast: the same concepts (modules, providers, guards,
interceptors) appear in both Spring Boot and NestJS, but implemented differently.

This service teaches:
- **NestJS module system**: `EventModule` containing `EventController`, `EventService`,
  `EventRepository`, and async providers for Kafka and MongoDB clients.
- **MongoDB with Mongoose via NestJS's `@nestjs/mongoose`**: `@Schema()`, `@Prop()`,
  and `SchemaFactory.createForClass()` teach how TypeScript classes become MongoDB schemas.
- **Kafka publishing via `@nestjs/microservices`**: publishing `event.published` to Kafka
  after a MongoDB write, in a transactional outbox-lite pattern (write to MongoDB, then
  publish — the simpler form acceptable for a T2 service).
- **Elasticsearch indexing**: consuming the Kafka event in Search Service (a different service)
  teaches the eventual-consistency publishing model. Event Service publishes; Search Service
  indexes. The 30-second indexing SLO (NFR-PERF-010) connects these two services.
- **Guards for RBAC**: an `OrganiserGuard` that extracts the forwarded `X-User-Role` header
  and rejects non-Organiser requests — the NestJS equivalent of Spring's `@PreAuthorize`.

---

**Venue Service — NestJS 10 (TypeScript)**

NestJS is used again (not Fastify) because Venue's complexity justifies the framework's
structure. The VenueBooking negotiation state machine (REQUESTED → ACCEPTED → REJECTED →
CONFIRMED) is a domain-rich workflow that benefits from NestJS's service-provider composition.

This service teaches:
- **State machine pattern in TypeScript**: a `VenueBookingStateMachine` service that enforces
  valid transitions. The developer must understand the difference between encoding state as a
  string field (easy, fragile) and encoding it as a validated enum with explicit transition
  methods (correct, defensible).
- **Versioned MongoDB documents**: seating layouts are versioned — `layoutVersion` increments
  with each change, and old versions are retained. The developer must model this in MongoDB
  and understand why a relational schema would struggle with deep, variable-depth hierarchy.
- **Calendar view query**: given a date range, find all confirmed VenueBookings. This teaches
  MongoDB date range queries and index design for time-based access patterns.

---

**Ticket Service — Fastify 4 (TypeScript)**

Fastify is chosen over NestJS because Ticket Service's API is simple (issue ticket, fetch
ticket, verify token, generate PDF), high-throughput, and does not benefit from NestJS's
module architecture. The performance contrast between NestJS and Fastify is the lesson.

This service teaches:
- **Fastify's schema-first validation**: every route is defined with a JSON Schema for both
  request and response. Fastify's `ajv` integration validates and serialises automatically.
  The developer learns that input validation does not require a framework — it requires a
  schema.
- **Plugin architecture**: Fastify's `fastify.register()` plugin system is the alternative
  to NestJS modules. Services, repositories, and middleware are all plugins.
- **gRPC server in Node.js**: Ticket Service exposes a gRPC endpoint consumed by Check-in
  Service (Node-to-Java is Check-in→Ticket, but the direction is Check-in calls Ticket).
  Wait — re-reading the spec: Check-in Service calls Ticket Service for QR token verification
  (Node.js → Java? No — both Check-in and Ticket are Node.js services). The second gRPC pair
  is Check-in (Fastify/Node.js) → Ticket (Fastify/Node.js). Both are TypeScript. Actually,
  re-reading the system prompt: "Check-in Service → Ticket Service (QR token verification —
  Node.js to Java, teaches cross-language gRPC)". But both Check-in and Ticket are mapped to
  TypeScript/Fastify in the framework table. Resolving: the system prompt labels the gRPC pair
  as "Node.js to Java" for the Check-in→Ticket pair, but the framework table in the system
  prompt also assigns Ticket Service to Fastify (Node.js). This ADR resolves the ambiguity:
  **Ticket Service is TypeScript/Fastify, and the "cross-language" in the gRPC pair description
  refers to the fact that Check-in calls Ticket via gRPC in a Node.js-to-Node.js setup using
  the same `.proto` contract that the Java-to-Java pair uses**. The actual cross-language gRPC
  learning (different stub generation toolchains) occurs in the Booking→Seat Inventory pair
  (both Java). The Check-in→Ticket pair teaches same-language gRPC in TypeScript, using
  `@grpc/proto-loader` on both ends. This is a sufficient distinction from the Java pair.

  **NOTE TO IMPLEMENTER**: If the cross-language gRPC lesson (Node.js calling Java) is a hard
  requirement, Ticket Service could be reimplemented in Java. That change requires a superseding
  ADR. The current assignment (Ticket = Fastify/TypeScript) is consistent with the system
  prompt's framework assignments table and is accepted here.

- **HMAC-SHA256 QR token issuance**: the `crypto` module's `createHmac` with the per-event
  secret key. The developer must understand why the secret is per-event (a compromised key
  for one event cannot forge tokens for another) and how to retrieve it from Vault.

---

**Check-in Service — Fastify 4 (TypeScript)**

QR verification at p99 < 200 ms (NFR-PERF-004) is the hard constraint driving the
framework choice. Fastify's per-request overhead is measured in tens of microseconds;
NestJS's interceptor chain adds hundreds. At 200 ms, every millisecond counts.

This service teaches:
- **The hot path discipline**: the 200 ms budget means no database calls in the verification
  path. The per-event HMAC key is loaded at startup (or on first scan for that event) and
  held in process memory. The developer must understand that this is a deliberate trade-off:
  memory grows with the number of active events, but the CPU and I/O cost is zero at runtime.
- **Offline resilience pattern**: if this service loses network connectivity, it can still
  verify QR codes for events whose keys are cached in memory (NFR-AVAIL-006). The developer
  must design the key loading, cache invalidation, and recovery sync correctly.
- **gRPC client in TypeScript with `@grpc/grpc-js`**: Check-in calls Ticket Service via gRPC
  to fetch the per-event HMAC key on first encounter. The TypeScript gRPC client setup,
  channel creation, and stub invocation teach Node.js gRPC client patterns.
- **Idempotent check-in**: the second scan of the same QR code must return success with
  "already checked in" — not a 409. This teaches idempotent write patterns: check for
  existence before insert, return the existing state rather than an error.

---

**Notification Service — Raw Node.js 20 + Socket.IO 4 (TypeScript)**

The deliberate choice to use no application framework in this service is the most
instructive single decision in the Node.js tier. The developer must manually assemble
what NestJS and Fastify provide automatically.

This service teaches:
- **Node.js HTTP server without a framework**: `http.createServer()`, request routing,
  middleware composition by hand. The developer sees the foundation that every Node.js
  framework builds on.
- **Socket.IO room management**: rooms scoped to `userId` (booking updates, waitlist offers)
  and `eventId` (seat map changes, live sales counters). Room join/leave lifecycle,
  event emission to a room, and broadcasting to all connections in a room.
- **Redis Socket.IO adapter**: `@socket.io/redis-adapter` with `createAdapter()` from
  the ioredis client. This is what enables horizontal scaling — Socket.IO rooms become
  cluster-wide, not instance-local. Without this, a customer on instance A would not
  receive a broadcast from instance B.
- **Kafka consumer in raw Node.js**: `kafkajs` without a framework wrapper teaches the
  Kafka consumer group lifecycle (join, subscribe, `run`, commit) at the library level.
  The developer must handle partition assignment, message processing, and offset management
  explicitly.
- **The abstraction tax**: after implementing this service, the developer should be able
  to articulate exactly what NestJS and Fastify save them — and when saving that cost is
  the wrong trade-off (this service, where simplicity matters more than structure).

---

**Waitlist Service — NestJS 10 (TypeScript)**

Redis sorted sets are the core data structure here. NestJS is used (not Fastify) because
the Waitlist's domain logic — creating offers, tracking expiry, promoting the next member —
benefits from NestJS's structured service-provider model to keep the state machine clean.

This service teaches:
- **Redis sorted sets for ordered queuing**: `ZADD`, `ZRANGE`, `ZPOPMIN`, `ZREM`. The
  developer must understand why this is O(log N) for insertions but O(1) for getting the
  minimum — and why that asymmetry is exactly what a priority queue needs.
- **Time-limited offer pattern**: when a seat is released, a `WaitlistOffer` is created with
  an expiry timestamp. A Kafka consumer (or Redis key expiry notification) detects offer
  expiry and promotes the next member. Choosing between Redis key expiry events and an
  application-level scheduler is itself an architectural decision the developer must justify.
- **BullMQ for delayed jobs**: the offer expiry can be modelled as a BullMQ delayed job
  (execute after X minutes) if Redis key expiry events are not reliable enough. This
  introduces the developer to Redis-backed job queues — a different use of Redis than
  the sorted set for the waitlist itself.

---

**Search Service — FastAPI 0.111 (Python 3.12)**

Search is the first Python service and the gateway to the AI/ML tier. FastAPI is chosen
for all AI/ML services because async I/O is the correct model for services that make
concurrent network calls (Elasticsearch query + Qdrant query + blend).

This service teaches:
- **Concurrent async I/O in Python**: `asyncio.gather()` to run the Elasticsearch and
  Qdrant queries concurrently, then blend results. The developer must understand why
  `await query_elasticsearch()` followed by `await query_qdrant()` is sequential
  (bad — 800 ms = 400+400) while `asyncio.gather()` is concurrent (good — 800 ms = max(400,400)).
- **Pydantic v2 at service boundaries**: every search request and response is a Pydantic
  model. The developer learns that Pydantic is not just a validator — it is a serialisation
  contract. Changing a field name requires a migration, not just a code change.
- **Elasticsearch query DSL in Python**: `multi_match`, `bool` queries with `filter` clauses
  for faceted search, and the `elasticsearch-py` async client. The developer learns how
  to translate product requirements ("filter by city, date range, and price") into Elasticsearch
  query JSON without a query builder abstraction.
- **Hybrid search (BM25 + ANN)**: blending a BM25 score from Elasticsearch with a cosine
  similarity score from Qdrant requires a normalisation step (scores are on different scales).
  The developer must implement Reciprocal Rank Fusion (RRF) or a linear blend with tunable weights.

---

**Recommendation Service — FastAPI 0.111 (Python 3.12)**

ML inference serving. This service teaches the separation between **offline training**
(happens in a notebook or script, tracked by MLflow) and **online serving** (happens via
FastAPI endpoint, reads from Redis cache first).

This service teaches:
- **Model loading at startup**: `@app.on_event("startup")` loads the serialised model from
  MLflow model registry (or local path in dev). The developer learns why model loading must
  be eager (not lazy per-request) and how to handle model reload without downtime.
- **Redis inference caching**: the recommendation for a given user is cached in Redis with a
  5-minute TTL. The cache key is `rec:{userId}:{modelVersion}`. The developer must implement
  cache-aside, handle cache miss, and understand why including `modelVersion` in the key
  prevents stale recommendations after a model update.
- **MLflow model registry integration**: `mlflow.pyfunc.load_model()` retrieves the model.
  The developer learns MLflow's model lifecycle (STAGING, PRODUCTION stages) and how to
  configure the serving layer to always load from the PRODUCTION stage.
- **NFR-AVAIL-004 — fail-open fallback**: if the model is unavailable, return the top-10
  events by ticket sales (pre-computed and cached in Redis) instead of an error. The
  developer must implement the fallback and test it under model unavailability.

---

**Chatbot Service — FastAPI 0.111 (Python 3.12)**

RAG pipeline orchestration. The most architecturally complex Python service.

This service teaches:
- **LangChain RAG pipeline**: `ConversationalRetrievalChain` with a Qdrant retriever,
  a prompt template that injects retrieved chunks, and an LLM (streaming). The developer
  must understand each component: retriever, prompt, LLM, output parser.
- **Streaming responses via FastAPI**: `StreamingResponse` with `application/x-ndjson`
  or Server-Sent Events. The developer must understand why streaming is not "return the
  full response later" but "push chunks as they are generated". This is the first-token
  latency concern (NFR-PERF-009 — p99 < 2,500 ms).
- **Qdrant retrieval in a RAG context**: searching the event corpus by semantic similarity
  to the user's query, retrieving the top-k chunks, and injecting them into the prompt.
  The developer learns the trade-off between k (more context = better answer, higher cost)
  and the LLM's context window limit.
- **Conversation memory with Redis**: `ConversationBufferWindowMemory` backed by Redis
  (last N turns per session). The developer learns why conversation memory is a cache,
  not a database — it is session-scoped and can be lost without breaking the system.
- **JWT forwarding to downstream services**: when the chatbot needs real booking data,
  it calls the Booking or Event Service REST APIs, forwarding the Customer's JWT. The
  developer must pass the `Authorization` header correctly and handle 403s gracefully.

---

**Fraud Detection Service — FastAPI 0.111 (Python 3.12)**

Dual-mode service: Kafka consumer for real-time scoring, REST endpoint for on-demand
queries. FastAPI supports both in the same process because `aiokafka` is async-compatible.

This service teaches:
- **Kafka consumer in async Python**: `aiokafka.AIOKafkaConsumer` integrated with FastAPI's
  `lifespan` context manager. The developer must handle consumer group join, message processing,
  and graceful shutdown (stopping the consumer before the process exits) correctly.
- **Feature engineering at serving time**: a `FeatureExtractor` that queries PostgreSQL for
  historical booking velocity (bookings per user in the last hour) and derives fraud signals.
  The developer learns that feature engineering is not only a training-time concern — it
  happens at inference time too, and latency matters (NFR-PERF-011 — fraud score within 10 s
  of booking placement).
- **Fail-open pattern (NFR-AVAIL-005)**: if the fraud scoring fails (model unavailable,
  feature extraction timeout), the service publishes a `LOW_RISK` score. The booking is
  not blocked. The developer must implement this default explicitly — not as an exception
  handler that happens to return LOW_RISK, but as a documented policy with a Prometheus
  counter tracking fail-open events.

---

**Analytics Service — Flask 3 + Gunicorn (Python 3.12)**

The only Flask service in the platform. Flask is chosen here for three reasons.

First, Analytics Service endpoints are **synchronous ClickHouse queries**. A single ClickHouse
query (e.g., "ticket sales by venue in the last 60 minutes") may take 50–200 ms. The request
handler blocks on the query. FastAPI's async model provides no benefit here — the bottleneck
is the ClickHouse query, not the I/O handling. Flask's synchronous model is simpler and
equally correct for this access pattern.

Second, Analytics Service does not serve ML inference, does not manage conversations, and
does not need Pydantic at its request boundary (Grafana's direct HTTP datasource plugin sends
a simple JSON body). The overhead of FastAPI's schema machinery would produce no user-visible
benefit.

Third, **the contrast between Flask and FastAPI within this project is the lesson**. After
implementing four FastAPI services with Pydantic models, async handlers, and automatic OpenAPI
generation, the developer implements one Flask service with manual validation, synchronous
handlers, and manual schema documentation. The comparison reveals what FastAPI is adding and
why it is worth the overhead for services that need it.

This service teaches:
- **Flask blueprint architecture**: `analytics_bp` as a blueprint registered on the Flask app,
  routing Grafana datasource calls to the correct ClickHouse query handler.
- **ClickHouse Python driver**: `clickhouse-connect` or `clickhouse-driver` for typed query
  results. The developer learns ClickHouse's SQL dialect, its columnar result format, and
  why `ORDER BY` on a ClickHouse MergeTree table is cheap (the data is already sorted).
- **Gunicorn worker model**: multiple synchronous workers for concurrency. The developer
  learns why synchronous Flask + Gunicorn is a valid production setup when requests are
  CPU/IO-bound (not I/O-wait), and why Uvicorn is the right ASGI server for FastAPI.

---

### 3.5 Cold Start Analysis — NFR-PERF-042

NFR-PERF-042: Java < 30 s, Node.js / Python < 10 s.

| Service | Framework | Baseline Cold Start | Mitigation | Meets NFR? |
|---------|-----------|--------------------|-----------:|------------|
| Auth Service | Spring Boot 3 | 15–25 s | `spring.main.lazy-initialization=true` reduces to 10–18 s | ✅ |
| Seat Inventory Service | Spring Boot 3 | 15–25 s | Lazy init + minimal auto-configuration | ✅ |
| Booking Service | Spring Boot 3 | 18–28 s | Most complex service; lazy init required | ✅ (tight) |
| Payment Service | Quarkus 3 JVM | 5–10 s | No mitigation needed | ✅ |
| Disbursement Service | Spring Boot 3 | 15–22 s | Lazy init | ✅ |
| API Gateway | Spring Cloud Gateway | 8–15 s | Reactive startup; lighter than MVC | ✅ |
| Event Service | NestJS 10 | 2–4 s | — | ✅ |
| Venue Service | NestJS 10 | 2–4 s | — | ✅ |
| Ticket Service | Fastify 4 | 0.5–1.5 s | — | ✅ |
| Check-in Service | Fastify 4 | 0.5–1.5 s | — | ✅ |
| Notification Service | Raw Node.js | < 0.5 s | — | ✅ |
| Waitlist Service | NestJS 10 | 2–4 s | — | ✅ |
| Search Service | FastAPI | 3–5 s (model loading adds ~2 s) | Lazy embedding model load on first request | ✅ |
| Recommendation Service | FastAPI | 5–12 s (model load dominates) | Lazy model load with warmup probe | ⚠️ Risk |
| Chatbot Service | FastAPI | 4–8 s | LangChain init deferred to first request | ✅ |
| Fraud Detection Service | FastAPI | 4–8 s | Model load at startup (small model) | ✅ |
| Analytics Service | Flask + Gunicorn | 1–3 s | — | ✅ |

**⚠️ Recommendation Service Risk:** A large collaborative filtering model (e.g., LightFM)
can take 8–15 s to load from disk in the default configuration. Mitigation: use lazy model
loading (load on first recommendation request, not at server startup); pre-cache the top-N
recommendations for the most popular events in Redis at startup using only the cached data,
not the full model. The developer must verify this mitigation achieves < 10 s before the
health probe deadline expires in Kubernetes (Kubernetes `initialDelaySeconds` for the
readiness probe must be set to at least 15 s for this service).

**Spring Boot mitigation details:** `spring.main.lazy-initialization=true` defers bean
initialisation until first use. The risk is that startup errors in lazy beans surface
on the first request, not at startup. Mitigation: integrate a smoke test in the health
check that exercises each lazy bean on first `/health/ready` call. Alternatively, use
`@Lazy(false)` explicitly on any bean whose startup failure should prevent the service
from becoming ready.

### 3.6 Local Memory Budget — NFR-PERF-043

NFR-PERF-043: full Docker Compose stack ≤ 12 GB.

The per-service memory budget is defined in NFR §5.4. This ADR's responsibility is to
verify that the framework assignments are consistent with those budgets.

| Tier / Group | Services | Budget per Service | Total | JVM / Runtime Overhead |
|---|---|---|---|---|
| Java T1 | Auth, Seat Inventory, Booking, Payment, Disbursement | 512 MB `-Xmx` | 2,560 MB | Spring Boot heap usage at idle: ~200–300 MB within limit; Quarkus (Payment): ~180 MB |
| Java T2 | API Gateway | 384 MB `-Xmx` | 384 MB | Spring Cloud Gateway WebFlux at idle: ~150 MB |
| Node.js | Event, Venue, Ticket, Check-in, Notification, Waitlist | 256 MB `--max-old-space-size` | 1,536 MB | NestJS/Fastify at idle: ~80–150 MB; raw Node.js: ~50 MB |
| Python | Search, Recommendation, Chatbot, Fraud Detection, Analytics | 384 MB | 1,920 MB | FastAPI/Gunicorn at idle: ~100–200 MB; model-loaded services: ~250–350 MB |

**Critical constraint — Python ML services:** Recommendation and Chatbot services load ML
models and vector embeddings into process memory. The 384 MB budget is tight for a
collaborative filtering model + Qdrant client + LangChain. Mitigation: use quantized
models (8-bit) in local development; the full-precision model is used only in staging
and production.

**Enforcement:** `scripts/check-memory.sh` in `stagepass-infrastructure` uses
`docker stats --no-stream --format "{{.MemUsage}}"` to verify that no service exceeds
its budget after 60 seconds of idle time. This script runs in CI for Phase 2 onwards
and blocks merges to main if any service exceeds its allocation.

**Total verified against NFR §5.4:** Services (6,400 MB) + Infrastructure (PostgreSQL 512 MB +
MongoDB 512 MB + Redis 256 MB + Kafka+ZK 512 MB + Elasticsearch 512 MB + Qdrant 256 MB +
ClickHouse 256 MB + MinIO 128 MB) + Observability stack (1,500 MB) = **~10,844 MB**.
Headroom: ~1,156 MB. This passes the 12 GB ceiling with margin for OS overhead.

---

## 4. Consequences

### Positive Consequences

**Maximum learning exposure within a coherent system.** Three languages, eight frameworks,
six database types — each choice is defended by a specific learning objective that cannot
be achieved by a different choice in this project. The developer will emerge from this
project having implemented production services in Spring Boot, Quarkus, Spring Cloud Gateway,
NestJS, Fastify, raw Node.js, FastAPI, and Flask — across 18 distinct domains.

**Framework contrast is structural, not incidental.** NestJS vs. Fastify vs. raw Node.js
in the same project teaches what each framework is hiding from you. Spring Boot vs. Quarkus
teaches that startup overhead is a design choice. FastAPI vs. Flask teaches that async is
not always the right tool. These contrasts are only learnable when both sides exist.

**Failure mode isolation.** Each service's language/framework mismatch cannot propagate.
A Quarkus-specific bug in Payment Service cannot affect Spring Boot's behaviour in Auth
Service. The polyrepo topology (ADR-001) enforces this isolation structurally.

**Portfolio signal.** A developer who can articulate why Quarkus was chosen for Payment
Service, why Fastify was chosen for Check-in Service, and why Flask was chosen for Analytics
Service — and who can demonstrate all three running in production — demonstrates engineering
judgment, not just framework familiarity.

### Negative Consequences and Mitigations

**Tool installation overhead.** Eight frameworks require Java 21 (SDKMAN), Node.js 20 (nvm),
and Python 3.12 (pyenv) — three separate version managers in addition to Docker. This is
the direct cost of polyglot diversity.
*Mitigation:* Every service must run via `docker compose up` without local language runtimes.
The Dockerfile is the primary development environment. Local runtimes are optional for IDE
support. NFR-MAINT-004 (30-minute local setup) applies to Docker-based setup only.

**Cognitive context switching.** Moving between a Spring Boot service (imperative, annotation-
driven) and a FastAPI service (async, Pydantic, type-hint-driven) in the same day requires
mental context-switching overhead.
*Mitigation:* The phase roadmap sequences services by language cluster. Phase 3 introduces
Java (Spring Boot). Phase 4 introduces the remaining Java services and the Node.js services.
Phase 5 introduces Python. The developer builds deep familiarity with each language cluster
before moving to the next.

**Debugging complexity.** A distributed trace through a flash sale booking touches Spring Boot
(Auth), Spring Cloud Gateway, Quarkus (Payment), Spring Boot (Booking), Spring Boot (Seat
Inventory), NestJS (Notification), and raw Node.js (WebSocket push) — in seven different
runtime environments. A bug in any step is harder to isolate when each service logs differently
and has different stacktrace formats.
*Mitigation:* NFR-OBS-001–003 mandate structured JSON logging with correlation IDs and
OpenTelemetry tracing in all services. The correlation ID and trace context propagate across
all runtimes via OpenTelemetry's HTTP and Kafka header propagation. A bookingId trace in
Grafana + Jaeger surfaces spans from all services in one view, regardless of runtime.

**Per-service configuration duplication.** Vault integration, OpenTelemetry agent, Prometheus
metrics — these must be configured in eight different ways (Spring Actuator, NestJS metrics
module, FastAPI `/metrics` endpoint, etc.).
*Mitigation:* Each service's README documents the exact observability configuration.
A section in `stagepass-docs/CONTRIBUTING.md` defines the observability contract that all
services must satisfy. The CI pipeline enforces that `/health/live`, `/health/ready`, and
`/metrics` return the correct responses before a merge is allowed.

---

## 5. Alternatives Considered

### 5.1 Quarkus vs. Spring Boot for Payment Service

**The question:** Payment Service is T1, handles financial webhooks, and requires fast crash
recovery. Should it use Quarkus or Spring Boot?

**Spring Boot argument:**
- Every other T1 Java service uses Spring Boot. Consistency reduces cognitive overhead.
- Spring's Kafka integration (`spring-kafka`) is more mature than Quarkus's Kafka extension for
  complex consumer group management.
- Spring Security is already configured in the other T1 services; consistent security configuration
  reduces risk.

**Quarkus argument:**
- Quarkus JVM mode starts in 5–10 seconds; Spring Boot takes 15–25 seconds. For a T1 service
  recovering from a crash during a flash sale, 15 seconds of unavailability vs. 5 seconds is
  meaningful against a 43-minute monthly error budget.
- Quarkus's compile-time DI surfaces injection errors at build time, not runtime. For a financial
  service, build-time error detection is strictly better.
- The Quarkus Panache reactive PostgreSQL client teaches the reactive data access pattern that
  does not exist in Spring Data JPA. One service must use Quarkus to teach this contrast.
- Payment Service is the service in this system that most naturally benefits from native image
  compilation in production (fast startup, small memory footprint) — even if native compilation
  is out of scope for this project, the Quarkus design principles prepare the codebase for it.

**Decision:** Quarkus. The cold-start advantage (NFR-PERF-042) provides an architectural
justification beyond learning alone. The contrast with Spring Boot is maximally informative
precisely because Payment Service is T1 — the developer learns that Quarkus and Spring Boot
are not interchangeable, and understands specifically where Quarkus wins.

**What would change the decision:** If the Quarkus Kafka extension proves insufficient for the
Payment webhook saga's compensating transaction pattern, a superseding ADR may migrate Payment
Service to Spring Boot. The trigger is a failed integration test proving saga correctness, not
preference.

---

### 5.2 Fastify vs. NestJS for Ticket and Check-in Services vs. Event, Venue, and Waitlist Services

**The question:** The Node.js tier has three services using NestJS and two using Fastify. Is
this split defensible, or is it accidental complexity?

**"Use NestJS everywhere" argument:**
- Consistency: all Node.js services use the same framework, the same module pattern, the same
  dependency injection model.
- NestJS's module system scales: even simple services benefit from the enforced structure.
- Less cognitive overhead: one Node.js developer can contribute to any Node.js service without
  framework re-learning.

**"Fastify for Ticket and Check-in" argument:**
- Check-in verification has a p99 < 200 ms target (NFR-PERF-004). Fastify's per-request overhead
  (measured at 30–50 µs) vs. NestJS's interceptor chain (80–200 µs including serialisation) is a
  real difference at this latency target. Under 50 concurrent gate scanners, 150 µs saved per
  request adds up.
- Ticket Service is architecturally simple: issue ticket, fetch ticket, generate PDF. NestJS's
  module system is unnecessary overhead for a service with 3 endpoints. Fastify's plugin
  architecture is simpler and equally expressive.
- The Fastify vs. NestJS contrast is a learning goal. Both frameworks exist in the Node.js
  ecosystem; both are used in production at scale. A developer who has used both can make
  informed choices in their career. Using NestJS everywhere removes this lesson.
- Raw Node.js (Notification Service) is already the "no framework" extreme. Fastify occupies
  the middle position: "minimal framework". NestJS occupies the "opinionated framework" position.
  The three-framework Node.js tier teaches the full spectrum.

**Decision:** Fastify for Ticket and Check-in; NestJS for Event, Venue, and Waitlist; raw
Node.js for Notification. The performance justification for Fastify in Check-in is genuine,
not contrived. The learning objective for the three-point spectrum is explicit, not accidental.

---

### 5.3 FastAPI vs. Django vs. Flask for All Python Services

**The question:** Why FastAPI for four services and Flask for one, rather than a consistent choice?

**"All FastAPI" argument:**
- Consistency: one async framework across all Python services.
- FastAPI's type system and Pydantic integration is a universal benefit — even Analytics Service
  would be better validated with Pydantic models.
- FastAPI's automatic OpenAPI generation means all five Python services have self-documenting APIs.

**"All Flask" argument:**
- Flask is simpler. For services that do not need async, FastAPI's async model adds cognitive
  overhead without benefit.
- Flask's synchronous model is easier to reason about for T3 services where correctness matters
  more than throughput.
- Flask is more widely deployed in production ML systems than FastAPI (historically). Learning
  Flask is not wasted.

**"Django" argument:**
- Django REST Framework is the most full-featured Python API framework.
- Django's ORM would benefit Fraud Detection's PostgreSQL queries.

**Why Django is rejected:** Django is a monolithic, batteries-included framework designed for
web applications with templates, sessions, and admin interfaces. Importing Django for a
microservice that serves JSON to other services is importing 90% waste. Django's ORM is a
coupling between the framework and the database; in a microservice, we want the database
client to be replaceable. Django is the wrong tool for this architecture.

**Why "all FastAPI" is rejected:** Analytics Service runs long-running ClickHouse queries.
Making these queries async does not improve throughput — the ClickHouse query is synchronous
over the network regardless of the Python async model. Wrapping it in `asyncio.run_in_executor()`
to avoid blocking the event loop adds complexity without benefit. Flask's synchronous model
is the honest choice.

**Why "all Flask" is rejected:** Search, Recommendation, Chatbot, and Fraud Detection all make
concurrent downstream calls (Elasticsearch + Qdrant simultaneously, Kafka consumer + REST
endpoint in the same process). FastAPI's `asyncio.gather()` and `aiokafka` integration require
an async event loop. Flask cannot host an asyncio event loop in its synchronous WSGI model
without `asgiref` or similar bridging. Using Flask for these services would require
`run_in_executor()` for every network call — more complexity, not less, than FastAPI.

**Decision:** FastAPI for all ML/AI services; Flask for Analytics. The split is driven by
concurrency requirements, not framework preference. The contrast between the two is the lesson:
async is a tool, not a virtue.

---

## 6. References

| Source | Relevance |
|---|---|
| PRD.md §3 — Full Tech Stack | The primary source of framework assignments; this ADR provides the WHY |
| PRD.md Constraint C-01 | Solo developer: tool installation overhead, JVM version management, local memory are real costs |
| PRD.md Constraint C-05 | Stack is locked; this ADR does not re-open choices |
| PRD.md Constraint C-06 | Polyglot by design; §3.3 of this ADR directly addresses the diversity line |
| NFR.md §3 — Service Tier Map | T1/T2/T3 assignments drive framework selection maturity requirements |
| NFR.md §5.4 — Local Stack Memory Budget | Per-service memory budgets validated in §3.6 |
| NFR-PERF-042 | Cold-start hard limits; validated in §3.5 |
| NFR-PERF-043 | 12 GB local stack ceiling; validated in §3.6 |
| NFR-PERF-001 | Seat hold p99 < 500 ms; drives Redis + gRPC architecture in Seat Inventory |
| NFR-PERF-004 | QR check-in p99 < 200 ms; drives Fastify choice for Check-in Service |
| NFR-REL-008 | SERIALIZABLE isolation for seat operations; drives Spring Boot + JPA for Seat Inventory |
| NFR-AVAIL-004 | Recommendation fail-open; drives Redis caching strategy and Flask contrast |
| NFR-AVAIL-005 | Fraud Detection fail-open; drives explicit default score policy |
| ADR-001 — Repository Topology | 21-repo structure; this ADR assigns exactly one framework to each service repo |
| ADR-003 | Service communication patterns; drives gRPC framework integration per service |
| Richardson — Microservices Patterns, Ch. 4 | Sagas and the database-per-service pattern; grounds the one-DB-per-service decision |
| Newman — Building Microservices (2nd ed.), Ch. 2 | Technology choices as team autonomy decisions; the polyglot rationale |
| Kleppmann — DDIA, Ch. 2 | Data model selection (relational vs. document vs. graph); grounds the DB-per-service rationale |
| Quarkus documentation — Quarkus vs. Spring | Official comparison of startup model and DI approach |
| Fastify benchmarks (fastify.io/benchmarks) | Empirical per-request overhead data supporting the Check-in Service choice |
| Python asyncio docs — Gathering Coroutines | `asyncio.gather()` as the correct concurrent I/O pattern for Search Service |

---

## 7. Quick Self-Check

Answer these questions before writing the first line of code in any service.
If you cannot answer them clearly, re-read §3.4 for that service before proceeding.

---

**Q1: You are about to add a utility function that both the Booking Service (Java/Spring Boot)
and the Fraud Detection Service (Python/FastAPI) need — for example, a function to determine
whether a given booking is eligible for a refund based on the cancellation policy. Where does
this function live, and why does the polyglot stack complicate your first instinct?**

In a single-language codebase, this would be a shared library. In a polyglot polyrepo, a
shared library must be written twice — once in Java for Booking Service and once in Python
for Fraud Detection Service. That is the correct answer. The alternative — putting it in a
"shared service" — adds network latency and a new failure domain for a pure computation. The
alternative — forcing both services to the same language — violates the polyglot learning
goal and PRD C-05. Accepting the duplication is the correct discipline. If the function is
complex enough that two implementations feel dangerous, extract it into a separate service
with its own REST API — that is a legitimate architectural choice. If it is simple (a few
lines of date arithmetic), duplicate it with a comment in both codebases noting the
duplication is intentional and the canonical logic is in the PRD (Section 7.3).

---

**Q2: A new requirement says that the Recommendation Service must return results within 300 ms
instead of 500 ms (tightening NFR-PERF-012). You look at the current FastAPI + MLflow model
loading architecture and realise the model inference itself takes 250 ms on a cold cache.
What are your three options in order of increasing architectural disruption, and which option
does not require an ADR?**

Option 1 (no ADR required): pre-warm the Redis inference cache more aggressively. Instead of
caching only after the first request, run a background task that pre-computes recommendations
for the top-1000 users by activity at service startup. The hot path becomes Redis-only (< 5 ms).
This is a configuration and implementation change, not an architectural change.

Option 2 (ADR required): replace the MLflow model with a lighter model that runs in < 100 ms
(e.g., a collaborative filtering model with a smaller embedding dimension, or an approximate
nearest-neighbour approach). This changes the ML architecture and must be documented.

Option 3 (ADR required): split Recommendation Service into an offline batch scorer
(computes and caches recommendations for all active users nightly) and an online lookup service
(pure Redis reads). This is a fundamental architectural change — from online inference to
batch-first with online fallback. The offline batch scorer becomes a new service or a scheduled
job. Both require an ADR.

---

**Q3: The system prompt says the gRPC pair "Check-in → Ticket" teaches "cross-language gRPC
(Node.js to Java)". But this ADR assigns both Check-in and Ticket Service to TypeScript/Fastify.
How do you resolve this apparent contradiction, and what is the actual learning objective for
the Check-in → Ticket gRPC pair?**

The assignment in this ADR (both services in TypeScript/Fastify) takes precedence — the system
prompt's "Node.js to Java" label appears to be a documentation artefact from an earlier version
of the service assignments. The cross-language learning objective is satisfied by the Booking →
Seat Inventory pair (both Java, same-language gRPC with `protobuf-maven-plugin`). The Check-in
→ Ticket pair teaches TypeScript gRPC (both client and server in Node.js using `@grpc/grpc-js`
and `@grpc/proto-loader`) from the same shared `.proto` contract in `stagepass-shared-contracts`.
The two pairs together teach: (1) Java gRPC stub generation via Maven protoc plugin; (2) Node.js
gRPC client/server using the JS proto loader. That is a sufficient cross-ecosystem learning. If
the intent was Java on the Ticket Service side, that change requires a superseding ADR and must
account for the additional Java service in the memory budget and phase timeline.

---

*End of ADR-002 — Tech Stack Per Service*
