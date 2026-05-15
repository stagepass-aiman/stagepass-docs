# ADR-003 — Service Communication Patterns

| Field        | Value                                                              |
|--------------|--------------------------------------------------------------------|
| **ID**       | ADR-003                                                            |
| **Title**    | Service Communication Patterns                                     |
| **Status**   | Accepted                                                           |
| **Date**     | 2025-01-15                                                         |
| **Author**   | StagePass Architecture                                             |
| **Version**  | 1.0.0                                                              |
| **Traces To**| PRD §3 (Tech Stack), NFR-REL-001, NFR-REL-002, NFR-REL-006,       |
|              | NFR-REL-007, NFR-PERF-001, NFR-PERF-004, NFR-PERF-005,           |
|              | NFR-OBS-003, ADR-001 (Repo Topology), ADR-002 (Tech Stack)        |
| **Supersedes**| —                                                                 |
| **Superseded By**| —                                                             |

---

## Change Log

| Version | Date       | Author              | Summary                     |
|---------|------------|---------------------|-----------------------------|
| 1.0.0   | 2025-01-15 | StagePass Architecture | Initial acceptance       |

---

## 1. Status

**Accepted.** This ADR is in effect. All service implementations must conform to the
communication patterns documented here. Any deviation requires a new ADR or an amendment
to this document, reviewed and merged before implementation begins.

---

## 2. Context

### 2.1 Why this decision exists

StagePass is a distributed system with 17 services plus an API Gateway — a polyglot
platform spanning Java, TypeScript, and Python. Every inter-service call is a network
hop with failure modes that do not exist in a monolith: partial failures, network
partitions, latency outliers, and semantic mismatches between caller and callee.

Before writing a single line of implementation code, the communication contract must
be locked. Leaving each service team (in this case, the same developer wearing different
hats) to decide independently how to communicate with neighbours produces:

- Inconsistent retry logic that either storms a recovering service or gives up too
  early to succeed
- Missing dead-letter handling, so message loss is discovered in production
- Trace context not propagated across a transport boundary, making a saga invisible in
  Jaeger
- WebSocket rooms that don't scale past a single process instance

This ADR eliminates all of those by specifying the protocol, library, configuration,
and observability contract for every inter-service communication channel on the platform.

### 2.2 Scope

This ADR decides:

- Which synchronous protocol is used for which service-to-service call (REST vs. gRPC)
- The exact two gRPC pairs, their `.proto` file locations in `stagepass-shared-contracts`,
  and their client library per language runtime
- Every Kafka topic name, partition key, consumer group strategy, idempotency requirement,
  and DLQ name
- WebSocket room topology, scaling strategy, and event routing
- Retry and circuit breaker policy applied uniformly to all synchronous calls
- OpenTelemetry propagation mechanism per transport type

### 2.3 Scope exclusions

This ADR does not decide:

- Language or framework per service (→ ADR-002)
- Repository topology (→ ADR-001)
- The internal booking saga step sequence (→ ADR-005)
- The seat inventory concurrency mechanism (→ ADR-006)
- Flash sale queue internals (→ ADR-007)
- Money type representation (→ ADR-004)
- Revenue disbursement model (→ ADR-008)

### 2.4 Forces at play

**Performance constraints are tight and per-path.** The seat hold must complete in
p99 < 500 ms (NFR-PERF-001). The check-in verification must complete in p99 < 200 ms
(NFR-PERF-004). These are not averages; they are 99th-percentile hard limits that
dictate whether a synchronous call can sit in the path at all.

**Non-fungible inventory creates hard ordering requirements.** Unlike a product inventory
decrement, every seat hold is a contention event. The communication pattern must not
introduce ordering ambiguity. A Kafka-based seat hold where two consumers race on the
same message would be catastrophic. This shapes the synchronous-first design for
the seat hold path.

**Event-driven flows must never lose messages.** A `booking.confirmed` event that
disappears before the Ticket Service processes it means a Customer has paid but has
no ticket. At-least-once delivery with idempotent consumers is the only acceptable
guarantee. DLQ handling is not optional.

**The system must be horizontally scalable.** The Notification Service runs multiple
instances in production. WebSocket rooms are process-local without a shared state
layer — a Customer connected to instance A cannot receive a message emitted by
instance B. The communication architecture must eliminate this problem by design,
not by accident.

**Observability must be end-to-end.** A booking saga spans: API Gateway → Booking
Service → Seat Inventory (gRPC) → Payment (REST) → Kafka → multiple consumers →
WebSocket. If trace context is not propagated across every transport boundary, the
trace breaks at the first Kafka hop and the saga becomes unobservable. This is not
an observability feature — it is a correctness requirement (NFR-OBS-003).

---

## 3. Decision

### 3.1 Summary of communication channels

| Channel       | Used for                                                  | Protocol      |
|---------------|-----------------------------------------------------------|---------------|
| REST (HTTP/1.1) | Gateway → all services; Booking → Payment initiation    | REST + JWT    |
| gRPC (HTTP/2) | Booking → Seat Inventory; Check-in → Ticket              | Protobuf/gRPC |
| Kafka         | All async: saga events, domain events, fan-out, AI input | Kafka (KafkaJS / spring-kafka / aiokafka) |
| WebSocket     | Server-to-browser push only, via Notification Service    | Socket.IO 4   |

---

### 3.2 Synchronous communication — REST

#### 3.2.1 When REST is used

REST is the default synchronous protocol for all calls except the two explicitly assigned
to gRPC (§3.3). It is used for:

- **API Gateway → all downstream services.** The Gateway is the single ingress point.
  Every inbound HTTP request from the browser or mobile app passes through the Gateway,
  which validates the JWT, forwards identity headers (`X-User-Id`, `X-User-Role`,
  `X-Correlation-Id`), and proxies to the appropriate service.

- **Booking Service → Payment Service (payment initiation).** The Booking saga initiates
  payment by making a synchronous REST call to the Payment Service to create a payment
  order. The call returns a payment order ID immediately; the actual payment result
  arrives later via a webhook (inbound Razorpay callback) and then Kafka. This keeps
  the synchronous path fast while payment processing is fully async.

