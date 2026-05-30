# High-Level Design — StagePass Platform

## Document Header

| Field        | Value                                                                                 |
|--------------|---------------------------------------------------------------------------------------|
| Document ID  | HLD                                                                                   |
| Title        | StagePass Platform — High-Level Design (C4 Level 1 + Level 2)                        |
| Version      | 1.0.0                                                                                 |
| Status       | Accepted                                                                              |
| Author       | StagePass Engineering                                                                 |
| Created      | 2025-05-16                                                                            |
| Last Updated | 2025-05-16                                                                            |
| Repo         | stagepass-docs                                                                        |
| Path         | /docs/architecture/HLD.md                                                             |
| Phase        | Phase 1 — Requirements & Design                                                       |
| Traces To    | PRD · NFR · ADR-001 · ADR-002 · ADR-003 · ADR-004 · ADR-005 · ADR-006 · ADR-007 · ADR-008 |

### Change Log

| Version | Date       | Author                | Summary                  |
|---------|------------|-----------------------|--------------------------|
| 1.0.0   | 2025-05-16 | StagePass Engineering | Initial accepted version |

---

## Table of Contents

1. [Purpose and Scope](#1-purpose-and-scope)
2. [Architectural Constraints Reflected](#2-architectural-constraints-reflected)
3. [C4 Level 1 — System Context Diagram](#3-c4-level-1--system-context-diagram)
4. [C4 Level 2 — Container Diagram](#4-c4-level-2--container-diagram)
   - 4.1 [Service Communication View](#41-service-communication-view)
   - 4.2 [Data Store Ownership Table](#42-data-store-ownership-table)
5. [Service Tier Map](#5-service-tier-map)
6. [Key Architectural Flows](#6-key-architectural-flows)
   - 6.1 [Normal Booking Path (Saga)](#61-normal-booking-path-saga)
   - 6.2 [Flash Sale Path](#62-flash-sale-path)
   - 6.3 [Event Cancellation Cascade](#63-event-cancellation-cascade)
   - 6.4 [Disbursement Trigger](#64-disbursement-trigger)
7. [Shared Infrastructure](#7-shared-infrastructure)
8. [Observability Architecture](#8-observability-architecture)
9. [NFR Compliance Notes](#9-nfr-compliance-notes)
10. [Quick Self-Check](#10-quick-self-check)

---

## 1. Purpose and Scope

This document is the authoritative High-Level Design for the StagePass platform. It provides a
system-level context view (C4 Level 1) and a full container view (C4 Level 2) spanning all
18 services (17 domain services + 1 API Gateway), all data stores, the Kafka message bus,
and shared infrastructure.

Every architectural element in this document traces to at least one of ADR-001 through ADR-008.
This document does not introduce new decisions — it visualises and consolidates decisions
already locked in the ADR corpus.

**Reading guide:** If a diagram does not render in your browser, paste the Mermaid code block
into [https://mermaid.live](https://mermaid.live) for an interactive view.

---

## 2. Architectural Constraints Reflected

The following locked decisions from ADR-001 through ADR-008 directly shape this diagram and
must be preserved in every subsequent design and implementation decision.

| ADR    | Constraint reflected in this HLD                                                                 |
|--------|--------------------------------------------------------------------------------------------------|
| ADR-001 | Polyrepo topology — service repo boundaries map exactly to container boundaries in L2           |
| ADR-002 | Exact framework per service used as container technology label; one database per service         |
| ADR-003 | REST for Gateway→services; exactly two gRPC pairs; Kafka for all async; Socket.IO for push only |
| ADR-004 | PostgreSQL NUMERIC(19,4), MongoDB Decimal128 for all money columns — annotated on DB nodes       |
| ADR-005 | Booking Service is the saga orchestrator; coordinates Seat Inventory, Payment, Ticket, Disbursement |
| ADR-006 | Seat Inventory uses a two-store pattern: Redis (atomic SETNX, 600s TTL) + PostgreSQL (durable)  |
| ADR-007 | Flash sale path routes via `flash-sale.hold-requests` Kafka topic, bypassing gRPC               |
| ADR-008 | Disbursement Service owns the logical escrow ledger; immutable append-only PostgreSQL records     |

---

## 3. C4 Level 1 — System Context Diagram

This view shows StagePass as a single system, all four human actors, and all external systems
the platform integrates with. Internal structure is intentionally hidden at this level.

```mermaid
C4Context
    title StagePass — System Context (C4 Level 1)

    Person(customer,   "Customer",   "Discovers events, selects seats, books tickets, attends shows. India market.")
    Person(organiser,  "Organiser",  "Creates events at confirmed Venues, manages ticket sales and revenue.")
    Person(venue_actor,"Venue",      "Lists physical spaces, responds to Organiser booking requests, configures seating.")
    Person(admin,      "Admin",      "Operates the platform: KYC approval, fraud review, disbursement release, category management.")

    Enterprise_Boundary(india, "India Market — INR only, DPDP Act + GST compliance") {
        System(stagepass, "StagePass Platform",
            "AI-enhanced multi-party event ticketing marketplace. 17 microservices + API Gateway. Real-time seat map, flash sale queue, booking saga with compensation, three-party revenue split and escrow disbursement.")
    }

    System_Ext(razorpay,  "Razorpay",            "Payment gateway. Creates payment orders, captures funds, processes refunds. Sends HMAC-SHA256-signed webhook callbacks on all payment state transitions. Stubbed with real webhook flow in Phases 3-8.")
    System_Ext(gstn,      "GSTN / Tax Authority", "India GST system. Platform remits collected GST periodically. Automated integration is Phase 9+; manual process in v1.0.")
    System_Ext(email_sms, "Email / SMS Gateway",  "Transactional notification delivery: booking confirmations, waitlist offers, refund receipts, event updates. Stubbed in Phases 3-8.")

    Rel(customer,    stagepass, "Browse events · select seats · book · download QR · check in",     "HTTPS · WebSocket")
    Rel(organiser,   stagepass, "Create events · monitor live sales · view disbursements",           "HTTPS")
    Rel(venue_actor, stagepass, "Manage spaces · accept/reject bookings · view occupancy",           "HTTPS")
    Rel(admin,       stagepass, "KYC approvals · fraud review · platform config · trigger disbursements", "HTTPS")

    Rel(stagepass,   razorpay,  "Create payment order · initiate refund",                           "HTTPS REST")
    Rel(razorpay,    stagepass, "Payment result webhook (HMAC-SHA256 verified — NFR-REL-009)",       "HTTPS POST")
    Rel(stagepass,   email_sms, "Send transactional notifications",                                 "HTTPS REST (stub)")
    Rel(stagepass,   gstn,      "Periodic GST remittance",                                          "Manual / Batch (Phase 9+)")
```

---

## 4. C4 Level 2 — Container Diagram

### 4.1 Service Communication View

This view shows all 18 containers (services + API Gateway) with their technology stack
and NFR tier, the Kafka message bus, MinIO, HashiCorp Vault, all data stores,
and all architecturally significant communication lines labelled by protocol.

**Tier annotation key:** `[T1]` = 99.9% SLO · `[T2]` = 99.5% SLO · `[T3]` = 99.0% SLO
(→ ADR-001 §3.3, system prompt §12)

**Footnotes:**
> 📦 Full local Docker Compose stack target: **< 12 GB RAM** (NFR-PERF-043).
> New developer local setup target: **< 30 minutes from clone** (NFR-MAINT-004).
> Java cold start: < 30s · Node.js / Python cold start: < 10s (NFR-PERF-042).

```mermaid
flowchart TB
    %% ─────────────────────────────────────────────
    %% Node styling
    %% ─────────────────────────────────────────────
    classDef person       fill:#08427B,color:#fff,stroke:#052e56
    classDef ext_sys      fill:#666666,color:#fff,stroke:#444444
    classDef t1_svc       fill:#B22222,color:#fff,stroke:#8B0000
    classDef t2_svc       fill:#1a6b9a,color:#fff,stroke:#0f4c70
    classDef t3_svc       fill:#2e7d32,color:#fff,stroke:#1b5e20
    classDef db_pg        fill:#336791,color:#fff,stroke:#1a4060
    classDef db_mongo     fill:#4DB33D,color:#fff,stroke:#2e7d23
    classDef db_redis     fill:#C0392B,color:#fff,stroke:#96281b
    classDef db_es        fill:#FFC300,color:#000,stroke:#b38f00
    classDef db_ch        fill:#FF6B35,color:#fff,stroke:#cc4400
    classDef db_qdrant    fill:#7B2FBE,color:#fff,stroke:#5a1f9a
    classDef infra        fill:#555555,color:#fff,stroke:#333333
    classDef obs          fill:#999999,color:#fff,stroke:#666666

    %% ─────────────────────────────────────────────
    %% External actors
    %% ─────────────────────────────────────────────
    CUST(["👤 Customer\nBrowser / Mobile App"]):::person
    ORG(["👤 Organiser\nBrowser"]):::person
    VEN_ACT(["👤 Venue\nBrowser"]):::person
    ADMIN_USR(["👤 Admin\nBrowser"]):::person
    RAZ(["Razorpay\nPayment Gateway\n[External System]"]):::ext_sys
    ESMS(["Email / SMS Gateway\n[External System, Stub]"]):::ext_sys

    %% ─────────────────────────────────────────────
    %% PLATFORM BOUNDARY
    %% ─────────────────────────────────────────────
    subgraph PLATFORM["🎫  StagePass Platform  (stagepass-aiman GitHub Org)"]

        %% ── INGRESS ──────────────────────────────
        subgraph INGRESS["Ingress"]
            GW["API Gateway [T2]\nSpring Cloud Gateway · Java 21\nJWT validation · routing · rate limiting\ncircuit breaking · correlation ID injection"]:::t2_svc
            GW_REDIS[("gateway-cache\nRedis\nsliding-window rate limits\ncircuit breaker state")]:::db_redis
        end

        %% ── IDENTITY ─────────────────────────────
        subgraph IDENTITY["Identity"]
            AUTH["Auth Service [T1]\nSpring Boot 3 + Spring Security 6 · Java 21\nRS256 JWT issuance · JWKS endpoint\nbcrypt/argon2id · JTI revocation · RBAC"]:::t1_svc
            AUTH_PG[("auth-db\nPostgreSQL\nUsers · sessions\nrefresh tokens")]:::db_pg
            AUTH_REDIS[("auth-cache\nRedis\nJTI blocklist sorted set\nTTL per token exp")]:::db_redis
        end

        %% ── EVENT DOMAIN ─────────────────────────
        subgraph EVENT_DOM["Event Domain"]
            EVENT["Event Service [T2]\nNestJS 10 · TypeScript\nEvent CRUD · status machine\nElasticsearch sync · Kafka publisher"]:::t2_svc
            EVENT_MDB[("event-db\nMongoDB · Decimal128\nEvents · seating configs\npricing tiers · surge rules")]:::db_mongo
            VENUE["Venue Service [T2]\nNestJS 10 · TypeScript\nVenue CRUD · layout versions\nVenueBooking negotiation workflow"]:::t2_svc
            VENUE_MDB[("venue-db\nMongoDB · Decimal128\nVenues · seating layout versions\nVenueBooking records")]:::db_mongo
        end

        %% ── BOOKING DOMAIN ───────────────────────
        subgraph BOOKING_DOM["Booking Domain  (all T1 — 99.9% SLO)"]
            SEAT_INV["Seat Inventory Service [T1]\nSpring Boot 3 · Java 21\nAtom. Redis Lua SETNX holds\nCommitSeats · ReleaseSeats · ExtendHold\ngRPC server (seat_inventory/v1)"]:::t1_svc
            SEAT_PG[("seat-db\nPostgreSQL · NUMERIC(19,4)\nDurable seat states\nSERIALIZABLE on CommitSeats")]:::db_pg
            SEAT_REDIS[("seat-hold-cache\nRedis\nSETNX per seat · EX 600s\nTTL = auto-release (NFR-REL-004)")]:::db_redis

            BOOKING["Booking Service [T1]\nSpring Boot 3 · Java 21\n★ Saga Orchestrator (ADR-005)\nOutbox pattern · CQRS write model\nFlash sale routing via Redis flag"]:::t1_svc
            BOOKING_PG[("booking-write-db\nPostgreSQL · NUMERIC(19,4)\nBookings · RevenueSplit (immutable)\nOutbox table · SERIALIZABLE txns")]:::db_pg
            BOOKING_MDB[("booking-read-db\nMongoDB\nCQRS read model\nDenormalised booking history")]:::db_mongo

            PAYMENT["Payment Service [T1]\nQuarkus 3 · Java 21\nPayment orders · Razorpay webhook\nHMAC-SHA256 webhook verify\nRefund initiation · idempotency"]:::t1_svc
            PAYMENT_PG[("payment-db\nPostgreSQL\nPayment records\nWebhook receipts\nIdempotency keys (24h TTL)")]:::db_pg

            DISBURSEMENT["Disbursement Service [T1]\nSpring Boot 3 · Java 21\n★ Logical escrow owner (ADR-008)\nThree-party split scheduling\nGST tracking · immutable ledger"]:::t1_svc
            DISBURSEMENT_PG[("disbursement-db\nPostgreSQL · NUMERIC(19,4)\nDisbursementSchedule\nImmutable ledger entries\nAppend-only; no UPDATE on amounts")]:::db_pg
        end

        %% ── TICKETING & ENTRY ────────────────────
        subgraph TICKETING["Ticketing & Entry"]
            TICKET["Ticket Service [T2]\nFastify 4 · TypeScript\nHMAC-SHA256 token issuance\nPer-event secret (from Vault)\nQR generation · gRPC server (ticket/v1)"]:::t2_svc
            TICKET_PG[("ticket-db\nPostgreSQL\nTicket records\nIssuance log")]:::db_pg

            CHECKIN["Check-in Service [T2]\nFastify 4 · TypeScript\nSignature-only QR verify (hot path)\nOffline resilience: cached event keys\ngRPC client → Ticket (cold path only)"]:::t2_svc
            CHECKIN_PG[("checkin-db\nPostgreSQL\nCheck-in records\nAttendance log")]:::db_pg

            NOTIF["Notification Service [T2]\nNode.js 20 + Socket.IO 4 · TypeScript\nKafka consumer → WebSocket push\nRooms: userId · eventId\nHoriz. scaling via Redis adapter"]:::t2_svc
            NOTIF_REDIS[("notification-cache\nRedis\nSocket.IO Redis adapter\nRoom membership state")]:::db_redis

            WAITLIST["Waitlist Service [T3]\nNestJS 10 · TypeScript\nOrdered waitlist per event\nTime-limited offers\nOffer expiry → next member promotion"]:::t3_svc
            WAITLIST_REDIS[("waitlist-store\nRedis\nSorted sets: per-event queues\nO(log N) insert/pop")]:::db_redis
        end

        %% ── AI / ML SERVICES ─────────────────────
        subgraph AI_ML["AI / ML Services  (all T3 unless noted — fail-open)"]
            SEARCH["Search Service [T2]\nFastAPI 0.111 · Python 3.12\nKeyword · semantic · image search\np99 < 500ms (NFR-PERF-006)"]:::t2_svc
            SEARCH_ES[("search-index\nElasticsearch\nEvents · venues · organisers\nFull-text + faceted + geo")]:::db_es

            REC["Recommendation Service [T3]\nFastAPI 0.111 · Python 3.12\nPersonalised event recs\nRedis-cached inference\np99 < 500ms from cache (NFR-PERF-012)"]:::t3_svc
            REC_QDRANT[("recommendation-vectors\nQdrant\nUser + event embeddings\nCollab. filtering vectors")]:::db_qdrant
            REC_REDIS[("recommendation-cache\nRedis\nCached inference results")]:::db_redis

            CHATBOT["Chatbot Service [T3]\nFastAPI 0.111 · Python 3.12\nRAG over event catalog\nLangChain · streaming LLM\nFirst-token p99 < 2500ms (NFR-PERF-009)"]:::t3_svc
            CHATBOT_QDRANT[("chatbot-vectors\nQdrant\nEvent catalog embeddings\nfor RAG retrieval")]:::db_qdrant
            CHATBOT_REDIS[("chatbot-cache\nRedis\nConversation memory\nper user session")]:::db_redis

            FRAUD["Fraud Detection Service [T3]\nFastAPI 0.111 · Python 3.12\nVelocity scoring · scalping detection\nFAIL-OPEN: default LOW-risk (NFR-AVAIL-005)\nScore published < 10s (NFR-PERF-011)"]:::t3_svc
            FRAUD_PG[("fraud-db\nPostgreSQL\nFraud signals\nVelocity counters\nRisk score history")]:::db_pg

            ANALYTICS["Analytics Service [T3]\nFlask 3 + Gunicorn · Python 3.12\nOLAP: sales velocity · revenue\nattendance trends · GMV/min"]:::t3_svc
            ANALYTICS_CH[("analytics-db\nClickHouse\nTicket sales events\nRevenue aggregations\nAttendance time-series")]:::db_ch
        end

        %% ── SHARED INFRASTRUCTURE ────────────────
        subgraph SHARED["Shared Infrastructure"]
            KAFKA{{"Apache Kafka  (Strimzi · Kafka 3.x)\nAll async flows: saga commands · domain events\nflash-sale.hold-requests/results · AI scoring\nDLQ per topic (<topic>.dlq) · NFR-REL-007"}}:::infra
            MINIO["MinIO  (S3-compatible)\nEvent posters · Venue photos\nPDF ticket storage · signed URLs"]:::infra
            VAULT["HashiCorp Vault\nDB credentials · JWT RS256 private keys\nPer-event HMAC-SHA256 secrets\nRazorpay API keys · Service certs"]:::infra
        end

        %% ── OBSERVABILITY STACK ──────────────────
        subgraph OBS["Observability Stack  (all services emit to all three)"]
            PROM["Prometheus + Grafana\nRED + USE + business metrics\nSLO burn-rate alerts\nBookings/min · GMV/min · queue depth"]:::obs
            LOKI["Loki + Promtail\nStructured JSON logs\nCorrelation ID + Trace ID on every line\nNFR-OBS-001"]:::obs
            JAEGER["Jaeger + OTel Collector\nDistributed traces: HTTP · gRPC · Kafka\nContext propagated via W3C traceparent\nNFR-OBS-003"]:::obs
        end

    end
    %% ─── END PLATFORM ────────────────────────────

    %% ─────────────────────────────────────────────
    %% EXTERNAL → API GATEWAY
    %% ─────────────────────────────────────────────
    CUST      -- "HTTPS" --> GW
    ORG       -- "HTTPS" --> GW
    VEN_ACT   -- "HTTPS" --> GW
    ADMIN_USR -- "HTTPS" --> GW

    %% ─────────────────────────────────────────────
    %% API GATEWAY → ALL DOWNSTREAM SERVICES (REST/JWT)
    %% ADR-003 §3.2 — REST is default; Gateway validates JWT,
    %% injects X-User-Id · X-User-Role · X-Correlation-Id headers
    %% ─────────────────────────────────────────────
    GW -- "REST/JWT" --> AUTH
    GW -- "REST/JWT" --> EVENT
    GW -- "REST/JWT" --> VENUE
    GW -- "REST/JWT" --> SEAT_INV
    GW -- "REST/JWT" --> BOOKING
    GW -- "REST/JWT" --> PAYMENT
    GW -- "REST/JWT" --> DISBURSEMENT
    GW -- "REST/JWT" --> TICKET
    GW -- "REST/JWT" --> CHECKIN
    GW -- "REST/JWT" --> NOTIF
    GW -- "REST/JWT" --> WAITLIST
    GW -- "REST/JWT" --> SEARCH
    GW -- "REST/JWT" --> REC
    GW -- "REST/JWT" --> CHATBOT
    GW -- "REST/JWT" --> FRAUD
    GW -- "REST/JWT" --> ANALYTICS

    %% ─────────────────────────────────────────────
    %% WEBSOCKET: SERVER → BROWSER
    %% Notification Service is the ONLY service that pushes to browser.
    %% ADR-003 §3.4 — no other service touches WebSocket.
    %% ─────────────────────────────────────────────
    NOTIF -- "WebSocket / Socket.IO\nrooms: userId → booking updates · waitlist offers\nrooms: eventId → seat map state · live sales counters\np99 push < 1s (NFR-PERF-005)" --> CUST
    NOTIF -- "WebSocket / Socket.IO\nrooms: eventId → live sales counters · attendance" --> ORG
    NOTIF -- "WebSocket / Socket.IO\nrooms: eventId → attendance counts" --> VEN_ACT

    %% ─────────────────────────────────────────────
    %% gRPC PAIR 1 — BOOKING → SEAT INVENTORY
    %% ADR-003 §3.3.2 · ADR-006 §3.2
    %% Java 21 → Java 21 (same-language gRPC)
    %% Proto: stagepass-shared-contracts/proto/stagepass/seat_inventory/v1/seat_inventory.proto
    %% Methods: CheckSeatAvailability (read-only) · HoldSeats (atomic, all-or-nothing)
    %%          CommitSeats (HELD→BOOKED) · ReleaseSeats · ExtendHold (+300s on Payment failure)
    %% ─────────────────────────────────────────────
    BOOKING -- "gRPC  ◀▶  Protobuf\n━━━━━━━━━━━━━━━━\nCheckSeatAvailability (read-only)\nHoldSeats (Redis Lua atomic, all-or-nothing)\nCommitSeats · ReleaseSeats · ExtendHold\n━━━━━━━━━━━━━━━━\nADR-003 §3.3.2  Java→Java\nproto: seat_inventory/v1" --> SEAT_INV

    %% ─────────────────────────────────────────────
    %% gRPC PAIR 2 — CHECK-IN → TICKET
    %% ADR-003 §3.3.3 · ADR-002 §7 Q3
    %% TypeScript (Fastify) → TypeScript (Fastify)
    %% Proto: stagepass-shared-contracts/proto/stagepass/ticket/v1/ticket.proto
    %% Method: GetEventKey — fetches per-event HMAC secret on cold path ONLY.
    %% Hot path (99%+ of scans) is in-memory cached: ZERO network calls.
    %% ─────────────────────────────────────────────
    CHECKIN -- "gRPC  ◀▶  Protobuf\n━━━━━━━━━━━━━━━━\nGetEventKey (cold path only)\nFetches per-event HMAC-SHA256 secret\nfor QR signature verification\n━━━━━━━━━━━━━━━━\nADR-003 §3.3.3  TypeScript→TypeScript\nproto: ticket/v1" --> TICKET

    %% ─────────────────────────────────────────────
    %% REST: BOOKING → PAYMENT  (payment initiation)
    %% ADR-003 §3.2.1 — synchronous payment order creation only.
    %% Result arrives via Razorpay webhook → Kafka, not via this HTTP response.
    %% ─────────────────────────────────────────────
    BOOKING -- "REST\nCreatePaymentOrder\n(sync initiation only;\nresult via Razorpay webhook→Kafka)" --> PAYMENT

    %% ─────────────────────────────────────────────
    %% REST: CHATBOT → EVENT / BOOKING  (RAG context)
    %% ADR-003 §3.2.1 — JWT forwarded; services enforce RBAC normally
    %% ─────────────────────────────────────────────
    CHATBOT -- "REST · JWT forwarded\nEvent detail for RAG grounding" --> EVENT
    CHATBOT -- "REST · JWT forwarded\nBooking status for RAG grounding" --> BOOKING

    %% ─────────────────────────────────────────────
    %% EXTERNAL: RAZORPAY ↔ PAYMENT
    %% ─────────────────────────────────────────────
    PAYMENT -- "HTTPS REST\nCreate payment order\nInitiate refund (Razorpay API)" --> RAZ
    RAZ     -- "HTTPS POST\nWebhook callback\nHMAC-SHA256 verified (NFR-REL-009)\nPayment captured · failed · refunded" --> PAYMENT
    PAYMENT -- "HTTPS REST (stub)" --> ESMS

    %% ─────────────────────────────────────────────
    %% KAFKA — KEY PRODUCERS  (ADR-003 §3.4)
    %% ─────────────────────────────────────────────
    BOOKING     -- "booking.events\nbooking.commands\nbooking.refund-requested" --> KAFKA
    PAYMENT     -- "payment.events\n(payment.confirmed · payment.failed)" --> KAFKA
    EVENT       -- "event.events\n(event.published · event.cancelled)" --> KAFKA
    VENUE       -- "venue.events\n(venue.suspended)" --> KAFKA
    SEAT_INV    -- "seat.events (seat.released)\nflash-sale.hold-results" --> KAFKA
    TICKET      -- "ticket.events (ticket.issued)" --> KAFKA
    WAITLIST    -- "waitlist.events\nwaitlist.offer-expired" --> KAFKA
    FRAUD       -- "fraud.score-results" --> KAFKA
    ANALYTICS   -- "analytics.events (internal)" --> KAFKA

    %% ─────────────────────────────────────────────
    %% KAFKA — KEY CONSUMERS  (ADR-003 §3.4)
    %% Each consumer group is independent; fan-out is by design.
    %% ─────────────────────────────────────────────
    KAFKA -- "booking.confirmed\n→ issue HMAC ticket tokens" --> TICKET
    KAFKA -- "booking.confirmed\n→ create DisbursementSchedule\n(PENDING escrow entries)" --> DISBURSEMENT
    KAFKA -- "booking.* · seat.released · payment.*\nwaitlist.* · ticket.issued\n→ push to WebSocket rooms" --> NOTIF
    KAFKA -- "booking.pending\n→ score fraud risk (fail-open)" --> FRAUD
    KAFKA -- "all domain events\n→ ClickHouse OLAP ingestion" --> ANALYTICS
    KAFKA -- "event.published / event.updated\n→ index in Elasticsearch" --> SEARCH
    KAFKA -- "seat.released\n→ promote top waitlist member\n→ time-limited offer" --> WAITLIST
    KAFKA -- "flash-sale.hold-requests\n→ Redis Lua hold (flash sale consumer)\nADR-007 §3.5" --> SEAT_INV
    KAFKA -- "flash-sale.hold-results\npayment.events\n→ saga state advance" --> BOOKING

    %% ─────────────────────────────────────────────
    %% SERVICE → DATABASE (each DB owned by exactly ONE service)
    %% ─────────────────────────────────────────────
    GW          --- GW_REDIS
    AUTH        --- AUTH_PG
    AUTH        --- AUTH_REDIS
    EVENT       --- EVENT_MDB
    VENUE       --- VENUE_MDB
    SEAT_INV    --- SEAT_PG
    SEAT_INV    --- SEAT_REDIS
    BOOKING     --- BOOKING_PG
    BOOKING     --- BOOKING_MDB
    PAYMENT     --- PAYMENT_PG
    DISBURSEMENT --- DISBURSEMENT_PG
    TICKET      --- TICKET_PG
    CHECKIN     --- CHECKIN_PG
    NOTIF       --- NOTIF_REDIS
    WAITLIST    --- WAITLIST_REDIS
    SEARCH      --- SEARCH_ES
    REC         --- REC_QDRANT
    REC         --- REC_REDIS
    CHATBOT     --- CHATBOT_QDRANT
    CHATBOT     --- CHATBOT_REDIS
    FRAUD       --- FRAUD_PG
    ANALYTICS   --- ANALYTICS_CH

    %% ─────────────────────────────────────────────
    %% SHARED INFRASTRUCTURE CONNECTIONS
    %% ─────────────────────────────────────────────
    EVENT    -- "S3 API · store/serve\nevent poster images" --> MINIO
    TICKET   -- "S3 API · store PDF tickets\nsigned URL per ticket" --> MINIO
    VAULT -. "Secrets injected at startup\nvia Vault agent sidecar\n(Phases 3-8: env injection)" .-> AUTH
    VAULT -. "Secrets injected at startup" .-> BOOKING
    VAULT -. "Secrets injected at startup" .-> PAYMENT
    VAULT -. "Per-event HMAC secrets\nfetched at ticket issuance" .-> TICKET

    %% ─────────────────────────────────────────────
    %% OBSERVABILITY (all services → all three pillars)
    %% Shown as aggregate; individual service lines omitted for clarity
    %% ─────────────────────────────────────────────
    BOOKING  -- "OTel spans · RED metrics\nstructured logs + trace_id + correlationId" --> JAEGER
    BOOKING  -- "/metrics (Prometheus format)" --> PROM
    BOOKING  -- "JSON logs · Promtail scrape" --> LOKI
```

---

### 4.2 Data Store Ownership Table

Each service owns its database(s) exclusively. No database is shared between services.
This is the database-per-service pattern (ADR-001 §3, ADR-002 §3.2).

Money column types per ADR-004: PostgreSQL uses `NUMERIC(19,4)`, MongoDB uses `Decimal128`.

| Service | Tier | Primary DB | Type | Notes | Secondary Store | Type |
|---------|------|-----------|------|-------|-----------------|------|
| **API Gateway** | T2 | gateway-cache | Redis | Rate-limit counters (sliding window) · circuit breaker state | — | — |
| **Auth Service** | T1 | auth-db | PostgreSQL | Users · sessions · refresh tokens | auth-cache | Redis |
| **Event Service** | T2 | event-db | MongoDB | Events · seating configs · pricing tiers (Decimal128) · surge rules | — | — |
| **Venue Service** | T2 | venue-db | MongoDB | Venues · seating layout versions · VenueBooking records (Decimal128) | — | — |
| **Seat Inventory Service** | T1 | seat-db | PostgreSQL (NUMERIC 19,4) | Durable seat states · SERIALIZABLE isolation on CommitSeats | seat-hold-cache | Redis |
| **Booking Service** | T1 | booking-write-db | PostgreSQL (NUMERIC 19,4) | Bookings · RevenueSplit (immutable) · Outbox table | booking-read-db | MongoDB |
| **Payment Service** | T1 | payment-db | PostgreSQL | Payment records · webhook receipts · idempotency keys | — | — |
| **Disbursement Service** | T1 | disbursement-db | PostgreSQL (NUMERIC 19,4) | DisbursementSchedule · immutable ledger entries · append-only | — | — |
| **Ticket Service** | T2 | ticket-db | PostgreSQL | Ticket records · issuance log | — | — |
| **Check-in Service** | T2 | checkin-db | PostgreSQL | Check-in records · attendance log | — | — |
| **Notification Service** | T2 | notification-cache | Redis | Socket.IO Redis adapter for horizontal scaling | — | — |
| **Waitlist Service** | T3 | waitlist-store | Redis | Sorted sets: per-event waitlist queues | — | — |
| **Search Service** | T2 | search-index | Elasticsearch | Events · venues · organisers (full-text + faceted + geo) | — | — |
| **Recommendation Service** | T3 | recommendation-vectors | Qdrant | User + event embeddings for collaborative filtering | recommendation-cache | Redis |
| **Chatbot Service** | T3 | chatbot-vectors | Qdrant | Event catalog embeddings for RAG retrieval | chatbot-cache | Redis |
| **Fraud Detection Service** | T3 | fraud-db | PostgreSQL | Fraud signals · velocity counters · risk score history | — | — |
| **Analytics Service** | T3 | analytics-db | ClickHouse | Ticket sales events · revenue aggregations · attendance time-series | — | — |

---

## 5. Service Tier Map

NFR tier drives SLO, on-call response, and allowed failure modes. Every architectural
decision must account for the tier of each service it touches. (→ ADR-001 §3.3, system prompt §12)

| Service | Tier | SLO | Monthly Error Budget | Framework | Degradation Path |
|---------|------|-----|---------------------|-----------|-----------------|
| Auth Service | **T1** | 99.9% | 43 min | Spring Boot 3 · Java 21 | None — Auth down = no authenticated requests |
| Seat Inventory Service | **T1** | 99.9% | 43 min | Spring Boot 3 · Java 21 | Checkout fails fast < 2s; seat state preserved (NFR-AVAIL-002) |
| Booking Service | **T1** | 99.9% | 43 min | Spring Boot 3 · Java 21 | None — all bookings require Booking Service |
| Payment Service | **T1** | 99.9% | 43 min | Quarkus 3 · Java 21 | 503 + Retry-After + seat hold extended +5 min (NFR-AVAIL-003) |
| Disbursement Service | **T1** | 99.9% | 43 min | Spring Boot 3 · Java 21 | None — ledger integrity is a financial compliance requirement |
| API Gateway | **T2** | 99.5% | 3.6 hr | Spring Cloud Gateway · Java 21 | N/A — all traffic passes through gateway |
| Event Service | **T2** | 99.5% | 3.6 hr | NestJS 10 · TypeScript | Event reads cached; writes queue via Kafka |
| Venue Service | **T2** | 99.5% | 3.6 hr | NestJS 10 · TypeScript | Venue reads served from cache |
| Ticket Service | **T2** | 99.5% | 3.6 hr | Fastify 4 · TypeScript | Ticket downloads delayed; QR tokens pre-issued |
| Check-in Service | **T2** | 99.5% | 3.6 hr | Fastify 4 · TypeScript | Offline QR verify using cached event keys (NFR-AVAIL-006) |
| Notification Service | **T2** | 99.5% | 3.6 hr | Node.js 20 + Socket.IO | Fire-and-forget Kafka; bookings complete without notification (NFR-AVAIL-001) |
| Search Service | **T2** | 99.5% | 3.6 hr | FastAPI 0.111 · Python 3.12 | Keyword search degrades gracefully; event pages still load |
| Waitlist Service | **T3** | 99.0% | 7.3 hr | NestJS 10 · TypeScript | Waitlist join queued; no immediate offer sent |
| Recommendation Service | **T3** | 99.0% | 7.3 hr | FastAPI 0.111 · Python 3.12 | "Popular events" fallback shown; page does not error (NFR-AVAIL-004) |
| Chatbot Service | **T3** | 99.0% | 7.3 hr | FastAPI 0.111 · Python 3.12 | Chatbot widget hidden or shows "temporarily unavailable" |
| Fraud Detection Service | **T3** | 99.0% | 7.3 hr | FastAPI 0.111 · Python 3.12 | Booking proceeds as LOW-risk; fraud never blocks booking (NFR-AVAIL-005) |
| Analytics Service | **T3** | 99.0% | 7.3 hr | Flask 3 · Python 3.12 | Analytics dashboards stale; no operational impact |

---

## 6. Key Architectural Flows

These flows are summaries. The definitive sequence diagrams live at
`stagepass-docs/docs/architecture/sequences/`. Each flow traces to its governing ADR.

### 6.1 Normal Booking Path (Saga)

**Governing ADR:** ADR-005 (Booking Saga Pattern) · ADR-006 (Seat Inventory Concurrency)

```
Customer → API Gateway (HTTPS)
         → Booking Service [REST] — creates PENDING booking, checks flash-sale mode (Redis)
           → Seat Inventory Service [gRPC: HoldSeats] — Redis Lua SETNX, all-or-nothing
           ← Hold confirmed (Redis HELD + PostgreSQL async write)
           → Payment Service [REST: CreatePaymentOrder]
           ← Payment order ID returned (saga now PAYMENT_PENDING)
         ← HTTP 200 + idempotency key returned to browser
         
Razorpay → Payment Service [HTTPS webhook, HMAC-SHA256 verified]
         → Kafka: payment.events (payment.confirmed)
         → Booking Service consumes payment.confirmed
           → PostgreSQL: SERIALIZABLE txn — booking CONFIRMED + RevenueSplit written
           → Outbox: booking.confirmed event written atomically
         → Seat Inventory Service [gRPC: CommitSeats] — HELD → BOOKED in PostgreSQL
         → Kafka: booking.confirmed published from Outbox relay
           ├─→ Ticket Service: issue HMAC-SHA256 signed QR token
           ├─→ Disbursement Service: create DisbursementSchedule (PENDING escrow)
           └─→ Notification Service → WebSocket push to Customer
```

**Synchronous path p99: < 2s** (NFR-PERF-002)
**Full saga end-to-end p99: < 10s** (NFR-PERF-003)

### 6.2 Flash Sale Path

**Governing ADR:** ADR-007 (Flash Sale Queue Pattern) · ADR-006 §3.3

```
Booking Service detects flash-sale:mode:{eventId} in Redis (set by Seat Inventory)

Customer → API Gateway → Booking Service [REST]
         → Booking creates PENDING record (not SEATS_HELD yet)
         → Publishes to flash-sale.hold-requests Kafka topic (eventId partition key)
         ← HTTP 202 Accepted + bookingId + WebSocket subscription instructions

Seat Inventory Service (consumer group: seat-inventory-service-consumer)
  ← Consumes flash-sale.hold-requests at controlled drain rate (~83 msg/s/partition)
  → Redis Lua SETNX hold (same mechanism as normal path — ADR-006 applies)
  → Publishes flash-sale.hold-results (HOLD_SUCCEEDED or HOLD_FAILED)

Notification Service consumes flash-sale.hold-results
  → WebSocket push: queue position updates + final hold result to Customer browser

Booking Service consumes flash-sale.hold-results
  → On HOLD_SUCCEEDED: saga continues (payment initiation) — identical to normal path
  → On HOLD_FAILED: booking CANCELLED, customer notified
```

**Guarantee:** FIFO fairness per event via Kafka partition ordering. No oversell possible —
Redis Lua is the mutual exclusion regardless of path. (NFR-PERF-013: 10× RPS for 60s,
error rate < 0.5%)

### 6.3 Event Cancellation Cascade

**Governing ADR:** ADR-005 §4.3 · ADR-008 §3.7

```
Admin/Organiser → Event Service [REST] — event.status = CANCELLED
Event Service → Kafka: event.events (event.cancelled, eventId)

Booking Service (booking-service-consumer):
  ← Consumes event.cancelled
  → For each CONFIRMED booking on this event:
    → POST refund to Payment Service [REST]
    → booking status → CANCELLED
    → Kafka: booking.refund-requested

Payment Service: processes each refund via Razorpay
  → Kafka: refund.completed

Disbursement Service:
  ← Consumes refund.completed → CANCEL DisbursementSchedule (idempotent)
  → Ledger entry: REFUND_FULL (computed from stored RevenueSplit amounts — ADR-008 §3.3)

Notification Service:
  ← Consumes booking.cancelled → WebSocket push to each affected Customer
```

**Venue suspension cascade:** Admin suspends Venue → `venue.suspended` consumed by Event
Service → each affected upcoming event emits `event.cancelled` → triggers this cascade for
every event. Multi-level saga: Venue suspension → N event cancellations → N×M refund requests.

**Idempotency guaranteed:** Cancelling an already-CANCELLED booking returns success with no
duplicate refund (NFR-REL-011). DisbursementSchedule CANCEL is idempotent by `disbursementId`.

### 6.4 Disbursement Trigger

**Governing ADR:** ADR-008 §3.5

```
Event completes (Admin confirms via Organiser request or auto-trigger)
  → Disbursement Service: query DisbursementSchedule WHERE eventId AND status=PENDING
  → Validation gates: event status = COMPLETED · dispute window closed (default 48h)
  → For each DisbursementSchedule record (Venue + Organiser):
    disbursableAmount = stored split amount − SUM(refund ledger entries)
    → Initiate bank transfer (Razorpay Payout API — Phase 4 stub)
    → Ledger entry: DISBURSEMENT_COMPLETED (append-only, immutable)
  
Platform fee: retained in-platform. No payout record created. Revenue recognised.
```

---

## 7. Shared Infrastructure

### 7.1 HashiCorp Vault

Vault is the authoritative secrets store for the entire platform (NFR-SEC-008).
No secret — database password, API key, JWT private key, HMAC secret — is stored in code,
environment files, Helm values files, or container images.

| Secret Category | Stored In | Consumed By |
|-----------------|-----------|-------------|
| PostgreSQL / MongoDB / Redis credentials | Vault KV v2 | Each owning service (sidecar injection) |
| JWT RS256 private key | Vault KV v2 | Auth Service (sign); JWKS endpoint serves public key |
| Per-event HMAC-SHA256 secrets | Vault KV v2 (keyed by eventId) | Ticket Service (issuance); Check-in Service (verification via gRPC GetEventKey) |
| Razorpay API key + webhook secret | Vault KV v2 | Payment Service |
| Inter-service mTLS certificates | Vault PKI engine | All services (Phase 9 Istio) |

### 7.2 MinIO (S3-Compatible Object Storage)

MinIO serves as the local S3-compatible object store for all binary assets.
Bucket layout:

```
stagepass-assets/
├── events/{eventId}/poster.{jpg,webp}        # Event Service writes; CDN serves
├── venues/{venueId}/photos/{n}.{jpg,webp}    # Venue Service writes
└── tickets/{ticketId}/ticket.pdf             # Ticket Service writes; Customer downloads
```

In Phase 9 (cloud), MinIO is replaced with AWS S3 + CloudFront. All service code uses
the S3 API — no MinIO-specific SDK calls — to ensure zero migration cost.

### 7.3 Apache Kafka (Strimzi)

Kafka is the sole async communication mechanism for all inter-service events, saga commands,
and AI pipeline inputs. No direct service-to-service async calls exist outside Kafka.

Key operational facts:
- **DLQ naming convention:** `<original-topic>.dlq` for every topic (NFR-REL-007)
- **Prometheus alert:** fires when any DLQ depth > 0 for > 5 continuous minutes
- **Consumer group naming:** `<service-name>-consumer` (e.g., `booking-service-consumer`)
- **Offset commit:** manual, after successful processing — never auto-commit
- **Schema evolution:** backward-compatible only (optional fields; no removes; no type changes)
- **Flash sale topics:** `flash-sale.hold-requests` (24 partitions) · `flash-sale.hold-results` (12 partitions) — separate from `seat.commands` so flash sale queue depth is independently observable

---

## 8. Observability Architecture

All services emit to all three pillars. OpenTelemetry context is propagated across HTTP,
gRPC, and Kafka message headers (NFR-OBS-003).

| Pillar | Tooling | What it captures | Key NFR |
|--------|---------|-----------------|---------|
| **Metrics** | Prometheus + Grafana | RED per endpoint · USE per resource · business metrics (bookings/min, GMV/min, active holds, flash sale queue depth, waitlist depth, DLQ depth) | NFR-OBS-002, NFR-OBS-005 |
| **Logs** | Loki + Promtail | Structured JSON, one event/line. Every line includes: `trace_id`, `span_id`, `correlationId`, `bookingId`, `eventId`, `userId`, `service`, `timestamp UTC` | NFR-OBS-001 |
| **Traces** | Jaeger + OTel Collector | Distributed trace spanning API Gateway → Booking → Seat Inventory (gRPC) → Payment → Kafka → Ticket → WebSocket. W3C `traceparent` propagated across all transports. | NFR-OBS-003, NFR-OBS-004 |

**Correlation ID vs Trace ID (distinction from ADR-003 §3.6):**
- **Trace ID** (`traceparent`): generated by OTel; links all spans in a request tree; used in Jaeger.
- **Correlation ID** (`X-Correlation-Id`): UUID generated by API Gateway per inbound request; present in every log line; used to find all logs for a user action in Loki even for spans without an OTel trace (e.g., background jobs).

**SLO burn-rate alerts** are configured in Grafana for all T1 and T2 services.
Multi-window burn-rate alerts (1h/6h) fire before the error budget is exhausted.
Every alert has a linked runbook in `stagepass-docs/docs/runbooks/`.

---

## 9. NFR Compliance Notes

| NFR | How this architecture addresses it |
|-----|------------------------------------|
| NFR-PERF-001: Seat hold p99 < 500ms | Redis Lua SETNX — no DB lock in hold path. ADR-006 §3.2. |
| NFR-PERF-004: Check-in p99 < 200ms | Signature-only hot path; gRPC GetEventKey is cold path only. ADR-003 §3.3.3. |
| NFR-PERF-013: 10× RPS, 0 oversell | Flash sale Kafka funnel converts concurrency to throughput. ADR-007. |
| NFR-PERF-043: Full stack < 12 GB RAM | See service tier map; per-service memory budgets in ADR-002 §3.6. |
| NFR-REL-001: Idempotency keys | All write endpoints accept `Idempotency-Key` header; 24h Redis cache. |
| NFR-REL-004: Hold TTL = 600s | Redis `EX 600` is the enforcing mechanism — not application polling. |
| NFR-REL-005: Outbox pattern | Booking Service (and Disbursement) write Kafka events to Outbox table in same SERIALIZABLE txn. |
| NFR-REL-008: SERIALIZABLE isolation | Applied to RevenueSplit write + booking state machine in Booking Service; CommitSeats in Seat Inventory. ADR-004 §3.4.1. |
| NFR-REL-010: No float for money | PostgreSQL `NUMERIC(19,4)`, MongoDB `Decimal128`, Java `BigDecimal`, TS `decimal.js`, Python `decimal.Decimal`. ADR-004. |
| NFR-REL-012: Split amounts immutable | PostgreSQL trigger on `revenue_split` table raises exception on any `UPDATE`. ADR-008 §3.3.3. |
| NFR-AVAIL-001: Notification down → bookings still complete | Notification Service is fire-and-forget Kafka consumer. Saga never awaits WebSocket delivery. |
| NFR-AVAIL-005: Fraud down → booking proceeds | Fraud Detection is fail-open; default score = LOW-risk. Never in saga critical path. |
| NFR-AVAIL-006: Check-in offline | Per-event HMAC keys cached in Check-in Service process memory. QR verification is signature-only with zero network calls on hot path. |
| NFR-SEC-003: RBAC at service layer | `X-User-Role` header checked in each service, not only at Gateway. |
| NFR-SEC-005: QR tokens are per-event | HMAC secret is keyed by `eventId`; a token for Event A cannot verify against Event B's secret. |
| NFR-OBS-004: Trace a saga in < 2 min | OTel context propagated across HTTP, gRPC, and Kafka. bookingId is a span attribute and log field across all saga participants. |

---

## 10. Quick Self-Check

Answer these questions before starting any Phase 2+ implementation work.
If you cannot answer them clearly, re-read the relevant ADR before proceeding.

---

**Q1: A developer proposes that the Recommendation Service should query
the Event Service's MongoDB database directly for personalised results,
arguing it would avoid an extra network hop. What is wrong with this design,
and which ADR and pattern does it violate?**

Direct database access from Recommendation Service into Event Service's `event-db`
violates the database-per-service pattern (ADR-001 §3.3, ADR-002 §3.2). The Event Service
owns `event-db` exclusively. Recommendation Service must retrieve event data via the Event
Service's REST API (through the API Gateway), or by consuming domain events from Kafka
(e.g., `event.published` → build an internal projection in Qdrant). The anti-pattern is
"shared database" — when two services share a database, their schemas become coupled,
preventing independent deployment and testing (Newman, *Building Microservices*, Ch. 4).
The correct pattern is to have Recommendation Service build its own read model from
Kafka-consumed events, owned in its own `recommendation-vectors` Qdrant store.

---

**Q2: You are debugging a booking that was created CONFIRMED in the Booking
Service's PostgreSQL database, but the Customer has no ticket in the Ticket
Service and the Disbursement Service has no DisbursementSchedule entry. No
errors appear in any service's logs. What is the most likely cause, and which
component do you inspect first?**

The most likely cause is an Outbox relay failure: the `booking.confirmed` event was written
to the Outbox table atomically with the CONFIRMED state (NFR-REL-005, ADR-005 §3.5), but the
Outbox polling relay has not yet published the event to Kafka — or published it to a Kafka
topic that is not being consumed. First inspection: the `booking_outbox` table in
`booking-write-db` for rows with `status = PENDING` whose `created_at` is older than
expected (> 5 minutes). Second: check the DLQ for `booking.events.dlq` in Kafka for the
`bookingId`. Third: check the Booking Service Kafka producer metrics in Prometheus.
The pattern being observed is the "published but not delivered" failure mode that the
Outbox pattern is designed to handle via at-least-once delivery semantics.

---

**Q3: The gRPC pair Check-in → Ticket is described in the system prompt as
"Node.js to Java, teaches cross-language gRPC". But the container diagram
shows both Check-in and Ticket as TypeScript/Fastify. Which is authoritative,
and what is the actual learning objective of this gRPC pair?**

The container diagram (and ADR-002 §7 Q3) is authoritative. Both Check-in and Ticket
are TypeScript/Fastify per the framework assignment table. The "Node.js to Java" label
in the system prompt is a documentation artefact from an earlier design iteration.
The actual learning objective for the Check-in → Ticket pair is: TypeScript gRPC
client/server using `@grpc/grpc-js` and `@grpc/proto-loader` loading the shared
`.proto` file from `stagepass-shared-contracts`. This teaches the proto-loader toolchain,
runtime type generation, and the discipline of a shared contract that is language-agnostic.
The cross-language gRPC lesson (different stub generation toolchains) is fully satisfied
by the Booking → Seat Inventory pair (both Java, using `protobuf-maven-plugin`). If
the original intent was Java on Ticket Service, a superseding ADR must be filed before
changing the service assignment — it would affect the memory budget (ADR-002 §3.6) and
the phase timeline.

---

*End of HLD v1.0.0 — StagePass Platform High-Level Design*
