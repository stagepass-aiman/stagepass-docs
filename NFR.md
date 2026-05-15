# NFR — StagePass: Non-Functional Requirements

## Document Header

| Field        | Value                                                                                  |
|--------------|----------------------------------------------------------------------------------------|
| Document ID  | NFR                                                                                    |
| Version      | 1.0.0                                                                                  |
| Status       | Accepted                                                                               |
| Author       | StagePass Engineering                                                                  |
| Created      | 2025-05-15                                                                             |
| Last Updated | 2025-05-15                                                                             |
| Repo         | stagepass-docs                                                                         |
| Path         | /NFR.md                                                                                |
| Phase        | Phase 1 — Requirements & Design                                                        |
| Traces To    | PRD.md · ADR-001 · ADR-002 · ADR-003 · ADR-004 · ADR-005 · ADR-006 · ADR-007 · ADR-008 · HLD.md |

### Change Log

| Version | Date       | Author                 | Summary                  |
|---------|------------|------------------------|--------------------------|
| 1.0.0   | 2025-05-15 | StagePass Engineering  | Initial accepted version |

---

## Table of Contents

1. [Purpose and Scope](#1-purpose-and-scope)
2. [How to Read This Document](#2-how-to-read-this-document)
3. [Service Tier Map and SLO Definitions](#3-service-tier-map-and-slo-definitions)
4. [SLO Burn-Rate Alert Windows](#4-slo-burn-rate-alert-windows)
5. [Capacity Model](#5-capacity-model)
6. [Performance Requirements — NFR-PERF](#6-performance-requirements--nfr-perf)
7. [Reliability Requirements — NFR-REL](#7-reliability-requirements--nfr-rel)
8. [Availability Requirements — NFR-AVAIL](#8-availability-requirements--nfr-avail)
9. [Security Requirements — NFR-SEC](#9-security-requirements--nfr-sec)
10. [Observability Requirements — NFR-OBS](#10-observability-requirements--nfr-obs)
11. [Maintainability Requirements — NFR-MAINT](#11-maintainability-requirements--nfr-maint)
12. [Compliance Requirements — NFR-COMP](#12-compliance-requirements--nfr-comp)
13. [Cross-Reference Index](#13-cross-reference-index)

---

## 1. Purpose and Scope

This document is the authoritative, measurable statement of how StagePass must
behave under load, in failure, under attack, and over time. It is not a wish list.
Every threshold stated here is a hard constraint that drives architectural
decisions, ADR content, service design, test criteria, and SLO alerting configuration.

**Scope:** All services in the StagePass platform as defined in the PRD. This
document applies from Phase 3 (first service shipped) onward. No service is
exempt. Where a threshold cannot be measured for a particular service, that gap
must be explicitly documented as a deferred NFR with a resolution trigger.

**Priority rule:** When an NFR threshold conflicts with a feature requirement, the
NFR is the constraint. Feature scope is reduced before an NFR threshold is relaxed.
Exceptions require a dedicated ADR.

**Traceability rule:** Every NFR in this document must trace to at least one PRD
requirement. NFRs without PRD anchors are invalid and must not be added.

---

## 2. How to Read This Document

Each NFR entry has a fixed schema:

```
ID          Unique identifier in the format NFR-<CATEGORY>-<NNN>
Statement   One declarative sentence stating the requirement
Threshold   The measurable pass/fail criterion
Method      How compliance is verified — test type, tool, measurement frequency
Traces To   The PRD section, goal (G-XX), or feature requirement (FR-X-XXX)
            that anchors this NFR
Notes       Implementation constraints, rationale, or linked ADRs where relevant
```

**Categories:**
- `PERF` — Performance latency, throughput, resource consumption
- `REL`  — Reliability: idempotency, saga correctness, data integrity
- `AVAIL`— Availability: uptime SLOs, graceful degradation
- `SEC`  — Security: auth, RBAC, cryptography, scanning
- `OBS`  — Observability: logging, metrics, tracing, alerting
- `MAINT`— Maintainability: test coverage, developer experience, documentation
- `COMP` — Compliance: DPDP Act, GST, Consumer Protection Act

---

## 3. Service Tier Map and SLO Definitions

The tier of a service determines the NFR rigour applied to it.
Tier assignment is a product decision, not an engineering preference.
Services are assigned the tier that reflects the consequence of their unavailability.

### 3.1 Tier Definitions

| Tier | SLO Target | Monthly Error Budget | Consequence of Breach |
|------|-----------|----------------------|-----------------------|
| **T1 — Critical** | 99.9% availability | 43.8 minutes downtime | Core booking flow unavailable; direct revenue and trust impact |
| **T2 — Important** | 99.5% availability | 3.65 hours downtime | Feature degradation; booking flow still operable via fallback |
| **T3 — Enhancement** | 99.0% availability | 7.31 hours downtime | Degraded experience; all core flows remain available via fallback |

Availability is measured as the proportion of 1-minute intervals in a calendar
month during which the service passes its `/health/ready` check AND returns
< 1% 5xx error rate on non-synthetic traffic.

### 3.2 Service Tier Assignments

| Service | Tier | Rationale |
|---------|------|-----------|
| Auth Service | T1 | Every authenticated request depends on it; local JWT validation means it is not in the hot path per-request, but key rotation and revocation operations are critical |
| Seat Inventory Service | T1 | Seat hold atomicity is the core correctness guarantee of the platform; unavailability means checkout is impossible |
| Booking Service | T1 | Saga coordinator; without it no booking can be created, tracked, or compensated |
| Payment Service | T1 | No payment confirmation = no booking completion; idempotency and saga integration are safety-critical |
| Disbursement Service | T1 | Escrow and ledger integrity; a corrupt ledger is a financial compliance failure |
| API Gateway | T2 | All external traffic routes through it; high impact but stateless; fast recovery |
| Event Service | T2 | Event CRUD and Elasticsearch publishing; unavailability prevents new event discovery but does not break in-progress bookings |
| Venue Service | T2 | Venue and seating layout management; impacts Organiser workflows but not Customer booking for existing events |
| Ticket Service | T2 | Ticket issuance and PDF generation; tickets already issued are unaffected by downtime |
| Check-in Service | T2 | Offline fallback keeps gate operations running; online service outage degrades real-time sync only |
| Notification Service | T2 | Fail-open: booking saga completes without awaiting notification delivery |
| Search Service | T2 | Discovery-critical; degraded mode (category browse) remains available |
| Waitlist Service | T3 | Fail-open; sold-out seat releases queue temporarily during downtime |
| Recommendation Service | T3 | Popularity fallback is always available |
| Chatbot Service | T3 | Static fallback message; rest of event page fully functional |
| Fraud Detection Service | T3 | Fail-open (LOW-risk default); retrospective scoring on recovery |
| Analytics Service | T3 | Last-known metrics displayed; no data lost from Kafka backlog |

### 3.3 SLO Window

SLOs are measured over a rolling 30-day window. The error budget resets at the
start of each calendar month. Burn-rate alerts (Section 4) fire before the budget
is exhausted, not after.

---

## 4. SLO Burn-Rate Alert Windows

Multi-window, multi-burn-rate alerting is mandatory for all T1 and T2 services.
This approach is taken from the Google SRE Workbook (Chapter 5). A single
threshold alert fires too late (budget already gone) or too often (noise).
The two-window approach reduces false positives while preserving fast detection.

### 4.1 Burn-Rate Alert Thresholds

For a service with a 99.9% SLO (T1), the error budget is 0.1% of the window.
A burn rate of 1× means the budget is consumed at exactly the rate that would
exhaust it over 30 days. A burn rate of 14.4× means the budget would be exhausted
in 2 hours.

| Alert Name | Severity | Short Window | Long Window | Burn Rate | Action |
|-----------|----------|-------------|------------|-----------|--------|
| T1-CriticalBurn | P1 — Page | 5 minutes | 1 hour | ≥ 14.4× | Immediate response; all hands |
| T1-FastBurn | P2 — Page | 30 minutes | 6 hours | ≥ 6× | Response within 30 minutes |
| T1-SlowBurn | P3 — Ticket | 6 hours | 24 hours | ≥ 3× | Response next business day |
| T2-FastBurn | P2 — Page | 30 minutes | 6 hours | ≥ 6× | Response within 1 hour |
| T2-SlowBurn | P3 — Ticket | 6 hours | 24 hours | ≥ 3× | Response next business day |
| T3-SlowBurn | P4 — Ticket | 24 hours | 72 hours | ≥ 2× | Scheduled sprint item |

**Alert rule format (Prometheus):**

```yaml
# Example: T1 Critical Burn — fires if error rate over 5m window is 14.4x the budget burn rate
# AND confirmed by 1h window to suppress transient spikes
- alert: T1SeatInventoryCriticalBurn
  expr: |
    (
      rate(http_requests_total{service="seat-inventory",code=~"5.."}[5m])
      / rate(http_requests_total{service="seat-inventory"}[5m])
    ) > (14.4 * 0.001)
    and
    (
      rate(http_requests_total{service="seat-inventory",code=~"5.."}[1h])
      / rate(http_requests_total{service="seat-inventory"}[1h])
    ) > (14.4 * 0.001)
  for: 2m
  labels:
    severity: critical
    tier: T1
  annotations:
    summary: "Seat Inventory critical SLO burn rate"
    runbook: "https://github.com/stagepass-aiman/stagepass-docs/blob/main/docs/runbooks/seat-inventory-slo-burn.md"
```

Every alert must have a corresponding runbook at
`stagepass-docs/docs/runbooks/<alert-name>.md`. The runbook must include:
diagnosis steps, mitigation options, escalation path, and rollback procedure.

---

## 5. Capacity Model

The capacity model defines the load assumptions that all performance thresholds
are measured against. Tests that do not replicate this load profile are not valid
compliance evidence.

### 5.1 Traffic Model

| Scenario | Requests Per Second | Duration | Notes |
|----------|-------------------|----------|-------|
| Normal sustained | 100 RPS | Continuous | Mixed read/write across all endpoints |
| Peak sustained | 300 RPS | Up to 1 hour | High-demand sale period; standard headroom |
| Flash sale burst | 1,000 RPS | 60 seconds | 10× normal; Kafka queue funnel engaged |
| Flash sale sustained | 300 RPS | Up to 4 hours | Post-burst stabilisation for high-demand events |

All RPS figures are platform-aggregate. Individual service RPS is lower.
The Seat Inventory Service is the bottleneck for booking; its rate governs
the flash sale queue drain rate.

### 5.2 Data Volume Model

| Entity | Scale | Notes |
|--------|-------|-------|
| Concurrent active events | Up to 100 | At peak: IPL season, festival season |
| Concurrent on-sale events with active seat maps | Up to 20 | Simultaneous WebSocket seat map rooms |
| Seats per event (maximum) | 50,000 | Stadium-scale; Wankhede, DY Patil |
| Tickets sold per day (peak) | 50,000 | Full stadium sell-out |
| Bookings per day (peak) | 10,000 | Average 5 tickets per booking |
| Concurrent checkout sessions | Up to 500 | Flash sale mode |
| Kafka events per second (peak) | ~2,000 | Booking sagas + seat state changes + notifications |
| Elasticsearch event documents | Up to 10,000 | All active and upcoming events |
| Qdrant vectors (events + chat corpus) | ~200,000 | Event descriptions + seating documents + policies |
| Redis memory (seat holds + JTI blocklist + Socket.IO adapter) | < 4 GB | See NFR-PERF-043 |
| PostgreSQL per-service storage (first year) | < 100 GB | Per T1 service |
| MongoDB event + layout storage | < 50 GB | Stadium layouts are large; MongoDBcompresses well |
| ClickHouse analytics events | ~500 M rows/year | Ticket sales + attendance events |

### 5.3 Kafka Throughput Estimates

| Topic | Peak Message Rate | Avg Message Size | Retention |
|-------|-----------------|-----------------|-----------|
| booking.commands | 100 msg/s | 2 KB | 7 days |
| booking.events | 150 msg/s | 3 KB | 30 days |
| seat.state-changes | 500 msg/s | 500 B | 24 hours |
| payment.events | 100 msg/s | 2 KB | 30 days |
| notification.outbox | 200 msg/s | 1 KB | 7 days |
| fraud.scored | 100 msg/s | 3 KB | 30 days |
| waitlist.offers | 20 msg/s | 1 KB | 7 days |
| disbursement.events | 5 msg/s | 2 KB | 90 days |
| analytics.ingest | 500 msg/s | 1 KB | 7 days |
| *.dlq (all DLQ topics) | < 1 msg/s | varies | 14 days |

### 5.4 Local Stack Memory Budget

The full local Docker Compose stack (all 17 services + infrastructure) must run
on a developer laptop with 16 GB RAM. The hard budget is 12 GB.

| Component | Target Heap/Memory |
|-----------|-------------------|
| Java services (×5, T1) | 512 MB each = 2.56 GB |
| Java services (×3, T2) | 384 MB each = 1.15 GB |
| Node services (×5) | 256 MB each = 1.28 GB |
| Python services (×5) | 384 MB each = 1.92 GB |
| PostgreSQL (shared instance, multiple DBs) | 512 MB |
| MongoDB | 512 MB |
| Redis | 256 MB |
| Kafka + Zookeeper | 512 MB |
| Elasticsearch | 512 MB |
| Qdrant | 256 MB |
| ClickHouse | 256 MB |
| MinIO | 128 MB |
| Observability (Prometheus + Grafana + Loki + Jaeger + OTel Collector) | 1.5 GB |
| API Gateway | 384 MB |
| **Total** | **~11.8 GB** |

Services use JVM ergonomics or container memory limits. Java services must be
started with `-Xmx` matching the above budget. Exceeding the local budget is a
build-breaking condition checked by `scripts/check-memory.sh`.

---

## 6. Performance Requirements — NFR-PERF

### Critical Path Latency

---

**NFR-PERF-001**

| Field | Value |
|-------|-------|
| Statement | A seat hold request (Customer selects a seat → HELD state confirmed) must complete within 500 ms at the 99th percentile. |
| Threshold | p99 ≤ 500 ms under 300 RPS sustained load; p50 ≤ 100 ms |
| Method | k6 load test targeting the seat hold endpoint at 300 RPS sustained for 5 minutes; percentiles reported by k6 summary; CI gate blocks on regression |
| Traces To | PRD G-03, G-04, FR-C-003 AC5, FR-P-001 AC5 |
| Notes | The hold is enforced by a Redis atomic operation (SETNX + EXPIRE). The 500 ms budget includes: Gateway routing (~10 ms), gRPC call Booking → Seat Inventory (~30 ms), Redis operation (~5 ms), and response marshalling. Pessimistic or optimistic DB locking must not be in this path. Any design that requires a PostgreSQL row lock to create a hold violates this NFR. See ADR-006. |

---

**NFR-PERF-002**

| Field | Value |
|-------|-------|
| Statement | The synchronous portion of checkout (PENDING Booking record created, idempotency key returned) must complete within 2 seconds at the 99th percentile. |
| Threshold | p99 ≤ 2,000 ms under 300 RPS sustained load; p50 ≤ 500 ms |
| Method | k6 load test against checkout endpoint at 300 RPS for 5 minutes; k6 summary percentiles; CI gate |
| Traces To | PRD G-03, FR-C-004 AC5 |
| Notes | The synchronous portion is bounded: create Booking record (PostgreSQL write), publish booking.commands event to Kafka via Outbox. Payment initiation is asynchronous after this response. The 2 s budget is conservative; the actual path should complete in ~300 ms under normal conditions. The budget accommodates Gateway overhead, JWT validation, and Outbox table write. |

---

**NFR-PERF-003**

| Field | Value |
|-------|-------|
| Statement | The end-to-end booking saga (checkout submit → booking confirmation WebSocket push received by Customer) must complete within 10 seconds at the 99th percentile. |
| Threshold | p99 ≤ 10,000 ms measured from the POST /checkout timestamp to the WebSocket `booking.confirmed` event receipt on the client |
| Method | Playwright E2E test with server-side timing injection; Jaeger trace span duration for the full saga; measured in staging with realistic stub latencies for Payment Service |
| Traces To | PRD G-03, G-04, FR-C-004 AC6 |
| Notes | The saga path is: checkout submit → seat hold confirm (gRPC) → payment initiation (REST) → payment confirmed (webhook/Kafka) → seat commit (Kafka) → ticket issuance (Kafka) → notification (Kafka → WebSocket). Each Kafka hop adds ~50 ms under normal load. The 10 s budget accommodates up to 3 Kafka hops, 2 gRPC calls, and 1 REST call with stub latencies. A real Razorpay integration would consume ~2 s of this budget on the payment step. |

---

**NFR-PERF-004**

| Field | Value |
|-------|-------|
| Statement | QR code check-in verification (token received → VALID/INVALID response returned) must complete within 200 ms at the 99th percentile. |
| Threshold | p99 ≤ 200 ms; p50 ≤ 50 ms; under a sustained burst of 50 concurrent scans (realistic gate concurrency) |
| Method | k6 load test with 50 VUs against the check-in verification endpoint; HMAC-SHA256 verification must be the only computation — no database call in the synchronous path |
| Traces To | PRD G-09, FR-P-002 AC1 AC2, NFR-SEC-005 |
| Notes | Signature verification is O(1) using a cached per-event HMAC key. The per-event key is loaded at Check-in Service startup (or on first scan for that event) and held in process memory. No Redis or PostgreSQL call is required in the verification hot path. The async Kafka publish (`ticket.checked-in` event) happens after the 200 ms response. This design enables offline resilience (NFR-AVAIL-006). See ADR-003 (gRPC: Check-in → Ticket Service). |

---

**NFR-PERF-005**

| Field | Value |
|-------|-------|
| Statement | A seat state change (e.g., AVAILABLE → HELD) must be pushed to all connected clients subscribed to that event's seat map room within 1 second of the state change being committed. |
| Threshold | p99 ≤ 1,000 ms from commit timestamp (Redis write) to client WebSocket message receipt |
| Method | Integration test: two browser clients subscribed to an event room; one client triggers a hold; assert the second client receives the state change event within 1 s; measured via browser performance API in Playwright |
| Traces To | PRD G-08, FR-P-001 AC2, FR-C-003 AC3 |
| Notes | Path: Redis commit → Seat Inventory publishes `seat.state-changed` Kafka event → Notification Service consumes → emits to Socket.IO room → client receives. Redis Socket.IO adapter is required for horizontal Notification Service scaling. The Kafka lag budget for this path is ~100 ms under normal load. |

---

**NFR-PERF-006**

| Field | Value |
|-------|-------|
| Statement | Keyword event search must return ranked results within 500 ms at the 99th percentile under 100 RPS sustained load. |
| Threshold | p99 ≤ 500 ms; error rate < 0.1%; at 100 RPS for 5 minutes |
| Method | k6 load test against the search endpoint with a representative query set (50 distinct queries cycled); k6 summary; CI gate |
| Traces To | PRD G-07, FR-C-002 AC1, FR-AI-002 |
| Notes | Keyword search hits Elasticsearch (BM25). Semantic blending (Qdrant) may be deferred to the background if blended latency cannot be achieved within budget. The 500 ms budget assumes a well-tuned Elasticsearch cluster with a cached query plan. |

---

**NFR-PERF-007**

| Field | Value |
|-------|-------|
| Statement | Semantic (natural language) event search must return ranked results within 800 ms at the 99th percentile under 100 RPS sustained load. |
| Threshold | p99 ≤ 800 ms; error rate < 0.1%; at 100 RPS for 5 minutes |
| Method | k6 load test with natural language queries; latency measured end-to-end at the API Gateway; embeddings must be pre-computed (not on-the-fly per query) for the event corpus |
| Traces To | PRD G-07, FR-C-002 AC3, FR-AI-002 AC3 |
| Notes | Query flow: Search Service generates query embedding (FastAPI → embedding model) + Qdrant ANN query + Elasticsearch BM25 query + blend. The embedding model must be served locally (not via external API call) to meet this budget. Model inference p99 must be < 200 ms to leave headroom for Qdrant and blend. |

---

**NFR-PERF-008**

| Field | Value |
|-------|-------|
| Statement | Image-based event search must return results within 3,000 ms at the 99th percentile. |
| Threshold | p99 ≤ 3,000 ms; error rate < 0.5% |
| Method | k6 test with a set of representative poster images; measure end-to-end at gateway; image upload size capped at 5 MB |
| Traces To | PRD G-07, FR-C-002 AC4, FR-AI-003 AC3 |
| Notes | Path: image upload → CLIP embedding generation (GPU-accelerated locally, CPU in dev) → Qdrant ANN → result ranking. The 3 s budget accommodates CPU-based CLIP inference (~1.5 s on modern hardware). If GPU is available, p99 target is 1,500 ms. |

---

**NFR-PERF-009**

| Field | Value |
|-------|-------|
| Statement | The chatbot first-token latency (Customer sends a message → first streamed token received) must be within 2,500 ms at the 99th percentile. |
| Threshold | p99 ≤ 2,500 ms first-token; streaming must be chunk-by-chunk, not a single block response |
| Method | Playwright E2E test measuring time from submit to first text node update in the chat window; measured across 20 representative queries |
| Traces To | PRD G-07, FR-C-008 AC3, FR-AI-004 |
| Notes | Path: Customer query → Chatbot Service (FastAPI) → Qdrant retrieval (~100 ms) → LLM prompt construction → LLM streaming response. First-token latency depends heavily on LLM provider. Local LLM (Ollama) in dev may not meet this threshold; a cloud LLM with streaming API (OpenAI, Anthropic) should achieve it. The RAG retrieval step must complete before the LLM call begins (sequential, not parallel, unless context is short). |

---

**NFR-PERF-010**

| Field | Value |
|-------|-------|
| Statement | A newly published event must appear in keyword and semantic search results within 30 seconds of the Organiser publishing it. |
| Threshold | Lag from Event Service `event.published` Kafka event emission to Elasticsearch document indexed + Qdrant vector indexed ≤ 30 s; measured p99 |
| Method | Integration test: publish event via API; poll search endpoint until result appears; assert elapsed time < 30 s; repeat 20 times |
| Traces To | PRD G-02, G-07, FR-C-002 AC7, FR-AI-002 AC4, FR-AI-003 AC4 |
| Notes | Path: Event Service → Kafka `event.published` → Search Service consumes → Elasticsearch index write + embedding generation + Qdrant upsert. The 30 s budget includes Kafka propagation (~100 ms), embedding generation (~1 s), and index write latency. |

---

**NFR-PERF-011**

| Field | Value |
|-------|-------|
| Statement | A fraud score must be published to the `fraud.scored` Kafka topic within 10 seconds of a booking being placed. |
| Threshold | p99 ≤ 10,000 ms from `booking.confirmed` event timestamp to `fraud.scored` event timestamp (both in Kafka headers) |
| Method | Integration test: place booking; consume from `fraud.scored` topic; assert timestamp delta < 10 s; measured across 100 bookings under load |
| Traces To | PRD G-07, FR-AI-007 AC1 |
| Notes | Fraud Detection Service consumes from `booking.events`, scores asynchronously, and publishes to `fraud.scored`. The 10 s budget is generous enough to allow model inference (~500 ms), velocity lookups (Redis), and device fingerprint checks. |

---

**NFR-PERF-012**

| Field | Value |
|-------|-------|
| Statement | Personalised event recommendations must be served within 500 ms at the 99th percentile (Redis cache hit path). |
| Threshold | p99 ≤ 500 ms on cache hit; p99 ≤ 3,000 ms on cache miss (model inference); cache TTL = 1 hour |
| Method | k6 test with authenticated user sessions; assert 99th percentile at gateway; cache hit ratio must be > 90% for users with prior activity |
| Traces To | PRD G-07, FR-C-002 AC5, FR-AI-001 AC2 |
| Notes | Recommendation inference is offline (daily batch). The serving path is purely Redis cache lookup. Cache miss triggers a synchronous inference call, which may exceed 500 ms. The 3 s cache-miss budget is acceptable because cold-start users see the popularity fallback. |

---

**NFR-PERF-013**

| Field | Value |
|-------|-------|
| Statement | Under flash sale load (10× normal RPS for 60 seconds), the platform must not oversell a single seat, and the HTTP error rate must not exceed 0.5%. |
| Threshold | Oversell count = 0; error rate ≤ 0.5% over the 60 s burst; measured at 1,000 RPS |
| Method | k6 flash sale load test: 1,000 VUs all attempting to hold seats for the same event simultaneously; assert no duplicate holds succeed for the same seat; assert seats booked ≤ total seat count; assert HTTP 5xx rate ≤ 0.5% |
| Traces To | PRD G-03, FR-P-003 AC4 |
| Notes | The flash sale queue (Kafka funnel) is the mechanism. Requests above the drain rate are queued, not rejected. Queue position is communicated to clients via WebSocket. The Seat Inventory Service's Redis atomic SETNX guarantees mutual exclusion even when multiple Booking Service instances process queue entries concurrently. See ADR-006, ADR-007. |

---

### Additional Performance Requirements

---

**NFR-PERF-014**

| Field | Value |
|-------|-------|
| Statement | Joining the waitlist for a sold-out event must complete (acknowledgement returned) within 500 ms at the 99th percentile. |
| Threshold | p99 ≤ 500 ms; error rate < 0.1% |
| Method | k6 test targeting the waitlist join endpoint at 100 RPS |
| Traces To | PRD G-03, FR-C-006 AC1 |
| Notes | Waitlist join writes to Redis sorted set (ZADD with timestamp score). This is an O(log N) operation. The sorted set scales to 100,000 entries without performance degradation. |

---

**NFR-PERF-015**

| Field | Value |
|-------|-------|
| Statement | WebSocket connection establishment (HTTP upgrade handshake to first message received) must complete within 1,000 ms at the 99th percentile. |
| Threshold | p99 ≤ 1,000 ms from connection initiation to Socket.IO `connect` event |
| Method | Playwright test measuring Socket.IO connect time across 50 concurrent connections; measured in CI against a running Notification Service |
| Traces To | PRD G-08, FR-P-001, FR-C-009 AC1 |
| Notes | Socket.IO uses long-polling as a fallback when WebSocket is unavailable. The 1 s budget covers the worst-case long-polling fallback scenario. Pure WebSocket upgrades complete in < 100 ms. |

---

**NFR-PERF-016**

| Field | Value |
|-------|-------|
| Statement | PDF ticket generation (booking confirmed → PDF available in MinIO) must complete within 5,000 ms at the 99th percentile. |
| Threshold | p99 ≤ 5,000 ms from `booking.confirmed` event to MinIO object availability |
| Method | Integration test: trigger booking confirmation; poll MinIO for object; measure elapsed time |
| Traces To | PRD G-03, FR-C-005 AC3 |
| Notes | PDF generation is asynchronous and initiated by the Ticket Service consuming from Kafka. The 5 s budget accommodates QR code embedding, PDF rendering, and MinIO upload. PDF download URL is a pre-signed MinIO URL (TTL: 24 hours). |

---

**NFR-PERF-017**

| Field | Value |
|-------|-------|
| Statement | The interactive seat map for a stadium-scale event (50,000 seats) must render the initial state within 3,000 ms on a standard connection (10 Mbps). |
| Threshold | p99 ≤ 3,000 ms from page load to seat map interactive; Lighthouse performance score ≥ 70 on the event detail page |
| Method | Lighthouse CI test on the event detail page with a seeded 50,000-seat event; measure Time to Interactive |
| Traces To | PRD G-08, FR-C-003 AC2, FR-V-002 AC2 |
| Notes | The seat map initial state payload must be paginated or virtualised. Sending 50,000 seat objects in a single HTTP response is not acceptable. Only the visible viewport's seats are rendered in full; others are lazy-loaded as the Customer pans/zooms. The SVG component must implement windowing (similar to React Virtual for DOM, but for SVG). |

---

**NFR-PERF-018**

| Field | Value |
|-------|-------|
| Statement | The Demand Prediction Service must generate a forecast for a newly published event within 5 minutes of the `event.published` Kafka event. |
| Threshold | Lag from `event.published` to `demand.forecast-ready` event ≤ 5 minutes; p99 |
| Method | Integration test: publish event; consume `demand.forecast-ready` topic; assert lag < 5 minutes |
| Traces To | PRD FR-AI-005 AC5, FR-O-005 AC5 |
| Notes | Demand prediction is a batch inference operation triggered by the `event.published` event. Initial forecasts use category + location priors. Subsequent forecasts improve as sales velocity data accumulates. |

---

**NFR-PERF-019**

| Field | Value |
|-------|-------|
| Statement | The No-Show Prediction Service must generate predictions for qualifying events within 30 minutes of the 24-hour pre-event trigger. |
| Threshold | Prediction available in Organiser dashboard ≤ 30 minutes after trigger time (event_start - 24 hours) |
| Method | Integration test with mocked event time; assert prediction widget populated within 30 minutes of trigger |
| Traces To | PRD FR-AI-008 AC1 |
| Notes | No-show prediction is batch inference. A Kafka-scheduled event (or cron via K8s CronJob) triggers prediction at T-24h for all qualifying events (>50% capacity sold). |

---

**NFR-PERF-020**

| Field | Value |
|-------|-------|
| Statement | Event cancellation fan-out (Organiser cancels event → all booking refund sagas initiated) must begin within 30 seconds of the cancellation command being processed. |
| Threshold | All booking refund saga messages published to Kafka within 30 s of `event.cancelled` event |
| Method | Integration test: cancel event with 100 confirmed bookings; count Kafka `booking.refund-requested` messages; assert all 100 published within 30 s |
| Traces To | PRD G-06, FR-O-004 AC3, FR-P-011 |
| Notes | Fan-out is accomplished via Kafka partition parallelism. The Event Service publishes one `event.cancelled` event; the Booking Service consumes it and publishes individual `booking.refund-requested` commands per booking. At 10,000 bookings, fan-out must complete within 5 minutes (Kafka throughput model: 150 msg/s booking events). |

---

**NFR-PERF-021**

| Field | Value |
|-------|-------|
| Statement | Surge pricing evaluation (at seat hold creation time) must add no more than 50 ms to the seat hold latency. |
| Threshold | Surge evaluation overhead ≤ 50 ms; total seat hold p99 must remain ≤ 500 ms (NFR-PERF-001) |
| Method | Unit test with mocked surge rules; measure evaluation time for up to 10 rules per event; integration test in the full hold path |
| Traces To | PRD FR-AI-006 AC1, FR-AI-006 AC2 |
| Notes | Surge rules are evaluated in-memory against the current seat count (cached from Redis, refreshed every 10 seconds). No synchronous DB query is performed during surge evaluation in the hold path. |

---

**NFR-PERF-022**

| Field | Value |
|-------|-------|
| Statement | The Organiser live operations dashboard must reflect booking metrics with a maximum staleness of 60 seconds. |
| Threshold | All dashboard metrics (tickets sold, revenue, seat availability) updated within 60 s of the underlying event |
| Method | Integration test: complete a booking; assert Organiser dashboard metrics reflect the booking within 60 s |
| Traces To | PRD FR-O-005 AC2 |
| Notes | Metrics are served by the Analytics Service (ClickHouse read). Live counter deltas are pushed via WebSocket immediately on `booking.confirmed`. The 60 s staleness applies to the batch-aggregated metrics, not the live counter. |

---

**NFR-PERF-023**

| Field | Value |
|-------|-------|
| Statement | The Admin platform health dashboard must load and display all health metrics within 3,000 ms at the 99th percentile. |
| Threshold | p99 ≤ 3,000 ms Time to Interactive for the Admin health dashboard page |
| Method | Lighthouse CI test on Admin dashboard page |
| Traces To | PRD FR-A-006 AC1 |
| Notes | The Admin dashboard embeds Grafana. Grafana dashboard load time depends on datasource query performance (Prometheus, Loki). Slow Prometheus queries must be pre-aggregated using recording rules. |

---

**NFR-PERF-024**

| Field | Value |
|-------|-------|
| Statement | Revenue split computation at booking confirmation must complete within 100 ms. |
| Threshold | RevenueSplit record created within 100 ms of `booking.seats-committed` event |
| Method | Unit test with mocked inputs; integration test measuring Booking Service saga step latency via OpenTelemetry spans |
| Traces To | PRD G-05, FR-P-001 (revenue split is part of the saga), Section 7.3 |
| Notes | Revenue split is a pure arithmetic computation (3 multiplications) + one PostgreSQL write. It must not involve an external service call. The split amounts are computed once and written atomically with the booking state transition. |

---

**NFR-PERF-025**

| Field | Value |
|-------|-------|
| Statement | A waitlist offer notification must be delivered to the eligible Customer within 5 seconds of a seat becoming available. |
| Threshold | p99 ≤ 5,000 ms from seat release event to WebSocket `waitlist.offer-received` event on Customer client |
| Method | Integration test: release a held seat; measure time to WebSocket event on a connected waitlist Customer client |
| Traces To | PRD G-03, FR-C-006 AC4 |
| Notes | Path: seat TTL expiry in Redis → Redis keyspace notification → Seat Inventory publishes `seat.released` → Waitlist Service consumes → pops top entry → creates WaitlistOffer → Notification Service pushes WebSocket event. Redis keyspace notification is a non-deterministic trigger; the 5 s budget accounts for polling fallback (1 s interval) in environments where keyspace notifications are unavailable. |

---

**NFR-PERF-026**

| Field | Value |
|-------|-------|
| Statement | KYC state transitions (Admin approve/reject) must complete and the applicant must receive notification within 10 seconds. |
| Threshold | p99 ≤ 10,000 ms from Admin action to email notification dispatched |
| Method | Integration test: Admin approves KYC; assert Organiser notification published to Kafka within 10 s |
| Traces To | PRD FR-O-001 AC5, FR-A-001 AC5 |

---

**NFR-PERF-027**

| Field | Value |
|-------|-------|
| Statement | Event vector embeddings for semantic and image search must be updated in Qdrant within 60 seconds of an event's poster image or metadata being changed. |
| Threshold | Qdrant vector updated ≤ 60 s after `event.metadata-updated` Kafka event |
| Method | Integration test: update event metadata; query Qdrant for updated vector; assert update within 60 s |
| Traces To | PRD FR-AI-002 AC4, FR-AI-003 AC4 |

---

**NFR-PERF-028**

| Field | Value |
|-------|-------|
| Statement | Flash sale queue position updates must be pushed to queued Customers via WebSocket within 2 seconds of each position change. |
| Threshold | p99 ≤ 2,000 ms from queue position change event to WebSocket delivery |
| Method | Integration test with 100 simulated queued Customers; measure WebSocket event delay per position update |
| Traces To | PRD FR-P-003 AC3 |

---

**NFR-PERF-029**

| Field | Value |
|-------|-------|
| Statement | Booking cancellation (Customer initiates → refund confirmed response) synchronous portion must complete within 3,000 ms at the 99th percentile. |
| Threshold | p99 ≤ 3,000 ms for the cancellation request response (refund amount confirmed, saga initiated) |
| Method | k6 test against the cancellation endpoint |
| Traces To | PRD G-06, FR-C-007 AC3 |

---

**NFR-PERF-030**

| Field | Value |
|-------|-------|
| Statement | The platform must sustain 100 RPS normal throughput and burst to 300 RPS for up to 60 seconds without degradation in error rate or p99 latency. |
| Threshold | Sustained: 100 RPS, error rate < 0.1%, all PERF thresholds met. Burst: 300 RPS for 60 s, error rate < 0.5%, p99 latency < 2× the sustained target |
| Method | k6 ramp test: 100 VUs for 5 min, spike to 300 VUs for 60 s, ramp down to 100 VUs; assert error rates and latencies at each stage |
| Traces To | PRD G-03, Constraints C-01 (capacity must be realistic for Indian market scale) |

---

**NFR-PERF-031**

| Field | Value |
|-------|-------|
| Statement | The platform must support at least 5,000 concurrent WebSocket connections across all event rooms without degraded message delivery. |
| Threshold | 5,000 concurrent Socket.IO connections; message delivery p99 ≤ 1,000 ms; no dropped messages |
| Method | k6 WebSocket load test with 5,000 VUs connected to Socket.IO; measure message delivery time across all connections |
| Traces To | PRD G-08, FR-P-001 AC4 |
| Notes | Horizontal scaling of the Notification Service via Redis Socket.IO adapter is required to meet this threshold. |

---

**NFR-PERF-032**

| Field | Value |
|-------|-------|
| Statement | Kafka consumers must maintain consumer group lag below 5,000 messages during peak booking periods. |
| Threshold | Consumer lag ≤ 5,000 messages per partition on `booking.events` and `seat.state-changes` topics under peak load |
| Method | Prometheus Kafka exporter metric `kafka_consumergroup_lag`; Alertmanager alert fires if lag > 5,000 for > 2 minutes |
| Traces To | PRD G-03, G-08 (real-time seat state), Section 5 Capacity Model |

---

**NFR-PERF-033**

| Field | Value |
|-------|-------|
| Statement | The gRPC call from Booking Service to Seat Inventory Service (seat availability check at checkout) must complete within 300 ms at the 99th percentile. |
| Threshold | p99 ≤ 300 ms; measured as the gRPC call duration span in Jaeger |
| Method | Integration test with Testcontainers; gRPC stub in Seat Inventory; measure Booking Service OpenTelemetry span `seat-inventory.check-availability` |
| Traces To | PRD FR-P-005 AC5, ADR-003 (gRPC communication pattern) |

---

**NFR-PERF-034**

| Field | Value |
|-------|-------|
| Statement | The gRPC call from Check-in Service to Ticket Service (QR token pre-validation, non-hot-path) must complete within 200 ms at the 99th percentile. |
| Threshold | p99 ≤ 200 ms; this is the gRPC pre-validation path, not the hot-path HMAC check |
| Method | Integration test with Testcontainers; measure OpenTelemetry span `ticket.validate-token` in Check-in Service |
| Traces To | PRD FR-P-002, ADR-003 |
| Notes | The primary check-in path (HMAC signature verification) requires no gRPC call. This gRPC path is used only for administrative validation (e.g., full revocation status check initiated by Admin). |

---

**NFR-PERF-035**

| Field | Value |
|-------|-------|
| Statement | The event detail page (including seat map for events < 5,000 seats) must reach Time to Interactive within 2,500 ms on a 10 Mbps connection. |
| Threshold | Lighthouse Time to Interactive ≤ 2,500 ms; Largest Contentful Paint ≤ 2,000 ms |
| Method | Lighthouse CI in the storefront CI pipeline; measured against a seeded event with 5,000 seats |
| Traces To | PRD FR-C-003 AC1 |

---

**NFR-PERF-036**

| Field | Value |
|-------|-------|
| Statement | The Customer's ticket history page must load and render all ticket records within 2,000 ms at the 99th percentile, for up to 100 historical bookings. |
| Threshold | p99 ≤ 2,000 ms for a Customer with 100 past bookings |
| Method | k6 test with seeded Customers; Lighthouse on the "My Tickets" page |
| Traces To | PRD FR-C-005 AC1 |
| Notes | Ticket history is served from the Booking Service read model (MongoDB CQRS). Pagination is cursor-based; the first page must load within the threshold regardless of total booking count. |

---

**NFR-PERF-037**

| Field | Value |
|-------|-------|
| Statement | The Organiser event creation form must submit and receive a DRAFT event confirmation within 2,000 ms at the 99th percentile. |
| Threshold | p99 ≤ 2,000 ms from form submit to DRAFT event ID returned |
| Method | k6 test against the Event Service event creation endpoint |
| Traces To | PRD FR-O-003 AC1 |

---

**NFR-PERF-038**

| Field | Value |
|-------|-------|
| Statement | Check-in records accumulated during offline mode must be synchronised to the Check-in Service within 30 seconds of network connectivity being restored. |
| Threshold | All offline check-in records synced ≤ 30 s after connectivity restored; no records lost |
| Method | Integration test: disconnect Check-in Service from network; accumulate 50 offline check-ins; restore connectivity; assert all 50 records appear in the service within 30 s |
| Traces To | PRD FR-P-002 AC7, NFR-AVAIL-006 |

---

**NFR-PERF-039**

| Field | Value |
|-------|-------|
| Statement | Disbursement trigger (Admin initiates → all DisbursementRecords created and notifications dispatched) must complete for events with up to 10,000 bookings within 5 minutes. |
| Threshold | All DisbursementRecords created and Venue/Organiser notified ≤ 5 minutes for 10,000 bookings |
| Method | Integration test with seeded disbursement; measure time from Admin trigger to final DisbursementRecord created |
| Traces To | PRD G-05, FR-A-005 AC3 |

---

**NFR-PERF-040**

| Field | Value |
|-------|-------|
| Statement | Admin platform configuration changes must take effect across all running service instances within 60 seconds without a service restart. |
| Threshold | Configuration propagated and applied ≤ 60 s after Admin saves change |
| Method | Integration test: change platform fee percentage via Admin API; assert new bookings use new fee within 60 s; assert existing bookings are not retroactively changed |
| Traces To | PRD FR-A-004 AC2 |
| Notes | Implemented via Spring Cloud Config refresh endpoint (`/actuator/refresh`) triggered by a Kafka `config.updated` event, or by Vault dynamic secret leases. |

---

**NFR-PERF-041**

| Field | Value |
|-------|-------|
| Statement | Bulk fraud hold (Admin bulk-holds all bookings from a device fingerprint cluster) must process up to 500 bookings within 2 minutes. |
| Threshold | 500 bookings transitioned to HELD state ≤ 2 minutes |
| Method | Integration test with 500 seeded bookings; Admin bulk-hold action; measure completion time |
| Traces To | PRD FR-A-003 AC4 |

---

**NFR-PERF-042**

| Field | Value |
|-------|-------|
| Statement | Services must reach a healthy, ready-to-serve state from a cold start within the following time limits. |
| Threshold | Java services (Spring Boot / Quarkus): cold start to `/health/ready` = OK within 30 seconds. Node.js services: within 10 seconds. Python services: within 10 seconds (excluding ML model load; models are pre-loaded in a separate readiness step with a 60 s timeout). |
| Method | Measure in CI: Docker container starts; Kubernetes readiness probe reports READY; assert elapsed time < threshold. Measured with a 1 s probe interval. |
| Traces To | PRD G-10, G-08 (fast recovery after restarts) |
| Notes | Java 21 virtual threads and Spring Boot 3 native compilation (GraalVM) can achieve < 5 s cold start if needed, but are not required by default. Quarkus (Payment Service) achieves < 5 s JVM mode, < 0.5 s native. Native compilation is an optimisation, not a requirement in Phase 2–8. |

---

**NFR-PERF-043**

| Field | Value |
|-------|-------|
| Statement | The full local Docker Compose stack (all services + infrastructure) must not exceed 12 GB total RAM. |
| Threshold | `docker stats` aggregate memory usage ≤ 12 GB after all services reach healthy state |
| Method | `scripts/check-memory.sh` runs `docker stats --no-stream` and sums memory; CI gate blocks if sum > 12 GB; also run manually by developer before PR merge |
| Traces To | PRD Constraints C-01 (solo developer on a 16 GB laptop) |
| Notes | Per-service memory limits are defined in Section 5.4. JVM services must use container-aware heap sizing (`-XX:MaxRAMPercentage`). Python services must use `--max-requests` in Gunicorn to reclaim memory from long-running workers. |

---

## 7. Reliability Requirements — NFR-REL

---

**NFR-REL-001**

| Field | Value |
|-------|-------|
| Statement | All write endpoints must be idempotent: repeating the same request with the same `Idempotency-Key` header within the key's TTL must return the original response without re-executing the operation. |
| Threshold | Zero duplicate side effects observed in any load test or chaos test; duplicate request detection p99 < 100 ms (cache lookup overhead) |
| Method | Integration test for every write endpoint: execute operation once; capture response; re-execute with same key; assert second response is identical and no new DB record created; verify via OpenTelemetry spans that second execution did not reach the business logic layer |
| Traces To | PRD FR-P-004 AC1 AC2 |
| Notes | Implementation: service checks an idempotency key store (Redis, TTL 24 hours) before processing. On cache miss: process and store result. On cache hit: return stored result immediately. The idempotency key namespace is per-endpoint to prevent cross-endpoint key collisions. |

---

**NFR-REL-002**

| Field | Value |
|-------|-------|
| Statement | All Kafka consumers must be idempotent: processing the same message twice (at-least-once delivery guarantee) must produce the same final state as processing it once. |
| Threshold | Zero unintended duplicate side effects in any Testcontainers-based duplicate-delivery test; verified for all 9+ Kafka consumer implementations |
| Method | Testcontainers integration test for each consumer: publish the same event twice consecutively; assert the resulting database state is identical to single-delivery state; measure via row count and revision number |
| Traces To | PRD FR-P-004 AC3, G-06 (refund idempotency), FR-P-011 |
| Notes | Idempotency key for Kafka consumers: the Kafka message key (correlating to booking ID, seat ID, etc.) is stored in the consumer's own database as a processed-event log. Before processing: check if key exists. If yes: skip. If no: process + upsert key. This is the standard pattern described in Kleppmann DDIA Chapter 11. |

---

**NFR-REL-003**

| Field | Value |
|-------|-------|
| Statement | Every forward step in every saga (booking, event cancellation, Venue suspension cascade) must have a documented, implemented, and tested compensation action. |
| Threshold | 100% saga compensation coverage: every saga step listed in a compensation matrix; every compensation action covered by at least one Testcontainers integration test that injects a failure at that step and asserts the compensated state |
| Method | Compensation matrix document (one per saga, in `stagepass-docs/docs/architecture/sequences/`); integration tests that kill a dependent service mid-saga using Testcontainers stop/start; assert booking reaches CANCELLED state and seat is released |
| Traces To | PRD G-06, FR-P-005, FR-C-004 AC7 |
| Notes | Booking saga compensation chain: Payment fails → seats released + booking CANCELLED. Seat commit fails after payment confirmed → payment refunded + seats released + booking CANCELLED. Ticket issuance fails → payment refunded + seats released + booking CANCELLED. Each compensation must be implemented in the Booking Service orchestrator and must not require manual intervention. |

---

**NFR-REL-004**

| Field | Value |
|-------|-------|
| Statement | Seat holds must auto-expire and release the seat exactly once after the configured hold window (default: 10 minutes), enforced by Redis TTL, not application polling. |
| Threshold | Hold expiry occurs within 5 seconds of the TTL expiring (Redis keyspace notification or TTL polling); zero holds persist beyond TTL + 5 s |
| Method | Integration test: create a hold with a 30 s TTL (test override); assert seat transitions to AVAILABLE within 35 s without any application timer; verify via Redis key existence check |
| Traces To | PRD G-04, FR-C-003 AC6 AC8, Section 7.1 (SeatHold) |
| Notes | Redis TTL is the authoritative timer. Application-layer polling is an acceptable supplement but must not be the primary release mechanism. On Redis restart, holds must be reconstructed from the database or treated as expired (the safer default). See ADR-006. |

---

**NFR-REL-005**

| Field | Value |
|-------|-------|
| Statement | All events that must not be lost (booking state changes, payment results, seat commits, disbursement triggers) must be published using the Outbox pattern: database write and event publish are atomic within the same local transaction. |
| Threshold | Zero event loss measurable in chaos tests where the Kafka broker is unavailable for up to 60 seconds during an active booking saga; all events are eventually published after broker recovery |
| Method | Testcontainers chaos test: create booking; stop Kafka container; complete saga steps; restart Kafka; assert all expected events appear in topics (none lost); measured via Kafka offset comparison |
| Traces To | PRD FR-P-005 AC4 |
| Notes | Outbox table schema: `outbox_events(id UUID PK, aggregate_id UUID, topic VARCHAR, payload JSONB, created_at TIMESTAMPTZ, published_at TIMESTAMPTZ NULL)`. A dedicated Outbox publisher thread (or Debezium CDC connector) polls unpublished rows and publishes to Kafka. Published rows are marked (not deleted) to allow replay. See ADR-005. |

---

**NFR-REL-006**

| Field | Value |
|-------|-------|
| Statement | All synchronous outbound service calls must implement a consistent retry policy with exponential backoff and jitter, and a circuit breaker that prevents cascading failures. |
| Threshold | Retry policy: max 3 attempts, backoff = 100ms × 2^attempt + rand(0, 100ms), max cap 5s, retryable on 429/503/504 only. Circuit breaker: opens after 5 consecutive failures, half-open after 30s, closes after 3 consecutive successes in half-open. Zero services implement custom retry logic that deviates from this policy. |
| Method | Unit test each service's HTTP/gRPC client configuration against these parameters; integration test with a Testcontainers WireMock stub returning 503 for N requests; assert retry behaviour matches policy; assert circuit breaker opens after 5 failures |
| Traces To | PRD FR-P-009, FR-P-007 (fail-closed services), FR-P-008 (fail-open services) |
| Notes | Java: Spring Retry + Resilience4j CircuitBreaker. Node: axios-retry + opossum circuit breaker. Python: tenacity for retries + pybreaker for circuit breaking. The retry configuration must be externalised (configurable per service via environment variable or Config Server) so it can be adjusted without code change. |

---

**NFR-REL-007**

| Field | Value |
|-------|-------|
| Statement | Every Kafka topic must have a corresponding dead-letter queue (DLQ) topic. A Prometheus alert must fire when any DLQ topic has depth > 0 for more than 5 continuous minutes. |
| Threshold | DLQ topic exists for every application topic; Alertmanager alert `KafkaDLQDepthNonZero` fires within 6 minutes of first DLQ message; runbook linked in alert annotations |
| Method | Kafka topic listing in CI validates DLQ topics exist; Prometheus metric `kafka_topic_partition_current_offset - kafka_consumergroup_current_offset` for DLQ consumer groups; alert rule tested in Alertmanager unit test framework |
| Traces To | PRD FR-P-005 AC6, G-10 |
| Notes | DLQ naming convention: `<original-topic>.dlq`. DLQ consumer group is separate from main consumer group. DLQ retention: 14 days (messages are investigated and replayed or discarded by Admin). Replay tooling must exist — Admin must not need to write code to replay a DLQ message. |

---

**NFR-REL-008**

| Field | Value |
|-------|-------|
| Statement | The system must never oversell a seat: no two confirmed bookings may reference the same seat for the same event. |
| Threshold | Oversell count = 0, verified by database constraint + application-level test + flash sale load test |
| Method | Database unique constraint on `(event_id, seat_id)` in the booking write model; flash sale k6 load test (NFR-PERF-013) asserts no oversell; a nightly audit query compares confirmed bookings against seat capacity |
| Traces To | PRD G-03, G-04, FR-C-003 AC7 |
| Notes | The unique constraint is the safety net. The Redis atomic hold (NFR-REL-004) is the performance mechanism. The constraint must fire if the Redis mechanism fails — defence in depth. See ADR-006. |

---

**NFR-REL-009**

| Field | Value |
|-------|-------|
| Statement | Payment webhook receivers must verify the HMAC-SHA256 signature of every inbound webhook payload. Payloads with invalid signatures must be rejected with HTTP 401 and logged as security events. |
| Threshold | Zero unverified webhook payloads processed; rejection rate of tampered payloads = 100% in security integration test |
| Method | Integration test: send webhook with valid signature → assert processed; send with invalid signature → assert HTTP 401 + security log entry; send with missing signature → assert HTTP 401 |
| Traces To | PRD Constraints C-02 (Razorpay reference), Section 9.4 |
| Notes | Razorpay HMAC verification: `HMAC-SHA256(razorpay_payment_id + '|' + razorpay_order_id, webhook_secret)`. The stub implementation must implement the same verification logic so the real integration is a config-only change. |

---

**NFR-REL-010**

| Field | Value |
|-------|-------|
| Statement | All monetary values in the codebase must use exact decimal types. Floating-point types (float, double, Number/Double in BSON) are prohibited for monetary calculations. |
| Threshold | Zero occurrences of `float` or `double` in money-handling code paths; enforced by static analysis rule in CI |
| Method | Semgrep rule that matches `float` or `double` type declarations in files under `*/service/payment/`, `*/service/booking/`, `*/service/disbursement/`, and similar; CI gate blocks on any match; custom rule in ESLint for TypeScript (`no-restricted-globals` / type checks); Pydantic validators for Python |
| Traces To | PRD Section 11 (Conventions — Data & Money), ADR-004 |
| Notes | Java: `java.math.BigDecimal`. TypeScript: `Decimal` from `decimal.js` (imported as a first-class type). Python: `decimal.Decimal`. MongoDB: `Decimal128` (never `Number`). JSON wire format: monetary amounts as string (`"1250.0000"`). Scale: 4 decimal places, HALF_UP rounding. See ADR-004. |

---

**NFR-REL-011**

| Field | Value |
|-------|-------|
| Statement | All fan-out cancellation sagas (event cancellation and Venue suspension cascade) must be idempotent: triggering the same cancellation twice must not produce duplicate refunds or duplicate state transitions. |
| Threshold | Zero duplicate refunds in any chaos test that replays cancellation commands; verified by refund amount reconciliation against RevenueSplit records |
| Method | Integration test: trigger event cancellation; re-trigger the same cancellation (simulate message replay); assert total refund amount equals original booking total exactly once; assert CANCELLED bookings are not transitioned again |
| Traces To | PRD G-06, FR-O-004 AC4, FR-A-002 AC3, FR-P-011 |
| Notes | Idempotency mechanism: Booking state machine rejects transitions to states already reached (`CANCELLED → CANCELLED` is a no-op, not an error). The compensation saga checks current state before executing each step. |

---

**NFR-REL-012**

| Field | Value |
|-------|-------|
| Statement | Revenue split amounts are immutable once created. No service may update a RevenueSplit record after creation. Refunds are applied proportionally by reading the original split amounts, never by recomputing them. |
| Threshold | Zero `UPDATE` statements on the `revenue_splits` table after initial insert; verified by database audit trigger; refund amount reconciliation tests confirm proportional application |
| Method | PostgreSQL row-level trigger that raises an exception on UPDATE or DELETE of `revenue_splits`; integration test that attempts to update a split and asserts the trigger fires; refund saga integration test that asserts refund amounts derive from stored split percentages |
| Traces To | PRD G-05, Section 7.1 (RevenueSplit), FR-O-006 AC4 |
| Notes | The `revenue_splits` table is an append-only ledger. Refund adjustments create a new negative `revenue_adjustments` record referencing the original split, rather than modifying the split record. This preserves a complete audit trail and enables reconstruction of the ledger state at any point in time. |

---

## 8. Availability Requirements — NFR-AVAIL

---

**NFR-AVAIL-001**

| Field | Value |
|-------|-------|
| Statement | Notification Service unavailability must never block or fail a booking saga. The booking saga must complete successfully and the Customer must receive a booking confirmation notification eventually when the Notification Service recovers. |
| Threshold | Booking saga success rate unaffected by Notification Service outage; notification delivery success rate ≥ 99% within 5 minutes of service recovery; zero bookings stuck in TICKETS_ISSUED state due to notification failure |
| Method | Chaos test: stop Notification Service container; complete 10 bookings; assert all bookings reach CONFIRMED state; restart Notification Service; assert all 10 notification events delivered within 5 minutes via Kafka backlog processing |
| Traces To | PRD FR-C-009 AC4, FR-P-008 (Notification in fail-open table) |
| Notes | Implementation: the Booking saga publishes notification commands to Kafka and moves to CONFIRMED state without awaiting notification delivery. The Notification Service is a pure consumer. Kafka ensures at-least-once delivery. Duplicate notifications are suppressed by idempotency keys. |

---

**NFR-AVAIL-002**

| Field | Value |
|-------|-------|
| Statement | When the Seat Inventory Service is unavailable, checkout must fail fast within 2 seconds and the Customer's seat selection state in the browser must be preserved for retry. |
| Threshold | Checkout failure response ≤ 2,000 ms when Seat Inventory is down; Customer browser retains selected seats after failure; no booking record created for the failed attempt |
| Method | Chaos test: stop Seat Inventory Service container; attempt checkout; assert 503 response within 2 s; assert no PENDING booking in DB; assert selected seats still shown in browser (Playwright test) |
| Traces To | PRD FR-P-007 (Seat Inventory in fail-closed table), FR-C-004 AC4 |
| Notes | The 2 s fast-fail is enforced by the circuit breaker (NFR-REL-006) combined with a 500 ms gRPC timeout on the Booking → Seat Inventory call. When the circuit is open, the Booking Service immediately returns a 503 without attempting the call. The UI preserves seat selection in component state (not server state) so that the Customer can retry without reselecting. |

---

**NFR-AVAIL-003**

| Field | Value |
|-------|-------|
| Statement | When the Payment Service is unavailable, checkout must fail fast with a 503 response, an idempotency key must be returned for safe retry, and the Customer's seat hold must be automatically extended by 5 minutes. |
| Threshold | 503 response ≤ 3,000 ms when Payment Service is down; seat hold TTL extended by exactly 5 minutes (not more, not less); idempotency key valid for retry for 24 hours |
| Method | Chaos test: stop Payment Service; initiate checkout (POST /checkout); assert 503 with idempotency key in response body within 3 s; inspect Redis hold TTL and assert it has been extended; restart Payment Service; retry with same idempotency key; assert checkout completes |
| Traces To | PRD FR-P-007 (Payment in fail-closed table), FR-C-004 AC4 |
| Notes | Hold extension is performed by the Booking Service as a compensating action when the payment initiation call fails. The Booking Service calls the Seat Inventory Service to extend the hold TTL (a separate gRPC operation from the hold creation). |

---

**NFR-AVAIL-004**

| Field | Value |
|-------|-------|
| Statement | When the Recommendation Service or Chatbot Service is unavailable, the event detail and homepage must render fully with a graceful fallback. No page must return an HTTP error or an empty state due to these services being down. |
| Threshold | Event detail page HTTP 200 and fully rendered with popularity-ranked recommendations (not personalised) when Recommendation Service is down; chat widget shows static fallback message when Chatbot Service is down; page Lighthouse performance score unaffected |
| Method | Chaos test: stop Recommendation Service; load event detail page; assert HTTP 200; assert "popular events" section rendered; assert no JavaScript error in console. Repeat for Chatbot Service. |
| Traces To | PRD FR-P-008 (both services in fail-open table), FR-AI-001 AC5, FR-C-008 AC7 |
| Notes | The API Gateway or BFF layer must have a timeout for recommendation calls (500 ms) and serve the fallback response on timeout, not on connection error only. The fallback event list is served by the Event Service (popularity-ranked query on MongoDB), not by the Recommendation Service. |

---

**NFR-AVAIL-005**

| Field | Value |
|-------|-------|
| Statement | When the Fraud Detection Service is unavailable, bookings must proceed as LOW-risk. Fraud detection failure must never block or delay a booking. When the service recovers, bookings from the outage window must be retrospectively scored. |
| Threshold | Zero bookings blocked due to Fraud Detection unavailability; retrospective scoring completes within 30 minutes of service recovery for all bookings placed during the outage |
| Method | Chaos test: stop Fraud Detection Service; complete 20 bookings; assert all 20 reach CONFIRMED state; restart service; assert all 20 bookings appear in `fraud.scored` topic within 30 minutes |
| Traces To | PRD FR-P-008 (Fraud Detection in fail-open table), FR-AI-007 AC4 |
| Notes | Retrospective scoring: when Fraud Detection recovers, it resets its Kafka consumer offset to the point before the outage and scores all missed events. High-risk findings trigger Admin alerts as normal. The Admin is informed that these scores are retrospective via a flag on the alert. |

---

**NFR-AVAIL-006**

| Field | Value |
|-------|-------|
| Statement | When the Check-in Service loses network connectivity, QR ticket verification must continue using cached per-event HMAC keys, and check-in records must sync to the service when connectivity is restored without data loss. |
| Threshold | Zero check-in failures due to network loss alone; offline check-in records synced within 30 seconds of connectivity restoration (NFR-PERF-038); duplicate scans detected and flagged during reconciliation |
| Method | Integration test: cache per-event key in Check-in Service; simulate network partition (iptables DROP); verify 20 QR scans; restore network; assert all 20 records synced to service and appear in database |
| Traces To | PRD G-09, FR-P-002 AC7, FR-P-008 (Check-in in fail-open table for network loss) |
| Notes | The per-event HMAC key is loaded into the Check-in Service process memory at service startup (or on first scan for that event) from the Ticket Service via gRPC. On network loss, the cached key is used. The local check-in record is stored in a local SQLite file (Check-in Service is a Fastify/Node service running on a gate terminal device or tablet). Sync is triggered on connectivity restoration via a background retry loop. |

---

## 9. Security Requirements — NFR-SEC

---

**NFR-SEC-001**

| Field | Value |
|-------|-------|
| Statement | All API endpoints (except explicitly public endpoints: `GET /events`, `GET /events/:id`, `GET /health/*`, `GET /auth/jwks`) must require a valid RS256-signed JWT in the `Authorization: Bearer` header. Requests without a valid token must be rejected with HTTP 401. |
| Threshold | 100% of non-public endpoints return 401 for missing/invalid JWT; zero authenticated operations proceeding without local JWT validation; JWKS endpoint returns the current public key set |
| Method | Security integration test suite: enumerate all registered routes; send unauthenticated request to each non-public endpoint; assert 401; send request with expired token; assert 401; send request with valid token; assert non-401 |
| Traces To | PRD G-10, FR-C-001, FR-O-001, FR-V-001, FR-A-001 |
| Notes | JWT validation is performed locally in each service using the public key from the JWKS endpoint (fetched at startup, cached, refreshed on key rotation). No per-request call to the Auth Service is required. This design means Auth Service availability is not a prerequisite for authenticated request processing. See ADR-002. |

---

**NFR-SEC-002**

| Field | Value |
|-------|-------|
| Statement | Access tokens must have a maximum TTL of 15 minutes. Refresh tokens must have a maximum TTL of 7 days and must be rotated on every use (refresh rotation). |
| Threshold | Access token `exp` claim ≤ current time + 900 seconds; refresh token `exp` claim ≤ current time + 604,800 seconds; a used refresh token must be invalidated and a new one issued; using a revoked refresh token returns 401 |
| Method | Unit test: decode generated access token and assert `exp` - `iat` ≤ 900; integration test: use a refresh token; use the same refresh token again; assert second use returns 401 |
| Traces To | PRD FR-C-001 AC4 |
| Notes | Refresh token rotation defends against refresh token theft: if an attacker steals a refresh token and uses it after the legitimate user has already refreshed, the second use (by either party after the first rotation) fails. The Auth Service must detect this race and optionally invalidate the entire session. Refresh tokens are stored server-side (Redis) as a security measure; the client sends the token but the server validates against the stored copy. |

---

**NFR-SEC-003**

| Field | Value |
|-------|-------|
| Statement | Role-Based Access Control (RBAC) must be enforced at the service layer, not only at the API Gateway or UI. A Customer JWT must not be accepted by Organiser-only or Admin-only endpoints, even when bypassing the Gateway. |
| Threshold | 100% of role-restricted endpoints enforce the role check independently of the Gateway; a Customer JWT calling an Organiser endpoint returns 403; tested with direct service-to-service calls bypassing the Gateway |
| Method | Security integration test: call Organiser-only endpoint directly (bypassing Gateway) with a Customer JWT; assert 403; repeat for each role restriction across all services |
| Traces To | PRD Section 6 (Actors and Roles, disjoint permissions), G-10 |
| Notes | Implementation: each service extracts the `role` claim from the JWT and evaluates it against the required role annotation for the endpoint. The Gateway may perform a preliminary role check for performance, but this is not the authoritative check. Spring Security 6 `@PreAuthorize` annotations or NestJS Guards or FastAPI dependencies enforce this at the handler level. |

---

**NFR-SEC-004**

| Field | Value |
|-------|-------|
| Statement | Multi-tenant data isolation must be enforced at the service layer. An Organiser must not be able to read or write another Organiser's data. A Venue must not be able to read another Venue's data. Cross-tenant access must return HTTP 403 and create a structured security log entry. |
| Threshold | Zero cross-tenant data access in any authenticated API call; every cross-tenant attempt returns 403 within 100 ms; security log entry present for every 403 |
| Method | Security integration test: create two Organiser accounts; Organiser A attempts to access Organiser B's event details, revenue data, and booking data; assert 403 in each case; assert security log entry created |
| Traces To | PRD Section 6.2 (Organiser constraints), Section 6.3 (Venue constraints) |
| Notes | Tenant isolation is implemented by extracting `organiserId` or `venueId` from the JWT `sub` or custom claims, and including it in every database query as a mandatory filter. An ORM-level global filter (Hibernate filter, Prisma middleware, SQLAlchemy event listener) ensures this filter cannot be omitted by developer error. |

---

**NFR-SEC-005**

| Field | Value |
|-------|-------|
| Statement | QR ticket tokens must be HMAC-SHA256 signed with a per-event secret. A token issued for one event must not be accepted at the check-in endpoint for a different event. |
| Threshold | 100% rejection rate for cross-event token reuse; WRONG_EVENT response with security log entry for any cross-event scan |
| Method | Security integration test: issue a ticket for Event A; present its QR token at Event B's check-in endpoint; assert WRONG_EVENT response; assert security log entry |
| Traces To | PRD FR-P-002 AC6, FR-C-005 AC4, G-09 |
| Notes | The signed payload includes `eventId`. The Check-in Service validates that the `eventId` in the token matches the event being checked into. The per-event secret is derived from a platform-level secret and the `eventId` (HKDF-derived), so a token cannot be forged for a different event even with knowledge of one event's secret. |

---

**NFR-SEC-006**

| Field | Value |
|-------|-------|
| Statement | Scanning the same QR token twice at check-in must return a success response (ALREADY_CHECKED_IN) without creating a duplicate check-in record. |
| Threshold | Zero duplicate check-in records for any ticket; second scan response: HTTP 200 with `status: ALREADY_CHECKED_IN`; measured under concurrent double-scan scenarios |
| Method | Concurrency integration test: send two simultaneous scan requests for the same token; assert exactly one check-in record created in DB; assert both responses return HTTP 200 (one VALID, one ALREADY_CHECKED_IN, or both ALREADY_CHECKED_IN if race is tight) |
| Traces To | PRD FR-P-002 AC4 |
| Notes | Implemented via a database unique constraint on `(event_id, ticket_id)` in the check-in log, combined with an `INSERT ... ON CONFLICT DO NOTHING` pattern. The constraint is the correctness guarantee; optimistic concurrency handles the race. |

---

**NFR-SEC-007**

| Field | Value |
|-------|-------|
| Statement | Customer passwords must be hashed with bcrypt at cost ≥ 12 or argon2id. Plaintext passwords must never be logged, stored, or transmitted after initial receipt. |
| Threshold | All stored password hashes verifiable as bcrypt (cost ≥ 12) or argon2id via hash format inspection; zero occurrences of plaintext password in any log output or database column |
| Method | Unit test: hash a password; decode the hash header and assert algorithm and cost; integration test: register a user; inspect the auth database and assert the password column matches bcrypt/argon2id format; grep logs for common password-like patterns (SAST rule) |
| Traces To | PRD FR-C-001 AC3 |
| Notes | argon2id is preferred for new implementations (more memory-hard, resistant to GPU attacks). bcrypt at cost 12 is acceptable. Cost must be configurable via environment variable so it can be increased as hardware improves without code change. |

---

**NFR-SEC-008**

| Field | Value |
|-------|-------|
| Statement | All secrets (database passwords, API keys, JWT signing keys, HMAC secrets, Vault tokens) must be managed in HashiCorp Vault. No secret may appear in source code, environment files committed to version control, or Docker images. |
| Threshold | Zero secrets detected by gitleaks or trufflehog in any commit across all repositories; all service startup logs show secrets retrieved from Vault (not from environment literals) |
| Method | gitleaks pre-commit hook on every repository; trufflehog scan in CI pipeline; CI gate blocks on any secret detection; Vault audit log confirms services fetch secrets at startup |
| Traces To | PRD G-10, Section 11 (Conventions — Security) |
| Notes | Services use Vault Agent Sidecar or the Vault SDK to fetch secrets at startup. Secrets are injected as environment variables or mounted files by the Vault Agent, not hardcoded. In the local Docker Compose environment, Vault is run in dev mode (not recommended for production); secret paths and Vault Agent configuration must be identical to production paths. |

---

**NFR-SEC-009**

| Field | Value |
|-------|-------|
| Statement | Rate limiting must be enforced at the API Gateway: 100 requests/minute for unauthenticated requests, 1,000 requests/minute for authenticated requests, per IP address. |
| Threshold | Requests exceeding the rate limit receive HTTP 429 with `Retry-After` header; rate limit counters persist in Redis and survive Gateway instance restarts; zero requests above the limit processed within the enforcement window |
| Method | k6 test: send 150 unauthenticated requests in 60 seconds from a single IP; assert at least 50 return 429; assert `Retry-After` header present; assert Redis rate limit counter matches expected value |
| Traces To | PRD G-10, FR-P-003 (flash sale protection) |
| Notes | Rate limiting is implemented in Spring Cloud Gateway using Redis-backed token bucket (bucket4j or Spring Cloud Gateway RateLimiter). Per-IP limits use `X-Forwarded-For` header (with trust verification to prevent IP spoofing). The flash sale queue (NFR-PERF-013) operates above the rate limiter — authenticated requests in the queue are counted against the 1,000/min limit. |

---

**NFR-SEC-010**

| Field | Value |
|-------|-------|
| Statement | Static Application Security Testing (SAST) must run on every pull request. Any HIGH-severity finding in SonarQube or Semgrep must block the CI pipeline from passing. |
| Threshold | Zero HIGH-severity SAST findings on the main branch of any service repository; CI pipeline blocks on any new HIGH finding introduced in a PR |
| Method | SonarQube Community + Semgrep configured in every service CI pipeline (`.github/workflows/ci.yml`); SonarQube Quality Gate set to "Fail on new blocker/critical issues"; Semgrep ruleset includes OWASP Top 10 rules; CI step fails if either tool reports HIGH+ findings |
| Traces To | PRD G-10, Success Metrics Section 11.3 |

---

**NFR-SEC-011**

| Field | Value |
|-------|-------|
| Statement | Software Composition Analysis (SCA) must run on every build. Any CRITICAL CVE in a direct or transitive dependency must block the CI pipeline. |
| Threshold | Zero CRITICAL CVEs in direct dependencies on main; CI blocks on CRITICAL CVE detection; Dependabot PRs for HIGH/CRITICAL CVEs auto-created within 24 hours of CVE publication |
| Method | OWASP Dependency-Check + Trivy in CI pipeline; Trivy `--exit-code 1 --severity CRITICAL` flag; Dependabot configured for all repositories with daily security update checks |
| Traces To | PRD G-10, Success Metrics Section 11.3 |

---

**NFR-SEC-012**

| Field | Value |
|-------|-------|
| Statement | Every Docker image produced by CI must be scanned for CRITICAL CVEs before being pushed to the registry. Images with CRITICAL CVEs must not be pushed. |
| Threshold | Zero CRITICAL CVEs in any Docker image on the container registry; Trivy image scan step precedes the push step in every CI pipeline |
| Method | Trivy image scan (`trivy image --exit-code 1 --severity CRITICAL <image>`) as a CI step after the Docker build step and before the push step |
| Traces To | PRD G-10 |

---

**NFR-SEC-013**

| Field | Value |
|-------|-------|
| Statement | Secrets scanning must run on every commit. Any detected secret must block the push. |
| Threshold | Zero secrets committed to any repository; gitleaks pre-commit hook blocks commit on detection; trufflehog CI scan blocks pipeline on detection |
| Method | gitleaks configured as a pre-commit hook (`pre-commit` framework) across all repositories; trufflehog `--fail` flag in CI pipeline; gitleaks and trufflehog baseline files maintained to suppress known false positives |
| Traces To | PRD G-10, NFR-SEC-008 |

---

## 10. Observability Requirements — NFR-OBS

---

**NFR-OBS-001**

| Field | Value |
|-------|-------|
| Statement | All services must emit structured JSON log events. Every log event must include the mandatory correlation fields: `traceId`, `spanId`, `service`, `timestamp` (UTC ISO-8601), `level`, and `message`. Transaction-scoped logs must also include `bookingId`, `eventId`, and `userId` where applicable. |
| Threshold | 100% of log events parseable as JSON; 100% of log events include mandatory fields; zero unstructured log output in any service; verified by log schema validation in CI |
| Method | Log schema validation test: run service; perform a representative operation; capture stdout; parse each line as JSON; assert mandatory fields present; assert no lines fail JSON parse. Loki log volume check confirms logs are shipped. |
| Traces To | PRD G-10, Section 10 (OBS-01), FR-A-001 AC4 (Admin action logging) |
| Notes | Java: Logback with `logstash-logback-encoder`. Node: Pino (structured by default). Python: structlog or standard logging with JSON formatter. The `traceId` and `spanId` values are injected by the OpenTelemetry SDK (NFR-OBS-003) into the logging context via MDC (Java), AsyncLocalStorage (Node), or structlog context (Python). |

---

**NFR-OBS-002**

| Field | Value |
|-------|-------|
| Statement | All services must expose a `/metrics` endpoint in Prometheus text format. RED metrics (Rate, Errors, Duration) must be captured per endpoint. USE metrics (Utilisation, Saturation, Errors) must be captured per external resource (DB connections, Redis connections, Kafka consumer lag). Business metrics must be exposed by the services that own the relevant data. |
| Threshold | `/metrics` endpoint returns HTTP 200 with `Content-Type: text/plain; version=0.0.4` on every service; Prometheus scrape succeeds for all targets; no scrape target in `down` state for > 2 minutes without an alert |
| Method | Prometheus target health check in CI (smoke test against running service); Grafana dashboard loads without "No Data" panels; alert `PrometheusTargetDown` configured and tested |
| Traces To | PRD G-10, Section 10 (OBS-02), Success Metrics Section 11.5 |
| Notes | Java: Micrometer with Prometheus registry (auto-configured in Spring Boot Actuator). Node: prom-client. Python: prometheus-client. Business metrics (bookings/min, GMV/min, active seat holds, flash sale queue depth, waitlist depth, DLQ depth, fraud holds): each metric owned by the service that writes the relevant data, exposed on `/metrics` and visualised in Grafana. |

---

**NFR-OBS-003**

| Field | Value |
|-------|-------|
| Statement | All services must emit OpenTelemetry traces. Trace context must be propagated across HTTP headers (`traceparent`), gRPC metadata, and Kafka message headers. |
| Threshold | Every inbound request creates a root span; every outbound call creates a child span; Kafka consumer handlers create spans linked to producer spans; Jaeger shows a complete trace across all services for a booking saga |
| Method | Integration test: initiate a booking; retrieve trace from Jaeger by `bookingId`; assert trace spans cover: API Gateway, Booking Service, Seat Inventory Service (gRPC), Kafka publish, Ticket Service (Kafka consume), Notification Service (Kafka consume); assert no missing links in the trace |
| Traces To | PRD G-10, Section 10 (OBS-03) |
| Notes | Java: OpenTelemetry Java agent (auto-instrumentation) + manual spans for Kafka and business operations. Node: `@opentelemetry/sdk-node` auto-instrumentation + `kafkajs` instrumentation. Python: `opentelemetry-sdk` + `opentelemetry-instrumentation-fastapi`. Kafka trace propagation: inject `traceparent` header in Kafka message headers on produce; extract on consume. |

---

**NFR-OBS-004**

| Field | Value |
|-------|-------|
| Statement | Given any `bookingId`, all logs and spans across all services involved in that booking must be locatable in Grafana (Loki) and Jaeger within 2 minutes of the query being initiated. |
| Threshold | End-to-end trace retrieval time ≤ 2 minutes from bookingId to complete trace in Jaeger; Loki query `{bookingId="<id>"}` returns results within 30 seconds |
| Method | Manual verification test documented in runbook; automated test: complete a booking; wait 60 s for log/trace ingestion; query Grafana Loki with `{bookingId="<id>"}`; assert results returned; query Jaeger with trace search by `bookingId` tag; assert complete trace found |
| Traces To | PRD G-10, Section 10 (OBS-04), Success Metrics Section 11.5 |
| Notes | The `bookingId` must be included in every log event and OpenTelemetry span attribute throughout the saga, including in the Kafka message headers, so that Loki's LogQL and Jaeger's tag search both work. This requires explicit propagation — neither framework does this automatically for custom business IDs. |

---

**NFR-OBS-005**

| Field | Value |
|-------|-------|
| Statement | Business metrics in Grafana must be refreshed at least every 60 seconds. The required business metrics are: bookings/minute, GMV/minute, active seat holds, flash sale queue depth, waitlist depth for top-10 events, fraud holds active, DLQ depth per topic. |
| Threshold | All business metrics panels in the Grafana dashboard show data ≤ 60 s old at any point; no business metric panel shows "No Data" during normal operation |
| Method | Grafana dashboard validation: load dashboard; assert no "No Data" panels; check panel refresh intervals; verify Prometheus recording rules are pre-aggregating business metrics for performance |
| Traces To | PRD Section 10 (OBS-05), FR-A-006 AC3 |
| Notes | Business metrics are exposed on `/metrics` by the owning service and scraped by Prometheus. High-cardinality metrics (per-event waitlist depth for 10,000 events) must be pre-aggregated using Prometheus recording rules or ClickHouse queries, not computed at query time in Grafana. |

---

## 11. Maintainability Requirements — NFR-MAINT

---

**NFR-MAINT-001**

| Field | Value |
|-------|-------|
| Statement | Unit test branch coverage for every service must be at or above 80%. The CI pipeline must block if coverage drops below this threshold. |
| Threshold | Branch coverage ≥ 80% per service; CI blocks if coverage regresses below 80% on any PR; measured per-service, not as a platform aggregate |
| Method | Java: JaCoCo in Maven/Gradle with `<limit><counter>BRANCH</counter><value>COVEREDRATIO</value><minimum>0.80</minimum></limit>` in CI; Node: Istanbul/c8 with `--branches 80` flag; Python: pytest-cov with `--cov-fail-under=80` |
| Traces To | PRD G-10, Success Metrics Section 11.4 |
| Notes | 80% branch coverage is a floor, not a target. Critical paths (seat hold, payment saga, cancellation compensation) must have 100% branch coverage. The 80% floor prevents coverage debt accumulating on new code. |

---

**NFR-MAINT-002**

| Field | Value |
|-------|-------|
| Statement | Every service must have Testcontainers integration tests for all external dependencies (databases, Kafka, Redis) that the service directly communicates with. |
| Threshold | At least one Testcontainers integration test per external dependency per service; tests must run in CI without requiring a pre-running infrastructure stack; all integration tests pass on a clean CI runner |
| Method | CI pipeline verification: integration tests run using `@Testcontainers` (Java), `testcontainers` npm package (Node), or `testcontainers` Python library; no test skipped with "requires real infrastructure" justification |
| Traces To | PRD G-10 |
| Notes | Testcontainers spin up real containers (PostgreSQL, Redis, Kafka, MongoDB) for each test run and tear them down after. This provides high-fidelity integration tests without requiring a pre-running stack. The startup overhead (~10–30 s per container) must be accounted for in CI timeout configuration. |

---

**NFR-MAINT-003**

| Field | Value |
|-------|-------|
| Statement | Every service must have a contract-first OpenAPI (for REST services) or AsyncAPI (for event-driven services) specification. The specification must be validated against the actual service implementation in CI. |
| Threshold | OpenAPI spec exists in `stagepass-docs/docs/api/<service>.yaml` before service implementation begins; every API endpoint in the spec is implemented; every implemented endpoint is in the spec; CI fails if spec and implementation diverge |
| Method | Dredd or Schemathesis contract test in CI: generate requests from the OpenAPI spec; send them to a running service (Testcontainers); assert responses match the spec schemas; AsyncAPI spec validated against Kafka message schemas using `asyncapi validate` |
| Traces To | PRD Section 11 (Conventions — API Design) |

---

**NFR-MAINT-004**

| Field | Value |
|-------|-------|
| Statement | A developer with access to the repository must be able to run any individual service locally from a clean clone within 30 minutes, following only the service's README. |
| Threshold | Time from `git clone` to `curl /health/ready` = 200 OK ≤ 30 minutes; measured by a new developer who has never seen the codebase |
| Method | Time measurement test: fresh environment (new Docker context, no cached layers); follow README steps exactly; measure total elapsed time; tested at least once per phase by a simulated clean setup |
| Traces To | PRD G-10, Constraints C-01 (solo developer onboarding), Success Metrics Section 11.4 |
| Notes | The 30-minute budget includes: Docker image pulls, dependency installation, database migration, and seed data loading. It does not include the initial Docker installation or initial JDK installation (these are prerequisites stated in the README). |

---

**NFR-MAINT-005**

| Field | Value |
|-------|-------|
| Statement | An Architecture Decision Record (ADR) must be written and merged before implementation begins for every non-trivial architectural decision. No implementation PR may be merged if it introduces a new architectural pattern or technology without a linked, merged ADR. |
| Threshold | 100% of non-trivial decisions have a preceding ADR; CI PR template checklist includes an ADR check; ADR review is part of the PR review process |
| Method | PR template checklist item: "ADR written if architectural decision was made"; code review gate: reviewer must verify ADR exists if the PR introduces a new pattern; monthly audit of merged PRs against ADR list |
| Traces To | PRD Success Metrics Section 11.4 |
| Notes | "Non-trivial" is defined as any decision that: (a) introduces a new technology into the stack, (b) changes the communication pattern between services, (c) changes the data model of a T1 service, or (d) affects the booking or revenue split computation logic. Trivial decisions (adding a new field to an existing response, changing a log level) do not require an ADR. |

---

## 12. Compliance Requirements — NFR-COMP

These requirements derive from the Indian regulatory environment described in
PRD Section 5.1 and Section 9. They are non-negotiable constraints.

---

**NFR-COMP-001**

| Field | Value |
|-------|-------|
| Statement | All ticket prices displayed to Customers must be GST-inclusive. The GST amount must be separately itemised on tax invoices generated per booking. |
| Threshold | Zero occurrences of GST-exclusive price display in the Customer storefront; every booking generates a tax invoice with separate `base_price`, `gst_rate`, `gst_amount`, and `total_price` fields; GSTIN displayed for Organiser (if provided) and platform |
| Method | UI test (Playwright): assert that the price shown on the event detail page and checkout page matches the total payable (including GST); assert that the generated invoice PDF contains all required fields; schema validation of tax invoice against GST invoice specification |
| Traces To | PRD Section 9.1 (GST requirements), G-05 |

---

**NFR-COMP-002**

| Field | Value |
|-------|-------|
| Statement | Customers must provide explicit, informed consent before any personal data is collected. The consent notice must state what data is collected, why, and for how long it is retained. Consent is recorded and auditable. |
| Threshold | Consent record created at registration with: timestamp, consent text hash (SHA-256 of the displayed text), Customer ID; zero PII collected before consent record exists; consent can be withdrawn and is processed within 72 hours |
| Method | Integration test: register a Customer; assert consent record exists in the database with all required fields; integration test: simulate consent withdrawal; assert PII fields anonymised within 72 hours (can be tested with a shorter SLA in dev with configurable delay) |
| Traces To | PRD Section 9.2 (DPDP Act requirements) |

---

**NFR-COMP-003**

| Field | Value |
|-------|-------|
| Statement | Customers must be able to export their personal data within 72 hours of request. Customer-initiated account deletion must anonymise PII while retaining financial records for GST audit purposes (minimum 7 years). |
| Threshold | Data export completed ≤ 72 hours after request; PII fields (name, email, phone, address) anonymised within 24 hours of account deletion; `booking_id`, `amount`, `gst_amount`, `created_at` retained in anonymised form for 7 years |
| Method | Integration test: delete a test Customer account; assert PII columns nulled or replaced with anonymised values within 24 hours; assert booking financial records still exist with anonymised customer reference |
| Traces To | PRD Section 9.2 (DPDP Act — right to erasure and access) |

---

**NFR-COMP-004**

| Field | Value |
|-------|-------|
| Statement | For platform-cancelled events, refunds must be initiated within 14 days of the cancellation. |
| Threshold | `refund_initiated_at` timestamp ≤ `cancellation_at` + 14 days for all bookings in a platform-cancelled event; refund saga initiated immediately on event cancellation (not delayed) |
| Method | Integration test: cancel an event; assert all booking refund sagas in REFUND_INITIATED state within 60 minutes; audit query: verify no refund_initiated_at timestamp exceeds cancellation_at + 14 days in production |
| Traces To | PRD Section 9.3 (Consumer Protection Rules), G-06 |

---

**NFR-COMP-005**

| Field | Value |
|-------|-------|
| Statement | The cancellation policy must be clearly displayed to Customers before purchase, at the point of checkout, not only in a separate terms page. |
| Threshold | Cancellation policy summary (refund brackets) is visible on the checkout page before payment initiation; UI test asserts the policy element is present and not hidden |
| Method | Playwright UI test: load checkout page for an event with a non-trivial cancellation policy; assert the policy summary element is visible in the viewport without scrolling |
| Traces To | PRD Section 9.3 (Consumer Protection Rules), FR-C-004 AC1 |

---

## 13. Cross-Reference Index

This index maps every NFR to the PRD requirement(s) it traces to, and to any
ADR that encodes the architectural decision implementing it.

| NFR ID | PRD Anchor | ADR |
|--------|-----------|-----|
| NFR-PERF-001 | G-03, G-04, FR-C-003, FR-P-001 | ADR-006, ADR-007 |
| NFR-PERF-002 | G-03, FR-C-004 | ADR-005 |
| NFR-PERF-003 | G-03, G-04, FR-C-004 | ADR-005 |
| NFR-PERF-004 | G-09, FR-P-002 | ADR-003 |
| NFR-PERF-005 | G-08, FR-P-001, FR-C-003 | ADR-003 |
| NFR-PERF-006 | G-07, FR-C-002 | ADR-002 |
| NFR-PERF-007 | G-07, FR-C-002, FR-AI-002 | ADR-002 |
| NFR-PERF-008 | G-07, FR-C-002, FR-AI-003 | ADR-002 |
| NFR-PERF-009 | G-07, FR-C-008, FR-AI-004 | ADR-002 |
| NFR-PERF-010 | G-02, G-07, FR-C-002, FR-AI-002 | ADR-003 |
| NFR-PERF-011 | G-07, FR-AI-007 | ADR-003 |
| NFR-PERF-012 | G-07, FR-C-002, FR-AI-001 | ADR-002 |
| NFR-PERF-013 | G-03, FR-P-003 | ADR-006, ADR-007 |
| NFR-PERF-014 | G-03, FR-C-006 | — |
| NFR-PERF-015 | G-08, FR-P-001, FR-C-009 | ADR-003 |
| NFR-PERF-016 | G-03, FR-C-005 | ADR-002 |
| NFR-PERF-017 | G-08, FR-C-003, FR-V-002 | ADR-002 |
| NFR-PERF-018 | FR-AI-005, FR-O-005 | ADR-002 |
| NFR-PERF-019 | FR-AI-008 | ADR-002 |
| NFR-PERF-020 | G-06, FR-O-004, FR-P-011 | ADR-005 |
| NFR-PERF-021 | FR-AI-006 | ADR-006 |
| NFR-PERF-022 | FR-O-005 | ADR-002 |
| NFR-PERF-023 | FR-A-006 | ADR-002 |
| NFR-PERF-024 | G-05, Section 7.3 | ADR-008 |
| NFR-PERF-025 | G-03, FR-C-006 | ADR-005 |
| NFR-PERF-026 | FR-O-001, FR-A-001 | — |
| NFR-PERF-027 | FR-AI-002, FR-AI-003 | ADR-002 |
| NFR-PERF-028 | FR-P-003 | ADR-007 |
| NFR-PERF-029 | G-06, FR-C-007 | ADR-005 |
| NFR-PERF-030 | G-03, Constraints C-01 | ADR-007 |
| NFR-PERF-031 | G-08, FR-P-001 | ADR-003 |
| NFR-PERF-032 | G-03, G-08, Capacity Model | ADR-003 |
| NFR-PERF-033 | FR-P-005, ADR-003 | ADR-003, ADR-006 |
| NFR-PERF-034 | FR-P-002, ADR-003 | ADR-003 |
| NFR-PERF-035 | FR-C-003 | ADR-002 |
| NFR-PERF-036 | FR-C-005 | ADR-002 |
| NFR-PERF-037 | FR-O-003 | ADR-002 |
| NFR-PERF-038 | FR-P-002, NFR-AVAIL-006 | ADR-003 |
| NFR-PERF-039 | G-05, FR-A-005 | ADR-008 |
| NFR-PERF-040 | FR-A-004 | ADR-002 |
| NFR-PERF-041 | FR-A-003 | — |
| NFR-PERF-042 | G-10, G-08 | ADR-002 |
| NFR-PERF-043 | Constraints C-01 | — |
| NFR-REL-001 | FR-P-004 | ADR-005 |
| NFR-REL-002 | FR-P-004 | ADR-005 |
| NFR-REL-003 | G-06, FR-P-005, FR-C-004 | ADR-005 |
| NFR-REL-004 | G-04, FR-C-003, Section 7.1 | ADR-006 |
| NFR-REL-005 | FR-P-005 | ADR-005 |
| NFR-REL-006 | FR-P-009, FR-P-007, FR-P-008 | ADR-003 |
| NFR-REL-007 | FR-P-005, G-10 | ADR-003 |
| NFR-REL-008 | G-03, G-04, FR-C-003 | ADR-006 |
| NFR-REL-009 | Constraints C-02, Section 9.4 | ADR-008 |
| NFR-REL-010 | Section 11 Conventions, ADR-004 | ADR-004 |
| NFR-REL-011 | G-06, FR-O-004, FR-A-002, FR-P-011 | ADR-005 |
| NFR-REL-012 | G-05, Section 7.1, FR-O-006 | ADR-008 |
| NFR-AVAIL-001 | FR-C-009, FR-P-008 | ADR-005 |
| NFR-AVAIL-002 | FR-P-007, FR-C-004 | ADR-006 |
| NFR-AVAIL-003 | FR-P-007, FR-C-004 | ADR-005 |
| NFR-AVAIL-004 | FR-P-008, FR-AI-001, FR-C-008 | ADR-002 |
| NFR-AVAIL-005 | FR-P-008, FR-AI-007 | ADR-002 |
| NFR-AVAIL-006 | G-09, FR-P-002, FR-P-008 | ADR-003 |
| NFR-SEC-001 | G-10, FR-C-001, FR-O-001 | ADR-002 |
| NFR-SEC-002 | FR-C-001 | ADR-002 |
| NFR-SEC-003 | Section 6, G-10 | ADR-002 |
| NFR-SEC-004 | Section 6.2, Section 6.3 | ADR-002 |
| NFR-SEC-005 | FR-P-002, FR-C-005, G-09 | ADR-003 |
| NFR-SEC-006 | FR-P-002 | ADR-003 |
| NFR-SEC-007 | FR-C-001 | ADR-002 |
| NFR-SEC-008 | G-10, Section 11 Conventions | — |
| NFR-SEC-009 | G-10, FR-P-003 | ADR-003 |
| NFR-SEC-010 | G-10, Success Metrics 11.3 | — |
| NFR-SEC-011 | G-10, Success Metrics 11.3 | — |
| NFR-SEC-012 | G-10 | — |
| NFR-SEC-013 | G-10, NFR-SEC-008 | — |
| NFR-OBS-001 | G-10, Section 10 OBS-01, FR-A-001 | — |
| NFR-OBS-002 | G-10, Section 10 OBS-02, Success 11.5 | — |
| NFR-OBS-003 | G-10, Section 10 OBS-03 | ADR-003 |
| NFR-OBS-004 | G-10, Section 10 OBS-04, Success 11.5 | — |
| NFR-OBS-005 | Section 10 OBS-05, FR-A-006 | — |
| NFR-MAINT-001 | G-10, Success 11.4 | — |
| NFR-MAINT-002 | G-10 | — |
| NFR-MAINT-003 | Section 11 Conventions | — |
| NFR-MAINT-004 | G-10, Constraints C-01, Success 11.4 | — |
| NFR-MAINT-005 | Success 11.4 | — |
| NFR-COMP-001 | Section 9.1, G-05 | ADR-008 |
| NFR-COMP-002 | Section 9.2 | — |
| NFR-COMP-003 | Section 9.2 | — |
| NFR-COMP-004 | Section 9.3, G-06 | ADR-005 |
| NFR-COMP-005 | Section 9.3, FR-C-004 | — |
