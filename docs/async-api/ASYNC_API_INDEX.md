<!--
  ASYNC_API_INDEX.md
  
  Repo:    stagepass-docs
  Path:    /docs/async-api/ASYNC_API_INDEX.md
  Version: 1.0.0
  Status:  Accepted
  Phase:   1 — Design
  
  Master index of all Kafka topics in the StagePass platform.
  Ground truth: ADR-003 §3.4.2 topic catalogue.
  
  Topic count: 22 base topics + 22 DLQ topics = 44 total Kafka topics.
  
  NOTE ON TASK-BRIEF COUNT:
  The Chat 13c task brief projected "26 topics." The actual count per
  ADR-003 §3.4.2 is 22 base topics. The discrepancy is a count error in
  the brief and has no impact on the YAML files or platform design.
  See Audit Findings section for the full list of discrepancies.
  
  References:
    ADR-003 §3.4.2   — Authoritative topic catalogue
    ADR-005 §3.9     — Booking saga Kafka interaction table
    NFR-REL-007      — Every topic has a .dlq variant
    NFR-OBS-005      — Business metrics in Grafana (queue depth, GMV)
  
  Changelog:
    1.0.0  2025-01-15  Initial — Phase 1 design (Chat 13c)
-->

# StagePass AsyncAPI Topic Index

**Version:** 1.0.0 | **Status:** Accepted | **Phase:** 1 — Design  
**Ground truth:** ADR-003 §3.4.2  
**Total base topics:** 22 | **Total DLQ topics:** 22 | **Grand total:** 44

---

## Topic Catalogue

| # | Topic | File | Partition Key | Partitions | Retention | Consumer Groups | DLQ Topic | Notes |
|---|-------|------|---------------|------------|-----------|-----------------|-----------|-------|
| 1 | `booking.commands` | `booking.yaml` | `bookingId` | 12 | 7 days | `booking-service-consumer` | `booking.commands.dlq` | |
| 2 | `booking.events` | `booking.yaml` | `bookingId` | 12 | 7 days | `notification-service-consumer`, `analytics-service-consumer`, `fraud-detection-service-consumer`, `disbursement-service-consumer`, `ticket-service-consumer` | `booking.events.dlq` | ⚠️ See AF-02: ADR-003 table lists only 3 consumers; YAML correctly implements 5 per ADR-005 §3.9 |
| 3 | `booking.refund-requested` | `disbursement.yaml` | `bookingId` | 12 | 7 days | `payment-service-consumer`, `disbursement-service-consumer` | `booking.refund-requested.dlq` | ⚠️ Booking-domain topic deferred to disbursement.yaml (AF-03). See notes below. |
| 4 | `payment.events` | `payment.yaml` | `paymentOrderId` | 6 | 7 days | `booking-service-consumer`, `disbursement-service-consumer` | `payment.events.dlq` | |
| 5 | `seat.commands` | `seat.yaml` | `eventId` | 24 | 3 days | `seat-inventory-service-consumer` | `seat.commands.dlq` | |
| 6 | `seat.state-changes` | `seat.yaml` | `eventId` | 24 | 1 day | `notification-service-consumer`, `booking-service-consumer` | `seat.state-changes.dlq` | |
| 7 | `ticket.commands` | `ticket.yaml` | `bookingId` | 12 | 3 days | `ticket-service-consumer` | `ticket.commands.dlq` | |
| 8 | `ticket.events` | `ticket.yaml` | `ticketId` | 12 | 30 days | `notification-service-consumer`, `check-in-service-consumer` | `ticket.events.dlq` | |
| 9 | `ticket.checked-in` | `ticket.yaml` | `ticketId` | 12 | 90 days | `analytics-service-consumer`, `notification-service-consumer` | `ticket.checked-in.dlq` | |
| 10 | `notification.commands` | `notification.yaml` | `userId` | 12 | 1 day | `notification-service-consumer` | `notification.commands.dlq` | |
| 11 | `event.commands` | `event.yaml` | `eventId` | 12 | 7 days | `event-service-consumer` | `event.commands.dlq` | |
| 12 | `event.events` | `event.yaml` | `eventId` | 12 | 30 days | `search-service-consumer`, `recommendation-service-consumer`, `analytics-service-consumer`, `booking-service-consumer`, `seat-inventory-service-consumer` | `event.events.dlq` | |
| 13 | `venue.events` | `venue.yaml` | `venueId` | 6 | 30 days | `event-service-consumer`, `search-service-consumer` | `venue.events.dlq` | |
| 14 | `disbursement.commands` | `disbursement.yaml` | `disbursementId` | 6 | 30 days | `disbursement-service-consumer` | `disbursement.commands.dlq` | |
| 15 | `disbursement.events` | `disbursement.yaml` | `disbursementId` | 6 | 90 days | `analytics-service-consumer`, `notification-service-consumer` | `disbursement.events.dlq` | |
| 16 | `fraud.score-requests` | `fraud.yaml` | `bookingId` | 12 | 3 days | `fraud-detection-service-consumer` | `fraud.score-requests.dlq` | |
| 17 | `fraud.score-results` | `fraud.yaml` | `bookingId` | 12 | 7 days | `booking-service-consumer`, `admin-notification-consumer` | `fraud.score-results.dlq` | |
| 18 | `waitlist.commands` | `waitlist.yaml` | `eventId` | 6 | 7 days | `waitlist-service-consumer` | `waitlist.commands.dlq` | |
| 19 | `waitlist.events` | `waitlist.yaml` | `eventId` | 6 | 7 days | `notification-service-consumer` | `waitlist.events.dlq` | |
| 20 | `waitlist.offer-expired` | `waitlist.yaml` | `customerId` | 6 | 1 day | `waitlist-service-consumer` | `waitlist.offer-expired.dlq` | Self-loop: Waitlist Service is both publisher and consumer |
| 21 | `flash-sale.hold-requests` | `flash-sale.yaml` | `eventId` | 24 | 1 day | `seat-inventory-service-consumer` | `flash-sale.hold-requests.dlq` | |
| 22 | `flash-sale.hold-results` | `flash-sale.yaml` | `customerId` | 12 | 1 day | `notification-service-consumer`, `booking-service-consumer` | `flash-sale.hold-results.dlq` | |

