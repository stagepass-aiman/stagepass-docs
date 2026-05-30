# PRD — StagePass: AI-Enhanced Multi-Party Event Ticketing Platform

## Document Header

| Field        | Value                                                                                  |
|--------------|----------------------------------------------------------------------------------------|
| Document ID  | PRD                                                                                    |
| Version      | 1.0.0                                                                                  |
| Status       | Accepted                                                                               |
| Author       | StagePass Engineering                                                                  |
| Created      | 2025-05-15                                                                             |
| Last Updated | 2025-05-15                                                                             |
| Repo         | stagepass-docs                                                                         |
| Path         | /PRD.md                                                                                |
| Phase        | Phase 1 — Requirements & Design                                                        |
| Traces To    | NFR.md · ADR-001 · ADR-002 · ADR-003 · ADR-004 · ADR-005 · ADR-006 · ADR-007 · ADR-008 · HLD.md |

### Change Log

| Version | Date       | Author                 | Summary                  |
|---------|------------|------------------------|--------------------------|
| 1.0.0   | 2025-05-15 | StagePass Engineering  | Initial accepted version |

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Problem Statement](#2-problem-statement)
3. [Goals](#3-goals)
4. [Non-Goals and Scope Boundaries](#4-non-goals-and-scope-boundaries)
5. [Market Context](#5-market-context)
6. [Actors and Roles](#6-actors-and-roles)
7. [Core Domain Model](#7-core-domain-model)
8. [Feature Requirements](#8-feature-requirements)
   - 8.1 [Customer](#81-customer)
   - 8.2 [Organiser](#82-organiser)
   - 8.3 [Venue](#83-venue)
   - 8.4 [Admin](#84-admin)
   - 8.5 [AI Features](#85-ai-features)
   - 8.6 [Platform-Wide and Cross-Cutting](#86-platform-wide-and-cross-cutting)
   - 8.7 [Resilience and Graceful Degradation](#87-resilience-and-graceful-degradation)
9. [Regulatory and Compliance Requirements](#9-regulatory-and-compliance-requirements)
10. [Observability Requirements](#10-observability-requirements)
11. [Success Metrics](#11-success-metrics)
12. [Constraints and Assumptions](#12-constraints-and-assumptions)
13. [Open Questions and Deferred Decisions](#13-open-questions-and-deferred-decisions)
14. [Glossary](#14-glossary)

---

## 1. Purpose

This document defines the product requirements for **StagePass**: a production-grade, AI-enhanced, multi-party event ticketing marketplace operating in the Indian market. StagePass connects three supply-side actors — Venues (physical spaces), Organisers (event creators), and the Platform (StagePass itself) — with one demand-side actor: Customers who discover and attend events.

This PRD is the authoritative statement of what StagePass must do and why. It drives all subsequent architectural decisions (ADR-001 through ADR-008), non-functional requirements (NFR.md), the high-level design (HLD.md), and service-level API contracts (OpenAPI and AsyncAPI specs). Every ADR and service design decision must trace back to a requirement expressed here.

---

## 2. Problem Statement

The Indian live event market — concerts, comedy shows, theatre, IPL, Pro Kabaddi, domestic cricket — suffers from a structural problem: the tools that connect Venues, Organisers, and fans are fragmented, opaque, and operationally fragile.

**For Venues:** Existing booking processes are manual (calls, emails, spreadsheets). Revenue sharing is agreed informally and tracked outside the platform. Venues have no real-time visibility into how their spaces are being sold and used.

**For Organisers:** Listing a new event requires negotiating with Venues offline, uploading to multiple ticketing platforms independently, managing customer communications manually, and reconciling revenue through slow, opaque bank transfers weeks after an event.

**For Customers:** Discovery is keyword-search-only. No personalisation. No intelligent recommendations. Seat selection is binary — available or not — with no hold mechanism, leading to checkout failures and lost purchases. Flash sale events (IPL opening matches, major concerts) routinely crash platforms, leaving genuine fans without tickets while scalpers exploit queue failures.

**For everyone:** Cancellation and refund handling is a trust-destroying experience. When an event is cancelled, customers do not know if or when they will be refunded. Organisers do not know which refunds have been processed. Venues have no clear view of their liability.

StagePass solves this by providing a single, integrated platform with: a structured Venue-Organiser negotiation workflow; a fair, queue-based flash sale mechanism; a reservation-based seat holding system; a transparent, automated three-party revenue split; a disciplined cancellation and refund saga; and AI-powered discovery and operations tooling that treats intelligence as a first-class feature, not an afterthought.

---

## 3. Goals

**G-01 — Venue Marketplace**
Venues can list their physical spaces with full seating layout configurations. Organisers can discover, request, and confirm Venue bookings entirely within the platform. No offline negotiation is required after initial contact.

**G-02 — Structured Event Creation**
Organisers can create, configure, and publish events only after Venue confirmation and KYC verification. Event configuration includes seating sections, ticket pricing tiers, surge pricing rules, cancellation policy, and event metadata.

**G-03 — Fair, High-Concurrency Ticket Booking**
Customers can discover events, select specific seats on an interactive real-time seat map, and complete checkout. Under flash sale load (up to 10× normal RPS), the platform must not oversell, must not crash, and must preserve fairness through a queue-based funnel. `[NFR-PERF-013]`

**G-04 — Reservation-Based Seat Holding**
Selected seats are held exclusively for a configurable window (default: 10 minutes) while the customer completes checkout. Hold expiry auto-releases seats. Payment completion commits the hold to BOOKED. `[NFR-REL-004]`

**G-05 — Transparent Three-Party Revenue Split**
Every ticket sale automatically splits revenue between Platform (fee), Venue (negotiated share), and Organiser (remainder). Split amounts are immutable once created. Funds are held in escrow and disbursed after event completion. `[NFR-REL-012]`

**G-06 — Disciplined Cancellation and Refund Handling**
Customer-initiated cancellations are governed by Organiser-defined policies, enforced by the platform. Organiser-initiated event cancellations trigger automated refund sagas for all affected bookings. Every refund is traceable to its originating booking and split record.

**G-07 — AI-Powered Discovery and Operations**
Personalised recommendations, semantic and image-based event search, a RAG chatbot over the live event catalogue, demand prediction, surge pricing guidance, fraud detection, and no-show prediction are first-class platform features, not optional add-ons.

**G-08 — Real-Time Seat Map and Operations**
Seat state changes (AVAILABLE → HELD → BOOKED → RELEASED) are pushed to all connected clients within one second. Organisers see live ticket sales counters. Venue staff see live attendance counts during events. `[NFR-PERF-005]`

**G-09 — QR-Based Contactless Check-In**
Every confirmed ticket carries a cryptographically signed QR token. Verification at the gate is signature-only — no network call required — enabling offline resilience. `[NFR-PERF-004]` `[NFR-AVAIL-006]`

**G-10 — Production-Grade Observability and Security**
The platform emits structured logs, RED metrics, and distributed traces from day one. Every service defends against the OWASP Top 10. All secrets are managed in Vault. `[NFR-OBS-001]` `[NFR-SEC-001]`

---

## 4. Non-Goals and Scope Boundaries

The following are explicitly **out of scope** for StagePass v1.0. Each exclusion is deliberate. Where a future phase might include it, a note is provided.

| Out of Scope | Rationale |
|---|---|
| **Streaming / virtual events** | No video infrastructure in scope. StagePass is a physical-attendance ticketing platform. |
| **League or governing body as an actor** | Sports events are created by Organisers promoting matches, not by BCCI, ISL, or any league. StagePass does not model governing-body authority. |
| **Resale / secondary ticketing market** | No peer-to-peer ticket transfers or resale listings. Anti-scalping controls are enforced; resale facilitation is not provided. |
| **Real payment processing** | Payment integration is mocked throughout. No real money is collected or disbursed. Razorpay is the reference implementation for architecture design only. |
| **Real KYC document verification** | KYC verification is modelled as a state machine with correct transitions. Document verification is a stub that transitions state after a configurable delay. |
| **Multi-currency support** | INR is the sole currency. Multi-currency is a future consideration after Indian market validation. |
| **Physical merchandise or add-ons** | No merchandise, food pre-orders, or parking alongside ticket purchases. |
| **Mobile native applications** | The frontend is a responsive web application. iOS and Android native apps are not in scope. |
| **Social features** | No friend activity feeds, group booking coordination, or shared wishlists. |
| **Subscription or membership tiers for Customers** | No loyalty points, annual passes, or priority access programmes. |
| **Fully automated AI-driven dynamic pricing** | The AI surfaces demand signals. Pricing decisions are made by Organisers via configured rules, not by the platform autonomously. |
| **Organiser-to-Organiser collaboration** | Co-promoter models (multiple Organisers on a single event) are not modelled. One event has exactly one Organiser. |
| **Venue-managed seat blocking for VIP/hospitality** | Admin can block seats globally. Venue cannot block individual seats independently. Future consideration. |
| **International regulatory compliance** | GDPR, PSD2, UK Consumer Rights Act are not in scope. DPDP Act and Indian GST are the operative frameworks. |
| **Terraform cloud deployment** | Terraform modules are designed in Phase 9. Cloud deployment is not a Phase 1–8 concern. |

---

## 5. Market Context

### 5.1 Geography and Regulatory Frame

StagePass operates in **India**. All product decisions must be compatible with the following regulatory environment:

**Goods and Services Tax (GST)**
Entertainment services are taxable under GST. The applicable rate varies by event type and state. StagePass must display GST-inclusive pricing to Customers, generate GST-compliant invoices, and ensure the revenue split logic accounts for GST obligations correctly. Detailed GST treatment is deferred to the disbursement ADR (ADR-008).

**Digital Personal Data Protection Act, 2023 (DPDP Act)**
StagePass collects personally identifiable information including names, phone numbers, email addresses, and payment-linked identifiers. The DPDP Act requires: a clear and specific consent notice before collection; the right to access and correction; the right to erasure; a data fiduciary registration if processing data at significant scale. StagePass must implement consent management at registration and at any point where new data categories are collected.

**RBI Payment Aggregator Guidelines**
Any platform that collects funds from Customers and disburses to third parties (Venues, Organisers) operates in the payment aggregator space. Since StagePass uses a licensed aggregator (Razorpay, stubbed) rather than collecting funds directly, it is not itself a PA. However, the Organiser and Venue onboarding and KYC requirements are designed to be compatible with what a licensed PA would require.

**Indian Contract Act and Consumer Protection Act**
Refund obligations on event cancellation are legally enforceable. The cancellation and refund saga design must ensure that Customer refunds are initiated within the timeframe required by the Consumer Protection (E-Commerce) Rules, 2020 (currently within the refund period stated at time of purchase, and no later than 14 days for platform-cancelled events).

### 5.2 Competitive Landscape

| Platform | Strengths | Weaknesses StagePass Addresses |
|---|---|---|
| **BookMyShow** | Market leader, brand recognition, deep inventory | No real-time seat map, poor flash sale fairness, opaque refunds, no AI discovery |
| **Paytm Insider** | UPI-native payments, music focus | Limited venue tooling, no Organiser-Venue negotiation workflow |
| **TicketNew** | South India presence | Regional only, no AI features, limited analytics |
| **Eventbrite** | Organiser UX, analytics | Not India-native, poor UPI integration, no sports-scale seat maps |

StagePass's differentiators: **fair flash sale queuing**, **real-time seat map**, **AI-first discovery**, **transparent three-party split with audit trail**, and **structured Venue-Organiser negotiation**.

---

## 6. Actors and Roles

StagePass has four actors. Permissions are **disjoint** — no actor can perform actions belonging to another actor's domain. Permission enforcement happens at the service layer, not only at the gateway or UI. `[NFR-SEC-003]`

### 6.1 Customer

A registered individual who discovers and purchases tickets to attend events.

**Core capabilities:**
- Register and manage their account and preferences
- Discover events through keyword search, semantic search, image search, and AI recommendations
- Select specific seats on an interactive seat map and hold them during checkout
- Complete ticket purchase via UPI, card, or net banking (all mocked)
- View, download, and share their tickets (PDF and QR code)
- Join the waitlist for sold-out events
- Cancel tickets per the Organiser's cancellation policy and receive refunds
- Receive real-time notifications (booking confirmations, waitlist offers, event updates, refunds)
- Interact with the AI chatbot to query event information and booking status

**Constraints:**
- Cannot access Organiser, Venue, or Admin functionality under any circumstances
- Cannot transfer tickets to another Customer (no resale)
- Cannot modify a confirmed booking; must cancel and rebook

### 6.2 Organiser

A verified business entity that creates and manages events at Venues. Organisers are self-registered and must complete KYC before publishing events. KYC verification is modelled as a state machine; document validation is a stub.

**Core capabilities:**
- Register, complete KYC verification, and manage their Organiser profile
- Discover and request available Venues for specific dates and configurations
- Negotiate and confirm Venue bookings (status-driven workflow)
- Create events in confirmed Venues: configure sections, seating, pricing tiers, surge rules, cancellation policy, and metadata
- Publish, postpone, or cancel events (each with distinct consequences)
- Monitor live ticket sales, revenue, and seat map status during sales periods
- Monitor live attendance counts during events
- View disbursement history and pending escrow balances
- Receive AI-generated demand predictions and surge pricing guidance
- Manage customer refund disputes escalated by Admin

**Constraints:**
- Cannot create events in unconfirmed Venues
- Cannot publish events before KYC is VERIFIED
- Cannot access another Organiser's events, revenue data, or customer information `[NFR-SEC-004]`
- Cannot modify a cancellation policy after the first ticket is sold
- Cannot change seat pricing after the first ticket for that price tier is sold (new tiers can be added)

### 6.3 Venue

A registered entity that owns and manages physical event spaces. Venues list their spaces, respond to booking requests from Organisers, and configure seating layouts.

**Core capabilities:**
- Register and manage their Venue profile (name, location, facilities, photos)
- Create and version seating layout configurations (sections, rows, seats, capacity, accessibility zones)
- View and respond to booking requests (ACCEPT or REJECT with reason)
- View a booking calendar showing all confirmed events and their status
- View per-event attendance data during events (live, for their own spaces)
- View revenue share history and pending escrow balances for their spaces
- Suspend availability for specific date ranges (maintenance, private hire)

**Constraints:**
- Cannot create or manage events (only accept/reject Organiser requests)
- Cannot access Organiser revenue data or Customer information `[NFR-SEC-004]`
- Cannot modify a confirmed booking's seating layout (changes require new booking request)
- Venue suspension triggers the platform-level Venue suspension cascade (event cancellation fan-out for all upcoming events)

### 6.4 Admin

StagePass platform operators. Admin has elevated permissions across the platform but operates through a dedicated Admin dashboard, not through Customer/Organiser/Venue interfaces.

**Core capabilities:**
- Review and action KYC submissions (approve, reject with reason, request more information)
- Suspend or reinstate Organisers and Venues (with cascade consequences)
- Manage event categories and taxonomy
- Review fraud alerts raised by the Fraud Detection Service and action them (hold booking, flag account, escalate)
- View platform-wide health metrics, DLQ depths, SLO burn rates
- Configure platform-level settings: Platform fee percentage, seat hold window duration, maximum surge multiplier, per-event QR secret rotation
- View and export audit logs for compliance and dispute resolution
- Process escalated Customer refund disputes
- Manage the ML model deployment lifecycle (promote/rollback models in the recommendation and fraud detection pipelines)

**Constraints:**
- Cannot book tickets as a Customer (must use a separate Customer account)
- Cannot modify Organiser or Venue revenue data directly; all financial actions go through the Disbursement Service audit trail
- All Admin actions are logged with the Admin's user ID, timestamp, and reason `[NFR-OBS-001]`

---

## 7. Core Domain Model

### 7.1 Key Entities

**Venue**
A physical space owned by a Venue actor. Has one or more SeatingLayouts. A SeatingLayout is versioned — once seats are sold against a layout version, that version is immutable. A Venue has a negotiated Revenue Share percentage that applies to all events hosted in it.

**SeatingLayout**
A hierarchical configuration: Venue → Section → Row → Seat. Each Seat has a unique identifier, a type (STANDARD, PREMIUM, ACCESSIBLE), and coordinates for seat map rendering. Layouts are stored in MongoDB (flexible schema) and versioned independently. Stadium-scale layouts may contain 50,000+ seat records.

**VenueBooking**
A time-bound agreement between an Organiser and a Venue for exclusive use of the Venue on specific dates. States: REQUESTED → ACCEPTED | REJECTED → CONFIRMED (on event creation). A confirmed VenueBooking creates a jointly-owned event context.

**Event**
Created by an Organiser against a confirmed VenueBooking. An Event carries: metadata (name, description, category, poster image), a snapshot of the seating layout at creation time, one or more PricingTiers, a CancellationPolicy, SurgePricingRules, and a state machine. Event states: DRAFT → PUBLISHED → ON_SALE → SOLD_OUT | POSTPONED | CANCELLED → COMPLETED.

**PricingTier**
A named price bracket applied to one or more sections (e.g., "Platinum — Section A: ₹5,000", "Gold — Section B/C: ₹2,500", "General Standing: ₹800"). Once the first ticket is sold in a tier, the tier price is immutable. New tiers can be added while seats remain unsold.

**SurgePricingRule**
An Organiser-defined rule evaluated continuously against seat availability thresholds. Example: "If remaining seats in Platinum tier < 20%, multiply base price by 1.15." Rules are evaluated at seat hold creation time. The price at hold creation is locked for that hold's duration. `[NFR-REL-004]`

**CancellationPolicy**
An Organiser-defined policy attached to an Event. Defines refund entitlements by time-to-event at cancellation. Example: "Full refund if cancelled > 48 hours before start; 50% refund if 24–48 hours before; no refund within 24 hours." Policy is immutable after the first ticket is sold.

**Seat**
A unique, non-fungible inventory item. Seat 12B in Section A at Wankhede is not interchangeable with Seat 12C. Every seat has an independent state: AVAILABLE, HELD, BOOKED, BLOCKED. State transitions are atomic operations enforced by Redis. `[NFR-PERF-001]`

**SeatHold**
A time-limited reservation created when a Customer selects a seat. Holds a specific seatId, customerId, eventId, price-at-hold-time (surge-adjusted), and an expiry timestamp (default: 10 minutes from creation). Enforced by Redis TTL — expiry is not handled by application polling. `[NFR-REL-004]`

**Booking**
The transactional record of a Customer purchasing tickets. Created in PENDING state when checkout begins. Progresses through states via saga: PENDING → PAYMENT_INITIATED → PAYMENT_CONFIRMED → SEATS_COMMITTED → TICKETS_ISSUED → CONFIRMED. Compensation states: CANCELLATION_REQUESTED → REFUND_INITIATED → REFUND_CONFIRMED → CANCELLED. The Booking is the unit of saga coordination.

**RevenueSplit**
An immutable record created at booking confirmation time. Records three amounts: platformFee, venueShare, organiserRemainder. Computed once from the booking total and the prevailing revenue share percentages at that instant. Never recomputed. Refunds are applied proportionally to the original split amounts. `[NFR-REL-012]`

**Ticket**
A confirmed, non-transferable entitlement to attend an event occupying a specific seat. Carries a cryptographically signed QR token: HMAC-SHA256 of (ticketId + eventId + seatId + customerId + issuedAt), keyed by a per-event secret. Verification is signature-only — no database call in the hot path. `[NFR-PERF-004]`

**WaitlistEntry**
An ordered queue entry for a sold-out event. Ordered by timestamp of entry (FIFO). On seat release, the platform creates a time-limited WaitlistOffer for the top entry. If the offer expires unclaimed, the next entry is promoted.

**DisbursementRecord**
An immutable ledger entry recording the transfer of escrowed funds to a Venue or Organiser after event completion. Linked to the originating RevenueSplit records. The disbursement ledger is append-only; no records are ever updated or deleted.

### 7.2 Core State Machines

**Event States**
```
DRAFT → PUBLISHED → ON_SALE → SOLD_OUT
                  ↘ POSTPONED
                  ↘ CANCELLED
ON_SALE → COMPLETED (after event date passes and Admin confirms)
```

**Booking States (forward path)**
```
PENDING → PAYMENT_INITIATED → PAYMENT_CONFIRMED
       → SEATS_COMMITTED → TICKETS_ISSUED → CONFIRMED
```

**Booking States (compensation path)**
```
CONFIRMED → CANCELLATION_REQUESTED → REFUND_INITIATED
          → REFUND_CONFIRMED → CANCELLED
```

**Organiser KYC States**
```
NOT_SUBMITTED → PENDING_REVIEW → VERIFIED
                              → REJECTED (with reason)
                              → MORE_INFO_REQUIRED
REJECTED → PENDING_REVIEW (resubmission)
```

**VenueBooking States**
```
REQUESTED → ACCEPTED → CONFIRMED (on Event creation)
          → REJECTED (with reason)
CONFIRMED → CANCELLED (if Event is cancelled)
```

### 7.3 Revenue Split Model

For every ticket sale at price P (GST-exclusive):

```
platformFee     = P × platform_fee_rate       (configured by Admin, e.g. 5%)
venueShare      = P × venue_share_rate        (negotiated at VenueBooking time, e.g. 20%)
organiserShare  = P - platformFee - venueShare (remainder)
```

Funds are held in escrow. Disbursement is triggered by event completion (Admin action). Refunds on customer-initiated cancellations reduce the disbursable amount proportionally across all three split components. The platform fee is non-refundable on late cancellations where the CancellationPolicy allows partial or no refund (the no-refund amount accrues to the Platform).

GST is collected on top of the ticket price and remitted separately. GST treatment in the split model is deferred to ADR-008.

---

## 8. Feature Requirements

Requirements are written as user needs. Implementation decisions belong in ADRs and the HLD, not here.

### 8.1 Customer

---

#### FR-C-001 — Registration and Authentication

**Need:** Customers must be able to create an account, verify their identity, log in securely, and manage their session.

**Acceptance Criteria:**
1. A Customer can register with email address and phone number. Phone number is verified via OTP.
2. A Customer can log in with email + password or OTP-based passwordless login.
3. Passwords are stored with bcrypt cost ≥ 12 or argon2id. `[NFR-SEC-007]`
4. Sessions use RS256-signed JWTs. Access token TTL ≤ 15 minutes. Refresh token TTL ≤ 7 days, rotated on every use. `[NFR-SEC-002]`
5. A Customer can log out from the current session; the access token's JTI is added to the revocation blocklist.
6. A Customer can view all active sessions and revoke any session individually.
7. Account deletion honours DPDP Act erasure rights: PII is deleted or anonymised; booking financial records are retained for GST audit purposes.

---

#### FR-C-002 — Event Discovery

**Need:** Customers must be able to find events through multiple discovery modalities: keyword search, filters, AI recommendations, and semantic / image-based search.

**Acceptance Criteria:**
1. A Customer can search events by keyword. Results are returned within 500 ms at 100 RPS. `[NFR-PERF-006]`
2. Search results can be filtered by: category (concert, comedy, theatre, sports), city, date range, price range, availability (has seats), venue.
3. A Customer can perform semantic search — expressing intent in natural language (e.g., "indie rock shows in Bengaluru next month under ₹1,000") — and receive ranked, relevant results. p99 < 800 ms. `[NFR-PERF-007]`
4. A Customer can perform image-based search — uploading or linking a poster image — to find visually similar events. p99 < 3,000 ms. `[NFR-PERF-008]`
5. A logged-in Customer's homepage surfaces AI-personalised event recommendations. Recommendation load p99 < 500 ms (Redis-cached inference). `[NFR-PERF-012]`
6. A Customer who has not logged in receives popularity-based recommendations as a fallback. `[NFR-AVAIL-004]`
7. New events published by Organisers are indexed in the search catalogue within 30 seconds of publication. `[NFR-PERF-010]`

---

#### FR-C-003 — Event Detail and Seat Selection

**Need:** Customers must be able to view full event details and select specific seats on an interactive, real-time seat map before committing to purchase.

**Acceptance Criteria:**
1. An event detail page displays: name, description, Organiser, Venue, date/time, category, poster image, pricing tiers, cancellation policy summary, and current availability.
2. A Customer can open an interactive seat map showing all seats colour-coded by state: AVAILABLE (selectable), HELD (greyed out, not selectable), BOOKED (greyed out, not selectable), BLOCKED (hidden or greyed).
3. Seat state changes made by other Customers are reflected on the seat map within 1 second via WebSocket push. `[NFR-PERF-005]`
4. A Customer can select up to 8 seats per booking in a single session.
5. Selecting a seat initiates a hold. Hold confirmation (seat transitions to HELD) is returned to the Customer within 500 ms. `[NFR-PERF-001]`
6. The Customer sees a countdown timer showing the remaining hold window (default 10 minutes).
7. A Customer cannot select a seat that is currently HELD by another Customer.
8. If the seat hold expires before checkout is completed, the seat is released and the Customer is notified with the option to reselect.

---

#### FR-C-004 — Checkout and Payment

**Need:** Customers must be able to complete ticket purchase through a reliable, idempotent checkout flow with clear pricing and UPI/card payment options.

**Acceptance Criteria:**
1. The checkout page displays: selected seats, price per seat (surge-adjusted, locked at hold creation time), GST amount, platform convenience fee (if any), and total payable.
2. A Customer can pay via UPI, credit/debit card, or net banking. All payment methods are mocked; no real transaction occurs.
3. The checkout flow is initiated with an idempotency key issued by the platform. Repeating the same checkout (e.g., double-tap on the Pay button) does not create duplicate bookings. `[NFR-REL-001]`
4. If Payment Service is unavailable, the checkout fails fast with a 503 response within 2 seconds. The seat hold is automatically extended by 5 minutes. An idempotency key is returned so the Customer can safely retry. `[NFR-AVAIL-003]`
5. The synchronous checkout submission (PENDING booking created, idempotency key returned) completes in p99 < 2 s. `[NFR-PERF-002]`
6. The Customer receives a booking confirmation notification (in-app and email) when the booking saga completes. End-to-end p99 < 10 s from checkout submit. `[NFR-PERF-003]`
7. If the saga fails at any step (seat commit, ticket issuance), the Customer's payment is automatically refunded and the seat is released. Compensation is logged and traceable. `[NFR-REL-003]`

---

#### FR-C-005 — Ticket Management

**Need:** Customers must be able to access, view, and download their confirmed tickets at any time.

**Acceptance Criteria:**
1. A Customer can view all confirmed, upcoming, and past tickets in their account.
2. Each ticket displays: event name, date/time, Venue, section, row, seat number, ticket ID, and QR code.
3. A Customer can download a PDF ticket for each booking. PDF is generated by the Ticket Service and stored in MinIO.
4. The QR code encodes a signed token: HMAC-SHA256 of (ticketId + eventId + seatId + customerId + issuedAt). The signing secret is per-event, rotated by Admin. `[NFR-SEC-005]`
5. A Customer can share a digital ticket link (expiring signed URL to the PDF).
6. A ticket for a cancelled event shows CANCELLED status and links to the refund status.

---

#### FR-C-006 — Waitlist

**Need:** Customers must be able to join a waitlist for sold-out events and receive fair, time-limited offers when seats become available.

**Acceptance Criteria:**
1. When an event is SOLD_OUT, a Customer can join the waitlist. Joining is acknowledged immediately.
2. The Customer's position in the waitlist queue is displayed (e.g., "You are #47 in the queue").
3. When a seat is released (cancellation or expired hold), the platform creates a WaitlistOffer for the Customer at the top of the queue.
4. The Customer receives a push notification and email with a time-limited link to claim the seat (default: 15 minutes).
5. If the offer is not claimed within the window, it expires and the next Customer in the queue receives an offer.
6. A Customer can leave the waitlist at any time.
7. A Customer already on the waitlist cannot join it again for the same event. `[NFR-SEC-006]` (by analogy — idempotent join)

---

#### FR-C-007 — Cancellation and Refund

**Need:** Customers must be able to cancel tickets within the terms of the Organiser's cancellation policy and receive refunds transparently.

**Acceptance Criteria:**
1. A Customer can initiate a cancellation for any confirmed ticket from their account.
2. Before confirming, the Customer is shown the exact refund amount based on the current time relative to the event date and the Organiser's CancellationPolicy.
3. Cancellation initiates the compensation saga: refund is processed, seat is released, ticket is voided, and the RevenueSplit is adjusted.
4. The Customer receives a refund confirmation notification with the refund amount and expected credit timeline (mocked: immediate in stub).
5. Attempting to cancel an already-cancelled booking returns a success response without re-processing. `[NFR-REL-011]`
6. If the event has already started, customer-initiated cancellation is blocked.

---

#### FR-C-008 — AI Chatbot

**Need:** Customers must be able to query event information, booking status, and platform policies through a conversational AI interface.

**Acceptance Criteria:**
1. A Customer can open a chat interface and ask questions in natural language about events (dates, pricing, availability, Venue details), their own bookings, and platform policies (refunds, cancellation).
2. The chatbot is backed by a RAG pipeline over the live event catalogue and policy documents.
3. First-token response latency p99 < 2,500 ms. `[NFR-PERF-009]`
4. The chatbot streams its response token-by-token (not a single block response).
5. The chatbot has conversation memory within a session (Redis-backed). It does not retain memory across sessions unless the Customer explicitly consents under DPDP Act requirements.
6. The chatbot cannot perform write operations (it cannot book tickets, cancel bookings, or make payments on behalf of the Customer). It provides information and links only.
7. If the Chatbot Service is unavailable, the chat UI shows a graceful fallback message; the rest of the event page remains fully functional. `[NFR-AVAIL-004]` (by analogy)

---

#### FR-C-009 — Real-Time Notifications

**Need:** Customers must receive timely notifications about booking events, waitlist updates, and event changes through multiple channels.

**Acceptance Criteria:**
1. In-app notifications are delivered via WebSocket to logged-in Customers within 1 second of the triggering event. `[NFR-PERF-005]`
2. Email notifications are sent for: booking confirmation, booking cancellation and refund, waitlist offer, event postponement, event cancellation.
3. Push notifications (browser push) are sent for: waitlist offer received, booking confirmation, event cancellation.
4. If the Notification Service is down, bookings still complete. The Notification Service failure must never block the booking saga. `[NFR-AVAIL-001]`
5. Customers can configure notification preferences (opt out of email, opt out of push) per notification category.

---

### 8.2 Organiser

---

#### FR-O-001 — Registration and KYC

**Need:** Organisers must be able to self-register and complete identity verification before they can publish events.

**Acceptance Criteria:**
1. An entity can register as an Organiser with: legal business name, GST number (optional at registration, required at first disbursement), PAN number, registered address, and primary contact.
2. KYC submission transitions status from NOT_SUBMITTED → PENDING_REVIEW.
3. KYC review is a stub: after a configurable delay (default: 30 seconds in development), status transitions to VERIFIED automatically. In the Admin UI, Admin can manually REJECT or request MORE_INFO.
4. An Organiser with KYC status NOT_SUBMITTED, PENDING_REVIEW, or REJECTED cannot publish events. They can create events in DRAFT state.
5. Organisers receive email notifications at each KYC state transition.
6. A REJECTED Organiser can resubmit KYC (transitions back to PENDING_REVIEW).

---

#### FR-O-002 — Venue Discovery and Booking Request

**Need:** Organisers must be able to discover suitable Venues and initiate booking requests without offline communication.

**Acceptance Criteria:**
1. An Organiser can search Venues by: city, capacity range, date availability, Venue type (arena, theatre, stadium, club), facilities.
2. Each Venue listing displays: name, location, photos, capacity, seating layout options, available dates, indicative revenue share percentage, and Venue description.
3. An Organiser can submit a VenueBooking request specifying: requested dates, expected attendance, proposed event type, and a message to the Venue.
4. The Venue receives a notification of the request. The Organiser receives a notification when the Venue responds.
5. A VenueBooking can only be in one state at a time. An Organiser cannot submit a second request for the same Venue on overlapping dates if a REQUESTED or ACCEPTED request exists.
6. A rejected VenueBooking includes the Venue's rejection reason.

---

#### FR-O-003 — Event Creation and Configuration

**Need:** Organisers must be able to fully configure an event once a Venue booking is confirmed.

**Acceptance Criteria:**
1. An Organiser can create an Event against a CONFIRMED VenueBooking. The Event inherits the Venue's confirmed seating layout.
2. Event configuration includes: name, description, category, poster image (uploaded to MinIO), date and time (door open, event start, event end), age restriction, language, tags.
3. The Organiser defines at least one PricingTier. Each tier is associated with one or more seating sections and has a base price in INR.
4. The Organiser defines SurgePricingRules: each rule specifies a section-availability threshold (as a percentage of total seats) and a price multiplier (1.0× to configured maximum, default maximum: 2.0×).
5. The Organiser defines a CancellationPolicy: one or more time-bracket refund rules. A bracket must cover the full timeline to event start (the final bracket can be "no refund").
6. The Organiser can block specific seats (e.g., reserved for performers, accessibility requirements). Blocked seats appear as BLOCKED on the seat map and cannot be purchased.
7. An event is created in DRAFT state. Moving to PUBLISHED requires: KYC VERIFIED, at least one PricingTier, a CancellationPolicy defined, poster image uploaded, and at least one seat in AVAILABLE state.

---

#### FR-O-004 — Event Lifecycle Management

**Need:** Organisers must be able to manage the full lifecycle of an event from publication through completion or cancellation.

**Acceptance Criteria:**
1. An Organiser can PUBLISH a DRAFT event, making it visible in the catalogue and available for ticket purchases.
2. An Organiser can POSTPONE an event: a new date must be specified. Customers are notified. Existing bookings remain valid for the new date. Customers retain the right to cancel with a full refund (overriding the CancellationPolicy) for 72 hours after a postponement announcement.
3. An Organiser can CANCEL an event. Cancellation triggers a fan-out saga: all confirmed bookings receive full refunds, all seats are released, all tickets are voided. The Organiser cannot cancel an event with active disbursements already completed.
4. Event cancellation fan-out is idempotent: if the cancellation saga is retried, bookings already refunded are not refunded again. `[NFR-REL-011]`
5. An Organiser can edit event metadata (name, description, poster, tags) at any time before the event starts. Metadata changes do not affect existing bookings.
6. Pricing tiers and cancellation policy are immutable after the first ticket is sold.
7. After the event start time passes, the Event transitions to COMPLETED (Admin-confirmed). Disbursement is triggered.

---

#### FR-O-005 — Live Operations Dashboard

**Need:** Organisers must have real-time visibility into ticket sales and attendance during sales periods and live events.

**Acceptance Criteria:**
1. The Organiser dashboard displays: total tickets sold, revenue to date (Organiser share, pre-disbursement), tickets sold by tier, remaining seats by section, current waitlist depth.
2. All metrics refresh every 60 seconds from the Analytics Service. Live counters (tickets sold since page load) update via WebSocket push. `[NFR-OBS-005]`
3. During the live event, the dashboard shows: attendees checked in, check-in rate over time, no-show count (seats BOOKED but not checked in by event start + 30 minutes).
4. The Organiser can view a read-only snapshot of the current seat map showing AVAILABLE/HELD/BOOKED/BLOCKED states.
5. AI no-show prediction is displayed as a widget: "Predicted no-shows: ~150 (±20)". This is informational; the Organiser takes no automated action on it.

---

#### FR-O-006 — Revenue and Disbursement

**Need:** Organisers must have full visibility into their revenue split, escrow balance, and disbursement history.

**Acceptance Criteria:**
1. An Organiser can view a revenue summary for each event: gross revenue, platform fee deducted, Venue share deducted, Organiser net share, refunds processed, and net disbursable amount.
2. The Organiser can view individual RevenueSplit records linked to each booking.
3. After event COMPLETION and Admin-initiated disbursement, the Organiser receives a disbursement notification with the exact amount.
4. Disbursement records are read-only. The Organiser cannot modify them. `[NFR-REL-012]`
5. The Organiser can view the full disbursement history across all their events.

---

### 8.3 Venue

---

#### FR-V-001 — Venue Registration and Profile

**Need:** Venue owners must be able to register their spaces on the platform with full configuration.

**Acceptance Criteria:**
1. A Venue can register with: name, type (arena, theatre, stadium, amphitheatre, club, convention centre), city, full address, Google Maps pin (latitude/longitude), facilities (parking, accessibility, F&B, AV), photos (uploaded to MinIO), and a contact person.
2. A Venue must be approved by Admin before it is visible to Organisers.
3. A Venue can update profile information (photos, facilities, contact) at any time. Seating layout changes follow the versioning rules in FR-V-002.

---

#### FR-V-002 — Seating Layout Management

**Need:** Venues must be able to define and version their seating configurations.

**Acceptance Criteria:**
1. A Venue can create a SeatingLayout with a hierarchical structure: Section → Row → Seat. Each seat has a unique identifier within the Venue, a type (STANDARD, PREMIUM, ACCESSIBLE), and SVG coordinates for map rendering.
2. Stadium-scale layouts (up to 50,000 seats) must be loadable in the seat map within 3 seconds on a standard connection.
3. A SeatingLayout is versioned. Once seats from a layout version are sold for any event, that layout version is immutable.
4. A Venue can create a new layout version (e.g., for renovation). New events use the latest layout version. In-flight events retain their version.
5. A Venue can define named sections (e.g., "North Stand", "Platinum Pavilion") with associated colours for seat map display.

---

#### FR-V-003 — Booking Request Management

**Need:** Venues must be able to review and respond to Organiser booking requests through a structured workflow.

**Acceptance Criteria:**
1. A Venue receives a notification when an Organiser submits a booking request.
2. The Venue can view the request details: Organiser profile, proposed dates, expected attendance, event type, and message.
3. The Venue can ACCEPT or REJECT a request. A rejection requires a reason.
4. The Venue can view a booking calendar showing all ACCEPTED, CONFIRMED, and past bookings, enabling them to identify scheduling conflicts before accepting new requests.
5. An ACCEPTED booking is not yet confirmed until the Organiser creates an Event against it. The Venue can see the status difference.
6. A Venue can mark specific date ranges as unavailable (maintenance, private events), preventing Organiser requests for those dates.

---

#### FR-V-004 — Revenue and Utilisation

**Need:** Venues must have visibility into their revenue share and space utilisation.

**Acceptance Criteria:**
1. A Venue can view revenue summaries per event: their revenue share amount, escrow status, and disbursement history.
2. A Venue can view utilisation analytics: events hosted per month, average seat occupancy rate, revenue per event.
3. Revenue share percentage is negotiated at VenueBooking time and stored as part of the VenueBooking record. It cannot be changed after ACCEPTED state.

---

### 8.4 Admin

---

#### FR-A-001 — KYC Review Workflow

**Need:** Admins must be able to review Organiser and Venue KYC submissions and action them.

**Acceptance Criteria:**
1. Admin sees a queue of PENDING_REVIEW KYC submissions, sorted by submission timestamp.
2. Admin can view submitted documents (identity proof, GST certificate, bank account proof — all stubbed as placeholder data).
3. Admin can APPROVE (transitions to VERIFIED), REJECT (with mandatory reason), or REQUEST_MORE_INFO (with specific questions).
4. All Admin actions on KYC are logged with Admin ID, timestamp, and reason. `[NFR-OBS-001]`
5. The Organiser or Venue is notified of the decision immediately.

---

#### FR-A-002 — Account Management and Suspension

**Need:** Admins must be able to suspend or reinstate Organisers and Venues, with the platform correctly cascading consequences.

**Acceptance Criteria:**
1. Admin can suspend an Organiser. Suspension blocks: new event creation, new Venue booking requests, new ticket sales for existing events. Existing confirmed bookings are not immediately cancelled (Admin chooses whether to also cancel events).
2. Admin can suspend a Venue. Venue suspension triggers the Venue suspension cascade: all upcoming events at the Venue are cancelled (with full customer refunds), and all future VenueBooking requests are blocked.
3. The Venue suspension cascade is idempotent. If triggered twice, no duplicate refunds are issued. `[NFR-REL-011]`
4. Admin can reinstate a suspended Organiser or Venue. Reinstatement does not restore cancelled events.
5. All suspension and reinstatement actions are logged.

---

#### FR-A-003 — Fraud Review

**Need:** Admins must be able to review fraud alerts and take action on flagged accounts and bookings.

**Acceptance Criteria:**
1. The Fraud Detection Service publishes scored events to a Kafka topic. Admin sees alerts for bookings scoring above a configurable HIGH-risk threshold.
2. Admin can view the fraud signals for a flagged booking: velocity score, device fingerprint anomaly, scalping pattern score.
3. Admin can: HOLD a booking (prevents ticket download and check-in), RELEASE a hold, FLAG an account (marks future bookings for enhanced scrutiny), SUSPEND an account.
4. Admin can bulk-action: bulk-hold all bookings from a specific IP range or device fingerprint cluster.
5. Fraud review actions are logged.

---

#### FR-A-004 — Platform Configuration

**Need:** Admins must be able to configure platform-level operational parameters without code deployment.

**Acceptance Criteria:**
1. Admin can configure: Platform fee percentage (applied to all new bookings; existing bookings are not retroactively changed), default seat hold window duration (in minutes), maximum surge multiplier allowed in SurgePricingRules, per-event QR signing secret rotation schedule.
2. Configuration changes are persisted (Spring Cloud Config or Vault) and take effect within 60 seconds without service restart.
3. All configuration changes are logged with the Admin ID, the previous value, the new value, and a mandatory reason.

---

#### FR-A-005 — Disbursement Management

**Need:** Admins must be able to trigger and monitor disbursements to Venues and Organisers after event completion.

**Acceptance Criteria:**
1. Admin sees a list of COMPLETED events with pending disbursements.
2. Admin can review the disbursement breakdown per event: total gross, platform fee, Venue share, Organiser share, refunds processed, and net disbursable amounts.
3. Admin triggers disbursement. The Disbursement Service creates immutable DisbursementRecords and sends notifications to Venue and Organiser.
4. Disbursement is mocked: no real bank transfer occurs. The disbursement record is the source of truth.
5. Admin can view the full disbursement audit trail for any event.

---

#### FR-A-006 — Platform Health Dashboard

**Need:** Admins must have a single view of platform health, SLO status, and operational alerts.

**Acceptance Criteria:**
1. The Admin dashboard embeds (or links to) a Grafana dashboard showing: request rates, error rates, p99 latency per service, active seat holds, flash sale queue depth, DLQ depth per Kafka topic, SLO burn rates.
2. Active Alertmanager alerts are surfaced in the Admin dashboard with severity, description, and runbook link.
3. Business metrics are visible: bookings/minute, GMV/minute, waitlist depth for top events, fraud holds active. `[NFR-OBS-005]`

---

### 8.5 AI Features

AI features are first-class platform capabilities. They degrade gracefully — no AI failure should make the core ticketing flow unavailable.

---

#### FR-AI-001 — Personalised Event Recommendations

**Need:** The platform must surface contextually relevant events to each logged-in Customer based on their behaviour and preferences.

**Acceptance Criteria:**
1. Recommendations are generated by an offline-trained collaborative filtering model served via the Recommendation Service.
2. Inference results are cached in Redis (TTL: 1 hour). Cache hit response p99 < 500 ms. `[NFR-PERF-012]`
3. Signals used for training: past bookings, search queries, event page views, waitlist joins, category preferences from registration.
4. Cold-start (new user) fallback: popularity-ranked events in the Customer's city. `[NFR-AVAIL-004]`
5. If the Recommendation Service is unavailable, the fallback is served. The event detail page does not return an error. `[NFR-AVAIL-004]`
6. Recommendations are refreshed offline daily. Model versions are tracked in MLflow.

---

#### FR-AI-002 — Semantic Event Search

**Need:** Customers must be able to search for events using natural language intent, not just keywords.

**Acceptance Criteria:**
1. The Search Service accepts a natural language query, generates an embedding via the Recommendation Service's embedding model, and queries Qdrant for nearest-neighbour event vectors.
2. Semantic search results are blended with keyword (BM25) results from Elasticsearch. Blending weights are configurable by Admin.
3. p99 < 800 ms at 100 RPS. `[NFR-PERF-007]`
4. Event vectors are updated in Qdrant whenever an event is published or its metadata changes. `[NFR-PERF-010]`

---

#### FR-AI-003 — Image-Based Event Search

**Need:** Customers must be able to search for events by uploading a poster or artist image.

**Acceptance Criteria:**
1. A Customer uploads an image. The Search Service generates a visual embedding (CLIP or similar) and queries Qdrant for nearest-neighbour event image vectors.
2. Results are ranked by visual similarity and filtered by availability (CANCELLED events are excluded).
3. p99 < 3,000 ms. `[NFR-PERF-008]`
4. Event image vectors are indexed when the Organiser uploads a poster image.

---

#### FR-AI-004 — RAG Chatbot over Event Catalogue

**Need:** Customers must be able to get accurate, contextually grounded answers about events and platform policies through a conversational interface. (See FR-C-008 for Customer-facing requirements; this entry covers the AI pipeline requirements.)

**Acceptance Criteria:**
1. The Chatbot Service maintains a retrieval corpus: live event documents, Venue documents, platform policy documents (refund policies, terms of service).
2. The retrieval pipeline uses Qdrant for vector retrieval and LangChain for orchestration.
3. Retrieved context is injected into the LLM prompt. The LLM generates a response grounded in the retrieved documents.
4. The chatbot cites its sources (event name, policy section) in responses where relevant.
5. Conversation history is maintained in Redis for the duration of the session. `[NFR-PERF-009]`
6. The chatbot does not hallucinate event details. If no relevant document is retrieved, it responds: "I don't have specific information about that. Here is a link to the event page / help centre."

---

#### FR-AI-005 — Demand Prediction

**Need:** Organisers must receive AI-generated demand forecasts to inform their pricing and event configuration decisions.

**Acceptance Criteria:**
1. For each published event, the Demand Prediction Service generates a forecast: expected ticket sales by date, expected sell-out probability, expected peak-demand windows.
2. Forecasts are trained offline using historical sales velocity, event category, Venue location, day-of-week, lead time, and similar-event data.
3. Forecasts are surfaced in the Organiser dashboard as: a sales velocity chart with a predicted trajectory, a sell-out probability indicator, and recommended surge pricing threshold configurations.
4. The forecast is a suggestion only. The Organiser takes no automated action; they may adjust SurgePricingRules based on the recommendation.
5. Forecasts are regenerated daily during the on-sale period.

---

#### FR-AI-006 — Dynamic Surge Pricing Enforcement

**Need:** The platform must enforce Organiser-configured surge pricing rules at seat hold creation time.

**Acceptance Criteria:**
1. At seat hold creation, the Seat Inventory Service evaluates all active SurgePricingRules for the event.
2. If the current availability of a section's seats has crossed a threshold defined in a rule, the surge multiplier is applied to the base price.
3. The surge-adjusted price is stored in the SeatHold record and is locked for the hold's duration. Subsequent rule triggers during the hold window do not change the held price.
4. The Customer sees the surge-adjusted price before confirming checkout, with an indicator that a demand premium applies.
5. Surge multipliers are capped at the platform maximum (Admin-configured, default 2.0×).
6. The AI Demand Prediction Service surfaces suggested threshold values to the Organiser when creating rules. The Organiser may accept, modify, or ignore these suggestions.

---

#### FR-AI-007 — Fraud Detection

**Need:** The platform must automatically score bookings for fraud patterns and surface high-risk bookings to Admin for review.

**Acceptance Criteria:**
1. The Fraud Detection Service consumes booking events from Kafka. A fraud score is published to a Kafka topic within 10 seconds of booking placement. `[NFR-PERF-011]`
2. Scored signals include: booking velocity from a single account, multiple accounts from the same device fingerprint, bulk seat purchases for high-demand events (scalping pattern), payment method anomalies.
3. Bookings scoring above a HIGH-risk threshold are flagged. Admin receives an alert. The booking is not automatically cancelled — Admin reviews and takes action. `[NFR-AVAIL-005]`
4. If the Fraud Detection Service is unavailable, bookings proceed as LOW-risk. Fraud detection failure never blocks a booking. `[NFR-AVAIL-005]`
5. Model performance is tracked in MLflow: precision, recall, false positive rate, measured against labelled dispute outcomes.

---

#### FR-AI-008 — No-Show Prediction

**Need:** Organisers must receive AI-generated no-show predictions to inform operational decisions (standby queues, overbooking policy, staff planning).

**Acceptance Criteria:**
1. For each event with >50% of capacity sold, the No-Show Prediction Service generates a predicted no-show count and confidence interval 24 hours before event start.
2. Prediction signals: historical no-show rates for the Organiser, event category, ticket price tier (higher price → lower no-show), day-of-week, weather data (future consideration).
3. Predictions are surfaced in the Organiser's live operations dashboard as an informational widget.
4. Predictions are informational only. The platform does not automatically overbook based on them. This capability is documented as a future feature.

---

### 8.6 Platform-Wide and Cross-Cutting

---

#### FR-P-001 — Real-Time Seat Map

**Need:** All connected clients must see accurate seat state in near-real-time, regardless of concurrent booking activity.

**Acceptance Criteria:**
1. The Notification Service pushes seat state change events to all clients subscribed to an eventId room via WebSocket (Socket.IO).
2. State changes are pushed within 1 second of the state change being committed. `[NFR-PERF-005]`
3. A new client joining the seat map receives a full state snapshot on connection. Subsequent messages are differential (changed seats only).
4. Socket.IO rooms are scoped per eventId for seat map state. Horizontal scaling of the Notification Service is supported via the Redis Socket.IO adapter.
5. The seat map gracefully handles stadium-scale events (50,000 seats): the WebSocket payload for a single seat state change must not include the full seat map.

---

#### FR-P-002 — QR Check-In

**Need:** Venue staff must be able to verify tickets at the gate with low latency and offline resilience.

**Acceptance Criteria:**
1. The Check-in Service exposes an endpoint that accepts a QR token and verifies its HMAC-SHA256 signature using the per-event secret. No database call is required in the verification hot path. `[NFR-PERF-004]`
2. Verification p99 < 200 ms. `[NFR-PERF-004]`
3. A valid, unscanned token returns: VALID — proceeds to mark the ticket as CHECKED_IN (async, Kafka event).
4. A token for an already-checked-in ticket returns: ALREADY_CHECKED_IN. No duplicate check-in record is created. `[NFR-SEC-006]`
5. An invalid or tampered token returns: INVALID. The event is logged as a security alert.
6. A token for the wrong event returns: WRONG_EVENT. `[NFR-SEC-005]`
7. If the Check-in Service loses network connectivity, it falls back to offline signature verification using the cached per-event key. Check-in continues. Check-in records sync when connectivity is restored. `[NFR-AVAIL-006]`

---

#### FR-P-003 — Flash Sale Queue

**Need:** Under extreme concurrent load (high-demand events), the platform must not oversell, must not collapse, and must maintain fairness.

**Acceptance Criteria:**
1. When incoming seat hold requests exceed a configurable threshold (flash sale mode is triggered by Admin or automatically when RPS exceeds a threshold), requests are queued via a Kafka-based funnel topic.
2. The queue is consumed at a controlled rate that the Seat Inventory Service can handle without contention collapse.
3. Queued Customers receive a real-time queue position update via WebSocket.
4. The system sustains 10× normal RPS for 60 seconds without overselling. Error rate during load < 0.5%. `[NFR-PERF-013]`
5. Fairness is FIFO within the queue. Customers who entered the queue first are processed first.
6. A Customer whose queue position is reached but whose requested seats are no longer available is notified immediately and offered the option to select alternative seats or join the waitlist.

---

#### FR-P-004 — Idempotency

**Need:** All write endpoints must be safe to retry without side effects.

**Acceptance Criteria:**
1. All write endpoints (checkout, cancellation, disbursement trigger, KYC submission) accept an `Idempotency-Key` header. `[NFR-REL-001]`
2. A repeat request with the same idempotency key within the key's TTL returns the cached response without re-executing the operation.
3. All Kafka consumers are idempotent: receiving the same event twice produces the same outcome as receiving it once. `[NFR-REL-002]`

---

#### FR-P-005 — Booking Saga with Compensation

**Need:** The multi-step booking flow must be reliably coordinated across service boundaries, with explicit compensation for every failure mode.

**Acceptance Criteria:**
1. The Booking Service orchestrates the booking saga: seat commit, payment initiation, payment confirmation, ticket issuance, notification.
2. Every forward step has a documented and tested compensation action. `[NFR-REL-003]`
3. Saga state is persisted in the Booking Service database. A saga can be resumed after a service restart.
4. The Outbox pattern is used for all saga-related events that must not be lost. `[NFR-REL-005]`
5. All synchronous outbound calls (to Payment Service, Seat Inventory Service via gRPC) use exponential backoff with jitter. Circuit breakers are configured per NFR-REL-006. `[NFR-REL-006]`
6. All saga events are published to Kafka topics with DLQ topics. Alerting fires if DLQ depth > 0 for > 5 minutes. `[NFR-REL-007]`

---

### 8.7 Resilience and Graceful Degradation

This section consolidates the platform's resilience philosophy and defines, for every service dependency, what happens when that dependency is unavailable. It is the authoritative statement of product intent. The NFR document captures the measurable constraints (timeouts, retry parameters, SLO budgets); this section captures the *why* — what experience must be preserved and what is acceptable to degrade.

---

#### FR-P-006 — Resilience Philosophy

**Core principle: the severity of unavailability must never exceed the severity of the failure it prevents.**

Two failure philosophies govern different parts of the platform:

**Fail-closed** applies where proceeding without a dependency risks a correctness violation that is worse than an outage. The platform would rather return an error to the Customer than produce an incorrect result.

**Fail-open** applies where a degraded experience is always preferable to blocking the Customer entirely. The platform proceeds with a safe default and restores full behaviour when the dependency recovers.

The fail-open/fail-closed classification of every service is a deliberate product decision, not an implementation convenience. It must be implemented, tested, and observable.

---

#### FR-P-007 — Fail-Closed Services

The following services are fail-closed. If unavailable, the dependent flow fails fast with an appropriate error. No unsafe default is acceptable.

| Service | Dependent Flow | Behaviour on Unavailability | Rationale |
|---|---|---|---|
| **Seat Inventory** | Seat hold, checkout | Checkout fails fast within 2 s. Customer's seat selection state in the browser is preserved for retry. `[NFR-AVAIL-002]` | Proceeding without Seat Inventory risks issuing holds with no atomicity guarantee. Overselling is a correctness violation worse than a checkout failure. |
| **Payment** | Checkout payment step | Checkout fails fast with 503. Idempotency key is returned for safe retry. Seat hold is automatically extended by 5 minutes. `[NFR-AVAIL-003]` | Proceeding without payment confirmation risks issuing tickets without funds collected. The hold extension prevents the Customer losing their seat during the retry window. |
| **Booking** | Any booking initiation | Checkout is unavailable. A clear error is shown. | The Booking Service is the saga coordinator. Without it, no booking state can be tracked and no compensation can be guaranteed. Proceeding would produce orphaned holds with no recovery path. |
| **Auth** | Any authenticated request | Request is rejected with 401. No authenticated operation proceeds without a valid, locally validated JWT. `[NFR-SEC-001]` | Proceeding without authentication would violate every tenant isolation and RBAC guarantee in the platform. |

---

#### FR-P-008 — Fail-Open Services

The following services are fail-open. If unavailable, the platform degrades gracefully to a defined fallback. The core booking flow and event browsing experience remain available.

| Service | Dependent Flow | Fallback Behaviour | Recovery Behaviour | NFR |
|---|---|---|---|---|
| **Recommendation** | Homepage personalisation, event detail "You might also like" | Popularity-ranked events in the Customer's city are shown. Page must not return an error. | Personalised recommendations restored automatically when service recovers. No Customer action required. | `[NFR-AVAIL-004]` |
| **Fraud Detection** | Booking placement | Booking proceeds as LOW-risk. Fraud scoring is deferred. No booking is blocked due to Fraud Detection unavailability. | When the service recovers, unscored bookings from the outage window are retrospectively scored from the Kafka topic. High-risk findings trigger Admin alerts. | `[NFR-AVAIL-005]` |
| **Notification** | Booking confirmation, cancellation, waitlist offer | Booking saga completes without waiting for notification delivery. Notifications are published to Kafka; delivery is eventually consistent. | Notifications are delivered when the Notification Service recovers, consuming from its Kafka topic. Duplicates are suppressed by idempotency keys. | `[NFR-AVAIL-001]` |
| **Chatbot** | Event detail chat widget | A static fallback message is displayed: "Chat is temporarily unavailable. Visit our help centre for event information." The event detail page renders fully without the chat widget. | Chat widget restored automatically on recovery. | `[NFR-AVAIL-004]` (by analogy) |
| **Analytics** | Organiser live dashboard metrics | Last-known metrics are displayed with a staleness indicator ("Last updated: 3 min ago"). The dashboard does not error. | Live metrics restored on recovery. No data is lost — Analytics Service catches up from Kafka. | — |
| **Search (Elasticsearch)** | Keyword and semantic search | A degraded search page is shown with a "Search is temporarily unavailable" message. Event browsing via category and city filters (served from the Event Service) remains available. | Full search restored on recovery. | — |
| **Check-in** (network loss at gate) | QR ticket verification | Offline signature verification using the cached per-event HMAC key. Check-in proceeds locally. Check-in records sync to the service when connectivity is restored. `[NFR-AVAIL-006]` | Online verification resumes on reconnection. Offline check-ins are reconciled; duplicate scans detected and flagged. | `[NFR-AVAIL-006]` |

---

#### FR-P-009 — Timeout and Retry Philosophy

All synchronous outbound calls (HTTP and gRPC) follow a single, platform-wide retry discipline. No service invents its own retry strategy.

**Retry policy (all synchronous calls):**
- Maximum attempts: 3 (1 original + 2 retries)
- Backoff: exponential with jitter — `delay = base × 2^attempt + rand(0, base)` where `base = 100 ms`
- Maximum delay cap: 5 seconds
- Retryable conditions: network timeout, 429 (rate limited), 503 (service unavailable), 504 (gateway timeout)
- Non-retryable conditions: 4xx (except 429), 500 (application error — retrying a 500 may duplicate side effects)

**Circuit breaker (all synchronous calls):**
- Opens after a configurable number of consecutive failures (default: 5)
- Half-open after 30 seconds: one probe request is allowed through
- Closes after 3 consecutive successes in half-open state
- While open: calls fail immediately with a fallback (fail-open services) or a fast error (fail-closed services)

`[NFR-REL-006]`

**Explicit timeouts (per call type):**
| Call | Timeout |
|---|---|
| Seat hold (Booking → Seat Inventory via gRPC) | 500 ms |
| Payment initiation (Booking → Payment via REST) | 3 s |
| Ticket issuance (Booking → Ticket via Kafka, async) | No synchronous timeout; saga tracks state |
| QR verification (Check-in → Ticket via gRPC) | 200 ms |
| Any gateway → service call | 10 s |

Fast-fail timeouts are non-negotiable. A missing timeout converts a dependency failure into an indefinite hang, exhausting thread pools and causing cascading failures across the platform.

---

#### FR-P-010 — Automatic Compensation vs Admin Intervention

Not every failure is handled the same way. This section defines which failures the platform resolves autonomously and which require a human decision.

**Automatic compensation (saga-managed, no human required):**

| Trigger | Compensating Action |
|---|---|
| Payment fails during booking saga | Seats released; booking cancelled; Customer notified |
| Seat commit fails after payment confirmed | Payment refunded; seats released; booking cancelled |
| Ticket issuance fails after seat committed | Seats released; payment refunded; booking cancelled |
| Customer cancels within CancellationPolicy terms | Refund computed from original RevenueSplit; seat released; ticket voided; waitlist offer triggered if applicable |
| Seat hold TTL expires (10-minute window) | Seat automatically released by Redis TTL; Customer notified; waitlist offer triggered if applicable |
| Waitlist offer TTL expires (15-minute window) | Offer expired; next waitlist member promoted automatically |

**Admin intervention required:**

| Trigger | Why Human Required | Admin Action |
|---|---|---|
| Fraud Detection scores a booking HIGH-risk | Fraud signals are probabilistic. Automatic cancellation would produce false positives, destroying legitimate bookings. A human must assess context. | Review signals; HOLD, RELEASE, FLAG, or SUSPEND |
| KYC submission review | Document verification requires human judgement (in the stubbed model, this is a configurable auto-approve; in production it would require human review) | APPROVE, REJECT, or REQUEST_MORE_INFO |
| Venue suspension | Suspending a Venue cancels all upcoming events at that Venue, each triggering a booking refund fan-out. This is an irreversible, high-impact action. A human must confirm intent. | Initiate suspension; platform executes cascade |
| Disbursement initiation | Disbursements release escrowed funds to Venues and Organisers. This is a financial action requiring human authorisation. | Review breakdown; trigger disbursement |
| DLQ messages unresolved > 5 minutes | Dead-lettered messages indicate a systematic failure that automated retry could not resolve. The cause must be diagnosed before reprocessing. | Diagnose; fix root cause; replay or discard |
| Booking saga stuck (no state transition for > 15 minutes) | An orphaned saga indicates an unhandled failure mode. Automatic recovery without understanding the cause risks double compensation. | Inspect saga state; manually trigger compensation or recovery |

---

#### FR-P-011 — Cascade Isolation

The platform has two multi-level cascade scenarios. Both must be bounded, idempotent, and observable.

**Event Cancellation Cascade:**
An Organiser cancels an Event → fan-out to N Booking refund sagas, one per confirmed booking. Each booking saga executes independently. Failure of one booking's refund does not block others. The cascade is complete when all bookings reach CANCELLED state. Completeness is tracked and visible to Admin. `[NFR-REL-011]`

**Venue Suspension Cascade:**
Admin suspends a Venue → fan-out to all upcoming Events at that Venue (each transitions to CANCELLED) → each Event cancellation triggers its own booking refund fan-out. This is a two-level cascade: Venue suspension → M event cancellations → N×M booking refunds. The cascade is orchestrated via Kafka to decouple the levels. Each level is independently idempotent. `[NFR-REL-011]`

Both cascades must:
- Be fully observable: a single Admin view shows progress (events cancelled, bookings refunded, refunds pending)
- Alert on stall: if cascade progress stops for > 15 minutes, Alertmanager fires
- Be resumable: if the platform restarts mid-cascade, progress is not lost

---

## 9. Regulatory and Compliance Requirements

### 9.1 Goods and Services Tax (GST)

- All ticket prices displayed to Customers must be GST-inclusive (the displayed price is what the Customer pays).
- GST at the applicable rate (deferred to ADR-008 for exact rate treatment) is computed on the GST-exclusive base price.
- Tax invoices must be generated for each booking, showing: GST-exclusive price, applicable GST rate, GST amount, total price, Organiser's GSTIN (if provided), and platform's GSTIN.
- GST amounts are tracked separately in the RevenueSplit model and are not included in the Organiser or Venue revenue share calculation.

### 9.2 Digital Personal Data Protection Act, 2023 (DPDP Act)

- Customers must provide explicit consent before any personal data is collected. Consent notice must state: what data is collected, why, and for how long it is retained.
- Customers have the right to access their personal data. A data export must be available within 72 hours of request.
- Customers have the right to erasure. PII is deleted or anonymised on account deletion. Financial records (bookings, revenue splits, disbursements) are retained for GST audit purposes (minimum 7 years) with PII anonymised.
- Data collected is limited to what is necessary for the stated purpose (data minimisation).
- The Chatbot Service's cross-session conversation memory requires explicit opt-in.

### 9.3 Consumer Protection (E-Commerce) Rules, 2020

- Cancellation policy must be displayed clearly before purchase, not only in fine print.
- For platform-cancelled events, refunds must be initiated within 14 days of the cancellation.
- Customers must not be charged for services they did not opt into.

### 9.4 Payment Aggregator Compliance (RBI Guidelines)

- All Organiser and Venue disbursements flow through the licensed aggregator (Razorpay, stubbed). The platform does not hold or transfer real funds.
- Organiser KYC is designed to be compatible with what a licensed PA requires before enabling disbursements: PAN, bank account verification, and GST registration.

---

## 10. Observability Requirements

These requirements apply to every service in the platform from the first service shipped. Observability is not added retrospectively.

**OBS-01 — Structured Logging** `[NFR-OBS-001]`
All services emit structured JSON logs. Every log event includes: `trace_id`, `span_id`, `service`, `timestamp` (UTC ISO-8601), `level`, `message`, and where applicable: `bookingId`, `eventId`, `userId`. Logs are shipped to Loki via Promtail.

**OBS-02 — Metrics** `[NFR-OBS-002]`
All services expose `/metrics` in Prometheus format. RED metrics (Rate, Errors, Duration) are captured per endpoint. USE metrics (Utilisation, Saturation, Errors) are captured per resource (DB connections, Redis connections, Kafka consumer lag). Business metrics per NFR-OBS-005 are captured in the services that own the relevant data.

**OBS-03 — Distributed Tracing** `[NFR-OBS-003]`
All services emit OpenTelemetry traces. Trace context is propagated via HTTP headers (`traceparent`), gRPC metadata, and Kafka message headers. Traces are shipped to Jaeger via the OpenTelemetry Collector.

**OBS-04 — Traceability** `[NFR-OBS-004]`
Given any `bookingId`, all logs and spans across all services involved in that booking must be locatable in Grafana and Jaeger within 2 minutes.

**OBS-05 — Alerting**
Every Prometheus alert has a corresponding runbook in `stagepass-docs/docs/runbooks/`. Alerts include SLO burn-rate alerts (multi-window), DLQ depth alerts, circuit breaker open alerts, and booking saga stuck alerts.

---

## 11. Success Metrics

Success metrics are grouped by dimension. Each metric is measurable, not vague. Baselines are to be established in Phase 7 (Quality Engineering) using k6 load tests and production-equivalent data volumes.

### 11.1 Performance

| Metric | Target | NFR Reference |
|---|---|---|
| Seat hold p99 latency | < 500 ms | NFR-PERF-001 |
| Checkout submission p99 latency | < 2 s | NFR-PERF-002 |
| Booking saga end-to-end p99 | < 10 s | NFR-PERF-003 |
| QR check-in verification p99 | < 200 ms | NFR-PERF-004 |
| Seat map WebSocket push p99 | < 1 s | NFR-PERF-005 |
| Keyword search p99 at 100 RPS | < 500 ms | NFR-PERF-006 |
| Semantic search p99 at 100 RPS | < 800 ms | NFR-PERF-007 |
| Flash sale: 10× RPS for 60 s, error rate | < 0.5% | NFR-PERF-013 |
| Event indexing lag (publish to search) | < 30 s | NFR-PERF-010 |
| Fraud score publication lag | < 10 s | NFR-PERF-011 |

### 11.2 Reliability

| Metric | Target | NFR Reference |
|---|---|---|
| T1 service availability (Auth, Seat Inventory, Booking, Payment, Disbursement) | 99.9% | Section 12 |
| T2 service availability | 99.5% | Section 12 |
| T3 service availability | 99.0% | Section 12 |
| Duplicate booking rate | 0 | NFR-REL-001 |
| Oversell rate | 0 | NFR-REL-008 |
| Saga compensation success rate | 100% (for compensable failures) | NFR-REL-003 |
| DLQ messages unresolved > 5 min | Alerts fire; 0 in normal operation | NFR-REL-007 |

### 11.3 Security

| Metric | Target | NFR Reference |
|---|---|---|
| HIGH-severity SAST findings blocking CI | 0 on main | NFR-SEC-010 |
| CRITICAL CVE SCA findings | 0 on main | NFR-SEC-011 |
| Secrets committed to any repo | 0 | NFR-SEC-013 |
| Cross-tenant data access incidents | 0 | NFR-SEC-004 |

### 11.4 Engineering Quality

| Metric | Target | NFR Reference |
|---|---|---|
| Unit test branch coverage (per service) | ≥ 80% | NFR-MAINT-001 |
| Time to run any service locally from clone | < 30 min | NFR-MAINT-004 |
| ADR written before implementation (non-trivial decisions) | 100% | NFR-MAINT-005 |
| Full local stack RAM footprint | < 12 GB | NFR-PERF-043 |

### 11.5 Learning Goals (Project-Specific)

These metrics define success for the learning objectives of this project:

| Goal | Definition of Done |
|---|---|
| Saga pattern | Booking saga with 5+ steps and all compensations implemented and tested |
| Outbox pattern | At least 2 services using Outbox; at-least-once delivery proven by Testcontainers test |
| CQRS | Booking Service write model (PostgreSQL) + read model (MongoDB) implemented and in use |
| Event sourcing | At least one aggregate (Booking or Seat) rebuilt from its event log |
| gRPC cross-language | Booking → Seat Inventory (Java-Java) and Check-in → Ticket (Node-Java) both running |
| Flash sale queue | k6 load test at 10× RPS passes: 0 oversells, < 0.5% error rate |
| JWT full lifecycle | JWKS, local validation, JTI revocation, refresh token rotation all implemented |
| Observability | Single bookingId traceable end-to-end across all services in < 2 min in Grafana + Jaeger |

---

## 12. Constraints and Assumptions

**C-01 — Solo development**
The entire platform is built by one developer over 9–12 months. Every architectural decision must account for the solo-developer constraint. Simplicity is a feature where it does not compromise learning goals.

**C-02 — No real money**
All payment and disbursement flows are mocked. The architecture is designed to be compatible with a real Razorpay integration (correct webhook flows, HMAC verification, idempotency keys are all implemented in the stub), but no live credentials are used.

**C-03 — No real KYC verification**
KYC document validation is stubbed. The state machine transitions are real. The document validation step auto-approves after a configurable delay in the development environment.

**C-04 — Local-first, cloud-ready**
The primary deployment target for Phases 2–8 is local Kubernetes (Minikube). Cloud deployment (AWS EKS) is Phase 9. All infrastructure choices must work locally first.

**C-05 — Stack is locked**
The technology stack defined in the system prompt (Section 3) is locked. No framework or database changes without an ADR. The PRD does not re-decide the stack; it drives requirements the stack must satisfy.

**C-06 — Polyglot by design**
Services are deliberately built in different languages and frameworks (Java, TypeScript, Python) to maximise learning exposure. Accidental polyglot complexity (using multiple frameworks for the same problem without a reason) is explicitly prohibited.

**C-07 — INR only**
All prices are in Indian Rupees. Multi-currency is out of scope for v1.0.

**A-01 — Seat map SVG rendering**
The interactive seat map is a custom-built SVG component. No third-party seat picker library is used. This is a deliberate learning choice.

**A-02 — GST rate treatment**
Exact GST rate treatment (whether the platform charges GST on the platform fee separately, whether GST is split across parties) is deferred to ADR-008. The PRD assumes GST is collected and tracked; the exact accounting treatment is an ADR decision.

**A-03 — Venue revenue share negotiation**
The revenue share percentage is agreed offline (or via a message in the platform) and entered into the VenueBooking by the Organiser. The platform does not host a negotiation or counter-offer workflow within the VenueBooking request/response cycle. This is a future feature.

---

## 13. Open Questions and Deferred Decisions

| ID | Question | Deferred To | Trigger |
|---|---|---|---|
| OQ-01 | Exact GST rate treatment for entertainment events and the platform fee | ADR-008 | Before Disbursement Service implementation (Phase 4) |
| OQ-02 | Razorpay Route split API mapping to three-party model | ADR-008 | Before Payment Service implementation (Phase 4) |
| OQ-03 | How to handle an event with multiple days (e.g., a 3-day music festival) | Future ADR | Before first multi-day event is onboarded |
| OQ-04 | Venue-managed VIP/hospitality seat blocking | Deferred feature | Post-v1.0 |
| OQ-05 | Co-promoter model (multiple Organisers on one event) | Deferred feature | Post-v1.0 |
| OQ-06 | Mobile native applications (iOS/Android) | Deferred feature | Post-v1.0, driven by adoption |
| OQ-07 | Overbooking policy based on no-show prediction | Deferred feature | After No-Show Prediction model is validated (Phase 5) |
| OQ-08 | Cross-session chatbot memory with DPDP consent model | NFR.md + ADR-TBD | Before Chatbot Service implementation (Phase 5) |
| OQ-09 | Exact DPDP Act data retention and anonymisation policy | Legal review | Before first data export request in production |

---

## 14. Glossary

| Term | Definition |
|---|---|
| **Actor** | A role in the StagePass system: Customer, Organiser, Venue, Admin. Each actor has disjoint permissions. |
| **Booking** | The transactional record of a Customer purchasing one or more tickets. The unit of saga coordination. |
| **CancellationPolicy** | An Organiser-defined set of time-bracket refund rules attached to an Event. Immutable after first ticket sold. |
| **DisbursementRecord** | An immutable ledger entry recording a fund transfer to a Venue or Organiser. |
| **Event** | A live entertainment or sports occasion created by an Organiser at a confirmed Venue. |
| **Flash Sale** | A high-demand ticket on-sale period where concurrent booking attempts may reach 10× normal RPS. |
| **GMV** | Gross Merchandise Value: total ticket revenue processed by the platform (before fee deductions). |
| **Non-fungible seat** | A seat that is unique and not interchangeable with any other seat. Every seat is a distinct inventory item. |
| **Organiser** | A verified business entity that creates and manages events. |
| **Outbox Pattern** | A reliability pattern where database writes and event publications are made atomic by writing events to an Outbox table in the same transaction, then publishing to Kafka from the Outbox. |
| **PricingTier** | A named price bracket applied to seating sections within an event. |
| **RAG** | Retrieval-Augmented Generation: an AI pattern where a language model's responses are grounded in retrieved documents. |
| **RevenueSplit** | An immutable record of how a booking's revenue is divided: Platform fee, Venue share, Organiser remainder. |
| **Saga** | A distributed transaction pattern that coordinates a sequence of local transactions across services, with explicit compensation for failures. |
| **SeatingLayout** | A versioned hierarchical seat configuration: Section → Row → Seat. |
| **SeatHold** | A time-limited (10-minute) exclusive reservation on a specific seat during checkout. |
| **SurgePricingRule** | An Organiser-configured rule that applies a price multiplier when seat availability crosses a defined threshold. |
| **T1/T2/T3** | Service criticality tiers determining SLO rigour. T1: 99.9%, T2: 99.5%, T3: 99.0%. |
| **Venue** | A registered entity that owns physical event spaces. |
| **VenueBooking** | A time-bound confirmed agreement between an Organiser and a Venue for exclusive use of the space. |
| **Waitlist** | An ordered FIFO queue for Customers who wish to purchase tickets to a sold-out event. |