- **Chatbot Service → Event Service / Booking Service (RAG context retrieval).** The
  Chatbot Service retrieves event detail and booking context for grounding LLM responses.
  It forwards the Customer's JWT, so Event and Booking Services enforce RBAC as normal.
  These are low-frequency, read-only calls. REST is appropriate.

#### 3.2.2 REST conventions

All REST calls between services follow these conventions:

**Error format:** Problem Details (RFC 9457). Every non-2xx response includes
`Content-Type: application/problem+json` with `type`, `title`, `status`, `detail`,
`instance`, and `traceId`. The `traceId` field carries the W3C `traceparent` trace ID
for correlation with Jaeger.

**Idempotency:** Every write endpoint accepts an `Idempotency-Key: <uuid>` header
(NFR-REL-001). Services that receive this header cache the response for 24 hours (Redis,
keyed by `<service>:idem:<key>`). Repeat requests within the cache window return the
cached response without re-execution. Callers always supply the key for mutating calls.

**Pagination:** Cursor-based for all unbounded list endpoints. The cursor is an opaque
base64-encoded string containing `{id, direction}`. Page size default is 20, maximum 100.
Offset pagination is forbidden for any collection that may grow unbounded.

**JWT forwarding:** The API Gateway validates the JWT signature and forwards the raw
`Authorization: Bearer <token>` header plus extracted `X-User-Id` and `X-User-Role`
headers downstream. Downstream services trust `X-User-Id` and `X-User-Role` without
re-validating the JWT signature on every request — local validation is reserved for
security-sensitive secondary checks. This avoids per-request network calls to the Auth
Service for every downstream request (an anti-pattern that creates Auth as a synchronous
dependency for every service).

**Retry:** See §3.5 for the uniform retry and circuit breaker policy.

---

### 3.3 Synchronous communication — gRPC

#### 3.3.1 Design rationale

gRPC is assigned to exactly two service pairs. The criterion for gRPC over REST on
a synchronous path is: the call is in a latency-critical hot path where the
performance characteristics of HTTP/2 multiplexing, binary Protobuf serialisation,
and generated strongly-typed stubs provide a measurable advantage — and where the
additional build-time complexity (code generation, `.proto` contract management)
is justified by the learning objective of teaching cross-language RPC with explicit
contracts.

REST on the same paths would work. gRPC is chosen here to teach gRPC; the latency
budget for each path is tight enough that the binary serialisation advantage is
non-trivial.

#### 3.3.2 gRPC pair 1 — Booking Service → Seat Inventory Service

| Field                | Value                                                              |
|----------------------|--------------------------------------------------------------------|
| **Caller**           | Booking Service (Spring Boot 3, Java 21)                           |
| **Callee**           | Seat Inventory Service (Spring Boot 3, Java 21)                    |
| **Language pair**    | Java → Java                                                        |
| **Primary purpose**  | Seat availability check at checkout initiation                     |
| **NFR**              | NFR-PERF-033: p99 ≤ 300 ms; NFR-PERF-001: seat hold p99 < 500 ms |
| **Proto file**       | `stagepass-shared-contracts/proto/stagepass/seat_inventory/v1/seat_inventory.proto` |
| **Proto package**    | `stagepass.seat_inventory.v1`                                      |
| **Client library**   | `grpc-spring-boot-starter` + `protobuf-maven-plugin` (stub generation at build time) |
| **Server library**   | `grpc-spring-boot-starter` (`@GrpcService` annotation on implementation) |
| **Call type**        | Unary (synchronous request-response)                               |

**What this call does:**
At checkout, the Booking Service must confirm that the seats selected by the Customer
are still AVAILABLE before it creates the booking record and initiates payment. This
is the "guard before commit" in the saga. The gRPC call asks Seat Inventory: "are
these seats available?" Seat Inventory performs the Redis atomic check (or
pessimistic lock — per ADR-006) and returns an availability response. If available,
the Booking Service proceeds to create the PENDING booking. The actual HOLD operation
is written to Seat Inventory via the saga command path (Kafka, async), not via this
gRPC call — the gRPC call is read-only.

**Why not REST here?**
The checkout path has a 2-second synchronous budget (NFR-PERF-002). Within that budget,
the Gateway, JWT validation, Booking write, and this availability check all compete.
HTTP/2 framing, binary serialisation, and no per-request DNS resolution (gRPC channels
are long-lived) shave ~30–50 ms off the round trip compared to REST under load. That
margin matters when the budget is tight. The strongly-typed generated stub also means
a schema mismatch between Booking and Seat Inventory is a compile-time error, not a
runtime 400.

**Proto file sketch (not the full spec — full spec lives in stagepass-shared-contracts):**

```protobuf
syntax = "proto3";
package stagepass.seat_inventory.v1;

service SeatInventoryService {
  // Checks whether all requested seats are AVAILABLE.
  // Does NOT acquire a hold — hold is a separate saga step.
  rpc CheckSeatAvailability(CheckSeatAvailabilityRequest)
      returns (CheckSeatAvailabilityResponse);
}

message CheckSeatAvailabilityRequest {
  string event_id   = 1; // UUID
  repeated string seat_ids = 2; // UUIDs of requested seats
  string correlation_id = 3; // W3C traceparent propagated here
}

message CheckSeatAvailabilityResponse {
  bool all_available = 1;
  repeated UnavailableSeat unavailable_seats = 2;
}

message UnavailableSeat {
  string seat_id = 1;
  SeatState current_state = 2;
}

enum SeatState {
  SEAT_STATE_UNSPECIFIED = 0;
  AVAILABLE = 1;
  HELD      = 2;
  BOOKED    = 3;
  BLOCKED   = 4;
}
```