---

## File-to-Domain Mapping

| File | Domain | Base Topics | DLQ Topics |
|------|--------|-------------|------------|
| `booking.yaml` | Booking | `booking.commands`, `booking.events` | `booking.commands.dlq`, `booking.events.dlq` |
| `disbursement.yaml` | Disbursement + Booking (deferred) | `disbursement.commands`, `disbursement.events`, `booking.refund-requested` | `disbursement.commands.dlq`, `disbursement.events.dlq`, `booking.refund-requested.dlq` |
| `event.yaml` | Event | `event.commands`, `event.events` | `event.commands.dlq`, `event.events.dlq` |
| `flash-sale.yaml` | Flash Sale | `flash-sale.hold-requests`, `flash-sale.hold-results` | `flash-sale.hold-requests.dlq`, `flash-sale.hold-results.dlq` |
| `fraud.yaml` | Fraud | `fraud.score-requests`, `fraud.score-results` | `fraud.score-requests.dlq`, `fraud.score-results.dlq` |
| `notification.yaml` | Notification | `notification.commands` | `notification.commands.dlq` |
| `payment.yaml` | Payment | `payment.events` | `payment.events.dlq` |
| `seat.yaml` | Seat | `seat.commands`, `seat.state-changes` | `seat.commands.dlq`, `seat.state-changes.dlq` |
| `ticket.yaml` | Ticket | `ticket.commands`, `ticket.events`, `ticket.checked-in` | `ticket.commands.dlq`, `ticket.events.dlq`, `ticket.checked-in.dlq` |
| `venue.yaml` | Venue | `venue.events` | `venue.events.dlq` |
| `waitlist.yaml` | Waitlist | `waitlist.commands`, `waitlist.events`, `waitlist.offer-expired` | `waitlist.commands.dlq`, `waitlist.events.dlq`, `waitlist.offer-expired.dlq` |

---

## DLQ Reference

All DLQ topics follow these conventions (NFR-REL-007, ADR-003 §3.4.1):

- **Naming:** `<original-topic>.dlq`
- **Retention:** 14 days (investigation + replay window)
- **Partition count:** Mirrors the source topic
- **Additional header:** `x-failure-reason: <exception class and message>`
- **Alert:** `KafkaDLQDepthNonZero` fires when depth > 0 for > 5 continuous minutes
- **Runbook:** `/docs/runbooks/kafka-dlq-depth.md`
- **Consumer group naming:** `<original-topic-slug>-dlq-consumer`

