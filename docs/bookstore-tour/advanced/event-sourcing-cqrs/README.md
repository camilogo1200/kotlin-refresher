# Selective event sourcing and CQRS

This optional track redesigns only the fulfillment aggregate because its business state is naturally a chronological timeline. Catalog and ordinary CRUD keep current-state persistence.

## Lessons

1. [EDA, Kafka, and event sourcing are different](00-eda-is-not-event-sourcing.md)
2. [Fulfillment event store and rehydration](01-fulfillment-event-store.md)
3. [CQRS tracking projections](02-cqrs-projections.md)
4. [Trade-offs, snapshots, and evolution](03-tradeoffs-and-evolution.md)

Adopt event sourcing only when audit/reconstruction, temporal reasoning, or multiple rebuildable projections justify the schema, replay, concurrency, and operational cost.