**OTel propagation:** The Booking Service injects the W3C `traceparent` and `tracestate`
values into the gRPC request metadata headers (standard gRPC OTel propagation via the
`opentelemetry-grpc` instrumentation library). The Seat Inventory Service extracts these
headers and creates a child span. The full checkout span is visible end-to-end in Jaeger.

---

#### 3.3.3 gRPC pair 2 — Check-in Service → Ticket Service

| Field                | Value                                                              |
|----------------------|--------------------------------------------------------------------|
| **Caller**           | Check-in Service (Fastify 4, TypeScript)                           |
| **Callee**           | Ticket Service (Fastify 4, TypeScript)                             |
| **Language pair**    | TypeScript → TypeScript (cross-framework, same runtime)            |
| **Primary purpose**  | Fetch per-event HMAC key on first scan (not the hot-path verify)   |
| **NFR**              | NFR-PERF-034: gRPC pre-validation p99 ≤ 200 ms                    |
| **Proto file**       | `stagepass-shared-contracts/proto/stagepass/ticket/v1/ticket.proto` |
| **Proto package**    | `stagepass.ticket.v1`                                              |
| **Client library**   | `@grpc/grpc-js` + `@grpc/proto-loader` (dynamic stub loading at startup) |
| **Server library**   | `@grpc/grpc-js` (raw gRPC server, no framework wrapper)            |
| **Call type**        | Unary (request-response)                                           |

**What this call does:**
QR check-in verification (NFR-PERF-004) must complete in p99 < 200 ms. To achieve
this, the hot path is a pure HMAC-SHA256 computation over a cached per-event secret
key — no network call. The gRPC call to Ticket Service is the **cold path only**: it
runs when the Check-in Service encounters an `eventId` for which it does not yet have
the per-event HMAC key in memory. This happens once per event per Check-in Service
instance (at startup or first scan). Subsequent verifications for the same event use
the in-memory cached key — zero network, zero disk, O(1) CPU.

