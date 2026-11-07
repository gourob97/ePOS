# ePOS — Requirements

Offline-first POS/inventory app for Android. Portfolio project: the goal is correct, defensible architecture, not feature breadth.

## Problem

Several devices at one store must keep selling with no network, then reconcile when connectivity returns — without losing data or double-charging customers.

## Scope

**In:** one store, multiple devices, inventory, sales, payments, background sync, mock backend.
**Out:** multi-store, auth/users, tax, receipts, real payment gateway, refunds.

## Functional requirements

| ID | Requirement |
|----|-------------|
| FR-1 | Every stock change (sale, delivery, manual adjustment) is stored as an immutable event. No mutable stock field exists. |
| FR-2 | Current stock per product is derived by replaying events. |
| FR-3 | A detected oversell is flagged for manual review — never clamped, rejected, or hidden. |
| FR-4 | Resolving an oversell flag is itself a new event (audit trail stays append-only). |
| FR-5 | A device can record sales with no network. |
| FR-6 | Events sync to the server when the network returns. |
| FR-7 | Each event has a stable ID created on the device; it is the sync idempotency key. A re-sent event is never applied twice. |
| FR-8 | A payment retry with the same key and amount returns the cached result and does not call the gateway again. |
| FR-9 | Same key with a different amount is an error. |
| FR-10 | UI: inventory list, record-sale flow, sync status indicator, offline-mode toggle (for demos). |

## Retry rules

| ID | Rule |
|----|------|
| RR-1 | Only transient failures (network, timeout, 5xx) are retried. |
| RR-2 | Business rejections (invalid data, declined card) are never retried. |
| RR-3 | Backoff is exponential with jitter, capped. |
| RR-4 | After N attempts the item moves to `FAILED_PERMANENT` and is shown in the UI. |
| RR-5 | A failed item does not block other items in the queue. |

## Non-functional requirements

| ID | Requirement |
|----|-------------|
| NFR-1 | Domain logic (event store, projector, sync engine, payment wrapper) is plain Kotlin with zero Android dependencies. |
| NFR-2 | Domain logic is unit-tested on the JVM without mocking Android classes. |
| NFR-3 | Room, Compose, networking, and WorkManager are thin adapters around the domain layer. |
| NFR-4 | Time and randomness are injected so retry/backoff tests are deterministic. |
| NFR-5 | Every non-obvious tradeoff is recorded in `docs/DECISIONS.md`. |

## Known constraint

If a payment request times out, the client cannot know whether the charge succeeded. A local cache alone cannot prevent a double charge in that case. The idempotency key must be saved **before** the first gateway call, and the gateway must honour it. The fake gateway will model "charged, but response lost".

## Open decisions (need owner answer)

| ID | Question | Options | Recommendation |
|----|----------|---------|----------------|
| OD-1 | When is a sale an oversell? | A: final stock < 0 (order doesn't matter) · B: stock < 0 at any point in time (needs reliable cross-device ordering) | A |
| OD-2 | Card payments while offline? | Cash only offline · Queue card and charge later (decline risk) | Cash only |
| OD-3 | Do devices download other devices' events? | Yes · No (each device sees only its own sales) | Yes |

## Build plan and acceptance criteria

| Step | Deliverable | Done when |
|------|-------------|-----------|
| 1 | `InventoryEvent`, `EventStore` + in-memory impl, `InventoryProjector` with oversell flags | JUnit tests pass |
| 2 | Sync engine: queue, backoff, dedup | Tests pass against a fake flaky `RemoteApi` |
| 3 | Payment idempotency wrapper | Tests pass against a fake flaky gateway, incl. lost-response case |
| 4 | Room-backed `EventStore` | Passes the same tests as the in-memory impl |
| 5 | Compose UI (FR-10) | Builds; screens wired to domain |
| 6 | Ktor mock backend + WorkManager sync | End-to-end sync works against the mock server |

## Environment note

The build container has JDK 21 and Gradle, but no Android SDK. Steps 1–3 and the Ktor server can be built and tested here. Steps 4–5 need the SDK installed; whether that's possible here is not yet confirmed.
