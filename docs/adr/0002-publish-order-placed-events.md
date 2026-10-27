# ADR-0002: Publish durable OrderPlaced events for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Student Architecture Review Board

## Context and problem

R-12 requires a MailFast confirmation, exactly-once loyalty point allocation within one minute, and an `order placed` analytics event after an order is placed (`docs/scenario.md`, R-12). Today, `checkoutservice.PlaceOrder` synchronously invokes `emailservice` via gRPC after payment and shipping (`system/online-boutique/src/checkoutservice/main.go`). A failure in email dispatch merely logs a warning. Crucially, `PlaceOrder` builds an in-memory `OrderResult` and returns without persisting the order state to a durable datastore, meaning there is no current transaction boundary to atomically write an outbox record.

Performance requirements mandate a checkout p95 latency of ≤ 500 ms (currently ~300 ms) at a peak of 200 orders/s. In contrast, MailFast exhibits a p95 of 1.5 s, a p99 of 8 s, a 0.5% failure rate, and enforces a strict rate limit of 50 emails/s (`docs/scenario.md`). Under peak load, an email backlog accumulates at ≥ 150 messages/s (540,000/hour). An asynchronous queue decouples latency but cannot solve this fundamental throughput mismatch without upstream rate control or quota expansion.

## Decision drivers

1. **Availability:** Decouple order acceptance from external/downstream service failures (Email, Loyalty, Analytics).
2. **Performance:** Maintain checkout p95 ≤ 500 ms at 200 orders/s peak.
3. **Correctness & Timeliness:** Award loyalty points within 60 seconds with strictly no double-awarding (at-least-once transport + idempotent execution).
4. **Capacity:** Adhere to MailFast’s 50 requests/s constraint and handle peak backlog predictably.
5. **Modifiability:** Facilitate future event consumers without altering core checkout logic.
6. **Team & Stack Constraints:** 4-person Go/gRPC team preferring self-hosted Kubernetes; 2 developers possess RabbitMQ operational experience.

## Considered options

1. **Synchronous gRPC calls** from `checkoutservice` to each downstream service.
2. **Asynchronous event-driven publishing via self-hosted RabbitMQ** (with Transactional Outbox pattern).
3. **Database-backed polling/task workers** using a shared relational database.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| **1. Synchronous calls** | Reuses existing gRPC topology; immediate response visibility. | MailFast latency directly breaches the 500 ms p95 SLA; cascading failures and strong checkout coupling. |
| **2. RabbitMQ + Outbox (Proposed)** | Decouples downstream latencies; independent retry policies & consumer scalability; aligns with team skill set and K8s setup. | Introduces broker operational overhead; requires deduplication/idempotency handling; requires introducing an order datastore for the outbox. |
| **3. DB-backed Polling** | Atomic order and event persistence in one DB transaction; no message broker needed. | Introduces database contention under 200 orders/s; polling latency risks missing the 1-minute loyalty deadline; less flexible routing. |

## Decision

**Adopt RabbitMQ fan-out messaging combined with a Transactional Outbox pattern on an introduced order datastore.**

1. **Transactional Outbox in `checkoutservice`:**
   - Introduce a lightweight durable datastore (e.g., PostgreSQL) for `checkoutservice`.
   - In `PlaceOrder`, persist the `orderID`, order details, and an `OrderPlaced` outbox record within a single ACID transaction before responding with `OrderResult`.
   - Use an asynchronous publisher (Relay worker or CDC) to read the outbox table and publish events to RabbitMQ with publisher confirms enabled.

2. **RabbitMQ Architecture:**
   - Publish `OrderPlaced` events (carrying `orderID`, `timestamp`, `userID`, and payload) to a **Topic Exchange** (`order.exchange`).
   - Bind dedicated durable queues: `order.email`, `order.loyalty`, and `order.analytics`.
   - Configure dead-letter exchanges (DLX) and dead-letter queues (DLQ) for poisoned/failed messages after bounded exponential backoff retries.

3. **Consumer Specifications:**
   - **Email Consumer:** Bound to `order.email`. Enforce a token bucket/rate limiter capped at 45–50 req/s to prevent MailFast throttling. Acknowledge messages (`basic.ack`) only after successful HTTP dispatch or routing to DLQ.
   - **Loyalty Consumer:** Bound to `order.loyalty`. Enforce idempotency by storing `orderID` within a unique constraint/transaction in `loyaltyservice`. If an `orderID` is already processed, silently acknowledge without re-awarding.
   - **Analytics Consumer:** Bound to `order.analytics`. Ingests events asynchronously into the target analytics engine.

## Consequences

- **Positive:**
  - Downstream latency and outages (MailFast, Loyalty) are fully decoupled from checkout latency, maintaining checkout p95 well below 500 ms.
  - Adding future event listeners requires no code changes to `checkoutservice`.
- **Negative:**
  - Increases operational surface area: requires managing RabbitMQ clustering, monitoring queue depths, and running an Outbox store.
  - Peak traffic (200 orders/s) causes unavoidable email delay due to the 50 emails/s cap.
- **Risks & Mitigations:**
  - *Risk:* Duplicate deliveries due to network partitions or worker restarts.
    - *Mitigation:* Ensure downstream consumers are idempotent via unique order constraints.
  - *Risk:* Loyalty points SLA breach (> 60 s).
    - *Mitigation:* Prioritize `order.loyalty` queue consumers; alert if queue age exceeds 30 seconds.

## Validation

1. **Performance SLA:** Under 200 orders/s peak load, verify checkout p95 remains ≤ 500 ms while introducing simulated 5-second latency and 50% errors on downstream consumers.
2. **Fault Injection & Consistency:** Kill the publisher and consumer processes mid-transaction (pre-outbox, post-outbox, post-broker delivery). Reconcile accepted orders against the event ledger and verify zero lost events.
3. **Idempotency Verification:** Replay identical `OrderPlaced` messages to `loyaltyservice`; verify loyalty points are applied strictly once per `orderID`.
4. **Rate Limiting:** Ensure outbound calls to MailFast never exceed 50 req/s under maximum queue backlog.

## Open questions

- What is the anticipated duration of the 200 orders/s peak, and what is the maximum acceptable delay for confirmation emails from a business standpoint?
- Can MailFast quotas be upgraded, or should a secondary email provider fallback be evaluated?
- Which specific database engine will be introduced for the order persistence and outbox table in `checkoutservice`?
- What are the data retention and PII redaction policies required for payload fields in `OrderPlaced` analytics events?