The gRPC call fetches: the per-event HMAC secret (returned encrypted using the
Check-in Service's service certificate public key, decrypted locally), the event's
valid date range (so the service can reject tokens presented outside event windows
without a network call), and the event status.

**Why gRPC here?**
This call teaches cross-service gRPC in TypeScript — the second half of the gRPC
learning objective. The cross-language lesson applies equally to TypeScript-to-TypeScript
because the contract discipline (`@grpc/proto-loader` loading the same `.proto` file
from `stagepass-shared-contracts`, generating matching types at runtime) is identical
to the cross-language case. The Protobuf contract is the thing that makes it
language-agnostic in production; we learn that lesson here.

Note: ADR-002 originally noted "TypeScript–TypeScript" for this pair (not Java as a
common misreading of the proto-loader setup). The Ticket Service is Fastify 4 TypeScript
per the framework table. The Check-in Service is also Fastify 4 TypeScript.

**OTel propagation:** `@opentelemetry/instrumentation-grpc` automatically injects and
extracts `traceparent` from gRPC metadata on both sides.

---

### 3.4 Asynchronous communication — Kafka

#### 3.4.1 General Kafka conventions

These conventions apply to every topic on the platform without exception.

**Delivery semantics:** At-least-once on the producer side (acks=all, retries=MAX_INT,
idempotent producer enabled). Every consumer must be idempotent — processing the same
message twice must produce the same result as processing it once (NFR-REL-002).

**Consumer groups:** Each service that consumes from a topic has its own named consumer
group. If two services independently consume the same topic (fan-out), they each have
their own group — messages are delivered to both. Consumer group naming convention:
`<service-name>-consumer`.

**Offset commit:** Manual offset commit after successful processing. Never auto-commit.
Auto-commit risks marking a message consumed before it has been processed, losing it on
crash.

**Dead-letter queues:** Every topic has a DLQ topic named `<original-topic>.dlq`
(NFR-REL-007). After 3 delivery attempts with exponential backoff, the consumer publishes
the original message to the DLQ with an additional header:
`x-failure-reason: <exception class and message>`. A Prometheus alert fires when any
DLQ topic depth exceeds 0 for more than 5 continuous minutes.

**Schema registry:** Kafka messages use JSON (not Avro) in Phase 1–4 for simplicity.
JSON schemas are documented in AsyncAPI 2.x and stored in
`stagepass-shared-contracts/schemas/events/<domain>/`. Each event carries a `schemaVersion`
field. Schema evolution is additive only: new optional fields may be added; existing
fields may not be removed or have their type changed. Consumers must tolerate unknown
fields (tolerant reader pattern — see Kleppmann DDIA §4, "Dataflow Through Message-Passing").

**Message envelope:** Every Kafka message carries these headers:

```
x-event-type:      <TopicName>          // e.g. "booking.seat-held"
x-correlation-id:  <UUID>               // same value as X-Correlation-Id in REST
x-traceparent:     <W3C traceparent>    // OTel trace context (NFR-OBS-003)
x-tracestate:      <W3C tracestate>     // OTel trace state
x-produced-at:     <ISO-8601 UTC>       // producer timestamp
x-schema-version:  <semver string>      // e.g. "1.0.0"
```

**OTel propagation:** Producers inject W3C `traceparent` and `tracestate` into message
headers using the OpenTelemetry Kafka instrumentation (OTel Java agent for Spring Boot
services; `@opentelemetry/instrumentation-kafkajs` for Node services; `opentelemetry-sdk`
manual propagation for Python). Consumers extract these headers and create child spans
linked to the producer span. This gives an unbroken trace across every Kafka hop in
the booking saga — observable end-to-end in Jaeger (NFR-OBS-003, NFR-OBS-004).

---

#### 3.4.2 Kafka topic catalogue

The following table is the authoritative list of all Kafka topics on the platform.
No topic may be created in code that is not in this table. New topics require an
ADR amendment or a new ADR.

##### Booking domain

| Topic | Partition Key | Partitions | Retention | Consumers (group) | DLQ |
|-------|--------------|------------|-----------|-------------------|-----|
| `booking.commands` | `bookingId` | 12 | 7 days | booking-service-consumer | `booking.commands.dlq` |
| `booking.events` | `bookingId` | 12 | 7 days | notification-service-consumer, analytics-service-consumer, fraud-detection-service-consumer | `booking.events.dlq` |
| `booking.refund-requested` | `bookingId` | 12 | 7 days | payment-service-consumer, disbursement-service-consumer | `booking.refund-requested.dlq` |

**Partition key rationale — `bookingId`:** All events in a booking's lifecycle must be
processed in order within a single consumer instance. If `booking.confirmed` is processed
before `booking.seat-held`, the saga state machine corrupts. Keying by `bookingId` ensures
all messages for a booking land on the same partition and are consumed in order by the
same consumer instance within the group.

##### Payment domain

| Topic | Partition Key | Partitions | Retention | Consumers (group) | DLQ |
|-------|--------------|------------|-----------|-------------------|-----|
| `payment.events` | `paymentOrderId` | 6 | 7 days | booking-service-consumer, disbursement-service-consumer | `payment.events.dlq` |

**Partition key rationale — `paymentOrderId`:** Payment events belong to a specific payment
order. The Booking Service needs ordered delivery of payment events (pending → authorised →
captured) to drive the saga. Keying by payment order ID ensures this ordering per order.

##### Seat domain

| Topic | Partition Key | Partitions | Retention | Consumers (group) | DLQ |
|-------|--------------|------------|-----------|-------------------|-----|
| `seat.commands` | `eventId` | 24 | 3 days | seat-inventory-service-consumer | `seat.commands.dlq` |
| `seat.state-changes` | `eventId` | 24 | 1 day | notification-service-consumer, booking-service-consumer | `seat.state-changes.dlq` |

**Partition key rationale — `eventId`:** Seat state changes for a given event must be
consumed in order — an AVAILABLE→HELD→BOOKED sequence must not be observed as
AVAILABLE→BOOKED→HELD by the consumer that feeds the real-time seat map. Keying by
`eventId` collocates all seat events for one event on the same partition. 24 partitions
support up to 24 concurrent events being processed at peak without cross-event ordering
constraints. The real-time seat map (NFR-PERF-005: p99 < 1 s push) requires low consumer
lag — see NFR-PERF-032.

##### Ticket domain

| Topic | Partition Key | Partitions | Retention | Consumers (group) | DLQ |
|-------|--------------|------------|-----------|-------------------|-----|
| `ticket.commands` | `bookingId` | 12 | 3 days | ticket-service-consumer | `ticket.commands.dlq` |
| `ticket.events` | `ticketId` | 12 | 30 days | notification-service-consumer, check-in-service-consumer | `ticket.events.dlq` |
| `ticket.checked-in` | `ticketId` | 12 | 90 days | analytics-service-consumer, notification-service-consumer | `ticket.checked-in.dlq` |

##### Notification domain

| Topic | Partition Key | Partitions | Retention | Consumers (group) | DLQ |
|-------|--------------|------------|-----------|-------------------|-----|
| `notification.commands` | `userId` | 12 | 1 day | notification-service-consumer | `notification.commands.dlq` |

**Partition key rationale — `userId`:** Notifications for a single user must arrive in
order at the Notification Service (e.g., "seat held" before "booking confirmed"). Keying
by `userId` collocates a user's notifications on one partition. The Notification Service
consumer will see them in order and emit WebSocket events to the user's room in order.

##### Event domain

| Topic | Partition Key | Partitions | Retention | Consumers (group) | DLQ |
|-------|--------------|------------|-----------|-------------------|-----|
| `event.commands` | `eventId` | 12 | 7 days | event-service-consumer | `event.commands.dlq` |
| `event.events` | `eventId` | 12 | 30 days | search-service-consumer, recommendation-service-consumer, analytics-service-consumer, booking-service-consumer, seat-inventory-service-consumer | `event.events.dlq` |

**`event.events` — multi-consumer fan-out:** When an event is cancelled, multiple services
react independently. Booking Service initiates refund sagas. Search Service removes the
event from the index. Recommendation Service invalidates related caches. Analytics Service
records the cancellation metric. Each service has its own consumer group, so each receives
every message independently. Kafka fan-out at zero additional producer cost.

##### Venue domain

| Topic | Partition Key | Partitions | Retention | Consumers (group) | DLQ |
|-------|--------------|------------|-----------|-------------------|-----|
| `venue.events` | `venueId` | 6 | 30 days | event-service-consumer, search-service-consumer | `venue.events.dlq` |

##### Disbursement domain

| Topic | Partition Key | Partitions | Retention | Consumers (group) | DLQ |
|-------|--------------|------------|-----------|-------------------|-----|
| `disbursement.commands` | `disbursementId` | 6 | 30 days | disbursement-service-consumer | `disbursement.commands.dlq` |
| `disbursement.events` | `disbursementId` | 6 | 90 days | analytics-service-consumer, notification-service-consumer | `disbursement.events.dlq` |

##### Fraud domain

| Topic | Partition Key | Partitions | Retention | Consumers (group) | DLQ |
|-------|--------------|------------|-----------|-------------------|-----|
| `fraud.score-requests` | `bookingId` | 12 | 3 days | fraud-detection-service-consumer | `fraud.score-requests.dlq` |
| `fraud.score-results` | `bookingId` | 12 | 7 days | booking-service-consumer, admin-notification-consumer | `fraud.score-results.dlq` |

**NFR-PERF-011:** Fraud score must be published within 10 s of booking placement. The
Booking Service publishes to `fraud.score-requests` immediately after creating the PENDING
booking record. Fraud Detection Service must consume and publish to `fraud.score-results`
within 10 s. The fraud score result is async — it does not block the booking saga. If the
Fraud Detection Service is down, the booking proceeds as LOW-risk (NFR-AVAIL-005,
fail-open). This is enforced by never making the booking saga await the fraud score.

##### Waitlist domain

| Topic | Partition Key | Partitions | Retention | Consumers (group) | DLQ |
|-------|--------------|------------|-----------|-------------------|-----|
| `waitlist.commands` | `eventId` | 6 | 7 days | waitlist-service-consumer | `waitlist.commands.dlq` |
| `waitlist.events` | `eventId` | 6 | 7 days | notification-service-consumer | `waitlist.events.dlq` |
| `waitlist.offer-expired` | `customerId` | 6 | 1 day | waitlist-service-consumer | `waitlist.offer-expired.dlq` |

##### Flash sale domain

| Topic | Partition Key | Partitions | Retention | Consumers (group) | DLQ |
|-------|--------------|------------|-----------|-------------------|-----|
| `flash-sale.hold-requests` | `eventId` | 24 | 1 day | seat-inventory-service-consumer | `flash-sale.hold-requests.dlq` |
| `flash-sale.hold-results` | `customerId` | 12 | 1 day | notification-service-consumer, booking-service-consumer | `flash-sale.hold-results.dlq` |

**Flash sale topics are separate from `seat.commands`.** The flash sale funnel pattern
(ADR-007) uses dedicated topics so that: (a) the flash sale queue can be consumed at a
controlled rate independently from normal seat operations; (b) flash sale queue depth
is a distinct observable metric (`kafka_consumergroup_lag` on `flash-sale.hold-requests`
consumer group) visible in Grafana (NFR-OBS-005); (c) normal bookings are not queued
behind flash sale traffic.

---

#### 3.4.3 Consumer library selection per runtime

| Runtime     | Library         | Notes                                               |
|-------------|-----------------|-----------------------------------------------------|
| Java (Spring Boot, Quarkus) | `spring-kafka` (`@KafkaListener`) | `ConcurrentKafkaListenerContainerFactory` for parallel partition consumption. `ErrorHandlingDeserializer` for poison pill protection. `SeekToCurrentErrorHandler` with backoff for retry before DLQ publish. |
| TypeScript (NestJS, Fastify, raw Node.js) | `kafkajs` | Consumer group join, subscribe, `run` with `eachMessage` handler. Manual `commitOffsets` after processing. No framework wrapper in Notification Service — raw `kafkajs` lifecycle explicit. |
| Python (FastAPI, Flask) | `aiokafka` | Async consumer compatible with FastAPI's `asyncio` event loop. `AIOKafkaConsumer` with `auto_offset_reset='earliest'`. Manual commit via `consumer.commit()`. |

---

### 3.5 Retry and circuit breaker policy

This policy applies uniformly to all synchronous outbound calls (REST and gRPC) from
every service (NFR-REL-006). It is not optional and is not configurable per-service
without an ADR amendment.

#### 3.5.1 Retry policy

```
max_attempts = 3
base_delay_ms = 100
max_delay_ms  = 5000

delay(attempt) = min(base_delay_ms × 2^attempt + random(0, base_delay_ms), max_delay_ms)

attempt 0 (first try):  no delay
attempt 1 (first retry): ~100–200 ms
attempt 2 (second retry): ~200–400 ms
(third failure → publish to DLQ or return error to caller)
```

**Jitter rationale:** Without random jitter, all callers retrying a recovering service
retry at the same interval — causing a thundering herd that re-fails the service. Adding
`random(0, base)` spreads retries across a window, giving the recovering service time to
process some requests between waves. This is the "decorrelated jitter" approach documented
by AWS (see References).

**Retryable conditions:** HTTP 429 (rate limited), HTTP 502/503/504 (gateway/service
unavailable), network timeout, gRPC status codes UNAVAILABLE and DEADLINE_EXCEEDED.

**Non-retryable conditions:** HTTP 400 (bad request — retrying will not fix a malformed
payload), HTTP 401/403 (authentication/authorisation failure), HTTP 409 (conflict —
retrying an idempotent request with the same key is safe, but the response will be the
same). Note: a 409 with an `Idempotency-Key` header should return the cached success
response, not a new 409 — idempotency handling takes precedence.

#### 3.5.2 Circuit breaker policy

```
failure_threshold        = 5 consecutive failures within 10 s
state: CLOSED → OPEN:    after failure_threshold
state: OPEN  → HALF_OPEN: after 30 s
state: HALF_OPEN → CLOSED: after 3 consecutive successes in HALF_OPEN
state: HALF_OPEN → OPEN:   on first failure in HALF_OPEN
```

**OPEN state behaviour:** Return a circuit-open error immediately without making a
network call. The caller receives the pre-configured fallback response (e.g., 503 with
`Retry-After` header, or a stub response where defined in the availability NFRs).

**Per-library implementation:**

| Runtime    | Library               |
|------------|-----------------------|
| Java       | Resilience4j (`CircuitBreaker`, `Retry` composable decorators) |
| TypeScript | `opossum` (circuit breaker) + `axios-retry` (retry) |
| Python     | `pybreaker` (circuit breaker) + `tenacity` (retry) |

All circuit breaker configuration values must be externalised via environment variables
or the Spring Cloud Config Server — never hardcoded. This enables per-environment
tuning (e.g., a more aggressive threshold in production, a looser one in development)
without a code change.

---

### 3.6 WebSocket — Socket.IO rooms and event routing

#### 3.6.1 Architecture

WebSocket connections terminate exclusively at the Notification Service. No other
service maintains a WebSocket connection to the browser. The Notification Service
is the single fan-out point: it consumes Kafka events from the topics listed in §3.4.2
and emits the corresponding Socket.IO events to the appropriate rooms.

```
Browser → Socket.IO handshake → Notification Service instance (any)
Browser ↑ receives events from room
                         ↕ (Redis Socket.IO adapter)
Notification Service instances (1..N) — share room state via Redis Pub/Sub
                         ↑
Kafka topics → Notification Service consumers
```

The Redis Socket.IO adapter (`@socket.io/redis-adapter`) using `ioredis` is mandatory.
Without it, a browser connected to instance A would not receive events emitted by
instance B. The adapter replicates Socket.IO room membership and messages across all
instances via Redis Pub/Sub channels. This is what enables horizontal scaling
(NFR-PERF-031: 5,000 concurrent connections).

#### 3.6.2 Room topology

| Room Name         | Scope      | Who joins             | Events pushed into this room |
|-------------------|------------|-----------------------|------------------------------|
| `user:{userId}`   | Per user   | Customer on connect (authenticated) | `booking.confirmed`, `booking.failed`, `booking.cancelled`, `ticket.issued`, `waitlist.offer-received`, `waitlist.offer-expired`, `payment.status-changed` |
| `event:{eventId}` | Per event  | Any client on seat map page; Organisers on event dashboard | `seat.state-changed`, `sales.counter-updated`, `attendance.updated` |
| `flash-sale:{eventId}` | Per flash sale | Customers in the flash sale queue | `queue.position-updated`, `queue.hold-granted`, `queue.hold-denied` |

**Room join authentication:** On Socket.IO connection, the client sends an `auth` object
with the JWT. The Notification Service validates the JWT (RS256, using the JWKS endpoint
cached in memory) before allowing room joins. Unauthenticated connections may join public
`event:{eventId}` rooms but not `user:{userId}` rooms. An unauthenticated client
attempting to join `user:{userId}` is disconnected immediately.

**Event payload constraint (NFR-PERF-005 and FR-P-001 AC5):** The WebSocket payload
for a `seat.state-changed` event must contain only the changed seat(s), not the full
seat map. For a stadium-scale event (50,000 seats), sending the full map on every state
change would exceed WebSocket frame limits and make the p99 < 1 s constraint impossible.
Payload: `{ eventId, seats: [{ seatId, newState }] }` — differential only.

**Initial snapshot on join (FR-P-001 AC3):** When a client joins an `event:{eventId}`
room, the Notification Service sends a `seat.snapshot` event with the full current seat
state map. This is fetched synchronously from the Seat Inventory Service (REST or Redis
read) at join time and emitted only to the joining socket. Subsequent messages are
differential. This separates the "sync on join" problem from the "differential push"
problem — solving them together in one complex stream is an anti-pattern.

#### 3.6.3 Socket.IO event naming convention

All Socket.IO events use `domain.action` naming matching the Kafka topic convention.
The mapping from Kafka topic to Socket.IO event:

| Kafka topic consumed             | Socket.IO event emitted       | Room target       |
|----------------------------------|-------------------------------|-------------------|
| `booking.events` (confirmed)     | `booking.confirmed`           | `user:{userId}`   |
| `booking.events` (failed)        | `booking.failed`              | `user:{userId}`   |
| `seat.state-changes`             | `seat.state-changed`          | `event:{eventId}` |
| `ticket.events` (issued)         | `ticket.issued`               | `user:{userId}`   |
| `waitlist.events` (offer)        | `waitlist.offer-received`     | `user:{userId}`   |
| `analytics events` (sales)       | `sales.counter-updated`       | `event:{eventId}` |
| `ticket.checked-in`              | `attendance.updated`          | `event:{eventId}` |
| `flash-sale.hold-results`        | `queue.hold-granted/denied`   | `flash-sale:{eventId}` |

---

### 3.7 OpenTelemetry propagation — per transport summary

This section is the canonical reference for trace context propagation. Every service
must implement it. NFR-OBS-003 and NFR-OBS-004 are only achievable if this is
universally applied.

| Transport  | Propagation format                     | How injected                          | How extracted                          |
|------------|----------------------------------------|---------------------------------------|----------------------------------------|
| REST (HTTP) | W3C TraceContext (`traceparent`, `tracestate`) | OTel Java agent / `@opentelemetry/instrumentation-http` / `opentelemetry-sdk` — automatic on outbound request | OTel agent / instrumentation — automatic on inbound request |
| gRPC       | W3C TraceContext in gRPC metadata headers | `opentelemetry-grpc` instrumentation — automatic on client stub call | `opentelemetry-grpc` — automatic on server handler entry |
| Kafka      | W3C TraceContext in Kafka message headers (`traceparent`, `tracestate`) | OTel Java Kafka agent instrumentation / `@opentelemetry/instrumentation-kafkajs` / manual Python SDK propagation on `producer.send()` | OTel instrumentation on `consumer.run()` / `@KafkaListener` entry |
| Socket.IO  | `x-trace-id` in Socket.IO event payload | Notification Service adds traceId from current span when emitting | Browser reads and logs; no server-side extraction needed for browser-originated events |

**Correlation ID vs. Trace ID:** These are different things used together.
- **Trace ID** (W3C `traceparent`): generated by OTel, propagated automatically, links all
  spans in one request tree across service boundaries. Used in Jaeger.
- **Correlation ID** (`X-Correlation-Id` header, `x-correlation-id` Kafka header): a business
  identifier (UUID) generated at the API Gateway for each inbound request. Included in every
  structured log entry alongside the trace ID. Used to find all logs for a user-initiated action
  in Loki. While trace ID finds spans in Jaeger, correlation ID finds log lines in Loki that may
  not have a span (e.g., a background scheduled job).

Both must be present in every log entry (NFR-OBS-001).

---

## 4. Consequences

### 4.1 Positive

**Uniform retry and circuit breaker policy** prevents cascading failures. A single
misconfigured retry (no jitter, no cap) from any service can cause a thundering herd that
takes down a recovering dependency. By standardising on Resilience4j / opossum / tenacity
with the formula in §3.5, this failure mode is eliminated across all 17 services at once.

**Named Kafka topics with explicit DLQs** mean message loss requires a bug in the
framework, not just an oversight. Every consumer knows where failures go. On-call engineers
have a runbook (triggered by the DLQ depth alert) that tells them what to do.

**WebSocket via Notification Service only** keeps the browser-facing push architecture
clean. Seat Inventory does not need to know about WebSocket. Payment does not need to know
about WebSocket. Each produces a Kafka event; the Notification Service handles the last mile.
Changing the WebSocket implementation (e.g., switching from Socket.IO to raw WebSocket)
touches exactly one service.

**OTel propagation across all transports** makes the booking saga — the most complex flow
in the system — fully visible in Jaeger from API Gateway to WebSocket push. This is the
difference between a system that is observable and one that merely emits logs.

**Two gRPC pairs** teach gRPC at the right scope. More gRPC would add operational complexity
(more `.proto` files to version, more stub generation to maintain) without proportional
learning return. Fewer gRPC pairs would miss the cross-language lesson. Two is the right
number for this project.

### 4.2 Negative / trade-offs

**Kafka adds operational overhead.** A six-consumer-group pattern on `event.events` means
six independent consumer groups to monitor, six independent lag metrics, and six independent
failure scenarios to handle. This is the correct trade-off for fan-out decoupling, but it
is not free. Every consumer group gets its own lag alert.

**At-least-once delivery requires idempotent consumers.** This is a correctness requirement
that must be implemented per-consumer, not once. The platform has approximately 22 consumer
group–topic pairs (from the catalogue in §3.4.2). Each must be individually tested for
idempotency. This is done, not waived.

**WebSocket rooms require join/leave lifecycle management.** If a client disconnects without
leaving an `event:{eventId}` room, the Notification Service must clean up the room
membership to avoid memory leaks. Socket.IO's built-in disconnect handler handles this, but
it must be explicitly implemented and tested.

**The circuit breaker adds a stateful component per outbound call.** Resilience4j circuit
breaker state is in-process memory. In a horizontally scaled service (multiple Booking
Service instances), each instance has its own circuit breaker state — one instance may be
OPEN while another is CLOSED. This is acceptable: the circuit breaker's purpose is to
protect the individual instance from making calls it expects to fail, not to coordinate a
platform-wide OPEN/CLOSED decision. If the actual downstream service is failing, all
instances will independently open their breakers within their failure window.

**gRPC binary protocol makes debugging harder than REST.** A REST payload can be read
with `curl`. A gRPC payload requires a gRPC client, `grpcurl`, or a service mesh
inspector (Istio in Phase 9). During development, add gRPC server reflection to both
Seat Inventory and Ticket gRPC servers so `grpcurl` works without a `.proto` file on the
debugging machine.

---

## 5. Alternatives Considered

### 5.1 gRPC vs. REST for Booking → Seat Inventory

**Option A: gRPC (chosen).** Binary serialisation, HTTP/2 multiplexing, compile-time
contract enforcement via generated stubs. The 300 ms p99 budget (NFR-PERF-033) and the
need to teach gRPC make this the right choice here.

**Option B: REST.** Simpler setup, easier to debug. Would add ~30–50 ms per call under
load (JSON parsing, connection setup overhead vs. long-lived HTTP/2 channel). The checkout
path's 2 s synchronous budget is tight enough that this margin is worth recovering.
More importantly, REST for this call would mean no Java-to-Java gRPC lesson, and the
project would need to find another place to teach gRPC.

**Option C: Redis-based IPC (direct key read).** The Booking Service could read seat state
directly from the Seat Inventory Service's Redis store. Faster, but breaks service
encapsulation — Booking Service would know the internal data model of Seat Inventory,
coupling the two at the storage layer. ADR-001 mandates one database per service, owned
exclusively by that service. This option violates that constraint.

**Decision: Option A.** The combination of latency benefit, compile-time contract
enforcement, and learning objective justifies the gRPC overhead for this specific path.

---

### 5.2 Kafka vs. REST for event-driven flows

**Option A: Kafka (chosen).** At-least-once delivery. Fan-out to multiple consumers via
consumer groups without producer awareness of consumers. Consumer-side back-pressure
(consumer reads at its own pace; Kafka buffers). Replay from any offset. Built-in DLQ
pattern. Correct for a distributed saga where any step can fail and must be retried.

**Option B: REST webhooks (service calls service directly).** Simpler. Works fine for
two services. Breaks at fan-out: when an event is cancelled, Booking Service would need
to know the addresses of all services that care (Ticket, Notification, Analytics, Search)
and call each one. Adding a new consumer requires changing the producer. This is the
tight coupling that Kafka eliminates. At-least-once delivery requires the caller to
implement retries; under load (10,000 booking cancellations), this produces an outbound
REST storm from Booking Service that targets already-loaded services.

**Option C: RabbitMQ or Amazon SQS.** Message queue alternatives with different
trade-off profiles. RabbitMQ uses push delivery (broker pushes to consumer); Kafka uses
pull (consumer reads at its own pace). Pull is strictly better for back-pressure under
load: a Kafka consumer that is behind simply reads slower; a RabbitMQ consumer that is
behind receives messages faster than it can process them, causing queue growth in the
consumer process. SQS adds AWS dependency too early (cloud is Phase 9). Kafka also
provides the ordered delivery guarantee (by partition key) that the booking saga requires.
RabbitMQ's ordering guarantees are weaker.

**Decision: Option A.** Kafka's fan-out, ordered delivery, replay, and back-pressure
model are the correct fit for a distributed saga platform. The operational overhead is
justified at this scale.

---

### 5.3 Socket.IO vs. raw WebSocket for real-time push

**Option A: Socket.IO (chosen).** Built-in room management. Automatic reconnection with
exponential backoff. Long-polling fallback for environments where WebSocket is unavailable
(some corporate proxies, older mobile networks). Redis adapter for horizontal scaling.
Heartbeat and timeout handling. Namespace isolation.

**Option B: Raw WebSocket (RFC 6455).** Lower overhead. No client library required. But:
room management must be hand-built (which rooms is a connection in? how do you broadcast
to a room? how do you handle disconnection cleanup?). Horizontal scaling requires a custom
pub/sub layer — exactly what the Redis Socket.IO adapter provides. Reconnection must be
hand-built on the client. For a learning project, building this from scratch would teach
more — but in the Notification Service, which is already using raw Node.js + manual HTTP
wiring, the scope of "what to build by hand" is already set. Adding raw WebSocket room
management on top would make the service too large for one phase.

**Option C: Server-Sent Events (SSE).** Unidirectional server-to-client push over
HTTP/1.1. Simpler than WebSocket for pure push (no bidirectional communication needed
in this service — the browser only receives, it never sends via WebSocket). But SSE has
per-browser connection limits (6 connections per origin in HTTP/1.1, unlimited in HTTP/2).
More importantly, Socket.IO's room semantics — subscribing a connection to a named room
and broadcasting to all members — require the fan-in/fan-out primitives that SSE does
not provide natively. A custom room mapping on top of SSE would reproduce what Socket.IO
already provides.

**Decision: Option A.** Socket.IO's room model, Redis adapter, and reconnection handling
solve the exact problems this service faces. The trade-off is the Socket.IO client
dependency in the browser (50 KB gzipped), which is acceptable at this scale.

---

## 6. References

| Source | Relevance |
|--------|-----------|
| PRD.md §3 — Tech Stack | Primary source for protocol assignments (gRPC pairs, Kafka, Socket.IO) |
| PRD.md §8.6 FR-P-001 | Real-time seat map acceptance criteria; drives room topology |
| PRD.md §8.6 FR-P-002 | QR check-in requirements; drives gRPC pair 2 design |
| PRD.md §8.6 FR-P-003 | Flash sale queue; drives dedicated flash-sale topic design |
| NFR-REL-001 | Idempotency key requirement on all write endpoints |
| NFR-REL-002 | Kafka consumer idempotency requirement |
| NFR-REL-006 | Retry policy with backoff formula; circuit breaker specification |
| NFR-REL-007 | DLQ requirement; depth alert threshold; naming convention |
| NFR-PERF-001 | Seat hold p99 < 500 ms; informs synchronous path design |
| NFR-PERF-004 | Check-in p99 < 200 ms; informs gRPC pair 2 cold-path design |
| NFR-PERF-005 | Seat map WebSocket push p99 < 1 s; informs differential payload requirement |
| NFR-PERF-031 | 5,000 concurrent WebSocket connections; mandates Redis Socket.IO adapter |
| NFR-PERF-032 | Consumer lag ≤ 5,000 per partition; informs partition count choices |
| NFR-PERF-033 | gRPC Booking → Seat Inventory p99 ≤ 300 ms |
| NFR-OBS-003 | OTel context propagation across HTTP, gRPC, and Kafka |
| NFR-OBS-004 | End-to-end saga traceability in < 2 minutes |
| ADR-001 | Repo topology; confirms stagepass-shared-contracts as the home for .proto files |
| ADR-002 | Framework per service; confirms gRPC library choices per runtime |
| Kleppmann — DDIA §4 | Schema evolution (tolerant reader), message-passing dataflow |
| Kleppmann — DDIA §8 | Distributed systems failure modes; retry and timeout rationale |
| Richardson — Microservices Patterns, Ch. 3 | Saga pattern; event-driven communication |
| Newman — Building Microservices, Ch. 5 | Service communication choices and trade-offs |
| AWS Architecture Blog — "Exponential Backoff and Jitter" | Jitter formula rationale |
| gRPC documentation — Metadata propagation | gRPC header-based OTel propagation |
| W3C TraceContext spec | `traceparent` and `tracestate` header format |
| Socket.IO documentation — Rooms, Redis Adapter | Room management and horizontal scaling |

---

## 7. Quick Self-Check

Answer these before implementing any inter-service call. If you cannot answer them
without looking at this ADR, re-read the relevant section before writing code.

**Question 1:** A new developer adds a REST call from Service A to Service B. They
implement it without an idempotency key and without any retry logic, reasoning that
"it's just a GET." What two things are wrong with this reasoning, and what must they
add even for a GET endpoint?

> *Expected answer:* (1) Idempotency keys are required on all write endpoints (NFR-REL-001),
> not GETs — but the real error is building a habit of skipping it for "simple" calls.
> (2) Retry logic is required on every synchronous outbound call regardless of method —
> network transients don't discriminate by HTTP verb. Even a GET needs the backoff formula
> from §3.5 applied. For a GET specifically, retries are safe by definition (GETs are
> idempotent at the HTTP level), so there is no excuse not to retry.

**Question 2:** The booking saga emits a `booking.events` message for a booking that is
confirmed. Three services (Notification, Analytics, Fraud Detection) must all receive it.
A junior developer proposes solving this with three separate REST webhook calls from
Booking Service. Name the two architectural problems this creates and explain how the
Kafka consumer group model solves both.

> *Expected answer:* (1) Tight coupling — Booking Service must know the addresses of all
> three consumers. Adding a fourth consumer requires a code change to Booking Service.
> With Kafka, new consumers join a new consumer group; the producer is unchanged.
> (2) Cascading failure risk — if one webhook call fails under load (e.g., Analytics is
> slow during a peak), Booking Service either blocks waiting for it (saga slowdown) or
> fires-and-forgets (no delivery guarantee). With Kafka, each consumer reads at its own
> pace; back-pressure does not propagate to the producer.

**Question 3:** You are on call. Grafana fires an alert: `KafkaDLQDepthNonZero` for
`booking.events.dlq`. Walk through the steps you take to determine root cause, and name
what two pieces of information the message in the DLQ header gives you that a log line
alone cannot.

> *Expected answer:* Steps: (1) Open Grafana, confirm which consumer group is lagging on
> `booking.events.dlq`. (2) Read the DLQ messages — each carries the `x-failure-reason`
> header with the exception class and message. (3) Correlate `x-correlation-id` and
> `x-traceparent` from the DLQ message header against Jaeger and Loki to find the original
> request's trace and logs. (4) Determine whether the failure is transient (network blip —
> replay the message) or a code bug (fix first, then replay). The two pieces DLQ headers
> give that a log line alone cannot: (a) the original Kafka message payload exactly as the
> producer sent it (the log may have logged a partial representation); (b) the
> `x-failure-reason` captures the exact exception at the consumer — which log line it
> corresponds to may not be obvious if multiple consumers log similarly.

---

*End of ADR-003*
