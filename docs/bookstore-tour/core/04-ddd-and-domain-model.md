# Core 04 — Pragmatic DDD and the richer domain model

## Business context

CRUD is working, but adding orders reveals that database-shaped records cannot safely own pricing, stock, and purchase invariants. The team also uses “book,” “stock,” “order,” and “fulfillment” inconsistently.

## Decision

Apply lightweight strategic and tactical DDD. The goal is shared language and correct consistency boundaries, not a large framework or ceremony.

## Bounded contexts

| Context | Owns |
|---|---|
| Catalog | Book identity, descriptive information, price, and current stock in the core monolith |
| Ordering | Order lifecycle, line snapshots, totals, and purchase policy |
| Fulfillment | Shipment and tracking lifecycle in the later WebFlux API |
| Notification | Delivery of customer messages, never the truth of an order |

## Tactical concepts

- **Entity:** identity matters across change, such as `Book` or `Order`.
- **Value object:** meaning comes from attributes, such as `Isbn`, `Money`, or `OrderLine` snapshot.
- **Aggregate:** consistency boundary changed in one local transaction.
- **Repository:** collection-like access to aggregate roots, not one repository per table.
- **Domain service:** domain policy that belongs to no single entity.
- **Application service:** coordinates a use case and ports; it does not own transport or SQL.
- **Domain event:** a past-tense fact raised by domain behavior.

## Refactor direction

Separate `Book` from `BookRow`. Persisted rows describe storage; domain objects enforce behavior. Orders reference books by id and capture unit price rather than navigating a cross-aggregate object graph.

Use `shared` only for stable concepts genuinely shared between contexts. It must not become a dumping ground for DTOs, exceptions, or helpers.

## When not to use rich DDD

Reference data and trivial admin CRUD can remain simple. A rich aggregate earns its cost when invariants, lifecycle, concurrency, or ubiquitous language matter. Event sourcing is not required for DDD.

## Exit criteria

The vocabulary, aggregates, and boundaries are explicit. Continue to [use cases and ports](05-use-cases-ports-and-services.md).

## Performance and production note

DDD is a modeling strategy, not a runtime optimization. Aggregate boundaries can improve consistency but overly large aggregates increase load/write cost and contention. Keep transactions around the smallest business consistency boundary.