---

## Audit Findings

Three findings were identified during the Chat 13c consistency audit. None require changes to existing YAML files.

---

### AF-01 — Topic Count Discrepancy (Task Brief)

| Field | Value |
|-------|-------|
| **Severity** | Informational |
| **Source** | Chat 13c task brief |
| **Finding** | Task brief states "all 26 topics from ADR-003 §3.4.2." The actual count of non-DLQ topics in ADR-003 §3.4.2 is **22**. |
| **Root cause** | Count error in the task brief. No additional topics exist in ADR-003 §3.4.2 beyond the 22 listed. |
| **Action required** | None. YAML files and this index accurately reflect the 22-topic catalogue. |

---

### AF-02 — `booking.events` Consumer Group Incompleteness in ADR-003 §3.4.2

| Field | Value |
|-------|-------|
| **Severity** | Documentation debt — ADR-003 amendment required |
| **Affected file** | `ADR-003-service-communication-patterns.md` |
| **Finding** | ADR-003 §3.4.2 booking domain table lists `booking.events` consumers as: `notification-service-consumer`, `analytics-service-consumer`, `fraud-detection-service-consumer`. It omits `disbursement-service-consumer` and `ticket-service-consumer`. |
| **Authoritative source** | ADR-005 §3.9 explicitly states both services consume `booking.events`: Disbursement Service (schedules post-event payouts on `booking.confirmed`), Ticket Service (issues signed QR tokens on `booking.confirmed`). |
| **YAML files** | `booking.yaml` (Chat 13a) correctly implements all **5** consumer groups per ADR-005 §3.9. This index reflects the correct 5-group definition. |
| **Action required** | ADR-003 §3.4.2 booking domain table must be amended to add `disbursement-service-consumer` and `ticket-service-consumer` to the `booking.events` row. This is a documentation amendment, not an implementation change. Track as Phase 1 cleanup issue in `stagepass-docs`. |

---

### AF-03 — `booking.refund-requested` Cross-File Placement

| Field | Value |
|-------|-------|
| **Severity** | Informational (by design) |
| **Affected topic** | `booking.refund-requested` |
| **Finding** | This booking-domain topic (ADR-003 §3.4.2, booking domain table, row 3) is defined in `disbursement.yaml` rather than `booking.yaml`. |
| **Rationale** | `booking.refund-requested` was explicitly deferred from Chat 13a (booking domain) to Chat 13b (disbursement domain) because its primary consumers are `payment-service-consumer` and `disbursement-service-consumer`, and the refund/disbursement interaction is the primary context for this message. The `disbursement.yaml` file header documents this placement decision. |
| **Action required** | None. The placement is intentional and documented. This index entry (row 3) and the cross-reference note serve as the permanent documentation of this cross-file relationship. |

---

## Cross-Reference: Topics Consumed by Multiple Domains

This table maps which services consume which topics, for quick operational reference.

| Consumer Service | Topics Consumed |
|-----------------|-----------------|
| `booking-service-consumer` | `payment.events`, `event.events`, `seat.state-changes` (reconciliation), `fraud.score-results`, `flash-sale.hold-results`, `booking.commands` |
| `notification-service-consumer` | `booking.events`, `ticket.events`, `ticket.checked-in`, `notification.commands`, `disbursement.events`, `waitlist.events`, `flash-sale.hold-results`, `seat.state-changes` |
| `analytics-service-consumer` | `booking.events`, `ticket.checked-in`, `disbursement.events`, `event.events` |
| `disbursement-service-consumer` | `booking.events`, `booking.refund-requested`, `payment.events`, `disbursement.commands` |
| `ticket-service-consumer` | `booking.events`, `ticket.commands` |
| `fraud-detection-service-consumer` | `fraud.score-requests` |
| `seat-inventory-service-consumer` | `seat.commands`, `event.events`, `flash-sale.hold-requests` |
| `event-service-consumer` | `event.commands`, `venue.events` |
| `search-service-consumer` | `event.events`, `venue.events` |
| `recommendation-service-consumer` | `event.events` |
| `check-in-service-consumer` | `ticket.events` |
| `waitlist-service-consumer` | `waitlist.commands`, `waitlist.offer-expired` |
| `admin-notification-consumer` | `fraud.score-results` |