# Core 00 — Business context and delivery plan

## Business context

A small bookstore sells through a storefront, an in-store terminal, and phone orders. Catalog changes are inconsistent across channels, and two customers can attempt to buy the last copy simultaneously. Staff need one API to own catalog rules and order intake before more channels are added.

The core problem is not “build CRUD.” It is to keep identity, price, stock, and order-total rules consistent under concurrent requests while exposing a stable contract operators can support.

## Request

Deliver a modular monolith that owns:

- Book creation, update, lookup, deletion, search, and paging.
- Order placement and retrieval.
- Server-side totals and verified prices.
- Atomic stock decrement with a clean conflict for the last-copy race.
- Versioned HTTP contracts, machine-readable errors, logs, health, metrics, and tests.

## Acceptance criteria

- Invalid ISBN, price, quantity, stock, or empty orders cannot enter a valid domain state.
- Two orders for the final copy produce one success and one `409 Conflict`.
- External quote calls finish before the short database transaction begins.
- Domain tests run without Spring.
- Controllers expose DTOs, never persisted rows.
- Every dependency from adapters points toward application/domain.

## Domain vocabulary

| Term | Meaning |
|---|---|
| Book | Catalog aggregate with ISBN, descriptive data, price, stock, and version |
| Order | Purchase aggregate containing immutable order-line snapshots |
| Price quote | External confirmation of price with identity and expiry |
| Reservation | Temporary allocation introduced only in the distributed Saga track |
| Fulfillment | Shipment lifecycle owned later by the reactive companion API |

## Delivery strategy

Begin with a deliberately small annotated CRUD slice so Spring mechanics are visible. When business rules and a second aggregate create pressure, refactor into feature-first hexagonal architecture. The lesson must show why each abstraction became necessary; no interface is added merely because a diagram contains a box.

## Do and don't

- **Do:** keep the first runnable slice small and admit its architectural shortcuts.
- **Do:** use one database transaction inside the monolith when one database owns the invariant.
- **Don't:** introduce Kafka, distributed transactions, or WebFlux before the local model works.
- **Don't:** split services around libraries. Split around ownership and independent change/scale needs.

## Exit criteria

You can describe the actors, invariants, failure behavior, and why the core starts as a monolith. Continue to [Kotlin domain foundations](01-kotlin-domain-foundations.md).

## Performance and production note

A monolith is not inherently slow or unscalable. One process and one local transaction are usually the lowest-latency, easiest-to-operate answer while one team and database own the invariant. Add distribution only from measured or organizational pressure.
