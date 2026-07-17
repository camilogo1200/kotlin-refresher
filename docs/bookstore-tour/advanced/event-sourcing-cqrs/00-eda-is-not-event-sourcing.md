# Event sourcing 00 — EDA is not event sourcing

## Business context

The system already publishes Kafka events, so it is tempting to claim it is event sourced. That would be incorrect: integration events distribute facts, while event sourcing changes the authoritative persistence model of an aggregate.

## Distinction

| Concept | Role |
|---|---|
| EDA | Communication style among producers, channels, and consumers |
| Kafka log | Durable distribution/replay mechanism partitioned for consumers |
| Event store | Aggregate-oriented source of truth with ordered stream and expected-version writes |
| Event sourcing | Reconstruct aggregate state by replaying persisted domain events |
| CQRS | Separate command/write and query/read models; may exist without event sourcing |

Kafka is not automatically the fulfillment event store. The lesson uses an append-only PostgreSQL event table with per-aggregate ordering and optimistic expected version, then publishes selected integration events to Kafka through an outbox.

## Acceptance criteria

- Current fulfillment state can be rebuilt from its event stream.
- Concurrent writers cannot append against the same expected version.
- Integration payload changes do not mutate historical domain events silently.
- Replaying projections does not resend external customer notifications.

## Use, avoid, and operational notes

Use event sourcing only when authoritative history/reconstruction is a business requirement. Avoid adopting it because Kafka already exists. Replay throughput, stream length, schema compatibility, storage growth, and privacy/retention are production concerns from day one.
